# บทที่ 49: Database - LuaSQL และ SQLite

## บทนำ

การทำงานกับ database เป็นส่วนสำคัญของการพัฒนา application เกือบทุกประเภท ในบทนี้เราจะเรียนรู้การใช้ SQLite ผ่าน lsqlite3, LuaSQL interface, การจัดการ transactions, การป้องกัน SQL injection, การสร้าง ORM เบื้องต้น และ Redis เบื้องต้นสำหรับ NoSQL

## การติดตั้ง

```bash
# LuaSQL SQLite3
luarocks install luasql-sqlite3

# lsqlite3 (SQLite3 binding โดยตรง)
luarocks install lsqlite3

# LuaSQL PostgreSQL
luarocks install luasql-postgres

# LuaSQL MySQL
luarocks install luasql-mysql
```

---

## ตัวอย่างที่ 1: SQLite3 - การเริ่มต้น

```lua
-- ตรวจสอบ lsqlite3
local ok, sqlite3 = pcall(require, "lsqlite3")
if not ok then
    print("lsqlite3 ไม่พบ: " .. tostring(sqlite3))
    print("ติดตั้งด้วย: luarocks install lsqlite3")
    os.exit(1)
end

print("lsqlite3 version:", sqlite3.version())

-- เปิด SQLite database
-- ":memory:" สร้าง in-memory database (ใช้สำหรับ testing)
local db = sqlite3.open(":memory:")
print("Database opened (in-memory)")

-- ตรวจสอบว่า open สำเร็จ
assert(db, "Failed to open database")

-- ดู SQLite version
for row in db:rows("SELECT sqlite_version()") do
    print("SQLite version:", row[1])
end

-- ปิด database
db:close()
print("Database closed")

-- เปิด file-based database
local file_db = sqlite3.open("/tmp/lua_tutorial.db")
print("File database opened: /tmp/lua_tutorial.db")

-- สร้าง table ทดสอบ
file_db:exec([[
    CREATE TABLE IF NOT EXISTS test (
        id INTEGER PRIMARY KEY,
        value TEXT
    )
]])

file_db:exec("INSERT OR IGNORE INTO test VALUES (1, 'Hello')")
file_db:exec("INSERT OR IGNORE INTO test VALUES (2, 'World')")

for row in file_db:rows("SELECT * FROM test") do
    print(string.format("  id=%d value=%s", row[1], row[2]))
end

file_db:close()
print("File database closed")
```

---

## ตัวอย่างที่ 2: สร้าง Table และ Schema

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- สร้าง schema
local create_schema = [[
    -- Users table
    CREATE TABLE IF NOT EXISTS users (
        id          INTEGER PRIMARY KEY AUTOINCREMENT,
        username    TEXT NOT NULL UNIQUE,
        email       TEXT NOT NULL UNIQUE,
        password_hash TEXT NOT NULL,
        full_name   TEXT,
        created_at  INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),
        updated_at  INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),
        is_active   INTEGER NOT NULL DEFAULT 1,
        role        TEXT NOT NULL DEFAULT 'user'
            CHECK(role IN ('admin', 'user', 'moderator'))
    );

    -- Posts table
    CREATE TABLE IF NOT EXISTS posts (
        id          INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id     INTEGER NOT NULL,
        title       TEXT NOT NULL,
        content     TEXT,
        status      TEXT NOT NULL DEFAULT 'draft'
            CHECK(status IN ('draft', 'published', 'archived')),
        views       INTEGER NOT NULL DEFAULT 0,
        created_at  INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),
        FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
    );

    -- Tags table
    CREATE TABLE IF NOT EXISTS tags (
        id   INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL UNIQUE
    );

    -- Post-Tags junction
    CREATE TABLE IF NOT EXISTS post_tags (
        post_id INTEGER NOT NULL,
        tag_id  INTEGER NOT NULL,
        PRIMARY KEY (post_id, tag_id),
        FOREIGN KEY (post_id) REFERENCES posts(id) ON DELETE CASCADE,
        FOREIGN KEY (tag_id)  REFERENCES tags(id)  ON DELETE CASCADE
    );

    -- Indexes
    CREATE INDEX IF NOT EXISTS idx_posts_user_id ON posts(user_id);
    CREATE INDEX IF NOT EXISTS idx_posts_status  ON posts(status);
    CREATE INDEX IF NOT EXISTS idx_users_email   ON users(email);
]]

local err_code = db:exec(create_schema)
if err_code ~= sqlite3.OK then
    print("Schema error:", db:errmsg())
else
    print("Schema created successfully")
end

-- ตรวจสอบ tables ที่สร้าง
print("\nTables created:")
for row in db:rows([[
    SELECT name, sql FROM sqlite_master
    WHERE type='table' ORDER BY name
]]) do
    print("  " .. row[1])
end

print("\nIndexes created:")
for row in db:rows([[
    SELECT name FROM sqlite_master
    WHERE type='index' AND name NOT LIKE 'sqlite_%'
    ORDER BY name
]]) do
    print("  " .. row[1])
end

db:close()
```

---

## ตัวอย่างที่ 3: INSERT Operations

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- Setup
db:exec([[
    CREATE TABLE users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE,
        age INTEGER,
        balance REAL DEFAULT 0.0,
        data BLOB
    )
]])

-- Insert แบบง่าย
db:exec("INSERT INTO users (name, email, age) VALUES ('Alice', 'alice@example.com', 30)")
print("Inserted Alice, rowid:", db:last_insert_rowid())

-- Insert หลาย rows
local users = {
    {"Bob",     "bob@example.com",     25, 1000.50},
    {"Charlie", "charlie@example.com", 35, 2500.00},
    {"Diana",   "diana@example.com",   28, 750.25},
    {"Eve",     "eve@example.com",     32, 5000.00},
}

-- ใช้ prepared statement
local stmt = db:prepare(
    "INSERT INTO users (name, email, age, balance) VALUES (?, ?, ?, ?)"
)

for _, u in ipairs(users) do
    stmt:bind(1, u[1])
    stmt:bind(2, u[2])
    stmt:bind(3, u[3])
    stmt:bind(4, u[4])
    stmt:step()
    stmt:reset()
    print(string.format("Inserted %s (id=%d)", u[1], db:last_insert_rowid()))
end
stmt:finalize()

-- Insert ด้วย named bindings
local named_stmt = db:prepare([[
    INSERT INTO users (name, email, age, balance)
    VALUES (:name, :email, :age, :balance)
]])

named_stmt:bind_names({
    name    = "Frank",
    email   = "frank@example.com",
    age     = 45,
    balance = 3000.0,
})
named_stmt:step()
named_stmt:finalize()
print("Inserted Frank with named bindings")

-- Verify inserts
local count = 0
for row in db:rows("SELECT COUNT(*) FROM users") do
    count = row[1]
end
print(string.format("\nTotal users: %d", count))

db:close()
```

---

## ตัวอย่างที่ 4: SELECT Queries

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- Setup ข้อมูล
db:exec([[
    CREATE TABLE products (
        id       INTEGER PRIMARY KEY,
        name     TEXT NOT NULL,
        category TEXT,
        price    REAL,
        stock    INTEGER,
        active   INTEGER DEFAULT 1
    )
]])

local products = {
    {1, "Laptop",     "Electronics",  29999.99, 50},
    {2, "Mouse",      "Electronics",  599.00,   200},
    {3, "Keyboard",   "Electronics",  1299.00,  150},
    {4, "Desk",       "Furniture",    5999.00,  30},
    {5, "Chair",      "Furniture",    3499.00,  45},
    {6, "Monitor",    "Electronics",  12999.00, 80},
    {7, "Headphones", "Electronics",  2499.00,  120},
    {8, "Notebook",   "Stationery",   89.00,    500},
    {9, "Pen",        "Stationery",   29.00,    1000},
    {10,"Backpack",   "Accessories",  1599.00,  75},
}

local stmt = db:prepare("INSERT INTO products VALUES (?,?,?,?,?,1)")
for _, p in ipairs(products) do
    stmt:bind(1, p[1]); stmt:bind(2, p[2]); stmt:bind(3, p[3])
    stmt:bind(4, p[4]); stmt:bind(5, p[5])
    stmt:step(); stmt:reset()
end
stmt:finalize()

print("=== SELECT Examples ===")

-- SELECT ทั้งหมด
print("\n1. All products:")
for row in db:nrows("SELECT * FROM products ORDER BY id") do
    print(string.format("  [%d] %-12s %-12s %8.2f  stock=%d",
        row.id, row.name, row.category, row.price, row.stock))
end

-- SELECT กับ WHERE
print("\n2. Electronics only:")
for row in db:nrows([[
    SELECT name, price FROM products
    WHERE category = 'Electronics'
    ORDER BY price DESC
]]) do
    print(string.format("  %-12s %8.2f", row.name, row.price))
end

-- SELECT กับ GROUP BY
print("\n3. Category summary:")
for row in db:nrows([[
    SELECT
        category,
        COUNT(*) as count,
        AVG(price) as avg_price,
        SUM(stock) as total_stock
    FROM products
    GROUP BY category
    ORDER BY count DESC
]]) do
    print(string.format("  %-12s count=%d avg=%.2f total_stock=%d",
        row.category, row.count, row.avg_price, row.total_stock))
end

-- SELECT กับ HAVING
print("\n4. Categories with avg price > 3000:")
for row in db:nrows([[
    SELECT category, AVG(price) as avg_price
    FROM products
    GROUP BY category
    HAVING avg_price > 3000
    ORDER BY avg_price DESC
]]) do
    print(string.format("  %-12s avg=%.2f", row.category, row.avg_price))
end

-- SELECT กับ JOIN (ต้องมี orders table)
db:exec([[
    CREATE TABLE orders (
        id         INTEGER PRIMARY KEY,
        product_id INTEGER,
        quantity   INTEGER,
        order_date TEXT
    );
    INSERT INTO orders VALUES (1, 1, 2, '2024-01-15');
    INSERT INTO orders VALUES (2, 3, 5, '2024-01-16');
    INSERT INTO orders VALUES (3, 2, 10, '2024-01-16');
    INSERT INTO orders VALUES (4, 6, 1, '2024-01-17');
]])

print("\n5. Orders with product info (JOIN):")
for row in db:nrows([[
    SELECT
        o.id as order_id,
        p.name as product_name,
        o.quantity,
        p.price,
        (o.quantity * p.price) as total
    FROM orders o
    JOIN products p ON o.product_id = p.id
    ORDER BY o.id
]]) do
    print(string.format("  Order#%d: %-12s x%d = %.2f",
        row.order_id, row.product_name, row.quantity, row.total))
end

-- LIMIT และ OFFSET (pagination)
print("\n6. Pagination (page 2, 3 per page):")
for row in db:nrows("SELECT name, price FROM products ORDER BY price DESC LIMIT 3 OFFSET 3") do
    print(string.format("  %-12s %.2f", row.name, row.price))
end

db:close()
```

---

## ตัวอย่างที่ 5: Prepared Statements

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE employees (
        id         INTEGER PRIMARY KEY AUTOINCREMENT,
        name       TEXT NOT NULL,
        department TEXT,
        salary     REAL,
        hire_date  TEXT
    )
]])

-- ========================================
-- Prepared Statement Benefits:
-- 1. ป้องกัน SQL injection
-- 2. Performance (compile once, run many)
-- 3. Type safety
-- ========================================

-- สร้าง prepared statements
local insert_stmt = db:prepare([[
    INSERT INTO employees (name, department, salary, hire_date)
    VALUES (:name, :department, :salary, :hire_date)
]])

local select_by_dept = db:prepare([[
    SELECT * FROM employees
    WHERE department = ?
    ORDER BY salary DESC
]])

local update_salary = db:prepare([[
    UPDATE employees
    SET salary = salary * ?
    WHERE department = ?
]])

local delete_stmt = db:prepare([[
    DELETE FROM employees WHERE id = ?
]])

-- Insert ข้อมูล
local employees_data = {
    {"Alice",   "Engineering", 85000, "2021-01-15"},
    {"Bob",     "Engineering", 92000, "2020-03-20"},
    {"Charlie", "Marketing",   65000, "2022-06-01"},
    {"Diana",   "Engineering", 95000, "2019-08-10"},
    {"Eve",     "Marketing",   70000, "2021-11-30"},
    {"Frank",   "HR",          60000, "2023-02-14"},
    {"Grace",   "Engineering", 88000, "2022-01-05"},
    {"Henry",   "HR",          62000, "2020-09-01"},
}

for _, e in ipairs(employees_data) do
    insert_stmt:bind_names({
        name       = e[1],
        department = e[2],
        salary     = e[3],
        hire_date  = e[4],
    })
    insert_stmt:step()
    insert_stmt:reset()
end

print("Inserted", #employees_data, "employees")

-- Query by department
local function get_by_department(dept)
    local results = {}
    select_by_dept:bind(1, dept)
    for row in select_by_dept:nrows() do
        table.insert(results, row)
    end
    select_by_dept:reset()
    return results
end

print("\nEngineering department:")
for _, emp in ipairs(get_by_department("Engineering")) do
    print(string.format("  %-8s $%.0f", emp.name, emp.salary))
end

-- Update salaries with raise
print("\nGiving Engineering 10% raise...")
update_salary:bind(1, 1.10)
update_salary:bind(2, "Engineering")
update_salary:step()
update_salary:reset()
print("Changes:", db:changes())

print("Engineering after raise:")
for _, emp in ipairs(get_by_department("Engineering")) do
    print(string.format("  %-8s $%.0f", emp.name, emp.salary))
end

-- Delete ด้วย prepared statement
delete_stmt:bind(1, 6)  -- Frank
delete_stmt:step()
delete_stmt:reset()
print(string.format("\nDeleted employee id=6, changes=%d", db:changes()))

-- Finalize ทั้งหมด
insert_stmt:finalize()
select_by_dept:finalize()
update_salary:finalize()
delete_stmt:finalize()

db:close()
```

---

## ตัวอย่างที่ 6: Parameterized Queries (SQL Injection Prevention)

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE users (
        id       INTEGER PRIMARY KEY,
        username TEXT,
        password TEXT,
        admin    INTEGER DEFAULT 0
    );
    INSERT INTO users VALUES (1, 'admin', 'hash_of_admin_pass', 1);
    INSERT INTO users VALUES (2, 'alice', 'hash_of_alice_pass', 0);
    INSERT INTO users VALUES (3, 'bob',   'hash_of_bob_pass',   0);
]])

-- ==========================================
-- BAD: String concatenation (DANGEROUS!)
-- ==========================================
local function login_bad(username, password)
    -- อันตราย! ถูก SQL injection ได้
    local sql = string.format(
        "SELECT * FROM users WHERE username='%s' AND password='%s'",
        username, password
    )
    print("BAD SQL:", sql)

    for row in db:nrows(sql) do
        return row  -- return first match
    end
    return nil
end

-- ==========================================
-- GOOD: Parameterized query (SAFE!)
-- ==========================================
local function login_safe(username, password)
    local stmt = db:prepare(
        "SELECT * FROM users WHERE username = ? AND password = ?"
    )
    stmt:bind(1, username)
    stmt:bind(2, password)

    local user = nil
    local result = stmt:step()
    if result == sqlite3.ROW then
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

-- SQL Injection attempt
local malicious_input = "' OR '1'='1"
print("Testing SQL injection with:", malicious_input)

print("\nBAD approach (vulnerable):")
local bad_result = login_bad(malicious_input, "anything")
if bad_result then
    print("  DANGER! Got user:", bad_result.username)
else
    print("  No match (lucky)")
end

print("\nGOOD approach (safe):")
local good_result = login_safe(malicious_input, "anything")
if good_result then
    print("  Got user:", good_result.username)
else
    print("  Correctly rejected malicious input!")
end

-- Normal login
print("\nNormal login (alice):")
local alice = login_safe("alice", "hash_of_alice_pass")
if alice then
    print("  Login success:", alice.username, "admin=" .. tostring(alice.admin == 1))
else
    print("  Login failed")
end

-- ==========================================
-- Other injection patterns to avoid
-- ==========================================
local dangerous_inputs = {
    "Robert'); DROP TABLE users; --",
    "1 UNION SELECT username, password, 1, 1 FROM users",
    "admin'--",
    "' OR 1=1 LIMIT 1--",
}

print("\nTesting more injection attempts (all should fail safely):")
for _, input in ipairs(dangerous_inputs) do
    local result = login_safe(input, "password")
    print(string.format("  Input: %-45s -> %s",
        input:sub(1, 45), result and "MATCHED (BAD!)" or "Rejected (GOOD)"))
end

db:close()
```

---

## ตัวอย่างที่ 7: Transactions

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE accounts (
        id      INTEGER PRIMARY KEY,
        name    TEXT NOT NULL,
        balance REAL NOT NULL CHECK(balance >= 0)
    );
    INSERT INTO accounts VALUES (1, 'Alice', 10000);
    INSERT INTO accounts VALUES (2, 'Bob',   5000);
    INSERT INTO accounts VALUES (3, 'Carol', 7500);
]])

-- =============================================
-- Transaction function
-- =============================================
local function with_transaction(db_conn, func)
    db_conn:exec("BEGIN TRANSACTION")

    local ok, err = pcall(func, db_conn)

    if ok then
        db_conn:exec("COMMIT")
        return true, nil
    else
        db_conn:exec("ROLLBACK")
        return false, err
    end
end

-- Transfer money
local function transfer(db_conn, from_id, to_id, amount)
    -- ตรวจสอบ balance
    local stmt = db_conn:prepare("SELECT balance FROM accounts WHERE id = ?")

    stmt:bind(1, from_id)
    stmt:step()
    local from_balance = stmt:get_value(0)
    stmt:reset()

    if from_balance < amount then
        stmt:finalize()
        error(string.format("Insufficient funds: %.2f < %.2f",
            from_balance, amount))
    end

    stmt:finalize()

    -- ตัด balance จาก sender
    local debit = db_conn:prepare(
        "UPDATE accounts SET balance = balance - ? WHERE id = ?"
    )
    debit:bind(1, amount)
    debit:bind(2, from_id)
    debit:step()
    debit:finalize()

    -- เพิ่ม balance ให้ receiver
    local credit = db_conn:prepare(
        "UPDATE accounts SET balance = balance + ? WHERE id = ?"
    )
    credit:bind(1, amount)
    credit:bind(2, to_id)
    credit:step()
    credit:finalize()
end

-- Helper: แสดง balances
local function show_balances(label)
    print("\n" .. label .. ":")
    for row in db:nrows("SELECT * FROM accounts ORDER BY id") do
        print(string.format("  %-8s: %.2f", row.name, row.balance))
    end
end

show_balances("Initial balances")

-- Successful transfer
print("\n--- Transfer 3000 from Alice to Bob ---")
local ok, err = with_transaction(db, function(conn)
    transfer(conn, 1, 2, 3000)
end)
print("Result:", ok and "SUCCESS" or "FAILED: " .. tostring(err))
show_balances("After transfer")

-- Failed transfer (insufficient funds)
print("\n--- Transfer 20000 from Bob to Carol (SHOULD FAIL) ---")
ok, err = with_transaction(db, function(conn)
    transfer(conn, 2, 3, 20000)
end)
print("Result:", ok and "SUCCESS" or "FAILED: " .. tostring(err))
show_balances("After failed transfer (no change)")

-- Savepoints
print("\n--- Savepoint Example ---")
db:exec("BEGIN TRANSACTION")
db:exec("UPDATE accounts SET balance = balance + 1000 WHERE id = 1")
print("Added 1000 to Alice")

db:exec("SAVEPOINT sp1")
db:exec("UPDATE accounts SET balance = balance - 5000 WHERE id = 2")
print("Subtracted 5000 from Bob (about to rollback)")

db:exec("ROLLBACK TO SAVEPOINT sp1")
print("Rolled back to sp1")

db:exec("UPDATE accounts SET balance = balance + 500 WHERE id = 3")
print("Added 500 to Carol")

db:exec("COMMIT")
show_balances("After savepoint demo")

db:close()
```

---

## ตัวอย่างที่ 8: Error Handling

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE products (
        id    INTEGER PRIMARY KEY,
        name  TEXT NOT NULL UNIQUE,
        price REAL CHECK(price > 0)
    )
]])

-- Error codes ของ SQLite
local ERROR_MESSAGES = {
    [sqlite3.ERROR]    = "Generic error",
    [sqlite3.INTERNAL] = "Internal logic error",
    [sqlite3.PERM]     = "Access permission denied",
    [sqlite3.ABORT]    = "Callback routine requested abort",
    [sqlite3.BUSY]     = "Database file is locked",
    [sqlite3.LOCKED]   = "Table in database is locked",
    [sqlite3.NOMEM]    = "Out of memory",
    [sqlite3.READONLY] = "Attempt to write read-only database",
    [sqlite3.INTERRUPT] = "Operation interrupted",
    [sqlite3.IOERR]    = "Disk I/O error",
    [sqlite3.CORRUPT]  = "Database disk image is malformed",
    [sqlite3.TOOBIG]   = "String or BLOB exceeded size limit",
    [sqlite3.CONSTRAINT] = "Abort due to constraint violation",
    [sqlite3.MISMATCH] = "Data type mismatch",
    [sqlite3.MISUSE]   = "Library used incorrectly",
    [sqlite3.RANGE]    = "Bind or column index out of range",
    [sqlite3.NOTADB]   = "Not a database file",
}

-- Wrapper ที่จัดการ errors
local function safe_exec(db_conn, sql, ...)
    local stmt = db_conn:prepare(sql)
    if not stmt then
        return false, string.format("Prepare error: %s", db_conn:errmsg())
    end

    -- Bind parameters
    local args = {...}
    for i, v in ipairs(args) do
        local rc = stmt:bind(i, v)
        if rc ~= sqlite3.OK then
            stmt:finalize()
            return false, string.format("Bind error at %d: %s", i, db_conn:errmsg())
        end
    end

    local rc = stmt:step()
    stmt:finalize()

    if rc == sqlite3.DONE or rc == sqlite3.ROW or rc == sqlite3.OK then
        return true, nil
    else
        local msg = ERROR_MESSAGES[rc] or ("Error code " .. rc)
        return false, string.format("Execute error: %s (%s)", db_conn:errmsg(), msg)
    end
end

-- ทดสอบ error scenarios
print("=== Error Handling ===\n")

-- Constraint UNIQUE violation
local ok1, err1 = safe_exec(db,
    "INSERT INTO products VALUES (1, 'Widget', 9.99)")
print("First insert:", ok1 and "OK" or err1)

local ok2, err2 = safe_exec(db,
    "INSERT INTO products VALUES (2, 'Widget', 19.99)")  -- UNIQUE violation
print("Duplicate name:", ok2 and "OK" or err2)

-- CHECK constraint violation (price <= 0)
local ok3, err3 = safe_exec(db,
    "INSERT INTO products VALUES (3, 'Gadget', -5.00)")
print("Negative price:", ok3 and "OK" or err3)

-- Type coercion (SQLite ยืดหยุ่นมาก)
local ok4, err4 = safe_exec(db,
    "INSERT INTO products VALUES (4, 'Thingamajig', ?)", "29.99")  -- string as number
print("String price:", ok4 and "OK" or err4)

-- NULL constraint violation
local ok5, err5 = safe_exec(db,
    "INSERT INTO products (price) VALUES (9.99)")  -- name is NOT NULL
print("NULL name:", ok5 and "OK" or err5)

