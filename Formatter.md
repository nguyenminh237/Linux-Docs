# Tài Liệu Chi Tiết - Custom Code Formatter

## Tổng Quan

Đây là một formatter tự viết hoàn toàn bằng Lua, không phụ thuộc vào công cụ bên ngoài như `clang-format` hay LSP. Formatter này hỗ trợ các ngôn ngữ C, C++, và Java.

### Tính Năng Chính
- ✅ Tự động indent code dựa trên dấu `{` và `}`
- ✅ Tách statements minified (code viết trên 1 dòng) thành nhiều dòng
- ✅ Thêm space quanh các operators (`=`, `+`, `-`, `*`, `/`)
- ✅ Xử lý đúng string và comment (không format nội dung bên trong)
- ✅ Xử lý đặc biệt cho `for` loop (không tách `;` trong `for(...)`)

---

## 1. Module và Config

### Module M
```lua
local M = {}
```
**Giải thích:**
- `local M = {}` - Tạo một table (giống object/dictionary) rỗng tên là `M`
- `local` có nghĩa biến này chỉ tồn tại trong file này, không leak ra ngoài
- Table `M` sẽ chứa tất cả các function của formatter
- Cuối file sẽ `return M` để file khác có thể dùng formatter này

---

### Config Object
```lua
local config = {
    indent_size = 4,
    use_spaces = true,
    space_around_ops = true,
    split_statements = true,
}
```

**Giải thích từng field:**

```lua
indent_size = 4,  -- Mỗi level indent thêm 4 khoảng trắng (hoặc 4 tab nếu use_spaces = false)
```

```lua
use_spaces = true,  -- true = dùng space để indent, false = dùng tab character (\t)
```

```lua
space_around_ops = true,  -- true = thêm space quanh operators: x=5 → x = 5
```

```lua
split_statements = true,  -- true = tách code minified thành nhiều dòng: {x();y();} → xuống dòng
```

**Ví dụ minh họa:**
- Với `indent_size = 4` và `use_spaces = true`:
  ```
  int main() {
  ....for(...) {  // 4 spaces
  ........x = 5;  // 8 spaces
  ....}
  }
  ```

---

## 2. Hàm calculate_indent_level()

### Chức Năng
Đếm số dấu `{` và `}` trong một dòng để biết cần tăng/giảm indent bao nhiêu level.

### Signature
```lua
local function calculate_indent_level(line)
```
- `local function` - Hàm local chỉ dùng trong file này
- `line` - Tham số: một string chứa một dòng code
- Trả về: `open_braces, close_braces` (hai số nguyên)

---

### Chi Tiết Từng Khối Code

#### Khởi Tạo Biến Đếm
```lua
local open_braces = 0      -- Đếm số dấu {
local close_braces = 0     -- Đếm số dấu }
local in_string = false    -- Flag: đang ở trong string hay không (để bỏ qua { } trong "text {}")
local in_comment = false   -- Flag: đang ở trong comment /* */ hay không
```

**Tại sao cần flag?**
- String và comment có thể chứa `{` hoặc `}` nhưng KHÔNG phải là code block
- Ví dụ: `cout << "Hello {world}";` - Cặp `{}` này không phải block code
- Ví dụ: `/* TODO: fix {bug} */` - `{}` trong comment cũng không tính

---

#### Vòng Lặp Duyệt Từng Ký Tự
```lua
for i = 1, #line do
```
- `for i = 1, #line` - Lặp từ vị trí 1 đến độ dài của string `line`
- **LƯU Ý:** Trong Lua, index của string bắt đầu từ **1** (không phải 0 như C/Java)
- `#line` - Operator `#` trả về độ dài của string

**Ví dụ:**
```lua
local text = "hello"
#text  -- trả về 5
```

---

```lua
local char = line:sub(i, i)         -- Lấy ký tự tại vị trí i
local next_char = line:sub(i+1, i+1) -- Lấy ký tự tiếp theo (i+1)
```

**Giải thích `string:sub(start, end)`:**
- `sub` là method của string trong Lua
- `line:sub(i, i)` nghĩa là "lấy substring từ vị trí i đến i" → 1 ký tự
- `line:sub(i+1, i+1)` → ký tự tiếp theo
- Dùng để detect các pattern 2 ký tự như `//` hoặc `/*`

**Ví dụ:**
```lua
local text = "hello"
text:sub(1, 1)  -- "h"
text:sub(2, 2)  -- "e"
text:sub(1, 3)  -- "hel"
```

---

#### Detect Comment Một Dòng (`//`)
```lua
if char == '/' and next_char == '/' then
    break  -- Gặp // → Phần còn lại là comment → Không cần kiểm tra nữa
end
```
- `char == '/'` - Kiểm tra ký tự hiện tại có phải `/` không
- `next_char == '/'` - Kiểm tra ký tự tiếp theo cũng là `/`
- Nếu cả 2 đúng → Đây là comment `//` → Phần còn lại của dòng không quan trọng
- `break` - Thoát khỏi vòng lặp ngay lập tức

**Ví dụ:**
```
int x = 5; // { này là comment, không tính
```
Khi gặp `//`, dừng luôn nên `{` trong comment không bị đếm.

---

#### Detect Bắt Đầu Comment Block (`/*`)
```lua
if char == '/' and next_char == '*' then
    in_comment = true  -- Bật flag "đang trong comment block"
end
```
- Tương tự trên, detect pattern `/*`
- Set flag `in_comment = true` để các ký tự tiếp theo được bỏ qua

