# บทที่ 30: JSON Processing ใน Lua

## บทนำ

JSON (JavaScript Object Notation) เป็น format การแลกเปลี่ยนข้อมูลที่ได้รับความนิยมสูงสุด ใช้กันทั่วไปใน Web APIs, config files, และการสื่อสารระหว่าง services Lua รองรับ JSON ผ่าน libraries หลายตัว โดยเฉพาะ **dkjson** (pure Lua) และ **cjson** (C extension)

---

## 30.1 JSON Format Overview

```lua
-- โครงสร้าง JSON มีแบบต่างๆ ดังนี้:
-- 1. Object: {"key": value}
-- 2. Array: [value1, value2, ...]
-- 3. String: "text"
-- 4. Number: 42, 3.14, -1
-- 5. Boolean: true, false
-- 6. Null: null

-- ตัวอย่าง JSON
local json_examples = {
    simple_object = '{"name": "Alice", "age": 30}',
    array = '[1, 2, 3, 4, 5]',
    nested = '{"user": {"id": 1, "name": "Bob"}, "scores": [95, 87, 92]}',
    with_null = '{"name": "Charlie", "email": null}',
    mixed = '{"active": true, "count": 0, "ratio": 1.5, "tag": null}',
}

for name, json in pairs(json_examples) do
    print(string.format("[%s]: %s", name, json))
end

print("\nJSON Rules:")
print("- String ต้องใช้ double quotes เท่านั้น (ไม่ใช่ single quotes)")
print("- Key ใน object ต้องเป็น string")
print("- ไม่มี trailing comma")
print("- Null ไม่ใช่ nil (ต้องจัดการแยก)")
print("- Number ไม่มี leading zeros (ยกเว้น 0.xxx)")
```

---

## 30.2 Pure Lua JSON Parser (dkjson-compatible)

เราจะสร้าง JSON parser/encoder แบบ pure Lua เองก่อน จากนั้นดูการใช้ library จริง

