# 08 — โมเดลเล็กในเครื่อง (Z.I.Z Mini-models)

> ร่าง v0.2 · 2026-11-01 · **เปลี่ยนคำตอบ Q8** จาก "ห้ามมีโมเดลใน v1" → "มีโมเดลเล็กในเครื่องได้ ไม่มี API key ไม่มี cloud"
> (เจ้าของสั่งเพิ่ม 2026-11-01) · ที่มาของแนวคิดแพ็กโมเดล: Kensaku (`src-tauri/src/models.rs`, `docs/ARCHITECTURE.md` §5.3) — ดู `09-fast-read-index.md`

---

## 1. ทำไม และขอบเขต

CLI agent ของผู้ใช้ (Claude Code / Codex / Gemini) เก่งแต่ **ช้า แพง และกินโควตาของผู้ใช้** — งานส่วนใหญ่ใน pipeline
ของ Z.I.Z (`04-internals.md` §5) เป็นงาน "จัดกอง" ที่โมเดลเล็กทำได้ดีพอ: หาชื่อเฉพาะ คัด chunk ที่ไม่เกี่ยว ค้นข้อมูลเดิมด้วยความหมาย
จับรูปแบบการทำซ้ำของผู้ใช้

**หลัก:** โมเดลเล็ก = **คัดกรองและเตรียม** · CLI agent = **ตัดสินและเขียนเนื้อหา** · ผู้ใช้ = **อนุมัติ**

| ทำ | ไม่ทำ |
|---|---|
| รันบน CPU ล้วน (ไม่ต้องมี GPU) · รวมทุกตัว RAM ≤ 250 MB ตอนทำงาน · 0 MB ตอนว่าง | ไม่มี API key · ไม่ส่งข้อมูลออกนอกเครื่อง · ไม่ telemetry |
| ผลของโมเดลเล็กเป็น **ผู้สมัคร (candidate)** เสมอ — ต้องผ่านตรวจ quote (`04` §5.7) และผู้ใช้ตรวจ | โมเดลเล็กไม่สร้าง op ที่เขียนลงแอปโดยตรง |
| ไม่มีโมเดล → Z.I.Z ทำงานเหมือนเดิมทุกอย่าง (fallback = ส่งให้ agent ทั้งหมด) | ไม่บังคับดาวน์โหลด |
| แยก repo ต่อโมเดล · ส่งเป็น release pack · อัปเดตได้ ย้อนได้ | ไม่ฝังน้ำหนักโมเดลใน installer ของ Z.I.Z |

---

## 2. รายชื่อโมเดล (roster v1)

