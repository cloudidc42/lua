# บทที่ 75: Cryptography กับ Lua

## บทนำ

Cryptography (วิทยาการเข้ารหัส) เป็นศาสตร์ที่เกี่ยวกับการปกป้องข้อมูลด้วยเทคนิคทางคณิตศาสตร์ ใน Lua มีหลาย library ที่รองรับการทำงานด้านนี้ เช่น luaossl, luacrypto, และ lua-openssl

## พื้นฐาน Cryptography

```lua
-- แนวคิดพื้นฐานของ Cryptography

local crypto_concepts = {
    confidentiality = "ข้อมูลอ่านได้เฉพาะผู้มีสิทธิ์เท่านั้น",
    integrity = "ข้อมูลไม่ถูกแก้ไขโดยไม่ได้รับอนุญาต",
    authentication = "ยืนยันตัวตนของผู้ส่งและผู้รับ",
    non_repudiation = "ผู้ส่งไม่สามารถปฏิเสธว่าส่งข้อมูลนั้น",
    availability = "ข้อมูลพร้อมใช้งานเมื่อต้องการ",
}

print("=== Cryptography Principles (CIA+) ===")
for prop, desc in pairs(crypto_concepts) do
    print(string.format("  %-20s: %s", prop, desc))
end

-- ประเภทของ Cryptography
local crypto_types = {
    {name = "Symmetric Encryption", example = "AES, DES, 3DES, ChaCha20",
     desc = "ใช้ key เดียวกัน encrypt และ decrypt"},
    {name = "Asymmetric Encryption", example = "RSA, ECC, ElGamal",
     desc = "ใช้ public key encrypt, private key decrypt"},
    {name = "Hash Functions", example = "SHA-256, SHA-512, Blake2",
     desc = "one-way function ไม่สามารถ reverse ได้"},
    {name = "MAC/HMAC", example = "HMAC-SHA256, CMAC",
     desc = "ตรวจสอบ integrity และ authenticity"},
    {name = "Digital Signatures", example = "RSA-SHA256, ECDSA",
     desc = "ลายเซ็นดิจิทัล"},
    {name = "Key Exchange", example = "Diffie-Hellman, ECDH",
     desc = "แลกเปลี่ยน key อย่างปลอดภัย"},
}

print("\nTypes of Cryptography:")
for _, t in ipairs(crypto_types) do
    print(string.format("\n  %s", t.name))
    print(string.format("    Examples: %s", t.example))
    print(string.format("    Purpose:  %s", t.desc))
end
```

## Hash Functions

### MD5 (ไม่ปลอดภัยแล้ว)

```lua
-- MD5 - Message Digest 5
-- ⚠️  WARNING: MD5 ถือว่าไม่ปลอดภัยแล้วสำหรับ security purposes
-- ใช้ได้เฉพาะ checksum, non-security purposes

-- MD5 implementation (pure Lua, simplified)
local md5 = {}

-- Constants
local function md5_sinconst()
    local t = {}
    for i = 1, 64 do
        t[i] = math.floor(math.abs(math.sin(i)) * 2^32)
    end
    return t
end

-- Rotate left
local function rotl(x, n)
    x = x & 0xFFFFFFFF
    return ((x << n) | (x >> (32 - n))) & 0xFFFFFFFF
end

-- MD5 computation (จำลองอย่างง่าย - ไม่ใช่ implementation จริง)
-- ใน production ให้ใช้ library เช่น:
-- local md5 = require("md5")  -- รองรับโดย luarocks

-- Simple non-cryptographic hash (สำหรับ demo เท่านั้น)
function md5.simple_hash(s)
    -- FNV-1a hash (ไม่ใช่ MD5 จริง แต่ใช้แสดงแนวคิด)
    local hash = 2166136261
    for i = 1, #s do
        hash = hash ~ string.byte(s, i)
        hash = (hash * 16777619) & 0xFFFFFFFF
    end
    return string.format("%08x", hash)
end

-- จำลอง MD5 API
function md5.sum(data)
    -- ใน production: require("md5").sum(data)
    -- ใช้ simple hash เพื่อ demo
    local h1 = md5.simple_hash(data)
    local h2 = md5.simple_hash(data:reverse())
    local h3 = md5.simple_hash(tostring(#data))
    local h4 = md5.simple_hash(h1 .. h2)
    return h1 .. h2 .. h3 .. h4  -- 32 hex chars
end

function md5.tohex(data)
    return md5.sum(data)
end

print("=== Hash Functions ===")
print("\nMD5 (⚠️  Insecure - for checksum only):")
local test_strings = {
    "Hello, World!",
    "สวัสดีชาวโลก",
    "The quick brown fox jumps over the lazy dog",
    "",
}

for _, s in ipairs(test_strings) do
    local hash = md5.sum(s)
    local display = #s > 30 and s:sub(1, 30) .. "..." or s
    print(string.format("  %-35s -> %s", '"' .. display .. '"', hash))
end

-- MD5 collision vulnerability demo
print("\nMD5 ปัญหา collision:")
print("  ผู้โจมตีสามารถสร้างข้อมูลสองชุดที่มี MD5 เหมือนกันได้")
print("  ใช้เพื่อ: file checksum, non-security deduplication")
print("  ห้ามใช้เพื่อ: password hashing, digital signatures, security")
```

### SHA-256 และ SHA-512

```lua
-- SHA (Secure Hash Algorithm)
-- SHA-256: output 256 bits (32 bytes), 64 hex chars
-- SHA-512: output 512 bits (64 bytes), 128 hex chars

-- SHA-256 constants
local SHA256_K = {
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

-- จำลอง SHA-256 API (สำหรับ luaossl หรือ lua-openssl)
-- ใน production ใช้:
-- local sha = require("openssl.digest")
-- local hash = sha.new("sha256"):final(data)

local sha256_mock = {}

-- สร้าง SHA-256 like hash สำหรับ demo
function sha256_mock.hash(data)
    -- นี่ไม่ใช่ SHA-256 จริง - ใช้เพื่อ demo API เท่านั้น
    -- ใน production ต้องใช้ library จริง
    local h = {
        0x6a09e667, 0xbb67ae85, 0x3c6ef372, 0xa54ff53a,
        0x510e527f, 0x9b05688c, 0x1f83d9ab, 0x5be0cd19,
    }
    
    -- จำลอง mixing
    for i = 1, #data do
        local b = string.byte(data, i)
        for j = 1, 8 do
            h[j] = (h[j] ~ b ~ SHA256_K[((i + j) % 64) + 1]) & 0xFFFFFFFF
            h[j] = rotl(h[j], (i + j) % 32)
        end
    end
    
    -- สร้าง hex string
    local parts = {}
    for _, v in ipairs(h) do
        table.insert(parts, string.format("%08x", v))
    end
    return table.concat(parts)
end

function sha256_mock.file_hash(filepath)
    -- จำลองการ hash file
    local f = io.open(filepath, "rb")
    if not f then return nil, "Cannot open file" end
    
    local content = f:read("*all")
    f:close()
    
    return sha256_mock.hash(content)
end

print("SHA-256 (จำลอง - ใช้ library จริงใน production):")
local test_data = {
    "Hello, World!",
    "Password123!",
    "สวัสดีครับ",
}

for _, data in ipairs(test_data) do
    local hash = sha256_mock.hash(data)
    print(string.format("  SHA256(%q) =", data))
    print(string.format("    %s", hash))
end

-- SHA-512 mock (สำหรับ demo API)
local sha512_mock = {}
function sha512_mock.hash(data)
    -- สร้าง 128 hex chars สำหรับ demo
    local h1 = sha256_mock.hash(data)
    local h2 = sha256_mock.hash(data:reverse() .. data)
    return h1 .. h2
end

print("\nSHA-512 hash length:")
local sha512_hash = sha512_mock.hash("test")
print(string.format("  Length: %d chars = %d bits", #sha512_hash, #sha512_hash * 4))

-- Password hashing best practices
print("\n=== Password Hashing Guidelines ===")
print("❌ ห้ามใช้: MD5, SHA1, SHA256 สำหรับ passwords")
print("✓ ให้ใช้: bcrypt, scrypt, Argon2, PBKDF2")
print("  เหตุผล: ต้องการ 'slow' hash function เพื่อต้านทาน brute force")
```

