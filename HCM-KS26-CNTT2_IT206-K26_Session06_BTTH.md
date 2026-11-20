

## 1. Bối cảnh và Thiết kế Giải pháp

### Bài toán thực tế

Trong các lớp học hoặc khóa học tự nghiên cứu trực tuyến, sinh viên thường được yêu cầu tự tóm tắt lại bài học theo **kỹ thuật Feynman** (giải thích kiến thức phức tạp bằng ngôn ngữ đời thường dễ hiểu). Tuy nhiên, nếu dùng một prompt duy nhất để AI vừa kiểm tra lỗi, vừa giảng lại bài, vừa chấm điểm thì prompt sẽ rất dài, tốn nhiều token và dễ bị "ngáo" khi gặp sinh viên nộp bài trống, spam ký tự rác hoặc copy-paste nguyên văn tài liệu.

### Kiến trúc chuỗi Prompt khép kín (Prompt Chaining 3 bước)

Để tối ưu token và không bị crash luồng xử lý khi gặp dữ liệu lạ, mình tách hệ thống thành **3 Prompt chạy tuần tự**, giao tiếp với nhau bằng chuỗi **JSON rút gọn** (chỉ giữ lại các trường dữ liệu thật sự cần thiết):

1. **Prompt 1 (Bộ lọc dữ liệu dị biệt):** Kiểm tra nhanh bài nộp của sinh viên. Nếu bài trống, toàn ký tự đặc biệt, quá ngắn hoặc copy nguyên văn tài liệu gốc thì bật cờ `"valid": false` để chặn lại ngay từ vòng gửi xe, tiết kiệm token cho các bước sau.

2. **Prompt 2 (Gia sư Feynman & Phân tích lỗ hổng):** Nhận dữ liệu sạch từ Bước 1, đối chiếu bài làm với kiến thức gốc để tìm xem sinh viên hiểu đúng chỗ nào, thiếu ý nào, đồng thời chuẩn bị sẵn lời giảng giải theo phong cách Feynman.

3. **Prompt 3 (Chấm điểm Rubric & Tổng hợp báo cáo):** Nhận phân tích từ Bước 2 để chấm điểm khách quan theo thang Rubric và xuất báo cáo cuối cùng sạch lỗi cho sinh viên.

## 2. Hệ thống Prompt Thực Thi (Tối ưu Token)

### Prompt 1: Kiểm tra tính hợp lệ & Chặn dữ liệu dị biệt

```
Nhiệm vụ: Kiểm tra bài giải thích Feynman của sinh viên trước khi chấm.
Dữ liệu:
- Tài liệu gốc: "{{TAI_LIEU_GOC}}"
- Bài nộp SV: "{{BAI_NOP_SV}}"

Quy tắc bắt lỗi:
1. Nếu Bài nộp SV rỗng, chứa toàn ký tự rác/số vô nghĩa, hoặc dưới 15 từ -> "valid": false, "err": "Bài nộp trống hoặc quá ngắn".
2. Nếu Bài nộp SV giống trên 80% câu chữ Tài liệu gốc (copy-paste) -> "valid": false, "err": "Lỗi học vẹt: Bạn đang chép lại tài liệu gốc, hãy tự diễn đạt bằng lời của mình".
3. Nếu nội dung không liên quan Tài liệu gốc -> "valid": false, "err": "Bài làm lạc đề".
4. Các trường hợp đạt -> "valid": true, "err": "".

Chỉ xuất JSON gọn (không giải thích):
{"valid": true/false, "err": "..."}

```

### Prompt 2: Gia sư Feynman phân tích mức độ hiểu bài

```
Nhiệm vụ: Đóng vai gia sư Feynman phân tích bài làm của sinh viên.
Nếu "valid" ở Bước 1 là false -> Trả về ngay: {"status": "STOP"}.

Nếu "valid" là true, hãy đối chiếu Bài nộp SV với Tài liệu gốc và xuất JSON gọn gồm:
- "y_dung": Các ý cốt lõi sinh viên đã nắm đúng.
- "lo_hong": Ý quan trọng bị thiếu hoặc giải thích còn mơ hồ.
- "feynman": Lời giảng lại khái niệm thật đơn giản (dưới 60 từ) kèm 1 ví dụ đời thường.

Dữ liệu đầu vào:
- Kết quả Bước 1: {{JSON_BUOC_1}}
- Tài liệu gốc: "{{TAI_LIEU_GOC}}"
- Bài nộp SV: "{{BAI_NOP_SV}}"

Chỉ xuất JSON gọn:
{"status": "OK", "y_dung": "...", "lo_hong": "...", "feynman": "..."}

```

### Prompt 3: Chấm điểm Rubric & Xuất báo cáo phản hồi

```
Nhiệm vụ: Dựa vào JSON Bước 2 để xuất phản hồi cuối cùng cho sinh viên.
- Nếu "status" == "STOP": Xuất thông báo lỗi từ mục "err" ở Bước 1 và dừng lại.
- Nếu "status" == "OK": Chấm điểm theo Rubric (Hiểu đúng bản chất: 5đ | Diễn đạt đơn giản dễ hiểu: 3đ | Ví dụ minh họa: 2đ) và trình bày gọn gàng theo đúng mẫu sau:

### KẾT QUẢ ĐÁNH GIÁ FEYNMAN
1. **Góc giải thích của Gia sư:** (Lấy nội dung từ "feynman")
2. **Bảng điểm Rubric:** (Kẻ bảng Markdown 3 cột: Tiêu chí | Điểm | Nhận xét dựa trên "y_dung" và "lo_hong")
3. **Gợi ý sửa bài:** (Đặt 1 câu hỏi ngắn giúp sinh viên tự lấp đầy phần "lo_hong")

Dữ liệu đầu vào:
- Bước 1: {{JSON_BUOC_1}}
- Bước 2: {{JSON_BUOC_2}}

```

