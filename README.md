# Finly AI Service — Quickstart & Operations

FastAPI service untuk **Smart Input AI** (scan struk/nota Indonesia → ekstrak `merchant`, `total`, `transaction_date`). Dipanggil oleh Express (`/api/scan/receipt`), bukan langsung oleh React.

> **Audience**: Full-stack devs (React + Express), AI Engineer (saya), ops. Cari item kamu via TOC di bawah.
>
> **Versi pipeline**: `hybrid-v3` (PaddleOCR + Qwen2-VL-2B-Instruct 4-bit grounded + rule extractor, best-of-both per field).
>
> Detail rancangan & rasional: lihat `claude.md` di repo root.

## Daftar Isi

1. [Arsitektur ringkas](#1-arsitektur-ringkas)
2. [Quickstart — jalankan di laptop dev](#2-quickstart--jalankan-di-laptop-dev)
3. [Quickstart — full-stack devs (yang tidak menyentuh AI)](#3-quickstart--full-stack-devs)
4. [Environment variables](#4-environment-variables)
5. [Endpoint & contract](#5-endpoint--contract)
6. [Integrasi Express ↔ FastAPI](#6-integrasi-express--fastapi)
7. [Testing](#7-testing)
8. [Phase 6 hardening — toggles & operasional](#8-phase-6-hardening--toggles--operasional)
9. [Ops scripts](#9-ops-scripts)
10. [Troubleshooting](#10-troubleshooting)
11. [Roadmap & known gaps](#11-roadmap--known-gaps)

---

## 1. Arsitektur ringkas

```
React (Vite, 5173) ── multipart ──► Express (5000) ── multipart ──► FastAPI (8000)
                                       │  /api/scan/receipt              /v1/scan
                                       │  - JWT user verify              - PaddleOCR (GPU)
                                       │  - multer 8 MB limit            - Qwen2-VL grounded
                                       │  - forward + DB audit           - rule extractor
                                       │                                 - merge per field
                                       ▼
                                    MySQL
                                    (scan_jobs, transactions)
```

- **React** — UI scan (BELUM dibuat, lihat §11).
- **Express** — gateway, JWT user auth, multer upload, simpan ke `scan_jobs`, forward ke FastAPI dengan internal token.
- **FastAPI** — stateless inference. Tidak akses DB. Hanya verify internal token.
- **MySQL** — `scan_jobs` (audit trail) + `transactions` (data final user).

---

## 2. Quickstart — jalankan di laptop dev

### Prasyarat

- **Hardware**: NVIDIA GPU dengan ≥ 4 GB VRAM (rekomendasi RTX 4050 6 GB), CUDA 12.x.
- **Software**: Python 3.11, Node ≥ 18, MySQL 8, `git`. Bila WSL2: lihat [Troubleshooting](#10-troubleshooting).
- **Disk**: ~5 GB free (model Qwen2-VL pertama kali download ~4 GB).

### Setup AI service

```bash
# Dari repo root
cd /path/to/Capstone-dicoding
python3.11 -m venv .venv
source .venv/bin/activate

# Install AI deps
pip install -r requirements.txt

# CUDA-specific: torch sesuai driver lokal (cek nvidia-smi → "CUDA Version: …")
pip install --index-url https://download.pytorch.org/whl/cu121 \
  torch==2.4.1 torchvision==0.19.1

# PaddleOCR GPU (sesuaikan suffix `cu120`/`cu121` dgn CUDA-mu):
pip install paddlepaddle-gpu==2.6.2.post120 \
  -i https://www.paddlepaddle.org.cn/packages/stable/cu120/

# Verifikasi GPU
python -c "import torch; print('cuda:', torch.cuda.is_available(), torch.cuda.get_device_name(0))"
```

### Setup Express + DB

```bash
# Dari repo root
npm install
mysql -u root -p < migrations/001_create_auth_tables.sql
mysql -u root -p < migrations/002_create_scan_and_transactions.sql

cp .env.example .env                  # edit DB creds + JWT_SECRET + AI_INTERNAL_TOKEN
cp ai_service/.env.example ai_service/.env  # AI_INTERNAL_TOKEN HARUS sama dgn root .env
```

**Generate token random** (pakai untuk `AI_INTERNAL_TOKEN` di kedua `.env`):
```bash
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

### Run

Pakai **dua terminal**:

```bash
# Terminal 1: AI service (port 8000)
cd ai_service
source ../.venv/bin/activate
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

```bash
# Terminal 2: Express (port 5000)
npm run dev
```

### Smoke test

```bash
# Health
curl http://127.0.0.1:5000/api/health
curl http://127.0.0.1:8000/v1/health

# Register user (Express)
curl -X POST http://127.0.0.1:5000/api/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"full_name":"Demo","email":"demo@finly.test","password":"demo1234"}'
# → { "data": { "token": "<JWT>" } }

# Scan struk pakai JWT user
TOKEN="<JWT dari register>"
curl -X POST http://127.0.0.1:5000/api/scan/receipt \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@data/image/struk/1.jpg"
```

Lebih banyak contoh curl di [`docs/curl-examples.sh`](../docs/curl-examples.sh).

---

## 3. Quickstart — full-stack devs

Kalau kamu hanya kerja di React/Express dan **tidak menyentuh AI service**:

1. Pastikan AI service jalan di port 8000 (minta AI Engineer untuk start, atau pakai `LOAD_OCR_ON_STARTUP=false` + `LOAD_VLM_ON_STARTUP=false` supaya startup cepat tapi pipeline tidak akan jalan — useful untuk test contract saja).
2. Kontrak API kamu hanya peduli ke **Express endpoint** `POST /api/scan/receipt`. Lihat §5.
3. Untuk dev fast iteration, full-stack devs boleh pakai dummy AI server. Salin file `ai_service/scripts/stress.py` style stub atau panggil `/v1/scan` real dengan gambar dataset.

**Tugas yang masih open di sisi full-stack** (lihat §11):
- Tombol "Smart Input AI" + halaman Tambah Transaksi
- Endpoint `POST /api/transactions` di Express

---

## 4. Environment variables

### Root `.env` (Express)
| Var | Default | Catatan |
|---|---|---|
| `PORT` | 5000 | Port Express |
| `CORS_ALLOWED_ORIGINS` | `http://127.0.0.1:5173,...` | Origin React Vite |
| `DB_HOST` / `DB_PORT` / `DB_USER` / `DB_PASSWORD` / `DB_NAME` | localhost / 3306 / root / "" / `finly_db` | MySQL |
| `JWT_SECRET` | **change_this** | Untuk sign user JWT |
| `JWT_EXPIRES_IN` | `7d` | Expiry user JWT |
| `AI_SERVICE_URL` | `http://127.0.0.1:8000` | Base URL FastAPI |
| `AI_INTERNAL_TOKEN` | **change_this** | **HARUS identik dengan `ai_service/.env`** |
| `AI_SERVICE_TIMEOUT_MS` | `30000` | Timeout aiClient → FastAPI |

### `ai_service/.env`
| Var | Default | Catatan |
|---|---|---|
| `APP_ENV` | `local` | `production`/`staging`/`local` — kontrol behavior dev-only flags |
| `APP_DEBUG` | `true` | Enables `/docs` (Swagger UI) |
| `HOST` / `PORT` | 127.0.0.1 / 8000 | |
| `AI_INTERNAL_TOKEN` | **change_this** | Sama persis dgn root `.env` |
| `DEV_DISABLE_AUTH` | `false` | Hanya valid kalau `APP_ENV=local`; bypass token check di Swagger |
| `CORS_ALLOWED_ORIGINS` | Express origins | FastAPI biasanya hanya dipanggil Express |
| `USE_GPU` | `true` | Set `false` untuk CPU-only (lambat) |
| `PADDLEOCR_LANG` | `id` | Bahasa OCR primary |
| `MAX_IMAGE_BYTES` | `8388608` | 8 MB batas request |
| `VLM_MODEL_ID` | `Qwen/Qwen2-VL-2B-Instruct` | HuggingFace model ID |
| `VLM_LORA_DIR` | `""` | Path LoRA adapter (kosong = pakai base) |
| `LOAD_OCR_ON_STARTUP` | `true` | Eager-load PaddleOCR |
| `LOAD_VLM_ON_STARTUP` | `false` | Eager-load Qwen2-VL (~50s startup, request pertama langsung warm) |
| `WARMUP_ON_STARTUP` | `false` | **Phase 6 Batch C** — sintetis warmup setelah load |
| `MAX_CONCURRENT_INFER` | `1` | **Phase 6 Batch B** — slot GPU paralel |
| `MAX_QUEUE_DEPTH` | `4` | Max antrian sebelum 503 |
| `INFER_QUEUE_TIMEOUT_S` | `25.0` | Timeout tunggu giliran GPU |
| `DRIFT_WATCH_ENABLED` | `false` | **Phase 6 Batch D** — simpan replay set |
| `DRIFT_WATCH_DIR` | `replay` | Folder relatif ke `ai_service/` |
| `DRIFT_WATCH_MAX` | `100` | Cap jumlah record |

⚠️ **Privasi drift watch**: gambar struk = data finansial user. Hanya nyalakan setelah ada consent + retention plan eksplisit.

---

## 5. Endpoint & contract

### `POST /api/scan/receipt` (React → Express)

**Request**
```http
POST /api/scan/receipt
Authorization: Bearer <user_jwt>
Content-Type: multipart/form-data
Body: file=<image>      # jpg/png/webp, max 8 MB
```

**Response 200**
```json
{
  "success": true,
  "scan_job_id": 42,
  "request_id": "uuid-v4-hex",
  "data": {
    "merchant": "Indomaret",
    "total": 47500,
    "transaction_date": "2025-09-12",
    "confidence": { "merchant": 0.97, "total": 0.92, "transaction_date": 0.84 },
    "source": { "merchant": "rule_whitelist", "total": "agreement", "transaction_date": "rule" },
    "doc_type": "struk",
    "processing_ms": 3245,
    "model_version": "hybrid-v3"
  }
}
```

**Response error**
| Status | Kondisi | UI behavior yang disarankan |
|---|---|---|
| 400 | File field kosong | Validasi form |
| 401 | User JWT tidak valid / expired | Redirect login |
| 413 | File > 8 MB | Toast "ukuran terlalu besar" |
| 415 | MIME bukan jpeg/png/webp | Toast "format tidak didukung" |
| **503** | **AI service overloaded** (queue penuh / timeout giliran GPU) | **Toast "coba lagi", retry boleh otomatis 1× setelah 2 dtk** |
| 502 | AI service error lain | Toast "scan gagal, silakan input manual" |
| 504 | AI service tidak respon | Toast "scan gagal, input manual" |

**Penting untuk frontend**: confidence < 0.6 → tampilkan indikator kuning ("perlu review"). User WAJIB bisa edit `merchant`, `total`, `transaction_date` sebelum submit ke `POST /api/transactions`.

### `POST /v1/scan` (Express → FastAPI, internal)

```http
POST /v1/scan
Authorization: Bearer <AI_INTERNAL_TOKEN>
X-Request-Id: <uuid>          # optional, di-echo di response header
Content-Type: multipart/form-data
Body: file=<image>
```

Response sukses sama seperti `data` di atas, plus field opsional `replay_path: string|null` (path internal AI service kalau drift watch aktif).

### `GET /v1/health`

```bash
curl http://127.0.0.1:8000/v1/health | jq
```

Memberi status: `cuda`, `paddle`, `paddleocr`, `vlm`, `metrics` (rolling p50/p95/error_rate), `concurrency` (gate inflight/capacity), `warmup` (status startup warmup), `drift_watch` (count/cap).

---

## 6. Integrasi Express ↔ FastAPI

| Komponen | File | Fungsi |
|---|---|---|
| Receive multipart | `src/middleware/upload.js` | multer memory storage, MIME whitelist, max 8 MB |
| Verify user JWT | `src/middleware/authJwt.js` | `req.user.id` |
| Audit row | `src/controllers/scanController.js` | INSERT `scan_jobs(pending)` → call AI → UPDATE done/failed |
| HTTP client | `src/services/aiClient.js` | native fetch + FormData (Node 18+), forward bearer token, timeout 30s |
| Error mapping | `src/services/aiClient.js` | 401→500, 503→503 (passthrough), other→502 |

**Audit trail di MySQL** (lihat `migrations/002_create_scan_and_transactions.sql`):
- `scan_jobs(id, user_id, image_path, image_size_bytes, image_mime, status, result_json, model_version, processing_ms, error_message, …)`
- `transactions(id, user_id, scan_job_id, amount, currency, note, transaction_date, source, …)`

`scan_jobs.image_path` diisi otomatis kalau drift watch aktif di FastAPI; `null` kalau tidak.

---

## 7. Testing

### Run tests

```bash
cd ai_service
source ../.venv/bin/activate
pytest                              # semua
pytest tests/unit -q                # unit only (cepat, tanpa GPU)
pytest tests/integration -q         # integration (TestClient, stub pipeline)
pytest -k currency                  # filter by name
pytest -v --tb=short                # verbose + short traceback
```

Coverage:

```bash
pytest --cov=app --cov-report=term-missing
```

### Apa yang di-cover

- ✅ **Unit pure** — `currency.parse_idr`, `dates.parse_indonesian_date`, `extractor.extract_*` (rule logic), `merchant_dict.fuzzy_match_merchant`.
- ✅ **Phase 6 modules** — `metrics`, `concurrency.InferenceGate`, `request_context`, `drift_watch.DriftWatcher`.
- ✅ **API smoke** — `/v1/health` schema, `/v1/scan` auth/MIME/size validation paths, 503 saat gate saturated. Pakai stub pipeline (tidak load PaddleOCR/Qwen).

### Apa yang TIDAK di-cover di pytest

- ❌ Real PaddleOCR run — perlu GPU env benar (cuDNN, libcuda). Pakai `scripts/scan_one.py` atau `scripts/stress.py` untuk validasi end-to-end.
- ❌ Real Qwen2-VL inference — model ~4 GB, lambat. Pakai `scripts/scan_one.py` untuk smoke manual.
- ❌ Eval akurasi — pakai `ml/training/hybrid_eval.py` di subset Indonesia.

---

## 8. Phase 6 hardening — toggles & operasional

Phase 6 memberi 4 fitur production-grade. Default semua **OFF/konservatif** — service jalan persis seperti tanpa Phase 6 kalau flag tidak di-enable.

### Batch A — Observability (selalu aktif)

- Setiap response punya header `X-Request-Id` (echo dari request, atau auto-generate UUID).
- Log `ai_service.request` per request: `method=… path=… status=… ms=… gpu_mb=… bytes_in=…`.
- Log lain (pipeline, OCR, dll.) sudah ber-prefix `[rid=<request_id>]`.
- `/v1/health.metrics` punya rolling p50/p95/error_rate window 200 req.

### Batch B — Concurrency gate

```env
MAX_CONCURRENT_INFER=1          # 6 GB VRAM tipis — JANGAN naikkan kecuali tahu
MAX_QUEUE_DEPTH=4               # antrian max sebelum 503
INFER_QUEUE_TIMEOUT_S=25        # < 30s timeout client Express
```

`/v1/health.concurrency.inflight` real-time, useful untuk dashboard.

### Batch C — Warm-up startup

Set saat production deploy:
```env
LOAD_OCR_ON_STARTUP=true
LOAD_VLM_ON_STARTUP=true
WARMUP_ON_STARTUP=true          # bayar ~50s sekali, request pertama warm
```

`/v1/health.warmup` menunjukkan `ran=true, ok=true, ms=…`.

### Batch D — Drift watch (opt-in)

```env
DRIFT_WATCH_ENABLED=true        # NYALAKAN HANYA setelah keputusan privasi
DRIFT_WATCH_MAX=100
DRIFT_WATCH_DIR=replay
```

Setelah aktif, `replay/` akan terisi dengan `<sha256>.jpg` + `manifest.jsonl`. Lihat §9.

---

## 9. Ops scripts

Semua jalan dari `ai_service/` directory.

### `scripts/stress.py` — load test
```bash
python scripts/stress.py --n 50 --concurrency 4 --report stress_v1.json
python scripts/stress.py --target http://127.0.0.1:8000 --source ../data/image/nota --n 100
python scripts/stress.py --help
```
Output: per-request log + summary (status counts, p50/p95/p99, peak GPU mem, rps). Pakai untuk validasi pre-rilis.

### `scripts/drift_replay.py` — re-run pipeline atas replay set
```bash
python scripts/drift_replay.py
python scripts/drift_replay.py --limit 20 --report drift_after_v2.json
```
Diff per field: `same` / `source_changed` / `changed` / `added` / `lost`. Pakai sebelum push perubahan ekstraktor / VLM.

### `scripts/drift_purge.py` — kelola replay set
```bash
python scripts/drift_purge.py --status
python scripts/drift_purge.py --keep-last 50         # rotate
python scripts/drift_purge.py --all --yes            # wipe (privasi/cleanup)
```

### `ml/training/scan_one.py` — manual smoke test 1 gambar
```bash
python ml/training/scan_one.py path/to/struk.jpg
```
Print intermediate (preprocess size, OCR lines, VLM raw, merge result) — useful untuk debug 1 gambar bermasalah.

---

## 10. Troubleshooting

### `paddleocr` SIGSEGV / ImportError di WSL2
Penyebab umum: cuDNN 8 vs 9 mismatch, libcuda.so path. Lihat memory `env_wsl_paddle.md`. Workaround sementara: set `LOAD_OCR_ON_STARTUP=false` untuk start service tanpa OCR (hanya untuk test contract API).

### Cold start ~50s di request pertama
Normal — Qwen2-VL 4-bit JIT compile + first generate. Hilangkan dengan:
```env
LOAD_VLM_ON_STARTUP=true
WARMUP_ON_STARTUP=true
```

### OOM saat inference
Cek `MAX_CONCURRENT_INFER=1`. Kalau set >1, GPU 6 GB akan OOM. Kurangi `max_pixels` di `app/services/vlm.py` (di processor) sebagai opsi terakhir.

### 401 dari FastAPI tapi token sudah benar
- Cek `AI_INTERNAL_TOKEN` di `ai_service/.env` **identik karakter** dengan root `.env`.
- Atau enable bypass dev-only: `APP_ENV=local DEV_DISABLE_AUTH=true` (jangan di production).

### Express 502 saat AI service hidup
- Cek `AI_SERVICE_URL` di root `.env` cocok dengan host:port FastAPI.
- Cek FastAPI `/v1/health` — kalau status `degraded`, ada problem CUDA/Paddle.
- Cek log Express + FastAPI dengan `X-Request-Id` yang sama untuk korelasi.

### Port 8000 / 5000 sudah dipakai
```bash
lsof -i :8000   # cari PID
kill <PID>
# atau ubah PORT di .env
```

### Anotasi 600 hilang setelah git pull
`ml/data/` mungkin di-overwrite. Backup `annotations.jsonl` + `splits/` sebelum pull.

---

## 11. Roadmap & known gaps

### Item AI Engineer (saya)
- [x] Pipeline hybrid-v3 + Phase 6 hardening (Batch A/B/C/D)
- [x] Ops tooling (stress, drift_replay, drift_purge)
- [x] Test suite unit + integration (file ini)
- [ ] Phase 3 LoRA fine-tune — di-defer; bisa diaktifkan kalau metrik real-user menunjukkan butuh
- [ ] Eval dataset penuh post-deploy (perlu real receipts dari demo)

### Item Full-stack devs (BLOCKER untuk demo end-to-end)
- [ ] **React UI** — tombol "Smart Input AI", modal kamera/gallery, halaman Tambah Transaksi dengan pre-fill + indikator confidence
- [ ] **`POST /api/transactions`** — Express endpoint untuk insert ke tabel `transactions` (saat user submit dari halaman Tambah Transaksi)
- [ ] End-to-end test dengan struk asli (foto tim sebelum demo)

### Kontak
AI Engineer — saya (lihat git log `ai_service/`). Untuk perubahan kontrak API: koordinasikan via `openapi.yaml` + bump versi di sana.

Detail rancangan & rasional yang lebih dalam: `claude.md` di root repo.
