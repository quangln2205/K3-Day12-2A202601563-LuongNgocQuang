# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu mình lỡ quên set `AGENT_API_KEY` trước khi deploy, app sẽ fail ngay lúc
khởi động thay vì âm thầm nhận request rồi trả lỗi về sau. Nhờ vậy mình phát
hiện sai cấu hình ngay khi build/deploy còn đang xanh, không để một service
chạy với khóa mặc định và vô tình mở API cho người khác.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Ví dụ log có thể là `{"event":"ask_completed","level":"info","timestamp":"2026-08-10T12:00:00+00:00","user_id":"sv01","tokens_in":12,"tokens_out":18,"cost_usd":0.0004}`.
Từ dòng này mình có thể thống kê ai dùng nhiều nhất và tổng chi phí theo user;
ngoài ra còn lọc theo event/timestamp để tìm request nào xảy ra lúc nào,
trong khi `print("đã trả lời xong")` chỉ là một câu chữ không có cấu trúc.

---

### Câu 3 — Kích thước image (CP2):

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

Image 1 stage thường lớn hơn nhiều vì nó mang theo cả dependency build,
compiler, cache và các file trung gian. Multi-stage chỉ giữ lại phần runtime
cần thiết như code app và package đã cài, nên nhẹ hơn vì bỏ được công cụ build
và lớp cài đặt dư thừa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa một ký tự trong `app/main.py`, Docker chỉ phải chạy lại layer copy code
và các layer phía sau nó; layer cài dependency từ `requirements.txt` vẫn được
cache nếu file đó không đổi. Nếu đặt `COPY . .` trước `RUN pip install`, thì chỉ
cần đổi code là mất cache từ đầu, Docker phải cài lại toàn bộ dependency, build
sẽ chậm hơn rất nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu container chạy bằng root thì một lỗ hổng cho phép thực thi lệnh trong app
có thể bị lợi dụng để ghi/xóa file hệ thống hoặc leo thang sang quyền mạnh hơn
trong container, rồi từ đó tác động sang host qua mount, socket, hay các đặc
quyền sai cấu hình. `USER appuser` cắt chuỗi đó bằng cách làm cho tiến trình app
chỉ có quyền của một user thường, nên cùng một lỗ hổng cũng ít gây hậu quả hơn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với cách reset theo phút đồng hồ, một người dùng có thể chen 10 request ngay
trước phút cũ và thêm 10 request ngay sau khi sang phút mới, nên trong 2 giây
liên tiếp họ có thể gửi tối đa 20 request. Họ chỉ cần canh sát mốc chuyển phút
để cửa sổ bị reset theo đồng hồ thay vì trượt liên tục.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit kiểm tra tần suất gọi: một user gọi quá nhanh thì bị 429 dù chưa tốn
quá nhiều tiền. Cost guard kiểm tra ngân sách tháng: user có thể gọi không quá
nhanh nhưng nếu đã vượt ngân sách thì phải bị chặn 402. Ví dụ: một user gửi 5
request rải đều, vẫn qua rate limit nhưng nếu tổng tiền tháng đã hết thì cost
guard chặn; ngược lại, một user spam liên tục trong vài giây thì rate limit
chặn trước khi cost guard cần xét.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

`/health` là liveness: chỉ hỏi process còn sống không, nên không được phụ thuộc
Redis. `/ready` là readiness: nó được phép kiểm tra dependency để quyết định có
nhận traffic không. Nếu gộp lại và `/health` cũng check Redis, khi Redis chết 30
giây thì orchestrator sẽ tưởng container hỏng, restart liên tục cả 3 instance
thay vì chỉ ngừng đẩy traffic vào chúng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử nằm trong dict Python thì mỗi container giữ một bộ nhớ riêng, nên
gọi `/ask` qua cùng một `X-User-Id` nhưng đổi container sẽ làm `history_length`
quay về thấp hoặc không tăng đúng. Khi lưu trong Redis, `history_length` tăng
ổn định theo cả cụm, vì mọi instance đều đọc cùng một nguồn state.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Mình từng gặp lỗi health check trên Render trả `Exited with status 3 while
running your code` vì app vẫn còn `NotImplementedError` trong `lifecycle.py`.
Mình tìm nguyên nhân bằng cách đọc log deploy, thấy traceback trỏ thẳng vào
`request_shutdown()`; sau đó mình hoàn thiện handler graceful shutdown và redeploy.
