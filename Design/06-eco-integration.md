# 06 — การเชื่อมกับ Eco

> ทุกข้ออ้างสัญญา `Zydraxyl-HUB/Design/eco/ECO-CONTRACT.md` ด้วยรหัส · สิ่งที่สัญญายังไม่มีอยู่ใน §8 "ข้อเสนอเพิ่มเข้า Eco contract"
> บทบาทของ Z.I.Z ในตาราง §17 ของสัญญา: ส่งออก/นำเข้า `.zeco` ✅ · MCP = **client** · locator ⬜ · สิทธิ์/snapshot = ไม่ใช่หน้าที่ (เป็นของแอปเจ้าของข้อมูล)

---

## 1. ตัวตนและแคตตาล็อก

| รายการ | ค่า | อ้าง |
|---|---|---|
| app id / code | `ziz` / `ziz` | ECO-I1 |
| format ที่เป็นเจ้าของ | `ziz.report` (part หลักของ `.zeco`) · `ziz.workspace` · `ziz.recipe` (ภายใน ไม่ส่งข้ามแอป) | ECO-I3, ECO-G4 |
| id ของ entity/โปรเจกต์ | UUIDv7 (`eid`, `projectUid` ของ workspace) | ECO-I4 |
| ลิงก์ | `zydraxyl://ziz/<workspaceUid>/<reportUid>` | ECO-I5 |

แถวใน `eco.json` (ECO-K3):
```json
{ "id": "ziz", "code": "ziz", "name": "Z.I.Z",
  "builds": [ { "build": "exe", "platform": "windows-x64", "releaseRepo": "ZYDRAXYL/Z.I.Z-DSG", "tagPattern": "v*",
                "rank": "version", "assets": { "installer": "*Setup*.exe", "portable": "*win-x64.zip" }, "minHub": "1.0.0" } ],
  "formats": {
    "writes": [ { "format": "ziz.report", "versions": ">=1.0.0 <2.0.0" }, { "format": "eco.entity-set", "versions": ">=1.0.0 <2.0.0" },
                { "format": "eco.changeset", "versions": ">=1.0.0 <2.0.0" } ],
    "reads":  [ { "format": "eco.entity-set", "majors": [1] }, { "format": "ziz.report", "majors": [1] },
                { "format": "eco.project", "majors": [1] }, { "format": "eco.locator", "majors": [1] } ] },
  "mcpClient": { "protocols": ["2025-11-25"] }, "ecoProtocol": 1 }
```
- Z.I.Z **ไม่ประกาศ `mcp`** (ไม่ใช่ server ให้แอปอื่น) — gateway `zydraxyl-ziz` เป็นของ session ภายในเท่านั้น
- `mcpClient` เป็นฟิลด์ใหม่ → ข้อเสนอ P8
- Z.I.Z อ่าน format ของแอปอื่น**ผ่าน part `eco.entity-set` เท่านั้น** — ไม่ต้องรู้จัก `baha.world` ฯลฯ (ECO-E1 บอกว่า Z.I.Z ต้องอ่านได้ และ ECO-R9 บังคับ part นี้สำหรับ format ที่ Z.I.Z ต้องอ่าน)
- `releaseRepo` ต้องเป็น repo public (ECO-K4) — ถ้า Z.I.Z-DSG เป็น private ต้องมี mirror แบบ DraconDex-WEB (❓ Q12)

---

## 2. ส่งออก / นำเข้า

| ทิศ | อะไร | กติกา |
|---|---|---|
| ส่งออก | รายงาน → `.zeco` (primary `ziz.report`, part `eco.entity-set` ของ entity ที่อ้าง, rendition HTML/MD/PNG) | ECO-B1–B3 · ECO-R9 (HTML ไม่มี script, CSS จาก token) · ECO-R11 (ไม่มี path/ความลับ — `workspace.root_path` ไม่ออก) |
| ส่งออก | "ส่งเป็น DraconDex Page" → `.zeco` primary `ddx.page` (เฉพาะบล็อกพื้นฐาน: heading/text/table/image/link) + `ziz.report` เป็น part `native` | coordinator default 5 · schema `ddx.page` เป็นของ DDX6 · Z.I.Z vendor schema + fixture จาก repo สัญญา (ECO-R32) — ❓ Q21 |
| นำเข้า | `.zeco` ใด ๆ (ลากวาง/เปิดด้วย) → **เป็นแหล่ง** (อ่าน `eco.entity-set` + `README.md`) ไม่ใช่การนำเข้าเข้าคลังข้อมูล | ECO-C1 ข้อ 1 · ตารางตัดสิน ECO-R15 · พรีวิวก่อน ECO-R16 · rendition เปิดใน sandbox ECO-R21 |
| นำเข้า | `.zeco` ที่ primary เป็น `ziz.report` → เปิดเป็นรายงาน (round-trip ฟิลด์ที่ไม่รู้จัก ECO-R20) | ECO-R14, ECO-R20 |

