# Tài Liệu Hướng Dẫn - MyExplorer Plugin

## Tổng Quan

MyExplorer là một plugin file explorer tùy chỉnh cho Neovim, cung cấp giao diện dạng cây để quản lý file và folder tương tự như VS Code.

### Tính Năng Chính
- ✔ Mở/Đóng thư mục (expand/collapse)
- ✔ Highlight file đang mở
- ✔ Icon với Nerd Font
- ✔ Hỗ trợ chuột click
- ✔ Tự động refresh khi file thay đổi
- ✔ Đổi tên / Xóa / Tạo file
- ✔ Mở file trong cửa sổ chính

---

## 1. Cấu Trúc Module và Biến Toàn Cục

### Module M
```lua
local M = {}
```
**Chức năng:** Khởi tạo module chính `M` - đây là table chứa tất cả các function và state của plugin.

**Chi tiết:**
- `local M = {}`: Tạo một table rỗng và gán cho biến local `M`. Table này sẽ được return ở cuối file và hoạt động như một module Lua.

---

### Biến Window và Buffer
```lua
M.win = nil
M.buf = nil
```
**Chức năng:** Lưu trữ ID của window và buffer của explorer.

**Chi tiết:**
- `M.win = nil`: Biến lưu ID của window sidebar. Ban đầu là `nil`, được gán giá trị khi explorer được mở.
- `M.buf = nil`: Biến lưu ID của buffer hiển thị nội dung explorer. Ban đầu là `nil`, được gán khi buffer được tạo.

---

### Cấu Trúc Cây
```lua
M.tree = {}
```
**Chức năng:** Lưu trữ cấu trúc dạng cây của toàn bộ thư mục.

**Chi tiết:**
- `M.tree = {}`: Table rỗng ban đầu, sau sẽ chứa root node với cấu trúc:
  - `name`: tên thư mục
  - `fullpath`: đường dẫn đầy đủ
  - `is_dir`: có phải folder không
  - `expanded`: trạng thái mở/đóng
  - `children`: mảng chứa các node con

---

### Icons
```lua
local icons = {
    folder_closed = "📁 ▸",
    folder_open = "📂 ▾",
    file = "📄"
}
```
**Chức năng:** Định nghĩa icon hiển thị cho folder và file.

**Chi tiết:**
- `local icons = {...}`: Tạo table local chứa các icon
- `folder_closed = "📁 ▸"`: Icon cho folder đang đóng (chưa expand), sử dụng emoji folder và mũi tên phải
- `folder_open = "📂 ▾"`: Icon cho folder đang mở (đã expand), sử dụng emoji folder mở và mũi tên xuống
- `file = "📄"`: Icon cho file thường, sử dụng emoji tài liệu

---

## 2. Hàm scan_dir()

```lua
local function scan_dir(path)
```
**Chức năng:** Quét thư mục và trả về danh sách tất cả file/folder trong đó.

**Tham số:**
- `path`: Đường dẫn đầy đủ của thư mục cần quét

**Trả về:** Mảng chứa các node (file/folder) đã được sắp xếp

---

### Chi Tiết Từng Dòng

```lua
local items = vim.fn.readdir(path)
```
- `vim.fn.readdir(path)`: Gọi hàm Vim `readdir()` để đọc tất cả file/folder trong thư mục `path`
- Kết quả trả về là một mảng string chứa tên các file/folder (không có đường dẫn đầy đủ)
- `local items`: Lưu kết quả vào biến local `items`

---

```lua
local result = {}
```
- Tạo một table rỗng `result` để chứa kết quả cuối cùng
- Mỗi phần tử trong `result` sẽ là một node với thông tin chi tiết

---

```lua
for _, name in ipairs(items) do
```
- Vòng lặp `for` duyệt qua từng phần tử trong mảng `items`
- `ipairs(items)`: Iterator cho mảng, trả về `(index, value)`
- `_`: Bỏ qua index vì không cần dùng
- `name`: Tên file/folder ở iteration hiện tại

---

```lua
local full = path .. "/" .. name
```
- Ghép đường dẫn đầy đủ bằng cách nối: `path + "/" + name`
- Ví dụ: `/home/user` + `/` + `Documents` = `/home/user/Documents`
- `local full`: Lưu đường dẫn đầy đủ

---

```lua
local is_dir = vim.fn.isdirectory(full) == 1
```
- `vim.fn.isdirectory(full)`: Gọi hàm Vim `isdirectory()` để kiểm tra xem `full` có phải folder không
- Trả về `1` nếu là folder, `0` nếu là file
- `== 1`: So sánh với 1 để convert sang boolean
- `local is_dir`: Lưu kết quả boolean (true/false)

---

```lua
table.insert(result, {
    name = name,
    fullpath = full,
    is_dir = is_dir,
    expanded = false,
    children = {}
})
```
- `table.insert(result, {...})`: Thêm một phần tử mới vào cuối mảng `result`
- Phần tử được thêm là một table (node) chứa:
  - `name = name`: Tên file/folder (không có path)
  - `fullpath = full`: Đường dẫn đầy đủ
  - `is_dir = is_dir`: Boolean cho biết là folder hay file
  - `expanded = false`: Trạng thái ban đầu là chưa mở (false)
  - `children = {}`: Mảng rỗng để chứa các node con (nếu là folder)

