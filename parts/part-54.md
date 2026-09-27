# บทที่ 54: WebSocket - Real-time Communication ด้วย Lua

## บทนำ

WebSocket เป็น protocol ที่ช่วยให้ client และ server สามารถสื่อสารแบบ full-duplex (สองทิศทางพร้อมกัน) ผ่าน connection เดียว ทำให้สร้าง real-time applications ได้อย่างมีประสิทธิภาพ เช่น chat, notifications, live updates และอื่นๆ

### WebSocket vs HTTP Polling
```
HTTP Polling:
Client → Request → Server
         ← Response (อาจไม่มีข้อมูลใหม่)
         (รอ N วินาที)
Client → Request → Server
         ← Response ...
         (ทำซ้ำเรื่อยๆ) → bandwidth สูงมาก

WebSocket:
Client → Upgrade Request → Server
         ← Upgrade Response (101 Switching Protocols)
         === Connection Established ===
Client ←→ Server (two-way, persistent connection)
Client ←→ Server
Client ←→ Server  ← real-time!
```

---

## 54.1 WebSocket Protocol Basics

```lua
-- WebSocket Handshake
-- Client ส่ง HTTP Upgrade request:
-- GET /chat HTTP/1.1
-- Host: localhost:8080
-- Upgrade: websocket
-- Connection: Upgrade
-- Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
-- Sec-WebSocket-Version: 13

-- Server ตอบ:
-- HTTP/1.1 101 Switching Protocols
-- Upgrade: websocket
-- Connection: Upgrade
-- Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

-- WebSocket Frame Structure:
-- Bit 0:    FIN (final frame)
-- Bit 1-3:  RSV1-3 (reserved)
-- Bit 4-7:  Opcode
--   0x0 = Continuation frame
--   0x1 = Text frame
--   0x2 = Binary frame
--   0x8 = Close frame
--   0x9 = Ping frame
--   0xA = Pong frame
-- Bit 8:    Mask (1 if client → server)
-- Bit 9-15: Payload length (or 126/127 for extended)
```

---

## 54.2 ติดตั้ง WebSocket Libraries

```bash
# OpenResty มาพร้อมกับ lua-resty-websocket
# ไม่ต้องติดตั้งเพิ่ม

# สำหรับ standalone Lua:
luarocks install lua-websockets  # หรือ
luarocks install websocket        # alternative

# ตรวจสอบ
lua -e "require('resty.websocket.server')"
# ถ้าไม่ error = ติดตั้งแล้ว
```

---

## 54.3 OpenResty WebSocket Server พื้นฐาน

```nginx
# nginx.conf
http {
    lua_shared_dict ws_clients 10m;
    
    server {
        listen 8080;
        
        location /ws {
            content_by_lua_block {
                local server = require "resty.websocket.server"
                
                -- Upgrade connection เป็น WebSocket
                local wb, err = server:new{
                    timeout         = 5000,    -- 5 seconds timeout
                    max_payload_len = 65535    -- max message size
                }
                
                if not wb then
                    ngx.log(ngx.ERR, "WebSocket upgrade failed: ", err)
                    return ngx.exit(444)  -- close connection
                end
                
                ngx.log(ngx.INFO, "New WebSocket connection from: " .. ngx.var.remote_addr)
                
                -- Main loop
                while true do
                    local data, typ, err = wb:recv_frame()
                    
                    if wb.fatal then
                        ngx.log(ngx.ERR, "Fatal error: " .. (err or "unknown"))
                        break
                    end
                    
                    if not data then
                        -- Timeout หรือ connection issue
                        -- ส่ง ping เพื่อตรวจสอบ connection
                        local ok, err = wb:send_ping()
                        if not ok then
                            ngx.log(ngx.ERR, "Ping failed: " .. (err or ""))
                            break
                        end
                        
                    elseif typ == "close" then
                        -- Client ขอปิด connection
                        ngx.log(ngx.INFO, "Client closing connection")
                        wb:send_close()
                        break
                        
                    elseif typ == "ping" then
                        -- ตอบ pong
                        local ok, err = wb:send_pong(data)
                        if not ok then
                            ngx.log(ngx.ERR, "Pong failed: " .. err)
                            break
                        end
                        
                    elseif typ == "text" then
                        -- ได้รับ text message
                        ngx.log(ngx.INFO, "Received text: " .. data)
                        
                        -- Echo back
                        local ok, err = wb:send_text("Echo: " .. data)
                        if not ok then
                            ngx.log(ngx.ERR, "Send failed: " .. err)
                            break
                        end
                        
                    elseif typ == "binary" then
                        -- ได้รับ binary message
                        local ok, err = wb:send_binary(data)
                        if not ok then
                            break
                        end
                    end
                end
                
                wb:close()
            }
        }
    }
}
```

---

## 54.4 Message Types และ Handling

