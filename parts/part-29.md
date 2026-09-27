# บทที่ 29: Advanced String Processing ใน Lua

## บทนำ

Lua มีความสามารถด้าน string processing ที่ทรงพลังผ่าน Lua pattern matching ซึ่งแตกต่างจาก regular expressions แต่มีประสิทธิภาพสูง บทนี้ครอบคลุมการประมวลผล string ขั้นสูงตั้งแต่ pattern matching ไปจนถึง parsing และ encoding

---

## 29.1 Complex Pattern Matching

Lua patterns ใช้ syntax พิเศษที่ต่างจาก regex

```lua
-- ตัวอย่าง Pattern Matching พื้นฐาน
local text = "สวัสดี Hello World 12345 foo@bar.com"

-- ค้นหาตัวเลข
for num in text:gmatch("%d+") do
    print("Number:", num)  -- 12345
end

-- ค้นหา email
for email in text:gmatch("[%w%.]+@[%w%.]+") do
    print("Email:", email)  -- foo@bar.com
end

-- ค้นหาคำ (ASCII)
for word in text:gmatch("[%a]+") do
    print("Word:", word)
end
```

```lua
-- Pattern classes
local examples = {
    ["digits only"]   = "abc123def456",
    ["mixed"]         = "Hello, World! 2024",
    ["special chars"] = "!@#$%^&*()",
    ["whitespace"]    = "hello   world\t\nfoo",
}

-- %d = digit, %a = alpha, %s = space, %p = punctuation
-- %w = alphanumeric, %l = lowercase, %u = uppercase

for desc, s in pairs(examples) do
    local digits = {}
    for d in s:gmatch("%d") do digits[#digits+1] = d end

    local words = {}
    for w in s:gmatch("%a+") do words[#words+1] = w end

    print(string.format("[%s]", desc))
    if #digits > 0 then print("  digits: " .. table.concat(digits, "")) end
    if #words > 0 then print("  words: " .. table.concat(words, ", ")) end
end
```

```lua
-- Anchors และ captures
local date = "วันที่: 2024-01-15"
local year, month, day = date:match("(%d%d%d%d)-(%d%d)-(%d%d)")
if year then
    print(string.format("Year: %s, Month: %s, Day: %s", year, month, day))
end

-- Multiple captures
local function parseKeyValue(line)
    return line:match("^([%w_]+)%s*=%s*(.+)$")
end

local lines = {
    "name = สมชาย",
    "age = 30",
    "city = กรุงเทพ",
    "email = somchai@example.com",
}

for _, line in ipairs(lines) do
    local key, value = parseKeyValue(line)
    if key then
        print(string.format("'%s' => '%s'", key, value))
    end
end
```

```lua
-- Pattern escaping และ advanced patterns
local function escapePattern(s)
    return s:gsub("([%.%+%-%*%?%[%]%^%$%(%)%%])", "%%%1")
end

local function findLiteral(text, searchStr)
    local escaped = escapePattern(searchStr)
    local positions = {}
    local start = 1
    while true do
        local s, e = text:find(escaped, start)
        if not s then break end
        positions[#positions+1] = s
        start = e + 1
    end
    return positions
end

local text = "Hello.World. Hello.Lua. Hello."
local positions = findLiteral(text, "Hello.")
print("Positions of 'Hello.':", table.concat(positions, ", "))  -- 1, 14, 24
```

---

## 29.2 Tokenizer / Lexer

