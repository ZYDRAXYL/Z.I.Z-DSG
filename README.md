# Z.I.Z

**Z.I.Z** คือแอปวิเคราะห์และจัดการข้อมูลของ **Zydraxyl Eco** บน PC (Windows) —
ต่อยอดระบบ Sage (สถิติ/วิเคราะห์) ของ DraconDex ให้มองเห็นข้อมูลของทุกแอปใน Eco
(DraconDex · BahaMyth · LevelthanQ · RyuLong · TiaMax · Ourobask · QuetzaLib) ในที่เดียว

- ใช้ **CLI agent ของผู้ใช้เอง** (Claude Code / Codex / Gemini CLI / agent ที่พูด ACP) สรุป จัดระเบียบ และแปลงข้อมูล
  — Z.I.Z ไม่เก็บ API key ของใคร · มี**โมเดลเล็กในเครื่อง**ช่วยคัดกรองงานก่อนส่งให้ agent (v0.2 — `Design/08-local-models.md`)
- อ่านนิยายทั้งเล่ม ไฟล์ Word/Excel แล้ว **เสนอ** ข้อมูลให้แอปปลายทาง (เช่น ตัวละคร/สถานที่ลง BahaMyth)
  ผู้ใช้ตรวจ diff ก่อน แล้วจึงเขียนผ่าน MCP ของแอปนั้น — ย้อนกลับได้
- เปิดโฟลเดอร์โปรเจกต์ Eco แล้วเห็นสถิติ ความเชื่อมโยง และรายงานข้ามแอป แม้ไม่มี agent ติดตั้ง

**สถานะ: ออกแบบ ยังไม่สร้าง** (ยังไม่มีโค้ด) · เอกสารออกแบบอยู่ที่ [`Design/`](./Design/README.md)
· สัญญากลางของ Eco: `Zydraxyl-HUB/Design/eco/ECO-CONTRACT.md`

app id `ziz` · format code `ziz` · Windows เท่านั้นใน Eco v1

---

## Repository ที่ต้องสร้าง (multi-repo)

> ร่าง · ยังไม่ได้สร้าง · ตั้งชื่อตามแบบ repo อื่นใน Zydraxyl (`DraconDex-APP` · `-EXE` · `-SDB` · `-PGI-Claude` …):
> `<แอป>-<รหัส 3 ตัว>` · hub = `-APP` · ตัวอย่างหลายตัวของชนิดเดียวกัน = `-<รหัส>-<ชื่อ>` · เกณฑ์ว่าเมื่อไรควรแยก repo: [`Design/02-architecture.md`](./Design/02-architecture.md) §6.1

Z.I.Z มี repo มากกว่าแอป Eco ทั่วไป เพราะนอกจากตัวแอปแล้วยังมีชิ้นที่**ไปอยู่ในแอปอื่น** (Link · Suggestion) และชิ้นที่**แอปอื่นใช้ร่วม** (index · โมเดล · สเปก protocol)

**ตัวแอปและ hub**

| รหัส | Repository | หน้าที่ | ที่มาในเอกสาร |
|---|---|---|---|
| APP | **Z.I.Z-APP** (เปลี่ยนชื่อจาก `Z.I.Z-DSG` นี้) | hub: `Design/` · `chain/chain.json` · `.claude/` · `tools/` — ไม่มีโค้ดแอป | — |
| EXE | **Z.I.Z-EXE** | แอป Windows (Electron + vanilla JS): Home · Mirror · Spaces · Terminal · gateway `zydraxyl-ziz` · ขอบเขตการอ่าน · `ziz-cache` · router | `02` · `10` · `12` §3 · `13` §3 |
| WEB | **Z.I.Z-WEB** | หน้าดาวน์โหลด + release mirror | `02` §9 |

**ใช้ร่วมกับแอป Eco อื่น**

| รหัส | Repository | หน้าที่ | ที่มาในเอกสาร |
|---|---|---|---|
| SPC | **Z.I.Z-SPC** | สเปก + ชุดทดสอบ conformance: `link/1.x` · `scope/1` · `suggest-rules/1` (JSON Schema) — ย้ายเข้าสัญญา Eco เมื่อ P14/P19/P20 ผ่าน | `11` §3.3 · `12` §3, §5 · Q59 ✅ |
| IDX | **Z.I.Z-IDX** | daemon `ziz-index` (Rust, fork จาก Kensaku MIT) อ่านไฟล์เร็ว · tokenizer ไทย · pipe ให้แอปอื่น | `09` · Q30 ✅ |

