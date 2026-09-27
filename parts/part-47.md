# บทที่ 47: Network Programming - LuaSocket

## บทนำ

LuaSocket เป็น library ที่ให้ความสามารถด้าน networking ให้กับ Lua โดยรองรับ TCP, UDP, HTTP, FTP, SMTP และ DNS ซึ่งเป็นพื้นฐานสำคัญสำหรับการพัฒนา application ที่ต้องติดต่อสื่อสารผ่าน network

## การติดตั้ง LuaSocket

```bash
# ด้วย LuaRocks
luarocks install luasocket

# Ubuntu/Debian
apt-get install lua-socket

# macOS
brew install luasocket
```

---

## ตัวอย่างที่ 1: ตรวจสอบ LuaSocket และ Version

```lua
-- ตรวจสอบว่า LuaSocket ใช้งานได้
local ok, socket = pcall(require, "socket")
if not ok then
    print("LuaSocket ไม่ได้ติดตั้ง:")
    print("  luarocks install luasocket")
    os.exit(1)
end

print("LuaSocket version:", socket._VERSION)
print("Available modules:")

-- ตรวจสอบ sub-modules
local modules = {"socket", "socket.http", "socket.ftp",
                  "socket.smtp", "socket.url", "mime", "ltn12"}
for _, mod in ipairs(modules) do
    local ok2, _ = pcall(require, mod)
    print(string.format("  %-20s %s", mod, ok2 and "OK" or "ไม่พบ"))
end

-- ข้อมูล network constants
print("\nNetwork constants:")
print("  INADDR_ANY:", socket.INADDR_ANY or "0.0.0.0")
```

---

## ตัวอย่างที่ 2: TCP Client พื้นฐาน

```lua
local socket = require("socket")

-- เชื่อมต่อ TCP แบบพื้นฐาน
local function tcp_connect(host, port, timeout)
    timeout = timeout or 5

    local client = socket.tcp()
    client:settimeout(timeout)

    local ok, err = client:connect(host, port)
    if not ok then
        client:close()
        return nil, "Connection failed: " .. tostring(err)
    end

    return client, nil
end

-- ทดสอบเชื่อมต่อไปยัง echo server
print("TCP Client Example")
print("==================")

-- ลองเชื่อมต่อไปยัง public echo server
local client, err = tcp_connect("tcpbin.com", 4242, 10)
if client then
    print("Connected to tcpbin.com:4242")

    -- ส่ง data
    local msg = "Hello from Lua!\n"
    local sent, err2 = client:send(msg)
    if sent then
        print("Sent:", sent, "bytes")

        -- รับ echo กลับมา
        local response, err3 = client:receive("*l")
        if response then
            print("Received:", response)
        else
            print("Receive error:", err3)
        end
    end
    client:close()
    print("Connection closed")
else
    print("Cannot connect (no internet?):", err)
    print("Showing code structure only")
end
```

---

## ตัวอย่างที่ 3: TCP Server พื้นฐาน

```lua
local socket = require("socket")

-- สร้าง Simple TCP Echo Server
local function create_echo_server(port)
    local server = socket.tcp()
    server:setoption("reuseaddr", true)

    local ok, err = server:bind("*", port)
    if not ok then
        return nil, "Bind failed: " .. tostring(err)
    end

    server:listen(5)
    print(string.format("Echo Server listening on port %d", port))
    return server
end

local function run_echo_server_once(server)
    print("Waiting for connection...")
    local client, err = server:accept()
    if not client then
        return false, err
    end

    local ip, port = client:getpeername()
    print(string.format("Client connected: %s:%d", ip, port))

    -- Echo loop
    local count = 0
    while count < 5 do  -- จำกัด 5 messages ต่อ client
        local data, err2 = client:receive("*l")
        if not data then
            if err2 ~= "closed" then
                print("Receive error:", err2)
            end
            break
        end

        print("Received:", data)
        local sent = client:send(data .. "\n")
        if not sent then break end
        count = count + 1

        if data:lower() == "quit" then break end
    end

    client:close()
    print(string.format("Client disconnected (handled %d messages)", count))
    return true
end

-- ตัวอย่างการสร้าง server (ไม่ run จริง เพราะจะ block)
print("TCP Server Example:")
print("  local server = create_echo_server(8080)")
print("  run_echo_server_once(server)")
print("  server:close()")

-- Demo ที่ทดสอบได้ - server/client ใน process เดียวกัน
local function demo_loopback()
    local server = socket.tcp()
    server:setoption("reuseaddr", true)
    server:bind("127.0.0.1", 19999)
    server:listen(1)
    server:settimeout(2)

    -- Fork-like: ใช้ coroutine แทน
    local co = coroutine.create(function()
        local client = socket.tcp()
        client:settimeout(2)
        socket.sleep(0.1)  -- รอให้ server พร้อม
        client:connect("127.0.0.1", 19999)
        client:send("Hello Server!\n")
        local response = client:receive("*l")
        print("Client received:", response)
        client:close()
    end)

    coroutine.resume(co)

    local conn = server:accept()
    if conn then
        local data = conn:receive("*l")
        print("Server received:", data)
        conn:send("Echo: " .. data .. "\n")
        conn:close()
    end

    coroutine.resume(co)
    server:close()
    print("Demo completed")
end

demo_loopback()
```

---

## ตัวอย่างที่ 4: UDP Client และ Server

```lua
local socket = require("socket")

-- UDP Socket พื้นฐาน
print("=== UDP Example ===")

-- UDP Server
local function create_udp_server(port)
    local server = socket.udp()
    server:setsockname("*", port)
    server:settimeout(2)
    print("UDP Server bound to port:", port)
    return server
end

-- UDP Client
local function udp_send(host, port, message)
    local client = socket.udp()
    client:settimeout(2)

    local ok, err = client:sendto(message, host, port)
    if not ok then
        client:close()
        return nil, err
    end

    -- รอรับ response
    local data, rip, rport = client:receivefrom()
    client:close()

    if data then
        return data, nil, rip, rport
    else
        return nil, rip  -- error message
    end
end

-- Demo UDP loopback
local server = socket.udp()
server:setsockname("127.0.0.1", 19998)
server:settimeout(1)

-- ส่งข้อความ
local client = socket.udp()
client:settimeout(1)
client:sendto("UDP Hello!", "127.0.0.1", 19998)

-- รับ
local data, ip, port = server:receivefrom()
if data then
    print(string.format("Server got from %s:%d: %s", ip, port, data))
    -- Echo กลับ
    server:sendto("Echo: " .. data, ip, port)
end

-- Client รับ echo
local response = client:receive()
if response then
    print("Client got:", response)
end

server:close()
client:close()
print("UDP demo done")
```

---

## ตัวอย่างที่ 5: Non-blocking Sockets

