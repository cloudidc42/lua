# บทที่ 42: Serialization และ Deserialization

## บทนำ

Serialization คือกระบวนการแปลง data structure (เช่น Lua table) ให้อยู่ในรูปแบบที่สามารถเก็บหรือส่งผ่านเครือข่ายได้ เช่น string หรือ binary data ส่วน Deserialization คือการแปลงกลับ

ในบทนี้เราจะสร้างระบบ serialization ตั้งแต่พื้นฐาน ไปจนถึงรูปแบบ binary และการจัดการกับปัญหาต่างๆ เช่น circular references

---

## 42.1 พื้นฐาน Serialization

### ตัวอย่างที่ 1: ทำไมต้องมี Serialization

```lua
-- ปัญหา: Lua tables ไม่สามารถเก็บลง disk หรือส่งผ่าน network ได้โดยตรง
local userData = {
    id = 42,
    name = "John Doe",
    email = "john@example.com",
    scores = {95, 87, 92, 88},
    settings = {
        theme = "dark",
        language = "th",
        notifications = true
    }
}

-- เราต้องแปลงเป็น string ก่อน
-- วิธีที่ง่ายที่สุด (แต่ไม่ปลอดภัย): tostring()
print(tostring(userData))  -- table: 0x55a5b3c4d2e0 - ไม่มีประโยชน์!

-- เราต้องการ serializer ที่ดีกว่า
local function simpleSerialize(t)
    if type(t) == "table" then
        local parts = {}
        for k, v in pairs(t) do
            local key = type(k) == "string" and string.format("[%q]", k) or string.format("[%d]", k)
            table.insert(parts, key .. "=" .. simpleSerialize(v))
        end
        return "{" .. table.concat(parts, ",") .. "}"
    elseif type(t) == "string" then
        return string.format("%q", t)
    elseif type(t) == "number" then
        return tostring(t)
    elseif type(t) == "boolean" then
        return tostring(t)
    elseif type(t) == "nil" then
        return "nil"
    end
    return tostring(t)
end

print(simpleSerialize(userData))
```

### ตัวอย่างที่ 2: Simple Round-Trip Serialization

```lua
-- round_trip.lua
-- Serialize ไป string แล้ว deserialize กลับได้

local function serialize(value, indent, seen)
    seen = seen or {}
    indent = indent or 0
    local t = type(value)
    
    if t == "nil" then
        return "nil"
    elseif t == "boolean" then
        return tostring(value)
    elseif t == "number" then
        -- จัดการ special values
        if value ~= value then return "(0/0)" end  -- NaN
        if value == math.huge then return "math.huge" end
        if value == -math.huge then return "-math.huge" end
        -- ใช้ format ที่ preserve precision
        if math.floor(value) == value and math.abs(value) < 2^53 then
            return string.format("%d", value)
        end
        return string.format("%.17g", value)
    elseif t == "string" then
        return string.format("%q", value)
    elseif t == "table" then
        -- ตรวจ circular reference
        if seen[value] then
            return '"[circular]"'
        end
        seen[value] = true
        
        local spaces = string.rep("  ", indent)
        local innerSpaces = string.rep("  ", indent + 1)
        
        -- ตรวจสอบว่าเป็น array
        local isArray = true
        local n = 0
        for k in pairs(value) do
            n = n + 1
            if type(k) ~= "number" or k < 1 or k ~= math.floor(k) then
                isArray = false
                break
            end
        end
        isArray = isArray and n == #value
        
        local parts = {}
        if isArray then
            for i, v in ipairs(value) do
                table.insert(parts, innerSpaces .. serialize(v, indent + 1, seen))
            end
            seen[value] = nil
            if #parts == 0 then return "{}" end
            return "{\n" .. table.concat(parts, ",\n") .. "\n" .. spaces .. "}"
        else
            local keys = {}
            for k in pairs(value) do table.insert(keys, k) end
            table.sort(keys, function(a, b)
                if type(a) == type(b) then
                    return tostring(a) < tostring(b)
                end
                return type(a) < type(b)
            end)
            
            for _, k in ipairs(keys) do
                local keyStr
                if type(k) == "string" and k:match("^[%a_][%w_]*$") then
                    keyStr = k
                else
                    keyStr = "[" .. serialize(k, 0, seen) .. "]"
                end
                local valStr = serialize(value[k], indent + 1, seen)
                table.insert(parts, innerSpaces .. keyStr .. " = " .. valStr)
            end
            seen[value] = nil
            if #parts == 0 then return "{}" end
            return "{\n" .. table.concat(parts, ",\n") .. "\n" .. spaces .. "}"
        end
    else
        return string.format('"[%s]"', t)
    end
end

local function deserialize(str)
    -- ใช้ loadstring/load เพื่อ deserialize
    local fn, err = load("return " .. str)
    if not fn then
        return nil, "Parse error: " .. (err or "unknown")
    end
    local ok, result = pcall(fn)
    if not ok then
        return nil, "Execution error: " .. tostring(result)
    end
    return result
end

-- ทดสอบ round-trip
local original = {
    name = "สวัสดี Lua",
    version = 5.4,
    active = true,
    tags = {"lua", "serialization", "round-trip"},
    config = {
        debug = false,
        maxRetries = 3,
        timeout = 30.5
    },
    nilValue = nil,
    emptyTable = {}
}

local serialized = serialize(original)
print("=== Serialized ===")
print(serialized)

local restored, err = deserialize(serialized)
if err then
    print("Error: " .. err)
else
    print("\n=== Restored ===")
    print("name: " .. (restored.name or "nil"))
    print("version: " .. (restored.version or 0))
    print("active: " .. tostring(restored.active))
    print("tags[1]: " .. (restored.tags and restored.tags[1] or "nil"))
    print("config.debug: " .. tostring(restored.config and restored.config.debug))
end
```

---

## 42.2 Handling Circular References

### ตัวอย่างที่ 3: ตรวจจับ Circular References

```lua
-- circular_detection.lua

local function detectCircular(value, seen, path)
    seen = seen or {}
    path = path or "root"
    
    if type(value) ~= "table" then return false end
    
    if seen[value] then
        print(string.format("CIRCULAR: %s -> %s", path, seen[value]))
        return true
    end
    
    seen[value] = path
    
    for k, v in pairs(value) do
        local keyPath = path .. "." .. tostring(k)
        if detectCircular(v, seen, keyPath) then
            seen[value] = nil
            return true
        end
    end
    
    seen[value] = nil
    return false
end

-- ทดสอบ circular references
local a = {name = "A"}
local b = {name = "B"}
local c = {name = "C"}

-- สร้าง cycle
a.child = b
b.child = c
-- c.child = a  -- uncomment นี้เพื่อสร้าง cycle

print("Without cycle:")
print(detectCircular(a))

-- สร้าง cycle
c.child = a
print("\nWith cycle:")
print(detectCircular(a))

-- Self-reference
local self_ref = {name = "Self"}
self_ref.self = self_ref
print("\nSelf-reference:")
print(detectCircular(self_ref))
```

### ตัวอย่างที่ 4: Serializer ที่จัดการ Circular References

```lua
-- safe_serializer.lua

local SafeSerializer = {}
SafeSerializer.__index = SafeSerializer

function SafeSerializer.new(options)
    local self = setmetatable({}, SafeSerializer)
    self.maxDepth = options and options.maxDepth or 50
    self.onCircular = options and options.onCircular or "reference"  -- "reference", "omit", "error"
    return self
end

function SafeSerializer:serialize(value)
    self._seen = {}
    self._refs = {}
    self._refCount = 0
    
    -- Phase 1: ค้นหา shared/circular references
    self:_scan(value, 1)
    
    -- Phase 2: serialize
    return self:_serialize(value, 0)
end

function SafeSerializer:_scan(value, depth)
    if type(value) ~= "table" or depth > self.maxDepth then return end
    
    local id = self._seen[value]
    if id then
        -- พบซ้ำ - เพิ่มลงใน refs
        if not self._refs[value] then
            self._refCount = self._refCount + 1
            self._refs[value] = self._refCount
        end
        return
    end
    
    self._seen[value] = true
    for _, v in pairs(value) do
        self:_scan(v, depth + 1)
    end
end

function SafeSerializer:_serialize(value, depth)
    if depth > self.maxDepth then
        return '"[MAX_DEPTH]"'
    end
    
    local t = type(value)
    if t == "nil" then return "nil"
    elseif t == "boolean" then return tostring(value)
    elseif t == "number" then return tostring(value)
    elseif t == "string" then return string.format("%q", value)
    elseif t == "table" then
        if self._refs[value] then
            -- เป็น shared/circular reference
            if self.onCircular == "omit" then
                return "nil"
            elseif self.onCircular == "error" then
                error("Circular reference detected")
            else
                return string.format('"[ref:%d]"', self._refs[value])
            end
        end
        
        -- Mark as being processed
        local refId = self._refs[value]
        self._refs[value] = "PROCESSING"
        
        local parts = {}
        local keys = {}
        for k in pairs(value) do table.insert(keys, k) end
        table.sort(keys, function(a, b) return tostring(a) < tostring(b) end)
        
        for _, k in ipairs(keys) do
            local keyStr = type(k) == "string" and
                (k:match("^[%a_][%w_]*$") and k or string.format("[%q]", k)) or
                string.format("[%d]", k)
            local valStr = self:_serialize(value[k], depth + 1)
            table.insert(parts, keyStr .. "=" .. valStr)
        end
        
        if refId then self._refs[value] = refId end
        return "{" .. table.concat(parts, ",") .. "}"
    end
    return '"[' .. t .. ']"'
end

-- ทดสอบ
local ser = SafeSerializer.new({onCircular = "reference"})

-- Normal table
local data = {a = 1, b = {c = 2, d = {e = 3}}}
print("Normal: " .. ser:serialize(data))

-- Shared reference
local shared = {value = 42}
local withShared = {x = shared, y = shared}
print("Shared: " .. ser:serialize(withShared))

-- Circular
local circ = {name = "node1"}
circ.next = {name = "node2", prev = circ}
print("Circular: " .. ser:serialize(circ))
```

---

## 42.3 Custom Serialization Formats

### ตัวอย่างที่ 5: Lua Table to String (Production Quality)

