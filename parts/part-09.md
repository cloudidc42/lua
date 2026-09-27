# บทที่ 9: I/O และการรับ-ส่งข้อมูล

การรับและส่งข้อมูล (Input/Output) เป็นหัวใจสำคัญของโปรแกรมทุกตัว Lua มี `io` library และ `os` library ที่ครอบคลุมการทำงานกับไฟล์ terminal และระบบปฏิบัติการ บทนี้จะสำรวจทุกด้านอย่างละเอียด

---

## 9.1 print() vs io.write()

```lua
-- print(): พิมพ์และขึ้นบรรทัดใหม่อัตโนมัติ
print("สวัสดี Lua!")        -- สวัสดี Lua!\n
print(1, 2, 3)              -- 1\t2\t3\n (คั่นด้วย tab)
print("a", "b", "c")       -- a\tb\tc\n
print()                     -- บรรทัดว่าง

-- io.write(): พิมพ์โดยไม่ขึ้นบรรทัดใหม่
io.write("Hello")
io.write(", ")
io.write("World")
io.write("\n")              -- ต้องใส่ \n เอง
-- Output: Hello, World
```

```lua
-- ความแตกต่างหลัก
-- print() แปลง argument เป็น string ด้วย tostring()
print(true)     -- true
print(nil)      -- nil
print({})       -- table: 0x...

-- io.write() ต้องการ string หรือ number เท่านั้น
io.write(42)                -- 42
io.write(3.14)              -- 3.14
-- io.write(true)           -- ERROR!
-- io.write(nil)            -- ERROR!
io.write(tostring(true))    -- true (ต้องแปลงก่อน)
```

```lua
-- print() ใช้ tostring() ซึ่งเรียก __tostring metamethod
-- io.write() ไม่เรียก metamethod
local t = setmetatable({}, {
    __tostring = function() return "Custom String" end
})

print(t)           -- Custom String (เรียก __tostring)
-- io.write(t)     -- ERROR! io.write ไม่เรียก __tostring
io.write(tostring(t) .. "\n")  -- Custom String
```

```lua
-- ตัวอย่าง: progress bar
local function progress_bar(current, total, width)
    width = width or 40
    local percent = current / total
    local filled = math.floor(percent * width)
    local bar = string.rep("█", filled) .. string.rep("░", width - filled)
    io.write(string.format("\r[%s] %3d%%", bar, math.floor(percent * 100)))
    io.flush()  -- บังคับส่ง output ทันที
end

-- จำลอง progress
for i = 0, 100, 5 do
    progress_bar(i, 100)
    -- ในโปรแกรมจริงจะมี os.execute("sleep 0.1") หรือ delay อื่น
end
print()  -- ขึ้นบรรทัดใหม่หลังเสร็จ
```

---

## 9.2 io.read() - รับข้อมูลจาก stdin

```lua
-- รูปแบบการอ่าน:
-- io.read("l")  หรือ io.read("*l") - อ่านบรรทัด (ไม่รวม \n) (default)
-- io.read("L")  หรือ io.read("*L") - อ่านบรรทัด (รวม \n)
-- io.read("n")  หรือ io.read("*n") - อ่านตัวเลข
-- io.read("a")  หรือ io.read("*a") - อ่านทั้งหมดถึง EOF
-- io.read(n)                       - อ่าน n bytes

-- อ่านบรรทัด (สำหรับ interactive input)
-- io.write("กรุณาใส่ชื่อ: ")
-- local name = io.read("l")   -- หรือ io.read()
-- print("สวัสดี", name)
```

```lua
-- ตัวอย่าง: โปรแกรมคิดเลข interactive
-- (comment ออกเพราะต้องการ input จาก user)
--[[
local function calculator()
    while true do
        io.write("ใส่ expression (เช่น 1+2) หรือ 'quit' เพื่อออก: ")
        local input = io.read("l")
        
        if input == "quit" or input == nil then
            print("ลาก่อน!")
            break
        end
        
        -- ประเมิน expression (ใช้ loadstring/load)
        local fn, err = load("return " .. input)
        if fn then
            local ok, result = pcall(fn)
            if ok then
                print("= " .. tostring(result))
            else
                print("Error:", result)
            end
        else
            print("Syntax error:", err)
        end
    end
end

calculator()
]]
```

```lua
-- อ่านหลายค่าพร้อมกัน
-- local a, b = io.read("n", "n")  -- อ่านตัวเลข 2 ตัว
-- print(a + b)

-- อ่านทั้งไฟล์/stdin
-- local content = io.read("a")
-- print("อ่านได้:", #content, "bytes")
```

```lua
-- ตัวอย่าง: อ่าน CSV จาก stdin
--[[
local function read_csv_stdin()
    local rows = {}
    for line in io.lines() do  -- io.lines() อ่านจาก stdin
        local row = {}
        for field in (line .. ","):gmatch("([^,]*),") do
            row[#row + 1] = field
        end
        rows[#rows + 1] = row
    end
    return rows
end

local data = read_csv_stdin()
for i, row in ipairs(data) do
    print("Row " .. i .. ":", table.concat(row, " | "))
end
]]
```

---

## 9.3 Standard I/O Files

```lua
-- io.stdin, io.stdout, io.stderr

-- เขียนไปยัง stdout
io.stdout:write("ข้อความปกติ\n")

-- เขียนไปยัง stderr (error messages)
io.stderr:write("ข้อความ error\n")

-- อ่านจาก stdin
-- local line = io.stdin:read("l")
```

```lua
-- เปลี่ยน default input/output
-- io.input(file_or_filename) -- เปลี่ยน default input
-- io.output(file_or_filename) -- เปลี่ยน default output

-- ตัวอย่าง: redirect stdout ไปยังไฟล์
-- io.output("output.txt")
-- print("ข้อความนี้จะไปที่ output.txt")
-- io.output(io.stdout)  -- reset กลับ

-- ตรวจสอบ current default files
-- local current_input = io.input()
-- local current_output = io.output()
```

---

## 9.4 io.open() - เปิดไฟล์

```lua
-- io.open(filename, mode)
-- Modes:
-- "r"  - read (default)
-- "w"  - write (เขียนทับ)
-- "a"  - append (ต่อท้าย)
-- "r+" - read + write (ต้องมีไฟล์อยู่แล้ว)
-- "w+" - read + write (เขียนทับ/สร้างใหม่)
-- "a+" - read + append
-- "rb", "wb", "ab" - binary modes
-- "rb+", "wb+", "ab+" - binary read+write

-- เปิดไฟล์สำหรับอ่าน
local file, err = io.open("/tmp/test.txt", "r")
if not file then
    -- ไม่มีไฟล์ - สร้างก่อน
    local f = io.open("/tmp/test.txt", "w")
    if f then
        f:write("บรรทัดที่ 1\n")
        f:write("บรรทัดที่ 2\n")
        f:write("บรรทัดที่ 3\n")
        f:close()
        print("สร้างไฟล์แล้ว")
    end
else
    print("เปิดไฟล์สำเร็จ")
    file:close()
end
```