## HMAC (Hash-based Message Authentication Code)

```lua
-- HMAC ใช้สำหรับ verify integrity และ authenticity

-- HMAC algorithm:
-- HMAC(K, m) = H((K' XOR opad) || H((K' XOR ipad) || m))
-- K' = key padded to block size
-- opad = 0x5c repeated, ipad = 0x36 repeated

local function xor_bytes(bytes, pad_byte)
    local result = {}
    for i = 1, #bytes do
        result[i] = bytes[i] ~ pad_byte
    end
    return result
end

local function bytes_to_string(bytes)
    local chars = {}
    for _, b in ipairs(bytes) do
        table.insert(chars, string.char(b))
    end
    return table.concat(chars)
end

local function string_to_bytes(s)
    local bytes = {}
    for i = 1, #s do
        table.insert(bytes, string.byte(s, i))
    end
    return bytes
end

-- HMAC implementation
local function hmac(hash_fn, block_size, key, message)
    local key_bytes = string_to_bytes(key)
    
    -- ถ้า key ยาวกว่า block size ให้ hash ก่อน
    if #key_bytes > block_size then
        local hashed = hash_fn(key)
        key_bytes = {}
        for i = 1, #hashed, 2 do
            table.insert(key_bytes, tonumber(hashed:sub(i, i+1), 16))
        end
    end
    
    -- Pad key to block size
    while #key_bytes < block_size do
        table.insert(key_bytes, 0)
    end
    
    -- Create ipad and opad keys
    local ipad_key = xor_bytes(key_bytes, 0x36)
    local opad_key = xor_bytes(key_bytes, 0x5c)
    
    -- Inner hash: H(ipad_key || message)
    local inner = bytes_to_string(ipad_key) .. message
    local inner_hash = hash_fn(inner)
    
    -- Outer hash: H(opad_key || inner_hash)
    local outer = bytes_to_string(opad_key) .. inner_hash
    local outer_hash = hash_fn(outer)
    
    return outer_hash
end

-- HMAC-SHA256
local function hmac_sha256(key, message)
    return hmac(sha256_mock.hash, 64, key, message)
end

print("=== HMAC ===")
print("HMAC-SHA256 examples:")

local secret_key = "my-secret-key-2024"
local messages = {
    "Hello, World!",
    "amount=100&currency=THB&merchant=shopA",
    '{"user": "alice", "action": "transfer"}',
}

for _, msg in ipairs(messages) do
    local mac = hmac_sha256(secret_key, msg)
    print(string.format("  Message: %q", msg:sub(1, 40)))
    print(string.format("  HMAC:    %s...", mac:sub(1, 32)))
    print()
end

-- ตรวจสอบ message integrity
local function verify_hmac(key, message, provided_mac)
    local computed_mac = hmac_sha256(key, message)
    -- Constant-time comparison (ป้องกัน timing attack)
    if #computed_mac ~= #provided_mac then return false end
    local diff = 0
    for i = 1, #computed_mac do
        diff = diff | (string.byte(computed_mac, i) ~ string.byte(provided_mac, i))
    end
    return diff == 0
end

local msg = "payment:amount=500:to=alice"
local mac = hmac_sha256(secret_key, msg)

print("HMAC Verification:")
print("  Valid MAC: " .. tostring(verify_hmac(secret_key, msg, mac)))
print("  Tampered message: " .. tostring(
    verify_hmac(secret_key, msg .. "tampered", mac)))
print("  Wrong key: " .. tostring(
    verify_hmac("wrong-key", msg, mac)))
```

## Symmetric Encryption: AES

