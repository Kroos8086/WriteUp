# WriteUp: NAND Simulator

> **CTF:** PicoCTF
> **Challenge:** NAND Simulator
> **Category:** Web / Logic
> **Difficulty:** Medium
> **URL:** `http://activist-birds.picoctf.net:64584`

---

## Mô tả bài

Challenge cung cấp một web app mô phỏng mạch logic NAND Gate. Người dùng có thể:

- Kéo thả các node (Input, NAND, Output) để tạo mạch điện
- Kết nối các node với nhau
- Submit mạch lên server để kiểm tra

Mục tiêu là thiết kế mạch đúng để server trả về flag.

---

## Phân tích Source Code

### Cấu trúc thư mục

```
server/
├── index.js          # Express server + endpoints
├── cpu.js            # CPU Verilog simulator (WASM)
├── utils.js          # Serialize circuit + bit manipulation
├── programs/
│   ├── nand_checker.bin  # Chương trình chạy trong CPU
│   └── flag.bin          # Chương trình cho admin endpoint
└── public/
    └── index.html    # Frontend UI
```

---

### Bước 1: Phân tích Frontend (`index.html`)

Nhìn vào hàm `resetGame()`:

```javascript
function resetGame() {
    nextNodeId = 5; // Start after output nodes (1-4)

    for (let i = 0; i < 4; i++) {
        createNode(x, y, value, 'input'); // Tạo 4 input nodes
    }
    createOutputNodes(); // Tạo 4 output nodes
}
```

Và hàm `createNode()`:

```javascript
function createNode(x, y, value, type) {
    if (type === 'output') {
        node.dataset.nodeId = (outputNodes.length + 1).toString();
        // → Output nodes nhận ID: 1, 2, 3, 4
    } else {
        node.dataset.nodeId = nextNodeId.toString();
        nextNodeId++;
        // → Input nodes nhận ID: 5, 6, 7, 8
    }
}
```

**Kết luận về Node ID:**


| Node     | Type   | ID |
| -------- | ------ | -- |
| Input 0  | input  | 5  |
| Input 1  | input  | 6  |
| Input 2  | input  | 7  |
| Input 3  | input  | 8  |
| Output 0 | output | 1  |
| Output 1 | output | 2  |
| Output 2 | output | 3  |
| Output 3 | output | 4  |

Khi người dùng nhấn **Submit**, frontend gửi request:

```javascript
async function submitCircuit() {
    const circuit = [];
    // ... build circuit array ...
    const response = await fetch('/check', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ circuit })
    });
    const result = await response.json();
    if (result.flag) {
        alert('Congratulations! Flag: ' + result.flag);
    }
}
```

---

### Bước 2: Phân tích Server (`index.js`)

Endpoint `/check` xử lý như sau:

```javascript
app.post('/check', async (req, res) => {
    // Validate: tất cả input1, input2, output phải là integer > 0 và <= 0xFFFF
    if (!Array.isArray(circuit) || 
        !circuit.every(entry => checkInt(entry?.input1) && 
                                checkInt(entry?.input2) && 
                                checkInt(entry?.output))) {
        return res.status(400).end();
    }

    // Tạo 4 input ngẫu nhiên (mỗi cái là 0x0000 hoặc 0xFFFF)
    const inputState = new Uint16Array(4);
    for (let i = 0; i < 4; i++) {
        inputState[i] = Math.random() < 0.5 ? 0x0000 : 0xffff;
    }

    // Output mong muốn = INVERSE của input
    const outputState = new Uint16Array(4);
    for (let i = 0; i < 4; i++) {
        outputState[i] = inputState[i] === 0xffff ? 0x0000 : 0xffff;
    }

    const serialized = serializeCircuit(circuit, program, inputState, outputState);
    doRun(res, serialized);
});
```

**→ Bài toán:** Thiết kế mạch NAND thực hiện **phép NOT cho 4 đầu vào song song**.

Hàm `doRun()` trả về flag khi:

```javascript
if (result === 0x1337) {
    resp += FLAG1 + "\n";  // ← Đây là flag cần lấy
}
```

---

### Bước 3: Layout bộ nhớ (`utils.js`)


| Địa chỉ | Nội dung                                     |
| ---------- | --------------------------------------------- |
| `0x0000`   | `nand_checker.bin` (chương trình CPU)      |
| `0x1000`   | Expected output (4 giá trị Uint16)          |
| `0x2000`   | Input ngẫu nhiên (4 giá trị Uint16)       |
| `0x3000`   | Circuit người dùng (3 × Uint16 mỗi gate) |

---

## Kiến thức cần có: Mạch số cơ bản

### Phép NOT bằng cổng NAND

Cổng NAND là cổng logic **đa năng** — có thể tạo ra mọi phép logic từ NAND. Riêng với NOT:

```
NOT(A) = NAND(A, A)
```

**Chứng minh:**

```
NAND(A, A) = NOT(A AND A)
           = NOT(A)        ✓
```

Bảng chân lý:


| A | NAND(A, A) | NOT(A) |
| - | ---------- | ------ |
| 0 | 1          | 1      |
| 1 | 0          | 0      |

---

## Exploit

### Thiết kế mạch

Vì server muốn `output = NOT(input)` cho 4 bit song song, ta cần 4 gate NAND:

```
Input[0] (ID=5) ──┬─→ NAND ──→ Output[1] (ID=1)
                  └──┘

Input[1] (ID=6) ──┬─→ NAND ──→ Output[2] (ID=2)
                  └──┘

Input[2] (ID=7) ──┬─→ NAND ──→ Output[3] (ID=3)
                  └──┘

Input[3] (ID=8) ──┬─→ NAND ──→ Output[4] (ID=4)
                  └──┘
```

### Script khai thác (`Script.py`)

```python
import requests

url = "http://activist-birds.picoctf.net:64584/check"

# NAND(input, input) = NOT(input)
# Input nodes: ID 5,6,7,8 | Output nodes: ID 1,2,3,4
circuit = [
    {"input1": 5, "input2": 5, "output": 1},
    {"input1": 6, "input2": 6, "output": 2},
    {"input1": 7, "input2": 7, "output": 3},
    {"input1": 8, "input2": 8, "output": 4}
]

r = requests.post(url, json={"circuit": circuit})
print(r.json())
```

### Kết quả

```json
{
  "status": "success",
  "flag": "picoCTF{...}"
}
```

---

## Bài học rút ra

1. **Không cần dùng UI** — Luôn đọc source để gửi request thẳng vào endpoint
2. **Đọc server-side trước** — Hiểu server muốn gì trước khi lo frontend
3. **Kiến thức nền tảng quan trọng** — Biết `NOT(A) = NAND(A,A)` là chìa khóa
4. **Node ID mapping** — Cần truy ngược code để biết ID thực tế của từng node

---

## Bonus: Bug trong Frontend

```javascript
// GOALS chỉ có key "flip", nhưng code lại gọi GOALS.xor → undefined
const GOALS = { flip: { description: 'Flip the outputs!' } };

function updateGoalDisplay() {
    goalDisplay.textContent = GOALS.xor.description; // ← TypeError!
}
```

Bug này làm UI bị lỗi nhưng **không ảnh hưởng** đến exploit vì ta bypass hoàn toàn frontend.

*
