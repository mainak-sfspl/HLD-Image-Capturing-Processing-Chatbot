# Image Capturing & Processing for a Chatbot (HLD v1)

---

## Document Version Control

| Version | Date | Author | Change Description |
|--------|------|--------|-------------------|
| 1.0 | 2026-01-28 | IT Team | Initial HLD for image capture & processing chatbot |

---

## Table of Contents

1. Introduction  
2. Business Goals & Use Cases  
3. System Overview (End-to-End Flow)  
4. HLD Architecture  
5. Detailed Processing Pipeline  
6. Storage & Vectorization Design  
7. Free Tools & Stack  
8. Tool Comparison Matrix  
9. Security, Compliance & Retention  
10. Observability & KPIs  
11. Risks & Mitigations  
12. Recommended Architecture Blueprints

---

## 1. Introduction

### What this chatbot feature does

- User sends an image (photo / screenshot / document scan)
- System extracts meaning using **Computer Vision + AI**
  - OCR
  - Object detection
  - Image captioning
  - Image & text embeddings
- Chatbot answers with:
  - **Direct extraction** – *"What is written here?"*
  - **Understanding** – *"What’s happening in this image?"*
  - **Search** – *"Do we have similar images in our knowledge base?"*
  - **Action** – *Create ticket, fill form, classify, route, etc.*

---

## 2. Business Goals & Use Cases

### 2.1 Key Goals

- **Accuracy-first**
  - Reduce hallucinations by grounding responses in OCR text, detected objects, captions, and KB retrieval
- **Latency control**
  - Fast preview → deeper analysis only when required
- **Cost control**
  - Prefer open-source / free tools
  - GPU optional and on-demand

### 2.2 Primary Use Cases

#### A) User uploads an image → bot responds

- ID cards / invoices / forms → OCR + field extraction
- Error screenshots → OCR + troubleshooting lookup
- Asset/device photos → Object detection + issue classification

#### B) Existing image repository → semantic search

- Manuals, internal screenshots, product catalogs
- Vectorize images and metadata for similarity search

---

## 3. System Overview

### 3.1 End-to-End Flow (High Level)

```mermaid
flowchart TD
  U[User] --> FE[Web / Mobile UI]
  FE --> API[API Gateway]
  API --> VQ[Validation & Quality Checks]
  VQ --> PRE[Preprocessing]
  PRE --> OCR[OCR Engine]
  PRE --> VIS[Vision Models]
  OCR --> FUSE[Signal Fusion]
  VIS --> FUSE
  FUSE --> EMB[Embedding Generator]
  EMB --> VDB[(Vector DB)]
  FUSE --> META[(Metadata Store)]
  API --> RAG[RAG Retrieval]
  RAG --> LLM[VLM / LLM]
  LLM --> RESP[Final Answer]
```

**Key principle:** The chatbot never guesses directly from pixels. It answers only after structured signal extraction.

---

## 4. HLD Architecture

### 4.1 Logical Architecture (Layers)

| Layer | Responsibilities | Free Tooling |
|------|------------------|--------------|
| Client (Web/Mobile) | Capture, preview, compression, consent | Browser APIs, HTML5 |
| API Gateway | Auth, routing, rate-limit | FastAPI, Nginx |
| Image Pipeline Service | Preprocess, OCR, detection, captioning | OpenCV, Tesseract |
| Retrieval Layer | Vector search & filters | pgvector / Qdrant / Milvus |
| LLM/VLM Reasoning | Evidence-grounded response | LLaVA |
| Storage | Raw images & artifacts | MinIO, Postgres |
| Observability | Logs, metrics, traces | Prometheus, Grafana |

### 4.2 Deployment Topology

```mermaid
graph TB
  Client --> Gateway
  Gateway --> ImagePipeline
  Gateway --> LLM
  ImagePipeline --> Storage
  ImagePipeline --> VectorDB
  LLM --> VectorDB
  Gateway --> Monitoring
```

---

## 5. Detailed Processing Pipeline

### 5.1 Upload Quality Gate

