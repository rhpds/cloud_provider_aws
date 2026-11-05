# AWS Nested Virtualization for OpenShift Workers Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** An opt-in AgnosticV variable, `openshift_aws_nested_virtualization_enable`, that turns on EC2 nested virtualization on an AWS OpenShift cluster's worker instances. It works in both the direct `openshift-cluster` path and the pool + `scale_ocp_workers` path, and it fails the provision if any worker ends up without `/dev/kvm`.

**Architecture:** Every `openshift-cluster` AWS stack creates a scoped IAM user. It can only stop, start, and modify CPU options on this cluster's `*-worker-*` instances. Its keys go into provision data. A new role, `agnosticd.cloud_provider_aws.nested_virtualization`, reads the worker Machines from the cluster and stops every target at once. It sets `NestedVirtualization=enabled`, starts the targets, and then checks `/dev/kvm` with one short-lived pod per node. `openshift-cluster/post_software.yml` calls the role in the direct path. `scale_ocp_workers` calls it in the scale path, and there `worker_only` follows the new variable.

**Tech Stack:** Ansible (`kubernetes.core` k8s modules), the AWS CLI v2 called through `ansible.builtin.command` in the `ee-multicloud` EE, CloudFormation (Jinja template), and AgnosticV YAML.

**Spec:** `cloud_provider_aws/docs/superpowers/specs/2026-11-05-aws-nested-virtualization-design.md` (approved, commit `e96987e`). Jira: GPTEINFRA-18454.

## Repos, branches, commits

| Repo (local path) | Branch | Base |
|---|---|---|
| `/Users/nstephan/devel/git/agDv2/cloud_provider_aws` | `GPTEINFRA-18454-aws-nested-virtualization` (exists, holds the spec) | `main` |
| `/Users/nstephan/devel/git/agDv2/agnosticd-v2` | `GPTEINFRA-18454-aws-nested-virtualization` (new) | `main` |
| `/Users/nstephan/devel/git/agnosticv` | `GPTEINFRA-18454-aws-nested-virtualization` (new) | `GPTEINFRA-18402-ocp-fsc-showroom` (it already has the m7i switch); confirm the base with Nate in Task 7 |

- Commit subjects use `[GPTEINFRA-18454] Imperative description`, with 75 characters or fewer after the key.
- Confirm the Jira key with Nate before **every** commit.
- Before **every** push, state the branch, remote, and commits, and get Nate's explicit approval.
- Each commit ends with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- zsh: quote git refspecs that contain paths, for example `"branch:path"`.

## Decisions this plan makes (the spec left them to the builder)

- **AWS CLI, not boto3 or a module.** The local `aws-cli/2.39.3` has `modify-instance-cpu-options --nested-virtualization enabled|disabled`. Task 1 confirms the EE has it.
- **The idempotency field** is `Reservations[].Instances[].CpuOptions.NestedVirtualization`, with values `enabled` or `disabled`. It may be absent on instances that never had it set. Treat anything other than `enabled` as not enabled.
- **The instance-type check** looks for `nested-virtualization` in `InstanceTypes[].ProcessorInfo.SupportedFeatures`.
- **CoreCount/ThreadsPerCore:** the CLI marks them optional, but the AWS doc example passes them. The role always sends the instance's current `CpuOptions.CoreCount` and `CpuOptions.ThreadsPerCore` from `DescribeInstances`, so it never depends on either answer.
- **The `/dev/kvm` check** runs one pod per node in a temporary namespace `agnosticd-nested-virt-check`. Each pod sets `nodeName`, tolerates everything, mounts a `hostPath` of type `CharDevice` at `/dev/kvm`, and runs `test -c /dev/kvm`. The image is the `cli` ImageStream in `openshift`, read from `status.tags[?tag=='latest'].items[0].dockerImageReference`. That image comes from the release, so nodes pull it with the cluster pull secret and the internal registry isn't involved. A node passes only if its pod reaches `Succeeded`. The namespace is deleted in an `always:` block.
- **Cluster auth is inherited from the caller.** The role sets no `module_defaults` for auth. In the scale path it runs inside `scale_ocp_workers`' `module_defaults` block. In the direct path, the play sets `KUBECONFIG`.
- **Provision data:** the spec says `agnosticd.core.agnosticd_user_data`, but in agnosticd-v2 that name is a *lookup*. The writer is `agnosticd.core.agnosticd_user_info` with `data:`, which `post_infra.yml` already uses. The plan uses `agnosticd_user_info`.
- **AWS waits** use the CLI waiters `aws ec2 wait instance-stopped` and `instance-running` (40 × 15 s by default). Start retries on capacity errors: 20 × 30 s.

## Global Constraints

