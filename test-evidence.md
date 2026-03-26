# PR #17226 + #17228 Test Evidence

## Cluster Info

- Rook version: v1.19.2 (Helm chart)
- Test image: `ghcr.io/ormandj/rook-ceph:test-osd-fixes` (based on v1.19.2 + both PR branches)
- Node: quasar (single-node, amd64, Talos Linux)
- 12 OSDs: 4x NVMe (Micron 7450 7.6TB), 8x SATA HDD (WDC WUH722422ALE6L4 22TB)
- Replication: 3x, min_size 2

---

## 1. Before: All OSDs using kernel device names

All OSD deployments use unstable kernel names. `ROOK_OSD_CRUSH_DEVICE_CLASS` is absent.

```
osd-0:  ROOK_BLOCK_PATH=/dev/sdd      (no ROOK_OSD_CRUSH_DEVICE_CLASS)
osd-1:  ROOK_BLOCK_PATH=/dev/sdg      (no ROOK_OSD_CRUSH_DEVICE_CLASS)
osd-2:  ROOK_BLOCK_PATH=/dev/sdh      (no ROOK_OSD_CRUSH_DEVICE_CLASS)
osd-3:  ROOK_BLOCK_PATH=/dev/sdi      (no ROOK_OSD_CRUSH_DEVICE_CLASS)
osd-4:  ROOK_BLOCK_PATH=/dev/nvme0n1  (no ROOK_OSD_CRUSH_DEVICE_CLASS)
osd-5:  ROOK_BLOCK_PATH=/dev/nvme2n1  (no ROOK_OSD_CRUSH_DEVICE_CLASS)
osd-6:  ROOK_BLOCK_PATH=/dev/sdb      (no ROOK_OSD_CRUSH_DEVICE_CLASS)
osd-7:  ROOK_BLOCK_PATH=/dev/sdc      (no ROOK_OSD_CRUSH_DEVICE_CLASS)
osd-8:  ROOK_BLOCK_PATH=/dev/sde      (no ROOK_OSD_CRUSH_DEVICE_CLASS)
osd-9:  ROOK_BLOCK_PATH=/dev/sdf      (no ROOK_OSD_CRUSH_DEVICE_CLASS)
osd-10: ROOK_BLOCK_PATH=/dev/nvme3n1  (no ROOK_OSD_CRUSH_DEVICE_CLASS)
osd-11: ROOK_BLOCK_PATH=/dev/nvme1n1  (no ROOK_OSD_CRUSH_DEVICE_CLASS)
```

---

## 2. After operator upgrade (before OSD reprovision)

Operator upgraded to test image. Operator reconciled all 12 OSD deployments.

### PR #17228: ROOK_OSD_CRUSH_DEVICE_CLASS now set on all OSD pods

```
osd-0:  ROOK_OSD_CRUSH_DEVICE_CLASS=hdd   ROOK_OSD_DEVICE_CLASS=hdd
osd-4:  ROOK_OSD_CRUSH_DEVICE_CLASS=nvme  ROOK_OSD_DEVICE_CLASS=nvme
osd-10: ROOK_OSD_CRUSH_DEVICE_CLASS=nvme  ROOK_OSD_DEVICE_CLASS=nvme
(etc. for all 12 OSDs -- verified)
```

### PR #17226: Prepare job resolves all paths

Prepare job log shows all 12 devices resolved to persistent paths:
```
resolved "/dev/sdd"     -> "/dev/disk/by-id/wwn-0x5000cca418d1d23e"
resolved "/dev/sdg"     -> "/dev/disk/by-id/wwn-0x5000cca415c546f1"
resolved "/dev/sdh"     -> "/dev/disk/by-id/wwn-0x5000cca2facfbd80"
resolved "/dev/sdi"     -> "/dev/disk/by-id/wwn-0x5000cca43fc31f75"
resolved "/dev/sdb"     -> "/dev/disk/by-id/wwn-0x5000cca418d314ed"
resolved "/dev/sdc"     -> "/dev/disk/by-id/wwn-0x5000cca2fad7b86d"
resolved "/dev/sde"     -> "/dev/disk/by-id/wwn-0x5000cca415cf2868"
resolved "/dev/sdf"     -> "/dev/disk/by-id/wwn-0x5000cca415c97027"
resolved "/dev/nvme0n1" -> "/dev/disk/by-id/nvme-eui.000000000000000100a0752551112b3c"
resolved "/dev/nvme1n1" -> "/dev/disk/by-id/nvme-eui.000000000000000100a07525511483b9"
resolved "/dev/nvme2n1" -> "/dev/disk/by-id/nvme-eui.000000000000000100a0752551112b71"
resolved "/dev/nvme3n1" -> "/dev/disk/by-id/nvme-eui.000000000000000100a0752550cfa8af"
```