**Fast validation checks:**

- File type allowlist (jpg, png, webp, pdf)
- Size limit & auto-resize
- Blur detection (Laplacian variance)
- Brightness / darkness check
- EXIF orientation fix
- Duplicate detection (hash)

**Why:** Prevents garbage-in → garbage-out and reduces compute cost.

---

### 5.2 Image Preprocessing (OpenCV)

- Resize to model-friendly resolution
- Noise reduction
- Contrast enhancement (OCR-friendly)
- Perspective correction (documents)
- Optional super-resolution (on demand)

---

### 5.3 OCR (Text Extraction)

**Tool:** Tesseract OCR

**Outputs:**

- Extracted text
- Word/line bounding boxes
- Confidence scores

---

### 5.4 Visual Understanding (Non-text)

#### Object Detection
- **YOLOv8** – fast, accurate (AGPL license)

#### Segmentation
- **Segment Anything (SAM / SAM2)** – pixel-level object masks

#### Image Captioning
- **BLIP** – converts image → descriptive text

#### Vision Chat / Reasoning
- **LLaVA** – image + text reasoning

---

### 5.5 Signal Fusion

A single structured payload is created:

| Field | Source | Example |
|------|-------|---------|
| ocr_text | Tesseract | "Windows error 0x..." |
| caption | BLIP / LLaVA | "Laptop screen showing error" |
| objects | YOLO | laptop, keyboard |
| regions | SAM | polygon masks |
| quality | OpenCV | blur=low |

All chatbot answers are grounded in this payload.

---

## 6. Storage & Vectorization Design

### 6.1 What to Store

- Raw image → Object storage (MinIO / S3)
- Derived artifacts:
  - OCR text
  - Captions
  - Detected objects
  - Embeddings

---

### 6.2 Vectorization Options

#### Option 1: Image Embeddings

- Use **CLIP / OpenCLIP**
- Best for image similarity search

#### Option 2: Text-First Embeddings

- OCR + caption → Sentence-Transformers
- Best for document-heavy images

**Recommended:** Store both image and text embeddings.

---

### 6.3 Retrieval Flow (Multimodal RAG)

```mermaid
flowchart LR
  Q[User Query] --> QE[Text Embed]
  IMG[User Image] --> IE[Image Embed]
  QE --> VDB
  IE --> VDB
  VDB --> K[Top-K Matches]
  K --> LLM
  LLM --> Answer
```

---

## 7. Free Tools & Stack

| Layer | Tool | Notes |
|-----|-----|------|
| Preprocessing | OpenCV | Mature & fast |
| OCR | Tesseract | Best for printed text |
| Captioning | BLIP | Auto tagging |
| Vision Chat | LLaVA | VLM reasoning |
| Embeddings | CLIP / OpenCLIP | Multimodal |
| Vector DB | pgvector / Qdrant / Milvus | Scale-based |
| Backend | FastAPI | Simple & reliable |
| Async Jobs | Celery / RQ | Heavy pipelines |

---

## 8. Tool Comparison Matrix

### OCR Tools

| Tool | Strengths | Weaknesses | Best Fit |
|----|-----------|------------|----------|
| Tesseract | Offline, multilingual | Poor handwriting | Docs, screenshots |

### Detection & Segmentation

| Tool | Strengths | Weaknesses | Best Fit |
|----|-----------|------------|----------|
| YOLOv8 | Fast & accurate | AGPL license | Real-time detection |
| SAM | Best segmentation | Heavy compute | Object isolation |

### Vector Databases

| Tool | Strengths | Weaknesses | Best Fit |
|----|-----------|------------|----------|
| pgvector | Simple ops | Not massive scale | Small/medium apps |
| Qdrant | Strong filters | Extra service | Search-heavy apps |
| Milvus | Highly scalable | Infra complexity | Large systems |

---

## 9. Security, Compliance & Retention

### Security Controls

- Virus scan on upload
- MIME + magic-byte validation
- Signed URLs for storage
- PII masking in logs
- RBAC-based access

### Retention Policy

