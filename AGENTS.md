# Agent Guidelines for Kubernetes Cluster Operations

Concise architectural rules and constraints for this repository. Follow to maintain consistency.

## Immutable GitOps & Ansible Separation (Hard Rule)

**GitOps-first is the default for ALL infrastructure changes.** Every change to cluster state — Kubernetes resources, database config, secrets, passwords, schemas, node configuration — must be made declaratively through Git (Flux reconciles) or Ansible playbooks/roles. The git/Ansible declaration is the source of truth; the live system converges to match it.

**Kubernetes-native configuration before Ansible (Hard Rule).** Use built-in resources
or operator CRDs reconciled from Git whenever they can express the required behavior.
Use Ansible only after verifying that Kubernetes-native configuration cannot meet
the requirement; explain the missing capability and obtain approval for the fallback.
Keep either path declarative; a scheduling guard is not a substitute for a hard limit.

**Never mutate live production state imperatively** (via `kubectl exec`, SQL clients, SSH commands, `helm install`, etc.) as the primary approach. Imperative mutations are the **exception**, not the rule — used only when a declarative path genuinely doesn't exist (rare), and even then must be documented with rationale and converted to a declarative equivalent as soon as possible. When in doubt: commit to git and let the operator/controller/Ansible reconcile.

### Strictly Prohibited (no exceptions once cluster is up)
- `kubectl delete` (namespace/deployment/service/configmap/secret/pvc/kustomization/pod — except CrashLoopBackOff)
- `kubectl apply/create/patch/edit/replace/expose/autoscale/scale/set` on Flux-managed resources
- `kubectl rollout restart / set image / set env`
- `kubectl exec` (mutating), `kubectl cp`, `kubectl port-forward` (prod), `kubectl proxy`, `kubectl debug` (mutating). **One bounded exception:** deleting a record external-dns wrote but cannot read (a type absent from the rfc2136 AXFR switch) requires `pdnsutil rrset delete`, because external-dns can never plan a delete for a record it cannot list. Say so explicitly when it applies — see the DNS section.
- `kubectl drain/cordon/uncordon/taint/label/annotate` (use Ansible `node-*` roles or git)
- `kubectl run`
- Direct `helm` CLI mutations (install/upgrade/uninstall/repo add) — must be inside Ansible roles
- SSH mutating commands (`apt`, `systemctl restart`, `kubeadm`, manual file copies)
- **SQL mutations on live databases** (`ALTER USER`, `ALTER ROLE`, `CREATE DATABASE`, password changes, schema mutations via `kubectl exec -- psql -c ...` or any SQL client). All database configuration (passwords, roles, schemas) must be managed declaratively through the operator's CR spec (e.g., CNPG `enableSuperuserAccess` + `superuserSecret`) or through Ansible roles — never via imperative SQL against a live database.

**Allowed read-only only:** `kubectl get/describe/logs/events/top/diff/auth/can-i`, `kubectl version`, `kubectl api-*`, etc.

**For Flux Kustomizations specifically:** Prefer committing the change to git and pushing — Flux's garbage collector prunes removed resources on the next reconcile (the webhook triggers this within seconds of a push; the 1m interval is the backstop). No manual `flux suspend` is needed for ordinary resource removals (HTTPRoutes, Deployments, Services, etc.). Suspend while preparing removal of a `Kustomization` CR or touching resources an ImageUpdateAutomation pipeline writes back to, to avoid reconciliation and commit-overwrite races. **Deleting a suspended Kustomization skips finalizer pruning and orphans its resources.** For complete retirement, first reconcile an empty resource set with `suspend: false`, verify its inventory is empty and its resources are gone, then remove the empty CR from Git. Never `kubectl delete` a Flux-managed resource directly — always remove from git and let Flux prune.

**Non-Flux exceptions (CoreDNS manifests only):** `kubectl apply -f` the committed `coredns-*-topology.yaml` files after node changes. Commit first.

## Core Architecture Principles

### 0. Kubernetes-Native First
Built-in primitives > ecosystem projects > third-party. Flux > Argo CD. CRD-native operators > agent-based tools. Gateway API over Ingress. Prefer `kubernetes/` and `kubernetes-sigs/` projects.

**Decision rule:** "Does Kubernetes/CNCF already provide this?" before choosing third-party. Document rationale for exceptions.

### 1. Immutable Infrastructure
Nodes are **never patched in place**. Reprovision from a clean slate on every run of a provisioning playbook/role.

