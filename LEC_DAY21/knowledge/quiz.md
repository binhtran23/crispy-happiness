# LEC_DAY21 — Quiz

## Q1 [priority: Essential]
**Question:** Vì sao deploy trong AI CI/CD được coi là "non-deterministic", dẫn tới nhu cầu về Eval Gate?
- A) Cùng code và data, mỗi lần train lại có thể ra model khác nhau
- B) Code AI thường chứa nhiều bug ẩn hơn code thường
- C) Model AI luôn cần GPU nên build lâu và dễ lỗi
- D) AI không dùng Git nên không kiểm soát được version

**Answer:** A
**Explanation:** Non-deterministic đến từ random seed, thứ tự batch, phần cứng — nên "test pass" không đảm bảo model mới không kém model cũ, cần Eval Gate so sánh. B và C mô tả đặc điểm khác không phải nguyên nhân non-deterministic; D sai vì AI vẫn dùng Git (cộng DVC).

## Q2 [priority: Essential]
**Question:** Trong Eval Gate, lý do CHÍNH khiến test set phải cố định (fixed, non-shuffled) là gì?
- A) Giúp con người kiểm chứng lại kết quả về sau
- B) Giữ test set nhỏ để job chạy nhanh và tốn ít RAM
- C) Tránh test set thay đổi làm nhiễu (confound) phép so sánh delta
- D) Vì MLflow bắt buộc test set phải cố định khi log

**Answer:** C
**Explanation:** Nếu test set đổi mỗi lần, không biết delta đến từ model tốt hơn hay đề dễ hơn. A đúng nhưng chỉ là lý do thứ yếu (hệ quả). B sai mục đích; D bịa ràng buộc không có.

## Q3 [priority: Essential]
**Question:** Eval Gate chỉ chấm một metric là accuracy với threshold 2%. Rủi ro lớn nhất của cấu hình này?
- A) Threshold 2% quá chặt nên model tốt vẫn bị chặn nhầm
- B) Model mới accuracy cao hơn nhưng nặng hơn làm latency P95 tăng vọt
- C) Accuracy tính chậm khiến gate trở thành nút thắt cổ chai
- D) Một metric là đủ, cấu hình này không có rủi ro đáng kể

**Answer:** B
**Explanation:** Single-metric là điểm mù: model có thể regression về latency, fairness/bias mà accuracy vẫn đạt → cần multi-metric gating. A hiểu sai chiều của threshold; C nhầm về hiệu năng tính toán; D chủ quan.

## Q4 [priority: Essential]
**Question:** DVC lưu data trên remote theo content-addressable storage. Rollback về data version cũ diễn ra thế nào?
- A) DVC train lại pipeline để tái tạo đúng data cũ
- B) DVC gọi API undo của S3 để phục hồi file đã ghi đè
- C) Sao chép data hiện tại rồi chỉnh tay về trạng thái cũ
- D) git checkout con trỏ .dvc cũ (hash cũ) rồi dvc pull object cũ còn nguyên trên remote

**Answer:** D
**Explanation:** Content-addressable = lưu theo hash, không ghi đè, nên object cũ vẫn tồn tại; rollback là trỏ lại hash cũ + pull. B giả định S3 ghi đè (sai bản chất); A và C không phải cơ chế của DVC.

## Q5 [priority: Essential]
**Question:** Phát biểu nào mô tả ĐÚNG quan hệ giữa DVC và MLflow?
- A) DVC lo pipeline + data versioning, MLflow lo metrics + model registry, dùng cả hai
- B) MLflow version data, còn DVC chuyên track metrics thí nghiệm
- C) DVC là bản thay thế nhẹ hơn của MLflow, chỉ cần chọn một
- D) Cả hai đều là công cụ chuyên dụng riêng cho LLMOps

**Answer:** A
**Explanation:** Bài nhấn "use both, don't choose one" — hai công cụ bổ sung nhau. B đảo ngược vai trò; C sai vì chúng không thay thế nhau; D sai phạm vi (đều dùng chung cho ML).

## Q6 [priority: Essential]
**Question:** Chiến lược deployment nào cho model mới xử lý toàn bộ traffic nhưng KHÔNG trả response của nó về user?
- A) Canary — chia 5% traffic rồi tăng dần qua các health check
- B) Blue/Green — dựng v2 song song rồi switch toàn bộ traffic
- C) Shadow — mirror traffic sang v2, chỉ log, không trả response
- D) Rolling — thay lần lượt từng pod trong K8s deployment

**Answer:** C
**Explanation:** Shadow (Dark Launch) đạt zero rủi ro user-facing vì response v2 chỉ để log/so sánh. A chia một phần traffic và có trả response; B switch toàn bộ; D thao tác ở tầng pod.

## Q7 [priority: Essential]
**Question:** Vì sao Data Validation được đặt TRƯỚC Model Training trong CI pipeline?
- A) Fail-fast: data hỏng thì chặn ngay, khỏi đốt hàng giờ GPU vô ích
- B) Vì Great Expectations chạy chậm nên cần ưu tiên khởi động sớm
- C) Để clean và chuẩn hoá dữ liệu trước khi đưa vào training
- D) Vì bước training không thực sự cần đến dữ liệu đầu vào

