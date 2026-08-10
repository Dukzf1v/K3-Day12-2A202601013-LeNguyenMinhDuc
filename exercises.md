# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.

Họ và tên: Lê Nguyễn Minh Đức  Mã học viên: 2A202601013

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy ứng dụng lên Cloud (Render/Railway), nếu tôi lỡ quên cấu hình biến môi trường `AGENT_API_KEY` trên Dashboard, việc `agent_api_key` không có mặc định giúp app chết ngay khi vừa khởi động (`ValidationError`). Lỗi hiển thị ngay trên log deploy giúp tôi phát hiện và khắc phục ngay lập tức. Ngược lại, nếu để giá trị mặc định `"changeme"`, server vẫn sẽ chạy và công khai với khóa bí mật `"changeme"`. Kẻ gian có thể dò ra chìa khóa mặc định này và tự do gọi API tiêu tốn tài nguyên server trước khi tôi kịp phát hiện.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T04:27:05.425221+00:00", "user_id": "sv-test", "tokens_in": 4, "tokens_out": 36, "cost_usd": 0.0000222}`

Hai việc làm được:
1. Thống kê và lọc chi tiết chi phí/lượng token tiêu thụ theo từng `user_id` cụ thể trong khoảng thời gian (ví dụ: dùng ElasticSearch/Datadog để tính tổng `cost_usd` của user `sv-test` trong tháng).
2. Thiết lập hệ thống cảnh báo tự động (Alerting) khi tỷ lệ lỗi tăng cao hoặc số lượng sự kiện `ask_completed` sụt giảm bất thường dựa trên truy vấn trực tiếp vào các trường JSON.

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
| 1 stage (bản đầu) | ~1020 MB |
| Multi-stage | ~210 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~810 MB) chính là các công cụ biên dịch (compiler như GCC/g++, build tools), header files, bộ nhớ đệm của pip (`pip cache`), và hệ điều hành đầy đủ ở stage `builder`. Nhờ Multi-stage Build, toàn bộ công cụ nặng đó bị loại bỏ, stage `runtime` chỉ copy phần kết quả binary/library cần thiết nên kích thước giảm mạnh.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Các layer cài đặt thư viện (`COPY requirements.txt` và `RUN pip install`) đứng trước sẽ được tái sử dụng hoàn toàn từ Docker Cache. Chỉ có các layer từ `COPY app ./app` trở về sau mới phải chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần sửa 1 ký tự code, Docker phát hiện layer `COPY` bị thay đổi và sẽ làm mất hiệu lực cache của tất cả các lệnh đứng sau ➔ Phải chạy lại câu lệnh `pip install` từ đầu, làm tốn thêm nhiều thời gian build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện: Code Python bị rò rỉ lỗ hổng RCE (Remote Code Execution) ➔ Kẻ tấn công gửi request chứa mã độc ➔ Vì container chạy quyền `root`, tiến trình chiếm quyền sẽ có đặc quyền `root` bên trong container ➔ Nếu container bị rò rỉ cách ly (Kernel exploit/Volume mount escape), kẻ tấn công thoát ra máy host và có luôn quyền `root` kiểm soát toàn bộ máy chủ vật lý.
- Lệnh `USER appuser` cắt đứt chuỗi ở bước tiến hành chiếm quyền: Đưa tiến trình ứng dụng chạy dưới dạng user thường (UID 10001). Khi bị khai thác lỗ hổng, hacker chỉ có quyền hạn chế của user thường, không thể cài phần mềm độc hại hay tương tác với tài nguyên hệ thống host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request trong 2 giây liên tiếp**.
Giải thích: Ở phút thứ 1 (ví dụ 10:00:00 - 10:01:00), user gửi 10 request vào giây 10:00:59 (vẫn đúng luật 10/10). Ngay giây tiếp theo 10:01:01 (bước sang phút mới, bộ đếm reset về 0), user tiếp tục gửi 10 request nữa (vẫn hợp lệ 10/10 của phút thứ 2). Như vậy chỉ trong 2 giây (từ 10:00:59 đến 10:01:01), user đã gửi 20 request. Thuật toán cửa sổ trượt 60s tính đúng 60 giây gần nhất nên sẽ chặn được kẽ hở này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác nhau: Rate limit giới hạn **số lượng/tần suất** request trong khoảng thời gian ngắn (ví dụ: 10 req/phút). Cost guard giới hạn **ngân sách tiền tệ/token** tiêu thụ trong khoảng thời gian dài (ví dụ: 10.0 USD/tháng).
- Rate limit cho qua nhưng Cost guard chặn: User chỉ gửi 2 request/phút (dưới hạn mức 10 req/phút), nhưng mỗi request có prompt siêu dài 50,000 token khiến ngân sách tháng vượt quá 10.0 USD ➔ Cost guard chặn (lỗi 402).
- Cost guard cho qua nhưng Rate limit chặn: User mới tiêu hết 0.1 USD (còn rất nhiều ngân sách tháng), nhưng lại spam 15 request liên tục trong vòng 5 giây ➔ Rate limit chặn (lỗi 429).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis bị mất kết nối mạng hoặc gián đoạn trong 30 giây.
2. Cả 3 container Agent kiểm tra Redis trong endpoint chung và đều trả về lỗi 500/503.
3. Bộ điều phối (Orchestrator/Docker/K8s) thấy `/health` báo lỗi ➔ Cho rằng cả 3 container Agent đã chết và tiến hành **restart (kill & bật lại)** đồng loạt cả 3 container.
4. Trong suốt 30s Redis sập, cả 3 container Agent liên tục bị restart trong vòng lặp. Đến khi Redis hồi phục, các container Agent vẫn chưa kịp khởi động xong ➔ Sự cố nhỏ ở Redis biến thành sự cố toàn hệ thống (sập toàn bộ dịch vụ).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế meo?

Nếu lưu trong dict Python của RAM: Khi gửi nhiều request liên tiếp với cùng `X-User-Id`, các request sẽ được Load Balancer điều phối ngẫu nhiên vào 3 container A, B, C khác nhau. Kết quả `history_length` sẽ tăng giảm hỗn loạn và bất thường (ví dụ: req 1 vào A ➔ history=0; req 2 vào B ➔ history=0; req 3 vào A ➔ history=2; req 4 vào C ➔ history=0). Agent bị "mất trí nhớ" bất thình lình tùy thuộc vào request rơi vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp phải: Endpoint `/ready` trên Cloud trả về lỗi `503 {"status": "not ready", "redis": false}` sau khi mới deploy dịch vụ lên Render.
Nguyên nhân: Kiểm tra mục Environment Variables trên Render Dashboard thì thấy biến `REDIS_URL` ban đầu bị cài nhầm thành `redis://localhost:6379/0` (localhost bên trong container agent nên không kết nối được tới Redis container).
Cách sửa: Đã cập nhật biến `REDIS_URL` trên Render Dashboard thành chuỗi Internal Connection String do dịch vụ Redis trên Render cung cấp (dạng `redis://red-xxx:6379`), sau đó nhấn Deploy lại ➔ `/ready` trả về `200 {"status":"ready","redis":true}`.
