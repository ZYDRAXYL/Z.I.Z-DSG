# 02 — สถาปัตยกรรม

> ตัดสินตาม coordinator default 2 (Electron + vanilla JS + SQLite + Eco token) และสัญญา Eco
> (`Zydraxyl-HUB/Design/eco/ECO-CONTRACT.md`) · ข้อที่ยังไม่ตัดสินชี้ไป `07-grill-me.md`

---

## 1. Stack และเหตุผล

| ชั้น | เลือก | licence | เหตุผล |
|---|---|---|---|
| shell | **Electron** (major เดียวกับ DraconDex EXE ตอน build) | MIT | ใช้ pattern เดิมของ DraconDex (preload allow-list, CSP, single-instance, i18n, theme engine) · `tokens.css` ใช้ได้ทันที (ECO-T3) · Eco v1 = Windows เท่านั้น จึงรับน้ำหนัก Electron ได้ |
| UI | **vanilla JS (ES modules) + `tokens.css`** | — | ตระกูลเดียวกับ DraconDex · ไม่มี framework ให้ตามอัปเดต |
| ฐานข้อมูล | **`node-sqlite3-wasm`** | MIT | ตัวเดียวกับ DraconDex EXE (`V5.md` §8.7) — ทีมเดียวดูแลชุดเดียว · Z.I.Z มี process เขียนคนเดียวจึงไม่เจอปัญหา lock ข้าม process |
| ACP | **`@agentclientprotocol/sdk`** (TypeScript) | Apache-2.0 (verify ชื่อแพ็กเกจ) | ACP เปลี่ยนเร็ว (มีโฟลเดอร์ schema v2 แล้ว — T1-A §3.1) ใช้ SDK ทางการแทนเขียนเอง |
| MCP | **`@modelcontextprotocol/sdk`** (client + server) | MIT | Z.I.Z ใช้ MCP ทั้งสองทิศ และ MCP เปลี่ยนรูประหว่าง revision (ECO-C4) · ⚠ ต้องตั้งค่าเองทุกตัว ห้ามพึ่งค่าเริ่มต้นของ SDK (T5 §4.1) |
| DOCX | **mammoth.js** | BSD-2 (verify) | DOCX → HTML เชิงความหมาย; heading → บท |
| HTML → IR | **htmlparser2** | MIT | mammoth ให้ HTML ชุด tag จำกัด; ingest worker ไม่มี DOM จึงต้องมี parser เล็ก ๆ |
| XLSX/XLS/CSV/ODS | **SheetJS CE** (tarball pin sha256) | Apache-2.0 | `01-research-summary.md` §3 |
| ไม่ใช้ | pandoc · MarkItDown · vector DB · LLM SDK ใด ๆ · local model runtime | — | Z.I.Z ไม่เรียกโมเดลเอง (หลักของเจ้าของ: ใช้ CLI agent ของผู้ใช้) — ❓ Q8 |

dependency ใหม่ทุกตัวต้องผ่าน review licence + pin เวอร์ชัน + บันทึกใน `THIRD-PARTY.md` ตอนเริ่มสร้าง

---

## 2. Process model

```
                        ┌──────────────────────── ZIZ.exe (main = orchestrator) ─────────────────────────┐
                        │  core/jobs · core/changeset · core/insight · core/store (ziz.db)                │
 ผู้ใช้ ⇄ renderer (UI) ⇄│  preload allow-list (ipc/*)                                                     │
                        │                                                                                 │
                        │  ① utilityProcess: ingest-worker  (mammoth · SheetJS · normalize · segment)     │
                        │  ② child: CLI agent  ── ACP JSON-RPC (stdio) ──►  core/agents/acp-driver         │
                        │        │                                                                        │
                        │        └─ child ของ agent: "ZIZ.exe --mcp" (gateway shim, ไม่มี UI)              │
                        │                │ \\.\pipe\zydraxyl-ziz-<SID> + token ต่อ session                 │
                        │                └──────────────────────────────► core/gateway (MCP server logic) │
                        │  ③ child: "<App>.exe --mcp" ต่อแอป (MCP client ของ Z.I.Z, เปิดเมื่อใช้)          │
                        │        └─ pipe ของแอปนั้น (ECO-C5) → แอปจริงที่รันอยู่                            │
                        └─────────────────────────────────────────────────────────────────────────────────┘
```

