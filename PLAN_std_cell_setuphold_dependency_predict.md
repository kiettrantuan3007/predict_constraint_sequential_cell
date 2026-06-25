# PLAN: GNN cho Standard Cell Characterization (NLDM/NLPM + LVF)

> Mục tiêu: Dùng GNN để dự đoán full characterization table (delay/transition/power/LVF sigma)
> từ một số ít điểm SPICE simulation thật (anchor points), giảm thời gian characterization.
> Giai đoạn mở rộng tiếp theo (sau khi MVP hoàn thành): dự đoán setup/hold dependency cho sequential cells.

---

## 1. Phạm vi dự án

### 1.1 Input
- Post-layout netlist (SPICE, đã có parasitic RC)
- HSPICE models (PDK, multi-corner)
- Characterization settings: PVT corners, slew/load range, voltage, temperature, index template
- **Giai đoạn đầu (MVP)**: dùng **45nm PDK open-source** (FreePDK45/NanGate) và **ngspice** (open-source SPICE simulator) thay cho HSPICE/PDK thương mại — chi tiết xem mục 6.6.

### 1.2 Output
- File `.lib` (Liberty format) chuẩn, chứa:
  - NLDM (Non-Linear Delay Model): delay, output transition theo (input slew × output load)
  - NLPM (Non-Linear Power Model): internal power, leakage
  - LVF (Liberty Variation Format): sigma table cho variation-aware STA — **ưu tiên cao nhất** vì tốn thời gian characterization nhiều nhất (cần nhiều run Monte Carlo / quasi-MC per index point)

### 1.3 Ngoài phạm vi (giai đoạn sau)
- CCS (Composite Current Source), Noise models
- Setup/hold dependency prediction (sequential cells) — dự án thử nghiệm kế tiếp sau MVP

### 1.4 Ý tưởng cốt lõi
- GNN map netlist → đồ thị (node = transistor, feature gồm W/L; node = net, feature gồm tổng R/C)
- Chạy SPICE thật trên lưới thưa làm **anchor points** — thay vì fixed 2×2 hoặc 3×3, nên nghiên cứu **adaptive index selection**: chọn động vị trí anchor dựa trên đặc tính cục bộ của bảng (vd. vùng có gradient cao, non-linearity, hoặc khả năng non-monotonicity) thay vì lưới đều cố định. Lý do: với cell có đặc tính phi tuyến mạnh hoặc khu vực bảng có nguy cơ non-monotonic (vd. vùng slew rất nhỏ/load rất lớn), lưới đều 2×2 có thể bỏ sót hoàn toàn các vùng critical này — xem mục 7 Open Items về chiến lược chọn anchor.
- GNN dự đoán phần còn lại của bảng đầy đủ (7×7 hoặc 10×10) — graph-conditioned interpolation/extrapolation, khác hẳn linear/cubic interpolation truyền thống vì học được từ cấu trúc mạch.

---

## 2. Kiến trúc & Pipeline kỹ thuật

### 2.1 Graph construction (giai đoạn quan trọng nhất — quyết định chất lượng học)
- **Node types** (heterogeneous graph):
  - Transistor node: W, L, Vth flavor, finger count, model corner tag, loại (PMOS/NMOS)
  - Net node: **Elmore delay** (tổng RC path từ driver đến mỗi sink — phản ánh chính xác hơn propagation delay so với lumped R/C đơn thuần), **Ceff** (effective capacitance — tương đương tải mà driver "nhìn thấy", quan trọng cho accuracy của NLDM delay/slew), fanout; lumped R và C tổng vẫn giữ như feature phụ trợ nếu cần
- **Edge types**: transistor–net theo terminal (gate/drain/source/bulk) — cần phân biệt loại kết nối
- **Global/context features** (broadcast vào graph hoặc concat vào output head):
  - PVT corner, input slew, output load, related output pin (cho multi-input cell), arc (rise/fall)
- **Target**: từ giá trị SPICE thật tại anchor points → dự đoán toàn bộ bảng

### 2.2 Model
- Encoder: Heterogeneous GNN (GraphSAGE / GAT / RGCN) → cell-level + arc-level embedding
- **Anchor injection**: kết hợp embedding với giá trị SPICE thật tại anchor points (không phải pure extrapolation mù) — đây là điểm khác biệt cốt lõi so với "GNN dự đoán từ đầu"
- Output heads riêng biệt:
  - NLDM head (delay, transition) — regression, nên dùng relative/log-scale loss
  - NLPM head (power) — regression
  - LVF head (sigma) — khó hơn, có thể cần model phân phối (predict variance/NLL) thay vì điểm đơn; data đắt hơn vì cần nhiều MC run

### 2.3 Sanity / Integration
- Liberty `.lib` writer: đúng cú pháp, đơn vị, index template
- Sanity checks: monotonicity (delay tăng theo load/slew), non-negativity của LVF sigma
- Fallback: nếu uncertainty cao → tự động chạy thêm SPICE point thay vì tin tưởng tuyệt đối

---

## 3. Roadmap theo giai đoạn

| Phase | Nội dung | Output | Trạng thái |
|---|---|---|---|
| 0 | Data pipeline: netlist → graph, RC extraction | Graph dataset format | TODO |
| 1 | NLDM/NLPM model (đơn giản hơn LVF) | Baseline GNN, validate trên vài cell | TODO |
| 2 | Mở rộng full standard cell library, OOD test | Model generalize tốt trên topology chưa thấy | TODO |
| 3 | LVF head/model | Variance/sigma prediction | TODO |
| 4 | `.lib` writer + sanity check + fallback mechanism | End-to-end pipeline | TODO |
| 5 (sau MVP) | Setup/hold **interdependent** curve prediction (sequential cells) — xem mục 5.7, 5.8 | Mở rộng sang flip-flop/latch với full curve, không phải scalar đơn lẻ | Nghiên cứu thử nghiệm |
| 6 (khi có commercial tool) | CCS timing + CCS-Noise — xem mục 5.8 | Cần stack thương mại (SiliconSmart/Liberate) để có ground-truth | Dài hạn |

---

## 4. Tại sao Setup/Hold (Phase 5) đặc biệt tốn kém — motivation

- Setup/hold là **constraint**, không đo được bằng 1 phép đo khoảng cách đơn lẻ như delay — cần nhiều simulation, dịch dần data edge về phía clock edge cho đến khi cell fail (binary search / bisection trong thực tế).
- Với cell có nhiều constraint pin: cần tối thiểu **4 lần optimization riêng** (setup rising, setup falling, hold rising, hold falling) — nhân thêm với số PVT corner → bùng nổ số lượng run.
- Setup và hold **phụ thuộc lẫn nhau** (dependent setup/hold): đo độc lập từng cái sẽ cho ra timing window quá nhỏ/pessimistic so với thực tế → cần sweep 2D (setup × hold), cấu trúc bài toán tương tự NLDM 2D table (slew × load) nhưng 2 biến đều là thời gian.
- ➡️ Cùng motivation và cùng dạng bài toán (sparse anchor → full table) như NLDM/LVF hiện tại — lý do hợp lý để tái dùng kiến trúc GNN đã xây ở Phase 0–4.

---

