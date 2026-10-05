# deploy-manifest

Declarative deploy orchestrator for the progmise APIs — the minimal equivalent
of Gluon/OAM. `manifest.yml` declares a **release**: a version plus the set of
components (repo + tag + dependencies) to deploy together.

## Flow

```mermaid
flowchart LR
    PR["PR bumps manifest.yml"] -->|"ci.yml → orch-ci"| V["Validate:<br/>schema, semver, tags,<br/>Docker images exist"]
    M["merge → main"] -->|"release.yml → orch-release"| R["draft GitHub<br/>Release v&lt;version&gt;<br/>+ mermaid graph"]
    R --> P["publish release<br/>= approval"]
    P -->|"deploy.yml → orch-deploy"| D["topo-sort by needs<br/>→ dispatch api-deploy<br/>per component, level by level"]
```

1. **PR** that bumps `version` and adjusts `components` → CI validates the
   manifest and renders the deploy plan.
2. **Merge to main** → a *draft* release `v<version>` is created.
3. **Publish the release** to approve it.
4. **Run Deploy** (Actions → Deploy) with `version` + `environment`
   (`pro`/`cert`/`pre` — must be in `vars.DEPLOY_ENVIRONMENTS`). Components
   deploy level by level following `needs`; each repo's `deploy.yml`
   (`api-deploy`) does the actual Vercel deploy.

## Setup

- Secret `ORCHESTRATOR_TOKEN`: PAT (fine-grained) with **Actions: write** on
  every repo listed in `manifest.yml` — needed to dispatch their deploys.
- Var `DEPLOY_ENVIRONMENTS` (JSON, default `["pro"]`), `DOCKER_USERNAME`.
- Optional `GRAFANA_OTLP_ENDPOINT`/`GRAFANA_OTLP_AUTH` for tracing.

See `reusable-workflows/docs/orchestrator.md` for the design.