กรณีของตาราง ECO-R15 ข้อ 3 (ไม่รู้จัก F): Z.I.Z **ไม่ปฏิเสธ** เพราะงานของมันคืออ่าน `eco.entity-set` — ถ้าไม่มี part นั้นจึงแสดงแค่ rendition + "เปิดด้วย <แอป>"

---

## 3. Connect

### 3.1 MCP — Z.I.Z เป็น client (ECO-C4–C8)

| เรื่อง | พฤติกรรม | อ้าง |
|---|---|---|
| เปิดการเชื่อมต่อ | spawn `<App>.exe --mcp` ตาม `eco.json apps[].mcp` · ชื่อ server `zydraxyl-<appId>` | ECO-C4 |
| แอปไม่ได้รัน | shim ของแอปเปิดแอปเองแล้วรอ pipe ≤ 20 วินาที — Z.I.Z แสดง "กำลังเปิด BahaMyth…" · ❓ Q11 | ECO-C5 |
| รุ่น MCP | เทียบ `mcp.protocols` ของแอปกับของ Z.I.Z ก่อน `initialize` | ECO-C4, ECO-G3 |
| tool ที่ใช้ | `eco_info` `eco_list_projects` `eco_search` `eco_get` `eco_export` `eco_propose` `eco_apply` (+ `eco_describe` `eco_stats` `eco_revert` ที่เสนอใน §8) · **ไม่เรียก `eco_delete`** · `eco_import` เฉพาะ "ส่งเป็น DraconDex Page" | ECO-C6 |
| tool เฉพาะแอป | ตาม §4 · ต้องขึ้นต้น `<code>_` | ECO-C8 |
| resource | อ่าน Eco reference เป็น MCP resource เมื่อแอปเปิดให้ | ECO-C7 |
| ไม่เชื่อ hint | การตัดสินว่า tool ไหนเขียนมาจากรายการ tool ที่ Z.I.Z รู้จัก (ตามชื่อ) ไม่ใช่จาก `readOnlyHint` | ECO-S11 |
| ไม่มี listener | Z.I.Z ไม่เปิด port · gateway ใช้ named pipe แบบเดียวกับ ECO-C5 | ECO-S7 |

### 3.2 gateway ของ session (ไม่ใช่ส่วนของสัญญา แต่ใช้กติกาเดียวกัน)

agent เห็นเฉพาะ `zydraxyl-ziz` (`02-architecture.md` §4.1) — pipe `\\.\pipe\zydraxyl-ziz-<SID>`, `FILE_FLAG_FIRST_PIPE_INSTANCE`,
`PIPE_REJECT_REMOTE_CLIENTS`, DACL ผู้ใช้ปัจจุบัน, token ต่อ session (เหมือน ECO-C5) · ไม่มี tool ที่เขียนลงแอป (ECO-S2 ทำได้แข็งแรงกว่า:
agent ไม่มีแม้แต่ช่องทางขอ)

### 3.3 Locator และโปรเจกต์ Eco (ECO-L1–L8)

