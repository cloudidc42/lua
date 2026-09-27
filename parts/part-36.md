# บทที่ 36: Bit Operations (การดำเนินการกับบิต)

## บทนำ

Lua 5.3 ได้เพิ่ม bitwise operators เข้ามาใน language โดยตรง ทำให้ไม่จำเป็นต้องใช้ library ภายนอกอีกต่อไป การดำเนินการกับบิตเป็นพื้นฐานสำคัญในการเขียนโปรแกรม low-level, การเข้ารหัส, network protocols, และการจัดการ flags

## 36.1 Bitwise Operators ใน Lua 5.3+

Lua 5.3 รองรับ bitwise operators ดังนี้:
- `&` - AND
- `|` - OR  
- `~` - XOR (binary) หรือ NOT (unary)
- `<<` - Left shift
- `>>` - Right shift

```lua
-- ตัวอย่างที่ 1: Bitwise operators พื้นฐาน
print("=== Bitwise Operators พื้นฐาน ===")

local a = 0b1010  -- 10 ในเลขฐาน 2
local b = 0b1100  -- 12 ในเลขฐาน 2

print(string.format("a = %d (binary: %s)", a, string.format("%04d", tonumber(tostring(a), 10) and 1010 or 0)))
print(string.format("b = %d", b))
print(string.format("a & b  = %d", a & b))   -- AND: 1000 = 8
print(string.format("a | b  = %d", a | b))   -- OR:  1110 = 14
print(string.format("a ~ b  = %d", a ~ b))   -- XOR: 0110 = 6
print(string.format("~a     = %d", ~a))       -- NOT: -(a+1)
print(string.format("a << 1 = %d", a << 1))  -- Left shift: 10100 = 20
print(string.format("a >> 1 = %d", a >> 1))  -- Right shift: 0101 = 5
```

```lua
-- ตัวอย่างที่ 2: ฟังก์ชันแสดงเลขฐานสอง
function toBinary(n, bits)
    bits = bits or 8
    local result = ""
    for i = bits - 1, 0, -1 do
        result = result .. ((n >> i) & 1)
        if i > 0 and i % 4 == 0 then
            result = result .. "_"
        end
    end
    return result
end

print("=== แสดงเลขฐานสอง ===")
print(string.format("10 = %s", toBinary(10, 8)))
print(string.format("12 = %s", toBinary(12, 8)))
print(string.format("255 = %s", toBinary(255, 8)))
print(string.format("256 = %s", toBinary(256, 16)))
```

```lua
-- ตัวอย่างที่ 3: แสดงเลขฐาน 16
function toHex(n, digits)
    digits = digits or 2
    return string.format("0x%0" .. digits .. "X", n)
end

print("=== แสดงเลขฐานสิบหก ===")
print(string.format("255  = %s", toHex(255, 2)))
print(string.format("256  = %s", toHex(256, 4)))
print(string.format("1024 = %s", toHex(1024, 4)))
print(string.format("65535 = %s", toHex(65535, 4)))
```

## 36.2 Integer Representation

ใน Lua 5.3+ integers เป็น 64-bit signed integers

```lua
-- ตัวอย่างที่ 4: ขอบเขตของ integer
print("=== Integer Boundaries ===")
print(string.format("MAX_INTEGER = %d", math.maxinteger))
print(string.format("MIN_INTEGER = %d", math.mininteger))
print(string.format("MAX_INTEGER (hex) = %s", toHex(math.maxinteger, 16)))
print(string.format("Type of 1 = %s", math.type(1)))
print(string.format("Type of 1.0 = %s", math.type(1.0)))
```

```lua
-- ตัวอย่างที่ 5: Integer overflow (wrap around)
print("=== Integer Overflow ===")
local max = math.maxinteger
print(string.format("maxinteger     = %d", max))
print(string.format("maxinteger + 1 = %d", max + 1))  -- overflow!
print(string.format("mininteger - 1 = %d", math.mininteger - 1))  -- underflow!
```

```lua
-- ตัวอย่างที่ 6: Two's complement
print("=== Two's Complement ===")
local function showTwosComplement(n)
    print(string.format("  %4d = %s", n, toBinary(n, 8)))
end

showTwosComplement(0)
showTwosComplement(1)
showTwosComplement(127)
showTwosComplement(-1)    -- 11111111
showTwosComplement(-128)  -- 10000000
```

## 36.3 Bit Manipulation Techniques

```lua
-- ตัวอย่างที่ 7: Setting a bit (เซ็ตบิต)
function setBit(value, position)
    return value | (1 << position)
end

print("=== Setting Bits ===")
local n = 0b00000000
print(string.format("Original: %s (%d)", toBinary(n), n))
n = setBit(n, 0)
print(string.format("Set bit 0: %s (%d)", toBinary(n), n))
n = setBit(n, 3)
print(string.format("Set bit 3: %s (%d)", toBinary(n), n))
n = setBit(n, 7)
print(string.format("Set bit 7: %s (%d)", toBinary(n), n))
```

```lua
-- ตัวอย่างที่ 8: Clearing a bit (เคลียร์บิต)
function clearBit(value, position)
    return value & ~(1 << position)
end

print("=== Clearing Bits ===")
local n = 0b11111111
print(string.format("Original: %s (%d)", toBinary(n), n))
n = clearBit(n, 0)
print(string.format("Clear bit 0: %s (%d)", toBinary(n), n))
n = clearBit(n, 4)
print(string.format("Clear bit 4: %s (%d)", toBinary(n), n))
n = clearBit(n, 7)
print(string.format("Clear bit 7: %s (%d)", toBinary(n), n))
```