```lua
-- ส่ง different message types
local server = require "resty.websocket.server"

local wb, err = server:new{timeout = 5000}
if not wb then return ngx.exit(444) end

-- ส่ง Text message
local ok, err = wb:send_text("Hello, World!")
if not ok then
    ngx.log(ngx.ERR, "Send text failed: " .. err)
end

-- ส่ง Binary message (เช่น image data, protocol buffers)
local binary_data = "\x48\x65\x6c\x6c\x6f"  -- "Hello" in bytes
local ok, err = wb:send_binary(binary_data)

-- ส่ง Ping (ตรวจสอบว่า connection ยังอยู่)
local ok, err = wb:send_ping("ping payload")

-- ส่ง Pong (ตอบกลับ ping)
local ok, err = wb:send_pong("pong payload")

-- ส่ง Close
local ok, err = wb:send_close(1000, "Normal closure")
-- Codes:
-- 1000 = Normal closure
-- 1001 = Going away
-- 1002 = Protocol error
-- 1003 = Unsupported data
-- 1008 = Policy violation
-- 1011 = Internal server error

-- รับ frame
local data, typ, err = wb:recv_frame()

if typ == "text"   then -- ... end
if typ == "binary" then -- ... end
if typ == "ping"   then -- ... end
if typ == "pong"   then -- ... end
if typ == "close"  then -- ... end
```

---

## 54.5 Connection Lifecycle

```lua
-- Complete connection lifecycle handler
local function handle_websocket_connection()
    local server = require "resty.websocket.server"
    local cjson  = require "cjson"
    
    -- 1. Upgrade connection
    local wb, err = server:new{
        timeout         = 60000,  -- 60 seconds
        max_payload_len = 1024 * 64  -- 64KB
    }
    
    if not wb then
        ngx.log(ngx.ERR, "WebSocket new() failed: " .. (err or "unknown"))
        return ngx.exit(444)
    end
    
    local client_ip = ngx.var.remote_addr
    ngx.log(ngx.INFO, "WebSocket connected: " .. client_ip)
    
    -- 2. Send welcome message
    local welcome = cjson.encode({
        type    = "welcome",
        message = "Connected to WebSocket server",
        time    = ngx.time()
    })
    wb:send_text(welcome)
    
    -- 3. Main message loop
    local ping_interval = 30  -- seconds
    local last_ping     = ngx.time()
    
    while true do
        -- Check if we need to send ping
        local now = ngx.time()
        if now - last_ping >= ping_interval then
            local ok = wb:send_ping()
            if not ok then
                ngx.log(ngx.INFO, "Ping failed, closing: " .. client_ip)
                break
            end
            last_ping = now
        end
        
        local data, typ, err = wb:recv_frame()
        
        -- Handle connection errors
        if wb.fatal then
            ngx.log(ngx.ERR, "Fatal error for " .. client_ip .. ": " .. (err or ""))
            break
        end
        
        if not data then
            -- Timeout - continue (will send ping on next iteration)
            goto continue
        end
        
        -- Handle close
        if typ == "close" then
            ngx.log(ngx.INFO, "Client " .. client_ip .. " closing")
            wb:send_close(1000, "OK")
            break
        end
        
        -- Handle ping from client
        if typ == "ping" then
            wb:send_pong(data)
            goto continue
        end
        
        -- Handle pong from our ping
        if typ == "pong" then
            -- Connection is alive
            goto continue
        end
        
        -- Handle text/binary messages
        if typ == "text" then
            -- Parse JSON message
            local ok, msg = pcall(cjson.decode, data)
            if not ok then
                wb:send_text(cjson.encode({
                    type  = "error",
                    error = "Invalid JSON"
                }))
                goto continue
            end
            
            -- Process message
            local response = process_message(msg, client_ip)
            if response then
                wb:send_text(cjson.encode(response))
            end
        end
        
        ::continue::
    end
    
    -- 4. Cleanup
    ngx.log(ngx.INFO, "WebSocket disconnected: " .. client_ip)
    wb:close()
end

-- Message processor
function process_message(msg, client_ip)
    if msg.type == "ping" then
        return {type = "pong", time = ngx.time()}
    elseif msg.type == "echo" then
        return {type = "echo", data = msg.data}
    elseif msg.type == "time" then
        return {type = "time", time = ngx.time(), iso = os.date("!%Y-%m-%dT%H:%M:%SZ")}
    end
    
    return {type = "error", error = "Unknown message type: " .. (msg.type or "nil")}
end
```

---

## 54.6 Broadcasting ไปยัง Multiple Clients

```nginx
# ใช้ shared memory สำหรับ broadcasting
http {
    lua_shared_dict ws_messages 10m;  -- Queue สำหรับ messages
    lua_shared_dict ws_clients  5m;   -- Connected clients
    
    server {
        listen 8080;
        
        location /ws {
            content_by_lua_block {
                require("websocket_handler").handle()
            }
        }
        
        location /broadcast {
            -- Internal endpoint สำหรับ broadcast
            internal;
            content_by_lua_block {
                local cjson = require "cjson"
                local messages = ngx.shared.ws_messages
                
                ngx.req.read_body()
                local body = ngx.req.get_body_data()
                
                -- เก็บ message ลง queue
                local msg_key = "msg:" .. tostring(ngx.now())
                messages:set(msg_key, body, 30)  -- expire in 30s
                messages:incr("msg_count", 1, 0)
                
                ngx.say('{"ok": true}')
            }
        }
    }
}
```

