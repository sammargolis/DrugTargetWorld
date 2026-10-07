# Architecture

How a DrugTargetWorld instance is built, what the agent can reach, and what stays sealed.
The scoring contract is in [EVALUATION.md](EVALUATION.md) and the agent-facing brief is in [TASK.md](TASK.md).

> [!NOTE]
> This document describes the design of the environment, not the contents of this repository.
> Every path named below refers to the [source tree](https://github.com/sammargolis/cardiobench/tree/v2RWEBench/harbor), not to a file in this repository.
> Running the benchmark does not require the source; Harbor pulls a prebuilt image.

## Loop

```
        seed + config
              │
              ▼
      ┌───────────────┐
      │  generator/   │
      └───────┬───────┘
              │
      ┌───────┴────────────────────────────┐
      ▼                                    ▼
┌───────────┐                      ┌───────────────┐
│ release/  │  the agent reads     │   sealed/     │  answer key
│  4.1 GB   │  only this           │    ~1 MB      │  organizer only
└─────┬─────┘                      └───────┬───────┘
      │                                    │ loaded once at startup
      │                                    ▼
      │                            ┌───────────────┐
      │        budgeted            │   oracle/     │  re-runs the SCM
      │        interventions ─────▶│    server     │  budget + audit here
      │                            └───────────────┘
      ▼                                    │
┌───────────┐                              │ sealed truth
│   agent   │                              │
└─────┬─────┘                              ▼
      │  submission.json            ┌───────────────┐
      └────────────────────────────▶│   scoring/    │──▶ score.json
                                    └───────────────┘
```

The two halves of an instance are asymmetric, and the asymmetry drives how they are handled.

| | size | regenerable | who may read it |
|---|---|---|---|
| `release/` | 243 MB without imaging, 4.1 GB with | yes, from the seed | the agent |
| `sealed/` | ~1 MB | no | organizer and scorer only |

Release data is therefore disposable and sealed data is not.

## The world, layer by layer

The generator is one causal chain.
Each arrow is a structural equation, and the whole chain re-runs under intervention.

```
  genotypes ────────────┐
  (LD blocks, strata)   │
                        ▼
  latent factors ──▶ proteome ──┬──▶ transcripts
  (12, unobserved)   (2,941)    └──▶ metabolites
                        │
                        │  drivers, weighted
                        ▼
  covariates ──────▶ latent trait L ──┬──▶ morphology ──▶ cine-MRI
  (age, sex, BMI,      (severity)     │    (theta)        phantom or ACDC
   smoking, site;          │          ├──▶ ICD-10 / ATC records
   released under          │          ├──▶ ECG, coronary
   UKB field IDs)          │          └──▶ survival: mortality, MACE
                           │
                           ▼  five years on
                    L2 = 0.75·L + 0.35·(P[drivers] @ w_late)
                           │
                           └──▶ visit-2 proteome, morphology, imaging
```

The latent disease state is a weighted sum over the hidden drivers, the covariates, a direct genetic effect, and noise:

```
L_i = z[ Σ_{j∈D} w_j P_ij  +  γᵀ C_i  +  δ G_i,direct  +  ε_i ]
```

`w_late` differs from `w` for one driver per world, which is what lets an effect be near-invisible at visit 1 and substantial by visit 2.

## Temporal and causal order

The fixed synthetic index date is `2020-01-01`.
Diagnosis dates before that date are prevalent (`date < index_date`) and dates on or after it are incident (`date >= index_date`).
EHR long-table rows carry the corresponding `disease_phase` value.

The realism extension is an acyclic sequence, evaluated after baseline latent severity exists and before release masking:

```
baseline C, G, P*, L
        │
        ▼
prevalent disease D- ──▶ treatment assignment A
        │                         │
        └──────────────┐          ▼
                       └──────▶ observed assays Y
                                      │
                                      ▼
                         informative missingness M
                                      │
                                      ▼
                               released Y with nulls
```

The contract equations:

```
P(D-_i = 1 | L_i, C_i)  = sigmoid(alpha_d + beta_d L_i + gamma_d C_i)
P(A_ik = 1 | D-_i, C_i) = sigmoid(alpha_k + theta_k D-_i + phi_k C_i)
Y_ij                    = P*_ij + sum_k B_jk A_ik + epsilon_obs_ij
P(M_ij = 1 | L_i, site_i, participation_i)
                        = sigmoid(alpha_j + beta_L L_i + beta_site[site_i]
                                  + beta_part participation_i)
```

Treatment assignment does not consume observed assays, and observed assays do not feed back into treatment assignment or latent severity.
The untreated assay `P*` is organizer-side state and is never released as a second table.
Missingness is applied last, so it cannot alter the latent trait, diagnosis dates, treatment state, or any downstream outcome.
Mechanism configuration contains no driver identity, driver identifier, or participant-level mechanism probability.
The named streams are `prevalent_disease`, `treatment_assignment`, `treatment_effects`, `missingness_proteomics`, `missingness_transcriptomics`, and `missingness_metabolomics`.

The release boundary therefore exposes observed assays, nulls, diagnoses, medications, and released covariates, while untreated assays, missingness probabilities, and driver-linked configuration remain sealed.

## Intervention

`simulate_scm(..., clamp={molecule: value})` pins a molecule and re-runs everything downstream.
That single hook is what makes an intervention a real counterfactual rather than a lookup.

```
  observational                     interventional
  ─────────────                     ──────────────
  proteome as generated             proteome with molecule j clamped
        │                                   │
        ▼                                   ▼
   L, morphology,                     L', morphology',
   survival                           survival'
        │                                   │
        └────────────  delta  ──────────────┘
                         │
                         ▼
              what the oracle returns
```

Non-drivers return a delta of exactly zero, because clamping them changes no downstream equation.

## Isolation

What the agent can reach, and what it cannot.

```
   organizer side                  │        agent side
  ─────────────────────────────────┼──────────────────────────────
   sealed/                         │   private copy of release/
   generator source                │   task statement
   scoring/                        │   oracle_client.py (stateless)
   oracle server process           │   its own working directory
   audit log                       │
                                   │
        ▲                          │            │
        └──── HTTP, budgeted ──────┴────────────┘
              token-authenticated
```

The oracle server loads sealed state at startup and never re-reads disk, so the sealed directory can be made unreachable while an agent runs.
The audit log is written outside the agent's working directory, and the client holds no state, so filesystem access on the agent side reveals nothing.

A second, governance audit sits beside it.
Every `/knockdown` request and every piece of agent-side evidence the harness captures is checked against a data-use policy — individual-level egress, cross-cohort joins, re-identification probing, out-of-scope access — and recorded to the same directory as the oracle audit log, outside the agent's working directory.
The oracle-side file is written only from the server's own in-memory state and never read back; the agent-side file is written once, after the agent process has exited, from evidence computed fresh each time rather than merged with whatever is already on disk.
Neither file is ever read before the agent starts, so nothing the agent could place at either path is ever trusted.
In v0.9 this is logged, not scored: the scorer does not import the governance policy and never reads either file.

## Components

| path | role |
|---|---|
| `generator/scm.py` | The causal chain in one call. Its `clamp` argument is the do-operator: pin a molecule, the chain re-runs. It also builds the per-molecule targetability sheet — constraint, localisation, binding pocket, paralog redundancy, tissue specificity — drawn from its own stream, independent of driver identity or weight, and published to `release/targetability.parquet`. The one planted exception is the T8 surrogate molecule, whose constraint and tissue-specificity draws come from a compressed range so it is measurably more constrained and more broadly expressed than an ordinary driver. The manifest also records `driver_direction`, the sign of each driver's weight as `inhibit` or `activate`, so direction-of-effect truth is never recomputed from `driver_weights`. |
| `generator/generate.py` | Orders the layers and writes both halves. Owns the visit-2 accumulation `L2 = 0.75*L + 0.35*(P[drivers] @ w_late)` that makes slow-acting drivers possible. |
| `generator/genotypes.py` | LD-blocked diploid genotypes, Gaussian-copula haplotypes, VCF export. |
| `generator/imaging.py`, `acdc_warp.py` | Phantom cine-MRI, or real cardiac anatomy warped to each subject's latent severity. |
| `generator/ehr.py`, `survival.py`, `modalities.py` | Downstream layers off the latent trait. |
| `generator/rng.py` | `stream(master_seed, component)`. All randomness. |
| `generator/provenance.py` | Content hash → `instance_id`. |
| `scoring/score.py` | Rubric v0.9, 100 points: target identification, causal confidence, discrimination, direction of effect, phenotype construction, and an asymmetric safety penalty. Prior rubrics are preserved byte-identically; scores are not comparable across versions. |
| `oracle/oracle_server.py` | Loads sealed state at startup and never re-reads disk. Server-side budget, token auth, audit log outside the agent's directory. Checks every request against the governance policy and records the result to a second file in the same directory. |
| `oracle/oracle_api.py` | Performs the intervention by re-running the structural equations. |
| `oracle/oracle_client.py` | The stateless stub the agent receives. |
| `oracle/governance_policy.py` | The data-use policy as data: four categories, each with a detection rule. Logged, not scored. |
| `harness/run_agent.py` | One agent, one instance, one protocol. Timeouts and crashes score zero rather than vanishing. |
| `harness/run_study.py`, `build_panel.py` | Agent×instance grids; registered instance panels with provenance. |
| `harness/prepare_arena.py` | Self-contained folder for an external agent; verifies no sealed leak. |
| `harness/budget_probe.py` | What the rubric pays for intervention budget, via an upper-bound policy. |
| `harness/baselines/` | Naive, informed, and an exploit control that games format without doing science. |
| `harness/agents/sandbox.py` | Runs agent-authored code in a subprocess handed the release path and nothing else, every turn rather than only for phenotype construction. Feedback reports whether the code ran and the shape of its output, never the score — revealing the score would hand over the answer being sought. |
| `harness/agents/code_loop.py` | One protocol for the whole episode: each turn the agent sends one Python block, the harness runs it in the sandbox and returns only what it printed. There is no action menu, because real UK Biobank has no `genetic_instrument` button and a policy learned against one could not transfer. The agent writes its own phenotype construction and its own genetics; the only things it cannot do in code are the two that cost money, requested via `request_experiment` and charged by the parent process. |
| `actions/registry.json` | The action catalogue. Analysis of released data costs nothing, because a regression over data the cohort already holds is compute rather than a purchase. Only laboratory work is priced, and the largest budget is exactly 5 knockdowns. |
| `actions/library.py` | The 7 actions: association; covariate-adjusted association with confounding and selection screens; phenotype construction from imaging; GWAS with pQTL mapping and a mediation test; a shared-instrument check; and 2 interventions. |
| `actions/meter.py` | Charges each action, bounds an episode by money and by steps, and appends the two canonical logs every figure is rebuilt from. Free actions cannot exhaust a budget, so the step ceiling is what terminates an episode. |
| `actions/world.py` | The only reader of a released tree. Sealed state loads through a separate class the policies never receive. |
| `policies/library.py` | 5 reference policies: random, association-only, a fixed evidence ladder, an adaptive reference, and an omniscient ceiling. They share one reporting layer, so the comparison measures which evidence each chose to buy rather than which kept better records. |
| `experiments/decision_value.py` | Forks each decision state, takes every available action once, completes with a reference policy, and reports the spread in attainable score. General to any agentic environment: it asks whether training is worth funding before it is funded. |
| `experiments/integrity_audit.py` | Truth isolation, score determinism, oracle parity against the in-process model, procedural variation, decoy strength, nontriviality. |
| `experiments/run_policies.py`, `run_agents.py`, `export_csv.py`, `make_figures.py` | The results pipeline. Resumable by episode id; every figure regenerates from the episode logs without rerunning an experiment. |
| `reasoning/` | Post-hoc interview, literature corroboration, method provenance. Answers what the score does not. |
| `realism/` | Compares the synthetic cohort against published real statistics. |

## Boundaries

**release / sealed.**
Anything handing data to an agent copies from `release/`, never exposes `sealed/`, and never discloses the instance path.
`prepare_arena.py` enforces both.

**The scorer depends on nothing upstream.**
It reads a submission and a sealed directory.
That isolation is what allows it to stay frozen while the rest changes.

**The oracle is out of process.**
Budget lives in server memory and the audit log is written where the agent cannot reach it.
Re-instantiating the client resets nothing.

**Named streams make extension safe.**
A new component takes a new stream name and perturbs no existing one.

## Extending

| to add | do |
|---|---|
| a planted trap | construct it in `scm.py` beside the others, record it in the sealed manifest, and give the scorer a way to recognise a correct rejection |
| an evidence action | extend the oracle; it already runs the real model |
| a difficulty axis | add a named suite to `benchmark_suites.json` so the condition is versioned rather than an ad hoc flag |
| a modality | read the latent trait and covariates, write a table to `release/`, take a new stream name |

## The task

The released data carries no phenotype column, and constructing one is part of the task rather than a preliminary to it.
An agent derives a per-subject phenotype from the raw cine and native T1 arrays, and that phenotype is scored against sealed truth on subjects held out from it.
Every later analysis regresses against what the agent built rather than against a coded outcome, which is what makes the genetic step proteome-wide Mendelian randomization on a derived trait.

Native T1 maps carry a global level and a spatial fibrosis pattern.
The level drifts between sessions for reasons unrelated to tissue, so averaging the myocardium recovers the latent state at r = 0.57 while reading the spatial arrangement reaches 0.80.
That gap is deliberate: it is the headroom a learned phenotype has to close, and it is the same separation reported when learned imaging features outperform standard T1 measures.

Two ambiguities cannot be resolved by observation.
Candidates driven by one shared instrument are not separable by genetics, and naming them as unidentifiable scores where guessing between them does not.
A surrogate marker moves the phenotype and survival in the same direction under intervention, where a genuine driver moves them apart, so only an intervention reporting both can tell them apart.

## Structural limits

- *cis* fraction is 1.0 against roughly 0.297 in real data, so Mendelian randomization is easier here than in reality.
- The hard and messy tiers are genuinely hard and the standard tier is the easy end, so results are reported by tier rather than pooled.
- Molecules carry no biological identity, so no prior knowledge of real biology transfers.
- Missingness is informative through bounded severity, site, participation, and assay-level intercept terms.
- Individuals are sampled independently, so there is no relatedness structure.