```lua
-- pattern: เปิดไฟล์อย่างปลอดภัย
local function safe_open(path, mode)
    local file, err = io.open(path, mode or "r")
    if not file then
        return nil, string.format("ไม่สามารถเปิดไฟล์ '%s': %s", path, err)
    end
    return file, nil
end

local f, err = safe_open("/tmp/test.txt", "r")
if err then
    print("Error:", err)
else
    print("เปิดได้!")
    f:close()
end
```

---

## 9.5 file:read(), file:write(), file:close()

```lua
-- สร้างไฟล์ทดสอบก่อน
local function create_test_file(path)
    local f = io.open(path, "w")
    if not f then return false end
    f:write("# ไฟล์ทดสอบ\n")
    f:write("บรรทัดที่ 1: สวัสดี\n")
    f:write("บรรทัดที่ 2: โลก\n")
    f:write("บรรทัดที่ 3: Lua\n")
    f:write("123\n")
    f:write("456.78\n")
    f:close()
    return true
end

local test_file = "/tmp/lua_test.txt"
create_test_file(test_file)
```

```lua
-- อ่านแบบต่างๆ
local f = io.open(test_file, "r")

-- อ่านทีละบรรทัด
local line1 = f:read("l")      -- อ่านบรรทัดแรก (ไม่รวม \n)
local line2 = f:read("L")      -- อ่านบรรทัดสอง (รวม \n)
print("บรรทัด 1:", line1)
print("บรรทัด 2:", line2)

-- อ่านต่อเนื่อง
local line3 = f:read()          -- default เป็น "l"
print("บรรทัด 3:", line3)

-- อ่านบรรทัด 4
local line4 = f:read()
print("บรรทัด 4:", line4)

-- อ่านตัวเลข
local num1 = f:read("n")        -- อ่านเป็น number
local num2 = f:read("n")
print("ตัวเลข:", num1, num2)    -- 123  456.78

f:close()
```

```lua
-- อ่านทั้งไฟล์ในครั้งเดียว
local f = io.open(test_file, "r")
local content = f:read("a")     -- อ่านทั้งหมด
f:close()

print("เนื้อหาทั้งหมด:")
print(content)
print("ขนาด:", #content, "bytes")
```

```lua
-- อ่านทีละบรรทัด (loop)
local f = io.open(test_file, "r")
print("อ่านทีละบรรทัด:")
local line_num = 0
for line in f:lines() do    -- f:lines() iterator
    line_num = line_num + 1
    print(string.format("  %d: %s", line_num, line))
end
f:close()
```

```lua
-- file:write() - เขียนไฟล์
local output_file = "/tmp/output.txt"
local f = io.open(output_file, "w")

-- เขียนหลายรูปแบบ
f:write("สวัสดี Lua!\n")
f:write("จำนวน: ", 42, "\n")             -- หลาย arguments
f:write(string.format("Pi = %.5f\n", math.pi))

-- method chaining
f:write("a"):write("b"):write("c\n")    -- abc

f:close()

-- ตรวจสอบผล
local check = io.open(output_file, "r")
print(check:read("a"))
check:close()
```

---

## 9.6 file:seek()

```lua
-- file:seek(whence, offset)
-- whence: "set" (จากต้น), "cur" (จากที่อยู่), "end" (จากท้าย)
-- คืนค่า: ตำแหน่งปัจจุบัน (bytes จากต้น)

local f = io.open(test_file, "r")

-- ตำแหน่งปัจจุบัน
print("ตำแหน่งเริ่ม:", f:seek())           -- 0

-- อ่าน 5 bytes
local chunk = f:read(5)
print("อ่าน 5 bytes:", chunk)
print("ตำแหน่งหลังอ่าน:", f:seek())       -- 5

-- ไปที่ต้นไฟล์
f:seek("set", 0)
print("หลัง seek ต้น:", f:seek())          -- 0

-- ไปที่ท้ายไฟล์
local size = f:seek("end")
print("ขนาดไฟล์:", size, "bytes")

-- ย้อนกลับ 10 bytes จากปัจจุบัน
f:seek("cur", -10)
local tail = f:read("a")
print("10 bytes สุดท้าย:", tail)

f:close()
```

```lua
-- ตัวอย่าง: อ่านไฟล์แบบ random access
local function read_at(filepath, offset, length)
    local f = io.open(filepath, "rb")
    if not f then return nil end
    f:seek("set", offset)
    local data = f:read(length)
    f:close()
    return data
end

-- อ่าน 10 bytes ที่ตำแหน่ง 5
local data = read_at(test_file, 5, 10)
print("อ่านที่ offset 5:", data)
```

---

## 9.7 io.lines() - อ่านไฟล์แบบ Iterator

```lua
-- io.lines(filename) - เปิดและอ่านทีละบรรทัด (ปิดอัตโนมัติ)
print("อ่านด้วย io.lines:")
local count = 0
for line in io.lines(test_file) do
    count = count + 1
    print(string.format("  %2d: %s", count, line))
end
print("รวม", count, "บรรทัด")
```

```lua
-- ตัวอย่าง: นับคำในไฟล์
local function count_words(filepath)
    local word_count = 0
    local line_count = 0
    
    for line in io.lines(filepath) do
        line_count = line_count + 1
        for word in line:gmatch("%S+") do
            word_count = word_count + 1
        end
    end
    
    return word_count, line_count
end

local words, lines = count_words(test_file)
print(string.format("คำ: %d, บรรทัด: %d", words, lines))
```

```lua
-- ตัวอย่าง: grep ง่ายๆ
local function grep(filepath, pattern)
    local results = {}
    local line_num = 0
    
    for line in io.lines(filepath) do
        line_num = line_num + 1
        if line:match(pattern) then
            results[#results + 1] = {
                line = line_num,
                text = line
            }
        end
    end
    
    return results
end

local matches = grep(test_file, "บรรทัด")
print("ผลการค้นหา 'บรรทัด':")
for _, m in ipairs(matches) do
    print(string.format("  บรรทัด %d: %s", m.line, m.text))
end
```

---

## 9.8 การจัดการ Error ใน I/O

```lua
-- io.open คืน nil, error message เมื่อล้มเหลว
local f, err = io.open("/path/that/does/not/exist/file.txt", "r")
if f then
    f:close()
else
    print("Error:", err)
    -- Error: /path/.../file.txt: No such file or directory
end
```

