# บทที่ 59: MySQL และ Database Patterns

## บทนำ

MySQL เป็น relational database ที่นิยมใช้กันอย่างแพร่หลาย ในบทนี้จะเรียนรู้การใช้งาน MySQL จาก Lua และ patterns สำคัญในการออกแบบ data access layer

---

## ตัวอย่างที่ 1: MySQL Connection Management

```lua
-- MySQL Connection configuration and management
local MySQLConfig = {}

-- Connection config
function MySQLConfig.create(params)
    return {
        host         = params.host or "127.0.0.1",
        port         = params.port or 3306,
        user         = params.user or "root",
        password     = params.password or "",
        database     = params.database or "myapp",
        charset      = params.charset or "utf8mb4",
        max_packet_size = params.max_packet_size or (1024 * 1024),
        timeout      = params.timeout or 1000,  -- ms
        ssl          = params.ssl or false,
        ssl_verify   = params.ssl_verify or false,
        auto_reconnect = params.auto_reconnect ~= false,
    }
end

-- Create DSN string
function MySQLConfig.to_dsn(config)
    return string.format("mysql://%s:%s@%s:%d/%s?charset=%s",
        config.user,
        config.password ~= "" and "***" or "",
        config.host,
        config.port,
        config.database,
        config.charset
    )
end

-- Parse DSN
function MySQLConfig.parse_dsn(dsn)
    local user, pass, host, port, dbname, opts =
        dsn:match("^mysql://([^:]+):([^@]*)@([^:]+):(%d+)/([^?]+)%??(.*)")
    
    return MySQLConfig.create({
        host     = host,
        port     = tonumber(port),
        user     = user,
        password = pass,
        database = dbname,
    })
end

-- Mock MySQL client
local MockMySQL = {}
MockMySQL.__index = MockMySQL

-- Shared mock database
local mysql_tables = {
    users = {
        { id=1, name="Alice", email="alice@example.com", age=30, status="active", created_at="2024-01-01" },
        { id=2, name="Bob",   email="bob@example.com",   age=25, status="active", created_at="2024-01-10" },
        { id=3, name="Charlie", email="charlie@example.com", age=35, status="inactive", created_at="2024-02-01" },
    },
    orders = {
        { id=1, user_id=1, product="Laptop",  amount=35000, status="delivered", created_at="2024-03-01" },
        { id=2, user_id=1, product="Mouse",   amount=1500,  status="delivered", created_at="2024-03-02" },
        { id=3, user_id=2, product="Keyboard", amount=2500, status="pending",   created_at="2024-03-15" },
    },
    products = {
        { id=1, name="Laptop",   price=35000, stock=50,  category="electronics" },
        { id=2, name="Mouse",    price=1500,  stock=200, category="accessories" },
        { id=3, name="Keyboard", price=2500,  stock=150, category="accessories" },
        { id=4, name="Monitor",  price=15000, stock=30,  category="electronics" },
    },
}

local auto_inc = { users=4, orders=4, products=5 }

function MockMySQL.connect(config)
    local self = setmetatable({}, MockMySQL)
    self.config     = config
    self.connected  = true
    self.in_tx      = false
    self.tx_savepoints = {}
    self.stmts      = {}
    print(string.format("[MySQL] Connected to %s@%s/%s",
        config.user, config.host, config.database))
    return self, nil
end

function MockMySQL:query(sql, params)
    if not self.connected then return nil, "Not connected" end
    
    -- Param substitution (? placeholders)
    if params then
        local pi = 1
        sql = sql:gsub("%?", function()
            local val = params[pi]
            pi = pi + 1
            if val == nil then return "NULL"
            elseif type(val) == "number" then return tostring(val)
            elseif type(val) == "boolean" then return val and "1" or "0"
            else return "'" .. tostring(val):gsub("'", "\\'") .. "'"
            end
        end)
    end
    
    local cmd = sql:match("^%s*(%u+)")
    if cmd == "SELECT" then return self:_select(sql)
    elseif cmd == "INSERT" then return self:_insert(sql)
    elseif cmd == "UPDATE" then return self:_update(sql)
    elseif cmd == "DELETE" then return self:_delete(sql)
    elseif cmd == "BEGIN" or cmd == "START" then
        self.in_tx = true
        return { affected_rows = 0 }, nil
    elseif cmd == "COMMIT" then
        self.in_tx = false
        return { affected_rows = 0 }, nil
    elseif cmd == "ROLLBACK" then
        self.in_tx = false
        return { affected_rows = 0 }, nil
    elseif cmd == "SET" then
        return { affected_rows = 0 }, nil
    end
    
    return nil, "Unsupported: " .. (cmd or sql:sub(1,20))
end

function MockMySQL:_select(sql)
    local tname = sql:match("[Ff][Rr][Oo][Mm]%s+`?(%w+)`?") or
                  sql:match("[Ff][Rr][Oo][Mm]%s+(%w+)")
    local tbl = mysql_tables[tname]
    if not tbl then return { rows={}, num_rows=0 }, nil end
    
    local rows = {}
    local where_val = sql:match("[Ww][Hh][Ee][Rr][Ee]%s+`?id`?%s*=%s*'?(%d+)'?")
    local where_uid = sql:match("[Ww][Hh][Ee][Rr][Ee]%s+`?user_id`?%s*=%s*'?(%d+)'?")
    local where_status = sql:match("[Ww][Hh][Ee][Rr][Ee]%s+`?status`?%s*=%s*'([^']+)'")
    
    for _, row in ipairs(tbl) do
        local ok = true
        if where_val and row.id ~= tonumber(where_val) then ok = false end
        if where_uid and row.user_id ~= tonumber(where_uid) then ok = false end
        if where_status and row.status ~= where_status then ok = false end
        if ok then table.insert(rows, row) end
    end
    
    local limit = sql:match("[Ll][Ii][Mm][Ii][Tt]%s+(%d+)")
    if limit then
        while #rows > tonumber(limit) do table.remove(rows) end
    end
    
    return { rows = rows, num_rows = #rows }, nil
end

function MockMySQL:_insert(sql)
    local tname = sql:match("[Ii][Nn][Tt][Oo]%s+`?(%w+)`?")
    local tbl = mysql_tables[tname]
    if not tbl then return nil, "Table not found: " .. (tname or "?") end
    
    local new_id = auto_inc[tname] or 100
    auto_inc[tname] = new_id + 1
    
    local new_row = { id = new_id }
    table.insert(tbl, new_row)
    
    return { affected_rows = 1, insert_id = new_id }, nil
end

function MockMySQL:_update(sql)
    local tname = sql:match("[Uu][Pp][Dd][Aa][Tt][Ee]%s+`?(%w+)`?")
    local tbl = mysql_tables[tname]
    if not tbl then return nil, "Table not found" end
    
    local affected = 0
    local where_id = sql:match("[Ww][Hh][Ee][Rr][Ee]%s+`?id`?%s*=%s*'?(%d+)'?")
    
    for _, row in ipairs(tbl) do
        if not where_id or row.id == tonumber(where_id) then
            -- Parse SET clause
            local set_pairs = sql:match("[Ss][Ee][Tt]%s+(.-)%s+[Ww][Hh][Ee][Rr][Ee]") or
                              sql:match("[Ss][Ee][Tt]%s+(.*)")
            if set_pairs then
                for col, val in set_pairs:gmatch("`?(%w+)`?%s*=%s*'?([^,']+)'?") do
                    if col ~= "WHERE" then
                        row[col] = tonumber(val) or val
                    end
                end
            end
            affected = affected + 1
        end
    end
    
    return { affected_rows = affected }, nil
end

function MockMySQL:_delete(sql)
    local tname = sql:match("[Ff][Rr][Oo][Mm]%s+`?(%w+)`?")
    local tbl = mysql_tables[tname]
    if not tbl then return nil, "Table not found" end
    
    local where_id = sql:match("[Ww][Hh][Ee][Rr][Ee]%s+`?id`?%s*=%s*'?(%d+)'?")
    local deleted = 0
    
    for i = #tbl, 1, -1 do
        if not where_id or tbl[i].id == tonumber(where_id) then
            table.remove(tbl, i)
            deleted = deleted + 1
            if where_id then break end
        end
    end
    
    return { affected_rows = deleted }, nil
end

function MockMySQL:prepare(sql)
    local id = tostring(#self.stmts + 1)
    self.stmts[id] = sql
    return { id = id, sql = sql }, nil
end

function MockMySQL:execute(stmt, params)
    if type(stmt) == "table" then
        return self:query(stmt.sql, params)
    end
    local sql = self.stmts[tostring(stmt)]
    if not sql then return nil, "Statement not found" end
    return self:query(sql, params)
end

function MockMySQL:close()
    self.connected = false
    print("[MySQL] Disconnected")
end

-- ทดสอบ
print("=== MySQL Connection ===")
local config = MySQLConfig.create({
    host = "127.0.0.1",
    user = "root",
    password = "secret",
    database = "myapp",
})
print("DSN:", MySQLConfig.to_dsn(config))

local db, err = MockMySQL.connect(config)

-- Basic SELECT
local res, qerr = db:query("SELECT * FROM users LIMIT 3")
if res then
    print(string.format("\nFound %d users:", res.num_rows))
    for _, u in ipairs(res.rows) do
        print(string.format("  [%d] %s <%s>", u.id, u.name, u.email))
    end
end
db:close()
```

