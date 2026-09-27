# บทที่ 60: SQLite และ NoSQL Patterns ใน Lua

## บทนำ

SQLite เป็น embedded database ที่ไม่ต้องการ server แยกต่างหาก ใช้งานได้ทันทีจาก application เพียงแค่ link library เข้าไป เหมาะสำหรับ desktop applications, mobile apps, หรือ applications ขนาดเล็กถึงกลางที่ไม่ต้องการ scalability ระดับสูง

---

## 60.1 SQLite พื้นฐานด้วย LuaSQLite3

### ตัวอย่างที่ 1: การเปิดและปิด Database

```lua
-- sqlite_basics.lua
-- ต้องติดตั้ง: luarocks install lsqlite3

local sqlite3 = require("lsqlite3")

-- เปิด database (สร้างใหม่ถ้าไม่มี)
local db = sqlite3.open("myapp.db")

print("SQLite version:", sqlite3.version())
print("Database opened successfully")

-- ปิด database
db:close()
print("Database closed")

-- เปิด in-memory database
local memdb = sqlite3.open(":memory:")
print("In-memory database opened")
memdb:close()
```

### ตัวอย่างที่ 2: สร้างตารางและ Insert ข้อมูล

```lua
-- create_table.lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- สร้างตาราง
db:exec([[
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL,
        age INTEGER,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
]])

-- Insert ข้อมูล
local stmt = db:prepare("INSERT INTO users (name, email, age) VALUES (?, ?, ?)")

local users_data = {
    {"สมชาย ใจดี", "somchai@example.com", 30},
    {"สมหญิง รักษ์ดี", "somying@example.com", 25},
    {"มานะ ขยันดี", "mana@example.com", 35},
    {"วิชัย เก่งมาก", "wichai@example.com", 28},
}

for _, user in ipairs(users_data) do
    stmt:bind_values(user[1], user[2], user[3])
    stmt:step()
    stmt:reset()
end

stmt:finalize()

print("Inserted", db:changes(), "rows")
print("Last row ID:", db:last_insert_rowid())

db:close()
```

### ตัวอย่างที่ 3: Query ข้อมูล

```lua
-- query_data.lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- สร้างและ populate ตาราง
db:exec([[
    CREATE TABLE products (
        id INTEGER PRIMARY KEY,
        name TEXT NOT NULL,
        price REAL NOT NULL,
        category TEXT,
        stock INTEGER DEFAULT 0
    )
]])

local insert = db:prepare(
    "INSERT INTO products (name, price, category, stock) VALUES (?, ?, ?, ?)"
)

local products = {
    {"Apple MacBook Pro", 59900, "Electronics", 15},
    {"iPhone 15", 35900, "Electronics", 50},
    {"โต๊ะไม้", 4500, "Furniture", 30},
    {"เก้าอี้สำนักงาน", 3200, "Furniture", 45},
    {"หนังสือ Lua Programming", 350, "Books", 100},
    {"หนังสือ Clean Code", 420, "Books", 80},
}

for _, p in ipairs(products) do
    insert:bind_values(p[1], p[2], p[3], p[4])
    insert:step()
    insert:reset()
end
insert:finalize()

-- Query ทั้งหมด
print("=== All Products ===")
for row in db:nrows("SELECT * FROM products ORDER BY price DESC") do
    print(string.format("  [%d] %s - %.2f บาท (คลัง: %d)",
        row.id, row.name, row.price, row.stock))
end

-- Query ด้วย WHERE
print("\n=== Electronics ===")
local stmt = db:prepare("SELECT * FROM products WHERE category = ? AND price < ?")
stmt:bind_values("Electronics", 40000)
for row in stmt:nrows() do
    print(string.format("  %s - %.2f บาท", row.name, row.price))
end
stmt:finalize()

-- Aggregate
print("\n=== Summary by Category ===")
for row in db:nrows([[
    SELECT category, 
           COUNT(*) as count,
           AVG(price) as avg_price,
           SUM(stock) as total_stock
    FROM products 
    GROUP BY category
    ORDER BY category
]]) do
    print(string.format("  %s: %d สินค้า, ราคาเฉลี่ย %.2f, คลัง %d",
        row.category, row.count, row.avg_price, row.total_stock))
end

db:close()
```

---

## 60.2 Prepared Statements และ Transactions

### ตัวอย่างที่ 4: Transaction Management

```lua
-- transactions.lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE accounts (
        id INTEGER PRIMARY KEY,
        owner TEXT NOT NULL,
        balance REAL NOT NULL DEFAULT 0
    )
]])

-- เพิ่มข้อมูลเริ่มต้น
db:exec([[
    INSERT INTO accounts (owner, balance) VALUES
    ('ลูกค้า A', 10000),
    ('ลูกค้า B', 5000)
]])

-- ฟังก์ชัน transfer เงิน
local function transfer(db, from_id, to_id, amount)
    -- ตรวจสอบยอดเงิน
    local stmt = db:prepare("SELECT balance FROM accounts WHERE id = ?")
    stmt:bind_values(from_id)
    local row = stmt:first_row()
    stmt:finalize()

    if not row then
        return false, "ไม่พบบัญชีต้นทาง"
    end

    if row[1] < amount then
        return false, string.format("ยอดเงินไม่พอ (มี %.2f, ต้องการ %.2f)", row[1], amount)
    end

    -- เริ่ม transaction
    db:exec("BEGIN TRANSACTION")

    local ok, err = pcall(function()
        local deduct = db:prepare("UPDATE accounts SET balance = balance - ? WHERE id = ?")
        deduct:bind_values(amount, from_id)
        deduct:step()
        deduct:finalize()

        local credit = db:prepare("UPDATE accounts SET balance = balance + ? WHERE id = ?")
        credit:bind_values(amount, to_id)
        credit:step()
        credit:finalize()
    end)

    if ok then
        db:exec("COMMIT")
        return true, "โอนเงินสำเร็จ"
    else
        db:exec("ROLLBACK")
        return false, "เกิดข้อผิดพลาด: " .. tostring(err)
    end
end

-- ทดสอบ transfer
local function show_balances()
    for row in db:nrows("SELECT owner, balance FROM accounts ORDER BY id") do
        print(string.format("  %s: %.2f บาท", row.owner, row.balance))
    end
end

print("=== ก่อน Transfer ===")
show_balances()

local ok, msg = transfer(db, 1, 2, 3000)
print("\nTransfer 3000 บาท:", msg)
print("=== หลัง Transfer ===")
show_balances()

-- ทดสอบ transfer ที่ล้มเหลว
ok, msg = transfer(db, 2, 1, 99999)
print("\nTransfer 99999 บาท:", msg)
print("=== ยอดไม่เปลี่ยน ===")
show_balances()

db:close()
```

### ตัวอย่างที่ 5: Batch Insert ด้วย Transaction

