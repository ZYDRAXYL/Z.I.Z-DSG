# 09 — อ่านไฟล์ให้เร็ว: `ziz-index` (ปรับจาก Kensaku)

> ร่าง v0.2 · 2026-11-01 · อ้างอิง **Kensaku** https://github.com/yjid/kensaku (MIT · Rust + Tauri 2 · v1.0.11 ที่อ่าน) —
> `docs/ARCHITECTURE.md` · `docs/AGENTS.md` · `src-tauri/src/{indexer,search,query,embed,agent,models.rs}` **[V]** อ่านจากต้นทางแล้ว

---

## 1. Kensaku ทำอะไร และ Z.I.Z ยืมอะไร

Kensaku = ตัวค้นไฟล์ในเครื่องที่ "อุ่นอยู่ตลอด": daemon ตัวเดียวถือ index · GUI/CLI/MCP เป็น client ผ่าน named pipe ·
ค้นชื่อไฟล์ 100k ไฟล์ ~1.3 ms ด้วย RAM ~12 MB · index ซ้ำเมื่อไม่มีอะไรเปลี่ยน 0.56 s (ตัวเลขของ Kensaku บน Linux 8 vCPU — **ไม่ใช่ตัวเลขของ Z.I.Z**)

ปัญหาของ Z.I.Z ตอนนี้ (`04-internals.md` §5): ทุกครั้งที่นำเข้า/วิเคราะห์ จะเปิด DOCX ด้วย mammoth ใหม่ทั้งไฟล์ · ไม่มี index ข้ามไฟล์ ·
agent หา "เคยเขียนถึงสิ่งนี้ที่ไหน" ไม่ได้นอกจากอ่านซ้ำ · แอป Eco อื่นแต่ละตัวก็ต้องอ่านไฟล์เองอีก

| แนวคิดของ Kensaku | ที่มา | ใช้ใน Z.I.Z อย่างไร |
|---|---|---|
| **daemon ตัวเดียว · single writer · client หลายตัวผ่าน pipe** · fallback อ่านอย่างเดียวเมื่อ daemon ไม่รัน | ARCH §3 | `ziz-index` เป็น process แยก — Z.I.Z, agent (ผ่าน gateway) และ**แอป Eco อื่น**ใช้ index เดียวกัน (§5) |
| **identity ไม่ใช่ path** (`file_id` ของ Windows) + `content_hash` BLAKE3 | ARCH §4.1, §7 ข้อ 1, 7 | ย้าย/เปลี่ยนชื่อไฟล์นิยายไม่ต้อง extract ใหม่ · ไฟล์เดิม = IR เดิม = chunk id เดิม (`04` §5.4 ใช้ sha256 อยู่แล้ว → เปลี่ยนเป็น BLAKE3 เพื่อความเร็ว แต่คง prefix 8 ตัวใน chunk id) |
| **search first, reconcile after** · stat-only reconcile · watcher เป็นแค่ "คำใบ้" | ARCH §7 ข้อ 3–4 | เปิดโปรเจกต์แล้วค้นได้ทันทีจาก snapshot เดิม · แถบสถานะบอก "index 4 วินาทีที่แล้ว" (หลัก UX ข้อ 5) |
| snapshot ในหน่วยความจำแบบ struct-of-arrays + `char_bloom` 64 bit กรองก่อน match | ARCH §4.2 | ค้นชื่อไฟล์/ชื่อบท/ชื่อ entity ใน command palette (`Ctrl+P`) ทันทีขณะพิมพ์ |
| **Tantivy BM25** + tokenizer เอง · ห้ามใช้ `QueryParser` · สร้าง query จาก AST | ARCH §5.2 | full-text ข้ามทุกแหล่งในโปรเจกต์ · **เพิ่ม Thai** (§3.2) |
| นโยบายเนื้อหา: เต็มถึง 1 MB · หัว 512 KB + ท้าย 512 KB · > 20 MB เก็บแค่ metadata | ARCH §5.2 | **ไม่ใช้ตรง ๆ** — นิยายต้องอ่านครบ: index เต็มทุกไฟล์ที่ผู้ใช้เพิ่มเป็น "แหล่ง" · ใช้นโยบายของ Kensaku กับไฟล์อื่นในโฟลเดอร์โปรเจกต์ |
| semantic tier: model2vec static i8 · ~50 µs/query · exact scan ≤ 400k unit | ARCH §5.3 | = `Z.I.Z-MDL-Embed` (`08` §2) · โปรเจกต์ Eco เล็กกว่า 400k unit มาก → exact scan อย่างเดียว ไม่ทำ HNSW |
| **model_id ต่อ index** — เปลี่ยนโมเดลแล้ว index เก่ายังเสิร์ฟจน index ใหม่เต็ม | ARCH §5.3 | กลไกอัปเดตโมเดล (`08` §7.3) |
| RRF รวมผลหลาย tier + recency + frecency | ARCH §6.5 | ค้น hybrid ใน `ziz_index_search` · frecency = ไฟล์/entity ที่ผู้ใช้เปิดบ่อย |
| extractor แยก worker · timeout · panic boundary · state `pending/ok/skipped/failed` + นับให้เห็น | ARCH §8 | ไฟล์ Word เสียไม่ทำ daemon ล้ม · "อ่านไม่ได้ 3 ไฟล์" แสดงใน Doctor |
| DOCX/XLSX ด้วย ZIP + quick-xml · **ปิด DTD** (billion-laughs) | ARCH §8 | extractor เร็วของ index (§4) — เร็วกว่า mammoth มากเพราะไม่สร้าง HTML |
| MCP: ผลลัพธ์มีงบขนาด · เนื้อหาไฟล์เป็น "ข้อมูล ไม่ใช่คำสั่ง" · อ่านอย่างเดียวโดยค่าเริ่ม | AGENTS.md | ตรงกับ `04` §7 อยู่แล้ว → tool ของ `ziz-index` ทำแบบเดียวกัน |
| แพ็กโมเดล: manifest คงที่ + sha256 + หลาย URL + Doctor แสดงเหตุผล fallback | `models.rs` | ต่อยอดเป็น catalog ที่ลงลายเซ็น + multi-repo (`08` §5–7) |
| UX invariants: พิมพ์ไม่เคยค้าง · ไม่ขึ้นจอว่าง→spinner→ผล · ผลไม่สลับลำดับขณะพิมพ์ · ความสดของ index เห็นเสมอ · filter เป็น chip ที่กดปิดได้ | ARCH §1 | ใช้กับ palette/ค้นหาของ Z.I.Z ทั้งหมด (`10` §6) |
| ภาษาธรรมชาติ → grammar ด้วยกฎตายตัว (`query/intent.rs`) · ทุกการตีความเป็น chip ที่ลบได้ | ARCH §9.1 | "บทที่พูดถึงอารินเมื่อสัปดาห์ก่อน" → `entity:อาริน modified:7d` เป็น chip |

