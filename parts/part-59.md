# บทที่ 59: SQLite กับ Lua

## บทนำ

SQLite เป็น relational database ที่ไม่ต้องมี server แยกต่างหาก ข้อมูลถูกเก็บในไฟล์เดียว เหมาะสำหรับ embedded applications, mobile apps, desktop software และ prototyping ใน Lua เราสามารถใช้ LuaSQLite3 library เพื่อเชื่อมต่อกับ SQLite ได้

> **บทก่อนหน้า**: [บทที่ 58: PostgreSQL Advanced](part-58.md)

---

## 59.1 LuaSQLite3 Library

### ตัวอย่างที่ 1: การติดตั้งและเริ่มต้น

```bash
# ติดตั้งด้วย LuaRocks
luarocks install lsqlite3

# หรือบน Ubuntu/Debian
sudo apt-get install lua-sqlite3

# ทดสอบ
lua -e "require('lsqlite3'); print('LuaSQLite3 OK')"
```

```lua
-- sqlite_intro.lua
local sqlite3 = require("lsqlite3")

-- ตรวจสอบ version
print("LuaSQLite3 version:", sqlite3.version())
print("SQLite version:", sqlite3.lversion())

-- constants ที่ใช้บ่อย
print("SQLITE_OK:", sqlite3.OK)       -- 0
print("SQLITE_ERROR:", sqlite3.ERROR) -- 1
print("SQLITE_ROW:", sqlite3.ROW)     -- 100
print("SQLITE_DONE:", sqlite3.DONE)   -- 101
```

---

## 59.2 เปิดและปิด Database

### ตัวอย่างที่ 2: เปิด file-based database

```lua
-- open_database.lua
local sqlite3 = require("lsqlite3")

-- เปิด database (สร้างใหม่ถ้าไม่มี)
local db = sqlite3.open("myapp.db")
if not db then
    error("Cannot open database")
end

print("Database opened:", db ~= nil)
print("Database filename:", db:db_filename("main"))

-- ทำงานกับ database...
-- db:exec(sql)
-- db:prepare(sql)

-- ปิด database
local rc = db:close()
if rc ~= sqlite3.OK then
    print("Error closing:", rc)
else
    print("Database closed successfully")
end

-- เปิดด้วย flags
local db2 = sqlite3.open("readonly.db", 
    sqlite3.OPEN_READONLY)  -- เปิดอ่านอย่างเดียว
if db2 then
    print("Readonly database opened")
    db2:close()
end

-- เปิดพร้อม URI
local db3 = sqlite3.open("file:data.db?mode=rwc", 
    sqlite3.OPEN_URI)
if db3 then
    print("URI database opened")
    db3:close()
end
```

### ตัวอย่างที่ 3: In-Memory Database

```lua
-- inmemory_db.lua
local sqlite3 = require("lsqlite3")

-- In-memory database (หายไปเมื่อปิด)
local db = sqlite3.open(":memory:")
print("In-memory DB:", db ~= nil)

-- ใช้งานเร็วมาก ไม่มี I/O
db:exec([[
    CREATE TABLE counters (
        name TEXT PRIMARY KEY,
        value INTEGER DEFAULT 0
    );
    INSERT INTO counters VALUES ('hits', 0);
    INSERT INTO counters VALUES ('errors', 0);
]])

-- อัปเดต counter
for i = 1, 100 do
    db:exec("UPDATE counters SET value = value + 1 WHERE name = 'hits'")
end

-- อ่านค่า
for row in db:nrows("SELECT * FROM counters") do
    print(row.name, "=", row.value)
end

db:close()

-- Shared in-memory database (ใช้ร่วมกันได้)
local db_a = sqlite3.open("file:shared?mode=memory&cache=shared",
    sqlite3.OPEN_URI + sqlite3.OPEN_READWRITE + sqlite3.OPEN_CREATE)
local db_b = sqlite3.open("file:shared?mode=memory&cache=shared",
    sqlite3.OPEN_URI + sqlite3.OPEN_READWRITE + sqlite3.OPEN_CREATE)

-- db_a และ db_b ใช้ memory เดียวกัน
if db_a then db_a:close() end
if db_b then db_b:close() end
```

---

## 59.3 CREATE TABLE และ Data Types

### ตัวอย่างที่ 4: สร้างตาราง

```lua
-- create_tables.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

-- SQLite data types: NULL, INTEGER, REAL, TEXT, BLOB
db:exec([[
    CREATE TABLE IF NOT EXISTS users (
        id          INTEGER PRIMARY KEY AUTOINCREMENT,
        username    TEXT    NOT NULL UNIQUE,
        email       TEXT    NOT NULL,
        age         INTEGER CHECK(age >= 0 AND age <= 150),
        score       REAL    DEFAULT 0.0,
        avatar      BLOB,
        created_at  TEXT    DEFAULT (datetime('now')),
        is_active   INTEGER DEFAULT 1
    );
    
    CREATE TABLE IF NOT EXISTS posts (
        id          INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id     INTEGER NOT NULL,
        title       TEXT    NOT NULL,
        content     TEXT,
        views       INTEGER DEFAULT 0,
        created_at  TEXT    DEFAULT (datetime('now')),
        FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
    );
    
    CREATE INDEX IF NOT EXISTS idx_posts_user_id ON posts(user_id);
    CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);
]])

-- ตรวจสอบ tables
for row in db:nrows("SELECT name FROM sqlite_master WHERE type='table'") do
    print("Table:", row.name)
end

-- ตรวจสอบ schema ของ table
for row in db:nrows("PRAGMA table_info(users)") do
    print(string.format("  Column: %-15s Type: %-10s NotNull: %d",
        row.name, row.type, row.notnull))
end

db:close()
```

---

## 59.4 INSERT, SELECT, UPDATE, DELETE

### ตัวอย่างที่ 5: INSERT ข้อมูล

```lua
-- crud_insert.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE products (
        id    INTEGER PRIMARY KEY AUTOINCREMENT,
        name  TEXT    NOT NULL,
        price REAL    NOT NULL,
        stock INTEGER DEFAULT 0
    )
]])

-- INSERT แบบ exec (ง่าย แต่เสี่ยง SQL injection)
db:exec("INSERT INTO products (name, price, stock) VALUES ('Apple', 0.99, 100)")
db:exec("INSERT INTO products (name, price, stock) VALUES ('Banana', 0.49, 200)")

-- INSERT หลายแถวพร้อมกัน
db:exec([[
    INSERT INTO products (name, price, stock) VALUES
        ('Cherry', 2.99, 50),
        ('Dragon Fruit', 5.99, 30),
        ('Elderberry', 8.99, 20)
]])

-- ดู last insert rowid
print("Last inserted ID:", db:last_insert_rowid())

-- ดูจำนวน rows ที่ถูก affect
print("Changes:", db:changes())

-- นับ
for row in db:nrows("SELECT COUNT(*) as cnt FROM products") do
    print("Total products:", row.cnt)
end

db:close()
```

### ตัวอย่างที่ 6: SELECT ข้อมูล

```lua
-- crud_select.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

-- สร้างข้อมูลตัวอย่าง
db:exec([[
    CREATE TABLE employees (
        id         INTEGER PRIMARY KEY,
        name       TEXT NOT NULL,
        department TEXT,
        salary     REAL,
        hire_date  TEXT
    );
    INSERT INTO employees VALUES
        (1, 'Alice', 'Engineering', 95000, '2020-01-15'),
        (2, 'Bob',   'Marketing',   75000, '2019-03-22'),
        (3, 'Carol', 'Engineering', 88000, '2021-06-01'),
        (4, 'Dave',  'HR',          65000, '2018-11-30'),
        (5, 'Eve',   'Engineering', 102000, '2020-08-14');
]])

-- SELECT ทั้งหมด
print("=== All Employees ===")
for row in db:nrows("SELECT * FROM employees ORDER BY name") do
    print(string.format("  [%d] %-10s %-15s $%.0f",
        row.id, row.name, row.department, row.salary))
end

-- SELECT ด้วย WHERE
print("\n=== Engineering Department ===")
for row in db:nrows([[
    SELECT name, salary 
    FROM employees 
    WHERE department = 'Engineering'
    ORDER BY salary DESC
]]) do
    print(string.format("  %-10s $%.0f", row.name, row.salary))
end

-- Aggregate functions
print("\n=== Statistics by Department ===")
for row in db:nrows([[
    SELECT 
        department,
        COUNT(*) as count,
        AVG(salary) as avg_salary,
        MAX(salary) as max_salary,
        MIN(salary) as min_salary
    FROM employees
    GROUP BY department
    ORDER BY avg_salary DESC
]]) do
    print(string.format("  %-15s Count:%-3d Avg:$%-8.0f Max:$%-8.0f Min:$%.0f",
        row.department, row.count, row.avg_salary, 
        row.max_salary, row.min_salary))
end

-- JOIN (ถ้ามีหลายตาราง)
-- SELECT e.name, d.budget FROM employees e JOIN departments d ON e.department = d.name

db:close()
```