---

```lua
end
```
- Kết thúc vòng lặp `for`

---

```lua
table.sort(result, function(a, b)
    if a.is_dir ~= b.is_dir then
        return a.is_dir
    end
    return a.name < b.name
end)
```
**Chức năng:** Sắp xếp mảng `result` theo quy tắc: folder lên trước, sau đó sắp xếp theo tên alphabet

**Chi tiết:**
- `table.sort(result, function(a, b) ... end)`: Gọi hàm sort với hàm so sánh tùy chỉnh
- `function(a, b)`: Hàm comparator nhận 2 tham số `a` và `b` (2 node)
- `if a.is_dir ~= b.is_dir then`: Nếu một là folder, một là file
  - `return a.is_dir`: Trả về true nếu `a` là folder (folder lên trước)
- `return a.name < b.name`: Nếu cả 2 cùng loại, so sánh tên theo thứ tự alphabet

---

```lua
return result
```
- Trả về mảng `result` đã được sắp xếp
- Kết thúc hàm `scan_dir()`

---

## 3. Hàm build_tree()

```lua
local function build_tree(node)
```
**Chức năng:** Xây dựng cây con cho một node (folder) cụ thể.

**Tham số:**
- `node`: Node (folder) cần expand và load children

---

### Chi Tiết

```lua
node.children = scan_dir(node.fullpath)
```
- `scan_dir(node.fullpath)`: Gọi hàm `scan_dir()` với đường dẫn của node
- Kết quả trả về là mảng các node con (file/folder trong thư mục này)
- `node.children = ...`: Gán mảng kết quả vào thuộc tính `children` của node
- Hàm này sẽ được gọi khi user click vào folder để expand

---

## 4. Hàm render_tree()

```lua
local function render_tree()
```
**Chức năng:** Render cấu trúc cây thành các dòng text và hiển thị lên buffer.

**Không có tham số**

**Không trả về giá trị**

---

### Chi Tiết Từng Khối

#### Kiểm Tra Buffer Hợp Lệ
```lua
if not (M.buf and vim.api.nvim_buf_is_valid(M.buf)) then return end
```
- `M.buf`: Kiểm tra xem biến `M.buf` có tồn tại không (không phải nil)
- `vim.api.nvim_buf_is_valid(M.buf)`: Gọi API Neovim để kiểm tra buffer có còn valid không
- `M.buf and ...`: Short-circuit evaluation - chỉ kiểm tra valid nếu M.buf tồn tại
- `not (...)`: Đảo ngược điều kiện - true khi buffer không valid
- `then return end`: Nếu buffer không valid, thoát hàm ngay

---

#### Khởi Tạo Biến
```lua
local lines = {}
local line_nodes = {}
```
- `local lines = {}`: Mảng chứa các dòng text sẽ hiển thị trên buffer
- `local line_nodes = {}`: Mảng mapping từ số dòng → node tương ứng (dùng để xử lý click)

---

#### Hàm walk() - Duyệt Cây Đệ Quy
```lua
local function walk(node, depth)
```
**Chức năng:** Hàm đệ quy để duyệt cây và tạo các dòng text với indent phù hợp.

**Tham số:**
- `node`: Node hiện tại đang duyệt
- `depth`: Độ sâu của node trong cây (0 = root, 1 = level 1, ...)

---

```lua
for _, child in ipairs(node.children) do
```
- Vòng lặp duyệt qua tất cả các node con của `node`
- `ipairs(node.children)`: Iterator cho mảng children
- `_`: Bỏ qua index
- `child`: Node con ở iteration hiện tại

---

```lua
local prefix = string.rep("  ", depth)
```
- `string.rep("  ", depth)`: Lặp lại string `"  "` (2 spaces) `depth` lần
- Ví dụ: depth=0 → `""`, depth=1 → `"  "`, depth=2 → `"    "`
- `local prefix`: Lưu string indent, dùng để tạo cấu trúc cây trực quan

---

```lua
local icon = child.is_dir
    and (child.expanded and icons.folder_open or icons.folder_closed)
    or icons.file
```
**Chức năng:** Chọn icon phù hợp dựa trên loại và trạng thái của node

**Chi tiết:**
- `child.is_dir and ... or icons.file`: Ternary operator trong Lua
  - Nếu `child.is_dir` = true: lấy phần `and`
  - Nếu `child.is_dir` = false: lấy `icons.file`
- `(child.expanded and icons.folder_open or icons.folder_closed)`: Ternary lồng nhau
  - Nếu `child.expanded` = true: lấy `icons.folder_open`
  - Nếu `child.expanded` = false: lấy `icons.folder_closed`
- Tổng hợp: 
  - Folder mở → `icons.folder_open`
  - Folder đóng → `icons.folder_closed`
  - File → `icons.file`

---

```lua
table.insert(lines, prefix .. icon .. " " .. child.name)
```
- Ghép string: `prefix` + `icon` + `" "` + `child.name`
- Ví dụ: `"  "` + `"📁 ▸"` + `" "` + `"Documents"` = `"  📁 ▸ Documents"`
- `table.insert(lines, ...)`: Thêm string đã ghép vào cuối mảng `lines`