| id (= ชื่อ repo) | งาน | รูปแบบ | ขนาดเป้า | แบ่งภาระอะไรจาก CLI |
|---|---|---|---|---|
| **`ZIZ-M-embed`** | เวกเตอร์ความหมาย TH+EN (ค้นด้วยความหมาย · จับคู่ชื่อ/alias · หา entity เดิมในแอป) | model2vec static table, i8 + scale ต่อแถว (แบบ Kensaku) — ไม่มี neural graph ตอน query | ≈ 15–30 MB (ตัด vocab เหลือ TH+EN — verify) | agent ไม่ต้องเรียก `ziz_app_search` หลายรอบเพื่อหา "ตัวละครนี้มีอยู่แล้วไหม" |
| **`ZIZ-M-ner`** | ติดป้ายชื่อเฉพาะระดับตัวอักษร: บุคคล · สถานที่ · องค์กร · สิ่งของ · เวลา | char-level tagger (distill จากโมเดลไทยใหญ่) ONNX int8 | ≈ 10–25 MB | ทำ pre-pass หา "รายชื่อผู้สมัคร + ตำแหน่งที่พบ + จำนวนครั้ง" ให้ agent → agent ยืนยัน/รวมชื่อ/เขียนคำอธิบาย แทนการอ่านหาเองทั้งเล่ม |
| **`ZIZ-M-route`** | จัดประเภท chunk (บทสนทนา · บรรยาย · ฉากต่อสู้ · ข้อความนอกเรื่อง/สารบัญ · **สงสัยเป็นคำสั่งแฝง**) + คะแนนความเกี่ยวกับ recipe | linear head บนเวกเตอร์ของ `ZIZ-M-embed` | < 1 MB | ข้าม chunk ที่ไม่เกี่ยว (สารบัญ/คำนำ/โฆษณาในไฟล์) · ช่วยตัวตรวจ injection (`04` §7.2) |
| **`ZIZ-M-habit`** | จับรูปแบบที่ผู้ใช้ทำซ้ำ + จัดอันดับคำแนะนำ (`10-ui-home-mirror-spaces.md` §4) | **ไม่ใช่ neural**: sequence mining (n-gram 2–6 ของ action) + Markov ลำดับถัดไป · pack มีแค่ vocab ของ action ต่อแอป + ค่าเริ่มต้น | < 1 MB (สถานะที่เรียนรู้อยู่ในเครื่องผู้ใช้) | agent ไม่ต้องถูกเรียกเพื่อ "เดา" ว่าผู้ใช้จะทำอะไรต่อ |
| **`ZIZ-M-gen`** (v1 · **opt-in** — Q31 ✅) | สรุปบท/synopsis สั้น · เขียน "สรุปไว" ของ Link เป็นย่อหน้า | LLM เล็ก ~0.5B GGUF Q4 · runtime แยก process (❓ Q43) | ≈ 300–400 MB | งานที่หนักที่สุดของ agent · ไม่ติดตั้งเป็นค่าเริ่ม · ผลทุกครั้งติดป้าย "สรุปโดย gen (โมเดลเล็ก)" และต้องผ่านตรวจ quote ก่อนกลายเป็นข้อเสนอ · ถ้าคะแนน eval ภาษาไทยต่ำกว่าเกณฑ์ ปุ่มติดตั้งแสดงคำเตือนคุณภาพ |

> ponytail: ไม่มีโมเดลตัดคำไทยแยก — index ใช้ n-gram (`09` §3.2) และ NER เป็น char-level จึงไม่ต้องตัดคำ ·
> ไม่มี reranker — RRF พอ (Kensaku §2: cross-encoder 30–60 ms ไม่คุ้ม) · เพิ่มเมื่อวัดได้ว่าขาด

### 2.1 ตัวชี้วัดที่ต้องวัดก่อนประกาศว่า "ช่วยได้" (เป้า ไม่ใช่ผลที่วัดแล้ว)

| ตัวชี้วัด | เป้า v1 | ชุดทดสอบ |
|---|---|---|
| ตัวอักษรที่ส่งให้ agent ต่อนิยาย 1 เล่ม | ลด ≥ 40 % | `tests/fixtures/ingest/novel-*` (`02` §6) |
| recall ตัวละครหลัก (ปรากฏ ≥ 3 ครั้ง) หลังใช้ NER pre-pass | ลดไม่เกิน 2 จุด เทียบ agent ล้วน | ชุดเดียวกัน + เฉลยมือ |
| เวลาจากกด "วิเคราะห์" ถึงข้อเสนอแรก | เร็วขึ้น ≥ 2× | — |
| injection ที่ route จับได้ (ชุด `04` §7.2) | ≥ 90 % · false positive ≤ 3 % | ชุด adversarial ภาษาไทย |

ไม่ผ่านเป้า → โมเดลนั้นไม่เปิดเป็นค่าเริ่มต้น (ยังให้ผู้ใช้เปิดเองได้)

---

## 3. ตัวแบ่งงาน (router) — ใครทำอะไร

ตายตัว (deterministic) ไม่มีโมเดลตัดสินว่าจะใช้โมเดล:

