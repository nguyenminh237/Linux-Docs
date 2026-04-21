# Giải thích chi tiết đoạn code SAM3 prompt + tiling + ghép mask

Tài liệu này giải thích chi tiết đoạn code bạn gửi: nạp ảnh, chạy SAM3 theo **prompt văn bản** (ví dụ `dog`, `cat`), (tuỳ chọn) áp dụng **tiling** để chia ảnh thành các mảnh nhỏ, ghép mask về ảnh gốc, lọc theo `score`, và hiển thị kết quả overlay.

> Ghi chú: đoạn code này giả định bạn đã chạy các cell **khởi tạo SAM3** trước đó để có sẵn:
>
> - `import torch`
> - `processor = Sam3Processor(model)`
> - GPU CUDA khả dụng (vì dùng `torch.autocast("cuda", dtype=torch.bfloat16)`).

---

## 1) Mục tiêu tổng quát của đoạn code

Đoạn code thực hiện pipeline sau:

1. Đọc ảnh `image` (PIL Image) từ đường dẫn.
2. (Tuỳ chọn) Chia ảnh thành lưới `num_tile_rows × num_tile_cols`.
3. Với mỗi tile:
   - Gọi `processor.set_image(tile_image)` để tạo `inference_state`.
   - Với mỗi prompt (ví dụ `"dog"`, `"cat"`):
     - Gọi `processor.set_text_prompt(state=..., prompt=...)` để lấy các `masks` và `scores`.
     - Với mỗi `mask` trên tile: dịch về đúng tọa độ ảnh gốc và lưu vào `all_masks`.
4. Gộp kết quả (masks + scores) và vẽ overlay lên ảnh gốc.

**Khi bạn đặt `num_tile_rows = num_tile_cols = 1`, pipeline vẫn chạy giống hệt, nhưng chỉ có 1 tile = toàn bộ ảnh**, tức là thực tế **không tiling**.

---

## 2) Các import và hàm tiện ích `show_mask`

### 2.1. Import NumPy

```python
import numpy as np
```

- `numpy` dùng để:
  - tạo mask đầy đủ kích thước ảnh gốc (`np.zeros((H, W), dtype=bool)`),
  - thao tác reshape mask để hiển thị (`mask.reshape(h, w, 1)`),
  - tạo màu RGBA.

### 2.2. Hàm `show_mask(mask, ax, random_color=False)`

```python
def show_mask(mask, ax, random_color=False):
    if random_color:
        color = np.concatenate([np.random.random(3), np.array([0.6])], axis=0)
    else:
        color = np.array([30/255, 144/255, 255/255, 0.6])

    h, w = mask.shape[-2:]
    mask_image = mask.reshape(h, w, 1) * color.reshape(1, 1, -1)
    ax.imshow(mask_image)
```

Ý nghĩa từng phần:

- **Chọn màu**:
  - Nếu `random_color=True`: tạo 3 số ngẫu nhiên (R,G,B) trong `[0,1)` và ghép thêm alpha = `0.6`.
  - Nếu `False`: dùng màu cố định kiểu “DodgerBlue” (xấp xỉ) với alpha = `0.6`.

- **Lấy kích thước mask**:
  - `mask.shape[-2:]` lấy 2 chiều cuối (H, W). Việc dùng `[-2:]` giúp hàm vẫn chạy nếu mask có thêm chiều batch/channel ở phía trước (miễn 2 chiều cuối là H,W).

- **Tạo ảnh màu từ mask**:
  - `mask.reshape(h, w, 1)` biến mask thành (H, W, 1) để broadcast với màu (1,1,4).
  - `color.reshape(1, 1, -1)` thành (1, 1, 4).
  - Nhân lại để tạo `mask_image` (H, W, 4) dạng RGBA.
  - `ax.imshow(mask_image)` vẽ lớp mask chồng lên ảnh đang có trên axis.

**Lưu ý quan trọng**:

- `mask` thường là boolean (`True/False`) hoặc 0/1. Khi nhân với RGBA, `True` sẽ thành 1, `False` thành 0.
- Nếu `mask` là float (ví dụ xác suất), overlay sẽ “mờ theo độ tin cậy” (có thể hữu ích, nhưng khác ý nghĩa nhị phân).

### 2.3. Việc định nghĩa `show_mask` bị lặp

