# ThoughtSpot content promotion utility

A Streamlit app that promotes ThoughtSpot content (tables, models, liveboards, answers, and
optionally Spotter feedback and instructions) from a source environment to a target
environment, with a Git repository in between as the record of what was promoted.

It uses ThoughtSpot's REST API v2 (TML export, validate and import) and the GitHub API. Objects
are matched across environments by **obj_id**, so an update lands on the existing object instead
of creating a copy.

> This is a ThoughtSpot Professional Services utility, not a supported ThoughtSpot product.
> Try it on non-production content first.

## What a promotion does

The app walks through six pages.

1. **Select Assets.** Pick a team, fetch its content from the source, and tick what to promote.
   Ticking a liveboard or answer pulls in the model and tables it sits on.
2. **obj_id Setup.** Checks every object has an obj_id on the source, and that objects already on
   the target carry the same obj_id. Offers to set or align them.
3. **Source Audit.** Optional checks against the source warehouse: columns the source no longer
   has, data types that drifted, and column casing. Nothing changes until you approve it.
4. **TML Validation.** Re-points connections (and optionally database and schema names) to the
   target, then validates everything against the target without importing (`VALIDATE_ONLY`).
   If the promotion removes a column that something on the target still uses, ThoughtSpot refuses
   it and names the objects in the way. The app works out exactly which answers, liveboard tiles
   and model columns use that column and shows them for you to decide.
5. **Git Operations.** Commits the promoted TML to a `dev` branch, opens a pull request to `main`,
   then imports into the target and merges.
6. **Import Results.** What was created, updated in place, or failed, verified against the target.

## What it will not do without asking

- **Remove anything from the target.** Deleting an answer, removing a liveboard tile or stripping a
  column from a model is listed first, needs you to type `DELETE`, and saves the current TML of
  every affected object under `logs/snapshots/` before writing. Those saved files are the way
  back; nothing restores them automatically.
- **Drop a column silently.** A column on the target that the promotion would remove is listed,
  and the import button stays disabled until you acknowledge it.
- **Import a liveboard with a broken visualization.** Liveboards and answers are validated against
  the target after the models land. If a visualization fails, you choose to drop it or import the
  board without it.

## How the import behaves

- Tables and models import as **one all-or-nothing batch**. If ThoughtSpot refuses any of them,
  none are applied and the run stops there: no feedback, instructions, liveboards or answers are
  imported, and the pull request is not merged.
- Liveboards and answers that validated clean also import all-or-nothing.
- The pull request is merged only after tables and models are in, so `main` does not claim a
  deployment that failed.
- Spotter feedback and Spotter instructions are imported in separate calls after the models land.
  They are not part of the all-or-nothing batch.
- If you choose to import a board with an unticked broken visualization, that import is partial
  by design: ThoughtSpot imports the board and skips the visualization.

## Requirements

- Python 3.9 or later.
- ThoughtSpot with REST API v2 and obj_id support, for both source and target.
- A ThoughtSpot account on each side. On the target it needs the **Can manage data** privilege to
  import tables and models, and edit access to any content the promotion has to change.
- A GitHub repository for the promoted TML, with a `main` branch, and a token that can push to
  it. The app creates and reuses a `dev` branch in that repository.

## Setup

```bash
git clone <this repository>
cd <this repository>
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env              # then fill it in
```

### `.env`

| Setting | What it is |
|---|---|
| `TS_SOURCE_HOST`, `TS_TARGET_HOST` | Cluster URLs. They can be different clusters, or the same cluster with different orgs. |
| `TS_SOURCE_TOKEN`, `TS_TARGET_TOKEN` | A bearer token for each side (for example from trusted authentication). Takes precedence over username and password. |
| `TS_SOURCE_USERNAME` / `_PASSWORD`, `TS_TARGET_USERNAME` / `_PASSWORD` | Local login, only if you are not using tokens. Does not work on SSO-only clusters. |
| `TS_SOURCE_ORG`, `TS_TARGET_ORG` | Org id, if the content is not in the default org. |
| `GITHUB_TOKEN`, `GITHUB_REPO` | Token and `owner/repo` for the repository that holds promoted TML. |
| `TS_PROXY` | Optional HTTPS proxy. |
| `TS_SOURCE_DBX_HOST` / `_WAREHOUSE` / `_TOKEN`, and the `TS_TARGET_DBX_*` equivalents | Optional, Databricks only. Lets the app read warehouse column types and casing directly. If only the target values are set, source reads use the target warehouse. |

Bearer tokens expire. After changing `.env`, stop and restart the app; a browser refresh keeps the
old connection.

### Teams

`config/teams.json` holds the defaults. On first run the app copies it to
`config/teams.local.json` (ignored by Git), and that file is what the app uses and edits from then
on. Each team has:

| Field | What it is |
|---|---|
| `tags` | Source tags that scope the team's content. Empty means fetch everything you can see and tick by hand. |
| `source_connection`, `target_connection` | Connection names exactly as they appear on each side. Tables are re-pointed from one to the other. |
| `db_map`, `schema_map` | Optional database and schema renames for the target. |

The sidebar edits these for the selected team.

## Run

```bash
streamlit run app.py
```

## Files it writes

Everything goes under `logs/` (ignored by Git): validation runs, raw import errors, a record of
every change made on the target, and the TML snapshots taken before any change.

## Tests

```bash
pip install -r requirements-dev.txt
pytest
```