```lua
-- lua_serializer.lua
-- สร้าง serializer ที่ output เป็น valid Lua code

local function luaSerialize(value, options)
    options = options or {}
    local indent = options.indent or "  "
    local sortKeys = options.sortKeys ~= false
    
    local function ser(val, depth, seen)
        seen = seen or {}
        depth = depth or 0
        local pad = string.rep(indent, depth)
        local innerPad = string.rep(indent, depth + 1)
        
        local t = type(val)
        if t == "nil" then
            return "nil"
        elseif t == "boolean" then
            return tostring(val)
        elseif t == "number" then
            if val ~= val then return "(0/0)" end
            if val == math.huge then return "math.huge" end
            if val == -math.huge then return "-math.huge" end
            if math.type and math.type(val) == "integer" then
                return string.format("%d", val)
            end
            local s = string.format("%.17g", val)
            return s
        elseif t == "string" then
            -- ตรวจสอบว่าใช้ long string หรือ quoted string
            if val:find("[\0\1\2\3\4\5\6\7\8\9\10\11\12\13\14\15\16\17\18\19\20\21\22\23\24\25\26\27\28\29\30\31]") then
                return string.format("%q", val)
            end
            if val:find('"') and not val:find("'") then
                return "'" .. val .. "'"
            end
            return string.format("%q", val)
        elseif t == "table" then
            if seen[val] then return '"__circular__"' end
            seen[val] = true
            
            -- แยก array part และ hash part
            local arrayPart = {}
            local hashPart = {}
            local arrayLen = #val
            
            for i = 1, arrayLen do
                arrayPart[i] = val[i]
            end
            
            local seenIdx = {}
            for i = 1, arrayLen do seenIdx[i] = true end
            
            for k, v in pairs(val) do
                if not seenIdx[k] then
                    table.insert(hashPart, {k = k, v = v})
                end
            end
            
            if sortKeys then
                table.sort(hashPart, function(a, b)
                    if type(a.k) == type(b.k) then
                        return tostring(a.k) < tostring(b.k)
                    end
                    return type(a.k) < type(b.k)
                end)
            end
            
            local parts = {}
            
            -- Array part
            for _, v in ipairs(arrayPart) do
                table.insert(parts, innerPad .. ser(v, depth + 1, seen))
            end
            
            -- Hash part
            for _, pair in ipairs(hashPart) do
                local keyStr
                if type(pair.k) == "string" and pair.k:match("^[%a_][%w_]*$") then
                    keyStr = pair.k
                else
                    keyStr = "[" .. ser(pair.k, 0, seen) .. "]"
                end
                table.insert(parts, innerPad .. keyStr .. " = " .. ser(pair.v, depth + 1, seen))
            end
            
            seen[val] = nil
            
            if #parts == 0 then return "{}" end
            return "{\n" .. table.concat(parts, ",\n") .. "\n" .. pad .. "}"
        else
            return string.format('"[%s: %s]"', t, tostring(val))
        end
    end
    
    return ser(value)
end

-- ทดสอบ
local complex = {
    -- Mixed array and hash
    "first",
    "second",
    "third",
    name = "complex table",
    nested = {
        a = 1,
        b = {true, false, nil, 3.14},
        c = "line1\nline2\ttabbed"
    },
    numbers = {
        int = 42,
        float = 3.14159265358979,
        big = 9007199254740992,
        neg = -1
    }
}

print(luaSerialize(complex))
print()
print("-- As Lua code, can be loaded with: load('return ' .. str)()")
```

### ตัวอย่างที่ 6: JSON Serializer (ไม่ต้องใช้ library)

```lua
-- json_serializer.lua

local JSON = {}

-- Escape special characters ใน JSON strings
local function escapeString(s)
    local escapes = {
        ['"']  = '\\"',
        ['\\'] = '\\\\',
        ['\n'] = '\\n',
        ['\r'] = '\\r',
        ['\t'] = '\\t',
        ['\b'] = '\\b',
        ['\f'] = '\\f',
    }
    return s:gsub('[%c"\\]', function(c)
        return escapes[c] or string.format('\\u%04x', c:byte())
    end)
end

function JSON.encode(value, pretty, indent)
    pretty = pretty or false
    indent = indent or "  "
    
    local function enc(val, depth, seen)
        seen = seen or {}
        depth = depth or 0
        local t = type(val)
        
        if t == "nil" then
            return "null"
        elseif t == "boolean" then
            return tostring(val)
        elseif t == "number" then
            if val ~= val or val == math.huge or val == -math.huge then
                return "null"
            end
            if math.floor(val) == val then
                return string.format("%d", val)
            end
            return string.format("%.10g", val)
        elseif t == "string" then
            return '"' .. escapeString(val) .. '"'
        elseif t == "table" then
            if seen[val] then
                error("Circular reference in JSON encoding")
            end
            seen[val] = true
            
            local pad = pretty and string.rep(indent, depth + 1) or ""
            local closePad = pretty and string.rep(indent, depth) or ""
            local sep = pretty and "\n" or ""
            local colon = pretty and ": " or ":"
            local comma = pretty and ("," .. sep) or ","
            
            -- ตรวจสอบว่าเป็น array หรือ object
            local isArray = true
            local n = 0
            for k in pairs(val) do
                n = n + 1
                if type(k) ~= "number" or k ~= math.floor(k) or k < 1 then
                    isArray = false
                    break
                end
            end
            isArray = isArray and n == #val
            
            local result
            if isArray then
                local parts = {}
                for _, v in ipairs(val) do
                    table.insert(parts, pad .. enc(v, depth + 1, seen))
                end
                if #parts == 0 then
                    result = "[]"
                elseif pretty then
                    result = "[\n" .. table.concat(parts, comma) .. sep .. closePad .. "]"
                else
                    result = "[" .. table.concat(parts, comma) .. "]"
                end
            else
                local parts = {}
                local keys = {}
                for k in pairs(val) do
                    if type(k) == "string" or type(k) == "number" then
                        table.insert(keys, k)
                    end
                end
                table.sort(keys, function(a, b) return tostring(a) < tostring(b) end)
                
                for _, k in ipairs(keys) do
                    local keyStr = '"' .. escapeString(tostring(k)) .. '"'
                    local valStr = enc(val[k], depth + 1, seen)
                    table.insert(parts, pad .. keyStr .. colon .. valStr)
                end
                
                if #parts == 0 then
                    result = "{}"
                elseif pretty then
                    result = "{\n" .. table.concat(parts, comma) .. sep .. closePad .. "}"
                else
                    result = "{" .. table.concat(parts, comma) .. "}"
                end
            end
            
            seen[val] = nil
            return result
        else
            return "null"
        end
    end
    
    return enc(value)
end

-- ทดสอบ JSON encoder
local data = {
    id = 42,
    name = "สวัสดี World",
    scores = {95, 87.5, 92},
    active = true,
    extra = nil,
    config = {
        theme = "dark",
        debug = false,
        maxItems = 100
    }
}

print("=== Compact JSON ===")
print(JSON.encode(data))

print("\n=== Pretty JSON ===")
print(JSON.encode(data, true))
```

---

## 42.4 JSON Decoder

### ตัวอย่างที่ 7: Simple JSON Decoder

```lua
-- json_decoder.lua

local function skipWhitespace(s, i)
    while i <= #s do
        local c = s:sub(i, i)
        if c ~= " " and c ~= "\t" and c ~= "\n" and c ~= "\r" then
            break
        end
        i = i + 1
    end
    return i
end

local function parseString(s, i)
    assert(s:sub(i, i) == '"', "Expected '\"' at position " .. i)
    i = i + 1
    local parts = {}
    
    while i <= #s do
        local c = s:sub(i, i)
        if c == '"' then
            return table.concat(parts), i + 1
        elseif c == '\\' then
            i = i + 1
            local esc = s:sub(i, i)
            local escMap = {
                ['"'] = '"', ['\\'] = '\\', ['/'] = '/',
                ['n'] = '\n', ['r'] = '\r', ['t'] = '\t',
                ['b'] = '\b', ['f'] = '\f'
            }
            if escMap[esc] then
                table.insert(parts, escMap[esc])
                i = i + 1
            elseif esc == 'u' then
                local hex = s:sub(i + 1, i + 4)
                local codepoint = tonumber(hex, 16) or 0
                if codepoint < 128 then
                    table.insert(parts, string.char(codepoint))
                else
                    table.insert(parts, "?")  -- simplified
                end
                i = i + 5
            else
                table.insert(parts, c)
                i = i + 1
            end
        else
            table.insert(parts, c)
            i = i + 1
        end
    end
    error("Unterminated string")
end

local parseValue  -- forward declaration

local function parseArray(s, i)
    assert(s:sub(i, i) == '[')
    i = i + 1
    i = skipWhitespace(s, i)
    
    local result = {}
    if s:sub(i, i) == ']' then
        return result, i + 1
    end
    
    while true do
        i = skipWhitespace(s, i)
        local val
        val, i = parseValue(s, i)
        table.insert(result, val)
        
        i = skipWhitespace(s, i)
        local c = s:sub(i, i)
        if c == ']' then
            return result, i + 1
        elseif c == ',' then
            i = i + 1
        else
            error("Expected ',' or ']' at position " .. i)
        end
    end
end

local function parseObject(s, i)
    assert(s:sub(i, i) == '{')
    i = i + 1
    i = skipWhitespace(s, i)
    
    local result = {}
    if s:sub(i, i) == '}' then
        return result, i + 1
    end
    
    while true do
        i = skipWhitespace(s, i)
        local key
        key, i = parseString(s, i)
        
        i = skipWhitespace(s, i)
        assert(s:sub(i, i) == ':', "Expected ':' at position " .. i)
        i = i + 1
        i = skipWhitespace(s, i)
        
        local val
        val, i = parseValue(s, i)
        result[key] = val
        
        i = skipWhitespace(s, i)
        local c = s:sub(i, i)
        if c == '}' then
            return result, i + 1
        elseif c == ',' then
            i = i + 1
        else
            error("Expected ',' or '}' at position " .. i)
        end
    end
end

parseValue = function(s, i)
    i = skipWhitespace(s, i)
    local c = s:sub(i, i)
    
    if c == '"' then
        return parseString(s, i)
    elseif c == '{' then
        return parseObject(s, i)
    elseif c == '[' then
        return parseArray(s, i)
    elseif c == 't' then
        assert(s:sub(i, i+3) == "true")
        return true, i + 4
    elseif c == 'f' then
        assert(s:sub(i, i+4) == "false")
        return false, i + 5
    elseif c == 'n' then
        assert(s:sub(i, i+3) == "null")
        return nil, i + 4
    elseif c == '-' or c:match("%d") then
        -- Parse number
        local numStr = s:match("^-?%d+%.?%d*[eE]?[+-]?%d*", i)
        local num = tonumber(numStr)
        return num, i + #numStr
    else
        error("Unexpected character '" .. c .. "' at position " .. i)
    end
end

local function jsonDecode(s)
    local val, _ = parseValue(s, 1)
    return val
end

-- ทดสอบ JSON decoder
local jsonStr = [[{
    "name": "John Doe",
    "age": 30,
    "active": true,
    "score": 95.5,
    "tags": ["lua", "json", "programming"],
    "address": {
        "city": "Bangkok",
        "country": "Thailand"
    },
    "nickname": null
}]]

local decoded = jsonDecode(jsonStr)
print("name:", decoded.name)
print("age:", decoded.age)
print("active:", decoded.active)
print("score:", decoded.score)
print("tags[1]:", decoded.tags[1])
print("city:", decoded.address.city)
print("nickname:", tostring(decoded.nickname))
```

