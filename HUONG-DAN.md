# Hướng dẫn làm Lab 17 — Data Pipeline Engineering (Track 2)

> Bản tóm tắt thực hành để **làm xong và nộp bài**. Đọc kèm `README.md` (chi tiết
> kiến thức) và `rubric.md` (thang điểm). Trọng số: Track-2 Daily Lab = 30%.

**Quan trọng:** lab này **đã code sẵn đầy đủ** — mọi module `pipeline/*.py` đã
implement, không có stub `TODO`. Việc của bạn **không phải viết pipeline**, mà là:
**chạy được → sinh artifact → viết reflection → nộp repo public**.

Trạng thái đã kiểm chứng: `verify.py` ra **16/16 ALL PASS**, `pytest` ra **18 passed**.

---

## 0) Môi trường (đọc trước — chỗ dễ vấp nhất)

Máy lab dùng Python 3.14 mặc định, nhưng **3.14 thiếu `venv` và bị PEP 668 chặn
`pip`**. Dùng **Python 3.12** cho đường lite:

```bash
python3.12 -m venv .venv
. .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

> Đường lite chạy được trên Python 3.10+. dbt track (tùy chọn) cần Python 3.10–3.13.

**Bẫy pytest:** chạy `pytest` trần sẽ lỗi `No module named 'pipeline'`. Phải chạy
qua module để cwd vào `sys.path`:

```bash
python -m pytest -q        # ĐÚNG (giống make test)
# pytest -q                # SAI — ImportError
```

---

## 1) Các bước chấm điểm (đường lite — 100đ lõi)

Chạy theo thứ tự, chụp màn hình 2 chỗ có ✅ để nộp.

| Bước | Lệnh | Kỳ vọng | Rubric |
|---|---|---|---|
| 1 | `python verify.py` | `16/16 — ALL PASS` ✅ chụp | +5 + bằng chứng mọi tiêu chí |
| 2 | `python main.py` | dedup 5, quarantine 3, Gold theo ngày | #1–4 (30đ) |
| 3 | `python flywheel.py` | 21 spans · eval 2 · pairs 3 raw → **1 clean (2 dropped)** | #7–12 (50đ) |
| 4 | `python -m pytest -q` | `18 passed` ✅ chụp | toàn bộ |
| 5 | `python kg_demo.py` | KG 2-hop widget→hanoi (bonus §13) | verify §13 |

Hoặc dùng `make`: `make setup` → `make verify` → `make run` → `make flywheel` →
`make test` → `make kg`.

`flywheel.py` (bước 3) **sinh ra artifact bắt buộc nộp**:
- `datasets/eval_golden.jsonl`
- `datasets/preference_pairs.jsonl`

### Pipeline làm gì (1 dòng mỗi tầng)

```
raw_orders.csv ─▶ extract → Bronze (raw, nguyên vẹn)
               ─▶ validate (Pandera gate) → tách 3 dòng xấu ra quarantine.csv
               ─▶ transform → Silver (dedup theo order_id, bỏ 5) → Gold (gộp theo ngày)

agent_traces.json ─▶ traces.flatten → Bronze spans (1 dòng/span)
                  ─▶ dataset → eval golden set + DPO pairs (prompt,chosen,rejected)
                  ─▶ decontaminate → bỏ pair có prompt trùng eval set (3→1)
                  ─▶ features → ASOF point-in-time join (chống rò rỉ tương lai)
```

---

## 2) Việc TAY duy nhất bạn phải làm

### a) Viết `submission/REFLECTION.md` (≤ 200 từ)
Đây là phần "bài tập" thật — chấm bằng **lập luận, không phải độ dài**. 4 câu hỏi:
1. Bước nào trong `traces → Bronze → datasets` hỏng **âm thầm** nhất nếu sai, và
   phát hiện thế nào?
2. Bỏ **decontamination** thì hỏng gì cụ thể? Lời nói dối lộ ra ở metric nào?
3. Một feature trong hệ thống bạn biết mà join **không** có guard ASOF sẽ nguy hiểm?
4. Một câu hỏi KG trả lời tốt mà vector retrieval đuối — và một câu KG là thừa?

Bám số liệu thật của bạn (vd: 2/3 pair bị loại do prompt nằm trong eval set).

### b) Nộp bài
1. Push lên repo **public** tên `<username>/Day17-Track2-DataPipeline-Lab`.
2. **Bẫy quan trọng:** `datasets/` nằm trong `.gitignore` nhưng rubric **bắt nộp**
   2 file jsonl. Phải force-add:
   ```bash
   git add -f datasets/eval_golden.jsonl datasets/preference_pairs.jsonl
   ```
3. Kèm trong repo/nộp kèm: screenshot `verify.py` ALL PASS + `pytest` 18 passed +
   2 file jsonl + `submission/REFLECTION.md`.
4. Dán **URL GitHub public** vào ô LMS Ngày 17 (không PR). **Giữ public tới khi có
   điểm** — private = 0.

---

## 3) dbt track (tùy chọn, #6 = 7đ, cần Python ≤ 3.13)

Bỏ qua vẫn được **93/100** lõi. Nếu làm:

```bash
python3.12 -m venv .venv-dbt && . .venv-dbt/bin/activate
pip install -r requirements-dbt.txt
cd dbt_project && DBT_PROFILES_DIR=. dbt build      # kỳ vọng PASS=11
```

---

## 4) Bonus (+20, tùy chọn — không ảnh hưởng điểm lõi)

**Không phải task cố định** — brainstorm **1 bài toán data-pipeline thật** thành
thiết kế. Tạo `bonus/DESIGN.md` (≥ 600 từ):
- 4–6 câu hỏi mở, mỗi câu nêu **1 tradeoff rõ** (X vs Y, vì sao chọn X) — 8đ
- ≥ 1 phương án **bị loại** + lý do — 3đ
- 1 sơ đồ kiến trúc (ASCII/ảnh) — 2đ
- ≥ 1 quyết định thể hiện hiểu về **data tiếng Việt / chi phí / failure-semantics** — 3đ

Đọc `BONUS-CHALLENGE.md`. Chấm bằng judgment, không checklist.

---

## Checklist nộp bài

- [ ] `python verify.py` → ALL PASS (chụp)
- [ ] `python -m pytest -q` → 18 passed (chụp)
- [ ] `datasets/eval_golden.jsonl` + `preference_pairs.jsonl` đã `git add -f`
- [ ] `submission/REFLECTION.md` viết xong (≤ 200 từ)
- [ ] (tùy chọn) `bonus/DESIGN.md`
- [ ] Repo **public**, URL dán vào LMS Ngày 17
