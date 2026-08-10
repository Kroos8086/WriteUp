# CTF Write-up: Click Here For Free Bricks 🧱

**Platform**: UMassCTF  
**Category**: Forensics / Wireshark  
**Difficulty**: Medium  
**Points**: 100  
**Solves**: 173  
**Author**: non_gonzo  

---

## 📋 Đề bài

> Hey! A man was caught with malware on his PC in Lego City. Luckily, we were able to get a packet capture of his device during the download. Help Lego City Police figure out the source of this malicious download.

**Flag format**: `UMASS{[String]_[Sha256 Hash]}`

**Ví dụ**: Nếu hash là `2cf24dba...`, tên virus trên VirusTotal (tab Details) sẽ có dạng:  
`ForensicsChallenge_2cf24dba...`

---

## 🧠 Phân tích đề bài

Từ đề bài, ta suy ra:
- Có file `.pcap` hoặc `.pcapng` cần phân tích bằng **Wireshark**
- Trong quá trình download, có một file malware được tải về
- Cần tìm **SHA256 của file malware thật** rồi tra trên **VirusTotal**
- Tên virus trên tab **Details** của VirusTotal sẽ có format `[String]_[SHA256]`

---

## 🔍 Bước 1: Phân tích PCAP bằng Wireshark

Mở file PCAP bằng Wireshark, sau đó export toàn bộ HTTP object:

> **File** → **Export Objects** → **HTTP**

### Kết quả HTTP Object List:

| Packet | Hostname | Content Type | Size | Filename |
|---|---|---|---|---|
| 63 | 156.234.52.16 | image/jpeg | 51 KB | `fungame.jpg` |
| 1112 | 156.234.52.16 | image/jpeg | 11 KB | `cooldog.jpeg` |
| **2966** | 156.234.52.16 | **text/x-python** | 614 B | **`installer.py`** |
| 3610 | 156.234.52.16 | image/jpeg | 13 KB | `literallyme.jpeg` |
| **4584** | 156.234.52.16 | **application/octet-stream** | 13 KB | **`launcher`** |

### 🚨 Nhận định:
- Các file `.jpg` / `.jpeg` → bình thường
- **`installer.py`** → Python script từ server lạ → rất đáng ngờ
- **`launcher`** → binary không có extension, content-type `octet-stream` → rất đáng ngờ

→ **Export cả 2 file `installer.py` và `launcher` ra để phân tích.**

---

## 🔍 Bước 2: Đọc nội dung `installer.py`

```bash
cat installer.py
```

```python
import subprocess
import subprocess
import hashlib
import nacl.secret

def fix_error():
    seed = "38093248092rsjrwedoaw3"
    key = hashlib.sha256(seed.encode()).digest()
    box = nacl.secret.SecretBox(key)
    with open("./launcher", "rb") as f:
        data = f.read()
    decrypted = box.decrypt(data)
    with open("./launcher", "wb") as f:
        f.write(decrypted)

print("Hello World")

try:
    fix_error()
    print("Installed Correctly")
    result = subprocess.run(["ping", "-c", "2", "76.54.32.144"])
    print(result)
except Exception as e:
    print(f"Installation failed, please try again {e}")
```

### 🚨 Phân tích script:

| Dòng code | Ý nghĩa |
|---|---|
| `seed = "38093248092rsjrwedoaw3"` | Hardcoded seed để tạo khóa giải mã |
| `hashlib.sha256(seed.encode()).digest()` | Tạo key 32 bytes từ SHA256 của seed |
| `nacl.secret.SecretBox(key)` | Dùng **NaCl symmetric encryption** (XSalsa20-Poly1305) |
| `box.decrypt(data)` | **Giải mã file `launcher`** |
| `subprocess.run(["ping", "-c", "2", "76.54.32.144"])` | Beacon về C2 server `76.54.32.144` |