## 5. Khảo sát các công trình liên quan (tham khảo / so sánh)

> Nhận xét tổng quan: **chưa tìm thấy paper public nào dùng GNN trực tiếp cho setup/hold table prediction hoặc NLDM/LVF table completion** — đây là điểm khá mới/novel của hướng đi này. Phần lớn công trình về setup/hold dùng Gaussian Process / active learning / bisection optimization, còn GNN trong EDA timing tập trung chủ yếu ở path-based STA, clock mesh, gate sizing.

### 5.1 Trực tiếp liên quan — Setup/Hold characterization

**[1] SetupKit: Efficient Multi-Corner Setup/Hold Time Characterization Using Bias-Enhanced Interpolation and Active Learning**
Junzhuo Zhou, Ziwen Wang, Haoxuan Xia, et al. (UCLA, Eastern Institute of Technology Ningbo, Sheffield), arXiv 2512.00044
- Phương pháp: BEIRA (bias-enhanced interpolation search từ statistical error modeling), initial search interval bằng circuit analysis, active learning dùng Gaussian Process để học correlation PVT-timing và chọn corner cần simulate nhất.
- Kết quả: 22nm industrial standard cell, 16 PVT corner → giảm 2.4× CPU time (720 → 290 ngày trên 1 core).
- **Vai trò**: Baseline mạnh nhất để so sánh tốc độ/độ chính xác khi làm Phase 5. Không dùng GNN (dùng GP), nên GNN approach của mình có thể là điểm khác biệt/đóng góp mới.
- Link: https://arxiv.org/pdf/2512.00044

**[1b] Exploiting Setup-Hold-Time Interdependence in Static Timing Analysis**
E. Salman, A. Dasdan, F. Taraporevala, K. Kucukcakar, E. G. Friedman. IEEE TCAD, vol. 26, no. 6, pp. 1114-1125, June 2007. doi: 10.1109/TCAD.2006.885834
- **Bài báo nền tảng (foundational)** trực tiếp về đúng chủ đề Phase 5 — phân tích lý thuyết về tính phụ thuộc lẫn nhau giữa setup time và hold time trong STA, là cơ sở lý thuyết cho mọi phương pháp sweep 2D (setup × hold) sau này, bao gồm cả SetupKit [1].
- **Vai trò**: Bắt buộc đọc trước khi thiết kế graph/label cho Phase 5 — giúp hiểu đúng bản chất toán học của interdependence trước khi áp GNN vào.
- *(Tìm thấy qua reference list của bài CharLib — chưa fetch trực tiếp paper, cần đọc kỹ khi bắt đầu Phase 5)*

**[2] Standard-cell characterization (F-Si wiki)** — tài liệu tham khảo nền tảng về quy trình đo setup/hold truyền thống (sweep data edge, bisection search, dependent setup/hold measurement).
- Link: https://wiki.f-si.org/index.php/Standard-cell_characterization

**[3-6] Các bằng sáng chế (patents) về setup/hold characterization** — mô tả phương pháp truyền thống (binary search, test point method) làm cơ sở so sánh baseline tốc độ:
- "Automated cell characterization system" — US Patent 5655109
- "Method and system for setup/hold characterization in sequential cells" — US Patent 7795939
- "Setup and hold time characterization device and method" — US Patent 7861200
- "System and method for setup and hold characterization in integrated circuit cells" — US Patent 6640330
- "Characterizing sequential cells using interdependent setup and hold times..." — US Patent 7774731 / 7506293 (Synopsys) — quan trọng vì nói trực tiếp về **interdependent setup/hold characterization**, gần nhất với hướng nghiên cứu Phase 5.

### 5.2 GNN cho EDA timing (không trực tiếp setup/hold nhưng kỹ thuật liên quan)

**[7] GATMesh: Clock Mesh Timing Analysis using Graph Neural Networks** — arXiv 2507.05681
- Dùng GAT (Graph Attention Network) với JK connections để predict delay và slew tại clock sink node.
- Nhấn mạnh: slew ảnh hưởng đáng kể đến setup/hold time của sequential element — củng cố lý do cần model slew chính xác trong pipeline.
- **Vai trò**: tham khảo kiến trúc GAT + node feature design cho graph timing.
- Link: https://arxiv.org/pdf/2507.05681

**[8] Graph Neural Networks for Integrated Circuit Design, Reliability, and Security: Survey and Tool** — ACM Computing Surveys, 2024/2025
- Đề cập EdgeGAT model dự đoán Path-Based Analysis (PBA) timing từ Graph-Based Analysis (GBA, rẻ hơn) — đạt nhanh hơn 24.8× so với PrimeTime PBA. Cùng dạng bài toán "predict-từ-rẻ-sang-đắt" như hướng anchor→full-table của mình.
- **Vai trò**: Survey tổng quan, hữu ích để rà soát thêm các công trình GNN-EDA khác (switching activity, parasitics prediction, gate sizing) có thể liên quan đến node/edge feature design.
- Link: https://dl.acm.org/doi/full/10.1145/3769081

**[9] Why are Graph Neural Networks Effective for EDA Problems?** — NVIDIA Research, 2022
- Lập luận GNN hiệu quả trong EDA vì implicit embed inductive bias của bài toán VLSI (khác với GNN cho social/citation network).
- **Vai trò**: cơ sở lý thuyết/motivation cho việc chọn GNN thay vì MLP/CNN thuần cho bài toán characterization.
- Link: https://research.nvidia.com/publication/2022-10_why-are-graph-neural-networks-effective-eda-problems

### 5.3 ML cho standard cell characterization (NLDM/LVF nói chung)

**[10] Machine Learning for Reliability-Aware, yet Confidential Standard Cell Characterization** — 2022
- So sánh active learning vs passive learning (random/greedy sampling) để giảm số lượng training sample/SPICE run cần thiết, áp dụng cho aging-induced degradation.
- **Vai trò**: tham khảo trực tiếp cho chiến lược **chọn anchor point thông minh** (active learning) thay vì fixed 2×2/3×3 grid — có thể cải thiện đáng kể accuracy/cost trade-off ở Phase 0-3.
- Link: https://www.researchgate.net/publication/362109680

**[11] CharLib: An Open Source Standard Cell Library Characterizer**
Mellor, M. & Stine, J. E. IEEE 67th MWSCAS, Aug 2024. doi: 10.1109/MWSCAS60917.2024.10658687
- Tool characterization open-source đã chốt dùng cho dự án (mục 6.6). Python-based, config YAML, hỗ trợ ngspice/Xyce, NLDM cho combinational + sequential cell.
- Kết quả benchmark: so sánh với commercial characterizer trên PDK open-source, sai số timing trong khoảng 1ns mean absolute error — mốc tham khảo tốt cho việc đặt kỳ vọng accuracy ở mục 6.
- Link: https://github.com/stineje/CharLib