**Ví dụ:**
```
/* This is a {block} comment */
```
`{block}` không bị tính vì `in_comment = true`

---

#### Detect Kết Thúc Comment Block (`*/`)
```lua
if in_comment and char == '*' and next_char == '/' then
    in_comment = false  -- Tắt flag "đang trong comment"
    goto continue       -- Skip sang iteration tiếp theo
end
```
- `in_comment and ...` - Chỉ check khi đang ở trong comment
- Detect pattern `*/` → Kết thúc comment block
- `goto continue` - Nhảy đến label `::continue::` ở cuối vòng lặp (skip phần còn lại)

**Tại sao dùng `goto`?**
- Trong Lua không có `continue` keyword như C/Java
- `goto continue` là cách để skip phần còn lại và chuyển sang iteration tiếp theo

---

#### Detect String (`"`)
```lua
if char == '"' and not in_comment then
    in_string = not in_string  -- Toggle flag string: false → true → false
end
```
- `char == '"'` - Gặp dấu ngoặc kép
- `not in_comment` - Chỉ xử lý nếu KHÔNG đang trong comment (tránh trường hợp `/* "text" */`)
- `not in_string` - Toán tử NOT (đảo ngược boolean)
  - Nếu `in_string = false` → `not in_string = true`
  - Nếu `in_string = true` → `not in_string = false`

**Logic Toggle:**
- Lần 1 gặp `"` → `in_string = not false = true` (vào string)
- Lần 2 gặp `"` → `in_string = not true = false` (ra khỏi string)

**Ví dụ:**
```lua
cout << "Hello {world}" << endl;
         ^             ^
       vào string    ra khỏi string
```

**LƯU Ý:** Code này không handle escaped quote `\"` - đây là limitation đơn giản hóa

---

#### Đếm Dấu Ngoặc `{` và `}`
```lua
if not in_string and not in_comment then  -- Chỉ đếm khi KHÔNG ở trong string/comment
    if char == '{' then
        open_braces = open_braces + 1     -- Tăng biến đếm { lên 1
    elseif char == '}' then
        close_braces = close_braces + 1   -- Tăng biến đếm } lên 1
    end
end
```

**Logic:**
- `not in_string and not in_comment` - Chỉ đếm khi cả 2 flag đều false
- `open_braces = open_braces + 1` - Tăng giá trị biến lên 1 (trong Lua không có `++`)
- `elseif` - Else + If (giống `else if` trong C/Java)

**Ví dụ Minh Họa:**

| Dòng Code | `open_braces` | `close_braces` |
|-----------|---------------|----------------|
| `int main() {` | 1 | 0 |
| `    if (x) { y(); }` | 2 | 2 (1 open, 1 close) |
| `}` | 0 | 1 |

---

#### Label `::continue::`
```lua
::continue::
```
- Đây là một **label** trong Lua (giống label trong C)
- Được dùng làm đích cho `goto continue`
- Khi gọi `goto continue`, chương trình nhảy đến đây và tiếp tục vòng lặp

**Tại sao cần?**
- Lua không có `continue` như C/Java
- `goto` với label là cách để skip phần còn lại của iteration hiện tại

---

#### Return Kết Quả
```lua
return open_braces, close_braces
```
- `return` nhiều giá trị cùng lúc (Lua hỗ trợ multiple return values)
- Hàm gọi có thể nhận như sau:
  ```lua
  local open, close = calculate_indent_level("int main() {")
  -- open = 1, close = 0
  ```

---

## 3. Hàm trim()

### Chức Năng
Xóa tất cả khoảng trắng (space, tab, newline) ở đầu và cuối string.

### Code
```lua
local function trim(str)
    return str:match("^%s*(.-)%s*$")
end
```

---

### Giải Thích Chi Tiết

```lua
str:match("^%s*(.-)%s*$")
```

**`string:match(pattern)`:**
- Method của string trong Lua để tìm pattern (regex-like)
- Trả về phần text khớp với pattern

**Pattern `^%s*(.-)%s*$` giải thích:**
- `^` - Bắt đầu string
- `%s` - Ký tự whitespace (space, tab, newline, etc.)
- `%s*` - Zero hoặc nhiều whitespace (`*` = 0 or more)
- `(.-)` - **Capture group**: bắt bất kỳ ký tự nào (`.-` = lazy match, ít nhất có thể)
- `%s*` - Zero hoặc nhiều whitespace ở cuối
- `$` - Kết thúc string

**Logic:**
1. Bỏ qua tất cả whitespace ở đầu (`^%s*`)
2. Bắt phần content giữa (`(.-)`)
3. Bỏ qua tất cả whitespace ở cuối (`%s*$`)
4. Trả về phần content giữa

**Ví dụ:**
```lua
trim("   hello world   ")  -- "hello world"
trim("\t\tint x = 5;\n")   -- "int x = 5;"
trim("abc")                -- "abc"
trim("   ")                -- ""
```

---

## 4. Hàm normalize_spaces()

### Chức Năng
Thêm space quanh các toán tử (operators) để code dễ đọc hơn.

### Signature
```lua
local function normalize_spaces(line)
```
- Tham số: `line` - Một dòng code (string)
- Trả về: Dòng code đã được chuẩn hóa space

---

### Chi Tiết Code

