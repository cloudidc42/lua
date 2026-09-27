# บทที่ 97: Real Project 1 - Chat Application

## บทนำ

บทนี้จะสร้าง Real-time Chat Application ที่สมบูรณ์ด้วย Lua + OpenResty + WebSocket + Redis โดยใช้ความรู้จากทุก Part ที่ผ่านมา

---

## 97.1 Architecture Overview

```
Architecture:
┌──────────────────────────────────────────────────────┐
│                     Clients (Browser)                 │
│   ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐            │
│   │ User1│  │ User2│  │ User3│  │ User4│            │
│   └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘            │
└──────┼─────────┼─────────┼─────────┼──────────────────┘
       │WebSocket│         │         │
┌──────▼─────────▼─────────▼─────────▼──────────────────┐
│               OpenResty / Nginx                         │
│   ┌────────────────────────────────────────────┐       │
│   │        Lua WebSocket Handler               │       │
│   │  - Connection management                  │       │
│   │  - Message routing                        │       │
│   │  - Authentication (JWT)                   │       │
│   │  - Rate limiting                          │       │
│   └────────────────────────┬───────────────────┘       │
└───────────────────────────┼────────────────────────────┘
                             │
┌────────────────────────────▼───────────────────────────┐
│                    Redis                                │
│   - Pub/Sub (real-time messaging)                       │
│   - Session storage                                     │
│   - Message history                                     │
│   - Online users list                                   │
│   - Room data                                           │
└────────────────────────────────────────────────────────┘
```

---

## 97.2 Project Structure

```
chat-app/
├── nginx.conf              - Nginx configuration
├── app/
│   ├── main.lua            - Application entry point
│   ├── auth.lua            - Authentication module
│   ├── chat.lua            - Chat business logic
│   ├── ws_handler.lua      - WebSocket handler
│   ├── redis_client.lua    - Redis connection pool
│   ├── rate_limiter.lua    - Rate limiting
│   └── utils.lua           - Utilities
├── static/
│   └── index.html          - Chat client UI
└── test/
    └── test_chat.lua       - Tests
```

---

## 97.3 nginx.conf

```nginx
# nginx.conf
worker_processes auto;
events { worker_connections 10000; }

http {
    lua_package_path "/path/to/app/?.lua;;";
    
    # Shared memory for rate limiting and presence
    lua_shared_dict chat_limits 10m;
    lua_shared_dict online_users 5m;
    
    # Redis connection pool
    lua_shared_dict redis_pool 5m;
    
    init_by_lua_block {
        require("app.main").init()
    }
    
    server {
        listen 8080;
        
        # WebSocket endpoint
        location /ws {
            lua_socket_log_errors off;
            
            access_by_lua_block {
                require("app.auth").check_token()
            }
            
            content_by_lua_block {
                require("app.ws_handler").handle()
            }
        }
        
        # REST API endpoints
        location /api/login {
            content_by_lua_block {
                require("app.auth").login()
            }
        }
        
        location /api/history {
            access_by_lua_block {
                require("app.auth").check_token()
            }
            content_by_lua_block {
                require("app.chat").get_history()
            }
        }
        
        location /api/rooms {
            access_by_lua_block {
                require("app.auth").check_token()
            }
            content_by_lua_block {
                require("app.chat").list_rooms()
            }
        }
        
        # Static files
        location / {
            root /path/to/static;
            index index.html;
        }
    }
}
```

---

## 97.4 auth.lua - Authentication Module

