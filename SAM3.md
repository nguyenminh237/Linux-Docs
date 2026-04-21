# CHƯƠNG 3. PHƯƠNG PHÁP THỰC HIỆN (BỔ SUNG MỤC 3.1 & 3.2)

Tài liệu này mô tả chi tiết:

- **3.1. Quy trình xử lý dữ liệu đầu vào** (chuẩn hoá ảnh, chuẩn bị prompt, chiến lược chia ảnh/tiling…)
- **3.2. Xây dựng quy trình Inference (Suy luận)** (khởi tạo mô hình, suy luận theo prompt, hậu xử lý và trực quan hoá)

Nội dung được viết theo đúng luồng thao tác đang thể hiện trong notebook `SAM3_tiling.ipynb` (Google Colab, GPU CUDA, đọc ảnh bằng PIL, suy luận bằng `Sam3Processor`, prompt văn bản, và có nhánh cải tiến **Tiling**).

---

## 3.1. Quy trình xử lý dữ liệu đầu vào

### 3.1.1. Mục tiêu của bước xử lý đầu vào

Bước tiền xử lý đầu vào nhằm đảm bảo dữ liệu ảnh:

- Được đọc **đúng định dạng** (RGB) và **đúng kích thước** (biết rõ $W \times H$).
- Phù hợp với mô hình segmentation “promptable” (SAM3), tức là chuẩn bị được **ảnh + prompt**.
- Sẵn sàng cho các chiến lược nâng chất lượng suy luận như **tiling/slicing** (chia ảnh lớn thành nhiều mảnh nhỏ để giữ chi tiết vật thể nhỏ).

Trong bài toán phân đoạn, chất lượng đầu vào ảnh và cách biểu diễn mục tiêu (prompt) ảnh hưởng trực tiếp đến:

- Số lượng mask tìm được.
- Độ sắc nét biên (boundary) của mask.
- Tỉ lệ bỏ sót vật thể nhỏ trong ảnh lớn.

---

### 3.1.2. Nguồn dữ liệu và dạng dữ liệu ảnh

Trong phạm vi đề tài, dữ liệu ảnh có thể đến từ nhiều miền (domain):

- **Ảnh giao thông**: mật độ vật thể cao, nhiều che khuất, nhiều đối tượng giống nhau.
- **Ảnh vệ tinh/drone**: ảnh rất lớn, vật thể mục tiêu rất nhỏ so với toàn ảnh.
- **Ảnh y tế**: cấu trúc phức tạp, tương phản thấp, vùng biên khó phân định.

Dù nguồn khác nhau, để đưa vào pipeline, dữ liệu nên được chuẩn hoá về:

- Ảnh 2D, định dạng phổ biến: `.jpg`, `.png`.
- Màu: **RGB**.
- Kích thước: có thể rất đa dạng; ảnh càng lớn càng dễ phát sinh nhu cầu chia nhỏ.

---

### 3.1.3. Đọc ảnh và chuẩn hoá định dạng

Trong notebook, ảnh được đọc bằng PIL:

- Đầu vào là đường dẫn, ví dụ:
  - `image_path = "/content/drive/MyDrive/images/test6.jpg"`
- Đọc ảnh và ép về RGB:
  - `image = Image.open(image_path).convert("RGB")`
- Lấy kích thước:
  - `W, H = image.size`

**Vì sao cần `convert("RGB")`?**

- Tránh tình trạng ảnh ở mode khác (L, RGBA, P…) làm sai pipeline.
- Đảm bảo mô hình nhận đầu vào nhất quán.

**Khuyến nghị bổ sung (mang tính quy trình, không bắt buộc trong code hiện tại):**

- Nếu ảnh có EXIF orientation (ảnh chụp điện thoại), nên chuẩn hoá orientation trước khi suy luận.
- Nếu làm đánh giá định lượng, nên log kích thước ảnh và metadata (nguồn, thời gian…).

---

### 3.1.4. Chuẩn bị prompt mục tiêu (đặc biệt: prompt văn bản)