```lua
-- Simple Tokenizer
local TokenType = {
    NUMBER   = "NUMBER",
    STRING   = "STRING",
    IDENT    = "IDENT",
    KEYWORD  = "KEYWORD",
    OP       = "OP",
    LPAREN   = "LPAREN",
    RPAREN   = "RPAREN",
    COMMA    = "COMMA",
    NEWLINE  = "NEWLINE",
    EOF      = "EOF",
    UNKNOWN  = "UNKNOWN",
}

local KEYWORDS = {
    ["if"] = true, ["then"] = true, ["else"] = true, ["end"] = true,
    ["while"] = true, ["do"] = true, ["for"] = true, ["return"] = true,
    ["local"] = true, ["function"] = true, ["true"] = true, ["false"] = true,
    ["nil"] = true, ["and"] = true, ["or"] = true, ["not"] = true,
}

local function tokenize(source)
    local tokens = {}
    local pos = 1
    local line = 1

    local function peek()
        if pos > #source then return nil end
        return source:sub(pos, pos)
    end

    local function advance()
        local ch = source:sub(pos, pos)
        pos = pos + 1
        if ch == "\n" then line = line + 1 end
        return ch
    end

    local function match(expected)
        if pos <= #source and source:sub(pos, pos) == expected then
            pos = pos + 1
            return true
        end
        return false
    end

    local function addToken(ttype, value)
        tokens[#tokens+1] = { type = ttype, value = value, line = line }
    end

    while pos <= #source do
        local ch = peek()

        -- Skip whitespace (except newline)
        if ch == " " or ch == "\t" or ch == "\r" then
            advance()

        -- Newline
        elseif ch == "\n" then
            advance()
            addToken(TokenType.NEWLINE, "\n")

        -- Numbers
        elseif ch:match("%d") then
            local start = pos
            while pos <= #source and source:sub(pos, pos):match("[%d%.]") do
                pos = pos + 1
            end
            addToken(TokenType.NUMBER, source:sub(start, pos - 1))

        -- Strings
        elseif ch == '"' or ch == "'" then
            local quote = advance()
            local str = {}
            while pos <= #source and peek() ~= quote do
                local c = advance()
                if c == "\\" then
                    local esc = advance()
                    if esc == "n" then str[#str+1] = "\n"
                    elseif esc == "t" then str[#str+1] = "\t"
                    else str[#str+1] = esc
                    end
                else
                    str[#str+1] = c
                end
            end
            advance()  -- closing quote
            addToken(TokenType.STRING, table.concat(str))

        -- Identifiers and keywords
        elseif ch:match("[%a_]") then
            local start = pos
            while pos <= #source and source:sub(pos, pos):match("[%w_]") do
                pos = pos + 1
            end
            local word = source:sub(start, pos - 1)
            if KEYWORDS[word] then
                addToken(TokenType.KEYWORD, word)
            else
                addToken(TokenType.IDENT, word)
            end

        -- Operators
        elseif ch == "+" or ch == "-" or ch == "*" or ch == "/" then
            addToken(TokenType.OP, advance())
        elseif ch == "=" then
            advance()
            if match("=") then addToken(TokenType.OP, "==")
            else addToken(TokenType.OP, "=")
            end
        elseif ch == "<" then
            advance()
            if match("=") then addToken(TokenType.OP, "<=")
            else addToken(TokenType.OP, "<")
            end
        elseif ch == ">" then
            advance()
            if match("=") then addToken(TokenType.OP, ">=")
            else addToken(TokenType.OP, ">")
            end
        elseif ch == "~" then
            advance()
            if match("=") then addToken(TokenType.OP, "~=")
            else addToken(TokenType.UNKNOWN, "~")
            end
        elseif ch == "(" then advance(); addToken(TokenType.LPAREN, "(")
        elseif ch == ")" then advance(); addToken(TokenType.RPAREN, ")")
        elseif ch == "," then advance(); addToken(TokenType.COMMA, ",")

        -- Comments
        elseif ch == "-" then
            advance()
            if match("-") then
                -- Line comment: skip to end of line
                while pos <= #source and peek() ~= "\n" do advance() end
            else
                addToken(TokenType.OP, "-")
            end
        else
            addToken(TokenType.UNKNOWN, advance())
        end
    end

    addToken(TokenType.EOF, "")
    return tokens
end

-- ทดสอบ Tokenizer
local code = [[
local x = 10
local y = x + 5
if y > 12 then
    return "big"
end
]]

local tokens = tokenize(code)
print("Tokens:")
for _, tok in ipairs(tokens) do
    if tok.type ~= TokenType.NEWLINE and tok.type ~= TokenType.EOF then
        print(string.format("  [%s] '%s' (line %d)", tok.type, tok.value, tok.line))
    end
end
```

---

## 29.3 Recursive Descent Parser

```lua
-- Simple expression parser (arithmetic)
local function parseExpression(tokens)
    local pos = 1

    local function peek()
        return tokens[pos]
    end

    local function consume(expected_type)
        local tok = tokens[pos]
        if expected_type and tok.type ~= expected_type then
            error(string.format("Expected %s, got %s at position %d",
                expected_type, tok.type, pos))
        end
        pos = pos + 1
        return tok
    end

    local function parseNumber()
        local tok = consume("NUMBER")
        return { type = "number", value = tonumber(tok.value) }
    end

    local function parseIdent()
        local tok = consume("IDENT")
        return { type = "ident", name = tok.value }
    end

    local parseExpr  -- forward declaration

    local function parsePrimary()
        local tok = peek()
        if tok.type == "NUMBER" then
            return parseNumber()
        elseif tok.type == "IDENT" then
            return parseIdent()
        elseif tok.type == "LPAREN" then
            consume("LPAREN")
            local expr = parseExpr()
            consume("RPAREN")
            return expr
        else
            error("Unexpected token: " .. tok.type)
        end
    end

    local function parseUnary()
        local tok = peek()
        if tok.type == "OP" and tok.value == "-" then
            consume("OP")
            return { type = "unary", op = "-", expr = parsePrimary() }
        end
        return parsePrimary()
    end

    local function parseMulDiv()
        local left = parseUnary()
        while peek().type == "OP" and
              (peek().value == "*" or peek().value == "/") do
            local op = consume("OP").value
            local right = parseUnary()
            left = { type = "binary", op = op, left = left, right = right }
        end
        return left
    end

    local function parseAddSub()
        local left = parseMulDiv()
        while peek().type == "OP" and
              (peek().value == "+" or peek().value == "-") do
            local op = consume("OP").value
            local right = parseMulDiv()
            left = { type = "binary", op = op, left = left, right = right }
        end
        return left
    end

    parseExpr = parseAddSub

    local ast = parseExpr()
    return ast
end

-- Evaluate AST
local function evalAST(node, env)
    env = env or {}
    if node.type == "number" then
        return node.value
    elseif node.type == "ident" then
        return env[node.name] or error("Undefined: " .. node.name)
    elseif node.type == "unary" then
        return -evalAST(node.expr, env)
    elseif node.type == "binary" then
        local l = evalAST(node.left, env)
        local r = evalAST(node.right, env)
        if node.op == "+" then return l + r
        elseif node.op == "-" then return l - r
        elseif node.op == "*" then return l * r
        elseif node.op == "/" then return l / r
        end
    end
end

-- ทดสอบ Parser
local exprStr = "3 + 4 * 2"
local exprTokens = tokenize(exprStr)
-- กรอง newline และ EOF
local filtered = {}
for _, t in ipairs(exprTokens) do
    if t.type ~= "NEWLINE" and t.type ~= "EOF" then
        filtered[#filtered+1] = t
    end
end
filtered[#filtered+1] = { type = "EOF", value = "" }

local ast = parseExpression(filtered)
local result = evalAST(ast)
print("3 + 4 * 2 =", result)  -- 11

exprStr = "(3 + 4) * 2"
exprTokens = tokenize(exprStr)
filtered = {}
for _, t in ipairs(exprTokens) do
    if t.type ~= "NEWLINE" and t.type ~= "EOF" then
        filtered[#filtered+1] = t
    end
end
filtered[#filtered+1] = { type = "EOF", value = "" }

ast = parseExpression(filtered)
result = evalAST(ast)
print("(3 + 4) * 2 =", result)  -- 14
```

