# Step-by-Step Explanation: Data Access → Final Customer Promotion Score

## Step 1. Data Access

**What happens:** The Gold customer tables (purchase history, engagement, promo history, consent flags) are connected to Azure ML through datastores and registered as versioned data assets. The ML team never copies files around; they reference a named, versioned asset (e.g., `customer_gold:v12`).

**Why it matters:** Every training run and scoring run points to an exact data snapshot, so results can be reproduced and audited.

**Patterns and best practices:**
- **Versioned data assets and lineage:** you can always answer "which data produced this score?"
- **Managed identity and Key Vault:** no credentials in code, least-privilege access to the lake.
- **Read from Gold only:** ML consumes curated, quality-checked data and doesn't re-clean raw data.

## Step 2. Feature Preparation

**What happens:** Pipeline components turn customer records into model-ready features: rolling 30/90-day spend, purchase frequency, recency, discount sensitivity, promo redemption rate, channel engagement. These are stored in the managed feature store.

**Why it matters:** Features are the model's inputs. If training uses one calculation and scoring uses another, scores become unreliable.

**Patterns and best practices:**
- **Feature store pattern:** one definition, reused in training and scoring, which prevents training/serving skew.
- **Point-in-time correctness:** features are built only from data available before the outcome, avoiding label leakage.
- **Pipeline (DAG) components:** each step is modular, testable, and cached, so unchanged steps don't rerun.

## Step 3. Training (PyTorch)

**What happens:** An Azure ML command job spins up a GPU or CPU cluster using a curated PyTorch environment. The PyTorch tabular network (propensity or uplift) is trained with a train/validation split, early stopping, and class-imbalance handling, since most customers don't respond. Compute shuts down when the job ends.

**Why it matters:** This is where the model learns who responds to promotions.

**Patterns and best practices:**
- **Separation of code, environment, and data:** the same job runs identically in dev, test, and prod.
- **Ephemeral compute:** you pay only while training runs.
- **Fixed random seeds and pinned environments:** reproducible results.
- **Baseline comparison:** compare the network against a simple model (e.g., logistic regression) to justify the complexity.

## Step 4. Tuning and Tracking

**What happens:** Sweep jobs run many trials with different hyperparameters (learning rate, layers, dropout, batch size). MLflow records every run's parameters, metrics, and artifacts.

**Why it matters:** It finds the best configuration systematically and keeps a full experiment history.

**Patterns and best practices:**
- **Experiment tracking:** any model can be traced to its exact settings and data version.
- **Early termination policies:** weak trials are stopped to save cost.
- **Tune on validation, report on a separate test set:** avoids over-optimistic results.

## Step 5. Evaluation and Registration

**What happens:** The best candidate is evaluated on held-out data using AUC, lift, precision in the top decile, and calibration. The Responsible AI dashboard checks fairness across segments and produces explainability insights. A threshold gate decides: pass means the model is registered in the model registry as a new version; fail means it stops.

**Why it matters:** Only proven models reach production.

**Patterns and best practices:**
- **Quality gate (MLOps gating):** promotion is automatic and rule-based, not opinion-based.
- **Model registry:** versioned, tagged models with metadata (data version, metrics, owner).
- **Business-relevant metrics:** top-decile lift matters more than raw accuracy because campaigns target the top of the list.
- **Fairness and explainability review:** reduces regulatory and brand risk.

## Step 6. Deployment

**What happens:** The registered model is deployed to a **batch endpoint** (nightly or weekly scoring of all customers) and optionally a **managed online endpoint** (real-time scoring for in-session offers). New versions go into a separate slot first.

**Why it matters:** The model becomes a callable service the rest of the platform can use.

**Patterns and best practices:**
- **Batch-first design:** most promotion campaigns are planned in bulk, so batch is cheaper and simpler. Real-time is added only where it earns its cost.
- **Blue/green, shadow, and champion-challenger rollout:** the new model proves itself on real traffic before replacing the current one, with instant rollback.
- **Autoscaling and health probes:** reliability under load.
- **Private networking:** endpoints are not exposed to the public internet.

## Step 7. Raw Scoring

**What happens:** The scoring script loads the model once at startup (the init function), then for each batch or request (the run function) pulls features, runs the PyTorch forward pass, and returns a raw propensity or uplift value per customer.

**Why it matters:** This is the pure model opinion, before any business policy.

