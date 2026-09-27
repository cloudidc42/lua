# บทที่ 56: Authentication & Authorization

## บทนำ

Authentication (การยืนยันตัวตน) และ Authorization (การอนุญาต) เป็นสองแนวคิดสำคัญในการรักษาความปลอดภัยของระบบ

- **Authentication**: "คุณเป็นใคร?" - การพิสูจน์ตัวตน
- **Authorization**: "คุณทำอะไรได้บ้าง?" - การกำหนดสิทธิ์

---

## ตัวอย่างที่ 1: ความแตกต่างระหว่าง Authentication และ Authorization

```lua
-- Authentication vs Authorization concepts
local auth_concepts = {}

-- Authentication: ยืนยันว่าผู้ใช้เป็นใคร
function auth_concepts.authenticate(username, password)
    local users = {
        { username = "alice", password_hash = "hash_alice_123", id = 1 },
        { username = "bob",   password_hash = "hash_bob_456",   id = 2 },
    }
    for _, user in ipairs(users) do
        if user.username == username then
            -- ในระบบจริงจะใช้ bcrypt compare
            local computed_hash = "hash_" .. username .. "_" .. password
            if computed_hash == user.password_hash then
                return { success = true, user_id = user.id, username = user.username }
            end
        end
    end
    return { success = false, error = "Invalid credentials" }
end

-- Authorization: ตรวจสอบว่าผู้ใช้มีสิทธิ์ทำสิ่งนั้นไหม
function auth_concepts.authorize(user_id, resource, action)
    local permissions = {
        [1] = { ["posts"] = {"read","write","delete"}, ["users"] = {"read"} },
        [2] = { ["posts"] = {"read"}, ["users"] = {} },
    }
    local user_perms = permissions[user_id]
    if not user_perms then return false end
    local resource_perms = user_perms[resource]
    if not resource_perms then return false end
    for _, perm in ipairs(resource_perms) do
        if perm == action then return true end
    end
    return false
end

-- ทดสอบ
local auth_result = auth_concepts.authenticate("alice", "123")
print("Auth result:", auth_result.success, auth_result.username)

local can_delete = auth_concepts.authorize(1, "posts", "delete")
local bob_delete = auth_concepts.authorize(2, "posts", "delete")
print("Alice can delete posts:", can_delete)
print("Bob can delete posts:", bob_delete)
```

---

## ตัวอย่างที่ 2: Session-Based Authentication

```lua
-- Session-based authentication system
local session_auth = {}
local sessions = {}
local session_counter = 0

-- สร้าง session ID
local function generate_session_id()
    session_counter = session_counter + 1
    local timestamp = os.time()
    return string.format("sess_%d_%d_%s", timestamp, session_counter,
        tostring(math.random(100000, 999999)))
end

-- สร้าง session ใหม่
function session_auth.create_session(user_id, user_data)
    local session_id = generate_session_id()
    local expires_at = os.time() + (60 * 60 * 24) -- 24 ชั่วโมง
    sessions[session_id] = {
        user_id   = user_id,
        user_data = user_data,
        created_at = os.time(),
        expires_at = expires_at,
        last_active = os.time(),
    }
    return session_id
end

-- ดึง session
function session_auth.get_session(session_id)
    local session = sessions[session_id]
    if not session then
        return nil, "Session not found"
    end
    if os.time() > session.expires_at then
        sessions[session_id] = nil
        return nil, "Session expired"
    end
    -- อัปเดต last_active
    session.last_active = os.time()
    return session, nil
end

-- ลบ session (logout)
function session_auth.destroy_session(session_id)
    if sessions[session_id] then
        sessions[session_id] = nil
        return true
    end
    return false
end

-- ตรวจสอบ session ทั้งหมด (สำหรับ cleanup)
function session_auth.cleanup_expired()
    local now = os.time()
    local cleaned = 0
    for id, session in pairs(sessions) do
        if now > session.expires_at then
            sessions[id] = nil
            cleaned = cleaned + 1
        end
    end
    return cleaned
end

-- ทดสอบ
math.randomseed(os.time())
local sid = session_auth.create_session(1, { username = "alice", role = "admin" })
print("Session created:", sid)

local sess, err = session_auth.get_session(sid)
if sess then
    print("User:", sess.user_data.username)
    print("Role:", sess.user_data.role)
else
    print("Error:", err)
end

session_auth.destroy_session(sid)
local sess2, err2 = session_auth.get_session(sid)
print("After destroy:", err2)
```

---

## ตัวอย่างที่ 3: JWT Header และ Payload

```lua
-- JWT structure concepts
local jwt_structure = {}

-- Base64url encoding (simplified)
local function base64url_encode(data)
    local b64chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
    local result = {}
    local padding = 0
    
    for i = 1, #data, 3 do
        local b1 = string.byte(data, i) or 0
        local b2 = string.byte(data, i+1) or 0
        local b3 = string.byte(data, i+2) or 0
        
        if i+1 > #data then padding = 2
        elseif i+2 > #data then padding = 1 end
        
        local n = (b1 * 65536) + (b2 * 256) + b3
        table.insert(result, string.sub(b64chars, math.floor(n/262144)+1, math.floor(n/262144)+1))
        table.insert(result, string.sub(b64chars, math.floor((n%262144)/4096)+1, math.floor((n%262144)/4096)+1))
        table.insert(result, padding < 2 and string.sub(b64chars, math.floor((n%4096)/64)+1, math.floor((n%4096)/64)+1) or "=")
        table.insert(result, padding < 1 and string.sub(b64chars, (n%64)+1, (n%64)+1) or "=")
    end
    
    local encoded = table.concat(result)
    -- แปลง base64 เป็น base64url
    encoded = encoded:gsub("+", "-"):gsub("/", "_"):gsub("=", "")
    return encoded
end

-- สร้าง JWT Header
function jwt_structure.create_header(algorithm)
    algorithm = algorithm or "HS256"
    local header = {
        alg = algorithm,
        typ = "JWT"
    }
    -- serialize to JSON (simplified)
    local json = string.format('{"alg":"%s","typ":"%s"}', header.alg, header.typ)
    return base64url_encode(json), json
end

-- สร้าง JWT Payload
function jwt_structure.create_payload(claims)
    local now = os.time()
    local payload = {
        sub = claims.subject or "user",
        iss = claims.issuer or "myapp",
        aud = claims.audience or "myapp-users",
        iat = now,
        exp = now + (claims.expires_in or 3600),
        jti = tostring(math.random(1000000, 9999999)),
    }
    -- เพิ่ม custom claims
    for k, v in pairs(claims) do
        if k ~= "subject" and k ~= "issuer" and k ~= "audience" and k ~= "expires_in" then
            payload[k] = v
        end
    end
    
    local parts = {}
    for k, v in pairs(payload) do
        if type(v) == "string" then
            table.insert(parts, string.format('"%s":"%s"', k, v))
        elseif type(v) == "number" then
            table.insert(parts, string.format('"%s":%d', k, v))
        end
    end
    local json = "{" .. table.concat(parts, ",") .. "}"
    return base64url_encode(json), json
end

math.randomseed(os.time())
local header_b64, header_json = jwt_structure.create_header("HS256")
local payload_b64, payload_json = jwt_structure.create_payload({
    subject = "user_123",
    issuer = "auth-service",
    role = "admin",
    expires_in = 7200,
})

print("=== JWT Structure ===")
print("Header JSON:", header_json)
print("Header B64:", header_b64)
print()
print("Payload JSON:", payload_json)
print("Payload B64:", payload_b64)
```

---

## ตัวอย่างที่ 4: HMAC Signature สำหรับ JWT

```lua
-- HMAC-SHA256 simulation for JWT signing
-- (ในการใช้งานจริงต้องใช้ library จริงเช่น lua-resty-hmac)

local jwt_sign = {}

-- Simple hash function (สำหรับตัวอย่าง - ไม่ใช้ใน production!)
local function simple_hmac(data, secret)
    local hash = 0
    for i = 1, #secret do
        hash = hash + string.byte(secret, i)
    end
    for i = 1, #data do
        local b = string.byte(data, i)
        hash = ((hash * 31) + b) % (2^32)
    end
    return string.format("%08x%08x%08x%08x", 
        hash % (2^32),
        (hash * 7) % (2^32),
        (hash * 13) % (2^32),
        (hash * 17) % (2^32))
end

local function base64url_simple(s)
    local result = ""
    for i = 1, #s do
        result = result .. string.format("%02x", string.byte(s, i))
    end
    return result:sub(1, 32) -- truncate for demo
end

-- สร้าง JWT token
function jwt_sign.create(header_b64, payload_b64, secret)
    local signing_input = header_b64 .. "." .. payload_b64
    local signature = simple_hmac(signing_input, secret)
    local sig_b64 = base64url_simple(signature)
    local token = signing_input .. "." .. sig_b64
    return token
end

-- ตรวจสอบ JWT token
function jwt_sign.verify(token, secret)
    local parts = {}
    for part in token:gmatch("[^%.]+") do
        table.insert(parts, part)
    end
    
    if #parts ~= 3 then
        return false, "Invalid token format"
    end
    
    local header_b64, payload_b64, signature = parts[1], parts[2], parts[3]
    local signing_input = header_b64 .. "." .. payload_b64
    local expected_sig = base64url_simple(simple_hmac(signing_input, secret))
    
    if signature ~= expected_sig then
        return false, "Invalid signature"
    end
    
    return true, "Valid token"
end

-- ทดสอบ
local SECRET = "my-super-secret-key-2024"
local header  = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9"
local payload = "eyJzdWIiOiJ1c2VyXzEyMyIsInJvbGUiOiJhZG1pbiJ9"

local token = jwt_sign.create(header, payload, SECRET)
print("JWT Token:", token)
print()

local valid, msg = jwt_sign.verify(token, SECRET)
print("Valid token:", valid, msg)

-- ทดสอบ token ที่ถูกแก้ไข
local tampered = token:sub(1, -5) .. "XXXX"
local valid2, msg2 = jwt_sign.verify(tampered, SECRET)
print("Tampered token:", valid2, msg2)
```

