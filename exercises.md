# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Việt Tùng  Mã học viên: 2A202601876

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> "Chết sớm" khi thiếu env là một dạng safety rail — nó hy sinh sự tiện lợi
> (app không chạy) để đổi lấy an toàn và dễ debug (không chạy sai). Tình huống
> cụ thể: lúc deploy lên Railway, tôi quên set biến `API_TOKEN` trên dashboard.
> App crash ngay với `ValidationError: api_token Field required` trong log —
> tôi biết ngay chỗ sai và sửa được trong vài phút. Nếu `api_token` có giá trị
> mặc định `"changeme"`, app sẽ khởi động bình thường, `/healthz` vẫn xanh, tôi
> tưởng deploy thành công — nhưng thực ra ai đoán được `"changeme"` cũng gọi
> được `/chat` công khai, tiêu ngân sách của tôi mà tôi không hề biết cho tới
> khi thấy chi phí bất thường hoặc bị lạm dụng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật:
> ```json
> {"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T08:49:48.766580+00:00", "client_id": "sv-test", "prompt_tokens": 41, "completion_tokens": 45, "usd_cost": 3.315e-05}
> ```
> Hai việc làm được: (1) lọc log theo `client_id` để xem riêng một khách hàng
> đang gọi gì, gặp lỗi gì — `print` chỉ ra một chuỗi phẳng, không có trường để
> `grep`/query theo. (2) Tính tổng `usd_cost` theo ngày bằng cách cộng dồn
> trường số trong các dòng JSON (ví dụ đưa vào BigQuery/Cloud Logging rồi
> `SUM(usd_cost) GROUP BY DATE(ts)`) — `print` không có cấu trúc để máy đọc
> ra con số, phải tự viết regex đoán mò rất dễ sai.

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
| 1 stage (bản đầu) | 1730 MB |
| Multi-stage | 296 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch ~1.43GB đến từ 3 nguồn: (1) base image `python:3.11` đầy đủ mang
> theo toàn bộ toolchain build (gcc, make, header files...) mà runtime không
> cần, trong khi `python:3.11-slim` bỏ hết phần đó; (2) bản 1-stage giữ
> nguyên pip cache và các gói build-time (wheel, setuptools mở rộng...) trong
> layer cuối cùng vì mọi thứ nằm chung 1 stage; (3) multi-stage chỉ copy đúng
> thư mục `/root/.local` (kết quả `pip install --user`) từ stage `builder`
> sang stage runtime, không mang theo compiler hay cache pip đã dùng để cài.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sửa `SERVICE_VERSION` từ `"1.0.0"` thành `"1.0.1"` rồi build lại, log thật:
> ```
> [builder 3/4] COPY requirements.txt .        CACHED
> [builder 4/4] RUN pip install ...            CACHED
> [stage-1 4/6] COPY --from=builder ...        CACHED
> [stage-1 5/6] COPY . .                       (chạy lại — có thay đổi)
> [stage-1 6/6] RUN chown -R appuser:appuser   (chạy lại — vì input đổi)
> ```
> `pip install` dùng lại cache vì Docker so khớp theo nội dung layer TRƯỚC nó
> (`requirements.txt` không đổi → hash layer giống hệt → cache hit). Chỉ 2
> layer cuối (COPY code + chown) chạy lại vì `app/main.py` đổi nội dung.
>
> Nếu đặt `COPY . .` LÊN TRƯỚC `RUN pip install`: sửa 1 ký tự bất kỳ trong
> code cũng làm layer `COPY . .` đổi hash → mọi layer PHÍA SAU nó (kể cả
> `pip install`) mất cache theo, dù `requirements.txt` không hề đổi — build
> lại chậm hẳn vì cài lại toàn bộ thư viện mỗi lần sửa 1 dòng code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi: (1) code Python trong `/chat` có lỗ hổng RCE (ví dụ deserialize input
> không an toàn) → (2) kẻ tấn công gửi payload khai thác, thực thi được lệnh
> shell trong container → (3) nếu container chạy bằng root, lệnh đó chạy với
> UID 0 — ghi được vào bất kỳ file nào trong container, đọc được secret, cài
> backdoor → (4) từ đó tìm cách thoát container (khai thác lỗ hổng kernel,
> Docker socket bị mount nhầm, capability thừa...) để chạm tới host, và vì đã
> có UID 0 sẵn nên bước thoát dễ hơn nhiều so với chạy bằng user thường.
>
> `USER appuser` cắt chuỗi tại bước (3): payload vẫn thực thi được (không chặn
> được RCE), nhưng chỉ chạy với quyền `appuser` — không ghi được ngoài `/app`,
> không đọc được file hệ thống nhạy cảm, và nếu có thoát được container thì
> cũng thoát ra với quyền thường chứ không phải root. Nó giảm đáng kể blast
> radius, nhưng không phải lá chắn cuối: muốn chặn triệt để bước thoát ra host
> vẫn cần bảo vệ thêm kernel/runtime, capabilities, mounts, Docker socket, và
> user namespace.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` là header bắt buộc theo chuẩn HTTP cho response
> 401 (RFC 7235) — nó nói cho client biết *phải xác thực bằng cách nào* (scheme
> gì) để thử lại đúng cách, thay vì đoán mò. Thiếu header này, client (đặc
> biệt là thư viện HTTP tự động) không biết nên gửi lại request kiểu gì.
>
> Trả **cùng một** thông báo cho cả 3 trường hợp (thiếu header, sai scheme,
> sai token) vì nói rõ sai ở đâu là tặng thông tin cho kẻ đang dò: nếu server
> trả lỗi khác nhau tuỳ trường hợp, kẻ tấn công dò được "token gần đúng chưa"
> hay "chỉ cần đổi scheme" — biến 401 thành một oracle giúp brute-force nhanh
> hơn. Đây cùng lý do với việc dùng `secrets.compare_digest` thay vì `==` khi
> so token: không để bất kỳ tín hiệu nào (thời gian, nội dung lỗi) rò rỉ thông
> tin về đáp án đúng.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> **10 request** rồi bị 429. Xô có sức chứa tối đa 10 (`min(capacity, tokens)`
> trong `available()`), nên dù im lặng bao lâu, token tích được cũng bị chặn
> lại ở 10 — không bao giờ vượt sức chứa.
>
> Nếu bỏ `min(capacity, ...)`: sau 10 phút im lặng, token tích được tính thẳng
> theo công thức `(now - last) * refill_per_second` = 10 phút × 10 token/phút
> = **100 token**, tức gửi được **100 request** liên tiếp trước khi cạn. Đây
> chính là lỗ hổng bug nêu trong docstring: bỏ dòng `min(...)` thì client im
> lặng đủ lâu (một ngày) sẽ tích được con số khổng lồ (14.400 token) và bắn
> hết trong một giây — token bucket lúc đó không còn giới hạn được gì cả, mất
> hết ý nghĩa rate limit.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Hạn mức **$30/tháng**: nếu sự cố xảy ra đầu tháng, client gần như chưa tiêu
> gì, nên thiệt hại tối đa lên tới gần **$30** trong một sự cố duy nhất — và
> service chỉ tự hồi phục khi sang **tháng sau** (có thể tới gần 30 ngày sau
> nếu sự cố xảy ra ngay đầu kỳ).
>
> Hạn mức **$1/ngày** (cách lab dùng, key theo `CostGuard.today()` UTC):
> thiệt hại tối đa mỗi sự cố chỉ **$1**, ít hơn 30 lần. Vì key ngân sách đổi
> theo ngày (`spend:{client_id}:{ngày}`), qua **0h UTC hôm sau** là hạn mức tự
> reset — service tự hồi phục mà không cần ai can thiệp, dù sự cố bắt đầu lúc
> 2h sáng thì cũng chỉ kéo dài tối đa vài chục giờ trước khi tự chặn ở $1 rồi
> chờ ngày mới. $1/ngày ít thiệt hại hơn hẳn và tự hồi phục nhanh hơn hẳn.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Không phải "3 container restart ngay lập tức". Đúng thứ tự: Redis mất kết
> nối → endpoint gộp bắt đầu trả fail (vì giờ nó check cả Redis) → từng
> container bị đánh dấu NotReady → load balancer/Service loại cả 3 container
> khỏi danh sách nhận traffic (readiness fail đúng như thiết kế, phần này
> không sai) → nhưng vì cùng một endpoint đó GIỜ CŨNG là liveness probe, nếu
> nó fail đủ số lần liên tiếp vượt threshold, orchestrator (kubelet/Docker)
> hiểu nhầm là process đã chết và bắt đầu **restart từng container** — dù
> process Python vẫn đang chạy bình thường, chỉ là mất kết nối Redis tạm thời.
> Cả cụm 3 container bị restart gần như cùng lúc (vì cùng phụ thuộc 1 Redis),
> có thể tạo thành restart loop khi container mới khởi động lại vẫn thấy Redis
> chưa kịp hồi, tiếp tục fail liveness, tiếp tục bị restart — đúng cái vụ tai
> nạn mà tách `/healthz` (không phụ thuộc gì) khỏi `/readyz` (được phép phụ
> thuộc Redis) ngăn chặn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi: container build xong nhưng cứ crash-loop ngay khi start, log Railway
> hiện:
> ```
> Error: Invalid value for '--port': '$PORT' is not a valid integer.
> ```
> lặp lại liên tục cho tới khi Railway dừng thử.
>
> Tìm nguyên nhân: đọc log thấy uvicorn nhận nguyên văn chuỗi `$PORT` làm giá
> trị port thay vì con số — nghĩa là biến môi trường không được shell expand.
> Nguyên nhân là `railway.toml` có khai `startCommand = "uvicorn ... --port
> $PORT"`, và Railway chạy `startCommand` không qua shell nên `$PORT` không
> được thay thế, bị truyền thẳng dưới dạng literal string vào uvicorn.
>
> Sửa: bỏ hẳn dòng `startCommand` khỏi `railway.toml`, để Railway dùng `CMD`
> có sẵn trong `Dockerfile` — CMD đó viết dạng `["sh", "-c", "uvicorn ...
> --port ${PORT:-8000}"]`, chạy qua `sh -c` nên `$PORT` được shell expand
> đúng thành số cổng Railway cấp. Sau khi sửa, ứng dụng đọc được port cloud
> cấp, bind trên `0.0.0.0` và vượt qua bước health check. Deployment chuyển
> sang trạng thái running bình thường.
