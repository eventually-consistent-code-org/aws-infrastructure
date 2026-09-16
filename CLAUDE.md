# aws-infrastructure

Control Tower landing zone + account vending for chain-vote, in OpenTofu via Terragrunt. The plan has pivoted (see `docs/Architectures/fips_140-3_quarantine.md`): AFT is out, vending is a GitLab CI + OpenTofu pipeline, home partition is GovCloud (`us-gov-west-1`, secondary `us-gov-east-1`), state is GitLab-managed HTTP state. **The code has not caught up** — everything under `management/` still targets commercial `us-east-1` with S3/DynamoDB state. Treat the doc as intent and the HCL as current reality; do not "fix" one to match the other without asking.

## Commands

```bash
# 1. state backend — management-account creds, run BEFORE any tofu init / terragrunt run-all (idempotent)
./scripts/bootstrap-state.sh          # prints TF_STATE_BUCKET / TF_STATE_LOCK_TABLE to export

# 2. read-only CT enrollment blocker check (Config recorders, SCPs, etc.) — expect 5 PASS, exit 0
./scripts/preflight.sh

# 3. CT landing zone is a ~45 min console session — no IaC path. Follow docs/runbooks/01-ct-landing-zone.md

# 4. ct-bootstrap: zero resources, data sources only; needs terraform.tfvars (copy from .example, gitignored)
cd management/ct-bootstrap && terragrunt init && terragrunt apply

# 5. OU tree (8 OUs: chain-vote-ai / chain-vote-voting x dev/staging/prod) — plan works pre-CT, apply does not
cd management/ou-structure && terragrunt plan
cd management/ou-structure && terragrunt apply

# 6. register OUs with CT — manual, strictly sequential. Follow docs/runbooks/02-ou-ct-registration.md
```

No test suite, no CI, no lint config in this repo yet. Validate with `terragrunt validate` in the unit dir.

## Layout

- `terragrunt.hcl` — root. Pins `terraform_binary = "tofu"`, generates `versions.tf` / `provider.tf` (region `us-east-1`), `remote_state` = S3 `chain-vote-tofu-state-<acct>` + DynamoDB `chain-vote-tofu-locks`.
- `management/ct-bootstrap/` — data-source wrapper over the CT landing zone + org. **Must stay zero-resource** (CT self-manages via StackSets; resource blocks will fight it).
- `management/ou-structure/` — `aws_organizations_organizational_unit` x8. `dependency "ct_bootstrap"` with `mock_outputs_allowed_terraform_commands = ["validate","plan"]`.
- `management/state-bootstrap/` — empty `.gitkeep` stub. The real bootstrap is `scripts/bootstrap-state.sh`.
- `scripts/` — `bootstrap-state.sh`, `preflight.sh`.
- `docs/runbooks/` — the two human checkpoints. `docs/Architectures/` — the GovCloud/FIPS plan (the current design of record).
- `.planning/` — legacy GSD artifacts (PROJECT/ROADMAP/STATE/research). Useful history; the tooling that generated them is not installed.
- `Alarms SSM Deploy Role.yaml` — untracked CloudFormation at repo root (GitLab-runner OIDC deploy role for alarms/SSM). Purpose in this repo unexplained; not referenced anywhere. Ask before touching or committing.

## Pins

| What | Version |
|------|---------|
| OpenTofu | `1.12.2` (constraint `>= 1.8, < 2.0`) |
| Terragrunt | `1.0.8` |
| hashicorp/aws | `6.50.0` (constraint `>= 6.0.0, < 7.0.0`) |
| tofuutils/tenv | latest — version manager for the above |

AFT module `aws-ia/terraform-aws-control_tower_account_factory 1.20.1` was the old pin; AFT cannot deploy in GovCloud and is dropped.

## Gotchas

- **State backend contradiction**: commit `6e02b18` moved the design to GitLab HTTP state, but root `terragrunt.hcl` still wires S3/DynamoDB and `bootstrap-state.sh` still creates them. Until someone rewrites the root, the S3 path is what actually runs.
- **Two credential contexts, no `AWS_PROFILE` convention**: GovCloud accounts are born from the *commercial* management account (`CreateGovCloudAccount`), so commercial creds and GovCloud creds are both needed. Nothing in the repo names profiles — state which context you are in before running anything.
- `TG_TF_PATH=tofu` in CI (root already sets `terraform_binary`, but both `tofu` and `terraform` on PATH makes Terragrunt's default non-deterministic).
- GovCloud pipeline jobs need `AWS_USE_FIPS_ENDPOINT=true` and partition-aware ARNs via `data.aws_partition` — hardcoded `arn:aws:` is a silent no-match in SCPs/trust policies there (`arn:aws-us-gov:`).
- `bootstrap-state.sh` deliberately omits `--create-bucket-configuration`; `us-east-1` rejects it. Don't "add it back".
- Never `terragrunt run-all apply` before the state bucket exists — parallel units race on bucket creation.
- `*.tfvars` are gitignored; `ct-bootstrap` needs a real `terraform.tfvars` from the account IDs captured in runbook 01.
- OU registration with CT is sequential: parents first, wait for `SUCCEEDED`, then children. See runbook 02 for the OUs you must NOT register.
- Phase 1 is "code complete — awaiting CT console steps" (`.planning/STATE.md`). Nothing in `management/` has been applied against a live CT.
- GitHub owner is `eventually-consistent-code` (renamed from BigJiggity in `32c2a0b`).

## Docs / runbooks

- `docs/runbooks/01-ct-landing-zone.md` — preflight, state gate, trusted access, console deploy, verification.
- `docs/runbooks/02-ou-ct-registration.md` — capture OU IDs from outputs, register with CT in order, troubleshoot.
- `docs/Architectures/fips_140-3_quarantine.md` — GovCloud + FIPS 140-3 + forensics quarantine plan, 10 phases. Read this before proposing any new vending/state/CI work.
- `.planning/research/PITFALLS.md` — Terragrunt/OpenTofu/CT pitfalls (binary detection, state races, etc.).

## Tracker

Beads (`bd`), not cairn — `.beads/` exists, no `cairn.json`. Rules and session-close protocol live in `AGENTS.md`; `bd prime` runs on SessionStart/PreCompact via `.claude/settings.json`.