```lua
-- ใช้ assert เพื่อ raise error
local function open_or_die(path, mode)
    local f, err = io.open(path, mode)
    return assert(f, err)   -- ถ้า f เป็น nil จะ error พร้อม err
end

local ok, result = pcall(open_or_die, "/nonexistent.txt", "r")
if not ok then
    print("Caught error:", result)
end
```

```lua
-- Pattern: RAII-style file handling
local function with_file(path, mode, fn)
    local f, err = io.open(path, mode)
    if not f then
        return nil, err
    end
    
    local ok, result = pcall(fn, f)
    f:close()  -- ปิดไฟล์เสมอแม้จะเกิด error
    
    if not ok then
        return nil, result
    end
    return result
end

-- ใช้งาน
local content, err = with_file(test_file, "r", function(f)
    return f:read("a")
end)

if content then
    print("อ่านได้:", #content, "bytes")
else
    print("Error:", err)
end
```

```lua
-- ตรวจสอบ file:write() errors
local function safe_write(filepath, content)
    local f, err = io.open(filepath, "w")
    if not f then
        return false, err
    end
    
    local ok, werr = f:write(content)
    f:close()
    
    if not ok then
        return false, werr
    end
    return true
end

local ok, err = safe_write("/tmp/safe_test.txt", "Hello, World!\n")
print(ok and "เขียนสำเร็จ" or "Error: " .. (err or "unknown"))
```

---

## 9.9 อ่านและเขียนไฟล์ทั้งหมด

```lua
-- ฟังก์ชัน utility สำหรับ read/write ทั้งไฟล์
local function read_file(path)
    local f, err = io.open(path, "r")
    if not f then return nil, err end
    local content = f:read("a")
    f:close()
    return content
end

local function write_file(path, content)
    local f, err = io.open(path, "w")
    if not f then return false, err end
    local ok, werr = f:write(content)
    f:close()
    if not ok then return false, werr end
    return true
end

local function append_file(path, content)
    local f, err = io.open(path, "a")
    if not f then return false, err end
    local ok, werr = f:write(content)
    f:close()
    if not ok then return false, werr end
    return true
end

-- ทดสอบ
write_file("/tmp/demo.txt", "บรรทัดแรก\n")
append_file("/tmp/demo.txt", "บรรทัดสอง\n")
append_file("/tmp/demo.txt", "บรรทัดสาม\n")

local content = read_file("/tmp/demo.txt")
print("เนื้อหา:")
print(content)
```

---

## 9.10 การอ่านทีละบรรทัด

```lua
-- อ่านทีละบรรทัดและประมวลผล
local function process_lines(filepath, processor)
    local results = {}
    local line_num = 0
    
    for line in io.lines(filepath) do
        line_num = line_num + 1
        local processed = processor(line, line_num)
        if processed ~= nil then
            results[#results + 1] = processed
        end
    end
    
    return results, line_num
end

-- ตัวอย่าง: แปลงเป็น uppercase และกรองบรรทัดว่าง
local results, total = process_lines(test_file, function(line, num)
    if line:match("^%s*$") then return nil end  -- ข้ามบรรทัดว่าง
    return num .. ": " .. line:upper()
end)

print("ผลลัพธ์:")
for _, r in ipairs(results) do
    print(" ", r)
end
print("รวม", total, "บรรทัด")
```

```lua
-- อ่านไฟล์ config ง่ายๆ (key=value format)
local function read_config(filepath)
    local config = {}
    
    for line in io.lines(filepath) do
        -- ข้าม comments และบรรทัดว่าง
        line = line:match("^%s*(.-)%s*$")  -- trim whitespace
        if line ~= "" and not line:match("^#") then
            local key, value = line:match("^([%w_]+)%s*=%s*(.+)$")
            if key and value then
                -- แปลง type อัตโนมัติ
                if value == "true" then value = true
                elseif value == "false" then value = false
                elseif tonumber(value) then value = tonumber(value)
                end
                config[key] = value
            end
        end
    end
    
    return config
end

-- สร้างไฟล์ config ทดสอบ
local config_file = "/tmp/app.conf"
local f = io.open(config_file, "w")
f:write("# App Configuration\n")
f:write("host = localhost\n")
f:write("port = 8080\n")
f:write("debug = true\n")
f:write("max_connections = 100\n")
f:write("app_name = MyApp\n")
f:close()

local config = read_config(config_file)
print("Config:")
for k, v in pairs(config) do
    print(string.format("  %-20s = %s (%s)", k, tostring(v), type(v)))
end
```

---

## 9.11 CSV Reading / Writing

```lua
-- CSV Writer
local function csv_write(filepath, headers, rows)
    local f, err = io.open(filepath, "w")
    if not f then return false, err end
    
    -- เขียน header
    f:write(table.concat(headers, ",") .. "\n")
    
    -- เขียน data
    for _, row in ipairs(rows) do
        local fields = {}
        for _, field in ipairs(row) do
            -- Escape fields ที่มี comma หรือ quote
            local s = tostring(field)
            if s:find('[,"\n]') then
                s = '"' .. s:gsub('"', '""') .. '"'
            end
            fields[#fields + 1] = s
        end
        f:write(table.concat(fields, ",") .. "\n")
    end
    
    f:close()
    return true
end

-- CSV Reader
local function csv_parse_line(line)
    local fields = {}
    local i = 1
    
    while i <= #line do
        if line:sub(i, i) == '"' then
            -- Quoted field
            local field = ""
            i = i + 1
            while i <= #line do
                if line:sub(i, i) == '"' then
                    if line:sub(i+1, i+1) == '"' then
                        field = field .. '"'
                        i = i + 2
                    else
                        i = i + 1
                        break
                    end
                else
                    field = field .. line:sub(i, i)
                    i = i + 1
                end
            end
            fields[#fields + 1] = field
            if line:sub(i, i) == "," then i = i + 1 end
        else
            -- Unquoted field
            local field = line:match("^([^,]*)", i)
            fields[#fields + 1] = field
            i = i + #field + 1
        end
    end
    
    return fields
end

local function csv_read(filepath)
    local rows = {}
    local headers = nil
    
    for line in io.lines(filepath) do
        local row = csv_parse_line(line)
        if not headers then
            headers = row
        else
            local record = {}
            for i, header in ipairs(headers) do
                record[header] = row[i]
            end
            rows[#rows + 1] = record
        end
    end
    
    return rows, headers
end

-- ทดสอบ
local csv_file = "/tmp/students.csv"
local headers = {"ชื่อ", "อายุ", "คะแนน", "เมือง"}
local data = {
    {"สมชาย", 25, 85, "กรุงเทพ"},
    {"นิดา", 22, 92, "เชียงใหม่"},
    {"อรุณ", 28, 78, "ขอนแก่น"},
    {"กมล, Jr.", 24, 95, "กรุงเทพ"},  -- มี comma ในชื่อ
}

csv_write(csv_file, headers, data)
print("เขียน CSV แล้ว")

local records, hdrs = csv_read(csv_file)
print("\nอ่าน CSV:")
print("Headers:", table.concat(hdrs, ", "))
for i, rec in ipairs(records) do
    print(string.format("  %d: %-15s อายุ %s คะแนน %s",
        i, rec["ชื่อ"], rec["อายุ"], rec["คะแนน"]))
end
```