**[12] Boosting Standard Cell Library Characterization with Machine Learning**
Z. Chen et al. Proceedings of the 30th Asia and South Pacific Design Automation Conference (ASP-DAC), March 2025, pp. 385-391.
- **Cực kỳ liên quan** — bài báo này được chính trang wiki của CharLib trích dẫn như một "third-party comparison", nghĩa là đã dùng ML để cải thiện/tăng tốc characterization, khả năng cao có overlap trực tiếp với hướng đi của dự án này (anchor point → full table).
- **Vai trò**: ⚠️ Cần đọc kỹ và đối chiếu ngay ở Phase 0-1 — đây có thể là công trình gần nhất với ý tưởng GNN-based sparse-to-full table prediction, cần xác định rõ phương pháp ML họ dùng (có phải GNN không, hay regression/GP khác) để định vị đóng góp mới của dự án này. *(chưa fetch full paper, mới biết qua trích dẫn)*

**[13] ZlibBoost: An Efficient and Flexible Open-Source Framework for Standard Cell Characterization**
ACM Transactions on Design Automation of Electronic Systems, 2024/2025. doi: 10.1145/3747182
- Framework characterization open-source mới, kiến trúc front-end/back-end tách biệt, thiết kế modular cho phép tùy biến, tích hợp mô hình machine learning, và mở rộng khả năng tương thích với nhiều simulator khác nhau.
- So với CharLib: ZlibBoost dùng chiến lược fine-grained task decomposition để tối đa hóa song song hóa — mỗi SPICE simulation task tương ứng với việc characterize đầy đủ một timing arc cụ thể dưới các transition time và load khác nhau, giảm load imbalance và tail latency so với cách tiếp cận coarser-grained của CharLib, đặc biệt với sequential cell phức tạp.
- **Vai trò**: Ứng viên hàng đầu để thay thế/bổ sung CharLib ở Phase 3-4 khi cần nhúng GNN prediction trực tiếp vào flow, vì kiến trúc vốn đã hướng tới việc tích hợp ML.
- Link: https://dl.acm.org/doi/10.1145/3747182

**[14] lctime** — T. Kramer, open-source tool suy luận logic trực tiếp từ netlist topology để characterize. Cách tiếp cận này có thể không đáng tin cậy với cell phức tạp do sự mơ hồ gây ra bởi feedback loop, custom logic style, và hành vi phụ thuộc trạng thái (state-dependent) — theo nhận xét từ ZlibBoost [13]. Ghi nhận làm tham khảo, không ưu tiên dùng cho sequential cell ở Phase 5.

### 5.4 Khoảng trống nghiên cứu (research gap) — ĐÃ CẬP NHẬT, quan trọng

> ⚠️ **Cập nhật quan trọng so với khảo sát lần đầu**: Sau khi đọc sâu hơn, mình tìm thấy **2 bài báo trùng trực tiếp** với ý tưởng cốt lõi của dự án — cần đọc kỹ trước khi code, không còn là "khoảng trống nghiên cứu trống" như nhận định ban đầu.

- **[15] Ma et al., "Fast Cell Library Characterization for DTCO Based on Graph Neural Networks" (ASP-DAC 2024)** — đây gần như là **bài báo "sinh đôi ý tưởng"** với phần NLDM/NLPM của dự án: dùng GNN, incorporate cell structure, predict delay/power/input pin capacitance qua nhiều PVT corner, đạt MAPE ≤0.95% và speedup 100x so với SPICE. **Cần đọc kỹ ngay đầu Phase 0** để xác định rõ điểm khác biệt của dự án bạn (có thể là: tập trung vào anchor-points-sparse-grid thay vì toàn bộ PVT space, và mở rộng LVF + setup/hold mà bài này chưa làm).
- **[16] Zhou et al., "LVFGen: Efficient LVF Generation Using Variational Analysis and Active Learning" (ISPD 2025)** — trực tiếp giải quyết đúng bài toán LVF mà bạn xác định là ưu tiên cao nhất, dùng active learning (không phải GNN) để giảm số MC sample cần thiết, đạt speedup 2.27–4.06× so với Quasi-MC. Cùng nhóm tác giả với SetupKit [1].
- Setup/hold: **Salman et al. 2007** [1b] vẫn là nền tảng lý thuyết quan trọng nhất, cùng SetupKit [1] là phương pháp hiện đại nhất — nhưng vẫn **chưa có ai dùng GNN cho riêng bài toán setup/hold table**, nên hướng Phase 5 của bạn vẫn còn mới.
- **Kết luận cập nhật**: GNN cho NLDM/NLPM **đã có precedent rõ ràng** (Ma et al. 2024) — dự án của bạn nên định vị mình khác biệt ở đâu: (a) cách tiếp cận sparse-anchor-grid (2×2/3×3 → 7×7/10×10) thay vì train trên toàn bộ PVT space, (b) tích hợp LVF cùng NLDM/NLPM trong một pipeline thống nhất, và (c) mở rộng sang setup/hold — đây vẫn là phần chưa ai làm bằng GNN.
- **Khuyến nghị**: cần tìm thêm tại DAC/ICCAD/DATE 2024–2026, và đọc thêm các bài được trích dẫn cùng nhóm: "Heterogeneous Graph Attention Network Based Statistical Timing Library Characterization with Parasitic RC Reduction" (ASP-DAC 2024), "Graph neural network based cell library characterization method for fast DTCO" (Integration VLSI Journal, 2025), "LIBMixer: An all-MLP Architecture for Cell Library Characterization" (ASP-DAC 2025), "GraphCAD: Leveraging GNN for Accuracy Prediction Handling Crosstalk-affected Delays" (ISPD 2025) — đều là các bài rất gần chủ đề, cần rà soát thêm trước khi Phase 0.

### 5.5 Đọc chi tiết 2 bài báo trọng tâm (theo yêu cầu khảo sát sâu)

#### 5.5.1 Salman, Dasdan, Taraporevala, Küçükçakar, Friedman — "Exploiting Setup–Hold-Time Interdependence in Static Timing Analysis" (IEEE TCAD, vol. 26, no. 6, pp. 1114–1125, 2007)

**Bài toán**: STA truyền thống coi setup time và hold time là 2 hằng số độc lập. Trong thực tế, clock-to-Q delay của flip-flop phụ thuộc vào *cả setup lẫn hold slack cùng lúc* — characterize độc lập từng cái sẽ gây ra 2 loại lỗi: **optimism** (bỏ sót vi phạm thật) và **pessimism** (báo vi phạm giả, không cần thiết).

**Phương pháp — 2 phase**:
1. **Phase 1 (characterization)**: thay vì đo 1 cặp (setup, hold) độc lập, đo **nhiều cặp constraint** (multiple setup-hold pairs) cho cùng 1 cell — kết quả là một tập điểm mô tả đường cong trade-off setup-hold (tương tự Pareto front: setup nhỏ hơn đòi hỏi hold lớn hơn và ngược lại).
2. **Phase 2 (STA algorithm)**: thuật toán khai thác các cặp constraint này trong lúc chạy STA — chọn động cặp setup/hold phù hợp nhất theo slack hiện tại của từng path, thay vì dùng 1 cặp cố định.

**Kết quả** (trên mạch công nghiệp thật):
- Giảm tới **53%** số lượng vi phạm constraint
- Giảm tới **48%** worst negative slack (WNS)
- Tương đương **15%** cải thiện clock period