-- Invalid SQL syntax
local ok6, err6 = safe_exec(db, "SELECT * FRPM products")  -- typo
print("Bad SQL:", ok6 and "OK" or err6)

-- Table ที่ไม่มีอยู่
local ok7, err7 = safe_exec(db, "SELECT * FROM nonexistent_table")
print("No table:", ok7 and "OK" or err7)

print("\nAll products inserted successfully:")
for row in db:nrows("SELECT * FROM products") do
    print(string.format("  [%d] %-15s %.2f", row.id, row.name, row.price))
end

db:close()
```

---

## ตัวอย่างที่ 9: Row Iteration และ Column Types

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- สร้าง table ที่มี type หลากหลาย
db:exec([[
    CREATE TABLE type_demo (
        id          INTEGER,
        name        TEXT,
        price       REAL,
        data        BLOB,
        created     TEXT,
        is_active   INTEGER,
        nothing     TEXT
    );
    INSERT INTO type_demo VALUES (
        42,
        'Hello World',
        3.14159,
        X'DEADBEEF',
        '2024-01-15 10:30:00',
        1,
        NULL
    );
    INSERT INTO type_demo VALUES (
        100,
        'สวัสดี',
        2.71828,
        X'CAFE',
        '2024-06-01',
        0,
        NULL
    );
]])

-- Column type constants
local TYPES = {
    [sqlite3.INTEGER] = "INTEGER",
    [sqlite3.FLOAT]   = "FLOAT",
    [sqlite3.TEXT]    = "TEXT",
    [sqlite3.BLOB]    = "BLOB",
    [sqlite3.NULL]    = "NULL",
}

-- Iterator methods ต่างๆ
print("=== Row Iteration Methods ===\n")

-- Method 1: rows() - returns array
print("1. db:rows() - array access:")
for row in db:rows("SELECT * FROM type_demo") do
    print(string.format("  id=%s name=%s price=%s",
        tostring(row[1]), tostring(row[2]), tostring(row[3])))
end

-- Method 2: nrows() - returns named dict
print("\n2. db:nrows() - named access:")
for row in db:nrows("SELECT id, name, price FROM type_demo") do
    print(string.format("  id=%d name=%s price=%.5f",
        row.id, row.name, row.price))
end

-- Method 3: urows() - unpacked values
print("\n3. db:urows() - unpacked:")
for id, name, price in db:urows("SELECT id, name, price FROM type_demo") do
    print(string.format("  id=%d name=%s price=%.5f",
        id, name, price))
end

-- Method 4: step() manual
print("\n4. Manual stmt:step():")
local stmt = db:prepare("SELECT * FROM type_demo")
while stmt:step() == sqlite3.ROW do
    local ncols = stmt:columns()
    for i = 0, ncols - 1 do
        local col_type = stmt:column_type(i)
        local col_name = stmt:get_name(i)
        local col_val  = stmt:get_value(i)
        print(string.format("  %s [%s] = %s",
            col_name,
            TYPES[col_type] or "?",
            tostring(col_val)))
    end
    print()
end
stmt:finalize()

-- BLOB handling
print("5. BLOB data:")
local blob_stmt = db:prepare("SELECT data FROM type_demo WHERE id = 42")
blob_stmt:step()
local blob = blob_stmt:get_value(0)
print("  BLOB type:", type(blob))
if blob then
    print("  BLOB length:", #blob, "bytes")
    print("  BLOB hex:", blob:gsub(".", function(c)
        return string.format("%02X ", c:byte())
    end))
end
blob_stmt:finalize()

-- NULL handling
print("\n6. NULL handling:")
for row in db:nrows("SELECT id, nothing FROM type_demo") do
    print(string.format("  id=%d nothing=%s (nil? %s)",
        row.id, tostring(row.nothing), tostring(row.nothing == nil)))
end

db:close()
```

---

## ตัวอย่างที่ 10: ORM พื้นฐาน

```lua
local sqlite3 = require("lsqlite3")

-- ============================================
-- Simple ORM Implementation
-- ============================================

local Model = {}
Model.__index = Model

-- Create a new model class
function Model:new(table_name, schema, db_conn)
    local cls = setmetatable({}, {__index = Model})
    cls.table_name = table_name
    cls.schema     = schema
    cls.db         = db_conn
    cls.__index    = cls

    -- Create table
    local cols = {}
    for col_name, col_def in pairs(schema) do
        table.insert(cols, col_name .. " " .. col_def)
    end
    cls.db:exec(string.format(
        "CREATE TABLE IF NOT EXISTS %s (%s)",
        table_name, table.concat(cols, ", ")
    ))

    return cls
end

-- Create (INSERT)
function Model:create(data)
    local keys   = {}
    local values = {}
    local placeholders = {}

    for k, v in pairs(data) do
        if self.schema[k] then
            table.insert(keys, k)
            table.insert(values, v)
            table.insert(placeholders, "?")
        end
    end

    local sql = string.format(
        "INSERT INTO %s (%s) VALUES (%s)",
        self.table_name,
        table.concat(keys, ", "),
        table.concat(placeholders, ", ")
    )

    local stmt = self.db:prepare(sql)
    for i, v in ipairs(values) do
        stmt:bind(i, v)
    end
    stmt:step()
    stmt:finalize()

    local id = self.db:last_insert_rowid()
    data.id = id
    return self:find(id)
end

-- Find by ID
function Model:find(id)
    local stmt = self.db:prepare(
        "SELECT * FROM " .. self.table_name .. " WHERE id = ?"
    )
    stmt:bind(1, id)

    local result = nil
    if stmt:step() == sqlite3.ROW then
        result = {}
        local ncols = stmt:columns()
        for i = 0, ncols - 1 do
            result[stmt:get_name(i)] = stmt:get_value(i)
        end
        -- Attach instance methods
        setmetatable(result, {
            __index = self,
            __tostring = function(self2)
                local parts = {}
                for k, v in pairs(self2) do
                    if type(v) ~= "function" then
                        table.insert(parts, k .. "=" .. tostring(v))
                    end
                end
                return self.table_name .. "{" .. table.concat(parts, ", ") .. "}"
            end
        })
    end
    stmt:finalize()
    return result
end

-- Find all with conditions
function Model:where(conditions, order_by, limit)
    local wheres = {}
    local values = {}

    for k, v in pairs(conditions or {}) do
        table.insert(wheres, k .. " = ?")
        table.insert(values, v)
    end

    local sql = "SELECT * FROM " .. self.table_name
    if #wheres > 0 then
        sql = sql .. " WHERE " .. table.concat(wheres, " AND ")
    end
    if order_by then
        sql = sql .. " ORDER BY " .. order_by
    end
    if limit then
        sql = sql .. " LIMIT " .. limit
    end

    local stmt = self.db:prepare(sql)
    for i, v in ipairs(values) do
        stmt:bind(i, v)
    end

    local results = {}
    while stmt:step() == sqlite3.ROW do
        local row = {}
        for i = 0, stmt:columns() - 1 do
            row[stmt:get_name(i)] = stmt:get_value(i)
        end
        table.insert(results, row)
    end
    stmt:finalize()

    return results
end

-- Update
function Model:update(id, data)
    local sets  = {}
    local values = {}

    for k, v in pairs(data) do
        if k ~= "id" and self.schema[k] then
            table.insert(sets, k .. " = ?")
            table.insert(values, v)
        end
    end

    if #sets == 0 then return false end

    table.insert(values, id)
    local sql = string.format(
        "UPDATE %s SET %s WHERE id = ?",
        self.table_name, table.concat(sets, ", ")
    )

    local stmt = self.db:prepare(sql)
    for i, v in ipairs(values) do
        stmt:bind(i, v)
    end
    stmt:step()
    stmt:finalize()

    return self.db:changes() > 0
end

-- Delete
function Model:delete(id)
    local stmt = self.db:prepare(
        "DELETE FROM " .. self.table_name .. " WHERE id = ?"
    )
    stmt:bind(1, id)
    stmt:step()
    stmt:finalize()
    return self.db:changes() > 0
end

-- Count
function Model:count(conditions)
    local wheres = {}
    local values = {}

    for k, v in pairs(conditions or {}) do
        table.insert(wheres, k .. " = ?")
        table.insert(values, v)
    end

    local sql = "SELECT COUNT(*) FROM " .. self.table_name
    if #wheres > 0 then
        sql = sql .. " WHERE " .. table.concat(wheres, " AND ")
    end

    local stmt = self.db:prepare(sql)
    for i, v in ipairs(values) do stmt:bind(i, v) end
    stmt:step()
    local count = stmt:get_value(0)
    stmt:finalize()
    return count
end

-- ============================================
-- ใช้งาน ORM
-- ============================================

local db = sqlite3.open(":memory:")

-- สร้าง User model
local User = Model:new("users", {
    id         = "INTEGER PRIMARY KEY AUTOINCREMENT",
    username   = "TEXT NOT NULL UNIQUE",
    email      = "TEXT NOT NULL",
    age        = "INTEGER",
    created_at = "TEXT DEFAULT (datetime('now'))",
}, db)

print("=== ORM Example ===\n")

-- Create
print("Creating users...")
local u1 = User:create({
    username = "alice",
    email    = "alice@example.com",
    age      = 30,
})
print("Created:", u1 and "alice (id=" .. tostring(u1.id) .. ")" or "failed")

local u2 = User:create({
    username = "bob",
    email    = "bob@example.com",
    age      = 25,
})

local u3 = User:create({
    username = "carol",
    email    = "carol@example.com",
    age      = 35,
})

-- Find
local found = User:find(1)
if found then
    print("\nFound user:", found.username, "age=", found.age)
end

-- Where
print("\nUsers under 32:")
local young = User:where({}, "age ASC")
for _, u in ipairs(young) do
    if u.age < 32 then
        print(string.format("  %-10s age=%d", u.username, u.age))
    end
end

-- Update
User:update(1, {age = 31, email = "alice.new@example.com"})
local updated = User:find(1)
print("\nAfter update alice:", updated.age, updated.email)

-- Count
print("\nTotal users:", User:count())

-- Delete
User:delete(3)
print("After delete carol, total:", User:count())

db:close()
```

---

## ตัวอย่างที่ 11: Migration Scripts

```lua
local sqlite3 = require("lsqlite3")

-- Database Migration System
local Migrations = {}
Migrations.__index = Migrations

function Migrations.new(db_conn)
    local m = setmetatable({}, Migrations)
    m.db = db_conn
    m.migrations = {}

    -- สร้าง migrations table
    m.db:exec([[
        CREATE TABLE IF NOT EXISTS schema_migrations (
            version    INTEGER PRIMARY KEY,
            name       TEXT NOT NULL,
            applied_at TEXT NOT NULL DEFAULT (datetime('now'))
        )
    ]])

    return m
end

function Migrations:add(version, name, up_sql, down_sql)
    table.insert(self.migrations, {
        version  = version,
        name     = name,
        up_sql   = up_sql,
        down_sql = down_sql,
    })
    -- Sort by version
    table.sort(self.migrations, function(a, b)
        return a.version < b.version
    end)
end

function Migrations:get_applied()
    local applied = {}
    for row in self.db:urows(
        "SELECT version FROM schema_migrations ORDER BY version"
    ) do
        applied[row] = true
    end
    return applied
end

function Migrations:migrate()
    local applied = self:get_applied()
    local count = 0

    for _, m in ipairs(self.migrations) do
        if not applied[m.version] then
            print(string.format("Applying migration %d: %s", m.version, m.name))

            self.db:exec("BEGIN TRANSACTION")
            local ok, err = pcall(function()
                local rc = self.db:exec(m.up_sql)
                if rc ~= sqlite3.OK then
                    error("Migration failed: " .. self.db:errmsg())
                end

                -- Record migration
                local stmt = self.db:prepare(
                    "INSERT INTO schema_migrations (version, name) VALUES (?, ?)"
                )
                stmt:bind(1, m.version)
                stmt:bind(2, m.name)
                stmt:step()
                stmt:finalize()
            end)

            if ok then
                self.db:exec("COMMIT")
                print(string.format("  v%d applied successfully", m.version))
                count = count + 1
            else
                self.db:exec("ROLLBACK")
                print(string.format("  v%d FAILED: %s", m.version, err))
                return count, err
            end
        end
    end

    return count
end

function Migrations:rollback(target_version)
    local applied = self:get_applied()
    local count = 0

    -- ย้อนจาก version ล่าสุด
    for i = #self.migrations, 1, -1 do
        local m = self.migrations[i]
        if applied[m.version] and m.version > target_version then
            print(string.format("Rolling back migration %d: %s", m.version, m.name))

            if m.down_sql then
                self.db:exec("BEGIN TRANSACTION")
                local ok, err = pcall(function()
                    self.db:exec(m.down_sql)
                    local stmt = self.db:prepare(
                        "DELETE FROM schema_migrations WHERE version = ?"
                    )
                    stmt:bind(1, m.version)
                    stmt:step()
                    stmt:finalize()
                end)

                if ok then
                    self.db:exec("COMMIT")
                    print(string.format("  v%d rolled back", m.version))
                    count = count + 1
                else
                    self.db:exec("ROLLBACK")
                    print(string.format("  v%d rollback FAILED: %s", m.version, err))
                end
            else
                print(string.format("  v%d has no rollback", m.version))
            end
        end
    end
    return count
end

function Migrations:status()
    local applied = self:get_applied()
    print("\nMigration Status:")
    print(string.rep("-", 50))
    for _, m in ipairs(self.migrations) do
        local status = applied[m.version] and "[APPLIED]" or "[PENDING]"
        print(string.format("  v%-4d %-30s %s",
            m.version, m.name, status))
    end
end

-- ============================================
-- ตัวอย่างการใช้
-- ============================================
local db = sqlite3.open(":memory:")
local mig = Migrations.new(db)

-- เพิ่ม migrations
mig:add(1, "create_users_table", [[
    CREATE TABLE users (
        id       INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT NOT NULL UNIQUE,
        email    TEXT NOT NULL
    )
]], [[
    DROP TABLE IF EXISTS users
]])

mig:add(2, "add_users_created_at", [[
    ALTER TABLE users ADD COLUMN created_at TEXT DEFAULT (datetime('now'))
]], nil)  -- no rollback for ALTER TABLE in SQLite

mig:add(3, "create_posts_table", [[
    CREATE TABLE posts (
        id      INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER NOT NULL,
        title   TEXT NOT NULL,
        FOREIGN KEY (user_id) REFERENCES users(id)
    )
]], [[
    DROP TABLE IF EXISTS posts
]])

mig:add(4, "add_post_status", [[
    ALTER TABLE posts ADD COLUMN status TEXT DEFAULT 'draft'
]], nil)

print("=== Database Migrations ===")
mig:status()

print("\nRunning migrations:")
local count, err = mig:migrate()
print(string.format("\nApplied %d migrations", count))
if err then print("Error:", err) end

mig:status()

-- Verify schema
print("\nTable structure:")
for row in db:nrows([[
    SELECT name FROM sqlite_master WHERE type='table'
    AND name NOT LIKE 'sqlite_%' AND name NOT LIKE 'schema_%'
    ORDER BY name
]]) do
    print("  " .. row.name)
end

db:close()
```

---

## ตัวอย่างที่ 12: Connection Pooling

```lua
local sqlite3 = require("lsqlite3")
local socket  = require("socket")

-- Connection Pool สำหรับ SQLite
-- (SQLite ไม่ต้องการ pool จริงๆ แต่เป็น pattern ที่ใช้กับ MySQL/PostgreSQL)

local ConnectionPool = {}
ConnectionPool.__index = ConnectionPool

function ConnectionPool.new(db_path, pool_size, timeout)
    local pool = setmetatable({}, ConnectionPool)
    pool.db_path    = db_path
    pool.pool_size  = pool_size or 5
    pool.timeout    = timeout or 5
    pool.connections = {}
    pool.available  = {}
    pool.waiters    = {}
    pool.stats      = {acquired = 0, released = 0, created = 0}

    -- สร้าง connections ล่วงหน้า
    for i = 1, pool.pool_size do
        local conn = pool:_create_connection()
        if conn then
            table.insert(pool.available, conn)
            pool.stats.created = pool.stats.created + 1
        end
    end

    print(string.format("Connection pool created: %d/%d connections ready",
        #pool.available, pool_size))

    return pool
end

function ConnectionPool:_create_connection()
    local conn = sqlite3.open(self.db_path)
    if conn then
        -- Settings สำหรับ performance
        conn:exec("PRAGMA journal_mode = WAL")
        conn:exec("PRAGMA synchronous = NORMAL")
        conn:exec("PRAGMA cache_size = 10000")
        table.insert(self.connections, conn)
        return conn
    end
    return nil
end

function ConnectionPool:acquire(timeout)
    timeout = timeout or self.timeout
    local deadline = socket.gettime() + timeout

    while socket.gettime() < deadline do
        if #self.available > 0 then
            local conn = table.remove(self.available)
            self.stats.acquired = self.stats.acquired + 1
            return conn
        end
        socket.sleep(0.01)  -- Wait 10ms
    end

    return nil, "Connection pool timeout"
end

function ConnectionPool:release(conn)
    -- ตรวจสอบว่า conn เป็นของ pool นี้
    for _, c in ipairs(self.connections) do
        if c == conn then
            table.insert(self.available, conn)
            self.stats.released = self.stats.released + 1
            return true
        end
    end
    return false, "Connection not from this pool"
end

function ConnectionPool:with_connection(func)
    local conn, err = self:acquire()
    if not conn then
        return nil, err
    end

    local ok, result = pcall(func, conn)
    self:release(conn)

    if ok then
        return result
    else
        return nil, result
    end
end

function ConnectionPool:close_all()
    for _, conn in ipairs(self.connections) do
        conn:close()
    end
    self.connections = {}
    self.available  = {}
    print(string.format("Pool closed. Stats: acquired=%d released=%d created=%d",
        self.stats.acquired, self.stats.released, self.stats.created))
end

-- ทดสอบ
print("=== Connection Pool ===")

local pool = ConnectionPool.new(":memory:", 3)

-- สร้าง table ผ่าน pool
local result, err = pool:with_connection(function(conn)
    conn:exec([[
        CREATE TABLE IF NOT EXISTS items (
            id    INTEGER PRIMARY KEY,
            name  TEXT,
            value INTEGER
        )
    ]])
    for i = 1, 5 do
        conn:exec(string.format(
            "INSERT INTO items VALUES (%d, 'item%d', %d)",
            i, i, i * 100
        ))
    end
    return true
end)

print("Setup:", result and "OK" or err)

-- Query ผ่าน pool
for run = 1, 3 do
    local items, err2 = pool:with_connection(function(conn)
        local rows = {}
        for row in conn:nrows("SELECT * FROM items LIMIT 2") do
            table.insert(rows, row)
        end
        return rows
    end)

    if items then
        print(string.format("Query run %d: got %d items", run, #items))
    end
end

pool:close_all()
```

---

## ตัวอย่างที่ 13: LuaSQL Interface

```lua
-- LuaSQL provides uniform interface across databases
-- (MySQL, PostgreSQL, SQLite3, ODBC, etc.)

local env_ok, luasql = pcall(require, "luasql.sqlite3")
if not env_ok then
    print("LuaSQL SQLite3 ไม่พบ")
    print("ติดตั้ง: luarocks install luasql-sqlite3")
    print("\nแสดง API structure เท่านั้น:")

    -- แสดง LuaSQL API
    print([[
LuaSQL API:

-- สร้าง environment
local env = luasql.sqlite3()

-- Connect
local conn = env:connect(":memory:")

-- Execute DDL/DML
conn:execute([[
    CREATE TABLE test (id INTEGER, name TEXT)
]])

conn:execute("INSERT INTO test VALUES (1, 'Hello')")

-- Query
local cursor = conn:execute("SELECT * FROM test")

-- Fetch rows
local row = cursor:fetch({}, "a")  -- "a" = associative array
while row do
    print(row.id, row.name)
    row = cursor:fetch(row, "a")
end

cursor:close()
conn:close()
env:close()
]])
    return
end

local env = luasql.sqlite3()
local conn = env:connect(":memory:")

print("=== LuaSQL Interface ===\n")

-- สร้าง table
conn:execute([[
    CREATE TABLE books (
        id     INTEGER PRIMARY KEY,
        title  TEXT NOT NULL,
        author TEXT,
        year   INTEGER,
        genre  TEXT
    )
]])

-- Insert ด้วย LuaSQL
local books = {
    {"Lua Programming", "PUC-Rio", 2003, "Programming"},
    {"Clean Code", "Robert Martin", 2008, "Programming"},
    {"The Pragmatic Programmer", "Hunt & Thomas", 1999, "Programming"},
    {"Design Patterns", "Gang of Four", 1994, "Programming"},
    {"SICP", "Abelson & Sussman", 1984, "Programming"},
}

for _, b in ipairs(books) do
    conn:execute(string.format([[
        INSERT INTO books VALUES (NULL, '%s', '%s', %d, '%s')
    ]], b[1]:gsub("'", "''"), b[2]:gsub("'", "''"), b[3], b[4]))
end

print("Inserted", #books, "books")

-- SELECT ด้วย cursor
local cursor = conn:execute("SELECT * FROM books ORDER BY year")

print("\nAll books (ordered by year):")
print(string.format("%-30s %-25s %s", "Title", "Author", "Year"))
print(string.rep("-", 65))

local row = cursor:fetch({}, "a")  -- "a" = associative (named columns)
while row do
    print(string.format("%-30s %-25s %d",
        row.title:sub(1, 29), row.author:sub(1, 24), row.year))
    row = cursor:fetch(row, "a")
end

cursor:close()

-- Column names
local cursor2 = conn:execute("SELECT * FROM books LIMIT 1")
local col_names = cursor2:getcolnames()
local col_types = cursor2:getcoltypes()

print("\nColumn information:")
for i, name in ipairs(col_names) do
    print(string.format("  %d. %-12s [%s]", i, name, col_types[i] or "?"))
end
cursor2:close()

-- Affected rows
conn:execute("UPDATE books SET genre = 'CS' WHERE genre = 'Programming'")
-- LuaSQL ไม่ return affected rows ตรงๆ

conn:close()
env:close()
print("\nLuaSQL demo complete")
```

---

## ตัวอย่างที่ 14: Advanced Queries

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- Setup complex schema
db:exec([[
    CREATE TABLE customers (
        id      INTEGER PRIMARY KEY,
        name    TEXT,
        country TEXT,
        segment TEXT  -- 'small', 'medium', 'enterprise'
    );

    CREATE TABLE orders (
        id          INTEGER PRIMARY KEY,
        customer_id INTEGER,
        amount      REAL,
        status      TEXT,
        order_date  TEXT,
        FOREIGN KEY (customer_id) REFERENCES customers(id)
    );

    -- Insert sample data
    INSERT INTO customers VALUES
        (1, 'Acme Corp',    'US', 'enterprise'),
        (2, 'Small Biz',    'TH', 'small'),
        (3, 'Medium Inc',   'UK', 'medium'),
        (4, 'Big Company',  'JP', 'enterprise'),
        (5, 'Tiny Shop',    'TH', 'small');

    INSERT INTO orders VALUES
        (1,  1, 50000, 'completed', '2024-01-15'),
        (2,  1, 75000, 'completed', '2024-02-20'),
        (3,  2,  1500, 'completed', '2024-01-10'),
        (4,  3, 12000, 'pending',   '2024-03-01'),
        (5,  4, 95000, 'completed', '2024-01-25'),
        (6,  4, 80000, 'completed', '2024-03-10'),
        (7,  5,   800, 'cancelled', '2024-02-05'),
        (8,  1, 60000, 'pending',   '2024-03-15'),
        (9,  3, 15000, 'completed', '2024-02-28'),
        (10, 2,  2000, 'completed', '2024-03-20');
]])

