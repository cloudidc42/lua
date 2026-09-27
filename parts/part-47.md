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

## ตัวอย่างที่ 22: FTP Client

```lua
local ftp  = require("socket.ftp")
local ltn12 = require("ltn12")

-- FTP Download
local function ftp_download(url_str, local_path)
    local f = io.open(local_path, "wb")
    if not f then
        return nil, "Cannot create file: " .. local_path
    end

    local ok, err = ftp.get{
        url  = url_str,
        sink = ltn12.sink.file(f),
    }

    f:close()

    if ok then
        return true, nil
    else
        os.remove(local_path)
        return nil, "FTP download failed: " .. tostring(err)
    end
end

-- FTP Upload
local function ftp_upload(url_str, local_path)
    local f = io.open(local_path, "rb")
    if not f then
        return nil, "Cannot open file: " .. local_path
    end

    local ok, err = ftp.put{
        url    = url_str,
        source = ltn12.source.file(f),
    }

    f:close()
    return ok and true or nil, err
end

-- FTP List directory
local function ftp_list(url_str)
    local listing = {}
    local ok, err = ftp.get{
        url  = url_str .. "/",
        sink = ltn12.sink.table(listing),
        type = "a",  -- ASCII mode
    }

    if ok then
        return table.concat(listing), nil
    else
        return nil, err
    end
end

print("=== FTP Client ===")
print("FTP functions defined:")
print("  ftp_download(url, local_path)")
print("  ftp_upload(url, local_path)")
print("  ftp_list(url)")
print("\nFTP URL format: ftp://user:pass@host:21/path/file.txt")

-- Example usage (ต้องมี FTP server จริงๆ)
local ftp_examples = {
    "ftp://anonymous:user@ftp.gnu.org/README",
    "ftp://user:pass@myserver.com/data/file.txt",
    "ftp://admin:secret@192.168.1.1:21/uploads/",
}

for _, u in ipairs(ftp_examples) do
    print("  " .. u)
end
```

---

## ตัวอย่างที่ 23: Socket Buffering

```lua
local socket = require("socket")

-- Buffered Socket Wrapper
local BufferedSocket = {}
BufferedSocket.__index = BufferedSocket

function BufferedSocket.new(sock, buffer_size)
    return setmetatable({
        sock        = sock,
        buffer_size = buffer_size or 8192,
        read_buf    = "",
        write_buf   = "",
        bytes_read  = 0,
        bytes_sent  = 0,
    }, BufferedSocket)
end

function BufferedSocket:read_until(delimiter)
    delimiter = delimiter or "\n"

    while true do
        -- ตรวจ buffer ก่อน
        local pos = self.read_buf:find(delimiter, 1, true)
        if pos then
            local line = self.read_buf:sub(1, pos - 1)
            self.read_buf = self.read_buf:sub(pos + #delimiter)
            return line
        end

        -- อ่านเพิ่ม
        local chunk, err = self.sock:receive(self.buffer_size)
        if chunk then
            self.bytes_read = self.bytes_read + #chunk
            self.read_buf = self.read_buf .. chunk
        else
            if err == "timeout" then
                -- return ข้อมูลที่มี ถ้า partial
                if #self.read_buf > 0 then
                    local data = self.read_buf
                    self.read_buf = ""
                    return data, "partial"
                end
            end
            return nil, err
        end
    end
end

function BufferedSocket:read_bytes(n)
    while #self.read_buf < n do
        local chunk, err = self.sock:receive(self.buffer_size)
        if chunk then
            self.bytes_read = self.bytes_read + #chunk
            self.read_buf = self.read_buf .. chunk
        else
            return nil, err
        end
    end

    local data = self.read_buf:sub(1, n)
    self.read_buf = self.read_buf:sub(n + 1)
    return data
end

function BufferedSocket:write(data)
    self.write_buf = self.write_buf .. data
    if #self.write_buf >= self.buffer_size then
        return self:flush()
    end
    return true
end

function BufferedSocket:flush()
    if #self.write_buf == 0 then return true end

    local sent, err = self.sock:send(self.write_buf)
    if sent then
        self.bytes_sent = self.bytes_sent + #self.write_buf
        self.write_buf = ""
        return true
    end
    return nil, err
end

function BufferedSocket:stats()
    return {
        bytes_read = self.bytes_read,
        bytes_sent = self.bytes_sent,
        read_buf_size  = #self.read_buf,
        write_buf_size = #self.write_buf,
    }
end

function BufferedSocket:close()
    self:flush()
    self.sock:close()
end

-- ทดสอบ buffered socket
print("=== Socket Buffering ===")

-- Create server/client pair
local server = socket.tcp()
server:setoption("reuseaddr", true)
server:bind("127.0.0.1", 19940)
server:listen(1)
server:settimeout(2)

local client_sock = socket.tcp()
client_sock:settimeout(2)
client_sock:connect("127.0.0.1", 19940)

local conn_sock = server:accept()

if conn_sock and client_sock then
    local client = BufferedSocket.new(client_sock)
    local conn   = BufferedSocket.new(conn_sock)

    -- Write buffered
    client:write("Hello ")
    client:write("Buffered ")
    client:write("World!\n")
    client:flush()

    -- Read line
    local line, err = conn:read_until("\n")
    if line then
        print("Received:", line)
    end

    -- Stats
    local stats = client:stats()
    print(string.format("Client sent: %d bytes", stats.bytes_sent))

    client:close()
    conn:close()
end

server:close()
print("Buffered socket demo complete")
```

---

## ตัวอย่างที่ 24: Network Protocol - Custom Binary Protocol

```lua
local socket = require("socket")

-- Custom Binary Protocol
-- Packet format:
--   [2 bytes] magic (0xCAFE)
--   [1 byte]  version
--   [1 byte]  type (1=request, 2=response, 3=error)
--   [4 bytes] length (big-endian)
--   [n bytes] payload

local Protocol = {
    MAGIC    = 0xCAFE,
    VERSION  = 1,
    TYPE_REQ = 1,
    TYPE_RES = 2,
    TYPE_ERR = 3,
}

function Protocol.encode(msg_type, payload)
    payload = payload or ""
    local header = string.pack(">I2I1I1I4",
        Protocol.MAGIC,
        Protocol.VERSION,
        msg_type,
        #payload)
    return header .. payload
end

function Protocol.decode_header(bytes)
    if #bytes < 8 then return nil, "Incomplete header" end
    local magic, version, msg_type, length =
        string.unpack(">I2I1I1I4", bytes)

    if magic ~= Protocol.MAGIC then
        return nil, string.format("Invalid magic: 0x%04X", magic)
    end
    if version ~= Protocol.VERSION then
        return nil, string.format("Version mismatch: %d", version)
    end

    return {
        magic    = magic,
        version  = version,
        msg_type = msg_type,
        length   = length,
    }
end

function Protocol.recv_packet(sock)
    -- รับ header
    local header_bytes, err = sock:receive(8)
    if not header_bytes then return nil, err end

    local header, err2 = Protocol.decode_header(header_bytes)
    if not header then return nil, err2 end

    -- รับ payload
    local payload = ""
    if header.length > 0 then
        payload, err = sock:receive(header.length)
        if not payload then return nil, err end
    end

    return {
        type    = header.msg_type,
        payload = payload,
    }
end

-- ทดสอบ protocol
print("=== Custom Binary Protocol ===")

-- Test encode/decode
local req_packet = Protocol.encode(Protocol.TYPE_REQ, "GET /api/users")
print(string.format("Request packet: %d bytes", #req_packet))
print("Magic:", string.format("0x%04X", req_packet:byte(1) * 256 + req_packet:byte(2)))

local header = Protocol.decode_header(req_packet:sub(1, 8))
if header then
    local types = {[1]="REQUEST", [2]="RESPONSE", [3]="ERROR"}
    print("Type:", types[header.msg_type])
    print("Length:", header.length)
end

local resp_packet = Protocol.encode(Protocol.TYPE_RES,
    '{"users": [{"id": 1, "name": "Alice"}]}')
print(string.format("\nResponse packet: %d bytes", #resp_packet))

-- Loopback test
local server = socket.tcp()
server:setoption("reuseaddr", true)
server:bind("127.0.0.1", 19930)
server:listen(1)
server:settimeout(2)

local co = coroutine.create(function()
    local conn = server:accept()
    if conn then
        conn:settimeout(2)
        local pkt = Protocol.recv_packet(conn)
        if pkt then
            print("Server got payload:", pkt.payload)
            -- Send response
            conn:send(Protocol.encode(Protocol.TYPE_RES, "OK: " .. pkt.payload))
        end
        conn:close()
    end
    server:close()
end)

coroutine.resume(co)

local c = socket.tcp()
c:settimeout(2)
if c:connect("127.0.0.1", 19930) then
    c:send(Protocol.encode(Protocol.TYPE_REQ, "PING"))
    local resp = Protocol.recv_packet(c)
    if resp then
        print("Client got response:", resp.payload)
    end
    c:close()
end

coroutine.resume(co)
print("Binary protocol test complete")
```