**Liên hệ trực tiếp đến dự án của bạn**:
- Đây chính là cơ sở lý thuyết giải thích **vì sao output của Phase 5 không nên là 1 cặp (setup, hold) đơn lẻ, mà nên là cả một "constraint curve/table"** — khớp hoàn toàn với ý tưởng "dự đoán full table từ anchor points" mà bạn đang làm cho NLDM, chỉ khác là 2 trục đều là thời gian thay vì (slew, load).
- Gợi ý thiết kế: GNN ở Phase 5 nên predict ra **toàn bộ đường cong/bề mặt trade-off setup-hold** (nhiều điểm), không phải chỉ 1 giá trị setup + 1 giá trị hold độc lập — nếu chỉ predict điểm đơn sẽ làm mất đi chính lợi ích mà bài báo này chứng minh.
- Vì bài quá cũ (2007) để fetch full-text miễn phí, mình mới đọc được abstract + kết quả qua nhiều trích dẫn thứ cấp; **khuyến nghị tìm bản PDF qua thư viện trường/IEEE Xplore** khi vào Phase 5 để đọc chi tiết thuật toán Phase 2.
- Bài liên quan cùng hướng (đáng đọc thêm): Srivastava & Roychowdhury (DAC 2007 / TCAD 2008) dùng Euler-Newton curve tracing để characterize đường cong interdependent setup/hold — có thể là kỹ thuật sinh ground-truth curve tốt hơn brute-force sweep.

#### 5.5.2 Chen, Guo, Song, Feng, et al. (nhóm Cheng Zhuo, Zhejiang University) — "Invited Paper: Boosting Standard Cell Library Characterization with Machine Learning" (ASP-DAC 2025, doi: 10.1145/3658617.3703638)

**Bối cảnh quan trọng**: Đây là bài "invited paper" tổng kết công trình của **cùng nhóm tác giả đã làm ra ZlibBoost** [13] — nghĩa là bài này về bản chất là phần "ML-enhancement" được xây trực tiếp trên framework ZlibBoost, không phải công trình độc lập.

**Nội dung chính** (theo trích dẫn trực tiếp từ bài báo): nhóm tác giả tận dụng ZlibBoost cho việc tối ưu hóa LUT index, mô hình hóa nhiễu nguồn điện động (dynamic power supply noise), và dự đoán dựa trên machine learning để tăng hiệu quả và độ chính xác trong characterization thư viện.

**3 thành phần ML chính** (suy ra từ structure bài báo và các related work được trích dẫn):
1. **LUT index optimization**: dùng ML để chọn vị trí index tối ưu trong bảng (thay vì lưới đều) — concept tương đồng với việc bạn chọn anchor point 2×2/3×3, nhưng đây là tối ưu hóa *vị trí* index chứ không phải tối ưu hóa model dự đoán phần còn lại.
2. **Dynamic power supply noise modeling** — ngoài phạm vi dự án hiện tại của bạn (mục 1.3), nhưng đáng lưu ý nếu sau này mở rộng sang sign-off chính xác cao.
3. **ML-based prediction** — đây là phần chồng lấn trực tiếp nhất với ý tưởng GNN của bạn, nhưng **chưa rõ có dùng GNN hay không** (rất có thể dùng Gaussian Process tương tự các bài khác của hệ sinh thái này, ví dụ bài liên quan "Cell Library Characterization for CCS Models Based on Gaussian Process Regression and Active Learning" của cùng nhóm tác giả).

**Hạn chế trong việc khảo sát**: bài báo này nằm sau paywall ACM Digital Library, mình mới đọc được phần reference list + 1 đoạn trích trực tiếp qua search engine, **chưa fetch được full-text/methodology chi tiết**. Cần truy cập qua ACM DL hoặc liên hệ tác giả để đọc đầy đủ phương pháp trước khi quyết định có trùng lặp với GNN approach của bạn hay không.

**Khuyến nghị hành động**: 
- Vì ZlibBoost + bài ML này đến từ cùng 1 nhóm và cùng định hướng "framework mở + tích hợp ML", đây là **đối thủ/đối tác tham khảo gần nhất** cho toàn bộ dự án (không chỉ phần LVF). Nên ưu tiên đọc full-text bài này (và cả ZlibBoost paper gốc) ngay khi có quyền truy cập ACM DL.
- Cân nhắc liên hệ trực tiếp nhóm tác giả (Zhejiang University, Cheng Zhuo) nếu cần làm rõ chi tiết phương pháp — đây là nhóm rất active trong mảng này (xuất hiện ở cả ZlibBoost, SetupKit có liên quan gián tiếp, và bài CCS+GP+active learning).

### 5.6 Kết quả đọc chi tiết — Trả lời Open Items #11, #12, #13

#### 5.6.1 [Open Item #11] Định vị lại novelty — sau khi đọc FULL-TEXT bài Ma et al. 2024

✅ Đã đọc được toàn văn (arXiv:2312.12784, bài đăng tự do, không paywall). Đây là tóm tắt chi tiết kỹ thuật và **điểm khác biệt rõ ràng với dự án của bạn**:

**Phương pháp của Ma et al.**:
- Graph: mỗi cell → directed graph với 5 loại node (IN, OUT, FET, VDD, VSS), feature gồm transistor Width, loại N/P-FET, và **toàn bộ điều kiện truy vấn (input_slew, output_load, Vth, Temperature, VDD, current/next state) được nhúng trực tiếp làm node feature** — không tách riêng "anchor point thật" và "điểm cần dự đoán".
- Model: chỉ 3-layer GCN (Kipf & Welling, không phải GAT/heterogeneous) + 2 fully-connected layer, ReLU, 128 neuron/layer — kiến trúc khá đơn giản.
- Training: 125 PVT corner (5 giá trị mỗi chiều: V_th, Temperature, V_DD) → test trên 512 PVT corner **chưa từng thấy**. Trong mỗi corner, model học **toàn bộ lưới (input_slew × output_load)** từ dữ liệu SPICE đầy đủ của 125 corner train — đây là **generalize qua PVT corner**, không phải "anchor sparse → dense table cho cùng 1 corner" như bạn định làm.
- Kết quả: MAPE ≤0.95% (delay/power/cap), speedup 100x runtime so với SPICE, đã validate ở system-level (WNS, leakage, dynamic power) trên 10 benchmark mạch thật.
- **Giới hạn tự nhận của họ**: chỉ làm **combinational cell** (33 loại), chưa làm sequential cell; "future work" của họ ghi rõ sẽ mở rộng sang sequential cells và cải thiện extrapolation/OOD corner.
- **Không có LVF/variation** — chỉ deterministic NLDM/NLPM, không có sigma/variance prediction nào.

**➡️ Định vị novelty của dự án bạn (3 điểm khác biệt rõ ràng, không trùng lặp)**:

