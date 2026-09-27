# บทที่ 58: PostgreSQL Advanced

## บทนำ

PostgreSQL เป็น relational database ที่ทรงพลังที่สุดตัวหนึ่ง มีคุณสมบัติขั้นสูงเช่น JSON columns, Array types, Full-text search และ LISTEN/NOTIFY ในบทนี้จะเรียนรู้การใช้งาน PostgreSQL จาก Lua

---

## ตัวอย่างที่ 1: Connection Setup

```lua
-- PostgreSQL connection with lua-resty-postgres / luapgsql

-- Connection configuration
local PgConfig = {}

function PgConfig.parse_dsn(dsn)
    -- รูปแบบ: postgres://user:password@host:port/dbname?sslmode=require
    local user, password, host, port, dbname, params =
        dsn:match("^postgres://([^:]+):([^@]+)@([^:]+):(%d+)/([^?]+)%??(.*)")
    
    if not user then
        -- ลอง format อื่น (without password)
        user, host, port, dbname, params =
            dsn:match("^postgres://([^@]+)@([^:]+):(%d+)/([^?]+)%??(.*)")
    end
    
    local config = {
        user     = user     or "postgres",
        password = password or "",
        host     = host     or "127.0.0.1",
        port     = tonumber(port) or 5432,
        dbname   = dbname   or "mydb",
        options  = {},
    }
    
    -- Parse query parameters
    if params and params ~= "" then
        for k, v in params:gmatch("([^=&]+)=([^&]*)") do
            config.options[k] = v
        end
    end
    
    return config
end

function PgConfig.build_connection_string(config)
    local parts = {}
    if config.host     then table.insert(parts, "host=" .. config.host) end
    if config.port     then table.insert(parts, "port=" .. config.port) end
    if config.dbname   then table.insert(parts, "dbname=" .. config.dbname) end
    if config.user     then table.insert(parts, "user=" .. config.user) end
    if config.password and config.password ~= "" then
        table.insert(parts, "password=" .. config.password)
    end
    if config.options then
        for k, v in pairs(config.options) do
            table.insert(parts, k .. "=" .. v)
        end
    end
    return table.concat(parts, " ")
end

-- ทดสอบ
print("=== PostgreSQL Connection Setup ===")

local dsn = "postgres://alice:secret123@db.example.com:5432/myapp?sslmode=require&connect_timeout=10"
local config = PgConfig.parse_dsn(dsn)
print("Host:", config.host)
print("Port:", config.port)
print("Database:", config.dbname)
print("User:", config.user)
print("SSL Mode:", config.options.sslmode)
print("Timeout:", config.options.connect_timeout)

local conn_str = PgConfig.build_connection_string(config)
print("\nConnection string:", conn_str)
```

---

## ตัวอย่างที่ 2: Mock PostgreSQL Client

```lua
-- Mock PostgreSQL client สำหรับ development/testing
local MockPg = {}
MockPg.__index = MockPg

-- Mock database state
local mock_tables = {
    users = {
        { id = 1, name = "Alice", email = "alice@example.com", age = 30, role = "admin",
          created_at = "2024-01-01", metadata = '{"theme":"dark","lang":"th"}' },
        { id = 2, name = "Bob",   email = "bob@example.com",   age = 25, role = "user",
          created_at = "2024-01-15", metadata = '{"theme":"light","lang":"en"}' },
        { id = 3, name = "Charlie", email = "charlie@example.com", age = 35, role = "user",
          created_at = "2024-02-01", metadata = '{"theme":"dark","lang":"en"}' },
    },
    posts = {
        { id = 1, title = "Lua Tutorial", content = "Learn Lua programming...",
          user_id = 1, tags = "{lua,programming}", views = 1500, published = true },
        { id = 2, title = "Redis Guide", content = "Redis is fast...",
          user_id = 2, tags = "{redis,database}", views = 800, published = true },
        { id = 3, title = "Draft Post", content = "Work in progress...",
          user_id = 1, tags = "{draft}", views = 0, published = false },
    },
}

local auto_increment = { users = 4, posts = 4 }

function MockPg.connect(config)
    local self = setmetatable({}, MockPg)
    self.config = config
    self.in_transaction = false
    self.transaction_log = {}
    self.prepared_stmts = {}
    print(string.format("[PG] Connected to %s@%s/%s",
        config.user or "postgres",
        config.host or "localhost",
        config.dbname or "mydb"))
    return self, nil
end

function MockPg:query(sql, params)
    -- Simple SQL parser for mock execution
    local cmd = sql:match("^%s*(%u+)") or ""
    
    if cmd == "SELECT" then
        return self:_mock_select(sql, params)
    elseif cmd == "INSERT" then
        return self:_mock_insert(sql, params)
    elseif cmd == "UPDATE" then
        return self:_mock_update(sql, params)
    elseif cmd == "DELETE" then
        return self:_mock_delete(sql, params)
    elseif cmd == "BEGIN" then
        self.in_transaction = true
        return { command = "BEGIN" }, nil
    elseif cmd == "COMMIT" then
        self.in_transaction = false
        self.transaction_log = {}
        return { command = "COMMIT" }, nil
    elseif cmd == "ROLLBACK" then
        -- Undo transaction log
        for i = #self.transaction_log, 1, -1 do
            local entry = self.transaction_log[i]
            if entry.type == "insert" then
                -- Remove inserted row
                local tbl = mock_tables[entry.table]
                for j = #tbl, 1, -1 do
                    if tbl[j].id == entry.id then
                        table.remove(tbl, j)
                        break
                    end
                end
            end
        end
        self.in_transaction = false
        self.transaction_log = {}
        return { command = "ROLLBACK" }, nil
    end
    
    return nil, "Unsupported SQL command: " .. cmd
end

function MockPg:_mock_select(sql, params)
    -- Parse table name
    local table_name = sql:match("[Ff][Rr][Oo][Mm]%s+(%w+)") or
                       sql:match("[Ff][Rr][Oo][Mm]%s+\"(%w+)\"")
    local tbl = mock_tables[table_name]
    if not tbl then return { rows = {}, row_count = 0 }, nil end
    
    -- Simple WHERE clause parsing
    local where_id = sql:match("[Ww][Hh][Ee][Rr][Ee]%s+id%s*=%s*%$1")
    local where_user = sql:match("[Ww][Hh][Ee][Rr][Ee]%s+user_id%s*=%s*%$1")
    
    local result_rows = {}
    for _, row in ipairs(tbl) do
        local include = true
        if where_id and params and params[1] then
            include = row.id == tonumber(params[1])
        elseif where_user and params and params[1] then
            include = row.user_id == tonumber(params[1])
        end
        if include then
            table.insert(result_rows, row)
        end
    end
    
    -- LIMIT
    local limit = sql:match("[Ll][Ii][Mm][Ii][Tt]%s+(%d+)")
    if limit then
        local lim = tonumber(limit)
        while #result_rows > lim do
            table.remove(result_rows)
        end
    end
    
    return { rows = result_rows, row_count = #result_rows }, nil
end

function MockPg:_mock_insert(sql, params)
    local table_name = sql:match("[Ii][Nn][Tt][Oo]%s+(%w+)")
    local tbl = mock_tables[table_name]
    if not tbl then return nil, "Table not found: " .. (table_name or "nil") end
    
    -- Create new row with auto-increment ID
    local new_id = auto_increment[table_name] or 100
    auto_increment[table_name] = new_id + 1
    
    local new_row = { id = new_id }
    if params then
        -- Map params to columns (simplified)
        local col_str = sql:match("%((.-)%)%s*[Vv][Aa][Ll][Uu][Ee][Ss]")
        if col_str then
            local cols = {}
            for col in col_str:gmatch("[%w_]+") do
                table.insert(cols, col)
            end
            for i, col in ipairs(cols) do
                new_row[col] = params[i]
            end
        end
    end
    
    table.insert(tbl, new_row)
    
    if self.in_transaction then
        table.insert(self.transaction_log, { type = "insert", table = table_name, id = new_id })
    end
    
    return { rows = { new_row }, row_count = 1, last_insert_id = new_id }, nil
end

function MockPg:_mock_update(sql, params)
    local table_name = sql:match("[Uu][Pp][Dd][Aa][Tt][Ee]%s+(%w+)")
    local tbl = mock_tables[table_name]
    if not tbl then return nil, "Table not found" end
    
    local where_id = params and params[#params]
    local updated = 0
    
    for _, row in ipairs(tbl) do
        if not where_id or row.id == tonumber(where_id) then
            -- Update fields (simplified - update first SET value)
            local col, val_idx = sql:match("[Ss][Ee][Tt]%s+(%w+)%s*=%s*%$(%d+)")
            if col and val_idx and params then
                row[col] = params[tonumber(val_idx)]
            end
            updated = updated + 1
        end
    end
    
    return { row_count = updated }, nil
end

function MockPg:_mock_delete(sql, params)
    local table_name = sql:match("[Ff][Rr][Oo][Mm]%s+(%w+)")
    local tbl = mock_tables[table_name]
    if not tbl then return nil, "Table not found" end
    
    local where_id = params and params[1]
    local deleted = 0
    
    for i = #tbl, 1, -1 do
        if not where_id or tbl[i].id == tonumber(where_id) then
            table.remove(tbl, i)
            deleted = deleted + 1
            if where_id then break end
        end
    end
    
    return { row_count = deleted }, nil
end

function MockPg:prepare(name, sql)
    self.prepared_stmts[name] = { name = name, sql = sql }
    return { name = name }, nil
end

function MockPg:execute(stmt_name, params)
    local stmt = self.prepared_stmts[stmt_name]
    if not stmt then return nil, "Statement not found: " .. stmt_name end
    return self:query(stmt.sql, params)
end

function MockPg:close()
    print("[PG] Connection closed")
end

-- ทดสอบ
print("\n=== Mock PostgreSQL Client ===")

local db, err = MockPg.connect({
    host   = "localhost",
    port   = 5432,
    dbname = "myapp",
    user   = "alice",
})

-- SELECT
local result, qerr = db:query("SELECT * FROM users LIMIT 2")
if result then
    print(string.format("Found %d users:", result.row_count))
    for _, row in ipairs(result.rows) do
        print(string.format("  id=%d name=%s email=%s", row.id, row.name, row.email))
    end
end

-- SELECT with WHERE
local result2, _ = db:query("SELECT * FROM users WHERE id = $1", { "1" })
print("\nUser by ID:")
for _, row in ipairs(result2.rows) do
    print(string.format("  %s (%s)", row.name, row.role))
end

db:close()
```