```lua
-- Pure Lua JSON Implementation
local JSON = {}

-- ==================== ENCODER ====================

local function encode(val, indent, currentIndent, seen)
    local t = type(val)

    if t == "nil" then
        return "null"
    elseif t == "boolean" then
        return tostring(val)
    elseif t == "number" then
        if val ~= val then return "null" end  -- NaN
        if val == math.huge or val == -math.huge then return "null" end
        if math.type(val) == "integer" then
            return tostring(val)
        else
            -- Format float
            local s = string.format("%.14g", val)
            return s
        end
    elseif t == "string" then
        -- Escape special characters
        local escaped = val:gsub('[\\"/\x00-\x1f]', function(c)
            local special = {
                ['"'] = '\\"',
                ['\\'] = '\\\\',
                ['/'] = '\\/',
                ['\b'] = '\\b',
                ['\f'] = '\\f',
                ['\n'] = '\\n',
                ['\r'] = '\\r',
                ['\t'] = '\\t',
            }
            return special[c] or string.format('\\u%04X', c:byte())
        end)
        return '"' .. escaped .. '"'
    elseif t == "table" then
        -- Check circular reference
        if seen[val] then
            error("circular reference detected")
        end
        seen[val] = true

        local result

        -- Check if array-like (consecutive integer keys from 1)
        local isArray = true
        local maxIdx = 0
        for k, _ in pairs(val) do
            if type(k) ~= "number" or k ~= math.floor(k) or k < 1 then
                isArray = false
                break
            end
            if k > maxIdx then maxIdx = k end
        end
        -- ตรวจสอบว่าไม่มีช่องว่าง
        if isArray then
            for i = 1, maxIdx do
                if val[i] == nil then
                    isArray = false
                    break
                end
            end
        end

        if isArray and maxIdx == #val then
            -- Array
            local parts = {}
            local newIndent = indent and (currentIndent .. indent) or nil
            for i = 1, #val do
                if newIndent then
                    parts[i] = newIndent .. encode(val[i], indent, newIndent, seen)
                else
                    parts[i] = encode(val[i], indent, currentIndent, seen)
                end
            end
            if newIndent and #parts > 0 then
                result = "[\n" .. table.concat(parts, ",\n") .. "\n" .. currentIndent .. "]"
            else
                result = "[" .. table.concat(parts, ",") .. "]"
            end
        else
            -- Object
            local parts = {}
            local keys = {}
            for k in pairs(val) do
                if type(k) == "string" then
                    keys[#keys+1] = k
                end
            end
            table.sort(keys)

            local newIndent = indent and (currentIndent .. indent) or nil
            for _, k in ipairs(keys) do
                local encodedKey = encode(k, nil, "", seen)
                local encodedVal = encode(val[k], indent, newIndent or "", seen)
                if newIndent then
                    parts[#parts+1] = newIndent .. encodedKey .. ": " .. encodedVal
                else
                    parts[#parts+1] = encodedKey .. ":" .. encodedVal
                end
            end

            if newIndent and #parts > 0 then
                result = "{\n" .. table.concat(parts, ",\n") .. "\n" .. currentIndent .. "}"
            else
                result = "{" .. table.concat(parts, ",") .. "}"
            end
        end

        seen[val] = nil
        return result
    else
        error("Cannot encode type: " .. t)
    end
end

function JSON.encode(val, options)
    options = options or {}
    local indent = options.indent
    if indent == true then indent = "  " end
    if type(indent) == "number" then
        indent = string.rep(" ", indent)
    end
    return encode(val, indent, "", {})
end

-- ==================== DECODER ====================

local function skipWhitespace(s, pos)
    while pos <= #s do
        local ch = s:sub(pos, pos)
        if ch == " " or ch == "\t" or ch == "\n" or ch == "\r" then
            pos = pos + 1
        else
            break
        end
    end
    return pos
end

local parseValue  -- forward declaration

local function parseString(s, pos)
    pos = pos + 1  -- skip opening "
    local chars = {}
    while pos <= #s do
        local ch = s:sub(pos, pos)
        if ch == '"' then
            return table.concat(chars), pos + 1
        elseif ch == '\\' then
            pos = pos + 1
            local esc = s:sub(pos, pos)
            if esc == '"' then chars[#chars+1] = '"'
            elseif esc == '\\' then chars[#chars+1] = '\\'
            elseif esc == '/' then chars[#chars+1] = '/'
            elseif esc == 'b' then chars[#chars+1] = '\b'
            elseif esc == 'f' then chars[#chars+1] = '\f'
            elseif esc == 'n' then chars[#chars+1] = '\n'
            elseif esc == 'r' then chars[#chars+1] = '\r'
            elseif esc == 't' then chars[#chars+1] = '\t'
            elseif esc == 'u' then
                local hex = s:sub(pos+1, pos+4)
                local code = tonumber(hex, 16)
                if code then
                    -- Simple UTF-8 encoding for BMP characters
                    if code < 0x80 then
                        chars[#chars+1] = string.char(code)
                    elseif code < 0x800 then
                        chars[#chars+1] = string.char(
                            0xC0 | (code >> 6),
                            0x80 | (code & 0x3F)
                        )
                    else
                        chars[#chars+1] = string.char(
                            0xE0 | (code >> 12),
                            0x80 | ((code >> 6) & 0x3F),
                            0x80 | (code & 0x3F)
                        )
                    end
                    pos = pos + 4
                end
            end
            pos = pos + 1
        else
            chars[#chars+1] = ch
            pos = pos + 1
        end
    end
    error("Unterminated string")
end

local function parseNumber(s, pos)
    local start = pos
    if s:sub(pos, pos) == '-' then pos = pos + 1 end
    if not s:sub(pos, pos):match('%d') then
        error("Invalid number at pos " .. pos)
    end
    while pos <= #s and s:sub(pos, pos):match('%d') do pos = pos + 1 end
    if pos <= #s and s:sub(pos, pos) == '.' then
        pos = pos + 1
        while pos <= #s and s:sub(pos, pos):match('%d') do pos = pos + 1 end
    end
    if pos <= #s and s:sub(pos, pos):match('[eE]') then
        pos = pos + 1
        if pos <= #s and s:sub(pos, pos):match('[%+%-]') then pos = pos + 1 end
        while pos <= #s and s:sub(pos, pos):match('%d') do pos = pos + 1 end
    end
    return tonumber(s:sub(start, pos - 1)), pos
end

local function parseArray(s, pos)
    pos = pos + 1  -- skip [
    local arr = {}
    pos = skipWhitespace(s, pos)

    if pos <= #s and s:sub(pos, pos) == ']' then
        return arr, pos + 1
    end

    while pos <= #s do
        local val
        val, pos = parseValue(s, pos)
        arr[#arr+1] = val
        pos = skipWhitespace(s, pos)

        if pos > #s then error("Unexpected end of array") end
        local ch = s:sub(pos, pos)
        if ch == ']' then return arr, pos + 1
        elseif ch == ',' then
            pos = pos + 1
            pos = skipWhitespace(s, pos)
        else
            error("Expected ',' or ']' in array, got: " .. ch)
        end
    end
    error("Unterminated array")
end

local function parseObject(s, pos)
    pos = pos + 1  -- skip {
    local obj = {}
    pos = skipWhitespace(s, pos)

    if pos <= #s and s:sub(pos, pos) == '}' then
        return obj, pos + 1
    end

    while pos <= #s do
        pos = skipWhitespace(s, pos)
        if s:sub(pos, pos) ~= '"' then
            error("Expected string key in object")
        end

        local key
        key, pos = parseString(s, pos)
        pos = skipWhitespace(s, pos)

        if s:sub(pos, pos) ~= ':' then
            error("Expected ':' in object")
        end
        pos = pos + 1
        pos = skipWhitespace(s, pos)

        local val
        val, pos = parseValue(s, pos)
        obj[key] = val

        pos = skipWhitespace(s, pos)
        local ch = s:sub(pos, pos)
        if ch == '}' then return obj, pos + 1
        elseif ch == ',' then
            pos = pos + 1
        else
            error("Expected ',' or '}' in object")
        end
    end
    error("Unterminated object")
end

parseValue = function(s, pos)
    pos = skipWhitespace(s, pos)
    if pos > #s then error("Unexpected end of JSON") end

    local ch = s:sub(pos, pos)

    if ch == '"' then
        return parseString(s, pos)
    elseif ch == '{' then
        return parseObject(s, pos)
    elseif ch == '[' then
        return parseArray(s, pos)
    elseif ch == 't' then
        if s:sub(pos, pos+3) == 'true' then return true, pos + 4 end
        error("Invalid literal at pos " .. pos)
    elseif ch == 'f' then
        if s:sub(pos, pos+4) == 'false' then return false, pos + 5 end
        error("Invalid literal at pos " .. pos)
    elseif ch == 'n' then
        if s:sub(pos, pos+3) == 'null' then return nil, pos + 4 end
        error("Invalid literal at pos " .. pos)
    elseif ch:match('[%-0-9]') then
        return parseNumber(s, pos)
    else
        error("Unexpected character '" .. ch .. "' at pos " .. pos)
    end
end

function JSON.decode(s)
    if type(s) ~= "string" then
        error("JSON.decode expects a string")
    end
    local val, pos = parseValue(s, 1)
    pos = skipWhitespace(s, pos)
    if pos <= #s then
        error("Trailing content after JSON value")
    end
    return val
end

-- Decode พร้อม error handling
function JSON.safeDecode(s)
    local ok, result = pcall(JSON.decode, s)
    if ok then
        return result, nil
    end
    return nil, result
end

print("JSON Parser built successfully!")
```

