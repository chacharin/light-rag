# Chatbot ตอบคำถามเชิงลึกและข้อมูลเฉพาะทาง ด้วยเทคนิค RAG

## 1. หน้าปก

- ชื่อบทเรียน: "Chatbot ตอบคำถามเชิงลึกและข้อมูลเฉพาะทาง ด้วยเทคนิค RAG (Retrieval-Augmented Generation)"
- คำโปรย (subtitle): จากคลังความรู้เฉพาะทาง สู่ Chatbot ที่ตอบได้ตรงประเด็น ไม่มโน

## 2. ภาพรวมโครงร่างการบรรยาย

สไลด์สรุป roadmap ทั้ง 7 หัวข้อ (agenda slide) เพื่อให้ผู้เรียนเห็นภาพรวมก่อนเข้าเนื้อหา:

- ทำไมต้องรู้เรื่องนี้ (Motivation)
- RAG คืออะไร (Concept)
- องค์ประกอบของ RAG (Architecture)
- เครื่องมือสำเร็จรูป (Tools landscape)
- Workshop: ลงมือสร้างจริงด้วย LightRAG + Vercel

## 3. ทำไมต้องรู้เรื่องนี้

ประเด็นหลักที่ควรพูดถึง:

- ข้อจำกัดของ LLM ล้วน ๆ (เช่น ChatGPT ธรรมดา): ตอบจากความรู้ที่ถูก train มาเท่านั้น ไม่รู้ข้อมูลใหม่ ไม่รู้ข้อมูลภายในองค์กร/หน่วยงาน และมีโอกาส "hallucinate" (มโนคำตอบ) เมื่อถูกถามเรื่องที่ไม่มีในความรู้
- ตัวอย่างสถานการณ์จริงที่ต้องการคำตอบเฉพาะทาง: คู่มือปฏิบัติงานภายใน, ระเบียบ/ข้อบังคับเฉพาะหน่วยงาน, เอกสารวิชาการ/งานวิจัยเฉพาะสาขา, FAQ ของหลักสูตร
- ทางเลือกที่มีอยู่ก่อน RAG และข้อจำกัด: Fine-tuning (ค่าใช้จ่ายสูง, ต้องอัปเดตโมเดลใหม่ทุกครั้งที่ข้อมูลเปลี่ยน) vs. ยัดข้อมูลทั้งหมดใน prompt (จำกัดด้วย context window, ค่าใช้จ่ายต่อ request สูงขึ้น)
- RAG คือคำตอบตรงกลาง: ใช้ LLM ที่มีอยู่แล้ว (ไม่ต้อง fine-tune) + "หาข้อมูลที่เกี่ยวข้องมาป้อนให้" ก่อนตอบ ทำให้ตอบได้ทันเวลา อัปเดตง่าย และอ้างอิงแหล่งที่มาได้

## 4. RAG คืออะไร

- นิยาม: Retrieval-Augmented Generation = เทคนิคที่ผสาน "การค้นคืนข้อมูล" (Retrieval) เข้ากับ "การสร้างคำตอบด้วย LLM" (Generation)
- หลักการทำงานแบบภาพรวม (flow 4 ขั้นตอน): คำถามผู้ใช้ → ค้นหาข้อมูลที่เกี่ยวข้องจากคลังความรู้ → นำข้อมูลที่ค้นได้ไปแนบกับคำถามเป็น prompt → LLM สร้างคำตอบโดยอ้างอิงจากข้อมูลนั้น
- เปรียบเทียบให้เห็นภาพ: เหมือนเปิดหนังสือสอบ (open-book exam) แทนที่จะให้ LLM ตอบจากความจำล้วน ๆ (closed-book)
- ข้อดีหลัก: ลด hallucination, อัปเดตความรู้ได้โดยไม่ต้อง train โมเดลใหม่, อ้างอิงแหล่งที่มาได้ (traceability), ควบคุมขอบเขตคำตอบได้

## 5. องค์ประกอบของ RAG

แจกแจงส่วนประกอบหลักที่ต้องมี:

- **Document Ingestion**: การนำเอกสารต้นทาง (PDF, DOCX, PPTX, TXT ฯลฯ) เข้าระบบ
- **Chunking**: การตัดแบ่งเอกสารเป็นชิ้นย่อย ๆ เพื่อให้ค้นคืนได้แม่นยำ
- **Embedding Model**: โมเดลที่แปลงข้อความเป็นเวกเตอร์ (ตัวเลข) เพื่อวัดความใกล้เคียงเชิงความหมาย
- **Vector Database**: ที่เก็บเวกเตอร์เพื่อค้นคืนแบบ similarity search (เช่น NanoVectorDB ที่ LightRAG ใช้, หรือ Milvus, Qdrant, pgvector)
- **Knowledge Graph** (ส่วนเสริมของ LightRAG โดยเฉพาะ): การสกัด entity/relationship จากเอกสาร เพื่อให้ตอบคำถามเชิงความสัมพันธ์ได้ดีขึ้น ไม่ใช่แค่ค้นคำที่คล้ายกัน
- **Retriever**: กลไกค้นคืนชิ้นข้อมูลที่เกี่ยวข้องที่สุดกับคำถาม
- **LLM (Generator)**: โมเดลที่นำข้อมูลที่ค้นได้มาสังเคราะห์เป็นคำตอบ
- **Prompt/Instruction Layer**: ส่วนควบคุมพฤติกรรมการตอบ (ภาษา, ขอบเขตคำตอบ, รูปแบบการอ้างอิง) — เชื่อมโยงกับ `user_prompt` ที่ใช้จริงใน workshop