---

## 29.4 Template Engine

```lua
-- Simple Template Engine
local function renderTemplate(template, vars)
    -- แทนที่ {{variable}} ด้วยค่าจริง
    return (template:gsub("{{(%s*([%w_%.]+)%s*)}}", function(_, key)
        -- รองรับ dot notation: user.name
        local parts = {}
        for p in key:gmatch("[%w_]+") do parts[#parts+1] = p end

        local val = vars
        for _, part in ipairs(parts) do
            if type(val) == "table" then
                val = val[part]
            else
                val = nil
                break
            end
        end

        if val == nil then return "{{" .. key .. "}}"
        else return tostring(val)
        end
    end))
end

-- ทดสอบ Template Engine
local template = [[
สวัสดีคุณ {{name}}!
อีเมล: {{email}}
อายุ: {{age}} ปี
เมือง: {{address.city}}
]]

local data = {
    name = "สมชาย วงศ์ดี",
    email = "somchai@example.com",
    age = 30,
    address = { city = "กรุงเทพ", country = "ไทย" }
}

print(renderTemplate(template, data))
```

```lua
-- Template Engine ขั้นสูงพร้อม loops และ conditionals
local function advancedTemplate(template, vars)
    -- Process {{#each items}}...{{/each}}
    template = template:gsub("{{#each ([%w_]+)}}(.-){{/each}}", function(listKey, body)
        local list = vars[listKey]
        if type(list) ~= "table" then return "" end
        local parts = {}
        for i, item in ipairs(list) do
            local ctx = {}
            if type(item) == "table" then
                for k, v in pairs(item) do ctx[k] = v end
            else
                ctx.value = item
                ctx.index = i
            end
            parts[#parts+1] = renderTemplate(body, ctx)
        end
        return table.concat(parts)
    end)

    -- Process {{#if condition}}...{{/if}}
    template = template:gsub("{{#if ([%w_%.]+)}}(.-){{/if}}", function(cond, body)
        local parts = {}
        for p in cond:gmatch("[%w_]+") do parts[#parts+1] = p end
        local val = vars
        for _, part in ipairs(parts) do
            if type(val) == "table" then val = val[part] else val = nil; break end
        end
        if val and val ~= false then
            return renderTemplate(body, vars)
        end
        return ""
    end)

    -- Simple variables
    return renderTemplate(template, vars)
end

local tmpl2 = [[รายการสินค้า:
{{#each products}}  - {{name}}: {{price}} บาท
{{/each}}
{{#if hasDiscount}}ส่วนลด: {{discount}}%
{{/if}}รวม: {{total}} บาท]]

local data2 = {
    products = {
        { name = "Apple MacBook", price = 45000 },
        { name = "Dell XPS",     price = 38000 },
        { name = "Sony Headphones", price = 3500 },
    },
    hasDiscount = true,
    discount = 10,
    total = 77850,
}

print(advancedTemplate(tmpl2, data2))
```

---

## 29.5 String Interpolation