```lua
-- app/auth.lua
local jwt = require("resty.jwt")
local redis = require("app.redis_client")

local M = {}

local JWT_SECRET = os.getenv("JWT_SECRET") or "dev-secret-change-in-prod"
local JWT_EXPIRY = 86400  -- 24 hours

function M.login()
    ngx.req.read_body()
    local body = ngx.req.get_body_data()
    
    if not body then
        ngx.status = 400
        ngx.say('{"error":"No body"}')
        return
    end
    
    local ok, data = pcall(require("cjson").decode, body)
    if not ok then
        ngx.status = 400
        ngx.say('{"error":"Invalid JSON"}')
        return
    end
    
    local username = data.username
    local password = data.password
    
    if not username or not password then
        ngx.status = 400
        ngx.say('{"error":"Missing credentials"}')
        return
    end
    
    -- Simple auth (in production: check database)
    -- Demo: any password works, username must be 3-20 chars
    if #username < 3 or #username > 20 then
        ngx.status = 401
        ngx.say('{"error":"Invalid username"}')
        return
    end
    
    -- Create JWT
    local payload = {
        sub = username,
        iat = ngx.time(),
        exp = ngx.time() + JWT_EXPIRY
    }
    
    local token = jwt:sign(JWT_SECRET, {
        header = { typ = "JWT", alg = "HS256" },
        payload = payload
    })
    
    ngx.header["Content-Type"] = "application/json"
    ngx.say(require("cjson").encode({
        token = token,
        username = username,
        expires_in = JWT_EXPIRY
    }))
end

function M.check_token()
    -- Check Authorization header
    local auth = ngx.req.get_headers()["Authorization"]
    if not auth then
        -- Check query param for WebSocket
        auth = ngx.var.arg_token
        if not auth then
            ngx.status = 401
            ngx.say('{"error":"Unauthorized"}')
            return ngx.exit(401)
        end
        auth = "Bearer " .. auth
    end
    
    local token = auth:match("Bearer (.+)")
    if not token then
        ngx.status = 401
        ngx.say('{"error":"Invalid auth format"}')
        return ngx.exit(401)
    end
    
    local jwt_obj = jwt:verify(JWT_SECRET, token)
    
    if not jwt_obj.verified then
        ngx.status = 401
        ngx.say('{"error":"Invalid token: ' .. (jwt_obj.reason or "unknown") .. '"}')
        return ngx.exit(401)
    end
    
    -- Set user in context
    ngx.ctx.user = jwt_obj.payload.sub
    ngx.ctx.user_id = jwt_obj.payload.sub
end

function M.get_current_user()
    return ngx.ctx.user
end

return M
```

---

## 97.5 redis_client.lua - Redis Connection Pool

```lua
-- app/redis_client.lua
local redis = require("resty.redis")

local M = {}

local REDIS_HOST = os.getenv("REDIS_HOST") or "127.0.0.1"
local REDIS_PORT = tonumber(os.getenv("REDIS_PORT")) or 6379
local REDIS_TIMEOUT = 1000  -- 1 second
local POOL_SIZE = 100
local POOL_IDLE = 10000  -- 10 seconds

function M.get()
    local red = redis:new()
    red:set_timeouts(REDIS_TIMEOUT, REDIS_TIMEOUT, REDIS_TIMEOUT)
    
    local ok, err = red:connect(REDIS_HOST, REDIS_PORT)
    if not ok then
        return nil, "Redis connect failed: " .. err
    end
    
    return red
end

function M.release(red)
    if not red then return end
    red:set_keepalive(POOL_IDLE, POOL_SIZE)
end

function M.with(fn)
    local red, err = M.get()
    if not red then return nil, err end
    
    local ok, result = pcall(fn, red)
    M.release(red)
    
    if not ok then
        return nil, result
    end
    return result
end

-- Convenience methods
function M.publish(channel, message)
    return M.with(function(red)
        return red:publish(channel, message)
    end)
end

function M.get_key(key)
    return M.with(function(red)
        local val = red:get(key)
        if val == ngx.null then return nil end
        return val
    end)
end

function M.set_key(key, value, expiry)
    return M.with(function(red)
        if expiry then
            return red:setex(key, expiry, value)
        else
            return red:set(key, value)
        end
    end)
end

function M.lpush(key, value)
    return M.with(function(red)
        return red:lpush(key, value)
    end)
end

function M.lrange(key, start, stop)
    return M.with(function(red)
        return red:lrange(key, start, stop)
    end)
end

function M.sadd(key, member)
    return M.with(function(red)
        return red:sadd(key, member)
    end)
end

function M.srem(key, member)
    return M.with(function(red)
        return red:srem(key, member)
    end)
end

function M.smembers(key)
    return M.with(function(red)
        return red:smembers(key)
    end)
end

return M
```

---

## 97.6 ws_handler.lua - WebSocket Handler