#### Kiểm Tra Config
```lua
if not config.space_around_ops then
    return line  -- Nếu config tắt tính năng này, return nguyên dòng không thay đổi
end
```
- `not config.space_around_ops` - Nếu config = false
- `return line` - Thoát hàm sớm, không xử lý gì

---

#### Xử Lý Toán Tử `=` (Assignment)
```lua
line = line:gsub("([^=!<>])=([^=])", "%1 = %2")
```

**`string:gsub(pattern, replacement)`:**
- Method để find & replace trong string (giống `str.replace()` trong JS)
- `line = line:gsub(...)` - Gán kết quả replace vào lại biến `line`

**Pattern `([^=!<>])=([^=])` giải thích:**
- `([^=!<>])` - **Capture group 1**: Bắt 1 ký tự KHÔNG phải `=`, `!`, `<`, `>`
  - `[^...]` nghĩa là "NOT in this set"
  - Dùng để tránh replace `==`, `!=`, `<=`, `>=`
- `=` - Ký tự `=` (assignment operator)
- `([^=])` - **Capture group 2**: Bắt 1 ký tự KHÔNG phải `=`
  - Tránh replace `==` thành `= =`

**Replacement `%1 = %2`:**
- `%1` - Thay bằng nội dung của capture group 1
- ` = ` - Thêm space trước và sau `=`
- `%2` - Thay bằng nội dung của capture group 2

**Ví dụ:**
```lua
"x=5"      → "x = 5"
"x==5"     → "x==5" (không đổi vì == match được exclude)
"x!=5"     → "x!=5" (không đổi)
```

---

#### Xử Lý Toán Tử `+`
```lua
line = line:gsub("([^+])%+([^+])", "%1 + %2")
```

**Pattern `([^+])%+([^+])` giải thích:**
- `([^+])` - Capture 1: Ký tự KHÔNG phải `+`
- `%+` - Ký tự `+` (phải escape bằng `%` vì `+` là special character trong pattern)
- `([^+])` - Capture 2: Ký tự KHÔNG phải `+`
- Logic tránh replace `++` thành `+ +`

**Ví dụ:**
```lua
"x+y"      → "x + y"
"x++"      → "x++" (không đổi)
"++x"      → "++x" (không đổi)
```

---

#### Xử Lý Toán Tử `-`
```lua
line = line:gsub("([^-])%-([^->])", "%1 - %2")
```

**Pattern `([^-])%-([^->])` giải thích:**
- `([^-])` - Capture 1: Ký tự KHÔNG phải `-`
- `%-` - Ký tự `-` (escape bởi `%`)
- `([^->])` - Capture 2: Ký tự KHÔNG phải `-` hoặc `>`
  - Tránh replace `--` và `->`

**Ví dụ:**
```lua
"x-y"      → "x - y"
"x--"      → "x--" (không đổi)
"ptr->val" → "ptr->val" (không đổi, giữ nguyên arrow operator)
```

---

#### Xử Lý Toán Tử `*`
```lua
line = line:gsub("([^*])%*([^*])", "%1 * %2")
```
- Tương tự như trên
- `%*` - Escape ký tự `*`
- Tránh replace `**` (pointer-to-pointer hoặc exponentiation)

**Ví dụ:**
```lua
"x*y"      → "x * y"
"int **p"  → "int **p" (không đổi)
```

---

#### Xử Lý Toán Tử `/`
```lua
line = line:gsub("([^/])/([^/])", "%1 / %2")
```
- Tránh replace `//` (comment) thành `/ /`

**Ví dụ:**
```lua
"x/y"      → "x / y"
"// comment" → "// comment" (không đổi)
```

---

#### Collapse Multiple Spaces
```lua
line = line:gsub("%s+", " ")
```

**Pattern `%s+`:**
- `%s` - Whitespace
- `+` - One or more (1 hoặc nhiều)
- Replace bằng một space duy nhất

**Tại sao cần?**
- Sau các replace trên, có thể xuất hiện nhiều space liên tiếp
- Ví dụ: `x  =  y` (2 space) → `x = y` (1 space)

**Ví dụ:**
```lua
"x    =    y"   → "x = y"
"int     x"     → "int x"
```

---

#### Return Kết Quả
```lua
return line
```
- Trả về dòng đã được normalize

---

## 5. Hàm split_statements()

### Chức Năng
Tách một dòng code minified (nhiều statement trên 1 dòng) thành nhiều statement riêng biệt.

### Ví Dụ Input/Output
```lua
Input:  "int main(){for(int i=0;i<10;i++){cout<<i;}return 0;}"

Output: {
  "int main(){",
  "for(int i=0;i<10;i++){",
  "cout<<i;",
  "}",
  "return 0;",
  "}"
}
```

---

### Signature
```lua
local function split_statements(line)
```
- Tham số: `line` - Một dòng code có thể chứa nhiều statement
- Trả về: Một array (table) chứa các statement riêng biệt

---

### Chi Tiết Code

#### Kiểm Tra Config
```lua
if not config.split_statements then
    return {line}  -- Nếu tắt tính năng, return array chứa 1 phần tử là dòng gốc
end
```
- `{line}` - Tạo table (array) với 1 phần tử là `line`
- Trong Lua, array là table với key là số nguyên bắt đầu từ 1

---

