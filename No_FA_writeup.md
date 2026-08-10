# Writeup: Web Exploitation - Two-Factor Authentication Bypass

## 1. Tổng quan bài toán (Challenge Overview)
*   **Mục tiêu:** Đăng nhập vào tài khoản `admin` để lấy Flag.
*   **Dữ liệu cung cấp:** Mã nguồn máy chủ (`app.py`) và một file cơ sở dữ liệu bị rò rỉ (`users.db`).

---

## 2. Phân tích mã nguồn (Source Code Analysis)
Kiểm tra file `app.py`, ta rút ra được các điểm mấu chốt:
1.  **Cấu trúc Flag:** Flag chỉ được hiển thị nếu người dùng đăng nhập thành công với tên `username` là `admin`.
2.  **Cơ chế mật khẩu:** Mật khẩu được lưu trữ trong cơ sở dữ liệu dưới dạng mã băm **SHA-256**.
3.  **Bảo mật 2 lớp (2FA):** Nếu người dùng có cột `two_fa=1` trong database, hệ thống sẽ tạo một mã OTP ngẫu nhiên từ 1000-9999 và lưu vào `session['otp_secret']`.
4.  **Phiên làm việc (Session):** Ứng dụng sử dụng Flask Session mặc định. Điều này có nghĩa là dữ liệu session được lưu trữ ở phía Client (trình duyệt) dưới dạng Cookie đã được ký tên (signed) nhưng không được mã hóa.

---

## 3. Khai thác (Exploitation)

### Bước 1: Khai thác cơ sở dữ liệu (Database Reconnaissance)
Sử dụng công cụ SQLite (như `sqliteonline.com` hoặc `DB Browser`), chúng ta mở file `users.db` và xem bảng `users`. Kết quả thu được thông tin tài khoản admin:
*   **Username:** `admin`
*   **Password Hash (SHA-256):** `c20fa16907343eef642d...` (ví dụ)
*   **Two_fa:** `1` (Xác nhận admin có bật bảo mật 2 lớp).

### Bước 2: Bẻ khóa mật khẩu (Password Cracking)
Sử dụng các công cụ như Hashcat hoặc các trang web giải mã hash online (như *CrackStation*), chúng ta tìm được mật khẩu gốc từ chuỗi hash trên.
*   **Mật khẩu tìm được:** `apple@123` (ví dụ từ quá trình giải bài).

### Bước 3: Vượt qua 2FA (2FA Bypass)
Khi đăng nhập bằng tài khoản `admin` và mật khẩu vừa tìm được, hệ thống chuyển hướng chúng ta đến trang `/two_fa` yêu cầu nhập mã OTP gồm 4 chữ số.

Thay vì tấn công vét cạn (Brute Force) 9000 mã số trong vòng 120 giây, chúng ta nhận thấy lỗ hổng nằm ở cách Flask lưu trữ Session. Vì mã OTP được lưu trực tiếp vào Session Cookie (`otp_secret`), chúng ta có thể đọc nó trực tiếp từ trình duyệt:

1.  Mở trình duyệt (F12), tìm đến mục **Cookies**, lấy giá trị của cookie `session`.
2.  Giá trị này thường có định dạng `.eJw...` (đã được nén bằng zlib và mã hóa Base64).
3.  Sử dụng script Python để giải mã và giải nén:
    ```python
    import zlib, base64
    # Giải mã phần payload của Flask Session
    # Bỏ dấu chấm ở đầu nếu có
    session_cookie = "eJwty0sKg..." 
    data = base64.urlsafe_b64decode(session_cookie + "===")
    print(zlib.decompress(data).decode())
    ```
4.  Kết quả trả về một chuỗi JSON chứa giá trị **`"otp_secret": "XXXX"`**.

### Bước 4: Lấy Flag
Nhập mã số vừa tìm được từ Session Cookie vào trang web. Chúc mừng! Bạn đã đăng nhập thành công với quyền `admin` và Flag sẽ hiển thị trên trang chủ.

---

## 4. Lỗ hổng bảo mật & Bài học rút ra
*   **Lỗ hổng:** **Insecure Session Management** (Quản lý phiên không an toàn).
*   **Bài học:** Không bao giờ lưu trữ các dữ liệu nhạy cảm (như mật khẩu, mã OTP, bí mật hệ thống) vào Cookie phía Client, ngay cả khi nó đã được ký tên. Các dữ liệu này cần được lưu trữ phía máy chủ (Server-side Session).