---

## ตัวอย่างที่ 25: Concurrent Downloads

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- Simulated concurrent downloads ด้วย coroutines
-- (LuaSocket ไม่มี threading จริงๆ แต่ใช้ coroutines + select)

local Downloader = {}
Downloader.__index = Downloader

function Downloader.new()
    local dl = setmetatable({}, Downloader)
    dl.tasks     = {}
    dl.results   = {}
    dl.completed = 0
    dl.failed    = 0
    return dl
end

function Downloader:add_url(url_str, save_as)
    table.insert(self.tasks, {
        url     = url_str,
        save_as = save_as,
        status  = "pending",
    })
end

function Downloader:download_one(task)
    local start = socket.gettime()
    local chunks = {}

    local ok, status, headers = http.request{
        url     = task.url,
        sink    = ltn12.sink.table(chunks),
        headers = {["User-Agent"] = "LuaDownloader/1.0"},
    }

    local elapsed = socket.gettime() - start
    local body = table.concat(chunks)

    if ok and status == 200 then
        -- บันทึกไฟล์ถ้ากำหนด
        if task.save_as then
            local f = io.open(task.save_as, "wb")
            if f then
                f:write(body)
                f:close()
            end
        end

        return {
            url     = task.url,
            status  = status,
            bytes   = #body,
            elapsed = elapsed,
            ok      = true,
        }
    else
        return {
            url     = task.url,
            status  = status or 0,
            elapsed = elapsed,
            ok      = false,
            error   = not ok and tostring(status) or nil,
        }
    end
end

function Downloader:run()
    local start = socket.gettime()

    for _, task in ipairs(self.tasks) do
        io.write(string.format("Downloading %s...", task.url:sub(-30)))
        io.flush()

        local result = self:download_one(task)
        table.insert(self.results, result)

        if result.ok then
            self.completed = self.completed + 1
            print(string.format(" OK (%d bytes, %.2fs)",
                result.bytes, result.elapsed))
        else
            self.failed = self.failed + 1
            print(string.format(" FAILED (%s)", result.error or tostring(result.status)))
        end
    end

    local total_time = socket.gettime() - start
    return {
        total     = #self.tasks,
        completed = self.completed,
        failed    = self.failed,
        time      = total_time,
    }
end

-- ทดสอบ
print("=== Concurrent Downloads ===")

local dl = Downloader.new()
dl:add_url("http://httpbin.org/bytes/1024",   "/tmp/file1.bin")
dl:add_url("http://httpbin.org/bytes/2048",   "/tmp/file2.bin")
dl:add_url("http://httpbin.org/json",          "/tmp/data.json")
dl:add_url("http://httpbin.org/status/404",    nil)
dl:add_url("http://httpbin.org/uuid",          nil)

local stats = dl:run()

print(string.format("\nDownload summary: %d/%d ok, %d failed, %.2fs total",
    stats.completed, stats.total, stats.failed, stats.time))
```

---

## ตัวอย่างที่ 26: WebSocket Handshake (HTTP Upgrade)

```lua
local socket = require("socket")
local mime   = require("mime")

-- WebSocket Handshake ด้วย LuaSocket
-- (WebSocket frame parsing ไม่ครอบคลุมใน LuaSocket ต้องทำเอง)

local function create_websocket_key()
    -- สร้าง random 16-byte key แล้ว base64 encode
    local bytes = {}
    for i = 1, 16 do
        bytes[i] = string.char(math.random(0, 255))
    end
    return mime.b64(table.concat(bytes))
end

local function compute_websocket_accept(key)
    -- WebSocket accept key = base64(SHA1(key + GUID))
    -- ใน LuaSocket ต้องใช้ library เพิ่ม สำหรับ SHA1
    -- นี่เป็น conceptual example
    local magic_guid = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
    return "computed_" .. key:sub(1, 8) .. "..."  -- simplified
end

local function ws_handshake_client(host, port, path)
    local sock = socket.tcp()
    sock:settimeout(5)

    local ok, err = sock:connect(host, port)
    if not ok then return nil, err end

    local key = create_websocket_key()

    -- ส่ง HTTP Upgrade request
    local request = table.concat({
        "GET " .. path .. " HTTP/1.1",
        "Host: " .. host,
        "Upgrade: websocket",
        "Connection: Upgrade",
        "Sec-WebSocket-Key: " .. key,
        "Sec-WebSocket-Version: 13",
        "Origin: http://" .. host,
        "",
        "",
    }, "\r\n")

    sock:send(request)

    -- รับ response
    local lines = {}
    while true do
        local line = sock:receive("*l")
        if not line or line == "" then break end
        table.insert(lines, line)
    end

    -- ตรวจสอบ 101 Switching Protocols
    if lines[1] and lines[1]:find("101") then
        return sock, nil, key
    else
        sock:close()
        return nil, "WebSocket handshake failed: " .. (lines[1] or "no response")
    end
end

-- WebSocket Frame Encoder (basic)
local function ws_encode_frame(data, opcode)
    opcode = opcode or 0x1  -- text frame
    local payload = data
    local len = #payload

    local header = {}

    -- FIN + opcode
    table.insert(header, string.char(0x80 | opcode))

    -- Mask bit + payload length
    if len <= 125 then
        table.insert(header, string.char(0x80 | len))
    elseif len <= 65535 then
        table.insert(header, string.char(0x80 | 126))
        table.insert(header, string.pack(">I2", len))
    else
        table.insert(header, string.char(0x80 | 127))
        table.insert(header, string.pack(">I8", len))
    end

    -- Masking key (4 random bytes)
    local mask = {
        math.random(0, 255), math.random(0, 255),
        math.random(0, 255), math.random(0, 255),
    }
    for _, b in ipairs(mask) do
        table.insert(header, string.char(b))
    end

    -- Mask payload
    local masked = {}
    for i = 1, #payload do
        masked[i] = string.char(
            payload:byte(i) ~ mask[((i-1) % 4) + 1]
        )
    end

    return table.concat(header) .. table.concat(masked)
end

