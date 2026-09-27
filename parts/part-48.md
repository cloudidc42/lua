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

## ตัวอย่างที่ 21: OAuth 2.0 Client Credentials Flow

OAuth 2.0 สำหรับ machine-to-machine API authentication

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- Simple base64 encoder
local b64chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
local function base64_encode(data)
    local result = {}
    for i = 1, #data, 3 do
        local b1, b2, b3 = data:byte(i, i+2)
        b2 = b2 or 0; b3 = b3 or 0
        local n = b1*65536 + b2*256 + b3
        local c1 = math.floor(n/262144) % 64
        local c2 = math.floor(n/4096)   % 64
        local c3 = math.floor(n/64)     % 64
        local c4 = n % 64
        result[#result+1] = b64chars:sub(c1+1,c1+1)
        result[#result+1] = b64chars:sub(c2+1,c2+1)
        result[#result+1] = (b2==0 and i+1>#data) and "=" or b64chars:sub(c3+1,c3+1)
        result[#result+1] = (b3==0 and i+2>#data) and "=" or b64chars:sub(c4+1,c4+1)
    end
    return table.concat(result)
end

-- URL encode
local function url_encode(s)
    return s:gsub("([^%w%-%.%_%~ ])", function(c)
        return string.format("%%%02X", c:byte())
    end):gsub(" ", "+")
end

-- Token cache
local OAuthClient = {}
OAuthClient.__index = OAuthClient

function OAuthClient.new(token_url, client_id, client_secret, scope)
    return setmetatable({
        token_url     = token_url,
        client_id     = client_id,
        client_secret = client_secret,
        scope         = scope,
        token         = nil,
        expires_at    = 0,
    }, OAuthClient)
end

function OAuthClient:_fetch_token()
    local body = table.concat({
        "grant_type=client_credentials",
        "client_id="     .. url_encode(self.client_id),
        "client_secret=" .. url_encode(self.client_secret),
        self.scope and ("scope=" .. url_encode(self.scope)) or nil,
    }, "&")

    local auth = "Basic " .. base64_encode(self.client_id .. ":" .. self.client_secret)
    local chunks = {}
    local _, code = http.request {
        url    = self.token_url,
        method = "POST",
        headers = {
            ["Content-Type"]   = "application/x-www-form-urlencoded",
            ["Content-Length"] = #body,
            ["Authorization"]  = auth,
        },
        source = ltn12.source.string(body),
        sink   = ltn12.sink.table(chunks),
    }

    if code ~= 200 then
        return nil, "token request failed: HTTP " .. tostring(code)
    end

    local resp_body = table.concat(chunks)
    -- Parse JSON: extract access_token and expires_in
    local token   = resp_body:match('"access_token"%s*:%s*"([^"]+)"')
    local expires = resp_body:match('"expires_in"%s*:%s*(%d+)')

    if not token then return nil, "no access_token in response" end

    self.token      = token
    self.expires_at = os.time() + (tonumber(expires) or 3600) - 60  -- 60s buffer
    return token
end

function OAuthClient:get_token()
    if self.token and os.time() < self.expires_at then
        return self.token  -- cached
    end
    return self:_fetch_token()
end

function OAuthClient:authorized_request(url, opts)
    local token, err = self:get_token()
    if not token then return nil, err end

    opts = opts or {}
    opts.headers = opts.headers or {}
    opts.headers["Authorization"] = "Bearer " .. token
    opts.url = url

    local chunks = {}
    opts.sink = ltn12.sink.table(chunks)
    local _, code, headers = http.request(opts)
    return table.concat(chunks), code, headers
end

-- Demo
print("=== OAuth 2.0 Client Credentials ===")
local client = OAuthClient.new(
    "https://auth.example.com/oauth/token",
    "my-app-client-id",
    "my-app-secret",
    "read:api write:api"
)
print("Token URL:", client.token_url)
print("Client ID:", client.client_id)
print("Scope:", client.scope)

-- Show what the token request body looks like
local body_example = "grant_type=client_credentials&client_id=my-app-client-id&scope=read%3Aapi"
print("\nRequest body example:")
print("  " .. body_example)
local auth_header = "Basic " .. base64_encode("my-app-client-id:my-app-secret")
print("\nAuthorization header:")
print("  " .. auth_header:sub(1, 50) .. "...")

print("\nUsage:")
print("  token = client:get_token()  -- fetches or returns cached")
print("  body, code = client:authorized_request('https://api.example.com/data')")
print("  -- Automatically adds: Authorization: Bearer <token>")
```

---

## ตัวอย่างที่ 22: GraphQL Client

GraphQL queries และ mutations ผ่าน HTTP

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- Simple JSON encoder for GraphQL requests
local function json_encode(val, indent)
    local t = type(val)
    if t == "nil" then return "null"
    elseif t == "boolean" then return val and "true" or "false"
    elseif t == "number" then return tostring(val)
    elseif t == "string" then
        return '"' .. val:gsub('\\','\\\\'):gsub('"','\\"')
                        :gsub('\n','\\n'):gsub('\t','\\t') .. '"'
    elseif t == "table" then
        if val[1] ~= nil or next(val) == nil then
            local items = {}
            for _, v in ipairs(val) do table.insert(items, json_encode(v)) end
            return "[" .. table.concat(items, ",") .. "]"
        else
            local kvs = {}
            for k, v in pairs(val) do
                table.insert(kvs, json_encode(tostring(k)) .. ":" .. json_encode(v))
            end
            return "{" .. table.concat(kvs, ",") .. "}"
        end
    end
    return "null"
end

-- Simple JSON string value extractor
local function json_get(json, key)
    local val = json:match('"' .. key .. '"%s*:%s*"([^"]*)"')
    if val then return val end
    val = json:match('"' .. key .. '"%s*:%s*(%d+)')
    if val then return tonumber(val) end
    val = json:match('"' .. key .. '"%s*:%s*(true)')
    if val then return true end
    val = json:match('"' .. key .. '"%s*:%s*(false)')
    if val then return false end
end

local GraphQL = {}
GraphQL.__index = GraphQL

function GraphQL.new(endpoint, opts)
    opts = opts or {}
    return setmetatable({
        endpoint = endpoint,
        headers  = opts.headers or {},
        timeout  = opts.timeout or 10,
    }, GraphQL)
end

function GraphQL:set_auth(token)
    self.headers["Authorization"] = "Bearer " .. token
end

function GraphQL:execute(query, variables, operation_name)
    local payload = { query = query }
    if variables     then payload.variables     = variables     end
    if operation_name then payload.operationName = operation_name end

    local body = json_encode(payload)
    local hdrs = {
        ["Content-Type"]   = "application/json",
        ["Accept"]         = "application/json",
        ["Content-Length"] = #body,
    }
    for k, v in pairs(self.headers) do hdrs[k] = v end

    local chunks = {}
    local _, code = http.request {
        url     = self.endpoint,
        method  = "POST",
        headers = hdrs,
        source  = ltn12.source.string(body),
        sink    = ltn12.sink.table(chunks),
    }

    local resp = table.concat(chunks)
    return resp, code
end

function GraphQL:query(q, vars) return self:execute(q, vars) end

function GraphQL:mutation(m, vars) return self:execute(m, vars) end

-- Introspection helper
function GraphQL:introspect_type(type_name)
    local q = string.format([[
        query IntrospectType {
            __type(name: "%s") {
                name kind
                fields { name type { name kind ofType { name kind } } }
            }
        }
    ]], type_name)
    return self:query(q)
end

-- Query builder
local GQLQuery = {}
GQLQuery.__index = GQLQuery

function GQLQuery.new(op_type, op_name)
    return setmetatable({
        op_type = op_type or "query",
        op_name = op_name or "",
        fields  = {},
        vars    = {},
    }, GQLQuery)
end

function GQLQuery:field(name, args, subfields)
    local f = name
    if args and next(args) then
        local a = {}
        for k, v in pairs(args) do
            table.insert(a, k .. ": " .. json_encode(v))
        end
        f = f .. "(" .. table.concat(a, ", ") .. ")"
    end
    if subfields then
        f = f .. " { " .. table.concat(subfields, " ") .. " }"
    end
    table.insert(self.fields, f)
    return self
end

function GQLQuery:build()
    local body = table.concat(self.fields, "\n  ")
    return string.format("%s %s {\n  %s\n}", self.op_type, self.op_name, body)
end

-- Demo
print("=== GraphQL Client Demo ===")
local gql = GraphQL.new("https://api.example.com/graphql", {
    headers = { ["X-App-Name"] = "LuaTutorial" }
})
gql:set_auth("my-bearer-token")

-- Build a query
local q = GQLQuery.new("query", "GetUser")
q:field("user", { id = 42 }, { "id", "name", "email", "createdAt" })
q:field("me",   nil,         { "id", "role" })

local query_str = q:build()
print("Built query:")
print(query_str)

-- Mutation example
local mutation_str = [[
mutation CreatePost($input: PostInput!) {
    createPost(input: $input) {
        id
        title
        publishedAt
    }
}
]]
print("\nMutation:")
print(mutation_str)

local vars = { input = { title = "Lua Tutorial", content = "..." } }
print("Variables:", json_encode(vars))

print("\nRequest body:")
local req_body = json_encode({ query = mutation_str:gsub("%s+", " "), variables = vars })
print(req_body:sub(1, 200))
```

---

## ตัวอย่างที่ 23: HTTP Cookie Jar

Cookie management สำหรับ HTTP sessions

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- Parse Set-Cookie header
local function parse_cookie(header_val, request_host)
    local name, value = header_val:match("^([^=]+)=([^;]*)")
    if not name then return nil end
    name  = name:match("^%s*(.-)%s*$")
    value = value:match("^%s*(.-)%s*$")

    local cookie = { name=name, value=value, domain=request_host,
                     path="/", secure=false, http_only=false,
                     expires=nil, same_site="Lax" }

    for attr in header_val:gmatch(";%s*([^;]+)") do
        local k, v = attr:match("^([^=]+)=?(.*)")
        k = k and k:match("^%s*(.-)%s*$"):lower() or ""
        v = v and v:match("^%s*(.-)%s*$") or ""
        if k == "domain"   then cookie.domain    = v:gsub("^%.", "")
        elseif k == "path" then cookie.path      = v
        elseif k == "expires" then
            -- Parse expires (simplified)
            cookie.expires = v
        elseif k == "max-age" then
            local secs = tonumber(v)
            if secs then cookie.max_age_ts = os.time() + secs end
        elseif k == "secure"    then cookie.secure    = true
        elseif k == "httponly"  then cookie.http_only = true
        elseif k == "samesite"  then cookie.same_site = v
        end
    end
    return cookie
end

-- Cookie Jar
local CookieJar = {}
CookieJar.__index = CookieJar

function CookieJar.new()
    return setmetatable({ cookies = {} }, CookieJar)
end

function CookieJar:store(cookie)
    local key = (cookie.domain or "") .. "|" .. cookie.path .. "|" .. cookie.name
    self.cookies[key] = cookie
end

function CookieJar:process_response(headers, host)
    local sc = headers["set-cookie"]
    if not sc then return end
    if type(sc) == "string" then sc = { sc } end
    for _, val in ipairs(sc) do
        local c = parse_cookie(val, host)
        if c then self:store(c) end
    end
end

function CookieJar:get_header(host, path, secure)
    local matching = {}
    for _, c in pairs(self.cookies) do
        -- Domain match
        local dom_ok = c.domain == host or
            host:sub(-#c.domain) == c.domain
        -- Path match
        local path_ok = path:sub(1, #c.path) == c.path
        -- Secure match
        local sec_ok = not c.secure or (secure == true)
        -- Expiry
        local alive = not c.max_age_ts or os.time() < c.max_age_ts

        if dom_ok and path_ok and sec_ok and alive then
            table.insert(matching, c.name .. "=" .. c.value)
        end
    end
    if #matching == 0 then return nil end
    return table.concat(matching, "; ")
end

function CookieJar:count()
    local n = 0
    for _ in pairs(self.cookies) do n = n + 1 end
    return n
end

function CookieJar:clear(domain)
    if domain then
        for k, c in pairs(self.cookies) do
            if c.domain == domain then self.cookies[k] = nil end
        end
    else
        self.cookies = {}
    end
end

-- HTTP client with cookie jar
local function http_get_with_cookies(url, jar)
    local host = url:match("https?://([^/]+)")
    local path = url:match("https?://[^/]+(/.*)") or "/"
    local cookie_hdr = jar:get_header(host, path, false)

    local hdrs = { ["User-Agent"] = "LuaCookieJar/1.0" }
    if cookie_hdr then hdrs["Cookie"] = cookie_hdr end

    local chunks = {}
    local _, code, resp_hdrs = http.request {
        url = url, method = "GET",
        headers = hdrs,
        sink = ltn12.sink.table(chunks),
    }

    jar:process_response(resp_hdrs or {}, host)
    return table.concat(chunks), code
end

-- Demo
print("=== HTTP Cookie Jar ===")
local jar = CookieJar.new()

-- Simulate Set-Cookie headers
local fake_set_cookies = {
    "session_id=abc123xyz; Path=/; HttpOnly; SameSite=Strict; Max-Age=3600",
    "user_pref=dark_mode; Path=/; Max-Age=86400",
    "tracking=off; Domain=.example.com; Path=/; Secure",
    "cart_id=cart_987; Path=/shop; Max-Age=1800",
}

for _, sc in ipairs(fake_set_cookies) do
    local c = parse_cookie(sc, "example.com")
    if c then
        jar:store(c)
        print(string.format("  Stored: %s=%s (path=%s, secure=%s)",
            c.name, c.value:sub(1,15), c.path, tostring(c.secure)))
    end
end

print(string.format("\nJar has %d cookies", jar:count()))

-- Get header for request
local hdr = jar:get_header("example.com", "/", false)
print("\nCookie header for example.com/:")
print("  Cookie: " .. tostring(hdr):sub(1, 80))

local hdr2 = jar:get_header("example.com", "/shop", false)
print("\nCookie header for example.com/shop:")
print("  Cookie: " .. tostring(hdr2):sub(1, 80))

-- Clear a domain
jar:clear("example.com")
print(string.format("\nAfter clear: %d cookies", jar:count()))
```

---

## ตัวอย่างที่ 24: HTTP Response Streaming

อ่าน HTTP response แบบ streaming สำหรับ large files

```lua
local socket = require("socket")

-- Streaming HTTP downloader
local StreamDownloader = {}
StreamDownloader.__index = StreamDownloader

function StreamDownloader.new(opts)
    opts = opts or {}
    return setmetatable({
        timeout      = opts.timeout      or 30,
        chunk_size   = opts.chunk_size   or 8192,
        max_redirects= opts.max_redirects or 5,
        on_progress  = opts.on_progress,   -- callback(downloaded, total)
        on_chunk     = opts.on_chunk,       -- callback(chunk_data)
    }, StreamDownloader)
end

function StreamDownloader:_parse_url(url)
    local scheme, host, path = url:match("^(https?)://([^/]+)(.*)")
    path = (path == "" or not path) and "/" or path
    local port = (scheme == "https") and 443 or 80
    local h, p = host:match("^([^:]+):(%d+)$")
    if h then host, port = h, tonumber(p) end
    return scheme, host, port, path
end

function StreamDownloader:download(url, output_sink)
    local scheme, host, port, path = self:_parse_url(url)

    -- Connect
    local conn = socket.tcp()
    conn:settimeout(self.timeout)
    local ok, err = conn:connect(host, port)
    if not ok then return nil, "connect: " .. tostring(err) end

    -- Send request
    local req = string.format(
        "GET %s HTTP/1.1\r\nHost: %s\r\nConnection: close\r\n"..
        "Accept-Encoding: identity\r\nUser-Agent: LuaStream/1.0\r\n\r\n",
        path, host)
    conn:send(req)

    -- Read status line
    local status_line = conn:receive("*l")
    if not status_line then conn:close(); return nil, "no response" end
    local code = tonumber(status_line:match("HTTP/%S+%s+(%d+)"))

    -- Read headers
    local headers = {}
    local content_length = nil
    while true do
        local line = conn:receive("*l")
        if not line or line == "" then break end
        local k, v = line:match("^([^:]+):%s*(.*)")
        if k then
            k = k:lower()
            headers[k] = v
            if k == "content-length" then content_length = tonumber(v) end
        end
    end

    if code ~= 200 then
        conn:close()
        return nil, string.format("HTTP %d", code)
    end

    -- Stream body
    local downloaded = 0
    local is_chunked = (headers["transfer-encoding"] or ""):lower() == "chunked"

    if is_chunked then
        -- Read chunked transfer encoding
        while true do
            local size_line = conn:receive("*l")
            if not size_line then break end
            local chunk_size = tonumber(size_line:match("^%x+"), 16)
            if not chunk_size or chunk_size == 0 then break end
            local chunk = conn:receive(chunk_size)
            if chunk then
                downloaded = downloaded + #chunk
                if output_sink then output_sink(chunk) end
                if self.on_chunk then self.on_chunk(chunk) end
                if self.on_progress then
                    self.on_progress(downloaded, content_length)
                end
            end
            conn:receive(2)  -- CRLF after chunk
        end
    else
        -- Read by content-length or until close
        while true do
            local chunk, recv_err = conn:receive(self.chunk_size)
            if not chunk then
                if recv_err == "closed" then break end
                break
            end
            downloaded = downloaded + #chunk
            if output_sink then output_sink(chunk) end
            if self.on_chunk then self.on_chunk(chunk) end
            if self.on_progress and content_length then
                self.on_progress(downloaded, content_length)
            end
            if content_length and downloaded >= content_length then break end
        end
    end

    conn:close()
    return downloaded, nil, headers
end

-- Progress bar helper
local function make_progress_bar(width)
    width = width or 40
    return function(downloaded, total)
        if not total then
            io.write(string.format("\r  Downloaded: %d bytes", downloaded))
        else
            local pct = downloaded / total
            local filled = math.floor(pct * width)
            local bar = string.rep("=", filled) .. string.rep(" ", width - filled)
            io.write(string.format("\r  [%s] %d%% (%d/%d)",
                bar, math.floor(pct*100), downloaded, total))
        end
        io.flush()
    end
end

-- Demo
print("=== HTTP Response Streaming ===")
local downloader = StreamDownloader.new({
    chunk_size  = 4096,
    on_progress = make_progress_bar(30),
})

print("Downloading example.com...")
local chunks_received = {}
local bytes, err = downloader:download(
    "http://example.com/",
    function(chunk) table.insert(chunks_received, chunk) end
)

if bytes then
    print(string.format("\n  Downloaded %d bytes in %d chunks",
        bytes, #chunks_received))
else
    print("\n  Error: " .. tostring(err))
end

-- Streaming to file sink
local function file_sink(filename)
    local f, err = io.open(filename, "wb")
    if not f then return nil, err end
    return function(chunk)
        if chunk then f:write(chunk) else f:close() end
    end
end

print("\nFile download pattern:")
print("  sink = file_sink('/tmp/output.html')")
print("  bytes, err = downloader:download('http://example.com/', sink)")
print("  sink(nil)  -- close the file")
```

---

## ตัวอย่างที่ 25: URL Builder และ Parser

สร้างและ parse URLs อย่างถูกต้อง

```lua
-- URL encoding/decoding
local function url_encode(s)
    return s:gsub("([^%w%-%.%_%~%!%'%(%)%*])", function(c)
        return string.format("%%%02X", c:byte())
    end)
end

local function url_decode(s)
    return s:gsub("%%(%x%x)", function(hex)
        return string.char(tonumber(hex, 16))
    end):gsub("+", " ")
end

-- URL Parser
local function parse_url(url)
    local result = {}
    -- scheme
    result.scheme, url = url:match("^([%a][%w%.%+%-]*):(.*)")
    if not result.scheme then return nil, "no scheme" end
    -- authority
    if url:sub(1,2) == "//" then
        url = url:sub(3)
        local auth
        auth, url = url:match("^([^/?#]*)(.*)")
        -- userinfo
        local userinfo, hostport = auth:match("^([^@]+)@(.*)")
        if userinfo then
            result.user, result.password = userinfo:match("^([^:]+):?(.*)")
        else
            hostport = auth
        end
        -- host and port
        result.host, result.port = hostport:match("^%[([^%]]+)%]$")  -- IPv6
        if not result.host then
            result.host, result.port = hostport:match("^([^:]+):?(%d*)")
            result.port = result.port ~= "" and tonumber(result.port) or nil
        end
    end
    -- path, query, fragment
    result.path,  url = url:match("^([^?#]*)(.*)")
    result.query, url = url:match("^%?([^#]*)(.*)")
    result.fragment   = url:match("^#(.*)")
    -- Default ports
    if not result.port then
        result.port = ({ http=80, https=443, ftp=21, ssh=22 })[result.scheme]
    end
    return result
end

-- Query string builder/parser
local function build_query(params)
    local parts = {}
    -- Sort for deterministic output
    local keys = {}
    for k in pairs(params) do table.insert(keys, k) end
    table.sort(keys)
    for _, k in ipairs(keys) do
        local v = params[k]
        if type(v) == "table" then
            for _, item in ipairs(v) do
                table.insert(parts, url_encode(k) .. "=" .. url_encode(tostring(item)))
            end
        else
            table.insert(parts, url_encode(k) .. "=" .. url_encode(tostring(v)))
        end
    end
    return table.concat(parts, "&")
end

local function parse_query(qs)
    local params = {}
    for pair in (qs .. "&"):gmatch("([^&]+)&") do
        local k, v = pair:match("^([^=]*)=?(.*)")
        if k and k ~= "" then
            k = url_decode(k)
            v = url_decode(v or "")
            if params[k] then
                if type(params[k]) ~= "table" then
                    params[k] = { params[k] }
                end
                table.insert(params[k], v)
            else
                params[k] = v
            end
        end
    end
    return params
end

-- URL Builder
local URLBuilder = {}
URLBuilder.__index = URLBuilder

function URLBuilder.new(base)
    local self = setmetatable({ params = {} }, URLBuilder)
    if base then
        local parsed = parse_url(base)
        self.scheme = parsed and parsed.scheme or "http"
        self.host   = parsed and parsed.host
        self.port   = parsed and parsed.port
        self.path   = parsed and parsed.path or "/"
        if parsed and parsed.query then
            self.params = parse_query(parsed.query)
        end
    end
    return self
end

function URLBuilder:scheme(s) self.scheme = s; return self end
function URLBuilder:host(h)   self.host = h;   return self end
function URLBuilder:port(p)   self.port = p;   return self end
function URLBuilder:path(p)   self.path = p;   return self end
function URLBuilder:param(k, v)
    self.params[k] = v; return self
end
function URLBuilder:params_add(t)
    for k, v in pairs(t) do self.params[k] = v end
    return self
end

function URLBuilder:build()
    local parts = {}
    table.insert(parts, (self.scheme or "http") .. "://")
    table.insert(parts, self.host or "localhost")
    local default_ports = { http=80, https=443 }
    if self.port and self.port ~= default_ports[self.scheme] then
        table.insert(parts, ":" .. self.port)
    end
    table.insert(parts, self.path or "/")
    local qs = build_query(self.params)
    if qs ~= "" then table.insert(parts, "?" .. qs) end
    return table.concat(parts)
end

-- Demo
print("=== URL Builder & Parser ===")

local urls = {
    "https://user:pass@api.example.com:8443/v2/users?page=1&limit=20#section",
    "http://example.com/search?q=lua+tutorial&lang=th",
    "https://192.168.1.1:8080/admin",
    "ftp://files.example.com/pub/data.tar.gz",
}

for _, url in ipairs(urls) do
    local parsed = parse_url(url)
    if parsed then
        print(string.format("\nURL: %s", url))
        print(string.format("  scheme=%s host=%s port=%s path=%s",
            parsed.scheme or "?", parsed.host or "?",
            tostring(parsed.port), parsed.path or "/"))
        if parsed.query then
            local q = parse_query(parsed.query)
            for k, v in pairs(q) do
                print(string.format("  param: %s = %s", k, tostring(v)))
            end
        end
    end
end

-- Build URLs
print("\nBuilt URLs:")
local b1 = URLBuilder.new("https://api.example.com")
b1:path("/v1/search"):params_add({ q="lua 5.4", page=1, limit=10, sort="relevance" })
print("  " .. b1:build())

local b2 = URLBuilder.new("http://localhost:3000/api")
b2:param("token", "abc def ghi"):param("format", "json")
print("  " .. b2:build())
```

---

## ตัวอย่างที่ 26: HTTP Digest Authentication

Digest Auth ตาม RFC 7616 สำหรับ secure authentication

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- MD5 implementation (simplified hex digest for demo)
-- In production use a C library or LuaCrypto
local function fake_md5(s)
    -- This is NOT real MD5 - just a demo placeholder
    local h = 0
    for i = 1, #s do
        h = ((h * 31) + s:byte(i)) & 0xFFFFFFFF
    end
    return string.format("%08x%08x%08x%08x", h, ~h & 0xFFFFFFFF,
        (h * 7) & 0xFFFFFFFF, (h * 13) & 0xFFFFFFFF)
end

-- Parse WWW-Authenticate header
local function parse_digest_challenge(header)
    local challenge = {}
    -- Extract scheme
    local scheme, rest = header:match("^(%w+)%s+(.*)")
    challenge.scheme = scheme

    -- Parse key=value pairs (handles quoted and unquoted values)
    for key, val in rest:gmatch('(%w+)%s*=%s*"([^"]*)"') do
        challenge[key:lower()] = val
    end
    for key, val in rest:gmatch('(%w+)%s*=%s*([^,"%s]+)') do
        if not challenge[key:lower()] then
            challenge[key:lower()] = val
        end
    end
    return challenge
end

-- Build Digest Authorization header
local function build_digest_auth(challenge, method, uri, username, password, nc, cnonce)
    nc     = nc     or "00000001"
    cnonce = cnonce or string.format("%08x", os.time())

    local realm  = challenge.realm  or ""
    local nonce  = challenge.nonce  or ""
    local qop    = challenge.qop    -- may be nil

    -- HA1 = MD5(username:realm:password)
    local ha1 = fake_md5(username .. ":" .. realm .. ":" .. password)

    -- HA2 = MD5(method:uri)
    local ha2 = fake_md5(method .. ":" .. uri)

    -- Response
    local response
    if qop == "auth" or qop == "auth-int" then
        response = fake_md5(ha1 .. ":" .. nonce .. ":" .. nc ..
            ":" .. cnonce .. ":" .. qop .. ":" .. ha2)
    else
        response = fake_md5(ha1 .. ":" .. nonce .. ":" .. ha2)
    end

    local parts = {
        string.format('Digest username="%s"', username),
        string.format('realm="%s"', realm),
        string.format('nonce="%s"', nonce),
        string.format('uri="%s"', uri),
        string.format('response="%s"', response),
    }
    if challenge.algorithm then
        table.insert(parts, 'algorithm=' .. challenge.algorithm)
    end
    if qop then
        table.insert(parts, 'qop=' .. qop)
        table.insert(parts, 'nc=' .. nc)
        table.insert(parts, string.format('cnonce="%s"', cnonce))
    end
    if challenge.opaque then
        table.insert(parts, string.format('opaque="%s"', challenge.opaque))
    end

    return table.concat(parts, ", ")
end

-- HTTP client with Digest Auth support
local function http_digest_get(url, username, password)
    local chunks = {}

    -- First request: get the 401 challenge
    local _, code, headers = http.request {
        url = url, method = "GET",
        headers = { ["User-Agent"] = "LuaDigest/1.0" },
        sink = ltn12.sink.table(chunks),
    }

    if code ~= 401 then
        return table.concat(chunks), code  -- No auth required
    end

    local www_auth = headers and headers["www-authenticate"]
    if not www_auth then return nil, "no WWW-Authenticate header" end

    local challenge = parse_digest_challenge(www_auth)
    if challenge.scheme:lower() ~= "digest" then
        return nil, "not Digest auth: " .. challenge.scheme
    end

    -- Build path for uri
    local uri = url:match("https?://[^/]+(/.*)") or "/"

    -- Second request with auth
    local auth_header = build_digest_auth(challenge, "GET", uri, username, password)
    chunks = {}
    local _, code2, hdrs2 = http.request {
        url = url, method = "GET",
        headers = {
            ["Authorization"] = auth_header,
            ["User-Agent"]    = "LuaDigest/1.0",
        },
        sink = ltn12.sink.table(chunks),
    }

    return table.concat(chunks), code2
end

-- Demo
print("=== HTTP Digest Authentication ===")
print("RFC 7616 Digest Auth flow:")
print("  1. Client sends request without auth")
print("  2. Server responds 401 with WWW-Authenticate: Digest ...")
print("  3. Client computes HA1=MD5(user:realm:pass)")
print("  4. Client computes HA2=MD5(method:uri)")
print("  5. Client computes response=MD5(HA1:nonce:nc:cnonce:qop:HA2)")
print("  6. Client retries with Authorization: Digest ...")

-- Simulate challenge parsing
local example_header = 'Digest realm="api.example.com", nonce="dcd98b7102dd2f0e8b11d0f600bfb0c093", qop="auth", algorithm=MD5'
local challenge = parse_digest_challenge(example_header)
print("\nParsed challenge:")
for k, v in pairs(challenge) do
    if k ~= "scheme" then print(string.format("  %s = %s", k, v)) end
end

local auth = build_digest_auth(challenge, "GET", "/protected/resource", "alice", "password123")
print("\nGenerated Authorization header:")
print("  " .. auth:sub(1, 100) .. "...")
```

---

## ตัวอย่างที่ 27: HTTP Request Queue

Queue HTTP requests พร้อม concurrency control และ retry

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

local RequestQueue = {}
RequestQueue.__index = RequestQueue

function RequestQueue.new(opts)
    opts = opts or {}
    return setmetatable({
        queue        = {},
        results      = {},
        max_concurrent = opts.max_concurrent or 3,
        retry_count  = opts.retry_count or 3,
        retry_delay  = opts.retry_delay or 1,
        rate_limit   = opts.rate_per_second or 10,
        _last_req_t  = 0,
        _active      = 0,
        _completed   = 0,
        _errors      = 0,
    }, RequestQueue)
end

function RequestQueue:add(id, url, opts)
    table.insert(self.queue, {
        id      = id,
        url     = url,
        method  = opts and opts.method  or "GET",
        headers = opts and opts.headers or {},
        body    = opts and opts.body,
        retries = 0,
        priority= opts and opts.priority or 0,
    })
    -- Higher priority first
    table.sort(self.queue, function(a, b) return a.priority > b.priority end)
end

function RequestQueue:_rate_limit_wait()
    local min_interval = 1.0 / self.rate_limit
    local elapsed = socket.gettime() - self._last_req_t
    if elapsed < min_interval then
        socket.sleep(min_interval - elapsed)
    end
    self._last_req_t = socket.gettime()
end

function RequestQueue:_execute_one(req)
    self:_rate_limit_wait()
    local chunks = {}
    local ok, code, headers
    ok, code, headers = pcall(function()
        local resp_chunks = {}
        local _, c, h = http.request {
            url     = req.url,
            method  = req.method,
            headers = req.headers,
            source  = req.body and ltn12.source.string(req.body) or nil,
            sink    = ltn12.sink.table(resp_chunks),
        }
        return resp_chunks, c, h
    end)

    if ok then
        local resp_body = type(code) == "table" and table.concat(code) or ""
        return {
            id      = req.id,
            status  = "ok",
            code    = headers,  -- code is actually the table here
            body    = resp_body,
        }
    end
    return nil, tostring(code)
end

function RequestQueue:_do_request(req)
    self._active = self._active + 1
    local chunks = {}
    local result_code, result_hdrs
    local success, err_msg = pcall(function()
        _, result_code, result_hdrs = http.request {
            url     = req.url,
            method  = req.method,
            headers = req.headers,
            source  = req.body and ltn12.source.string(req.body) or nil,
            sink    = ltn12.sink.table(chunks),
        }
    end)
    self._active = math.max(0, self._active - 1)

    if not success or not result_code then
        req.retries = req.retries + 1
        if req.retries < self.retry_count then
            socket.sleep(self.retry_delay * req.retries)
            return self:_do_request(req)  -- retry
        end
        self._errors = self._errors + 1
        return { id=req.id, status="error", error=tostring(err_msg) }
    end

    self._completed = self._completed + 1
    return {
        id     = req.id,
        status = "ok",
        code   = result_code,
        body   = table.concat(chunks),
        retries= req.retries,
    }
end

function RequestQueue:run()
    local results = {}
    while #self.queue > 0 do
        -- Respect max_concurrent (simplified synchronous version)
        local req = table.remove(self.queue, 1)
        local result = self:_do_request(req)
        table.insert(results, result)
    end
    return results
end

function RequestQueue:stats()
    return {
        queued    = #self.queue,
        active    = self._active,
        completed = self._completed,
        errors    = self._errors,
    }
end

-- Demo
print("=== HTTP Request Queue ===")
local q = RequestQueue.new({
    max_concurrent  = 3,
    retry_count     = 2,
    retry_delay     = 0.5,
    rate_per_second = 5,
})

-- Add various requests
local requests = {
    { "req-1", "http://example.com/",             { priority=10 } },
    { "req-2", "http://example.com/about",        { priority=5  } },
    { "req-3", "http://httpbin.org/get",          { priority=1  } },
    { "req-4", "http://httpbin.org/status/404",   { priority=1  } },
    { "req-5", "http://nonexistent.invalid/path", { priority=0  } },
}

for _, r in ipairs(requests) do
    q:add(r[1], r[2], r[3])
    print(string.format("  Queued: %s -> %s (priority=%d)",
        r[1], r[2], (r[3] and r[3].priority or 0)))
end

local s = q:stats()
print(string.format("\nQueue stats: %d queued, %d active", s.queued, s.active))

print("\nProcessing queue (may take a moment)...")
local t0 = socket.gettime()
local results = q:run()
local elapsed = socket.gettime() - t0

print(string.format("Completed in %.2fs:", elapsed))
for _, r in ipairs(results) do
    local status_str = r.status == "ok"
        and string.format("HTTP %s", tostring(r.code))
        or  ("ERROR: " .. tostring(r.error):sub(1, 40))
    print(string.format("  [%s] %s retries=%d",
        r.id, status_str, r.retries or 0))
end

local final = q:stats()
print(string.format("\nFinal: completed=%d errors=%d",
    final.completed, final.errors))
```

---

## ตัวอย่างที่ 28: HTTP Mock Server for Testing

Mock HTTP server สำหรับ unit testing HTTP clients

```lua
local socket = require("socket")

-- Mock response definition
local MockServer = {}
MockServer.__index = MockServer

function MockServer.new(host, port)
    return setmetatable({
        host    = host or "127.0.0.1",
        port    = port or 18080,
        routes  = {},   -- { method, path_pattern, handler }
        log     = {},   -- request log
        server  = nil,
    }, MockServer)
end

function MockServer:on(method, path_pattern, handler)
    table.insert(self.routes, {
        method  = method:upper(),
        pattern = path_pattern,
        handler = handler,
    })
    return self  -- chainable
end

function MockServer:respond(status, body, headers)
    headers = headers or {}
    headers["Content-Type"]   = headers["Content-Type"]   or "application/json"
    headers["Content-Length"] = tostring(#body)
    headers["Server"]         = "LuaMockServer/1.0"
    headers["Connection"]     = "close"

    local status_text = ({
        [200]="OK",[201]="Created",[204]="No Content",
        [400]="Bad Request",[401]="Unauthorized",[403]="Forbidden",
        [404]="Not Found",[500]="Internal Server Error"
    })[status] or "Unknown"

    local lines = { string.format("HTTP/1.1 %d %s", status, status_text) }
    for k, v in pairs(headers) do
        table.insert(lines, k .. ": " .. v)
    end
    table.insert(lines, "")
    table.insert(lines, body)
    return table.concat(lines, "\r\n")
end

function MockServer:_find_route(method, path)
    for _, route in ipairs(self.routes) do
        if route.method == method then
            -- Simple pattern match (: prefix = wildcard segment)
            local pattern = "^" .. route.pattern:gsub(":([%w_]+)", "([^/]+)") .. "$"
            local captures = { path:match(pattern) }
            if #captures > 0 or path == route.pattern then
                return route, captures
            end
        end
    end
end

function MockServer:_handle_client(client)
    client:settimeout(2)
    local request_line = client:receive("*l")
    if not request_line then client:close(); return end

    local method, path = request_line:match("^(%u+)%s+([^%s]+)")
    if not method then client:close(); return end

    -- Read headers
    local req_headers = {}
    local content_length = 0
    while true do
        local line = client:receive("*l")
        if not line or line == "" then break end
        local k, v = line:match("^([^:]+):%s*(.*)")
        if k then
            k = k:lower()
            req_headers[k] = v
            if k == "content-length" then content_length = tonumber(v) or 0 end
        end
    end

    -- Read body
    local body = ""
    if content_length > 0 then
        body = client:receive(content_length) or ""
    end

    -- Log request
    table.insert(self.log, {
        method = method, path = path,
        headers = req_headers, body = body,
        time = os.time()
    })

    -- Find and execute route
    local route, captures = self:_find_route(method, path)
    local response
    if route then
        local req = { method=method, path=path, headers=req_headers, body=body }
        -- Inject path params
        req.params = {}
        local param_names = {}
        for pname in route.pattern:gmatch(":([%w_]+)") do
            table.insert(param_names, pname)
        end
        for i, name in ipairs(param_names) do
            req.params[name] = captures[i]
        end
        local ok, result = pcall(route.handler, req, self)
        if ok then
            response = result
        else
            response = self:respond(500, '{"error":"handler error"}')
        end
    else
        response = self:respond(404, string.format(
            '{"error":"not found","path":"%s"}', path))
    end

    client:send(response)
    client:close()
end

function MockServer:start()
    local srv = socket.tcp()
    srv:setoption("reuseaddr", true)
    local ok, err = srv:bind(self.host, self.port)
    if not ok then return nil, err end
    srv:listen(10)
    srv:settimeout(0)
    self.server = srv
    print(string.format("[MockServer] Listening on %s:%d", self.host, self.port))
    return true
end

function MockServer:serve_one(timeout)
    if not self.server then return nil, "not started" end
    self.server:settimeout(timeout or 0.1)
    local client, err = self.server:accept()
    if client then
        self:_handle_client(client)
        return true
    end
    return false, err
end

function MockServer:serve(iterations)
    for _ = 1, (iterations or 10) do
        self:serve_one(0.05)
    end
end

function MockServer:stop()
    if self.server then self.server:close(); self.server = nil end
end

function MockServer:request_count() return #self.log end
function MockServer:last_request()  return self.log[#self.log] end

-- Demo
print("=== HTTP Mock Server for Testing ===")
local mock = MockServer.new("127.0.0.1", 18082)

-- Setup routes
mock:on("GET",  "/health",       function(req, s) return s:respond(200, '{"status":"ok"}') end)
mock:on("GET",  "/users",        function(req, s) return s:respond(200,
    '[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]') end)
mock:on("GET",  "/users/:id",    function(req, s) return s:respond(200,
    string.format('{"id":%s,"name":"User%s"}', req.params.id, req.params.id)) end)
mock:on("POST", "/users",        function(req, s) return s:respond(201,
    '{"id":3,"created":true}', {["Location"]="/users/3"}) end)
mock:on("DELETE","/users/:id",   function(req, s) return s:respond(204, "") end)

local ok, err = mock:start()
if ok then
    -- Use LuaSocket to test it
    local http = require("socket.http")
    local ltn12 = require("ltn12")
    local base = "http://127.0.0.1:18082"

    local function test_request(method, path, expected_code)
        -- Prime the server for one request
        local t = socket.tcp()
        t:settimeout(1)
        t:connect("127.0.0.1", 18082)
        t:send(string.format("%s %s HTTP/1.1\r\nHost: localhost\r\n\r\n", method, path))
        local resp = t:receive("*a")
        t:close()
        mock:serve(1)
        local code = resp:match("HTTP/%S+ (%d+)")
        local ok2 = tonumber(code) == expected_code
        print(string.format("  [%s] %s %s -> HTTP %s %s",
            ok2 and "PASS" or "FAIL", method, path, tostring(code),
            ok2 and "" or "(expected " .. expected_code .. ")"))
    end

    -- Actually serve requests via serve_one in a simple loop
    local test_cases = {
        { "GET",    "/health",    200 },
        { "GET",    "/users",     200 },
        { "GET",    "/users/42",  200 },
        { "GET",    "/missing",   404 },
    }

    for _, tc in ipairs(test_cases) do
        -- Simplified test (just shows server is set up)
        local route, _ = mock:_find_route(tc[1], tc[2])
        local found = route ~= nil
        local would_be = found and 200 or 404
        print(string.format("  Route %s %s: %s -> expected %d",
            tc[1], tc[2], found and "FOUND" or "not found", tc[3]))
    end
    mock:stop()
    print("Mock server stopped. Requests logged: " .. mock:request_count())
else
    print("Server error: " .. tostring(err))
end
```

---

## ตัวอย่างที่ 29: HTTP Content Negotiation

Content negotiation ด้วย Accept headers

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- Parse Accept header into quality-sorted list
local function parse_accept(header)
    local types = {}
    for item in (header .. ","):gmatch("([^,]+),") do
        item = item:match("^%s*(.-)%s*$")
        local mime, q_part = item:match("^([^;]+);?(.*)")
        mime  = mime:match("^%s*(.-)%s*$")
        local q = 1.0
        if q_part then
            local qval = q_part:match("q%s*=%s*([%d%.]+)")
            if qval then q = tonumber(qval) or 1.0 end
        end
        table.insert(types, { mime=mime, q=q })
    end
    table.sort(types, function(a, b) return a.q > b.q end)
    return types
end

-- Select best matching content type
local function negotiate(accept_header, available)
    local accepted = parse_accept(accept_header or "*/*")
    for _, a in ipairs(accepted) do
        -- Exact match
        for _, avail in ipairs(available) do
            if a.mime == avail then return avail end
        end
        -- Wildcard match: text/* matches text/html
        local a_type, a_sub = a.mime:match("^([^/]+)/([^/]+)$")
        if a_sub == "*" then
            for _, avail in ipairs(available) do
                local av_type = avail:match("^([^/]+)/")
                if av_type == a_type then return avail end
            end
        end
        -- Full wildcard */*
        if a.mime == "*/*" and #available > 0 then
            return available[1]
        end
    end
    return nil  -- No acceptable type
end

-- Content renderer
local ContentNegotiator = {}
ContentNegotiator.__index = ContentNegotiator

function ContentNegotiator.new()
    return setmetatable({ renderers = {} }, ContentNegotiator)
end

function ContentNegotiator:add_renderer(content_type, fn)
    self.renderers[content_type] = fn
    return self
end

function ContentNegotiator:render(data, accept_header)
    local available = {}
    for ct in pairs(self.renderers) do
        table.insert(available, ct)
    end
    table.sort(available)  -- deterministic

    local ct = negotiate(accept_header, available)
    if not ct then
        return nil, nil, 406  -- Not Acceptable
    end

    local body = self.renderers[ct](data)
    return body, ct, 200
end

-- JSON encoder
local function to_json(val)
    local t = type(val)
    if t == "nil" then return "null"
    elseif t == "boolean" then return val and "true" or "false"
    elseif t == "number" then return tostring(val)
    elseif t == "string" then return '"' .. val:gsub('"', '\\"') .. '"'
    elseif t == "table" then
        if val[1] ~= nil then
            local items = {}
            for _, v in ipairs(val) do table.insert(items, to_json(v)) end
            return "[" .. table.concat(items, ",") .. "]"
        else
            local kvs = {}
            for k, v in pairs(val) do
                table.insert(kvs, '"'..k..'":'..to_json(v))
            end
            return "{" .. table.concat(kvs, ",") .. "}"
        end
    end
    return "null"
end

-- Demo
print("=== HTTP Content Negotiation ===")

-- Test Accept header parsing
local accept_headers = {
    "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "application/json",
    "application/json, text/plain; q=0.5",
    "*/*",
    "image/webp,image/apng,*/*;q=0.8",
}

for _, h in ipairs(accept_headers) do
    local parsed = parse_accept(h)
    print(string.format("\nAccept: %s", h:sub(1,60)))
    for _, p in ipairs(parsed) do
        print(string.format("  %s (q=%.2f)", p.mime, p.q))
    end
end

-- Content negotiator
local negotiator = ContentNegotiator.new()
negotiator:add_renderer("application/json", function(data) return to_json(data) end)
negotiator:add_renderer("text/plain", function(data)
    if type(data) == "table" then
        local lines = {}
        for k, v in pairs(data) do
            table.insert(lines, k .. ": " .. tostring(v))
        end
        return table.concat(lines, "\n")
    end
    return tostring(data)
end)
negotiator:add_renderer("text/html", function(data)
    if type(data) == "table" then
        local rows = {}
        for k, v in pairs(data) do
            table.insert(rows, string.format("<tr><td>%s</td><td>%s</td></tr>", k, v))
        end
        return "<table>" .. table.concat(rows) .. "</table>"
    end
    return "<p>" .. tostring(data) .. "</p>"
end)

local sample_data = { id=1, name="Alice", email="alice@example.com", role="admin" }

print("\n\nContent negotiation results:")
local test_accepts = {
    "application/json",
    "text/plain",
    "text/html",
    "image/png",  -- unsupported → 406
    "text/csv, application/json;q=0.5",
}

for _, acc in ipairs(test_accepts) do
    local body, ct, code = negotiator:render(sample_data, acc)
    print(string.format("\nAccept: %s  -> HTTP %d %s",
        acc:sub(1,40), code, ct or "(none)"))
    if body then print("  Body: " .. body:sub(1, 80)) end
end
```

---

## ตัวอย่างที่ 30: Complete API Client Framework

Framework สมบูรณ์สำหรับสร้าง API clients

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- JSON encoder/decoder (minimal)
local json = {}
function json.encode(val)
    local t = type(val)
    if t == "nil"     then return "null"
    elseif t == "boolean" then return val and "true" or "false"
    elseif t == "number"  then return tostring(val)
    elseif t == "string"  then
        return '"' .. val:gsub('\\','\\\\'):gsub('"','\\"')
                        :gsub('\n','\\n'):gsub('\r','\\r') .. '"'
    elseif t == "table" then
        if #val > 0 then
            local a = {}; for _, v in ipairs(val) do a[#a+1] = json.encode(v) end
            return "[" .. table.concat(a, ",") .. "]"
        else
            local o = {}; for k, v in pairs(val) do
                o[#o+1] = json.encode(tostring(k)) .. ":" .. json.encode(v)
            end
            return "{" .. table.concat(o, ",") .. "}"
        end
    end
    return "null"
end

function json.decode_simple(s)
    -- Very simplified: just extract top-level string values for demo
    local result = {}
    for k, v in s:gmatch('"([^"]+)"%s*:%s*"([^"]+)"') do result[k] = v end
    for k, v in s:gmatch('"([^"]+)"%s*:%s*(%d+)') do result[k] = tonumber(v) end
    return result
end

-- API Client
local APIClient = {}
APIClient.__index = APIClient

function APIClient.new(base_url, opts)
    opts = opts or {}
    return setmetatable({
        base_url   = base_url:gsub("/$", ""),
        timeout    = opts.timeout    or 10,
        headers    = opts.headers    or {},
        interceptors = { request={}, response={} },
        middlewares  = {},
        _req_count   = 0,
        _err_count   = 0,
    }, APIClient)
end

function APIClient:set_auth(token_or_type, token)
    if token then
        self.headers["Authorization"] = token_or_type .. " " .. token
    else
        self.headers["Authorization"] = "Bearer " .. token_or_type
    end
    return self
end

function APIClient:set_header(k, v) self.headers[k] = v; return self end

function APIClient:use(middleware)
    table.insert(self.middlewares, middleware)
    return self
end

function APIClient:on_request(fn)
    table.insert(self.interceptors.request, fn)
    return self
end

function APIClient:on_response(fn)
    table.insert(self.interceptors.response, fn)
    return self
end

function APIClient:request(method, path, opts)
    opts = opts or {}
    local url = self.base_url .. path

    -- Build headers
    local hdrs = {}
    for k, v in pairs(self.headers) do hdrs[k] = v end
    if opts.headers then
        for k, v in pairs(opts.headers) do hdrs[k] = v end
    end

    -- Encode body
    local body = opts.body
    if type(body) == "table" then
        body = json.encode(body)
        hdrs["Content-Type"] = "application/json"
    end
    if body then hdrs["Content-Length"] = #body end

    -- Request interceptors
    local req_ctx = { method=method, url=url, headers=hdrs, body=body }
    for _, fn in ipairs(self.interceptors.request) do
        local r = fn(req_ctx)
        if r then req_ctx = r end
    end

    -- Execute
    self._req_count = self._req_count + 1
    local chunks = {}
    local ok, code, resp_hdrs = pcall(function()
        return http.request {
            url     = req_ctx.url,
            method  = req_ctx.method,
            headers = req_ctx.headers,
            source  = req_ctx.body and ltn12.source.string(req_ctx.body) or nil,
            sink    = ltn12.sink.table(chunks),
        }
    end)

    if not ok then
        self._err_count = self._err_count + 1
        return nil, tostring(code)
    end

    local resp = {
        status  = code,
        headers = resp_hdrs or {},
        body    = table.concat(chunks),
        ok      = code and code >= 200 and code < 300,
    }

    -- Auto-decode JSON
    local ct = (resp.headers["content-type"] or "")
    if ct:find("application/json") then
        resp.json = json.decode_simple(resp.body)
    end

    -- Response interceptors
    for _, fn in ipairs(self.interceptors.response) do
        local r = fn(resp, req_ctx)
        if r then resp = r end
    end

    return resp
end

function APIClient:get(path, opts)    return self:request("GET",    path, opts) end
function APIClient:post(path, opts)   return self:request("POST",   path, opts) end
function APIClient:put(path, opts)    return self:request("PUT",    path, opts) end
function APIClient:delete(path, opts) return self:request("DELETE", path, opts) end
function APIClient:patch(path, opts)  return self:request("PATCH",  path, opts) end

function APIClient:stats()
    return { requests=self._req_count, errors=self._err_count }
end

-- Demo
print("=== Complete API Client Framework ===")
local api = APIClient.new("http://httpbin.org", {
    headers = { ["Accept"] = "application/json", ["X-Client"] = "LuaAPI/1.0" }
})

-- Add interceptors
api:on_request(function(req)
    req.headers["X-Request-ID"] = string.format("req-%d-%d",
        os.time(), math.random(1000))
    print(string.format("  [Request] %s %s", req.method, req.url))
    return req
end)

api:on_response(function(resp)
    print(string.format("  [Response] HTTP %s, body=%d bytes, ok=%s",
        tostring(resp.status), #resp.body, tostring(resp.ok)))
    return resp
end)

-- Set auth
api:set_auth("Bearer", "my-api-token-123")

-- Make requests
print("\nGET /get:")
local resp1 = api:get("/get")
if resp1 then
    print(string.format("  Status: %d, Body: %d bytes",
        resp1.status or 0, #resp1.body))
end

print("\nGET /status/404:")
local resp2 = api:get("/status/404")
if resp2 then
    print(string.format("  Status: %d, OK: %s", resp2.status or 0, tostring(resp2.ok)))
end

print("\nPOST /post with JSON body:")
local resp3 = api:post("/post", {
    body = { name="Alice", action="login", timestamp=os.time() }
})
if resp3 then
    print(string.format("  Status: %d, Body: %d bytes", resp3.status or 0, #resp3.body))
end

local stats = api:stats()
print(string.format("\nStats: %d requests, %d errors",
    stats.requests, stats.errors))
```

---

## ตัวอย่างที่ 31: HTTP Server-Sent Events Client

อ่าน Server-Sent Events (SSE) stream สำหรับ real-time data

```lua
local socket = require("socket")

-- SSE Event structure
local function parse_sse_event(lines)
    local event = { event="message", data="", id=nil, retry=nil }
    for _, line in ipairs(lines) do
        if line:sub(1,1) == ":" then
            -- comment, skip
        elseif line == "" then
            -- dispatch event (handled by caller)
        else
            local field, value = line:match("^([^:]+):?%s?(.*)")
            if field == "event" then event.event = value
            elseif field == "data"  then
                event.data = event.data ~= "" and (event.data .. "\n" .. value) or value
            elseif field == "id"    then event.id = value
            elseif field == "retry" then event.retry = tonumber(value)
            end
        end
    end
    return event
end

local SSEClient = {}
SSEClient.__index = SSEClient

function SSEClient.new(url, opts)
    opts = opts or {}
    local scheme, host, path = url:match("^(https?)://([^/]+)(.*)")
    path = (not path or path == "") and "/" or path
    local port = scheme == "https" and 443 or 80
    local h, p = host:match("^([^:]+):(%d+)$")
    if h then host, port = h, tonumber(p) end

    return setmetatable({
        host        = host,
        port        = port,
        path        = path,
        timeout     = opts.timeout or 30,
        last_id     = nil,
        handlers    = {},
        reconnect_ms= 3000,
        _sock       = nil,
    }, SSEClient)
end

function SSEClient:on(event_type, fn)
    self.handlers[event_type] = fn
    return self
end

function SSEClient:_connect()
    local sock = socket.tcp()
    sock:settimeout(self.timeout)
    local ok, err = sock:connect(self.host, self.port)
    if not ok then return nil, err end

    local hdrs = {
        string.format("GET %s HTTP/1.1", self.path),
        "Host: " .. self.host,
        "Accept: text/event-stream",
        "Cache-Control: no-cache",
        "Connection: keep-alive",
    }
    if self.last_id then
        table.insert(hdrs, "Last-Event-ID: " .. self.last_id)
    end
    table.insert(hdrs, ""); table.insert(hdrs, "")
    sock:send(table.concat(hdrs, "\r\n"))

    -- Read status
    local status = sock:receive("*l")
    local code = status and tonumber(status:match("HTTP/%S+%s+(%d+)"))
    if code ~= 200 then
        sock:close()
        return nil, string.format("HTTP %d", code or 0)
    end

    -- Read response headers
    while true do
        local line = sock:receive("*l")
        if not line or line == "" then break end
        local k, v = line:match("^([^:]+):%s*(.*)")
        if k and k:lower() == "retry" then
            self.reconnect_ms = tonumber(v) or self.reconnect_ms
        end
    end

    self._sock = sock
    return sock
end

function SSEClient:read_event()
    if not self._sock then return nil, "not connected" end
    local lines = {}
    self._sock:settimeout(self.timeout)
    while true do
        local line, err = self._sock:receive("*l")
        if not line then return nil, err end
        if line == "" then
            -- Empty line = event boundary
            if #lines > 0 then
                local event = parse_sse_event(lines)
                if event.id then self.last_id = event.id end
                return event
            end
        else
            table.insert(lines, line)
        end
    end
end

function SSEClient:close()
    if self._sock then self._sock:close(); self._sock = nil end
end

-- Demo (simulate SSE stream)
print("=== Server-Sent Events Client ===")
print("SSE stream format:")
local sse_stream = table.concat({
    "id: 1",
    "event: user_joined",
    "data: {\"user\":\"Alice\",\"room\":\"general\"}",
    "",
    "id: 2",
    "data: Hello everyone!",
    "",
    "id: 3",
    "event: user_left",
    "data: {\"user\":\"Bob\"}",
    "",
    ": keep-alive comment",
    "",
    "id: 4",
    "data: line1",
    "data: line2",
    "data: line3",
    "",
}, "\n")

print(sse_stream:sub(1, 200))

-- Parse events from stream
print("\nParsed events:")
local events = {}
local current = {}
for line in (sse_stream .. "\n"):gmatch("([^\n]*)\n") do
    if line == "" then
        if #current > 0 then
            table.insert(events, parse_sse_event(current))
            current = {}
        end
    elseif line:sub(1,1) ~= ":" then
        table.insert(current, line)
    end
end

for _, e in ipairs(events) do
    print(string.format("  [id=%s event=%s] data=%s",
        tostring(e.id), e.event, e.data:sub(1,50)))
end

-- Show client usage pattern
print("\nClient usage:")
print("  client = SSEClient.new('http://api.example.com/events')")
print("  client:on('user_joined', function(event) print(event.data) end)")
print("  sock = client:_connect()")
print("  while true do")
print("    local event = client:read_event()")
print("    local handler = client.handlers[event.event] or client.handlers['message']")
print("    if handler then handler(event) end")
print("  end")
```

---

## ตัวอย่างที่ 32: HTTP File Upload Progress

Upload files พร้อม progress tracking และ multipart streaming

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- Multipart form data builder
local MultipartUpload = {}
MultipartUpload.__index = MultipartUpload

function MultipartUpload.new()
    local boundary = string.format(
        "LuaBoundary%d%d", os.time(), math.random(100000, 999999))
    return setmetatable({
        boundary = boundary,
        parts    = {},
    }, MultipartUpload)
end

function MultipartUpload:add_field(name, value)
    table.insert(self.parts, {
        type    = "field",
        name    = name,
        value   = value,
    })
    return self
end

function MultipartUpload:add_file(name, filename, content, mime_type)
    table.insert(self.parts, {
        type      = "file",
        name      = name,
        filename  = filename,
        content   = content,
        mime_type = mime_type or "application/octet-stream",
    })
    return self
end

function MultipartUpload:content_type()
    return "multipart/form-data; boundary=" .. self.boundary
end

function MultipartUpload:build_body()
    local chunks = {}
    for _, part in ipairs(self.parts) do
        table.insert(chunks, "--" .. self.boundary .. "\r\n")
        if part.type == "field" then
            table.insert(chunks, string.format(
                'Content-Disposition: form-data; name="%s"\r\n\r\n%s\r\n',
                part.name, part.value))
        elseif part.type == "file" then
            table.insert(chunks, string.format(
                'Content-Disposition: form-data; name="%s"; filename="%s"\r\n',
                part.name, part.filename))
            table.insert(chunks, "Content-Type: " .. part.mime_type .. "\r\n\r\n")
            table.insert(chunks, part.content .. "\r\n")
        end
    end
    table.insert(chunks, "--" .. self.boundary .. "--\r\n")
    return table.concat(chunks)
end

function MultipartUpload:total_size()
    return #self:build_body()
end

-- Progress tracking source
local function progress_source(data, on_progress)
    local sent = 0
    local total = #data
    local chunk_size = 1024
    local pos = 1
    return function()
        if pos > #data then return nil end
        local chunk = data:sub(pos, pos + chunk_size - 1)
        pos = pos + #chunk
        sent = sent + #chunk
        if on_progress then on_progress(sent, total) end
        return chunk
    end
end

-- Upload function
local function upload_with_progress(url, multipart, on_progress)
    local body = multipart:build_body()
    local ct   = multipart:content_type()

    local chunks = {}
    local _, code, headers = http.request {
        url    = url,
        method = "POST",
        headers = {
            ["Content-Type"]   = ct,
            ["Content-Length"] = #body,
        },
        source = progress_source(body, on_progress),
        sink   = ltn12.sink.table(chunks),
    }
    return table.concat(chunks), code
end

-- Demo
print("=== HTTP File Upload with Progress ===")
local upload = MultipartUpload.new()
upload:add_field("user_id", "12345")
upload:add_field("description", "Lua tutorial screenshots")
upload:add_file("avatar", "profile.jpg",
    string.rep("JPEG_DATA_", 1000),  -- simulate 10KB image
    "image/jpeg")
upload:add_file("document", "tutorial.pdf",
    string.rep("PDF_CONTENT_", 2000),  -- simulate 24KB PDF
    "application/pdf")

local total_size = upload:total_size()
print(string.format("Upload: boundary=%s", upload.boundary:sub(1,20) .. "..."))
print(string.format("Parts: %d, Total size: %d bytes (%.1f KB)",
    #upload.parts, total_size, total_size/1024))

-- Show body structure
local body = upload:build_body()
local first_200 = body:sub(1, 200)
print("\nBody preview:")
print(first_200:gsub("\r\n", "\\r\\n\n"))

-- Progress bar
local last_pct = -1
local function on_upload_progress(sent, total)
    local pct = math.floor(sent / total * 100)
    if pct ~= last_pct and pct % 10 == 0 then
        last_pct = pct
        local filled = math.floor(pct / 5)
        io.write(string.format("\r  Uploading: [%s%s] %d%% (%d/%d bytes)",
            string.rep("#", filled), string.rep(".", 20-filled),
            pct, sent, total))
        io.flush()
    end
end

-- Simulate upload progress (local)
print("\nSimulating upload progress:")
local body_data = upload:build_body()
local source = progress_source(body_data, on_upload_progress)
local total_read = 0
while true do
    local chunk = source()
    if not chunk then break end
    total_read = total_read + #chunk
end
print(string.format("\n  Uploaded %d bytes total", total_read))
```

---

## ตัวอย่างที่ 33: HTTP Proxy Implementation

Simple HTTP proxy สำหรับ intercept และ log requests

```lua
local socket = require("socket")

-- HTTP Proxy
local HTTPProxy = {}
HTTPProxy.__index = HTTPProxy

function HTTPProxy.new(host, port)
    return setmetatable({
        host = host or "127.0.0.1",
        port = port or 8888,
        log  = {},
        rules = {},  -- { pattern, action } action="block"|"allow"|fn
        server = nil,
    }, HTTPProxy)
end

function HTTPProxy:allow(url_pattern) table.insert(self.rules, { url_pattern, "allow" }) end
function HTTPProxy:block(url_pattern) table.insert(self.rules, { url_pattern, "block" }) end
function HTTPProxy:rewrite(url_pattern, fn) table.insert(self.rules, { url_pattern, fn }) end

function HTTPProxy:_check_rules(url)
    for _, rule in ipairs(self.rules) do
        if url:match(rule[1]) then
            return rule[2]
        end
    end
    return "allow"  -- default allow
end

function HTTPProxy:_forward_request(method, url, headers, body)
    -- Parse URL
    local scheme = url:match("^(https?)://") or "http"
    local host_port = url:match("^https?://([^/]+)")
    local path = url:match("^https?://[^/]+(.*)") or "/"
    if path == "" then path = "/" end

    local host, port = host_port:match("^([^:]+):?(%d*)$")
    port = (port and port ~= "") and tonumber(port) or
        (scheme == "https" and 443 or 80)

    local conn = socket.tcp()
    conn:settimeout(10)
    local ok, err = conn:connect(host, port)
    if not ok then return nil, nil, "connect: " .. tostring(err) end

    -- Forward request
    local req_lines = { string.format("%s %s HTTP/1.0", method, path) }
    for k, v in pairs(headers) do
        if k:lower() ~= "proxy-connection" then
            table.insert(req_lines, k .. ": " .. v)
        end
    end
    table.insert(req_lines, "Connection: close")
    table.insert(req_lines, ""); table.insert(req_lines, "")
    conn:send(table.concat(req_lines, "\r\n"))
    if body and #body > 0 then conn:send(body) end

    -- Read response
    local status_line = conn:receive("*l") or ""
    local code = tonumber(status_line:match("HTTP/%S+%s+(%d+)")) or 502
    local resp_hdrs = {}
    while true do
        local line = conn:receive("*l")
        if not line or line == "" then break end
        local k, v = line:match("^([^:]+):%s*(.*)")
        if k then resp_hdrs[k:lower()] = v end
    end
    local resp_body = conn:receive("*a") or ""
    conn:close()
    return code, resp_hdrs, resp_body
end

function HTTPProxy:_handle(client)
    client:settimeout(5)
    local request_line = client:receive("*l")
    if not request_line then client:close(); return end

    local method, url = request_line:match("^(%u+)%s+(%S+)")
    if not method then client:close(); return end

    -- Read client headers
    local headers = { host = url:match("^https?://([^/:]+)") or "" }
    local body_len = 0
    while true do
        local line = client:receive("*l")
        if not line or line == "" then break end
        local k, v = line:match("^([^:]+):%s*(.*)")
        if k then
            k = k:lower()
            headers[k] = v
            if k == "content-length" then body_len = tonumber(v) or 0 end
        end
    end
    local body = body_len > 0 and (client:receive(body_len) or "") or ""

    -- Log
    local log_entry = { method=method, url=url, time=os.time(), code=nil }
    table.insert(self.log, log_entry)

    -- Check rules
    local action = self:_check_rules(url)
    if action == "block" then
        client:send("HTTP/1.0 403 Forbidden\r\nContent-Length: 9\r\n\r\nBlocked.")
        log_entry.code = 403
        client:close()
        return
    end

    -- Handle CONNECT (HTTPS tunneling) — simplified
    if method == "CONNECT" then
        client:send("HTTP/1.0 200 Connection established\r\n\r\n")
        log_entry.code = 200
        client:close()  -- In real proxy, would tunnel here
        return
    end

    -- Forward
    local code, resp_hdrs, resp_body = self:_forward_request(method, url, headers, body)
    log_entry.code = code

    if not code then
        client:send("HTTP/1.0 502 Bad Gateway\r\nContent-Length: 11\r\n\r\nBad Gateway")
        client:close()
        return
    end

    -- Build response
    local resp_lines = { string.format("HTTP/1.0 %d OK", code) }
    for k, v in pairs(resp_hdrs) do
        table.insert(resp_lines, k .. ": " .. v)
    end
    if not resp_hdrs["content-length"] then
        table.insert(resp_lines, "Content-Length: " .. #resp_body)
    end
    table.insert(resp_lines, ""); table.insert(resp_lines, "")
    client:send(table.concat(resp_lines, "\r\n"))
    if #resp_body > 0 then client:send(resp_body) end
    client:close()
end

function HTTPProxy:start()
    local srv = socket.tcp()
    srv:setoption("reuseaddr", true)
    local ok, err = srv:bind(self.host, self.port)
    if not ok then return nil, err end
    srv:listen(50)
    srv:settimeout(0)
    self.server = srv
    print(string.format("[Proxy] Listening on %s:%d", self.host, self.port))
    return true
end

function HTTPProxy:serve(iterations)
    for _ = 1, (iterations or 5) do
        self.server:settimeout(0.05)
        local client = self.server:accept()
        if client then self:_handle(client) end
    end
end

function HTTPProxy:stop()
    if self.server then self.server:close(); self.server = nil end
end

-- Demo
print("=== HTTP Proxy Implementation ===")
local proxy = HTTPProxy.new("127.0.0.1", 18090)
proxy:block("%.ads%.example%.com")
proxy:block("/malware/")

local ok2, err = proxy:start()
if ok2 then
    print("Proxy started on 127.0.0.1:18090")
    print("Rules configured:")
    print("  BLOCK: *.ads.example.com")
    print("  BLOCK: /malware/")
    proxy:serve(2)
    proxy:stop()
    print(string.format("Proxy stopped. Requests logged: %d", #proxy.log))
else
    print("Error: " .. tostring(err))
end

print("\nUsage: configure browser/app to use proxy 127.0.0.1:8888")
print("All HTTP traffic flows through the proxy for:")
print("  - Logging and debugging")
print("  - Content filtering")
print("  - Caching")
print("  - Load balancing")
```

---

## ตัวอย่างที่ 34: HTTP API Rate Limiter

Rate limiting สำหรับ outbound API calls

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- Token Bucket rate limiter
local TokenBucket = {}
TokenBucket.__index = TokenBucket

function TokenBucket.new(rate, capacity)
    return setmetatable({
        rate      = rate,      -- tokens per second
        capacity  = capacity,  -- max tokens
        tokens    = capacity,
        last_time = socket.gettime(),
    }, TokenBucket)
end

function TokenBucket:_refill()
    local now = socket.gettime()
    local elapsed = now - self.last_time
    self.tokens = math.min(self.capacity, self.tokens + elapsed * self.rate)
    self.last_time = now
end

function TokenBucket:consume(cost)
    self:_refill()
    cost = cost or 1
    if self.tokens >= cost then
        self.tokens = self.tokens - cost
        return true, 0
    end
    -- Calculate wait time
    local wait = (cost - self.tokens) / self.rate
    return false, wait
end

function TokenBucket:wait_and_consume(cost)
    local ok, wait_s = self:consume(cost)
    if not ok then
        socket.sleep(wait_s)
        self:_refill()
        self.tokens = math.max(0, self.tokens - (cost or 1))
    end
    return true
end

-- Sliding Window rate limiter
local SlidingWindow = {}
SlidingWindow.__index = SlidingWindow

function SlidingWindow.new(limit, window_seconds)
    return setmetatable({
        limit   = limit,
        window  = window_seconds,
        requests= {},
    }, SlidingWindow)
end

function SlidingWindow:allow()
    local now = socket.gettime()
    local cutoff = now - self.window
    -- Remove old entries
    local new_reqs = {}
    for _, t in ipairs(self.requests) do
        if t > cutoff then table.insert(new_reqs, t) end
    end
    self.requests = new_reqs

    if #self.requests < self.limit then
        table.insert(self.requests, now)
        return true, 0
    end
    -- Wait until oldest request expires
    local wait = self.requests[1] + self.window - now
    return false, math.max(0, wait)
end

-- Rate-limited HTTP client
local RateLimitedClient = {}
RateLimitedClient.__index = RateLimitedClient

function RateLimitedClient.new(base_url, rate_per_second, burst)
    return setmetatable({
        base_url = base_url:gsub("/$", ""),
        bucket   = TokenBucket.new(rate_per_second or 5, burst or 10),
        window   = SlidingWindow.new(burst or 10, 1),
        stats    = { sent=0, queued=0, dropped=0, total_wait=0 },
    }, RateLimitedClient)
end

function RateLimitedClient:request(method, path, opts)
    opts = opts or {}
    local url = self.base_url .. path
    local chunks = {}

    -- Rate limit
    local t0 = socket.gettime()
    self.bucket:wait_and_consume(1)
    self.stats.total_wait = self.stats.total_wait + (socket.gettime() - t0)
    self.stats.sent = self.stats.sent + 1

    local body = opts.body
    local hdrs = opts.headers or {}
    if body then hdrs["Content-Length"] = #body end

    local _, code = http.request {
        url     = url,
        method  = method,
        headers = hdrs,
        source  = body and ltn12.source.string(body) or nil,
        sink    = ltn12.sink.table(chunks),
    }
    return table.concat(chunks), code
end

function RateLimitedClient:get(path, opts)  return self:request("GET", path, opts) end
function RateLimitedClient:post(path, opts) return self:request("POST", path, opts) end

-- Demo
print("=== HTTP API Rate Limiter ===")

-- TokenBucket demo
print("Token Bucket (5 req/s, capacity=3):")
local bucket = TokenBucket.new(5, 3)
for i = 1, 6 do
    local ok, wait = bucket:consume(1)
    print(string.format("  Request %d: %s (wait=%.3fs, tokens=%.2f)",
        i, ok and "ALLOWED" or "LIMITED", wait, bucket.tokens))
    if not ok then socket.sleep(wait) end
end

-- Sliding Window demo
print("\nSliding Window (5 req per 2s):")
local sw = SlidingWindow.new(5, 2)
for i = 1, 7 do
    local ok2, wait2 = sw:allow()
    print(string.format("  Request %d: %s (queue=%d, wait=%.3fs)",
        i, ok2 and "ALLOWED" or "LIMITED", #sw.requests, wait2))
end

-- Rate-limited client
print("\nRate-limited client (3 req/s):")
local client = RateLimitedClient.new("http://example.com", 3, 3)
print(string.format("  Base URL: %s", client.base_url))
print(string.format("  Rate: %d req/s, Burst: %d",
    client.bucket.rate, client.bucket.capacity))

-- Simulate 5 requests
for i = 1, 5 do
    local t0 = socket.gettime()
    client.bucket:wait_and_consume(1)
    client.stats.sent = client.stats.sent + 1
    local elapsed = socket.gettime() - t0
    print(string.format("  [req %d] waited=%.3fs tokens=%.2f",
        i, elapsed, client.bucket.tokens))
end

print(string.format("\nStats: sent=%d total_wait=%.3fs",
    client.stats.sent, client.stats.total_wait))
```

---

## ตัวอย่างที่ 35: HTTP Response Validation

Validate HTTP responses ด้วย schemas และ assertions

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- Response assertion library
local Assert = {}

function Assert.status(resp, expected)
    if resp.code ~= expected then
        error(string.format("Expected HTTP %d, got %d", expected, resp.code), 2)
    end
    return resp
end

function Assert.status_range(resp, min, max)
    if not resp.code or resp.code < min or resp.code > max then
        error(string.format("Expected HTTP %d-%d, got %s", min, max, tostring(resp.code)), 2)
    end
    return resp
end

function Assert.header(resp, name, expected_value)
    local val = resp.headers and resp.headers[name:lower()]
    if not val then
        error(string.format("Missing header: %s", name), 2)
    end
    if expected_value and not val:lower():find(expected_value:lower(), 1, true) then
        error(string.format("Header %s: expected %q, got %q", name, expected_value, val), 2)
    end
    return resp
end

function Assert.content_type(resp, expected)
    return Assert.header(resp, "content-type", expected)
end

function Assert.body_contains(resp, pattern)
    if not resp.body:find(pattern) then
        error(string.format("Body does not contain: %q", pattern), 2)
    end
    return resp
end

function Assert.body_not_empty(resp)
    if #resp.body == 0 then error("Response body is empty", 2) end
    return resp
end

function Assert.json_field(resp, field, expected)
    local val = resp.body:match('"' .. field .. '"%s*:%s*"([^"]*)"')
        or resp.body:match('"' .. field .. '"%s*:%s*(%d+)')
        or resp.body:match('"' .. field .. '"%s*:%s*(true)')
        or resp.body:match('"' .. field .. '"%s*:%s*(false)')
    if val == nil then
        error(string.format("JSON field %q not found", field), 2)
    end
    if expected ~= nil and tostring(val) ~= tostring(expected) then
        error(string.format("JSON field %q: expected %q, got %q",
            field, tostring(expected), tostring(val)), 2)
    end
    return resp
end

function Assert.response_time(resp, max_ms)
    if resp.elapsed_ms and resp.elapsed_ms > max_ms then
        error(string.format("Response too slow: %.1fms > %dms",
            resp.elapsed_ms, max_ms), 2)
    end
    return resp
end

-- HTTP client that returns structured response
local function timed_get(url, headers)
    local t0 = require("socket").gettime()
    local chunks = {}
    local _, code, hdrs = http.request {
        url     = url,
        method  = "GET",
        headers = headers or {},
        sink    = ltn12.sink.table(chunks),
    }
    local elapsed = (require("socket").gettime() - t0) * 1000
    return {
        code        = code,
        headers     = hdrs or {},
        body        = table.concat(chunks),
        elapsed_ms  = elapsed,
    }
end

-- Test runner
local TestRunner = {}
TestRunner.__index = TestRunner

function TestRunner.new(name)
    return setmetatable({
        name   = name,
        passed = 0,
        failed = 0,
        tests  = {}
    }, TestRunner)
end

function TestRunner:test(name, fn)
    local ok, err = pcall(fn)
    if ok then
        self.passed = self.passed + 1
        print(string.format("  [PASS] %s", name))
    else
        self.failed = self.failed + 1
        print(string.format("  [FAIL] %s: %s", name, tostring(err):sub(1,60)))
    end
end

function TestRunner:summary()
    local total = self.passed + self.failed
    print(string.format("\n[%s] %d/%d passed", self.name, self.passed, total))
    return self.failed == 0
end

-- Demo
print("=== HTTP Response Validation ===")

local runner = TestRunner.new("HTTP API Tests")

-- Test real endpoint
runner:test("example.com returns 200", function()
    local resp = timed_get("http://example.com/")
    Assert.status(resp, 200)
    Assert.body_not_empty(resp)
end)

runner:test("example.com has HTML content", function()
    local resp = timed_get("http://example.com/")
    Assert.content_type(resp, "text/html")
    Assert.body_contains(resp, "Example Domain")
end)

runner:test("404 page returns 404", function()
    local resp = timed_get("http://example.com/this-page-does-not-exist-xyz")
    Assert.status_range(resp, 400, 499)
end)

runner:test("response time under 5s", function()
    local resp = timed_get("http://example.com/")
    Assert.response_time(resp, 5000)
end)

-- Simulate JSON API test
runner:test("JSON field validation (simulated)", function()
    local fake_resp = {
        code    = 200,
        headers = { ["content-type"] = "application/json" },
        body    = '{"id":42,"name":"Alice","active":true,"role":"admin"}',
    }
    Assert.status(fake_resp, 200)
    Assert.json_field(fake_resp, "name", "Alice")
    Assert.json_field(fake_resp, "id",   "42")
    Assert.body_contains(fake_resp, "admin")
end)

runner:test("missing header detection", function()
    local fake_resp = {
        code    = 200,
        headers = {},
        body    = "OK",
    }
    local ok, err = pcall(Assert.header, fake_resp, "Content-Type")
    if ok then error("Should have failed for missing header") end
end)

runner:summary()
```

---

## ตัวอย่างที่ 36: HTTP Retry with Backoff

Retry logic พร้อม exponential backoff สำหรับ resilient HTTP calls

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- Retry configuration
local RetryConfig = {
    DEFAULT = {
        max_attempts  = 3,
        initial_delay = 0.5,   -- seconds
        max_delay     = 30,
        multiplier    = 2.0,
        jitter        = true,
        retryable_codes = { 429, 500, 502, 503, 504 },
        retryable_errors = { "timeout", "refused", "closed" },
    }
}

local function should_retry(code, err, config)
    if err then
        for _, e in ipairs(config.retryable_errors) do
            if tostring(err):lower():find(e) then return true end
        end
    end
    if code then
        for _, c in ipairs(config.retryable_codes) do
            if code == c then return true end
        end
    end
    return false
end

local function calc_delay(attempt, config)
    local delay = config.initial_delay * (config.multiplier ^ (attempt - 1))
    delay = math.min(delay, config.max_delay)
    if config.jitter then
        -- Add ±25% jitter
        delay = delay * (0.75 + math.random() * 0.5)
    end
    return delay
end

-- Retry wrapper
local function retry_request(req_fn, config)
    config = config or RetryConfig.DEFAULT
    local last_code, last_err, last_body
    local attempts = {}

    for attempt = 1, config.max_attempts do
        local t0 = socket.gettime()
        local ok, code_or_err, body, hdrs = pcall(req_fn)
        local elapsed = socket.gettime() - t0

        if not ok then
            -- pcall error
            last_err = tostring(code_or_err)
            table.insert(attempts, {
                attempt = attempt,
                error   = last_err,
                elapsed = elapsed,
            })
        else
            last_code, last_body = code_or_err, body
            table.insert(attempts, {
                attempt  = attempt,
                code     = last_code,
                elapsed  = elapsed,
            })
        end

        -- Success?
        if last_code and last_code >= 200 and last_code < 300 then
            return last_body, last_code, attempts
        end

        -- Should we retry?
        if attempt < config.max_attempts and
           should_retry(last_code, last_err, config) then
            local delay = calc_delay(attempt, config)
            print(string.format("  [Retry %d/%d] HTTP %s, waiting %.2fs...",
                attempt, config.max_attempts,
                tostring(last_code or last_err), delay))
            socket.sleep(delay)
        else
            break
        end
    end

    return last_body, last_code, attempts, last_err
end

-- HTTP GET with retry
local function get_with_retry(url, config)
    return retry_request(function()
        local chunks = {}
        local _, code, hdrs = http.request {
            url    = url,
            method = "GET",
            sink   = ltn12.sink.table(chunks),
        }
        return code, table.concat(chunks), hdrs
    end, config)
end

-- Demo
print("=== HTTP Retry with Exponential Backoff ===")

-- Test with httpbin.org /status endpoints
local test_cases = {
    { url = "http://httpbin.org/status/200", desc = "200 OK (no retry)" },
    { url = "http://httpbin.org/status/503", desc = "503 (should retry)" },
    { url = "http://httpbin.org/status/429", desc = "429 (rate limited)" },
}

local cfg = {
    max_attempts    = 2,
    initial_delay   = 0.2,
    max_delay       = 2,
    multiplier      = 2.0,
    jitter          = false,
    retryable_codes  = { 429, 500, 502, 503, 504 },
    retryable_errors = { "timeout", "refused", "closed" },
}

for _, tc in ipairs(test_cases) do
    print(string.format("\nTest: %s", tc.desc))
    local t0 = socket.gettime()
    local body, code, attempts, err = get_with_retry(tc.url, cfg)
    local total = socket.gettime() - t0

    print(string.format("  Final: HTTP %s in %.2fs (%d attempts)",
        tostring(code or err), total, #attempts))
    for _, a in ipairs(attempts) do
        print(string.format("    attempt=%d code=%s elapsed=%.3fs",
            a.attempt, tostring(a.code or a.error), a.elapsed))
    end
end

-- Circuit breaker pattern
print("\n\nCircuit Breaker:")
local CircuitBreaker = {}
CircuitBreaker.__index = CircuitBreaker

function CircuitBreaker.new(threshold, timeout)
    return setmetatable({
        threshold  = threshold or 5,
        timeout    = timeout   or 30,
        failures   = 0,
        state      = "closed",  -- closed, open, half-open
        opened_at  = nil,
    }, CircuitBreaker)
end

function CircuitBreaker:allow()
    if self.state == "closed" then return true end
    if self.state == "open" then
        if os.time() - self.opened_at > self.timeout then
            self.state = "half-open"
            return true
        end
        return false
    end
    return true  -- half-open: allow one test request
end

function CircuitBreaker:success()
    self.failures = 0
    self.state = "closed"
end

function CircuitBreaker:failure()
    self.failures = self.failures + 1
    if self.failures >= self.threshold then
        self.state = "open"
        self.opened_at = os.time()
        print(string.format("  [CB] OPEN after %d failures", self.failures))
    end
end

local cb = CircuitBreaker.new(3, 10)
local outcomes = { true, false, false, false, true, true }
for i, success in ipairs(outcomes) do
    if cb:allow() then
        if success then cb:success()
        else cb:failure() end
        print(string.format("  Request %d: %s, state=%s, failures=%d",
            i, success and "OK" or "FAIL", cb.state, cb.failures))
    else
        print(string.format("  Request %d: BLOCKED (circuit open)", i))
    end
end
```

---

## ตัวอย่างที่ 37: WebSocket Client (Handshake)

WebSocket upgrade handshake และ frame encoding/decoding

```lua
local socket = require("socket")
local mime   = require("mime")  -- for base64

-- WebSocket frame opcodes
local WS_OP = {
    CONTINUATION = 0x0,
    TEXT         = 0x1,
    BINARY       = 0x2,
    CLOSE        = 0x8,
    PING         = 0x9,
    PONG         = 0xA,
}

-- Generate WebSocket accept key
local function sha1_fake(s)
    -- NOT real SHA1: placeholder for demo
    -- Real implementation requires luacrypto or a C extension
    local h = 0x67452301
    for i = 1, #s do
        h = ((h * 31) + s:byte(i)) & 0xFFFFFFFF
    end
    return string.pack(">I4I4I4I4I4", h, ~h&0xFFFFFFFF,
        (h*7)&0xFFFFFFFF, (h*13)&0xFFFFFFFF, (h*17)&0xFFFFFFFF)
end

local WS_GUID = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
local function ws_accept_key(client_key)
    local combined = client_key .. WS_GUID
    local hash = sha1_fake(combined)  -- In production: real SHA1
    return (mime.b64(hash))
end

-- Encode WebSocket frame (client-side, masked)
local function ws_encode(opcode, payload, fin)
    fin = (fin == nil) and true or fin
    local byte1 = (fin and 0x80 or 0x00) | (opcode & 0x0F)
    local mask_key = string.pack(">I4", math.random(0, 2^32-1))
    local payload_len = #payload

    local byte2_and_len
    if payload_len <= 125 then
        byte2_and_len = string.pack("BB", 0x80 | payload_len, mask_key:byte(1))
        -- Actually: byte2 = 0x80|len, then 4-byte mask
        byte2_and_len = string.pack("B", 0x80 | payload_len) .. mask_key
    elseif payload_len <= 65535 then
        byte2_and_len = string.pack("B>I2", 0x80 | 126, payload_len) .. mask_key
    else
        byte2_and_len = string.pack("B>I8", 0x80 | 127, payload_len) .. mask_key
    end

    -- Mask payload
    local masked = {}
    for i = 1, #payload do
        local mask_byte = mask_key:byte(((i-1) % 4) + 1)
        masked[i] = string.char(payload:byte(i) ~ mask_byte)
    end

    return string.char(byte1) .. byte2_and_len .. table.concat(masked)
end

-- Decode WebSocket frame (server-side, unmasked)
local function ws_decode(data)
    if #data < 2 then return nil, "incomplete" end
    local byte1 = data:byte(1)
    local byte2 = data:byte(2)
    local fin    = (byte1 & 0x80) ~= 0
    local opcode = byte1 & 0x0F
    local masked = (byte2 & 0x80) ~= 0
    local plen   = byte2 & 0x7F

    local offset = 3
    if plen == 126 then
        if #data < 4 then return nil, "incomplete" end
        plen = string.unpack(">I2", data, 3)
        offset = 5
    elseif plen == 127 then
        if #data < 10 then return nil, "incomplete" end
        plen = string.unpack(">I8", data, 3)
        offset = 11
    end

    local mask_key = ""
    if masked then
        if #data < offset + 3 then return nil, "incomplete" end
        mask_key = data:sub(offset, offset + 3)
        offset = offset + 4
    end

    if #data < offset + plen - 1 then return nil, "incomplete" end
    local raw_payload = data:sub(offset, offset + plen - 1)

    -- Unmask if needed
    local payload
    if masked then
        local bytes = {}
        for i = 1, #raw_payload do
            bytes[i] = string.char(raw_payload:byte(i) ~ mask_key:byte(((i-1)%4)+1))
        end
        payload = table.concat(bytes)
    else
        payload = raw_payload
    end

    return {
        fin     = fin,
        opcode  = opcode,
        masked  = masked,
        payload = payload,
        total   = offset + plen - 1,
    }
end

-- WebSocket client
local WSClient = {}
WSClient.__index = WSClient

function WSClient.new(host, port, path)
    return setmetatable({
        host = host, port = port or 80, path = path or "/",
        sock = nil, handlers = {},
    }, WSClient)
end

function WSClient:on(event, fn) self.handlers[event] = fn end

function WSClient:connect()
    local sock = socket.tcp()
    sock:settimeout(10)
    local ok, err = sock:connect(self.host, self.port)
    if not ok then return nil, err end

    -- Generate key
    local key_bytes = {}
    for _ = 1, 16 do table.insert(key_bytes, string.char(math.random(0,255))) end
    local client_key = mime.b64(table.concat(key_bytes))

    -- Upgrade request
    local req = table.concat({
        string.format("GET %s HTTP/1.1", self.path),
        "Host: " .. self.host,
        "Upgrade: websocket",
        "Connection: Upgrade",
        "Sec-WebSocket-Key: " .. client_key,
        "Sec-WebSocket-Version: 13",
        "", ""
    }, "\r\n")
    sock:send(req)

    -- Read response
    local status = sock:receive("*l")
    if not status:match("101") then
        sock:close()
        return nil, "upgrade failed: " .. status
    end
    while true do
        local line = sock:receive("*l")
        if not line or line == "" then break end
    end

    self.sock = sock
    return true
end

function WSClient:send_text(text)
    if not self.sock then return nil, "not connected" end
    local frame = ws_encode(WS_OP.TEXT, text)
    return self.sock:send(frame)
end

function WSClient:send_ping()
    if not self.sock then return nil, "not connected" end
    return self.sock:send(ws_encode(WS_OP.PING, "ping"))
end

function WSClient:close()
    if self.sock then
        pcall(function()
            self.sock:send(ws_encode(WS_OP.CLOSE, ""))
        end)
        self.sock:close()
        self.sock = nil
    end
end

-- Demo
print("=== WebSocket Client (Handshake) ===")

-- Show frame encoding
print("Frame encoding examples:")
local frames = {
    { WS_OP.TEXT,   "Hello, WebSocket!", "TEXT" },
    { WS_OP.BINARY, "\x00\x01\x02\x03",  "BINARY" },
    { WS_OP.PING,   "ping",               "PING" },
    { WS_OP.CLOSE,  "",                   "CLOSE" },
}

for _, f in ipairs(frames) do
    local frame = ws_encode(f[1], f[2])
    print(string.format("  [%s] payload=%d bytes, frame=%d bytes",
        f[3], #f[2], #frame))
end

-- Decode server frame (unmasked)
print("\nFrame decoding test (unmasked server frame):")
local server_frame = "\x81\x0DHello, Client!"  -- TEXT, 13 bytes
local decoded = ws_decode(server_frame)
if decoded then
    print(string.format("  fin=%s opcode=%d payload=%q",
        tostring(decoded.fin), decoded.opcode, decoded.payload))
end

-- Handshake request format
print("\nHandshake request format:")
print("  GET /ws HTTP/1.1")
print("  Host: api.example.com")
print("  Upgrade: websocket")
print("  Connection: Upgrade")
print("  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==")
print("  Sec-WebSocket-Version: 13")
print("\nExpected 101 Switching Protocols response")
```

---

## ตัวอย่างที่ 38: HTTP JSON API CRUD

Complete CRUD operations ผ่าน JSON REST API

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- Minimal JSON library
local json = {}

function json.encode(val)
    local t = type(val)
    if t == "nil"     then return "null"
    elseif t == "boolean" then return val and "true" or "false"
    elseif t == "number"  then
        if val ~= val then return "null" end  -- NaN
        return string.format(val == math.floor(val) and "%d" or "%.10g", val)
    elseif t == "string"  then
        return '"' .. val:gsub('\\','\\\\'):gsub('"','\\"')
                        :gsub('\n','\\n'):gsub('\r','\\r')
                        :gsub('\t','\\t') .. '"'
    elseif t == "table" then
        if #val > 0 or next(val) == nil then
            local a = {}
            for _, v in ipairs(val) do a[#a+1] = json.encode(v) end
            return "[" .. table.concat(a, ",") .. "]"
        else
            local o = {}
            for k, v in pairs(val) do
                o[#o+1] = json.encode(tostring(k)) .. ":" .. json.encode(v)
            end
            return "{" .. table.concat(o, ",") .. "}"
        end
    end
    return "null"
end

-- REST client
local RESTClient = {}
RESTClient.__index = RESTClient

function RESTClient.new(base_url, token)
    return setmetatable({
        base   = base_url:gsub("/$",""),
        token  = token,
        timeout= 10,
    }, RESTClient)
end

function RESTClient:_req(method, path, body_data)
    local url  = self.base .. path
    local body = body_data and json.encode(body_data) or nil
    local hdrs = {
        ["Accept"]       = "application/json",
        ["Content-Type"] = "application/json",
    }
    if self.token then hdrs["Authorization"] = "Bearer " .. self.token end
    if body then hdrs["Content-Length"] = #body end

    local chunks = {}
    local _, code, resp_hdrs = http.request {
        url     = url,
        method  = method,
        headers = hdrs,
        source  = body and ltn12.source.string(body) or nil,
        sink    = ltn12.sink.table(chunks),
    }
    local resp_body = table.concat(chunks)

    -- Parse simple JSON values
    local data = {}
    if resp_hdrs and (resp_hdrs["content-type"] or ""):find("json") then
        for k, v in resp_body:gmatch('"([^"]+)"%s*:%s*"([^"]*)"') do data[k] = v end
        for k, v in resp_body:gmatch('"([^"]+)"%s*:%s*(%d+)') do data[k] = tonumber(v) end
    end

    return {
        ok     = code and code >= 200 and code < 300,
        code   = code,
        body   = resp_body,
        data   = data,
    }
end

function RESTClient:list(resource)
    return self:_req("GET", "/" .. resource)
end

function RESTClient:get(resource, id)
    return self:_req("GET", string.format("/%s/%s", resource, id))
end

function RESTClient:create(resource, data)
    return self:_req("POST", "/" .. resource, data)
end

function RESTClient:update(resource, id, data)
    return self:_req("PUT", string.format("/%s/%s", resource, id), data)
end

function RESTClient:patch(resource, id, data)
    return self:_req("PATCH", string.format("/%s/%s", resource, id), data)
end

function RESTClient:delete(resource, id)
    return self:_req("DELETE", string.format("/%s/%s", resource, id))
end

-- Demo
print("=== HTTP JSON API CRUD ===")
local api = RESTClient.new("http://jsonplaceholder.typicode.com", nil)

print("\nCRUD operations on /posts:")

-- List (GET /posts)
print("\n1. GET /posts (list):")
local resp = api:list("posts")
print(string.format("   HTTP %d, body=%d bytes, ok=%s",
    resp.code or 0, #resp.body, tostring(resp.ok)))

-- Get one (GET /posts/1)
print("\n2. GET /posts/1:")
resp = api:get("posts", 1)
print(string.format("   HTTP %d, ok=%s", resp.code or 0, tostring(resp.ok)))
if resp.data.title then print("   title: " .. resp.data.title) end

-- Create (POST /posts)
print("\n3. POST /posts (create):")
resp = api:create("posts", {
    title  = "Lua HTTP Client Tutorial",
    body   = "Learning HTTP with LuaSocket",
    userId = 1
})
print(string.format("   HTTP %d, ok=%s", resp.code or 0, tostring(resp.ok)))
if resp.data.id then print("   new id: " .. tostring(resp.data.id)) end

-- Update (PUT /posts/1)
print("\n4. PUT /posts/1 (full update):")
resp = api:update("posts", 1, {
    id     = 1,
    title  = "Updated Title",
    body   = "Updated content",
    userId = 1
})
print(string.format("   HTTP %d, ok=%s", resp.code or 0, tostring(resp.ok)))

-- Patch (PATCH /posts/1)
print("\n5. PATCH /posts/1 (partial update):")
resp = api:patch("posts", 1, { title = "Partially Updated" })
print(string.format("   HTTP %d, ok=%s", resp.code or 0, tostring(resp.ok)))

-- Delete (DELETE /posts/1)
print("\n6. DELETE /posts/1:")
resp = api:delete("posts", 1)
print(string.format("   HTTP %d, ok=%s", resp.code or 0, tostring(resp.ok)))

print("\nAll CRUD operations completed!")
```

---

## ตัวอย่างที่ 39: HTTP Security Headers Checker

ตรวจสอบ security headers ของ HTTP responses

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- Security header definitions
local SECURITY_HEADERS = {
    {
        name        = "Strict-Transport-Security",
        key         = "strict-transport-security",
        required    = true,
        description = "Force HTTPS (HSTS)",
        check = function(val)
            return val:find("max%-age=") ~= nil,
                not val:find("max%-age=%s*0")
                    and "max-age should be > 0" or "max-age=0 disables HSTS"
        end
    },
    {
        name        = "X-Content-Type-Options",
        key         = "x-content-type-options",
        required    = true,
        description = "Prevent MIME sniffing",
        check = function(val)
            return val:lower() == "nosniff", "should be 'nosniff'"
        end
    },
    {
        name        = "X-Frame-Options",
        key         = "x-frame-options",
        required    = true,
        description = "Prevent clickjacking",
        check = function(val)
            local v = val:upper()
            return v == "DENY" or v == "SAMEORIGIN" or v:find("ALLOW%-FROM"),
                "should be DENY, SAMEORIGIN, or ALLOW-FROM"
        end
    },
    {
        name        = "Content-Security-Policy",
        key         = "content-security-policy",
        required    = false,  -- recommended
        description = "Content Security Policy",
        check = function(val)
            local has_default = val:find("default%-src") ~= nil
            return has_default, "should include default-src directive"
        end
    },
    {
        name        = "X-XSS-Protection",
        key         = "x-xss-protection",
        required    = false,
        description = "Legacy XSS protection",
        check = function(val)
            return true, "present"  -- Any value is ok
        end
    },
    {
        name        = "Referrer-Policy",
        key         = "referrer-policy",
        required    = false,
        description = "Referrer information control",
        check = function(val)
            local valid = {
                ["no-referrer"]=true, ["same-origin"]=true,
                ["strict-origin"]=true, ["strict-origin-when-cross-origin"]=true,
                ["no-referrer-when-downgrade"]=true, ["origin"]=true,
                ["origin-when-cross-origin"]=true, ["unsafe-url"]=true,
            }
            return valid[val:lower()] ~= nil, "unrecognized policy: " .. val
        end
    },
    {
        name        = "Permissions-Policy",
        key         = "permissions-policy",
        required    = false,
        description = "Feature/permissions policy",
        check = function(val)
            return true, "present (" .. #val .. " bytes)"
        end
    },
}

local function check_security_headers(url)
    local chunks = {}
    local _, code, headers = http.request {
        url     = url,
        method  = "HEAD",
        headers = { ["User-Agent"] = "SecurityChecker/1.0" },
        sink    = ltn12.sink.table(chunks),
    }

    if not code then
        return nil, "request failed"
    end

    headers = headers or {}
    local results = {
        url     = url,
        code    = code,
        score   = 0,
        max     = 0,
        headers = {},
    }

    for _, hdef in ipairs(SECURITY_HEADERS) do
        local val = headers[hdef.key]
        local found = val ~= nil
        local valid, msg = false, ""

        if found then
            valid, msg = hdef.check(val)
        end

        local points = (found and valid) and 1 or 0
        results.score = results.score + points
        results.max   = results.max + 1

        table.insert(results.headers, {
            name        = hdef.name,
            required    = hdef.required,
            found       = found,
            valid       = valid,
            value       = val,
            message     = msg,
            description = hdef.description,
        })
    end

    return results
end

local function grade(score, max)
    local pct = score / max * 100
    if pct >= 90 then return "A+"
    elseif pct >= 80 then return "A"
    elseif pct >= 70 then return "B"
    elseif pct >= 60 then return "C"
    elseif pct >= 40 then return "D"
    else return "F" end
end

-- Demo
print("=== HTTP Security Headers Checker ===")
local urls = {
    "http://example.com/",
    "http://httpbin.org/",
}

for _, url in ipairs(urls) do
    print(string.format("\nChecking: %s", url))
    local results, err = check_security_headers(url)
    if not results then
        print("  Error: " .. tostring(err))
    else
        print(string.format("  HTTP %d | Score: %d/%d | Grade: %s",
            results.code, results.score, results.max,
            grade(results.score, results.max)))
        print(string.format("  %-38s %-8s %-8s %s",
            "Header", "Found", "Valid", "Notes"))
        print("  " .. string.rep("-", 75))
        for _, h in ipairs(results.headers) do
            local req_mark = h.required and "*" or " "
            print(string.format("  %s%-37s %-8s %-8s %s",
                req_mark,
                h.name:sub(1,36),
                h.found and "YES" or "NO",
                (h.found and h.valid) and "OK" or (h.found and "WARN" or "-"),
                (h.message or ""):sub(1,30)))
        end
        print("  (* = required)")
    end
end
```

---

## ตัวอย่างที่ 40: HTTP Load Testing Tool

Load testing tool วัด throughput และ latency

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

local LoadTester = {}
LoadTester.__index = LoadTester

function LoadTester.new(url, opts)
    opts = opts or {}
    return setmetatable({
        url           = url,
        concurrency   = opts.concurrency   or 1,
        total_requests= opts.requests      or 10,
        timeout       = opts.timeout       or 10,
        method        = opts.method        or "GET",
        body          = opts.body,
        headers       = opts.headers       or {},
        results       = {},
    }, LoadTester)
end

function LoadTester:_single_request()
    local t0 = socket.gettime()
    local chunks = {}
    local hdrs = {}
    for k, v in pairs(self.headers) do hdrs[k] = v end
    if self.body then hdrs["Content-Length"] = #self.body end

    local ok, code_or_err, resp_hdrs = pcall(function()
        return http.request {
            url     = self.url,
            method  = self.method,
            headers = hdrs,
            source  = self.body and ltn12.source.string(self.body) or nil,
            sink    = ltn12.sink.table(chunks),
        }
    end)

    local elapsed_ms = (socket.gettime() - t0) * 1000
    if not ok then
        return { ok=false, error=tostring(code_or_err), elapsed_ms=elapsed_ms }
    end
    return {
        ok         = code_or_err and code_or_err >= 200 and code_or_err < 400,
        code       = code_or_err,
        elapsed_ms = elapsed_ms,
        body_size  = #table.concat(chunks),
    }
end

function LoadTester:run()
    self.results = {}
    local start_time = socket.gettime()

    for i = 1, self.total_requests do
        local r = self:_single_request()
        r.seq = i
        table.insert(self.results, r)
    end

    local total_time = socket.gettime() - start_time
    return self:_analyze(total_time)
end

function LoadTester:_analyze(total_time)
    local stats = {
        total       = #self.results,
        success     = 0,
        failed      = 0,
        total_time  = total_time,
        latencies   = {},
        codes       = {},
        errors      = {},
    }

    for _, r in ipairs(self.results) do
        if r.ok then
            stats.success = stats.success + 1
        else
            stats.failed = stats.failed + 1
            local e = r.error or ("HTTP " .. tostring(r.code))
            stats.errors[e] = (stats.errors[e] or 0) + 1
        end
        if r.elapsed_ms then
            table.insert(stats.latencies, r.elapsed_ms)
        end
        if r.code then
            local c = tostring(r.code)
            stats.codes[c] = (stats.codes[c] or 0) + 1
        end
    end

    -- Calculate percentiles
    table.sort(stats.latencies)
    local n = #stats.latencies
    if n > 0 then
        stats.p50  = stats.latencies[math.floor(n * 0.50)] or 0
        stats.p90  = stats.latencies[math.floor(n * 0.90)] or 0
        stats.p99  = stats.latencies[math.floor(n * 0.99) + 1] or stats.latencies[n]
        stats.min  = stats.latencies[1]
        stats.max  = stats.latencies[n]
        local sum  = 0
        for _, v in ipairs(stats.latencies) do sum = sum + v end
        stats.avg  = sum / n
    end
    stats.rps = stats.total / total_time

    return stats
end

-- Demo
print("=== HTTP Load Testing Tool ===")
local tester = LoadTester.new("http://example.com/", {
    requests    = 5,
    timeout     = 10,
    method      = "GET",
})

print(string.format("Load test: %s", tester.url))
print(string.format("Requests: %d, Timeout: %ds", tester.total_requests, tester.timeout))
print("Running...")

local t0 = socket.gettime()
local stats = tester:run()
local elapsed = socket.gettime() - t0

print(string.format("\nResults (%d requests in %.2fs):", stats.total, elapsed))
print(string.format("  Success:  %d (%.0f%%)", stats.success, stats.success/stats.total*100))
print(string.format("  Failed:   %d", stats.failed))
print(string.format("  RPS:      %.1f req/s", stats.rps))
print(string.format("\nLatency (ms):"))
print(string.format("  Min:  %.1f  Avg: %.1f  Max: %.1f",
    stats.min or 0, stats.avg or 0, stats.max or 0))
print(string.format("  P50:  %.1f  P90: %.1f  P99: %.1f",
    stats.p50 or 0, stats.p90 or 0, stats.p99 or 0))

if next(stats.codes) then
    print("\nHTTP Status codes:")
    local code_list = {}
    for c, n in pairs(stats.codes) do table.insert(code_list, {c, n}) end
    table.sort(code_list, function(a,b) return a[1] < b[1] end)
    for _, cv in ipairs(code_list) do
        print(string.format("  HTTP %s: %d", cv[1], cv[2]))
    end
end

if next(stats.errors) then
    print("\nErrors:")
    for e, n in pairs(stats.errors) do
        print(string.format("  %s: %d", e:sub(1,50), n))
    end
end

-- Histogram of latencies
if #tester.results > 0 then
    print("\nLatency histogram:")
    local buckets = {}
    local bucket_size = math.max(1, math.floor((stats.max - stats.min) / 5))
    for _, r in ipairs(tester.results) do
        if r.elapsed_ms then
            local b = math.floor((r.elapsed_ms - stats.min) / bucket_size)
            buckets[b] = (buckets[b] or 0) + 1
        end
    end
    local max_count = 0
    for _, c in pairs(buckets) do max_count = math.max(max_count, c) end
    local b_keys = {}
    for k in pairs(buckets) do table.insert(b_keys, k) end
    table.sort(b_keys)
    for _, b in ipairs(b_keys) do
        local low  = stats.min + b * bucket_size
        local high = low + bucket_size
        local bar_len = math.floor(buckets[b] / max_count * 20)
        print(string.format("  %5.0f-%5.0fms |%s %d",
            low, high, string.rep("█", bar_len), buckets[b]))
    end
end
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
