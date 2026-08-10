# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phạm Thế Dũng  Mã học viên: 2A202601985

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu deploy lên Render mà quên đặt `AGENT_API_KEY`, app sẽ fail ngay khi khởi động thay vì chạy công khai với một khóa mặc định mà ai cũng biết. Nhờ vậy lỗi được phát hiện trong log deploy, trước khi có request hoặc chi phí phát sinh. Dùng `changeme` nguy hiểm vì service vẫn chạy và người khác có thể đoán được khóa đó.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Ví dụ: `{"event":"ask_completed","level":"info","timestamp":"2026-08-10T10:30:00+00:00","user_id":"sv01","tokens_in":12,"tokens_out":40,"cost_usd":0.000026}`. Từ một dòng JSON, mình có thể lọc riêng các event `ask_completed` và tính tổng `cost_usd`; đồng thời có thể nhóm theo `user_id` để tìm user gọi nhiều hoặc theo timestamp để đo lưu lượng. `print` thông thường không có các trường cố định nên việc lọc và tổng hợp khó tin cậy.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Mình chưa ghi số đo image thực tế vào file vì kích thước phụ thuộc Docker cache và phiên bản image tại máy chạy. Có thể đo bằng `docker images agent:single` và `docker images agent:multi`. Multi-stage thường nhỏ hơn vì runtime chỉ chứa Python, dependencies và source cần chạy; các file tạm, cache build và công cụ không cần khi chạy không được đưa sang image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Vì `requirements.txt` được copy và cài trước khi source được copy, sửa một dòng trong `app/main.py` chỉ làm chạy lại các layer từ bước copy source trở đi; layer cài dependency vẫn dùng cache. Nếu đặt `COPY . .` trước `pip install`, mọi thay đổi source sẽ làm mất cache của bước sau và phải cài lại toàn bộ dependency, khiến build chậm hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng trong app có thể cho phép attacker thực thi lệnh bên trong container. Nếu process chạy bằng root, attacker có quyền cao nhất trong container và có thể đọc/sửa nhiều file hoặc khai thác tiếp vào Docker host nếu cấu hình có lỗ hổng. `USER appuser` chuyển process sang user thường, nên quyền của attacker bị giới hạn và chuỗi leo thang đặc quyền bị cắt ở lớp container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi 20 request trong khoảng 2 giây: 10 request ngay trước thời điểm chuyển phút, ví dụ giây 59, rồi 10 request ngay sau khi đồng hồ reset ở giây 00. Bộ đếm theo phút đồng hồ coi chúng thuộc hai phút khác nhau, dù thực tế chỉ cách nhau khoảng 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số lần gọi, còn cost guard giới hạn số tiền đã tiêu. Ví dụ user gọi 3 request nhỏ trong hạn mức 10 request/phút nên rate limit cho qua, nhưng tổng chi phí trước đó đã gần 10 USD nên cost guard chặn request tiếp theo bằng 402. Ngược lại, user có thể đã hết rate limit với nhiều request rẻ và bị 429 dù ngân sách vẫn còn; cost guard không thay thế được rate limit.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> `/health` là liveness probe, chỉ kiểm tra process còn sống và không phụ thuộc Redis. `/ready` là readiness probe, kiểm tra Redis trước khi nhận traffic. Nếu gộp chúng và Redis mất 30 giây, cả 3 container sẽ trả lỗi health; orchestrator có thể restart cả cụm, dù process vẫn chạy và Redis chỉ đang tạm gián đoạn. Tách hai endpoint giúp load balancer ngừng gửi traffic qua `/ready` mà không restart nhầm toàn bộ app qua `/health`.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lịch sử nằm trong dict của từng process, request thứ nhất vào container A được lưu ở RAM A, còn request tiếp theo vào B không thấy dữ liệu nên `history_length` có thể quay lại 0 hoặc thay đổi thất thường. Khi lưu trong Redis, cả 3 container đọc cùng một nguồn state nên lịch sử và `history_length` nhất quán dù request đi qua instance nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi deploy Render, lỗi đầu tiên của mình là endpoint `/` trả `{"detail":"Not Found"}`. Nguyên nhân không phải service chết mà vì app chỉ khai báo `/health`, `/ready` và `/ask`, không khai báo `/`. Mình kiểm tra bằng cách gọi đúng `/health` và `/ready`; cả hai trả 200, sau đó ghi URL đúng vào `DEPLOYMENT.md` và dùng `/health` làm health check path trên Render.
