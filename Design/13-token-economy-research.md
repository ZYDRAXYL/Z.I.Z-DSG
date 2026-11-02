# 13 — งานวิจัย: Assistant ฟรี/ถูก บน CLI agent ของผู้ใช้ · ประหยัดโทเคน

> ร่าง v0.5 · 2026-11-02 · เจ้าของสั่ง: เป้าหมายของ Z.I.Z คือ **assistant ที่ใช้ฟรีหรือถูก โดยใช้ CLI agent ของผู้ใช้ และประหยัดโทเคนให้ผู้ใช้** —
> ค้นทั้ง paper และ open source ที่ช่วยได้ · ต่อจาก `08` (โมเดลเล็ก) · `09` (`ziz-index`) · `12` (Suggestion · ขอบเขต) · ❓ ใหม่อยู่ `07` รอบ 11
>
> ⚠ ตัวเลขทุกตัวในเอกสารนี้เป็น **ผลที่ผู้เขียน paper/โครงการรายงานเอง บนชุดทดสอบของเขา** ไม่ใช่ผลที่ Z.I.Z วัด · licence ที่มาจากแหล่งรองติด **(verify)** · ต้องวัดซ้ำบน fixture ภาษาไทยของเรา (`08` §2.1) ก่อนอ้าง

---

## 1. สรุปสั้น — 6 ชั้นของการประหยัด

โทเคนที่ผู้ใช้จ่าย ≈ (จำนวนครั้งที่เรียก agent) × (โทเคนเข้าที่ไม่ได้ cache + โทเคนเข้าที่ cache × ~0.1 + โทเคนออก) ·
ทุกชั้นข้างล่างลดตัวใดตัวหนึ่งในสูตรนี้ เรียงจาก **ถูกที่สุด → แพงที่สุด** · Z.I.Z ลองชั้นบนก่อนเสมอ

| ชั้น | หลัก | ลดอะไร | สถานะใน Z.I.Z |
|---|---|---|---|
| **E0 ไม่เรียกเลย** | ตอบด้วยของในเครื่อง: index · กฎ · โมเดลเล็ก · cache | จำนวนครั้ง | มีแล้ว (`08` · `09` · `11` §4 · `12` S0/S1) — **เพิ่ม cache คำตอบ §3.1** |
| **E1 เรียกตัวที่ถูกกว่า** | route งานง่ายไป CLI/โมเดลที่ถูกหรือฟรี · งานยากเท่านั้นที่ไปตัวแพง | ราคาต่อโทเคน | **ใหม่ §3.2** |
| **E2 ส่งให้น้อยลง** | ขอบเขต · outline ก่อนเนื้อหา · ตัด chunk · บีบ prompt · ซ่อนผล tool เก่า | โทเคนเข้า | บางส่วน (`12` §3 · `08` route) — **เพิ่ม §3.3–3.5** |
| **E3 ให้ cache ทำงาน** | จัด prompt ให้ส่วนที่ไม่เปลี่ยนอยู่หน้าสุด · ไม่แก้ tool definition กลาง session | โทเคนเข้าที่ไม่ได้ cache | **ใหม่ §3.6** |
| **E4 ให้ตอบสั้นลง** | ผลเป็น JSON/op แทนร้อยแก้ว · draft สั้นแทน CoT ยาว · effort ต่ำสำหรับงานจัดกอง | โทเคนออก | บางส่วน (changeset เป็น op อยู่แล้ว) — **เพิ่ม §3.7** |
| **E5 ตรวจแทนการทำซ้ำ** | ตรวจ quote/ref แบบ deterministic แทนการถาม agent ซ้ำ · ตรวจเฉพาะส่วนที่ไม่ผ่าน | จำนวนครั้ง (รอบแก้) + hallucination | มี (`04` §5.7) — **เพิ่ม §3.8** |

---

## 2. สิ่งที่ค้นเจอ (paper + open source)

### 2.1 Routing / cascade — เรียกตัวถูกก่อน (E1)