| Khía cạnh | Ma et al. 2024 | Dự án của bạn |
|---|---|---|
| Cơ chế dự đoán | Generalize qua **PVT corner** (train 125 corner → test 512 corner mới), dùng full SPICE data ở các corner train | Generalize qua **slew×load index trong cùng 1 cell+corner**, dùng vài **anchor point SPICE thật** (2×2/3×3) làm conditioning — đây là bài toán "few-shot/sparse-to-dense" khác hẳn về bản chất thống kê |
| Anchor injection | Không có — model thuần túy học từ training distribution, không "nhìn thấy" giá trị thật nào tại thời điểm inference | Có — kết hợp embedding GNN với giá trị SPICE thật tại anchor point, giúp giảm sai số đáng kể đặc biệt với cell ngoài phân phối train |
| Loại cell | Chỉ combinational (33 loại) | Cả combinational lẫn dự kiến sequential (Phase 5) |
| LVF / variation | Không có | LVF là **ưu tiên cao nhất** của dự án bạn — khoảng trống rõ ràng họ chưa chạm tới |
| Kiến trúc GNN | GCN đơn giản (3 layer, không phân biệt edge type) | Heterogeneous GNN (phân biệt gate/drain/source/bulk edge) — biểu diễn mạch chính xác hơn |

**Kết luận**: Dự án của bạn **không trùng lặp trực tiếp** với Ma et al. 2024 — cơ chế "anchor sparse-grid + LVF + heterogeneous graph" là tổ hợp mới. Tuy nhiên, **bắt buộc phải cite và so sánh trực tiếp với bài này** trong bất kỳ báo cáo/paper nào sau này, vì đây là "first comprehensive exploration of GNN for cell library characterization" theo chính lời họ tự nhận — nếu không cite sẽ bị reviewer bắt lỗi thiếu related work nghiêm trọng.

#### 5.6.2 [Open Item #12] Bài Chen et al. (ASP-DAC 2025) — vẫn chưa fetch được full-text, nhưng có thêm chi tiết

Tìm thêm được 1 câu mô tả ngắn gọn nhất từ chính bài báo (không phải qua trích dẫn gián tiếp): họ dùng ZlibBoost cho 3 mục đích — tối ưu hóa LUT index, mô hình hóa nhiễu nguồn điện động, và **dự đoán dựa trên machine learning** để tăng hiệu quả/độ chính xác trong characterization thư viện. Bài này về bản chất là bài tổng quan giới thiệu nhanh (invited paper, 6 trang) các tính năng ML được build trên nền ZlibBoost — không phải bài nghiên cứu sâu 1 phương pháp cụ thể.

**Vẫn chưa đọc được phần phương pháp luận chi tiết (ML-based prediction dùng kỹ thuật gì cụ thể)** — bài nằm sau paywall ACM DL, không có bản preprint công khai. **Khuyến nghị**: nếu trường/tổ chức của bạn có quyền truy cập ACM Digital Library, nên tải bản PDF trực tiếp; nếu không, có thể thử liên hệ tác giả qua ResearchGate (Zhengrui Chen, Cheng Zhuo, Zhejiang University) để xin bản preprint — đây là cách phổ biến và được chấp nhận trong giới academic.

#### 5.6.3 [Open Item #13] Các bài liên quan khác — phát hiện thêm 1 manh mối quan trọng

Trong lúc tìm, phát hiện thêm bài **"Cell Library Characterization for Composite Current Source Models Based on Gaussian Process Regression and Active Learning"** (arXiv:2505.10799, cùng hệ sinh thái nhóm Zhejiang University) — phần Related Works của bài này tiết lộ một chi tiết quan trọng: mô hình deep neural network (DNN) đã được áp dụng cho standard cell characterization ở cả nominal timing và statistical timing, dùng dữ liệu từ sparse characterization để sinh ra delay model tại các sign-off corner cần thiết, nhưng bỏ qua mối quan hệ với cấu trúc mạch ở mức transistor (transistor-level cell topology).

**Đây là phát hiện quan trọng nhất trong đợt tìm này**: nó xác nhận **đã có precedent cho ý tưởng "sparse characterization → sign-off corner"** (gần với ý tưởng anchor-point của bạn), nhưng theo chính nhận xét của bài báo, các công trình đó **bỏ qua thông tin cấu trúc mạch** (cell topology) — tức là dùng MLP/DNN thuần thay vì GNN. **Đây chính là khoảng trống novelty rõ ràng nhất**: kết hợp (a) ý tưởng sparse-anchor-to-full-table đã có tiền lệ, với (b) GNN để encode cấu trúc mạch (mà Ma et al. 2024 có làm nhưng theo cơ chế khác — generalize qua corner chứ không phải sparse-to-dense trong cùng corner) — **chưa ai làm cả 2 cùng lúc**.

- 4 bài còn lại (Cheng et al. Heterogeneous GAT ASP-DAC 2024, Ma et al. Integration VLSI Journal 2025, LIBMixer ASP-DAC 2025, GraphCAD ISPD 2025) — **chưa fetch được full-text trong đợt tìm này** do giới hạn thời gian, vẫn nằm trong Open Items, cần đọc trước khi chốt kiến trúc graph chi tiết (mục 2.1).
- Gợi ý bổ sung: nên đọc thêm 2 reference được trích trong bài Ma et al. 2024 — **[12] Ebrahimipour et al. "Aadam" (ICCAD 2020)** và **[13] Klemme et al. "Cell library characterization using ML for DTCO" (ICCAD 2020)** — đây là 2 baseline MLP/linear-regression mà Ma et al. so sánh trực tiếp và vượt qua, cũng đáng đưa vào bảng so sánh accuracy của bạn sau này.

### 5.7 ⚠️ Cập nhật quan trọng — Ma et al. đã có bản mở rộng GNN-predict setup/hold (độc lập, KHÔNG phải interdependent)

Đọc full-text bài **"Late Breaking Results: Fast System Technology Co-Optimization Framework for Emerging Technology Based on GNN"** (arXiv:2404.06939, cùng nhóm tác giả Ma et al., bản mở rộng) phát hiện: nhóm này đã mở rộng cell library lên **35 loại cell gồm cả sequential**, và GNN của họ **đã dự đoán được "Minimum Setup" và "Minimum Hold"** với MAPE rất thấp (0.50%/0.27% cho LTPS, 0.27%/0.38% cho CNT).

**Điểm mấu chốt cần làm rõ ngay**: Đây là dự đoán **2 giá trị scalar độc lập** (1 setup time + 1 hold time, mỗi cái 1 con số) cho mỗi corner — **không phải** bài toán "interdependent setup-hold curve" mà Salman et al. 2007 [1b] và SetupKit [1] giải quyết (toàn bộ đường cong trade-off, nhiều cặp (setup, hold) hợp lệ). Vì vậy:
- ✅ Phase 5 dạng "GNN dự đoán bảng full interdependent setup-hold curve từ vài anchor point" **vẫn còn mới**, chưa ai làm.
- ⚠️ Nhưng phần "GNN dự đoán 1 giá trị setup/hold đơn lẻ" **đã có người làm và làm tốt** — nếu Phase 5 của bạn chỉ dừng ở mức này thì sẽ trùng lặp trực tiếp, cần nâng cấp lên bài toán interdependent curve để giữ novelty.

### 5.8 Đánh giá khả thi: GNN cho CCS-Noise — có nên thay thế setup/hold dependent không?

> Trả lời cho câu hỏi: *"GNN dự đoán noise model (CCS-Noise) có khả thi thay thế việc setup-hold dependent không?"*

