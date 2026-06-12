# Day 12 Lab Report — Tran Nguyen Anh Thu (2A202600915)

**Họ và Tên**: Trần Nguyễn Anh Thư

**MSSV**: 2A202600915

---

## Part 1: Localhost vs Production

### Exercise 1.1 — 6 vấn đề tìm được trong `develop/app.py`

| # | Vấn đề | Dòng | Hậu quả |
|---|--------|------|---------|
| 1 | API key ghi thẳng vào code | 17 | Đẩy lên GitHub là lộ key ngay |
| 2 | Không có config management | 18 | Ai clone repo là có luôn password DB |
| 3 | `print()` in ra cả API key trong log | 34 | Ai xem log là thấy secret |
| 4 | Không có endpoint `/health` check endpoint| — | Cloud không biết app đang crash để khởi động lại |
| 5 | Port cố định — không đọc từ environment | 51 | Chỉ chạy được local, hông chạy được trong container hoặc trên cloud |

### Exercise 1.2 — Chạy thử develop version

**Lệnh chạy:**
**Lệnh test:**
```bash
curl "http://localhost:8000/ask?question=Hello" -X POST
```


**Kết quả API trả về:**
```json
{"answer": "Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

**Server log thực tế (vấn đề lộ secret):**
```
[DEBUG] Got question: Hello
[DEBUG] Using key: sk-hardcoded-fake-key-never-do-this   ← secret in ra thật
[DEBUG] Response: Tôi là AI agent...
```

App chạy được, nhưng không nên đưa lên cloud vì lý do trên.

### Exercise 1.3 — So sánh develop vs production

| Tính năng | Develop | Production | Tại sao quan trọng |
|-----------|---------|------------|-------------------|
| Config | Ghi cứng trong code | Đọc từ biến môi trường (`os.getenv`) | Thay đổi config mà không cần sửa code, không sợ lộ secret |
| Health check | Không có | Có `/health` + `/ready` | Cloud biết khi nào app cần được khởi động lại |
| Logging | `print()` in ra cả secret | JSON có cấu trúc, không log secret | Dễ tìm lỗi, an toàn hơn |
| Shutdown | Cúp điện đột ngột | Xử lý xong request hiện tại mới tắt | Người dùng đang gửi câu hỏi không bị mất câu trả lời |
| Host/Port | `localhost:8000` cứng | `0.0.0.0` + `PORT` từ env | Container và cloud tự inject port phù hợp |

**Production `/health` trả về:**
```json
{
  "status": "ok",
  "uptime_seconds": 2.7,
  "version": "1.0.0",
  "environment": "development",
  "timestamp": "2026-06-12T05:06:05.689476+00:00"
}
```

---

## Part 2: Docker

### Exercise 2.1 — Đọc Dockerfile cơ bản

**Trả lời 4 câu hỏi:**

1. **Base image là gì?** `python:3.11` — bản Python đầy đủ, nặng khoảng 1 GB
2. **Working directory là gì?** `/app` — thư mục làm việc bên trong container
3. **Tại sao copy `requirements.txt` trước khi copy code?**
   Docker build từng bước và lưu cache. Nếu chỉ đổi code (không đổi requirements), Docker dùng lại bước cài thư viện đã cache → build lần sau nhanh hơn nhiều. Nếu copy code và requirements cùng lúc thì mỗi lần đổi 1 dòng code cũng phải cài lại toàn bộ thư viện.
4. **CMD vs ENTRYPOINT khác nhau thế nào?**
   - `CMD` là lệnh mặc định, có thể bị thay bằng lệnh khác khi chạy container
   - `ENTRYPOINT` luôn chạy, không bị override được — dùng khi muốn container luôn chạy 1 chương trình cố định

### Exercise 2.2 — Image size (basic vs multi-stage)


| Image | Base | Kích thước ước tính |
|-------|------|---------------------|
| `my-agent:develop` | 413 MB | 1.67GB  |
| `my-agent:production` | 56.7MB | 262 MB |

### Exercise 2.3 — Tại sao multi-stage build nhỏ hơn?

Nhỏ hơn ~7.3 lần. Multi-stage build tách thành 2 stage:
- Stage 1 (Builder): cài gcc, build tools để compile thư viện
- Stage 2 (Runtime): chỉ copy kết quả đã compile, bỏ lại toàn bộ build tools
- Base image `python:3.11-slim` thay vì `python:3.11` full → bỏ bớt OS packages không cần thiết
- Kết quả: image chỉ chứa đúng những gì cần để *chạy*, không có thứ gì dùng để *build*nhỏ hơn và an toàn hơn (ít phần mềm hơn = ít lỗ hổng hơn).

### Exercise 2.4 — Docker Compose 
- `agent` — FastAPI AI agent (build từ Dockerfile multi-stage)
- `redis` — Cache cho session và rate limiting  
- `qdrant` — Vector database cho RAG
- `nginx` — Reverse proxy, load balancer (port 80)

```
Client (curl/browser)
│ port 80
▼
[ Nginx ]  ← reverse proxy
│ port 8000 (internal network)
▼
[ Agent ]  ← FastAPI app
│
├── [ Redis ]   ← cache, rate limit
└── [ Qdrant ]  ← vector DB
```

**Test thực chạy:**
```bash
curl http://localhost/health
# → {"status":"ok","uptime_seconds":134.3,"version":"2.0.0",...}

