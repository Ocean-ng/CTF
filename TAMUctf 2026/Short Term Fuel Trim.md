# TamuCTF — Short Term Fuel Trim

## Nhận diện bài

Tên bài **"Short Term Fuel Trim"** chơi chữ của **STFT** — *Short-Time Fourier Transform* (Biến đổi Fourier thời gian ngắn). Đây chính là hint duy nhất và cũng là toàn bộ lời giải của bài.


## Nguyên lý STFT

Để giải được bài này, cần hiểu STFT hoạt động như thế nào.

Biến đổi Fourier thông thường chỉ cho biết một đoạn âm thanh chứa những tần số nào, nhưng **không cho biết tần số đó xuất hiện lúc nào**. STFT giải quyết vấn đề này bằng cách:

1. **"Băm"** đoạn âm thanh dài thành những khung thời gian rất ngắn (ví dụ mỗi khung dài 0.01 giây).
2. **Áp dụng FFT** lên từng khung nhỏ đó.

Kết quả không còn là một đường sóng 1D nữa, mà là một **ma trận 2 chiều (2D Matrix)**:

| Chiều | Ý nghĩa |
|---|---|
| **Trục X** | Thời gian (các khung cửa sổ) |
| **Trục Y** | Tần số (các dải cao độ) |
| **Giá trị tại (X, Y)** | Một số phức (Complex Number) |

<img width="1402" height="611" alt="image" src="https://github.com/user-attachments/assets/ade353f7-fb5b-461d-a64e-06c5dbb29949" />


**Số phức tại mỗi điểm mang hai thông tin:**
- Phần **biên độ** `abs(z)` → âm thanh **to hay nhỏ** ở tần số đó, tại thời điểm đó
- Phần **pha** `angle(z)` → vị trí của sóng trong chu kỳ

Khi vẽ spectrogram, người ta lấy `abs(Zxx)` để biểu diễn độ sáng tối tại mỗi điểm — phần pha không cần thiết để đọc flag bằng mắt.

---

## Phân tích file

| Thông số | Giá trị | Ý nghĩa |
|---|---|---|
| Shape | `(129, 1380)` | 129 frequency bins × 1380 time frames |
| Dtype | `complex64` | Số phức 64-bit (32-bit real + 32-bit imaginary) |
| nperseg | `256` | Suy ra từ: `129 = nperseg / 2 + 1` → `nperseg = 256` |
| Tổng dòng số liệu | `129 × 1380 = 177,820` | Khớp với số dòng trong file |

> **Tại sao chỉ có 129 frequency bins thay vì 256?**  
> FFT của tín hiệu thực luôn đối xứng, nên chỉ cần giữ `nperseg / 2 + 1 = 129` dải là đủ thông tin. Nửa còn lại là thừa.

---

## Script giải

```python
import numpy as np
from scipy.signal import istft
from scipy.io import wavfile
import matplotlib.pyplot as plt

# Đọc và parse số phức từ file
with open('numbers.txt', 'r') as f:
    lines = f.readlines()

data = []
for line in lines[1:]:  # Bỏ dòng header
    line = line.strip()
    if line:
        c_str = line[1:-1]  # Bỏ dấu ngoặc đơn ()
        data.append(complex(c_str))

# Reshape thành ma trận STFT gốc (129 frequency bins × 1380 time frames)
Zxx = np.array(data, dtype=np.complex64).reshape((129, 1380))

# Vẽ Spectrogram — lấy abs() để bỏ phần pha, chỉ giữ biên độ
plt.figure(figsize=(16, 6))
plt.pcolormesh(np.abs(Zxx), cmap='hot')
plt.title('Spectrogram')
plt.ylabel('Frequency')
plt.xlabel('Time')
plt.savefig('spectrogram_flag.png', dpi=150)

# Khôi phục file âm thanh bằng ISTFT
_, audio_data = istft(Zxx, nperseg=256)
audio_norm = np.int16(audio_data / np.max(np.abs(audio_data)) * 32767)
wavfile.write('audio_flag.wav', 22050, audio_norm)
```

---

## Kết quả

có thể lên trang này để dịch đoạn âm thanh ra file txt: <img width="2554" height="1093" alt="image" src="https://github.com/user-attachments/assets/4dda9187-0be4-43a6-bb64-b1d626fb2cd0" />

<img width="1764" height="81" alt="image" src="https://github.com/user-attachments/assets/21aaae12-adf0-4ade-9181-1a85b227d043" />


---

## Tóm tắt flow

```
numbers.txt (178k dòng số phức)
        │
        ▼
  Parse + Reshape → Ma trận 2D (129 × 1380)
        │
        │
ISTFT (nperseg=256) ► audio_flag.wav       →  Nghe flag bằng tai
```