**Answer:** A
**Explanation:** Nguyên tắc fail-fast tiết kiệm tài nguyên. C là bẫy hay gặp: Data Validation là quality GATE (phát hiện + chặn), không phải bước clean. B và D sai về vai trò của các stage.

## Q8 [priority: Important] [weak-area]
**Question:** Great Expectations đảm nhận vai trò gì trong pipeline?
- A) Tự động sửa và chuẩn hoá dữ liệu bẩn trước khi train
- B) Kiểm tra chất lượng data và fail pipeline nếu vi phạm, nhưng không sửa data
- C) Sinh thêm dữ liệu tổng hợp để cân bằng các lớp thiểu số
- D) Chọn ra tập held-out cố định để Eval Gate dùng so sánh

**Answer:** B
**Explanation:** Great Expectations là quality gate: phát hiện data xấu (schema, null, drift, volume, freshness) và chặn, không biến đổi data. A nhầm validation với cleaning; C và D là chức năng của công cụ/bước khác.

## Q9 [priority: Important]
**Question:** Cơ chế nào của Canary giúp giới hạn "blast radius" khi model mới có lỗi Eval Gate không bắt được?
- A) Dựng hai môi trường đầy đủ rồi switch toàn bộ khi sẵn sàng
- B) Route một phần nhỏ traffic tăng dần qua các health-check gate, rollback nếu vượt ngưỡng
- C) Mirror toàn bộ traffic sang model mới nhưng không trả response
- D) Thay lần lượt từng pod cho tới khi cụm chạy hết model mới

**Answer:** B
**Explanation:** Canary chỉ để lỗi chạm % nhỏ user, phát hiện sớm trên traffic thật rồi rollback (P99 latency vượt ngưỡng hoặc accuracy giảm >2%). A là Blue/Green, C là Shadow, D là Rolling.

## Q10 [priority: Important]
**Question:** Trong A/B testing, vì sao phải hash user_id để mỗi user luôn thấy cùng một variant?
- A) Để phân bổ nhiều traffic hơn cho variant mới cần dữ liệu
- B) Vì MD5 tính nhanh nên giảm được độ trễ khi routing
- C) Để tiết kiệm băng thông giữa load balancer và model
- D) Giữ UX nhất quán và tránh nhiễm chéo phép đo, nhờ đó so sánh sạch

**Answer:** D
**Explanation:** Deterministic routing giữ user dính một variant → mỗi click/conversion quy được về đúng một model. A, B, C nêu lợi ích phụ hoặc không liên quan tới lý do consistency của phép đo.

## Q11 [priority: Important]
**Question:** Model Test loại "Invariance" khác "Directional" ở tiêu chí nào?
- A) Invariance đổi input theo cách không nên ảnh hưởng → output giữ nguyên; Directional thì output phải đổi đúng chiều
- B) Invariance chỉ áp dụng cho ảnh, còn Directional chỉ áp dụng cho văn bản
- C) Invariance đo độ chính xác, còn Directional đo tốc độ suy luận của model
- D) Invariance kiểm tra trên golden set, còn Directional kiểm tra trên data drift

**Answer:** A
**Explanation:** Tiêu chí là thay đổi input CÓ nên làm output đổi hay không: xoay ảnh 90° → giữ nguyên (invariance); thêm "not" → đảo sentiment (directional). B, C, D nêu tiêu chí sai (modal, loại metric, loại tập test).

## Q12 [priority: Important]
**Question:** Trong MLflow Registry, alias "champion" và "challenger" phân biệt nhau theo tiêu chí nào?
- A) Theo loại model: champion cho traditional ML, challenger cho LLM
- B) Theo vai trò/stage: champion là version Production hiện tại, challenger là ứng viên A/B
- C) Theo kiến trúc: champion có accuracy cao hơn nên được đặt tên vậy
- D) Theo thời điểm tạo: champion là version cũ nhất còn lưu trong registry

**Answer:** B
**Explanation:** Phân biệt bằng VAI TRÒ trong lifecycle, không phải loại/kiến trúc/tuổi model. A, C, D đều gán sai tiêu chí.

## Q13 [priority: Supporting]
**Question:** Vì sao Testing Pyramid có nhiều unit test ở đáy và ít load test ở đỉnh?
- A) Vì unit test quan trọng hơn nên cần viết nhiều hơn load test
- B) Vì pytest chỉ hỗ trợ unit test, còn load test cần công cụ khác
- C) Vì unit test nhanh/rẻ (bắt lỗi sớm), load test chậm/tốn tài nguyên (làm ít)
- D) Vì load test có thể bỏ qua nếu unit test đã pass đầy đủ

**Answer:** C
**Explanation:** Kim tự tháp tối ưu chi phí bắt lỗi: nhiều test rẻ ở đáy chặn bug sớm; bug lọt tới load test thì đắt hơn để tìm/sửa. A nhầm "quan trọng" với "số lượng"; B và D sai (load test vẫn cần và không thể bỏ).
