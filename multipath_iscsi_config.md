### How to setup multipath iscsi disk on one rbd block device(ceph rbd)

memo:

1. ceph rbd not support write rbd device from different client at same time, So the iscsi service based rbd must be active-enable
2. iscsi based rbd could use tgtd or lio.
3. Need set the write-cache off
4. tgtd use the librbd, which use the rbd cache, so we need set the rbd_cache off in ceph.conf
    [client]
    rbd_cache = false
5. lio use krbd for iscsi backing-store, the krbd use the page-cache, so we suggest use the tgtd
6. the lio maybe have a good support on active-active iscsi based on rbd, but it's only roadmap now.

How to:
1. Ceph envirment:
```console
[root@node1 ~]# ceph -s
    cluster 3ee07517-1fad-43e7-98ee-091148c65f67
     health HEALTH_WARN clock skew detected on mon.node2, mon.node3
     monmap e1: 3 mons at {node1=192.168.56.81:6789/0,node2=192.168.56.82:6789/0,node3=192.168.56.83:6789/0}, election epoch 32, quorum 0,1,2 node1,node2,node3
     osdmap e44: 3 osds: 3 up, 3 in
      pgmap v678: 448 pgs, 5 pools, 21340 kB data, 539 objects
            173 MB used, 13653 MB / 13826 MB avail
                 448 active+clean
[root@node1 ~]# ceph osd tree
# id	weight	type name	up/down	reweight
-1	3	root default
-2	1		host node1
0	1			osd.0	up	1
-3	1		host node2
1	1			osd.1	up	1
-4	1		host node3
2	1			osd.2	up	1
[root@node1 ~]# ceph osd lspools
0 data,1 metadata,2 rbd,3 mypool,4 rbd_pool,
```
2. create rbd named iscsi_rbd

```
[root@node1 ~]# rbd -p rbd_pool create iscsi_rbd --size 512
[root@node1 ~]# rbd -p rbd_pool ls -l
NAME              SIZE PARENT FMT PROT LOCK
iscsi_rbd         512M          1
```

3. install tgtd on iscsi server
	1. if you use rhel7, because there is nothing about tgtd package in rhel7, and the scsi-target-utils package in epel is not support rbd backstore type.
	   Hoho~~~, you can download the package in my baidu-disk:
	   http://pan.baidu.com/s/1kToZrkr
	2. on the iscsi server, you need install ceph-common package and copy the ceph.conf and admin key from one ceph node.

	3. you need config the rhel7 repo and ceph repo

```
[root@node1 tgtd]# ls
perl-Config-General-2.51-2.el7.noarch.rpm
scsi-target-utils-1.0.55-1.el7.x86_64.rpm
[root@node1 tgtd]# yum install -y ./perl-Config-General-2.51-2.el7.noarch.rpm
[root@node1 tgtd]# yum install -y scsi-target-utils-1.0.55-1.el7.x86_64.rpm
[root@node1 tgtd]# systemctl start tgtd
[root@node1 tgtd]# systemctl enable tgtd
```
    
    Now please make sure the tgtd support rbd backing store

```
[root@node1 tgtd]# tgtadm --lld iscsi --op show --mode system
```
    
    If you found the follow message in output, that mean your tgtd support rbd:)
        Backing stores:
         rbd (bsoflags sync:direct)
Note: you need do the same install work in another iscsi-server(node2)

4. configure  tgtd
    
    Modify the tgtd config file /etc/tgtd/targets.conf

```console
[root@node1 tgt]# vi /etc/tgt/conf.d/rbd-iscsi.conf
```

```xml
<target iqn.2015-06.rbd.example.com:iscsi-rbd1>
    driver iscsi
    bs-type rbd
    backing-store rbd_pool/iscsi_rbd  # pool_name/rbd_name
    write-cache off
    vendor_id Xiaobing Inc
    scsi_sn multipath-iscsi-rbd-1
    scsi_id IET bd4ab034
</target>
```

Note:

    1. the multipath program use scsi_id get the scsi id from iscsi disk, so we must specify the scsi_id configure in config file.
    2. scsi_id can be any string,such as "3231b32",but must unique in your cluser.
    3. specify the verdor_id and scsi_sn is the interest thing，little useage for multipath
    4. turn off the write-cache for data safe.

```console
[root@node1 tgt]# /etc/init.d/tgtd reload
[root@node1 tgt]# tgt-admin -s
...
        LUN: 1
            Type: disk
            SCSI ID: IET    bd4ab034
            SCSI SN: multipath-iscsi-rbd-1
            Size: 537 MB, Block size: 512
            ...
            Backing store type: rbd
            Backing store path: rbd_pool/iscsi_rbd
```
Now, config tgtd on node2, every thing is same expect the follow config:
    <target iqn.2015-06.rbd.example.com:iscsi-rbd2>
Because there can't have the same iqn target in one iscsi cluster.

5. config multipath in iscsi client
  1. install multipath package and iscsi package

[root@admin ~]# yum install -y iscsi-initiator-utils device-mapper-multipath

  2. scan iscsi disk
```console
[root@admin ~]# iscsiadm -m discovery -t sendtargets -p 192.168.56.81
192.168.56.81:3260,1 iqn.2015-06.rbd.example.com:iscsi-rbd1
[root@admin ~]# iscsiadm -m discovery -t sendtargets -p 192.168.56.82
192.168.56.82:3260,1 iqn.2015-06.rbd.example.com:iscsi-rbd2

[root@admin ~]# iscsiadm -m node -T iqn.2015-06.rbd.example.com:iscsi-rbd1 -l
[root@admin ~]# iscsiadm -m node -T iqn.2015-06.rbd.example.com:iscsi-rbd2 -l

[root@admin ~]# lsblk
...
sdb             8:16   0  512M  0 disk
sdc             8:32   0  512M  0 disk
...
```
    OK, we can found two disk with same size

  3. config multipath

  Create the multipath config file:
```console
[root@admin /]# vi /etc/multipath.conf
defaults {
    udev_dir        /dev
    selector        "round-robin 0"
    path_grouping_policy    failover
    uid_attribute       ID_SERIAL
    path_checker        readsector0
    rr_min_io       100
    max_fds         8192
    rr_weight       priorities
    failback        immediate
    no_path_retry       fail
    user_friendly_names yes
}
```
**Note： The most important configuration is "path_grouping_policy failover".**

```console
[root@admin /]# systemctl stop multipathd
[root@admin /]# systemctl start multipathd
[root@admin /]# multipath -ll
Jun 03 22:38:41 | multipath.conf +2, invalid keyword: udev_dir
Jun 03 22:38:41 | multipath.conf +3, invalid keyword: selector
mpathb (3600000000000000000000e00bd4ab034) dm-2 Xiaobing,VIRTUAL-DISK
size=512M features='0' hwhandler='0' wp=rw
|-+- policy='service-time 0' prio=1 status=active
| `- 3:0:0:1 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=1 status=enabled
  `- 4:0:0:1 sdc 8:32 active ready running
```

**Note: interesting thing, you could use scsi_id command get the iscsi disk id:**

```console
[root@admin /]# /lib/udev/scsi_id -u -g -p0x83 /dev/sdc
3600000000000000000000e00bd4ab034
```
If you config the "scsi_id" is "12345678", please try ,what is the output? :)

We could see one path is active and another path is enabled.

OK! That's all, good luck!


