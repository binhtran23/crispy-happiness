# LEC_DAY21 — Quiz

## Q1 [priority: Essential]
**Question:** Điểm khác biệt nào khiến deploy trong AI CI/CD được coi là "non-deterministic" và cần thêm một cơ chế đặc biệt so với software truyền thống?
- A) Vì code AI luôn có bug nên phải test nhiều hơn
- B) Vì cùng code + data, train lại có thể ra model khác nhau, nên phải có Eval Gate so sánh với production trước khi deploy
- C) Vì AI deploy chậm hơn nên cần cache
- D) Vì AI không dùng Git

**Answer:** B
**Explanation:** Non-deterministic nghĩa là kết quả train không cố định (random seed, thứ tự batch, phần cứng...), nên "code test pass" không đảm bảo model mới tốt bằng cũ → cần Eval Gate. A/C/D không liên quan đến tính non-deterministic.

## Q2 [priority: Essential]
**Question:** Trong Eval Gate, tại sao test set BẮT BUỘC phải cố định (fixed, non-shuffled) khi so sánh model mới với model production?
- A) Để tiết kiệm bộ nhớ khi load test set
- B) Để test set nhỏ hơn, chạy nhanh hơn
- C) Để delta accuracy phản ánh thuần chênh lệch năng lực model, không bị test set thay đổi làm nhiễu (confound)
- D) Vì MLflow yêu cầu test set cố định

**Answer:** C
**Explanation:** Nếu test set shuffle mới mỗi lần, model mới và model cũ bị chấm trên hai bộ đề khác nhau → không biết delta đến từ model tốt hơn hay tập đề dễ hơn. Cố định test set giữ biến này là hằng số. Lý do "reproducible/kiểm chứng" đúng nhưng thứ yếu; A/B/D là distractor.

## Q3 [priority: Essential]
**Question:** Eval Gate chỉ tính `delta = new_acc - prod_acc` với threshold 2%. Rủi ro lớn nhất của cấu hình chỉ-một-metric này là gì?
- A) Không có rủi ro gì, accuracy là đủ
- B) Một model accuracy cao hơn nhưng nặng hơn có thể làm latency P95 tăng vọt mà gate không phát hiện
- C) Threshold 2% quá cao
- D) Gate chạy quá chậm

**Answer:** B
**Explanation:** Single-metric gate là điểm mù: model có thể regression về latency, fairness/bias trong khi accuracy vẫn đạt. Best practice là multi-metric gating (accuracy ±2%, latency P95 ±10%, F1, fairness), mỗi chiều một threshold.

## Q4 [priority: Essential]
**Question:** DVC lưu data trên remote (vd S3) theo content-addressable storage. Rollback về data version cũ hoạt động như thế nào?
- A) DVC gọi API undo của S3 để khôi phục file cũ
- B) git checkout con trỏ (.dvc) cũ → chứa hash cũ → dvc pull lấy object cũ vẫn còn nguyên trên remote (vì DVC không ghi đè, chỉ thêm object mới)
- C) DVC train lại model từ đầu để tái tạo data
- D) Không thể rollback data với DVC

**Answer:** B
**Explanation:** Content-addressable = lưu theo hash, không bao giờ ghi đè; version cũ (hash cũ) vẫn tồn tại trên remote. Rollback = quay con trỏ về hash cũ + pull. Giới hạn: nếu object bị xoá tay/lifecycle policy expire thì con trỏ dangling và pull fail.

## Q5 [priority: Essential]
**Question:** Phát biểu nào ĐÚNG về quan hệ giữa DVC và MLflow?
- A) DVC thay thế MLflow, chỉ cần dùng một
- B) MLflow version data, DVC track metrics
- C) DVC = pipeline + data versioning; MLflow = metrics + model registry; dùng cả hai vì bổ sung nhau
- D) Cả hai đều chỉ dùng cho LLM

**Answer:** C
**Explanation:** Bài nhấn mạnh "use both, don't choose one" — chúng bổ sung: DVC trả lời "data nào, stage nào tạo ra"; MLflow trả lời "run nào, model version nào, stage lifecycle nào". B đảo ngược vai trò.

## Q6 [priority: Essential]
**Question:** Chiến lược deployment nào mirror toàn bộ traffic sang model mới nhưng KHÔNG trả response của nó cho user (chỉ log để so sánh)?
- A) Canary
- B) Blue/Green
- C) Shadow (Dark Launch)
- D) Rolling

**Answer:** C
**Explanation:** Shadow = zero rủi ro user-facing (response v2 chỉ log, không trả), đổi lại không đo được phản ứng thật của user + gấp đôi compute. Canary chia traffic một phần tăng dần; Blue/Green switch toàn bộ; Rolling thay pod lần lượt.

