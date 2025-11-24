# Ceph SNO on RHEL 9
I am configuring ceph SNO on a RHEL 9 node which is running on KVM.
1. First install RHEL 9 OS in the VM.
2. On your base machine where KVM is running, Create 3 additional disks to be attached RHEL9 VM. I am using 100G storage for each disk.
```bash
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/osd1.qcow2 100G
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/osd2.qcow2 100G
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/osd3.qcow2 100G
```
3. Now attach these disks to the RHEL 9 VM. After attaching disks, it should be listed as shown below.
![Adding Disks](pictures/vm-disks.png)
5. After attaching the disks verify that disks are listed in VM. In below output vdb,vdc and vdd are listed.
```bash
 lsblk 
NAME                                                                                                  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0                                                                                                    11:0    1  1024M  0 rom  
vda                                                                                                   252:0    0   500G  0 disk 
├─vda1                                                                                                252:1    0     2G  0 part /boot
└─vda2                                                                                                252:2    0 496.3G  0 part 
  ├─rhel-root                                                                                         253:0    0 488.5G  0 lvm  /var/lib/containers/storage/overlay
  │                                                                                                                             /
  └─rhel-swap                                                                                         253:1    0   7.8G  0 lvm  [SWAP]
vdb                                                                                                   252:16   0   100G  0 disk 
vdc                                                                                                   252:32   0   100G  0 disk 
vdd                                                                                                   252:48   0   100G  0 disk 
```
5. Now set hostname on the system and make an entry in /etc/hosts
```bash
hostnamectl set-hostname cephsno
echo "192.168.1.5 cephsno.example.com cephsno" >> /etc/hosts
```
6. Register the system with Red Hat and enable below repository
```bash
subscription-manager register
subscription-manager repos --enable rhceph-5-tools-for-rhel-9-x86_64-rpms
```
7. Install required packages
```bash
dnf install podman cephadm ceph-common ceph-base -y
```
8. Fetch IP address of RHEL9 host, we will provide it during ceph bootstrap.
```
ip a
```
9. Now run the cephbootstrap command with Node's IP address and network.
```bash
cephadm bootstrap --cluster-network 192.168.1.0/24 --mon-ip 192.168.1.5 --registry-url registry.redhat.io --registry-username 'my-redhatuser --registry-password 'mypassword' --dashboard-password-noupdate --initial-dashboard-user admin --initial-dashboard-password redhat --allow-fqdn-hostname --single-host-defaults
Ceph Dashboard is now available at:

	     URL: https://cephsno.example.com:8443/
	    User: admin
	Password: redhat

Enabling client.admin keyring and conf on hosts with "admin" label
Enabling autotune for osd_memory_target
You can access the Ceph CLI as following in case of multi-cluster or non-default config:

	sudo /sbin/cephadm shell --fsid 0e2f3af0-c841-11f0-a6ce-525400fb6063 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

Or, if you are only running a single cluster on this host:

	sudo /sbin/cephadm shell 

Please consider enabling telemetry to help improve Ceph:

	ceph telemetry on

For more information see:

	https://docs.ceph.com/en/pacific/mgr/telemetry/

```
Run the Following Command
```bash
sudo /sbin/cephadm shell
```

10. Check cluster status
```bash
ceph -s
  cluster:
    id:     0e2f3af0-c841-11f0-a6ce-525400fb6063
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 2
 
  services:
    mon: 1 daemons, quorum cephsno (age 3m)
    mgr: cephsno.qnegpl(active, since 2m), standbys: cephsno.nzgcwq
    osd: 0 osds: 0 up, 0 in
```
11. Add additional disks, vdb, vdc, vdd to the ceph
```bash
ceph orch daemon add osd cephsno:/dev/vdb
Created osd(s) 0 on host 'cephsno'
ceph orch daemon add osd cephsno:/dev/vdc
Created osd(s) 1 on host 'cephsno'
ceph orch daemon add osd cephsno:/dev/vdd
```
12. Check status again, it will show 3 OSD.
```bash
ceph -s
  cluster:
    id:     0e2f3af0-c841-11f0-a6ce-525400fb6063
    health: HEALTH_OK
 
  services:
    mon: 1 daemons, quorum cephsno (age 6m)
    mgr: cephsno.qnegpl(active, since 3m), standbys: cephsno.nzgcwq
    osd: 3 osds: 3 up (since 1.21724s), 3 in (since 9s)
```
13. Also run lsblk and check the ceph initiated under the storage disk.
```bash
lsblk 
NAME                                                                                                  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0                                                                                                    11:0    1  1024M  0 rom  
vda                                                                                                   252:0    0   500G  0 disk 
├─vda1                                                                                                252:1    0     2G  0 part /boot
└─vda2                                                                                                252:2    0 496.3G  0 part 
  ├─rhel-root                                                                                         253:0    0 488.5G  0 lvm  /var/lib/containers/storage/overlay
  │                                                                                                                             /
  └─rhel-swap                                                                                         253:1    0   7.8G  0 lvm  [SWAP]
vdb                                                                                                   252:16   0   100G  0 disk 
└─ceph--36153621--96bd--44eb--a468--ec379d90b638-osd--block--e9ed4476--4b2c--444b--94a3--6c810ede1391 253:2    0   100G  0 lvm  
vdc                                                                                                   252:32   0   100G  0 disk 
└─ceph--0ba0fb77--ecff--41e7--93c2--013ec431acf6-osd--block--43ab203e--5f9a--4f9f--80a1--4ae8b0edb361 253:3    0   100G  0 lvm  
vdd                                                                                                   252:48   0   100G  0 disk 
└─ceph--fa5b9e09--f26b--4ea6--b968--c14e1aa37ae4-osd--block--2388d653--7056--4712--aba2--a171d3a1c3c0 253:4    0   100G  0 lvm  

```
If you are planning to use this ceph cluster with ODF by external method then create a rbd pool.
```bash
ceph osd pool create rbd 128 128
```
