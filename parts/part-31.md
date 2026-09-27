# บทที่ 31: LuaRocks - Package Manager

## บทนำ

LuaRocks คือ package manager อย่างเป็นทางการสำหรับภาษา Lua ซึ่งช่วยให้นักพัฒนาสามารถติดตั้ง แชร์ และจัดการ library ต่างๆ ได้อย่างสะดวก เปรียบเสมือน npm สำหรับ Node.js หรือ pip สำหรับ Python

---

## 31.1 LuaRocks คืออะไร

LuaRocks ทำงานด้วยแนวคิดของ **rock** ซึ่งเป็นหน่วยพื้นฐานของ package ประกอบด้วย:
- **rockspec**: ไฟล์ข้อมูลที่อธิบาย package (ชื่อ เวอร์ชัน dependencies ฯลฯ)
- **rock**: ไฟล์ binary หรือ source ที่ถูก compile แล้ว

### ประโยชน์ของ LuaRocks
- จัดการ dependencies อัตโนมัติ
- รองรับหลาย version ของ Lua
- มี repository กลาง (luarocks.org)
- ติดตั้งได้ทั้งแบบ global และ local

---

## 31.2 การติดตั้ง LuaRocks

### Linux (Ubuntu/Debian)

```bash
# ติดตั้งผ่าน apt
sudo apt-get install luarocks

# หรือติดตั้งจาก source
wget https://luarocks.org/releases/luarocks-3.9.2.tar.gz
tar zxpf luarocks-3.9.2.tar.gz
cd luarocks-3.9.2
./configure --with-lua-include=/usr/include/lua5.4
make
sudo make install
```

### macOS

```bash
# ติดตั้งผ่าน Homebrew
brew install luarocks

# ตรวจสอบ version
luarocks --version
```

### Windows

```batch
:: ดาวน์โหลด installer จาก https://luarocks.org/releases/
:: แล้วรัน LuaRocks_installer.exe

:: ตรวจสอบว่าติดตั้งสำเร็จ
luarocks --version
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ version
luarocks --version

# ดู configuration ปัจจุบัน
luarocks config

# ดู tree ที่ใช้งาน
luarocks config --system-tree
```

---

## 31.3 คำสั่งพื้นฐาน

### luarocks install

```bash
# ติดตั้ง package
luarocks install luasocket

# ติดตั้ง version เฉพาะ
luarocks install luasocket 3.1.0

# ติดตั้งแบบ local (เฉพาะ user ปัจจุบัน)
luarocks install --local luasocket

# ติดตั้งพร้อม dependencies
luarocks install penlight
```

### luarocks remove

```bash
# ลบ package
luarocks remove luasocket

# ลบ version เฉพาะ
luarocks remove luasocket 3.1.0

# ลบแบบ local
luarocks remove --local luasocket
```

### luarocks search

```bash
# ค้นหา package
luarocks search socket

# ค้นหาพร้อม version
luarocks search --all socket

# ค้นหาจาก description
luarocks search --all "http client"
```

### luarocks list

```bash
# แสดง package ทั้งหมดที่ติดตั้ง
luarocks list

# แสดงเฉพาะ outdated packages
luarocks list --outdated

# แสดงแบบ porcelain (สำหรับ scripting)
luarocks list --porcelain
```

### luarocks show

```bash
# แสดงข้อมูล package
luarocks show luasocket

# แสดง dependencies
luarocks show --deps luasocket

# แสดง files ที่ติดตั้ง
luarocks show --files luasocket
```

---

## 31.4 การใช้งาน Rocks ในโค้ด

### ตัวอย่างที่ 1: ใช้ LuaSocket

```lua
-- example_01_socket.lua
-- ต้องติดตั้งก่อน: luarocks install luasocket

local socket = require("socket")

-- ตรวจสอบ version
print("LuaSocket version:", socket._VERSION)

-- สร้าง TCP client
local client = socket.tcp()

-- กำหนด timeout
client:settimeout(5)

-- ลองเชื่อมต่อ (จะ fail ถ้าไม่มี server)
local ok, err = client:connect("www.lua.org", 80)
if ok then
    print("เชื่อมต่อสำเร็จ!")
    client:send("GET / HTTP/1.0\r\nHost: www.lua.org\r\n\r\n")
    local response = client:receive("*l")
    print("Response:", response)
    client:close()
else
    print("ไม่สามารถเชื่อมต่อ:", err)
end
```