---

## ตัวอย่างที่ 5: JWT Manager ครบวงจร

```lua
-- Complete JWT Manager
local JWTManager = {}
JWTManager.__index = JWTManager

function JWTManager.new(config)
    local self = setmetatable({}, JWTManager)
    self.secret = config.secret or "default-secret"
    self.algorithm = config.algorithm or "HS256"
    self.default_expiry = config.default_expiry or 3600
    self.issuer = config.issuer or "lua-app"
    self.revoked_tokens = {} -- blacklist
    return self
end

-- Encode (สำหรับตัวอย่าง)
local function encode_b64(s)
    return s:gsub(".", function(c)
        return string.format("%02x", string.byte(c))
    end)
end

local function sign(data, secret)
    local h = 5381
    local combined = data .. secret
    for i = 1, #combined do
        h = ((h * 33) + string.byte(combined, i)) % (2^31)
    end
    return string.format("%x", h)
end

function JWTManager:generate(claims)
    local now = os.time()
    local payload = {
        iss = self.issuer,
        iat = now,
        exp = now + self.default_expiry,
        jti = tostring(now) .. tostring(math.random(1000,9999)),
    }
    for k, v in pairs(claims or {}) do
        payload[k] = v
    end
    
    -- สร้าง token parts
    local header_str = self.algorithm .. "|" .. payload.jti
    local payload_str = table.concat({
        payload.iss,
        tostring(payload.iat),
        tostring(payload.exp),
        payload.sub or "unknown",
        payload.role or "user",
    }, "|")
    
    local sig = sign(header_str .. "." .. payload_str, self.secret)
    local token = encode_b64(header_str) .. "." .. encode_b64(payload_str) .. "." .. sig
    
    return token, payload
end

function JWTManager:verify(token)
    -- ตรวจสอบ blacklist
    if self.revoked_tokens[token] then
        return nil, "Token has been revoked"
    end
    
    local parts = {}
    for p in token:gmatch("[^%.]+") do table.insert(parts, p) end
    if #parts ~= 3 then return nil, "Invalid format" end
    
    local header_enc, payload_enc, sig = parts[1], parts[2], parts[3]
    local expected_sig = sign(header_enc .. "." .. payload_enc, self.secret)
    
    if sig ~= expected_sig then
        return nil, "Invalid signature"
    end
    
    -- Decode payload (simplified)
    local payload_raw = payload_enc:gsub("%x%x", function(h)
        return string.char(tonumber(h, 16))
    end)
    
    local fields = {}
    for f in payload_raw:gmatch("[^|]+") do table.insert(fields, f) end
    
    local payload = {
        iss = fields[1],
        iat = tonumber(fields[2]),
        exp = tonumber(fields[3]),
        sub = fields[4],
        role = fields[5],
    }
    
    -- ตรวจสอบ expiry
    if os.time() > payload.exp then
        return nil, "Token expired"
    end
    
    return payload, nil
end

function JWTManager:revoke(token)
    self.revoked_tokens[token] = os.time()
end

-- ทดสอบ
math.randomseed(os.time())
local jm = JWTManager.new({
    secret = "super-secret-key",
    issuer = "my-service",
    default_expiry = 3600,
})

local token, payload = jm:generate({ sub = "user_42", role = "admin" })
print("Generated token:", token:sub(1, 50) .. "...")

local claims, err = jm:verify(token)
if claims then
    print("Verified! Subject:", claims.sub, "Role:", claims.role)
else
    print("Error:", err)
end

-- Revoke และตรวจสอบ
jm:revoke(token)
local claims2, err2 = jm:verify(token)
print("After revoke:", err2)
```

---

## ตัวอย่างที่ 6: Refresh Token System

```lua
-- Refresh Token implementation
local RefreshTokenSystem = {}

local access_tokens  = {}
local refresh_tokens = {}

-- สร้าง random token
local function gen_token(prefix)
    local chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    local token = prefix .. "_"
    for _ = 1, 32 do
        local idx = math.random(1, #chars)
        token = token .. chars:sub(idx, idx)
    end
    return token
end

-- Issue tokens คู่ (access + refresh)
function RefreshTokenSystem.issue_tokens(user_id, metadata)
    local access_token  = gen_token("acc")
    local refresh_token = gen_token("ref")
    local now = os.time()
    
    access_tokens[access_token] = {
        user_id = user_id,
        metadata = metadata or {},
        created_at = now,
        expires_at = now + 900, -- 15 นาที
        refresh_token = refresh_token,
    }
    
    refresh_tokens[refresh_token] = {
        user_id = user_id,
        metadata = metadata or {},
        created_at = now,
        expires_at = now + (7 * 24 * 3600), -- 7 วัน
        access_token = access_token,
        rotated = false,
    }
    
    return {
        access_token  = access_token,
        refresh_token = refresh_token,
        expires_in    = 900,
        token_type    = "Bearer",
    }
end

-- ตรวจสอบ access token
function RefreshTokenSystem.verify_access(token)
    local info = access_tokens[token]
    if not info then return nil, "Token not found" end
    if os.time() > info.expires_at then
        access_tokens[token] = nil
        return nil, "Token expired"
    end
    return info, nil
end

-- Refresh: สร้าง access token ใหม่จาก refresh token
function RefreshTokenSystem.refresh(refresh_token)
    local rt = refresh_tokens[refresh_token]
    if not rt then return nil, "Refresh token not found" end
    if rt.rotated then return nil, "Refresh token already used (rotation)" end
    if os.time() > rt.expires_at then
        refresh_tokens[refresh_token] = nil
        return nil, "Refresh token expired"
    end
    
    -- ยกเลิก access token เก่า
    if rt.access_token and access_tokens[rt.access_token] then
        access_tokens[rt.access_token] = nil
    end
    
    -- Mark refresh token เป็น rotated
    rt.rotated = true
    
    -- ออก tokens ใหม่
    local new_tokens = RefreshTokenSystem.issue_tokens(rt.user_id, rt.metadata)
    return new_tokens, nil
end

-- Revoke all tokens สำหรับ user
function RefreshTokenSystem.revoke_all(user_id)
    local count = 0
    for k, v in pairs(access_tokens) do
        if v.user_id == user_id then
            access_tokens[k] = nil
            count = count + 1
        end
    end
    for k, v in pairs(refresh_tokens) do
        if v.user_id == user_id then
            refresh_tokens[k] = nil
            count = count + 1
        end
    end
    return count
end

-- ทดสอบ
math.randomseed(os.time())
print("=== Refresh Token System ===")

local tokens = RefreshTokenSystem.issue_tokens(1, { username = "alice" })
print("Access token:", tokens.access_token:sub(1, 20) .. "...")
print("Refresh token:", tokens.refresh_token:sub(1, 20) .. "...")
print("Expires in:", tokens.expires_in, "seconds")

-- ตรวจสอบ access token
local info, err = RefreshTokenSystem.verify_access(tokens.access_token)
if info then print("Access token valid, user:", info.user_id)
else print("Error:", err) end

-- Refresh
local new_tokens, err2 = RefreshTokenSystem.refresh(tokens.refresh_token)
if new_tokens then
    print("New access token:", new_tokens.access_token:sub(1, 20) .. "...")
    print("New refresh token:", new_tokens.refresh_token:sub(1, 20) .. "...")
else
    print("Refresh error:", err2)
end

-- ลอง refresh อีกครั้ง (ต้องล้มเหลว - rotation)
local _, err3 = RefreshTokenSystem.refresh(tokens.refresh_token)
print("Double refresh attempt:", err3)
```

---

## ตัวอย่างที่ 7: OAuth2 Authorization Code Flow