- New variable: `openshift_aws_nested_virtualization_enable`, default `false` (in `openshift-cluster/aws/default_vars.yml` and `scale_ocp_workers/defaults/main.yml`).
- New template gate: `aws_nested_virt_iam_user_create`, default `false` in `template_generate`, `true` in `openshift-cluster/aws/default_vars.yml`.
- Credential names, the same everywhere: `aws_nested_virt_access_key_id`, `aws_nested_virt_secret_access_key`.
- IAM policy: allow `ec2:StopInstances`, `ec2:StartInstances`, `ec2:ModifyInstanceCpuOptions` on `arn:aws:ec2:*:*:instance/*` with `StringEquals aws:ResourceTag/guid = <guid>` **and** `StringLike aws:ResourceTag/Name = *-worker-*`. Allow `ec2:DescribeInstances` and `ec2:DescribeInstanceTypes` on `*`. Nothing else: no `RunInstances`, no `CreateTags`, no IAM.
- The role takes no `guid` or `aws_region` input. It reads the region from Machine `spec.providerSpec.value.placement.region`.
- Candidates: Machines in `openshift-machine-api` labeled `machine.openshift.io/cluster-api-machine-role: worker` with phase `Running`. Control-plane Machines are never touched or checked.
- The role does not install packages at runtime.
- No credentials, endpoints, or account-specific values in committed files. Never print the secret key, and use `no_log: true` on any task whose args or result contain it.
- Supported families are all Intel: M7i, M7i-flex, M8i, M8id, M8i-flex, C7i, C7i-flex, C8i, C8id, C8i-flex, R7i, R7iz, R8i, R8id, R8i-flex, X8i, I7i, I7ie. The role doesn't hardcode this list. It asks `DescribeInstanceTypes`.
- `ocp4_workload_openshift_virtualization_enabled_nested_kvm` stays `true`. Do not change it.
- Lint: `yamllint .` and `ansible-lint` must pass in `cloud_provider_aws`. Run them with `tox -c tests/static`, the same as CI.

## Review Focus

1. **Workers not settled yet in the direct path.** A Machine can still be `Provisioned`, or `Running` without a `nodeRef`, when Step 005.2 ends. The role must wait for all worker Machines to be `Running` with a `nodeRef`, with a bounded wait, and fail if a Machine is `Failed`. It must not silently skip them. Covered by `test_candidates_not_ready` (Task 3).
2. **Mixed state on re-run.** Some workers are already enabled and some aren't, for example after a partial failure or in the scale path. Only the not-enabled instances get stopped, but `/dev/kvm` is checked on **all** candidates. Covered by `test_targets_mixed` (Task 3).
3. **Old or zero-replica worker MachineSets.** A pool's `m6a` worker MachineSets at 0 replicas have no Machines, so they must not cause an unsupported-type failure. A *Running* `m6a` worker must. Covered by `test_unsupported_types` (Task 3), with the candidate list built only from Running Machines.
4. **Keys present but empty.** A pool from before this change propagates `aws_nested_virt_*` as missing or `""`. Pre-flight must fail with the "pool cluster created before nested virtualization support" message. Undefined alone isn't enough to catch it. Covered by `test_preflight_missing_keys` (Task 3).
5. **Secrets in logs.** `post_infra` must not add the nested-virt secret to the existing debug loop that prints Route53 credentials. Role tasks that pass keys through `environment:` must not echo them. Covered by a grep check in Task 4 Step 6 and Task 6 Step 4.

---

### Task 1: EE capability check (report only, no commit)

**Files:** none. The output is a short report to Nate in the session, plus a Jira comment if Nate wants one.

**Interfaces:**
- Produces: a yes/no answer to "can `quay.io/agnosticd/ee-multicloud:chained-2026-05-04` run the role as written?", and the minimum `awscli` and `botocore` versions that know `NestedVirtualization`.

- [ ] **Step 1: Pull the image** (amd64 on an arm64 Mac)

Run: `podman pull --platform linux/amd64 quay.io/agnosticd/ee-multicloud:chained-2026-05-04`
Expected: the pull completes.

- [ ] **Step 2: Check the CLI and botocore inside the EE**

Run:
```bash
podman run --rm --platform linux/amd64 quay.io/agnosticd/ee-multicloud:chained-2026-05-04 bash -c '
  aws --version;
  aws ec2 modify-instance-cpu-options help | grep -c -- --nested-virtualization;
  python3 -c "import botocore, boto3, botocore.session as s; print(botocore.__version__, boto3.__version__);
m = s.get_session().get_service_model(\"ec2\");
print(\"NestedVirtualization\" in m.shape_for(\"ModifyInstanceCpuOptionsRequest\").members)"'
```
Expected: versions are printed, the grep count is `≥1`, and the last line is `True`. If the CLI is v1 (`aws-cli/1.x`), it uses the EE's botocore, so the botocore line decides.

- [ ] **Step 3: Find the minimum versions**

Search the `aws/aws-cli` v2 `CHANGELOG.rst` and the `boto/botocore` `CHANGELOG.rst` for the first `api-change:ec2` entry that mentions nested virtualization. Fetch both with `gh api` or WebFetch.
Expected: one version each for aws-cli v2 and botocore.