> **⚠️ Key insight**: File `launcher` bị **MÃ HÓA**!  
> Hash mà Wireshark thấy là hash của file **encrypted** → không phải malware thật.  
> Cần decrypt trước mới ra file thực sự.

---

## 🔍 Bước 3: Giải mã file `launcher`

Viết script Python để decrypt file `launcher`:

```bash
cd ~/Downloads
nano decrypt.py
```

```python
import hashlib
import nacl.secret

# Lấy key từ seed hardcoded trong installer.py
seed = "38093248092rsjrwedoaw3"
key = hashlib.sha256(seed.encode()).digest()
box = nacl.secret.SecretBox(key)

# Đọc file launcher đã được mã hóa
with open("./launcher", "rb") as f:
    data = f.read()

# Giải mã
decrypted = box.decrypt(data)

# Lưu file đã giải mã
with open("./launcher_decrypted", "wb") as f:
    f.write(decrypted)

# Tính SHA256 của file thật
sha256 = hashlib.sha256(decrypted).hexdigest()
print(f"SHA256 of decrypted launcher: {sha256}")
```

```bash
pip install pynacl
python3 decrypt.py
```

### Kết quả:
```
SHA256 of decrypted launcher: e7a09064fc40dd4e5dd2e14aa8dad89b228ef1b1fdb3288e4ef04b0bd497ccae
```

---

## 🔍 Bước 4: Tra cứu SHA256 trên VirusTotal

Truy cập [https://www.virustotal.com](https://www.virustotal.com) → paste SHA256 vào ô search:

```
e7a09064fc40dd4e5dd2e14aa8dad89b228ef1b1fdb3288e4ef04b0bd497ccae
```

### Tab Details — Basic Properties:

| Trường | Giá trị |
|---|---|
| SHA-256 | `e7a09064fc40dd4e5dd2e14aa8dad89b228ef1b1fdb3288e4ef04b0bd497ccae` |
| File type | **ELF** (Linux executable) |
| Magic | FreeBSD/i386 compact demand paged dynamically linked executable |
| File size | 12.87 KB |
| First Seen | **2007-04-23** (malware cổ điển!) |

### Tab Details — Mục Names (scroll xuống):

Trong danh sách rất nhiều tên, tìm entry theo đúng format `[String]_[SHA256]`:

```
TheZoo_e7a09064fc40dd4e5dd2e14aa8dad89b228ef1b1fdb3288e4ef04b0bd497ccae
```

Ngoài ra còn thấy các tên xác nhận đây là virus đã biết:
- `Virus.Unix.Snoopy.c`
- `UNIX.Snoopy.c`

---

## 🏁 Flag

```
UMASS{TheZoo_e7a09064fc40dd4e5dd2e14aa8dad89b228ef1b1fdb3288e4ef04b0bd497ccae}
```

---

## 📚 Tổng kết kỹ thuật

### Attack Chain của malware:
```
Victim tải → fungame.jpg (mồi nhử)
           → installer.py (dropper/decryptor)
           → launcher (payload mã hóa NaCl)
                ↓
           installer.py decrypt launcher
                ↓
           Beacon về C2: 76.54.32.144
                ↓
           Thực thi ELF payload (Snoopy virus)
```

### Kỹ năng áp dụng:
| Kỹ năng | Chi tiết |
|---|---|
| **Network Forensics** | Phân tích PCAP, Export HTTP Objects |
| **Code Analysis** | Đọc và hiểu Python script malware |
| **Cryptography** | NaCl SecretBox (XSalsa20-Poly1305) |
| **OSINT** | Tra cứu hash trên VirusTotal |

### Tên malware:
- **TheZoo** → Đây là repo GitHub nổi tiếng chứa sample malware ([github.com/ytisf/theZoo](https://github.com/ytisf/theZoo))
- **Snoopy virus** → Unix virus cổ điển từ 2007, lan theo file ELF

---

*Write-up by: CTF Player | UMassCTF | Forensics Category*