| Data | Retention | Reason |
|----|----------|-------|
| Raw Images | 30–180 days | Privacy & cost |
| OCR Text | Same as image | PII |
| Embeddings | Same or shorter | Sensitive |

---

## 10. Observability & KPIs

**Metrics:**

- Upload success rate
- OCR confidence average
- Retrieval hit-rate
- p95 latency
- GPU utilization
- Cost per request

---

## 11. Risks & Mitigations

| Risk | Impact | Mitigation |
|----|-------|-----------|
| Low-quality images | Wrong answers | Quality gate UX |
| Hallucination | Trust loss | Evidence-based responses |
| License issues | Legal risk | Early OSS review |
| Data leakage | High | Encryption & strict logs |

---

## 12. Recommended Architecture Blueprints

### Option A – Enterprise-ready (Free)

- OpenCV + Tesseract
- BLIP + LLaVA
- OpenCLIP embeddings
- Qdrant Vector DB
- MinIO + Postgres

### Option B – Simplest Ops

- OCR + captions → text embeddings
- pgvector inside Postgres

### Option C – Massive Scale

- Milvus Vector DB
- Distributed async workers
- Aggressive caching

---

# Image Understanding Chatbot — Architecture Flow (Layman Version)

## 🧠 Architecture Flow — Explained Like a Human Story

Think of this system as a **very disciplined office assistant** who follows fixed steps every time someone gives them a photo.

---

## 🧍 Step 1: User uploads an image
**(Phone / Laptop / Browser)**

**What happens**
- A person uploads a photo, screenshot, or scanned document.

**Why this step exists**
- This is the entry point. Nothing fancy yet.

**Tools used**
- Browser upload (HTML, mobile camera)

---

## 🚪 Step 2: Frontend sends image to the backend (Server)

**What happens**
- The image is sent to the server safely.
- The user sees a preview immediately.

**Why**
- Fast feedback so the user knows the upload worked.

**Tools**
- Browser APIs  
- FastAPI

---

## 🛂 Step 3: Image Quality & Safety Check (Gatekeeper)

Before doing any smart work, the system asks:
> “Is this image even worth processing?”

**Checks**
- Is the file real?
- Is it blurry or dark?
- Is it a supported format?
- Is it a duplicate?

**Tools**
- OpenCV

**Why this step is critical**
- Bad image = bad answer  
- Saves time and money

---

## 🧹 Step 4: Image Cleaning (Like wiping glasses)

**What we do**
- Resize
- Remove noise
- Fix rotation
- Improve contrast

**Tools**
- OpenCV

---

## 📝 Step 5: Read the text inside the image (OCR)

**Examples**
- Bills
- Error messages
- Forms
- IDs

**Tool**
- Tesseract OCR

**Why**
- Accurate, offline, free

---

## 👀 Step 6: Understand what the image shows (Non-text)

Even if there is no text, images still have meaning.

**Examples**
- Laptop screen
- Phone
- Document
- Error screenshot

**Tools**
- YOLO (object detection)
- BLIP (image description)
- LLaVA (image + question reasoning)

---

## 🧩 Step 7: Combine everything into ONE understanding

**Merged data**
- Text from OCR
- Objects detected
- Image description
- Image quality

**Why**
- AI answers only from facts
- No guessing or hallucination

---

## 🗄️ Step 8: Save useful information

**What gets stored**
- Image
- Extracted text
- Image meaning
- Vector fingerprints

**Tools**
- MinIO / S3 (image storage)
- PostgreSQL (metadata)
- pgvector / Qdrant (similarity search)

---

## 🔍 Step 9: Search old knowledge (If needed)

If user asks:
> “Have we seen this before?”

**Tools**
- Vector DB (pgvector / Qdrant)

---

## 🗣️ Step 10: Final Answer to User

**How AI answers**
- Based on extracted facts
- Based on stored knowledge
- Never guesses

**Tool**
- LLM / VLM

---

## 🧰 Tool Summary