### ตัวอย่างที่ 2: ใช้ inspect

```lua
-- example_02_inspect.lua
-- ต้องติดตั้งก่อน: luarocks install inspect

local inspect = require("inspect")

-- inspect ช่วยแสดง table อย่างสวยงาม
local data = {
    name = "Lua Developer",
    age = 25,
    skills = {"Lua", "C", "Python"},
    address = {
        city = "Bangkok",
        country = "Thailand"
    }
}

-- แสดงแบบ formatted
print(inspect(data))

-- แสดงแบบ compact
print(inspect(data, {indent = "  ", newline = " "}))

-- ใช้กับ nested tables
local config = {
    database = {
        host = "localhost",
        port = 5432,
        name = "mydb"
    },
    cache = {
        ttl = 3600,
        max_size = 1000
    }
}

print(inspect(config))
```

### ตัวอย่างที่ 3: ใช้ Penlight

```lua
-- example_03_penlight.lua
-- ต้องติดตั้งก่อน: luarocks install penlight

-- Penlight มีหลาย module
local pl = require("pl.import_into")()

-- หรือ import แบบเฉพาะ module
local tablex = require("pl.tablex")
local stringx = require("pl.stringx")
local path = require("pl.path")

-- ตัวอย่างการใช้ tablex
local t = {1, 2, 3, 4, 5}
print("sum:", tablex.reduce('+', t))    -- 15
print("size:", tablex.size(t))          -- 5

-- copy table แบบ deep
local t2 = tablex.deepcopy(t)
t2[1] = 100
print("ต้นฉบับ:", t[1])   -- 1
print("สำเนา:", t2[1])     -- 100

-- ตัวอย่างการใช้ stringx
local str = "Hello, World! How are you?"
print("split:", stringx.split(str, " "))
print("startswith:", stringx.startswith(str, "Hello"))
print("strip:", stringx.strip("  spaces  "))

-- ตัวอย่างการใช้ path
print("current dir:", path.currentdir())
print("exists:", path.exists("/tmp"))
```

### ตัวอย่างที่ 4: ใช้ Middleclass

```lua
-- example_04_middleclass.lua
-- ต้องติดตั้งก่อน: luarocks install middleclass

local class = require("middleclass")

-- สร้าง base class
local Animal = class("Animal")

function Animal:initialize(name, sound)
    self.name = name
    self.sound = sound
end

function Animal:speak()
    return self.name .. " พูดว่า: " .. self.sound
end

function Animal:__tostring()
    return "Animal(" .. self.name .. ")"
end

-- สร้าง subclass
local Dog = class("Dog", Animal)

function Dog:initialize(name)
    Animal.initialize(self, name, "โฮ่ง!")
    self.tricks = {}
end

function Dog:learnTrick(trick)
    table.insert(self.tricks, trick)
end

function Dog:showTricks()
    if #self.tricks == 0 then
        return self.name .. " ยังไม่รู้ลูกเล่นใดๆ"
    end
    return self.name .. " รู้ลูกเล่น: " .. table.concat(self.tricks, ", ")
end

-- ใช้งาน
local cat = Animal:new("แมวส้ม", "เมี๊ยว!")
local dog = Dog:new("น้องหมา")

print(cat:speak())
print(dog:speak())

dog:learnTrick("นั่ง")
dog:learnTrick("นอน")
dog:learnTrick("กลิ้ง")
print(dog:showTricks())

-- ตรวจสอบ class
print("cat เป็น Animal?", cat:isInstanceOf(Animal))  -- true
print("dog เป็น Dog?", dog:isInstanceOf(Dog))        -- true
print("dog เป็น Animal?", dog:isInstanceOf(Animal))  -- true
print("cat เป็น Dog?", cat:isInstanceOf(Dog))        -- false
```