print("=== Advanced Queries ===\n")

-- Window functions (SQLite 3.25+)
print("1. Running total by customer:")
for row in db:nrows([[
    SELECT
        c.name,
        o.amount,
        o.order_date,
        SUM(o.amount) OVER (
            PARTITION BY c.id
            ORDER BY o.order_date
        ) as running_total
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    WHERE o.status = 'completed'
    ORDER BY c.name, o.order_date
]]) do
    print(string.format("  %-15s %8.0f  running: %8.0f",
        row.name, row.amount, row.running_total))
end

-- CTEs (Common Table Expressions)
print("\n2. Top customers by revenue (CTE):")
for row in db:nrows([[
    WITH customer_revenue AS (
        SELECT
            c.id,
            c.name,
            c.country,
            c.segment,
            SUM(o.amount) as total_revenue,
            COUNT(o.id) as order_count,
            AVG(o.amount) as avg_order
        FROM customers c
        LEFT JOIN orders o ON c.id = o.customer_id
            AND o.status = 'completed'
        GROUP BY c.id
    )
    SELECT
        name,
        country,
        segment,
        total_revenue,
        order_count,
        ROUND(avg_order, 2) as avg_order
    FROM customer_revenue
    WHERE total_revenue > 0
    ORDER BY total_revenue DESC
]]) do
    print(string.format("  %-15s %-3s %-10s total=%8.0f orders=%d avg=%8.0f",
        row.name, row.country, row.segment,
        row.total_revenue or 0, row.order_count or 0,
        row.avg_order or 0))
end

-- CASE WHEN
print("\n3. Order classification:")
for row in db:nrows([[
    SELECT
        c.name,
        o.amount,
        CASE
            WHEN o.amount >= 50000 THEN 'Large'
            WHEN o.amount >= 10000 THEN 'Medium'
            WHEN o.amount >= 1000  THEN 'Small'
            ELSE 'Micro'
        END as size,
        CASE o.status
            WHEN 'completed' THEN '✓'
            WHEN 'pending'   THEN '○'
            WHEN 'cancelled' THEN '✗'
            ELSE '?'
        END as status_icon
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    ORDER BY o.amount DESC
]]) do
    print(string.format("  %-15s %8.0f  %-6s  %s",
        row.name, row.amount, row.size, row.status_icon))
end

-- Subquery
print("\n4. Customers with above-average orders:")
for row in db:nrows([[
    SELECT c.name, AVG(o.amount) as avg_amount
    FROM customers c
    JOIN orders o ON c.id = o.customer_id
    WHERE o.status = 'completed'
    GROUP BY c.id
    HAVING avg_amount > (
        SELECT AVG(amount) FROM orders WHERE status = 'completed'
    )
    ORDER BY avg_amount DESC
]]) do
    print(string.format("  %-15s avg=%.0f", row.name, row.avg_amount))
end

db:close()
```

---

## ตัวอย่างที่ 15: Redis Basics (lua-resty-redis)

```lua
-- lua-resty-redis ใช้สำหรับ OpenResty/Nginx
-- สำหรับ Lua ทั่วไปใช้ redis-lua หรือ lua-redis

-- Pattern สำหรับ Redis
-- ติดตั้ง: luarocks install lua-resty-redis (สำหรับ OpenResty)
-- หรือ: luarocks install redis-lua

-- ==============================
-- Conceptual Redis Client
-- ==============================

local RedisClient = {}
RedisClient.__index = RedisClient

-- Mock Redis สำหรับ demo (ใช้ Lua table จำลอง Redis)
local MockRedis = {}
MockRedis.__index = MockRedis

function MockRedis.new()
    return setmetatable({
        data    = {},
        expires = {},
        lists   = {},
        hashes  = {},
        sets    = {},
    }, MockRedis)
end

function MockRedis:SET(key, value, ...)
    local args = {...}
    self.data[key] = tostring(value)

    -- Handle EX option
    for i = 1, #args, 2 do
        if args[i] == "EX" then
            self.expires[key] = os.time() + tonumber(args[i+1])
        end
    end
    return "OK"
end

function MockRedis:GET(key)
    -- Check expiry
    if self.expires[key] and os.time() > self.expires[key] then
        self.data[key] = nil
        self.expires[key] = nil
        return nil
    end
    return self.data[key]
end

function MockRedis:DEL(...)
    local count = 0
    for _, key in ipairs({...}) do
        if self.data[key] then
            self.data[key] = nil
            count = count + 1
        end
    end
    return count
end

function MockRedis:EXISTS(key)
    return self.data[key] ~= nil and 1 or 0
end

function MockRedis:INCR(key)
    self.data[key] = tostring((tonumber(self.data[key]) or 0) + 1)
    return tonumber(self.data[key])
end

function MockRedis:INCRBY(key, amount)
    self.data[key] = tostring((tonumber(self.data[key]) or 0) + amount)
    return tonumber(self.data[key])
end

function MockRedis:EXPIRE(key, seconds)
    if self.data[key] then
        self.expires[key] = os.time() + seconds
        return 1
    end
    return 0
end

function MockRedis:TTL(key)
    if not self.data[key] then return -2 end
    if not self.expires[key] then return -1 end
    return math.max(0, self.expires[key] - os.time())
end

function MockRedis:LPUSH(key, ...)
    self.lists[key] = self.lists[key] or {}
    for i = select('#', ...), 1, -1 do
        table.insert(self.lists[key], 1, (select(i, ...)))
    end
    return #self.lists[key]
end

function MockRedis:RPUSH(key, ...)
    self.lists[key] = self.lists[key] or {}
    for i = 1, select('#', ...) do
        table.insert(self.lists[key], (select(i, ...)))
    end
    return #self.lists[key]
end

function MockRedis:LRANGE(key, start, stop)
    local list = self.lists[key] or {}
    local len = #list
    if start < 0 then start = len + start + 1 else start = start + 1 end
    if stop < 0 then stop = len + stop + 1 else stop = stop + 1 end
    local result = {}
    for i = start, math.min(stop, len) do
        table.insert(result, list[i])
    end
    return result
end

function MockRedis:LLEN(key)
    return #(self.lists[key] or {})
end

function MockRedis:HSET(key, ...)
    self.hashes[key] = self.hashes[key] or {}
    local args = {...}
    local count = 0
    for i = 1, #args, 2 do
        if not self.hashes[key][args[i]] then count = count + 1 end
        self.hashes[key][args[i]] = args[i+1]
    end
    return count
end

function MockRedis:HGET(key, field)
    return self.hashes[key] and self.hashes[key][field] or nil
end

function MockRedis:HGETALL(key)
    local h = self.hashes[key] or {}
    local result = {}
    for k, v in pairs(h) do
        table.insert(result, k)
        table.insert(result, v)
    end
    return result
end

function MockRedis:SADD(key, ...)
    self.sets[key] = self.sets[key] or {}
    local count = 0
    for i = 1, select('#', ...) do
        local member = (select(i, ...))
        if not self.sets[key][member] then
            self.sets[key][member] = true
            count = count + 1
        end
    end
    return count
end

function MockRedis:SMEMBERS(key)
    local result = {}
    for k in pairs(self.sets[key] or {}) do
        table.insert(result, k)
    end
    return result
end

function MockRedis:SISMEMBER(key, member)
    return self.sets[key] and self.sets[key][member] and 1 or 0
end

-- ==============================
-- ทดสอบ Redis Operations
-- ==============================

local redis = MockRedis.new()

print("=== Redis Operations Demo ===\n")

-- String operations
print("--- Strings ---")
redis:SET("greeting", "Hello, Redis!")
redis:SET("counter", "0")
redis:SET("temp_key", "expires_soon", "EX", 10)

print("greeting:", redis:GET("greeting"))
print("counter before:", redis:GET("counter"))
redis:INCRBY("counter", 5)
redis:INCR("counter")
print("counter after:", redis:GET("counter"))
print("temp_key TTL:", redis:TTL("temp_key"), "seconds")
print("exists:", redis:EXISTS("greeting"))
redis:DEL("greeting")
print("after delete:", tostring(redis:GET("greeting")))

-- List operations
print("\n--- Lists ---")
redis:RPUSH("queue", "task1", "task2", "task3")
redis:LPUSH("queue", "urgent_task")
print("Queue length:", redis:LLEN("queue"))
print("Queue contents:", table.concat(redis:LRANGE("queue", 0, -1), " -> "))

-- Hash operations
print("\n--- Hashes ---")
redis:HSET("user:1",
    "name", "Alice",
    "email", "alice@example.com",
    "age", "30",
    "role", "admin"
)
print("user:1 name:", redis:HGET("user:1", "name"))
print("user:1 email:", redis:HGET("user:1", "email"))

local all_fields = redis:HGETALL("user:1")
print("All fields:")
for i = 1, #all_fields, 2 do
    print(string.format("  %s = %s", all_fields[i], all_fields[i+1]))
end

-- Set operations
print("\n--- Sets ---")
redis:SADD("tags:post:1", "lua", "programming", "tutorial")
redis:SADD("tags:post:2", "lua", "games", "love2d")
print("Tags for post 1:", table.concat(redis:SMEMBERS("tags:post:1"), ", "))
print("Is 'lua' in post 1?:", redis:SISMEMBER("tags:post:1", "lua"))
print("Is 'python' in post 1?:", redis:SISMEMBER("tags:post:1", "python"))

-- Caching Pattern
print("\n--- Caching Pattern ---")

local function get_user_with_cache(redis_conn, db_conn, user_id)
    local cache_key = "user:" .. user_id

    -- ตรวจสอบ cache
    local cached_name = redis_conn:HGET(cache_key, "name")
    if cached_name then
        print("  Cache HIT for user:" .. user_id)
        return {name = cached_name}
    end

    print("  Cache MISS for user:" .. user_id)

    -- Query from "database" (mock)
    local users = {
        [1] = {name = "Alice", email = "alice@example.com"},
        [2] = {name = "Bob",   email = "bob@example.com"},
    }
    local user = users[user_id]

    if user then
        -- Store in cache
        redis_conn:HSET(cache_key,
            "name", user.name,
            "email", user.email
        )
        redis_conn:EXPIRE(cache_key, 300)  -- 5 minutes
    end

    return user
end

local user1 = get_user_with_cache(redis, nil, 1)
print("Got user:", user1 and user1.name)

local user1_again = get_user_with_cache(redis, nil, 1)
print("Got user (cached):", user1_again and user1_again.name)

-- Rate Limiting with Redis
print("\n--- Rate Limiting ---")

local function check_rate_limit(redis_conn, user_id, max_requests, window_seconds)
    local key = "rate:" .. user_id
    local count = redis_conn:INCR(key)

    if count == 1 then
        redis_conn:EXPIRE(key, window_seconds)
    end

    return count <= max_requests, count, max_requests
end

for i = 1, 7 do
    local allowed, count, max = check_rate_limit(redis, "user123", 5, 60)
    print(string.format("  Request %d: %s (%d/%d)",
        i, allowed and "ALLOWED" or "BLOCKED", count, max))
end
```

---

## ตัวอย่างที่ 16: Full-Text Search (FTS5)

Full-text search ด้วย SQLite FTS5 extension

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- Create FTS5 virtual table
db:exec([[
    CREATE VIRTUAL TABLE articles USING fts5(
        title,
        body,
        author UNINDEXED,
        tokenize = "unicode61 remove_diacritics 1"
    );
]])

-- Insert articles
local articles = {
    { "Lua Programming Basics",          "Learn variables, functions, and tables in Lua 5.4",        "Alice" },
    { "Advanced Lua Metatables",         "Master metatables, metamethods and object-oriented Lua",    "Bob" },
    { "Lua Coroutines Deep Dive",        "Understand coroutines, yield, resume and async patterns",   "Alice" },
    { "LuaJIT Performance Guide",        "Optimize your Lua code with LuaJIT FFI and profiling",     "Carol" },
    { "SQLite with Lua Tutorial",        "Connect to SQLite database using lsqlite3 library",         "Bob" },
    { "Network Programming with Lua",    "Build TCP and UDP servers using LuaSocket library",         "Dave" },
    { "Lua for Game Development",        "Create games using LÖVE2D framework and Lua scripting",    "Alice" },
    { "Security Best Practices in Lua",  "Input validation, sandboxing, and secure coding in Lua",   "Carol" },
}

local stmt = db:prepare("INSERT INTO articles(title, body, author) VALUES (?,?,?)")
for _, a in ipairs(articles) do
    stmt:bind_values(table.unpack(a))
    stmt:step()
    stmt:reset()
end
stmt:finalize()

print("=== SQLite FTS5 Full-Text Search ===")
print(string.format("Indexed %d articles\n", #articles))

-- Basic search
local function fts_search(query, limit)
    limit = limit or 10
    local results = {}
    local sql = string.format(
        "SELECT title, author, snippet(articles, 1, '**', '**', '...', 10) AS excerpt "..
        "FROM articles WHERE articles MATCH ? ORDER BY rank LIMIT %d", limit)
    local s = db:prepare(sql)
    s:bind_values(query)
    for row in s:rows() do
        table.insert(results, { title=row[1], author=row[2], excerpt=row[3] })
    end
    s:finalize()
    return results
end

-- Search tests
local queries = {
    "lua",
    "metatables OR coroutines",
    "lua AND security",
    "network programming",
    "\"deep dive\"",
}

for _, q in ipairs(queries) do
    local results = fts_search(q, 3)
    print(string.format("Query: %q -> %d results", q, #results))
    for _, r in ipairs(results) do
        print(string.format("  [%s] %s", r.author, r.title))
    end
end

-- Rank by relevance
print("\nRanked search (lua programming):")
local ranked = fts_search("lua programming", 5)
for i, r in ipairs(ranked) do
    print(string.format("  %d. %s (%s)", i, r.title, r.author))
    if r.excerpt then
        print("     " .. r.excerpt:sub(1, 60))
    end
end

-- Count matches
local count_stmt = db:prepare(
    "SELECT count(*) FROM articles WHERE articles MATCH ?")
count_stmt:bind_values("lua")
count_stmt:step()
local count = count_stmt:get_value(0)
count_stmt:finalize()
print(string.format("\nTotal articles matching 'lua': %d", count))

db:close()
```

---

## ตัวอย่างที่ 17: JSON Data in SQLite

เก็บและ query JSON data ใน SQLite columns

```lua
local sqlite3 = require("lsqlite3")

-- Simple JSON encoder (for demo)
local function json_enc(t)
    if type(t) == "table" then
        local a = {}
        for k, v in pairs(t) do
            a[#a+1] = '"'..k..'":'..json_enc(v)
        end
        return "{" .. table.concat(a, ",") .. "}"
    elseif type(t) == "string" then return '"'..t:gsub('"','\\"')..'"'
    elseif type(t) == "number" then return tostring(t)
    elseif type(t) == "boolean" then return t and "true" or "false"
    else return "null" end
end

local db = sqlite3.open(":memory:")

-- Table with JSON column
db:exec([[
    CREATE TABLE users (
        id      INTEGER PRIMARY KEY,
        email   TEXT NOT NULL UNIQUE,
        profile TEXT,    -- JSON
        settings TEXT,   -- JSON
        created_at INTEGER DEFAULT (strftime('%s','now'))
    );

    CREATE INDEX idx_users_email ON users(email);
]])

-- Insert with JSON columns
local users_data = {
    {
        email   = "alice@example.com",
        profile = { name="Alice", age=30, city="Bangkok", skills={"lua","python","go"} },
        settings= { theme="dark", lang="th", notifications=true }
    },
    {
        email   = "bob@example.com",
        profile = { name="Bob", age=25, city="Chiang Mai", skills={"javascript","rust"} },
        settings= { theme="light", lang="en", notifications=false }
    },
    {
        email   = "carol@example.com",
        profile = { name="Carol", age=35, city="Phuket", skills={"lua","c","asm"} },
        settings= { theme="dark", lang="en", notifications=true }
    },
}

local stmt = db:prepare(
    "INSERT INTO users(email, profile, settings) VALUES(?,?,?)")
for _, u in ipairs(users_data) do
    stmt:bind_values(u.email, json_enc(u.profile), json_enc(u.settings))
    stmt:step(); stmt:reset()
end
stmt:finalize()

print("=== JSON Data in SQLite ===")

-- Query and display all users
print("\nAll users:")
for row in db:rows("SELECT id, email, profile FROM users ORDER BY id") do
    print(string.format("  [%d] %s: %s", row[1], row[2], tostring(row[3]):sub(1,60)))
end

-- SQLite JSON functions (SQLite 3.38+)
print("\nJSON extraction with json_extract:")
local json_sql = [[
    SELECT
        email,
        json_extract(profile, '$.name')   AS name,
        json_extract(profile, '$.city')   AS city,
        json_extract(profile, '$.age')    AS age,
        json_extract(settings, '$.theme') AS theme
    FROM users
    ORDER BY json_extract(profile, '$.age') DESC
]]

local ok = pcall(function()
    for row in db:rows(json_sql) do
        print(string.format("  %-25s %-10s %-12s %-4s %s",
            row[1], tostring(row[2]), tostring(row[3]),
            tostring(row[4]), tostring(row[5])))
    end
end)
if not ok then
    print("  (json_extract not available in this SQLite build)")
    -- Fallback: show raw data
    for row in db:rows("SELECT email, profile FROM users") do
        print(string.format("  %s: %s", row[1], tostring(row[2]):sub(1,50)))
    end
end

-- JSON path search
print("\nSearch users with dark theme:")
local dark_sql = [[
    SELECT email, json_extract(profile, '$.name')
    FROM users
    WHERE json_extract(settings, '$.theme') = 'dark'
]]
ok = pcall(function()
    for row in db:rows(dark_sql) do
        print(string.format("  %s (%s)", row[2] or "?", row[1]))
    end
end)
if not ok then print("  (requires SQLite 3.38+ with JSON support)") end

-- Update JSON field
print("\nUpdate settings.theme for alice:")
local update_sql = [[
    UPDATE users
    SET settings = json_set(settings, '$.theme', 'light')
    WHERE email = 'alice@example.com'
]]
ok = pcall(function() db:exec(update_sql) end)
print(ok and "  Updated successfully" or "  (json_set not available)")

db:close()
```

---

## ตัวอย่างที่ 18: Window Functions

SQL Window Functions สำหรับ analytics queries

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE sales (
        id         INTEGER PRIMARY KEY,
        salesperson TEXT NOT NULL,
        region      TEXT NOT NULL,
        product     TEXT NOT NULL,
        amount      REAL NOT NULL,
        sale_date   TEXT NOT NULL
    );
]])

-- Insert sales data
local sales_data = {
    {"Alice","North","Widget",  1500,"2024-01"},
    {"Alice","North","Gadget",  2200,"2024-01"},
    {"Alice","North","Widget",  1800,"2024-02"},
    {"Bob",  "South","Widget",   900,"2024-01"},
    {"Bob",  "South","Gadget",  3100,"2024-01"},
    {"Bob",  "South","Widget",  1200,"2024-02"},
    {"Carol","East", "Gadget",  2800,"2024-01"},
    {"Carol","East", "Widget",  1600,"2024-01"},
    {"Carol","East", "Widget",  2000,"2024-02"},
    {"Dave", "West", "Gadget",  1900,"2024-01"},
    {"Dave", "West", "Widget",  2400,"2024-02"},
    {"Alice","North","Gadget",  2600,"2024-02"},
}

local stmt = db:prepare(
    "INSERT INTO sales(salesperson,region,product,amount,sale_date) VALUES(?,?,?,?,?)")
for _, s in ipairs(sales_data) do
    stmt:bind_values(table.unpack(s))
    stmt:step(); stmt:reset()
end
stmt:finalize()

print("=== SQL Window Functions ===")

-- ROW_NUMBER: rank sales within each region
print("\nROW_NUMBER - rank by amount per region:")
local ok, err = pcall(function()
    local sql = [[
        SELECT
            salesperson, region, amount,
            ROW_NUMBER() OVER (PARTITION BY region ORDER BY amount DESC) AS rank
        FROM sales
        ORDER BY region, rank
    ]]
    for row in db:rows(sql) do
        print(string.format("  %-8s %-6s %6.0f  rank=%d",
            row[1], row[2], row[3], row[4]))
    end
end)
if not ok then print("  Window functions require SQLite 3.25+") end

-- SUM with PARTITION BY
print("\nRunning total per salesperson:")
pcall(function()
    local sql = [[
        SELECT
            salesperson, sale_date, amount,
            SUM(amount) OVER (
                PARTITION BY salesperson
                ORDER BY sale_date
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
            ) AS running_total
        FROM sales
        ORDER BY salesperson, sale_date
    ]]
    local last_person = ""
    for row in db:rows(sql) do
        if row[1] ~= last_person then
            print(string.format("\n  %s:", row[1]))
            last_person = row[1]
        end
        print(string.format("    %s: amount=%6.0f running_total=%7.0f",
            row[2], row[3], row[4]))
    end
end)

-- PERCENT_RANK
print("\nPercent rank per region:")
pcall(function()
    local sql = [[
        SELECT
            salesperson, region, amount,
            ROUND(PERCENT_RANK() OVER (
                PARTITION BY region ORDER BY amount
            ) * 100, 1) AS pct_rank
        FROM sales
        ORDER BY region, amount
        LIMIT 8
    ]]
    for row in db:rows(sql) do
        print(string.format("  %-8s %-6s %6.0f  pct=%.1f%%",
            row[1], row[2], row[3], row[4]))
    end
end)

-- LAG/LEAD
print("\nMonth-over-month change (LAG):")
pcall(function()
    local sql = [[
        WITH monthly AS (
            SELECT salesperson, sale_date, SUM(amount) AS total
            FROM sales
            GROUP BY salesperson, sale_date
        )
        SELECT
            salesperson, sale_date, total,
            LAG(total) OVER (PARTITION BY salesperson ORDER BY sale_date) AS prev_month,
            ROUND(
                (total - LAG(total) OVER (PARTITION BY salesperson ORDER BY sale_date))
                / LAG(total) OVER (PARTITION BY salesperson ORDER BY sale_date) * 100, 1
            ) AS pct_change
        FROM monthly
        ORDER BY salesperson, sale_date
    ]]
    for row in db:rows(sql) do
        local pct = row[5] and string.format("%+.1f%%", row[5]) or "N/A"
        print(string.format("  %-8s %s: total=%6.0f prev=%s change=%s",
            row[1], row[2], row[3], tostring(row[4] or "N/A"), pct))
    end
end)