## 6. เครื่องมือสำเร็จรูป

ภาพรวมกลุ่มเครื่องมือที่มีในตลาดปัจจุบัน (แบ่งตามลักษณะการใช้งาน):

- **Framework/Library สำหรับนักพัฒนา**: LangChain, LlamaIndex, Haystack — ยืดหยุ่นสูง แต่ต้องเขียนโค้ดประกอบเอง
- **RAG-as-a-Service / All-in-one server**: LightRAG (ที่ใช้ใน workshop), ทำหน้าที่ทั้ง ingestion, graph extraction, vector store, API server ในตัวเดียว ติดตั้งง่ายผ่าน Docker
- **No-code/Low-code platform**: Dify, Flowise — สร้าง RAG pipeline ผ่าน UI แบบลากวาง เหมาะกับผู้ไม่ถนัดเขียนโค้ด
- **Managed/Cloud platform**: บริการ RAG บน cloud (เช่นของ OpenAI, Azure AI Search, Google Vertex AI Search) — สะดวกแต่ผูกกับผู้ให้บริการและมีค่าใช้จ่ายตามการใช้งาน
- เกณฑ์การเลือกเครื่องมือ: ขนาดคลังข้อมูล, งบประมาณ, ต้องการ knowledge graph หรือไม่, ทีมมีนักพัฒนาหรือไม่, ต้องการ host เองหรือใช้ cloud

## 7. Workshop

ภาคปฏิบัติ ต่อยอดจากงานที่ทำสำเร็จแล้วในคลาสนี้ (อ้างอิง `prepare-lesson.md`):

- ติดตั้ง LightRAG ผ่าน Docker Compose บนเครื่อง local
- ตั้งค่า LLM/Embedding ผ่าน OpenRouter (`.env`)
- นำเข้าเอกสารเฉพาะทางของผู้เรียนเอง (เช่น คู่มือ/ระเบียบของหน่วยงาน) เพื่อสร้างคลังความรู้
- ทดสอบถาม-ตอบผ่าน LightRAG WebUI และ Postman
- Build image (`Dockerfile.vercel`) ที่ฝังคลังความรู้ (baked-in knowledge base) เพื่อ deploy เป็น read-only API
- Deploy ขึ้น Vercel ผ่าน GitHub (ตั้งค่า Environment Variables, ปิด Deployment Protection)
- ทดสอบยิง query ผ่าน Postman จนได้คำตอบจริงจาก cloud
- (โจทย์ฝึกต่อยอด) ให้ผู้เรียนลองใช้ `user_prompt` เพื่อบังคับให้ตอบเฉพาะภาษาไทยและอยู่ในขอบเขตคลังข้อมูล

### ตัวเลือกทั้งหมดของ `POST /query`

ตรวจสอบจากซอร์สโค้ดจริง (`lightrag/api/routers/query_routes.py`, class `QueryRequest`) มีทั้งหมด 17 ฟิลด์ แบ่งเป็นกลุ่มดังนี้:

**จำเป็น**

- `query` (string, บังคับ) — คำถามที่จะถาม ห้ามว่าง

**ควบคุมโหมดการค้นคืน**

- `mode` (`local` | `global` | `hybrid` | `naive` | `mix` | `bypass`, default `"mix"`)
  - `local` — เน้น entity เฉพาะจุดและความสัมพันธ์โดยตรงของมัน
  - `global` — วิเคราะห์รูปแบบ/ความสัมพันธ์ในภาพกว้างทั่ว knowledge graph
  - `hybrid` — ผสาน local + global
  - `naive` — vector similarity search ธรรมดา ไม่ใช้ knowledge graph เลย
  - `mix` — ผสาน knowledge graph + vector retrieval (ค่า default, แนะนำให้ใช้)
  - `bypass` — ยิงคำถามตรงเข้า LLM เลย ไม่ค้นคลังข้อมูลใด ๆ (เทียบเท่า ChatGPT ธรรมดา ใช้เทียบผลหรือ debug)

**ควบคุมปริมาณ/ขอบเขตข้อมูลที่ดึงมา**