```lua
-- AES (Advanced Encryption Standard)
-- Block cipher, key sizes: 128, 192, 256 bits
-- Modes: ECB, CBC, CFB, OFB, GCM

-- ใน production ใช้ luaossl:
-- local aes = require("openssl.cipher")
-- local cipher = aes.new("aes-256-gcm")

-- จำลอง AES operations สำหรับ demo
local AES = {}

-- Key generation
function AES.generate_key(bits)
    bits = bits or 256
    local bytes = bits // 8
    local key = {}
    for i = 1, bytes do
        key[i] = math.random(0, 255)
    end
    local key_str = bytes_to_string(key)
    -- แปลงเป็น hex
    local hex_parts = {}
    for i = 1, #key_str do
        table.insert(hex_parts, string.format("%02x", string.byte(key_str, i)))
    end
    return key_str, table.concat(hex_parts)
end

-- IV (Initialization Vector) generation
function AES.generate_iv()
    local iv = {}
    for i = 1, 16 do
        iv[i] = math.random(0, 255)
    end
    local iv_str = bytes_to_string(iv)
    local hex_parts = {}
    for i = 1, #iv_str do
        table.insert(hex_parts, string.format("%02x", string.byte(iv_str, i)))
    end
    return iv_str, table.concat(hex_parts)
end

-- Padding (PKCS7)
function AES.pkcs7_pad(data, block_size)
    block_size = block_size or 16
    local pad_len = block_size - (#data % block_size)
    return data .. string.rep(string.char(pad_len), pad_len)
end

function AES.pkcs7_unpad(data)
    if #data == 0 then return data end
    local pad_len = string.byte(data, #data)
    if pad_len > 16 or pad_len == 0 then
        error("Invalid padding")
    end
    -- ตรวจสอบ padding
    for i = #data - pad_len + 1, #data do
        if string.byte(data, i) ~= pad_len then
            error("Invalid padding")
        end
    end
    return data:sub(1, #data - pad_len)
end

-- Encrypt (จำลอง - ไม่ใช่ AES จริง)
function AES.encrypt_cbc(plaintext, key, iv)
    -- CBC mode: ciphertext[i] = encrypt(plaintext[i] XOR ciphertext[i-1])
    -- นี่เป็นการจำลอง API เท่านั้น
    
    local padded = AES.pkcs7_pad(plaintext)
    
    -- จำลอง encryption
    local key_bytes = string_to_bytes(key:sub(1, 32))
    local iv_bytes = string_to_bytes(iv:sub(1, 16))
    
    local ciphertext = {}
    local prev_block = iv_bytes
    
    for i = 1, #padded, 16 do
        local block = {}
        for j = 1, 16 do
            local pt_byte = string.byte(padded, i + j - 1) or 0
            local key_byte = key_bytes[((i // 16 + j - 1) % #key_bytes) + 1] or 0
            block[j] = pt_byte ~ prev_block[j] ~ key_byte
        end
        for _, b in ipairs(block) do
            table.insert(ciphertext, b)
        end
        prev_block = block
    end
    
    return bytes_to_string(ciphertext)
end

function AES.decrypt_cbc(ciphertext, key, iv)
    -- จำลอง decryption
    local key_bytes = string_to_bytes(key:sub(1, 32))
    local iv_bytes = string_to_bytes(iv:sub(1, 16))
    
    local plaintext = {}
    local prev_block = iv_bytes
    
    for i = 1, #ciphertext, 16 do
        local block = {}
        for j = 1, 16 do
            local ct_byte = string.byte(ciphertext, i + j - 1) or 0
            block[j] = ct_byte
        end
        
        for j = 1, 16 do
            local key_byte = key_bytes[((i // 16 + j - 1) % #key_bytes) + 1] or 0
            table.insert(plaintext, block[j] ~ prev_block[j] ~ key_byte)
        end
        
        prev_block = block
    end
    
    local result = bytes_to_string(plaintext)
    return AES.pkcs7_unpad(result)
end

-- AES-GCM (Authenticated Encryption)
function AES.encrypt_gcm(plaintext, key, aad)
    -- GCM = Counter Mode + GMAC authentication
    -- ให้ทั้ง confidentiality และ integrity
    
    local iv, iv_hex = AES.generate_iv()
    local ciphertext = AES.encrypt_cbc(plaintext, key, iv)
    
    -- Authentication tag (จำลอง)
    local tag_data = ciphertext .. (aad or "")
    local auth_tag = hmac_sha256(key, tag_data):sub(1, 32)
    
    return {
        ciphertext = ciphertext,
        iv = iv,
        iv_hex = iv_hex,
        auth_tag = auth_tag,
        aad = aad,
    }
end

function AES.decrypt_gcm(encrypted, key)
    local {ciphertext, iv, auth_tag, aad} = encrypted
    
    -- ตรวจสอบ authentication tag ก่อน decrypt
    local tag_data = ciphertext .. (aad or "")
    local computed_tag = hmac_sha256(key, tag_data):sub(1, 32)
    
    if computed_tag ~= auth_tag then
        error("Authentication failed: data may have been tampered")
    end
    
    return AES.decrypt_cbc(ciphertext, key, iv)
end

-- ทดสอบ AES
print("=== AES Encryption (จำลอง) ===")
math.randomseed(42)

local key, key_hex = AES.generate_key(256)
print(string.format("Generated AES-256 key (%d bytes): %s...", #key, key_hex:sub(1, 32)))

local iv, iv_hex = AES.generate_iv()
print(string.format("Generated IV (%d bytes): %s", #iv, iv_hex))

local message = "ข้อมูลลับ: เลขบัตรเครดิต 4111-1111-1111-1111"
print(string.format("\nPlaintext: %q", message))

-- CBC encryption
local ciphertext = AES.encrypt_cbc(message, key, iv)
local ct_hex = {}
for i = 1, math.min(16, #ciphertext) do
    table.insert(ct_hex, string.format("%02x", string.byte(ciphertext, i)))
end
print(string.format("Ciphertext (%d bytes): %s...", #ciphertext, table.concat(ct_hex)))

-- Decrypt
local decrypted = AES.decrypt_cbc(ciphertext, key, iv)
print(string.format("Decrypted: %q", decrypted))
print("Match: " .. tostring(message == decrypted))

-- GCM mode
print("\nAES-GCM (Authenticated Encryption):")
local gcm_result = AES.encrypt_gcm(message, key, "user-id:12345")
print(string.format("  Auth tag: %s", gcm_result.auth_tag:sub(1, 16) .. "..."))
print(string.format("  AAD: %s", gcm_result.aad))

local function decrypt_gcm_safe(encrypted, key)
    local ok, result = pcall(function()
        local ct, iv_val, auth_tag, aad = 
            encrypted.ciphertext, encrypted.iv, 
            encrypted.auth_tag, encrypted.aad
        
        local tag_data = ct .. (aad or "")
        local computed_tag = hmac_sha256(key, tag_data):sub(1, 32)
        
        if computed_tag ~= auth_tag then
            error("Authentication failed")
        end
        
        return AES.decrypt_cbc(ct, key, iv_val)
    end)
    return ok, result
end

local ok, result = decrypt_gcm_safe(gcm_result, key)
print("  Decrypt GCM: " .. (ok and "✓ " .. result:sub(1, 20) .. "..." or "✗ " .. tostring(result)))
```

## Asymmetric Encryption: RSA