```lua
-- OAuth2 Authorization Code Flow simulation
local OAuth2 = {}

local auth_codes  = {}
local oauth_tokens = {}
local clients = {
    ["client_123"] = {
        secret = "secret_abc",
        redirect_uris = { "https://myapp.com/callback", "http://localhost:3000/callback" },
        scopes = { "read", "write", "profile" },
        name = "My Web App",
    }
}

local function gen_code(len)
    local chars = "abcdefghijklmnopqrstuvwxyz0123456789"
    local code = ""
    for _ = 1, len do
        code = code .. chars:sub(math.random(1, #chars), math.random(1, #chars))
    end
    return code
end

-- Step 1: Authorization Endpoint - ผู้ใช้อนุญาต
function OAuth2.authorize(params)
    local client = clients[params.client_id]
    if not client then
        return nil, "Unknown client"
    end
    
    -- ตรวจสอบ redirect_uri
    local valid_uri = false
    for _, uri in ipairs(client.redirect_uris) do
        if uri == params.redirect_uri then valid_uri = true; break end
    end
    if not valid_uri then return nil, "Invalid redirect_uri" end
    
    -- ตรวจสอบ scope
    local requested_scopes = {}
    for scope in (params.scope or ""):gmatch("%S+") do
        table.insert(requested_scopes, scope)
    end
    
    -- สร้าง authorization code
    local code = gen_code(32)
    auth_codes[code] = {
        client_id    = params.client_id,
        user_id      = params.user_id,
        redirect_uri = params.redirect_uri,
        scope        = requested_scopes,
        expires_at   = os.time() + 600, -- 10 นาที
        used         = false,
        state        = params.state,
    }
    
    return {
        code  = code,
        state = params.state,
        redirect_uri = params.redirect_uri .. "?code=" .. code .. "&state=" .. (params.state or ""),
    }, nil
end

-- Step 2: Token Endpoint - แลก code เป็น token
function OAuth2.exchange_code(params)
    local code_info = auth_codes[params.code]
    if not code_info then return nil, "Invalid authorization code" end
    if code_info.used then return nil, "Code already used" end
    if os.time() > code_info.expires_at then
        auth_codes[params.code] = nil
        return nil, "Code expired"
    end
    if code_info.client_id ~= params.client_id then
        return nil, "Client mismatch"
    end
    
    -- ตรวจสอบ client secret
    local client = clients[params.client_id]
    if client.secret ~= params.client_secret then
        return nil, "Invalid client secret"
    end
    
    -- Mark code ว่าใช้แล้ว
    code_info.used = true
    
    -- สร้าง tokens
    local access_token  = "at_" .. gen_code(40)
    local refresh_token = "rt_" .. gen_code(40)
    local now = os.time()
    
    oauth_tokens[access_token] = {
        client_id  = params.client_id,
        user_id    = code_info.user_id,
        scope      = code_info.scope,
        created_at = now,
        expires_at = now + 3600,
        token_type = "Bearer",
    }
    
    return {
        access_token  = access_token,
        refresh_token = refresh_token,
        token_type    = "Bearer",
        expires_in    = 3600,
        scope         = table.concat(code_info.scope, " "),
    }, nil
end

-- ตรวจสอบ access token
function OAuth2.introspect(token)
    local info = oauth_tokens[token]
    if not info then
        return { active = false }
    end
    if os.time() > info.expires_at then
        oauth_tokens[token] = nil
        return { active = false }
    end
    return {
        active     = true,
        client_id  = info.client_id,
        user_id    = info.user_id,
        scope      = table.concat(info.scope, " "),
        exp        = info.expires_at,
        token_type = info.token_type,
    }
end

-- ทดสอบ OAuth2 flow
math.randomseed(os.time())
print("=== OAuth2 Authorization Code Flow ===")

-- Step 1: User authorizes
local auth_result, err = OAuth2.authorize({
    client_id    = "client_123",
    redirect_uri = "https://myapp.com/callback",
    scope        = "read profile",
    state        = "random_state_xyz",
    user_id      = 42,
})

if auth_result then
    print("Authorization code:", auth_result.code:sub(1,10) .. "...")
    print("Redirect URL:", auth_result.redirect_uri:sub(1, 60) .. "...")
else
    print("Auth error:", err)
    return
end

-- Step 2: Exchange code for token
local token_result, err2 = OAuth2.exchange_code({
    code          = auth_result.code,
    client_id     = "client_123",
    client_secret = "secret_abc",
    redirect_uri  = "https://myapp.com/callback",
})

if token_result then
    print("Access token:", token_result.access_token:sub(1, 15) .. "...")
    print("Token type:", token_result.token_type)
    print("Expires in:", token_result.expires_in)
    print("Scope:", token_result.scope)
    
    -- Introspect token
    local introspect = OAuth2.introspect(token_result.access_token)
    print("Token active:", introspect.active)
    print("User ID:", introspect.user_id)
else
    print("Token error:", err2)
end
```

---

## ตัวอย่างที่ 8: API Key Authentication

```lua
-- API Key Authentication system
local APIKeyAuth = {}

local api_keys = {
    ["ak_live_abc123xyz"] = {
        name        = "Production App",
        user_id     = 1,
        permissions = { "read", "write" },
        rate_limit  = 1000, -- requests per hour
        requests    = 0,
        window_start = os.time(),
        active      = true,
        created_at  = os.time(),
    },
    ["ak_test_def456uvw"] = {
        name        = "Test App",
        user_id     = 2,
        permissions = { "read" },
        rate_limit  = 100,
        requests    = 0,
        window_start = os.time(),
        active      = true,
        created_at  = os.time(),
    },
}

-- สร้าง API key ใหม่
function APIKeyAuth.create_key(user_id, name, permissions, rate_limit)
    local key_id = "ak_live_"
    local chars = "abcdefghijklmnopqrstuvwxyz0123456789"
    for _ = 1, 16 do
        key_id = key_id .. chars:sub(math.random(1, #chars), math.random(1, #chars))
    end
    
    api_keys[key_id] = {
        name        = name,
        user_id     = user_id,
        permissions = permissions or { "read" },
        rate_limit  = rate_limit or 100,
        requests    = 0,
        window_start = os.time(),
        active      = true,
        created_at  = os.time(),
    }
    
    return key_id
end

-- ตรวจสอบ API key
function APIKeyAuth.validate(api_key, required_permission)
    local key_info = api_keys[api_key]
    if not key_info then
        return nil, "Invalid API key"
    end
    if not key_info.active then
        return nil, "API key is disabled"
    end
    
    -- Rate limiting: reset window ทุก 1 ชั่วโมง
    local now = os.time()
    if now - key_info.window_start >= 3600 then
        key_info.requests = 0
        key_info.window_start = now
    end
    
    -- ตรวจสอบ rate limit
    if key_info.requests >= key_info.rate_limit then
        local reset_in = 3600 - (now - key_info.window_start)
        return nil, string.format("Rate limit exceeded. Reset in %d seconds", reset_in)
    end
    
    -- ตรวจสอบ permission
    if required_permission then
        local has_perm = false
        for _, p in ipairs(key_info.permissions) do
            if p == required_permission then has_perm = true; break end
        end
        if not has_perm then
            return nil, "Insufficient permissions"
        end
    end
    
    key_info.requests = key_info.requests + 1
    
    return {
        user_id     = key_info.user_id,
        name        = key_info.name,
        permissions = key_info.permissions,
        requests_remaining = key_info.rate_limit - key_info.requests,
    }, nil
end

-- Revoke API key
function APIKeyAuth.revoke(api_key)
    if api_keys[api_key] then
        api_keys[api_key].active = false
        return true
    end
    return false
end

-- ทดสอบ
math.randomseed(os.time())
print("=== API Key Authentication ===")

-- สร้าง key ใหม่
local new_key = APIKeyAuth.create_key(3, "Mobile App", {"read", "write"}, 500)
print("New API key:", new_key)

-- ตรวจสอบ key ที่มีอยู่
local result, err = APIKeyAuth.validate("ak_live_abc123xyz", "write")
if result then
    print("Valid! User:", result.user_id, "Remaining:", result.requests_remaining)
else
    print("Error:", err)
end

-- ตรวจสอบ key ที่ไม่มี permission
local result2, err2 = APIKeyAuth.validate("ak_test_def456uvw", "write")
print("Read-only key trying to write:", err2)

-- Revoke key
APIKeyAuth.revoke("ak_live_abc123xyz")
local result3, err3 = APIKeyAuth.validate("ak_live_abc123xyz")
print("Revoked key:", err3)
```

---

## ตัวอย่างที่ 9: Basic Authentication

