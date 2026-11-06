# ocp_nested_virtualization

Turns on EC2 nested virtualization on an OpenShift cluster's AWS worker
instances, so OpenShift Virtualization uses `/dev/kvm` instead of software
emulation. The Machine API cannot set this at launch, so the role stops the
workers, sets the CPU option, and starts them again. All workers are handled
at once. The role fails if any worker node ends up without `/dev/kvm`.

It is called by:

- `openshift-cluster` (`post_software.yml`) when
  `openshift_aws_nested_virtualization_enable` is true
- `scale_ocp_workers` after new workers are Ready, with the same variable

## Usage

```yaml
- name: Enable nested virtualization on worker instances
  ansible.builtin.include_role:
    name: agnosticd.cloud_provider_aws.ocp_nested_virtualization
```

- `aws_nested_virt_access_key_id` and `aws_nested_virt_secret_access_key`
  must be set (in the direct path by `post_infra`, in the scale path from
  provision data). The role writes them to a temporary 0600 credentials
  file for the AWS CLI and removes it afterwards.

- The credentials belong to the scoped `NestedVirtUser` IAM user created by
  the cluster's CloudFormation stack (`aws_nested_virt_iam_user_create`). It
  can only stop, start, and change CPU options on this cluster's worker
  instances.
- Cluster API access comes from the caller: `module_defaults` for
  `group/k8s`, or `KUBECONFIG`.
- The region and instance IDs are read from the worker Machines.
- Workers need a supported Intel instance type (for example `m7i`). The
  role checks this before stopping anything.
- Already enabled instances are skipped, so re-running the role is safe.
  Instances left stopped by an earlier failed run are started again.
- Machines created later (scaling MachineSets, autoscaler, replacements) do
  not get nested virtualization.

## Pre-flight check

`preflight_config.yml` is an order-time check that needs no cluster access.
It runs in the `openshift-cluster` config's `pre_infra` phase (before the
CloudFormation stack is created) and at the top of `scale_ocp_workers`
(before new machines are created), so a bad order fails in minutes instead
of after the cluster is deployed. It checks that `worker_instance_count` is
at least 1 and that the worker instance type has the AWS
`nested-virtualization` feature (`aws ec2 describe-instance-types`), and
fails with a message that says how to fix the order, including the QEMU
emulation option for single-node or workerless clusters.

```yaml
- name: Check nested virtualization prerequisites
  ansible.builtin.include_role:
    name: agnosticd.cloud_provider_aws.ocp_nested_virtualization
    tasks_from: preflight_config.yml
```

It uses the scoped `aws_nested_virt_*` credentials when they are defined
(scale path) and the deployer credentials otherwise (pre_infra path).

See [defaults/main.yml](defaults/main.yml) for timeouts.