```lua
local socket = require("socket")

-- Non-blocking TCP ด้วย timeout = 0
local function non_blocking_connect(host, port)
    local s = socket.tcp()
    s:settimeout(0)  -- non-blocking

    local ok, err = s:connect(host, port)
    -- non-blocking connect จะ return "timeout" ก่อน
    if ok or err == "timeout" then
        return s
    else
        s:close()
        return nil, err
    end
end

-- Async-like pattern ด้วย coroutines
local function async_recv(sock, pattern)
    while true do
        local data, err = sock:receive(pattern)
        if data then
            return data
        elseif err == "timeout" then
            coroutine.yield()  -- ให้ coroutine อื่นทำงานก่อน
        else
            return nil, err
        end
    end
end

-- Connection pool
local ConnectionPool = {}
ConnectionPool.__index = ConnectionPool

function ConnectionPool.new(host, port, size)
    local pool = setmetatable({}, ConnectionPool)
    pool.host  = host
    pool.port  = port
    pool.size  = size
    pool.conns = {}
    pool.available = {}
    return pool
end

function ConnectionPool:get()
    if #self.available > 0 then
        return table.remove(self.available)
    end

    if #self.conns < self.size then
        local conn = socket.tcp()
        conn:settimeout(5)
        local ok, err = conn:connect(self.host, self.port)
        if ok then
            table.insert(self.conns, conn)
            return conn
        else
            conn:close()
            return nil, err
        end
    end

    return nil, "pool exhausted"
end

function ConnectionPool:release(conn)
    -- ตรวจสอบว่า connection ยังดีอยู่
    conn:settimeout(0)
    local _, err = conn:receive(0)
    conn:settimeout(5)

    if err ~= "closed" then
        table.insert(self.available, conn)
    else
        -- Remove from pool
        for i, c in ipairs(self.conns) do
            if c == conn then
                table.remove(self.conns, i)
                break
            end
        end
        conn:close()
    end
end

function ConnectionPool:close_all()
    for _, conn in ipairs(self.conns) do
        conn:close()
    end
    self.conns = {}
    self.available = {}
end

print("Non-blocking socket patterns defined")
print("Connection pool size:", 5)
```

---

## ตัวอย่างที่ 6: socket.select() สำหรับ Multiplexing

```lua
local socket = require("socket")

-- select() ช่วยให้จัดการหลาย connections พร้อมกัน
print("=== socket.select() Example ===")

-- Multi-server ที่รับหลาย connections
local function multi_server_demo()
    -- สร้าง 2 servers บน port ต่างกัน
    local servers = {}
    local ports = {19990, 19991}

    for _, port in ipairs(ports) do
        local s = socket.tcp()
        s:setoption("reuseaddr", true)
        s:bind("127.0.0.1", port)
        s:listen(5)
        s:settimeout(0)  -- non-blocking
        table.insert(servers, s)
        print(string.format("Server listening on port %d", port))
    end

    -- รายการ sockets ที่รอ read
    local read_set = {}
    for _, s in ipairs(servers) do
        table.insert(read_set, s)
    end

    local clients = {}
    local max_iterations = 10

    -- ส่ง clients เพื่อทดสอบ (ใน background)
    for i, port in ipairs(ports) do
        local c = socket.tcp()
        c:settimeout(1)
        if c:connect("127.0.0.1", port) then
            c:send("Hello server " .. i .. "!\n")
            table.insert(clients, c)
        end
    end

    -- Event loop
    for iter = 1, max_iterations do
        local readable, _, err = socket.select(read_set, nil, 0.5)

        if err == "timeout" then
            -- timeout ปกติ
        elseif readable then
            for _, s in ipairs(readable) do
                -- ตรวจว่าเป็น server หรือ client
                local is_server = false
                for _, srv in ipairs(servers) do
                    if s == srv then
                        is_server = true
                        break
                    end
                end

                if is_server then
                    -- Accept new connection
                    local conn, _ = s:accept()
                    if conn then
                        conn:settimeout(0)
                        table.insert(read_set, conn)
                        local ip, port2 = conn:getpeername()
                        print(string.format("  New client: %s:%d", ip, port2))
                    end
                else
                    -- Read from existing client
                    local data, err2 = s:receive("*l")
                    if data then
                        print("  Received:", data)
                        s:send("Echo: " .. data .. "\n")
                    else
                        -- Remove from set
                        for i, rs in ipairs(read_set) do
                            if rs == s then
                                table.remove(read_set, i)
                                break
                            end
                        end
                        s:close()
                    end
                end
            end
        end

        -- หยุดถ้าไม่มี clients แล้ว
        if #read_set <= #servers and iter > 3 then
            break
        end
    end

    -- Cleanup
    for _, c in ipairs(clients) do c:close() end
    for _, s in ipairs(servers) do s:close() end
    print("Multi-server demo complete")
end

multi_server_demo()
```

---

## ตัวอย่างที่ 7: HTTP Client พื้นฐาน

```lua
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- GET Request พื้นฐาน
local function http_get(url)
    local response_body = {}

    local result, status, headers, status_line = http.request{
        url    = url,
        method = "GET",
        sink   = ltn12.sink.table(response_body),
        headers = {
            ["User-Agent"] = "LuaSocket/3.0 Tutorial",
            ["Accept"]     = "text/html,application/json",
        },
    }

    if not result then
        return nil, status  -- status เป็น error message ในกรณีนี้
    end

    return {
        status  = status,
        headers = headers,
        body    = table.concat(response_body),
    }
end

-- POST Request
local function http_post(url, data, content_type)
    content_type = content_type or "application/x-www-form-urlencoded"
    local response_body = {}

    local body_str = type(data) == "string" and data or
        (function()
            local parts = {}
            for k, v in pairs(data) do
                table.insert(parts, k .. "=" .. tostring(v))
            end
            return table.concat(parts, "&")
        end)()

    local result, status, headers = http.request{
        url    = url,
        method = "POST",
        sink   = ltn12.sink.table(response_body),
        source = ltn12.source.string(body_str),
        headers = {
            ["Content-Type"]   = content_type,
            ["Content-Length"] = #body_str,
            ["User-Agent"]     = "LuaSocket/3.0",
        },
    }

    if not result then
        return nil, status
    end

    return {
        status  = status,
        headers = headers,
        body    = table.concat(response_body),
    }
end

-- ทดสอบ HTTP GET
print("=== HTTP Client ===")
local resp, err = http_get("http://httpbin.org/get")
if resp then
    print("Status:", resp.status)
    print("Content-Type:", resp.headers["content-type"] or "unknown")
    print("Body length:", #resp.body)
    -- แสดง JSON ส่วนแรก
    print("Body (first 200 chars):", resp.body:sub(1, 200))
else
    print("Error:", err)
    print("(ต้องการ internet connection)")
end
```