```lua
-- batch_insert.lua
local sqlite3 = require("lsqlite3")
local os = require("os")

local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE logs (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        level TEXT NOT NULL,
        message TEXT NOT NULL,
        created_at INTEGER NOT NULL
    )
]])

-- วัดประสิทธิภาพ: insert ทีละรายการ vs batch
local function time_insert(label, count, use_transaction)
    local start = os.clock()

    if use_transaction then
        db:exec("BEGIN TRANSACTION")
    end

    local stmt = db:prepare(
        "INSERT INTO logs (level, message, created_at) VALUES (?, ?, ?)"
    )

    local levels = {"INFO", "WARN", "ERROR", "DEBUG"}
    for i = 1, count do
        local level = levels[(i % 4) + 1]
        stmt:bind_values(level, "Log message #" .. i, os.time())
        stmt:step()
        stmt:reset()
    end

    stmt:finalize()

    if use_transaction then
        db:exec("COMMIT")
    end

    local elapsed = os.clock() - start
    print(string.format("  %s: %d rows ใน %.3f วินาที",
        label, count, elapsed))

    -- ล้างข้อมูล
    db:exec("DELETE FROM logs")
end

print("=== Benchmark Insert ===")
time_insert("ไม่มี Transaction", 100, false)
time_insert("มี Transaction", 100, true)
time_insert("Transaction 1000 rows", 1000, true)

db:close()
```

---

## 60.3 Advanced Queries และ Indexes

### ตัวอย่างที่ 6: Indexes และ Query Optimization

```lua
-- indexes.lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- สร้างตารางและ indexes
db:exec([[
    CREATE TABLE orders (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        customer_id INTEGER NOT NULL,
        product_id INTEGER NOT NULL,
        quantity INTEGER NOT NULL,
        total_price REAL NOT NULL,
        status TEXT DEFAULT 'pending',
        order_date TEXT NOT NULL,
        shipped_date TEXT
    );
    
    -- สร้าง indexes เพื่อเพิ่มประสิทธิภาพ
    CREATE INDEX idx_orders_customer ON orders(customer_id);
    CREATE INDEX idx_orders_status ON orders(status);
    CREATE INDEX idx_orders_date ON orders(order_date);
    
    -- Composite index
    CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);
]])

-- Populate ข้อมูลทดสอบ
db:exec("BEGIN")
local insert = db:prepare([[
    INSERT INTO orders (customer_id, product_id, quantity, total_price, status, order_date)
    VALUES (?, ?, ?, ?, ?, ?)
]])

local statuses = {"pending", "processing", "shipped", "delivered", "cancelled"}
math.randomseed(42)

for i = 1, 500 do
    local customer_id = math.random(1, 50)
    local product_id = math.random(1, 100)
    local qty = math.random(1, 10)
    local price = qty * (math.random(100, 5000) / 10)
    local status = statuses[math.random(#statuses)]
    local days_ago = math.random(0, 365)
    local date = string.format("2024-%02d-%02d",
        math.random(1, 12), math.random(1, 28))

    insert:bind_values(customer_id, product_id, qty, price, status, date)
    insert:step()
    insert:reset()
end
insert:finalize()
db:exec("COMMIT")

-- Query ที่ใช้ประโยชน์จาก index
print("=== Orders by Customer ===")
local stmt = db:prepare([[
    SELECT COUNT(*) as cnt, SUM(total_price) as total
    FROM orders 
    WHERE customer_id = ? AND status = 'delivered'
]])
stmt:bind_values(1)
for row in stmt:nrows() do
    print(string.format("  Customer 1 delivered: %d orders, total %.2f", 
        row.cnt, row.total or 0))
end
stmt:finalize()

-- Complex JOIN-like query (SQLite ไม่มี JOIN กับ in-memory tables แต่ทำได้)
print("\n=== Top 5 Customers by Revenue ===")
for row in db:nrows([[
    SELECT 
        customer_id,
        COUNT(*) as order_count,
        SUM(total_price) as total_revenue,
        AVG(total_price) as avg_order
    FROM orders
    WHERE status IN ('shipped', 'delivered')
    GROUP BY customer_id
    ORDER BY total_revenue DESC
    LIMIT 5
]]) do
    print(string.format("  Customer %d: %d orders, %.2f รวม",
        row.customer_id, row.order_count, row.total_revenue))
end

db:close()
```

### ตัวอย่างที่ 7: Full-Text Search ด้วย FTS5

```lua
-- fts_search.lua
local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- สร้าง FTS virtual table
db:exec([[
    CREATE VIRTUAL TABLE articles USING fts5(
        title,
        content,
        author,
        tokenize = "unicode61 remove_diacritics 1"
    )
]])

-- เพิ่มบทความ
local articles = {
    {
        title = "Introduction to Lua Programming",
        content = "Lua is a powerful scripting language designed for embedded systems. It has a simple syntax and is easy to learn.",
        author = "John Doe"
    },
    {
        title = "Advanced Lua Techniques",
        content = "Learn advanced concepts like metatables, coroutines, and closures in Lua programming language.",
        author = "Jane Smith"
    },
    {
        title = "Lua for Game Development",
        content = "Many game engines use Lua for scripting. LÖVE2D and Defold are popular choices for indie game developers.",
        author = "Game Dev"
    },
    {
        title = "SQLite with Lua",
        content = "SQLite is an embedded database perfect for Lua applications. Learn how to use lsqlite3 library.",
        author = "DB Expert"
    },
    {
        title = "Web Development with OpenResty",
        content = "OpenResty combines Nginx with Lua to create high-performance web applications and APIs.",
        author = "Web Dev"
    },
}

local insert = db:prepare("INSERT INTO articles VALUES (?, ?, ?)")
for _, article in ipairs(articles) do
    insert:bind_values(article.title, article.content, article.author)
    insert:step()
    insert:reset()
end
insert:finalize()

-- ค้นหา
local function search(query)
    print(string.format("\n=== Search: '%s' ===", query))
    local stmt = db:prepare([[
        SELECT title, author, 
               snippet(articles, 1, '<mark>', '</mark>', '...', 15) as excerpt
        FROM articles 
        WHERE articles MATCH ?
        ORDER BY rank
    ]])
    stmt:bind_values(query)

    local found = 0
    for row in stmt:nrows() do
        found = found + 1
        print(string.format("  [%d] %s (โดย %s)", found, row.title, row.author))
        print(string.format("      %s", row.excerpt))
    end

    if found == 0 then
        print("  ไม่พบผลลัพธ์")
    end
    stmt:finalize()
end

search("Lua")
search("game development")
search("embedded database")
search("Python")  -- ไม่มีใน database

db:close()
```

---

## 60.4 ORM Pattern สำหรับ SQLite

### ตัวอย่างที่ 8: Simple ORM

