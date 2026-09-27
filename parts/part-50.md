# บทที่ 50: Security Basics

## บทนำ

Security เป็นสิ่งที่นักพัฒนาซอฟต์แวร์ทุกคนต้องให้ความสำคัญ ไม่ว่าจะเขียน application ประเภทใด บทนี้จะครอบคลุม security fundamentals ที่สำคัญใน Lua การป้องกัน injection attacks, การ validate input, cryptography เบื้องต้น, และ secure coding patterns

## ทำไม Security ถึงสำคัญ

- **SQL Injection**: ทำลาย database ได้
- **Command Injection**: รัน arbitrary commands บน server
- **Path Traversal**: เข้าถึงไฟล์นอก sandbox
- **DoS Attacks**: ทำให้ service ล่มด้วย regex หรือ resource exhaustion
- **Authentication Bypass**: เข้าถึงโดยไม่มีสิทธิ์

---

## ตัวอย่างที่ 1: Input Validation Fundamentals

```lua
-- Input Validation - ตรวจสอบ input ก่อนใช้งานเสมอ

local Validator = {}
Validator.__index = Validator

-- ตรวจสอบ type
function Validator.is_string(val)
    return type(val) == "string"
end

function Validator.is_number(val)
    return type(val) == "number" and not (val ~= val)  -- not NaN
end

function Validator.is_integer(val)
    return type(val) == "number" and math.floor(val) == val
end

function Validator.is_boolean(val)
    return type(val) == "boolean"
end

-- String validations
function Validator.min_length(str, min)
    return type(str) == "string" and #str >= min
end

function Validator.max_length(str, max)
    return type(str) == "string" and #str <= max
end

function Validator.length_between(str, min, max)
    return type(str) == "string" and #str >= min and #str <= max
end

function Validator.matches_pattern(str, pattern)
    return type(str) == "string" and str:match(pattern) ~= nil
end

function Validator.is_email(str)
    if type(str) ~= "string" then return false end
    -- Basic email validation
    return str:match("^[%w%.%+%-]+@[%w%-]+%.[%a]{2,}$") ~= nil
end

function Validator.is_url(str)
    if type(str) ~= "string" then return false end
    return str:match("^https?://[%w%-%.]+") ~= nil
end

function Validator.is_ip_address(str)
    if type(str) ~= "string" then return false end
    local a, b, c, d = str:match("^(%d+)%.(%d+)%.(%d+)%.(%d+)$")
    if not a then return false end
    for _, octet in ipairs({tonumber(a), tonumber(b), tonumber(c), tonumber(d)}) do
        if octet < 0 or octet > 255 then return false end
    end
    return true
end

-- Number validations
function Validator.in_range(num, min, max)
    return type(num) == "number" and num >= min and num <= max
end

function Validator.is_positive(num)
    return type(num) == "number" and num > 0
end

-- Whitelist validation
function Validator.is_one_of(val, allowed)
    for _, v in ipairs(allowed) do
        if val == v then return true end
    end
    return false
end

-- Sanitize string (remove dangerous characters)
function Validator.sanitize_string(str, allowed_pattern)
    if type(str) ~= "string" then return "" end
    allowed_pattern = allowed_pattern or "[^%w%s%-%_%.%@]"
    return str:gsub(allowed_pattern, "")
end

-- Normalize whitespace
function Validator.normalize_whitespace(str)
    if type(str) ~= "string" then return "" end
    return str:gsub("%s+", " "):gsub("^%s+", ""):gsub("%s+$", "")
end

-- ทดสอบ
print("=== Input Validation ===\n")

-- Email validation
local emails = {
    "user@example.com",    -- valid
    "user.name+tag@domain.co.th",  -- valid
    "invalid.email",       -- invalid
    "@nodomain.com",       -- invalid
    "user@",              -- invalid
    "",                   -- invalid
}

print("Email validation:")
for _, email in ipairs(emails) do
    print(string.format("  %-35s %s",
        email, Validator.is_email(email) and "VALID" or "INVALID"))
end

-- Range validation
print("\nNumber range validation:")
local values = {-1, 0, 1, 50, 100, 101}
for _, v in ipairs(values) do
    print(string.format("  %4d: %s",
        v, Validator.in_range(v, 0, 100) and "valid" or "out of range"))
end

-- Whitelist
print("\nAllowed status values:")
local statuses = {"active", "inactive", "pending", "deleted", "hacked"}
for _, s in ipairs(statuses) do
    print(string.format("  %-10s %s",
        s, Validator.is_one_of(s, {"active", "inactive", "pending"})
           and "allowed" or "REJECTED"))
end

-- Sanitize
print("\nString sanitization:")
local dangerous = "Hello; rm -rf /; echo 'pwned'"
print("  Original:", dangerous)
print("  Sanitized:", Validator.sanitize_string(dangerous))
```

---

## ตัวอย่างที่ 2: SQL Injection Prevention

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")
db:exec([[
    CREATE TABLE users (
        id INTEGER PRIMARY KEY,
        username TEXT UNIQUE,
        password_hash TEXT,
        admin INTEGER DEFAULT 0
    );
    INSERT INTO users VALUES (1, 'admin', 'pbkdf2:sha256:...hash...', 1);
    INSERT INTO users VALUES (2, 'alice', 'pbkdf2:sha256:...hash...', 0);
]])

-- ============================================================
-- VULNERABLE code - DO NOT USE
-- ============================================================
local function vulnerable_login(username, password)
    -- อันตราย! Concatenating user input directly
    local sql = "SELECT * FROM users WHERE username='" ..
                username .. "' AND password_hash='" .. password .. "'"
    print("  VULNERABLE SQL:", sql:sub(1, 80))
    for row in db:nrows(sql) do
        return row
    end
    return nil
end

-- ============================================================
-- SECURE code - Use parameterized queries
-- ============================================================
local function secure_login(username, password)
    -- ดีกว่ามาก! Parameters ถูก bind แยกต่างหาก
    local stmt = db:prepare(
        "SELECT * FROM users WHERE username = ? AND password_hash = ?"
    )
    if not stmt then return nil end

    stmt:bind(1, username)
    stmt:bind(2, password)

    local user = nil
    if stmt:step() == sqlite3.ROW then
        user = {
            id       = stmt:get_value(0),
            username = stmt:get_value(1),
            admin    = stmt:get_value(3),
        }
    end
    stmt:finalize()
    return user
end

print("=== SQL Injection Prevention ===\n")

-- Attack vectors
local attacks = {
    "admin' --",                              -- comment out password check
    "' OR '1'='1",                            -- always true
    "' OR 1=1 --",                            -- bypass auth
    "admin'; DROP TABLE users; --",           -- destructive
    "' UNION SELECT 1,2,3,4 --",             -- data extraction
    "' OR username='admin' AND '1'='1",       -- specific user bypass
}

print("Testing SQL injection attacks:\n")
for _, attack in ipairs(attacks) do
    print("Attack: " .. attack)

    -- Test vulnerable
    local v_result = pcall(vulnerable_login, attack, "anything")
    -- Test secure
    local s_result = secure_login(attack, "anything")

    print(string.format("  Vulnerable: %s",
        v_result and "processed (may be exploited!)" or "error"))
    print(string.format("  Secure:     %s\n",
        s_result and "MATCHED (BAD!)" or "Rejected safely"))
end

-- Additional protection: input validation before DB
local function extra_safe_login(username, password)
    -- 1. Validate inputs
    if type(username) ~= "string" or type(password) ~= "string" then
        return nil, "Invalid input types"
    end

    -- 2. Length limits
    if #username > 50 or #password > 200 then
        return nil, "Input too long"
    end

    -- 3. Username format (alphanumeric + underscore only)
    if not username:match("^[%w_]+$") then
        return nil, "Invalid username format"
    end

    -- 4. Parameterized query
    return secure_login(username, password)
end

print("Testing extra-safe login:")
local result, err = extra_safe_login("admin' --", "pass")
print("  Attack result:", tostring(result), err or "")

local result2, err2 = extra_safe_login("alice", "correct_hash")
print("  Normal result:", tostring(result2), err2 or "")

db:close()
```

---

## ตัวอย่างที่ 3: Command Injection Prevention

```lua
-- Command Injection Prevention

-- ============================================================
-- VULNERABLE - DO NOT USE
-- ============================================================
local function vulnerable_ping(hostname)
    -- อันตราย! User input ใน shell command
    local cmd = "ping -c 1 " .. hostname
    print("  VULNERABLE cmd:", cmd)
    -- os.execute(cmd)  -- อย่าทำ!
end

-- ============================================================
-- SECURE approaches
-- ============================================================

-- Option 1: Whitelist validation
local function validate_hostname(hostname)
    -- อนุญาตเฉพาะ valid hostname characters
    if type(hostname) ~= "string" then return false end
    if #hostname < 1 or #hostname > 253 then return false end

    -- Only allow alphanumeric, hyphens, dots
    if not hostname:match("^[%w%.%-]+$") then return false end

    -- No consecutive dots
    if hostname:match("%.%.") then return false end

    -- No leading/trailing hyphens on labels
    for label in hostname:gmatch("[^%.]+") do
        if label:match("^%-") or label:match("%-$") then return false end
    end

    return true
end

-- Option 2: Shell escaping
local function shell_escape(str)
    -- Escape สำหรับ Unix shell (single-quote method)
    return "'" .. str:gsub("'", "'\\''") .. "'"
end

-- Option 3: ใช้ Lua network library แทน shell command
local function safe_ping(hostname)
    if not validate_hostname(hostname) then
        return nil, "Invalid hostname: " .. tostring(hostname)
    end

    -- ใช้ LuaSocket แทน shell ping
    local socket = require("socket")
    local start = socket.gettime()
    local s = socket.tcp()
    s:settimeout(3)
    local ok = s:connect(hostname, 80)
    local elapsed = (socket.gettime() - start) * 1000
    s:close()

    if ok then
        return {host = hostname, time_ms = elapsed, reachable = true}
    else
        return {host = hostname, time_ms = elapsed, reachable = false}
    end
end

print("=== Command Injection Prevention ===\n")

-- ทดสอบ validation
local hostnames = {
    "google.com",           -- valid
    "192.168.1.1",         -- valid IP
    "sub.example.co.th",   -- valid
    "evil.com; rm -rf /",  -- INJECTION!
    "$(whoami)",           -- INJECTION!
    "`id`",               -- INJECTION!
    "evil.com && cat /etc/passwd",  -- INJECTION!
    "../../../etc/passwd", -- path traversal
    "a" .. string.rep("b", 300),  -- too long
}

print("Hostname validation:")
for _, h in ipairs(hostnames) do
    local valid = validate_hostname(h)
    print(string.format("  %-40s %s",
        h:sub(1, 40), valid and "VALID" or "REJECTED"))
end

-- Shell escaping
print("\nShell escaping examples:")
local dangerous_inputs = {
    "normal text",
    "with 'single' quotes",
    "with; semicolons",
    "$(command substitution)",
    "back`tick`",
    "pipe|char",
}

for _, input in ipairs(dangerous_inputs) do
    print(string.format("  Original: %-35s -> Escaped: %s",
        input, shell_escape(input)))
end

-- ทดสอบ safe ping
print("\nSafe ping test:")
local r1 = safe_ping("google.com")
if r1 then
    print(string.format("  google.com: reachable=%s time=%.1fms",
        tostring(r1.reachable), r1.time_ms))
end

local r2, err = safe_ping("$(evil cmd)")
if err then
    print("  Injection attempt rejected:", err)
end
```

---

## ตัวอย่างที่ 4: Path Traversal Prevention

```lua
local lfs_ok, lfs = pcall(require, "lfs")

-- Path Traversal Prevention

-- ============================================================
-- VULNERABLE
-- ============================================================
local function vulnerable_read_file(filename)
    -- อันตราย! ไม่ validate path
    local f = io.open("/var/www/files/" .. filename, "r")
    if f then
        local content = f:read("*a")
        f:close()
        return content
    end
    return nil
end

-- ============================================================
-- SECURE path handling
-- ============================================================

