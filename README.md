# Expand Viya 4 IaC Standard Storage (azure)

## Why 

By default, the Viya4 IaC ["standard" storage option](https://github.com/sassoftware/viya4-iac-azure/blob/main/docs/CONFIG-VARS.md#nfs-server-vm-only-when-storage_typestandard) gives you four 128 GB disks with a RAID 5 configuration (which means you’ll get around 380 GB of usable disk space for your SAS Viya Persistent Volumes).

It might seem to be a lot...but if you leave your environment running for a little while, it will quickly be consumed 😊 

To increase the space available in the IaC provisioned NFS server, we start by stopping the Viya environment, then followed the Azure documentation [steps](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/expand-disks) to de-allocate the NFS VM, then increase the size of the attached Azure disks, but the trickiest part was to find the proper command to release the RAID stack and rebuild it.

This repo contains the steps performed in a test environment to increase the NFS VM storage capacity.

**IMPORTANT WARNING: Please consider the code provided below as an example, not as an official documentation to do it.**

## Example

### Phase 1 : Expand the disk size on the NFS VM

```bash
# stop the Viya environment (use the viya-stop-all job)
kubectl -n viya4nfs create job sas-stop-now-`date +%s` --from cronjob/sas-stop-all
# install the azure client
# login to azure

# https://docs.microsoft.com/en-us/azure/virtual-machines/linux/expand-disks

# Deallocate the azure NFS VM
# az vm deallocate --resource-group myResourceGroup --name myVM
az vm deallocate --resource-group pg298-pdcesx02144-rg --name pg298-pdcesx02144-nfs-vm

# list disks
az disk list \
    --resource-group pg298-pdcesx02144-rg \
    --query '[*].{Name:name,Gb:diskSizeGb,Tier:accountType}' \
    --output table

Name                                                                Gb
------------------------------------------------------------------  ----
pg298-pdcesx02144-nfs-disk01                                        128
pg298-pdcesx02144-nfs-disk02                                        128
pg298-pdcesx02144-nfs-disk03                                        128
pg298-pdcesx02144-nfs-disk04                                        128
pg298-pdcesx02144-nfs-vm_OsDisk_1_a53d4d658da84911965a993a5ba74d5e  64

# increase disks sizes
az disk update \
    --resource-group pg298-pdcesx02144-rg \
    --name pg298-pdcesx02144-nfs-disk01 \
    --size-gb 256

az disk update \
    --resource-group pg298-pdcesx02144-rg \
    --name pg298-pdcesx02144-nfs-disk02 \
    --size-gb 256

az disk update \
    --resource-group pg298-pdcesx02144-rg \
    --name pg298-pdcesx02144-nfs-disk03 \
    --size-gb 256

az disk update \
    --resource-group pg298-pdcesx02144-rg \
    --name pg298-pdcesx02144-nfs-disk04 \
    --size-gb 256

# restart the NFS VM
az vm start --resource-group pg298-pdcesx02144-rg --name pg298-pdcesx02144-nfs-vm
```

### Resize the drives with LVM (Local Volume Manager)

```bash
# Repartition the drives as explained in https://docs.microsoft.com/en-us/azure/virtual-machines/linux/expand-disks#expand-a-disk-partition-and-filesystem
# Connect with ssh to the NFS VM
chmod 700 ~/.ssh/nfsvm_private_key
ssh -i ~/.ssh/nfsvm_private_key nfsuser@192.168.2.4

# list disks and partitions
lsblk
# unmount 
sudo umount -l /dev/mapper/data--vg01-data--lv01

# resize the physical volume structures
sudo pvresize /dev/sda
sudo pvresize /dev/sdb
sudo pvresize /dev/sdc
sudo pvresize /dev/sdd

# check that the new size is correctly identified by LVM
sudo vgs
sudo pvs

# resize your LVM volume
sudo lvresize -l +100%FREE /dev/data-vg01/data-lv01

# resize the filesystem itself 
sudo resize2fs /dev/data-vg01/data-lv01


# check the new size of the FS
df -h /export

Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/data--vg01-data--lv01  755G   19G  702G   3% /export

#restart the NFS Server
sudo exportfs -a
sudo systemctl restart nfs-kernel-server

# start the Viya environment (use the viya-stop-all job)
kubectl -n viya4nfs create job sas-start-now-`date +%s` --from cronjob/sas-start-all
```

NOTE : I had to reinstall the nfs provisionner

```bash
kubectl delete ns nfs-client 
```

and reinstall it with Helm.