---

## ตัวอย่างที่ 3: Prepared Statements

```lua
-- Prepared Statements ป้องกัน SQL Injection
local PreparedStmt = {}
PreparedStmt.__index = PreparedStmt

function PreparedStmt.new(db)
    local self = setmetatable({}, PreparedStmt)
    self.db    = db
    self.stmts = {}
    return self
end

-- Prepare statement
function PreparedStmt:prepare(name, sql)
    self.stmts[name] = {
        sql     = sql,
        created = os.time(),
        uses    = 0,
    }
    print(string.format("[PG] PREPARE %s AS %s...", name, sql:sub(1, 50)))
    return self
end

-- Execute prepared statement
function PreparedStmt:execute(name, params)
    local stmt = self.stmts[name]
    if not stmt then
        return nil, "Statement '" .. name .. "' not prepared"
    end
    
    stmt.uses = stmt.uses + 1
    
    -- Substitute parameters safely (no SQL injection)
    local sql_with_params = stmt.sql
    for i, param in ipairs(params or {}) do
        local safe_param
        if type(param) == "number" then
            safe_param = tostring(param)
        elseif type(param) == "string" then
            -- Escape single quotes
            safe_param = "'" .. param:gsub("'", "''") .. "'"
        elseif type(param) == "boolean" then
            safe_param = param and "TRUE" or "FALSE"
        elseif param == nil then
            safe_param = "NULL"
        else
            safe_param = tostring(param)
        end
        sql_with_params = sql_with_params:gsub("%$" .. i, safe_param, 1)
    end
    
    print(string.format("[PG] EXECUTE %s -> %s", name, sql_with_params:sub(1, 80)))
    
    -- Return mock result
    return { sql = sql_with_params, params = params, rows = {}, row_count = 0 }, nil
end

-- Deallocate
function PreparedStmt:deallocate(name)
    if self.stmts[name] then
        print(string.format("[PG] DEALLOCATE %s (used %d times)", name, self.stmts[name].uses))
        self.stmts[name] = nil
        return true
    end
    return false
end

-- ทดสอบ
print("=== Prepared Statements ===")
local mock_db = { query = function() return {}, nil end }

local ps = PreparedStmt.new(mock_db)

-- Define statements
ps:prepare("get_user", "SELECT * FROM users WHERE id = $1")
ps:prepare("insert_user",
    "INSERT INTO users (name, email, age) VALUES ($1, $2, $3) RETURNING *")
ps:prepare("update_role",
    "UPDATE users SET role = $1 WHERE id = $2")
ps:prepare("search_users",
    "SELECT * FROM users WHERE name ILIKE $1 OR email ILIKE $1 LIMIT $2")

-- Execute
ps:execute("get_user", { 42 })
ps:execute("insert_user", { "Dave", "dave@example.com", 28 })
ps:execute("update_role", { "moderator", 3 })
ps:execute("search_users", { "%alice%", 10 })

-- SQL Injection attempt (should be safe)
print("\nSQL Injection test:")
ps:execute("search_users", { "'; DROP TABLE users; --", 10 })

ps:deallocate("get_user")
ps:deallocate("insert_user")
```

---

## ตัวอย่างที่ 4: Transactions

```lua
-- PostgreSQL Transactions
local Transaction = {}
Transaction.__index = Transaction

function Transaction.new(db)
    local self = setmetatable({}, Transaction)
    self.db  = db
    self.log = {}
    return self
end

function Transaction:begin()
    print("[PG] BEGIN")
    self.log = {}
    self.started = true
    return self
end

function Transaction:exec(sql, params)
    if not self.started then
        return nil, "Transaction not started"
    end
    table.insert(self.log, { sql = sql, params = params })
    print(string.format("[PG] EXEC: %s", sql:sub(1, 60)))
    return { row_count = 1 }, nil
end

function Transaction:commit()
    if not self.started then return nil, "No active transaction" end
    print(string.format("[PG] COMMIT (%d operations)", #self.log))
    self.started = false
    self.log = {}
    return true, nil
end

function Transaction:rollback()
    if not self.started then return nil, "No active transaction" end
    print(string.format("[PG] ROLLBACK (%d operations undone)", #self.log))
    self.started = false
    self.log = {}
    return true, nil
end

-- Higher-level transaction wrapper
function Transaction.run(db, fn)
    local tx = Transaction.new(db)
    tx:begin()
    
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
print("=== PostgreSQL Transactions ===")

-- Mock db
local mock_db2 = {}

-- Successful transaction: transfer money
print("\nTransaction 1: Transfer money")
Transaction.run(mock_db2, function(tx)
    tx:exec("UPDATE accounts SET balance = balance - $1 WHERE id = $2", { 500, 1 })
    tx:exec("UPDATE accounts SET balance = balance + $1 WHERE id = $2", { 500, 2 })
    tx:exec("INSERT INTO transfers (from_id, to_id, amount) VALUES ($1, $2, $3)",
        { 1, 2, 500 })
    print("Transfer completed!")
end)

-- Failed transaction: validation error
print("\nTransaction 2: Failed transfer")
local result, err = Transaction.run(mock_db2, function(tx)
    tx:exec("UPDATE accounts SET balance = balance - $1 WHERE id = $2", { 10000, 1 })
    -- Simulate validation error
    error("Insufficient funds: balance would be negative")
end)
print("Error:", err)
```

---

## ตัวอย่างที่ 5: Savepoints