- [ ] **Step 4: Report to Nate**

Report the EE versions found, whether they support the flag, and the minimum versions. If the image is too old, tell Nate that development continues with a local dev EE built in Task 8 Step 1, and that the EE maintainer needs the minimums. Don't change the role design either way.

---

### Task 2: Scoped IAM user in the CloudFormation template

**Files:**
- Modify: `cloud_provider_aws/roles/template_generate/defaults/main.yml`: add `aws_nested_virt_iam_user_create: false` with a comment.
- Modify: `cloud_provider_aws/roles/template_generate/templates/cloud_template.yaml.j2`: resources inside the existing `{% if secondary_stack is not defined %}` block after `Route53UserAccessKey` (around line 378), and outputs inside the matching block under `Outputs:` (around line 409).
- Create: `cloud_provider_aws/tests/template_generate/test_nested_virt_iam_user.yml`
- Create: `cloud_provider_aws/tests/template_generate/vars_minimal.yml`: the smallest set of variables that renders the template, with placeholder values only (`guid: abc12`, `base_domain: example.com`, and so on).

**Interfaces:**
- Produces: CloudFormation resources `NestedVirtUser` (`AWS::IAM::User`) and `NestedVirtUserAccessKey` (`AWS::IAM::AccessKey`), and stack outputs `NestedVirtUserAccessKey` (Ref) and `NestedVirtUserSecretAccessKey` (`Fn::GetAtt SecretAccessKey`). Task 6 reads these output names.

- [ ] **Step 1: Write the failing render test**

`test_nested_virt_iam_user.yml` is a localhost playbook. It loads `../../roles/template_generate/defaults/main.yml` and `vars_minimal.yml` through `vars_files`, and renders the template twice with `lookup('ansible.builtin.template', ..., template_vars={...}) | from_yaml`:
```yaml
# gate on
- ansible.builtin.assert:
    that:
    - "'NestedVirtUser' in _on.Resources"
    - "'NestedVirtUserAccessKey' in _on.Resources"
    - "'NestedVirtUserAccessKey' in _on.Outputs"
    - "'NestedVirtUserSecretAccessKey' in _on.Outputs"
    - _scoped.Action | sort == ['ec2:ModifyInstanceCpuOptions', 'ec2:StartInstances', 'ec2:StopInstances']
    - _scoped.Resource == 'arn:aws:ec2:*:*:instance/*'
    - _scoped.Condition.StringEquals['aws:ResourceTag/guid'] == 'abc12'
    - _scoped.Condition.StringLike['aws:ResourceTag/Name'] == '*-worker-*'
    - _read.Action | sort == ['ec2:DescribeInstanceTypes', 'ec2:DescribeInstances']
    - _read.Resource == '*'
    - _all_actions | length == 5   # nothing else granted
# gate off (default)
- ansible.builtin.assert:
    that:
    - "'NestedVirtUser' not in _off.Resources"
    - "'NestedVirtUserSecretAccessKey' not in _off.Outputs"
# gate on + secondary_stack defined
- ansible.builtin.assert:
    that:
    - "'NestedVirtUser' not in _secondary.Resources"
```
`_scoped` and `_read` are the two statements of `_on.Resources.NestedVirtUser.Properties.Policies[0].PolicyDocument.Statement`. `_all_actions` is the flattened action list across all statements.

- [ ] **Step 2: Run it and confirm it fails**

Run: `cd cloud_provider_aws && ansible-playbook tests/template_generate/test_nested_virt_iam_user.yml`
Expected: the gate-on assert fails with `'NestedVirtUser' in _on.Resources`. If rendering errors on an undefined variable, add the variable to `vars_minimal.yml` and run again until the assert is what fails.

- [ ] **Step 3: Add the gate default and the template resources**

Gate the resources and outputs with `{% if aws_nested_virt_iam_user_create | bool %}`, nested inside the existing `secondary_stack` blocks. Use policy name `NestedVirtAccess`. Write the policy exactly as the Global Constraints describe, with `guid` rendered from `{{ guid }}`. Copy the shape of the outputs from the `Route53User*` outputs.

- [ ] **Step 4: Run the test and lint**

Run: `ansible-playbook tests/template_generate/test_nested_virt_iam_user.yml && tox -c tests/static`
Expected: all asserts pass, and yamllint and ansible-lint are clean.

- [ ] **Step 5: Commit** (confirm the Jira key first)

```bash
git add roles/template_generate tests/template_generate
git commit -m "[GPTEINFRA-18454] Add scoped nested virt IAM user to the AWS stack template"
```

---

### Task 3: `nested_virtualization` role — selection and pre-flight logic

The decision logic lives in task files that turn input facts into output facts, so localhost playbooks can test it with fixture JSON and no cluster or AWS.