---

## 9.12 Log File Implementation

```lua
-- Logger ที่มีความสามารถเต็มรูปแบบ
local Logger = {}
Logger.__index = Logger

Logger.LEVEL = {
    DEBUG = 1,
    INFO = 2,
    WARN = 3,
    ERROR = 4,
    FATAL = 5
}

Logger.LEVEL_NAMES = {
    [1] = "DEBUG",
    [2] = "INFO",
    [3] = "WARN",
    [4] = "ERROR",
    [5] = "FATAL"
}

function Logger.new(filepath, min_level, max_size)
    local self = setmetatable({}, Logger)
    self.filepath = filepath or "/tmp/app.log"
    self.min_level = min_level or Logger.LEVEL.DEBUG
    self.max_size = max_size or 1024 * 1024  -- 1MB
    self.file = nil
    self:_open()
    return self
end

function Logger:_open()
    self.file = io.open(self.filepath, "a")
end

function Logger:_get_timestamp()
    return os.date("%Y-%m-%d %H:%M:%S")
end

function Logger:_log(level, message, ...)
    if level < self.min_level then return end
    if not self.file then return end
    
    if ... then
        message = string.format(message, ...)
    end
    
    local entry = string.format("[%s] [%-5s] %s\n",
        self:_get_timestamp(),
        Logger.LEVEL_NAMES[level],
        message)
    
    self.file:write(entry)
    self.file:flush()
    
    -- แสดงที่ stderr ด้วยสำหรับ level >= WARN
    if level >= Logger.LEVEL.WARN then
        io.stderr:write(entry)
    end
end

function Logger:debug(msg, ...) self:_log(Logger.LEVEL.DEBUG, msg, ...) end
function Logger:info(msg, ...) self:_log(Logger.LEVEL.INFO, msg, ...) end
function Logger:warn(msg, ...) self:_log(Logger.LEVEL.WARN, msg, ...) end
function Logger:error(msg, ...) self:_log(Logger.LEVEL.ERROR, msg, ...) end
function Logger:fatal(msg, ...) self:_log(Logger.LEVEL.FATAL, msg, ...) end

function Logger:close()
    if self.file then
        self.file:close()
        self.file = nil
    end
end

-- ทดสอบ Logger
local log = Logger.new("/tmp/myapp.log", Logger.LEVEL.DEBUG)

log:debug("โปรแกรมเริ่มทำงาน")
log:info("เชื่อมต่อฐานข้อมูล: %s:%d", "localhost", 5432)
log:warn("Memory usage สูง: %d%%", 85)
log:error("ไม่พบไฟล์: %s", "/path/to/file.txt")

log:close()

-- แสดงเนื้อหา log
print("\nLog file:")
for line in io.lines("/tmp/myapp.log") do
    print(" ", line)
end
```

---

## 9.13 Binary File I/O

```lua
-- เขียน binary file
local function write_binary_demo(filepath)
    local f = io.open(filepath, "wb")
    if not f then return false end
    
    -- เขียน bytes แบบต่างๆ
    f:write("\x00\x01\x02\x03")   -- 4 bytes
    f:write(string.char(255, 254, 253, 252))  -- อีก 4 bytes
    f:write("TEXT")                -- ASCII text
    
    f:close()
    return true
end

-- อ่าน binary file
local function read_binary_demo(filepath)
    local f = io.open(filepath, "rb")
    if not f then return nil end
    
    local bytes = {}
    local byte = f:read(1)
    while byte do
        bytes[#bytes + 1] = string.byte(byte)
        byte = f:read(1)
    end
    
    f:close()
    return bytes
end

local bin_file = "/tmp/test.bin"
write_binary_demo(bin_file)

local bytes = read_binary_demo(bin_file)
io.write("Bytes: ")
for _, b in ipairs(bytes) do
    io.write(string.format("%02X ", b))
end
print()
```

```lua
-- สร้างและอ่าน BMP header อย่างง่าย
local function pack_uint16_le(n)
    return string.char(n & 0xFF, (n >> 8) & 0xFF)
end

local function pack_uint32_le(n)
    return string.char(
        n & 0xFF,
        (n >> 8) & 0xFF,
        (n >> 16) & 0xFF,
        (n >> 24) & 0xFF
    )
end

local function unpack_uint32_le(s, offset)
    offset = offset or 1
    local b1, b2, b3, b4 = string.byte(s, offset, offset + 3)
    return b1 | (b2 << 8) | (b3 << 16) | (b4 << 24)
end

-- สร้าง BMP header
local function write_simple_bmp(filepath, width, height)
    local f = io.open(filepath, "wb")
    if not f then return false end
    
    local pixel_data_size = width * height * 3  -- RGB
    local file_size = 54 + pixel_data_size
    
    -- BMP File Header (14 bytes)
    f:write("BM")                         -- Signature
    f:write(pack_uint32_le(file_size))     -- File size
    f:write(pack_uint32_le(0))             -- Reserved
    f:write(pack_uint32_le(54))            -- Pixel data offset
    
    -- DIB Header (40 bytes)
    f:write(pack_uint32_le(40))            -- Header size
    f:write(pack_uint32_le(width))         -- Width
    f:write(pack_uint32_le(height))        -- Height
    f:write(pack_uint16_le(1))             -- Color planes
    f:write(pack_uint16_le(24))            -- Bits per pixel
    f:write(pack_uint32_le(0))             -- Compression
    f:write(pack_uint32_le(pixel_data_size)) -- Image size
    f:write(pack_uint32_le(2835))          -- X pixels per meter
    f:write(pack_uint32_le(2835))          -- Y pixels per meter
    f:write(pack_uint32_le(0))             -- Total colors
    f:write(pack_uint32_le(0))             -- Important colors
    
    -- Pixel data: สีแดงล้วน
    for y = 1, height do
        for x = 1, width do
            f:write("\xFF\x00\x00")  -- BGR: สีน้ำเงิน=0, สีเขียว=0, สีแดง=255
        end
    end
    
    f:close()
    return true
end

-- สร้าง BMP ขนาด 10x10
if write_simple_bmp("/tmp/test.bmp", 10, 10) then
    print("สร้าง BMP แล้ว (10x10 pixels สีแดง)")
    
    -- ตรวจสอบ header
    local f = io.open("/tmp/test.bmp", "rb")
    local sig = f:read(2)
    local size_data = f:read(4)
    f:close()
    
    print("Signature:", sig)
    print("File size:", unpack_uint32_le(size_data), "bytes")
end
```