---

## 42.5 MessagePack-Inspired Binary Serialization

### ตัวอย่างที่ 8: Binary Serializer พื้นฐาน

```lua
-- binary_serializer.lua
-- สร้าง simple binary format

-- Type tags
local TYPE = {
    NIL     = 0x00,
    FALSE   = 0x01,
    TRUE    = 0x02,
    INT8    = 0x10,
    INT16   = 0x11,
    INT32   = 0x12,
    INT64   = 0x13,
    FLOAT   = 0x20,
    DOUBLE  = 0x21,
    STR8    = 0x30,
    STR16   = 0x31,
    ARR8    = 0x40,
    ARR16   = 0x41,
    MAP8    = 0x50,
    MAP16   = 0x51,
}

-- Helper functions
local function uint8(n)
    return string.char(n & 0xFF)
end

local function uint16BE(n)
    return string.char((n >> 8) & 0xFF, n & 0xFF)
end

local function uint32BE(n)
    return string.char(
        (n >> 24) & 0xFF,
        (n >> 16) & 0xFF,
        (n >>  8) & 0xFF,
        n & 0xFF)
end

local function int8(n)
    if n < 0 then n = n + 256 end
    return string.char(n & 0xFF)
end

-- Double to bytes (IEEE 754)
local function doubleToBytes(n)
    -- ใช้ string.pack ถ้ามี (Lua 5.3+)
    if string.pack then
        return string.pack(">d", n)
    end
    -- Fallback: encode as string
    return string.format("%.17g", n)
end

local function pack(value, seen)
    seen = seen or {}
    local t = type(value)
    
    if t == "nil" then
        return string.char(TYPE.NIL)
    elseif t == "boolean" then
        return string.char(value and TYPE.TRUE or TYPE.FALSE)
    elseif t == "number" then
        if math.type and math.type(value) == "integer" then
            if value >= -128 and value <= 127 then
                return string.char(TYPE.INT8) .. int8(value)
            elseif value >= -32768 and value <= 32767 then
                local n = value < 0 and value + 65536 or value
                return string.char(TYPE.INT16) .. uint16BE(n)
            else
                local n = value < 0 and value + 4294967296 or value
                return string.char(TYPE.INT32) .. uint32BE(n)
            end
        else
            -- Float/Double - store as string for portability
            local s = string.format("%.17g", value)
            return string.char(TYPE.DOUBLE) .. uint8(#s) .. s
        end
    elseif t == "string" then
        local len = #value
        if len <= 255 then
            return string.char(TYPE.STR8) .. uint8(len) .. value
        else
            return string.char(TYPE.STR16) .. uint16BE(len) .. value
        end
    elseif t == "table" then
        if seen[value] then
            -- สำหรับ circular refs ส่ง nil
            return string.char(TYPE.NIL)
        end
        seen[value] = true
        
        -- ตรวจสอบ array
        local isArray = true
        local n = 0
        for k in pairs(value) do
            n = n + 1
            if type(k) ~= "number" or k ~= math.floor(k) or k < 1 then
                isArray = false
                break
            end
        end
        isArray = isArray and n == #value
        
        local parts = {}
        if isArray then
            local count = #value
            if count <= 255 then
                table.insert(parts, string.char(TYPE.ARR8) .. uint8(count))
            else
                table.insert(parts, string.char(TYPE.ARR16) .. uint16BE(count))
            end
            for _, v in ipairs(value) do
                table.insert(parts, pack(v, seen))
            end
        else
            local keys = {}
            for k in pairs(value) do table.insert(keys, k) end
            local count = #keys
            if count <= 255 then
                table.insert(parts, string.char(TYPE.MAP8) .. uint8(count))
            else
                table.insert(parts, string.char(TYPE.MAP16) .. uint16BE(count))
            end
            table.sort(keys, function(a, b) return tostring(a) < tostring(b) end)
            for _, k in ipairs(keys) do
                table.insert(parts, pack(k, seen))
                table.insert(parts, pack(value[k], seen))
            end
        end
        
        seen[value] = nil
        return table.concat(parts)
    end
    
    return string.char(TYPE.NIL)
end

-- ทดสอบ
local data = {
    id = 42,
    name = "Lua Test",
    active = true,
    score = 3.14,
    tags = {"a", "b", "c"},
    config = {debug = false, maxItems = 100}
}

local packed = pack(data)
print("Original size (Lua repr):", #string.format("%q", require and tostring(data) or tostring(data)))
print("Packed size:", #packed)
print("Pack successful: " .. tostring(#packed > 0))
print("First bytes (hex):")
for i = 1, math.min(20, #packed) do
    io.write(string.format("%02X ", packed:byte(i)))
end
print()
```

### ตัวอย่างที่ 9: Binary Format ที่มี Header

```lua
-- binary_with_header.lua

local BinaryFormat = {}
BinaryFormat.__index = BinaryFormat

local MAGIC = "LBF1"  -- Lua Binary Format version 1
local HEADER_SIZE = 16

function BinaryFormat.new()
    return setmetatable({}, BinaryFormat)
end

function BinaryFormat:encode(data)
    -- Serialize เป็น string ก่อน
    local function ser(v)
        local t = type(v)
        if t == "nil" then return "N"
        elseif t == "boolean" then return v and "T" or "F"
        elseif t == "number" then return "n" .. tostring(v) .. ","
        elseif t == "string" then return "s" .. #v .. ":" .. v
        elseif t == "table" then
            local parts = {}
            for k, val in pairs(v) do
                table.insert(parts, ser(k) .. "=" .. ser(val))
            end
            return "{" .. table.concat(parts, ";") .. "}"
        end
        return "N"
    end
    
    local payload = ser(data)
    
    -- สร้าง header
    local timestamp = os.time()
    local checksum = 0
    for i = 1, #payload do
        checksum = (checksum + payload:byte(i)) % 65536
    end
    
    -- Header format: MAGIC(4) + timestamp(4) + payloadLen(4) + checksum(2) + flags(2)
    if string.pack then
        local header = MAGIC .. string.pack(">I4I4I2I2", timestamp, #payload, checksum, 0)
        return header .. payload
    else
        -- Manual packing
        local header = MAGIC
        header = header .. string.char(
            (timestamp >> 24) & 0xFF,
            (timestamp >> 16) & 0xFF,
            (timestamp >> 8) & 0xFF,
            timestamp & 0xFF)
        local plen = #payload
        header = header .. string.char(
            (plen >> 24) & 0xFF,
            (plen >> 16) & 0xFF,
            (plen >> 8) & 0xFF,
            plen & 0xFF)
        header = header .. string.char((checksum >> 8) & 0xFF, checksum & 0xFF)
        header = header .. string.char(0, 0)  -- flags
        return header .. payload
    end
end

function BinaryFormat:decode(data)
    -- ตรวจสอบ header
    if #data < HEADER_SIZE then
        return nil, "Data too short"
    end
    
    if data:sub(1, 4) ~= MAGIC then
        return nil, "Invalid magic bytes"
    end
    
    local payload = data:sub(HEADER_SIZE + 1)
    
    -- Parse payload
    local function unser(s, pos)
        pos = pos or 1
        if pos > #s then return nil, pos end
        
        local tag = s:sub(pos, pos)
        pos = pos + 1
        
        if tag == "N" then
            return nil, pos
        elseif tag == "T" then
            return true, pos
        elseif tag == "F" then
            return false, pos
        elseif tag == "n" then
            local numEnd = s:find(",", pos, true)
            if not numEnd then return nil, pos end
            local num = tonumber(s:sub(pos, numEnd - 1))
            return num, numEnd + 1
        elseif tag == "s" then
            local colonPos = s:find(":", pos, true)
            if not colonPos then return nil, pos end
            local len = tonumber(s:sub(pos, colonPos - 1))
            if not len then return nil, pos end
            local str = s:sub(colonPos + 1, colonPos + len)
            return str, colonPos + len + 1
        elseif tag == "{" then
            local tbl = {}
            while s:sub(pos, pos) ~= "}" and pos <= #s do
                local k, newPos = unser(s, pos)
                if s:sub(newPos, newPos) ~= "=" then break end
                newPos = newPos + 1
                local v
                v, newPos = unser(s, newPos)
                tbl[k] = v
                if s:sub(newPos, newPos) == ";" then newPos = newPos + 1 end
                pos = newPos
            end
            return tbl, pos + 1
        end
        return nil, pos
    end
    
    local result, _ = unser(payload, 1)
    return result
end

-- ทดสอบ
local bf = BinaryFormat.new()
local original = {
    user = "alice",
    level = 10,
    active = true,
    scores = {100, 200, 150}
}

local encoded = bf:encode(original)
print("Encoded size:", #encoded, "bytes")
print("Magic bytes:", encoded:sub(1, 4))

local decoded, err = bf:decode(encoded)
if err then
    print("Error:", err)
else
    print("user:", decoded.user)
    print("level:", decoded.level)
    print("active:", decoded.active)
    if decoded.scores then
        print("scores:", table.concat(decoded.scores, ", "))
    end
end
```

---

## 42.6 Object Versioning

### ตัวอย่างที่ 10: Versioned Serialization