```lua
-- Savepoints ใน PostgreSQL transactions
local SavepointTx = {}
SavepointTx.__index = SavepointTx

function SavepointTx.new()
    local self = setmetatable({}, SavepointTx)
    self.operations = {}
    self.savepoints = {}
    self.active = false
    return self
end

function SavepointTx:begin()
    self.operations = {}
    self.savepoints = {}
    self.active = true
    print("[PG] BEGIN")
    return self
end

function SavepointTx:exec(sql, params)
    if not self.active then error("No active transaction") end
    local op = { sql = sql:sub(1, 50), params = params, time = os.time() }
    table.insert(self.operations, op)
    print("[PG] EXEC:", sql:sub(1, 50))
    return self
end

function SavepointTx:savepoint(name)
    self.savepoints[name] = #self.operations
    print("[PG] SAVEPOINT " .. name)
    return self
end

function SavepointTx:rollback_to(name)
    local sp_pos = self.savepoints[name]
    if not sp_pos then
        error("Savepoint '" .. name .. "' not found")
    end
    
    -- Remove operations after savepoint
    local removed = 0
    while #self.operations > sp_pos do
        table.remove(self.operations)
        removed = removed + 1
    end
    
    print(string.format("[PG] ROLLBACK TO %s (undid %d operations)", name, removed))
    return self
end

function SavepointTx:release(name)
    if self.savepoints[name] then
        self.savepoints[name] = nil
        print("[PG] RELEASE SAVEPOINT " .. name)
    end
    return self
end

function SavepointTx:commit()
    print(string.format("[PG] COMMIT (%d total operations)", #self.operations))
    for i, op in ipairs(self.operations) do
        print(string.format("  %d: %s", i, op.sql))
    end
    self.active = false
end

function SavepointTx:rollback()
    print("[PG] ROLLBACK ALL")
    self.active = false
end

-- ทดสอบ
print("=== Savepoints ===")

local tx = SavepointTx.new()
tx:begin()
tx:exec("INSERT INTO orders (user_id, total) VALUES ($1, $2)", { 1, 1500 })
tx:exec("UPDATE inventory SET stock = stock - $1 WHERE product_id = $2", { 2, 101 })

-- Savepoint ก่อน payment processing
tx:savepoint("before_payment")
tx:exec("INSERT INTO payments (order_id, amount, method) VALUES ($1, $2, $3)",
    { 1, 1500, "credit_card" })
tx:exec("UPDATE accounts SET balance = balance - $1 WHERE user_id = $2", { 1500, 1 })

-- สมมุติว่า payment ล้มเหลว
print("\nPayment gateway timeout - rolling back to savepoint...")
tx:rollback_to("before_payment")

-- ลอง payment method อื่น
tx:exec("INSERT INTO payments (order_id, amount, method) VALUES ($1, $2, $3)",
    { 1, 1500, "bank_transfer" })
tx:release("before_payment")

tx:commit()
```

---

## ตัวอย่างที่ 6: Batch Operations

```lua
-- Batch insert/update operations
local BatchOps = {}

-- Batch insert
function BatchOps.batch_insert(table_name, columns, rows)
    if #rows == 0 then return "No rows to insert" end
    
    local col_str = table.concat(columns, ", ")
    local value_placeholders = {}
    local all_params = {}
    local param_idx = 1
    
    for _, row in ipairs(rows) do
        local row_placeholders = {}
        for _, col in ipairs(columns) do
            table.insert(row_placeholders, "$" .. param_idx)
            table.insert(all_params, row[col])
            param_idx = param_idx + 1
        end
        table.insert(value_placeholders, "(" .. table.concat(row_placeholders, ", ") .. ")")
    end
    
    local sql = string.format("INSERT INTO %s (%s) VALUES %s",
        table_name,
        col_str,
        table.concat(value_placeholders, ", \n  ")
    )
    
    return sql, all_params
end

-- Batch update with CASE
function BatchOps.batch_update(table_name, id_col, update_col, updates)
    -- Build: UPDATE table SET col = CASE id WHEN 1 THEN v1 WHEN 2 THEN v2 END WHERE id IN (...)
    local when_clauses = {}
    local ids = {}
    local params = {}
    local param_idx = 1
    
    for _, update in ipairs(updates) do
        table.insert(when_clauses, string.format("WHEN $%d THEN $%d",
            param_idx, param_idx + 1))
        table.insert(params, update.id)
        table.insert(params, update.value)
        table.insert(ids, "$" .. param_idx)
        param_idx = param_idx + 2
    end
    
    local sql = string.format(
        "UPDATE %s SET %s = CASE %s\n  %s\n  END\nWHERE %s IN (%s)",
        table_name, update_col, id_col,
        table.concat(when_clauses, "\n  "),
        id_col,
        table.concat(ids, ", ")
    )
    
    return sql, params
end

-- COPY command สำหรับ bulk insert (fastest)
function BatchOps.generate_copy(table_name, columns, rows)
    local lines = {}
    for _, row in ipairs(rows) do
        local values = {}
        for _, col in ipairs(columns) do
            local val = row[col]
            if val == nil then
                table.insert(values, "\\N")  -- NULL
            elseif type(val) == "string" then
                table.insert(values, val:gsub("\t", "\\t"):gsub("\n", "\\n"))
            else
                table.insert(values, tostring(val))
            end
        end
        table.insert(lines, table.concat(values, "\t"))
    end
    
    local copy_sql = string.format("COPY %s (%s) FROM STDIN",
        table_name, table.concat(columns, ", "))
    
    return copy_sql, table.concat(lines, "\n")
end

-- ทดสอบ
print("=== Batch Operations ===")

-- Batch insert
local columns = { "name", "email", "age", "role" }
local rows = {
    { name = "Dave",  email = "dave@example.com",  age = 28, role = "user" },
    { name = "Eve",   email = "eve@example.com",   age = 32, role = "user" },
    { name = "Frank", email = "frank@example.com", age = 27, role = "editor" },
    { name = "Grace", email = "grace@example.com", age = 45, role = "admin" },
}

local insert_sql, params = BatchOps.batch_insert("users", columns, rows)
print("Batch INSERT SQL:")
print(insert_sql:sub(1, 200))
print(string.format("Parameters: %d values", #params))

-- Batch update
print("\nBatch UPDATE SQL:")
local updates = {
    { id = 1, value = 1600 },
    { id = 3, value = 950 },
    { id = 5, value = 2200 },
}
local update_sql, update_params = BatchOps.batch_update("orders", "id", "total", updates)
print(update_sql)

-- COPY command
print("\nCOPY SQL:")
local copy_cmd, copy_data = BatchOps.generate_copy("users", {"name","email","age"}, {
    { name = "Harry", email = "harry@example.com", age = 29 },
    { name = "Iris",  email = "iris@example.com",  age = 34 },
})
print(copy_cmd)
print("Data:")
print(copy_data)
```

---

## ตัวอย่างที่ 7: JSON Column Operations