---

## 31.5 การสร้าง Rockspec File

Rockspec คือไฟล์ที่บอก LuaRocks ว่า package ของคุณทำอะไร

### โครงสร้าง Rockspec พื้นฐาน

```lua
-- mypackage-1.0-1.rockspec
-- ตัวอย่างที่ 5: rockspec พื้นฐาน

package = "mypackage"
version = "1.0-1"

-- ข้อมูล source
source = {
    url = "git+https://github.com/username/mypackage.git",
    tag = "v1.0"
}

-- ข้อมูลทั่วไป
description = {
    summary = "คำอธิบายสั้นๆ ของ package",
    detailed = [[
        คำอธิบายละเอียดของ package
        สามารถเขียนหลายบรรทัดได้
    ]],
    homepage = "https://github.com/username/mypackage",
    license = "MIT"
}

-- Dependencies
dependencies = {
    "lua >= 5.1",
    "luasocket >= 3.0",
}

-- การ build
build = {
    type = "builtin",
    modules = {
        mypackage = "src/mypackage.lua",
        ["mypackage.utils"] = "src/utils.lua",
    }
}
```

### Rockspec สำหรับ C extension

```lua
-- myext-1.0-1.rockspec
-- ตัวอย่างที่ 6: rockspec สำหรับ C extension

package = "myext"
version = "1.0-1"

source = {
    url = "https://example.com/myext-1.0.tar.gz",
    md5 = "abc123...",
}

description = {
    summary = "C extension สำหรับ Lua",
    license = "MIT"
}

dependencies = {
    "lua >= 5.4"
}

build = {
    type = "builtin",
    modules = {
        myext = {
            sources = {"src/myext.c"},
            libraries = {"m"},  -- link กับ libm
            incdirs = {"include/"},
            libdirs = {"/usr/local/lib"},
        }
    }
}
```

---

## 31.6 Dependency Management

### ตัวอย่างที่ 7: จัดการ Dependencies

```lua
-- dependency_demo.lua

-- LuaRocks จัดการ dependencies อัตโนมัติ
-- เมื่อติดตั้ง package ที่มี dependencies
-- LuaRocks จะติดตั้ง dependencies ให้ด้วย

-- ตัวอย่าง: penlight ต้องการ luafilesystem
-- luarocks install penlight
-- -> ติดตั้ง luafilesystem อัตโนมัติ

-- ตรวจสอบ dependencies ของ package
-- luarocks show --deps penlight

-- ใช้งาน package ที่มี dependencies
local lfs = require("lfs")  -- luafilesystem
local pl_path = require("pl.path")  -- penlight path

-- ทำงานร่วมกัน
local current = lfs.currentdir()
print("Current directory:", current)

-- วนดูไฟล์ใน directory
for file in lfs.dir(current) do
    if file ~= "." and file ~= ".." then
        local fullpath = current .. "/" .. file
        local attr = lfs.attributes(fullpath)
        if attr then
            print(string.format("%-30s %s", file, attr.mode))
        end
    end
end
```

### ตัวอย่างที่ 8: Version Pinning

```bash
# ปักหมุด version เฉพาะ
luarocks install luasocket 3.0.0-2

# ติดตั้ง version ล่าสุดของ major version 3
luarocks install luasocket "~> 3"

# ติดตั้ง version อย่างน้อย 3.0
luarocks install luasocket ">= 3.0"
```

```lua
-- ตัวอย่างที่ 9: ตรวจสอบ version ใน code
local socket = require("socket")

-- ตรวจสอบ version
local version = socket._VERSION
print("LuaSocket version:", version)

-- ตรวจสอบว่า version เพียงพอ
local function checkVersion(current, required)
    local cur_major, cur_minor = current:match("(%d+)%.(%d+)")
    local req_major, req_minor = required:match("(%d+)%.(%d+)")
    
    cur_major, cur_minor = tonumber(cur_major), tonumber(cur_minor)
    req_major, req_minor = tonumber(req_major), tonumber(req_minor)
    
    if cur_major > req_major then return true end
    if cur_major == req_major and cur_minor >= req_minor then return true end
    return false
end

if checkVersion(version, "3.0") then
    print("Version เพียงพอ!")
else
    print("Version ต่ำเกินไป กรุณา upgrade")
end
```