**Kết luận cập nhật sau khảo sát tool: CCS-Noise bị loại khỏi phạm vi khả thi trong giai đoạn MVP open-source vì không có tool open-source nào hỗ trợ sinh ground-truth CCS (kể cả CCS timing, huống chi CCS-Noise).** Đây là constraint thực tiễn cứng, không phải vấn đề kỹ thuật GNN.

#### Tình trạng hỗ trợ CCS của các open-source tool hiện có

| Tool | CCS Timing | CCS-Noise | Ghi chú |
|---|---|---|---|
| **CharLib** (đã chọn) | ❌ Không — chỉ NLDM | ❌ Không | <cite index="10-1">CharLib bị giới hạn ở NLDM cho timing modeling và không có power modeling</cite> — xác nhận chính thức từ ZlibBoost paper |
| **ZlibBoost** | ⚠️ Chưa rõ — chưa đọc full-text | ❌ Không (không đề cập) | Kiến trúc modular có thể extend, nhưng chưa có tài liệu confirm CCS support |
| **AutoLibGen** | ✅ Có — CCS tại 65nm | ❌ Không | <cite index="2-1">AutoLibGen dùng CCS-based characterization cho very deep sub-micron technologies, test với 65nm, chỉ cho combinational cell</cite>. **Nhưng**: tool này rất cũ (~2010), không còn được maintain tích cực, không hỗ trợ sequential cell, và chưa validate với ngspice — **không nên dùng làm ground-truth tool** |
| **lctime** | ❌ Không — NLDM/NLPM | ❌ Không | Có vấn đề robustness với sequential cell (parser crash, solver instability) |
| **libretto** | ❌ Không — NLDM | ❌ Không | Tiền thân CharLib, ít hoạt động |

**Kết luận về CCS open-source**: <cite index="10-1">Các open-source tool hiện tại bị giới hạn bởi functionality, efficiency và accuracy, khiến chúng không đủ đáp ứng cho các thiết kế hiện đại</cite> — và điều này đặc biệt đúng với CCS: **không có tool open-source nào đang được maintain tích cực hỗ trợ CCS timing**, huống chi CCS-Noise (phức tạp hơn CCS timing nhiều bậc). CCS-Noise về cơ bản là proprietary territory của Synopsys (SiliconSmart) và Cadence (Liberate).

#### Điều chỉnh roadmap Phase 5a

- **Phase 5a (CCS-Noise)** bị **tạm hoãn** — không khả thi trong giai đoạn MVP open-source (không có ground-truth tool), không phải vì lý do GNN mà vì thiếu data generation pipeline. Có thể cân nhắc lại khi dự án chuyển sang stack thương mại.
- **Phase 5b (Setup/Hold interdependent)** vẫn khả thi vì CharLib + ngspice đã hỗ trợ sequential cell constraint characterization (setup/hold) ở mức NLDM — đây là bài toán timing constraint, không cần CCS.
- **Điều chỉnh ưu tiên Phase 5**: chuyển hoàn toàn focus sang **setup/hold interdependent curve** (không còn 2 nhánh song song 5a/5b), vì đây là bài toán còn novel (GNN-based) và có ground-truth tool open-source hỗ trợ.

#### So sánh cấu trúc bài toán (giữ lại để tham khảo khi có commercial tool)

| Khía cạnh | Setup/Hold Interdependent (Phase 5 — ưu tiên) | CCS-Noise (tạm hoãn đến khi có commercial tool) |
|---|---|---|
| Bản chất output | Đường cong/Pareto front các cặp (setup, hold) hợp lệ | Bảng 2D (noise height × noise width) → output waveform parameterized |
| Độ tương thích pipeline | Trung bình — cần thiết kế thêm output head dạng curve | Cao — tái sử dụng được anchor-sparse-grid |
| Ground-truth tool open-source | ✅ CharLib/ZlibBoost hỗ trợ | ❌ Không có |
| Novelty GNN approach | Cao — chưa ai làm interdependent curve bằng GNN | Cao — chưa có paper nào |
| Khả thi trong MVP | ✅ Có | ❌ Không |

---

## 6. Tiêu chí đánh giá (Evaluation Metrics)

> Phần này được soạn dựa trên best practice ngành (sign-off tolerance) — **cần review lại với yêu cầu cụ thể của team/foundry**, vì đây là phần model chưa có đủ thông tin để khẳng định chắc chắn.

### 6.1 Accuracy metrics
| Metric | Áp dụng cho | Ghi chú |
|---|---|---|
| Relative error (%) so với SPICE ground truth | Delay, transition (NLDM) | Nên dùng % thay vì absolute vì range giá trị lớn |
| MAE / RMSE (log-scale) | Power (NLPM) | Power có thể trải dài nhiều order of magnitude |
| Sigma prediction error | LVF | So sánh phân phối dự đoán vs MC ground truth (KL divergence hoặc relative sigma error) |
| Max error tại các điểm biên (extreme corner) | Tất cả | Quan trọng vì sign-off thường lo nhất ở worst-case corner |

### 6.2 Threshold tham khảo (cần xác nhận lại)
- Sign-off / production: thường yêu cầu sai số **<2–3%** so với full SPICE
- Early ECO / design exploration: có thể nới lỏng hơn (5–10%)
- ⚠️ *Đây là số tham khảo chung của ngành, cần xác nhận với spec thực tế của dự án/foundry trước khi dùng làm gate criteria.*

### 6.3 Generalization / OOD test
- **Out-of-distribution test**: test trên cell có topology GNN chưa từng thấy trong training — quan trọng nhất để chứng minh GNN học được cấu trúc mạch chứ không chỉ overfit theo cell quen thuộc.
- Test trên sizing variant (cùng topology, khác W/L) để đánh giá khả năng generalize theo kích thước transistor.

### 6.4 Efficiency metrics
- Speedup so với full SPICE characterization (số lượng SPICE run giảm bao nhiêu %, thời gian wall-clock giảm bao nhiêu lần)
- So sánh trực tiếp với SetupKit [1] (2.4× CPU time reduction) làm baseline khi vào Phase 5
- Số lượng anchor point cần thiết để đạt target accuracy (trade-off curve: số anchor vs accuracy)

### 6.5 Robustness / Production-readiness
- Monotonicity check pass rate (delay phải tăng đơn điệu theo load/slew)
- Tỷ lệ trigger fallback mechanism (uncertainty cao → cần chạy thêm SPICE) — theo dõi để đánh giá độ tin cậy thực tế
- Confidence calibration: uncertainty model dự đoán có khớp với actual error không

---

## 6.6 Môi trường thử nghiệm ban đầu (Initial Evaluation Setup)

> Giai đoạn đầu (MVP) sẽ dùng toàn bộ stack open-source để dễ tái lập, dễ chia sẻ, không phụ thuộc license thương mại.

- **PDK**: 45nm open-source (FreePDK45 / NanGate 45nm Open Cell Library) — đã có sẵn HSPICE-compatible model (BSIM) và standard cell layout/netlist tham khảo, phù hợp với nhiều công trình academic đã khảo sát ở mục 5 (vd. GATMesh dùng Nangate45).
- **SPICE simulator**: **ngspice** (open-source) thay cho HSPICE — cần kiểm tra/convert model card và netlist syntax tương thích (HSPICE → ngspice có một số khác biệt cú pháp `.model`, `.subckt`, corner file cần adapt).
- **Characterization tool**: **CharLib** (đã chốt) — open-source, Python-based, hỗ trợ NLDM timing characterization cho cả combinational và sequential cell, cấu hình qua single YAML file, hỗ trợ trực tiếp ngspice và Xyce làm backend simulator. Xem bảng so sánh các lựa chọn open-source khác bên dưới.

