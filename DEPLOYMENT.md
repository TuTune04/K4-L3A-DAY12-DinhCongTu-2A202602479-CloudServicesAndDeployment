# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Dinh Cong Tu |
| Mã học viên | 2A202602479 |
| Repo | https://github.com/TuTune04/K4-L3A-DAY12-DinhCongTu-2A202602479-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-1b0f.up.railway.app |
| Platform | Railway (build từ `Dockerfile`, cấu hình `railway.toml`, region Southeast Asia, cùng region với Redis) |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự gán (đang là 8080), app đọc qua `${PORT}` trong CMD của Dockerfile |
| `AGENT_API_KEY` | ✅ | đặt qua `railway variables`, khóa production riêng, khác khóa local, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis service của Railway, reference variable `${{Redis.REDIS_URL}}` → mạng private `redis.railway.internal` |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

Chạy ngày 2026-09-28, `<URL>` = Public URL ở trên, API key truyền qua biến môi trường:

```
$ curl -i <URL>/health
HTTP/2 200 
content-type: application/json
{"status":"ok","service":"day12-agent","version":"1.0.0"}

$ curl -i <URL>/ready
HTTP/2 200 
content-type: application/json
{"status":"ready","redis":true}

$ curl -i -X POST <URL>/ask (không có API key)
HTTP/2 401 
{"detail":"invalid or missing API key"}

$ curl -i -X POST <URL>/ask (có API key)
HTTP/2 200 
{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud. (Mình đang nhớ 2 lượt trao đổi trước đó.)","user_id":"sv-test","history_length":2,"cost_usd":3.315e-05,"tokens":{"in":41,"out":45}}

$ rate limit — 15 lần liên tiếp
200 200 200 200 200 200 200 200 200 429 429 429 429 429 429
```

Lượt `/ask` có key thấy `history_length: 2` vì đã gọi thử 1 lần trước đó — lịch sử nằm trong Redis, không nằm trong process.
Rate limit: 9 request đầu `200` (cộng lượt ở trên = 10/phút), từ request thứ 11 trả `429`.

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

```
Không dùng — đã deploy thành công lên Railway.
```