```lua
-- ตัวอย่างที่ 9: Toggling a bit (สลับบิต)
function toggleBit(value, position)
    return value ~ (1 << position)
end

print("=== Toggling Bits ===")
local n = 0b10101010
print(string.format("Original: %s (%d)", toBinary(n), n))
n = toggleBit(n, 0)
print(string.format("Toggle bit 0: %s (%d)", toBinary(n), n))
n = toggleBit(n, 1)
print(string.format("Toggle bit 1: %s (%d)", toBinary(n), n))
n = toggleBit(n, 7)
print(string.format("Toggle bit 7: %s (%d)", toBinary(n), n))
```

```lua
-- ตัวอย่างที่ 10: Checking a bit (ตรวจสอบบิต)
function checkBit(value, position)
    return (value >> position) & 1 == 1
end

function getBit(value, position)
    return (value >> position) & 1
end

print("=== Checking Bits ===")
local n = 0b10110101
print(string.format("Value: %s (%d)", toBinary(n), n))
for i = 7, 0, -1 do
    print(string.format("  Bit %d: %d (%s)", i, getBit(n, i),
        checkBit(n, i) and "set" or "clear"))
end
```

## 36.4 Bit Masks

```lua
-- ตัวอย่างที่ 11: Bit masks พื้นฐาน
print("=== Bit Masks ===")

-- สร้าง mask สำหรับ n บิต
function createMask(bits)
    return (1 << bits) - 1
end

-- ดึงค่า n บิตจากตำแหน่ง position
function extractBits(value, position, bits)
    local mask = createMask(bits)
    return (value >> position) & mask
end

-- ใส่ค่าลงใน n บิตที่ตำแหน่ง position
function insertBits(target, value, position, bits)
    local mask = createMask(bits) << position
    return (target & ~mask) | ((value << position) & mask)
end

local data = 0b11001010
print(string.format("Data: %s", toBinary(data)))
print(string.format("Extract bits 2-4: %d", extractBits(data, 2, 3)))
print(string.format("Extract bits 4-7: %d", extractBits(data, 4, 4)))

local result = insertBits(data, 0b111, 0, 3)
print(string.format("Insert 111 at pos 0-2: %s", toBinary(result)))
```

```lua
-- ตัวอย่างที่ 12: Nibble operations
print("=== Nibble Operations ===")

function highNibble(byte)
    return (byte >> 4) & 0x0F
end

function lowNibble(byte)
    return byte & 0x0F
end

function makeByteFromNibbles(high, low)
    return ((high & 0x0F) << 4) | (low & 0x0F)
end

local byte = 0xAB
print(string.format("Byte: 0x%02X", byte))
print(string.format("High nibble: 0x%X (%d)", highNibble(byte), highNibble(byte)))
print(string.format("Low nibble:  0x%X (%d)", lowNibble(byte), lowNibble(byte)))
local reconstructed = makeByteFromNibbles(highNibble(byte), lowNibble(byte))
print(string.format("Reconstructed: 0x%02X", reconstructed))
```

## 36.5 Flags with Bits

```lua
-- ตัวอย่างที่ 13: Permission flags
print("=== Permission Flags ===")

local Permissions = {
    READ    = 1 << 0,  -- 1
    WRITE   = 1 << 1,  -- 2
    EXECUTE = 1 << 2,  -- 4
    DELETE  = 1 << 3,  -- 8
    ADMIN   = 1 << 4,  -- 16
}

function createPermissions(...)
    local perm = 0
    for _, p in ipairs({...}) do
        perm = perm | p
    end
    return perm
end

function hasPermission(userPerm, required)
    return (userPerm & required) == required
end

function addPermission(userPerm, perm)
    return userPerm | perm
end

function removePermission(userPerm, perm)
    return userPerm & ~perm
end

function describePermissions(perm)
    local perms = {}
    if hasPermission(perm, Permissions.READ)    then table.insert(perms, "READ") end
    if hasPermission(perm, Permissions.WRITE)   then table.insert(perms, "WRITE") end
    if hasPermission(perm, Permissions.EXECUTE) then table.insert(perms, "EXECUTE") end
    if hasPermission(perm, Permissions.DELETE)  then table.insert(perms, "DELETE") end
    if hasPermission(perm, Permissions.ADMIN)   then table.insert(perms, "ADMIN") end
    return #perms > 0 and table.concat(perms, " | ") or "NONE"
end

-- ทดสอบ
local userPerm = createPermissions(Permissions.READ, Permissions.WRITE)
print(string.format("User permissions: %s (%d)", describePermissions(userPerm), userPerm))
print(string.format("Has READ: %s", tostring(hasPermission(userPerm, Permissions.READ))))
print(string.format("Has EXECUTE: %s", tostring(hasPermission(userPerm, Permissions.EXECUTE))))

userPerm = addPermission(userPerm, Permissions.EXECUTE)
print(string.format("After adding EXECUTE: %s", describePermissions(userPerm)))

userPerm = removePermission(userPerm, Permissions.WRITE)
print(string.format("After removing WRITE: %s", describePermissions(userPerm)))
```

