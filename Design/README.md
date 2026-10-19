# Z.I.Z — เอกสารออกแบบ

> **สถานะ: ออกแบบ ยังไม่สร้าง** · ร่าง v0.1 · 2026-10-18
> app id `ziz` · format code `ziz` (ECO-I1) · Eco v1 = Windows PC เท่านั้น

## Z.I.Z คืออะไร

Z.I.Z คือ "ห้องควบคุม" ของ Zydraxyl Eco: เปิดโฟลเดอร์โปรเจกต์ Eco แล้ว (1) แสดง**สถิติและความเชื่อมโยงข้ามแอป**
แบบที่ Sage Hut ของ DraconDex ทำภายใน Nexus เดียว (`DraconDex-APP/docs/SYSTEMS.md` §8) และ (2) ใช้ **CLI agent ของผู้ใช้**
อ่านนิยาย ไอเดีย ไฟล์ Word/Excel แล้ว**เสนอ**การสร้าง/แก้ข้อมูลในแอปปลายทาง ผู้ใช้ตรวจ diff ทีละรายการ → เขียนผ่าน MCP
ของแอปเจ้าของข้อมูล (ECO-C4–C6) → ย้อนกลับได้ (ECO-S3)

สามคำที่ผู้ใช้ต้องรู้: **แหล่ง** (สิ่งที่ป้อนเข้า) → **ข้อเสนอ** (changeset ที่ยังไม่เขียน) → **การเขียน** (ลงแอปแล้ว มีจุดย้อน)

## สารบัญ

| ไฟล์ | เนื้อหา |
|---|---|
| [`01-research-summary.md`](./01-research-summary.md) | สรุปการค้นคว้า: ผลิตภัณฑ์/OSS ที่เทียบ · สิ่งที่ยืม · สิ่งที่เลี่ยง · ข้อสรุปต่อหัวข้อ |
| [`02-architecture.md`](./02-architecture.md) | stack · process model · การขับ agent (ACP ก่อน, headless สำรอง) · gateway MCP · ชั้น · โฟลเดอร์ · ของที่ยกมาจาก DraconDex |
| [`03-features.md`](./03-features.md) | รายการฟีเจอร์ MVP / v1 / later พร้อมเหตุผลและที่มา |
| [`04-internals.md`](./04-internals.md) | data model (SQLite) · รูปแบบไฟล์ · pipeline นำเข้า (DOCX/XLSX/นิยายไทยยาว) · changeset · recipe · การป้องกัน prompt injection · migration |
| [`05-ux-ui.md`](./05-ux-ui.md) | หน้าจอ · ASCII layout · flow ทีละขั้น · onboarding · ธีม · accessibility · ตัวอักษรไทย |
| [`06-eco-integration.md`](./06-eco-integration.md) | format ที่อ่าน/เขียน · MCP client/gateway · recipe ต่อแอป · locator · fixture · ข้อเสนอเพิ่มเข้า Eco contract |
| [`07-grill-me.md`](./07-grill-me.md) | 29 คำถามที่เจ้าของต้องตัดสิน (เรียงเป็นรอบ) |

## ลำดับการอ่าน

1. ไฟล์นี้ → `07-grill-me.md` รอบ 1 (การตัดสินใจที่ทุกอย่างขึ้นกับมัน)
2. `02-architecture.md` §2–§4 (agent + gateway) — หัวใจทางเทคนิค
3. `04-internals.md` §5–§7 (pipeline + changeset + injection) — หัวใจด้านความปลอดภัยของข้อมูลผู้ใช้
4. `06-eco-integration.md` — สิ่งที่แอปอื่นต้องทำให้ Z.I.Z ทำงานได้
5. ที่เหลือตามต้องการ

## เอกสารภายนอกที่อ้างถึง

- สัญญา Eco: `Zydraxyl-HUB/Design/eco/ECO-CONTRACT.md` (อ้างด้วยรหัส ECO-*)
- งานวิจัยดิบ: `Zydraxyl-HUB/Design/eco/research/T1-A-structure.md` · `T2-A-features.md` · `T3-A-internals.md` · `T4-A-uxui.md` · `T5-eco-research.md`
- DraconDex: `DraconDex-APP/docs/SYSTEMS.md` §8 (Sage) · `V5.md` §11.9 (Sage Hut → Insight) · `PLUGINS.md` · `TRANSFER.md`

## สัญลักษณ์

❓ = รอเจ้าของตัดสิน (ลิงก์ไปคำถามใน `07-grill-me.md`) · **(verify)** = ข้อเท็จจริงภายนอกที่ต้องตรวจอีกครั้งก่อนเขียนโค้ด