| ขั้นใน job | มีโมเดล | ไม่มีโมเดล |
|---|---|---|
| ① ingest + chunk | เหมือนเดิม (`04` §5.1–5.4) + `ziz-index` cache IR (`09` §4) | เหมือนเดิม |
| ② คัด chunk | `route` ให้คะแนน → chunk คะแนน < เกณฑ์ (ค่าเริ่ม 0.15) = **ข้าม แต่แสดงในรายงานว่าข้าม** + ผู้ใช้กด "อ่านส่วนที่ข้ามด้วย" ได้ | ส่งทุก chunk |
| ③ หาชื่อเฉพาะ | `ner` → ผู้สมัคร + offset → `embed` จัดกลุ่ม alias ("อาริน" / "ริน" / "Arin") → ส่งให้ agent เป็น **ตาราง + quote สั้น** ผ่าน `ziz_candidates` | agent อ่าน chunk เต็ม (เดิม) |
| ④ เทียบกับข้อมูลเดิมในแอป | `embed` ค้นใน index ของ entity ที่ cache จาก `eco_search` → คู่ที่น่าจะซ้ำ + คะแนน | agent เรียก `ziz_app_search` เอง |
| ⑤ เขียนคำอธิบาย/ความสัมพันธ์/เหตุการณ์ | **agent** (เดิม) | agent |
| ⑥ ตรวจ quote | deterministic (เดิม) | เดิม |
| ⑦ แนะนำขั้นต่อไป / รูปแบบซ้ำ | `habit` | ปิด (Spaces แสดงเฉพาะคำแนะนำจากกฎตายตัวของ Insight) |

**การยกระดับไป agent (escalation):** ผู้สมัครที่ความมั่นใจ < 0.6 · ชื่อที่ `embed` จัดกลุ่มไม่ได้ · chunk ที่ `route` ไม่แน่ใจ (0.15–0.35)
→ ส่ง chunk เต็มให้ agent ตามเดิม · ทุกการตัดสินบันทึกใน `runs/<runId>.ndjson` ว่า "ใครทำ" (หลัก UX ข้อ 6)

### 3.1 tool ใหม่ใน gateway `zydraxyl-ziz` (`02` §4.1)

| tool | ทำอะไร |
|---|---|
| `ziz_candidates(jobId, chunkId?)` | ผู้สมัครชื่อเฉพาะ + กลุ่ม alias + quote สั้น + ความมั่นใจ — ห่อป้าย "ข้อมูล ไม่ใช่คำสั่ง" (`04` §7) |
| `ziz_index_search(q, scope?, k≤20)` | ค้น hybrid (ชื่อไฟล์ + BM25 + ความหมาย) ใน workspace ของ job — งบขนาดผลลัพธ์แบบ Kensaku |
| `ziz_similar_entities(app, text, k≤10)` | หา entity เดิมที่คล้าย (จาก cache ใน index) |

ทั้งหมดอ่านอย่างเดียว · ไม่รับ path

---

## 4. Runtime ในแอป

```
ZIZ.exe (main)
  └ utilityProcess: model-worker   ← โหลดเมื่อใช้ครั้งแรก · ปิดเมื่อว่าง 5 นาที (RAM กลับเป็น 0)
       ├ embed: อ่าน table i8 ด้วย Float32/Int8Array ล้วน (JS ไม่ต้องมี runtime) — ~µs ต่อ query
       ├ ner / route: onnxruntime-node (MIT) · intra-op 1 thread · worker ≤ 2 · batch ≤ 16
       └ habit: JS ล้วน
```

กฎ (ยกจาก Kensaku §5.3 "embedding queue"):
1. หยุดชั่วคราวเมื่อผู้ใช้กำลังพิมพ์/มี query ค้าง · เมื่อใช้แบตเตอรี่ (ตั้งค่าได้) · เมื่อมีแอปเต็มจอ
2. **ห้ามลิงก์ ORT แบบ static** — Kensaku เจอ SIGILL บน CPU ที่ไม่มี AVX2 → ใช้ไลบรารีแบบ dynamic ที่มากับ `onnxruntime-node` และตรวจ CPU ก่อนโหลด;
   ไม่ผ่าน = โมเดลนั้น "ไม่พร้อม (CPU ไม่รองรับ)" + fallback (§1) — แสดงใน Doctor