### ตัวอย่างที่ 7: UPDATE และ DELETE

```lua
-- crud_update_delete.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE inventory (
        id       INTEGER PRIMARY KEY,
        item     TEXT,
        quantity INTEGER,
        location TEXT
    );
    INSERT INTO inventory VALUES
        (1, 'Widget A', 100, 'Warehouse 1'),
        (2, 'Widget B', 50,  'Warehouse 2'),
        (3, 'Gadget X', 25,  'Warehouse 1'),
        (4, 'Gadget Y', 0,   'Warehouse 3'),
        (5, 'Doohickey', 75, 'Warehouse 2');
]])

-- UPDATE เดียว
db:exec("UPDATE inventory SET quantity = quantity + 50 WHERE id = 2")
print("Updated rows:", db:changes())

-- UPDATE หลาย rows
db:exec([[
    UPDATE inventory 
    SET location = 'Main Warehouse'
    WHERE location = 'Warehouse 1'
]])
print("Moved to Main Warehouse:", db:changes(), "items")

-- UPDATE ด้วย CASE
db:exec([[
    UPDATE inventory 
    SET location = CASE
        WHEN quantity = 0 THEN 'Out of Stock'
        WHEN quantity < 30 THEN 'Low Stock Room'
        ELSE location
    END
]])

-- แสดงผลหลัง UPDATE
print("\nAfter updates:")
for row in db:nrows("SELECT * FROM inventory ORDER BY id") do
    print(string.format("  [%d] %-12s Qty:%-5d Loc:%s",
        row.id, row.item, row.quantity, row.location))
end

-- DELETE
db:exec("DELETE FROM inventory WHERE quantity = 0")
print("\nDeleted zero-quantity items:", db:changes())

-- DELETE ทั้งหมด (แต่ไม่ drop table)
-- db:exec("DELETE FROM inventory")
-- หรือเร็วกว่าคือ
-- db:exec("DELETE FROM inventory") -- triggers ทำงาน
-- db:exec("TRUNCATE TABLE ...") -- ไม่มีใน SQLite

print("Remaining items:")
for row in db:nrows("SELECT COUNT(*) as n FROM inventory") do
    print(" ", row.n, "items")
end

db:close()
```

---

## 59.5 Prepared Statements และ Parameter Binding

### ตัวอย่างที่ 8: Prepared Statements พื้นฐาน

```lua
-- prepared_stmt.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE messages (
        id      INTEGER PRIMARY KEY,
        from_id INTEGER,
        to_id   INTEGER,
        content TEXT,
        sent_at TEXT DEFAULT (datetime('now'))
    )
]])

-- Prepare statement (ทำครั้งเดียว ใช้ซ้ำได้)
local insert_stmt = db:prepare([[
    INSERT INTO messages (from_id, to_id, content)
    VALUES (?, ?, ?)
]])

-- Insert หลาย rows ด้วย prepared statement
local messages = {
    {1, 2, "Hello Bob!"},
    {2, 1, "Hi Alice!"},
    {1, 3, "Hey Carol!"},
    {3, 1, "What's up?"},
    {2, 3, "Meeting at 3pm"},
}

for _, msg in ipairs(messages) do
    insert_stmt:bind_values(msg[1], msg[2], msg[3])
    insert_stmt:step()
    insert_stmt:reset()
end

insert_stmt:finalize()
print("Inserted messages:", db:changes())

-- Prepared SELECT
local select_stmt = db:prepare([[
    SELECT m.*, 
           f.username as from_name,
           t.username as to_name
    FROM messages m
    WHERE m.from_id = ?
    ORDER BY m.sent_at
]])

-- รองรับ username lookup (สมมติ)
-- ในตัวอย่างนี้แค่แสดง messages
local query_stmt = db:prepare("SELECT * FROM messages WHERE from_id = ?")
query_stmt:bind_values(1)
print("\nMessages from user 1:")
for row in query_stmt:nrows() do
    print(string.format("  To:%d Content:%s", row.to_id, row.content))
end
query_stmt:finalize()

db:close()
```

### ตัวอย่างที่ 9: Named Parameters

```lua
-- named_params.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE events (
        id          INTEGER PRIMARY KEY,
        name        TEXT NOT NULL,
        category    TEXT,
        start_date  TEXT,
        end_date    TEXT,
        max_attend  INTEGER DEFAULT 100,
        price       REAL DEFAULT 0
    )
]])

-- Named parameters ด้วย :name หรือ @name หรือ $name
local stmt = db:prepare([[
    INSERT INTO events (name, category, start_date, end_date, max_attend, price)
    VALUES (:name, :category, :start_date, :end_date, :max_attend, :price)
]])

local events = {
    {name = "Tech Conference", category = "Technology",
     start_date = "2025-03-15", end_date = "2025-03-17",
     max_attend = 500, price = 299.99},
    {name = "Jazz Festival", category = "Music",
     start_date = "2025-04-20", end_date = "2025-04-21",
     max_attend = 2000, price = 45.00},
    {name = "Art Exhibition", category = "Art",
     start_date = "2025-05-01", end_date = "2025-05-31",
     max_attend = 0, price = 15.00},
}

for _, event in ipairs(events) do
    stmt:bind_names(event)
    stmt:step()
    stmt:reset()
end

stmt:finalize()

-- Query ด้วย named params
local search = db:prepare("SELECT * FROM events WHERE category = :cat")
search:bind_names({cat = "Technology"})

print("Technology events:")
for row in search:nrows() do
    print(string.format("  %s (%s to %s) $%.2f",
        row.name, row.start_date, row.end_date, row.price))
end

search:finalize()
db:close()
```

### ตัวอย่างที่ 10: Bind ประเภทต่างๆ

```lua
-- bind_types.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE data_types (
        id      INTEGER PRIMARY KEY,
        int_val INTEGER,
        real_val REAL,
        text_val TEXT,
        blob_val BLOB,
        null_val TEXT
    )
]])

local stmt = db:prepare([[
    INSERT INTO data_types 
    (int_val, real_val, text_val, blob_val, null_val)
    VALUES (?, ?, ?, ?, ?)
]])

-- bind แต่ละ type
stmt:bind(1, 42)                          -- INTEGER
stmt:bind(2, 3.14159)                     -- REAL
stmt:bind(3, "Hello, SQLite!")            -- TEXT
stmt:bind_blob(4, "\x00\x01\x02\x03")    -- BLOB (binary data)
stmt:bind_null(5)                         -- NULL

stmt:step()
stmt:reset()
stmt:finalize()

-- อ่านข้อมูลและตรวจสอบ type
local q = db:prepare("SELECT * FROM data_types LIMIT 1")
q:step()

print("Column types:")
for i = 0, q:columns() - 1 do
    print(string.format("  [%d] %-10s type=%d value=%s",
        i, q:get_name(i), q:get_column_type(i),
        tostring(q:get_value(i))))
end

-- column type constants:
-- 1 = INTEGER, 2 = FLOAT, 3 = TEXT, 4 = BLOB, 5 = NULL

q:finalize()
db:close()
```

---

## 59.6 Transactions

### ตัวอย่างที่ 11: Basic Transactions

