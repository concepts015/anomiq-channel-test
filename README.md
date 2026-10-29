# anomiq-channel-test

The public test repository for AnomIQ v2's GitHub Actions execution channel (ADR-019, Gate 2 proof B).
It holds one workflow, `.github/workflows/rollback-exec-target.yml`, and nothing else of value. It is a
reference and a test fixture, not a product: the customer owns the real pipeline.

**What the workflow does.** AnomIQ starts it with `workflow_dispatch` at the tag `channel-rollback-v1`. The
first job asks AnomIQ's start gate whether this run may act, using the workflow's own OIDC token, and
stops unless the answer is `active`. The second job assumes an IAM role in the customer's AWS account
by OIDC and starts the SSM Automation document `AnomIQ-ECS-Service-Rollback` with the vendor token as
`ClientToken`, so the effect happens once however often the run is repeated. The run is named
`anomiq <step key> <dispatch hash>`; AnomIQ finds the run by that name and reads the hash back.

**What it deliberately is not.** No other trigger. No `environment:`. Every job runs on the operator's own self-hosted runner (label `anomiq-channel-test`), never on a GitHub-hosted one;
the only trigger is `workflow_dispatch`, so a pull request from a fork cannot start a job on it. No third-party action that is not pinned by full commit SHA. No input is ever
placed inside a script: inputs reach scripts only as environment variables, after they are checked
against a pattern.

## Setup (done by the repository owner, once)

The commands are in `publish.sh` (it prints them; it changes nothing). In short:

1. The repository already exists and is public. Add `.github/workflows/rollback-exec-target.yml` and
   this README on the default branch.
2. Create the annotated tag `channel-rollback-v1` on that commit and protect it with a tag ruleset
   (no update, no delete). The customer's IAM role trusts exactly this tag.
3. Set four repository **variables** (none is a secret): `ANOMIQ_GATE_AUDIENCE`, `ANOMIQ_GATE_BASE_URL`,
   `ANOMIQ_CHANNEL_ROLE_ARN` (stack output `ChannelRollbackRoleArn`), `ANOMIQ_AWS_REGION`.
4. Install AnomIQ's execute App on this repository only (permissions `Actions: write`, `Metadata: read`).

AnomIQ pins the channel by the tag's commit and the workflow file's blob SHA. It reads both again
before every dispatch and after every run; a moved tag or an edited workflow stops the dispatch and
revokes the authorization.

## The inputs

`cluster`, `service`, `expected_current`, `target_revision`, plus the reserved envelope
`anomiq_attempt_id`, `anomiq_step_key`, `anomiq_vendor_token` (a UUID), `anomiq_plan_hash`,
`anomiq_authorization_id`, `anomiq_dispatch_hash`, `anomiq_gate_url`. Nothing is free text. The gate URL
must equal `ANOMIQ_GATE_BASE_URL` + attempt id + `/gate`, so whoever dispatches cannot send the
identity token anywhere else.

## Status

Not run yet. Written against the contract in the AnomIQ repository (`docs/v2/servicenow/CHANNEL_FLOW_SPEC.md`
for the envelope, ADR-019 for the flow). Two details are fixed by AnomIQ's gate and must be checked on
the first live run: the gate body (`dispatch_hash`, `run_ref` = `<run id>-<run attempt>`) and the
`job_workflow_ref` claim in the IAM trust.