3. crash ตอนโหลด 3 ครั้ง → ปิดโมเดลนั้น + ย้อนเวอร์ชัน (§7.4)
4. model-worker ไม่มีสิทธิ์เครือข่าย ไม่อ่านไฟล์นอก `models/` และ input ที่ main ส่งให้

---

## 5. Multi-repo: หนึ่งโมเดล หนึ่ง repo + catalog กลาง

```
ZYDRAXYL/
  Z.I.Z-DSG  (→ Z.I.Z)     แอป · รู้จัก ABI ที่รองรับ · ฝัง public key ของ catalog
  ZIZ-MODELS               catalog กลาง: catalog.json (ลงลายเซ็น) · ABI spec · เครื่องมือ pack/verify · ชุด eval กลาง
  ZIZ-M-embed              ┐
  ZIZ-M-ner                │ repo ต่อโมเดล: train · export · eval · release pack
  ZIZ-M-route              │
  ZIZ-M-habit              ┘
  ziz-index                daemon อ่านไฟล์เร็ว (`09-fast-read-index.md`) — แยก repo เพราะแอป Eco อื่นใช้ร่วม
```

**ทำไมแยก repo ต่อโมเดล:** รอบ release ไม่ผูกกับแอป (train ใหม่ไม่ต้องออกเวอร์ชัน Z.I.Z) · licence ของโมเดลฐานต่างกัน ·
ประวัติ/ขนาดไฟล์ไม่ปน repo แอป · CI ของโมเดลใช้ Python/GPU ได้โดยไม่ลาก CI ของแอป

### 5.1 โครง repo โมเดล

```
ZIZ-M-ner/
  pack.json            ← ข้อมูลของ pack (§6.1) — แหล่งความจริงเดียวของ id/version/abi
  MODEL_CARD.md        ← ข้อมูลฝึก (ไม่มีข้อมูลผู้ใช้) · ข้อจำกัด · ภาษา · licence ของโมเดลฐาน
  LICENSE  THIRD-PARTY.md
  train/               ← สคริปต์ฝึก (reproducible, pin เวอร์ชัน) — น้ำหนักโมเดล **ไม่ commit**
  export/              ← แปลงเป็น ONNX int8 / table i8
  eval/                ← ชุดทดสอบ + เกณฑ์ขั้นต่ำ (thresholds.json)
  selftest.json        ← input 5–20 ตัว + output ที่คาด (ใช้ตรวจหลังติดตั้งบนเครื่องผู้ใช้)
  .github/workflows/
    ci.yml             ← lint + eval บนโมเดลล่าสุด
    release.yml        ← tag v* → (§6.2)
```

### 5.2 ABI — สัญญาระหว่างแอปกับโมเดล

| ABI | input | output |
|---|---|---|
| `embed/1` | ข้อความ UTF-8 (NFC + normalize ไทย `04` §5.6) | `Float32[256]` unit-norm |
| `ner/1` | ข้อความ ≤ 4,000 ตัวอักษร | `[{start, end, label: PER\|LOC\|ORG\|ITEM\|TIME, score}]` (offset = UTF-16 code unit ตาม JS) |
| `route/1` | `Float32[256]` จาก `embed/1` | `{ kind: {...prob}, relevance: {recipeId: score}, injection: score }` |
| `habit/1` | ลำดับ action token (`10` §4.2) | `[{pattern, support, nextAction, score}]` |

- ABI อยู่ใน `ZIZ-MODELS/abi/*.md` · แอปประกาศ ABI ที่รองรับ: `{ "embed": [1], "ner": [1], "route": [1], "habit": [1] }`
- `route/1` **ขึ้นกับ** `embed/1` รุ่นที่ใช้ฝึก → `pack.json.dependsOn: { "ZIZ-M-embed": ">=1.2.0 <1.3.0" }` · ตัวติดตั้งแก้ dependency ให้

---

## 6. Pack เป็น release