#### So sánh các open-source characterization tool (tham khảo thêm ngoài CharLib)

| Tool | Đặc điểm chính | Ghi chú liên quan đến dự án |
|---|---|---|
| **CharLib** (đã chọn) | Python-based, config qua YAML, hỗ trợ ngspice/Xyce, hỗ trợ cả combinational + sequential (NLDM) | Khớp trực tiếp với stack đã chọn (45nm open + ngspice); có sẵn benchmark so với commercial tool (MAE <1ns) |
| **lctime** (T. Kramer) | Suy luận logic trực tiếp từ netlist topology | Tiện nhưng có thể không tin cậy với cell phức tạp (feedback loop, custom logic, state-dependent) — ZlibBoost paper có chỉ ra nhược điểm này |
| **libretto** (Nishizawa & Nakura) | Open cell timing characterizer cho open-source VLSI design; CharLib ban đầu dựa trên nền tảng ý tưởng của libretto | Tham khảo lịch sử phát triển, ít hoạt động hơn CharLib hiện tại |
| **LiChEn** | Tập trung vào **asynchronous** standard cell, đã test với >500 component | Không phù hợp trực tiếp (dự án này dùng synchronous cell) nhưng có thể tham khảo kiến trúc |
| **AutoLibGen** | Dùng CCS (Composite Current Source) thay vì NLDM, test ở 65nm | Hướng CCS — ngoài phạm vi MVP hiện tại (mục 1.3) nhưng hữu ích khi mở rộng sang CCS sau này |
| **ZlibBoost** (2024/2025) | Framework mới, kiến trúc front-end/back-end tách biệt, hỗ trợ tích hợp ML model và nhiều simulator, fine-grained task decomposition để tăng tốc song song hóa | **Đáng chú ý nhất** — được thiết kế sẵn để tích hợp ML, có thể là điểm tham khảo kiến trúc tốt cho việc nhúng GNN prediction vào flow characterization. Tự nhận có giải quyết một số hạn chế của CharLib (vd. redundant computation trong xác định timing arc) |

**Khuyến nghị**: Bắt đầu với **CharLib** vì đơn giản, đã có sẵn ecosystem cho ngspice + 45nm, dễ làm MVP nhanh. Khi vào Phase 3-4 (tích hợp GNN prediction vào flow thật) nên đánh giá thêm **ZlibBoost** vì kiến trúc của nó vốn đã hướng tới việc nhúng ML — có thể tiết kiệm effort engineering khi integrate.
- **Mục đích của setup này**:
  1. Validate toàn bộ pipeline (graph construction → GNN → `.lib` writer) trên môi trường miễn phí, dễ debug trước khi chuyển sang PDK/tool thương mại nội bộ.
  2. Baseline ground truth (full SPICE characterization bằng ngspice + open tool) dùng làm "nhãn" supervised learning và mốc so sánh accuracy/runtime ở mục 6.
  3. Kết quả trên 45nm open-source cũng dễ so sánh chéo với các paper academic đã khảo sát (GATMesh, SetupKit dùng node tương đương) vì cùng publicly available PDK.
- **Giới hạn cần lưu ý**: ngspice có thể khác HSPICE về độ chính xác numerical/convergence ở corner cực trị (fast-fast, slow-slow) — cần kiểm chứng trước khi dùng làm ground truth tuyệt đối; khi chuyển sang PDK/HSPICE thương mại ở giai đoạn sau, cần re-validate vì mô hình transistor (BSIM version, model card) có thể khác.

---

## 7. Việc còn thiếu / cần làm rõ (Open Items)

> Đây là các điểm mình note lại vì còn thiếu thông tin hoặc cần quyết định thêm trước khi triển khai chi tiết. Đánh dấu để bạn review.

1. **Threshold accuracy chính xác** (mục 6.2) — cần lấy từ spec thực tế của team/foundry, không nên dùng số tham khảo chung.
2. **Tập cell training cụ thể** — đã chốt dùng 45nm PDK open-source (mục 6.6), nhưng chưa xác định: dùng cụ thể FreePDK45 hay NanGate45 Open Cell Library, số lượng cell, độ đa dạng topology cần thiết để đảm bảo OOD test có ý nghĩa.
3. **Compute budget cho ground truth generation** — chạy full SPICE (ngspice) characterization cho training set tốn bao nhiêu thời gian/compute, có giới hạn gì không.
4. **Chiến lược chọn anchor point — cần nghiên cứu adaptive index selection**: thay vì fixed 2×2/3×3 đều, nên khảo sát các phương pháp chọn anchor thông minh để xử lý tình huống non-monotonicity hoặc non-linearity cục bộ trong bảng. Các hướng cần khảo sát: (a) **active learning / uncertainty sampling** (theo [10]) — chọn điểm tiếp theo dựa trên uncertainty cao nhất của model hiện tại; (b) **gradient-based adaptive sampling** — ước lượng gradient bảng từ vài điểm ban đầu, chọn thêm điểm ở vùng gradient cao; (c) **curvature-aware index placement** — detect vùng non-linear bằng finite difference trước khi chọn anchor. Cần benchmark các chiến lược này so với fixed grid để đánh giá trade-off (số SPICE run thêm vs accuracy gain, đặc biệt ở vùng biên bảng).
5. **LVF cụ thể**: cần làm rõ định nghĩa sigma table đang dùng — early-voltage/late-voltage based hay statistical timing based, vì ảnh hưởng trực tiếp đến cách thiết kế loss function cho LVF head.
6. **Search bổ sung chưa thực hiện**: DAC/ICCAD/DATE 2024–2026, "Bayesian optimization standard cell characterization", "surrogate model timing characterization" — nên làm trước khi bắt đầu Phase 5 để tránh trùng lặp nghiên cứu.
7. **Format .lib writer**: cần xác nhận target STA tool (PrimeTime, Tempus, OpenSTA, etc.) để đảm bảo Liberty syntax/index template tương thích — đặc biệt với MVP open-source, có thể ưu tiên OpenSTA để giữ toàn bộ flow open-source.
8. **Đánh giá uncertainty/fallback mechanism (mục 2.3, 6.5)**: chưa quyết định kỹ thuật cụ thể (ensemble, MC dropout, hay predict variance trực tiếp) — cần chọn trước khi vào Phase 3-4.
9. **[ĐÃ CHỐT] Công cụ characterization open-source**: dùng **CharLib** (xem mục 6.6) — không còn là open item, nhưng cần đọc kỹ bài **[12] Boosting Standard Cell Library Characterization with Machine Learning (ASP-DAC 2025)** càng sớm càng tốt vì có khả năng overlap trực tiếp với hướng GNN của dự án — nên làm ngay đầu Phase 0 trước khi thiết kế chi tiết model.
10. **Kế hoạch chuyển từ open-source (45nm/ngspice) sang PDK + HSPICE thương mại nội bộ** — chưa xác định mốc thời gian/tiêu chí chuyển giai đoạn, cũng như mức độ re-validate cần thiết (model card BSIM version, convergence behavior, corner definition có thể khác đáng kể giữa hai môi trường).
11. **⚠️ QUAN TRỌNG — Định vị lại novelty của dự án**: đã phát hiện bài [16] Ma et al. (ASP-DAC 2024) dùng GNN cho đúng bài toán NLDM/NLPM prediction qua PVT corner, đạt MAPE ≤0.95%, speedup 100x. Cần đọc full-text bài này càng sớm càng tốt (Phase 0) để: (a) xác định họ có dùng sparse-anchor-point approach (2×2/3×3 → full table) như bạn hay không, hay train trên toàn bộ PVT space; (b) xác định rõ đóng góp khác biệt của dự án bạn (có thể là: kết hợp LVF trong cùng pipeline, hoặc mở rộng sang setup/hold mà họ chưa làm).
12. **Cần đọc full-text 2 bài chưa fetch được**: [14] Chen et al. ASP-DAC 2025 (paywall ACM DL) và [1b] Salman et al. 2007 (paywall IEEE Xplore) — hiện chỉ có abstract + trích dẫn gián tiếp, chưa đủ chi tiết phương pháp để thiết kế chính xác model/baseline so sánh.
13. **Rà soát thêm 4 bài mới phát hiện nhưng chưa đọc kỹ**: Cheng et al. (Heterogeneous GAT + parasitic RC reduction, ASP-DAC 2024), Ma et al. (Integration VLSI Journal 2025), LIBMixer (ASP-DAC 2025), GraphCAD (ISPD 2025, GNN cho crosstalk-affected delay) — cả 4 đều rất gần chủ đề graph construction (mục 2.1) của dự án, nên đọc trước khi chốt kiến trúc graph.

