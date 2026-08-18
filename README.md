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
`tenants/<app>/identity.yaml` (one level deep) is the per-app, per-cluster tenant
identity — consumed by *two different repos'* ApplicationSets. `tenants/<app>/<env>/
identity.yaml` (two levels deep) is deployment onboarding, and never actually appears
in this repo at all - see its own entry below.

```
tenants/
  <app-name>/
    xr-requests/         # Real onboarding entry point (idp/docs/service-catalog-
                         # design.md §0) - the ACTUAL Bootstrap-tier XR manifests
                         # (NodeJSApplication, ApplicationEnvironment), committed here
                         # directly (by a human today, by Backstage eventually). Read
                         # by the xr-requests ApplicationSet, one Application per
                         # tenants/<app>/ directory (a `directories` generator, not a
                         # `files` one keyed on identity.yaml - identity.yaml doesn't
                         # exist yet for a brand-new app, since NodeJSApplication's own
                         # Composition is what writes it). This is how these XRs get
                         # created for real now - not `kubectl apply`.
    identity.yaml        # platformIdentity: {appName, type, gitopsRepoUrl, appRepoUrl,
                         # githubOwner, catalogNamespace} - the ONE per-app, per-cluster
                         # tenant identity file (2026-08-18: absorbed what used to be a
                         # separate app.yaml sitting right next to it - see "History"
                         # below). Read by TWO independent ApplicationSets: the
                         # tenant-appprojects ApplicationSet in THIS cluster's own
                         # gitops-cluster-dev (one per app, not per env) to build, per
                         # app: the upper-env AppProject (sourceRepos
                         # [idp-service-catalog, this app's own gitopsRepoUrl],
                         # destinations app-<appName>-* on this cluster), a SECOND,
                         # narrower AppProject for the dev-cluster-only self-service
                         # lower-env tier (sourceRepos [idp-service-catalog, this app's
                         # own appRepoUrl - never gitopsRepoUrl] - idp/docs/
                         # gitops-strategy.md §10), and a per-app ApplicationSet whose
                         # own git generator watches THIS app's own appRepoUrl for
                         # platform/envs/*.yaml (the mechanism that actually provisions
                         # a dev env's namespace, since ApplicationEnvironment below
                         # structurally never targets this cluster) — AND
                         # platform-cicd's OWN tenant-onboarding ApplicationSet (a
                         # completely different repo/ArgoCD-generator pairing -
                         # platform-cicd's chart just happens to read this same commit,
                         # via .Values.tenantsRepoUrl pointed here) to stand up the
                         # app's real Tekton pipeline: RBAC, CDEvents Triggers, a
                         # build-cache PVC. gitopsRepoUrl/appRepoUrl are bare - NO
                         # `.git` suffix - required for PaC's exact-string match
                         # against GitHub's webhook payload URL; ArgoCD's own
                         # consumers of these same values tolerate the bare form fine
                         # (it normalizes `.git` away when matching sourceRepos/
                         # repoURL). Written by NodeJSApplication's Composition
                         # (idp-service-catalog, 2026-08-16 - previously a dedicated
                         # platform-cicd-kind-dev-tenants repo, eliminated once live
                         # history showed it only ever held throwaway apps). See
                         # compositions/nodejsapplication/templates/render-github-
                         # resources/cicd-identity-yaml.yaml's own header in
                         # idp-service-catalog for the full reasoning.
    <env>/
      identity.yaml      # appName, gitopsRepoUrl, githubOwner, env - DEPLOYMENT
                         # onboarding, a completely different file from the one directly
                         # above despite the identical filename, and deliberately NOT
                         # nested under platformIdentity: (different consumer, no shared
                         # schema to keep in sync). Self-contained, deliberately
                         # duplicating tenants/<app>/identity.yaml's own gitopsRepoUrl/
                         # githubOwner rather than requiring an ApplicationSet matrix
                         # generator to join the two files. Read by the
                         # tenant-onboarding ApplicationSet (one per app+env) to render
                         # the real deployment Application via idp-service-catalog's
                         # idp-application chart. NEVER actually appears in THIS repo -
                         # ApplicationEnvironment.spec.cluster structurally rejects
                         # targeting kind-dev itself (§0's multi-cluster section), so
                         # this commit always lands in whichever upper-env cluster's
                         # own gitops-cluster-<cluster>-tenants repo the env targets
                         # instead. Documented here anyway since it's the same XR/
                         # Composition, just writing elsewhere.