---

## 30.3 Encoding Lua Tables to JSON

```lua
-- ตัวอย่างการ encode Lua tables ต่างๆ

-- Simple table
local user = {
    name = "สมชาย",
    age = 30,
    active = true,
    score = 9.5,
}
print("Simple object:")
print(JSON.encode(user))

-- Nested table
local company = {
    name = "TechCorp",
    employees = {
        { id = 1, name = "Alice", dept = "IT" },
        { id = 2, name = "Bob",   dept = "HR" },
        { id = 3, name = "Carol", dept = "IT" },
    },
    config = {
        maxEmployees = 100,
        remote = true,
    }
}
print("\nNested object:")
print(JSON.encode(company))

-- Array
local numbers = {1, 2, 3, 4, 5}
print("\nArray:")
print(JSON.encode(numbers))

-- Mixed types
local mixed = {
    string_val = "hello",
    number_val = 42,
    float_val = 3.14159,
    bool_true = true,
    bool_false = false,
    null_val = nil,  -- จะถูก skip โดย encoder
    array_val = {1, 2, 3},
    nested_obj = { a = 1, b = 2 },
}
print("\nMixed types:")
print(JSON.encode(mixed))
```

---

## 30.4 Decoding JSON to Lua Tables

```lua
-- Decode JSON strings ต่างๆ

-- Simple decode
local jsonStr1 = '{"name":"Alice","age":25,"active":true}'
local decoded1 = JSON.decode(jsonStr1)
print("Decoded name:", decoded1.name)    -- Alice
print("Decoded age:", decoded1.age)      -- 25
print("Decoded active:", decoded1.active) -- true

-- Array decode
local jsonArr = '[10, 20, 30, 40, 50]'
local decodedArr = JSON.decode(jsonArr)
for i, v in ipairs(decodedArr) do
    print(string.format("  [%d] = %d", i, v))
end

-- Nested decode
local jsonNested = [[{
    "id": 1,
    "profile": {
        "firstName": "John",
        "lastName": "Doe",
        "age": 30
    },
    "scores": [85, 92, 78, 96],
    "address": {
        "street": "123 Main St",
        "city": "Bangkok",
        "country": "Thailand"
    }
}]]

local decoded2 = JSON.decode(jsonNested)
print("\nDecoded nested:")
print("ID:", decoded2.id)
print("Name:", decoded2.profile.firstName, decoded2.profile.lastName)
print("City:", decoded2.address.city)
print("Scores:", table.concat(decoded2.scores, ", "))

-- Decode array of objects
local jsonList = [[[
    {"name": "Product A", "price": 99.99},
    {"name": "Product B", "price": 149.50},
    {"name": "Product C", "price": 29.99}
]]]

local products = JSON.decode(jsonList)
for _, p in ipairs(products) do
    print(string.format("  %s: $%.2f", p.name, p.price))
end
```

---

## 30.5 Handling null / nil Mapping

```lua
-- JSON null กับ Lua nil มีความแตกต่างกัน
-- null ใน JSON = ค่าที่มีแต่ไม่มีค่า
-- nil ใน Lua = ไม่มีอยู่เลย

-- วิธีจัดการ null: ใช้ sentinel value
local JSON_NULL = setmetatable({}, {
    __tostring = function() return "null" end
})

-- Encode พร้อม null support
local function encodeWithNull(val)
    if val == JSON_NULL then
        return "null"
    end
    return JSON.encode(val)
end

-- Decode พร้อม null -> JSON_NULL
local function decodeWithNull(s)
    -- แทนที่ null ด้วย sentinel ก่อน decode ยาก
    -- วิธีง่าย: ใช้ post-processing
    local result = JSON.safeDecode(s)
    return result
end

-- ตัวอย่าง: API response ที่มี null values
local apiResponse = [[{
    "id": 42,
    "name": "Product X",
    "description": null,
    "price": 299.99,
    "discount": null,
    "tags": ["electronics", "sale"],
    "manufacturer": {
        "name": "ACME Corp",
        "contact": null
    }
}]]

local data, err = JSON.safeDecode(apiResponse)
if err then
    print("Decode error:", err)
else
    print("ID:", data.id)
    print("Name:", data.name)
    print("Description:", tostring(data.description))  -- nil
    print("Price:", data.price)
    print("Discount:", tostring(data.discount))  -- nil
    print("Tags:", table.concat(data.tags, ", "))
    print("Manufacturer:", data.manufacturer.name)
    print("Contact:", tostring(data.manufacturer.contact))  -- nil
end

-- Helper: get with default (handle nil/null)
local function getOrDefault(obj, key, default)
    local val = obj[key]
    if val == nil then return default end
    return val
end

print("\nWith defaults:")
print("Description:", getOrDefault(data, "description", "N/A"))  -- N/A
print("Discount:", getOrDefault(data, "discount", 0))            -- 0
```

---

## 30.6 Pretty Printing