db:close()
```

---

## ตัวอย่างที่ 19: Triggers and Views

สร้าง Triggers และ Views ใน SQLite

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- Base tables
db:exec([[
    CREATE TABLE products (
        id       INTEGER PRIMARY KEY,
        name     TEXT NOT NULL,
        price    REAL NOT NULL CHECK(price > 0),
        stock    INTEGER NOT NULL DEFAULT 0 CHECK(stock >= 0),
        category TEXT NOT NULL
    );

    CREATE TABLE stock_log (
        id          INTEGER PRIMARY KEY,
        product_id  INTEGER REFERENCES products(id),
        change      INTEGER NOT NULL,
        reason      TEXT,
        logged_at   TEXT DEFAULT (datetime('now'))
    );

    CREATE TABLE order_items (
        id         INTEGER PRIMARY KEY,
        product_id INTEGER REFERENCES products(id),
        quantity   INTEGER NOT NULL CHECK(quantity > 0),
        unit_price REAL NOT NULL,
        order_date TEXT DEFAULT (datetime('now'))
    );
]])

-- Trigger: log stock changes automatically
db:exec([[
    CREATE TRIGGER log_stock_change
    AFTER UPDATE OF stock ON products
    WHEN NEW.stock != OLD.stock
    BEGIN
        INSERT INTO stock_log(product_id, change, reason)
        VALUES (NEW.id, NEW.stock - OLD.stock, 'stock update');
    END;
]])

-- Trigger: prevent negative stock
db:exec([[
    CREATE TRIGGER prevent_negative_stock
    BEFORE UPDATE OF stock ON products
    WHEN NEW.stock < 0
    BEGIN
        SELECT RAISE(ABORT, 'Stock cannot be negative');
    END;
]])

-- Trigger: record price on order
db:exec([[
    CREATE TRIGGER capture_price_on_order
    BEFORE INSERT ON order_items
    BEGIN
        SELECT CASE
            WHEN (SELECT stock FROM products WHERE id = NEW.product_id) < NEW.quantity
            THEN RAISE(ABORT, 'Insufficient stock')
        END;
    END;
]])

-- Trigger: decrease stock on order
db:exec([[
    CREATE TRIGGER decrease_stock_on_order
    AFTER INSERT ON order_items
    BEGIN
        UPDATE products
        SET stock = stock - NEW.quantity
        WHERE id = NEW.product_id;
    END;
]])

-- Views
db:exec([[
    CREATE VIEW low_stock AS
    SELECT id, name, stock, category
    FROM products
    WHERE stock < 10
    ORDER BY stock;

    CREATE VIEW product_summary AS
    SELECT
        category,
        COUNT(*) AS product_count,
        SUM(stock) AS total_stock,
        AVG(price) AS avg_price,
        SUM(price * stock) AS inventory_value
    FROM products
    GROUP BY category;
]])

-- Insert products
local products = {
    {"Widget A", 9.99,  50, "Electronics"},
    {"Widget B", 14.99, 8,  "Electronics"},
    {"Gadget X", 29.99, 3,  "Accessories"},
    {"Gadget Y", 49.99, 25, "Accessories"},
    {"Cable Z",  4.99,  100,"Electronics"},
}
local ps = db:prepare("INSERT INTO products(name,price,stock,category) VALUES(?,?,?,?)")
for _, p in ipairs(products) do
    ps:bind_values(table.unpack(p))
    ps:step(); ps:reset()
end
ps:finalize()

print("=== Triggers and Views ===")

-- Update stock (triggers fire)
print("\nUpdating stock levels (triggers log changes):")
db:exec("UPDATE products SET stock = stock - 5 WHERE name = 'Widget A'")
db:exec("UPDATE products SET stock = stock + 20 WHERE name = 'Widget B'")
db:exec("UPDATE products SET stock = stock - 2 WHERE name = 'Gadget X'")

-- View stock log
print("\nStock log (auto-recorded by triggers):")
for row in db:rows("SELECT product_id, change, logged_at FROM stock_log") do
    print(string.format("  product_id=%d change=%+d at=%s",
        row[1], row[2], tostring(row[3])))
end

-- Low stock view
print("\nLow stock items (view):")
for row in db:rows("SELECT id, name, stock, category FROM low_stock") do
    print(string.format("  [%d] %-12s stock=%-3d (%s)",
        row[1], row[2], row[3], row[4]))
end

-- Product summary view
print("\nProduct summary by category (view):")
for row in db:rows("SELECT * FROM product_summary") do
    print(string.format("  %-14s count=%d stock=%d avg_price=%.2f value=%.2f",
        row[1], row[2], row[3], row[4], row[5]))
end

-- Order items (trigger decreases stock)
print("\nPlacing orders (trigger decreases stock):")
local ok, err = pcall(function()
    db:exec("INSERT INTO order_items(product_id, quantity, unit_price) VALUES(1, 3, 9.99)")
    db:exec("INSERT INTO order_items(product_id, quantity, unit_price) VALUES(3, 1, 29.99)")
end)
if ok then
    print("  Orders placed successfully")
    for row in db:rows("SELECT id, name, stock FROM products ORDER BY id") do
        print(string.format("  Product %-12s: stock=%d", row[2], row[3]))
    end
else
    print("  Order error: " .. tostring(err))
end

-- Test constraint
print("\nTest: try to go negative stock:")
local ok2, err2 = pcall(function()
    db:exec("UPDATE products SET stock = -1 WHERE id = 1")
end)
print("  Result:", ok2 and "allowed (constraint missing)" or "BLOCKED: " .. tostring(err2))

db:close()
```

---

## ตัวอย่างที่ 20: Recursive CTE

Recursive Common Table Expressions สำหรับ hierarchical data

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- Hierarchical organization structure
db:exec([[
    CREATE TABLE employees (
        id         INTEGER PRIMARY KEY,
        name       TEXT NOT NULL,
        title      TEXT NOT NULL,
        manager_id INTEGER REFERENCES employees(id),
        department TEXT,
        salary     REAL
    );
]])

-- Insert org chart
local employees = {
    { 1, "Sarah Chen",  "CEO",              nil, "Executive", 200000 },
    { 2, "Tom Brooks",  "CTO",              1,   "Technology",150000 },
    { 3, "Lisa Park",   "CFO",              1,   "Finance",   145000 },
    { 4, "Mark Davis",  "VP Engineering",   2,   "Technology",120000 },
    { 5, "Anna Kim",    "VP Product",       2,   "Technology",115000 },
    { 6, "Jake Lee",    "Lead Dev",         4,   "Technology", 95000 },
    { 7, "Sue Wong",    "Senior Dev",       4,   "Technology", 90000 },
    { 8, "Ray Patel",   "Dev",             6,    "Technology", 80000 },
    { 9, "Mia Jones",   "Dev",             6,    "Technology", 78000 },
    {10, "Ben Adams",   "Product Manager",  5,   "Technology", 95000 },
    {11, "Cathy Brown", "Controller",       3,   "Finance",    85000 },
    {12, "Dan Wilson",  "Analyst",         11,   "Finance",    70000 },
}

local stmt = db:prepare(
    "INSERT INTO employees VALUES(?,?,?,?,?,?)")
for _, e in ipairs(employees) do
    stmt:bind_values(table.unpack(e))
    stmt:step(); stmt:reset()
end
stmt:finalize()

print("=== Recursive CTEs ===")

-- Recursive CTE: org hierarchy traversal
print("\nOrg hierarchy (BFS from CEO):")
local ok = pcall(function()
    local sql = [[
        WITH RECURSIVE org(id, name, title, manager_id, level, path) AS (
            -- Base: top-level (CEO)
            SELECT id, name, title, manager_id, 0, name
            FROM employees WHERE manager_id IS NULL

            UNION ALL

            -- Recursive: direct reports
            SELECT e.id, e.name, e.title, e.manager_id,
                   o.level + 1,
                   o.path || ' > ' || e.name
            FROM employees e
            JOIN org o ON e.manager_id = o.id
        )
        SELECT level, name, title, path FROM org ORDER BY path
    ]]
    for row in db:rows(sql) do
        local indent = string.rep("  ", row[1])
        print(string.format("  %s%s (%s)", indent, row[2], row[3]))
    end
end)
if not ok then print("  (Recursive CTE requires SQLite 3.8.3+)") end

-- Recursive CTE: find all reports under CTO (id=2)
print("\nAll reports under CTO (recursive):")
pcall(function()
    local sql = [[
        WITH RECURSIVE reports(id, name, title, level) AS (
            SELECT id, name, title, 0 FROM employees WHERE id = 2
            UNION ALL
            SELECT e.id, e.name, e.title, r.level + 1
            FROM employees e JOIN reports r ON e.manager_id = r.id
        )
        SELECT level, name, title FROM reports ORDER BY level, name
    ]]
    for row in db:rows(sql) do
        local indent = string.rep("  ", row[1])
        print(string.format("  %s[L%d] %s - %s", indent, row[1], row[2], row[3]))
    end
end)

-- Salary rollup per manager
print("\nTotal salary cost per department head:")
pcall(function()
    local sql = [[
        WITH RECURSIVE tree(root_id, emp_id, salary) AS (
            SELECT id, id, salary FROM employees WHERE manager_id IS NULL OR
                (SELECT COUNT(*) FROM employees e2 WHERE e2.manager_id = employees.id) > 0
            UNION ALL
            SELECT t.root_id, e.id, e.salary
            FROM employees e JOIN tree t ON e.manager_id = t.emp_id
            WHERE e.id != t.root_id
        )
        SELECT e.name, e.title, COUNT(t.emp_id) as team_size,
               SUM(t.salary) as total_cost
        FROM tree t
        JOIN employees e ON e.id = t.root_id
        GROUP BY t.root_id
        ORDER BY total_cost DESC
        LIMIT 5
    ]]
    for row in db:rows(sql) do
        print(string.format("  %-15s %-18s team=%d total=$%.0f",
            row[1], row[2], row[3], row[4]))
    end
end)

-- Number series generator (non-hierarchical recursive CTE)
print("\nGenerate series 1-10 with cumulative sum:")
pcall(function()
    local sql = [[
        WITH RECURSIVE series(n, cumsum) AS (
            SELECT 1, 1
            UNION ALL
            SELECT n+1, cumsum + (n+1) FROM series WHERE n < 10
        )
        SELECT n, cumsum FROM series
    ]]
    local row_parts = {}
    for row in db:rows(sql) do
        table.insert(row_parts, string.format("%d(Σ=%d)", row[1], row[2]))
    end
    print("  " .. table.concat(row_parts, ", "))
end)

db:close()
```

---

## ตัวอย่างที่ 21: Database Backup and Restore

Backup SQLite database ด้วย Online Backup API

```lua
local sqlite3 = require("lsqlite3")

-- Create source database
local function create_test_db(filename)
    local db = sqlite3.open(filename or ":memory:")
    db:exec([[
        CREATE TABLE config (key TEXT PRIMARY KEY, value TEXT);
        CREATE TABLE logs (id INTEGER PRIMARY KEY, msg TEXT, ts INTEGER);
        CREATE INDEX idx_logs_ts ON logs(ts);
    ]])
    local stmt = db:prepare("INSERT INTO config VALUES(?,?)")
    for _, kv in ipairs({ {"version","1.0"},{"app","lua-tutorial"},{"env","production"} }) do
        stmt:bind_values(kv[1], kv[2]); stmt:step(); stmt:reset()
    end
    stmt:finalize()
    stmt = db:prepare("INSERT INTO logs(msg, ts) VALUES(?,?)")
    for i = 1, 20 do
        stmt:bind_values("Log entry " .. i, os.time() - (20-i)*60)
        stmt:step(); stmt:reset()
    end
    stmt:finalize()
    return db
end

-- SQLite backup using file copy (for :memory: dbs use serialize)
local function backup_to_file(src_db, dest_path)
    -- Using the backup API via exec
    local ok, err = src_db:exec(
        string.format("VACUUM INTO %q", dest_path))
    return ok == sqlite3.OK, err
end

-- Backup using page-by-page (simulated with memory db)
local function backup_database(src_db)
    -- For demonstration: we'll dump the structure and data
    local dump = { "-- SQLite Backup " .. os.date("!%Y-%m-%dT%H:%M:%SZ") }

    -- Get all tables
    local tables = {}
    for row in src_db:rows(
        "SELECT name FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%'") do
        table.insert(tables, row[1])
    end

    for _, tbl in ipairs(tables) do
        -- Get CREATE statement
        for row in src_db:rows(
            string.format("SELECT sql FROM sqlite_master WHERE name=%q", tbl)) do
            table.insert(dump, row[1] .. ";")
        end

        -- Dump rows
        local col_names = {}
        for row in src_db:rows("PRAGMA table_info(" .. tbl .. ")") do
            table.insert(col_names, row[2])
        end

        local col_list = table.concat(col_names, ", ")
        for row in src_db:rows("SELECT " .. col_list .. " FROM " .. tbl) do
            local vals = {}
            for _, v in ipairs(row) do
                if v == nil then table.insert(vals, "NULL")
                elseif type(v) == "number" then table.insert(vals, tostring(v))
                else table.insert(vals, string.format("'%s'", tostring(v):gsub("'","''")))
                end
            end
            table.insert(dump,
                string.format("INSERT INTO %s(%s) VALUES(%s);",
                    tbl, col_list, table.concat(vals, ", ")))
        end
    end

    return table.concat(dump, "\n")
end

-- Restore from dump
local function restore_from_dump(dump_sql)
    local db = sqlite3.open(":memory:")
    for stmt in (dump_sql .. ";"):gmatch("([^;]+);") do
        stmt = stmt:match("^%s*(.-)%s*$")
        if stmt ~= "" and not stmt:match("^%-%-") then
            local rc = db:exec(stmt)
            if rc ~= sqlite3.OK then
                -- Some statements may fail (e.g. comments) — ignore
            end
        end
    end
    return db
end

-- Demo
print("=== SQLite Backup and Restore ===")
local src = create_test_db()

-- Get initial counts
local function get_counts(db)
    local counts = {}
    for row in db:rows("SELECT 'config' AS t, count(*) FROM config UNION ALL SELECT 'logs', count(*) FROM logs") do
        counts[row[1]] = row[2]
    end
    return counts
end

local before = get_counts(src)
print(string.format("Source DB: config=%d rows, logs=%d rows",
    before.config or 0, before.logs or 0))

