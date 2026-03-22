# gigem.onnx

## Mô tả

Bài cho một file `gigem.onnx`. Không có mô tả gì thêm ngoài hint:

> *HINT: don't fall for the oldest trick in the book*

---

## ONNX là gì?

**ONNX (Open Neural Network Exchange)** là định dạng file chuẩn để lưu trữ mô hình AI. Giống như `.docx` dành cho Word, `.onnx` là định dạng chung cho các mô hình neural network

---

## Phân tích mô hình

vào trang netron để phân tích mô hình

<img width="2559" height="1250" alt="image" src="https://github.com/user-attachments/assets/c323fd79-caa2-4e89-adb6-1ef5a48bff4e" />

Mô hình nhận vào **48 ký tự** (dưới dạng mã ASCII) và trả ra **1 con số (score)**. Score càng cao thì chuỗi đó càng "đúng"

---

## Nguyên lý hoạt động từng tầng

```
input_ids (48 ký tự ASCII)
         │
         ▼
    ┌─────────┐
    │  GATHER │  ← Tra bảng embedding [256×64]
    └─────────┘    Đổi mỗi ký tự → vector 64 số
         │         'g'(103) → [-1.369, +0.196, +1.490, ...]
         │         "'"( 39) → [+0.486, +0.351, +1.008, ...]
         ▼
   ┌───────────┐
   │ TRANSPOSE │  ← Xoay ma trận [48×64] → [64×48]
   └───────────┘    Chuẩn bị đúng chiều cho Conv
         │
         ▼
    ┌────────┐
    │  CONV  │  ← Conv1D với 128 filters, kernel_size=3
    └────────┘    128 "kính lúp" cùng trượt qua chuỗi
         │        Mỗi cái nhìn 3 ký tự liền nhau một lúc:
         │          "the", "and" → pattern tiếng Anh → điểm cao 
         │          "u'l"        → viết tắt hợp lệ  → điểm cao 
         │          "ubl"        → vô nghĩa         → điểm thấp 
         ▼
    ┌──────┐
    │ GELU │  ← Activation: lọc tín hiệu yếu, giữ tín hiệu mạnh
    └──────┘
         │
         ▼
   ┌─────────┐
   │ RESHAPE │  ← Duỗi thẳng thành 6144 số [128 filters × 48 positions]
   └─────────┘
         │
         ▼
    ┌──────┐
    │ GEMM │  ← Linear: 6144 → 64  (tổng hợp lần 1)
    └──────┘
         │
         ▼
    ┌──────┐
    │ GELU │  ← Lọc lần 2
    └──────┘
         │
         ▼
    ┌──────┐
    │ GEMM │  ← Linear: 64 → 1  (ra điểm cuối)
    └──────┘
         │
         ▼
   score (1 con số)
```

**Điểm mấu chốt:** Conv1D nhìn **3 ký tự liền nhau** → các ký tự ảnh hưởng lẫn nhau → không thể tối ưu từng ký tự độc lập.

---

## Cách giải

### Bước 1 — Xác nhận format flag

```python
import onnxruntime as ort
import numpy as np

sess = ort.InferenceSession('gigem.onnx')

def score(s):
    arr = np.array([[ord(c) for c in s]], dtype=np.int64)
    return sess.run(['score'], {'input_ids': arr})[0][0]

# Input phải đúng 48 ký tự
print(score("gigem{" + "a"*41 + "}"))  # -28.25 → thấp nhưng đúng format
```

→ Flag có dạng `gigem{...}`, tổng 48 ký tự, phần trong ngoặc dài 41 ký tự.

---

### Bước 2 — Greedy Search (multi-pass)

Vì Conv1D nhìn 3 ký tự một lúc, cần chạy **nhiều vòng** cho đến khi hội tụ:

