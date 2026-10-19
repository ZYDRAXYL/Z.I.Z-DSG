# 01 — สรุปการค้นคว้า

> ที่มา: `Zydraxyl-HUB/Design/eco/research/T1-A-structure.md` §3 · `T2-A-features.md` §ZIZ · `T3-A-internals.md` §6–§9 ·
> `T4-A-uxui.md` Part C/D/F · `T5-eco-research.md` §4, §8 · สัญญา `Zydraxyl-HUB/Design/eco/ECO-CONTRACT.md`
> ป้าย: **[V]** อ่านจากต้นทางแล้ว · **[S]** มาจาก snippet การค้นหา · **(verify)** ต้องตรวจก่อนเขียนโค้ด

---

## 1. ผลิตภัณฑ์และ OSS ที่เทียบ

| ชื่อ | URL | licence · stack | ยืมอะไร | เลี่ยงอะไร |
|---|---|---|---|---|
| **Zed + ACP** (Agent Client Protocol) | https://github.com/agentclientprotocol/agent-client-protocol · https://zed.dev/acp | Apache-2.0 (spec) · JSON-RPC over stdio | วิธีหลักในการขับ CLI agent จาก GUI: `initialize` → `session/new {cwd, mcpServers}` → `session/prompt` · agent ขอสิทธิ์กลับมาที่ client (`session/request_permission`) [V/S] | ผูกกับ adapter ของบุคคลที่สามโดยไม่ pin เวอร์ชัน |
| **acp-ui** | https://github.com/formulahendry/acp-ui | (verify) | ตัวอย่าง client ACP ที่เป็น GUI ล้วน | — |
| **Goose** (Block) | https://github.com/block/goose | Apache-2.0 · Rust | "ทุกอย่างคือ MCP" · ระดับสิทธิ์ต่อ tool (Always / Ask / Never) · โหมด Manual/Smart/Autonomous [V/S] | โหมด autonomous เป็นค่าเริ่มต้น |
| **Cline** | https://github.com/cline/cline | Apache-2.0 · VS Code ext | ทุกการแก้เป็น diff ให้ตรวจ · Plan → Act · checkpoint ย้อนงานของ agent [V] | checkpoint ครอบแค่ไฟล์ — ข้อมูลในแอปต้องมีกลไกย้อนของตัวเอง (T2-A Z19/Z20) |
| **Claude Code** (headless + permission modes) | https://code.claude.com/docs/en/permissions · https://docs.claude.com/en/docs/claude-code/headless | proprietary CLI | `-p --output-format stream-json --mcp-config --strict-mcp-config --allowedTools` · "อ่านไม่ต้องถาม เขียนต้องถาม" [V] | `--dangerously-*` |
| **Codex CLI** | https://github.com/openai/codex | (verify) | `codex exec --json -s read-only` | รายงานว่า MCP tool call ถูกยกเลิกอัตโนมัติในโหมด exec (issue #24135 — verify) → headless ของ Codex ใช้ MCP ไม่ได้ |
| **Gemini CLI** | https://geminicli.com/docs/cli/headless | Apache-2.0 | `--acp` ในตัว (reference impl ของ ACP) · `-p --output-format stream-json` | `trust: true` ข้ามการยืนยันทุกครั้ง · ⚠ หยุดให้บริการผู้ใช้ฟรี/AI Pro ตั้งแต่ 2026-06-18 → Antigravity CLI (ปิด source) — https://www.theregister.com/ai-ml/2026/05/20/bye-bye-gemini-cli-google-nudges-devs-toward-antigravity/5243605 |
| **AnythingLLM** | https://github.com/Mintplex-Labs/anything-llm | MIT · Electron + Node | "collector" แยกเป็นโมดูล แปลงไฟล์ต่อรูปแบบ · workspace = โปรเจกต์ · อ้างอิงแหล่งพร้อมคำตอบ [V] | vector DB ในตัว (Z.I.Z ไม่ต้องการ RAG ใน v1 — ใช้ chunk + tool อ่านแบบมีขอบเขตแทน) |
| **Jan** | https://github.com/janhq/jan | Apache-2.0 · Tauri | หลักฐานว่า UI แบบ webview + token ทำงานดีกับแอป AI desktop | Tauri (Eco เลือก Electron — ตาม coordinator default 2) · โมเดลในตัว |
| **Open WebUI** | https://github.com/open-webui/open-webui | Open WebUI License (มี branding clause) | `#` ดึงเอกสารเข้าบทสนทนา | ห้ามลอกข้อความ/แบรนด์ UI |
| **Cursor** | https://cursor.com/features | proprietary | ยอมรับ/ปฏิเสธรายก้อน + "Keep all / Undo all" [M] | — |
| **GitHub Desktop** | https://github.com/desktop/desktop | MIT · Electron | "รายการเปลี่ยนแปลง → commit" ≈ "ข้อเสนอ → นำไปใช้" | — |
| **Terraform** | https://github.com/hashicorp/terraform | BUSL | plan ก่อน apply — โครงคิดของ propose → apply [V] | — |
| **mammoth.js** | https://github.com/mwilliamson/mammoth.js | BSD-2 (verify) · JS | DOCX → HTML เชิงความหมาย (heading → บท) ผ่าน `styleMap` [V] | ไม่ sanitize เอง → ต้องผ่าน walker ที่รับ tag จำกัด |
| **SheetJS CE** | https://github.com/SheetJS/sheetjs | Apache-2.0 | อ่าน xlsx/xls/csv/ods โดยไม่ต้องมี Office [V] | npm `xlsx` ค้างเวอร์ชันเก่าที่มี CVE → ใช้ tarball จาก cdn.sheetjs.com ที่ pin sha256 (verify) · formula ไม่คำนวณ (ใช้ค่าที่ cache ในไฟล์) |
| **ExcelJS** | https://github.com/exceljs/exceljs | MIT | ทางเลือกสำรองถ้า SheetJS มีปัญหา licence/ที่มา | อ่าน `.xls` เก่าไม่ได้ |
| **MarkItDown** | https://github.com/microsoft/markitdown | (licence ไม่ระบุใน fetch) · Python | แนวคิด "ทุกไฟล์ → Markdown" | Python runtime + licence ไม่ชัด → ไม่ bundle |
| **pdf.js** | https://github.com/mozilla/pdf.js | Apache-2.0 | `getTextContent` (later) | PDF ภาษาไทยมักดึงข้อความเพี้ยน (ลำดับสระ/วรรณยุกต์, ไม่มี ToUnicode) — https://pdfa.org/presentation/support-of-complex-scripts-in-pdf-2/ |
| **Pandoc** | https://github.com/jgm/pandoc | GPL-2+ | — | ไม่ bundle (GPL + ขนาด) · เรียกได้ถ้าผู้ใช้ติดตั้งเอง (later) |
| **LangChain map-reduce** | https://python.langchain.com/docs/how_to/summarize_map_reduce | MIT | split → map → reduce สำหรับข้อความยาว [S] | chunk ขนาดคงที่ไม่สนบท (Z.I.Z ตัดตามย่อหน้าและไม่ข้ามบท) |
| **OnlyWorlds** | (ใน T5 §4.1) | — | บังคับแค่ `Id` + `Name` ใน entity กลาง — ที่มาของ `eco.entity-set` | — |
| **DraconDex Sage / Sage Hut** | `DraconDex-APP/docs/SYSTEMS.md` §8 · `V5.md` §11.9 | ของเจ้าของเอง | 4 มุมมอง: ขนาดข้อมูล · จำนวนรายการ · รายการเชื่อมต่อ · กราฟเชื่อมต่อ → **Insight** ของ Z.I.Z ขยายจาก "Nexus เดียว" เป็น "ทั้งโปรเจกต์ Eco" | Sage อ่าน vault ตรง ๆ (`sage.js`) — Z.I.Z ห้ามอ่านฐานข้อมูลแอปอื่นตรง (หลักการข้อ 3 ของสัญญา) |
| **QuetzaLib name sets** | `QuetzaLib-APP/CLAUDE.md` "Name sets" | ของเจ้าของเอง | กลุ่มชื่อเทียบเท่า (ไลลา / ไลล่า / Laila) → ใช้รวม entity ซ้ำตอน reduce | — |