```lua
-- simple_orm.lua
local sqlite3 = require("lsqlite3")

-- Base Model class
local Model = {}
Model.__index = Model

function Model.new(db, table_name, schema)
    local instance = setmetatable({}, Model)
    instance._db = db
    instance._table = table_name
    instance._schema = schema
    instance:_create_table()
    return instance
end

function Model:_create_table()
    local columns = {"id INTEGER PRIMARY KEY AUTOINCREMENT"}
    for _, col in ipairs(self._schema) do
        table.insert(columns, col.name .. " " .. col.type ..
            (col.not_null and " NOT NULL" or "") ..
            (col.default ~= nil and " DEFAULT " .. tostring(col.default) or ""))
    end
    table.insert(columns, "created_at INTEGER DEFAULT (strftime('%s', 'now'))")
    table.insert(columns, "updated_at INTEGER DEFAULT (strftime('%s', 'now'))")

    local sql = string.format("CREATE TABLE IF NOT EXISTS %s (%s)",
        self._table, table.concat(columns, ", "))
    self._db:exec(sql)
end

function Model:create(data)
    local cols = {}
    local placeholders = {}
    local values = {}

    for _, col in ipairs(self._schema) do
        if data[col.name] ~= nil then
            table.insert(cols, col.name)
            table.insert(placeholders, "?")
            table.insert(values, data[col.name])
        end
    end

    local sql = string.format(
        "INSERT INTO %s (%s) VALUES (%s)",
        self._table,
        table.concat(cols, ", "),
        table.concat(placeholders, ", ")
    )

    local stmt = self._db:prepare(sql)
    stmt:bind_values(table.unpack(values))
    stmt:step()
    stmt:finalize()

    return self:find(self._db:last_insert_rowid())
end

function Model:find(id)
    local stmt = self._db:prepare(
        "SELECT * FROM " .. self._table .. " WHERE id = ?"
    )
    stmt:bind_values(id)
    local row = stmt:first_row()
    stmt:finalize()

    if not row then return nil end

    -- แปลง numeric row เป็น named table
    local result = {id = row[1]}
    for i, col in ipairs(self._schema) do
        result[col.name] = row[i + 1]
    end
    result.created_at = row[#self._schema + 2]
    result.updated_at = row[#self._schema + 3]
    return result
end

function Model:where(conditions, limit, order)
    local wheres = {}
    local values = {}

    for col, val in pairs(conditions) do
        if type(val) == "table" then
            -- Support {op, value}
            table.insert(wheres, col .. " " .. val[1] .. " ?")
            table.insert(values, val[2])
        else
            table.insert(wheres, col .. " = ?")
            table.insert(values, val)
        end
    end

    local sql = "SELECT * FROM " .. self._table
    if #wheres > 0 then
        sql = sql .. " WHERE " .. table.concat(wheres, " AND ")
    end
    if order then sql = sql .. " ORDER BY " .. order end
    if limit then sql = sql .. " LIMIT " .. limit end

    local stmt = self._db:prepare(sql)
    if #values > 0 then
        stmt:bind_values(table.unpack(values))
    end

    local results = {}
    for numeric_row in stmt:urows() do
        local row = {id = numeric_row[1]}
        for i, col in ipairs(self._schema) do
            row[col.name] = numeric_row[i + 1]
        end
        table.insert(results, row)
    end
    stmt:finalize()

    return results
end

function Model:update(id, data)
    local sets = {}
    local values = {}

    for _, col in ipairs(self._schema) do
        if data[col.name] ~= nil then
            table.insert(sets, col.name .. " = ?")
            table.insert(values, data[col.name])
        end
    end

    table.insert(sets, "updated_at = strftime('%s', 'now')")
    table.insert(values, id)

    local sql = string.format(
        "UPDATE %s SET %s WHERE id = ?",
        self._table, table.concat(sets, ", ")
    )

    local stmt = self._db:prepare(sql)
    stmt:bind_values(table.unpack(values))
    stmt:step()
    stmt:finalize()

    return self:find(id)
end

function Model:delete(id)
    local stmt = self._db:prepare(
        "DELETE FROM " .. self._table .. " WHERE id = ?"
    )
    stmt:bind_values(id)
    stmt:step()
    local affected = self._db:changes()
    stmt:finalize()
    return affected > 0
end

function Model:count(conditions)
    local wheres = {}
    local values = {}

    if conditions then
        for col, val in pairs(conditions) do
            table.insert(wheres, col .. " = ?")
            table.insert(values, val)
        end
    end

    local sql = "SELECT COUNT(*) FROM " .. self._table
    if #wheres > 0 then
        sql = sql .. " WHERE " .. table.concat(wheres, " AND ")
    end

    local stmt = self._db:prepare(sql)
    if #values > 0 then
        stmt:bind_values(table.unpack(values))
    end
    local row = stmt:first_row()
    stmt:finalize()

    return row and row[1] or 0
end

-- ทดสอบ ORM
local db = sqlite3.open(":memory:")

-- สร้าง User model
local User = Model.new(db, "users", {
    {name = "name", type = "TEXT", not_null = true},
    {name = "email", type = "TEXT", not_null = true},
    {name = "age", type = "INTEGER"},
    {name = "active", type = "INTEGER", default = 1},
})

-- Create
print("=== Creating Users ===")
local u1 = User:create({name = "สมชาย", email = "somchai@test.com", age = 30})
local u2 = User:create({name = "สมหญิง", email = "somying@test.com", age = 25})
local u3 = User:create({name = "มานะ", email = "mana@test.com", age = 35, active = 0})

print("Created user:", u1.id, u1.name, u1.email)

-- Find
print("\n=== Finding User ===")
local found = User:find(2)
print("Found:", found.name, "age:", found.age)

-- Where
print("\n=== Active Users ===")
local actives = User:where({active = 1}, nil, "name ASC")
for _, u in ipairs(actives) do
    print("  -", u.name, "(อายุ", u.age, ")")
end

-- Update
print("\n=== Update User ===")
local updated = User:update(1, {age = 31, active = 1})
print("Updated:", updated.name, "age:", updated.age)

-- Count
print("\n=== Count ===")
print("Total users:", User:count())
print("Active users:", User:count({active = 1}))

-- Delete
print("\n=== Delete User ===")
local deleted = User:delete(3)
print("Deleted:", deleted)
print("Remaining:", User:count())

db:close()
```

---

## 60.5 NoSQL Patterns ใน Lua

### ตัวอย่างที่ 9: Document Store ด้วย SQLite JSON