---

## 31.7 Local vs System Installation

### ตัวอย่างที่ 10: Local Installation

```bash
# ติดตั้งแบบ local (เฉพาะ user ปัจจุบัน)
luarocks install --local luasocket

# ดู local tree
luarocks list --local

# ใช้ local rock ใน Lua
# ต้องเพิ่ม path ก่อน
eval $(luarocks path --local)
lua myscript.lua
```

```lua
-- ตัวอย่างที่ 11: การ setup path สำหรับ local rocks

-- เพิ่ม local rocks path ด้วยตนเอง
local home = os.getenv("HOME")
if home then
    -- เพิ่ม Lua path สำหรับ local rocks
    package.path = home .. "/.luarocks/share/lua/5.4/?.lua;" ..
                   home .. "/.luarocks/share/lua/5.4/?/init.lua;" ..
                   package.path
    
    -- เพิ่ม C library path
    package.cpath = home .. "/.luarocks/lib/lua/5.4/?.so;" ..
                    package.cpath
end

-- ตอนนี้สามารถ require local rocks ได้
local ok, socket = pcall(require, "socket")
if ok then
    print("LuaSocket version:", socket._VERSION)
else
    print("ไม่พบ LuaSocket:", socket)
end
```

### ตัวอย่างที่ 12: System-wide Installation

```bash
# ติดตั้งแบบ system-wide (ต้องการ sudo)
sudo luarocks install luasocket

# ดู system tree
luarocks list --system-tree

# ดู path ทั้งหมด
luarocks path
```

---

## 31.8 LuaRocks Tree

LuaRocks tree คือตำแหน่งที่เก็บ packages

### ตัวอย่างที่ 13: จัดการ Multiple Trees

```bash
# สร้าง project-specific tree
luarocks init

# ดูโครงสร้าง
ls -la

# ติดตั้ง package เข้า project tree
luarocks install --tree ./lua_modules luasocket

# รัน script ด้วย project tree
luarocks exec lua myscript.lua
```

```lua
-- ตัวอย่างที่ 14: ตรวจสอบ package path

-- แสดง package path ปัจจุบัน
print("Lua Path:")
for path in package.path:gmatch("[^;]+") do
    print("  " .. path)
end

print("\nC Path:")
for path in package.cpath:gmatch("[^;]+") do
    print("  " .. path)
end
```

---

## 31.9 Popular Packages

### LuaSocket - Network Library

```lua
-- ตัวอย่างที่ 15: LuaSocket - HTTP request พื้นฐาน
-- luarocks install luasocket

local http = require("socket.http")
local ltn12 = require("ltn12")

-- GET request
local response_body = {}
local ok, status, headers = http.request{
    url = "http://httpbin.org/get",
    sink = ltn12.sink.table(response_body)
}

if ok then
    print("Status:", status)
    print("Content-Type:", headers["content-type"])
    print("Body:", table.concat(response_body))
else
    print("Error:", status)
end
```

```lua
-- ตัวอย่างที่ 16: LuaSocket - UDP server
local socket = require("socket")

-- สร้าง UDP server
local server = socket.udp()
server:setsockname("localhost", 12345)
server:settimeout(5)

print("UDP Server รอรับข้อมูลที่ port 12345...")

-- รับข้อมูล
local data, ip, port = server:receivefrom()
if data then
    print(string.format("ได้รับจาก %s:%d: %s", ip, port, data))
    -- ส่งกลับ
    server:sendto("Echo: " .. data, ip, port)
else
    print("Timeout หรือ error")
end

server:close()
```

### Inspect - Debug Tool