---

## ตัวอย่างที่ 2: CRUD Operations

```lua
-- Complete CRUD operations
local CRUD = {}
CRUD.__index = CRUD

function CRUD.new(db, table_name, primary_key)
    local self = setmetatable({}, CRUD)
    self.db  = db
    self.tbl = table_name
    self.pk  = primary_key or "id"
    return self
end

-- CREATE
function CRUD:insert(data)
    local cols, vals, placeholders = {}, {}, {}
    for col, val in pairs(data) do
        table.insert(cols, "`" .. col .. "`")
        table.insert(vals, val)
        table.insert(placeholders, "?")
    end
    
    local sql = string.format("INSERT INTO `%s` (%s) VALUES (%s)",
        self.tbl,
        table.concat(cols, ", "),
        table.concat(placeholders, ", ")
    )
    
    local res, err = self.db:query(sql, vals)
    if err then return nil, err end
    return res.insert_id, nil
end

-- READ
function CRUD:find(id)
    local sql = string.format("SELECT * FROM `%s` WHERE `%s` = ? LIMIT 1",
        self.tbl, self.pk)
    local res, err = self.db:query(sql, { id })
    if err then return nil, err end
    return res.rows[1], nil
end

function CRUD:find_all(conditions, options)
    local where_parts, params = {}, {}
    
    if conditions then
        for col, val in pairs(conditions) do
            if val == nil then
                table.insert(where_parts, "`" .. col .. "` IS NULL")
            else
                table.insert(where_parts, "`" .. col .. "` = ?")
                table.insert(params, val)
            end
        end
    end
    
    options = options or {}
    local sql = string.format("SELECT * FROM `%s`", self.tbl)
    
    if #where_parts > 0 then
        sql = sql .. " WHERE " .. table.concat(where_parts, " AND ")
    end
    
    if options.order_by then
        sql = sql .. " ORDER BY `" .. options.order_by .. "`"
        if options.order_dir then sql = sql .. " " .. options.order_dir end
    end
    
    if options.limit then
        sql = sql .. " LIMIT " .. options.limit
    end
    if options.offset then
        sql = sql .. " OFFSET " .. options.offset
    end
    
    local res, err = self.db:query(sql, params)
    if err then return nil, err end
    return res.rows, nil
end

-- UPDATE
function CRUD:update(id, data)
    local set_parts, params = {}, {}
    for col, val in pairs(data) do
        table.insert(set_parts, "`" .. col .. "` = ?")
        table.insert(params, val)
    end
    table.insert(params, id)
    
    local sql = string.format("UPDATE `%s` SET %s WHERE `%s` = ?",
        self.tbl,
        table.concat(set_parts, ", "),
        self.pk
    )
    
    local res, err = self.db:query(sql, params)
    if err then return nil, err end
    return res.affected_rows, nil
end

-- DELETE
function CRUD:delete(id)
    local sql = string.format("DELETE FROM `%s` WHERE `%s` = ?", self.tbl, self.pk)
    local res, err = self.db:query(sql, { id })
    if err then return nil, err end
    return res.affected_rows, nil
end

-- UPSERT (INSERT ... ON DUPLICATE KEY UPDATE)
function CRUD:upsert(data, update_cols)
    local cols, vals, placeholders, updates = {}, {}, {}, {}
    
    for col, val in pairs(data) do
        table.insert(cols, "`" .. col .. "`")
        table.insert(vals, val)
        table.insert(placeholders, "?")
    end
    
    local update_list = update_cols or cols
    for _, col in ipairs(update_list) do
        local clean = col:gsub("`", "")
        if clean ~= self.pk then
            table.insert(updates, string.format("%s = VALUES(%s)", col, col))
        end
    end
    
    local sql = string.format(
        "INSERT INTO `%s` (%s) VALUES (%s) ON DUPLICATE KEY UPDATE %s",
        self.tbl,
        table.concat(cols, ", "),
        table.concat(placeholders, ", "),
        table.concat(updates, ", ")
    )
    
    return self.db:query(sql, vals)
end

-- ทดสอบ
print("=== CRUD Operations ===")
local config = MySQLConfig.create({ host="127.0.0.1", database="myapp" })
local db2 = MockMySQL.connect(config)

local user_crud = CRUD.new(db2, "users")
local order_crud = CRUD.new(db2, "orders")

-- INSERT
print("\nInsert new user:")
local new_id, ins_err = user_crud:insert({
    name  = "Dave",
    email = "dave@example.com",
    age   = 28,
    status = "active",
})
print("New ID:", new_id, "Error:", ins_err)

-- SELECT
print("\nFind user by ID:")
local user, find_err = user_crud:find(1)
if user then
    print(string.format("  Found: %s <%s>", user.name, user.email))
end

-- Find all
print("\nFind active users:")
local users, _ = user_crud:find_all({ status = "active" }, { order_by = "name", limit = 5 })
for _, u in ipairs(users or {}) do
    print(string.format("  [%d] %s", u.id, u.name))
end

-- UPDATE
print("\nUpdate user status:")
local affected, upd_err = user_crud:update(2, { status = "inactive", age = 26 })
print("Affected rows:", affected, upd_err)