- **อ่าน** `%LOCALAPPDATA%\Zydraxyl\locator\*.json` + `instances\*.json` เท่านั้น (ECO-L2) · path หาย = แถวจาง + Locate (ECO-L4)
- เปิดโปรเจกต์ Eco = อ่าน `.zydraxyl\project.json` (ECO-L6) → `members` → สร้าง workspace ใน `<EcoProject>\ZIZ\` (❓ Q6)
- เขียน `members.ziz = {projectUid: <workspaceUid>, path: "ZIZ", role: "analysis"}` เท่านั้น — atomic + อ่านซ้ำก่อนเขียน (ECO-L7) · ไม่แตะ `members` ของแอปอื่น
- Z.I.Z **ไม่เขียน locator ของตัวเองใน v1** (§17 ให้ ⬜) — workspace ของ Z.I.Z ไม่ใช่ "โปรเจกต์" ที่ Hub ต้องเปิด; ถ้าเจ้าของอยากให้ Hub แสดง เพิ่มได้ภายหลัง
- `links` ใน `project.json`: Z.I.Z **อ่าน** เพื่อ Insight · เพิ่มลิงก์ได้ (ECO-L7 อนุญาต) เฉพาะเมื่อผู้ใช้กด "บันทึกความเชื่อมโยงนี้" ในรายงานตรวจความสอดคล้อง

### 3.4 Deep link / argv / ไฟล์

- รับ `--eco-open zydraxyl://ziz/<workspaceUid>[/<reportUid>]` + single instance (ECO-C3, ECO-C11)
- ปุ่ม "เปิดใน <แอป>" ส่ง `zydraxyl://<app>/<project>/<eid>` ผ่าน shell → Hub (ECO-C2) · ไม่มี Hub → ปุ่มซ่อน (❓ ECO §16 Q6)
- `.zeco` อยู่ใน `OpenWithProgids` ของ Z.I.Z (ECO-C9) — ไม่ตั้งตัวเองเป็น Default
- clipboard: คัดลอก entity/รายงานเป็น `text/plain` + `text/html` ที่มีลิงก์ Eco reference (ECO-C10)

---

## 4. Recipe ต่อแอป

ทุก recipe: แอป/โปรเจกต์ปลายทางล็อกตอนสร้าง job · ops = create/update/link เท่านั้น · ผ่าน `eco_propose` → `eco_apply`
· รูปฟิลด์มาจาก `eco_describe` ของแอป (ไม่ hardcode — แอปเป็นเจ้าของ schema) · kind ด้านล่างเป็น `kind` ของ `eco.entity-set` ที่ Z.I.Z **คาดว่า**
แอปประกาศ — ชื่อจริงต้องตรงกับเอกสารออกแบบของแอปนั้น

| recipe | ระดับ | input | agent | ขั้น | kind ที่สร้าง | tool ที่ต้องมีจากแอป (นอกชุดมาตรฐาน) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| **novel → BahaMyth** | MVP | วาง/TXT/MD/DOCX นิยาย | ✅ | segment·map·verify·reduce·match·propose | `character` `place` (→ realm) `faction` `event` (→ timeline) `item` + link `in_realm` `rival` `family` … | `baha_realm_tree` (อ่านต้นไม้ realm เพื่อวางสถานที่ใต้ parent) · `baha_timeline_list` | เหตุการณ์ที่ไม่มีวันที่ชัด → ใส่ลำดับตามบท (`order`) ไม่เดาวันที่ · ค่าขัดกันระหว่างบท → ผู้ใช้เลือก (`04` §5.8) |
| **idea → LevelthanQ** | v1 | ข้อความสั้น/ยาว ไอเดียเกม | ✅ | **plan** (agent ถามกลับได้ผ่านแชต) · ผู้ใช้แก้โครง → propose | `game` (VN/RPG) `actor` `item` `skill` `quest` + `quest_step` `variable` (gvar) `scene` `dialogue` (outline) `ending` | `lvq_validate_structure` (ตรวจเงื่อนไข/ตัวแปรที่อ้างไม่มีอยู่) | ไม่มีหลักฐาน quote (ไอเดียไม่ใช่ต้นฉบับ) → ขั้น verify ข้าม แต่ **plan-first บังคับ** · condition ใช้ JSON ของ LVQ (T3-B §3.2) ไม่ให้ agent เขียน expression อิสระ |
| **manuscript → RyuLong** | MVP | DOCX/TXT/MD ต้นฉบับ | ❌ (✅ ถ้าขอ synopsis) | segment·match·propose (+ map สำหรับ synopsis/POV) | `book` · `chapter` · `scene` (content เป็น Markdown ตาม RYU D7) | `ryu_book_outline` (ดูโครงเล่มที่มีอยู่เพื่อเลือก "ต่อท้าย/แทนที่บท") | แบ่งบทแบบ deterministic (`04` §5.3) · ตัวคั่น `***` → scene · ข้อความยาวต่อ op → ยังอยู่ใต้เพดาน 500 entity (ECO-S10) แต่ต้องดูเพดานขนาด → ข้อเสนอ P6 |
| **tasks → Ourobask** | v1 | ข้อความ (บันทึกประชุม/รายการ) · Excel | ✅ ข้อความ · ❌ Excel | map·propose หรือ table-map·propose | `task` (title, notes, due, priority, projectUid) · `note` (ต้องมี project) · `idea` | `our_list_projects` · `our_claim_task`/`our_submit_result` (งานค้าง §5.4 ของ `05`) | วันที่ไทย "15 ต.ค. 69" — ปี 2 หลักตีความเป็น พ.ศ. 25xx (ค่าเริ่มต้น เปลี่ยนได้ใน job) · Note ห้ามไม่มี project (Ourobask CLAUDE.md) |
| **books → QuetzaLib** | MVP | Excel/CSV · ข้อความรายชื่อ/ISBN | ❌ ตาราง · ✅ ข้อความอิสระ | table-map/map·verify (ISBN checksum)·match·propose | `book` (title, authors, isbn, series, readingStatus, categories) | `qzl_search_books` (จับคู่ตาม ISBN ก่อนชื่อ) · `qzl_update_books` (bulk patch) · ธง `lookup` ของ `qzl_add_book` | ISBN ตรวจในเครื่อง (ISBN-10/13 checksum) · ค้นข้อมูลออนไลน์ทำโดย QuetzaLib ไม่ใช่ Z.I.Z (❓ Q23) · bulk edit "เปลี่ยนสถานะเล่ม X–Y" = update ops |
| **pages → DraconDex** | v1 | รายงาน `ziz.report` / สรุปจาก agent | ❌/✅ | report → render `ddx.page` → `eco_import` | Page (บล็อกพื้นฐาน) | ไม่มี — ใช้ `eco_import` (DraconDex แสดงพรีวิว ECO-R16 เอง) | ❓ Q21: ทางเลือกคือ tool `ddx_create_page` แบบ propose/apply |
| **(อ่านอย่างเดียว) TiaMax** | v1 | `.zeco` `tmx.actions` / MCP ถ้ามี | ✅ สรุป | Insight + สรุปเท่านั้น | — | — | TiaMax MCP เป็น "ควร" (ECO-C1) → v1 อ่านผ่าน export · ❓ Q29 |
| **table → ใด ๆ** | MVP | Excel/CSV | ❌ | table-map·propose | kind ใดก็ได้ที่แอปประกาศ | — | recipe ทั่วไปของ "Word/Excel → ข้อมูลแอป" |

