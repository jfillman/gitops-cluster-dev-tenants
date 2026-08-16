# gitops-cluster-dev-tenants

Which apps are onboarded to the `kind-dev` cluster — the churny per-app-onboarding half
split out of `gitops-cluster-dev` (`idp/docs/gitops-strategy.md` §1), so the rare,
high-stakes cluster-config changes in that repo don't get diluted by frequent,
individually low-stakes onboarding PRs.

Read by `gitops-cluster-dev/02-argocd-apps/`'s three `ApplicationSet`s — that's where the
generator logic and the full design rationale live (see that directory's own README).
This repo only holds the data they read and receive.

## Layout

**Two files are both named `identity.yaml` but live at different depths and mean
completely different things — read this before assuming which one a path refers to.**
`tenants/<app>/identity.yaml` (one level deep) is CICD pipeline onboarding, consumed by
a *different repo's* ApplicationSet entirely. `tenants/<app>/<env>/identity.yaml` (two
levels deep) is deployment onboarding, and never actually appears in this repo at all -
see its own entry below.

```
tenants/
  <app-name>/
    xr-requests/         # Real onboarding entry point (idp/docs/service-catalog-
                         # design.md §0) - the ACTUAL Bootstrap-tier XR manifests
                         # (NodeJSApplication, ApplicationEnvironment), committed here
                         # directly (by a human today, by Backstage eventually). Read
                         # by the xr-requests ApplicationSet, one Application per
                         # tenants/<app>/ directory (a `directories` generator, not a
                         # `files` one keyed on app.yaml - app.yaml doesn't exist yet
                         # for a brand-new app, since NodeJSApplication's own
                         # Composition is what writes it). This is how these XRs get
                         # created for real now - not `kubectl apply`.
    app.yaml            # appName, gitopsRepoUrl, appRepoUrl, githubOwner - one per app,
                         # shared across every env this app has on this cluster. Read by
                         # the tenant-appprojects ApplicationSet (one per app, not per
                         # env) to build that app's AppProject: sourceRepos
                         # [idp-service-catalog, this app's own gitopsRepoUrl],
                         # destinations scoped to app-<appName>-* namespaces on this
                         # cluster only.
    identity.yaml        # platformIdentity: {appName, type, gitopsRepoUrl, appRepoUrl,
                         # githubOwner, catalogNamespace} - CICD PIPELINE onboarding,
                         # NOT deployment (see <env>/identity.yaml below for that). Read
                         # by platform-cicd's OWN tenant-onboarding ApplicationSet
                         # (a completely different repo/ArgoCD-generator pairing than
                         # this repo's own three ApplicationSets - platform-cicd's chart
                         # just happens to read this same commit, via
                         # .Values.tenantsRepoUrl pointed here) to stand up the app's
                         # real Tekton pipeline: RBAC, CDEvents Triggers, a build-cache
                         # PVC. Written by NodeJSApplication's Composition, same commit
                         # as app.yaml above (idp-service-catalog, 2026-08-16 -
                         # previously a dedicated platform-cicd-kind-dev-tenants repo,
                         # eliminated once live history showed it only ever held
                         # throwaway apps). See
                         # compositions/nodejsapplication/templates/render-github-
                         # resources/cicd-identity-yaml.yaml's own header in
                         # idp-service-catalog for the full reasoning.
    <env>/
      identity.yaml      # appName, gitopsRepoUrl, githubOwner, env - DEPLOYMENT
                         # onboarding, a completely different file from the one directly
                         # above despite the identical filename. Self-contained,
                         # deliberately duplicating app.yaml's fields rather than
                         # requiring an ApplicationSet matrix generator to join the two
                         # files. Read by the tenant-onboarding ApplicationSet (one per
                         # app+env) to render the real deployment Application via
                         # idp-service-catalog's idp-application chart. NEVER actually
                         # appears in THIS repo - ApplicationEnvironment.spec.cluster
                         # structurally rejects targeting kind-dev itself (§0's
                         # multi-cluster section), so this commit always lands in
                         # whichever upper-env cluster's own
                         # gitops-cluster-<cluster>-tenants repo the env targets
                         # instead. Documented here anyway since it's the same XR/
                         # Composition, just writing elsewhere.
```

`app.yaml` and `tenants/<app>/identity.yaml` are both written by `NodeJSApplication`'s
Composition (`idp-service-catalog`, `compositions/nodejsapplication/`), in the same
commit — app-level, create-once, same reasoning that XRD already uses for creating its
`gitops-<app-name>` repo at bootstrap time (see `idp/docs/service-catalog-design.md`
Item 1/2). `<env>/identity.yaml` is written by `ApplicationEnvironment`'s Composition.
All three follow the same shape `platform-cicd`'s own onboarding flow already uses:
operator-owned, PR-reviewed (lighter bar than `gitops-cluster-dev` itself), minimal —
identity and repo URLs only, no live config.

## Status

Bootstrapped 2026-08-13 alongside `gitops-cluster-dev/02-argocd-apps/`'s two
ApplicationSets — real infrastructure, live-verified at the time with a throwaway
`test-app` entry, since removed. `app.yaml` is now written for real by `NodeJSApplication`
(built 2026-08-13, see `idp-service-catalog/xrds/nodejsapplication.yaml`);
`<env>/identity.yaml` is now written for real by `ApplicationEnvironment` (built
2026-08-15, see `idp-service-catalog/xrds/applicationenvironment.yaml`) — matches
the shape documented above exactly (`appName`/`gitopsRepoUrl`/`githubOwner`/`env`).

**`tenants/<app>/identity.yaml` (CICD onboarding) added 2026-08-16.** Previously
`NodeJSApplication` committed this file into a dedicated `platform-cicd-kind-dev-
tenants` repo instead - eliminated once live history showed that repo only ever held
throwaway verification apps and both `platform-cicd/docs/onboarding.md` and
`idp/README.md` confirmed `kind-dev`'s platform-cicd instance is idp-exclusive (unlike
`kind-observe`, which hosts real, independent, non-idp tenants and keeps its own
separate tenants list). `idp-service-catalog` v0.3.5, `platform-cicd`'s own
`hack/values-kind-dev.yaml` repointed to match - end-to-end live verification with a
throwaway app pending.

**`xr-requests/` — built and live-verified 2026-08-15** (`idp/docs/
service-catalog-design.md` §0). A real throwaway app (`xr-onboarding-verify`) onboarded
end-to-end via nothing but git commits into this repo, across two passes, including a
full teardown each time. One real open bug: deleting an `ApplicationEnvironment`
xr-request deadlocks against the `Usage` it composes (ArgoCD prune vs. the `Usage`
controller's own finalizer wait) — confirmed twice, not yet fixed. See the `xr-requests`
`ApplicationSet`'s own header comment in `gitops-cluster-dev` for the full mechanism and
recovery steps.
