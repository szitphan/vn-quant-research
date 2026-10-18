# Research log

Ghi **mọi** lần thử, kể cả thất bại và phần khám phá ngoài notebook. Số lần thử càng nhiều thì kết quả tốt nhất càng dễ là may mắn. Log này là thứ để tự kiểm tra điều đó.

Định dạng: ngày · alpha · giai đoạn data đã nhìn · đã thử gì · kết quả · quyết định.

---

## 2026-10-16 — Khởi tạo, data

- **Data:** VN30F1M, VN30F2M, VN30 từ VNDirect, SSI, Entrade (08/2017 → 12/2025). Lỗi vendor tìm được:
  - SSI thiếu 6 phiên F1M; VNDirect thiếu 2021-04-29.
  - SSI có dữ liệu nửa phiên ngày 2019-03-13 và 2019-04-02.
  - SSI chép F1M sang F2M ngày 2022-06-20.
  - Entrade trả 2 bar cùng ngày (12 ngày, 5 ngày khác giá).
  - Sau khi bỏ phiếu đa số: 0 close conflict.
- **Sự kiện thật, không phải lỗi:**
  - HOSE không giao dịch 23–24/01/2018 trong khi HNX phái sinh vẫn chạy.
  - VN30F2005 đóng cửa trần 864 ngày đáo hạn 21/05/2020, trong khi VN30 là 815.55.
- **Roll:** chuỗi nối ngây thơ sai trung bình 0.81%, tối đa 8.83% mỗi ngày roll. Đã điều chỉnh bằng F2M.

## 2026-10-16 — Alpha 1: breakout momentum

| Giai đoạn | Thử                                                                                                                                                                    | Kết quả                                                                                          |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| train       | Event study: N ∈ {10,20,40} × H ∈ {1,2,3,5} × {up→long, down→short} = 24 ô                                                                                       | Up → long dương (t 2.3–2.9 ở H=2). Down → short âm, tức là VN hồi lại sau thủng đáy. |
| train       | Khám phá ngoài notebook: 6 định nghĩa F1 × 3 N ở H=2 (close>max close, close>max high, +vol>1.2, +CLV>0.7, breakout đầu tiên trong 5 phiên, vượt 0.5 ATR) | Volume xác nhận cho excess cao nhất (+27 đến +52 bps) nhưng ít mẫu (34–50).               |
| train       | Rule A (đối xứng), B (chỉ long), C (long + vol>1.2), N=20, H=2                                                                                                      | Sharpe 0.60 / 0.47 / 1.10. Chọn C.                                                                |
| train       | Volume tổng F1M+F2M (khử méo chu kỳ roll); ngưỡng 1.0 vs 1.2                                                                                                      | Không cải thiện. Ở ngưỡng 1.0 edge gần như biến mất, quá nhạy với tham số.           |
| val         | 9 config N × H                                                                                                                                                         | Sharpe val −0.33 → 0.48. Config chọn (N=40, H=2): 5 lệnh, t 0.67.                              |

**Quyết định:** FAIL validation (tiêu chí 1, 2, 3). Không đưa vào OOS.
**Ý tưởng tiếp:** breakout trên VN30 spot + breadth thị trường; hoặc dùng làm bộ lọc cho alpha khác.

## 2026-10-17 — Alpha 2: basis carry

| Giai đoạn | Thử                                                   | Kết quả                                                                                                                  |
| ----------- | ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| train       | Basis chia 5 nhóm × H ∈ {1,2,3}                     | Đơn điệu: Q1 (chiết khấu sâu) +34 bps phiên sau, Q5 −13 bps. HĐTL hội tụ về spot, spot không đi theo HĐTL. |
| train       | Khám phá: carry ngưỡng 5/10/20 bps/ngày           | t = 3.24 / 2.38 / 1.68                                                                                                     |
| train       | Rule A (basis θ=0.5%), B (carry θ=5 bps/ngày), H=1  | Sharpe 0.88 / 1.46. Chọn B.                                                                                               |
| val         | 9 config θ × H                                       | Cả 9 ô dương (0.11 → 0.84). Plateau chọn θ=5 bps, H=3. Sharpe val 0.84, CAGR 15.7%, max DD −16.3%.                 |
| val         | Tách chân long/short; beta; trượt giá ×1/×2/×3 | Hai chân luân phiên gánh nhau. Beta val 0.53. Trượt giá ×3 vẫn còn Sharpe 0.64.                                  |

**Quyết định:** PASS 5/5 tiêu chí, là ứng viên OOS duy nhất.
**Rủi ro ghi nhận:**

- Train và val không đồng thuận về H (train thích H=1).
- t-stat val chỉ 1.18.
- Chi phí ăn 3.4–3.9%/năm, trượt giá ×3 thì 6.9%/năm. _Bản trước ghi 6–9% do lỗi metric, xem đính chính 2026-10-18._

## 2026-10-18 — Đính chính metric chi phí

- **Lỗi:** `cost_drag_ann` chia tổng chi phí cho **vốn ban đầu**. Nhưng backtest size theo equity, số hợp đồng tăng khi equity tăng, nên chi phí ở giai đoạn sau bị phóng đại. Đã sửa thành chi phí / equity hôm trước, cùng mẫu số với `ret`.
- **Alpha 2:** train 5.6% → 3.9%, val 8.7% → 3.4%. Ở val, trượt giá ×2 / ×3: 11.9% / 13.9% → 5.1% / 6.9%.
- **Phạm vi:** chỉ dòng `cost_drag_ann` thay đổi. Sharpe, CAGR, max DD và scorecard giữ nguyên, không quyết định nào thay đổi. Chỉ chạy lại trên train + val, OOS vẫn khoá.

## OOS

_Chưa chạy._ Khi chạy, chép kết quả từ `oos_ledger.json` vào đây, kèm nhận định. Không sửa config sau khi đã xem kết quả.