- No "already done" guard logic in roles (provision, storage-longhorn, etc. always run fully).
- Correctness/safety nets only (e.g. don't double-add a btrfs device).
- A node that gets stuck is re-provisioned from scratch — never "finish the remaining steps" over SSH.

### 2. Immutable Data Operations (Stateful Stores)
Never `UPDATE`/`DELETE` on live production tables or in-place schema changes that lose data.

All migrations/enrichments (ClickHouse, object storage, DB schemas) use the 4-step pattern:
1. Dual-write (new writes go to both old and new versioned object)
2. Backfill historical data
3. Validate (counts, spot checks)
4. Swap read paths + archive old (rename to `_v1`/`_archived`; drop only after confirmed production period)

Naming convention for versioned stateful objects: `<name><delimiter><version><delimiter><status>` (e.g. `events_v2_staging`, `events-v2-staging`).

### 2b. Service Component Colocalization
Every microservice owns one dedicated namespace containing:
- Its Deployment/StatefulSet
- Its database (CNPG Cluster)
- Its Secrets/ConfigMaps
- Its NetworkPolicy/ReferenceGrant + routes
- Any sidecars/init jobs specific to it

Cross-namespace is the exception (shared infra like Cilium/cert-manager/monitoring; oauth2-proxy for internal tools).

## Policies

### Repository-Owned Assets and Deliverables (Hard Rule)
**Save every agent-created asset and deliverable inside the repository, never in `~/.copilot/`, `.copilot/`, session-state folders, agent memory folders, or other agent-private storage.** This includes CSV exports, reports, images, diagrams, and reusable generated files. Colocate them with their owning feature; use `assets/` for standalone deliverables without an existing owner. Link to the repository copy, not a session-private copy. A tool-provided session folder does not override this rule.

Keep reusable, non-sensitive assets under version control. Sensitive exports must still live inside the repository, but keep their exact paths gitignored unless stored with the repository's approved encryption workflow; never commit or push plaintext financial data or secrets.

### Missing CLI tools
If a CLI tool is needed but not installed, do NOT use `sudo apt-get install` or any system package manager. Instead:
1. Check if mise provides it — add to `[tools]` in `mise.toml` (e.g. `dig`/`nslookup` via `"apt:bind9-dnsutils"` in `[bootstrap.packages]`).
2. If it's a system C library or OS package, add to `[bootstrap.packages]` in `mise.toml`.
3. Run `mise install` (for tools) or `mise bootstrap packages apply --yes` (for system packages).
4. If mise refuses to run (untrusted directory), run `mise trust` first, then retry.
This keeps the environment fully declarative and reproducible from `mise.toml` alone.

### Edit Tooling Preference (Hard Rule)
Never use terminal commands (`sed`, `awk`, `echo >`, `cat >`, `tee`, `tr`, `perl -i`, etc.) for text edits. **Always use the VS Code edit tools** (`replace_string_in_file`, `multi_replace_string_in_file`, `create_file`) so changes are visible in the diff editor for review. Terminal commands bypass the review workflow and can silently corrupt files — especially SOPS-encrypted files where a stray `sed` breaks the MAC integrity check. This applies to **all** file edits, including trivial one-line changes, namespace renames, and temp file processing. If you need to transform file contents (e.g., decrypt → modify → re-encrypt), use the edit tools for the modification step, not `sed`/`awk`.
### No Truncating Terminal Output (Hard Rule)
**Never use `tail`, `head`, `grep`, or any pipe that cuts off output when running commands.** Always run commands directly and let the full output stream so we can follow along together. This applies to **all** terminal commands — `cargo check`/`build`/`test`, `docker build`, `kubectl logs`, `ansible-playbook`, `mise run`, and every other long-running or multi-line command. If a command produces a lot of output, let it scroll — do not filter it. `tail`/`head` are acceptable **only** for post-hoc analysis **after** a run has fully completed (e.g. re-reading a completed build's output to extract a specific line), never to limit output during the run itself.

**The same rule applies to suppressing output at the source — never quiet a tool's logging.** Don't pass `--log error`/`--quiet`/`-q`/`--no-progress` or similar to shrink output, and don't build tasks/scripts whose default is a summary instead of a per-item log. This is the same mistake as truncating a pipe, just moved upstream: a silent run is unverifiable, and the operator loses the ability to follow along and confirm each item worked. We **always** prefer log lines — verbose per-item output is the expected default, and a run with more lines is better than a run with fewer. If a tool emits a line per item (per object, per file, per request), leave it on and let it scroll. Only reduce output when the operator explicitly asks for it, and never as a default baked into a task.
### Default-First Configuration (Hard Preference)
Use maintainer defaults. Omit settings that merely restate them. Override a default only when it fails a concrete requirement and upstream documentation prescribes the change. Record the failure or unmet requirement and the upstream rationale beside the override. Explicit security and privacy requirements, such as binding a private service to loopback, must still be enforced.

### Docs-First for Tooling (Hard Rule)
When using a tool or library for the first time (or encountering a non-trivial configuration question), **always check the official documentation first** before trial-and-error. Find the canonical/prescribed way to do it, implement that, and record it in AGENTS.md under the relevant tooling section. Never guess at API shapes, configuration fields, or patterns — fetch the docs, read the prescribed approach, and follow it exactly. This prevents hours of wasted iteration on wrong assumptions.

**Binary search debugging for procedural code:** When stuck after a couple of failed iterations on Dockerfiles, scripts, or any multi-step procedural code, comment out everything except the first essential step. Verify it works (locally + in-cluster). Then uncomment one line/step at a time, verifying each works before proceeding to the next. This isolates the exact failing step without rewriting from scratch.

### Component Versioning
Select the latest stable release compatible with the cluster. When a deployment mechanism requires an exact reference, record that resolved release in its owning declaration and keep it current through the dependency-update workflow. Upgrade one component at a time: one control plane at a time, then workers, then add-ons.

### Dependency Update Discovery

Dependabot owns application dependency manifests, Dockerfiles, and GitHub Actions;
keep its directories aligned with maintained projects. Renovate owns mise tools,
Ansible dependencies, and the generic infrastructure release/image discovery below.
Assign each dependency to one bot; disable Renovate managers covered by Dependabot.

Dependabot updates, including majors, are squash-merged only after the Linux/macOS
infrastructure checks and website type-check, tests, and build pass for the tested
head against `main`. Enforce this in the merge workflow, not branch-wide rules:
Flux writes website image tags directly to `main`. The privileged `workflow_run`
job must not check out or execute pull request code or consume its artifacts.

**New image declarations must be automatically discoverable without per-image changes to `renovate.json`.** Prefer built-in managers and generic structured-data discovery. Use complete `image: repository:tag` references, or declare `repository` alongside `tag`/`version` in Helm image objects; never leave a tag-only override whose repository exists only in chart defaults. Do not add image-specific extractors. Validate additions with `mise run renovate-configuration-checks`. Flux retains ownership of internally built application image updates.

### Latest Stable Versions (Hard Preference)
Use the latest stable release of tooling, runtimes, and dependencies, and the publisher's default revision of the selected model. Leave requests unpinned where the installer supports it. An exact version in a generated lockfile or a required deployment reference records a resolution; it is not a reason to stay on an older release. Pin a request only when the operator explicitly asks or a verified compatibility failure requires an upstream-documented workaround. If a change breaks a dependent, upgrade the dependent too. Verify affected software before pushing.

**Never revert backwards.** When a dependency has already been bumped (even by an automated tool like Dependabot/Renovate), never revert it to an older version to work around a break. Instead, fix the breakage by upgrading the *downstream* consumer (e.g. if `syn` 3 breaks `build.rs`, upgrade `prettyplease` to a syn-3-compatible version and fix the API change — do not revert `syn` to 2.x). Pinning to old versions accumulates technical debt and defeats the purpose of automated dependency management. Only revert if the user explicitly requests it.

### Sweeping Changes — Per-Component Verification (Hard Rule)
When making sweeping changes (removing a feature, restructuring workspaces, bumping shared deps, etc.) that touch multiple components, **verify each affected component individually** before pushing:
1. **Compile** — `cargo check` / `cargo build` / `mise run <app>-build` for each affected component.
2. **Test** — `cargo test` / `mise run <app>-test`. Tests requiring external services (PostgreSQL) that fail with "Connection refused" are expected locally; verify no *new* test failures from the change.
3. **Run locally** — `mise run <app>-run` (starts the app's daemon via mise) to verify the app starts and the health endpoint responds.
4. **Push** — once all components pass, push to git and let CI/CD handle deployment.
5. **Validate on prod** — after deployment, verify the service is healthy in production (read-only `kubectl get/describe/logs`, health endpoint, smoke test).
Never push sweeping changes without first verifying every affected component compiles and runs locally.

### Alphabetical Dependency Ordering
In all `Cargo.toml` (and other manifest) files, list dependencies **alphabetically by key** within each `[dependencies]` section. Merge third-party and local/path deps into a single list (no separator comments). This makes it easier for human reviewers to find and parse dependency lists, and avoids duplicate entries. Apply this to `[workspace.dependencies]`, `[dependencies]`, `[dev-dependencies]`, and `[build-dependencies]` sections alike.

### Mise Tasks for Repeated Workflows (Hard Rule)
**Create mise tasks only for repeated operations and reusable workflows. One-off commands do not need a mise task; run them directly instead of adding a task solely for a single execution.** Reuse an existing task when it provides required environment variables, age secrets, dependencies, or safety checks. For one-off commands, name the required tools explicitly, for example `mise exec kubectl -- kubectl get pods`, so mise does not auto-install unrelated project runtimes. Promote an operation to a task when it becomes a recurring workflow.

This exception applies to the mise-only wording elsewhere in this document, including build, test, validation, and Ansible tooling. It does not permit imperative production mutations: infrastructure changes still belong in GitOps declarations or Ansible roles/playbooks. Required migration review, secret handling, and other safety checks must not be bypassed.

### Local Pull Request Validation (Preference)
Run the same existing mise tasks locally against the exact changes being merged, and proceed when the relevant local checks pass rather than waiting for GitHub Actions. Keep CI and Renovate auto-merging enabled; do not rerun or watch Actions merely to duplicate successful local validation. Respect any enforced GitHub required checks without administrator bypasses. This preference does not waive post-push Flux reconciliation and production health verification.

### Isolated Worktrees (Preference)
Use a separate Git worktree for changes by default, especially when other sessions or the operator are editing the repository. Never switch branches, stage, or commit another task's shared checkout. Start from the appropriate committed base and validate the combined result before merging; do not copy unrelated uncommitted edits. In the VS Code Agents window, select New Worktree; `sessions.useWorktree` sets the initial default, but an existing workspace remembers its last isolation choice. Worktrees isolate files, not cluster operations or ports, so continue coordinating production changes.

### Leptos Macro-Free & Builder Syntax (Hard Rule)
All Leptos-rs UI code must avoid macros wherever a macro-free alternative exists. This is a hard preference against macros in all Rust code we write, not just Leptos — macros bypass rust-analyzer's type inference, autocomplete, refactoring, and error messages. Apply the following rules to all Leptos components, pages, and views across the repo — both new code and existing code. When touching any file that violates these rules, convert it in the same change (gradual migration, no big-bang rewrite):

**1. No `view!` macro — use builder syntax:**
Construct all HTML views with `leptos::html` builder functions and method chaining: `html::div().child(...)`, `.attr(...)`, `.on(...)`, `.class(...)`, `.style(...)`. See https://book.leptos.dev/view/builder.html for the canonical reference.

**2. No `#[component]` macro — use plain functions:**
A Leptos component is just a function that returns `impl IntoView`. Define components as plain functions with regular function arguments instead of `#[prop]` destructuring:
```rust
// YES — plain function, no macro
pub fn blog_post_card(title: String, slug: String, tags: Vec<String>) -> impl IntoView {
    html::a().attr("href", slug).child(html::h2().child(title))
}

// NO — #[component] macro
#[component]
fn BlogPostCard(title: String, slug: String, tags: Vec<String>) -> impl IntoView { ... }
```
When using component macros from Leptos itself (e.g. `Router`, `Routes`, `Route`, `Title`, `Show`, `For`), invoke them via their props builder structs (e.g. `Router(RouterProps::builder().build())`, `Route(RouteProps::builder().path(...).view(...).build())`) rather than via `view! { <Router>...</Router> }`.

**3. `#[island]` macro — necessary exception:**
The `#[island]` macro (for islands architecture) is the one Leptos macro that has no macro-free equivalent. It generates a `#[wasm_bindgen]` export with a unique hash-based name, wires up `Island::new()`, handles prop serialization for `data-props`, and wraps `children` in `IslandChildren`. The JS hydration script (`island_script.js`) looks up islands by matching `data-component` against WASM module exports — this requires the macro's code generation. Do NOT attempt to replace `#[island]` with manual code. The body of an `#[island]` function must still use builder syntax (no `view!`), same as all other views.

### Descriptive Naming (Hard Rule)
Always use well-named, descriptive variable and type names. Never use shortened or abbreviated names. For code we are touching, rename abbreviated identifiers to their elongated, descriptive forms in the same change. This applies to **all code and configuration** — Rust, Kotlin, TypeScript, YAML, Ansible, TOML, etc.:
- **Type names** — `ConversationResponse` not `ConversationDto`, `ChatMessage` not `ChatMessageDto`, `SendMessageRequest` not `SendMsgReq`. Do not carry over naming conventions from other languages (e.g. Kotlin's `Dto` suffix) into Rust.
- **Variable names** — `influencer_identifier` not `inf_id`, `conversation_identifier` not `conv_id`, `authentication_token` not `auth_tok`.
- **Function names** — `get_authentication_token` not `get_auth_tok`, `create_conversation` not `create_conv`.
- **Constants** — `AGENT_BACKEND_URL` not `AGENT_URL`, `DISCOVERY_FEED_PATH` not `DISC_PATH`.
- **Configuration keys** — `database_connection_timeout_seconds` not `db_timeout`, `maximum_retry_attempts` not `max_retries`.

Modern IDEs and language tooling make long names ergonomic regardless of language. Descriptive names serve as inline documentation and make grep/code-search effective. Abbreviations create cognitive overhead and inconsistency.

**No legacy domain jargon (Hard Rule):** Do not carry over naming from previous platforms. Call things what they actually are in the current system (e.g. `oauth_subject` / `user_id` for user identifiers, `service_id` for service identifiers). When touching code that uses legacy jargon from a prior platform, rename to the accurate current term in the same change. This prevents conflation and makes the code self-documenting.
### Comments Are the Exception, Not the Default (Hard Rule)
**Hard preference: do not add unnecessary comments. Remove resolved TODO comments without replacing them with findings, incident summaries, or explanatory comments unless explicitly requested.**

**Default to writing no comment. Names and types carry the documentation burden.** We write far too many comments, and a comment is a maintenance liability: it duplicates a decision that already lives in the code, so it drifts, contradicts, and misleads the next reader. Before writing one, ask whether a better name or type would say the same thing — if it would, rename instead of commenting.

**Every comment that IS written carries exactly one of four tags, as the first character after the comment marker.** Better Comments colours them and Todo Tree indexes the `TODO` ones, so the tag tells a reader *why* the line is annotated before they read it — and a comment without a tag is a violation, not a stylistic choice:

| Tag    | Category  | Use for                                                                                                                                                                        |
| ------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `*`    | Highlight | important info, general notes — the counter-intuitive discovery that cost real time to find                                                                                    |
| `!`    | Alert     | warnings / critical notes — the correct action is the opposite of the obvious (an ordering constraint, an operation that looks safe but destroys state, a silent failure mode) |
| `?`    | Query     | an open question, something to investigate or confirm before relying on it                                                                                                     |
| `TODO` | Todo      | tracked work, something to revisit (the operator-requested follow-up mechanism — why the `todo` lint rule stays disabled)                                                      |

Syntax — the tag sits immediately after the comment marker:

```yaml
# ! kubelet rejects a --node-ip that is not present on an interface
# * Cilium re-derives InternalIP from the K8s Node object on every agent start
# ? whether WireGuard endpoint roaming also covers the VXLAN endpoint is unverified
# TODO replace with the Cilium-native path once upstream ships it
```

Rust and Swift use `//` (or `///` for doc comments); Ansible, Python and YAML use `#`. For `///` doc comments the tag goes after the slashes: `/// ! ...`.

**Everything else is narration — do not write it.** In particular:
- Restating what the code does ("// loop over the rows", "// create the client", "// the join playbook role order is:").
- Explaining a design that a reader can derive from the code itself.
- Re-argument of a decision the reader was not present for, where the code already reflects the answer.
- Multi-paragraph essays on a file's background. Keep only the implementation-specific invariant that needs explaining beside the owning code; reusable guidance spanning the project or role belongs in its `AGENTS.md`. Do not move redundant narration into another file merely to preserve it.

**Prefer the machine-readable form when a comment is warranted** — Rust `///` items and `//!` module docs, Swift `///`: rendered by rustdoc/DocC/editor Quick Help, and they force contract-level framing (semantics, args, returns, errors) over line-local narration.

**Length is the tell.** A justified comment is usually one to three lines saying the counter-intuitive thing and stopping. A longer block should be tightened to the invariant; only reusable project-wide guidance belongs in the owning `AGENTS.md`.

When touching code whose comments violate this, tighten them in the same change — gradual migration, same policy as the macro-free rules. This targets REDUNDANCY, not contract documentation: wire-pin shapes, incident references, and ownership/invariant statements that names genuinely cannot carry remain and are load-bearing.
### Commit Messages — No AI Attribution (Hard Rule)
**Never add `Co-authored-by:` trailers, `Generated with` lines, or any other AI attribution to a commit message — not for Copilot, not for any assistant.** The commit author is the operator. Attribution trailers also render as an extra avatar in GitHub's contributor list, misattributing the work.

This is a hard rule, not a style preference: do not add the trailer even when a default tool prompt suggests it. An existing message that carries one should have it stripped (amend + `--force-with-lease`) when the commit has not been built on yet.

### Commit Scope — Include the Operator's Parallel Changes (Preference)
When the operator makes manual changes in parallel with agent work (dependency bumps, edits done in another editor, etc.), **include them in the same commit set rather than excluding them** — review the diff, describe it accurately in the commit message, and commit everything together. Deliberately excluding the operator's own changes strands their work uncommitted while the agent's work ships. Only exception: changes the operator explicitly asks to keep separate.

### Colocated Code over Abstractions (Hard Rule)
**Operating procedure: inline by default; abstractions only with explicit approval.** Most code should be written inline at the call site. Extract a helper/wrapper/protocol/manager ONLY when duplication is massive AND an abstraction genuinely requires it — and even then, ASK the operator before introducing it. Prefer calling methods directly at the call site rather than hiding a single expression behind a helper function. A one-line helper that wraps `ctx.sender_auth().jwt().map(|c| c.subject().to_string()).unwrap_or_else(|| ctx.sender().to_hex().to_string())` is worse than inlining it — the reader has to jump to the helper to understand what it does, when the expression itself is already readable at the call site. Simple expressions, even if repeated, are better inlined for scannability. Example: no custom HTTP-client wrapper classes around platform defaults (`URLSession`, Ktor `HttpClient`) — construct requests and check statuses inline at each call site.

**Colocate logic beside its caller (Hard Rule).** Business/data logic lives in the same folder as the UI (or other code) that calls it — feature-local by default, not in a parallel "core/common/utils" tree. Only when MULTIPLE features consume the same logic should it graduate to a shared location (and ask before creating that shared module). Do not pre-create shared/core modules "for later" — dead code ahead of its first consumer is speculative structure.

### UI File Organisation: Suffixes, Previews, Atomic Components (Hard Rule)
**Applies to every UI we write, in every stack**, including Svelte (`apps/my-website`) and any future UI. Platform-specific spelling belongs in the app's own AGENTS.md; the shape is the same everywhere:

1. **Organise by top-level feature, one folder per feature.** Logic is colocated inside the feature folder beside the UI that calls it — no per-layer folders, no core/utils tree (this is the Colocated Code rule above, applied to UI).

2. **Every UI file's filename states what it is, via a suffix.** A reader must know from the filename alone whether a file is a screen, a reusable component, a state machine, or logic — without opening it. `Screen` / `Component` / `Machine` / no suffix (logic) are the four kinds. This is the Descriptive Naming rule applied to file names, and it is what stops `FooView.swift` from silently holding five unrelated things.

3. **Every screen and component file ships a live preview** in the same file, so opening it renders something immediately. Previews are not optional polish and not dead code: they are the component's manual test and the canonical place its variations (empty / populated / error / long-content) are enumerated. A UI file without a preview is unfinished. Previews must render offline with injected fixtures — a preview that needs the network or real credentials is one nobody runs.

4. **Reusable UI lives in a single top-level `components/` folder, tiered by atomic design:** `components/atoms|molecules|organisms/`. Per Brad Frost's *Atomic Design* (https://atomicdesign.bradfrost.com/chapter-2 — the canonical source; read it, do not argue from memory):
   - **atoms** — indivisible primitives (button, icon, label, text style).
   - **molecules** — a few atoms bonded into a simple unit with one responsibility (label + field + button = a search form).
   - **organisms** — relatively complex assemblies forming a discrete interface *section* (a header, a feed row, a card).

   **The tiers are a mental model, not a build order, not a maturity ladder.** Frost is explicit that the stages work *concurrently* and that it "would be foolish to design buttons in isolation, then cross your fingers and hope everything comes together". Atoms are not "more reusable" than organisms; an organism is not a promoted molecule. Place a piece by **what it is**, never by how shared it feels. If it resists categorisation, it is usually doing two jobs — split it rather than inventing a tier.

   **Stop at organisms — do not add `templates/` or `pages/` folders.** We already have both, and they are the feature folders: a screen file *is* the page (a specific instance with real content) and its markup *is* the template (the layout skeleton). Adding those folders would create two homes for one concept, which the Colocated Code rule exists to prevent.

   **Promotion trigger is a second consumer, never anticipation** — the same rule as every other shared module. Feature-local UI stays in the feature folder until a second feature actually uses it.

   Atomic design is technology-agnostic (Frost applies it to native Instagram, not CSS) — the taxonomy is for placement and naming, not for a particular framework.

### Zero Warnings in Maintained Code (Hard Rule)
Code we maintain compiles and lints with **zero warnings** — not "acceptable" warnings, zero. A warning is a fix-item, never background noise: it either gets fixed in the same change that introduced it, or the change isn't done. This covers compiler warnings, linter violations, and generator/tool warnings emitted while processing our inputs. Warnings originating in an upstream dependency's OUTPUT (e.g. swift-openapi-generator warnings caused by anyOf-null schemas in a service's OpenAPI spec) are fixed by a PR to the upstream repo — the same contract-improvement flow as spec defects — never ignored. The bar: `mise run <app>-checks` (or the equivalent suite) ends clean, including the toolchain's warning output.

### No Lint Suppression (Hard Rule)
**Never use `#[allow(...)]`, `#[allow(unused)]`, `#[allow(dead_code)]`, `#[allow(clippy::...)]`, or any other lint suppression to silence a warning.** Warnings indicate a real problem in the code — fix the root cause, not the symptom. Suppressions hide bugs, accumulate dead code, and defeat the purpose of the linter. Every warning must be resolved by changing the code:

- **`unused variable`** — the variable is genuinely unused. Either use it (read the value, pass it somewhere) or remove it entirely. If it's unused only in certain `#[cfg]` feature combinations, gate the variable's declaration with `#[cfg(...)]` so it only exists when it's actually used.
- **`unused import`** — the import is not needed. Remove it, or if it's only used in certain feature combinations, gate the `use` statement with `#[cfg(...)]`.
- **`dead_code` / `field never read`** — the field, function, or variant is never accessed. Either use it (read the field, call the function, match the variant) or remove it. If it's only used in certain feature combinations, gate it with `#[cfg(...)]`.
- **`clippy::needless_pass_by_value` / `clippy::needless_pass_by_ref`** — change the function signature to pass by reference (`&T`) or by value (`T`) as the lint suggests, rather than suppressing it.
- **`unreachable_patterns`** — the catch-all `_` arm is needed for `#[non_exhaustive]` or `#[cfg]`-gated variants. This is the **one** acceptable `#[allow(unreachable_patterns)]` — it's not suppressing a warning about dead code, it's acknowledging that the exhaustive match is intentionally non-exhaustive across feature flags. Even then, prefer restructuring the match to avoid the catch-all where possible.

When `#[cfg]` feature gates cause a variable/function/import to be unused in some configurations: gate the declaration itself with the matching `#[cfg(...)]` (or `#[cfg(any(...))]` for multiple features), so the symbol only exists when it's used. This is the correct pattern — the code is genuinely absent in that configuration, not present-but-suppressed. Gate both the definition and all call sites consistently.

### Regression Tests for Every Bug Fix (Hard Rule)
**Every bug fix ships with tests: at least one asserting the fixed behavior, and negative tests pinning the original failure case** (the exact input that was broken, asserting the wrong output never returns). A bug that was once hit must never be silently reintroduced. Scope judgment applies — not a barrage of meaningless tests: pin the bug's essence (the boundary/shape that was wrong), not every permutation around it. Wire/protocol bugs pin the exact request/response bodies against the authoritative contract. Pure-function bugs pin inputs/outputs at the broken boundary. UI-state bugs pin the state transitions that were wrong. The regression test lands in the SAME change as the fix — never deferred.

### Check Live API Signatures Before Porting/Calling (Hard Rule)
**When calling or porting a call to any API, read the authoritative contract first — the generated bindings, OpenAPI spec, or the module's own source — instead of blindly copying/porting code from an existing caller.** Existing callers can predate an upstream migration. Port behavior, not stale wire shapes.

**API bindings come from the live contract, generated, never hand-curated (Hard Rule).** For any HTTP service we call, generate typed bindings from the service's OWN OpenAPI spec. The spec is consumed VERBATIM — no filtering to "only the endpoints we use today" and no hand-written subsets; the full API generates so future features have the whole context. Keep byte-verbatim committed contract snapshots current through the consuming service's build workflow. Spec DEFECTS (untyped responses, missing security declarations) are fixed in the owning service — not patched locally — so the contract improves for every consumer.

**Surface upstream API errors verbatim — never bury them in generic custom messages (Hard Rule).** In ALL API-consumer code (Swift data sources, Rust service clients, reducers calling external services, CLI wrappers), error paths must carry the upstream status code AND response body through to the user-facing error — the API's own words ("Name 'X' is already taken", FastAPI's `detail` array, the provider's error JSON) are the message. Never replace them with invented generic text like "unexpected response (spec drift)" or "something went wrong": it misclassifies real server failures as contract bugs and sends diagnosis to the wrong layer (observed: a live HTTP 500 from LLM generation was reported as "spec drift" — the iOS `undocumented()` mapping discarded the status and body the generated client already carried). Rules: (1) error types propagate `statusCode` + `body`, (2) response bodies are surfaced verbatim (cap buffering, e.g. 1 MB), (3) custom text is a FALLBACK only when the upstream body is unreadable, and it must include the status code, (4) `LocalizedError`/display mappings prefer the upstream body first.

### Finite State Machines for Stateful Logic (Hard Rule)
**Model anything stateful as a finite state machine — in any language, not just UI.** The trigger is not "is this a screen?" but **"does this hold more than one meaningful mode?"** If a type has a lifecycle with transitions, a phase/step/stage, a status that moves through stages, or two or more fields that must agree, it is a state machine — model it as one. Applies to Swift, Rust, TypeScript, Python, Ansible/ops code — everything.

In scope (non-exhaustive):
- **UI/screen lifecycle** — a sheet, form, wizard, or flow (which screen/step/phase are we in).
- **Application/service lifecycle** — sign-in/session state, account creation or deletion, upload/generation jobs, ETL or migration progress, retry/backoff cycles.
- **Background work** — a job with queued → running → succeeded/failed, a stream with connecting → streaming → closed, a poller with idle → polling → error.
- **Protocol/wire state** — a handshake, an auth flow, a multi-step negotiation, a cursor advancing through pages.
- **Domain entity status** — an order/row/record moving through stages, an entity that can be created/deleted/suspended.

The reference model is **XState** (https://stately.ai/docs — read the docs, not this summary, before designing a machine); we **port the model, not the library**.

**Why:** state bugs come from invalid states being *representable* — a boolean pair that should never be `(true, true)`, a phase enum plus three optionals that must agree, a "deleted" flag on an entity that is also still listed. An FSM makes illegal states unrepresentable and illegal transitions explicit, so the bug cannot be *written* rather than merely not-yet-observed. (Concrete precedent: the deleted-bot resurrection — a profile row, an id list, and an existence marker that could all disagree, with no single owner of the invariant.)

**Do not model a machine where there is genuinely no state change.** A pure function, a stateless transformation, or a value with one mode is not a state machine — wrapping it in one is ceremony. The rule is "anything *stateful* is a machine", not "everything is a machine".

**Swift and Rust default to this model for all new, rewritten, or refactored non-trivial logic**, especially stateful logic. Prefer explicit states and payload-bearing events for workflows and processing rather than scattered flags, callbacks, or ad hoc orchestration. Trivial stateless calculations remain pure functions; complexity alone does not justify inventing a lifecycle that does not exist. Apply this incrementally to touched code, not as an unrelated repository-wide rewrite.

#### The core model: a snapshot is finite state PLUS context

These are **two separate things** in XState, and conflating them is the most common modelling mistake:

| XState          | What it is                                                                                       | Swift/Rust equivalent           |
| --------------- | ------------------------------------------------------------------------------------------------ | ------------------------------- |
| `state.value`   | the **finite state** — a small, enumerable set of modes, possibly nested (`{ form: 'invalid' }`) | the **enum** (`state`)          |
| `state.context` | **extended state** — a data bag (tokens, ids, counts) that exists across *all* states            | the **data struct** (`context`) |

So a machine's snapshot is `(state, context)` — one enum saying *where* you are, one struct holding *what you know*. `context` is **immutable**; it is updated only by an `assign` action within a transition, never by direct mutation. **State-specific data belongs inside the enum variant** (via nested/compound states), never as optionals on the shared context that only make sense in some modes.

**`context` is for data that is genuinely shared across states — a machine whose states share nothing should have no context at all.** An empty `Context` struct (or one whose fields nobody reads) is speculative structure: it adds a wrapper to every return and implies shared state that does not exist. XState lets `context` be `undefined` for exactly this reason. Add it when a second state actually needs the same datum, not preemptively — the same "colocate, don't pre-create" rule as everywhere else.

```swift
// YES — finite state carries its own payload; context holds only what all states share
enum State {
    case signedOut
    case signingIn(provider: AuthenticationProvider)
    case signedIn(Session)                  // tokens + subject live HERE
}
struct Context {                         // shared across every state
    var lastActiveSubject: String?
    var deviceId: String
}
struct Machine { var state: State; var context: Context }   // snapshot = (state, context)

// NO — one flat bag where illegal combinations are representable
struct Machine { var tokens: Tokens?; var isRefreshing: Bool; var subject: String?; var provider: String? }
```

#### States are typed: atomic, compound, parallel, final

- **Atomic** — no children. A leaf mode.
- **Compound (parent)** — has children; one child is active at a time, and a parent state **must declare its own initial child**. Only *one* top-level initial state exists per machine (with several, the machine wouldn't know where to start).
- **Parallel** — all regions active **simultaneously**, each with its own initial child. One event may transition several regions at once (`target: ['.mode.dark', '.theme.custom']`). Use this instead of cross-multiplying orthogonal concerns into one enum.
- **Final** — `type: 'final'`; the machine reaches `status == 'done'` and emits an `output`. This is how a lifecycle *ends* — don't model completion as a bool.

#### Transitions are deterministic, selected deepest-first

An **event** is `{ type, ...payload }`. Given `(state, event)` the next state is always the same — that determinism is what makes the machine testable.

Selection algorithm, in order:
1. Start at the deepest **active atomic** states.
2. Take the transition if it is **enabled** — no `guard`, or its guard evaluates `true`. Guards are pure predicates (see the Pure Functions rule): they filter, they never mutate.
3. Otherwise walk **up to the parent**, repeat.
4. If nothing is enabled, **the state does not change.**

Two special forms worth knowing: a **forbidden transition** (`{}` — an explicitly empty transition) *stops* the parent search rather than continuing up, which is how a child blocks a parent's catch-all; and a **wildcard** (`'*'`, `'mouse.*'`) has the **lowest priority**, matching only when nothing else did.

In a typed port, `default:` IS the wildcard: it satisfies exhaustiveness, and it is the correct home for "no enabled transition". Do **not** enumerate no-op pairs explicitly — that is ceremony, not rigour (it adds a line whenever a state is added without forcing any real decision). Enumerate only the transitions that *do* something; the safety that matters comes from those being explicit and from the machine being total. Reserve an explicit no-op for the case where you are deliberately **blocking** a transition a parent would otherwise handle — that is the forbidden-transition form, and the comment should say so.

#### Actions: the only place side effects and context changes happen

- **`assign`** updates `context` — declaratively in the transition, never by mutation. Assignments are computed from `({ context, event })`.
- **Entry / exit actions** run on entering/leaving a state node (`entry: [...]`, `exit: [...]`) — not only on transitions. Use exit actions to tear down what an entry action set up.
- **`raise`** sends an event to the machine itself, queued internally and processed FIFO **after** the current transition completes; external events are only handled once that internal queue drains. This is how you sequence multi-step work without side effects mid-transition.

#### Actors: invoked vs spawned

A running machine **is an actor** — it owns its state, processes **one event at a time** from a mailbox, and shares state with others *only* by sending events (never by shared mutable memory). Two lifecycle flavours:

- **Invoked** — started when the parent *enters* the invoking state, stopped when it *exits*. Use for a fixed, state-bound side task (an in-flight request, a stream).
- **Spawned** — started dynamically in a transition, stopped explicitly (`stopChild`) or when the parent stops. Use for a dynamic number of children.

`context` holding an in-flight task's handle is the normal pattern — the actor's lifecycle is then tied to the state that owns it, not to a stray callback.

#### Pure transition functions

XState exposes `initialTransition(machine)` and `transition(machine, state, event)` — **pure** functions returning `(nextState, actions)`. Port this: the machine's `transition` decides the next state and *returns* the effects; the caller performs the I/O. That makes every transition unit-testable with no network, no database, no clock.

#### Swift and Rust implementation conventions

- **Pure kernel, thin actor runtime.** Represent finite states and events as enums with associated payloads. Compute `(nextSnapshot, effects)` from `(snapshot, event)` synchronously and without I/O; when there are no effects, returning only the next snapshot is sufficient. The runtime installs the returned snapshot and executes typed effects. Success, failure, cancellation, and timer results return as payload-bearing events, never as direct state mutation. Supply time, randomness, and external results as inputs rather than reading them inside transitions. Initialization must also account for initial entry effects when they exist. See https://stately.ai/docs/pure-transitions.
- **Keep case arms short.** Once a `switch`/`match` arm exceeds roughly 8-10 lines of logic, extract that logic into a colocated pure function and accompany it with unit tests. Keep transition selection visible in the arm. This specific extraction is pre-approved under the Colocated Code rule; it does not authorize generic wrappers, managers, or shared utility modules.
- **Swift concurrency uses `actor`, not a shared mutable struct.** A non-concurrent machine may be a value-type struct returning a new snapshot. A concurrent lifecycle is owned by a Swift actor whose event handler delegates to the same synchronous pure kernel. Commit the transition before any `await`: Swift actors are reentrant across suspension points, so actor isolation alone does not guarantee that an entire asynchronous event handler runs uninterrupted. Perform I/O outside the pure transition and send its result back as an event. Use `Sendable` payloads and snapshots across isolation boundaries.
- **Rust concurrency uses channels and task-owned actors.** Prefer typed channels carrying event enums with attached payloads. One owning task/thread receives events and applies the pure transition sequentially; other tasks communicate through channels instead of shared mutable machine state or mutex-protected state bags. Execute asynchronous effects without blocking the event receiver where cancellation or other events must remain responsive, and send completion/error events back to the owner. Choose channel bounds and backpressure deliberately, and handle receiver closure and actor shutdown explicitly.
- **Own asynchronous work by lifecycle.** Cancel invoked work on state exit and stop spawned work explicitly or on parent shutdown. Correlate asynchronous results with a request identifier or generation when overlapping requests or late completions are possible; ignore stale events through pure transition logic. Native task cancellation is cooperative, so cancellation alone is not proof that a completion cannot arrive. See https://stately.ai/docs/actors.
- **Do not duplicate derived data.** Prefer computing values such as password length or a minimum-length predicate from the canonical password instead of storing fields that can disagree with it. If caching is necessary, update all dependent values in the same pure transition. A length threshold is a length policy, not a complete password-strength assessment.
- **Test decisions and runtime separately.** Unit-test initial snapshots, every meaningful transition, guards, emitted effects, invalid/no-op events, and relevant failure/cancellation/stale-result cases without services or real clocks. Test actor/channel integration separately for event delivery, isolation, cancellation, and shutdown. Returning a new value does not update the old one: callers of a value-type machine must retain the returned snapshot.

#### Corollaries
- **One typed `transition(_ event:)` per machine.** No direct mutation from views, data sources, or callbacks — they send events. Everything stateful flows through it.
- **Every transition gets a test.** Drive the machine through the event that was previously mishandled and assert the resulting `(state, context)`. This is the state half of the Regression Tests rule.
- **Model the real lifecycle, not the screen.** One machine per lifecycle (session, account deletion, upload job, ETL cursor, generation job) — callers read the state and send events. A machine is not inherently a UI object; service/background/domain lifecycles get one too.
- **Wrap in an `actor` (Swift) / task-isolated type (Rust) where there is concurrency** — in-flight requests, streams, timers — rather than sharing mutable state across call sites.
- **A boolean flag pair that must agree is a machine that hasn't been written yet.** Two `Bool`s, or an enum plus optionals describing the same concern, are the trigger — collapse them into one enum whose variants carry their own payload.
- **Gradual migration.** Convert when touching a file; do not big-bang rewrite. Existing flat-state types get folded in as they are next modified.

### Pure Functions & Thin API Wrappers (Hard Rule)
All business logic must be implemented as **pure functions** — no I/O, no side effects, no external service calls. Functions that call external APIs (HTTP, database, KV store, etc.) must be **thin wrappers** that delegate to pure functions for all logic. This applies to all application code (Rust, Kotlin, TypeScript, etc.):

- **Pure functions** take inputs, return outputs, and have no side effects. They contain all business logic: validation, transformation, computation, decision-making.
- **Thin wrappers** handle only I/O: calling the external API, passing the result to a pure function, and returning/writing the result. No business logic in wrappers.
- **Unit tests** must be written for all pure functions. Thin wrappers are tested via integration tests (which may require external services).
- When touching existing code that mixes logic and I/O, refactor to separate the two in the same change.

Example pattern:
```rust
// Pure — fully testable
pub fn build_token_claims(user_id: &str, issuer: &str, ...) -> AccessTokenClaims { ... }

// Thin wrapper — just I/O + delegation
pub async fn generate_token(ctx: &ServerCtx, user_id: &str) -> Result<String, Error> {
    let claims = build_token_claims(user_id, &ctx.server_url, ...);
    Ok(jsonwebtoken::encode(&header(), &claims, &ctx.encoding_key)?)
}
```

### Workspace Dependency Versioning (Hard Rule)
All dependency **versions** must be declared once in the root `Cargo.toml` `[workspace.dependencies]` section — never specify versions in individual member `Cargo.toml` files. Member crates use `{ workspace = true }` (optionally adding `features = [...]` if needed). This keeps all members in sync and avoids version drift. If a new dependency is needed, add it to `[workspace.dependencies]` first, then reference it with `workspace = true` in the member.

**Features belong on the consumer, not the root (Hard Rule):** The root `[workspace.dependencies]` entry declares only the **version** (and any `default-features = false` or `path = ...`/`git = ...` source specification). **Never** put `features = [...]` in the workspace root entry — specify features on each member crate's `{ workspace = true, features = [...] }` line instead. Different consumers need different feature subsets (e.g. `jsonwebtoken` `aws_lc_rs` + `use_pem` for crates that mint/verify JWTs, none for crates that don't use crypto); forcing a single feature set from the root pulls unwanted deps into crates that don't need them and makes feature intent invisible at the consumer site.

### Shared vs Module-Local Constants
Maintain a top-level `constants.rs` beside `lib.rs` for constants shared between multiple modules within a crate. Module-specific constants stay at the top of the module file that uses them. Only promote a constant to `constants.rs` when at least two modules reference it.

## Repository-Specific Patterns & Rules

### Hetzner + Provisioning
- `provision` role uses Hetzner Robot API + installimage (Ubuntu).
- OS partition full disk on all nodes (via `provision_os_partition_size: "all"`). Longhorn uses the btrfs filesystem (expanded to both NVMe drives via RAID0).
- Node add/remove operations regenerate inventory-owned DNS through `pdns-zones`; commit and push the generated declarations for Flux and external-dns to converge.
- `namecheap-dns` role + `update-namecheap-dns.yml` playbook manages custom nameserver glue records at Namecheap from `namecheap_nameserver` per-host annotations in `hosts.yml`.

**Nameservers live on the control planes** (node-1..node-6 — `ns1`/`ns4`/`ns5`/`ns8`/`ns9`/`ns10`), not on workers. The annotation drives all three copies of the address list — the registrar glue records, the zone's NS records, and the PowerDNS Service `externalIPs` — so they cannot drift. A nameserver needs only its address bound to `:53`; Cilium routes that to whichever node hosts a PowerDNS pod over the VXLAN tunnel, so **no PowerDNS pod need run on the annotated node** (verified: the previous worker-based nameservers answered queries while running no pod at all). The prefixes are reused across the move rather than renumbered, which keeps each zone's NS set byte-identical and makes the change a glue-IP update; re-point a prefix, never just delete one, because the role has no glue-record delete path.

**Full init flow (never partial):** follow the complete node lifecycle playbook, then regenerate inventory-owned DNS and push its declarations. Do not manually update cluster DNS at Cloudflare.

### Kubernetes Cluster
- kubeadm, stacked etcd, 10 stacked CP nodes: node-1..node-7 (Hetzner — 5 Helsinki, 2 Falkenstein) + 3 Azure Central India.
- All CP nodes run both control plane and workloads (`stacked_control_plane: true` per host in inventory).
- etcd quorum=6, tolerates 4 node failures. The three Azure members are full voters, not learners — their commit latency is ~900ms against 57–287ms for the Hetzner members, because the India↔Europe RTT is ~146ms. That is the cost of the cross-region placement, and it is why the timeouts are 250/2500 (see `ansible/inventory/group_vars/all/vars.yml`): etcd's 100/1000 defaults cannot be kept once a WAN member is in the quorum.
- Control plane HA uses the inventory-derived `kubernetes-api.saikat.dev` DNS record. Node lifecycle DNS converges through `pdns-zones` and Flux, not Cloudflare writes.
- Cilium + WireGuard encryption. Gateway API for exposure.
- Serial node operations.
- CoreDNS topology: see `kubernetes/infrastructure/coredns/coredns-*-topology.yaml` (non-Flux, kubeadm-owned). Run `kubectl apply -f` after adding workers in new zones.
- Storage (Longhorn): see `kubernetes/infrastructure/longhorn/helmrelease.yaml` for version and settings. Default `longhorn` SC (2 replicas, LUKS2 encryption, `dataLocality: best-effort` for local primary replica). `longhorn-1replica` SC for workloads with app-layer replication. `storage-longhorn` role handles btrfs RAID0 expansion + Longhorn data dir creation on all nodes.
- Node naming convention: `node-N` for all cluster nodes (control plane + workers). Workers still named `worker-N` until reprovisioned.
- **kubeadm ClusterConfiguration is declarative only:** the single source of truth is `ansible/manifests/kubeadm-cluster-configuration.yaml.j2`, rendered at bootstrap and uploaded before control-plane joins. Component extraArgs are config-file-only in kubeadm v1beta4. ConfigMap edits do not propagate to running static pod manifests. The historical metrics backfill entry point is retired; future changes converge through serial node replacement using the remove/add lifecycle, with surviving etcd quorum verified before each removal. Never patch the live ConfigMap as a substitute.
- **`apiServer.certSANs` changes require new certificates:** editing the declaration does not change existing certificates, and `kubeadm certs renew` preserves existing SANs. Node replacement remains the default; an explicitly approved, one-time SAN-only reissuance may use `ansible/playbooks/operations/reissue-api-serving-certificates.yml`. It stages and validates candidates with the existing CA, activates one API server at a time, and restores the original pair and manifest on failure. This does not authorize OS changes, node reset or etcd membership changes. **Adding a SAN must be additive**: preserve existing names and addresses, verify every serving endpoint before DNS/client cutover, and retire old names only after every client has moved. Recurring in-place backfill remains retired.

### Storage Replication Policy

**Default: Longhorn 2-replica** (`longhorn` StorageClass). Use for ALL stateful workloads that do NOT have their own app-layer replication — Loki, Prometheus, Harbor, PowerDNS, GeoIP. This gives storage-level HA (tolerates 1 node failure) with LUKS2 encryption at rest.

**Exception: Longhorn 1-replica** (`longhorn-1replica` StorageClass). Use ONLY for workloads with app-layer replication that is strictly stronger than Longhorn 2-replica. All three criteria must be met:
1. The application implements its own replication natively (not just backups)
2. App-level replication is strictly stronger than Longhorn 2-replica (tolerates more failures)
3. The write amplification from layering both is measurable and unacceptable

Workloads currently using 1-replica:
- **CNPG PostgreSQL** (streaming replication — primary + replicas, each with their own PVC)

Do NOT use 1-replica for workloads without app-layer replication. Losing the single node that holds the data means permanent data loss.
- PostgreSQL (CloudNativePG): see `kubernetes/infrastructure/cloudnative-pg/helmrelease.yaml` for the per-service model and conventions.

### etcd Shares the Root Disk with Longhorn — Bound Longhorn's I/O (Hard Rule)

**etcd's WAL lives on the same 2 TB root disk (`nvme0n1`) as Longhorn's replica data (`/var/lib/longhorn`). That sharing is the intended layout, and it is fine — the rule is not "these must be on separate disks", it is that Longhorn's I/O must stay bounded, because etcd's commit path is only as fast as its co-tenant allows.** The VM has exactly one persistent disk; `nvme0n2` (32 TB) is already dedicated to rustfs. There is no spare disk to relocate to (see below), so the control is the I/O budget, not the placement.

Measured 2026-10-10: a Longhorn snapshot-integrity + replica-rebuild burst drove instance-manager writes to ~300 MiB/s and 52–61 % disk utilisation, pushing etcd WAL fsync p99 from a **7.5 ms baseline to 6,032 ms** (the histogram saturates at 8192 ms — it hit that too, at 12:02–12:08). etcd's own guidance is that heartbeats begin timing out when these writes take too long, which is exactly what happened.

The failure mode is a **whole-cluster restart storm, and it looks like a dozen unrelated bugs**. The etcd stall forced leader elections (`etcdHighNumberOfLeaderChanges` 6.4/15 m), failed 100 % of LeaseGrants, took down the aggregated `metrics.k8s.io` API (`KubeAggregatedAPIDown`), and — because the apiserver's own etcd client retried until it exhausted — restarted *every* long-lived controller in lockstep (CNPG 148, shipwright 128, kube-state-metrics 126, flux, csi-*). The same burst reached 508 ms the day before, killing apiserver-1. **etcd itself restarts 0 times** (it needs no API round-trip), which is the tell: when many unrelated controllers share a restart timeline but etcd does not, suspect disk contention, not their code.

Rules:
- **Keep Longhorn's I/O bounded.** `snapshotDataIntegrity` stays at the default `fast-check` (not `enabled`, which re-hashes every snapshot file), and keep `concurrentReplicaRebuildPerNodeLimit` / `snapshotHeavyTaskConcurrentLimit` low — the chart default of 5 let five concurrent rebuilds/purges saturate the device.
- **Correlate before diagnosing.** `histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))` against `rate(container_fs_writes_bytes_total{namespace="longhorn-system"})` — if they spike together, it is disk contention, not the crashing component.
- **The `nvme1n1` / `nvme2n1` (440 GB) devices are not ours.** They are unformatted, unmounted, absent from `/etc/fstab`, and missing entirely on node-1 — they are Azure-internal, not provisioned by us. Do not build anything on them, and do not cite them as a "spare disk" for a future storage-isolation change.

### CNPG Superuser Password Management (Hard Rule)

**Never `ALTER USER postgres PASSWORD ...` on a live CNPG database.** The password is managed declaratively via the CNPG Cluster CR spec:

- `enableSuperuserAccess: true` — the operator generates/syncs the `postgres` superuser credential from the `superuserSecret`. With it `false` (the default), **the operator sets the `postgres` password to `NULL`, disabling password auth for that role.** Measured on `harbor-db` (2026-10-02): flipping the flag to `false` took `postgres` from a working password to `rolpassword IS NULL` — verified via `pg_authid` and by a TCP auth attempt that went from succeeding to `fe_sendauth: no password supplied`.
- `superuserSecret: name: <secret-name>` — references a Secret of type `kubernetes.io/basic-auth` (or `Opaque` with `username` + `password` keys) containing the desired superuser credentials.
- To change the superuser password: update the SOPS-encrypted secret in git, ensure `enableSuperuserAccess: true` is set on the Cluster, push, and let the CNPG operator reconcile the new password into the database.

**! None of that makes the superuser Secret the winner for a *running client* — and believing it does is how a six-hour outage started.** Measured on `harbor-db` (2026-10-02), with `enableSuperuserAccess: true` and a correctly-declared `superuserSecret`:

| Credential                       | TCP auth result                                                   |
| -------------------------------- | ----------------------------------------------------------------- |
| the `superuserSecret`'s password | **FAILED** — `password authentication failed for user "postgres"` |
| the `-app` Secret's password     | **succeeded**                                                     |

The superuser Secret's value was never synced into the database, and its password did not match `pg_authid`. So **never assume which of a cluster's `postgres` credentials the server actually accepts** — a `psql` without `-h` cannot tell you either, because it connects over the Unix socket where CNPG trusts any password and will report *both* as valid. Test with `psql -h 127.0.0.1 -U postgres ...` and compare against `pg_authid.rolpassword`.

**The real rule: do not share a role between the application and the superuser.** The root cause of that outage was `initdb.owner: postgres`, which made the application role and CNPG's superuser the **same role** — two credential sources owning one password, with the client reading the one that drifted. A dedicated low-privilege app role removes the entire class of bug: nothing in the app's path then shares a name with `postgres`, so a superuser rotation (or toggling `enableSuperuserAccess`) cannot invalidate the app's credentials. This is the pattern to follow, and `harbor-db` now matches it:

- `initdb.owner: <app>` — a dedicated role, never `postgres`. See `kubernetes/infrastructure/external-dns/cnpg-cluster.yaml` for the worked example and the full reasoning.
- Declare that role with a standalone `DatabaseRole` (`passwordSecret` + `databaseRoleReclaimPolicy: retain`), which is CNPG's recommended GitOps approach and means the operator, not a hand-written Secret, owns the password. Worked example: `kubernetes/infrastructure/harbor/databaserole.yaml`.
- **Only enable `enableSuperuserAccess` if something genuinely performs administrative work** — a backup plugin, an operator, manual DB maintenance. If nothing does, omit it entirely (as `pdns-db` and `harbor-db` both now do), because the superuser role is what the app must then be kept away from.
- Adding `enableSuperuserAccess` to "fix" an auth failure is the wrong move — it *creates* the ambiguity described above. The fix is to stop sharing the role.

### CNPG Application Role (Hard Rule)

`initdb.owner` provisions a **low-privilege application role** that owns the database, and the operator generates an `-app` Secret for it. Use it — do NOT reach for the superuser for application connections.

- **Prefer `-app` over `-superuser` for every application.** The `-superuser` Secret is administrative only and is generated ONLY when `enableSuperuserAccess: true`. Without it, CNPG sets the `postgres` password to `NULL`, so any client authenticating as `postgres` gets `FATAL: password authentication failed`.
- **The application role must be a distinct role, not `postgres`.** `initdb.owner: postgres` makes the app role and the superuser the same role, so two credential sources own one password and a client reads whichever drifted — that is the outage described under "CNPG Superuser Password Management". With a distinct `owner`, `enableSuperuserAccess` and the superuser Secret become irrelevant to the app, and can be omitted.
- **`enableSuperuserAccess` does NOT reliably make the superuser Secret authoritative for a client.** Measured on `harbor-db` (2026-10-02): with the flag `true`, the `superuserSecret`'s password failed TCP auth while the `-app` Secret's succeeded — so the operator had not synced the declared secret into `pg_authid`. Do not reason from which secret "should" win; test over TCP (`psql -h 127.0.0.1`) and read `pg_authid.rolpassword`. Adding `enableSuperuserAccess` to "fix" an auth failure is the wrong move — the fix is to give the application its own role.
- **`postInitApplicationSQLRefs` always runs as the superuser**, regardless of `initdb.owner`. SQL it applies (e.g. from a schema ConfigMap) therefore creates **superuser-owned tables**, and the app role then hits `permission denied for table ...` on its first write. Pair any schema ConfigMap with a grants ConfigMap that transfers ownership (`ALTER TABLE ... OWNER TO <app>` / `ALTER SEQUENCE ...` / `ALTER SCHEMA public OWNER TO <app>`). List the schema ConfigMap first in `configMapRefs` — ordering is Secrets first, then ConfigMaps, each in array order.
- Verify ownership directly before wiring the client: `SELECT tablename, tableowner FROM pg_tables WHERE schemaname='public';`. Sequence names are easy to miss — list them from the live database rather than guessing.

### Backups
Use Longhorn S3 volume backups for workloads without application-native recovery,
and the CloudNativePG Barman Cloud plugin for PostgreSQL base backups and WAL
archiving. Schedules and retention belong in the owning manifests. Velero is
retired; do not reintroduce it without approval. RustFS runs inside this cluster
on separate data disks, so this policy provides localized recovery, not an
independent off-cluster disaster-recovery copy.

### Networking & Exposure Model
- Everything exposed via Cilium Gateway (Gateway + HTTPRoutes or TLSRoutes).
- DNS: wildcard + all-node A records (all workers + CPs). Adding an HTTPRoute is sufficient; no DNS change needed. See `kubernetes/networking/gateway.yaml` and `kubernetes/infrastructure/external-dns/dnsendpoint-saikat-dev.yaml` for details.
- **No L2 Announcements:** Cilium L2 announcements are intentionally disabled. L2 ARP/NDP only works within a single L2 broadcast domain — our cluster spans multiple DCs and will include nodes from other providers. The only network requirement for adding nodes is internet connectivity with a static IP. Service exposure uses `externalIPs` (L3, cross-provider via VXLAN tunnel) and LoadBalancer services. Do not enable `l2announcements` without revisiting this constraint.
- **ReferenceGrant (cross-namespace routes):** When an HTTPRoute/TLSRoute in namespace A references a Service in namespace B, a `ReferenceGrant` must exist in the **target** namespace (B) allowing it. Without it, Cilium Gateway silently returns HTTP 500 for all requests (`ResolvedRefs: False, RefNotPermitted`). Always add the ReferenceGrant in the same commit as the HTTPRoute when they span namespaces.
- **NetworkPolicy rule:** Never rely on `fromCIDRSet`/`namespaceSelector`/`podSelector` for gateway-originated traffic (cilium-envoy runs hostNetwork; appears as node IP). Use an explicit ingress rule with only a `ports:` clause (app-level auth does the real enforcement).
- **SOPS encryption mechanism:** To encrypt `*.sops.yaml` files, use the mise tasks: `mise run sops-encrypt -- <file>.sops.yaml` (encrypts in-place) or `mise run sops-edit -- <file>.sops.yaml` (opens in `$EDITOR`). These tasks extract the age key from Ansible vault automatically. The `.sops.yaml` creation rules in repo root handle age key selection automatically — no need to pass `--age` explicitly. For decryption, use `mise run sops-decrypt -- <file>.sops.yaml`. To get the raw age key for ad-hoc use: `mise run ansible-vault-view -- vault_age_private_key`.
  - Remove a secret key without exposing plaintext with `mise run sops-edit -- unset <file>.sops.yaml '["stringData"]["KEY"]'`. SOPS updates encryption and the MAC atomically; never delete ciphertext lines manually. See [SOPS common operations](https://getsops.io/docs/usage/common-operations/#unset-a-sub-part-in-a-document-tree).
- **Editing SOPS files (decrypt → edit → re-encrypt workflow):** To add/modify keys in an existing `*.sops.yaml` file without an interactive editor: (1) `mise run sops-decrypt -- --in-place <file>.sops.yaml` (decrypt in-place), (2) edit `<file>.sops.yaml` with VS Code edit tools (`replace_string_in_file`, etc.), (3) `mise run sops-encrypt -- <file>.sops.yaml` (encrypt in-place). This works reliably every time — no `sops --set` JSON syntax issues, no MAC integrity errors, no temp files to clean up. Never commit the file while it's in the decrypted (plaintext) state.
- **Create the whole file, then encrypt it in ONE pass (Hard Rule).** Write the complete plaintext YAML first — including `apiVersion`, `kind`, `metadata.name`, `metadata.namespace`, `type`, and the secret values — and run `sops --encrypt --in-place` over the finished file exactly once. **Never create the file, then post-process it** (e.g. template a placeholder and substitute the real value with a script afterwards): the second write rewrites an already-SOPS-encrypted document, and the tree-walker ends up with a half-processed tree where some sibling keys are `ENC[...]` and others are still plaintext. Measured on `harbor-role-secret.sops.yaml` (2026-10-02): that produced `metadata.name` encrypted while `metadata.namespace` stayed plaintext, and Flux reported `decryption failed for '<encrypted-value>'` / `failed to decrypt and format 'harbor/<encrypted-name>' ENC[...]` on every reconcile. For a password placeholder, run `mise run sops-edit -- <file>.sops.yaml` after the first encryption instead — `sops` re-encrypts the whole document consistently.
  - **! `sops --decrypt` succeeding proves nothing.** A malformed file decrypts perfectly locally and with the cluster's own key, because the failure is in how *kustomize-controller* walks the tree, not in the ciphertext. Do not treat a clean local decrypt as evidence the file is usable by Flux.
  - **! The visible tell is inconsistent sibling encryption** — one key under a block `ENC[...]`, the next in plaintext. A correctly encrypted file never mixes them, and the mix is visible without decrypting anything.
  - **? Unresolved:** whether a *cleanly* generated file that encrypts `apiVersion`/`kind`/`metadata.name` (rather than only `data`/`stringData`) also builds under Flux is **not established**. Fully-encrypted files do reconcile elsewhere in the repo (e.g. `infrastructure/loki/loki-s3-credentials.sops.yaml`), so "encrypting the keys breaks the build" is **not** a rule — do not cite it. A regenerated-but-fully-encrypted revision of this same file also failed, while its plaintext-keys successor succeeded, and no difference between them has been identified that explains the outcome. If a secret must be fully encrypted, verify it through an actual Flux reconcile before relying on it; when in doubt, encrypt only `data`/`stringData` (which every file in `infrastructure/harbor/` does) and sidestep the question.

### DNS (PowerDNS + external-dns)

DNS is self-hosted, not Cloudflare. PowerDNS Authoritative runs from `kubernetes/infrastructure/external-dns/` with the `gpgsql` (PostgreSQL) backend on a CNPG cluster, so every replica reads and writes one shared record store — the previous LMDB backend was single-writer (`Multiple instances: No` in PowerDNS's backend matrix) and pinned DNS to one node. `external-dns` owns records via RFC2136 dynamic updates.

**Component detail lives on the manifests, not here** — which zones are served, the per-zone setup, record types that cannot go through external-dns, the re-run procedures, and every measured gotcha are documented as comments on the manifests under `kubernetes/infrastructure/external-dns/`, starting with `kustomization.yaml`. Read those before changing DNS.

**One generalisable rule worth knowing before you touch DNS: a wrong answer here looks exactly like success.** DNS changes fail quietly in several independent ways — an omitted directive, a flag with the wrong separator, a config change that never reaches a running pod, a record type that is accepted but discarded. In every case the config reads correctly, nothing errors, and the tool reports it is up to date. So never conclude a DNS change worked because a tool said so: verify against the authoritative source (`dig @<nameserver>`, `whois`, `dig +trace`) rather than a recursive resolver, and read the live state rather than the declared state. This burned several wrong conclusions in a single session, including one that was reported as verified when it had actually succeeded for an unrelated reason.

**When changing which addresses a delegation points at, the cache window is the PARENT (registry) NS TTL, not the zone's own.** Measured 2026-10-02: the parent TLD serves the delegation at TTL **10800 (3h)** for `.dev` and **3600 (1h)** for `.ai`, while the child zone's own NS records are the usual 300. A resolver that cached the delegation from the TLD keeps the old address set for the *parent* TTL, so bounding a cutover by the 300 in the zone retires those addresses hours too early — and a retired address is a nameserver that times out, with nothing in the cluster reporting it. Keep both address sets bound (the Service `externalIPs` is the one place they need to overlap) for at least the parent TTL, and never feed the outgoing addresses into the glue records or the NS set, or the transition becomes permanent.

**Apex records are owned by `pdnsutil`, never by external-dns.** external-dns structurally cannot manage a zone apex: it never writes an ownership marker for an apex name (the name would be valid — `a-saikat.dev` — it is simply never emitted), so `plan.go` drops the apex from **both** creates and updates, and no record type can ever be converged there through a DNSEndpoint. The failure is silent — external-dns logs "Reconciliation succeeded". Tracked upstream as kubernetes-sigs/external-dns#6125 (with our independent confirmation on the issue). Consequence: the bare apex A + NS rrsets for every zone live in the GENERATED `kubernetes/infrastructure/external-dns/pdns-apex-records.yaml`, applied by the `pdns-zones` role with `pdnsutil rrset replace`. That role runs as the second play of `add-node` / `remove-node`, so node churn converges the apex automatically — a new node reaches the apex A records, and a removed node drops out of them. Its ConfigMap and Job are named by a content checksum, so a changed node set is a NEW Job that Flux creates (and prunes the previous one): no manual `kubectl delete job`, no `--force` on the Kustomization. **Never** declare an apex record in a DNSEndpoint (a declaration nothing acts on), and **never** hand-write the apex records or their ownership marker — regenerate from the inventory instead.

### Browser Browsing & Control

- For all web browsing and browser control, always use Chrome DevTools MCP
  connected to the user's installed Chrome and already-open profile. Do not use
  Computer Use or VS Code's embedded browser for browser work. Computer Use is
  reserved for other desktop apps. If Chrome MCP is unavailable, diagnose its
  connection or report the blocker; do not fall back to another UI tool.
  Reuse existing tabs where possible and verify the connection, intended profile,
  and website account in the current agent session before acting.
- Operator confirmed on 20261028 that installed Chrome, the intended open profile,
  remote debugging, and local MCP setup are already enabled. Do not ask the user
  to repeat setup or sign in to a fresh profile when MCP exposes only blank or
  sign-in tabs. Investigate MCP routing and attachment to the existing local
  profile first; this confirmation is not proof of the current connection.
- If the intended Chrome profile is unavailable, report the blocker and ask before
  switching browsers or starting a separate profile. A blank tab or sign-in page
  is not proof that the user's intended open profile is connected.
- Connect through a local user-profile MCP server on the machine running Chrome;
  installing a server on the remote VS Code host does not grant local browser
  access. Never copy profiles, export cookies, or expose debugging ports publicly.
- Enter passwords and OTPs directly in the browser, never through chat. Browser
  access does not waive approval requirements or production mutation restrictions.

### Outbound Communication - Drafts Only

- Do not create a Markdown draft file for every communication. Create one only
  when the user explicitly asks for a local file or an archival copy.
- Requests to send, reply, forward or message authorize preparing an unsent draft
  directly in the intended browser text field or mailbox draft. Verify the
  sending account, recipient or space, and attachments before entering the text.
- Leave the draft available for the user to review and send. Never click Send,
  Post or Submit, or use a keyboard shortcut that transmits the message, even
  after approval of the text. The user performs that action.
- This applies to email, Google Chat, SMS and ticketing; it does not authorize
  other changes to the outbound system or bypass production mutation restrictions.
- Rishi issue drafts belong in the matching `Ole-Ole-AI` repository:
  `oleole-api` for service/API work and `swarm-infra` for shared infrastructure.

### Local Environment & Parity
The root `mise.toml` is the single source of truth for tool versions, plaintext environment values, age-encrypted local secrets, and daemon definitions. Task definitions live in owner-scoped `mise/tasks/<owner>.toml` files, selected explicitly by root `[task_config].includes`. Do not add per-application tooling or secret configuration files. Keep application-specific values beside their existing root declarations.

**Linux and macOS compatibility is required for shared workflows.** Windows development runs this repository inside a Linux guest; macOS runs the full stack, including native iOS development. Verify each workflow's actual CI runner rather than assuming Linux; Xcode/iOS checks require macOS. Validate shared bootstrap changes on both Linux and macOS. Use portable shell commands or explicit platform branches; Linux-only validation does not establish macOS compatibility.

**User dotfiles live in `~/.dotfiles`, never as a cluster-repo submodule.** Root `[bootstrap.repos]` declares the checkout. `mise run bootstrap` invokes `dotfiles-setup` after provisioning the SSH identity; the standalone task uses pinned GitHub host keys and applies only `~/.gitconfig` from the dotfiles repo's own declaration. Other machine-wide changes require explicit review in that repo. Use [mise repository bootstrap](https://mise.jdx.dev/bootstrap/repos.html) and [targeted dotfile application](https://mise.jdx.dev/cli/dotfiles/apply.html); never copy Git identity values into this repo.

**mise (not direnv)** manages tools and environment values. Local secrets are native age values under root `[env]`, committed as ciphertext. No `.env` file is generated or loaded.

For PyPI tools requiring a different Python minor, use mise's per-tool `python` option rather than downgrading the project interpreter or overriding installer arguments. This declares a private runtime dependency; refresh and commit `mise.lock` with the change. See [mise's interpreter selection](https://mise.jdx.dev/dev-tools/backends/pypi.html#choosing-python).

**Never declare tool-standard credentials repo-wide (Hard Rule).** Native age values become ambient in activated shells and child processes; redaction is not process isolation. Store credentials under descriptive names such as `CLUSTER_GITHUB_TOKEN`, `CLUSTER_CLOUDFLARE_API_TOKEN`, `CLUSTER_GEMINI_API_KEY`, and `HETZNER_S3_SECRET_ACCESS_KEY`. Map them to standard names such as `GH_TOKEN` or `AWS_SECRET_ACCESS_KEY` only in the consuming task or command. CLIs often prefer environment credentials over saved accounts, so global standard-name exports can silently hijack unrelated tooling. Inspect ambient variables before diagnosing an authentication failure.

**Local age secrets and Kubernetes SOPS use different identities.** Local mise values use the infrastructure SSH ed25519 identity extracted by `mise run bootstrap` into the gitignored `.saikat-machine-access-ed25519`. Its source is `vault_saikat_machine_access_ssh_private_key`; `age.ssh_identity_files` selects it. Do not substitute a personal SSH key or commit the private identity.

For a single-line secret, encrypt with hidden input and an explicit infrastructure recipient:

```sh
mise set --age-encrypt --prompt --age-ssh-recipient "$(ssh-keygen -y -f .saikat-machine-access-ed25519)" SECRET_NAME
```

For multiline or Base64 input, use `--stdin` instead of `--prompt`, keeping plaintext out of command arguments and logs. Explicit recipient flags prevent accidental encryption to additional personal keys discovered by mise. Use application prefixes when the same credential name would otherwise describe different values.

`age.strict = false` permits fresh checkout and CI setup before bootstrap provisions the identity. Secret-consuming tasks must check the values they require with `mise run secrets-check -- <KEY>...`; skipped decryption is not proof a credential is available. Optional empty fallbacks belong at their consumers. Never print `mise env`, `mise set <KEY>`, or a whole process environment when secrets may be present; validate availability or compare values in memory instead.

**Key 2 — SOPS (native age key):**
- SOPS-encrypted `*.sops.yaml` files under `kubernetes/` use a **separate native age key** (not the SSH key).
- The age private key is stored in Ansible vault as `vault_age_private_key` (a standard age key with `AGE-SECRET-KEY-...` format).
- The public recipient is declared in `.sops.yaml`; do not duplicate it in local secret configuration.
- To extract and use the age key: `mise run ansible-vault-view -- vault_age_private_key` → write to a temp file → `SOPS_AGE_KEY_FILE=/tmp/sops-age.key sops --decrypt <file>.sops.yaml`. Or simply use `mise run sops-decrypt -- <file>.sops.yaml` which handles this automatically.
- **Never edit SOPS files outside of `sops`** — use `sops <file>.sops.yaml` to edit, or decrypt → edit → re-encrypt. Manual edits break the SOPS MAC integrity check.
- When importing local values from SOPS, keep plaintext in memory or stdin and encrypt it as a native mise age value with the intended recipient.

**Shared:**
- Ansible vault remains the source for cluster-ops secrets; roles consume `vault_*` variables directly. This local-secret consolidation does not migrate Kubernetes SOPS or Ansible vault into mise.

**Repeated workflow tasks live in `mise/tasks/<owner>.toml`, not bash scripts.** Reusable setup, build, test, and run workflows are `mise` tasks (`mise tasks` to list). `scripts/setup-local-env.sh` is a thin `exec mise run setup` wrapper for compatibility only — do not add logic there. New recurring workflow steps belong in a new or updated mise task; one-off commands do not require one. Never create bespoke bash scripts for repo workflows.

**Task catalogs use mise's [included TOML format](https://mise.jdx.dev/tasks/task-discovery.html#included-toml-files):** write `[<task-name>]`, not `[tasks.<task-name>]`. Preserve app-prefixed task names. Included tasks inherit the root tools, environment, variables, and project working directory; retain explicit `dir` overrides. Register new catalogs in root `task_config.includes`; top-level `include` does not accept tasks. Code inspecting task declarations must read the selected catalogs rather than assume root `[tasks]` exists.

Do not create task catalogs to assert snapshots of deployment configuration. Keep executable-logic regression tests beside their owning code and run them in CI; retain reusable release-contract and dependency-discovery validation.

**Prefer existing mise tasks for recurring workflows.** Use the corresponding `mise run` task when its environment, secrets, dependency chain, or safety checks are needed. One-off tooling commands may run directly with the necessary prerequisites; do not create a task just to run them once.

Use `my-website-build`, `my-website-run`, and `my-website-stop` for the maintained application; inspect daemon status with `mise daemons ls`. Do not duplicate these with generic `dev-*` tasks. Retain indirect tool dependencies: `uv` installs mise's PyPI tools, including Azure CLI. See [mise PyPI requirements](https://mise.jdx.dev/dev-tools/backends/pypi.html#requirements).

Use `mise exec pypi:vastai -- vastai ...` for the [official Vast.ai CLI](https://docs.vast.ai/cli/hello-world), installed from PyPI through mise.

**Long-running processes (dev servers, containers) are declared in `[daemons]` in the root `mise.toml` and managed by mise.** There is no `pitchfork.toml` — `mise daemons start|stop|restart|ls|status|logs|urls|tui` drives them (see the `[daemons]` section and the top-of-file note in `mise.toml`).

**pitchfork is still a hard dependency** — `mise daemons` is a front-end that generates a pitchfork config and delegates process supervision and readiness checks to it. Keep `pitchfork` declared in `[tools]` and resolvable in the project (`mise daemons stop` fails with "No version is set for shim: pitchfork" if it is not).

**Start daemons per task, not all at once.** A task declares what it needs with `daemons = ["<name>"]` (or `daemons = true` for every project daemon); mise starts them, waits for readiness, then runs the task body, and reuses anything already running. `auto = ["start", "stop"]` and `pitchfork activate` are gone — **nothing starts merely by entering the repo.** Prefer `mise run <app>-run` (which adds the `depends = ["<app>-build"]` it needs) over calling `mise daemons start` directly. Stop with `mise run <app>-stop` or `mise daemons stop`.

**Mise version locking:** Request `"latest"` for local tools and keep exact resolved versions, checksums, and provenance in the committed `mise.lock`. Use `mise lock --bump <tool>` to update the resolution without pinning the request.

For Aqua recipes that publish SLSA provenance but omit the signer, set both
`slsa_signer_identity` and `slsa_signer_issuer` on the consuming tool, using the
exact identity and issuer from its published certificate. Regenerate its lock
entries with `mise lock <tool>`; never remove provenance to permit an update.
See [Aqua signer options](https://mise.jdx.dev/dev-tools/backends/aqua.html#slsa-signer-identity-and-slsa-signer-issuer).

**Declarative-only tooling (no imperative system installs):** The local environment must be fully reproducible from `mise.toml` alone — a Nix-like experience. Never run `sudo apt-get install`, `brew install`, or any imperative system package manager command to satisfy a build dependency. If a tool or library is needed:
1. Check if mise provides it (add to `[tools]` in `mise.toml`).
2. If it's a Rust/Cargo tool, install via `cargo binstall` in a mise task.
3. If it's a system C library, linker, or other OS package, declare it in `[bootstrap.packages]` (e.g. `"apt:musl-tools" = "latest"`). mise handles the install via `mise bootstrap packages apply` — OS-filtered so apt entries are ignored on macOS, brew entries on Linux, etc. List entries alphabetically by key for easy scanning and editing.

This ensures `mise bootstrap --yes && mise run setup` is the only command needed to go from a fresh machine to a working environment.

**Per-service local dev convention (all services):** Every maintained service uses the root `mise.toml`, without per-application tooling, secret, or daemon config files:
- **Tasks** → `mise/tasks/<app>.toml`, selected by root `task_config.includes`; tasks use `<app>-` prefix.
- **Plaintext env vars** → root `mise.toml` (app-specific env vars grouped by `# ── <app> ──` comment headers in `[env]`).
- **Secrets** → root `mise.toml [env]` native age values; prefix names by their owner where needed.
- **Long-running dev servers** → root `mise.toml` `[daemons]` (ready checks, restart policies); each `<app>-run` task declares `daemons = ["<app>"]`
- Reuse shared secret definitions across services rather than duplicating their values.
- Onboard service environment values, age secrets, and its daemon in root `mise.toml`; build/test/run/image tasks belong in its selected task catalog. Run tasks declare only the daemons they require.

### API Pagination

Paginated APIs use cursor-based pagination, not offset/limit. Pass the last record's identifier or timestamp as an optional cursor alongside a page size, and return `next_cursor` (`None` when exhausted).

### External Service Config & Secrets
For any external service the repo calls:
- **Locally:** plaintext values and native age-encrypted secrets both live in root `mise.toml [env]`.
- **In-cluster:** plaintext → k8s `ConfigMap`; secrets → SOPS-encrypted `*.sops.yaml`. **Only provision cluster config if a service in our cluster actually calls the external service.** External clients (mobile apps, out-of-workspace backends) carry their own config and do not consume our cluster ConfigMaps/SOPS.
- Wire config into the consuming app through its existing mechanism, with mise providing local process environments.

### Inventory

Every node is declared **once** in `ansible/inventory/00-hosts.yml` under a single `nodes:`
group. `control_plane` and `worker_nodes` are **derived** from each host's `node_role` by
`ansible/inventory/10-role-groups.yml` (the `constructed` inventory plugin), so the two
views cannot drift from the list.

18 nodes today: **10 control planes** (node-1..node-7 Hetzner stacked + 3 Azure
Central India) and **8 workers**. Target via `-e target_host=...`.

**One playbook per operation, not one per node kind** — `add-node.yml` and
`upgrade-node.yml` each handle both, and the host's `node_role` decides which roles run.

The traps (silent `node_role` omission, the load-bearing `10-` filename, why the groups
exist rather than a `hosts:` expression) are in `ansible/AGENTS.md`.

### Ansible / Playbook Execution
- `become` is globally false; remote plays SSH as root; localhost plays as vscode user.
- **Prefer a collection module over a raw `uri`/`command` call.** If a maintained
  collection module exists for the API being touched, use it — `community.hrobot`
  covers the Hetzner Robot API. Raw `uri` calls in older roles are legacy, not
  precedent. Details, the two parameter traps, and the reasoning: `ansible/AGENTS.md`.
- **! `regex_search(...) | bool` is not a validity test — it is always `false`.** `regex_search`
  returns the matched **string**, and `| bool` coerces an unrecognised string to `False`, even
  one that plainly reads `Status="OK"`. A role gated on it silently takes the wrong branch
  forever. Use `is search` (returns a real boolean) or `not (x is search(...))`. Also: many
  XML APIs put data in **attributes**, not elements (`<Result ... IP="1.2.3.4">`), so a
  parse for `<IP>` matches nothing and yields an empty value. Both defects report `changed`
  and look like success — measured 2026-10-02 in `namecheap-dns`, where glue records were
  never updated for months while every run reported changes. A remote API answering errors
  with HTTP 200 compounds it: assert the parse succeeded, and fail on an error **body**,
  where the cause is still visible.
- Always run playbooks in foreground.
- **Always use mise tasks for Ansible operations.** Never invoke `ansible-playbook`, `ansible-vault`, `ansible-lint`, or `.venv/bin/ansible-*` directly — use the corresponding mise task instead:
  - `mise run ansible-playbook -- ansible/playbooks/operations/<playbook>.yml [-e target_host=...]` — run a playbook
  - `mise run ansible-lint` — lint playbooks and roles (runs on `ansible/playbooks/operations/`)
  - `mise run ansible-vault-decrypt` / `ansible-vault-encrypt` — decrypt/encrypt the vault for editing
  - `mise run ansible-vault-view -- <yaml key>` — extract a single vault key (e.g. `mise run ansible-vault-view -- vault_age_private_key`)
  - `mise run sops-decrypt -- <file>.sops.yaml` / `sops-encrypt -- <file>.sops.yaml` / `sops-edit -- <file>.sops.yaml` — SOPS file operations (extracts age key from vault automatically)
  This ensures the `.venv` Python environment is used, the vault password file is found via `ansible.cfg`, and all operations are reproducible. Raw `ansible-vault view ... | grep ...` pipes in scripts bypass this and should be replaced with `mise run ansible-vault-view`.
- **No truncating terminal output** — see the top-level "No Truncating Terminal Output" policy. Applies to `ansible-playbook`, `mise run`, and all other commands.
- Short poll loops (≤10s sleep) when waiting. Never use long sleeps (e.g. 120s) — instead poll with short intervals and re-check, or stop and let the user reinitiate.
- Lint before PRs.

### Flux

For an existing GitHub webhook URL change, use the [configuration-only endpoint](https://docs.github.com/en/rest/repos/webhooks#update-a-webhook-configuration-for-a-repository),
`PATCH /repos/{owner}/{repo}/hooks/{hook_id}/config`. Patching the parent webhook
with a replacement `config` removes its signing secret unless supplied again.
Verify a signed delivery to the new URL before retiring the old route.

Bootstrap with `--token-auth --components-extra=image-reflector-controller,image-automation-controller`.
Webhook receiver for near-real-time reconcile on push.
`dependsOn` for ordering.
Manual apply order only before bootstrap (and only for non-Flux resources).
**No manual `flux reconcile` or forced reconciliation** — the webhook receiver triggers reconciliation within seconds of a git push. Forcing reconciliation with `fluxcd.io/reconcile-at` annotations creates race conditions with the ImageUpdateAutomation controller (which commits image tag updates back to git). These concurrent reconciliations can overwrite each other's commits, causing git history divergence and lost changes. **After pushing to git, wait. Do not force reconcile.** The webhook + interval (1m) will handle it. Only use `flux reconcile` if the webhook is confirmed down (check GitHub webhook delivery status) or for explicit debugging — never as a shortcut to speed things up.

**Post-push validation (Non-Negotiable):** After every push, watch the cluster until the change has reconciled and the desired effect is confirmed in production. Do not declare done and walk away on "pushed." Concretely: poll the affected Flux Kustomizations (`flux get kustomizations --watch` or read-only `kubectl get kustomization -n flux-system`) until `Ready=True` with the new revision; then verify the target resources are healthy (`kubectl get/describe`, readiness, recent logs, health endpoints, smoke tests). If anything is not green or not as expected, stop and flag it to the operator immediately — then iterate (read-only diagnosis → fix in git/role → push → re-validate) until production is in a confirmed-fixed state. Never report success without this verification cycle having completed.

**Health check stalls:** When a Deployment is in `Failed`/`CrashLoopBackOff` status, Flux's `apps` kustomization health check will report unhealthy and may not apply new revisions. This is by design. Do not force reconcile or suspend/resume kustomizations to work around it. Instead, fix the root cause (the crashing pod) — build a new image with the fix, let the ImagePolicy/IUA pipeline update the tag in git, and push. Once pods come up healthy, Flux resumes normal operation automatically.

**Immutable Job pod templates (any spec change needs a delete):** A `Job`'s `spec.template` is immutable, so amending it — a new volume, env var, or command — fails with `field is immutable` and Flux cannot patch it. Unlike a Deployment, it will not roll. Delete the Job and let Flux recreate it (`kubectl delete job -n <ns> <name>`); this is routine, not a workaround, and applies to every Job in the repo. Two traps make this worse in practice: Flux reports it as a dry-run failure on the **Job**, but since the whole resource set is dry-run first, that one Job blocks application of **every** other resource in the same Kustomization (a ConfigMap added alongside never appears, with no error naming it) — so if a new ConfigMap is missing and Flux reports something unrelated, suspect a sibling Job. Also, Flux's `Ready` message caches the last failure: after deleting the Job, the message still describes the deleted object until the next attempt, so verify against live state (`kubectl get`) rather than the condition text.

### Image Registry (Harbor)
In-cluster Harbor at `harbor.saikat.dev` is the registry for custom-built app images (NOT bootstrap infra images — those stay on GHCR/quay.io for disaster recovery).

**Simplicity over fine-grained access control:** This is a single-contributor repo/cluster. Use the Harbor admin credentials directly for CI pushes, Flux image scanning, and pod image pulls. Store as a SOPS-encrypted `kubernetes.io/dockerconfigjson` Secret (e.g., `harbor-pull.sops.yaml`) in each app namespace. Admin password lives in `kubernetes/infrastructure/harbor/harbor-admin-secret.sops.yaml`.
- Every namespace pulling from Harbor must have an `imagePullSecrets` reference in the Deployment/StatefulSet spec pointing to that Secret.
- Use Flux `ImageRepository` + `ImagePolicy` + `ImageUpdateAutomation` for automatic tag updates from Harbor into git manifests.

### Image Building (Shipwright + BuildKit)
In-cluster image building via Shipwright (CRD-native, CNCF Sandbox) wrapping BuildKit (rootless). Tekton Pipelines is the execution engine. Triggered by Flux on git push — no external CI needed. See `kubernetes/infrastructure/shipwright/kustomization.yaml` and `kubernetes/infrastructure/tekton-pipelines/kustomization.yaml` for version pins and compatibility notes.

Two components are in play — Shipwright for builds, Tekton Triggers for the git-push → BuildRun hop — because Shipwright's own event triggering (SHIP-0031) is still WIP upstream. Verified 2026-09-22: the `shipwright-triggers` controller runs only inventory/pipelineRun/customRun reconcilers — no HTTP/webhook server and no GitHub provider exist in its source, so `Build.spec.trigger` is present in the CRD but unconsumed. **Do not adopt `Build.spec.trigger`** (it silently does nothing); Tekton Triggers stays until upstream implements the webhook handler. Full detail + migration steps: `TODO(consolidate-build-pipeline)` in `kubernetes/infrastructure/shipwright/kustomization.yaml`.

**Test locally before deploying (Hard Rule):** Always build and test code changes locally before pushing to git and deploying to production. The full end-to-end workflow uses mise tasks for every step — both human operators and agents must use the same mise tooling:

1. **Compile locally** — `mise run <app>-build` (release musl binary) or `mise run <app>-build-local` (debug, same code as production).
2. **Test** — `mise run <app>-test` (unit tests; external-service-dependent tests are removed — only pure-logic and production-endpoint tests remain).
3. **Run locally** — `mise run <app>-run` (release binary as a mise-managed daemon with native age values) or `mise run <app>-run-local` (debug build).
4. **Build image locally** — `mise run <app>-image` (podman build from repo root, `.dockerignore` excludes `target/` and `node_modules/`).
5. **Run image locally** — `mise run <app>-image-run` (podman container as a mise-managed daemon, port forwarded).
6. **Push** — `git push` and let the CI/CD pipeline (Tekton → Shipwright → Harbor → Flux) build and deploy.
7. **Validate on prod** — `mise run <app>-validate` (read-only kubectl checks: pods, readiness, recent logs).

Never push code changes to git without first verifying they compile and run locally. Waiting for an in-cluster Shipwright build to discover a compile error or runtime failure wastes 10+ minutes per iteration.

**Ignore patterns are explicit paths, never globs (Hard Rule).** In `.dockerignore` (and equally for anything that walks the tree — `.gitignore`, `.helmignore`, tool-specific ignore files), spell out the **specific** directory or file that must be excluded (`target/`, `.venv/`). Never use `**`-prefixed globs (`**/target/`, `**/.build/`) and never a broad `**`.

- **Measured, not assumed** (Apple CLI 1.4.1, 2026-09-24): an identical build context transferred **1.07 GB** with the glob set `**/target/` `**/node_modules/` `**/.build/` `**/DerivedData/`, and **66 MB** with the equivalent explicit paths. `dockerignore` matching is effectively all-or-nothing — an unhandled pattern degrades filtering rather than failing loudly, so the build still succeeds while transferring far more than intended. Explicit paths are unambiguous, are what the docs prescribe, and cannot silently defeat a rule set.
- **For `*.dockerignore` naming:** Apple's `container` CLI derives the ignore path from the `-f` value (`<dockerfile>.dockerignore`), so each app carries `apps/<app>/Dockerfile.buildkit.dockerignore` symlinked to the root `.dockerignore` for one source of truth (podman/Linux reads the root file directly, so that file must exist and stay authoritative).
- **List only paths that exist**, and re-check after each change — stale entries imply filtering that is not happening.
- **Build-context cost has two components that need different fixes.** *Transferred* size is what `dockerignore` reduces. *Walked-tree* size is what it does **not** — the CLI stats every path before applying ignore rules, so an excluded 111k-file tree still costs ~80 s of start-up latency. When image builds are slow to start or exhaust memory, delete only specific regenerable trees (`target/`, `.venv/`, `ansible/collections/`) — the ignore rule is a safety net, not the fix.
- **Verify the number, one build at a time.** Measure `transferring context:` from the build's own log. Never infer it from a side channel such as the newest `*/…/*.tar` in `$TMPDIR` — that file may belong to a different, concurrent build, which produces convincing but wrong results. (This mistake cost most of a debugging session.)

**Local image builds on macOS need explicit builder sizing (Hard Rule).** The Apple `container` builder VM defaults to 2 CPUs / 2048 MB, which **OOM-kills** our Rust builds. Measured 2026-09-24 on the Leptos image (`cargo leptos build`, ~200 crates across a WASM front + musl server target):

```
error: could not compile `syn`            (signal: 9, SIGKILL: kill)
Error: resourceExhausted: "...process ... did not complete successfully:
        cannot allocate memory"
```

So every `<app>-image` task passes `--cpus 17 --memory 40gb` on macOS (all but one of the host's 18 cores, 8 GB spare of 48). **Verified at this setting**: build succeeded (529 s), peak 46 GB used, **swap 0, swapfiles 0, no host panic**. Do not remove the flags — the default genuinely cannot build these images.

**The old host panic was a build-context problem, not a sizing problem.** An earlier revision at this same `--cpus 17 --memory 40gb` panicked the host (`watchdog timeout: no checkins from watchdogd`), which made the near-total allocation look unsafe. The actual cause was the ~15 GB build context (un-applied `.dockerignore` `**`-globs) driving memory-compressor/swap thrash — the 50 swapfiles the panic log showed. With the context fixed to 66 MB the same allocation runs clean. **Diagnose the context before blaming builder resources.** If the host ever does become unresponsive, drop to `--cpus 8 --memory 24gb`.

**Builder sizing and build-context cost are independent problems — do not conflate them.** A slow build start (a ~80 s stall Apple reports on the unhelpful `transferring dockerfile` line) is the context tree walk, fixed in `.dockerignore`; an OOM at the compile step is VM size, fixed by the flags. Sizing never speeds up the walk, and ignore rules never prevent the OOM.

**CI/CD Pipeline (Tekton Triggers → Shipwright → Harbor → Flux):** Git push → Tekton EventListener → BuildRun → Harbor image push → Flux ImageRepository/ImagePolicy/ImageUpdateAutomation → git commit updating deployment manifest → Flux Kustomization reconcile → new pods.

- Per-app Tekton Trigger resources in `kubernetes/apps/<app>/tekton-trigger.yaml`.
- GitHub webhook → EventListener Service (exposed via Cilium Gateway HTTPRoute).
- **Build webhook setup (per new app):** When adding a new app with Tekton build triggers:
  1. Create HTTPRoute for `<app>-build.saikat.dev` in `kubernetes/networking/routes/<app>-build.yaml` (HTTP→HTTPS redirect + HTTPS route to `el-<app>-build-listener` in the app namespace).
  2. Create ReferenceGrant in the app namespace (`kubernetes/apps/<app>/build-reference-grant.yaml`) allowing HTTPRoute from kube-system to reference the EventListener Service.
  3. Add both to their respective kustomization.yaml files.
  4. Create the GitHub webhook via `gh` CLI: `gh api repos/<owner>/<repo>/hooks --input - <<'EOF' { "name": "web", "active": true, "events": ["push"], "config": { "url": "https://<app>-build.saikat.dev/", "content_type": "json" } } EOF`
  5. Push to git and wait for Flux to apply the HTTPRoute.
- **Filter out fluxcdbot commits**: The CEL interceptor MUST filter `body.head_commit.author.username != 'fluxcdbot'` to prevent a feedback loop.

**Image tagging (timestamp-prefixed, NOT raw commit SHA):** Tags are `YYYYMMDDHHMMSS-<8-char-sha>`. The timestamp prefix ensures `Alphabetical:asc` in Flux ImagePolicy correctly selects the most recent build.

**Tekton Triggers CEL Interceptor (official pattern):**
- Overlay `key`: just the field name (e.g. `image_tag`), NOT prefixed with `body.` or `extensions.`
- Overlay `expression`: CEL expression evaluated against `body` (JSON payload), `header`, `extensions`, `requestURL`
- TriggerBinding access: `$(extensions.image_tag)` — NOT `$(body.extensions.image_tag)`
- Available CEL functions: `truncate(uint)`, `translate(regex, repl)`, `split(sep)`, `join(sep)`, `replace(old, new)`, `substring(start, end)`, `lowerAscii()`, `upperAscii()`, `parseJSON()`, `parseURL()`
- **Image tag CEL expression:** Use `.substring(0, 19).translate('[-:TZ]', '')` to extract the timestamp portion (first 19 chars of ISO 8601: `YYYY-MM-DDTHH:MM:SS`) before the timezone offset, then strip separators. This produces exactly 14 digits regardless of the committer's timezone (UTC uses `Z` suffix, non-UTC uses `+HH:MM` — both are stripped by the substring). The full expression: `body.head_commit.timestamp.substring(0, 19).translate('[-:TZ]', '') + '-' + body.after.truncate(8)`

**Flux ImagePolicy (official — only 3 policy types exist):**
- `SemVer` — semantic versioning range (e.g. `>=1.0.0`)
- `Alphabetical` — string sort, `asc` picks Z (last/highest), `desc` picks A (first/lowest) — use `asc` for timestamp-prefixed tags
- `Numerical` — numeric sort, same order semantics
- NO `Latest` policy type exists.

**Shared cluster-scoped resources (avoid kustomize "id matched 2 resources"):** When multiple app sub-kustomizations are included in the same parent Kustomization (`apps`), cluster-scoped resources (ClusterRole, ClusterRoleBinding) and shared namespace resources (e.g., `harbor-scan-secret` in `flux-system`) must be defined only ONCE. Duplicate definitions with the same name cause kustomize build failures.

## Questions for Agents (When in Doubt)

1. Is this logic in a role? → Move it.
2. Is the playbook a thin wrapper? → Extract to role.
3. Does this mutate existing nodes? → Reprovision from scratch (immutability).
4. Could this trigger a reboot? → Ensure `base-system` handles it.
5. Can this be tested independently? → Refactor if coupled.
6. Is this a mutation? → Role + playbook. Never SSH, `kubectl exec`, terminal `helm`/`kubectl apply` (Flux resources), etc.
7. About to create a new playbook? → Fit into existing via a new atomic role, or ask first.
8. About to SSH-mutate a node? → Stop. Role it.
9. About to touch multiple nodes? → Stop. Serial only; complete + verify before next.
10. About to `kubectl delete` a Flux Kustomization/resource? → Make the change in git and push (Flux prunes). `flux suspend` first only if removing the Kustomization CR itself or IUA-managed resources.
11. About to `kubectl apply/delete` a Flux-managed resource? → Make the change in git and push.

## Deployment / Upgrade Workflow

Full clean playbook run (no partial apply). On failure: diagnose read-only, fix the role, re-run the entire playbook from scratch.

**When a step fails:**
1. Read-only investigation only.
2. Fix the corresponding role/task.
3. Re-run full playbook.
4. Never "apply the fix by hand on the node".

### Ansible Idempotency & Graceful Failure (Hard Rule)
All Ansible roles and playbooks must be **idempotent** (safe to re-run without side effects) and **handle failures gracefully** (no manual intervention required). If a step fails, the role must be fixed so that re-running it succeeds — never bypass a failure with a manual `kubectl` command, SSH command, or any imperative action.

Specifically:
- **Longhorn replica eviction timeout**: If replicas can't be evacuated (insufficient space, scheduling constraints), the role must detect this, verify that the remaining replicas are sufficient for data safety (replica count ≥ 2 for 2-replica volumes), and proceed with a forced eviction instead of hanging. The node will be reprovisioned from scratch — the replica on the doomed node is already lost; waiting indefinitely for evacuation that can't complete is worse than proceeding.
- **Evict Longhorn replicas BEFORE draining**: `node-remove` must run `longhorn-evict` (set `evictionRequested` on the node's disks) BEFORE `kubectl drain`. This is the opposite of what this line used to say, and the old order deadlocks. Draining first cordons the node, and a cordoned node cannot host the pod that performs the eviction — Longhorn's instance-manager (owned by an InstanceManager CR, *not* a DaemonSet, so `--ignore-daemonsets` does not protect it). The InstanceManager CR goes to `currentState: error`, the eviction ticket can never detach, and the wait times out. Measured on worker-16 (2026-09-30): the volume stayed exclusively attached to the cordoned node, its pod stuck in `ContainerCreating`, and harbor-registry ended up with zero ready endpoints.
  - **Eviction is a REBUILD, and it takes as long as the data takes to copy.** It is not a move: Longhorn builds a replacement replica elsewhere and deletes the local one once the replacement is healthy. A 1-replica volume goes through the same path as a 2-replica one. Budget for a cross-region copy (a ~50 GiB replica took ~30 min Helsinki→India).
  - **A replica on an attached volume rebuilds fine.** Only the *engine* must wait for the drain, because it moves once the pods are evicted and the volume detaches.
  - **A failed/timed-out eviction is recoverable, not fatal.** Re-running `remove-node` skips the eviction when the node has 0 replicas and proceeds to the drain. The pattern that works when a 1-replica volume's only replica is on the target: run it once to arm the eviction and let it fail the wait, let Longhorn finish the rebuild in the background, then run it again — the second run skips straight through.
- **Partial failure recovery**: If a playbook run fails midway, re-running it must detect the partial state and resume from where it left off — not start over from the beginning or fail because of stale state.
- **No manual kubectl/SSH between playbook runs**: If something goes wrong, fix the role and re-run. Never intervene manually.

## Maintaining This Document

Add to / update AGENTS.md only for new patterns, clarifications that prevent repeated mistakes, or new core constraints.

Do **not** add one-off workarounds, temporary fixes, or incident narratives to `AGENTS.md` files. Preserve a durable implementation-specific lesson beside the owning code or manifest, with a succinct tagged comment only when names and types cannot express it. Preserve a reusable folder-wide rule in that folder's `AGENTS.md`.

Process: identify section, make minimal prescriptive update, remove obsolete guidance immediately.

**AGENTS.md is the canonical instruction file for humans and agents.** Do not create or maintain `README.md` files in code and configuration we own. When retiring a README, put reusable project- or role-wide guidance in the corresponding `AGENTS.md`; keep implementation-specific facts inline beside the owning code or manifest. Remove stale and redundant material instead of copying it wholesale, and update references. Independent submodules and vendored/generated content keep their upstream documentation and are not swept by this rule.

**Use checked-in, shared knowledge only.** Do not use agent-specific memory systems (e.g. `/memories/`, `.copilot/`, or similar) — that memory doesn't get shared between hosts and isn't checked into source control. Persistent instructions live in the appropriately scoped `AGENTS.md`; implementation details live in the code and manifests that own them.

**Inline and colocate by default.** The top-level `AGENTS.md` contains only global guidance. Child `AGENTS.md` files contain rules shared across their section, project, application, or role, at the narrowest common owner. Specific behavior, values, contracts, and pitfalls belong beside the code or manifest that implements them, not in a parallel instruction essay. Read applicable child instructions before working in that area. The root may point to those owners, but must not duplicate their guidance; child files must not duplicate declarations or inline implementation notes.