```lua
-- HTTP Basic Authentication
local BasicAuth = {}

-- decode base64 (simplified)
local function decode_base64(s)
    -- ในการใช้งานจริงต้องใช้ library
    -- นี่เป็น mock สำหรับตัวอย่าง
    return s -- return as-is สำหรับ demo
end

-- Parse Authorization header
function BasicAuth.parse_header(auth_header)
    if not auth_header then return nil, "No Authorization header" end
    
    local scheme, credentials = auth_header:match("^(%S+)%s+(.+)$")
    if not scheme or scheme:lower() ~= "basic" then
        return nil, "Invalid scheme (expected Basic)"
    end
    
    -- Decode base64 credentials
    -- ในระบบจริง: credentials = decode_base64(credentials)
    -- สำหรับตัวอย่าง: ใช้ "username:password" โดยตรง
    local username, password = credentials:match("^([^:]+):(.+)$")
    if not username or not password then
        return nil, "Invalid credentials format"
    end
    
    return { username = username, password = password }, nil
end

-- ตรวจสอบ credentials
local user_db = {
    ["admin"]  = { password = "admin123",  role = "admin",  id = 1 },
    ["alice"]  = { password = "alice456",  role = "user",   id = 2 },
    ["apibot"] = { password = "botpass789", role = "service", id = 3 },
}

function BasicAuth.authenticate(username, password)
    local user = user_db[username]
    if not user then return nil, "User not found" end
    
    -- ในระบบจริงใช้ bcrypt.verify
    if user.password ~= password then
        return nil, "Invalid password"
    end
    
    return { id = user.id, username = username, role = user.role }, nil
end

-- Middleware สำหรับ HTTP server
function BasicAuth.middleware(request)
    local auth_header = request.headers and request.headers["Authorization"]
    
    local creds, err = BasicAuth.parse_header(auth_header)
    if not creds then
        return {
            status = 401,
            headers = { ["WWW-Authenticate"] = 'Basic realm="My API"' },
            body = '{"error": "' .. err .. '"}',
        }
    end
    
    local user, auth_err = BasicAuth.authenticate(creds.username, creds.password)
    if not user then
        return {
            status = 401,
            headers = { ["WWW-Authenticate"] = 'Basic realm="My API"' },
            body = '{"error": "' .. auth_err .. '"}',
        }
    end
    
    request.user = user
    return nil -- ผ่าน middleware
end

-- ทดสอบ
print("=== Basic Authentication ===")

-- สร้าง mock requests
local requests = {
    { headers = { ["Authorization"] = "Basic admin:admin123" }, name = "Valid admin" },
    { headers = { ["Authorization"] = "Basic alice:wrongpass" }, name = "Wrong password" },
    { headers = {}, name = "No auth header" },
    { headers = { ["Authorization"] = "Bearer some-token" }, name = "Wrong scheme" },
}

for _, req in ipairs(requests) do
    local err_response = BasicAuth.middleware(req)
    if err_response then
        print(req.name .. ": HTTP " .. err_response.status)
    else
        print(req.name .. ": OK, user=" .. req.user.username .. ", role=" .. req.user.role)
    end
end
```

---

## ตัวอย่างที่ 10: Password Hashing (bcrypt concepts)

```lua
-- Password hashing concepts (bcrypt simulation)
local PasswordHash = {}

-- Simulate bcrypt-like hashing
-- ในการใช้งานจริงต้องใช้ lua-bcrypt หรือ OpenSSL binding

local function pseudo_hash(password, salt, rounds)
    rounds = rounds or 12
    local hash = password .. salt
    -- Simulate CPU-intensive operation
    for _ = 1, rounds * 100 do
        local val = 0
        for i = 1, #hash do
            val = (val * 31 + string.byte(hash, i)) % (2^31)
        end
        hash = string.format("%x", val) .. hash:sub(1, 16)
    end
    return hash:sub(1, 60)
end

local function gen_salt(length)
    local chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789./"
    local salt = ""
    for _ = 1, (length or 22) do
        salt = salt .. chars:sub(math.random(1, #chars), math.random(1, #chars))
    end
    return salt
end

-- Hash password
function PasswordHash.hash(password, rounds)
    rounds = rounds or 12
    local salt = gen_salt(22)
    local hash = pseudo_hash(password, salt, rounds)
    -- bcrypt format: $2b$ROUNDS$SALT+HASH
    return string.format("$2b$%02d$%s%s", rounds, salt, hash:sub(1, 31))
end

-- ตรวจสอบ password
function PasswordHash.verify(password, stored_hash)
    -- Parse stored hash
    local rounds_str, salt_and_hash = stored_hash:match("^%$2b%$(%d+)%$(.+)$")
    if not rounds_str then return false end
    
    local rounds = tonumber(rounds_str)
    local salt = salt_and_hash:sub(1, 22)
    local stored = salt_and_hash:sub(23)
    
    local computed = pseudo_hash(password, salt, rounds)
    return computed:sub(1, 31) == stored
end

-- Password strength checker
function PasswordHash.check_strength(password)
    local score = 0
    local feedback = {}
    
    if #password >= 8  then score = score + 1 else table.insert(feedback, "ต้องมีอย่างน้อย 8 ตัวอักษร") end
    if #password >= 12 then score = score + 1 end
    if password:match("%d")  then score = score + 1 else table.insert(feedback, "ควรมีตัวเลข") end
    if password:match("%u")  then score = score + 1 else table.insert(feedback, "ควรมีตัวพิมพ์ใหญ่") end
    if password:match("%l")  then score = score + 1 else table.insert(feedback, "ควรมีตัวพิมพ์เล็ก") end
    if password:match("[%p%W]") then score = score + 2 else table.insert(feedback, "ควรมีอักขระพิเศษ") end
    
    -- ตรวจสอบ common passwords
    local common = { "password", "123456", "admin", "qwerty", "letmein" }
    for _, p in ipairs(common) do
        if password:lower() == p then
            score = 0
            table.insert(feedback, "รหัสผ่านนี้พบบ่อยเกินไป")
            break
        end
    end
    
    local strength
    if score <= 2 then strength = "WEAK"
    elseif score <= 4 then strength = "FAIR"
    elseif score <= 6 then strength = "GOOD"
    else strength = "STRONG" end
    
    return { score = score, strength = strength, feedback = feedback }
end

-- ทดสอบ
math.randomseed(os.time())
print("=== Password Hashing ===")

local password = "MyP@ssw0rd2024!"
local hashed = PasswordHash.hash(password)
print("Original:", password)
print("Hashed:", hashed:sub(1, 40) .. "...")

local verify1 = PasswordHash.verify(password, hashed)
local verify2 = PasswordHash.verify("WrongPassword", hashed)
print("Correct password:", verify1)
print("Wrong password:", verify2)

print()
print("=== Password Strength ===")
local passwords_to_test = {
    "abc", "password", "MyPass1", "MyP@ss1234", "C0mpl3x!P@ssw0rd#2024"
}
for _, p in ipairs(passwords_to_test) do
    local strength = PasswordHash.check_strength(p)
    print(string.format("'%s': %s (score: %d)", p, strength.strength, strength.score))
    if #strength.feedback > 0 then
        print("  Feedback: " .. table.concat(strength.feedback, ", "))
    end
end
```

---

## ตัวอย่างที่ 11: Role-Based Access Control (RBAC)

```lua
-- RBAC - Role-Based Access Control
local RBAC = {}
RBAC.__index = RBAC

function RBAC.new()
    local self = setmetatable({}, RBAC)
    self.roles = {}
    self.user_roles = {}
    self.role_hierarchy = {}
    return self
end

-- กำหนด role และ permissions
function RBAC:define_role(role_name, permissions, parent_roles)
    self.roles[role_name] = {
        permissions = {},
        parents = parent_roles or {},
    }
    for _, perm in ipairs(permissions or {}) do
        self.roles[role_name].permissions[perm] = true
    end
end

-- ดึง permissions ทั้งหมดของ role (รวม inherited)
function RBAC:get_permissions(role_name, visited)
    visited = visited or {}
    if visited[role_name] then return {} end
    visited[role_name] = true
    
    local role = self.roles[role_name]
    if not role then return {} end
    
    local perms = {}
    -- permissions ของ role นี้
    for perm in pairs(role.permissions) do
        perms[perm] = true
    end
    
    -- inherited permissions จาก parent roles
    for _, parent in ipairs(role.parents) do
        for perm in pairs(self:get_permissions(parent, visited)) do
            perms[perm] = true
        end
    end
    
    return perms
end

-- กำหนด roles ให้ผู้ใช้
function RBAC:assign_role(user_id, role_name)
    if not self.roles[role_name] then
        return false, "Role not found"
    end
    if not self.user_roles[user_id] then
        self.user_roles[user_id] = {}
    end
    self.user_roles[user_id][role_name] = true
    return true
end

-- ตรวจสอบว่า user มี permission ไหม
function RBAC:can(user_id, permission)
    local roles = self.user_roles[user_id]
    if not roles then return false end
    
    for role_name in pairs(roles) do
        local perms = self:get_permissions(role_name)
        if perms[permission] then return true end
    end
    
    return false
end

-- ดึง roles ของ user
function RBAC:get_user_roles(user_id)
    local result = {}
    for role in pairs(self.user_roles[user_id] or {}) do
        table.insert(result, role)
    end
    return result
end

-- ทดสอบ RBAC
local rbac = RBAC.new()

-- กำหนด roles (hierarchy)
rbac:define_role("viewer", { "posts:read", "comments:read" })
rbac:define_role("editor", { "posts:write", "posts:update", "comments:write" }, { "viewer" })
rbac:define_role("moderator", { "comments:delete", "users:read" }, { "editor" })
rbac:define_role("admin", { "users:write", "users:delete", "settings:manage" }, { "moderator" })
rbac:define_role("superadmin", { "*:*" }, { "admin" })

-- กำหนด roles ให้ users
rbac:assign_role(1, "viewer")
rbac:assign_role(2, "editor")
rbac:assign_role(3, "admin")
rbac:assign_role(4, "superadmin")

print("=== RBAC System ===")

local tests = {
    { user = 1, perm = "posts:read",    expected = true  },
    { user = 1, perm = "posts:write",   expected = false },
    { user = 2, perm = "posts:write",   expected = true  },
    { user = 2, perm = "users:delete",  expected = false },
    { user = 3, perm = "comments:delete", expected = true },
    { user = 3, perm = "settings:manage", expected = true },
    { user = 4, perm = "*:*",           expected = true  },
}

for _, test in ipairs(tests) do
    local result = rbac:can(test.user, test.perm)
    local status = (result == test.expected) and "PASS" or "FAIL"
    print(string.format("[%s] User %d can '%s': %s (expected: %s)",
        status, test.user, test.perm, tostring(result), tostring(test.expected)))
end

print()
print("User 3 roles:", table.concat(rbac:get_user_roles(3), ", "))

-- แสดง permissions ทั้งหมดของ editor
local editor_perms = rbac:get_permissions("editor")
local perm_list = {}
for p in pairs(editor_perms) do table.insert(perm_list, p) end
table.sort(perm_list)
print("Editor permissions:", table.concat(perm_list, ", "))
```