```lua
-- RSA (Rivest-Shamir-Adleman)
-- Public key encryption

-- RSA concepts (ไม่ implement จริง - ต้องใช้ library)
local RSA = {}

-- RSA Key Generation concepts
--[[
1. เลือก prime numbers p และ q
2. คำนวณ n = p * q  (modulus)
3. คำนวณ φ(n) = (p-1)(q-1)  (Euler's totient)
4. เลือก e: 1 < e < φ(n), gcd(e, φ(n)) = 1  (public exponent)
5. คำนวณ d: d * e ≡ 1 (mod φ(n))  (private exponent)
6. Public key = (n, e)
7. Private key = (n, d)

Encrypt: c = m^e mod n
Decrypt: m = c^d mod n
]]

-- Small RSA example (ใช้ตัวเลขเล็กสำหรับ demo)
local function gcd(a, b)
    while b ~= 0 do
        a, b = b, a % b
    end
    return a
end

local function mod_pow(base, exp, modulus)
    -- Fast modular exponentiation
    local result = 1
    base = base % modulus
    while exp > 0 do
        if exp % 2 == 1 then
            result = (result * base) % modulus
        end
        exp = exp >> 1
        base = (base * base) % modulus
    end
    return result
end

local function is_prime(n)
    if n < 2 then return false end
    if n == 2 then return true end
    if n % 2 == 0 then return false end
    for i = 3, math.floor(math.sqrt(n)), 2 do
        if n % i == 0 then return false end
    end
    return true
end

-- Extended Euclidean Algorithm
local function extended_gcd(a, b)
    if b == 0 then
        return a, 1, 0
    end
    local g, x, y = extended_gcd(b, a % b)
    return g, y, x - math.floor(a / b) * y
end

local function mod_inverse(e, phi)
    local g, x = extended_gcd(e % phi, phi)
    if g ~= 1 then return nil end
    return (x % phi + phi) % phi
end

-- Simple RSA demo with small numbers
function RSA.demo_keygen()
    -- ตัวเลขเล็กสำหรับ demo
    local p = 61
    local q = 53
    
    local n = p * q
    local phi = (p - 1) * (q - 1)
    
    -- เลือก e
    local e = 17
    while gcd(e, phi) ~= 1 do
        e = e + 2
    end
    
    -- คำนวณ d
    local d = mod_inverse(e, phi)
    
    return {
        public_key = {n = n, e = e},
        private_key = {n = n, d = d},
        p = p, q = q, phi = phi,
    }
end

function RSA.encrypt(message_num, public_key)
    return mod_pow(message_num, public_key.e, public_key.n)
end

function RSA.decrypt(ciphertext, private_key)
    return mod_pow(ciphertext, private_key.d, private_key.n)
end

print("=== RSA Demo (Small numbers for illustration) ===")
local keys = RSA.demo_keygen()
print(string.format("p = %d, q = %d", keys.p, keys.q))
print(string.format("n = %d (modulus)", keys.public_key.n))
print(string.format("φ(n) = %d (Euler's totient)", keys.phi))
print(string.format("e = %d (public exponent)", keys.public_key.e))
print(string.format("d = %d (private exponent)", keys.private_key.d))
print(string.format("\nPublic key: (n=%d, e=%d)", keys.public_key.n, keys.public_key.e))
print(string.format("Private key: (n=%d, d=%d)", keys.private_key.n, keys.private_key.d))

-- ทดสอบ encrypt/decrypt
local messages = {42, 88, 123}
print("\nEncrypt/Decrypt Test:")
for _, m in ipairs(messages) do
    local ct = RSA.encrypt(m, keys.public_key)
    local pt = RSA.decrypt(ct, keys.private_key)
    print(string.format("  message=%d -> encrypted=%d -> decrypted=%d %s",
        m, ct, pt, m == pt and "✓" or "✗"))
end

-- RSA real-world API (ด้วย luaossl)
print("\nReal-world RSA with luaossl:")
print([[
  -- การใช้งาน luaossl:
  local pkey = require("openssl.pkey")
  
  -- สร้าง RSA key pair
  local private_key = pkey.new({type="RSA", bits=2048})
  local public_key = private_key:getPublicKey()
  
  -- Export keys
  local priv_pem = private_key:toPEM("private")
  local pub_pem = public_key:toPEM("public")
  
  -- Encrypt (ด้วย public key)
  local ciphertext = public_key:encrypt(message)
  
  -- Decrypt (ด้วย private key)
  local plaintext = private_key:decrypt(ciphertext)
]])
```

## Digital Signatures

```lua
-- Digital Signatures ใช้สำหรับ non-repudiation

-- Digital Signature scheme:
-- Sign:    signature = sign(private_key, hash(message))
-- Verify:  verify(public_key, hash(message), signature)

local DigitalSignature = {}

-- จำลอง ECDSA-like signature (ไม่ใช่ implementation จริง)
function DigitalSignature.sign(message, private_key)
    -- ใน production ใช้ luaossl:
    -- local signer = require("openssl.digest")
    -- local sig = private_key:sign(message, "sha256")
    
    local message_hash = sha256_mock.hash(message)
    
    -- จำลอง signature
    local sig_data = hmac_sha256(private_key, message_hash)
    
    return {
        signature = sig_data,
        algorithm = "HMAC-SHA256",
        message_hash = message_hash,
    }
end

function DigitalSignature.verify(message, signature_obj, public_key)
    local message_hash = sha256_mock.hash(message)
    
    -- ตรวจสอบว่า hash ตรงกัน
    if message_hash ~= signature_obj.message_hash then
        return false, "Message hash mismatch"
    end
    
    -- ตรวจสอบ signature
    local expected = hmac_sha256(public_key, message_hash)
    
    return expected == signature_obj.signature, 
        expected == signature_obj.signature and "Valid" or "Invalid signature"
end

-- Certificate-based signature workflow
local function demo_signing_workflow()
    print("=== Digital Signature Workflow ===")
    
    -- สร้าง key pair (จำลอง)
    local private_key = "private-key-secret-value-abc123"
    local public_key = "public-key-derived-from-private"
    
    -- Document ที่ต้องการ sign
    local document = [[
สัญญาซื้อขาย
วันที่: 2024-03-15
ผู้ขาย: บริษัท ABC จำกัด
ผู้ซื้อ: นาย ก. ข.
จำนวนเงิน: 100,000 บาท
]]
    
    print("Document:")
    print(document)
    
    -- Sign
    local sig = DigitalSignature.sign(document, private_key)
    print(string.format("Signature (%s): %s...", 
        sig.algorithm, sig.signature:sub(1, 32)))
    print(string.format("Message hash: %s...", sig.message_hash:sub(1, 32)))
    
    -- Verify (valid)
    local valid, reason = DigitalSignature.verify(document, sig, public_key)
    print("\nVerification (original): " .. (valid and "✓ " or "✗ ") .. reason)
    
    -- Verify (tampered document)
    local tampered = document:gsub("100,000", "1,000,000")
    valid, reason = DigitalSignature.verify(tampered, sig, public_key)
    print("Verification (tampered): " .. (valid and "✓ " or "✗ ") .. reason)
    
    return sig
end

demo_signing_workflow()

-- JWT (JSON Web Token) implementation
local JWT = {}

local function base64url_encode(data)
    -- Base64 URL encoding (no padding)
    local b64 = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
    local result = {}
    
    for i = 1, #data, 3 do
        local b1 = string.byte(data, i) or 0
        local b2 = string.byte(data, i+1) or 0
        local b3 = string.byte(data, i+2) or 0
        
        local n = (b1 << 16) | (b2 << 8) | b3
        
        for shift = 18, 0, -6 do
            local idx = ((n >> shift) & 63) + 1
            table.insert(result, b64:sub(idx, idx))
        end
    end
    
    local encoded = table.concat(result)
    -- Remove padding and replace + and /
    encoded = encoded:gsub("=", ""):gsub("+", "-"):gsub("/", "_")
    
    return encoded
end

local function json_encode_simple(t)
    local parts = {}
    for k, v in pairs(t) do
        local val
        if type(v) == "string" then
            val = '"' .. v .. '"'
        elseif type(v) == "number" then
            val = tostring(v)
        elseif type(v) == "boolean" then
            val = tostring(v)
        end
        if val then
            table.insert(parts, '"' .. k .. '":' .. val)
        end
    end
    return "{" .. table.concat(parts, ",") .. "}"
end

function JWT.create(payload, secret, algorithm)
    algorithm = algorithm or "HS256"
    
    local header = json_encode_simple({alg = algorithm, typ = "JWT"})
    local header_b64 = base64url_encode(header)
    
    -- เพิ่ม timestamps
    payload.iat = payload.iat or os.time()
    payload.exp = payload.exp or (os.time() + 3600)
    
    local payload_json = json_encode_simple(payload)
    local payload_b64 = base64url_encode(payload_json)
    
    local signing_input = header_b64 .. "." .. payload_b64
    
    -- Sign
    local signature = base64url_encode(hmac_sha256(secret, signing_input))
    
    return signing_input .. "." .. signature
end

function JWT.verify(token, secret)
    local parts = {}
    for part in token:gmatch("[^.]+") do
        table.insert(parts, part)
    end
    
    if #parts ~= 3 then
        return nil, "Invalid token format"
    end
    
    local header_b64, payload_b64, signature = parts[1], parts[2], parts[3]
    
    -- Verify signature
    local signing_input = header_b64 .. "." .. payload_b64
    local expected_sig = base64url_encode(hmac_sha256(secret, signing_input))
    
    if expected_sig ~= signature then
        return nil, "Invalid signature"
    end
    
    -- ใน production จะ decode base64 และ parse JSON
    return {
        verified = true,
        header_b64 = header_b64,
        payload_b64 = payload_b64,
    }, nil
end

print("\n=== JWT (JSON Web Token) ===")
local jwt_secret = "super-secret-jwt-key"
local jwt_payload = {
    sub = "user-12345",
    name = "สมชาย ใจดี",
    role = "admin",
}

local token = JWT.create(jwt_payload, jwt_secret)
print("JWT Token:")
print(token)
print(string.format("Token parts: %d", select(2, token:gsub("%.", ".")) + 1))

local claims, err = JWT.verify(token, jwt_secret)
if claims then
    print("\nToken verified: ✓")
    print("Payload header_b64: " .. claims.header_b64)
else
    print("\nVerification failed: " .. err)
end

-- Tampered token
local tampered_token = token:sub(1, -5) .. "XXXX"
local _, tam_err = JWT.verify(tampered_token, jwt_secret)
print("Tampered token: " .. (tam_err or "valid"))
```

