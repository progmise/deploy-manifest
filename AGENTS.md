# AGENTS.md

Guide for working on **deploy-manifest** — the declarative deploy
orchestrator for the progmise platform (minimal Gluon/OAM equivalent).
`manifest.yml` declares a release: a version plus the set of components
(repo + tag + dependencies) deployed together.

## Golden rule

`manifest.yml` is the deployable projection of the platform — every field is
validated by `orch-ci` and consumed by `topo-deploy.py` in
`progmise/reusable-workflows` (`@v1`). Never store secrets here: any `*Id`
property **names** a credential/variable held in the *consumer repo's*
GitHub secrets or variables — values never live in this file.

## Schema (keep README.md in sync when it changes)

```yaml
version: 1.2.0            # semver — merge to main drafts release v<version>
environments:             # optional — declared deploy targets (OAM)
  - name: pro
    type: production
    infrastructures:
      - id: loans-api-pro
        type: vercel      # vercel | artifact-store | ...
        properties:       # free-form provider fields
          project: loans-api
          credentialsId: VERCEL_TOKEN     # names a consumer repo secret
          orgId: VERCEL_ORG_ID            # names a consumer repo variable
components:
  - name: loans-api
    repo: progmise/loans-api
    tag: "0.1.0"          # git tag + Docker Hub image must exist
    needs: []             # deploy-after deps (component names, acyclic)
    infra: [loans-api-pro]
```

CI rules: semver `version`, unique env/component/infra ids, `needs` ⊆
components (acyclic), each `repo:tag` exists, `infra` ⊆ infra ids. A PR that
registers a component **fails validation until its tag exists** — merge the
registration after the component's first release.

## Flow

PR bumping `manifest.yml` → `ci.yml` (`orch-ci` validates + renders plan) →
merge → `release.yml` (`orch-release` drafts Release `v<version>` + mermaid
graph) → **publishing the release approves it** → `deploy.yml` (`orch-deploy`
topo-sorts by `needs` and dispatches each consumer's `deploy.yml`, level by
level; needs `ORCHESTRATOR_TOKEN` PAT with `actions:write`).

The registration PR for a **new** component is opened automatically by
`deploy-orchestrator-api`'s provisioner — don't hand-edit those PRs' branch
content; review/merge after the repo's `0.1.0` tag exists.

## Conventions

- Single `main` branch (protected) — no `development`; work lands on
  `<type>/<snake_description>` + PR straight to `main`.
- Thin callers only in `.github/workflows/` — all logic lives in
  `progmise/reusable-workflows`; keep callers thin.
- `ORCHESTRATOR_TOKEN` is the only secret this repo needs (repo-level).
- No `.env.example` needed — nothing here reads env vars; credentials are
  referenced by name in `properties:*Id` only.

## Verify before done

Manifest changes are validated by CI on the PR (`orch-ci`): schema, semver,
unique names, acyclic `needs`, `repo:tag` existence. Locally you can run the
same validator via `topo-deploy.py validate` from `reusable-workflows`.