**ไม่ยืม:** OCR/รูปภาพ (ไม่ใช่งานหลักของ Eco v1 — later) · tree-sitter `find_symbol` · launcher/tray · HTTP serve (ใช้ pipe อย่างเดียว) · index ทั้งเครื่อง (Z.I.Z index เฉพาะโปรเจกต์ที่เปิด)

---

## 2. ตัดสินใจเรื่อง stack — ❓ Q30

| ทาง | ข้อดี | ข้อเสีย |
|---|---|---|
| **(ก) `ziz-index` = Rust daemon แยก repo, ยกแกนจาก Kensaku แบบ copy-then-own** (แนะนำ) | ได้ของที่วัดผลแล้ว (Tantivy, watcher, arena) · แอป Eco อื่นใช้ร่วมได้โดยไม่ต้องมี Electron · MIT ยกได้ | เพิ่มภาษาที่สองในระบบ (Rust) · ต้อง build Windows x64 |
| (ข) ทำใน Node ด้วย SQLite FTS5 trigram ใน `ziz.db` | stack เดียว · น่าเบื่อ (ดี) | Kensaku วัดแล้ว FTS5 trigram ใหญ่และช้าที่ 1M doc (918 MB, 258 ms) — โปรเจกต์ Eco เล็กกว่ามากจึงอาจพอ · ต้อง verify ว่า `node-sqlite3-wasm` มี FTS5 · แอปอื่นใช้ร่วมไม่ได้ |
| (ค) รัน Kensaku ทั้งตัวเป็น dependency | เร็วที่สุดในการเริ่ม | index ทั้งเครื่อง (เกินขอบเขต) · ไม่มี tokenizer ไทย · รอบ release ไม่ใช่ของเรา |

ถ้าเลือก (ก): `vendor` ไม่ได้ — repo `Z.I.Z-IDX` (daemon `ziz-index`) คือ fork ที่ตัดส่วนไม่ใช้ + `PROVENANCE.json` (commit ต้นทาง · ไฟล์ · การเปลี่ยน) แบบเดียวกับ `02` §8 ·
คง `LICENSE` MIT ของ Kensaku + ใส่ชื่อผู้เขียนเดิมใน `THIRD-PARTY.md`

---

## 3. สิ่งที่ต้องเปลี่ยนจาก Kensaku

### 3.1 ขอบเขต = โปรเจกต์ Eco

