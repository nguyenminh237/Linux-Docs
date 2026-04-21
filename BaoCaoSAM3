# Ứng dụng SAM3 với kỹ thuật Tiling cho phân đoạn theo prompt văn bản

## Abstract

Phân đoạn ảnh theo ngữ nghĩa là một bài toán nền tảng trong thị giác máy tính, đặc biệt hữu ích trong các ứng dụng cần khoanh vùng đối tượng mà không muốn/không thể gán nhãn pixel tốn kém. Gần đây, các mô hình _promptable segmentation_ như Segment Anything cho phép người dùng điều khiển đầu ra bằng prompt (điểm, hộp, văn bản), giúp triển khai nhanh theo hướng _zero-shot_.

Tuy nhiên, khi ảnh đầu vào có độ phân giải lớn và đối tượng mục tiêu nhỏ, mô hình thường gặp khó khăn do: (i) chi tiết bị “nuốt” trong toàn cảnh, (ii) hạn chế bộ nhớ GPU khi suy luận ảnh lớn, và (iii) ngữ cảnh đa tỉ lệ. Bài viết này trình bày quy trình suy luận với **SAM3** trong notebook [SAM3_tiling.ipynb](SAM3_tiling.ipynb), đồng thời giới thiệu **kỹ thuật Tiling**: chia ảnh thành lưới các mảnh nhỏ, suy luận từng mảnh, sau đó ghép mask về ảnh gốc. Cách làm này giúp tăng khả năng bắt được chi tiết nhỏ và cho phép xử lý ảnh lớn ổn định hơn.

**Từ khóa:** SAM3, prompt văn bản, segmentation, tiling, SAHI, high-resolution inference.

**Gợi ý hình (đặt sau Abstract):**

- _Hình 1 – Tổng quan pipeline_: Ảnh gốc → chia tile → SAM3 + text prompt → ghép mask → lọc theo score → hiển thị overlay.
  - Từ khóa tìm hình: `tiling inference pipeline`, `SAHI sliced inference diagram`, `promptable segmentation pipeline`.

---

## 1. Introduction

Trong các bài toán thực tế, người dùng thường cần khoanh vùng một loại đối tượng cụ thể (ví dụ: “ngôi nhà”, “xe”, “cây”…), nhưng không có dữ liệu gán nhãn để huấn luyện một mô hình chuyên biệt. Nhóm mô hình _foundation_ như Segment Anything giải quyết vấn đề này bằng cách “học trước” trên dữ liệu lớn và cho phép điều khiển theo prompt.

Notebook [SAM3_tiling.ipynb](SAM3_tiling.ipynb) triển khai:

- Thiết lập môi trường (PyTorch + CUDA, NumPy < 2, cài SAM3).
- Tải checkpoint SAM3 từ Hugging Face.
- Suy luận phân đoạn theo **prompt văn bản**.
- Áp dụng **Tiling** (lưới 2×2 mặc định), ghép mask theo tọa độ tile về ảnh gốc.
- Lọc kết quả theo score (ví dụ ngưỡng 0.1) và trực quan hóa.

Đóng góp trọng tâm của bài viết là mô tả rõ:

1. vì sao tiling giúp bắt đối tượng nhỏ/chi tiết,
2. cách triển khai tiling an toàn (tọa độ, biên tile),
3. các điểm cần lưu ý khi ghép mask và hậu xử lý.

**Gợi ý hình (đặt cuối mục 1):**

- _Hình 2 – Ví dụ “đối tượng nhỏ trong ảnh lớn”_: một ảnh aerial/đông đối tượng + zoom-in một vùng nhỏ.
  - Có thể tự tạo từ dữ liệu của bạn bằng cách crop vùng có nhiều chi tiết.

---

## 2. Related Work

### 2.1. Promptable Segmentation và họ Segment Anything

Các mô hình Segment Anything (SAM và các biến thể) thường có cấu trúc gồm:

- **Image Encoder**: trích xuất đặc trưng ảnh (thường dựa trên ViT).
- **Prompt Encoder**: mã hóa prompt (điểm/hộp/văn bản… tùy biến thể).
- **Mask Decoder**: tạo mask và score chất lượng.

