# Multi-County Plan — Per-County Isolation from Upload to Commit

Plan for making the import service safe for many counties (Marion, Riley,
Johnson, …) at once. Each county uploads its own master juror file, and that
data lands in **that county's** SQL Server database (`WebKS{County}Jury`)
and nowhere else.

**Status:** plan only. No code, template, or schema has been changed.

Companion docs:
- [`SERVICE_FLOW.md`](SERVICE_FLOW.md): how the two Lambdas work today.
- [`FLOW_DIAGRAM.md`](FLOW_DIAGRAM.md): the current flow as a diagram.
- [`API.md`](API.md): REST, event, and direct-invoke reference.
- [`STORED_PROCEDURE_ANALYSIS.md`](STORED_PROCEDURE_ANALYSIS.md): the legacy SP pipeline.

Cross-repo references (to `icon.web.justice360nextgen`, the Justice360 repo)
are written as plain paths, not links, because that code lives in a sibling
repo.

---

## 1. Goal and the Isolation Invariant

> **Every record carries exactly one county from the moment it is uploaded until
> it is committed. Every hop (API, S3, ingestion, DynamoDB, orchestration,
> sync, SQL) checks that county against the caller or the target. A mismatch
> means the record is rejected, never re-routed.**

DynamoDB stays the shared buffer between ingestion and sync. Isolation comes
from the **key design and from each processing step**, not from separate
tables per county.

---

## 2. Current-State Gaps

I confirmed every gap below in code. **Critical** means one county's data can
end up in, or be deleted from, another county's space today.

| # | Gap | Evidence | Severity |
|---|---|---|---|
| 1 | The DynamoDB key is `id` only. There is no county or upload-batch partition. | PK is `id` ([`template.yaml:98-107`](../template.yaml)). Jury rows use `voter_id`, or a random `uuid4` when blank ([`lambda_function.py:169`](../backend/lambda_function.py)), so re-uploads **duplicate** those rows instead of overwriting. Voter rows use `VR-{Last}-{First}-{DOB}` ([`lambda_function.py:283`](../backend/lambda_function.py)), so two different people (in different counties) with the same name and DOB **overwrite** each other. [`API.md:182`](API.md) documents "Duplicates by id overwrite." | High |
| 2 | Sync scans the whole table. Nothing filters by county. | `table.scan` with only an optional `source` filter ([`dynamo_reader.py:20-33`](../backend-tenant-sync/app/dynamo_reader.py)); every batch goes to one tenant ([`pipeline.py:152-157`](../backend-tenant-sync/app/pipeline.py)). | **Critical** |
| 3 | The voter-registration parser hardcodes Riley. | `"county": "Riley", "county_code": "RL"` ([`lambda_function.py:306-307`](../backend/lambda_function.py)). | High |
| 4 | The JuryListing file is statewide. Rows aren't checked against the uploader. | The real file `JuryListing_20250625.txt` has **2,043,120 rows covering all 105 Kansas county codes** (JO 473,211; SG 357,839; SN 122,338; …). One row has county code `19970610`: the naive `line.split(",")` ([`lambda_function.py:113`](../backend/lambda_function.py)) shifts columns when a field contains a comma. Any code is accepted without checks ([`lambda_function.py:131,167`](../backend/lambda_function.py)). | **Critical** (combined with #2) |
| 5 | Sync concurrency is capped at 1 globally. | `ReservedConcurrentExecutions: 1` ([`template.yaml:224-226`](../template.yaml)). Staging (`Juror.tblJurorList`) is **per DB**, so the real constraint is one run *per county*, not one run total. SP #8 also assumes one reconstitution per county per calendar day ([`STORED_PROCEDURE_ANALYSIS.md:227`](STORED_PROCEDURE_ANALYSIS.md)). | Medium (throughput) |
| 6 | Sync IAM allows exactly one secret, and the payload can change it. | IAM grants `secret:${TenantDbSecretName}*` only ([`template.yaml:17-24,256`](../template.yaml)); the env var is required ([`config.py:45-50`](../backend-tenant-sync/app/config.py)). **Any invoker can override `tenant_db_secret_name`** ([`handler.py:26`](../backend-tenant-sync/app/handler.py)). Once IAM is widened to many county secrets, that override lets a caller point one county's data at another county's DB. | **Critical** once widened |
| 7 | The API has no authorizer and CORS is `*`. | No `Auth` on `JuryPoolApi` ([`template.yaml:383-391`](../template.yaml)); `Access-Control-Allow-Origin: *` in code ([`lambda_function.py:408-412`](../backend/lambda_function.py)); documented as "trusted-network only" ([`API.md:23-25`](API.md)). `/upload-url`, `/records`, and `/stats` are open and not scoped to a county. | High |
| 8 | Ingestion reads the whole file into memory and calls KMS once per row. | `read().decode(...)` plus a full in-memory `records` list ([`lambda_function.py:325-337`](../backend/lambda_function.py)); one `kms:Encrypt` per jury row ([`lambda_function.py:72-76,174`](../backend/lambda_function.py)). That's **about 2M KMS calls** for the 451,529,583-byte file, inside a 900 s / 1024 MB Lambda ([`template.yaml:7-8`](../template.yaml)). This will not finish. | High (statewide file) |
| 9 | The idempotency check can never match. | The check reads `Audit.tblTransaction.Notes LIKE '%idempotency_key=…%'` ([`pipeline.py:93-97`](../backend-tenant-sync/app/pipeline.py)), but the key is only written to the **reconstitution** `@Note` ([`pipeline.py:71`](../backend-tenant-sync/app/pipeline.py)). The audit begin call passes no note ([`pipeline.py:54-59`](../backend-tenant-sync/app/pipeline.py)). An empty key skips the check ([`pipeline.py:90-91`](../backend-tenant-sync/app/pipeline.py)), and any SQL error is swallowed ([`pipeline.py:99-101`](../backend-tenant-sync/app/pipeline.py)). Tests mock the check ([`test_pipeline.py:226-246`](../backend-tenant-sync/tests/test_pipeline.py)), so CI can't catch this. The claim in [`CONTEXT.toon:300`](../CONTEXT.toon) that a re-run is a no-op is false. | High |
| 10 | Cleanup deletes by `id` only. | `batch.delete_item(Key={"id": item_id})` ([`dynamo_reader.py:61`](../backend-tenant-sync/app/dynamo_reader.py)) on every id that was scanned ([`pipeline.py:166,235-238`](../backend-tenant-sync/app/pipeline.py)). Today that deletes other counties' rows along with this run's. It also deletes any newer upload that overwrote the same `id` while the sync was running, before that upload was ever synced. | **Critical** |
| 11 | `/stats` and its cache are global. | Counts come from table-wide scans ([`lambda_function.py:21-50`](../backend/lambda_function.py)); one SSM parameter ([`template.yaml:276-297`](../template.yaml)) is served to every caller ([`lambda_function.py:462-464`](../backend/lambda_function.py)). | Medium (information leak) |
| 12 | `COUNTY_CODES` has McPherson twice and no Mitchell. | `"MC": "McPherson"` and `"MP": "McPherson"` ([`lambda_function.py:95-96`](../backend/lambda_function.py)). The map has 105 codes but only 104 names; **Mitchell is missing**. The file has MC = 4,339 rows and MP = 21,622, which fits Mitchell (small) and McPherson (larger). So MC is almost certainly Mitchell, and Mitchell's jurors are labelled McPherson today. **Confirm against the official KDOR county code list** before fixing. | High (mislabel) |