```lua
-- ตัวอย่างที่ 17: inspect - debug complex tables
-- luarocks install inspect

local inspect = require("inspect")

-- ใช้กับ circular references
local a = {}
local b = {parent = a}
a.child = b

-- inspect จัดการ circular reference ได้
print(inspect(a))

-- กำหนด options
local options = {
    depth = 3,        -- ความลึกสูงสุด
    indent = "    ",  -- indentation
    newline = "\n",   -- newline character
}

local data = {
    level1 = {
        level2 = {
            level3 = {
                level4 = "deep value"
            }
        }
    }
}

print(inspect(data, options))
```

### LuaFileSystem

```lua
-- ตัวอย่างที่ 18: LuaFileSystem - จัดการไฟล์
-- luarocks install luafilesystem

local lfs = require("lfs")

-- สร้าง directory
local ok, err = lfs.mkdir("/tmp/lua_test")
if ok then
    print("สร้าง directory สำเร็จ")
else
    print("Error:", err)
end

-- เปลี่ยน directory
lfs.chdir("/tmp/lua_test")
print("Current dir:", lfs.currentdir())

-- ดึง attributes ของไฟล์
local attr = lfs.attributes("/tmp")
if attr then
    print("Mode:", attr.mode)
    print("Size:", attr.size)
    print("Modified:", os.date("%Y-%m-%d %H:%M:%S", attr.modification))
end

-- วนดู directory
for file in lfs.dir("/tmp") do
    if file ~= "." and file ~= ".." then
        local fullpath = "/tmp/" .. file
        local a = lfs.attributes(fullpath)
        if a then
            print(string.format("%-20s %-10s %d bytes", 
                file, a.mode, a.size or 0))
        end
    end
end
```

### Busted - Testing Framework

```lua
-- ตัวอย่างที่ 19: busted - ตัวอย่าง test file
-- luarocks install busted

-- test_basic.lua
describe("ทดสอบ math functions", function()
    it("ควร 2+2 เท่ากับ 4", function()
        assert.are.equal(4, 2 + 2)
    end)
    
    it("ควร sqrt(9) เท่ากับ 3", function()
        assert.are.equal(3, math.sqrt(9))
    end)
end)
```

### LuaCheck - Linting Tool

```bash
# ตัวอย่างที่ 20: ใช้ luacheck สำหรับ lint
# luarocks install luacheck

# ตรวจสอบไฟล์เดียว
luacheck myfile.lua

# ตรวจสอบหลายไฟล์
luacheck src/*.lua

# ตรวจสอบพร้อม config
luacheck --config .luacheckrc myfile.lua
```

```lua
-- ตัวอย่างที่ 21: .luacheckrc configuration
-- สร้างไฟล์ .luacheckrc ใน project root

return {
    -- global ที่อนุญาต
    globals = {
        "love",      -- Love2D
        "vim",       -- Neovim
        "_G",        -- global table
    },
    
    -- ไม่ต้องการ warning เรื่องนี้
    ignore = {
        "611",  -- line is too long
        "614",  -- trailing whitespace
    },
    
    -- max line length
    max_line_length = 120,
    
    -- Lua version
    std = "lua54",
}
```

---

## 31.10 การเขียน Rock ของตัวเอง

### ตัวอย่างที่ 22: โครงสร้าง Package

```
mylib/
├── mylib-1.0-1.rockspec
├── src/
│   ├── mylib.lua
│   └── mylib/
│       ├── utils.lua
│       └── math.lua
└── spec/
    └── mylib_spec.lua
```

### ตัวอย่างที่ 23: เขียน Library

```lua
-- src/mylib.lua
-- Library หลัก

local mylib = {}
mylib._VERSION = "1.0.0"

-- ดึง submodules
mylib.utils = require("mylib.utils")
mylib.math = require("mylib.math")

-- ฟังก์ชันหลัก
function mylib.greet(name)
    return "สวัสดี, " .. (name or "ผู้ใช้") .. "!"
end

function mylib.version()
    return mylib._VERSION
end

return mylib
```