**ไปอยู่ในแอปอื่น**

| รหัส | Repository | หน้าที่ | ที่มาในเอกสาร |
|---|---|---|---|
| LNK | **Z.I.Z-LNK** | Z.I.Z Link: `ui/` (`<ziz-link-panel>`) + `adapters/<app>/` → `.zizext` ต่อแอป · ติดตั้งผ่าน catalog เมื่อมีทั้งสองฝั่ง | `11` |
| SGT | **Z.I.Z-SGT** | Suggestion ชั้น S0 `ziz-suggest-core` + `rules/<app>/rules.json` → แอปเจ้าบ้าน vendor ตอน build installer | `12` §2.1 · Q56 ✅ · Q60 ✅ |

**โมเดลในเครื่อง** (หนึ่งโมเดล หนึ่ง repo — แบบ `DraconDex-PGI-<ชื่อ>`)

| รหัส | Repository | หน้าที่ | ระดับ |
|---|---|---|---|
| MDL | **Z.I.Z-MDL** | `catalog.json` ที่ลงลายเซ็น (โมเดล + extensions) · ABI · เครื่องมือ pack/verify · eval กลาง | v1 |
| MDL-Embed | **Z.I.Z-MDL-Embed** | เวกเตอร์ความหมาย TH+EN (model2vec) | v1 |
| MDL-NER | **Z.I.Z-MDL-NER** | ชื่อเฉพาะระดับตัวอักษร | v1 |
| MDL-Route | **Z.I.Z-MDL-Route** | คัด chunk · ตรวจคำสั่งแฝง | v1 |
| MDL-Habit | **Z.I.Z-MDL-Habit** | รูปแบบที่ทำซ้ำ (ไม่ใช่ neural) | v1 |
| MDL-Gen | **Z.I.Z-MDL-Gen** | LLM เล็กในเครื่อง (opt-in · ฐานเลือกจาก eval — Q62 ✅) | v1 opt-in |
| MDL-Compress | **Z.I.Z-MDL-Compress** | บีบ prompt แบบ LLMLingua-2 | later (Q61 ✅) — **ยังไม่ต้องสร้าง** |

ทั้งหมด public (Q12 ✅) · licence MIT ยกเว้นที่ต้นทางกำหนด (ระบุใน `THIRD-PARTY.md` ของ repo นั้น)

### Chain

```
แอป      APP > SPC > IDX, MDL > EXE > WEB
Link      APP > SPC > LNK, SGT > MDL (section extensions)          → แอป Eco ติดตั้ง/vendor
โมเดล     APP > MDL-Embed, MDL-NER, MDL-Route, MDL-Habit, MDL-Gen > MDL > EXE
```
การเปลี่ยนไหลซ้ายไปขวา ไม่ย้อนกลับ · รูปแบบที่เครื่องอ่านได้: [`chain/chain.json`](./chain/chain.json) (ร่าง — schema เดียวกับ `DraconDex-APP/chain/chain.json`) ·
ทุก repo พกสำเนาเมื่อสร้างแล้ว

### ลำดับการสร้าง

1. เปลี่ยนชื่อ `Z.I.Z-DSG` → **Z.I.Z-APP** (GitHub redirect ลิงก์เดิมให้เอง)
2. **Z.I.Z-EXE** · **Z.I.Z-SPC** — MVP ต้องมี gateway + ขอบเขตการอ่าน (F97–F98) ซึ่งใช้ `scope/1`
3. **Z.I.Z-IDX** · **Z.I.Z-MDL** + **MDL-Embed / NER / Route / Habit** — v1
4. **Z.I.Z-LNK** · **Z.I.Z-SGT** — v1 (DraconDex ก่อน แล้ว BahaMyth + RyuLong — Q39 ✅)
5. **Z.I.Z-WEB** — ก่อน release แรก
6. **Z.I.Z-MDL-Gen** — เมื่อพร้อม eval ไทย · **MDL-Compress** — later
