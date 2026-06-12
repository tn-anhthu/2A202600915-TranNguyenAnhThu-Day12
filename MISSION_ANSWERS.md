# Day 12 Lab Report — Tran Nguyen Anh Thu (2A202600915)

---

## Part 1: Localhost vs Production

### Exercise 1.1 — 6 vấn đề tìm được trong `develop/app.py`

| # | Vấn đề | Dòng | Hậu quả |
|---|--------|------|---------|
| 1 | API key ghi thẳng vào code | 17 | Đẩy lên GitHub là lộ key ngay |
| 2 | Database URL có password ghi thẳng vào code | 18 | Ai clone repo là có luôn password DB |
| 3 | `print()` in ra cả API key trong log | 34 | Ai xem log là thấy secret |
| 4 | Không có endpoint `/health` | — | Cloud không biết app đang crash để khởi động lại |
| 5 | `host="localhost"` cứng trong code | 51 | Không chạy được trong container hoặc trên cloud |
| 6 | `reload=True` cứng trong code | 53 | Tốn tài nguyên, không ổn định khi chạy thật |

### Exercise 1.2 — Chạy thử develop version

**Lệnh chạy:**
```bash
cd 01-localhost-vs-production/develop
python app.py
curl -X POST "http://localhost:8000/ask?question=Hello"
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
| `my-agent:develop` | `python:3.11` (full) | 1.3 GB |
| `my-agent:production` | `python:3.11-slim` (multi-stage) | 24.4 MB |
| Tiết kiệm | — | ~70% |

### Exercise 2.3 — Tại sao multi-stage build nhỏ hơn?

Multi-stage build giống như đặt đồ ăn delivery: bếp (Stage 1) cần dao, thớt, nồi để nấu — nhưng khi mang đến tay khách (Stage 2) chỉ cần hộp đựng thức ăn, không cần mang cả bếp theo. Stage 1 cài `gcc`, `pip`, build tools để compile thư viện; Stage 2 chỉ copy kết quả đã compile, không copy công cụ → image nhỏ hơn và an toàn hơn (ít phần mềm hơn = ít lỗ hổng hơn).

### Exercise 2.4 — Docker Compose architecture

```
Client (trình duyệt/curl)
        │
        ▼ port 80
   [ Nginx ]  ← load balancer
        │
        ▼ port 8000 (internal)
   [ Agent ]  ← Python app
```

Services: `nginx` nhận traffic từ ngoài, chuyển vào `agent`. Khi scale thêm agent instances, Nginx tự phân tán request cho đều.

---

## Part 3: Cloud Deployment

> **Lưu ý:** Phần này chưa thực hiện deploy thực tế. Ghi lại các bước sẽ làm.

### Exercise 3.1 — Deploy Railway

```bash
cd 03-cloud-deployment/railway
npm i -g @railway/cli
railway login
railway init
railway variables set PORT=8000
railway variables set AGENT_API_KEY=my-secret-key
railway up
railway domain
```


### Exercise 3.2 — So sánh railway.toml vs render.yaml

| | railway.toml | render.yaml |
|---|---|---|
| Nền tảng | Railway | Render |
| Cách deploy | CLI (`railway up`) hoặc connect GitHub | Chỉ qua GitHub |
| Health check | Tự phát hiện | Khai báo trong file |
| Free tier | $5 credit | 750 giờ/tháng |

---

## Part 4: API Security

### Exercise 4.1 — API Key authentication

**Test thực tế:**

```bash
# Test 1: Không có key
curl -X POST http://localhost:8000/ask -d '{"question":"Hello"}'
# → 401: "Missing API key. Include header: X-API-Key: <your-key>"

# Test 2: Sai key  
curl -H "X-API-Key: wrong" -X POST http://localhost:8000/ask ...
# → 403: "Invalid API key."

# Test 3: Đúng key
curl -H "X-API-Key: secret-key-123" -X POST http://localhost:8000/ask ...
# → 200: {"question":"Hello","answer":"..."}
```

**Kết quả thực chạy:**
```
Test 1 → Status: 401  ✅
Test 2 → Status: 403  ✅
Test 3 → Status: 200  ✅
```

**API key được check ở đâu?** Hàm `verify_api_key()` trong `app.py` — đây là một "dependency" mà FastAPI tự gọi trước khi chạy bất kỳ endpoint nào được bảo vệ.

**Điều gì xảy ra nếu sai key?** App trả về HTTP 403 ngay lập tức, không bao giờ chạy đến phần gọi AI.

**Làm sao rotate key?** Chỉ cần thay giá trị biến môi trường `AGENT_API_KEY` và khởi động lại app — không cần sửa code.

### Exercise 4.2 — JWT authentication

**Lấy token:**
```bash
curl -X POST http://localhost:8000/auth/token \
  -d '{"username": "student", "password": "demo123"}'
```

**Kết quả thực chạy:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "expires_in_minutes": 60
}
```

**Dùng token gọi API:**
```bash
curl -H "Authorization: Bearer <token>" \
  -X POST http://localhost:8000/ask \
  -d '{"question": "Explain JWT"}'
# → 200: {"question":"Explain JWT","answer":"...","usage":{...}}
```

### Exercise 4.3 — Rate limiting

**Algorithm:** Sliding Window — đếm số request trong 60 giây gần nhất, nếu vượt quá giới hạn thì chặn.

**Test thực chạy 12 lần liên tiếp:**
```
Kết quả: [200, 200, 200, 200, 200, 200, 200, 200, 200, 429, 429, 429]
→ 9 lần thành công, 3 lần bị chặn
```