```lua
-- ตัวอย่างที่ 14: Game entity flags
print("=== Game Entity Flags ===")

local EntityFlags = {
    VISIBLE    = 1 << 0,
    COLLIDABLE = 1 << 1,
    ACTIVE     = 1 << 2,
    INVINCIBLE = 1 << 3,
    FLYING     = 1 << 4,
    BURNING    = 1 << 5,
    FROZEN     = 1 << 6,
    POISONED   = 1 << 7,
}

local Entity = {}
Entity.__index = Entity

function Entity.new(name, flags)
    return setmetatable({
        name = name,
        flags = flags or (EntityFlags.VISIBLE | EntityFlags.ACTIVE | EntityFlags.COLLIDABLE)
    }, Entity)
end

function Entity:hasFlag(flag)
    return (self.flags & flag) ~= 0
end

function Entity:setFlag(flag)
    self.flags = self.flags | flag
end

function Entity:clearFlag(flag)
    self.flags = self.flags & ~flag
end

function Entity:toggleFlag(flag)
    self.flags = self.flags ~ flag
end

function Entity:describe()
    local parts = {}
    for name, flag in pairs(EntityFlags) do
        if self:hasFlag(flag) then
            table.insert(parts, name)
        end
    end
    table.sort(parts)
    return string.format("%s: [%s]", self.name, table.concat(parts, ", "))
end

local hero = Entity.new("Hero")
print(hero:describe())
hero:setFlag(EntityFlags.FLYING)
hero:setFlag(EntityFlags.INVINCIBLE)
print(hero:describe())
hero:clearFlag(EntityFlags.COLLIDABLE)
print(hero:describe())
hero:setFlag(EntityFlags.BURNING)
print(hero:describe())
```

## 36.6 Bit Fields

```lua
-- ตัวอย่างที่ 15: Bit field structure
print("=== Bit Fields ===")

-- IP header flags: version(4), IHL(4), DSCP(6), ECN(2), total_length(16)
local BitField = {}
BitField.__index = BitField

function BitField.new()
    return setmetatable({ value = 0 }, BitField)
end

function BitField:set(value, startBit, length)
    local mask = ((1 << length) - 1) << startBit
    self.value = (self.value & ~mask) | ((value << startBit) & mask)
    return self
end

function BitField:get(startBit, length)
    local mask = (1 << length) - 1
    return (self.value >> startBit) & mask
end

-- จำลอง IP header (simplified)
local header = BitField.new()
header:set(4, 28, 4)    -- version = 4 (IPv4)
header:set(5, 24, 4)    -- IHL = 5 (20 bytes)
header:set(0, 18, 6)    -- DSCP = 0
header:set(0, 16, 2)    -- ECN = 0

print(string.format("Header value: 0x%08X", header.value))
print(string.format("Version: %d", header:get(28, 4)))
print(string.format("IHL: %d", header:get(24, 4)))
```

```lua
-- ตัวอย่างที่ 16: Color packing (RGBA)
print("=== Color Packing ===")

function packRGBA(r, g, b, a)
    a = a or 255
    return ((r & 0xFF) << 24) | ((g & 0xFF) << 16) | ((b & 0xFF) << 8) | (a & 0xFF)
end

function unpackRGBA(color)
    return {
        r = (color >> 24) & 0xFF,
        g = (color >> 16) & 0xFF,
        b = (color >> 8) & 0xFF,
        a = color & 0xFF
    }
end

function colorToHex(color)
    return string.format("#%08X", color)
end

local red   = packRGBA(255, 0, 0, 255)
local green = packRGBA(0, 255, 0, 255)
local blue  = packRGBA(0, 0, 255, 128)

print(string.format("Red:   %s", colorToHex(red)))
print(string.format("Green: %s", colorToHex(green)))
print(string.format("Blue (50%% alpha): %s", colorToHex(blue)))

local c = unpackRGBA(blue)
print(string.format("Unpacked blue: R=%d G=%d B=%d A=%d", c.r, c.g, c.b, c.a))
```

## 36.7 Pack/Unpack Binary Data

```lua
-- ตัวอย่างที่ 17: String pack/unpack สำหรับ binary data
print("=== Binary Data Pack/Unpack ===")

-- pack integer เป็น bytes
function packUint32BE(n)
    return string.char(
        (n >> 24) & 0xFF,
        (n >> 16) & 0xFF,
        (n >> 8) & 0xFF,
        n & 0xFF
    )
end

function unpackUint32BE(s, offset)
    offset = offset or 1
    local b1, b2, b3, b4 = string.byte(s, offset, offset + 3)
    return (b1 << 24) | (b2 << 16) | (b3 << 8) | b4
end

function packUint16BE(n)
    return string.char((n >> 8) & 0xFF, n & 0xFF)
end

function unpackUint16BE(s, offset)
    offset = offset or 1
    local b1, b2 = string.byte(s, offset, offset + 1)
    return (b1 << 8) | b2
end

local n = 0xDEADBEEF
local packed = packUint32BE(n)
print(string.format("Original: 0x%08X", n))
print(string.format("Packed bytes: %02X %02X %02X %02X",
    string.byte(packed, 1), string.byte(packed, 2),
    string.byte(packed, 3), string.byte(packed, 4)))
print(string.format("Unpacked: 0x%08X", unpackUint32BE(packed)))
```

```lua
-- ตัวอย่างที่ 18: Using string.pack and string.unpack (Lua 5.3+)
print("=== string.pack / string.unpack ===")

-- pack หลายค่าเข้าด้วยกัน
local data = string.pack(">I4I2I1", 0xDEADBEEF, 0x1234, 0xFF)
print(string.format("Packed data length: %d bytes", #data))

local a, b, c = string.unpack(">I4I2I1", data)
print(string.format("Unpacked: 0x%08X, 0x%04X, 0x%02X", a, b, c))

-- pack structure
local point = string.pack("<f f", 3.14, 2.72)
local x, y = string.unpack("<f f", point)
print(string.format("Point: (%.2f, %.2f)", x, y))
```

## 36.8 Binary Protocols