curl http://localhost/ask -X POST -H "Content-Type: application/json" \
  -d '{"question": "Explain microservices"}'
# → {"answer":"Tôi là AI agent được deploy lên cloud..."}
```

**Các service communicate thế nào?** Tất cả trong cùng internal Docker network — agent gọi Redis qua `redis://redis:6379`, gọi Qdrant qua `http://qdrant:6333`. Nginx nhận traffic từ ngoài, forward vào agent. Client không bao giờ gọi thẳng vào agent hay Redis.

---

## Part 3: Cloud Deployment

> **Lưu ý:** Phần này chưa thực hiện deploy thực tế. Ghi lại các bước sẽ làm.

### Exercise 3.1 — Deploy Railway

**URL:** https://lab2-deploy-production.up.railway.app

**Test thực chạy:**
```bash
curl https://lab2-deploy-production.up.railway.app/health
# → {"status":"ok","uptime_seconds":298.9,"platform":"Railway","timestamp":"..."}

curl https://lab2-deploy-production.up.railway.app/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Am I on the cloud?"}'
# → {"question":"Am I on the cloud?","answer":"Đây là câu trả lời từ AI agent (mock)...","platform":"Railway"}
```

**Screenshot:** [screenshots/running.png]


### Exercise 3.2 — So sánh railway.toml vs render.yaml

| | railway.toml | render.yaml |
|---|---|---|
| **Format** | TOML | YAML |
| **Deploy** | CLI (`railway up`) hoặc GitHub | Chỉ qua GitHub (Blueprint) |
| **Build** | Nixpacks (auto-detect) | Khai báo `buildCommand` rõ ràng |
| **Health check** | `healthcheckPath = "/health"` | `healthCheckPath: /health` |
| **Secrets** | Set qua CLI hoặc Dashboard | `sync: false` = set thủ công trên Dashboard, hoặc `generateValue: true` để Render tự sinh |
| **Redis** | Add-on riêng trên Dashboard | Khai báo thẳng trong file (`type: redis`) |
| **Region** | Chọn trên Dashboard | Khai báo trong file (`region: singapore`) |
| **Auto deploy** | Mặc định khi connect GitHub | `autoDeploy: true` |

**Điểm khác biệt chính:** `render.yaml` là Infrastructure as Code hoàn chỉnh hơn — khai báo cả Redis, region, secrets policy ngay trong file. `railway.toml` tối giản hơn, nhiều thứ config qua Dashboard.

---

## Part 4: API Security

### Exercise 4.1 — API Key authentication

**Test thực chạy:**
```bash
# Test 1: Không có key → 401
curl http://localhost:8000/ask -X POST -d '{"question":"Hello"}'
# → {"detail":"Missing API key. Include header: X-API-Key: "}

# Test 2: Sai key → 401
curl http://localhost:8000/ask -X POST -H "X-API-Key: wrong-key" -d '{"question":"Hello"}'
# → {"detail":"Invalid API key."}

# Test 3: Đúng key → 200
curl http://localhost:8000/ask -X POST -H "X-API-Key: secret-key-123" -d '{"question":"Hello"}'
# → {"question":"Hello","answer":"Agent đang hoạt động tốt!..."}
```