---

## 8. Tài liệu tham khảo (Reference List)

1. Zhou, J., Wang, Z., Xia, H., et al. "SetupKit: Efficient Multi-Corner Setup/Hold Time Characterization Using Bias-Enhanced Interpolation and Active Learning." arXiv:2512.00044. https://arxiv.org/pdf/2512.00044
2. "Standard-cell characterization." F-Si Wiki. https://wiki.f-si.org/index.php/Standard-cell_characterization
3. "Automated cell characterization system." US Patent 5,655,109. https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/5655109
4. "Method and system for setup/hold characterization in sequential cells." US Patent 7,795,939. https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/7795939
5. "Setup and hold time characterization device and method." US Patent 7,861,200. https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/7861200
6. "System and method for setup and hold characterization in integrated circuit cells." US Patent 6,640,330. https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/6640330
7. "Characterizing sequential cells using interdependent setup and hold times, and utilizing the sequential cell characterizations in static timing analysis." US Patent 7,774,731 / 7,506,293 (Synopsys). https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/7774731
8. "GATMesh: Clock Mesh Timing Analysis using Graph Neural Networks." arXiv:2507.05681. https://arxiv.org/pdf/2507.05681
9. "Graph Neural Networks for Integrated Circuit Design, Reliability, and Security: Survey and Tool." ACM Computing Surveys, 2024/2025. https://dl.acm.org/doi/full/10.1145/3769081
10. "Why are Graph Neural Networks Effective for EDA Problems?" NVIDIA Research, 2022. https://research.nvidia.com/publication/2022-10_why-are-graph-neural-networks-effective-eda-problems
11. "Machine Learning for Reliability-Aware, yet Confidential Standard Cell Characterization." 2022. https://www.researchgate.net/publication/362109680
12. Salman, E., Dasdan, A., Taraporevala, F., Kucukcakar, K., Friedman, E. G. "Exploiting Setup-Hold-Time Interdependence in Static Timing Analysis." IEEE TCAD, vol. 26, no. 6, pp. 1114-1125, June 2007. doi: 10.1109/TCAD.2006.885834
13. Mellor, M., Stine, J. E. "CharLib: An Open Source Standard Cell Library Characterizer." IEEE 67th MWSCAS, Aug 2024. doi: 10.1109/MWSCAS60917.2024.10658687. https://github.com/stineje/CharLib
14. Chen, Z. et al. "Boosting Standard Cell Library Characterization with Machine Learning." Proceedings of the 30th Asia and South Pacific Design Automation Conference (ASP-DAC), March 2025, pp. 385-391.
15. "ZlibBoost: An Efficient and Flexible Open-Source Framework for Standard Cell Characterization." ACM TODAES, 2024/2025. doi: 10.1145/3747182. https://dl.acm.org/doi/10.1145/3747182
16. Ma, T., Deng, Z., Sun, X., Shao, L. "Fast Cell Library Characterization for Design Technology Co-Optimization Based on Graph Neural Networks." ASP-DAC 2024, pp. 472-477. arXiv:2312.12784. https://arxiv.org/abs/2312.12784
17. Zhou, J., Lin, T.-J., Xia, H., Huang, L., Xing, W., He, L. "LVFGen: Efficient Liberty Variation Format (LVF) Generation Using Variational Analysis and Active Learning." ISPD 2025, pp. 182-190. https://dl.acm.org/doi/10.1145/3698364.3705359
18. Srivastava, S., Roychowdhury, J. "Interdependent Latch Setup/Hold Time Characterization via Euler-Newton Curve Tracing on State-Transition Equations." DAC 2007, pp. 136-141. (bản mở rộng: IEEE TCAD, vol. 27, no. 5, pp. 817-830, 2008)
19. Cheng, X. et al. "Heterogeneous Graph Attention Network Based Statistical Timing Library Characterization with Parasitic RC Reduction." ASP-DAC 2024. *(chưa đọc kỹ — cần khảo sát thêm)*
20. Ma, T. et al. "Graph neural network based cell library characterization method for fast design technology co-optimization." Integration, the VLSI Journal, 101, 2025. doi: 10.1016/j.vlsi.2024.102316 *(chưa đọc kỹ — cần khảo sát thêm)*
21. "LIBMixer: An all-MLP Architecture for Cell Library Characterization towards Design Space Optimization." ASP-DAC 2025. *(chưa đọc kỹ — cần khảo sát thêm)*
22. Liu, F., Guo, G., Ye, Y., Wang, Z., Fu, W., Sheng, W., Yu, B. "GraphCAD: Leveraging Graph Neural Networks for Accuracy Prediction Handling Crosstalk-affected Delays." ISPD 2025, pp. 125-133. ✅ Đã đọc — xem mục 5.8. https://dl.acm.org/doi/10.1145/3698364.3705345
23. Ma, T., Fan, G., Sun, X., Deng, Z., Low, K., Shao, L. "Late Breaking Results: Fast System Technology Co-Optimization Framework for Emerging Technology Based on Graph Neural Networks." arXiv:2404.06939 (bản mở rộng của [16], có GNN-predict setup/hold độc lập). ✅ Đã đọc — xem mục 5.7. https://arxiv.org/pdf/2404.06939

---

*Tài liệu này nên được cập nhật liên tục khi dự án tiến triển — đặc biệt mục 7 (Open Items) cần được resolve trước khi bắt đầu mỗi phase mới.*
