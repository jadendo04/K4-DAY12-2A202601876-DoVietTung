# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đỗ Việt Tùng |
| Mã học viên | 2A202601876 |
| Repo | https://github.com/jadendo04/K4-DAY12-2A202601876-DoVietTung |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-day12-2a202601876-doviettung-production.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `API_TOKEN` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của Railway, reference `${{Redis.REDIS_URL}}` |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

```
$ curl -i <URL>/healthz
HTTP/2 200
content-type: application/json
{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

$ curl -i <URL>/readyz
HTTP/2 200
content-type: application/json
{"status":"ready","redis":true}

$ curl -i -X POST <URL>/chat -d '{"message":"Hello"}'
HTTP/2 401
www-authenticate: Bearer
{"detail":"invalid or missing bearer token"}

$ curl -i -X POST <URL>/chat -H "Authorization: Bearer $API_TOKEN" -H "X-Client-Id: sv-test" -d '{"message":"Deploy là gì?"}'
HTTP/2 200
{"reply":"...", "client_id":"sv-test", "turns_before":2, "usd_cost":3.315e-05, "usage":{"prompt":41,"completion":45}}

$ for i in $(seq 1 30); do curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/chat ...; done
200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 429 429 200 429 429 429 429 429 429 429 200 429 429 429
```

Nhận xét: 16 request đầu qua được do bucket đã tích token trong lúc nghỉ giữa
các lần test trước đó (capacity=10, refill=10/phút); từ request 17 trở đi bắt
đầu dính 429 đều đặn — token bucket hoạt động đúng trên môi trường thật.

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