```lua
-- versioned_serializer.lua

local VersionedSerializer = {}
VersionedSerializer.__index = VersionedSerializer

function VersionedSerializer.new()
    local self = setmetatable({}, VersionedSerializer)
    self.schemas = {}  -- version -> {encode, decode}
    self.currentVersion = 1
    return self
end

function VersionedSerializer:registerSchema(version, schema)
    self.schemas[version] = schema
    if version > self.currentVersion then
        self.currentVersion = version
    end
end

function VersionedSerializer:encode(data)
    local schema = self.schemas[self.currentVersion]
    if not schema then
        error("No schema for version " .. self.currentVersion)
    end
    
    local encoded = schema.encode(data)
    -- เพิ่ม version prefix
    return string.format("v%d:%s", self.currentVersion, encoded)
end

function VersionedSerializer:decode(str)
    -- Parse version
    local version, payload = str:match("^v(%d+):(.+)$")
    if not version then
        return nil, "Invalid format: missing version"
    end
    version = tonumber(version)
    
    local schema = self.schemas[version]
    if not schema then
        return nil, "Unknown schema version: " .. version
    end
    
    -- Migrate if needed
    local data, err = schema.decode(payload)
    if err then return nil, err end
    
    -- Migrate from old version to current
    for v = version, self.currentVersion - 1 do
        local nextSchema = self.schemas[v + 1]
        if nextSchema and nextSchema.migrate then
            data = nextSchema.migrate(data)
        end
    end
    
    return data
end

-- ทดสอบ versioned serialization

-- Version 1: Simple format
local v1_encode = function(data)
    return string.format("name=%s,age=%d", data.name, data.age)
end
local v1_decode = function(s)
    local name, age = s:match("name=([^,]+),age=(%d+)")
    return {name = name, age = tonumber(age)}
end

-- Version 2: เพิ่ม email field
local v2_encode = function(data)
    return string.format("name=%s,age=%d,email=%s",
        data.name, data.age, data.email or "")
end
local v2_decode = function(s)
    local name, age, email = s:match("name=([^,]+),age=(%d+),email=([^,]*)")
    return {name = name, age = tonumber(age), email = email}
end
local v2_migrate = function(v1data)
    -- migrate from v1 to v2: add default email
    v1data.email = v1data.email or "unknown@example.com"
    return v1data
end

-- Version 3: เพิ่ม active field + เปลี่ยน format
local v3_encode = function(data)
    return string.format("n=%s;a=%d;e=%s;x=%s",
        data.name, data.age, data.email or "", tostring(data.active ~= false))
end
local v3_decode = function(s)
    local name, age, email, active = s:match("n=([^;]+);a=(%d+);e=([^;]*);x=(%a+)")
    return {
        name = name, age = tonumber(age),
        email = email, active = active == "true"
    }
end
local v3_migrate = function(v2data)
    v2data.active = true  -- default
    return v2data
end

local ser = VersionedSerializer.new()
ser:registerSchema(1, {encode = v1_encode, decode = v1_decode})
ser:registerSchema(2, {encode = v2_encode, decode = v2_decode, migrate = v2_migrate})
ser:registerSchema(3, {encode = v3_encode, decode = v3_decode, migrate = v3_migrate})

-- Encode with latest version
local userData = {name = "Alice", age = 30, email = "alice@example.com", active = true}
local encoded = ser:encode(userData)
print("Encoded (v3):", encoded)

-- Decode current version
local decoded, err = ser:decode(encoded)
print("Decoded:", decoded.name, decoded.age, decoded.email, decoded.active)

-- Simulate reading old v1 data
local oldV1Data = "v1:name=Bob,age=25"
local migratedData, migErr = ser:decode(oldV1Data)
if migErr then
    print("Migration error:", migErr)
else
    print("\nMigrated from v1:", migratedData.name, migratedData.age, 
          migratedData.email, migratedData.active)
end
```

---

## 42.7 Schema Evolution

### ตัวอย่างที่ 11: Schema with Defaults

```lua
-- schema_evolution.lua

local Schema = {}
Schema.__index = Schema

function Schema.new(version, fields)
    local self = setmetatable({}, Schema)
    self.version = version
    self.fields = fields  -- {name, type, default, required}
    return self
end

function Schema:validate(data)
    local errors = {}
    local result = {}
    
    for _, field in ipairs(self.fields) do
        local val = data[field.name]
        
        if val == nil then
            if field.required then
                table.insert(errors, "Missing required field: " .. field.name)
            elseif field.default ~= nil then
                result[field.name] = field.default
            end
        elseif type(val) ~= field.type then
            -- Type coercion
            if field.type == "number" and type(val) == "string" then
                result[field.name] = tonumber(val) or field.default
            elseif field.type == "string" then
                result[field.name] = tostring(val)
            elseif field.type == "boolean" then
                result[field.name] = val ~= false and val ~= nil
            else
                table.insert(errors, string.format(
                    "Field '%s': expected %s, got %s",
                    field.name, field.type, type(val)))
            end
        else
            -- Validation
            if field.min and val < field.min then
                table.insert(errors, string.format(
                    "Field '%s': value %s is less than minimum %s",
                    field.name, val, field.min))
            elseif field.max and val > field.max then
                table.insert(errors, string.format(
                    "Field '%s': value %s exceeds maximum %s",
                    field.name, val, field.max))
            else
                result[field.name] = val
            end
        end
    end
    
    if #errors > 0 then
        return nil, errors
    end
    return result
end

-- ทดสอบ schema validation
local userSchema = Schema.new(1, {
    {name = "id",       type = "number",  required = true},
    {name = "name",     type = "string",  required = true},
    {name = "age",      type = "number",  required = false, default = 18, min = 0, max = 150},
    {name = "email",    type = "string",  required = false, default = ""},
    {name = "active",   type = "boolean", required = false, default = true},
    {name = "score",    type = "number",  required = false, default = 0.0, min = 0.0, max = 100.0},
})

-- Valid data
local valid, errs = userSchema:validate({id = 1, name = "Alice", age = 25, score = 95.5})
if errs then
    print("Validation errors:", table.concat(errs, "; "))
else
    print("Valid user:", valid.id, valid.name, valid.age, valid.active, valid.score)
end

-- Missing defaults
local withDefaults, errs2 = userSchema:validate({id = 2, name = "Bob"})
if not errs2 then
    print("With defaults:", withDefaults.id, withDefaults.name, 
          withDefaults.age, withDefaults.active)
end

-- Invalid
local invalid, errs3 = userSchema:validate({name = "Missing ID"})
if errs3 then
    print("Validation failed:")
    for _, e in ipairs(errs3) do print("  -", e) end
end

-- Out of range
local outOfRange, errs4 = userSchema:validate({id = 3, name = "Charlie", score = 150})
if errs4 then
    print("Range error:")
    for _, e in ipairs(errs4) do print("  -", e) end
end
```

---

## 42.8 Checksums และ Validation

### ตัวอย่างที่ 12: CRC32 Implementation

```lua
-- crc32.lua

-- สร้าง CRC32 lookup table
local function makeCRC32Table()
    local table = {}
    for i = 0, 255 do
        local crc = i
        for _ = 1, 8 do
            if crc & 1 == 1 then
                crc = (crc >> 1) ~ 0xEDB88320
            else
                crc = crc >> 1
            end
        end
        table[i] = crc
    end
    return table
end

local crcTable = makeCRC32Table()

local function crc32(data, prev)
    local crc = prev and (~prev & 0xFFFFFFFF) or 0xFFFFFFFF
    for i = 1, #data do
        local b = data:byte(i)
        crc = (crc >> 8) ~ crcTable[(crc ~ b) & 0xFF]
    end
    return ~crc & 0xFFFFFFFF
end

-- ทดสอบ CRC32
local testData = "Hello, World!"
local checksum = crc32(testData)
print(string.format("CRC32('%s') = 0x%08X", testData, checksum))

-- ตรวจสอบ integrity
local function withChecksum(data)
    local checksum = crc32(data)
    return data .. string.format("|%08X", checksum)
end

local function verifyChecksum(dataWithChecksum)
    local data, storedCRC = dataWithChecksum:match("^(.+)|(%x%x%x%x%x%x%x%x)$")
    if not data then return nil, "Invalid format" end
    
    local computed = crc32(data)
    local expected = tonumber(storedCRC, 16)
    
    if computed ~= expected then
        return nil, string.format("Checksum mismatch: expected 0x%08X, got 0x%08X",
            expected, computed)
    end
    return data
end

-- ทดสอบ
local message = "{"id":42,"name":"Alice","balance":1000.00}"
local protected = withChecksum(message)
print("Protected:", protected)

local verified, err = verifyChecksum(protected)
print("Verified:", verified ~= nil, verified and "Data intact" or err)

-- Tampered data
local tampered = protected:gsub("1000", "9999")
local _, tampErr = verifyChecksum(tampered)
print("Tampered:", tampErr)
```

### ตัวอย่างที่ 13: Adler32 (เร็วกว่า CRC32)

```lua
-- adler32.lua
-- Adler-32 checksum (ใช้ใน zlib)

local MOD_ADLER = 65521

local function adler32(data, prev)
    local a, b
    if prev then
        a = prev & 0xFFFF
        b = (prev >> 16) & 0xFFFF
    else
        a, b = 1, 0
    end
    
    for i = 1, #data do
        a = (a + data:byte(i)) % MOD_ADLER
        b = (b + a) % MOD_ADLER
    end
    
    return (b << 16) | a
end

-- Simple hash สำหรับ string
local function fnv1a(data)
    local hash = 2166136261  -- FNV offset basis
    for i = 1, #data do
        hash = hash ~ data:byte(i)
        hash = (hash * 16777619) & 0xFFFFFFFF
    end
    return hash
end

-- ทดสอบ
local strings = {
    "Hello",
    "World",
    "Lua serialization",
    "1234567890",
    "สวัสดี"
}

print("Adler32 checksums:")
for _, s in ipairs(strings) do
    print(string.format("  %-25s -> 0x%08X", s, adler32(s)))
end

print("\nFNV1a hashes:")
for _, s in ipairs(strings) do
    print(string.format("  %-25s -> 0x%08X", s, fnv1a(s)))
end
```

---

## 42.9 Compression หลัง Serialization

### ตัวอย่างที่ 14: RLE (Run-Length Encoding)

```lua
-- rle_compression.lua
-- Simple Run-Length Encoding สำหรับ compress ข้อมูลที่มีข้อมูลซ้ำ

local function rleEncode(data)
    if #data == 0 then return "" end
    
    local result = {}
    local i = 1
    
    while i <= #data do
        local char = data:sub(i, i)
        local count = 1
        
        while i + count <= #data and 
              data:sub(i + count, i + count) == char and
              count < 255 do
            count = count + 1
        end
        
        if count > 3 or char:byte() < 32 then
            -- Encode as: escape(1) + count(1) + char(1)
            table.insert(result, string.char(0xFF, count) .. char)
        else
            -- Store literally
            table.insert(result, char:rep(count))
        end
        
        i = i + count
    end
    
    return table.concat(result)
end

local function rleDecode(data)
    local result = {}
    local i = 1
    
    while i <= #data do
        local b = data:byte(i)
        if b == 0xFF and i + 2 <= #data then
            local count = data:byte(i + 1)
            local char = data:sub(i + 2, i + 2)
            table.insert(result, char:rep(count))
            i = i + 3
        else
            table.insert(result, data:sub(i, i))
            i = i + 1
        end
    end
    
    return table.concat(result)
end

-- ทดสอบ
local testData = "AAAAAABBBCCCCCCCDDEEEEEEEEEEEF"
print("Original:", testData, "(" .. #testData .. " bytes)")
local encoded = rleEncode(testData)
print("Encoded:  hex=" .. #encoded .. " bytes")
local decoded = rleDecode(encoded)
print("Decoded:", decoded)
print("Lossless:", testData == decoded)

-- Compress JSON
local jsonData = '{"status":"ok","data":[{"id":1,"active":true},{"id":2,"active":true},{"id":3,"active":false}]}'
print("\nJSON:", #jsonData, "bytes")
local compressedJSON = rleEncode(jsonData)
print("RLE compressed:", #compressedJSON, "bytes (RLE not ideal for JSON)")
```

