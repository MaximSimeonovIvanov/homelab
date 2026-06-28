# Hardware

## Machine: Dell OptiPlex 7040 SFF

| Component    | Detail                                      |
|--------------|---------------------------------------------|
| Form Factor  | Small Form Factor (SFF)                     |
| CPU          | Intel Core i5-6500 (4 cores / 4 threads)    |
| RAM          | 16 GB DDR4                                  |
| Boot Drive   | 128 GB SSD (OS + Docker config)             |
| Data Drive   | Not yet installed — HDD to be added         |
| GPU          | None (integrated Intel HD 530 only)         |

---

## CPU Notes

The i5-6500 is a 6th generation Skylake chip. Relevant capabilities for this homelab:

- **Intel Quick Sync** — hardware-accelerated video transcoding. Used by Nextcloud Memories (`go-vod`) and potentially Immich for video previews.
- **No AVX-512** — this limits some ML inference optimisations. Immich facial recognition will run on CPU via ONNX Runtime and will be noticeably slow (expect minutes per batch, not seconds). This is functional but not fast.
- **No dedicated GPU** — all AI/ML workloads (Immich machine learning container) run on CPU.

---

## Storage Layout Plan

| Path               | Drive    | Purpose                              |
|--------------------|----------|--------------------------------------|
| `/`                | SSD      | OS, system files                     |
| `/opt/docker/`     | SSD      | Docker Compose files, configs, `.env`|
| `/var/lib/docker/` | SSD      | Docker image layers and metadata     |
| `/mnt/data/`       | HDD      | All persistent data volumes          |

Data volumes (Nextcloud files, Immich photos, databases) will mount from `/mnt/data/` once the HDD is installed. Until then, they live on the SSD temporarily.

---

## Drive Bay Availability (SFF)

The OptiPlex 7040 SFF has limited internal expansion:

- 1× 3.5" HDD bay
- 1× 2.5" drive slot (currently occupied by the boot SSD)

This means a maximum of one additional data drive internally. A 4 TB 3.5" HDD is the recommended first purchase.

---

## Drive Purchasing Decision

**Recommendation: 1× 4 TB HDD** (not 2× 2 TB with RAID/mergerfs)

Reasons:
- Simpler setup — focus on services first, storage redundancy later
- SnapRAID + mergerfs adds real complexity for a first build
- True backup strategy (3-2-1 rule) matters more than local redundancy at this scale

Revisit RAID/mergerfs if a second drive is added in future.

### Storage allocation (LVM) — [verified June 28]
- `ubuntu-vg` is **fully allocated**: `vgs` shows `VFree 0`, `pvs` shows `PFree 0`.
  Root (`ubuntu-lv`, ext4, mounted `/`) occupies the entire ~116 GB `sdb3` PV.
- **Consequence for the 4 TB HDD plan:** growing `/` is impossible with the current
  pool — there is no free space to `lvextend` into. The migration must be:
  `vgextend ubuntu-vg <new-PV>` → `lvextend` root into the new space → `resize2fs`.
  (PV → VG/LV → filesystem, bottom-up.)
- Root usage is volatile (~26–31% across June 28); track consumers
  (`/var/lib/docker` images, `/opt/docker` service data), not the percentage.