```lua
-- String interpolation ด้วย format strings
local function interpolate(s, ...)
    local args = {...}
    local i = 0
    return (s:gsub("{}", function()
        i = i + 1
        return tostring(args[i] or "")
    end))
end

print(interpolate("Hello, {}! You are {} years old.", "Alice", 25))
-- Hello, Alice! You are 25 years old.

-- Named interpolation
local function namedInterpolate(s, vars)
    return (s:gsub("{([%w_]+)}", function(key)
        return tostring(vars[key] or "")
    end))
end

print(namedInterpolate("Dear {name}, your order #{order} is ready.", {
    name = "สมชาย",
    order = "ORD-2024-001"
}))

-- Format interpolation (printf-style)
local function fmt(template, vars)
    return (template:gsub("{([%w_]+):([^}]*)}", function(key, format)
        local val = vars[key]
        if val == nil then return "{" .. key .. "}" end
        if format:match("^%.%d+f$") then
            return string.format("%" .. format, tonumber(val) or 0)
        elseif format == "d" or format == "i" then
            return string.format("%d", math.floor(tonumber(val) or 0))
        elseif format == "s" then
            return tostring(val)
        end
        return tostring(val)
    end))
end

print(fmt("Price: {price:.2f} THB, Qty: {qty:d}", {
    price = 1234.5678,
    qty = 5
}))
-- Price: 1234.57 THB, Qty: 5
```

---

## 29.6 URL Encode / Decode

```lua
-- URL Encoding
local function urlEncode(str)
    str = str:gsub("\n", "\r\n")
    str = str:gsub("([^%w%-%.%_%~ ])", function(c)
        return string.format("%%%02X", c:byte())
    end)
    str = str:gsub(" ", "+")
    return str
end

local function urlDecode(str)
    str = str:gsub("+", " ")
    str = str:gsub("%%(%x%x)", function(h)
        return string.char(tonumber(h, 16))
    end)
    return str
end

-- ทดสอบ
local original = "Hello World! สวัสดี #test=value&foo=bar baz"
local encoded = urlEncode(original)
local decoded = urlDecode(encoded)

print("Original:", original)
print("Encoded:", encoded)
print("Decoded:", decoded)
print("Match:", original == decoded)

-- Build query string
local function buildQueryString(params)
    local parts = {}
    for k, v in pairs(params) do
        parts[#parts+1] = urlEncode(tostring(k)) .. "=" .. urlEncode(tostring(v))
    end
    table.sort(parts)  -- deterministic output
    return table.concat(parts, "&")
end

local params = {
    name = "สมชาย วงศ์ดี",
    city = "กรุงเทพ",
    page = 1,
    search = "lua programming",
}
print("\nQuery string:")
print(buildQueryString(params))

-- Parse query string
local function parseQueryString(qs)
    local result = {}
    for pair in (qs .. "&"):gmatch("([^&]+)&") do
        local key, value = pair:match("^([^=]*)=?(.*)$")
        if key and key ~= "" then
            result[urlDecode(key)] = urlDecode(value)
        end
    end
    return result
end

local qs = "name=Alice&age=25&city=Bangkok"
local parsed = parseQueryString(qs)
for k, v in pairs(parsed) do
    print(string.format("  %s = %s", k, v))
end
```

---

## 29.7 HTML Entities

```lua
-- HTML Entity Encoding/Decoding
local HTML_ENTITIES_ENCODE = {
    ["&"] = "&amp;",
    ["<"] = "&lt;",
    [">"] = "&gt;",
    ['"'] = "&quot;",
    ["'"] = "&#39;",
}

local HTML_ENTITIES_DECODE = {}
for k, v in pairs(HTML_ENTITIES_ENCODE) do
    HTML_ENTITIES_DECODE[v] = k
end

local function htmlEncode(str)
    return (str:gsub('[&<>"\']', HTML_ENTITIES_ENCODE))
end

local function htmlDecode(str)
    -- Named entities
    str = str:gsub("&(%a+);", function(name)
        local entity = "&" .. name .. ";"
        return HTML_ENTITIES_DECODE[entity] or entity
    end)
    -- Numeric entities
    str = str:gsub("&#(%d+);", function(code)
        return string.char(tonumber(code))
    end)
    str = str:gsub("&#x(%x+);", function(hex)
        return string.char(tonumber(hex, 16))
    end)
    return str
end

-- ทดสอบ
local html = '<script>alert("XSS & danger");</script>'
local encoded = htmlEncode(html)
local decoded = htmlDecode(encoded)

print("Original:", html)
print("Encoded:", encoded)
print("Decoded:", decoded)
print("Match:", html == decoded)

-- Strip HTML tags
local function stripHTML(html_str)
    local text = html_str:gsub("<[^>]+>", "")
    text = htmlDecode(text)
    text = text:gsub("%s+", " ")
    return text:match("^%s*(.-)%s*$")  -- trim
end

local html2 = "<h1>Hello</h1><p>This is <strong>bold</strong> &amp; <em>italic</em>.</p>"
print("\nStripped HTML:", stripHTML(html2))
```

---

## 29.8 Base64 Encode / Decode