print("=== WebSocket Handshake ===")
print("WebSocket key:", create_websocket_key())
print("Frame encoding:")
local frame = ws_encode_frame("Hello, WebSocket!")
print(string.format("  Encoded %d bytes -> %d bytes",
    #"Hello, WebSocket!", #frame))
print("  (Full WS support needs SHA1 library)")
print("\nFor production use: lua-resty-websocket (OpenResty)")
print("or: websocket-lua library")
```

---

## ตัวอย่างที่ 27: Network Statistics และ Monitoring

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- Network Statistics Collector
local NetStats = {}
NetStats.__index = NetStats

function NetStats.new()
    return setmetatable({
        measurements = {},
        start_time   = socket.gettime(),
    }, NetStats)
end

function NetStats:measure(name, func)
    local start  = socket.gettime()
    local ok, result = pcall(func)
    local elapsed = socket.gettime() - start

    table.insert(self.measurements, {
        name    = name,
        elapsed = elapsed,
        ok      = ok,
        result  = ok and result or tostring(result),
        time    = start,
    })

    return ok and result or nil, not ok and tostring(result) or nil
end

function NetStats:report()
    print("\nNetwork Statistics Report")
    print(string.rep("=", 60))
    print(string.format("%-30s %10s  %s",
        "Measurement", "Time (ms)", "Status"))
    print(string.rep("-", 60))

    local total_time = 0
    local success_count = 0

    for _, m in ipairs(self.measurements) do
        local ms = m.elapsed * 1000
        total_time = total_time + ms

        if m.ok then success_count = success_count + 1 end

        print(string.format("%-30s %10.2f  %s",
            m.name:sub(1, 30), ms,
            m.ok and "OK" or "FAIL"))
    end

    print(string.rep("-", 60))
    print(string.format("Total: %d measurements, %d/%d succeeded",
        #self.measurements, success_count, #self.measurements))
    print(string.format("Total time: %.2f ms", total_time))
    if #self.measurements > 0 then
        print(string.format("Average: %.2f ms",
            total_time / #self.measurements))
    end
end

-- ทดสอบ
print("=== Network Statistics ===")

local stats = NetStats.new()

-- DNS lookup
stats:measure("DNS: google.com", function()
    return socket.dns.toip("google.com")
end)

stats:measure("DNS: localhost", function()
    return socket.dns.toip("localhost")
end)

stats:measure("DNS: invalid.xyz", function()
    return socket.dns.toip("invalid.xyz.no.exist")
end)

-- TCP connect
stats:measure("TCP: httpbin.org:80", function()
    local s = socket.tcp()
    s:settimeout(5)
    local ok = s:connect("httpbin.org", 80)
    s:close()
    return ok ~= nil
end)

-- HTTP request timing
stats:measure("HTTP GET /get", function()
    local chunks = {}
    local ok, status = http.request{
        url  = "http://httpbin.org/get",
        sink = ltn12.sink.table(chunks),
    }
    return ok and status or nil
end)

stats:measure("HTTP GET /delay/1 (slow)", function()
    local chunks = {}
    local ok, status = http.request{
        url  = "http://httpbin.org/delay/1",
        sink = ltn12.sink.table(chunks),
    }
    return ok and status or nil
end)

stats:report()
```

---

## ตัวอย่างที่ 28: Simple HTTP Router

```lua
local socket = require("socket")

-- Simple HTTP Server ที่มี routing
local Router = {}
Router.__index = Router

function Router.new()
    local r = setmetatable({}, Router)
    r.routes = {}
    r.middleware = {}
    return r
end

function Router:add_route(method, path_pattern, handler)
    table.insert(self.routes, {
        method  = method:upper(),
        pattern = path_pattern,
        handler = handler,
    })
end

function Router:get(path, handler) self:add_route("GET", path, handler) end
function Router:post(path, handler) self:add_route("POST", path, handler) end
function Router:put(path, handler) self:add_route("PUT", path, handler) end
function Router:delete(path, handler) self:add_route("DELETE", path, handler) end

function Router:use(middleware_func)
    table.insert(self.middleware, middleware_func)
end

function Router:match(method, path)
    for _, route in ipairs(self.routes) do
        if route.method == method then
            -- Simple pattern matching
            local params = {}
            local pattern = route.pattern
                :gsub(":(%w+)", function(name)
                    params[#params + 1] = name
                    return "([^/]+)"
                end)
                :gsub("%*", "(.+)")

            local matches = {path:match("^" .. pattern .. "$")}
            if #matches > 0 then
                local named_params = {}
                for i, name in ipairs(params) do
                    named_params[name] = matches[i]
                end
                return route.handler, named_params
            end
        end
    end
    return nil, {}
end

-- HTTP Response helpers
local function make_response(status, body, content_type)
    content_type = content_type or "text/plain"
    return {
        status       = status,
        body         = body or "",
        content_type = content_type,
    }
end

local function json_response(data)
    local body = type(data) == "string" and data or
        string.format('{"result": "%s"}', tostring(data))
    return make_response(200, body, "application/json")
end

local function text_response(text, status)
    return make_response(status or 200, text, "text/plain")
end

-- สร้าง simple router
local app = Router.new()

-- Routes
app:get("/", function(req)
    return text_response("Welcome to Lua HTTP Server!")
end)

app:get("/hello/:name", function(req, params)
    return text_response("Hello, " .. (params.name or "World") .. "!")
end)

app:get("/api/users", function(req)
    return json_response('{"users": [{"id": 1, "name": "Alice"}]}')
end)

app:get("/api/users/:id", function(req, params)
    return json_response(string.format(
        '{"user": {"id": %s, "name": "User %s"}}',
        params.id, params.id))
end)

app:post("/api/echo", function(req)
    return json_response(string.format(
        '{"echo": "%s"}', req.body or ""))
end)

-- ทดสอบ routing
print("=== HTTP Router ===")

local test_requests = {
    {method = "GET",  path = "/"},
    {method = "GET",  path = "/hello/Alice"},
    {method = "GET",  path = "/hello/สมชาย"},
    {method = "GET",  path = "/api/users"},
    {method = "GET",  path = "/api/users/42"},
    {method = "POST", path = "/api/echo", body = "test data"},
    {method = "GET",  path = "/notfound"},
    {method = "DELETE", path = "/api/users/1"},
}

for _, req in ipairs(test_requests) do
    local handler, params = app:match(req.method, req.path)
    if handler then
        local resp = handler(req, params)
        print(string.format("  %s %-25s -> %d: %s",
            req.method, req.path, resp.status,
            resp.body:sub(1, 50)))
    else
        print(string.format("  %s %-25s -> 404: Not Found",
            req.method, req.path))
    end
end
```

---

## ตัวอย่างที่ 29: Keep-Alive Connections

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- HTTP Keep-Alive Connection Pool
-- (LuaSocket ปิด connection หลัง request โดย default)
-- นี่เป็น pattern สำหรับ reusing connections

local KeepAlivePool = {}
KeepAlivePool.__index = KeepAlivePool

function KeepAlivePool.new(max_connections_per_host)
    return setmetatable({
        pools     = {},  -- {host:port -> [connections]}
        max_conns = max_connections_per_host or 5,
        stats     = {reused = 0, new = 0, closed = 0},
    }, KeepAlivePool)
end

function KeepAlivePool:_pool_key(host, port)
    return host .. ":" .. port
end

function KeepAlivePool:get_connection(host, port)
    local key = self:_pool_key(host, port)
    local pool = self.pools[key]

    if pool and #pool > 0 then
        -- ดึง connection จาก pool
        local conn = table.remove(pool)
        -- ตรวจสอบว่ายังใช้งานได้
        conn:settimeout(0)
        local _, err = conn:receive(1)
        conn:settimeout(5)

        if err ~= "closed" then
            self.stats.reused = self.stats.reused + 1
            return conn, true  -- true = reused
        else
            conn:close()
            self.stats.closed = self.stats.closed + 1
        end
    end

    -- สร้าง connection ใหม่
    local conn = socket.tcp()
    conn:settimeout(5)
    local ok, err = conn:connect(host, port)
    if not ok then
        return nil, false, err
    end

    self.stats.new = self.stats.new + 1
    return conn, false
end

function KeepAlivePool:return_connection(host, port, conn)
    local key = self:_pool_key(host, port)
    self.pools[key] = self.pools[key] or {}

    if #self.pools[key] < self.max_conns then
        table.insert(self.pools[key], conn)
    else
        conn:close()
        self.stats.closed = self.stats.closed + 1
    end
end

function KeepAlivePool:get_stats()
    return self.stats
end

-- ทดสอบ
print("=== Keep-Alive Connection Pool ===")

local pool = KeepAlivePool.new(3)

-- Simulate connection reuse
print("Connection pool stats:")
print("  max connections per host:", 3)

-- สร้างและ return connections (simulated)
local host, port = "httpbin.org", 80
local conns = {}

for i = 1, 3 do
    local conn, reused, err = pool:get_connection(host, port)
    if conn then
        table.insert(conns, conn)
        print(string.format("  Got connection %d: %s",
            i, reused and "REUSED" or "NEW"))
    else
        print(string.format("  Failed to get connection %d: %s", i, tostring(err)))
    end
end

-- Return connections to pool
for i, conn in ipairs(conns) do
    pool:return_connection(host, port, conn)
    print(string.format("  Returned connection %d to pool", i))
end

local stats = pool:get_stats()
print(string.format("\nPool stats: new=%d reused=%d closed=%d",
    stats.new, stats.reused, stats.closed))
```

---

## ตัวอย่างที่ 30: Network Testing Utilities

```lua
local socket = require("socket")
local http   = require("socket.http")
local ltn12  = require("ltn12")

-- Network Testing Toolkit

-- ตรวจสอบ internet connectivity
local function check_internet(timeout)
    timeout = timeout or 3
    local hosts = {
        {"8.8.8.8",         53},
        {"1.1.1.1",         53},
        {"google.com",      80},
        {"cloudflare.com",  80},
    }

    for _, h in ipairs(hosts) do
        local s = socket.tcp()
        s:settimeout(timeout)
        local ok = s:connect(h[1], h[2])
        s:close()
        if ok then return true, h[1] end
    end

    return false, nil
end

-- Bandwidth estimation (rough)
local function estimate_bandwidth(url, size_bytes)
    size_bytes = size_bytes or 102400  -- 100KB default
    local chunks = {}
    local start = socket.gettime()

    local ok, status = http.request{
        url  = string.format("http://httpbin.org/bytes/%d", size_bytes),
        sink = ltn12.sink.table(chunks),
        headers = {["User-Agent"] = "LuaBandwidthTest/1.0"},
    }

    local elapsed = socket.gettime() - start

    if ok and status == 200 then
        local received = 0
        for _, chunk in ipairs(chunks) do
            received = received + #chunk
        end

        local kbps = (received / 1024) / elapsed
        local mbps = kbps / 1024

        return {
            bytes    = received,
            elapsed  = elapsed,
            kbps     = kbps,
            mbps     = mbps,
        }
    end

    return nil
end

-- Traceroute-like (TCP based)
local function tcp_traceroute(host, port, max_hops)
    max_hops = max_hops or 10
    print(string.format("TCP traceroute to %s:%d (max %d hops)",
        host, port, max_hops))

    for ttl = 1, max_hops do
        local s = socket.tcp()
        s:settimeout(1)

        local start = socket.gettime()
        local ok, err = s:connect(host, port)
        local elapsed = (socket.gettime() - start) * 1000
        s:close()

        if ok then
            print(string.format("  %2d  %s (reached in %.1fms)",
                ttl, host, elapsed))
            break
        else
            print(string.format("  %2d  * (%.1fms %s)",
                ttl, elapsed, err or "?"))
        end
    end
end

-- ทดสอบ
print("=== Network Testing Utilities ===\n")

-- Check internet
print("Internet connectivity check:")
local connected, via = check_internet(3)
if connected then
    print("  Connected via:", via)
else
    print("  No internet connection detected")
end

-- Bandwidth test
print("\nBandwidth estimation (100KB test):")
local bw = estimate_bandwidth(nil, 102400)
if bw then
    print(string.format("  Downloaded: %d bytes in %.3fs",
        bw.bytes, bw.elapsed))
    print(string.format("  Speed: %.1f KB/s (%.2f Mbps)",
        bw.kbps, bw.mbps))
else
    print("  Bandwidth test failed (no internet)")
end

-- TCP traceroute
print("\nTCP traceroute:")
tcp_traceroute("google.com", 80, 5)

-- Network summary
print("\n--- Network Summary ---")
print(string.format("  Socket version: %s", socket._VERSION))
print(string.format("  Local hostname: %s", socket.dns.tohostname("127.0.0.1") or "localhost"))
local local_ip = socket.dns.toip("localhost")
print(string.format("  Local IP: %s", local_ip or "127.0.0.1"))
```

---

## ตัวอย่างที่ 31: DNS Advanced Lookup

DNS lookup ขั้นสูงด้วย LuaSocket รองรับ A records และ DNS cache

```lua
local socket = require("socket")
local dns = socket.dns

-- DNS Cache implementation
local DNSCache = {}
DNSCache.__index = DNSCache

function DNSCache.new(ttl)
    return setmetatable({ cache = {}, ttl = ttl or 300 }, DNSCache)
end

function DNSCache:get(hostname)
    local entry = self.cache[hostname]
    if not entry then return nil end
    if os.time() - entry.timestamp > self.ttl then
        self.cache[hostname] = nil
        return nil
    end
    return entry.ip
end

function DNSCache:set(hostname, ip)
    self.cache[hostname] = { ip = ip, timestamp = os.time() }
end

function DNSCache:resolve(hostname)
    local cached = self:get(hostname)
    if cached then return cached, "cached" end
    local ip, err = dns.toip(hostname)
    if ip then
        self:set(hostname, ip)
        return ip, "fresh"
    end
    return nil, err
end

function DNSCache:stats()
    local count = 0
    for _ in pairs(self.cache) do count = count + 1 end
    return { entries = count, ttl = self.ttl }
end

local cache = DNSCache.new(60)

-- Resolve with cache
local hosts = { "localhost", "localhost", "127.0.0.1" }
for _, h in ipairs(hosts) do
    local ip, source = cache:resolve(h)
    print(string.format("  resolve(%s) -> %s [%s]", h, tostring(ip), source))
end

-- Reverse lookup
local function reverse_lookup(ip)
    local host, err = dns.tohostname(ip)
    return host or ("error: " .. tostring(err))
end
print("reverse(127.0.0.1) ->", reverse_lookup("127.0.0.1"))

-- Batch resolver
local function batch_resolve(hostnames)
    local results = {}
    for _, hostname in ipairs(hostnames) do
        local ip, err = dns.toip(hostname)
        results[hostname] = { ip = ip, error = err }
    end
    return results
end

local resolved = batch_resolve({ "localhost" })
for host, info in pairs(resolved) do
    print(string.format("  batch: %s -> %s", host, tostring(info.ip)))
end

local s = cache:stats()
print(string.format("Cache stats: %d entries, TTL=%ds", s.entries, s.ttl))
```

---

## ตัวอย่างที่ 32: TCP Echo Server with Timeout

TCP Echo Server รองรับ timeout และ multiple clients ด้วย socket.select()

```lua
local socket = require("socket")

local function run_echo_server(host, port, max_seconds)
    local server = socket.tcp()
    server:setoption("reuseaddr", true)
    local ok, err = server:bind(host or "127.0.0.1", port or 9998)
    if not ok then return nil, "bind: " .. tostring(err) end
    server:listen(5)
    server:settimeout(0)  -- non-blocking

    print(string.format("[EchoServer] Listening %s:%d", host or "127.0.0.1", port or 9998))

    local clients = {}
    local start = os.time()

    while os.time() - start < (max_seconds or 2) do
        -- Accept new connections
        local client = server:accept()
        if client then
            client:settimeout(0)
            local ip, p = client:getpeername()
            print(string.format("  Client connected: %s:%d", tostring(ip), tostring(p)))
            table.insert(clients, client)
        end

        -- Service existing clients
        local readable = socket.select(clients, nil, 0.05)
        local to_remove = {}
        for _, c in ipairs(readable) do
            local line, err2 = c:receive("*l")
            if line then
                c:send("ECHO: " .. line .. "\r\n")
            elseif err2 == "closed" then
                c:close()
                for i, cc in ipairs(clients) do
                    if cc == c then table.insert(to_remove, i) end
                end
            end
        end
        for i = #to_remove, 1, -1 do table.remove(clients, to_remove[i]) end
    end

    for _, c in ipairs(clients) do c:close() end
    server:close()
    print("[EchoServer] Stopped, served", #clients + 0, "connections")
    return true
end

-- Run for 1 second (demo)
run_echo_server("127.0.0.1", 9997, 1)
```

---

## ตัวอย่างที่ 33: UDP Multicast Service Discovery

UDP Multicast สำหรับ service discovery ในเครือข่าย local

```lua
local socket = require("socket")

local MULTICAST_GROUP = "224.0.0.251"
local MULTICAST_PORT  = 5353

-- Message encoder/decoder
local function encode_announcement(service, port, meta)
    return string.format("SERVICE:%s:%d:%s", service, port, meta or "")
end

local function decode_announcement(msg)
    local svc, port, meta = msg:match("^SERVICE:([^:]+):(%d+):(.*)$")
    if svc then
        return { name = svc, port = tonumber(port), meta = meta }
    end
end

-- Service Registry (simulation)
local ServiceRegistry = {}
ServiceRegistry.__index = ServiceRegistry

function ServiceRegistry.new(name, port)
    return setmetatable({
        name = name, port = port,
        known = {}
    }, ServiceRegistry)
end

function ServiceRegistry:announce_message()
    return encode_announcement(self.name, self.port, "v=1.0")
end

function ServiceRegistry:process_message(msg, from_ip)
    local info = decode_announcement(msg)
    if info then
        info.ip = from_ip
        self.known[info.name .. "@" .. from_ip] = info
        return info
    end
end

function ServiceRegistry:list_services()
    local result = {}
    for _, svc in pairs(self.known) do
        table.insert(result, svc)
    end
    return result
end

-- Demo
print("=== UDP Multicast Service Discovery ===")
local reg = ServiceRegistry.new("api-server", 8080)

-- Simulate receiving announcements
local fake_messages = {
    { msg = encode_announcement("web-ui", 3000, "v=2.1"), ip = "192.168.1.10" },
    { msg = encode_announcement("auth-svc", 3001, "jwt=true"), ip = "192.168.1.11" },
    { msg = encode_announcement("db-proxy", 5432, "readonly=false"), ip = "192.168.1.12" },
    { msg = "INVALID_MSG", ip = "192.168.1.99" },
}

for _, fm in ipairs(fake_messages) do
    local info = reg:process_message(fm.msg, fm.ip)
    if info then
        print(string.format("  Discovered: %s at %s:%d (%s)",
            info.name, info.ip, info.port, info.meta))
    end
end

print("\nAnnouncement:", reg:announce_message())
print("Known services:", #reg:list_services())

-- Show UDP multicast send code pattern
print("\nMulticast send pattern:")
print("  sock = socket.udp()")
print("  sock:setoption('ip-multicast-ttl', 1)")
print("  sock:sendto(msg, '224.0.0.251', 5353)")
```

---

## ตัวอย่างที่ 34: TCP Connection Pool

Connection Pool สำหรับ TCP connections ประสิทธิภาพสูง

```lua
local socket = require("socket")

local TCPPool = {}
TCPPool.__index = TCPPool

function TCPPool.new(host, port, opts)
    opts = opts or {}
    return setmetatable({
        host = host, port = port,
        max_size  = opts.max_size  or 10,
        timeout   = opts.timeout   or 5,
        idle_ttl  = opts.idle_ttl  or 60,
        pool      = {},   -- { sock, created_at, last_used, id }
        active    = 0,
        created   = 0,
        errors    = 0,
    }, TCPPool)
end

function TCPPool:_new_conn()
    local conn = socket.tcp()
    conn:settimeout(self.timeout)
    local ok, err = conn:connect(self.host, self.port)
    if not ok then
        conn:close()
        self.errors = self.errors + 1
        return nil, err
    end
    self.created = self.created + 1
    return { sock = conn, created_at = os.time(),
             last_used = os.time(), id = self.created }
end

function TCPPool:_healthy(c)
    if not c then return false end
    if os.time() - c.last_used > self.idle_ttl then return false end
    -- non-blocking peek
    c.sock:settimeout(0)
    local _, err = c.sock:receive(0)
    c.sock:settimeout(self.timeout)
    return err ~= "closed"
end

function TCPPool:acquire()
    -- Return a healthy pooled connection
    while #self.pool > 0 do
        local c = table.remove(self.pool)
        if self:_healthy(c) then
            self.active = self.active + 1
            c.last_used = os.time()
            return c
        end
        c.sock:close()
    end
    -- Create new
    if self.active < self.max_size then
        local c, err = self:_new_conn()
        if c then
            self.active = self.active + 1
            return c
        end
        return nil, err
    end
    return nil, "pool exhausted"
end

function TCPPool:release(c)
    self.active = math.max(0, self.active - 1)
    if self:_healthy(c) and #self.pool < self.max_size then
        c.last_used = os.time()
        table.insert(self.pool, c)
    else
        c.sock:close()
    end
end

function TCPPool:with_conn(fn)
    local c, err = self:acquire()
    if not c then return nil, err end
    local ok, res = pcall(fn, c.sock)
    self:release(c)
    if not ok then return nil, res end
    return res
end

function TCPPool:stats()
    return { pool = #self.pool, active = self.active,
             created = self.created, errors = self.errors }
end

-- Demo
print("=== TCP Connection Pool ===")
local pool = TCPPool.new("127.0.0.1", 9999, { max_size = 5, timeout = 1 })

-- Simulate usage without actual server
print("Pool stats:", "pool=" .. pool:stats().pool,
    "active=" .. pool:stats().active,
    "created=" .. pool:stats().created)

print("\nUsage pattern:")
print("  result = pool:with_conn(function(sock)")
print("      sock:send('PING\\r\\n')")
print("      return sock:receive('*l')")
print("  end)")
print("\nBenefits:")
print("  - Reuse TCP handshakes")
print("  - Limit max concurrent connections")
print("  - Auto-health check on acquire")
print("  - Idle connection timeout")
```

---

## ตัวอย่างที่ 35: Binary Message Framing

Binary protocol framing สำหรับ reliable message delivery ผ่าน TCP streams

```lua
local socket = require("socket")

-- Frame: [magic:2][type:1][flags:1][length:4][checksum:2][payload:N]
local MAGIC = 0xAB1C
local HEADER_LEN = 10

local MsgType = { PING=1, PONG=2, DATA=3, ERROR=4, AUTH=5 }

local function cksum(data)
    local s = 0
    for i = 1, #data do s = (s + data:byte(i)) & 0xFFFF end
    return s
end

local function encode_frame(mtype, payload, flags)
    payload = payload or ""
    flags   = flags   or 0
    local hdr = string.pack(">I2 B B I4 I2",
        MAGIC, mtype, flags, #payload, cksum(payload))
    return hdr .. payload
end

local function decode_frame(raw)
    if #raw < HEADER_LEN then return nil, "incomplete" end
    local magic, mtype, flags, length, expected_cksum =
        string.unpack(">I2 B B I4 I2", raw)
    if magic ~= MAGIC then
        return nil, string.format("bad magic 0x%04X", magic)
    end
    if #raw < HEADER_LEN + length then return nil, "need more data" end
    local payload = raw:sub(HEADER_LEN + 1, HEADER_LEN + length)
    if cksum(payload) ~= expected_cksum then
        return nil, "checksum error"
    end
    return { type=mtype, flags=flags, payload=payload,
             consumed=HEADER_LEN + length }
end

-- FrameBuffer reassembles frames from stream
local FrameBuffer = {}
FrameBuffer.__index = FrameBuffer
function FrameBuffer.new() return setmetatable({buf=""}, FrameBuffer) end
function FrameBuffer:feed(data) self.buf = self.buf .. data end
function FrameBuffer:next()
    local frame, err = decode_frame(self.buf)
    if not frame then
        if err == "incomplete" or err == "need more data" then return nil end
        self.buf = self.buf:sub(2)  -- skip bad byte
        return nil, err
    end
    self.buf = self.buf:sub(frame.consumed + 1)
    return frame
end

-- Demo
print("=== Binary Message Framing ===")
local msgs = {
    { MsgType.AUTH, '{"user":"alice","pw":"secret"}' },
    { MsgType.PING, "" },
    { MsgType.DATA, "Hello binary world!" },
    { MsgType.ERROR, "permission denied" },
}

local stream = ""
for _, m in ipairs(msgs) do
    local f = encode_frame(m[1], m[2])
    stream = stream .. f
    print(string.format("  encoded type=%d payload=%d bytes frame=%d bytes",
        m[1], #m[2], #f))
end
print(string.format("Stream total: %d bytes\n", #stream))

local type_names = { [1]="AUTH", [2]="PING", [3]="DATA", [4]="PONG", [5]="ERROR" }
-- Oops, fix: 4=ERROR, 5=AUTH above — correct now
type_names[MsgType.AUTH] = "AUTH"
type_names[MsgType.PING] = "PING"
type_names[MsgType.DATA] = "DATA"
type_names[MsgType.ERROR] = "ERROR"

local fb = FrameBuffer.new()
fb:feed(stream)
while true do
    local frame = fb:next()
    if not frame then break end
    print(string.format("  decoded [%s] payload=%q",
        type_names[frame.type] or "?", frame.payload))
end

-- Corruption test
print("\nCorruption detection:")
local good = encode_frame(MsgType.DATA, "test")
local bad  = good:sub(1,-2) .. string.char(good:byte(-1) ~ 0xFF)
print("  good frame:", decode_frame(good) and "OK" or "FAIL")
local r, e = decode_frame(bad)
print("  corrupted:", r and "OK" or ("FAIL: " .. tostring(e)))
```

---

## ตัวอย่างที่ 36: HTTP Long Polling

Long Polling pattern สำหรับ real-time updates ผ่าน HTTP

```lua
local socket = require("socket")

local MessageBus = {}
MessageBus.__index = MessageBus

function MessageBus.new()
    return setmetatable({ messages = {}, next_id = 1 }, MessageBus)
end

function MessageBus:publish(channel, data)
    local msg = { id=self.next_id, channel=channel, data=data, ts=os.time() }
    self.next_id = self.next_id + 1
    table.insert(self.messages, msg)
    if #self.messages > 500 then table.remove(self.messages, 1) end
    return msg
end

function MessageBus:poll(channel, since_id, timeout_s)
    local deadline = socket.gettime() + (timeout_s or 30)
    while socket.gettime() < deadline do
        local result = {}
        for _, m in ipairs(self.messages) do
            if m.channel == channel and m.id > since_id then
                table.insert(result, m)
            end
        end
        if #result > 0 then return result end
        socket.sleep(0.05)
    end
    return {}
end

-- Simple JSON encode
local function json_arr(items)
    local parts = {}
    for _, item in ipairs(items) do
        table.insert(parts, string.format(
            '{"id":%d,"data":%q,"ts":%d}', item.id, item.data, item.ts))
    end
    return "[" .. table.concat(parts, ",") .. "]"
end

-- Demo
print("=== HTTP Long Polling Demo ===")
local bus = MessageBus.new()

-- Publish messages
bus:publish("news", "Lua 5.5 released!")
bus:publish("alerts", "High CPU warning")
bus:publish("news", "New package available")
bus:publish("chat", "Hello everyone")

-- Poll channel
local news = bus:poll("news", 0, 0.1)
print(string.format("Poll 'news' since_id=0: got %d messages", #news))
for _, m in ipairs(news) do
    print(string.format("  [id=%d] %s", m.id, m.data))
end

-- Poll with timeout (no new messages)
local latest_id = news[#news] and news[#news].id or 0
local new_msgs = bus:poll("news", latest_id, 0.2)
print(string.format("\nPoll 'news' since_id=%d (0.2s timeout): %d new",
    latest_id, #new_msgs))

-- HTTP response format
local body = json_arr(news)
print("\nHTTP Response body:")
print(string.format("  Content-Type: application/json"))
print(string.format("  Content-Length: %d", #body))
print("  Body: " .. body:sub(1, 100))

print("\nClient pattern:")
print("  last_id = 0")
print("  while true do")
print("    msgs = poll('/events?channel=news&since=' .. last_id, timeout=30)")
print("    for _, m in ipairs(msgs) do process(m) end")
print("    last_id = msgs[#msgs] and msgs[#msgs].id or last_id")
print("  end")
```

---

## ตัวอย่างที่ 37: SMTP Email Builder

สร้าง MIME email messages ด้วย LuaSocket smtp module

```lua
local socket = require("socket")
local smtp   = require("socket.smtp")

-- Email builder
local Email = {}
Email.__index = Email

function Email.new()
    return setmetatable({
        _from = nil, _to = {}, _cc = {}, _bcc = {},
        _subject = "(no subject)",
        _text = nil, _html = nil,
        _headers = {}
    }, Email)
end
function Email:from(addr)    self._from = addr; return self end
function Email:to(addr)      table.insert(self._to, addr); return self end
function Email:cc(addr)      table.insert(self._cc, addr); return self end
function Email:subject(s)    self._subject = s; return self end
function Email:text(t)       self._text = t; return self end
function Email:html(h)       self._html = h; return self end
function Email:header(k, v)  self._headers[k] = v; return self end

function Email:rcpt_list()
    local all = {}
    for _, a in ipairs(self._to) do table.insert(all, "<"..a..">") end
    for _, a in ipairs(self._cc) do table.insert(all, "<"..a..">") end
    return all
end

function Email:source()
    -- Build headers table for smtp.message
    local hdrs = {
        from    = self._from,
        to      = table.concat(self._to, ", "),
        subject = self._subject,
    }
    if #self._cc > 0 then hdrs.cc = table.concat(self._cc, ", ") end
    for k, v in pairs(self._headers) do hdrs[k] = v end

    -- Body: prefer HTML with text fallback (multipart/alternative)
    if self._html and self._text then
        local boundary = "mixed_" .. math.floor(os.time())
        return smtp.message {
            headers = hdrs,
            body = {
                preamble = "This is a MIME email",
                [1] = {
                    headers = { ["content-type"] = "text/plain; charset=UTF-8" },
                    body = self._text
                },
                [2] = {
                    headers = { ["content-type"] = "text/html; charset=UTF-8" },
                    body = self._html
                }
            }
        }
    end
    hdrs["content-type"] = self._html
        and "text/html; charset=UTF-8"
        or  "text/plain; charset=UTF-8"
    return smtp.message { headers = hdrs, body = self._html or self._text or "" }
end

-- SMTP sender
local function send_email(email, server_cfg)
    server_cfg = server_cfg or {}
    local source = email:source()
    local ok, err = smtp.send {
        from   = "<" .. email._from .. ">",
        rcpt   = email:rcpt_list(),
        source = source,
        server = server_cfg.host or "127.0.0.1",
        port   = server_cfg.port or 1025,
        user   = server_cfg.user,
        password = server_cfg.password,
    }
    return ok, err
end

-- Demo
print("=== SMTP Email Builder Demo ===")
local email = Email.new()
    :from("sender@example.com")
    :to("recipient@example.com")
    :cc("manager@example.com")
    :subject("Lua Tutorial - Part 37: SMTP")
    :text("สวัสดี!\n\nนี่คือตัวอย่าง SMTP ด้วย Lua\n\nขอบคุณ")
    :html("<h1>สวัสดี!</h1><p>ตัวอย่าง <b>HTML email</b> ด้วย Lua</p>")

print("From:", email._from)
print("To:", table.concat(email._to, ", "))
print("Cc:", table.concat(email._cc, ", "))
print("Subject:", email._subject)
print("Recipients:", table.concat(email:rcpt_list(), ", "))

print("\nTo send (requires SMTP server):")
print("  ok, err = send_email(email, { host='smtp.gmail.com', port=587,")
print("      user='you@gmail.com', password=os.getenv('SMTP_PASS') })")

-- Template emails
local function template_email(to, name, order_id, total)
    return Email.new()
        :from("shop@example.com")
        :to(to)
        :subject(string.format("Order #%s Confirmed", order_id))
        :text(string.format(
            "Dear %s,\nYour order #%s for $%.2f has been confirmed.",
            name, order_id, total))
        :html(string.format(
            "<p>Dear <strong>%s</strong>,</p>"..
            "<p>Order <code>#%s</code> for <strong>$%.2f</strong> confirmed.</p>",
            name, order_id, total))
end

local order_email = template_email("customer@example.com", "Alice", "ORD-123", 99.99)
print("\nTemplate email subject:", order_email._subject)
```

---

## ตัวอย่างที่ 38: Service Health Monitor

Monitor TCP services ด้วย health checks และ alerting

```lua
local socket = require("socket")

local Monitor = {}
Monitor.__index = Monitor

function Monitor.new()
    return setmetatable({
        services = {}, alerts = {}, handlers = {}
    }, Monitor)
end

function Monitor:add(name, host, port, opts)
    opts = opts or {}
    self.services[name] = {
        name = name, host = host, port = port,
        timeout   = opts.timeout   or 3,
        threshold = opts.threshold or 3,
        status    = "unknown",
        failures  = 0, successes = 0,
        latency   = 0, history   = {}
    }
end

function Monitor:on_alert(fn) table.insert(self.handlers, fn) end

function Monitor:_emit(event, svc)
    for _, fn in ipairs(self.handlers) do fn(event, svc) end
    table.insert(self.alerts, { event=event, service=svc.name, time=os.time() })
end

function Monitor:check(name)
    local s = self.services[name]
    if not s then return nil, "not found" end
    local t0 = socket.gettime()
    local conn = socket.tcp()
    conn:settimeout(s.timeout)
    local ok, err = conn:connect(s.host, s.port)
    local lat = (socket.gettime() - t0) * 1000
    conn:close()

    s.latency = lat
    local prev = s.status

    if ok then
        s.failures = 0
        s.successes = s.successes + 1
        s.status = "up"
        table.insert(s.history, { up=true, latency=lat, time=os.time() })
        if prev == "down" then self:_emit("recovery", s) end
    else
        s.failures = s.failures + 1
        table.insert(s.history, { up=false, error=err, time=os.time() })
        if s.failures >= s.threshold then
            s.status = "down"
            if prev ~= "down" then self:_emit("failure", s) end
        else
            s.status = "degraded"
        end
    end
    return s.status, lat
end

function Monitor:check_all()
    local results = {}
    for name in pairs(self.services) do
        results[name] = { self:check(name) }
    end
    return results
end

function Monitor:uptime(name)
    local s = self.services[name]
    if not s or #s.history == 0 then return 0 end
    local up = 0
    for _, h in ipairs(s.history) do if h.up then up = up + 1 end end
    return up / #s.history * 100
end

-- Demo
print("=== Service Health Monitor ===")
local mon = Monitor.new()
mon:on_alert(function(event, svc)
    print(string.format("  [ALERT] %s: %s", event:upper(), svc.name))
end)

mon:add("http",    "127.0.0.1", 80,   { timeout=1, threshold=2 })
mon:add("https",   "127.0.0.1", 443,  { timeout=1, threshold=2 })
mon:add("redis",   "127.0.0.1", 6379, { timeout=1, threshold=2 })
mon:add("mysql",   "127.0.0.1", 3306, { timeout=1, threshold=2 })

print("\nRunning checks...")
mon:check_all()

print("\nService Status:")
print(string.format("  %-15s %-10s %-12s %-8s", "Name", "Status", "Latency(ms)", "Uptime%"))
print(string.rep("-", 50))
local names = {}
for n in pairs(mon.services) do table.insert(names, n) end
table.sort(names)
for _, name in ipairs(names) do
    local s = mon.services[name]
    print(string.format("  %-15s %-10s %-12.1f %-8.1f",
        name, s.status, s.latency, mon:uptime(name)))
end
print(string.format("\nAlerts generated: %d", #mon.alerts))
```

---

## ตัวอย่างที่ 39: SOCKS5 Proxy Client

Connect ผ่าน SOCKS5 proxy สำหรับ tunneling

```lua
local socket = require("socket")

-- SOCKS5 constants
local S5_VER      = 5
local S5_NO_AUTH  = 0x00
local S5_CMD_CONN = 0x01
local S5_ATYP_IP4 = 0x01
local S5_ATYP_DOM = 0x03
local S5_REP_OK   = 0x00

local SOCKS5 = {}
SOCKS5.__index = SOCKS5

function SOCKS5.new(proxy_host, proxy_port, user, pass)
    return setmetatable({
        proxy_host = proxy_host, proxy_port = proxy_port,
        user = user, pass = pass, sock = nil
    }, SOCKS5)
end

function SOCKS5:connect(host, port, timeout)
    timeout = timeout or 10
    local sock = socket.tcp()
    sock:settimeout(timeout)

    local ok, err = sock:connect(self.proxy_host, self.proxy_port)
    if not ok then return nil, "proxy connect: " .. tostring(err) end

    -- Step 1: greeting
    local methods = self.user and "\x05\x02\x00\x02" or "\x05\x01\x00"
    sock:send(methods)
    local resp, rerr = sock:receive(2)
    if not resp then sock:close(); return nil, "greeting: " .. tostring(rerr) end

    local ver, method = resp:byte(1), resp:byte(2)
    if ver ~= S5_VER then sock:close(); return nil, "bad version" end

    -- Step 2: auth
    if method == 0x02 then
        local u, p = self.user or "", self.pass or ""
        sock:send("\x01" .. string.char(#u) .. u .. string.char(#p) .. p)
        local ar = sock:receive(2)
        if not ar or ar:byte(2) ~= 0 then
            sock:close(); return nil, "auth failed"
        end
    elseif method ~= S5_NO_AUTH then
        sock:close(); return nil, "no acceptable auth"
    end

    -- Step 3: CONNECT request
    local req
    if host:match("^%d+%.%d+%.%d+%.%d+$") then
        local a,b,c,d = host:match("(%d+)%.(%d+)%.(%d+)%.(%d+)")
        req = string.pack(">BBBB BBBB I2",
            S5_VER, S5_CMD_CONN, 0, S5_ATYP_IP4,
            tonumber(a), tonumber(b), tonumber(c), tonumber(d), port)
    else
        req = string.pack(">BBBB Bc0 I2",
            S5_VER, S5_CMD_CONN, 0, S5_ATYP_DOM, host, port)
    end
    sock:send(req)

    local hdr = sock:receive(4)
    if not hdr then sock:close(); return nil, "no reply" end
    local _, rep, _, atyp = hdr:byte(1), hdr:byte(2), hdr:byte(3), hdr:byte(4)
    if rep ~= S5_REP_OK then
        sock:close(); return nil, string.format("SOCKS5 error %d", rep)
    end

    -- skip bound addr
    if atyp == S5_ATYP_IP4 then sock:receive(6)
    elseif atyp == S5_ATYP_DOM then
        local lb = sock:receive(1); sock:receive(lb:byte(1) + 2)
    else sock:receive(18) end  -- IPv6

    self.sock = sock
    return sock
end

function SOCKS5:send(d)    return self.sock and self.sock:send(d) end
function SOCKS5:receive(p) return self.sock and self.sock:receive(p) end
function SOCKS5:close()    if self.sock then self.sock:close(); self.sock=nil end end

-- Demo
print("=== SOCKS5 Proxy Client ===")
print("Protocol negotiation steps:")
print("  1. TCP connect to proxy")
print("  2. Greeting: send supported auth methods")
print("  3. Server picks method (no-auth or user/pass)")
print("  4. Authenticate (if required)")
print("  5. CONNECT request with target host:port")
print("  6. Proxy connects, returns bound addr")
print("  7. Tunnel open — send/receive normally")

-- Show packet construction
local target_host = "example.com"
local target_port = 443
local req = string.pack(">BBBB Bc0 I2",
    S5_VER, S5_CMD_CONN, 0, S5_ATYP_DOM, target_host, target_port)
print(string.format("\nCONNECT packet: %d bytes for %s:%d",
    #req, target_host, target_port))
print("Usage:")
print("  proxy = SOCKS5.new('proxy.example.com', 1080, 'user', 'pass')")
print("  sock  = proxy:connect('api.example.com', 443)")
print("  sock:send('GET / HTTP/1.0\\r\\n\\r\\n')")
print("  print(proxy:receive('*a'))")
```

---

## ตัวอย่างที่ 40: Network Packet Analysis

Packet capture simulation พร้อม traffic analysis

```lua
local socket = require("socket")

local PTYPE = { TCP="TCP", UDP="UDP", ICMP="ICMP" }

local Capture = {}
Capture.__index = Capture

function Capture.new(filter)
    return setmetatable({
        packets = {}, filter = filter or {},
        stats = { total=0, bytes=0, tcp=0, udp=0, icmp=0,
                  by_src={}, by_dst={}, by_port={} }
    }, Capture)
end

function Capture:_matches(p)
    local f = self.filter
    if f.src_ip  and p.src_ip  ~= f.src_ip  then return false end
    if f.dst_ip  and p.dst_ip  ~= f.dst_ip  then return false end
    if f.port    and p.dst_port ~= f.port and p.src_port ~= f.port then return false end
    if f.ptype   and p.ptype   ~= f.ptype   then return false end
    return true
end

function Capture:add(src_ip, dst_ip, src_port, dst_port, ptype, payload)
    local p = { src_ip=src_ip, dst_ip=dst_ip, src_port=src_port,
                dst_port=dst_port, ptype=ptype, payload=payload,
                ts=socket.gettime(), len=#payload+40 }
    if not self:_matches(p) then return false end
    table.insert(self.packets, p)
    local s = self.stats
    s.total = s.total + 1
    s.bytes = s.bytes + p.len
    if ptype == PTYPE.TCP  then s.tcp  = s.tcp  + 1
    elseif ptype == PTYPE.UDP  then s.udp  = s.udp  + 1
    elseif ptype == PTYPE.ICMP then s.icmp = s.icmp + 1 end
    s.by_src[src_ip] = (s.by_src[src_ip] or 0) + 1
    s.by_dst[dst_ip] = (s.by_dst[dst_ip] or 0) + 1
    local pk = tostring(dst_port)
    s.by_port[pk] = (s.by_port[pk] or 0) + 1
    return true
end

function Capture:top(tbl, n)
    local list = {}
    for k, v in pairs(tbl) do table.insert(list, {k=k, v=v}) end
    table.sort(list, function(a,b) return a.v > b.v end)
    local r = {}
    for i = 1, math.min(n or 5, #list) do r[i] = list[i] end
    return r
end

function Capture:summary()
    local s = self.stats
    return string.format(
        "%d packets (%d bytes) TCP:%d UDP:%d ICMP:%d",
        s.total, s.bytes, s.tcp, s.udp, s.icmp)
end

local PORT_NAMES = {
    [80]="HTTP",[443]="HTTPS",[22]="SSH",[53]="DNS",
    [25]="SMTP",[3306]="MySQL",[5432]="Postgres",
    [6379]="Redis",[8080]="HTTP-ALT",[27017]="MongoDB"
}

-- Demo
print("=== Network Packet Analysis ===")
local cap = Capture.new()

local test_data = {
    {"192.168.1.10","10.0.0.1",  54321, 80,   PTYPE.TCP, "GET / HTTP/1.1"},
    {"192.168.1.10","10.0.0.1",  54322, 443,  PTYPE.TCP, "TLS ClientHello"},
    {"192.168.1.11","8.8.8.8",   12345, 53,   PTYPE.UDP, "DNS query A"},
    {"192.168.1.10","10.0.0.2",  54323, 3306, PTYPE.TCP, "MySQL handshake"},
    {"192.168.1.12","10.0.0.1",  54324, 80,   PTYPE.TCP, "POST /api/v1"},
    {"10.0.0.1",    "192.168.1.10",80, 54321, PTYPE.TCP, "HTTP 200 OK"},
    {"192.168.1.10","10.0.0.3",  54325, 6379, PTYPE.TCP, "PING"},
    {"192.168.1.11","10.0.0.1",  54326, 443,  PTYPE.TCP, "TLS AppData"},
    {"192.168.1.13","8.8.8.8",   12346, 53,   PTYPE.UDP, "DNS query AAAA"},
    {"192.168.1.10","10.0.0.4",  54327, 27017,PTYPE.TCP, "Mongo find()"},
}

for _, d in ipairs(test_data) do cap:add(table.unpack(d)) end

print("Summary:", cap:summary())

print("\nTop source IPs:")
for _, t in ipairs(cap:top(cap.stats.by_src, 3)) do
    print(string.format("  %-18s %d pkts", t.k, t.v))
end

print("\nTop destination ports:")
for _, t in ipairs(cap:top(cap.stats.by_port, 5)) do
    local pnum = tonumber(t.k)
    print(string.format("  %-6s (%s): %d pkts",
        t.k, PORT_NAMES[pnum] or "unknown", t.v))
end

print("\nFirst 3 packets:")
for i = 1, math.min(3, #cap.packets) do
    local p = cap.packets[i]
    print(string.format("  [%s] %s:%d -> %s:%d '%s'",
        p.ptype, p.src_ip, p.src_port, p.dst_ip, p.dst_port,
        p.payload:sub(1,20)))
end

-- Filtered capture
print("\nFiltered (TCP port 443 only):")
local https_cap = Capture.new({ ptype=PTYPE.TCP, port=443 })
for _, d in ipairs(test_data) do https_cap:add(table.unpack(d)) end
print("  " .. https_cap:summary())
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