SAM3 trong notebook được dùng theo hướng _inference_ với lớp `Sam3Processor` để:

- nạp ảnh (`set_image`) tạo state suy luận,
- đặt prompt văn bản (`set_text_prompt`) để lấy `masks` và `scores`.

**Gợi ý hình (đặt sau 2.1):**

- _Hình 3 – Kiến trúc Segment Anything (encoder/prompt encoder/decoder)_.
  - Từ khóa tìm hình: `Segment Anything architecture prompt encoder mask decoder`, `SAM model diagram`, `ViT image encoder mask decoder`.
- Nếu muốn cụ thể hơn SAM3:
  - Từ khóa: `facebookresearch sam3`, `SAM3 image model build_sam3_image_model` (thường sẽ dẫn tới repo/paper/blog mô tả).

### 2.2. Suy luận ảnh độ phân giải lớn và kỹ thuật Tiling (SAHI)

Khi ảnh quá lớn, suy luận trực tiếp có thể:

- tốn VRAM,
- làm giảm chất lượng cho đối tượng nhỏ,
- giảm ổn định do cần resize mạnh.

Tiling (hoặc _sliced inference_) chia ảnh thành các mảnh nhỏ hơn để:

- tăng “mật độ pixel” của đối tượng trong tile,
- giảm yêu cầu bộ nhớ,
- cho phép xử lý ảnh kích thước tùy ý.

Nhược điểm thường gặp:

- đứt gãy ở biên tile,
- trùng lặp mask (một đối tượng xuất hiện ở 2 tile),
- mất ngữ cảnh toàn cục nếu tile quá nhỏ.

**Gợi ý hình (đặt sau 2.2):**

- _Hình 4 – Minh họa lưới tiling 2×2/3×3 + overlap_.
  - Từ khóa: `image tiling grid overlap`, `SAHI overlap tiles`.

### 2.3. Hậu xử lý: ngưỡng score, khử trùng lặp và ghép mask

Notebook hiện dùng một hậu xử lý tối giản:

- thu tất cả mask từ các tile,
- lọc theo score (ví dụ `> 0.1`) khi vẽ.

Trong các hệ thống hoàn chỉnh, thường cần thêm:

- **khử trùng lặp** (mask NMS theo IoU),
- **ưu tiên mask tốt hơn** (score cao hơn),
- **xử lý seam** (overlap + chọn mask theo score/độ phủ),
- **lọc theo kích thước** (loại mask quá nhỏ hoặc quá lớn tùy bài toán).

**Gợi ý hình (đặt sau 2.3):**

- _Hình 5 – Minh họa “trùng lặp mask do overlap” và cách NMS loại bớt_.
  - Từ khóa: `mask NMS IoU diagram`, `instance segmentation NMS mask`.

---

## 3. Data and Methodology

Phần này trình bày quy trình theo dạng “nhiều giai đoạn” (tương tự cách trình bày trong PDF tham khảo), nhưng nội dung bám sát notebook [SAM3_tiling.ipynb](SAM3_tiling.ipynb).

### 3.1. Giai đoạn 1: Chuẩn bị dữ liệu đầu vào

**Đầu vào:** một ảnh RGB (trong notebook lấy từ Google Drive), ví dụ:

- `image_path = "/content/drive/MyDrive/images/test6.jpg"`

**Tiền xử lý:**

- mở ảnh bằng PIL và chuyển RGB: `Image.open(...).convert("RGB")`.
- lấy kích thước gốc `(W, H)` để phục vụ ghép mask.

**Gợi ý hình (đặt sau 3.1):**

- _Hình 6 – Ảnh đầu vào_: ảnh gốc độ phân giải lớn.
  - Có thể dùng ngay ảnh bạn đang chạy trong notebook.

### 3.2. Giai đoạn 2: Thiết lập môi trường và nạp mô hình SAM3

Notebook triển khai các bước chính:

