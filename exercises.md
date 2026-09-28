# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder `*Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phạm Quang Huy  Mã học viên: 2A202602900

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: mình deploy lên Railway nhưng quên khai báo `AGENT_API_KEY` trong
> phần Variables (hoặc gõ sai tên thành `AGENT_APIKEY`).
>
> - **Có fail fast:** container chết ngay lúc khởi động. Mình đã thử chạy
>   `Settings()` khi không có biến này và nhận được
>   `ValidationError: 1 validation error for Settings — agent_api_key Field required [type=missing]`.
>   Health check không bao giờ xanh nên platform đánh dấu deploy fail và giữ
>   bản cũ đang chạy. Mình đọc deploy log là thấy ngay tên biến bị thiếu, thêm
>   vào rồi deploy lại, mất khoảng 1 phút.
> - **Có mặc định `"changeme"`:** app vẫn khởi động bình thường, `/health` trả
>   200, nhìn bề ngoài mọi thứ đều ổn. Nhưng chuỗi `"changeme"` nằm ngay trong
>   source code trên GitHub, nên ai đọc repo cũng gọi được `/ask` với
>   `X-API-Key: changeme`. Không có log lỗi hay cảnh báo nào. Mình chỉ biết khi
>   hóa đơn LLM tăng vọt, hoặc khi user thật bị 402 vì người lạ đã tiêu hết
>   ngân sách.
>
> Nói ngắn gọn, fail fast biến một lỗ hổng bảo mật âm thầm thành một lỗi
> deploy ồn ào, và lỗi đó hiện ra lúc mình đang ngồi nhìn log chứ không phải
> vài tuần sau.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Mình gọi `/ask` 3 lần với `X-User-Id: sv01`. Đây là dòng log của lần thứ 2:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:55:26.718121+00:00", "user_id": "sv01", "tokens_in": 47, "tokens_out": 56, "cost_usd": 4.065e-05}
> ```
>
> (`tokens_in` tăng qua các lần gọi, lần lượt là 3 → 47 → 110, vì history được
> đưa vào prompt.)
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Lọc theo trường.** Có thể lấy đúng các request của một user
>    (`jq 'select(.user_id=="sv01")'`, hoặc filter `user_id:sv01` trên log
>    explorer của Railway/Datadog), hay chỉ lấy `level == "error"`. Khi chạy
>    3 instance, log của chúng trộn vào nhau, nhưng vẫn sắp xếp và ghép lại
>    được nhờ `timestamp` ISO theo UTC.
> 2. **Tính toán và đặt cảnh báo.** Có thể cộng `cost_usd` theo user hoặc
>    theo ngày, vẽ biểu đồ `tokens_in`/`tokens_out`, hay đặt alert khi tổng
>    cost trong 1 giờ vượt ngưỡng. Dòng `print` không có con số nào, không
>    biết của user nào, không có thời điểm chuẩn, nên chỉ đọc bằng mắt được
>    chứ máy không xử lý được.

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
| 1 stage (bản đầu) | 1730 MB (1,73 GB; nén còn 446 MB) |
| Multi-stage | 271 MB (nén còn 63,9 MB) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản multi-stage nhỏ hơn khoảng **6,4 lần**, tức gọn hơn khoảng 1,46 GB.
> Mình dùng `docker history` để tách từng layer:
>
> | Thành phần | 1 stage | Multi-stage |
> |---|---|---|
> | Base image | `python:3.11`, khoảng 1,63 GB | `python:3.11-slim`, khoảng 205 MB |
> | Layer thư viện Python | 95,1 MB (`RUN pip install`) | 65,5 MB (`COPY --from=builder /install`) |
> | Source code | 139 kB (`COPY . .`) | khoảng 140 kB (`app/` + `utils/`) |
>
> Phần chênh lệch gồm:
>
> 1. **Khoảng 1,4 GB là bộ công cụ build của base image đầy đủ.** `python:3.11`
>    được build trên `buildpack-deps` của Debian, nên mang theo `gcc`, `g++`,
>    `make`, `git`, các gói header `*-dev` (libssl, libffi, libpq, libxml...)
>    và nhiều thư viện hệ thống khác. Mình kiểm tra thì
>    `which gcc g++ make git` trong `agent:single` tìm thấy đủ cả bốn, còn
>    trong image multi-stage không có cái nào. Những công cụ này chỉ cần lúc
>    biên dịch, lúc chạy thì vô dụng. Chúng còn là bề mặt tấn công: kẻ xâm
>    nhập có sẵn compiler và git để dùng.
> 2. **Khoảng 30 MB ở layer pip.** Trong đó 17 MB là pip cache nằm ở
>    `/root/.cache/pip` (bản đầu không có `--no-cache-dir`). Phần còn lại là
>    file tạm của quá trình cài đặt. Bản multi-stage chỉ copy thư mục
>    `/install` đã cài xong, còn builder stage bị bỏ lại.
> 3. **Source code gần như không đổi.** Nhờ `.dockerignore` đã loại `.venv`,
>    `.git`, `.env` nên `COPY . .` chỉ có 139 kB. Nếu không có file đó thì
>    `.venv` sẽ làm image phình thêm vài trăm MB, và `.env` sẽ làm lộ secret.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình sửa `SERVICE_VERSION = "1.0.0"` thành `"1.0.1"` trong `app/main.py`
> rồi chạy `docker build --progress=plain`.
>
> **Được dùng lại từ cache (`CACHED`):**
> - Stage builder: `FROM python:3.11-slim`, `WORKDIR /build`,
>   `COPY requirements.txt .`, `RUN pip install ...`
> - Stage runtime: `WORKDIR /app`, `COPY --from=builder /install /usr/local`,
>   `RUN useradd ... appuser`
>
> **Phải chạy lại:** `COPY app ./app` (vì checksum file trong `app/` đã đổi)
> và `COPY utils ./utils`. `utils/` không đổi nhưng vẫn chạy lại, vì mọi layer
> đứng sau một layer bị invalidate đều mất cache. Cả lần build chỉ mất khoảng
> 0,3 giây.
>
> **Nếu đặt `COPY . .` trước `RUN pip install`:** mình đã thử bằng một
> Dockerfile phụ. Sửa một ký tự trong code làm checksum của layer `COPY . .`
> thay đổi, và `RUN pip install` đứng sau nó cũng mất cache theo. Kết quả là
> cài lại toàn bộ thư viện, bước đó mất **118,2 giây** (so với 0 giây khi
> dùng cache). Quy tắc rút ra: sắp xếp lệnh từ thứ ít thay đổi nhất
> (base image → dependency) tới thứ thay đổi nhiều nhất (source code).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện khi container chạy bằng root:
>
> 1. **Lỗ hổng trong app.** Ví dụ code dùng `eval`/`pickle.loads` trên dữ
>    liệu người dùng gửi lên, hoặc một thư viện trong `requirements.txt` có
>    lỗ hổng RCE. Kẻ tấn công chạy được lệnh shell trong container, với đúng
>    quyền của process uvicorn.
> 2. **Có root trong container.** Process là uid 0, nên kẻ tấn công có toàn
>    quyền trong container: `apt install` thêm công cụ, sửa file hệ thống,
>    cài backdoor vào `/usr/local/lib/python3.11`, và có các capability mặc
>    định của Docker như `CAP_DAC_OVERRIDE`, `CAP_SETUID`, `CAP_NET_RAW`.
> 3. **Thoát ra host.** Mặc định Docker không bật user namespace remap, nên
>    uid 0 trong container cũng là uid 0 trên host. Nếu container có mount
>    `/var/run/docker.sock` hoặc một thư mục của host, root ghi thẳng được vào
>    host. Nếu không có mount, kẻ tấn công có thể khai thác lỗ hổng của
>    runtime/kernel (ví dụ CVE-2019-5736 ghi đè binary `runc`, lỗi này cần
>    root trong container). Cả hai đường đều dẫn tới root trên máy host.
>
> **`USER appuser` cắt chuỗi ở bước 2.** Mình kiểm tra bằng
> `docker run --rm --entrypoint id day12-agent:prod` và nhận được
> `uid=10001(appuser) gid=10001(appuser)`. Khi đó RCE ở bước 1 chỉ cho kẻ tấn
> công một user thường: không có capability, không cài được gói, không ghi được
> vào thư mục hệ thống. Phần lớn kỹ thuật escape ở bước 3 cần root nên không
> dùng được, và nếu có thoát ra thì trên host đó cũng chỉ là uid 10001, không
> có quyền gì.
>
> Lưu ý: `USER` không bảo vệ được mọi thứ. Kẻ tấn công vẫn đọc được biến môi
> trường (`AGENT_API_KEY`, `REDIS_URL`), và vì Dockerfile của mình dùng
> `--chown=appuser`, họ vẫn sửa được code trong `/app`. Muốn chặt hơn thì để
> root sở hữu code (appuser chỉ có quyền đọc) và chạy container với
> `--read-only`.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> **Tối đa 20 request trong 2 giây.**
>
> Cách đạt được:
> - Lúc `10:00:59` gửi 10 request. Tất cả rơi vào bộ đếm của phút `10:00`,
>   vừa chạm hạn mức 10 nên vẫn hợp lệ.
> - Lúc `10:01:00` bộ đếm reset về 0 vì đã sang phút mới. Gửi tiếp 10 request,
>   tất cả rơi vào phút `10:01` và cũng hợp lệ.
>
> Như vậy có 20 request trong khoảng từ `10:00:59` tới `10:01:00,x`, gấp đôi
> hạn mức. Trong 2 giây liên tiếp chỉ vượt qua được đúng một mốc giây 00, nên
> không lên cao hơn 20 được.
>
> Với sliding window của mình, 10 request lúc `10:00:59` vẫn nằm trong sorted
> set tới `10:01:59`. Vì vậy lúc `10:01:00`, `hit_count` trả về 10 và request
> tiếp theo bị 429. Mình đã kiểm tra: user `sv02` gửi 10 request liên tục đều
> được 200, request thứ 11 bị 429 kèm `Retry-After: 60`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> | | Rate limit | Cost guard |
> |---|---|---|
> | Đếm cái gì | **Số request** | **Số tiền** (USD) |
> | Khung thời gian | 60 giây trượt | Theo tháng (`cost:{user}:2026-09`) |
> | Bảo vệ khỏi | Spam, burst, quá tải service | Hóa đơn LLM vượt ngân sách |
> | Mã lỗi | 429 + `Retry-After: 60`, chờ 1 phút là hết | 402, bị chặn tới tháng sau |
> | Lưu trong Redis | Sorted set, TTL 60 giây | Counter `INCRBYFLOAT`, TTL 40 ngày |
>
> **Rate limit cho qua nhưng cost guard chặn:** một script gọi đều đặn 9
> request/phút, suốt ngày đêm. Nó không bao giờ chạm mức 10/phút nên không
> bao giờ bị 429. Nhưng mỗi request đều có câu hỏi dài gần 2000 ký tự cộng
> thêm 20 message history trong prompt, nên chi phí cộng dồn liên tục. Tới
> một ngày giữa tháng, `spent` vượt `monthly_budget_usd = 10.0` và cost guard
> trả 402. Rate limit chỉ nhìn tốc độ nên không thấy được tổng số tiền này.
>
> **Cost guard cho qua nhưng rate limit chặn:** đầu tháng, một user mới
> (`sv02`, chưa tiêu gì) có client bị lỗi retry nên gửi 11 câu hỏi ngắn trong
> vài giây. Cả tháng họ mới tiêu khoảng 0,0003 USD, còn rất xa mức 10 USD,
> nhưng request thứ 11 vẫn bị 429 (mình đã thấy điều này khi chạy thử). Ở
> đây ngân sách không có vấn đề gì, chỉ có tốc độ gọi là quá nhanh.
>
> Trong `/ask`, cả hai đều được kiểm tra **trước** khi gọi `ask_llm`, vì tiền
> mất ở bước gọi LLM. Rate limit đứng trước để request spam bị loại luôn, không
> tốn thêm một lần đọc Redis cho cost.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện khi `/health` và `/ready` bị gộp thành một endpoint có kiểm tra
> Redis (sụp đổ dây chuyền):
>
> 1. Redis mất kết nối trong 30 giây, do sự cố mạng hoặc Redis đang restart.
> 2. Endpoint gộp, lúc này đang làm liveness probe, gọi Redis thất bại nên trả
>    503 ở **cả 3 container cùng lúc**.
> 3. Orchestrator (Docker, Kubernetes, Railway, Render...) hiểu liveness 503 là
>    process đã chết hoặc treo, nên đánh dấu cả 3 container là unhealthy.
> 4. Nó restart đồng loạt cả 3 container để "tự chữa" (SIGTERM rồi SIGKILL).
> 5. Trong lúc cả 3 đang khởi động lại, không còn instance nào nhận traffic.
>    Hệ thống mất trắng 100%: mọi request đều nhận 502, kể cả những request
>    không cần tới Redis.
> 6. Redis quay lại ở giây thứ 30, nhưng các container mới vẫn đang khởi động
>    rồi cùng lúc ồ ạt kết nối lại Redis (thundering herd). Kết quả là sự cố 30
>    giây của Redis biến thành outage của toàn hệ thống, kéo dài lâu hơn nhiều.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> - **Lưu lịch sử trong Redis (stateless):** load balancer chia request vào
>   container A, B hay C tùy lượt, nhưng container nào cũng đọc/ghi cùng một key
>   `history:<user_id>` trong Redis. Vì vậy `history_length` tăng đều sau mỗi
>   lượt hỏi: 0 → 2 → 4 → 6 → 8...
> - **Lưu lịch sử trong dict Python:** mỗi container có vùng RAM riêng, còn
>   request được chia kiểu round-robin nên rơi rải rác vào các container.
>   `history_length` sẽ nhảy lộn xộn, ví dụ: lượt 1 vào A → 0; lượt 2 vào B → 0
>   (B chưa từng nói chuyện với user này); lượt 3 vào C → 0; lượt 4 quay lại A
>   → 2. Người dùng thấy agent "mất trí nhớ" ngẫu nhiên: câu trước vừa nói, câu
>   sau đã quên.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi gặp phải:** deploy lên Render thành công ngay lần đầu, và `/health`,
> `/ready`, `/ask` không có key đều trả đúng. Nhưng khi chạy lệnh kiểm tra số 4
> trong `DEPLOYMENT.md` (gọi `/ask` với câu hỏi tiếng Việt `"Deploy là gì?"`)
> bằng `curl` trong Git Bash trên Windows, service trả về:
>
> ```
> HTTP/1.1 400 Bad Request
> {"detail":"There was an error parsing the body"}
> ```
>
> **Tìm nguyên nhân:**
> - Lỗi là 400 chứ không phải 401, dù lần chạy đó mình chưa điền
>   `DEPLOY_API_KEY`. Vậy request bị chặn ngay ở bước FastAPI đọc JSON body,
>   trước cả bước kiểm tra API key, tức là body gửi lên không phải JSON hợp lệ.
> - So sánh thì cùng lệnh đó với câu hỏi không dấu (`"Hello"`, `"test"`) vẫn
>   chạy bình thường. Vấn đề nằm ở ký tự tiếng Việt: chuỗi `-d '...'` mà `curl`
>   trong Git Bash gửi đi không được mã hóa UTF-8.
>
> **Cách sửa:** ghi body ra file với encoding UTF-8 rồi gửi bằng
> `--data-binary @ask_body.json`, kèm header
> `Content-Type: application/json; charset=utf-8`. Lần chạy lại trả
> `200 OK` kèm câu trả lời (output nằm trong `DEPLOYMENT.md`). Service trên
> cloud không cần sửa gì, lỗi nằm ở phía client gửi request.
>
> Cùng lúc đó mình còn gặp thêm một lỗi: loạt 15 request kiểm tra rate limit
> đều trả 401, vì `DEPLOY_API_KEY` trong `.env` đang để trống. Mình lấy lại
> khóa trong tab Environment của service trên Render, điền vào `.env`, và lần
> chạy lại ra đúng `200 × 9` rồi `429 × 6`.