---

```lua
table.insert(line_nodes, child)
```
- Thêm `child` node vào cuối mảng `line_nodes`
- Tạo mapping: index của dòng text ↔ node tương ứng
- Dùng để khi user click vào dòng X, ta biết được node nào đang được chọn

---

```lua
if child.is_dir and child.expanded then
    walk(child, depth + 1)
end
```
- `if child.is_dir and child.expanded then`: Kiểm tra 2 điều kiện:
  - `child.is_dir`: Node phải là folder
  - `child.expanded`: Folder phải đang ở trạng thái mở
- `walk(child, depth + 1)`: Gọi đệ quy hàm `walk()` với:
  - `child`: Node con làm root mới
  - `depth + 1`: Tăng độ sâu lên 1 (indent thêm 2 spaces)
- `end`: Kết thúc if và vòng lặp for

---

#### Gọi Hàm walk()
```lua
walk(M.tree, 0)
```
- Khởi động quá trình duyệt cây
- `M.tree`: Root node (thư mục gốc)
- `0`: Độ sâu ban đầu = 0 (không indent)

---

#### Ghi Nội Dung Vào Buffer
```lua
vim.api.nvim_buf_set_lines(M.buf, 0, -1, false, lines)
```
- `vim.api.nvim_buf_set_lines()`: API Neovim để set nội dung buffer
- Tham số:
  - `M.buf`: ID của buffer cần update
  - `0`: Bắt đầu từ dòng 0 (đầu file)
  - `-1`: Đến dòng -1 (cuối file)
  - `false`: Không strict indexing
  - `lines`: Mảng các dòng text cần ghi
- Kết quả: Replace toàn bộ nội dung buffer bằng mảng `lines`

---

```lua
M.line_nodes = line_nodes
```
- Lưu mảng `line_nodes` vào thuộc tính module `M.line_nodes`
- Cho phép các hàm khác (như `open_or_expand()`) truy cập mapping dòng → node

---

#### Highlight File Đang Mở
```lua
local cur_file = vim.fn.expand("%:p")
```
- `vim.fn.expand("%:p")`: Expand đường dẫn của file hiện tại
  - `%`: Đại diện cho file đang mở
  - `:p`: Modifier để lấy đường dẫn đầy đủ (full path)
- `local cur_file`: Lưu đường dẫn đầy đủ của file đang được edit

---

```lua
for i, node in ipairs(line_nodes) do
```
- Vòng lặp duyệt qua mảng `line_nodes` với index
- `ipairs(line_nodes)`: Iterator trả về `(index, value)`
- `i`: Index của node (cũng là số dòng, 1-indexed)
- `node`: Node tại position `i`

---

```lua
if node.fullpath == cur_file then
    vim.api.nvim_buf_add_highlight(M.buf, -1, "Visual", i - 1, 0, -1)
end
```
- `if node.fullpath == cur_file then`: Kiểm tra xem node có phải file đang mở không
- `vim.api.nvim_buf_add_highlight()`: API để add highlight cho buffer
- Tham số:
  - `M.buf`: ID của buffer
  - `-1`: Namespace ID (-1 = namespace mặc định)
  - `"Visual"`: Highlight group (dùng Visual để giống khi select text)
  - `i - 1`: Số dòng cần highlight (0-indexed, nên phải trừ 1)
  - `0`: Cột bắt đầu (0 = đầu dòng)
  - `-1`: Cột kết thúc (-1 = cuối dòng)
- Kết quả: Dòng chứa file đang mở sẽ có background highlight

---

## 5. Hàm M.open_or_expand()

```lua
function M.open_or_expand()
```
**Chức năng:** Xử lý khi user nhấn Enter hoặc click vào một dòng. Nếu là folder → expand/collapse, nếu là file → mở file.

**Không có tham số (lấy cursor position)**

**Không trả về giá trị**

---

### Chi Tiết Từng Khối

```lua
local row = unpack(vim.api.nvim_win_get_cursor(M.win))
```
**Chức năng:** Lấy số dòng hiện tại của cursor

**Chi tiết:**
- `vim.api.nvim_win_get_cursor(M.win)`: API lấy vị trí cursor trong window `M.win`
- Trả về: `{row, col}` - một mảng có 2 phần tử (dòng và cột)
- `unpack(...)`: Unpack mảng thành các giá trị riêng biệt
- `local row`: Chỉ lấy giá trị đầu tiên (row), bỏ qua col
- Note: row là 1-indexed (dòng đầu tiên = 1)

---

```lua
local node = M.line_nodes[row]
```
- `M.line_nodes[row]`: Truy cập mảng mapping để lấy node tương ứng với dòng `row`
- `local node`: Lưu node vào biến

---

```lua
if not node then return end
```
- `if not node then`: Kiểm tra xem node có tồn tại không (có thể nil nếu click vào dòng trống)
- `return end`: Nếu không có node, thoát hàm ngay

---

#### Xử Lý Folder
```lua
if node.is_dir then
```
- Kiểm tra xem node có phải là folder không

---