SATA HDDs -> `wwn-*` (World Wide Name, IEEE-assigned hardware ID)
NVMe SSDs -> `nvme-eui.*` (EUI-64, IEEE-assigned hardware ID)

Existing OSD deployments keep their kernel names until reprovisioned (by design -- safe for running OSDs).

---

## 3. NVMe OSD purge/reprovision test (OSD 10)

### Before purge
```
ROOK_BLOCK_PATH=/dev/nvme3n1
ROOK_OSD_DEVICE_CLASS=nvme
ROOK_OSD_CRUSH_DEVICE_CLASS=nvme
CRUSH tree: osd.10  nvme  6.98630  up
```

### Purge steps
```
kubectl -n rook-ceph scale deployment rook-ceph-osd-10 --replicas=0
ceph osd purge 10 --yes-i-really-mean-it
kubectl -n rook-ceph delete deployment rook-ceph-osd-10
ceph-volume lvm zap /dev/nvme3n1 --destroy
(operator restart to trigger reconciliation)
```

### ceph-volume raw prepare command (from prepare job logs)
```
ceph-volume raw prepare --bluestore --data /dev/nvme3n1 --crush-device-class nvme
```

### After reprovision
```
ROOK_BLOCK_PATH=/dev/disk/by-id/nvme-eui.000000000000000100a0752550cfa8af
ROOK_OSD_DEVICE_CLASS=nvme
ROOK_OSD_CRUSH_DEVICE_CLASS=nvme
CRUSH tree: osd.10  nvme  6.98630  up
```

OSD 10 came up healthy, HEALTH_OK.

**Result: `/dev/nvme3n1` -> `/dev/disk/by-id/nvme-eui.000000000000000100a0752550cfa8af`**

---

## 4. SATA OSD purge/reprovision test (OSD 0)

### Before purge
```
ROOK_BLOCK_PATH=/dev/sdd
ROOK_OSD_DEVICE_CLASS=hdd
ROOK_OSD_CRUSH_DEVICE_CLASS=hdd
CRUSH tree: osd.0  hdd  20.00980  up
```

### Purge steps
```
kubectl -n rook-ceph scale deployment rook-ceph-osd-0 --replicas=0
ceph osd purge 0 --yes-i-really-mean-it
kubectl -n rook-ceph delete deployment rook-ceph-osd-0
ceph-volume lvm zap /dev/sdd --destroy
(operator restart to trigger reconciliation)
```

### After reprovision
```
ROOK_BLOCK_PATH=/dev/disk/by-id/wwn-0x5000cca418d1d23e
ROOK_OSD_DEVICE_CLASS=hdd
ROOK_OSD_CRUSH_DEVICE_CLASS=hdd
CRUSH tree: osd.0  hdd  20.00980  up
```

**Result: `/dev/sdd` -> `/dev/disk/by-id/wwn-0x5000cca418d1d23e`**

---

## 5. Final cluster state

All 12 OSDs up, HEALTH_OK. Two reprovisioned OSDs now use persistent paths:

```
ID  CLASS  WEIGHT     TYPE NAME        STATUS
 0    hdd   20.00980  osd.0        up    <- /dev/disk/by-id/wwn-0x5000cca418d1d23e (NEW)
 1    hdd   20.00980  osd.1        up    <- /dev/sdg (original)
 2    hdd   20.00980  osd.2        up    <- /dev/sdh (original)
 3    hdd   20.00980  osd.3        up    <- /dev/sdi (original)
 4   nvme    6.98630  osd.4        up    <- /dev/nvme0n1 (original)
 5   nvme    6.98630  osd.5        up    <- /dev/nvme2n1 (original)
 6    hdd   20.00980  osd.6        up    <- /dev/sdb (original)
 7    hdd   20.00980  osd.7        up    <- /dev/sdc (original)
 8    hdd   20.00980  osd.8        up    <- /dev/sde (original)
 9    hdd   20.00980  osd.9        up    <- /dev/sdf (original)
10   nvme    6.98630  osd.10       up    <- /dev/disk/by-id/nvme-eui.000000000000000100a0752550cfa8af (NEW)
11   nvme    6.98630  osd.11       up    <- /dev/nvme1n1 (original)
```

---

## Summary

| PR | Fix | Verified |
|----|-----|----------|
| #17226 | Persistent /dev/disk/by-id/ paths for raw mode OSDs | Yes -- both NVMe (eui) and SATA (wwn) paths resolved correctly on reprovision |
| #17228 | ROOK_OSD_CRUSH_DEVICE_CLASS set on OSD pods | Yes -- env var present on all 12 OSD deployments after operator reconciliation |
