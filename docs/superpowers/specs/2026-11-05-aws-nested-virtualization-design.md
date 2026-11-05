# AWS nested virtualization for OpenShift workers

Jira: GPTEINFRA-18454

## Intent

Catalog items where OpenShift Virtualization is the star of the demo or lab
run on CNV today because CNV supports nested virtualization. On AWS, VMs on
non-metal workers run under QEMU software emulation, which works but is too
slow for those items.

AWS now supports nested virtualization on regular (non-metal) instances
([AWS docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/amazon-ec2-nested-virtualization.html)).
Once an instance has it turned on, the node exposes `/dev/kvm` and KubeVirt
uses hardware virtualization on its own. No operator change is needed.

The OpenShift Machine API cannot turn it on at launch:
`AWSMachineProviderConfig.cpuOptions` only has `confidentialCompute`
(openshift/api `machine/v1beta1/types_awsprovider.go`), and neither the
installer nor `machine-api-provider-aws` has a nested virtualization field.
So we turn it on after the Machine API launches the workers: stop the
instances, set the CPU option, start them again.

This is opt-in and not mainstream. It is for items that must run on AWS and
need OpenShift Virtualization to perform reliably. It has to be functional
and reliable, not perfect.

### How it looks to a CI author

- **`ocp-cluster-aws-pools` (pool CI): no change in behavior.** Still one
  or three control-plane nodes, no nested virtualization. The only
  difference is that its stack now creates the scoped IAM user and its keys
  appear in provision data, ready for the next-level item.
- **A new CI using the `openshift-cluster` config directly** (bypassing
  the pool) sets control plane and worker types and counts, plus
  `openshift_aws_nested_virtualization_enable: true`. The AgnosticD run
  turns on nested virtualization on the workers before any workload runs.
  VMs run only on workers.
- **A CI using `openshift-workloads` with `ocp-cluster-aws-pools` as its
  component** sets the same variable. `scale_ocp_workers` adds the workers
  and does the same thing. `worker_only` turns on automatically, so VMs run
  only on workers. A CI can set `worker_only: false` to opt out, and then
  workloads can also land on the pool's control-plane nodes (emulated for
  VMs).

Same variable, same result, two places it runs.

## Goals

- An opt-in variable, set in AgnosticV, that makes the provisioned cluster's
  workers support nested virtualization.
- Works in both provisioning paths:
  1. **Direct:** the `openshift-cluster` config provisions the cluster with
     workers.
  2. **Pool + scale:** an orderable CI uses a pool cluster
     (`agd-v2/ocp-cluster-aws-pools`) as a component, and the
     `openshift-workloads` config adds workers through
     `agnosticd.cloud_provider_aws.scale_ocp_workers`.
- Fast. All target workers are handled at the same time. Adding nested
  virtualization should cost a few minutes whether there are 2 workers or 10.
- Fails loudly. If any target worker ends up without `/dev/kvm`, the
  provision fails rather than shipping a slow cluster.

## Non-goals

- SNO pools. The single node is `m6a` (AMD, unsupported) and no workers are
  added. Changing the pool topology is out of scope.
- Keeping nested virtualization on machines created after provisioning (the
  user scaling MachineSets, autoscaler, machine replacement). If someone
  changes the MachineSets after we hand over the cluster, they have broken
  the contract.
- Changes to Babylon, the Sandbox API, or the governor.
- Changing the `ocp4_workload_openshift_virtualization` operator config.

## Anti-goals

- Credentials that can create EC2 instances or reach beyond this cluster's
  workers. Cluster AWS credentials were removed from clusters because they
  were abused. The scoped user must not reopen that.
- Rolling workers one at a time. That would add close to an hour for larger
  clusters.
- A cluster that silently falls back to emulation and is reported as
  successful.

## Background: the two variables

These stay separate.

| Variable | Where | What it does |
|---|---|---|
| `ocp4_workload_openshift_virtualization_enabled_nested_kvm` (existing, `core_workloads`) | OpenShift Virtualization workload | Sets `KVM_EMULATION=true` on the operator subscription, which turns on KubeVirt `useEmulation`: QEMU software emulation as a fallback when `/dev/kvm` is missing. Despite the name, it does not enable nested virtualization. |
| `openshift_aws_nested_virtualization_enable` (new) | AgnosticD config / cloud provider, next to `openshift_aws_scale_cluster` | Turns on the EC2 nested virtualization CPU option on the cluster's workers. AWS only. |