**Files:**
- Create: `cloud_provider_aws/roles/nested_virtualization/defaults/main.yml`
- Create: `cloud_provider_aws/roles/nested_virtualization/meta/main.yml` (copy the shape from `scale_ocp_workers` if it has one, otherwise a minimal `galaxy_info`)
- Create: `cloud_provider_aws/roles/nested_virtualization/tasks/select_candidates.yml`
- Create: `cloud_provider_aws/roles/nested_virtualization/tasks/select_targets.yml`
- Create: `cloud_provider_aws/roles/nested_virtualization/tasks/check_instance_types.yml`
- Create: `cloud_provider_aws/roles/nested_virtualization/tasks/preflight_keys.yml`
- Create: `cloud_provider_aws/tests/nested_virtualization/test_selection.yml` and `tests/nested_virtualization/fixtures/*.json` (sanitized: fake IDs like `i-0aaa...`, cluster `ocp-abc12`, region `us-east-2`)

**Interfaces:**
- Consumes: `aws_nested_virt_access_key_id`, `aws_nested_virt_secret_access_key` (role inputs).
- Produces (facts that Task 4 relies on):
  - `select_candidates.yml`: input `_nv_machines` (list of Machine resources). Outputs: `_nv_candidates`, a list of `{machine, instance_id, instance_type, region, node}`; `_nv_region` (str); `_nv_not_ready`, a list of machine names that aren't `Running` or lack a `nodeRef`; `_nv_failed`, a list of machine names in phase `Failed`.
  - `select_targets.yml`: input `_nv_describe` (parsed `aws ec2 describe-instances` JSON). Output: `_nv_targets`, a list of `{instance_id, core_count, threads_per_core}` for instances where `CpuOptions.NestedVirtualization != 'enabled'`.
  - `check_instance_types.yml`: input `_nv_describe_types` (parsed `describe-instance-types` JSON). Output: `_nv_unsupported_types`, a list of type names. It fails with `Instance type(s) <list> do not support nested virtualization; choose a supported Intel type (for example m7i)` when the list isn't empty.
  - `preflight_keys.yml`: fails with `aws_nested_virt_access_key_id/aws_nested_virt_secret_access_key are not set. The cluster was probably created before nested virtualization support; use a newer pool cluster.` when either key is undefined or empty.
  - Defaults: `nested_virt_machine_wait_retries: 60`, `nested_virt_machine_wait_delay: 10`, `nested_virt_start_retries: 20`, `nested_virt_start_delay: 30`, `nested_virt_node_ready_retries: 90`, `nested_virt_node_ready_delay: 10`, `nested_virt_kvm_check_retries: 60`, `nested_virt_kvm_check_delay: 5`, `nested_virt_kvm_check_namespace: agnosticd-nested-virt-check`. Each one gets a comment.

- [ ] **Step 1: Write the failing tests**

`test_selection.yml` runs on localhost and calls `include_role: name=agnosticd.cloud_provider_aws.nested_virtualization tasks_from=<file>`. Run it with `ANSIBLE_COLLECTIONS_PATH` pointing at a parent of an `ansible_collections/agnosticd/cloud_provider_aws` symlink to the repo, or use `ANSIBLE_ROLES_PATH=roles` and plain role name `nested_virtualization`. Pick one and document it in the playbook header. Each case is a block with `rescue` for the expected failures:
- `test_candidates_basic`: 2 Running workers with nodeRef, 3 master Machines, and 1 `infra` role Machine. Assert `_nv_candidates | map(attribute='instance_id') | sort == ['i-0aaa1', 'i-0aaa2']`, `_nv_region == 'us-east-2'`, and that `node` comes from `status.nodeRef.name`.
- `test_candidates_not_ready`: 1 Running and 1 `Provisioned` without a nodeRef. Assert `_nv_not_ready == ['ocp-abc12-worker-us-east-2a-xyz2']`.
- `test_candidates_failed`: 1 Machine in `Failed`. Assert `_nv_failed` lists it.
- `test_targets_mixed`: describe fixture with `i-0aaa1` `NestedVirtualization: enabled` and `i-0aaa2` without the key. Assert `_nv_targets == [{'instance_id': 'i-0aaa2', 'core_count': 4, 'threads_per_core': 2}]`.
- `test_targets_none`: all enabled. Assert `_nv_targets == []`.
- `test_unsupported_types`: types fixture with `m7i.2xlarge` supporting `nested-virtualization` and `m6a.2xlarge` without it. Assert the play failed and `_nv_unsupported_types == ['m6a.2xlarge']`.
- `test_supported_types`: only `m7i.2xlarge`. Assert no failure.
- `test_preflight_missing_keys`: run three ways (both undefined, access key `""`, secret `""`). Each must fail, and the message must contain `created before nested virtualization support`.
- `test_preflight_keys_ok`: dummy `AKIAEXAMPLE` and `CHANGEME`. No failure.

- [ ] **Step 2: Run them and confirm they fail**

Run: `ansible-playbook tests/nested_virtualization/test_selection.yml`
Expected: FAIL. The role or task files are missing.