---

## ตัวอย่างที่ 12: Permission System with Resources

```lua
-- Fine-grained permission system
local PermissionSystem = {}
PermissionSystem.__index = PermissionSystem

function PermissionSystem.new()
    local self = setmetatable({}, PermissionSystem)
    self.policies = {}
    return self
end

-- เพิ่ม policy: user X สามารถ action Y บน resource Z
function PermissionSystem:add_policy(subject, action, resource, conditions)
    local key = subject .. ":" .. action .. ":" .. resource
    self.policies[key] = {
        subject    = subject,
        action     = action,
        resource   = resource,
        conditions = conditions or {},
        effect     = "allow",
    }
end

-- ลบ policy
function PermissionSystem:remove_policy(subject, action, resource)
    local key = subject .. ":" .. action .. ":" .. resource
    self.policies[key] = nil
end

-- ตรวจสอบด้วย wildcard
function PermissionSystem:check(subject, action, resource, context)
    -- ตรวจสอบตั้งแต่ specific ไป wildcard
    local patterns = {
        subject .. ":" .. action .. ":" .. resource,
        subject .. ":" .. action .. ":*",
        subject .. ":*:" .. resource,
        subject .. ":*:*",
        "role:" .. (context and context.role or "") .. ":" .. action .. ":" .. resource,
    }
    
    for _, pattern in ipairs(patterns) do
        local policy = self.policies[pattern]
        if policy then
            -- ตรวจสอบ conditions
            if context and next(policy.conditions) then
                local conditions_met = true
                for cond_key, cond_val in pairs(policy.conditions) do
                    if context[cond_key] ~= cond_val then
                        conditions_met = false
                        break
                    end
                end
                if conditions_met then return true, policy end
            else
                return true, policy
            end
        end
    end
    
    return false, nil
end

-- ทดสอบ
local ps = PermissionSystem.new()

-- กำหนด policies
ps:add_policy("user:1", "read",   "document:*")
ps:add_policy("user:1", "write",  "document:own")
ps:add_policy("user:2", "read",   "document:public")
ps:add_policy("user:3", "*",      "*") -- admin
ps:add_policy("user:1", "delete", "document:*", { owner = true })

print("=== Permission System ===")

local checks = {
    { sub="user:1", act="read",   res="document:doc_1",   ctx=nil },
    { sub="user:1", act="write",  res="document:own",     ctx=nil },
    { sub="user:2", act="write",  res="document:public",  ctx=nil },
    { sub="user:3", act="delete", res="anything",         ctx=nil },
    { sub="user:1", act="delete", res="document:*",       ctx={ owner=true } },
    { sub="user:1", act="delete", res="document:*",       ctx={ owner=false } },
}

for _, c in ipairs(checks) do
    local allowed, _ = ps:check(c.sub, c.act, c.res, c.ctx)
    local ctx_str = c.ctx and (" (owner=" .. tostring(c.ctx.owner) .. ")") or ""
    print(string.format("%s can %s %s%s: %s",
        c.sub, c.act, c.res, ctx_str, allowed and "ALLOWED" or "DENIED"))
end
```

---

## ตัวอย่างที่ 13: Token Revocation (Blacklist)

```lua
-- Token revocation with blacklist
local TokenRevocation = {}

-- In-memory blacklist (ในระบบจริงใช้ Redis)
local blacklist = {}
local blacklist_cleanup_interval = 3600

-- เพิ่ม token ใน blacklist
function TokenRevocation.revoke(token_id, reason, expires_at)
    blacklist[token_id] = {
        reason = reason or "manual_revoke",
        revoked_at = os.time(),
        expires_at = expires_at or (os.time() + 86400),
    }
    return true
end

-- ตรวจสอบว่า token ถูก revoke หรือยัง
function TokenRevocation.is_revoked(token_id)
    local entry = blacklist[token_id]
    if not entry then return false end
    
    -- ถ้า expires_at ผ่านไปแล้ว ไม่ต้องอยู่ใน blacklist แล้ว
    if os.time() > entry.expires_at then
        blacklist[token_id] = nil
        return false
    end
    
    return true, entry.reason
end

-- Revoke all tokens สำหรับ user (เก็บ timestamp)
local user_revoke_timestamps = {}

function TokenRevocation.revoke_all_for_user(user_id)
    user_revoke_timestamps[user_id] = os.time()
end

-- ตรวจสอบว่า token ออกก่อน revoke timestamp
function TokenRevocation.is_valid_for_user(user_id, token_issued_at)
    local revoke_time = user_revoke_timestamps[user_id]
    if not revoke_time then return true end
    return token_issued_at > revoke_time
end

-- Cleanup expired entries
function TokenRevocation.cleanup()
    local now = os.time()
    local removed = 0
    for id, entry in pairs(blacklist) do
        if now > entry.expires_at then
            blacklist[id] = nil
            removed = removed + 1
        end
    end
    return removed
end

-- Stats
function TokenRevocation.stats()
    local count = 0
    for _ in pairs(blacklist) do count = count + 1 end
    return {
        blacklisted_count = count,
        user_revocations = (function()
            local c = 0
            for _ in pairs(user_revoke_timestamps) do c = c + 1 end
            return c
        end)(),
    }
end

-- ทดสอบ
print("=== Token Revocation ===")

-- Revoke specific tokens
TokenRevocation.revoke("jti_abc123", "logout", os.time() + 3600)
TokenRevocation.revoke("jti_def456", "password_change", os.time() + 7200)

-- ตรวจสอบ
print("jti_abc123 revoked:", TokenRevocation.is_revoked("jti_abc123"))
print("jti_unknown revoked:", TokenRevocation.is_revoked("jti_unknown"))

-- User-level revocation
TokenRevocation.revoke_all_for_user(1)

-- Token ออกก่อน revoke
local old_token_iat  = os.time() - 100
local new_token_iat  = os.time() + 100

print("Old token (before revoke) valid:", TokenRevocation.is_valid_for_user(1, old_token_iat))
print("New token (after revoke) valid:", TokenRevocation.is_valid_for_user(1, new_token_iat))

-- Stats
local stats = TokenRevocation.stats()
print("Blacklisted tokens:", stats.blacklisted_count)
print("Users with revoked tokens:", stats.user_revocations)
```

---

## ตัวอย่างที่ 14: Multi-Factor Authentication (MFA)