| ชื่อ | ชนิด | สาระ | licence | Z.I.Z ใช้อย่างไร |
|---|---|---|---|---|
| **FrugalGPT** (Chen, Zaharia, Zou 2023 · arXiv 2305.05176) | paper | 3 กลยุทธ์: ปรับ prompt ให้สั้น · ใช้โมเดลถูกเลียนแบบโมเดลแพงเฉพาะงาน · **cascade** ถามตัวถูกก่อนแล้วยกระดับเมื่อไม่มั่นใจ · รายงานว่าเทียบ GPT-4 ได้โดยลดค่าใช้จ่ายสูงสุด 98 % (บนชุดของเขา) | — | กรอบคิดหลักของทั้งเอกสารนี้ · router ของ `08` §3 คือ cascade แบบ deterministic อยู่แล้ว |
| **RouteLLM** (LMSYS/Berkeley 2024 · arXiv 2406.18665 · `lm-sys/RouteLLM`) | paper + OSS | train router จาก preference data ให้เลือก "ตัวแรง/ตัวอ่อน" ต่อคำถาม · รายงานว่าประหยัด > 2× บน benchmark ทั่วไป | Apache-2.0 (verify) | **ยืมแนวคิด ไม่ยกโค้ด**: router ของเราเลือกระหว่าง *ชั้น* (E0 local → CLI ถูก → CLI แพง) ด้วยกฎ + คะแนน `route` (§3.2) · ponytail: ไม่ train router ใน v1 |

### 2.2 ส่งให้น้อยลง — บีบ/คัด context (E2)

| ชื่อ | ชนิด | สาระ | licence | Z.I.Z ใช้อย่างไร |
|---|---|---|---|---|
| **LLMLingua / LongLLMLingua / LLMLingua-2** (Microsoft · arXiv 2403.12968 · `microsoft/LLMLingua`) | paper + OSS | บีบ prompt โดยตัด token ที่ข้อมูลน้อย · v2 = token classifier เล็ก (distill จาก GPT-4) ทำงานไม่ขึ้นกับงาน · มี port JS (`@atjsh/llmlingua-2`) | โค้ด MIT · demo Space บน HF ติด CC-BY-NC-SA — **น้ำหนักโมเดลต้อง verify** | ผู้สมัครโมเดล `ZIZ-M-compress` (§3.4, ❓ Q61) · ต้องวัดกับไทย — classifier ฝึกบนภาษาอังกฤษเป็นหลัก **และห้ามบีบข้อความที่จะใช้เป็น quote** (ตรวจ quote ต้องเทียบกับต้นฉบับ) |
| **Aider repo map** (`Aider-AI/aider` · `repomap.py`) | OSS | สร้าง "แผนที่" ของ repo ด้วย tree-sitter + PageRank ของสัญลักษณ์ แล้วตัดให้พอดีงบโทเคน (`--map-tokens` ค่าเริ่ม 1,024) | Apache-2.0 (verify) | **ต้นแบบของ `ziz_scope_outline`** (`12` §3.4): แทนสัญลักษณ์โค้ดด้วย entity/บท/relation · จัดอันดับด้วยจำนวนลิงก์ข้ามแอป · ตัดให้พอดีงบด้วย binary search |
| **Agentless** (Xia et al. 2024 · arXiv 2407.01489) | paper + OSS | pipeline ตายตัว localize → repair → validate แทน agent ที่เลือกทางเอง · รายงานว่าถูกกว่า agent อื่นมากบน SWE-bench Lite | — (verify) | ยืนยันแนวทางที่ Z.I.Z ใช้อยู่แล้ว: **recipe เป็นขั้นตายตัว** (`04` §3.3) · agent ทำเฉพาะขั้นที่ต้องเขียนเนื้อหา ไม่เดินสำรวจเอง |
| **"The Complexity Trap"** (arXiv 2508.21433) | paper | ใน SWE-agent การ **ซ่อนผล observation เก่า** (masking) ลดค่าใช้จ่ายราวครึ่งหนึ่ง และได้ผลเท่าการให้ LLM สรุปประวัติ | — | ทำใน gateway (§3.5): ผล tool ที่เก่ากว่า N รอบถูกแทนด้วย `‹ผลเดิม: ziz_source_read ch.3–4 · ดูซ้ำได้›` — ไม่ต้องมีโมเดลสรุป |
| **OpenHands context condenser** (`OpenHands` SDK · `LLMSummarizingCondenser`) | OSS | สรุปประวัติเมื่อยาวเกิน · ใช้ LLM ที่ถูกกว่าเป็นตัวสรุปได้ · blog ของโครงการรายงานว่าลดค่าใช้จ่ายต่อ turn ได้ถึง 2× | MIT (verify) | ไม่ยกมา — ใช้ masking (ข้างบน) ก่อน เพราะถูกกว่าและไม่เรียกโมเดลเพิ่ม · ถ้าต้องสรุปจริงใช้ `ZIZ-M-gen` ในเครื่อง |
| **RAG-MCP** (Gan & Sun 2025 · arXiv 2505.03275) | paper | เก็บคำอธิบาย tool ใน index แล้วส่งเฉพาะที่เกี่ยว · รายงาน: prompt token ลดราวครึ่ง และเลือก tool ถูกบ่อยขึ้นมาก | — | gateway ของเรามี tool ไม่กี่ตัว จึงยังไม่จำเป็น · **แต่** ประกาศเฉพาะ tool ของ recipe นั้น (§3.6) · Claude Code เองโหลด MCP tool แบบ deferred อยู่แล้ว |
| **Lost in the Middle** (Liu et al. 2023 · TACL 2024 · arXiv 2307.03172) | paper | โมเดลใช้ข้อมูลกลาง context ยาวได้แย่กว่าหัว/ท้าย แม้เป็นโมเดล long-context | — | เหตุผลที่ **ไม่ยัดทั้งเล่ม** แม้ context ใหญ่พอ: ส่ง chunk ที่คัดแล้ว + ข้อเท็จจริงสำคัญไว้หัว/ท้าย |
| **LightRAG** (HKUDS 2024 · arXiv 2410.05779) | paper + OSS | index แบบกราฟ (entity + relation) ค้นสองระดับ: entity เจาะจง / ธีมกว้าง · อัปเดต index ทีละส่วนได้ | MIT (verify) | ยืนยันว่า **กราฟ entity ข้ามแอปของ Insight** ใช้เป็น retrieval ได้ (low-level = entity · high-level = กลุ่ม/บท) · ponytail: ไม่ยก LightRAG ทั้งตัว (ใช้ LLM สร้างกราฟ = กินโทเคน) — กราฟเรามาจากข้อมูลจริงในแอปฟรีอยู่แล้ว |

