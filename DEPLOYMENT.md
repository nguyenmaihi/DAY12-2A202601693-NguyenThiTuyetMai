# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Thị Tuyết Mai |
| Mã học viên | 2A202601693 |
| Repo | https://github.com/nguyenmaihi/DAY12-2A202601693-NguyenThiTuyetMai |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-b2py.onrender.com |
| Platform | Render |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của platform |
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

```
HTTP/1.1 200 OK
content-type: application/json
{"status":"ok","service":"day12-agent","version":"1.0.0"}

HTTP/1.1 200 OK
content-type: application/json
{"status":"ready","redis":true}

HTTP/1.1 401 Unauthorized
{"detail":"invalid or missing API key"}

HTTP/1.1 200 OK
{"answer":"Deploy là quá trình đưa ứng dụng lên máy chủ hoặc hạ tầng cloud để phục vụ người dùng.","user_id":"sv-test","history_length":0,"cost_usd":0.0002,"tokens":{"in":20,"out":35}}

200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```
==========================================================================
CHẤM ĐIỂM TỰ ĐỘNG — K3 Ngày 12: Hạ Tầng Cloud & Deployment
==========================================================================

>>> CP1 — 12-Factor Config, Health & Logging
.............                                                                                [100%]
13 passed in 0.49s

>>> CP2 — Docker: multi-stage, bảo mật image
................                                                                             [100%]
16 passed in 3.69s

>>> CP3 — API Security: auth, rate limit, cost guard
......................                                                                       [100%]
22 passed in 0.22s

>>> CP4 — Scaling & Reliability: stateless, probe, shutdown
...................                                                                          [100%]
19 passed in 0.10s

>>> CP5 — Cloud Deployment: service chạy thật
........sssss                                                                                [100%]
8 passed, 5 skipped in 1.98s

>>> BONUS — CI/CD với GitHub Actions (không bắt buộc)
.............                                                                                [100%]
13 passed in 0.92s

==========================================================================
BẢNG ĐIỂM
==========================================================================
  CP1 — 12-Factor Config, Health & Logging         13/13 test            15.0/15
  CP2 — Docker: multi-stage, bảo mật image         16/16 test            15.0/15
  CP3 — API Security: auth, rate limit, cost guard 22/22 test            20.0/20
  CP4 — Scaling & Reliability: stateless, probe, shutdown 19/19 test            20.0/20
  CP5 — Cloud Deployment: service chạy thật        8/8 test (5 bỏ qua)   15.0/15
  Exercises — câu hỏi phản ánh                     10/10 câu             15.0/15
--------------------------------------------------------------------------
  Điểm phần bắt buộc                                                    100.0/100
  BONUS — CI/CD với GitHub Actions                 13/13 test            +10.0/10
--------------------------------------------------------------------------
  TỔNG CUỐI (trần 100)                                                  100.0/100
==========================================================================
  ⚠  điểm bonus bị cắt 10.0đ do chạm trần 100

  Xuất sắc. Service của bạn đã đạt chuẩn production.

Ghi chú: điểm exercises là điểm hoàn thành — chất lượng nội dung
sẽ được giảng viên chấm lại thủ công.
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