```lua
-- transactions.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE accounts (
        id      INTEGER PRIMARY KEY,
        name    TEXT,
        balance REAL
    );
    INSERT INTO accounts VALUES (1, 'Alice', 1000.00);
    INSERT INTO accounts VALUES (2, 'Bob', 500.00);
]])

-- ฟังก์ชัน transfer ด้วย transaction
local function transfer(from_id, to_id, amount)
    -- เริ่ม transaction
    db:exec("BEGIN TRANSACTION")
    
    local ok = true
    local err_msg = ""
    
    -- ตรวจสอบ balance
    local stmt = db:prepare("SELECT balance FROM accounts WHERE id = ?")
    stmt:bind_values(from_id)
    stmt:step()
    local balance = stmt:get_value(0)
    stmt:finalize()
    
    if balance < amount then
        ok = false
        err_msg = "Insufficient funds"
    else
        -- หัก balance จากผู้โอน
        local debit = db:prepare(
            "UPDATE accounts SET balance = balance - ? WHERE id = ?")
        debit:bind_values(amount, from_id)
        debit:step()
        debit:finalize()
        
        -- เพิ่ม balance ให้ผู้รับ
        local credit = db:prepare(
            "UPDATE accounts SET balance = balance + ? WHERE id = ?")
        credit:bind_values(amount, to_id)
        credit:step()
        credit:finalize()
    end
    
    if ok then
        db:exec("COMMIT")
        print(string.format("Transfer $%.2f: SUCCESS", amount))
    else
        db:exec("ROLLBACK")
        print(string.format("Transfer $%.2f: FAILED (%s)", amount, err_msg))
    end
end

-- ทดสอบ
print("=== Before ===")
for row in db:nrows("SELECT * FROM accounts") do
    print(string.format("  %s: $%.2f", row.name, row.balance))
end

transfer(1, 2, 200.00)   -- สำเร็จ
transfer(2, 1, 1000.00)  -- ล้มเหลว (insufficient)

print("\n=== After ===")
for row in db:nrows("SELECT * FROM accounts") do
    print(string.format("  %s: $%.2f", row.name, row.balance))
end

db:close()
```

### ตัวอย่างที่ 12: Savepoints

```lua
-- savepoints.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec("CREATE TABLE log (id INTEGER PRIMARY KEY, msg TEXT)")

-- Nested transactions ด้วย SAVEPOINT
db:exec("BEGIN")

db:exec("INSERT INTO log (msg) VALUES ('Step 1')")

db:exec("SAVEPOINT sp1")
db:exec("INSERT INTO log (msg) VALUES ('Step 2')")
db:exec("INSERT INTO log (msg) VALUES ('Step 3')")

-- ทดสอบ rollback to savepoint
db:exec("SAVEPOINT sp2")
db:exec("INSERT INTO log (msg) VALUES ('Step 4 - will rollback')")
db:exec("INSERT INTO log (msg) VALUES ('Step 5 - will rollback')")

-- Rollback ไปถึง sp2 (undo step 4 และ 5)
db:exec("ROLLBACK TO SAVEPOINT sp2")
db:exec("RELEASE SAVEPOINT sp2")

-- ต่อจาก sp1
db:exec("INSERT INTO log (msg) VALUES ('Step 4 - actual')")

db:exec("RELEASE SAVEPOINT sp1")

-- Commit ทั้งหมด
db:exec("COMMIT")

-- ผลลัพธ์
print("Log entries:")
for row in db:nrows("SELECT * FROM log ORDER BY id") do
    print("  " .. row.id .. ": " .. row.msg)
end
-- ควรเห็น: Step 1, Step 2, Step 3, Step 4 - actual

db:close()
```

### ตัวอย่างที่ 13: Transaction Helper

```lua
-- transaction_helper.lua
local sqlite3 = require("lsqlite3")

-- Helper function สำหรับ transaction
local function with_transaction(db, func)
    db:exec("BEGIN")
    local ok, err = pcall(func)
    if ok then
        db:exec("COMMIT")
    else
        db:exec("ROLLBACK")
        error(err, 2)
    end
end

-- ใช้งาน
local db = sqlite3.open(":memory:")
db:exec("CREATE TABLE t (id INTEGER PRIMARY KEY, v INTEGER)")

-- สำเร็จ
with_transaction(db, function()
    for i = 1, 10 do
        db:exec("INSERT INTO t VALUES (" .. i .. ", " .. i*100 .. ")")
    end
end)

-- ล้มเหลว (rollback)
local ok, err = pcall(with_transaction, db, function()
    db:exec("INSERT INTO t VALUES (11, 1100)")
    error("Simulated error!")  -- จะ trigger rollback
end)

print("Transaction failed:", not ok, "Error:", err)

for row in db:nrows("SELECT COUNT(*) as n FROM t") do
    print("Rows in table:", row.n)  -- 10 (ไม่ใช่ 11)
end

db:close()
```

---

## 59.7 Error Handling

### ตัวอย่างที่ 14: Error Handling ครบถ้วน

```lua
-- error_handling.lua
local sqlite3 = require("lsqlite3")

-- Error codes ที่สำคัญ
local ERROR_NAMES = {
    [sqlite3.OK]         = "OK",
    [sqlite3.ERROR]      = "ERROR",
    [sqlite3.INTERNAL]   = "INTERNAL",
    [sqlite3.PERM]       = "PERM",
    [sqlite3.ABORT]      = "ABORT",
    [sqlite3.BUSY]       = "BUSY",
    [sqlite3.LOCKED]     = "LOCKED",
    [sqlite3.NOMEM]      = "NOMEM",
    [sqlite3.READONLY]   = "READONLY",
    [sqlite3.INTERRUPT]  = "INTERRUPT",
    [sqlite3.IOERR]      = "IOERR",
    [sqlite3.CORRUPT]    = "CORRUPT",
    [sqlite3.NOTFOUND]   = "NOTFOUND",
    [sqlite3.FULL]       = "FULL",
    [sqlite3.CANTOPEN]   = "CANTOPEN",
    [sqlite3.CONSTRAINT] = "CONSTRAINT",
    [sqlite3.MISMATCH]   = "MISMATCH",
    [sqlite3.MISUSE]     = "MISUSE",
    [sqlite3.TOOBIG]     = "TOOBIG",
    [sqlite3.RANGE]      = "RANGE",
    [sqlite3.NOTADB]     = "NOTADB",
}

local function db_error(db, msg)
    return string.format("[%s] %s: %s",
        ERROR_NAMES[db:error_code()] or tostring(db:error_code()),
        msg,
        db:errmsg()
    )
end

local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE users (
        id       INTEGER PRIMARY KEY,
        username TEXT UNIQUE NOT NULL,
        email    TEXT UNIQUE NOT NULL
    )
]])

-- ลอง insert ข้อมูลซ้ำ (UNIQUE constraint violation)
local stmt = db:prepare("INSERT INTO users (username, email) VALUES (?, ?)")

local function safe_insert(username, email)
    stmt:bind_values(username, email)
    local rc = stmt:step()
    stmt:reset()
    
    if rc == sqlite3.DONE then
        return true, db:last_insert_rowid()
    elseif rc == sqlite3.CONSTRAINT then
        return false, "Duplicate username or email"
    else
        return false, db_error(db, "INSERT failed")
    end
end

local ok, result = safe_insert("alice", "alice@example.com")
print("Insert alice:", ok, result)  -- true, 1

ok, result = safe_insert("bob", "bob@example.com")
print("Insert bob:", ok, result)    -- true, 2

ok, result = safe_insert("alice", "alice2@example.com")
print("Insert alice dup:", ok, result)  -- false, Duplicate...

ok, result = safe_insert("charlie", "bob@example.com")
print("Insert charlie dup email:", ok, result)  -- false, Duplicate...

stmt:finalize()
db:close()
```

### ตัวอย่างที่ 15: Error Callback

```lua
-- error_callback.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

-- ตั้ง error callback (บาง versions ของ LuaSQLite3 รองรับ)
-- db:set_authorizer(function(...) ... end)

-- Wrap exec ด้วย error checking
local function exec(database, sql)
    local rc = database:exec(sql)
    if rc ~= sqlite3.OK then
        error(string.format("SQL Error [%d]: %s\nSQL: %s",
            rc, database:errmsg(), sql))
    end
    return rc
end

local ok, err = pcall(exec, db, "CREATE TABLE t (id INTEGER PRIMARY KEY)")
print("Create table:", ok)

ok, err = pcall(exec, db, "INVALID SQL STATEMENT !!!")
print("Invalid SQL:", ok, err and err:match("SQL Error.*"))

ok, err = pcall(exec, db, "INSERT INTO nonexistent VALUES (1)")
print("Insert to nonexistent:", ok, err and "error caught" or "no error")

db:close()
```