```lua
-- Pretty Print JSON
function JSON.prettyPrint(val, spaces)
    spaces = spaces or 2
    return JSON.encode(val, { indent = spaces })
end

-- ตัวอย่าง pretty print
local config = {
    server = {
        host = "0.0.0.0",
        port = 8080,
        ssl = false,
    },
    database = {
        host = "localhost",
        port = 5432,
        name = "myapp",
        pool = {
            min = 2,
            max = 10,
        }
    },
    features = {
        analytics = true,
        notifications = false,
        cache = {
            enabled = true,
            ttl = 3600,
        }
    },
    allowedOrigins = {
        "https://example.com",
        "https://api.example.com",
        "http://localhost:3000",
    },
}

print("=== Pretty Printed JSON ===")
print(JSON.prettyPrint(config, 2))
```

---

## 30.7 JSON Streaming / Large Data

```lua
-- JSON Line-by-Line Processing (JSONL format)
-- JSONL = JSON Lines - หนึ่งบรรทัดต่อหนึ่ง JSON object

local function processJSONL(text, callback)
    local count = 0
    local errors = 0
    for line in (text .. "\n"):gmatch("([^\n]*)\n") do
        line = line:match("^%s*(.-)%s*$")  -- trim
        if #line > 0 then
            local data, err = JSON.safeDecode(line)
            if data then
                count = count + 1
                callback(data, count)
            else
                errors = errors + 1
                -- optionally: callback(nil, count, err)
            end
        end
    end
    return count, errors
end

-- ตัวอย่าง JSONL data
local jsonlData = [[
{"id": 1, "name": "Alice", "score": 95}
{"id": 2, "name": "Bob", "score": 87}
{"id": 3, "name": "Carol", "score": 92}
{"id": 4, "name": "Dave", "score": 78}
{"id": 5, "name": "Eve", "score": 88}
]]

print("=== JSONL Processing ===")
local totalScore = 0
local count = 0

local processed, errs = processJSONL(jsonlData, function(record, idx)
    totalScore = totalScore + record.score
    count = count + 1
    print(string.format("  [%d] %s: %d", record.id, record.name, record.score))
end)

print(string.format("Processed: %d records, Errors: %d", processed, errs))
print(string.format("Average score: %.1f", totalScore / count))
```

---

## 30.8 JSON Schema Validation (Basic)

```lua
-- JSON Schema Validation
local function validateSchema(data, schema)
    local errors = {}

    local function validate(value, s, path)
        path = path or "#"

        -- Check type
        if s.type then
            local luaType = type(value)
            local jsonType

            if luaType == "boolean" then jsonType = "boolean"
            elseif luaType == "number" then
                if math.type(value) == "integer" then jsonType = "integer"
                else jsonType = "number"
                end
            elseif luaType == "string" then jsonType = "string"
            elseif luaType == "table" then
                -- Check if array or object
                if #value > 0 or next(value) == nil then
                    jsonType = "array"
                else
                    jsonType = "object"
                end
            elseif luaType == "nil" then jsonType = "null"
            end

            local typeOK = false
            if type(s.type) == "string" then
                typeOK = jsonType == s.type or
                         (s.type == "number" and jsonType == "integer")
            elseif type(s.type) == "table" then
                for _, t in ipairs(s.type) do
                    if jsonType == t or (t == "number" and jsonType == "integer") then
                        typeOK = true
                        break
                    end
                end
            end

            if not typeOK then
                errors[#errors+1] = string.format(
                    "%s: expected type '%s', got '%s'",
                    path, type(s.type) == "string" and s.type or table.concat(s.type, "|"),
                    jsonType or luaType
                )
                return
            end
        end

        -- String validations
        if type(value) == "string" then
            if s.minLength and #value < s.minLength then
                errors[#errors+1] = string.format(
                    "%s: string length %d < minLength %d", path, #value, s.minLength)
            end
            if s.maxLength and #value > s.maxLength then
                errors[#errors+1] = string.format(
                    "%s: string length %d > maxLength %d", path, #value, s.maxLength)
            end
            if s.pattern and not value:match(s.pattern) then
                errors[#errors+1] = string.format(
                    "%s: string does not match pattern '%s'", path, s.pattern)
            end
            if s.enum then
                local found = false
                for _, v in ipairs(s.enum) do
                    if v == value then found = true; break end
                end
                if not found then
                    errors[#errors+1] = string.format(
                        "%s: value '%s' not in enum", path, value)
                end
            end
        end

        -- Number validations
        if type(value) == "number" then
            if s.minimum and value < s.minimum then
                errors[#errors+1] = string.format(
                    "%s: %g < minimum %g", path, value, s.minimum)
            end
            if s.maximum and value > s.maximum then
                errors[#errors+1] = string.format(
                    "%s: %g > maximum %g", path, value, s.maximum)
            end
        end

        -- Object validations
        if type(value) == "table" and s.properties then
            -- Check required
            if s.required then
                for _, req in ipairs(s.required) do
                    if value[req] == nil then
                        errors[#errors+1] = string.format(
                            "%s: missing required field '%s'", path, req)
                    end
                end
            end
            -- Validate properties
            for prop, propSchema in pairs(s.properties) do
                if value[prop] ~= nil then
                    validate(value[prop], propSchema, path .. "." .. prop)
                end
            end
        end

        -- Array validations
        if type(value) == "table" and s.items then
            for i, item in ipairs(value) do
                validate(item, s.items, path .. "[" .. i .. "]")
            end
            if s.minItems and #value < s.minItems then
                errors[#errors+1] = string.format(
                    "%s: array length %d < minItems %d", path, #value, s.minItems)
            end
            if s.maxItems and #value > s.maxItems then
                errors[#errors+1] = string.format(
                    "%s: array length %d > maxItems %d", path, #value, s.maxItems)
            end
        end
    end

    validate(data, schema, "#")
    return #errors == 0, errors
end

-- ทดสอบ Schema Validation
local userSchema = {
    type = "object",
    required = {"name", "email", "age"},
    properties = {
        name = {
            type = "string",
            minLength = 2,
            maxLength = 100,
        },
        email = {
            type = "string",
            pattern = "[^@]+@[^@]+%.[^@]+",
        },
        age = {
            type = "integer",
            minimum = 0,
            maximum = 150,
        },
        role = {
            type = "string",
            enum = {"admin", "user", "guest"},
        },
        tags = {
            type = "array",
            items = { type = "string" },
            maxItems = 5,
        },
    }
}

-- Valid user
local validUser = {
    name = "Alice Smith",
    email = "alice@example.com",
    age = 25,
    role = "user",
    tags = {"developer", "lua"},
}

local ok, errs = validateSchema(validUser, userSchema)
print("\n=== Schema Validation ===")
print("Valid user:", ok)  -- true

-- Invalid user
local invalidUser = {
    name = "A",  -- too short
    email = "not-an-email",  -- invalid format
    age = 200,  -- too high
    role = "superadmin",  -- not in enum
}

ok, errs = validateSchema(invalidUser, userSchema)
print("\nInvalid user:", ok)  -- false
for _, e in ipairs(errs) do
    print("  Error:", e)
end
```