### ตัวอย่างที่ 15: LZ77-Inspired Compression

```lua
-- simple_lz77.lua
-- Simplified LZ77-based compression

local function lzCompress(data)
    local window = 255   -- lookback window size
    local maxMatch = 255  -- max match length
    local result = {}
    local i = 1
    
    while i <= #data do
        local bestLen = 0
        local bestOff = 0
        
        -- ค้นหา match ใน window
        local windowStart = math.max(1, i - window)
        for j = windowStart, i - 1 do
            local len = 0
            while len < maxMatch and
                  i + len <= #data and
                  data:sub(j + len, j + len) == data:sub(i + len, i + len) do
                len = len + 1
            end
            if len > bestLen then
                bestLen = len
                bestOff = i - j
            end
        end
        
        if bestLen >= 4 then
            -- Encode as reference: flag(1) + offset(1) + length(1)
            table.insert(result, string.char(0xFF, bestOff, bestLen))
            i = i + bestLen
        else
            -- Literal
            local c = data:sub(i, i)
            if c:byte() == 0xFF then
                table.insert(result, string.char(0xFF, 0, 1) .. c)
            else
                table.insert(result, c)
            end
            i = i + 1
        end
    end
    
    return table.concat(result)
end

local function lzDecompress(data)
    local result = {}
    local i = 1
    
    while i <= #data do
        local b = data:byte(i)
        if b == 0xFF and i + 2 <= #data then
            local off = data:byte(i + 1)
            local len = data:byte(i + 2)
            if off == 0 then
                -- Escaped 0xFF literal
                table.insert(result, data:sub(i + 3, i + 2 + len))
                i = i + 3 + len
            else
                -- Back reference
                local resultStr = table.concat(result)
                local start = #resultStr - off + 1
                for k = 0, len - 1 do
                    local charIdx = start + (k % off)
                    if charIdx >= 1 and charIdx <= #resultStr then
                        table.insert(result, resultStr:sub(charIdx, charIdx))
                    end
                end
                i = i + 3
            end
        else
            table.insert(result, data:sub(i, i))
            i = i + 1
        end
    end
    
    return table.concat(result)
end

-- ทดสอบ
local testStrings = {
    "ABABABABABABABAB",
    "the the the the the",
    "aaaaaaaaaaaaaaaa",
    "Hello World Hello World Hello"
}

print("LZ Compression test:")
for _, s in ipairs(testStrings) do
    local compressed = lzCompress(s)
    local decompressed = lzDecompress(compressed)
    local ratio = #compressed / #s * 100
    print(string.format("  '%s'", s:sub(1, 30)))
    print(string.format("    Original: %d | Compressed: %d | Ratio: %.1f%% | OK: %s",
        #s, #compressed, ratio, tostring(s == decompressed)))
end
```

---

## 42.10 Serialization Formats เปรียบเทียบ

### ตัวอย่างที่ 16: Benchmark Comparison

```lua
-- serialization_benchmark.lua

local function makeLuaSerializer()
    local function ser(v, seen)
        seen = seen or {}
        local t = type(v)
        if t == "nil" then return "nil"
        elseif t == "boolean" then return tostring(v)
        elseif t == "number" then return tostring(v)
        elseif t == "string" then return string.format("%q", v)
        elseif t == "table" then
            if seen[v] then return "{}" end
            seen[v] = true
            local parts = {}
            for k, val in pairs(v) do
                parts[#parts+1] = "[" .. ser(k, seen) .. "]=" .. ser(val, seen)
            end
            seen[v] = nil
            return "{" .. table.concat(parts, ",") .. "}"
        end
        return "nil"
    end
    return ser
end

local function makeJSONSerializer()
    local function ser(v, seen)
        seen = seen or {}
        local t = type(v)
        if t == "nil" then return "null"
        elseif t == "boolean" then return tostring(v)
        elseif t == "number" then return tostring(v)
        elseif t == "string" then return '"' .. v:gsub('"', '\\"'):gsub('\n', '\\n') .. '"'
        elseif t == "table" then
            if seen[v] then return "null" end
            seen[v] = true
            -- Check array
            local isArr = #v > 0
            local parts = {}
            if isArr then
                for _, val in ipairs(v) do parts[#parts+1] = ser(val, seen) end
                seen[v] = nil
                return "[" .. table.concat(parts, ",") .. "]"
            else
                for k, val in pairs(v) do
                    parts[#parts+1] = '"' .. tostring(k) .. '":' .. ser(val, seen)
                end
                seen[v] = nil
                return "{" .. table.concat(parts, ",") .. "}"
            end
        end
        return "null"
    end
    return ser
end

-- ข้อมูลทดสอบ
local testData = {}
for i = 1, 100 do
    testData[i] = {
        id = i,
        name = "user_" .. i,
        active = i % 2 == 0,
        score = i * 3.14,
        tags = {"tag1", "tag2", "tag3"}
    }
end

local luaSerializer = makeLuaSerializer()
local jsonSerializer = makeJSONSerializer()

-- Benchmark
local ITERS = 1000

local start = os.clock()
for _ = 1, ITERS do
    luaSerializer(testData)
end
local luaTime = os.clock() - start

start = os.clock()
for _ = 1, ITERS do
    jsonSerializer(testData)
end
local jsonTime = os.clock() - start

local luaOutput = luaSerializer(testData)
local jsonOutput = jsonSerializer(testData)

print("=== Serialization Benchmark ===")
print(string.format("Data: %d records", #testData))
print(string.format("Iterations: %d", ITERS))
print()
print(string.format("Lua format:  %.4fs | %d bytes per record", luaTime, #luaOutput // #testData))
print(string.format("JSON format: %.4fs | %d bytes per record", jsonTime, #jsonOutput // #testData))
print(string.format("Lua/JSON speed ratio: %.2f", luaTime / math.max(jsonTime, 0.0001)))
```

---

## 42.11 Protocol Buffers Concepts

### ตัวอย่างที่ 17: Varint Encoding (Proto concept)

```lua
-- varint.lua
-- Variable-length integer encoding (เหมือนที่ Protocol Buffers ใช้)

-- Encode unsigned integer เป็น varint
local function encodeVarint(n)
    local bytes = {}
    repeat
        local b = n & 0x7F
        n = n >> 7
        if n > 0 then b = b | 0x80 end
        table.insert(bytes, string.char(b))
    until n == 0
    return table.concat(bytes)
end

-- Decode varint
local function decodeVarint(data, pos)
    pos = pos or 1
    local result = 0
    local shift = 0
    
    repeat
        if pos > #data then
            error("Truncated varint")
        end
        local b = data:byte(pos)
        pos = pos + 1
        result = result | ((b & 0x7F) << shift)
        shift = shift + 7
    until (data:byte(pos - 1) & 0x80) == 0
    
    return result, pos
end

-- ZigZag encoding สำหรับ signed integers
local function zigzagEncode(n)
    if n >= 0 then
        return n * 2
    else
        return (-n * 2) - 1
    end
end

local function zigzagDecode(n)
    if n % 2 == 0 then
        return n // 2
    else
        return -(n // 2) - 1
    end
end

-- ทดสอบ varint
print("Varint encoding:")
local numbers = {0, 1, 127, 128, 255, 300, 16384, 2097151, 268435455}
for _, n in ipairs(numbers) do
    local encoded = encodeVarint(n)
    local decoded, _ = decodeVarint(encoded)
    local bytes = {}
    for i = 1, #encoded do bytes[i] = string.format("%02X", encoded:byte(i)) end
    print(string.format("  %10d -> [%s] (%d byte%s) -> %d %s",
        n, table.concat(bytes, " "), #encoded,
        #encoded > 1 and "s" or " ",
        decoded,
        decoded == n and "OK" or "FAIL"))
end

print("\nZigzag encoding:")
local signed = {0, -1, 1, -2, 2, -100, 100, -1000, 1000}
for _, n in ipairs(signed) do
    local encoded = zigzagEncode(n)
    local decoded = zigzagDecode(encoded)
    print(string.format("  %6d -> %d -> %d %s",
        n, encoded, decoded, decoded == n and "OK" or "FAIL"))
end
```

### ตัวอย่างที่ 18: Simple Proto-like Message Encoding

```lua
-- proto_simple.lua
-- สร้าง simple binary encoding คล้าย Protocol Buffers

-- Field types
local WIRE_TYPE = {
    VARINT   = 0,
    FIXED64  = 1,
    LEN      = 2,  -- length-delimited (string, bytes, nested message)
    FIXED32  = 5
}

local function encodeVarint(n)
    local bytes = {}
    repeat
        local b = n & 0x7F
        n = n >> 7
        if n > 0 then b = b | 0x80 end
        table.insert(bytes, string.char(b))
    until n == 0
    return table.concat(bytes)
end

local function encodeField(fieldNum, wireType, value)
    local tag = (fieldNum << 3) | wireType
    local tagBytes = encodeVarint(tag)
    
    if wireType == WIRE_TYPE.VARINT then
        return tagBytes .. encodeVarint(value)
    elseif wireType == WIRE_TYPE.LEN then
        local lenBytes = encodeVarint(#value)
        return tagBytes .. lenBytes .. value
    end
    return ""
end

-- Message encoder
local function encodeMessage(fields)
    local parts = {}
    for _, field in ipairs(fields) do
        local num, wireType, val = field[1], field[2], field[3]
        table.insert(parts, encodeField(num, wireType, val))
    end
    return table.concat(parts)
end

-- ทดสอบ proto-like encoding
-- Schema: User { id(1)=int, name(2)=string, active(3)=bool, score(4)=int }
local function encodeUser(user)
    local active = user.active and 1 or 0
    local score = math.floor(user.score * 100)  -- store as cents
    
    return encodeMessage({
        {1, WIRE_TYPE.VARINT, user.id},
        {2, WIRE_TYPE.LEN, user.name},
        {3, WIRE_TYPE.VARINT, active},
        {4, WIRE_TYPE.VARINT, score}
    })
end

local user = {id = 42, name = "Alice Smith", active = true, score = 95.75}
local encoded = encodeUser(user)

print("User encoded:")
print("  id:", user.id)
print("  name:", user.name)
print("  active:", user.active)
print("  score:", user.score)
print()
print("Encoded size:", #encoded, "bytes")
print("Bytes (hex):")
local hexParts = {}
for i = 1, #encoded do
    table.insert(hexParts, string.format("%02X", encoded:byte(i)))
end
print(table.concat(hexParts, " "))

-- เปรียบเทียบกับ JSON
local jsonSize = #string.format('{"id":%d,"name":"%s","active":%s,"score":%s}',
    user.id, user.name, tostring(user.active), user.score)
print(string.format("\nJSON size: %d bytes (binary: %d bytes, %.0f%% smaller)",
    jsonSize, #encoded, (1 - #encoded/jsonSize) * 100))
```