### 2.3 ตอบด้วยของในเครื่อง (E0) — โมเดล/เครื่องมือเล็ก

| ชื่อ | สาระ | licence | Z.I.Z ใช้อย่างไร |
|---|---|---|---|
| **Model2Vec** (`MinishLab/model2vec`) · **potion-multilingual-128M** | กลั่น sentence transformer เป็นตาราง embedding แบบ static · ไม่ต้องใช้ข้อมูลฝึก · CPU ล้วน · potion-multilingual กลั่นจาก BGE-M3, 101 ภาษา, 256 มิติ | MIT (repo + model card) | **ฐานของ `ZIZ-M-embed`** (`08` §2 เขียนไว้แล้วว่า model2vec) · มิติ 256 ตรงกับ ABI `embed/1` · ต้องวัด recall ภาษาไทยก่อน |
| **GLiNER** (`urchade/GLiNER`) · `gliner_multi-v2.1` | NER ที่ระบุชนิด entity ตอนใช้งานได้ (zero-shot) · ~200M พารามิเตอร์ | v2.1 = Apache-2.0 · รุ่นแรก `gliner_multi` = **CC-BY-NC** (verify บน model card) | ผู้สมัคร teacher สำหรับ distill `ZIZ-M-ner` หรือใช้ตรงถ้า ONNX int8 พอดีงบ RAM · **ห้ามใช้รุ่น NC** |
| **PyThaiNLP** | ตัดคำ (newmm) · NER ไทย (thainer) · เครื่องมือไทยครบ | โค้ด Apache-2.0 · ข้อมูลของโครงการ CC0 · **โมเดล NER ต้อง verify แยก** | ใช้ใน `ZIZ-M-ner/train/` (สร้าง label · ประเมิน) — **ไม่ฝังในแอป** (Python) |
| **Typhoon 2** (SCB 10X · 1B/3B/…) · **Typhoon T1** (3B reasoning, arXiv 2502.09042) | LLM เน้นไทย มีรุ่นเล็ก 1B/3B สำหรับบนเครื่อง | Typhoon-7B รุ่นแรก Apache-2.0 · **Typhoon 2 ต้อง verify** | ผู้สมัครฐานของ `ZIZ-M-gen` แทน ~0.5B ทั่วไป (`08` §2 · ❓ Q62) — 1B Q4 ≈ 0.7 GB เกินงบเดิม 300–400 MB ต้องตัดสินใจ |
| **GPTCache** (`zilliztech/GPTCache`) | semantic cache: คำถามคล้ายเดิม → คืนคำตอบเดิมโดยไม่เรียก LLM · เตือนเองว่ามี false positive | MIT (verify) | **ยืมแนวคิด ไม่ยกโค้ด** (Python) → `ziz-cache` ใน `ziz.db` (§3.1) ใช้ `ZIZ-M-embed` ที่มีอยู่ |
| **llama.cpp** | runtime GGUF บน CPU | MIT | มีใน `08` แล้ว (Q43 ✅) |

### 2.4 ตอบสั้นลง (E4)