```lua
if not node.expanded then
    build_tree(node)
end
```
- `if not node.expanded then`: Kiểm tra folder chưa được expand (chưa load children)
- `build_tree(node)`: Gọi hàm để load các file/folder con vào `node.children`
- Chỉ load khi cần thiết (lazy loading) → tối ưu performance

---

```lua
node.expanded = not node.expanded
```
- Toggle trạng thái expanded: true → false, false → true
- `not node.expanded`: Đảo ngược giá trị boolean
- Kết quả: 
  - Folder đang đóng → mở
  - Folder đang mở → đóng

---

```lua
render_tree()
```
- Gọi hàm `render_tree()` để vẽ lại toàn bộ cây với trạng thái mới
- Cập nhật display ngay lập tức

---

#### Xử Lý File
```lua
else
```
- Nhánh else: Xử lý khi node là file (không phải folder)

---

```lua
vim.cmd("wincmd l")
```
- `vim.cmd()`: Thực thi lệnh Vim command
- `"wincmd l"`: Window command di chuyển cursor sang window bên phải
- Mục đích: Chuyển focus từ sidebar sang cửa sổ chính để mở file ở đó

---

```lua
vim.cmd("edit " .. node.fullpath)
```
- Ghép string: `"edit "` + đường dẫn file
- Ví dụ: `"edit /home/user/file.txt"`
- `vim.cmd(...)`: Thực thi lệnh `:edit` để mở file
- File sẽ được mở trong window hiện tại (window chính sau khi chuyển focus)

---

```lua
vim.cmd("wincmd h")
```
- `"wincmd h"`: Window command di chuyển cursor sang window bên trái
- Mục đích: Quay lại sidebar sau khi mở file
- User có thể tiếp tục duyệt file mà không cần chuyển focus thủ công

---

```lua
end
```
- Kết thúc khối else

---

## 6. Hàm rename_buffer()

```lua
local function rename_buffer(old_path, new_path)
```
**Chức năng:** Cập nhật tên buffer khi file được rename, tránh trường hợp buffer vẫn giữ tên cũ.

**Tham số:**
- `old_path`: Đường dẫn cũ của file
- `new_path`: Đường dẫn mới của file

---

### Chi Tiết

```lua
for _, buf in ipairs(vim.api.nvim_list_bufs()) do
```
- `vim.api.nvim_list_bufs()`: API lấy danh sách tất cả buffer đang tồn tại
- Trả về: Mảng các buffer ID
- `ipairs(...)`: Iterator để duyệt mảng
- `_`: Bỏ qua index
- `buf`: Buffer ID ở iteration hiện tại

---

```lua
if vim.api.nvim_buf_is_loaded(buf) then
```
- `vim.api.nvim_buf_is_loaded(buf)`: Kiểm tra xem buffer có đang được load không
- Chỉ xử lý buffer đang loaded (đang được sử dụng)

---

```lua
local name = vim.api.nvim_buf_get_name(buf)
```
- `vim.api.nvim_buf_get_name(buf)`: Lấy tên (đường dẫn) của buffer
- `local name`: Lưu đường dẫn vào biến

---

```lua
if name == old_path then
```
- So sánh đường dẫn buffer với `old_path`
- Tìm buffer đang mở file cũ (trước khi rename)

---

```lua
vim.api.nvim_buf_set_name(buf, new_path)
```
- `vim.api.nvim_buf_set_name()`: API để đổi tên buffer
- Tham số:
  - `buf`: Buffer ID cần đổi tên
  - `new_path`: Đường dẫn mới
- Buffer sẽ được cập nhật tên từ old_path → new_path

---

```lua
vim.api.nvim_buf_set_option(buf, "modified", false)
```
- `vim.api.nvim_buf_set_option()`: API để set option của buffer
- Tham số:
  - `buf`: Buffer ID
  - `"modified"`: Option name - đánh dấu buffer có thay đổi chưa save
  - `false`: Giá trị false = không có thay đổi
- Mục đích: Tránh Neovim hiển thị warning "buffer has unsaved changes" khi đổi tên

---

```lua
end -- kết thúc if name == old_path
end -- kết thúc if nvim_buf_is_loaded
end -- kết thúc vòng lặp for
```

---

## 7. Hàm M.rename()

```lua
function M.rename()
```
**Chức năng:** Đổi tên file/folder được chọn.

**Không có tham số**

**Không trả về giá trị**

---

### Chi Tiết

```lua
local row = unpack(vim.api.nvim_win_get_cursor(M.win))
local node = M.line_nodes[row]
if not node then return end
```
- Giống như trong `open_or_expand()`: Lấy node tại cursor position
- Thoát hàm nếu không có node

---

```lua
local new = vim.fn.input("Rename to: ", node.name)
```
- `vim.fn.input()`: Hiển thị prompt input để user nhập dữ liệu
- Tham số:
  - `"Rename to: "`: Text prompt hiển thị
  - `node.name`: Giá trị mặc định (tên hiện tại)
- Trả về: String mà user nhập vào
- `local new`: Lưu tên mới

---

```lua
if new == "" then return end
```
- Kiểm tra user có hủy thao tác không (nhấn Esc hoặc không nhập gì)
- `new == ""`: True khi user không nhập gì
- `return`: Thoát hàm, không rename