SAM3 cho phép prompt nhiều dạng; trong notebook đang dùng **prompt văn bản**.

Cách tổ chức trong notebook:

- Danh sách đối tượng mục tiêu:
  - `target_objects = ["individual house"]`
- Vòng lặp prompt:
  - với mỗi `obj` trong `target_objects`, gọi `processor.set_text_prompt(state, prompt=obj)`.

**Nguyên tắc viết prompt văn bản để ổn định hơn:**

1. **Cụ thể hoá đối tượng**: thay vì “house” → “individual house”, “single-family house”.
2. **Tránh từ quá rộng**: “vehicle”, “thing” thường tạo mask lẫn lộn.
3. **Bổ sung thuộc tính khi cần**: “white car”, “motorbike helmet”, “lung nodule” (tuỳ miền).
4. **Tách prompt theo lớp con** (liên quan kỹ thuật multi-stage/semantic decomposition):
   - Ví dụ giao thông: “motorbike” → “helmet”, “wheel”, “person”.
   - Ví dụ y tế: “hand bones” → “phalanges”, “metacarpal”, “wrist joint space”.

**Kết quả đầu ra của prompt văn bản** thường gồm:

- `masks`: danh sách mask nhị phân (mỗi mask biểu diễn một vùng đối tượng).
- `scores`: điểm tin cậy cho từng mask.

---

### 3.1.5. Chiến lược xử lý ảnh lớn: Tiling/Slicing ở mức dữ liệu

Khi ảnh rất lớn hoặc có nhiều chi tiết nhỏ, ta có thể chia ảnh thành lưới các mảnh (tile).

Trong notebook, tiling được cấu hình bằng:

- `num_tile_rows = 2`
- `num_tile_cols = 2`

Toạ độ từng tile (không chồng lấn) được tính như sau:

- `left = c * (W // num_tile_cols)`
- `top = r * (H // num_tile_rows)`
- `right = (c + 1) * (W // num_tile_cols)`
- `bottom = (r + 1) * (H // num_tile_rows)`

Sau đó có bước **sửa tile cuối** để không bị hụt do chia không đều:

- Nếu là cột cuối: `right = W`
- Nếu là hàng cuối: `bottom = H`

**Mục tiêu của tiling:**

- Mỗi tile có vùng nhìn nhỏ hơn → mô hình “tập trung” hơn vào chi tiết.
- Giảm nguy cơ vật thể nhỏ bị mất chi tiết sau các bước downsample nội bộ.

**Khuyến nghị quy trình (tuỳ chọn): thêm overlap**
Trong thực tế, nên có **overlap** giữa các tile để:

- Giảm hiện tượng “đứt mask” tại biên tile.
- Tăng tính liên tục của biên.

Một cấu hình điển hình:

- Overlap theo chiều ngang/ dọc: 10–30% kích thước tile.
- Khi ghép tile, cần quy tắc hợp nhất mask để tránh trùng lặp.

---

### 3.1.6. Ánh xạ mask từ tile về ảnh gốc (chuẩn bị dữ liệu đầu ra)

Vì suy luận trên tile tạo mask trong hệ toạ độ tile, cần đưa mask về hệ toạ độ ảnh gốc.

Trong notebook, quy trình là:

1. Tạo mask full kích thước ảnh gốc:
   - `full_mask = np.zeros((H, W), dtype=bool)`
2. Lấy mask trên tile:
   - `mask_on_tile = m.cpu().numpy()`
3. Gán vào đúng vùng:
   - `full_mask[top:bottom, left:right] = mask_on_tile`

Kết quả:

- `all_masks`: danh sách các mask đã nằm đúng vị trí trên ảnh gốc.
- `all_scores`: danh sách score đi kèm.

**Lưu ý quan trọng:**

- Cách ghép “gán thẳng” như trên phù hợp khi tile **không overlap**.
- Nếu tile **có overlap**, có thể phát sinh trùng lặp hoặc chồng lấn; cần bước hợp nhất (NMS theo IoU, ưu tiên score cao hơn, hoặc union/intersection tuỳ bài toán).