- [ ] **Step 3: Write the four task files and `defaults/main.yml`** using the fact names and messages above. Use `selectattr` and `json_query` like the existing roles. `select_candidates.yml` filters on `metadata.labels['machine.openshift.io/cluster-api-machine-role'] == 'worker'` and doesn't decide whether to wait. Task 4 owns the wait loop.

- [ ] **Step 4: Run the tests and lint**

Run: `ansible-playbook tests/nested_virtualization/test_selection.yml && tox -c tests/static`
Expected: every case passes, and lint is clean.

- [ ] **Step 5: Commit** (confirm the Jira key)

```bash
git add roles/nested_virtualization tests/nested_virtualization
git commit -m "[GPTEINFRA-18454] Add nested virt role target selection and pre-flight checks"
```

---

### Task 4: `nested_virtualization` role — AWS actions, `/dev/kvm` verification, entry point

**Files:**
- Create: `cloud_provider_aws/roles/nested_virtualization/tasks/main.yml`
- Create: `cloud_provider_aws/roles/nested_virtualization/tasks/enable.yml` (stop, modify, start)
- Create: `cloud_provider_aws/roles/nested_virtualization/tasks/wait_for_nodes.yml`
- Create: `cloud_provider_aws/roles/nested_virtualization/tasks/verify_kvm.yml`
- Create: `cloud_provider_aws/roles/nested_virtualization/README.md` (short: purpose, the two callers, inputs, the "keep auth from caller" note, the non-goal that new machines are not covered)

**Interfaces:**
- Consumes: the Task 3 facts and messages, and the caller's k8s auth (`module_defaults` or `KUBECONFIG`).
- Produces: the role entry point `agnosticd.cloud_provider_aws.nested_virtualization`, with required vars `aws_nested_virt_access_key_id` and `aws_nested_virt_secret_access_key`. Tasks 5 and 6 include it.

`main.yml` order (one task or include per line):
1. `preflight_keys.yml`
2. `k8s_info` Machines in `openshift-machine-api` with label selector `machine.openshift.io/cluster-api-machine-role=worker`. Retry (`nested_virt_machine_wait_*`) until `select_candidates.yml` gives an empty `_nv_not_ready`. Simplest structure: an `until` on a `json_query` expression over the raw result that mirrors the same rule. After the loop, include `select_candidates.yml`, then fail if `_nv_failed` isn't empty, if `_nv_not_ready` is still not empty, or if `_nv_candidates` is empty (`openshift_aws_nested_virtualization_enable is true but the cluster has no worker Machines`).
3. `aws ec2 describe-instance-types --instance-types <unique types> --region <_nv_region> --output json`, then `check_instance_types.yml`.
4. `aws ec2 describe-instances --instance-ids <candidate IDs> ...`, then `select_targets.yml`.
5. `enable.yml` when `_nv_targets | length > 0`, then `wait_for_nodes.yml`.
6. `verify_kvm.yml` on **all** candidate nodes.
7. A debug summary: candidates, targets enabled this run, verified nodes.

All `aws` commands run through `ansible.builtin.command` with `environment: {AWS_ACCESS_KEY_ID: ..., AWS_SECRET_ACCESS_KEY: ..., AWS_DEFAULT_REGION: "{{ _nv_region }}"}`, with `changed_when: false` on the describes. Put the environment on a `block:` around steps 3–5. Don't use `no_log` on describe or modify tasks, because their output is needed for debugging and contains no secret.

- [ ] **Step 1: Write `enable.yml`**
  - `aws ec2 stop-instances --instance-ids <all target IDs>`, then `aws ec2 wait instance-stopped --instance-ids <same>`.
  - Loop over `_nv_targets`: `aws ec2 modify-instance-cpu-options --instance-id {{ item.instance_id }} --core-count {{ item.core_count }} --threads-per-core {{ item.threads_per_core }} --nested-virtualization enabled`.
  - `aws ec2 start-instances --instance-ids <all>`, with `retries: nested_virt_start_retries`, `delay: nested_virt_start_delay`, and `until: rc == 0`. Fail right away (no retry) unless stderr contains `InsufficientInstanceCapacity`. Write the `until` as `rc == 0 or 'InsufficientInstanceCapacity' not in stderr`, followed by a `failed_when` on `rc != 0`.
  - `aws ec2 wait instance-running --instance-ids <all>`.
  - Re-describe the targets and fail if any `CpuOptions.NestedVirtualization != 'enabled'`. Reuse `select_targets.yml` and assert `_nv_targets == []`.

- [ ] **Step 2: Write `wait_for_nodes.yml`.** Use `k8s_info` on Nodes and retry (`nested_virt_node_ready_*`) until every `_nv_candidates[].node` has a `Ready=True` condition. Copy the condition expression from `scale_ocp_workers/tasks/wait_for_nodes.yml`. Fail with the names of nodes that aren't ready.