```lua
-- Base64 Encoding/Decoding
local BASE64_CHARS = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"

local function base64Encode(data)
    local encoded = {}
    local padding = (3 - (#data % 3)) % 3

    -- Pad data
    data = data .. string.rep("\0", padding)

    for i = 1, #data, 3 do
        local b1, b2, b3 = data:byte(i, i+2)
        local n = b1 * 65536 + b2 * 256 + b3

        encoded[#encoded+1] = BASE64_CHARS:sub(math.floor(n / 262144) % 64 + 1, math.floor(n / 262144) % 64 + 1)
        encoded[#encoded+1] = BASE64_CHARS:sub(math.floor(n / 4096) % 64 + 1, math.floor(n / 4096) % 64 + 1)
        encoded[#encoded+1] = BASE64_CHARS:sub(math.floor(n / 64) % 64 + 1, math.floor(n / 64) % 64 + 1)
        encoded[#encoded+1] = BASE64_CHARS:sub(n % 64 + 1, n % 64 + 1)
    end

    -- Replace padding
    local result = table.concat(encoded)
    result = result:sub(1, #result - padding) .. string.rep("=", padding)
    return result
end

local function base64Decode(data)
    -- Build lookup table
    local lookup = {}
    for i = 1, #BASE64_CHARS do
        lookup[BASE64_CHARS:sub(i, i)] = i - 1
    end
    lookup["="] = 0

    -- Remove whitespace
    data = data:gsub("[^%w%+%/%=]", "")

    local decoded = {}
    for i = 1, #data, 4 do
        local b1 = lookup[data:sub(i, i)] or 0
        local b2 = lookup[data:sub(i+1, i+1)] or 0
        local b3 = lookup[data:sub(i+2, i+2)] or 0
        local b4 = lookup[data:sub(i+3, i+3)] or 0

        local n = b1 * 262144 + b2 * 4096 + b3 * 64 + b4
        decoded[#decoded+1] = string.char(math.floor(n / 65536) % 256)
        if data:sub(i+2, i+2) ~= "=" then
            decoded[#decoded+1] = string.char(math.floor(n / 256) % 256)
        end
        if data:sub(i+3, i+3) ~= "=" then
            decoded[#decoded+1] = string.char(n % 256)
        end
    end

    return table.concat(decoded)
end

-- ทดสอบ Base64
local tests = {
    "Hello, World!",
    "Man",
    "M",
    "Ma",
    "Lua is awesome!",
    "1234567890",
}

for _, test in ipairs(tests) do
    local enc = base64Encode(test)
    local dec = base64Decode(enc)
    local ok = dec == test
    print(string.format("'%s' -> '%s' -> ok=%s", test, enc, tostring(ok)))
end
```

---

## 29.9 String Hashing

```lua
-- Simple Hash Functions
local function djb2Hash(str)
    local hash = 5381
    for i = 1, #str do
        hash = (hash * 33 + str:byte(i)) & 0xFFFFFFFF
    end
    return hash
end

local function fnv1aHash(str)
    local hash = 2166136261
    for i = 1, #str do
        hash = hash ~ str:byte(i)
        hash = (hash * 16777619) & 0xFFFFFFFF
    end
    return hash
end

-- Simple rolling hash สำหรับ substring matching
local function rollingHash(str, windowSize, base, mod)
    base = base or 31
    mod = mod or 10^9 + 7

    local hashes = {}
    local h = 0
    local power = 1

    -- คำนวณ hash ของ window แรก
    for i = 1, windowSize do
        h = (h + str:byte(i) * power) % mod
        if i < windowSize then power = (power * base) % mod end
    end
    hashes[1] = h

    -- Rolling hash
    for i = windowSize + 1, #str do
        -- ลบตัวแรก เพิ่มตัวใหม่
        h = (h - str:byte(i - windowSize)) % mod
        h = (h * (mod - math.floor(mod / base))) % mod  -- divide by base
        h = (h + str:byte(i) * power) % mod
        if h < 0 then h = h + mod end
        hashes[i - windowSize + 1] = h
    end

    return hashes
end

-- ทดสอบ hash functions
local strings = {"hello", "world", "lua", "programming", "data"}
print("\nHash values:")
for _, s in ipairs(strings) do
    print(string.format("  %-15s djb2=%10u, fnv1a=%10u",
        s, djb2Hash(s), fnv1aHash(s)))
end

-- ทดสอบ collision resistance
local similar = {"cat", "bat", "hat", "mat", "sat"}
print("\nSimilar strings - hash distribution:")
for _, s in ipairs(similar) do
    print(string.format("  %s -> %d", s, djb2Hash(s) % 100))
end
```

---

## 29.10 Soundex Algorithm

