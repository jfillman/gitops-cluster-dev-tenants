# gitops-cluster-dev-tenants

Which apps are onboarded to the `kind-dev` cluster — the churny per-app-onboarding half
split out of `gitops-cluster-dev` (`idp/docs/gitops-strategy.md` §1), so the rare,
high-stakes cluster-config changes in that repo don't get diluted by frequent,
individually low-stakes onboarding PRs.

Read by `gitops-cluster-dev/02-argocd-apps/`'s two `ApplicationSet`s — that's where the
generator logic and the full design rationale live (see that directory's own README).
This repo only holds the data they read.

## Layout

```
tenants/
  <app-name>/
    app.yaml            # appName, gitopsRepoUrl, githubOwner - one per app, shared
                         # across every env this app has on this cluster. Read by the
                         # tenant-appprojects ApplicationSet (one per app, not per env)
                         # to build that app's AppProject: sourceRepos [idp-service-
                         # catalog, this app's own gitopsRepoUrl], destinations scoped
                         # to app-<appName>-* namespaces on this cluster only.
    <env>/
      identity.yaml      # appName, gitopsRepoUrl, githubOwner, env - self-contained,
                         # deliberately duplicating app.yaml's fields rather than
                         # requiring an ApplicationSet matrix generator to join the two
                         # files. Read by the tenant-onboarding ApplicationSet (one per
                         # app+env) to render the real deployment Application via
                         # idp-service-catalog's idp-application chart.
```

`app.yaml` is written by `NodeJSApplication`'s Composition (`idp-service-catalog`,
`compositions/nodejsapplication/`) — app-level, create-once, same reasoning that XRD
already uses for creating its `gitops-<app-name>` repo at bootstrap time (see
`idp/docs/service-catalog-design.md` Item 1/2). `<env>/identity.yaml` is written by
`ApplicationEnvironment`'s Composition (that XRD doesn't exist yet). Both follow the same
shape `platform-cicd`'s own `tenants/<app>/identity.yaml` onboarding flow already uses:
operator-owned, PR-reviewed (lighter bar than `gitops-cluster-dev` itself), minimal —
identity and repo URLs only, no live config.

## Status

Bootstrapped 2026-08-13 alongside `gitops-cluster-dev/02-argocd-apps/`'s two
ApplicationSets — real infrastructure, live-verified at the time with a throwaway
`test-app` entry, since removed. `app.yaml` is now written for real by `NodeJSApplication`
(built 2026-08-13, see `idp-service-catalog/xrds/nodejsapplication.yaml`);
`ApplicationEnvironment` (and therefore `<env>/identity.yaml`) still isn't built.