```lua
-- app/ws_handler.lua
local ws = require("resty.websocket.server")
local redis = require("app.redis_client")
local chat = require("app.chat")
local json = require("cjson")
local auth = require("app.auth")

local M = {}

-- Connected clients per room: { room_id -> { user -> ws_obj } }
-- Note: This is per-worker. For multi-worker, use Redis Pub/Sub
local connections = {}

function M.handle()
    local user = auth.get_current_user()
    if not user then
        ngx.status = 401
        return
    end
    
    -- Upgrade to WebSocket
    local wb, err = ws:new({
        timeout = 5000,
        max_payload_len = 65535
    })
    
    if not wb then
        ngx.log(ngx.ERR, "WebSocket upgrade failed: ", err)
        return ngx.exit(444)
    end
    
    -- Get room from query params
    local room = ngx.var.arg_room or "general"
    
    -- Register connection
    if not connections[room] then
        connections[room] = {}
    end
    connections[room][user] = wb
    
    -- Notify room of join
    chat.broadcast_to_room(connections, room, {
        type = "system",
        message = user .. " joined the room",
        room = room,
        timestamp = ngx.time()
    }, user)
    
    -- Update online users
    redis.sadd("room:" .. room .. ":users", user)
    
    ngx.log(ngx.INFO, "User connected: ", user, " to room: ", room)
    
    -- Main message loop
    while true do
        local data, typ, err = wb:recv_frame()
        
        if wb.fatal then
            ngx.log(ngx.ERR, "Fatal WebSocket error: ", err)
            break
        end
        
        if not data then
            ngx.log(ngx.INFO, "WebSocket closed: ", err)
            break
        end
        
        if typ == "close" then
            wb:send_close()
            break
        elseif typ == "ping" then
            wb:send_pong()
        elseif typ == "text" then
            local ok, msg = pcall(json.decode, data)
            if not ok then
                wb:send_text(json.encode({
                    type = "error",
                    message = "Invalid JSON"
                }))
            else
                M.handle_message(wb, connections, user, room, msg)
            end
        end
    end
    
    -- Cleanup on disconnect
    if connections[room] then
        connections[room][user] = nil
    end
    redis.srem("room:" .. room .. ":users", user)
    
    -- Notify room of leave
    chat.broadcast_to_room(connections, room, {
        type = "system",
        message = user .. " left the room",
        room = room,
        timestamp = ngx.time()
    }, nil)  -- nil = broadcast to all including sender (who is gone)
    
    ngx.log(ngx.INFO, "User disconnected: ", user)
end

function M.handle_message(wb, connections, user, room, msg)
    local msg_type = msg.type
    
    if msg_type == "message" then
        -- Validate
        if not msg.content or #msg.content == 0 then
            return wb:send_text(json.encode({
                type = "error", message = "Empty message"
            }))
        end
        
        if #msg.content > 2000 then
            return wb:send_text(json.encode({
                type = "error", message = "Message too long (max 2000 chars)"
            }))
        end
        
        -- Rate limiting
        local limit_key = "ratelimit:" .. user
        local count = tonumber(redis.get_key(limit_key) or "0")
        if count and count > 30 then  -- 30 messages per minute
            return wb:send_text(json.encode({
                type = "error", message = "Rate limit exceeded"
            }))
        end
        
        redis.with(function(red)
            red:incr(limit_key)
            red:expire(limit_key, 60)
        end)
        
        -- Build message
        local chat_msg = {
            type = "message",
            id = ngx.time() .. "_" .. user,
            user = user,
            content = msg.content,
            room = room,
            timestamp = ngx.time()
        }
        
        -- Store in history (last 100 messages per room)
        local msg_json = json.encode(chat_msg)
        redis.with(function(red)
            red:lpush("room:" .. room .. ":history", msg_json)
            red:ltrim("room:" .. room .. ":history", 0, 99)
        end)
        
        -- Broadcast to room
        chat.broadcast_to_room(connections, room, chat_msg, nil)
        
    elseif msg_type == "join_room" then
        local new_room = msg.room
        if not new_room then return end
        
        -- Leave current room
        if connections[room] then
            connections[room][user] = nil
            redis.srem("room:" .. room .. ":users", user)
            chat.broadcast_to_room(connections, room, {
                type = "system",
                message = user .. " left",
                room = room,
                timestamp = ngx.time()
            }, nil)
        end
        
        -- Join new room
        room = new_room  -- Update room variable
        if not connections[room] then
            connections[room] = {}
        end
        connections[room][user] = wb
        redis.sadd("room:" .. room .. ":users", user)
        
        chat.broadcast_to_room(connections, room, {
            type = "system",
            message = user .. " joined",
            room = room,
            timestamp = ngx.time()
        }, nil)
        
        wb:send_text(json.encode({
            type = "room_joined",
            room = room
        }))
        
    elseif msg_type == "typing" then
        -- Broadcast typing indicator (don't store)
        chat.broadcast_to_room(connections, room, {
            type = "typing",
            user = user,
            room = room
        }, user)  -- Don't send back to self
        
    elseif msg_type == "ping" then
        wb:send_text(json.encode({ type = "pong" }))
    end
end

return M
```

