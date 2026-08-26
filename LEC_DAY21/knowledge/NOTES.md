# LEC_DAY21 — CI/CD for AI Systems

## Keypoints
- AI CI/CD khác software truyền thống: non-deterministic (→ cần Eval Gate), data cũng là input của pipeline (→ DVC + data validation), rollback trigger gồm cả accuracy drop/bias, build time có thể hàng giờ (training).
- MLflow = lifecycle management (KHÔNG phải QA). 4 thành phần: Tracking / Projects / Models / Registry. Tracking log 4 entity: params (immutable), metrics (per step/epoch), artifacts, signature (schema I/O, bắt buộc để serving).
- MLflow Registry lifecycle: None → Staging → Production → Archived (+ reject loop). Alias champion (Production hiện tại) / challenger (A/B candidate). Webhook trên stage transition → trigger CD. Approval workflow: ≥1 reviewer trước Staging→Production.
- MLflow có LLM Autolog (2.8+) — hỗ trợ OpenAI/LangChain/LlamaIndex/Anthropic Claude — nhưng phần LLM-specific (prompt versioning, LangSmith/W&B Weave) để dành Day 22.
- DVC = "Git cho data". Pointer file `.dvc` chứa hash; Git track con trỏ, remote (S3...) lưu nội dung theo content-addressable storage (bất biến, dedup → 100 version không tốn thêm chỗ). Rollback = git checkout con trỏ cũ + dvc pull (object cũ vẫn còn trên remote).
- DVC pipeline (`dvc.yaml`): stages prepare/train/evaluate với cmd/deps/outs/metrics. `dvc repro` chỉ chạy lại stage stale (smart caching).
- DVC vs MLflow: DVC = pipeline + data versioning; MLflow = metrics + model registry. → Dùng cả hai, bổ sung nhau, không thay thế.
- Eval Gate = "tấm lưới an toàn quan trọng nhất": load model mới + production, eval trên FIXED held-out test set, delta = acc_new - acc_prod; delta < -threshold (default 2%) → exit failure → block deploy. Best practice: multi-metric (accuracy ±2%, latency P95 ±10%, F1, AUC), log vào MLflow, post PR comment, borderline → manual review.
- Data Validation (Great Expectations) = quality GATE (phát hiện + chặn, KHÔNG clean data). Đặt trước Training → fail-fast, tiết kiệm GPU. Checks: schema, null rate ≤5%, drift (KL/KS), duplicates <0.1%, label balance ≥10%, volume ≥10k rows, freshness ≤7 ngày.
- GitHub Actions: triggers on push/PR; secrets qua `secrets.*` (không hardcode); dependency caching (`actions/cache`, cắt 60-80% setup); job deps qua `needs:` (hiện thực fail-fast); path filters; OIDC federation cho production.
- 4 deployment strategies: Canary (chia traffic dần 5%→100%, rollback nếu P99 latency vượt ngưỡng OR accuracy giảm >2%), Blue/Green (switch toàn bộ giữa 2 env đầy đủ, giữ v1 ~30 phút), Shadow/Dark Launch (mirror traffic, không trả response v2), Rolling (thay pod lần lượt trong K8s).
- AI Testing Pyramid (dưới→trên): Unit → Integration → Model → Data → Load. Model Tests: Behavioral, Invariance, Directional, Regression (golden set ≥ v_prev-0.5%), Fairness (<3% gap).
- A/B testing: deterministic routing hash(user_id) MD5 → cùng user cùng variant; min 1000 samples/variant; p<0.05 (Bonferroni khi multiple comparisons); Primary metrics (CTR/revenue) vs Guardrail metrics (latency/error không được tệ đi).

## Terms
- Eval Gate — cổng so sánh model mới vs production trên fixed test set, chặn deploy nếu accuracy giảm quá threshold.
- MLflow Tracking — log params/metrics/artifacts/signature cho mỗi run.
- Model Registry — quản lý stage lifecycle của model (None/Staging/Production/Archived).
- champion / challenger — alias: model Production hiện tại / ứng viên A/B.
- DVC (Data Version Control) — version control cho data qua pointer file + content-addressable storage.
- content-addressable storage — lưu dữ liệu theo hash nội dung, bất biến, dedup.
- pointer file (.dvc) — file nhỏ chứa hash của data, được Git track.
- Great Expectations — công cụ data validation (quality gate) trong CI.
- data drift (distribution drift) — thay đổi phân phối dữ liệu theo thời gian.
- fail-fast — dừng pipeline sớm khi phát hiện lỗi (data validation trước training).
- Canary / Blue-Green / Shadow / Rolling — 4 chiến lược deployment.
- Testing Pyramid — 5 tầng test (unit→integration→model→data→load).
- guardrail metric — metric không được phép tệ đi trong A/B test (latency, error rate).

## Covered / To revisit
- [x] AI CI/CD vs traditional software — vững, tự suy ra được hàng "data là input".
- [x] MLflow Tracking + Registry lifecycle — vững; hiểu vai trò approval của con người (bias, borderline, business context).
- [x] DVC pointer + content-addressable + rollback — rất vững, hỏi câu sâu và chỉ ra được giới hạn.
- [x] Eval Gate cơ chế + fixed test set + multi-metric — vững; bắt được lỗ hổng single-metric (latency).
- [x] 4 deployment strategies — vững, phân biệt 3/3 tình huống đúng.
- [x] Testing Pyramid hình dạng + chi phí — vững.
- [x] A/B testing hashing consistency — vững.
- [ ] Data Validation vs preprocessing — ban đầu lẫn "validation" với "cleaning/chuẩn hoá"; đã làm rõ nhưng nên ôn lại (GE là gate phát hiện+chặn, không sửa data).

## Misconceptions / Examiner findings
- Data Validation — ban đầu hiểu Data Validation là bước xử lý/chuẩn hoá data để "show cái tốt cho model". Gap: nhầm quality GATE (phát hiện + fail pipeline) với preprocessing (biến đổi data). Đã đính chính; cũng làm rõ ranh giới bỏ-lỗi vs giữ-độ-khó (không curate quá tay kẻo mất generalization).
- Fixed test set — ban đầu giải thích lý do cố định là "để con người kiểm chứng" (reproducible), đúng nhưng là lý do thứ yếu. Lý do chính: tránh confound — nếu test set đổi mỗi lần thì delta accuracy không quy được về năng lực model. Đã bổ sung.
- LLM vs traditional ML (câu hỏi tốt của student) — hiểu đúng rằng Day 21 nghiêng traditional ML; cần nhớ MLflow không giới hạn ở đó (có LLM Autolog) và MLflow là lifecycle mgmt chứ không phải QA.