| ชื่อ | สาระ | Z.I.Z ใช้อย่างไร |
|---|---|---|
| **Chain of Draft** (Xu et al. 2025 · arXiv 2502.18600) | ให้โมเดลเขียนขั้นคิดแบบโน้ตสั้นแทน CoT ยาว · รายงานว่าแม่นใกล้ CoT โดยใช้โทเคนเหลือราว 7.6 % ในบางงาน · ผู้เขียนเตือนว่าไม่ใช่ตัวแทน CoT ทุกงาน | ใช้ใน prompt ของ recipe ที่เป็นงานจัดกอง (ยืนยันชื่อ · จับคู่ alias) — **ไม่ใช้**กับงานเขียนคำอธิบาย/สังเคราะห์ · วัด A/B ก่อนเปิดเป็นค่าเริ่ม |
| effort / thinking ของ CLI | Claude Code: ตั้ง `/effort` ต่ำลงสำหรับงานง่าย · thinking คิดเป็นโทเคนออก · (เอกสาร Claude Code "Manage costs") | job ของ Z.I.Z ส่งระดับ effort ตามชนิดขั้น (§3.7) ถ้า CLI รองรับทาง ACP/flag (verify ต่อ CLI) |

### 2.5 ลด hallucination แบบถูก (E5)

| ชื่อ | สาระ | Z.I.Z ใช้อย่างไร |
|---|---|---|
| **Chain-of-Verification (CoVe)** (Dhuliawala et al. · ACL Findings 2024 · arXiv 2309.11495) | ร่างคำตอบ → ตั้งคำถามตรวจ → ตอบคำถามตรวจแยกกัน → แก้คำตอบ | **ทำแบบ deterministic แทนการเรียก LLM เพิ่ม**: "คำถามตรวจ" = ทุกข้อเท็จจริงต้องมี quote + ref · ตรวจด้วยโค้ด (`04` §5.7) · ส่งกลับให้ agent **เฉพาะข้อที่ไม่ผ่าน** (ไม่ส่งทั้งชุดซ้ำ) |
| **Self-RAG** (Asai et al. · ICLR 2024 · arXiv 2310.11511) | โมเดลเรียนรู้ว่าเมื่อไรควรค้น และตัดสินว่าคำตอบ "มีหลักฐานรองรับ" ไหม · token ตรวจ "Is Supported" สำคัญที่สุดในผลทดลอง | ยืนยันว่า **การตรวจว่ามีหลักฐานรองรับ** คือส่วนที่ได้ผลที่สุด → ทำเป็นโค้ด ไม่ต้อง train · "ค้นเมื่อจำเป็น" = agent ได้ outline ก่อน แล้วเรียก `ziz_source_read` เอง |
| Lost in the Middle (2.2) | — | ข้อเท็จจริงที่ต้องตรวจวางท้าย prompt ใกล้คำสั่ง |

### 2.6 ภาษาไทยจ่ายแพงกว่า

**Petrov et al., "Language Model Tokenizers Introduce Unfairness Between Languages"** (NeurIPS 2023 · arXiv 2305.15425 · ข้อมูล `AleksandarPetrov/tokenization-fairness`):
ข้อความเดียวกันต่างภาษาใช้จำนวนโทเคนต่างกันได้ถึง ~15× แม้ tokenizer แบบหลายภาษา → ผู้ใช้ภาษาที่ไม่ใช่อังกฤษจ่ายแพงกว่าและ context เต็มเร็วกว่า

ผลต่อ Z.I.Z (ผู้ใช้หลักเป็นนิยายไทย):
- **ทุกตัวอักษรไทยที่ไม่ต้องส่ง มีค่ามากกว่าภาษาอังกฤษหลายเท่า** → E0/E2 สำคัญกับเรามากกว่าแอปทั่วไป
- ตัวเลข "≈ ตัวอักษร ÷ 3" ใน `12` §4 ต้องแทนด้วยค่าที่วัดจริงต่อ tokenizer (ดูไทยในไฟล์ `tokenization_lengths.csv` ของ Petrov และวัดเองกับ CLI ที่รองรับ)
- ส่วนที่เป็น**คำสั่ง/โครงสร้าง** (system prompt · schema · ชื่อ field) เขียนเป็นภาษาอังกฤษ — สั้นกว่าและ cache ได้ · เนื้อหาผู้ใช้คงเป็นไทย (ต่อ Q24)

### 2.7 CLI ฟรี/ถูก (สถานะเปลี่ยนเร็วมาก — ตรวจทุกรอบ release)

