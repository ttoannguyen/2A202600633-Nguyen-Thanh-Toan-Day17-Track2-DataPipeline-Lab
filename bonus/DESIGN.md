# Bonus — Thiết kế: Data flywheel cho agent chăm sóc khách hàng của một ví điện tử Việt Nam

## Bài toán thực tế

Một ví điện tử kiểu MoMo/ZaloPay xử lý ~5 triệu hội thoại CSKH mỗi tháng: nạp/rút
tiền lỗi, tranh chấp giao dịch, KYC, hoàn tiền. Một **agent LLM** đã trực tuyến,
trả lời ~70% lượt ở tầng 1, phần khó đẩy cho người. Mỗi lượt sinh một **trace
`gen_ai.*`** (đúng như Lab 17): cây span `invoke_agent → retrieve_policy →
call_tool(refund_lookup) → chat`.

Mục tiêu: biến chính traffic đó thành **vòng lặp dữ liệu** — eval set vàng + cặp
DPO `(prompt, chosen, rejected)` để fine-tune hằng tuần, **đồng thời** nuôi một
**feature store rủi ro** point-in-time để agent biết "user này có đang bị khóa vì
nghi gian lận không" *tại thời điểm hỏi*. Mở rộng trực tiếp T6–T8 của lab.

## Ràng buộc (constraint set)

- **Zero-PII rò rỉ vào train**: hội thoại chứa CCCD, số thẻ, số dư — luật
  (Nghị định 13/2023 về bảo vệ dữ liệu cá nhân) bắt buộc khử nhận dạng *trước* khi
  một byte vào dataset train.
- **Ngân sách**: < 8 triệu VND/tháng cho toàn bộ tầng dữ liệu (đội nhỏ, không
  Snowflake). Ưu tiên OSS tự host.
- **Độ trễ feature lúc serve**: p99 < 150ms — agent chờ feature rủi ro mới trả lời.
- **Tính đúng thời điểm**: nhãn "gian lận" và "số dư" thay đổi liên tục; train phải
  point-in-time, sai một lần là model học rò rỉ tương lai.
- **Tiếng Việt**: prompt có dấu / không dấu, teen-code ("ck" = chuyển khoản,
  "stk" = số tài khoản), trộn Anh-Việt. Chuẩn hóa và decontamination phải xử lý được.

## Kiến trúc đề xuất

```
                         ┌─────────────────────────────────────────────┐
  Agent (Day 13)         │              LAKEHOUSE (DuckDB→Iceberg)       │
  traces gen_ai.* ──────►│  Bronze: spans verbatim (append-only, raw)    │
        │  (OTLP)        │     │ PII scrubber (regex + NER tiếng Việt)    │
        ▼                │     ▼                                          │
   Redpanda topic        │  Silver: span phẳng, đã khử PII, đã chuẩn hóa  │
   key = user_id ───────►│     │                                          │
   (idempotent           │     ├─► eval_golden  (split='eval', ok)        │
    consumer, dedup       │     ├─► DPO pairs ──► decontaminate (13-gram) │
    theo span_id)        │     └─► risk_features (ASOF, point-in-time)    │
                         │              │                                 │
                         │              ▼                                 │
                         │  Feature store online (Redis)  ──► agent serve │
                         └─────────────────────────────────────────────┘
                                        │ weekly
                                        ▼
                              Day-22 SFT/DPO finetune
```

Lõi giống lab: Bronze append-only → Silver typed/deduped → dataset + feature
point-in-time. Khác: thêm Redpanda thật (partition theo `user_id`, consumer
idempotent), tách online store (Redis) cho serve, và lớp khử PII đứng giữa Bronze
và mọi thứ phía sau.

## Các câu hỏi mở + tradeoff

**1. Khử PII trước hay sau khi land Bronze?**
*Trước (scrub-on-write) vs sau (giữ raw, scrub khi đọc).* Chọn **giữ raw ở Bronze +
scrub khi tạo Silver/dataset**. Lý do: Bronze rule #1 của lab — nguồn-sự-thật phải
dựng lại được; nếu một quy tắc scrub sai, ta vá lại từ raw thay vì mất dữ liệu vĩnh
viễn. Đổi lại Bronze thành dữ liệu nhạy cảm → mã hóa at-rest + ACL chặt, TTL 30 ngày.