## Random Number Generation (CSPRNG)

```lua
-- Cryptographically Secure Pseudo-Random Number Generator

local CSPRNG = {}

-- ใน production ใช้:
-- /dev/urandom บน Linux/macOS
-- CryptGenRandom บน Windows
-- หรือ library เช่น luaossl

-- จำลอง secure random bytes
function CSPRNG.random_bytes(n)
    -- ใน production ใช้ os.urandom หรือ openssl.rand
    -- นี่เป็น demo เท่านั้น - ไม่ปลอดภัย
    
    local bytes = {}
    for i = 1, n do
        bytes[i] = math.random(0, 255)
    end
    return bytes_to_string(bytes)
end

function CSPRNG.random_hex(n)
    local bytes = CSPRNG.random_bytes(n)
    local hex = {}
    for i = 1, #bytes do
        table.insert(hex, string.format("%02x", string.byte(bytes, i)))
    end
    return table.concat(hex)
end

function CSPRNG.random_int(min, max)
    local bytes = CSPRNG.random_bytes(4)
    local n = 0
    for i = 1, 4 do
        n = n * 256 + string.byte(bytes, i)
    end
    return min + (n % (max - min + 1))
end

function CSPRNG.uuid_v4()
    local hex = CSPRNG.random_hex(16)
    -- UUID v4 format: xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx
    local uuid = string.format(
        "%s-%s-4%s-%s%s-%s",
        hex:sub(1, 8),
        hex:sub(9, 12),
        hex:sub(14, 16),
        ({8, 9, "a", "b"})[CSPRNG.random_int(1, 4)],
        hex:sub(18, 20),
        hex:sub(21, 32)
    )
    return uuid
end

print("=== CSPRNG ===")
math.randomseed(os.time())  -- ใน production ใช้ entropy จาก OS

print("Random bytes (16):")
print("  hex: " .. CSPRNG.random_hex(16))

print("\nRandom integers:")
for i = 1, 5 do
    io.write(CSPRNG.random_int(1, 100) .. " ")
end
print()

print("\nUUID v4 examples:")
for i = 1, 3 do
    print("  " .. CSPRNG.uuid_v4())
end

-- Secure token generation
function CSPRNG.generate_token(length, charset)
    charset = charset or "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789"
    local token = {}
    for i = 1, length do
        local idx = CSPRNG.random_int(1, #charset)
        table.insert(token, charset:sub(idx, idx))
    end
    return table.concat(token)
end

print("\nSecure tokens:")
print("  Session token (32): " .. CSPRNG.generate_token(32))
print("  API key (48):       " .. CSPRNG.generate_token(48))
print("  OTP (6 digits):     " .. CSPRNG.generate_token(6, "0123456789"))
print("  Recovery code:      " .. CSPRNG.generate_token(4, "ABCDEFGHJKLMNPQRSTUVWXYZ23456789") ..
    "-" .. CSPRNG.generate_token(4, "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"))
```

## Key Derivation Functions