| CLI | สถานะที่พบ (2026-11) | ความน่าเชื่อถือ |
|---|---|---|
| **Gemini CLI** (open source) | เอกสารโควตาทางการระบุล็อกอิน Google ส่วนตัว **1,000 คำขอ/วัน** · API key ฟรี 250/วัน (เฉพาะ Flash) · **มีแหล่งหนึ่งรายงานว่า personal login ถูกปิดเมื่อ 2026-06-18** ขัดกับแหล่งอื่น | ❓ ขัดกัน — verify ก่อนเขียนลง UI |
| **Qwen Code** (open source) | free OAuth เคยมี 1,000–2,000/วัน · มี issue เสนอลดเหลือ 100/วัน และปิด 2026-04-15 · ตัว CLI ยังฟรีและ open source | ❓ น่าจะปิดแล้ว — verify |
| **Claude Code** | ใช้กับแผนสมัครสมาชิก/API · มี prompt caching อัตโนมัติ (cache read ≈ 0.1× ของราคา input) · compaction อัตโนมัติ · `/usage` บอกสัดส่วน cache | เอกสารทางการ |
| CLI + โมเดลในเครื่อง (เช่น CLI ที่ต่อ endpoint ภายในเครื่อง) | ฟรีจริงแต่คุณภาพไทยต่ำกว่ามาก | — |

**ข้อสรุป:** "ฟรี" ของ CLI เป็นเงื่อนไขของผู้ให้บริการ ไม่ใช่ของเรา และเปลี่ยนได้ทุกเดือน → Z.I.Z **ไม่ผูกคุณสมบัติใดกับโควตาฟรีของใคร** ·
ทางที่ฟรีแน่นอนคือ **E0 (ในเครื่อง)** · ส่วน CLI: ตรวจ/แสดงสถานะที่ CLI รายงานเอง และให้ router (§3.2) ใช้ CLI ที่ผู้ใช้ติดตั้งไว้หลายตัวอย่างคุ้มที่สุด

---

## 3. สิ่งที่ Z.I.Z จะทำ (ข้อเสนอการออกแบบ)

### 3.1 `ziz-cache` — ไม่ถามคำถามเดิมซ้ำ (E0)

- key = hash(recipe step · scope roots · content_hash ของ chunk ที่ใช้ · เวอร์ชัน prompt) → **cache ตรงตัว** ของผลขั้นนั้น (findings/ops) ใน `ziz.db`
  - นิยายแก้ 1 บท → เฉพาะ chunk ที่ hash เปลี่ยนต้องส่งใหม่ (ต่อจาก IR cache `09` §4 ที่ cache การแยกไฟล์ — ตัวนี้ cache **ผลของ agent**)
- **semantic cache** (แบบ GPTCache) **เฉพาะ Link `link_ask` / S2**: คำถามคล้ายเดิม (cosine ≥ 0.92 ด้วย `ZIZ-M-embed`) **และ** ข้อมูลในขอบเขตไม่เปลี่ยน → แสดงคำตอบเดิมพร้อมป้าย `↺ คำตอบเมื่อ 2 ชม.ก่อน · [ถามใหม่]`
  - ponytail: ไม่ทำ semantic cache กับขั้นที่สร้าง op — false positive = เขียนข้อมูลผิด · ใช้แบบตรงตัวเท่านั้น
- ล้างเมื่อ: เปลี่ยนเวอร์ชัน recipe/prompt · เปลี่ยน CLI/โมเดล · ผู้ใช้กด `ถามใหม่`

### 3.2 Router ระหว่างชั้น (E1) — ขยาย `08` §3

```
งาน/คำถาม → E0 ตอบได้ไหม (index · กฎ · โมเดลเล็ก · cache)?  ใช่ → จบ (ป้าย "ในเครื่อง · 0 โทเคน")
          → ไม่ → ชนิดงาน (จาก recipe step):
               จัดกอง/ยืนยัน/จับคู่        → CLI "ประหยัด" ที่ผู้ใช้ตั้ง (เช่น Claude Code + โมเดลเล็กกว่า / effort ต่ำ / CLI ที่ยังมีโควตา)
               เขียน/สังเคราะห์/ตัดสินยาก  → CLI "หลัก"
          → ผลไม่ผ่านตรวจ (E5) หรือความมั่นใจต่ำ → ยกระดับไป CLI หลัก ครั้งเดียว (cascade แบบ FrugalGPT)
```
- ผู้ใช้กำหนดใน ตั้งค่า → agent: `หลัก` 1 ตัว · `ประหยัด` 0–1 ตัว (ตัวเดียวกันแต่โมเดล/effort ต่างก็ได้) · ไม่ตั้ง "ประหยัด" = ใช้หลักทุกงาน (เหมือนเดิม)
- กฎตายตัว + คะแนน `route` — **ไม่ train router** (ponytail: เปลี่ยนเป็น RouteLLM-style เมื่อมีข้อมูลผล accept/reject มากพอ) ❓ Q63
- ทุกการตัดสินบันทึก "ใครทำ" ใน `runs/*.ndjson` (หลัก UX ข้อ 6)