**2. Exact-match hay fuzzy decontamination?**
*Exact (như lab) vs 13-gram/embedding.* Chọn **13-gram**. Lý do: tiếng Việt cùng
một câu hỏi viết "Sao tôi ck bị treo?" vs "Tại sao chuyển khoản của tôi bị treo?"
— exact-match cho lọt, eval nói dối. Đổi lại tốn CPU; chấp nhận vì rò rỉ eval đắt
hơn nhiều so với vài giây băm n-gram.

**3. Feature rủi ro: ASOF point-in-time hay "giá trị mới nhất"?**
*ASOF vs latest.* Bắt buộc **ASOF** (bài học §11). Nhãn gian lận được gán *sau* khi
điều tra — join "mới nhất" sẽ nhét kết quả điều tra tương lai vào dòng train, model
"thiên tài" offline rồi chết khi serve. Đổi lại query ASOF phức tạp hơn và cần cột
`valid_from` chuẩn cho mọi feature.

**4. DuckDB đơn-node có đủ ở 5 triệu lượt/tháng không?**
*DuckDB embedded vs Spark/warehouse đám mây.* Chọn **DuckDB → Parquet/Iceberg trên
S3-compatible (MinIO tự host)**. Lý do: 5M lượt/tháng ≈ vài chục GB span/tháng,
thừa sức cho DuckDB out-of-core; trần ngân sách 8 triệu VND loại Snowflake. Đổi lại
mất orchestration phân tán — chấp nhận, batch hằng đêm là đủ, chưa cần real-time
train.

**5. Idempotency lúc replay stream?**
*Dedup theo `span_id` vs theo offset.* Chọn **theo `span_id`** (giống consumer
idempotent của lab). Lý do: Redpanda at-least-once + retry của agent → cùng span
gửi 2 lần; offset không bền khi đổi consumer group. Đổi lại phải giữ một set/bloom
filter span_id đã thấy.

**6. Lịch fine-tune: hằng tuần hay liên tục?**
*Weekly batch vs online.* Chọn **weekly**. Lý do: cặp DPO cần đủ khối lượng và cần
con người spot-check trước khi train (an toàn tài chính); online learning dễ
poisoning từ chính lỗi agent. Đổi lại agent cải thiện chậm hơn 1 tuần.

## Phương án bị loại (rejected alternative)

**Pinecone/vector DB quản lý cho toàn bộ retrieval + dataset.** Bị loại vì: (a) chi
phí trả theo vector vượt trần 8 triệu VND ở quy mô này; (b) lab đã chứng minh
vector **không** trả lời được câu đa-hop ("user này có giao dịch tranh chấp nào
liên quan ví bị khóa không?" — cần bắc cầu nhiều fact), nên một **knowledge graph
quan hệ giao dịch** hợp hơn cho phần đó; (c) khóa nhà cung cấp (vendor lock-in)
trái với yêu cầu OSS tự host. Giữ vector chỉ cho FAQ 1-fact, KG cho quan hệ.

## Một quyết định thể hiện hiểu dữ liệu Việt / chi phí / failure-semantics

**Chuẩn hóa tiếng Việt + decontamination (câu hỏi 2 và 5)** là nơi ba thứ gặp nhau.
Người Việt nhắn không dấu và teen-code ("ck", "stk", "ko" = không) cực nhiều, nên
exact-match decontamination của lab sẽ để lọt prompt eval viết lại → train trên cái
mình chấm → **eval nói dối âm thầm**. Giải pháp: bước chuẩn hóa thêm bảng ánh xạ
teen-code + bỏ dấu trước khi băm 13-gram, và về **failure-semantics**, chọn
at-least-once + dedup theo `span_id` thay vì exactly-once — vì mất một trace CSKH
chỉ làm dataset nhỏ đi (chấp nhận được), còn xử lý *trùng* một giao dịch hoàn tiền
là rủi ro tài chính thật. Tức là ta cố ý đặt ngữ nghĩa lỗi khác nhau cho dữ liệu
train (ưu tiên đừng mất) và cho hành động tiền bạc (ưu tiên đừng trùng).
