# PLAN.md

> **Status: v8. Phase 0 in progress**: one step left, Conditional Access on (incl. short sessions, D39). Standing admin access is gone. v8 adds the release flow to prod (D37, D38).
> Working rules live in `AGENTS.md`. History lives in `.prompts/`.

## 1. Goal

A showcase todo application built entirely with AI that is **infrastructure-first and AI-first**:

- Humans and agents can use every feature equally well, under the same access control.
- Every layer (frontend, backend, database, infrastructure) is locked down as close to zero trust as we can get, and still lets an authorized agent read traces, logs and metrics end to end.
- Features: create, list, get, edit and delete todos, then share a todo with other people by email using permission policies.
- The app is **semi-public**: anyone can sign up, no organization membership needed.
- **Cost: as cheap as is feasible and normal, but security comes first** (see §9).

## 2. Repositories and governance

| Repo | Contains | Who can merge |
|---|---|---|
| `Ulberg/todo` (this repo) | F# backend, React frontend, tests, app CI, **`platform.yaml`** | `todo` lead developers |
| `Ulberg/platform` (new) | Pulumi in **F#** (Azure + Entra + GitHub), ArgoCD apps, Helm values, cluster policies, observability stack, the `platform.yaml` schema + renderer, **all access definitions** | `platform` lead developers (a subset of infra developers) |

### 2.1 `platform.yaml`: the contract between app and platform

The app repo *declares* what it needs. The platform repo *decides* how (or whether) to provide it. App developers never get credentials, cluster access or deploy rights.

```yaml
# todo/platform.yaml (illustrative, the schema lives in the platform repo)
apiVersion: platform/v1
app: todo
services:
  - name: backend
    image: todo-backend        # tag is supplied by CI, never written here by hand
    port: 8080
    replicas: { min: 2, max: 4 }
    resources: { cpu: 250m, memory: 256Mi }
    health: { live: /health/live, ready: /health/ready }
    routes: [ /api, /mcp ]
    database: todo             # only services that list the DB get network + identity access to it
  - name: frontend
    image: todo-frontend
    port: 8080
    replicas: { min: 2, max: 2 }
    routes: [ / ]
databases:
  todo: { engine: postgres }
auth:
  spa: { redirectPaths: [ /auth/callback ] }
  api: { scopes: [ todos.read, todos.write, todos.share ] }
egress: []                     # anything outside the cluster has to be listed
```

Flow, from change to prod (D37):
1. **Change.** An app developer opens a PR in `todo`. The app's repo lead approves it as code owner, and it merges to `main`.
2. **Build.** App CI validates `platform.yaml` against the schema, builds and signs the images, and pushes them to ACR.
3. **Deploy to dev.** A GitHub App with **only** "open PR" rights on the platform repo (`ulberg-platform-pr-bot`) opens a PR that sets the new image tags in `apps/todo/dev/` together with the current `platform.yaml`. Platform CI checks it against policy.
   - **Image-tag-only change that passes policy → auto-merge**, and ArgoCD syncs it into dev. Tags are GitOps manifests, not Pulumi, so no environment approval is involved.
   - **Any `platform.yaml` change → approval from a platform lead.** If it needs Azure resources, the Pulumi apply also waits for a platform lead (step 5).
4. **Release to prod.** When the dev deploy is healthy and the post-deploy smoke tests pass, the pr-bot opens (or updates) one **release PR** per app that copies exactly those tags to `apps/todo/prod/`.
   - Policy check: the tags must be the ones running healthy in dev, and the images must be signed.
   - **The app's repo leads approve it** (they are the code owners of `apps/todo/prod/` tags). The merge is the release: ArgoCD syncs prod.
   - A release PR that changes anything besides tags also needs a platform lead.
   - Rollback is a release PR with the previous tags, with the same approval.
5. **Platform CI runs `pulumi up` only after approval.** Each environment's apply job runs in a GitHub **Environment** (`dev`, later `prod`) whose required reviewers are the platform leads. The Azure federated credential trusts *only* `repo:Ulberg@<owner-id>/platform@<repo-id>:environment:<env>` (GitHub's subject carries immutable IDs, so a renamed or recreated repo doesn't match), so a workflow without that approval can't even get an Azure token. PR previews use a separate read-only identity. (This is the reference-infra OIDC pattern, scoped tighter.) Nobody applies from a laptop.

How it's enforced:
- The platform repo's `CODEOWNERS`, generated from `people.yaml` and checked by the `access` stack (§2.2):
  - `apps/<app>/dev/` tags have no owner;
  - `apps/<app>/prod/` tags are owned by that app's repo leads;
  - everything else is owned by the platform leads.