---

```lua
local old_path = node.fullpath
```
- Lưu đường dẫn cũ vào biến `old_path`

---

```lua
local new_path = vim.fn.fnamemodify(old_path, ":h") .. "/" .. new
```
**Chức năng:** Tạo đường dẫn mới bằng cách giữ nguyên thư mục cha và thay đổi tên file

**Chi tiết:**
- `vim.fn.fnamemodify(old_path, ":h")`: Modify filename với modifier `:h` (head)
  - Lấy phần thư mục cha của đường dẫn
  - Ví dụ: `/home/user/file.txt` → `/home/user`
- `.. "/" .. new`: Ghép: thư mục cha + `/` + tên mới
- Ví dụ: `/home/user` + `/` + `newfile.txt` = `/home/user/newfile.txt`
- `local new_path`: Lưu đường dẫn đầy đủ mới

---

```lua
os.rename(old_path, new_path)
```
- `os.rename()`: Hàm Lua standard library để rename file trong filesystem
- Tham số:
  - `old_path`: Đường dẫn cũ
  - `new_path`: Đường dẫn mới
- Thực hiện rename file/folder thực sự trên hệ thống

---

```lua
rename_buffer(old_path, new_path)
```
- Gọi hàm helper `rename_buffer()` để cập nhật buffer đang mở file này
- Đảm bảo buffer trong Neovim sync với filesystem

---

```lua
node.fullpath = new_path
node.name = new
```
- Cập nhật thông tin node trong cấu trúc cây:
  - `node.fullpath = new_path`: Đường dẫn đầy đủ mới
  - `node.name = new`: Tên mới (không có path)

---

```lua
render_tree()
```
- Vẽ lại cây để hiển thị tên mới

---

## 8. Hàm handle_deleted_buffer()

```lua
local function handle_deleted_buffer(deleted_path)
```
**Chức năng:** Xử lý buffer khi file tương ứng bị xóa - đóng window và xóa buffer.

**Tham số:**
- `deleted_path`: Đường dẫn của file đã bị xóa

---

### Chi Tiết

```lua
for _, buf in ipairs(vim.api.nvim_list_bufs()) do
```
- Duyệt qua tất cả buffer (giống như trong `rename_buffer()`)

---

```lua
if vim.api.nvim_buf_is_loaded(buf) then
    local name = vim.api.nvim_buf_get_name(buf)
```
- Kiểm tra buffer đang loaded và lấy tên

---

```lua
if name == deleted_path then
```
- Tìm buffer đang mở file đã bị xóa

---

```lua
for _, win in ipairs(vim.api.nvim_list_wins()) do
```
- `vim.api.nvim_list_wins()`: API lấy danh sách tất cả window
- Duyệt qua từng window

---

```lua
if vim.api.nvim_win_get_buf(win) == buf then
```
- `vim.api.nvim_win_get_buf(win)`: Lấy buffer đang hiển thị trong window `win`
- So sánh với `buf`: Tìm window đang hiển thị buffer của file đã xóa

---

```lua
vim.api.nvim_set_current_win(win)
```
- `vim.api.nvim_set_current_win(win)`: Set window hiện tại thành `win`
- Chuyển focus đến window đang hiển thị file đã xóa

---

```lua
vim.cmd("enew")
```
- `vim.cmd("enew")`: Thực thi lệnh `:enew` - tạo buffer trống mới
- Thay thế buffer của file đã xóa bằng buffer trống
- Tránh hiển thị nội dung của file không còn tồn tại

---

```lua
end -- kết thúc if nvim_win_get_buf
end -- kết thúc for windows
```

---

```lua
vim.api.nvim_buf_delete(buf, { force = true })
```
- `vim.api.nvim_buf_delete()`: API để xóa buffer
- Tham số:
  - `buf`: Buffer ID cần xóa
  - `{ force = true }`: Option force = true để xóa ngay cả khi buffer có thay đổi chưa save

---

```lua
end -- kết thúc if name == deleted_path
end -- kết thúc if nvim_buf_is_loaded
end -- kết thúc for buffers
```

---

## 9. Hàm M.delete()

```lua
function M.delete()
```
**Chức năng:** Xóa file/folder được chọn.

**Không có tham số**

**Không trả về giá trị**

---

### Chi Tiết

```lua
local row = unpack(vim.api.nvim_win_get_cursor(M.win))
local node = M.line_nodes[row]
if not node then return end
```
- Lấy node tại cursor position (giống các hàm trước)

---

```lua
local confirm = vim.fn.input("Delete " .. node.name .. "? [y/N] ")
```
- `vim.fn.input()`: Hiển thị prompt xác nhận
- Ghép string: `"Delete "` + tên file + `"? [y/N] "`
- Ví dụ: `"Delete file.txt? [y/N] "`
- `local confirm`: Lưu câu trả lời của user

---

```lua
if confirm ~= "y" then return end
```
- Kiểm tra user có confirm bằng "y" không
- `confirm ~= "y"`: True khi user nhập bất kỳ ký tự nào khác "y"
- `return`: Hủy thao tác delete

---

```lua
local path = node.fullpath
```
- Lưu đường dẫn file cần xóa

---