```lua
-- src/mylib/utils.lua
-- Utility functions

local utils = {}

-- แยก string ด้วย separator
function utils.split(str, sep)
    local result = {}
    local pattern = "([^" .. sep .. "]+)"
    for match in str:gmatch(pattern) do
        table.insert(result, match)
    end
    return result
end

-- รวม table เป็น string
function utils.join(t, sep)
    return table.concat(t, sep or "")
end

-- ตรวจสอบว่า table มี element
function utils.contains(t, value)
    for _, v in ipairs(t) do
        if v == value then
            return true
        end
    end
    return false
end

-- copy table แบบ shallow
function utils.copy(t)
    local new = {}
    for k, v in pairs(t) do
        new[k] = v
    end
    return new
end

return utils
```

```lua
-- src/mylib/math.lua
-- Math functions

local math_utils = {}

-- factorial
function math_utils.factorial(n)
    if n < 0 then error("factorial ต้องการ n >= 0") end
    if n == 0 then return 1 end
    local result = 1
    for i = 2, n do
        result = result * i
    end
    return result
end

-- fibonacci
function math_utils.fibonacci(n)
    if n <= 1 then return n end
    local a, b = 0, 1
    for i = 2, n do
        a, b = b, a + b
    end
    return b
end

-- ตรวจสอบจำนวนเฉพาะ
function math_utils.isPrime(n)
    if n < 2 then return false end
    if n == 2 then return true end
    if n % 2 == 0 then return false end
    for i = 3, math.sqrt(n), 2 do
        if n % i == 0 then return false end
    end
    return true
end

-- Greatest Common Divisor
function math_utils.gcd(a, b)
    while b ~= 0 do
        a, b = b, a % b
    end
    return a
end

-- Least Common Multiple
function math_utils.lcm(a, b)
    return (a * b) / math_utils.gcd(a, b)
end

return math_utils
```

### ตัวอย่างที่ 24: Rockspec สมบูรณ์

```lua
-- mylib-1.0-1.rockspec
package = "mylib"
version = "1.0-1"

source = {
    url = "git+https://github.com/myuser/mylib.git",
    tag = "v1.0"
}

description = {
    summary = "ไลบรารี Lua สำหรับงานทั่วไป",
    detailed = [[
        mylib เป็น collection ของ utility functions
        สำหรับงานประจำวันของนักพัฒนา Lua
        รวมถึง string utilities, math functions
        และ helper functions ต่างๆ
    ]],
    homepage = "https://github.com/myuser/mylib",
    license = "MIT"
}

dependencies = {
    "lua >= 5.1, < 5.5",
    -- ไม่มี external dependencies
}

build = {
    type = "builtin",
    modules = {
        ["mylib"] = "src/mylib.lua",
        ["mylib.utils"] = "src/mylib/utils.lua",
        ["mylib.math"] = "src/mylib/math.lua",
    },
    copy_directories = {
        "doc",
        "spec",
    }
}
```

---

## 31.11 การ Publish ไปยัง LuaRocks

### ตัวอย่างที่ 25: ขั้นตอนการ Publish

```bash
# 1. สร้าง account บน luarocks.org

# 2. ตั้งค่า API key
luarocks config --local upload.server "https://luarocks.org"
luarocks config --local upload.api_key "YOUR_API_KEY"

# 3. ตรวจสอบ rockspec
luarocks lint mylib-1.0-1.rockspec

# 4. build rock
luarocks pack mylib-1.0-1.rockspec

# 5. upload ไปยัง luarocks.org
luarocks upload mylib-1.0-1.rockspec

# 6. ตรวจสอบว่า upload สำเร็จ
luarocks search mylib
```

### ตัวอย่างที่ 26: Build และทดสอบก่อน Publish

```bash
# สร้าง source rock
luarocks pack mylib-1.0-1.rockspec

# ทดสอบติดตั้งจาก local
luarocks install mylib-1.0-1.src.rock

# ทดสอบว่า module load ได้
lua -e "local m = require('mylib'); print(m.version())"

# ทดสอบ uninstall
luarocks remove mylib
```

---

## 31.12 LuaRocks กับ Project Management

### ตัวอย่างที่ 27: การใช้ luarocks init

