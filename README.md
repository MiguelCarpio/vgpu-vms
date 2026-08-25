# vgpu-vms

Create MIG-backed vGPU VMs on an NVIDIA vGPU host — one vGPU (mdev) per VM.

The role discovers each GPU and how many VFs the chosen vGPU type
(`vgpu_vms_mdev_type`) can back per GPU (its `max_instance` — the total capacity
per GPU, i.e. the number of mdevs a GPU can host for that type), creates an mdev on
each free one of those VFs, clones a guest disk, attaches the mdev, and boots the
VM. By default it provisions the maximum the host supports for that type (one VM
per backable free VF, across all GPUs); pass `vgpu_vms_num` for an explicit
count. After boot it grows the guest root filesystem and confirms the vGPU is
visible from inside the guest.

This only provisions the VMs (mdev + libvirt guest) — it does **not** install the
guest vGPU driver or run any validation.

> **Note:** the role is designed for a **homogeneous MIG geometry** — every GPU
> carved into identical MIG instances backing a single vGPU type
> (`vgpu_vms_mdev_type`). How many VMs a GPU yields depends on the type's
> `max_instance`, not the geometry alone: on `47,47,47,47` (four identical `1g`
> instances) a full-instance type like `nvidia-1563` (DC-1-24C) gives four VMs
> per GPU, while a half-instance type like `nvidia-1560` (DC-1-12C) gives eight.
> The role fills every free VF the type can back.

## Requirements

Run **on the vGPU host**, as a user who can `sudo`.

- SR-IOV VFs enabled and the MIG geometry already applied on the host.
- `virt-install`, `qemu-img`, and `guestfs-tools` (`virt-customize`) installed.
- `mdevctl` installed (starts the mdevs).
- `nvidia-smi` available.

## Usage

`main.yml` lives inside the role directory and invokes the role by name, so point
Ansible at the parent directory with `ANSIBLE_ROLES_PATH`:

Provision the maximum (one 1-vGPU VM per free VF, across all GPUs):

```bash
ANSIBLE_ROLES_PATH=.. ansible-playbook -i localhost, -c local \
  main.yml -e vgpu_vms_mdev_type=nvidia-xxx
```

Provision a specific number. By default this fills each GPU's free VFs before
moving to the next, so a small count lands on the first GPU(s); pass
`-e vgpu_vms_round_robin=true` to instead spread one VM per GPU (vf0 across all
GPUs, then vf1). No effect when `vgpu_vms_num=max`:

```bash
ANSIBLE_ROLES_PATH=.. ansible-playbook -i localhost, -c local \
  main.yml -e vgpu_vms_mdev_type=nvidia-xxx -e vgpu_vms_num=4
```

`vgpu_vms_mdev_type` is required — set it to a type listed under a VF's
`mdev_supported_types/` (e.g. `nvidia-xxx`). All other variables are overridable
with `-e`; defaults are in [defaults/main.yaml](defaults/main.yaml).

By default each run first tears down **all** existing VMs and mdevs on the host,
so it starts from a clean host (see [Teardown](#teardown)). Set
`-e vgpu_vms_teardown=false` to skip it and provision alongside what's there.

### Running against a remote host

The guest NICs sit on the host's libvirt network (e.g. `192.168.122.0/24`),
which a remote controller can't reach. Point the inventory at the host and jump
through it to reach the guests with `vgpu_vms_ssh_proxy_host`:

```bash
ANSIBLE_ROLES_PATH=.. ansible-playbook -i 'gpu-host.example.com,' \
  main.yml -e vgpu_vms_mdev_type=nvidia-xxx \
  -e vgpu_vms_ssh_proxy_host=admin@gpu-host.example.com
```

For full control of the tunnel, set `vgpu_vms_ssh_proxy_command` instead (the
inner `ssh -W %h:%p ...` string). When running on the host itself, leave both
empty.

## Teardown

Every `main.yml` run tears down first by default: it destroys + undefines
**every** VM on the host (removing its disk) and stops **all** mdevs (freeing the
MIG instances and VFs), so it starts from a clean host. Set
`-e vgpu_vms_teardown=false` to skip it.

To also delete the shared base image during teardown:

```bash
ANSIBLE_ROLES_PATH=.. ansible-playbook -i localhost, -c local \
  main.yml -e vgpu_vms_mdev_type=nvidia-xxx -e vgpu_vms_teardown_base_image=true
```

The mdevs are **transient** (`mdevctl start`, not `define`) — a host reboot also
clears them. To remove one by hand:

```bash
mdevctl stop -u <uuid>
```