- `top_k` (int) — จำนวนรายการบนสุดที่ดึงมา (โหมด `local` = จำนวน entity, โหมด `global` = จำนวน relationship) ค่า default มาจาก env `TOP_K` ของ server
- `chunk_top_k` (int) — จำนวน text chunk ที่ดึงจาก vector search ตอนแรก แล้วคัดเหลือหลัง rerank ค่า default จาก env `CHUNK_TOP_K`
- `max_entity_tokens` / `max_relation_tokens` / `max_total_tokens` (int) — งบ token สูงสุดสำหรับส่วน entity, relationship, และรวมทั้งหมด (entity+relation+chunk+system prompt) ตามลำดับ ใช้คุมไม่ให้ context ยาวเกินจนแพง/เกิน context window ค่า default จาก env `MAX_ENTITY_TOKENS` / `MAX_RELATION_TOKENS` / `MAX_TOTAL_TOKENS`

**ควบคุม keyword ที่ใช้ค้น (ไม่บังคับ)**

- `hl_keywords` (list[str]) — high-level keyword ที่อยากให้เน้นค้น ถ้าปล่อยว่าง ระบบจะให้ LLM สร้าง keyword เองจาก query
- `ll_keywords` (list[str]) — low-level keyword เพื่อเจาะจงขอบเขตค้นให้แคบลง
- ใส่สองตัวนี้เพื่อ "ข้าม" ขั้นตอนที่ LLM ต้องวิเคราะห์คำถามเพื่อสกัด keyword เอง (ประหยัด 1 LLM call) เหมาะกับตอนรู้คำสำคัญอยู่แล้วหรือทำ automation

**ควบคุมรูปแบบ/พฤติกรรมคำตอบ**

- `response_type` (string) — รูปแบบคำตอบ เช่น `"Multiple Paragraphs"`, `"Single Paragraph"`, `"Bullet Points"` (default `"Multiple Paragraphs"`)
- `user_prompt` (string) — คำสั่งเพิ่มเติมที่แทรกเข้าไปในส่วน "Additional Instructions" ของ prompt ตอบคำถาม **ไม่มีผลต่อการค้นคืนข้อมูล** แต่มีผลต่อ "วิธีตอบ" — นี่คือฟิลด์ที่ใช้ใส่ instruction เช่น "ตอบเป็นภาษาไทยเท่านั้น" ค่านี้จะถูกต่อท้าย (append) เข้ากับ prefix ที่ server ตั้งไว้ (ถ้ามี)
- `disable_user_prompt_prefix` (bool) — ถ้า `true` จะไม่เอา global prefix (ที่ตั้งไว้ฝั่ง server) มาต่อกับ `user_prompt` ให้ client คุมข้อความ instruction เองทั้งหมด (default `false`) หมายเหตุ: โหมด `bypass` ไม่รับผลจากทั้ง `user_prompt` และ prefix นี้เลย
- `conversation_history` (list of `{"role": "...", "content": "..."}`) — ประวัติบทสนทนา ส่งให้ LLM ใช้เป็นบริบทตอบเท่านั้น **ไม่ถูกใช้ค้นคืนข้อมูล** เหมาะกับทำ multi-turn chat
- `enable_rerank` (bool) — เปิด/ปิดการ rerank chunk ที่ดึงมา (default `true` — ถ้าเปิดแต่ไม่ได้ตั้ง rerank model ไว้ จะมี warning เฉย ๆ ไม่ error)

**ควบคุมสิ่งที่ตอบกลับมา**

- `only_need_context` (bool) — ถ้า `true` จะคืนแค่ context ที่ค้นได้ ไม่เรียก LLM สร้างคำตอบ (เหมาะกับ debug ว่าค้นเจอข้อมูลถูกไหม)
- `only_need_prompt` (bool) — ถ้า `true` จะคืนแค่ prompt ที่ถูกประกอบขึ้น (query+context+instruction) โดยไม่ยิงไป LLM (เหมาะกับตรวจสอบว่า prompt สุดท้ายหน้าตาเป็นอย่างไร)
- `include_references` (bool, default `true`) — แนบรายการอ้างอิง (แหล่งที่มา) มาด้วยไหม
- `include_chunk_content` (bool, default `false`) — ถ้า `true` จะใส่เนื้อหาข้อความจริงของ chunk ไว้ใน reference ด้วย (ใช้ตอน evaluate/debug ว่าอ้างอิงถูกจุดไหม) ใช้ได้เมื่อ `include_references=true` เท่านั้น
- `include_progress` (bool, default `false`) — เฉพาะ `/query/stream` เท่านั้น ถ้า `true` จะส่ง event ความคืบหน้า (เช่น "extracting_keywords") มาก่อนคำตอบ
- `stream` (bool) — เปิด streaming หรือไม่ (default `false` สำหรับ `/query`, `true` สำหรับ `/query/stream`)

**ตัวอย่าง — บังคับภาษาไทยและจำกัดขอบเขตคลังข้อมูล** (ต่อยอดจากโจทย์ฝึกด้านบน):

```json
{
  "query": "คำถามทดสอบ",
  "mode": "hybrid",
  "user_prompt": "ตอบคำถามจากคลัง data และสนทนาเป็นภาษาไทยเท่านั้น ถ้าไม่พบคำตอบให้ตอบกลับว่า 'ไม่สามารถให้คำตอบได้เนื่องจากอยู่นอกขอบเขตคลังข้อมูล'"
}
```