```lua
-- PostgreSQL JSON/JSONB column operations
local PGJSON = {}

-- Build JSON queries
function PGJSON.select_json_field(table_name, json_col, json_path)
    -- JSONB operator: ->> for text, -> for json
    return string.format("SELECT %s->>'%s' FROM %s",
        json_col, json_path, table_name)
end

function PGJSON.select_nested(table_name, json_col, path_parts)
    local path = table.concat(path_parts, "'->'")
    return string.format("SELECT %s->'%s' FROM %s",
        json_col, path, table_name)
end

function PGJSON.where_json_field(json_col, field, value)
    if type(value) == "string" then
        return string.format("%s->>'%s' = '%s'", json_col, field, value)
    elseif type(value) == "number" then
        return string.format("(%s->>'%s')::numeric = %s", json_col, field, value)
    elseif type(value) == "boolean" then
        return string.format("(%s->>'%s')::boolean = %s", json_col,
            field, value and "true" or "false")
    end
end

function PGJSON.update_json_field(table_name, id, json_col, field, value)
    local value_str
    if type(value) == "string" then
        value_str = '"' .. value .. '"'
    elseif type(value) == "number" or type(value) == "boolean" then
        value_str = tostring(value)
    end
    
    return string.format(
        "UPDATE %s SET %s = jsonb_set(%s, '{%s}', '%s') WHERE id = %d",
        table_name, json_col, json_col, field, value_str, id
    )
end

-- JSON aggregation queries
function PGJSON.json_agg(table_name, columns, group_by)
    local col_list = table.concat(columns, ", ")
    if group_by then
        return string.format(
            "SELECT %s, json_agg(json_build_object(%s)) as items FROM %s GROUP BY %s",
            group_by, 
            (function()
                local pairs_list = {}
                for _, c in ipairs(columns) do
                    table.insert(pairs_list, string.format("'%s', %s", c, c))
                end
                return table.concat(pairs_list, ", ")
            end)(),
            table_name, group_by
        )
    end
    return string.format("SELECT json_agg(%s) FROM %s", col_list, table_name)
end

-- Parse JSON-like data (mock)
local function parse_json_mock(json_str)
    -- สำหรับตัวอย่าง: parse simple key-value JSON
    local obj = {}
    for k, v in json_str:gmatch('"(%w+)"%s*:%s*"([^"]*)"') do
        obj[k] = v
    end
    for k, v in json_str:gmatch('"(%w+)"%s*:%s*(%d+)') do
        obj[k] = tonumber(v)
    end
    return obj
end

-- ทดสอบ
print("=== PostgreSQL JSON Operations ===")

-- JSON field selection
print("\nJSON Queries:")
print(PGJSON.select_json_field("users", "metadata", "theme"))
print(PGJSON.where_json_field("metadata", "lang", "th"))
print(PGJSON.where_json_field("metadata", "age", 30))

-- Update JSON field
print("\nJSON Update:")
print(PGJSON.update_json_field("users", 1, "metadata", "theme", "system"))

-- JSON aggregation
print("\nJSON Aggregation:")
print(PGJSON.json_agg("orders", {"id", "total", "status"}, "user_id"))

-- Working with JSON data
local json_data = '{"theme":"dark","lang":"th","notifications":true}'
local parsed = parse_json_mock(json_data)
print("\nParsed JSON:")
for k, v in pairs(parsed) do
    print(string.format("  %s = %s (%s)", k, tostring(v), type(v)))
end

-- Build jsonb queries
print("\nAdvanced JSONB:")
local queries = {
    "SELECT * FROM users WHERE metadata @> '{\"role\":\"admin\"}'::jsonb",
    "SELECT * FROM users WHERE metadata ? 'phone'",  -- has key
    "SELECT metadata - 'password' FROM users",       -- remove key
    "SELECT * FROM events WHERE data->>'type' = 'purchase'",
    "SELECT jsonb_pretty(metadata) FROM users WHERE id = 1",
}
for _, q in ipairs(queries) do
    print("  " .. q)
end
```

---

## ตัวอย่างที่ 8: Array Columns

```lua
-- PostgreSQL Array column operations
local PGArray = {}

-- Array operators
function PGArray.contains(col, value)
    if type(value) == "string" then
        return string.format("%s @> ARRAY['%s']", col, value)
    end
    return string.format("%s @> ARRAY[%s]", col, value)
end

function PGArray.overlap(col, values)
    local val_strs = {}
    for _, v in ipairs(values) do
        table.insert(val_strs, type(v) == "string" and ("'" .. v .. "'") or tostring(v))
    end
    return string.format("%s && ARRAY[%s]", col, table.concat(val_strs, ", "))
end

function PGArray.any(col, value)
    if type(value) == "string" then
        return string.format("'%s' = ANY(%s)", value, col)
    end
    return string.format("%s = ANY(%s)", value, col)
end

function PGArray.append(table_name, id, col, value)
    if type(value) == "string" then
        return string.format(
            "UPDATE %s SET %s = array_append(%s, '%s') WHERE id = %d",
            table_name, col, col, value, id)
    end
    return string.format(
        "UPDATE %s SET %s = array_append(%s, %s) WHERE id = %d",
        table_name, col, col, value, id)
end

function PGArray.remove_elem(table_name, id, col, value)
    if type(value) == "string" then
        return string.format(
            "UPDATE %s SET %s = array_remove(%s, '%s') WHERE id = %d",
            table_name, col, col, value, id)
    end
    return string.format(
        "UPDATE %s SET %s = array_remove(%s, %s) WHERE id = %d",
        table_name, col, col, value, id)
end

-- Array aggregation
function PGArray.aggregate_queries(table_name)
    return {
        unnest    = string.format("SELECT unnest(tags) as tag, count(*) FROM %s GROUP BY tag ORDER BY count DESC", table_name),
        all_tags  = string.format("SELECT array_agg(DISTINCT unnest) FROM (SELECT unnest(tags) FROM %s) t", table_name),
        array_len = string.format("SELECT id, array_length(tags, 1) as tag_count FROM %s", table_name),
    }
end

-- Mock array data operations
local posts_with_tags = {
    { id = 1, title = "Lua Guide",   tags = { "lua", "programming", "tutorial" } },
    { id = 2, title = "Redis Tips",  tags = { "redis", "database", "performance" } },
    { id = 3, title = "PG Advanced", tags = { "postgresql", "database", "lua" } },
}

-- ค้นหา posts ที่มี tag ที่ต้องการ
local function find_by_tag(tag)
    local result = {}
    for _, post in ipairs(posts_with_tags) do
        for _, t in ipairs(post.tags) do
            if t == tag then
                table.insert(result, post)
                break
            end
        end
    end
    return result
end

-- ทดสอบ
print("=== PostgreSQL Array Operations ===")

print("\nArray SQL Queries:")
print("Contains:", PGArray.contains("tags", "lua"))
print("Overlap:", PGArray.overlap("tags", {"lua", "redis"}))
print("Any:", PGArray.any("tags", "database"))
print("Append:", PGArray.append("posts", 1, "tags", "beginner"))
print("Remove:", PGArray.remove_elem("posts", 1, "tags", "tutorial"))

print("\nArray Aggregation:")
local agg = PGArray.aggregate_queries("posts")
for name, sql in pairs(agg) do
    print(string.format("  [%s]: %s", name, sql:sub(1, 70)))
end

print("\nMock: Posts with tag 'database':")
local db_posts = find_by_tag("database")
for _, p in ipairs(db_posts) do
    print(string.format("  [%d] %s - tags: %s", p.id, p.title, table.concat(p.tags, ", ")))
end

print("\nMock: Posts with tag 'lua':")
local lua_posts = find_by_tag("lua")
for _, p in ipairs(lua_posts) do
    print(string.format("  [%d] %s", p.id, p.title))
end
```

---

## ตัวอย่างที่ 9: Full-Text Search

```lua
-- PostgreSQL Full-Text Search
local FTS = {}

-- สร้าง tsvector query
function FTS.build_search(query, language)
    language = language or "english"
    -- ล้าง special characters
    local clean = query:gsub("[^%w%s]", " "):gsub("%s+", " "):match("^%s*(.-)%s*$")
    
    -- Split words
    local words = {}
    for word in clean:gmatch("%S+") do
        if #word > 2 then  -- skip short words
            table.insert(words, word)
        end
    end
    
    return {
        tsquery  = "plainto_tsquery('" .. language .. "', $1)",
        tsvector = "to_tsvector('" .. language .. "', $1)",
        words    = words,
        original = query,
    }
end

-- Full-text search query builder
function FTS.search_query(table_name, columns, search_term, options)
    options = options or {}
    local lang = options.language or "english"
    local limit = options.limit or 20
    local offset = options.offset or 0
    
    -- Build tsvector from multiple columns
    local vector_parts = {}
    for i, col in ipairs(columns) do
        local weight = ({ "A", "B", "C", "D" })[i] or "D"
        table.insert(vector_parts, string.format(
            "setweight(to_tsvector('%s', coalesce(%s, '')), '%s')",
            lang, col, weight))
    end
    local tsvector = table.concat(vector_parts, " || ")
    
    local sql = string.format([[
SELECT 
    *,
    ts_rank(%s, plainto_tsquery('%s', $1)) as rank,
    ts_headline('%s', %s, plainto_tsquery('%s', $1)) as headline
FROM %s
WHERE %s @@ plainto_tsquery('%s', $1)
ORDER BY rank DESC
LIMIT %d OFFSET %d
    ]],
        tsvector, lang,
        lang, columns[1], lang,
        table_name,
        tsvector, lang,
        limit, offset
    )
    
    return sql
end

-- GIN index creation
function FTS.create_index(table_name, columns, language)
    language = language or "english"
    local vector_expr = {}
    for _, col in ipairs(columns) do
        table.insert(vector_expr, string.format(
            "to_tsvector('%s', coalesce(%s, ''))", language, col))
    end
    
    return string.format(
        "CREATE INDEX idx_%s_fts ON %s USING gin((%s))",
        table_name,
        table_name,
        table.concat(vector_expr, " || ")
    )
end

-- Simulated full-text search
local documents = {
    { id = 1, title = "Lua Programming Language",
      content = "Lua is a lightweight scripting language designed for embedded use." },
    { id = 2, title = "Redis Data Structures",
      content = "Redis provides data structures like strings, hashes, lists, sets." },
    { id = 3, title = "PostgreSQL Advanced Features",
      content = "PostgreSQL supports JSON, full-text search, and complex queries." },
    { id = 4, title = "Lua and Redis Integration",
      content = "Using Lua scripts in Redis for atomic operations and performance." },
    { id = 5, title = "Database Design Patterns",
      content = "Learn about normalization, indexing, and query optimization." },
}

local function simple_fts(query, docs)
    local terms = {}
    for word in query:lower():gmatch("%S+") do
        table.insert(terms, word)
    end
    
    local results = {}
    for _, doc in ipairs(docs) do
        local score = 0
        local full_text = (doc.title .. " " .. doc.content):lower()
        for _, term in ipairs(terms) do
            local _, count = full_text:gsub(term, "")
            if count > 0 then
                score = score + count * (full_text:sub(1, #doc.title):find(term) and 2 or 1)
            end
        end
        if score > 0 then
            table.insert(results, { doc = doc, score = score })
        end
    end
    
    table.sort(results, function(a, b) return a.score > b.score end)
    return results
end

-- ทดสอบ
print("=== Full-Text Search ===")

-- SQL generation
local search_sql = FTS.search_query("documents",
    { "title", "content", "tags" }, "lua redis", { limit = 10 })
print("FTS Query:")
print(search_sql:sub(1, 300) .. "...")

local index_sql = FTS.create_index("documents", { "title", "content" })
print("\nGIN Index:")
print(index_sql)

-- Simulated search
print("\nSimulated search results:")
local queries = { "lua programming", "redis", "database" }
for _, query in ipairs(queries) do
    local results = simple_fts(query, documents)
    print(string.format("\nSearch: '%s' -> %d results", query, #results))
    for i, r in ipairs(results) do
        if i <= 3 then
            print(string.format("  [%.0f] %s", r.score, r.doc.title))
        end
    end
end
```