#### Comment Giải Thích Logic
```lua
-- NOTE:
-- Trước đây chỉ tách theo ';' và có tối ưu sớm (<= 1 ';' thì return).
-- Điều đó làm các dòng "minify" kiểu `if(x){y();}` hoặc
-- `int main(){for(...){...}return 0;}` không bung layout như mong đợi.
-- Giờ tách theo cả `{` và `}` (ngoài string/comment), và vẫn bảo vệ
-- không tách ';' bên trong ngoặc đơn bằng paren_depth.
```
- Đây là comment trong code giải thích tại sao cần tách theo `{`, `}`, và `;`
- Trong Lua, comment đơn dòng bắt đầu bằng `--`
- Comment nhiều dòng: `--[[ comment ]]`

---

#### Khởi Tạo Biến
```lua
local statements = {}       -- Array chứa các statement đã tách
local current = ""          -- String tạm để build statement hiện tại
local in_string = false     -- Flag: đang trong string
local in_comment = false    -- Flag: đang trong comment
local paren_depth = 0       -- Đếm độ sâu ngoặc đơn ()
```

**Giải thích `paren_depth`:**
- Dùng để track xem đang ở trong `(...)` hay không
- Khi gặp `(` → tăng 1
- Khi gặp `)` → giảm 1
- Khi `paren_depth > 0` → đang trong `(...)`

**Tại sao cần?**
- Trong `for(int i=0; i<10; i++)`, có 2 dấu `;` nhưng KHÔNG nên tách
- Dùng `paren_depth` để detect xem `;` có nằm trong `()` hay không

---

#### Vòng Lặp Duyệt Từng Ký Tự
```lua
for i = 1, #line do
    local char = line:sub(i, i)         -- Ký tự hiện tại
    local next_char = line:sub(i+1, i+1) -- Ký tự tiếp theo
```
- Giống như trong `calculate_indent_level()`

---

#### Xử Lý Comment `//`
```lua
-- Track comment
if char == '/' and next_char == '/' and not in_string then
    current = current .. line:sub(i)  -- Append phần còn lại của dòng vào current
    break                             -- Thoát vòng lặp (phần sau // không quan trọng)
end
```

**`current = current .. line:sub(i)`:**
- `..` - Toán tử concatenate string trong Lua (giống `+` trong Java/JS)
- `line:sub(i)` - Lấy substring từ vị trí `i` đến hết dòng (không có end parameter)
- Ví dụ: `"hello":sub(3)` → `"llo"`

**Logic:**
- Gặp `//` → Phần còn lại là comment
- Append toàn bộ vào `current`
- `break` để thoát vòng lặp

---

#### Xử Lý String `"`
```lua
-- Track string
if char == '"' and not in_comment then
    in_string = not in_string  -- Toggle flag
end
```
- Tương tự như trong `calculate_indent_level()`

---

#### Track Parentheses Depth
```lua
-- Track parentheses (để detect for loop)
if not in_string and not in_comment then
    if char == '(' then
        paren_depth = paren_depth + 1  -- Vào trong ()
    elseif char == ')' then
        paren_depth = paren_depth - 1  -- Ra khỏi ()
    end
end
```

**Ví dụ Minh Họa:**
```
for(int i=0; i<10; i++)
   ^        ^       ^
   depth=1  depth=1 depth=0 (đóng ngoặc)
```

---

#### Append Ký Tự Vào Current
```lua
current = current .. char  -- Thêm ký tự hiện tại vào statement đang build
```

---

#### Logic Tách Statement Theo `{` hoặc `}`
```lua
if not in_string and not in_comment then
    -- Tách khi gặp '{' hoặc '}' nếu sau đó còn code (hoặc để '}' đứng riêng)
    if (char == '{' or char == '}') and paren_depth == 0 then
        local after = trim(line:sub(i + 1))  -- Lấy phần sau ký tự hiện tại
```

**`char == '{' or char == '}'`:**
- `or` - Toán tử logic OR
- Đúng nếu ký tự hiện tại là `{` hoặc `}`

**`paren_depth == 0`:**
- Chỉ tách khi KHÔNG trong `()`
- Tránh tách nhầm trong trường hợp `if (...) {` (có thể có `{` bên trong condition)

**`trim(line:sub(i + 1))`:**
- `line:sub(i + 1)` - Lấy phần còn lại SAU ký tự hiện tại
- `trim(...)` - Xóa whitespace để check xem còn code không

---

```lua
        -- Luôn đẩy statement chứa brace hiện tại ra (nếu có nội dung)
        if trim(current) ~= "" then
            table.insert(statements, trim(current))  -- Thêm vào array
            current = ""                             -- Reset để build statement mới
        end
```

**`table.insert(array, value)`:**
- Hàm built-in của Lua để thêm phần tử vào cuối array
- `statements` là array (table)
- `trim(current)` là giá trị cần thêm

**`current = ""`:**
- Reset string về rỗng để bắt đầu build statement tiếp theo

**Ví dụ:**
```
Input: "int main(){return 0;}"
              ↓
Gặp '{', current = "int main(){"
Push "int main(){" vào array
Reset current = ""
```

---

```lua
        -- Nếu không còn gì phía sau thì kết thúc
        if after == "" then
            break  -- Thoát vòng lặp
        end
    end
```
- Nếu sau `{` hoặc `}` không còn code → Thoát luôn
- Tiết kiệm thời gian không cần duyệt phần rỗng

---

#### Logic Tách Statement Theo `;`
```lua
    -- Nếu gặp ';' và KHÔNG trong parentheses => tách statement
    if char == ';' and paren_depth == 0 then
        local after = trim(line:sub(i + 1))
```

