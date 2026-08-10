# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Nhật Quang  
Mã học viên: 2A202601452

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên môi trường production, nếu quên set biến môi trường `API_TOKEN`:
> - Nếu để mặc định `"changeme"`: App vẫn khởi động bình thường nhưng bất kỳ ai dò ra token mặc định `"changeme"` đều có thể gọi API công khai, làm rò rỉ dữ liệu hoặc tiêu tốn ngân sách LLM .
> - Khi không có mặc định (Fail fast): App ném lỗi `ValidationError` và ngắt khởi động lập tức. 

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Log JSON thu được:
> `{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T10:54:13.640359+00:00", "client_id": "sv01", "prompt_tokens": 131, "completion_tokens": 45, "usd_cost": 4.665e-05}`
>
> Hai việc làm được với log JSON mà `print("đã trả lời xong")` không làm được:
> 1. **Lọc và tạo cảnh báo tự động theo client (Filtering & Alerting):** Các hệ thống xem log (Google Cloud Logging, Datadog) có thể tự động parse các trường `severity: "INFO"`, `client_id: "sv01"`, `usd_cost` để lọc lịch sử gọi API của từng client hoặc gửi cảnh báo khi chi phí `usd_cost` tăng bất thường.
> 2. **Thống kê và đối soát dữ liệu định lượng (Aggregations):** Có thể chạy truy vấn (SQL/Elasticsearch) để tính tổng chi phí `usd_cost` và tổng số token (`prompt_tokens`, `completion_tokens`) đã dùng nhờ cấu trúc dữ liệu định lượng chuẩn, điều mà chuỗi văn bản tự do từ `print()` không làm được.


---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

>| Bản | Dung lượng |
>|-----|-----------|
>| 1 stage (bản đầu) | ~1010 MB |
>| Multi-stage | ~209 MB |

> Giải thích: phần dung lượng cắt giảm được (~801 MB) bao gồm:
> 1. Chuyển từ base image `python:3.11` (chứa toàn bộ Debian OS, gcc compiler, header files...) sang base image siêu nhẹ `python:3.11-slim`.
> 2. Loại bỏ toàn bộ trình biên dịch C/C++, cache của `pip`, và các file wheel trung gian ở stage `builder`, stage runtime chỉ giữ lại thư viện Python đã được cài hoàn chỉnh.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

>- Khi sửa 1 ký tự trong `app/main.py`: Các layer trước bước `COPY app/ app/` (gồm base image, `COPY requirements.txt`, `RUN pip install`, `RUN useradd`) đều được dùng lại từ cache. Chỉ có layer `COPY app/ app/` và các lệnh sau nó (`HEALTHCHECK`, `CMD`) phải chạy lại.
>- Nếu đặt `COPY . .` trước `RUN pip install`: Mỗi lần sửa dù chỉ 1 ký tự code, layer `COPY . .` bị hỏng cache kéo theo layer `RUN pip install` phải tải và cài lại toàn bộ thư viện từ Internet, làm tốn rất nhiều thời gian build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện:
> 1. Code Python có lỗ hổng thực thi lệnh từ xa (RCE).
> 2. Kẻ tấn công lợi dụng lỗ hổng để mở shell trong container.
> 3. Do container chạy dưới quyền root, tiến trình có UID 0.
> 4. Kẻ tấn công lợi dụng lỗ hổng thoát khỏi container (container escape). Do có UID 0, khi ra máy host họ trở thành root của máy host.
> 
>Lệnh `USER appuser` cắt đứt chuỗi ở bước 3: Tiến trình chạy với quyền user thường (non-root). Dù kẻ tấn công chiếm được quyền chạy code trong container cũng bị giới hạn quyền tối thiểu, không thể can thiệp hệ thống container hay chiếm quyền root máy host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> - Header `WWW-Authenticate: Bearer` là bắt buộc theo chuẩn HTTP (RFC 6750) để báo cho HTTP client biết phương thức xác thực mà server yêu cầu là Bearer Token.
> - Trả cùng một thông báo lỗi giúp bảo mật: Tránh rò rỉ thông tin cho kẻ tấn công (ví dụ báo "sai token" vô tình xác nhận rằng header/scheme đã đúng, giúp hacker khoanh vùng và mò các trường còn lại dễ hơn).

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> - Client gửi được tối đa **10 request** liên tiếp trước khi bị lỗi 429 (do xô chỉ chứa tối đa `capacity = 10` token).
> - Nếu bỏ đoạn `min(capacity, ...)`: Con số sẽ thành **100 request** (vì sau 10 phút xô tự nạp `10 token/phút * 10 phút = 100 token`). Không có `min()`, client im lặng lâu sẽ tích trữ hàng ngàn token và có thể xả đợt tấn công bùng nổ (burst attack) làm sập server trong 1 giây.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> - Với hạn mức **$30/tháng**: Thiệt hại tối đa là **$30** (bị đốt sạch chỉ trong vài giờ đầu). Service bị chặn cho tới **đầu tháng sau** mới tự hồi phục.
> - Với hạn mức **$1/ngày**: Thiệt hại tối đa bị khoanh vùng ở mức **$1**. Sang **0h ngày hôm sau**, key chi tiêu tự hết hạn và service tự động hồi phục bình thường.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện:
> 1. Redis mất kết nối 30 giây.
> 2. Endpoint gộp kiểm tra Redis thất bại, trả về HTTP 500/503.
> 3. Orchestrator (K8s/Docker) coi container đã chết nên **kill và khởi động lại (restart)** cả 3 container.
> 4. Hệ thống rơi vào trạng thái downtime 100% trong lúc restart.
> 5. Container mới khởi động lại tiếp tục kiểm tra Redis (vẫn đang mất kết nối) ➔ Liveness probe thất bại ➔ Lại bị restart liên tục (CrashLoopBackOff), biến sự cố chập chờn tạm thời thành thảm họa sập dịch vụ toàn bộ.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi:** `pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings` -> `lỗi thiếu biến API_TOKEN`. Sau khi add biến API_TOKEN và redeploy vẫn bị lỗi.
- **Cách tìm nguyên nhân:** Xem tab View Logs trên Railway Dashboard, phát hiện app bị ném ngoại lệ `ValidationError` ngắt khởi động do thiếu biến môi trường `API_TOKEN`. Sau khi ra dashboard của Railway thì thấy 1 dòng yêu cầu redeploy toàn bộ thì mới được
- **Cách sửa:** Sau khi thêm API_TOKEN thì ra hẳn ngoài dự án (chỗ mà hiển thị dự án đang chạy và đang kết nối với database redis) và bấm vào nút "Redeploy this project".