```lua
-- document_store.lua
-- SQLite 3.38+ รองรับ JSON functions

local sqlite3 = require("lsqlite3")

local db = sqlite3.open(":memory:")

-- สร้าง document store table
db:exec([[
    CREATE TABLE documents (
        id TEXT PRIMARY KEY,
        collection TEXT NOT NULL,
        data TEXT NOT NULL,  -- JSON
        created_at INTEGER DEFAULT (strftime('%s', 'now')),
        updated_at INTEGER DEFAULT (strftime('%s', 'now'))
    );
    
    CREATE INDEX idx_doc_collection ON documents(collection);
]])

-- JSON encoding/decoding แบบง่าย
local function json_encode(t)
    if type(t) == "number" then return tostring(t) end
    if type(t) == "boolean" then return t and "true" or "false" end
    if type(t) == "string" then
        return '"' .. t:gsub('"', '\\"'):gsub('\n', '\\n') .. '"'
    end
    if type(t) == "table" then
        -- ตรวจว่าเป็น array หรือ object
        local is_array = #t > 0
        if is_array then
            local parts = {}
            for _, v in ipairs(t) do
                table.insert(parts, json_encode(v))
            end
            return "[" .. table.concat(parts, ",") .. "]"
        else
            local parts = {}
            for k, v in pairs(t) do
                table.insert(parts, '"' .. k .. '":' .. json_encode(v))
            end
            return "{" .. table.concat(parts, ",") .. "}"
        end
    end
    return "null"
end

-- UUID generator แบบง่าย
local function uuid()
    math.randomseed(os.time() + math.random(1000000))
    return string.format("%08x-%04x-%04x-%04x-%012x",
        math.random(0xFFFFFFFF),
        math.random(0xFFFF),
        math.random(0xFFFF),
        math.random(0xFFFF),
        math.random(0xFFFFFFFFFFFF)
    )
end

-- DocumentStore class
local DocumentStore = {}
DocumentStore.__index = DocumentStore

function DocumentStore.new(db)
    return setmetatable({_db = db}, DocumentStore)
end

function DocumentStore:insert(collection, doc)
    local id = doc._id or uuid()
    doc._id = id

    local stmt = self._db:prepare([[
        INSERT OR REPLACE INTO documents (id, collection, data)
        VALUES (?, ?, ?)
    ]])
    stmt:bind_values(id, collection, json_encode(doc))
    stmt:step()
    stmt:finalize()

    return id
end

function DocumentStore:findById(collection, id)
    local stmt = self._db:prepare([[
        SELECT data FROM documents WHERE id = ? AND collection = ?
    ]])
    stmt:bind_values(id, collection)
    local row = stmt:first_row()
    stmt:finalize()

    if row then
        -- ใน production ควรใช้ JSON library ที่ดีกว่า
        return {_raw = row[1], _id = id}
    end
    return nil
end

function DocumentStore:count(collection)
    local stmt = self._db:prepare(
        "SELECT COUNT(*) FROM documents WHERE collection = ?"
    )
    stmt:bind_values(collection)
    local row = stmt:first_row()
    stmt:finalize()
    return row and row[1] or 0
end

function DocumentStore:listAll(collection, limit)
    local sql = "SELECT id, data FROM documents WHERE collection = ? ORDER BY created_at DESC"
    if limit then sql = sql .. " LIMIT " .. limit end

    local stmt = self._db:prepare(sql)
    stmt:bind_values(collection)

    local results = {}
    for row in stmt:urows() do
        table.insert(results, {_id = row[1], _raw = row[2]})
    end
    stmt:finalize()

    return results
end

function DocumentStore:delete(collection, id)
    local stmt = self._db:prepare(
        "DELETE FROM documents WHERE id = ? AND collection = ?"
    )
    stmt:bind_values(id, collection)
    stmt:step()
    local affected = self._db:changes()
    stmt:finalize()
    return affected > 0
end

-- ทดสอบ
local store = DocumentStore.new(db)

-- Insert documents
print("=== Inserting Documents ===")
local blog_id = store:insert("posts", {
    title = "First Post",
    content = "Hello World from SQLite Document Store",
    tags = {"lua", "sqlite", "nosql"},
    author = {name = "Admin", email = "admin@test.com"},
    published = true,
    views = 0
})
print("Inserted post:", blog_id)

store:insert("posts", {
    title = "Lua Tutorial",
    content = "Learning Lua programming language",
    tags = {"lua", "tutorial"},
    author = {name = "Teacher", email = "teacher@test.com"},
    published = false,
    views = 0
})

store:insert("users", {
    name = "Test User",
    email = "test@example.com",
    role = "admin"
})

-- Count
print("\n=== Collections ===")
print("Posts:", store:count("posts"))
print("Users:", store:count("users"))

-- List
print("\n=== All Posts ===")
local posts = store:listAll("posts")
for _, post in ipairs(posts) do
    print("  [" .. post._id .. "]:", post._raw:sub(1, 60) .. "...")
end

-- Find by ID
print("\n=== Find Post ===")
local found = store:findById("posts", blog_id)
if found then
    print("Found:", found._raw:sub(1, 80))
end

-- Delete
print("\n=== Delete Post ===")
local deleted = store:delete("posts", blog_id)
print("Deleted:", deleted)
print("Remaining posts:", store:count("posts"))

db:close()
```

---

## 60.6 Key-Value Store

### ตัวอย่างที่ 10: Persistent Key-Value Store

```lua
-- kv_store.lua
local sqlite3 = require("lsqlite3")

-- KV Store class
local KVStore = {}
KVStore.__index = KVStore

function KVStore.new(path)
    local self = setmetatable({}, KVStore)
    self._db = sqlite3.open(path or ":memory:")
    self._db:exec([[
        CREATE TABLE IF NOT EXISTS kv_store (
            key TEXT PRIMARY KEY,
            value TEXT,
            type TEXT DEFAULT 'string',
            expires_at INTEGER,
            created_at INTEGER DEFAULT (strftime('%s', 'now')),
            updated_at INTEGER DEFAULT (strftime('%s', 'now'))
        );
        CREATE INDEX IF NOT EXISTS idx_kv_expires ON kv_store(expires_at)
            WHERE expires_at IS NOT NULL;
    ]])
    return self
end

function KVStore:set(key, value, ttl_seconds)
    local vtype = type(value)
    local encoded_value

    if vtype == "table" then
        -- Simple serialization
        encoded_value = require and tostring(value) or "{}"
        vtype = "table"
    elseif vtype == "number" or vtype == "boolean" then
        encoded_value = tostring(value)
    else
        encoded_value = tostring(value or "")
    end

    local expires_at = ttl_seconds and (os.time() + ttl_seconds) or nil

    local stmt = self._db:prepare([[
        INSERT OR REPLACE INTO kv_store (key, value, type, expires_at, updated_at)
        VALUES (?, ?, ?, ?, strftime('%s', 'now'))
    ]])
    stmt:bind_values(key, encoded_value, vtype, expires_at)
    stmt:step()
    stmt:finalize()

    return true
end

function KVStore:get(key)
    -- ลบ expired keys ก่อน
    self:_cleanup_expired()

    local stmt = self._db:prepare([[
        SELECT value, type, expires_at FROM kv_store
        WHERE key = ? AND (expires_at IS NULL OR expires_at > strftime('%s', 'now'))
    ]])
    stmt:bind_values(key)
    local row = stmt:first_row()
    stmt:finalize()

    if not row then return nil end

    local value, vtype = row[1], row[2]

    if vtype == "number" then
        return tonumber(value)
    elseif vtype == "boolean" then
        return value == "true"
    else
        return value
    end
end

function KVStore:delete(key)
    local stmt = self._db:prepare("DELETE FROM kv_store WHERE key = ?")
    stmt:bind_values(key)
    stmt:step()
    local affected = self._db:changes()
    stmt:finalize()
    return affected > 0
end

function KVStore:exists(key)
    self:_cleanup_expired()
    local stmt = self._db:prepare([[
        SELECT 1 FROM kv_store 
        WHERE key = ? AND (expires_at IS NULL OR expires_at > strftime('%s', 'now'))
    ]])
    stmt:bind_values(key)
    local row = stmt:first_row()
    stmt:finalize()
    return row ~= nil
end

function KVStore:ttl(key)
    local stmt = self._db:prepare([[
        SELECT expires_at - strftime('%s', 'now') FROM kv_store
        WHERE key = ? AND expires_at IS NOT NULL
    ]])
    stmt:bind_values(key)
    local row = stmt:first_row()
    stmt:finalize()
    return row and row[1] or -1
end

function KVStore:keys(pattern)
    self:_cleanup_expired()
    local sql = [[
        SELECT key FROM kv_store
        WHERE expires_at IS NULL OR expires_at > strftime('%s', 'now')
    ]]

    if pattern then
        sql = sql .. " AND key LIKE ?"
    end
    sql = sql .. " ORDER BY key"

    local stmt = self._db:prepare(sql)
    if pattern then
        stmt:bind_values(pattern:gsub("*", "%%"))
    end

    local results = {}
    for row in stmt:urows() do
        table.insert(results, row[1])
    end
    stmt:finalize()

    return results
end

function KVStore:incr(key, amount)
    amount = amount or 1
    local current = tonumber(self:get(key)) or 0
    local new_val = current + amount
    self:set(key, new_val)
    return new_val
end

function KVStore:_cleanup_expired()
    self._db:exec([[
        DELETE FROM kv_store 
        WHERE expires_at IS NOT NULL AND expires_at <= strftime('%s', 'now')
    ]])
end

function KVStore:close()
    self._db:close()
end

-- ทดสอบ KV Store
local kv = KVStore.new()

print("=== Basic Operations ===")
kv:set("username", "admin")
kv:set("login_count", 42)
kv:set("is_active", true)

print("username:", kv:get("username"))
print("login_count:", kv:get("login_count"))
print("is_active:", kv:get("is_active"))

print("\n=== Increment ===")
kv:incr("login_count")
kv:incr("login_count")
kv:incr("login_count", 10)
print("login_count after incr:", kv:get("login_count"))

print("\n=== TTL (Time-To-Live) ===")
kv:set("session_token", "abc123xyz", 3600)  -- expires in 1 hour
print("session_token:", kv:get("session_token"))
print("TTL remaining:", kv:ttl("session_token"), "seconds")

print("\n=== Keys Pattern ===")
kv:set("user:1:name", "สมชาย")
kv:set("user:1:email", "somchai@test.com")
kv:set("user:2:name", "สมหญิง")
kv:set("user:2:email", "somying@test.com")
kv:set("config:debug", "true")

local user_keys = kv:keys("user:*")
print("User keys:")
for _, k in ipairs(user_keys) do
    print("  " .. k .. " =", kv:get(k))
end

print("\nAll keys count:", #kv:keys())

kv:close()
```