---

## 30.9 Config File in JSON

```lua
-- Config File Management
local ConfigManager = {}
ConfigManager.__index = ConfigManager

function ConfigManager.new(defaults)
    return setmetatable({
        _config = {},
        _defaults = defaults or {},
        _validators = {},
    }, ConfigManager)
end

function ConfigManager:loadFromString(jsonStr)
    local data, err = JSON.safeDecode(jsonStr)
    if not data then
        return false, "JSON parse error: " .. tostring(err)
    end

    -- Merge with defaults
    local function merge(base, override)
        local result = {}
        for k, v in pairs(base) do
            result[k] = v
        end
        for k, v in pairs(override) do
            if type(v) == "table" and type(result[k]) == "table" then
                result[k] = merge(result[k], v)
            else
                result[k] = v
            end
        end
        return result
    end

    self._config = merge(self._defaults, data)
    return true, nil
end

function ConfigManager:get(key, default)
    -- Support dot notation: "server.port"
    local parts = {}
    for p in (key .. "."):gmatch("([^%.]+)%.") do
        parts[#parts+1] = p
    end

    local val = self._config
    for _, p in ipairs(parts) do
        if type(val) ~= "table" then return default end
        val = val[p]
    end

    if val == nil then return default end
    return val
end

function ConfigManager:toJSON(pretty)
    return JSON.encode(self._config, { indent = pretty and 2 or nil })
end

-- ทดสอบ Config Manager
local defaultConfig = {
    server = {
        host = "localhost",
        port = 8080,
        timeout = 30,
    },
    database = {
        host = "localhost",
        port = 5432,
        name = "myapp",
        poolSize = 5,
    },
    logging = {
        level = "info",
        file = "app.log",
    },
    debug = false,
}

local configJSON = [[{
    "server": {
        "port": 9000,
        "ssl": true
    },
    "database": {
        "host": "db.production.com",
        "password": "secret123"
    },
    "debug": false
}]]

local cfg = ConfigManager.new(defaultConfig)
local ok, err = cfg:loadFromString(configJSON)
print("Config loaded:", ok)

print("\nConfig values:")
print("server.host:", cfg:get("server.host"))          -- localhost (from default)
print("server.port:", cfg:get("server.port"))          -- 9000 (overridden)
print("server.ssl:", cfg:get("server.ssl"))            -- true (added)
print("server.timeout:", cfg:get("server.timeout"))    -- 30 (from default)
print("database.host:", cfg:get("database.host"))      -- db.production.com
print("database.poolSize:", cfg:get("database.poolSize")) -- 5 (from default)
print("debug:", cfg:get("debug"))                      -- false
print("missing:", cfg:get("notexist", "default_value")) -- default_value
```

---

## 30.10 API Response Handling

```lua
-- API Response Handler
local function createAPIResponse(success, data, error_msg, metadata)
    local response = {
        success = success,
        timestamp = os.time(),
    }

    if success then
        response.data = data
    else
        response.error = {
            message = error_msg,
            code = metadata and metadata.errorCode or "UNKNOWN_ERROR",
        }
    end

    if metadata then
        response.meta = metadata
    end

    return response
end

-- Pagination wrapper
local function paginatedResponse(items, page, pageSize, total)
    return createAPIResponse(true, items, nil, {
        pagination = {
            page = page,
            pageSize = pageSize,
            total = total,
            totalPages = math.ceil(total / pageSize),
            hasNext = page < math.ceil(total / pageSize),
            hasPrev = page > 1,
        }
    })
end

-- ทดสอบ
local users = {
    { id = 1, name = "Alice", email = "alice@example.com" },
    { id = 2, name = "Bob",   email = "bob@example.com" },
    { id = 3, name = "Carol", email = "carol@example.com" },
}

local response = paginatedResponse(users, 1, 10, 25)
print("\n=== API Response ===")
print(JSON.prettyPrint(response))

-- Error response
local errorResponse = createAPIResponse(
    false,
    nil,
    "User not found",
    { errorCode = "USER_NOT_FOUND" }
)
print("\n=== Error Response ===")
print(JSON.prettyPrint(errorResponse))

-- Parse API response
local function parseAPIResponse(jsonStr)
    local data, err = JSON.safeDecode(jsonStr)
    if not data then
        return nil, "Parse error: " .. tostring(err)
    end

    if not data.success then
        return nil, data.error and data.error.message or "Unknown API error"
    end

    return data.data, nil, data.meta
end

-- Simulate API call
local mockResponse = JSON.encode(response)
local parsed, parseErr, meta = parseAPIResponse(mockResponse)
if parseErr then
    print("Error:", parseErr)
else
    print("\nParsed API data:")
    for _, u in ipairs(parsed) do
        print(string.format("  %d: %s <%s>", u.id, u.name, u.email))
    end
    if meta and meta.pagination then
        print(string.format("  Page %d/%d (total: %d)",
            meta.pagination.page,
            meta.pagination.totalPages,
            meta.pagination.total))
    end
end
```

