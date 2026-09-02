# Dodo - Vulnerable Baseline

**OWASP mapping:** CICD-SEC-1, Insufficient Flow Control Mechanisms

**Target:** `wonderland-dodo` Jenkins job, backed by the `Wonderland/dodo`
Gitea repository. Unlike every other challenge in this project, direct
push access to `main` is permitted, no read restriction and no PR/fork
step required.

**Context:** the job has no Jenkinsfile; it runs a "Scan and Deploy" stage
configured directly in Jenkins. That stage runs Checkov (a real,
open-source static analysis tool for infrastructure-as-code) against the
repo's Terraform files, then applies the Terraform against LocalStack (a
local AWS emulator) if Checkov succeeds, then queries the resulting S3
bucket's actual ACL to confirm whether it is genuinely publicly readable.

**Baseline `main.tf` (relevant resource):**

```hcl
resource "aws_s3_bucket" "dodo" {
  bucket        = var.bucket_name
  acl           = "private"
  versioning {
    enabled = true
  }
  replication_configuration {
    role = aws_iam_role.replication.arn
    rules {
      id     = "foobar"
      status = "Enabled"
      destination {
        bucket        = aws_s3_bucket.backup.arn
        storage_class = "STANDARD"
      }
    }
  }
}
```

A second bucket, `backup`, exists in the same file with its own separate
`aws_s3_bucket_public_access_block` guardrails. That bucket is unrelated to
this challenge and was left untouched throughout.

**The pipeline's actual Checkov invocation**, confirmed directly from
console output rather than assumed:

```
checkov -d . --check CKV2_AWS_39,CKV2_AWS_38,CKV_AWS_20,CKV_AWS_57
```

Four specific, real Checkov rule IDs hardcoded into the pipeline itself.
`CKV_AWS_20` is the relevant one here: "S3 Bucket has an ACL defined which
allows public READ access."

**Root cause (preview):** the pipeline's enforcement mechanism is nothing
more than treating Checkov's process exit code as pass/fail, standard,
reasonable pipeline design in isolation. The actual flaw is that Checkov,
by default, also reads a `.checkov.yaml` configuration file from the
directory it scans, and nothing separates that configuration from the
infrastructure code under test: both live in the same repository, editable
by the same account, at the same trust level.