---

## 60.7 Database Migration Pattern

### ตัวอย่างที่ 11: Schema Migration System

```lua
-- migrations.lua
local sqlite3 = require("lsqlite3")

-- Migration Manager
local MigrationManager = {}
MigrationManager.__index = MigrationManager

function MigrationManager.new(db)
    local self = setmetatable({}, MigrationManager)
    self._db = db
    self._migrations = {}
    self:_init()
    return self
end

function MigrationManager:_init()
    self._db:exec([[
        CREATE TABLE IF NOT EXISTS schema_migrations (
            version TEXT PRIMARY KEY,
            name TEXT NOT NULL,
            applied_at INTEGER DEFAULT (strftime('%s', 'now'))
        )
    ]])
end

function MigrationManager:add(version, name, up_fn, down_fn)
    table.insert(self._migrations, {
        version = version,
        name = name,
        up = up_fn,
        down = down_fn,
    })
    table.sort(self._migrations, function(a, b)
        return a.version < b.version
    end)
end

function MigrationManager:_is_applied(version)
    local stmt = self._db:prepare(
        "SELECT 1 FROM schema_migrations WHERE version = ?"
    )
    stmt:bind_values(version)
    local row = stmt:first_row()
    stmt:finalize()
    return row ~= nil
end

function MigrationManager:migrate()
    local applied = 0

    for _, migration in ipairs(self._migrations) do
        if not self:_is_applied(migration.version) then
            print(string.format("  Applying migration %s: %s",
                migration.version, migration.name))

            self._db:exec("BEGIN")
            local ok, err = pcall(migration.up, self._db)

            if ok then
                local stmt = self._db:prepare(
                    "INSERT INTO schema_migrations (version, name) VALUES (?, ?)"
                )
                stmt:bind_values(migration.version, migration.name)
                stmt:step()
                stmt:finalize()
                self._db:exec("COMMIT")
                applied = applied + 1
                print("    -> Applied successfully")
            else
                self._db:exec("ROLLBACK")
                error(string.format("Migration %s failed: %s",
                    migration.version, err))
            end
        end
    end

    if applied == 0 then
        print("  No pending migrations")
    else
        print(string.format("  Applied %d migration(s)", applied))
    end
end

function MigrationManager:rollback(steps)
    steps = steps or 1
    local rolled_back = 0

    -- หา migrations ที่ applied แล้ว (เรียง descending)
    local applied_migrations = {}
    for _, migration in ipairs(self._migrations) do
        if self:_is_applied(migration.version) then
            table.insert(applied_migrations, migration)
        end
    end

    -- Reverse
    for i = 1, #applied_migrations / 2 do
        local j = #applied_migrations - i + 1
        applied_migrations[i], applied_migrations[j] = applied_migrations[j], applied_migrations[i]
    end

    for i = 1, math.min(steps, #applied_migrations) do
        local migration = applied_migrations[i]
        if not migration.down then
            print(string.format("  Migration %s has no rollback", migration.version))
            break
        end

        print(string.format("  Rolling back %s: %s",
            migration.version, migration.name))

        self._db:exec("BEGIN")
        local ok, err = pcall(migration.down, self._db)

        if ok then
            local stmt = self._db:prepare(
                "DELETE FROM schema_migrations WHERE version = ?"
            )
            stmt:bind_values(migration.version)
            stmt:step()
            stmt:finalize()
            self._db:exec("COMMIT")
            rolled_back = rolled_back + 1
            print("    -> Rolled back successfully")
        else
            self._db:exec("ROLLBACK")
            error(string.format("Rollback %s failed: %s",
                migration.version, err))
        end
    end

    print(string.format("  Rolled back %d migration(s)", rolled_back))
end

function MigrationManager:status()
    print("=== Migration Status ===")
    for _, migration in ipairs(self._migrations) do
        local applied = self:_is_applied(migration.version)
        print(string.format("  [%s] %s: %s",
            applied and "x" or " ",
            migration.version,
            migration.name))
    end
end

-- ทดสอบ Migrations
local db = sqlite3.open(":memory:")
local mgr = MigrationManager.new(db)

-- Define migrations
mgr:add("001", "create_users_table",
    function(db)  -- up
        db:exec([[
            CREATE TABLE users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                email TEXT UNIQUE NOT NULL
            )
        ]])
    end,
    function(db)  -- down
        db:exec("DROP TABLE users")
    end
)

mgr:add("002", "add_users_age_column",
    function(db)
        db:exec("ALTER TABLE users ADD COLUMN age INTEGER")
    end,
    function(db)
        -- SQLite ไม่รองรับ DROP COLUMN ใน version เก่า
        -- แต่สำหรับ demo:
        print("    (SQLite: cannot drop column, skipping)")
    end
)

mgr:add("003", "create_products_table",
    function(db)
        db:exec([[
            CREATE TABLE products (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                price REAL NOT NULL,
                created_at INTEGER DEFAULT (strftime('%s', 'now'))
            )
        ]])
    end,
    function(db)
        db:exec("DROP TABLE products")
    end
)

-- Run migrations
print("=== Running Migrations ===")
mgr:migrate()

mgr:status()

-- Run again (should skip)
print("\n=== Re-running (should skip) ===")
mgr:migrate()

-- Rollback 1
print("\n=== Rollback 1 Step ===")
mgr:rollback(1)
mgr:status()

-- Re-migrate
print("\n=== Re-migrate ===")
mgr:migrate()
mgr:status()

db:close()
```