```lua
-- websocket_handler.lua
local cjson   = require "cjson"
local clients = ngx.shared.ws_clients
local M       = {}

-- Register client
local function register_client(client_id)
    clients:set("client:" .. client_id, 1, 3600)
    clients:incr("total_clients", 1, 0)
end

-- Unregister client
local function unregister_client(client_id)
    clients:delete("client:" .. client_id)
    clients:incr("total_clients", -1, 0)
    if (clients:get("total_clients") or 0) < 0 then
        clients:set("total_clients", 0)
    end
end

function M.handle()
    local server = require "resty.websocket.server"
    
    local wb, err = server:new{timeout = 10000, max_payload_len = 65535}
    if not wb then return ngx.exit(444) end
    
    -- Generate unique client ID
    local client_id = ngx.md5(ngx.var.remote_addr .. tostring(ngx.now()) .. math.random(100000))
    
    register_client(client_id)
    ngx.log(ngx.INFO, "Client " .. client_id .. " connected. Total: " .. 
        (clients:get("total_clients") or 0))
    
    -- Send client their ID
    wb:send_text(cjson.encode({
        type      = "connected",
        client_id = client_id,
        message   = "Welcome!"
    }))
    
    -- Main loop
    local last_msg_count = clients:get("msg_count") or 0
    
    while true do
        local data, typ, err = wb:recv_frame()
        
        if wb.fatal then break end
        
        -- Check for broadcast messages (polling shared dict)
        local current_count = clients:get("msg_count") or 0
        if current_count > last_msg_count then
            -- มี messages ใหม่ ส่งให้ client
            local messages = ngx.shared.ws_messages
            local keys     = messages:keys()
            
            for _, key in ipairs(keys) do
                if key:match("^msg:") then
                    local msg = messages:get(key)
                    if msg then
                        wb:send_text(msg)
                    end
                end
            end
            
            last_msg_count = current_count
        end
        
        if not data then goto continue end
        
        if typ == "close" then
            wb:send_close()
            break
        elseif typ == "ping" then
            wb:send_pong(data)
        elseif typ == "text" then
            local ok, msg = pcall(cjson.decode, data)
            if ok then
                -- Handle client message
                if msg.type == "message" then
                    -- Store as broadcast message
                    local broadcast = cjson.encode({
                        type      = "message",
                        from      = client_id,
                        text      = msg.text,
                        timestamp = ngx.time()
                    })
                    local messages = ngx.shared.ws_messages
                    messages:set("msg:" .. ngx.now(), broadcast, 60)
                    messages:incr("msg_count", 1, 0)
                end
            end
        end
        
        ::continue::
        ngx.sleep(0.1)  -- ป้องกัน busy loop
    end
    
    unregister_client(client_id)
    wb:close()
end

return M
```

---

## 54.7 Real-time Chat Application