### 4.1 ตัวอย่างการไหลของ recipe novel → BahaMyth (ระดับ tool call)

```
Z.I.Z → bahamyth   eco_info · eco_describe → kinds/fields · baha_realm_tree
Z.I.Z → agent      session/prompt: recipe map prompt (ไทย) + "ใช้ ziz_job_next_chunk จนหมด"
agent → gateway    ziz_job_next_chunk(job) → {chunkId, units: <<<SOURCE …>>>}
agent → gateway    ziz_app_search(bahamyth, "อาริน") → มีอยู่แล้ว 1 รายการ
agent → gateway    ziz_finding_submit(job, chunkId, [{kind:"character", name:"เซลีน", fields:{…}, evidence:[{unit:1834, quote:"…"}]}])
   … ×31 chunk …
Z.I.Z              verify (quote ตรงตัว) → reduce (alias) → match (eco_search/eco_get) → changeset
ผู้ใช้              Review → "นำไปใช้ 38"
Z.I.Z → bahamyth   eco_propose(eco.changeset 38 ops) → proposalId · eco_apply(proposalId)
bahamyth           หน้ายืนยันในแอป / batch grant → snapshot → เขียน → audit (ECO-S3, S4) → ใบเสร็จ
```

---

## 5. สิ่งที่แอปอื่นต้องทำให้ Z.I.Z ทำงานได้

| แอป | ชุดมาตรฐาน ECO-C6 | ข้อเสนอ P2–P4 (`eco_stats` `eco_revert` `eco_describe`) | tool เฉพาะที่ recipe ใช้ | ส่งออก `eco.entity-set` |
|---|---|---|---|---|
| DraconDex 6 | ✅ | ✅ | — (ใช้ `eco_import` รับ Page) | ✅ (pages + modules ที่ผู้ใช้เลือก) |
| BahaMyth | ✅ | ✅ | `baha_realm_tree` `baha_timeline_list` | ✅ |
| LevelthanQ | ✅ | ✅ | `lvq_validate_structure` | ✅ |
| RyuLong | ✅ | ✅ | `ryu_book_outline` | ✅ |
| QuetzaLib PC | ✅ | ✅ | `qzl_search_books` `qzl_update_books` (`qzl_add_book{lookup}`) | ✅ |
| Ourobask PC | ✅ | ✅ | `our_list_projects` `our_list_tasks` `our_claim_task` `our_submit_result` | ✅ |
| TiaMax | ⬜ ควร | ⬜ | — | ✅ (`tmx.actions` + `eco.entity-set`) |

