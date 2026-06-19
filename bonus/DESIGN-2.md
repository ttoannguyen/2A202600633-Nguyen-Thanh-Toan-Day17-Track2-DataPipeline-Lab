# Bonus — Thiết kế: Pipeline sinh đề thi từ đề thật (ảnh/PDF) + chatbot sinh đề

## Bài toán thực tế

Một nền tảng ed-tech Việt Nam muốn xây ngân hàng đề và một **chatbot sinh đề**:
giáo viên upload **đề thi thật** ở mọi định dạng — ảnh chụp đề giấy, PDF scan,
PDF số, file Word — hệ thống bóc tách thành **câu hỏi có cấu trúc** (đề bài, đáp
án, lời giải, môn, chương, độ khó), rồi chatbot dùng kho đó để **sinh đề mới** bám
chuẩn kiến thức và độ khó mong muốn. Giáo viên duyệt/sửa/loại từng câu sinh ra —
chính phản hồi đó là **vòng lặp dữ liệu** để fine-tune mô hình sinh đề tốt dần.

Đây là mở rộng trực tiếp của Lab 17: unstructured (ảnh/PDF) → ingest → **gate chất
lượng** → Silver có cấu trúc (dedup câu gần-trùng) → Gold ngân hàng đề; trace của
chatbot → eval set + cặp DPO `(prompt, chosen, rejected)` + decontamination chống
**lọt đề**; độ khó câu hỏi là feature **point-in-time**.

## Ràng buộc (constraint set)

- **OCR tiếng Việt + công thức**: đề Toán/Lý/Hóa đầy công thức → cần OCR ra LaTeX,
  không chỉ text; dấu tiếng Việt và ký hiệu (≤, →, phân số) dễ sai trên bản scan mờ.
- **Chống lọt đề (decontamination)** là *yêu cầu sống còn*: câu sinh cho học sinh
  luyện **không được trùng** câu trong đề thi thật sắp dùng — lọt đề là sự cố
  nghiêm trọng, không chỉ "eval nói dối" như lab.
- **Point-in-time độ khó**: độ khó (IRT) của câu tính từ kết quả làm bài của học
  sinh; train mô hình sinh đề phải dùng độ khó *đã biết tại thời điểm đó*, không
  được nhét kết quả tương lai.
- **Bản quyền**: nhiều đề có bản quyền (NXB, sở GD) → phải gắn nguồn, tách kho
  "tham khảo nội bộ" khỏi kho "phát hành cho học sinh".
- **Ngân sách**: ưu tiên OCR/LLM tự host hoặc API rẻ; chi phí trên mỗi trang scan
  phải kiểm soát được khi quét hàng chục nghìn trang.

## Kiến trúc đề xuất

```
  ảnh / PDF scan / PDF số / Word
        │ upload
        ▼
  ┌────────────────────────────────────────────────────────────────┐
  │  Bronze: file gốc + OCR raw (LaTeX+text) verbatim, append-only   │
  │     │  (giữ nguyên để dựng lại; gắn nguồn + bản quyền)           │
  │     ▼                                                            │
  │  GATE chất lượng (Pandera-style):                               │
  │     đủ đề bài? có đáp án? công thức parse được? → bad→quarantine│
  │     ▼ (clean)                                                   │
  │  Silver: câu hỏi có cấu trúc {đề, đáp án, lời giải, môn,         │
  │     chương, độ khó}  ── dedup câu gần-trùng (MinHash/13-gram)    │
  │     ├─► Gold: ngân hàng đề (truy vấn theo môn/chuẩn/độ khó)      │
  │     ├─► KG: graph chuẩn kiến thức (môn→chương→kỹ năng→câu)       │
  │     └─► difficulty_history (ASOF, point-in-time từ kết quả HS)   │
  └────────────────────────────────────────────────────────────────┘
        ▲                                   │
        │ chatbot sinh đề (trace gen_ai.*)  │ retrieve (KG + vector)
        │                                   ▼
   giáo viên DUYỆT/SỬA/LOẠI ──► DPO pairs (chosen=duyệt, rejected=loại)
        │                          │
        │                          ▼  decontaminate ⟂ đề thi sắp dùng
        └──────────────────► weekly fine-tune mô hình sinh đề (Day 22)
```

## Các câu hỏi mở + tradeoff

