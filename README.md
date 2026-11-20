# Ceph experiments

Lab notes for a Ceph Squid cluster running on one Linux laptop and two Ubuntu VMs. The experiments cover cluster health, native RADOS, CephFS, and S3 through RADOS Gateway.

## 20 November 2026 — Cluster baseline and monitor disk space

### Observed baseline

The cephadm container reported Ceph Squid `v19.2.3`. The laptop and both VMs each ran a monitor. All three monitors were in quorum. The manager and CephFS metadata server each had one standby.

| Storage host | OSDs | Observed state |
| --- | --- | --- |
| VM 1 | `osd.0`, `osd.1` | Both up and in |
| VM 2 | `osd.4`, `osd.5` | Both up and in |

All 305 placement groups were `active+clean` across four pools. Ceph reported approximately 80 GiB of raw OSD capacity available. The two VMs share a single physical laptop, so this lab has two logical OSD hosts in one physical failure domain.

### Monitor warning and resolution

The baseline showed `HEALTH_WARN` with `MON_DISK_LOW` on VM 2. Ceph reported 23% free on the filesystem containing that monitor’s database. Guest checks showed the database directory on `/`, the ext4 filesystem backed by the Ubuntu LVM root volume. Before expansion, `/` showed approximately 14 GiB total, 9.8 GiB used, and 3.2 GiB available. `/var` accounted for about 4.5 GiB.

After a reported 14 GB expansion of the VM system storage, `ceph health detail` returned **`HEALTH_OK`**. The final ext4 filesystem size was not captured in the supplied evidence.

The monitor warning and the cluster’s raw storage figure measure different filesystems or devices. The monitor warning concerns its local database filesystem. The raw OSD figure describes capacity across cluster storage devices. [Ceph Squid health checks](https://docs.ceph.com/en/squid/rados/operations/health-checks/#mon-disk-low)

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

### Next experiment — Native RADOS across the two VMs

**Status:** Planned. No object operation output has been recorded for this run.

1. Create a dedicated replicated test pool. Confirm the CRUSH rule selects different hosts and the test pool’s placement groups become `active+clean`. Use a pool-scoped CephX identity on both VMs.
2. Write a synthetic payload with `rados put` on VM 1. Retrieve it with `rados get` on VM 2 and check the bytes. Overwrite it on VM 2 and verify the replacement bytes on VM 1.
3. While the object exists, run `ceph osd map POOL OBJECT`. Record the placement group and acting OSD IDs. Use `ceph osd tree` to verify which host serves each copy.
4. Remove the test object with `rados rm`. Confirm a fresh `rados stat` on the other VM reports that the object is absent.

A verified cross-VM read demonstrates that both clients reach the same object. The acting set and OSD tree show where the object’s placement group is served. [RADOS CLI](https://docs.ceph.com/en/squid/man/8/rados/) · [Ceph architecture](https://docs.ceph.com/en/squid/architecture/)