Trong code bạn gửi, `show_mask` được định nghĩa **2 lần** (đầu và gần cuối).

- Python sẽ giữ **phiên bản định nghĩa sau cùng**.
- Lặp như vậy không sai, nhưng không cần thiết và dễ gây nhầm nếu 2 bản khác nhau.

---

## 3) Nạp ảnh và hiển thị ảnh gốc

### 3.1. Import phục vụ nạp ảnh và hiển thị (Colab/Jupyter)

```python
import requests
import numpy as np
from io import BytesIO
from PIL import Image
from IPython.display import display
```

- `requests` + `BytesIO`: dùng để tải ảnh từ URL rồi đọc trực tiếp vào RAM.
- `PIL.Image`: mở ảnh từ file/bytes.
- `display(image)`: hiển thị ảnh trong notebook.

Trong code của bạn, phần tải ảnh từ URL **đang được comment**, nên `requests` và `BytesIO` thực tế không được dùng.

### 3.2. Đọc ảnh từ Drive

```python
image_path = "/content/drive/MyDrive/images/test3.jpg"
image = Image.open(image_path).convert("RGB")

print("Ảnh gốc:")
display(image)
```

- `Image.open(...)`: mở file ảnh.
- `.convert("RGB")`: đảm bảo ảnh có 3 kênh RGB (tránh trường hợp ảnh là RGBA, grayscale…).
- `display(image)`: hiển thị ảnh gốc để bạn quan sát.

---

## 4) Khai báo prompt mục tiêu và các biến tích lũy kết quả

```python
target_objects = ["dog", "cat"]
all_masks = []
all_scores = []
```

- `target_objects`: danh sách prompt văn bản. Mỗi prompt sẽ tạo một lần suy luận.
- `all_masks`: danh sách chứa toàn bộ mask đã ghép về ảnh gốc.
- `all_scores`: danh sách chứa toàn bộ score tương ứng.

**Tại sao cần list tích lũy?**

- Vì mỗi tile có thể tạo nhiều mask.
- Bạn muốn gộp tất cả mask trên mọi tile để vẽ một lần trên ảnh gốc.

**Ràng buộc cần nhớ**:

- Thứ tự `all_masks[i]` phải khớp với `all_scores[i]` khi lọc theo score.
- Trong code của bạn: mỗi lần có `output["masks"]`, bạn append mask vào `all_masks`, rồi `extend(output["scores"])`. Điều này _thường_ khớp nếu:
  - số mask == số score,
  - scores được trả về theo cùng thứ tự masks.

---

## 5) Cấu hình tiling và kích thước ảnh

```python
num_tile_rows = 1
num_tile_cols = 1

W, H = image.size
```

- `num_tile_rows`, `num_tile_cols` quyết định số mảnh theo chiều dọc/ngang.
  - Ví dụ 2×2 nghĩa là 4 tile.
  - 3×3 nghĩa 9 tile.

- `image.size` trả về `(W, H)` theo chuẩn PIL:
  - `W` = width (chiều ngang)
  - `H` = height (chiều dọc)

**Trường hợp bạn đang đặt 1×1**:

- Chỉ có một tile duy nhất:
  - `left=0`, `top=0`, `right=W`, `bottom=H`.
- Do đó kết quả tương đương chạy trực tiếp trên ảnh gốc.

---

## 6) Vòng lặp suy luận (autocast + tile + prompt)

### 6.1. Mixed precision với `torch.autocast`

```python
with torch.autocast("cuda", dtype=torch.bfloat16):
    ...
```

- `autocast` cho phép một số phép toán chạy ở độ chính xác thấp hơn (`bfloat16`) để:
  - giảm dùng VRAM,
  - tăng tốc,
  - đôi khi vẫn giữ chất lượng tốt.

**Điều kiện**:

- Cần CUDA.
- GPU cần hỗ trợ `bfloat16` tốt (đa số GPU mới hỗ trợ; nếu không, có thể chậm hoặc lỗi tùy môi trường).

### 6.2. Lặp qua các tile

```python
for r in range(num_tile_rows):
    for c in range(num_tile_cols):
        ...
```

- `r`: chỉ số hàng tile (0 → num_tile_rows-1)
- `c`: chỉ số cột tile (0 → num_tile_cols-1)

### 6.3. Tính tọa độ crop cho tile