| Tool | What it does | Why chosen |
|----|----|----|
| OpenCV | Image cleaning & checks | Fast, free |
| Tesseract | OCR | Reliable, offline |
| YOLO | Object detection | Accurate |
| BLIP | Image caption | Simple |
| LLaVA | Image reasoning | Open-source |
| FastAPI | Backend | Stable |
| PostgreSQL | Data storage | Rock-solid |
| pgvector / Qdrant | Similarity search | Best fit |
| MinIO | Image storage | Cheap & safe |

---

## 🧩 One-Page Architecture Diagram (Mermaid)

```mermaid
flowchart TB

U["User uploads image"] --> FE["Web/Mobile UI\nTools: Browser APIs"]
FE --> API["API Gateway\nTools: FastAPI"]
API --> QC["Quality Check\nTools: OpenCV"]
QC -->|Bad| REJ["Reject & Re-upload"]
QC -->|Good| PRE["Preprocess Image\nTools: OpenCV"]
PRE --> OCR["OCR\nTools: Tesseract"]
PRE --> VIS["Visual Understanding\nTools: YOLO + BLIP + LLaVA"]
OCR --> FUSE["Signal Fusion"]
VIS --> FUSE
FUSE --> EMB["Embeddings\nTools: CLIP"]
EMB --> VDB["Vector DB\npgvector / Qdrant"]
FUSE --> META["Metadata\nPostgreSQL"]
FUSE --> OBJ["Image Storage\nMinIO"]
VDB --> AI["Answer Generation\nLLM/VLM"]
META --> AI
AI --> RESP["Final Answer"]
```

---

## 🎯 One-line summary

This architecture works like a careful human assistant who **checks, cleans, reads, understands, remembers, and then answers using only facts**.

# 🧾 OCR Engines – Complete Difference Table (All Practical Points)

## Compared OCRs
- Tesseract OCR  
- PaddleOCR  
- EasyOCR  
- OCRmyPDF  
- Keras-OCR  

---

## 🔍 1. General & Legal

| Point | Tesseract | PaddleOCR | EasyOCR | OCRmyPDF | Keras-OCR |
|---|---|---|---|---|---|
| License | Apache 2.0 | Apache 2.0 | Apache 2.0 | MIT | MIT |
| Free for commercial use | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| Vendor lock-in | ❌ None | ❌ None | ❌ None | ❌ None | ❌ None |
| Community maturity | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Enterprise adoption | Very high | High | Medium | Medium | Low |

---

## ⚙️ 2. Installation & Setup (Azure UAT VM)

| Point | Tesseract | PaddleOCR | EasyOCR | OCRmyPDF | Keras-OCR |
|---|---|---|---|---|---|
| Install complexity | Very low | Medium | Low | Very low | High |
| OS dependency pain | Low | Medium | Low | Low | High |
| Python required | ❌ Optional | ✅ Yes | ✅ Yes | ❌ Optional | ✅ Yes |
| GPU required | ❌ No | ❌ No (optional) | ❌ No | ❌ No | ⚠️ Recommended |
| Azure VM friendly | ✅ Excellent | ✅ Yes (tuning needed) | ✅ Yes | ✅ Yes | ⚠️ Risky |
| Docker friendly | ✅ Excellent | ✅ Yes | ✅ Yes | ✅ Yes | ⚠️ Heavy |

---

## 🧠 3. OCR Capability & Accuracy

| Point | Tesseract | PaddleOCR | EasyOCR | OCRmyPDF | Keras-OCR |
|---|---|---|---|---|---|
| Printed text accuracy | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| Screenshot accuracy | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| Handwritten text | ❌ Poor | ⚠️ Medium | ⚠️ Medium | ❌ Poor | ⚠️ Medium |
| Low-quality images | ⚠️ Medium | ⭐⭐⭐⭐ | ⭐⭐ | ⚠️ Medium | ⭐⭐ |
| Mixed fonts | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Curved text | ❌ Weak | ⚠️ Partial | ⚠️ Partial | ❌ No | ⚠️ Partial |

---

## 📐 4. Layout, Tables & Structure