**API key check ở đâu?** Hàm `verify_api_key()` — FastAPI dependency, tự gọi trước mọi endpoint được bảo vệ.

**Sai key → gì xảy ra?** Trả về 401 ngay, không chạy đến phần gọi AI.

**Rotate key?** Thay `AGENT_API_KEY` trong env var, restart app — không cần sửa code.

### Exercise 4.2 — JWT authentication

**Lấy token:**
```bash
curl http://localhost:8000/auth/token -X POST \
  -H "Content-Type: application/json" \
  -d '{"username": "student", "password": "demo123"}'
# → {"access_token":"eyJhbGci...","token_type":"bearer","expires_in_minutes":60}
```

**Dùng token gọi API:**
```bash
curl http://localhost:8000/ask -X POST \
  -H "Authorization: Bearer " \
  -d '{"question": "Explain JWT"}'
# → {"answer":"...","usage":{"requests_remaining":9,"budget_remaining_usd":2.1e-05}}
```


### Exercise 4.3 — Rate limiting

**Algorithm:** Sliding window — 10 req/phút cho user thường, 100 req/phút cho admin.

**Test thực chạy 12 lần liên tiếp:**
```
Kết quả: [200, 200, 200, 200, 200, 200, 200, 200, 200, 429, 429, 429]
→ 9 lần thành công, 3 lần bị chặn
```

### Exercise 4.4 — Cost guard

Cost guard theo dõi số tiền đã tiêu mỗi tháng. Logic:

```python
def check_budget(user_id: str, estimated_cost: float) -> bool:
    month_key = datetime.now().strftime("%Y-%m")
    key = f"budget:{user_id}:{month_key}"
    
    current = float(r.get(key) or 0)
    if current + estimated_cost > 10:  # $10/tháng
        return False
    
    r.incrbyfloat(key, estimated_cost)
    r.expire(key, 32 * 24 * 3600)  # tự reset sau 32 ngày
    return True
```

Cách hoạt động: Mỗi lần user gửi câu hỏi, app ước tính chi phí gọi AI (dựa vào số từ). Nếu tổng cộng tháng này vượt $10 thì trả về lỗi 402 (Payment Required), không gọi AI nữa.

---

## Part 5: Scaling & Reliability

### Exercise 5.1 — Health checks thực chạy

```bash
curl http://localhost:8000/health
# → {"status":"ok","uptime_seconds":85.6,"version":"1.0.0","environment":"development",
#    "timestamp":"2026-06-12T09:40:33.022089+00:00",
#    "checks":{"memory":{"status":"ok","used_percent":75.9}}}

curl http://localhost:8000/ready
# → {"ready":true,"in_flight_requests":1}
```

**/health vs /ready khác nhau:**
- `/health` = "app còn sống không?" → cloud dùng để quyết định có restart container không
- `/ready` = "app sẵn sàng nhận request chưa?" → load balancer dùng để quyết định có route traffic vào không. Trả về 503 khi đang khởi động hoặc đang tắt


### Exercise 5.2 — Graceful shutdown

App xử lý SIGTERM (signal cloud gửi khi muốn tắt container):
```python
signal.signal(signal.SIGTERM, handle_sigterm)
```
Khi nhận SIGTERM → hoàn thành request đang xử lý → mới tắt. Nếu không có graceful shutdown, user đang chờ response sẽ nhận lỗi giữa chừng.

### Exercise 5.3 — Tại sao stateless quan trọng?

**Anti-pattern (state trong memory):**
```python
conversation_history = {}  # ❌ chỉ tồn tại trong RAM của 1 instance
```

**Production (state trong Redis):**
```python
history = redis.lrange(f"history:{session_id}", 0, -1)  # ✅ shared across all instances
```