```lua
-- chat_server.lua - Complete Chat Server

local cjson   = require "cjson"
local server  = require "resty.websocket.server"

-- Shared state
local rooms    = ngx.shared.chat_rooms    -- room data
local messages = ngx.shared.chat_messages -- message queue

-- Chat handlers
local Chat = {}

function Chat.join_room(wb, client_id, room_name)
    -- Add client to room
    local room_key = "room:" .. room_name .. ":clients"
    local client_list = cjson.decode(rooms:get(room_key) or "[]")
    
    -- Check if already in room
    for _, cid in ipairs(client_list) do
        if cid == client_id then return true end
    end
    
    table.insert(client_list, client_id)
    rooms:set(room_key, cjson.encode(client_list), 3600)
    
    -- Notify others in room
    Chat.broadcast_to_room(room_name, {
        type    = "user_joined",
        user    = client_id,
        room    = room_name,
        time    = ngx.time()
    }, client_id)
    
    -- Send room history to new client
    local history_key = "room:" .. room_name .. ":history"
    local history     = cjson.decode(rooms:get(history_key) or "[]")
    
    wb:send_text(cjson.encode({
        type    = "room_history",
        room    = room_name,
        messages = history
    }))
    
    return true
end

function Chat.leave_room(client_id, room_name)
    local room_key = "room:" .. room_name .. ":clients"
    local client_list = cjson.decode(rooms:get(room_key) or "[]")
    
    local new_list = {}
    for _, cid in ipairs(client_list) do
        if cid ~= client_id then
            table.insert(new_list, cid)
        end
    end
    
    rooms:set(room_key, cjson.encode(new_list), 3600)
    
    -- Notify others
    Chat.broadcast_to_room(room_name, {
        type = "user_left",
        user = client_id,
        room = room_name,
        time = ngx.time()
    }, client_id)
end

function Chat.send_message(client_id, room_name, text)
    local msg = {
        id      = ngx.md5(client_id .. tostring(ngx.now())),
        type    = "message",
        room    = room_name,
        from    = client_id,
        text    = text,
        time    = ngx.time()
    }
    
    -- Store in history (keep last 50 messages)
    local history_key = "room:" .. room_name .. ":history"
    local history     = cjson.decode(rooms:get(history_key) or "[]")
    table.insert(history, msg)
    while #history > 50 do table.remove(history, 1) end
    rooms:set(history_key, cjson.encode(history), 86400)
    
    -- Broadcast to room
    Chat.broadcast_to_room(room_name, msg, nil)
end

function Chat.broadcast_to_room(room_name, msg_data, exclude_client)
    -- Store message in shared queue for all clients to pick up
    local queue_key = "broadcast:" .. room_name .. ":" .. tostring(ngx.now()) .. ":" .. math.random(100000)
    local msg_json  = cjson.encode(msg_data)
    
    messages:set(queue_key, msg_json, 30)
    messages:incr("broadcast_count", 1, 0)
end

-- WebSocket handler
local function handle()
    local wb, err = server:new{
        timeout         = 30000,
        max_payload_len = 1024 * 16  -- 16KB
    }
    
    if not wb then return ngx.exit(444) end
    
    local client_id = ngx.md5(ngx.var.remote_addr .. tostring(ngx.now()))
    local joined_rooms = {}
    
    -- Welcome
    wb:send_text(cjson.encode({
        type      = "welcome",
        client_id = client_id
    }))
    
    local last_broadcast = messages:get("broadcast_count") or 0
    
    -- Main loop
    while true do
        -- Check broadcasts
        local current_broadcast = messages:get("broadcast_count") or 0
        if current_broadcast > last_broadcast then
            -- ส่ง broadcasts ที่ค้างอยู่
            for room_name, _ in pairs(joined_rooms) do
                local prefix = "broadcast:" .. room_name .. ":"
                -- ดึง messages สำหรับ room นี้
                local keys = messages:keys(200)
                for _, k in ipairs(keys) do
                    if k:sub(1, #prefix) == prefix then
                        local msg = messages:get(k)
                        if msg then
                            wb:send_text(msg)
                        end
                    end
                end
            end
            last_broadcast = current_broadcast
        end
        
        -- Receive from client
        local data, typ, err = wb:recv_frame()
        
        if wb.fatal then break end
        if not data then goto continue end
        
        if typ == "close" then
            for room_name, _ in pairs(joined_rooms) do
                Chat.leave_room(client_id, room_name)
            end
            wb:send_close()
            break
        elseif typ == "ping" then
            wb:send_pong(data)
        elseif typ == "text" then
            local ok, msg = pcall(cjson.decode, data)
            if not ok then
                wb:send_text(cjson.encode({type="error", error="Invalid JSON"}))
                goto continue
            end
            
            -- Handle different message types
            if msg.type == "join" then
                local room = msg.room or "general"
                Chat.join_room(wb, client_id, room)
                joined_rooms[room] = true
                wb:send_text(cjson.encode({
                    type   = "joined",
                    room   = room,
                    message = "Joined room: " .. room
                }))
                
            elseif msg.type == "leave" then
                local room = msg.room
                if room and joined_rooms[room] then
                    Chat.leave_room(client_id, room)
                    joined_rooms[room] = nil
                end
                
            elseif msg.type == "message" then
                local room = msg.room or "general"
                if joined_rooms[room] then
                    Chat.send_message(client_id, room, msg.text or "")
                else
                    wb:send_text(cjson.encode({
                        type  = "error",
                        error = "You are not in room: " .. room
                    }))
                end
                
            elseif msg.type == "list_rooms" then
                -- List available rooms
                local room_list = {}
                local keys = rooms:keys()
                for _, k in ipairs(keys) do
                    if k:match("^room:(.+):clients$") then
                        local room_name = k:match("^room:(.+):clients$")
                        table.insert(room_list, room_name)
                    end
                end
                wb:send_text(cjson.encode({type="rooms", rooms=room_list}))
                
            elseif msg.type == "private" then
                -- Private message (simplified - just echo back)
                wb:send_text(cjson.encode({
                    type    = "private",
                    from    = client_id,
                    to      = msg.to,
                    text    = msg.text,
                    time    = ngx.time()
                }))
            end
        end
        
        ::continue::
        ngx.sleep(0.05)  -- 50ms polling interval
    end
    
    wb:close()
end

-- Expose handler
return {handle = handle}
```

---

## 54.8 Client-side WebSocket (JavaScript)