Doc drift found along the way (fix in the same PR as Phase 0):
- [`API.md:119`](API.md) says the PII is "KMS-envelope-encrypted." It's direct `kms:Encrypt` ([`crypto.py:3-6`](../backend-tenant-sync/app/crypto.py)).
- [`API.md:205-207`](API.md) describes an SSM State Manager association. The template uses an EventBridge rule ([`template.yaml:262-297`](../template.yaml)).
- `frontend/` was removed in commit `9875d73`, but [`azure-pipelines-frontend.yml:27,52`](../azure-pipelines-frontend.yml) and [`CONTEXT.toon:195-219`](../CONTEXT.toon) still refer to it.

---

## 3. Target Architecture

```mermaid
flowchart TD
    User([County staff user]):::actor
    UI[React UI]:::ui

    subgraph AWS["AWS — jury-pool-stack-{env}"]
        APIGW[[API Gateway<br/>Cognito authorizer]]:::aws
        Reg[(ImportCounties<br/>derived registry cache)]:::storage
        S3[(S3 uploads/{countyCode}/{uploadId}/file<br/>tags: county-code, upload-id)]:::storage
        Ingest[Ingestion λ<br/>stream · validate · per-row county check<br/>envelope encrypt]:::step
        Rej[(S3 rejects/{countyCode}/{uploadId}.csv)]:::storage
        DDB[(JuryPool-{env}<br/>PK COUNTY#code#BATCH#uploadId)]:::storage
        Batches[(ImportBatches<br/>status per upload)]:::storage
        Q[[SQS FIFO<br/>MessageGroupId = countyCode]]:::aws
        Sync[Tenant-sync λ<br/>Query one partition · registry-derived secret]:::step
    end

    subgraph Setup["Justice360 setup DB"]
        CC[(dbo.CountyConnection<br/>authoritative)]:::db
    end
    subgraph Tenants["County SQL Servers (VPC)"]
        DBM[(WebKSMarionJury)]:::db
        DBR[(WebKSRileyJury)]:::db
    end

    User --> UI -- "JWT + county" --> APIGW
    APIGW -- "county from claim/header,<br/>checked vs Reg" --> Ingest
    UI -. "presigned PUT (county in key+tags)" .-> S3
    S3 --> Ingest
    Ingest -- "rows where row.county == upload.county" --> DDB
    Ingest -- "mismatched / malformed rows" --> Rej
    Ingest --> Batches
    Batches -- "PARSED" --> Q --> Sync
    Sync -- "Query PK = COUNTY#code#BATCH#id" --> DDB
    Sync -- "secret for code only" --> Tenants
    CC -- "scheduled projection" --> Reg
    Sync -. "reads" .-> CC

    classDef actor fill:#2d3748,stroke:#a0aec0,color:#fff
    classDef ui fill:#4a5568,stroke:#cbd5e0,color:#fff
    classDef aws fill:#ff9900,stroke:#b35900,color:#1a202c
    classDef storage fill:#3182ce,stroke:#2c5282,color:#fff
    classDef step fill:#38a169,stroke:#22543d,color:#fff
    classDef db fill:#c53030,stroke:#742a2a,color:#fff
```