```lua
os.remove(path)
```
- `os.remove(path)`: Hàm Lua standard library để xóa file
- Xóa file/folder thực sự khỏi filesystem
- Note: Chỉ xóa được file hoặc folder rỗng

---

```lua
handle_deleted_buffer(path)
```
- Gọi hàm helper để xử lý buffer đang mở file này
- Đóng window và xóa buffer nếu file đang được mở

---

```lua
render_tree()
```
- Vẽ lại cây để file/folder đã xóa biến mất khỏi danh sách

---

## 10. Hàm M.create()

```lua
function M.create()
```
**Chức năng:** Tạo file hoặc folder mới trong thư mục root.

**Không có tham số**

**Không trả về giá trị**

---

### Chi Tiết

```lua
local name = vim.fn.input("New file/folder: ")
```
- `vim.fn.input()`: Hiển thị prompt để user nhập tên
- `"New file/folder: "`: Text prompt
- `local name`: Lưu tên user nhập vào

---

```lua
if name == "" then return end
```
- Kiểm tra user có hủy không
- Thoát hàm nếu không nhập gì

---

```lua
local path = M.tree.fullpath .. "/" .. name
```
- Tạo đường dẫn đầy đủ: thư mục root + `/` + tên mới
- `M.tree.fullpath`: Đường dẫn của root folder

---

```lua
if name:match("/$") then
```
**Chức năng:** Phân biệt folder và file bằng dấu `/` ở cuối

**Chi tiết:**
- `name:match("/$")`: Pattern matching Lua
  - `"/$"`: Pattern tìm dấu `/` ở cuối string (`$` = end of string)
  - Trả về match nếu tìm thấy, nil nếu không
- `if ... then`: True khi tên kết thúc bằng `/` → user muốn tạo folder

---

```lua
vim.fn.mkdir(path, "p")
```
- `vim.fn.mkdir()`: Hàm Vim để tạo thư mục
- Tham số:
  - `path`: Đường dẫn thư mục cần tạo
  - `"p"`: Flag giống `mkdir -p` trong Linux - tạo cả thư mục cha nếu chưa tồn tại

---

```lua
else
```
- Nhánh else: User muốn tạo file (không có `/` ở cuối)

---

```lua
local f = io.open(path, "w")
```
- `io.open(path, "w")`: Hàm Lua standard library để mở file
- Tham số:
  - `path`: Đường dẫn file
  - `"w"`: Mode write - tạo file mới nếu chưa tồn tại
- `local f`: File handle, nil nếu lỗi

---

```lua
if f then f:close() end
```
- `if f then`: Kiểm tra file có mở thành công không
- `f:close()`: Đóng file handle
- Tạo file rỗng (mở rồi đóng ngay)

---

```lua
end
```
- Kết thúc khối if/else

---

```lua
M.tree.children = scan_dir(M.tree.fullpath)
```
- Quét lại thư mục root để cập nhật danh sách file/folder
- Thêm file/folder mới vào cấu trúc cây

---

```lua
render_tree()
```
- Vẽ lại cây để hiển thị file/folder mới

---

## 11. Hàm setup_autorefresh()

```lua
local function setup_autorefresh()
```
**Chức năng:** Thiết lập auto refresh - tự động cập nhật cây khi có thay đổi.

**Không có tham số**

**Không trả về giá trị**

---

### Chi Tiết

```lua
vim.api.nvim_create_autocmd({"CursorHold", "BufWritePost"}, {
```
**Chức năng:** Tạo autocmd để lắng nghe events

**Chi tiết:**
- `vim.api.nvim_create_autocmd()`: API để tạo autocmd (auto command)
- Tham số 1: `{"CursorHold", "BufWritePost"}` - mảng các event:
  - `"CursorHold"`: Trigger khi cursor không di chuyển trong một khoảng thời gian (default 4s)
  - `"BufWritePost"`: Trigger sau khi save file (`:w`)

---

```lua
callback = function()
```
- `callback = function()`: Định nghĩa hàm sẽ được gọi khi event trigger

---

```lua
if M.win and vim.api.nvim_win_is_valid(M.win) then
```
- Kiểm tra window explorer có đang mở và valid không
- Chỉ refresh khi explorer đang hiển thị

---

```lua
M.tree.children = scan_dir(M.tree.fullpath)
```
- Quét lại thư mục root để cập nhật danh sách file mới nhất

---

```lua
render_tree()
```
- Vẽ lại cây với dữ liệu mới

---

```lua
end -- kết thúc if
end -- kết thúc callback function
})
```
- Kết thúc callback và autocmd

---

## 12. Hàm setup_mouse()

```lua
local function setup_mouse()
```
**Chức năng:** Thiết lập hỗ trợ click chuột.

**Không có tham số**

**Không trả về giá trị**

---

### Chi Tiết

```lua
vim.api.nvim_buf_set_keymap(M.buf, "n", "<LeftMouse>",
    ":lua require'myexplorer'.open_or_expand()<CR>",
    { noremap = true, silent = true })
```
**Chức năng:** Map click chuột trái thành hành động open/expand