```html
<!-- index.html - WebSocket Client -->
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Chat</title>
</head>
<body>
    <div id="messages"></div>
    <input type="text" id="msg-input" placeholder="Type a message...">
    <button onclick="sendMessage()">Send</button>
    
    <script>
    // WebSocket connection
    const ws = new WebSocket('ws://localhost:8080/ws');
    
    // Connection events
    ws.onopen = function(event) {
        console.log('Connected to WebSocket server');
        addMessage('System', 'Connected!', 'system');
        
        // Join a room
        ws.send(JSON.stringify({
            type: 'join',
            room: 'general'
        }));
    };
    
    // Receive messages
    ws.onmessage = function(event) {
        const msg = JSON.parse(event.data);
        console.log('Received:', msg);
        
        switch(msg.type) {
            case 'welcome':
                addMessage('Server', 'Welcome! Your ID: ' + msg.client_id, 'system');
                break;
            case 'message':
                addMessage(msg.from, msg.text, 'chat');
                break;
            case 'user_joined':
                addMessage('System', msg.user + ' joined ' + msg.room, 'system');
                break;
            case 'user_left':
                addMessage('System', msg.user + ' left ' + msg.room, 'system');
                break;
            case 'error':
                addMessage('Error', msg.error, 'error');
                break;
        }
    };
    
    // Handle close
    ws.onclose = function(event) {
        console.log('Disconnected:', event.code, event.reason);
        addMessage('System', 'Disconnected from server', 'system');
        
        // Auto-reconnect after 3 seconds
        setTimeout(() => {
            window.location.reload();
        }, 3000);
    };
    
    // Handle errors
    ws.onerror = function(error) {
        console.error('WebSocket error:', error);
    };
    
    // Send message
    function sendMessage() {
        const input = document.getElementById('msg-input');
        const text = input.value.trim();
        
        if (text && ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify({
                type: 'message',
                room: 'general',
                text: text
            }));
            input.value = '';
        }
    }
    
    // Add message to DOM
    function addMessage(from, text, type) {
        const div = document.getElementById('messages');
        const msg = document.createElement('div');
        msg.className = 'msg-' + type;
        msg.innerHTML = '<strong>' + from + '</strong>: ' + text;
        div.appendChild(msg);
        div.scrollTop = div.scrollHeight;
    }
    
    // Enter key to send
    document.getElementById('msg-input').addEventListener('keypress', function(e) {
        if (e.key === 'Enter') sendMessage();
    });
    
    // Heartbeat (ping every 30 seconds)
    setInterval(() => {
        if (ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify({type: 'ping'}));
        }
    }, 30000);
    </script>
</body>
</html>
```

---

## 54.9 Heartbeat Implementation

```lua
-- Server-side heartbeat
local function heartbeat_handler(wb, client_id)
    local last_pong = ngx.time()
    local timeout   = 60  -- seconds
    
    -- ตั้ง timer สำหรับ ping
    local function send_heartbeat()
        while true do
            ngx.sleep(30)  -- ping ทุก 30 วินาที
            
            if ngx.time() - last_pong > timeout then
                ngx.log(ngx.WARN, "Client " .. client_id .. " heartbeat timeout")
                wb:send_close(1001, "Heartbeat timeout")
                break
            end
            
            local ok, err = wb:send_ping(tostring(ngx.time()))
            if not ok then
                ngx.log(ngx.ERR, "Heartbeat ping failed: " .. (err or ""))
                break
            end
        end
    end
    
    -- Update last pong time
    local function on_pong()
        last_pong = ngx.time()
    end
    
    return on_pong
end

-- ใช้งาน
local wb, err = server:new{timeout = 5000}
if not wb then return ngx.exit(444) end

local client_id = ngx.var.remote_addr
local update_pong = heartbeat_handler(wb, client_id)

while true do
    local data, typ, err = wb:recv_frame()
    
    if wb.fatal then break end
    if not data then
        -- Timeout - ส่ง ping
        wb:send_ping()
        goto continue
    end
    
    if typ == "pong" then
        update_pong()  -- อัพเดท last pong time
    elseif typ == "close" then
        wb:send_close()
        break
    elseif typ == "ping" then
        wb:send_pong(data)
    elseif typ == "text" then
        -- process message
    end
    
    ::continue::
end

wb:close()
```

---

## 54.10 Rooms และ Channels