```lua
-- ตัวอย่างที่ 19: Simple binary protocol
print("=== Binary Protocol ===")

local Protocol = {}
Protocol.__index = Protocol

-- Message types
Protocol.MSG_PING    = 0x01
Protocol.MSG_PONG    = 0x02
Protocol.MSG_DATA    = 0x03
Protocol.MSG_ERROR   = 0xFF

function Protocol.encodeMessage(msgType, payload)
    payload = payload or ""
    local header = string.pack(">I1I2", msgType, #payload)
    return header .. payload
end

function Protocol.decodeMessage(data)
    if #data < 3 then return nil, "Message too short" end
    local msgType, payloadLen = string.unpack(">I1I2", data)
    if #data < 3 + payloadLen then
        return nil, "Incomplete payload"
    end
    local payload = string.sub(data, 4, 3 + payloadLen)
    return {
        type = msgType,
        payload = payload,
        length = payloadLen
    }
end

-- ทดสอบ
local ping = Protocol.encodeMessage(Protocol.MSG_PING)
local data = Protocol.encodeMessage(Protocol.MSG_DATA, "Hello, World!")
local error_msg = Protocol.encodeMessage(Protocol.MSG_ERROR, "Bad request")

print(string.format("PING  message: %d bytes", #ping))
print(string.format("DATA  message: %d bytes", #data))
print(string.format("ERROR message: %d bytes", #error_msg))

local decoded = Protocol.decodeMessage(data)
print(string.format("Decoded type: 0x%02X, payload: '%s'", decoded.type, decoded.payload))
```

```lua
-- ตัวอย่างที่ 20: DNS-like message encoding
print("=== DNS-like Encoding ===")

function encodeDNSName(name)
    local result = ""
    for label in name:gmatch("[^.]+") do
        result = result .. string.char(#label) .. label
    end
    result = result .. "\0"  -- terminate
    return result
end

function decodeDNSName(data, offset)
    offset = offset or 1
    local name = ""
    local first = true
    while true do
        local len = string.byte(data, offset)
        if len == nil or len == 0 then
            break
        end
        if not first then name = name .. "." end
        name = name .. string.sub(data, offset + 1, offset + len)
        offset = offset + len + 1
        first = false
    end
    return name, offset + 1
end

local encoded = encodeDNSName("www.example.com")
print(string.format("Encoded length: %d", #encoded))
local decoded, nextOffset = decodeDNSName(encoded)
print(string.format("Decoded: %s", decoded))
```

## 36.9 CRC Calculation

```lua
-- ตัวอย่างที่ 21: CRC-8 calculation
print("=== CRC-8 Calculation ===")

function makeCRC8Table(polynomial)
    polynomial = polynomial or 0x07
    local table = {}
    for i = 0, 255 do
        local crc = i
        for _ = 1, 8 do
            if crc & 0x80 ~= 0 then
                crc = ((crc << 1) ~ polynomial) & 0xFF
            else
                crc = (crc << 1) & 0xFF
            end
        end
        table[i] = crc
    end
    return table
end

local crc8Table = makeCRC8Table()

function crc8(data, crcTable)
    crcTable = crcTable or crc8Table
    local crc = 0
    for i = 1, #data do
        local byte = string.byte(data, i)
        crc = crcTable[(crc ~ byte) & 0xFF]
    end
    return crc
end

local testData = "Hello, World!"
local crc = crc8(testData)
print(string.format("Data: '%s'", testData))
print(string.format("CRC-8: 0x%02X (%d)", crc, crc))
```

```lua
-- ตัวอย่างที่ 22: CRC-16 calculation
print("=== CRC-16 Calculation ===")

function crc16(data)
    local crc = 0xFFFF
    for i = 1, #data do
        local byte = string.byte(data, i)
        crc = crc ~ byte
        for _ = 1, 8 do
            if crc & 0x0001 ~= 0 then
                crc = (crc >> 1) ~ 0xA001
            else
                crc = crc >> 1
            end
        end
    end
    return crc & 0xFFFF
end

local data = "123456789"
local crc = crc16(data)
print(string.format("Data: '%s'", data))
print(string.format("CRC-16: 0x%04X (%d)", crc, crc))
```

```lua
-- ตัวอย่างที่ 23: CRC-32 calculation
print("=== CRC-32 Calculation ===")

function makeCRC32Table()
    local t = {}
    for i = 0, 255 do
        local c = i
        for _ = 1, 8 do
            if c & 1 ~= 0 then
                c = 0xEDB88320 ~ (c >> 1)
            else
                c = c >> 1
            end
        end
        t[i] = c
    end
    return t
end

local crc32Table = makeCRC32Table()

function crc32(data)
    local crc = 0xFFFFFFFF
    for i = 1, #data do
        local byte = string.byte(data, i)
        crc = crc32Table[(crc ~ byte) & 0xFF] ~ (crc >> 8)
    end
    return (crc ~ 0xFFFFFFFF) & 0xFFFFFFFF
end

local text = "The quick brown fox jumps over the lazy dog"
local checksum = crc32(text)
print(string.format("Data: '%s'", text))
print(string.format("CRC-32: 0x%08X", checksum))
```

## 36.10 Hamming Weight (Population Count)

