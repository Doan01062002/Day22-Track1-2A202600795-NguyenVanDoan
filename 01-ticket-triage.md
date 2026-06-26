# Case 1 - Support Ticket Triage

## Mục tiêu

Case này giúp học viên luyện 4 câu hỏi nên bật ra ngay khi gặp một AI task:

- Cái gì deterministic và nên chấm bằng code?
- Cái gì cần semantic judgment và nên giao cho LLM judge hoặc human?
- Cái gì high-risk nên cần gate chặt hơn?
- Sai ở đâu thì cần escalation sang người thật?

Chỉ cần thiết kế eval ban đầu, không cần code full system.

Case này nối trực tiếp từ track **AI Customer Support Agent** ở Day 18/19, nhưng đổi góc nhìn từ **thiết kế trải nghiệm** sang **thiết kế eval**.

---

## 1. Bối cảnh

Một công ty SaaS B2B dùng AI để đọc ticket support mới và tạo output triage cho hệ thống nội bộ.

Output này không gửi trực tiếp cho khách hàng, nhưng nó được dùng để:

- phân loại ticket,
- đánh dấu mức độ gấp,
- route đến đúng team,
- quyết định có cần người thật nhảy vào hay không.

Nếu AI route sai, ticket có thể bị trễ, bỏ sót escalation, hoặc đẩy sai sang team không xử lý được.

---

## 2. Workflow logic (ASCII)

```text
Khách hàng gửi ticket hỗ trợ
    ↓
AI đọc:
- tiêu đề
- nội dung ticket
- loại khách hàng
    ↓
Hệ thống phải quyết định:
- đây là loại vấn đề gì?
- mức độ khẩn cấp ra sao?
- có cần người thật xử lý ngay không?
- ticket nên vào hàng của team nào?
    ↓
UI inbox nội bộ hiển thị:
- nhãn loại yêu cầu
- mức độ khẩn
- team phụ trách
- cờ "cần xử lý ngay"
- lý do tóm tắt
    ↓
Nếu khách doanh nghiệp + có dấu hiệu chặn công việc
    ↓
Đẩy lên hàng ưu tiên cao / escalation
```

---

## 3. UI hiển thị dự kiến (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Công ty ABC (Enterprise)                            |
| Tiêu đề: Thanh toán lỗi, tài khoản bị khóa                      |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: [ ? ]                                           |
| - Mức độ khẩn: [ ? ]                                            |
| - Team phụ trách: [ ? ]                                         |
| - Cần người xử lý ngay: [ ? ]                                   |
| - Lý do tóm tắt: [ .......................................... ] |
|----------------------------------------------------------------|
| Hàng đợi hiện tại: [ Bình thường ] hoặc [ Ưu tiên cao ]         |
+----------------------------------------------------------------+
```

Học viên cần tự đề xuất output contract tối thiểu phía sau để màn hình này hiển thị được.

---

## 4. Input mẫu

```json
{
  "ticket_id": "T-001",
  "subject": "Cannot login after password reset",
  "message": "I reset my password twice but still cannot log in. This is blocking my work.",
  "customer_tier": "enterprise"
}
```

Một input khác:

```json
{
  "ticket_id": "T-002",
  "subject": "URGENT: payment failed and account disabled",
  "message": "Our team is locked out because your billing system failed. Fix this now.",
  "customer_tier": "enterprise"
}
```

---

## 5. Business rules / operational rules

- Output phải đúng schema và đúng allowed enums.
- `confidence` phải nằm trong khoảng `0-1`.
- Nếu `customer_tier = enterprise` và `urgency` là `high` hoặc `critical`, `requires_human` phải bằng `true`.
- Ticket billing không được route sang `product_team`.
- Ticket có dấu hiệu “blocking work”, “locked out”, hoặc “account disabled” không nên bị đánh `low`.
- `reason_codes` phải phản ánh được nội dung ticket, không được bốc thêm sự thật không có trong input.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

Khách hàng doanh nghiệp nhắn vào kênh hỗ trợ:

```text
Chị ơi bên em reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào được.
Bên em đang bị chặn công việc từ sáng.
```

### Data mẫu

- `customer_tier`: `enterprise`
- `account_name`: `Công ty Minh Phát Logistics`
- `previous_tickets_7d`: `0`
- `channel`: `Zalo OA`

### Workflow ASCII

```text
Khách nhắn vấn đề đăng nhập
    ↓
