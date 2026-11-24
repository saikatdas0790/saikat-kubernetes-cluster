# RustFS Declarations

Console and S3 exposure are independent. A Console hostname change updates the
browser origin and named-provider callback in [helmrelease.yaml](helmrelease.yaml)
and the Google client's exact authorized callback before rollout. Preserve the
S3 route and storage declarations until their own migration is approved.

Follow the [native OIDC contract](https://docs.rustfs.com/en/security-compliance/oidc).
Authentication does not grant authorization: retain the IAM policy and Console
Service routing in [google-authorization.yaml](google-authorization.yaml).
Validate real login and administrator reads before retiring the old route,
listener and callback. Update the dashboard shortcut only after that verification.

S3 signatures bind the hostname and request path. Add a TLS alias without Host
rewrites, path rewrites, redirects, or browser-auth proxies. Verify native signed
object reads and a fresh [AWS CLI presigned GET](https://docs.aws.amazon.com/cli/latest/reference/s3/presign.html)
before retiring an old public hostname; confirm no external clients or unexpired
presigned URLs still depend on it. Keep presigned URLs out of logs. In-cluster
backups and Loki continue using Kubernetes Service DNS, not the public Gateway.

## Object encryption

[RustFS 1.0.5 standalone SSE-S3](https://github.com/rustfs/rustfs/blob/1.0.5/docs/operations/kms-backend-security.md#no-kms-configured-the-sse-s3-local-master-key)
uses one shared Base64-encoded 32-byte key across all replicas. Flux decrypts
[sse-s3-master-key.sops.yaml](sse-s3-master-key.sops.yaml); keep the ciphertext and
an independently recoverable SOPS age identity outside the cluster. Never replace
or remove the key while any retained object depends on it. Test key recovery
before relying on encrypted backups. Rolling back a bucket default affects only
future writes; it does not make the master key safe to remove.

The [bucket encryption Job](bucket-encryption.yaml) gates bucket defaults on an
encrypted write and a successful read through every replica. Check that Flux and
the Job are ready, then verify new Longhorn backups, PostgreSQL base backups/WAL,
and Loki writes and reads. New buckets need their own default and verification.
SSE-S3 protects object contents, not the full disk or all metadata.

Bucket defaults do not encrypt historical objects. Migrate each workload separately:

1. Inventory keys, sizes, retained versions, and writer/retention behavior. Create
   versioned destination buckets with SSE-S3 and verify a test write and read.
2. Establish dual writes where supported; otherwise schedule a writer pause for
   the final sync. Copy historical objects to the destination with explicit
   SSE-S3, preserving metadata and using multipart copy for large objects.
3. Compare source and destination inventories and content checksums, check the
   `AES256` response on destination objects, and perform isolated restore/query
   tests. For PostgreSQL, include a restore through archived WAL. Do not assume
   encrypted ETags are plaintext checksums.
4. Switch each consumer declaratively, verify production writes and reads, and
   retain the original bucket for rollback. Retire plaintext copies, historical
   versions, and any old snapshots only after separate retention approval.

## Storj `disk` migration

The [migration Job](storj-disk-migration/job.yaml) streams ciphertext from the
[native rclone Storj backend](https://rclone.org/storj/) to the
[`saikat-personal-disk` bucket on RustFS](https://rclone.org/s3/), not through a
`crypt` remote.
The crypt password and salt stay with the client, never in the Job. The sync
requests metadata preservation where supported and SSE-S3 on destination writes.
The S3 backend creates the destination bucket if missing; no `mkdir` is needed.

Use `mise run sops-edit -- kubernetes/infrastructure/rustfs/storj-disk-migration/storj-read-only.sops.yaml`
locally to supply a serialized Storj Access Grant scoped to list and read `disk`.
The grant must use the same Storj encryption context as the existing client so
it sees the same keys. Compare the Job's source totals with the expected inventory
before cutover; endpoint access by itself does not prove the grant sees all keys.
The dedicated Flux Kustomization depends on RustFS but does not wait for the
long-running Job. Verify the Job and follow rclone's periodic transfer stats with
`mise exec kubectl -- kubectl logs -n rustfs -f job/storj-disk-to-rustfs-1`.

Storj is authoritative until cutover. Every run deletes destination-only objects
from `saikat-personal-disk`; keep other writers off that bucket until cutover.
Use `rclone sync --dry-run` to preview planned changes when needed. The Job checks
keys and sizes in both directions after syncing; this is not full content
verification. Before cutover, pause Storj writes and reconcile a new, uniquely
named Job for the final pass. Do not rerun it after switching writers to RustFS.
Download representative files through the new crypt remote and compare contents;
full `rclone check --download` also works but reads the entire source again and
incurs further Storj egress. Keep the original bucket and crypt configuration for
rollback, verify an independent backup, and revoke the temporary Storj credential
when the migration is finished.