ชื่อ tool เฉพาะในตารางเป็น**คำขอจาก Z.I.Z** — เอกสารออกแบบของแอปนั้นเป็นผู้ตัดสินชื่อและรูปจริง (ECO-G4)

---

## 6. ความปลอดภัยและสิทธิ์ (ECO §14) — ฝั่ง Z.I.Z ทำอะไร

| กฎ | สิ่งที่ Z.I.Z ทำ |
|---|---|
| ECO-S1 สิทธิ์ตั้งในแอปเป้าหมาย | Z.I.Z แสดงสิทธิ์ที่ได้จาก `eco_info` แบบอ่านอย่างเดียว + ปุ่มเปิดหน้าสิทธิ์ของแอป · ส่ง `clientInfo.name="Z.I.Z"` ตรงความจริง |
| ECO-S2 เสนอก่อนเขียน | ทุกการเขียน = `eco_propose` → ผู้ใช้กดใน Z.I.Z → `eco_apply` · agent ไม่มีทางเรียก apply (gateway) |
| ECO-S3 snapshot + ย้อน | ใช้ของแอป · Z.I.Z เก็บ `snapshot_id`/`apply_id` ในใบเสร็จ · ปุ่ม "ย้อนชุดนี้" (P3 หรือ deep link) |
| ECO-S4 audit | ฝั่งแอปเขียน · Z.I.Z ส่ง `client.job` ใน `eco.changeset` เพื่อให้ audit โยงกลับมาที่ job ได้ (P7) |
| ECO-S5 ข้อมูลออกนอกเครื่อง | opt-in ต่อโปรเจกต์ (`workspace.ai_allowed`) + ข้อความแจ้งครั้งแรก + บรรทัดใต้ช่องพิมพ์ |
| ECO-S6 ไม่มีความลับ | gateway ส่งต่อเฉพาะผลของ `eco_search`/`eco_get` (ซึ่งแอปกรองแล้ว) · Z.I.Z ไม่เก็บ/ไม่ส่ง credential ใด ๆ (ไม่มี API key) |
| ECO-S10 ≤ 500 ต่อคำสั่ง | แบ่ง proposal อัตโนมัติ |
| ECO-S11 annotations ไม่ใช่การป้องกัน | ✔ (§3.1) |
| เพิ่มเติม | ป้องกัน prompt injection 9 ชั้น — `04-internals.md` §7 |

---

## 7. Conformance — fixture และเทสต์

| เทสต์ | อะไร | อ้าง |
|---|---|---|
| ผู้เขียน | สร้าง `ziz.report` fixture ด้วยเครื่องมือของ Z.I.Z (`tools/build-fixtures.mjs`) → diff + validate JSON Schema | ECO-R28, ECO-R29 |
| ผู้อ่าน | นำเข้า fixture `eco.entity-set` ทุก major ที่ประกาศ + `.zeco` ของทุกแอปที่มีใน repo สัญญา → อ่าน entity ครบ · ฟิลด์ไม่รู้จักถูกข้าม · ไฟล์ "ใหม่กว่า" ได้ผลตาม ECO-R15 | ECO-R30 |
| สำเนาสัญญา | `tools/eco-contract.mjs --check` | ECO-R32 |
| MCP client | ทดสอบกับ **แอปปลอม** (MCP server จำลองที่ทำตามชุด ECO-C6 + P2–P4) ที่ repo สัญญาควรมีให้ทุกแอปใช้ (P9) — ตรวจ propose/apply/revert/ifUpdatedAt/แบ่ง 500 | ECO-C6 |
| ACP | agent ปลอมตามสคริปต์: ปกติ · crash กลางงาน · ส่ง finding ที่ quote ไม่ตรง · ส่ง op นอกขอบเขต · ขอ shell | `04` §7 |
| injection | ชุดไฟล์ทดสอบภาษาไทย/อังกฤษที่มีคำสั่งแฝง → ต้องไม่มี op ที่ไม่ได้ขอผ่านเข้าไปแบบติ๊กไว้ก่อน | `04` §7.3 |
| ภาษาไทย | Windows-874, ZWSP, สระอำสองแบบ, เลขไทยในหัวบท, ย่อหน้ายาวมาก, `Intl.Segmenter` บน Electron ที่ส่งจริง | `04` §5 |