- Đồng bộ NumPy và PyTorch để tránh xung đột phiên bản (cài `numpy<2`, PyTorch CUDA 12.6).
- Clone và cài SAM3 từ GitHub (`facebookresearch/sam3`).
- Đăng nhập Hugging Face và tải checkpoint `sam3.pt` bằng `snapshot_download`.
- Nạp mô hình:
  - `model = build_sam3_image_model(checkpoint_path=..., load_from_HF=False, device="cuda")`
  - `processor = Sam3Processor(model)`

**Lưu ý bảo mật:** trong notebook có hard-code token Hugging Face. Khi viết báo cáo hoặc chia sẻ repo, nên ẩn token và dùng biến môi trường/secret.

**Gợi ý hình (đặt sau 3.2):**

- _Hình 7 – Sơ đồ “cài đặt và nạp checkpoint”_: GitHub repo → pip install → HuggingFace snapshot → load model.
  - Đây là hình “quy trình”, có thể tự vẽ bằng draw.io/PowerPoint.

### 3.3. Giai đoạn 3: Định nghĩa prompt và mục tiêu phân đoạn

Notebook dùng prompt văn bản qua:

- `target_objects = ["individual house"]`

Trong thực hành, chất lượng prompt ảnh hưởng lớn. Một số nguyên tắc:

- dùng danh từ cụ thể thay vì chung chung (ví dụ: “individual house” thay vì “house”),
- nếu quá nhiều mask “rác”, tăng độ chặt của prompt hoặc tăng ngưỡng score,
- nếu bỏ sót, thử synonym hoặc thêm mô tả ngữ cảnh.

**Gợi ý hình (đặt sau 3.3):**

- _Hình 8 – Minh họa prompt tốt/xấu_: cùng ảnh, so sánh 2–3 prompt.
  - Có thể tự tạo bằng cách chạy 2–3 prompt khác nhau.

### 3.4. Giai đoạn 4: Tiling và suy luận theo từng tile

**Ý tưởng:** chia ảnh thành lưới `num_tile_rows × num_tile_cols` (mặc định 2×2). Mỗi tile được crop từ ảnh gốc theo tọa độ:

- `left, top, right, bottom`

**Suy luận per-tile:**

1. `tile_image = image.crop((left, top, right, bottom))`
2. `inference_state = processor.set_image(tile_image)`
3. với mỗi prompt `obj`:
   - `output = processor.set_text_prompt(state=inference_state, prompt=obj)`
   - lấy `output["masks"]` và `output["scores"]`

**Tối ưu hiệu năng:** notebook dùng mixed precision:

- `with torch.autocast("cuda", dtype=torch.bfloat16): ...`

**Gợi ý hình (đặt sau 3.4):**

- _Hình 9 – Minh họa crop tile + hệ tọa độ_: đánh số tile (r,c), vẽ khung tile trên ảnh.
  - Từ khóa: `image coordinate system crop bounding box`.

### 3.5. Giai đoạn 5: Ghép mask về ảnh gốc và hậu xử lý tối thiểu

Với mỗi mask dự đoán trên tile, notebook tạo một mask “full” kích thước ảnh gốc:

- `full_mask = np.zeros((H, W), dtype=bool)`
- `full_mask[top:bottom, left:right] = mask_on_tile`

Sau đó:

- gom tất cả `full_mask` vào `all_masks`
- gom score vào `all_scores`

Ở bước vẽ, notebook lọc theo score:

- chỉ hiển thị mask với `scores[i] > 0.1`

**Gợi ý hình (đặt sau 3.5):**

- _Hình 10 – Minh họa “shift mask về ảnh gốc”_: mask tile nằm đúng vị trí sau khi ghép.
  - Đây là hình tự vẽ/screenshot từ notebook.

---

## 4. Experience

Phần này mô tả trải nghiệm chạy notebook và kết quả minh họa.

### 4.1. Thiết lập và cấu hình chạy

