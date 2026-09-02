# Dodo - Attack

**Technique:** CICD-SEC-1, Insufficient Flow Control Mechanisms. A security
scanner's own configuration lives inside the same repository it scans,
letting the party being checked also rewrite the rules of the check.

## Setup

1. Cloned `Wonderland/dodo` (direct write access to `main`, no fork or PR
   needed):
   ```bash
   git clone http://thealice:thealice@localhost:3000/Wonderland/dodo.git
   cd dodo
   ```
2. Changed the `dodo` bucket's ACL in `main.tf`:
   ```diff
   -  acl           = "private"
   +  acl           = "public-read"
   ```

## Run 1: `check` override plus `soft-fail` together

Added `.checkov.yaml` at the repo root:

```yaml
soft-fail: true
check:
  - MY_CHECK
```

The intent (matching the technique documented in this project's own
official solution): restrict Checkov to a single, nonexistent check ID so
no real rule ever evaluates, with `soft-fail` as a backup in case anything
still failed. Pushed directly to `main` and triggered the job.

**Result: deployment succeeded, flag printed.** But the console output
showed something more precise than "the scanner was bypassed": Checkov
still ran its real four hardcoded checks, `--check
CKV2_AWS_39,CKV2_AWS_38,CKV_AWS_20,CKV_AWS_57` on the command line took
precedence over the config file's `check:` override, and it correctly
detected and reported the violation:

```
Check: CKV_AWS_20: "S3 Bucket has an ACL defined which allows public READ access."
	FAILED for resource: aws_s3_bucket.dodo
```

Despite that correct detection, the pipeline proceeded to `terraform
apply` and the bucket's public ACL was confirmed live, printing the flag.

## Run 2: isolating the variable

To determine which line in `.checkov.yaml` actually mattered, removed
`soft-fail: true` and kept only the (apparently ineffective) `check:`
override:

```yaml
check:
  - MY_CHECK
```

Pushed and triggered the job again.

**Result: pipeline failed.** Checkov ran the same real four checks, found
the same violation, and this time exited with a non-zero status. Jenkins
treated that as a stage failure and stopped before Terraform ran at all:

```
ERROR: script returned exit code 1
Finished: FAILURE
```

## Conclusion

Two isolated runs establish precisely what does and doesn't matter here:

- The `check:` override (restricting Checkov to a nonexistent rule ID) had
  no effect in this environment, since the pipeline's own `--check`
  command-line flag takes precedence over the same setting in a config
  file.
- `soft-fail: true` alone is fully sufficient to defeat the pipeline. The
  scanner detects and reports the real violation accurately in both runs;
  what changes is only whether that result is allowed to stop the
  pipeline. The pipeline's entire enforcement mechanism is Checkov's
  process exit code, and that exit code is itself configurable from
  inside the repository under test.