```lua
-- Room Manager
local RoomManager = {}
local shared = ngx.shared.chat_rooms

function RoomManager.create(room_name, options)
    options = options or {}
    local room_data = {
        name       = room_name,
        created_at = ngx.time(),
        max_users  = options.max_users or 100,
        private    = options.private or false,
        password   = options.password,
        clients    = {}
    }
    shared:set("room:" .. room_name, cjson.encode(room_data), 0)
    return room_data
end

function RoomManager.get(room_name)
    local data = shared:get("room:" .. room_name)
    if not data then return nil end
    local ok, room = pcall(cjson.decode, data)
    return ok and room or nil
end

function RoomManager.list()
    local rooms = {}
    local keys  = shared:keys()
    for _, k in ipairs(keys) do
        local name = k:match("^room:(.+)$")
        if name then
            local room = RoomManager.get(name)
            if room then
                table.insert(rooms, {
                    name       = room.name,
                    user_count = #(room.clients or {}),
                    max_users  = room.max_users,
                    created_at = room.created_at
                })
            end
        end
    end
    return rooms
end

function RoomManager.add_client(room_name, client_id)
    local room = RoomManager.get(room_name)
    if not room then
        room = RoomManager.create(room_name)
    end
    
    if #room.clients >= room.max_users then
        return false, "Room is full"
    end
    
    for _, cid in ipairs(room.clients) do
        if cid == client_id then return true end  -- already in room
    end
    
    table.insert(room.clients, client_id)
    shared:set("room:" .. room_name, cjson.encode(room), 0)
    return true
end

function RoomManager.remove_client(room_name, client_id)
    local room = RoomManager.get(room_name)
    if not room then return end
    
    local new_clients = {}
    for _, cid in ipairs(room.clients) do
        if cid ~= client_id then
            table.insert(new_clients, cid)
        end
    end
    room.clients = new_clients
    
    -- Auto-delete empty rooms (except general)
    if #room.clients == 0 and room_name ~= "general" then
        shared:delete("room:" .. room_name)
    else
        shared:set("room:" .. room_name, cjson.encode(room), 0)
    end
end

function RoomManager.get_clients(room_name)
    local room = RoomManager.get(room_name)
    if not room then return {} end
    return room.clients or {}
end

return RoomManager
```

---

## 54.11 Real-time Notifications

```lua
-- Notification Server
-- เมื่อ event เกิดขึ้น (เช่น new comment, new message, etc.)
-- ส่ง notification ไปยัง client ที่เกี่ยวข้อง

local Notifications = {}
local notification_queue = ngx.shared.notification_queue

-- ส่ง notification สำหรับ user คนหนึ่ง
function Notifications.send_to_user(user_id, notification)
    local key = "notif:" .. tostring(user_id) .. ":" .. tostring(ngx.now())
    local data = cjson.encode({
        id        = ngx.md5(key),
        user_id   = user_id,
        type      = notification.type,
        title     = notification.title,
        body      = notification.body,
        data      = notification.data,
        timestamp = ngx.time(),
        read      = false
    })
    
    notification_queue:set(key, data, 300)  -- expire in 5 minutes
    notification_queue:incr("notif_count:" .. tostring(user_id), 1, 0)
end

-- ส่ง notification ไปหลาย users
function Notifications.broadcast(user_ids, notification)
    for _, user_id in ipairs(user_ids) do
        Notifications.send_to_user(user_id, notification)
    end
end

-- WebSocket handler สำหรับ notifications
local function notification_ws_handler()
    local server = require "resty.websocket.server"
    
    local wb, err = server:new{timeout = 60000}
    if not wb then return ngx.exit(444) end
    
    -- Authenticate user
    local token = ngx.var.arg_token
    if not token then
        wb:send_text(cjson.encode({type="error", error="Token required"}))
        wb:send_close(1008, "Authentication required")
        wb:close()
        return
    end
    
    -- Verify token and get user_id
    local user_id = verify_token_get_user_id(token)  -- implementation needed
    if not user_id then
        wb:send_text(cjson.encode({type="error", error="Invalid token"}))
        wb:send_close(1008, "Invalid token")
        wb:close()
        return
    end
    
    -- Send pending notifications
    local function send_pending()
        local prefix = "notif:" .. tostring(user_id) .. ":"
        local keys   = notification_queue:keys()
        local sent   = 0
        
        for _, k in ipairs(keys) do
            if k:sub(1, #prefix) == prefix then
                local data = notification_queue:get(k)
                if data then
                    wb:send_text(data)
                    notification_queue:delete(k)
                    sent = sent + 1
                end
            end
        end
        
        if sent > 0 then
            notification_queue:set("notif_count:" .. tostring(user_id), 0)
        end
    end
    
    -- Send welcome
    wb:send_text(cjson.encode({
        type    = "connected",
        user_id = user_id,
        message = "Notification stream started"
    }))
    
    -- Send any pending notifications
    send_pending()
    
    local last_count = notification_queue:get("notif_count:" .. tostring(user_id)) or 0
    
    while true do
        -- Check for new notifications
        local current_count = notification_queue:get("notif_count:" .. tostring(user_id)) or 0
        if current_count > last_count then
            send_pending()
            last_count = 0  -- reset after sending
        end
        
        local data, typ, err = wb:recv_frame()
        
        if wb.fatal then break end
        if not data then goto continue end
        
        if typ == "close" then
            wb:send_close()
            break
        elseif typ == "ping" then
            wb:send_pong(data)
        elseif typ == "text" then
            local ok, msg = pcall(cjson.decode, data)
            if ok and msg.type == "mark_read" and msg.notification_id then
                -- Mark notification as read
                notification_queue:delete("notif:" .. tostring(user_id) .. ":" .. msg.notification_id)
            end
        end
        
        ::continue::
        ngx.sleep(0.5)  -- Poll every 500ms
    end
    
    wb:close()
end
```