---

## 30.11 Error Handling ใน JSON Processing

```lua
-- Comprehensive Error Handling
local function safeJSONOperation(operation, ...)
    local ok, result = pcall(operation, ...)
    if ok then
        return result, nil
    end
    return nil, tostring(result)
end

-- ทดสอบ error cases ต่างๆ
local invalidJSONs = {
    { input = "{name: 'Alice'}",  desc = "ขาด quotes รอบ key" },
    { input = "{'name': 'Alice'}", desc = "ใช้ single quotes" },
    { input = "{\"name\": \"Alice\",}", desc = "trailing comma" },
    { input = "{\"name\": undefined}", desc = "undefined value" },
    { input = "[1, 2, 3",         desc = "unclosed array" },
    { input = "",                  desc = "empty string" },
    { input = "null",              desc = "valid null" },
    { input = "true",              desc = "valid boolean" },
    { input = "42",                desc = "valid number" },
    { input = '"hello"',           desc = "valid string" },
}

print("=== Error Handling Tests ===")
for _, test in ipairs(invalidJSONs) do
    local result, err = JSON.safeDecode(test.input)
    if err then
        print(string.format("  [ERROR] %s: %s", test.desc, err:sub(1, 50)))
    else
        print(string.format("  [OK] %s: %s", test.desc, tostring(result)))
    end
end

-- Retry with error recovery
local function robustDecode(s, maxRetries)
    maxRetries = maxRetries or 3

    -- Try as-is first
    local data, err = JSON.safeDecode(s)
    if data ~= nil then return data, nil end

    -- Try to fix common issues
    local fixes = {
        -- Remove trailing commas
        function(str) return str:gsub(",(%s*[%]%}])", "%1") end,
        -- Replace single quotes with double (simple case)
        function(str)
            return str:gsub("'([^']*)'", '"' .. "%1" .. '"')
        end,
    }

    for i, fix in ipairs(fixes) do
        if i <= maxRetries then
            local fixed = fix(s)
            data, err = JSON.safeDecode(fixed)
            if data ~= nil then
                print("Fixed with strategy " .. i)
                return data, nil
            end
        end
    end

    return nil, err
end

-- ทดสอบ robust decode
local brokenJSON = '{"name": "Alice", "age": 25,}'  -- trailing comma
local fixed, err = robustDecode(brokenJSON)
if fixed then
    print("\nFixed JSON - name:", fixed.name, "age:", fixed.age)
end
```

---

## 30.12 JSON Diff

```lua
-- JSON Diff: เปรียบเทียบ JSON objects
local function jsonDiff(old, new, path)
    path = path or "#"
    local changes = {}

    local function addChange(changeType, p, oldVal, newVal)
        changes[#changes+1] = {
            type = changeType,
            path = p,
            old = oldVal,
            new = newVal,
        }
    end

    if type(old) ~= type(new) then
        addChange("modified", path, old, new)
        return changes
    end

    if type(old) == "table" then
        -- Check for additions and modifications
        for k, newVal in pairs(new) do
            local childPath = type(k) == "number"
                and (path .. "[" .. k .. "]")
                or (path .. "." .. k)

            if old[k] == nil then
                addChange("added", childPath, nil, newVal)
            elseif type(newVal) == "table" then
                local subChanges = jsonDiff(old[k], newVal, childPath)
                for _, c in ipairs(subChanges) do
                    changes[#changes+1] = c
                end
            elseif old[k] ~= newVal then
                addChange("modified", childPath, old[k], newVal)
            end
        end

        -- Check for deletions
        for k, oldVal in pairs(old) do
            local childPath = type(k) == "number"
                and (path .. "[" .. k .. "]")
                or (path .. "." .. k)
            if new[k] == nil then
                addChange("deleted", childPath, oldVal, nil)
            end
        end
    else
        if old ~= new then
            addChange("modified", path, old, new)
        end
    end

    return changes
end

-- ทดสอบ JSON Diff
local v1 = {
    name = "Alice",
    age = 30,
    email = "alice@old.com",
    address = {
        city = "Bangkok",
        zip = "10110",
    },
    tags = {"developer"},
}

local v2 = {
    name = "Alice",
    age = 31,  -- changed
    -- email removed
    address = {
        city = "Bangkok",
        zip = "10120",  -- changed
        country = "Thailand",  -- added
    },
    tags = {"developer", "manager"},  -- changed
    phone = "081-234-5678",  -- added
}

local changes = jsonDiff(v1, v2)
print("\n=== JSON Diff ===")
for _, change in ipairs(changes) do
    if change.type == "added" then
        print(string.format("  + %s: %s", change.path,
            JSON.encode(change.new)))
    elseif change.type == "deleted" then
        print(string.format("  - %s: %s", change.path,
            JSON.encode(change.old)))
    elseif change.type == "modified" then
        print(string.format("  ~ %s: %s -> %s", change.path,
            JSON.encode(change.old), JSON.encode(change.new)))
    end
end
```

