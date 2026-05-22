# compliant-s3

A Terraform primitive that provisions a compliant AWS S3 data bucket and a separate S3 log bucket. This design satisfies the baseline requirements for NIST SP 800-53 controls SC-28, AU-3, AU-6, CM-6, and AC-3 while producing machine-readable evidence through Terraform plan/state JSON and outputs.

## Purpose

This primitive creates:
- a **primary S3 bucket** for data storage
- a **secondary log S3 bucket** to receive server access logs from the primary bucket

Both buckets enforce a shared compliance baseline:
- AES-256 server-side encryption (SC-28)
- versioning enabled on the primary bucket (AU-6)
- full public access block (AC-3)
- provider-level compliance tags on every resource (CM-6)
- server access logging from primary to log bucket (AU-3)

## Architecture

The primitive uses provider-level default tags and Terraform-managed AWS resources to keep the architecture simple, auditable, and repeatable.

```
                ┌──────────────────────────┐
   default_tags │  Project / Environment / │
   (provider)   │  ManagedBy / ComplianceScope
                └──────────────┬───────────┘
                               │
            ┌──────────────────┴───────────────────┐
            ▼                                      ▼
  ┌────────────────────┐               ┌─────────────────────┐
  │ aws_s3_bucket      │   logs (AU-3) │ aws_s3_bucket (log) │
  │ primary            │──────────────▶│ ACL: log-delivery   │
  │ AES256 (SC-28)     │               │ write               │
  │ versioning ON      │               │ AES256              │
  │ public-block (AC-3)│               │ public-block (AC-3) │
  └────────────────────┘               └─────────────────────┘
```

### Resource map

- `provider "aws"` with `default_tags`
- `random_id.bucket_suffix` to produce a unique bucket suffix
- `aws_s3_bucket.primary`
- `aws_s3_bucket_server_side_encryption_configuration.primary`
- `aws_s3_bucket_versioning.primary`
- `aws_s3_bucket_public_access_block.primary`
- `aws_s3_bucket.log`
- `aws_s3_bucket_ownership_controls.log`
- `aws_s3_bucket_acl.log`
- `aws_s3_bucket_server_side_encryption_configuration.log`
- `aws_s3_bucket_public_access_block.log`
- `aws_s3_bucket_logging.primary`

## Compliance control mapping

- **SC-28**: Server-side encryption is enforced with `AES256` for both primary and log buckets.
- **AU-3**: S3 server access logging is enabled from the primary bucket to the log bucket.
- **AU-6**: Versioning is enabled on the primary bucket to support object change history and forensic reconstruction.
- **CM-6**: `default_tags` on the provider create a consistent tagging baseline for `Project`, `Environment`, `ManagedBy`, and `ComplianceScope`.
- **AC-3**: Full public access block is enabled for both buckets to prevent accidental public exposure.

## Tech stack

- Terraform `>= 1.6`
- AWS provider `hashicorp/aws` `~> 5.0`
- Random provider `hashicorp/random` `~> 3.6`
- AWS S3 resources for bucket creation, encryption, versioning, access control, public access block, and logging

## Inputs

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `project_name` | `string` | Short project identifier used in bucket names and tags | n/a |
| `environment` | `string` | Deployment environment; must be one of `dev`, `staging`, `prod` | n/a |
| `bucket_suffix` | `string` | Optional suffix to force a specific bucket name; if empty, a random suffix is generated | `""` |

### Validation

- `project_name` must match `^[a-z][a-z0-9-]{2,20}$`
- `environment` must be one of `dev`, `staging`, or `prod`

## Outputs

| Name | Description |
|------|-------------|
| `bucket_arn` | ARN of the primary data bucket |
| `bucket_name` | Name of the primary data bucket |
| `log_bucket_arn` | ARN of the log bucket |
| `encryption_algorithm` | Detected encryption algorithm for the primary bucket (SC-28 attestation) |

## Usage

This primitive is intended to be applied from the `primitives/compliant-s3` directory.

Example variable assignment:

```hcl
project_name = "testi"
environment  = "dev"
# bucket_suffix = "custom123"  # optional
```

Apply with Terraform:

```bash
terraform init
terraform apply -auto-approve
```

## Machine-readable evidence

Evidence is produced by Terraform state and plan JSON, along with an explicit `encryption_algorithm` output.

Existing evidence files in this repository:
- `evidence/plan.json`
- `evidence/state.json`

Example commands to generate evidence:

```bash
terraform plan -out=tfplan
terraform show -json tfplan > evidence/plan.json
terraform show -json terraform.tfstate > evidence/state.json
```

The output `encryption_algorithm` provides an explicit attestable value for SC-28.

## Notes

- The log bucket is intentionally separate from the primary bucket to preserve audit logs and support independent access controls.
- The log bucket uses `aws_s3_bucket_acl.log` with `log-delivery-write` and `aws_s3_bucket_ownership_controls.log`.
- A KMS-based encryption option is present in `main.tf` as commented code for future enhancement, but the current implementation uses AES-256.

## Recommended review items

- Verify tag values in `default_tags` meet your organizational CM-6 policy.
- Confirm the log bucket receives access logs under `access-logs/`.
- If you need stricter encryption auditing, enable the commented KMS section and add a KMS key resource.