-- DELETE
print("\nDelete user:")
local del_affected, _ = user_crud:delete(3)
print("Deleted:", del_affected)

-- Orders
print("\nUser 1's orders:")
local orders, _ = order_crud:find_all({ user_id = 1 })
for _, o in ipairs(orders or {}) do
    print(string.format("  [%d] %s - %.0f THB (%s)",
        o.id, o.product, o.amount, o.status))
end

db2:close()
```

---

## ตัวอย่างที่ 3: Transactions

```lua
-- MySQL Transactions
local TxManager = {}
TxManager.__index = TxManager

function TxManager.new(db)
    local self = setmetatable({}, TxManager)
    self.db      = db
    self.active  = false
    self.log     = {}
    return self
end

function TxManager:begin(isolation_level)
    isolation_level = isolation_level or "REPEATABLE READ"
    self.db:query("SET TRANSACTION ISOLATION LEVEL " .. isolation_level)
    self.db:query("BEGIN")
    self.active = true
    self.log = {}
    print("[TX] BEGIN (isolation: " .. isolation_level .. ")")
    return self
end

function TxManager:exec(sql, params)
    if not self.active then error("No active transaction") end
    local res, err = self.db:query(sql, params)
    if err then
        self:rollback()
        error("Query failed: " .. err)
    end
    table.insert(self.log, { sql = sql:sub(1, 80) })
    return res
end