---

## 30.13 cjson Library Usage

```lua
-- หมายเหตุ: cjson ต้องติดตั้งแยก (C extension)
-- ตัวอย่างนี้แสดง API ที่ควรรู้

print("=== cjson Library API (Reference) ===")
print([[
-- Installation:
--   luarocks install lua-cjson
-- หรือ
--   apt-get install lua-cjson

-- Usage:
local cjson = require("cjson")

-- Encode
local data = {name = "Alice", age = 30, scores = {95, 87, 92}}
local json_str = cjson.encode(data)
print(json_str)
-- {"age":30,"name":"Alice","scores":[95,87,92]}

-- Decode
local decoded = cjson.decode(json_str)
print(decoded.name)    -- Alice
print(decoded.age)     -- 30
print(decoded.scores[1]) -- 95

-- Handle null
local with_null = cjson.decode('{"a": null, "b": 1}')
print(with_null.a == cjson.null)  -- true
print(with_null.b)  -- 1

-- Encode null
local obj = { value = cjson.null }
print(cjson.encode(obj))  -- {"value":null}

-- Settings
cjson.new()  -- create new state
cjson.encode_invalid_numbers(false)  -- error on NaN/Inf
cjson.decode_invalid_numbers(false)

-- Pretty print (ไม่ built-in ต้องทำเอง)
]])

-- สร้าง compatibility layer
local function getJSON()
    local ok, cjson = pcall(require, "cjson")
    if ok then
        return {
            encode = cjson.encode,
            decode = cjson.decode,
            null = cjson.null,
            _impl = "cjson",
        }
    end
    -- fallback to our pure Lua implementation
    return {
        encode = JSON.encode,
        decode = JSON.decode,
        null = nil,  -- our impl doesn't have sentinel
        _impl = "pure-lua",
    }
end

local json = getJSON()
print("Using JSON implementation:", json._impl)
```

---

## 30.14 dkjson Library Usage

```lua
-- dkjson เป็น pure Lua library ที่นิยมใช้
-- Installation: luarocks install dkjson

print("=== dkjson Library API (Reference) ===")
print([[
-- Installation:
--   luarocks install dkjson
-- หรือ download dkjson.lua จาก
--   http://dkolf.de/src/dkjson-lua.fsl/

-- Usage:
local json = require("dkjson")

-- Encode
local t = {name = "Alice", age = 30}
local str = json.encode(t)                        -- compact
local str2 = json.encode(t, {indent = true})      -- pretty
local str3 = json.encode(t, {indent = "  "})      -- custom indent

-- Decode
local obj, pos, err = json.decode(str)
if not obj then
    print("Error:", err)
end

-- Decode with custom options
local obj2 = json.decode(str, 1, {})

-- Handle null
-- dkjson returns nil for null by default
-- or you can configure it to use a sentinel

-- Iterating over decoded JSON
-- Arrays: ipairs or for i=1,#arr
-- Objects: pairs

-- Error handling
local ok_obj, ok_pos, ok_err = json.decode('{"valid": true}')
local bad_obj, bad_pos, bad_err = json.decode('{invalid}')
print(bad_err)  -- error message

-- null handling with sentinel
json.null  -- the null sentinel value
]])

-- ทดสอบ compatibility กับ dkjson API
local function dkjsonCompat(impl)
    return {
        encode = function(val, opts)
            if opts and opts.indent then
                return impl.encode(val, { indent = opts.indent == true and 2 or opts.indent })
            end
            return impl.encode(val)
        end,
        decode = function(str)
            local val, err = impl.safeDecode and impl.safeDecode(str) or pcall(impl.decode, str)
            return val
        end
    }
end
```

---

## 30.15 Advanced JSON Examples