**Limit:** User thường = 10 req/phút, Admin = 100 req/phút

**Response khi bị chặn (429):**
```json
{
  "detail": {
    "error": "Rate limit exceeded",
    "limit": 10,
    "window_seconds": 60,
    "retry_after_seconds": 59
  }
}
```

**Làm sao bypass limit cho admin?** Dùng tài khoản có `role = "admin"` — app dùng instance `rate_limiter_admin` riêng với giới hạn cao hơn.

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
cd 05-scaling-reliability/develop
python app.py
```

**`/health` response:**
```json
{
  "status": "ok",
  "uptime_seconds": 2.7,
  "version": "1.0.0",
  "environment": "development",
  "timestamp": "2026-06-12T05:09:59.643285+00:00",
  "checks": {
    "memory": {"status": "ok", "used_percent": 82.4}
  }
}
```

**`/ready` response:**
```json
{"ready": true, "in_flight_requests": 1}
```

Hai endpoint này khác nhau:
- `/health` = "app còn sống không?" — cloud dùng để quyết định có restart không
- `/ready` = "app sẵn sàng nhận request chưa?" — load balancer dùng để quyết định có gửi traffic vào không. Trả về 503 khi app đang khởi động hoặc đang tắt.

### Exercise 5.2 — Graceful shutdown

**Log thực tế khi tắt app:**
```
INFO  Agent is ready!
...
INFO  Graceful shutdown initiated...
INFO  Shutdown complete
INFO  Received signal 15 — uvicorn will handle graceful shutdown
```

App nhận SIGTERM (signal 15 từ cloud/Docker), hoàn thành request đang xử lý, rồi mới tắt. Nếu không có graceful shutdown, người dùng đang nhờ AI trả lời sẽ nhận được lỗi giữa chừng.

### Exercise 5.3 — Tại sao stateless quan trọng?

**Vấn đề khi lưu state trong memory:**
```python
# Code sai — dữ liệu nằm trong RAM của 1 máy
conversation_history = {}

# User A gửi request 1 → vào Instance 1 → lưu history vào memory Instance 1
# User A gửi request 2 → vào Instance 2 → KHÔNG có history → trả lời như lần đầu
```

**Giải pháp — lưu state trong Redis:**
```python
# Bất kỳ instance nào cũng đọc được từ Redis
history = redis.lrange(f"history:{user_id}", 0, -1)
```

Khi có 3 instances, chúng share cùng 1 Redis nên dù request vào instance nào cũng có đủ thông tin. Khi 1 instance crash, 2 cái còn lại tiếp tục hoạt động bình thường, không mất dữ liệu.

### Exercise 5.4 — Load balancing

```bash
docker compose up --scale agent=3
```

3 container agent chạy song song, Nginx phân tán request theo round-robin. Khi 1 container chết, Nginx tự chuyển traffic sang 2 cái còn lại. Không downtime.

### Exercise 5.5 — Stateless design (production app)

Production app (`05-scaling-reliability/production/app.py`) có fallback thông minh:

```
✅ Connected to Redis     ← chạy đúng
⚠️  Redis not available — using in-memory store (not scalable!)  ← khi không có Redis
```

Khi không có Redis (ví dụ chạy local), app tự dùng in-memory dict — vẫn chạy được nhưng cảnh báo rõ là không scale được.

---

## Part 6: Final Project — Production Readiness Check

```bash
cd 06-lab-complete
python check_production_ready.py
```

**Kết quả:**

```
=======================================================
  Production Readiness Check — Day 12 Lab
=======================================================

📁 Required Files
  ✅ Dockerfile exists
  ✅ docker-compose.yml exists
  ✅ .dockerignore exists
  ✅ .env.example exists
  ✅ requirements.txt exists
  ✅ railway.toml or render.yaml exists

🔒 Security
  ✅ .env in .gitignore
  ✅ No hardcoded secrets in code

🌐 API Endpoints (code check)
  ✅ /health endpoint defined
  ✅ /ready endpoint defined
  ✅ Authentication implemented
  ✅ Rate limiting implemented
  ✅ Graceful shutdown (SIGTERM)
  ✅ Structured logging (JSON)

🐳 Docker
  ✅ Multi-stage build
  ✅ Non-root user
  ✅ HEALTHCHECK instruction
  ✅ Slim base image
  ✅ .dockerignore covers .env
  ✅ .dockerignore covers __pycache__

=======================================================
  Result: 20/20 checks passed (100%)
  🎉 PRODUCTION READY! Deploy nào!
=======================================================
```

**Tóm tắt những gì `06-lab-complete` đã có:**

| Tính năng | Có không | Ghi chú |
|-----------|----------|---------|
| Config từ env vars | ✅ | `app/config.py` dùng `os.getenv` |
| API key auth | ✅ | Header `X-API-Key` bắt buộc |
| Rate limiting | ✅ | 20 req/phút mặc định, đổi được qua env |
| Cost guard | ✅ | Giới hạn $5/ngày mặc định |
| `/health` endpoint | ✅ | Trả về uptime, version, request count |
| `/ready` endpoint | ✅ | Trả về 503 khi chưa sẵn sàng |
| Graceful shutdown | ✅ | Xử lý SIGTERM |
| JSON logging | ✅ | Mỗi request log ra dưới dạng JSON |
| Multi-stage Dockerfile | ✅ | `python:3.11-slim`, non-root user |
| Không có hardcoded secrets | ✅ | Không có `sk-`, `password123` trong code |
| `.env` không vào git | ✅ | Đã có trong `.gitignore` |
