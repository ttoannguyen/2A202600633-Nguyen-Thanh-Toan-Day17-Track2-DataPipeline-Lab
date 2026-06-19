# Reflection — Day 17 (≤ 200 words)

Answer briefly, in your own words. This is graded on reasoning, not length.

1. **The flywheel.** Day 13 emitted agent traces; today you turned them into an
   eval set and DPO pairs that Day 22 will train on. Which step in
   `traces → Bronze → datasets` would break most silently in production if you
   got it wrong — and how would you detect it?

2. **Decontamination.** Your run dropped 2 of 3 preference pairs because their
   prompts were in the eval set. What concretely goes wrong if you *skip* this
   step and train on those pairs? How would the lie show up in your metrics?

3. **Point-in-time.** The naive join leaked a future `lifetime_spend` into the
   training row. Describe one feature in a system you know that would be
   dangerous to join without an `ASOF`/point-in-time guard.

4. **Graph vs vector.** From `kg_demo.py`, name one question the knowledge graph
   answers well that flat chunk retrieval (`embed.py`) would struggle with, and
   one where the graph is overkill.

## Câu trả lời

**1. Flywheel.** Hỏng âm thầm nhất là gán nhãn trong `flatten()`: `status`/`split`
của span gốc. Lượt `error` bị gán nhầm `ok` → câu trả lời lỗi lọt vào `reference`
eval *và* thành `chosen` DPO, đầu độc cả thước đo lẫn dữ liệu train, pipeline vẫn
"xanh". Phát hiện bằng invariant: mỗi trace 1 span `depth=0`; eval/pair count khác
0; không `chosen` nào mang `error.type`.

**2. Decontamination.** Bỏ thì 2/3 pair (prompt "Can I return a widget…" và "What
warranty…") vừa ở eval vừa được train → model học thuộc đáp án rồi mình chấm bằng
chính prompt đó. Lời nói dối lộ ở **khoảng cách offline-vs-production**: điểm
offline tăng giả nhưng holdout sạch không nhúc nhích.

**3. Point-in-time.** Nguy hiểm nếu thiếu ASOF: `lifetime_spend` hay số chargeback
90 ngày của user — lúc score phải dùng giá trị *tại thời điểm event*, không phải
hiện tại (ASOF u100 = 50/120, naive rò rỉ 300 từ tương lai).

**4. Graph vs vector.** KG thắng câu đa-hop "widget ship từ đâu?"
(widget→accessory→hanoi, 2 fact ở 2 chunk — vector không bắc cầu). KG thừa cho
tra cứu 1-fact "gadget bảo hành bao lâu?" — một chunk trả lời được, vector rẻ hơn.