**Chi tiết:**
- `vim.api.nvim_buf_set_keymap()`: API để set keymap cho buffer
- Tham số:
  - `M.buf`: Buffer ID của explorer
  - `"n"`: Normal mode
  - `"<LeftMouse>"`: Key sequence - click chuột trái
  - `":lua require'myexplorer'.open_or_expand()<CR>"`: Command sẽ thực thi:
    - `:lua`: Chạy code Lua
    - `require'myexplorer'`: Load module myexplorer
    - `.open_or_expand()`: Gọi function
    - `<CR>`: Carriage return (Enter)
  - `{ noremap = true, silent = true }`: Options:
    - `noremap = true`: Không recursive mapping
    - `silent = true`: Không hiển thị command trong command line

---

## 13. Hàm M.open()

```lua
function M.open()
```
**Chức năng:** Mở explorer window/sidebar.

**Không có tham số**

**Không trả về giá trị**

---

### Chi Tiết Từng Khối

#### Kiểm Tra Explorer Đã Mở
```lua
if M.win and vim.api.nvim_win_is_valid(M.win) then
    vim.api.nvim_set_current_win(M.win)
    return
end
```
- `M.win and vim.api.nvim_win_is_valid(M.win)`: Kiểm tra window explorer còn tồn tại không
- `vim.api.nvim_set_current_win(M.win)`: Focus lại window explorer
- `return`: Thoát hàm, không tạo split mới
- Mục đích: Tránh tạo nhiều sidebar khi gọi `open()` nhiều lần

---

#### Tạo Sidebar
```lua
vim.cmd("topleft vsplit | vertical resize 30")
```
**Chức năng:** Tạo vertical split ở bên trái với width 30

**Chi tiết:**
- `vim.cmd()`: Thực thi Vim command
- `"topleft vsplit"`: Tạo vertical split ở vị trí top-left
- `|`: Command separator
- `"vertical resize 30"`: Resize width thành 30 columns

---

#### Lưu Window và Tạo Buffer
```lua
M.win = vim.api.nvim_get_current_win()
```
- `vim.api.nvim_get_current_win()`: Lấy ID của window hiện tại (vừa tạo)
- `M.win = ...`: Lưu ID vào module variable

---

```lua
M.buf = vim.api.nvim_create_buf(false, true)
```
- `vim.api.nvim_create_buf()`: API tạo buffer mới
- Tham số:
  - `false`: `listed` - buffer không xuất hiện trong buffer list
  - `true`: `scratch` - buffer tạm, không gắn với file
- `M.buf = ...`: Lưu buffer ID

---

```lua
vim.api.nvim_win_set_buf(M.win, M.buf)
```
- `vim.api.nvim_win_set_buf()`: Set buffer cho window
- Hiển thị buffer `M.buf` trong window `M.win`

---

#### Cấu Hình Buffer
```lua
vim.api.nvim_buf_set_option(M.buf, "buftype", "nofile")
```
- `vim.api.nvim_buf_set_option()`: Set option cho buffer
- `"buftype"`: Loại buffer
- `"nofile"`: Buffer không gắn với file trên disk

---

```lua
vim.api.nvim_buf_set_option(M.buf, "bufhidden", "wipe")
```
- `"bufhidden"`: Hành động khi buffer bị hide
- `"wipe"`: Xóa buffer hoàn toàn khi bị hide (giải phóng memory)

---

```lua
vim.api.nvim_buf_set_option(M.buf, "modifiable", true)
```
- `"modifiable"`: Buffer có thể chỉnh sửa không
- `true`: Cho phép chỉnh sửa (cần thiết cho `nvim_buf_set_lines`)

---

```lua
vim.api.nvim_buf_set_option(M.buf, "swapfile", false)
```
- `"swapfile"`: Có tạo swap file không
- `false`: Không tạo swap file (không cần cho buffer tạm)

---

#### Tắt Số Dòng
```lua
vim.api.nvim_win_set_option(M.win, "number", false)
vim.api.nvim_win_set_option(M.win, "relativenumber", false)
vim.api.nvim_win_set_option(M.win, "signcolumn", "no")
```
- `vim.api.nvim_win_set_option()`: Set option cho window
- `"number", false`: Tắt số dòng thường
- `"relativenumber", false`: Tắt số dòng tương đối
- `"signcolumn", "no"`: Ẩn sign column (cột cho git signs, diagnostics)
- Mục đích: Giao diện sidebar gọn gàng hơn

---

#### Khởi Tạo Root Tree
```lua
M.tree = {
    name = vim.fn.getcwd(),
    fullpath = vim.fn.getcwd(),
    is_dir = true,
    expanded = true,
    children = scan_dir(vim.fn.getcwd())
}
```
**Chức năng:** Tạo root node của cây

**Chi tiết:**
- `vim.fn.getcwd()`: Lấy current working directory (thư mục hiện tại)
- Tạo table với:
  - `name`: Tên = đường dẫn đầy đủ
  - `fullpath`: Đường dẫn đầy đủ
  - `is_dir = true`: Là folder
  - `expanded = true`: Mở sẵn
  - `children = scan_dir(...)`: Quét ngay các file/folder con

---

#### Setup Keymaps