---

## 59.8 Working with NULL Values

### ตัวอย่างที่ 16: NULL handling

```lua
-- null_values.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE contacts (
        id       INTEGER PRIMARY KEY,
        name     TEXT NOT NULL,
        phone    TEXT,        -- nullable
        email    TEXT,        -- nullable
        address  TEXT,        -- nullable
        notes    TEXT         -- nullable
    );
    INSERT INTO contacts VALUES
        (1, 'Alice', '+1-555-0100', 'alice@example.com', '123 Main St', NULL),
        (2, 'Bob',   NULL,         'bob@example.com',   NULL,          'VIP customer'),
        (3, 'Carol', '+1-555-0300', NULL,               '456 Oak Ave', NULL),
        (4, 'Dave',  NULL,         NULL,                NULL,          NULL);
]])

-- ค้นหาที่มี NULL
print("Contacts without phone:")
for row in db:nrows("SELECT name FROM contacts WHERE phone IS NULL") do
    print(" ", row.name)
end

-- ค้นหาที่ไม่มี NULL
print("\nContacts with complete info:")
for row in db:nrows([[
    SELECT name FROM contacts 
    WHERE phone IS NOT NULL AND email IS NOT NULL
]]) do
    print(" ", row.name)
end

-- COALESCE - ใช้ค่าแรกที่ไม่ใช่ NULL
print("\nWith COALESCE:")
for row in db:nrows([[
    SELECT 
        name,
        COALESCE(phone, email, 'No contact info') as contact
    FROM contacts
    ORDER BY id
]]) do
    print(string.format("  %-10s -> %s", row.name, row.contact))
end

-- NULL in Lua (LuaSQLite3 return nil สำหรับ NULL)
for row in db:nrows("SELECT * FROM contacts") do
    local has_null = false
    for k, v in pairs(row) do
        if v == nil then has_null = true; break end
    end
    if has_null then
        -- ตรวจสอบแต่ละ field
        local info = {}
        if row.phone == nil then table.insert(info, "no phone") end
        if row.email == nil then table.insert(info, "no email") end
        print(string.format("  %s: %s", row.name, table.concat(info, ", ")))
    end
end

db:close()
```

---

## 59.9 Reading Results แบบต่างๆ

### ตัวอย่างที่ 17: วิธีอ่าน Results

```lua
-- read_results.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE scores (
        player TEXT,
        game   TEXT,
        score  INTEGER
    );
    INSERT INTO scores VALUES
        ('Alice', 'Chess',    1850),
        ('Bob',   'Chess',    1720),
        ('Alice', 'Go',       2100),
        ('Carol', 'Chess',    1650),
        ('Bob',   'Go',       1900),
        ('Carol', 'Go',       2050);
]])

-- วิธี 1: nrows() - iterator ที่ return table
print("=== nrows() ===")
for row in db:nrows("SELECT * FROM scores") do
    print(string.format("  %-8s %-8s %d", row.player, row.game, row.score))
end

-- วิธี 2: rows() - return array (ต้องรู้ column order)
print("\n=== rows() ===")
for row in db:rows("SELECT player, game, score FROM scores") do
    print(string.format("  %-8s %-8s %d", row[1], row[2], row[3]))
end

-- วิธี 3: urows() - return values แยก
print("\n=== urows() ===")
for player, game, score in db:urows("SELECT player, game, score FROM scores") do
    print(string.format("  %-8s %-8s %d", player, game, score))
end

-- วิธี 4: prepare + step (manual iteration)
print("\n=== prepare + step ===")
local stmt = db:prepare("SELECT * FROM scores WHERE score > ?")
stmt:bind_values(1800)
while stmt:step() == sqlite3.ROW do
    print(string.format("  %-8s %-8s %d",
        stmt:get_value(0), stmt:get_value(1), stmt:get_value(2)))
end
stmt:finalize()

-- วิธี 5: collect all to table
local function query_all(database, sql, ...)
    local results = {}
    local stmt2 = database:prepare(sql)
    if select('#', ...) > 0 then
        stmt2:bind_values(...)
    end
    for row in stmt2:nrows() do
        table.insert(results, row)
    end
    stmt2:finalize()
    return results
end

local high_scores = query_all(db, "SELECT * FROM scores WHERE score > ?", 2000)
print("\nHigh scores (>2000):", #high_scores)
for _, row in ipairs(high_scores) do
    print(string.format("  %s: %d", row.player, row.score))
end

db:close()
```

---

## 59.10 Simple ORM Implementation

### ตัวอย่างที่ 18: ORM พื้นฐาน

```lua
-- simple_orm.lua
local sqlite3 = require("lsqlite3")

-- Base Model class
local Model = {}
Model.__index = Model

function Model:new(attrs)
    local instance = setmetatable({}, self)
    self.__index = self
    for k, v in pairs(attrs or {}) do
        instance[k] = v
    end
    return instance
end

function Model:save()
    if self.id then
        return self:update()
    else
        return self:insert()
    end
end

function Model:insert()
    local cols = {}
    local vals = {}
    local placeholders = {}
    
    for _, col in ipairs(self.__columns) do
        if col ~= "id" and self[col] ~= nil then
            table.insert(cols, col)
            table.insert(vals, self[col])
            table.insert(placeholders, "?")
        end
    end
    
    local sql = string.format(
        "INSERT INTO %s (%s) VALUES (%s)",
        self.__table,
        table.concat(cols, ", "),
        table.concat(placeholders, ", ")
    )
    
    local stmt = self.__db:prepare(sql)
    stmt:bind_values(table.unpack(vals))
    local rc = stmt:step()
    stmt:finalize()
    
    if rc == sqlite3.DONE then
        self.id = self.__db:last_insert_rowid()
        return true
    end
    return false
end

function Model:update()
    local sets = {}
    local vals = {}
    
    for _, col in ipairs(self.__columns) do
        if col ~= "id" and self[col] ~= nil then
            table.insert(sets, col .. " = ?")
            table.insert(vals, self[col])
        end
    end
    table.insert(vals, self.id)
    
    local sql = string.format(
        "UPDATE %s SET %s WHERE id = ?",
        self.__table,
        table.concat(sets, ", ")
    )
    
    local stmt = self.__db:prepare(sql)
    stmt:bind_values(table.unpack(vals))
    stmt:step()
    stmt:finalize()
    return true
end

-- สร้าง Model class
local function define_model(db, table_name, columns)
    local cls = setmetatable({}, {__index = Model})
    cls.__index = cls
    cls.__db = db
    cls.__table = table_name
    cls.__columns = columns
    
    cls.find = function(id)
        local stmt = db:prepare("SELECT * FROM " .. table_name .. " WHERE id = ?")
        stmt:bind_values(id)
        if stmt:step() == sqlite3.ROW then
            local row = {}
            for _, col in ipairs(columns) do
                row[col] = stmt:get_named_value(col)
            end
            stmt:finalize()
            return cls:new(row)
        end
        stmt:finalize()
        return nil
    end
    
    cls.where = function(condition, ...)
        local results = {}
        local sql = "SELECT * FROM " .. table_name
        if condition then sql = sql .. " WHERE " .. condition end
        local stmt = db:prepare(sql)
        if select('#', ...) > 0 then
            stmt:bind_values(...)
        end
        for row in stmt:nrows() do
            table.insert(results, cls:new(row))
        end
        stmt:finalize()
        return results
    end
    
    cls.all = function()
        return cls.where()
    end
    
    return cls
end

-- ทดสอบ
local db = sqlite3.open(":memory:")
db:exec([[
    CREATE TABLE todos (
        id       INTEGER PRIMARY KEY AUTOINCREMENT,
        title    TEXT NOT NULL,
        done     INTEGER DEFAULT 0,
        priority INTEGER DEFAULT 1
    )
]])

local Todo = define_model(db, "todos", {"id", "title", "done", "priority"})

-- สร้าง todos
local t1 = Todo:new({title = "Buy groceries", priority = 2})
t1:save()

local t2 = Todo:new({title = "Write report", priority = 3})
t2:save()

local t3 = Todo:new({title = "Exercise", done = 0, priority = 1})
t3:save()

-- อ่านทั้งหมด
print("All todos:")
for _, todo in ipairs(Todo.all()) do
    print(string.format("  [%d] %s (pri=%d, done=%d)",
        todo.id, todo.title, todo.priority, todo.done or 0))
end

-- หา by id
local found = Todo.find(2)
print("\nFind id=2:", found and found.title or "not found")

-- Update
found.done = 1
found:save()
print("Updated todo:", Todo.find(2).done == 1 and "marked done" or "not done")

-- Where clause
print("\nHigh priority (>=2):")
for _, todo in ipairs(Todo.where("priority >= ?", 2)) do
    print("  " .. todo.title)
end

db:close()
```

