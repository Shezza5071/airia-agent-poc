# Airia agent dev → UAT → prod POC

A working, live-tested 3-environment promotion pipeline: one push to `main`
runs a **single linked GitHub Actions workflow** that deploys to Dev
automatically, then pauses for human approval before UAT, then pauses for
human approval before Production. Each stage targets its own real Airia
project.

Everything in this repo has been run against a real tenant (ANGLICARE SA) —
the `PipelineImport/definition` call, the upsert-by-name lookup, and the
`example-agent` pipeline itself. This is not a guess-and-hope POC.

## The 3 Airia projects (already created)

| Env | Project | Project ID |
|---|---|---|
| Dev | POC - Agent CI Dev | `01a0622f-d255-7206-af53-52b53bab66f8` |
| UAT | POC - Agent CI UAT | `01a0622f-d599-7de5-894f-9f1336e64ec3` |
| Prod | POC - Agent CI Prod | `01a0622f-d626-7082-9d7a-5a989e64ccb8` |

Tenant ID: `d382097e-7d7d-4a6b-b70f-c8c57dbea90f`. All three created via the
real `POST /v1/Project` API (not the console) on 2026-09-02, no budget cap
set (see below). `example-agent` is already live in the Dev project — it was
imported and test-executed for real (`POST /v2/PipelineExecution/{id}`
returned `{"result":"yes",...}`) before this repo was ever pushed.

## How the linking + gates work

`.github/workflows/deploy.yml` has three jobs — `deploy-dev`, `deploy-uat`,
`deploy-prod` — chained with `needs:` so they always run in order. Each job's
`environment:` does double duty:

- it's where that stage's Airia project id + API key live (GitHub
  Environment secrets), so Dev/UAT/Prod targeting can never leak into each
  other
- it's where the approval gate lives — add **Required reviewers** to an
  Environment in `Settings → Environments` and the job for that stage
  literally will not start until someone approves it in the Actions tab

`development` ships with no protection rules (auto-deploys on every push).
`production` has a required reviewer. Add one to `uat` too if you want a gate
before UAT as well.

## How a deploy actually works (real, verified API)

`.github/actions/promote-to-airia` does an **upsert by name within the
target project**:

1. `GET /v1/PipelinesConfig?projectId=<env's project>` and look for an
   existing agent whose `name` matches `agents/<x>/pipelineDefinition.json`'s
   `agent.name`.
2. If found: `PUT /v1/PipelineImport/definition/{existingId}` (update in
   place — this is how deploying agent v2 to a project that already has v1
   works, based on Dev/UAT/Prod being *separate* projects each with their own
   independent copy of the agent).
3. If not found: `POST /v1/PipelineImport/definition` (first deploy to that
   project).

Auth is `X-API-Key: <key>` (not `Authorization: Bearer` — that header from an
earlier version of this action was wrong, reverse-engineered from a
third-party repo instead of Airia's real API). Base host is
`https://prodaus.api.airia.ai` for the ANGLICARE SA tenant specifically —
this is tenant-scoped, not a generic Airia URL.

## Setup checklist (projects already exist — just wire GitHub)

**1. In GitHub — create three Environments**
`Settings → Environments → New environment`
- `development` — no protection rules needed
- `uat` — optionally add required reviewers if you want a gate before UAT too
- `production` — add a **Required reviewer** under Deployment protection
  rules. This one setting is the entire approval gate for that stage;
  nothing else in this repo enforces it

**2. Add secrets to each Environment** (not repo-level — this keeps
Dev/UAT/Prod targeting from leaking into each other, even though this POC
currently reuses one shared API key across all three — see note below)

| Secret | Dev | UAT | Prod |
|---|---|---|---|
| `AIRIA_API_TOKEN` | same key in all three (see note) | | |
| `AIRIA_API_ENDPOINT` | `https://prodaus.api.airia.ai` (same in all three) | | |
| `AIRIA_PROJECT_ID` | `01a0622f-d255-7206-af53-52b53bab66f8` | `01a0622f-d599-7de5-894f-9f1336e64ec3` | `01a0622f-d626-7082-9d7a-5a989e64ccb8` |

**On credential isolation:** Airia has no API to mint a new API key (that's
console-only, unlike Project creation) — so this POC deliberately reuses one
tenant-level key across all three GitHub Environments, and gets isolation
from `AIRIA_PROJECT_ID` differing per Environment + the key only ever having
access to its own tenant. For genuinely separate per-environment credentials,
create 3 named keys by hand in Airia's console (Settings → API Keys) and put
a different one in each Environment's `AIRIA_API_TOKEN` — no code change
needed, the action already takes the key as a plain input.

**3. Try it**
Push a change to `agents/example-agent/pipelineDefinition.json` on `main` (or
run the workflow manually with `workflow_dispatch`). `deploy-dev` runs
immediately (upserting the already-live agent, so this becomes a v1.01-style
update, not a fresh create); `deploy-uat` starts once it succeeds; `deploy-prod`
starts once `deploy-uat` succeeds, pausing for the required reviewer.

## Data source binding — real mechanism found, not wired up yet

Per your choice this session, `example-agent` doesn't use a data source, so
this isn't wired into the action. When you add one: Airia has a real,
verified endpoint for exactly this —
`PATCH /v1/PipelineImport/library/{importedAgentId}/step/{stepId}/datasource/{datasourceId}`
— call it once per environment, right after the promote step, passing that
environment's own data source id (from a `AIRIA_DATASOURCE_ID` Environment
secret) and the datasource step's id from the pipeline definition. Creating
the data source *connector* itself is still console-only (no API for that).
An earlier version of this README described a text-token-substitution
workaround for this — that was a guess made before this endpoint was found
and should not be used; this real PATCH call is a better fit.

## Budget cap — real API exists, not enabled in this POC by choice

Correcting an earlier claim in this README: Airia's `Project` entity **does**
have real, API-settable budget fields — confirmed directly from the
`ProjectDto` schema: `budgetAmount` (float), `budgetPeriod` (int enum),
`budgetAlert` (int, alert threshold), `budgetStop` (bool — blocks execution
at the cap). All three projects above were created with these left `null`/
`false` (no cap), per your choice this session. To turn one on:
`PUT /v1/Project/{id}` with `budgetAmount`, `budgetPeriod`, and
`budgetStop: true` set. `budgetPeriod`'s exact integer-to-period mapping
wasn't confirmed live (no need to, since it's unused here) — check
`GET /v1/Project/{id}` after a manual console-side cap is set once, to read
back the real enum value before scripting it.

## Evaluation — removed, was fabricated

An earlier version of this action called a guessed `AgentEvaluation` POST
endpoint that doesn't match the real API. The real `AgentEvaluation` surface
is a full evaluation-job feature (`POST /v1/AgentEvaluation` to create a job
against a dataset, `POST /v1/AgentEvaluation/{id}/run` to run it) — a bigger
feature than a one-line CI step, and out of scope for this POC. Not
reintroduced here; ask for it explicitly if you want automated eval wired
into `deploy-uat`/`deploy-prod`.
