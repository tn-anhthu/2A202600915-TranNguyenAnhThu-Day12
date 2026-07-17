# Section 1 — Từ Localhost Đến Production

**Mã học viên** : 2A202600915

**Họ và tên**: Trần Nguyễn Anh Thư

## Mục tiêu học
- Hiểu tại sao "it works on my machine" là vấn đề
- Nhận ra sự khác biệt giữa dev và production environment
- Áp dụng 4 nguyên tắc 12-factor cơ bản

---

## Ví dụ Basic — Agent "Kiểu Localhost"

```
develop/
├── app.py          # ❌ Anti-patterns: hardcode secrets, no config, no health check
└── requirements.txt
```

### Chạy thử
```bash
cd develop
pip install -r requirements.txt
python app.py
# Truy cập: http://localhost:8000
```

### Những vấn đề trong code này:
1. API key hardcode trong code
2. Không có health check endpoint
3. Debug mode bật cứng
4. Không xử lý SIGTERM gracefully
5. Config không đến từ environment

---

## Ví dụ Advanced — 12-Factor Compliant Agent

```
production/
├── app.py          # ✅ Clean: config from env, health check, graceful shutdown
├── config.py       # ✅ Centralized config management
├── .env.example    # ✅ Template — không commit .env thật
└── requirements.txt
```

### Chạy thử
```bash
cd production
pip install -r requirements.txt
cp .env.example .env
# Sửa .env nếu cần
python app.py
```

### So sánh với Basic:

| | Basic (❌) | Advanced (✅) |
|--|-----------|--------------|
| Config | Hardcode trong code | Đọc từ env vars |
| Secrets | `api_key = "sk-abc123"` | `os.getenv("OPENAI_API_KEY")` |
| Port | Cố định `8000` | Từ `PORT` env var |
| Health check | Không có | `GET /health` |
| Shutdown | Tắt đột ngột | Graceful — hoàn thành request hiện tại |
| Logging | `print()` | Structured JSON logging |

---

## Câu hỏi thảo luận

1. Điều gì xảy ra nếu bạn push code với API key hardcode lên GitHub public?
Key bị lộ ngay, bị hacker/người lạ lợi dụng để làm các illegal activities, đào coin,.... gây thiệt hại nặng nề đến tài chính, uy tín. 

2. Tại sao stateless quan trọng khi scale?
Nếu state nằm trong memory của instance, phải đảm bảo cùng một user luôn vào cùng một instance ("sticky sessions") — điều này làm load balancing kém hiệu quả và khiến khi instance đó crash, user mất toàn bộ dữ liệu.

Stateless cho phép:
- Scale ngang tự do — thêm/bớt instance bất kỳ lúc nào
- Rolling deploy — tắt instance cũ, bật instance mới, không downtime
- Crash recovery — instance chết không mất state vì state ở Redis

3. 12-factor nói "dev/prod parity" — nghĩa là gì trong thực tế?
Dev và prod nên sử dụng cùng một tool, cùng một ngôn ngữ, cùng một version, cùng một thư viện, cùng một môi trường,....


---

## 5 anti-patterns

| Vấn đề | Dòng code | Tác hại |
|--|--|--|
| OPENAI_API_KEY hardcode | dòng 17 | Push GitHub → key bị lộ ngay |
| DATABASE_URL hardcode | dòng 18 | Credentials trong git history |
| print() log ra secret | dòng 34 | Using key: sk-hardcoded... hiện ra log |
| Không có /health endpoint | — | Platform không biết khi nào restart |
| host="localhost" + port=8000 cứng | dòng 51-52 | Không chạy được trong container/cloud |
| reload=True trong production | dòng 53 | Ngốn CPU, không ổn định |

