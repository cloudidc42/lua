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