กฎ:

1. **main เป็นเจ้าของ child process ทุกตัว** · ยกเลิกงาน = ฆ่าทั้งต้นไม้ด้วย Windows Job Object
   (CLI agent spawn shell และ MCP server ลูกอีกทอด — T3-A §6.2)
2. **agent ไม่เคยคุยกับแอป Eco โดยตรงในงานของ Z.I.Z** — เห็นแค่ gateway `zydraxyl-ziz` (§4) · Z.I.Z คือ MCP client คนเดียวของแอป
   (❓ Q3)
3. **renderer ไม่ถือความลับ ไม่ถือ path ดิบ** — เรียกผ่าน `window.ziz.*` ที่ preload เปิดให้เท่านั้น (pattern DraconDex)
4. **ingest ทำใน utilityProcess** — ไฟล์ Word 50 MB หรือนิยาย 1 ล้านตัวอักษรต้องไม่ทำให้ UI ค้าง
5. **`ZIZ.exe --mcp`** ตรวจ argv ก่อนสร้างหน้าต่างและก่อน `requestSingleInstanceLock` → ทำงานเป็น shim แล้วจบ
   (ไม่แย่ง instance หลัก) — โครงเดียวกับ `<app>.exe --mcp` ของ ECO-C5
6. single instance + `--eco-open <uri>` ตาม ECO-C3/ECO-C11

---

## 3. การขับ CLI agent ของผู้ใช้

### 3.1 interface เดียว สอง implementation

```js
// core/agents/driver.js — ponytail: สอง driver พอ ไม่ทำ plugin system
AgentDriver = {
  start({ profile, cwd, mcpServers, systemPrompt }) → Session,
  Session.prompt(contentBlocks)  → AsyncIterable<AgentEvent>,   // message_chunk | tool_call | plan | permission_request | done | error
  Session.cancel(),
  Session.close()
}
```

| driver | ใช้เมื่อ | สิ่งที่ได้ |
|---|---|---|
| **`acp`** (หลัก) | agent มีโหมด ACP (ในตัวหรือผ่าน adapter) | event มีโครงสร้าง · คำขอสิทธิ์กลับมาที่ Z.I.Z · ส่ง `mcpServers` ต่อ session · ยกเลิกกลางทางได้ |
| **`headless`** (สำรอง) | ไม่มี ACP / adapter พัง | spawn `-p` + อ่าน NDJSON · มี **profile ต่อ CLI** (args template + parser + `mcp: true/false`) |

`headless` มีสองระดับ: **with-tools** (ส่ง `--mcp-config` ให้ใช้ gateway ได้) และ **prompt-only** (ไม่มี tool เลย: Z.I.Z ใส่ chunk
ลงใน prompt แล้วขอผลเป็น JSON ตาม schema ในข้อความสุดท้าย — ทำงานได้กับทุก CLI ที่มี `-p` แต่ช้ากว่าและ agent มองไม่เห็นข้อมูลเดิมในแอป)

### 3.2 CLI ที่รองรับ (v1 — ❓ Q2)