- roots = `<EcoProject>\` (ไม่ใช่ทั้งดิสก์) + ไฟล์ "แหล่ง" ที่ผู้ใช้เพิ่มจากที่อื่น (index เฉพาะไฟล์นั้น)
- ข้ามเสมอ: `.zydraxyl\locks\` · `<App>\*.db-wal` · `ZIZ\scratch\` · `ZIZ\runs\` · ไฟล์ขยะของ Office (`~$*.docx` — Kensaku กรองอยู่แล้ว)
- index ของโปรเจกต์อยู่ที่ `<EcoProject>\ZIZ\index\` (ย้ายโปรเจกต์ไปเครื่องอื่นได้; ลบทิ้งแล้วสร้างใหม่ได้เสมอ — เป็น cache ไม่ใช่ความจริง)

### 3.2 tokenizer ภาษาไทย

Kensaku แตก CJK (Han/Hiragana/Katakana/Hangul) เป็น unigram + bigram สลับกัน (`indexer/tokenizer.rs::is_cjk`) แต่**ไม่รวมอักษรไทย**
(U+0E00–U+0E7F) → ข้อความไทยทั้งย่อหน้ากลายเป็น token เดียว ค้นคำกลางประโยคไม่เจอ

แก้: เพิ่มช่วงไทยเป็น "สคริปต์ไม่มีช่องว่าง" แต่**ตัดที่ขอบ grapheme ไทย** ไม่ใช่ทีละ code point —
สระบน/ล่างและวรรณยุกต์ (U+0E31, U+0E34–U+0E3A, U+0E47–U+0E4E) ติดกับพยัญชนะหน้าเสมอ:

```
"อารินเดินทาง" → grapheme: อา ริ น เดิ น ทา ง   (ponytail: ใช้ Intl.Segmenter แบบ grapheme ไม่ได้ใน Rust → ตารางช่วง combining ไทย 1 ฟังก์ชัน)
index: unigram ทุก grapheme + bigram ต่อเนื่อง  → "ริน" = bigram(ริ,น) ✓   "เดินทาง" = ลำดับ bigram ✓
```
- ค้นแบบ phrase บน bigram ต่อเนื่อง = ค้น substring ภาษาไทยได้แม่นโดยไม่ต้องตัดคำ (เหตุผลเดียวกับที่ Kensaku ทำกับ CJK)
- normalize ก่อน index ตาม `04` §5.6 (ลำดับสระ/วรรณยุกต์ซ้อน, `ํา` → `ำ`, เลขไทย ↔ อารบิกเก็บทั้งสองแบบ)
- bump `content.schema` → index เก่าถูกย้ายเป็น `index.legacy-*` และ re-extract อัตโนมัติ (กลไกเดิมของ Kensaku)
- ทดสอบ: fixture ประโยคไทยที่ค้นคำกลางประโยค · คำที่มีสระลอย · ชื่อที่สะกดสองแบบ

### 3.3 folded tier สำหรับไทย

Kensaku มีตาราง look-alike สำหรับ OCR (`0/o`, `rn/m`) → ไทยใช้ตารางสะกดที่สับสนบ่อยแทน: `ศ/ษ/ส` · `ณ/น` · `ฅ/ค` · `ฃ/ข` · `ใ/ไ` ·
วรรณยุกต์หาย — ใช้เฉพาะเมื่อ tier ตรงตัวได้ผลไม่พอ และต้องตรวจกับข้อความดิบเสมอ (กฎเดิมของ Kensaku: ผิดได้ 1 ตัวใน 4–7 ตัว)

---

## 4. "อ่านเร็ว" ใน pipeline ของ Z.I.Z: IR cache

ตอนนี้ (`04` §5): ไฟล์ → mammoth → HTML → htmlparser2 → Document IR — ทำใหม่ทุกครั้งที่นำเข้า

ใหม่:
```
watcher/reconcile เห็นไฟล์ใหม่หรือเปลี่ยน (size, mtime, file_id)
  → extractor ใน ziz-index (ZIP + quick-xml, ไม่สร้าง HTML) → IR-lite:
      [{ unit: "p"|"h1".."h6"|"li"|"cell", style: "หัวเรื่อง 1", text, start, end }]
  → เก็บใน cache ตาม content_hash (zstd) · index BM25 + เวกเตอร์ต่อย่อหน้า
Z.I.Z นำเข้า (Z4) → ขอ IR จาก ziz-index ด้วย content_hash
  → มีแล้ว: ได้ทันที (ไม่เปิดไฟล์) · ไม่มี/daemon ไม่รัน: fallback mammoth เดิม
