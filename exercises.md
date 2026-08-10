# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
> Cách trả lời: thay dòng mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thị Tuyết Mai  Mã học viên: 2A202601693

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu đặt giá trị mặc định là `"changeme"`, khi deploy lên Cloud mà quên cấu hình biến môi trường `AGENT_API_KEY`, ứng dụng vẫn khởi động bình thường. Kẻ tấn công hoặc các bot quét API có thể sử dụng key mặc định `"changeme"` để gọi API miễn phí, tiêu tốn toàn bộ hạn ngạch/ngân sách LLM và bạn chỉ nhận ra khi hóa đơn gửi về. Khi không để giá trị mặc định, app ném `ValidationError` và crash ngay lúc khởi động (Fail fast), giúp ta phát hiện và sửa lỗi cấu hình ngay lập tức trong quá trình deploy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T10:05:00+00:00", "user_id": "sv01", "tokens_in": 120, "tokens_out": 45, "cost_usd": 0.00033}
```

Hai việc làm được với log JSON mà print không làm được:
1. Cho phép các hệ thống giám sát log tập trung (như Datadog, CloudWatch, Loki) tự động parse, truy vấn và tổng hợp các chỉ số theo trường (ví dụ: tính tổng chi phí `cost_usd` của từng `user_id` trong ngày).
2. Dễ dàng thiết lập các quy tắc cảnh báo tự động (alert rules) khi có bất thường (ví dụ: cảnh báo khi `cost_usd` của một request vượt ngưỡng, hoặc đếm tỷ lệ log có `level: error` trong 5 phút).

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | ~185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Bản 1 stage sử dụng base image `python:3.11` đầy đủ chứa trình biên dịch gcc, C++ toolchains, build-essential, package manager caches, header files và các file tĩnh không cần thiết cho runtime. Trong bản Multi-stage, stage `builder` dùng để cài đặt thư viện vào thư mục `/install`, còn stage `runtime` sử dụng `python:3.11-slim` và chỉ copy các file thư viện đã cài đặt sang, loại bỏ toàn bộ compiler, apt cache và tooling nặng, giúp dung lượng giảm hơn 800 MB.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại:
- Các layer trước `COPY app ./app` (như `FROM`, `WORKDIR`, `COPY requirements.txt`, `RUN pip install`) đều được tái sử dụng từ Docker cache.
- Chỉ có layer `COPY app ./app` và các bước sau nó mới phải build lại.
Nếu đặt `COPY . .` lên trước `RUN pip install`, mỗi khi sửa dù chỉ 1 ký tự trong source code, cache của layer `COPY . .` bị vô hiệu hóa, khiến Docker buộc phải thực thi lại lệnh `RUN pip install` từ đầu, làm tăng thời gian build từ vài giây lên vài phút mỗi lần deploy.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Kẻ tấn công phát hiện một lỗ hổng thực thi mã từ xa (RCE - Remote Code Execution) hoặc arbitrary file write trong code Python / thư viện bên thứ 3.
2. Nếu container chạy với quyền root (mặc định), payload của kẻ tấn công được thực thi dưới quyền UID 0 (root) bên trong container.
3. Nếu có lỗ hổng thoát container (container breakout) hoặc có mount volume hệ thống từ host, tiến trình root bên trong container có thể tương tác trực tiếp với host kernel / file system của máy host dưới quyền root của host.
Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay từ bước 2: tiến trình app chỉ có quyền của user thường không có đặc quyền (UID 10001), ngăn chặn việc truy cập / chỉnh sửa các file nhạy cảm của hệ điều hành, cài đặt rootkit và vô hiệu hóa phần lớn các kỹ thuật leo thang đặc quyền ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Con số tối đa: 20 request trong 2 giây liên tiếp.
Cách đạt được:
Người dùng gửi 10 request vào giây cuối cùng của phút thứ nhất (10:00:59). Đến 10:01:00, bộ đếm theo phút đồng hồ tự động reset về 0. Người dùng gửi tiếp 10 request vào giây đầu tiên của phút thứ hai (10:01:01). Như vậy, trong khoảng thời gian chỉ 2 giây (từ 10:00:59 đến 10:01:01), hệ thống đã nhận và cho qua 20 request, gấp đôi hạn mức quy định (10 req/phút). Cửa sổ trượt (Sliding Window) giải quyết triệt để lỗi này bằng cách luôn xét đúng 60 giây gần nhất tính từ thời điểm gọi.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Sự khác nhau: Rate limit giới hạn **tần suất/số lượng** request trong một khoảng thời gian ngắn (chống nghẽn hạ tầng/DoS), còn Cost guard giới hạn **tổng chi phí/tiền** tiêu thụ trong một kỳ hạn (tháng) dựa trên lượng token thực tế sử dụng.
- Tình huống Rate limit cho qua nhưng Cost guard chặn: User chỉ gọi 1 request trong phút (vẫn trong hạn mức 10 req/phút), nhưng tài khoản của user đó trong tháng đã tiêu hết $10.0 ngân sách (vượt quota tháng).
- Tình huống ngược lại: User mới bắt đầu tháng, ngân sách còn nguyên $10.0, nhưng gửi 15 request dồn dập trong 5 giây -> Cost guard còn đủ tiền nhưng Rate limit sẽ chặn từ request thứ 11 trả về 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện khi gộp `/health` và `/ready` lại làm một và kiểm tra Redis:
1. Redis mất kết nối 30 giây.
2. Bộ điều phối (Orchestrator / Docker / Kubernetes) liên tục gửi liveness probe vào `/health`. Do probe kiểm tra cả Redis, cả 3 container đều đồng loạt phản hồi lỗi / unhealthy.
3. Orchestrator cho rằng cả 3 tiến trình container đều đã chết nên kích hoạt cơ chế restart đồng loạt cả 3 container.
4. Quá trình restart container tốn tài nguyên và thời gian khởi động lại ứng dụng.
5. Khi Redis phục hồi sau 30 giây, toàn bộ các container vẫn đang trong trạng thái khởi động lại / crashing, khiến toàn bộ hệ thống bị gián đoạn hoàn toàn (cascade failure) thay vì chỉ tạm dừng nhận traffic mới ở tầng Load Balancer.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử hội thoại được lưu trong một `dict` Python trong RAM:
Do 3 container chạy độc lập với 3 vùng nhớ RAM riêng biệt, Load Balancer phân phối các request luân phiên qua từng container (Round-Robin). Request 1 vào container A (ghi vào RAM của A), Request 2 vào container B (B có dict rỗng nên `history_length` lại là 0), Request 3 vào container C (`history_length` là 0), Request 4 lại vào container A (`history_length` lúc này là 2). Kết quả là `history_length` sẽ nhảy lung tung không theo thứ tự tăng dần đều và agent liên tục bị "mất trí nhớ" giữa các lượt hội thoại.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp phải: Healthcheck timeout hoặc `502 Bad Gateway` khi deploy lên Cloud platform (như Railway / Render).
- Nguyên nhân: Ứng dụng hardcode cổng `8000` và bind vào `127.0.0.1` trong lệnh khởi chạy `uvicorn`, trong khi các nền tảng Cloud tự cấp phát một cổng ngẫu nhiên qua biến môi trường `$PORT` và yêu cầu bind vào `0.0.0.0` để proxy/load balancer bên ngoài có thể định tuyến vào.
- Cách khắc phục: Sửa `CMD` trong Dockerfile thành `["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` để linh hoạt đọc cổng từ biến môi trường `$PORT` do platform cấp.