**1. OCR: API quản lý (Google Vision/Mathpix) hay model tự host?**
*Mathpix (giỏi LaTeX, trả phí/trang) vs PaddleOCR/Surya tự host.* Chọn **lai**:
tự host cho text tiếng Việt thường, gọi Mathpix **chỉ cho vùng có công thức** (phát
hiện bằng layout detector). Lý do: công thức sai một ký hiệu là hỏng cả câu, đáng
trả tiền; còn text thường thì tự host rẻ ở quy mô lớn. Đổi lại pipeline 2 nhánh
phức tạp hơn.

**2. Dedup câu hỏi: exact hay near-duplicate?**
*Exact-hash vs MinHash/embedding.* Chọn **near-duplicate (MinHash + ngưỡng)**. Lý
do: cùng một câu Toán chỉ đổi số ("x=3" vs "x=5") hoặc đổi thứ tự đáp án A/B/C/D là
trùng bản chất — exact-hash để lọt, ngân hàng đề phình rác. Đổi lại nguy cơ gộp
nhầm 2 câu khác nhau; chấp nhận bằng ngưỡng thận trọng + review ngẫu nhiên.

**3. Sinh đề: RAG từ kho đề có sẵn hay sinh thuần từ LLM?**
*RAG (bám câu thật) vs free-form.* Chọn **RAG bắt buộc**, mọi câu sinh phải neo vào
≥1 câu mẫu + node chuẩn kiến thức trong KG. Lý do: LLM sinh thuần hay "bịa" kiến
thức sai chương trình GDPT — neo vào kho thật giảm ảo giác và giữ đúng độ khó. Đổi
lại sáng tạo bị giới hạn; chấp nhận vì đúng chương trình quan trọng hơn lạ.

**4. Decontamination chống lọt đề ở mức nào?**
*Exact (như lab) vs 13-gram + embedding-similarity.* Chọn **cả hai tầng**: 13-gram
chặn copy gần nguyên văn, embedding chặn câu *cùng ý đổi lời*. Lý do: lọt đề là sự
cố pháp lý/uy tín, không chỉ sai số liệu — phải chặn cả bản viết lại. Đổi lại false
positive (loại nhầm câu hợp lệ); chấp nhận vì sai về phía an toàn.

**5. Độ khó câu hỏi: ASOF hay giá trị mới nhất?**
*ASOF point-in-time vs latest.* Bắt buộc **ASOF**. Lý do: độ khó cập nhật sau mỗi
kỳ thi; nếu train mô hình sinh "đề khó 7/10" bằng độ khó *sau này* thì model học rò
rỉ tương lai, sinh đề lệch khi triển khai. Đổi lại cần lưu `valid_from` cho mọi bản
độ khó.

**6. Lịch fine-tune: theo lượt hay theo tuần có người duyệt?**
*Online vs weekly có human-in-the-loop.* Chọn **weekly + giáo viên duyệt** trước
khi vào tập train. Lý do: dữ liệu giáo dục nhạy cảm, một câu sai kiến thức lan ra
hàng nghìn học sinh; cần chốt chặn người. Đổi lại cải thiện chậm hơn.

## Phương án bị loại (rejected alternative)

**Bỏ pipeline có cấu trúc, nhét thẳng PDF vào một LLM đa phương tiện rồi để nó tự
sinh đề.** Bị loại vì: (a) không truy vết được nguồn/bản quyền từng câu — rủi ro
pháp lý; (b) không có **gate chất lượng** nên câu OCR hỏng (mất đáp án, sai công
thức) lọt thẳng vào đề học sinh; (c) không kiểm soát được **lọt đề** vì không có
bước decontamination so với đề sắp thi; (d) chi phí token cho PDF thô lặp lại mỗi
lần truy vấn rất đắt. Cấu trúc-hóa một lần rồi tái dùng rẻ và an toàn hơn nhiều.

## Quyết định thể hiện hiểu data Việt / chi phí / failure-semantics

Điểm hội tụ là **OCR công thức tiếng Việt + decontamination chống lọt đề**. Đề thi
Việt trộn dấu tiếng Việt và ký hiệu toán trên bản scan chất lượng kém, nên ép tất
cả qua một OCR text sẽ phá công thức — vì vậy tách nhánh và trả tiền Mathpix *chỉ*
cho vùng công thức (cân bằng **chi phí vs chất lượng**). Về **failure-semantics**,
ta cố ý đặt ngưỡng lệch nhau: với ingest đề, chấp nhận bỏ sót vài câu OCR khó (mất
một câu chỉ làm kho nhỏ đi); nhưng với decontamination thì sai về phía **chặn quá
tay** — thà loại nhầm vài câu hợp lệ còn hơn để **lọt một câu trùng đề thi thật**,
vì hệ quả của lọt đề là không thể đảo ngược.