| Point | Tesseract | PaddleOCR | EasyOCR | OCRmyPDF | Keras-OCR |
|---|---|---|---|---|---|
| Table detection | ❌ No | ✅ Yes | ❌ No | ❌ No | ❌ No |
| Form structure | ❌ No | ✅ Yes | ⚠️ Limited | ❌ No | ❌ No |
| Column awareness | ❌ No | ✅ Yes | ❌ No | ❌ No | ❌ No |
| Bounding boxes | ✅ Yes | ✅ Excellent | ⚠️ Basic | ❌ No | ⚠️ Yes |
| Layout ordering | ❌ No | ✅ Yes | ❌ No | ❌ No | ❌ No |

---

## 🌍 5. Language Support

| Point | Tesseract | PaddleOCR | EasyOCR | OCRmyPDF | Keras-OCR |
|---|---|---|---|---|---|
| Number of languages | 100+ | 80+ | 80+ | Same as Tesseract | Limited |
| Indic languages | ✅ Strong | ✅ Strong | ⚠️ Medium | Depends | ⚠️ Medium |
| Mixed languages | ⚠️ Partial | ✅ Good | ⚠️ Partial | ⚠️ Partial | ❌ Weak |
| Language auto-detect | ⚠️ Limited | ✅ Yes | ✅ Yes | ❌ No | ❌ No |

---

## 🚀 6. Performance & Resource Usage (CPU VM)

| Point | Tesseract | PaddleOCR | EasyOCR | OCRmyPDF | Keras-OCR |
|---|---|---|---|---|---|
| CPU speed | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| RAM usage | Low | Medium–High | Medium | Low | High |
| Parallel processing | ⚠️ Manual | ✅ Good | ⚠️ Limited | ❌ No | ⚠️ Manual |
| Large batch handling | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |

---

## 🧩 7. Integration & Pipeline Fit (Chatbot / RAG)

| Point | Tesseract | PaddleOCR | EasyOCR | OCRmyPDF | Keras-OCR |
|---|---|---|---|---|---|
| Easy to call from Python | ✅ Yes | ✅ Yes | ✅ Yes | ⚠️ Indirect | ⚠️ Yes |
| JSON-friendly output | ⚠️ Needs parsing | ✅ Yes | ✅ Yes | ❌ No | ⚠️ Yes |
| Confidence scores | ⚠️ Limited | ✅ Good | ✅ Good | ❌ No | ⚠️ Yes |
| Good for RAG grounding | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Chatbot-ready | ✅ Yes | ✅ Best | ⚠️ Yes | ⚠️ Limited | ❌ Poor |

---

## 🧪 8. Maintenance & Ops (Azure UAT → Prod)

| Point | Tesseract | PaddleOCR | EasyOCR | OCRmyPDF | Keras-OCR |
|---|---|---|---|---|---|
| Debuggability | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Upgrade risk | Low | Medium | Low | Low | High |
| Model version control | ❌ No | ✅ Yes | ⚠️ Limited | ❌ No | ⚠️ Manual |
| Ops complexity | Low | Medium | Low | Low | High |

---

## 🏁 9. Best-Fit Use Cases

| Use Case | Best OCR |
|---|---|
| Simple documents, screenshots | Tesseract |
| Forms, tables, structured docs | PaddleOCR |
| Quick demo / PoC | EasyOCR |
| Scanned PDFs | OCRmyPDF |
| Research / experiments | Keras-OCR |

---

## 🏆 Final Verdict (No Sugar-Coating)

| Scenario | Recommendation |
|---|---|
| Azure UAT VM, stability first | Tesseract OCR |
| Accuracy + future GPU scaling | PaddleOCR |
| Fastest MVP demo | EasyOCR |
| PDF-only pipeline | OCRmyPDF |
| Production system | ❌ Not Keras-OCR |

---

## 🎯 One-Line Decision Rule

**If you want boring, stable, and reliable → Tesseract.**  
**If you want smarter, layout-aware, and future-ready → PaddleOCR.**

**End of Document**