### 6.1 `.zizpack` = zip ธรรมดา

```
ziz-m-ner-1.3.0.zizpack
  pack.json
  model.onnx            (หรือ table.i8 + scales.f32 + vocab.json สำหรับ embed)
  tokenizer.json        (ถ้ามี)
  selftest.json
  LICENSE  MODEL_CARD.md
```

```json
{
  "id": "ZIZ-M-ner", "version": "1.3.0", "abi": "ner/1",
  "requires": { "ziz": ">=1.2.0 <2.0.0" },
  "dependsOn": {},
  "runtime": "onnx", "cpu": { "minIsa": "sse4.2" },
  "files": [ { "path": "model.onnx", "sha256": "…", "size": 18442210 } ],
  "ramPeakMb": 60, "license": "Apache-2.0", "baseModelLicense": "(verify)",
  "languages": ["th", "en"], "eval": { "f1_per": 0.91, "f1_loc": 0.86 }
}
```
ไวยากรณ์ช่วงเวอร์ชันตาม ECO-K5 (เหมือน profile CLI ใน `04` §8)

### 6.2 workflow release ของ repo โมเดล

```
maintainer ฝึกบนเครื่องตัวเอง → อัปโหลดน้ำหนักเป็น asset ของ draft release
tag vX.Y.Z → release.yml:
  1. ดึงน้ำหนักจาก draft · export (ถ้ายังไม่ได้ export)
  2. รัน eval/ → ต่ำกว่า thresholds.json = ล้ม (ห้ามปล่อยโมเดลที่แย่ลง)
  3. รัน selftest บน Windows runner ด้วย onnxruntime-node รุ่นเดียวกับแอป
  4. สร้าง .zizpack + sha256 → publish release
  5. เปิด PR เข้า ZIZ-MODELS: เพิ่ม/แก้ entry ใน catalog (id, version, channel, sha256, size, url, abi, requires, eval)
ZIZ-MODELS (merge PR) → catalog.yml: ตรวจ entry (ดาวน์โหลด pack, เทียบ sha256, ตรวจ requires/abi/dependsOn)
                      → ลงลายเซ็น ed25519 → publish `catalog.json` + `catalog.json.sig` เป็น release asset "latest"
```

- **กุญแจลงลายเซ็นมีที่เดียว** (secret ของ `ZIZ-MODELS`) — repo โมเดลไม่ถือกุญแจ · pack ปลอดภัยเพราะ sha256 อยู่ใน catalog ที่ลงลายเซ็น
- แอปตรวจลายเซ็นด้วย `crypto.verify('ed25519', …)` ของ Node — ไม่ต้องมี dependency เพิ่ม
- ช่อง (channel): `stable` · `beta` — entry มี `channel` · ผู้ใช้เลือกในตั้งค่า (ค่าเริ่ม stable)

### 6.3 `catalog.json`

```json
{
  "schema": 1, "generatedAt": "2026-11-01T00:00:00Z", "expires": "2026-12-01T00:00:00Z",
  "models": [
    { "id": "ZIZ-M-ner", "version": "1.3.0", "channel": "stable", "abi": "ner/1",
      "requires": { "ziz": ">=1.2.0 <2.0.0" }, "dependsOn": {},
      "size": 18501120, "sha256": "…",
      "urls": [ "https://github.com/ZYDRAXYL/ZIZ-M-ner/releases/download/v1.3.0/ziz-m-ner-1.3.0.zizpack" ],
      "notes": { "th": "แม่นขึ้นกับชื่อที่มีคำนำหน้า", "en": "…" } }
  ]
}
```
`expires` กันการเสิร์ฟ catalog เก่าซ้ำ (rollback attack) — หมดอายุ = ไม่อัปเดต แต่โมเดลที่ติดตั้งแล้วยังใช้ได้ ·
แอปดึงไฟล์เดียวนี้ (ไม่เรียก GitHub API ทีละ repo → ไม่ชน rate limit 60 ครั้ง/ชม. ของคำขอไม่ยืนยันตัวตน)