```python
import onnxruntime as ort
import numpy as np
import string

sess = ort.InferenceSession('gigem.onnx')

def score(s):
    arr = np.array([[ord(c) for c in s]], dtype=np.int64)
    return sess.run(['score'], {'input_ids': arr})[0][0]

charset = string.ascii_lowercase + string.digits + '_'
flag = list("gigem{" + "a"*41 + "}")

for iteration in range(15):
    improved = False
    for pos in range(6, 47):           # duyệt từng vị trí trong ngoặc
        best_char = flag[pos]
        best_s = score("".join(flag))
        for c in charset:              # thử từng ký tự
            flag[pos] = c
            s = score("".join(flag))
            if s > best_s:
                best_s = s
                best_char = c
                improved = True
        flag[pos] = best_char
    if not improved:
        break

print("".join(flag))
# → gigem{just_catch_the_local_max_and_ubll_be_fine}
# → score = 98.4274
```

---

### Bước 3 — Bị bẫy Local Maximum

Kết quả trông có vẻ ổn, nhưng hint đã cảnh báo:

> *"don't fall for the oldest trick in the book"*

**"Oldest trick" = Local Maximum** — điểm cao nhất trong vùng lân cận, nhưng không phải cao nhất toàn cục.

Vấn đề nằm ở **charset quá hẹp**: chỉ dùng `a-z 0-9 _` → bỏ sót dấu `'` (apostrophe).

```
Greedy thử:  a b c ... z  0 1 ... 9  _
             ↑ không có dấu ' ở đây!
```

→ Bị kẹt ở `ubll` (score 98.4) vì không bao giờ "nhìn thấy" `u'll` (score 100.7).

---

### Bước 4 — Thoát Local Max

Thêm dấu `'` vào charset và chạy lại:

```python
import onnxruntime as ort
import numpy as np
import string

sess = ort.InferenceSession('gigem.onnx')

def score(s):
    arr = np.array([[ord(c) for c in s]], dtype=np.int64)
    return sess.run(['score'], {'input_ids': arr})[0][0]

charset = string.ascii_lowercase + string.digits + "_'"  # thêm dấu nháy đơn!
flag = list("gigem{just_catch_the_local_max_and_ubll_be_fine}")

for iteration in range(10):
    improved = False
    for pos in range(6, 47):
        best_char = flag[pos]
        best_s = score("".join(flag))
        for c in charset:
            flag[pos] = c
            s = score("".join(flag))
            if s > best_s:
                best_s = s
                best_char = c
                improved = True
        flag[pos] = best_char
    if not improved:
        break

print("".join(flag))
# → gigem{just_catch_the_local_max_and_u'll_be_fine}
# → score = 100.6753 
```

---

## So sánh kết quả

| Chuỗi | Score | Kết quả |
|---|---|---|
| `gigem{aaa...a}` | -28.25 | Sai hoàn toàn |
| `gigem{just_catch_the_local_max_and_ubll_be_fine}` | 98.43 | Local max  |
| `gigem{just_catch_the_local_max_and_u'll_be_fine}` | 100.68 | Flag đúng  |

---

## Script đầy đủ

```python
import onnxruntime as ort
import numpy as np
import string

sess = ort.InferenceSession('gigem.onnx')

def score(s):
    arr = np.array([[ord(c) for c in s]], dtype=np.int64)
    return sess.run(['score'], {'input_ids': arr})[0][0]

# Charset đầy đủ — quan trọng: phải có dấu nháy đơn!
charset = string.ascii_lowercase + string.digits + "_'"

flag = list("gigem{" + "a"*41 + "}")

for iteration in range(20):
    improved = False
    for pos in range(6, 47):
        best_char = flag[pos]
        best_s = score("".join(flag))
        for c in charset:
            flag[pos] = c
            s = score("".join(flag))
            if s > best_s:
                best_s = s
                best_char = c
                improved = True
        flag[pos] = best_char
    print(f"Iter {iteration+1}: {''.join(flag)} | score={score(''.join(flag)):.4f}")
    if not improved:
        print("Converged!")
        break

print("\nFlag:", "".join(flag))
```

## Bài học

Tên bài, nội dung flag, và hint đều nói cùng một thứ:

```
Tên bài:       gigem.onnx
Nội dung flag: "just catch the local max and u'll be fine"
Hint:          "don't fall for the oldest trick in the book"
```

"Oldest trick in the book" trong optimization chính là **local maximum** — bẫy mà greedy search luôn dễ rơi vào nếu không cẩn thận với charset.

Cách tránh: **luôn dùng đủ printable ASCII** thay vì giới hạn charset tùy tiện.