Where the county boundary is enforced, hop by hop:

```
Hop                          County comes from                     Enforced by
─────────────────────────────────────────────────────────────────────────────────────
1 UI → API GW                Cognito CountyID claim / X-Tenant-Id  Authorizer + registry lookup (exists, active)
2 API → presigned URL        server-side resolved county           Key uploads/{code}/{uploadId}/ built server-side;
                                                                   tags signed into the URL
3 S3 → ingestion             S3 key prefix + object tags           Must agree, or the batch FAILS before parsing
4 ingestion → DynamoDB       batch county                          Row county_code must equal batch county;
                                                                   otherwise it goes to the reject file
5 DynamoDB → queue           ImportBatches row                     MessageGroupId = countyCode
6 queue → sync               message {countyCode, uploadId}        Secret derived from the registry, never from payload
7 sync → SQL                 registry JuryDatabase                 Connected DB name must equal registry JuryDatabase
8 cleanup                    same partition                        Delete only PK = COUNTY#code#BATCH#uploadId
```

---

## 4. County Registry

**Decision: `dbo.CountyConnection` in the Justice360 setup DB stays the single
source of truth.** The import service reads it. It never edits it and never
keeps a hand-maintained copy.

What already exists (Justice360 repo):

| Fact | Where |
|---|---|
| Columns: `JuryDatabase, JuryConnectionName, ConnectionName, CountyConnectionID, CountyID, CountyCode, CountyName, SiteUrl, TenantId, WebSiteID`, filtered `active=1` | `IconSoftware.Jury.DataLayer/SQLHelper.cs:245` |
| Cached for 1 hour | `SQLHelper.cs:271` |
| The setup DB is reached through the secret named in `AWS:CountySetupConnectionSecretName` | `SQLHelper.cs:85` |
| The per-county secret (named by `JuryConnectionName`) is JSON `{ "<JuryConnectionName>": "<ADO.NET connection string>" }`; the catalog is then **overridden** with `JuryDatabase` | `SQLHelper.cs:97-107`, `GetConnectionStringFromAwsSecrets` |
| That means several counties can share one server secret and differ only by `JuryDatabase` | same |
| The county list is served at `GET /api/v1/jury/auth/counties`; code → CountyID lookup exists | `JurorApi.Shared/Auth/CountyLookupService.cs:29-45` |
| The signed-in staff user's county is `Justice360.Classes.Environment.CountyID`: the `CountyID` claim, falling back to the `CID` cookie. It's the real `CountyConnection.CountyID` | `Jury/Classes/Environment.cs:492-523` |

### How the import service uses it

```
dbo.CountyConnection  ──(sync λ, VPC, hourly schedule)──►  ImportCounties (DynamoDB, derived)
        ▲                                                    CountyID (PK), CountyCode, CountyName,
        │ direct read at sync start                          JuryConnectionName, JuryDatabase, active,
        │ (authoritative for the secret)                     refreshed_at
   tenant-sync λ
```

- **Sync Lambda:** already VPC-attached. At the start of each run it resolves
  `countyCode → {JuryConnectionName, JuryDatabase}` **directly** from
  `CountyConnection`, so a stale cache can never route data to the wrong DB.
- **Ingestion Lambda:** has no VPC or DB access by design
  ([`SERVICE_FLOW.md` §8](SERVICE_FLOW.md)). It reads the derived
  `ImportCounties` table only to translate CountyID ↔ CountyCode and to check
  that a county is active.
- **Why a derived cache and not a new registry:** it saves the ingestion
  Lambda from getting DB and VPC access just for lookups. It's rebuilt from
  `CountyConnection` on every refresh and never edited by hand, so it isn't a
  second source of truth. Sync always re-checks against the authoritative
  table.
- **Import-only settings** (`enabled_for_import`, sync schedule, subnet/SG
  profile) don't belong in Justice360's table. They go in a small
  `ImportCountySettings` config, keyed by CountyID and kept in this repo's
  SAM parameters or SSM. A county with no settings row is **not enabled** for
  import.
- **Secret format adapter:** today `secrets.py` expects
  `{host, port, username, password, database}`
  ([`secrets.py:37,72-80`](../backend-tenant-sync/app/secrets.py)). Add a
  second parser for the Justice360 connection-string format. Parse
  `Server/User ID/Password` and set `database = JuryDatabase` from the
  registry, ignoring whatever catalog the string contains. Both formats stay
  supported during migration.

---

## 5. DynamoDB Key Design

