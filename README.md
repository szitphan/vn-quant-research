# VN Quant Research

Hệ thống nghiên cứu định lượng cho thị trường Việt Nam (phái sinh VN30F và cổ phiếu), xây từng bước. Mỗi vòng nghiên cứu gói gọn trong **một notebook**. Backtest framework **tự viết**, chỉ dùng numpy/pandas.

## Cấu trúc

```
vn-quant-research/
├── notebooks/
│   └── 01_vn30f_alpha_research.ipynb   # toàn bộ logic: data → feature → rule → backtest → train/val → OOS
├── data/
│   ├── raw/      # snapshot từ API vendor (VNDirect, SSI, Entrade), giữ nguyên cả lỗi
│   └── clean/    # chuỗi VN30F1M liên tục đã reconcile + điều chỉnh roll (sinh lại được từ raw)
├── research_log.md   # nhật ký mọi lần thử, kể cả thất bại
├── oos_ledger.json   # (chưa có) sinh ra khi chạy OOS lần đầu, là bằng chứng OOS chỉ dùng 1 lần
└── requirements.txt
```

## Chạy

```bash
cd ~/Projects/vn-quant-research
/opt/anaconda3/bin/jupyter lab notebooks/01_vn30f_alpha_research.ipynb
# hoặc mở bằng VS Code, chọn interpreter /opt/anaconda3/bin/python
```

- Mặc định đọc snapshot trong `data/raw/` để tái lập đúng kết quả. Muốn tải lại từ API thì đặt `REFRESH_RAW = True`, nhưng vendor có thể đã sửa dữ liệu quá khứ.
- Chạy toàn bộ notebook mất khoảng 1 phút.

## Quy tắc nghiên cứu

| Giai đoạn | Thời gian | Được làm gì |
|---|---|---|
| **Train** | 08/2017 – 12/2021 | Mọi thứ: khám phá, event study, chọn logic/feature. Nhưng **ít tham số**. |
| **Validation** | 2022 – 2023 | Chọn hyperparameter từ **lưới nhỏ khai báo trước**, chọn vùng ổn định (plateau) chứ không chọn đỉnh. Kiểm tra độ ổn định. |
| **Test / OOS** | 2024 – 2025 | **Khoá.** Chỉ chạy một lần mỗi alpha qua `run_oos_once()`. Kết quả ghi vào `oos_ledger.json`. Chạy lại với config khác sẽ bị từ chối. |
| *(ngoài phạm vi)* | 2026 → | Không tải. Để dành làm forward test / paper trading. |

Code đảm bảo kỷ luật này: `period(df, "test")` và `report(..., periods=("test",))` sẽ raise `OOSLocked`. Mỗi alpha có `assert_causal()` để chứng minh không có lookahead.

## Roadmap

- [x] **Bước 0 — Data sạch.** Đối chiếu 3 vendor bằng bỏ phiếu đa số. Xử lý ngày thiếu, dữ liệu nửa phiên, nhầm mã, bar trùng ngày. Lịch đáo hạn, chuỗi liên tục điều chỉnh roll, giá thanh toán cuối cùng = VN30 close. Bộ assert sanity.
- [x] **Bước 1 — Backtest framework tự viết.** Khớp lệnh ở open phiên sau, mark-to-market theo ngày, phí HNX/VSDC/thuế/trượt giá thật. Tự tất toán khi đáo hạn. Ràng buộc cổ phiếu (không short, T+2.5, trần/sàn) đã có sẵn. 6 unit test.
- [x] **Bước 2 — Alpha 1: breakout momentum.** Pipeline đầy đủ. **Không qua validation** (5 lệnh/2 năm, Sharpe val 0.48), nên không đưa vào OOS.
- [x] **Bước 3 — Alpha 2: basis carry (HĐTL hội tụ về VN30).** Qua validation: Sharpe val 0.84, lãi 7/7 năm, chịu được trượt giá ×3.
- [ ] **Bước 4 — Chạy OOS một lần cho Alpha 2.** Đặt `RUN_OOS = True` ở Section 6. Quyết định của bạn, không ai chạy thay.
- [ ] **Bước 5 — Forward test 2026.** Paper trading / tải dữ liệu 2026 khi kết thúc năm làm OOS thứ hai. Đo trượt giá thực tế so với giả định 0,3 điểm.
- [ ] **Bước 6 — Alpha cổ phiếu.** Cần dữ liệu không survivorship bias (lịch sử thành phần VN30/VN100), giá điều chỉnh cổ tức/chia tách. Long-only, T+2.5, trần/sàn ±7%, thanh khoản tối thiểu.
- [ ] **Bước 7 — Gộp danh mục.** Kết hợp nhiều alpha ít tương quan (alpha1 và alpha2 tương quan khoảng 0), sizing theo volatility, giới hạn rủi ro.
- [ ] **Bước 8 — Intraday VN30F** (ATO/ATC, phiên chiều) và thực thi qua API công ty chứng khoán.

## Giả định chi phí VN30F (mỗi chiều, mỗi hợp đồng)

Phí HNX 2.700 đ + phí bù trừ VSDC 2.550 đ + phí môi giới 3.000 đ (giả định) + thuế TNCN 0,1% × giá × 100.000 × 18% / 2 + trượt giá 0,3 điểm. Tính thêm 2.550 đ mỗi đêm giữ vị thế (bảo thủ). Tổng khứ hồi ≈ 1 điểm. Kiểm tra lại với biểu phí của công ty chứng khoán bạn dùng: tham số nằm trong `VN30F = InstrumentSpec(...)`.