---

## 9.14 io.popen() - Process / Pipe

```lua
-- io.popen(cmd, mode) - เปิด subprocess และอ่าน/เขียน output
-- mode "r" - อ่าน stdout ของ command
-- mode "w" - เขียนไปยัง stdin ของ command

-- อ่าน output ของ command
local function run_command(cmd)
    local handle = io.popen(cmd, "r")
    if not handle then
        return nil, "ไม่สามารถรัน command"
    end
    local result = handle:read("a")
    handle:close()
    return result
end

-- แสดงวันที่และเวลา
local date_output = run_command("date")
print("วันที่:", date_output:gsub("\n", ""))

-- แสดงไฟล์ใน /tmp
local ls_output = run_command("ls /tmp/*.txt 2>/dev/null")
print("ไฟล์ .txt ใน /tmp:")
if ls_output and ls_output ~= "" then
    for file in ls_output:gmatch("[^\n]+") do
        print(" ", file)
    end
end
```

```lua
-- อ่าน output แบบ streaming
local function stream_command(cmd, callback)
    local handle = io.popen(cmd, "r")
    if not handle then return end
    
    local line_num = 0
    for line in handle:lines() do
        line_num = line_num + 1
        callback(line, line_num)
    end
    
    handle:close()
end

stream_command("ls /tmp/", function(line, num)
    if num <= 5 then  -- แสดงแค่ 5 บรรทัดแรก
        print(string.format("  %d: %s", num, line))
    end
end)
```

```lua
-- ตัวอย่าง: wc (word count) ด้วย popen
local function wc_file(filepath)
    local cmd = string.format("wc -lwc '%s' 2>/dev/null", filepath)
    local output = run_command(cmd)
    if not output then return nil end
    
    local lines, words, bytes = output:match("(%d+)%s+(%d+)%s+(%d+)")
    if lines then
        return tonumber(lines), tonumber(words), tonumber(bytes)
    end
    return nil
end

local function run_command(cmd)
    local handle = io.popen(cmd, "r")
    if not handle then return nil end
    local result = handle:read("a")
    handle:close()
    return result
end

local l, w, b = wc_file(test_file)
if l then
    print(string.format("ไฟล์: %d บรรทัด, %d คำ, %d bytes", l, w, b))
end
```

---

## 9.15 os.clock(), os.time(), os.date()

```lua
-- os.clock(): CPU time ที่ใช้ไป (วินาที)
local start = os.clock()

-- ทำงานบางอย่าง
local sum = 0
for i = 1, 1000000 do
    sum = sum + i
end

local elapsed = os.clock() - start
print(string.format("ใช้เวลา CPU: %.4f วินาที", elapsed))
print("ผลลัพธ์:", sum)
```

```lua
-- os.time(): Unix timestamp (วินาทีตั้งแต่ 1970-01-01 00:00:00 UTC)
local now = os.time()
print("Unix timestamp:", now)

-- os.time(table): แปลง date table เป็น timestamp
local t = os.time({
    year = 2024,
    month = 1,
    day = 1,
    hour = 0,
    min = 0,
    sec = 0
})
print("2024-01-01 timestamp:", t)

-- os.difftime(t2, t1): ความต่างของเวลา (วินาที)
local diff = os.difftime(os.time(), t)
print(string.format("ผ่านมาแล้ว: %.0f วัน", diff / 86400))
```

```lua
-- os.date(): format วันที่และเวลา
-- os.date(format, time)
-- ถ้าไม่ระบุ time จะใช้เวลาปัจจุบัน

print(os.date())                     -- Mon Jan  1 00:00:00 2024
print(os.date("%Y-%m-%d"))           -- 2024-01-01
print(os.date("%H:%M:%S"))           -- 00:00:00
print(os.date("%d/%m/%Y %H:%M"))     -- 01/01/2024 00:00
print(os.date("%A, %B %d, %Y"))      -- Monday, January 01, 2024

-- Format codes:
-- %Y - ปี 4 หลัก
-- %m - เดือน (01-12)
-- %d - วันที่ (01-31)
-- %H - ชั่วโมง 24h (00-23)
-- %M - นาที (00-59)
-- %S - วินาที (00-59)
-- %A - ชื่อวันเต็ม
-- %a - ชื่อวันย่อ
-- %B - ชื่อเดือนเต็ม
-- %b - ชื่อเดือนย่อ
-- %p - AM/PM
-- %I - ชั่วโมง 12h (01-12)
```

```lua
-- os.date("*t"): คืน table แทน string
local dt = os.date("*t")
print("year:", dt.year)
print("month:", dt.month)
print("day:", dt.day)
print("hour:", dt.hour)
print("min:", dt.min)
print("sec:", dt.sec)
print("wday:", dt.wday)   -- 1=อาทิตย์, 7=เสาร์
print("yday:", dt.yday)   -- วันที่เท่าไหร่ของปี
print("isdst:", dt.isdst) -- Daylight Saving Time

-- ชื่อวันในภาษาไทย
local thai_days = {"อาทิตย์", "จันทร์", "อังคาร", "พุธ", "พฤหัสบดี", "ศุกร์", "เสาร์"}
print("วัน:", thai_days[dt.wday])
```

```lua
-- ตัวอย่าง: คำนวณวันเกิดครบรอบ
local function days_until_birthday(birth_month, birth_day)
    local now = os.date("*t")
    local this_year_bday = os.time({
        year = now.year,
        month = birth_month,
        day = birth_day,
        hour = 0, min = 0, sec = 0
    })
    
    local now_time = os.time()
    local diff = os.difftime(this_year_bday, now_time)
    
    if diff < 0 then
        -- วันเกิดผ่านไปแล้วในปีนี้ คำนวณปีหน้า
        this_year_bday = os.time({
            year = now.year + 1,
            month = birth_month,
            day = birth_day,
            hour = 0, min = 0, sec = 0
        })
        diff = os.difftime(this_year_bday, now_time)
    end
    
    return math.ceil(diff / 86400)
end

local days = days_until_birthday(12, 25)  -- Christmas
print(string.format("วันคริสต์มาสอีก %d วัน", days))
```

---

## 9.16 os.exit(), os.getenv(), os.tmpname()

```lua
-- os.getenv(varname): อ่าน environment variable
local home = os.getenv("HOME") or "ไม่พบ"
local path = os.getenv("PATH") or "ไม่พบ"
local user = os.getenv("USER") or os.getenv("USERNAME") or "ไม่พบ"

print("HOME:", home)
print("USER:", user)
print("PATH (ส่วนแรก):", path:match("^[^:]+"))
```

