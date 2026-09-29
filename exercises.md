# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Dương Hữu Đạt  Mã học viên: 2A202602544

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định là `"changeme"`, ứng dụng vẫn sẽ khởi động bình thường trên production (Render/Railway). Hacker có thể dùng khóa "changeme" này để gọi API vô tội vạ, khiến dự án bị cạn kiệt ngân sách thật mà ta không hề hay biết. "Fail fast" giúp ta phát hiện ra ngay lập tức ở lúc vừa khởi động (báo lỗi sập app) là ta đã quên cài đặt khóa bí mật trên Cloud.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-01T09:30:00+00:00", "user_id": "sv-test", "cost_usd": 0.0001}`
> Hai việc làm được: (1) Tính tổng trường `cost_usd` trong database log để tìm ra top user nào đang tiêu tốn nhiều tiền nhất. (2) Tìm kiếm nhanh theo `event="ask_completed"` để đếm tổng số request thành công mỗi ngày. Việc in chữ bằng `print` thì hệ thống không thể bóc tách dữ liệu để phân tích thống kê tự động như thế này được.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

> | 1 stage (bản đầu) | ~1000 MB |
> | Multi-stage | ~150-200 MB |
> Phần dung lượng chênh lệch đó chính là các công cụ hỗ trợ build (như C++ header, trình biên dịch GCC, các file tạm `build-essential`). Multi-stage build chỉ copy "thành phẩm" cuối cùng sang stage chạy và vứt bỏ toàn bộ đống công cụ nặng nề đó đi.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Layer `RUN pip install` sẽ được tái sử dụng lại (cached). Chỉ có layer `COPY app ./app` và các layer nằm bên dưới nó mới bị chạy lại. 
> Nếu đặt `COPY . .` lên trước `RUN pip install`, hệ thống Docker sẽ xem như code bị thay đổi nên hủy toàn bộ cache của layer `pip install`. Hậu quả là mỗi lần ta sửa 1 dấu phẩy trong code, ta phải ngồi đợi tải và cài đặt lại toàn bộ thư viện từ đầu rất lâu.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Lỗ hổng code -> Attacker chạy được lệnh độc hại -> Nếu chạy root, tiến trình có quyền quản trị cao nhất -> Lợi dụng lỏng lẻo của Docker (nếu có volume mount) -> Thoát khỏi container chiếm quyền máy Host. Lệnh `USER appuser` cắt đứt chuỗi ở bước thứ 2: ứng dụng chạy dưới dạng user thường không có đặc quyền, do đó kể cả hacker thực thi được lệnh thì chúng cũng bị giới hạn quyền truy cập, giảm thiểu tối đa sức tàn phá.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi tối đa 20 request trong 2 giây. User chỉ cần gửi 10 request vào lúc 10:00:59 và 10 request vào lúc 10:01:01. Vì hệ thống đếm theo phút đồng hồ sẽ tự reset về 0 ở giây 00 (tức là lúc 10:01:00), nên cả 20 lệnh này đều hợp lệ theo luật, để lọt một lưu lượng lớn bất thường vào hệ thống. Cửa sổ trượt (sliding window) lấp được lỗ hổng này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Rate Limit** chặn theo tần suất gửi request trong 1 khung thời gian rất ngắn. **Cost Guard** chặn theo tổng ngân sách (quy ra tiền) trong một khung thời gian dài.
> - Rate Limit cho qua nhưng Cost Guard chặn: Gửi 1 câu hỏi rất ngắn (chưa vượt 10 request/phút) nhưng do cả tháng qua user đó đã gọi quá nhiều nên bị chạm giới hạn ngân sách $10.
> - Rate Limit chặn nhưng Cost Guard cho qua: User dội bomb 15 request liên tục trong vòng 1 giây. Tổng chi phí thì vẫn còn thừa nhiều, nhưng tần suất gọi đã bị vi phạm quá đà.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Sự kiện 1: Redis nấc (mất mạng) -> Cả 3 container đều báo lỗi 503 khi bị Health Probe hỏi (vì bị gộp chung).
> Sự kiện 2: Orchestrator tưởng cả 3 con server bị chết cứng nên khởi động lại (restart) toàn bộ.
> Sự kiện 3: Khi Redis phục hồi lại bình thường, hệ thống web lại không sẵn sàng vì 3 con container đang bận khởi động lại -> Tự dưng biến một sự cố nhỏ thành sự cố lớn (downtime).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Con số history_length sẽ nhảy lung tung (gọi là hội chứng mất trí nhớ). Ví dụ: Gọi lần 1 nó bay vào container A nên dài = 1. Gọi lần 2 load balancer ném vào container B, con B này đang dùng dict python độc lập trống rỗng nên nó lại báo dài = 1. Gọi lần 3 nó rơi vào lại container A thì dài = 2.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp phải: Container trên Render Deploy thành công nhưng bị báo Health check timeout và Service sập liên tục. 
> Nguyên nhân: Ứng dụng code fix cứng lắng nghe ở cổng `--port 8000`, trong khi Cloud Render không mở port đó.
> Cách sửa: Sửa dòng CMD trong file Dockerfile thành đọc biến số từ Cloud: `uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}`.
