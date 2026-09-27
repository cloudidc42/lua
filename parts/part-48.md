# บทที่ 48: HTTP Client และ Web Requests

## บทนำ

HTTP (HyperText Transfer Protocol) เป็นโปรโตคอลพื้นฐานของ World Wide Web ในบทนี้เราจะเรียนรู้การทำ HTTP requests จาก Lua ทั้งการใช้ LuaSocket โดยตรงและ library ระดับสูงอื่นๆ รวมถึงการจัดการ JSON, authentication, SSL/TLS และ patterns สำคัญสำหรับการใช้ REST APIs

## HTTP Protocol Basics

HTTP ทำงานแบบ request-response:
- **Request**: Method + URL + Headers + Body
- **Response**: Status Code + Headers + Body

### HTTP Methods หลัก
- **GET**: ดึงข้อมูล
- **POST**: ส่งข้อมูลใหม่
- **PUT**: อัปเดตทั้งหมด
- **PATCH**: อัปเดตบางส่วน
- **DELETE**: ลบข้อมูล
- **HEAD**: ดึงแค่ headers
- **OPTIONS**: ดูว่า server รองรับอะไร

---

## ตัวอย่างที่ 1: HTTP Request พื้นฐานด้วย LuaSocket

```lua
local socket = require("socket")

-- สร้าง raw HTTP request
local function raw_http_get(host, path, port)
    port = port or 80
    path = path or "/"

    local s = socket.tcp()
    s:settimeout(10)

    local ok, err = s:connect(host, port)
    if not ok then
        return nil, "Connection failed: " .. tostring(err)
    end

    -- ส่ง HTTP request
    local request = table.concat({
        "GET " .. path .. " HTTP/1.1",
        "Host: " .. host .. (port ~= 80 and ":" .. port or ""),
        "User-Agent: Lua/5.4",
        "Accept: */*",
        "Connection: close",
        "",
        "",
    }, "\r\n")

    s:send(request)

    -- รับ response
    local response = {}
    while true do
        local chunk, err2 = s:receive(4096)
        if chunk then
            table.insert(response, chunk)
        else
            break
        end
    end

    s:close()
    local full_response = table.concat(response)

    -- แยก header และ body
    local header_end = full_response:find("\r\n\r\n")
    if not header_end then
        return nil, "Invalid response"
    end

    local headers_raw = full_response:sub(1, header_end - 1)
    local body = full_response:sub(header_end + 4)

    -- Parse status line
    local status_line = headers_raw:match("^HTTP/[%d.]+ (%d+) ([^\r\n]*)")

    return {
        status  = tonumber(headers_raw:match("HTTP/[%d.]+ (%d+)")),
        headers = headers_raw,
        body    = body,
        status_line = status_line,
    }
end

-- ทดสอบ
print("=== Raw HTTP GET ===")
local resp, err = raw_http_get("httpbin.org", "/get")
if resp then
    print("Status:", resp.status)
    print("Body (first 300 chars):", resp.body:sub(1, 300))
else
    print("Error:", err)
end
```

---

## ตัวอย่างที่ 2: ใช้ socket.http สำหรับ GET

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- Simple GET request
local function get(url, headers)
    local response_chunks = {}
    local req_headers = {
        ["User-Agent"] = "LuaHTTP/1.0",
        ["Accept"]     = "application/json, text/html",
    }

    -- Merge custom headers
    for k, v in pairs(headers or {}) do
        req_headers[k] = v
    end

    local ok, status, resp_headers, status_line = http.request{
        url     = url,
        method  = "GET",
        headers = req_headers,
        sink    = ltn12.sink.table(response_chunks),
    }

    if not ok then
        return nil, "Request failed: " .. tostring(status)
    end

    return {
        ok       = status == 200,
        status   = status,
        headers  = resp_headers,
        body     = table.concat(response_chunks),
    }, nil
end

-- GET หลาย URLs
local urls = {
    "http://httpbin.org/get",
    "http://httpbin.org/ip",
    "http://httpbin.org/user-agent",
    "http://httpbin.org/headers",
}