```lua
-- ตัวอย่างที่ 24: Hamming weight (จำนวนบิต 1)
print("=== Hamming Weight ===")

-- วิธีที่ 1: loop ธรรมดา
function hammingWeight_basic(n)
    local count = 0
    while n ~= 0 do
        count = count + (n & 1)
        n = n >> 1
    end
    return count
end

-- วิธีที่ 2: Brian Kernighan's algorithm
function hammingWeight_fast(n)
    local count = 0
    while n ~= 0 do
        n = n & (n - 1)  -- clear least significant set bit
        count = count + 1
    end
    return count
end

-- วิธีที่ 3: parallel bit counting
function hammingWeight_parallel(n)
    n = n & 0x7FFFFFFFFFFFFFFF  -- ทำงานกับ 63 บิต (หลีกเลี่ยง sign bit)
    n = n - ((n >> 1) & 0x5555555555555555)
    n = (n & 0x3333333333333333) + ((n >> 2) & 0x3333333333333333)
    n = (n + (n >> 4)) & 0x0F0F0F0F0F0F0F0F
    return (n * 0x0101010101010101) >> 56
end

local values = {0, 1, 7, 255, 0xDEADBEEF}
for _, v in ipairs(values) do
    print(string.format("popcount(%s) = %d (fast: %d)",
        toBinary(v, 16), hammingWeight_basic(v), hammingWeight_fast(v)))
end
```

## 36.11 Power of 2 Checks

```lua
-- ตัวอย่างที่ 25: ตรวจสอบว่าเป็น power of 2
print("=== Power of 2 Checks ===")

function isPowerOf2(n)
    if n <= 0 then return false end
    return (n & (n - 1)) == 0
end

function nextPowerOf2(n)
    if n <= 0 then return 1 end
    n = n - 1
    n = n | (n >> 1)
    n = n | (n >> 2)
    n = n | (n >> 4)
    n = n | (n >> 8)
    n = n | (n >> 16)
    n = n | (n >> 32)
    return n + 1
end

function prevPowerOf2(n)
    if n <= 0 then return 0 end
    n = n | (n >> 1)
    n = n | (n >> 2)
    n = n | (n >> 4)
    n = n | (n >> 8)
    n = n | (n >> 16)
    n = n | (n >> 32)
    return n - (n >> 1)
end

for _, n in ipairs({1, 2, 3, 4, 5, 8, 9, 16, 100, 128, 256}) do
    print(string.format("isPowerOf2(%4d) = %-5s  next=%4d  prev=%4d",
        n, tostring(isPowerOf2(n)), nextPowerOf2(n), prevPowerOf2(n)))
end
```

```lua
-- ตัวอย่างที่ 26: Log2 สำหรับ power of 2
print("=== Log2 for Power of 2 ===")

function log2(n)
    if n <= 0 then return -1 end
    local count = 0
    while n > 1 do
        n = n >> 1
        count = count + 1
    end
    return count
end

-- หา position ของ MSB (Most Significant Bit)
function msbPosition(n)
    if n <= 0 then return -1 end
    local pos = 0
    while n > 1 do
        n = n >> 1
        pos = pos + 1
    end
    return pos
end

for _, n in ipairs({1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024}) do
    print(string.format("log2(%5d) = %2d  MSB pos = %2d", n, log2(n), msbPosition(n)))
end
```

## 36.12 XOR Encryption

```lua
-- ตัวอย่างที่ 27: XOR Encryption พื้นฐาน
print("=== XOR Encryption ===")

function xorEncrypt(text, key)
    local result = {}
    local keyLen = #key
    for i = 1, #text do
        local charCode = string.byte(text, i)
        local keyCode = string.byte(key, ((i - 1) % keyLen) + 1)
        table.insert(result, string.char(charCode ~ keyCode))
    end
    return table.concat(result)
end

-- XOR encryption is symmetric (encrypt == decrypt)
xorDecrypt = xorEncrypt

local message = "Hello, Secret World!"
local key = "MyKey123"

local encrypted = xorEncrypt(message, key)
local decrypted = xorDecrypt(encrypted, key)

print(string.format("Original:  '%s'", message))
print(string.format("Key:       '%s'", key))
print("Encrypted: (binary data)")
for i = 1, #encrypted do
    io.write(string.format("%02X ", string.byte(encrypted, i)))
end
print()
print(string.format("Decrypted: '%s'", decrypted))
print(string.format("Match: %s", tostring(message == decrypted)))
```

```lua
-- ตัวอย่างที่ 28: XOR key stream encryption (Vernam cipher variant)
print("=== XOR Key Stream ===")

function generateKeyStream(seed, length)
    local stream = {}
    local state = seed
    for _ = 1, length do
        -- Linear Congruential Generator
        state = (state * 1664525 + 1013904223) & 0xFFFFFFFF
        table.insert(stream, state & 0xFF)
    end
    return stream
end

function encryptWithStream(text, seed)
    local keyStream = generateKeyStream(seed, #text)
    local result = {}
    for i = 1, #text do
        local charCode = string.byte(text, i)
        table.insert(result, string.char(charCode ~ keyStream[i]))
    end
    return table.concat(result)
end

local original = "Secret message here!"
local seed = 42

local enc = encryptWithStream(original, seed)
local dec = encryptWithStream(enc, seed)

print(string.format("Original:  '%s'", original))
print(string.format("Decrypted: '%s'", dec))
print(string.format("Match: %s", tostring(original == dec)))
```

## 36.13 Compression Basics

```lua
-- ตัวอย่างที่ 29: Run-Length Encoding (RLE) พื้นฐาน
print("=== Run-Length Encoding ===")

function rleEncode(data)
    if #data == 0 then return "" end
    local result = {}
    local count = 1
    local current = string.byte(data, 1)
    
    for i = 2, #data do
        local byte = string.byte(data, i)
        if byte == current and count < 255 then
            count = count + 1
        else
            table.insert(result, string.char(count, current))
            current = byte
            count = 1
        end
    end
    table.insert(result, string.char(count, current))
    return table.concat(result)
end

function rleDecode(data)
    local result = {}
    local i = 1
    while i <= #data - 1 do
        local count = string.byte(data, i)
        local byte = string.byte(data, i + 1)
        for _ = 1, count do
            table.insert(result, string.char(byte))
        end
        i = i + 2
    end
    return table.concat(result)
end

local original = "AAABBBBBCCDDDDDDDDEE"
local encoded = rleEncode(original)
local decoded = rleDecode(encoded)

print(string.format("Original (%d bytes): '%s'", #original, original))
print(string.format("Encoded  (%d bytes)", #encoded))
print(string.format("Decoded  (%d bytes): '%s'", #decoded, decoded))
print(string.format("Compression ratio: %.1f%%", (1 - #encoded / #original) * 100))
print(string.format("Match: %s", tostring(original == decoded)))
```