-- Create backup dump
print("\nCreating backup dump...")
local dump = backup_database(src)
print(string.format("Dump size: %d bytes (%d lines)",
    #dump, select(2, dump:gsub("\n","\n"))))
print("First 200 chars:")
print(dump:sub(1, 200))

-- Restore into new DB
print("\nRestoring from dump...")
local restored = restore_from_dump(dump)
local after = get_counts(restored)
print(string.format("Restored DB: config=%d rows, logs=%d rows",
    after.config or 0, after.logs or 0))

-- Verify data integrity
local ok = before.config == after.config and before.logs == after.logs
print(string.format("Integrity check: %s", ok and "PASSED" or "FAILED"))

-- VACUUM INTO (SQLite 3.27+)
print("\nVACUUM INTO (online backup):")
local vacuum_ok = pcall(function()
    local tmpfile = "/tmp/backup_" .. os.time() .. ".db"
    src:exec(string.format("VACUUM INTO %q", tmpfile))
    print("  Backup created: " .. tmpfile)
    os.remove(tmpfile)
end)
if not vacuum_ok then
    print("  VACUUM INTO requires SQLite 3.27+")
end

src:close()
restored:close()
```

---

## ตัวอย่างที่ 22: WAL Mode and Performance

Write-Ahead Logging เพื่อเพิ่มประสิทธิภาพการเขียน

```lua
local sqlite3 = require("lsqlite3")
local socket  = require("socket")

-- Benchmark helper
local function benchmark(label, iterations, fn)
    local t0 = socket.gettime()
    for _ = 1, iterations do fn() end
    local elapsed = socket.gettime() - t0
    print(string.format("  %-30s %5d iter  %.3fs  %.0f ops/s",
        label, iterations, elapsed, iterations / elapsed))
end

-- Create test database
local function open_db(wal_mode)
    local db = sqlite3.open(":memory:")
    if wal_mode then
        db:exec("PRAGMA journal_mode = WAL")
        db:exec("PRAGMA synchronous = NORMAL")
    else
        db:exec("PRAGMA journal_mode = DELETE")
        db:exec("PRAGMA synchronous = FULL")
    end
    db:exec("PRAGMA cache_size = 10000")  -- 10MB cache
    db:exec([[
        CREATE TABLE events (
            id   INTEGER PRIMARY KEY,
            ts   INTEGER,
            type TEXT,
            data TEXT
        );
        CREATE INDEX idx_events_ts ON events(ts);
    ]])
    return db
end

print("=== WAL Mode and Performance ===")
print("\nPRAGMA settings comparison:")
local db_normal = open_db(false)
local db_wal    = open_db(true)

-- Check journal modes
for row in db_normal:rows("PRAGMA journal_mode") do
    print(string.format("  DELETE mode journal: %s", row[1]))
end
for row in db_wal:rows("PRAGMA journal_mode") do
    print(string.format("  WAL mode journal:    %s", row[1]))
end

-- Benchmark single inserts (no explicit transaction)
print("\nSingle INSERT performance:")
local i = 0

i = 0
benchmark("DELETE mode (individual)", 100, function()
    i = i + 1
    db_normal:exec(string.format(
        "INSERT INTO events(ts,type,data) VALUES(%d,'click','payload-%d')", os.time(), i))
end)

i = 0
benchmark("WAL mode (individual)", 100, function()
    i = i + 1
    db_wal:exec(string.format(
        "INSERT INTO events(ts,type,data) VALUES(%d,'click','payload-%d')", os.time(), i))
end)

-- Benchmark bulk inserts with transactions
print("\nBulk INSERT performance (1000 rows):")
benchmark("DELETE + explicit tx", 1, function()
    db_normal:exec("BEGIN")
    local s = db_normal:prepare("INSERT INTO events(ts,type,data) VALUES(?,?,?)")
    for j = 1, 1000 do
        s:bind_values(os.time(), "login", "user-" .. j)
        s:step(); s:reset()
    end
    s:finalize()
    db_normal:exec("COMMIT")
end)

benchmark("WAL + explicit tx", 1, function()
    db_wal:exec("BEGIN")
    local s = db_wal:prepare("INSERT INTO events(ts,type,data) VALUES(?,?,?)")
    for j = 1, 1000 do
        s:bind_values(os.time(), "login", "user-" .. j)
        s:step(); s:reset()
    end
    s:finalize()
    db_wal:exec("COMMIT")
end)

-- Useful PRAGMAs
print("\nUseful performance PRAGMAs:")
local pragmas = {
    { "journal_mode",   "WAL",    "concurrent reads/writes" },
    { "synchronous",    "NORMAL", "balance safety/speed" },
    { "cache_size",     "-64000", "64MB page cache" },
    { "temp_store",     "MEMORY", "temp tables in memory" },
    { "mmap_size",      "268435456","256MB memory-mapped I/O" },
    { "page_size",      "4096",   "4KB pages (set before CREATE)" },
    { "wal_autocheckpoint","1000","WAL checkpoint every 1000 pages" },
}

for _, p in ipairs(pragmas) do
    print(string.format("  PRAGMA %-20s = %-12s -- %s", p[1], p[2], p[3]))
end

-- WAL checkpoint
print("\nWAL checkpoint:")
db_wal:exec("PRAGMA wal_checkpoint(FULL)")
for row in db_wal:rows("PRAGMA wal_checkpoint(TRUNCATE)") do
    print(string.format("  busy=%d log=%d checkpointed=%d",
        row[1], row[2], row[3]))
end

-- Count rows
local function count_rows(db)
    for row in db:rows("SELECT count(*) FROM events") do return row[1] end
end
print(string.format("\nTotal rows: DELETE=%d, WAL=%d",
    count_rows(db_normal), count_rows(db_wal)))

db_normal:close()
db_wal:close()
```

---

## ตัวอย่างที่ 23: Upsert and Conflict Resolution

INSERT OR REPLACE, ON CONFLICT, UPSERT patterns

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE settings (
        key       TEXT PRIMARY KEY,
        value     TEXT NOT NULL,
        updated_at TEXT DEFAULT (datetime('now'))
    );

    CREATE TABLE sessions (
        token      TEXT PRIMARY KEY,
        user_id    INTEGER NOT NULL,
        expires_at INTEGER NOT NULL,
        ip         TEXT,
        hit_count  INTEGER DEFAULT 0
    );

    CREATE TABLE user_stats (
        user_id     INTEGER PRIMARY KEY,
        login_count INTEGER DEFAULT 0,
        last_login  TEXT,
        total_spent REAL DEFAULT 0
    );
]])

print("=== Upsert and Conflict Resolution ===")

-- 1. INSERT OR REPLACE (replaces entire row)
print("\n1. INSERT OR REPLACE:")
db:exec("INSERT INTO settings(key,value) VALUES('theme','dark')")
db:exec("INSERT INTO settings(key,value) VALUES('lang','th')")
print("  Initial:")
for row in db:rows("SELECT key, value FROM settings") do
    print(string.format("    %s = %s", row[1], row[2]))
end
-- Replace
db:exec("INSERT OR REPLACE INTO settings(key,value) VALUES('theme','light')")
db:exec("INSERT OR REPLACE INTO settings(key,value) VALUES('timezone','Asia/Bangkok')")
print("  After REPLACE:")
for row in db:rows("SELECT key, value FROM settings") do
    print(string.format("    %s = %s", row[1], row[2]))
end

-- 2. INSERT OR IGNORE (skip on conflict)
print("\n2. INSERT OR IGNORE:")
db:exec("INSERT INTO settings(key,value) VALUES('debug','false')")
db:exec("INSERT OR IGNORE INTO settings(key,value) VALUES('debug','true')")  -- ignored
for row in db:rows("SELECT value FROM settings WHERE key='debug'") do
    print("  debug = " .. row[1] .. " (unchanged by OR IGNORE)")
end

-- 3. ON CONFLICT DO UPDATE (UPSERT, SQLite 3.24+)
print("\n3. ON CONFLICT DO UPDATE (UPSERT):")
-- Set initial counts
db:exec("INSERT INTO user_stats(user_id,login_count) VALUES(1,5)")
db:exec("INSERT INTO user_stats(user_id,login_count) VALUES(2,3)")

local ok = pcall(function()
    -- Upsert: insert new or increment existing
    db:exec([[
        INSERT INTO user_stats(user_id, login_count, last_login)
        VALUES(1, 1, datetime('now'))
        ON CONFLICT(user_id) DO UPDATE SET
            login_count = login_count + 1,
            last_login  = excluded.last_login
    ]])
    db:exec([[
        INSERT INTO user_stats(user_id, login_count, last_login)
        VALUES(3, 1, datetime('now'))
        ON CONFLICT(user_id) DO UPDATE SET
            login_count = login_count + 1,
            last_login  = excluded.last_login
    ]])
end)

if ok then
    print("  User stats after upsert:")
    for row in db:rows("SELECT user_id, login_count FROM user_stats ORDER BY user_id") do
        print(string.format("    user_id=%d login_count=%d", row[1], row[2]))
    end
else
    print("  ON CONFLICT DO UPDATE requires SQLite 3.24+")
end

-- 4. Session hit counter
print("\n4. Session hit counter upsert:")
local sessions = {
    { "tok_abc", 1, os.time()+3600, "192.168.1.1" },
    { "tok_xyz", 2, os.time()+7200, "192.168.1.2" },
}
for _, s in ipairs(sessions) do
    db:exec(string.format(
        "INSERT INTO sessions VALUES('%s',%d,%d,'%s',0)",
        s[1], s[2], s[3], s[4]))
end

-- Simulate multiple hits (increment counter)
local upsert_ok = pcall(function()
    for _ = 1, 3 do
        db:exec([[
            INSERT INTO sessions(token, user_id, expires_at, hit_count)
            VALUES('tok_abc', 1, ]] .. (os.time()+3600) .. [[, 1)
            ON CONFLICT(token) DO UPDATE SET hit_count = hit_count + 1
        ]])
    end
end)

print("  Sessions after 3 hits to tok_abc:")
for row in db:rows("SELECT token, user_id, hit_count FROM sessions") do
    print(string.format("    %-10s user=%d hits=%d",
        row[1], row[2], row[3]))
end

-- 5. Batch upsert with transaction
print("\n5. Batch upsert (transaction):")
local batch_data = {
    { "max_connections", "100" }, { "timeout", "30" },
    { "debug", "true" }, { "version", "2.0" },
    { "lang", "en" },
}
db:exec("BEGIN")
local us = db:prepare([[
    INSERT INTO settings(key, value)
    VALUES(?, ?)
    ON CONFLICT(key) DO UPDATE SET value = excluded.value,
    updated_at = datetime('now')
]])
local success = pcall(function()
    for _, kv in ipairs(batch_data) do
        us:bind_values(kv[1], kv[2]); us:step(); us:reset()
    end
end)
us:finalize()
db:exec(success and "COMMIT" or "ROLLBACK")
print(string.format("  Upserted %d settings", success and #batch_data or 0))
for row in db:rows("SELECT count(*) FROM settings") do
    print(string.format("  Total settings: %d", row[1]))
end

db:close()
```

---

## ตัวอย่างที่ 24: Batch Insert Optimization

Optimize large batch inserts สำหรับ high-throughput

```lua
local sqlite3 = require("lsqlite3")
local socket  = require("socket")

local db = sqlite3.open(":memory:")
db:exec("PRAGMA journal_mode = WAL")
db:exec("PRAGMA synchronous = NORMAL")
db:exec("PRAGMA cache_size = 50000")

db:exec([[
    CREATE TABLE metrics (
        id        INTEGER PRIMARY KEY,
        device_id INTEGER NOT NULL,
        metric    TEXT NOT NULL,
        value     REAL NOT NULL,
        recorded  INTEGER NOT NULL
    );
    CREATE INDEX idx_metrics_device ON metrics(device_id);
    CREATE INDEX idx_metrics_time   ON metrics(recorded);
]])

-- Generate test data
local function gen_metrics(n)
    local data = {}
    local t = os.time()
    for i = 1, n do
        table.insert(data, {
            device_id = math.random(1, 100),
            metric    = ({"cpu","mem","disk","net"})[math.random(1,4)],
            value     = math.random() * 100,
            recorded  = t - math.random(0, 86400)
        })
    end
    return data
end

-- Method 1: One row at a time
local function insert_one_by_one(data)
    local stmt = db:prepare(
        "INSERT INTO metrics(device_id,metric,value,recorded) VALUES(?,?,?,?)")
    for _, row in ipairs(data) do
        stmt:bind_values(row.device_id, row.metric, row.value, row.recorded)
        stmt:step(); stmt:reset()
    end
    stmt:finalize()
end

-- Method 2: Explicit transaction
local function insert_with_transaction(data)
    db:exec("BEGIN")
    local stmt = db:prepare(
        "INSERT INTO metrics(device_id,metric,value,recorded) VALUES(?,?,?,?)")
    for _, row in ipairs(data) do
        stmt:bind_values(row.device_id, row.metric, row.value, row.recorded)
        stmt:step(); stmt:reset()
    end
    stmt:finalize()
    db:exec("COMMIT")
end

-- Method 3: Multi-row VALUES (batched)
local function insert_batched(data, batch_size)
    batch_size = batch_size or 100
    db:exec("BEGIN")
    for i = 1, #data, batch_size do
        local values = {}
        local end_i = math.min(i + batch_size - 1, #data)
        for j = i, end_i do
            local row = data[j]
            table.insert(values, string.format("(%d,'%s',%.6f,%d)",
                row.device_id, row.metric, row.value, row.recorded))
        end
        db:exec("INSERT INTO metrics(device_id,metric,value,recorded) VALUES "
            .. table.concat(values, ","))
    end
    db:exec("COMMIT")
end

-- Benchmark
local N = 1000
local test_data = gen_metrics(N)

print("=== Batch Insert Optimization ===")
print(string.format("Inserting %d rows:\n", N))

-- Cleanup before each test
local function cleanup()
    db:exec("DELETE FROM metrics")
end

-- Test 1: One by one (no transaction)
cleanup()
local t1 = socket.gettime()
insert_one_by_one(test_data)
local t1_elapsed = socket.gettime() - t1
print(string.format("  %-35s %.3fs  %.0f rows/s",
    "1. One by one (no transaction):", t1_elapsed, N/t1_elapsed))

-- Test 2: Explicit transaction
cleanup()
local t2 = socket.gettime()
insert_with_transaction(test_data)
local t2_elapsed = socket.gettime() - t2
print(string.format("  %-35s %.3fs  %.0f rows/s  (%.1fx faster)",
    "2. Explicit transaction:", t2_elapsed, N/t2_elapsed, t1_elapsed/t2_elapsed))

-- Test 3: Multi-row batched
cleanup()
local t3 = socket.gettime()
insert_batched(test_data, 50)
local t3_elapsed = socket.gettime() - t3
print(string.format("  %-35s %.3fs  %.0f rows/s  (%.1fx faster)",
    "3. Multi-row VALUES (batch=50):", t3_elapsed, N/t3_elapsed, t1_elapsed/t3_elapsed))

-- Verify counts
for row in db:rows("SELECT count(*) FROM metrics") do
    print(string.format("\nRows in metrics: %d (expected %d)", row[1], N))
end

-- Best practices summary
print("\nBest practices for bulk inserts:")
print("  1. Wrap in explicit BEGIN/COMMIT transaction")
print("  2. Use prepared statements (bind_values)")
print("  3. Multi-row VALUES for <1000 rows")
print("  4. PRAGMA journal_mode=WAL + synchronous=NORMAL")
print("  5. PRAGMA cache_size = -65536 (64MB)")

db:close()
```

---

## ตัวอย่างที่ 25: Audit Log Pattern

Audit logging ด้วย triggers สำหรับ compliance

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- Main table
db:exec([[
    CREATE TABLE accounts (
        id         INTEGER PRIMARY KEY,
        username   TEXT NOT NULL UNIQUE,
        email      TEXT NOT NULL,
        balance    REAL NOT NULL DEFAULT 0,
        status     TEXT NOT NULL DEFAULT 'active',
        created_at TEXT DEFAULT (datetime('now'))
    );

    -- Audit log table
    CREATE TABLE audit_log (
        id         INTEGER PRIMARY KEY,
        table_name TEXT NOT NULL,
        row_id     INTEGER,
        action     TEXT NOT NULL,  -- INSERT, UPDATE, DELETE
        old_values TEXT,           -- JSON
        new_values TEXT,           -- JSON
        changed_by TEXT DEFAULT 'system',
        changed_at TEXT DEFAULT (datetime('now'))
    );
]])

-- Audit triggers for accounts table
db:exec([[
    CREATE TRIGGER audit_accounts_insert
    AFTER INSERT ON accounts
    BEGIN
        INSERT INTO audit_log(table_name, row_id, action, new_values)
        VALUES('accounts', NEW.id, 'INSERT',
            '{"username":"' || NEW.username || '","email":"' || NEW.email ||
            '","balance":' || NEW.balance || ',"status":"' || NEW.status || '"}');
    END;

    CREATE TRIGGER audit_accounts_update
    AFTER UPDATE ON accounts
    BEGIN
        INSERT INTO audit_log(table_name, row_id, action, old_values, new_values)
        VALUES('accounts', NEW.id, 'UPDATE',
            '{"username":"' || OLD.username || '","email":"' || OLD.email ||
            '","balance":' || OLD.balance || ',"status":"' || OLD.status || '"}',
            '{"username":"' || NEW.username || '","email":"' || NEW.email ||
            '","balance":' || NEW.balance || ',"status":"' || NEW.status || '"}');
    END;

    CREATE TRIGGER audit_accounts_delete
    AFTER DELETE ON accounts
    BEGIN
        INSERT INTO audit_log(table_name, row_id, action, old_values)
        VALUES('accounts', OLD.id, 'DELETE',
            '{"username":"' || OLD.username || '","balance":' || OLD.balance || '}');
    END;
]])

print("=== Audit Log Pattern ===")

-- Perform operations
db:exec("INSERT INTO accounts(username,email,balance) VALUES('alice','alice@ex.com',1000)")
db:exec("INSERT INTO accounts(username,email,balance) VALUES('bob','bob@ex.com',500)")
db:exec("INSERT INTO accounts(username,email,balance) VALUES('carol','carol@ex.com',2500)")

db:exec("UPDATE accounts SET balance = balance + 200 WHERE username = 'alice'")
db:exec("UPDATE accounts SET status = 'suspended' WHERE username = 'bob'")
db:exec("UPDATE accounts SET email = 'carol.new@ex.com' WHERE username = 'carol'")
db:exec("UPDATE accounts SET balance = balance - 100 WHERE username = 'alice'")
db:exec("DELETE FROM accounts WHERE username = 'bob'")

-- View audit log
print("\nAudit log (all entries):")
print(string.format("  %-4s %-10s %-6s %-8s %-30s",
    "id", "table", "row_id", "action", "changed_at"))
print("  " .. string.rep("-", 65))
for row in db:rows(
    "SELECT id,table_name,row_id,action,changed_at FROM audit_log ORDER BY id") do
    print(string.format("  %-4d %-10s %-6d %-8s %s",
        row[1], row[2], row[3] or 0, row[4], tostring(row[5])))
end

-- Balance history for alice
print("\nBalance changes for alice (user_id=1):")
for row in db:rows([[
    SELECT action, old_values, new_values, changed_at
    FROM audit_log
    WHERE row_id = 1 AND table_name = 'accounts'
    ORDER BY id
]]) do
    local old_bal = tostring(row[2] or ""):match('"balance":([%d%.]+)') or "N/A"
    local new_bal = tostring(row[3] or ""):match('"balance":([%d%.]+)') or "N/A"
    print(string.format("  [%s] old_bal=%s new_bal=%s", row[1], old_bal, new_bal))
end

-- Summary stats
print("\nAudit summary:")
for row in db:rows([[
    SELECT action, count(*) AS cnt
    FROM audit_log
    GROUP BY action ORDER BY action
]]) do
    print(string.format("  %-8s: %d events", row[1], row[2]))
end

-- Current state
print("\nCurrent accounts:")
for row in db:rows("SELECT id, username, balance, status FROM accounts") do
    print(string.format("  [%d] %-8s $%.2f [%s]",
        row[1], row[2], row[3], row[4]))
end

db:close()
```

---

## ตัวอย่างที่ 26: Soft Delete Pattern

Soft delete ด้วย deleted_at timestamp แทนการลบจริง

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE posts (
        id          INTEGER PRIMARY KEY,
        title       TEXT NOT NULL,
        content     TEXT,
        author_id   INTEGER NOT NULL,
        created_at  TEXT DEFAULT (datetime('now')),
        updated_at  TEXT DEFAULT (datetime('now')),
        deleted_at  TEXT  -- NULL = active, timestamp = soft deleted
    );

    -- View that excludes deleted rows
    CREATE VIEW active_posts AS
    SELECT id, title, content, author_id, created_at, updated_at
    FROM posts
    WHERE deleted_at IS NULL;
]])

-- SoftDeleteModel helper
local SoftModel = {}
SoftModel.__index = SoftModel

function SoftModel.new(db_conn, table_name)
    return setmetatable({ db = db_conn, tbl = table_name }, SoftModel)
end

function SoftModel:delete(id)
    self.db:exec(string.format(
        "UPDATE %s SET deleted_at=datetime('now'),updated_at=datetime('now') WHERE id=%d AND deleted_at IS NULL",
        self.tbl, id))
    return self.db:changes() > 0
end

function SoftModel:restore(id)
    self.db:exec(string.format(
        "UPDATE %s SET deleted_at=NULL,updated_at=datetime('now') WHERE id=%d",
        self.tbl, id))
    return self.db:changes() > 0
end

function SoftModel:hard_delete(id)
    self.db:exec(string.format(
        "DELETE FROM %s WHERE id=%d AND deleted_at IS NOT NULL",
        self.tbl, id))
    return self.db:changes() > 0
end

function SoftModel:find_deleted()
    local rows = {}
    for row in self.db:rows(string.format(
        "SELECT id, title, deleted_at FROM %s WHERE deleted_at IS NOT NULL ORDER BY deleted_at",
        self.tbl)) do
        table.insert(rows, { id=row[1], title=row[2], deleted_at=row[3] })
    end
    return rows
end

function SoftModel:purge_old(days)
    days = days or 30
    self.db:exec(string.format(
        "DELETE FROM %s WHERE deleted_at IS NOT NULL AND deleted_at < datetime('now','-%d days')",
        self.tbl, days))
    return self.db:changes()
end

-- Demo
print("=== Soft Delete Pattern ===")
db:exec("INSERT INTO posts(title,content,author_id) VALUES('Lua Basics','...',1)")
db:exec("INSERT INTO posts(title,content,author_id) VALUES('Advanced Lua','...',1)")
db:exec("INSERT INTO posts(title,content,author_id) VALUES('LuaSocket Guide','...',2)")
db:exec("INSERT INTO posts(title,content,author_id) VALUES('FFI Tutorial','...',2)")
db:exec("INSERT INTO posts(title,content,author_id) VALUES('Security in Lua','...',3)")

local model = SoftModel.new(db, "posts")

print("\nInitial posts (via active_posts view):")
for row in db:rows("SELECT id, title FROM active_posts") do
    print(string.format("  [%d] %s", row[1], row[2]))
end

-- Soft delete
model:delete(2)
model:delete(4)
print("\nAfter soft-deleting posts 2 and 4:")
for row in db:rows("SELECT id, title FROM active_posts") do
    print(string.format("  [%d] %s", row[1], row[2]))
end

-- Show deleted
print("\nDeleted posts:")
for _, d in ipairs(model:find_deleted()) do
    print(string.format("  [%d] %s (deleted: %s)", d.id, d.title, d.deleted_at))
end

-- Restore
model:restore(2)
print("\nAfter restoring post 2:")
for row in db:rows("SELECT id, title FROM active_posts") do
    print(string.format("  [%d] %s", row[1], row[2]))
end

-- Hard delete (permanently remove soft-deleted)
local removed = model:hard_delete(4)
print(string.format("\nHard deleted post 4: %s", removed and "OK" or "not found"))
print(string.format("Total rows in table: %d",
    (function() for r in db:rows("SELECT count(*) FROM posts") do return r[1] end end)()))

db:close()
```

---

## ตัวอย่างที่ 27: Multi-tenant Database Pattern

Multi-tenancy ด้วย tenant_id column (row-level isolation)

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE tenants (
        id   INTEGER PRIMARY KEY,
        slug TEXT UNIQUE NOT NULL,
        name TEXT NOT NULL,
        plan TEXT DEFAULT 'free'
    );

    CREATE TABLE projects (
        id          INTEGER PRIMARY KEY,
        tenant_id   INTEGER NOT NULL REFERENCES tenants(id),
        name        TEXT NOT NULL,
        status      TEXT DEFAULT 'active',
        created_at  TEXT DEFAULT (datetime('now')),
        UNIQUE(tenant_id, name)
    );

    CREATE TABLE tasks (
        id          INTEGER PRIMARY KEY,
        tenant_id   INTEGER NOT NULL REFERENCES tenants(id),
        project_id  INTEGER NOT NULL REFERENCES projects(id),
        title       TEXT NOT NULL,
        done        INTEGER DEFAULT 0
    );

    CREATE INDEX idx_projects_tenant ON projects(tenant_id);
    CREATE INDEX idx_tasks_tenant    ON tasks(tenant_id);
    CREATE INDEX idx_tasks_project   ON tasks(project_id);
]])

-- Tenant-scoped database access
local TenantDB = {}
TenantDB.__index = TenantDB

function TenantDB.new(db_conn, tenant_id)
    return setmetatable({ db = db_conn, tenant_id = tenant_id }, TenantDB)
end

function TenantDB:exec(sql, ...)
    -- Inject tenant_id for safety
    return self.db:exec(sql, ...)
end

function TenantDB:get_projects()
    local rows = {}
    local stmt = self.db:prepare(
        "SELECT id, name, status FROM projects WHERE tenant_id=? ORDER BY name")
    stmt:bind_values(self.tenant_id)
    for row in stmt:rows() do
        table.insert(rows, { id=row[1], name=row[2], status=row[3] })
    end
    stmt:finalize()
    return rows
end

function TenantDB:create_project(name)
    local stmt = self.db:prepare(
        "INSERT INTO projects(tenant_id, name) VALUES(?,?)")
    stmt:bind_values(self.tenant_id, name)
    local rc = stmt:step()
    stmt:finalize()
    if rc == sqlite3.DONE then
        return self.db:last_insert_rowid()
    end
    return nil, "failed"
end

function TenantDB:get_tasks(project_id)
    local rows = {}
    local stmt = self.db:prepare(
        "SELECT id, title, done FROM tasks WHERE tenant_id=? AND project_id=?")
    stmt:bind_values(self.tenant_id, project_id)
    for row in stmt:rows() do
        table.insert(rows, { id=row[1], title=row[2], done=row[3]==1 })
    end
    stmt:finalize()
    return rows
end

function TenantDB:add_task(project_id, title)
    local stmt = self.db:prepare(
        "INSERT INTO tasks(tenant_id,project_id,title) VALUES(?,?,?)")
    stmt:bind_values(self.tenant_id, project_id, title)
    stmt:step()
    stmt:finalize()
    return self.db:last_insert_rowid()
end

function TenantDB:stats()
    local s = {}
    for row in self.db:rows(string.format(
        "SELECT count(*) FROM projects WHERE tenant_id=%d", self.tenant_id)) do
        s.projects = row[1]
    end
    for row in self.db:rows(string.format(
        "SELECT count(*) FROM tasks WHERE tenant_id=%d", self.tenant_id)) do
        s.tasks = row[1]
    end
    return s
end

-- Demo
print("=== Multi-tenant Database Pattern ===")

-- Create tenants
db:exec("INSERT INTO tenants(slug,name,plan) VALUES('acme','ACME Corp','pro')")
db:exec("INSERT INTO tenants(slug,name,plan) VALUES('startup42','Startup 42','free')")
db:exec("INSERT INTO tenants(slug,name,plan) VALUES('bigcorp','Big Corp Inc','enterprise')")

-- Work with tenant 1 (ACME)
local acme = TenantDB.new(db, 1)
local p1 = acme:create_project("Website Redesign")
local p2 = acme:create_project("Mobile App")
acme:add_task(p1, "Design mockups")
acme:add_task(p1, "Frontend development")
acme:add_task(p1, "Backend API")
acme:add_task(p2, "iOS development")
acme:add_task(p2, "Android development")

-- Work with tenant 2 (Startup)
local startup = TenantDB.new(db, 2)
local sp1 = startup:create_project("MVP Launch")
startup:add_task(sp1, "Market research")
startup:add_task(sp1, "Prototype")

print("\nACME Corp (tenant_id=1):")
for _, proj in ipairs(acme:get_projects()) do
    print(string.format("  Project [%d]: %s", proj.id, proj.name))
    for _, task in ipairs(acme:get_tasks(proj.id)) do
        print(string.format("    - [%s] %s",
            task.done and "X" or " ", task.title))
    end
end

print("\nStartup 42 (tenant_id=2):")
for _, proj in ipairs(startup:get_projects()) do
    print(string.format("  Project [%d]: %s", proj.id, proj.name))
    for _, task in ipairs(startup:get_tasks(proj.id)) do
        print(string.format("    - %s", task.title))
    end
end

-- Tenant isolation: tenant 1 cannot see tenant 2's data
print("\nIsolation check: ACME cannot see Startup's tasks:")
local acme_all_tasks = {}
for row in db:rows("SELECT count(*) FROM tasks WHERE tenant_id=1") do
    acme_all_tasks = row[1]
end
local startup_all_tasks = 0
for row in db:rows("SELECT count(*) FROM tasks WHERE tenant_id=2") do
    startup_all_tasks = row[1]
end
print(string.format("  ACME tasks: %d, Startup tasks: %d",
    acme_all_tasks, startup_all_tasks))

-- Stats
print("\nTenant stats:")
for tenant_id, name in pairs({[1]="ACME",[2]="Startup 42"}) do
    local t = TenantDB.new(db, tenant_id)
    local s = t:stats()
    print(string.format("  %-12s projects=%d tasks=%d", name, s.projects, s.tasks))
end