---

## 59.11 Database Migrations

### ตัวอย่างที่ 19: Migration System

```lua
-- migrations.lua
local sqlite3 = require("lsqlite3")

local function create_migration_table(db)
    db:exec([[
        CREATE TABLE IF NOT EXISTS schema_migrations (
            version    INTEGER PRIMARY KEY,
            name       TEXT NOT NULL,
            applied_at TEXT DEFAULT (datetime('now'))
        )
    ]])
end

local function get_current_version(db)
    local version = 0
    for row in db:nrows("SELECT MAX(version) as v FROM schema_migrations") do
        version = row.v or 0
    end
    return version
end

local function run_migrations(db, migrations)
    create_migration_table(db)
    local current = get_current_version(db)
    print("Current schema version:", current)
    
    local applied = 0
    for _, migration in ipairs(migrations) do
        if migration.version > current then
            print(string.format("Applying migration %d: %s",
                migration.version, migration.name))
            
            local ok, err = pcall(function()
                db:exec("BEGIN")
                migration.up(db)
                local stmt = db:prepare([[
                    INSERT INTO schema_migrations (version, name)
                    VALUES (?, ?)
                ]])
                stmt:bind_values(migration.version, migration.name)
                stmt:step()
                stmt:finalize()
                db:exec("COMMIT")
            end)
            
            if not ok then
                db:exec("ROLLBACK")
                error("Migration failed: " .. tostring(err))
            end
            applied = applied + 1
        end
    end
    
    print(string.format("Applied %d migrations. Current version: %d",
        applied, get_current_version(db)))
end

-- ตาราง migrations
local migrations = {
    {
        version = 1,
        name    = "create_users",
        up = function(db)
            db:exec([[
                CREATE TABLE users (
                    id       INTEGER PRIMARY KEY,
                    username TEXT UNIQUE NOT NULL,
                    email    TEXT UNIQUE NOT NULL
                )
            ]])
        end
    },
    {
        version = 2,
        name    = "add_user_profile",
        up = function(db)
            db:exec([[
                ALTER TABLE users ADD COLUMN bio TEXT;
                ALTER TABLE users ADD COLUMN avatar_url TEXT;
                ALTER TABLE users ADD COLUMN created_at TEXT DEFAULT (datetime('now'));
            ]])
        end
    },
    {
        version = 3,
        name    = "create_posts",
        up = function(db)
            db:exec([[
                CREATE TABLE posts (
                    id         INTEGER PRIMARY KEY,
                    user_id    INTEGER NOT NULL REFERENCES users(id),
                    title      TEXT NOT NULL,
                    body       TEXT,
                    published  INTEGER DEFAULT 0,
                    created_at TEXT DEFAULT (datetime('now'))
                );
                CREATE INDEX idx_posts_user ON posts(user_id);
            ]])
        end
    },
}

-- รัน migrations
local db = sqlite3.open(":memory:")
run_migrations(db, migrations)

-- แสดงผล schema
print("\nSchema migrations history:")
for row in db:nrows("SELECT * FROM schema_migrations ORDER BY version") do
    print(string.format("  v%d: %s (%s)",
        row.version, row.name, row.applied_at))
end

-- แสดง tables
print("\nTables created:")
for row in db:nrows([[
    SELECT name FROM sqlite_master 
    WHERE type='table' AND name NOT LIKE 'sqlite_%'
    ORDER BY name
]]) do
    print("  " .. row.name)
end

db:close()
```

---

## 59.12 Advanced Features

### ตัวอย่างที่ 20: Full-Text Search

```lua
-- fts_search.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

-- สร้าง FTS5 virtual table
db:exec([[
    CREATE VIRTUAL TABLE articles USING fts5(
        title,
        body,
        author,
        content='',
        tokenize='porter unicode61'
    )
]])

-- Insert articles
local stmt = db:prepare(
    "INSERT INTO articles (title, body, author) VALUES (?, ?, ?)")
local articles = {
    {"Introduction to Lua", "Lua is a powerful scripting language created in Brazil.", "Alice"},
    {"Advanced Lua Tables", "Tables are the primary data structure in Lua programming.", "Bob"},
    {"Lua for Game Development", "Many games use Lua for scripting game logic and AI.", "Alice"},
    {"Web Development with Lua", "OpenResty brings Lua to web server development with nginx.", "Carol"},
    {"Machine Learning Basics", "Python and R are popular languages for machine learning.", "Dave"},
}

for _, a in ipairs(articles) do
    stmt:bind_values(a[1], a[2], a[3])
    stmt:step()
    stmt:reset()
end
stmt:finalize()

-- Full-text search
print("=== Search 'Lua' ===")
for row in db:nrows("SELECT title, author FROM articles WHERE articles MATCH 'Lua'") do
    print(string.format("  '%s' by %s", row.title, row.author))
end

print("\n=== Search 'game OR web' ===")
for row in db:nrows([[
    SELECT title, rank FROM articles 
    WHERE articles MATCH 'game OR web'
    ORDER BY rank
]]) do
    print(string.format("  '%s' (rank=%.4f)", row.title, row.rank or 0))
end

-- Prefix search
print("\n=== Prefix 'script*' ===")
for row in db:nrows("SELECT title FROM articles WHERE articles MATCH 'script*'") do
    print("  " .. row.title)
end

db:close()
```

### ตัวอย่างที่ 21: Custom Functions

```lua
-- custom_functions.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

-- ลงทะเบียน Lua functions เป็น SQL functions
db:create_function("double", 1, function(ctx, n)
    ctx:result_number(n * 2)
end)

db:create_function("celsius_to_f", 1, function(ctx, c)
    ctx:result_number(c * 9/5 + 32)
end)

db:create_function("word_count", 1, function(ctx, text)
    if not text then
        ctx:result_null()
        return
    end
    local count = 0
    for _ in tostring(text):gmatch("%S+") do
        count = count + 1
    end
    ctx:result_int(count)
end)

-- Aggregate function
db:create_aggregate("my_sum", 1, 
    function(ctx, n)
        ctx:set_aggregate_context(
            (ctx:get_aggregate_context() or 0) + (n or 0))
    end,
    function(ctx)
        ctx:result_number(ctx:get_aggregate_context() or 0)
    end
)

-- ทดสอบ
db:exec([[
    CREATE TABLE temperatures (
        city    TEXT,
        celsius REAL
    );
    INSERT INTO temperatures VALUES
        ('Bangkok',  35.0),
        ('London',   15.0),
        ('New York', 22.0),
        ('Tokyo',    28.0);
]])

print("Custom functions:")
for row in db:nrows([[
    SELECT city, celsius, celsius_to_f(celsius) as fahrenheit
    FROM temperatures
    ORDER BY celsius DESC
]]) do
    print(string.format("  %-12s %.1f°C = %.1f°F",
        row.city, row.celsius, row.fahrenheit))
end

print("\ndouble(5) =", db:first_irow("SELECT double(5)")[1])

-- word_count
db:exec("CREATE TABLE texts (content TEXT)")
db:exec("INSERT INTO texts VALUES ('Hello World this is Lua')")
db:exec("INSERT INTO texts VALUES ('Short text')")
for row in db:nrows("SELECT content, word_count(content) as wc FROM texts") do
    print(string.format("  '%s' -> %d words", row.content, row.wc))
end

db:close()
```