---

## 54.12 Error Handling ใน WebSocket

```lua
-- Comprehensive error handling
local function safe_ws_handler()
    local server = require "resty.websocket.server"
    
    local wb, err = server:new{
        timeout         = 30000,
        max_payload_len = 65535
    }
    
    if not wb then
        ngx.log(ngx.ERR, "WebSocket upgrade failed: " .. (err or "unknown"))
        
        -- ส่ง error response ก่อน close
        ngx.status = 400
        ngx.header["Content-Type"] = "application/json"
        ngx.say('{"error": "WebSocket upgrade failed: ' .. (err or "unknown") .. '"}')
        return
    end
    
    -- Wrap main loop ด้วย pcall
    local ok, err = pcall(function()
        while true do
            local data, typ, err = wb:recv_frame()
            
            -- Connection lost
            if wb.fatal then
                ngx.log(ngx.ERR, "WebSocket fatal error: " .. (err or ""))
                break
            end
            
            -- Timeout (recv_frame returns nil data on timeout, not fatal)
            if not data then
                -- Optional: ส่ง ping เพื่อตรวจสอบ
                local ok, ping_err = wb:send_ping()
                if not ok then
                    ngx.log(ngx.ERR, "Cannot ping: " .. (ping_err or ""))
                    break
                end
                goto continue
            end
            
            -- Process based on type
            if typ == "close" then
                local code, msg = wb:recv_close()
                ngx.log(ngx.INFO, "Close received: code=" .. (code or "?") .. " msg=" .. (msg or ""))
                
                -- Echo close frame back
                local ok, close_err = wb:send_close(code or 1000)
                if not ok then
                    ngx.log(ngx.ERR, "Cannot send close: " .. (close_err or ""))
                end
                break
                
            elseif typ == "text" then
                -- Validate message size
                if #data > 16384 then  -- 16KB
                    wb:send_text('{"error": "Message too large"}')
                    goto continue
                end
                
                -- Process with error catching
                local msg_ok, msg_err = pcall(function()
                    local ok, msg = pcall(cjson.decode, data)
                    if not ok then
                        wb:send_text('{"error": "Invalid JSON: ' .. tostring(msg) .. '"}')
                        return
                    end
                    
                    -- Process message...
                    wb:send_text('{"type": "ack", "id": "' .. (msg.id or "") .. '"}')
                end)
                
                if not msg_ok then
                    ngx.log(ngx.ERR, "Message processing error: " .. tostring(msg_err))
                    pcall(function()
                        wb:send_text('{"error": "Internal processing error"}')
                    end)
                end
            end
            
            ::continue::
        end
    end)
    
    if not ok then
        ngx.log(ngx.ERR, "WebSocket handler error: " .. tostring(err))
    end
    
    -- Always close
    pcall(function() wb:close() end)
end
```

---

## 54.13 Load Balancing WebSockets

```nginx
# nginx.conf สำหรับ WebSocket load balancing

upstream ws_backends {
    # Sticky session สำหรับ WebSocket
    ip_hash;  # หรือใช้ consistent_hash
    
    server ws1:8080;
    server ws2:8080;
    server ws3:8080;
    
    keepalive 100;
}

server {
    listen 80;
    
    location /ws {
        proxy_pass http://ws_backends;
        
        # WebSocket upgrade headers
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Timeouts (WebSocket connections ยาวนาน)
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;
        proxy_connect_timeout 10s;
        
        # Buffer settings
        proxy_buffering off;
    }
}
```

---

## 54.14 WebSocket Client Library