---

## 60.8 Connection Pool สำหรับ SQLite

### ตัวอย่างที่ 12: Database Connection Pool

```lua
-- connection_pool.lua
local sqlite3 = require("lsqlite3")

-- SQLite Connection Pool
local Pool = {}
Pool.__index = Pool

function Pool.new(db_path, pool_size)
    local self = setmetatable({}, Pool)
    self._path = db_path or ":memory:"
    self._size = pool_size or 5
    self._connections = {}
    self._available = {}

    -- สร้าง connections ล่วงหน้า
    for i = 1, self._size do
        local conn = sqlite3.open(self._path)
        -- เปิด WAL mode สำหรับ concurrent access
        conn:exec("PRAGMA journal_mode=WAL")
        conn:exec("PRAGMA synchronous=NORMAL")
        self._connections[i] = conn
        self._available[i] = true
    end

    return self
end

function Pool:acquire(timeout)
    timeout = timeout or 5  -- seconds
    local start = os.time()

    while true do
        for i, available in ipairs(self._available) do
            if available then
                self._available[i] = false
                return i, self._connections[i]
            end
        end

        -- ถ้าหมด pool รอ
        if os.time() - start > timeout then
            return nil, nil, "timeout: no connections available"
        end

        -- ใน production จะใช้ coroutine yield ที่นี่
        -- แต่สำหรับ demo ใช้ busy wait แทน
        os.execute("sleep 0.01")
    end
end

function Pool:release(conn_id)
    if conn_id and self._available[conn_id] == false then
        self._available[conn_id] = true
    end
end

function Pool:execute(fn)
    local conn_id, conn, err = self:acquire()
    if not conn then
        return nil, err
    end

    local ok, result = pcall(fn, conn)
    self:release(conn_id)

    if ok then
        return result
    else
        return nil, result
    end
end

function Pool:stats()
    local total = #self._connections
    local used = 0
    for _, available in ipairs(self._available) do
        if not available then used = used + 1 end
    end
    return {total = total, used = used, available = total - used}
end

function Pool:close()
    for _, conn in ipairs(self._connections) do
        conn:close()
    end
end

-- ทดสอบ Pool
print("=== Connection Pool Demo ===")
local pool = Pool.new(":memory:", 3)

-- Initialize schema ใน connection แรก
local _, conn1 = pool:acquire()
conn1:exec([[
    CREATE TABLE IF NOT EXISTS counter (
        key TEXT PRIMARY KEY,
        value INTEGER DEFAULT 0
    )
]])
conn1:exec("INSERT OR IGNORE INTO counter (key, value) VALUES ('hits', 0)")
pool:release(1)

print("Pool stats:", pool:stats().total, "total,",
    pool:stats().available, "available")

-- Execute queries ผ่าน pool
for i = 1, 5 do
    local result = pool:execute(function(db)
        db:exec("UPDATE counter SET value = value + 1 WHERE key = 'hits'")
        for row in db:nrows("SELECT value FROM counter WHERE key = 'hits'") do
            return row.value
        end
    end)
    print(string.format("  Request %d: counter = %s", i, tostring(result)))
end

-- Stats
local stats = pool:stats()
print(string.format("\nFinal pool stats: %d total, %d available, %d used",
    stats.total, stats.available, stats.used))

pool:close()
```

---

## 60.9 Caching Layer

### ตัวอย่างที่ 13: SQLite-backed Cache

```lua
-- sqlite_cache.lua
local sqlite3 = require("lsqlite3")

local Cache = {}
Cache.__index = Cache

function Cache.new(options)
    options = options or {}
    local self = setmetatable({}, Cache)
    self._db = sqlite3.open(options.path or ":memory:")
    self._max_size = options.max_size or 1000
    self._default_ttl = options.default_ttl or 3600
    self._hits = 0
    self._misses = 0
    self:_init()
    return self
end

function Cache:_init()
    self._db:exec([[
        CREATE TABLE IF NOT EXISTS cache (
            key TEXT PRIMARY KEY,
            value TEXT NOT NULL,
            created_at INTEGER DEFAULT (strftime('%s', 'now')),
            expires_at INTEGER,
            hit_count INTEGER DEFAULT 0,
            last_accessed INTEGER DEFAULT (strftime('%s', 'now'))
        );
        CREATE INDEX IF NOT EXISTS idx_cache_expires ON cache(expires_at);
        CREATE INDEX IF NOT EXISTS idx_cache_accessed ON cache(last_accessed);
    ]])
end

function Cache:set(key, value, ttl)
    ttl = ttl or self._default_ttl
    local expires_at = os.time() + ttl

    -- Serialize value
    local serialized
    if type(value) == "string" then
        serialized = "s:" .. value
    elseif type(value) == "number" then
        serialized = "n:" .. tostring(value)
    elseif type(value) == "boolean" then
        serialized = "b:" .. (value and "1" or "0")
    else
        serialized = "t:" .. tostring(value)
    end

    -- Evict if too large
    local count_row = self._db:nrows("SELECT COUNT(*) as c FROM cache")()
    if count_row and count_row.c >= self._max_size then
        self:_evict()
    end

    local stmt = self._db:prepare([[
        INSERT OR REPLACE INTO cache (key, value, expires_at, hit_count, last_accessed)
        VALUES (?, ?, ?, 0, strftime('%s', 'now'))
    ]])
    stmt:bind_values(key, serialized, expires_at)
    stmt:step()
    stmt:finalize()
end

function Cache:get(key)
    -- Cleanup expired
    self._db:exec(string.format(
        "DELETE FROM cache WHERE expires_at <= %d", os.time()
    ))

    local stmt = self._db:prepare([[
        SELECT value FROM cache 
        WHERE key = ? AND expires_at > strftime('%s', 'now')
    ]])
    stmt:bind_values(key)
    local row = stmt:first_row()
    stmt:finalize()

    if not row then
        self._misses = self._misses + 1
        return nil
    end

    -- Update stats
    self._db:exec(string.format([[
        UPDATE cache 
        SET hit_count = hit_count + 1, last_accessed = strftime('%%s', 'now')
        WHERE key = '%s'
    ]], key:gsub("'", "''")))

    self._hits = self._hits + 1

    -- Deserialize
    local serialized = row[1]
    local prefix = serialized:sub(1, 2)
    local val = serialized:sub(3)

    if prefix == "s:" then return val
    elseif prefix == "n:" then return tonumber(val)
    elseif prefix == "b:" then return val == "1"
    else return val
    end
end

function Cache:delete(key)
    local stmt = self._db:prepare("DELETE FROM cache WHERE key = ?")
    stmt:bind_values(key)
    stmt:step()
    stmt:finalize()
end

function Cache:_evict()
    -- LRU eviction: ลบ 10% ของ entries ที่ใช้งานน้อยที่สุด
    local evict_count = math.max(1, math.floor(self._max_size * 0.1))
    self._db:exec(string.format([[
        DELETE FROM cache WHERE key IN (
            SELECT key FROM cache ORDER BY last_accessed ASC LIMIT %d
        )
    ]], evict_count))
end

function Cache:stats()
    local count_row = self._db:nrows("SELECT COUNT(*) as c FROM cache")()
    local total = count_row and count_row.c or 0
    local hit_rate = (self._hits + self._misses) > 0
        and (self._hits / (self._hits + self._misses) * 100)
        or 0

    return {
        size = total,
        hits = self._hits,
        misses = self._misses,
        hit_rate = string.format("%.1f%%", hit_rate)
    }
end

-- Memoize helper
function Cache:memoize(key_fn, compute_fn, ttl)
    return function(...)
        local key = key_fn(...)
        local cached = self:get(key)
        if cached ~= nil then
            return cached
        end
        local result = compute_fn(...)
        self:set(key, result, ttl)
        return result
    end
end

-- ทดสอบ Cache
local cache = Cache.new({max_size = 100, default_ttl = 60})

print("=== Basic Cache Operations ===")
cache:set("user:1", "สมชาย")
cache:set("config:debug", true)
cache:set("counter", 42)

print("user:1 =", cache:get("user:1"))
print("config:debug =", cache:get("config:debug"))
print("counter =", cache:get("counter"))
print("missing =", tostring(cache:get("nonexistent")))

print("\n=== Memoization ===")
local expensive_compute = function(n)
    -- จำลอง expensive operation
    local sum = 0
    for i = 1, n do sum = sum + i end
    return sum
end

local memoized = cache:memoize(
    function(n) return "sum:" .. n end,
    expensive_compute,
    300
)

print("sum(100) =", memoized(100))
print("sum(100) cached =", memoized(100))  -- จาก cache
print("sum(200) =", memoized(200))

print("\n=== Cache Stats ===")
local stats = cache:stats()
print(string.format("  Size: %d, Hits: %d, Misses: %d, Hit Rate: %s",
    stats.size, stats.hits, stats.misses, stats.hit_rate))
```