```lua
-- ตัวอย่างการใช้งาน JSON จริงๆ

-- 1. JSON-based Event System
local EventLog = {}
EventLog.__index = EventLog

function EventLog.new()
    return setmetatable({ _events = {} }, EventLog)
end

function EventLog:record(eventType, data)
    self._events[#self._events+1] = {
        type = eventType,
        data = data,
        timestamp = os.time(),
        id = #self._events + 1,
    }
end

function EventLog:export()
    return JSON.encode({
        version = "1.0",
        exportedAt = os.time(),
        count = #self._events,
        events = self._events,
    }, { indent = 2 })
end

function EventLog:importEvents(jsonStr)
    local data, err = JSON.safeDecode(jsonStr)
    if not data then return false, err end
    for _, event in ipairs(data.events or {}) do
        self._events[#self._events+1] = event
    end
    return true
end

-- ทดสอบ Event Log
local log = EventLog.new()
log:record("user.login", { userId = 1, ip = "192.168.1.1" })
log:record("user.view_page", { userId = 1, page = "/home" })
log:record("user.purchase", { userId = 1, itemId = 42, amount = 299.99 })
log:record("user.logout", { userId = 1 })

print("=== Event Log Export ===")
print(log:export())

-- 2. JSON Merge Strategy
local function deepMerge(target, source, strategy)
    strategy = strategy or "override"  -- "override", "keep", "concat"

    local result = {}
    for k, v in pairs(target) do result[k] = v end

    for k, v in pairs(source) do
        if result[k] == nil then
            result[k] = v
        elseif type(v) == "table" and type(result[k]) == "table" then
            if strategy == "concat" and #v > 0 and #result[k] > 0 then
                -- Concatenate arrays
                local merged = {}
                for _, item in ipairs(result[k]) do merged[#merged+1] = item end
                for _, item in ipairs(v) do merged[#merged+1] = item end
                result[k] = merged
            else
                result[k] = deepMerge(result[k], v, strategy)
            end
        elseif strategy == "override" then
            result[k] = v
        end
        -- "keep" strategy: result[k] stays
    end

    return result
end

local base = { a = 1, b = { x = 10, y = 20 }, tags = {"foo"} }
local override = { b = { y = 99, z = 30 }, c = 3, tags = {"bar"} }

local merged = deepMerge(base, override, "concat")
print("\n=== Deep Merge (concat) ===")
print(JSON.prettyPrint(merged))

-- 3. JSON Transform Pipeline
local function transformJSON(data, transformers)
    local result = data
    for _, transformer in ipairs(transformers) do
        result = transformer(result)
    end
    return result
end

local rawData = {
    users = {
        { id = 1, first = "Alice", last = "Smith", age = 25 },
        { id = 2, first = "Bob",   last = "Jones", age = 30 },
        { id = 3, first = "Carol", last = "White", age = 28 },
    }
}

local processed = transformJSON(rawData, {
    -- Flatten name
    function(data)
        local result = { users = {} }
        for _, u in ipairs(data.users) do
            result.users[#result.users+1] = {
                id = u.id,
                name = u.first .. " " .. u.last,
                age = u.age,
            }
        end
        return result
    end,
    -- Add computed field
    function(data)
        for _, u in ipairs(data.users) do
            u.isAdult = u.age >= 18
            u.ageGroup = u.age < 25 and "young" or u.age < 35 and "adult" or "senior"
        end
        return data
    end,
    -- Sort by name
    function(data)
        table.sort(data.users, function(a, b) return a.name < b.name end)
        return data
    end,
})

print("\n=== Transform Pipeline ===")
print(JSON.prettyPrint(processed))
```

---

## 30.16 JSON Performance Tips

```lua
-- Performance Tips for JSON processing
print("=== JSON Performance Tips ===")

-- 1. Build large JSON incrementally
local function buildLargeJSON(count)
    local parts = {"["}
    for i = 1, count do
        if i > 1 then parts[#parts+1] = "," end
        parts[#parts+1] = string.format(
            '{"id":%d,"name":"User_%d","score":%d}',
            i, i, math.random(0, 100)
        )
    end
    parts[#parts+1] = "]"
    return table.concat(parts)
end

-- ทดสอบ performance
local t1 = os.clock()
local json1 = buildLargeJSON(1000)
local t2 = os.clock()
print(string.format("Build 1000 records: %.4f s, size: %d bytes",
    t2 - t1, #json1))

-- 2. Decode ครั้งเดียว cache ผล
local cached_configs = {}
local function getCachedConfig(key, jsonStr)
    if not cached_configs[key] then
        cached_configs[key] = JSON.decode(jsonStr)
    end
    return cached_configs[key]
end

-- 3. Lazy decode: decode เฉพาะส่วนที่ต้องการ
local function extractField(jsonStr, fieldName)
    -- Fast regex extraction ก่อน parse เต็ม (สำหรับ top-level string/number)
    local pattern = '"' .. fieldName .. '"%s*:%s*"([^"]*)"'
    local val = jsonStr:match(pattern)
    if val then return val end

    -- Number
    pattern = '"' .. fieldName .. '"%s*:%s*(-?%d+%.?%d*)'
    val = jsonStr:match(pattern)
    if val then return tonumber(val) end

    return nil
end

local json_str = '{"id": 42, "name": "Alice", "score": 98.5, "city": "Bangkok"}'
print("\nFast field extraction:")
print("id:", extractField(json_str, "id"))
print("name:", extractField(json_str, "name"))
print("score:", extractField(json_str, "score"))

print("\n=== Summary ===")
print("1. ใช้ cjson สำหรับ performance (C extension)")
print("2. ใช้ dkjson สำหรับ compatibility (pure Lua)")
print("3. Cache decoded results เมื่อใช้ซ้ำหลายครั้ง")
print("4. Build JSON string โดยตรงสำหรับ large datasets")
print("5. Validate schema ก่อน process เพื่อ fail fast")
print("6. Handle null/nil แยกกันอย่างชัดเจน")
print("7. ใช้ streaming สำหรับ very large JSON (JSONL)")
```

---

## บทสรุป

JSON Processing ใน Lua ครอบคลุม:

1. **Pure Lua Implementation** - เข้าใจ internals ของ JSON parser
2. **dkjson/cjson Libraries** - ใช้งานจริงใน production
3. **Encoding/Decoding** - แปลงระหว่าง Lua tables และ JSON
4. **null Handling** - จัดการ JSON null อย่างถูกต้อง
5. **Pretty Printing** - human-readable output
6. **Streaming (JSONL)** - จัดการข้อมูลขนาดใหญ่
7. **Schema Validation** - ตรวจสอบโครงสร้างข้อมูล
8. **Config Management** - จัดการ configuration files
9. **API Integration** - รับ-ส่งข้อมูลกับ Web APIs
10. **Error Handling** - จัดการข้อผิดพลาดอย่างครบถ้วน
11. **JSON Diff** - เปรียบเทียบ JSON objects
12. **Performance** - เพิ่มประสิทธิภาพการประมวลผล

JSON เป็นหัวใจสำคัญของการพัฒนา modern applications ความเข้าใจ JSON processing อย่างลึกซึ้งจะช่วยให้สามารถพัฒนา APIs, config systems, และ data pipelines ได้อย่างมีประสิทธิภาพ
