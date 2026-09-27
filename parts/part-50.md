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