---

## ตัวอย่างที่ 10: Stored Procedures Call

```lua
-- Calling PostgreSQL Stored Procedures from Lua
local StoredProcs = {}

-- Define stored procedure calls
function StoredProcs.call(db, proc_name, params)
    local placeholders = {}
    for i = 1, #(params or {}) do
        table.insert(placeholders, "$" .. i)
    end
    
    local sql = string.format("SELECT * FROM %s(%s)",
        proc_name,
        table.concat(placeholders, ", ")
    )
    
    print(string.format("[PG] CALL %s(%s)",
        proc_name,
        table.concat(params or {}, ", ")))
    
    return sql, params
end

-- Common stored procedures
local procedures = {
    -- Get user with posts count
    get_user_stats = [[
CREATE OR REPLACE FUNCTION get_user_stats(p_user_id INTEGER)
RETURNS TABLE(
    user_id INTEGER,
    username TEXT,
    post_count BIGINT,
    total_views BIGINT,
    avg_views NUMERIC
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        u.id,
        u.name,
        COUNT(p.id),
        COALESCE(SUM(p.views), 0),
        COALESCE(AVG(p.views), 0)
    FROM users u
    LEFT JOIN posts p ON p.user_id = u.id
    WHERE u.id = p_user_id
    GROUP BY u.id, u.name;
END;
$$ LANGUAGE plpgsql;
]],
    
    -- Transfer credits
    transfer_credits = [[
CREATE OR REPLACE FUNCTION transfer_credits(
    p_from_user INTEGER,
    p_to_user   INTEGER,
    p_amount    NUMERIC
) RETURNS BOOLEAN AS $$
DECLARE
    v_balance NUMERIC;
BEGIN
    -- Lock rows (FOR UPDATE)
    SELECT balance INTO v_balance FROM accounts WHERE user_id = p_from_user FOR UPDATE;
    
    IF v_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient balance: % < %', v_balance, p_amount;
    END IF;
    
    UPDATE accounts SET balance = balance - p_amount WHERE user_id = p_from_user;
    UPDATE accounts SET balance = balance + p_amount WHERE user_id = p_to_user;
    
    INSERT INTO transfer_log (from_user, to_user, amount, created_at)
    VALUES (p_from_user, p_to_user, p_amount, NOW());
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
]],
    
    -- Pagination helper
    paginate = [[
CREATE OR REPLACE FUNCTION paginate_posts(
    p_page     INTEGER DEFAULT 1,
    p_per_page INTEGER DEFAULT 10,
    p_user_id  INTEGER DEFAULT NULL
) RETURNS TABLE(
    id        INTEGER,
    title     TEXT,
    views     INTEGER,
    created_at TIMESTAMP,
    total_count BIGINT
) AS $$
DECLARE
    v_offset INTEGER := (p_page - 1) * p_per_page;
BEGIN
    RETURN QUERY
    SELECT 
        p.id, p.title, p.views, p.created_at,
        COUNT(*) OVER() as total_count
    FROM posts p
    WHERE (p_user_id IS NULL OR p.user_id = p_user_id)
      AND p.published = true
    ORDER BY p.created_at DESC
    LIMIT p_per_page OFFSET v_offset;
END;
$$ LANGUAGE plpgsql;
]],
}

-- ทดสอบ
print("=== Stored Procedures ===")

print("Stored Procedure Definitions:")
for name in pairs(procedures) do
    print("  - " .. name)
end

print("\nCalling stored procedures:")
local mock_db = {}

-- Call get_user_stats
local sql1, params1 = StoredProcs.call(mock_db, "get_user_stats", { 1 })
print("SQL:", sql1)

-- Call transfer_credits
local sql2, params2 = StoredProcs.call(mock_db, "transfer_credits", { 1, 2, 500.00 })
print("SQL:", sql2)

-- Call paginate_posts
local sql3, params3 = StoredProcs.call(mock_db, "paginate_posts", { 1, 10 })
print("SQL:", sql3)

print("\nProcedure: get_user_stats")
print(procedures.get_user_stats:sub(1, 200) .. "...")

print("\nProcedure: transfer_credits")
print(procedures.transfer_credits:sub(1, 200) .. "...")
```

---

## ตัวอย่างที่ 11: LISTEN/NOTIFY

```lua
-- PostgreSQL LISTEN/NOTIFY for real-time events
local PgNotify = {}

-- Notification handlers
local listeners = {}
local notification_queue = {}

-- LISTEN to channel
function PgNotify.listen(channel, handler)
    if not listeners[channel] then
        listeners[channel] = {}
    end
    table.insert(listeners[channel], handler)
    print("[PG] LISTEN " .. channel)
    return true
end

-- UNLISTEN from channel
function PgNotify.unlisten(channel)
    listeners[channel] = nil
    print("[PG] UNLISTEN " .. channel)
end

-- Simulate NOTIFY (from another connection/trigger)
function PgNotify.notify(channel, payload)
    table.insert(notification_queue, {
        channel = channel,
        payload = payload,
        pid     = math.random(10000, 99999),  -- sender PID
        time    = os.time(),
    })
    print(string.format("[PG] NOTIFY %s '%s' (queued)", channel, payload))
end

-- Process notifications (would be async in real system)
function PgNotify.process_notifications()
    local processed = 0
    while #notification_queue > 0 do
        local notif = table.remove(notification_queue, 1)
        local handlers = listeners[notif.channel] or {}
        for _, handler in ipairs(handlers) do
            handler(notif.channel, notif.payload, notif.pid)
        end
        processed = processed + 1
    end
    return processed
end

-- Common patterns using LISTEN/NOTIFY
local notify_patterns = {
    -- Trigger that NOTIFYs on INSERT
    order_trigger = [[
CREATE OR REPLACE FUNCTION notify_new_order()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify(
        'new_order',
        json_build_object(
            'id',       NEW.id,
            'user_id',  NEW.user_id,
            'total',    NEW.total,
            'status',   NEW.status
        )::text
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER after_insert_order
AFTER INSERT ON orders
FOR EACH ROW EXECUTE FUNCTION notify_new_order();
]],
    
    -- Cache invalidation via NOTIFY
    cache_invalidation = [[
CREATE OR REPLACE FUNCTION notify_cache_invalidate()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify(
        'cache_invalidate',
        json_build_object(
            'table',    TG_TABLE_NAME,
            'id',       COALESCE(NEW.id, OLD.id),
            'operation', TG_OP
        )::text
    );
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;
]],
}

-- ทดสอบ
math.randomseed(os.time())
print("=== LISTEN/NOTIFY ===")

-- Set up listeners
PgNotify.listen("new_order", function(channel, payload, pid)
    print(string.format("  [%s from PID %d] New order: %s", channel, pid, payload))
end)

PgNotify.listen("cache_invalidate", function(channel, payload, pid)
    print(string.format("  [Cache] Invalidate: %s", payload))
end)

PgNotify.listen("user_activity", function(channel, payload, pid)
    print(string.format("  [Activity] %s", payload))
end)

-- Simulate notifications from DB triggers
PgNotify.notify("new_order", '{"id":1001,"user_id":5,"total":2500}')
PgNotify.notify("new_order", '{"id":1002,"user_id":3,"total":890}')
PgNotify.notify("cache_invalidate", '{"table":"users","id":5,"operation":"UPDATE"}')
PgNotify.notify("user_activity", '{"user_id":5,"action":"login","ip":"192.168.1.1"}')

print("\nProcessing notifications:")
local count = PgNotify.process_notifications()
print(string.format("Processed %d notifications", count))
```