```lua
-- Multi-Factor Authentication concepts
local MFA = {}

-- TOTP (Time-based One-Time Password) simulation
-- ในระบบจริงใช้ library TOTP ที่ถูกต้อง

local function generate_totp_code(secret, time_step)
    time_step = time_step or 30
    local timestamp = math.floor(os.time() / time_step)
    
    -- สร้าง code จาก secret และ timestamp (simplified)
    local combined = secret .. tostring(timestamp)
    local hash = 0
    for i = 1, #combined do
        hash = (hash * 31 + string.byte(combined, i)) % (10^8)
    end
    return string.format("%06d", hash % 1000000)
end

-- สร้าง TOTP secret
function MFA.generate_secret()
    local chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567" -- Base32 alphabet
    local secret = ""
    for _ = 1, 32 do
        secret = secret .. chars:sub(math.random(1, #chars), math.random(1, #chars))
    end
    return secret
end

-- สร้าง provisioning URI สำหรับ QR code
function MFA.get_provisioning_uri(secret, username, issuer)
    issuer = issuer or "MyApp"
    return string.format("otpauth://totp/%s:%s?secret=%s&issuer=%s&algorithm=SHA1&digits=6&period=30",
        issuer, username, secret, issuer)
end

-- ตรวจสอบ TOTP code
function MFA.verify_totp(secret, code, window)
    window = window or 1 -- ยอมรับ 1 step ก่อนและหลัง
    local time_step = 30
    local current_step = math.floor(os.time() / time_step)
    
    for i = -window, window do
        local expected = generate_totp_code(secret, time_step)
        -- คำนวณสำหรับ step ที่ต่างกัน
        local step_time = (current_step + i) * time_step
        local combined = secret .. tostring(math.floor(step_time / time_step))
        local hash = 0
        for j = 1, #combined do
            hash = (hash * 31 + string.byte(combined, j)) % (10^8)
        end
        local expected_code = string.format("%06d", hash % 1000000)
        if code == expected_code then return true end
    end
    
    return false
end

-- SMS OTP simulation
local pending_otps = {}

function MFA.send_sms_otp(user_id, phone)
    local otp = string.format("%06d", math.random(100000, 999999))
    pending_otps[user_id] = {
        code       = otp,
        phone      = phone,
        expires_at = os.time() + 300, -- 5 นาที
        attempts   = 0,
    }
    -- ในระบบจริง: ส่ง SMS ผ่าน provider
    print(string.format("SMS sent to %s: Your OTP is %s (expires in 5 min)", phone, otp))
    return true
end

function MFA.verify_sms_otp(user_id, code)
    local otp_info = pending_otps[user_id]
    if not otp_info then return false, "No OTP found" end
    if os.time() > otp_info.expires_at then
        pending_otps[user_id] = nil
        return false, "OTP expired"
    end
    
    otp_info.attempts = otp_info.attempts + 1
    if otp_info.attempts > 3 then
        pending_otps[user_id] = nil
        return false, "Too many attempts"
    end
    
    if otp_info.code == code then
        pending_otps[user_id] = nil
        return true, "OTP verified"
    end
    
    return false, "Invalid OTP"
end

-- MFA enrollment
function MFA.enroll_totp(user_id, username)
    local secret = MFA.generate_secret()
    local uri = MFA.get_provisioning_uri(secret, username, "MySecureApp")
    
    return {
        secret = secret,
        qr_uri = uri,
        backup_codes = (function()
            local codes = {}
            for _ = 1, 10 do
                table.insert(codes, string.format("%04d-%04d",
                    math.random(1000, 9999), math.random(1000, 9999)))
            end
            return codes
        end)(),
    }
end

-- ทดสอบ
math.randomseed(os.time())
print("=== Multi-Factor Authentication ===")

-- TOTP enrollment
local enrollment = MFA.enroll_totp(1, "alice@example.com")
print("TOTP Secret:", enrollment.secret)
print("QR URI:", enrollment.qr_uri:sub(1, 60) .. "...")
print("Backup codes (first 3):")
for i = 1, 3 do print("  " .. enrollment.backup_codes[i]) end

-- สร้างและตรวจสอบ TOTP
local current_code = generate_totp_code(enrollment.secret, 30)
print("\nCurrent TOTP code:", current_code)

local verified = MFA.verify_totp(enrollment.secret, current_code)
print("TOTP verified:", verified)

local wrong_verified = MFA.verify_totp(enrollment.secret, "000000")
print("Wrong TOTP:", wrong_verified)

-- SMS OTP
print()
MFA.send_sms_otp(1, "+66-81-234-5678")
local ok, msg = MFA.verify_sms_otp(1, "wrong_code")
print("Wrong SMS OTP:", ok, msg)
```

---

## ตัวอย่างที่ 15: Secure Cookie Handling

```lua
-- Secure Cookie management
local CookieManager = {}

-- Cookie attributes
function CookieManager.create_session_cookie(name, value, options)
    options = options or {}
    
    local cookie_parts = {
        name .. "=" .. value,
    }
    
    -- Path
    table.insert(cookie_parts, "Path=" .. (options.path or "/"))
    
    -- Domain
    if options.domain then
        table.insert(cookie_parts, "Domain=" .. options.domain)
    end
    
    -- Max-Age หรือ Expires
    if options.max_age then
        table.insert(cookie_parts, "Max-Age=" .. options.max_age)
    elseif options.expires then
        table.insert(cookie_parts, "Expires=" .. os.date("!%a, %d %b %Y %H:%M:%S GMT", options.expires))
    end
    
    -- Secure flag (HTTPS only)
    if options.secure ~= false then
        table.insert(cookie_parts, "Secure")
    end
    
    -- HttpOnly flag (ป้องกัน JavaScript access)
    if options.http_only ~= false then
        table.insert(cookie_parts, "HttpOnly")
    end
    
    -- SameSite
    local same_site = options.same_site or "Strict"
    table.insert(cookie_parts, "SameSite=" .. same_site)
    
    return table.concat(cookie_parts, "; ")
end

-- Parse cookie header
function CookieManager.parse(cookie_header)
    local cookies = {}
    if not cookie_header then return cookies end
    
    for pair in (cookie_header .. ";"):gmatch("([^;]*);") do
        local name, value = pair:match("^%s*([^=]+)=(.*)%s*$")
        if name and value then
            cookies[name:match("^%s*(.-)%s*$")] = value:match("^%s*(.-)%s*$")
        end
    end
    
    return cookies
end

-- Cookie signing เพื่อป้องกันการแก้ไข
function CookieManager.sign(value, secret)
    local hash = 0
    local combined = value .. secret
    for i = 1, #combined do
        hash = ((hash * 31) + string.byte(combined, i)) % (2^31)
    end
    return value .. "." .. string.format("%x", hash)
end

function CookieManager.verify_signed(signed_value, secret)
    local value, sig = signed_value:match("^(.+)%.([^%.]+)$")
    if not value or not sig then return nil, "Invalid format" end
    
    local expected = CookieManager.sign(value, secret)
    local _, expected_sig = expected:match("^(.+)%.([^%.]+)$")
    
    if sig ~= expected_sig then
        return nil, "Invalid signature"
    end
    
    return value, nil
end

-- ทดสอบ
print("=== Secure Cookie Handling ===")

-- สร้าง session cookie
local session_cookie = CookieManager.create_session_cookie(
    "session_id",
    "sess_abc123xyz",
    {
        max_age   = 86400,
        secure    = true,
        http_only = true,
        same_site = "Lax",
        domain    = "example.com",
    }
)
print("Session cookie:")
print(session_cookie)
print()

-- สร้าง cookie สำหรับ remember-me
local remember_cookie = CookieManager.create_session_cookie(
    "remember_token",
    "rmb_xyz789",
    {
        max_age   = 30 * 24 * 3600,
        secure    = true,
        http_only = true,
        same_site = "Strict",
    }
)
print("Remember-me cookie:")
print(remember_cookie)
print()

-- Parse cookies จาก request
local cookie_header = "session_id=sess_abc123xyz; user_pref=dark_mode; lang=th"
local cookies = CookieManager.parse(cookie_header)
for name, value in pairs(cookies) do
    print(string.format("Cookie '%s' = '%s'", name, value))
end
print()

-- Signed cookies
local SECRET = "cookie-signing-secret"
local signed = CookieManager.sign("user_id=42", SECRET)
print("Signed cookie:", signed)

local verified, err = CookieManager.verify_signed(signed, SECRET)
print("Verified:", verified, err)

local tampered = signed:sub(1, -5) .. "XXXX"
local v2, e2 = CookieManager.verify_signed(tampered, SECRET)
print("Tampered:", v2, e2)
```

---

## ตัวอย่างที่ 16: Session Management ครบวงจร