---

## 42.12 ตัวอย่างเพิ่มเติม

### ตัวอย่างที่ 19: Config File Format

```lua
-- config_serializer.lua
-- Simple TOML-like config format

local ConfigSerializer = {}

function ConfigSerializer.encode(config, section)
    local lines = {}
    
    if section then
        table.insert(lines, "[" .. section .. "]")
    end
    
    local function encodeValue(v)
        local t = type(v)
        if t == "string" then
            return '"' .. v:gsub('"', '\\"') .. '"'
        elseif t == "number" then
            return tostring(v)
        elseif t == "boolean" then
            return tostring(v)
        elseif t == "table" then
            -- Inline array
            local parts = {}
            for _, item in ipairs(v) do
                table.insert(parts, encodeValue(item))
            end
            return "[" .. table.concat(parts, ", ") .. "]"
        end
        return tostring(v)
    end
    
    -- Sort keys for deterministic output
    local keys = {}
    for k in pairs(config) do
        if type(config[k]) ~= "table" or #config[k] > 0 then
            table.insert(keys, k)
        end
    end
    table.sort(keys)
    
    for _, k in ipairs(keys) do
        local v = config[k]
        if type(v) ~= "table" or #v > 0 then
            table.insert(lines, k .. " = " .. encodeValue(v))
        end
    end
    
    -- Process sub-sections
    local sectionKeys = {}
    for k in pairs(config) do
        if type(config[k]) == "table" and #config[k] == 0 then
            table.insert(sectionKeys, k)
        end
    end
    table.sort(sectionKeys)
    
    for _, k in ipairs(sectionKeys) do
        table.insert(lines, "")
        local fullSection = section and (section .. "." .. k) or k
        local subLines = ConfigSerializer.encode(config[k], fullSection)
        table.insert(lines, subLines)
    end
    
    return table.concat(lines, "\n")
end

-- ทดสอบ
local config = {
    appName = "MyApp",
    version = "2.0.0",
    debug = false,
    port = 8080,
    allowedHosts = {"localhost", "127.0.0.1"},
    database = {
        host = "db.example.com",
        port = 5432,
        name = "mydb",
        ssl = true
    },
    cache = {
        driver = "redis",
        ttl = 3600,
        maxSize = 1000
    }
}

print(ConfigSerializer.encode(config))
```

### ตัวอย่างที่ 20: CSV Serializer

```lua
-- csv_serializer.lua

local CSV = {}

function CSV.encode(records, headers)
    local lines = {}
    
    -- ถ้าไม่มี headers ให้ใช้ keys จาก record แรก
    if not headers and records[1] then
        headers = {}
        for k in pairs(records[1]) do
            table.insert(headers, k)
        end
        table.sort(headers)
    end
    
    if headers then
        local headerLine = {}
        for _, h in ipairs(headers) do
            table.insert(headerLine, CSV.escapeField(h))
        end
        table.insert(lines, table.concat(headerLine, ","))
    end
    
    for _, record in ipairs(records) do
        local fields = {}
        if headers then
            for _, h in ipairs(headers) do
                table.insert(fields, CSV.escapeField(record[h]))
            end
        else
            for _, v in ipairs(record) do
                table.insert(fields, CSV.escapeField(v))
            end
        end
        table.insert(lines, table.concat(fields, ","))
    end
    
    return table.concat(lines, "\n")
end

function CSV.escapeField(value)
    if value == nil then return "" end
    local s = tostring(value)
    if s:find('[,"\n\r]') then
        return '"' .. s:gsub('"', '""') .. '"'
    end
    return s
end

function CSV.decode(csvStr, hasHeaders)
    local lines = {}
    for line in (csvStr .. "\n"):gmatch("([^\n]*)\n") do
        if line ~= "" then
            table.insert(lines, line)
        end
    end
    
    local function parseLine(line)
        local fields = {}
        local i = 1
        while i <= #line do
            if line:sub(i, i) == '"' then
                -- Quoted field
                i = i + 1
                local parts = {}
                while i <= #line do
                    if line:sub(i, i) == '"' then
                        if line:sub(i + 1, i + 1) == '"' then
                            table.insert(parts, '"')
                            i = i + 2
                        else
                            i = i + 1
                            break
                        end
                    else
                        table.insert(parts, line:sub(i, i))
                        i = i + 1
                    end
                end
                table.insert(fields, table.concat(parts))
                if line:sub(i, i) == ',' then i = i + 1 end
            else
                local field = line:match("^([^,]*)", i)
                table.insert(fields, field or "")
                i = i + (field and #field or 0) + 1
            end
        end
        return fields
    end
    
    if #lines == 0 then return {} end
    
    local startLine = 1
    local headers = nil
    if hasHeaders then
        headers = parseLine(lines[1])
        startLine = 2
    end
    
    local records = {}
    for i = startLine, #lines do
        local fields = parseLine(lines[i])
        if headers then
            local record = {}
            for j, h in ipairs(headers) do
                record[h] = fields[j]
            end
            table.insert(records, record)
        else
            table.insert(records, fields)
        end
    end
    
    return records
end

-- ทดสอบ
local data = {
    {name = "Alice", age = 30, city = "Bangkok", score = 95.5},
    {name = "Bob", age = 25, city = "Chiang Mai", score = 87.0},
    {name = "Charlie, Jr.", age = 35, city = "Phuket", score = 92.3},
    {name = 'Dave "The Man"', age = 28, city = "Pattaya", score = 88.7},
}

print("=== CSV Export ===")
local csvStr = CSV.encode(data, {"name", "age", "city", "score"})
print(csvStr)

print("\n=== CSV Import ===")
local imported = CSV.decode(csvStr, true)
for _, record in ipairs(imported) do
    print(string.format("  name=%s, age=%s, city=%s, score=%s",
        record.name, record.age, record.city, record.score))
end
```

### ตัวอย่างที่ 21: MessagePack-lite

```lua
-- msgpack_lite.lua
-- Simplified MessagePack encoding

local function packUint(n)
    if n <= 0x7F then
        return string.char(n)
    elseif n <= 0xFF then
        return string.char(0xCC, n)
    elseif n <= 0xFFFF then
        return string.char(0xCD, (n >> 8) & 0xFF, n & 0xFF)
    else
        return string.char(0xCE,
            (n >> 24) & 0xFF, (n >> 16) & 0xFF,
            (n >> 8) & 0xFF, n & 0xFF)
    end
end

local function packStr(s)
    local n = #s
    if n <= 31 then
        return string.char(0xA0 | n) .. s
    elseif n <= 0xFF then
        return string.char(0xD9, n) .. s
    else
        return string.char(0xDA, (n >> 8) & 0xFF, n & 0xFF) .. s
    end
end

local function pack(value, seen)
    seen = seen or {}
    local t = type(value)
    
    if t == "nil" then
        return string.char(0xC0)
    elseif t == "boolean" then
        return string.char(value and 0xC3 or 0xC2)
    elseif t == "number" then
        if math.floor(value) == value and value >= 0 and value <= 0xFFFFFFFF then
            return packUint(math.floor(value))
        else
            -- Store as string for simplicity
            local s = string.format("%.17g", value)
            return string.char(0xC1) .. packStr(s)  -- custom float-as-string
        end
    elseif t == "string" then
        return packStr(value)
    elseif t == "table" then
        if seen[value] then return string.char(0xC0) end
        seen[value] = true
        
        -- Check if array
        local isArray = #value > 0
        if isArray then
            local n = #value
            local header
            if n <= 15 then
                header = string.char(0x90 | n)
            else
                header = string.char(0xDC, (n >> 8) & 0xFF, n & 0xFF)
            end
            local parts = {header}
            for _, v in ipairs(value) do
                table.insert(parts, pack(v, seen))
            end
            seen[value] = nil
            return table.concat(parts)
        else
            local keys = {}
            for k in pairs(value) do table.insert(keys, k) end
            local n = #keys
            local header
            if n <= 15 then
                header = string.char(0x80 | n)
            else
                header = string.char(0xDE, (n >> 8) & 0xFF, n & 0xFF)
            end
            local parts = {header}
            table.sort(keys, function(a, b) return tostring(a) < tostring(b) end)
            for _, k in ipairs(keys) do
                table.insert(parts, pack(k, seen))
                table.insert(parts, pack(value[k], seen))
            end
            seen[value] = nil
            return table.concat(parts)
        end
    end
    return string.char(0xC0)
end

-- ทดสอบ
local testData = {
    {name = "Alice", age = 30, scores = {95, 87, 92}},
    {name = "Bob",   age = 25, scores = {88, 91, 79}},
    {name = "Eve",   age = 28, scores = {100, 99, 98}},
}

local packed = pack(testData)
print("Data size comparison:")
print("  Records: " .. #testData)

-- JSON size
local jsonParts = {}
for _, r in ipairs(testData) do
    table.insert(jsonParts, string.format('{"name":"%s","age":%d,"scores":[%s]}',
        r.name, r.age, table.concat(r.scores, ",")))
end
local jsonStr = "[" .. table.concat(jsonParts, ",") .. "]"
print(string.format("  JSON:    %d bytes", #jsonStr))
print(string.format("  MsgPack: %d bytes (%.0f%% of JSON)", 
    #packed, #packed/#jsonStr*100))
```

### ตัวอย่างที่ 22: Streaming Serializer