### `JuryPool-{env}` (records), replacing PK=`id`

| Attribute | Value | Purpose |
|---|---|---|
| `PK` | `COUNTY#{countyCode}#BATCH#{uploadId}` | One partition per county upload. Sync `Query`s exactly one. |
| `SK` | `REC#{recordKey}` | Unique within a batch |
| `recordKey` | JuryListing: `voter_id`; if blank, `NOVID#{sha256(last,first,dob,res_zip)}`. Voter reg: `VR#{sha256(last,first,middle,dob,street,zip)}` | Deterministic, so re-parsing the same file is idempotent. Cuts the name+DOB collisions of gap #1. |
| `county_code`, `upload_id` | duplicated as attributes | Assertions in sync; GSI keys |
| `GSI1PK` / `GSI1SK` | `COUNTY#{code}` / `{source}#{last_name}` | Per-county `/records` without a Scan |
| `ttl` | set when the batch reaches `SYNCED` (+N days, see Decisions Needed) | Replaces the delete-by-id cleanup, and the delete is scoped to the partition by definition |

Duplicate handling within a batch: the same `recordKey` twice in one file
keeps the **last** row, and the reject file records it as a duplicate.
Nothing is silently merged across batches, because each batch is its own
partition.

### `ImportBatches` (new)

| Attribute | Notes |
|---|---|
| `PK` | `COUNTY#{countyCode}` |
| `SK` | `BATCH#{createdAt}#{uploadId}` (per-county history, newest-last) |
| `status` | `UPLOADED → PARSING → PARSED → SYNCING → SYNCED` / `FAILED` (with `failed_stage`) |
| counts | `rows_read`, `rows_accepted`, `rows_rejected`, `rows_staged`, `rows_synced` |
| refs | `s3_key`, `reject_key`, `reconstitution_id`, `transaction_id` |
| audit | `uploaded_by` (Cognito sub), `county_id`, timestamps per status |
| GSI | `uploadId` → row (status lookups from the UI and sync) |

Status changes use a conditional write (for example, only `PARSED → SYNCING`
when `status = PARSED`). That doubles as the per-batch lock.

---

## 6. S3 Layout

```
s3://jury-pool-uploads-{env}-{acct}/
  uploads/{countyCode}/{uploadId}/{originalFilename}   ← presigned PUT target
  rejects/{countyCode}/{uploadId}.csv                  ← row, line_no, reason
```

- `GET /upload-url` builds the key **server-side** from the county resolved in
  §9. The client's `filename` only fills the last path segment, sanitized.
  Today the key is `uploads/{uuid}/{filename}`
  ([`lambda_function.py:423`](../backend/lambda_function.py)).
- The presigned URL signs the `x-amz-tagging` header
  (`county-code=…&upload-id=…`) and metadata `uploaded-by`. The bucket policy
  **denies `PutObject` under `uploads/`** unless the `county-code` tag matches
  the key's second path segment.
- The `ImportBatches` row is created as `UPLOADED` when the URL is issued, so
  an orphaned URL shows up as a stale `UPLOADED` batch.
- Ingestion checks that key prefix, object tag, and `ImportBatches.county`
  all agree before reading a byte. If any disagree, the batch goes to
  `FAILED` with stage `precheck`.

---

## 7. Ingestion Changes