---

## ตัวอย่างที่ 12: Connection Pooling

```lua
-- PostgreSQL Connection Pool
local PgPool = {}
PgPool.__index = PgPool

function PgPool.new(config)
    local self = setmetatable({}, PgPool)
    self.config    = config
    self.min_size  = config.min_size or 2
    self.max_size  = config.max_size or 10
    self.idle_timeout = config.idle_timeout or 300
    self.available = {}  -- idle connections
    self.in_use    = {}  -- active connections
    self.waiting   = {}  -- waiting requests
    self.stats     = { acquired = 0, released = 0, created = 0, destroyed = 0 }
    
    -- Pre-create minimum connections
    for i = 1, self.min_size do
        self:_create_connection()
    end
    
    return self
end

local function create_mock_conn(id)
    return {
        id          = id,
        created_at  = os.time(),
        last_used   = os.time(),
        query_count = 0,
        in_use      = false,
        query = function(self, sql, params)
            self.query_count = self.query_count + 1
            self.last_used = os.time()
            return { rows = {}, row_count = 0 }, nil
        end,
        close = function(self)
            print("[Pool] Closing connection #" .. self.id)
        end,
    }
end

local conn_id_counter = 0

function PgPool:_create_connection()
    conn_id_counter = conn_id_counter + 1
    local conn = create_mock_conn(conn_id_counter)
    self.stats.created = self.stats.created + 1
    table.insert(self.available, conn)
    print(string.format("[Pool] Created connection #%d (pool: %d/%d)",
        conn.id, #self.available + #self.in_use, self.max_size))
    return conn
end

function PgPool:acquire()
    -- Check available connections
    if #self.available > 0 then
        local conn = table.remove(self.available)
        conn.in_use = true
        conn.last_used = os.time()
        self.in_use[conn.id] = conn
        self.stats.acquired = self.stats.acquired + 1
        return conn, nil
    end
    
    -- Create new connection if under max
    local total = #self.available + (function()
        local c = 0
        for _ in pairs(self.in_use) do c = c + 1 end
        return c
    end)()
    
    if total < self.max_size then
        local conn = self:_create_connection()
        table.remove(self.available)  -- remove from available
        conn.in_use = true
        self.in_use[conn.id] = conn
        self.stats.acquired = self.stats.acquired + 1
        return conn, nil
    end
    
    return nil, "Connection pool exhausted"
end

function PgPool:release(conn)
    conn.in_use = false
    self.in_use[conn.id] = nil
    
    -- Check if connection is still healthy
    if os.time() - conn.created_at < self.idle_timeout then
        table.insert(self.available, conn)
    else
        conn:close()
        self.stats.destroyed = self.stats.destroyed + 1
        -- Create replacement if below minimum
        if #self.available < self.min_size then
            self:_create_connection()
        end
    end
    
    self.stats.released = self.stats.released + 1
end

function PgPool:with_connection(fn)
    local conn, err = self:acquire()
    if not conn then
        return nil, err
    end
    
    local ok, result = pcall(fn, conn)
    self:release(conn)
    
    if not ok then
        return nil, tostring(result)
    end
    
    return result, nil
end

function PgPool:status()
    local in_use_count = 0
    for _ in pairs(self.in_use) do in_use_count = in_use_count + 1 end
    return {
        available = #self.available,
        in_use    = in_use_count,
        total     = #self.available + in_use_count,
        max       = self.max_size,
        stats     = self.stats,
    }
end

-- ทดสอบ
print("=== Connection Pool ===")
local pool = PgPool.new({
    host     = "localhost",
    dbname   = "myapp",
    min_size = 2,
    max_size = 5,
})

print("\nPool status:", pool:status().available, "available,",
      pool:status().in_use, "in use")

-- Simulate concurrent requests
print("\nSimulating 4 concurrent requests:")
local conns = {}
for i = 1, 4 do
    local conn, err = pool:acquire()
    if conn then
        table.insert(conns, conn)
        print(string.format("  Request %d: got connection #%d", i, conn.id))
    else
        print(string.format("  Request %d: %s", i, err))
    end
end

print("\nPool status:", pool:status().available, "available,",
      pool:status().in_use, "in use")

-- Release connections
for _, conn in ipairs(conns) do
    pool:release(conn)
end

print("After release:", pool:status().available, "available")

-- with_connection pattern
pool:with_connection(function(conn)
    print("\nExecuting query in pool connection #" .. conn.id)
    conn:query("SELECT * FROM users WHERE id = $1", {1})
end)

local final_stats = pool:status()
print("\nPool Stats: acquired=" .. final_stats.stats.acquired ..
      ", created=" .. final_stats.stats.created)
```

---

## ตัวอย่างที่ 13: Query Builder

```lua
-- Query Builder for PostgreSQL
local QB = {}
QB.__index = QB

function QB.new(table_name)
    local self = setmetatable({}, QB)
    self._table    = table_name
    self._selects  = {}
    self._wheres   = {}
    self._joins    = {}
    self._orders   = {}
    self._limit    = nil
    self._offset   = nil
    self._params   = {}
    self._param_idx = 1
    return self
end

function QB:select(...)
    for _, col in ipairs({...}) do
        table.insert(self._selects, col)
    end
    return self
end

function QB:where(condition, ...)
    -- Replace ? with $N
    local processed = condition
    for _, val in ipairs({...}) do
        processed = processed:gsub("%?", "$" .. self._param_idx, 1)
        table.insert(self._params, val)
        self._param_idx = self._param_idx + 1
    end
    table.insert(self._wheres, processed)
    return self
end

function QB:where_in(col, values)
    local placeholders = {}
    for _, val in ipairs(values) do
        table.insert(placeholders, "$" .. self._param_idx)
        table.insert(self._params, val)
        self._param_idx = self._param_idx + 1
    end
    table.insert(self._wheres, col .. " IN (" .. table.concat(placeholders, ", ") .. ")")
    return self
end

function QB:join(type, table, condition)
    table.insert(self._joins, string.format("%s JOIN %s ON %s", type:upper(), table, condition))
    return self
end

function QB:order(col, direction)
    table.insert(self._orders, col .. " " .. (direction or "ASC"):upper())
    return self
end

function QB:limit(n)
    self._limit = n
    return self
end

function QB:offset(n)
    self._offset = n
    return self
end

function QB:build()
    local parts = {}
    
    -- SELECT
    local cols = #self._selects > 0 and table.concat(self._selects, ", ") or "*"
    table.insert(parts, "SELECT " .. cols)
    
    -- FROM
    table.insert(parts, "FROM " .. self._table)
    
    -- JOINs
    for _, join in ipairs(self._joins) do
        table.insert(parts, join)
    end
    
    -- WHERE
    if #self._wheres > 0 then
        table.insert(parts, "WHERE " .. table.concat(self._wheres, "\n  AND "))
    end
    
    -- ORDER BY
    if #self._orders > 0 then
        table.insert(parts, "ORDER BY " .. table.concat(self._orders, ", "))
    end
    
    -- LIMIT/OFFSET
    if self._limit then
        table.insert(parts, "LIMIT " .. self._limit)
    end
    if self._offset then
        table.insert(parts, "OFFSET " .. self._offset)
    end
    
    return table.concat(parts, "\n"), self._params
end

-- ทดสอบ
print("=== Query Builder ===")

-- Simple select
local sql1, p1 = QB.new("users")
    :select("id", "name", "email")
    :where("age > ?", 25)
    :where("role = ?", "admin")
    :order("name", "asc")
    :limit(10)
    :build()
print("Query 1:")
print(sql1)
print("Params:", table.concat(p1, ", "))

-- Complex query with join
print("\nQuery 2 (with join):")
local sql2, p2 = QB.new("posts p")
    :select("p.id", "p.title", "u.name as author", "COUNT(c.id) as comment_count")
    :join("inner", "users u", "u.id = p.user_id")
    :join("left", "comments c", "c.post_id = p.id")
    :where("p.published = ?", true)
    :where("p.created_at > ?", "2024-01-01")
    :where_in("p.user_id", { 1, 2, 3 })
    :order("p.created_at", "desc")
    :limit(20)
    :offset(0)
    :build()
print(sql2)
print("Params:", table.concat(p2, ", "))
```