```
- IR-lite มี style name ของ Word → ตรวจหัวบท (`04` §5.3) ทำต่อได้เหมือนเดิม
- mammoth ยังอยู่สำหรับ: ตาราง/เชิงอรรถ/รูปที่ IR-lite ไม่ครอบ · และเป็น fallback — ponytail: ไม่เขียน extractor ซ้อนให้ครบทุก element จนกว่า fixture จะบอกว่าขาด
- chunk id (`<hash[:8]>:c<NNNN>`) คงที่ข้ามสองทาง เพราะคำนวณจาก IR หลัง normalize

**ผลที่ผู้ใช้เห็น:** เปิดหน้า Ingest ของนิยายที่เคยนำเข้า = ทันที · "นิยายเล่มนี้พูดถึง X ที่ไหนบ้าง" ตอบจาก index ไม่ต้องให้ agent อ่าน ·
แก้ไฟล์ Word ไปบทเดียว → re-extract ทั้งไฟล์ (เร็ว) แต่ diff IR แล้ว**ส่งให้ agent เฉพาะ chunk ที่เปลี่ยน**

---

## 5. ต่อยอดกับระบบอื่นใน Zydraxyl Eco

`ziz-index` คุยผ่าน pipe `\\.\pipe\zydraxyl-index-<SID>` (ACL ผู้ใช้ปัจจุบัน แบบ ECO-C5) + มีโหมด `ziz-index --mcp` (stdio) ให้ agent ของผู้ใช้

| แอป | ใช้ทำอะไร |
|---|---|
| **RyuLong** (ต้นฉบับนิยาย) | ค้นทั้งต้นฉบับขณะเขียน (ภาษาไทยค้นกลางคำได้) · "ฉากที่คล้ายกัน" ด้วยเวกเตอร์ · ตรวจชื่อที่สะกดไม่ตรงกันทั้งเล่ม |
| **BahaMyth** (โลก/ตัวละคร) | "ตัวละครนี้ถูกพูดถึงในต้นฉบับบทไหน" — ลิงก์ entity ↔ ตำแหน่งในไฟล์ |
| **DraconDex** | ค้นเนื้อหาไฟล์แนบใน Page · แทน/เสริมการค้นของ Sage |
| **QuetzaLib** | ค้นเนื้อหาในไฟล์หนังสือ (EPUB/DOCX/PDF ข้อความ) ที่ผู้ใช้มี |
| **LevelthanQ** | ค้นเอกสารออกแบบเกม/สคริปต์เควสต์ |
| **Ourobask** | แนบไฟล์ที่เกี่ยวกับงานให้อัตโนมัติ (ค้นด้วยชื่องาน) |
| **TiaMax** | (ถ้ามี) ค้นบันทึก action |
| **Zydraxyl Hub** | ติดตั้ง/อัปเดต `ziz-index` และโมเดลร่วม (`08` §7.6) |

**tool ของ `ziz-index` (อ่านอย่างเดียวทั้งหมด):** `index_search(q, scope?, k, mode)` · `index_read(nodeId, from, to)` (งบตัวอักษรต่อครั้ง) ·
`index_ir(contentHash)` · `index_similar(text|nodeId, k)` · `index_state()` (ความสด 3 generation แบบ Kensaku: meta/content/vector)

**กฎ:** แอปเห็นเฉพาะ root ของโปรเจกต์ที่ตัวเองเป็นสมาชิก (`members` ใน `.zydraxyl/project.json` — ECO-L7) · ไม่มี tool เขียน · ไม่มี reindex จาก agent ·
ผลเป็นข้อมูล ไม่ใช่คำสั่ง

### 5.1 ข้อเสนอเพิ่มเข้า Eco contract (ต่อจาก P1–P10 ใน `06-eco-integration.md` §8)

| รหัส | ข้อเสนอ |
|---|---|
| **P11** | บริการกลาง `zydraxyl-index`: ชื่อ pipe · รูปแบบคำขอ (JSON-RPC แบบเดียวกับ MCP) · ขอบเขตตาม `members` · แอปไม่ index ไฟล์ของโปรเจกต์เองซ้ำ |
| **P12** | `eco.activity` — log การกระทำแบบไม่มีเนื้อหา (สำหรับ Spaces/รูปแบบซ้ำ — `10` §4.2) |
| **P13** | `catalog.json` ของโมเดลกลาง + ABI — ให้ Hub และแอปอื่นใช้โมเดลชุดเดียวกัน (`08` §5–6) |

---

## 6. งบประสิทธิภาพ (เป้าของ Z.I.Z — ต้องวัดเองบน Windows)

| อะไร | เป้า |
|---|---|
| เปิดโปรเจกต์ → ค้นได้ | < 400 ms (โหลด snapshot) |
| ค้นชื่อ (palette) ต่อการกดแป้น | p50 < 15 ms |
| ค้น full-text ไทยในโปรเจกต์ 2,000 ไฟล์ / นิยาย 3 ล้านตัวอักษร | p50 < 30 ms |
| เปิด Ingest ของไฟล์ที่ index แล้ว | < 200 ms |
| RAM ของ `ziz-index` ตอนว่าง | < 60 MB |

benchmark อยู่ใน repo `ziz-index/bench/` (ยก harness ของ Kensaku `docs/BENCHMARKS.md`) · CI ล้มถ้าช้าลง > 20 % จาก baseline
