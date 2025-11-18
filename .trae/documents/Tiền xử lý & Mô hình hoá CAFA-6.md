## Mục tiêu
- Xây dựng pipeline tiền xử lý và mô hình hoá multi-label cho GO với ~40k nhãn, tôn trọng DAG.
- Tạo đầu vào ổn định cho huấn luyện, đánh giá và suy luận; sinh submission theo định dạng CAFA.

## Chuẩn bị dữ liệu
- Xác nhận và nạp `Train/train_sequences.fasta`, `Train/train_terms.tsv`, `Train/train_taxonomy.tsv`, `Train/go-basic.obo`, `IA.tsv`.
- Chuẩn hoá `EntryID` từ FASTA (định dạng UniProt `>sp|ID|...`) và đồng bộ với bảng nhãn/taxonomy.
- Kiểm tra rỗng/trùng, loại bỏ entries không có chuỗi hoặc nhãn.

## Parse GO Ontology
- Đọc `go-basic.obo` (ưu tiên `obonet`+`networkx`; dự phòng parser tối giản) để lấy `is_a` edges, `name`, `namespace`.
- Xây map `parents[term]` và các tiện ích: tìm tổ tiên, kiểm tra root, lọc obsolete.

## Chuẩn hoá & mở rộng nhãn
- Mở rộng nhãn từng protein lên tất cả tổ tiên theo DAG để đảm bảo tính đóng theo ontology.
- Tách nhãn theo `aspect` (BP/CC/MF) cho huấn luyện độc lập từng nhánh (tuỳ chọn).
- Tạo `label_index` (GO→cột) và vector hoá multi-hot dạng sparse `CSR` để tiết kiệm bộ nhớ.

## Lọc và chọn nhãn
- Phân tích tần suất; đặt ngưỡng tối thiểu (vd. ≥N lần) cho mỗi aspect để giảm chiều nhãn.
- Tuỳ chọn: gom nhãn hiếm thành nhóm cha gần nhất (roll-up) hoặc giữ để dự đoán bằng heuristics.

## Đặc trưng chuỗi
- Cơ bản: `AAC` (20 chiều) + `k-mer hashing` (k=3–4, dim 1–4k) + meta (length, frac_known).
- Nâng cao: embeddings ESM/ProtBERT nếu khả dụng (cache để tiết kiệm thời gian).
- Chuẩn hoá đặc trưng: z-score/robust scalar; kiểm tra rò rỉ dữ liệu, học scaler từ train.

## Tích hợp thông tin taxonomy & IA
- Thêm đặc trưng taxonomy (one-hot/embedding cho `OX`) hoặc xen kẽ theo aspect.
- Tạo trọng số nhãn từ `IA.tsv` để điều chỉnh loss và ngưỡng quyết định theo độ quan trọng.

## Chia tập dữ liệu
- Split `train/val` với random seed cố định; ưu tiên `iterative stratification` multi-label để giữ phân phối nhãn.
- Tuỳ chọn: k-fold cross-validation per aspect.

## Mô hình baseline
- Binary Relevance: `LogisticRegression` hoặc `Linear SVM` (`OneVsRestClassifier`) trên đặc trưng thủ công.
- Tree-based: `LightGBM`/`XGBoost` (multi-label qua BR) cho hiệu năng nhanh.
- Classifier Chains: khai thác phụ thuộc nhãn; thử một vài chuỗi với seed khác nhau.

## Mô hình nâng cao
- MLP/CNN nhẹ trên AAC+k-mer; dropout, batch norm.
- Embedding-based: fine-tune linear head trên ESM/ProtBERT embeddings; huấn luyện per-aspect.
- Ontology-aware: regularization ép nhất quán DAG (penalty nếu con được dự đoán mà cha không).

## Huấn luyện & tối ưu
- Loss: BCE với trọng số nhãn (từ IA hoặc inverse frequency). Batch size theo bộ nhớ.
- Tối ưu hoá: Adam/SGD, early stopping theo Fmax/Smin trên validation.
- Tìm ngưỡng: per-label hoặc per-aspect bằng cách quét threshold tối ưu Fmax.

## Đánh giá
- Metrics CAFA: `Fmax`, `Smin`, `AUPR`; báo cáo theo BP/CC/MF.
- Kiểm tra tính đóng DAG của dự đoán: sau suy luận, ép ancestors (post-processing) để hợp lệ.
- Phân tích lỗi: confusion nhãn, độ phủ ontology, ảnh hưởng taxonomy.

## Suy luận & Submission
- Pipeline suy luận: đặc trưng → dự đoán xác suất → áp ngưỡng → đóng DAG → sinh `sample_submission.tsv`.
- Đảm bảo định dạng cột (protein ID, GO term, score nếu yêu cầu) và sắp xếp ổn định.

## Kiến trúc pipeline
- Module hoá: loader, ontology, features, labeler, splitter, models, metrics, inference.
- Sử dụng cache (embeddings, ancestor sets) và lưu artifacts (scaler, label_index, model params).

## Kiểm thử & xác nhận
- Unit tests nhẹ: parser OBO, mở rộng tổ tiên, vector hoá nhãn, nhất quán ancestors ở đầu ra.
- Sanity checks: phân phối đặc trưng, tần suất nhãn trước/sau lọc, chất lượng split.

## Tái lập & cấu hình
- Seed cố định; cấu hình hyperparams bằng YAML/JSON hoặc biến trong notebook.
- Ghi lại phiên bản thư viện; thời gian chạy từng khối.

## Kế hoạch Notebooks
- `01_Data_Exploration_and_Preprocessing.ipynb`: EDA + parse ontology + thống kê.
- `02_Preprocessing_and_Feature_Engineering.ipynb`: mở rộng nhãn, multi-hot, AAC/k-mer, split, IA.
- `03_Baseline_Modeling.ipynb`: BR Logistic/SVM/LightGBM, huấn luyện và Fmax/Smin.
- `04_Embedding_Models.ipynb`: ESM/ProtBERT, linear head, ngưỡng theo aspect.
- `05_Inference_and_Submission.ipynb`: post-processing DAG, tạo submission.

## Đầu việc cụ thể
- Viết tiện ích: đọc dữ liệu, parser OBO, ancestors, vector hoá nhãn sparse.
- Xây dựng features AAC+k-mer và scaler; tích hợp taxonomy.
- Lọc nhãn theo ngưỡng; tạo splits với iterative stratification.
- Huấn luyện baseline BR; chọn ngưỡng tối ưu Fmax; đánh giá per-aspect.
- Thêm post-processing DAG cho dự đoán; sinh submission.

Vui lòng xác nhận để bắt đầu hiện thực hoá (tạo/hoàn thiện notebooks 03–05, tiện ích chung, và chạy baseline).