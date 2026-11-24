# Kubernetes Declarations

This tree owns Kubernetes workload, networking, operator, and GitOps declarations. Ansible owns provisioning and node lifecycle operations; its bootstrap inputs live in `ansible/manifests/`. Follow the repository-root [operating rules](../AGENTS.md), including the prohibition on imperative mutations of Flux-managed resources.

## Sources of Truth

- [clusters/saikat-kubernetes-cluster/kustomization.yaml](clusters/saikat-kubernetes-cluster/kustomization.yaml) is the cluster reconciliation entry point. Bootstrap targets this directory, not the whole `kubernetes/` tree.
- `infrastructure/` owns shared operators and infrastructure services.
- `apps/` owns application deployment declarations, separate from application source in the root `apps/` tree.
- [networking/gateway.yaml](networking/gateway.yaml) owns the shared Gateway and its listeners; `networking/routes/` owns service routes.

Ordering is declared through the individual Flux Kustomizations' `dependsOn` fields. Do not substitute manual apply sequences or forced reconciliation for those dependencies.

When transferring a public GitOps repository across GitHub owners, first let
Flux reconcile without HTTPS credentials and suspend image writebacks in Git.
[Fine-grained tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
are scoped to one owner. Keep anonymous reads working through the transfer,
then reconcile a SOPS-encrypted Flux Git credential scoped to the destination
repository with `Contents: Read and write` before resuming writebacks. If
bootstrap still co-owns `spec.secretRef`, point it at an empty HTTPS Secret
instead of omitting it; server-side apply can preserve a co-owned field.
Customize Flux-generated sync manifests with the adjacent Kustomize overlay.
Enable SOPS decryption on the root Flux Kustomization and verify it reconciles
before adding its first encrypted Secret; it cannot apply both in one pass.

When moving the reconciliation entry point, retain a forwarding Kustomize
directory at the old path until the root Flux CR has reconciled its new
`spec.path`. Removing the only readable source path prevents Flux from applying
the declaration that would repair it.

## Upstream Release Bundles

Consume official, version-pinned release bundles in Kustomize `resources`, not
copied controller/CRD snapshots or independently overridden bundled images.
Keep cluster-specific behavior in patches; append arguments rather than replacing
upstream lists that contain versioned helper images. Renovate's generic release
URL managers update all references to the same project together, including startup
configuration reconciled separately. New GitHub release assets, tagged raw GitHub
manifests, and official Tekton mirror URLs need no per-image extractor.

Run `mise run infrastructure-release-checks` and
`mise run renovate-configuration-checks` before deploying bundle changes. The
Infrastructure Releases workflow runs without credentials on both Linux and
macOS, including bot PRs. Push validation targets `main`; branch updates receive
PR validation without a duplicate push run. It installs only `kubectl`, Python and Node via
mise-action and uses `mise run --skip-tools` to avoid installing unrelated
project runtimes during validation. Renovate performs check-gated merging itself rather
than delegating to platform automerge. Do not rely on native automerge to wait
for CI unless branch protection actually requires those checks. Validate components
merged by Renovate as well as manually merged updates before continuing.

When an operator chart upgrade requires updated CRDs, declare
`spec.upgrade.crds: CreateReplace` on its HelmRelease rather than applying CRDs
imperatively. Flux's default `Create` policy does not update existing CRDs; see
the [CRD lifecycle documentation](https://fluxcd.io/flux/components/helm/helmreleases/#controlling-the-lifecycle-of-custom-resource-definitions).

## Networking and TLS

Cilium cluster names must satisfy the [official Helm reference](https://docs.cilium.io/en/stable/helm-reference/):
at most 32 lowercase alphanumeric/dash characters, starting and ending with an
alphanumeric character. Preserve `cluster.id` during a name-only migration and
verify the serial agent rollout and Hubble relay before retiring old endpoints.
The Helm certificate method reuses the existing Hubble leaf after a rename.
Use [Cilium certgen](https://docs.cilium.io/en/stable/observability/hubble/configuration/tls/)
to reissue the serving leaf with both names and the existing CA. Verify its SANs
and every Relay replica before removing the one-time rotation resources. Use
[Flux upgrade retries without rollback](https://fluxcd.io/flux/components/helm/helmreleases/#upgrade-strategy)
when a rollback would restore the stale serving certificate.

External traffic reaches the shared Cilium Gateway through inventory-generated DNS and the declared exposure model. Node addresses, regional placement, and component versions are maintained in inventory and manifests, not copied into documentation tables.

Add services through Gateway API routes attached to the appropriate existing listener, with an HTTP-to-HTTPS redirect where needed. Check the Gateway's actual hostname coverage and certificate references before assuming a new hostname is covered. TLS issuance and renewal are managed declaratively by cert-manager; never delete a live certificate Secret to force reissuance.

The certificate shim deletes Gateway-controlled Certificates when their Secret
names are no longer referenced by that Gateway. Retire the listener in Git and
verify the unused Certificate is gone; do not delete it imperatively. Certificate
deletion does not by itself establish that its Secret was garbage-collected.

Cross-namespace backend references require a ReferenceGrant in the target namespace. Follow the root network-policy rules for gateway-originated traffic. Add every new route and grant to its owning Kustomize resource list.

## Validation and Retirement

Render the affected tree locally with `mise exec -- kubectl kustomize <directory>` before pushing. After a push, verify the exact source revision, Kustomization readiness, workload health, and public endpoints. Validate DNS against authoritative nameservers rather than only recursive caches or controller status.

Remove ordinary resources from Git and let Flux prune them. For complete removal of a Flux Kustomization, first reconcile an empty resource set with the CR resumed, verify its inventory and resources are gone, then remove the empty CR. Deleting a suspended CR orphans its resources. Stateful storage and external data require their own explicit retention or migration decision.

Flux pruning tracks [status.inventory](https://fluxcd.io/flux/components/kustomize/kustomizations/#inventory),
not ownership labels alone. For approved orphan cleanup, first reconcile the exact
existing declaration into the owning Kustomization and verify inventory membership;
then remove it from Git. Do not recreate a retired backend merely to delete its route.

For approved Longhorn backup deletion, adopt only the exact `BackupVolume` CRs
into GitOps before pruning them. Their [native finalizer](https://github.com/longhorn/longhorn-manager/blob/v1.12.1/controller/backup_volume_controller.go)
deletes remote data only while the BackupTarget is available and no
`delete-custom-resource-only` label is present. Retire writers first, preserve
the shared BackupTarget, and verify local volumes and remote backups are gone.
If remote objects are already verified absent but native cleanup is stalled,
declare `longhorn.io/delete-custom-resource-only: "true"` on only those exact
BackupVolume CRs. Longhorn propagates it to their Backup CRs and finalizes its
own metadata; do not strip finalizers manually.

Helm retains CRDs installed from a chart's `crds/` directory during uninstall.
For full retirement, verify no custom-resource instances remain, reconcile the
chart's same-version official CRD definitions into the owning Flux Kustomization,
and verify inventory ownership before emptying that resource set to prune them.
Preserve every `status.storedVersions` entry during adoption, even with zero
instances; an empty collection does not clear the CRD's stored-version history.

RustFS supports server-side bucket removal with a signed `DELETE /<bucket>` and
`x-rustfs-force-delete: true`; ordinary S3 bucket deletion requires an empty
bucket. Check the deployed version's
[request handler](https://github.com/rustfs/rustfs/blob/1.0.1/rustfs/src/app/bucket_usecase.rs)
and [header contract](https://github.com/rustfs/rustfs/blob/1.0.1/crates/utils/src/http/header_compat.rs)
before using this extension. Retire writers first, scope a GitOps Job to the
explicitly approved bucket, and require an authenticated `HEAD` to return 404
before removing the completed declaration. Do not confuse removing a bucket
provisioning Job with deleting the bucket's external data.