```lua
-- Key Derivation Functions (KDF)
-- แปลง password เป็น cryptographic key

local KDF = {}

-- PBKDF2 (Password-Based Key Derivation Function 2)
function KDF.pbkdf2(password, salt, iterations, key_length, hash_fn)
    hash_fn = hash_fn or hmac_sha256
    key_length = key_length or 32
    iterations = iterations or 100000
    
    local PRF = function(data)
        return hash_fn(password, data)
    end
    
    local key = ""
    local block_num = 0
    
    while #key < key_length do
        block_num = block_num + 1
        
        -- U1 = PRF(salt || INT(i))
        local block_salt = salt .. string.char(
            (block_num >> 24) & 0xFF,
            (block_num >> 16) & 0xFF,
            (block_num >> 8) & 0xFF,
            block_num & 0xFF
        )
        
        local u = PRF(block_salt)
        local f = u
        
        -- Ui = PRF(U(i-1)), F = XOR of all Ui
        for i = 2, iterations do
            u = PRF(u)
            -- XOR f and u
            local xored = {}
            for j = 1, #f do
                local fb = string.byte(f, j)
                local ub = string.byte(u, j) or 0
                table.insert(xored, string.char(fb ~ ub))
            end
            f = table.concat(xored)
            
            -- Break early สำหรับ demo (production ควร iterate ทั้งหมด)
            if i == 3 then break end
        end
        
        key = key .. f
    end
    
    return key:sub(1, key_length)
end

-- bcrypt concepts (ไม่ implement จริง)
print("=== Key Derivation Functions ===")
print("\nPBKDF2:")

local password = "MySecureP@ssw0rd!"
local salt = CSPRNG.random_bytes(16)
local salt_hex = CSPRNG.random_hex(16)
print(string.format("  Password: %q", password))
print(string.format("  Salt: %s", salt_hex))

-- จำลอง PBKDF2 (iterations ลดลงสำหรับ demo)
local start = os.clock()
local derived_key = KDF.pbkdf2(password, salt, 10, 32)  -- เพียง 10 iterations สำหรับ demo
local elapsed = (os.clock() - start) * 1000

-- แปลง derived key เป็น hex
local dk_hex = {}
for i = 1, #derived_key do
    table.insert(dk_hex, string.format("%02x", string.byte(derived_key, i)))
end
print(string.format("  Derived key: %s...", table.concat(dk_hex):sub(1, 32)))
print(string.format("  Time (10 iter): %.2fms", elapsed))
print("  Note: Production ใช้ 100,000+ iterations สำหรับ bcrypt/PBKDF2")

-- Argon2 concepts
print("\nArgon2 (concepts):")
print([[
  Argon2 เป็น KDF ที่ชนะ Password Hashing Competition (2015)
  
  Variants:
  - Argon2d: resistant to GPU cracking
  - Argon2i: resistant to side-channel attacks  
  - Argon2id: combination (แนะนำ)
  
  Parameters:
  - Memory: จำนวน RAM ที่ใช้ (ยิ่งมาก = ยิ่งปลอดภัย)
  - Iterations: จำนวนรอบ
  - Parallelism: จำนวน threads
  
  ตัวอย่าง (ด้วย lua-argon2):
  local argon2 = require("argon2")
  local hash = argon2.hash_encoded(password, salt, {
      variant = argon2.variants.argon2id,
      t_cost = 3,      -- iterations
      m_cost = 65536,  -- 64MB memory
      parallelism = 4, -- 4 threads
  })
]])

-- Password hashing workflow
local function hash_password(password)
    local salt = CSPRNG.random_hex(16)
    
    -- ใน production ใช้ bcrypt หรือ argon2
    -- นี่เป็น demo ของ workflow
    local derived = KDF.pbkdf2(password, salt, 10, 32)
    local dk_hex = {}
    for i = 1, #derived do
        table.insert(dk_hex, string.format("%02x", string.byte(derived, i)))
    end
    
    -- Format: algorithm$salt$hash
    return string.format("pbkdf2-sha256$%s$%s", salt, table.concat(dk_hex))
end

local function verify_password(password, stored_hash)
    local algorithm, salt, stored_dk = stored_hash:match("^(.-)%$(.-)%$(.+)$")
    
    if not algorithm then return false end
    
    -- คำนวณ hash ใหม่
    local derived = KDF.pbkdf2(password, salt, 10, 32)
    local dk_hex = {}
    for i = 1, #derived do
        table.insert(dk_hex, string.format("%02x", string.byte(derived, i)))
    end
    local computed = table.concat(dk_hex)
    
    -- Constant-time comparison
    if #computed ~= #stored_dk then return false end
    local diff = 0
    for i = 1, #computed do
        diff = diff | (string.byte(computed, i) ~ string.byte(stored_dk, i))
    end
    return diff == 0
end

print("=== Password Hashing Workflow ===")
local pwd = "user_password_123!"
local stored = hash_password(pwd)
print("Stored hash: " .. stored:sub(1, 60) .. "...")

print("\nVerification:")
print("  Correct password: " .. tostring(verify_password(pwd, stored)))
print("  Wrong password:   " .. tostring(verify_password("wrongpassword", stored)))
```

## Certificate Handling

```lua
-- X.509 Certificate concepts

local Certificate = {}

-- Certificate structure (simplified)
function Certificate.parse_pem_info(pem_like_data)
    -- จำลอง certificate parsing
    return {
        version = 3,
        serial_number = CSPRNG.random_hex(16),
        subject = {
            CN = "example.com",
            O = "Example Corp",
            C = "TH",
            L = "Bangkok",
        },
        issuer = {
            CN = "Example CA",
            O = "Example CA Corp",
            C = "TH",
        },
        validity = {
            not_before = "2024-01-01T00:00:00Z",
            not_after = "2025-01-01T00:00:00Z",
        },
        public_key = {
            algorithm = "RSA",
            bits = 2048,
        },
        extensions = {
            subject_alt_names = {"example.com", "www.example.com", "api.example.com"},
            key_usage = {"digitalSignature", "keyEncipherment"},
            extended_key_usage = {"serverAuth", "clientAuth"},
            basic_constraints = {is_ca = false},
        },
        signature_algorithm = "sha256WithRSAEncryption",
    }
end

function Certificate.verify_chain(cert, issuer_cert)
    -- จำลอง certificate chain verification
    return {
        valid = true,
        chain_valid = true,
        not_expired = true,
        hostname_valid = true,
        revoked = false,
    }
end

function Certificate.is_expired(cert)
    -- ตรวจสอบว่า cert หมดอายุหรือยัง
    local now = os.date("%Y-%m-%dT%H:%M:%SZ")
    return now > cert.validity.not_after
end

print("=== X.509 Certificate ===")
local cert = Certificate.parse_pem_info("")
print("Subject: CN=" .. cert.subject.CN .. ", O=" .. cert.subject.O)
print("Issuer: CN=" .. cert.issuer.CN)
print("Valid from: " .. cert.validity.not_before)
print("Valid until: " .. cert.validity.not_after)
print("Public key: " .. cert.public_key.algorithm .. " " .. cert.public_key.bits .. "-bit")
print("SANs: " .. table.concat(cert.extensions.subject_alt_names, ", "))
print("Key Usage: " .. table.concat(cert.extensions.key_usage, ", "))
print("Expired: " .. tostring(Certificate.is_expired(cert)))

local chain_result = Certificate.verify_chain(cert, {})
print("\nChain verification:")
for k, v in pairs(chain_result) do
    print(string.format("  %-20s: %s", k, tostring(v)))
end

-- Certificate Pinning
local function check_cert_pin(cert_hash, pinned_hashes)
    for _, pinned in ipairs(pinned_hashes) do
        if cert_hash == pinned then
            return true
        end
    end
    return false
end

local pinned_hashes = {
    "abc123...",  -- production cert
    "def456...",  -- backup cert
}
local current_cert_hash = "abc123..."
print("\nCertificate pinning: " .. 
    (check_cert_pin(current_cert_hash, pinned_hashes) and "✓ Pinned cert" or "✗ Unknown cert"))
```

## TLS/SSL ใน Lua