| Change | Replaces | Detail |
|---|---|---|
| Streaming read | `read().decode()` ([`lambda_function.py:326`](../backend/lambda_function.py)) | Iterate `StreamingBody.iter_lines()`; for CSV, wrap in `io.TextIOWrapper` and feed `csv.reader`. Memory stays bounded to one write batch. |
| Real CSV parsing for JuryListing | `line.split(",")` ([`lambda_function.py:113`](../backend/lambda_function.py)) | Use `csv.reader` so quoted commas stop shifting columns (the `19970610` row). Rows that don't have exactly the expected field count are rejected with reason `field_count`. |
| Per-row county check | accept any code ([`lambda_function.py:131,167`](../backend/lambda_function.py)) | `row.county_code` must equal the batch county. Otherwise the row is rejected (`county_mismatch`). The default is **quarantine, not fan-out** (see Decisions Needed #1). |
| County for voter files | hardcoded Riley ([`lambda_function.py:306-307`](../backend/lambda_function.py)) | Voter files carry no county column, so the county is the batch county. |
| County map | `COUNTY_CODES` ([`lambda_function.py:80-108`](../backend/lambda_function.py)) | Validate codes against the registry cache. Fix MC/MP after KDOR confirmation (gap #12). |
| Envelope encryption | one `kms:Encrypt` per row ([`lambda_function.py:72-76`](../backend/lambda_function.py)) | One `GenerateDataKey` per write batch (for example, 1,000 rows). AES-GCM each PII field locally, with the row's `county/uploadId/recordKey` as AAD. Store `v2:{keyRef}:{nonce}:{ct}`; the wrapped data key is stored once per batch on the batch row. KMS encryption context is `{county, uploadId}`. That's about 2,000 KMS calls instead of about 2M. Sync reads both `v1:` and `v2:` during migration. |
| Reject file | none | Stream rejected rows to `rejects/{code}/{uploadId}.csv` with `line_no, reason, raw` (PII columns masked). |
| Batch status | none | `PARSING` at start, counts on every flush, `PARSED` or `FAILED` at end. |
| Runtime budget | 900 s / 1024 MB ([`template.yaml:7-8`](../template.yaml)) | Once KMS is batched, the cost is dominated by DynamoDB writes (about 2M items for the statewide file). The Phase 2 load test decides the runtime. If it can't finish in 900 s, move parsing to a Step Functions Distributed Map over S3 byte ranges, or to a Fargate task (Decisions Needed #6). A county-sized file (Johnson ≈ 473k rows) is the realistic worst case once statewide files are quarantined per county. |

---

## 8. Sync Orchestration

**Recommendation: SQS FIFO with `MessageGroupId = countyCode`.** Different
counties run in parallel, and each county runs one at a time. The staging
table is per DB, which is exactly what this ordering protects
([`template.yaml:224-226`](../template.yaml) note).

```
ImportBatches PARSED ──(DDB Stream / ingestion emits)──► SQS FIFO
                                                          MessageGroupId        = countyCode
                                                          MessageDeduplicationId = uploadId
EventBridge schedule (per county, optional) ─────────────►  same queue
                         │
                         ▼
             tenant-sync λ (reserved concurrency = N counties in parallel, e.g. 5)
             1. conditional update ImportBatches PARSED → SYNCING   (per-batch lock)
             2. registry lookup countyCode → JuryConnectionName, JuryDatabase (direct read)
             3. fetch that secret; assert connected DB_NAME() == JuryDatabase
             4. idempotency: skip if ImportBatches.status == SYNCED or
                tblReconstitution.Note LIKE '%upload_id={uploadId}%'
             5. Query PK = COUNTY#{code}#BATCH#{uploadId}  (paged; replaces Scan)
             6. existing SP pipeline unchanged (pipeline.py:141-208)
             7. COMMIT → ImportBatches SYNCED (+ reconstitution_id) → set TTL on partition
             failure → ROLLBACK → ImportBatches FAILED, message retried (maxReceiveCount 3) → DLQ
```

Code-level changes this implies (Phase 0/3):
- Remove `tenant_db_secret_name` and `kms_key_id` from `_OVERRIDABLE`
  ([`handler.py:25-36`](../backend-tenant-sync/app/handler.py)). The secret
  always comes from the registry. The payload carries only
  `{countyCode, uploadId, dry_run}`.
- `iter_records` takes `(county, upload_id)` and uses `Query`
  ([`dynamo_reader.py:12-45`](../backend-tenant-sync/app/dynamo_reader.py)).
- **Idempotency fix (gap #9):** write `upload_id=` into the reconstitution
  `@Note`, which already happens for the key
  ([`pipeline.py:71`](../backend-tenant-sync/app/pipeline.py)), and check
  **`Juror.tblReconstitution.Note`**, not `Audit.tblTransaction.Notes`. The
  reconstitution row rolls back with a failed run, so it only exists for
  successful runs, which is what an idempotency marker needs. Stop swallowing
  errors on the check ([`pipeline.py:99-101`](../backend-tenant-sync/app/pipeline.py)):
  if the check can't run, fail the run. Replace the mocked test with one that
  asserts the SQL text and table.
- Cleanup (gap #10): drop `delete_items` by id. The partition gets a TTL, or
  is deleted by `Query` on the same PK.
- `SOURCE_FILTER` becomes irrelevant, because one batch is one file of one
  source. This also closes the CCJJ-592 two-invocation risk, provided the
  exclusion diff (CCJJ-589) is keyed on a *cycle*, not a batch (Open
  Questions #8).

Step Functions (Map over counties with a DynamoDB lock) is the alternative. It
gives better visual tracing, but you have to build the per-county
serialization that FIFO gives you for free.

---

## 9. Security

| Area | Plan |
|---|---|
| **Authentication** | Attach a Cognito User Pool authorizer to `JuryPoolApi` ([`template.yaml:383-391`](../template.yaml)). Unauthenticated calls get 401. |
| **County resolution** | Follows the existing Justice360 decision. Once the caller is authenticated, the county they present is **trusted for routing**: the `CountyID` claim if the token has one, otherwise the `X-Tenant-Id` header, or the embedded `county-id` value when the UI is served inside Justice360 (the same source as `Environment.CountyID`, `Jury/Classes/Environment.cs:492-523`). The backend only checks that the CountyID **exists and is active** in `ImportCounties` and is `enabled_for_import`. |
| **Per-user county membership** | **Not designed in**, by deliberate choice in Justice360. Listed as an option in Decisions Needed #4. If adopted later, it plugs into the same resolution step (the CountyConnectionMembership tables already exist, `SQLHelper.cs:198-200`). |
| **CORS** | Replace `*` with the Justice360 / UI origins ([`lambda_function.py:409`](../backend/lambda_function.py), [`template.yaml:388-391`](../template.yaml), and the upload bucket's CORS [`template.yaml:85-90`](../template.yaml)). |
| **IAM, sync** | `secretsmanager:GetSecretValue` scoped to the naming prefix used by `JuryConnectionName` secrets (Open Questions #4), plus the setup-DB secret. No wildcard across the account. |
| **IAM, ingestion** | `kms:GenerateDataKey` only, with the condition `kms:EncryptionContext:county` present. DynamoDB access limited to the records and batches tables. No Secrets Manager access. |
| **KMS context** | `{county, uploadId}` on every data key. Sync decrypts with the context rebuilt from the partition it is reading, so ciphertext copied across counties fails to decrypt. |
| **Audit trail** | `ImportBatches` keeps who, what, when, and the counts; the tenant `Audit.tblTransaction` keeps the SQL side. CloudTrail covers S3 and KMS. |
| **Logs** | No names, DOBs, voter IDs, or addresses in logs. Log the county, uploadId, counts, and line numbers only. Reject files mask PII columns. |

---

## 10. Networking

- Today there's one subnet/SG set for the sync Lambda
  ([`template.yaml:25-35,239-243`](../template.yaml)).
  `CountyConnection.JuryConnectionName` lets counties sit on different
  servers (`SQLHelper.cs:97-107`). Whether they actually do is Open
  Questions #3.
- **If all county servers are reachable from one VPC** (peering, Transit
  Gateway, or a single hosting VPC): keep one sync function, with SG egress
  to each server's 1433.
- **If some aren't:** deploy one sync function per *network profile* (not
  per county), each with its own `VpcConfig`. The registry settings map
  county → profile, and each profile has its own FIFO queue.
- The VPC-attached sync needs **VPC endpoints**: a DynamoDB gateway endpoint,
  plus interface endpoints for Secrets Manager, KMS, and SQS. The alternative
  is a NAT gateway (simpler, but more cost and more egress surface).
  Recommend endpoints.
- The setup DB (`dbo.CountyConnection`) must also be reachable from the sync
  VPC.

---

## 11. Frontend (React UI)

The UI source isn't in this repo any more (commit `9875d73`; Open Questions
#5). Required behavior wherever it lives:

- **County-aware, no picker.** The county comes from the embedded Justice360
  county or the token, the same pattern as the Questionnaire Builder SPA. A
  static chip shows the county name.
- **Upload:** call `/upload-url`, send the PUT with the signed tag header,
  then poll the batch status.
- **Batch status page:** list `ImportBatches` for the county
  (`GET /batches`), with status, counts, timestamps, and failure stage.
- **Reject download:** `GET /batches/{uploadId}/rejects` returns a
  short-lived presigned GET for `rejects/{code}/{uploadId}.csv`.
- **Records and stats:** `/records` and `/stats` are served per county
  (`GSI1`, and a per-county stats item or SSM parameter
  `/jury-pool/{env}/stats/{countyCode}`). Replaces the global cache
  ([`lambda_function.py:21-50`](../backend/lambda_function.py)).
- Every county-scoped query in the UI waits until the county is known, so it
  never fires a request with no county.

---

## 12. Onboarding a New County — Checklist

| # | Step | Owner | Done when |
|---|---|---|---|
| 1 | `dbo.CountyConnection` row exists and is `active=1`, with `CountyID`, `CountyCode`, `JuryDatabase`, `JuryConnectionName` | Justice360 admin | Row visible in `GET /api/v1/jury/auth/counties` |
| 2 | `CountyCode` matches the KDOR 2-letter code used in state files | Import owner | Code matches the county's rows in a sample JuryListing |
| 3 | Secret `JuryConnectionName` exists, follows the naming prefix, and its login can `EXEC` the Juror/Audit SPs | Infra / DBA | Sync IAM can read it; test login succeeds |
| 4 | Network path from the sync profile to the county server on 1433 | Infra | `SELECT 1` from the sync Lambda test event |
| 5 | Tenant DB prerequisites: `Juror.tblJurorList` passes `discover()` ([`staging.py:172-198`](../backend-tenant-sync/app/staging.py)); SPs #1–#13 present; `PrivateDataSymmetricKey` and `PrivateDataCertificate` present ([`SERVICE_FLOW.md` §7](SERVICE_FLOW.md)) | DBA | Dry-run gets past discovery and the encryption SP |
| 6 | `ImportCountySettings` entry: `enabled_for_import=false`, network profile, schedule | Import owner | Visible in the `ImportCounties` projection |
| 7 | Cognito: county users can obtain a token that carries or is paired with this CountyID | Identity owner | Test user resolves to the right county on `/upload-url` |
| 8 | Dry run with a real file (`dry_run=true`) | Import owner + county | `staged_rows` matches accepted rows; reject file reviewed with the county |
| 9 | Go-live: set `enabled_for_import=true`; first real sync watched | Import owner | `ImportBatches` SYNCED; row counts match; spot-check 10 jurors in the county UI |
| 10 | Alarms subscribed for that county | Ops | Test alarm delivered |

---

## 13. Migration from the Single-Table Design (Zero Data Loss)

1. **Freeze.** Disable sync schedules. Take an on-demand DynamoDB backup of
   `JuryPool-{env}` and keep it until step 7.
2. **Deploy new tables** (`JuryPool-v2-{env}`, `ImportBatches`,
   `ImportCounties`) side by side. Leave the old table untouched.
3. **Backfill** with a one-off job that scans the old table and writes each
   item into `COUNTY#{county_code}#BATCH#legacy-{yyyymmdd}`:
   - Jury-listing rows: use their own `county_code`. Remap MC after the KDOR
     check.
   - Voter-registration rows: `county_code` is always `RL` because of the
     hardcode (gap #3). Import them as `RL`, but create the batch with
     `status=PARSED, needs_review=true` so they **can't sync until a human
     confirms** they really are Riley.
   - Rows with a code not in the registry go to `COUNTY#UNKNOWN#…` and are
     reported. Nothing is dropped.
   - `v1:` ciphertext is copied as-is. Its KMS context is `{"id": old_id}`,
     so keep `legacy_id` on the item and let sync decrypt with it.
4. **Verify** counts per county (old vs new) and checksums of `recordKey`
   sets. The migration fails if any count differs.
5. **Cut over.** Point ingestion and sync at the new tables. Keep the old
   table read-only for one release.
6. **Re-enable** sync per county, one county at a time, starting with a
   dry-run.
7. **Retire** the old table after one clean sync cycle per enabled county.
   Delete the backup per the retention decision.

Rollback at any step before 5: drop the new tables; nothing old was
modified. After 5: point env vars back at the old table (it's untouched).

---

## 14. Testing

| Level | What | Pass criteria |
|---|---|---|
| Unit: parsers | JuryListing via `csv.reader`, quoted commas, short/long rows, bad DOB, the `19970610` row pattern; voter CSV with no county | Correct fields, or a reject with the right reason |
| Unit: keys | PK/SK builders, `recordKey` determinism, collision cases (same name+DOB, different address) | Stable keys; no collisions in the collision cases |
| Unit: county check | Row county ≠ batch county; unknown code; MC/MP | Rejected, or mapped per registry |
| Unit: idempotency | Real SQL text targets `tblReconstitution.Note`; a check failure fails the run | Replaces the mocked [`test_pipeline.py:226-246`](../backend-tenant-sync/tests/test_pipeline.py) |
| Unit: handler | Payload containing `tenant_db_secret_name` is ignored or rejected | Secret comes from the registry only |
| Integration: **isolation** | Two counties upload at the same time (moto or localstack DynamoDB + SQS; two SQL Server containers) | **Zero** rows from county A in DB B, and vice versa; cleanup of A leaves B's partition intact |
| Integration: ordering | Two batches for the same county queued together | Run one after the other; the second sees the first's reconstitution as prior |
| Integration: auth | No token; valid token with an unknown or inactive county | 401 / 403 |
| Dry-run, real tenant | Non-prod county DB with a real file, `dry_run=true` | Discovery passes; counts match; rollback leaves no rows |
| Load | Full 451 MB `JuryListing_20250625.txt` split into county batches, plus the largest single county (Johnson, 473k) | Ingestion finishes inside the chosen runtime budget; KMS calls ≈ rows/batch_size; no throttling alarms |

---

## 15. Observability

- Metrics (EMF, dimension `County`): `RowsRead`, `RowsAccepted`,
  `RowsRejected`, `RowsSynced`, `IngestSeconds`, `SyncSeconds`,
  `BatchFailed`.
- Alarms per county: a batch `FAILED`; a batch stuck in `PARSING` or
  `SYNCING` longer than 2× its normal duration; reject rate above X% (set per
  county after the first dry-run); DLQ depth > 0.
- Notifications: SNS topic `jury-import-failures-{env}`. The message body
  carries the county, uploadId, stage, and error class, with no PII.
- Dashboard: one row per county showing the last batch, its status, counts,
  and age.

---

## 16. Phased Rollout

Each phase is 1–2 sprints. Phases ship in order.

### Phase 0: Stop the bleeding (before a second county goes live)
| Task | Acceptance |
|---|---|
| Remove `tenant_db_secret_name`/`kms_key_id` from `_OVERRIDABLE` | Payload override ignored (unit test) |
| Sync refuses to run unless the invocation names a county, and filters the scan by `county_code` (interim, still a Scan) | No cross-county rows staged in the isolation test |
| Cleanup deletes only items it staged **and** whose `county_code` matches | Other county's items survive |
| Idempotency check moved to `tblReconstitution.Note`; errors no longer swallowed | Re-run with the same key is skipped against a real DB |
| Confirm the KDOR list; fix MC/MP + Mitchell | Unit test on the map |
| Doc drift fixes (§2) | — |

Risk: the interim filter still Scans. That's slow, but correct. Rollback:
redeploy the previous image; there are no data changes.

### Phase 1: Registry and auth
| Task | Acceptance |
|---|---|
| `ImportCounties` projection job (sync λ, hourly) | Matches `CountyConnection` active rows |
| Secret adapter for the Justice360 connection-string format | Dry-run against one county using its `JuryConnectionName` secret |
| Cognito authorizer, county resolution, restricted CORS | Auth integration tests pass |

Risk: the Cognito token shape is unknown (Open Questions #7). Rollback:
detach the authorizer (config only).

### Phase 2: Keys, S3, and streaming ingestion
| Task | Acceptance |
|---|---|
| New tables + key builders | Unit tests |
| County-scoped presigned URL + tags + bucket policy | A PUT with a wrong tag is denied |
| Streaming parser, per-row check, reject file, envelope encryption, batch status | Load test passes on the chosen runtime |

Risk: runtime budget. Rollback: the old table and old ingestion path stay
deployable until Phase 5.

### Phase 3: Per-county sync orchestration
| Task | Acceptance |
|---|---|
| SQS FIFO + DLQ; sync triggered by `PARSED` | Ordering and isolation integration tests pass |
| `Query` by partition; TTL-based cleanup | No Scan calls in the sync (CloudWatch or test spy) |
| Concurrency raised from 1 to N | Two counties sync in parallel in staging |

Risk: DB contention on shared servers. Rollback: set concurrency back to 1;
FIFO still orders.

### Phase 4: Frontend, stats, and observability
| Task | Acceptance |
|---|---|
| Batch status page, reject download, per-county records/stats | County A user never sees B's counts or rows |
| Metrics, alarms, SNS | Test alarm fires per county |

Rollback: UI-only, redeploy the previous bundle.

### Phase 5: Migration and retirement
| Task | Acceptance |
|---|---|
| Backfill job + verification report | Counts equal per county |
| Voter rows under review released per county | Human sign-off recorded on the batch |
| Old table retired | One clean cycle per enabled county |

Rollback: see §13.

---

## 17. Open Questions (not confirmable from code)

1. **Who uploads the statewide JuryListing file?** Each county (and we
   quarantine the other 104 counties' rows), or one state-level operator? The
   file content says statewide; the process owner is unknown.
2. **Is `CountyConnection.CountyCode` the KDOR 2-letter code** (`MN`, `RL`,
   …), or an internal code? Per-row validation depends on it.
3. **Do counties share SQL servers in practice?** The schema allows it
   (`JuryConnectionName` + `JuryDatabase`); the actual production layout
   isn't in either repo.
4. **How are the existing `JuryConnectionName` secrets named?** The IAM
   prefix scoping in §9 depends on it.
5. **Where does the React UI live now** that `frontend/` is gone from this
   repo, and is it embedded in Justice360 like the Questionnaire Builder?
6. **Official KDOR county code list**, to settle MC/MP/Mitchell (gap #12).
7. **How is the Cognito `CountyID` claim issued** for this app's users, if
   at all? The Justice360 staff Cognito path carries no county claim today;
   the county comes from `Environment.CountyID` (claim or `CID` cookie).
8. **Reconstitution "cycle" vs upload "batch".** If a county sends DL and
   Voter files separately, is that one cycle or two (CCJJ-592)? This decides
   whether sync merges several batches into one reconstitution.

---

## 18. Decisions Needed

| # | Decision | Options | Recommendation | Owner |
|---|---|---|---|---|
| 1 | Statewide JuryListing rows for other counties | (a) Quarantine to the reject file; (b) state-admin role fans one file out to all counties | **(a) now**; (b) as a later, separately-authorized role if Open Question #1 says a state operator uploads | Product + Court admin |
| 2 | How ingestion reads the registry | (a) Derived `ImportCounties` cache; (b) give ingestion VPC + setup-DB access | **(a)**; sync always re-checks against `CountyConnection` | Import tech lead |
| 3 | Sync orchestration | (a) SQS FIFO `MessageGroupId=county`; (b) Step Functions Map + DDB lock | **(a)** (per-county serialization built in) | Import tech lead |
| 4 | Per-user county-membership check | (a) None: trust the authenticated county (current Justice360 decision); (b) check `CountyConnectionMembership` per user | **(a) by default**, matching Justice360. Revisit (b) as its own feature if the risk appetite changes | Security + Product |
| 5 | Existing `v1:` ciphertext | (a) Leave as-is, decrypt with `legacy_id`; (b) re-encrypt to `v2` during backfill | **(a)**. It drains naturally as batches sync and expire | Import tech lead |
| 6 | Ingestion runtime for large files | (a) Lambda streaming; (b) Step Functions Distributed Map; (c) Fargate | **(a)**, with (b) as the fallback if the Phase 2 load test misses 900 s | Import tech lead |
| 7 | Retention of synced DynamoDB batches and reject files | TTL 7 / 30 / 90 days | **30 days** for records, **90 days** for reject files; S3 originals follow court retention | Court records owner |
| 8 | County code fix | (a) MC=Mitchell, MP=McPherson; (b) other | Confirm with KDOR, then **(a)** if confirmed | Import owner |
| 9 | One cycle = one batch? | (a) Yes, one file per reconstitution; (b) multiple batches merged into one cycle | Depends on CCJJ-592. **(b)** if DL and Voter files arrive separately | Product + Import tech lead |