```python
left = c * (W // num_tile_cols)
top = r * (H // num_tile_rows)
right = (c + 1) * (W // num_tile_cols)
bottom = (r + 1) * (H // num_tile_rows)

if c == num_tile_cols - 1:
    right = W
if r == num_tile_rows - 1:
    bottom = H
```

Giải thích:

- Bạn chia ảnh bằng phép chia nguyên `//`.
  - Ví dụ `W=1000`, `num_tile_cols=3` → `W//3 = 333`.
  - 2 tile đầu sẽ có width=333, tile cuối cần “ăn phần dư” để đủ 1000.

- 2 dòng `if` đảm bảo tile cuối cùng không bị thiếu pixel:
  - Nếu là cột cuối: `right=W`
  - Nếu là hàng cuối: `bottom=H`

=> Đây là cách xử lý quan trọng để **tile cuối phủ kín ảnh**, tránh tình trạng mất viền phải hoặc viền dưới.

### 6.4. Crop tile và set ảnh vào processor

```python
tile_image = image.crop((left, top, right, bottom))
inference_state = processor.set_image(tile_image)
```

- `image.crop((left, top, right, bottom))`:
  - Cắt ảnh theo box (left, top, right, bottom).
  - Tọa độ tính theo pixel trên ảnh gốc.

- `processor.set_image(tile_image)`:
  - Thường sẽ chạy encoder ảnh và tạo “state”/cache để suy luận prompt nhanh hơn.
  - Bạn gọi lại cho mỗi tile vì mỗi tile là một ảnh khác.

### 6.5. Lặp qua prompt văn bản

```python
for obj in target_objects:
    output = processor.set_text_prompt(state=inference_state, prompt=obj)
```

- `obj` là prompt (ví dụ `dog`, `cat`).
- `processor.set_text_prompt(...)` trả về một dict `output`.

Trong notebook SAM3 phổ biến, `output` thường có:

- `output["masks"]`: danh sách/stack các mask dự đoán trên tile.
- `output["scores"]`: điểm chất lượng cho từng mask.

---

## 7) Ghép mask từ tile về ảnh gốc (tọa độ toàn cục)

Đây là phần “xương sống” của tiling: **mask tính trên tile phải được đặt đúng vị trí trong ảnh gốc**.

```python
for m in output["masks"]:
    full_mask = np.zeros((H, W), dtype=bool)
    mask_on_tile = m.cpu().numpy()
    full_mask[top:bottom, left:right] = mask_on_tile
    all_masks.append(full_mask)

all_scores.extend(output["scores"])
```

Giải thích chi tiết:

### 7.1. `full_mask = np.zeros((H, W), dtype=bool)`

- Tạo một mask rỗng cho toàn ảnh.
- Kích thước (H, W) khớp với ảnh gốc.

### 7.2. `mask_on_tile = m.cpu().numpy()`

- `m` thường là tensor trên GPU.
- `.cpu()` đưa về CPU.
- `.numpy()` chuyển sang mảng NumPy.

**Lưu ý**:

- Bạn đang gán vào `full_mask` kiểu boolean.
- Nếu `mask_on_tile` không phải boolean (ví dụ float 0..1), NumPy sẽ ép kiểu khi gán.
  - Khi gán float vào bool: giá trị khác 0 sẽ thành `True`.
  - Điều này vẫn “tạo mask nhị phân”, nhưng có thể không như mong muốn nếu cần ngưỡng.

### 7.3. `full_mask[top:bottom, left:right] = mask_on_tile`

- Đây là thao tác “dịch tọa độ”:
  - vùng `[top:bottom, left:right]` trên ảnh gốc chính là tile.
  - bạn chèn mask của tile vào đúng vùng đó.

**Điều kiện bắt buộc để không lỗi shape**:

- `mask_on_tile.shape` phải đúng bằng `(bottom-top, right-left)`.
- Nếu tile bị resize bên trong processor (tùy triển khai), có thể mismatch. Trong notebook của bạn, logic đang giả định mask output có kích thước đúng bằng tile crop.

### 7.4. `all_scores.extend(output["scores"])`

- Nối danh sách score vào `all_scores`.

**Cảnh báo nhỏ về căn chỉnh mask-score**:

- Code của bạn append tất cả `masks` trước, rồi extend `scores`.
- Nếu `output["masks"]` và `output["scores"]` có cùng độ dài và cùng thứ tự, bạn sẽ có:
  - `all_masks[i]` ↔ `all_scores[i]`.