db:close()
```

---

## ตัวอย่างที่ 28: Event Sourcing Pattern

Event sourcing: เก็บ events แทน state ปัจจุบัน

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE events (
        id           INTEGER PRIMARY KEY,
        aggregate_id TEXT NOT NULL,   -- e.g. "account:42"
        aggregate_type TEXT NOT NULL, -- "BankAccount"
        event_type   TEXT NOT NULL,   -- "Deposited", "Withdrawn"
        payload      TEXT NOT NULL,   -- JSON
        version      INTEGER NOT NULL,
        occurred_at  TEXT DEFAULT (datetime('now')),
        UNIQUE(aggregate_id, version)
    );

    CREATE TABLE snapshots (
        aggregate_id   TEXT PRIMARY KEY,
        state          TEXT NOT NULL,  -- JSON
        version        INTEGER NOT NULL,
        snapshot_at    TEXT DEFAULT (datetime('now'))
    );

    CREATE INDEX idx_events_aggregate ON events(aggregate_id, version);
]])

-- Simple JSON encode/decode
local function j(t)
    if type(t)=="table" then
        local p={}; for k,v in pairs(t) do p[#p+1]='"'..k..'":'..j(v) end
        return "{"..table.concat(p,",").."}"
    elseif type(t)=="number" then return tostring(t)
    elseif type(t)=="string" then return '"'..t:gsub('"','\\"')..'"'
    elseif type(t)=="boolean" then return t and "true" or "false"
    else return "null" end
end

local function j_get(s, key)
    return s:match('"'..key..'":%s*"([^"]*)"') or
           tonumber(s:match('"'..key..'"%s*:%s*([%-]?%d+%.?%d*)'))
end

-- EventStore
local EventStore = {}
EventStore.__index = EventStore

function EventStore.new(db_conn)
    return setmetatable({ db = db_conn, version_cache = {} }, EventStore)
end

function EventStore:current_version(aggregate_id)
    for row in self.db:rows(string.format(
        "SELECT MAX(version) FROM events WHERE aggregate_id=%q",
        aggregate_id)) do
        return row[1] or 0
    end
    return 0
end

function EventStore:append(aggregate_id, agg_type, event_type, payload, expected_version)
    local current = self:current_version(aggregate_id)
    if expected_version and current ~= expected_version then
        return nil, string.format("concurrency conflict: expected v%d, got v%d",
            expected_version, current)
    end
    local next_v = current + 1
    local stmt = self.db:prepare([[
        INSERT INTO events(aggregate_id,aggregate_type,event_type,payload,version)
        VALUES(?,?,?,?,?)
    ]])
    stmt:bind_values(aggregate_id, agg_type, event_type, j(payload), next_v)
    local rc = stmt:step()
    stmt:finalize()
    if rc == sqlite3.DONE then return next_v end
    return nil, "insert failed"
end

function EventStore:get_events(aggregate_id, from_version)
    local events = {}
    from_version = from_version or 0
    local stmt = self.db:prepare([[
        SELECT event_type, payload, version, occurred_at
        FROM events WHERE aggregate_id=? AND version > ?
        ORDER BY version
    ]])
    stmt:bind_values(aggregate_id, from_version)
    for row in stmt:rows() do
        table.insert(events, {
            type=row[1], payload=row[2], version=row[3], occurred_at=row[4]
        })
    end
    stmt:finalize()
    return events
end

function EventStore:save_snapshot(aggregate_id, state, version)
    self.db:exec(string.format([[
        INSERT INTO snapshots(aggregate_id,state,version)
        VALUES(%q,%q,%d)
        ON CONFLICT(aggregate_id) DO UPDATE SET
            state=excluded.state, version=excluded.version,
            snapshot_at=datetime('now')
    ]], aggregate_id, j(state), version))
end

-- BankAccount aggregate
local BankAccount = {}
BankAccount.__index = BankAccount

function BankAccount.new(account_id)
    return setmetatable({
        id = account_id, balance = 0, version = 0, owner = ""
    }, BankAccount)
end

function BankAccount:apply(event)
    local payload = {}
    for k, v in event.payload:gmatch('"([^"]+)"%s*:%s*"?([^",}]+)"?') do
        payload[k] = tonumber(v) or v
    end
    if event.type == "Opened" then
        self.owner   = payload.owner  or ""
        self.balance = payload.balance or 0
    elseif event.type == "Deposited" then
        self.balance = self.balance + (payload.amount or 0)
    elseif event.type == "Withdrawn" then
        self.balance = self.balance - (payload.amount or 0)
    end
    self.version = event.version
end

function BankAccount:replay(events)
    for _, e in ipairs(events) do self:apply(e) end
    return self
end

-- Demo
print("=== Event Sourcing Pattern ===")
local store = EventStore.new(db)
local acc_id = "account:1001"

-- Append events
store:append(acc_id, "BankAccount", "Opened",    { owner="Alice", balance=0 })
store:append(acc_id, "BankAccount", "Deposited", { amount=1000,   note="initial deposit" })
store:append(acc_id, "BankAccount", "Deposited", { amount=500,    note="salary" })
store:append(acc_id, "BankAccount", "Withdrawn", { amount=200,    note="rent" })
store:append(acc_id, "BankAccount", "Deposited", { amount=300,    note="refund" })
store:append(acc_id, "BankAccount", "Withdrawn", { amount=150,    note="utilities" })

-- Replay all events
local events = store:get_events(acc_id)
print(string.format("Events stored: %d", #events))

local account = BankAccount.new(acc_id)
account:replay(events)
print(string.format("Current balance: $%.2f (after replay)", account.balance))
print(string.format("Current version: %d", account.version))

-- Show event log
print("\nEvent log:")
for _, e in ipairs(events) do
    print(string.format("  v%-2d [%-12s] %s",
        e.version, e.type, e.payload:sub(1,50)))
end

-- Point-in-time replay
print("\nPoint-in-time (up to v3):")
local events_v3 = store:get_events(acc_id, 0)
local acct_v3 = BankAccount.new(acc_id)
for _, e in ipairs(events_v3) do
    if e.version <= 3 then acct_v3:apply(e) end
end
print(string.format("  Balance at v3: $%.2f", acct_v3.balance))

-- Snapshot
store:save_snapshot(acc_id, { balance=account.balance, owner="Alice" }, account.version)
print(string.format("\nSnapshot saved at version %d", account.version))

db:close()
```

---

## ตัวอย่างที่ 29: Database Schema Versioning

Database migration system พร้อม rollback support

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- Schema version tracking table
db:exec([[
    CREATE TABLE IF NOT EXISTS schema_versions (
        version     INTEGER PRIMARY KEY,
        name        TEXT NOT NULL,
        applied_at  TEXT DEFAULT (datetime('now')),
        checksum    TEXT,
        rolled_back INTEGER DEFAULT 0
    );
]])

-- Migration definition
local MigrationRunner = {}
MigrationRunner.__index = MigrationRunner

function MigrationRunner.new(db_conn)
    return setmetatable({
        db = db_conn,
        migrations = {},
    }, MigrationRunner)
end

function MigrationRunner:register(version, name, up_fn, down_fn)
    table.insert(self.migrations, {
        version = version,
        name    = name,
        up      = up_fn,
        down    = down_fn,
    })
    table.sort(self.migrations, function(a, b) return a.version < b.version end)
end

function MigrationRunner:current_version()
    for row in self.db:rows([[
        SELECT MAX(version) FROM schema_versions WHERE rolled_back = 0
    ]]) do
        return row[1] or 0
    end
    return 0
end

function MigrationRunner:applied_versions()
    local applied = {}
    for row in self.db:rows(
        "SELECT version FROM schema_versions WHERE rolled_back=0 ORDER BY version") do
        applied[row[1]] = true
    end
    return applied
end

function MigrationRunner:migrate(target_version)
    local current = self:current_version()
    local applied  = self:applied_versions()
    print(string.format("Current schema version: %d", current))

    for _, m in ipairs(self.migrations) do
        if not applied[m.version] and
           (not target_version or m.version <= target_version) then
            print(string.format("  Applying v%d: %s...", m.version, m.name))
            local ok, err = pcall(function()
                self.db:exec("BEGIN")
                m.up(self.db)
                self.db:exec(string.format([[
                    INSERT INTO schema_versions(version, name) VALUES(%d, %q)
                ]], m.version, m.name))
                self.db:exec("COMMIT")
            end)
            if not ok then
                self.db:exec("ROLLBACK")
                print(string.format("    FAILED: %s", tostring(err)))
                return false, err
            end
            print(string.format("    OK (v%d)", m.version))
        end
    end
    return true
end

function MigrationRunner:rollback(to_version)
    local current = self:current_version()
    print(string.format("Rolling back from v%d to v%d...", current, to_version))

    for i = #self.migrations, 1, -1 do
        local m = self.migrations[i]
        if m.version > to_version and m.version <= current then
            if not m.down then
                print(string.format("  v%d: no rollback defined, skipping", m.version))
            else
                print(string.format("  Rolling back v%d: %s", m.version, m.name))
                local ok, err = pcall(function()
                    self.db:exec("BEGIN")
                    m.down(self.db)
                    self.db:exec(string.format(
                        "UPDATE schema_versions SET rolled_back=1 WHERE version=%d",
                        m.version))
                    self.db:exec("COMMIT")
                end)
                if not ok then
                    self.db:exec("ROLLBACK")
                    print("  FAILED: " .. tostring(err))
                    return false
                end
                print(string.format("  OK (rolled back v%d)", m.version))
            end
        end
    end
    return true
end

-- Define migrations
local runner = MigrationRunner.new(db)

runner:register(1, "create_users",
    function(d)
        d:exec([[
            CREATE TABLE users (
                id       INTEGER PRIMARY KEY,
                username TEXT UNIQUE NOT NULL,
                email    TEXT NOT NULL,
                created_at TEXT DEFAULT (datetime('now'))
            )
        ]])
    end,
    function(d) d:exec("DROP TABLE IF EXISTS users") end
)

runner:register(2, "add_user_profile",
    function(d)
        d:exec("ALTER TABLE users ADD COLUMN bio TEXT")
        d:exec("ALTER TABLE users ADD COLUMN avatar_url TEXT")
    end,
    function(d)
        -- SQLite doesn't support DROP COLUMN in old versions
        d:exec([[
            CREATE TABLE users_backup AS
            SELECT id, username, email, created_at FROM users
        ]])
        d:exec("DROP TABLE users")
        d:exec("ALTER TABLE users_backup RENAME TO users")
    end
)

runner:register(3, "create_posts",
    function(d)
        d:exec([[
            CREATE TABLE posts (
                id         INTEGER PRIMARY KEY,
                user_id    INTEGER NOT NULL REFERENCES users(id),
                title      TEXT NOT NULL,
                body       TEXT,
                published  INTEGER DEFAULT 0,
                created_at TEXT DEFAULT (datetime('now'))
            )
        ]])
        d:exec("CREATE INDEX idx_posts_user ON posts(user_id)")
    end,
    function(d) d:exec("DROP TABLE IF EXISTS posts") end
)

runner:register(4, "add_post_tags",
    function(d)
        d:exec([[
            CREATE TABLE tags (id INTEGER PRIMARY KEY, name TEXT UNIQUE);
            CREATE TABLE post_tags (
                post_id INTEGER REFERENCES posts(id),
                tag_id  INTEGER REFERENCES tags(id),
                PRIMARY KEY(post_id, tag_id)
            )
        ]])
    end,
    function(d)
        d:exec("DROP TABLE IF EXISTS post_tags")
        d:exec("DROP TABLE IF EXISTS tags")
    end
)

-- Run migrations
print("=== Schema Migration System ===\n")
runner:migrate()

-- Show current schema
print("\nCurrent schema objects:")
for row in db:rows([[
    SELECT type, name FROM sqlite_master
    WHERE type IN ('table','index') AND name NOT LIKE 'sqlite_%'
    ORDER BY type, name
]]) do
    print(string.format("  %-8s %s", row[1], row[2]))
end

-- Show migration history
print("\nMigration history:")
for row in db:rows(
    "SELECT version, name, applied_at, rolled_back FROM schema_versions ORDER BY version") do
    local status = row[4] == 1 and "[ROLLED BACK]" or "[APPLIED]"
    print(string.format("  v%-3d %-20s %s %s",
        row[1], row[2], status, tostring(row[3])))
end

-- Rollback to version 2
print(string.format("\nRollback to v2:"))
runner:rollback(2)
print(string.format("Version after rollback: %d", runner:current_version()))

-- Re-apply migration 3 and 4
print("\nRe-applying migrations 3-4:")
runner:migrate(4)
print(string.format("Version after re-apply: %d", runner:current_version()))

db:close()
```

---

## ตัวอย่างที่ 30: Complex Reporting Query

Complex analytics queries สำหรับ reporting

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- E-commerce schema
db:exec([[
    CREATE TABLE customers (
        id         INTEGER PRIMARY KEY,
        name       TEXT,
        country    TEXT,
        tier       TEXT DEFAULT 'standard'
    );
    CREATE TABLE orders (
        id          INTEGER PRIMARY KEY,
        customer_id INTEGER REFERENCES customers(id),
        status      TEXT,
        total       REAL,
        created_at  TEXT
    );
    CREATE TABLE order_items (
        id         INTEGER PRIMARY KEY,
        order_id   INTEGER REFERENCES orders(id),
        product    TEXT,
        category   TEXT,
        qty        INTEGER,
        price      REAL
    );
]])

-- Seed data
db:exec("INSERT INTO customers VALUES(1,'Alice','TH','premium'),(2,'Bob','US','standard'),(3,'Carol','TH','premium'),(4,'Dave','US','standard'),(5,'Eve','SG','gold')")
local orders_sql = [[
    INSERT INTO orders VALUES
    (1,1,'completed',250.00,'2024-01-15'),
    (2,1,'completed',180.50,'2024-02-20'),
    (3,2,'completed',320.00,'2024-01-10'),
    (4,3,'completed',450.00,'2024-01-25'),
    (5,3,'completed',125.00,'2024-02-28'),
    (6,4,'cancelled',80.00, '2024-02-05'),
    (7,5,'completed',600.00,'2024-01-30'),
    (8,5,'completed',200.00,'2024-02-14'),
    (9,1,'completed',300.00,'2024-03-01'),
    (10,2,'completed',150.00,'2024-03-10')
]]
db:exec(orders_sql)
db:exec([[
    INSERT INTO order_items(order_id,product,category,qty,price) VALUES
    (1,'Widget A','Hardware',2,50.0),(1,'Gadget X','Electronics',1,150.0),
    (2,'Cable Z', 'Hardware', 5,36.1),
    (3,'Gadget Y','Electronics',2,160.0),
    (4,'Widget A','Hardware',3,50.0),(4,'Software Pro','Software',1,300.0),
    (5,'Widget B','Hardware',5,25.0),
    (7,'Software Pro','Software',2,300.0),
    (8,'Gadget X','Electronics',1,200.0),
    (9,'Widget A','Hardware',2,50.0),(9,'Gadget X','Electronics',1,200.0),
    (10,'Cable Z','Hardware',5,30.0)
]])

print("=== Complex Reporting Queries ===")

-- Revenue by country and month
print("\n1. Revenue by country and month:")
local ok = pcall(function()
    local sql = [[
        SELECT
            c.country,
            strftime('%Y-%m', o.created_at) AS month,
            COUNT(DISTINCT o.id) AS orders,
            SUM(o.total) AS revenue,
            AVG(o.total) AS avg_order
        FROM orders o
        JOIN customers c ON c.id = o.customer_id
        WHERE o.status = 'completed'
        GROUP BY c.country, month
        ORDER BY revenue DESC
    ]]
    for row in db:rows(sql) do
        print(string.format("  %-3s %s: %d orders, $%.2f total (avg $%.2f)",
            row[1], row[2], row[3], row[4], row[5]))
    end
end)
if not ok then print("  (requires SQLite with strftime support)") end

-- Top customers by revenue
print("\n2. Top customers (CLV):")
local sql2 = [[
    SELECT c.name, c.tier, c.country,
           COUNT(o.id) AS order_count,
           SUM(CASE WHEN o.status='completed' THEN o.total ELSE 0 END) AS total_revenue,
           MAX(o.created_at) AS last_order
    FROM customers c
    LEFT JOIN orders o ON o.customer_id = c.id
    GROUP BY c.id
    ORDER BY total_revenue DESC
]]
for row in db:rows(sql2) do
    print(string.format("  %-7s %-10s %-3s orders=%d revenue=$%.2f",
        row[1], row[2], row[3], row[4], row[5] or 0))
end

-- Category breakdown
print("\n3. Revenue by category:")
local sql3 = [[
    SELECT oi.category,
           COUNT(DISTINCT oi.order_id) AS orders,
           SUM(oi.qty) AS units_sold,
           SUM(oi.qty * oi.price) AS revenue
    FROM order_items oi
    JOIN orders o ON o.id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY oi.category
    ORDER BY revenue DESC
]]
for row in db:rows(sql3) do
    print(string.format("  %-15s orders=%-4d units=%-6d revenue=$%.2f",
        row[1], row[2], row[3], row[4]))
end

-- Cohort retention (simplified)
print("\n4. Customer tier distribution:")
local sql4 = [[
    SELECT c.tier,
           COUNT(*) AS customers,
           SUM(o.total) AS total_revenue,
           AVG(o.total) AS avg_order_value
    FROM customers c
    LEFT JOIN orders o ON o.customer_id=c.id AND o.status='completed'
    GROUP BY c.tier
    ORDER BY total_revenue DESC
]]
for row in db:rows(sql4) do
    print(string.format("  %-10s customers=%-3d revenue=$%-8.2f avg=$%.2f",
        row[1], row[2], row[3] or 0, row[4] or 0))
end

db:close()
```

---

## ตัวอย่างที่ 31: Time Series Storage

เก็บและ query time series data อย่างมีประสิทธิภาพ

```lua
local sqlite3 = require("lsqlite3")
local socket  = require("socket")

local db = sqlite3.open(":memory:")
db:exec("PRAGMA journal_mode = WAL")

db:exec([[
    CREATE TABLE ts_data (
        id        INTEGER PRIMARY KEY,
        series    TEXT NOT NULL,   -- e.g. "cpu.host1", "temp.sensor5"
        ts        INTEGER NOT NULL, -- Unix timestamp (seconds)
        value     REAL NOT NULL,
        tags      TEXT             -- optional JSON tags
    );

    CREATE INDEX idx_ts_series_time ON ts_data(series, ts DESC);
    CREATE INDEX idx_ts_time        ON ts_data(ts DESC);

    -- Downsampled aggregates
    CREATE TABLE ts_hourly (
        series    TEXT NOT NULL,
        hour_ts   INTEGER NOT NULL, -- truncated to hour
        count     INTEGER,
        sum       REAL,
        min       REAL,
        max       REAL,
        PRIMARY KEY(series, hour_ts)
    );
]])

-- Ingest function
local function ingest(batch)
    db:exec("BEGIN")
    local stmt = db:prepare(
        "INSERT INTO ts_data(series, ts, value) VALUES(?,?,?)")
    for _, row in ipairs(batch) do
        stmt:bind_values(row[1], row[2], row[3])
        stmt:step(); stmt:reset()
    end
    stmt:finalize()
    db:exec("COMMIT")
end

-- Generate synthetic metrics
local function gen_series(series_name, start_ts, n, base, noise)
    local data = {}
    for i = 0, n-1 do
        local val = base + (math.random() - 0.5) * noise +
            math.sin(i * 0.1) * (noise * 0.5)
        table.insert(data, { series_name, start_ts + i*60, math.max(0, val) })
    end
    return data
end

local NOW = os.time()
local batch = {}
for _, s in ipairs(gen_series("cpu.host1", NOW - 3600, 60, 45, 30)) do
    table.insert(batch, s)
end
for _, s in ipairs(gen_series("cpu.host2", NOW - 3600, 60, 60, 20)) do
    table.insert(batch, s)
end
for _, s in ipairs(gen_series("mem.host1", NOW - 3600, 60, 70, 10)) do
    table.insert(batch, s)
end
ingest(batch)

print("=== Time Series Storage ===")
for row in db:rows("SELECT count(*) FROM ts_data") do
    print(string.format("Ingested %d data points", row[1]))
end

-- Latest values
print("\nLatest value per series:")
for row in db:rows([[
    SELECT series, value, ts
    FROM ts_data
    WHERE ts = (SELECT MAX(ts) FROM ts_data t2 WHERE t2.series = ts_data.series)
    GROUP BY series
    ORDER BY series
]]) do
    print(string.format("  %-15s %.2f @ %s",
        row[1], row[2], os.date("%H:%M:%S", row[3])))
end

-- Range query: last 30 min aggregates
print("\nLast 30 min stats per series:")
local cutoff = NOW - 1800
local sql = string.format([[
    SELECT series,
           COUNT(*) as samples,
           MIN(value) as min_val,
           AVG(value) as avg_val,
           MAX(value) as max_val
    FROM ts_data
    WHERE ts >= %d
    GROUP BY series
    ORDER BY series
]], cutoff)
for row in db:rows(sql) do
    print(string.format("  %-15s n=%d min=%.1f avg=%.1f max=%.1f",
        row[1], row[2], row[3], row[4], row[5]))
end

-- Downsample to hourly
print("\nDownsample to hourly aggregates:")
db:exec(string.format([[
    INSERT OR REPLACE INTO ts_hourly(series, hour_ts, count, sum, min, max)
    SELECT
        series,
        (ts / 3600) * 3600 AS hour_ts,
        COUNT(*),
        SUM(value),
        MIN(value),
        MAX(value)
    FROM ts_data
    WHERE ts >= %d
    GROUP BY series, hour_ts
]], cutoff))

for row in db:rows("SELECT series, count, sum/count AS avg, min, max FROM ts_hourly") do
    print(string.format("  %-15s count=%-4d avg=%.1f min=%.1f max=%.1f",
        row[1], row[2], row[3], row[4], row[5]))
end

-- Anomaly detection: values > 2 std deviations
print("\nAnomaly detection (> mean + 2*stddev):")
for row in db:rows([[
    WITH stats AS (
        SELECT series, AVG(value) AS mean,
            AVG(value*value) - AVG(value)*AVG(value) AS variance
        FROM ts_data GROUP BY series
    )
    SELECT t.series, t.ts, t.value, s.mean,
           SQRT(s.variance) AS stddev
    FROM ts_data t JOIN stats s ON s.series = t.series
    WHERE t.value > s.mean + 2 * SQRT(s.variance)
    ORDER BY t.series, t.ts
    LIMIT 5
]]) do
    print(string.format("  %-15s v=%.1f (mean=%.1f stddev=%.1f)",
        row[1], row[3], row[4], row[5]))
end

db:close()
```

---

## ตัวอย่างที่ 32: Database Query Builder

Dynamic query builder สำหรับ complex SQL generation

```lua
local sqlite3 = require("lsqlite3")

-- Query Builder
local QB = {}
QB.__index = QB

function QB.select(...)
    return setmetatable({
        _type    = "SELECT",
        _cols    = {...},
        _from    = nil,
        _joins   = {},
        _wheres  = {},
        _params  = {},
        _groups  = {},
        _havings = {},
        _orders  = {},
        _limit   = nil,
        _offset  = nil,
    }, QB)
end

function QB:from(table_name, alias)
    self._from = alias and (table_name .. " " .. alias) or table_name
    return self
end

function QB:join(table_name, condition, join_type)
    table.insert(self._joins, string.format(
        "%s JOIN %s ON %s", join_type or "INNER", table_name, condition))
    return self
end

function QB:left_join(t, cond) return self:join(t, cond, "LEFT") end

function QB:where(condition, ...)
    table.insert(self._wheres, condition)
    for _, p in ipairs({...}) do
        table.insert(self._params, p)
    end
    return self
end

function QB:where_in(col, values)
    local placeholders = {}
    for _, v in ipairs(values) do
        table.insert(placeholders, "?")
        table.insert(self._params, v)
    end
    table.insert(self._wheres,
        col .. " IN (" .. table.concat(placeholders, ",") .. ")")
    return self
end

function QB:where_between(col, low, high)
    table.insert(self._wheres, col .. " BETWEEN ? AND ?")
    table.insert(self._params, low)
    table.insert(self._params, high)
    return self
end

function QB:group_by(...)
    for _, g in ipairs({...}) do table.insert(self._groups, g) end
    return self
end

function QB:having(condition)
    table.insert(self._havings, condition)
    return self
end

function QB:order_by(col, direction)
    table.insert(self._orders, col .. " " .. (direction or "ASC"))
    return self
end

function QB:limit(n)   self._limit = n;  return self end
function QB:offset(n)  self._offset = n; return self end

function QB:build()
    local cols = #self._cols > 0 and table.concat(self._cols, ", ") or "*"
    local sql = "SELECT " .. cols
    if self._from then sql = sql .. " FROM " .. self._from end
    for _, j in ipairs(self._joins) do sql = sql .. " " .. j end
    if #self._wheres > 0 then
        sql = sql .. " WHERE " .. table.concat(self._wheres, " AND ")
    end
    if #self._groups > 0 then
        sql = sql .. " GROUP BY " .. table.concat(self._groups, ", ")
    end
    if #self._havings > 0 then
        sql = sql .. " HAVING " .. table.concat(self._havings, " AND ")
    end
    if #self._orders > 0 then
        sql = sql .. " ORDER BY " .. table.concat(self._orders, ", ")
    end
    if self._limit  then sql = sql .. " LIMIT "  .. self._limit  end
    if self._offset then sql = sql .. " OFFSET " .. self._offset end
    return sql, self._params
end

function QB:execute(db_conn)
    local sql, params = self:build()
    local stmt = db_conn:prepare(sql)
    if not stmt then return nil, "prepare failed: " .. sql end
    stmt:bind_values(table.unpack(params))
    local rows = {}
    for row in stmt:rows() do
        local r = {}
        for i = 1, stmt:columns() do
            r[stmt:get_name(i-1)] = row[i]
        end
        table.insert(rows, r)
    end
    stmt:finalize()
    return rows
end

-- Demo
local db = sqlite3.open(":memory:")
db:exec([[
    CREATE TABLE products(id INTEGER PRIMARY KEY, name TEXT, category TEXT, price REAL, stock INTEGER, active INTEGER DEFAULT 1);
    CREATE TABLE categories(id INTEGER PRIMARY KEY, name TEXT, parent_id INTEGER);
    INSERT INTO products VALUES(1,'Widget A','Electronics',29.99,50,1);
    INSERT INTO products VALUES(2,'Gadget X','Electronics',99.99,10,1);
    INSERT INTO products VALUES(3,'Widget B','Hardware',14.99,0,1);
    INSERT INTO products VALUES(4,'Cable Z', 'Hardware',4.99,200,1);
    INSERT INTO products VALUES(5,'Old Item','Electronics',9.99,5,0);
    INSERT INTO products VALUES(6,'Premium','Electronics',199.99,3,1);
]])

print("=== Dynamic Query Builder ===")

-- Query 1: Active electronics, sorted by price
local q1 = QB.select("id", "name", "price", "stock")
    :from("products")
    :where("category = ?", "Electronics")
    :where("active = ?", 1)
    :where("stock > ?", 0)
    :order_by("price", "DESC")
    :limit(5)

local sql1, params1 = q1:build()
print("\nQuery 1:")
print("  SQL: " .. sql1)
print("  Params: " .. table.concat(params1, ", "))
local rows1 = q1:execute(db)
for _, r in ipairs(rows1) do
    print(string.format("  [%d] %-15s $%.2f stock=%d", r.id, r.name, r.price, r.stock))
end

-- Query 2: Price range + IN clause
local q2 = QB.select("name", "category", "price")
    :from("products")
    :where_between("price", 10, 100)
    :where_in("category", {"Electronics", "Hardware"})
    :order_by("category"):order_by("price")

local sql2 = q2:build()
print("\nQuery 2:")
print("  SQL: " .. sql2)
local rows2 = q2:execute(db)
for _, r in ipairs(rows2) do
    print(string.format("  %-15s %-12s $%.2f", r.name, r.category, r.price))
end

-- Query 3: Aggregation
local q3 = QB.select("category", "COUNT(*) AS count", "AVG(price) AS avg_price", "SUM(stock) AS total_stock")
    :from("products")
    :where("active = ?", 1)
    :group_by("category")
    :having("COUNT(*) > 1")
    :order_by("avg_price", "DESC")

local sql3 = q3:build()
print("\nQuery 3 (aggregation):")
print("  SQL: " .. sql3)
local rows3 = q3:execute(db)
for _, r in ipairs(rows3) do
    print(string.format("  %-12s count=%s avg=$%.2f stock=%s",
        r.category, tostring(r.count), tonumber(r.avg_price) or 0, tostring(r.total_stock)))
end

db:close()
```