```lua
-- Soundex: phonetic algorithm สำหรับ English
local function soundex(name)
    -- Convert to uppercase
    name = name:upper()
    if #name == 0 then return "0000" end

    local SOUNDEX_MAP = {
        B = 1, F = 1, P = 1, V = 1,
        C = 2, G = 2, J = 2, K = 2, Q = 2, S = 2, X = 2, Z = 2,
        D = 3, T = 3,
        L = 4,
        M = 5, N = 5,
        R = 6,
    }

    local first = name:sub(1, 1)
    local code = {first}
    local lastCode = SOUNDEX_MAP[first] or 0

    for i = 2, #name do
        local ch = name:sub(i, i)
        local c = SOUNDEX_MAP[ch]
        if c and c ~= lastCode then
            code[#code+1] = tostring(c)
            if #code == 4 then break end
        end
        if ch:match("[%a]") then
            lastCode = c or 0
        end
    end

    -- Pad with zeros
    while #code < 4 do code[#code+1] = "0" end

    return table.concat(code):sub(1, 4)
end

-- ทดสอบ Soundex
local names = {
    "Robert", "Rupert", "Rubin",  -- similar sounds
    "Smith", "Smythe",
    "Thomson", "Thompson",
    "Johnson", "Jonson",
}

print("\nSoundex:")
for _, name in ipairs(names) do
    print(string.format("  %-15s -> %s", name, soundex(name)))
end
-- Robert, Rupert, Rubin ควรให้ผลเหมือนกัน (R163)
```

---

## 29.11 String Formatting Library

```lua
-- Custom String Formatter
local Formatter = {}
Formatter.__index = Formatter

function Formatter.new()
    return setmetatable({ _formatters = {} }, Formatter)
end

function Formatter:register(name, fn)
    self._formatters[name] = fn
    return self
end

function Formatter:format(template, vars)
    return (template:gsub("{([%w_]+)(?::(.-))}", function(key, fmt)
        -- รองรับ pattern {key:format}
        local parts = {}
        for p in (key .. ":"):gmatch("([^:]+):") do
            parts[#parts+1] = p
        end
        key = parts[1]
        fmt = parts[2]

        local val = vars[key]
        if val == nil then return "{" .. key .. "}" end

        if fmt and self._formatters[fmt] then
            return self._formatters[fmt](val)
        end
        return tostring(val)
    end))
end

-- ทดสอบ
local fmt = Formatter.new()
fmt:register("upper", string.upper)
fmt:register("lower", string.lower)
fmt:register("money", function(n)
    return string.format("฿%,.2f":gsub(",", ","), n)
end)
fmt:register("date", function(t)
    return os.date("%d/%m/%Y", t)
end)
fmt:register("percent", function(n)
    return string.format("%.1f%%", n * 100)
end)

-- สร้าง string format functions
local function padLeft(s, width, ch)
    s = tostring(s)
    ch = ch or " "
    while #s < width do s = ch .. s end
    return s
end

local function padRight(s, width, ch)
    s = tostring(s)
    ch = ch or " "
    while #s < width do s = s .. ch end
    return s
end

local function center(s, width, ch)
    s = tostring(s)
    ch = ch or " "
    local pad = width - #s
    local lpad = math.floor(pad / 2)
    local rpad = pad - lpad
    return string.rep(ch, lpad) .. s .. string.rep(ch, rpad)
end

local function numberFormat(n, decimals, thousandSep, decimalSep)
    decimals = decimals or 0
    thousandSep = thousandSep or ","
    decimalSep = decimalSep or "."

    local formatted = string.format("%." .. decimals .. "f", n)
    local int_part, dec_part = formatted:match("^(-?%d+)(%.?.*)$")

    -- เพิ่ม thousand separator
    local result = {}
    local len = #int_part
    local start = int_part:sub(1, 1) == "-" and 2 or 1

    for i = start, len do
        result[#result+1] = int_part:sub(i, i)
        local remaining = len - i
        if remaining > 0 and remaining % 3 == 0 then
            result[#result+1] = thousandSep
        end
    end

    local intStr = (start > 1 and "-" or "") .. table.concat(result)
    if dec_part and #dec_part > 1 then
        return intStr .. decimalSep .. dec_part:sub(2)
    end
    return intStr
end

print("\nString Formatting:")
print(padLeft("42", 10))           -- "        42"
print(padRight("hello", 10, "-"))  -- "hello-----"
print(center("Lua", 11, "*"))      -- "****Lua****"
print(numberFormat(1234567.89, 2)) -- "1,234,567.89"
print(numberFormat(9876543, 0, ".")) -- "9.876.543"
```

---

## 29.12 Internationalization (i18n) Basics