```lua
-- ตัวอย่างที่ 30: Bit packing (เก็บค่าเล็กในพื้นที่น้อย)
print("=== Bit Packing ===")

-- เก็บค่า 0-15 (4 bits) หลายค่าใน string
function packNibbles(values)
    local bytes = {}
    for i = 1, #values, 2 do
        local high = values[i] & 0x0F
        local low = i + 1 <= #values and values[i + 1] & 0x0F or 0
        table.insert(bytes, string.char((high << 4) | low))
    end
    return table.concat(bytes)
end

function unpackNibbles(data, count)
    local values = {}
    for i = 1, #data do
        local byte = string.byte(data, i)
        table.insert(values, (byte >> 4) & 0x0F)
        if #values < count then
            table.insert(values, byte & 0x0F)
        end
    end
    return values
end

local original = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12}
local packed = packNibbles(original)
local unpacked = unpackNibbles(packed, #original)

print(string.format("Original values (%d bytes): %s",
    #original, table.concat(original, ", ")))
print(string.format("Packed (%d bytes)", #packed))
print(string.format("Unpacked: %s", table.concat(unpacked, ", ")))
print(string.format("Space saved: %d bytes", #original - #packed))
```

## 36.14 Advanced Bit Operations

```lua
-- ตัวอย่างที่ 31: Byte reversal
print("=== Byte Reversal ===")

function reverseByte(b)
    b = ((b & 0xF0) >> 4) | ((b & 0x0F) << 4)
    b = ((b & 0xCC) >> 2) | ((b & 0x33) << 2)
    b = ((b & 0xAA) >> 1) | ((b & 0x55) << 1)
    return b
end

function reverseUint32(n)
    local result = 0
    for _ = 1, 4 do
        result = (result << 8) | (n & 0xFF)
        n = n >> 8
    end
    return result
end

print(string.format("reverseByte(0b00000001) = %s", toBinary(reverseByte(0b00000001))))
print(string.format("reverseByte(0b10110100) = %s", toBinary(reverseByte(0b10110100))))
print(string.format("reverseUint32(0x12345678) = 0x%08X", reverseUint32(0x12345678)))
```

```lua
-- ตัวอย่างที่ 32: Bit rotation
print("=== Bit Rotation ===")

function rotateLeft(n, bits, size)
    size = size or 32
    bits = bits % size
    local mask = (1 << size) - 1
    return ((n << bits) | (n >> (size - bits))) & mask
end

function rotateRight(n, bits, size)
    size = size or 32
    bits = bits % size
    local mask = (1 << size) - 1
    return ((n >> bits) | (n << (size - bits))) & mask
end

local n = 0b10110001
print(string.format("Original:    %s", toBinary(n, 8)))
print(string.format("ROL by 1:    %s", toBinary(rotateLeft(n, 1, 8), 8)))
print(string.format("ROL by 3:    %s", toBinary(rotateLeft(n, 3, 8), 8)))
print(string.format("ROR by 1:    %s", toBinary(rotateRight(n, 1, 8), 8)))
print(string.format("ROR by 3:    %s", toBinary(rotateRight(n, 3, 8), 8)))
```

```lua
-- ตัวอย่างที่ 33: Swap two values โดยไม่ใช้ temporary variable
print("=== XOR Swap ===")

local function xorSwap(a, b)
    a = a ~ b
    b = a ~ b
    a = a ~ b
    return a, b
end

local x, y = 42, 137
print(string.format("Before: x=%d, y=%d", x, y))
x, y = xorSwap(x, y)
print(string.format("After:  x=%d, y=%d", x, y))
```

```lua
-- ตัวอย่างที่ 34: Absolute value โดยใช้ bit operations
print("=== Bit Tricks ===")

function abs_bit(n)
    local mask = n >> 63  -- sign bit extended: 0 or -1
    return (n + mask) ~ mask
end

function min_bit(a, b)
    return b + ((a - b) & ((a - b) >> 63))
end

function max_bit(a, b)
    return a - ((a - b) & ((a - b) >> 63))
end

print(string.format("abs(-42) = %d", abs_bit(-42)))
print(string.format("abs(42)  = %d", abs_bit(42)))
print(string.format("min(3, 7) = %d", min_bit(3, 7)))
print(string.format("max(3, 7) = %d", max_bit(3, 7)))
```

## 36.15 Bloom Filter (ตัวอย่างการใช้งานจริง)