### ตัวอย่างที่ 22: Backup Database

```lua
-- backup_db.lua
local sqlite3 = require("lsqlite3")

-- สร้าง source database
local src = sqlite3.open(":memory:")
src:exec([[
    CREATE TABLE data (id INTEGER PRIMARY KEY, value TEXT);
    INSERT INTO data VALUES (1, 'Hello');
    INSERT INTO data VALUES (2, 'World');
    INSERT INTO data VALUES (3, 'Lua');
]])

-- Backup ไป file
local dst = sqlite3.open("/tmp/backup.db")

-- Manual backup ด้วย SQL
dst:exec("BEGIN")
for row in src:nrows("SELECT name FROM sqlite_master WHERE type='table'") do
    -- drop ถ้ามีอยู่แล้ว
    dst:exec("DROP TABLE IF EXISTS " .. row.name)
    
    -- ดึง CREATE TABLE statement
    for cr in src:nrows(string.format(
        "SELECT sql FROM sqlite_master WHERE type='table' AND name='%s'",
        row.name)) do
        dst:exec(cr.sql)
    end
    
    -- copy rows
    for data_row in src:nrows("SELECT * FROM " .. row.name) do
        local vals = {}
        for k, v in pairs(data_row) do
            if type(v) == "string" then
                table.insert(vals, string.format("'%s'", v:gsub("'", "''")))
            elseif v == nil then
                table.insert(vals, "NULL")
            else
                table.insert(vals, tostring(v))
            end
        end
        dst:exec(string.format("INSERT INTO %s VALUES (%s)",
            row.name, table.concat(vals, ",")))
    end
end
dst:exec("COMMIT")

print("Backup complete!")
for row in dst:nrows("SELECT * FROM data") do
    print(string.format("  [%d] %s", row.id, row.value))
end

src:close()
dst:close()

-- ลบ backup file
os.remove("/tmp/backup.db")
```

---

## 59.13 Connection Pool Pattern

### ตัวอย่างที่ 23: Connection Pool

```lua
-- connection_pool.lua
local sqlite3 = require("lsqlite3")

local ConnectionPool = {}
ConnectionPool.__index = ConnectionPool

function ConnectionPool.new(db_path, pool_size)
    local self = setmetatable({}, ConnectionPool)
    self.db_path = db_path
    self.pool_size = pool_size or 5
    self.connections = {}
    self.available = {}
    
    -- สร้าง connections ล่วงหน้า
    for i = 1, pool_size do
        local db = sqlite3.open(db_path)
        -- เปิด WAL mode สำหรับ concurrency
        db:exec("PRAGMA journal_mode=WAL")
        db:exec("PRAGMA synchronous=NORMAL")
        self.connections[i] = db
        table.insert(self.available, i)
    end
    
    return self
end

function ConnectionPool:acquire(timeout)
    local deadline = os.clock() + (timeout or 5)
    
    while os.clock() < deadline do
        if #self.available > 0 then
            local idx = table.remove(self.available)
            return self.connections[idx], idx
        end
        -- busy wait (ใน production ควรใช้ coroutine หรือ async)
        -- สำหรับตัวอย่างนี้ simulate ด้วย loop
    end
    
    error("Connection pool timeout")
end

function ConnectionPool:release(idx)
    table.insert(self.available, idx)
end

function ConnectionPool:execute(sql, ...)
    local db, idx = self:acquire()
    local results = {}
    
    local ok, err = pcall(function()
        local stmt = db:prepare(sql)
        if select('#', ...) > 0 then
            stmt:bind_values(...)
        end
        for row in stmt:nrows() do
            table.insert(results, row)
        end
        stmt:finalize()
    end)
    
    self:release(idx)
    
    if not ok then error(err) end
    return results
end

function ConnectionPool:close()
    for _, db in ipairs(self.connections) do
        db:close()
    end
    self.connections = {}
    self.available = {}
end

-- ใช้งาน (single-threaded example)
local pool = ConnectionPool.new(":memory:", 3)

-- ทุก connection ต้องมี schema เดียวกัน
for i = 1, 3 do
    pool.connections[i]:exec([[
        CREATE TABLE IF NOT EXISTS kv (
            key TEXT PRIMARY KEY,
            value TEXT
        )
    ]])
end

-- Insert ผ่าน pool
for i = 1, 5 do
    local db, idx = pool:acquire()
    local stmt = db:prepare("INSERT OR REPLACE INTO kv VALUES (?, ?)")
    stmt:bind_values("key_" .. i, "value_" .. i)
    stmt:step()
    stmt:finalize()
    pool:release(idx)
end

-- Query ผ่าน pool
local results = pool:execute("SELECT * FROM kv ORDER BY key")
print("KV Store contents:")
for _, row in ipairs(results) do
    print(string.format("  %s = %s", row.key, row.value))
end

print("Pool size:", pool.pool_size)
print("Available connections:", #pool.available)

pool:close()
```

---

## 59.14 Performance Tips

### ตัวอย่างที่ 24: Bulk Insert Performance

```lua
-- bulk_insert.lua
local sqlite3 = require("lsqlite3")

local function benchmark_insert(mode, n)
    local db = sqlite3.open(":memory:")
    db:exec("CREATE TABLE data (id INTEGER, value REAL)")
    
    -- Configure SQLite for speed
    if mode == "fast" then
        db:exec("PRAGMA synchronous = OFF")
        db:exec("PRAGMA journal_mode = MEMORY")
        db:exec("PRAGMA cache_size = 10000")
        db:exec("PRAGMA temp_store = MEMORY")
    end
    
    local stmt = db:prepare("INSERT INTO data VALUES (?, ?)")
    local t = os.clock()
    
    if mode == "no_transaction" then
        for i = 1, n do
            stmt:bind_values(i, math.random())
            stmt:step()
            stmt:reset()
        end
    else
        db:exec("BEGIN")
        for i = 1, n do
            stmt:bind_values(i, math.random())
            stmt:step()
            stmt:reset()
        end
        db:exec("COMMIT")
    end
    
    local elapsed = os.clock() - t
    stmt:finalize()
    
    local count_result
    for row in db:nrows("SELECT COUNT(*) as n FROM data") do
        count_result = row.n
    end
    
    db:close()
    return elapsed, count_result
end

local N = 100000
print(string.format("Inserting %d rows...", N))

local t1, c1 = benchmark_insert("no_transaction", N)
print(string.format("No transaction:     %.3fs (%d rows)", t1, c1))

local t2, c2 = benchmark_insert("transaction", N)
print(string.format("With transaction:   %.3fs (%d rows) - %.1fx faster", 
    t2, c2, t1/t2))

local t3, c3 = benchmark_insert("fast", N)
print(string.format("Fast pragma:        %.3fs (%d rows) - %.1fx faster",
    t3, c3, t1/t3))
```

### ตัวอย่างที่ 25: Index Optimization

```lua
-- index_optimization.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE orders (
        id          INTEGER PRIMARY KEY,
        customer_id INTEGER,
        product_id  INTEGER,
        quantity    INTEGER,
        total       REAL,
        status      TEXT,
        created_at  TEXT
    )
]])

-- Insert test data
local stmt = db:prepare([[
    INSERT INTO orders VALUES (?, ?, ?, ?, ?, ?, datetime('now', ?))
]])
db:exec("BEGIN")
for i = 1, 10000 do
    stmt:bind_values(
        i,
        math.random(1, 100),
        math.random(1, 50),
        math.random(1, 10),
        math.random(10, 1000) * 0.99,
        ({[1]="pending",[2]="shipped",[3]="delivered",[4]="cancelled"})[math.random(4)],
        string.format("-%d days", math.random(0, 365))
    )
    stmt:step()
    stmt:reset()
end
db:exec("COMMIT")
stmt:finalize()

-- EXPLAIN QUERY PLAN
print("=== Query Plan WITHOUT index ===")
for row in db:nrows([[
    EXPLAIN QUERY PLAN
    SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending'
]]) do
    print("  " .. (row.detail or row[4] or ""))
end

-- สร้าง index
db:exec("CREATE INDEX idx_cust_status ON orders(customer_id, status)")

print("\n=== Query Plan WITH index ===")
for row in db:nrows([[
    EXPLAIN QUERY PLAN
    SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending'
]]) do
    print("  " .. (row.detail or row[4] or ""))
end

-- วัดเวลา
local function time_query(label, sql)
    local t = os.clock()
    local n = 0
    for _ in db:nrows(sql) do n = n + 1 end
    print(string.format("  %-30s %.4fs (%d rows)", label, os.clock()-t, n))
end

print("\n=== Query Times ===")
time_query("customer_id filter:", 
    "SELECT * FROM orders WHERE customer_id = 42")
time_query("status filter:",
    "SELECT * FROM orders WHERE status = 'pending'")
time_query("combined filter:",
    "SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending'")

db:close()
```