```lua
-- Simple i18n System
local i18n = {}
i18n.__index = i18n

function i18n.new(defaultLocale)
    return setmetatable({
        _translations = {},
        _locale = defaultLocale or "th",
        _fallback = "en",
    }, i18n)
end

function i18n:addTranslations(locale, translations)
    if not self._translations[locale] then
        self._translations[locale] = {}
    end
    for k, v in pairs(translations) do
        self._translations[locale][k] = v
    end
end

function i18n:t(key, vars)
    -- ค้นหาใน locale ปัจจุบันก่อน
    local trans = self._translations[self._locale]
    local text = trans and trans[key]

    -- Fallback to default locale
    if not text then
        trans = self._translations[self._fallback]
        text = trans and trans[key]
    end

    -- ถ้ายังไม่เจอ return key
    if not text then return key end

    -- แทนที่ variables
    if vars then
        text = text:gsub("{(%w+)}", function(varKey)
            return tostring(vars[varKey] or "")
        end)
    end

    return text
end

function i18n:setLocale(locale)
    self._locale = locale
end

-- ทดสอบ i18n
local translator = i18n.new("th")

translator:addTranslations("th", {
    greeting = "สวัสดีคุณ {name}!",
    farewell = "ลาก่อนคุณ {name}",
    items_count = "คุณมีสินค้า {count} รายการ",
    welcome = "ยินดีต้อนรับสู่ระบบ",
})

translator:addTranslations("en", {
    greeting = "Hello, {name}!",
    farewell = "Goodbye, {name}",
    items_count = "You have {count} items",
    welcome = "Welcome to the system",
})

translator:addTranslations("ja", {
    greeting = "こんにちは、{name}さん！",
    welcome = "システムへようこそ",
})

print("Thai:")
print(translator:t("greeting", { name = "สมชาย" }))
print(translator:t("items_count", { count = 5 }))

print("\nEnglish:")
translator:setLocale("en")
print(translator:t("greeting", { name = "Alice" }))
print(translator:t("items_count", { count = 3 }))

print("\nJapanese:")
translator:setLocale("ja")
print(translator:t("greeting", { name = "田中" }))
print(translator:t("items_count", { count = 2 }))  -- fallback to en
```

---

## 29.13 Multi-line String Processing

```lua
-- Multi-line string utilities
local function splitLines(text)
    local lines = {}
    for line in (text .. "\n"):gmatch("([^\n]*)\n") do
        lines[#lines+1] = line
    end
    return lines
end

local function joinLines(lines, sep)
    return table.concat(lines, sep or "\n")
end

local function trimLine(line)
    return line:match("^%s*(.-)%s*$")
end

local function indentLines(text, spaces)
    local indent = string.rep(" ", spaces)
    return (text:gsub("([^\n]+)", indent .. "%1"))
end

local function wrapText(text, width)
    local result = {}
    local currentLine = ""

    for word in text:gmatch("%S+") do
        if #currentLine + #word + 1 > width and #currentLine > 0 then
            result[#result+1] = currentLine
            currentLine = word
        elseif #currentLine == 0 then
            currentLine = word
        else
            currentLine = currentLine .. " " .. word
        end
    end
    if #currentLine > 0 then
        result[#result+1] = currentLine
    end
    return table.concat(result, "\n")
end

local function countWords(text)
    local count = 0
    for _ in text:gmatch("%S+") do count = count + 1 end
    return count
end

local function countChars(text, includeSpaces)
    if includeSpaces then return #text end
    local count = 0
    for _ in text:gmatch("%S") do count = count + 1 end
    return count
end

-- ทดสอบ
local multilineText = [[
    สวัสดี นี่คือบทความเกี่ยวกับ Lua
    ซึ่งเป็นภาษาโปรแกรมที่เบาและยืดหยุ่น
    
    Lua ถูกสร้างในปี 1993 ที่บราซิล
    และเป็นที่นิยมในวงการ game development
]]

local lines = splitLines(multilineText)
print("จำนวนบรรทัด:", #lines)

-- กรองบรรทัดว่าง
local nonEmpty = {}
for _, line in ipairs(lines) do
    local trimmed = trimLine(line)
    if #trimmed > 0 then
        nonEmpty[#nonEmpty+1] = trimmed
    end
end

print("บรรทัดที่ไม่ว่าง:", #nonEmpty)
for i, line in ipairs(nonEmpty) do
    print(string.format("  %d: %s", i, line))
end

-- Word wrap
local longText = "Lua is a powerful, efficient, lightweight, embeddable scripting language"
print("\nWord wrap (30 chars):")
print(wrapText(longText, 30))

print("\nIndented 4 spaces:")
print(indentLines("line one\nline two\nline three", 4))
```

---

## 29.14 CSV Parser