```lua
-- ตัวอย่างที่ 35: Bloom Filter implementation
print("=== Bloom Filter ===")

local BloomFilter = {}
BloomFilter.__index = BloomFilter

function BloomFilter.new(size, hashCount)
    size = size or 1024
    hashCount = hashCount or 3
    local bf = setmetatable({
        size = size,
        hashCount = hashCount,
        bits = {},
        count = 0
    }, BloomFilter)
    -- Initialize all bits to 0
    for i = 0, math.ceil(size / 64) do
        bf.bits[i] = 0
    end
    return bf
end

function BloomFilter:_hash(item, seed)
    -- Simple hash function
    local h = seed
    for i = 1, #item do
        h = h ~ (string.byte(item, i) * (i + 1))
        h = rotateLeft(h, 7, 32)
        h = (h * 0x9e3779b9) & 0xFFFFFFFF
    end
    return h % self.size
end

function BloomFilter:_setBit(pos)
    local idx = pos >> 6
    local bit = pos & 63
    self.bits[idx] = self.bits[idx] | (1 << bit)
end

function BloomFilter:_checkBit(pos)
    local idx = pos >> 6
    local bit = pos & 63
    return (self.bits[idx] & (1 << bit)) ~= 0
end

function BloomFilter:add(item)
    for seed = 1, self.hashCount do
        local pos = self:_hash(item, seed * 0x12345678)
        self:_setBit(pos)
    end
    self.count = self.count + 1
end

function BloomFilter:contains(item)
    for seed = 1, self.hashCount do
        local pos = self:_hash(item, seed * 0x12345678)
        if not self:_checkBit(pos) then
            return false  -- definitely not in set
        end
    end
    return true  -- probably in set
end

local bf = BloomFilter.new(256, 3)
local words = {"apple", "banana", "cherry", "date", "elderberry"}

for _, word in ipairs(words) do
    bf:add(word)
    print(string.format("Added: '%s'", word))
end

print("\nTesting membership:")
local testWords = {"apple", "banana", "fig", "grape", "date", "kiwi"}
for _, word in ipairs(testWords) do
    local contains = bf:contains(word)
    print(string.format("  '%s': %s", word, contains and "probably yes" or "definitely no"))
end
```

## 36.16 Gray Code

```lua
-- ตัวอย่างที่ 36: Gray Code conversion
print("=== Gray Code ===")

function toGrayCode(n)
    return n ~ (n >> 1)
end

function fromGrayCode(g)
    local n = g
    while g ~= 0 do
        g = g >> 1
        n = n ~ g
    end
    return n
end

print("Binary   Gray     Decimal")
print("------   ------   -------")
for i = 0, 15 do
    local gray = toGrayCode(i)
    print(string.format("  %s   %s   %2d -> %2d",
        toBinary(i, 4), toBinary(gray, 4), i, fromGrayCode(gray)))
end
```

## 36.17 Checksum และ Parity

```lua
-- ตัวอย่างที่ 37: Parity check
print("=== Parity Check ===")

function evenParity(n)
    local count = hammingWeight_fast(n)
    return count % 2 == 0
end

function oddParity(n)
    return not evenParity(n)
end

function addParityBit(byte, useEven)
    useEven = useEven ~= false  -- default true (even parity)
    byte = byte & 0x7F  -- use 7 bits
    local parity = evenParity(byte)
    if useEven then
        return byte | (parity and 0 or 0x80)
    else
        return byte | (parity and 0x80 or 0)
    end
end

function checkParityBit(byte, useEven)
    useEven = useEven ~= false
    return evenParity(byte) == useEven
end

for _, byte in ipairs({0x41, 0x42, 0x7F, 0x00}) do
    local withParity = addParityBit(byte)
    local valid = checkParityBit(withParity)
    print(string.format("0x%02X -> 0x%02X (parity: %s)",
        byte, withParity, valid and "valid" or "invalid"))
end
```

```lua
-- ตัวอย่างที่ 38: Internet checksum (used in IP/TCP/UDP)
print("=== Internet Checksum ===")

function internetChecksum(data)
    local sum = 0
    local i = 1
    
    -- Process in 16-bit words
    while i < #data do
        local word = (string.byte(data, i) << 8)
        if i + 1 <= #data then
            word = word | string.byte(data, i + 1)
        end
        sum = sum + word
        i = i + 2
    end
    
    -- Add carry
    while sum > 0xFFFF do
        sum = (sum & 0xFFFF) + (sum >> 16)
    end
    
    return (~sum) & 0xFFFF
end

local data = "Hello, Checksum!"
local checksum = internetChecksum(data)
print(string.format("Data: '%s'", data))
print(string.format("Checksum: 0x%04X", checksum))
```

## 36.18 Bit Matrix Operations

```lua
-- ตัวอย่างที่ 39: Bit matrix (บิตแมทริกซ์)
print("=== Bit Matrix ===")

local BitMatrix = {}
BitMatrix.__index = BitMatrix

function BitMatrix.new(rows, cols)
    local m = setmetatable({
        rows = rows,
        cols = cols,
        data = {}
    }, BitMatrix)
    local wordsPerRow = math.ceil(cols / 64)
    for r = 1, rows do
        m.data[r] = {}
        for w = 1, wordsPerRow do
            m.data[r][w] = 0
        end
    end
    return m
end

function BitMatrix:set(row, col, value)
    local wordIdx = ((col - 1) >> 6) + 1
    local bitIdx = (col - 1) & 63
    if value then
        self.data[row][wordIdx] = self.data[row][wordIdx] | (1 << bitIdx)
    else
        self.data[row][wordIdx] = self.data[row][wordIdx] & ~(1 << bitIdx)
    end
end

function BitMatrix:get(row, col)
    local wordIdx = ((col - 1) >> 6) + 1
    local bitIdx = (col - 1) & 63
    return (self.data[row][wordIdx] >> bitIdx) & 1 == 1
end

function BitMatrix:print()
    for r = 1, self.rows do
        local row = ""
        for c = 1, self.cols do
            row = row .. (self:get(r, c) and "1" or "0")
            if c < self.cols then row = row .. " " end
        end
        print("  " .. row)
    end
end

local m = BitMatrix.new(4, 4)
-- สร้าง identity matrix
for i = 1, 4 do m:set(i, i, true) end
print("Identity Matrix:")
m:print()

-- สร้าง checkerboard
local checker = BitMatrix.new(4, 4)
for r = 1, 4 do
    for c = 1, 4 do
        checker:set(r, c, (r + c) % 2 == 0)
    end
end
print("Checkerboard:")
checker:print()
```

