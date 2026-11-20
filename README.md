# Ceph experiments

Lab notes for a Ceph Squid cluster running on one Linux laptop and two Ubuntu VMs. The experiments cover cluster health, native RADOS, CephFS, and S3 through RADOS Gateway.

## 20 November 2026 — Cluster baseline and monitor disk space

### Observed baseline

The cephadm container reported Ceph Squid `v19.2.3`. The laptop and both VMs each ran a monitor. All three monitors were in quorum. The manager and CephFS metadata server each had one standby.

Each Ubuntu VM has three virtual disks. One holds the operating system and two are dedicated to Ceph, with one OSD per Ceph disk.

| Storage host | OS disks | Dedicated Ceph disks | OSDs observed up and in |
| --- | ---: | ---: | --- |
| VM 1 | 1 | 2 | `osd.0`, `osd.1` |
| VM 2 | 1 | 2 | `osd.4`, `osd.5` |

Disk layout confirmed by the lab operator. OSD state comes from the cluster baseline.

All 305 placement groups were `active+clean` across four pools. Ceph reported approximately 80 GiB of raw OSD capacity available. The two VMs share a single physical laptop, so this lab has two logical OSD hosts in one physical failure domain.

### Monitor warning and resolution

The baseline showed `HEALTH_WARN` with `MON_DISK_LOW` on VM 2. Ceph reported 23% free on the filesystem containing that monitor’s database. Guest checks showed the database directory on `/`, the ext4 filesystem backed by the Ubuntu LVM root volume. Before expansion, `/` showed approximately 14 GiB total, 9.8 GiB used, and 3.2 GiB available. `/var` accounted for about 4.5 GiB.

After a reported 14 GB expansion of the VM system storage, `ceph health detail` returned **`HEALTH_OK`**. The final ext4 filesystem size was not captured in the supplied evidence.

The monitor database on VM 2 uses the OS disk. The raw OSD capacity describes the dedicated Ceph disks across both VMs. [Ceph Squid health checks](https://docs.ceph.com/en/squid/rados/operations/health-checks/#mon-disk-low)

Commands used to capture the baseline on the administration host:

```bash
sudo cephadm shell -- ceph -s
sudo cephadm shell -- ceph health detail
sudo cephadm shell -- ceph osd tree
sudo cephadm shell -- ceph orch host ls
```

On the affected VM, use the local monitor directory to identify the filesystem that backs it:

```bash
# Set this to the monitor data directory on the affected VM.
mon_dir=/var/lib/ceph/CLUSTER_FSID/mon.MONITOR_ID
sudo df -hT "$mon_dir"
sudo findmnt -T "$mon_dir"
```

Source for the observed results: terminal output from the lab run on 20 November 2026.

### Native RADOS across the two VMs

**Run 02:** `cross-vm-run02` in `thesis-rados-lab`. The cross-VM read and overwrite checks passed. VM 2 verified version 1 with `cmp`. VM 1 then verified the exact version 2 bytes written by VM 2. Deletion and the subsequent lookup remain pending.

At OSD map epoch `263`, the object mapped to placement group `5.9`. The up and acting sets both listed `[5,1]`. `osd.5` on VM 2 was the primary and `osd.1` on VM 1 was the replica.

```text
osdmap e263 pool 'thesis-rados-lab' (5) object 'cross-vm-run02' -> pg 5.dcadc69 (5.9) -> up ([5,1], p5) acting ([5,1], p5)
```

**Previous attempt:** VM 2 retrieved the version 1 payload written on VM 1 from `cross-vm-run01`. VM 2 then ran `rados put` for version 2 and the terminal showed no write error. The return read, placement and deletion remain unverified for that attempt.

An earlier read from the administration host stopped because the test configuration file was unavailable inside the Cephadm container. The read from VM 2 succeeded with the local lab directory mounted into the container.

1. Reuse the dedicated replicated test pool and pool-scoped CephX identity on both VMs. Confirm the CRUSH rule selects different hosts and the test pool’s placement groups are `active+clean`.
2. Write a synthetic payload with `rados put` on VM 1. Retrieve it with `rados get` on VM 2 and check the bytes. Overwrite it on VM 2 and verify the replacement bytes on VM 1.
3. While the object exists, run `ceph osd map POOL OBJECT`. Record the placement group and acting OSD IDs. Use `ceph osd tree` to verify which host serves each copy.
4. Remove the test object with `rados rm`. Confirm a fresh `rados stat` on the other VM reports that the object is absent.

A verified cross-VM read demonstrates that both clients reach the same object. The acting set and OSD tree show where the object’s placement group is served. [RADOS CLI](https://docs.ceph.com/en/squid/man/8/rados/) · [Ceph architecture](https://docs.ceph.com/en/squid/architecture/)