- [ ] **Step 3: Write `verify_kvm.yml`**
  - In a `block:`, create Namespace `{{ nested_virt_kvm_check_namespace }}` with labels `pod-security.kubernetes.io/enforce: privileged` and `security.openshift.io/scc.podSecurityLabelSync: "false"`.
  - Read the `cli` ImageStream in namespace `openshift` and set `_nv_cli_image` from the `latest` tag's `items[0].dockerImageReference`. Fail if it's empty.
  - Loop over `_nv_candidates` and create one Pod per node: name `kvm-check-{{ item.node | regex_replace('[^a-z0-9-]', '-') | truncate(50, True, '') }}`, annotation `openshift.io/required-scc: privileged`, `spec.nodeName: {{ item.node }}`, `restartPolicy: Never`, `tolerations: [{operator: Exists}]`, one container using `_nv_cli_image` with `command: ["test", "-c", "/dev/kvm"]` and the `kvm` volume mounted at `/dev/kvm`, and volume `hostPath: {path: /dev/kvm, type: CharDevice}`.
  - `k8s_info` Pods in the namespace. Retry (`nested_virt_kvm_check_*`) until all are `Succeeded` or any is `Failed`. A missing device keeps a pod `Pending` (hostPath type check fails), which the retry limit catches.
  - Fail with `Nodes without /dev/kvm: <list>` for every pod that isn't `Succeeded`.
  - In `always:`, delete the namespace with `wait: false`.

- [ ] **Step 4: Write `main.yml` and the README** per the order above.

- [ ] **Step 5: Syntax check and lint**

Run: `ansible-playbook tests/nested_virtualization/test_selection.yml && ansible-playbook --syntax-check tests/nested_virtualization/test_entrypoint_syntax.yml && tox -c tests/static`. The last playbook is a two-line play that includes the role, created in this step.
Expected: everything passes and lint is clean. Task 8 runs this role against a real cluster.

- [ ] **Step 6: Secret check**

Run: `grep -rn "secret_access_key" roles/nested_virtualization`
Expected: the only matches are in `preflight_keys.yml`, `defaults` or README comments, and `environment:` values. None appears in a `debug`, `msg`, or `fail` text.

- [ ] **Step 7: Commit** (confirm the Jira key)

```bash
git add roles/nested_virtualization tests/nested_virtualization
git commit -m "[GPTEINFRA-18454] Enable nested virt on workers and verify /dev/kvm per node"
```

---

### Task 5: Hook the role into `scale_ocp_workers`

**Files:**
- Modify: `cloud_provider_aws/roles/scale_ocp_workers/defaults/main.yml:24-26`
- Modify: `cloud_provider_aws/roles/scale_ocp_workers/tasks/main.yml:76-82`
- Modify: `cloud_provider_aws/roles/scale_ocp_workers/README.md` (one short "Nested virtualization" usage example)

**Interfaces:**
- Consumes: the `agnosticd.cloud_provider_aws.nested_virtualization` entry point from Task 4.
- Produces: `openshift_aws_nested_virtualization_enable` (default `false`) and `worker_only: "{{ openshift_aws_nested_virtualization_enable | bool }}"`.

- [ ] **Step 1: Defaults.** Add `openshift_aws_nested_virtualization_enable: false` with a comment. Change `worker_only` to `"{{ openshift_aws_nested_virtualization_enable | bool }}"`, and say in its comment that it follows nested virtualization and that a CI can override it.

- [ ] **Step 2: Tasks.** Between "Wait for worker nodes to be Ready" and "Configure worker-only mode", add the following. It sits inside the existing `module_defaults` block, so the role inherits the auth.
```yaml
  - name: Enable nested virtualization on worker instances
    when: openshift_aws_nested_virtualization_enable | bool
    ansible.builtin.include_role:
      name: agnosticd.cloud_provider_aws.nested_virtualization
```

- [ ] **Step 3: Check that `worker_only` resolves**

Add a case to `tests/nested_virtualization/test_selection.yml`, or create a small `tests/scale_ocp_workers/test_worker_only_default.yml`, that loads `roles/scale_ocp_workers/defaults/main.yml`. Assert `worker_only | bool` is `false` by default, `true` with `openshift_aws_nested_virtualization_enable: true`, and `false` with the variable `true` plus `worker_only: false` as an extra var.
Run: `ansible-playbook tests/scale_ocp_workers/test_worker_only_default.yml && tox -c tests/static`
Expected: PASS, and lint is clean.

- [ ] **Step 4: Commit** (confirm the Jira key)

```bash
git add roles/scale_ocp_workers tests/scale_ocp_workers
git commit -m "[GPTEINFRA-18454] Run nested virt in scale_ocp_workers and default worker_only"
```

- [ ] **Step 5: Push `cloud_provider_aws`.** State the branch `GPTEINFRA-18454-aws-nested-virtualization`, the remote `origin` (`rhpds/cloud_provider_aws`), and the commits from Tasks 2–5. Push only after Nate approves. Don't open a PR yet; Task 8 tests first.