**`paren_depth == 0`:**
- QUAN TRỌNG: Chỉ tách khi `;` KHÔNG nằm trong `()`
- Điều này bảo vệ `for(int i=0; i<10; i++)` không bị tách nhầm

**Ví dụ:**
```
for(int i=0; i<10; i++) { x(); }
           ^       ^       ^
        không tách  không tách  TÁCH (paren_depth = 0)
```

---

```lua
        if trim(current) ~= "" then
            table.insert(statements, trim(current))
            current = ""
        end

        if after == "" then
            break
        end
    end
end
```
- Logic tương tự như phần tách theo `{` và `}`

---

#### Thêm Phần Còn Lại
```lua
-- Thêm phần còn lại
if trim(current) ~= "" then
    table.insert(statements, trim(current))
end
```
- Sau vòng lặp, có thể còn code trong `current` chưa được push
- Ví dụ: `int x = 5;` (dòng kết thúc bằng `;`, sau đó là rỗng)
  - Sau khi xử lý `;`, `current = ""` nhưng statement đã được push
  - Block này handle case như `int x = 5` (KHÔNG có `;` ở cuối)

---

#### Return Kết Quả
```lua
-- Nếu không tách được gì → return original
if #statements == 0 then
    return {line}  -- Return array chứa dòng gốc
end

return statements
```

**`#statements`:**
- Operator `#` trả về số phần tử trong array
- `#statements == 0` nghĩa là array rỗng (không tách được gì)

**Tại sao cần check?**
- Nếu dòng không có `;`, `{`, `}` nào → Array rỗng
- Return `{line}` để đảm bảo luôn trả về ít nhất 1 statement

---

## 6. Hàm M.format_buffer()

### Chức Năng
Hàm chính để format toàn bộ buffer (file đang mở) của Neovim.

### Signature
```lua
function M.format_buffer()
```
- `function M.format_buffer()` - Gán hàm vào field `format_buffer` của table `M`
- Không có `local` → Hàm này public, có thể gọi từ bên ngoài
- Không có tham số (sẽ lấy buffer hiện tại của Neovim)

---

### Chi Tiết Code

#### Kiểm Tra Filetype
```lua
local filetype = vim.bo.filetype
```

**`vim.bo.filetype`:**
- `vim` - Global object của Neovim Lua API
- `bo` - Buffer Options (các setting của buffer hiện tại)
- `filetype` - Loại file (c, cpp, java, python, etc.)

**Ví dụ:**
- File `main.cpp` → `filetype = "cpp"`
- File `Main.java` → `filetype = "java"`
- File không extension → `filetype = ""`

---

```lua
if filetype ~= "c" and filetype ~= "cpp" and filetype ~= "java" then
    vim.notify("❌ Chỉ hỗ trợ C/C++/Java", vim.log.levels.WARN)
    return
end
```

**`~=`:**
- Toán tử "not equal" trong Lua (giống `!=` trong C/Java)

**`and`:**
- Toán tử logic AND
- Tất cả điều kiện phải đúng

**`vim.notify(message, level)`:**
- Hiển thị notification trong Neovim
- `vim.log.levels.WARN` - Level cảnh báo (màu vàng)
- Các level khác: `INFO`, `ERROR`, `DEBUG`

**`return`:**
- Thoát hàm sớm nếu filetype không hỗ trợ

---

#### Hiển Thị Notification Bắt Đầu
```lua
vim.notify("🔨 Đang format code...", vim.log.levels.INFO)
```
- Thông báo cho user biết formatter đang chạy
- `vim.log.levels.INFO` - Level thông tin (màu xanh/trắng)

---

#### Lấy Tất Cả Dòng Từ Buffer
```lua
local lines = vim.api.nvim_buf_get_lines(0, 0, -1, false)
```

**`vim.api.nvim_buf_get_lines(buffer, start, end, strict_indexing)`:**
- `vim.api` - Namespace cho Neovim API functions
- `nvim_buf_get_lines` - Hàm lấy dòng từ buffer
- `0` - Buffer number (0 = buffer hiện tại)
- `0` - Start line (0 = dòng đầu tiên)
- `-1` - End line (-1 = dòng cuối cùng)
- `false` - Strict indexing (không quan trọng trong case này)

**Trả về:**
- Array (table) chứa tất cả các dòng trong buffer
- Ví dụ: `{"#include <iostream>", "int main() {", "}"}`

---

#### Khởi Tạo Biến
```lua
local formatted = {}      -- Array chứa dòng đã format
local current_indent = 0  -- Indent level hiện tại (bắt đầu từ 0)
```

**`current_indent`:**
- Biến track indent level hiện tại
- Bắt đầu từ 0 (không indent)
- Mỗi `{` tăng 1, mỗi `}` giảm 1

---

#### Vòng Lặp Xử Lý Từng Dòng
```lua
for i, line in ipairs(lines) do
```

**`ipairs(array)`:**
- Iterator để duyệt array trong Lua
- Trả về `(index, value)` cho mỗi phần tử
- `i` - Index (1, 2, 3, ...)
- `line` - Nội dung dòng

**Ví dụ:**
```lua
local arr = {"hello", "world"}
for i, val in ipairs(arr) do
    print(i, val)  -- 1 hello
                   -- 2 world
end
```

---

#### Xử Lý Dòng Trống
```lua
    local trimmed = trim(line)  -- Xóa whitespace đầu/cuối
    
    if trimmed == "" then
        table.insert(formatted, "")  -- Giữ dòng trống
        goto continue                -- Skip sang dòng tiếp theo
    end
```