```bash
# เริ่มต้น project ใหม่
mkdir myproject
cd myproject
luarocks init

# โครงสร้างที่ได้
# myproject/
# ├── .luarocks/
# │   └── config.lua
# ├── lua_modules/
# ├── luarocks.lock    (lock file)
# └── myproject-dev-1.rockspec
```

```lua
-- ตัวอย่างที่ 28: luarocks.lock - Lock File
-- lock file ใช้สำหรับ reproducible builds

-- เนื้อหา luarocks.lock
{
   ["luarocks-build-builtin"] = {
      server = "https://luarocks.org",
      version = "0.6.0-1",
      sha256 = "abc123...",
   },
   ["luasocket"] = {
      server = "https://luarocks.org",
      version = "3.1.0-1",
      sha256 = "def456...",
   },
}
```

### ตัวอย่างที่ 29: Makefile สำหรับ Project

```makefile
# Makefile
LUAROCKS = luarocks
LUA = lua

.PHONY: install test lint clean

install:
	$(LUAROCKS) install --only-deps myproject-dev-1.rockspec

test:
	$(LUAROCKS) exec busted spec/

lint:
	$(LUAROCKS) exec luacheck src/

clean:
	rm -rf lua_modules/
```

---

## 31.13 Advanced Topics

### ตัวอย่างที่ 30: Custom Repository

```bash
# เพิ่ม repository ส่วนตัว
luarocks config --add-server https://my-private-rocks.example.com

# ดู servers ทั้งหมด
luarocks config rock_servers

# ติดตั้งจาก repository เฉพาะ
luarocks install --server https://my-private-rocks.example.com mypackage
```

### ตัวอย่างที่ 31: Namespace ใน Package

```lua
-- ตัวอย่างที่ 31: จัดการ package ที่มีชื่อเดียวกัน

-- ใช้ namespace เพื่อแยก packages
-- ติดตั้ง: luarocks install organization/package

-- หรือ require ด้วย full path
local mymod = require("company.division.module")

-- โครงสร้างไฟล์
-- company/
-- └── division/
--     └── module.lua
```

### ตัวอย่างที่ 32: Preloading Modules

```lua
-- ตัวอย่างที่ 32: preload module สำหรับ testing

-- บางครั้งต้องการ mock module สำหรับ test
local original_socket = package.loaded["socket"]

-- สร้าง mock
package.loaded["socket"] = {
    tcp = function()
        return {
            connect = function() return true end,
            send = function() return 100 end,
            receive = function() return "mock data" end,
            close = function() end,
            settimeout = function() end,
        }
    end,
    _VERSION = "3.0-mock"
}

-- test code ที่ใช้ socket
local mymodule = require("mymodule")  -- จะได้รับ mock socket
-- ... run tests ...

-- restore original
package.loaded["socket"] = original_socket
```

### ตัวอย่างที่ 33: LuaRocks ใน CI/CD

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        lua-version: ['5.1', '5.2', '5.3', '5.4']
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Lua
      uses: leafo/gh-actions-lua@v9
      with:
        luaVersion: ${{ matrix.lua-version }}
    
    - name: Setup LuaRocks
      uses: leafo/gh-actions-luarocks@v4
    
    - name: Install Dependencies
      run: luarocks install --only-deps mypackage-dev-1.rockspec
    
    - name: Run Tests
      run: luarocks exec busted spec/
    
    - name: Run Lint
      run: luarocks exec luacheck src/
```

### ตัวอย่างที่ 34: การใช้ Scoped Installation

```lua
-- ตัวอย่างที่ 34: ใช้ luarocks_modules สำหรับ project isolation

-- เพิ่ม project-local modules เข้า path
local function setupProjectPath()
    local projectRoot = debug.getinfo(1, "S").source:match("^@(.+)/")
    if not projectRoot then
        projectRoot = "."
    end
    
    -- เพิ่ม lua_modules path
    package.path = projectRoot .. "/lua_modules/share/lua/5.4/?.lua;" ..
                   projectRoot .. "/lua_modules/share/lua/5.4/?/init.lua;" ..
                   package.path
    
    package.cpath = projectRoot .. "/lua_modules/lib/lua/5.4/?.so;" ..
                    package.cpath
end