---

## 97.7 chat.lua - Business Logic

```lua
-- app/chat.lua
local redis = require("app.redis_client")
local json = require("cjson")

local M = {}

-- Broadcast message to all users in a room
function M.broadcast_to_room(connections, room, msg, exclude_user)
    local room_conns = connections[room]
    if not room_conns then return end
    
    local msg_str = json.encode(msg)
    local failed_users = {}
    
    for user, wb in pairs(room_conns) do
        if user ~= exclude_user then
            local bytes, err = wb:send_text(msg_str)
            if not bytes then
                table.insert(failed_users, user)
            end
        end
    end
    
    -- Clean up failed connections
    for _, user in ipairs(failed_users) do
        room_conns[user] = nil
    end
end

function M.get_history()
    local room = ngx.var.arg_room or "general"
    local limit = tonumber(ngx.var.arg_limit) or 50
    
    limit = math.min(limit, 100)
    
    local messages = redis.lrange("room:" .. room .. ":history", 0, limit - 1)
    
    local result = {}
    if messages then
        for _, msg_str in ipairs(messages) do
            local ok, msg = pcall(json.decode, msg_str)
            if ok then
                table.insert(result, msg)
            end
        end
    end
    
    -- Messages are stored newest-first, reverse for display
    local reversed = {}
    for i = #result, 1, -1 do
        table.insert(reversed, result[i])
    end
    
    ngx.header["Content-Type"] = "application/json"
    ngx.say(json.encode({ messages = reversed, room = room }))
end

function M.list_rooms()
    -- Hard-coded rooms (in production: store in Redis)
    local rooms = {
        { id = "general",  name = "General",  description = "General discussion" },
        { id = "tech",     name = "Tech",     description = "Technology topics" },
        { id = "random",   name = "Random",   description = "Off-topic" },
        { id = "lua",      name = "Lua",      description = "Lua programming" }
    }
    
    -- Add online count from Redis
    for _, room in ipairs(rooms) do
        local members = redis.smembers("room:" .. room.id .. ":users")
        room.online = members and #members or 0
    end
    
    ngx.header["Content-Type"] = "application/json"
    ngx.say(json.encode({ rooms = rooms }))
end

return M
```

---

## 97.8 static/index.html - Chat Client