**Logic:**
- Nếu dòng chỉ chứa whitespace (hoặc rỗng)
- Thêm dòng trống vào kết quả
- Skip phần xử lý còn lại

**Tại sao giữ dòng trống?**
- Dòng trống giúp tách các block code, dễ đọc hơn
- Ví dụ: Tách giữa các hàm

---

#### Tách Statement
```lua
    -- MỚI: Tách statements trước khi format
    local statements = split_statements(trimmed)
```
- Gọi hàm `split_statements()` đã giải thích ở trên
- `trimmed` - Dòng đã xóa whitespace
- `statements` - Array chứa các statement riêng biệt

---

#### Vòng Lặp Xử Lý Từng Statement
```lua
    for _, stmt in ipairs(statements) do
        stmt = trim(stmt)  -- Trim lại để chắc chắn
```

**`_, stmt`:**
- `_` - Convention trong Lua để bỏ qua biến không dùng (index trong case này)
- `stmt` - Statement hiện tại

---

```lua
        if stmt == "" then
            goto continue_stmt  -- Skip statement rỗng
        end
```
- Bảo vệ: nếu statement rỗng (sau trim), skip luôn

---

#### Kiểm Tra Statement Bắt Đầu Bằng `}`
```lua
        -- Check nếu statement bắt đầu với }
        local starts_with_close = stmt:match("^}")
```

**`string:match("^}")`:**
- `^}` - Pattern: bắt đầu string là `}`
- Trả về `"}"` nếu match, `nil` nếu không match

**`starts_with_close`:**
- Có giá trị (truthy) nếu statement bắt đầu bằng `}`
- `nil` (falsy) nếu không

**Ví dụ:**
```lua
"}".match("^}")              -- "}" (truthy)
"} else {".match("^}")       -- "}" (truthy)
"return 0;".match("^}")      -- nil (falsy)
```

---

#### Giảm Indent Trước Khi Render `}`
```lua
        if starts_with_close then
            current_indent = math.max(0, current_indent - 1)
        end
```

**`math.max(0, value)`:**
- Hàm built-in trả về giá trị lớn nhất
- Đảm bảo indent không bao giờ < 0 (âm)

**Logic:**
- Nếu statement bắt đầu bằng `}` → Giảm indent TRƯỚC khi render
- Vì `}` đánh dấu kết thúc block, nên phải "unindent" trước

**Ví dụ:**
```
    if (x) {
        y();
    }          ← Dòng này phải unindent
```

---

#### Tạo Indent String
```lua
        -- Tạo indent
        local indent_str = ""
        if config.use_spaces then
            indent_str = string.rep(" ", current_indent * config.indent_size)
        else
            indent_str = string.rep("\t", current_indent)
        end
```

**`string.rep(str, n)`:**
- Hàm built-in repeat string `n` lần
- Ví dụ: `string.rep("x", 3)` → `"xxx"`

**Logic:**
- Nếu dùng space: Tạo `current_indent * indent_size` spaces
  - Ví dụ: `indent=2`, `size=4` → `"        "` (8 spaces)
- Nếu dùng tab: Tạo `current_indent` tabs
  - Ví dụ: `indent=2` → `"\t\t"`

---

#### Normalize Spaces
```lua
        -- Normalize spaces
        stmt = normalize_spaces(stmt)
```
- Gọi hàm `normalize_spaces()` để thêm space quanh operators

---

#### Thêm Dòng Vào Kết Quả
```lua
        -- Thêm line
        table.insert(formatted, indent_str .. stmt)
```

**`indent_str .. stmt`:**
- Concatenate indent + statement
- Ví dụ: `"    " .. "int x = 5;"` → `"    int x = 5;"`

---

#### Tính Indent Cho Dòng Tiếp Theo
```lua
        -- Tính indent cho dòng tiếp theo
        local open_count, close_count = calculate_indent_level(stmt)
```
- Gọi hàm `calculate_indent_level()` đếm `{` và `}`
- Nhận 2 giá trị return

---

#### Điều Chỉnh `close_count` Nếu Statement Bắt Đầu Bằng `}`
```lua
        -- Nếu statement bắt đầu bằng '}' thì ta đã giảm indent trước khi render.
        -- Vì calculate_indent_level() vẫn đếm '}' đó, cần trừ đi 1 để tránh giảm 2 lần.
        if starts_with_close then
            close_count = math.max(0, close_count - 1)
        end
```

**Tại sao cần điều chỉnh?**
- Trước khi render `}`, ta đã giảm indent 1 lần (ở block `if starts_with_close`)
- Hàm `calculate_indent_level()` vẫn đếm `}` đó
- Nếu không trừ đi, indent sẽ bị giảm 2 lần → Sai!

**Ví dụ:**
```
Statement: "}"
1. Trước render: current_indent = 2 - 1 = 1
2. calculate_indent_level("}"): open=0, close=1
3. Sau statement: current_indent = 1 + 0 - 1 = 0 ← SAI! (phải là 1)
4. FIX: close_count = 1 - 1 = 0
5. Sau statement: current_indent = 1 + 0 - 0 = 1 ← ĐÚNG!
```

---

#### Cập Nhật Indent
```lua
        current_indent = current_indent + open_count - close_count
        current_indent = math.max(0, current_indent)
```

