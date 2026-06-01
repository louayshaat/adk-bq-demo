# Agents, ADK & BigQuery — Live Demo Run-Sheet

**Demo:** CleanSight (`GoogleCloudPlatform/data-to-ai`) · **Length:** 10–15 min · **Audience:** mixed (business + dev)

**The story (say it once):**

> *“From a photo in a bucket to a crew getting dispatched — no model training, just SQL and an agent.”*

**Flow:** bus cameras → images in Cloud Storage → BigQuery object table → Gemini analysis → `image_reports` → auto-created `incidents` → ADK agent prioritizes via ridership forecast → schedules + drafts the crew email.

-----

## ✅ PRE-BUILD — the day before (NOT live, ~30–60 min, one-time)

Use the repo’s **Open in Cloud Shell** button, or:

```bash
git clone https://github.com/GoogleCloudPlatform/data-to-ai && cd data-to-ai

# 1) Enable APIs
cd infrastructure/project-setup
echo 'project_id = "core-demos"' > terraform.tfvars
terraform init && terraform apply

# 2) Build env: bucket, dataset, Gemini models, Cloud Run, scheduler, agent IAM
cd ../terraform
printf 'project_id = "core-demos"\nnotification_email = "louays@google.com"\n' > terraform.tfvars
terraform init && terraform apply
# Transient "Error creating Routine / Model not found"? From this dir:
#   ./force-rerunning-model-creation-scripts.sh && terraform apply
```

Load data so the agent has OPEN incidents to act on:

```bash
cd ../.. && ./upload-batch.sh data/batch-1.txt
```

```sql
-- in BigQuery Studio
CALL bus_stop_image_processing.process_images();
CALL bus_stop_image_processing.update_incidents();
CALL bus_stop_image_processing.generate_synthetic_ridership();
```

Start the agent locally:

```bash
cd agents/maintenance-scheduler
pip3 install poetry && poetry install && poetry env activate
cp .env_sample .env        # set project id; set show_thought=True for the demo
export GOOGLE_CLOUD_PROJECT=core-demos
export GOOGLE_GENAI_USE_VERTEXAI=1
export GOOGLE_CLOUD_LOCATION=us-central1
adk web                    # open the dev UI → pick "maintenance_scheduler"
```

**Verify before you trust it:**

- [ ] `incidents` has rows with status `OPEN`
- [ ] the forecast tool returns numbers (not empty)
- [ ] the agent answers *“Check if any bus stops require maintenance”* locally

### Optional finale prep — Agentspace (needs extra time)

```bash
# from agents/maintenance-scheduler
poetry build --format=wheel --output=deployment
cd deployment && python deploy.py
#   → copy the printed resource name into GOOGLE_AGENT_RESOURCE_NAME in .env
#   → create an Agentspace app, copy its app id into AGENTSPACE_APP_ID in .env
cd .. && ./register-with-agentspace.sh
```

The agent then appears in the Agentspace app under **Agents → Bus Stop Maintenance Scheduler**.

-----

## 🎬 PRE-FLIGHT — 5 min before you go on

- [ ] BigQuery Studio open; **all queries pre-pasted into tabs** (don’t type live)
- [ ] ADK dev UI already running (`maintenance_scheduler` selected; `show_thought` on)
- [ ] Agentspace app tab open (if doing the finale)
- [ ] **Run every query + the agent’s first prompt once** — the first model call is cold/slow
- [ ] Safety net ready: the repo’s “incident detection in action” GIF + 3–4 screenshots of agent output

-----

## 🗣️ ON STAGE — script & timings (~14 min)

**0:00–0:30 — Cold open**
“A city puts cheap cameras on its buses. Photos of bus stops land in Cloud Storage. Watch what happens between the photo landing and a crew getting dispatched — with no ML training.”

**0:30–5:30 — Act 1: Data → Insight** *(BigQuery Studio)*

1. `SELECT * FROM bus_stop_image_processing.images LIMIT 5` — “just JPEGs in a bucket, now a queryable table.”
1. Upload a dirty stop live:
   `./copy-image.sh data/bus-stop-1-dirty.jpeg bus-stop-1-dirty.jpeg stop-1`
1. `CALL bus_stop_image_processing.process_images();` then
   `SELECT cleanliness_level, description FROM bus_stop_image_processing.image_reports LIMIT 5`
   — Gemini’s verdict as structured JSON **and** plain English. *(Slides 7–10, live.)*
1. `SELECT * FROM bus_stop_image_processing.incidents` — it auto-opened an incident. Zero humans so far.
1. *(Optional wow)* `SELECT * FROM bus_stop_image_processing.semantic_search_text_embeddings("a bus stop with broken glass") ORDER BY distance`
   — search images by meaning, even if those exact words never appear.

**5:30–11:30 — Act 2: Insight → Action** *(ADK dev UI, `show_thought` ON)*

1. Type: **“Check if there are any bus stops which require maintenance.”**
   → it lists stops with descriptions and asks whether to schedule.
1. Reply **“Yes”** → it pulls the ridership forecast, prioritizes by traffic, targets low-traffic working hours, proposes a time.
1. Reply **“Yes”** → it schedules, then moves to the next stop.
1. **Money shot:** “Show the email notification generated for bus stop 5” → it drafts the crew dispatch email.
   Bonus prompts: *“What is the URL of the image of bus stop #7?”* · *“Describe yourself.”*

**11:30–13:00 — Finale: Agentspace** *(optional)*
Open the Agentspace app → **Agents → Bus Stop Maintenance Scheduler** → run the same first prompt.
“Same agent, same code — now a business user drives it from a polished UI.”

**13:00–14:00 — Close**
“Photo → structured insight → an agent that prioritizes and dispatches. Gemini + SQL + ADK, deployable to Agent Engine and Gemini Enterprise unchanged.” Land on slide 14.

**10-minute cut:** drop Act 1 step 5 and the Agentspace finale.

-----

## 🎯 Mixed-audience cues

- **Business:** keep eyes on outcomes — photo → verdict → prioritized dispatch + drafted email. Plain-English rules, no data science required.
- **Dev:** show the `show_thought` reasoning and the tool-call trace; note it runs on plain natural-language instructions (not fine-tuned), combines BigQuery TimesFM forecasting with Gemini, and deploys unchanged to Agent Engine + Agentspace.

## 🛟 No-infra fallback

Run only the **Part 1 CleanSight notebook** in BigQuery Studio (project + bucket only — no Terraform). It delivers all of Act 1. Narrate the agent from the README transcript + the agent-workflow diagram.

## ⚠️ Gotchas

- **Forecast returns nothing** → ridership data is stale:
  `TRUNCATE TABLE bus_stop_image_processing.bus_ridership; CALL bus_stop_image_processing.generate_synthetic_ridership();`
- **LLMs are non-deterministic** — the agent may pick a different priority than you expect. Don’t over-rehearse one exact path.
- Re-running `terraform apply` **re-disables the scheduler**; manual `CALL`s keep timing in your control.
- **Cleanup after:** `terraform -chdir infrastructure/terraform destroy`

-----

*Built from the official `GoogleCloudPlatform/data-to-ai` repo (CleanSight + maintenance-scheduler agent).*