---

## 59.15 ตัวอย่างครบวงจร: Task Manager

### ตัวอย่างที่ 26: Task Manager Application

```lua
-- task_manager.lua
local sqlite3 = require("lsqlite3")

local TaskDB = {}
TaskDB.__index = TaskDB

function TaskDB.new(path)
    local self = setmetatable({}, TaskDB)
    self.db = sqlite3.open(path or ":memory:")
    self:_init()
    return self
end

function TaskDB:_init()
    self.db:exec([[
        PRAGMA foreign_keys = ON;
        PRAGMA journal_mode = WAL;
        
        CREATE TABLE IF NOT EXISTS projects (
            id         INTEGER PRIMARY KEY AUTOINCREMENT,
            name       TEXT NOT NULL UNIQUE,
            color      TEXT DEFAULT '#3498db',
            created_at TEXT DEFAULT (datetime('now'))
        );
        
        CREATE TABLE IF NOT EXISTS tasks (
            id          INTEGER PRIMARY KEY AUTOINCREMENT,
            project_id  INTEGER REFERENCES projects(id) ON DELETE CASCADE,
            title       TEXT NOT NULL,
            description TEXT,
            status      TEXT DEFAULT 'todo' CHECK(status IN ('todo','doing','done')),
            priority    INTEGER DEFAULT 2 CHECK(priority BETWEEN 1 AND 5),
            due_date    TEXT,
            created_at  TEXT DEFAULT (datetime('now')),
            updated_at  TEXT DEFAULT (datetime('now'))
        );
        
        CREATE TABLE IF NOT EXISTS tags (
            id   INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL UNIQUE
        );
        
        CREATE TABLE IF NOT EXISTS task_tags (
            task_id INTEGER REFERENCES tasks(id) ON DELETE CASCADE,
            tag_id  INTEGER REFERENCES tags(id) ON DELETE CASCADE,
            PRIMARY KEY (task_id, tag_id)
        );
        
        CREATE INDEX IF NOT EXISTS idx_tasks_project  ON tasks(project_id);
        CREATE INDEX IF NOT EXISTS idx_tasks_status   ON tasks(status);
        CREATE INDEX IF NOT EXISTS idx_tasks_priority ON tasks(priority);
    ]])
end

function TaskDB:add_project(name, color)
    local stmt = self.db:prepare(
        "INSERT INTO projects (name, color) VALUES (?, ?)")
    stmt:bind_values(name, color or "#3498db")
    stmt:step()
    stmt:finalize()
    return self.db:last_insert_rowid()
end

function TaskDB:add_task(project_id, title, opts)
    opts = opts or {}
    local stmt = self.db:prepare([[
        INSERT INTO tasks (project_id, title, description, priority, due_date)
        VALUES (?, ?, ?, ?, ?)
    ]])
    stmt:bind_values(project_id, title, opts.description,
        opts.priority or 2, opts.due_date)
    stmt:step()
    stmt:finalize()
    return self.db:last_insert_rowid()
end

function TaskDB:update_status(task_id, status)
    local stmt = self.db:prepare([[
        UPDATE tasks SET status = ?, updated_at = datetime('now')
        WHERE id = ?
    ]])
    stmt:bind_values(status, task_id)
    stmt:step()
    stmt:finalize()
    return self.db:changes() > 0
end

function TaskDB:get_board(project_id)
    local board = {todo = {}, doing = {}, done = {}}
    local stmt = self.db:prepare([[
        SELECT t.*, GROUP_CONCAT(tg.name, ',') as tags
        FROM tasks t
        LEFT JOIN task_tags tt ON t.id = tt.task_id
        LEFT JOIN tags tg ON tt.tag_id = tg.id
        WHERE t.project_id = ?
        GROUP BY t.id
        ORDER BY t.priority DESC, t.created_at
    ]])
    stmt:bind_values(project_id)
    for row in stmt:nrows() do
        table.insert(board[row.status], row)
    end
    stmt:finalize()
    return board
end

function TaskDB:stats(project_id)
    local stmt = self.db:prepare([[
        SELECT 
            status,
            COUNT(*) as count,
            AVG(priority) as avg_priority
        FROM tasks
        WHERE project_id = ?
        GROUP BY status
    ]])
    stmt:bind_values(project_id)
    local stats = {}
    for row in stmt:nrows() do
        stats[row.status] = {count = row.count, avg_priority = row.avg_priority}
    end
    stmt:finalize()
    return stats
end

function TaskDB:close()
    self.db:close()
end

-- ตัวอย่างการใช้งาน
local tdb = TaskDB.new()

local proj_id = tdb:add_project("Website Redesign", "#e74c3c")

tdb:add_task(proj_id, "Design mockups",      {priority = 5, due_date = "2025-02-01"})
tdb:add_task(proj_id, "Frontend development",{priority = 4, due_date = "2025-02-15"})
tdb:add_task(proj_id, "Backend API",         {priority = 4, due_date = "2025-02-20"})
tdb:add_task(proj_id, "Write tests",         {priority = 3})
tdb:add_task(proj_id, "Deploy to staging",   {priority = 3, due_date = "2025-03-01"})
tdb:add_task(proj_id, "UAT",                 {priority = 5, due_date = "2025-03-15"})

-- อัปเดต status
tdb:update_status(1, "done")
tdb:update_status(2, "doing")

-- แสดง board
local board = tdb:get_board(proj_id)
print("=== Kanban Board ===")
for _, status in ipairs({"todo", "doing", "done"}) do
    print(string.format("\n[%s]", status:upper()))
    for _, task in ipairs(board[status]) do
        print(string.format("  [P%d] %s%s",
            task.priority, task.title,
            task.due_date and (" (due: " .. task.due_date .. ")") or ""))
    end
end

-- แสดง stats
local stats = tdb:stats(proj_id)
print("\n=== Stats ===")
for status, s in pairs(stats) do
    print(string.format("  %-8s: %d tasks (avg priority: %.1f)",
        status, s.count, s.avg_priority))
end

tdb:close()
```

---

## 59.16 ตัวอย่างเพิ่มเติม

### ตัวอย่างที่ 27: Batch Operations

```lua
-- batch_operations.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec("CREATE TABLE items (id INTEGER PRIMARY KEY, name TEXT, value REAL)")

-- Batch insert ด้วย transaction (เร็วมาก)
local function batch_insert(database, records)
    local stmt = database:prepare(
        "INSERT INTO items (name, value) VALUES (?, ?)")
    database:exec("BEGIN")
    local count = 0
    for _, rec in ipairs(records) do
        stmt:bind_values(rec.name, rec.value)
        stmt:step()
        stmt:reset()
        count = count + 1
    end
    database:exec("COMMIT")
    stmt:finalize()
    return count
end

-- สร้างข้อมูล
local records = {}
for i = 1, 1000 do
    table.insert(records, {name = "item_" .. i, value = i * 1.5})
end

local t = os.clock()
local n = batch_insert(db, records)
print(string.format("Batch insert %d records: %.4fs", n, os.clock() - t))

-- Batch update
local update_stmt = db:prepare("UPDATE items SET value = value * ? WHERE id = ?")
db:exec("BEGIN")
for i = 1, 100 do
    update_stmt:bind_values(1.1, i)
    update_stmt:step()
    update_stmt:reset()
end
db:exec("COMMIT")
update_stmt:finalize()
print("Batch update done, changes:", db:changes())

-- Batch delete
db:exec("DELETE FROM items WHERE id > 500")
for row in db:nrows("SELECT COUNT(*) as n FROM items") do
    print("Remaining items:", row.n)
end

db:close()
```