AI đọc nội dung + loại khách hàng
    ↓
AI phát hiện tín hiệu:
- login issue
- blocked work
- enterprise customer
    ↓
Hệ thống gợi ý:
- category = technical
- urgency = high hoặc critical
- requires_human = true
- route_to = technical_support
    ↓
UI nội bộ đẩy ticket lên hàng ưu tiên
    ↓
Nhân viên hỗ trợ xem lại rồi tiếp nhận
```

### UI trước khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Kênh: Zalo OA                                                   |
| Khách hàng: Minh Phát Logistics                                 |
|----------------------------------------------------------------|
| Nội dung khách nhắn:                                            |
| "Reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào..."  |
|----------------------------------------------------------------|
| AI gợi ý: Chưa có                                               |
+----------------------------------------------------------------+
```

### UI sau khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Khách hàng: Minh Phát Logistics (Enterprise)                    |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Technical                                       |
| - Mức độ khẩn: High                                             |
| - Team phụ trách: Technical Support                             |
| - Cần người xử lý ngay: Có                                      |
| - Lý do tóm tắt: Lỗi đăng nhập đang chặn công việc              |
| - Hàng đợi: Ưu tiên cao                                         |
+----------------------------------------------------------------+
```

Ví dụ này giúp người đọc hình dung ngay:

- AI đang quyết định gì,
- quyết định nào hiển thị ra UI,
- và sai ở đâu thì ảnh hưởng vận hành.

---

## 7. Seed cases

Đây không phải full dataset. Đây chỉ là 3 seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Happy path

- `subject`: `Cannot login after password reset`
- Kỳ vọng: `category = technical`, `requires_human = true` nếu urgency đủ cao, route về `technical_support`

### Seed B - Ambiguous / low-info

- `subject`: `Help`
- `message`: `Please help asap`
- Kỳ vọng: AI không nên tự tin gán category quá mạnh; cần `unknown` hoặc route theo hướng cần review

### Seed C - High-risk / escalation

- `subject`: `URGENT: payment failed and account disabled`
- Kỳ vọng: `category = billing`, `urgency = critical`, `requires_human = true`, route về `billing_ops` hoặc `human_escalation`

---

## 8. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc seed cases ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nộp một bảng coverage riêng. Hãy chọn 5 case đại diện cho các lát cắt khác nhau, ví dụ: match rõ, thiếu tín hiệu, ambiguity, escalation, và regression.

1. Happy path:
2. Ambiguous input:
3. Missing information:
4. High-risk / escalation:
5. Regression case:

Với mỗi case, thêm 1 dòng ngắn giải thích:

- case này dùng để bắt failure gì?

---

## 9. Mock outcome để soi

Giả sử trên UI nội bộ, hệ thống hiển thị kết quả gợi ý như sau cho `T-002`:

```text
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Enterprise                                          |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Product question                                |
| - Mức độ khẩn: Medium                                           |
| - Team phụ trách: Support L1                                    |
| - Cần người xử lý ngay: Không                                   |
| - Lý do tóm tắt: Có vấn đề thanh toán                           |
| - Độ tin cậy: 0.91                                              |
+----------------------------------------------------------------+
```

Kết quả này trông có vẻ “ổn” nếu chỉ nhìn bề mặt, nhưng khả năng cao là sai về judgment vận hành.

---

## 10. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết eval runner,
- viết prompt judge thật,
- làm lại `User Input Grid` đầy đủ như bài test inputs hôm trước,
- tạo full dataset lớn,
- code full system.

Cần làm:

- chọn đúng nguồn chấm cho từng thành phần,
- viết các rule kiểm tra đủ cụ thể để có thể implement sau,
- đặt release gate có ý nghĩa vận hành,
- đề xuất 5 edge cases cần đưa vào reference dataset,
- và lập một pilot plan có thời gian + chi phí sơ bộ.

---

## 11. Bạn nên làm gì ở case 1?

Đây là case scaffold cao, nên cách làm tốt nhất là:

1. Đọc ví dụ full luồng trước để hiểu “một output tốt trông như thế nào”.
2. So mock outcome với ví dụ full luồng để thấy lỗi đang nằm ở đâu.
3. Nhìn từ UI để suy ra các field tối thiểu hệ thống phải có.
4. Điền `Eval Decision Map` trước, rồi mới quay lại viết các kiểm tra tự động và gate.

Case này thường **không bắt buộc phải có domain expert chuyên sâu**. Nếu chọn không cần expert, bạn vẫn phải giải thích vì sao human review vận hành là đủ.

---

## 12. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ hệ thống hỗ trợ khách hàng”.
- Ở case này, một `Unit of Work` tốt thường là: **một ticket đi vào -> AI gán nhãn, đánh mức ưu tiên, đề xuất route và cờ escalation**.

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- bạn chọn lát cắt nào,
- và vì sao đây là đơn vị đủ nhỏ để eval.

Tôi chọn lát cắt là một ticket support đi vào và AI phải gán category, mức độ khẩn, team phụ trách, và cờ cần người thật xử lý ngay. Đây là đơn vị đủ nhỏ để eval vì nó gắn trực tiếp với quyết định vận hành ở inbox nội bộ, nhưng vẫn đủ lớn để bắt các lỗi ảnh hưởng thật như route sai, thiếu escalation, hoặc vỡ schema.

Nếu AI sai ở unit này, ticket có thể bị đưa sang team không xử lý được, hoặc bị chậm đến mức enterprise customer bị ảnh hưởng công việc. Vì vậy đây là lát cắt vừa đủ nhỏ để đo, nhưng hậu quả của lỗi vẫn rõ ràng và có thể kiểm tra được.

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “AI có triage tốt không?”
- Nếu AI làm sai ở đây, điều gì sẽ khiến khách hàng mất trust hoặc không hoàn thành mục tiêu?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **AI có gắn đúng route và escalation để ticket không bị đi sai hàng xử lý không?**

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- câu hỏi chất lượng bạn chọn,
- và vì sao nếu fail ở đây thì ticket sẽ đi sai hoặc gây mất trust.

AI có gắn đúng route và escalation để ticket không bị đi sai hàng xử lý không, đặc biệt với ticket enterprise có dấu hiệu chặn công việc hoặc liên quan billing. Bắt buộc là category phải hợp lý, urgency phải không bị hạ thấp sai, và ticket high-risk phải được đẩy cho người thật.

Nếu fail ở đây, ticket sẽ vào sai queue hoặc bị xem như case bình thường dù đang chặn công việc của khách. Điều đó làm mất trust của support ops và có thể làm trễ xử lý các case cần phản ứng nhanh.

### 3. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- route đúng hàng xử lý,
- trigger escalation nếu cần,
- và chạy eval sau này.

Mẹo lấy từ ví dụ full luồng:

- Hãy nhìn ngược từ UI và mock outcome.
- Field nào không làm thay đổi màn hình, routing hoặc gate thì chưa cần đưa vào.

**Trả lời của bạn:**

Đừng chỉ liệt kê field. Với mỗi field bạn giữ lại, hãy giải thích ngắn vì sao nó cần cho UI, routing, escalation, hoặc eval.

- `ticket_id`: để trace và đối chiếu với hệ thống inbox, phục vụ eval regression.
- `category`: để render nhãn loại yêu cầu và route sang team đúng.
- `urgency`: để quyết định hàng ưu tiên và mức độ khẩn hiển thị trên UI.
- `route_to`: để hệ thống biết ticket đi vào team nào, đây là quyết định vận hành chính.
- `requires_human`: để bật checkpoint người thật khi ticket có rủi ro cao.
- `reason_codes`: để giải thích ngắn vì sao AI ra quyết định, phục vụ human review và LLM judge.
- `confidence`: để chọn case nào cần review thêm, nhưng không được dùng một mình làm gate.
- `queue_priority`: để đẩy ticket lên ưu tiên cao hoặc giữ ở hàng bình thường.

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng chép lại toàn bộ business rules hay toàn bộ UI. Hãy chọn ra những thành phần quan trọng nhất, bám vào:

- `Output Contract` bạn đã đề xuất
- quyết định nào thật sự làm thay đổi route, escalation, hoặc safety
- chỗ nào nếu sai sẽ gây hậu quả vận hành rõ ràng

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema và allowed enums | ✓ |  |  |  | Đây là ràng buộc deterministic, chỉ cần code để bắt vỡ schema và giá trị ngoài danh sách cho phép. |
| `ticket_id` và trace mapping | ✓ |  |  |  | Dữ liệu định danh phải khớp tuyệt đối để eval và logging không bị lệch. |
| `category` |  | ✓ | ✓ |  | Có thể chấm bằng LLM theo nghĩa ticket, nhưng human cần cho các case khó/ambiguous để làm gold. |
| `urgency` |  | ✓ | ✓ |  | Mức khẩn có ngữ nghĩa vận hành, code không đọc được ngữ cảnh như “blocking work”. |
| `route_to` | ✓ |  | ✓ |  | Một số route có thể kiểm bằng rule, nhưng các case mơ hồ cần human xác nhận để tránh nhầm queue. |
| `requires_human` / escalation | ✓ |  | ✓ |  | Đây là invariant an toàn, code phải bắt rule bắt buộc; human review các case rìa để tránh bỏ sót escalation. |
| `reason_codes` |  | ✓ | ✓ |  | Cần đọc hiểu nội dung ticket để biết lý do có bám sát input hay không. |
| `confidence` | ✓ |  |  |  | Có thể kiểm format/range bằng code; không nên để confidence tự quyết định chất lượng. |

Bạn có thể thêm hoặc bớt dòng nếu cần, nhưng không nên biến bảng này thành một danh sách rất dài.

Không chấp nhận bảng chỉ tick `Yes/No`. Cột `Lý do` phải nêu được vì sao bạn chọn nguồn chấm đó.

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì ticket sẽ đi sai hàng, thiếu escalation, hoặc vỡ schema.

Mỗi ý nên viết theo dạng:

- Kiểm tra: [rule]
  Vì sao nên giao cho code:

- Kiểm tra: Output phải parse được đúng schema JSON và không thiếu field bắt buộc.
    Vì sao nên giao cho code: Đây là validation xác định, chỉ cần schema checker là bắt được.
- Kiểm tra: `category` phải thuộc allowed enum.
    Vì sao nên giao cho code: Enum là rule cứng, không cần judgment ngữ nghĩa.
- Kiểm tra: `urgency` phải thuộc `low`, `medium`, `high`, `critical`.
    Vì sao nên giao cho code: Đây là kiểm tra format và giá trị hợp lệ.
- Kiểm tra: Nếu `customer_tier = enterprise` và `urgency` là `high` hoặc `critical` thì `requires_human = true`.
    Vì sao nên giao cho code: Đây là invariant business rule rõ ràng.
- Kiểm tra: Ticket có signal billing không được route sang `product_team`.
    Vì sao nên giao cho code: Mapping cấm là rule deterministic.
- Kiểm tra: Ticket có dấu hiệu `blocking work`, `locked out`, hoặc `account disabled` không được bị gắn `low`.
    Vì sao nên giao cho code: Có thể bắt bằng rule / keyword / assertion trực tiếp.
- Kiểm tra: `confidence` phải nằm trong khoảng từ 0 đến 1.
    Vì sao nên giao cho code: Đây là numeric bound check.
- Kiểm tra: `reason_codes` phải được suy ra từ nội dung ticket và không bịa thêm thực thể không có trong input.
    Vì sao nên giao cho code: Có thể đối chiếu token / entity source ở trace hoặc rule-based extraction.
- Kiểm tra: Ticket enterprise high-risk phải được đưa vào hàng ưu tiên cao.
    Vì sao nên giao cho code: Đây là mapping state / queue có thể assert được.
- Kiểm tra: Nếu `route_to` là `technical_support` thì không được gắn đồng thời route billing-only.
    Vì sao nên giao cho code: Có thể kiểm tra xung đột route bằng logic đơn giản.

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu nghĩa của ticket hoặc mức độ hợp lý của lý do tóm tắt.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

- Tiêu chí: Category có phản ánh đúng bản chất vấn đề của ticket không.
    Vì sao code không bắt tốt: Cần đọc hiểu ngữ cảnh toàn ticket thay vì chỉ keyword.
- Tiêu chí: Urgency có tương xứng với mức độ blocking và mức độ ảnh hưởng đến khách enterprise không.
    Vì sao code không bắt tốt: Mức khẩn phụ thuộc vào judgment vận hành.
- Tiêu chí: Route có đi đúng team xử lý thực tế không.
    Vì sao code không bắt tốt: Có nhiều case biên giữa technical, billing, và support.
- Tiêu chí: Escalation có hợp lý với ticket enterprise và dấu hiệu chặn công việc không.
    Vì sao code không bắt tốt: Cần hiểu tác động thực tế, không chỉ tìm từ khóa.
- Tiêu chí: Reason summary có bám đúng lý do chính trong ticket và không bịa thêm chi tiết không có trong input không.
    Vì sao code không bắt tốt: Đây là semantic groundedness, khó chấm bằng rule thuần.
- Tiêu chí: Output có đủ ngắn gọn và hữu ích cho nhân sự nội bộ khi đọc inbox không.
    Vì sao code không bắt tốt: Đây là đánh giá chất lượng sử dụng thực tế.
- Tiêu chí: AI có biết giữ độ thận trọng khi ticket thiếu tín hiệu hoặc mơ hồ không.
    Vì sao code không bắt tốt: Cần đọc hiểu hành vi dưới thiếu thông tin.

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

Nhân sự support ops hoặc team lead nội bộ là nhóm cần review, vì họ hiểu routing thực tế và có thể xác nhận ticket nào cần ưu tiên hoặc escalate. Họ nên review các case mơ hồ, các case enterprise high/critical, và các case mà summary nghe hợp lý nhưng route hoặc escalation có vẻ sai.

Không cần domain expert riêng vì đây là bài toán vận hành support, không đụng đến kiến thức y khoa, pháp lý hay các quyết định chuyên môn khó tách bằng quy trình ops. Human review từ team vận hành đủ để xác nhận what-to-route, what-to-escalate, và các case biên.

Nếu chọn **có domain expert**, bạn phải làm thêm 2 phần dưới đây. Nếu **không cần domain expert**, hãy ghi `Không áp dụng` và giải thích 1 câu.

#### 7A. Màn hình cho Domain Expert (ASCII)

Mock một màn hình review cho expert.

Expert cần thấy tối thiểu:

- AI đã route hoặc gắn nhãn gì,
- dấu hiệu hoặc evidence nào khiến case bị đẩy sang expert,
- expert có thể duyệt / sửa / escalation ở đâu.

**Trả lời của bạn:**

```text
Không áp dụng.
```

#### 7B. Tiêu chí review của Domain Expert

Không áp dụng. Case này không cần domain expert vì các quyết định chính là triage vận hành, có thể được xác nhận bởi support ops và team lead theo rule và rubric đã thống nhất.

### 8. Release Gate

Tôi đề xuất gate theo hai lớp. Lớp cứng: fail ngay nếu schema vỡ, enum sai, `requires_human` sai với enterprise high/critical, hoặc billing bị route sang `product_team`. Lớp chất lượng: trên tập high-risk phải đạt 100% recall cho escalation bắt buộc, còn route đúng team và urgency đúng phải đạt tối thiểu 95% trên bộ validation đã cân bằng mẫu.

Human review bắt buộc cho các case `unknown`, các case có dấu hiệu blocked work nhưng thiếu thông tin, và các case có confidence thấp nhưng lại chạm business rule cứng. Không cho release nếu còn false negative ở các case enterprise bị chặn công việc hoặc còn route sai lặp lại ở nhóm billing/technical biên.

### 9. Kế hoạch chạy thử và dự toán chi phí

Tôi sẽ pilot với 72 cases, trong đó trộn đủ happy path, ambiguous, missing info, high-risk escalation, và regression. Mỗi case chạy qua 40 lượt lặp lại để so prompt/model changes và rubric stability, tổng cộng 2,880 lượt eval.

Phần máy tính tôi tính bằng OpenAI API Pricing cho GPT-5.4 mini, với giá input $0.75 / 1M tokens và output $4.50 / 1M tokens. Giả định trung bình mỗi lượt judge tiêu thụ 700 input tokens và 120 output tokens, tổng token cho 2,880 lượt là 2.016M input và 0.346M output, tương đương khoảng $1.51 + $1.56 = $3.07 API spend.

Phần người: PM / thiết kế eval khoảng 8 giờ, support ops khoảng 10 giờ, human review khoảng 14 giờ. Tổng effort khoảng 32 giờ, đủ để chốt rubric ban đầu, đo lỗi routing/escalation, và chứng minh hệ thống có thể pilot trước khi mở rộng.

Mục tiêu của pilot này là chứng minh 3 việc: schema và rule cứng không vỡ, escalation bắt buộc không bị bỏ sót, và các route phổ biến đã ổn định đủ để đề xuất triển khai thử cho team vận hành.
```