---

## 7. ติดตั้ง · อัปเดต · ย้อน บนเครื่องผู้ใช้

### 7.1 ที่เก็บ

```
%LOCALAPPDATA%\ZIZ\models\
  active.json                      { "ZIZ-M-ner": "1.3.0", "ZIZ-M-embed": "1.2.4", … }  ← เปลี่ยนด้วย write-tmp + rename เท่านั้น
  ZIZ-M-ner\1.3.0\…                เวอร์ชันที่ใช้
  ZIZ-M-ner\1.2.1\…                เวอร์ชันก่อนหน้า (เก็บไว้ 1 รุ่นเพื่อย้อน)
  .staging\ziz-m-ner-1.4.0.zizpack.part
  catalog.json  catalog.json.sig   cache ล่าสุด
```
อยู่ในโปรไฟล์ผู้ใช้ ไม่อยู่ในโฟลเดอร์โปรเจกต์ (โมเดลใช้ร่วมทุก workspace) · ไม่อยู่ใน `%LOCALAPPDATA%\Zydraxyl\` (`02` §7)

### 7.2 ขั้นการอัปเดต

```
ตรวจ (เปิดแอป · ไม่เกินวันละครั้ง · หรือกด "ตรวจตอนนี้")
 → ดึง catalog.json + .sig → ลายเซ็นผิด/หมดอายุ = หยุด + แจ้งใน Doctor
 → ต่อโมเดล: เลือกเวอร์ชันสูงสุดใน channel ที่ abi ∈ ที่แอปรองรับ และ requires.ziz ตรง และ dependsOn แก้ได้
 → ใหม่กว่า active? → ตามนโยบาย: อัตโนมัติ | แจ้งก่อน (ค่าเริ่ม) | ด้วยตัวเอง
 → ดาวน์โหลดลง .staging (HTTP Range ต่อจากเดิมได้ · ลอง urls ตามลำดับ)
 → sha256 ตรง catalog? → แตก zip ลง <id>\<ver>\ (ปฏิเสธ path ที่มี .. หรือ absolute — zip-slip)
 → sha256 ต่อไฟล์ตรง pack.json? → model-worker รัน selftest.json (ค่าคลาดเคลื่อน ≤ 1e-3 สำหรับเวกเตอร์ · label ต้องตรง)
 → ผ่าน: เขียน active.json ใหม่ · ลบรุ่นที่เก่ากว่ารุ่นก่อนหน้า · toast "อัปเดต ner เป็น 1.3.0 · ย้อนได้"
 → ไม่ผ่าน: ลบ <ver> · คงรุ่นเดิม · บันทึกเหตุผล
