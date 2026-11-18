# blog-disk-expand-online

Companion code for the blog post
[Expand Disk Space on a VMware Linux VM Without Rebooting](https://blog.bsdserver.nl/expand-disk-space-on-a-vmware-linux-vm-without-rebooting/).

The post grows a VMDK in vSphere and then makes Linux use the space without a reboot. The
`disk_expand_online` role does steps 2 to 4 for every disk you list:

2. rescan the disk (`echo 1 > /sys/block/sdX/device/rescan`);
3. grow the partition with `growpart` (from `cloud-guest-utils` or `cloud-utils-growpart`);
4. for LVM: `pvresize`, then `lvextend -l +100%FREE` with `resize2fs` or `xfs_growfs`;
   for a filesystem directly on the partition: `resize2fs` or `xfs_growfs`.

The rescan works the same on any SCSI disk, so this repository uses Proxmox VE instead of vSphere for
step 1: Terraform creates an Ubuntu 24.04 VM with two data disks, and growing them is a change of
`data_disks` in `terraform.tfvars` followed by `terraform apply`. The `disk_expand_demo` role puts the two
layouts from the post on those disks: LVM on `sdb1` (ext4) and XFS directly on `sdc1`.

## Requirements

- Terraform >= 1.9 and a Proxmox VE API token (the bpg/proxmox provider, pinned in `terraform/versions.tf`).
- Ansible (ansible-core 2.17 or later; tested with 2.22) with the collections in `ansible/requirements.yml`.
- Ubuntu/Debian (tested on Ubuntu 24.04) or RHEL-family hosts.

## Quick start

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars   # endpoint, token, node, IP, SSH key
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

cd ../ansible
ansible-galaxy collection install -r requirements.yml -p collections
ansible-playbook site.yml      # first run: builds the practice partitions and filesystems

# Step 1: grow the disks at the hypervisor, e.g. data_disks = [20, 20]
cd ../terraform && terraform apply && cd ../ansible

ansible-playbook site.yml      # rescan, growpart, pvresize/lvextend, resize2fs/xfs_growfs
ansible-playbook verify.yml    # step 5 (df): every filesystem fills its disk
```

On vSphere or your own machine: grow the disk there, set `disk_expand_demo_enabled: false` and describe your
disks in `disk_expand_online_targets`, for example the post's `sda2` with LVM:

```yaml
disk_expand_online_targets:
  - {disk: sda, partition: 2, vg: ubuntu-vg, lv: ubuntu-lv, mount: /}
```

## Variables

| Variable | Default | Meaning |
| --- | --- | --- |
| `disk_expand_online_targets` | `sda` partition 2 with LVM | Disks to expand: `disk`, `partition` (omit for a whole-disk PV or filesystem), `vg` + `lv` for LVM, `mount` |
| `disk_expand_online_scan_new` | `false` | Also scan every SCSI host for newly added disks (`echo "- - -" > /sys/class/scsi_host/*/scan`) |
| `disk_expand_demo_enabled` | `true` in `group_vars` | Build the practice layouts on `sdb` and `sdc` first |

Running the role on a disk that has not been grown changes nothing (`growpart` reports `NOCHANGE`), so it is
safe to run again.

## Tested

Lab-tested on Ubuntu 24.04 (kernel 6.8) with ansible-core 2.22 and community.general 14.0.0. Two data
disks with LVM on partition 1 (ext4) and XFS on partition 1 were each grown by 1 GiB at the hypervisor
while mounted. The next run grew both partitions and filesystems without a reboot, the files on them were
intact, the run after that reported `changed=0`, and `verify.yml` passed. In the lab the disks were grown
through the Proxmox API rather than `terraform apply`. Terraform is checked with `fmt -check`, `validate`
and `tflint`; `terraform apply` was not run as part of these tests.

## License

MIT