```lua
-- streaming_serializer.lua
-- Serializer ที่ทำงานแบบ streaming (ไม่ต้องสร้าง string ทั้งหมดในครั้งเดียว)

local StreamSerializer = {}
StreamSerializer.__index = StreamSerializer

function StreamSerializer.new(output)
    local self = setmetatable({}, StreamSerializer)
    self.output = output  -- io.write compatible
    self.depth = 0
    self.needComma = false
    return self
end

function StreamSerializer:_write(s)
    self.output(s)
end

function StreamSerializer:beginObject()
    if self.needComma then self:_write(",") end
    self:_write("{")
    self.needComma = false
    self.depth = self.depth + 1
end

function StreamSerializer:endObject()
    self:_write("}")
    self.depth = self.depth - 1
    self.needComma = true
end

function StreamSerializer:beginArray()
    if self.needComma then self:_write(",") end
    self:_write("[")
    self.needComma = false
end

function StreamSerializer:endArray()
    self:_write("]")
    self.needComma = true
end

function StreamSerializer:key(k)
    if self.needComma then self:_write(",") end
    self:_write('"' .. k:gsub('"', '\\"') .. '":')
    self.needComma = false
end

function StreamSerializer:value(v)
    if self.needComma then self:_write(",") end
    local t = type(v)
    if t == "nil" then self:_write("null")
    elseif t == "boolean" then self:_write(tostring(v))
    elseif t == "number" then self:_write(tostring(v))
    elseif t == "string" then
        self:_write('"' .. v:gsub('"', '\\"'):gsub('\n', '\\n') .. '"')
    end
    self.needComma = true
end

-- ทดสอบ
local output = {}
local ser = StreamSerializer.new(function(s) table.insert(output, s) end)

-- Serialize large dataset เป็น stream
ser:beginArray()
for i = 1, 5 do
    ser:beginObject()
    ser:key("id");    ser:value(i)
    ser:key("name");  ser:value("User " .. i)
    ser:key("active"); ser:value(i % 2 == 0)
    ser:key("score"); ser:value(math.random(60, 100))
    ser:endObject()
end
ser:endArray()

print("Streaming output:")
print(table.concat(output))
```

### ตัวอย่างที่ 23: Differential Serialization

```lua
-- diff_serializer.lua
-- Serialize เฉพาะ fields ที่เปลี่ยนแปลง

local function computeDiff(old, new)
    if type(old) ~= "table" or type(new) ~= "table" then
        if old ~= new then
            return {type = "replace", value = new}
        end
        return nil
    end
    
    local diff = {type = "patch", changes = {}}
    local hasChanges = false
    
    -- ตรวจ new/changed fields
    for k, newVal in pairs(new) do
        local oldVal = old[k]
        if oldVal == nil then
            diff.changes[k] = {op = "add", value = newVal}
            hasChanges = true
        elseif type(newVal) == "table" and type(oldVal) == "table" then
            local subDiff = computeDiff(oldVal, newVal)
            if subDiff then
                diff.changes[k] = subDiff
                hasChanges = true
            end
        elseif oldVal ~= newVal then
            diff.changes[k] = {op = "replace", value = newVal}
            hasChanges = true
        end
    end
    
    -- ตรวจ removed fields
    for k in pairs(old) do
        if new[k] == nil then
            diff.changes[k] = {op = "remove"}
            hasChanges = true
        end
    end
    
    return hasChanges and diff or nil
end

local function applyDiff(original, diff)
    if not diff then return original end
    if diff.type == "replace" then return diff.value end
    if diff.type ~= "patch" then return original end
    
    -- Deep copy
    local result = {}
    for k, v in pairs(original) do
        if type(v) == "table" then
            result[k] = {}
            for ik, iv in pairs(v) do result[k][ik] = iv end
        else
            result[k] = v
        end
    end
    
    for k, change in pairs(diff.changes) do
        if change.op == "add" or change.op == "replace" then
            result[k] = change.value
        elseif change.op == "remove" then
            result[k] = nil
        elseif change.type == "patch" then
            result[k] = applyDiff(result[k] or {}, change)
        end
    end
    
    return result
end

-- ทดสอบ
local v1 = {
    id = 1,
    name = "Alice",
    age = 30,
    settings = {theme = "light", lang = "en"},
    tags = {"user", "member"}
}

local v2 = {
    id = 1,
    name = "Alice Smith",  -- changed
    age = 30,
    email = "alice@example.com",  -- added
    settings = {theme = "dark", lang = "th"},  -- changed
    -- tags removed
}

local diff = computeDiff(v1, v2)

print("Diff:")
if diff and diff.changes then
    for k, change in pairs(diff.changes) do
        if change.op then
            if change.op == "remove" then
                print(string.format("  %s: REMOVED", k))
            elseif change.op == "add" then
                print(string.format("  %s: ADDED = %s", k, tostring(change.value)))
            elseif change.op == "replace" then
                print(string.format("  %s: %s -> %s", k, tostring(v1[k]), tostring(change.value)))
            end
        end
    end
end

local restored = applyDiff(v1, diff)
print("\nAfter applying diff:")
print("  name:", restored.name)
print("  email:", restored.email)
print("  settings.theme:", restored.settings and restored.settings.theme)
```

### ตัวอย่างที่ 24: Lua Table Persistence

```lua
-- table_persistence.lua
-- บันทึกและโหลด Lua tables จากไฟล์

local Persistence = {}

function Persistence.save(filepath, data, varName)
    varName = varName or "data"
    
    local function ser(v, depth)
        depth = depth or 0
        local pad = string.rep("  ", depth)
        local innerPad = string.rep("  ", depth + 1)
        local t = type(v)
        
        if t == "nil" then return "nil"
        elseif t == "boolean" then return tostring(v)
        elseif t == "number" then return tostring(v)
        elseif t == "string" then return string.format("%q", v)
        elseif t == "table" then
            local parts = {}
            -- Array part
            for i, val in ipairs(v) do
                table.insert(parts, innerPad .. ser(val, depth + 1))
            end
            -- Hash part
            local arrayLen = #v
            local keys = {}
            for k in pairs(v) do
                if not (type(k) == "number" and k >= 1 and k <= arrayLen and math.floor(k) == k) then
                    table.insert(keys, k)
                end
            end
            table.sort(keys, function(a, b) return tostring(a) < tostring(b) end)
            for _, k in ipairs(keys) do
                local kStr = type(k) == "string" and k:match("^[%a_][%w_]*$") and k or
                             "[" .. ser(k, 0) .. "]"
                table.insert(parts, innerPad .. kStr .. " = " .. ser(v[k], depth + 1))
            end
            
            if #parts == 0 then return "{}" end
            return "{\n" .. table.concat(parts, ",\n") .. "\n" .. pad .. "}"
        end
        return '"' .. tostring(v) .. '"'
    end
    
    local content = "-- Auto-generated by Lua Persistence\n"
    content = content .. "-- Saved: " .. os.date("%Y-%m-%d %H:%M:%S") .. "\n\n"
    content = content .. "return " .. ser(data) .. "\n"
    
    local file = io.open(filepath, "w")
    if not file then
        return false, "Cannot open file: " .. filepath
    end
    file:write(content)
    file:close()
    return true
end

function Persistence.load(filepath)
    local file = io.open(filepath, "r")
    if not file then
        return nil, "Cannot open file: " .. filepath
    end
    local content = file:read("*all")
    file:close()
    
    local fn, err = load(content)
    if not fn then
        return nil, "Parse error: " .. (err or "unknown")
    end
    
    local ok, result = pcall(fn)
    if not ok then
        return nil, "Execution error: " .. tostring(result)
    end
    
    return result
end

-- ทดสอบ
local testData = {
    version = "1.2.3",
    users = {
        {id = 1, name = "Alice", active = true},
        {id = 2, name = "Bob",   active = false},
    },
    config = {
        debug = false,
        maxConnections = 100,
        allowedIPs = {"127.0.0.1", "192.168.1.0/24"}
    }
}

-- บันทึก
local ok, err = Persistence.save("/tmp/test_persistence.lua", testData)
if ok then
    print("Saved successfully")
    
    -- โหลดกลับ
    local loaded, loadErr = Persistence.load("/tmp/test_persistence.lua")
    if loaded then
        print("Loaded successfully")
        print("version:", loaded.version)
        print("users:", #loaded.users)
        print("config.maxConnections:", loaded.config.maxConnections)
    else
        print("Load error:", loadErr)
    end
else
    print("Save error:", err)
end
```

### ตัวอย่างที่ 25: Compact String Format

```lua
-- compact_format.lua
-- Format ที่ compact สำหรับ keys ที่รู้จักล่วงหน้า (dictionary encoding)

local DictionaryEncoder = {}
DictionaryEncoder.__index = DictionaryEncoder

function DictionaryEncoder.new(dictionary)
    local self = setmetatable({}, DictionaryEncoder)
    self.dict = {}  -- string -> index
    self.revDict = {}  -- index -> string
    for i, word in ipairs(dictionary) do
        self.dict[word] = i
        self.revDict[i] = word
    end
    return self
end

function DictionaryEncoder:encode(value)
    local t = type(value)
    if t == "string" then
        local idx = self.dict[value]
        if idx then
            return string.format("@%d", idx)  -- dictionary reference
        end
        return string.format("%q", value)
    elseif t == "number" then
        return tostring(value)
    elseif t == "boolean" then
        return value and "!" or "?"
    elseif t == "table" then
        local parts = {}
        for k, v in pairs(value) do
            local encodedKey = self:encode(tostring(k))
            local encodedVal = self:encode(v)
            table.insert(parts, encodedKey .. ":" .. encodedVal)
        end
        return "{" .. table.concat(parts, ",") .. "}"
    end
    return "~"  -- nil
end

function DictionaryEncoder:decode(s)
    local i = 1
    
    local function parseValue()
        if i > #s then return nil end
        local c = s:sub(i, i)
        
        if c == "@" then
            -- Dictionary reference
            i = i + 1
            local numStr = s:match("^(%d+)", i)
            i = i + #numStr
            return self.revDict[tonumber(numStr)]
        elseif c == "!" then
            i = i + 1
            return true
        elseif c == "?" then
            i = i + 1
            return false
        elseif c == "~" then
            i = i + 1
            return nil
        elseif c == '"' then
            -- Quoted string
            local str = s:match('^"(.-)"', i)
            i = i + #str + 2
            return str
        elseif c == "{" then
            i = i + 1
            local result = {}
            while i <= #s and s:sub(i, i) ~= "}" do
                local key = parseValue()
                i = i + 1  -- skip ":"
                local val = parseValue()
                result[key] = val
                if s:sub(i, i) == "," then i = i + 1 end
            end
            i = i + 1  -- skip "}"
            return result
        else
            -- Number
            local numStr = s:match("^(-?%d+%.?%d*)", i)
            if numStr then
                i = i + #numStr
                return tonumber(numStr)
            end
        end
        return nil
    end
    
    return parseValue()
end

-- ทดสอบ dictionary encoding
local dict = {
    "status", "active", "inactive", "pending",
    "userId", "name", "email", "role",
    "admin", "user", "moderator",
    "true", "false", "null"
}

local enc = DictionaryEncoder.new(dict)

local data = {
    status = "active",
    userId = 42,
    name = "Alice",
    role = "admin",
    active = true
}

local encoded = enc:encode(data)
print("Dictionary-encoded: " .. encoded)
print("Size: " .. #encoded .. " chars")

-- Regular JSON for comparison
local function toSimpleJSON(t)
    local parts = {}
    for k, v in pairs(t) do
        local vStr = type(v) == "string" and ('"'..v..'"') or tostring(v)
        table.insert(parts, '"'..k..'":' .. vStr)
    end
    return "{" .. table.concat(parts, ",") .. "}"
end

local json = toSimpleJSON(data)
print("Regular JSON:        " .. json)
print("JSON size: " .. #json .. " chars")
print(string.format("Compression ratio: %.1f%%", #encoded/#json*100))
```