### ตัวอย่างที่ 28: Database Triggers

```lua
-- triggers.lua
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE products (
        id       INTEGER PRIMARY KEY AUTOINCREMENT,
        name     TEXT NOT NULL,
        price    REAL NOT NULL,
        stock    INTEGER DEFAULT 0
    );
    
    CREATE TABLE audit_log (
        id         INTEGER PRIMARY KEY AUTOINCREMENT,
        table_name TEXT,
        operation  TEXT,
        record_id  INTEGER,
        old_data   TEXT,
        new_data   TEXT,
        changed_at TEXT DEFAULT (datetime('now'))
    );
    
    -- Trigger: บันทึก INSERT
    CREATE TRIGGER products_after_insert
    AFTER INSERT ON products
    BEGIN
        INSERT INTO audit_log (table_name, operation, record_id, new_data)
        VALUES ('products', 'INSERT', NEW.id,
            json_object('name', NEW.name, 'price', NEW.price, 'stock', NEW.stock));
    END;
    
    -- Trigger: บันทึก UPDATE  
    CREATE TRIGGER products_after_update
    AFTER UPDATE ON products
    BEGIN
        INSERT INTO audit_log (table_name, operation, record_id, old_data, new_data)
        VALUES ('products', 'UPDATE', NEW.id,
            json_object('price', OLD.price, 'stock', OLD.stock),
            json_object('price', NEW.price, 'stock', NEW.stock));
    END;
    
    -- Trigger: ป้องกัน stock ติดลบ
    CREATE TRIGGER check_stock
    BEFORE UPDATE ON products
    WHEN NEW.stock < 0
    BEGIN
        SELECT RAISE(ABORT, 'Stock cannot be negative');
    END;
]])

-- ทดสอบ triggers
db:exec("INSERT INTO products (name, price, stock) VALUES ('Widget', 9.99, 100)")
db:exec("UPDATE products SET price = 12.99, stock = 95 WHERE id = 1")

-- ลอง set stock ติดลบ
local ok, err = pcall(function()
    db:exec("UPDATE products SET stock = -5 WHERE id = 1")
end)
print("Negative stock prevented:", not ok)

-- ดู audit log
print("\nAudit Log:")
for row in db:nrows("SELECT * FROM audit_log ORDER BY id") do
    print(string.format("  [%s] %s on record %d",
        row.operation, row.table_name, row.record_id))
    if row.new_data then print("    New:", row.new_data) end
end

db:close()
```

### ตัวอย่างที่ 29: Window Functions

```lua
-- window_functions.lua
-- SQLite 3.25+ รองรับ Window Functions
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE sales (
        id         INTEGER PRIMARY KEY,
        rep        TEXT,
        region     TEXT,
        amount     REAL,
        sale_date  TEXT
    );
    INSERT INTO sales (rep, region, amount, sale_date) VALUES
        ('Alice', 'North', 1200, '2025-01-05'),
        ('Bob',   'South', 800,  '2025-01-08'),
        ('Alice', 'North', 1500, '2025-01-12'),
        ('Carol', 'East',  950,  '2025-01-15'),
        ('Bob',   'South', 1100, '2025-01-20'),
        ('Alice', 'North', 900,  '2025-01-22'),
        ('Carol', 'East',  1300, '2025-01-28');
]])

-- ROW_NUMBER, RANK
print("=== Rankings ===")
for row in db:nrows([[
    SELECT 
        rep,
        amount,
        ROW_NUMBER() OVER (ORDER BY amount DESC) as row_num,
        RANK()       OVER (ORDER BY amount DESC) as rank,
        DENSE_RANK() OVER (ORDER BY amount DESC) as dense_rank
    FROM sales
    ORDER BY amount DESC
]]) do
    print(string.format("  %-6s $%-8.2f Row:%d Rank:%d Dense:%d",
        row.rep, row.amount, row.row_num, row.rank, row.dense_rank))
end

-- Running total (Cumulative SUM)
print("\n=== Running Total ===")
for row in db:nrows([[
    SELECT
        sale_date,
        rep,
        amount,
        SUM(amount) OVER (ORDER BY sale_date) as running_total
    FROM sales
    ORDER BY sale_date
]]) do
    print(string.format("  %s %-6s $%-8.2f Total: $%.2f",
        row.sale_date, row.rep, row.amount, row.running_total))
end

-- Rank within partition (by region)
print("\n=== Rank per Region ===")
for row in db:nrows([[
    SELECT
        region,
        rep,
        amount,
        RANK() OVER (PARTITION BY region ORDER BY amount DESC) as region_rank
    FROM sales
    ORDER BY region, region_rank
]]) do
    print(string.format("  %-6s %-6s $%-8.2f Rank in region: %d",
        row.region, row.rep, row.amount, row.region_rank))
end

db:close()
```

### ตัวอย่างที่ 30: JSON Operations

```lua
-- sqlite_json.lua
-- SQLite 3.38+ มี built-in JSON functions
local sqlite3 = require("lsqlite3")
local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE configs (
        id      INTEGER PRIMARY KEY,
        app     TEXT NOT NULL,
        settings TEXT NOT NULL  -- JSON column
    );
    INSERT INTO configs VALUES
        (1, 'web', '{"theme":"dark","lang":"en","features":["auth","api","logs"]}'),
        (2, 'mobile', '{"theme":"light","lang":"th","features":["auth","push"]}'),
        (3, 'admin', '{"theme":"system","lang":"en","debug":true,"features":["all"]}');
]])

-- json_extract: ดึงค่าจาก JSON
print("=== Extract JSON values ===")
for row in db:nrows([[
    SELECT 
        app,
        json_extract(settings, '$.theme') as theme,
        json_extract(settings, '$.lang')  as lang,
        json_extract(settings, '$.debug') as debug
    FROM configs
]]) do
    print(string.format("  %-8s theme=%-8s lang=%s debug=%s",
        row.app, row.theme, row.lang, tostring(row.debug)))
end

-- json_array_length
print("\n=== Feature count ===")
for row in db:nrows([[
    SELECT app,
           json_array_length(settings, '$.features') as feature_count
    FROM configs
]]) do
    print(string.format("  %-8s features: %d", row.app, row.feature_count))
end

-- json_each: แตก JSON array เป็น rows
print("\n=== All features (json_each) ===")
for row in db:nrows([[
    SELECT c.app, f.value as feature
    FROM configs c,
         json_each(c.settings, '$.features') f
    ORDER BY c.app, f.value
]]) do
    print(string.format("  %-8s %s", row.app, row.feature))
end

-- json_patch: อัปเดต JSON
db:exec([[
    UPDATE configs
    SET settings = json_patch(settings, '{"theme":"auto"}')
    WHERE app = 'admin'
]])

for row in db:nrows("SELECT settings FROM configs WHERE app='admin'") do
    print("\nUpdated admin settings:", row.settings)
end

db:close()
```

---

## แบบฝึกหัด

**ข้อ 1**: สร้าง database schema สำหรับระบบจัดการห้องสมุด มีตาราง `books`, `members`, `loans` พร้อม foreign keys และ indexes ที่เหมาะสม เขียนฟังก์ชัน `borrow_book` และ `return_book` พร้อม transaction

**ข้อ 2**: เขียน migration system ที่รองรับทั้ง `up` และ `down` (rollback) สำหรับแต่ละ migration ทดสอบการ migrate ขึ้นและลง

**ข้อ 3**: สร้าง query builder class ที่รองรับ method chaining: `db:from("users"):where("age > ?", 18):order("name"):limit(10):select("name, email"):exec()`

**ข้อ 4**: Implement simple caching layer บน SQLite ที่มี TTL (Time-To-Live) สำหรับแต่ละ cache entry และ auto-expire entries ที่หมดอายุ

**ข้อ 5**: เขียน CSV importer ที่อ่าน CSV file และ import เข้า SQLite อัตโนมัติ รองรับ column type detection (INTEGER/REAL/TEXT) และ bulk insert ด้วย transaction

---

> **บทถัดไป**: [บทที่ 60: Message Queue กับ Redis](part-60.md)
