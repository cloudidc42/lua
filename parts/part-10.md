# บทที่ 10: การจัดการไฟล์และ OS Library

## สารบัญ
1. [บทนำการจัดการไฟล์](#บทนำ)
2. [การเปิดและปิดไฟล์](#การเปิดและปิดไฟล์)
3. [การอ่านไฟล์](#การอ่านไฟล์)
4. [การเขียนไฟล์](#การเขียนไฟล์)
5. [ไฟล์ Binary](#ไฟล์-binary)
6. [Metadata ของไฟล์](#metadata-ของไฟล์)
7. [การจัดการ Directory](#การจัดการ-directory)
8. [Temporary Files](#temporary-files)
9. [Path Manipulation](#path-manipulation)
10. [Config File Reader (INI)](#config-file-reader)
11. [CSV Parser/Writer](#csv-parserwriter)
12. [Log Rotation](#log-rotation)
13. [Backup Utility](#backup-utility)
14. [JSON File Storage](#json-file-storage)
15. [Data Persistence Patterns](#data-persistence-patterns)
16. [OS Library](#os-library)
17. [Environment Variables](#environment-variables)
18. [Cross-Platform Paths](#cross-platform-paths)

---

## บทนำ

การจัดการไฟล์เป็นส่วนสำคัญของโปรแกรมในชีวิตจริง Lua มี `io` library สำหรับจัดการ input/output และ `os` library สำหรับการทำงานกับ operating system

```lua
-- ตัวอย่างที่ 1: โครงสร้างพื้นฐานของ io library
print("=== Lua File I/O Overview ===")
print("io.open()   - เปิดไฟล์")
print("io.close()  - ปิดไฟล์")
print("io.read()   - อ่านจาก stdin")
print("io.write()  - เขียนไปยัง stdout")
print("io.lines()  - อ่านทีละบรรทัด")
print("")
print("File methods:")
print("file:read()  - อ่านจากไฟล์")
print("file:write() - เขียนไปยังไฟล์")
print("file:seek()  - ย้าย position")
print("file:close() - ปิดไฟล์")
```

```lua
-- ตัวอย่างที่ 2: Mode ของการเปิดไฟล์
local modes = {
    ["r"]  = "อ่านอย่างเดียว (text mode) - ค่าเริ่มต้น",
    ["w"]  = "เขียน (สร้างใหม่หรือล้างข้อมูลเดิม)",
    ["a"]  = "เพิ่มข้อมูลต่อท้าย (append)",
    ["r+"] = "อ่านและเขียน (ต้องมีไฟล์อยู่แล้ว)",
    ["w+"] = "อ่านและเขียน (สร้างใหม่หรือล้างข้อมูลเดิม)",
    ["a+"] = "อ่านและเพิ่มข้อมูลต่อท้าย",
    ["rb"] = "อ่านอย่างเดียว (binary mode)",
    ["wb"] = "เขียน (binary mode)",
    ["ab"] = "เพิ่มข้อมูลต่อท้าย (binary mode)",
}

print("=== File Open Modes ===")
for mode, desc in pairs(modes) do
    print(string.format("  %-4s : %s", mode, desc))
end
```

---

## การเปิดและปิดไฟล์

```lua
-- ตัวอย่างที่ 3: การเปิดไฟล์แบบพื้นฐาน
local function open_file_example()
    -- เปิดไฟล์สำหรับเขียน
    local file, err = io.open("/tmp/test_lua.txt", "w")
    
    if not file then
        print("Error opening file: " .. (err or "unknown error"))
        return
    end
    
    -- เขียนข้อมูล
    file:write("Hello, Lua!\n")
    file:write("การจัดการไฟล์ใน Lua\n")
    
    -- ปิดไฟล์
    file:close()
    print("เขียนไฟล์สำเร็จ")
end

open_file_example()
```

```lua
-- ตัวอย่างที่ 4: Pattern การจัดการไฟล์แบบปลอดภัย (RAII-style)
local function with_file(filename, mode, func)
    local file, err = io.open(filename, mode)
    if not file then
        return nil, "Cannot open file '" .. filename .. "': " .. (err or "unknown")
    end
    
    local ok, result = pcall(func, file)
    file:close()
    
    if not ok then
        return nil, "Error during file operation: " .. tostring(result)
    end
    
    return result
end

-- ใช้งาน
local result, err = with_file("/tmp/test_lua.txt", "w", function(f)
    f:write("Line 1\n")
    f:write("Line 2\n")
    f:write("Line 3\n")
    return "เขียนสำเร็จ"
end)

if result then
    print(result)
else
    print("Error: " .. err)
end
```

```lua
-- ตัวอย่างที่ 5: ตรวจสอบว่าไฟล์มีอยู่หรือไม่
local function file_exists(filename)
    local file = io.open(filename, "r")
    if file then
        file:close()
        return true
    end
    return false
end

-- ทดสอบ
local files = {"/tmp/test_lua.txt", "/tmp/nonexistent.txt", "/etc/hosts"}
for _, path in ipairs(files) do
    print(string.format("%-30s : %s", path, 
          file_exists(path) and "มีอยู่" or "ไม่มี"))
end
```

```lua
-- ตัวอย่างที่ 6: Standard IO streams
print("=== Standard Streams ===")
-- io.stdin  - standard input
-- io.stdout - standard output  
-- io.stderr - standard error

-- เขียนไปยัง stdout โดยตรง
io.stdout:write("ข้อความนี้ไปยัง stdout\n")

-- เขียนไปยัง stderr
io.stderr:write("ข้อความ error นี้ไปยัง stderr\n")

-- เปลี่ยน default output
local old_output = io.output()
io.output("/tmp/redirected.txt")
io.write("ข้อความนี้ถูก redirect ไปยังไฟล์\n")
io.output(old_output)  -- คืนค่าเดิม
print("คืนค่า output เดิมแล้ว")
```

---

## การอ่านไฟล์

```lua
-- ตัวอย่างที่ 7: การอ่านไฟล์แบบต่างๆ
-- สร้างไฟล์ทดสอบก่อน
local function create_test_file(path)
    local f = io.open(path, "w")
    f:write("บรรทัดที่ 1\n")
    f:write("บรรทัดที่ 2\n")
    f:write("บรรทัดที่ 3\n")
    f:write("12345\n")
    f:write("สุดท้าย")
    f:close()
end

create_test_file("/tmp/read_test.txt")

-- อ่านทั้งไฟล์ด้วย "*all" หรือ "a"
local f = io.open("/tmp/read_test.txt", "r")
local all_content = f:read("*all")  -- หรือ f:read("a") ใน Lua 5.4
f:close()
print("=== อ่านทั้งไฟล์ ===")
print(all_content)
```

```lua
-- ตัวอย่างที่ 8: อ่านทีละบรรทัด
local f = io.open("/tmp/read_test.txt", "r")
print("=== อ่านทีละบรรทัด ===")
local line_num = 0
while true do
    local line = f:read("*line")  -- หรือ "*l" หรือ "l"
    if line == nil then break end
    line_num = line_num + 1
    print(string.format("บรรทัด %d: %s", line_num, line))
end
f:close()
```

```lua
-- ตัวอย่างที่ 9: อ่านจำนวนกำหนด (bytes)
local f = io.open("/tmp/read_test.txt", "r")
print("=== อ่าน 10 bytes แรก ===")
local chunk = f:read(10)
print("10 bytes: '" .. chunk .. "'")

-- อ่านต่อ
chunk = f:read(10)
print("10 bytes ถัดไป: '" .. (chunk or "nil") .. "'")
f:close()
```

```lua
-- ตัวอย่างที่ 10: อ่านเป็น number
local f = io.open("/tmp/read_test.txt", "r")
-- ข้ามบรรทัด 1-3 ก่อน
f:read("l"); f:read("l"); f:read("l")

-- อ่าน number
local num = f:read("*number")  -- หรือ "*n"
print("Number read: " .. tostring(num))  -- 12345
f:close()
```

```lua
-- ตัวอย่างที่ 11: io.lines() iterator
print("=== io.lines() ===")
for line in io.lines("/tmp/read_test.txt") do
    print("> " .. line)
end
```

```lua
-- ตัวอย่างที่ 12: อ่านหลาย format พร้อมกัน
local f = io.open("/tmp/read_test.txt", "r")
-- อ่านหลาย format ใน call เดียว
local line1, line2, line3 = f:read("l", "l", "l")
print("อ่าน 3 บรรทัดพร้อมกัน:")
print("1:", line1)
print("2:", line2)
print("3:", line3)
f:close()
```

```lua
-- ตัวอย่างที่ 13: อ่านไฟล์ขนาดใหญ่แบบ chunk
local function read_file_chunked(filename, chunk_size)
    chunk_size = chunk_size or 4096
    local f = io.open(filename, "rb")
    if not f then return nil, "Cannot open file" end
    
    local chunks = {}
    local total_bytes = 0
    
    while true do
        local chunk = f:read(chunk_size)
        if not chunk then break end
        table.insert(chunks, chunk)
        total_bytes = total_bytes + #chunk
    end
    
    f:close()
    return table.concat(chunks), total_bytes
end

local content, bytes = read_file_chunked("/tmp/read_test.txt", 10)
print(string.format("อ่านไฟล์สำเร็จ: %d bytes", bytes))
```

---

## การเขียนไฟล์

```lua
-- ตัวอย่างที่ 14: การเขียนไฟล์แบบต่างๆ
local f = io.open("/tmp/write_test.txt", "w")

-- เขียน string
f:write("Hello World\n")

-- เขียนหลาย argument พร้อมกัน
f:write("A", "B", "C", "\n")

-- เขียน number (จะแปลงเป็น string อัตโนมัติ)
f:write(42, "\n")
f:write(3.14, "\n")

f:close()
print("เขียนไฟล์สำเร็จ")

-- อ่านกลับมาตรวจสอบ
print(io.open("/tmp/write_test.txt", "r"):read("a"))
```

```lua
-- ตัวอย่างที่ 15: Append mode
local filename = "/tmp/append_test.txt"

-- สร้างไฟล์ใหม่
local f = io.open(filename, "w")
f:write("บรรทัดเริ่มต้น\n")
f:close()

-- เพิ่มข้อมูลต่อท้าย
for i = 1, 3 do
    local f2 = io.open(filename, "a")
    f2:write("เพิ่มบรรทัดที่ " .. i .. "\n")
    f2:close()
end

-- ตรวจสอบผล
print("=== Append Result ===")
for line in io.lines(filename) do
    print(line)
end
```

```lua
-- ตัวอย่างที่ 16: Buffered writing และ flush
local f = io.open("/tmp/buffered.txt", "w")

-- เขียนข้อมูล (อาจถูก buffer ไว้)
f:write("ข้อมูลที่อาจถูก buffer\n")

-- บังคับ flush ทันที
f:flush()
print("Flush แล้ว - ข้อมูลถูกเขียนลงดิสก์")

f:write("ข้อมูลต่อมา\n")
f:close()  -- close อัตโนมัติ flush
```

```lua
-- ตัวอย่างที่ 17: เขียน formatted text ด้วย string.format
local function write_report(filename, data)
    local f = io.open(filename, "w")
    if not f then return false end
    
    f:write(string.rep("=", 40) .. "\n")
    f:write(string.format("%-20s %10s\n", "ชื่อสินค้า", "ราคา"))
    f:write(string.rep("-", 40) .. "\n")
    
    local total = 0
    for _, item in ipairs(data) do
        f:write(string.format("%-20s %10.2f\n", item.name, item.price))
        total = total + item.price
    end
    
    f:write(string.rep("-", 40) .. "\n")
    f:write(string.format("%-20s %10.2f\n", "รวมทั้งหมด", total))
    f:write(string.rep("=", 40) .. "\n")
    f:close()
    return true
end

local products = {
    {name = "Laptop", price = 35000},
    {name = "Mouse", price = 500},
    {name = "Keyboard", price = 1200},
    {name = "Monitor", price = 8500},
}

write_report("/tmp/report.txt", products)
print(io.open("/tmp/report.txt", "r"):read("a"))
```

---

## ไฟล์ Binary

```lua
-- ตัวอย่างที่ 18: การเขียนและอ่านไฟล์ binary
local function write_binary_file(filename)
    local f = io.open(filename, "wb")
    if not f then return false end
    
    -- เขียน bytes โดยตรง
    -- string.char() แปลง number เป็น byte
    f:write(string.char(0xFF, 0xFE, 0x00, 0x01))  -- 4 bytes
    f:write(string.char(72, 101, 108, 108, 111))   -- "Hello" เป็น ASCII
    
    -- เขียน 32-bit integer แบบ little-endian
    local function write_uint32_le(file, n)
        file:write(string.char(
            n & 0xFF,
            (n >> 8) & 0xFF,
            (n >> 16) & 0xFF,
            (n >> 24) & 0xFF
        ))
    end
    
    write_uint32_le(f, 0x12345678)
    f:close()
    return true
end

local function read_binary_file(filename)
    local f = io.open(filename, "rb")
    if not f then return nil end
    
    local data = f:read("a")
    f:close()
    
    print(string.format("อ่าน %d bytes", #data))
    
    -- แสดงเป็น hex
    local hex = {}
    for i = 1, #data do
        table.insert(hex, string.format("%02X", data:byte(i)))
    end
    print("Hex: " .. table.concat(hex, " "))
    
    return data
end

write_binary_file("/tmp/binary_test.bin")
read_binary_file("/tmp/binary_test.bin")
```

```lua
-- ตัวอย่างที่ 19: Binary file - อ่าน/เขียน struct-like data
-- จำลองการอ่าน/เขียน header ของไฟล์ (เหมือน BMP, PNG header)

local function pack_header(magic, version, size)
    -- magic: 4 chars, version: 2 bytes, size: 4 bytes (little-endian)
    local header = magic  -- 4 bytes
    header = header .. string.char(version & 0xFF, (version >> 8) & 0xFF)
    header = header .. string.char(
        size & 0xFF,
        (size >> 8) & 0xFF,
        (size >> 16) & 0xFF,
        (size >> 24) & 0xFF
    )
    return header
end

local function unpack_header(data)
    local magic = data:sub(1, 4)
    local version = data:byte(5) + data:byte(6) * 256
    local size = data:byte(7) + 
                 data:byte(8) * 256 + 
                 data:byte(9) * 65536 + 
                 data:byte(10) * 16777216
    return magic, version, size
end

-- เขียน
local header = pack_header("MYFT", 100, 2048)
local f = io.open("/tmp/header_test.bin", "wb")
f:write(header)
f:write("ข้อมูลหลัง header\n")
f:close()

-- อ่าน
local f2 = io.open("/tmp/header_test.bin", "rb")
local raw_header = f2:read(10)
local magic, version, size = unpack_header(raw_header)
local rest = f2:read("a")
f2:close()

print(string.format("Magic: %s, Version: %d, Size: %d", magic, version, size))
print("ข้อมูลส่วนที่เหลือ: " .. rest)
```

```lua
-- ตัวอย่างที่ 20: Seek operations
local f = io.open("/tmp/seek_test.txt", "w")
f:write("0123456789ABCDEF")  -- 16 bytes
f:close()

local f = io.open("/tmp/seek_test.txt", "r+")

-- f:seek(whence, offset)
-- whence: "set" (ตั้งแต่ต้น), "cur" (จากปัจจุบัน), "end" (จากท้าย)

-- อ่านจากตำแหน่งเริ่มต้น
print("Position เริ่มต้น:", f:seek())  -- 0

-- ไปที่ byte ที่ 5
f:seek("set", 5)
print("หลัง seek(set,5):", f:read(3))  -- "567"

-- เลื่อนไปข้างหน้า 2 bytes จากปัจจุบัน
f:seek("cur", 2)
print("หลัง seek(cur,2):", f:read(3))  -- "ABC"

-- ไปที่ท้ายไฟล์
f:seek("end", 0)
print("Size ของไฟล์:", f:seek())  -- 16

-- ไปที่ 3 bytes ก่อนท้าย
f:seek("end", -3)
print("3 bytes ก่อนท้าย:", f:read(3))  -- "DEF"

f:close()
```

---

## Metadata ของไฟล์

```lua
-- ตัวอย่างที่ 21: การดู file metadata ผ่าน os.execute และ io.popen
local function get_file_info(filename)
    -- ใช้ io.popen เพื่อรัน shell command และอ่าน output
    local cmd
    if package.config:sub(1,1) == '\\' then
        -- Windows
        cmd = 'dir /b /s "' .. filename .. '" 2>NUL'
    else
        -- Unix/Linux/Mac
        cmd = 'stat "' .. filename .. '" 2>/dev/null'
    end
    
    local pipe = io.popen(cmd)
    local result = pipe:read("a")
    pipe:close()
    return result
end

-- ทดสอบ
local info = get_file_info("/tmp/report.txt")
if info ~= "" then
    print("File info:")
    print(info:sub(1, 200))  -- แสดงแค่ 200 chars แรก
else
    print("ไม่สามารถอ่าน file info ได้")
end
```

```lua
-- ตัวอย่างที่ 22: ขนาดไฟล์
local function get_file_size(filename)
    local f = io.open(filename, "rb")
    if not f then return nil end
    local size = f:seek("end")
    f:close()
    return size
end

local function format_size(bytes)
    if bytes < 1024 then
        return string.format("%d B", bytes)
    elseif bytes < 1024 * 1024 then
        return string.format("%.1f KB", bytes / 1024)
    elseif bytes < 1024 * 1024 * 1024 then
        return string.format("%.1f MB", bytes / (1024 * 1024))
    else
        return string.format("%.2f GB", bytes / (1024 * 1024 * 1024))
    end
end

-- ทดสอบกับหลายไฟล์
local test_files = {"/tmp/report.txt", "/tmp/binary_test.bin", "/tmp/read_test.txt"}
print("=== File Sizes ===")
for _, path in ipairs(test_files) do
    local size = get_file_size(path)
    if size then
        print(string.format("%-30s : %s", path, format_size(size)))
    end
end
```

---

## การจัดการ Directory

```lua
-- ตัวอย่างที่ 23: os.rename - เปลี่ยนชื่อ/ย้ายไฟล์
local function rename_file(old_name, new_name)
    local ok, err = os.rename(old_name, new_name)
    if ok then
        print(string.format("เปลี่ยนชื่อ '%s' เป็น '%s' สำเร็จ", old_name, new_name))
        return true
    else
        print(string.format("Error: %s", err or "unknown"))
        return false
    end
end

-- สร้างไฟล์ทดสอบ
io.open("/tmp/old_name.txt", "w"):write("test"):close()

rename_file("/tmp/old_name.txt", "/tmp/new_name.txt")
-- ตรวจสอบ
print("ไฟล์เดิมมีอยู่:", io.open("/tmp/old_name.txt", "r") ~= nil)
print("ไฟล์ใหม่มีอยู่:", io.open("/tmp/new_name.txt", "r") ~= nil)
```

```lua
-- ตัวอย่างที่ 24: os.remove - ลบไฟล์
local function safe_remove(filename)
    -- ตรวจสอบก่อนลบ
    local f = io.open(filename, "r")
    if not f then
        print("ไฟล์ไม่มีอยู่: " .. filename)
        return false
    end
    f:close()
    
    local ok, err = os.remove(filename)
    if ok then
        print("ลบไฟล์สำเร็จ: " .. filename)
        return true
    else
        print("ลบไฟล์ไม่ได้: " .. (err or "unknown"))
        return false
    end
end

safe_remove("/tmp/new_name.txt")
safe_remove("/tmp/nonexistent_file.txt")
```

```lua
-- ตัวอย่างที่ 25: สร้าง Directory และ list ไฟล์
local function make_dir(path)
    local cmd
    if package.config:sub(1,1) == '\\' then
        cmd = 'mkdir "' .. path .. '" 2>NUL'
    else
        cmd = 'mkdir -p "' .. path .. '"'
    end
    return os.execute(cmd)
end

local function list_dir(path)
    local cmd
    if package.config:sub(1,1) == '\\' then
        cmd = 'dir /b "' .. path .. '"'
    else
        cmd = 'ls -la "' .. path .. '"'
    end
    
    local pipe = io.popen(cmd)
    local result = {}
    for line in pipe:lines() do
        table.insert(result, line)
    end
    pipe:close()
    return result
end

-- สร้าง directory
make_dir("/tmp/lua_test_dir")
make_dir("/tmp/lua_test_dir/subdir")

-- สร้างไฟล์ทดสอบ
io.open("/tmp/lua_test_dir/file1.txt", "w"):write("a"):close()
io.open("/tmp/lua_test_dir/file2.lua", "w"):write("b"):close()

-- List ไฟล์
print("=== Files in /tmp/lua_test_dir ===")
local files = list_dir("/tmp/lua_test_dir")
for _, f in ipairs(files) do
    print("  " .. f)
end
```

```lua
-- ตัวอย่างที่ 26: ค้นหาไฟล์ด้วย pattern
local function find_files(dir, pattern)
    local cmd
    if package.config:sub(1,1) == '\\' then
        cmd = string.format('dir /b /s "%s\\%s"', dir, pattern)
    else
        cmd = string.format('find "%s" -name "%s" -type f', dir, pattern)
    end
    
    local pipe = io.popen(cmd)
    local files = {}
    for line in pipe:lines() do
        table.insert(files, line)
    end
    pipe:close()
    return files
end

print("=== ค้นหาไฟล์ .txt ใน /tmp ===")
local txt_files = find_files("/tmp", "*.txt")
for i, f in ipairs(txt_files) do
    if i <= 5 then  -- แสดงแค่ 5 ไฟล์แรก
        print("  " .. f)
    end
end
print(string.format("พบทั้งหมด %d ไฟล์", #txt_files))
```

---

## Temporary Files

```lua
-- ตัวอย่างที่ 27: สร้าง Temporary File
local function create_temp_file(prefix, suffix)
    prefix = prefix or "lua_tmp_"
    suffix = suffix or ".tmp"
    
    -- สร้างชื่อไฟล์ที่ unique
    local timestamp = tostring(os.time())
    local random = tostring(math.random(100000, 999999))
    local name = os.tmpname()  -- Lua built-in temp name
    
    -- หรือสร้างเอง
    local custom_name = "/tmp/" .. prefix .. timestamp .. "_" .. random .. suffix
    
    return name, custom_name
end

math.randomseed(os.time())
local tmp1, tmp2 = create_temp_file("myapp_", ".dat")
print("Temp file 1 (os.tmpname):", tmp1)
print("Temp file 2 (custom):", tmp2)

-- สร้างและใช้งาน
local f = io.open(tmp2, "w")
f:write("temporary data\n")
f:close()

-- ทำงานกับ temp file
local content = io.open(tmp2, "r"):read("a")
print("Content:", content)

-- ลบหลังใช้
os.remove(tmp2)
print("ลบ temp file แล้ว")
```

```lua
-- ตัวอย่างที่ 28: Temp file manager
local TempFileManager = {}
TempFileManager.__index = TempFileManager

function TempFileManager.new()
    return setmetatable({files = {}}, TempFileManager)
end

function TempFileManager:create(suffix)
    suffix = suffix or ".tmp"
    local name = "/tmp/luatmp_" .. os.time() .. "_" .. 
                 tostring(math.random(10000, 99999)) .. suffix
    table.insert(self.files, name)
    return name
end

function TempFileManager:cleanup()
    local removed = 0
    for _, filename in ipairs(self.files) do
        if os.remove(filename) then
            removed = removed + 1
        end
    end
    self.files = {}
    return removed
end

-- ใช้งาน
local manager = TempFileManager.new()

local t1 = manager:create(".txt")
local t2 = manager:create(".csv")
local t3 = manager:create(".json")

io.open(t1, "w"):write("test1"):close()
io.open(t2, "w"):write("test2"):close()
io.open(t3, "w"):write("test3"):close()

print("สร้าง temp files:", #manager.files)
local removed = manager:cleanup()
print("ลบไปแล้ว:", removed, "ไฟล์")
```

---

## Path Manipulation

```lua
-- ตัวอย่างที่ 29: Path utilities
local path = {}

-- ตัวคั่น path ตาม OS
path.sep = package.config:sub(1, 1)  -- '\' บน Windows, '/' บน Unix

function path.join(...)
    local parts = {...}
    return table.concat(parts, path.sep)
end

function path.dirname(p)
    return p:match("^(.*)" .. path.sep .. "[^" .. path.sep .. "]*$") or "."
end

function path.basename(p)
    return p:match("[^" .. path.sep .. "]+$") or p
end

function path.extension(p)
    local base = path.basename(p)
    return base:match("^.+(%.[^.]+)$") or ""
end

function path.stem(p)
    local base = path.basename(p)
    return base:match("^(.+)%.[^.]+$") or base
end

function path.is_absolute(p)
    if path.sep == '\\' then
        return p:match("^[A-Za-z]:") ~= nil or p:sub(1,2) == "\\\\"
    else
        return p:sub(1,1) == "/"
    end
end

-- ทดสอบ
local test_paths = {
    "/home/user/documents/report.txt",
    "/tmp/test.lua",
    "relative/path/file.dat",
    "noextension",
    "/root/.config",
}

print("=== Path Manipulation ===")
for _, p in ipairs(test_paths) do
    print(string.format("\nPath: %s", p))
    print(string.format("  dirname  : %s", path.dirname(p)))
    print(string.format("  basename : %s", path.basename(p)))
    print(string.format("  extension: %s", path.extension(p)))
    print(string.format("  stem     : %s", path.stem(p)))
    print(string.format("  absolute : %s", tostring(path.is_absolute(p))))
end
```

```lua
-- ตัวอย่างที่ 30: Normalize path
function path.normalize(p)
    local parts = {}
    local sep = path.sep
    
    -- แยก path ออกเป็น parts
    for part in (p .. sep):gmatch("([^" .. sep .. "]*)" .. sep) do
        if part == ".." then
            if #parts > 0 and parts[#parts] ~= ".." then
                table.remove(parts)
            else
                table.insert(parts, "..")
            end
        elseif part ~= "." and part ~= "" then
            table.insert(parts, part)
        end
    end
    
    local result = table.concat(parts, sep)
    if p:sub(1,1) == sep then
        result = sep .. result
    end
    return result ~= "" and result or "."
end

print("=== Normalize Paths ===")
local paths_to_normalize = {
    "/home/user/../user/./documents/../docs/file.txt",
    "./relative/./path/../to/file",
    "/absolute/path/./here",
}

for _, p in ipairs(paths_to_normalize) do
    print(string.format("Original : %s", p))
    print(string.format("Normalized: %s", path.normalize(p)))
    print()
end
```

---

## Config File Reader

```lua
-- ตัวอย่างที่ 31: INI File Parser
local ini_parser = {}

function ini_parser.load(filename)
    local f = io.open(filename, "r")
    if not f then return nil, "Cannot open: " .. filename end
    
    local config = {}
    local current_section = "default"
    config[current_section] = {}
    
    for line in f:lines() do
        -- ตัด whitespace
        line = line:match("^%s*(.-)%s*$")
        
        -- ข้ามบรรทัดว่างและ comment
        if line == "" or line:sub(1,1) == ";" or line:sub(1,1) == "#" then
            -- skip
        -- Section header [section_name]
        elseif line:match("^%[(.+)%]$") then
            current_section = line:match("^%[(.+)%]$")
            if not config[current_section] then
                config[current_section] = {}
            end
        -- Key=Value
        elseif line:match("^([^=]+)=(.*)$") then
            local key, value = line:match("^([^=]+)=(.*)$")
            key = key:match("^%s*(.-)%s*$")
            value = value:match("^%s*(.-)%s*$")
            
            -- แปลงค่า boolean
            if value:lower() == "true" then
                value = true
            elseif value:lower() == "false" then
                value = false
            -- แปลงค่า number
            elseif tonumber(value) then
                value = tonumber(value)
            end
            
            config[current_section][key] = value
        end
    end
    
    f:close()
    return config
end

function ini_parser.save(filename, config)
    local f = io.open(filename, "w")
    if not f then return false, "Cannot open: " .. filename end
    
    f:write("; Generated by Lua INI Parser\n")
    f:write("; " .. os.date("%Y-%m-%d %H:%M:%S") .. "\n\n")
    
    for section, data in pairs(config) do
        f:write("[" .. section .. "]\n")
        for key, value in pairs(data) do
            if type(value) == "boolean" then
                f:write(key .. " = " .. (value and "true" or "false") .. "\n")
            else
                f:write(key .. " = " .. tostring(value) .. "\n")
            end
        end
        f:write("\n")
    end
    
    f:close()
    return true
end

-- สร้างไฟล์ INI ทดสอบ
local ini_content = [[
; Application Configuration
[database]
host = localhost
port = 5432
name = mydb
ssl = true

[server]
host = 0.0.0.0
port = 8080
debug = false
max_connections = 100

[logging]
level = info
file = /var/log/app.log
rotate = true
]]

local ini_file = "/tmp/test_config.ini"
io.open(ini_file, "w"):write(ini_content):close()

-- อ่าน config
local config, err = ini_parser.load(ini_file)
if config then
    print("=== INI Config ===")
    for section, data in pairs(config) do
        print(string.format("[%s]", section))
        for k, v in pairs(data) do
            print(string.format("  %s = %s (%s)", k, tostring(v), type(v)))
        end
    end
    
    -- เข้าถึงค่าเฉพาะ
    print("\nDatabase host:", config.database.host)
    print("Server port:", config.server.port)
    print("Debug mode:", config.server.debug)
end
```

```lua
-- ตัวอย่างที่ 32: Config ที่รองรับ interpolation
function ini_parser.get(config, section, key, default)
    if config[section] and config[section][key] ~= nil then
        return config[section][key]
    end
    return default
end

function ini_parser.set(config, section, key, value)
    if not config[section] then
        config[section] = {}
    end
    config[section][key] = value
end

-- ทดสอบ
local cfg = ini_parser.load(ini_file)
ini_parser.set(cfg, "server", "timeout", 30)
ini_parser.set(cfg, "cache", "enabled", true)
ini_parser.set(cfg, "cache", "ttl", 3600)

-- บันทึก
ini_parser.save("/tmp/updated_config.ini", cfg)
print("บันทึก config ใหม่แล้ว")
print("Cache enabled:", ini_parser.get(cfg, "cache", "enabled", false))
print("Cache TTL:", ini_parser.get(cfg, "cache", "ttl", 300))
```

---

## CSV Parser/Writer

```lua
-- ตัวอย่างที่ 33: CSV Parser ที่รองรับ quoting
local csv = {}

function csv.parse_line(line, delimiter)
    delimiter = delimiter or ","
    local fields = {}
    local field = ""
    local in_quotes = false
    local i = 1
    
    while i <= #line do
        local c = line:sub(i, i)
        
        if c == '"' then
            if in_quotes and line:sub(i+1, i+1) == '"' then
                -- escaped quote ""
                field = field .. '"'
                i = i + 2
            else
                in_quotes = not in_quotes
                i = i + 1
            end
        elseif c == delimiter and not in_quotes then
            table.insert(fields, field)
            field = ""
            i = i + 1
        else
            field = field .. c
            i = i + 1
        end
    end
    
    table.insert(fields, field)
    return fields
end

function csv.parse(content, delimiter)
    local rows = {}
    for line in (content .. "\n"):gmatch("([^\n]*)\n") do
        if line ~= "" then
            table.insert(rows, csv.parse_line(line, delimiter))
        end
    end
    return rows
end

function csv.escape_field(value, delimiter)
    delimiter = delimiter or ","
    value = tostring(value)
    -- ถ้ามี comma, newline, หรือ quote ต้องใส่ quotes
    if value:find('[' .. delimiter .. '"\n\r]') then
        value = '"' .. value:gsub('"', '""') .. '"'
    end
    return value
end

function csv.format_line(fields, delimiter)
    delimiter = delimiter or ","
    local escaped = {}
    for _, v in ipairs(fields) do
        table.insert(escaped, csv.escape_field(v, delimiter))
    end
    return table.concat(escaped, delimiter)
end

function csv.write(filename, data, headers, delimiter)
    local f = io.open(filename, "w")
    if not f then return false end
    
    if headers then
        f:write(csv.format_line(headers, delimiter) .. "\n")
    end
    
    for _, row in ipairs(data) do
        f:write(csv.format_line(row, delimiter) .. "\n")
    end
    
    f:close()
    return true
end

-- ทดสอบ
local csv_content = [[Name,Age,City,Notes
"Smith, John",30,Bangkok,"Has comma, in name"
Alice,25,"New York","Normal ""quoted"" text"
Bob,35,Tokyo,Simple
]]

local rows = csv.parse(csv_content)
print("=== CSV Parse ===")
print(string.format("%-15s %-5s %-10s %s", "Name", "Age", "City", "Notes"))
print(string.rep("-", 50))
for i, row in ipairs(rows) do
    if i == 1 then
        -- skip header
    else
        print(string.format("%-15s %-5s %-10s %s", 
              row[1] or "", row[2] or "", row[3] or "", row[4] or ""))
    end
end

-- เขียน CSV
local data = {
    {"Product A", 1500, "In Stock", "Good quality"},
    {'Product B, "Special"', 2500, "Out of Stock", "Limited edition"},
    {"Product C", 500, "In Stock", "Budget option"},
}
csv.write("/tmp/products.csv", data, {"Product", "Price", "Status", "Notes"})
print("\nบันทึก CSV สำเร็จ")
```

---

## Log Rotation

```lua
-- ตัวอย่างที่ 34: Log Rotation System
local LogRotator = {}
LogRotator.__index = LogRotator

function LogRotator.new(options)
    options = options or {}
    return setmetatable({
        filename = options.filename or "/tmp/app.log",
        max_size = options.max_size or 1024 * 1024,  -- 1 MB
        max_files = options.max_files or 5,
        current_file = nil,
        current_size = 0,
    }, LogRotator)
end

function LogRotator:_get_file_size(filename)
    local f = io.open(filename, "rb")
    if not f then return 0 end
    local size = f:seek("end")
    f:close()
    return size
end

function LogRotator:_rotate()
    if self.current_file then
        self.current_file:close()
        self.current_file = nil
    end
    
    -- เลื่อน log files
    -- app.log.5 -> ลบ
    -- app.log.4 -> app.log.5
    -- ...
    -- app.log.1 -> app.log.2
    -- app.log   -> app.log.1
    
    local old_max = self.filename .. "." .. self.max_files
    os.remove(old_max)
    
    for i = self.max_files - 1, 1, -1 do
        local old = self.filename .. "." .. i
        local new_name = self.filename .. "." .. (i + 1)
        os.rename(old, new_name)
    end
    
    os.rename(self.filename, self.filename .. ".1")
    self.current_size = 0
    
    print(string.format("[LogRotator] Rotated logs (max=%d files)", self.max_files))
end

function LogRotator:_open()
    if not self.current_file then
        self.current_file = io.open(self.filename, "a")
        self.current_size = self:_get_file_size(self.filename)
    end
end

function LogRotator:write(message)
    self:_open()
    
    local timestamp = os.date("%Y-%m-%d %H:%M:%S")
    local log_line = string.format("[%s] %s\n", timestamp, message)
    
    self.current_file:write(log_line)
    self.current_file:flush()
    self.current_size = self.current_size + #log_line
    
    -- ตรวจสอบขนาด
    if self.current_size >= self.max_size then
        self:_rotate()
    end
end

function LogRotator:close()
    if self.current_file then
        self.current_file:close()
        self.current_file = nil
    end
end

-- ทดสอบ
local logger = LogRotator.new({
    filename = "/tmp/rotating.log",
    max_size = 500,  -- 500 bytes สำหรับทดสอบ
    max_files = 3,
})

for i = 1, 20 do
    logger:write(string.format("Log message #%d - some application event occurred", i))
end

logger:close()
print("Log files:")
os.execute("ls -la /tmp/rotating.log* 2>/dev/null")
```

---

## Backup Utility

```lua
-- ตัวอย่างที่ 35: Backup Utility
local Backup = {}

function Backup.create_backup(source, dest_dir)
    -- สร้างชื่อ backup ด้วย timestamp
    local timestamp = os.date("%Y%m%d_%H%M%S")
    
    -- หาชื่อไฟล์
    local basename = source:match("[^/\\]+$") or source
    local backup_name = basename .. "." .. timestamp .. ".bak"
    local backup_path = dest_dir .. "/" .. backup_name
    
    -- สร้าง dest_dir ถ้าไม่มี
    os.execute('mkdir -p "' .. dest_dir .. '"')
    
    -- คัดลอกไฟล์
    local src = io.open(source, "rb")
    if not src then
        return nil, "Cannot open source: " .. source
    end
    
    local dst = io.open(backup_path, "wb")
    if not dst then
        src:close()
        return nil, "Cannot create backup: " .. backup_path
    end
    
    -- คัดลอกทีละ chunk
    local chunk_size = 8192
    while true do
        local chunk = src:read(chunk_size)
        if not chunk then break end
        dst:write(chunk)
    end
    
    src:close()
    dst:close()
    
    return backup_path
end

function Backup.list_backups(dest_dir, original_name)
    local cmd = string.format('ls -t "%s"/%s.*.bak 2>/dev/null', dest_dir, original_name)
    local pipe = io.popen(cmd)
    local backups = {}
    for line in pipe:lines() do
        table.insert(backups, line)
    end
    pipe:close()
    return backups
end

function Backup.cleanup_old_backups(dest_dir, original_name, keep)
    keep = keep or 5
    local backups = Backup.list_backups(dest_dir, original_name)
    
    local removed = 0
    for i = keep + 1, #backups do
        os.remove(backups[i])
        removed = removed + 1
    end
    return removed
end

-- ทดสอบ
local source_file = "/tmp/important_data.txt"
io.open(source_file, "w"):write("Important data v1\nLine 2\nLine 3\n"):close()

local backup_dir = "/tmp/backups"
print("=== Backup Utility ===")

-- สร้าง backup หลายครั้ง
for i = 1, 3 do
    io.open(source_file, "a"):write("Update #" .. i .. "\n"):close()
    local backup, err = Backup.create_backup(source_file, backup_dir)
    if backup then
        print("Backup created: " .. backup)
    else
        print("Backup failed: " .. (err or "unknown"))
    end
    os.execute("sleep 1 2>/dev/null || ping -n 2 127.0.0.1 >NUL 2>&1")
end

-- List backups
local backups = Backup.list_backups(backup_dir, "important_data.txt")
print("\nAvailable backups: " .. #backups)

-- Cleanup
local removed = Backup.cleanup_old_backups(backup_dir, "important_data.txt", 2)
print("Removed old backups: " .. removed)
```

---

## JSON File Storage

```lua
-- ตัวอย่างที่ 36: Simple JSON Encoder/Decoder
local json = {}

function json.encode(value, indent, current_indent)
    indent = indent or ""
    current_indent = current_indent or ""
    local next_indent = current_indent .. indent
    
    local t = type(value)
    
    if t == "nil" then
        return "null"
    elseif t == "boolean" then
        return value and "true" or "false"
    elseif t == "number" then
        if value ~= value then return "null" end  -- NaN
        if value == math.huge or value == -math.huge then return "null" end
        return tostring(value)
    elseif t == "string" then
        -- Escape special characters
        value = value:gsub('\\', '\\\\')
        value = value:gsub('"', '\\"')
        value = value:gsub('\n', '\\n')
        value = value:gsub('\r', '\\r')
        value = value:gsub('\t', '\\t')
        return '"' .. value .. '"'
    elseif t == "table" then
        -- ตรวจสอบว่าเป็น array หรือ object
        local is_array = true
        local max_i = 0
        for k, _ in pairs(value) do
            if type(k) ~= "number" or k ~= math.floor(k) or k < 1 then
                is_array = false
                break
            end
            if k > max_i then max_i = k end
        end
        is_array = is_array and max_i == #value
        
        local parts = {}
        if is_array then
            for _, v in ipairs(value) do
                if indent ~= "" then
                    table.insert(parts, next_indent .. json.encode(v, indent, next_indent))
                else
                    table.insert(parts, json.encode(v, indent, next_indent))
                end
            end
            if indent ~= "" then
                return "[\n" .. table.concat(parts, ",\n") .. "\n" .. current_indent .. "]"
            else
                return "[" .. table.concat(parts, ",") .. "]"
            end
        else
            for k, v in pairs(value) do
                local key = json.encode(tostring(k))
                if indent ~= "" then
                    table.insert(parts, next_indent .. key .. ": " .. json.encode(v, indent, next_indent))
                else
                    table.insert(parts, key .. ":" .. json.encode(v, indent, next_indent))
                end
            end
            if indent ~= "" then
                return "{\n" .. table.concat(parts, ",\n") .. "\n" .. current_indent .. "}"
            else
                return "{" .. table.concat(parts, ",") .. "}"
            end
        end
    end
    
    return '"[' .. t .. ']"'
end

function json.decode(str)
    local pos = 1
    
    local function skip_whitespace()
        while pos <= #str and str:sub(pos,pos):match("%s") do
            pos = pos + 1
        end
    end
    
    local function parse_value()
        skip_whitespace()
        local c = str:sub(pos, pos)
        
        if c == '"' then
            -- String
            pos = pos + 1
            local result = ""
            while pos <= #str do
                local ch = str:sub(pos, pos)
                if ch == '"' then
                    pos = pos + 1
                    return result
                elseif ch == '\\' then
                    pos = pos + 1
                    local esc = str:sub(pos, pos)
                    if esc == '"' then result = result .. '"'
                    elseif esc == '\\' then result = result .. '\\'
                    elseif esc == 'n' then result = result .. '\n'
                    elseif esc == 'r' then result = result .. '\r'
                    elseif esc == 't' then result = result .. '\t'
                    end
                    pos = pos + 1
                else
                    result = result .. ch
                    pos = pos + 1
                end
            end
        elseif c == '{' then
            -- Object
            pos = pos + 1
            local obj = {}
            skip_whitespace()
            if str:sub(pos,pos) == '}' then
                pos = pos + 1
                return obj
            end
            while true do
                skip_whitespace()
                local key = parse_value()
                skip_whitespace()
                if str:sub(pos,pos) ~= ':' then break end
                pos = pos + 1
                local val = parse_value()
                obj[key] = val
                skip_whitespace()
                if str:sub(pos,pos) == ',' then
                    pos = pos + 1
                elseif str:sub(pos,pos) == '}' then
                    pos = pos + 1
                    break
                end
            end
            return obj
        elseif c == '[' then
            -- Array
            pos = pos + 1
            local arr = {}
            skip_whitespace()
            if str:sub(pos,pos) == ']' then
                pos = pos + 1
                return arr
            end
            while true do
                local val = parse_value()
                table.insert(arr, val)
                skip_whitespace()
                if str:sub(pos,pos) == ',' then
                    pos = pos + 1
                elseif str:sub(pos,pos) == ']' then
                    pos = pos + 1
                    break
                end
            end
            return arr
        elseif str:sub(pos, pos+3) == "true" then
            pos = pos + 4
            return true
        elseif str:sub(pos, pos+4) == "false" then
            pos = pos + 5
            return false
        elseif str:sub(pos, pos+3) == "null" then
            pos = pos + 4
            return nil
        else
            -- Number
            local num_str = str:match("^-?%d+%.?%d*[eE]?[+-]?%d*", pos)
            if num_str then
                pos = pos + #num_str
                return tonumber(num_str)
            end
        end
    end
    
    return parse_value()
end

-- ทดสอบ JSON
local data = {
    name = "Lua Tutorial",
    version = 5.4,
    features = {"fast", "embeddable", "lightweight"},
    author = {
        name = "PUC-Rio",
        country = "Brazil"
    },
    active = true,
}

local json_str = json.encode(data, "  ")
print("=== JSON Encode ===")
print(json_str)

-- บันทึกลงไฟล์
local jf = io.open("/tmp/data.json", "w")
jf:write(json_str)
jf:close()

-- อ่านกลับ
local raw = io.open("/tmp/data.json", "r"):read("a")
local decoded = json.decode(raw)

print("\n=== JSON Decode ===")
print("Name:", decoded.name)
print("Version:", decoded.version)
print("Features:", table.concat(decoded.features, ", "))
print("Author:", decoded.author.name, "from", decoded.author.country)
```

---

## Data Persistence Patterns

```lua
-- ตัวอย่างที่ 37: Simple Database ด้วยไฟล์
local SimpleDB = {}
SimpleDB.__index = SimpleDB

function SimpleDB.new(filename)
    local db = setmetatable({
        filename = filename,
        data = {},
    }, SimpleDB)
    db:load()
    return db
end

function SimpleDB:load()
    local f = io.open(self.filename, "r")
    if not f then return end
    
    local content = f:read("a")
    f:close()
    
    -- Simple serialization format: key=value pairs
    for line in content:gmatch("[^\n]+") do
        local key, value = line:match("^([^=]+)=(.*)$")
        if key then
            -- Try to parse as number, boolean, or string
            if value == "true" then value = true
            elseif value == "false" then value = false
            elseif tonumber(value) then value = tonumber(value)
            else
                -- Unescape newlines
                value = value:gsub("\\n", "\n")
            end
            self.data[key] = value
        end
    end
end

function SimpleDB:save()
    local f = io.open(self.filename, "w")
    if not f then return false end
    
    for key, value in pairs(self.data) do
        if type(value) == "string" then
            value = value:gsub("\n", "\\n")
        end
        f:write(string.format("%s=%s\n", key, tostring(value)))
    end
    
    f:close()
    return true
end

function SimpleDB:set(key, value)
    self.data[key] = value
    return self:save()
end

function SimpleDB:get(key, default)
    return self.data[key] ~= nil and self.data[key] or default
end

function SimpleDB:delete(key)
    self.data[key] = nil
    return self:save()
end

function SimpleDB:all()
    return self.data
end

-- ทดสอบ
local db = SimpleDB.new("/tmp/simple.db")

db:set("username", "john_doe")
db:set("score", 1500)
db:set("premium", true)
db:set("last_login", os.date("%Y-%m-%d"))

print("=== SimpleDB ===")
print("username:", db:get("username"))
print("score:", db:get("score"))
print("premium:", db:get("premium"))
print("last_login:", db:get("last_login"))
print("nonexistent:", db:get("nonexistent", "default_value"))

-- อ่านกลับใหม่
local db2 = SimpleDB.new("/tmp/simple.db")
print("\nหลังโหลดใหม่:")
print("username:", db2:get("username"))
print("score:", db2:get("score"))
```

```lua
-- ตัวอย่างที่ 38: Atomic File Write (ป้องกันการเสียหายบางส่วน)
local function atomic_write(filename, content)
    -- เขียนไปยัง temp file ก่อน
    local tmp_file = filename .. ".tmp"
    
    local f = io.open(tmp_file, "w")
    if not f then
        return false, "Cannot create temp file"
    end
    
    local ok, err = pcall(function()
        f:write(content)
        f:flush()
        -- sync ลงดิสก์ (ต้องการ os call)
    end)
    
    f:close()
    
    if not ok then
        os.remove(tmp_file)
        return false, err
    end
    
    -- Atomic rename (บน Unix)
    local rename_ok, rename_err = os.rename(tmp_file, filename)
    if not rename_ok then
        os.remove(tmp_file)
        return false, rename_err
    end
    
    return true
end

-- ทดสอบ
local ok, err = atomic_write("/tmp/atomic_test.txt", 
    "Important data that must not be corrupted\n" ..
    "Written atomically at " .. os.date("%Y-%m-%d %H:%M:%S") .. "\n")

if ok then
    print("Atomic write สำเร็จ")
    print(io.open("/tmp/atomic_test.txt", "r"):read("a"))
else
    print("Error:", err)
end
```

---

## OS Library

```lua
-- ตัวอย่างที่ 39: os.time() และ os.date()
print("=== os.time() ===")

-- Unix timestamp (วินาทีนับจาก 1970-01-01 00:00:00 UTC)
local now = os.time()
print("Current timestamp:", now)

-- แปลง timestamp เป็นตาราง
local t = os.date("*t", now)
print("Year:", t.year)
print("Month:", t.month)
print("Day:", t.day)
print("Hour:", t.hour)
print("Min:", t.min)
print("Sec:", t.sec)
print("Weekday:", t.wday)  -- 1=Sunday
print("Yearday:", t.yday)  -- วันที่กี่ของปี
print("DST:", t.isdst)     -- Daylight Saving Time

-- แปลงตารางเป็น timestamp
local specific_time = os.time({
    year = 2024,
    month = 1,
    day = 1,
    hour = 0,
    min = 0,
    sec = 0,
})
print("\n2024-01-01 00:00:00 timestamp:", specific_time)
```

```lua
-- ตัวอย่างที่ 40: os.date() format strings
print("=== os.date() Formats ===")

local formats = {
    {"%Y-%m-%d",              "ISO Date"},
    {"%d/%m/%Y",              "TH Date format"},
    {"%H:%M:%S",              "Time"},
    {"%Y-%m-%d %H:%M:%S",    "DateTime"},
    {"%A, %B %d, %Y",        "Full Date"},
    {"%a %b %d",              "Short Date"},
    {"%I:%M %p",              "12-hour"},
    {"%j",                    "Day of year"},
    {"%W",                    "Week number"},
    {"%Z",                    "Timezone"},
    {"%c",                    "Locale datetime"},
    {"%x",                    "Locale date"},
    {"%X",                    "Locale time"},
}

for _, fmt in ipairs(formats) do
    local result = os.date(fmt[1])
    print(string.format("  %-25s -> %s  (%s)", fmt[1], result, fmt[2]))
end
```

```lua
-- ตัวอย่างที่ 41: คำนวณเวลา
local function time_diff(t1, t2)
    local diff = math.abs(os.difftime(t2, t1))
    
    local seconds = diff % 60
    local minutes = math.floor(diff / 60) % 60
    local hours = math.floor(diff / 3600) % 24
    local days = math.floor(diff / 86400)
    
    local parts = {}
    if days > 0 then table.insert(parts, days .. " วัน") end
    if hours > 0 then table.insert(parts, hours .. " ชั่วโมง") end
    if minutes > 0 then table.insert(parts, minutes .. " นาที") end
    if seconds > 0 then table.insert(parts, seconds .. " วินาที") end
    
    return table.concat(parts, " ") 
end

local birthday = os.time({year=1990, month=6, day=15, hour=0, min=0, sec=0})
local now = os.time()
print("อายุ:", time_diff(birthday, now))

local new_year = os.time({year=os.date("*t").year + 1, month=1, day=1, hour=0, min=0, sec=0})
print("ถึงปีใหม่อีก:", time_diff(now, new_year))
```

```lua
-- ตัวอย่างที่ 42: os.clock() สำหรับ profiling
print("=== os.clock() Profiling ===")

local function measure_time(func, ...)
    local start = os.clock()
    local results = {func(...)}
    local elapsed = os.clock() - start
    return elapsed, table.unpack(results)
end

-- ทดสอบ string concatenation vs table.concat
local function string_concat(n)
    local s = ""
    for i = 1, n do
        s = s .. tostring(i) .. ","
    end
    return s
end

local function table_concat_method(n)
    local t = {}
    for i = 1, n do
        t[i] = tostring(i)
    end
    return table.concat(t, ",")
end

local N = 10000
local t1, _ = measure_time(string_concat, N)
local t2, _ = measure_time(table_concat_method, N)

print(string.format("String concat (%d iterations): %.4f seconds", N, t1))
print(string.format("Table concat  (%d iterations): %.4f seconds", N, t2))
print(string.format("Table concat เร็วกว่า: %.1fx", t1/t2))
```

```lua
-- ตัวอย่างที่ 43: os.execute()
print("=== os.execute() ===")

-- รัน shell command
local function run_command(cmd)
    local status = os.execute(cmd)
    return status == true or status == 0
end

-- ตรวจสอบว่า command ทำงานสำเร็จหรือไม่
if run_command("mkdir -p /tmp/lua_test 2>/dev/null") then
    print("สร้าง directory สำเร็จ")
end

if run_command("touch /tmp/lua_test/test.txt 2>/dev/null") then
    print("สร้างไฟล์สำเร็จ")
end

-- รัน command แบบ background (Unix)
-- os.execute("some_command &")  -- รัน background

-- io.popen - รัน command และอ่าน output
local function get_command_output(cmd)
    local pipe = io.popen(cmd .. " 2>&1")
    local output = pipe:read("a")
    local ok, status, code = pipe:close()
    return output, (status == "exit" and code == 0)
end

local output, success = get_command_output("echo 'Hello from shell'")
print("Output:", output:match("^(.-)%s*$"))  -- trim whitespace
print("Success:", success)
```

```lua
-- ตัวอย่างที่ 44: os.exit()
local function safe_exit(code, message)
    if message then
        if code == 0 then
            io.stdout:write(message .. "\n")
        else
            io.stderr:write("Error: " .. message .. "\n")
        end
    end
    
    -- ทำ cleanup ก่อน exit
    -- io.flush ทุก open file handles
    io.stdout:flush()
    io.stderr:flush()
    
    os.exit(code, true)  -- true = ปิด Lua state อย่างสมบูรณ์
end

-- ตัวอย่างการใช้ (comment ออกเพื่อไม่ให้ exit จริง)
-- safe_exit(0, "Program completed successfully")
-- safe_exit(1, "Failed to connect to database")

print("os.exit() จะออกจากโปรแกรมทันที")
print("ใช้ os.exit(0) สำหรับ success, os.exit(1) สำหรับ error")
```

---

## Environment Variables

```lua
-- ตัวอย่างที่ 45: อ่าน Environment Variables
print("=== Environment Variables ===")

-- ใช้ os.getenv()
local function get_env(name, default)
    local value = os.getenv(name)
    return value ~= nil and value or default
end

-- Environment variables ที่ใช้บ่อย
local env_vars = {"HOME", "PATH", "USER", "SHELL", "LANG", "TEMP", "TMP"}

for _, var in ipairs(env_vars) do
    local value = os.getenv(var)
    if value then
        -- Truncate long values
        local display = #value > 50 and value:sub(1, 47) .. "..." or value
        print(string.format("  %-10s = %s", var, display))
    end
end

-- ค่า default สำหรับ config
local config = {
    db_host    = get_env("DB_HOST", "localhost"),
    db_port    = tonumber(get_env("DB_PORT", "5432")),
    log_level  = get_env("LOG_LEVEL", "info"),
    debug      = get_env("DEBUG", "false") == "true",
    home_dir   = get_env("HOME", "/tmp"),
}

print("\n=== App Config from Env ===")
for k, v in pairs(config) do
    print(string.format("  %-15s = %s", k, tostring(v)))
end
```

```lua
-- ตัวอย่างที่ 46: .env file parser
local function load_dotenv(filename)
    filename = filename or ".env"
    local f = io.open(filename, "r")
    if not f then return {} end
    
    local env = {}
    for line in f:lines() do
        -- ตัด whitespace
        line = line:match("^%s*(.-)%s*$")
        
        -- ข้าม comment และบรรทัดว่าง
        if line ~= "" and line:sub(1,1) ~= "#" then
            local key, value = line:match("^([^=]+)=(.*)$")
            if key then
                key = key:match("^%s*(.-)%s*$")
                value = value:match("^%s*(.-)%s*$")
                
                -- ลบ quotes รอบ value
                if (value:sub(1,1) == '"' and value:sub(-1) == '"') or
                   (value:sub(1,1) == "'" and value:sub(-1) == "'") then
                    value = value:sub(2, -2)
                end
                
                env[key] = value
            end
        end
    end
    
    f:close()
    return env
end

-- สร้างไฟล์ .env ทดสอบ
local dotenv_content = [[
# Application settings
APP_NAME=MyLuaApp
APP_VERSION=1.0.0
DEBUG=false

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME="my_database"
DB_PASSWORD='secret123'

# API Keys
API_KEY=abc123def456
]]

io.open("/tmp/test.env", "w"):write(dotenv_content):close()

local env_vars = load_dotenv("/tmp/test.env")
print("=== .env file ===")
for k, v in pairs(env_vars) do
    print(string.format("  %s=%s", k, v))
end
```

---

## Cross-Platform File Paths

```lua
-- ตัวอย่างที่ 47: Cross-platform path handling
local Platform = {}

function Platform.detect()
    local sep = package.config:sub(1,1)
    if sep == '\\' then
        return "windows"
    else
        return "unix"
    end
end

function Platform.temp_dir()
    local os_name = Platform.detect()
    if os_name == "windows" then
        return os.getenv("TEMP") or os.getenv("TMP") or "C:\\Temp"
    else
        return os.getenv("TMPDIR") or "/tmp"
    end
end

function Platform.home_dir()
    local os_name = Platform.detect()
    if os_name == "windows" then
        return os.getenv("USERPROFILE") or os.getenv("HOMEDRIVE") .. os.getenv("HOMEPATH")
    else
        return os.getenv("HOME") or "~"
    end
end

function Platform.config_dir(app_name)
    local os_name = Platform.detect()
    if os_name == "windows" then
        local appdata = os.getenv("APPDATA") or Platform.home_dir()
        return appdata .. "\\" .. app_name
    else
        local home = Platform.home_dir()
        return home .. "/.config/" .. app_name
    end
end

function Platform.path_join(...)
    local sep = package.config:sub(1,1)
    local parts = {...}
    -- Normalize separators
    local normalized = {}
    for _, p in ipairs(parts) do
        p = p:gsub("[/\\]", sep)
        -- ตัด trailing separator
        p = p:match("^(.-)[/\\]?$") or p
        if p ~= "" then
            table.insert(normalized, p)
        end
    end
    return table.concat(normalized, sep)
end

print("=== Platform Info ===")
print("OS:", Platform.detect())
print("Temp dir:", Platform.temp_dir())
print("Home dir:", Platform.home_dir())
print("Config dir:", Platform.config_dir("MyApp"))

print("\n=== Path Join Examples ===")
print(Platform.path_join("/home", "user", "documents", "file.txt"))
print(Platform.path_join(Platform.home_dir(), ".config", "app", "settings.ini"))
```

```lua
-- ตัวอย่างที่ 48: ตรวจสอบ path ข้าม platforms
local function normalize_path(p)
    -- แปลง backslash เป็น forward slash (ทำงานได้บน Windows ด้วย)
    p = p:gsub("\\", "/")
    -- ลบ double slashes
    p = p:gsub("//+", "/")
    -- ลบ trailing slash (ยกเว้น root)
    if #p > 1 then
        p = p:match("^(.-)/+$") or p
    end
    return p
end

local paths = {
    "C:\\Users\\John\\Documents",
    "/home/user/documents/",
    "relative\\path\\file.txt",
    "/var//log///app.log",
}

print("=== Normalize Paths ===")
for _, p in ipairs(paths) do
    print(string.format("  %-35s -> %s", p, normalize_path(p)))
end
```

```lua
-- ตัวอย่างที่ 49: สรุปฟังก์ชัน file utility library
local FileUtils = {}

-- อ่านทั้งไฟล์
function FileUtils.read(filename)
    local f = io.open(filename, "r")
    if not f then return nil end
    local content = f:read("a")
    f:close()
    return content
end

-- เขียนทั้งไฟล์
function FileUtils.write(filename, content)
    local f = io.open(filename, "w")
    if not f then return false end
    f:write(content)
    f:close()
    return true
end

-- เพิ่มต่อท้าย
function FileUtils.append(filename, content)
    local f = io.open(filename, "a")
    if not f then return false end
    f:write(content)
    f:close()
    return true
end

-- คัดลอกไฟล์
function FileUtils.copy(src, dst)
    local s = io.open(src, "rb")
    if not s then return false end
    local d = io.open(dst, "wb")
    if not d then s:close(); return false end
    
    while true do
        local chunk = s:read(8192)
        if not chunk then break end
        d:write(chunk)
    end
    s:close()
    d:close()
    return true
end

-- ขนาดไฟล์
function FileUtils.size(filename)
    local f = io.open(filename, "rb")
    if not f then return nil end
    local size = f:seek("end")
    f:close()
    return size
end

-- ตรวจสอบว่าไฟล์มีอยู่
function FileUtils.exists(filename)
    local f = io.open(filename, "r")
    if f then f:close(); return true end
    return false
end

-- นับบรรทัด
function FileUtils.count_lines(filename)
    local count = 0
    for _ in io.lines(filename) do
        count = count + 1
    end
    return count
end

-- ทดสอบ
local test_file = "/tmp/fileutils_test.txt"
FileUtils.write(test_file, "Line 1\nLine 2\nLine 3\n")
FileUtils.append(test_file, "Line 4\nLine 5\n")
FileUtils.copy(test_file, test_file .. ".bak")

print("=== FileUtils ===")
print("Size:", FileUtils.size(test_file), "bytes")
print("Lines:", FileUtils.count_lines(test_file))
print("Exists:", FileUtils.exists(test_file))
print("Content:\n" .. FileUtils.read(test_file))
```

```lua
-- ตัวอย่างที่ 50: สรุป - โปรแกรมจัดการไฟล์ Config แบบ Complete
local ConfigManager = {}
ConfigManager.__index = ConfigManager

function ConfigManager.new(config_file)
    local self = setmetatable({
        config_file = config_file,
        backup_dir = config_file .. "_backups",
        data = {},
    }, ConfigManager)
    self:load()
    return self
end

function ConfigManager:load()
    local f = io.open(self.config_file, "r")
    if not f then
        self.data = {_meta = {version = 1, created = os.date("%Y-%m-%d")}}
        return false
    end
    
    local content = f:read("a")
    f:close()
    
    -- Parse simple key=value
    self.data = {}
    for line in content:gmatch("[^\n]+") do
        line = line:match("^%s*(.-)%s*$")
        if line ~= "" and line:sub(1,1) ~= "#" then
            local key, value = line:match("^([^=]+)=(.*)$")
            if key then
                self.data[key:match("^%s*(.-)%s*$")] = value:match("^%s*(.-)%s*$")
            end
        end
    end
    return true
end

function ConfigManager:save()
    -- สร้าง backup ก่อนบันทึก
    if io.open(self.config_file, "r") then
        os.execute('mkdir -p "' .. self.backup_dir .. '"')
        local backup = self.backup_dir .. "/config." .. os.date("%Y%m%d_%H%M%S") .. ".bak"
        
        local src = io.open(self.config_file, "r")
        if src then
            local dst = io.open(backup, "w")
            if dst then
                dst:write(src:read("a"))
                dst:close()
            end
            src:close()
        end
    end
    
    local f = io.open(self.config_file, "w")
    if not f then return false end
    
    f:write("# Config file\n")
    f:write("# Last updated: " .. os.date("%Y-%m-%d %H:%M:%S") .. "\n\n")
    
    for k, v in pairs(self.data) do
        f:write(k .. "=" .. tostring(v) .. "\n")
    end
    
    f:close()
    return true
end

function ConfigManager:get(key, default)
    return self.data[key] ~= nil and self.data[key] or default
end

function ConfigManager:set(key, value)
    self.data[key] = tostring(value)
    return self:save()
end

-- ทดสอบ
local cm = ConfigManager.new("/tmp/app_config.txt")
cm:set("app_name", "MyLuaApp")
cm:set("version", "1.2.3")
cm:set("debug", "false")
cm:set("last_run", os.date("%Y-%m-%d"))

print("=== ConfigManager ===")
print("app_name:", cm:get("app_name"))
print("version:", cm:get("version"))
print("missing_key:", cm:get("missing_key", "default_value"))
print("\nไฟล์ config:")
print(io.open("/tmp/app_config.txt", "r"):read("a"))
```

---

## สรุปบทที่ 10

| หัวข้อ | ฟังก์ชัน/Method |
|--------|----------------|
| เปิดไฟล์ | `io.open(file, mode)` |
| อ่านไฟล์ | `file:read("a", "l", "n", N)` |
| เขียนไฟล์ | `file:write(...)` |
| ย้าย position | `file:seek("set/cur/end", offset)` |
| ปิดไฟล์ | `file:close()` |
| ลบไฟล์ | `os.remove(file)` |
| เปลี่ยนชื่อ | `os.rename(old, new)` |
| เวลาปัจจุบัน | `os.time()`, `os.date()` |
| CPU time | `os.clock()` |
| รัน command | `os.execute(cmd)` |
| อ่าน output | `io.popen(cmd)` |
| Environment | `os.getenv(name)` |
| ออกโปรแกรม | `os.exit(code)` |

> **Tip:** เสมอใช้ `pcall` เมื่อทำงานกับไฟล์เพื่อจัดการ error อย่างปลอดภัย และปิดไฟล์ทุกครั้งหลังใช้งาน