---

## ตัวอย่างที่ 8: HTTP Request with Headers

```lua
local http  = require("socket.http")
local ltn12 = require("ltn12")

-- HTTP Request พร้อม custom headers
local function http_request_with_headers(options)
    local response_body = {}
    local headers = options.headers or {}

    -- Default headers
    headers["User-Agent"] = headers["User-Agent"] or "Lua/5.4 LuaSocket"
    headers["Accept"]     = headers["Accept"] or "*/*"
    headers["Connection"] = "close"

    if options.body then
        headers["Content-Length"] = #options.body
    end

    local result, status, resp_headers = http.request{
        url    = options.url,
        method = options.method or "GET",
        headers = headers,
        sink   = ltn12.sink.table(response_body),
        source = options.body and
                 ltn12.source.string(options.body) or nil,
    }

    return result and {
        ok      = true,
        status  = status,
        headers = resp_headers,
        body    = table.concat(response_body),
    } or {
        ok  = false,
        err = status,
    }
end

-- ตัวอย่างการใช้งาน
print("HTTP with Headers Example:")

-- GET with authentication header
local r1 = http_request_with_headers{
    url = "http://httpbin.org/headers",
    headers = {
        ["Authorization"] = "Bearer mytoken123",
        ["X-Custom-Header"] = "LuaTutorial",
        ["Accept"] = "application/json",
    }
}

if r1.ok then
    print("GET /headers status:", r1.status)
    print("Response:", r1.body:sub(1, 300))
else
    print("Error:", r1.err)
end

-- POST with JSON body
local json_body = '{"name": "Lua Test", "version": "5.4"}'
local r2 = http_request_with_headers{
    url    = "http://httpbin.org/post",
    method = "POST",
    body   = json_body,
    headers = {
        ["Content-Type"] = "application/json",
    }
}

if r2.ok then
    print("\nPOST status:", r2.status)
    print("Response:", r2.body:sub(1, 300))
end
```

---

## ตัวอย่างที่ 9: DNS Lookup

```lua
local socket = require("socket")

-- DNS resolution
local function dns_lookup(hostname)
    local result = {}

    -- ตรวจสอบ IPv4
    local ip4, err = socket.dns.toip(hostname)
    if ip4 then
        result.ipv4 = ip4
    end

    -- ตรวจสอบ hostname
    local name, aliases, addresses = socket.dns.tohostname(
        ip4 or hostname
    )

    result.hostname = name
    result.aliases  = aliases or {}
    result.addresses = addresses or {}

    return result
end

-- Batch DNS lookup
local function batch_dns_lookup(hostnames)
    local results = {}
    for _, host in ipairs(hostnames) do
        local ok, info = pcall(dns_lookup, host)
        if ok then
            results[host] = info
        else
            results[host] = {error = tostring(info)}
        end
    end
    return results
end

-- ทดสอบ
print("=== DNS Lookup ===")

local hosts = {
    "google.com",
    "github.com",
    "localhost",
    "invalid.host.xyz",
}

for _, host in ipairs(hosts) do
    local ip, err = socket.dns.toip(host)
    if ip then
        print(string.format("%-25s -> %s", host, ip))
    else
        print(string.format("%-25s -> ERROR: %s", host, err or "failed"))
    end
end

-- Reverse lookup
print("\nReverse DNS:")
local test_ips = {"8.8.8.8", "1.1.1.1", "127.0.0.1"}
for _, ip in ipairs(test_ips) do
    local hostname, err = socket.dns.tohostname(ip)
    if hostname then
        print(string.format("%-15s -> %s", ip, hostname))
    else
        print(string.format("%-15s -> %s", ip, err or "no PTR record"))
    end
end
```

---

## ตัวอย่างที่ 10: Timeout Handling

```lua
local socket = require("socket")

-- การตั้ง timeout หลายระดับ
local function connect_with_timeout(host, port, timeout)
    local sock = socket.tcp()

    -- ตั้ง timeout สำหรับการ connect
    sock:settimeout(timeout or 5)

    local ok, err = sock:connect(host, port)
    if not ok then
        sock:close()
        if err == "timeout" then
            return nil, string.format("Connection timeout (%.1fs)", timeout)
        end
        return nil, "Connection refused: " .. tostring(err)
    end

    return sock
end

-- Wrapper ที่ retry ถ้า timeout
local function reliable_connect(host, port, opts)
    opts = opts or {}
    local max_retries = opts.retries or 3
    local timeout     = opts.timeout or 5
    local backoff     = opts.backoff or 1.0

    for attempt = 1, max_retries do
        local sock, err = connect_with_timeout(host, port, timeout)
        if sock then
            print(string.format("Connected on attempt %d", attempt))
            return sock
        end

        print(string.format("Attempt %d/%d failed: %s",
            attempt, max_retries, err))

        if attempt < max_retries then
            print(string.format("Retrying in %.1f seconds...", backoff))
            socket.sleep(backoff)
            backoff = backoff * 2  -- exponential backoff
        end
    end

    return nil, "All retries failed"
end

-- ทดสอบ timeout handling
print("=== Timeout Handling ===")

-- ทดสอบ connection ที่ timeout
local start = socket.gettime()
local sock, err = connect_with_timeout("192.0.2.1", 80, 2)  -- non-routable IP
local elapsed = socket.gettime() - start

if not sock then
    print(string.format("Timeout test: failed after %.2f seconds", elapsed))
    print("Error:", err)
else
    sock:close()
end

-- Socket timeout settings
local s = socket.tcp()
s:settimeout(3)          -- overall timeout
print("\nTimeout settings:")
print("  Default:", 3, "seconds")

-- Separate timeouts (LuaSocket 3.x)
-- s:settimeout(2, "t")  -- total
-- s:settimeout(1, "b")  -- block

s:close()
```

---

## ตัวอย่างที่ 11: Error Handling ที่ครบถ้วน