---

## 2. ข้อสรุปต่อหัวข้อ

### 2.1 โครงสร้างแอป (T1-A §3)

1. **Electron + vanilla JS + token ร่วม** — แอปตระกูลเดียวกับ DraconDex EXE ใช้ pattern เดิมได้ (preload allow-list, CSP, i18n th/en, theme engine) — coordinator default 2
2. **main process เป็นเจ้าของ child process ทั้งหมด** (agent, MCP client ไปหาแอป, ingest worker) — renderer เป็น UI ล้วน
3. **`core/` ไม่ import electron** เพื่อเทสต์แบบ headless ได้ (pattern `core/` vs `ui/` ของ TiaMax Zyd Hub)
4. **ไม่มี vector DB / RAG ใน v1** — งานของ Z.I.Z คือ "สกัดเป็นโครงสร้าง" ไม่ใช่ "ถามตอบเอกสาร"; การอ่านแบบมีขอบเขต
   (`ziz_source_read`) + chunk ตามบทพอ

### 2.2 ฟีเจอร์ (T2-A §ZIZ)

- สิ่งที่ทุกผลิตภัณฑ์อ้างอิงมีร่วมกัน: **อนุมัติก่อนเขียน** · **diff ก่อนเขียน** · **ย้อนได้** · **ระบุว่า agent ตัวไหนทำ**
- ช่องว่างที่ไม่มีใครทำ: **ย้อนการเขียนผ่าน MCP** ไม่มีผลิตภัณฑ์ไหนทำ [NS] → Eco ต้องกำหนดเอง (ECO-S3 + ข้อเสนอ `eco_revert` ใน `06-eco-integration.md` §8)
- จุดอ่อนที่รู้: "อนุญาตเสมอ" ไม่ถูกจำข้าม session ใน Claude Desktop (bug report — T2-A Z10) → การจำสิทธิ์ต้องมีเทสต์ของตัวเอง

