# บทที่ 1: บทนำสู่ภาษา Lua

## สารบัญ

1. [ประวัติของภาษา Lua](#ประวัติของภาษา-lua)
2. [ทำไมต้องใช้ Lua?](#ทำไมต้องใช้-lua)
3. [กรณีการใช้งาน](#กรณีการใช้งาน)
4. [เปรียบเทียบกับภาษาอื่น](#เปรียบเทียบกับภาษาอื่น)
5. [การติดตั้ง](#การติดตั้ง)
6. [โปรแกรมแรก](#โปรแกรมแรก)
7. [Interactive Mode vs Script Mode](#interactive-mode-vs-script-mode)
8. [ความคิดเห็น (Comments)](#ความคิดเห็น-comments)
9. [กฎไวยากรณ์พื้นฐาน](#กฎไวยากรณ์พื้นฐาน)
10. [ตัวแปรเบื้องต้น](#ตัวแปรเบื้องต้น)
11. [ฟังก์ชัน print()](#ฟังก์ชัน-print)
12. [ฟังก์ชัน tostring() และ tonumber()](#ฟังก์ชัน-tostring-และ-tonumber)
13. [การกำหนดค่าหลายตัวพร้อมกัน](#การกำหนดค่าหลายตัวพร้อมกัน)
14. [ตัวแปร _VERSION](#ตัวแปร-_version)
15. [io.write()](#iowrite)
16. [การเชื่อมต่อ String ด้วย .. operator](#การเชื่อมต่อ-string-ด้วย--operator)
17. [แบบฝึกหัด](#แบบฝึกหัด)

---

## ประวัติของภาษา Lua

ภาษา Lua ถูกสร้างขึ้นที่ **Pontifical Catholic University of Rio de Janeiro (PUC-Rio)** ในประเทศบราซิล เมื่อปี **ค.ศ. 1993** โดยทีมนักพัฒนา 3 คน ได้แก่:

- **Roberto Ierusalimschy** - ผู้นำทีมพัฒนาและสถาปนิกหลัก
- **Waldemar Celes** - ผู้พัฒนา C API และระบบหน่วยความจำ
- **Luiz Henrique de Figueiredo** - ผู้พัฒนาไวยากรณ์และ parser

ชื่อ "Lua" มาจากภาษาโปรตุเกส แปลว่า **"ดวงจันทร์"** ซึ่งสะท้อนถึงความเรียบง่ายและความงดงามของภาษา

### เส้นทางการพัฒนา

| เวอร์ชัน | ปี     | ความสำคัญ                                          |
|---------|--------|-----------------------------------------------------|
| 1.0     | 1993   | เวอร์ชันแรก ใช้ภายใน PUC-Rio                       |
| 2.0     | 1994   | เพิ่ม fallthrough ใน table constructors             |
| 3.0     | 1997   | เพิ่ม pattern matching และ debug library             |
| 4.0     | 2000   | เพิ่ม coroutines เบื้องต้น                          |
| 5.0     | 2003   | Register-based VM, proper tail calls                |
| 5.1     | 2006   | ใช้กันอย่างแพร่หลาย (World of Warcraft)            |
| 5.2     | 2011   | Bitwise operators ผ่าน library                      |
| 5.3     | 2015   | Bitwise operators เป็น built-in, integer subtype    |
| 5.4     | 2020   | Generational GC, integer ชัดเจนขึ้น (เวอร์ชันล่าสุด) |

### แรงบันดาลใจในการสร้าง Lua

Lua ถูกสร้างขึ้นจากความต้องการของ Petrobras (บริษัทน้ำมันของบราซิล) ที่ต้องการภาษา scripting ขนาดเล็กสำหรับกำหนดค่า configuration ในโปรแกรมวิศวกรรม ทีมพัฒนาต้องการภาษาที่:

1. ฝัง (embed) เข้าไปในโปรแกรม C ได้ง่าย
2. ขนาดเล็ก ไม่กิน RAM มาก
3. เรียนรู้ได้ง่าย ไม่ซับซ้อน
4. ส่ง data ระหว่าง C กับ scripting ได้
5. ทำงานเร็วพอสำหรับงาน real-time

---

## ทำไมต้องใช้ Lua?

### 1. ขนาดเล็กมาก (Lightweight)

Lua interpreter มีขนาดไม่เกิน **250KB** (ไฟล์ binary) ซึ่งเล็กกว่า Python ประมาณ **100 เท่า** เหมาะสำหรับ:
- Embedded systems ที่ RAM จำกัด
- โปรแกรมที่ต้องการลด overhead
- Mobile applications

### 2. ฝังได้ง่าย (Embeddable)

Lua มี C API ที่สมบูรณ์ สามารถฝังเข้าไปในโปรแกรม C/C++ ได้ง่าย:

```c
// ตัวอย่างการฝัง Lua ใน C (ไม่ต้องรันตอนนี้ แค่ดูโครงสร้าง)
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>

int main() {
    lua_State *L = luaL_newstate();  // สร้าง Lua instance
    luaL_openlibs(L);                // โหลด standard libraries
    luaL_dofile(L, "script.lua");    // รัน Lua script
    lua_close(L);                    // ปิด Lua instance
    return 0;
}
```

### 3. ทำงานเร็ว (Fast)

- Lua เป็น interpreted language ที่เร็วที่สุดภาษาหนึ่ง
- มี **LuaJIT** (Just-In-Time compiler) ที่ทำให้เร็วกว่า C ที่เขียนไม่ดีได้
- Benchmarks แสดงว่า LuaJIT เร็วกว่า Python 10-100 เท่า

### 4. เรียนรู้ง่าย (Simple and Clean)

Lua มี keywords เพียง **22 คำ** เท่านั้น:
```
and       break     do        else      elseif
end       false     for       function  goto
if        in        local     nil       not
or        repeat    return    then      true
until     while
```

### 5. Dynamic Typing และ Flexible

```lua
-- Lua มี dynamic typing - ตัวแปรเปลี่ยนชนิดได้
local x = 42        -- number
x = "hello"         -- string (ไม่มี error)
x = true            -- boolean (ยังไม่มี error)
x = nil             -- nil (ลบตัวแปรออกจาก memory)

print(x)  -- nil
```

---

## กรณีการใช้งาน

### 1. เกม (Game Development)

**World of Warcraft (WoW)**
- Blizzard Entertainment ใช้ Lua สำหรับ UI addon system
- ผู้เล่นสามารถเขียน addon ด้วย Lua เพื่อปรับแต่ง interface
- มี addon มากกว่า 100,000 ตัวบน CurseForge

```lua
-- ตัวอย่าง WoW addon (โครงสร้างทั่วไป)
-- สร้าง frame ใหม่
local myFrame = CreateFrame("Frame", "MyAddon", UIParent)
myFrame:SetSize(200, 100)
myFrame:SetPoint("CENTER")

-- สร้าง text
local myText = myFrame:CreateFontString(nil, "OVERLAY", "GameFontNormal")
myText:SetPoint("CENTER")
myText:SetText("Hello from Lua!")
```

**Roblox**
- แพลตฟอร์มเกมที่ใหญ่ที่สุดในโลกสำหรับเด็ก
- ใช้ Lua (เวอร์ชัน Luau ที่ปรับปรุงแล้ว) สำหรับ game scripting ทั้งหมด
- นักพัฒนากว่า 9 ล้านคนใช้งาน

```lua
-- ตัวอย่าง Roblox script (Luau)
local Players = game:GetService("Players")

Players.PlayerAdded:Connect(function(player)
    print("ผู้เล่นใหม่เข้ามา: " .. player.Name)
    
    -- ให้เงินเริ่มต้น
    local leaderstats = Instance.new("Folder")
    leaderstats.Name = "leaderstats"
    leaderstats.Parent = player
    
    local coins = Instance.new("IntValue")
    coins.Name = "Coins"
    coins.Value = 100
    coins.Parent = leaderstats
end)
```

**เกมอื่นๆ ที่ใช้ Lua:**
- Garry's Mod (GMod)
- Factorio
- Angry Birds
- Civilization V และ VI
- Far Cry series
- Saints Row series

### 2. Web Server (Nginx/OpenResty)

**OpenResty** คือ web platform ที่ผสาน Nginx กับ LuaJIT ทำให้สามารถเขียน web application logic ได้โดยตรงใน Nginx:

```lua
-- ตัวอย่าง OpenResty/Nginx Lua (nginx.conf)
-- location /hello {
--     content_by_lua_block {
local name = ngx.var.arg_name or "World"
ngx.say("สวัสดี " .. name .. "!")
ngx.say("เวลาปัจจุบัน: " .. os.date())
--     }
-- }
```

ประโยชน์:
- Handle ได้หลายแสน requests ต่อวินาที
- ลด latency ด้วยการเขียน logic ใน Nginx โดยตรง
- ใช้โดย CloudFlare, Taobao, Youku

### 3. Database Configuration (Redis)

Redis ใช้ Lua สำหรับ scripting ที่ atomic:

```lua
-- ตัวอย่าง Redis Lua script
-- รันด้วยคำสั่ง: redis-cli EVAL "script" numkeys key1 arg1

-- Atomic increment กับ rate limiting
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local current = redis.call("INCR", key)

if current == 1 then
    redis.call("EXPIRE", key, 60)  -- หมดอายุใน 60 วินาที
end

if current > limit then
    return 0  -- เกิน rate limit
else
    return 1  -- อนุญาต
end
```

### 4. Text Editor Configuration (Neovim)

Neovim ใช้ Lua แทน VimScript สำหรับการกำหนดค่า:

```lua
-- ตัวอย่าง Neovim configuration (~/.config/nvim/init.lua)

-- ตั้งค่าพื้นฐาน
vim.opt.number = true           -- แสดงเลขบรรทัด
vim.opt.relativenumber = true   -- เลขบรรทัดแบบ relative
vim.opt.tabstop = 4             -- ขนาด tab
vim.opt.shiftwidth = 4          -- ขนาด indent
vim.opt.expandtab = true        -- ใช้ spaces แทน tabs
vim.opt.termguicolors = true    -- รองรับ 256 สี

-- กำหนด keybindings
vim.keymap.set('n', '<leader>ff', ':Telescope find_files<CR>')
vim.keymap.set('n', '<leader>fg', ':Telescope live_grep<CR>')

print("Neovim config โหลดเสร็จแล้ว!")
```

### 5. Embedded Systems และ IoT

```lua
-- ตัวอย่าง NodeMCU (ESP8266/ESP32) - Lua บน microcontroller
-- ควบคุม LED ด้วย Lua

local pin = 4  -- GPIO pin 4

gpio.mode(pin, gpio.OUTPUT)

-- กระพริบ LED 10 ครั้ง
for i = 1, 10 do
    gpio.write(pin, gpio.HIGH)
    tmr.delay(500000)   -- รอ 0.5 วินาที (microseconds)
    gpio.write(pin, gpio.LOW)
    tmr.delay(500000)
    print("กระพริบครั้งที่: " .. i)
end
```

---

## เปรียบเทียบกับภาษาอื่น

### Lua vs Python

| ด้าน              | Lua                     | Python                   |
|-------------------|-------------------------|--------------------------|
| ขนาด interpreter  | ~250KB                  | ~30MB+                   |
| ความเร็ว          | เร็วกว่า 10-50x          | ช้ากว่า                  |
| การฝังใน C        | ง่ายมาก, built-in        | ยุ่งยาก, ต้องใช้ ctypes  |
| Libraries         | น้อยกว่า                | มาก (PyPI)               |
| Community         | เล็กกว่า                | ใหญ่มาก                  |
| การใช้งาน         | Games, embedded         | Data science, web, AI    |
| Array index       | เริ่มที่ 1              | เริ่มที่ 0               |
| Syntax            | end แทน {}              | indentation              |

```lua
-- Lua: ลูป for
for i = 1, 5 do
    print("Lua: " .. i)
end
```

```python
# Python: ลูป for (เปรียบเทียบ - ไม่ต้องรัน)
for i in range(1, 6):
    print(f"Python: {i}")
```

### Lua vs JavaScript

| ด้าน              | Lua                     | JavaScript               |
|-------------------|-------------------------|--------------------------|
| Environment       | Embedded, standalone    | Browser, Node.js         |
| Prototype chain   | ไม่มี (ใช้ metatables)  | มี prototype chain       |
| Table/Object      | Table ทำหน้าที่ทั้งคู่  | Array และ Object แยกกัน  |
| Async             | Coroutines              | Promises, async/await    |
| Type system       | Dynamic, 8 types        | Dynamic, 7 types         |
| Semicolons        | Optional                | Optional                 |

```lua
-- Lua: function และ table
local person = {
    name = "สมชาย",
    age = 25,
    greet = function(self)
        print("สวัสดี ฉันชื่อ " .. self.name)
    end
}

person:greet()  -- Output: สวัสดี ฉันชื่อ สมชาย
```

---

## การติดตั้ง

### Linux (Ubuntu/Debian)

```bash
# วิธีที่ 1: ผ่าน apt package manager
sudo apt update
sudo apt install lua5.4

# ตรวจสอบการติดตั้ง
lua5.4 -v
# Output: Lua 5.4.x  Copyright (C) 1994-2023 Lua.org, PUC-Rio

# วิธีที่ 2: ติดตั้งพร้อม LuaRocks (package manager)
sudo apt install lua5.4 luarocks

# ตรวจสอบ LuaRocks
luarocks --version
```

```bash
# Linux (Fedora/RHEL/CentOS)
sudo dnf install lua lua-devel

# Arch Linux
sudo pacman -S lua
```

### macOS

```bash
# ผ่าน Homebrew (แนะนำ)
brew install lua

# ตรวจสอบ
lua -v

# ติดตั้ง LuaRocks ด้วย
brew install luarocks
```

### Windows

**วิธีที่ 1: ผ่าน Scoop**
```powershell
# ติดตั้ง Scoop ก่อน (ถ้ายังไม่มี)
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
irm get.scoop.sh | iex

# ติดตั้ง Lua
scoop install lua
```

**วิธีที่ 2: ดาวน์โหลด Binary**
1. ไปที่ https://luabinaries.sourceforge.net/
2. ดาวน์โหลด `lua-5.4.x_Win64_bin.zip`
3. แตกไฟล์ไปที่ `C:\lua`
4. เพิ่ม `C:\lua` ใน PATH environment variable

**วิธีที่ 3: ผ่าน Chocolatey**
```cmd
choco install lua
```

### ตรวจสอบการติดตั้ง

```bash
lua -v
# ควรแสดง: Lua 5.4.x  Copyright (C) 1994-2023 Lua.org, PUC-Rio
```

---

## โปรแกรมแรก

### Hello World แบบง่ายที่สุด

```lua
-- ตัวอย่างที่ 1: Hello World
print("Hello, World!")
```

**Output:**
```
Hello, World!
```

### Hello World ภาษาไทย

```lua
-- ตัวอย่างที่ 2: Hello World ภาษาไทย
print("สวัสดีชาวโลก!")
print("ยินดีต้อนรับสู่โลกของ Lua")
```

**Output:**
```
สวัสดีชาวโลก!
ยินดีต้อนรับสู่โลกของ Lua
```

### โปรแกรมรับ Input จากผู้ใช้

```lua
-- ตัวอย่างที่ 3: รับ input จาก keyboard
io.write("กรุณาใส่ชื่อของคุณ: ")
local name = io.read()  -- อ่านบรรทัดจาก keyboard
print("สวัสดี " .. name .. "!")
print("ยินดีที่ได้รู้จัก " .. name)
```

**Output (ถ้าพิมพ์ "สมชาย"):**
```
กรุณาใส่ชื่อของคุณ: สมชาย
สวัสดี สมชาย!
ยินดีที่ได้รู้จัก สมชาย
```

### รับตัวเลขและคำนวณ

```lua
-- ตัวอย่างที่ 4: รับตัวเลขและคำนวณ
io.write("ใส่ตัวเลขแรก: ")
local a = tonumber(io.read())

io.write("ใส่ตัวเลขที่สอง: ")
local b = tonumber(io.read())

print("ผลรวม: " .. (a + b))
print("ผลต่าง: " .. (a - b))
print("ผลคูณ: " .. (a * b))

if b ~= 0 then
    print("ผลหาร: " .. (a / b))
else
    print("ไม่สามารถหารด้วยศูนย์ได้!")
end
```

### โปรแกรมแสดงข้อมูลส่วนตัว

```lua
-- ตัวอย่างที่ 5: แสดงข้อมูลส่วนตัว
local firstName = "สมชาย"
local lastName = "ใจดี"
local age = 25
local city = "กรุงเทพมหานคร"

print("=== ข้อมูลส่วนตัว ===")
print("ชื่อ: " .. firstName)
print("นามสกุล: " .. lastName)
print("ชื่อเต็ม: " .. firstName .. " " .. lastName)
print("อายุ: " .. age .. " ปี")
print("เมือง: " .. city)
print("====================")
```

**Output:**
```
=== ข้อมูลส่วนตัว ===
ชื่อ: สมชาย
นามสกุล: ใจดี
ชื่อเต็ม: สมชาย ใจดี
อายุ: 25 ปี
เมือง: กรุงเทพมหานคร
====================
```

---

## Interactive Mode vs Script Mode

### Interactive Mode (REPL)

เปิด terminal แล้วพิมพ์ `lua` (ไม่มี argument):

```bash
$ lua
Lua 5.4.6  Copyright (C) 1994-2023 Lua.org, PUC-Rio
> 
```

```lua
-- ตัวอย่างที่ 6: ใช้ Interactive Mode
-- พิมพ์ทีละบรรทัดใน REPL

> print("Hello from REPL!")
Hello from REPL!

> 2 + 3
-- ใน Lua 5.4 REPL จะแสดง:
-- 5

> x = 42
> print(x)
42

> x * 2
84
```

ออกจาก interactive mode ด้วย:
- `os.exit()` หรือ
- `Ctrl+D` (Linux/Mac) หรือ
- `Ctrl+Z` แล้ว Enter (Windows)

### Script Mode

บันทึก code ลงไฟล์ `.lua` แล้วรัน:

```bash
# สร้างไฟล์ hello.lua แล้วรัน
lua hello.lua
```

```lua
-- ตัวอย่างที่ 7: ไฟล์ script hello.lua
-- สามารถมี shebang line เพื่อรันโดยตรงบน Linux/Mac
#!/usr/bin/env lua

print("นี่คือ script mode!")
print("ไฟล์ถูกรันจาก command line")

-- รับ command line arguments
local args = {...}  -- ตัวแปร vararg ที่เก็บ arguments
print("จำนวน arguments: " .. #args)

for i, arg in ipairs(args) do
    print("  arg[" .. i .. "] = " .. arg)
end
```

```bash
# รันพร้อม arguments
lua hello.lua foo bar baz

# Output:
# นี่คือ script mode!
# ไฟล์ถูกรันจาก command line
# จำนวน arguments: 3
#   arg[1] = foo
#   arg[2] = bar
#   arg[3] = baz
```

### การใช้ shebang บน Linux/Mac

```lua
-- ตัวอย่างที่ 8: ไฟล์ที่รันได้โดยตรง
#!/usr/bin/env lua5.4
-- หรือ #!/usr/local/bin/lua

print("รันโดยตรงโดยไม่ต้องพิมพ์ lua!")
```

```bash
# ให้สิทธิ์รัน
chmod +x myscript.lua

# รันโดยตรง
./myscript.lua
```

---

## ความคิดเห็น (Comments)

### Single-line Comment

```lua
-- ตัวอย่างที่ 9: Single-line comments
-- นี่คือ comment บรรทัดเดียว
-- ขึ้นต้นด้วย -- (double dash)

print("Hello")  -- comment ต่อท้าย code ก็ได้

-- เราใช้ comment เพื่อ:
-- 1. อธิบาย code
-- 2. ปิด code ชั่วคราว (debug)
-- 3. ใส่ metadata หรือ copyright

-- print("บรรทัดนี้ถูกปิดไว้ ไม่จะถูกรัน")
print("บรรทัดนี้จะถูกรัน")
```

### Multi-line Comment (Long Comments)

```lua
--[[
    ตัวอย่างที่ 10: Multi-line comment
    ใช้ --[[ ]]-- หรือ --[[ ]] เพื่อปิด comment หลายบรรทัด
    
    โปรแกรมนี้ทำอะไร:
    - รับ input จากผู้ใช้
    - คำนวณผลลัพธ์
    - แสดงผลลัพธ์
    
    เขียนโดย: สมชาย ใจดี
    วันที่: 2025-01-01
    เวอร์ชัน: 1.0
]]

print("Multi-line comment ข้างบนไม่มีผลต่อ code นี้")
```

```lua
-- ตัวอย่างที่ 11: Comment แบบต่างๆ
-- ปิด block ของ code ด้วย multi-line comment

--[[
local x = 10
local y = 20
print(x + y)
-- code ทั้งหมดนี้ถูกปิดไว้
]]

-- Long brackets ระดับต่างๆ
--[==[
    comment ด้วย level 2 bracket
    ใช้เมื่อ comment มี ]] อยู่ข้างใน
]==]

print("Code นี้ทำงานปกติ")
```

### ความแตกต่างระหว่าง Comment Styles

```lua
-- ตัวอย่างที่ 12: เปรียบเทียบ comment styles

-- 1. Single-line: ง่ายและใช้บ่อยที่สุด
local a = 10  -- ค่าเริ่มต้น

--[[
2. Multi-line: ใช้สำหรับ:
   - Documentation ขนาดใหญ่
   - ปิด code block ชั่วคราว
   - Function documentation
]]

local function add(x, y)
    --[[
    ฟังก์ชันบวกเลขสองตัว
    Parameters:
        x: ตัวเลขแรก (number)
        y: ตัวเลขที่สอง (number)
    Returns:
        ผลรวมของ x และ y (number)
    ]]
    return x + y
end

print(add(3, 4))  -- Output: 7
```

---

## กฎไวยากรณ์พื้นฐาน

### Case Sensitivity

```lua
-- ตัวอย่างที่ 13: Lua เป็น case-sensitive
local name = "สมชาย"
local Name = "สมหญิง"   -- ตัวแปรคนละตัว!
local NAME = "สมศรี"    -- ตัวแปรอีกตัว!

print(name)   -- สมชาย
print(Name)   -- สมหญิง
print(NAME)   -- สมศรี

-- Keywords ต้องเป็น lowercase
-- if, then, end, while, for, do, return, etc.
-- IF หรือ For จะ error!
```

### Semicolons เป็น Optional

```lua
-- ตัวอย่างที่ 14: Semicolons ใน Lua
-- Semicolon ไม่บังคับ แต่ใช้ได้
local x = 10
local y = 20;   -- ใส่ ; ก็ได้
local z = 30; local w = 40;  -- หลายคำสั่งในบรรทัดเดียว

print(x)
print(y)
print(z)
print(w)

-- สไตล์ที่แนะนำ: ไม่ใส่ ; ยกเว้นจำเป็น
```

### Whitespace Insensitive

```lua
-- ตัวอย่างที่ 15: Whitespace ไม่มีผลต่อ Lua (ต่างจาก Python)
local
    a
    =
    10
    
print(
    a
)

-- ผลลัพธ์เหมือนกับ:
-- local a = 10
-- print(a)
```

### Multiple Statements ในบรรทัดเดียว

```lua
-- ตัวอย่างที่ 16: หลาย statement ในบรรทัดเดียว
local a = 1; local b = 2; local c = 3

-- หรือ multiple assignment (ดีกว่า)
local x, y, z = 10, 20, 30

print(a, b, c)  -- 1    2    3
print(x, y, z)  -- 10   20   30
```

### Identifiers (ชื่อตัวแปร)

```lua
-- ตัวอย่างที่ 17: กฎการตั้งชื่อตัวแปร
-- ถูกต้อง:
local myVariable = 1
local _private = 2
local camelCase = 3
local snake_case = 4
local PascalCase = 5
local var123 = 6
local _ = 7  -- underscore ตัวเดียว (ใช้สำหรับค่าที่ไม่ต้องการ)

-- ผิด (จะ error):
-- local 123abc = 8     -- ขึ้นต้นด้วยตัวเลขไม่ได้
-- local my-var = 9     -- ใช้ - ไม่ได้
-- local my var = 10    -- มีช่องว่างไม่ได้

-- Convention ที่นิยมใช้:
local MAX_SIZE = 100         -- constants: UPPER_CASE
local userName = "สมชาย"    -- variables: camelCase
local function myFunction()  -- functions: camelCase
end
local MyClass = {}           -- classes: PascalCase
```

---

## ตัวแปรเบื้องต้น

### Global Variables

```lua
-- ตัวอย่างที่ 18: Global variables
-- ตัวแปรที่ไม่มี local keyword เป็น global

myGlobal = "ฉันเป็น global"
anotherGlobal = 42

print(myGlobal)      -- ฉันเป็น global
print(anotherGlobal) -- 42

-- ทำงานได้ทุกที่ในโปรแกรม
local function test()
    print(myGlobal)  -- เข้าถึง global จาก function ได้
end

test()  -- ฉันเป็น global
```

### Local Variables

```lua
-- ตัวอย่างที่ 19: Local variables
local localVar = "ฉันเป็น local"
print(localVar)  -- ฉันเป็น local

do
    local innerLocal = "ฉันอยู่ในบล็อก"
    print(innerLocal)   -- ฉันอยู่ในบล็อก
    print(localVar)     -- เข้าถึง outer local ได้
end

-- print(innerLocal)   -- ERROR! innerLocal ไม่มีในขอบเขตนี้

-- ข้อแนะนำ: ใช้ local เสมอ เพื่อประสิทธิภาพและความปลอดภัย
```

### การกำหนดค่าตัวแปร

```lua
-- ตัวอย่างที่ 20: การกำหนดค่าตัวแปรแบบต่างๆ
-- กำหนดค่าเดียว
local x = 10
local name = "สมชาย"
local flag = true
local nothing = nil

-- กำหนดค่าโดยไม่ระบุค่าเริ่มต้น (ค่าเป็น nil)
local empty
print(empty)  -- nil

-- เปลี่ยนค่าตัวแปร
x = 20        -- เปลี่ยนค่า
x = "hello"   -- เปลี่ยนชนิดได้เลย (dynamic typing)
print(x)      -- hello
```

---

## ฟังก์ชัน print()

### การใช้งานพื้นฐาน

```lua
-- ตัวอย่างที่ 21: print() พื้นฐาน
print()                    -- บรรทัดว่าง
print("Hello, World!")     -- string
print(42)                  -- number
print(3.14)                -- float
print(true)                -- boolean
print(false)               -- boolean
print(nil)                 -- nil
```

**Output:**
```

Hello, World!
42
3.14
true
false
nil
```

### print() กับหลาย Arguments

```lua
-- ตัวอย่างที่ 22: print() กับหลาย arguments
-- arguments คั่นด้วย tab (\t)
print("ชื่อ", "อายุ", "เมือง")
print("สมชาย", 25, "กรุงเทพ")
print(1, 2, 3, 4, 5)
print("a", "b", "c")
```

**Output:**
```
ชื่อ	อายุ	เมือง
สมชาย	25	กรุงเทพ
1	2	3	4	5
a	b	c
```

### print() vs io.write()

```lua
-- ตัวอย่างที่ 23: เปรียบเทียบ print() และ io.write()

-- print() เพิ่ม newline อัตโนมัติ
print("บรรทัดที่ 1")
print("บรรทัดที่ 2")

-- io.write() ไม่เพิ่ม newline
io.write("ส่วนที่ 1 ")
io.write("ส่วนที่ 2 ")
io.write("ส่วนที่ 3\n")  -- ต้องใส่ \n เอง
```

**Output:**
```
บรรทัดที่ 1
บรรทัดที่ 2
ส่วนที่ 1 ส่วนที่ 2 ส่วนที่ 3
```

---

## ฟังก์ชัน tostring() และ tonumber()

### tostring()

```lua
-- ตัวอย่างที่ 24: tostring() - แปลงเป็น string
local num = 42
local pi = 3.14159
local flag = true
local nothing = nil

print(tostring(num))      -- "42"
print(tostring(pi))       -- "3.14159"
print(tostring(flag))     -- "true"
print(tostring(nothing))  -- "nil"

-- ใช้เพื่อเชื่อม string กับ non-string values
local message = "ค่าคือ: " .. tostring(num)
print(message)  -- ค่าคือ: 42

-- หมายเหตุ: number สามารถ concatenate โดยตรงได้
local msg2 = "ค่าคือ: " .. num  -- Lua แปลงให้อัตโนมัติ
print(msg2)  -- ค่าคือ: 42
```

### tonumber()

```lua
-- ตัวอย่างที่ 25: tonumber() - แปลงเป็น number
local str1 = "42"
local str2 = "3.14"
local str3 = "0xFF"    -- Hex string
local str4 = "hello"   -- ไม่ใช่ตัวเลข
local bool = true

print(tonumber(str1))   -- 42
print(tonumber(str2))   -- 3.14
print(tonumber(str3))   -- 255 (แปลง hex)
print(tonumber(str4))   -- nil (แปลงไม่ได้)
print(tonumber(bool))   -- nil (boolean แปลงไม่ได้)
print(tonumber(42))     -- 42 (number คืนค่าเดิม)

-- ระบุ base (ฐาน)
print(tonumber("1010", 2))   -- 10 (binary)
print(tonumber("FF", 16))    -- 255 (hexadecimal)
print(tonumber("77", 8))     -- 63 (octal)
```

### การตรวจสอบ Conversion

```lua
-- ตัวอย่างที่ 26: ตรวจสอบก่อนแปลง
local input = "123abc"  -- ค่าที่อาจแปลงไม่ได้

local num = tonumber(input)
if num then
    print("แปลงสำเร็จ: " .. num)
else
    print("แปลงไม่ได้ - '" .. input .. "' ไม่ใช่ตัวเลข")
end

-- ตัวอย่างการรับ input และแปลง
io.write("ใส่ตัวเลข: ")
local userInput = io.read()
local number = tonumber(userInput)

if number then
    print("ตัวเลขที่ใส่: " .. number)
    print("คูณ 2: " .. (number * 2))
else
    print("กรุณาใส่ตัวเลขที่ถูกต้อง!")
end
```

---

## การกำหนดค่าหลายตัวพร้อมกัน

### Multiple Assignment พื้นฐาน

```lua
-- ตัวอย่างที่ 27: Multiple assignment
local a, b, c = 1, 2, 3
print(a, b, c)  -- 1    2    3

-- กำหนดค่าไม่ครบ (ตัวที่เกินจะได้ nil)
local x, y, z = 10, 20
print(x, y, z)  -- 10   20   nil

-- ค่ามากกว่าตัวแปร (ค่าที่เกินถูกทิ้ง)
local p, q = 100, 200, 300
print(p, q)  -- 100  200  (300 ถูกทิ้ง)
```

### Swap Variables

```lua
-- ตัวอย่างที่ 28: Swap ค่าตัวแปรด้วย multiple assignment
local a = "apple"
local b = "banana"

print("ก่อน swap: a=" .. a .. ", b=" .. b)

-- Swap แบบ Lua (ไม่ต้องใช้ตัวแปรชั่วคราว!)
a, b = b, a

print("หลัง swap: a=" .. a .. ", b=" .. b)

-- เปรียบเทียบกับภาษาอื่นที่ต้องทำแบบนี้:
-- local temp = a
-- a = b
-- b = temp
```

**Output:**
```
ก่อน swap: a=apple, b=banana
หลัง swap: a=banana, b=apple
```

### Function ที่คืนหลายค่า

```lua
-- ตัวอย่างที่ 29: รับหลายค่าจาก function
local function getMinMax(t)
    local min, max = t[1], t[1]
    for _, v in ipairs(t) do
        if v < min then min = v end
        if v > max then max = v end
    end
    return min, max  -- คืน 2 ค่า
end

local numbers = {5, 3, 8, 1, 9, 2, 7}
local minimum, maximum = getMinMax(numbers)

print("ค่าต่ำสุด: " .. minimum)   -- 1
print("ค่าสูงสุด: " .. maximum)   -- 9
```

### Multiple Assignment กับ String

```lua
-- ตัวอย่างที่ 30: Multiple assignment กับ string operations
local str = "Hello, World!"

-- string.find คืน start, end positions
local start, finish = string.find(str, "World")
print("พบ 'World' ที่ตำแหน่ง: " .. start .. " ถึง " .. finish)

-- string.byte คืน multiple values
local b1, b2, b3 = string.byte("ABC", 1, 3)
print("ASCII: " .. b1 .. ", " .. b2 .. ", " .. b3)
-- Output: ASCII: 65, 66, 67
```

---

## ตัวแปร _VERSION

```lua
-- ตัวอย่างที่ 31: ตัวแปร _VERSION
-- _VERSION เก็บเวอร์ชันของ Lua interpreter

print(_VERSION)        -- Lua 5.4 (หรือเวอร์ชันที่ใช้)
print(type(_VERSION))  -- string

-- ใช้ตรวจสอบเวอร์ชัน
if _VERSION == "Lua 5.4" then
    print("คุณใช้ Lua 5.4 - ยอดเยี่ยม!")
elseif _VERSION == "Lua 5.3" then
    print("คุณใช้ Lua 5.3 - แนะนำให้อัพเกรด")
else
    print("เวอร์ชัน: " .. _VERSION)
end

-- แยก major และ minor version
local major, minor = _VERSION:match("Lua (%d+)%.(%d+)")
print("Major version: " .. major)
print("Minor version: " .. minor)
```

### ตรวจสอบ Features ตามเวอร์ชัน

```lua
-- ตัวอย่างที่ 32: ตรวจสอบ feature ตามเวอร์ชัน
local version = _VERSION:match("Lua (%d+%.%d+)")
local major, minor = version:match("(%d+)%.(%d+)")
major = tonumber(major)
minor = tonumber(minor)

if major >= 5 and minor >= 3 then
    print("รองรับ bitwise operators (&, |, ~, <<, >>)")
else
    print("ต้องใช้ bit library แทน")
end

if major >= 5 and minor >= 4 then
    print("รองรับ integer subtype และ generational GC")
end
```

---

## io.write()

### การใช้งานพื้นฐาน

```lua
-- ตัวอย่างที่ 33: io.write() พื้นฐาน
io.write("Hello")
io.write(", ")
io.write("World")
io.write("!\n")  -- ต้องใส่ newline เอง

-- Output: Hello, World!
```

### io.write() กับ Formatting

```lua
-- ตัวอย่างที่ 34: io.write() สำหรับ output ที่ควบคุมได้
local name = "สมชาย"
local score = 95.5

io.write(string.format("ชื่อ: %-15s คะแนน: %6.2f\n", name, score))
io.write(string.format("ชื่อ: %-15s คะแนน: %6.2f\n", "สมหญิง", 87.3))
io.write(string.format("ชื่อ: %-15s คะแนน: %6.2f\n", "สมศักดิ์", 92.0))
```

**Output:**
```
ชื่อ: สมชาย          คะแนน:  95.50
ชื่อ: สมหญิง         คะแนน:  87.30
ชื่อ: สมศักดิ์        คะแนน:  92.00
```

### Progress Bar ด้วย io.write()

```lua
-- ตัวอย่างที่ 35: Progress bar ด้วย io.write()
local total = 20

io.write("กำลังประมวลผล: [")
for i = 1, total do
    io.write("=")
    io.flush()  -- บังคับให้ flush buffer ทันที
    -- os.execute("sleep 0.1")  -- uncomment เพื่อดู animation
end
io.write("] เสร็จแล้ว!\n")
```

### string.format()

```lua
-- ตัวอย่างที่ 36: string.format() สำหรับจัดรูปแบบ
-- %d = integer, %f = float, %s = string, %q = quoted string
-- %x = hex, %o = octal, %e = scientific notation

print(string.format("จำนวนเต็ม: %d", 42))
print(string.format("ทศนิยม: %.2f", 3.14159))
print(string.format("String: %s", "hello"))
print(string.format("Hex: %x", 255))       -- ff
print(string.format("Hex uppercase: %X", 255))  -- FF
print(string.format("Scientific: %e", 123456789.0))  -- 1.234568e+08
print(string.format("ความกว้าง: %10d", 42))   -- จัดขวา
print(string.format("ซ้าย: %-10d|", 42))     -- จัดซ้าย
print(string.format("เติมศูนย์: %05d", 42))  -- 00042
```

---

## การเชื่อมต่อ String ด้วย .. operator

### Concatenation พื้นฐาน

```lua
-- ตัวอย่างที่ 37: String concatenation ด้วย ..
local first = "สวัสดี"
local second = " "
local third = "โลก"

local result = first .. second .. third
print(result)  -- สวัสดี โลก

-- เชื่อมกับตัวเลข (Lua แปลงให้อัตโนมัติ)
local age = 25
local message = "อายุ " .. age .. " ปี"
print(message)  -- อายุ 25 ปี

-- เชื่อมหลายครั้ง
local greeting = "สวัสดี" .. " " .. "ชาว" .. " " .. "โลก" .. "!"
print(greeting)  -- สวัสดี ชาว โลก!
```

### Concatenation กับตัวแปรชนิดต่างๆ

```lua
-- ตัวอย่างที่ 38: Concatenation กับ number types
local integer = 42
local float = 3.14
local neg = -100

print("integer: " .. integer)  -- integer: 42
print("float: " .. float)      -- float: 3.14
print("negative: " .. neg)     -- negative: -100

-- Concatenate boolean ต้องแปลงก่อน
local flag = true
-- print("flag: " .. flag)       -- ERROR! boolean ต้องแปลงก่อน
print("flag: " .. tostring(flag))  -- flag: true
```

### String สร้างด้วย Concatenation

```lua
-- ตัวอย่างที่ 39: สร้าง string ที่ซับซ้อน
local stars = 5
local line = ""
for i = 1, stars do
    line = line .. "*"
end
print(line)  -- *****

-- สร้าง table display
local header = "+" .. string.rep("-", 20) .. "+"
local row = "| " .. string.format("%-18s", "สินค้า") .. " |"
local footer = "+" .. string.rep("-", 20) .. "+"

print(header)
print(row)
print(footer)
```

**Output:**
```
+--------------------+
| สินค้า             |
+--------------------+
```

### Performance tip: ใช้ table.concat แทน .. ในลูป

```lua
-- ตัวอย่างที่ 40: Performance - table.concat vs ..
-- สำหรับ string ขนาดใหญ่ ให้ใช้ table.concat

-- วิธีที่ช้า (ไม่แนะนำสำหรับ loop ใหญ่)
local result_slow = ""
for i = 1, 100 do
    result_slow = result_slow .. tostring(i) .. " "
end

-- วิธีที่เร็ว (แนะนำ)
local parts = {}
for i = 1, 100 do
    parts[i] = tostring(i)
end
local result_fast = table.concat(parts, " ")

print("Slow result length: " .. #result_slow)
print("Fast result length: " .. #result_fast)
```

### โปรแกรมตัวอย่างสมบูรณ์

```lua
-- ตัวอย่างที่ 41: โปรแกรมสมบูรณ์ - คำนวณ BMI
print("=== โปรแกรมคำนวณ BMI ===")
print()

io.write("ใส่น้ำหนัก (กิโลกรัม): ")
local weight = tonumber(io.read())

io.write("ใส่ส่วนสูง (เซนติเมตร): ")
local height_cm = tonumber(io.read())

if weight and height_cm and weight > 0 and height_cm > 0 then
    local height_m = height_cm / 100
    local bmi = weight / (height_m * height_m)
    
    print()
    print("=== ผลลัพธ์ ===")
    print("น้ำหนัก: " .. weight .. " กก.")
    print("ส่วนสูง: " .. height_cm .. " ซม. (" .. 
          string.format("%.2f", height_m) .. " ม.)")
    print("BMI: " .. string.format("%.2f", bmi))
    
    local category
    if bmi < 18.5 then
        category = "น้ำหนักน้อยกว่าเกณฑ์"
    elseif bmi < 25 then
        category = "น้ำหนักปกติ"
    elseif bmi < 30 then
        category = "น้ำหนักเกิน"
    else
        category = "อ้วน"
    end
    
    print("สถานะ: " .. category)
else
    print("กรุณาใส่ข้อมูลที่ถูกต้อง!")
end
```

### โปรแกรมตารางสูตรคูณ

```lua
-- ตัวอย่างที่ 42: ตารางสูตรคูณ
print("=== ตารางสูตรคูณ ===")
print()

-- หัวตาราง
io.write(string.format("%6s", "x"))
for i = 1, 10 do
    io.write(string.format("%6d", i))
end
io.write("\n")
io.write(string.rep("-", 66) .. "\n")

-- เนื้อหาตาราง
for i = 1, 10 do
    io.write(string.format("%6d", i))
    for j = 1, 10 do
        io.write(string.format("%6d", i * j))
    end
    io.write("\n")
end
```

### โปรแกรมแปลงอุณหภูมิ

```lua
-- ตัวอย่างที่ 43: แปลงอุณหภูมิ
local function celsiusToFahrenheit(c)
    return (c * 9/5) + 32
end

local function fahrenheitToCelsius(f)
    return (f - 32) * 5/9
end

local function celsiusToKelvin(c)
    return c + 273.15
end

print("=== แปลงอุณหภูมิ ===")
print()

local temps = {0, 20, 37, 100}

print(string.format("%-10s %-15s %-15s %-15s",
    "Celsius", "Fahrenheit", "Kelvin", "หมายเหตุ"))
print(string.rep("-", 60))

local notes = {"จุดเยือกแข็งน้ำ", "อุณหภูมิห้อง", "อุณหภูมิร่างกาย", "จุดเดือดน้ำ"}

for i, c in ipairs(temps) do
    local f = celsiusToFahrenheit(c)
    local k = celsiusToKelvin(c)
    print(string.format("%-10.1f %-15.2f %-15.2f %s",
        c, f, k, notes[i]))
end
```

---

## โปรแกรมตัวอย่างเพิ่มเติม

### โปรแกรมแสดงปฏิทินเดือน

```lua
-- ตัวอย่างที่ 44: แสดงปฏิทินอย่างง่าย
local function isLeapYear(year)
    return (year % 4 == 0 and year % 100 ~= 0) or (year % 400 == 0)
end

local function getDaysInMonth(month, year)
    local days = {31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31}
    if month == 2 and isLeapYear(year) then
        return 29
    end
    return days[month]
end

-- แสดงเดือนมกราคม 2025
local month = 1
local year = 2025
local monthNames = {"มกราคม", "กุมภาพันธ์", "มีนาคม", "เมษายน",
                    "พฤษภาคม", "มิถุนายน", "กรกฎาคม", "สิงหาคม",
                    "กันยายน", "ตุลาคม", "พฤศจิกายน", "ธันวาคม"}

print("    " .. monthNames[month] .. " " .. year)
print("อา  จ   อ   พ   พฤ  ศ   ส")
print(string.rep("-", 27))

-- วันแรกของเดือน (สมมติ) - ในที่นี้คือวันพุธ = 3
local firstDay = 3  -- 0=อาทิตย์, 1=จันทร์, ..., 6=เสาร์

local line = string.rep("    ", firstDay)
for day = 1, getDaysInMonth(month, year) do
    line = line .. string.format("%-4d", day)
    if (day + firstDay - 1) % 7 == 6 then
        print(line)
        line = ""
    end
end
if line ~= "" then
    print(line)
end
```

### โปรแกรมเปลี่ยนหน่วยระยะทาง

```lua
-- ตัวอย่างที่ 45: เปลี่ยนหน่วยระยะทาง
print("=== แปลงหน่วยระยะทาง ===")
print()

local function metersToAll(meters)
    return {
        meters = meters,
        km = meters / 1000,
        miles = meters / 1609.344,
        feet = meters * 3.28084,
        inches = meters * 39.3701,
        cm = meters * 100,
        mm = meters * 1000
    }
end

local distances = {1, 100, 1000, 42195}  -- 1m, 100m, 1km, มาราธอน
local labels = {"1 เมตร", "100 เมตร", "1 กิโลเมตร", "มาราธอน"}

for i, d in ipairs(distances) do
    local r = metersToAll(d)
    print("=== " .. labels[i] .. " (" .. d .. " m) ===")
    print(string.format("  กิโลเมตร: %.4f km", r.km))
    print(string.format("  ไมล์: %.4f miles", r.miles))
    print(string.format("  ฟุต: %.2f feet", r.feet))
    print(string.format("  นิ้ว: %.2f inches", r.inches))
    print()
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Hello World ที่กำหนดเอง

เขียนโปรแกรมที่:
1. รับชื่อผู้ใช้
2. รับอายุ
3. แสดงข้อความทักทายที่มีทั้งชื่อและอายุ
4. บอกปีเกิดโดยประมาณ (ปีปัจจุบัน - อายุ)

```lua
-- แบบฝึกหัดที่ 1 - เฉลย
io.write("ชื่อของคุณ: ")
local name = io.read()

io.write("อายุของคุณ: ")
local age = tonumber(io.read())

local currentYear = 2025
local birthYear = currentYear - age

print()
print("สวัสดี " .. name .. "!")
print("คุณอายุ " .. age .. " ปี")
print("คุณเกิดประมาณปี " .. birthYear)
print("ยินดีต้อนรับสู่โลกของ Lua!")
```

### แบบฝึกหัดที่ 2: เครื่องคิดเลข

เขียนเครื่องคิดเลขที่รองรับ +, -, *, /

```lua
-- แบบฝึกหัดที่ 2 - เฉลย
print("=== เครื่องคิดเลข ===")

io.write("ใส่ตัวเลขแรก: ")
local a = tonumber(io.read())

io.write("ใส่ operator (+, -, *, /): ")
local op = io.read()

io.write("ใส่ตัวเลขที่สอง: ")
local b = tonumber(io.read())

local result
if op == "+" then
    result = a + b
elseif op == "-" then
    result = a - b
elseif op == "*" then
    result = a * b
elseif op == "/" then
    if b ~= 0 then
        result = a / b
    else
        print("Error: หารด้วยศูนย์ไม่ได้!")
        os.exit(1)
    end
else
    print("Error: operator ไม่ถูกต้อง!")
    os.exit(1)
end

print()
print(string.format("%.4g %s %.4g = %.4g", a, op, b, result))
```

### แบบฝึกหัดที่ 3: ตรวจสอบ Lua Version

เขียนโปรแกรมที่:
1. แสดงเวอร์ชัน Lua ที่ใช้งาน
2. แยก major และ minor version number
3. แสดงว่ารองรับ features ใดบ้าง

```lua
-- แบบฝึกหัดที่ 3 - เฉลย
print("=== ข้อมูล Lua Interpreter ===")
print("เวอร์ชัน: " .. _VERSION)

local major, minor = _VERSION:match("Lua (%d+)%.(%d+)")
major = tonumber(major)
minor = tonumber(minor)

print("Major version: " .. major)
print("Minor version: " .. minor)
print()

print("=== Features ที่รองรับ ===")

if major > 5 or (major == 5 and minor >= 1) then
    print("✓ Coroutines")
end
if major > 5 or (major == 5 and minor >= 2) then
    print("✓ Ephemeron tables")
    print("✓ Emergency garbage collector")
end
if major > 5 or (major == 5 and minor >= 3) then
    print("✓ Bitwise operators (&, |, ~, <<, >>)")
    print("✓ Integer subtype")
end
if major > 5 or (major == 5 and minor >= 4) then
    print("✓ Generational garbage collector")
    print("✓ <const> and <close> attributes")
    print("✓ Integer/float distinction")
end
```

### แบบฝึกหัดที่ 4: Pattern สี่เหลี่ยม

เขียนโปรแกรมที่รับขนาดและแสดงรูปสี่เหลี่ยมด้วย * 

```lua
-- แบบฝึกหัดที่ 4 - เฉลย
io.write("ใส่ความกว้าง: ")
local width = tonumber(io.read())

io.write("ใส่ความสูง: ")
local height = tonumber(io.read())

print()
print("=== สี่เหลี่ยมผืนผ้า " .. width .. "x" .. height .. " ===")

-- แบบ solid
print("แบบเต็ม:")
for i = 1, height do
    print(string.rep("*", width))
end

print()

-- แบบ hollow
print("แบบกลวง:")
for i = 1, height do
    if i == 1 or i == height then
        print(string.rep("*", width))
    else
        print("*" .. string.rep(" ", width - 2) .. "*")
    end
end

print()
print("พื้นที่: " .. (width * height) .. " ตารางหน่วย")
print("เส้นรอบรูป: " .. (2 * (width + height)) .. " หน่วย")
```

### แบบฝึกหัดที่ 5: แปลงเลขฐาน

เขียนโปรแกรมที่แสดงตัวเลข 1-20 ในฐาน 2, 8, 10, 16

```lua
-- แบบฝึกหัดที่ 5 - เฉลย
print("=== ตารางแปลงเลขฐาน ===")
print(string.format("%-6s %-12s %-8s %-6s", "DEC", "BIN", "OCT", "HEX"))
print(string.rep("-", 35))

local function toBinary(n)
    if n == 0 then return "0" end
    local result = ""
    while n > 0 do
        result = tostring(n % 2) .. result
        n = math.floor(n / 2)
    end
    return result
end

for i = 1, 20 do
    print(string.format("%-6d %-12s %-8o %-6X", 
        i, toBinary(i), i, i))
end
```

---

## สรุปบทที่ 1

ในบทนี้เราได้เรียนรู้:

1. **ประวัติ Lua** - สร้างที่ PUC-Rio ในปี 1993 โดยทีม 3 คน
2. **ข้อดีของ Lua** - เล็ก, เร็ว, ฝังได้, เรียนรู้ง่าย
3. **การใช้งาน** - Games (WoW, Roblox), Web (OpenResty), Database (Redis), Editor (Neovim)
4. **การติดตั้ง** - Linux, macOS, Windows
5. **Interactive vs Script mode** - REPL และการรันไฟล์
6. **Comments** - `--` และ `--[[ ]]`
7. **ไวยากรณ์พื้นฐาน** - case-sensitive, optional semicolons
8. **print() และ io.write()** - การแสดงผล
9. **tostring() และ tonumber()** - การแปลงชนิดข้อมูล
10. **Multiple assignment** - กำหนดค่าหลายตัวพร้อมกัน
11. **_VERSION** - ตรวจสอบเวอร์ชัน Lua
12. **String concatenation (..)** - การเชื่อมต่อ string

### คำสั่งสำคัญที่ต้องจำ

```lua
-- การแสดงผล
print("Hello")           -- พร้อม newline
io.write("Hello\n")      -- ควบคุม newline เอง

-- การรับ input
local s = io.read()              -- อ่าน string
local n = tonumber(io.read())    -- อ่าน number

-- การแปลงชนิด
tostring(42)      -- "42"
tonumber("42")    -- 42

-- ข้อมูล interpreter
print(_VERSION)   -- Lua 5.4

-- String concatenation
"Hello" .. " " .. "World"  -- "Hello World"
```

**บทถัดไป:** บทที่ 2 - ตัวแปรและชนิดข้อมูล จะลึกลงไปใน 8 ชนิดข้อมูลของ Lua ได้แก่ nil, boolean, number, string, table, function, userdata, thread
