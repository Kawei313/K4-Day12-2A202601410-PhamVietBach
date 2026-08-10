# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng gợi ý dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phạm Việt Bách Mã học viên: 2A202601410

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

---Ví dụ khi deploy lên Render, tôi quên khai báo API_TOKEN trong Environment Variables. Nếu app dùng token mặc định "changeme", deploy vẫn thành công nhưng endpoint /chat có thể bị truy cập bằng token mặc định, dẫn đến spam và phát sinh chi phí. Khi không có giá trị mặc định, app dừng ngay lúc khởi động và log báo thiếu API_TOKEN; tôi phát hiện lỗi trước khi service được mở cho Internet.

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:

```json
{"event":"chat_completed","severity":"INFO","ts":"2026-08-10T...+00:00","client_id":"sv01","usd_cost":0.0000...}
Hai việc có thể làm với dòng log này:
Lọc và tìm kiếm theo từng trường, ví dụ lọc toàn bộ request của client_id: "sv01" hoặc chỉ các log có severity: "ERROR". Với print("đã trả lời xong"), máy chỉ nhận một chuỗi chữ chung chung, không có dữ liệu để lọc chính xác.

Tổng hợp số liệu tự động, ví dụ cộng trường usd_cost để theo dõi chi phí theo từng client, hoặc đếm event chat_completed theo thời gian để biết số lượng request. Log JSON có cấu trúc cố định nên công cụ giám sát đọc được; print không có các trường dữ liệu riêng.
---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | Chưa đo — thiếu Dockerfile.single |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Image multi-stage chỉ 183MB vì stage cuối dùng python:3.11-slim và chỉ nhận dependency đã cài từ stage builder cùng source code cần chạy. Các công cụ biên dịch, file trung gian và cache cài đặt không đi vào runtime. Bản 1-stage dùng python:3.11 đầy đủ, copy toàn bộ project rồi cài thư viện trực tiếp nên thường lớn hơn đáng kể; phần chênh lệch là các thành phần build và gói hệ điều hành không cần để chạy service.
---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi chỉ sửa một ký tự trong app/main.py:
Các layer trước COPY app ./app được dùng lại từ cache: base image, WORKDIR, COPY requirements.txt, và RUN pip install.
COPY app ./app bị chạy lại vì mã nguồn app/main.py đã đổi.
Các layer phía sau nó cũng chạy lại, ví dụ tạo user và các layer sau đó.
pip install không chạy lại nên build nhanh hơn nhiều.
Nếu đặt COPY . . trước RUN pip install, sửa một ký tự code sẽ làm layer COPY . . thay đổi. Docker phải làm lại mọi layer sau đó, bao gồm cả RUN pip install, dù requirements.txt không đổi. Đây là lý do ta copy requirements.txt và cài dependency trước, rồi mới copy source code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Một lỗ hổng trong Python app (ví dụ thực thi lệnh từ dữ liệu người dùng) có thể cho kẻ tấn công chạy lệnh bên trong container. Nếu container chạy bằng root, lệnh đó có đặc quyền root trong container. Nếu container lại có cấu hình nguy hiểm như mount Docker socket, bind-mount thư mục host, hoặc có lỗ hổng container runtime/kernel, họ có thể lợi dụng quyền đó để đọc/sửa dữ liệu host hoặc thoát container, dẫn đến quyền cao trên host.
Lệnh:
USER appuser
cắt chuỗi tại bước app bị chiếm quyền: tiến trình Python và lệnh do kẻ tấn công chạy chỉ có quyền của appuser, không phải root. Vì vậy họ khó cài package hệ thống, sửa file nhạy cảm, đọc nhiều tài nguyên đặc quyền hoặc khai thác đường thoát container. Đây là giảm thiểu rủi ro, không thay thế việc vá lỗ hổng hay tránh mount Docker socket.
---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

WWW-Authenticate: Bearer đi kèm 401 Unauthorized để nói cho client biết endpoint dùng cơ chế xác thực Bearer token. Các thư viện HTTP, API client hoặc frontend nhờ đó biết cần gửi header dạng:
                  Authorization: Bearer <token>
Ta trả cùng một thông báo lỗi cho thiếu header, sai scheme và sai token để không tiết lộ thông tin cho người tấn công. Nếu báo riêng “token không tồn tại” hoặc “token sai”, họ có thể dò token hoặc suy ra hệ thống đang dùng loại xác thực nào. Một lỗi chung như invalid or missing token vừa đủ cho client biết cần kiểm tra cấu hình, vừa giảm thông tin hữu ích cho việc tấn công.
---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

Với capacity=10, client gửi được 10 request rồi request thứ 11 bị 429. Sau 10 phút, xô đã đầy nhưng không thể vượt quá sức chứa 10.

Nếu bỏ min(capacity, ...), client tích được 100 token sau 10 phút (10 token/phút × 10 phút), nên có thể gửi khoảng 100 request liên tiếp trước khi bị 429.

Lý do: min(capacity, tokens) là “nắp xô”, giới hạn số token tối đa. Bỏ nó đi thì client càng im lặng lâu càng tích được nhiều token, rồi có thể tạo một đợt request lớn trong thời gian rất ngắn.
---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

Hạn mức $30/tháng: sự cố từ 2h sáng có thể tiêu hết tối đa $30 trong tháng đó. Client chỉ tự gọi lại được khi sang tháng mới, tức hạn mức được reset đầu tháng.

Hạn mức $1/ngày: thiệt hại tối đa trong một ngày là $1. Client bị chặn phần còn lại của ngày và tự hoạt động lại khi sang ngày mới (sau 00:00 theo múi giờ hệ thống).

Vì vậy hạn mức theo ngày giảm “blast radius”: sự cố không thể đốt toàn bộ $30 ngay trong vài giờ đầu ngày.
---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Trong 30 giây Redis mất kết nối, endpoint gộp sẽ trả lỗi vì không kiểm tra được Redis. Orchestrator hiểu nhầm cả 3 container đều “chết”, lần lượt restart hoặc thay thế chúng. Trong lúc restart, các request đang xử lý có thể bị ngắt và số container sẵn sàng phục vụ giảm, dù bản thân FastAPI/Uvicorn vẫn hoạt động bình thường. Khi Redis kết nối lại, các container mới khởi động xong và nhận traffic trở lại. Việc restart hàng loạt là không cần thiết; vì vậy /healthz chỉ nên kiểm tra app còn sống, còn /readyz mới kiểm tra Redis để tạm rút container khỏi traffic.
---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Khi deploy, service không chạy được vì Dockerfile đặt lệnh CMD ở stage builder thay vì stage runtime cuối cùng. Render chỉ chạy image của stage cuối, nên image runtime không có lệnh khởi động Uvicorn. Tôi nhận ra nguyên nhân bằng cách kiểm tra lại Dockerfile theo thứ tự các stage và xem log build/deploy của Render: runtime stage thiếu CMD. Tôi sửa bằng cách chuyển CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"] xuống cuối stage runtime, sau đó commit/push để Render deploy lại. Health check /healthz trả về HTTP 200 sau khi service chạy thành công.