```lua
-- os.tmpname(): ชื่อไฟล์ temporary ที่ unique
local tmpfile = os.tmpname()
print("ชื่อไฟล์ temp:", tmpfile)

-- สร้างและลบไฟล์ temp
local f = io.open(tmpfile, "w")
if f then
    f:write("ข้อมูลชั่วคราว\n")
    f:close()
    
    -- อ่านกลับ
    local content = io.open(tmpfile, "r"):read("a")
    io.open(tmpfile, "r"):close()
    print("เนื้อหา temp:", content:gsub("\n", ""))
    
    -- ลบไฟล์
    os.remove(tmpfile)
    print("ลบไฟล์ temp แล้ว")
end
```

```lua
-- os.exit(code): ออกจากโปรแกรม
-- os.exit(0)   -- ออกปกติ (success)
-- os.exit(1)   -- ออกด้วย error code
-- os.exit(false) -- ออกด้วย error code
-- os.exit(true)  -- ออกปกติ

-- ตัวอย่าง: graceful shutdown
local function shutdown(message, code)
    print("กำลังปิดโปรแกรม:", message)
    -- cleanup operations here
    -- os.exit(code or 0)
    print("(simulation: would exit with code " .. (code or 0) .. ")")
end

shutdown("งานเสร็จสิ้น")
```

---

## 9.17 ตัวอย่างขั้นสูง: ระบบ Config Management

```lua
-- ระบบจัดการ configuration file ที่สมบูรณ์
local Config = {}
Config.__index = Config

function Config.new(filepath)
    local self = setmetatable({}, Config)
    self.filepath = filepath
    self.data = {}
    self:load()
    return self
end

function Config:load()
    local f, err = io.open(self.filepath, "r")
    if not f then
        -- ไฟล์ไม่มีอยู่ เริ่มใหม่
        return
    end
    
    local section = "default"
    
    for line in f:lines() do
        -- trim whitespace
        line = line:match("^%s*(.-)%s*$")
        
        -- ข้าม comments และบรรทัดว่าง
        if line == "" or line:match("^[#;]") then
            -- skip
        -- Section header [section_name]
        elseif line:match("^%[(.+)%]$") then
            section = line:match("^%[(.+)%]$")
            if not self.data[section] then
                self.data[section] = {}
            end
        -- key = value
        elseif line:match("^([%w_]+)%s*=%s*(.*)$") then
            local key, value = line:match("^([%w_]+)%s*=%s*(.*)$")
            if not self.data[section] then
                self.data[section] = {}
            end
            -- Auto-type conversion
            if value == "true" then value = true
            elseif value == "false" then value = false
            elseif value:match("^%-?%d+$") then value = tonumber(value)
            elseif value:match("^%-?%d+%.%d+$") then value = tonumber(value)
            end
            self.data[section][key] = value
        end
    end
    
    f:close()
end

function Config:save()
    local f, err = io.open(self.filepath, "w")
    if not f then return false, err end
    
    f:write("# Auto-generated config file\n")
    f:write("# Generated: " .. os.date("%Y-%m-%d %H:%M:%S") .. "\n\n")
    
    for section, values in pairs(self.data) do
        f:write("[" .. section .. "]\n")
        for key, value in pairs(values) do
            f:write(key .. " = " .. tostring(value) .. "\n")
        end
        f:write("\n")
    end
    
    f:close()
    return true
end

function Config:get(section, key, default)
    if self.data[section] and self.data[section][key] ~= nil then
        return self.data[section][key]
    end
    return default
end

function Config:set(section, key, value)
    if not self.data[section] then
        self.data[section] = {}
    end
    self.data[section][key] = value
end

-- ทดสอบ
local conf = Config.new("/tmp/app_config.ini")
conf:set("database", "host", "db.example.com")
conf:set("database", "port", 5432)
conf:set("database", "name", "mydb")
conf:set("server", "host", "0.0.0.0")
conf:set("server", "port", 8080)
conf:set("server", "debug", true)

conf:save()
print("บันทึก config แล้ว")

-- โหลดใหม่
local conf2 = Config.new("/tmp/app_config.ini")
print("DB Host:", conf2:get("database", "host"))
print("DB Port:", conf2:get("database", "port"))
print("Server Debug:", conf2:get("server", "debug"))
print("Missing:", conf2:get("other", "key", "ค่า default"))
```

---

## 9.18 ตัวอย่างขั้นสูง: File Watcher

```lua
-- ตรวจสอบการเปลี่ยนแปลงของไฟล์
local function get_file_info(filepath)
    -- ใช้ os.stat equivalent ผ่าน trick
    -- Lua ไม่มี os.stat โดยตรง ใช้ io.popen
    local cmd = string.format("stat -c '%%s %%Y' '%s' 2>/dev/null", filepath)
    local handle = io.popen(cmd, "r")
    if not handle then return nil end
    local output = handle:read("l")
    handle:close()
    
    if not output then return nil end
    local size, mtime = output:match("(%d+) (%d+)")
    return {
        size = tonumber(size),
        mtime = tonumber(mtime)
    }
end

-- ทดสอบ
local info = get_file_info(test_file)
if info then
    print(string.format("ไฟล์: size=%d, mtime=%d", info.size, info.mtime))
end
```

---

## 9.19 การวัดประสิทธิภาพ I/O

```lua
-- Benchmark การ read/write แบบต่างๆ
local function benchmark(name, fn, iterations)
    iterations = iterations or 1
    local start = os.clock()
    for i = 1, iterations do
        fn(i)
    end
    local elapsed = os.clock() - start
    print(string.format("%-30s: %.4f วินาที", name, elapsed))
    return elapsed
end

-- เตรียมข้อมูล
local large_content = string.rep("X", 1024 * 100)  -- 100KB

-- เปรียบเทียบ write modes
benchmark("write (100KB, 1 call)", function()
    local f = io.open("/tmp/bench.txt", "w")
    f:write(large_content)
    f:close()
end, 10)

benchmark("write (100KB, 1024 calls)", function()
    local f = io.open("/tmp/bench.txt", "w")
    local chunk = string.rep("X", 100)
    for i = 1, 1024 do
        f:write(chunk)
    end
    f:close()
end, 10)

benchmark("read entire file", function()
    local f = io.open("/tmp/bench.txt", "r")
    local _ = f:read("a")
    f:close()
end, 10)

benchmark("read line by line", function()
    local _ = 0
    for _ in io.lines("/tmp/bench.txt") do end
end, 10)
```

---

## 9.20 ตัวอย่างขั้นสูง: Simple HTTP-like Log Analyzer