**Logic:**
- `current_indent` tăng thêm `open_count` (mỗi `{` tăng 1)
- `current_indent` giảm đi `close_count` (mỗi `}` giảm 1)
- `math.max(0, ...)` đảm bảo không âm

**Ví dụ:**
```
Statement: "if (x) {"
open_count = 1, close_count = 0
current_indent = 0 + 1 - 0 = 1  (dòng tiếp theo indent 1 level)
```

---

#### Label `::continue_stmt::`
```lua
        ::continue_stmt::
    end
```
- Label để `goto continue_stmt` nhảy đến
- Kết thúc vòng lặp xử lý statements

---

#### Label `::continue::`
```lua
    ::continue::
end
```
- Label để `goto continue` nhảy đến
- Kết thúc vòng lặp xử lý dòng

---

#### Ghi Kết Quả Vào Buffer
```lua
vim.api.nvim_buf_set_lines(0, 0, -1, false, formatted)
```

**`nvim_buf_set_lines(buffer, start, end, strict_indexing, replacement)`:**
- `0` - Buffer hiện tại
- `0` - Start line (dòng 0)
- `-1` - End line (cuối file)
- `false` - Strict indexing
- `formatted` - Array chứa dòng mới

**Hiệu ứng:**
- Replace TẤT CẢ dòng trong buffer bằng `formatted`
- Đây chính là lúc code được format!

---

#### Hiển Thị Notification Hoàn Tất
```lua
vim.notify("✅ Format hoàn tất!", vim.log.levels.INFO)
```
- Thông báo cho user biết format xong

---

## 7. Keybinding và Command

### Keybinding 1: `<leader>f`
```lua
vim.keymap.set("n", "<leader>f", M.format_buffer, { desc = "Format code (tự viết)" })
```

**`vim.keymap.set(mode, lhs, rhs, opts)`:**
- `mode` - `"n"` = Normal mode (các mode khác: `"i"` Insert, `"v"` Visual)
- `lhs` - Left-hand side: Tổ hợp phím (`<leader>f`)
- `rhs` - Right-hand side: Hàm hoặc command cần thực thi (`M.format_buffer`)
- `opts` - Options: `{ desc = "..." }` hiển thị description trong help

**`<leader>`:**
- Leader key trong Vim (mặc định là `\`)
- Người dùng có thể thay đổi thành space, `,`, etc.
- Ví dụ: Nếu leader = space, `<leader>f` = `Space + f`

**Hiệu ứng:**
- Khi ở Normal mode, nhấn `<leader>f` sẽ gọi `M.format_buffer()`

---

### Keybinding 2: `<S-A-f>` (Shift+Alt+F)
```lua
vim.keymap.set("n", "<S-A-f>", M.format_buffer, { desc = "Format code (Shift+Alt+F)" })
```

**`<S-A-f>`:**
- `S` - Shift
- `A` - Alt (Meta key)
- `f` - Chữ f

**Tại sao thêm?**
- Shift+Alt+F là shortcut format phổ biến trong VS Code
- Giúp user quen thuộc với VS Code dễ chuyển sang

---

### User Command `:Format`
```lua
vim.api.nvim_create_user_command("Format", M.format_buffer, {})
```

**`nvim_create_user_command(name, command, opts)`:**
- `name` - `"Format"` - Tên command (viết hoa chữ cái đầu theo convention)
- `command` - Hàm sẽ được gọi
- `opts` - Options (rỗng trong case này)

**Hiệu ứng:**
- Tạo command `:Format` trong Neovim
- User có thể gõ `:Format` và nhấn Enter để format code

**Ví dụ sử dụng:**
```vim
:Format           " Format buffer hiện tại
:Format<CR>       " CR = Carriage Return (Enter)
```

---

## 8. Return Module
```lua
return M
```
- Return table `M` để file khác có thể require
- Ví dụ: `local formatter = require("formatter")` trong file khác

---

## Tổng Kết Flow Hoàn Chỉnh

### Khi User Nhấn `<leader>f`:

1. **Kiểm tra filetype**
   - Nếu không phải C/C++/Java → Cảnh báo và thoát

2. **Lấy tất cả dòng từ buffer**
   - `vim.api.nvim_buf_get_lines(...)` → Array `lines`

3. **Duyệt từng dòng:**
   - Trim whitespace
   - Nếu dòng trống → Giữ nguyên
   - Nếu không: Tách thành nhiều statement bằng `split_statements()`

4. **Xử lý từng statement:**
   - Kiểm tra nếu bắt đầu bằng `}` → Giảm indent
   - Tạo indent string (spaces hoặc tabs)
   - Normalize spaces quanh operators
   - Thêm vào kết quả: `indent + statement`
   - Tính indent cho dòng tiếp theo

5. **Ghi kết quả vào buffer**
   - `vim.api.nvim_buf_set_lines(...)` replace tất cả dòng

6. **Hiển thị notification**
   - "✅ Format hoàn tất!"

---

## Các Khái Niệm Lua Quan Trọng

### 1. Table (Array + Dictionary)
```lua
local arr = {1, 2, 3}           -- Array (index bắt đầu từ 1)
local dict = {name = "John"}    -- Dictionary
local mixed = {
    x = 10,                     -- Key-value
    "hello",                    -- Index 1
    "world"                     -- Index 2
}
```

### 2. String Methods
```lua
local s = "hello"
s:sub(1, 3)      -- "hel" (substring)
s:match("^he")   -- "he" (regex match)
s:gsub("l", "L") -- "heLLo" (replace)
#s               -- 5 (length)
```

### 3. Operators
```lua
..    -- Concatenate: "a" .. "b" = "ab"
~=    -- Not equal: x ~= 5
#     -- Length: #"hello" = 5
and   -- Logic AND
or    -- Logic OR
not   -- Logic NOT
```

### 4. Control Flow
```lua
if condition then
    -- code
