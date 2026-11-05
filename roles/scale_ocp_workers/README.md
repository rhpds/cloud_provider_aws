# scale_ocp_workers

Scales OpenShift worker nodes on AWS by managing MachineSets.

## Usage

Scale to 3 workers using existing instance type:

```yaml
- name: Scale OCP workers
  ansible.builtin.include_role:
    name: agnosticd.cloud_provider_aws.scale_ocp_workers
  vars:
    worker_instance_count: 3
```

Scale to 5 workers with a different instance type:

```yaml
- name: Scale OCP workers with custom instance type
  ansible.builtin.include_role:
    name: agnosticd.cloud_provider_aws.scale_ocp_workers
  vars:
    worker_instance_count: 5
    worker_instance_type: m5.2xlarge
    worker_machineset_suffix: compute
```

Scale to 3 workers and enable worker-only mode (compact/SNO clusters):

```yaml
- name: Scale OCP workers with worker-only mode
  ansible.builtin.include_role:
    name: agnosticd.cloud_provider_aws.scale_ocp_workers
  vars:
    worker_instance_count: 3
    worker_only: true
```

When `worker_only: true`, the role will:
1. Remove the `node-role.kubernetes.io/worker` label from control-plane nodes
2. Set `mastersSchedulable: false` on the Scheduler object (adds `NoSchedule` taint automatically)
3. Drain control-plane nodes to move existing workloads to the new workers (set `worker_drain_control_plane: false` to skip)

Scale to 3 workers with nested virtualization (OpenShift Virtualization with `/dev/kvm`):

```yaml
- name: Scale OCP workers with nested virtualization
  ansible.builtin.include_role:
    name: agnosticd.cloud_provider_aws.scale_ocp_workers
  vars:
    worker_instance_count: 3
    worker_instance_type: m7i.2xlarge
    openshift_aws_nested_virtualization_enable: true
```

The new workers get nested virtualization through the
[nested_virtualization](../nested_virtualization/README.md) role, and
`worker_only` turns on by default so VMs only run on those workers.
The cluster's provision data must include `aws_nested_virt_access_key_id`
and `aws_nested_virt_secret_access_key`.

## Variables

See [defaults/main.yml](defaults/main.yml) for all variables and their defaults.