---

## 60.10 Pattern: Repository Pattern

### ตัวอย่างที่ 14: Repository Pattern พร้อม Unit of Work

```lua
-- repository_pattern.lua
local sqlite3 = require("lsqlite3")

-- Repository base class
local Repository = {}
Repository.__index = Repository

function Repository.new(db, table_name)
    return setmetatable({
        _db = db,
        _table = table_name,
    }, Repository)
end

function Repository:findAll(options)
    options = options or {}
    local sql = "SELECT * FROM " .. self._table

    local wheres = {}
    local values = {}

    if options.where then
        for col, val in pairs(options.where) do
            table.insert(wheres, col .. " = ?")
            table.insert(values, val)
        end
    end

    if #wheres > 0 then
        sql = sql .. " WHERE " .. table.concat(wheres, " AND ")
    end

    if options.order then sql = sql .. " ORDER BY " .. options.order end
    if options.limit then sql = sql .. " LIMIT " .. options.limit end
    if options.offset then sql = sql .. " OFFSET " .. options.offset end

    local stmt = self._db:prepare(sql)
    if #values > 0 then
        stmt:bind_values(table.unpack(values))
    end

    local results = {}
    for row in stmt:nrows() do
        table.insert(results, row)
    end
    stmt:finalize()

    return results
end

function Repository:findById(id)
    local stmt = self._db:prepare(
        "SELECT * FROM " .. self._table .. " WHERE id = ?"
    )
    stmt:bind_values(id)
    local row = stmt:first_row()
    stmt:finalize()
    return row
end

function Repository:save(entity)
    if entity.id then
        return self:_update(entity)
    else
        return self:_insert(entity)
    end
end

function Repository:_insert(entity)
    local cols = {}
    local placeholders = {}
    local values = {}

    for k, v in pairs(entity) do
        if k ~= "id" then
            table.insert(cols, k)
            table.insert(placeholders, "?")
            table.insert(values, v)
        end
    end

    local sql = string.format(
        "INSERT INTO %s (%s) VALUES (%s)",
        self._table,
        table.concat(cols, ", "),
        table.concat(placeholders, ", ")
    )

    local stmt = self._db:prepare(sql)
    stmt:bind_values(table.unpack(values))
    stmt:step()
    stmt:finalize()

    entity.id = self._db:last_insert_rowid()
    return entity
end

function Repository:_update(entity)
    local sets = {}
    local values = {}

    for k, v in pairs(entity) do
        if k ~= "id" then
            table.insert(sets, k .. " = ?")
            table.insert(values, v)
        end
    end

    table.insert(values, entity.id)

    local sql = string.format(
        "UPDATE %s SET %s WHERE id = ?",
        self._table,
        table.concat(sets, ", ")
    )

    local stmt = self._db:prepare(sql)
    stmt:bind_values(table.unpack(values))
    stmt:step()
    stmt:finalize()

    return entity
end

function Repository:delete(id)
    local stmt = self._db:prepare(
        "DELETE FROM " .. self._table .. " WHERE id = ?"
    )
    stmt:bind_values(id)
    stmt:step()
    local affected = self._db:changes()
    stmt:finalize()
    return affected > 0
end

-- Unit of Work
local UnitOfWork = {}
UnitOfWork.__index = UnitOfWork

function UnitOfWork.new(db)
    local self = setmetatable({}, UnitOfWork)
    self._db = db
    self._new = {}
    self._dirty = {}
    self._deleted = {}
    return self
end

function UnitOfWork:register_new(entity, repo)
    table.insert(self._new, {entity = entity, repo = repo})
end

function UnitOfWork:register_dirty(entity, repo)
    table.insert(self._dirty, {entity = entity, repo = repo})
end

function UnitOfWork:register_deleted(id, repo)
    table.insert(self._deleted, {id = id, repo = repo})
end

function UnitOfWork:commit()
    self._db:exec("BEGIN")

    local ok, err = pcall(function()
        for _, item in ipairs(self._new) do
            item.repo:save(item.entity)
        end
        for _, item in ipairs(self._dirty) do
            item.repo:save(item.entity)
        end
        for _, item in ipairs(self._deleted) do
            item.repo:delete(item.id)
        end
    end)

    if ok then
        self._db:exec("COMMIT")
        self._new = {}
        self._dirty = {}
        self._deleted = {}
        return true
    else
        self._db:exec("ROLLBACK")
        return false, err
    end
end

-- ทดสอบ
local db = sqlite3.open(":memory:")

db:exec([[
    CREATE TABLE customers (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT,
        tier TEXT DEFAULT 'standard'
    );
    CREATE TABLE orders (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        customer_id INTEGER NOT NULL,
        amount REAL NOT NULL,
        status TEXT DEFAULT 'pending'
    )
]])

local customer_repo = Repository.new(db, "customers")
local order_repo = Repository.new(db, "orders")

-- สร้าง Unit of Work
local uow = UnitOfWork.new(db)

local c1 = {name = "ลูกค้า VIP", email = "vip@test.com", tier = "premium"}
local c2 = {name = "ลูกค้าทั่วไป", email = "normal@test.com", tier = "standard"}

uow:register_new(c1, customer_repo)
uow:register_new(c2, customer_repo)

local ok, err = uow:commit()
print("Commit result:", ok, err)

print("\n=== Customers ===")
for _, c in ipairs(customer_repo:findAll({order = "id"})) do
    print(string.format("  [%d] %s (%s)", c.id, c.name, c.tier))
end

-- Add orders
local uow2 = UnitOfWork.new(db)
uow2:register_new({customer_id = c1.id, amount = 5000, status = "paid"}, order_repo)
uow2:register_new({customer_id = c1.id, amount = 3500, status = "pending"}, order_repo)
uow2:register_new({customer_id = c2.id, amount = 1200, status = "paid"}, order_repo)
uow2:commit()

print("\n=== Orders for Customer 1 ===")
for _, o in ipairs(order_repo:findAll({where = {customer_id = c1.id}})) do
    print(string.format("  Order #%d: %.2f (%s)", o.id, o.amount, o.status))
end

db:close()
```