```lua
-- สร้างไฟล์ log จำลอง
local function create_access_log(filepath, num_entries)
    local f = io.open(filepath, "w")
    if not f then return false end
    
    local ips = {"192.168.1.1", "10.0.0.2", "172.16.0.3", "203.0.113.1"}
    local methods = {"GET", "POST", "PUT", "DELETE"}
    local paths = {"/", "/api/users", "/api/products", "/login", "/logout", "/static/app.js"}
    local statuses = {200, 200, 200, 301, 404, 500}
    
    math.randomseed(42)
    
    for i = 1, num_entries do
        local timestamp = os.date("%d/%b/%Y:%H:%M:%S +0700",
            os.time() - math.random(0, 86400))
        local ip = ips[math.random(#ips)]
        local method = methods[math.random(#methods)]
        local path = paths[math.random(#paths)]
        local status = statuses[math.random(#statuses)]
        local size = math.random(100, 50000)
        
        f:write(string.format('%s - - [%s] "%s %s HTTP/1.1" %d %d\n',
            ip, timestamp, method, path, status, size))
    end
    
    f:close()
    return true
end

-- วิเคราะห์ log
local function analyze_log(filepath)
    local stats = {
        total = 0,
        by_status = {},
        by_method = {},
        by_ip = {},
        by_path = {},
        total_bytes = 0
    }
    
    for line in io.lines(filepath) do
        -- Parse Apache/Nginx combined log format
        local ip, method, path, status, size = line:match(
            '(%S+) %S+ %S+ %[.-%] "(%S+) (%S+) %S+" (%d+) (%d+)'
        )
        
        if ip and status then
            stats.total = stats.total + 1
            stats.total_bytes = stats.total_bytes + tonumber(size)
            
            stats.by_status[status] = (stats.by_status[status] or 0) + 1
            stats.by_method[method] = (stats.by_method[method] or 0) + 1
            stats.by_ip[ip] = (stats.by_ip[ip] or 0) + 1
            stats.by_path[path] = (stats.by_path[path] or 0) + 1
        end
    end
    
    return stats
end

local log_file = "/tmp/access.log"
create_access_log(log_file, 200)

local stats = analyze_log(log_file)

print("=== สถิติ Access Log ===")
print(string.format("รวม requests: %d", stats.total))
print(string.format("รวม data: %.2f KB", stats.total_bytes / 1024))

print("\nสถานะ HTTP:")
local status_list = {}
for s, c in pairs(stats.by_status) do
    status_list[#status_list + 1] = {status = s, count = c}
end
table.sort(status_list, function(a, b) return a.status < b.status end)
for _, s in ipairs(status_list) do
    print(string.format("  %s: %d (%.1f%%)", s.status, s.count,
        s.count / stats.total * 100))
end

print("\nHTTP Methods:")
for method, count in pairs(stats.by_method) do
    print(string.format("  %-8s: %d", method, count))
end

print("\nTop Paths:")
local path_list = {}
for p, c in pairs(stats.by_path) do
    path_list[#path_list + 1] = {path = p, count = c}
end
table.sort(path_list, function(a, b) return a.count > b.count end)
for i = 1, math.min(5, #path_list) do
    print(string.format("  %-25s: %d", path_list[i].path, path_list[i].count))
end
```

---

## 9.21 os Library เพิ่มเติม

```lua
-- os.clock() สำหรับ profiling
local function profile(name, fn)
    local start = os.clock()
    local result = fn()
    local elapsed = os.clock() - start
    print(string.format("[%s] %.6f seconds", name, elapsed))
    return result
end

local result = profile("fibonacci", function()
    local function fib(n)
        if n <= 1 then return n end
        return fib(n-1) + fib(n-2)
    end
    return fib(30)
end)
print("fib(30) =", result)
```

```lua
-- os.date formats สำหรับ logging
local function log_timestamp()
    return os.date("[%Y-%m-%d %H:%M:%S]")
end

print(log_timestamp(), "โปรแกรมเริ่มทำงาน")

-- สร้าง filename ตามวันที่
local function dated_filename(prefix, ext)
    return prefix .. "_" .. os.date("%Y%m%d_%H%M%S") .. "." .. ext
end

print("Log file:", dated_filename("app", "log"))
print("Backup:", dated_filename("backup", "tar.gz"))
```

```lua
-- os.time สำหรับ expiry checking
local function create_token(ttl_seconds)
    return {
        created_at = os.time(),
        expires_at = os.time() + ttl_seconds,
        value = string.format("%x", math.random(0xFFFFFFFF))
    }
end

local function is_valid_token(token)
    return os.time() < token.expires_at
end

local function time_remaining(token)
    return math.max(0, os.difftime(token.expires_at, os.time()))
end

-- ทดสอบ
local token = create_token(3600)  -- valid 1 ชั่วโมง
print("Token:", token.value)
print("Valid:", is_valid_token(token))
print(string.format("เหลือเวลา: %d นาที", time_remaining(token) / 60))
```

---

## แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: File Statistics

เขียนโปรแกรมที่รับ path ไฟล์และแสดงสถิติ:
- จำนวนบรรทัด
- จำนวนคำ
- จำนวน characters
- บรรทัดที่ยาวที่สุดและสั้นที่สุด
- 10 คำที่ปรากฏบ่อยที่สุด

```lua
-- เฉลย
local function file_statistics(filepath)
    local stats = {
        lines = 0,
        words = 0,
        chars = 0,
        longest_line = 0,
        shortest_line = math.huge,
        word_freq = {}
    }
    
    for line in io.lines(filepath) do
        stats.lines = stats.lines + 1
        stats.chars = stats.chars + #line + 1  -- +1 for newline
        
        local line_len = #line
        if line_len > stats.longest_line then stats.longest_line = line_len end
        if line_len < stats.shortest_line then stats.shortest_line = line_len end
        
        for word in line:gmatch("%a+") do
            word = word:lower()
            stats.words = stats.words + 1
            stats.word_freq[word] = (stats.word_freq[word] or 0) + 1
        end
    end
    
    if stats.shortest_line == math.huge then stats.shortest_line = 0 end
    
    -- Top 10 words
    local freq_list = {}
    for word, count in pairs(stats.word_freq) do
        freq_list[#freq_list + 1] = {word = word, count = count}
    end
    table.sort(freq_list, function(a, b) return a.count > b.count end)
    
    stats.top_words = {}
    for i = 1, math.min(10, #freq_list) do
        stats.top_words[i] = freq_list[i]
    end
    
    return stats
end

local s = file_statistics(test_file)
print("=== สถิติไฟล์ ===")
print("บรรทัด:", s.lines)
print("คำ:", s.words)
print("อักขระ:", s.chars)
print("บรรทัดยาวสุด:", s.longest_line)
print("บรรทัดสั้นสุด:", s.shortest_line)
print("\nคำที่ปรากฏบ่อย:")
for i, w in ipairs(s.top_words) do
    print(string.format("  %2d. %-15s: %d ครั้ง", i, w.word, w.count))
end
```