```lua
local socket = require("socket")

-- Error types ใน LuaSocket
local function handle_socket_error(err)
    local error_types = {
        ["connection refused"]  = "server ปิดอยู่หรือ port ไม่ถูกต้อง",
        ["timeout"]             = "การเชื่อมต่อใช้เวลานานเกินไป",
        ["host not found"]      = "ไม่พบ hostname",
        ["network unreachable"] = "ไม่มี network",
        ["connection reset"]    = "connection ถูกตัดกะทันหัน",
        ["closed"]              = "connection ถูกปิดแล้ว",
        ["broken pipe"]         = "ส่งข้อมูลไปยัง connection ที่ปิดแล้ว",
    }

    for pattern, description in pairs(error_types) do
        if err and err:lower():find(pattern) then
            return description
        end
    end
    return "Unknown error: " .. tostring(err)
end

-- Robust send/receive
local function safe_send(sock, data, max_retries)
    max_retries = max_retries or 3
    local sent = 0
    local total = #data

    for attempt = 1, max_retries do
        local bytes, err = sock:send(data, sent + 1)
        if bytes then
            sent = bytes
            if sent >= total then
                return true, sent
            end
        else
            if err == "timeout" then
                -- retry
            else
                return false, handle_socket_error(err)
            end
        end
    end

    return false, "Max retries exceeded"
end

local function safe_receive(sock, pattern, timeout)
    sock:settimeout(timeout or 5)
    local data, err = sock:receive(pattern or "*l")
    if data then
        return data
    else
        return nil, handle_socket_error(err)
    end
end

-- ทดสอบ error handling
print("=== Error Handling ===")

local sock = socket.tcp()
sock:settimeout(2)
local ok, err = sock:connect("localhost", 9)  -- discard port (ไม่น่าจะมี)
if not ok then
    print("Expected error:", handle_socket_error(err))
end
sock:close()

-- Test invalid hostname
local sock2 = socket.tcp()
sock2:settimeout(2)
local ok2, err2 = sock2:connect("invalid.hostname.test", 80)
if not ok2 then
    print("DNS error:", handle_socket_error(err2))
end
sock2:close()

print("Error handling examples complete")
```

---

## ตัวอย่างที่ 12: Simple Chat Server

```lua
local socket = require("socket")

-- Multi-client Chat Server
-- ใช้ select() สำหรับ non-blocking I/O

local ChatServer = {}
ChatServer.__index = ChatServer

function ChatServer.new(port)
    local cs = setmetatable({}, ChatServer)
    cs.port    = port
    cs.clients = {}  -- {socket, name, id}
    cs.next_id = 1

    cs.server = socket.tcp()
    cs.server:setoption("reuseaddr", true)
    cs.server:bind("*", port)
    cs.server:listen(10)
    cs.server:settimeout(0)

    print(string.format("Chat server started on port %d", port))
    return cs
end

function ChatServer:broadcast(message, exclude_id)
    local dead = {}
    for i, client in ipairs(self.clients) do
        if client.id ~= exclude_id then
            local ok, err = client.sock:send(message)
            if not ok then
                table.insert(dead, i)
            end
        end
    end

    -- Remove dead clients (ย้อนหลัง)
    for i = #dead, 1, -1 do
        table.remove(self.clients, dead[i])
    end
end

function ChatServer:get_sockets()
    local sockets = {self.server}
    for _, client in ipairs(self.clients) do
        table.insert(sockets, client.sock)
    end
    return sockets
end

function ChatServer:handle_new_connection()
    local sock = self.server:accept()
    if sock then
        sock:settimeout(0)
        local id = self.next_id
        self.next_id = self.next_id + 1

        local ip, port = sock:getpeername()
        local name = string.format("User%d", id)

        table.insert(self.clients, {
            sock = sock,
            name = name,
            id   = id,
        })

        print(string.format("[%s] Connected from %s:%d", name, ip, port))
        self:broadcast(string.format(
            "[SERVER] %s joined the chat\n", name
        ))
        sock:send(string.format(
            "Welcome to Lua Chat! Your name is %s\n", name
        ))
    end
end

function ChatServer:handle_client_message(client_idx)
    local client = self.clients[client_idx]
    local line, err = client.sock:receive("*l")

    if line then
        local msg = string.format("[%s] %s\n", client.name, line)
        print(msg:sub(1, -2))  -- log without newline
        self:broadcast(msg, client.id)
    else
        -- Client disconnected
        local name = client.name
        client.sock:close()
        table.remove(self.clients, client_idx)
        print(string.format("[%s] Disconnected", name))
        self:broadcast(string.format(
            "[SERVER] %s left the chat\n", name
        ))
    end
end

function ChatServer:run(max_ticks)
    max_ticks = max_ticks or 100
    local tick = 0

    while tick < max_ticks do
        local read_set = self:get_sockets()
        local readable, _, err = socket.select(read_set, nil, 0.1)

        if readable then
            for _, s in ipairs(readable) do
                if s == self.server then
                    self:handle_new_connection()
                else
                    -- หา index ของ client
                    for i, client in ipairs(self.clients) do
                        if client.sock == s then
                            self:handle_client_message(i)
                            break
                        end
                    end
                end
            end
        end

        tick = tick + 1
    end
end

-- ทดสอบ chat server ด้วย simulated clients
print("=== Chat Server Demo ===")
local server = ChatServer.new(19980)

-- Simulate clients
local function simulate_client(port, name, messages)
    local c = socket.tcp()
    c:settimeout(1)
    if c:connect("127.0.0.1", port) then
        -- รับ welcome
        c:receive("*l")

        for _, msg in ipairs(messages) do
            c:send(name .. ": " .. msg .. "\n")
            socket.sleep(0.05)
        end
        return c
    end
    return nil
end

-- Run server ใน background ผ่าน coroutines
local clients = {}
local co = coroutine.create(function()
    socket.sleep(0.05)

    local c1 = simulate_client(19980, "Alice",
        {"Hello everyone!", "How's the Lua learning going?"})
    local c2 = simulate_client(19980, "Bob",
        {"Hi Alice!", "Lua is awesome!"})

    if c1 then table.insert(clients, c1) end
    if c2 then table.insert(clients, c2) end
    socket.sleep(0.1)
end)

coroutine.resume(co)
server:run(20)
coroutine.resume(co)

for _, c in ipairs(clients) do c:close() end
server.server:close()
print("Chat server demo complete")
```

---

## ตัวอย่างที่ 13: Port Scanner

```lua
local socket = require("socket")

-- Simple Port Scanner
local function scan_port(host, port, timeout)
    timeout = timeout or 0.5
    local sock = socket.tcp()
    sock:settimeout(timeout)

    local ok, err = sock:connect(host, port)
    sock:close()

    if ok then
        return true, "open"
    elseif err == "connection refused" then
        return false, "closed"
    elseif err == "timeout" then
        return false, "filtered"
    else
        return false, err
    end
end

-- Scan a range of ports
local function port_scan(host, port_range, timeout)
    local results = {
        host     = host,
        open     = {},
        closed   = {},
        filtered = {},
    }

    local start_port, end_port
    if type(port_range) == "table" then
        start_port = port_range[1]
        end_port   = port_range[2]
    else
        start_port = port_range
        end_port   = port_range
    end

    print(string.format("Scanning %s ports %d-%d...",
        host, start_port, end_port))

    local start_time = socket.gettime()

    for port = start_port, end_port do
        local open, status = scan_port(host, port, timeout)
        if open then
            table.insert(results.open, port)
            -- ลอง identify service
            local services = {
                [21]   = "FTP",    [22]   = "SSH",   [23]  = "Telnet",
                [25]   = "SMTP",   [53]   = "DNS",   [80]  = "HTTP",
                [110]  = "POP3",   [143]  = "IMAP",  [443] = "HTTPS",
                [3306] = "MySQL",  [5432] = "PostgreSQL",
                [6379] = "Redis",  [8080] = "HTTP-Alt",
                [27017]= "MongoDB",
            }
            local service = services[port] or "unknown"
            print(string.format("  %d/tcp  OPEN  (%s)", port, service))
        elseif status == "filtered" then
            table.insert(results.filtered, port)
        else
            table.insert(results.closed, port)
        end
    end

    results.elapsed = socket.gettime() - start_time
    return results
end

-- Scan localhost common ports
print("=== Port Scanner ===")
local results = port_scan("127.0.0.1", {1, 1024}, 0.1)

print(string.format("\nScan complete in %.2f seconds:", results.elapsed))
print(string.format("  Open ports: %d", #results.open))
print(string.format("  Closed ports: %d", #results.closed))
print(string.format("  Filtered ports: %d", #results.filtered))

if #results.open > 0 then
    print("Open:", table.concat(results.open, ", "))
end
```