```

### 7.3 ความสอดคล้องระหว่างอัปเดต

- **job ที่กำลังรันปักเวอร์ชันโมเดล** (`job.model_versions` JSON เหมือน `recipe_version` — `04` §8) — สลับรุ่นมีผลกับ job ใหม่เท่านั้น ·
  ผลใน `runs/*.ndjson` บอกรุ่นที่ใช้ → ทำซ้ำได้
- **เปลี่ยน `embed` = เวกเตอร์เดิมใช้ไม่ได้** → ทำแบบ Kensaku: เวกเตอร์ผูก `model_id` (`<id>@<ver>`) · index เก่ายังเสิร์ฟจนกว่า re-embed เบื้องหลังเสร็จ ·
  ไม่มีวันผสมเวกเตอร์ต่างรุ่นในผลค้นเดียว · `route` ที่ผูกกับ embed รุ่นเก่าอัปเดตพร้อมกัน (dependsOn)
- อัปเดต MAJOR ของ ABI ที่แอปยังไม่รองรับ → ไม่เสนอ · แสดง "มีโมเดลใหม่ ต้องอัปเดต Z.I.Z ก่อน"

### 7.4 ย้อน

- ตั้งค่า → โมเดล → `ย้อนเป็น 1.2.1` (รุ่นก่อนหน้าที่เก็บไว้) — เปลี่ยน active.json ทันที ไม่ต้องดาวน์โหลด
- อัตโนมัติ: selftest ไม่ผ่าน หรือโหลดแล้ว crash 3 ครั้ง (§4 ข้อ 3)
- ปักรุ่น (pin): ผู้ใช้ล็อกโมเดลไว้ไม่ให้อัปเดตได้

### 7.5 ออฟไลน์

ลากไฟล์ `.zizpack` + `catalog.json`/`.sig` ที่ดาวน์โหลดจากเครื่องอื่นเข้าหน้า "โมเดล" → ตรวจเหมือน §7.2 ทุกขั้น ·
pack ที่ไม่อยู่ใน catalog ที่ลงลายเซ็น = **ปฏิเสธ** (ยกเว้นโหมดนักพัฒนา ที่ตั้งด้วย flag `--dev-models` และแสดงป้ายแดงตลอดเวลา)

### 7.6 Zydraxyl Hub (later)

Hub ติดตั้งแอปหลายเวอร์ชันอยู่แล้ว (ECO-K7) → ต่อไป Hub อ่าน `catalog.json` เดียวกันเพื่อจัดการโมเดลให้ทุกแอปที่ใช้ (เช่น `ziz-index` ที่แอปอื่นใช้ embed ร่วม) ·
v1: Z.I.Z จัดการเอง — ข้อเสนอเข้า Eco contract ใน `09` §5.1 (P13)

---

## 8. UI ที่เกี่ยวข้อง (รายละเอียดหน้าจอ `10` §5)

- ตั้งค่า → **โมเดล**: การ์ดต่อโมเดล — สถานะ (ไม่ได้ติดตั้ง · พร้อม · กำลังดาวน์โหลด 34 % · มีรุ่นใหม่ · ปิดอยู่ · CPU ไม่รองรับ) · รุ่น · ขนาด · RAM ·
  ปุ่ม `ติดตั้ง` / `อัปเดต` / `ย้อน` / `ปิด` / `ปักรุ่น` · ลิงก์ MODEL_CARD
- เริ่มครั้งแรก (Z1) เพิ่มขั้นเลือกได้ "ดาวน์โหลดโมเดลช่วยงาน (≈ 50 MB)" — ข้ามได้ (แบบ Kensaku: optional packs ใน setup)
- การ์ดงาน (Z5) แสดง "ใครทำ" ต่อขั้น: `ner 1.3 · หาชื่อ 214 ผู้สมัคร` → `Claude Code · ยืนยัน 38`
- Doctor: CPU ISA · โมเดลที่โหลดได้/ไม่ได้ + เหตุผล fallback · อายุ catalog · ลายเซ็น

---

## 9. ความปลอดภัย

| ภัย | กัน |
|---|---|
| pack ถูกสลับระหว่างดาวน์โหลด | sha256 จาก catalog ที่ลงลายเซ็น ed25519 · HTTPS |
| catalog เก่าถูกเสิร์ฟซ้ำ | `expires` + ไม่ยอมรับ `generatedAt` เก่ากว่าที่มี |
| zip-slip / symlink ใน pack | แตกเองทีละไฟล์ · ปฏิเสธ `..`/absolute/symlink · ตรวจ sha ต่อไฟล์ |
| ONNX ที่มี custom op อันตราย | โหลดด้วย session options ปิด custom op library · ใช้เฉพาะ op มาตรฐาน · eval ใน CI ของ catalog |
| ผลของโมเดลถูกเนื้อหาในไฟล์ปั่น (adversarial) | ผลโมเดล = candidate เท่านั้น · ผ่านตรวจ quote · ผู้ใช้ตรวจ · ไม่มีเส้นทางเขียนอัตโนมัติ |
| ข้อมูลการใช้งานของผู้ใช้ (habit) รั่ว | เก็บใน `ziz-global.db` เท่านั้น · ไม่มีการส่งออก · ปุ่ม "ลบประวัติการเรียนรู้" · ปิดได้ทั้งระบบ |