---

## 60.11 สรุปและ Best Practices

### ตัวอย่างที่ 15: Best Practices Checklist

```lua
-- best_practices.lua
-- รวม best practices ทั้งหมดสำหรับ SQLite/NoSQL ใน Lua

--[[
✅ SQLite Best Practices:

1. Connection Management
   - ใช้ connection pool สำหรับ multi-threaded apps
   - ปิด connection เมื่อไม่ใช้งาน
   - ใช้ WAL mode สำหรับ concurrent reads

2. Performance
   - ใช้ transactions สำหรับ batch writes
   - สร้าง indexes บน columns ที่ query บ่อย
   - ใช้ prepared statements (ไม่ต้อง parse SQL ซ้ำ)
   - เปิด PRAGMA cache_size สำหรับ large databases

3. Data Safety
   - ใช้ transactions สำหรับ operations ที่ต้องเป็น atomic
   - Handle errors อย่างเหมาะสม
   - Backup ก่อน migration

4. Schema Design
   - ใช้ INTEGER PRIMARY KEY AUTOINCREMENT
   - เพิ่ม created_at/updated_at ในทุกตาราง
   - ใช้ FOREIGN KEY constraints

5. Security
   - ใช้ parameterized queries (ไม่ใช้ string concatenation)
   - Validate input ก่อน insert
   - จำกัด file permissions ของ .db file
]]

local sqlite3 = require("lsqlite3")

local function demonstrate_pragmas(db)
    print("=== SQLite PRAGMAs ===")

    -- WAL mode: เพิ่ม concurrent read performance
    db:exec("PRAGMA journal_mode=WAL")

    -- Cache size (หน่วย KB หรือ จำนวน pages ถ้าเป็น negative)
    db:exec("PRAGMA cache_size=-10000")  -- ~10MB cache

    -- Synchronous mode (NORMAL = good balance)
    db:exec("PRAGMA synchronous=NORMAL")

    -- Foreign key support (ปิดโดย default!)
    db:exec("PRAGMA foreign_keys=ON")

    -- ตรวจสอบ pragmas
    for row in db:nrows("PRAGMA journal_mode") do
        print("  journal_mode:", row[1])
    end
    for row in db:nrows("PRAGMA foreign_keys") do
        print("  foreign_keys:", row[1] == 1 and "ON" or "OFF")
    end
    for row in db:nrows("PRAGMA cache_size") do
        print("  cache_size:", row[1])
    end
end

local function demonstrate_safe_queries(db)
    print("\n=== Safe Query Patterns ===")

    db:exec("CREATE TABLE test (id INTEGER PRIMARY KEY, name TEXT, value INTEGER)")
    db:exec("INSERT INTO test VALUES (1, 'Alice', 100), (2, 'Bob', 200)")

    -- ✅ GOOD: Parameterized query
    local stmt = db:prepare("SELECT * FROM test WHERE name = ? AND value > ?")
    stmt:bind_values("Alice", 50)
    for row in stmt:nrows() do
        print("  Safe query found:", row.name, row.value)
    end
    stmt:finalize()

    -- ❌ BAD: String concatenation (SQL injection risk!)
    -- local user_input = "' OR '1'='1"
    -- db:exec("SELECT * FROM test WHERE name = '" .. user_input .. "'")
    -- ^ อย่าทำแบบนี้!

    print("  ✅ Always use parameterized queries!")
end

local db = sqlite3.open(":memory:")

demonstrate_pragmas(db)
demonstrate_safe_queries(db)

print("\n=== Summary ===")
print([[
SQLite ใน Lua เหมาะสำหรับ:
  ✅ Desktop/mobile applications
  ✅ Configuration storage
  ✅ Local caching
  ✅ Prototype development
  ✅ Embedded systems

ไม่เหมาะสำหรับ:
  ❌ High concurrency write workloads
  ❌ Very large datasets (>1TB)
  ❌ Distributed systems
  ❌ Complex analytics queries
]])

db:close()
```

---

## แบบฝึกหัด

**ข้อที่ 1:** สร้าง Todo application โดยใช้ SQLite ที่มีความสามารถ:
- เพิ่ม/ลบ/แก้ไข task
- จัดกลุ่ม task ด้วย tags
- ค้นหา task ด้วย keyword
- Export ข้อมูลเป็น JSON

**ข้อที่ 2:** สร้าง Simple Event Sourcing system ที่:
- บันทึก events ทุกอย่างลง SQLite
- Rebuild state จาก events
- Query events ตามช่วงเวลา
- Snapshot state เป็น optimization

**ข้อที่ 3:** พัฒนา Cache system ที่:
- รองรับ TTL ต่างกันต่อ key
- Implement LRU eviction
- มี statistics (hit rate, miss rate)
- รองรับ namespace (user:*, session:*, etc.)

**ข้อที่ 4:** สร้าง Database Migration tool ที่:
- อ่าน migration files จาก directory
- Apply migrations ตาม version order
- รองรับ rollback
- แสดง migration status

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- **SQLite พื้นฐาน**: การเปิด/ปิด database, CRUD operations
- **Prepared Statements**: ป้องกัน SQL injection, เพิ่มประสิทธิภาพ
- **Transactions**: ACID properties, batch operations
- **Indexes**: Query optimization
- **ORM Pattern**: Abstract database operations
- **Document Store**: NoSQL pattern ด้วย SQLite JSON
- **Key-Value Store**: Simple persistent KV storage
- **Connection Pool**: Manage multiple connections
- **Caching**: SQLite-backed cache ด้วย TTL
- **Repository Pattern**: Clean code architecture
- **Migration System**: Schema version management

**ถัดไป**: บทที่ 61 จะเรียนรู้เกี่ยวกับ LÖVE2D สำหรับพัฒนาเกม 2D ด้วย Lua