setupProjectPath()

-- ตอนนี้สามารถ require packages ใน lua_modules ได้
```

### ตัวอย่างที่ 35: อัปเดต Packages

```bash
# ดู packages ที่มีเวอร์ชันใหม่
luarocks list --outdated

# อัปเดต package เดียว
luarocks install luasocket  # ติดตั้ง version ล่าสุด

# อัปเดตทุก packages
for pkg in $(luarocks list --outdated --porcelain | awk '{print $1}'); do
    luarocks install $pkg
done

# ตรวจสอบหลังอัปเดต
luarocks list
```

---

## 31.14 Best Practices

### ตัวอย่างที่ 36: ตรวจสอบ Package ก่อนใช้

```lua
-- ตัวอย่างที่ 36: safe require pattern

local function safeRequire(moduleName, alternatives)
    -- ลอง require module หลัก
    local ok, mod = pcall(require, moduleName)
    if ok then
        return mod
    end
    
    print(string.format("Warning: ไม่พบ '%s': %s", moduleName, mod))
    
    -- ลอง alternatives
    if alternatives then
        for _, alt in ipairs(alternatives) do
            ok, mod = pcall(require, alt)
            if ok then
                print("ใช้ alternative:", alt)
                return mod
            end
        end
    end
    
    return nil, "ไม่พบ module ใดๆ"
end

-- ใช้งาน
local json = safeRequire("cjson", {"dkjson", "json"})
if json then
    print(json.encode({name = "Lua", version = "5.4"}))
else
    print("ไม่สามารถ encode JSON ได้")
end
```

### ตัวอย่างที่ 37: Package Verification

```lua
-- ตัวอย่างที่ 37: ตรวจสอบ package integrity

local function verifyPackage(name, requiredVersion)
    local ok, pkg = pcall(require, name)
    if not ok then
        return false, "ไม่พบ package: " .. name
    end
    
    -- ตรวจสอบ version ถ้ามี
    local version = pkg._VERSION or pkg.version or pkg.VERSION
    if not version then
        return true, "ติดตั้งแล้ว (ไม่รู้ version)"
    end
    
    if requiredVersion then
        -- เปรียบเทียบ version อย่างง่าย
        if version < requiredVersion then
            return false, string.format(
                "Version %s ต่ำกว่าที่ต้องการ %s", 
                version, requiredVersion
            )
        end
    end
    
    return true, "OK (version: " .. tostring(version) .. ")"
end

-- ตรวจสอบ packages ที่จำเป็น
local required = {
    {"socket", "3.0"},
    {"inspect", nil},
    {"lfs", "1.8"},
}

print("ตรวจสอบ packages:")
for _, pkg in ipairs(required) do
    local ok, msg = verifyPackage(pkg[1], pkg[2])
    print(string.format("  %-15s %s %s", 
        pkg[1], 
        ok and "✓" or "✗", 
        msg))
end
```

---

## สรุปบทที่ 31

LuaRocks เป็นเครื่องมือที่จำเป็นสำหรับการพัฒนา Lua สมัยใหม่:

| คำสั่ง | การใช้งาน |
|--------|----------|
| `luarocks install <pkg>` | ติดตั้ง package |
| `luarocks remove <pkg>` | ลบ package |
| `luarocks search <query>` | ค้นหา package |
| `luarocks list` | แสดง packages ที่ติดตั้ง |
| `luarocks show <pkg>` | แสดงข้อมูล package |
| `luarocks pack <rockspec>` | สร้าง rock file |
| `luarocks upload <rockspec>` | เผยแพร่ไปยัง LuaRocks |
| `luarocks init` | เริ่มต้น project |

### Package ยอดนิยม

| Package | การใช้งาน |
|---------|----------|
| luasocket | Network programming |
| inspect | Debug tables |
| penlight | Utility library |
| middleclass | OOP framework |
| busted | Testing framework |
| luacheck | Linting tool |
| luafilesystem | File system operations |
| dkjson / cjson | JSON encoding/decoding |

---

*บทต่อไป: บทที่ 32 - Testing ด้วย Busted*