---

### Task 6: agnosticd-v2 `openshift-cluster` wiring

**Files:**
- Modify: `agnosticd-v2/ansible/configs/openshift-cluster/aws/default_vars.yml`: add a "Nested virtualization" section with `openshift_aws_nested_virtualization_enable: false` (with a comment: requires a supported Intel `worker_instance_type`, because the default family is `m7a`) and `aws_nested_virt_iam_user_create: true` (with a comment that it's created for every cluster so pool orders can opt in later).
- Modify: `agnosticd-v2/ansible/configs/openshift-cluster/post_infra.yml`: after the two "Save Route53User credentials" tasks (lines 30–50).
- Modify: `agnosticd-v2/ansible/configs/openshift-cluster/post_software.yml`: a new play between Step 005.2 (ends line 66) and Step 005.3 (line 68).

**Interfaces:**
- Consumes: the stack outputs `NestedVirtUserAccessKey` and `NestedVirtUserSecretAccessKey` (Task 2), and the role entry point (Task 4).
- Produces: the facts and provision-data keys `aws_nested_virt_access_key_id` and `aws_nested_virt_secret_access_key`. Task 7 propagates them.

- [ ] **Step 1: Create the branch**

Run: `cd agnosticd-v2 && git switch -c GPTEINFRA-18454-aws-nested-virtualization main`

- [ ] **Step 2: `post_infra.yml`.** Add two `set_fact` tasks named "Save NestedVirtUser credentials from stack outputs (…)". Mirror the Route53 pair exactly, with the same `cloudformation_already_deployed`, `cloudformation_out_final`, and `cloudformation_existing[project_tag]` conditions, and set the two `aws_nested_virt_*` facts. Then add one task:
```yaml
    - name: Save nested virtualization credentials to provision data
      when: aws_nested_virt_access_key_id is defined
      no_log: true
      agnosticd.core.agnosticd_user_info:
        data:
          aws_nested_virt_access_key_id: "{{ aws_nested_virt_access_key_id }}"
          aws_nested_virt_secret_access_key: "{{ aws_nested_virt_secret_access_key }}"
```
Don't add them to the "Debug Route53 variables" loop.

- [ ] **Step 3: `post_software.yml`.** Add the new play:
```yaml
- name: Step 005.2.1 - Enable nested virtualization on AWS workers
  hosts: localhost
  gather_facts: false
  become: false
  environment:
    KUBECONFIG: "{{ hostvars.localhost.output_dir }}/{{ config }}_{{ guid }}_kubeconfig"
  tasks:
  - name: Enable nested virtualization on worker instances
    when:
    - cloud_provider == 'aws'
    - openshift_aws_nested_virtualization_enable | default(false) | bool
    ansible.builtin.include_role:
      name: agnosticd.cloud_provider_aws.nested_virtualization
```

- [ ] **Step 4: Check syntax, lint, and secrets**

Run: `ansible-playbook --syntax-check ansible/configs/openshift-cluster/post_software.yml -i localhost, -e config=openshift-cluster -e guid=abc12` (add whatever extra vars syntax-check needs), then run the repo's lint target if it has one (check `tox.ini`, `Makefile`, `.github/workflows`), then `git diff | grep -n "secret"`.
Expected: the syntax check passes, lint is clean, and the secret appears only in `set_fact` values and the `no_log` task.

- [ ] **Step 5: Commit** (confirm the Jira key)

```bash
git add ansible/configs/openshift-cluster
git commit -m "[GPTEINFRA-18454] Wire AWS nested virt credentials and role into openshift-cluster"
```

- [ ] **Step 6: Push** only after Nate approves (branch, `origin`, commit).

---

### Task 7: AgnosticV `ocp-field-asset-aws`

**Files:**
- Modify: `agnosticv/agd_v2/ocp-field-asset-aws/common.yaml`: the scaling section (lines 64–80), the virt workload comment (lines 279–286), the `enable_base_virt` description (lines 411–416), and `propagate_provision_data` (line 703 onward).

**Interfaces:**
- Consumes: the provision-data keys from Task 6, and the role behavior from Task 5 (`worker_only` follows automatically).

- [ ] **Step 1: Confirm the base branch with Nate.** The recommended base is `GPTEINFRA-18402-ocp-fsc-showroom`, because the m7i switch only exists there. Then run `git switch -c GPTEINFRA-18454-aws-nested-virtualization <base>`.

- [ ] **Step 2: Edit `common.yaml`**
  - After `worker_instance_type`, add:
    `openshift_aws_nested_virtualization_enable: "{{ enable_base_virt | default(false) | bool and cluster_size != 'sno' }}"`
    Comment: turns on EC2 nested virtualization on the new workers, and `worker_only` follows.
  - Under the `openshift_base` component's `propagate_provision_data`, add:
    ```yaml
    - name: aws_nested_virt_access_key_id
      var: aws_nested_virt_access_key_id
    - name: aws_nested_virt_secret_access_key
      var: aws_nested_virt_secret_access_key
    ```
  - Update the line-71 and line-281 comments and the `enable_base_virt` description. It currently says "VMs use QEMU software emulation until the workers get nested virtualization", which is no longer true for multinode. Leave `ocp4_workload_openshift_virtualization_enabled_nested_kvm: true` alone.
  - Don't set `worker_only`.

- [ ] **Step 3: Validate**

Run: `agnosticv --validate` or the repo's CI lint (check `.github/workflows` and `Makefile`). At minimum, run `yamllint agd_v2/ocp-field-asset-aws/common.yaml`.
Expected: clean.

- [ ] **Step 4: Commit** (confirm the Jira key), then push only after approval.

```bash
git add agd_v2/ocp-field-asset-aws/common.yaml
git commit -m "[GPTEINFRA-18454] Enable AWS nested virt on OCP FSC items with base virt"
```

---

### Task 8: Live verification (with Nate)

Run these against real sandboxes. Use Nate's pool provision bundles (`~/Downloads/xrfct_*`) only to read API URL, token, and key *names*. **Never print their secret values.**

- [ ] **Step 1: Dev EE, only if Task 1 found the EE too old.** Build a local image `FROM quay.io/agnosticd/ee-multicloud:chained-2026-05-04` that upgrades only `awscli` (v2 installer, or `pip install -U awscli botocore` for v1) to at least the Task 1 minimums. Point the dev CI or local `agnosticd` runner at it. This image isn't committed.
- [ ] **Step 2: Point dev CIs at the branches.** Set `tag` (cloud_provider_aws collection) and `scm_ref` (agnosticd-v2) in the dev catalog items to `GPTEINFRA-18454-aws-nested-virtualization`. Nate decides whether that's a local `agnosticv` dev.yaml change or a one-off order.
- [ ] **Step 3: Direct path.** Provision `openshift-cluster` on AWS with `worker_instance_type: m7i.2xlarge`, `worker_instance_count: 2`, and `openshift_aws_nested_virtualization_enable: true`. Expected: the role log shows 2 targets stopped, modified, and started, and kvm-check pods succeed. Then `oc debug node/<worker> -- chroot /host ls -l /dev/kvm` shows a character device, and a test VM's `virt-launcher` domain XML shows `type='kvm'`.
- [ ] **Step 4: Pool + scale path.** Order a fresh multinode pool cluster, then `ocp-field-asset-aws` (dev) with `enable_base_virt: true` and 2+ workers. Expected: control-plane nodes no longer carry `node-role.kubernetes.io/worker`, `mastersSchedulable` is `false`, and VMs land only on `-compute` workers with KVM.
- [ ] **Step 5: Re-run the role** on the cluster from Step 3 or 4. Expected: `_nv_targets == []`, no stop calls, and kvm verification passes.
- [ ] **Step 6: Negative cases.** (a) Scale with `worker_instance_type: m6a.2xlarge` and the variable on: fails in pre-flight with the unsupported-type message, before any stop. (b) Use an old pool cluster without the keys: fails with the "created before nested virtualization support" message.
- [ ] **Step 7: Lifecycle.** Stop and start the provisioned service (Babylon lifecycle). Expected: `/dev/kvm` is still present on workers, and `DescribeInstances` still shows `NestedVirtualization: enabled`.
- [ ] **Step 8: Policy check with the scoped keys** (export them only in a subshell): `aws ec2 run-instances --dry-run ...` is denied (`UnauthorizedOperation`), `aws ec2 stop-instances --dry-run --instance-ids <control-plane id>` is denied, the same for the bastion, and `stop-instances --dry-run` on a worker returns `DryRunOperation`.
- [ ] **Step 9: Record the evidence.** Post a Jira milestone comment on GPTEINFRA-18454 with the test results, and update the description's DONE/TODO list. End the comment with `Produced by Nate, Generated by AI`. Then open PRs in all three repos (Summary and Changes sections, ending with the Claude Code attribution), linked from Jira. Ask Nate before transitioning the issue.

---

## Spec coverage check

| Spec item | Task |
|---|---|
| Scoped IAM user, gated, policy, outputs | 2 |
| Keys into facts and provision data | 6 |
| Propagation in the orderable CI | 7 |
| Role steps 1–8 (candidates, pre-flight, targets, stop, modify, start, Ready, `/dev/kvm`) | 3, 4 |
| Direct caller plus default | 6 |
| Scale caller, default, ordering before worker-only, `worker_only` default | 5 |
| AgnosticV variable, comments, nested_kvm left `true` | 7 |
| EE check and minimum versions | 1 |
| CoreCount/ThreadsPerCore, CpuOptions field, CLI vs boto3, `/dev/kvm` method, timeouts | Decisions section, 3, 4 |
| Role README and defaults comments | 3, 4 |
| Testing list (direct, pool, re-run, unsupported type, old pool, lifecycle, policy) | 8 |