---

## 42.13 Serpent Library Concepts

### ตัวอย่างที่ 26: Serpent-like Pretty Printer

```lua
-- serpent_like.lua
-- Reimplementation ของ concepts จาก serpent library

local function serpent(value, options)
    options = options or {}
    local indent = options.indent or "  "
    local comment = options.comment ~= false
    local sortKeys = options.sortKeys ~= false
    local nocode = options.nocode or false
    local maxlevel = options.maxlevel or math.huge
    
    local function isArray(t)
        if type(t) ~= "table" then return false end
        local n = #t
        if n == 0 then return false end
        local count = 0
        for _ in pairs(t) do count = count + 1 end
        return count == n
    end
    
    local function ser(val, depth, seen)
        seen = seen or {}
        depth = depth or 0
        
        if depth >= maxlevel then
            return '"[max depth]"'
        end
        
        local t = type(val)
        
        if t == "nil" then return "nil"
        elseif t == "boolean" then return tostring(val)
        elseif t == "number" then
            if val ~= val then return "(0/0)"
            elseif val == math.huge then return "math.huge"
            elseif val == -math.huge then return "-math.huge"
            else
                if math.floor(val) == val then return string.format("%d", val)
                else return string.format("%.17g", val) end
            end
        elseif t == "string" then
            if comment and val:len() > 40 then
                return string.format("%q --[[%d chars]]", val, #val)
            end
            return string.format("%q", val)
        elseif t == "table" then
            if seen[val] then return '"--[[cyclic]]"' end
            seen[val] = true
            
            local pad = string.rep(indent, depth)
            local inner = string.rep(indent, depth + 1)
            
            local parts = {}
            
            if isArray(val) then
                for i, v in ipairs(val) do
                    if comment then
                        table.insert(parts, inner .. ser(v, depth + 1, seen) ..
                            " --[[" .. i .. "]]")
                    else
                        table.insert(parts, inner .. ser(v, depth + 1, seen))
                    end
                end
            else
                local keys = {}
                for k in pairs(val) do table.insert(keys, k) end
                if sortKeys then
                    table.sort(keys, function(a, b) return tostring(a) < tostring(b) end)
                end
                
                for _, k in ipairs(keys) do
                    local kStr
                    if type(k) == "string" and k:match("^[%a_][%w_]*$") then
                        kStr = k
                    else
                        kStr = "[" .. ser(k, 0, seen) .. "]"
                    end
                    table.insert(parts, inner .. kStr .. " = " ..
                        ser(val[k], depth + 1, seen))
                end
            end
            
            seen[val] = nil
            
            if #parts == 0 then return "{}" end
            if depth == 0 or #parts == 1 then
                return "{\n" .. table.concat(parts, ",\n") .. "\n" .. pad .. "}"
            end
            return "{\n" .. table.concat(parts, ",\n") .. "\n" .. pad .. "}"
        elseif not nocode and t == "function" then
            return '"function"'
        else
            return string.format('"[%s]"', t)
        end
    end
    
    return ser(value)
end

-- ทดสอบ
local data = {
    -- Mixed array and hash
    "element1",
    "element2",
    name = "Test Data",
    nested = {
        a = 1,
        b = {true, false, nil, 3.14},
        c = "short",
        longString = "This is a very long string that should be commented"
    },
    numbers = {1, 2, 3, 4, 5},
    config = {debug = false, timeout = 30}
}

print("=== Default (with comments) ===")
print(serpent(data))

print("\n=== Without comments ===")
print(serpent(data, {comment = false}))
```

### ตัวอย่างที่ 27: Binary Serialization with Types

```lua
-- typed_binary.lua
-- Binary format ที่รองรับ type information

local TYPES = {
    NIL     = 0,
    BOOL    = 1,
    INT     = 2,
    FLOAT   = 3,
    STRING  = 4,
    ARRAY   = 5,
    MAP     = 6,
}

local Writer = {}
Writer.__index = Writer

function Writer.new()
    local self = setmetatable({}, Writer)
    self.buf = {}
    return self
end

function Writer:writeByte(b)
    table.insert(self.buf, string.char(b & 0xFF))
end

function Writer:writeUint32(n)
    self:writeByte((n >> 24) & 0xFF)
    self:writeByte((n >> 16) & 0xFF)
    self:writeByte((n >>  8) & 0xFF)
    self:writeByte(n & 0xFF)
end

function Writer:writeString(s)
    self:writeUint32(#s)
    table.insert(self.buf, s)
end

function Writer:writeValue(value, seen)
    seen = seen or {}
    local t = type(value)
    
    if t == "nil" then
        self:writeByte(TYPES.NIL)
    elseif t == "boolean" then
        self:writeByte(TYPES.BOOL)
        self:writeByte(value and 1 or 0)
    elseif t == "number" then
        if math.floor(value) == value and math.abs(value) <= 2147483647 then
            self:writeByte(TYPES.INT)
            local n = value < 0 and value + 4294967296 or value
            self:writeUint32(n)
        else
            self:writeByte(TYPES.FLOAT)
            self:writeString(string.format("%.17g", value))
        end
    elseif t == "string" then
        self:writeByte(TYPES.STRING)
        self:writeString(value)
    elseif t == "table" then
        if seen[value] then
            self:writeByte(TYPES.NIL)
            return
        end
        seen[value] = true
        
        -- Check array
        if #value > 0 then
            self:writeByte(TYPES.ARRAY)
            self:writeUint32(#value)
            for _, v in ipairs(value) do
                self:writeValue(v, seen)
            end
        else
            local keys = {}
            for k in pairs(value) do table.insert(keys, k) end
            self:writeByte(TYPES.MAP)
            self:writeUint32(#keys)
            table.sort(keys, function(a, b) return tostring(a) < tostring(b) end)
            for _, k in ipairs(keys) do
                self:writeValue(k, seen)
                self:writeValue(value[k], seen)
            end
        end
        seen[value] = nil
    end
end

function Writer:getBytes()
    return table.concat(self.buf)
end

-- ทดสอบ
local testData = {
    id = 42,
    name = "Test",
    active = true,
    score = 3.14,
    tags = {"a", "b", "c"},
    nested = {x = 1, y = 2}
}

local writer = Writer.new()
writer:writeValue(testData)
local bytes = writer:getBytes()

print("Binary serialization:")
print("  Data size:", #bytes, "bytes")
print("  Hex dump (first 40 bytes):")
local hex = {}
for i = 1, math.min(40, #bytes) do
    table.insert(hex, string.format("%02X", bytes:byte(i)))
end
print("  " .. table.concat(hex, " "))
```

### ตัวอย่างที่ 28: INI File Format

```lua
-- ini_serializer.lua

local INI = {}

function INI.encode(data)
    local lines = {}
    
    -- Global keys ก่อน
    local globals = {}
    local sections = {}
    
    for k, v in pairs(data) do
        if type(v) == "table" then
            table.insert(sections, k)
        else
            table.insert(globals, k)
        end
    end
    
    table.sort(globals)
    table.sort(sections)
    
    for _, k in ipairs(globals) do
        table.insert(lines, k .. " = " .. tostring(data[k]))
    end
    
    for _, sectionName in ipairs(sections) do
        if #lines > 0 then table.insert(lines, "") end
        table.insert(lines, "[" .. sectionName .. "]")
        
        local section = data[sectionName]
        local keys = {}
        for k in pairs(section) do table.insert(keys, k) end
        table.sort(keys)
        
        for _, k in ipairs(keys) do
            table.insert(lines, k .. " = " .. tostring(section[k]))
        end
    end
    
    return table.concat(lines, "\n")
end

function INI.decode(iniStr)
    local result = {}
    local currentSection = result
    
    for line in (iniStr .. "\n"):gmatch("([^\n]*)\n") do
        -- Skip comments and empty lines
        line = line:match("^%s*(.-)%s*$")
        if line == "" or line:sub(1, 1) == ";" or line:sub(1, 1) == "#" then
            -- skip
        elseif line:sub(1, 1) == "[" then
            -- Section header
            local sectionName = line:match("^%[(.-)%]$")
            if sectionName then
                result[sectionName] = result[sectionName] or {}
                currentSection = result[sectionName]
            end
        else
            -- Key = Value
            local k, v = line:match("^([^=]+)%s*=%s*(.*)$")
            if k then
                k = k:match("^%s*(.-)%s*$")
                -- Try to convert value
                if v == "true" then v = true
                elseif v == "false" then v = false
                elseif tonumber(v) then v = tonumber(v)
                end
                currentSection[k] = v
            end
        end
    end
    
    return result
end

-- ทดสอบ
local config = {
    appName = "MyApp",
    debug = false,
    database = {
        host = "localhost",
        port = 5432,
        name = "mydb",
        pool = 10
    },
    cache = {
        driver = "redis",
        ttl = 3600
    }
}

print("=== INI Format ===")
local iniStr = INI.encode(config)
print(iniStr)

print("\n=== Decoded ===")
local decoded = INI.decode(iniStr)
print("appName:", decoded.appName)
print("debug:", decoded.debug)
print("database.host:", decoded.database and decoded.database.host)
print("database.port:", decoded.database and decoded.database.port)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Serialization และ Deserialization ใน Lua:

1. **พื้นฐาน Serialization** - แปลง Lua tables เป็น strings และกลับ
2. **Round-trip Serialization** - serialize แล้ว deserialize ให้ได้ข้อมูลเดิม
3. **Circular References** - ตรวจจับและจัดการ circular references
4. **JSON Format** - encode/decode JSON โดยไม่ใช้ library
5. **Binary Serialization** - MessagePack-inspired format
6. **Object Versioning** - จัดการ schema versions
7. **Schema Evolution** - migration ระหว่าง versions
8. **Checksums** - CRC32, Adler32 สำหรับ data integrity
9. **Compression** - RLE, LZ77 หลัง serialization
10. **Protocol Buffers** - varint encoding concepts
11. **Streaming** - serialize ข้อมูลขนาดใหญ่เป็น stream
12. **Config Formats** - TOML-like, INI, CSV
13. **Differential Serialization** - serialize เฉพาะ changes

การเลือก format ที่เหมาะสมขึ้นอยู่กับ use case: JSON สำหรับ interoperability, Binary สำหรับ performance, Lua format สำหรับ config files ที่ต้องการให้มนุษย์อ่านได้