---

## ตัวอย่างที่ 14: ORM Concepts

```lua
-- Simple ORM implementation
local Model = {}
Model.__index = Model

local model_registry = {}

function Model.define(name, config)
    local cls = setmetatable({}, { __index = Model })
    cls.__index = cls
    cls._table     = config.table or name:lower() .. "s"
    cls._columns   = config.columns or {}
    cls._relations = config.relations or {}
    cls._name      = name
    
    model_registry[name] = cls
    
    -- Create instance
    cls.new = function(data)
        local instance = setmetatable({}, cls)
        instance._data = {}
        instance._dirty = {}
        instance._new   = true
        
        for _, col in ipairs(cls._columns) do
            instance._data[col.name] = data and data[col.name] or col.default
        end
        
        return instance
    end
    
    -- Find by ID
    cls.find = function(id)
        print(string.format("[ORM] SELECT * FROM %s WHERE id = %s", cls._table, id))
        -- Mock return
        return cls.new({ id = id })
    end
    
    -- Find with conditions
    cls.where = function(conditions)
        local where_parts = {}
        local params = {}
        for col, val in pairs(conditions) do
            table.insert(where_parts, col .. " = ?")
            table.insert(params, val)
        end
        print(string.format("[ORM] SELECT * FROM %s WHERE %s", 
            cls._table, table.concat(where_parts, " AND ")))
        return { cls.new(conditions) }
    end
    
    -- All records
    cls.all = function(limit)
        print(string.format("[ORM] SELECT * FROM %s%s",
            cls._table, limit and (" LIMIT " .. limit) or ""))
        return {}
    end
    
    return cls
end

-- Instance methods
function Model:get(field)
    return self._data[field]
end

function Model:set(field, value)
    self._data[field] = value
    self._dirty[field] = true
    return self
end

function Model:save()
    if self._new then
        print(string.format("[ORM] INSERT INTO %s (%s) VALUES (%s)",
            self.__index._table,
            (function()
                local cols = {}
                for k in pairs(self._data) do table.insert(cols, k) end
                return table.concat(cols, ", ")
            end)(),
            (function()
                local vals = {}
                for _, v in pairs(self._data) do table.insert(vals, tostring(v or "NULL")) end
                return table.concat(vals, ", ")
            end)()
        ))
        self._new = false
    elseif next(self._dirty) then
        local sets = {}
        for col in pairs(self._dirty) do
            table.insert(sets, col .. " = " .. tostring(self._data[col] or "NULL"))
        end
        print(string.format("[ORM] UPDATE %s SET %s WHERE id = %s",
            self.__index._table,
            table.concat(sets, ", "),
            tostring(self._data.id or "NULL")
        ))
        self._dirty = {}
    end
    return self
end

function Model:delete()
    print(string.format("[ORM] DELETE FROM %s WHERE id = %s",
        self.__index._table, tostring(self._data.id or "NULL")))
end

function Model:to_table()
    return self._data
end

-- Define models
local User = Model.define("User", {
    table = "users",
    columns = {
        { name = "id",         type = "integer" },
        { name = "name",       type = "string" },
        { name = "email",      type = "string" },
        { name = "role",       type = "string", default = "user" },
        { name = "created_at", type = "timestamp" },
    },
})

local Post = Model.define("Post", {
    table = "posts",
    columns = {
        { name = "id",         type = "integer" },
        { name = "title",      type = "string" },
        { name = "content",    type = "text" },
        { name = "user_id",    type = "integer" },
        { name = "published",  type = "boolean", default = false },
    },
})

-- ทดสอบ
print("=== ORM Concepts ===")

-- Create
local user = User.new({ name = "Alice", email = "alice@example.com" })
user:save()

-- Find
local found = User.find(1)
print("Found user id:", found:get("id"))

-- Update
found:set("role", "admin"):set("name", "Alice Admin"):save()

-- Query
local admins = User.where({ role = "admin" })
print("Found admins:", #admins)

-- New post
local post = Post.new({
    title    = "My First Post",
    content  = "Hello World!",
    user_id  = 1,
})
post:set("published", true):save()
```

---

## ตัวอย่างที่ 15: EXPLAIN ANALYZE & Performance

```lua
-- Query Performance Analysis
local PgPerformance = {}

-- Parse EXPLAIN ANALYZE output (simplified)
function PgPerformance.parse_explain(explain_output)
    local lines = {}
    for line in (explain_output .. "\n"):gmatch("([^\n]*)\n") do
        table.insert(lines, line)
    end
    
    local result = {
        plan_type   = nil,
        total_cost  = nil,
        actual_time = nil,
        rows        = nil,
        loops       = nil,
        index_used  = false,
        seq_scan    = false,
    }
    
    for _, line in ipairs(lines) do
        -- Extract cost
        local cost = line:match("cost=(%d+%.%d+)%.%.(%d+%.%d+)")
        if cost then result.total_cost = tonumber(cost) end
        
        -- Extract actual time
        local time = line:match("actual time=(%d+%.%d+)%.%.(%d+%.%d+)")
        if time then result.actual_time = tonumber(time) end
        
        -- Check for index scan
        if line:match("Index Scan") or line:match("Index Only Scan") then
            result.index_used = true
            result.plan_type = "Index Scan"
        end
        
        -- Check for sequential scan (potentially slow)
        if line:match("Seq Scan") then
            result.seq_scan = true
            result.plan_type = "Seq Scan"
        end
        
        -- Hash Join, Merge Join etc
        if line:match("Hash Join") then result.plan_type = "Hash Join" end
        if line:match("Nested Loop") then result.plan_type = "Nested Loop" end
    end
    
    return result
end

-- Recommend indexes based on query patterns
function PgPerformance.suggest_indexes(queries)
    local suggestions = {}
    
    for _, q in ipairs(queries) do
        -- Check WHERE clauses
        for col in q:gmatch("[Ww][Hh][Ee][Rr][Ee]%s+([%w_]+)%s*=") do
            table.insert(suggestions, {
                type   = "btree",
                column = col,
                reason = "Equality filter in WHERE clause",
            })
        end
        
        -- Check ORDER BY
        for col in q:gmatch("[Oo][Rr][Dd][Ee][Rr]%s+[Bb][Yy]%s+([%w_]+)") do
            table.insert(suggestions, {
                type   = "btree",
                column = col,
                reason = "ORDER BY clause",
            })
        end
        
        -- Check JOIN conditions
        for col in q:gmatch("[Oo][Nn]%s+[%w_]+%.([%w_]+)%s*=") do
            table.insert(suggestions, {
                type   = "btree",
                column = col,
                reason = "JOIN condition",
            })
        end
    end
    
    -- Deduplicate
    local seen = {}
    local unique = {}
    for _, s in ipairs(suggestions) do
        local key = s.column
        if not seen[key] then
            seen[key] = true
            table.insert(unique, s)
        end
    end
    
    return unique
end

-- N+1 problem detector
function PgPerformance.detect_n_plus_1(query_log)
    local patterns = {}
    local issues = {}
    
    for _, entry in ipairs(query_log) do
        local normalized = entry.sql
            :gsub("%d+", "?")
            :gsub("'[^']*'", "?")
        
        if not patterns[normalized] then
            patterns[normalized] = { count = 0, sql = entry.sql }
        end
        patterns[normalized].count = patterns[normalized].count + 1
    end
    
    for pattern, info in pairs(patterns) do
        if info.count > 5 then  -- threshold
            table.insert(issues, {
                pattern = pattern,
                count   = info.count,
                example = info.sql,
                suggestion = "Consider using JOIN or eager loading",
            })
        end
    end
    
    return issues
end

-- ทดสอบ
print("=== Query Performance ===")

-- EXPLAIN ANALYZE output
local explain_output = [[
Seq Scan on users  (cost=0.00..35.50 rows=1000 width=100) (actual time=0.015..0.542 rows=1000 loops=1)
Planning Time: 0.123 ms
Execution Time: 0.680 ms
]]

local plan = PgPerformance.parse_explain(explain_output)
print("Plan type:", plan.plan_type)
print("Uses index:", plan.index_used)
print("Seq scan:", plan.seq_scan)

-- Index suggestions
print("\nIndex Suggestions:")
local slow_queries = {
    "SELECT * FROM posts WHERE user_id = 1",
    "SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at",
    "SELECT * FROM users u JOIN orders o ON u.id = o.user_id",
}

local suggestions = PgPerformance.suggest_indexes(slow_queries)
for _, s in ipairs(suggestions) do
    print(string.format("  CREATE INDEX ON ... (%s) -- %s", s.column, s.reason))
end

-- N+1 detection
print("\nN+1 Detection:")
local query_log = {}
-- Simulate N+1: 1 query to get posts, then N queries for each post's author
table.insert(query_log, { sql = "SELECT * FROM posts LIMIT 10" })
for i = 1, 10 do
    table.insert(query_log, { sql = "SELECT * FROM users WHERE id = " .. i })
end

local issues = PgPerformance.detect_n_plus_1(query_log)
if #issues > 0 then
    for _, issue in ipairs(issues) do
        print(string.format("  N+1 detected! Query ran %d times", issue.count))
        print("  Pattern:", issue.pattern)
        print("  Fix:", issue.suggestion)
    end
else
    print("  No N+1 issues detected")
end
```

