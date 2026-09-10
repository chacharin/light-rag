# เตรียมบทเรียน: Deploy LightRAG ขึ้น Vercel (Docker)

คู่มือนี้สรุปขั้นตอนทั้งหมดสำหรับติดตั้ง LightRAG, สร้างคลังความรู้ (knowledge base), แล้ว deploy เป็น API สาธารณะบน Vercel ผ่าน Docker — เขียนจากประสบการณ์จริงที่เจอ error ระหว่างทางและแก้ไขจนสำเร็จ ทำตามลำดับนี้จะไม่ต้องเจอ error ซ้ำ

## สิ่งที่ต้องเตรียมก่อนเริ่ม

- **Docker Desktop** ติดตั้งแล้ว และ **ต้องเปิดโปรแกรมทิ้งไว้จนขึ้นสถานะ "Engine running"** ก่อนใช้คำสั่ง `docker` ใดๆ (แค่ลงโปรแกรมไม่พอ ต้องเปิดรอ engine start ด้วย)
- **Git** ติดตั้งแล้ว
- บัญชี **GitHub** (สำหรับเก็บ repo ส่วนตัว)
- บัญชี **Vercel** (แนะนำ login ด้วย GitHub จะเชื่อมง่ายสุด)
- **API key ของ LLM provider** — ตัวอย่างนี้ใช้ OpenRouter (https://openrouter.ai) ซึ่งรองรับทั้งภาษาโมเดล LLM และ embedding ผ่าน endpoint เดียว

## ขั้นที่ 1 — Clone และรัน local ให้คลังข้อมูล "นิ่ง" ก่อน

```
git clone https://github.com/HKUDS/LightRAG.git
cd LightRAG
copy env.example .env
```

แก้ไฟล์ `.env` ใส่ค่า LLM/embedding โดยใช้ OpenRouter ผ่านช่อง `openai`-compatible endpoint:

```
LLM_BINDING=openai
LLM_BINDING_HOST=https://openrouter.ai/api/v1
LLM_MODEL=openai/gpt-4o-mini
LLM_BINDING_API_KEY=<OpenRouter API key>

EMBEDDING_BINDING=openai
EMBEDDING_BINDING_HOST=https://openrouter.ai/api/v1
EMBEDDING_MODEL=openai/text-embedding-3-large
EMBEDDING_BINDING_API_KEY=<OpenRouter API key>
EMBEDDING_DIM=3072
```

> ⚠️ **จุดพลาดที่พบบ่อยที่สุด:** ถ้าใช้โมเดล embedding เป็น `text-embedding-3-large` (หรือโมเดล custom อื่น) **ต้องใส่ `EMBEDDING_DIM=3072` เสมอ** ไม่งั้น server จะ crash ทันทีตอน start ด้วย error `ValueError: EMBEDDING_DIM must be set when EMBEDDING_MODEL selects a custom openai model`

รันด้วย Docker Compose:

```
docker compose up -d
```

เข้า `http://localhost:9621/webui/` → แท็บ Documents → **Upload** เอกสารที่ต้องการ → รอจนสถานะขึ้น **Completed** — ตอนนี้จะมี 2 โฟลเดอร์เกิดขึ้นในโปรเจกต์:

- `data/rag_storage/` — vector DB, knowledge graph (ไฟล์ `.json` และ `.graphml`)
- `data/inputs/` — เอกสารต้นฉบับที่อัปโหลดไป

ทดสอบผ่าน WebUI แท็บ **Retrieval** ให้แน่ใจว่าคลังข้อมูลตอบคำถามได้ถูกต้องก่อน ค่อยไปขั้นถัดไป

## ขั้นที่ 2 — แก้ไฟล์ที่จำเป็นก่อน commit (5 จุด)

### 2.1 แก้ `.gitignore`

หาหมวด `# Data & Storage` แล้วคอมเมนต์ 3 บรรทัดนี้ออก (ไม่งั้น Git จะไม่เก็บคลังข้อมูลเข้า repo เลย ทำให้ build บน Vercel หา source ไม่เจอ):

```diff
# Data & Storage
- inputs/
output/
- rag_storage/
- data/
+ # inputs/
+ # rag_storage/
+ # data/
```

`output/` ปล่อยไว้เหมือนเดิม (ไม่เกี่ยว), `.env` ปล่อย ignore ต่อไปตามเดิม (ไม่ต้องการให้ API key หลุดขึ้น GitHub)

### 2.2 แก้ `.dockerignore`

**คนละไฟล์กับข้อ 2.1** แต่ต้องแก้คู่กันเสมอ เพราะ `docker build` อ่านไฟล์นี้แยกจาก Git หาหมวด `# Exclude other projects` แล้วคอมเมนต์ 3 บรรทัดนี้:

```diff
# Exclude other projects
/tests
/scripts
- /data
/dickens
/reproduce
/output_complete
- /rag_storage
- /inputs
+ # /data
+ # /rag_storage
+ # /inputs
```

### 2.3 สร้างไฟล์ `Dockerfile.vercel` ใหม่ (ที่ root ของโปรเจกต์)

```dockerfile
FROM ghcr.io/hkuds/lightrag:latest

# Vercel's build tool (buildah) ไม่สืบทอด PATH จาก base image เหมือน Docker
# ปกติ ทำให้เจอ ModuleNotFoundError: No module named 'fastapi' ถ้าไม่ตั้งซ้ำเอง
ENV PATH=/app/.venv/bin:/root/.local/bin:$PATH

# Bake คลังความรู้ที่ประมวลผลเสร็จแล้วเข้า image (ใช้ตอบคำถามอย่างเดียว
# ไม่รองรับอัปโหลดเอกสารใหม่ผ่าน API บน Vercel เพราะ filesystem ไม่ persist)
COPY data/rag_storage /app/data/rag_storage
COPY data/inputs /app/data/inputs

# data/prompts เป็นโฟลเดอร์ว่าง (ไม่มี custom prompt) และ Git ไม่เก็บ
# โฟลเดอร์ว่างเข้า repo ใช้ RUN สร้างเองแทน COPY
RUN mkdir -p /app/data/prompts

ENV WORKING_DIR=/app/data/rag_storage
ENV INPUT_DIR=/app/data/inputs
ENV PROMPT_DIR=/app/data/prompts

# ห้ามตั้งเป็น 80: container รันด้วย user ที่ไม่ใช่ root (ดู
# docker-entrypoint.sh ของ LightRAG) จึง bind พอร์ตต่ำกว่า 1024 ไม่ได้
# (Permission denied) ตั้งพอร์ตจริงผ่าน Vercel Environment Variable แทน
EXPOSE 8080

# เรียก python จาก path เต็มใน .venv ตรงๆ กันปัญหา PATH ซ้ำอีกชั้น
CMD ["/app/.venv/bin/python", "-m", "lightrag.api.lightrag_server"]
```

**ห้ามทำ 4 อย่างนี้เด็ดขาด** (แต่ละอย่างคือ error ที่เจอมาแล้วจริง):
1. ห้าม `COPY data/prompts /app/data/prompts` ตรงๆ — ต้องใช้ `RUN mkdir -p` แทน
2. ห้ามลืม `ENV PATH=...` และห้ามเรียก `CMD ["python", ...]` เฉยๆ — ต้องใช้ path เต็ม `/app/.venv/bin/python`
3. ห้ามตั้ง `ENV PORT=80` หรือ `EXPOSE 80` — ใช้พอร์ต ≥ 1024 (เช่น 8080) เท่านั้น
4. ห้าม `COPY .env` เข้า image — ใส่ credential ผ่าน Vercel Environment Variables เท่านั้น

### 2.3b สร้างไฟล์ `vercel.json` ใหม่ (ที่ root ของโปรเจกต์ คนละไฟล์กับ Dockerfile.vercel)

LightRAG มี `pyproject.toml` ที่ระบุ `fastapi` เป็น dependency อยู่ในตัว — ปกติ Vercel จะ detect `Dockerfile.vercel` และ build เป็น container ให้อัตโนมัติโดยไม่ต้องมีไฟล์นี้ก็ได้ (ตามที่เคยทำสำเร็จมา) แต่ในบางกรณี (โปรเจกต์ใหม่, import ใหม่, หรือมีคนไปตั้งค่า Framework Preset ในหน้า Settings ไว้ก่อน) Vercel จะไป detect เป็น **FastAPI/Python project** แทน แล้ว build fail ด้วย:

```
Error: No FastAPI entrypoint found. Set "tool.vercel.entrypoint" in pyproject.toml or
define an entrypoint in one of: app.py, index.py, server.py, main.py, ...
```

ป้องกันไว้ล่วงหน้าด้วยการสร้าง `vercel.json` เพื่อบังคับให้ build เป็น container เสมอ ไม่ต้องพึ่งการ auto-detect:

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "framework": "container"
}
```

### 2.4 เปลี่ยน Git remote

โฟลเดอร์นี้ clone มาจาก `HKUDS/LightRAG` ต้นทาง (ไม่มีสิทธิ์ push) ต้องสร้าง repo ใหม่ของตัวเองบน GitHub ก่อน (private แนะนำ เพราะมีเอกสารของคุณอยู่ในนั้น) แล้วเปลี่ยน remote:

```
git remote set-url origin https://github.com/<username>/<ชื่อ repo ของตัวเอง>.git
```

(ใช้ `set-url` ไม่ใช่ `add` เพราะ `origin` มีอยู่แล้วจากตอน clone)

## ขั้นที่ 3 — Commit และ Push ครั้งเดียวจบ

```
git add .
git commit -m "Add Vercel deployment with baked-in knowledge base"
git push -u origin main
```

## ขั้นที่ 4 — Deploy บน Vercel (ผ่านหน้าเว็บ ไม่ต้องใช้ CLI)

1. เข้า vercel.com → **Add New** → **Project** → **Import Git Repository**
2. Authorize ให้ Vercel เข้าถึง repo ที่เพิ่ง push แล้วเลือก import
3. ที่หน้า Import ก่อนกด Deploy ให้เช็ค **Framework Preset** ต้องเป็น **Container** (ถ้า `vercel.json` มี `"framework": "container"` ตามข้อ 2.3b แล้ว ช่องนี้จะล็อกให้อัตโนมัติ — แต่ถ้าเคย import โปรเจกต์นี้มาก่อนแล้วมันดันเป็น **FastAPI** หรือ **Python** ให้เปลี่ยนเป็น Container ด้วยมือที่ **Settings → Build and Deployment → Framework Preset**) แล้วใส่ Environment Variables ให้ครบทุกตัวนี้ในทีเดียว:

```
LLM_BINDING=openai
LLM_BINDING_HOST=https://openrouter.ai/api/v1
LLM_MODEL=openai/gpt-4o-mini
LLM_BINDING_API_KEY=<OpenRouter API key>