##### Enter để Open/Expand
```lua
vim.api.nvim_buf_set_keymap(M.buf, "n", "<CR>",
    ":lua require'myexplorer'.open_or_expand()<CR>",
    { noremap = true, silent = true })
```
- Map phím Enter (`<CR>`) gọi `open_or_expand()`

---

##### 'r' để Rename
```lua
vim.api.nvim_buf_set_keymap(M.buf, "n", "r",
    ":lua require'myexplorer'.rename()<CR>",
    { noremap = true, silent = true })
```
- Map phím `r` gọi `rename()`

---

##### 'd' để Delete
```lua
vim.api.nvim_buf_set_keymap(M.buf, "n", "d",
    ":lua require'myexplorer'.delete()<CR>",
    { noremap = true, silent = true })
```
- Map phím `d` gọi `delete()`

---

##### 'a' để Create
```lua
vim.api.nvim_buf_set_keymap(M.buf, "n", "a",
    ":lua require'myexplorer'.create()<CR>",
    { noremap = true, silent = true })
```
- Map phím `a` gọi `create()`

---

#### Hoàn Tất Setup
```lua
setup_mouse()
```
- Gọi hàm setup mouse support

---

```lua
setup_autorefresh()
```
- Gọi hàm setup auto refresh

---

```lua
render_tree()
```
- Vẽ cây lần đầu tiên để hiển thị nội dung

---

## 14. Hàm M.toggle()

```lua
function M.toggle()
```
**Chức năng:** Toggle explorer - mở nếu đang đóng, đóng nếu đang mở.

**Không có tham số**

**Không trả về giá trị**

---

### Chi Tiết

```lua
if M.win and vim.api.nvim_win_is_valid(M.win) then
```
- Kiểm tra explorer có đang mở không

---

```lua
vim.api.nvim_win_close(M.win, true)
```
- `vim.api.nvim_win_close()`: API đóng window
- Tham số:
  - `M.win`: Window ID cần đóng
  - `true`: Force close (không hỏi confirm)

---

```lua
M.win = nil
M.buf = nil
```
- Reset các biến về nil
- Đánh dấu explorer đã bị đóng

---

```lua
return
```
- Thoát hàm (explorer đã đóng)

---

```lua
end
```
- Kết thúc khối if (explorer đang mở)

---

```lua
M.open()
```
- Nếu không vào khối if → explorer đang đóng
- Gọi `M.open()` để mở explorer

---

## 15. Return Module

```lua
return M
```
**Chức năng:** Return module M để có thể require từ nơi khác

**Chi tiết:**
- `return M`: Return table M chứa tất cả các function và state
- Cho phép sử dụng: `local explorer = require('myexplorer')`
- Truy cập functions: `explorer.open()`, `explorer.toggle()`, etc.

---

## Tổng Kết Sử Dụng

### Cách Sử Dụng Plugin

1. **Mở Explorer:**
   ```lua
   :lua require('myexplorer').open()
   ```

2. **Toggle Explorer:**
   ```lua
   :lua require('myexplorer').toggle()
   ```

3. **Trong Explorer:**
   - `Enter` hoặc Click chuột: Mở file hoặc expand/collapse folder
   - `r`: Rename file/folder
   - `d`: Delete file/folder
   - `a`: Tạo file/folder mới
     - Tên kết thúc bằng `/`: Tạo folder
     - Tên không có `/`: Tạo file

### Keymap Đề Xuất

Thêm vào `init.vim` hoặc `init.lua`:

```lua
-- Map Ctrl+B để toggle explorer
vim.keymap.set('n', '<C-b>', ':lua require("myexplorer").toggle()<CR>', { noremap = true, silent = true })
```

### Cấu Trúc Dữ Liệu Node

Mỗi node trong cây có structure:
```lua
{
    name = "filename.txt",           -- Tên file/folder
    fullpath = "/path/to/file.txt",  -- Đường dẫn đầy đủ
    is_dir = false,                  -- true = folder, false = file
    expanded = false,                -- true = đang mở, false = đang đóng
    children = {}                    -- Mảng các node con (nếu là folder)
}
```

---

## Các Tối Ưu và Best Practices

1. **Lazy Loading:** Children chỉ được load khi folder được expand lần đầu
2. **Buffer Management:** Tự động xử lý buffer khi rename/delete file
3. **Auto Refresh:** Tự động cập nhật khi có thay đổi (CursorHold hoặc BufWritePost)
4. **Error Handling:** Kiểm tra nil trước khi truy cập node
5. **Singleton Pattern:** Chỉ một explorer window duy nhất tại một thời điểm

---

## Giới Hạn và Cải Tiến Có Thể

### Giới Hạn Hiện Tại:
- `os.remove()` chỉ xóa được file hoặc folder rỗng
- Không hỗ trợ copy/paste
- Không hỗ trợ move file
- Không có search function

### Cải Tiến Đề Xuất:
1. Sử dụng `vim.fn.delete()` thay vì `os.remove()` để xóa folder có nội dung
2. Thêm function copy/paste
3. Thêm function move/cut
4. Thêm search/filter
5. Thêm git integration (hiển thị git status)
6. Thêm custom icons dựa trên file extension

---

*Tài liệu này được tạo để giải thích chi tiết cách hoạt động của MyExplorer plugin.*
