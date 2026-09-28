# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay phần trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Lê Phúc Thắng  Mã học viên: 2A202602638

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là lúc deploy lên Railway nhưng quên khai báo
> `AGENT_API_KEY`. Nếu ứng dụng dùng mặc định `"changeme"`, container vẫn báo
> chạy bình thường và bất kỳ ai đoán được giá trị mặc định đều có thể gọi
> `/ask`, làm phát sinh chi phí. Với cấu hình hiện tại, tiến trình dừng ngay khi
> khởi động, log chỉ thẳng biến còn thiếu và bản lỗi không bao giờ nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng mình thu được khi gọi `/ask` trên container local là:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:42:05.458543+00:00", "user_id": "exercise-log", "tokens_in": 2, "tokens_out": 36, "cost_usd": 2.19e-05}
> ```
>
> Từ dòng này mình có thể lọc theo `event`, `user_id` và khoảng thời gian để
> điều tra một request; đồng thời cộng `tokens_in`, `tokens_out` hoặc `cost_usd`
> để làm dashboard và cảnh báo ngân sách. Chuỗi `print("đã trả lời xong")`
> không có trường ổn định để máy lọc hay tổng hợp hai việc đó.

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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 310 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Mình build lại Dockerfile ban đầu từ commit gốc thành
> `day12-agent:single` và đo được 1.73 GB; image hiện tại là 310 MB, giảm khoảng
> 1.42 GB. Phần chênh lệch chủ yếu đến từ image `python:3.11` đầy đủ cùng các
> thành phần hệ điều hành/công cụ không cần khi chạy. Bản multi-stage dùng
> `python:3.11-slim`; stage runtime chỉ nhận virtualenv, `app/` và `utils/`, nên
> không mang toàn bộ môi trường build sang production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình thêm một dòng comment vào `app/main.py` rồi build lại. Docker báo các
> bước tạo virtualenv, `COPY requirements.txt` và `RUN pip install` đều
> `CACHED`; chỉ `COPY app ./app`, `COPY utils ./utils` và bước export image chạy
> lại. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi mã nguồn sẽ làm
> hash của layer copy đổi, khiến layer cài dependency và tất cả layer phía sau
> phải chạy lại dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro là: input độc hại khai thác lỗi trong Python, chiếm quyền tiến
> trình web, rồi lợi dụng cấu hình container yếu (capability dư thừa, volume
> nhạy cảm hoặc lỗ hổng runtime) để tác động ra ngoài container. Nếu tiến trình
> đang là root thì mã khai thác có toàn quyền trong container và hậu quả của một
> lần container escape sẽ lớn hơn nhiều. `USER appuser` cắt chuỗi ngay sau bước
> chiếm tiến trình: kẻ tấn công chỉ nhận UID thường, không thể tự ý sửa file hệ
> thống hay thực hiện thao tác cần root. Đây là giảm quyền, không thay thế hoàn
> toàn lớp cô lập của Docker.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request trong 2 giây: gửi 10 request ở giây 59 của phút trước,
> rồi gửi tiếp 10 request ở giây 00 của phút sau. Bộ đếm theo phút đồng hồ reset
> ở ranh giới đó nên cả hai nhóm đều hợp lệ, dù thực tế 20 request nằm sát nhau.
> Sliding window 60 giây vẫn nhìn thấy nhóm đầu khi nhóm sau đến nên không cho
> phép burst này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit bảo vệ tốc độ trong một cửa sổ ngắn, còn cost guard bảo vệ tổng chi
> phí tích lũy theo ngân sách. Một người gửi mỗi phút một câu rất dài/đắt có thể
> luôn dưới 10 request/phút nhưng cuối tháng vẫn vượt ngân sách, lúc đó cost
> guard phải chặn. Ngược lại, 11 câu cực ngắn gửi liên tiếp khi ngân sách còn
> nhiều sẽ được cost guard cho qua nhưng request thứ 11 bị rate limit trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp probe và bắt nó kiểm tra Redis, khi Redis mất 30 giây thì cả ba
> container lần lượt trả probe 503. Orchestrator đánh dấu cả ba là unhealthy,
> ngừng chuyển traffic rồi restart chúng gần như cùng lúc. Các container mới
> vẫn không kết nối được Redis nên tiếp tục fail và tạo vòng restart, dù tiến
> trình web của chúng không hỏng; dịch vụ vì thế mất toàn bộ capacity. Khi tách
> probe, `/health` vẫn 200 nên container không bị restart, còn `/ready` trả 503
> để tạm rút chúng khỏi traffic. Redis trở lại thì `/ready` tự lên 200 mà không
> cần một đợt khởi động đồng loạt.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Mình chạy ba container cùng nối vào một Redis rồi lần lượt gọi cùng một
> `X-User-Id`. Ba response cho `history_length` là 0, 2 và 4, chứng tỏ container
> sau đọc được lịch sử do container trước ghi. Nếu dùng dict Python, mỗi
> container có một vùng nhớ riêng; khi request luân phiên giữa ba instance mình
> sẽ thấy 0, 0, 0 ở lần đầu của từng instance, sau đó các số tăng theo từng
> chuỗi riêng và nhìn từ client sẽ nhảy không đều.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thật mình gặp là domain Railway trả `502 Application failed to respond`
> dù deployment hiển thị `SUCCESS`. Mình mở runtime log và thấy Uvicorn đang
> nghe `0.0.0.0:8080`, trong khi domain vừa tạo lại route vào port 8000. Mình
> cập nhật target port của Railway domain từ 8000 sang 8080. Sau đó `/health`
> trả 200, `/ready` trả 200 với `"redis": true`, và `/ask` không có key trả 401
> đúng yêu cầu.