---

## 8) Gán `masks` và `scores` để vẽ

```python
masks = all_masks
scores = all_scores

print(f"✅ Đã áp dụng Tiling ({num_tile_rows}x{num_tile_cols}) và tìm thấy tổng cộng {len(masks)} mask.")
```

- Chỉ là đổi tên biến để giữ tương thích với phần code vẽ phía sau.
- In ra số lượng mask tổng (trên tất cả tile và tất cả prompt).

---

## 9) Trực quan hóa: overlay mask lên ảnh gốc

### 9.1. Tạo 2 khung hình (subplot)

```python
fig, axes = plt.subplots(1, 2, figsize=(20, 10))
axes[0].imshow(image)
axes[0].set_title("Ảnh gốc")

axes[1].imshow(image)
```

- `axes[0]`: chỉ hiển thị ảnh gốc để đối chiếu.
- `axes[1]`: hiển thị ảnh gốc + các mask overlay.

### 9.2. Lọc theo score và vẽ mask

```python
count = 0
for i, mask in enumerate(masks):
    if scores[i] > 0.1:
        show_mask(mask, axes[1], random_color=True)
        count += 1

axes[1].set_title(f"SAM 3 Multi-stage: {count} masks")
plt.show()
```

- Duyệt từng mask, lấy score tương ứng.
- Nếu `scores[i] > 0.1`:
  - vẽ mask lên `axes[1]`.
  - tăng biến đếm `count`.

**Ý nghĩa của ngưỡng 0.1**:

- Đây là ngưỡng khá “thoáng”.
- Ưu điểm: dễ giữ lại các đối tượng nhỏ/yếu.
- Nhược điểm: có thể vẽ rất nhiều mask nhiễu.

**`random_color=True`**:

- Mỗi mask được tô một màu khác nhau, giúp phân biệt các vùng.

---

## 10) Một số lưu ý/pitfall khi chạy thực tế

1. **Thiếu import `torch` trong đoạn code rút gọn**
   - Trong đoạn bạn paste, phần `torch.autocast` sẽ lỗi nếu chưa `import torch` từ trước.

2. **Phụ thuộc biến `processor`**
   - Cần đảm bảo `processor` đã được khởi tạo từ SAM3.

3. **Tiling 1×1 không cải thiện đối tượng nhỏ**
   - 1×1 = không tiling; nếu muốn thấy hiệu quả, thử 2×2 hoặc 3×3.

4. **Đối tượng nằm sát biên tile**
   - Với tiling thật (2×2, 3×3), đối tượng có thể bị “cắt đôi”.
   - Cách khắc phục phổ biến: dùng tile **overlap** và hậu xử lý khử trùng lặp.

5. **Trùng lặp mask khi dùng overlap hoặc đối tượng lớn**
   - Một đối tượng có thể xuất hiện ở nhiều tile → ra nhiều mask giống nhau.
   - Cần bước NMS theo IoU (mask NMS) nếu muốn sạch.

6. **Mismatch shape khi model resize nội bộ**
   - Nếu output mask không cùng size với tile crop, phép gán `full_mask[top:bottom, left:right] = mask_on_tile` sẽ lỗi.
   - Khi đó cần resize mask về đúng `(bottom-top, right-left)` trước khi ghép.

---

## 11) Pseudocode tóm tắt

```text
read image
W,H = image.size
all_masks = []
all_scores = []

for each tile (r,c):
  tile = crop(image, box)
  state = processor.set_image(tile)

  for each prompt in target_objects:
    output = processor.set_text_prompt(state, prompt)

    for each mask_on_tile in output.masks:
      full_mask = zeros(H,W)
      paste mask_on_tile into full_mask[top:bottom,left:right]
      all_masks.append(full_mask)

    all_scores.extend(output.scores)

visualize image + masks filtered by score
```

---

## 12) Gợi ý bạn có thể thêm vào báo cáo (mang tính mô tả, không sửa code)

- So sánh `num_tile_rows=num_tile_cols=1` vs `2` vs `3` (thời gian chạy, số mask, chất lượng đối tượng nhỏ).
- Thử nhiều prompt và thảo luận prompt nào “ít nhiễu” hơn.
- Thảo luận trade-off giữa ngưỡng score (0.1 / 0.3 / 0.5) và số lượng mask.