**Patterns and best practices:**
- **Load once, score many:** efficient inference.
- **Same feature definitions as training:** consistent inputs.
- **Idempotent batch jobs:** rerunning a scoring job gives the same output, with no duplicates.
- **Input validation:** bad or missing values are caught before they reach the model.

## Step 8. Business Rules

**What happens:** A rules component takes each raw score and applies policy:
- **Eligibility:** no consent, opted out, or inactive means excluded.
- **Suppression:** recently contacted or already redeemed means held back.
- **Caps:** budget, contact frequency, and margin limits are enforced.
- **Overrides:** VIP treatment, regional or strategic campaigns.

Optionally, a Foundry agent or prompt flow step interprets unstructured policy text (for example, a new campaign brief) and helps translate it into rule updates for human approval.

**Why it matters:** A high model score doesn't always mean "send the offer." Compliance, cost, and strategy must be honored.

**Patterns and best practices:**
- **Hybrid decisioning (ML + rules):** the model predicts, the rules decide. Policy changes don't require retraining.
- **Externalized, versioned, config-driven rules:** business teams can change thresholds with an audit trail and no redeployment.
- **Compliance first:** consent and opt-out checks are hard rules that the model can never override.
- **Human-in-the-loop for AI-suggested rules:** the Foundry agent proposes, a person approves.

## Step 9. Final Score and Tier

**What happens:** Raw score and rule outcomes are blended into the **Customer Promotion Score**, converted to a tier (High, Medium, Low), and given reason codes such as "high recent engagement" or "suppressed: contacted 3 days ago." Each record is tagged with model version, feature snapshot, and rules version, then written back to Gold.

**Why it matters:** This is the deliverable. Marketing gets a usable, explainable number instead of a raw probability.

**Patterns and best practices:**
- **Explainability by design:** reason codes build trust and help marketers act.
- **Lineage tagging:** any score can be reproduced and defended in an audit.
- **Upsert, not overwrite:** history is kept so score changes can be analyzed.
- **Data contract on the output:** a fixed schema so downstream consumers don't break.

## Step 10. Foundry Layer (optional intelligence)

**What happens:** The scoring endpoint is registered in an Azure AI Foundry project as a governed connection or tool. An agent can then answer questions like "Why is this customer High tier?" or "Draft a promotion rationale for this segment," by calling the endpoint and reading the reason codes. Foundry evaluations, content safety, and tracing monitor the agent's behavior.

**Why it matters:** It turns scores into conversations and decisions for non-technical users.

**Patterns and best practices:**
- **Model-as-a-tool:** the model stays decoupled from its consumers, so agents, apps, and BI all use the same governed endpoint.
- **Guardrails:** content safety and grounding checks keep agent answers within the scoring data.
- **Tracing and evaluation:** agent quality is measured, not assumed.
- **Least-privilege tool access:** the agent can read scores but not change rules or models.

## Step 11. Monitoring and Retraining

**What happens:** Azure ML model monitoring watches for data drift (customer behavior changes), prediction drift (score distribution shifts), and outcome performance (did high-score customers actually respond?). Campaign results flow back through Bronze, Silver, and Gold as new labels. Alerts or schedules trigger the pipeline to retrain from Step 2 or 3.

**Why it matters:** Customer behavior, seasons, and markets change, so a model that was good in January may be weak by June.

**Patterns and best practices:**
- **Closed feedback loop:** the system improves from its own results.
- **Drift thresholds with alerts:** retrain when needed, not blindly.
- **Champion-challenger on retrain:** the new model must beat the incumbent before replacing it.
- **Holdout control group:** a small group receives no model-driven targeting, so true campaign lift can be measured.

## End Result

Marketing, CRM, and BI receive a daily or weekly **Customer Promotion Score** per customer with tier, reason codes, and lineage, produced by a model that is versioned, tested, fair, monitored, and constrained by business policy. Any score can be traced back to the exact data, features, model, and rules that produced it.

## How patterns tie the whole flow together

| Concern | Pattern that solves it |
|---|---|
| Reproducibility | Versioned data assets, pinned environments, registry |
| Consistency | Feature store (same features in training and scoring) |
| Trust in the model | Evaluation gates, fairness checks, explainability |
| Safe change | Blue/green, shadow, champion-challenger |
| Policy compliance | Hybrid decisioning with externalized rules |
| Cost control | Batch-first, ephemeral compute, early termination |
| Auditability | Lineage tags, upserts with history, tracing |
| Long-term accuracy | Monitoring, drift detection, closed feedback loop |
| Reuse across teams | Model-as-a-tool through Foundry |