```lua
-- ws_client.lua - WebSocket Client สำหรับ Lua
-- ใช้ใน tests หรือ server-to-server communication

local socket = ngx.socket.tcp

local Client = {}

function Client.new(url, opts)
    opts = opts or {}
    
    -- Parse URL
    local host, port, path = url:match("^wss?://([^:/]+):?(%d*)(.*)")
    port = tonumber(port) or 80
    path = path ~= "" and path or "/"
    
    local sock = socket()
    sock:settimeout(opts.timeout or 5000)
    
    -- Connect
    local ok, err = sock:connect(host, port)
    if not ok then
        return nil, "Connect failed: " .. err
    end
    
    -- WebSocket handshake
    local key = ngx.encode_base64(
        ngx.md5(tostring(ngx.now()) .. tostring(math.random(1000000)))
    )
    
    local handshake = 
        "GET " .. path .. " HTTP/1.1\r\n" ..
        "Host: " .. host .. ":" .. port .. "\r\n" ..
        "Upgrade: websocket\r\n" ..
        "Connection: Upgrade\r\n" ..
        "Sec-WebSocket-Key: " .. key .. "\r\n" ..
        "Sec-WebSocket-Version: 13\r\n" ..
        "\r\n"
    
    local bytes, err = sock:send(handshake)
    if not bytes then
        return nil, "Handshake send failed: " .. err
    end
    
    -- Read response
    local line, err = sock:receive("*l")
    if not line then
        return nil, "No response: " .. (err or "")
    end
    
    if not line:match("^HTTP/1%.1 101") then
        return nil, "Unexpected response: " .. line
    end
    
    -- Skip headers
    while true do
        local header = sock:receive("*l")
        if not header or header == "" then break end
    end
    
    -- Return client object
    local client = {
        sock = sock,
        
        send_text = function(self, data)
            return self:_send_frame(0x81, data)  -- FIN + text opcode
        end,
        
        send_binary = function(self, data)
            return self:_send_frame(0x82, data)  -- FIN + binary opcode
        end,
        
        send_ping = function(self, data)
            return self:_send_frame(0x89, data or "")  -- ping
        end,
        
        recv = function(self)
            -- Read frame header
            local header, err = self.sock:receive(2)
            if not header then return nil, nil, err end
            
            local byte1 = header:byte(1)
            local byte2 = header:byte(2)
            local opcode = byte1 % 16
            local masked  = byte2 >= 128
            local length  = byte2 % 128
            
            -- Extended length
            if length == 126 then
                local ext, err = self.sock:receive(2)
                if not ext then return nil, nil, err end
                length = ext:byte(1) * 256 + ext:byte(2)
            elseif length == 127 then
                local ext, err = self.sock:receive(8)
                if not ext then return nil, nil, err end
                -- Simplified: use last 4 bytes
                length = ext:byte(5)*16777216 + ext:byte(6)*65536 + 
                         ext:byte(7)*256 + ext:byte(8)
            end
            
            -- Read payload
            local payload, err = self.sock:receive(length)
            if not payload then return nil, nil, err end
            
            local types = {[0x1]="text", [0x2]="binary", [0x8]="close", 
                          [0x9]="ping", [0xA]="pong"}
            local typ = types[opcode] or "unknown"
            
            return payload, typ, nil
        end,
        
        close = function(self)
            self:_send_frame(0x88, "\x03\xe8")  -- close with code 1000
            self.sock:close()
        end,
        
        _send_frame = function(self, opcode, data)
            local len = #data
            local frame
            
            if len <= 125 then
                frame = string.char(opcode, len + 128)
            elseif len <= 65535 then
                frame = string.char(opcode, 126 + 128, 
                    math.floor(len/256), len%256)
            else
                -- Large frame (simplified)
                frame = string.char(opcode, 127 + 128, 
                    0,0,0,0,
                    math.floor(len/16777216)%256,
                    math.floor(len/65536)%256,
                    math.floor(len/256)%256,
                    len%256)
            end
            
            -- Masking key (required for client->server)
            local mask = {math.random(256)-1, math.random(256)-1, 
                         math.random(256)-1, math.random(256)-1}
            frame = frame .. string.char(unpack(mask))
            
            -- Mask payload
            local masked_data = {}
            for i = 1, len do
                local b = data:byte(i)
                masked_data[i] = string.char(b ~ mask[((i-1)%4)+1])
            end
            
            return self.sock:send(frame .. table.concat(masked_data))
        end
    }
    
    return client
end

return Client
```

---

## 54.15 WebSocket Stats Monitor

```nginx
location /ws/stats {
    content_by_lua_block {
        local clients  = ngx.shared.ws_clients
        local cjson    = require "cjson"
        
        local stats = {
            total_connections = clients:get("total_clients") or 0,
            worker_id = ngx.worker.id(),
            uptime    = ngx.time() - (clients:get("start_time") or ngx.time()),
            timestamp = ngx.time()
        }
        
        ngx.header["Content-Type"] = "application/json"
        ngx.say(cjson.encode(stats))
    }
}
```

---

## สรุป

ในบทนี้เราเรียนรู้ WebSocket กับ Lua ครบถ้วน:

1. **WebSocket Protocol** - ทำงานอย่างไรเทียบกับ HTTP
2. **OpenResty WebSocket** - `resty.websocket.server`
3. **Message Types** - text, binary, ping, pong, close
4. **Connection Lifecycle** - open, message loop, close
5. **Error Handling** - ป้องกัน fatal errors
6. **Broadcasting** - ส่ง messages ไปหลาย clients
7. **Rooms/Channels** - จัดกลุ่ม clients
8. **Heartbeat** - ตรวจสอบ connection ที่ยังมีชีวิต
9. **Real-time Chat** - complete chat application
10. **Real-time Notifications** - push notification system
11. **Load Balancing** - sticky sessions สำหรับ WebSocket
12. **Client Library** - WebSocket client สำหรับ Lua

WebSocket เป็นเครื่องมือที่ทรงพลังสำหรับ real-time applications!