### 2.3 ระบบภายใน (T3-A §6–§9)

| หัวข้อ | ข้อสรุป |
|---|---|
| ขับ agent | **ACP ก่อน** (protocol เดียว ได้ event มีโครงสร้าง + คำขอสิทธิ์กลับมาที่ Z.I.Z + `cwd`/`mcpServers` ต่อ session) · **headless สำรอง** ต่อ CLI ที่ไม่มี ACP |
| transport ไปหาแอป | T3-A §7.2 เสนอ HTTP บน 127.0.0.1 แต่ **สัญญาเลือก stdio + named pipe (ECO-C4/C5)** — ทำตามสัญญา เพราะไม่มี listener/port/Origin check และแอปยังเป็นผู้เขียนคนเดียว |
| ข้อความยาว | Z.I.Z ไม่เรียกโมเดลเอง แต่เปิด MCP ของตัวเองให้ agent **ดึง chunk ทีละก้อน** → map → **ตรวจ quote แบบตรงตัว** (ตัวกรองการแต่งเรื่องหลัก) → reduce → propose · งานต่อได้หลัง crash |
| ภาษาไทย | ไม่มีช่องว่างระหว่างคำ → **ตัดที่ย่อหน้า ไม่ตัดที่ประโยค** · normalize (NFC, ลบ U+200B/200C/FEFF, รวมสระอำ) ก่อน hash และก่อนเทียบ quote · `Intl.Segmenter('th')` ให้ผลต่างกันตาม runtime → เทสต์บน Electron ที่ส่งจริง |
| data model | `ziz.db` ต่อ workspace: source/unit/job/chunk/finding/evidence/proposal/applied — source เปลี่ยนไม่ได้, finding ไม่แก้ทับ |

### 2.4 UX (T4-A Part C)

- โมเดลในหัวผู้ใช้: **แหล่ง → ข้อเสนอ → การเขียน** สามคำเท่านั้น
- **Review/Diff สำคัญกว่าแชต** ("chat คือของทั่วไป การเขียนกลับที่ไว้ใจได้คือตัวผลิตภัณฑ์" — T4-A Part G ข้อ 5)
- diff ระดับฟิลด์ (ข้อมูลเป็น record ไม่ใช่บรรทัดข้อความ) + **หลักฐาน (quote) ทุกรายการ** + ความมั่นใจเป็นจุด + ตัวเลข ไม่ใช้สีอย่างเดียว
- ถามต่อ **changeset** ไม่ใช่ต่อการเรียก tool (กันอาการกดอนุญาต 42 ครั้ง)
- ปุ่ม "นำไปใช้" ปุ่มเดียวในแผงขวา · ปุ่มปลอดภัยรับ focus ในกล่องขอสิทธิ์

### 2.5 Eco (T5 §4, §8 + สัญญา)

- Z.I.Z เป็น **MCP client** ของแอปอื่น (ECO §17) · การเขียนทุกครั้ง `eco_propose` → `eco_apply` (ECO-S2) · ผู้ตัดสินคือแอปเจ้าของข้อมูล (ECO §14)
- เนื้อหาที่ผู้ใช้วางอาจมีคำสั่งแฝง → การป้องกันต้องอยู่ฝั่งแอป (S1–S4) **และ** ฝั่ง Z.I.Z (gateway + provenance — `04-internals.md` §7)
- annotations ของ MCP เป็นแค่ hint ที่ปลอมได้ (ECO-S11)

---

## 3. ความเห็นที่ขัดกันในงานวิจัย และที่เลือก

| เรื่อง | ขัดกันอย่างไร | เลือก | เหตุผล 1 บรรทัด |
|---|---|---|---|
| transport ของ MCP แอป | T3-A §7.2: HTTP loopback · T5 + สัญญา: stdio shim + pipe | stdio + pipe | สัญญา ECO-C4/C5 ชนะ และไม่มี port ให้โจมตี |
| ชื่อ tool | T3-C ใช้ `qzl.search_books` (จุด) | `qzl_search_books` (ขีดล่าง) | ECO-C8 กำหนด `<code>_` |
| สีประจำแอป | T4-A F2: hue เฉพาะ tile/stripe ไม่แตะ accent · สัญญา ECO-T2: ต่างกันได้แค่ accent | accent ตาม ECO-T2 (ใช้ค่าบานเย็นที่ T4-A คำนวณ contrast แล้ว) | สัญญาชนะ — ❓ Q28 |
| ใครเป็น MCP host | T1-A §3.1: ให้ agent spawn MCP ของแอปเอง · T4-A: Z.I.Z คุม tier เอง | **gateway**: agent เห็นแค่ MCP ของ Z.I.Z, Z.I.Z เป็น client คนเดียวของแอป | agent ไม่มีทางเรียก `eco_apply`/`eco_delete` เอง — ❓ Q3 |
| ตาราง XLSX | SheetJS CE vs ExcelJS | SheetJS CE (pin tarball) | อ่าน `.xls`/`.csv`/`.ods` ได้ด้วย; ไฟล์ Excel เก่าของผู้ใช้ไทยยังเป็น `.xls` บ่อย |