elseif other then
    -- code
else
    -- code
end

for i = 1, 10 do          -- for (int i=1; i<=10; i++)
    -- code
end

for k, v in pairs(t) do   -- Duyệt dictionary
    -- code
end

for i, v in ipairs(arr) do -- Duyệt array
    -- code
end

while condition do
    -- code
end
```

### 5. Functions
```lua
local function foo(x, y)
    return x + y
end

local bar = function(x)  -- Anonymous function
    return x * 2
end

local a, b = foo(1, 2)   -- Multiple return values
```

### 6. Goto và Label
```lua
::mylabel::
-- code
goto mylabel  -- Jump to label
```

---

## Ví Dụ Test Case Hoàn Chỉnh

### Input (Minified Code):
```cpp
#include <iostream>
using namespace std;
int main() {for(int i = 1; i <= 10; i++) {cout << i << endl;}return 0;
}
```

### Quá Trình Xử Lý:

**Dòng 1:** `#include <iostream>`
- Không có `{`, `}`, `;`
- Indent = 0
- Output: `#include <iostream>`

**Dòng 2:** `using namespace std;`
- Có `;` ở cuối, không có code sau
- Indent = 0
- Output: `using namespace std;`

**Dòng 3:** `int main() {for(int i = 1; i <= 10; i++) {cout << i << endl;}return 0;`
- Tách thành:
  1. `int main() {`
  2. `for(int i = 1; i <= 10; i++) {`
  3. `cout << i << endl;`
  4. `}`
  5. `return 0;`

**Statement 1:** `int main() {`
- Indent = 0 (ban đầu)
- Output: `int main() {`
- Có 1 `{` → Indent tiếp theo = 1

**Statement 2:** `for(int i = 1; i <= 10; i++) {`
- Indent = 1
- Output: `    for(int i = 1; i <= 10; i++) {`
- Có 1 `{` → Indent tiếp theo = 2

**Statement 3:** `cout << i << endl;`
- Indent = 2
- Output: `        cout << i << endl;`
- Không có `{}` → Indent giữ nguyên = 2

**Statement 4:** `}`
- Bắt đầu bằng `}` → Giảm indent: 2 - 1 = 1
- Indent = 1
- Output: `    }`
- Có 1 `}` (đã trừ 1 lần) → Indent tiếp theo = 1

**Statement 5:** `return 0;`
- Indent = 1
- Output: `    return 0;`

**Dòng 4:** `}`
- Bắt đầu bằng `}` → Giảm indent: 1 - 1 = 0
- Output: `}`

### Output (Formatted):
```cpp
#include <iostream>
using namespace std;
int main() {
    for(int i = 1; i <= 10; i++) {
        cout << i << endl;
    }
    return 0;
}
```

---

## Hạn Chế Của Formatter

### 1. Không Xử Lý Escaped String
```cpp
string s = "He said \"hello\"";  // \" không được handle
```
→ Có thể nhầm `"` thứ 2 là kết thúc string

### 2. Không Xử Lý Character Literal
```cpp
char c = '{';  // Nhầm { là open brace
```

### 3. Không Hỗ Trợ Nhiều Ngôn Ngữ
- Chỉ C, C++, Java
- Không có Python, JavaScript, etc.

### 4. Không Có Formatting Style Options
- Không có Allman style, K&R style, etc.
- Indent size cố định

---

## Cách Mở Rộng Trong Tương Lai

### 1. Thêm Hỗ Trợ Ngôn Ngữ Khác
```lua
local config = {
    supported_filetypes = {"c", "cpp", "java", "javascript", "python"}
}
```

### 2. Thêm Config Cho Style
```lua
local config = {
    brace_style = "K&R",  -- or "Allman", "GNU", etc.
    indent_style = "spaces", -- or "tabs"
}
```

### 3. Tích Hợp External Formatters
```lua
if vim.fn.executable("clang-format") == 1 then
    -- Use clang-format
else
    -- Fallback to custom formatter
end
```

---

## Câu Hỏi Thường Gặp (FAQ)

### Q: Tại sao không dùng clang-format?
**A:** Formatter này là bài học để hiểu cách parsing và formatting hoạt động. Trong production, nên dùng clang-format.

### Q: Làm sao kiểm tra filetype của file?
**A:** Trong Neovim, gõ `:set filetype?` hoặc `:echo &filetype`

### Q: Có thể thay đổi leader key?
**A:** Có, thêm vào config:
```lua
vim.g.mapleader = " "  -- Đổi leader thành Space
```

### Q: Formatter có chạy được trên Visual mode (format selection)?
**A:** Hiện tại không. Cần thêm code để xử lý range.

### Q: Có thể format file khác (không phải buffer hiện tại)?
**A:** Có, cần thêm tham số:
```lua
function M.format_buffer(bufnr)
    bufnr = bufnr or 0  -- Default to current buffer
    -- ...
end
```

---

Tài liệu này bao gồm TẤT CẢ các khái niệm cần thiết để hiểu code formatter. Nếu có câu hỏi về bất kỳ phần nào, hãy hỏi thêm!