```lua
-- TLS/SSL configuration และ best practices

local TLS = {}

-- TLS Configuration
function TLS.create_server_config(options)
    options = options or {}
    return {
        -- Protocol versions
        min_protocol = options.min_protocol or "TLSv1.2",
        max_protocol = options.max_protocol or "TLSv1.3",
        
        -- Cipher suites (secure ones)
        cipher_suites = options.cipher_suites or {
            -- TLS 1.3
            "TLS_AES_256_GCM_SHA384",
            "TLS_CHACHA20_POLY1305_SHA256",
            "TLS_AES_128_GCM_SHA256",
            -- TLS 1.2
            "ECDHE-ECDSA-AES256-GCM-SHA384",
            "ECDHE-RSA-AES256-GCM-SHA384",
            "ECDHE-ECDSA-CHACHA20-POLY1305",
            "ECDHE-RSA-CHACHA20-POLY1305",
        },
        
        -- Certificate files
        cert_file = options.cert_file or "/etc/ssl/certs/server.crt",
        key_file = options.key_file or "/etc/ssl/private/server.key",
        ca_file = options.ca_file,
        
        -- Settings
        verify_client = options.verify_client or false,
        session_tickets = options.session_tickets ~= false,
        ocsp_stapling = options.ocsp_stapling ~= false,
        
        -- HSTS
        hsts = options.hsts or {
            max_age = 31536000,  -- 1 year
            include_subdomains = true,
            preload = true,
        },
    }
end

function TLS.create_client_config(options)
    options = options or {}
    return {
        min_protocol = options.min_protocol or "TLSv1.2",
        verify_peer = options.verify_peer ~= false,  -- ต้อง verify เสมอ
        verify_hostname = options.verify_hostname ~= false,
        ca_bundle = options.ca_bundle or "/etc/ssl/certs/ca-certificates.crt",
        cert_pinning = options.cert_pinning or {},
        timeout = options.timeout or 30,
    }
end

print("=== TLS Configuration ===")

local server_tls = TLS.create_server_config({
    cert_file = "/etc/ssl/certs/myapp.crt",
    key_file = "/etc/ssl/private/myapp.key",
    verify_client = false,
})

print("Server TLS Config:")
print("  Protocol: " .. server_tls.min_protocol .. " - " .. server_tls.max_protocol)
print("  Ciphers: " .. #server_tls.cipher_suites .. " secure suites")
print("  OCSP stapling: " .. tostring(server_tls.ocsp_stapling))
print("  HSTS max-age: " .. server_tls.hsts.max_age .. "s")

local client_tls = TLS.create_client_config({
    cert_pinning = {"sha256/AAAB...=", "sha256/BBBB...="},
})
print("\nClient TLS Config:")
print("  Verify peer: " .. tostring(client_tls.verify_peer))
print("  Verify hostname: " .. tostring(client_tls.verify_hostname))
print("  Cert pins: " .. #client_tls.cert_pinning)

-- TLS Handshake Steps
print("\nTLS 1.3 Handshake Steps:")
local tls_steps = {
    "1. Client Hello (with supported ciphers, key_share)",
    "2. Server Hello (selected cipher, key_share)",
    "3. Server Certificate",
    "4. Server Certificate Verify",
    "5. Server Finished",
    "6. Client Finished",
    "7. Application Data (encrypted)",
}
for _, step in ipairs(tls_steps) do
    print("  " .. step)
end

-- HTTPS client using TLS
print("\nHTTPS Client (with luaossl):")
print([[
  local http = require("http.client")
  local ssl = require("openssl.ssl")
  
  local ctx = ssl.ctx_new("TLS", ssl.TLSv1_2_method())
  ctx:setVerify(ssl.VERIFY_PEER)
  ctx:setCAFile("/etc/ssl/certs/ca-certificates.crt")
  
  local client = http.new({
      tls_context = ctx,
      timeout = 30,
  })
  
  local response = client:request("GET", "https://api.example.com/data")
]])
```

## Secure Token Generation

```lua
-- Secure token สำหรับ authentication, CSRF, sessions

local TokenGen = {}

-- Session token
function TokenGen.session_token()
    return CSPRNG.random_hex(32)  -- 256 bits
end

-- CSRF token
function TokenGen.csrf_token()
    return CSPRNG.random_hex(16)  -- 128 bits
end

-- API key
function TokenGen.api_key(prefix)
    prefix = prefix or "ak"
    local random_part = CSPRNG.random_hex(24)
    local checksum = sha256_mock.hash(random_part):sub(1, 4)
    return string.format("%s_%s_%s", prefix, random_part, checksum)
end

-- Verification token (email/phone)
function TokenGen.verification_token(type)
    if type == "numeric" then
        -- 6-digit OTP
        return string.format("%06d", CSPRNG.random_int(0, 999999))
    elseif type == "alphanumeric" then
        -- Short alphanumeric token
        return CSPRNG.generate_token(8, "ABCDEFGHJKLMNPQRSTUVWXYZ23456789")
    else
        return CSPRNG.random_hex(20)
    end
end

-- Reset token (เช่น password reset)
function TokenGen.reset_token()
    local token = CSPRNG.random_hex(32)
    local expires_at = os.time() + 3600  -- 1 hour
    local signature = hmac_sha256("token-signing-key", token .. ":" .. expires_at)
    
    return {
        token = token,
        expires_at = expires_at,
        signature = signature:sub(1, 32),
    }
end

function TokenGen.verify_reset_token(token_data, token_string)
    -- ตรวจสอบว่า token ยังไม่หมดอายุ
    if os.time() > token_data.expires_at then
        return false, "Token expired"
    end
    
    -- ตรวจสอบ signature
    local expected_sig = hmac_sha256(
        "token-signing-key", 
        token_string .. ":" .. token_data.expires_at
    ):sub(1, 32)
    
    return expected_sig == token_data.signature, 
        expected_sig == token_data.signature and "Valid" or "Invalid"
end

print("=== Secure Token Generation ===")

print("Session tokens:")
for i = 1, 3 do
    print("  " .. TokenGen.session_token())
end

print("\nCSRF tokens:")
for i = 1, 2 do
    print("  " .. TokenGen.csrf_token())
end

print("\nAPI keys:")
print("  " .. TokenGen.api_key("sk"))
print("  " .. TokenGen.api_key("pk"))

print("\nVerification tokens:")
print("  OTP (numeric):      " .. TokenGen.verification_token("numeric"))
print("  Token (alphanum):   " .. TokenGen.verification_token("alphanumeric"))
print("  Token (hex):        " .. TokenGen.verification_token("hex"))

print("\nPassword reset token:")
local reset = TokenGen.reset_token()
print(string.format("  Token: %s...", reset.token:sub(1, 16)))
print(string.format("  Expires: %s", os.date("%Y-%m-%d %H:%M:%S", reset.expires_at)))
print(string.format("  Sig: %s...", reset.signature:sub(1, 16)))

local valid, msg = TokenGen.verify_reset_token(reset, reset.token)
print("  Verification: " .. (valid and "✓ " or "✗ ") .. msg)
```

## Cryptographic Best Practices