### 3.3 ส่งเฉพาะที่ต้องใช้ (E2) — ต่อ `12` §3

| เทคนิค | ที่มา | ทำที่ |
|---|---|---|
| ขอบเขต Secured เป็นค่าเริ่มของงาน AI (Q53 ✅) | — | gateway |
| outline ก่อนเนื้อหา (`ziz_scope_outline`) จัดอันดับแบบ repo map | Aider | gateway |
| ข้าม chunk ไม่เกี่ยว (`route`) + NER pre-pass ส่งเป็นตาราง | FrugalGPT (prompt adaptation) · `08` | router |
| ข้อเท็จจริงสำคัญวางหัว/ท้าย ไม่วางกลาง | Lost in the Middle | ตัวประกอบ prompt |
| ประกาศเฉพาะ tool ของ recipe นั้น | RAG-MCP | gateway (`tools/list` ต่อ job) |

### 3.4 บีบ prompt (E2) — `ZIZ-M-compress` (ทางเลือก · ❓ Q61)

- โมเดลแบบ LLMLingua-2 (token classifier) บีบ **เฉพาะ "บริบทประกอบ"** (สรุปบทก่อนหน้า · คำอธิบาย entity เดิม) — **ไม่บีบ** ข้อความที่ agent ต้อง quote กลับ และไม่บีบคำสั่ง
- เป้า: ลดโทเคนเข้าของบริบทประกอบ ≥ 40 % โดย recall ตัวละครหลักลดไม่เกิน 1 จุด (fixture `novel-*`) · ไม่ผ่าน = ไม่ทำ
- ต้องฝึก/กลั่นใหม่ให้ไทย (classifier ต้นฉบับเน้นอังกฤษ) และ verify licence ของน้ำหนักก่อน

### 3.5 ซ่อนผล tool เก่า (E2) — observation masking

- gateway เห็นทุก tool call ของ job · ผล `ziz_source_read`/`ziz_index_search` ที่เก่ากว่า **3 รอบ** ในรอบถัดไป: Z.I.Z **ไม่ได้คุมประวัติแชตของ CLI โดยตรง** →
  ทำได้เฉพาะ (ก) แบ่ง job เป็น **session สั้นต่อ chunk/ขั้น** (map/reduce ของ `04` §5 ทำอยู่แล้ว — session ใหม่ = ประวัติว่าง) และ (ข) สถานะสะสมเก็บใน `ziz_findings_query` ไม่ใช่ในแชต
- นี่คือรูปแบบ masking ที่ถูกที่สุด: agent ไม่ต้องลากผลเก่า เพราะเริ่ม session ใหม่ทุกขั้นแล้วดึงสิ่งที่ต้องใช้จาก gateway · ❓ Q64 (session ต่อขั้น vs ต่อ job — ขัดกับ E3 cache)

### 3.6 จัด prompt ให้ cache ได้ (E3)

prompt caching ของผู้ให้บริการ (เช่น Claude: อ่านจาก cache ≈ 0.1× ของราคา input · เขียน cache 1.25× ที่ TTL 5 นาที) **คุ้มเมื่อ prefix เดิมถูกใช้ซ้ำก่อนหมดอายุ** — ถ้าไม่ซ้ำจะจ่ายค่าเขียนแพงขึ้นเปล่า ๆ

| กฎ | เหตุผล |
|---|---|
| ลำดับคงที่: **system/recipe → schema ของ op → outline ของขอบเขต** → (ส่วนที่เปลี่ยน) chunk ปัจจุบัน → คำถาม | ส่วนหน้า 3 ก้อนเหมือนกันทุก chunk ของ job → cache hit |
| ไม่ใส่เวลา/สุ่ม/ตัวนับในส่วนหน้า | ตัวอักษรเปลี่ยน 1 ตัว = cache miss ทั้งก้อน |
| tool definition ของ gateway คงที่ตลอด job (ไม่เพิ่ม/ลบกลางทาง) | เอกสาร Claude Code: "tool definitions changed" เป็นสาเหตุ miss |
| ส่ง chunk ถัดไปทันที (ไม่รอ) · คิวงานต่อ job ไม่แทรกงานอื่น | อยู่ใน TTL |
| แสดง `% จาก cache` ต่อ job (ถ้า CLI รายงาน — Claude Code มีใน `/usage`) | ผู้ใช้เห็นว่าประหยัดจริงไหม |

