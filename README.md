# ammonium-lab-utilities

Utility scripts สำหรับดูแล Proxmox homelab (คลัสเตอร์ `ubuntu-ammonium`, node pve01-06) และ
fleet ของ LXC container ที่รัน orchestrator/CI runner ให้ repo [`parts-inventory`](https://github.com/ppongprom/parts-inventory)

Repo นี้เป็น public เพราะไม่มี credential ฝังอยู่เลย (ทุกอย่างดึงสดผ่าน `gh api`/`ssh` ตอนรัน) —
IP ทั้งหมดที่ปรากฏเป็น private range (`192.168.88.0/24`, `fd88:69::/64`) เข้าถึงได้เฉพาะผ่าน
Tailscale เท่านั้น ไม่มี port forward จากภายนอก

## bin/fleet-utils

รวมงานดูแล fleet ที่เคยรันแบบ ad-hoc (26 ส.ค. 2569 — แก้ GitHub Actions runner ที่ค้าง OOM
ทั้ง 5 เครื่อง app11-15 + เก็บกวาด disk ทั้ง fleet กู้คืนได้ ~48GB)

```
fleet-utils discover              # ดู mapping LXC↔pve host สดทุกครั้ง (ไม่ hardcode)
fleet-utils runner-status         # เช็คสถานะ GitHub Actions runner ทั้ง 5 ตัว (app11-15)
fleet-utils runner-fix <app11>    # re-register runner ตัวเดียว (แก้ token/OOM ค้าง)
fleet-utils runner-fix-all        # ไล่แก้ทุกตัวที่ว่าง (ข้ามตัวที่กำลังทำงานอยู่)
fleet-utils disk-survey <app01>   # สำรวจ disk หา candidate ลบ (read-only)
fleet-utils disk-clean-safe <app01> # ลบเฉพาะ cache/log ที่ปลอดภัยชัดเจน
```

**ทำไม dynamic discovery ไม่ hardcode host/vmid:** mapping ระหว่าง container กับ pve host
เปลี่ยนได้ตลอดเวลา (Proxmox live migration) — เจอจริง 27 ส.ค. 2569 ตอน `conductor`/`app08`
ถูกย้ายออกจาก `pve03` และพบ `pve05` เป็น node ที่ไม่เคยรู้จักมาก่อนจนกว่าจะเช็คคลัสเตอร์ตรงๆ
ทุกคำสั่งใน script นี้เช็ค `pvecm nodes` + `pct list` สดก่อนเสมอ

รันจากเครื่องที่มี SSH key เข้า pve host ทุกตัวได้ (`root@pveXX`) และ login `gh` แล้ว
(ใช้ `gh` สำหรับ GitHub Actions runner registration token)

**ข้อควรระวัง:** `runner-fix-all` และ `disk-clean-safe` เป็น destructive action จริง — ทดสอบ
`discover`/`runner-status`/`disk-survey` (read-only) ก่อนเสมอ แล้วค่อยรันตัวที่แก้ไขจริงทีละเครื่อง

## bin/cpu-clock

วัดความเร็ว CPU ของทุก node ในคลัสเตอร์ (pve01-06) — ทั้ง clock ที่ตั้งไว้ (lscpu) และ
compute throughput จริง (openssl speed sha256 พร้อมกันทุก node) เพราะ MHz อย่างเดียวไม่บอกว่า
เครื่องไหนช้าจริงเวลามีงานพร้อมกันหลายตัว ไม่ต้องติดตั้งอะไรเพิ่ม (lscpu/openssl มีอยู่แล้วทุกเครื่อง)

```
cpu-clock static   # lscpu ของทุก node (model, min/max MHz, จำนวน core)
cpu-clock bench    # openssl speed sha256 จริง พร้อมกันทุก node (~10 วิ)
cpu-clock all      # ทั้งสองอย่าง (ค่าเริ่มต้นถ้าไม่ระบุ)
```

## บริบทเพิ่มเติม

Root cause ของปัญหา runner ค้าง OOM (ที่ `runner-fix` แก้) และรายละเอียด cluster mapping
บันทึกไว้ใน Claude memory: `parts-inventory-selenium-runner-chrome-leak.md` และ
`homelab-proxmox-ip-map.md`