---

## 3.2. Xây dựng quy trình Inference (Suy luận)

### 3.2.1. Mục tiêu của inference pipeline

Inference pipeline là chuỗi bước biến đổi:

> **Ảnh + prompt** → **mask(s) + score(s)** → (lọc / ghép / hậu xử lý) → **kết quả phân đoạn cuối cùng**

Yêu cầu của pipeline:

- Tái lập (reproducible), dễ chạy lại.
- Linh hoạt thay đổi prompt và cấu hình.
- Có thể mở rộng sang các chiến lược cải tiến (tiling, multi-stage, semantic decomposition, post-processing).

---

### 3.2.2. Khởi tạo môi trường suy luận

Trong notebook, môi trường inference được chuẩn bị theo hướng Google Colab + GPU:

1. Kiểm tra GPU và CUDA:
   - `!nvidia-smi`
   - `torch.cuda.is_available()`
2. Đảm bảo phiên bản thư viện tương thích:
   - Hạ `numpy < 2` để tránh lỗi hệ sinh thái.
   - Cài PyTorch CUDA phù hợp.
3. Cài và tải SAM3:
   - `git clone` repo SAM3.
   - `pip install -e .[notebooks]`.
   - Tải checkpoint từ Hugging Face.

**Ghi chú bảo mật (rất quan trọng):**

- Token Hugging Face không nên hard-code trong notebook/báo cáo công khai.
- Thực hành tốt: dùng biến môi trường/secret của Colab.

---

### 3.2.3. Nạp mô hình và tạo Processor

Trong notebook, mô hình được nạp từ checkpoint local và chạy trên GPU:

- Nạp model:
  - `model = build_sam3_image_model(checkpoint_path=..., load_from_HF=False, device="cuda")`
- Tạo processor:
  - `processor = Sam3Processor(model)`

**Vai trò của `Sam3Processor`:**

- Là lớp “điều phối” inference: nhận ảnh, tạo state, nhận prompt, trả về masks/scores.
- Giúp chuẩn hoá cách gọi mô hình thay vì phải tự xử lý toàn bộ tensor/encode prompt.

---

### 3.2.4. Thiết lập ảnh đầu vào và inference state

Trước khi đặt prompt, pipeline cần “đưa ảnh vào” processor:

- Không tiling:
  1. `inference_state = processor.set_image(image)`

- Có tiling:
  1. Với mỗi tile: `tile_image = image.crop((left, top, right, bottom))`
  2. `inference_state = processor.set_image(tile_image)`

`inference_state` có thể hiểu là trạng thái chứa embedding/feature đã trích xuất từ ảnh, để các prompt sau đó có thể truy vấn nhanh.

---

### 3.2.5. Thực hiện suy luận theo prompt văn bản

Với mỗi prompt `obj`, gọi:

- `output = processor.set_text_prompt(state=inference_state, prompt=obj)`

Đầu ra `output` (theo notebook) có các trường quan trọng:

- `output["masks"]`: danh sách mask tensor (thường là nhị phân hoặc gần nhị phân).
- `output["scores"]`: danh sách điểm tin cậy cho các mask tương ứng.

Pipeline thu thập kết quả:

- Nếu không tiling: dùng trực tiếp `output["masks"]` và `output["scores"]`.
- Nếu tiling: ánh xạ mask về ảnh gốc (mục 3.1.6), rồi cộng dồn vào `all_masks`, `all_scores`.

---

### 3.2.6. Tối ưu suy luận bằng mixed precision (`autocast`)

Notebook sử dụng:

- `with torch.autocast("cuda", dtype=torch.bfloat16):`

Tác dụng:

- Giảm thời gian suy luận.
- Giảm dùng VRAM, hữu ích với ảnh lớn hoặc nhiều tile.

**Lưu ý:**

- Mixed precision có thể ảnh hưởng nhẹ đến biên mask trong một số tình huống; cần kiểm chứng thực nghiệm.

---