---

## ตัวอย่างที่ 14: HTTP Load Tester

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- Simple Load Tester
local function load_test(url, options)
    options = options or {}
    local requests    = options.requests    or 10
    local concurrency = options.concurrency or 1  -- LuaSocket ไม่ support จริงๆ
    local timeout     = options.timeout     or 5

    local results = {
        total      = 0,
        success    = 0,
        failed     = 0,
        times      = {},
        status_codes = {},
    }

    print(string.format("Load testing %s", url))
    print(string.format("  %d requests, timeout=%.1fs", requests, timeout))
    print(string.rep("-", 50))

    local overall_start = socket.gettime()

    for i = 1, requests do
        local start = socket.gettime()
        local body = {}

        local ok, status = http.request{
            url     = url,
            sink    = ltn12.sink.table(body),
            headers = {
                ["User-Agent"] = "LuaLoadTester/1.0",
                ["Connection"] = "close",
            },
        }

        local elapsed = socket.gettime() - start
        results.total = results.total + 1

        if ok and status == 200 then
            results.success = results.success + 1
            table.insert(results.times, elapsed)
            results.status_codes[status] = (results.status_codes[status] or 0) + 1
            io.write(".")
        else
            results.failed = results.failed + 1
            results.status_codes[status or 0] = (results.status_codes[status or 0] or 0) + 1
            io.write("F")
        end
        io.flush()
    end

    print()
    results.total_time = socket.gettime() - overall_start

    -- คำนวณ statistics
    if #results.times > 0 then
        table.sort(results.times)
        local sum = 0
        for _, t in ipairs(results.times) do sum = sum + t end
        results.avg_time = sum / #results.times
        results.min_time = results.times[1]
        results.max_time = results.times[#results.times]
        results.p50 = results.times[math.floor(#results.times * 0.50)]
        results.p95 = results.times[math.floor(#results.times * 0.95)]
        results.p99 = results.times[math.floor(#results.times * 0.99)]
        results.rps = results.total / results.total_time
    end

    return results
end

local function print_results(r)
    print("\nLoad Test Results:")
    print(string.rep("=", 40))
    print(string.format("Total Requests:  %d", r.total))
    print(string.format("Successful:      %d (%.1f%%)",
        r.success, r.success / r.total * 100))
    print(string.format("Failed:          %d", r.failed))
    print(string.format("Total Time:      %.3f seconds", r.total_time))
    print(string.format("Requests/sec:    %.1f", r.rps or 0))

    if r.avg_time then
        print(string.format("\nResponse Times:"))
        print(string.format("  Min:  %6.1f ms", r.min_time * 1000))
        print(string.format("  Avg:  %6.1f ms", r.avg_time * 1000))
        print(string.format("  Max:  %6.1f ms", r.max_time * 1000))
        print(string.format("  P50:  %6.1f ms", (r.p50 or 0) * 1000))
        print(string.format("  P95:  %6.1f ms", (r.p95 or 0) * 1000))
        print(string.format("  P99:  %6.1f ms", (r.p99 or 0) * 1000))
    end

    print("\nStatus Codes:")
    for code, count in pairs(r.status_codes) do
        print(string.format("  %s: %d", tostring(code), count))
    end
end

-- Run load test
local results = load_test("http://httpbin.org/get", {
    requests = 5,
    timeout  = 5,
})
print_results(results)
```

---

## ตัวอย่างที่ 15: SMTP Client

```lua
local smtp   = require("socket.smtp")
local mime   = require("mime")
local ltn12  = require("ltn12")

-- SMTP Email Sender
local function send_email(config)
    -- config = {
    --   from = "sender@example.com",
    --   to   = {"recipient@example.com"},
    --   subject = "Test Email",
    --   body = "Email body",
    --   server = "smtp.example.com",
    --   port   = 587,
    --   user   = "username",
    --   password = "password"
    -- }

    -- สร้าง message source
    local msg_body = table.concat({
        "Date: " .. os.date("%a, %d %b %Y %H:%M:%S +0000"),
        "From: " .. config.from,
        "To: " .. table.concat(config.to, ", "),
        "Subject: " .. (config.subject or "No Subject"),
        "MIME-Version: 1.0",
        "Content-Type: text/plain; charset=UTF-8",
        "",
        config.body or "",
    }, "\r\n")

    local result, err = smtp.send{
        from     = config.from,
        rcpt     = config.to,
        source   = ltn12.source.string(msg_body),
        server   = config.server or "localhost",
        port     = config.port or 25,
        user     = config.user,
        password = config.password,
    }

    return result, err
end

-- HTML Email
local function send_html_email(config)
    local boundary = "----LuaMailBoundary" .. os.time()

    local msg_parts = {
        "Date: " .. os.date("%a, %d %b %Y %H:%M:%S +0000"),
        "From: " .. config.from,
        "To: " .. table.concat(config.to, ", "),
        "Subject: " .. (config.subject or "No Subject"),
        "MIME-Version: 1.0",
        "Content-Type: multipart/alternative; boundary=" .. boundary,
        "",
        "--" .. boundary,
        "Content-Type: text/plain; charset=UTF-8",
        "",
        config.text or "Please view as HTML",
        "",
        "--" .. boundary,
        "Content-Type: text/html; charset=UTF-8",
        "",
        config.html or "<p>" .. (config.text or "") .. "</p>",
        "",
        "--" .. boundary .. "--",
    }

    local msg_body = table.concat(msg_parts, "\r\n")

    -- Demo: แสดง email structure
    print("Email structure:")
    print(msg_body:sub(1, 500))
end

-- ตัวอย่างการใช้งาน (ไม่ส่งจริง)
print("=== SMTP Client Example ===")

local email_config = {
    from     = "sender@example.com",
    to       = {"recipient@example.com", "cc@example.com"},
    subject  = "Hello from Lua!",
    body     = "This is a test email sent from Lua using LuaSocket.",
    server   = "smtp.gmail.com",
    port     = 587,
    user     = "your_email@gmail.com",
    password = "your_app_password",
}

print("Email configuration:")
print("  From:", email_config.from)
print("  To:", table.concat(email_config.to, ", "))
print("  Subject:", email_config.subject)
print("  Server:", email_config.server .. ":" .. email_config.port)
print("\n(ไม่ได้ส่งจริง - ต้องใส่ credentials จริงๆ)")

send_html_email{
    from    = "test@example.com",
    to      = {"user@example.com"},
    subject = "HTML Email Test",
    text    = "This is plain text",
    html    = "<h1>Hello!</h1><p>This is <b>HTML</b> email</p>",
}
```

---

## ตัวอย่างที่ 16: URL Parsing

```lua
local url = require("socket.url")

-- URL parsing
local function parse_url(url_string)
    local parts = url.parse(url_string)
    return {
        scheme   = parts.scheme,
        host     = parts.host,
        port     = parts.port,
        path     = parts.path,
        query    = parts.query,
        fragment = parts.fragment,
        user     = parts.user,
        password = parts.password,
    }
end

-- URL building
local function build_url(parts)
    return url.build(parts)
end

-- ทดสอบ URL parsing
print("=== URL Parsing ===")

local test_urls = {
    "https://user:pass@example.com:8080/path/to/page?key=value&foo=bar#section",
    "http://api.example.com/v1/users?limit=10&offset=0",
    "ftp://files.example.com/public/file.zip",
    "http://localhost:3000/api/data",
}

for _, u in ipairs(test_urls) do
    print("\nURL:", u)
    local parts = parse_url(u)
    for k, v in pairs(parts) do
        if v then
            print(string.format("  %-10s = %s", k, tostring(v)))
        end
    end
end

-- URL encoding/decoding
print("\n=== URL Encoding ===")
local raw = "Hello World! สวัสดี + special=chars&more=data"
local encoded = url.escape(raw)
print("Original:", raw)
print("Encoded:", encoded)
print("Decoded:", url.unescape(encoded))

-- Query string parsing
local function parse_query(query_string)
    local params = {}
    for key, value in (query_string or ""):gmatch("([^&=]+)=([^&]*)") do
        params[url.unescape(key)] = url.unescape(value)
    end
    return params
end

local query = "name=John+Doe&age=30&city=New+York&lang=Lua%205.4"
local params = parse_query(query)
print("\nQuery parameters:")
for k, v in pairs(params) do
    print(string.format("  %s = %s", k, v))
end
```

---

## ตัวอย่างที่ 17: Ping Implementation

```lua
local socket = require("socket")

-- TCP Ping (ใช้ TCP connect แทน ICMP)
-- ICMP ต้องการ root privileges แต่ TCP ping ไม่ต้อง

local function tcp_ping(host, port, timeout)
    port    = port or 80
    timeout = timeout or 2

    local start = socket.gettime()
    local sock  = socket.tcp()
    sock:settimeout(timeout)

    local ok, err = sock:connect(host, port)
    local elapsed = (socket.gettime() - start) * 1000  -- ms

    sock:close()

    if ok then
        return elapsed, nil
    else
        return nil, err
    end
end

-- Ping multiple times และ คำนวณ stats
local function ping(host, port, count, interval)
    count    = count    or 4
    interval = interval or 1.0
    port     = port     or 80

    print(string.format("PING %s port %d:", host, port))

    local times    = {}
    local lost     = 0

    for i = 1, count do
        local ms, err = tcp_ping(host, port, 5)
        if ms then
            table.insert(times, ms)
            print(string.format("  Seq %d: %.2f ms", i, ms))
        else
            lost = lost + 1
            print(string.format("  Seq %d: timeout (%s)", i, err))
        end

        if i < count then
            socket.sleep(interval)
        end
    end

    -- Statistics
    if #times > 0 then
        table.sort(times)
        local sum = 0
        for _, t in ipairs(times) do sum = sum + t end
        local avg = sum / #times

        print(string.format("\n--- ping statistics ---"))
        print(string.format("%d packets sent, %d received, %d%% packet loss",
            count, #times, math.floor(lost/count * 100)))
        print(string.format("round-trip min/avg/max = %.2f/%.2f/%.2f ms",
            times[1], avg, times[#times]))
    else
        print("All packets lost!")
    end
end

-- ทดสอบ ping
print("=== TCP Ping ===")
ping("google.com", 80, 3, 0.5)
print()
ping("localhost", 22, 3, 0.5)
```

---

## ตัวอย่างที่ 18: HTTP GET/POST Requests

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")
local url    = require("socket.url")

-- Complete HTTP Client
local HTTPClient = {}
HTTPClient.__index = HTTPClient

function HTTPClient.new(base_url, options)
    local client = setmetatable({}, HTTPClient)
    client.base_url  = base_url or ""
    client.timeout   = (options or {}).timeout or 10
    client.headers   = (options or {}).headers or {}
    client.cookies   = {}
    return client
end

function HTTPClient:request(method, path, options)
    options = options or {}
    local full_url = self.base_url .. path

    local req_headers = {}
    -- Copy default headers
    for k, v in pairs(self.headers) do
        req_headers[k] = v
    end
    -- Copy request-specific headers
    for k, v in pairs(options.headers or {}) do
        req_headers[k] = v
    end

    req_headers["User-Agent"] = req_headers["User-Agent"] or "LuaHTTP/1.0"
    req_headers["Connection"] = "close"

    -- Add cookies
    local cookie_str = {}
    for name, value in pairs(self.cookies) do
        table.insert(cookie_str, name .. "=" .. value)
    end
    if #cookie_str > 0 then
        req_headers["Cookie"] = table.concat(cookie_str, "; ")
    end

    local body = options.body
    if options.json then
        -- Basic JSON serialization
        body = require("json") and require("json").encode(options.json)
               or '{"error": "json module not found"}'
        req_headers["Content-Type"] = "application/json"
    elseif options.form then
        local parts = {}
        for k, v in pairs(options.form) do
            table.insert(parts, url.escape(k) .. "=" .. url.escape(tostring(v)))
        end
        body = table.concat(parts, "&")
        req_headers["Content-Type"] = "application/x-www-form-urlencoded"
    end

    if body then
        req_headers["Content-Length"] = #body
    end

    local response_body = {}
    local result, status, resp_headers = http.request{
        url     = full_url,
        method  = method,
        headers = req_headers,
        sink    = ltn12.sink.table(response_body),
        source  = body and ltn12.source.string(body) or nil,
    }

    -- Extract cookies
    if resp_headers then
        for _, cookie_val in ipairs{resp_headers["set-cookie"] or {}} do
            local name, value = cookie_val:match("^([^=]+)=([^;]*)")
            if name then
                self.cookies[name] = value
            end
        end
    end

    return {
        ok      = result ~= nil,
        status  = status,
        headers = resp_headers,
        body    = table.concat(response_body),
    }
end

function HTTPClient:get(path, options)
    return self:request("GET", path, options)
end

function HTTPClient:post(path, options)
    return self:request("POST", path, options)
end

function HTTPClient:put(path, options)
    return self:request("PUT", path, options)
end

function HTTPClient:delete(path, options)
    return self:request("DELETE", path, options)
end

-- ตัวอย่างการใช้
print("=== HTTP Client Class ===")

local client = HTTPClient.new("http://httpbin.org", {timeout = 5})

-- GET request
local r1 = client:get("/get")
if r1.ok then
    print("GET /get -> status:", r1.status)
    print("Body length:", #r1.body)
end

-- POST with form data
local r2 = client:post("/post", {
    form = {username = "lua_user", password = "secret"}
})
if r2.ok then
    print("POST /post -> status:", r2.status)
end

-- POST with JSON
local r3 = client:post("/post", {
    body = '{"key": "value", "num": 42}',
    headers = {["Content-Type"] = "application/json"}
})
if r3.ok then
    print("POST JSON -> status:", r3.status)
end
```

---

## ตัวอย่างที่ 19: Binary Data over Network

```lua
local socket = require("socket")

-- Protocol สำหรับ binary data
-- Format: [4-byte length][data]

local function encode_packet(data)
    local len = #data
    -- Big-endian 4-byte length
    local header = string.pack(">I4", len)
    return header .. data
end

local function decode_packet_header(header_bytes)
    if #header_bytes < 4 then return nil end
    local len = string.unpack(">I4", header_bytes)
    return len
end

-- Binary Protocol Server
local function binary_echo_server(port, max_connections)
    local server = socket.tcp()
    server:setoption("reuseaddr", true)
    server:bind("127.0.0.1", port)
    server:listen(5)
    server:settimeout(2)

    local handled = 0
    while handled < max_connections do
        local client = server:accept()
        if client then
            client:settimeout(2)

            -- รับ header
            local header, err = client:receive(4)
            if header then
                local data_len = decode_packet_header(header)
                if data_len and data_len > 0 and data_len <= 65536 then
                    -- รับ data
                    local data, err2 = client:receive(data_len)
                    if data then
                        print(string.format(
                            "Received binary packet: %d bytes: %s...",
                            data_len, data:sub(1, 30)))
                        -- Echo กลับ
                        client:send(encode_packet(data))
                    end
                end
            end
            client:close()
            handled = handled + 1
        end
    end

    server:close()
end

-- ทดสอบ binary protocol
print("=== Binary Protocol Example ===")

-- ทดสอบ encode/decode
local test_data = string.char(0x01, 0x02, 0x03) ..
                  "Hello Binary!" ..
                  string.char(0xFF, 0xFE, 0xFD)

local packet = encode_packet(test_data)
print(string.format("Encoded: %d bytes total, %d header + %d data",
    #packet, 4, #test_data))
print("Header bytes:", string.format("0x%02X 0x%02X 0x%02X 0x%02X",
    packet:byte(1, 4)))

local len = decode_packet_header(packet:sub(1, 4))
print("Decoded length:", len)
print("Data matches:", packet:sub(5) == test_data)

-- Loopback test ด้วย server/client
local server_co = coroutine.create(function()
    binary_echo_server(19970, 1)
end)

coroutine.resume(server_co)

socket.sleep(0.05)

-- Client
local c = socket.tcp()
c:settimeout(2)
if c:connect("127.0.0.1", 19970) then
    local msg = "Binary test data: " .. string.char(0x00, 0x7F, 0xFF)
    c:send(encode_packet(msg))

    local header = c:receive(4)
    if header then
        local len2 = decode_packet_header(header)
        local response = c:receive(len2)
        print("Echo received:", response:sub(1, 30), "...")
        print("Echo match:", response == msg)
    end
    c:close()
end

coroutine.resume(server_co)
print("Binary protocol demo complete")
```

---

## ตัวอย่างที่ 20: Socket Utilities

```lua
local socket = require("socket")

-- Socket utility functions

-- Check if port is in use
local function is_port_open(host, port, timeout)
    local s = socket.tcp()
    s:settimeout(timeout or 1)
    local ok = s:connect(host, port)
    s:close()
    return ok ~= nil
end

-- Get local IP address
local function get_local_ip()
    local s = socket.udp()
    s:setpeername("8.8.8.8", 80)
    local ip = s:getsockname()
    s:close()
    return ip
end

-- Wait for port to become available
local function wait_for_port(host, port, timeout, interval)
    timeout  = timeout  or 30
    interval = interval or 0.5
    local deadline = socket.gettime() + timeout

    while socket.gettime() < deadline do
        if is_port_open(host, port, 1) then
            return true
        end
        socket.sleep(interval)
    end
    return false
end

-- Format bytes
local function format_bytes(bytes)
    local units = {"B", "KB", "MB", "GB", "TB"}
    local i = 1
    while bytes >= 1024 and i < #units do
        bytes = bytes / 1024
        i = i + 1
    end
    return string.format("%.2f %s", bytes, units[i])
end

-- Network speed test (จำลอง)
local function measure_transfer_speed(size_bytes)
    local data = string.rep("X", size_bytes)
    local server = socket.tcp()
    server:setoption("reuseaddr", true)
    server:bind("127.0.0.1", 19960)
    server:listen(1)
    server:settimeout(5)

    local bytes_received = 0
    local server_co = coroutine.create(function()
        local conn = server:accept()
        if conn then
            conn:settimeout(5)
            while true do
                local chunk, err = conn:receive(8192)
                if chunk then
                    bytes_received = bytes_received + #chunk
                else
                    break
                end
            end
            conn:close()
        end
        server:close()
    end)

    coroutine.resume(server_co)

    local client = socket.tcp()
    client:settimeout(5)
    client:connect("127.0.0.1", 19960)

    local start = socket.gettime()
    client:send(data)
    client:close()
    local elapsed = socket.gettime() - start

    coroutine.resume(server_co)

    local speed = size_bytes / elapsed
    return speed, elapsed, bytes_received
end

-- ทดสอบ utilities
print("=== Socket Utilities ===")
print("Local IP:", get_local_ip())
print("Port 80 open (google.com):", is_port_open("google.com", 80, 2))
print("Port 9999 open (localhost):", is_port_open("127.0.0.1", 9999, 0.5))

print("\nTransfer speed test:")
local speed, elapsed, received = measure_transfer_speed(1024 * 1024)  -- 1MB
print(string.format("  Sent: %s in %.3f seconds",
    format_bytes(1024*1024), elapsed))
print(string.format("  Speed: %s/s", format_bytes(speed)))
```

---

## ตัวอย่างที่ 21-30: Pattern การใช้งาน Network ขั้นสูง

```lua
local socket = require("socket")

-- ============================================================
-- 21. Connection Timeout with Retry Backoff
-- ============================================================
local function connect_with_backoff(host, port, opts)
    opts = opts or {}
    local max_attempts = opts.max_attempts or 5
    local base_delay   = opts.base_delay   or 0.5
    local max_delay    = opts.max_delay    or 30
    local timeout      = opts.timeout      or 5

    local delay = base_delay

    for attempt = 1, max_attempts do
        local s = socket.tcp()
        s:settimeout(timeout)

        local ok, err = s:connect(host, port)
        if ok then
            return s, nil
        end
        s:close()

        local jitter = math.random() * 0.5 * delay
        local sleep_time = math.min(delay + jitter, max_delay)

        print(string.format(
            "Attempt %d/%d failed (%s), retry in %.1fs",
            attempt, max_attempts, err, sleep_time))

        if attempt < max_attempts then
            socket.sleep(sleep_time)
        end

        delay = math.min(delay * 2, max_delay)
    end

    return nil, "Max retries exceeded"
end

-- ============================================================
-- 22. Streaming Data Reader
-- ============================================================
local function stream_reader(sock, chunk_callback, delimiter)
    delimiter = delimiter or "\n"
    local buffer = ""

    while true do
        local chunk, err = sock:receive(4096)
        if chunk then
            buffer = buffer .. chunk
            -- Process complete lines
            while true do
                local pos = buffer:find(delimiter, 1, true)
                if not pos then break end
                local line = buffer:sub(1, pos - 1)
                buffer = buffer:sub(pos + #delimiter)
                if chunk_callback(line) == false then
                    return  -- stop
                end
            end
        else
            -- Send remaining buffer
            if #buffer > 0 then
                chunk_callback(buffer)
            end
            if err ~= "closed" then
                print("Stream error:", err)
            end
            return
        end
    end
end

-- ============================================================
-- 23. Rate Limiter
-- ============================================================
local RateLimiter = {}
RateLimiter.__index = RateLimiter

function RateLimiter.new(requests_per_second)
    return setmetatable({
        rate      = requests_per_second,
        tokens    = requests_per_second,
        last_time = socket.gettime(),
    }, RateLimiter)
end

function RateLimiter:acquire()
    local now = socket.gettime()
    local elapsed = now - self.last_time
    self.last_time = now

    -- Add tokens based on elapsed time
    self.tokens = math.min(self.rate, self.tokens + elapsed * self.rate)

    if self.tokens >= 1 then
        self.tokens = self.tokens - 1
        return true, 0
    else
        -- Wait time
        local wait = (1 - self.tokens) / self.rate
        return false, wait
    end
end

function RateLimiter:wait_and_acquire()
    while true do
        local ok, wait = self:acquire()
        if ok then return end
        socket.sleep(wait)
    end
end

-- ============================================================
-- 24. Connection Health Monitor
-- ============================================================
local ConnectionMonitor = {}
ConnectionMonitor.__index = ConnectionMonitor

function ConnectionMonitor.new(host, port, interval)
    local m = setmetatable({}, ConnectionMonitor)
    m.host     = host
    m.port     = port
    m.interval = interval or 5
    m.stats    = {
        checks   = 0,
        success  = 0,
        failures = 0,
        times    = {},
    }
    m.running = false
    return m
end

function ConnectionMonitor:check()
    local start = socket.gettime()
    local s = socket.tcp()
    s:settimeout(3)

    local ok = s:connect(self.host, self.port)
    local elapsed = (socket.gettime() - start) * 1000
    s:close()

    self.stats.checks = self.stats.checks + 1

    if ok then
        self.stats.success = self.stats.success + 1
        table.insert(self.stats.times, elapsed)
        return true, elapsed
    else
        self.stats.failures = self.stats.failures + 1
        return false, nil
    end
end

function ConnectionMonitor:report()
    local s = self.stats
    local availability = s.checks > 0 and
        (s.success / s.checks * 100) or 0

    local avg_ms = 0
    if #s.times > 0 then
        local sum = 0
        for _, t in ipairs(s.times) do sum = sum + t end
        avg_ms = sum / #s.times
    end

    return {
        host         = self.host,
        port         = self.port,
        checks       = s.checks,
        success      = s.success,
        failures     = s.failures,
        availability = availability,
        avg_ms       = avg_ms,
    }
end

-- ============================================================
-- Test All Patterns
-- ============================================================
print("\n=== Advanced Network Patterns ===")

-- Rate limiter test
local limiter = RateLimiter.new(5)  -- 5 req/s
local allowed = 0
for i = 1, 10 do
    local ok, wait = limiter:acquire()
    if ok then
        allowed = allowed + 1
    end
end
print(string.format("Rate limiter: %d/10 allowed immediately", allowed))

-- Connection monitor
local monitor = ConnectionMonitor.new("localhost", 80, 1)
for i = 1, 3 do
    local ok, ms = monitor:check()
    if ok then
        print(string.format("Health check %d: OK (%.1fms)", i, ms))
    else
        print(string.format("Health check %d: FAILED", i))
    end
end

local report = monitor:report()
print(string.format("Availability: %.1f%%", report.availability))
print(string.format("Avg response: %.1fms", report.avg_ms))
```

---

## สรุป LuaSocket

LuaSocket เป็น library ที่ครบครันสำหรับ network programming ใน Lua:

### Features หลัก
- **TCP**: client/server, accept, send/receive
- **UDP**: connectionless protocol
- **HTTP**: GET, POST, headers
- **SMTP**: ส่ง email
- **DNS**: lookup, reverse lookup
- **URL**: parse, encode, decode
- **select()**: multiplexing หลาย connections

### Best Practices

```lua
-- 1. ตั้ง timeout เสมอ
sock:settimeout(5)

-- 2. ตรวจสอบ error ทุกครั้ง
local data, err = sock:receive("*l")
if not data then
    if err == "closed" then
        -- connection closed normally
    elseif err == "timeout" then
        -- handle timeout
    else
        -- other error
    end
end

-- 3. ปิด socket เมื่อเสร็จ
sock:close()

-- 4. ใช้ setoption("reuseaddr") สำหรับ server
server:setoption("reuseaddr", true)

-- 5. จัดการ partial send
local function send_all(sock, data)
    local total = 0
    while total < #data do
        local sent, err = sock:send(data, total + 1)
        if not sent then return nil, err end
        total = sent
    end
    return total
end
```