## 36.19 Practical Example: Network Packet Flags

```lua
-- ตัวอย่างที่ 40: TCP Flags
print("=== TCP Flags ===")

local TCPFlags = {
    FIN = 1 << 0,   -- 0x01
    SYN = 1 << 1,   -- 0x02
    RST = 1 << 2,   -- 0x04
    PSH = 1 << 3,   -- 0x08
    ACK = 1 << 4,   -- 0x10
    URG = 1 << 5,   -- 0x20
    ECE = 1 << 6,   -- 0x40
    CWR = 1 << 7,   -- 0x80
}

function describeTCPFlags(flags)
    local active = {}
    for name, flag in pairs(TCPFlags) do
        if (flags & flag) ~= 0 then
            table.insert(active, name)
        end
    end
    table.sort(active)
    return table.concat(active, "|")
end

local packets = {
    {flags = TCPFlags.SYN,              desc = "SYN (connection initiation)"},
    {flags = TCPFlags.SYN | TCPFlags.ACK, desc = "SYN-ACK (server response)"},
    {flags = TCPFlags.ACK,              desc = "ACK (acknowledgment)"},
    {flags = TCPFlags.PSH | TCPFlags.ACK, desc = "PSH-ACK (data push)"},
    {flags = TCPFlags.FIN | TCPFlags.ACK, desc = "FIN-ACK (connection close)"},
    {flags = TCPFlags.RST,              desc = "RST (reset)"},
}

for _, pkt in ipairs(packets) do
    print(string.format("0x%02X (%s) -> %s",
        pkt.flags, describeTCPFlags(pkt.flags), pkt.desc))
end
```

```lua
-- ตัวอย่างที่ 41: IPv4 address manipulation
print("=== IPv4 Address Manipulation ===")

function ipToInt(ip)
    local a, b, c, d = ip:match("(%d+)%.(%d+)%.(%d+)%.(%d+)")
    return (tonumber(a) << 24) | (tonumber(b) << 16) | 
           (tonumber(c) << 8) | tonumber(d)
end

function intToIp(n)
    return string.format("%d.%d.%d.%d",
        (n >> 24) & 0xFF,
        (n >> 16) & 0xFF,
        (n >> 8) & 0xFF,
        n & 0xFF)
end

function subnetMask(prefix)
    if prefix == 0 then return 0 end
    return (~((1 << (32 - prefix)) - 1)) & 0xFFFFFFFF
end

function networkAddress(ip, prefix)
    local ipInt = ipToInt(ip)
    local mask = subnetMask(prefix)
    return intToIp(ipInt & mask)
end

function broadcastAddress(ip, prefix)
    local ipInt = ipToInt(ip)
    local mask = subnetMask(prefix)
    return intToIp(ipInt | (~mask & 0xFFFFFFFF))
end

function hostsCount(prefix)
    return (1 << (32 - prefix)) - 2
end

local ip = "192.168.1.100"
local prefix = 24

print(string.format("IP Address:        %s", ip))
print(string.format("Subnet Prefix:     /%d", prefix))
print(string.format("Subnet Mask:       %s", intToIp(subnetMask(prefix))))
print(string.format("Network Address:   %s", networkAddress(ip, prefix)))
print(string.format("Broadcast Address: %s", broadcastAddress(ip, prefix)))
print(string.format("Usable Hosts:      %d", hostsCount(prefix)))
print(string.format("IP as integer:     %d (0x%08X)", ipToInt(ip), ipToInt(ip)))
```

## 36.20 สรุปบทที่ 36

```lua
-- ตัวอย่างที่ 42: สรุปการใช้งาน bit operations ในระบบจริง
print("=== สรุป Bit Operations ===")

-- ตารางสรุป operators
local summary = {
    {"&",  "AND",        "เปรียบเทียบบิต: 1 เมื่อทั้งคู่เป็น 1"},
    {"|",  "OR",         "เปรียบเทียบบิต: 1 เมื่ออย่างน้อยหนึ่งตัวเป็น 1"},
    {"~",  "XOR / NOT",  "XOR: ต่างกัน=1; NOT (unary): สลับบิต"},
    {"<<", "Left Shift", "เลื่อนบิตไปซ้าย (คูณด้วย 2^n)"},
    {">>", "Right Shift","เลื่อนบิตไปขวา (หารด้วย 2^n)"},
}

print("\nBitwise Operators:")
for _, row in ipairs(summary) do
    print(string.format("  %-4s %-12s %s", row[1], row[2], row[3]))
end

-- Use cases
print("\nUse Cases:")
print("  - Setting/clearing/toggling flags")
print("  - Color packing (RGBA)")  
print("  - Network protocol handling")
print("  - Encryption (XOR)")
print("  - Checksums and CRC")
print("  - Compression")
print("  - Fast arithmetic (power of 2)")
print("  - Bloom filters")
print("  - Permission systems")
```

---

## แบบฝึกหัดบทที่ 36

1. เขียนฟังก์ชัน `countOnes(n)` ที่นับจำนวนบิต 1 ในจำนวน n โดยใช้วิธี Brian Kernighan
2. เขียนฟังก์ชัน `rotateLeft32(n, k)` ที่หมุน 32-bit integer ไปทางซ้าย k บิต
3. สร้าง `FlagSystem` class ที่รองรับ named flags และ CRUD operations
4. เขียน function เพื่อแปลง IP CIDR notation เป็น range ของ IP addresses
5. Implement simple Adler-32 checksum โดยใช้ bit operations