ขัดกับ §3.5 (session ใหม่ต่อขั้น): prefix ที่เหมือนกันยัง cache ข้าม session ได้ถ้าเนื้อหาตรงทุกตัวอักษรและอยู่ใน TTL (พฤติกรรมของผู้ให้บริการ — verify ต่อ CLI) → แบ่ง session ได้โดยไม่เสีย cache ถ้าทำตามกฎข้างบน

### 3.7 ให้ตอบสั้น (E4)

- ทุกขั้นที่สร้างข้อเสนอ: ผลเป็น **op JSON ผ่าน `ziz_changeset_add_ops`** เท่านั้น · สั่งชัดว่า "ไม่ต้องอธิบายในแชต" (ร้อยแก้วในแชตคือโทเคนออกที่ทิ้ง)
- ขั้นจัดกอง: prompt แบบ Chain of Draft ("คิดเป็นโน้ตสั้น ≤ 5 คำต่อขั้น") + effort ต่ำ (ถ้า CLI รองรับ) · วัด A/B บน fixture ก่อนเปิด
- รายงาน: agent ส่ง **ตาราง/ตัวเลข** → Z.I.Z จัดรูปเอง (ไม่ให้ agent เขียน markdown ยาว)

### 3.8 ตรวจแทนการทำซ้ำ (E5)

- ผลไม่ผ่านตรวจ quote/ref/schema → ส่งกลับให้ agent **เฉพาะรายการที่ไม่ผ่าน + เหตุผล** (ไม่ส่ง chunk ทั้งก้อนซ้ำ) · แก้ได้ไม่เกิน 1 รอบ แล้วทิ้งรายการนั้นพร้อมแจ้งในรายงาน
- "ไม่พบในขอบเขต" เป็นคำตอบที่ถูก (`12` §3.4) — ไม่ยกระดับไป Opened เอง

### 3.9 สมุดโทเคน (token ledger) — ต้องวัดได้ก่อนอ้างว่าประหยัด

| ที่ | ข้อมูล |
|---|---|
| ต่อ job / ต่อคำขอ S2 / ต่อ `link_ask` | โทเคนเข้า (ใหม่/cache) · ออก · CLI/โมเดล · ชั้นที่ตอบ (E0…E5) · เวลา — จาก usage ที่ CLI รายงานทาง ACP/headless JSON (verify ต่อ CLI) · ไม่มี → ประมาณจากตัวอักษร ติด `≈` |
| หน้า Node (`12` §4) | `วันนี้: 14 งาน · ในเครื่อง 9 (0 โทเคน) · CLI 5 (~41k · cache 78 %)` |
| รายงานรายสัปดาห์ (Insight) | "ประหยัดได้" = โทเคนที่ E0/cache ตอบแทน (ประมาณจากขนาดบริบทที่จะส่ง) — ติดป้ายว่าเป็นค่าประมาณเสมอ |
| ไม่ทำ | ไม่ส่ง ledger ออกนอกเครื่อง · ไม่ตั้งราคาเป็นเงินเอง (แผนสมัครสมาชิกไม่คิดต่อโทเคน) — แสดงเป็นโทเคน + % ของโควตาถ้า CLI บอก |

---

## 4. ไม่ทำ (และเหตุผล)

| ไม่ทำ | เหตุผล |
|---|---|
| train router (RouteLLM) ใน v1 | ยังไม่มีข้อมูล preference · กฎ + `route` พอ |
| LightRAG/GraphRAG ที่ใช้ LLM สร้างกราฟ | กินโทเคนตอนสร้าง · เรามีกราฟจากข้อมูลแอปจริงอยู่แล้ว |
| LLM สรุปประวัติแชต (condenser) | masking/แบ่ง session ถูกกว่าและได้ผลเท่ากันในงานวิจัย (Complexity Trap) |
| semantic cache กับขั้นที่สร้าง op | false positive = ข้อมูลผิด |
| ผูกฟีเจอร์กับโควตาฟรีของ CLI ใด | เปลี่ยนเร็ว (Gemini/Qwen 2026) |
| บีบข้อความที่ใช้เป็น quote | ทำลายการตรวจหลักฐาน |

---

## 5. ฟีเจอร์ (ต่อ `03` หมวด K)