### 3.2.7. Lọc kết quả theo score và hậu xử lý cơ bản

Sau khi có danh sách `masks` và `scores`, thường cần:

1. **Lọc theo ngưỡng score**
   Trong notebook khi vẽ, đang dùng ngưỡng thấp:

- `if scores[i] > 0.1:`

Ý nghĩa:

- Ngưỡng thấp giúp “bắt” nhiều đối tượng nhỏ hơn, nhưng tăng nhiễu/false positive.

2. **Loại bỏ mask quá nhỏ (tuỳ chọn)**
   Một quy tắc phổ biến:

- Tính diện tích mask (số pixel True) và bỏ mask dưới một ngưỡng.

3. **Gộp mask trùng lặp (tuỳ chọn nhưng rất hữu ích)**
   Nếu pipeline sinh nhiều mask chồng lấn (đặc biệt khi có overlap tile hoặc multi-stage prompt):

- Dùng IoU để loại mask trùng nhau (giống NMS).
- Ưu tiên mask score cao.

Gợi ý quy trình hợp nhất:

- Sắp xếp mask theo score giảm dần.
- Duyệt lần lượt, chỉ giữ mask có IoU với các mask đã giữ < ngưỡng (ví dụ 0.5).

---

### 3.2.8. Trực quan hoá kết quả phân đoạn

Notebook có hàm `show_mask` để phủ mask lên ảnh bằng màu (alpha=0.6).

Quy trình vẽ:

- Vẽ ảnh gốc ở subplot 1.
- Vẽ ảnh + toàn bộ mask đạt ngưỡng ở subplot 2.

Điểm cần chú ý khi trực quan hoá:

- Nên hiển thị số lượng mask cuối cùng sau lọc.
- Nếu có nhiều mask, nên random color để dễ phân biệt.

---

### 3.2.9. Pseudocode tổng quát cho pipeline inference

Dưới đây là pseudocode tổng quát (tương ứng với notebook, có nhánh tiling):

```text
Input: image (PIL RGB), target_objects (list of strings)
Config: use_tiling, num_tile_rows, num_tile_cols, score_threshold

Load model + processor

if not use_tiling:
    state = processor.set_image(image)
    for obj in target_objects:
        output = processor.set_text_prompt(state, obj)
        collect masks, scores
else:
    W,H = image.size
    for each tile (r,c) in grid:
        tile = crop(image, tile_box)
        state = processor.set_image(tile)
        for obj in target_objects:
            output = processor.set_text_prompt(state, obj)
            for each mask in output.masks:
                full_mask = place_mask_to_full_image(mask, tile_box, H, W)
                collect full_mask, score

Filter masks by score_threshold
(Optional) remove tiny masks
(Optional) merge duplicates
Visualize or export masks
```

---

### 3.2.10. Đầu ra của pipeline và cách lưu kết quả (khuyến nghị)

Để phục vụ Chương 4 (thực nghiệm), nên chuẩn hoá đầu ra pipeline thành:

- Ảnh overlay kết quả (`.png`).
- Mask nhị phân theo từng đối tượng (có thể lưu `.npy` hoặc `.png` 0/255).
- File log metadata:
  - Prompt đã dùng.
  - `num_tile_rows/cols`.
  - Ngưỡng score.
  - Thời gian suy luận.
  - Số mask trước/sau lọc.

---

## Kết nối với mục 3.3 (các kỹ thuật cải tiến)

Hai mục 3.1 và 3.2 là “xương sống” của hệ thống; từ pipeline này có thể gắn trực tiếp các cải tiến ở 3.3:

- **Multi-stage Prompting**: tổ chức `target_objects` theo tầng (từ lớp tổng quát → lớp con).
- **Tiling/Slicing**: đã mô tả ở 3.1.5 và đang triển khai trong notebook.
- **Semantic Decomposition**: xem như một quy tắc sinh prompt/chuỗi prompt để giảm bỏ sót.
- **Post-processing**: thêm bước lọc/ghép mask và ngưỡng thích nghi theo lớp đối tượng.