print("=== Multiple GET Requests ===")
for _, url in ipairs(urls) do
    local resp, err = get(url)
    if resp then
        print(string.format("%-40s -> %d (%d bytes)",
            url, resp.status, #resp.body))
    else
        print(string.format("%-40s -> ERROR: %s", url, err))
    end
end
```

---

## ตัวอย่างที่ 3: POST Request

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")
local url   = require("socket.url")

-- POST with form data
local function post_form(url_str, form_data)
    -- Encode form data
    local parts = {}
    for k, v in pairs(form_data) do
        table.insert(parts,
            url.escape(tostring(k)) .. "=" .. url.escape(tostring(v)))
    end
    local body = table.concat(parts, "&")

    local response_chunks = {}
    local ok, status, headers = http.request{
        url    = url_str,
        method = "POST",
        headers = {
            ["Content-Type"]   = "application/x-www-form-urlencoded",
            ["Content-Length"] = #body,
            ["User-Agent"]     = "LuaHTTP/1.0",
        },
        source = ltn12.source.string(body),
        sink   = ltn12.sink.table(response_chunks),
    }

    return ok and {
        status  = status,
        body    = table.concat(response_chunks),
    } or nil, status
end

-- POST with JSON body
local function post_json(url_str, data_table)
    -- Simple JSON serialization (ใช้ของ basic)
    local function serialize(val, indent)
        indent = indent or 0
        local t = type(val)
        if t == "string" then
            return '"' .. val:gsub('"', '\\"'):gsub('\n', '\\n') .. '"'
        elseif t == "number" then
            return tostring(val)
        elseif t == "boolean" then
            return tostring(val)
        elseif val == nil then
            return "null"
        elseif t == "table" then
            -- Check if array
            local is_array = #val > 0
            if is_array then
                local items = {}
                for _, v in ipairs(val) do
                    table.insert(items, serialize(v))
                end
                return "[" .. table.concat(items, ",") .. "]"
            else
                local items = {}
                for k, v in pairs(val) do
                    table.insert(items,
                        '"' .. tostring(k) .. '":' .. serialize(v))
                end
                return "{" .. table.concat(items, ",") .. "}"
            end
        end
        return '"unknown"'
    end

    local body = serialize(data_table)
    local response_chunks = {}

    local ok, status, headers = http.request{
        url    = url_str,
        method = "POST",
        headers = {
            ["Content-Type"]   = "application/json",
            ["Content-Length"] = #body,
            ["Accept"]         = "application/json",
            ["User-Agent"]     = "LuaHTTP/1.0",
        },
        source = ltn12.source.string(body),
        sink   = ltn12.sink.table(response_chunks),
    }

    return ok and {
        status  = status,
        body    = table.concat(response_chunks),
    } or nil, status
end

-- ทดสอบ
print("=== POST Requests ===")

local r1, err1 = post_form("http://httpbin.org/post", {
    username = "lua_user",
    password = "secret123",
    action   = "login",
})

if r1 then
    print("Form POST status:", r1.status)
    print("Body:", r1.body:sub(1, 200))
else
    print("Form POST failed:", err1)
end

local r2, err2 = post_json("http://httpbin.org/post", {
    name    = "Lua Tutorial",
    version = "5.4",
    tags    = {"lua", "programming", "tutorial"},
    active  = true,
    score   = 98.5,
})

if r2 then
    print("\nJSON POST status:", r2.status)
    print("Body:", r2.body:sub(1, 200))
end
```

---

## ตัวอย่างที่ 4: PUT และ DELETE Requests

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- Generic request function
local function http_request(method, url_str, body, headers)
    local response_chunks = {}
    local req_headers = headers or {}
    req_headers["User-Agent"] = req_headers["User-Agent"] or "LuaHTTP/1.0"
    req_headers["Connection"] = "close"

    if body then
        req_headers["Content-Length"] = #body
    end

    local ok, status, resp_headers = http.request{
        url     = url_str,
        method  = method,
        headers = req_headers,
        source  = body and ltn12.source.string(body) or nil,
        sink    = ltn12.sink.table(response_chunks),
    }

    if not ok then
        return nil, string.format("HTTP %s failed: %s", method, tostring(status))
    end

    return {
        method  = method,
        status  = status,
        headers = resp_headers,
        body    = table.concat(response_chunks),
    }
end

-- REST API operations
local base_url = "http://jsonplaceholder.typicode.com"

print("=== REST API Operations ===")

-- GET - ดึง resource
local r_get = http_request("GET", base_url .. "/posts/1")
if r_get then
    print("GET /posts/1 ->", r_get.status)
    print(r_get.body:sub(1, 150))
end

-- POST - สร้าง resource ใหม่
local new_post = '{"title":"Lua Post","body":"Written in Lua!","userId":1}'
local r_post = http_request("POST", base_url .. "/posts", new_post, {
    ["Content-Type"] = "application/json",
})
if r_post then
    print("\nPOST /posts ->", r_post.status)
    print(r_post.body)
end

-- PUT - อัปเดตทั้ง resource
local updated = '{"id":1,"title":"Updated","body":"New content","userId":1}'
local r_put = http_request("PUT", base_url .. "/posts/1", updated, {
    ["Content-Type"] = "application/json",
})
if r_put then
    print("\nPUT /posts/1 ->", r_put.status)
    print(r_put.body:sub(1, 100))
end

-- PATCH - อัปเดตบางส่วน
local patch_data = '{"title":"Patched Title"}'
local r_patch = http_request("PATCH", base_url .. "/posts/1", patch_data, {
    ["Content-Type"] = "application/json",
})
if r_patch then
    print("\nPATCH /posts/1 ->", r_patch.status)
    print(r_patch.body:sub(1, 100))
end

-- DELETE
local r_delete = http_request("DELETE", base_url .. "/posts/1")
if r_delete then
    print("\nDELETE /posts/1 ->", r_delete.status)
end
```

---

## ตัวอย่างที่ 5: Query Parameters

```lua
local url = require("socket.url")
local http = require("socket.http")
local ltn12 = require("ltn12")

-- สร้าง URL พร้อม query parameters
local function build_url_with_params(base, params)
    local parts = {}
    for k, v in pairs(params) do
        if type(v) == "table" then
            -- Array parameter: key=val1&key=val2
            for _, item in ipairs(v) do
                table.insert(parts,
                    url.escape(tostring(k)) .. "=" ..
                    url.escape(tostring(item)))
            end
        else
            table.insert(parts,
                url.escape(tostring(k)) .. "=" ..
                url.escape(tostring(v)))
        end
    end

    if #parts > 0 then
        return base .. "?" .. table.concat(parts, "&")
    end
    return base
end

-- ทดสอบการสร้าง URL
print("=== Query Parameters ===")

local urls = {
    build_url_with_params("http://api.example.com/search", {
        q      = "lua programming",
        lang   = "th",
        limit  = 10,
        offset = 0,
    }),
    build_url_with_params("http://api.example.com/filter", {
        status = {"active", "pending"},
        sort   = "created_at",
        order  = "desc",
    }),
    build_url_with_params("http://api.example.com/data", {
        date_from = "2024-01-01",
        date_to   = "2024-12-31",
        format    = "json",
    }),
}

for _, u in ipairs(urls) do
    print(u)
end

-- Request with query params
local function get_with_params(base_url, params, headers)
    local full_url = build_url_with_params(base_url, params)
    local chunks = {}

    local ok, status, _ = http.request{
        url     = full_url,
        headers = headers or {["User-Agent"] = "LuaHTTP/1.0"},
        sink    = ltn12.sink.table(chunks),
    }

    return ok and {
        url    = full_url,
        status = status,
        body   = table.concat(chunks),
    } or nil
end

-- ทดสอบ
local result = get_with_params("http://httpbin.org/get", {
    key1  = "value1",
    key2  = "hello world",
    count = 42,
})

if result then
    print("\nURL:", result.url)
    print("Status:", result.status)
    print("Response (partial):", result.body:sub(1, 300))
end
```

---

## ตัวอย่างที่ 6: HTTP Headers

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- Header management
local function inspect_response_headers(url_str)
    local chunks = {}
    local ok, status, headers = http.request{
        url    = url_str,
        method = "HEAD",
        sink   = ltn12.sink.table(chunks),
    }

    if not ok then
        return nil, "Failed: " .. tostring(status)
    end

    return status, headers
end

-- ส่ง request พร้อม custom headers
local function request_with_custom_headers(url_str, extra_headers)
    local chunks = {}
    local headers = {
        ["User-Agent"]      = "Mozilla/5.0 (compatible; LuaBot/1.0)",
        ["Accept"]          = "application/json",
        ["Accept-Language"] = "th-TH,th;q=0.9,en;q=0.8",
        ["Accept-Encoding"] = "identity",
        ["Cache-Control"]   = "no-cache",
        ["Pragma"]          = "no-cache",
    }

    for k, v in pairs(extra_headers or {}) do
        headers[k] = v
    end

    local ok, status, resp_headers = http.request{
        url     = url_str,
        headers = headers,
        sink    = ltn12.sink.table(chunks),
    }

    return {
        ok      = ok ~= nil,
        status  = status,
        headers = resp_headers,
        body    = table.concat(chunks),
    }
end

-- ตรวจสอบ response headers
print("=== HTTP Headers ===")

local status, headers = inspect_response_headers("http://httpbin.org/get")
if headers then
    print("Response Headers from httpbin.org/get:")
    for name, value in pairs(headers) do
        print(string.format("  %-30s: %s", name, tostring(value)))
    end
end

-- ส่ง custom headers
local r = request_with_custom_headers("http://httpbin.org/headers", {
    ["X-API-Key"]     = "my-secret-key",
    ["X-Request-ID"]  = "req-" .. os.time(),
    ["X-Client-Name"] = "LuaApp-v1",
})

if r.ok then
    print("\nCustom headers sent, response status:", r.status)
    print("Body (shows what server received):", r.body:sub(1, 400))
end
```

---

## ตัวอย่างที่ 7: Basic Authentication

```lua
local http   = require("socket.http")
local ltn12  = require("ltn12")
local mime   = require("mime")

-- Basic Auth
local function basic_auth_request(url_str, username, password)
    local credentials = username .. ":" .. password
    local encoded = mime.b64(credentials)

    local chunks = {}
    local ok, status, headers = http.request{
        url  = url_str,
        headers = {
            ["Authorization"] = "Basic " .. encoded,
            ["User-Agent"]    = "LuaHTTP/1.0",
        },
        sink = ltn12.sink.table(chunks),
    }

    return {
        ok      = ok ~= nil,
        status  = status,
        headers = headers,
        body    = table.concat(chunks),
    }
end

-- Bearer Token Authentication
local function bearer_token_request(url_str, token, method, body)
    method = method or "GET"
    local chunks = {}
    local req_headers = {
        ["Authorization"] = "Bearer " .. token,
        ["User-Agent"]    = "LuaHTTP/1.0",
        ["Accept"]        = "application/json",
    }

    if body then
        req_headers["Content-Type"]   = "application/json"
        req_headers["Content-Length"] = #body
    end

    local ok, status, headers = http.request{
        url     = url_str,
        method  = method,
        headers = req_headers,
        source  = body and ltn12.source.string(body) or nil,
        sink    = ltn12.sink.table(chunks),
    }

    return {
        ok      = ok ~= nil,
        status  = status,
        headers = headers,
        body    = table.concat(chunks),
    }
end

-- API Key Authentication (header)
local function api_key_request(url_str, api_key, key_header)
    key_header = key_header or "X-API-Key"
    local chunks = {}

    local ok, status, headers = http.request{
        url     = url_str,
        headers = {
            [key_header]   = api_key,
            ["User-Agent"] = "LuaHTTP/1.0",
            ["Accept"]     = "application/json",
        },
        sink = ltn12.sink.table(chunks),
    }

    return {
        ok   = ok ~= nil,
        status  = status,
        body = table.concat(chunks),
    }
end

-- ทดสอบ
print("=== Authentication Examples ===")

-- Basic Auth test
local r1 = basic_auth_request(
    "http://httpbin.org/basic-auth/testuser/testpass",
    "testuser",
    "testpass"
)
print("Basic Auth status:", r1.status)
if r1.ok then
    print("Response:", r1.body)
end

-- Basic Auth ผิด
local r2 = basic_auth_request(
    "http://httpbin.org/basic-auth/testuser/testpass",
    "testuser",
    "wrongpass"
)
print("\nWrong password status:", r2.status)

-- Bearer Token
local r3 = bearer_token_request(
    "http://httpbin.org/bearer",
    "my-jwt-token-here"
)
print("\nBearer Token status:", r3.status)

-- API Key
local r4 = api_key_request(
    "http://httpbin.org/headers",
    "sk-abc123def456",
    "X-API-Key"
)
print("API Key status:", r4.status)
```

---

## ตัวอย่างที่ 8: JSON Handling

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- JSON encoder/decoder แบบง่าย
-- (ในการใช้งานจริงให้ใช้ library เช่น dkjson, cjson, rapidjson)

local JSON = {}

-- Simple encoder
function JSON.encode(val, indent, level)
    indent = indent or ""
    level  = level  or 0

    local t = type(val)

    if val == nil then
        return "null"
    elseif t == "boolean" then
        return tostring(val)
    elseif t == "number" then
        if val ~= val then return "null" end  -- NaN
        if val == math.huge or val == -math.huge then return "null" end
        if math.floor(val) == val and math.abs(val) < 2^53 then
            return string.format("%.0f", val)
        end
        return string.format("%.17g", val)
    elseif t == "string" then
        return '"' .. val
            :gsub('\\', '\\\\')
            :gsub('"', '\\"')
            :gsub('\n', '\\n')
            :gsub('\r', '\\r')
            :gsub('\t', '\\t')
            .. '"'
    elseif t == "table" then
        -- Check if array (consecutive integer keys from 1)
        local is_array = true
        local max_key = 0
        for k, _ in pairs(val) do
            if type(k) ~= "number" or k ~= math.floor(k) or k < 1 then
                is_array = false
                break
            end
            max_key = math.max(max_key, k)
        end
        is_array = is_array and (max_key == #val)

        if is_array then
            local items = {}
            for _, v in ipairs(val) do
                table.insert(items, JSON.encode(v, indent, level + 1))
            end
            return "[" .. table.concat(items, ",") .. "]"
        else
            local items = {}
            for k, v in pairs(val) do
                if type(k) == "string" or type(k) == "number" then
                    table.insert(items,
                        '"' .. tostring(k) .. '":' ..
                        JSON.encode(v, indent, level + 1))
                end
            end
            table.sort(items)
            return "{" .. table.concat(items, ",") .. "}"
        end
    end
    return '"[' .. t .. ']"'
end

-- Simple decoder (basic implementation)
function JSON.decode(str)
    local pos = 1

    local function skip_whitespace()
        while pos <= #str and str:sub(pos, pos):match("%s") do
            pos = pos + 1
        end
    end

    local function parse_value()
        skip_whitespace()
        local ch = str:sub(pos, pos)

        if ch == '"' then
            -- String
            pos = pos + 1
            local s = {}
            while pos <= #str do
                local c = str:sub(pos, pos)
                if c == '"' then
                    pos = pos + 1
                    break
                elseif c == '\\' then
                    pos = pos + 1
                    local esc = str:sub(pos, pos)
                    local escapes = {n="\n", r="\r", t="\t",
                                    ['"']='"', ['\\']='\\', ['/']=='/'}
                    table.insert(s, escapes[esc] or esc)
                else
                    table.insert(s, c)
                end
                pos = pos + 1
            end
            return table.concat(s)

        elseif ch == '{' then
            -- Object
            pos = pos + 1
            local obj = {}
            skip_whitespace()
            if str:sub(pos, pos) == '}' then
                pos = pos + 1
                return obj
            end
            while pos <= #str do
                skip_whitespace()
                local key = parse_value()
                skip_whitespace()
                pos = pos + 1  -- skip ':'
                skip_whitespace()
                local val = parse_value()
                obj[key] = val
                skip_whitespace()
                local sep = str:sub(pos, pos)
                pos = pos + 1
                if sep == '}' then break end
            end
            return obj

        elseif ch == '[' then
            -- Array
            pos = pos + 1
            local arr = {}
            skip_whitespace()
            if str:sub(pos, pos) == ']' then
                pos = pos + 1
                return arr
            end
            while pos <= #str do
                skip_whitespace()
                table.insert(arr, parse_value())
                skip_whitespace()
                local sep = str:sub(pos, pos)
                pos = pos + 1
                if sep == ']' then break end
            end
            return arr

        elseif str:sub(pos, pos + 3) == "true" then
            pos = pos + 4
            return true
        elseif str:sub(pos, pos + 4) == "false" then
            pos = pos + 5
            return false
        elseif str:sub(pos, pos + 3) == "null" then
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

        return nil
    end

    return parse_value()
end

-- ทดสอบ JSON
print("=== JSON Handling ===")

-- Encode
local data = {
    name    = "Lua Tutorial",
    version = 5.4,
    tags    = {"lua", "programming", "thai"},
    author  = {name = "สมชาย", email = "somchai@example.com"},
    active  = true,
    score   = nil,
}

local json_str = JSON.encode(data)
print("Encoded JSON:")
print(json_str)

-- Decode
local decoded = JSON.decode(json_str)
print("\nDecoded:")
print("  name:", decoded.name)
print("  version:", decoded.version)
print("  tags[1]:", decoded.tags and decoded.tags[1])
print("  author.name:", decoded.author and decoded.author.name)
print("  active:", decoded.active)

-- HTTP with JSON
local function json_get(url_str)
    local chunks = {}
    local ok, status, headers = http.request{
        url     = url_str,
        headers = {
            ["Accept"]     = "application/json",
            ["User-Agent"] = "LuaHTTP/1.0",
        },
        sink = ltn12.sink.table(chunks),
    }

    if not ok then return nil, status end

    local body = table.concat(chunks)
    local parsed = JSON.decode(body)

    return {
        status  = status,
        data    = parsed,
        raw     = body,
    }
end

local r = json_get("http://httpbin.org/json")
if r then
    print("\nHTTP JSON GET status:", r.status)
    if r.data then
        print("Parsed data type:", type(r.data))
    end
end
```

---

## ตัวอย่างที่ 9: File Upload (Multipart Form)

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- Multipart form data encoder
local function create_multipart(boundary, fields)
    local parts = {}
    local crlf = "\r\n"

    for _, field in ipairs(fields) do
        table.insert(parts, "--" .. boundary .. crlf)

        if field.filename then
            -- File field
            table.insert(parts, string.format(
                'Content-Disposition: form-data; name="%s"; filename="%s"%s',
                field.name, field.filename, crlf))
            table.insert(parts, "Content-Type: " ..
                (field.content_type or "application/octet-stream") .. crlf)
        else
            -- Text field
            table.insert(parts, string.format(
                'Content-Disposition: form-data; name="%s"%s',
                field.name, crlf))
        end

        table.insert(parts, crlf)
        table.insert(parts, field.data or "")
        table.insert(parts, crlf)
    end

    table.insert(parts, "--" .. boundary .. "--" .. crlf)
    return table.concat(parts)
end

-- Upload file
local function upload_file(url_str, file_path, field_name, extra_fields)
    -- อ่าน file
    local f = io.open(file_path, "rb")
    if not f then
        return nil, "Cannot open file: " .. file_path
    end
    local file_data = f:read("*a")
    f:close()

    local filename = file_path:match("[^/\\]+$") or "file"
    local boundary = "LuaBoundary" .. tostring(os.time())

    -- สร้าง fields
    local fields = {}

    -- Extra text fields
    for k, v in pairs(extra_fields or {}) do
        table.insert(fields, {name = k, data = tostring(v)})
    end

    -- File field
    table.insert(fields, {
        name         = field_name or "file",
        filename     = filename,
        data         = file_data,
        content_type = "application/octet-stream",
    })

    local body = create_multipart(boundary, fields)
    local chunks = {}

    local ok, status, headers = http.request{
        url    = url_str,
        method = "POST",
        headers = {
            ["Content-Type"]   = "multipart/form-data; boundary=" .. boundary,
            ["Content-Length"] = #body,
            ["User-Agent"]     = "LuaHTTP/1.0",
        },
        source = ltn12.source.string(body),
        sink   = ltn12.sink.table(chunks),
    }

    return ok and {
        status = status,
        body   = table.concat(chunks),
    } or nil, status
end

-- ทดสอบ multipart
print("=== File Upload (Multipart) ===")

-- สร้าง test file
local test_file = "/tmp/lua_upload_test.txt"
local f = io.open(test_file, "w")
f:write("This is a test file\nLine 2\nLine 3\n")
f:close()

local r, err = upload_file(
    "http://httpbin.org/post",
    test_file,
    "upload",
    {description = "Test upload from Lua", version = "1.0"}
)

if r then
    print("Upload status:", r.status)
    print("Response (partial):", r.body:sub(1, 400))
else
    print("Upload failed:", err)
end

-- แสดง multipart body ตัวอย่าง
print("\nMultipart body example:")
local example = create_multipart("BOUNDARY123", {
    {name = "name", data = "John Doe"},
    {name = "email", data = "john@example.com"},
    {name = "file", filename = "data.csv",
     data = "id,name,value\n1,test,100\n",
     content_type = "text/csv"},
})
print(example:sub(1, 400))
```

---

## ตัวอย่างที่ 10: Response Handling และ Status Codes

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- HTTP Status Code categories
local STATUS_CODES = {
    -- 1xx Informational
    [100] = "Continue",
    [101] = "Switching Protocols",
    -- 2xx Success
    [200] = "OK",
    [201] = "Created",
    [202] = "Accepted",
    [204] = "No Content",
    [206] = "Partial Content",
    -- 3xx Redirection
    [301] = "Moved Permanently",
    [302] = "Found",
    [304] = "Not Modified",
    [307] = "Temporary Redirect",
    [308] = "Permanent Redirect",
    -- 4xx Client Error
    [400] = "Bad Request",
    [401] = "Unauthorized",
    [403] = "Forbidden",
    [404] = "Not Found",
    [405] = "Method Not Allowed",
    [409] = "Conflict",
    [410] = "Gone",
    [422] = "Unprocessable Entity",
    [429] = "Too Many Requests",
    -- 5xx Server Error
    [500] = "Internal Server Error",
    [502] = "Bad Gateway",
    [503] = "Service Unavailable",
    [504] = "Gateway Timeout",
}

local function get_status_description(code)
    return STATUS_CODES[code] or "Unknown Status"
end

local function is_success(code)    return code >= 200 and code < 300 end
local function is_redirect(code)   return code >= 300 and code < 400 end
local function is_client_error(code) return code >= 400 and code < 500 end
local function is_server_error(code) return code >= 500 and code < 600 end

-- Response object
local function make_response(status, headers, body)
    return {
        status = status,
        status_text = get_status_description(status),
        headers = headers or {},
        body    = body or "",

        is_success     = is_success(status),
        is_redirect    = is_redirect(status),
        is_client_error = is_client_error(status),
        is_server_error = is_server_error(status),

        -- Helper methods
        json = function(self)
            -- Parse JSON body
            local ok, data = pcall(function()
                -- ใช้ JSON.decode จากตัวอย่างก่อนหน้า
                return self.body
            end)
            return ok and data or nil
        end,

        text = function(self)
            return self.body
        end,

        content_type = function(self)
            return self.headers["content-type"] or ""
        end,
    }
end

-- Request function ที่ return response object
local function fetch(url_str, options)
    options = options or {}
    local chunks = {}

    local ok, status, headers = http.request{
        url     = url_str,
        method  = options.method or "GET",
        headers = options.headers or {["User-Agent"] = "LuaFetch/1.0"},
        source  = options.body and ltn12.source.string(options.body) or nil,
        sink    = ltn12.sink.table(chunks),
    }

    if not ok then
        return nil, "Network error: " .. tostring(status)
    end

    return make_response(status, headers, table.concat(chunks))
end

-- ทดสอบ status codes
print("=== Status Code Handling ===")

local test_cases = {
    {url = "http://httpbin.org/status/200", expected = 200},
    {url = "http://httpbin.org/status/201", expected = 201},
    {url = "http://httpbin.org/status/404", expected = 404},
    {url = "http://httpbin.org/status/500", expected = 500},
}

for _, tc in ipairs(test_cases) do
    local r, err = fetch(tc.url)
    if r then
        local status_str = string.format("%d %s", r.status, r.status_text)
        local category = ""
        if r.is_success then category = "[SUCCESS]"
        elseif r.is_redirect then category = "[REDIRECT]"
        elseif r.is_client_error then category = "[CLIENT ERROR]"
        elseif r.is_server_error then category = "[SERVER ERROR]"
        end
        print(string.format("  %-45s -> %s %s",
            tc.url, status_str, category))
    else
        print(string.format("  %-45s -> ERROR: %s", tc.url, err))
    end
end
```

---

## ตัวอย่างที่ 11: Redirect Handling

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")
local url   = require("socket.url")

-- Follow redirects
local function request_with_redirects(initial_url, options, max_redirects)
    max_redirects = max_redirects or 10
    options = options or {}

    local current_url = initial_url
    local redirect_count = 0
    local history = {}

    while redirect_count <= max_redirects do
        local chunks = {}
        local ok, status, headers = http.request{
            url     = current_url,
            method  = redirect_count == 0 and (options.method or "GET") or "GET",
            headers = options.headers or {["User-Agent"] = "LuaHTTP/1.0"},
            sink    = ltn12.sink.table(chunks),
        }

        table.insert(history, {
            url    = current_url,
            status = status,
        })

        if not ok then
            return nil, "Request failed", history
        end

        -- ตรวจสอบว่าเป็น redirect
        if status >= 300 and status < 400 then
            local location = headers and headers["location"]
            if not location then
                return nil, "Redirect without Location header", history
            end

            -- Handle relative redirects
            if not location:match("^https?://") then
                -- Parse base URL
                local parsed = url.parse(current_url)
                if location:sub(1, 1) == "/" then
                    location = parsed.scheme .. "://" ..
                               parsed.host ..
                               (parsed.port and ":" .. parsed.port or "") ..
                               location
                else
                    local base_path = current_url:match("(.*/)") or ""
                    location = base_path .. location
                end
            end

            print(string.format("  Redirect %d: %d -> %s",
                redirect_count + 1, status, location))

            current_url = location
            redirect_count = redirect_count + 1
        else
            -- Final response
            return {
                status    = status,
                headers   = headers,
                body      = table.concat(chunks),
                url       = current_url,
                redirects = redirect_count,
                history   = history,
            }, nil
        end
    end

    return nil, "Max redirects exceeded", history
end

-- ทดสอบ redirect handling
print("=== Redirect Handling ===")

-- httpbin มี /redirect/{n} endpoint
local r, err, history = request_with_redirects(
    "http://httpbin.org/redirect/3"
)

if r then
    print(string.format("Final URL: %s", r.url))
    print(string.format("Final Status: %d", r.status))
    print(string.format("Redirects followed: %d", r.redirects))
    print("History:")
    for i, h in ipairs(r.history) do
        print(string.format("  %d. %d %s", i, h.status, h.url))
    end
else
    print("Error:", err)
    if history then
        print("Partial history:")
        for _, h in ipairs(history) do
            print(string.format("  %d %s", h.status, h.url))
        end
    end
end
```

---

## ตัวอย่างที่ 12: Rate Limiting และ Retry

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- Rate Limiter สำหรับ HTTP requests
local RateLimiter = {}
RateLimiter.__index = RateLimiter

function RateLimiter.new(requests_per_second)
    return setmetatable({
        rps       = requests_per_second,
        tokens    = requests_per_second,
        last_refill = socket.gettime(),
    }, RateLimiter)
end

function RateLimiter:wait()
    local now = socket.gettime()
    local elapsed = now - self.last_refill
    self.last_refill = now

    -- Refill tokens
    self.tokens = math.min(self.rps, self.tokens + elapsed * self.rps)

    if self.tokens < 1 then
        local wait_time = (1 - self.tokens) / self.rps
        socket.sleep(wait_time)
        self.tokens = 0
    else
        self.tokens = self.tokens - 1
    end
end

-- Retry with exponential backoff
local function fetch_with_retry(url_str, options)
    options = options or {}
    local max_retries   = options.max_retries   or 3
    local initial_delay = options.initial_delay or 1.0
    local max_delay     = options.max_delay     or 60.0
    local backoff_mult  = options.backoff_mult  or 2.0
    local retry_codes   = options.retry_codes   or {429, 500, 502, 503, 504}

    -- สร้าง Set สำหรับ retry codes
    local retry_set = {}
    for _, code in ipairs(retry_codes) do
        retry_set[code] = true
    end

    local delay = initial_delay

    for attempt = 1, max_retries + 1 do
        local chunks = {}
        local ok, status, headers = http.request{
            url     = url_str,
            headers = options.headers or {["User-Agent"] = "LuaHTTP/1.0"},
            sink    = ltn12.sink.table(chunks),
        }

        if ok and not retry_set[status] then
            return {
                status  = status,
                headers = headers,
                body    = table.concat(chunks),
                attempts = attempt,
            }
        end

        -- ตรวจสอบ Retry-After header (สำหรับ 429)
        local retry_after = headers and headers["retry-after"]
        local wait = delay

        if retry_after then
            local ra_seconds = tonumber(retry_after)
            if ra_seconds then
                wait = ra_seconds
            end
        end

        if attempt <= max_retries then
            print(string.format(
                "Attempt %d/%d: status=%s, retrying in %.1fs...",
                attempt, max_retries + 1, tostring(status), wait))
            socket.sleep(wait)
            delay = math.min(delay * backoff_mult, max_delay)
        else
            return nil, string.format(
                "Failed after %d attempts (last status: %s)",
                attempt, tostring(status))
        end
    end
end

-- ทดสอบ rate limiting และ retry
print("=== Rate Limiting & Retry ===")

local limiter = RateLimiter.new(2)  -- 2 requests per second

print("Making 5 rate-limited requests:")
local times = {}
for i = 1, 5 do
    local t = socket.gettime()
    limiter:wait()
    local elapsed = socket.gettime() - t

    -- Make request (simulated)
    print(string.format("  Request %d: waited %.3fs", i, elapsed))
end

-- Retry test
print("\nRetry with backoff test:")
local r, err = fetch_with_retry("http://httpbin.org/status/503", {
    max_retries   = 2,
    initial_delay = 0.5,
    retry_codes   = {503},
})

if r then
    print(string.format("Success after %d attempts: status=%d",
        r.attempts, r.status))
else
    print("Failed:", err)
end
```

---

## ตัวอย่างที่ 13: REST Client Class

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")
local url   = require("socket.url")

-- Comprehensive REST Client
local RESTClient = {}
RESTClient.__index = RESTClient

function RESTClient.new(base_url, options)
    options = options or {}
    return setmetatable({
        base_url    = base_url:gsub("/$", ""),  -- remove trailing slash
        timeout     = options.timeout or 30,
        headers     = options.headers or {},
        auth        = options.auth,
        max_retries = options.max_retries or 0,
        rate_limiter = options.rate_limiter,
        middleware  = options.middleware or {},
        log         = options.log or false,
    }, RESTClient)
end

function RESTClient:_request(method, path, options)
    options = options or {}

    -- สร้าง URL
    local full_url = self.base_url .. (path:sub(1,1) == "/" and path or "/" .. path)
    if options.params then
        local parts = {}
        for k, v in pairs(options.params) do
            table.insert(parts, url.escape(k) .. "=" .. url.escape(tostring(v)))
        end
        full_url = full_url .. "?" .. table.concat(parts, "&")
    end

    -- Headers
    local headers = {}
    for k, v in pairs(self.headers) do headers[k] = v end
    for k, v in pairs(options.headers or {}) do headers[k] = v end

    headers["User-Agent"] = headers["User-Agent"] or "LuaREST/1.0"
    headers["Accept"]     = headers["Accept"] or "application/json"

    -- Authentication
    if self.auth then
        if self.auth.type == "basic" then
            local mime = require("mime")
            local creds = self.auth.username .. ":" .. self.auth.password
            headers["Authorization"] = "Basic " .. mime.b64(creds)
        elseif self.auth.type == "bearer" then
            headers["Authorization"] = "Bearer " .. self.auth.token
        elseif self.auth.type == "api_key" then
            headers[self.auth.header or "X-API-Key"] = self.auth.key
        end
    end

    -- Body
    local body = nil
    if options.json then
        body = type(options.json) == "string" and options.json
               or '{"data": "json"}' -- simplified
        headers["Content-Type"]   = "application/json"
        headers["Content-Length"] = #body
    elseif options.form then
        local parts = {}
        for k, v in pairs(options.form) do
            table.insert(parts, url.escape(k) .. "=" .. url.escape(tostring(v)))
        end
        body = table.concat(parts, "&")
        headers["Content-Type"]   = "application/x-www-form-urlencoded"
        headers["Content-Length"] = #body
    elseif options.body then
        body = options.body
        headers["Content-Length"] = #body
    end

    if self.log then
        print(string.format("[REST] %s %s", method, full_url))
    end

    -- Execute request
    local chunks = {}
    local ok, status, resp_headers = http.request{
        url     = full_url,
        method  = method,
        headers = headers,
        source  = body and ltn12.source.string(body) or nil,
        sink    = ltn12.sink.table(chunks),
    }

    local resp_body = table.concat(chunks)

    if self.log then
        print(string.format("[REST] -> %s (%d bytes)", tostring(status), #resp_body))
    end

    if not ok then
        return nil, "Request failed: " .. tostring(status)
    end

    return {
        ok      = status >= 200 and status < 300,
        status  = status,
        headers = resp_headers,
        body    = resp_body,
        url     = full_url,
    }
end

-- HTTP method shortcuts
function RESTClient:get(path, options) return self:_request("GET", path, options) end
function RESTClient:post(path, options) return self:_request("POST", path, options) end
function RESTClient:put(path, options) return self:_request("PUT", path, options) end
function RESTClient:patch(path, options) return self:_request("PATCH", path, options) end
function RESTClient:delete(path, options) return self:_request("DELETE", path, options) end

function RESTClient:head(path, options)
    return self:_request("HEAD", path, options)
end

-- ตัวอย่างการใช้งาน
print("=== REST Client Class ===")

-- สร้าง client
local api = RESTClient.new("http://jsonplaceholder.typicode.com", {
    headers = {
        ["X-App-Version"] = "1.0",
    },
    log = true,
})

-- GET
local posts = api:get("/posts", {params = {_limit = 3}})
if posts and posts.ok then
    print("Posts fetched, status:", posts.status)
    print("Body length:", #posts.body)
end

-- POST
local new_post = api:post("/posts", {
    json = '{"title":"Test","body":"Content","userId":1}',
})
if new_post then
    print("New post created, status:", new_post.status)
end

-- DELETE
local deleted = api:delete("/posts/1")
if deleted then
    print("Post deleted, status:", deleted.status)
end
```

---

## ตัวอย่างที่ 14: API Wrapper Pattern

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- API Wrapper Pattern - ตัวอย่าง GitHub API wrapper
local GitHubAPI = {}
GitHubAPI.__index = GitHubAPI

function GitHubAPI.new(token)
    local api = setmetatable({}, GitHubAPI)
    api.base_url = "https://api.github.com"
    api.headers  = {
        ["Authorization"] = "Bearer " .. (token or ""),
        ["Accept"]        = "application/vnd.github.v3+json",
        ["User-Agent"]    = "LuaGitHubClient/1.0",
    }
    return api
end

function GitHubAPI:_get(path, params)
    local url_module = require("socket.url")
    local full_url = self.base_url .. path

    if params then
        local parts = {}
        for k, v in pairs(params) do
            table.insert(parts, url_module.escape(k) .. "=" ..
                url_module.escape(tostring(v)))
        end
        full_url = full_url .. "?" .. table.concat(parts, "&")
    end

    local chunks = {}
    local ok, status, headers = http.request{
        url     = full_url,
        headers = self.headers,
        sink    = ltn12.sink.table(chunks),
    }

    return ok and {
        status = status,
        body   = table.concat(chunks),
    } or nil
end

function GitHubAPI:get_user(username)
    return self:_get("/users/" .. username)
end

function GitHubAPI:get_repos(username, options)
    return self:_get("/users/" .. username .. "/repos", options)
end

function GitHubAPI:get_repo(owner, repo)
    return self:_get("/repos/" .. owner .. "/" .. repo)
end

function GitHubAPI:search_repos(query, options)
    options = options or {}
    options.q = query
    return self:_get("/search/repositories", options)
end

-- ตัวอย่างการใช้
print("=== GitHub API Wrapper ===")

local gh = GitHubAPI.new()  -- ไม่ใส่ token สำหรับ public endpoints

-- Get user info
local user = gh:get_user("torvalds")
if user then
    print("User API status:", user.status)
    print("Response length:", #user.body)
    -- Parse name from JSON (simplified)
    local name = user.body:match('"name":"([^"]+)"')
    if name then print("Name:", name) end
end

-- Search repos
local search = gh:search_repos("lua", {per_page = 3, sort = "stars"})
if search then
    print("\nSearch status:", search.status)
    print("Search response length:", #search.body)
end

-- Rate limit info (public endpoints จำกัดที่ 60/hour)
local rate = gh:_get("/rate_limit")
if rate and rate.status == 200 then
    local limit = rate.body:match('"limit":(%d+)')
    local remaining = rate.body:match('"remaining":(%d+)')
    print(string.format("\nAPI Rate Limit: %s remaining of %s",
        remaining or "?", limit or "?"))
end
```

---

## ตัวอย่างที่ 15: SSL/TLS HTTPS

```lua
-- LuaSocket รองรับ HTTPS ผ่าน luasec
-- ต้องติดตั้ง: luarocks install luasec

local https_available = false
local https

local ok, result = pcall(require, "ssl.https")
if ok then
    https = result
    https_available = true
    print("LuaSec (HTTPS) available")
else
    print("LuaSec ไม่พบ:", result)
    print("ติดตั้งด้วย: luarocks install luasec")
end

if https_available then
    local ltn12 = require("ltn12")

    -- HTTPS GET
    local function https_get(url_str)
        local chunks = {}
        local ok2, status, headers = https.request{
            url      = url_str,
            sink     = ltn12.sink.table(chunks),
            verify   = "peer",
            cafile   = "/etc/ssl/certs/ca-certificates.crt",
            protocol = "tlsv1_2",
        }

        return ok2 and {
            status  = status,
            headers = headers,
            body    = table.concat(chunks),
        } or nil, status
    end

    -- ทดสอบ HTTPS
    local r, err = https_get("https://httpbin.org/get")
    if r then
        print("HTTPS GET status:", r.status)
        -- SSL info
        if r.headers then
            print("Server:", r.headers["server"] or "unknown")
        end
    else
        print("HTTPS error:", err)
    end
end

-- สำหรับ Lua ที่ไม่มี LuaSec
-- ใช้ curl ผ่าน os.execute หรือ io.popen แทน

local function https_get_via_curl(url_str, options)
    options = options or {}
    local cmd_parts = {"curl", "-s"}

    -- Headers
    for k, v in pairs(options.headers or {}) do
        table.insert(cmd_parts, string.format("-H '%s: %s'", k, v))
    end

    -- Method
    if options.method and options.method ~= "GET" then
        table.insert(cmd_parts, "-X " .. options.method)
    end

    -- Body
    if options.body then
        table.insert(cmd_parts, string.format("-d '%s'", options.body))
    end

    -- Add URL
    table.insert(cmd_parts, string.format("'%s'", url_str))

    -- Output status code
    table.insert(cmd_parts, "-w '\\n%{http_code}'")

    local cmd = table.concat(cmd_parts, " ")
    local handle = io.popen(cmd)
    if not handle then return nil, "Cannot execute curl" end

    local output = handle:read("*a")
    handle:close()

    -- Parse status code (last line)
    local lines = {}
    for line in output:gmatch("[^\n]+") do
        table.insert(lines, line)
    end

    local status_line = table.remove(lines)
    local status_code = tonumber(status_line)
    local body = table.concat(lines, "\n")

    return {
        status = status_code or 0,
        body   = body,
    }
end

print("\n=== HTTPS via curl ===")
local r2 = https_get_via_curl("https://api.github.com", {
    headers = {["User-Agent"] = "LuaHTTPS/1.0"}
})

if r2 then
    print("Status:", r2.status)
    print("Body (first 200 chars):", r2.body:sub(1, 200))
end
```

---

## ตัวอย่างที่ 16: Streaming Response

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- Stream response ทีละ chunk
local function http_stream(url_str, chunk_callback, chunk_size)
    chunk_size = chunk_size or 4096

    local ok, status, headers = http.request{
        url     = url_str,
        headers = {["User-Agent"] = "LuaStream/1.0"},
        sink    = function(chunk, err)
            if chunk then
                return chunk_callback(chunk, nil)
            else
                return chunk_callback(nil, err)
            end
        end,
    }

    return ok and status or nil, not ok and status or nil
end

-- Download file with progress
local function download_file(url_str, output_path, progress_callback)
    local f = io.open(output_path, "wb")
    if not f then
        return nil, "Cannot create file: " .. output_path
    end

    local downloaded = 0
    local start_time = socket.gettime()

    local status, err = http_stream(url_str, function(chunk, err2)
        if chunk then
            f:write(chunk)
            downloaded = downloaded + #chunk

            if progress_callback then
                local elapsed = socket.gettime() - start_time
                local speed = elapsed > 0 and downloaded / elapsed or 0
                progress_callback(downloaded, speed)
            end
            return true
        end
        return true
    end)

    f:close()

    if not status then
        os.remove(output_path)
        return nil, err
    end

    return {
        bytes_downloaded = downloaded,
        time_elapsed     = socket.gettime() - start_time,
        path             = output_path,
        status           = status,
    }
end

-- ทดสอบ streaming
print("=== Streaming Response ===")

-- Stream และ count chunks
local chunk_count = 0
local total_bytes = 0

local status, err = http_stream(
    "http://httpbin.org/stream-bytes/10000",
    function(chunk)
        if chunk then
            chunk_count = chunk_count + 1
            total_bytes = total_bytes + #chunk
        end
        return true
    end
)

if status then
    print(string.format("Received %d bytes in %d chunks",
        total_bytes, chunk_count))
else
    print("Stream error:", err)
end

-- Download test
print("\nDownload test:")
local result, err2 = download_file(
    "http://httpbin.org/bytes/1024",
    "/tmp/lua_download_test.bin",
    function(bytes, speed)
        io.write(string.format("\rDownloaded: %d bytes (%.1f B/s)   ",
            bytes, speed))
        io.flush()
    end
)

if result then
    print(string.format("\nDownload complete: %d bytes in %.3fs",
        result.bytes_downloaded, result.time_elapsed))
else
    print("\nDownload failed:", err2)
end
```

---

## ตัวอย่างที่ 17: HTTP Caching

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- Simple HTTP Cache
local HTTPCache = {}
HTTPCache.__index = HTTPCache

function HTTPCache.new(options)
    options = options or {}
    return setmetatable({
        store     = {},
        max_size  = options.max_size or 100,
        max_age   = options.max_age or 300,  -- 5 minutes
        hit_count  = 0,
        miss_count = 0,
    }, HTTPCache)
end

function HTTPCache:_make_key(url_str, method)
    return (method or "GET") .. ":" .. url_str
end

function HTTPCache:get(url_str, method)
    local key = self:_make_key(url_str, method)
    local entry = self.store[key]

    if entry then
        local age = socket.gettime() - entry.time
        if age <= self.max_age then
            self.hit_count = self.hit_count + 1
            return entry.response, age
        else
            -- Expired
            self.store[key] = nil
        end
    end

    self.miss_count = self.miss_count + 1
    return nil
end

function HTTPCache:set(url_str, method, response)
    -- Evict old entries if at capacity
    local count = 0
    for _ in pairs(self.store) do count = count + 1 end

    if count >= self.max_size then
        -- LRU eviction (simplified: remove oldest)
        local oldest_key, oldest_time = nil, math.huge
        for k, v in pairs(self.store) do
            if v.time < oldest_time then
                oldest_time = v.time
                oldest_key  = k
            end
        end
        if oldest_key then
            self.store[oldest_key] = nil
        end
    end

    local key = self:_make_key(url_str, method)
    self.store[key] = {
        response = response,
        time     = socket.gettime(),
    }
end

function HTTPCache:stats()
    local count = 0
    for _ in pairs(self.store) do count = count + 1 end

    local total = self.hit_count + self.miss_count
    return {
        entries    = count,
        hits       = self.hit_count,
        misses     = self.miss_count,
        hit_rate   = total > 0 and (self.hit_count / total * 100) or 0,
    }
end

-- Cached HTTP client
local function make_cached_client(cache_options)
    local cache = HTTPCache.new(cache_options)

    return {
        get = function(url_str, force_refresh)
            -- Check cache
            if not force_refresh then
                local cached, age = cache:get(url_str, "GET")
                if cached then
                    print(string.format("  [CACHE HIT] %.1f seconds old: %s",
                        age, url_str))
                    return cached
                end
            end

            print("  [CACHE MISS] fetching:", url_str)
            local chunks = {}
            local ok, status, headers = http.request{
                url     = url_str,
                headers = {["User-Agent"] = "LuaCacheClient/1.0"},
                sink    = ltn12.sink.table(chunks),
            }

            if not ok then return nil end

            local response = {
                status  = status,
                headers = headers,
                body    = table.concat(chunks),
            }

            -- Cache only successful responses
            if status == 200 then
                cache:set(url_str, "GET", response)
            end

            return response
        end,

        stats = function()
            return cache:stats()
        end,
    }
end

-- ทดสอบ caching
print("=== HTTP Caching ===")

local client = make_cached_client({max_age = 60})

local urls_to_fetch = {
    "http://httpbin.org/get",
    "http://httpbin.org/ip",
    "http://httpbin.org/get",  -- should be cached
    "http://httpbin.org/ip",   -- should be cached
    "http://httpbin.org/user-agent",
}

for _, u in ipairs(urls_to_fetch) do
    local r = client.get(u)
    if r then
        print(string.format("    -> %d (%d bytes)", r.status, #r.body))
    end
end

local stats = client.stats()
print(string.format("\nCache stats: %d entries, %.1f%% hit rate",
    stats.entries, stats.hit_rate))
print(string.format("  Hits: %d, Misses: %d",
    stats.hits, stats.misses))
```

---

## ตัวอย่างที่ 18: Webhook Handler

```lua
local socket = require("socket")

-- Simple Webhook Server (รับ HTTP POST)
local WebhookServer = {}
WebhookServer.__index = WebhookServer

function WebhookServer.new(port, secret)
    local ws = setmetatable({}, WebhookServer)
    ws.port     = port
    ws.secret   = secret
    ws.handlers = {}

    ws.server = socket.tcp()
    ws.server:setoption("reuseaddr", true)
    ws.server:bind("*", port)
    ws.server:listen(10)
    ws.server:settimeout(0.1)

    return ws
end

function WebhookServer:on(event_type, handler)
    if not self.handlers[event_type] then
        self.handlers[event_type] = {}
    end
    table.insert(self.handlers[event_type], handler)
end

function WebhookServer:_parse_request(conn)
    conn:settimeout(5)
    local lines = {}
    local content_length = 0

    -- Read headers
    while true do
        local line = conn:receive("*l")
        if not line or line == "" then break end
        table.insert(lines, line)

        -- Extract Content-Length
        local cl = line:match("^[Cc]ontent%-[Ll]ength:%s*(%d+)")
        if cl then
            content_length = tonumber(cl)
        end
    end

    -- Parse method and path
    local method, path = lines[1] and
        lines[1]:match("^(%S+) (%S+)") or nil, nil

    -- Read body
    local body = ""
    if content_length > 0 then
        body = conn:receive(content_length) or ""
    end

    return method, path, body
end

function WebhookServer:_send_response(conn, status, body)
    local response = table.concat({
        "HTTP/1.1 " .. status,
        "Content-Type: application/json",
        "Content-Length: " .. #body,
        "Connection: close",
        "",
        body,
    }, "\r\n")
    conn:send(response)
end

function WebhookServer:handle_request(conn)
    local method, path, body = self:_parse_request(conn)

    if method == "POST" then
        -- Parse event type from path or body
        local event_type = path:match("/webhook/([^/]+)") or "unknown"

        local handlers = self.handlers[event_type] or
                         self.handlers["*"] or {}

        for _, handler in ipairs(handlers) do
            local ok, err = pcall(handler, {
                event = event_type,
                body  = body,
                path  = path,
            })
            if not ok then
                print("Handler error:", err)
            end
        end

        self:_send_response(conn, "200 OK", '{"status":"ok"}')
    else
        self:_send_response(conn, "405 Method Not Allowed",
            '{"error":"POST only"}')
    end
end

-- ทดสอบ webhook server
print("=== Webhook Server Example ===")

local ws = WebhookServer.new(19950, "secret123")

-- Register handlers
ws:on("payment", function(event)
    print("Payment webhook received!")
    print("  Body:", event.body:sub(1, 100))
end)

ws:on("user.signup", function(event)
    print("New user signup webhook!")
    print("  Path:", event.path)
end)

ws:on("*", function(event)
    print(string.format("Generic handler for event: %s", event.event))
end)

print("Webhook server configured on port", ws.port)
print("Handlers registered: payment, user.signup, *")
print("(Server not started - would block in real usage)")

-- Simulate webhook
local function simulate_webhook(server, path, body)
    -- สร้าง mock connection object
    local mock_conn = {
        data = {},
        received = path .. " " .. body,

        receive = function(self, pattern)
            if pattern == "*l" then
                local line = table.remove(self.data, 1)
                return line or ""
            else
                return string.rep("X", pattern)
            end
        end,

        send = function(self, data)
            print("Response sent:", data:sub(1, 50))
            return #data
        end,

        settimeout = function() end,
    }

    -- Populate mock data
    mock_conn.data = {
        "POST " .. path .. " HTTP/1.1",
        "Host: localhost",
        "Content-Type: application/json",
        "Content-Length: " .. #body,
        "",
    }

    -- Manual dispatch (since mock doesn't work perfectly)
    local event_type = path:match("/webhook/([^/]+)") or "unknown"
    local handlers = server.handlers[event_type] or
                     server.handlers["*"] or {}

    for _, handler in ipairs(handlers) do
        pcall(handler, {event = event_type, body = body, path = path})
    end
end

simulate_webhook(ws, "/webhook/payment",
    '{"amount":100,"currency":"THB","order_id":"ORD-123"}')
simulate_webhook(ws, "/webhook/user.signup",
    '{"email":"user@example.com","name":"สมชาย"}')
simulate_webhook(ws, "/webhook/unknown_event", '{}')
```

---

## ตัวอย่างที่ 19: HTTP Middleware Pattern

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")
local socket = require("socket")

-- Middleware Pattern สำหรับ HTTP Client
local function compose_middlewares(middlewares, final_handler)
    -- สร้าง chain จาก right to left
    local chain = final_handler
    for i = #middlewares, 1, -1 do
        local middleware = middlewares[i]
        local next_handler = chain
        chain = function(req)
            return middleware(req, next_handler)
        end
    end
    return chain
end

-- Middleware: Logging
local function logging_middleware(req, next)
    local start = socket.gettime()
    print(string.format("[LOG] %s %s", req.method or "GET", req.url))

    local response = next(req)

    local elapsed = (socket.gettime() - start) * 1000
    if response then
        print(string.format("[LOG] <- %d (%.1fms)",
            response.status or 0, elapsed))
    end
    return response
end

-- Middleware: Retry
local function retry_middleware(max_retries, retry_delay)
    return function(req, next)
        local attempts = 0
        while attempts < max_retries do
            attempts = attempts + 1
            local response = next(req)

            if response and response.status and
               response.status >= 200 and response.status < 500 then
                return response
            end

            if attempts < max_retries then
                print(string.format("[RETRY] Attempt %d/%d failed, waiting %.1fs",
                    attempts, max_retries, retry_delay))
                socket.sleep(retry_delay)
            end
        end
        return next(req)
    end
end

-- Middleware: Auth header injection
local function auth_middleware(token)
    return function(req, next)
        req.headers = req.headers or {}
        req.headers["Authorization"] = "Bearer " .. token
        return next(req)
    end
end

-- Middleware: Request validation
local function validation_middleware(req, next)
    if not req.url then
        return nil, "URL is required"
    end
    if not req.url:match("^https?://") then
        return nil, "URL must start with http:// or https://"
    end
    return next(req)
end

-- Core HTTP handler
local function http_handler(req)
    local chunks = {}
    local ok, status, headers = http.request{
        url     = req.url,
        method  = req.method or "GET",
        headers = req.headers or {},
        source  = req.body and ltn12.source.string(req.body) or nil,
        sink    = ltn12.sink.table(chunks),
    }

    return ok and {
        status  = status,
        headers = headers,
        body    = table.concat(chunks),
    } or {status = 0, body = "", error = tostring(status)}
end

-- สร้าง middleware stack
local middlewares = {
    validation_middleware,
    logging_middleware,
    auth_middleware("my-token-123"),
    retry_middleware(2, 0.5),
}

local execute = compose_middlewares(middlewares, http_handler)

-- ทดสอบ
print("=== HTTP Middleware Pattern ===")

-- Valid request
local r1 = execute{
    url    = "http://httpbin.org/get",
    method = "GET",
    headers = {["Accept"] = "application/json"},
}
if r1 then
    print("Result status:", r1.status)
end

-- Invalid request
local r2 = execute{url = "not-a-url", method = "GET"}
if r2 then
    print("Invalid URL result:", r2.error)
end
```

---

## ตัวอย่างที่ 20: Complete HTTP Utility Module

```lua
-- HTTP Utility Module สรุปทุก patterns

local HTTP = {}
local http  = require("socket.http")
local ltn12 = require("ltn12")
local url   = require("socket.url")
local socket = require("socket")

-- ================================
-- Core Request Function
-- ================================
function HTTP.request(options)
    assert(options.url, "URL is required")

    local method  = options.method or "GET"
    local headers = {
        ["User-Agent"] = "LuaHTTP/2.0",
        ["Connection"] = "close",
    }

    for k, v in pairs(options.headers or {}) do
        headers[k] = v
    end

    local body = nil
    if options.json and type(options.json) == "string" then
        body = options.json
        headers["Content-Type"]   = "application/json"
        headers["Content-Length"] = #body
    elseif options.form then
        local parts = {}
        for k, v in pairs(options.form) do
            table.insert(parts, url.escape(tostring(k)) .. "=" ..
                url.escape(tostring(v)))
        end
        body = table.concat(parts, "&")
        headers["Content-Type"]   = "application/x-www-form-urlencoded"
        headers["Content-Length"] = #body
    elseif options.body then
        body = options.body
        if not headers["Content-Length"] then
            headers["Content-Length"] = #body
        end
    end

    -- Query params
    local request_url = options.url
    if options.params then
        local parts = {}
        for k, v in pairs(options.params) do
            table.insert(parts, url.escape(tostring(k)) .. "=" ..
                url.escape(tostring(v)))
        end
        request_url = request_url .. "?" .. table.concat(parts, "&")
    end

    -- Auth
    if options.bearer_token then
        headers["Authorization"] = "Bearer " .. options.bearer_token
    elseif options.basic_auth then
        local mime = require("mime")
        local creds = options.basic_auth.user .. ":" ..
                      options.basic_auth.pass
        headers["Authorization"] = "Basic " .. mime.b64(creds)
    end

    local chunks = {}
    local timeout = options.timeout or 30

    local ok, status, resp_headers = http.request{
        url     = request_url,
        method  = method,
        headers = headers,
        source  = body and ltn12.source.string(body) or nil,
        sink    = ltn12.sink.table(chunks),
    }

    local resp_body = table.concat(chunks)

    return {
        ok      = ok ~= nil and status >= 200 and status < 300,
        status  = ok and status or 0,
        headers = resp_headers or {},
        body    = resp_body,
        url     = request_url,
        error   = not ok and tostring(status) or nil,
    }
end

-- Convenience methods
function HTTP.get(url_str, options)
    options = options or {}
    options.url = url_str
    options.method = "GET"
    return HTTP.request(options)
end

function HTTP.post(url_str, options)
    options = options or {}
    options.url = url_str
    options.method = "POST"
    return HTTP.request(options)
end

function HTTP.put(url_str, options)
    options = options or {}
    options.url = url_str
    options.method = "PUT"
    return HTTP.request(options)
end

function HTTP.delete(url_str, options)
    options = options or {}
    options.url = url_str
    options.method = "DELETE"
    return HTTP.request(options)
end

-- ทดสอบ utility module
print("=== HTTP Utility Module ===")

local r1 = HTTP.get("http://httpbin.org/get")
print("GET:", r1.status, "ok=" .. tostring(r1.ok))

local r2 = HTTP.post("http://httpbin.org/post", {
    form = {name = "Lua", version = "5.4"},
})
print("POST form:", r2.status)

local r3 = HTTP.post("http://httpbin.org/post", {
    json = '{"test": true, "value": 42}',
})
print("POST JSON:", r3.status)

local r4 = HTTP.get("http://httpbin.org/get", {
    params = {key = "value", page = 1},
    headers = {["X-Custom"] = "header-value"},
    bearer_token = "test-token",
})
print("GET with params:", r4.status, "url:", r4.url:sub(1, 60))
```

---

## สรุป HTTP Client และ Web Requests

HTTP Client ใน Lua มีหลายระดับ:

### 1. Low-level (LuaSocket TCP)
- ควบคุมได้มากที่สุด
- ต้องจัดการ HTTP protocol เอง
- ใช้สำหรับ custom protocols

### 2. socket.http
- HTTP/1.1 support
- ง่ายกว่า raw TCP
- รองรับ GET, POST, PUT, DELETE
- ไม่รองรับ HTTPS (ต้องใช้ luasec)

### 3. luasec (HTTPS)
- เพิ่ม TLS/SSL บน LuaSocket
- ต้องติดตั้งแยก
- `require("ssl.https")`

### Best Practices

```lua
-- 1. ตั้ง User-Agent เสมอ
headers["User-Agent"] = "MyApp/1.0"

-- 2. ตรวจสอบ status code
if resp.status >= 200 and resp.status < 300 then
    -- success
elseif resp.status == 401 then
    -- unauthorized
elseif resp.status == 429 then
    -- rate limited - wait and retry
end

-- 3. ใช้ ltn12.sink.table() สำหรับเก็บ response
local body_chunks = {}
http.request{sink = ltn12.sink.table(body_chunks)}
local body = table.concat(body_chunks)

-- 4. Set timeout
sock:settimeout(10)

-- 5. Handle errors
local ok, status, headers = http.request{...}
if not ok then
    print("Error:", status)  -- status = error message ในกรณีนี้
end
```