function TxManager:commit()
    self.db:query("COMMIT")
    self.active = false
    print(string.format("[TX] COMMIT (%d operations)", #self.log))
    self.log = {}
end

function TxManager:rollback()
    self.db:query("ROLLBACK")
    self.active = false
    print(string.format("[TX] ROLLBACK (%d operations undone)", #self.log))
    self.log = {}
end

-- Run transaction with automatic rollback on error
function TxManager.run(db, fn, isolation)
    local tx = TxManager.new(db)
    tx:begin(isolation)
    
    local ok, result = pcall(fn, tx)
    
    if ok then
        tx:commit()
        return result, nil
    else
        tx:rollback()
        return nil, tostring(result)
    end
end

-- ทดสอบ
print("=== MySQL Transactions ===")
local config2 = MySQLConfig.create({ database = "myapp" })
local db3 = MockMySQL.connect(config2)

-- Successful transaction
print("\nTransaction 1: Order processing")
TxManager.run(db3, function(tx)
    -- Check stock
    local product_res = tx:exec("SELECT * FROM products WHERE id = 1")
    print("  Checking product stock...")
    
    -- Create order
    local order_res = tx:exec(
        "INSERT INTO orders (user_id, product, amount, status) VALUES (?, ?, ?, ?)",
        { 1, "Laptop", 35000, "pending" }
    )
    print("  Order created, ID:", order_res.insert_id)
    
    -- Decrease stock
    tx:exec("UPDATE products SET stock = stock - ? WHERE id = ?", { 1, 1 })
    print("  Stock updated")
    
    return order_res.insert_id
end)

-- Failed transaction
print("\nTransaction 2: Insufficient stock (will rollback)")
local result, err = TxManager.run(db3, function(tx)
    tx:exec("UPDATE products SET stock = stock - 1000 WHERE id = 2")
    -- Simulate validation
    error("Cannot have negative stock!")
end)
print("Error:", err)

db3:close()
```

---

## ตัวอย่างที่ 4: Stored Procedures with MySQL

```lua
-- MySQL Stored Procedures
local MySQLProcs = {}

-- Define stored procedures
local procedures = {
    get_user_orders = [[
CREATE PROCEDURE get_user_orders(IN p_user_id INT, IN p_status VARCHAR(50))
BEGIN
    SELECT 
        o.id,
        o.product,
        o.amount,
        o.status,
        o.created_at,
        u.name as customer_name
    FROM orders o
    JOIN users u ON u.id = o.user_id
    WHERE o.user_id = p_user_id
      AND (p_status IS NULL OR o.status = p_status)
    ORDER BY o.created_at DESC;
END
]],

    calculate_user_stats = [[
CREATE PROCEDURE calculate_user_stats(
    IN  p_user_id INT,
    OUT p_order_count INT,
    OUT p_total_spent DECIMAL(10,2),
    OUT p_avg_order  DECIMAL(10,2)
)
BEGIN
    SELECT 
        COUNT(*),
        COALESCE(SUM(amount), 0),
        COALESCE(AVG(amount), 0)
    INTO p_order_count, p_total_spent, p_avg_order
    FROM orders
    WHERE user_id = p_user_id
      AND status != 'cancelled';
END
]],

    bulk_update_prices = [[
CREATE PROCEDURE bulk_update_prices(
    IN p_category VARCHAR(100),
    IN p_multiplier DECIMAL(4,2)
)
BEGIN
    DECLARE affected INT DEFAULT 0;
    
    UPDATE products 
    SET price = ROUND(price * p_multiplier, 2)
    WHERE category = p_category;
    
    SET affected = ROW_COUNT();
    SELECT affected as updated_count;
END
]],
}

-- Call stored procedure (simple)
function MySQLProcs.call(db, proc_name, params)
    local placeholders = {}
    for _ in ipairs(params or {}) do
        table.insert(placeholders, "?")
    end
    
    local sql = string.format("CALL %s(%s)", proc_name,
        table.concat(placeholders, ", "))
    
    print("[MySQL] " .. sql)
    return db:query(sql, params)
end

-- Call with OUT parameters
function MySQLProcs.call_with_out(db, proc_name, in_params, out_vars)
    -- Set user variables for output
    for _, var in ipairs(out_vars) do
        db:query("SET @" .. var .. " = NULL")
    end
    
    -- Build CALL with @var for outputs
    local placeholders = {}
    for _ in ipairs(in_params) do
        table.insert(placeholders, "?")
    end
    for _, var in ipairs(out_vars) do
        table.insert(placeholders, "@" .. var)
    end
    
    local call_sql = string.format("CALL %s(%s)", proc_name,
        table.concat(placeholders, ", "))
    db:query(call_sql, in_params)
    
    -- Read output variables
    local out_sql = "SELECT " .. table.concat(
        (function()
            local s = {}
            for _, var in ipairs(out_vars) do
                table.insert(s, "@" .. var .. " as " .. var)
            end
            return s
        end)(), ", ")
    
    return db:query(out_sql)
end

-- ทดสอบ
print("=== MySQL Stored Procedures ===")
print("\nDefined procedures:")
for name in pairs(procedures) do
    print("  - " .. name)
end

local config3 = MySQLConfig.create({ database = "myapp" })
local db4 = MockMySQL.connect(config3)

-- Call procedure
print("\nCalling get_user_orders:")
MySQLProcs.call(db4, "get_user_orders", { 1, "delivered" })

-- Call with OUT params
print("\nCalling calculate_user_stats:")
local res = MySQLProcs.call_with_out(
    db4,
    "calculate_user_stats",
    { 1 },
    { "order_count", "total_spent", "avg_order" }
)

-- Show procedure definition
print("\nProcedure: get_user_orders")
print(procedures.get_user_orders:sub(1, 200))

db4:close()
```

---

## ตัวอย่างที่ 5: Batch Inserts

```lua
-- Batch insert optimization
local BatchInsert = {}

-- Build multi-row INSERT
function BatchInsert.build(table_name, columns, rows, options)
    options = options or {}
    if #rows == 0 then return nil, "No rows to insert" end
    
    local col_str = table.concat(
        (function()
            local cs = {}
            for _, c in ipairs(columns) do
                table.insert(cs, "`" .. c .. "`")
            end
            return cs
        end)(), ", "
    )
    
    local all_params = {}
    local value_groups = {}
    
    for _, row in ipairs(rows) do
        local placeholders = {}
        for _, col in ipairs(columns) do
            table.insert(placeholders, "?")
            table.insert(all_params, row[col])
        end
        table.insert(value_groups, "(" .. table.concat(placeholders, ", ") .. ")")
    end
    
    local keyword = options.ignore and "INSERT IGNORE" or "INSERT"
    
    local sql = string.format("%s INTO `%s` (%s)\nVALUES\n  %s",
        keyword, table_name, col_str,
        table.concat(value_groups, ",\n  ")
    )
    
    return sql, all_params
end

-- Chunk large inserts
function BatchInsert.chunked(db, table_name, columns, all_rows, chunk_size)
    chunk_size = chunk_size or 1000
    local total_inserted = 0
    local chunks = 0
    
    for i = 1, #all_rows, chunk_size do
        local chunk = {}
        for j = i, math.min(i + chunk_size - 1, #all_rows) do
            table.insert(chunk, all_rows[j])
        end
        
        local sql, params = BatchInsert.build(table_name, columns, chunk)
        if sql then
            local res, err = db:query(sql, params)
            if err then
                return total_inserted, err
            end
            total_inserted = total_inserted + (res.affected_rows or #chunk)
            chunks = chunks + 1
        end
    end
    
    print(string.format("[Batch] Inserted %d rows in %d chunks", total_inserted, chunks))
    return total_inserted, nil
end

-- ทดสอบ
print("=== Batch Inserts ===")

local columns = { "name", "email", "age", "status" }
local sample_rows = {}
for i = 1, 10 do
    table.insert(sample_rows, {
        name   = "User" .. i,
        email  = "user" .. i .. "@example.com",
        age    = 20 + i,
        status = "active",
    })
end

-- Build SQL
local sql, params = BatchInsert.build("users", columns, sample_rows)
print("SQL preview:")
print(sql:sub(1, 200) .. "...")
print(string.format("Parameters: %d values", #params))

-- Chunked insert
print("\nChunked insert (chunk size 3):")
local config4 = MySQLConfig.create({ database = "myapp" })
local db5 = MockMySQL.connect(config4)

local inserted, _ = BatchInsert.chunked(db5, "users", columns, sample_rows, 3)
print("Total inserted:", inserted)

-- INSERT IGNORE
print("\nINSERT IGNORE (skip duplicates):")
local sql2, _ = BatchInsert.build("users", {"name", "email"}, {
    { name="Alice", email="alice@example.com" },  -- duplicate
    { name="New",   email="new@example.com" },
}, { ignore = true })
print(sql2:sub(1, 150))

db5:close()
```

---

## ตัวอย่างที่ 6: Connection Pool

```lua
-- MySQL Connection Pool
local MySQLPool = {}
MySQLPool.__index = MySQLPool

function MySQLPool.new(config, options)
    local self = setmetatable({}, MySQLPool)
    self.config     = config
    self.min_conns  = options.min or 2
    self.max_conns  = options.max or 10
    self.idle_time  = options.idle_time or 300
    self.pool       = {}   -- available connections
    self.active     = {}   -- in-use connections
    self.wait_queue = {}
    self.created    = 0
    self.total_queries = 0
    
    -- Initialize minimum connections
    for _ = 1, self.min_conns do
        self:_new_connection()
    end
    
    return self
end

local pool_conn_id = 0

function MySQLPool:_new_connection()
    pool_conn_id = pool_conn_id + 1
    local conn = MockMySQL.connect(self.config)
    conn._pool_id = pool_conn_id
    conn._created_at = os.time()
    conn._last_used  = os.time()
    conn._queries    = 0
    self.created = self.created + 1
    table.insert(self.pool, conn)
    return conn
end

function MySQLPool:get()
    -- Return idle connection
    if #self.pool > 0 then
        local conn = table.remove(self.pool)
        conn._in_use = true
        self.active[conn._pool_id] = conn
        return conn, nil
    end
    
    -- Create new if under max
    local active_count = 0
    for _ in pairs(self.active) do active_count = active_count + 1 end
    
    if active_count + #self.pool < self.max_conns then
        local conn = self:_new_connection()
        table.remove(self.pool)
        conn._in_use = true
        self.active[conn._pool_id] = conn
        return conn, nil
    end
    
    return nil, "Pool exhausted (max=" .. self.max_conns .. ")"
end

function MySQLPool:put(conn)
    conn._in_use  = false
    conn._last_used = os.time()
    self.active[conn._pool_id] = nil
    
    -- Keep if still fresh
    if os.time() - conn._created_at < self.idle_time then
        table.insert(self.pool, conn)
    else
        conn:close()
        -- Replenish if below min
        if #self.pool < self.min_conns then
            self:_new_connection()
        end
    end
end

function MySQLPool:execute(fn)
    local conn, err = self:get()
    if not conn then return nil, err end
    
    local ok, result = pcall(fn, conn)
    self:put(conn)
    
    if not ok then return nil, tostring(result) end
    return result, nil
end

function MySQLPool:stats()
    local active = 0
    for _ in pairs(self.active) do active = active + 1 end
    return {
        idle    = #self.pool,
        active  = active,
        total   = #self.pool + active,
        max     = self.max_conns,
        created = self.created,
    }
end

function MySQLPool:close_all()
    for _, conn in ipairs(self.pool) do conn:close() end
    for _, conn in pairs(self.active) do conn:close() end
    self.pool   = {}
    self.active = {}
    print("[Pool] All connections closed")
end

-- ทดสอบ
print("=== MySQL Connection Pool ===")
local config5 = MySQLConfig.create({ database = "myapp" })
local pool = MySQLPool.new(config5, { min = 2, max = 5 })

print("\nInitial pool:", pool:stats().idle, "idle connections")

-- Simulate requests
print("\nAcquiring connections:")
local conns = {}
for i = 1, 4 do
    local conn, err = pool:get()
    if conn then
        table.insert(conns, conn)
        print(string.format("  Got connection #%d", conn._pool_id))
    else
        print("  Failed:", err)
    end
end

print("Stats:", pool:stats().idle, "idle,", pool:stats().active, "active")

-- Return connections
for _, conn in ipairs(conns) do
    pool:put(conn)
end
print("After return:", pool:stats().idle, "idle")

-- Execute pattern
print("\nExecute pattern:")
pool:execute(function(conn)
    conn:query("SELECT * FROM users LIMIT 3")
    print("  Query executed")
end)

pool:close_all()
```

---

## ตัวอย่างที่ 7: Repository Pattern

```lua
-- Repository Pattern
local BaseRepository = {}
BaseRepository.__index = BaseRepository

function BaseRepository.new(db, table_name, config)
    local self = setmetatable({}, BaseRepository)
    self.db         = db
    self.table      = table_name
    self.pk         = config and config.pk or "id"
    self.timestamps = config and config.timestamps ~= false or true
    return self
end

function BaseRepository:find_by_id(id)
    local sql = string.format("SELECT * FROM `%s` WHERE `%s` = ? LIMIT 1",
        self.table, self.pk)
    local res, err = self.db:query(sql, {id})
    if err then return nil, err end
    return res.rows[1]
end

function BaseRepository:find_by(conditions)
    local wheres, params = {}, {}
    for col, val in pairs(conditions) do
        table.insert(wheres, "`" .. col .. "` = ?")
        table.insert(params, val)
    end
    
    local sql = string.format("SELECT * FROM `%s` WHERE %s",
        self.table, table.concat(wheres, " AND "))
    local res, err = self.db:query(sql, params)
    if err then return nil, err end
    return res.rows
end

function BaseRepository:find_all(options)
    options = options or {}
    local sql = "SELECT * FROM `" .. self.table .. "`"
    local params = {}
    
    if options.where then
        local parts = {}
        for col, val in pairs(options.where) do
            table.insert(parts, "`" .. col .. "` = ?")
            table.insert(params, val)
        end
        sql = sql .. " WHERE " .. table.concat(parts, " AND ")
    end
    
    if options.order_by then
        sql = sql .. " ORDER BY `" .. options.order_by .. "` "
        sql = sql .. (options.desc and "DESC" or "ASC")
    end
    
    if options.limit  then sql = sql .. " LIMIT "  .. options.limit  end
    if options.offset then sql = sql .. " OFFSET " .. options.offset end
    
    local res, err = self.db:query(sql, params)
    if err then return nil, err end
    return res.rows, res.num_rows
end

function BaseRepository:create(data)
    if self.timestamps then
        data.created_at = data.created_at or os.date("%Y-%m-%d %H:%M:%S")
        data.updated_at = data.updated_at or data.created_at
    end
    
    local cols, vals, holders = {}, {}, {}
    for col, val in pairs(data) do
        table.insert(cols, "`" .. col .. "`")
        table.insert(vals, val)
        table.insert(holders, "?")
    end
    
    local sql = string.format("INSERT INTO `%s` (%s) VALUES (%s)",
        self.table,
        table.concat(cols, ", "),
        table.concat(holders, ", ")
    )
    
    local res, err = self.db:query(sql, vals)
    if err then return nil, err end
    
    data[self.pk] = res.insert_id
    return data
end

function BaseRepository:update(id, data)
    if self.timestamps then
        data.updated_at = os.date("%Y-%m-%d %H:%M:%S")
    end
    
    local sets, params = {}, {}
    for col, val in pairs(data) do
        if col ~= self.pk then
            table.insert(sets, "`" .. col .. "` = ?")
            table.insert(params, val)
        end
    end
    table.insert(params, id)
    
    local sql = string.format("UPDATE `%s` SET %s WHERE `%s` = ?",
        self.table, table.concat(sets, ", "), self.pk)
    
    local res, err = self.db:query(sql, params)
    if err then return nil, err end
    return res.affected_rows > 0
end

function BaseRepository:delete(id)
    local sql = string.format("DELETE FROM `%s` WHERE `%s` = ?", self.table, self.pk)
    local res, err = self.db:query(sql, {id})
    if err then return nil, err end
    return res.affected_rows > 0
end

function BaseRepository:count(conditions)
    local sql = "SELECT COUNT(*) as cnt FROM `" .. self.table .. "`"
    local params = {}
    
    if conditions then
        local parts = {}
        for col, val in pairs(conditions) do
            table.insert(parts, "`" .. col .. "` = ?")
            table.insert(params, val)
        end
        sql = sql .. " WHERE " .. table.concat(parts, " AND ")
    end
    
    local res, err = self.db:query(sql, params)
    if err then return 0, err end
    return res.rows[1] and res.rows[1].cnt or 0
end

function BaseRepository:exists(conditions)
    local count, err = self:count(conditions)
    return count > 0, err
end

-- Specialized repository: UserRepository
local UserRepository = setmetatable({}, { __index = BaseRepository })
UserRepository.__index = UserRepository

function UserRepository.new(db)
    local self = BaseRepository.new(db, "users")
    return setmetatable(self, UserRepository)
end

function UserRepository:find_by_email(email)
    local results = self:find_by({ email = email })
    return results and results[1]
end

function UserRepository:find_active()
    local rows, _ = self:find_all({
        where    = { status = "active" },
        order_by = "name",
    })
    return rows or {}
end

function UserRepository:deactivate(user_id)
    return self:update(user_id, { status = "inactive" })
end

-- ทดสอบ
print("=== Repository Pattern ===")
local config6 = MySQLConfig.create({ database = "myapp" })
local db6 = MockMySQL.connect(config6)

local user_repo  = UserRepository.new(db6)
local order_repo = BaseRepository.new(db6, "orders")

-- Find by email
print("\nFind by email:")
local alice = user_repo:find_by_email("alice@example.com")
print("Found:", alice and (alice.name .. " (id=" .. alice.id .. ")") or "nil")

-- Find active users
print("\nActive users:")
local active = user_repo:find_active()
for _, u in ipairs(active) do
    print(string.format("  [%d] %s - %s", u.id, u.name, u.status))
end

-- Create
print("\nCreate user:")
local new_user, _ = user_repo:create({
    name   = "Eve",
    email  = "eve@example.com",
    status = "active",
})
print("Created:", new_user and ("id=" .. new_user.id) or "failed")

-- Count
local active_count, _ = user_repo:count({ status = "active" })
print("Active users count:", active_count)

-- Exists
local exists, _ = user_repo:exists({ email = "bob@example.com" })
print("Bob exists:", exists)

db6:close()
```

---

## ตัวอย่างที่ 8: Active Record Pattern

```lua
-- Active Record Pattern
local ActiveRecord = {}
ActiveRecord.__index = ActiveRecord

local ar_db = nil  -- shared db connection

function ActiveRecord.set_db(db)
    ar_db = db
end

function ActiveRecord.define(class_name, config)
    local cls = {}
    cls.__index = cls
    cls._table      = config.table
    cls._pk         = config.pk or "id"
    cls._attributes = config.attributes or {}
    
    setmetatable(cls, {
        __index = ActiveRecord,
        __call  = function(t, data)
            return t.new(data)
        end
    })
    
    -- Constructor
    function cls.new(data)
        local instance = setmetatable({}, cls)
        instance._attrs  = {}
        instance._dirty  = {}
        instance._persisted = false
        
        -- Set defaults
        for attr_name, attr_def in pairs(cls._attributes) do
            instance._attrs[attr_name] = attr_def.default
        end
        
        -- Apply provided data
        if data then
            for k, v in pairs(data) do
                instance._attrs[k] = v
            end
            if data[cls._pk] then
                instance._persisted = true
            end
        end
        
        return instance
    end
    
    -- Attribute accessors
    for attr_name in pairs(config.attributes or {}) do
        cls[attr_name] = function(self, value)
            if value ~= nil then
                if self._attrs[attr_name] ~= value then
                    self._dirty[attr_name] = true
                    self._attrs[attr_name] = value
                end
                return self
            end
            return self._attrs[attr_name]
        end
    end
    
    -- Finders
    function cls.find(id)
        local sql = string.format("SELECT * FROM `%s` WHERE `%s` = ? LIMIT 1",
            cls._table, cls._pk)
        local res, err = ar_db:query(sql, {id})
        if err or not res.rows[1] then return nil end
        
        local instance = cls.new(res.rows[1])
        instance._persisted = true
        instance._dirty = {}
        return instance
    end
    
    function cls.where(conditions)
        local parts, params = {}, {}
        for col, val in pairs(conditions) do
            table.insert(parts, "`" .. col .. "` = ?")
            table.insert(params, val)
        end
        local sql = string.format("SELECT * FROM `%s` WHERE %s",
            cls._table, table.concat(parts, " AND "))
        local res, err = ar_db:query(sql, params)
        if err then return {} end
        
        local results = {}
        for _, row in ipairs(res.rows) do
            local inst = cls.new(row)
            inst._persisted = true
            inst._dirty = {}
            table.insert(results, inst)
        end
        return results
    end
    
    function cls.all(limit)
        local sql = "SELECT * FROM `" .. cls._table .. "`"
        if limit then sql = sql .. " LIMIT " .. limit end
        local res, _ = ar_db:query(sql)
        local results = {}
        for _, row in ipairs(res and res.rows or {}) do
            local inst = cls.new(row)
            inst._persisted = true
            table.insert(results, inst)
        end
        return results
    end
    
    -- Instance methods
    function cls:save()
        if self._persisted then
            -- UPDATE
            if not next(self._dirty) then
                print("[AR] No changes, skip save")
                return true
            end
            
            local sets, params = {}, {}
            for col in pairs(self._dirty) do
                table.insert(sets, "`" .. col .. "` = ?")
                table.insert(params, self._attrs[col])
            end
            table.insert(params, self._attrs[cls._pk])
            
            local sql = string.format("UPDATE `%s` SET %s WHERE `%s` = ?",
                cls._table, table.concat(sets, ", "), cls._pk)
            local res, err = ar_db:query(sql, params)
            if err then return false, err end
            
            self._dirty = {}
            return true
        else
            -- INSERT
            local cols, vals, holders = {}, {}, {}
            for col, val in pairs(self._attrs) do
                if val ~= nil then
                    table.insert(cols, "`" .. col .. "`")
                    table.insert(vals, val)
                    table.insert(holders, "?")
                end
            end
            
            local sql = string.format("INSERT INTO `%s` (%s) VALUES (%s)",
                cls._table,
                table.concat(cols, ", "),
                table.concat(holders, ", ")
            )
            local res, err = ar_db:query(sql, vals)
            if err then return false, err end
            
            self._attrs[cls._pk] = res.insert_id
            self._persisted = true
            self._dirty = {}
            return true
        end
    end
    
    function cls:destroy()
        if not self._persisted then return false end
        local sql = string.format("DELETE FROM `%s` WHERE `%s` = ?",
            cls._table, cls._pk)
        local res, err = ar_db:query(sql, { self._attrs[cls._pk] })
        if err then return false, err end
        self._persisted = false
        return true
    end
    
    function cls:to_table()
        local t = {}
        for k, v in pairs(self._attrs) do
            t[k] = v
        end
        return t
    end
    
    function cls:is_new()
        return not self._persisted
    end
    
    function cls:is_dirty()
        return next(self._dirty) ~= nil
    end
    
    return cls
end

-- Define models
local config7 = MySQLConfig.create({ database = "myapp" })
local db7 = MockMySQL.connect(config7)
ActiveRecord.set_db(db7)

local User = ActiveRecord.define("User", {
    table = "users",
    attributes = {
        id     = { type = "integer" },
        name   = { type = "string" },
        email  = { type = "string" },
        age    = { type = "integer" },
        status = { type = "string", default = "active" },
    },
})

-- ทดสอบ
print("=== Active Record Pattern ===")

-- Create and save
local user = User.new({ name = "Frank", email = "frank@example.com", age = 33 })
print("Is new:", user:is_new())
user:save()
print("After save, is new:", user:is_new())
print("ID:", user:id())

-- Find
local found = User.find(1)
if found then
    print("\nFound user:", found:name(), "| email:", found:email())
end

-- Update via attribute setters
found:name("Alice Admin"):age(31):save()
print("Updated:", found:name(), found:age())

-- Where
local active_users = User.where({ status = "active" })
print("\nActive users:", #active_users)

-- All
local all_users = User.all(3)
print("All users (limit 3):", #all_users)
for _, u in ipairs(all_users) do
    print(string.format("  [%d] %s (%s)", u:id() or 0, u:name() or "?", u:status() or "?"))
end

db7:close()
```

---

## ตัวอย่างที่ 9: Data Mapper Pattern

```lua
-- Data Mapper Pattern separates domain objects from DB logic
local DataMapper = {}

-- Domain object: ไม่รู้จัก database
local User_Domain = {}
User_Domain.__index = User_Domain

function User_Domain.new(data)
    local self = setmetatable({}, User_Domain)
    self.id     = data.id
    self.name   = data.name
    self.email  = data.email
    self.age    = data.age
    self.status = data.status or "active"
    return self
end

function User_Domain:is_adult()
    return (self.age or 0) >= 18
end

function User_Domain:is_active()
    return self.status == "active"
end

function User_Domain:full_label()
    return string.format("%s <%s>", self.name, self.email)
end

-- Mapper: แปลงระหว่าง Domain object และ Database row
local UserMapper = {}
UserMapper.__index = UserMapper

function UserMapper.new(db)
    local self = setmetatable({}, UserMapper)
    self.db    = db
    self.table = "users"
    return self
end

function UserMapper:to_domain(row)
    if not row then return nil end
    return User_Domain.new(row)
end

function UserMapper:to_row(user)
    return {
        id     = user.id,
        name   = user.name,
        email  = user.email,
        age    = user.age,
        status = user.status,
    }
end

function UserMapper:find_by_id(id)
    local res, err = self.db:query(
        "SELECT * FROM users WHERE id = ?", { id })
    if err or not res.rows[1] then return nil, err end
    return self:to_domain(res.rows[1])
end

function UserMapper:find_all()
    local res, err = self.db:query("SELECT * FROM users")
    if err then return nil, err end
    local users = {}
    for _, row in ipairs(res.rows) do
        table.insert(users, self:to_domain(row))
    end
    return users
end

function UserMapper:save(user)
    if user.id then
        local row = self:to_row(user)
        local sets, params = {}, {}
        for col, val in pairs(row) do
            if col ~= "id" then
                table.insert(sets, "`" .. col .. "` = ?")
                table.insert(params, val)
            end
        end
        table.insert(params, user.id)
        local sql = string.format("UPDATE users SET %s WHERE id = ?",
            table.concat(sets, ", "))
        return self.db:query(sql, params)
    else
        local row = self:to_row(user)
        local cols, vals, holders = {}, {}, {}
        for col, val in pairs(row) do
            if col ~= "id" then
                table.insert(cols, "`" .. col .. "`")
                table.insert(vals, val)
                table.insert(holders, "?")
            end
        end
        local sql = string.format("INSERT INTO users (%s) VALUES (%s)",
            table.concat(cols, ", "), table.concat(holders, ", "))
        local res, err = self.db:query(sql, vals)
        if res then user.id = res.insert_id end
        return res, err
    end
end

function UserMapper:delete(id)
    return self.db:query("DELETE FROM users WHERE id = ?", {id})
end

-- ทดสอบ
print("=== Data Mapper Pattern ===")
local config8 = MySQLConfig.create({ database = "myapp" })
local db8 = MockMySQL.connect(config8)

local mapper = UserMapper.new(db8)

-- Find and use domain logic
local user = mapper:find_by_id(1)
if user then
    print("User:", user:full_label())
    print("Is adult:", user:is_adult())
    print("Is active:", user:is_active())
end

-- Find all and filter with domain logic
local all_users = mapper:find_all()
local adults = {}
for _, u in ipairs(all_users or {}) do
    if u:is_adult() and u:is_active() then
        table.insert(adults, u)
    end
end
print(string.format("\nActive adults: %d of %d",
    #adults, #(all_users or {})))

-- Create new user (domain object doesn't know about DB)
local new_user = User_Domain.new({
    name   = "Grace",
    email  = "grace@example.com",
    age    = 27,
    status = "active",
})
print("New user is adult:", new_user:is_adult())
mapper:save(new_user)
print("Saved with id:", new_user.id)

db8:close()
```

---

## ตัวอย่างที่ 10: Unit of Work Pattern

```lua
-- Unit of Work Pattern: tracks changes and commits as one unit
local UnitOfWork = {}
UnitOfWork.__index = UnitOfWork

function UnitOfWork.new(db)
    local self = setmetatable({}, UnitOfWork)
    self.db       = db
    self.new      = {}    -- entities to INSERT
    self.dirty    = {}    -- entities to UPDATE
    self.removed  = {}    -- entities to DELETE
    self._identity_map = {}  -- id -> entity cache
    return self
end

function UnitOfWork:register_new(entity_type, entity)
    if not self.new[entity_type] then
        self.new[entity_type] = {}
    end
    table.insert(self.new[entity_type], entity)
end

function UnitOfWork:register_dirty(entity_type, entity)
    if not self.dirty[entity_type] then
        self.dirty[entity_type] = {}
    end
    local key = entity.id or tostring(entity)
    self.dirty[entity_type][key] = entity
end

function UnitOfWork:register_removed(entity_type, entity_id)
    if not self.removed[entity_type] then
        self.removed[entity_type] = {}
    end
    table.insert(self.removed[entity_type], entity_id)
end

function UnitOfWork:commit()
    -- Begin transaction
    self.db:query("BEGIN")
    
    local total_changes = 0
    
    -- Process INSERTs
    for entity_type, entities in pairs(self.new) do
        for _, entity in ipairs(entities) do
            local cols, vals, holders = {}, {}, {}
            for col, val in pairs(entity) do
                if type(val) ~= "function" then
                    table.insert(cols, "`" .. col .. "`")
                    table.insert(vals, val)
                    table.insert(holders, "?")
                end
            end
            local sql = string.format("INSERT INTO `%s` (%s) VALUES (%s)",
                entity_type,
                table.concat(cols, ", "),
                table.concat(holders, ", ")
            )
            local res, err = self.db:query(sql, vals)
            if err then
                self.db:query("ROLLBACK")
                return false, "Insert failed: " .. err
            end
            if res then entity.id = res.insert_id end
            total_changes = total_changes + 1
        end
    end
    
    -- Process UPDATEs
    for entity_type, entities in pairs(self.dirty) do
        for _, entity in pairs(entities) do
            local sets, params = {}, {}
            for col, val in pairs(entity) do
                if col ~= "id" and type(val) ~= "function" then
                    table.insert(sets, "`" .. col .. "` = ?")
                    table.insert(params, val)
                end
            end
            table.insert(params, entity.id)
            local sql = string.format("UPDATE `%s` SET %s WHERE `id` = ?",
                entity_type, table.concat(sets, ", "))
            local _, err = self.db:query(sql, params)
            if err then
                self.db:query("ROLLBACK")
                return false, "Update failed: " .. err
            end
            total_changes = total_changes + 1
        end
    end
    
    -- Process DELETEs
    for entity_type, ids in pairs(self.removed) do
        for _, id in ipairs(ids) do
            local sql = string.format("DELETE FROM `%s` WHERE `id` = ?", entity_type)
            local _, err = self.db:query(sql, {id})
            if err then
                self.db:query("ROLLBACK")
                return false, "Delete failed: " .. err
            end
            total_changes = total_changes + 1
        end
    end
    
    self.db:query("COMMIT")
    
    -- Clear tracked changes
    self.new     = {}
    self.dirty   = {}
    self.removed = {}
    
    print(string.format("[UoW] Committed %d changes", total_changes))
    return true, nil
end

function UnitOfWork:rollback()
    self.new     = {}
    self.dirty   = {}
    self.removed = {}
    print("[UoW] Changes discarded")
end

-- ทดสอบ
print("=== Unit of Work Pattern ===")
local config9 = MySQLConfig.create({ database = "myapp" })
local db9 = MockMySQL.connect(config9)

local uow = UnitOfWork.new(db9)

-- Register new entities
uow:register_new("users", {
    name = "Hannah", email = "hannah@example.com", age = 29, status = "active"
})
uow:register_new("users", {
    name = "Ivan", email = "ivan@example.com", age = 44, status = "active"
})

-- Register updates
uow:register_dirty("users", { id = 1, name = "Alice Updated", status = "active" })
uow:register_dirty("orders", { id = 1, status = "completed" })

-- Register deletes
uow:register_removed("users", 99)

print("Pending changes:")
print("  New:", #(uow.new["users"] or {}), "users")
print("  Dirty:", (function()
    local c = 0
    for _ in pairs(uow.dirty) do c = c + 1 end
    return c
end)(), "entity types")
print("  Removed:", #(uow.removed["users"] or {}), "users")

-- Commit all at once
uow:commit()

db9:close()
```

---

## ตัวอย่างที่ 11: Generic Repository with Query Builder

```lua
-- Generic Repository with Query Builder
local GenericRepo = {}
GenericRepo.__index = GenericRepo

function GenericRepo.new(db, table_name)
    local self = setmetatable({}, GenericRepo)
    self.db    = db
    self.table = table_name
    return self
end

-- Fluent Query Builder
function GenericRepo:query()
    local q = {
        repo       = self,
        _selects   = {},
        _wheres    = {},
        _orwheres  = {},
        _joins     = {},
        _orders    = {},
        _groups    = {},
        _limit     = nil,
        _offset    = nil,
        _params    = {},
    }
    
    function q:select(...)
        for _, col in ipairs({...}) do
            table.insert(self._selects, col)
        end
        return self
    end
    
    function q:where(col, op, val)
        if val == nil then val = op; op = "=" end
        table.insert(self._params, val)
        table.insert(self._wheres, string.format("`%s` %s ?", col, op))
        return self
    end
    
    function q:or_where(col, op, val)
        if val == nil then val = op; op = "=" end
        table.insert(self._params, val)
        table.insert(self._orwheres, string.format("`%s` %s ?", col, op))
        return self
    end
    
    function q:order_by(col, dir)
        table.insert(self._orders, "`" .. col .. "` " .. (dir or "ASC"))
        return self
    end
    
    function q:limit(n) self._limit = n; return self end
    function q:offset(n) self._offset = n; return self end
    
    function q:get()
        local cols = #self._selects > 0 and
            table.concat(self._selects, ", ") or "*"
        
        local sql = "SELECT " .. cols .. " FROM `" .. self.repo.table .. "`"
        
        local all_wheres = {}
        for _, w in ipairs(self._wheres) do table.insert(all_wheres, w) end
        
        if #all_wheres > 0 then
            local where_str = table.concat(all_wheres, " AND ")
            if #self._orwheres > 0 then
                where_str = "(" .. where_str .. ") OR (" ..
                    table.concat(self._orwheres, " OR ") .. ")"
            end
            sql = sql .. " WHERE " .. where_str
        end
        
        if #self._orders > 0 then
            sql = sql .. " ORDER BY " .. table.concat(self._orders, ", ")
        end
        
        if self._limit  then sql = sql .. " LIMIT "  .. self._limit  end
        if self._offset then sql = sql .. " OFFSET " .. self._offset end
        
        print("[Query] " .. sql)
        local res, err = self.repo.db:query(sql, self._params)
        if err then return nil, err end
        return res.rows
    end
    
    function q:first()
        self._limit = 1
        local rows, err = self:get()
        return rows and rows[1], err
    end
    
    function q:count()
        local orig_selects = self._selects
        self._selects = { "COUNT(*) as cnt" }
        local rows, err = self:get()
        self._selects = orig_selects
        return rows and rows[1] and (rows[1].cnt or 0) or 0, err
    end
    
    return q
end

-- ทดสอบ
print("=== Generic Repository ===")
local config10 = MySQLConfig.create({ database = "myapp" })
local db10 = MockMySQL.connect(config10)

local repo = GenericRepo.new(db10, "users")

-- Fluent queries
print("\nFind active users age > 25:")
local users = repo:query()
    :select("id", "name", "age", "status")
    :where("status", "active")
    :order_by("name", "ASC")
    :limit(5)
    :get()
print("Found:", users and #users or 0)

print("\nFind first admin:")
local first = repo:query()
    :where("status", "active")
    :order_by("id", "ASC")
    :first()
print("First:", first and first.name or "none")

db10:close()
```

---

## ตัวอย่างที่ 12: Database Seeding

```lua
-- Database Seeding for development/testing
local Seeder = {}
Seeder.__index = Seeder

function Seeder.new(db)
    local self = setmetatable({}, Seeder)
    self.db      = db
    self.seeders = {}
    return self
end

-- Register seeder
function Seeder:add(name, seeder_fn)
    table.insert(self.seeders, { name = name, fn = seeder_fn })
    return self
end

-- Run specific seeder
function Seeder:run(seeder_name)
    for _, seeder in ipairs(self.seeders) do
        if seeder.name == seeder_name then
            print(string.format("[Seed] Running: %s", seeder.name))
            local ok, err = pcall(seeder.fn, self.db)
            if ok then
                print(string.format("[Seed] ✓ %s completed", seeder.name))
            else
                print(string.format("[Seed] ✗ %s failed: %s", seeder.name, err))
            end
            return ok
        end
    end
    print("[Seed] Not found: " .. seeder_name)
    return false
end

-- Run all seeders
function Seeder:run_all()
    for _, seeder in ipairs(self.seeders) do
        print(string.format("[Seed] Running: %s", seeder.name))
        local ok, err = pcall(seeder.fn, self.db)
        if ok then
            print(string.format("[Seed] ✓ %s", seeder.name))
        else
            print(string.format("[Seed] ✗ %s: %s", seeder.name, err))
        end
    end
end

-- Faker-like data generator
local Faker = {}

local first_names = { "Alice", "Bob", "Charlie", "Diana", "Eve", "Frank", "Grace", "Harry" }
local last_names  = { "Smith", "Johnson", "Williams", "Brown", "Jones", "Miller", "Davis" }
local domains     = { "example.com", "test.com", "sample.org", "demo.net" }
local statuses    = { "active", "active", "active", "inactive" }  -- 75% active
local roles       = { "user", "user", "user", "editor", "admin" }  -- weighted

function Faker.name()
    local fn = first_names[math.random(#first_names)]
    local ln = last_names[math.random(#last_names)]
    return fn .. " " .. ln
end

function Faker.email(name)
    local clean = name:lower():gsub("%s+", "."):gsub("[^%w.]", "")
    return clean .. "@" .. domains[math.random(#domains)]
end

function Faker.age() return math.random(18, 65) end
function Faker.status() return statuses[math.random(#statuses)] end
function Faker.role()   return roles[math.random(#roles)] end

function Faker.price(min, max)
    return math.floor(math.random(min or 100, max or 10000) / 10) * 10
end

-- ทดสอบ
math.randomseed(os.time())
print("=== Database Seeding ===")

local config11 = MySQLConfig.create({ database = "myapp" })
local db11 = MockMySQL.connect(config11)

local seeder = Seeder.new(db11)

seeder:add("users", function(db)
    local generated = {}
    for i = 1, 5 do
        local name = Faker.name()
        local user = {
            name   = name,
            email  = Faker.email(name),
            age    = Faker.age(),
            status = Faker.status(),
        }
        table.insert(generated, user)
        print(string.format("  Created user: %s (%s)", user.name, user.status))
    end
    return generated
end)

seeder:add("products", function(db)
    local products = {
        { name = "Widget Pro", category = "electronics", price = 2999 },
        { name = "Super Cable", category = "accessories", price = 299 },
        { name = "PowerBank X", category = "electronics", price = 1499 },
    }
    for _, p in ipairs(products) do
        print(string.format("  Created product: %s (%.0f THB)", p.name, p.price))
    end
end)

seeder:add("orders", function(db)
    for i = 1, 3 do
        local order = {
            user_id = math.random(1, 3),
            amount  = Faker.price(500, 5000),
            status  = ({ "pending", "delivered", "cancelled" })[math.random(3)],
        }
        print(string.format("  Created order: user=%d amount=%.0f status=%s",
            order.user_id, order.amount, order.status))
    end
end)

-- Run all seeders
print("\nRunning all seeders:")
seeder:run_all()

db11:close()
```

---

## สรุปบทที่ 59

ในบทนี้เราได้เรียนรู้:

1. **MySQL Connection** - Config, DSN, mock client
2. **CRUD Operations** - INSERT, SELECT, UPDATE, DELETE แบบ abstracted
3. **Transactions** - BEGIN/COMMIT/ROLLBACK, isolation levels
4. **Stored Procedures** - CALL, IN/OUT parameters
5. **Batch Inserts** - Multi-row INSERT, chunked, INSERT IGNORE
6. **Connection Pool** - Acquire/release, min/max connections
7. **Repository Pattern** - BaseRepository, UserRepository
8. **Active Record** - Define, find, save, destroy
9. **Data Mapper** - Domain objects แยกจาก DB logic
10. **Unit of Work** - Batch all changes in one transaction
11. **Generic Repository** - Fluent query builder
12. **Database Seeding** - Faker, seeder registration

> **Note**: ใช้ `lua-resty-mysql` สำหรับ OpenResty หรือ `luamysql`/`LuaSQL` สำหรับ standalone Lua