| # | ฟีเจอร์ | ระดับ |
|---|---|---|
| F101 | `ziz-cache` แบบตรงตัวต่อขั้น (content_hash) | MVP |
| F102 | semantic cache เฉพาะ `link_ask`/S2 | v1 |
| F103 | router ระหว่างชั้น E0 → CLI ประหยัด → CLI หลัก + cascade 1 ครั้ง | v1 |
| F104 | จัด prompt ให้ cache ได้ + แสดง % cache | MVP (ไม่มีต้นทุน แค่ลำดับ) |
| F105 | สมุดโทเคน + สรุปบนหน้า Node | MVP |
| F106 | ผลเป็น op/ตารางเท่านั้น + prompt แบบ Chain of Draft ในขั้นจัดกอง (หลังผ่าน A/B) | v1 |
| F107 | `ZIZ-M-compress` (LLMLingua-2 แบบไทย) | later (ต้อง eval + licence) |
| F108 | วัดอัตราโทเคนไทยต่อ CLI แทน "÷ 3" | MVP (ก่อนแสดงตัวเลขใด ๆ) |

---

## 6. แหล่งอ้างอิง

**Paper:** FrugalGPT [arXiv 2305.05176](https://arxiv.org/abs/2305.05176v1) · RouteLLM [arXiv 2406.18665](https://arxiv.org/html/2406.18665v3) · [LMSYS blog](https://lmsys.org/blog/2024-07-01-routellm/) ·
LLMLingua-2 [arXiv 2403.12968](https://www.huggingface.co/spaces/microsoft/llmlingua-2/blob/main/README.md) · Agentless [arXiv 2407.01489](https://arxiv.org/pdf/2407.01489.pdf) ·
The Complexity Trap [arXiv 2508.21433](https://arxiv.org/html/2508.21433v3) · RAG-MCP [arXiv 2505.03275](https://arxiv.org/abs/2505.03275) ·
Lost in the Middle [arXiv 2307.03172](https://arxiv.org/abs/2307.03172v2) · LightRAG [arXiv 2410.05779](https://arxiv.org/html/2410.05779v2) ·
Chain of Draft [arXiv 2502.18600](https://arxiv.org/abs/2502.18600) · Chain-of-Verification [arXiv 2309.11495](https://preview.aclanthology.org/add_acl24_videos/2024.findings-acl.212) ·
Self-RAG [arXiv 2310.11511](https://arxiv.org/abs/2310.11511v1) · Tokenizer unfairness [NeurIPS 2023](https://proceedings.neurips.cc/paper_files/paper/2023/hash/74bb24dca8334adce292883b4b651eda-Abstract.html) ·
Typhoon T1 [arXiv 2502.09042](https://arxiv.org/html/2502.09042v2) · PyThaiNLP [arXiv 2312.04649](https://ar5iv.labs.arxiv.org/html/2312.04649)

**Open source / เอกสาร:** [microsoft/LLMLingua](https://www.sourcepulse.org/projects/1237038) · [LLMLingua-2 JS port](https://cdn.jsdelivr.net/npm/@atjsh/llmlingua-2@2.0.3/README.md) ·
[lm-sys/RouteLLM](https://www.sourcepulse.org/projects/1841806) · [aider repo map (repo-mapper)](https://docs.rs/crate/repo-mapper/latest/source/README.md) ·
[OpenHands condenser](https://docs.openhands.dev/sdk/guides/context-condenser) · [OpenHands blog](https://openhands.dev/blog/openhands-context-condensensation-for-more-efficient-ai-agents) ·
[GPTCache](https://gpt-cache-test.readthedocs.io/en/latest/index.html) · [model2vec](https://github.com/MinishLab/model2vec) · [potion-multilingual-128M](https://huggingface.co/minishlab/potion-multilingual-128M/blob/main/README.md) ·
[GLiNER](https://pypi.org/project/gliner/0.1.9/) · [PyThaiNLP FAQ](https://pythainlp.github.io/FAQ) · [Typhoon 2](https://medium.com/scb10x/introducing-typhoon-2-0806fb040c45) ·
[tokenization-fairness data](https://github.com/AleksandarPetrov/tokenization-fairness) · [HKUDS/LightRAG](https://arxiv.org/html/2410.05779v2) · [AkariAsai/self-rag](https://github.com/AkariAsai/self-rag)

**CLI:** [Claude Code — Manage costs](https://code.claude.com/docs/en/costs) · [Gemini CLI quota](https://geminicli.com/docs/resources/quota-and-pricing/) · [รายงานการเปลี่ยน Gemini CLI](https://agentdeals.dev/vendor/gemini-cli) ·
[Qwen Code issue #3203](https://github.com/QwenLM/qwen-code/issues/3203) · [รายงาน Qwen Code 2026](https://inventivehq.com/blog/qwen-code-still-free-2026-shutdown) · [prompt caching multipliers (แหล่งรอง)](https://docs.helicone.ai/gateway/concepts/prompt-caching)