- The ruleset requires code-owner review plus the policy check, with 0 general approvals. No app and no bot gets a ruleset bypass.
- GitHub Apps:
  - `ulberg-platform-admin`: the apply job only, in the `dev` and `prod` environments;
  - `ulberg-platform-reader`: PR previews, read-only;
  - `ulberg-platform-pr-bot`: deploy PRs from todo `main`, and release PRs.

### 2.2 Access defined as code (IaC)

All people and roles live in the platform repo, in one place:

```yaml
# platform/access/people.yaml (illustrative)
people:
  endre: { github: Ulberg, entra: <object-id> }
groups:
  infra-developers: [ endre ]   # Azure/K8s/DB read, infra logs, infra-dev Claude
  app-developers:   [ endre ]   # app logs + metrics + traces only
repo-leads:
  platform: [ endre ]           # may approve/merge in Ulberg/platform
  todo:     [ endre ]           # may approve/merge in Ulberg/todo
```

Pulumi (F#) turns this into:
- Entra security groups + memberships (workforce tenant), which feed the Azure RBAC, AKS RBAC, Postgres and Grafana org mappings.
- GitHub (Pulumi GitHub provider):
  - repository settings and branch rulesets: PR required, code-owner review required, signed commits, no force push, status checks required;
  - a `CODEOWNERS` file in each repo (`* @<leads>`). It is committed by hand and Pulumi fails the run if it doesn't match `people.yaml`, because a bot pushing it to `main` would need a ruleset bypass;
  - collaborator permissions (leads get `maintain`, other developers get `write`, no one gets `admin` except the bootstrap owner).
- Changing who is a lead or an infra developer is a PR to `platform/access/` that needs a platform lead's approval.

**GitHub constraint**: `Ulberg` is a personal account, not an organization. That means:
- no GitHub Teams, so `CODEOWNERS` lists users directly;
- branch rulesets / required reviews on *private* repos need GitHub Pro.

So **both repos are public** (D26). gitleaks secret scanning runs in CI as a required check, GitHub push protection and secret scanning are on, and Dependabot handles updates and alerts. **No git pre-commit hooks** (D29).

## 3. Decisions made

| # | Decision | Source |
|---|---|---|
| D1 | `.prompts/` holds one `.txt` per message under `.prompts/<date>/` | Phase 0 |
| D2 | `CLAUDE.md` only imports `AGENTS.md`, so there is one source of truth | Phase 0 |
| D3 | Agent interface is **MCP**. REST and MCP are both thin adapters over the same use-case functions | msg 2 |
| D4 | App users sign in with **Entra External ID** (anyone can sign up). Admins/developers use the **workforce tenant** | msg 2 |
| D5 | Separate **platform repo**, driven by `platform.yaml` (§2.1) | msg 2 |
| D6 | **SPA + MSAL (PKCE), bearer token in `Authorization` header**. No auth cookies, strict CORS | msg 2 |
| D7 | Cheapest *normal* setup with security first (§9). Starts with one `dev` environment; `prod` is added after Phase 6 (D38) | msg 2–3 |
| D8 | Features: create, list, get, edit, delete | msg 2 |
| D9 | Access matrix per group (§5). App devs' Claude gets **app logs + metrics + traces** | msg 2–3 |
| D10 | **No HTTP `QUERY`**; `GET` + query params for reads | msg 2 |
| D11 | Sharing follows the Unix/Linux permission model (§6) | msg 2–3 |
| D12 | Traces in **Tempo**; OTel Collector → Loki / Prometheus / Tempo → Grafana | msg 2 |
| D13 | **Valkey and Keycloak deferred** until a trigger in §8 happens | msg 2 |
| D14 | Local development in three tiers (§11) | msg 2–3 |
| D15 | Repo leads + `CODEOWNERS` + branch rules defined as code in the platform repo (§2.2) | msg 3 |
| D16 | **No Postgres RLS in v1**; it is future work (§13) | msg 3 |
| D17 | The database is private: **no public endpoint; only backend pods can reach it** (§9.1) | msg 3 |
| D18 | Infra is applied **only by platform CI after approval** | msg 3 |
| D19 | **Pulumi in F#** (Pulumi has no official Rust SDK) | msg 3 |
| D20 | No custom domain for now | msg 3 |
| D21 | GitHub owner: **`Ulberg`** (personal account) | msg 3 |
| D22 | Sharing never reveals whether a user exists (§6.3) | msg 3 |
| D23 | **Entra ID P2 + PIM is implemented** for infra developers (§7) | msg 4 |
| D24 | **DB auth option B**: Entra-only, split migrator / app / read-only identities, no passwords (§9.2) | msg 4 |
| D25 | A write-only grant appears in the grantee's list as `{ id, permissions }` with no content | msg 4 |
| D26 | **Public repos**, with secret scanning (gitleaks in CI, GitHub push protection + secret scanning, Dependabot) | msg 4–5 |
| D27 | No "leave a share" in v1; it is future work | msg 4 |
| D28 | Azure bootstrap is done by you, following a checklist + script (§4) | msg 4 |
| D29 | **No git pre-commit hooks.** Every check runs in CI (Claude Code hooks for `.prompts/` are not git hooks and are unaffected) | msg 5 |
| D30 | `reference-infra/` is a read-only pattern source like `reference/`; both are git-ignored and never pushed | msg 5 |
| D31 | Adopted reference-infra patterns (§12): nested read ⊂ write ⊂ admin groups, generated RBAC, environment-scoped OIDC, External Secrets Operator, cert-manager | msg 5 |
| D32 | **No service mesh in v1**; add one when a trigger in §8 happens | msg 6 |
| D33 | Infra devs reach the DB through a `db-access` proxy pod, not Teleport (§4.1) | msg 6 |
| D34 | **Bootstrap exception:** while you are the only person (infra dev, admin, app dev and lead of both repos), commits go straight to `main` without required review. GitHub doesn't allow approving your own PR. This is temporary; see the TODO in §13 | msg 7 |
| D35 | The owner Microsoft account keeps only its billing role: no Global Admin and no subscription Owner, not even eligible | msg 2026-11-09 |
| D36 | Subscription **Owner only via the break-glass accounts**. `platform-admin` (PIM, 1 h) gets no Owner | msg 2026-11-09 |
| D37 | **Release flow** (§2.1). A PR approved by the app lead merges to app `main`; tag-only changes auto-deploy to `dev`; a release PR copies the dev tags to `prod`, and **the app lead approves it** | msg 2026-11-09 |
| D38 | **`prod` gets its own AKS cluster, Postgres, VNet and Key Vault** (own resource groups, same subscription). It is built after Phase 6, once the solo-dev exception (D34) is gone | msg 2026-11-09 |
| D39 | **Short sessions** (Conditional Access, workforce tenant only): a fresh phishing-resistant sign-in for **every** PIM activation (authentication context); infra developers sign in to Azure portal/CLI/ARM again every **4 h**, with no persistent browser session; all developers sign in to everything else again every **12 h**. App users (External ID tenant) are not affected | msg 2026-11-09 |
| D40 | **Privilege alerts**: PIM emails every activation and every new assignment of Global Admin, `platform-write` and `platform-admin` to the platform alert list (GitHub secret `PLATFORM_ALERT_EMAILS`, kept out of the public repos). Log-based alerts (break-glass sign-ins, role changes not made by the apply identity, PIM/CA policy changes) come with Phase 5 | msg 2026-11-09 |

## 4. Sign-off and bootstrap

### 4.1 Database access path for infra devs (D33)

The DB only accepts traffic from backend pods, so infra devs and their Claude reach it through a **`db-access` proxy pod** in its own namespace:
- only `infra-developers` get Kubernetes RBAC to `port-forward` to it;
- only that pod and the backend may reach Postgres;
- Postgres still requires the user's own Entra token, mapped to the read-only role;
- the AKS audit log records every port-forward.

Teleport was considered (it's in the reference) but needs its paid Enterprise edition for Entra SSO. Without it there is no session recording; `pgaudit` (future work) covers that.

### 4.2 Sign-off

**To sign off**, confirm the phase order in §10. Then I start Phase 0. Creating the two public GitHub repos and pushing to them is the first outward-facing step, and I'll do it as soon as you sign off.

**Azure bootstrap steps for you** (they need your payment details and Global Admin rights; I'll write them up as a checklist and a script in Phase 0):
1. Create an Azure account. This creates your workforce tenant + a pay-as-you-go subscription.
2. Set a budget alert (e.g. USD 200/month).
3. Buy Entra ID P2 licenses for the infra developers (one, for you, to start with).
4. Create 2 emergency-access accounts with FIDO2 keys, kept offline.
5. Create the Entra External ID tenant (for app users) and link it to the subscription.
6. Run the bootstrap script: Pulumi state storage + CI identity with GitHub OIDC trust.

Everything after that, including PIM settings and Conditional Access, is Pulumi.

## 5. Identity and access

### 5.1 Access matrix

| Capability | Infra dev (human) | Infra dev's Claude | App dev (human) | App dev's Claude | App users / their agents |
|---|---|---|---|---|---|
| Change infra | Approve PR → CI applies | Writes code, opens PRs; never applies | ✗ | ✗ | ✗ |
| Merge into a repo | If listed as that repo's lead | ✗ (only opens PRs) | If listed as that repo's lead | ✗ | ✗ |
| Release an app to prod (approve its release PR) | If listed as that app repo's lead | ✗ | If listed as that app repo's lead | ✗ | ✗ |
| Azure resources (read) | Reader | Reader | ✗ | ✗ | ✗ |
| Kubernetes (read: pods, events, describe) | Read-only RBAC | Read-only RBAC | ✗ | ✗ | ✗ |
| Database | Read-only role (dev) | Read-only role (dev) | ✗ | ✗ | Own/granted rows via the app |
| Infra logs, audit logs, cluster metrics | ✓ | ✓ | ✗ | ✗ | ✗ |
| App logs, app metrics, app traces | ✓ | ✓ | ✓ | ✓ | ✗ |
| Use the app (REST / MCP) | as a normal user | as a normal user | as a normal user | as a normal user | ✓ |

How it's enforced:
- **Azure / AKS / Postgres**: Entra RBAC on the groups from §2.2. AKS local accounts disabled.
- **Grafana (OSS)**: two organizations, because Grafana OSS has no per-data-source permissions.
  - `platform`: all data sources, for `infra-developers`.
  - `app`: the app tenant in Loki and Tempo + app metrics in Prometheus, for `app-developers`.
  Org membership comes from Entra group → org mapping.
- **Loki / Tempo multi-tenancy**: app and infra telemetry go to separate tenants, so the `app` org can't query infra data even by accident.
- **Claude** uses the developer's own Entra login through the Grafana MCP server (`mcp-grafana`). Infra devs' Claude can also use `az` and `kubectl` with the read-only roles. Group membership decides what each Claude sees. No secrets on disk.

### 5.2 App users and their agents
- Users sign in through Entra External ID. MCP clients get a delegated token for that user, so they get the same permissions as the user.
- **Only verified emails count.** Permissions are looked up by the `email` claim, and that claim must be verified. Otherwise someone could sign up as `alice@…` and pick up Alice's pending shares.
- Scopes: `todos.read`, `todos.write`, `todos.share`. Delete and share tools are marked destructive, audit-logged with the client id, and rate-limited per client.
- Shared content from other users is marked as untrusted in MCP tool results (prompt-injection defense).

## 6. Sharing model (Unix/Linux permission model)

### 6.1 Permission bits

Each grant is a set of independent bits on one todo for one email, just like `rwx` on a file:

| Bit | Allows | Unix analogy |
|---|---|---|
| `read` | see the todo's content | `r` |
| `write` | change the todo's content | `w` (works without `r`: write-only is allowed; it shows in lists as `{ id, permissions }` only) |
| `share` | give *other* emails a subset of your own bits | (no direct analogy; the closest is a capability you can delegate) |
| `admin:share_creator` | change or revoke **anyone's** grants on this todo | owner of the file (`chmod`) |

- The **creator** of a todo is its owner. The owner holds `read`, `write`, `share` and `admin:share_creator`.
- **Delete** follows Unix: you remove a file through *its directory*, and the todo "lives" in its owner's list, so **only the owner can delete it**. In v1 grantees can't remove their own access ("leave" is future work).
- `admin:share_creator` **can't be granted through the API or MCP.** Giving it to anyone else is a manual database change by a developer (the "root" in this analogy).

### 6.2 Rules (a pure domain function, property-tested)
- A user may give a set `Q` to an email only if `share ∈ P_grantor`, `Q ⊆ P_grantor` (excluding `admin:share_creator`) and `Q ≠ ∅`.
  - Example: holding `{share, read}` lets you give `{read}`, `{share}` or `{share, read}`, never `{write}`.
- Grants only ever **add** bits. A `share` holder can't remove or lower anyone's bits; only an `admin:share_creator` holder can.
- **No cascade.** Revoking someone's bits doesn't touch the grants they made earlier.
- Effective bits = union of all active grants to that email.
- Property tests prove that no sequence of grants ever gives anyone more than its grantor held.
- Storage: `todo_grant(todo_id, grantee_email, can_read, can_write, can_share, granted_by, created_at, revoked_at)`; owner/admin is a separate `todo_admin(todo_id, user_id)` table.

### 6.3 No user enumeration
- Granting to an email always returns the **same response** (`204`, same body, same timing) whether the email belongs to a user or not.
- Grants are stored by normalized email and activate on that email's first verified sign-in.
- There is no endpoint or MCP tool that lists or searches users.
- Only `admin:share_creator` holders can see the grant list of a todo, and it shows only emails someone already typed in.
- Share endpoints are rate-limited per user and per client, so emails can't be probed in bulk.

## 7. Privileged access: Entra P2 + PIM (D23)

**Background.** PIM (Privileged Identity Management, part of Entra ID P2) replaces *standing* privileged access with *eligible* access. You activate a role for a limited time, with MFA and a justification; every activation is audit-logged and expires automatically. P2 includes P1, so we also get Conditional Access. Licenses are per user (roughly USD 9/user/month, to verify); only infra developers need one.

**Design**: every rule below is defined in Pulumi (F#) in the platform repo, generated from `access/people.yaml`.

| Access | Type | Who | Activation rules |
|---|---|---|---|
| Read (Azure Reader, AKS read RBAC, DB read-only, Grafana `platform` org) | Standing | `infra-developers` | Covered by the Conditional Access rules below (MFA) |
| **Break-glass write** (`infra-break-glass` group → Contributor on the app resource groups, AKS RBAC Cluster Admin, DB admin role) | **PIM for Groups, eligible only** | `infra-developers` | Max 2 h, MFA, written justification, notification to all platform leads. **Approval by another platform lead once there are ≥2 leads** (Entra doesn't allow approving your own request, so with one lead it's MFA + justification + alert). |
| Entra admin roles (Global Admin, Privileged Role Admin, Application Admin) | **PIM eligible, no permanent assignments** | Platform leads | Max 1 h, phishing-resistant MFA, justification |
| Emergency access | 2 cloud-only accounts, FIDO2, stored offline | You | Excluded from Conditional Access; any sign-in alerts all leads |

**Conditional Access** (replaces security defaults; the emergency accounts are excluded from every policy):
- CA01: MFA for everyone in the workforce tenant;
- CA02: block legacy authentication;
- CA03: phishing-resistant MFA for Azure management;
- CA04: sign-in risk policies (Identity Protection, a P2 feature) for infra developers;
- CA05: a fresh phishing-resistant sign-in for **every** PIM activation. An authentication context with sign-in frequency "every time", which the PIM policies of Global Admin, `platform-write` and `platform-admin` require. Without it, PIM accepts the MFA from an hours-old sign-in (D39);
- CA06: Azure portal, CLI and ARM for infra developers: sign in again every 4 h, no persistent browser session (D39);
- CA07: every other app for infra and app developers: sign in again every 12 h (D39).

**Alerts** (D40): PIM emails every activation and every new eligible or active assignment of a privileged role to the platform alert list. Until Phase 5 that is the only alert; Phase 5 adds the log-based ones.

**Why this matters.** Normal writes still go only through approved PR → CI (D18). PIM covers the moments when CI or ArgoCD itself is broken, and it makes even those moments time-boxed, justified and visible.

**Limits to verify in Phase 3**:
- PIM and Conditional Access apply to the **workforce tenant**. App users in the External ID tenant are separate, and that tenant's admin roles are kept to as few people as possible; I'll check which PIM features External ID supports.
- Buying the license is a manual bootstrap step (§4).
- Pulumi support for PIM resources (`azuread` PIM-for-Groups eligibility/policy, `azure-native` role management policies) will be checked in a short spike. Anything Pulumi can't manage gets a documented script that runs in CI.

## 8. Deferred components (add when the trigger happens)

| Component | Add when |
|---|---|
| **Valkey** | We need shared state across pods: distributed rate limiting (e.g. the share endpoint across ≥2 pods), a permission cache, or MCP session state. MCP starts in stateless mode. |
| **Service mesh (Linkerd)** | We get real service-to-service calls between our own pods, or a compliance need for encryption inside the cluster. Accepted until then: the hops ingress → backend and app → OTel Collector are plaintext inside the cluster, but restricted by network policies. |
| **Keycloak** | Entra External ID can't do something we need. The most likely case is MCP clients that require Dynamic Client Registration, which Entra doesn't support; Keycloak would then broker in front of External ID. |

## 9. Cost and network security (estimate, verify in Phase 3)

### 9.1 Network: the database is never reachable from the internet
- PostgreSQL Flexible Server with **VNet integration** (delegated subnet, *private access*). It has **no public endpoint at all**; this isn't a firewall rule that could be opened later. It is resolved through a private DNS zone.
- **NSG on the DB subnet**: inbound 5432 only from the AKS node subnet; everything else denied.
- **Pod level (Cilium network policies)**: default-deny egress in every namespace. Only pods labeled `app=todo-backend` (rendered from `database: todo` in `platform.yaml`) may reach the DB's private IP on 5432. Frontend, observability and ArgoCD pods can't.

  *Why both layers*: with Azure CNI overlay, pod traffic leaves with the node's IP, so the NSG alone can't tell backend pods from other pods. The Cilium policy does that part. A dedicated node pool for the backend would let the NSG do it too, but it costs an extra node; I'll note it as an option.
- **Identity**: the DB accepts only identities on its allowlist (§9.2), so even reachable traffic has to authenticate.
- Same pattern for Key Vault and ACR (private endpoints or firewall + trusted services, depending on SKU cost; decided in Phase 3). Grafana/ArgoCD/Kubernetes API access is via Entra SSO.

### 9.2 Database user security (option B chosen, D24)

| Option | How | Pros / cons |
|---|---|---|
| A. Password user | One app user, password in Key Vault, CSI-mounted | Simple; a password exists and must be rotated |
| **B. Entra-only, split identities (chosen)** | Password auth disabled. Workload identities: `todo-migrator` (DDL on schema `todo`, used only by the migration Job), `todo-app` (`SELECT/INSERT/UPDATE/DELETE` on `todo` tables only), `infra-readers` group (`SELECT` only). Short-lived tokens. | No secrets anywhere, least privilege, every connection is attributable to an identity. Locally we use password users with the same three roles, so the role model is still tested. |
| C. B + column-level grants + `pgaudit` | Same as B, plus audit logging of all statements by human/agent identities | Strongest; `pgaudit` adds log volume. A good "future work" item. |

### 9.3 Monthly estimate (one `dev` environment)

| Item | Choice | ~USD/month |
|---|---|---|
| AKS control plane | Free tier | 0 |
| Nodes | 2 × `Standard_B2ms` (2 vCPU / 8 GB) | ~120 |
| PostgreSQL Flexible | Burstable `B1ms`, 32 GB, private access | ~15–20 |
| Ingress | Standard LB (comes with AKS) + Envoy Gateway in-cluster + cert-manager/Let's Encrypt | ~20 |
| Logs/traces storage | Loki + Tempo on Azure Blob, short retention | ~5–10 |
| ACR | Basic | ~5 |
| Key Vault, External ID (free ≤ 50k MAU), VNet, private DNS | | ~1–3 |
| Entra ID P2 | 1 license (you); +1 per extra infra developer | ~9 |
| **Total** | | **~175–190** |

**With `prod` (D38)**, the cluster, Postgres, ingress, storage and network costs repeat, so the total is about **USD 350/month**. ACR, Entra and P2 licences are shared.

Savings levers: stop AKS + Postgres between demos (both environments). Trade-off kept: no managed WAF (Application Gateway WAF costs more than the rest combined). Instead we use rate limiting, strict routing, security headers and app-level parsing.

## 10. Phases (in order; each ends green in CI with no broken windows)

### Phase 0: Ways of working + bootstrap
- Done: `AGENTS.md`, `CLAUDE.md`, `.prompts/`, `PLAN.md`.
- `git init` both repos (`reference/` in `.gitignore`), push to `Ulberg/todo` and `Ulberg/platform`.
- `.editorconfig`, Fantomas, Prettier/ESLint (all enforced in CI only, no git hooks), Claude Code prompt-capture hooks, gitleaks + Dependabot.
- Azure bootstrap checklist + script (§4). Pulumi "bootstrap" stack (F#): state storage, CI identity, `access/people.yaml` → Entra groups, PIM eligibility + policies, Conditional Access, GitHub rulesets, CODEOWNERS.
- **Done when** both repos are protected by rules defined in code, and the CI identity can run `pulumi preview`.
- Status (2026-11-09): bootstrap done; `platform/stacks/access` merged and applied (PR #2) (groups, PIM for Groups, Reader for `platform-read`, rulesets, environments, CODEOWNERS check), preview runs clean. Steps after that:
  1. **Conditional Access on.** The policies are in code behind `access:conditionalAccess` (`off` → `report-only` → `on`). Entra doesn't allow CA policies while security defaults are on, so the switch is: you turn security defaults off, an apply sets `report-only`, you check the report-only results in the sign-in logs, and a second apply sets `on` the same day. Report-only enforces nothing, so that window is kept to hours, not days. The emergency accounts are excluded from every policy. This step also adds CA05–CA07 (short sessions and a fresh passkey for every PIM activation, D39).
  2. ✅ **Remove standing admin access** (done 2026-11-09, platform PRs #3–#8). Global Admin is PIM-only (1 h, MFA, justification; no permanent active assignment) and was tested; the admin account has Reader day to day; the owner Microsoft account keeps only billing (D35); subscription Owner is held only by the two emergency accounts (D36); every activation and new assignment emails the alert list (D40). `check_standing_access.sh` fails the `pulumi preview` check (every PR, main and nightly) if standing Global Admin, Privileged Role Administrator, Owner or User Access Administrator appears outside PIM and the emergency accounts.

### Phase 1: Walking skeleton backend + local tier 1 + CI
- F# Oxpecker (.NET 10): `/health/live`, `/health/ready`, `/version`.
- OpenTelemetry traces/metrics/logs over OTLP; JSON logs carrying `trace_id`; `ProblemDetails` errors that include `trace_id`; no PII (hashed user id).
- Local tier 1 (§11). Testcontainers integration tests.
- Hardened Dockerfile; app CI: format → build (warnings as errors) → tests → `platform.yaml` schema check → image → Trivy → SBOM.
- **Done when** CI is green and one request shows locally as trace + log + metric.

### Phase 2: Local platform (tier 2, kind)
- Platform repo: `platform.yaml` schema + F# renderer → K8s manifests, with tests.
- kind + Cilium + Envoy Gateway + ArgoCD + LGTM, the same Helm values as Azure, synced from the local checkout.
- Automated policy tests, e.g. "frontend pod can't reach Postgres", "pod without a label has no egress".
- **Done when** `make up` brings the full stack up locally and the Phase 1 app deploys via ArgoCD.

### Phase 3: Azure foundation (Pulumi F#, applied only by CI after approval)
- Workforce groups, External ID app registrations, workload identities.
- VNet + subnets + NSGs, private DNS, ACR, Key Vault, AKS (Entra RBAC, no local accounts, workload identity, Azure CNI overlay + Cilium), Postgres Flexible (private access, Entra-only auth with migrator/app/read-only identities).
- Pulumi unit tests (mocks); `preview` on every PR, `up` after approval.
- **Done when** everything exists, and the access tests show: app devs can change nothing; infra devs' Claude can read but not write; the DB isn't resolvable or reachable from outside the VNet.

### Phase 4: GitOps + cluster baseline on Azure
- ArgoCD (Entra SSO, admin login disabled), app-of-apps, the `platform.yaml` PR flow (§2.1 steps 1–3), signed images + admission policy.
- The release-PR flow (§2.1 step 4): the pr-bot opens it, the policy check runs, and the path-based `CODEOWNERS` (app leads own `apps/<app>/prod/`) is generated and checked by the `access` stack. Until prod exists (Phase 6b), nothing syncs `apps/<app>/prod/`.
- Backend ≥2 replicas, HPA, PDB, topology spread; default-deny network policies (identical to tier 2).
- **Done when** an app-repo merge reaches the cluster with no human touching the cluster.

### Phase 5: Observability platform on Azure
- OTel Collector → Loki / Prometheus / Tempo (Blob) → Grafana (Entra SSO, two orgs, tenants).
- Infra signals: kube-state-metrics, node metrics, AKS audit logs, Postgres metrics/logs, Envoy access logs, ArgoCD.
- Dashboards + alerts as code; `mcp-grafana` setup documented for both groups.
- Entra sign-in and audit logs → alerts (D40): any emergency-account sign-in, role or group-membership changes not made by the apply identity, changes to PIM policies or Conditional Access.
- **Done when** each group's Claude sees exactly its column of §5.1, proven by tests.

### Phase 6: Edge + application security
- TLS, HSTS, CSP, security headers, strict CORS, rate limiting.
- External ID JWT validation (issuer, audience, scopes, **verified email**), email → user → permissions (`bindPermissionsByBearer` pattern), deny by default.
- Users table, migrations Job.
- **Done when** the security suite (authz matrix, CORS, headers, anonymous/wrong-scope rejection) passes locally and against dev.

### Phase 6b: `prod` environment + releases (D37, D38)
- First: remove the solo-dev exception (D34, §13), so every change to `main` gets a review.
- A `prod` Pulumi stack from the same code: its own AKS cluster, Postgres, VNet and Key Vault. A `prod` GitHub environment whose reviewers are the platform leads, and an apply identity federated to it.
- ArgoCD syncs `apps/<app>/prod/` into the prod cluster. Post-deploy smoke + access tests run against both environments.
- **Done when** an app change goes PR → app-lead approval → dev (automatic) → release PR → app-lead approval → prod, with no human touching either cluster, and an app developer who isn't a lead can't release.

### Phase 7: Frontend shell
- Vite + React + TS, TanStack Router (CSR) + Query, shadcn/ui, MSAL.
- `openapi-typescript` + `openapi-fetch` with auth + `traceparent` middleware; the spec is committed and checked for drift in CI.
- Browser telemetry (OTel web / Faro) joined to the backend traces.
- **Done when** login works and a "who am I" page shows one trace browser → API → DB.

### Phase 8: Agent shell (MCP)
- Stateless `/mcp`, protected-resource metadata, pre-registered client(s), scopes, audit log, per-client rate limit.
- REST/MCP parity check in CI; first tool `whoami`.
- **Done when** the feature template (domain → DBO → use case → REST → MCP → FE hook → tests → dashboard panel) is documented in `AGENTS.md`.

### Phase 9: Todo features
- Create, list, get, edit, delete, each through the template.
- **Done when** all five operations work for a human (FE) and an agent (MCP) under the same permissions, with tests and a dashboard panel for each.

### Phase 10: Sharing
- Grant, admin change/revoke, grant list for admins, pending grants, no-enumeration tests (same response + timing), property tests, untrusted-content marking in MCP.
- **Done when** the property tests (no escalation), the no-enumeration tests (same response and timing) and the authz matrix all pass, and sharing works the same from the FE and from MCP.

## 11. Local development

| Tier | For | What runs | Reliable for |
|---|---|---|---|
| **1: inner loop** | App devs, daily | `docker compose`: Postgres (with the same 3 roles as §9.2), `grafana/otel-lgtm`, **mock OIDC server** issuing External-ID-shaped tokens (incl. `email_verified`). Backend `dotnet watch`, frontend Vite | All app code, auth/permissions/sharing, telemetry, MCP (Claude Code connects to local `/mcp`) |
| **2: local platform** | Infra devs, plus app devs checking a `platform.yaml` change | `kind` + Cilium + Envoy Gateway + ArgoCD + LGTM, the same charts/values as Azure | Rendered manifests, network policies, Pod Security, HPA/PDB, ArgoCD sync, ingress/CORS/headers, Grafana org/tenant split |
| **3: cloud-only** | CI | Pulumi unit tests with mocks; `pulumi preview` against real dev | Azure resources, Entra, NSGs, private DNS/endpoints, workload identity |

**How reliable is tier 2?** Kubernetes-level behavior (network policy enforcement with Cilium, RBAC, ArgoCD, Envoy routing, the observability stack) behaves the same as on AKS, because we run the same components and the same versions. What it can't reproduce:
- Azure networking (NSGs, private DNS, Standard LB);
- workload identity → Entra tokens (locally we use password users / mock OIDC);
- Postgres Flexible specifics.

Those are covered by tier 3 plus a post-deploy smoke + access test suite against dev. So you can do almost all work locally before touching the real environment; only Azure/Entra resource changes need CI.

Tests: unit + property (domain), Testcontainers (storage), in-process API tests, Playwright smoke on tier 1/2, access-matrix tests on tier 2 and dev.

## 12. Patterns from `reference-infra/`

| Reference pattern | What we do with it |
|---|---|
| `azure_ad_fas_group`: per team/env groups `read`, `write`, `admin`, **nested** (admin ∈ write ∈ read) | **Adopt.** `people.yaml` renders `<scope>-read/-write/-admin` Entra groups with the same nesting, so every consumer (Azure RBAC, AKS, Grafana, DB, PIM) maps to only three levels. `write`/`admin` are only reachable through PIM (§7). To verify in Phase 3: nested group claims in AKS tokens. |
| `generate-rbac.py`: RoleBindings generated per namespace × level from group names | **Adopt**, inside the F# `platform.yaml` renderer. No hand-written RBAC. |
| `fas_application/ci-cd`: GitHub OIDC trust scoped to a repo, and to `main` for prod | **Adopt, tighter**: scoped to a GitHub *Environment* with required reviewers (§2.1). |
| `fas_application/service_profile`: one cloud identity per service, bound to its K8s service account, with least-privilege policies derived from declared needs (queues, buckets, secrets) | **Adopt.** It's the same idea as `platform.yaml`: declared needs → workload identity + exactly those role assignments. |
| `.fas/` folder in app repos with raw Kustomize base/overlays + image tag | **Replaced** by the higher-level `platform.yaml`. Raw manifests in app repos would give app devs control over pod specs, which we don't want. |
| External Secrets Operator + per-app `SecretStore` bound to the app's identity | **Adopt** with Azure Key Vault + workload identity. There will be few secrets (DB is Entra-only), but the pattern is in place. |
| cert-manager + ClusterIssuer, external-dns, custom error pages | **Adopt** cert-manager now; external-dns when we add a custom domain. |
| Linkerd service mesh | **Deferred** (D32, §8). |
| Teleport for K8s/DB/SSH access | **Not adopted** (D33): Entra SSO needs the paid Enterprise edition. Entra RBAC + `kubelogin` + the `db-access` proxy cover it. |
| Dependabot reporting / `VULNERABILITYOWNERS` | **Adopt** a light version: Dependabot + GitHub security alerts, with Trivy results in CI. |
| Flux (Lab) | **Not adopted**: we use ArgoCD as agreed. |
| Jenkins, Fluent Bit → Splunk, Skogul, SOC logger | **Not adopted**: GitHub Actions + the OTel/Loki stack cover these. |

## 13. Future work

- Postgres row-level security as defense in depth (D16).
- `pgaudit` + column-level grants (§9.2 option C).
- "Leave a share": a grantee drops their own access (D27).
- Custom domain.
- Managed WAF, or Coraza in Envoy.
- A2A Agent Card, if the app ever contains its own agent.
- **TODO, solo-developer exception (D34).** Remove the bootstrap exception so every change to `main` gets a review, even with one person. Proposed approach:
  - Claude gets its own GitHub identity (a `ulberg-claude` GitHub App with only contents + pull-request access) and opens every PR as that app.
  - The human lead reviews and approves as code owner. Required reviews are switched on in the rulesets, with no bypass for anyone.
  - Human-authored changes go through Claude-opened PRs as well, or wait for a second lead.
  - This must be done before a `prod` environment exists (first step of Phase 6b), and before anyone else joins.