```lua
-- CSV Parser
local function parseCSV(text, options)
    options = options or {}
    local sep = options.separator or ","
    local quote = options.quote or '"'
    local hasHeader = options.header ~= false

    local function parseLine(line)
        local fields = {}
        local i = 1

        while i <= #line do
            local ch = line:sub(i, i)
            if ch == quote then
                -- Quoted field
                local field = {}
                i = i + 1
                while i <= #line do
                    local c = line:sub(i, i)
                    if c == quote then
                        if i < #line and line:sub(i+1, i+1) == quote then
                            -- Escaped quote
                            field[#field+1] = quote
                            i = i + 2
                        else
                            i = i + 1
                            break
                        end
                    else
                        field[#field+1] = c
                        i = i + 1
                    end
                end
                fields[#fields+1] = table.concat(field)
                -- Skip separator
                if i <= #line and line:sub(i, i) == sep then i = i + 1 end
            else
                -- Unquoted field
                local start = i
                while i <= #line and line:sub(i, i) ~= sep do
                    i = i + 1
                end
                fields[#fields+1] = line:sub(start, i - 1)
                if i <= #line then i = i + 1 end  -- skip separator
            end
        end

        return fields
    end

    local lines = splitLines(text)
    local result = {}
    local headers = nil
    local startLine = 1

    if hasHeader and #lines > 0 then
        headers = parseLine(lines[1])
        startLine = 2
    end

    for i = startLine, #lines do
        local line = lines[i]
        if #trimLine(line) > 0 then
            local fields = parseLine(line)
            if headers then
                local row = {}
                for j, h in ipairs(headers) do
                    row[h] = fields[j] or ""
                end
                result[#result+1] = row
            else
                result[#result+1] = fields
            end
        end
    end

    return result, headers
end

-- ทดสอบ CSV
local csv = [[Name,Age,City,Score
"สมชาย วงศ์ดี",30,กรุงเทพ,92.5
"Alice ""Bob"" Smith",25,New York,88.0
สมหญิง,28,เชียงใหม่,95.0
]]

local data, headers = parseCSV(csv)
print("Headers:", table.concat(headers, " | "))
print("Rows:", #data)
for _, row in ipairs(data) do
    print(string.format("  %s (%s) - Score: %s",
        row.Name, row.City, row.Score))
end

-- CSV Encoder
local function encodeCSV(data, headers)
    local lines = {}
    if headers then
        lines[1] = table.concat(headers, ",")
    end
    for _, row in ipairs(data) do
        local fields = {}
        for _, h in ipairs(headers or {}) do
            local val = tostring(row[h] or "")
            if val:find('[,"\n]') then
                val = '"' .. val:gsub('"', '""') .. '"'
            end
            fields[#fields+1] = val
        end
        lines[#lines+1] = table.concat(fields, ",")
    end
    return table.concat(lines, "\n")
end

local newData = {
    { Name = "Test User", Age = "35", City = "Bangkok,Thailand", Score = "100" },
}
print("\nEncoded CSV:")
print(encodeCSV(newData, {"Name", "Age", "City", "Score"}))
```

---

## 29.15 String Compression (Run-Length Encoding)

```lua
-- Run-Length Encoding (RLE)
local function rleEncode(s)
    if #s == 0 then return "" end
    local result = {}
    local i = 1

    while i <= #s do
        local ch = s:sub(i, i)
        local count = 1
        while i + count <= #s and s:sub(i + count, i + count) == ch do
            count = count + 1
        end
        if count > 1 then
            result[#result+1] = count .. ch
        else
            result[#result+1] = ch
        end
        i = i + count
    end

    return table.concat(result)
end

local function rleDecode(s)
    return (s:gsub("(%d+)(.)", function(count, ch)
        return ch:rep(tonumber(count))
    end))
end

-- ทดสอบ RLE
local tests = {
    "AAABBBCCDDDDEEEE",
    "ABCDE",
    "AAAAAAAAAAAA",
    "ABBBBBBBBBBA",
}

print("\nRun-Length Encoding:")
for _, t in ipairs(tests) do
    local enc = rleEncode(t)
    local dec = rleDecode(enc)
    local ratio = #enc / #t
    print(string.format("  '%s' -> '%s' (%.1f%% of original)",
        t, enc, ratio * 100))
    assert(dec == t, "Decode mismatch!")
end

-- Huffman coding ตัวอย่างอย่างง่าย (frequency analysis)
local function charFrequency(s)
    local freq = {}
    for i = 1, #s do
        local ch = s:sub(i, i)
        freq[ch] = (freq[ch] or 0) + 1
    end
    return freq
end

local sample = "this is a sample string to analyze frequency"
local freq = charFrequency(sample)

-- Sort by frequency
local freqList = {}
for ch, count in pairs(freq) do
    freqList[#freqList+1] = { char = ch, count = count }
end
table.sort(freqList, function(a, b) return a.count > b.count end)

print("\nCharacter Frequency:")
for i = 1, math.min(10, #freqList) do
    local item = freqList[i]
    local ch = item.char == " " and "SPACE" or item.char
    print(string.format("  '%s': %d (%.1f%%)",
        ch, item.count, item.count / #sample * 100))
end
```

---

## บทสรุป

Advanced String Processing ใน Lua ครอบคลุม:
1. **Pattern Matching** - Lua's powerful pattern system
2. **Tokenizer/Lexer** - การแยกข้อความเป็น tokens
3. **Parser** - การแปลง tokens เป็น AST
4. **Template Engine** - dynamic text generation
5. **URL/HTML encoding** - web-safe text
6. **Base64** - binary-to-text encoding
7. **Hashing** - fingerprinting strings
8. **Formatting** - presentation formatting
9. **i18n** - multilingual support
10. **CSV** - structured text data
11. **Compression** - reduce string size

Lua pattern matching ที่ดีช่วยลดความซับซ้อนของโค้ดได้มาก แม้จะไม่รองรับ full regex แต่ก็เพียงพอสำหรับงานส่วนใหญ่