**Bằng chứng thực tế:** Test 5.4 — 10 requests với cùng `session_id="test-session"` được phục vụ bởi 3 instances khác nhau nhưng `turn` vẫn tăng liên tục → state được share qua Redis, không mất khi đổi instance.

### Exercise 5.4 — Load balancing

**Chạy 3 instances:**
```bash
docker compose up --scale agent=3
```

**Test 10 requests — kết quả served_by:**
- Request 1:  instance-5ac1ed
- Request 2:  instance-76c6b0
- Request 3:  instance-ae0fb7
- Request 4:  instance-5ac1ed
- Request 5:  instance-76c6b0
- Request 6:  instance-ae0fb7
- Request 7:  instance-5ac1ed
- Request 8:  instance-76c6b0
- Request 9:  instance-ae0fb7
- Request 10: instance-5ac1ed

**Nhận xét:** Nginx phân tán request theo round-robin — 3 instances luân phiên đều đặn. Dù cùng `session_id`, conversation history vẫn đúng vì state lưu trong Redis dùng chung, không phải trong memory của từng instance.

### Exercise 5.5 — Stateless design (production app)

```bash
python test_stateless.py
```

**Kết quả:**
Session ID: 7ccb3d20-891a-436a-855f-906da38a7378
- Request 1: [instance-8888b7] → Q: What is Docker?
- Request 2: [instance-0b294e] → Q: Why do we need containers?
- Request 3: [instance-f47a26] → Q: What is Kubernetes?
- Request 4: [instance-8888b7] → Q: How does load balancing work?
- Request 5: [instance-0b294e] → Q: What is Redis used for?

Instances used: 3 khác nhau
- ✅ All requests served despite different instances!
- ✅ Session history preserved across all instances via Redis!

**Kết luận:** 5 requests cùng session được phục vụ bởi 3 instances khác nhau nhưng conversation history đầy đủ 10 messages (5 user + 5 assistant). State không nằm trong memory của instance nào — tất cả lưu trong Redis dùng chung. Đây là stateless design đúng nghĩa.
---

## Part 6: Final Project — Containerize Day 08 RAG Chatbot

**Repo được đóng gói:** [2A202600915-TranNguyenAnhThu-Day08](https://github.com/tn-anhthu/2A202600915-TranNguyenAnhThu-Day08)

**App:** RAG Chatbot — Pháp luật Ma tuý Việt Nam (Streamlit UI + Fireworks AI generation)

### Deployment Files

| File | Mô tả |
|------|-------|
| `Dockerfile` | Multi-stage build (builder + python:3.11-slim runtime), non-root user |
| `docker-compose.yml` | Local dev setup với HuggingFace cache volume |
| `.dockerignore` | Loại trừ data/, .env, __pycache__ |
| `.env.example` | Template không có secrets |
| `railway.toml` | Railway deploy config dùng Dockerfile |
| `DEPLOYMENT.md` | Hướng dẫn deploy và test commands |

### Docker Image

```
Image size: 491MB (content) / 2.35GB (disk with layers)
Base: python:3.11-slim (multi-stage)
Non-root user: appuser
PYTHONPATH=/app (fix import src.*)
CPU-only PyTorch (giảm ~1GB so với full CUDA)
```

### Public URL

https://day8-deploy-production.up.railway.app

### Health Check

```bash
curl https://day8-deploy-production.up.railway.app/_stcore/health
```

```
ok
```

### Production Checklist

| Tính năng | Có không | Ghi chú |
|-----------|----------|---------|
| Multi-stage Dockerfile | ✅ | builder + runtime stage |
| Non-root user | ✅ | `appuser` |
| HEALTHCHECK instruction | ✅ | `/_stcore/health` |
| Slim base image | ✅ | `python:3.11-slim` |
| .env không vào git | ✅ | `.gitignore` có `.env.local` |
| .dockerignore | ✅ | Loại trừ data/, .env, __pycache__ |
| .env.example | ✅ | FIREWORKS_API_KEY, WEAVIATE_URL, etc. |
| railway.toml | ✅ | builder = DOCKERFILE |
| Deploy thành công | ✅ | Railway — Active |
| Public URL | ✅ | day8-deploy-production.up.railway.app |