- Thiết bị: GPU CUDA (notebook có cell kiểm tra `nvidia-smi` và `torch.cuda.is_available()`).
- Precision: `bfloat16` với `torch.autocast`.
- Tiling: `num_tile_rows = 2`, `num_tile_cols = 2`.
- Prompt ví dụ: `individual house`.
- Ngưỡng hiển thị: `score > 0.1`.

### 4.2. Kết quả minh họa

Trong output minh họa của notebook:

- Ảnh gốc được hiển thị.
- Kết quả overlay ghi nhận **“SAM 3 Multi-stage: 99 masks”** (với prompt `individual house`, tiling 2×2, ngưỡng 0.1).

Diễn giải nhanh:

- Việc chia tile giúp mô hình nhận ra nhiều mái nhà riêng lẻ hơn so với suy luận một lần trên toàn ảnh (kỳ vọng), đặc biệt khi nhà có kích thước nhỏ trong ảnh aerial.
- Ngưỡng 0.1 tương đối “thoáng”, nên số lượng mask có thể tăng, đôi khi kèm mask nhiễu.

**Gợi ý hình (đặt trong 4.2):**

- _Hình 11 – So sánh trước/sau tiling_: (A) không tiling, (B) tiling 2×2.
  - Nếu bạn chưa có (A), chỉ cần chạy lại notebook với `num_tile_rows = num_tile_cols = 1`.
- _Hình 12 – Zoom-in khu vực đông nhà_: so sánh chất lượng mask ở vùng nhỏ.

### 4.3. Thảo luận: ưu điểm, hạn chế, và hướng cải tiến

**Ưu điểm chính:**

- Tăng khả năng bắt chi tiết nhỏ.
- Giảm áp lực bộ nhớ khi ảnh lớn.
- Dễ triển khai, chỉ cần crop + ghép tọa độ.

**Hạn chế:**

- Có thể xuất hiện mask bị cắt ở biên tile.
- Trùng lặp đối tượng nếu dùng overlap (hoặc khi đối tượng nằm sát biên tile).
- Mất ngữ cảnh toàn cục nếu tile quá nhỏ.

**Gợi ý cải tiến (không thay đổi UX của notebook, chỉ là đề xuất kỹ thuật):**

- Tiling có **overlap** (ví dụ 10–20%) + bước khử trùng lặp mask theo IoU.
- Chiến lược đa tỉ lệ: chạy grid thô (2×2) + grid mịn (3×3) rồi hợp nhất.
- Thử nhiều prompt tương đương (synonyms) và hợp nhất theo score.
- Lưu mask ra file (PNG/RLE) để tái sử dụng, thay vì chỉ vẽ.

**Gợi ý hình (đặt cuối mục 4.3):**

- _Hình 13 – Minh họa seam/biên tile_: mask bị cắt ở mép và cách overlap xử lý.
  - Từ khóa: `tiling seam artifact`, `overlapping tiles segmentation`.

---

## 5. Conclusion

Notebook [SAM3_tiling.ipynb](SAM3_tiling.ipynb) cho thấy một quy trình triển khai SAM3 theo prompt văn bản có thể mở rộng cho ảnh độ phân giải lớn bằng kỹ thuật Tiling. Phương pháp chia ảnh thành nhiều tile giúp mô hình tập trung vào chi tiết, đồng thời giữ suy luận ổn định về bộ nhớ. Kết quả minh họa với prompt “individual house” trên ảnh aerial cho thấy số lượng mask phát hiện được có thể tăng đáng kể (ví dụ 99 mask trong output mẫu).

Trong các bước tiếp theo, để hệ thống “production-ready”, cần bổ sung hậu xử lý (khử trùng lặp, xử lý biên tile, lọc theo kích thước) và một kịch bản đánh giá định lượng phù hợp (thời gian, VRAM, chất lượng theo tiêu chí bài toán).

---

## Tài liệu tham khảo (gợi ý)

1. Repo SAM3: https://github.com/facebookresearch/sam3
2. Hugging Face Hub: https://huggingface.co/
3. Segment Anything (SAM): bài báo và tài liệu công bố chính thức của nhóm tác giả.
4. SAHI (Slicing Aided Hyper Inference): tài liệu/README của dự án SAHI.
