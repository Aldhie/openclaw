# Deployment Decisions & Known Issues

> **PENTING:** Dokumen ini mencatat keputusan konfigurasi yang **disengaja** dan harus
> **DIPERTAHANKAN** saat greenfield ulang. Jangan di-revert tanpa memahami alasannya.
>
> Last updated: 2026-05-01 | Server: sg2-ded

---

## [KEPUTUSAN-001] `docker-compose.yml` — Hapus Nested Variable Fallback

| | |
|---|---|
| **Tanggal** | 2026-05-01 |
| **Status** | ✅ FINAL — PERTAHANKAN |
| **File** | `docker-compose.yml` (baris 29, 30, 93, 94) |

### Perubahan

```diff
- ${OPENCLAW_CONFIG_DIR:-${HOME:-/tmp}/.openclaw}
+ ${OPENCLAW_CONFIG_DIR}

- ${OPENCLAW_WORKSPACE_DIR:-${HOME:-/tmp}/.openclaw/workspace}
+ ${OPENCLAW_WORKSPACE_DIR}
```

### Alasan

Docker Compose v2.x **tidak support** nested `${VAR:-${FALLBACK:-default}}` syntax.
Error yang muncul bila di-revert ke upstream:

```
invalid interpolation format for services.openclaw-cli.volumes.[]:
"${HOME:-/tmp". You may need to escape any $ with another $.
```

Error ini terjadi di fase `==> Fixing data-directory permissions` dalam `setup.sh`
karena script memanggil `docker compose` dengan file yang masih punya nested syntax.

### Syarat Wajib

Variable berikut **HARUS** ada di `.env` sebelum `./scripts/docker/setup.sh`:

```env
OPENCLAW_CONFIG_DIR=/root/.openclaw
OPENCLAW_WORKSPACE_DIR=/root/.openclaw/workspace
```

### Verifikasi

```bash
grep -E 'OPENCLAW_CONFIG_DIR|OPENCLAW_WORKSPACE_DIR' .env
grep -n 'HOME:-' docker-compose.yml && echo "BERMASALAH!" || echo "✅ Bersih"
```

---

## [KEPUTUSAN-002] `Dockerfile` — Node 22 bukan Node 24

| | |
|---|---|
| **Tanggal** | 2026-05-01 |
| **Status** | ✅ FINAL — PERTAHANKAN |
| **File** | `Dockerfile` |

### Perubahan

```diff
- node:24-bookworm@sha256:3a09aa6354567619...
+ node:22-bookworm

- node:24-bookworm-slim@sha256:e8e2e91b1378f83c...
+ node:22-bookworm-slim

- OPENCLAW_NODE_BOOKWORM_SLIM_DIGEST="sha256:e8e2e91b..."
+ OPENCLAW_NODE_BOOKWORM_SLIM_DIGEST=""
```

### Alasan

Kompatibilitas native modules pada Node 24 menyebabkan build/runtime issues.
Node 22 (LTS) lebih stabil untuk ekosistem dependency yang dipakai.
Digest dikosongkan karena tidak pakai pinned SHA — lebih mudah dapat update.

---

## [KEPUTUSAN-003] `.dockerignore` — Exclude `migrate-claude` & `migrate-hermes`

| | |
|---|---|
| **Tanggal** | 2026-05-01 |
| **Status** | ✅ FINAL — PERTAHANKAN |
| **File** | `.dockerignore` |

### Perubahan

```diff
+ extensions/migrate-claude
+ extensions/migrate-hermes
```

### Alasan

Plugin `migrate-claude` dan `migrate-hermes` hanya muncul di test files (`*.test.ts`),
bukan di runtime production. Docs resmi OpenClaw tidak menyebut keduanya sebagai
requirement. Menyertakan keduanya menyebabkan build error karena dependency tidak
tersedia di image.

Diverifikasi via grep seluruh source tree — hanya ada di:
- `extensions/migrate-claude/*.test.ts`
- `extensions/migrate-hermes/*.test.ts`

### Verifikasi

```bash
grep -n 'migrate-claude\|migrate-hermes' .dockerignore
```

---

## [KEPUTUSAN-004] `docker-compose.override.yml` — Explicit Env Injection

| | |
|---|---|
| **Tanggal** | 2026-05-01 |
| **Status** | ✅ FINAL — PERTAHANKAN |
| **File** | `docker-compose.override.yml` |

### Pattern

Semua API keys harus di-map **eksplisit** di `override.yml` pada service
`openclaw-gateway` DAN `openclaw-cli`. Tanpa mapping eksplisit, container
tidak bisa baca env dari `.env` host.

```yaml
services:
  openclaw-gateway:
    environment:
      # Provider keys
      OPENROUTER_API_KEY: ${OPENROUTER_API_KEY:-}
      # NVIDIA NIM rotation keys (9 keys)
      NVIDIA_API_KEY: ${NVIDIA_API_KEY:-}
      NVIDIA_API_KEY_1: ${NVIDIA_API_KEY_1:-}
      # ... s/d NVIDIA_API_KEY_9
      # OKX Trade Kit
      OKX_API_KEY: ${OKX_API_KEY:-}
      OKX_SECRET_KEY: ${OKX_SECRET_KEY:-}
      OKX_PASSPHRASE: ${OKX_PASSPHRASE:-}
```

### Verifikasi setelah deploy

```bash
docker exec $(docker ps -qf name=openclaw-gateway) env | grep -E 'NVIDIA|OKX|OPENROUTER'
```

---

## [KEPUTUSAN-005] Gateway target di dalam container

| | |
|---|---|
| **Tanggal** | 2026-05-01 |
| **Status** | ✅ FINAL — PERTAHANKAN |

### Masalah

Bila CLI container mencoba connect ke `ws://127.0.0.1:18789`, koneksi gagal karena
loopback di dalam container tidak reach gateway di container lain.

### Fix

Pastikan `openclaw.json` menggunakan **service name**, bukan `127.0.0.1`:

```json
{
  "gateway": {
    "target": "openclaw-gateway:18789"
  }
}
```

---

## Checklist Pre-Build Greenfield

Jalankan ini sebelum setiap `./scripts/docker/setup.sh`:

```bash
# 1. Cek env variables wajib
grep -E 'OPENCLAW_CONFIG_DIR|OPENCLAW_WORKSPACE_DIR' .env
grep 'OKX_API_KEY=' .env | grep -v '^#'
grep 'NVIDIA_API_KEY=' .env | grep -v '^#'

# 2. Cek docker-compose.yml tidak punya nested fallback
grep -n 'HOME:-' docker-compose.yml && echo "❌ BERMASALAH" || echo "✅ Bersih"

# 3. Cek override.yml ada dan lengkap  
grep -c 'OKX_API_KEY\|NVIDIA_API_KEY' docker-compose.override.yml

# 4. Cek .dockerignore ada exclude
grep 'migrate-claude' .dockerignore

# 5. Cek Docker Compose versi
docker compose version
```

**Semua harus hijau sebelum build.**
