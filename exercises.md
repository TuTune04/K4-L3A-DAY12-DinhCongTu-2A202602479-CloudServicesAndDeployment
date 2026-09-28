# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Dinh Cong Tu  Mã học viên: 2A202602479

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, nếu tôi quên chạy `railway variables` để set `AGENT_API_KEY`: với mặc định `"changeme"`, app vẫn khởi động bình thường, `/health` trả 200, deploy báo xanh và tôi tưởng mọi thứ ổn. Nhưng lúc đó service public đang được bảo vệ bằng một khóa mà ai đọc repo (repo công khai) cũng biết, nên bất kỳ ai cũng gọi được `/ask` và đốt ngân sách LLM của tôi mà tôi không hề hay biết. Với fail fast, container chết ngay lúc start, Railway đánh dấu deploy thất bại và log ghi rõ thiếu trường `agent_api_key`. Tôi phát hiện lỗi trong vài phút, trước khi có request nào đi vào, và sửa bằng cách set biến rồi redeploy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thu được trên Railway:
>
> ```
> [INFO]  event="ask_completed" timestamp="2026-09-28T09:54:30.343005+00:00" user_id="cp5-test" tokens_in=139 tokens_out=45 cost_usd=0.00004785
> ```
>
> Railway tự tách JSON thành từng trường `key=value`, nghĩa là máy đã đọc được cấu trúc của log.
>
> 1. **Lọc theo trường:** tìm tất cả log có `user_id="cp5-test"` hoặc `event="ask_completed"` để xem một người dùng cụ thể đã hỏi bao nhiêu lần, lúc nào. Với `print("đã trả lời xong")` thì không biết câu trả lời đó là của ai.
> 2. **Tính toán và cảnh báo:** cộng dồn `cost_usd` hoặc `tokens_in`/`tokens_out` theo giờ, theo ngày, theo user, rồi đặt cảnh báo khi chi phí tăng bất thường. Chuỗi text tự do không có con số nào để cộng.

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
| 1 stage (bản đầu, `python:3.11`) | 1730 MB (1.73 GB) |
| Multi-stage (`python:3.11-slim`) | 310 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Khoảng 1.4 GB chênh lệch chủ yếu là những thứ chỉ cần lúc build, không cần lúc chạy. Phần lớn nhất là base image `python:3.11` bản đầy đủ: nó mang theo cả bộ công cụ build của Debian. Khi kiểm tra bản 1-stage, tôi thấy có sẵn `gcc 14.2`, các file header, thư viện dev và nhiều tiện ích hệ thống mà app FastAPI không dùng khi chạy. Ngoài ra còn cache của pip, và vì dùng `COPY . .` nên `/app` chứa cả `Dockerfile`, `railway.toml`, `.env.example`... Bản multi-stage chỉ cài package trong stage builder, rồi copy riêng thư mục venv và code `app/`, `utils/` sang base `slim`. Compiler và rác build nằm lại ở stage builder, không vào image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile của tôi, các layer tạo venv, `COPY requirements.txt`, `RUN pip install` và `COPY --from=builder` đều báo **CACHED**. Chỉ `COPY app` và `COPY utils` (cùng các layer sau nó) chạy lại, mất khoảng 0.1 giây. Lý do là `requirements.txt` không đổi, nên Docker không cần cài lại thư viện.
>
> Khi thử đặt `COPY . .` lên trước `pip install`: sửa một ký tự trong `main.py` làm checksum của layer `COPY . .` đổi, nên mọi layer phía sau mất cache. `pip install` phải chạy lại từ đầu, mất 12.1 giây trên máy tôi, và trên Railway còn lâu hơn. Nguyên tắc rút ra: thứ gì ít thay đổi (dependencies) thì copy trước, thứ hay thay đổi (code) thì copy sau.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện:
>
> 1. Code của tôi hoặc một thư viện có lỗ hổng cho phép thực thi lệnh từ xa (ví dụ `eval`/`pickle` trên dữ liệu người dùng, hay path traversal ghi được file).
> 2. Kẻ tấn công chạy được lệnh bên trong container, với đúng user của process Python. Nếu không có `USER`, đó là **root (uid 0)**.
> 3. Là root trong container, hắn có toàn quyền: sửa code trong `/app`, đọc mọi file và biến môi trường chứa secret, cài thêm công cụ.
> 4. uid 0 trong container cũng chính là uid 0 trên host (nếu không bật user namespace). Hắn có thể dùng các capability của root để tấn công kernel và thoát khỏi container, hoặc ghi vào volume được mount từ host (tệ nhất là `docker.sock`). Khi thoát ra, hắn là root trên máy host.
>
> `USER appuser` cắt chuỗi ở bước 2: process chạy bằng một user không đặc quyền, nên dù khai thác được lỗ hổng, kẻ tấn công chỉ có quyền của user đó. Hắn không ghi được vào file của root, không cài được package, không có capability để escape dễ dàng, và nếu có thoát ra thì cũng chỉ là một user thường trên host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request**. Cách làm: gửi 10 request lúc `10:00:59`. Chúng thuộc phút 10:00 và đều hợp lệ vì chưa vượt 10. Sang `10:01:00` bộ đếm reset về 0, gửi tiếp 10 request nữa và tất cả lại hợp lệ. Như vậy trong khoảng 2 giây (thực tế có thể chỉ vài phần mười giây quanh mốc giây 00) có 20 request lọt qua, gấp đôi hạn mức. Với sliding window 60 giây, tại bất kỳ thời điểm nào hệ thống cũng nhìn lại đúng 60 giây trước đó, nên 10 request lúc `:59` vẫn được tính khi sang `:00`. Request thứ 11 bị trả 429, giống như khi tôi test trên Railway.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Khác nhau:** rate limit đếm **số request** trong một khoảng thời gian ngắn (10 request/phút/user), để chống spam và bảo vệ server. Cost guard cộng dồn **số tiền** thực tế (dựa trên token) trong một khoảng dài (10 USD/tháng), để bảo vệ ngân sách. Một cơ chế đo tần suất, cơ chế kia đo chi phí.
>
> - **Rate limit cho qua, cost guard chặn:** một người dùng chỉ gửi 5 request/phút, không bao giờ chạm giới hạn, nhưng mỗi câu hỏi dán vào cả một tài liệu dài hàng nghìn token và gửi đều đặn suốt nhiều ngày. Hoặc nhiều user, mỗi người đều dưới 10/phút, nhưng cộng lại đã hết 10 USD của tháng. Lúc đó cost guard phải chặn.
> - **Rate limit chặn, cost guard cho qua:** như lúc tôi test, gửi 15 request `"test"` liên tiếp trong vài giây. Mỗi request chỉ tốn khoảng 0.00003 USD, tổng chi phí không đáng kể so với ngân sách, nhưng từ request thứ 11 rate limit đã trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối. Cả 3 container cùng lúc gọi Redis thất bại, nên endpoint health gộp trả lỗi (503) trên cả 3.
> 2. Orchestrator (Docker/Railway) thấy health check fail liên tiếp vượt ngưỡng, đánh dấu cả 3 container là **unhealthy**.
> 3. Nó coi process đã chết và **restart/kill cả 3 container gần như cùng lúc**. Load balancer không còn instance nào healthy để gửi traffic.
> 4. Trong lúc restart, mọi request đều lỗi, kể cả những request lẽ ra không cần Redis. Restart cũng không sửa được gì vì lỗi nằm ở Redis, không nằm ở app.
> 5. Các container mới khởi động, health check vẫn fail nếu Redis chưa về, nên chúng tiếp tục bị restart, có thể thành vòng lặp restart.
> 6. Sau 30 giây Redis trở lại, nhưng các container vẫn đang boot hoặc chờ đợt health check tiếp theo, nên thời gian sập thực tế dài hơn 30 giây. Toàn bộ lịch sử request đang xử lý dở cũng bị mất.
>
> Khi tách riêng: `/health` (liveness) chỉ kiểm tra process còn sống nên vẫn trả 200, không container nào bị restart. `/ready` trả 503 nên load balancer tạm ngừng gửi traffic, và ngay khi Redis về, `/ready` trả 200 là nhận traffic lại luôn, không mất thời gian khởi động.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Kết quả tôi quan sát được (3 instance dùng chung Redis, cùng một user, gọi xoay vòng):
>
> ```
> request 1 → :8011  history_length=0
> request 2 → :8012  history_length=2
> request 3 → :8013  history_length=4
> request 4 → :8011  history_length=6
> request 5 → :8012  history_length=8
> request 6 → :8013  history_length=10
> ```
>
> Con số tăng đều 2 mỗi lượt (1 câu hỏi + 1 câu trả lời) dù mỗi request rơi vào một instance khác, vì lịch sử nằm trong Redis dùng chung.
>
> Nếu lưu bằng dict Python, mỗi instance chỉ có bộ nhớ riêng và chỉ thấy các lượt rơi vào chính nó. Dãy số sẽ là `0, 0, 0, 2, 2, 2`: agent "quên" 2/3 cuộc trò chuyện, và người dùng thấy câu trả lời lúc nhớ lúc không tùy request rơi vào đâu. Khi một container restart, phần lịch sử của nó cũng mất hẳn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi:** sau khi deploy lên Railway, `/health` và `/ready` đều 200, nhưng gọi `/ask` có gửi header `X-API-Key: $AGENT_API_KEY` vẫn nhận `HTTP/2 401 {"detail":"invalid or missing API key"}`.
>
> **Tìm nguyên nhân:** vì có response 401 đúng định dạng của app, tôi biết request đã tới được app và app đang chạy bình thường, nên vấn đề nằm ở giá trị khóa chứ không phải mạng hay Redis. Tôi so sánh: biến `$AGENT_API_KEY` trong shell của tôi là khóa dùng cho local (hoặc chưa được nạp vào shell), trong khi trên Railway tôi đã đặt một khóa production riêng. Hai khóa khác nhau nên app từ chối.
>
> **Sửa:** tôi lưu khóa production vào một biến riêng `DEPLOY_API_KEY` trong shell (không ghi vào repo) và dùng `-H "X-API-Key: $DEPLOY_API_KEY"` khi gọi URL production. Sau đó `/ask` trả 200 và test rate limit ra đúng `200 ×9` rồi `429`. Bài học: tách rõ khóa local và khóa production, và khi thấy 401 thì kiểm tra giá trị khóa trước khi nghi ngờ code.