```

`tenants/<app>/identity.yaml` is written by `NodeJSApplication`'s Composition
(`idp-service-catalog`, `compositions/nodejsapplication/`) — app-level, create-once,
same reasoning that XRD already uses for creating its `gitops-<app-name>` repo at
bootstrap time (see `idp/docs/service-catalog-design.md` Item 1/2). The SAME filename
at the SAME one-level depth also gets written onto every *other* upper-env cluster's
own `gitops-cluster-<cluster>-tenants` by `ApplicationEnvironment`'s Composition
instead (`cluster-identity-yaml.yaml`, not this repo — see
`gitops-cluster-kind-prod-tenants/README.md`) — one schema, every cluster.
`<env>/identity.yaml` is written by `ApplicationEnvironment`'s Composition too. All
follow the same shape `platform-cicd`'s own onboarding flow already established:
operator-owned, PR-reviewed (lighter bar than `gitops-cluster-dev` itself), minimal —
identity and repo URLs only, no live config.

### History: `app.yaml` retired (2026-08-18)

Until 2026-08-18, this repo had a SECOND, separate file, `tenants/<app>/app.yaml`
(flat: `appName`/`gitopsRepoUrl`/`appRepoUrl`/`githubOwner`, `.git`-suffixed URLs),
written by `NodeJSApplication` in the same commit as `identity.yaml`, read only by
this cluster's own `tenant-appprojects` ApplicationSet. Two files "almost identical,
one a subset of the other," sitting side by side in the same tenant directory — but
not byte-for-byte mergeable as-is, because the overlapping `gitopsRepoUrl`/
`appRepoUrl` fields deliberately held *different string values*: `identity.yaml`'s
copies were bare (no `.git`) because PaC needs an exact string match against GitHub's
webhook payload URL, which never has `.git` (a real bug, fixed 2026-08-16 — see
`idp-service-catalog`'s `cicd-identity-yaml.yaml` header). `app.yaml`'s copies kept
the `.git` suffix matching ArgoCD's own `repoURL` convention. Consolidated by
standardizing on ONE convention system-wide (bare, no `.git`, everywhere — ArgoCD
normalizes the suffix away when matching `sourceRepos`/`repoURL`, live-verified as
part of this same change) and retiring `app.yaml` entirely: `tenant-appprojects` now
reads the same `identity.yaml` platform-cicd's own ApplicationSet already read,
nested under `platformIdentity:` instead of flat fields.

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
`hack/values-kind-dev.yaml` repointed to match - end-to-end live-verified with a
throwaway app (`cicd-redirect-verify`, onboarded via a real `NodeJSApplication` XR
through `xr-requests/`, not a manual commit): `identity.yaml` landed here alongside
`app.yaml` in the same commit, `CicdOnboarded` flipped `True`, platform-cicd's
`tenant-onboarding` ApplicationSet in the `argocd` namespace picked it up from this repo
and stood up the real `cicd-redirect-verify-cicd` Application (RBAC, ExternalSecret,
build-cache PVC, ConfigMaps all correctly named). Torn down after.

**`xr-requests/` — built and live-verified 2026-08-15** (`idp/docs/
service-catalog-design.md` §0). A real throwaway app (`xr-onboarding-verify`) onboarded
end-to-end via nothing but git commits into this repo, across two passes, including a
full teardown each time. One real open bug: deleting an `ApplicationEnvironment`
xr-request deadlocks against the `Usage` it composes (ArgoCD prune vs. the `Usage`
controller's own finalizer wait) — confirmed twice, not yet fixed. See the `xr-requests`
`ApplicationSet`'s own header comment in `gitops-cluster-dev` for the full mechanism and
recovery steps.

**`app.yaml`'s lower-env tier (idp/docs/gitops-strategy.md §10) — built and
live-verified 2026-08-16.** `tenant-appprojects` now also renders a second AppProject
and a per-app ApplicationSet per app — this repo itself is unchanged by that (the
lower-env tier reads `platform/envs/*.yaml` live from each app's own src repo, never
from here), but `app.yaml`'s own consumer count went from one to three. Live-verified
end to end on the real `nodejs-demo-app` entry already in this repo: a real
`platform/envs/dev.yaml` commit to that app's own repo provisioned
`app-nodejs-demo-app-dev` for real, and the new AppProject's rejection boundary was
proven with two throwaway `Application`s. See `idp/docs/gitops-strategy.md` §10 for
the full mechanism, including three real bugs hit building it.

**`app.yaml` retired, `tenant-appprojects` moved onto `identity.yaml` — built and
live-verified 2026-08-18** (`idp-service-catalog` v0.3.21). Bumped the pinned
`idp-service-catalog` `Application` in `gitops-cluster-dev` to v0.3.21, synced it, and
force-triggered a reconcile — `NodeJSApplication` auto-deleted both `checkout-api/
app.yaml` and `nodejs-demo-app/app.yaml` from this repo on its own within seconds of
the new Composition landing (confirmed via `provider-github`'s own deletion commit,
still carrying the original "add ... app.yaml" `commitMessage` since deletes reuse
the last-applied value - no separate manual cleanup needed here, unlike
`gitops-cluster-kind-prod-tenants`, whose `app.yaml` writer deliberately excludes
`Delete`). `tenant-appprojects`' generator switch to `tenants/*/identity.yaml`
confirmed live: `checkout-api-onboarding`/`nodejs-demo-app-onboarding` both stayed
`Synced`/`Healthy` through the cutover (same `Application` names, no UID churn, since
`.platformIdentity.appName` renders identical to the old `.appName`), and the
regenerated `AppProject`s' `sourceRepos` came out bare (`https://github.com/jfillman/
gitops-checkout-api`, no `.git`) exactly as `identity.yaml` now writes them - the
`.git`-suffix-drop this whole consolidation depended on. **The load-bearing
assumption behind that drop - that ArgoCD tolerates a bare `sourceRepos` entry
against `Application` sources that may still carry `.git` - held up empirically: every
downstream `Application` (including ones pointed at the still-`.git`-suffixed
`idp-service-catalog.git` chart source) stayed `Synced`, no rejected sync, no
`PermissionDenied` from the `AppProject` boundary.** See `gitops-cluster-kind-prod-
tenants/README.md`'s own Status section for the equivalent upper-env-cluster
verification, including the one real orphaned-file cleanup that consolidation did
need by hand.