### แบบฝึกหัดที่ 2: JSON-like Serializer

เขียน serializer ที่แปลง Lua table เป็น JSON string และกลับ

```lua
-- เฉลย: JSON Serializer
local JSON = {}

function JSON.encode(val, indent, current_indent)
    indent = indent or ""
    current_indent = current_indent or ""
    local next_indent = current_indent .. indent
    
    local t = type(val)
    
    if t == "nil" then
        return "null"
    elseif t == "boolean" then
        return tostring(val)
    elseif t == "number" then
        if val ~= val then return "null"  -- NaN
        elseif val == math.huge then return "1e999"
        elseif val == -math.huge then return "-1e999"
        elseif math.type(val) == "integer" then return tostring(val)
        else return string.format("%.10g", val)
        end
    elseif t == "string" then
        return '"' .. val:gsub('\\', '\\\\')
                         :gsub('"', '\\"')
                         :gsub('\n', '\\n')
                         :gsub('\t', '\\t')
                         :gsub('\r', '\\r') .. '"'
    elseif t == "table" then
        -- Detect array
        local is_array = true
        local max_idx = 0
        for k in pairs(val) do
            if type(k) ~= "number" or k ~= math.floor(k) or k < 1 then
                is_array = false
                break
            end
            if k > max_idx then max_idx = k end
        end
        if max_idx ~= #val then is_array = false end
        
        local parts = {}
        if is_array then
            for i, v in ipairs(val) do
                if indent ~= "" then
                    parts[i] = next_indent .. JSON.encode(v, indent, next_indent)
                else
                    parts[i] = JSON.encode(v, indent, next_indent)
                end
            end
            if indent ~= "" then
                return "[\n" .. table.concat(parts, ",\n") .. "\n" .. current_indent .. "]"
            else
                return "[" .. table.concat(parts, ",") .. "]"
            end
        else
            for k, v in pairs(val) do
                local key = JSON.encode(tostring(k))
                local value = JSON.encode(v, indent, next_indent)
                if indent ~= "" then
                    parts[#parts + 1] = next_indent .. key .. ": " .. value
                else
                    parts[#parts + 1] = key .. ":" .. value
                end
            end
            if indent ~= "" then
                return "{\n" .. table.concat(parts, ",\n") .. "\n" .. current_indent .. "}"
            else
                return "{" .. table.concat(parts, ",") .. "}"
            end
        end
    else
        return '"[' .. t .. ']"'
    end
end

-- ทดสอบ
local data = {
    name = "สมชาย",
    age = 25,
    scores = {85, 92, 78},
    address = {
        city = "กรุงเทพ",
        zip = "10110"
    },
    active = true
}

-- Compact JSON
print("Compact JSON:")
print(JSON.encode(data))
print()

-- Pretty JSON
print("Pretty JSON:")
print(JSON.encode(data, "  "))
```

### แบบฝึกหัดที่ 3: Rolling Log File

เขียน logger ที่:
1. แบ่งไฟล์ log อัตโนมัติเมื่อถึงขนาดที่กำหนด
2. เก็บไฟล์ log เก่าไม่เกิน N ไฟล์
3. บีบอัด log เก่า (simulate ด้วยการเพิ่ม .old extension)

### แบบฝึกหัดที่ 4: Directory Scanner

เขียนโปรแกรมที่ scan directory (ด้วย io.popen + ls/find) และสร้าง report:
- รายการไฟล์ทั้งหมด
- ขนาดรวม
- ไฟล์ที่แก้ไขล่าสุด
- สถิติตาม extension

### แบบฝึกหัดที่ 5: Simple Database

เขียน flat-file database ที่:
- เก็บข้อมูลใน CSV format
- รองรับ CRUD operations
- มี simple query (ค้นหาตาม field)
- Auto-save เมื่อมีการเปลี่ยนแปลง

---

## ตาราง Quick Reference

### io Functions

| ฟังก์ชัน | หน้าที่ |
|---------|---------|
| `io.open(path, mode)` | เปิดไฟล์ |
| `io.close(file)` | ปิดไฟล์ |
| `io.read(...)` | อ่านจาก stdin |
| `io.write(...)` | เขียนไป stdout |
| `io.lines(file)` | iterator อ่านทีละบรรทัด |
| `io.input(file)` | เปลี่ยน default input |
| `io.output(file)` | เปลี่ยน default output |
| `io.popen(cmd, mode)` | เปิด subprocess |
| `io.flush()` | flush output buffer |

### File Methods

| method | หน้าที่ |
|--------|---------|
| `f:read(fmt)` | อ่านข้อมูล |
| `f:write(...)` | เขียนข้อมูล |
| `f:close()` | ปิดไฟล์ |
| `f:lines()` | iterator |
| `f:seek(whence, offset)` | เลื่อน position |
| `f:flush()` | flush buffer |

### os Functions

| ฟังก์ชัน | หน้าที่ |
|---------|---------|
| `os.time()` | Unix timestamp |
| `os.date(fmt)` | format วันที่ |
| `os.clock()` | CPU time |
| `os.difftime(t2, t1)` | ผลต่างเวลา |
| `os.exit(code)` | ออกจากโปรแกรม |
| `os.getenv(var)` | อ่าน env variable |
| `os.tmpname()` | ชื่อไฟล์ temp |
| `os.remove(path)` | ลบไฟล์ |
| `os.rename(old, new)` | เปลี่ยนชื่อไฟล์ |

### Read Format Codes

| Format | หน้าที่ |
|--------|---------|
| `"l"` หรือ `"*l"` | อ่านบรรทัด (ไม่รวม \\n) |
| `"L"` หรือ `"*L"` | อ่านบรรทัด (รวม \\n) |
| `"n"` หรือ `"*n"` | อ่าน number |
| `"a"` หรือ `"*a"` | อ่านทั้งหมด |
| number `n` | อ่าน n bytes |

---

## สรุปบทที่ 9

I/O ใน Lua มีความยืดหยุ่นสูงและใช้งานง่าย หลักการสำคัญ:

1. **ปิดไฟล์ทุกครั้ง** — หรือใช้ pattern `with_file`
2. **ตรวจสอบ error** — `io.open` คืน nil เมื่อล้มเหลว
3. **ใช้ buffer** — `io.write` + `io.flush` ดีกว่า `print` สำหรับ streaming
4. **io.lines()** — elegant สำหรับอ่านทีละบรรทัด
5. **os.time() + os.date()** — ใช้สำหรับ timestamp และ formatting

I/O เป็นพื้นฐานของทุกโปรแกรมที่ทำงานกับข้อมูลจริง เข้าใจดีจะทำให้เขียนโปรแกรม Lua ที่ robust และ production-ready ได้!