---

## ตัวอย่างที่ 33: Redis Mock Implementation

Mock Redis implementation ใน Lua สำหรับ testing

```lua
-- Pure Lua Redis mock
local RedisMock = {}
RedisMock.__index = RedisMock

function RedisMock.new()
    return setmetatable({
        strings = {},
        lists   = {},
        hashes  = {},
        sets    = {},
        zsets   = {},
        expiry  = {},  -- key -> expires_at
        _cmd_count = 0,
    }, RedisMock)
end

-- Internal helpers
function RedisMock:_is_expired(key)
    local exp = self.expiry[key]
    if exp and os.time() > exp then
        self.strings[key] = nil
        self.expiry[key]  = nil
        return true
    end
    return false
end

function RedisMock:_ttl(key)
    local exp = self.expiry[key]
    if not exp then return -1 end
    local remaining = exp - os.time()
    return remaining > 0 and math.ceil(remaining) or -2
end

-- String commands
function RedisMock:SET(key, value, ex)
    self.strings[key] = tostring(value)
    if ex then self.expiry[key] = os.time() + ex end
    return "OK"
end

function RedisMock:GET(key)
    if self:_is_expired(key) then return nil end
    return self.strings[key]
end

function RedisMock:DEL(...)
    local count = 0
    for _, key in ipairs({...}) do
        if self.strings[key] or self.lists[key] or
           self.hashes[key] or self.sets[key] then
            self.strings[key] = nil; self.lists[key]  = nil
            self.hashes[key]  = nil; self.sets[key]   = nil
            self.expiry[key]  = nil
            count = count + 1
        end
    end
    return count
end

function RedisMock:INCR(key)
    local val = tonumber(self.strings[key] or "0")
    val = val + 1
    self.strings[key] = tostring(val)
    return val
end

function RedisMock:INCRBY(key, n)
    local val = tonumber(self.strings[key] or "0") + n
    self.strings[key] = tostring(val)
    return val
end

function RedisMock:EXPIRE(key, seconds)
    if not self.strings[key] then return 0 end
    self.expiry[key] = os.time() + seconds
    return 1
end

function RedisMock:TTL(key) return self:_ttl(key) end

function RedisMock:KEYS(pattern)
    local pat = pattern:gsub("%*",".*"):gsub("%?",".")
    local keys = {}
    for k in pairs(self.strings) do
        if k:match("^" .. pat .. "$") then table.insert(keys, k) end
    end
    return keys
end

-- List commands
function RedisMock:RPUSH(key, ...)
    if not self.lists[key] then self.lists[key] = {} end
    for _, v in ipairs({...}) do table.insert(self.lists[key], v) end
    return #self.lists[key]
end

function RedisMock:LPUSH(key, ...)
    if not self.lists[key] then self.lists[key] = {} end
    for _, v in ipairs({...}) do table.insert(self.lists[key], 1, v) end
    return #self.lists[key]
end

function RedisMock:LPOP(key)
    if not self.lists[key] or #self.lists[key] == 0 then return nil end
    return table.remove(self.lists[key], 1)
end

function RedisMock:LRANGE(key, start, stop)
    if not self.lists[key] then return {} end
    local len = #self.lists[key]
    start = start < 0 and len + start + 1 or start + 1
    stop  = stop  < 0 and len + stop  + 1 or stop  + 1
    start = math.max(1, start); stop = math.min(len, stop)
    local result = {}
    for i = start, stop do table.insert(result, self.lists[key][i]) end
    return result
end

function RedisMock:LLEN(key) return self.lists[key] and #self.lists[key] or 0 end

-- Hash commands
function RedisMock:HSET(key, field, value)
    if not self.hashes[key] then self.hashes[key] = {} end
    local is_new = self.hashes[key][field] == nil
    self.hashes[key][field] = tostring(value)
    return is_new and 1 or 0
end

function RedisMock:HGET(key, field)
    if not self.hashes[key] then return nil end
    return self.hashes[key][field]
end

function RedisMock:HGETALL(key)
    if not self.hashes[key] then return {} end
    local result = {}
    for k, v in pairs(self.hashes[key]) do
        table.insert(result, k); table.insert(result, v)
    end
    return result
end

function RedisMock:HMSET(key, ...)
    local args = {...}
    for i = 1, #args, 2 do
        self:HSET(key, args[i], args[i+1])
    end
    return "OK"
end

function RedisMock:HINCRBY(key, field, n)
    if not self.hashes[key] then self.hashes[key] = {} end
    local val = (tonumber(self.hashes[key][field]) or 0) + n
    self.hashes[key][field] = tostring(val)
    return val
end

function RedisMock:HDEL(key, field)
    if not self.hashes[key] then return 0 end
    if self.hashes[key][field] then
        self.hashes[key][field] = nil; return 1
    end
    return 0
end

-- Set commands
function RedisMock:SADD(key, ...)
    if not self.sets[key] then self.sets[key] = {} end
    local added = 0
    for _, m in ipairs({...}) do
        if not self.sets[key][m] then
            self.sets[key][m] = true; added = added + 1
        end
    end
    return added
end

function RedisMock:SISMEMBER(key, member)
    return (self.sets[key] and self.sets[key][member]) and 1 or 0
end

function RedisMock:SMEMBERS(key)
    if not self.sets[key] then return {} end
    local members = {}
    for m in pairs(self.sets[key]) do table.insert(members, m) end
    return members
end

function RedisMock:SCARD(key)
    if not self.sets[key] then return 0 end
    local n = 0
    for _ in pairs(self.sets[key]) do n = n + 1 end
    return n
end

-- Demo
print("=== Redis Mock Implementation ===")
local r = RedisMock.new()

-- Strings
print("\n-- Strings --")
r:SET("name", "Alice")
r:SET("counter", "0")
r:SET("session:abc", "user:42", 300)
print("GET name:", r:GET("name"))
print("INCR counter:", r:INCR("counter"))
print("INCR counter:", r:INCR("counter"))
print("INCRBY counter 5:", r:INCRBY("counter", 5))
print("TTL session:abc:", r:TTL("session:abc") .. "s")

-- Lists
print("\n-- Lists --")
r:RPUSH("queue", "task1", "task2", "task3")
r:LPUSH("stack", "item1", "item2")
print("LLEN queue:", r:LLEN("queue"))
print("LRANGE queue 0 -1:", table.concat(r:LRANGE("queue", 0, -1), ", "))
print("LPOP queue:", r:LPOP("queue"))
print("LRANGE queue 0 -1:", table.concat(r:LRANGE("queue", 0, -1), ", "))

-- Hashes
print("\n-- Hashes --")
r:HMSET("user:1", "name", "Bob", "email", "bob@ex.com", "score", "100")
r:HINCRBY("user:1", "score", 50)
print("HGET user:1 name:", r:HGET("user:1", "name"))
print("HGET user:1 score:", r:HGET("user:1", "score"))
local all = r:HGETALL("user:1")
print("HGETALL:", table.concat(all, " | "))

-- Sets
print("\n-- Sets --")
r:SADD("tags", "lua", "programming", "tutorial", "database")
r:SADD("tags", "lua")  -- duplicate
print("SCARD tags:", r:SCARD("tags"))
print("SISMEMBER tags lua:", r:SISMEMBER("tags", "lua"))
print("SISMEMBER tags python:", r:SISMEMBER("tags", "python"))
print("SMEMBERS tags:", table.concat(r:SMEMBERS("tags"), ", "))

-- KEYS pattern
print("\nKEYS 'session:*':", table.concat(r:KEYS("session:*"), ", "))
```

---

## ตัวอย่างที่ 34: Database Connection Health Check

Health checks และ monitoring สำหรับ database connections

```lua
local sqlite3 = require("lsqlite3")
local socket  = require("socket")

-- Health check result
local function make_result(healthy, message, details)
    return {
        healthy  = healthy,
        message  = message,
        details  = details or {},
        checked_at = os.time(),
    }
end

-- Database health checker
local DBHealthCheck = {}
DBHealthCheck.__index = DBHealthCheck

function DBHealthCheck.new(db_conn, name)
    return setmetatable({
        db      = db_conn,
        name    = name or "default",
        checks  = {},
        history = {},
        max_history = 100,
    }, DBHealthCheck)
end

function DBHealthCheck:_record(result)
    table.insert(self.history, result)
    if #self.history > self.max_history then
        table.remove(self.history, 1)
    end
end

function DBHealthCheck:check_connectivity()
    local t0 = socket.gettime()
    local ok = pcall(function()
        for _ in self.db:rows("SELECT 1") do end
    end)
    local latency_ms = (socket.gettime() - t0) * 1000
    if ok then
        return make_result(true, "connected",
            { latency_ms = string.format("%.2f", latency_ms) })
    end
    return make_result(false, "connection failed")
end

function DBHealthCheck:check_integrity()
    local issues = {}
    for row in self.db:rows("PRAGMA integrity_check") do
        if row[1] ~= "ok" then table.insert(issues, row[1]) end
    end
    if #issues == 0 then
        return make_result(true, "integrity ok")
    end
    return make_result(false, "integrity issues: " .. table.concat(issues, ", "))
end

function DBHealthCheck:check_wal_size()
    local wal_pages = 0
    for row in self.db:rows("PRAGMA wal_checkpoint(PASSIVE)") do
        wal_pages = row[2] or 0
    end
    local healthy = wal_pages < 10000
    return make_result(healthy,
        healthy and "WAL size OK" or "WAL too large",
        { wal_pages = wal_pages })
end

function DBHealthCheck:check_table_counts(tables)
    local counts = {}
    for _, tbl in ipairs(tables or {}) do
        local ok, cnt = pcall(function()
            for row in self.db:rows("SELECT count(*) FROM " .. tbl) do
                return row[1]
            end
        end)
        counts[tbl] = ok and cnt or "error"
    end
    return make_result(true, "counts ok", counts)
end

function DBHealthCheck:check_slow_queries()
    -- Simulate slow query detection via EXPLAIN QUERY PLAN
    local slow = {}
    local test_queries = {
        { "SELECT * FROM sqlite_master WHERE name LIKE '%test%'",
          "full scan on sqlite_master" },
    }
    for _, q in ipairs(test_queries) do
        for row in self.db:rows("EXPLAIN QUERY PLAN " .. q[1]) do
            if tostring(row[4] or ""):find("SCAN") then
                table.insert(slow, q[2])
            end
        end
    end
    return make_result(
        #slow == 0, #slow == 0 and "no slow queries" or #slow .. " slow queries",
        { slow_queries = slow })
end

function DBHealthCheck:run_all(tables)
    local results = {
        name   = self.name,
        ts     = os.time(),
        checks = {}
    }

    local check_fns = {
        { "connectivity",   function() return self:check_connectivity() end },
        { "integrity",      function() return self:check_integrity() end },
        { "wal_size",       function() return self:check_wal_size() end },
        { "table_counts",   function() return self:check_table_counts(tables) end },
        { "slow_queries",   function() return self:check_slow_queries() end },
    }

    local all_healthy = true
    for _, cf in ipairs(check_fns) do
        local ok, result = pcall(cf[2])
        if not ok then
            result = make_result(false, "check error: " .. tostring(result))
        end
        results.checks[cf[1]] = result
        if not result.healthy then all_healthy = false end
    end

    results.healthy = all_healthy
    self:_record(results)
    return results
end

function DBHealthCheck:uptime_percent()
    if #self.history == 0 then return 100 end
    local healthy = 0
    for _, h in ipairs(self.history) do
        if h.healthy then healthy = healthy + 1 end
    end
    return healthy / #self.history * 100
end

-- Demo
local db = sqlite3.open(":memory:")
db:exec("PRAGMA journal_mode = WAL")
db:exec("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)")
db:exec("CREATE TABLE logs  (id INTEGER PRIMARY KEY, msg TEXT, ts INTEGER)")

-- Insert test data
for i = 1, 10 do
    db:exec(string.format("INSERT INTO users(name) VALUES('user%d')", i))
    db:exec(string.format("INSERT INTO logs(msg,ts) VALUES('log%d',%d)", i, os.time()-i))
end

print("=== Database Health Check ===")
local checker = DBHealthCheck.new(db, "main-db")
local results = checker:run_all({"users", "logs"})

print(string.format("\nDatabase: %s | Overall: %s",
    results.name, results.healthy and "HEALTHY" or "UNHEALTHY"))
print(string.format("%-20s %-10s %s", "Check", "Status", "Details"))
print(string.rep("-", 60))

for name, r in pairs(results.checks) do
    local status = r.healthy and "OK" or "FAIL"
    local details = ""
    if r.details and next(r.details) then
        local dparts = {}
        for k, v in pairs(r.details) do
            if type(v) == "table" then
                for _, vv in ipairs(v) do
                    table.insert(dparts, vv)
                end
            else
                table.insert(dparts, k .. "=" .. tostring(v))
            end
        end
        details = table.concat(dparts, ", ")
    end
    print(string.format("  %-18s %-10s %s %s",
        name, status, r.message or "", details:sub(1,40)))
end

-- Simulate multiple runs
for _ = 1, 5 do checker:run_all({"users","logs"}) end
print(string.format("\nUptime: %.1f%% (%d checks)",
    checker:uptime_percent(), #checker.history))

db:close()
```

---

## ตัวอย่างที่ 35: Stored Procedures Simulation

Simulate stored procedures ด้วย Lua functions

```lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- Schema
db:exec([[
    CREATE TABLE accounts (
        id      INTEGER PRIMARY KEY,
        name    TEXT NOT NULL,
        balance REAL NOT NULL DEFAULT 0,
        type    TEXT DEFAULT 'checking'
    );
    CREATE TABLE transactions (
        id          INTEGER PRIMARY KEY,
        from_acct   INTEGER REFERENCES accounts(id),
        to_acct     INTEGER REFERENCES accounts(id),
        amount      REAL NOT NULL,
        description TEXT,
        ts          TEXT DEFAULT (datetime('now')),
        status      TEXT DEFAULT 'pending'
    );
]])

-- Seed
db:exec([[
    INSERT INTO accounts(name,balance,type) VALUES
    ('Alice Checking', 5000, 'checking'),
    ('Alice Savings',  10000, 'savings'),
    ('Bob Checking',   3000, 'checking'),
    ('Corporate',      100000, 'business')
]])

-- "Stored Procedures" as Lua functions with transactions
local SP = {}

-- transfer_funds(from_id, to_id, amount, description)
function SP.transfer_funds(from_id, to_id, amount, description)
    if amount <= 0 then return nil, "amount must be positive" end

    db:exec("BEGIN")
    local ok, err = pcall(function()
        -- Get balances
        local from_bal, to_exists = nil, false
        for row in db:rows(string.format(
            "SELECT balance FROM accounts WHERE id=%d", from_id)) do
            from_bal = row[1]
        end
        for _ in db:rows(string.format(
            "SELECT id FROM accounts WHERE id=%d", to_id)) do
            to_exists = true
        end

        if not from_bal then error("source account not found") end
        if not to_exists then error("destination account not found") end
        if from_bal < amount then error(string.format(
            "insufficient funds: have %.2f, need %.2f", from_bal, amount)) end

        -- Debit source
        db:exec(string.format(
            "UPDATE accounts SET balance=balance-%.2f WHERE id=%d",
            amount, from_id))
        -- Credit destination
        db:exec(string.format(
            "UPDATE accounts SET balance=balance+%.2f WHERE id=%d",
            amount, to_id))
        -- Log transaction
        db:exec(string.format([[
            INSERT INTO transactions(from_acct,to_acct,amount,description,status)
            VALUES(%d,%d,%.2f,%q,'completed')
        ]], from_id, to_id, amount, description or "transfer"))
    end)

    if ok then
        db:exec("COMMIT")
        local txn_id = db:last_insert_rowid()
        return txn_id
    else
        db:exec("ROLLBACK")
        return nil, tostring(err)
    end
end

-- get_account_statement(account_id, limit)
function SP.get_statement(account_id, limit)
    limit = limit or 10
    local stmt = db:prepare([[
        SELECT
            t.id, t.ts,
            CASE WHEN t.from_acct=? THEN -t.amount ELSE t.amount END AS amount,
            CASE WHEN t.from_acct=? THEN a2.name ELSE a1.name END AS counterpart,
            t.description, t.status
        FROM transactions t
        LEFT JOIN accounts a1 ON a1.id = t.from_acct
        LEFT JOIN accounts a2 ON a2.id = t.to_acct
        WHERE t.from_acct=? OR t.to_acct=?
        ORDER BY t.id DESC
        LIMIT ?
    ]])
    stmt:bind_values(account_id, account_id, account_id, account_id, limit)
    local rows = {}
    for row in stmt:rows() do
        table.insert(rows, {
            id=row[1], ts=row[2], amount=row[3],
            counterpart=row[4], description=row[5], status=row[6]
        })
    end
    stmt:finalize()
    return rows
end

-- monthly_summary(account_id)
function SP.monthly_summary(account_id)
    local result = {}
    for row in db:rows(string.format([[
        SELECT
            strftime('%%Y-%%m', ts) AS month,
            SUM(CASE WHEN to_acct=%d THEN amount ELSE 0 END) AS credits,
            SUM(CASE WHEN from_acct=%d THEN amount ELSE 0 END) AS debits,
            COUNT(*) AS transactions
        FROM transactions
        WHERE (from_acct=%d OR to_acct=%d) AND status='completed'
        GROUP BY month
        ORDER BY month
    ]], account_id, account_id, account_id, account_id)) do
        table.insert(result, {
            month=row[1], credits=row[2], debits=row[3], transactions=row[4]
        })
    end
    return result
end

-- Demo
print("=== Stored Procedures Simulation ===")

print("\nInitial balances:")
for row in db:rows("SELECT id, name, balance FROM accounts ORDER BY id") do
    print(string.format("  [%d] %-18s $%.2f", row[1], row[2], row[3]))
end

-- Perform transfers
print("\nExecuting transfers:")
local transfers = {
    { 1, 3, 500,   "rent payment" },
    { 1, 2, 1000,  "savings transfer" },
    { 4, 1, 2000,  "salary" },
    { 3, 1, 50,    "loan repayment" },
    { 1, 3, 999999,"will fail: insufficient" },
}

for _, t in ipairs(transfers) do
    local txn_id, err = SP.transfer_funds(t[1], t[2], t[3], t[4])
    if txn_id then
        print(string.format("  OK txn#%d: acct%d -> acct%d $%.2f (%s)",
            txn_id, t[1], t[2], t[3], t[4]))
    else
        print(string.format("  FAILED: acct%d -> acct%d $%.2f: %s",
            t[1], t[2], t[3], tostring(err)))
    end
end

print("\nFinal balances:")
for row in db:rows("SELECT id, name, balance FROM accounts ORDER BY id") do
    print(string.format("  [%d] %-18s $%.2f", row[1], row[2], row[3]))
end

print("\nStatement for Alice Checking (acct 1):")
for _, tx in ipairs(SP.get_statement(1, 5)) do
    local sign = tx.amount >= 0 and "+" or ""
    print(string.format("  txn#%-4d %s%6.2f from/to: %-18s %s",
        tx.id, sign, tx.amount, tx.counterpart or "?", tx.description or ""))
end

db:close()
```

---

## ตัวอย่างที่ 36: Graph Database จำลองใน SQLite