```lua
-- Complete Session Management System
local SessionManager = {}
SessionManager.__index = SessionManager

function SessionManager.new(config)
    local self = setmetatable({}, SessionManager)
    self.config = {
        secret            = config.secret or "session-secret",
        max_age           = config.max_age or 86400,
        rolling           = config.rolling ~= false, -- extend on activity
        max_per_user      = config.max_per_user or 5,
        regenerate_on_login = config.regenerate_on_login ~= false,
    }
    self.store = {} -- storage backend (ใช้ Redis ในระบบจริง)
    self.user_sessions = {} -- index: user_id -> [session_ids]
    return self
end

local function gen_id()
    local id = ""
    local chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    for _ = 1, 40 do
        id = id .. chars:sub(math.random(1, #chars), math.random(1, #chars))
    end
    return id
end

function SessionManager:create(user_id, data)
    -- ตรวจสอบ max sessions per user
    local user_sids = self.user_sessions[user_id] or {}
    if #user_sids >= self.config.max_per_user then
        -- ลบ session เก่าสุด
        local oldest_sid = user_sids[1]
        self:destroy(oldest_sid)
    end
    
    local sid = gen_id()
    local now = os.time()
    
    self.store[sid] = {
        id         = sid,
        user_id    = user_id,
        data       = data or {},
        created_at = now,
        updated_at = now,
        expires_at = now + self.config.max_age,
        ip         = data and data.ip,
        user_agent = data and data.user_agent,
    }
    
    -- Update user sessions index
    self.user_sessions[user_id] = self.user_sessions[user_id] or {}
    table.insert(self.user_sessions[user_id], sid)
    
    return sid
end

function SessionManager:get(sid)
    local session = self.store[sid]
    if not session then return nil, "Session not found" end
    
    local now = os.time()
    if now > session.expires_at then
        self:destroy(sid)
        return nil, "Session expired"
    end
    
    -- Rolling: extend expiry on access
    if self.config.rolling then
        session.expires_at = now + self.config.max_age
        session.updated_at = now
    end
    
    return session, nil
end

function SessionManager:set_data(sid, key, value)
    local session = self.store[sid]
    if not session then return false end
    session.data[key] = value
    session.updated_at = os.time()
    return true
end

function SessionManager:destroy(sid)
    local session = self.store[sid]
    if not session then return false end
    
    -- Remove from user index
    local user_sids = self.user_sessions[session.user_id]
    if user_sids then
        for i, id in ipairs(user_sids) do
            if id == sid then
                table.remove(user_sids, i)
                break
            end
        end
    end
    
    self.store[sid] = nil
    return true
end

function SessionManager:destroy_all_for_user(user_id, except_sid)
    local sids = self.user_sessions[user_id] or {}
    local destroyed = 0
    for _, sid in ipairs(sids) do
        if sid ~= except_sid then
            self.store[sid] = nil
            destroyed = destroyed + 1
        end
    end
    if except_sid then
        self.user_sessions[user_id] = { except_sid }
    else
        self.user_sessions[user_id] = {}
    end
    return destroyed
end

function SessionManager:get_user_sessions(user_id)
    local result = {}
    for _, sid in ipairs(self.user_sessions[user_id] or {}) do
        local session = self.store[sid]
        if session then
            table.insert(result, {
                id         = sid:sub(1, 8) .. "...",
                created_at = session.created_at,
                updated_at = session.updated_at,
                ip         = session.ip,
                user_agent = session.user_agent,
            })
        end
    end
    return result
end

-- ทดสอบ
math.randomseed(os.time())
local sm = SessionManager.new({
    secret       = "my-session-secret",
    max_age      = 3600,
    max_per_user = 3,
})

print("=== Session Management ===")

-- สร้าง sessions
local sid1 = sm:create(1, { ip = "192.168.1.1", user_agent = "Chrome/120" })
local sid2 = sm:create(1, { ip = "10.0.0.1", user_agent = "Firefox/121" })
print("Session 1:", sid1:sub(1, 10) .. "...")
print("Session 2:", sid2:sub(1, 10) .. "...")

-- Set data
sm:set_data(sid1, "cart_items", 3)
sm:set_data(sid1, "last_page", "/products")

-- Get session
local sess, err = sm:get(sid1)
if sess then
    print("Session data:", sess.data.cart_items, sess.data.last_page)
end

-- List user sessions
local sessions = sm:get_user_sessions(1)
print("User has", #sessions, "active sessions")

-- Destroy all except current
local destroyed = sm:destroy_all_for_user(1, sid1)
print("Destroyed", destroyed, "other sessions")

local remaining = sm:get_user_sessions(1)
print("Remaining sessions:", #remaining)
```

---

## ตัวอย่างที่ 17: Complete Authentication Middleware

```lua
-- Authentication middleware สำหรับ web framework
local AuthMiddleware = {}

-- Configuration
local config = {
    jwt_secret     = "my-jwt-secret",
    session_secret = "my-session-secret",
    token_header   = "Authorization",
    api_key_header = "X-API-Key",
}

-- Strategies
local strategies = {}

-- Strategy 1: Bearer Token (JWT)
strategies.bearer = function(request)
    local auth_header = request.headers and request.headers[config.token_header]
    if not auth_header then return nil end
    
    local token = auth_header:match("^Bearer%s+(.+)$")
    if not token then return nil end
    
    -- Verify JWT (simplified)
    local parts = {}
    for p in token:gmatch("[^%.]+") do table.insert(parts, p) end
    if #parts ~= 3 then return nil, "Invalid JWT format" end
    
    -- Mock decode (ในระบบจริงใช้ JWT library)
    return {
        user_id = 1,
        role    = "admin",
        method  = "bearer",
    }, nil
end

-- Strategy 2: API Key
strategies.api_key = function(request)
    local api_key = request.headers and request.headers[config.api_key_header]
    if not api_key then return nil end
    
    -- Mock API key validation
    local mock_keys = {
        ["ak_live_abc"] = { user_id = 1, scopes = {"read", "write"} },
        ["ak_live_xyz"] = { user_id = 2, scopes = {"read"} },
    }
    
    local key_info = mock_keys[api_key]
    if not key_info then return nil, "Invalid API key" end
    
    return {
        user_id = key_info.user_id,
        scopes  = key_info.scopes,
        method  = "api_key",
    }, nil
end

-- Strategy 3: Session Cookie
strategies.session = function(request)
    local cookies = request.headers and request.headers["Cookie"]
    if not cookies then return nil end
    
    local session_id = cookies:match("session_id=([^;]+)")
    if not session_id then return nil end
    
    -- Mock session lookup
    local mock_sessions = {
        ["sess_abc123"] = { user_id = 3, username = "charlie", role = "user" },
    }
    
    local session = mock_sessions[session_id]
    if not session then return nil, "Invalid session" end
    
    return {
        user_id  = session.user_id,
        username = session.username,
        role     = session.role,
        method   = "session",
    }, nil
end

-- Main middleware function
function AuthMiddleware.authenticate(request, options)
    options = options or {}
    local required = options.required ~= false
    
    -- ลอง strategies ตามลำดับ
    local strategy_order = options.strategies or { "bearer", "api_key", "session" }
    
    for _, strategy_name in ipairs(strategy_order) do
        local strategy = strategies[strategy_name]
        if strategy then
            local user, err = strategy(request)
            if user then
                request.user = user
                request.auth_method = strategy_name
                return true, nil
            elseif err then
                -- Strategy พยายามแต่ล้มเหลว
                if required then
                    return false, err
                end
            end
        end
    end
    
    if required then
        return false, "Authentication required"
    end
    
    return true, nil -- Optional auth - ยังผ่านได้
end

-- Authorization middleware
function AuthMiddleware.require_role(role)
    return function(request)
        if not request.user then
            return false, "Not authenticated"
        end
        if request.user.role ~= role and request.user.role ~= "admin" then
            return false, string.format("Required role: %s, got: %s", role, request.user.role)
        end
        return true, nil
    end
end

-- ทดสอบ
print("=== Auth Middleware ===")

local requests = {
    {
        name = "JWT Bearer",
        headers = { ["Authorization"] = "Bearer eyJ0.eyJzdWI.abc123" },
    },
    {
        name = "API Key",
        headers = { ["X-API-Key"] = "ak_live_abc" },
    },
    {
        name = "Session Cookie",
        headers = { ["Cookie"] = "session_id=sess_abc123; lang=th" },
    },
    {
        name = "No auth (required)",
        headers = {},
    },
}

for _, req in ipairs(requests) do
    local ok, err = AuthMiddleware.authenticate(req, { required = true })
    if ok and req.user then
        print(string.format("[%s] Authenticated via %s: user_id=%d",
            req.name, req.auth_method, req.user.user_id))
    else
        print(string.format("[%s] Failed: %s", req.name, err or "no user"))
    end
end

-- Test role check
local test_req = { headers = { ["X-API-Key"] = "ak_live_abc" } }
AuthMiddleware.authenticate(test_req)

local role_check = AuthMiddleware.require_role("admin")
local can, role_err = role_check(test_req)
print("\nAPI key user needs admin role:", can, role_err)
```

---

## ตัวอย่างที่ 18: Rate Limiting สำหรับ Auth

```lua
-- Rate limiting for authentication attempts
local AuthRateLimit = {}

local attempts = {} -- { key -> { count, window_start, blocked_until } }

-- Configuration
local limits = {
    login       = { max = 5,  window = 900,  block_for = 900 },  -- 5 ครั้งใน 15 นาที
    api         = { max = 100, window = 60,   block_for = 60 },   -- 100 ครั้งใน 1 นาที
    password_reset = { max = 3, window = 3600, block_for = 3600 }, -- 3 ครั้งใน 1 ชั่วโมง
}

function AuthRateLimit.check(key, limit_type)
    local limit = limits[limit_type]
    if not limit then return true, nil end
    
    local now = os.time()
    local record = attempts[key]
    
    if not record then
        record = { count = 0, window_start = now, blocked_until = 0 }
        attempts[key] = record
    end
    
    -- ตรวจสอบว่า blocked อยู่ไหม
    if now < record.blocked_until then
        local remaining = record.blocked_until - now
        return false, string.format("Too many attempts. Try again in %d seconds", remaining)
    end
    
    -- Reset window ถ้าหมดแล้ว
    if now - record.window_start >= limit.window then
        record.count = 0
        record.window_start = now
    end
    
    -- เพิ่ม count
    record.count = record.count + 1
    
    -- ตรวจสอบ limit
    if record.count > limit.max then
        record.blocked_until = now + limit.block_for
        return false, string.format("Rate limit exceeded. Blocked for %d seconds", limit.block_for)
    end
    
    local remaining_attempts = limit.max - record.count
    return true, nil, remaining_attempts
end

function AuthRateLimit.reset(key)
    attempts[key] = nil
end

-- Progressive delay
function AuthRateLimit.get_delay(key)
    local record = attempts[key]
    if not record then return 0 end
    
    local delays = { 0, 0, 1, 2, 5, 10, 30 }
    local idx = math.min(record.count, #delays)
    return delays[idx]
end

-- ทดสอบ
print("=== Auth Rate Limiting ===")

local test_ip = "192.168.1.1"
for i = 1, 7 do
    local ok, err, remaining = AuthRateLimit.check(test_ip, "login")
    if ok then
        print(string.format("Attempt %d: OK (remaining: %d, delay: %ds)",
            i, remaining or 0, AuthRateLimit.get_delay(test_ip)))
    else
        print(string.format("Attempt %d: BLOCKED - %s", i, err))
    end
end
```