| agent | โหมดหลัก | คำสั่ง (ระดับออกแบบ — verify ทุกบรรทัดก่อนสร้าง) | สำรอง | หมายเหตุ |
|---|---|---|---|---|
| **Claude Code** | ACP ผ่าน adapter | `node <bundled>/claude-agent-acp/index.js` รันด้วย `ELECTRON_RUN_AS_NODE=1` + `process.execPath` (ไม่ต้องมี Node ในเครื่อง) | headless with-tools: `claude -p --output-format stream-json --input-format stream-json --mcp-config <tmp.json> --strict-mcp-config --allowedTools "mcp__zydraxyl-ziz__*"` | adapter `@agentclientprotocol/claude-agent-acp` (เดิม `@zed-industries/claude-code-acp`) ใช้ login ของ Claude Code ที่ผู้ใช้มีอยู่ — ❓ Q13 การ bundle |
| **Codex** | ACP ผ่าน `codex-acp` | binary/แพ็กเกจของ adapter (ชื่อต่างกันตามแหล่ง — verify) | headless **prompt-only**: `codex exec --json -s read-only -C <scratch>` | MCP ในโหมด exec ถูกยกเลิกอัตโนมัติ (openai/codex#24135 — verify) → ❓ Q14 |
| **Gemini CLI** | ACP ในตัว: `gemini --acp` | — | headless: `gemini -p … --output-format stream-json` | ผู้ใช้ฟรี/AI Pro ใช้ไม่ได้แล้วตั้งแต่ 2026-06-18 (เหลือ enterprise/API key) → "รองรับแบบพยายาม" |
| **agent อื่นที่พูด ACP** | ACP | ผู้ใช้กรอกคำสั่งเอง (ขั้นสูง) | — | เช่น Goose ที่ต่อ Ollama ได้ → ทางของผู้ใช้ที่ต้องการโมเดลในเครื่อง |

**ห้ามเด็ดขาดในทุก driver:** `--dangerously-*` · `--yolo` · `trust: true` · `--dangerously-bypass-approvals-and-sandbox`
(ความปลอดภัยของการเขียนมาจาก gateway + แอปปลายทาง ไม่ใช่จากการยืนยันของ agent — T3-A §6.2, ECO §14)

### 3.3 การหา agent ในเครื่อง

1. probe `where claude` · `where codex` · `where gemini` + ตำแหน่งที่ติดตั้งบ่อย (`%APPDATA%\npm\`, `%USERPROFILE%\.local\bin\`)
2. `<cli> --version` (timeout 10 วินาที) → บันทึกลง `agent_profile`
3. ปุ่ม "ทดสอบการเชื่อมต่อ": เปิด session จริง ส่ง prompt "ตอบคำว่า OK เท่านั้น" timeout 90 วินาที → วัดเวลา ·
   แยก error เป็นภาษาคน: ยังไม่ login (`ให้เปิด terminal แล้วรัน claude หนึ่งครั้งเพื่อ login`) · adapter ไม่เข้ากับเวอร์ชัน CLI ·
   เน็ตไม่ต่อ · quota หมด
4. ผู้ใช้ระบุ path เองได้เสมอ · Z.I.Z **ไม่ติดตั้ง agent ให้** (ลิงก์ไปเอกสารทางการเท่านั้น)

### 3.4 ไม่มี agent ติดตั้ง — Z.I.Z ยังใช้ได้ (❓ Q1)

| ใช้ได้โดยไม่มี agent | ต้องมี agent |
|---|---|
| **Insight** (สถิติ/กราฟเชื่อมต่อข้ามแอป — `03-features.md` F20–F24) | สรุปนิยาย / สกัดตัวละคร / แปลงไอเดียเป็นโครงเกม |
| นำเข้า **Excel → รายการ** ด้วยการจับคู่คอลัมน์เอง (recipe `table→*`) | เดาการจับคู่คอลัมน์ให้ |
| แตก **ต้นฉบับ Word/TXT → book/chapter/scene** ของ RyuLong ด้วยการตรวจหัวบท (deterministic) | เขียน synopsis ต่อบท |
| ตรวจ ISBN + นำเข้ารายการหนังสือเข้า QuetzaLib | — |
| ตรวจข้อเสนอ · นำไปใช้ · ย้อนกลับ · ส่งออกรายงาน `ziz.report` | รายงานที่เป็นข้อความสรุป |

UI: ฟีเจอร์ที่ต้องมี agent แสดงเป็นการ์ดจางพร้อมป้าย "ต้องมี CLI agent" + ปุ่ม "วิธีติดตั้ง" — ไม่ซ่อน (ผู้ใช้ต้องรู้ว่ามีอยู่)

### 3.5 วงจรชีวิต session แบบ ACP

```
initialize   { protocolVersion, clientCapabilities: { fs: { readTextFile:false, writeTextFile:false }, terminal:false } }
             ← agentCapabilities (loadSession?, promptCapabilities …) — บันทึกลง agent_profile.capabilities
session/new  { cwd: <workspace>/scratch/<sessionId>,  mcpServers: [ gateway เท่านั้น (§4.1) ] }
session/prompt [ text: recipe prompt + คำสั่งผู้ใช้,  resource_link: ziz://source/<id> ]
  ← session/update: agent_message_chunk | tool_call | tool_call_update | plan
  ← session/request_permission  → นโยบาย §3.6
session/cancel  (ปุ่ม "หยุด") · session/load ถ้า agent รองรับ (ทำงานต่อหลังปิดแอป)
```

- `cwd` = **โฟลเดอร์ scratch ว่าง** ของ session ไม่ใช่โฟลเดอร์โปรเจกต์จริง — tool ไฟล์ในตัวของ agent จึงไม่เห็นข้อมูลผู้ใช้
  นอกจากที่ gateway ยื่นให้ (ปิดช่องให้ prompt injection สั่งอ่าน/ส่งไฟล์อื่น)
- client ประกาศ `fs` และ `terminal` เป็น false → agent ขอให้ Z.I.Z อ่าน/เขียนไฟล์หรือรันคำสั่งแทนไม่ได้
- ทุก event เขียนลง `runs/<runId>.ndjson` (audit/replay — `04-internals.md` §2)

### 3.6 นโยบายสิทธิ์ของ agent ภายใน Z.I.Z

agent มี tool ในตัว (shell, แก้ไฟล์, web fetch) ซึ่ง Z.I.Z คุมไม่ได้โดยตรง แต่ ACP ส่งคำขอสิทธิ์กลับมาที่ Z.I.Z:

| คำขอ | ค่าเริ่มต้น | ผู้ใช้เปลี่ยนได้ |
|---|---|---|
| tool ของ `zydraxyl-ziz` ที่อ่าน/เสนอ (ทุกตัวใน §4.1) | อนุญาตเงียบ (ไม่ใช่การเขียน — T4-A C3.2 T0/T1) | — |
| อ่านไฟล์ใน `cwd` (scratch) | อนุญาต | — |
| shell / รันคำสั่ง | **ปฏิเสธอัตโนมัติ** + บันทึก | ไม่ได้ใน v1 |
| web fetch / web search | **ปฏิเสธอัตโนมัติ** | ต่อ session (เช่น recipe ที่ต้องค้นข้อมูลหนังสือ) — แสดงป้ายเตือน |
| แก้/สร้างไฟล์ | ปฏิเสธ (ไม่มีอะไรให้เขียนใน scratch) | — |
| อื่น ๆ ที่ไม่รู้จัก | ถามผู้ใช้ (กล่อง Z8 ใน `05-ux-ui.md`) ปุ่ม focus = "ปฏิเสธ" | — |

headless ใช้ allow-list ที่มีผลเทียบเท่า (`--allowedTools` เฉพาะ gateway · Codex `-s read-only`)

---

## 4. MCP: gateway (Z.I.Z เป็น server) และ client (Z.I.Z ไปหาแอป)

### 4.1 gateway `zydraxyl-ziz` — สิ่งเดียวที่ agent เห็น

ส่งเข้า `session/new.mcpServers`:
```json
{ "name": "zydraxyl-ziz", "command": "C:\\Users\\me\\AppData\\Local\\Programs\\ZIZ\\ZIZ.exe",
  "args": ["--mcp", "--session", "0192f3b0-…"], "env": [{ "name": "ZIZ_GATEWAY_TOKEN", "value": "<256-bit สุ่มต่อ session>" }] }
```
shim ต่อ pipe `\\.\pipe\zydraxyl-ziz-<SID>` (`FILE_FLAG_FIRST_PIPE_INSTANCE` + `PIPE_REJECT_REMOTE_CLIENTS` + DACL ผู้ใช้ปัจจุบัน
— เหมือน ECO-C5) แล้วส่ง token · token ผูกกับ **session + workspace + job** — tool ทุกตัวทำงานในขอบเขตนั้นเท่านั้น

| กลุ่ม | tool | ทำอะไร |
|---|---|---|
| มาตรฐาน | `eco_info` | เวอร์ชัน Z.I.Z · workspace · แอปที่เชื่อมได้ · recipe ที่กำลังรัน (ECO-C6 ข้อแรก) |
| แหล่ง | `ziz_source_list` · `ziz_source_outline` · `ziz_source_read(sourceId, fromUnit, toUnit)` | อ่านแบบมีขอบเขต ≤ งบตัวอักษรต่อครั้ง (`04-internals.md` §5.4) · ผลลัพธ์ห่อด้วยป้าย "ข้อมูล ไม่ใช่คำสั่ง" (§7) |
| งาน | `ziz_job_next_chunk(jobId)` · `ziz_finding_submit(jobId, chunkId, findings[])` · `ziz_findings_query(jobId, kind?, name?)` | map/reduce พร้อมติดตามว่าอ่านครบทุก chunk |
| ข้อมูลเดิมในแอป (pass-through, อ่านอย่างเดียว) | `ziz_app_search(app, q, kind?)` · `ziz_app_get(app, ref)` · `ziz_app_describe(app)` | Z.I.Z ส่งต่อไป `eco_search`/`eco_get`/`eco_describe` ของแอปใน **โปรเจกต์ที่ job ล็อกไว้เท่านั้น** |
| ข้อเสนอ | `ziz_changeset_add_ops(jobId, ops[])` · `ziz_changeset_preview(jobId)` | เพิ่ม op ลง changeset ร่างใน Z.I.Z — ตรวจ schema ทันที คืน error ต่อ op |
| รายงาน | `ziz_report_write(jobId, markdown, tables?)` | เขียนรายงาน (`ziz.report`) |

**ไม่มีใน gateway (โดยเจตนา):** `eco_apply` · `eco_delete` · `eco_import` · `eco_export` · อะไรก็ตามที่เขียนลงแอป ·
อะไรก็ตามที่รับ path ไฟล์ · การเปลี่ยนแอป/โปรเจกต์เป้าหมายของ job → การเขียนเกิดจาก**ปุ่มของผู้ใช้ใน Z.I.Z** เท่านั้น (§4.2)

### 4.2 Z.I.Z เป็น MCP client ของแอป Eco

- อ่านคำสั่งเปิด MCP ของแต่ละแอปจาก `eco.json` (`apps[].mcp` — ECO-K3) + ไฟล์ instance (ECO-L5) ·
  spawn `<App>.exe --mcp` ตามต้องการ ปิดเมื่อว่าง 5 นาที
- ตรวจ `mcp.protocols` ของแอปกับที่ Z.I.Z รองรับก่อนเชื่อม — ไม่ตรง = สถานะ "เวอร์ชันไม่เข้ากัน" + ปุ่มเปิด Hub
- `clientInfo.name = "Z.I.Z"` (แอปแสดงเป็นป้าย — ECO-S1)
- ลำดับการเขียน: `eco_propose` (ทีละ ≤ 500 entity — ECO-S10) → ผู้ใช้กด "นำไปใช้" ใน Z.I.Z → `eco_apply(proposalId)` →
  แอปแสดงยืนยัน/ใช้ batch grant (ECO-S2) → snapshot (ECO-S3) → เขียน → ใบเสร็จ (`apply_receipt`) — รายละเอียด `04-internals.md` §6

---

## 5. ชั้น (layers)

```
ui/        renderer: views, components (ยกจาก DraconDex), i18n th/en      ← รู้จักแค่ window.ziz.*
ipc/       main: แปลงคำขอ UI → core (allow-list ต่อ channel)
core/      Node ล้วน ห้าม import 'electron' — เทสต์ headless ได้
  agents/      driver.js · acp-driver.js · headless-driver.js · profiles/{claude,codex,gemini}.json · detect.js
  gateway/     server.js (tool ใน §4.1) · pipe.js · tokens.js
  eco/         catalog.js (อ่าน eco.json ที่ vendor/ที่ Hub cache) · locator.js · project.js · client.js (MCP client ต่อแอป)
  ingest/      worker.js · docx.js · xlsx.js · text.js · normalize-th.js · segment.js · ir.js
  jobs/        queue.js · map.js · verify.js · reduce.js · resume.js
  changeset/   build.js · validate.js · diff.js · apply.js · undo.js
  recipes/     loader.js · builtin/*.json (`06-eco-integration.md` §4)
  insight/     stats.js · graph.js · checks.js (ต่อยอด Sage)
  zeco/        read.js · write.js (`.zeco` ตาม ECO-B1–B3 · ใช้ validator ที่ vendor มา)
  store/       db.js · migrations/NNNN-*.sql
platform/  win: job object, pipe ACL, where-probe, shell.openPath
```

ทิศพึ่งพา: `ui → ipc → core → platform` เท่านั้น · `core/gateway` เรียก `core/eco/client` ได้แต่**ไม่เรียก** `core/changeset/apply`

---

## 6. โครงโฟลเดอร์ repo

```
Z.I.Z-DSG/                      (❓ Q12: repo เดียว)
  README.md  Design/
  electron/  main.js · preload.js · ipc/
  core/      (ตาม §5)
  ui/        index.html · views/ · components/ · i18n/{th,en}.json · css/
  vendor/eco/                    ← จาก repo สัญญา (ECO-R32): schemas · validator · fixtures · tokens.css · eco.lock.json — ห้ามแก้มือ
  vendor/dracondex/              ← ของที่ยกมาจาก DraconDex แบบ copy-then-own + PROVENANCE.json (§8)
  tools/     eco-contract.mjs (vendor) · build-fixtures.mjs
  tests/     core/** (node:test) · fixtures/ingest/** (ไฟล์ทดสอบภาษาไทย) · e2e/ (driver ปลอม)
  .github/workflows/  ci.yml · release.yml
```

## 7. ข้อมูลขณะรัน (Windows)

| ที่ | ของ |
|---|---|
| `%LOCALAPPDATA%\ZIZ\ziz-global.db` | agent profiles · workspace ล่าสุด · การตั้งค่า · กฎสิทธิ์ของ agent (§3.6) |
| `%LOCALAPPDATA%\ZIZ\workspaces\<uid>\` | workspace ที่**ไม่ใช่**โปรเจกต์ Eco (เช่นเปิดโฟลเดอร์ธรรมดา) |
| `<EcoProject>\ZIZ\` | workspace ของโปรเจกต์ Eco — `ziz.db` · `sources/` · `runs/` · `reports/` · `scratch/` (❓ Q6) · ลงทะเบียนเป็น `members.ziz` ใน `.zydraxyl/project.json` (ECO-L7) |
| `%LOCALAPPDATA%\Zydraxyl\` | **อ่านอย่างเดียว**: `locator\*.json` · `instances\*.json` · cache `eco.json` ของ Hub (ECO-L2) |

ข้อมูลของ Z.I.Z เองไม่ใส่ใน `%LOCALAPPDATA%\Zydraxyl\` (โฟลเดอร์นั้นเป็นของ locator/Hub — และยังชนกับ TiaMax ตาม ECO §15 #2)

---

## 8. ของที่ยกมาจาก DraconDex (copy-then-own — coordinator default 4)

| ยกอะไร | จากไฟล์ใน DraconDex-EXE (`electron/…`) | ทำไม | เปลี่ยนอะไร |
|---|---|---|---|
| มุมมอง Sage Hut 4 แบบ (dataSize · objectAmount · linkerList · linkerGraph) + ตัววาดกราฟ | `src/db/sage.js` (รูปข้อมูล) · `mod/sagehut.js` (มุมมอง) | "ต่อยอดระบบ Analysis เดิม" ตามที่เจ้าของสั่ง | แหล่งข้อมูลเปลี่ยนจาก SQL ตรงใน vault → `eco_stats`/`eco_search` ผ่าน MCP · โหนด = entity ข้ามแอป (สีตามแอป ไม่ใช่ตาม module) |
| component กลาง: list · popover · modal · toast+Undo · chip · segmented · emptyState | `core/list.js` · component ใน `SYSTEMS.md` §10.14–10.15 · `COMPONENTS.md` | หน้าตาและพฤติกรรมเหมือน DraconDex (กฎ Eco ข้อ 2) | ตัดส่วนที่ผูกกับ Nexus |
| preload allow-list · CSP · single-instance · i18n loader · theme engine | `preload.js` · `main.js` · ระบบธีม | ความปลอดภัยที่ผ่านการ harden แล้ว (`SYSTEMS.md` §10.9) | ช่อง IPC ของ Z.I.Z เอง |
| command palette (`Ctrl+P`) | quick switcher (`SYSTEMS.md` §2d) | "ทุกคำสั่งหาเจอใน palette" (UX-LAYOUT §5) | รายการคำสั่งของ Z.I.Z |

**token ไม่ใช่การ copy** — มาจาก repo สัญญาแบบ byte-identical (ECO-T1, ECO-R32)

`vendor/dracondex/PROVENANCE.json` (บังคับ):
```json
{ "source": "ZYDRAXYL/DraconDex-EXE", "commit": "<sha ตอนยก>", "copiedAt": "2027-…",
  "files": [ { "from": "electron/mod/sagehut.js", "to": "ui/views/insight/sagehut-views.js", "changed": "data source → MCP" } ],
  "policy": "copy-then-own: ไม่ sync อัตโนมัติ; บั๊กที่แก้ฝั่ง DraconDex ต้องยกมาเองโดยอ่าน diff" }
```
⚠ DraconDex-EXE เป็น repo private — ถ้า Z.I.Z เป็น public การยกโค้ดคือการเปิดเผยโค้ดส่วนนั้น (เจ้าของคนเดียวกัน จึงทำได้ แต่ต้องตั้งใจ) — ❓ Q12

---

## 9. Build และ release

- tag `v*` → workflow สร้าง `ZIZ-Setup-<v>.exe` (NSIS) + `ZIZ-<v>-win-x64.zip` (portable สำหรับ Hub ติดตั้งหลายเวอร์ชัน — ECO-K7) + `checksums-sha256.txt`
- `eco.json`: `{id:"ziz", code:"ziz", builds:[{build:"exe", platform:"windows-x64", tagPattern:"v*", …}], formats:{writes:["ziz.report"], reads:[ทุก format ผ่าน part eco.entity-set]}, mcp: ไม่มี (Z.I.Z ไม่ใช่แอปเจ้าของข้อมูลที่ให้คนอื่นเขียน)}` — รายละเอียด `06-eco-integration.md` §1
- CI: `node --test` (core) · `tools/eco-contract.mjs --check` (ECO-R32) · fixture ผู้อ่าน (ECO-R30) · e2e ด้วย agent ปลอม (ACP server จำลองที่ตอบตามสคริปต์) — ไม่เรียก agent จริงใน CI