## Q7 [priority: Essential]
**Question:** Trong CI pipeline cho AI, tại sao Data Validation được đặt TRƯỚC Model Training?
- A) Vì Great Expectations chạy chậm nên phải chạy trước
- B) Fail-fast: nếu data hỏng, chặn pipeline ngay trước khi đốt hàng giờ GPU để train ra model vô dụng
- C) Vì training không cần data
- D) Để clean và chuẩn hoá data cho model

**Answer:** B
**Explanation:** Nguyên tắc fail-fast tiết kiệm tài nguyên training. Lưu ý D sai: Data Validation là quality GATE (phát hiện + chặn), KHÔNG phải bước clean/chuẩn hoá data.

## Q8 [priority: Important] [weak-area]
**Question:** Great Expectations trong pipeline làm nhiệm vụ gì?
- A) Tự động sửa và chuẩn hoá dữ liệu bẩn trước khi train
- B) Kiểm tra chất lượng data (schema, null rate, drift, volume, freshness) và FAIL pipeline nếu vi phạm, nhưng không tự sửa data
- C) Train model
- D) Deploy model lên production

**Answer:** B
**Explanation:** Great Expectations là quality GATE: phát hiện data xấu và chặn, KHÔNG biến đổi/clean data. Đây là điểm dễ nhầm giữa "validation (gate)" và "preprocessing (cleaning)".

## Q9 [priority: Important]
**Question:** Cơ chế nào của Canary deployment giúp giới hạn "blast radius" khi model mới có lỗi mà Eval Gate không bắt được?
- A) Chạy smoke test trước
- B) Route một tỉ lệ nhỏ traffic (vd 5%) sang model mới và tăng dần qua các health-check gate, rollback nếu P99 latency vượt ngưỡng hoặc accuracy giảm >2%
- C) Deploy song song 2 env đầy đủ rồi switch
- D) Mirror traffic không trả response

**Answer:** B
**Explanation:** Canary giới hạn thiệt hại vì lỗi chỉ chạm % nhỏ user, phát hiện sớm trên traffic thật rồi rollback trước khi lên 100%. C là Blue/Green, D là Shadow.

## Q10 [priority: Important]
**Question:** Trong A/B testing, tại sao phải hash user_id để cùng một user luôn thấy cùng một variant?
- A) Để tiết kiệm băng thông
- B) Vì thuật toán MD5 nhanh
- C) Đảm bảo UX nhất quán VÀ tránh nhiễm chéo phép đo (outcome quy được về đúng một model) → so sánh sạch
- D) Để model mới luôn nhận nhiều traffic hơn

**Answer:** C
**Explanation:** Deterministic routing giữ user dính một variant → mỗi click/conversion quy được về đúng một model. Nếu random mỗi request, dữ liệu A/B bị nhiễm chéo và kết luận "model nào thắng" vô nghĩa — cùng tinh thần fixed test set ở Eval Gate.

## Q11 [priority: Important]
**Question:** Model Test loại "Invariance" khác "Directional" ở điểm nào?
- A) Invariance đổi input theo cách KHÔNG nên ảnh hưởng output → output giữ nguyên; Directional đổi input theo cách NÊN ảnh hưởng → output đổi đúng chiều
- B) Chúng giống hệt nhau
- C) Invariance chỉ dùng cho text, Directional chỉ cho ảnh
- D) Directional kiểm tra tốc độ, Invariance kiểm tra accuracy

**Answer:** A
**Explanation:** Ví dụ: Invariance = xoay ảnh 90° → prediction không đổi. Directional = thêm "not" vào câu → sentiment đảo chiều. Tiêu chí: thay đổi input có "nên" làm output đổi hay không.

## Q12 [priority: Important]
**Question:** Trong MLflow Model Registry, alias "champion" và "challenger" khác nhau ở điểm nào?
- A) champion là model tốt hơn về mặt kiến trúc
- B) champion = version đang chạy Production; challenger = version ứng viên đang A/B test
- C) champion dùng cho LLM, challenger cho traditional ML
- D) Chúng là hai loại model khác nhau

**Answer:** B
**Explanation:** Tiêu chí phân biệt là VAI TRÒ/STAGE trong Registry, không phải loại model. champion = Production hiện tại; challenger = candidate đang thử nghiệm A/B.

## Q13 [priority: Supporting]
**Question:** Vì sao Testing Pyramid có nhiều unit test ở đáy và ít load test ở đỉnh?
- A) Vì unit test quan trọng hơn load test
- B) Unit test nhanh & rẻ nhất (làm nhiều, bắt lỗi sớm & rẻ); load test chậm & tốn tài nguyên nhất (làm ít)
- C) Vì load test không cần thiết
- D) Vì đó là quy định bắt buộc của pytest

**Answer:** B
**Explanation:** Kim tự tháp tối ưu chi phí bắt lỗi: nhiều test rẻ ở đáy bắt bug sớm; bug lọt xuống load test thì đắt hơn nhiều để tìm và sửa.