---

## ตัวอย่างที่ 19: Secure Token Storage

```lua
-- Secure token storage patterns
local TokenStorage = {}

-- Encrypt/Decrypt (simplified - ใช้ AES ในระบบจริง)
local function simple_encrypt(data, key)
    local result = {}
    local key_len = #key
    for i = 1, #data do
        local b = string.byte(data, i)
        local k = string.byte(key, ((i - 1) % key_len) + 1)
        table.insert(result, string.format("%02x", bit32 and bit32.bxor(b, k) or ((b + k) % 256)))
    end
    return table.concat(result)
end

local function simple_decrypt(hex_data, key)
    local result = {}
    local key_len = #key
    local i = 0
    for hex in hex_data:gmatch("%x%x") do
        i = i + 1
        local b = tonumber(hex, 16)
        local k = string.byte(key, ((i - 1) % key_len) + 1)
        table.insert(result, string.char(bit32 and bit32.bxor(b, k) or ((b - k + 256) % 256)))
    end
    return table.concat(result)
end

-- Token vault
local vault = {}
local ENCRYPTION_KEY = "my-32-byte-encryption-key-here!!"

function TokenStorage.store(token_id, token_data, metadata)
    local serialized = string.format("%s|%s|%d",
        token_data.access_token or "",
        token_data.refresh_token or "",
        token_data.expires_at or 0
    )
    
    vault[token_id] = {
        encrypted = simple_encrypt(serialized, ENCRYPTION_KEY),
        metadata  = metadata or {},
        stored_at = os.time(),
    }
    return true
end

function TokenStorage.retrieve(token_id)
    local entry = vault[token_id]
    if not entry then return nil, "Token not found" end
    
    local decrypted = simple_decrypt(entry.encrypted, ENCRYPTION_KEY)
    local access, refresh, expires_str = decrypted:match("^([^|]*)|([^|]*)|(%d+)$")
    
    if not access then return nil, "Corrupted token data" end
    
    return {
        access_token  = access,
        refresh_token = refresh,
        expires_at    = tonumber(expires_str),
        metadata      = entry.metadata,
    }, nil
end

function TokenStorage.delete(token_id)
    vault[token_id] = nil
    return true
end

-- ทดสอบ
print("=== Token Storage ===")

local stored = TokenStorage.store("user_1_oauth_github", {
    access_token  = "gho_abc123xyz789",
    refresh_token = "ghr_def456uvw012",
    expires_at    = os.time() + 3600,
}, { provider = "github", user_id = 1 })

print("Stored:", stored)

local data, err = TokenStorage.retrieve("user_1_oauth_github")
if data then
    print("Access token:", data.access_token)
    print("Refresh token:", data.refresh_token)
    print("Provider:", data.metadata.provider)
else
    print("Error:", err)
end
```

---

## ตัวอย่างที่ 20: Complete Auth System Integration

```lua
-- Integration ระบบ auth ทั้งหมด
local AuthSystem = {}

-- Components
local users_db = {
    [1] = { username = "alice", email = "alice@example.com", role = "admin",
            password_hash = "$2b$12$mock_alice_hash", mfa_secret = "MFASECRETALICE",
            mfa_enabled = true, active = true },
    [2] = { username = "bob",   email = "bob@example.com",   role = "user",
            password_hash = "$2b$12$mock_bob_hash", mfa_secret = nil,
            mfa_enabled = false, active = true },
}

local active_sessions = {}
local token_blacklist = {}

function AuthSystem.login(credentials)
    -- Step 1: Find user
    local user = nil
    for id, u in pairs(users_db) do
        if u.username == credentials.username or u.email == credentials.username then
            user = u
            user.id = id
            break
        end
    end
    
    if not user then
        return nil, "Invalid username or password"
    end
    
    if not user.active then
        return nil, "Account is disabled"
    end
    
    -- Step 2: Verify password (mock)
    local mock_passwords = { alice = "AlicePass123!", bob = "BobPass456!" }
    if mock_passwords[user.username] ~= credentials.password then
        return nil, "Invalid username or password"
    end
    
    -- Step 3: MFA check (ถ้าเปิดใช้งาน)
    if user.mfa_enabled then
        if not credentials.mfa_code then
            return nil, "MFA_REQUIRED", { mfa_required = true, user_id = user.id }
        end
        -- Verify MFA code (mock)
        if credentials.mfa_code ~= "123456" then
            return nil, "Invalid MFA code"
        end
    end
    
    -- Step 4: สร้าง session/token
    local session_id = "sess_" .. tostring(os.time()) .. "_" .. tostring(user.id)
    active_sessions[session_id] = {
        user_id    = user.id,
        username   = user.username,
        role       = user.role,
        created_at = os.time(),
        expires_at = os.time() + 86400,
        ip         = credentials.ip,
    }
    
    return {
        session_id = session_id,
        user = {
            id       = user.id,
            username = user.username,
            email    = user.email,
            role     = user.role,
        },
        expires_in = 86400,
    }, nil
end

function AuthSystem.logout(session_id)
    if active_sessions[session_id] then
        token_blacklist[session_id] = os.time()
        active_sessions[session_id] = nil
        return true
    end
    return false
end

function AuthSystem.verify_session(session_id)
    if token_blacklist[session_id] then
        return nil, "Session revoked"
    end
    local session = active_sessions[session_id]
    if not session then return nil, "Session not found" end
    if os.time() > session.expires_at then
        active_sessions[session_id] = nil
        return nil, "Session expired"
    end
    return session, nil
end

function AuthSystem.check_permission(session_id, resource, action)
    local session, err = AuthSystem.verify_session(session_id)
    if not session then return false, err end
    
    local role_perms = {
        admin = { ["*"] = true },
        user  = { ["posts:read"] = true, ["profile:write"] = true },
    }
    
    local perms = role_perms[session.role] or {}
    local perm_key = resource .. ":" .. action
    
    if perms["*"] or perms[perm_key] then
        return true, nil
    end
    
    return false, "Insufficient permissions"
end

-- ทดสอบ
print("=== Complete Auth System ===")

-- Login สำเร็จ
local result, err = AuthSystem.login({ username = "alice", password = "AlicePass123!", mfa_code = "123456" })
if result then
    print("Login successful:", result.user.username, "| Session:", result.session_id:sub(1, 20))
    
    -- Check permissions
    local can, perm_err = AuthSystem.check_permission(result.session_id, "users", "delete")
    print("Admin can delete users:", can)
    
    -- Logout
    AuthSystem.logout(result.session_id)
    local _, sess_err = AuthSystem.verify_session(result.session_id)
    print("After logout:", sess_err)
else
    print("Login failed:", err)
end

-- Login ต้องการ MFA
local result2, err2, extra = AuthSystem.login({ username = "alice", password = "AlicePass123!" })
if extra and extra.mfa_required then
    print("\nMFA required for user ID:", extra.user_id)
end

-- Login ล้มเหลว
local _, err3 = AuthSystem.login({ username = "alice", password = "wrongpassword" })
print("Wrong password:", err3)
```

---

## สรุปบทที่ 56

ในบทนี้เราได้เรียนรู้:

1. **Authentication vs Authorization** - ความแตกต่างและการทำงานร่วมกัน
2. **Session-based auth** - การสร้าง จัดการ และ cleanup sessions
3. **JWT** - โครงสร้าง header/payload/signature, การสร้างและตรวจสอบ
4. **Refresh tokens** - token rotation และการจัดการวงจรชีวิต
5. **OAuth2** - Authorization Code Flow แบบครบวงจร
6. **API Key auth** - การสร้าง validate และ rate limiting
7. **Basic auth** - HTTP Basic Authentication
8. **Password hashing** - bcrypt concepts และ strength checking
9. **RBAC** - Role hierarchy และ permission inheritance
10. **Permission system** - Fine-grained control พร้อม conditions
11. **Token revocation** - Blacklist และ user-level revocation
12. **MFA** - TOTP และ SMS OTP
13. **Secure cookies** - Flags, signing, parsing
14. **Session management** - Rolling sessions, per-user limits
15. **Auth middleware** - Multi-strategy authentication
16. **Rate limiting** - Progressive delays, blocking
17. **Token storage** - Encrypted vault pattern
18. **Integration** - ระบบครบวงจรรวม login flow

> **สำคัญ**: ในระบบ production ต้องใช้ library ที่ผ่านการ audit แล้ว เช่น `lua-resty-jwt`, `lua-bcrypt`, และ `lua-resty-openssl` แทนการ implement เอง