```lua
-- Best practices สำหรับ Cryptography ใน production

print("=== Cryptographic Best Practices ===")

local best_practices = {
    {
        category = "Hashing",
        dos = {
            "ใช้ SHA-256 หรือ SHA-512 สำหรับ data integrity",
            "ใช้ bcrypt, Argon2id, PBKDF2 สำหรับ password hashing",
            "ใช้ SHA-512 สำหรับ HMAC keys",
        },
        donts = {
            "ห้ามใช้ MD5 หรือ SHA-1 สำหรับ security",
            "ห้ามใช้ SHA-256 โดยตรงสำหรับ password (ต้องใช้ slow hash)",
            "ห้ามใช้ unsalted hash",
        }
    },
    {
        category = "Encryption",
        dos = {
            "ใช้ AES-256-GCM สำหรับ symmetric encryption",
            "สร้าง random IV ทุกครั้ง",
            "ใช้ authenticated encryption (AEAD)",
        },
        donts = {
            "ห้ามใช้ ECB mode",
            "ห้าม reuse IV",
            "ห้ามใช้ DES หรือ 3DES",
        }
    },
    {
        category = "Key Management",
        dos = {
            "เก็บ keys ใน environment variables หรือ key vault",
            "Rotate keys เป็นประจำ",
            "ใช้ different keys สำหรับ different purposes",
        },
        donts = {
            "ห้าม hardcode keys ในโค้ด",
            "ห้าม log keys",
            "ห้ามส่ง private keys ผ่าน network",
        }
    },
    {
        category = "Random Numbers",
        dos = {
            "ใช้ CSPRNG สำหรับ security-critical randomness",
            "ใช้ /dev/urandom บน Linux/macOS",
            "ใช้ library เช่น luaossl สำหรับ crypto random",
        },
        donts = {
            "ห้ามใช้ math.random() สำหรับ cryptographic purposes",
            "ห้ามใช้ timestamp เป็น seed",
            "ห้ามใช้ predictable seeds",
        }
    },
}

for _, bp in ipairs(best_practices) do
    print("\n" .. bp.category .. ":")
    print("  ✓ Do:")
    for _, d in ipairs(bp.dos) do
        print("    - " .. d)
    end
    print("  ✗ Don't:")
    for _, d in ipairs(bp.donts) do
        print("    - " .. d)
    end
end

-- Security checklist
print("\n=== Security Implementation Checklist ===")
local checklist = {
    {item = "ใช้ TLS 1.2+ สำหรับ all network communication", critical = true},
    {item = "Verify certificate chains", critical = true},
    {item = "ใช้ strong cipher suites เท่านั้น", critical = true},
    {item = "Hash passwords ด้วย bcrypt/argon2", critical = true},
    {item = "ใช้ HMAC สำหรับ message authentication", critical = true},
    {item = "Generate random tokens ด้วย CSPRNG", critical = true},
    {item = "Implement rate limiting", critical = false},
    {item = "Log security events", critical = false},
    {item = "ใช้ Content Security Policy (CSP)", critical = false},
    {item = "Implement CSRF protection", critical = false},
}

for _, item in ipairs(checklist) do
    local marker = item.critical and "[CRITICAL]" or "[IMPORTANT]"
    print(string.format("  %s %s", marker, item.item))
end
```

## Practical: Secure Storage

```lua
-- Secure data storage implementation

local SecureStorage = {}
SecureStorage.__index = SecureStorage

function SecureStorage.new(master_key)
    local self = setmetatable({}, SecureStorage)
    -- Derive encryption key from master key
    local salt = "storage-encryption-v1"
    self.enc_key = KDF.pbkdf2(master_key, salt, 5, 32)
    self.hmac_key = KDF.pbkdf2(master_key, salt .. "-hmac", 5, 32)
    self.store = {}
    return self
end

function SecureStorage:set(key, value)
    -- Serialize value
    local data = type(value) == "table" and 
        json_encode_simple(value) or tostring(value)
    
    -- Generate IV
    local iv = CSPRNG.random_bytes(16)
    
    -- Encrypt
    local ciphertext = AES.encrypt_cbc(data, self.enc_key, iv)
    
    -- MAC
    local mac = hmac_sha256(self.hmac_key, ciphertext .. iv)
    
    -- Store
    self.store[key] = {
        ciphertext = ciphertext,
        iv = iv,
        mac = mac:sub(1, 32),
        created_at = os.time(),
    }
    
    return true
end

function SecureStorage:get(key)
    local entry = self.store[key]
    if not entry then return nil, "Key not found" end
    
    -- Verify MAC
    local expected_mac = hmac_sha256(
        self.hmac_key, entry.ciphertext .. entry.iv
    ):sub(1, 32)
    
    if expected_mac ~= entry.mac then
        return nil, "Data integrity check failed"
    end
    
    -- Decrypt
    local ok, plaintext = pcall(AES.decrypt_cbc, entry.ciphertext, self.enc_key, entry.iv)
    if not ok then
        return nil, "Decryption failed"
    end
    
    return plaintext, nil
end

function SecureStorage:delete(key)
    self.store[key] = nil
end

function SecureStorage:list_keys()
    local keys = {}
    for k in pairs(self.store) do
        table.insert(keys, k)
    end
    return keys
end

-- ทดสอบ SecureStorage
print("=== Secure Storage Demo ===")
math.randomseed(777)

local storage = SecureStorage.new("master-password-should-be-strong!")

-- เก็บข้อมูลลับ
storage:set("api_key", "sk-prod-abc123xyz789")
storage:set("db_password", "super$ecret#DB!pass")
storage:set("config", {host = "db.internal", port = 5432, ssl = true})

print("Stored keys: " .. table.concat(storage:list_keys(), ", "))

-- อ่านข้อมูล
local api_key, err = storage:get("api_key")
print("Retrieved api_key: " .. (api_key and api_key:sub(1, 10) .. "..." or "ERROR: " .. err))

local config, err2 = storage:get("config")
print("Retrieved config: " .. (config and config:sub(1, 30) .. "..." or "ERROR: " .. err2))

-- ลองอ่าน key ที่ไม่มี
local _, miss_err = storage:get("nonexistent")
print("Missing key: " .. (miss_err or "found"))
```

## สรุปบทที่ 75

```lua
-- สรุปสิ่งที่ได้เรียนรู้ในบทที่ 75

print("=" .. string.rep("=", 55))
print("บทที่ 75: Cryptography กับ Lua")
print("=" .. string.rep("=", 55))

local topics = {
    "Cryptography Basics - CIA triad, ประเภทต่างๆ",
    "Hash Functions - MD5 (weak), SHA-256, SHA-512",
    "HMAC - Message Authentication Code",
    "AES Encryption - symmetric, CBC, GCM modes",
    "RSA - asymmetric encryption concepts",
    "Digital Signatures - non-repudiation",
    "JWT - JSON Web Tokens",
    "CSPRNG - Cryptographically Secure Random",
    "Key Derivation - PBKDF2, bcrypt, Argon2",
    "X.509 Certificates - PKI",
    "TLS/SSL - transport security",
    "Secure Token Generation",
    "Cryptographic Best Practices",
    "Secure Storage",
}

for i, topic in ipairs(topics) do
    print(string.format("  %2d. %s", i, topic))
end

print("\nKey Libraries:")
print("  luaossl     - OpenSSL bindings (แนะนำ)")
print("  luacrypto   - classic crypto library")
print("  lua-openssl - OpenSSL wrapper")
print("  lua-mbedtls - mbed TLS bindings")

print("\nCritical Security Rules:")
print("  1. ห้าม implement crypto yourself (ใช้ proven libraries)")
print("  2. ใช้ CSPRNG เสมอสำหรับ security-critical randomness")
print("  3. ใช้ AES-256-GCM สำหรับ symmetric encryption")
print("  4. ใช้ bcrypt/Argon2 สำหรับ password hashing")
print("  5. ห้าม reuse IVs/nonces")
print("  6. Verify TLS certificates เสมอ")
print("  7. ใช้ constant-time comparison สำหรับ sensitive comparisons")
print("  8. เก็บ keys อย่างปลอดภัย ไม่ hardcode ในโค้ด")
```