---

## ตัวอย่างที่ 16: Migration Framework

```lua
-- Database Migration System
local Migration = {}
Migration.__index = Migration

function Migration.new(db)
    local self = setmetatable({}, Migration)
    self.db = db
    self.migrations = {}
    self.applied = {}
    return self
end

-- Register migration
function Migration:add(version, description, up_fn, down_fn)
    table.insert(self.migrations, {
        version     = version,
        description = description,
        up          = up_fn,
        down        = down_fn,
        applied_at  = nil,
    })
    -- Sort by version
    table.sort(self.migrations, function(a, b) return a.version < b.version end)
    return self
end

-- Get current version
function Migration:current_version()
    local max_ver = 0
    for ver in pairs(self.applied) do
        if tonumber(ver) > max_ver then
            max_ver = tonumber(ver)
        end
    end
    return max_ver
end

-- Run pending migrations
function Migration:migrate(target_version)
    local current = self:current_version()
    local pending = {}
    
    for _, m in ipairs(self.migrations) do
        local ver = tonumber(m.version)
        if ver > current and (not target_version or ver <= target_version) then
            table.insert(pending, m)
        end
    end
    
    if #pending == 0 then
        print("No pending migrations")
        return 0
    end
    
    local applied_count = 0
    for _, m in ipairs(pending) do
        print(string.format("[Migration] Running %s: %s", m.version, m.description))
        local ok, err = pcall(m.up, self.db)
        if ok then
            self.applied[m.version] = os.time()
            applied_count = applied_count + 1
            print(string.format("[Migration] ✓ %s applied", m.version))
        else
            print(string.format("[Migration] ✗ %s FAILED: %s", m.version, err))
            break
        end
    end
    
    return applied_count
end

-- Rollback last N migrations
function Migration:rollback(steps)
    steps = steps or 1
    local applied_list = {}
    
    for ver, time in pairs(self.applied) do
        table.insert(applied_list, { version = ver, time = time })
    end
    table.sort(applied_list, function(a, b)
        return tonumber(a.version) > tonumber(b.version)
    end)
    
    local rolled_back = 0
    for i = 1, math.min(steps, #applied_list) do
        local entry = applied_list[i]
        -- Find migration definition
        for _, m in ipairs(self.migrations) do
            if m.version == entry.version and m.down then
                print(string.format("[Migration] Rolling back %s: %s", m.version, m.description))
                local ok, err = pcall(m.down, self.db)
                if ok then
                    self.applied[m.version] = nil
                    rolled_back = rolled_back + 1
                    print(string.format("[Migration] ✓ %s rolled back", m.version))
                else
                    print(string.format("[Migration] ✗ Rollback %s FAILED: %s", m.version, err))
                    break
                end
            end
        end
    end
    
    return rolled_back
end

function Migration:status()
    local status_list = {}
    for _, m in ipairs(self.migrations) do
        table.insert(status_list, {
            version     = m.version,
            description = m.description,
            applied     = self.applied[m.version] ~= nil,
            applied_at  = self.applied[m.version],
        })
    end
    return status_list
end

-- ทดสอบ
print("=== Migration Framework ===")
local mock_db = {}

local migrator = Migration.new(mock_db)

-- Define migrations
migrator:add("001", "Create users table", function(db)
    print("  CREATE TABLE users (id SERIAL PRIMARY KEY, name TEXT, email TEXT)")
    print("  CREATE INDEX idx_users_email ON users (email)")
end, function(db)
    print("  DROP TABLE users")
end)

migrator:add("002", "Add role column to users", function(db)
    print("  ALTER TABLE users ADD COLUMN role TEXT DEFAULT 'user'")
    print("  UPDATE users SET role = 'user' WHERE role IS NULL")
end, function(db)
    print("  ALTER TABLE users DROP COLUMN role")
end)

migrator:add("003", "Create posts table", function(db)
    print("  CREATE TABLE posts (id SERIAL PRIMARY KEY, title TEXT, user_id INT, published BOOLEAN)")
    print("  CREATE INDEX idx_posts_user_id ON posts (user_id)")
end, function(db)
    print("  DROP TABLE posts")
end)

migrator:add("004", "Add metadata JSON column", function(db)
    print("  ALTER TABLE users ADD COLUMN metadata JSONB DEFAULT '{}'")
    print("  CREATE INDEX idx_users_metadata ON users USING gin(metadata)")
end, function(db)
    print("  ALTER TABLE users DROP COLUMN metadata")
end)

-- Run migrations
print("\nRunning all migrations:")
local applied = migrator:migrate()
print(string.format("\nApplied %d migrations", applied))

-- Status
print("\nMigration Status:")
for _, s in ipairs(migrator:status()) do
    print(string.format("  [%s] %s - %s",
        s.applied and "✓" or " ",
        s.version,
        s.description))
end

-- Rollback
print("\nRolling back 2 migrations:")
migrator:rollback(2)

print("\nAfter rollback - Current version:", migrator:current_version())
```

---

## สรุปบทที่ 58

ในบทนี้เราได้เรียนรู้:

1. **Connection setup** - DSN parsing, connection strings
2. **Mock PostgreSQL client** - สำหรับ testing
3. **Prepared statements** - SQL injection prevention
4. **Transactions** - BEGIN/COMMIT/ROLLBACK
5. **Savepoints** - Partial rollback
6. **Batch operations** - Bulk INSERT, UPDATE, COPY
7. **JSON columns** - JSONB operators, jsonb_set
8. **Array columns** - @>, &&, ANY, array_append
9. **Full-text search** - tsvector, tsquery, ts_rank
10. **Stored procedures** - Calling from Lua
11. **LISTEN/NOTIFY** - Real-time events
12. **Connection pooling** - Pool management
13. **Query builder** - Fluent interface
14. **ORM concepts** - Model definition, CRUD
15. **EXPLAIN ANALYZE** - Performance analysis
16. **N+1 detection** - Query optimization
17. **Migration framework** - Schema versioning

> **Note**: ในระบบ production ใช้ `lua-resty-postgres` (OpenResty) หรือ `luapgsql` สำหรับ standalone Lua