```html
<!DOCTYPE html>
<html>
<head>
    <title>Lua Chat</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body { font-family: Arial, sans-serif; height: 100vh; display: flex; flex-direction: column; }
        #login { display: flex; flex-direction: column; align-items: center; justify-content: center; height: 100vh; gap: 10px; }
        #chat-app { display: none; flex-direction: column; height: 100vh; }
        #toolbar { background: #2c3e50; color: white; padding: 10px 20px; display: flex; gap: 10px; align-items: center; }
        #messages { flex: 1; overflow-y: auto; padding: 10px; background: #f5f5f5; }
        .msg { margin: 5px 0; padding: 8px; background: white; border-radius: 5px; box-shadow: 0 1px 2px rgba(0,0,0,0.1); }
        .msg .user { font-weight: bold; color: #2c3e50; }
        .msg .time { float: right; font-size: 0.8em; color: #999; }
        .msg.system { background: #e8f4fd; color: #666; font-style: italic; }
        #input-area { padding: 10px; background: white; border-top: 1px solid #ddd; display: flex; gap: 10px; }
        #msg-input { flex: 1; padding: 10px; border: 1px solid #ddd; border-radius: 5px; }
        button { padding: 10px 20px; background: #3498db; color: white; border: none; border-radius: 5px; cursor: pointer; }
        button:hover { background: #2980b9; }
        input { padding: 10px; border: 1px solid #ddd; border-radius: 5px; width: 300px; }
        #status { font-size: 0.8em; }
        .online { color: #2ecc71; } .offline { color: #e74c3c; }
        #rooms { background: #34495e; padding: 10px; width: 150px; overflow-y: auto; }
        .room-btn { display: block; width: 100%; padding: 8px; margin: 2px 0; background: transparent; color: #ecf0f1; text-align: left; border: none; cursor: pointer; border-radius: 3px; }
        .room-btn:hover, .room-btn.active { background: #3d5a7a; }
        #main-area { display: flex; flex: 1; overflow: hidden; }
        #typing-indicator { padding: 5px 10px; color: #999; font-size: 0.85em; min-height: 20px; }
    </style>
</head>
<body>
    <div id="login">
        <h2>Lua Chat</h2>
        <input id="username-input" type="text" placeholder="Username" />
        <input id="password-input" type="password" placeholder="Password (any)" />
        <button onclick="login()">Login</button>
        <p id="login-error" style="color:red"></p>
    </div>
    
    <div id="chat-app">
        <div id="toolbar">
            <strong>Lua Chat</strong>
            <span id="room-name">#general</span>
            <span style="flex:1"></span>
            <span id="user-display"></span>
            <span id="status" class="offline">● Disconnected</span>
        </div>
        <div id="main-area">
            <div id="rooms">
                <div style="color:#aaa;font-size:0.8em;padding:5px">ROOMS</div>
            </div>
            <div style="flex:1;display:flex;flex-direction:column">
                <div id="messages"></div>
                <div id="typing-indicator"></div>
                <div id="input-area">
                    <input id="msg-input" placeholder="Type a message..." onkeypress="handleKey(event)" />
                    <button onclick="sendMessage()">Send</button>
                </div>
            </div>
        </div>
    </div>

<script>
let ws = null, token = null, username = null, currentRoom = 'general';
let typingTimeout = null;

async function login() {
    const user = document.getElementById('username-input').value.trim();
    const pass = document.getElementById('password-input').value;
    
    if (!user) return;
    
    const resp = await fetch('/api/login', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({username: user, password: pass})
    });
    
    const data = await resp.json();
    if (data.error) {
        document.getElementById('login-error').textContent = data.error;
        return;
    }
    
    token = data.token;
    username = data.username;
    
    document.getElementById('login').style.display = 'none';
    document.getElementById('chat-app').style.display = 'flex';
    document.getElementById('user-display').textContent = username;
    
    loadRooms();
    connectWS('general');
}

async function loadRooms() {
    const resp = await fetch('/api/rooms', {headers: {'Authorization': 'Bearer ' + token}});
    const data = await resp.json();
    
    const roomsEl = document.getElementById('rooms');
    roomsEl.innerHTML = '<div style="color:#aaa;font-size:0.8em;padding:5px">ROOMS</div>';
    
    for (const room of data.rooms) {
        const btn = document.createElement('button');
        btn.className = 'room-btn' + (room.id === currentRoom ? ' active' : '');
        btn.textContent = '#' + room.id + ' (' + room.online + ')';
        btn.onclick = () => joinRoom(room.id);
        roomsEl.appendChild(btn);
    }
}

function connectWS(room) {
    if (ws) ws.close();
    
    currentRoom = room;
    ws = new WebSocket(`ws://${location.host}/ws?token=${token}&room=${room}`);
    
    ws.onopen = () => {
        document.getElementById('status').textContent = '● Connected';
        document.getElementById('status').className = 'online';
        loadHistory(room);
    };
    
    ws.onclose = () => {
        document.getElementById('status').textContent = '● Disconnected';
        document.getElementById('status').className = 'offline';
    };
    
    ws.onmessage = (e) => {
        const msg = JSON.parse(e.data);
        handleMessage(msg);
    };
    
    // Heartbeat
    setInterval(() => {
        if (ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify({type: 'ping'}));
        }
    }, 30000);
}

async function loadHistory(room) {
    const resp = await fetch(`/api/history?room=${room}&limit=50`, {
        headers: {'Authorization': 'Bearer ' + token}
    });
    const data = await resp.json();
    
    document.getElementById('messages').innerHTML = '';
    for (const msg of data.messages) {
        appendMessage(msg);
    }
    scrollToBottom();
}

function handleMessage(msg) {
    if (msg.type === 'message') {
        appendMessage(msg);
        scrollToBottom();
    } else if (msg.type === 'system') {
        appendSystemMessage(msg.message);
    } else if (msg.type === 'typing') {
        if (msg.user !== username) {
            showTyping(msg.user);
        }
    } else if (msg.type === 'room_joined') {
        document.getElementById('room-name').textContent = '#' + msg.room;
        loadHistory(msg.room);
    }
}

function appendMessage(msg) {
    const el = document.createElement('div');
    el.className = 'msg';
    const time = new Date(msg.timestamp * 1000).toLocaleTimeString();
    el.innerHTML = `<span class="user">${escapeHtml(msg.user)}</span> <span class="time">${time}</span><br>${escapeHtml(msg.content)}`;
    document.getElementById('messages').appendChild(el);
}