EMBEDDING_BINDING=openai
EMBEDDING_BINDING_HOST=https://openrouter.ai/api/v1
EMBEDDING_MODEL=openai/text-embedding-3-large
EMBEDDING_BINDING_API_KEY=<OpenRouter API key>
EMBEDDING_DIM=3072

LIGHTRAG_API_KEY=<ตั้งรหัสลับที่คาดเดายาก ห้ามใช้ค่าง่ายๆ>
PORT=8080
```

4. กด **Deploy** รอ build เสร็จ (ประมาณ 1-3 นาที)
5. ไปที่ **Settings → Deployment Protection** → **ปิด Vercel Authentication สำหรับ Production** — ขั้นนี้สำคัญมาก ถ้าไม่ปิด ใครก็ยิง request มาจะโดนบล็อกด้วย `401 Protected deployment` ตั้งแต่ก่อนถึงตัว LightRAG เองเลย (คนละชั้นกับ `LIGHTRAG_API_KEY`)

## ขั้นที่ 5 — ทดสอบ

ใช้ Postman หรือเครื่องมือยิง API อื่นๆ:

```
POST https://<ชื่อโปรเจกต์>.vercel.app/query
```

Headers:
```
X-API-Key: <ค่าเดียวกับ LIGHTRAG_API_KEY>
Content-Type: application/json
```

Body:
```json
{
  "query": "คำถามทดสอบ",
  "mode": "hybrid"
}
```

ถ้าได้ `200 OK` พร้อม `response` และ `references` กลับมา แปลว่าใช้งานได้แล้ว

## หมายเหตุสำคัญสำหรับผู้สอน

- คลังความรู้ที่ bake เข้า image เป็นแบบ **อ่านอย่างเดียว** — endpoint `/documents/*` บน Vercel จะดูเหมือนอัปโหลดสำเร็จ แต่ข้อมูลจะหายไปทันทีที่ instance สลับ/สเกลลง เพราะ filesystem ของ Vercel Functions ไม่ persist ข้าม request ถ้าต้องการเพิ่มเอกสารใหม่ ต้องกลับมาทำที่เครื่อง local แล้ว build + push + deploy ใหม่ทั้งหมด
- ทุกครั้งที่แก้ `Dockerfile.vercel` แล้ว push ขึ้น GitHub, Vercel จะ build และ deploy ใหม่ให้อัตโนมัติ (เพราะเชื่อม Git integration ไว้แล้ว) ไม่ต้องสั่งอะไรเพิ่มในหน้าเว็บ
- ถ้าต้องการให้คำตอบเป็นภาษาไทยเสมอ/จำกัดขอบเขตคำตอบเฉพาะคลังข้อมูล ต้องให้ client (UI ที่จะยิง `/query`) แนบ field `user_prompt` มาด้วยทุกครั้ง เช่น:
  ```json
  {
    "query": "...",
    "mode": "hybrid",
    "user_prompt": "ตอบคำถามจากคลังข้อมูลนี้เท่านั้น และสนทนาเป็นภาษาไทยเท่านั้น หากไม่พบคำตอบในคลังข้อมูล ให้ตอบกลับด้วยข้อความนี้เท่านั้น: \"ไม่สามารถให้คำตอบได้เนื่องจากอยู่นอกขอบเขตคลังข้อมูล\""
  }
  ```
  (ไม่มีวิธีตั้งค่านี้ให้เป็นค่า default ของเซิร์ฟเวอร์แบบไม่ต้องพึ่ง client ส่งมาเอง เพราะ `PROMPT_DIR` ของ LightRAG ใช้สำหรับ custom entity-type prompt เท่านั้น ไม่ใช่ system prompt ของการตอบคำถาม)
