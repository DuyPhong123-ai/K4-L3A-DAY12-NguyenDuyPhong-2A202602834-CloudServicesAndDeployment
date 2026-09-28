# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng câu trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Duy Phong  Mã học viên: 2A202602834

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để giá trị mặc định là `"changeme"`, khi triển khai lên môi trường staging hoặc production, lập trình viên/DevOps rất dễ quên thiết lập biến môi trường `AGENT_API_KEY`. Ứng dụng vẫn khởi động trơn tru và mở cổng ra internet. Kẻ xấu có thể dùng botnet quét các endpoint công khai và thử các API key mặc định phổ biến như `"changeme"`. Khi đó, kẻ tấn công dễ dàng chiếm quyền gọi API, làm cạn kiệt ngân sách LLM hoặc truy cập dữ liệu nhạy cảm mà hệ thống không hề phát tín hiệu cảnh báo. 
Nhờ cơ chế "fail fast", `Settings` của Pydantic sẽ ném ra lỗi `ValidationError` ngay lúc khởi động (trước khi ứng dụng bind port nhận traffic), khiến container crash ngay lập tức. Nền tảng cloud (như Railway/Kubernetes) sẽ báo đỏ trạng thái deployment lập tức, buộc người quản trị phải cấu hình API key an toàn trước khi ứng dụng được đưa vào hoạt động.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"timestamp": "2026-09-28T09:30:15.123456Z", "level": "INFO", "event": "ask_llm", "user_id": "cp5-test", "cost_usd": 0.00015, "history_length": 3, "status": "success"}
```

Hai việc làm được với dòng log có cấu trúc (JSON) mà `print("đã trả lời xong")` không làm được:
1. **Truy vấn và tổng hợp tự động (Querying & Aggregation):** Các hệ thống thu thập log tập trung (như ElasticSearch, Grafana Loki, Datadog) có thể tự động parse các trường dữ liệu số và chuỗi. Từ đó ta có thể dễ dàng chạy các truy vấn chính xác như: lọc tất cả request của một `user_id` cụ thể, tính tổng chi phí `cost_usd` theo giờ hoặc tính độ dài hội thoại trung bình. Với `print` chuỗi tự do, ta phải viết biểu thức chính quy (regex) rất phức tạp và dễ vỡ khi đổi câu chữ.
2. **Thiết lập Giám sát và Cảnh báo theo thời gian thực (Real-time Alerting):** Có thể thiết lập trigger tự động cảnh báo lên Slack/PagerDuty khi trường `status != "success"` tăng đột biến, hoặc khi có request tiêu tốn `cost_usd` vượt ngưỡng an toàn trong một khung thời gian ngắn, giúp phát hiện sự cố ngay lập tức.

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
| 1 stage (bản đầu) | ~465 MB |
| Multi-stage | ~185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch gần 280 MB bao gồm:
1. Cache của trình quản lý gói pip (`~/.cache/pip`) sinh ra trong quá trình tải và cài đặt các wheel/package.
2. Các file mã nguồn trung gian, file header C/C++ và các công cụ biên dịch tạm thời cần thiết khi build package.
3. Lịch sử các layer trung gian trong image single-stage.
Trong Dockerfile multi-stage, stage `runtime` chỉ sao chép thư mục `site-packages` sạch và mã nguồn ứng dụng từ stage `builder`. Tất cả cache pip, tệp thừa và công cụ build đều bị loại bỏ hoàn toàn khỏi image cuối cùng, giúp image nhẹ hơn, deploy nhanh hơn và giảm diện tích tấn công (attack surface).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Khi sửa một ký tự trong `app/main.py`:
  + Các layer phía trước như `FROM ...`, `WORKDIR ...`, `COPY requirements.txt .`, `RUN pip install ...` đều được Docker tái sử dụng từ cache (`CACHED`) vì file `requirements.txt` không thay đổi hash.
  + Chỉ từ layer `COPY . .` và các layer kế tiếp (`USER appuser`, `EXPOSE`, `CMD`) mới bị invalidate cache và phải chạy lại. Do đó thời gian build lại chỉ mất khoảng 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  + Mỗi lần sửa đổi bất kỳ file code nào (dù chỉ 1 ký tự trong `app/main.py`), cache của layer `COPY . .` sẽ bị mất hiệu lực ngay lập tức.
  + Kéo theo toàn bộ các layer phía sau nó, bao gồm cả `RUN pip install`, đều bị buộc phải thực thi lại từ đầu. Quá trình build sẽ phải tải lại và cài đặt lại toàn bộ thư viện, làm tiêu tốn rất nhiều băng thông và khiến mỗi lần build mất thêm vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện dẫn tới chiếm quyền máy host:
  1. Ứng dụng Python tồn tại lỗ hổng (ví dụ Remote Code Execution thông qua `pickle.loads`, `eval`, command injection hoặc lỗ hổng buffer overflow từ thư viện C phụ thuộc).
  2. Kẻ tấn công khai thác lỗ hổng và chiếm được interactive shell bên trong container.
  3. Mặc định không có chỉ thị `USER`, tiến trình Python chạy dưới quyền UID 0 (`root`). Do đó kẻ tấn công sở hữu quyền root bên trong container, có khả năng sửa đổi file hệ thống container, cài đặt công cụ thăm dò và khai thác kernel.
  4. Nếu container có mount các volume nhạy cảm (như docker socket `/var/run/docker.sock` hoặc thư mục `/etc` của host), hoặc nếu kernel của máy host tồn tại lỗ hổng leo thang/vượt ngục (container breakout như Dirty COW, cgroup release_agent), kẻ tấn công với quyền root container sẽ vượt qua ranh giới namespace/cgroup và chiếm quyền root trực tiếp trên máy host.
- Điểm cắt đứt:
  Lệnh `USER appuser` cắt đứt chuỗi ngay tại **Bước 3**. Khi chạy dưới quyền non-root (UID 10001) với đặc quyền tối thiểu: kẻ tấn công dù có chiếm được shell cũng không thể chỉnh sửa file hệ thống container, không thể ghi đè các socket nhạy cảm, và không có đủ capabilities để thực hiện các kỹ thuật khai thác hạt nhân nhằm vượt ngục container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong vòng 2 giây liên tiếp.

Cách đạt được:
- Với bộ đếm theo phút cố định (fixed window), hạn mức được reset về 0 vào mỗi đầu phút (giây `00`).
- Người dùng gửi 10 request vào giây `00:00:59` (giây cuối cùng của phút thứ nhất) -> hệ thống chấp nhận vì 10/10 req cho phút đó.
- Ngay tại giây tiếp theo `00:01:00` (giây đầu tiên của phút thứ hai), bộ đếm được reset về 0. Người dùng gửi tiếp 10 request nữa -> hệ thống tiếp tục chấp nhận 10/10 req cho phút mới.
- Tổng cộng: Trong khoảng thời gian chỉ 2 giây (từ `00:00:59` đến `00:01:00`), hệ thống đã phải gánh tới 20 request, gây spike tải gấp đôi hạn ngạch cho phép.
Ngược lại, sliding window 60s tính tổng request trong đúng khoảng thời gian liên tục `[hiện_tại - 60s, hiện_tại]`, do đó tại bất kỳ 2 giây nào cũng không thể vượt quá 10 request.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Điểm khác nhau:
  + **Rate limit** kiểm soát **tốc độ / tần suất request** trong một đơn vị thời gian ngắn (ví dụ: tối đa 10 request / 60 giây) nhằm bảo vệ máy chủ khỏi nguy cơ nghẽn mạng, quá tải CPU/RAM hoặc tấn công DoS/spam.
  + **Cost guard** kiểm soát **tổng chi phí tài chính / hạn mức ngân sách** trong một chu kỳ dài (ví dụ: tối đa 10 USD / tháng) nhằm bảo vệ doanh nghiệp khỏi bị thâm hụt tài chính do các API tính phí bên thứ ba (như token của OpenAI/Gemini/Anthropic).
- Tình huống Rate limit cho qua nhưng Cost guard chặn:
  Đầu ngày, người dùng chỉ gửi duy nhất 1 request (tần suất cực thấp, 1 req/phút << 10 req/phút nên Rate limit cho qua). Tuy nhiên, tổng chi tiêu của hệ thống trong tháng đã đạt 10.00 USD (hết budget tháng). Cost guard kiểm tra thấy `spent >= budget` nên ngay lập tức chặn lại và trả về lỗi `HTTP 402 Payment Required`.
- Tình huống Cost guard cho qua nhưng Rate limit chặn:
  Vào ngày đầu tiên của tháng mới, ngân sách vừa được cấp mới 10 USD và chi tiêu hiện tại là 0 USD. Một người dùng gửi liên tiếp 15 request chỉ trong vòng 3 giây. Cost guard thấy ngân sách còn rất dồi dào nên không có lý do chặn, nhưng Rate limit sẽ lập tức chặn từ request thứ 11 trở đi và trả về lỗi `HTTP 429 Too Many Requests` do vi phạm ngưỡng tần suất 10 req/phút.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis gặp sự cố tạm thời (như restart, network blip) và mất kết nối trong vòng 30 giây.
2. Bộ điều phối (Container Orchestrator như Docker Swarm hoặc Kubernetes) định kỳ gọi liveness probe (nay đã gộp chung kiểm tra Redis) tới cả 3 container agent.
3. Vì Redis không kết nối được, endpoint kiểm tra trả về mã lỗi 503 Service Unavailable trên cả 3 container.
4. Orchestrator coi việc liveness probe thất bại là dấu hiệu ứng dụng đã chết hoặc rơi vào deadlock. Do đó, orchestrator lập tức gửi tín hiệu tiêu diệt (SIGKILL) và khởi động lại (restart) toàn bộ 3 container.
5. Khi 3 container mới khởi động lại, chúng lại lập tức kiểm tra Redis trong khi Redis vẫn chưa hoàn tất 30 giây phục hồi. Liveness probe tiếp tục fail, orchestrator lại tiếp tục restart liên tục, rơi vào vòng lặp sự cố `CrashLoopBackOff`.
6. Ngay cả khi Redis đã online trở lại sau 30 giây, dịch vụ vẫn bị gián đoạn thêm một khoảng thời gian dài vì cả 3 container đều đang bị kẹt trong backoff delay của orchestrator, biến một gián đoạn mạng tạm thời thành thảm họa sập toàn diện dịch vụ (cascading failure). 
(Nếu tách riêng `/health` và `/ready`: `/health` vẫn 200 giúp container không bị restart; chỉ `/ready` fail để load balancer tạm dừng chuyển traffic tới cho đến khi Redis kết nối lại).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu lịch sử trong Redis (Stateless):
  Dù các request được reverse proxy/load balancer phân phối ngẫu nhiên (round-robin) đến bất kỳ container nào trong số 3 container (agent-1, agent-2, agent-3), tất cả các container đều truy vấn và cập nhật vào chung một cơ sở dữ liệu Redis. Vì vậy giá trị `history_length` luôn tăng đều đặn, chính xác và nhất quán: 1, 2, 3, 4, 5...
- Nếu lịch sử được lưu trong `dict` Python trong RAM (Stateful):
  Vì mỗi container chạy trong một tiến trình độc lập và có vùng nhớ RAM hoàn toàn cô lập:
  + Request 1 rơi vào container A: container A tạo session mới trong RAM của nó -> trả về `history_length = 1`.
  + Request 2 rơi vào container B: container B chưa từng gặp user này -> trả về `history_length = 1`.
  + Request 3 rơi vào container C: container C cũng chưa từng gặp user này -> trả về `history_length = 1`.
  + Request 4 lại rơi vào container A: container A tìm thấy user từ request 1 -> trả về `history_length = 2`.
  Kết quả là người dùng sẽ thấy `history_length` nhảy bất định (1, 1, 1, 2, 2, 3...), ngữ cảnh hội thoại bị đứt gãy phụ thuộc vào việc request rơi trúng container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi gặp phải:**
  Khi chạy kiểm thử `pytest tests/test_cp5.py` để xác thực dịch vụ cloud trên Railway, bài test `test_ask_hoat_dong_voi_key_that` trả về lỗi:
  `AssertionError: {"detail":"invalid or missing API key"} - assert 401 == 200`.
- **Cách tìm ra nguyên nhân:**
  Tôi kiểm tra mã nguồn xác thực `app/auth.py` và nhận thấy endpoint `/ask` so sánh header `X-API-Key` với giá trị của biến môi trường `AGENT_API_KEY`. Khi deploy lên Railway, trên tab *Variables* của Web Service, tôi đã sinh một API key mới ngẫu nhiên an toàn cho production. Tuy nhiên trong file `.env` ở môi trường kiểm thử local, biến `DEPLOY_API_KEY` vẫn giữ giá trị cũ (hoặc chuỗi placeholder), dẫn đến việc pytest gửi API key không khớp với key được cấu hình trên cloud.
- **Cách khắc phục:**
  1. Mở trang quản trị của dự án trên Railway Dashboard, vào phần *Variables* của service và copy chính xác chuỗi `AGENT_API_KEY` production.
  2. Cập nhật giá trị đó vào biến `DEPLOY_API_KEY` trong file `.env` ở máy local.
  3. Dùng lệnh `curl -X POST https://<railway-domain>/ask -H "X-API-Key: <key>" -H "Content-Type: application/json" -d '{"question":"test"}'` để xác nhận server trả về status code 200 OK cùng câu trả lời từ agent. Sau đó chạy lại `pytest tests/test_cp5.py`, bài test đã vượt qua hoàn toàn.