---

## 8. ข้อเสนอเพิ่มเข้า Eco contract

> Z.I.Z ไม่แก้สัญญาเอง — รายการนี้ส่งให้เจ้าของสัญญา (writer ของ HUB) พิจารณา

| # | ข้อเสนอ | ทำไม | กระทบ |
|---|---|---|---|
| **P1** | ลงทะเบียน format **`eco.changeset` 1.0** (`04-internals.md` §3.4): `ops[]` = create/update/link · `eid` ที่ผู้เสนอออก (idempotent) · `ifUpdatedAt` · `evidence[]` optional · `client{app,version,job}` | ECO-C6 บอกแค่ "eco.entity-set patch" — ถ้าไม่กำหนดรูป ทุกแอปจะรับคนละแบบ | ECO-I3, ECO-C6 |
| **P2** | tool มาตรฐาน **`eco_stats`** (readOnly, สิทธิ์ `read`): `{counts: {kind: n}, updatedAt, crossLinks: [{from, to, rel}], sizeBytes?}` | Insight (ต่อยอด Sage) ต้องการตัวเลขโดยไม่ไล่ `eco_search` ทั้งโปรเจกต์ | ECO-C6 ทุกแอป |
| **P3** | tool มาตรฐาน **`eco_revert(applyId)`** (สิทธิ์ `write`, ยืนยันในแอป) + **`eco_apply` คืนใบเสร็จ** `{applyId, snapshotId, created[], updated[], failed[]}` · revert คืน `{reverted[], conflicts[]}` | ECO-S3 มีปุ่มย้อนในแอป แต่ client ไม่มีทางเรียกหรือรู้ผล · ไม่มีผลิตภัณฑ์ใดทำ revert ผ่าน MCP (T2-A Z20) จึงต้องกำหนดเอง | ECO-C6, ECO-S3 · ❓ Q5 |
| **P4** | tool มาตรฐาน **`eco_describe`** (readOnly): JSON Schema ของ kind/ฟิลด์ที่ `eco_propose` รับ + ref kinds + enum (open/closed ตาม ECO-R6) | recipe ต้องรู้รูปปลายทางตอนรันโดยไม่ hardcode | ECO-C6 |
| **P5** | **batch grant ผูกกับ job**: `eco_apply` รับ `grantScope: {client, jobId, maxEntities, expiresAt}` — แอปแสดงหน้ายืนยันครั้งแรกของ job ครั้งเดียว ครั้งต่อไปในงานเดียวกันไม่ถาม (ยกเว้น `delete`) | งานนิยาย 1 เล่ม = หลาย proposal (ทีละ 500) → ถามซ้ำหลายครั้งในงานเดียว | ECO-S1, ECO-S2 · ❓ Q16 |
| **P6** | เพดานขนาด**ไบต์**ต่อ `eco_propose` (เสนอ ≤ 16 MB) นอกเหนือจาก 500 entity | ต้นฉบับ RyuLong ส่งข้อความยาวต่อ op | ECO-S10 |
| **P7** | audit (ECO-S4) เพิ่มฟิลด์ `job` จาก `eco.changeset.client.job` | โยงประวัติในแอปกลับมาที่งานใน Z.I.Z | ECO-S4 |
| **P8** | `eco.json` เพิ่ม `apps[].mcpClient.protocols` สำหรับแอปที่เป็น client | Hub/compat matrix รู้ว่า Z.I.Z เวอร์ชันไหนคุยกับแอปไหนได้ | ECO-K3, ECO-R31 |
| **P9** | repo สัญญาเพิ่ม **แอปปลอม MCP** (`fixtures/mcp-fake-app/`) ที่ทำตามชุดมาตรฐาน | ทุกแอปและ Z.I.Z เทสต์กับตัวเดียวกัน (บทเรียน `TRANSFER.md` §3: logic เดียวหลายภาษาต้องมีชุดทดสอบกลาง) | ECO-R28 |
| **P10** | (later) แอปส่ง recipe ของตัวเองผ่าน **MCP prompts** รูป `ziz.recipe` | แอปรู้ดีที่สุดว่าข้อมูลตัวเองควรสกัดยังไง | ECO-C8 · ❓ Q25 |
