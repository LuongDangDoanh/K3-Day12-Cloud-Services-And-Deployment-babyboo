# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lương Đăng Doanh  Mã học viên: 2A202601209

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ cụ thể là lúc deploy lên Railway nhưng quên tạo biến
> `AGENT_API_KEY`. Nếu code có mặc định `"changeme"`, container vẫn healthy và
> URL công khai bắt đầu nhận request bằng một khóa ai cũng đoán được. Khi field
> này không có mặc định, Pydantic ném `ValidationError` ngay lúc start; deployment
> đỏ trước khi có traffic, nên tôi phát hiện và bổ sung secret khi vẫn đang xem
> log deploy thay vì chỉ phát hiện sau khi API đã bị dùng và phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thực tế tôi thu được là:
> `{"event":"ask_completed","level":"info","timestamp":"2026-08-10T04:56:18.594381+00:00","user_id":"exercise-user","tokens_in":3,"tokens_out":42,"cost_usd":2.565e-05}`.
> Từ các field này, hệ thống log có thể (1) nhóm theo `user_id` và cộng
> `cost_usd` để tìm user tiêu nhiều nhất, và (2) đếm event theo thời gian/level
> để vẽ tỷ lệ lỗi hoặc tạo cảnh báo. Chuỗi `print("đã trả lời xong")` không có
> user, timestamp, token hay chi phí để lọc và tổng hợp tự động.

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
| 1 stage (bản đầu) | 1.69 GB (khoảng 1690 MB) |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build thật bằng Docker và thấy bản one-stage dùng `python:3.11` là
> 1.69 GB, còn `python:3.11-slim` multi-stage là 270 MB. Phần chênh lệch chủ yếu
> là hệ điều hành/base image đầy đủ, công cụ build và các dữ liệu cài đặt chỉ cần
> lúc build. Stage runtime chỉ nhận dependency đã cài từ builder cùng source
> cần chạy, nên không mang toàn bộ môi trường build sang production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, stage builder vẫn dùng cache cho `COPY
> requirements.txt` và `pip install` vì `requirements.txt` không đổi. Ở runtime,
> các layer tạo user và copy dependency từ builder cũng được dùng lại; cache bị
> mất từ `COPY app ./app` trở đi. Nếu đặt `COPY . .` trước `RUN pip install`, một
> thay đổi nhỏ trong source làm checksum của layer COPY đổi và kéo theo việc cài
> lại toàn bộ dependency, khiến build chậm hơn rõ rệt dù requirements không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu ứng dụng Python có lỗ hổng thực thi lệnh, kẻ tấn công có thể chạy lệnh với
> UID của process trong container. Khi process là root, kết hợp với cấu hình nguy
> hiểm như mount thư mục host/Docker socket hoặc một lỗ hổng container runtime,
> quyền đó có thể được dùng để sửa file host, điều khiển container khác hoặc leo
> thang trên host. `USER appuser` cắt chuỗi ở bước đầu: mã khai thác chỉ có UID
> 10001 với quyền filesystem tối thiểu trong container. Nó không thay thế việc vá
> lỗ hổng/runtime, nhưng giảm đáng kể tác động nếu ứng dụng bị chiếm quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi 20 request trong khoảng 2 giây: gửi 10 request ở cuối phút, ví dụ
> 10:00:59.x, rồi ngay sau khi counter reset gửi thêm 10 request ở đầu phút kế
> tiếp, 10:01:00.x. Mỗi bucket phút riêng vẫn chỉ có 10 request. Sliding window
> nhìn lại đúng 60 giây gần nhất nên lượt thứ 11 trong cụm đó bị chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ/số request trong 60 giây, còn cost guard giới hạn
> tổng tiền của từng user trong tháng. Một user mới chỉ gửi một request nhưng
> prompt rất lớn, hoặc đã gần hết ngân sách tháng, vẫn qua rate limit nhưng phải
> bị cost guard chặn. Ngược lại, một user còn nguyên ngân sách nhưng gửi 11 câu
> hỏi rất ngắn trong một phút sẽ bị rate limiter chặn ở lượt 11 dù tổng chi phí
> vẫn thấp.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm endpoint gộp trả 503 trên cả ba container. Orchestrator
> hiểu đó là lỗi liveness và lần lượt restart cả ba. Container mới start vẫn
> không gọi được Redis nên tiếp tục unhealthy và bị restart lặp lại. Khi Redis
> phục hồi sau 30 giây, có thể không còn instance ổn định nào sẵn sàng phục vụ,
> biến sự cố dependency ngắn thành outage toàn cụm. Tách `/health` giúp process
> vẫn sống; chỉ `/ready` trả 503 để load balancer tạm ngừng gửi traffic mà không
> restart container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, mỗi request thấy lịch sử mà request trước đã ghi dù đi
> vào instance khác, nên `history_length` tăng đều theo cặp message: 0, 2, 4,
> 6,... Nếu dùng dict riêng trong từng process, request bị load balancer chuyển
> sang instance khác sẽ thấy một lịch sử khác; số có thể thành 0, 0, 2, 0, 2...
> hoặc tăng rồi đột ngột giảm, tạo cảm giác agent mất trí nhớ ngẫu nhiên.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thật tôi gặp là Railway log: `Error: Invalid value for '--port': '$PORT'
> is not a valid integer`. Tôi kiểm tra deployment bằng `railway status` và
> `railway logs`, từ đó thấy `startCommand` trong `railway.toml` được thực thi
> trực tiếp nên chuỗi `$PORT` không được shell mở rộng. Tôi xóa override
> `startCommand` để Railway dùng `CMD` của Dockerfile; CMD này chạy qua `sh -c`
> với `${PORT:-8000}`. Sau khi deploy lại, Uvicorn bind `0.0.0.0:8080`, health
> check qua và deployment chuyển sang `SUCCESS`.