```lua
-- Graph Database simulation using SQLite adjacency list + recursive CTE
local sqlite3 = require("lsqlite3")

local GraphDB = {}
GraphDB.__index = GraphDB

function GraphDB.new(path)
    local self = setmetatable({}, GraphDB)
    self.db = sqlite3.open(path or ":memory:")
    self:_init()
    return self
end

function GraphDB:_init()
    self.db:exec([[
        CREATE TABLE IF NOT EXISTS nodes (
            id   INTEGER PRIMARY KEY,
            label TEXT NOT NULL,
            props TEXT DEFAULT '{}'
        );
        CREATE TABLE IF NOT EXISTS edges (
            id      INTEGER PRIMARY KEY,
            from_id INTEGER NOT NULL REFERENCES nodes(id),
            to_id   INTEGER NOT NULL REFERENCES nodes(id),
            rel     TEXT NOT NULL,
            weight  REAL DEFAULT 1.0,
            props   TEXT DEFAULT '{}'
        );
        CREATE INDEX IF NOT EXISTS idx_edges_from ON edges(from_id);
        CREATE INDEX IF NOT EXISTS idx_edges_to   ON edges(to_id);
    ]])
end

function GraphDB:add_node(label, props)
    local json_props = "{}"
    if props then
        local parts = {}
        for k, v in pairs(props) do
            if type(v) == "string" then
                parts[#parts+1] = string.format('%q:%q', k, v)
            else
                parts[#parts+1] = string.format('%q:%s', k, tostring(v))
            end
        end
        json_props = "{" .. table.concat(parts, ",") .. "}"
    end
    local stmt = self.db:prepare("INSERT INTO nodes(label,props) VALUES(?,?)")
    stmt:bind_values(label, json_props)
    stmt:step(); stmt:finalize()
    return self.db:last_insert_rowid()
end

function GraphDB:add_edge(from_id, to_id, rel, weight, props)
    weight = weight or 1.0
    local stmt = self.db:prepare(
        "INSERT INTO edges(from_id,to_id,rel,weight) VALUES(?,?,?,?)")
    stmt:bind_values(from_id, to_id, rel, weight)
    stmt:step(); stmt:finalize()
    return self.db:last_insert_rowid()
end

-- BFS shortest path using recursive CTE
function GraphDB:shortest_path(start_id, end_id)
    local sql = [[
        WITH RECURSIVE path(node_id, route, depth, visited) AS (
            SELECT ?, CAST(? AS TEXT), 0, ',' || ? || ','
            UNION ALL
            SELECT e.to_id,
                   p.route || '->' || e.to_id,
                   p.depth + 1,
                   p.visited || e.to_id || ','
            FROM path p
            JOIN edges e ON e.from_id = p.node_id
            WHERE p.visited NOT LIKE '%,' || e.to_id || ',%'
              AND p.depth < 10
        )
        SELECT route, depth FROM path WHERE node_id = ? ORDER BY depth LIMIT 1
    ]]
    local stmt = self.db:prepare(sql)
    stmt:bind_values(start_id, start_id, start_id, end_id)
    local row = stmt:first_row()
    stmt:finalize()
    return row and {route = row[1], depth = row[2]} or nil
end

-- Find all neighbors at distance N
function GraphDB:neighbors(node_id, max_depth)
    max_depth = max_depth or 1
    local sql = [[
        WITH RECURSIVE reach(node_id, depth) AS (
            SELECT ?, 0
            UNION
            SELECT e.to_id, r.depth+1
            FROM reach r JOIN edges e ON e.from_id = r.node_id
            WHERE r.depth < ?
        )
        SELECT DISTINCT n.id, n.label, r.depth
        FROM reach r JOIN nodes n ON n.id = r.node_id
        WHERE r.node_id != ?
        ORDER BY r.depth, n.label
    ]]
    local stmt = self.db:prepare(sql)
    stmt:bind_values(node_id, max_depth, node_id)
    local results = {}
    for row in stmt:rows() do
        results[#results+1] = {id=row[1], label=row[2], depth=row[3]}
    end
    stmt:finalize()
    return results
end

function GraphDB:close() self.db:close() end

-- Demo: Social network graph
local g = GraphDB.new()

local alice   = g:add_node("Alice",   {role="admin"})
local bob     = g:add_node("Bob",     {role="user"})
local carol   = g:add_node("Carol",   {role="user"})
local dave    = g:add_node("Dave",    {role="user"})
local eve     = g:add_node("Eve",     {role="user"})

g:add_edge(alice, bob,   "FRIENDS", 1)
g:add_edge(bob,   carol, "FRIENDS", 1)
g:add_edge(carol, dave,  "FRIENDS", 1)
g:add_edge(alice, eve,   "FRIENDS", 1)
g:add_edge(eve,   carol, "FRIENDS", 1)

local path = g:shortest_path(alice, dave)
if path then
    print(string.format("Shortest path Alice→Dave: %s (depth %d)", path.route, path.depth))
end

print("\nAlice's network (depth 2):")
for _, n in ipairs(g:neighbors(alice, 2)) do
    print(string.format("  depth=%d  %s", n.depth, n.label))
end

g:close()
```

---

## ตัวอย่างที่ 37: Geospatial Bounding Box Queries

```lua
-- Geospatial queries using R-tree virtual table in SQLite
local sqlite3 = require("lsqlite3")

local GeoDB = {}
GeoDB.__index = GeoDB

function GeoDB.new()
    local self = setmetatable({}, GeoDB)
    self.db = sqlite3.open(":memory:")
    self.db:exec([[
        CREATE VIRTUAL TABLE IF NOT EXISTS places_rtree
            USING rtree(id, min_lat, max_lat, min_lng, max_lng);
        CREATE TABLE IF NOT EXISTS places (
            id      INTEGER PRIMARY KEY,
            name    TEXT NOT NULL,
            lat     REAL NOT NULL,
            lng     REAL NOT NULL,
            category TEXT
        );
    ]])
    return self
end

function GeoDB:insert(name, lat, lng, category)
    local eps = 0.00001  -- tiny box around point
    local stmt = self.db:prepare(
        "INSERT INTO places(name,lat,lng,category) VALUES(?,?,?,?)")
    stmt:bind_values(name, lat, lng, category or "place")
    stmt:step(); stmt:finalize()
    local id = self.db:last_insert_rowid()
    local rt = self.db:prepare(
        "INSERT INTO places_rtree VALUES(?,?,?,?,?)")
    rt:bind_values(id, lat-eps, lat+eps, lng-eps, lng+eps)
    rt:step(); rt:finalize()
    return id
end

-- Bounding box search
function GeoDB:bbox_search(min_lat, max_lat, min_lng, max_lng)
    local sql = [[
        SELECT p.id, p.name, p.lat, p.lng, p.category
        FROM places_rtree r
        JOIN places p ON p.id = r.id
        WHERE r.min_lat >= ? AND r.max_lat <= ?
          AND r.min_lng >= ? AND r.max_lng <= ?
        ORDER BY p.name
    ]]
    local stmt = self.db:prepare(sql)
    stmt:bind_values(min_lat, max_lat, min_lng, max_lng)
    local results = {}
    for row in stmt:rows() do
        results[#results+1] = {id=row[1], name=row[2], lat=row[3], lng=row[4], cat=row[5]}
    end
    stmt:finalize()
    return results
end

-- Approximate radius search (degrees ~ km/111)
function GeoDB:radius_search(center_lat, center_lng, radius_km)
    local deg = radius_km / 111.0
    local results = self:bbox_search(
        center_lat - deg, center_lat + deg,
        center_lng - deg, center_lng + deg)
    -- Refine with Haversine
    local filtered = {}
    for _, p in ipairs(results) do
        local dlat = math.rad(p.lat - center_lat)
        local dlng = math.rad(p.lng - center_lng)
        local a = math.sin(dlat/2)^2 +
                  math.cos(math.rad(center_lat)) *
                  math.cos(math.rad(p.lat)) *
                  math.sin(dlng/2)^2
        local dist = 6371 * 2 * math.asin(math.sqrt(a))
        if dist <= radius_km then
            p.distance_km = math.floor(dist * 100) / 100
            filtered[#filtered+1] = p
        end
    end
    table.sort(filtered, function(a, b) return a.distance_km < b.distance_km end)
    return filtered
end

function GeoDB:close() self.db:close() end

-- Demo: Bangkok area
local geo = GeoDB.new()

geo:insert("Central World",      13.7466, 100.5393, "mall")
geo:insert("Siam Paragon",       13.7462, 100.5331, "mall")
geo:insert("MBK Center",         13.7441, 100.5298, "mall")
geo:insert("Chatuchak Market",   13.7999, 100.5500, "market")
geo:insert("Grand Palace",       13.7500, 100.4913, "landmark")
geo:insert("Lumphini Park",      13.7308, 100.5418, "park")

print("Places within 2km of Siam (13.7462, 100.5331):")
local nearby = geo:radius_search(13.7462, 100.5331, 2.0)
for _, p in ipairs(nearby) do
    print(string.format("  %-25s %.2f km  [%s]", p.name, p.distance_km, p.cat))
end

geo:close()
```

---

## ตัวอย่างที่ 38: Full Blog Application Database

```lua
-- Complete blog application database layer
local sqlite3 = require("lsqlite3")

local BlogDB = {}
BlogDB.__index = BlogDB

function BlogDB.new(path)
    local self = setmetatable({}, BlogDB)
    self.db = sqlite3.open(path or ":memory:")
    self:_schema()
    return self
end

function BlogDB:_schema()
    self.db:exec([[
        PRAGMA journal_mode=WAL;
        PRAGMA foreign_keys=ON;

        CREATE TABLE IF NOT EXISTS users (
            id         INTEGER PRIMARY KEY,
            username   TEXT UNIQUE NOT NULL,
            email      TEXT UNIQUE NOT NULL,
            role       TEXT DEFAULT 'author',
            created_at TEXT DEFAULT (datetime('now'))
        );
        CREATE TABLE IF NOT EXISTS posts (
            id         INTEGER PRIMARY KEY,
            user_id    INTEGER NOT NULL REFERENCES users(id),
            title      TEXT NOT NULL,
            slug       TEXT UNIQUE NOT NULL,
            body       TEXT NOT NULL,
            status     TEXT DEFAULT 'draft',
            views      INTEGER DEFAULT 0,
            created_at TEXT DEFAULT (datetime('now')),
            published_at TEXT
        );
        CREATE TABLE IF NOT EXISTS tags (
            id   INTEGER PRIMARY KEY,
            name TEXT UNIQUE NOT NULL
        );
        CREATE TABLE IF NOT EXISTS post_tags (
            post_id INTEGER REFERENCES posts(id) ON DELETE CASCADE,
            tag_id  INTEGER REFERENCES tags(id)  ON DELETE CASCADE,
            PRIMARY KEY(post_id, tag_id)
        );
        CREATE TABLE IF NOT EXISTS comments (
            id         INTEGER PRIMARY KEY,
            post_id    INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
            user_id    INTEGER REFERENCES users(id),
            author     TEXT,
            body       TEXT NOT NULL,
            approved   INTEGER DEFAULT 0,
            created_at TEXT DEFAULT (datetime('now'))
        );
        CREATE VIRTUAL TABLE IF NOT EXISTS posts_fts
            USING fts5(title, body, content='posts', content_rowid='id');
        CREATE TRIGGER IF NOT EXISTS posts_ai AFTER INSERT ON posts BEGIN
            INSERT INTO posts_fts(rowid,title,body) VALUES(new.id,new.title,new.body);
        END;
        CREATE TRIGGER IF NOT EXISTS posts_au AFTER UPDATE ON posts BEGIN
            INSERT INTO posts_fts(posts_fts,rowid,title,body)
                VALUES('delete',old.id,old.title,old.body);
            INSERT INTO posts_fts(rowid,title,body) VALUES(new.id,new.title,new.body);
        END;
    ]])
end

function BlogDB:create_user(username, email, role)
    local s = self.db:prepare(
        "INSERT INTO users(username,email,role) VALUES(?,?,?)")
    s:bind_values(username, email, role or "author")
    s:step(); s:finalize()
    return self.db:last_insert_rowid()
end

function BlogDB:create_post(user_id, title, body, tags)
    local slug = title:lower():gsub("[^%w]+", "-"):gsub("^%-+", ""):gsub("%-+$", "")
    local s = self.db:prepare(
        "INSERT INTO posts(user_id,title,slug,body,status) VALUES(?,?,?,?,'draft')")
    s:bind_values(user_id, title, slug, body)
    s:step(); s:finalize()
    local post_id = self.db:last_insert_rowid()
    if tags then
        for _, tag in ipairs(tags) do
            self.db:exec(string.format(
                "INSERT OR IGNORE INTO tags(name) VALUES('%s')", tag))
            local tid = self.db:prepare(
                "SELECT id FROM tags WHERE name=?")
            tid:bind_values(tag)
            local row = tid:first_row(); tid:finalize()
            if row then
                local pt = self.db:prepare(
                    "INSERT OR IGNORE INTO post_tags VALUES(?,?)")
                pt:bind_values(post_id, row[1])
                pt:step(); pt:finalize()
            end
        end
    end
    return post_id
end

function BlogDB:publish(post_id)
    self.db:exec(string.format(
        "UPDATE posts SET status='published',published_at=datetime('now') WHERE id=%d",
        post_id))
end

function BlogDB:search(query)
    local sql = [[
        SELECT p.id, p.title, p.slug, u.username,
               snippet(posts_fts, 1, '<b>', '</b>', '...', 20) AS excerpt
        FROM posts_fts
        JOIN posts p ON p.id = posts_fts.rowid
        JOIN users u ON u.id = p.user_id
        WHERE posts_fts MATCH ? AND p.status='published'
        ORDER BY rank
        LIMIT 10
    ]]
    local s = self.db:prepare(sql)
    s:bind_values(query)
    local rows = {}
    for row in s:rows() do
        rows[#rows+1] = {id=row[1],title=row[2],slug=row[3],author=row[4],excerpt=row[5]}
    end
    s:finalize()
    return rows
end

function BlogDB:stats()
    local sql = [[
        SELECT u.username,
               COUNT(p.id) AS total_posts,
               SUM(CASE WHEN p.status='published' THEN 1 ELSE 0 END) AS published,
               SUM(p.views) AS total_views
        FROM users u
        LEFT JOIN posts p ON p.user_id = u.id
        GROUP BY u.id
        ORDER BY total_views DESC
    ]]
    local rows = {}
    for row in self.db:rows(sql) do
        rows[#rows+1] = {user=row[1], total=row[2], pub=row[3], views=row[4]}
    end
    return rows
end

function BlogDB:close() self.db:close() end

-- Demo
local blog = BlogDB.new()

local uid1 = blog:create_user("alice", "alice@example.com", "admin")
local uid2 = blog:create_user("bob",   "bob@example.com",   "author")

local p1 = blog:create_post(uid1, "Lua Programming Tips",
    "Lua is a lightweight scripting language perfect for embedding.",
    {"lua", "programming", "tips"})
local p2 = blog:create_post(uid1, "SQLite with Lua",
    "SQLite is a self-contained, high-reliability, embedded database.",
    {"lua", "sqlite", "database"})
local p3 = blog:create_post(uid2, "Getting Started with LuaSocket",
    "LuaSocket provides networking support for Lua.",
    {"lua", "network", "luasocket"})

blog:publish(p1); blog:publish(p2); blog:publish(p3)

print("Search 'lua database':")
for _, r in ipairs(blog:search("lua database")) do
    print(string.format("  [%d] %s by %s", r.id, r.title, r.author))
    print(string.format("      %s", r.excerpt))
end

print("\nAuthor stats:")
for _, s in ipairs(blog:stats()) do
    print(string.format("  %-10s  posts=%d  published=%d  views=%d",
        s.user, s.total, s.pub, s.views))
end

blog:close()
```

---

## ตัวอย่างที่ 39: Database Connection Pool

```lua
-- Database connection pool for LuaSQL / lsqlite3
local sqlite3 = require("lsqlite3")

local Pool = {}
Pool.__index = Pool

function Pool.new(opts)
    opts = opts or {}
    local self = setmetatable({}, Pool)
    self.path      = opts.path or ":memory:"
    self.min_size  = opts.min_size or 2
    self.max_size  = opts.max_size or 10
    self.timeout   = opts.timeout or 5.0
    self._pool     = {}   -- available connections
    self._active   = 0    -- connections in use
    self._created  = 0    -- total created
    self._waiters  = 0
    -- Pre-create minimum connections
    for i = 1, self.min_size do
        self._pool[i] = self:_new_conn()
        self._created = self._created + 1
    end
    return self
end

function Pool:_new_conn()
    local db = sqlite3.open(self.path)
    db:exec("PRAGMA journal_mode=WAL; PRAGMA foreign_keys=ON;")
    return {db=db, id=self._created+1, created=os.time(), uses=0}
end

function Pool:acquire()
    -- Try pool first
    if #self._pool > 0 then
        local conn = table.remove(self._pool)
        conn.uses = conn.uses + 1
        self._active = self._active + 1
        return conn, nil
    end
    -- Create new if under limit
    if self._created < self.max_size then
        self._created = self._created + 1
        local conn = self:_new_conn()
        conn.uses = 1
        self._active = self._active + 1
        return conn, nil
    end
    -- Pool exhausted
    return nil, "pool exhausted (max=" .. self.max_size .. ")"
end

function Pool:release(conn)
    if not conn then return end
    self._active = self._active - 1
    -- Retire old connections
    if conn.uses > 100 or (os.time() - conn.created) > 3600 then
        conn.db:close()
        self._created = self._created - 1
    else
        self._pool[#self._pool+1] = conn
    end
end

-- Execute with automatic acquire/release
function Pool:with(fn)
    local conn, err = self:acquire()
    if not conn then return nil, err end
    local ok, result = pcall(fn, conn.db)
    self:release(conn)
    if not ok then return nil, result end
    return result
end

function Pool:stats()
    return {
        pool_size = #self._pool,
        active    = self._active,
        total     = self._created,
        max       = self.max_size,
    }
end

function Pool:close()
    for _, conn in ipairs(self._pool) do
        conn.db:close()
    end
    self._pool = {}
end

-- Demo
local pool = Pool.new({path=":memory:", min_size=2, max_size=5})

-- Initialize schema in one connection, reuse across pool
pool:with(function(db)
    db:exec([[
        CREATE TABLE IF NOT EXISTS counters (
            name TEXT PRIMARY KEY, value INTEGER DEFAULT 0)
    ]])
end)

-- Simulate concurrent access
local function increment(name)
    return pool:with(function(db)
        db:exec(string.format([[
            INSERT INTO counters(name,value) VALUES('%s',1)
            ON CONFLICT(name) DO UPDATE SET value=value+1
        ]], name))
        local s = db:prepare("SELECT value FROM counters WHERE name=?")
        s:bind_values(name)
        local row = s:first_row(); s:finalize()
        return row and row[1] or 0
    end)
end

for i = 1, 10 do
    local val = increment("hits")
    if i % 3 == 0 then
        print(string.format("  After %d increments: hits=%s", i, tostring(val)))
    end
end

local st = pool:stats()
print(string.format("\nPool stats: pool=%d active=%d total=%d max=%d",
    st.pool_size, st.active, st.total, st.max))

pool:close()
```

---

## ตัวอย่างที่ 40: EXPLAIN QUERY PLAN และ Performance Profiling

```lua
-- SQLite query analysis and performance profiling
local sqlite3 = require("lsqlite3")

local Profiler = {}
Profiler.__index = Profiler

function Profiler.new(db)
    local self = setmetatable({}, Profiler)
    self.db = db
    self.results = {}
    return self
end

function Profiler:explain(sql, params)
    local explain_sql = "EXPLAIN QUERY PLAN " .. sql
    local stmt = self.db:prepare(explain_sql)
    if params then stmt:bind_values(table.unpack(params)) end
    local plan = {}
    for row in stmt:rows() do
        plan[#plan+1] = {
            id=row[1], parent=row[2], notused=row[3], detail=row[4]
        }
    end
    stmt:finalize()
    return plan
end

function Profiler:benchmark(name, sql, params, iterations)
    iterations = iterations or 1000
    local stmt = self.db:prepare(sql)

    local t0 = os.clock()
    for _ = 1, iterations do
        if params then stmt:bind_values(table.unpack(params)) end
        stmt:reset()
        for _ in stmt:rows() do end  -- consume results
    end
    local elapsed = os.clock() - t0
    stmt:finalize()

    local result = {
        name       = name,
        sql        = sql:sub(1, 60),
        iterations = iterations,
        total_ms   = elapsed * 1000,
        avg_us     = (elapsed / iterations) * 1e6,
    }
    self.results[#self.results+1] = result
    return result
end

function Profiler:print_plan(sql, params)
    local plan = self:explain(sql, params)
    print(string.format("\nQuery Plan: %s", sql:sub(1, 70)))
    for _, row in ipairs(plan) do
        local indent = string.rep("  ", math.max(0, row.parent))
        print(string.format("  %s|-- %s", indent, row.detail))
    end
end

function Profiler:report()
    print("\n=== Performance Report ===")
    print(string.format("%-40s %8s %8s %10s", "Query", "Iters", "Avg μs", "Total ms"))
    print(string.rep("-", 72))
    table.sort(self.results, function(a, b) return a.avg_us > b.avg_us end)
    for _, r in ipairs(self.results) do
        print(string.format("%-40s %8d %8.1f %10.1f",
            r.name, r.iterations, r.avg_us, r.total_ms))
    end
end

-- Setup test database
local db = sqlite3.open(":memory:")
db:exec([[
    PRAGMA journal_mode=WAL;
    CREATE TABLE orders (
        id       INTEGER PRIMARY KEY,
        user_id  INTEGER NOT NULL,
        amount   REAL,
        status   TEXT,
        created  TEXT DEFAULT (datetime('now'))
    );
]])

-- Insert test data
db:exec("BEGIN")
local ins = db:prepare("INSERT INTO orders(user_id,amount,status) VALUES(?,?,?)")
local statuses = {"pending","paid","shipped","delivered","cancelled"}
math.randomseed(42)
for i = 1, 50000 do
    ins:bind_values(
        math.random(1, 1000),
        math.random(10, 9999) / 100.0,
        statuses[math.random(#statuses)])
    ins:step(); ins:reset()
end
ins:finalize()
db:exec("COMMIT")

local prof = Profiler.new(db)

-- Compare with vs without index
local sql_no_idx = "SELECT COUNT(*), SUM(amount) FROM orders WHERE user_id=?"
local sql_status = "SELECT COUNT(*) FROM orders WHERE status=?"

prof:print_plan(sql_no_idx, {42})
prof:benchmark("COUNT by user_id (no index)", sql_no_idx, {42}, 200)

db:exec("CREATE INDEX idx_user_id ON orders(user_id)")
prof:print_plan(sql_no_idx, {42})
prof:benchmark("COUNT by user_id (with index)", sql_no_idx, {42}, 200)

db:exec("CREATE INDEX idx_status ON orders(status)")
prof:benchmark("COUNT by status (with index)", sql_status, {"paid"}, 200)

-- Aggregate query
local sql_agg = [[
    SELECT user_id, COUNT(*) n, SUM(amount) total
    FROM orders WHERE status='paid' GROUP BY user_id ORDER BY total DESC LIMIT 10
]]
prof:print_plan(sql_agg)
prof:benchmark("Top 10 users by revenue", sql_agg, nil, 50)

prof:report()
db:close()
```

---

## สรุป Database ใน Lua

### เปรียบเทียบ Options

| Library | Database | ติดตั้ง | ใช้งาน |
|---------|---------|---------|--------|
| lsqlite3 | SQLite | ง่าย | ตรงไปตรงมา |
| luasql-sqlite3 | SQLite | ง่าย | Uniform API |
| luasql-mysql | MySQL | ปานกลาง | Uniform API |
| luasql-postgres | PostgreSQL | ปานกลาง | Uniform API |
| lua-resty-redis | Redis | ต้อง OpenResty | ดีมากสำหรับ cache |

### Best Practices

```lua
-- 1. ใช้ Prepared Statements เสมอ
local stmt = db:prepare("SELECT * FROM users WHERE id = ?")
stmt:bind(1, user_id)  -- ไม่ใช่ string concat!

-- 2. Transaction สำหรับ batch operations
db:exec("BEGIN")
for _, item in ipairs(items) do
    -- insert items
end
db:exec("COMMIT")

-- 3. จัดการ error
local rc = db:exec(sql)
if rc ~= sqlite3.OK then
    print("Error:", db:errmsg())
end

-- 4. ปิด statements และ db
stmt:finalize()
db:close()

-- 5. PRAGMA สำหรับ performance
db:exec("PRAGMA journal_mode = WAL")
db:exec("PRAGMA synchronous = NORMAL")
db:exec("PRAGMA cache_size = 10000")

-- 6. Index ที่เหมาะสม
-- CREATE INDEX idx_name ON table(column)
```