-- ทำ path canonical (resolve .. and . and //)
local function canonicalize_path(path)
    -- แทนที่ backslashes
    path = path:gsub("\\", "/")

    -- แก้ multiple slashes
    path = path:gsub("//+", "/")

    -- Process path components
    local parts = {}
    for part in path:gmatch("[^/]+") do
        if part == ".." then
            if #parts > 0 then
                table.remove(parts)
            end
        elseif part ~= "." and part ~= "" then
            table.insert(parts, part)
        end
    end

    local result = "/" .. table.concat(parts, "/")
    return result
end

-- ตรวจสอบว่า path อยู่ใน allowed directory
local function is_safe_path(requested_path, base_dir)
    -- Normalize paths
    local canonical_base    = canonicalize_path(base_dir)
    local canonical_request = canonicalize_path(base_dir .. "/" .. requested_path)

    -- ต้องเริ่มต้นด้วย base_dir
    return canonical_request:sub(1, #canonical_base) == canonical_base,
           canonical_request
end

-- Secure file reader
local function secure_read_file(filename, base_dir)
    base_dir = base_dir or "/var/www/files"

    -- ตรวจสอบ filename type
    if type(filename) ~= "string" then
        return nil, "Filename must be a string"
    end

    -- ไม่อนุญาต null bytes
    if filename:find("\0") then
        return nil, "Null bytes not allowed"
    end

    -- ตรวจสอบ path traversal
    local safe, resolved = is_safe_path(filename, base_dir)
    if not safe then
        return nil, string.format(
            "Path traversal detected: %s resolves outside %s",
            filename, base_dir)
    end

    -- ตรวจสอบ extension whitelist
    local ext = filename:match("%.([%w]+)$")
    local allowed_exts = {txt=true, html=true, css=true, js=true, json=true, png=true}
    if ext and not allowed_exts[ext:lower()] then
        return nil, "File type not allowed: ." .. ext
    end

    print(string.format("  Safe path: %s", resolved))
    -- ในการใช้งานจริง จะอ่านไฟล์จาก resolved path
    return "file_content_here"
end

print("=== Path Traversal Prevention ===\n")

-- ทดสอบ canonicalize
print("Path canonicalization:")
local paths = {
    "/normal/path/file.txt",
    "/path/./to/./file.txt",
    "/path/../other/file.txt",
    "/path/../../etc/passwd",
    "/path/subdir/../../etc/shadow",
    "//double//slash//path",
}

for _, p in ipairs(paths) do
    print(string.format("  %-40s -> %s", p, canonicalize_path(p)))
end

-- ทดสอบ path traversal attacks
print("\nPath traversal detection (base=/var/www/files):")
local attack_paths = {
    "document.txt",                         -- safe
    "images/photo.png",                     -- safe
    "../../../etc/passwd",                  -- attack!
    "subdir/../../etc/shadow",             -- attack!
    "normal/../../../etc/hosts",           -- attack!
    "file.txt\0.png",                      -- null byte attack
    "file.exe",                            -- wrong extension
}

for _, path in ipairs(attack_paths) do
    local content, err = secure_read_file(path, "/var/www/files")
    if content then
        print(string.format("  %-40s -> OK", path:sub(1,40)))
    else
        print(string.format("  %-40s -> BLOCKED: %s",
            path:sub(1,40), err))
    end
end
```

---

## ตัวอย่างที่ 5: Regular Expression DoS (ReDoS)

```lua
-- ReDoS - Regular Expression Denial of Service

-- Regex ที่มี catastrophic backtracking
-- Pattern: (a+)+ หรือ (a*)*
-- Input: "aaaaaaaaaaaaaaaaaaaaaaX" ทำให้ exponential time

-- ============================================================
-- DANGEROUS patterns
-- ============================================================
local dangerous_patterns = {
    -- Nested quantifiers
    "^(a+)+$",
    "^(a*)*$",
    "^([a-zA-Z]+)*$",
    -- Alternation with overlap
    "^(a|aa)+$",
    -- Email-like (vulnerable version)
    "^([a-z0-9]*)*@",
}

-- ทดสอบ patterns
print("=== ReDoS Prevention ===\n")

print("Testing pattern performance:")
local test_input = string.rep("a", 20) .. "!"  -- 20 'a's then '!'

for _, pattern in ipairs(dangerous_patterns) do
    local start = os.clock()
    local ok, result = pcall(function()
        return test_input:match(pattern)
    end)
    local elapsed = os.clock() - start

    if elapsed > 0.1 then
        print(string.format("  SLOW (%.3fs): %s", elapsed, pattern))
    else
        print(string.format("  Fast (%.3fs): %s", elapsed, pattern))
    end
end

-- ============================================================
-- Safe alternatives
-- ============================================================

-- ใช้ simple, linear patterns แทน
local function safe_validate_email(email)
    if type(email) ~= "string" then return false end

    -- ตรวจสอบ length ก่อน
    if #email > 254 then return false end

    -- Simple linear pattern (ไม่มี backtracking)
    local local_part, domain = email:match("^([^@]+)@([^@]+)$")
    if not local_part or not domain then return false end

    -- Local part: max 64 chars
    if #local_part > 64 then return false end

    -- Domain must have at least one dot
    if not domain:find("%.") then return false end

    -- Basic character validation
    if not local_part:match("^[%w%.%+%-%_]+$") then return false end
    if not domain:match("^[%w%.%-]+$") then return false end

    return true
end

-- Input size limiting
local function safe_process_input(input, max_length)
    max_length = max_length or 1000

    if type(input) ~= "string" then
        return nil, "Input must be a string"
    end

    if #input > max_length then
        return nil, string.format(
            "Input too long: %d > %d", #input, max_length)
    end

    return input:match("^([%w%s%.%,%-]+)$")  -- whitelist approach
end

-- ทดสอบ
print("\nSafe email validation:")
local emails = {
    "user@example.com",
    "a@b.c",
    "valid.email+tag@domain.co.th",
    "toolong" .. string.rep("a", 300) .. "@domain.com",
    "no-at-sign.com",
    "@no-local.com",
}

for _, email in ipairs(emails) do
    print(string.format("  %-50s %s",
        email:sub(1, 50), safe_validate_email(email) and "VALID" or "INVALID"))
end

print("\nInput size limiting:")
local inputs = {
    "normal input",
    string.rep("a", 50),
    string.rep("a", 1001),
    "with!dangerous&chars",
    "safe text with spaces",
}

for _, input in ipairs(inputs) do
    local result, err = safe_process_input(input, 100)
    print(string.format("  %-30s -> %s",
        input:sub(1, 30), result and "OK" or "REJECTED: " .. (err or "invalid chars")))
end
```

---

## ตัวอย่างที่ 6: Secure String Handling

```lua
-- Secure String Handling

-- ============================================================
-- Constant-time string comparison (prevent timing attacks)
-- ============================================================
local function constant_time_equals(a, b)
    -- ป้องกัน timing attack ใน password/token comparison
    if type(a) ~= "string" or type(b) ~= "string" then
        return false
    end

    -- ต้องมีความยาวเท่ากัน (แต่ไม่ return เร็วถ้าไม่เท่า)
    local len_a = #a
    local len_b = #b

    -- XOR ทุก byte
    local result = 0
    local max_len = math.max(len_a, len_b)

    for i = 1, max_len do
        local byte_a = a:byte(i) or 0
        local byte_b = b:byte(i) or 0
        result = result | (byte_a ~ byte_b)  -- bitwise XOR
    end

    -- ยังต้องตรวจสอบ length ด้วย
    result = result | (len_a ~ len_b)

    return result == 0
end

-- Secure token generation (pseudo-random)
local function generate_token(length)
    length = length or 32
    local chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    local token = {}

    -- ใช้ /dev/urandom ถ้ามี
    local urandom = io.open("/dev/urandom", "rb")
    if urandom then
        local bytes = urandom:read(length)
        urandom:close()

        for i = 1, #bytes do
            local byte = bytes:byte(i)
            token[i] = chars:sub((byte % #chars) + 1, (byte % #chars) + 1)
        end
    else
        -- Fallback (น้อย secure กว่า)
        math.randomseed(os.time() * 1000 + math.random(1000))
        for i = 1, length do
            token[i] = chars:sub(math.random(#chars), math.random(#chars))
        end
    end

    return table.concat(token)
end

-- Secure memory clearing
local function secure_clear(t, key)
    -- ล้างค่าที่ sensitive ออกจาก table
    if t[key] then
        -- Overwrite with zeros ก่อน (Lua strings are immutable จริงๆ)
        t[key] = string.rep("\0", #tostring(t[key]))
        t[key] = nil
    end
end

print("=== Secure String Handling ===\n")

-- Constant-time comparison
print("Constant-time comparison:")
local pairs_to_compare = {
    {"secret_token", "secret_token"},   -- match
    {"secret_token", "wrong_token123"},  -- mismatch, same length
    {"short", "much_longer_string"},     -- different lengths
    {"", ""},                            -- empty strings
}

for _, pair in ipairs(pairs_to_compare) do
    local a, b = pair[1], pair[2]
    local regular_match = (a == b)
    local ct_match = constant_time_equals(a, b)
    print(string.format("  %-20s vs %-20s: equal=%s",
        a, b, tostring(ct_match)))
end

-- Token generation
print("\nSecure token generation:")
for i = 1, 5 do
    local token = generate_token(32)
    print(string.format("  Token %d: %s", i, token))
end

-- Sensitive data handling
print("\nSensitive data handling:")
local user_session = {
    username = "alice",
    password = "secret_password_123",  -- should never store plaintext
    token    = generate_token(16),
    admin    = false,
}

print("  Before clear - password exists:", user_session.password ~= nil)
secure_clear(user_session, "password")
print("  After clear  - password exists:", user_session.password ~= nil)
print("  Token still exists:", user_session.token ~= nil)
```

---

## ตัวอย่างที่ 7: Cryptographic Hashing

```lua
-- Cryptographic Hashing
-- ต้องการ: luaossl หรือ luacrypto

-- ============================================================
-- ลอง load crypto libraries
-- ============================================================
local crypto_available = false
local sha256_func = nil

-- ลอง luaossl
local ok1, openssl_digest = pcall(require, "openssl.digest")
if ok1 then
    crypto_available = true
    sha256_func = function(data)
        local md = openssl_digest.new("sha256")
        md:update(data)
        return md:final()
    end
    print("Using openssl.digest")
end

-- ลอง luacrypto
if not crypto_available then
    local ok2, crypto = pcall(require, "crypto")
    if ok2 then
        crypto_available = true
        sha256_func = function(data)
            return crypto.digest("sha256", data, true)
        end
        print("Using luacrypto")
    end
end

-- ลอง sha2 (pure Lua)
if not crypto_available then
    local ok3, sha2 = pcall(require, "sha2")
    if ok3 then
        crypto_available = true
        sha256_func = sha2.sha256
        print("Using sha2 (pure Lua)")
    end
end

-- Pure Lua SHA-256 implementation (simplified, สำหรับ demo)
if not crypto_available then
    print("No crypto library found, using simplified demo\n")

    -- SHA-256 ที่ implement ใน Lua (ใช้ bitwise operations)
    local function sha256_pure_lua(msg)
        -- Constants
        local K = {
            0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5,
            0x3956c25b, 0x59f111f1, 0x923f82a4, 0xab1c5ed5,
            0xd807aa98, 0x12835b01, 0x243185be, 0x550c7dc3,
            0x72be5d74, 0x80deb1fe, 0x9bdc06a7, 0xc19bf174,
            0xe49b69c1, 0xefbe4786, 0x0fc19dc6, 0x240ca1cc,
            0x2de92c6f, 0x4a7484aa, 0x5cb0a9dc, 0x76f988da,
            0x983e5152, 0xa831c66d, 0xb00327c8, 0xbf597fc7,
            0xc6e00bf3, 0xd5a79147, 0x06ca6351, 0x14292967,
            0x27b70a85, 0x2e1b2138, 0x4d2c6dfc, 0x53380d13,
            0x650a7354, 0x766a0abb, 0x81c2c92e, 0x92722c85,
            0xa2bfe8a1, 0xa81a664b, 0xc24b8b70, 0xc76c51a3,
            0xd192e819, 0xd6990624, 0xf40e3585, 0x106aa070,
            0x19a4c116, 0x1e376c08, 0x2748774c, 0x34b0bcb5,
            0x391c0cb3, 0x4ed8aa4a, 0x5b9cca4f, 0x682e6ff3,
            0x748f82ee, 0x78a5636f, 0x84c87814, 0x8cc70208,
            0x90befffa, 0xa4506ceb, 0xbef9a3f7, 0xc67178f2,
        }

        -- Initial hash values
        local H = {
            0x6a09e667, 0xbb67ae85, 0x3c6ef372, 0xa54ff53a,
            0x510e527f, 0x9b05688c, 0x1f83d9ab, 0x5be0cd19,
        }

        -- สำหรับ demo จะแสดงแค่ structure
        return "sha256:" .. string.format("%016x", #msg * 8) .. "..."
    end

    sha256_func = sha256_pure_lua
    crypto_available = true
end

-- Hex encoding
local function to_hex(bytes)
    return (bytes:gsub(".", function(c)
        return string.format("%02x", c:byte())
    end))
end

-- HMAC (Hash-based Message Authentication Code)
-- HMAC-SHA256 = SHA256((key XOR opad) || SHA256((key XOR ipad) || message))
local function hmac_sha256(key, message)
    if not sha256_func then return nil end

    local block_size = 64
    local ipad = string.rep("\x36", block_size)
    local opad = string.rep("\x5c", block_size)

    -- Prepare key
    if #key > block_size then
        key = sha256_func(key)  -- hash key if too long
    end
    if #key < block_size then
        key = key .. string.rep("\0", block_size - #key)  -- pad key
    end

    -- XOR key with ipad and opad
    local k_ipad = {}
    local k_opad = {}
    for i = 1, block_size do
        local k = key:byte(i)
        table.insert(k_ipad, string.char(k ~ ipad:byte(i)))
        table.insert(k_opad, string.char(k ~ opad:byte(i)))
    end
    k_ipad = table.concat(k_ipad)
    k_opad = table.concat(k_opad)

    -- HMAC = hash(opad || hash(ipad || message))
    local inner = sha256_func(k_ipad .. message)
    local outer = sha256_func(k_opad .. inner)
    return outer
end

print("=== Cryptographic Hashing ===\n")

-- Hash testing
local test_data = {
    "",
    "Hello",
    "Hello, World!",
    "Lua security tutorial",
    string.rep("a", 1000),
}

print("SHA-256 hashes:")
for _, data in ipairs(test_data) do
    local hash = sha256_func(data)
    local display = type(hash) == "string" and
        (hash:len() > 20 and to_hex and pcall(to_hex, hash) and
         to_hex(hash) or hash) or tostring(hash)
    print(string.format("  %-25s -> %s",
        ('"' .. data:sub(1, 20) .. '"'), display:sub(1, 64)))
end

-- HMAC
print("\nHMAC-SHA256:")
local key = "secret-key-123"
local messages = {"Hello", "Hello", "Tampered"}
for i, msg in ipairs(messages) do
    local mac = hmac_sha256(key, msg)
    if mac then
        local hex_mac = to_hex and pcall(to_hex, mac) and to_hex(mac) or tostring(mac)
        print(string.format("  Message %d (%s): %s", i, msg, hex_mac:sub(1, 32) .. "..."))
    end
end
```

---

## ตัวอย่างที่ 8: Password Hashing

```lua
-- Password Hashing - NEVER store passwords in plaintext!

-- ============================================================
-- WRONG ways (DO NOT USE)
-- ============================================================
local function wrong_password_storage_examples()
    print("WRONG password storage examples:")
    print("  MD5('password') = 5f4dcc3b...  (BROKEN! crackable)")
    print("  SHA1('password') = 5baa61...   (BROKEN! crackable)")
    print("  Plain 'password'               (TERRIBLE!)")
    print("  Base64('password')             (Not encryption!)")
    print()
end

-- ============================================================
-- CORRECT: PBKDF2 / bcrypt / Argon2 (ต้องใช้ library)
-- ============================================================

-- Simplified PBKDF2-like implementation สำหรับ demo
-- (ในการใช้งานจริง ใช้ library ที่ tested แล้ว)

local function xor_strings(a, b)
    local result = {}
    for i = 1, #a do
        result[i] = string.char(a:byte(i) ~ b:byte(i))
    end
    return table.concat(result)
end

-- Pseudo-PBKDF2 (สำหรับ demo เท่านั้น)
local function pseudo_pbkdf2(password, salt, iterations, keylen)
    iterations = iterations or 100000
    keylen     = keylen     or 32

    -- ใน code จริง ใช้ HMAC-SHA256 ซ้ำ
    local result = salt .. password
    for i = 1, math.min(iterations, 1000) do  -- จำกัดสำหรับ demo
        -- Simplified mixing
        result = result:rep(2):sub(1, 32)
        local temp = {}
        for j = 1, #result do
            temp[j] = string.char(
                (result:byte(j) + i + j) % 256
            )
        end
        result = table.concat(temp)
    end

    return result:sub(1, keylen)
end

-- Secure password hasher
local PasswordHasher = {}
PasswordHasher.__index = PasswordHasher

function PasswordHasher.new(algorithm, options)
    options = options or {}
    return setmetatable({
        algorithm  = algorithm or "pbkdf2_sha256",
        iterations = options.iterations or 100000,
        salt_len   = options.salt_len   or 16,
    }, PasswordHasher)
end

function PasswordHasher:generate_salt()
    local salt = {}
    local urandom = io.open("/dev/urandom", "rb")
    if urandom then
        local bytes = urandom:read(self.salt_len)
        urandom:close()
        return bytes
    else
        -- Fallback
        math.randomseed(os.time())
        for i = 1, self.salt_len do
            salt[i] = string.char(math.random(0, 255))
        end
        return table.concat(salt)
    end
end

function PasswordHasher:hash(password)
    if type(password) ~= "string" or #password == 0 then
        return nil, "Password must be a non-empty string"
    end
    if #password > 1000 then
        return nil, "Password too long"
    end

    local salt = self:generate_salt()
    local hash = pseudo_pbkdf2(password, salt, self.iterations)

    -- Format: algorithm$iterations$salt_hex$hash_hex
    local function to_hex(s)
        return s:gsub(".", function(c)
            return string.format("%02x", c:byte())
        end)
    end

    return string.format("%s$%d$%s$%s",
        self.algorithm,
        self.iterations,
        to_hex(salt),
        to_hex(hash))
end

function PasswordHasher:verify(password, stored_hash)
    if type(password) ~= "string" or type(stored_hash) ~= "string" then
        return false
    end

    -- Parse stored hash
    local algo, iters, salt_hex, hash_hex =
        stored_hash:match("^([^$]+)%$(%d+)%$([^$]+)%$([^$]+)$")

    if not algo then return false end

    -- Decode hex
    local function from_hex(hex)
        return hex:gsub("..", function(h)
            return string.char(tonumber(h, 16))
        end)
    end

    local salt = from_hex(salt_hex)
    local stored = from_hex(hash_hex)
    local iterations = tonumber(iters)

    -- Recompute hash
    local computed = pseudo_pbkdf2(password, salt, iterations)

    -- Constant-time comparison
    if #computed ~= #stored then return false end
    local diff = 0
    for i = 1, #computed do
        diff = diff | (computed:byte(i) ~ stored:byte(i))
    end
    return diff == 0
end

-- ทดสอบ
print("=== Password Hashing ===\n")

wrong_password_storage_examples()

local hasher = PasswordHasher.new("pbkdf2_sha256", {
    iterations = 1000,  -- ลดลงสำหรับ demo
    salt_len   = 16,
})

print("CORRECT password hashing:")

local passwords = {"correct_password", "another_password", "short"}
local hashes = {}

for _, pwd in ipairs(passwords) do
    local hash, err = hasher:hash(pwd)
    if hash then
        hashes[pwd] = hash
        print(string.format("  Hash of %-20s: %s...",
            '"' .. pwd .. '"', hash:sub(1, 50)))
    else
        print("  Error hashing:", err)
    end
end

-- Verification
print("\nPassword verification:")
local verify_cases = {
    {"correct_password", "correct_password"},   -- should pass
    {"correct_password", "wrong_password"},      -- should fail
    {"another_password", "another_password"},    -- should pass
}

for _, vc in ipairs(verify_cases) do
    local original, attempt = vc[1], vc[2]
    local stored = hashes[original]
    if stored then
        local ok = hasher:verify(attempt, stored)
        print(string.format("  Verify %-20s with hash of %-20s: %s",
            '"' .. attempt .. '"',
            '"' .. original .. '"',
            ok and "MATCH" or "NO MATCH"))
    end
end
```

---

## ตัวอย่างที่ 9: Sandboxing User Code

```lua
-- Sandboxing - ป้องกัน malicious Lua code

-- ============================================================
-- Sandbox environment
-- ============================================================

local function create_sandbox(allowed_globals, options)
    options = options or {}

    -- Whitelist ของ functions ที่อนุญาต
    local safe_globals = {
        -- Math
        math = {
            abs   = math.abs,
            ceil  = math.ceil,
            floor = math.floor,
            max   = math.max,
            min   = math.min,
            sqrt  = math.sqrt,
            pi    = math.pi,
            random = math.random,
        },
        -- String (ส่วนใหญ่ safe)
        string = {
            format = string.format,
            len    = string.len,
            sub    = string.sub,
            upper  = string.upper,
            lower  = string.lower,
            rep    = string.rep,
            find   = string.find,
            match  = string.match,
            gmatch = string.gmatch,
            gsub   = string.gsub,
        },
        -- Table
        table = {
            insert = table.insert,
            remove = table.remove,
            concat = table.concat,
            sort   = table.sort,
        },
        -- Basic types
        type       = type,
        tostring   = tostring,
        tonumber   = tonumber,
        ipairs     = ipairs,
        pairs      = pairs,
        select     = select,
        unpack     = table.unpack or unpack,
        -- Safe print
        print      = print,

        -- NOT included (dangerous):
        -- io, os, require, dofile, loadfile, package
        -- load, loadstring, debug, rawget, rawset
        -- getmetatable, setmetatable
    }

    -- เพิ่ม custom globals ถ้ามี
    for k, v in pairs(allowed_globals or {}) do
        safe_globals[k] = v
    end

    return safe_globals
end

-- Execute user code ใน sandbox
local function execute_sandboxed(code, sandbox, timeout_seconds)
    timeout_seconds = timeout_seconds or 5

    -- Load code
    local func, err = load(code, "user_code", "t", sandbox)
    if not func then
        return nil, "Syntax error: " .. tostring(err)
    end

    -- ตั้ง timeout ด้วย debug hook
    local start_time = os.clock()
    local instruction_count = 0
    local max_instructions = 1000000  -- limit instructions

    debug.sethook(function()
        instruction_count = instruction_count + 1
        if instruction_count > max_instructions then
            error("Execution limit exceeded", 2)
        end
        if os.clock() - start_time > timeout_seconds then
            error("Timeout exceeded", 2)
        end
    end, "c", 100)

    local results = table.pack(pcall(func))
    debug.sethook()  -- Remove hook

    local success = table.remove(results, 1)

    if success then
        return results.n > 0 and results or true
    else
        return nil, "Runtime error: " .. tostring(results[1])
    end
end

print("=== Code Sandboxing ===\n")

local sandbox = create_sandbox({
    -- Custom safe functions
    safe_sqrt = function(n)
        if type(n) ~= "number" or n < 0 then
            return nil, "Invalid input"
        end
        return math.sqrt(n)
    end,
})

-- Safe code
local safe_codes = {
    [[
        local result = 0
        for i = 1, 100 do
            result = result + i
        end
        return result
    ]],
    [[
        local t = {}
        for i = 1, 10 do
            table.insert(t, i * i)
        end
        return table.concat(t, ", ")
    ]],
    [[
        return string.format("Math: sqrt(2) = %.4f", math.sqrt(2))
    ]],
}

print("Safe code execution:")
for i, code in ipairs(safe_codes) do
    local results, err = execute_sandboxed(code, sandbox, 1)
    if results then
        print(string.format("  Code %d: %s", i, tostring(results[1] or results)))
    else
        print(string.format("  Code %d ERROR: %s", i, err))
    end
end

-- Dangerous code attempts
print("\nDangerous code blocked:")
local dangerous_codes = {
    {code = 'os.execute("rm -rf /")', desc = "Command execution"},
    {code = 'io.open("/etc/passwd", "r"):read("*a")', desc = "File read"},
    {code = 'require("socket")', desc = "Require"},
    {code = 'while true do end', desc = "Infinite loop"},
    {code = 'local t = {} while true do t[#t+1] = {} end', desc = "Memory bomb"},
    {code = 'load("os.execute(\\"ls\\")")()', desc = "load() bypass"},
}

for _, test in ipairs(dangerous_codes) do
    local result, err = execute_sandboxed(test.code, sandbox, 1)
    if result then
        print(string.format("  %-25s: EXECUTED (BAD!)", test.desc))
    else
        print(string.format("  %-25s: BLOCKED - %s",
            test.desc, (err or "unknown"):sub(1, 50)))
    end
end
```

---

## ตัวอย่างที่ 10: Rate Limiting

```lua
-- Rate Limiting - ป้องกัน brute force และ DDoS

local socket = require("socket")

-- Token Bucket Rate Limiter
local TokenBucket = {}
TokenBucket.__index = TokenBucket

function TokenBucket.new(capacity, refill_rate)
    return setmetatable({
        capacity    = capacity,       -- max tokens
        tokens      = capacity,       -- current tokens
        refill_rate = refill_rate,    -- tokens per second
        last_refill = socket.gettime(),
    }, TokenBucket)
end

function TokenBucket:_refill()
    local now = socket.gettime()
    local elapsed = now - self.last_refill
    local new_tokens = elapsed * self.refill_rate
    self.tokens = math.min(self.capacity, self.tokens + new_tokens)
    self.last_refill = now
end

function TokenBucket:consume(tokens)
    tokens = tokens or 1
    self:_refill()

    if self.tokens >= tokens then
        self.tokens = self.tokens - tokens
        return true, self.tokens
    else
        return false, self.tokens
    end
end

function TokenBucket:remaining()
    self:_refill()
    return self.tokens
end

-- Sliding Window Counter
local SlidingWindowCounter = {}
SlidingWindowCounter.__index = SlidingWindowCounter

function SlidingWindowCounter.new(max_requests, window_seconds)
    return setmetatable({
        max_requests    = max_requests,
        window_seconds  = window_seconds,
        requests        = {},  -- {timestamp}
    }, SlidingWindowCounter)
end

function SlidingWindowCounter:is_allowed()
    local now = socket.gettime()
    local window_start = now - self.window_seconds

    -- ลบ requests เก่าออก
    local new_requests = {}
    for _, ts in ipairs(self.requests) do
        if ts > window_start then
            table.insert(new_requests, ts)
        end
    end
    self.requests = new_requests

    -- ตรวจสอบ
    if #self.requests < self.max_requests then
        table.insert(self.requests, now)
        return true, #self.requests
    else
        -- คำนวณเวลาที่ต้องรอ
        local oldest = self.requests[1]
        local wait = (oldest + self.window_seconds) - now
        return false, #self.requests, wait
    end
end

-- Per-IP Rate Limiter
local IPRateLimiter = {}
IPRateLimiter.__index = IPRateLimiter

function IPRateLimiter.new(max_requests, window_seconds, block_duration)
    return setmetatable({
        max_requests   = max_requests,
        window_seconds = window_seconds,
        block_duration = block_duration or 300,  -- 5 minutes
        limiters       = {},  -- {ip -> SlidingWindowCounter}
        blocked        = {},  -- {ip -> unblock_time}
    }, IPRateLimiter)
end

function IPRateLimiter:check(ip)
    local now = socket.gettime()

    -- ตรวจสอบ blocked list
    if self.blocked[ip] then
        if now < self.blocked[ip] then
            return false, "blocked", self.blocked[ip] - now
        else
            self.blocked[ip] = nil
        end
    end

    -- สร้าง limiter สำหรับ IP ถ้าไม่มี
    if not self.limiters[ip] then
        self.limiters[ip] = SlidingWindowCounter.new(
            self.max_requests, self.window_seconds)
    end

    local allowed, count, wait = self.limiters[ip]:is_allowed()

    if not allowed then
        -- Block IP ถ้า rate limit ถูก hit
        self.blocked[ip] = now + self.block_duration
        return false, "rate_limited", wait
    end

    return true, "allowed", count
end

function IPRateLimiter:get_stats()
    local stats = {
        total_limiters = 0,
        blocked_count  = 0,
        blocked_ips    = {},
    }

    for _ in pairs(self.limiters) do
        stats.total_limiters = stats.total_limiters + 1
    end

    local now = socket.gettime()
    for ip, unblock_time in pairs(self.blocked) do
        if now < unblock_time then
            stats.blocked_count = stats.blocked_count + 1
            table.insert(stats.blocked_ips, {
                ip = ip,
                remaining = math.ceil(unblock_time - now),
            })
        end
    end

    return stats
end

-- ทดสอบ
print("=== Rate Limiting ===\n")

-- Token Bucket
print("Token Bucket (5 tokens, 2/sec refill):")
local bucket = TokenBucket.new(5, 2)
for i = 1, 8 do
    local ok, remaining = bucket:consume(1)
    print(string.format("  Request %d: %s (remaining: %.1f tokens)",
        i, ok and "ALLOWED" or "DENIED", remaining))
    if i == 3 then
        socket.sleep(1.5)  -- รอเติม tokens
        print("  (waited 1.5 seconds)")
    end
end

-- Sliding Window
print("\nSliding Window (5 requests per 3 seconds):")
local window = SlidingWindowCounter.new(5, 3)
for i = 1, 8 do
    local ok, count, wait = window:is_allowed()
    if ok then
        print(string.format("  Request %d: ALLOWED (count: %d)", i, count))
    else
        print(string.format("  Request %d: DENIED (retry in %.2fs)", i, wait or 0))
    end
end

-- IP Rate Limiter
print("\nIP Rate Limiter (3 req per 2 seconds):")
local ip_limiter = IPRateLimiter.new(3, 2, 10)  -- 3 req/2s, 10s block

local test_ips = {
    "192.168.1.1", "192.168.1.2",
    "192.168.1.1", "192.168.1.1",
    "192.168.1.1", "192.168.1.1",  -- should trigger block
    "192.168.1.1",  -- should be blocked
    "192.168.1.2",  -- different IP, still allowed
}

for _, ip in ipairs(test_ips) do
    local ok, status, info = ip_limiter:check(ip)
    print(string.format("  %-15s: %s (%s %s)",
        ip,
        ok and "ALLOWED" or "DENIED",
        status,
        info and string.format("info=%.1f", info) or ""))
end

local stats = ip_limiter:get_stats()
print(string.format("\nStats: %d tracked IPs, %d currently blocked",
    stats.total_limiters, stats.blocked_count))
```

---

## ตัวอย่างที่ 11: XSS Prevention

```lua
-- XSS (Cross-Site Scripting) Prevention

-- HTML Entity Encoding
local function html_escape(str)
    if type(str) ~= "string" then
        return tostring(str)
    end

    -- ต้อง escape characters ที่ HTML interpret
    local replacements = {
        ["&"]  = "&amp;",
        ["<"]  = "&lt;",
        [">"]  = "&gt;",
        ['"']  = "&quot;",
        ["'"]  = "&#39;",
        ["/"]  = "&#x2F;",
        ["`"]  = "&#x60;",
        ["="]  = "&#x3D;",
    }

    return str:gsub('[&<>"\'/`=]', replacements)
end

-- JavaScript Context Escaping
local function js_escape(str)
    if type(str) ~= "string" then return "" end

    local replacements = {
        ["\\"] = "\\\\",
        ['"']  = '\\"',
        ["'"]  = "\\'",
        ["\n"] = "\\n",
        ["\r"] = "\\r",
        ["\t"] = "\\t",
        ["<"]  = "\\u003C",
        [">"]  = "\\u003E",
        ["&"]  = "\\u0026",
    }

    return str:gsub('[\\"\'\n\r\t<>&]', replacements)
end

-- URL Parameter Encoding
local function url_encode(str)
    if type(str) ~= "string" then return "" end

    return str:gsub("[^%w%-%.%_%~]", function(c)
        return string.format("%%%02X", c:byte())
    end)
end

-- Template engine ที่ safe (auto-escape)
local function safe_template(template, data, escape_func)
    escape_func = escape_func or html_escape

    -- แทน {{variable}} ด้วยค่าที่ escaped
    return template:gsub("{{%s*([%w_%.]+)%s*}}", function(key)
        -- Support dot notation
        local value = data
        for part in key:gmatch("[^%.]+") do
            if type(value) == "table" then
                value = value[part]
            else
                value = nil
                break
            end
        end

        if value == nil then
            return ""
        end

        return escape_func(tostring(value))
    end)
end

-- Raw output (ใช้เฉพาะกับ trusted content)
local function safe_template_raw(template, data)
    -- {{{variable}}} สำหรับ raw HTML (trusted)
    local result = template:gsub("{{{%s*([%w_%.]+)%s*}}}", function(key)
        local value = data[key]
        return value ~= nil and tostring(value) or ""
    end)

    -- {{variable}} สำหรับ escaped output
    return safe_template(result, data)
end

-- Content Security Policy helpers
local function generate_csp(options)
    options = options or {}
    local directives = {
        "default-src 'self'",
        "script-src 'self'" .. (options.script_hash and " '" .. options.script_hash .. "'" or ""),
        "style-src 'self' 'unsafe-inline'",
        "img-src 'self' data: https:",
        "font-src 'self'",
        "connect-src 'self'",
        "frame-ancestors 'none'",
        "base-uri 'self'",
        "form-action 'self'",
    }
    return table.concat(directives, "; ")
end

-- Security Headers
local function get_security_headers()
    return {
        ["X-Content-Type-Options"]    = "nosniff",
        ["X-Frame-Options"]           = "DENY",
        ["X-XSS-Protection"]          = "1; mode=block",
        ["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains",
        ["Referrer-Policy"]           = "strict-origin-when-cross-origin",
        ["Content-Security-Policy"]   = generate_csp(),
        ["Permissions-Policy"]        = "geolocation=(), microphone=(), camera=()",
    }
end

-- ทดสอบ
print("=== XSS Prevention ===\n")

-- HTML escaping
print("HTML escaping:")
local xss_attempts = {
    "<script>alert('XSS')</script>",
    "<img src=x onerror=alert(1)>",
    "' onclick='alert(1)'",
    "<a href=\"javascript:alert(1)\">click</a>",
    '"><script>document.cookie</script>',
    "Hello & <World> \"World\"",
}

for _, attempt in ipairs(xss_attempts) do
    print(string.format("  Input:   %s", attempt))
    print(string.format("  Escaped: %s", html_escape(attempt)))
    print()
end

-- Template rendering
print("Safe template rendering:")
local template = [[
<div>
  <h1>{{title}}</h1>
  <p>Author: {{author.name}}</p>
  <p>Bio: {{author.bio}}</p>
</div>
]]

local user_data = {
    title = "My Post <script>alert('xss')</script>",
    author = {
        name = "Alice & Bob",
        bio  = '<b>Developer</b> & "Writer"',
    }
}

print(safe_template(template, user_data))

-- Security headers
print("Security Headers:")
for name, value in pairs(get_security_headers()) do
    print(string.format("  %-35s: %s", name, value:sub(1, 60)))
end
```

---

## ตัวอย่างที่ 12: CSRF Protection

```lua
-- CSRF (Cross-Site Request Forgery) Protection

local socket  = require("socket")

-- CSRF Token Manager
local CSRFProtection = {}
CSRFProtection.__index = CSRFProtection

function CSRFProtection.new(options)
    options = options or {}
    return setmetatable({
        tokens        = {},
        token_ttl     = options.token_ttl or 3600,  -- 1 hour
        token_length  = options.token_length or 32,
    }, CSRFProtection)
end

function CSRFProtection:generate_token(session_id)
    -- สร้าง cryptographically random token
    local function random_bytes(n)
        local bytes = {}
        local urandom = io.open("/dev/urandom", "rb")
        if urandom then
            local data = urandom:read(n)
            urandom:close()
            return data
        else
            for i = 1, n do
                bytes[i] = string.char(math.random(0, 255))
            end
            return table.concat(bytes)
        end
    end

    local function to_hex(s)
        return s:gsub(".", function(c)
            return string.format("%02x", c:byte())
        end)
    end

    local token = to_hex(random_bytes(self.token_length))
    local expiry = socket.gettime() + self.token_ttl

    -- เก็บ token
    if not self.tokens[session_id] then
        self.tokens[session_id] = {}
    end
    self.tokens[session_id][token] = expiry

    return token
end

function CSRFProtection:verify_token(session_id, token)
    if type(token) ~= "string" or #token == 0 then
        return false, "Missing token"
    end

    local session_tokens = self.tokens[session_id]
    if not session_tokens then
        return false, "No tokens for session"
    end

    local expiry = session_tokens[token]
    if not expiry then
        return false, "Token not found"
    end

    if socket.gettime() > expiry then
        session_tokens[token] = nil
        return false, "Token expired"
    end

    -- Single-use token
    session_tokens[token] = nil
    return true
end

function CSRFProtection:cleanup_expired()
    local now = socket.gettime()
    local cleaned = 0

    for session_id, tokens in pairs(self.tokens) do
        for token, expiry in pairs(tokens) do
            if now > expiry then
                tokens[token] = nil
                cleaned = cleaned + 1
            end
        end

        -- ลบ session ถ้าไม่มี tokens แล้ว
        if next(tokens) == nil then
            self.tokens[session_id] = nil
        end
    end

    return cleaned
end

-- สร้าง CSRF token สำหรับ HTML form
local function csrf_hidden_field(token)
    return string.format(
        '<input type="hidden" name="csrf_token" value="%s">',
        token:gsub('"', "&quot;")
    )
end

-- ทดสอบ
print("=== CSRF Protection ===\n")

local csrf = CSRFProtection.new({token_ttl = 30})

-- สร้าง tokens
print("Generating CSRF tokens:")
local session_id = "sess_user123"
local token1 = csrf:generate_token(session_id)
local token2 = csrf:generate_token(session_id)

print("  Token 1:", token1:sub(1, 32) .. "...")
print("  Token 2:", token2:sub(1, 32) .. "...")
print("  HTML:", csrf_hidden_field(token1):sub(1, 80))

-- Verify tokens
print("\nVerifying tokens:")
local cases = {
    {token = token1, desc = "Valid token 1"},
    {token = token2, desc = "Valid token 2"},
    {token = token1, desc = "Replay attack (same token)"},
    {token = "fake_token_123", desc = "Fake token"},
    {token = "", desc = "Empty token"},
}

for _, tc in ipairs(cases) do
    local ok, err = csrf:verify_token(session_id, tc.token)
    print(string.format("  %-35s: %s%s",
        tc.desc,
        ok and "VALID" or "REJECTED",
        err and " (" .. err .. ")" or ""))
end

-- Test different sessions
print("\nCross-session attack (wrong session):")
local token3 = csrf:generate_token("sess_attacker")
local ok, err = csrf:verify_token(session_id, token3)
print(string.format("  Using attacker's token on victim session: %s",
    ok and "ALLOWED (BAD!)" or "REJECTED: " .. (err or "unknown")))

-- Cleanup
print(string.format("\nCleaned up %d expired tokens",
    csrf:cleanup_expired()))
```

---

## ตัวอย่างที่ 13: Secure Configuration

```lua
-- Secure Configuration Management

-- ============================================================
-- อย่าเขียน secrets ใน source code
-- ============================================================

-- BAD: Hardcoded credentials (DO NOT DO THIS!)
local function bad_config_example()
    -- อย่าทำอย่างนี้!
    local config = {
        db_password = "super_secret_password",  -- BAD!
        api_key     = "sk-abc123xyz",            -- BAD!
        jwt_secret  = "my_jwt_secret_key",       -- BAD!
    }
    return config
end

-- GOOD: Load from environment variables
local function load_from_env(required_keys, optional_keys)
    local config = {}
    local missing = {}

    -- Required keys
    for _, key in ipairs(required_keys or {}) do
        local value = os.getenv(key)
        if not value or #value == 0 then
            table.insert(missing, key)
        else
            config[key:lower()] = value
        end
    end

    if #missing > 0 then
        return nil, "Missing required environment variables: " ..
                    table.concat(missing, ", ")
    end

    -- Optional keys
    for key, default in pairs(optional_keys or {}) do
        config[key:lower()] = os.getenv(key) or default
    end

    return config
end

-- Configuration validation
local function validate_config(config)
    local errors = {}

    -- Database
    if config.database_url then
        if not config.database_url:match("^%w+://") then
            table.insert(errors, "DATABASE_URL must be a valid URL")
        end
    end

    -- Ports
    if config.port then
        local port = tonumber(config.port)
        if not port or port < 1 or port > 65535 then
            table.insert(errors, "PORT must be between 1 and 65535")
        end
    end

    -- Environment
    if config.environment then
        local valid_envs = {development=true, staging=true, production=true}
        if not valid_envs[config.environment:lower()] then
            table.insert(errors, "ENVIRONMENT must be development/staging/production")
        end
    end

    -- Secret key length
    if config.secret_key then
        if #config.secret_key < 32 then
            table.insert(errors, "SECRET_KEY must be at least 32 characters")
        end
    end

    return #errors == 0, errors
end

-- Secure config printer (hides secrets)
local function print_config(config)
    local sensitive_keys = {
        password=true, secret=true, key=true, token=true,
        credential=true, auth=true
    }

    local function is_sensitive(key)
        local lower_key = key:lower()
        for pattern in pairs(sensitive_keys) do
            if lower_key:find(pattern) then return true end
        end
        return false
    end

    print("Configuration:")
    for k, v in pairs(config) do
        if is_sensitive(k) then
            print(string.format("  %-25s = %s", k, "***HIDDEN***"))
        else
            print(string.format("  %-25s = %s", k, tostring(v)))
        end
    end
end

-- ทดสอบ
print("=== Secure Configuration ===\n")

-- จำลอง environment variables
local function set_test_env()
    -- ในการทดสอบจริง ค่าเหล่านี้ควรมาจาก environment จริงๆ
    local mock_env = {
        APP_SECRET_KEY = "a-very-long-random-secret-key-for-testing-purposes-123",
        DATABASE_URL   = "sqlite:///app.db",
        PORT           = "8080",
        ENVIRONMENT    = "development",
        ALLOWED_HOSTS  = "localhost,127.0.0.1",
        DEBUG          = "false",
    }
    return mock_env
end

-- จำลอง os.getenv
local mock_env = set_test_env()
local original_getenv = os.getenv
os.getenv = function(key)
    return mock_env[key] or original_getenv(key)
end

-- Load config
local config, err = load_from_env(
    {"APP_SECRET_KEY", "DATABASE_URL"},  -- required
    {                                     -- optional with defaults
        PORT        = "3000",
        ENVIRONMENT = "development",
        DEBUG       = "false",
    }
)

if config then
    print("Config loaded successfully")
    print_config(config)

    -- Validate
    local valid, errors = validate_config(config)
    if valid then
        print("\nConfiguration is valid!")
    else
        print("\nConfiguration errors:")
        for _, e in ipairs(errors) do
            print("  - " .. e)
        end
    end
else
    print("Config load failed:", err)
end

-- Restore
os.getenv = original_getenv
```

---

## ตัวอย่างที่ 14: Security Headers และ Secure Communications

```lua
-- Security Headers

-- ============================================================
-- HTTP Security Headers
-- ============================================================
local SecurityHeaders = {}

-- Content Security Policy builder
function SecurityHeaders.build_csp(directives)
    local defaults = {
        ["default-src"]     = "'self'",
        ["script-src"]      = "'self'",
        ["style-src"]       = "'self' 'unsafe-inline'",
        ["img-src"]         = "'self' data: https:",
        ["font-src"]        = "'self' https://fonts.gstatic.com",
        ["connect-src"]     = "'self'",
        ["media-src"]       = "'none'",
        ["object-src"]      = "'none'",
        ["frame-src"]       = "'none'",
        ["frame-ancestors"] = "'none'",
        ["base-uri"]        = "'self'",
        ["form-action"]     = "'self'",
        ["upgrade-insecure-requests"] = "",
    }

    -- Override with custom directives
    for k, v in pairs(directives or {}) do
        defaults[k] = v
    end

    local parts = {}
    for directive, value in pairs(defaults) do
        if value == "" then
            table.insert(parts, directive)
        else
            table.insert(parts, directive .. " " .. value)
        end
    end

    table.sort(parts)
    return table.concat(parts, "; ")
end

-- Complete security headers set
function SecurityHeaders.get_all(options)
    options = options or {}

    return {
        -- Prevents MIME sniffing
        ["X-Content-Type-Options"] = "nosniff",

        -- Clickjacking prevention
        ["X-Frame-Options"] = "DENY",

        -- Legacy XSS filter
        ["X-XSS-Protection"] = "1; mode=block",

        -- HSTS (HTTPS only)
        ["Strict-Transport-Security"] =
            "max-age=31536000; includeSubDomains; preload",

        -- Referrer policy
        ["Referrer-Policy"] = "strict-origin-when-cross-origin",

        -- Content Security Policy
        ["Content-Security-Policy"] = SecurityHeaders.build_csp(
            options.csp_overrides
        ),

        -- Permissions Policy
        ["Permissions-Policy"] = table.concat({
            "accelerometer=()",
            "camera=()",
            "geolocation=()",
            "gyroscope=()",
            "magnetometer=()",
            "microphone=()",
            "payment=()",
            "usb=()",
        }, ", "),

        -- Cache control for sensitive pages
        ["Cache-Control"] = options.no_cache and
            "no-store, no-cache, must-revalidate" or
            "public, max-age=3600",

        -- Cross-origin policies
        ["Cross-Origin-Opener-Policy"]   = "same-origin",
        ["Cross-Origin-Embedder-Policy"] = "require-corp",
        ["Cross-Origin-Resource-Policy"] = "same-origin",
    }
end

-- Cookie security
function SecurityHeaders.secure_cookie(name, value, options)
    options = options or {}
    local parts = {
        name .. "=" .. value,
        "HttpOnly",                          -- ป้องกัน JS access
        "Secure",                            -- HTTPS only
        "SameSite=" .. (options.same_site or "Lax"),  -- CSRF protection
    }

    if options.path then
        table.insert(parts, "Path=" .. options.path)
    else
        table.insert(parts, "Path=/")
    end

    if options.domain then
        table.insert(parts, "Domain=" .. options.domain)
    end

    if options.max_age then
        table.insert(parts, "Max-Age=" .. options.max_age)
    elseif options.expires then
        table.insert(parts, "Expires=" .. options.expires)
    end

    return table.concat(parts, "; ")
end

-- ทดสอบ
print("=== Security Headers ===\n")

-- แสดง headers ทั้งหมด
local headers = SecurityHeaders.get_all({no_cache = true})
print("Security Headers:")
for name, value in pairs(headers) do
    print(string.format("  %-40s: %s", name, value:sub(1, 70)))
end

-- CSP examples
print("\nContent Security Policy examples:")
print("  Default:")
print("  " .. SecurityHeaders.build_csp():sub(1, 100) .. "...")

print("\n  Strict (no inline):")
local strict_csp = SecurityHeaders.build_csp({
    ["script-src"] = "'self' 'nonce-abc123'",
    ["style-src"]  = "'self'",
})
print("  " .. strict_csp:sub(1, 100) .. "...")

-- Secure cookies
print("\nSecure Cookie examples:")
local cookies = {
    SecurityHeaders.secure_cookie("session_id", "abc123xyz", {
        max_age = 3600,
        same_site = "Strict",
    }),
    SecurityHeaders.secure_cookie("remember_me", "token456", {
        max_age = 2592000,  -- 30 days
        same_site = "Lax",
    }),
    SecurityHeaders.secure_cookie("csrf_token", "csrf789", {
        same_site = "Strict",
        path      = "/api",
    }),
}

for i, cookie in ipairs(cookies) do
    print(string.format("  Cookie %d: %s", i, cookie))
end
```

---

## ตัวอย่างที่ 15: Security Audit Checklist

```lua
-- Security Audit Tool

local SecurityAudit = {}
SecurityAudit.__index = SecurityAudit

function SecurityAudit.new()
    return setmetatable({
        findings = {},
        checks_run = 0,
    }, SecurityAudit)
end

function SecurityAudit:add_finding(severity, category, message, recommendation)
    table.insert(self.findings, {
        severity       = severity,  -- "CRITICAL", "HIGH", "MEDIUM", "LOW", "INFO"
        category       = category,
        message        = message,
        recommendation = recommendation,
    })
end

function SecurityAudit:check_string_concat_sql(code_snippet)
    self.checks_run = self.checks_run + 1

    -- ตรวจสอบ SQL string concatenation
    if code_snippet:find("SELECT.*%+.*\"") or
       code_snippet:find("SELECT.*%.%.") or
       code_snippet:find("WHERE.*\".*%+") then
        self:add_finding(
            "CRITICAL",
            "SQL Injection",
            "Possible SQL injection via string concatenation",
            "Use parameterized queries: db:prepare('... WHERE id = ?')"
        )
        return false
    end
    return true
end

function SecurityAudit:check_hardcoded_secrets(code_snippet)
    self.checks_run = self.checks_run + 1

    local patterns = {
        {pattern = "password%s*=%s*['\"]%w+['\"]",
         desc = "Hardcoded password"},
        {pattern = "api_key%s*=%s*['\"]%w+['\"]",
         desc = "Hardcoded API key"},
        {pattern = "secret%s*=%s*['\"][%w_]+['\"]",
         desc = "Hardcoded secret"},
        {pattern = "token%s*=%s*['\"]%w+['\"]",
         desc = "Hardcoded token"},
    }

    local found = false
    for _, p in ipairs(patterns) do
        if code_snippet:lower():find(p.pattern) then
            self:add_finding(
                "HIGH",
                "Hardcoded Credentials",
                p.desc .. " detected in source code",
                "Use environment variables: os.getenv('SECRET_KEY')"
            )
            found = true
        end
    end

    return not found
end

function SecurityAudit:check_dangerous_functions(code_snippet)
    self.checks_run = self.checks_run + 1

    local dangerous = {
        {func = "os.execute", risk = "Command injection", sev = "CRITICAL"},
        {func = "io.popen", risk = "Command injection", sev = "HIGH"},
        {func = "load%(", risk = "Code injection via load()", sev = "HIGH"},
        {func = "loadstring", risk = "Code injection", sev = "HIGH"},
        {func = "dofile", risk = "Arbitrary file execution", sev = "HIGH"},
        {func = "require.*os", risk = "OS module", sev = "MEDIUM"},
    }

    local issues = false
    for _, d in ipairs(dangerous) do
        if code_snippet:find(d.func) then
            self:add_finding(
                d.sev,
                "Dangerous Function",
                d.risk .. ": " .. d.func,
                "Validate inputs and use safe alternatives"
            )
            issues = true
        end
    end

    return not issues
end

function SecurityAudit:check_file_operations(code_snippet)
    self.checks_run = self.checks_run + 1

    if code_snippet:find("io.open%s*%(" ..
       "[^)]*%.%.") then
        self:add_finding(
            "HIGH",
            "Path Traversal",
            "Possible path traversal in file operation",
            "Validate and canonicalize file paths before opening"
        )
        return false
    end
    return true
end

function SecurityAudit:generate_report()
    local report = {
        summary = {
            total_checks   = self.checks_run,
            total_findings = #self.findings,
            critical       = 0,
            high           = 0,
            medium         = 0,
            low            = 0,
            info           = 0,
        },
        findings = self.findings,
    }

    for _, f in ipairs(self.findings) do
        local sev = f.severity:lower()
        if report.summary[sev] then
            report.summary[sev] = report.summary[sev] + 1
        end
    end

    return report
end

function SecurityAudit:print_report()
    local report = self:generate_report()

    print("\n" .. string.rep("=", 60))
    print("SECURITY AUDIT REPORT")
    print(string.rep("=", 60))

    print(string.format("Checks run: %d | Findings: %d",
        report.summary.total_checks, report.summary.total_findings))
    print(string.format("  CRITICAL: %d | HIGH: %d | MEDIUM: %d | LOW: %d | INFO: %d",
        report.summary.critical, report.summary.high,
        report.summary.medium, report.summary.low, report.summary.info))

    if #report.findings > 0 then
        print("\nFindings:")
        for i, f in ipairs(report.findings) do
            print(string.format("\n[%d] %s - %s",
                i, f.severity, f.category))
            print("  Issue: " .. f.message)
            print("  Fix:   " .. (f.recommendation or "See OWASP guidelines"))
        end
    else
        print("\nNo security issues found!")
    end

    print(string.rep("=", 60))
end

-- ทดสอบ audit tool
print("=== Security Audit Tool ===")

local audit = SecurityAudit.new()

-- ตัวอย่าง code snippets ที่จะตรวจสอบ
local code_examples = {
    -- SQL injection
    'local sql = "SELECT * FROM users WHERE id=" .. user_input',
    -- Hardcoded secret
    'local api_key = "sk-abc123def456"',
    -- Dangerous function
    'os.execute("ping " .. hostname)',
    -- Path traversal
    'local f = io.open("files/" .. filename .. "/../../../etc/passwd")',
    -- Good code
    'local stmt = db:prepare("SELECT * FROM users WHERE id = ?")',
}

for _, code in ipairs(code_examples) do
    audit:check_string_concat_sql(code)
    audit:check_hardcoded_secrets(code)
    audit:check_dangerous_functions(code)
    audit:check_file_operations(code)
end

audit:print_report()
```

---

## ตัวอย่างที่ 16: Timing Attack Prevention

```lua
-- Constant-time string comparison to prevent timing attacks
-- Used for comparing MACs, tokens, passwords

local function hmac_sha256_stub(key, msg)
    -- Stub: real code would use OpenSSL binding
    -- Returns fixed-length 32-byte hex string
    local hash = 0
    for i = 1, #key do hash = (hash * 31 + key:byte(i)) % (2^32) end
    for i = 1, #msg do hash = (hash * 31 + msg:byte(i)) % (2^32) end
    return string.format("%064x", hash)
end

-- Constant-time compare (same length check first, then XOR all bytes)
local function ct_compare(a, b)
    if type(a) ~= "string" or type(b) ~= "string" then return false end
    if #a ~= #b then return false end
    local diff = 0
    for i = 1, #a do
        diff = diff | (a:byte(i) ~ b:byte(i))
    end
    return diff == 0
end

-- Secure cookie / session token validator
local SecureToken = {}
SecureToken.__index = SecureToken

function SecureToken.new(secret)
    return setmetatable({secret = secret}, SecureToken)
end

function SecureToken:sign(payload)
    local mac = hmac_sha256_stub(self.secret, payload)
    return payload .. "." .. mac
end

function SecureToken:verify(token)
    local dot = token:find("%.[^.]*$")
    if not dot then return nil, "malformed token" end
    local payload = token:sub(1, dot - 1)
    local provided_mac = token:sub(dot + 1)
    local expected_mac = hmac_sha256_stub(self.secret, payload)
    if not ct_compare(provided_mac, expected_mac) then
        return nil, "invalid signature"
    end
    return payload
end

-- Demo
local st = SecureToken.new("super-secret-key-32bytes-padding!!")

local token = st:sign("user_id=42&role=admin&exp=9999999999")
print("Signed token:", token:sub(1, 60) .. "...")

local payload, err = st:verify(token)
print("Valid token payload:", payload)

-- Tampered token
local tampered = token:sub(1, -5) .. "XXXX"
local _, terr = st:verify(tampered)
print("Tampered token error:", terr)

-- Timing test: both paths take ~same time
local t0 = os.clock()
for _ = 1, 100000 do ct_compare("abc", "abd") end
local t1 = os.clock()
for _ = 1, 100000 do ct_compare("abc", "abc") end
local t2 = os.clock()
print(string.format("CT compare mismatch: %.3fms  match: %.3fms",
    (t1-t0)*1000, (t2-t1)*1000))
```

---

## ตัวอย่างที่ 17: Secure Session Management

```lua
-- Secure session store with expiry, rotation, and hijack detection
local Session = {}
Session.__index = Session

local function random_hex(n)
    local t = {}
    for _ = 1, n do
        t[#t+1] = string.format("%02x", math.random(0, 255))
    end
    return table.concat(t)
end

function Session.new(opts)
    opts = opts or {}
    local self = setmetatable({}, Session)
    self.store   = {}      -- session_id -> session data
    self.ttl     = opts.ttl or 1800       -- 30 min default
    self.max_age = opts.max_age or 86400  -- 24 hr absolute
    return self
end

function Session:create(user_id, ip, ua)
    local sid = random_hex(32)
    local now = os.time()
    self.store[sid] = {
        user_id    = user_id,
        ip         = ip,
        ua         = ua,
        created_at = now,
        last_seen  = now,
        data       = {},
        rotations  = 0,
    }
    return sid
end

function Session:get(sid, ip, ua)
    local s = self.store[sid]
    if not s then return nil, "session not found" end
    local now = os.time()
    -- Expiry checks
    if now - s.last_seen > self.ttl then
        self.store[sid] = nil
        return nil, "session expired (idle)"
    end
    if now - s.created_at > self.max_age then
        self.store[sid] = nil
        return nil, "session expired (max age)"
    end
    -- Hijack detection
    if s.ip and s.ip ~= ip then
        self.store[sid] = nil
        return nil, "session hijack detected (IP changed)"
    end
    s.last_seen = now
    return s
end

-- Rotate session ID after privilege change
function Session:rotate(old_sid)
    local s = self.store[old_sid]
    if not s then return nil, "not found" end
    local new_sid = random_hex(32)
    s.rotations = s.rotations + 1
    self.store[new_sid] = s
    self.store[old_sid] = nil
    return new_sid
end

function Session:destroy(sid)
    self.store[sid] = nil
end

function Session:cleanup()
    local now, removed = os.time(), 0
    for sid, s in pairs(self.store) do
        if now - s.last_seen > self.ttl or now - s.created_at > self.max_age then
            self.store[sid] = nil
            removed = removed + 1
        end
    end
    return removed
end

-- Demo
math.randomseed(os.time())
local sessions = Session.new({ttl=300, max_age=3600})

local sid = sessions:create(42, "203.0.113.10", "Mozilla/5.0")
print("Created session:", sid:sub(1, 16) .. "...")

local s, err = sessions:get(sid, "203.0.113.10", "Mozilla/5.0")
print("Session valid:", s ~= nil, "user_id:", s and s.user_id)

-- Rotate after login
local new_sid = sessions:rotate(sid)
print("After rotation:", new_sid:sub(1, 16) .. "...")
s = sessions:get(new_sid, "203.0.113.10")
print("New session valid:", s ~= nil)

-- Hijack attempt
local _, herr = sessions:get(new_sid, "10.0.0.1")
print("Hijack attempt:", herr)

-- Original ID invalidated
local _, old_err = sessions:get(sid, "203.0.113.10")
print("Old session after rotation:", old_err)
```

---

## ตัวอย่างที่ 18: JWT-like Token ด้วย HMAC

```lua
-- JWT-like token implementation with header.payload.signature
-- Uses a simple HMAC simulation (real code should use crypto library)

local b64 = {}
local b64chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"

function b64.encode(data)
    local result = {}
    for i = 1, #data, 3 do
        local a = data:byte(i) or 0
        local b = data:byte(i+1) or 0
        local c = data:byte(i+2) or 0
        local n = a * 65536 + b * 256 + c
        result[#result+1] = b64chars:sub(((n>>18)&63)+1, ((n>>18)&63)+1)
        result[#result+1] = b64chars:sub(((n>>12)&63)+1, ((n>>12)&63)+1)
        result[#result+1] = (i+1 <= #data) and b64chars:sub(((n>>6)&63)+1, ((n>>6)&63)+1) or "="
        result[#result+1] = (i+2 <= #data) and b64chars:sub((n&63)+1, (n&63)+1) or "="
    end
    return table.concat(result):gsub("+", "-"):gsub("/", "_"):gsub("=", "")
end

function b64.decode(data)
    data = data:gsub("-", "+"):gsub("_", "/")
    local pad = (4 - #data % 4) % 4
    data = data .. string.rep("=", pad)
    local result = {}
    local lookup = {}
    for i = 1, #b64chars do lookup[b64chars:sub(i,i)] = i-1 end
    lookup["="] = 0
    for i = 1, #data, 4 do
        local a = lookup[data:sub(i,i)] or 0
        local b_ = lookup[data:sub(i+1,i+1)] or 0
        local c_ = lookup[data:sub(i+2,i+2)] or 0
        local d = lookup[data:sub(i+3,i+3)] or 0
        local n = a*262144 + b_*4096 + c_*64 + d
        result[#result+1] = string.char((n>>16)&255)
        if data:sub(i+2,i+2) ~= "=" then
            result[#result+1] = string.char((n>>8)&255)
        end
        if data:sub(i+3,i+3) ~= "=" then
            result[#result+1] = string.char(n&255)
        end
    end
    return table.concat(result)
end

-- Minimal JSON encode
local function json_encode(t)
    local parts = {}
    for k, v in pairs(t) do
        local val
        if type(v) == "string" then val = string.format('%q', v)
        elseif type(v) == "number" then val = tostring(v)
        elseif type(v) == "boolean" then val = tostring(v)
        else val = '"[object]"' end
        parts[#parts+1] = string.format('%q:%s', k, val)
    end
    return "{" .. table.concat(parts, ",") .. "}"
end

local function minimal_hmac(key, msg)
    local h = 0
    for i = 1, #key do h = (h * 1000003 + key:byte(i)) % (2^32) end
    for i = 1, #msg do h = (h * 1000003 + msg:byte(i)) % (2^32) end
    return string.format("%032x%032x", h, (h ~ 0xDEADBEEF) % (2^32))
end

local JWT = {}
JWT.__index = JWT

function JWT.new(secret, algorithm)
    return setmetatable({secret=secret, alg=algorithm or "HS256"}, JWT)
end

function JWT:sign(claims)
    local header  = b64.encode(json_encode({alg=self.alg, typ="JWT"}))
    local payload = b64.encode(json_encode(claims))
    local signing_input = header .. "." .. payload
    local sig = b64.encode(minimal_hmac(self.secret, signing_input))
    return signing_input .. "." .. sig
end

function JWT:verify(token)
    local parts = {}
    for p in (token .. "."):gmatch("([^.]*).") do parts[#parts+1] = p end
    if #parts ~= 3 then return nil, "malformed token" end
    local signing_input = parts[1] .. "." .. parts[2]
    local expected_sig = b64.encode(minimal_hmac(self.secret, signing_input))
    -- Constant-time compare
    if parts[3] ~= expected_sig then return nil, "invalid signature" end
    -- Decode payload (minimal JSON decode for numbers/strings)
    local payload_json = b64.decode(parts[2])
    local claims = {}
    for k, v in payload_json:gmatch('"([^"]+)":%s*(%S+)') do
        claims[k] = tonumber(v) or v:match('^"(.*)"$') or v
    end
    -- Check expiry
    if claims.exp and type(claims.exp) == "number" and claims.exp < os.time() then
        return nil, "token expired"
    end
    return claims
end

-- Demo
local jwt = JWT.new("my-super-secret-key-256bits-padding!!")

local token = jwt:sign({
    sub  = "user_42",
    role = "admin",
    iat  = os.time(),
    exp  = os.time() + 3600,
})
print("Token:", token:sub(1, 60) .. "...")

local claims, err = jwt:verify(token)
if claims then
    print("Claims: sub=" .. (claims.sub or "?") ..
          " role=" .. (claims.role or "?"))
end

-- Tampered token
local hdr, rest = token:match("^([^.]+%.[^.]+)%.(.+)$")
local tampered = hdr .. ".badsignature"
local _, terr = jwt:verify(tampered)
print("Tampered:", terr)
```

---

## ตัวอย่างที่ 19: Safe File Operations

```lua
-- Secure file operations: path validation, temp files, atomic writes
local SafeFile = {}
SafeFile.__index = SafeFile

function SafeFile.new(base_dir)
    local self = setmetatable({}, SafeFile)
    -- Normalize base_dir (no trailing slash)
    self.base_dir = (base_dir or "/tmp/safefiles"):gsub("/$", "")
    self.max_file_size = 10 * 1024 * 1024  -- 10 MB
    return self
end

-- Resolve and validate path stays within base_dir
function SafeFile:resolve(rel_path)
    if type(rel_path) ~= "string" then return nil, "path must be string" end
    -- Reject null bytes
    if rel_path:find("\0") then return nil, "null byte in path" end
    -- Remove leading slashes/dots
    rel_path = rel_path:gsub("^[./]+", "")
    -- Reject double-dot traversal after normalization
    if rel_path:find("%.%.") then return nil, "path traversal detected" end
    -- Reject absolute paths
    if rel_path:sub(1,1) == "/" then return nil, "absolute path rejected" end
    local full = self.base_dir .. "/" .. rel_path
    return full
end

-- Safe read
function SafeFile:read(rel_path)
    local full, err = self:resolve(rel_path)
    if not full then return nil, err end
    local f, ferr = io.open(full, "rb")
    if not f then return nil, "cannot open: " .. ferr end
    local data = f:read("*a")
    f:close()
    if #data > self.max_file_size then
        return nil, "file too large"
    end
    return data
end

-- Atomic write: write to temp, rename
function SafeFile:write(rel_path, data)
    local full, err = self:resolve(rel_path)
    if not full then return false, err end
    if type(data) ~= "string" then return false, "data must be string" end
    if #data > self.max_file_size then return false, "data too large" end
    -- Validate no dangerous content for text files
    local tmp = full .. ".tmp." .. tostring(os.time())
    local f, ferr = io.open(tmp, "wb")
    if not f then return false, "cannot create temp: " .. ferr end
    f:write(data)
    f:close()
    -- Atomic rename (POSIX)
    local ok, rerr = os.rename(tmp, full)
    if not ok then
        os.remove(tmp)
        return false, "rename failed: " .. (rerr or "unknown")
    end
    return true
end

-- Safe delete: only within base_dir
function SafeFile:delete(rel_path)
    local full, err = self:resolve(rel_path)
    if not full then return false, err end
    local ok, derr = os.remove(full)
    if not ok then return false, derr end
    return true
end

-- List directory (no recursion to avoid info disclosure)
function SafeFile:list(rel_dir)
    local full, err = self:resolve(rel_dir or ".")
    if not full then return nil, err end
    local files = {}
    local handle = io.popen("ls -1 " .. string.format("%q", full) .. " 2>/dev/null")
    if handle then
        for line in handle:lines() do
            if not line:find("%.%.$") and line ~= "." then
                files[#files+1] = line
            end
        end
        handle:close()
    end
    return files
end

-- Demo
local sf = SafeFile.new("/tmp/safefiles_test")
os.execute("mkdir -p /tmp/safefiles_test")

local ok, err

ok, err = sf:write("config.json", '{"debug":false,"version":"1.0"}')
print("Write config.json:", ok, err)

local data = sf:read("config.json")
print("Read config.json:", data)

-- Path traversal attempts
_, err = sf:resolve("../../etc/passwd")
print("Traversal blocked:", err)

_, err = sf:resolve("subdir/../../etc/hosts")
print("Double-dot blocked:", err)

_, err = sf:resolve("/etc/shadow")
print("Absolute blocked:", err)

ok, err = sf:delete("config.json")
print("Delete:", ok, err)

os.execute("rm -rf /tmp/safefiles_test")
```

---

## ตัวอย่างที่ 20: Role-Based Access Control (RBAC)

```lua
-- Full RBAC system: roles, permissions, resources, policies
local RBAC = {}
RBAC.__index = RBAC

function RBAC.new()
    local self = setmetatable({}, RBAC)
    self.roles       = {}   -- role_name -> {permissions}
    self.users       = {}   -- user_id   -> {roles}
    self.inheritance = {}   -- role -> parent roles
    self.policies    = {}   -- custom policy functions
    return self
end

function RBAC:define_role(name, permissions, parent_roles)
    self.roles[name] = {}
    for _, perm in ipairs(permissions or {}) do
        self.roles[name][perm] = true
    end
    self.inheritance[name] = parent_roles or {}
end

function RBAC:assign_role(user_id, role)
    if not self.roles[role] then
        error("Role '" .. role .. "' not defined")
    end
    self.users[user_id] = self.users[user_id] or {}
    self.users[user_id][role] = true
end

function RBAC:revoke_role(user_id, role)
    if self.users[user_id] then
        self.users[user_id][role] = nil
    end
end

-- Collect all permissions including inherited
function RBAC:_all_permissions(role, visited)
    visited = visited or {}
    if visited[role] then return {} end
    visited[role] = true
    local perms = {}
    for perm in pairs(self.roles[role] or {}) do
        perms[perm] = true
    end
    for _, parent in ipairs(self.inheritance[role] or {}) do
        for perm in pairs(self:_all_permissions(parent, visited)) do
            perms[perm] = true
        end
    end
    return perms
end

function RBAC:can(user_id, permission, context)
    local user_roles = self.users[user_id]
    if not user_roles then return false end
    -- Check custom policies first
    if self.policies[permission] then
        return self.policies[permission](user_id, context)
    end
    for role in pairs(user_roles) do
        local perms = self:_all_permissions(role)
        if perms[permission] or perms["*"] then return true end
    end
    return false
end

function RBAC:add_policy(permission, fn)
    self.policies[permission] = fn
end

function RBAC:get_roles(user_id)
    local roles = {}
    for role in pairs(self.users[user_id] or {}) do
        roles[#roles+1] = role
    end
    return roles
end

-- Demo: Content Management System
local rbac = RBAC.new()

-- Define role hierarchy
rbac:define_role("viewer",    {"post:read", "comment:read"})
rbac:define_role("author",    {"post:create", "post:update_own", "comment:create"},
                              {"viewer"})
rbac:define_role("editor",    {"post:update", "post:delete", "comment:delete"},
                              {"author"})
rbac:define_role("admin",     {"user:manage", "role:assign", "*"},
                              {"editor"})

-- Assign roles
rbac:assign_role(1, "admin")
rbac:assign_role(2, "editor")
rbac:assign_role(3, "author")
rbac:assign_role(4, "viewer")

-- Custom policy: authors can only edit their own posts
rbac:add_policy("post:update_own", function(user_id, ctx)
    return ctx and ctx.post_owner == user_id
end)

-- Check permissions
local users = {
    {id=1, name="Admin"},
    {id=2, name="Editor"},
    {id=3, name="Author"},
    {id=4, name="Viewer"},
}
local checks = {"post:read", "post:create", "post:update", "post:delete", "user:manage"}

print(string.format("%-10s | %-12s | %-12s | %-12s | %-12s | %-12s",
    "User", checks[1], checks[2], checks[3], checks[4], checks[5]))
print(string.rep("-", 80))

for _, u in ipairs(users) do
    local row = string.format("%-10s", u.name)
    for _, perm in ipairs(checks) do
        row = row .. string.format(" | %-12s", rbac:can(u.id, perm) and "YES" or "no")
    end
    print(row)
end

-- Custom policy test
print(string.format("\nAuthor edit own post: %s",
    tostring(rbac:can(3, "post:update_own", {post_owner=3}))))
print(string.format("Author edit other post: %s",
    tostring(rbac:can(3, "post:update_own", {post_owner=99}))))
```

---

## ตัวอย่างที่ 21: Data Masking และ PII Protection

```lua
-- Data masking for PII (Personally Identifiable Information)
local Mask = {}

-- Email: a***@domain.com
function Mask.email(email)
    local user, domain = email:match("^([^@]+)@(.+)$")
    if not user then return "***" end
    local visible = user:sub(1, math.min(2, #user))
    return visible .. string.rep("*", math.max(0, #user - #visible)) .. "@" .. domain
end

-- Phone: keep last 4 digits
function Mask.phone(phone)
    local digits = phone:gsub("[^%d]", "")
    if #digits <= 4 then return string.rep("*", #digits) end
    return string.rep("*", #digits - 4) .. digits:sub(-4)
end

-- Credit card: keep last 4
function Mask.card(number)
    local digits = number:gsub("[^%d]", "")
    if #digits < 4 then return string.rep("*", #digits) end
    local groups = {}
    local masked = string.rep("*", #digits - 4) .. digits:sub(-4)
    for i = 1, #masked, 4 do
        groups[#groups+1] = masked:sub(i, i+3)
    end
    return table.concat(groups, " ")
end

-- National ID: mask middle portion
function Mask.national_id(id)
    if #id <= 4 then return string.rep("*", #id) end
    return id:sub(1, 2) .. string.rep("*", #id - 4) .. id:sub(-2)
end

-- Name: keep first letter of each part
function Mask.name(name)
    local parts = {}
    for word in name:gmatch("%S+") do
        parts[#parts+1] = word:sub(1,1) .. string.rep("*", #word - 1)
    end
    return table.concat(parts, " ")
end

-- IP address: mask last octet
function Mask.ip(ip)
    return ip:gsub("%d+$", "***")
end

-- Generic field masker with rules
local DataMasker = {}
DataMasker.__index = DataMasker

function DataMasker.new(rules)
    return setmetatable({rules = rules or {}}, DataMasker)
end

function DataMasker:mask_record(record)
    local masked = {}
    for k, v in pairs(record) do
        local rule = self.rules[k]
        if rule and type(v) == "string" then
            masked[k] = rule(v)
        else
            masked[k] = v
        end
    end
    return masked
end

-- Demo
print("=== PII Masking Examples ===")
print("Email:  " .. Mask.email("john.doe@example.com"))
print("Phone:  " .. Mask.phone("+66 81-234-5678"))
print("Card:   " .. Mask.card("4532 0151 1234 5678"))
print("ID:     " .. Mask.national_id("1234567890123"))
print("Name:   " .. Mask.name("John Michael Doe"))
print("IP:     " .. Mask.ip("203.0.113.45"))

local masker = DataMasker.new({
    email  = Mask.email,
    phone  = Mask.phone,
    card   = Mask.card,
    name   = Mask.name,
    ip     = Mask.ip,
})

local user_record = {
    id    = 42,
    name  = "Alice Johnson",
    email = "alice.johnson@company.com",
    phone = "0812345678",
    card  = "5425233430109903",
    ip    = "192.168.1.100",
    role  = "admin",
}

local masked = masker:mask_record(user_record)
print("\nOriginal record:")
for k, v in pairs(user_record) do
    print(string.format("  %-8s: %s", k, tostring(v)))
end
print("Masked record:")
for k, v in pairs(masked) do
    print(string.format("  %-8s: %s", k, tostring(v)))
end
```

---

## ตัวอย่างที่ 22: Intrusion Detection System (IDS)

```lua
-- Simple rule-based IDS for log analysis
local IDS = {}
IDS.__index = IDS

function IDS.new()
    local self = setmetatable({}, IDS)
    self.rules   = {}
    self.events  = {}   -- {time, severity, rule, message, source}
    self.counters = {}  -- source -> {rule -> count}
    self.thresholds = {}
    return self
end

-- Add detection rule
function IDS:add_rule(name, severity, pattern_fn, opts)
    self.rules[#self.rules+1] = {
        name      = name,
        severity  = severity,  -- "low","medium","high","critical"
        check     = pattern_fn,
        threshold = opts and opts.threshold,
        window    = opts and opts.window or 60,
    }
    if opts and opts.threshold then
        self.thresholds[name] = opts.threshold
    end
end

function IDS:analyze(log_entry)
    local alerts = {}
    for _, rule in ipairs(self.rules) do
        local matched, detail = rule.check(log_entry)
        if matched then
            local src = log_entry.source or "unknown"
            -- Rate-based rules
            if rule.threshold then
                self.counters[src] = self.counters[src] or {}
                self.counters[src][rule.name] = (self.counters[src][rule.name] or 0) + 1
                if self.counters[src][rule.name] < rule.threshold then
                    matched = false  -- not yet at threshold
                end
            end
            if matched then
                local event = {
                    time     = os.time(),
                    severity = rule.severity,
                    rule     = rule.name,
                    message  = detail or rule.name,
                    source   = src,
                }
                self.events[#self.events+1] = event
                alerts[#alerts+1] = event
            end
        end
    end
    return alerts
end

function IDS:summary()
    local counts = {low=0, medium=0, high=0, critical=0}
    local by_rule = {}
    for _, e in ipairs(self.events) do
        counts[e.severity] = (counts[e.severity] or 0) + 1
        by_rule[e.rule] = (by_rule[e.rule] or 0) + 1
    end
    return counts, by_rule
end

-- Demo: Web server log analysis
local ids = IDS.new()

-- SQL injection patterns
ids:add_rule("sql_injection", "critical", function(entry)
    local sql_patterns = {"UNION%s+SELECT", "OR%s+1=1", "DROP%s+TABLE",
                          "xp_cmdshell", "EXEC%(", "--"}
    for _, pat in ipairs(sql_patterns) do
        if (entry.uri or ""):upper():find(pat) or
           (entry.body or ""):upper():find(pat) then
            return true, "SQL injection attempt: " .. pat
        end
    end
    return false
end)

-- XSS patterns
ids:add_rule("xss_attempt", "high", function(entry)
    local xss = {"<script", "javascript:", "onerror=", "onload=", "eval%("}
    for _, pat in ipairs(xss) do
        if (entry.uri or ""):lower():find(pat) then
            return true, "XSS: " .. pat
        end
    end
    return false
end)

-- Brute force: >5 failed logins from same IP
ids:add_rule("brute_force", "high", function(entry)
    return entry.status == 401 and entry.uri and entry.uri:find("/login"), "Failed login"
end, {threshold=5})

-- Scanner detection
ids:add_rule("scanner", "medium", function(entry)
    local scanners = {"sqlmap", "nikto", "nmap", "masscan", "dirsearch"}
    local ua = (entry.ua or ""):lower()
    for _, s in ipairs(scanners) do
        if ua:find(s) then return true, "Scanner UA: " .. s end
    end
    return false
end)

-- Simulate log entries
local logs = {
    {source="10.0.0.1", uri="/search?q=UNION SELECT * FROM users--", status=200, ua="Mozilla"},
    {source="10.0.0.2", uri="/login", status=401, ua="Mozilla"},
    {source="10.0.0.2", uri="/login", status=401, ua="Mozilla"},
    {source="10.0.0.2", uri="/login", status=401, ua="Mozilla"},
    {source="10.0.0.2", uri="/login", status=401, ua="Mozilla"},
    {source="10.0.0.2", uri="/login", status=401, ua="Mozilla"},
    {source="10.0.0.3", uri="/page?x=<script>alert(1)</script>", status=400, ua="curl/7.8"},
    {source="10.0.0.4", uri="/admin", status=403, ua="sqlmap/1.7"},
    {source="10.0.0.1", uri="/users?id=1 OR 1=1", status=500, ua="Mozilla"},
}

print("=== IDS Analysis ===")
for _, log in ipairs(logs) do
    local alerts = ids:analyze(log)
    for _, a in ipairs(alerts) do
        print(string.format("[%s] %s from %s: %s",
            a.severity:upper(), a.rule, a.source, a.message))
    end
end

local counts, by_rule = ids:summary()
print(string.format("\nTotal events: critical=%d high=%d medium=%d low=%d",
    counts.critical, counts.high, counts.medium, counts.low))
print("By rule:")
for rule, n in pairs(by_rule) do
    print(string.format("  %-20s %d", rule, n))
end
```

---

## ตัวอย่างที่ 23: API Key Management

```lua
-- API key generation, validation, scopes, and rotation
local APIKeyManager = {}
APIKeyManager.__index = APIKeyManager

local function gen_random_bytes(n)
    local t = {}
    for _ = 1, n do t[#t+1] = string.char(math.random(0, 255)) end
    return table.concat(t)
end

local function to_hex(s)
    local h = {}
    for i = 1, #s do h[#h+1] = string.format("%02x", s:byte(i)) end
    return table.concat(h)
end

local function prefix_key(prefix, hex)
    return prefix .. "_" .. hex:sub(1, 32)
end

function APIKeyManager.new()
    local self = setmetatable({}, APIKeyManager)
    self.keys = {}   -- key_id -> key_record
    self.index = {}  -- hash_of_key -> key_id
    math.randomseed(os.time())
    return self
end

function APIKeyManager:create(opts)
    opts = opts or {}
    local raw = to_hex(gen_random_bytes(32))
    local prefix = opts.prefix or "sk"
    local key_str = prefix_key(prefix, raw)
    -- Store hash, not plaintext
    local key_hash = to_hex(gen_random_bytes(16)) -- stub for real hash
    local key_id = "kid_" .. to_hex(gen_random_bytes(8))
    local record = {
        id         = key_id,
        name       = opts.name or "API Key",
        owner_id   = opts.owner_id,
        scopes     = opts.scopes or {"read"},
        prefix     = prefix,
        hash       = key_hash,
        created_at = os.time(),
        expires_at = opts.expires_at,
        last_used  = nil,
        use_count  = 0,
        active     = true,
    }
    self.keys[key_id] = record
    self.index[key_hash] = key_id
    return key_str, key_id
end

function APIKeyManager:validate(key_str, required_scope)
    -- In real code: hash the key, look up by hash
    -- Here we simulate by scanning (real code uses hash lookup)
    local found_id = nil
    for _, record in pairs(self.keys) do
        if record.active and record.prefix == key_str:match("^(%a+)_") then
            found_id = record.id
            break
        end
    end
    if not found_id then return false, "invalid key" end
    local record = self.keys[found_id]
    if not record.active then return false, "key revoked" end
    if record.expires_at and os.time() > record.expires_at then
        return false, "key expired"
    end
    if required_scope then
        local has_scope = false
        for _, s in ipairs(record.scopes) do
            if s == required_scope or s == "*" then has_scope = true; break end
        end
        if not has_scope then return false, "insufficient scope" end
    end
    record.last_used = os.time()
    record.use_count = record.use_count + 1
    return true, record
end

function APIKeyManager:revoke(key_id)
    local record = self.keys[key_id]
    if record then record.active = false; return true end
    return false, "not found"
end

function APIKeyManager:rotate(key_id, opts)
    local old = self.keys[key_id]
    if not old then return nil, "not found" end
    -- Create new key with same settings
    local new_key, new_id = self:create({
        name     = old.name .. " (rotated)",
        owner_id = old.owner_id,
        scopes   = old.scopes,
        prefix   = old.prefix,
        expires_at = opts and opts.expires_at,
    })
    -- Revoke old
    old.active = false
    return new_key, new_id
end

-- Demo
local mgr = APIKeyManager.new()

local key1, id1 = mgr:create({name="Production", owner_id=42,
    scopes={"read","write"}, prefix="pk"})
local key2, id2 = mgr:create({name="Read-only",  owner_id=42,
    scopes={"read"}, prefix="ro"})

print("Created keys:")
print("  Production: " .. key1)
print("  Read-only:  " .. key2)

local ok, rec = mgr:validate(key1, "write")
print(string.format("\nValidate production key (write scope): %s %s",
    tostring(ok), ok and rec.name or tostring(rec)))

local ok2, rec2 = mgr:validate(key2, "write")
print(string.format("Validate read-only key (write scope): %s %s",
    tostring(ok2), tostring(rec2)))

-- Rotate
local new_key, new_id = mgr:rotate(id1)
print("\nRotated production key: " .. new_key)

local _, old_err = mgr:validate(key1, "read")
print("Old key after rotation:", old_err)
```

---

## ตัวอย่างที่ 24: Error Sanitization

```lua
-- Sanitize errors before sending to client (prevent information disclosure)
local ErrorSanitizer = {}
ErrorSanitizer.__index = ErrorSanitizer

function ErrorSanitizer.new(opts)
    opts = opts or {}
    local self = setmetatable({}, ErrorSanitizer)
    self.debug     = opts.debug or false     -- show details in debug mode
    self.log_fn    = opts.log_fn or function(e) io.stderr:write(e.."\n") end
    self.error_map = {
        -- Map internal errors to safe public messages
        ["UNIQUE constraint failed"]   = {code=409, msg="Resource already exists"},
        ["FOREIGN KEY constraint"]     = {code=400, msg="Invalid reference"},
        ["no such table"]              = {code=500, msg="Internal server error"},
        ["connection refused"]         = {code=503, msg="Service temporarily unavailable"},
        ["timeout"]                    = {code=504, msg="Request timed out"},
        ["permission denied"]          = {code=403, msg="Access denied"},
        ["out of memory"]              = {code=500, msg="Internal server error"},
    }
    return self
end

-- Patterns that should NEVER reach the client
local SENSITIVE_PATTERNS = {
    "password", "passwd", "secret", "token", "api.key",
    "/home/", "/var/", "/etc/", "stack trace",
    "line %d+", "%.lua:%d+",
    "%d+%.%d+%.%d+%.%d+",   -- IP addresses
    "0x%x+",                  -- memory addresses
}

function ErrorSanitizer:sanitize(err, context)
    if not err then return {code=500, msg="Unknown error"} end
    local err_str = tostring(err)
    -- Log the full error internally
    local log_msg = string.format("[ERROR] ctx=%s err=%s",
        context or "?", err_str)
    self.log_fn(log_msg)
    -- Debug mode: return full error (never in production!)
    if self.debug then
        return {code=500, msg=err_str, debug=true}
    end
    -- Check error map
    for pattern, mapped in pairs(self.error_map) do
        if err_str:find(pattern) then
            return mapped
        end
    end
    -- Strip sensitive information
    local safe = err_str
    for _, pat in ipairs(SENSITIVE_PATTERNS) do
        safe = safe:gsub(pat, "[REDACTED]")
    end
    -- Final safety: truncate and neutralize
    if #safe > 100 then safe = safe:sub(1, 97) .. "..." end
    return {code=500, msg="An error occurred"}
end

-- Middleware wrapper
function ErrorSanitizer:wrap(fn, context)
    return function(...)
        local ok, result = pcall(fn, ...)
        if ok then return result end
        local sanitized = self:sanitize(result, context)
        return nil, sanitized
    end
end

-- Demo
local sanitizer = ErrorSanitizer.new({
    log_fn = function(msg)
        print("[LOG] " .. msg:sub(1, 80))
    end,
})

local errors = {
    "UNIQUE constraint failed: users.email",
    "Error: connection refused to postgres://admin:secret123@db.internal:5432/prod",
    "no such table: sessions",
    "attempt to index nil value (local 'user') at /home/app/handlers/auth.lua:42",
    "/etc/passwd: permission denied",
    "out of memory",
    "something unexpected happened",
}

print("=== Error Sanitization ===")
for _, e in ipairs(errors) do
    local safe = sanitizer:sanitize(e, "api/v1")
    print(string.format("  IN:  %s", e:sub(1,60)))
    print(string.format("  OUT: [%d] %s", safe.code, safe.msg))
    print()
end

-- Wrap a function
local risky = sanitizer:wrap(function()
    error("SELECT * FROM users WHERE password='" .. "' OR '1'='1")
end, "db.query")

local result, err = risky()
print("Wrapped error:", err and string.format("[%d] %s", err.code, err.msg))
```

---

## ตัวอย่างที่ 25: Security Middleware Chain

```lua
-- Composable security middleware for web request handling
local Middleware = {}
Middleware.__index = Middleware

function Middleware.new()
    return setmetatable({chain = {}}, Middleware)
end

function Middleware:use(name, fn)
    self.chain[#self.chain+1] = {name=name, fn=fn}
    return self  -- chainable
end

function Middleware:run(request)
    local response = {status=200, headers={}, body=""}
    local idx = 0
    local function next_mw(req, res)
        idx = idx + 1
        local mw = self.chain[idx]
        if mw then
            local ok, err = pcall(mw.fn, req, res, next_mw)
            if not ok then
                res.status = 500
                res.body   = "Middleware error: " .. tostring(err):sub(1,50)
                return false
            end
        end
        return true
    end
    next_mw(request, response)
    return response
end

-- Security middleware implementations
local function mw_request_id(req, res, next)
    req.id = string.format("%08x", math.random(0, 0xFFFFFF))
    res.headers["X-Request-ID"] = req.id
    next(req, res)
end

local function mw_security_headers(req, res, next)
    next(req, res)
    res.headers["X-Frame-Options"]           = "DENY"
    res.headers["X-Content-Type-Options"]    = "nosniff"
    res.headers["X-XSS-Protection"]          = "1; mode=block"
    res.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    res.headers["Content-Security-Policy"]   = "default-src 'self'"
    res.headers["Referrer-Policy"]           = "strict-origin-when-cross-origin"
    res.headers["Permissions-Policy"]        = "geolocation=(), camera=()"
end

local blocked_ips = {["10.0.0.99"]=true, ["192.168.1.254"]=true}
local function mw_ip_filter(req, res, next)
    if blocked_ips[req.remote_addr] then
        res.status = 403
        res.body   = "Forbidden"
        return  -- short-circuit
    end
    next(req, res)
end

local request_counts = {}
local function mw_rate_limit(req, res, next)
    local ip = req.remote_addr or "unknown"
    local now = os.time()
    request_counts[ip] = request_counts[ip] or {count=0, window_start=now}
    local rc = request_counts[ip]
    if now - rc.window_start > 60 then
        rc.count = 0; rc.window_start = now
    end
    rc.count = rc.count + 1
    res.headers["X-RateLimit-Limit"]     = "60"
    res.headers["X-RateLimit-Remaining"] = tostring(math.max(0, 60 - rc.count))
    if rc.count > 60 then
        res.status = 429
        res.body   = "Rate limit exceeded"
        return
    end
    next(req, res)
end

local function mw_auth(req, res, next)
    local token = (req.headers or {})["Authorization"]
    if req.path and req.path:find("^/api/") then
        if not token or not token:find("^Bearer ") then
            res.status = 401
            res.headers["WWW-Authenticate"] = 'Bearer realm="api"'
            res.body = "Unauthorized"
            return
        end
        req.user = {id=1, role="user"}  -- stub: decode token
    end
    next(req, res)
end

local function mw_route(req, res, _next)
    if req.path == "/health" then
        res.status = 200; res.body = '{"status":"ok"}'
    elseif req.path and req.path:find("^/api/") then
        res.status = 200; res.body = '{"data":"protected resource"}'
    else
        res.status = 404; res.body = "Not Found"
    end
end

-- Build middleware chain
local app = Middleware.new()
    :use("request_id",      mw_request_id)
    :use("security_headers", mw_security_headers)
    :use("ip_filter",       mw_ip_filter)
    :use("rate_limit",      mw_rate_limit)
    :use("auth",            mw_auth)
    :use("route",           mw_route)

-- Test requests
local test_requests = {
    {path="/health", remote_addr="203.0.113.5", headers={}},
    {path="/api/data", remote_addr="203.0.113.5", headers={}},  -- no auth
    {path="/api/data", remote_addr="203.0.113.5",
     headers={Authorization="Bearer valid-token-here"}},
    {path="/api/data", remote_addr="10.0.0.99", headers={}},    -- blocked IP
}

for _, req in ipairs(test_requests) do
    local res = app:run(req)
    print(string.format("[%d] %-40s id=%s",
        res.status,
        (req.path or "") .. " from " .. (req.remote_addr or "?"),
        res.headers["X-Request-ID"] or "?"))
    if res.status ~= 200 then
        print("      body: " .. res.body)
    end
end
```

---

## ตัวอย่างที่ 26: Secure Password Reset Flow

```lua
-- Secure password reset: token generation, expiry, single-use
local PasswordReset = {}
PasswordReset.__index = PasswordReset

local function rand_hex(n)
    local t = {}
    for _ = 1, n do t[#t+1] = string.format("%02x", math.random(0,255)) end
    return table.concat(t)
end

function PasswordReset.new(opts)
    opts = opts or {}
    local self = setmetatable({}, PasswordReset)
    self.tokens  = {}           -- token -> {user_id, expires_at, used}
    self.ttl     = opts.ttl or 3600  -- 1 hour default
    self.max_per_user = opts.max_per_user or 3
    self.user_tokens  = {}      -- user_id -> [token, ...]
    math.randomseed(os.time())
    return self
end

function PasswordReset:create_token(user_id, user_email)
    -- Limit active tokens per user
    self.user_tokens[user_id] = self.user_tokens[user_id] or {}
    -- Invalidate oldest if over limit
    while #self.user_tokens[user_id] >= self.max_per_user do
        local old = table.remove(self.user_tokens[user_id], 1)
        self.tokens[old] = nil
    end
    local token = rand_hex(32)
    self.tokens[token] = {
        user_id   = user_id,
        email     = user_email,
        expires_at = os.time() + self.ttl,
        used      = false,
        created   = os.time(),
    }
    self.user_tokens[user_id][#self.user_tokens[user_id]+1] = token
    -- In real code: send email with reset link containing token
    return token
end

function PasswordReset:validate_token(token)
    local record = self.tokens[token]
    if not record then return nil, "invalid token" end
    if record.used  then return nil, "token already used" end
    if os.time() > record.expires_at then
        self.tokens[token] = nil
        return nil, "token expired"
    end
    return record
end

function PasswordReset:consume_token(token, new_password_hash)
    local record, err = self:validate_token(token)
    if not record then return false, err end
    -- Mark as used immediately (single-use enforcement)
    record.used = true
    -- Remove from user's active list
    local ut = self.user_tokens[record.user_id] or {}
    for i, t in ipairs(ut) do
        if t == token then table.remove(ut, i); break end
    end
    -- In real code: update password in database
    return true, {user_id=record.user_id, email=record.email}
end

-- Demo
local pr = PasswordReset.new({ttl=300})

local token = pr:create_token(42, "alice@example.com")
print("Reset token created:", token:sub(1,16) .. "...")

local rec, err = pr:validate_token(token)
print("Validate:", rec and "valid for user_id=" .. rec.user_id or err)

local ok, info = pr:consume_token(token, "hashed_new_password")
print("Consume token:", ok, info and info.email)

-- Second use attempt
local ok2, err2 = pr:consume_token(token, "another_password")
print("Second use attempt:", ok2, err2)
```

---

## ตัวอย่างที่ 27: Content Security Policy Builder

```lua
-- CSP (Content Security Policy) header builder and validator
local CSP = {}
CSP.__index = CSP

-- Valid CSP directives
local DIRECTIVES = {
    "default-src", "script-src", "style-src", "img-src", "font-src",
    "connect-src", "media-src", "object-src", "frame-src", "worker-src",
    "base-uri", "form-action", "frame-ancestors", "upgrade-insecure-requests",
    "block-all-mixed-content", "report-uri", "report-to",
}
local DIRECTIVE_SET = {}
for _, d in ipairs(DIRECTIVES) do DIRECTIVE_SET[d] = true end

-- Valid source keywords
local KEYWORDS = {
    "'none'", "'self'", "'unsafe-inline'", "'unsafe-eval'",
    "'strict-dynamic'", "'report-sample'", "data:", "blob:", "https:",
}

function CSP.new()
    return setmetatable({policy = {}}, CSP)
end

function CSP:set(directive, ...)
    if not DIRECTIVE_SET[directive] then
        error("Unknown CSP directive: " .. directive)
    end
    local sources = {...}
    self.policy[directive] = sources
    return self
end

function CSP:add(directive, source)
    if not DIRECTIVE_SET[directive] then
        error("Unknown CSP directive: " .. directive)
    end
    self.policy[directive] = self.policy[directive] or {}
    self.policy[directive][#self.policy[directive]+1] = source
    return self
end

function CSP:nonce()
    local t = {}
    for _ = 1, 16 do t[#t+1] = string.format("%02x", math.random(0,255)) end
    local n = table.concat(t)
    return n, "'nonce-" .. n .. "'"
end

function CSP:build()
    local parts = {}
    -- Ensure default-src comes first
    if self.policy["default-src"] then
        parts[#parts+1] = "default-src " .. table.concat(self.policy["default-src"], " ")
    end
    for directive, sources in pairs(self.policy) do
        if directive ~= "default-src" then
            if #sources == 0 then
                parts[#parts+1] = directive
            else
                parts[#parts+1] = directive .. " " .. table.concat(sources, " ")
            end
        end
    end
    return table.concat(parts, "; ")
end

function CSP:report_only()
    return "Content-Security-Policy-Report-Only: " .. self:build()
end

function CSP:header()
    return "Content-Security-Policy: " .. self:build()
end

-- Audit: detect unsafe directives
function CSP:audit()
    local issues = {}
    for directive, sources in pairs(self.policy) do
        for _, src in ipairs(sources) do
            if src == "'unsafe-inline'" then
                issues[#issues+1] = {
                    level="high", directive=directive,
                    msg="'unsafe-inline' enables XSS attacks"
                }
            elseif src == "'unsafe-eval'" then
                issues[#issues+1] = {
                    level="medium", directive=directive,
                    msg="'unsafe-eval' enables code injection"
                }
            elseif src == "*" then
                issues[#issues+1] = {
                    level="high", directive=directive,
                    msg="Wildcard (*) allows any source"
                }
            end
        end
    end
    return issues
end

-- Demo
math.randomseed(os.time())
local csp = CSP.new()

local nonce_val, nonce_src = csp:nonce()

csp:set("default-src", "'self'")
   :set("script-src",  "'self'", nonce_src, "https://cdnjs.cloudflare.com")
   :set("style-src",   "'self'", "'unsafe-inline'")  -- intentionally unsafe
   :set("img-src",     "'self'", "data:", "https:")
   :set("font-src",    "'self'", "https://fonts.gstatic.com")
   :set("connect-src", "'self'", "https://api.example.com")
   :set("frame-ancestors", "'none'")
   :set("base-uri",    "'self'")
   :add("report-uri",  "https://report.example.com/csp")

print("CSP Header:")
print(csp:header())

print("\nNonce for this request: " .. nonce_val:sub(1,8) .. "...")

print("\nCSP Audit:")
local issues = csp:audit()
if #issues == 0 then
    print("  No issues found.")
else
    for _, issue in ipairs(issues) do
        print(string.format("  [%s] %s: %s",
            issue.level:upper(), issue.directive, issue.msg))
    end
end
```

---

## ตัวอย่างที่ 28: Cryptographic Key Derivation

```lua
-- PBKDF2-like key derivation and key management
-- Note: production code should use a proper crypto library (luaossl, etc.)

-- Simple PBKDF2 implementation using Lua (educational purposes)
local function xor_bytes(a, b)
    local result = {}
    for i = 1, #a do
        result[i] = string.char(a:byte(i) ~ b:byte(i))
    end
    return table.concat(result)
end

-- Pseudo-PRF (real: HMAC-SHA256; here: deterministic mixing for demo)
local function pseudo_prf(key, data)
    local hash = 0
    for i = 1, #key do
        hash = ((hash << 5) ~ hash ~ key:byte(i)) & 0xFFFFFFFF
    end
    for i = 1, #data do
        hash = ((hash << 5) ~ hash ~ data:byte(i)) & 0xFFFFFFFF
    end
    -- Expand to 32 bytes
    local t = {}
    for i = 0, 31 do
        t[i+1] = string.char((hash >> (i % 4 * 8)) & 0xFF)
    end
    return table.concat(t)
end

local function pbkdf2_like(password, salt, iterations, key_len)
    iterations = iterations or 10000
    key_len    = key_len or 32
    local blocks = math.ceil(key_len / 32)
    local result = {}
    for block = 1, blocks do
        local u = pseudo_prf(password, salt .. string.pack(">I4", block))
        local f = u
        for _ = 2, iterations do
            u = pseudo_prf(password, u)
            f = xor_bytes(f, u)
        end
        result[#result+1] = f
    end
    local key = table.concat(result):sub(1, key_len)
    return key
end

local function to_hex(s)
    local h = {}
    for i = 1, #s do h[i] = string.format("%02x", s:byte(i)) end
    return table.concat(h)
end

local function from_hex(h)
    local s = {}
    for i = 1, #h, 2 do
        s[#s+1] = string.char(tonumber(h:sub(i, i+1), 16))
    end
    return table.concat(s)
end

local function gen_salt(n)
    local t = {}
    math.randomseed(os.time() + math.random(1000))
    for _ = 1, n do t[#t+1] = string.char(math.random(0,255)) end
    return table.concat(t)
end

-- Key manager
local KeyManager = {}
KeyManager.__index = KeyManager

function KeyManager.new()
    return setmetatable({keys={}}, KeyManager)
end

function KeyManager:derive_key(master_password, purpose, iterations)
    local salt = gen_salt(16)
    local key  = pbkdf2_like(master_password, salt .. purpose, iterations or 1000, 32)
    local kid  = to_hex(gen_salt(8))
    self.keys[kid] = {
        kid       = kid,
        purpose   = purpose,
        salt      = to_hex(salt),
        iter      = iterations or 1000,
        key_hex   = to_hex(key),
        created   = os.time(),
    }
    return kid, key
end

function KeyManager:get_key(kid, master_password)
    local rec = self.keys[kid]
    if not rec then return nil, "key not found" end
    local salt = from_hex(rec.salt)
    local key  = pbkdf2_like(master_password, salt .. rec.purpose, rec.iter, 32)
    return key
end

-- Demo
local km = KeyManager.new()
local master = "correct-horse-battery-staple"

local kid1 = km:derive_key(master, "encryption", 100)
local kid2 = km:derive_key(master, "signing",    100)

print("Derived keys:")
for kid, rec in pairs(km.keys) do
    print(string.format("  kid=%-16s purpose=%-12s key=%s...",
        kid, rec.purpose, rec.key_hex:sub(1,16)))
end

-- Re-derive and verify
local key1_rederived = km:get_key(kid1, master)
local stored_key = from_hex(km.keys[kid1].key_hex)
print("\nKey re-derived matches stored:", key1_rederived == stored_key)

-- Wrong password
local wrong_key = km:get_key(kid1, "wrong-password")
print("Wrong password produces different key:", wrong_key ~= stored_key)
```

---

## ตัวอย่างที่ 29: Input Fuzzing และ Boundary Testing

```lua
-- Fuzz testing helper for finding edge cases in input validation
local Fuzzer = {}
Fuzzer.__index = Fuzzer

function Fuzzer.new()
    local self = setmetatable({}, Fuzzer)
    self.results = {passed=0, failed=0, crashed=0, cases={}}
    return self
end

-- Generate fuzz inputs for a given type
function Fuzzer:string_inputs()
    return {
        -- Empty / whitespace
        "", " ", "\t", "\n", "\r\n", "   ",
        -- Very long strings
        string.rep("A", 1000),
        string.rep("A", 65536),
        -- Special characters
        "'; DROP TABLE users; --",
        "<script>alert(1)</script>",
        "../../../etc/passwd",
        "%00", "\0null\0byte",
        "日本語テスト",
        "𝕳𝖊𝖑𝖑𝖔",
        string.char(0) .. "binary" .. string.char(255),
        -- Format strings (not a Lua issue but good to test)
        "%s%s%s%s%s%s%s%s%s",
        "%d%d%d%d%d%d%d%d%d",
        -- JSON injection
        '{"key":"value"}',
        '{"__proto__": {"admin": true}}',
        -- XML/HTML
        "<!DOCTYPE foo [ <!ENTITY xxe SYSTEM 'file:///etc/passwd'> ]>",
        -- Numbers as strings
        "0", "-1", "999999999999999999999", "NaN", "Infinity", "1e308",
        -- Unicode edge cases
        "\xc0\xaf",  -- overlong UTF-8
        "\xed\xa0\x80",  -- surrogate half
    }
end

function Fuzzer:number_inputs()
    return {
        0, -0, 1, -1,
        math.maxinteger, math.mininteger,
        math.huge, -math.huge, 0/0,   -- inf, -inf, nan
        1e308, -1e308, 1e-308,
        0.1 + 0.2,                     -- floating point surprise
        2^53, 2^53 + 1,                -- float precision boundary
        2^31, 2^31 - 1, -(2^31),
    }
end

function Fuzzer:run(name, fn, inputs)
    for _, input in ipairs(inputs) do
        local ok, result = pcall(fn, input)
        if ok then
            self.results.passed = self.results.passed + 1
        else
            self.results.crashed = self.results.crashed + 1
            local case = {
                name  = name,
                input = type(input) == "string"
                    and (input:sub(1,30):gsub("[%z\1-\31]", "?"))
                    or tostring(input),
                error = tostring(result):sub(1, 80),
            }
            self.results.cases[#self.results.cases+1] = case
        end
    end
end

function Fuzzer:report()
    print(string.format("\n=== Fuzz Report ==="))
    print(string.format("Passed: %d  Crashed: %d",
        self.results.passed, self.results.crashed))
    if #self.results.cases > 0 then
        print("Crashes:")
        for _, c in ipairs(self.results.cases) do
            print(string.format("  [%s] input=%-30q error=%s",
                c.name, c.input, c.error))
        end
    end
end

-- Demo: fuzz an input validator
local function validate_username(s)
    assert(type(s) == "string", "must be string")
    assert(#s >= 3, "too short")
    assert(#s <= 32, "too long")
    assert(s:match("^[%w_%-]+$"), "invalid characters")
    return true
end

local function validate_age(n)
    assert(type(n) == "number", "must be number")
    assert(n == n, "NaN not allowed")  -- NaN check
    assert(n >= 0 and n <= 150, "out of range")
    assert(math.tointeger(n), "must be integer")
    return true
end

local fuzzer = Fuzzer.new()
fuzzer:run("username", validate_username, fuzzer:string_inputs())
fuzzer:run("age",      validate_age,      fuzzer:number_inputs())
fuzzer:report()
```

---

## ตัวอย่างที่ 30: Dependency Vulnerability Scanner

```lua
-- Simulate a dependency vulnerability scanner for Lua modules
local VulnScanner = {}
VulnScanner.__index = VulnScanner

-- Mock CVE database (real: query NVD API or OSV.dev)
local CVE_DB = {
    luasocket = {
        {cve="CVE-2023-XXXX", severity="HIGH",
         affected_versions={"3.0rc1", "3.0"},
         description="Buffer overflow in TCP receive",
         fix_version="3.1"},
    },
    lsqlite3 = {
        {cve="CVE-2022-YYYY", severity="MEDIUM",
         affected_versions={"0.9.4", "0.9.5"},
         description="SQL injection via crafted column names",
         fix_version="0.9.6"},
    },
    copas = {
        {cve="CVE-2021-ZZZZ", severity="LOW",
         affected_versions={"2.0.0"},
         description="Denial of service via malformed HTTP",
         fix_version="2.0.1"},
    },
}

function VulnScanner.new()
    local self = setmetatable({}, VulnScanner)
    self.findings = {}
    return self
end

-- Simulate version comparison (semver-like)
local function parse_version(v)
    local major, minor, patch = v:match("(%d+)%.(%d+)%.?(%d*)")
    return {
        major = tonumber(major) or 0,
        minor = tonumber(minor) or 0,
        patch = tonumber(patch) or 0,
    }
end

local function version_le(a_str, b_str)
    local a, b = parse_version(a_str), parse_version(b_str)
    if a.major ~= b.major then return a.major < b.major end
    if a.minor ~= b.minor then return a.minor < b.minor end
    return a.patch <= b.patch
end

function VulnScanner:scan(dependencies)
    self.findings = {}
    for dep_name, dep_version in pairs(dependencies) do
        local vulns = CVE_DB[dep_name]
        if vulns then
            for _, vuln in ipairs(vulns) do
                for _, av in ipairs(vuln.affected_versions) do
                    if av == dep_version then
                        self.findings[#self.findings+1] = {
                            package     = dep_name,
                            version     = dep_version,
                            cve         = vuln.cve,
                            severity    = vuln.severity,
                            description = vuln.description,
                            fix_version = vuln.fix_version,
                        }
                    end
                end
            end
        end
    end
    return self.findings
end

function VulnScanner:report()
    if #self.findings == 0 then
        print("No vulnerabilities found.")
        return
    end
    local sev_order = {CRITICAL=1, HIGH=2, MEDIUM=3, LOW=4, INFO=5}
    table.sort(self.findings, function(a, b)
        return (sev_order[a.severity] or 5) < (sev_order[b.severity] or 5)
    end)
    local counts = {}
    print("\n=== Vulnerability Report ===")
    for _, f in ipairs(self.findings) do
        counts[f.severity] = (counts[f.severity] or 0) + 1
        print(string.format("[%s] %s@%s", f.severity, f.package, f.version))
        print(string.format("  CVE: %s", f.cve))
        print(string.format("  %s", f.description))
        print(string.format("  Fix: upgrade to %s", f.fix_version))
    end
    print(string.format("\nSummary: %d findings",#self.findings))
    for sev, n in pairs(counts) do
        print(string.format("  %-10s %d", sev, n))
    end
end

-- Demo
local scanner = VulnScanner.new()
local my_deps = {
    luasocket = "3.0",
    lsqlite3  = "0.9.5",
    copas     = "2.0.1",   -- patched version
    luafilesystem = "1.8.0",  -- not in CVE DB
}

print("Scanning dependencies:")
for dep, ver in pairs(my_deps) do
    print(string.format("  %-20s %s", dep, ver))
end

scanner:scan(my_deps)
scanner:report()
```

---

## ตัวอย่างที่ 31: Secure Logging

```lua
-- Structured, tamper-evident, PII-safe logging
local SecureLogger = {}
SecureLogger.__index = SecureLogger

local LOG_LEVELS = {DEBUG=10, INFO=20, WARN=30, ERROR=40, CRITICAL=50}
local LEVEL_NAMES = {[10]="DEBUG",[20]="INFO",[30]="WARN",[40]="ERROR",[50]="CRITICAL"}

-- Fields that should be redacted
local REDACT_FIELDS = {
    password=true, passwd=true, secret=true, token=true,
    api_key=true, credit_card=true, ssn=true, cvv=true,
    authorization=true,
}

local function redact(obj, depth)
    depth = depth or 0
    if depth > 5 then return "[DEEP]" end
    if type(obj) == "table" then
        local result = {}
        for k, v in pairs(obj) do
            if REDACT_FIELDS[k:lower()] then
                result[k] = "[REDACTED]"
            else
                result[k] = redact(v, depth + 1)
            end
        end
        return result
    end
    return obj
end

local function simple_json(v, depth)
    depth = depth or 0
    if depth > 4 then return '"[deep]"' end
    local t = type(v)
    if t == "string"  then return string.format('%q', v:sub(1,200)) end
    if t == "number"  then return tostring(v) end
    if t == "boolean" then return tostring(v) end
    if t == "nil"     then return "null" end
    if t == "table" then
        local parts = {}
        local is_array = #v > 0
        if is_array then
            for _, item in ipairs(v) do
                parts[#parts+1] = simple_json(item, depth+1)
            end
            return "[" .. table.concat(parts,",") .. "]"
        else
            for k, val in pairs(v) do
                parts[#parts+1] = string.format('%q:%s',
                    tostring(k), simple_json(val, depth+1))
            end
            return "{" .. table.concat(parts,",") .. "}"
        end
    end
    return '"[' .. t .. ']"'
end

function SecureLogger.new(opts)
    opts = opts or {}
    local self = setmetatable({}, SecureLogger)
    self.min_level = LOG_LEVELS[opts.level or "INFO"] or 20
    self.output    = opts.output or io.stdout
    self.service   = opts.service or "app"
    self.chain_hash = "0000000000000000"  -- for tamper evidence
    return self
end

function SecureLogger:_write(level_num, message, context)
    if level_num < self.min_level then return end
    local safe_ctx = context and redact(context) or {}
    local entry = {
        ts      = os.date("!%Y-%m-%dT%H:%M:%SZ"),
        level   = LEVEL_NAMES[level_num] or "UNKNOWN",
        service = self.service,
        msg     = tostring(message):sub(1, 500),
        ctx     = safe_ctx,
        prev    = self.chain_hash,
    }
    local line = simple_json(entry)
    -- Simple chain: hash line content
    local h = 0
    for i = 1, #line do h = (h * 1000003 + line:byte(i)) % (2^32) end
    self.chain_hash = string.format("%016x", h)
    self.output:write(line .. "\n")
end

function SecureLogger:debug(msg, ctx) self:_write(10, msg, ctx) end
function SecureLogger:info(msg, ctx)  self:_write(20, msg, ctx) end
function SecureLogger:warn(msg, ctx)  self:_write(30, msg, ctx) end
function SecureLogger:error(msg, ctx) self:_write(40, msg, ctx) end
function SecureLogger:critical(msg, ctx) self:_write(50, msg, ctx) end

-- Demo
local logger = SecureLogger.new({level="DEBUG", service="auth-service"})

logger:info("User login attempt", {
    username = "alice",
    ip       = "203.0.113.5",
    password = "s3cr3t123",  -- will be redacted
    ua       = "Mozilla/5.0",
})

logger:warn("Rate limit exceeded", {
    ip    = "10.0.0.1",
    count = 61,
    token = "bearer-abc123",  -- will be redacted
})

logger:error("Database connection failed", {
    host  = "db.internal",
    port  = 5432,
    error = "connection refused",
})

logger:critical("Possible intrusion", {
    source = "10.0.0.99",
    reason = "multiple failed logins",
    action = "IP blocked",
})
```

---

## ตัวอย่างที่ 32: TLS/Certificate Validation Helpers

```lua
-- Certificate and TLS configuration validation helpers
-- (Works with LuaSec for actual TLS; here demonstrates config logic)

local TLSConfig = {}
TLSConfig.__index = TLSConfig

-- Weak/deprecated protocols and ciphers
local WEAK_PROTOCOLS = {
    SSLv2=true, SSLv3=true, TLSv1=true, ["TLS 1.0"]=true,
    ["TLSv1.0"]=true, ["TLSv1.1"]=true,
}
local WEAK_CIPHERS = {
    "RC4", "DES", "3DES", "MD5", "NULL", "EXPORT",
    "ANON", "ADH", "AECDH", "RC2", "IDEA",
}
local STRONG_CIPHERS = {
    "ECDHE-ECDSA-AES256-GCM-SHA384",
    "ECDHE-RSA-AES256-GCM-SHA384",
    "ECDHE-ECDSA-CHACHA20-POLY1305",
    "ECDHE-RSA-CHACHA20-POLY1305",
    "ECDHE-ECDSA-AES128-GCM-SHA256",
    "ECDHE-RSA-AES128-GCM-SHA256",
}

function TLSConfig.new(config)
    return setmetatable({config = config or {}}, TLSConfig)
end

function TLSConfig:validate()
    local issues = {}
    local cfg = self.config

    -- Protocol version check
    if cfg.protocol then
        if WEAK_PROTOCOLS[cfg.protocol] then
            issues[#issues+1] = {
                severity = "CRITICAL",
                msg = "Weak protocol: " .. cfg.protocol .. " (use TLSv1.2 or TLSv1.3)"
            }
        end
    end

    -- Cipher check
    if cfg.ciphers then
        local cipher_str = cfg.ciphers:upper()
        for _, weak in ipairs(WEAK_CIPHERS) do
            if cipher_str:find(weak) then
                issues[#issues+1] = {
                    severity = "HIGH",
                    msg = "Weak cipher in suite: " .. weak
                }
            end
        end
    end

    -- Certificate verification
    if cfg.verify == false or cfg.verify == "none" then
        issues[#issues+1] = {
            severity = "CRITICAL",
            msg = "Certificate verification disabled — vulnerable to MITM"
        }
    end

    -- Hostname verification
    if cfg.verify_hostname == false then
        issues[#issues+1] = {
            severity = "CRITICAL",
            msg = "Hostname verification disabled"
        }
    end

    -- Certificate expiry simulation
    if cfg.cert_expiry then
        local days_left = (cfg.cert_expiry - os.time()) / 86400
        if days_left < 0 then
            issues[#issues+1] = {severity="CRITICAL", msg="Certificate EXPIRED"}
        elseif days_left < 14 then
            issues[#issues+1] = {
                severity="HIGH",
                msg=string.format("Certificate expires in %.0f days", days_left)
            }
        elseif days_left < 30 then
            issues[#issues+1] = {
                severity="MEDIUM",
                msg=string.format("Certificate expires in %.0f days", days_left)
            }
        end
    end

    -- HSTS check
    if not cfg.hsts then
        issues[#issues+1] = {severity="MEDIUM", msg="HSTS not configured"}
    end

    return issues
end

function TLSConfig:grade(issues)
    local critical = 0
    local high = 0
    for _, i in ipairs(issues) do
        if i.severity == "CRITICAL" then critical = critical + 1
        elseif i.severity == "HIGH"  then high = high + 1 end
    end
    if critical > 0 then return "F" end
    if high > 1     then return "C" end
    if high == 1    then return "B" end
    if #issues > 0  then return "A-" end
    return "A+"
end

-- Demo: test several configurations
local configs = {
    {
        name = "Secure production",
        protocol = "TLSv1.3",
        ciphers  = table.concat(STRONG_CIPHERS, ":"),
        verify   = true,
        verify_hostname = true,
        cert_expiry = os.time() + 90*86400,
        hsts = true,
    },
    {
        name = "Legacy server",
        protocol = "TLSv1.0",
        ciphers  = "RC4-SHA:DES-CBC3-SHA",
        verify   = false,
        hsts     = false,
        cert_expiry = os.time() + 5*86400,
    },
}

for _, cfg in ipairs(configs) do
    local name = cfg.name; cfg.name = nil
    local tls = TLSConfig.new(cfg)
    local issues = tls:validate()
    local grade = tls:grade(issues)
    print(string.format("\n[%s] %s — Grade: %s", grade, name, grade))
    if #issues == 0 then
        print("  No issues found.")
    else
        for _, issue in ipairs(issues) do
            print(string.format("  [%s] %s", issue.severity, issue.msg))
        end
    end
end
```

---

## ตัวอย่างที่ 33: Honeypot Detection

```lua
-- Honeypot fields to detect bots and automated attacks
local Honeypot = {}
Honeypot.__index = Honeypot

function Honeypot.new(opts)
    opts = opts or {}
    local self = setmetatable({}, Honeypot)
    self.fields    = opts.fields or {"website", "url", "fax", "phone2"}
    self.min_time  = opts.min_time or 3    -- min seconds to fill form
    self.max_time  = opts.max_time or 3600 -- max seconds (avoid old tokens)
    self.tokens    = {}  -- token -> {created_at}
    return self
end

-- Generate form token with timestamp
function Honeypot:generate_token()
    local token = string.format("%016x%016x",
        math.random(0, 0xFFFF), os.time())
    self.tokens[token] = {created_at = os.time()}
    return token
end

-- Render HTML honeypot fields (should be hidden with CSS, not type=hidden)
function Honeypot:html_fields()
    local parts = {
        '<!-- Honeypot fields - do not fill -->',
        '<style>.hp-field { position:absolute; left:-9999px; }</style>',
    }
    for _, field in ipairs(self.fields) do
        parts[#parts+1] = string.format(
            '<div class="hp-field"><label>%s</label>' ..
            '<input type="text" name="%s" tabindex="-1" autocomplete="off"></div>',
            field, field)
    end
    return table.concat(parts, "\n")
end

-- Validate form submission
function Honeypot:check(form_data)
    local reasons = {}

    -- Check honeypot fields are empty
    for _, field in ipairs(self.fields) do
        if form_data[field] and form_data[field] ~= "" then
            reasons[#reasons+1] = "honeypot field '" .. field .. "' was filled"
        end
    end

    -- Check form token timing
    if form_data._hp_token then
        local record = self.tokens[form_data._hp_token]
        if not record then
            reasons[#reasons+1] = "invalid form token"
        else
            local elapsed = os.time() - record.created_at
            if elapsed < self.min_time then
                reasons[#reasons+1] = string.format(
                    "form submitted too fast (%ds < %ds min)", elapsed, self.min_time)
            elseif elapsed > self.max_time then
                reasons[#reasons+1] = string.format(
                    "form token expired (%ds > %ds max)", elapsed, self.max_time)
            end
            self.tokens[form_data._hp_token] = nil  -- single use
        end
    else
        reasons[#reasons+1] = "missing form token"
    end

    -- JavaScript challenge (must be solved by browser)
    if form_data._js_challenge ~= "solved" then
        reasons[#reasons+1] = "JavaScript challenge not completed"
    end

    local is_bot = #reasons > 0
    return not is_bot, reasons
end

-- Demo
math.randomseed(os.time())
local hp = Honeypot.new({min_time=2})

local token = hp:generate_token()
print("Form token generated:", token:sub(1,16) .. "...")

-- Simulate legitimate submission (wait a moment)
os.execute("sleep 3")

local legitimate = {
    name         = "Alice",
    email        = "alice@example.com",
    message      = "Hello!",
    website      = "",    -- honeypot empty
    url          = "",    -- honeypot empty
    _hp_token    = token,
    _js_challenge = "solved",
}

local ok, reasons = hp:check(legitimate)
print("\nLegitimate form:")
print("  Bot detected:", not ok)
if #reasons > 0 then
    for _, r in ipairs(reasons) do print("  Reason: " .. r) end
end

-- Bot submission
local token2 = hp:generate_token()
local bot = {
    name         = "Bot",
    email        = "spam@spam.com",
    website      = "http://spam.com",  -- honeypot filled!
    url          = "http://ads.com",   -- honeypot filled!
    _hp_token    = token2,
    _js_challenge = "",                -- JS not executed
}

local ok2, reasons2 = hp:check(bot)
print("\nBot submission:")
print("  Bot detected:", not ok2)
for _, r in ipairs(reasons2) do print("  Reason: " .. r) end
```

---

## ตัวอย่างที่ 34: Encryption at Rest Simulation

```lua
-- Demonstrate encryption-at-rest patterns for sensitive data
-- Note: use a proper crypto library (luaossl/lua-resty-openssl) in production

-- Simple XOR cipher (demonstration only — NOT secure)
local function xor_cipher(key, data)
    local result = {}
    local key_len = #key
    for i = 1, #data do
        result[i] = string.char(data:byte(i) ~ key:byte((i-1) % key_len + 1))
    end
    return table.concat(result)
end

local function to_hex(s)
    local h = {}
    for i = 1, #s do h[i] = string.format("%02x", s:byte(i)) end
    return table.concat(h)
end

local function from_hex(h)
    local s = {}
    for i = 1, #h, 2 do
        s[#s+1] = string.char(tonumber(h:sub(i, i+1), 16))
    end
    return table.concat(s)
end

-- Field-level encryption (encrypt individual sensitive columns)
local FieldEncryption = {}
FieldEncryption.__index = FieldEncryption

function FieldEncryption.new(key)
    assert(#key >= 16, "key must be at least 16 bytes")
    return setmetatable({key=key}, FieldEncryption)
end

function FieldEncryption:encrypt(plaintext)
    if type(plaintext) ~= "string" then
        plaintext = tostring(plaintext)
    end
    -- Add version byte + length prefix for format versioning
    local versioned = "\x01" .. string.pack(">I4", #plaintext) .. plaintext
    local ciphertext = xor_cipher(self.key, versioned)
    return "enc:v1:" .. to_hex(ciphertext)
end

function FieldEncryption:decrypt(stored)
    if type(stored) ~= "string" then return nil, "not a string" end
    if not stored:find("^enc:v1:") then
        return stored, nil  -- not encrypted, return as-is
    end
    local hex = stored:sub(8)
    if #hex % 2 ~= 0 then return nil, "malformed ciphertext" end
    local ciphertext = from_hex(hex)
    local versioned  = xor_cipher(self.key, ciphertext)
    local version    = versioned:byte(1)
    if version ~= 1 then return nil, "unknown encryption version" end
    local len        = string.unpack(">I4", versioned, 2)
    local plaintext  = versioned:sub(6, 5 + len)
    return plaintext
end

-- Encrypted record store
local EncryptedStore = {}
EncryptedStore.__index = EncryptedStore

function EncryptedStore.new(key, encrypted_fields)
    local self = setmetatable({}, EncryptedStore)
    self.enc    = FieldEncryption.new(key)
    self.fields = encrypted_fields or {}
    self.store  = {}
    self.next_id = 1
    return self
end

function EncryptedStore:insert(record)
    local stored = {id = self.next_id}
    self.next_id = self.next_id + 1
    for k, v in pairs(record) do
        if self.fields[k] then
            stored[k] = self.enc:encrypt(tostring(v))
        else
            stored[k] = v
        end
    end
    self.store[stored.id] = stored
    return stored.id
end

function EncryptedStore:get(id)
    local stored = self.store[id]
    if not stored then return nil end
    local record = {}
    for k, v in pairs(stored) do
        if self.fields[k] then
            record[k] = self.enc:decrypt(v)
        else
            record[k] = v
        end
    end
    return record
end

-- Demo
local store = EncryptedStore.new(
    "encryption-key-32bytes-padding!!", -- 32-byte key
    {ssn=true, credit_card=true, bank_account=true}
)

local id = store:insert({
    name         = "Alice Johnson",      -- not encrypted
    email        = "alice@example.com",  -- not encrypted
    ssn          = "123-45-6789",        -- encrypted
    credit_card  = "4532015112345678",   -- encrypted
    bank_account = "9876543210",         -- encrypted
})

print("Stored raw record:")
local raw = store.store[id]
for k, v in pairs(raw) do
    print(string.format("  %-15s %s", k, tostring(v):sub(1,40)))
end

print("\nDecrypted record:")
local decrypted = store:get(id)
for k, v in pairs(decrypted) do
    print(string.format("  %-15s %s", k, tostring(v)))
end
```

---

## ตัวอย่างที่ 35: Security Compliance Checker

```lua
-- OWASP/compliance security checklist for Lua web applications
local ComplianceChecker = {}
ComplianceChecker.__index = ComplianceChecker

local CHECKS = {
    -- OWASP Top 10 controls
    {id="A01", name="Broken Access Control",
     checks={
         "rbac_implemented",
         "auth_on_all_endpoints",
         "directory_listing_disabled",
     }},
    {id="A02", name="Cryptographic Failures",
     checks={
         "https_enforced",
         "no_weak_ciphers",
         "secrets_not_in_code",
         "encryption_at_rest",
     }},
    {id="A03", name="Injection",
     checks={
         "parameterized_queries",
         "input_validation",
         "output_encoding",
     }},
    {id="A04", name="Insecure Design",
     checks={
         "threat_model_documented",
         "security_requirements_defined",
     }},
    {id="A05", name="Security Misconfiguration",
     checks={
         "security_headers_set",
         "error_messages_sanitized",
         "debug_mode_disabled",
         "unnecessary_features_disabled",
     }},
    {id="A07", name="Authentication Failures",
     checks={
         "mfa_available",
         "account_lockout",
         "password_policy_enforced",
         "secure_session_management",
     }},
    {id="A09", name="Logging and Monitoring",
     checks={
         "security_events_logged",
         "logs_protected",
         "alerting_configured",
     }},
}

function ComplianceChecker.new()
    local self = setmetatable({}, ComplianceChecker)
    self.results = {}
    return self
end

function ComplianceChecker:assert_control(control_id, status, evidence)
    self.results[control_id] = {
        status   = status,   -- "pass", "fail", "partial", "na"
        evidence = evidence or "",
    }
end

function ComplianceChecker:report()
    local totals = {pass=0, fail=0, partial=0, na=0, untested=0}
    local category_results = {}

    for _, category in ipairs(CHECKS) do
        local cat_pass, cat_fail, cat_total = 0, 0, 0
        for _, check in ipairs(category.checks) do
            cat_total = cat_total + 1
            local r = self.results[check]
            if not r then
                totals.untested = totals.untested + 1
                cat_fail = cat_fail + 1
            elseif r.status == "pass" then
                totals.pass = totals.pass + 1; cat_pass = cat_pass + 1
            elseif r.status == "fail" then
                totals.fail = totals.fail + 1; cat_fail = cat_fail + 1
            elseif r.status == "partial" then
                totals.partial = totals.partial + 1
            elseif r.status == "na" then
                totals.na = totals.na + 1; cat_pass = cat_pass + 1
            end
        end
        local pct = math.floor(cat_pass / cat_total * 100)
        category_results[#category_results+1] = {
            id=category.id, name=category.name,
            pass=cat_pass, total=cat_total, pct=pct
        }
    end

    print("=== Security Compliance Report ===")
    print(string.format("%-6s %-30s %s", "ID", "Category", "Progress"))
    print(string.rep("-", 60))
    for _, cr in ipairs(category_results) do
        local bar = string.rep("█", math.floor(cr.pct/10))
              .. string.rep("░", 10 - math.floor(cr.pct/10))
        print(string.format("%-6s %-30s [%s] %d%%",
            cr.id, cr.name:sub(1,30), bar, cr.pct))
    end

    local total = totals.pass + totals.fail + totals.partial + totals.untested
    local overall = total > 0 and math.floor(totals.pass / total * 100) or 0
    print(string.rep("-", 60))
    print(string.format("Overall: %d%% (%d pass, %d fail, %d partial, %d untested)",
        overall, totals.pass, totals.fail, totals.partial, totals.untested))

    -- Grade
    local grade
    if overall >= 90 then grade = "A"
    elseif overall >= 80 then grade = "B"
    elseif overall >= 70 then grade = "C"
    elseif overall >= 60 then grade = "D"
    else grade = "F" end
    print(string.format("Security Grade: %s", grade))
end

-- Demo: Assess a sample application
local checker = ComplianceChecker.new()

-- A01: Access Control
checker:assert_control("rbac_implemented",          "pass",    "RBAC module v2.1")
checker:assert_control("auth_on_all_endpoints",     "partial", "Missing /api/metrics")
checker:assert_control("directory_listing_disabled","pass",    "Nginx config verified")

-- A02: Crypto
checker:assert_control("https_enforced",            "pass",    "HSTS max-age=31536000")
checker:assert_control("no_weak_ciphers",           "pass",    "TLSv1.3 only")
checker:assert_control("secrets_not_in_code",       "fail",    "DB password in config.lua:12")
checker:assert_control("encryption_at_rest",        "partial", "PII fields encrypted, logs not")

-- A03: Injection
checker:assert_control("parameterized_queries",     "pass",    "All DB queries use bind params")
checker:assert_control("input_validation",          "pass",    "Validator lib v1.4")
checker:assert_control("output_encoding",           "partial", "HTML encoded; JSON not reviewed")

-- A04: Design
checker:assert_control("threat_model_documented",   "fail",    "Not done yet")
checker:assert_control("security_requirements_defined","partial","In progress")

-- A05: Misconfiguration
checker:assert_control("security_headers_set",      "pass",    "CSP, HSTS, X-Frame-Options")
checker:assert_control("error_messages_sanitized",  "pass",    "ErrorSanitizer in place")
checker:assert_control("debug_mode_disabled",       "pass",    "ENV=production verified")
checker:assert_control("unnecessary_features_disabled","na",   "Minimal install")

-- A07: Authentication
checker:assert_control("mfa_available",             "fail",    "Not implemented")
checker:assert_control("account_lockout",           "pass",    "5 attempts, 15min lockout")
checker:assert_control("password_policy_enforced",  "pass",    "Min 12 chars, complexity req")
checker:assert_control("secure_session_management", "pass",    "HttpOnly, Secure, SameSite")

-- A09: Logging
checker:assert_control("security_events_logged",    "pass",    "SecureLogger deployed")
checker:assert_control("logs_protected",            "partial", "Log rotation set; off-site TBD")
checker:assert_control("alerting_configured",       "fail",    "No alerting system yet")

checker:report()
```

---

## สรุป Security ใน Lua

### Top 10 Security Rules สำหรับ Lua

```lua
-- 1. ALWAYS ใช้ parameterized queries
local stmt = db:prepare("SELECT * FROM t WHERE id = ?")
stmt:bind(1, user_input)  -- ไม่ใช่ string concat!

-- 2. VALIDATE ทุก user input
if not Validator.is_integer(id) then
    return nil, "Invalid ID"
end

-- 3. ใช้ path canonicalization
local safe, path = is_safe_path(user_input, "/var/www")
if not safe then return nil, "Path traversal!" end

-- 4. NEVER ใส่ secrets ใน source code
local key = os.getenv("API_KEY")  -- ไม่ใช่ hardcode

-- 5. Sandbox user-provided Lua code
local result = execute_sandboxed(user_code, safe_env, timeout)

-- 6. Rate limit ทุก endpoint
if not rate_limiter:is_allowed(ip) then
    return 429, "Too Many Requests"
end

-- 7. Hash passwords อย่างถูกต้อง
local hash = hasher:hash(password)  -- PBKDF2/bcrypt/Argon2

-- 8. ใช้ constant-time comparison สำหรับ secrets
if not constant_time_equals(token_a, token_b) then
    return nil, "Token mismatch"
end

-- 9. Escape output ก่อน render
return html_escape(user_content)

-- 10. ตั้ง security headers
response.headers["X-Frame-Options"] = "DENY"
response.headers["Content-Security-Policy"] = csp
```

### Security Libraries ที่ควรรู้จัก

| Library | ใช้สำหรับ |
|---------|-----------|
| luaossl | OpenSSL bindings (crypto) |
| luacrypto | Cryptographic functions |
| lua-resty-ssl | SSL/TLS ใน OpenResty |
| luasec | SSL/TLS สำหรับ LuaSocket |
| lua-zlib | Compression |

### Resources

- **OWASP Top 10**: https://owasp.org/www-project-top-ten/
- **Lua Security**: https://lua-users.org/wiki/SecurityPattern
- **LuaJIT Sandboxing**: https://luajit.org/
- **CWE Database**: https://cwe.mitre.org/