CIs leave the existing variable `true` (decided). Emulation stays as the
fallback for anything that lands on a node without `/dev/kvm`. Keeping VMs
on the fast workers is done with `worker_only` instead (see "Compact pool
clusters" below), because that is the existing AgnosticD way to force
workloads onto workers, and it sits in the same config layer as the new
variable. Asking CI authors to keep two variables in different workloads in
sync was rejected as too easy to get wrong. A later rename of the existing
variable (for example to `..._enable_software_emulation`) is a separate
cleanup.

## Design

### Overview

```
openshift-cluster (direct or pool)
  CloudFormation stack ─ creates scoped IAM user NestedVirtUser (always, on AWS)
  post_infra.yml       ─ reads its keys from stack outputs, saves them as facts
                         and in provision data (agnosticd_user_data)
  post_software.yml    ─ if openshift_aws_nested_virtualization_enable:
                         include agnosticd.cloud_provider_aws.nested_virtualization
                         (before workloads)

orderable CI (openshift-workloads, cloud_provider: none)
  propagate_provision_data ─ aws_nested_virt_* keys from the pool component
  pre_infra.yml → scale_ocp_workers ─ adds workers, waits for Ready, then
                         if openshift_aws_nested_virtualization_enable:
                         include agnosticd.cloud_provider_aws.nested_virtualization
```

Both paths call the same role with the same scoped credentials.

### Scoped IAM user (decided)

Created by default in every `openshift-cluster` AWS CloudFormation stack.
It must exist by default because the pool cannot know whether a later order
will want nested virtualization. It is deleted with the stack on normal
destroy; sandbox cleanup is a backstop.

`roles/template_generate/templates/cloud_template.yaml.j2` is shared by all
AWS configs, so the user is gated (my call): a new variable
`aws_nested_virt_iam_user_create`, default `false` in the
`template_generate` role, set `true` in
`agnosticd-v2/ansible/configs/openshift-cluster/aws/default_vars.yml`. RHEL
and other AWS configs get no such user. Inside the template it sits in the
existing `{% if secondary_stack is not defined %}` block next to
`Route53User`, with an access key and stack outputs.

Policy:

- **Allow** `ec2:StopInstances`, `ec2:StartInstances`,
  `ec2:ModifyInstanceCpuOptions` on `arn:aws:ec2:*:*:instance/*` only when
  - `aws:ResourceTag/guid` equals the cluster `guid`, and
  - `aws:ResourceTag/Name` is `StringLike` `*-worker-*`.

  The `guid` tag reaches every cluster instance through install-config
  `platform.aws.userTags` (`cloud_tags_final`, from
  `infra_cloud_tags` `default_cloud_tags`). MachineSets created by
  `scale_ocp_workers` copy the existing MachineSet tags, so new `-compute`
  machines carry it too. Machine API sets `Name` to the machine name
  (`<infra-id>-worker-<az>-...`), so control plane (`-master-`) and bastion
  instances are excluded.
- **Allow** `ec2:DescribeInstances`, `ec2:DescribeInstanceTypes` on `*`
  (read-only; these actions do not support resource-level restrictions).
- Nothing else. In particular no `ec2:RunInstances`, no `ec2:CreateTags`
  (so it cannot pull other instances into scope by tagging them), no IAM.

Known limit: `ModifyInstanceCpuOptions` cannot be restricted to only the
nested virtualization setting. A key holder could also change core count or
threads per core, but only on this cluster's stopped worker instances.

Exposure: the keys sit in the pool's provision data next to
`openshift_kubeadmin_password` and `openshift_cluster_admin_token`, which
already give the orderer full control of the cluster. The extra power is
stopping, starting, and changing CPU options on that cluster's own workers.

### Passing the keys

- `openshift-cluster/post_infra.yml` (agnosticd-v2): next to the existing
  "Save Route53User credentials" tasks, read the new outputs from
  `cloudformation_out_final` (new stack) or `cloudformation_existing`
  (existing stack) and set facts. Save them with
  `agnosticd.core.agnosticd_user_data` as:
  - `aws_nested_virt_access_key_id`
  - `aws_nested_virt_secret_access_key`
- Orderable CI (AgnosticV): add both to the pool component's
  `propagate_provision_data`, as `autoshift-lab-aws` already does with
  `aws_access_key_id`.
- Direct path: the role reads the same names from the facts set in
  `post_infra`.

Nothing else is propagated (my call). The role reads the pool's region and
instance IDs from the cluster itself (see the role steps), because in the
`openshift-workloads` path `guid` is the orderable CI's own guid, not the
pool's, and `ocp-field-asset-aws` does not propagate `aws_region`.

### The role: `agnosticd.cloud_provider_aws.nested_virtualization` (decided outline)

New role in this collection, next to `scale_ocp_workers`. Runs on
localhost. Inputs: `aws_nested_virt_access_key_id`,
`aws_nested_virt_secret_access_key`, and the cluster API access already used
by the caller. It does not take `guid` or `aws_region` (my call): both are
read from the cluster so the role behaves the same in both paths.

1. **Find candidate machines from the cluster.** List Machines in
   `openshift-machine-api` with
   `machine.openshift.io/cluster-api-machine-role: worker` and phase
   `Running`. Take each one's instance ID
   (`status.providerStatus.instanceId`), instance type and region
   (`spec.providerSpec.value.instanceType` / `.placement.region`), and its
   node (`status.nodeRef`). Control-plane Machines have role `master` and
   are never candidates, even on compact clusters where masters also carry
   the worker node role.
2. **Pre-flight.** Fail with a clear message if there are no worker
   Machines (the variable is on but the cluster has no workers), if the
   keys are missing (for example a pool cluster created before this
   change), or if a candidate's
   instance type does not list `nested-virtualization` in
   `ProcessorInfo.SupportedFeatures` (`DescribeInstanceTypes`).
3. **Pick targets.** `DescribeInstances` on the candidate IDs. Keep the
   ones whose `CpuOptions` do not already show nested virtualization
   enabled. This makes the role idempotent and, in the scale path, limits
   it to new workers without knowing which MachineSets were scaled. If
   there are no targets, go to the verify step. The IAM policy still
   enforces the `guid`/`Name` scope on the AWS side.
4. **Stop all targets** in one call and wait until all are `stopped`.
   No cordon or drain: this runs before any workload is deployed.
5. **Modify each** with nested virtualization enabled. These are quick API
   calls; run them back to back or async.
6. **Start all targets** in one call. Retry on `InsufficientInstanceCapacity`
   for a bounded time; a stop/start can land on a new host.
7. **Wait for the matching nodes to be Ready.**
8. **Verify `/dev/kvm` on every candidate worker Machine's node** (step 1)
   and fail if any is missing. Control-plane nodes are not checked.

Machine API behavior: `machine-api-provider-aws` counts `stopped` and
`stopping` instances as existing (`existingInstanceStates()` in
`pkg/actuators/machine/utils.go`), so it does not replace workers while they
are off. No component in these repos creates a MachineHealthCheck.

### Callers

- **Direct:** `agnosticd-v2/ansible/configs/openshift-cluster/post_software.yml`,
  new play between Step 005.2 (API reachable) and Step 005.3 (workloads),
  `when: cloud_provider == 'aws' and openshift_aws_nested_virtualization_enable | bool`.
  Default `false` in `openshift-cluster/aws/default_vars.yml`.
- **Pool + scale:** `scale_ocp_workers/tasks/main.yml`, after "Wait for
  worker nodes to be Ready", `when: openshift_aws_nested_virtualization_enable | bool`.
  Default `false` in `scale_ocp_workers/defaults/main.yml`.

### Compact pool clusters (decided)

Pool clusters are compact: three `m6a` control-plane nodes that are also
schedulable workers. Without a change, VMs could land on them and run
emulated. In the scale path, nested virtualization turns on `worker_only`,
which `scale_ocp_workers` already implements
(`configure_worker_only.yml`: removes the worker role from control-plane
nodes, sets `mastersSchedulable: false`, drains them by default).

So nobody has to remember it, `scale_ocp_workers/defaults/main.yml` makes
the default follow the new variable (my call on the mechanism):

```yaml
worker_only: "{{ openshift_aws_nested_virtualization_enable | bool }}"
```

A CI can still set `worker_only` explicitly. Ordering in
`scale_ocp_workers/tasks/main.yml`: scale, wait for Ready, nested
virtualization role, then worker-only (so the control-plane drain moves
pods onto workers that already have their final CPU options and are not
about to be stopped).

The direct `openshift-cluster` path gets the same outcome without extra
code: when install-config has compute replicas, the installer sets
`mastersSchedulable: false` and control-plane nodes do not get the worker
role. That path has no `worker_only` variable today and does not need one
for this.

### AgnosticV (orderable CI)

For `agd_v2/ocp-field-asset-aws` (branch
`GPTEINFRA-18402-ocp-fsc-showroom` already switches workers to
`m7i.2xlarge`/`m7i.4xlarge` when `enable_base_virt` is set):

- `openshift_aws_nested_virtualization_enable: "{{ enable_base_virt | default(false) | bool and cluster_size != 'sno' }}"`
- Add the two `aws_nested_virt_*` keys to `propagate_provision_data`.
- Leave `ocp4_workload_openshift_virtualization_enabled_nested_kvm: true`.
- `worker_only` follows automatically; no CI setting needed.

The pool CI needs no change: the user is created by default.

### Rollout (decided)

Pool clusters created before this change have no scoped user, and virt
orders that claim one fail pre-flight. That is acceptable: pools turn over
in a couple of days, and testing uses fresh pool clusters. The code and the
AgnosticV change can ship together.

## Constraints

- Supported instance families: M7i, M7i-flex, M8i, M8id, M8i-flex, C7i,
  C7i-flex, C8i, C8id, C8i-flex, R7i, R7iz, R8i, R8id, R8i-flex, X8i, I7i,
  I7ie. All Intel. The `openshift-cluster` AWS default worker family is
  `m7a` (AMD), so opting in also means choosing a supported type.
- Only pool clusters created after this change have the scoped user.
- Execution environment: `ee-multicloud` must have an `awscli` or
  `botocore` new enough to know the nested virtualization CPU option. The
  `amazon.aws` `ec2_instance` module's `cpu_options` does not appear to
  support it, so expect the AWS CLI or boto3. If the current image is too
  old, the fix is a new EE image (the EE maintainer has agreed to update
  it). The role does not install packages at runtime. Development can use
  a locally built EE until the official image ships.
- No credentials or environment-specific values in committed files.

## Left to the builder

- **EE check (first task).** Pull `quay.io/agnosticd/ee-multicloud`
  (currently `chained-2026-05-04` in the AWS CIs) and check
  `aws ec2 modify-instance-cpu-options help` for `--nested-virtualization`
  and the botocore model for `NestedVirtualization`. Report the minimum
  versions needed so the EE maintainer can update the image.
- Whether `ModifyInstanceCpuOptions` requires `CoreCount` and
  `ThreadsPerCore` alongside `NestedVirtualization` (the AWS doc example
  passes them). If so, read the current values from `DescribeInstances`.
- How nested virtualization shows up in `DescribeInstances` `CpuOptions`,
  for the idempotency filter in step 2.
- AWS CLI vs boto3 vs a small module.
- How to check `/dev/kvm` without `oc` on the controller. One option: a
  short-lived pod per node (`nodeName` set) with a `hostPath` volume of
  type `CharDevice` at `/dev/kvm`, using the cluster's
  `openshift/cli` image stream image; it only starts if the device exists.
- Timeouts and retry counts, following `scale_ocp_workers` conventions.
- Role README (short: purpose and usage) and variable comments in
  `defaults/main.yml`.

## Testing

- Direct: `openshift-cluster` on AWS with 2+ `m7i` workers and the new
  variable on. Confirm `/dev/kvm` on workers and a VM booting with KVM
  (not emulation).
- Pool + scale: a new pool cluster (multinode), then an orderable CI with
  `enable_base_virt` and 2+ workers. Confirm control-plane nodes lost the
  worker role and VMs schedule only on the new workers.
- Re-run the role on an already enabled cluster: no stops.
- Unsupported instance type: fails in pre-flight.
- Pool cluster without the scoped user: fails in pre-flight with a clear
  message.
- Stop/start lifecycle on a provisioned cluster keeps nested virtualization.
- Policy check with the scoped keys: `RunInstances`, stopping a control
  plane node, and stopping the bastion are all denied.
