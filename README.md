# Quick and Dirty Ceph Cluster
This is a bare-bones guide on how to setup a Ceph cluster using `ceph-ansible`

This guide assumes:

- Ansible is installed on your local machine
- Eight Centos 7.2 nodes are provisioned and formatted with XFS
- You have ssh and sudo access to your nodes

### Ansible Setup
First, clone the ceph-ansible repo.

```
git clone https://github.com/ceph/ceph-ansible.git
cd ceph-ansible
git checkout a41eddf6f0d840580bc842fa59751defef10fa69
```

Next, add an `inventory` file to the base of the repo. Using the following contents as a guide, replace the example IPs with your own machine IPs.

```
[mons]
10.0.0.1
10.0.0.2
10.0.0.3

[osds]
10.0.0.4
10.0.0.5
10.0.0.6
10.0.0.7

[clients]
10.0.0.8

```
We need a minimum of 3 MONs (cluster state daemons) 4 OSDs (object storage daemons) and 1 client (to mount the storage).

Next, we need to copy the samples in place.

```
cp site.yml.sample site.yml
cp group_vars/all.sample group_vars/all
cp group_vars/mons.sample group_vars/mons
cp group_vars/osds.sample group_vars/osds
```

Now, we need to modify some variables in `group_vars/all`:

```
ceph_origin: 'upstream' 
ceph_stable: true

monitor_interface: eth0
public_network: 172.20.0.0/16

journal_size: 5120
```

And in `groups_vars/osd`:

```
osd_directory: true
osd_directories:
  - /var/lib/ceph/osd/mydir1
```

### Running ceph-ansible playbook
Once you've created your inventory file and modified your variables:

```
ansible-playbook site.yml -vvvv -i inventory -u centos
```
Once done, you should have a functioning Ceph cluster.

### Interacting with Ceph

For these next steps, we will need to SSH to a machine in your inventory with the `client` role (10.0.0.8 in the example inventory) and sudo to root.

#### Examine cluster status
Most of what you need to know you can see at a glance by using

```
ceph status
```
Abnormal statuses will be reported regarding MONs, OSDs, and other daemons/objects

#### Create an RBD volume
Due to kernel differences, it is safest to create a volume with minimal features. Here we are creating a 1GB volume with layering enabled.

```
rbd create test --size 1024  --image-feature=layering
```

#### Map an RBD volume
Once created, you can map the RBD volume to your client.

```
rbd map test
```
#### Format a volume
Once Mapped, format the volume with your filesystem of choice. Make sure you have the right device per the `rbd map` output

```
mkfs.ext4 /dev/rbd0
```

#### Mount the volume
```
mkdir /mnt/test
mount /dev/rbd0 /mnt/test
```

#### Delete the volume
```
umount /dev/rbd0
rbd unmap /dev/rbd0
rbd rm test
```