function appendSystemMessage(text) {
    const el = document.createElement('div');
    el.className = 'msg system';
    el.textContent = text;
    document.getElementById('messages').appendChild(el);
    scrollToBottom();
}

function showTyping(user) {
    document.getElementById('typing-indicator').textContent = user + ' is typing...';
    clearTimeout(typingTimeout);
    typingTimeout = setTimeout(() => {
        document.getElementById('typing-indicator').textContent = '';
    }, 3000);
}

function sendMessage() {
    const input = document.getElementById('msg-input');
    const content = input.value.trim();
    if (!content || !ws) return;
    
    ws.send(JSON.stringify({type: 'message', content: content}));
    input.value = '';
}

function joinRoom(room) {
    if (room === currentRoom) return;
    ws.send(JSON.stringify({type: 'join_room', room: room}));
    currentRoom = room;
    
    document.querySelectorAll('.room-btn').forEach(btn => {
        btn.classList.toggle('active', btn.textContent.startsWith('#' + room));
    });
}

function handleKey(e) {
    if (e.key === 'Enter') {
        sendMessage();
    } else {
        // Send typing indicator
        if (ws && ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify({type: 'typing'}));
        }
    }
}

function scrollToBottom() {
    const el = document.getElementById('messages');
    el.scrollTop = el.scrollHeight;
}

function escapeHtml(str) {
    return str.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
}
</script>
</body>
</html>
```

---

## 97.9 test/test_chat.lua - Testing

```lua
-- test/test_chat.lua
local describe = require("busted").describe
local it = require("busted").it
local assert = require("busted").assert

-- Mock modules
local mock_redis = {
    data = {},
    sets = {},
    
    get = function(self, key)
        return self.data[key]
    end,
    set = function(self, key, value)
        self.data[key] = value
        return "OK"
    end,
    sadd = function(self, key, member)
        self.sets[key] = self.sets[key] or {}
        self.sets[key][member] = true
        return 1
    end,
    srem = function(self, key, member)
        if self.sets[key] then
            self.sets[key][member] = nil
        end
        return 1
    end,
    smembers = function(self, key)
        local result = {}
        if self.sets[key] then
            for m in pairs(self.sets[key]) do
                table.insert(result, m)
            end
        end
        return result
    end
}

describe("Chat Application", function()
    describe("Authentication", function()
        it("should reject empty username", function()
            local auth = require("app.auth")
            -- Test with mock ngx
            local result = auth.validate_username("")
            assert.is_false(result)
        end)
        
        it("should accept valid username", function()
            local auth = require("app.auth")
            local result = auth.validate_username("alice")
            assert.is_true(result)
        end)
        
        it("should reject username too short", function()
            local auth = require("app.auth")
            local result = auth.validate_username("ab")
            assert.is_false(result)
        end)
    end)
    
    describe("Message Validation", function()
        it("should reject empty message", function()
            local chat = require("app.chat")
            local ok, err = chat.validate_message("")
            assert.is_false(ok)
            assert.truthy(err)
        end)
        
        it("should reject too long message", function()
            local chat = require("app.chat")
            local long_msg = string.rep("x", 2001)
            local ok, err = chat.validate_message(long_msg)
            assert.is_false(ok)
        end)
        
        it("should accept valid message", function()
            local chat = require("app.chat")
            local ok = chat.validate_message("Hello World")
            assert.is_true(ok)
        end)
    end)
    
    describe("Room Operations", function()
        it("should track online users", function()
            mock_redis:sadd("room:general:users", "alice")
            mock_redis:sadd("room:general:users", "bob")
            
            local members = mock_redis:smembers("room:general:users")
            assert.equals(2, #members)
        end)
        
        it("should remove user on disconnect", function()
            mock_redis:sadd("room:general:users", "alice")
            mock_redis:srem("room:general:users", "alice")
            
            local members = mock_redis:smembers("room:general:users")
            assert.equals(0, #members)
        end)
    end)
end)
```

---

## แบบฝึกหัด

1. **Private Messages**: เพิ่ม direct message ระหว่าง user 2 คน
2. **File Sharing**: อนุญาตให้แชร์ไฟล์ image
3. **Message Reactions**: เพิ่ม emoji reactions ให้กับ messages
4. **Message Search**: สร้าง full-text search สำหรับ history
5. **Bot**: สร้าง simple chatbot ที่ตอบสนองต่อ commands

---

*ต่อไป: [Part 98 - Real Project: Game Server](part-98.md)*
