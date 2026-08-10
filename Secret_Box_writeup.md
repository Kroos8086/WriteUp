# Writeup: Web Exploitation - SecretBox SQL Injection

## 1. Tổng quan bài toán (Challenge Overview)
*   **Mục tiêu:** Đọc được bí mật (Flag) của tài khoản `admin`.
*   **Công nghệ:** Node.js (Express), PostgreSQL, Knex.js.
*   **Dữ liệu cung cấp:** Mã nguồn ứng dụng và file cấu hình Docker.

---

## 2. Phân tích hệ thống (System Analysis)

### Cấu trúc Cơ sở dữ liệu (`initdb.sql`)
Hệ thống có 3 bảng chính: `users`, `tokens`, và `secrets`.
*   Tài khoản `admin` có ID cố định: `e2a66f7d-2ce6-4861-b4aa-be8e069601cb`.
*   Dữ liệu Flag được lưu trong bảng `secrets` và thuộc sở hữu của `admin`.

### Phân tích mã nguồn (`server.js`)
Lỗ hổng nằm ở chức năng tạo Secret mới (`POST /secrets/create`):
```javascript
// dòng 131-133
const content = req.body.content;
const query = await db.raw(
    `INSERT INTO secrets(owner_id, content) VALUES ('${userId}', '${content}')` 
);
```
**Lỗ hổng:** Ứng dụng sử dụng Template String (`${}`) để chèn trực tiếp dữ liệu `content` từ người dùng vào câu lệnh SQL. Điều này cho phép thực hiện tấn công **SQL Injection**.

---

## 3. Quá trình khai thác (Exploitation)

### Bước 1: Xác định chiến thuật
Mặc dù chúng ta biết mật khẩu của Admin được khởi tạo qua biến môi trường, nhưng việc bẻ khóa mật khẩu này là không khả thi. Thay vào đó, chúng ta sẽ lợi dụng lệnh `INSERT` để "copy" Flag của Admin sang tài khoản của mình.

### Bước 2: Tìm mã định danh người dùng (User ID)
Sau khi đăng ký một tài khoản và đăng nhập, ID của chúng ta sẽ được lưu trong Session. Khi thực hiện một chuỗi SQL lỗi, thông báo lỗi của hệ thống đã tiết lộ ID hiện tại (ví dụ: `66c84368-9213-4883-9ae6-fe72bbc71651`).

### Bước 3: Xây dựng Payload
Chúng ta cần chèn một dòng mới vào bảng `secrets` có `owner_id` là ID của chúng ta, nhưng `content` là dữ liệu lấy từ tài khoản Admin.

**Thử thách:** Một số hệ thống không cho phép dùng dấu comment (`--`) hoặc nhiều câu lệnh cùng lúc. Do đó, chúng ta cần một Payload "mềm dẻo" để khớp với dấu nháy đơn (`'`) còn dư ở cuối câu lệnh gốc.

**Payload cuối cùng:**
```sql
test'), ('66c84368-9213-4883-9ae6-fe72bbc71651', (SELECT content FROM secrets WHERE owner_id = 'e2a66f7d-2ce6-4861-b4aa-be8e069601cb' LIMIT 1) || '
```

**Giải thích cú pháp:**
1.  `test')`: Đóng phần giá trị đầu tiên.
2.  `, ('ID', (SELECT ...))`: Chèn thêm một hàng thứ hai. Trong đó nội dung Flag được lấy thông qua một câu lệnh `SELECT` phụ (Subquery).
3.  `|| '`: Sử dụng phép cộng chuỗi của PostgreSQL để nối kết quả Flag với dấu nháy đơn `'` cuối cùng của mã nguồn, tạo thành một chuỗi hợp lệ.

### Bước 4: Lấy Flag
Sau khi gửi yêu cầu với Payload trên, ứng dụng thực thi câu lệnh SQL thành công. Truy cập lại trang chủ (`/`), hệ thống sẽ liệt kê các Secret thuộc sở hữu của chúng ta, bao gồm cả Flag vừa "chôm" được từ Admin.

---

## 4. Bài học rút ra
*   **Lỗ hổng:** **SQL Injection** do không tham số hóa (parameterize) dữ liệu đầu vào.
*   **Phòng tránh:** Luôn sử dụng thư viện query builder (như Knex) một cách đúng đắn với tham số hóa: 
    `db.raw('INSERT INTO ... VALUES (?, ?)', [userId, content])` 
    hoặc dùng các hàm tương tác database có sẵn thay vì viết SQL thuần.