## 3. Kết quả chạy thử thực tế (Đầu ra sạch lỗi)

Mình chạy thử chuỗi 3 prompt trên với **2 tình huống thực tế** (1 tình huống dính bẫy dữ liệu dị biệt và 1 tình huống bài làm hợp lệ) để kiểm tra tính ổn định.

### Trường hợp 1: Bẫy dữ liệu dị biệt (Sinh viên copy-paste nguyên văn tài liệu)

* **Input đưa vào:**

  * *Tài liệu gốc:* `"Mảng (Array) là một cấu trúc dữ liệu dùng để lưu trữ một tập hợp các phần tử có cùng kiểu dữ liệu nằm liền kề nhau trong bộ nhớ."`

  * *Bài nộp SV:* `"Mảng (Array) là một cấu trúc dữ liệu dùng để lưu trữ một tập hợp các phần tử có cùng kiểu dữ liệu nằm liền kề nhau trong bộ nhớ."`

* **Kết quả Prompt 1:**

  ```json
  {"valid": false, "err": "Lỗi học vẹt: Bạn đang chép lại tài liệu gốc, hãy tự diễn đạt bằng lời của mình"}
  ```

* **Kết quả Prompt 2:**

  ```json
  {"status": "STOP"}
  ```

* **Kết quả Prompt 3 (Đầu ra cuối cùng):**

  > Lỗi học vẹt: Bạn đang chép lại tài liệu gốc, hãy tự diễn đạt bằng lời của mình.
  > *(Hệ thống dừng an toàn, không tốn token giảng bài hay chấm điểm bừa, không bị crash).*

### Trường hợp 2: Dữ liệu hợp lệ (Sinh viên tự diễn đạt nhưng thiếu ví dụ)

* **Input đưa vào:**

  * *Tài liệu gốc:* `"Mảng (Array) là một cấu trúc dữ liệu dùng để lưu trữ một tập hợp các phần tử có cùng kiểu dữ liệu nằm liền kề nhau trong bộ nhớ."`

  * *Bài nộp SV:* `"Theo mình hiểu thì mảng giống như một dãy tủ đựng đồ xếp sát cạnh nhau ở siêu thị. Tất cả các ngăn tủ trong dãy đó chỉ được dùng để đựng cùng một loại đồ giống nhau và được đánh số thứ tự từ 0 để dễ tìm lấy đồ."`

* **Kết quả Prompt 1:**

  ```json
  {"valid": true, "err": ""}
  ```

* **Kết quả Prompt 2:**

  ```json
  {
    "status": "OK",
    "y_dung": "Nắm đúng bản chất lưu trữ liền kề, cùng kiểu dữ liệu và truy xuất qua chỉ số (index từ 0).",
    "lo_hong": "Chưa nhắc tới việc kích thước mảng thường cố định khi khai báo và chưa có ví dụ code cụ thể.",
    "feynman": "Mảng giống như một vỉ thuốc có 10 ô nằm liền nhau, mỗi ô chỉ chứa đúng 1 loại viên thuốc giống hệt nhau. Muốn lấy viên thứ 3, bạn chỉ cần đếm đúng vị trí ô đó mà không cần xé tung cả vỉ."
  }
  ```

* **Kết quả Prompt 3 (Đầu ra cuối cùng):**

  > ### KẾT QUẢ ĐÁNH GIÁ FEYNMAN
  >
  > 1. **Góc giải thích của Gia sư:** Mảng giống như một vỉ thuốc có 10 ô nằm liền nhau, mỗi ô chỉ chứa đúng 1 loại viên thuốc giống hệt nhau. Muốn lấy viên thứ 3, bạn chỉ cần đếm đúng vị trí ô đó mà không cần xé tung cả vỉ.
  >
  > 2. **Bảng điểm Rubric:**
  >
  > | Tiêu chí | Điểm | Nhận xét |
  > | :--- | :---: | :--- |
  > | Hiểu đúng bản chất | 4.5 / 5 | Hiểu rất rõ tính liền kề, cùng kiểu dữ liệu và cách đánh chỉ số từ 0. Chỉ thiếu ý kích thước cố định. |
  > | Diễn đạt đơn giản dễ hiểu | 3 / 3 | Cách dùng từ tự nhiên, thoát hoàn toàn khỏi câu chữ sách giáo khoa. |
  > | Ví dụ minh họa | 1.5 / 2 | Hình ảnh "dãy tủ siêu thị" rất dễ hiểu nhưng chưa có ví dụ dữ liệu lập trình cụ thể (như mảng số nguyên). |
  > | **TỔNG ĐIỂM** | **9.0 / 10** | **Xuất sắc** |
  >
  > 3. **Gợi ý sửa bài:** Nếu dãy tủ siêu thị đó chỉ xây cố định 5 ngăn mà bạn muốn cất thêm món đồ thứ 6 thì chuyện gì sẽ xảy ra với mảng?
