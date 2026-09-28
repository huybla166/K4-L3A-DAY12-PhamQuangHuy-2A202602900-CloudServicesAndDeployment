# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bên dưới mỗi câu bằng câu trả lời.
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
> - **Khi có fail fast:** container dừng ngay từ lúc khởi động. Mình đã thử chạy
>   `Settings()` khi không có biến này và thu được
>   `ValidationError: 1 validation error for Settings — agent_api_key Field required [type=missing]`.
>   Health check không bao giờ chuyển sang trạng thái xanh nên platform đánh dấu lần deploy là thất bại và giữ
>   phiên bản cũ tiếp tục chạy. Chỉ cần xem deploy log là mình thấy ngay biến bị thiếu, thêm
>   vào rồi triển khai lại, tổng cộng chỉ mất khoảng 1 phút.
> - **Có mặc định `"changeme"`:** app vẫn khởi động bình thường, `/health` trả
>   200, nhìn bề ngoài mọi thứ đều ổn. Nhưng chuỗi `"changeme"` nằm ngay trong
>   source code trên GitHub, nên ai đọc repo cũng gọi được `/ask` với
>   `X-API-Key: changeme`. Không hề có log lỗi hay cảnh báo nào. Mình chỉ phát hiện khi
>   chi phí LLM tăng bất thường, hoặc khi người dùng thật nhận 402 vì người lạ đã dùng hết
>   ngân sách.
>
> Tóm lại, fail fast biến một rủi ro bảo mật âm thầm thành một lỗi
> deploy dễ nhận thấy, và lỗi xuất hiện ngay khi mình đang theo dõi log thay vì
> chỉ lộ ra sau vài tuần.
---
### Câu 2 — Log cho máy đọc (CP1)
Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không thể làm được.
> Mình gọi `/ask` 3 lần với `X-User-Id: sv01`. Đây là dòng log của lần thứ 2:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:55:26.718121+00:00", "user_id": "sv01", "tokens_in": 47, "tokens_out": 56, "cost_usd": 4.065e-05}
> ```
>
> (`tokens_in` tăng qua các lần gọi, lần lượt là 3 → 47 → 110, vì history được
> đưa kèm vào prompt.)
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Lọc theo từng trường.** Có thể truy xuất chính xác các request của một user
>    (`jq 'select(.user_id=="sv01")'`, hoặc filter `user_id:sv01` trên log
>    explorer của Railway/Datadog), hay chỉ lấy `level == "error"`. Khi chạy
>    3 instance, log của các instance sẽ bị trộn lẫn, nhưng vẫn có thể sắp xếp và ghép lại
>    được nhờ `timestamp` ISO theo UTC.
> 2. **Tính toán và thiết lập cảnh báo.** Có thể cộng `cost_usd` theo user hoặc
>    theo ngày, vẽ biểu đồ `tokens_in`/`tokens_out`, hoặc đặt cảnh báo khi tổng
>    cost trong 1 giờ vượt ngưỡng. Dòng `print` không có con số nào, không
>    cho biết thuộc user nào, cũng không có mốc thời gian chuẩn, nên chỉ phù hợp để đọc bằng mắt
>    chứ không thuận tiện cho máy xử lý.
---
### Câu 3 — Kích thước image (CP2)
Build cả hai phiên bản và ghi lại dung lượng đo được thực tế:
```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```
| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1730 MB (1,73 GB; nén còn 446 MB) |
| Multi-stage | 271 MB (nén còn 63,9 MB) |
Hãy giải thích phần dung lượng chênh lệch đó đến từ đâu.
> Bản multi-stage nhỏ hơn khoảng **6,4 lần**, tức gọn hơn khoảng 1,46 GB.
> Mình dùng `docker history` để tách từng layer:
>
> | Thành phần | 1 stage | Multi-stage |
> |---|---|---|
> | Base image | `python:3.11`, khoảng 1,63 GB | `python:3.11-slim`, khoảng 205 MB |
> | Layer thư viện Python | 95,1 MB (`RUN pip install`) | 65,5 MB (`COPY --from=builder /install`) |
> | Source code | 139 kB (`COPY . .`) | khoảng 140 kB (`app/` + `utils/`) |
>
> Phần chênh lệch chủ yếu gồm:
>
> 1. **Khoảng 1,4 GB đến từ bộ công cụ build của base image đầy đủ.** `python:3.11`
>    được build trên `buildpack-deps` của Debian, nên mang theo `gcc`, `g++`,
>    `make`, `git`, các gói header `*-dev` (libssl, libffi, libpq, libxml...)
>    cùng nhiều thư viện hệ thống khác. Khi kiểm tra, mình thấy
>    `which gcc g++ make git` trong `agent:single` đều tìm thấy đủ cả bốn, trong khi
>    trong image multi-stage không có cái nào. Các công cụ này chỉ cần thiết trong lúc
>    biên dịch, còn khi runtime thì không cần. Chúng cũng làm tăng bề mặt tấn công vì kẻ xâm
>    nhập có sẵn compiler và git để sử dụng.
> 2. **Khoảng 30 MB nằm ở layer pip.** Trong đó có 17 MB pip cache tại
>    `/root/.cache/pip` (bản đầu không có `--no-cache-dir`). Phần dung lượng còn lại là
>    các file tạm sinh ra trong quá trình cài đặt. Bản multi-stage chỉ copy thư mục
>    `/install` đã cài hoàn tất, còn toàn bộ builder stage được loại bỏ.
> 3. **Dung lượng source code hầu như không thay đổi.** Nhờ `.dockerignore` đã loại `.venv`,
>    `.git`, `.env` nên `COPY . .` chỉ có 139 kB. Nếu không có file đó thì
>    `.venv` sẽ làm image phình thêm vài trăm MB, và `.env` sẽ làm lộ secret.
---
### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)
Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được tái sử dụng từ cache và layer nào phải chạy lại? Nếu đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?
> Mình sửa `SERVICE_VERSION = "1.0.0"` thành `"1.0.1"` trong `app/main.py`
> rồi chạy `docker build --progress=plain`.
>
> **Được dùng lại từ cache (`CACHED`):**
> - Stage builder: `FROM python:3.11-slim`, `WORKDIR /build`,
>   `COPY requirements.txt .`, `RUN pip install ...`
> - Stage runtime: `WORKDIR /app`, `COPY --from=builder /install /usr/local`,
>   `RUN useradd ... appuser`
>
> **Các layer phải chạy lại:** `COPY app ./app` (vì checksum file trong `app/` đã đổi)
> và `COPY utils ./utils`. `utils/` không thay đổi nhưng vẫn phải chạy lại, vì mọi layer
> nằm sau một layer bị invalidate đều không còn dùng được cache. Toàn bộ lần build chỉ mất khoảng
> 0,3 giây.
>
> **Nếu đặt `COPY . .` trước `RUN pip install`:** mình đã kiểm tra bằng một
> Dockerfile phụ. Sửa một ký tự trong code làm checksum của layer `COPY . .`
> thay đổi, và `RUN pip install` đứng sau nó cũng mất cache theo. Kết quả là
> cài lại toàn bộ thư viện, bước đó mất **118,2 giây** (so với 0 giây khi
> dùng cache). Kinh nghiệm rút ra là nên sắp xếp lệnh từ phần ít thay đổi nhất
> (base image → dependency) đến phần thay đổi thường xuyên nhất (source code).
---
### Câu 5 — Vì sao không chạy bằng root (CP2)
Container mặc định chạy với quyền root. Hãy mô tả chuỗi sự kiện từ "một lỗ hổng
trong code Python của bạn" đến "kẻ tấn công có quyền cao trên máy host", đồng thời
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.
> Chuỗi sự kiện trong trường hợp container chạy bằng root:
>
> 1. **Lỗ hổng ở ứng dụng.** Ví dụ code dùng `eval`/`pickle.loads` trên dữ
>    liệu người dùng gửi lên, hoặc một thư viện trong `requirements.txt` có
>    lỗ hổng RCE. Khi đó kẻ tấn công có thể chạy lệnh shell trong container với đúng
>    quyền mà process uvicorn đang có.
> 2. **Có quyền root bên trong container.** Process chạy với uid 0 nên kẻ tấn công có toàn
>    quyền trong container: `apt install` thêm công cụ, sửa file hệ thống,
>    cài backdoor vào `/usr/local/lib/python3.11`, và có các capability mặc
>    định của Docker như `CAP_DAC_OVERRIDE`, `CAP_SETUID`, `CAP_NET_RAW`.
> 3. **Từ container thoát ra host.** Theo mặc định, Docker không bật user namespace remap nên
>    uid 0 trong container cũng là uid 0 trên host. Nếu container có mount
>    `/var/run/docker.sock` hoặc mount một thư mục của host thì root có thể ghi trực tiếp vào
>    host. Nếu không có mount, kẻ tấn công vẫn có thể thử khai thác lỗ hổng của
>    runtime/kernel (ví dụ CVE-2019-5736 ghi đè binary `runc`, lỗi này cần
>    root trong container). Cả hai hướng đều có thể dẫn tới quyền root trên máy host.
>
> **`USER appuser` cắt chuỗi ở bước 2.** Mình kiểm tra bằng
> `docker run --rm --entrypoint id day12-agent:prod` và thu được
> `uid=10001(appuser) gid=10001(appuser)`. Khi đó, RCE ở bước 1 chỉ giúp kẻ tấn
> công chiếm được một user thường: không có capability, không cài được gói và không ghi được
> vào thư mục hệ thống. Phần lớn kỹ thuật escape ở bước 3 đòi hỏi quyền root nên không
> thể áp dụng; ngay cả khi thoát được ra host thì cũng chỉ mang uid 10001, không
> có quyền cao.
>
> Lưu ý: `USER` không bảo vệ được mọi thứ. Kẻ tấn công vẫn đọc được biến môi
> trường (`AGENT_API_KEY`, `REDIS_URL`), và vì Dockerfile của mình dùng
> `--chown=appuser`, họ vẫn sửa được code trong `/app`. Muốn chặt hơn thì để
> root sở hữu code (appuser chỉ được đọc) và chạy container với
> `--read-only`.
---
### Câu 6 — Cửa sổ trượt (CP3)
Rate limit hiện tại sử dụng sliding window 60 giây. Nếu đổi sang cách đếm theo
từng phút cố định trên đồng hồ (reset ở giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp với giới hạn 10/phút? Hãy giải thích cách đạt tới
con số đó.
> **Tối đa là 20 request trong 2 giây.**
>
> Cách đạt con số này:
> - Lúc `10:00:59` gửi 10 request. Tất cả đều được tính vào bộ đếm của phút `10:00`,
>   vừa đủ chạm giới hạn 10 nên vẫn được chấp nhận.
> - Lúc `10:01:00` bộ đếm được reset về 0 khi sang phút mới. Sau đó gửi thêm 10 request,
>   tất cả rơi vào phút `10:01` và cũng hợp lệ.
>
> Như vậy tổng cộng có 20 request trong khoảng từ `10:00:59` tới `10:01:00,x`, tức gấp đôi
> giới hạn. Trong 2 giây liên tục chỉ có thể đi qua đúng một mốc giây 00, vì vậy
> không thể vượt quá 20.
>
> Với cơ chế sliding window hiện tại, 10 request lúc `10:00:59` vẫn nằm trong sorted
> set tới `10:01:59`. Do đó tại `10:01:00`, `hit_count` trả về 10 và request
> tiếp theo sẽ bị 429. Mình đã kiểm tra thực tế: user `sv02` gửi 10 request liên tục đều
> được 200, request thứ 11 bị 429 kèm `Retry-After: 60`.
---
### Câu 7 — Rate limit và cost guard (CP3)
Hai cơ chế này khác nhau như thế nào? Hãy đưa ra một tình huống mà rate limit cho phép
nhưng cost guard buộc phải chặn, cùng một tình huống theo chiều ngược lại.
> | | Rate limit | Cost guard |
> |---|---|---|
> | Đếm cái gì | **Số request** | **Số tiền** (USD) |
> | Khung thời gian | 60 giây trượt | Theo tháng (`cost:{user}:2026-09`) |
> | Bảo vệ khỏi | Spam, burst, quá tải service | Hóa đơn LLM vượt ngân sách |
> | Mã lỗi | 429 + `Retry-After: 60`, chờ 1 phút là hết | 402, bị chặn tới tháng sau |
> | Lưu trong Redis | Sorted set, TTL 60 giây | Counter `INCRBYFLOAT`, TTL 40 ngày |
>
> **Trường hợp rate limit cho qua nhưng cost guard chặn:** một script gửi đều đặn 9
> request/phút, suốt ngày đêm. Nó không bao giờ chạm mức 10/phút nên không
> bao giờ bị 429. Tuy nhiên, mỗi request lại có câu hỏi dài gần 2000 ký tự kèm
> 20 message history trong prompt, khiến chi phí tiếp tục cộng dồn. Đến
> một ngày giữa tháng, `spent` vượt `monthly_budget_usd = 10.0` và cost guard
> trả 402. Rate limit chỉ theo dõi tốc độ gọi nên không phát hiện được tổng chi phí này.
>
> **Trường hợp cost guard cho qua nhưng rate limit chặn:** ở đầu tháng, một user mới
> (`sv02`, chưa tiêu gì) có client gặp lỗi retry và gửi 11 câu hỏi ngắn chỉ trong
> vài giây. Trong cả tháng họ mới tiêu khoảng 0,0003 USD, còn rất xa mức 10 USD,
> nhưng request thứ 11 vẫn nhận 429 (mình đã quan sát điều này khi thử). Trong
> trường hợp này ngân sách vẫn ổn, vấn đề chỉ nằm ở tốc độ gọi quá cao.
>
> Trong `/ask`, cả hai đều được kiểm tra **trước** khi gọi `ask_llm`, vì tiền
> phát sinh ở bước gọi LLM. Rate limit được đặt trước để loại request spam ngay lập tức, không
> phải tốn thêm một lượt đọc Redis cho cost guard.
---
### Câu 8 — /health khác /ready (CP4)
Nếu gộp hai endpoint thành một và để endpoint đó kiểm tra Redis, điều gì sẽ xảy ra với cụm
3 container khi Redis mất kết nối trong 30 giây? Hãy trình bày theo đúng trình tự sự kiện.
> Thứ tự sự kiện khi `/health` và `/ready` bị gộp thành một endpoint có kiểm tra
> Redis (dẫn tới hiệu ứng sụp đổ dây chuyền):
>
> 1. Redis bị mất kết nối trong 30 giây, có thể do lỗi mạng hoặc Redis đang restart.
> 2. Endpoint đã gộp, khi đang được dùng cho liveness probe, gọi Redis thất bại nên trả
>    503 ở **cả 3 container cùng lúc**.
> 3. Orchestrator (Docker, Kubernetes, Railway, Render...) diễn giải liveness 503 là
>    process đã chết hoặc treo, vì vậy đánh dấu cả 3 container là unhealthy.
> 4. Orchestrator sau đó restart đồng thời cả 3 container để "tự chữa" (SIGTERM rồi SIGKILL).
> 5. Trong thời gian cả 3 container đang khởi động lại, không còn instance nào có thể nhận traffic.
>    Hệ thống bị mất khả dụng hoàn toàn: mọi request đều nhận 502, kể cả các request
>    không hề phụ thuộc vào Redis.
> 6. Đến giây thứ 30, Redis hoạt động trở lại nhưng các container mới vẫn đang khởi động
>    sau đó đồng loạt kết nối lại Redis (thundering herd). Hệ quả là sự cố Redis kéo dài 30
>    giây có thể biến thành outage toàn hệ thống và kéo dài lâu hơn đáng kể.
---
### Câu 9 — Stateless (CP4)
Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay cho Redis, giá trị đó sẽ thay đổi như thế nào?
> - **Khi lưu lịch sử trong Redis (stateless):** load balancer phân phối request tới
>   container A, B hay C theo từng lượt, nhưng container nào cũng đọc/ghi chung một key
>   `history:<user_id>` trong Redis. Vì vậy `history_length` tăng đều sau mỗi
>   lượt hỏi: 0 → 2 → 4 → 6 → 8...
> - **Nếu lưu lịch sử trong dict Python:** mỗi container sở hữu vùng RAM riêng, trong khi
>   request được phân phối theo kiểu round-robin nên rơi vào các container khác nhau.
>   `history_length` sẽ nhảy lộn xộn, ví dụ: lượt 1 vào A → 0; lượt 2 vào B → 0
>   (B chưa từng xử lý cuộc trò chuyện của user này); lượt 3 vào C → 0; lượt 4 quay lại A
>   → 2. Người dùng sẽ cảm thấy agent "mất trí nhớ" một cách ngẫu nhiên: câu trước vừa trao đổi, câu
>   sau đã không còn nhớ.
---
### Câu 10 — Deploy thật (CP5)
Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
xác định nguyên nhân bằng cách nào và đã sửa như thế nào?
> **Lỗi mình gặp:** deploy lên Render thành công ngay lần đầu, và `/health`,
> `/ready`, `/ask` không có key đều phản hồi đúng. Tuy nhiên, khi chạy lệnh kiểm tra số 4
> trong `DEPLOYMENT.md` (gọi `/ask` với câu hỏi tiếng Việt `"Deploy là gì?"`)
> bằng `curl` trong Git Bash trên Windows, service trả về:
>
> ```
> HTTP/1.1 400 Bad Request
> {"detail":"There was an error parsing the body"}
> ```
>
> **Cách mình tìm nguyên nhân:**
> - Response là 400 chứ không phải 401, mặc dù ở lần chạy đó mình chưa điền
>   `DEPLOY_API_KEY`. Điều đó cho thấy request bị chặn ngay tại bước FastAPI parse JSON body,
>   trước khi đến bước kiểm tra API key, nghĩa là body gửi lên không phải JSON hợp lệ.
> - So sánh thì cùng lệnh đó với câu hỏi không dấu (`"Hello"`, `"test"`) vẫn
>   chạy bình thường. Vấn đề nằm ở ký tự tiếng Việt: chuỗi `-d '...'` mà `curl`
>   trong Git Bash gửi đi không được encode theo UTF-8.
>
> **Cách khắc phục:** mình ghi body ra file với encoding UTF-8 rồi gửi bằng
> `--data-binary @ask_body.json`, kèm header
> `Content-Type: application/json; charset=utf-8`. Ở lần chạy lại, server trả
> `200 OK` kèm câu trả lời (output nằm trong `DEPLOYMENT.md`). Phía service trên
> cloud không cần thay đổi gì vì lỗi nằm ở client gửi request.
>
> Đồng thời mình cũng gặp thêm một lỗi khác: loạt 15 request dùng để kiểm tra rate limit
> đều trả 401, vì `DEPLOY_API_KEY` trong `.env` đang để trống. Mình lấy lại
> khóa trong tab Environment của service trên Render, điền vào `.env`, và lần
> chạy lại ra đúng `200 × 9` rồi `429 × 6`.