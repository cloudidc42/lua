# บทที่ 78: DSL (Domain Specific Language) ใน Lua

## บทนำ

DSL (Domain Specific Language) คือ language ที่ออกแบบมาเฉพาะสำหรับ domain หนึ่งๆ ใน Lua เราสามารถสร้าง DSL ได้ง่ายมากเนื่องจาก syntax ที่ยืดหยุ่น และ metatables ที่ทรงพลัง

---

## 1. Internal DSL vs External DSL

```lua
-- Internal DSL: เขียนใน Lua, ใช้ Lua syntax
-- External DSL: มี syntax เป็นของตัวเอง, ต้องมี parser

-- === Internal DSL Example ===
-- Configuration DSL (Internal - ใช้ Lua syntax)
local config = {
    server = {
        host = "localhost",
        port = 8080,
        workers = 4
    },
    database = {
        url = "postgres://localhost/mydb",
        poolSize = 10
    }
}

-- Better Internal DSL style
local function configDSL(fn)
    local cfg = {}
    
    local dsl = {
        server = function(serverConfig)
            cfg.server = serverConfig
        end,
        database = function(dbConfig)
            cfg.database = dbConfig
        end,
        logging = function(logConfig)
            cfg.logging = logConfig
        end
    }
    
    fn(dsl)
    return cfg
end

local myConfig = configDSL(function(c)
    c.server {
        host = "localhost",
        port = 8080,
        workers = 4
    }
    c.database {
        url = "postgres://localhost/mydb",
        poolSize = 10
    }
    c.logging {
        level = "info",
        file = "/var/log/app.log"
    }
end)

print("Server host:", myConfig.server.host)
print("DB pool size:", myConfig.database.poolSize)
print("Log level:", myConfig.logging.level)
```

---

## 2. Method Chaining สำหรับ DSL

```lua
-- Method Chaining เป็น pattern หลักของ Internal DSL
-- ทำให้ code อ่านง่ายและ fluent

local QueryDSL = {}
QueryDSL.__index = QueryDSL

function QueryDSL.from(tableName)
    return setmetatable({
        _table = tableName,
        _fields = {},
        _joins = {},
        _conditions = {},
        _groupBy = nil,
        _having = nil,
        _orderBy = {},
        _limit = nil,
        _offset = nil
    }, QueryDSL)
end

function QueryDSL:select(...)
    self._fields = { ... }
    return self
end

function QueryDSL:join(table, condition, type_)
    table.insert(self._joins, {
        table = table,
        condition = condition,
        type = type_ or "INNER"
    })
    return self
end

function QueryDSL:where(condition, params)
    table.insert(self._conditions, { condition = condition, params = params })
    return self
end

function QueryDSL:andWhere(condition)
    return self:where(condition)
end

function QueryDSL:orWhere(condition)
    if #self._conditions > 0 then
        local last = self._conditions[#self._conditions]
        last.or_ = true
    end
    return self:where(condition)
end

function QueryDSL:groupBy(...)
    self._groupBy = { ... }
    return self
end

function QueryDSL:having(condition)
    self._having = condition
    return self
end

function QueryDSL:orderBy(field, dir)
    table.insert(self._orderBy, field .. " " .. (dir or "ASC"))
    return self
end

function QueryDSL:limit(n)
    self._limit = n
    return self
end

function QueryDSL:offset(n)
    self._offset = n
    return self
end

function QueryDSL:build()
    local parts = {}
    
    -- SELECT
    local fields = #self._fields > 0 and table.concat(self._fields, ", ") or "*"
    table.insert(parts, "SELECT " .. fields)
    
    -- FROM
    table.insert(parts, "FROM " .. self._table)
    
    -- JOINs
    for _, j in ipairs(self._joins) do
        table.insert(parts, string.format("%s JOIN %s ON %s", j.type, j.table, j.condition))
    end
    
    -- WHERE
    if #self._conditions > 0 then
        local conditions = {}
        for i, cond in ipairs(self._conditions) do
            if i > 1 and cond.or_ then
                table.insert(conditions, "OR " .. cond.condition)
            elseif i > 1 then
                table.insert(conditions, "AND " .. cond.condition)
            else
                table.insert(conditions, cond.condition)
            end
        end
        table.insert(parts, "WHERE " .. table.concat(conditions, " "))
    end
    
    -- GROUP BY
    if self._groupBy then
        table.insert(parts, "GROUP BY " .. table.concat(self._groupBy, ", "))
    end
    
    -- HAVING
    if self._having then
        table.insert(parts, "HAVING " .. self._having)
    end
    
    -- ORDER BY
    if #self._orderBy > 0 then
        table.insert(parts, "ORDER BY " .. table.concat(self._orderBy, ", "))
    end
    
    -- LIMIT / OFFSET
    if self._limit then
        table.insert(parts, "LIMIT " .. self._limit)
    end
    if self._offset then
        table.insert(parts, "OFFSET " .. self._offset)
    end
    
    return table.concat(parts, "\n")
end

-- Test query DSL
local query = QueryDSL
    .from("users u")
    :select("u.id", "u.name", "u.email", "COUNT(o.id) as order_count")
    :join("orders o", "o.user_id = u.id", "LEFT")
    :where("u.active = 1")
    :andWhere("u.age >= 18")
    :groupBy("u.id", "u.name", "u.email")
    :having("COUNT(o.id) > 0")
    :orderBy("order_count", "DESC")
    :limit(10)
    :offset(0)
    :build()

print("Generated SQL:")
print(query)
```

---

## 3. Builder Pattern เป็น DSL

```lua
-- Builder Pattern: ก้าวสู่ DSL ที่สมบูรณ์
local HttpRequestBuilder = {}
HttpRequestBuilder.__index = HttpRequestBuilder

function HttpRequestBuilder.request(method, url)
    return setmetatable({
        _method = method:upper(),
        _url = url,
        _headers = {},
        _queryParams = {},
        _body = nil,
        _timeout = 30,
        _retries = 0,
        _auth = nil
    }, HttpRequestBuilder)
end

-- Convenience methods
function HttpRequestBuilder.get(url) return HttpRequestBuilder.request("GET", url) end
function HttpRequestBuilder.post(url) return HttpRequestBuilder.request("POST", url) end
function HttpRequestBuilder.put(url) return HttpRequestBuilder.request("PUT", url) end
function HttpRequestBuilder.delete(url) return HttpRequestBuilder.request("DELETE", url) end

function HttpRequestBuilder:header(name, value)
    self._headers[name] = value
    return self
end

function HttpRequestBuilder:contentType(type_)
    return self:header("Content-Type", type_)
end

function HttpRequestBuilder:accept(type_)
    return self:header("Accept", type_)
end

function HttpRequestBuilder:bearer(token)
    self._auth = { type = "bearer", token = token }
    return self:header("Authorization", "Bearer " .. token)
end

function HttpRequestBuilder:basic(username, password)
    -- Base64 encode (simplified)
    self._auth = { type = "basic", username = username, password = password }
    return self:header("Authorization", "Basic " .. username .. ":" .. password)
end

function HttpRequestBuilder:query(params)
    for k, v in pairs(params) do
        self._queryParams[k] = tostring(v)
    end
    return self
end

function HttpRequestBuilder:body(data, contentType)
    self._body = data
    if contentType then
        self:contentType(contentType)
    end
    return self
end

function HttpRequestBuilder:json(data)
    -- Serialize to JSON (simplified)
    local function toJSON(v)
        if type(v) == "table" then
            local parts = {}
            for k, val in pairs(v) do
                if type(k) == "string" then
                    table.insert(parts, '"' .. k .. '":' .. toJSON(val))
                end
            end
            return "{" .. table.concat(parts, ",") .. "}"
        elseif type(v) == "string" then
            return '"' .. v .. '"'
        else
            return tostring(v)
        end
    end
    return self:body(toJSON(data), "application/json")
end

function HttpRequestBuilder:timeout(seconds)
    self._timeout = seconds
    return self
end

function HttpRequestBuilder:retry(times)
    self._retries = times
    return self
end

function HttpRequestBuilder:build()
    local url = self._url
    
    -- Add query params
    local params = {}
    for k, v in pairs(self._queryParams) do
        table.insert(params, k .. "=" .. v)
    end
    if #params > 0 then
        url = url .. "?" .. table.concat(params, "&")
    end
    
    return {
        method = self._method,
        url = url,
        headers = self._headers,
        body = self._body,
        timeout = self._timeout,
        retries = self._retries
    }
end

function HttpRequestBuilder:toString()
    local req = self:build()
    local lines = { req.method .. " " .. req.url }
    
    for name, value in pairs(req.headers) do
        table.insert(lines, name .. ": " .. value)
    end
    
    if req.body then
        table.insert(lines, "")
        table.insert(lines, req.body)
    end
    
    return table.concat(lines, "\n")
end

-- ใช้งาน Builder DSL
local request = HttpRequestBuilder
    .post("https://api.example.com/users")
    :bearer("my-secret-token")
    :accept("application/json")
    :json({ name = "Alice", email = "alice@example.com" })
    :timeout(10)
    :retry(3)
    :build()

print("Method:", request.method)
print("URL:", request.url)
print("Timeout:", request.timeout)
print("Body:", request.body)

local getRequest = HttpRequestBuilder
    .get("https://api.example.com/users")
    :bearer("token123")
    :query({ page = 1, limit = 20, search = "alice" })
    :toString()

print("\nGET Request:")
print(getRequest)
```

---

## 4. HTML Generator DSL

```lua
-- HTML Generator DSL แบบ Complete
local HTML = {}

-- ฟังก์ชันสร้าง tag
local function tag(name, selfClosing)
    return function(attrs_or_content, content)
        local attrs = {}
        local innerContent = ""
        
        if type(attrs_or_content) == "table" then
            -- Check if it's an attribute table or content table
            local isAttrs = false
            for k, _ in pairs(attrs_or_content) do
                if type(k) == "string" then
                    isAttrs = true
                    break
                end
            end
            
            if isAttrs then
                attrs = attrs_or_content
                if type(content) == "string" then
                    innerContent = content
                elseif type(content) == "table" then
                    innerContent = table.concat(content, "\n")
                end
            else
                -- Array of children
                innerContent = table.concat(attrs_or_content, "\n")
            end
        elseif type(attrs_or_content) == "string" then
            innerContent = attrs_or_content
        end
        
        -- Build attribute string
        local attrStr = ""
        for k, v in pairs(attrs) do
            if v == true then
                attrStr = attrStr .. " " .. k
            elseif v then
                attrStr = attrStr .. string.format(' %s="%s"', k, tostring(v))
            end
        end
        
        if selfClosing then
            return string.format("<%s%s/>", name, attrStr)
        else
            return string.format("<%s%s>%s</%s>", name, attrStr, innerContent, name)
        end
    end
end

-- Define HTML tags
local h = {
    -- Document structure
    html = tag("html"),
    head = tag("head"),
    body = tag("body"),
    title = tag("title"),
    
    -- Headings
    h1 = tag("h1"),
    h2 = tag("h2"),
    h3 = tag("h3"),
    h4 = tag("h4"),
    
    -- Block elements
    div = tag("div"),
    p = tag("p"),
    section = tag("section"),
    article = tag("article"),
    header = tag("header"),
    footer = tag("footer"),
    nav = tag("nav"),
    main = tag("main"),
    aside = tag("aside"),
    
    -- Inline elements
    span = tag("span"),
    a = tag("a"),
    strong = tag("strong"),
    em = tag("em"),
    code = tag("code"),
    
    -- Lists
    ul = tag("ul"),
    ol = tag("ol"),
    li = tag("li"),
    
    -- Table
    table = tag("table"),
    thead = tag("thead"),
    tbody = tag("tbody"),
    tr = tag("tr"),
    th = tag("th"),
    td = tag("td"),
    
    -- Form
    form = tag("form"),
    input = tag("input", true),
    button = tag("button"),
    label = tag("label"),
    select = tag("select"),
    option = tag("option"),
    textarea = tag("textarea"),
    
    -- Media
    img = tag("img", true),
    br = tag("br", true),
    hr = tag("hr", true),
    
    -- Script/Style
    script = tag("script"),
    style = tag("style"),
    link = tag("link", true),
    meta = tag("meta", true)
}

-- Helper for joining elements
local function join(...)
    local parts = {}
    for _, v in ipairs({ ... }) do
        if type(v) == "string" then
            table.insert(parts, v)
        end
    end
    return table.concat(parts, "\n")
end

-- สร้าง HTML Page ด้วย DSL
local function renderPage(data)
    return h.html({ lang = "th" },
        join(
            h.head(join(
                h.meta({ charset = "UTF-8" }),
                h.meta({ name = "viewport", content = "width=device-width, initial-scale=1.0" }),
                h.title(data.title),
                h.style([[
                    body { font-family: sans-serif; margin: 0; padding: 20px; }
                    .container { max-width: 800px; margin: 0 auto; }
                    .card { border: 1px solid #ccc; padding: 15px; margin: 10px 0; border-radius: 8px; }
                    table { width: 100%; border-collapse: collapse; }
                    th, td { padding: 8px; border: 1px solid #ddd; text-align: left; }
                    th { background: #f5f5f5; }
                ]])
            )),
            h.body(
                h.div({ class = "container" },
                    join(
                        h.h1(data.title),
                        h.p(data.description),
                        h.h2("Users"),
                        h.table(join(
                            h.thead(
                                h.tr(join(
                                    h.th("ID"),
                                    h.th("Name"),
                                    h.th("Email"),
                                    h.th("Role")
                                ))
                            ),
                            h.tbody(join(
                                (function()
                                    local rows = {}
                                    for _, user in ipairs(data.users) do
                                        table.insert(rows, h.tr(join(
                                            h.td(tostring(user.id)),
                                            h.td(user.name),
                                            h.td(user.email),
                                            h.td(user.role)
                                        )))
                                    end
                                    return table.concat(rows, "\n")
                                end)()
                            ))
                        ))
                    )
                )
            )
        )
    )
end

local pageData = {
    title = "User Management",
    description = "List of all registered users",
    users = {
        { id = 1, name = "Alice", email = "alice@example.com", role = "admin" },
        { id = 2, name = "Bob", email = "bob@example.com", role = "user" },
        { id = 3, name = "Charlie", email = "charlie@example.com", role = "moderator" }
    }
}

local html_output = renderPage(pageData)
print("Generated HTML (first 500 chars):")
print(html_output:sub(1, 500))
print("...")
print("Total length:", #html_output, "chars")
```

---

## 5. SQL Builder DSL

```lua
-- SQL Builder DSL แบบ Complete
local SQL = {}
SQL.__index = SQL

-- Condition builder
local Condition = {}
Condition.__index = Condition

function Condition.new(expr, params)
    return setmetatable({ expr = expr, params = params or {} }, Condition)
end

function Condition:and_(other)
    local combined = Condition.new(
        "(" .. self.expr .. " AND " .. other.expr .. ")"
    )
    for _, p in ipairs(self.params) do table.insert(combined.params, p) end
    for _, p in ipairs(other.params) do table.insert(combined.params, p) end
    return combined
end

function Condition:or_(other)
    local combined = Condition.new(
        "(" .. self.expr .. " OR " .. other.expr .. ")"
    )
    for _, p in ipairs(self.params) do table.insert(combined.params, p) end
    for _, p in ipairs(other.params) do table.insert(combined.params, p) end
    return combined
end

-- Condition helpers
local function eq(field, value)
    return Condition.new(field .. " = ?", { value })
end

local function ne(field, value)
    return Condition.new(field .. " != ?", { value })
end

local function gt(field, value)
    return Condition.new(field .. " > ?", { value })
end

local function gte(field, value)
    return Condition.new(field .. " >= ?", { value })
end

local function lt(field, value)
    return Condition.new(field .. " < ?", { value })
end

local function lte(field, value)
    return Condition.new(field .. " <= ?", { value })
end

local function like(field, pattern)
    return Condition.new(field .. " LIKE ?", { pattern })
end

local function inList(field, values)
    local placeholders = {}
    for _ in ipairs(values) do
        table.insert(placeholders, "?")
    end
    return Condition.new(
        field .. " IN (" .. table.concat(placeholders, ", ") .. ")",
        values
    )
end

local function isNull(field)
    return Condition.new(field .. " IS NULL")
end

local function isNotNull(field)
    return Condition.new(field .. " IS NOT NULL")
end

local function between(field, min, max)
    return Condition.new(field .. " BETWEEN ? AND ?", { min, max })
end

-- Main SQL builder
local SelectBuilder = {}
SelectBuilder.__index = SelectBuilder

function SelectBuilder.new()
    return setmetatable({
        _select = {},
        _from = nil,
        _joins = {},
        _where = nil,
        _groupBy = {},
        _having = nil,
        _orderBy = {},
        _limit = nil,
        _offset = nil,
        _params = {}
    }, SelectBuilder)
end

function SelectBuilder:select(...)
    self._select = { ... }
    return self
end

function SelectBuilder:from(table_, alias)
    self._from = alias and (table_ .. " " .. alias) or table_
    return self
end

function SelectBuilder:innerJoin(table_, on)
    table.insert(self._joins, "INNER JOIN " .. table_ .. " ON " .. on)
    return self
end

function SelectBuilder:leftJoin(table_, on)
    table.insert(self._joins, "LEFT JOIN " .. table_ .. " ON " .. on)
    return self
end

function SelectBuilder:where(condition)
    if type(condition) == "string" then
        condition = Condition.new(condition)
    end
    self._where = condition
    return self
end

function SelectBuilder:andWhere(condition)
    if type(condition) == "string" then
        condition = Condition.new(condition)
    end
    if self._where then
        self._where = self._where:and_(condition)
    else
        self._where = condition
    end
    return self
end

function SelectBuilder:orWhere(condition)
    if type(condition) == "string" then
        condition = Condition.new(condition)
    end
    if self._where then
        self._where = self._where:or_(condition)
    else
        self._where = condition
    end
    return self
end

function SelectBuilder:groupBy(...)
    self._groupBy = { ... }
    return self
end

function SelectBuilder:having(condition)
    self._having = condition
    return self
end

function SelectBuilder:orderBy(field, dir)
    table.insert(self._orderBy, field .. " " .. (dir or "ASC"))
    return self
end

function SelectBuilder:limit(n)
    self._limit = n
    return self
end

function SelectBuilder:offset(n)
    self._offset = n
    return self
end

function SelectBuilder:build()
    local parts = {}
    local params = {}
    
    -- SELECT
    local fields = #self._select > 0 and table.concat(self._select, ", ") or "*"
    table.insert(parts, "SELECT " .. fields)
    
    -- FROM
    if self._from then
        table.insert(parts, "FROM " .. self._from)
    end
    
    -- JOINs
    for _, j in ipairs(self._joins) do
        table.insert(parts, j)
    end
    
    -- WHERE
    if self._where then
        table.insert(parts, "WHERE " .. self._where.expr)
        for _, p in ipairs(self._where.params) do
            table.insert(params, p)
        end
    end
    
    -- GROUP BY
    if #self._groupBy > 0 then
        table.insert(parts, "GROUP BY " .. table.concat(self._groupBy, ", "))
    end
    
    -- HAVING
    if self._having then
        table.insert(parts, "HAVING " .. self._having)
    end
    
    -- ORDER BY
    if #self._orderBy > 0 then
        table.insert(parts, "ORDER BY " .. table.concat(self._orderBy, ", "))
    end
    
    -- LIMIT/OFFSET
    if self._limit then
        table.insert(parts, "LIMIT " .. self._limit)
        if self._offset then
            table.insert(parts, "OFFSET " .. self._offset)
        end
    end
    
    return table.concat(parts, "\n"), params
end

-- ใช้งาน SQL DSL
local qb = SelectBuilder.new()
local sql, params = qb
    :select("u.id", "u.name", "u.email", "r.name as role")
    :from("users", "u")
    :leftJoin("roles r", "r.id = u.role_id")
    :where(eq("u.active", 1))
    :andWhere(gte("u.age", 18))
    :andWhere(
        like("u.email", "%@company.com"):or_(
            inList("u.role_id", { 1, 2, 3 })
        )
    )
    :orderBy("u.name")
    :limit(20)
    :offset(0)
    :build()

print("SQL:")
print(sql)
print("\nParams:", table.concat(
    (function()
        local s = {}
        for _, v in ipairs(params) do table.insert(s, tostring(v)) end
        return s
    end)(),
    ", "
))
```

---

## 6. Test Framework DSL (describe/it)

```lua
-- Test Framework DSL - คล้าย RSpec, Mocha, Jasmine
local TestFramework = {}

local currentDescribe = nil
local testResults = { passed = 0, failed = 0, skipped = 0 }
local allTests = {}

-- Colors for output
local RED = "\27[31m"
local GREEN = "\27[32m"
local YELLOW = "\27[33m"
local RESET = "\27[0m"

function TestFramework.describe(name, fn)
    local suite = {
        name = name,
        tests = {},
        parent = currentDescribe,
        beforeEach = {},
        afterEach = {},
        beforeAll = {},
        afterAll = {}
    }
    
    local parentDescribe = currentDescribe
    currentDescribe = suite
    
    fn()
    
    currentDescribe = parentDescribe
    
    if currentDescribe then
        table.insert(currentDescribe.tests, suite)
    else
        table.insert(allTests, suite)
    end
end

function TestFramework.it(name, fn)
    if not currentDescribe then
        error("'it' must be inside 'describe'")
    end
    
    table.insert(currentDescribe.tests, {
        name = name,
        fn = fn,
        _isTest = true
    })
end

function TestFramework.xit(name, fn)
    if currentDescribe then
        table.insert(currentDescribe.tests, {
            name = name,
            fn = fn,
            _isTest = true,
            skipped = true
        })
    end
end

function TestFramework.beforeEach(fn)
    if currentDescribe then
        table.insert(currentDescribe.beforeEach, fn)
    end
end

function TestFramework.afterEach(fn)
    if currentDescribe then
        table.insert(currentDescribe.afterEach, fn)
    end
end

-- Matchers
local function expect(value)
    return {
        value = value,
        
        toBe = function(self, expected)
            if self.value ~= expected then
                error(string.format("Expected %s to be %s", 
                    tostring(self.value), tostring(expected)), 2)
            end
        end,
        
        toEqual = function(self, expected)
            -- Deep equal
            local function deepEqual(a, b)
                if type(a) ~= type(b) then return false end
                if type(a) ~= "table" then return a == b end
                for k, v in pairs(a) do
                    if not deepEqual(v, b[k]) then return false end
                end
                for k, v in pairs(b) do
                    if not deepEqual(v, a[k]) then return false end
                end
                return true
            end
            
            if not deepEqual(self.value, expected) then
                error(string.format("Expected deep equality"), 2)
            end
        end,
        
        toBeTruthy = function(self)
            if not self.value then
                error(string.format("Expected %s to be truthy", tostring(self.value)), 2)
            end
        end,
        
        toBeFalsy = function(self)
            if self.value then
                error(string.format("Expected %s to be falsy", tostring(self.value)), 2)
            end
        end,
        
        toBeNil = function(self)
            if self.value ~= nil then
                error(string.format("Expected nil, got %s", tostring(self.value)), 2)
            end
        end,
        
        toContain = function(self, item)
            if type(self.value) == "table" then
                for _, v in ipairs(self.value) do
                    if v == item then return end
                end
            elseif type(self.value) == "string" then
                if self.value:find(tostring(item), 1, true) then return end
            end
            error(string.format("Expected to contain %s", tostring(item)), 2)
        end,
        
        toThrow = function(self, expectedMsg)
            if type(self.value) ~= "function" then
                error("toThrow() expects a function", 2)
            end
            local ok, err = pcall(self.value)
            if ok then
                error("Expected function to throw", 2)
            end
            if expectedMsg and not err:find(expectedMsg, 1, true) then
                error(string.format("Expected error '%s' to contain '%s'",
                    err, expectedMsg), 2)
            end
        end,
        
        toBeGreaterThan = function(self, n)
            if not (self.value > n) then
                error(string.format("Expected %s to be greater than %s",
                    tostring(self.value), tostring(n)), 2)
            end
        end,
        
        toBeLessThan = function(self, n)
            if not (self.value < n) then
                error(string.format("Expected %s to be less than %s",
                    tostring(self.value), tostring(n)), 2)
            end
        end,
        
        not_ = {
            -- Create NOT versions
        }
    }
end

-- Run test suite
local function runSuite(suite, depth, parentBeforeEach, parentAfterEach)
    depth = depth or 0
    parentBeforeEach = parentBeforeEach or {}
    parentAfterEach = parentAfterEach or {}
    
    local indent = string.rep("  ", depth)
    print(indent .. suite.name)
    
    -- Combine hooks
    local allBefore = {}
    for _, h in ipairs(parentBeforeEach) do table.insert(allBefore, h) end
    for _, h in ipairs(suite.beforeEach) do table.insert(allBefore, h) end
    
    local allAfter = {}
    for _, h in ipairs(suite.afterEach) do table.insert(allAfter, h) end
    for _, h in ipairs(parentAfterEach) do table.insert(allAfter, h) end
    
    for _, test in ipairs(suite.tests) do
        if test._isTest then
            if test.skipped then
                print(indent .. "  " .. YELLOW .. "○ " .. test.name .. RESET)
                testResults.skipped = testResults.skipped + 1
            else
                -- Run hooks
                for _, hook in ipairs(allBefore) do hook() end
                
                local ok, err = pcall(test.fn)
                
                for _, hook in ipairs(allAfter) do hook() end
                
                if ok then
                    print(indent .. "  " .. GREEN .. "✓ " .. test.name .. RESET)
                    testResults.passed = testResults.passed + 1
                else
                    print(indent .. "  " .. RED .. "✗ " .. test.name .. RESET)
                    print(indent .. "    " .. RED .. tostring(err) .. RESET)
                    testResults.failed = testResults.failed + 1
                end
            end
        else
            -- Nested describe
            runSuite(test, depth + 1, allBefore, allAfter)
        end
    end
end

function TestFramework.run()
    for _, suite in ipairs(allTests) do
        runSuite(suite)
    end
    
    print(string.format("\n%d passed, %d failed, %d skipped",
        testResults.passed, testResults.failed, testResults.skipped))
end

-- ใช้งาน Test DSL
local describe = TestFramework.describe
local it = TestFramework.it
local xit = TestFramework.xit
local beforeEach = TestFramework.beforeEach

describe("Calculator", function()
    local calc
    
    beforeEach(function()
        calc = { value = 0 }
        calc.add = function(self, n) self.value = self.value + n end
        calc.sub = function(self, n) self.value = self.value - n end
        calc.mul = function(self, n) self.value = self.value * n end
    end)
    
    describe("addition", function()
        it("adds positive numbers", function()
            calc:add(5)
            expect(calc.value):toBe(5)
        end)
        
        it("adds negative numbers", function()
            calc:add(-3)
            expect(calc.value):toBe(-3)
        end)
    end)
    
    describe("subtraction", function()
        it("subtracts numbers", function()
            calc:add(10)
            calc:sub(3)
            expect(calc.value):toBe(7)
        end)
    end)
    
    xit("handles division by zero", function()
        -- Skipped test
    end)
end)

describe("String utilities", function()
    it("concatenates strings", function()
        expect("Hello" .. " " .. "World"):toBe("Hello World")
    end)
    
    it("checks string length", function()
        expect(#"Hello"):toBe(5)
    end)
    
    it("finds substring", function()
        expect("Hello World"):toContain("World")
    end)
    
    it("throws on nil concat", function()
        expect(function()
            local x = nil
            local y = "hello" .. x
        end):toThrow()
    end)
end)

TestFramework.run()
```

---

## 7. Config DSL

```lua
-- Configuration DSL แบบ Complete
local ConfigDSL = {}

function ConfigDSL.define(fn)
    local config = {
        _data = {},
        _validators = {},
        _defaults = {}
    }
    
    local dsl = setmetatable({}, {
        __index = function(t, section)
            return function(data)
                config._data[section] = data
                return config
            end
        end
    })
    
    -- Special methods
    dsl.validate = function(rules)
        config._validators = rules
    end
    
    dsl.defaults = function(defaults)
        config._defaults = defaults
    end
    
    fn(dsl)
    
    -- Apply defaults
    for section, defaults in pairs(config._defaults) do
        if not config._data[section] then
            config._data[section] = defaults
        else
            for k, v in pairs(defaults) do
                if config._data[section][k] == nil then
                    config._data[section][k] = v
                end
            end
        end
    end
    
    -- Create read-only config accessor
    local proxy = setmetatable({}, {
        __index = function(t, key)
            return config._data[key]
        end,
        __newindex = function(t, key, value)
            error("Config is read-only. Use ConfigDSL.define() to create config")
        end
    })
    
    return proxy
end

-- App configuration
local appConfig = ConfigDSL.define(function(config)
    config.defaults {
        server = {
            host = "0.0.0.0",
            port = 3000,
            debug = false,
            workers = 1
        },
        security = {
            secretKey = "change-me",
            jwtExpiry = 3600,
            bcryptRounds = 10
        }
    }
    
    config.server {
        host = "localhost",
        port = 8080,
        workers = 4,
        debug = true,
        ssl = {
            enabled = false,
            cert = "/etc/ssl/cert.pem",
            key = "/etc/ssl/key.pem"
        }
    }
    
    config.database {
        driver = "postgres",
        host = "localhost",
        port = 5432,
        name = "myapp_db",
        user = "myapp",
        pool = {
            min = 2,
            max = 10,
            idle = 30000
        }
    }
    
    config.cache {
        driver = "redis",
        host = "localhost",
        port = 6379,
        ttl = 3600
    }
    
    config.logging {
        level = "info",
        format = "json",
        destinations = { "stdout", "file" },
        file = {
            path = "/var/log/app.log",
            maxSize = "100mb",
            maxFiles = 5
        }
    }
end)

-- Access config
print("Server:", appConfig.server.host .. ":" .. appConfig.server.port)
print("DB:", appConfig.database.driver .. "://" .. appConfig.database.host)
print("Cache TTL:", appConfig.cache.ttl)
print("Security defaults preserved:", appConfig.security.jwtExpiry)
```

---

## 8. Workflow DSL

```lua
-- Workflow DSL - กำหนด workflow steps
local Workflow = {}
Workflow.__index = Workflow

function Workflow.define(name, fn)
    local workflow = setmetatable({
        name = name,
        _steps = {},
        _onError = nil,
        _onComplete = nil,
        _context = {}
    }, Workflow)
    
    local dsl = {
        step = function(stepName, fn)
            table.insert(workflow._steps, {
                name = stepName,
                fn = fn,
                conditions = {}
            })
        end,
        
        conditional = function(stepName, condition, fn)
            table.insert(workflow._steps, {
                name = stepName,
                fn = fn,
                condition = condition
            })
        end,
        
        parallel = function(stepName, fns)
            table.insert(workflow._steps, {
                name = stepName,
                parallel = true,
                fns = fns
            })
        end,
        
        onError = function(fn)
            workflow._onError = fn
        end,
        
        onComplete = function(fn)
            workflow._onComplete = fn
        end
    }
    
    fn(dsl)
    return workflow
end

function Workflow:run(initialContext)
    self._context = initialContext or {}
    
    print(string.format("[Workflow] Starting: %s", self.name))
    
    for _, step in ipairs(self._steps) do
        -- Check condition
        if step.condition then
            local shouldRun = step.condition(self._context)
            if not shouldRun then
                print(string.format("  [Skip] %s (condition not met)", step.name))
                goto continue
            end
        end
        
        print(string.format("  [Step] %s", step.name))
        
        local ok, err
        if step.parallel then
            -- Run parallel steps (simulated sequentially)
            for pName, pFn in pairs(step.fns) do
                print(string.format("    [Parallel] %s", pName))
                ok, err = pcall(pFn, self._context)
                if not ok then break end
            end
        else
            ok, err = pcall(step.fn, self._context)
        end
        
        if not ok then
            print(string.format("  [Error] in %s: %s", step.name, tostring(err)))
            if self._onError then
                self._onError(err, step, self._context)
            end
            return false, err
        end
        
        ::continue::
    end
    
    print(string.format("[Workflow] Complete: %s", self.name))
    
    if self._onComplete then
        self._onComplete(self._context)
    end
    
    return true, self._context
end

-- ตัวอย่าง User Registration Workflow
local registrationWorkflow = Workflow.define("UserRegistration", function(w)
    w.step("validate_input", function(ctx)
        if not ctx.email or not ctx.password then
            error("Missing required fields")
        end
        if #ctx.password < 8 then
            error("Password too short")
        end
        print("    Input validated")
    end)
    
    w.step("check_email_exists", function(ctx)
        -- Simulate DB check
        ctx.emailExists = false
        print("    Email availability checked:", not ctx.emailExists)
    end)
    
    w.conditional("abort_if_email_exists",
        function(ctx) return ctx.emailExists end,
        function(ctx)
            error("Email already registered")
        end
    )
    
    w.step("hash_password", function(ctx)
        ctx.hashedPassword = "hashed:" .. ctx.password
        ctx.password = nil
        print("    Password hashed")
    end)
    
    w.parallel("notifications", {
        sendWelcomeEmail = function(ctx)
            print("    Sending welcome email to " .. ctx.email)
        end,
        createProfile = function(ctx)
            print("    Creating user profile")
        end
    })
    
    w.step("save_to_database", function(ctx)
        ctx.userId = math.random(1000, 9999)
        print("    Saved to DB with id=" .. ctx.userId)
    end)
    
    w.onError(function(err, step, ctx)
        print("    Rolling back due to error in: " .. step.name)
    end)
    
    w.onComplete(function(ctx)
        print("    Registration complete. User ID:", ctx.userId)
    end)
end)

local success, result = registrationWorkflow:run({
    email = "newuser@example.com",
    password = "securepassword123"
})

print("\nWorkflow success:", success)
if result.userId then
    print("User ID:", result.userId)
end
```

---

## 9. State Machine DSL

```lua
-- State Machine DSL - กำหนด states และ transitions
local StateMachine = {}
StateMachine.__index = StateMachine

function StateMachine.define(config)
    local sm = setmetatable({
        _states = {},
        _transitions = {},
        _currentState = config.initial,
        _history = {},
        _listeners = {}
    }, StateMachine)
    
    -- DSL for defining states
    local function state(name, stateConfig)
        sm._states[name] = {
            name = name,
            onEnter = stateConfig.onEnter,
            onExit = stateConfig.onExit,
            onWhile = stateConfig.onWhile
        }
    end
    
    -- DSL for defining transitions
    local function transition(from, event, to, guard)
        local key = from .. ":" .. event
        sm._transitions[key] = {
            from = from,
            event = event,
            to = to,
            guard = guard
        }
    end
    
    -- Execute config DSL
    config.define(state, transition)
    
    -- Trigger initial state enter
    local initState = sm._states[sm._currentState]
    if initState and initState.onEnter then
        initState.onEnter(sm)
    end
    
    return sm
end

function StateMachine:send(event, data)
    local key = self._currentState .. ":" .. event
    local trans = self._transitions[key]
    
    if not trans then
        print(string.format("[SM] No transition: %s -> %s (ignored)", 
            self._currentState, event))
        return false
    end
    
    -- Check guard
    if trans.guard and not trans.guard(self, data) then
        print(string.format("[SM] Guard rejected: %s", event))
        return false
    end
    
    -- Exit current state
    local currentState = self._states[self._currentState]
    if currentState and currentState.onExit then
        currentState.onExit(self, data)
    end
    
    -- Record history
    table.insert(self._history, {
        from = self._currentState,
        event = event,
        to = trans.to,
        time = os.time()
    })
    
    -- Notify listeners
    if self._listeners[event] then
        for _, listener in ipairs(self._listeners[event]) do
            listener(self._currentState, trans.to, data)
        end
    end
    
    -- Enter new state
    self._currentState = trans.to
    local newState = self._states[self._currentState]
    if newState and newState.onEnter then
        newState.onEnter(self, data)
    end
    
    return true
end

function StateMachine:on(event, fn)
    if not self._listeners[event] then
        self._listeners[event] = {}
    end
    table.insert(self._listeners[event], fn)
    return self
end

function StateMachine:is(stateName)
    return self._currentState == stateName
end

function StateMachine:state()
    return self._currentState
end

-- ตัวอย่าง: Order Status State Machine
local orderSM = StateMachine.define({
    initial = "cart",
    define = function(state, transition)
        state("cart", {
            onEnter = function(sm)
                print("[State] Entered: cart")
            end
        })
        
        state("pending_payment", {
            onEnter = function(sm)
                print("[State] Entered: pending_payment")
            end
        })
        
        state("paid", {
            onEnter = function(sm)
                print("[State] Entered: paid - processing order")
            end
        })
        
        state("shipped", {
            onEnter = function(sm)
                print("[State] Entered: shipped")
            end
        })
        
        state("delivered", {
            onEnter = function(sm)
                print("[State] Entered: delivered")
            end
        })
        
        state("cancelled", {
            onEnter = function(sm)
                print("[State] Entered: cancelled")
            end
        })
        
        -- Transitions
        transition("cart", "checkout", "pending_payment")
        transition("pending_payment", "pay", "paid")
        transition("pending_payment", "cancel", "cancelled")
        transition("paid", "ship", "shipped")
        transition("shipped", "deliver", "delivered")
        
        -- Any state can be cancelled (except delivered)
        transition("cart", "cancel", "cancelled")
        transition("paid", "cancel", "cancelled",
            function(sm, data)
                -- Can only cancel paid orders within 1 hour
                print("[Guard] Checking cancellation eligibility")
                return true
            end
        )
    end
})

print("Current state:", orderSM:state())

orderSM:send("checkout")
orderSM:send("pay")
orderSM:send("ship")
orderSM:send("deliver")

print("\nFinal state:", orderSM:state())
print("Is delivered:", orderSM:is("delivered"))
print("History entries:", #orderSM._history)
```

---

## 10. Tokenizer สำหรับ External DSL

```lua
-- Tokenizer: แปลง string เป็น tokens
local Tokenizer = {}
Tokenizer.__index = Tokenizer

function Tokenizer.new(rules)
    return setmetatable({ rules = rules }, Tokenizer)
end

function Tokenizer:tokenize(input)
    local tokens = {}
    local pos = 1
    
    while pos <= #input do
        -- Skip whitespace
        local ws = input:match("^%s+", pos)
        if ws then
            pos = pos + #ws
        end
        
        if pos > #input then break end
        
        local matched = false
        
        for _, rule in ipairs(self.rules) do
            local pattern, tokenType = rule[1], rule[2]
            local match = input:match("^" .. pattern, pos)
            
            if match then
                if tokenType ~= "SKIP" then
                    table.insert(tokens, {
                        type = tokenType,
                        value = match,
                        pos = pos
                    })
                end
                pos = pos + #match
                matched = true
                break
            end
        end
        
        if not matched then
            error(string.format("Unexpected character at position %d: '%s'",
                pos, input:sub(pos, pos)))
        end
    end
    
    table.insert(tokens, { type = "EOF", value = "", pos = pos })
    return tokens
end

-- ตัวอย่าง: Simple Expression Tokenizer
local expressionTokenizer = Tokenizer.new({
    { "%d+%.%d+", "FLOAT" },
    { "%d+", "INT" },
    { "[a-zA-Z_][a-zA-Z0-9_]*", "IDENT" },
    { "%+", "PLUS" },
    { "%-", "MINUS" },
    { "%*", "STAR" },
    { "/", "SLASH" },
    { "%(", "LPAREN" },
    { "%)", "RPAREN" },
    { "==", "EQ" },
    { "!=", "NEQ" },
    { "<=", "LTE" },
    { ">=", "GTE" },
    { "<", "LT" },
    { ">", "GT" },
    { "=", "ASSIGN" },
    { '"[^"]*"', "STRING" },
    { "'[^']*'", "STRING" },
    { ",", "COMMA" },
    { ";", "SEMICOLON" },
    { "%s+", "SKIP" }
})

local tokens = expressionTokenizer:tokenize("x = (10 + 20) * 3.14")

print("Tokens:")
for _, token in ipairs(tokens) do
    if token.type ~= "EOF" then
        print(string.format("  %-12s '%s'", token.type, token.value))
    end
end
```

---

## 11. Recursive Descent Parser

```lua
-- Recursive Descent Parser สำหรับ arithmetic expressions
local Parser = {}
Parser.__index = Parser

function Parser.new(tokens)
    return setmetatable({
        tokens = tokens,
        pos = 1
    }, Parser)
end

function Parser:peek()
    return self.tokens[self.pos]
end

function Parser:consume(expectedType)
    local token = self.tokens[self.pos]
    if expectedType and token.type ~= expectedType then
        error(string.format("Expected %s but got %s at pos %d",
            expectedType, token.type, token.pos))
    end
    self.pos = self.pos + 1
    return token
end

function Parser:match(type_)
    if self:peek().type == type_ then
        return self:consume(type_)
    end
    return nil
end

-- Grammar:
-- expr    := term (('+' | '-') term)*
-- term    := factor (('*' | '/') factor)*
-- factor  := NUMBER | '(' expr ')' | IDENT

function Parser:parseExpr()
    local left = self:parseTerm()
    
    while self:peek().type == "PLUS" or self:peek().type == "MINUS" do
        local op = self:consume()
        local right = self:parseTerm()
        left = { type = "BinaryOp", op = op.value, left = left, right = right }
    end
    
    return left
end

function Parser:parseTerm()
    local left = self:parseFactor()
    
    while self:peek().type == "STAR" or self:peek().type == "SLASH" do
        local op = self:consume()
        local right = self:parseFactor()
        left = { type = "BinaryOp", op = op.value, left = left, right = right }
    end
    
    return left
end

function Parser:parseFactor()
    local token = self:peek()
    
    if token.type == "INT" or token.type == "FLOAT" then
        self:consume()
        return { type = "Number", value = tonumber(token.value) }
    
    elseif token.type == "IDENT" then
        self:consume()
        return { type = "Identifier", name = token.value }
    
    elseif token.type == "LPAREN" then
        self:consume("LPAREN")
        local expr = self:parseExpr()
        self:consume("RPAREN")
        return expr
    
    elseif token.type == "MINUS" then
        self:consume()
        local operand = self:parseFactor()
        return { type = "UnaryOp", op = "-", operand = operand }
    
    else
        error(string.format("Unexpected token: %s '%s'", token.type, token.value))
    end
end

-- AST Evaluator
local function evaluate(node, env)
    env = env or {}
    
    if node.type == "Number" then
        return node.value
    
    elseif node.type == "Identifier" then
        if env[node.name] == nil then
            error("Undefined variable: " .. node.name)
        end
        return env[node.name]
    
    elseif node.type == "UnaryOp" then
        local operand = evaluate(node.operand, env)
        if node.op == "-" then return -operand end
    
    elseif node.type == "BinaryOp" then
        local left = evaluate(node.left, env)
        local right = evaluate(node.right, env)
        
        if node.op == "+" then return left + right
        elseif node.op == "-" then return left - right
        elseif node.op == "*" then return left * right
        elseif node.op == "/" then
            if right == 0 then error("Division by zero") end
            return left / right
        end
    end
    
    error("Unknown node type: " .. tostring(node.type))
end

-- Tokenizer for the parser
local function tokenize(input)
    local tokens = {}
    local pos = 1
    
    local patterns = {
        { "%d+%.%d+", "FLOAT" },
        { "%d+", "INT" },
        { "[a-zA-Z_][a-zA-Z0-9_]*", "IDENT" },
        { "%+", "PLUS" },
        { "%-", "MINUS" },
        { "%*", "STAR" },
        { "/", "SLASH" },
        { "%(", "LPAREN" },
        { "%)", "RPAREN" },
    }
    
    while pos <= #input do
        local ws = input:match("^%s+", pos)
        if ws then pos = pos + #ws end
        if pos > #input then break end
        
        for _, rule in ipairs(patterns) do
            local match = input:match("^" .. rule[1], pos)
            if match then
                table.insert(tokens, { type = rule[2], value = match, pos = pos })
                pos = pos + #match
                break
            end
        end
    end
    
    table.insert(tokens, { type = "EOF", value = "", pos = pos })
    return tokens
end

-- Test parser
local expressions = {
    "2 + 3",
    "10 * (2 + 3)",
    "100 / (5 * 4)",
    "x + y * 2"
}

local env = { x = 10, y = 5 }

for _, expr in ipairs(expressions) do
    local tokens = tokenize(expr)
    local parser = Parser.new(tokens)
    local ast = parser:parseExpr()
    local result = evaluate(ast, env)
    print(string.format("  %s = %g", expr, result))
end
```

---

## 12. AST Construction และ Visitor Pattern

```lua
-- AST Construction สำหรับ custom language
-- Visitor Pattern สำหรับ traverse AST

-- AST Node types
local AST = {}

function AST.program(body)
    return { type = "Program", body = body }
end

function AST.assign(name, value)
    return { type = "Assignment", name = name, value = value }
end

function AST.if_(condition, then_, else_)
    return { type = "If", condition = condition, then_ = then_, else_ = else_ }
end

function AST.while_(condition, body)
    return { type = "While", condition = condition, body = body }
end

function AST.func(name, params, body)
    return { type = "Function", name = name, params = params, body = body }
end

function AST.call(name, args)
    return { type = "Call", name = name, args = args }
end

function AST.number(value)
    return { type = "Number", value = value }
end

function AST.string_(value)
    return { type = "String", value = value }
end

function AST.ident(name)
    return { type = "Identifier", name = name }
end

function AST.binop(op, left, right)
    return { type = "BinaryOp", op = op, left = left, right = right }
end

function AST.return_(value)
    return { type = "Return", value = value }
end

-- Visitor Pattern
local Visitor = {}
Visitor.__index = Visitor

function Visitor.new(handlers)
    return setmetatable({ handlers = handlers }, Visitor)
end

function Visitor:visit(node)
    if not node then return end
    
    local handler = self.handlers[node.type]
    if handler then
        return handler(self, node)
    else
        -- Default: visit children
        self:visitChildren(node)
    end
end

function Visitor:visitChildren(node)
    for _, child in pairs(node) do
        if type(child) == "table" and child.type then
            self:visit(child)
        elseif type(child) == "table" then
            for _, item in ipairs(child) do
                if type(item) == "table" and item.type then
                    self:visit(item)
                end
            end
        end
    end
end

-- Code Generator Visitor
local CodeGenerator = Visitor.new({
    Program = function(v, node)
        local parts = {}
        for _, stmt in ipairs(node.body) do
            table.insert(parts, v:visit(stmt))
        end
        return table.concat(parts, "\n")
    end,
    
    Assignment = function(v, node)
        return string.format("local %s = %s", node.name, v:visit(node.value))
    end,
    
    If = function(v, node)
        local cond = v:visit(node.condition)
        local then_ = v:visit(node.then_)
        local result = string.format("if %s then\n  %s", cond, then_)
        if node.else_ then
            result = result .. "\nelse\n  " .. v:visit(node.else_)
        end
        return result .. "\nend"
    end,
    
    Function = function(v, node)
        local params = table.concat(node.params, ", ")
        local body = v:visit(node.body)
        return string.format("function %s(%s)\n%s\nend", node.name, params, body)
    end,
    
    Call = function(v, node)
        local args = {}
        for _, arg in ipairs(node.args) do
            table.insert(args, v:visit(arg))
        end
        return string.format("%s(%s)", node.name, table.concat(args, ", "))
    end,
    
    BinaryOp = function(v, node)
        return string.format("(%s %s %s)", v:visit(node.left), node.op, v:visit(node.right))
    end,
    
    Number = function(v, node)
        return tostring(node.value)
    end,
    
    String = function(v, node)
        return '"' .. node.value .. '"'
    end,
    
    Identifier = function(v, node)
        return node.name
    end,
    
    Return = function(v, node)
        return "return " .. (node.value and v:visit(node.value) or "")
    end
})

-- Build an AST
local ast = AST.program({
    AST.assign("x", AST.number(10)),
    AST.assign("y", AST.number(20)),
    AST.assign("result", AST.binop("+", AST.ident("x"), AST.ident("y"))),
    AST.if_(
        AST.binop(">", AST.ident("result"), AST.number(25)),
        AST.call("print", { AST.string_("Large result") }),
        AST.call("print", { AST.string_("Small result") })
    )
})

print("Generated Lua code:")
print(CodeGenerator:visit(ast))
```

---

## 13. DSL สำหรับ Animation

```lua
-- Animation DSL - กำหนด animations แบบ declarative
local AnimationDSL = {}

function AnimationDSL.animate(target, fn)
    local animation = {
        target = target,
        keyframes = {},
        duration = 1000,
        easing = "linear",
        loop = false,
        callbacks = {}
    }
    
    local dsl = {
        from = function(props)
            animation.from = props
            return dsl
        end,
        to = function(props)
            animation.to = props
            return dsl
        end,
        keyframe = function(time, props)
            table.insert(animation.keyframes, { time = time, props = props })
            return dsl
        end,
        duration = function(ms)
            animation.duration = ms
            return dsl
        end,
        easing = function(type_)
            animation.easing = type_
            return dsl
        end,
        loop = function(times)
            animation.loop = times or true
            return dsl
        end,
        onStart = function(fn)
            animation.callbacks.start = fn
            return dsl
        end,
        onEnd = function(fn)
            animation.callbacks.end_ = fn
            return dsl
        end,
        onUpdate = function(fn)
            animation.callbacks.update = fn
            return dsl
        end,
        build = function()
            return animation
        end
    }
    
    if fn then fn(dsl) end
    return dsl
end

-- Easing functions
local Easing = {
    linear = function(t) return t end,
    easeIn = function(t) return t * t end,
    easeOut = function(t) return t * (2 - t) end,
    easeInOut = function(t)
        if t < 0.5 then return 2 * t * t
        else return -1 + (4 - 2 * t) * t
        end
    end,
    bounce = function(t)
        if t < 1/2.75 then
            return 7.5625 * t * t
        elseif t < 2/2.75 then
            t = t - 1.5/2.75
            return 7.5625 * t * t + 0.75
        elseif t < 2.5/2.75 then
            t = t - 2.25/2.75
            return 7.5625 * t * t + 0.9375
        else
            t = t - 2.625/2.75
            return 7.5625 * t * t + 0.984375
        end
    end
}

-- Simulate animation player
local function playAnimation(animDef)
    local anim = animDef.build()
    local steps = 10
    
    print(string.format("Playing animation for: %s", tostring(anim.target)))
    
    if anim.callbacks.start then
        anim.callbacks.start()
    end
    
    local easingFn = Easing[anim.easing] or Easing.linear
    
    for i = 0, steps do
        local t = i / steps
        local easedT = easingFn(t)
        
        if anim.from and anim.to then
            local props = {}
            for k, fromV in pairs(anim.from) do
                local toV = anim.to[k] or fromV
                if type(fromV) == "number" then
                    props[k] = fromV + (toV - fromV) * easedT
                end
            end
            
            if anim.callbacks.update then
                anim.callbacks.update(props, easedT)
            end
        end
    end
    
    if anim.callbacks.end_ then
        anim.callbacks.end_()
    end
    
    print("Animation complete")
end

-- ใช้งาน Animation DSL
local fadeIn = AnimationDSL.animate("element#hero")
    .from({ opacity = 0, y = -50 })
    .to({ opacity = 1, y = 0 })
    .duration(500)
    .easing("easeOut")
    .onStart(function()
        print("  Animation started")
    end)
    .onUpdate(function(props, t)
        if math.floor(t * 10) % 3 == 0 then
            print(string.format("  opacity=%.2f y=%.1f", props.opacity, props.y))
        end
    end)
    .onEnd(function()
        print("  Animation ended")
    end)

playAnimation(fadeIn)
```

---

## 14. Pipeline DSL

```lua
-- Pipeline DSL - data processing pipeline
local Pipeline = {}
Pipeline.__index = Pipeline

function Pipeline.create()
    return setmetatable({ _stages = {} }, Pipeline)
end

function Pipeline:pipe(name, fn)
    table.insert(self._stages, { name = name, fn = fn, type = "transform" })
    return self
end

function Pipeline:filter(name, predicate)
    table.insert(self._stages, { name = name, fn = predicate, type = "filter" })
    return self
end

function Pipeline:map(name, fn)
    table.insert(self._stages, { name = name, fn = fn, type = "map" })
    return self
end

function Pipeline:reduce(name, fn, initial)
    table.insert(self._stages, { name = name, fn = fn, initial = initial, type = "reduce" })
    return self
end

function Pipeline:tap(fn)
    table.insert(self._stages, { 
        name = "tap", 
        fn = function(data) fn(data); return data end, 
        type = "transform" 
    })
    return self
end

function Pipeline:execute(input)
    local data = input
    
    for _, stage in ipairs(self._stages) do
        print(string.format("  [Pipeline:%s] processing", stage.name))
        
        if stage.type == "transform" then
            data = stage.fn(data)
        
        elseif stage.type == "filter" then
            if type(data) == "table" then
                local filtered = {}
                for _, item in ipairs(data) do
                    if stage.fn(item) then
                        table.insert(filtered, item)
                    end
                end
                data = filtered
            end
        
        elseif stage.type == "map" then
            if type(data) == "table" then
                local mapped = {}
                for i, item in ipairs(data) do
                    mapped[i] = stage.fn(item)
                end
                data = mapped
            end
        
        elseif stage.type == "reduce" then
            if type(data) == "table" then
                local acc = stage.initial
                for _, item in ipairs(data) do
                    acc = stage.fn(acc, item)
                end
                data = acc
            end
        end
    end
    
    return data
end

-- ใช้งาน Pipeline DSL
local dataPipeline = Pipeline.create()
    :pipe("fetch", function(input)
        -- Simulate fetching data
        return {
            { id = 1, name = "Alice", age = 25, salary = 50000 },
            { id = 2, name = "Bob", age = 17, salary = 30000 },
            { id = 3, name = "Charlie", age = 30, salary = 70000 },
            { id = 4, name = "Diana", age = 22, salary = 45000 },
            { id = 5, name = "Eve", age = 15, salary = 25000 }
        }
    end)
    :filter("adults_only", function(person)
        return person.age >= 18
    end)
    :map("enhance", function(person)
        return {
            id = person.id,
            name = person.name,
            age = person.age,
            salary = person.salary,
            taxBracket = person.salary > 60000 and "high" or "normal"
        }
    end)
    :pipe("sort", function(people)
        table.sort(people, function(a, b) return a.salary > b.salary end)
        return people
    end)
    :tap(function(data)
        print("  After pipeline, count:", #data)
    end)
    :reduce("stats", function(acc, person)
        acc.totalSalary = acc.totalSalary + person.salary
        acc.count = acc.count + 1
        table.insert(acc.people, person.name)
        return acc
    end, { totalSalary = 0, count = 0, people = {} })

print("Running data pipeline:")
local result = dataPipeline:execute(nil)

print("\nResults:")
print("Count:", result.count)
print("Total salary:", result.totalSalary)
print("Average:", math.floor(result.totalSalary / result.count))
print("People:", table.concat(result.people, ", "))
```

---

## 15. Mini Language Interpreter

```lua
-- Mini scripting language interpreter
-- Language: TINY
-- Syntax:
--   let x = 5
--   print x
--   if x > 3 then print "big" end
--   while x > 0 do let x = x - 1 done

local TinyInterpreter = {}
TinyInterpreter.__index = TinyInterpreter

function TinyInterpreter.new()
    return setmetatable({
        env = {},
        output = {}
    }, TinyInterpreter)
end

-- Simple tokenizer
function TinyInterpreter:tokenize(input)
    local tokens = {}
    local patterns = {
        { "^%d+%.%d+", "FLOAT" },
        { "^%d+", "INT" },
        { '^"[^"]*"', "STRING" },
        { "^let%s", "LET" },
        { "^print%s", "PRINT" },
        { "^if%s", "IF" },
        { "^then%s", "THEN" },
        { "^else%s", "ELSE" },
        { "^end%s?", "END" },
        { "^while%s", "WHILE" },
        { "^do%s", "DO" },
        { "^done%s?", "DONE" },
        { "^==", "EQ" },
        { "^!=", "NEQ" },
        { "^<=", "LTE" },
        { "^>=", "GTE" },
        { "^<", "LT" },
        { "^>", "GT" },
        { "^=", "ASSIGN" },
        { "^%+", "PLUS" },
        { "^%-", "MINUS" },
        { "^%*", "STAR" },
        { "^/", "SLASH" },
        { "^[a-zA-Z_][a-zA-Z0-9_]*", "IDENT" },
        { "^%s+", "WS" },
        { "^\n", "NL" }
    }
    
    local pos = 1
    while pos <= #input do
        local matched = false
        for _, rule in ipairs(patterns) do
            local match = input:match(rule[1], pos)
            if match then
                if rule[2] ~= "WS" then
                    table.insert(tokens, { type = rule[2], value = match:gsub("%s+$", "") })
                end
                pos = pos + #match
                matched = true
                break
            end
        end
        if not matched then
            error("Unexpected char at " .. pos .. ": " .. input:sub(pos, pos))
        end
    end
    
    table.insert(tokens, { type = "EOF", value = "" })
    return tokens
end

-- Run TINY program
function TinyInterpreter:run(code)
    local tokens = self:tokenize(code .. " ")
    local pos = 1
    
    local function peek() return tokens[pos] end
    local function next_() 
        local t = tokens[pos]
        pos = pos + 1
        return t
    end
    local function expect(type_)
        local t = next_()
        if t.type ~= type_ then
            error("Expected " .. type_ .. " got " .. t.type)
        end
        return t
    end
    
    local function parseExpr()
        local token = next_()
        local left
        
        if token.type == "INT" then
            left = tonumber(token.value)
        elseif token.type == "FLOAT" then
            left = tonumber(token.value)
        elseif token.type == "STRING" then
            left = token.value:sub(2, -2)
        elseif token.type == "IDENT" then
            left = self.env[token.value]
            if left == nil then
                error("Undefined variable: " .. token.value)
            end
        else
            error("Unexpected token in expr: " .. token.type)
        end
        
        -- Check for operator
        local op = peek()
        if op and (op.type == "PLUS" or op.type == "MINUS" or 
                   op.type == "STAR" or op.type == "SLASH" or
                   op.type == "GT" or op.type == "LT" or
                   op.type == "GTE" or op.type == "LTE" or
                   op.type == "EQ" or op.type == "NEQ") then
            next_()
            local right = parseExpr()
            if op.type == "PLUS" then return left + right
            elseif op.type == "MINUS" then return left - right
            elseif op.type == "STAR" then return left * right
            elseif op.type == "SLASH" then return left / right
            elseif op.type == "GT" then return left > right
            elseif op.type == "LT" then return left < right
            elseif op.type == "GTE" then return left >= right
            elseif op.type == "LTE" then return left <= right
            elseif op.type == "EQ" then return left == right
            elseif op.type == "NEQ" then return left ~= right
            end
        end
        
        return left
    end
    
    local function runStmts(stopAt)
        while peek().type ~= "EOF" and peek().type ~= stopAt do
            local token = peek()
            
            if token.type == "LET" then
                next_()
                local name = expect("IDENT").value
                expect("ASSIGN")
                local value = parseExpr()
                self.env[name] = value
                
            elseif token.type == "PRINT" then
                next_()
                local value = parseExpr()
                table.insert(self.output, tostring(value))
                print("[TINY] " .. tostring(value))
                
            elseif token.type == "IF" then
                next_()
                local cond = parseExpr()
                expect("THEN")
                if cond then
                    runStmts("END")
                    if peek().type == "ELSE" then
                        next_()
                        -- Skip else branch
                        local depth = 0
                        while peek().type ~= "EOF" do
                            if peek().type == "END" then
                                if depth == 0 then break end
                                depth = depth - 1
                            end
                            next_()
                        end
                    end
                else
                    -- Skip then branch
                    local depth = 0
                    while peek().type ~= "EOF" do
                        if peek().type == "ELSE" and depth == 0 then
                            next_()
                            runStmts("END")
                            break
                        elseif peek().type == "END" then
                            if depth == 0 then break end
                            depth = depth - 1
                        end
                        next_()
                    end
                end
                if peek().type == "END" then next_() end
                
            elseif token.type == "NL" or token.type == "WS" then
                next_()
                
            else
                break
            end
        end
    end
    
    runStmts("EOF")
    return self.output
end

-- Test TINY interpreter
local interpreter = TinyInterpreter.new()

local program = [[
let x = 10
let y = 20
let sum = x + y
print sum
if sum > 25 then
print "Sum is large"
else
print "Sum is small"
end
let name = "Lua"
print name
]]

interpreter:run(program)
```

---

## สรุป

บทนี้ครอบคลุม:

1. **Internal vs External DSL** - ความแตกต่างและเมื่อใช้อะไร
2. **Method Chaining DSL** - Fluent interface pattern
3. **Builder Pattern DSL** - HTTP Request builder
4. **HTML Generator DSL** - Generate HTML ด้วย Lua
5. **SQL Builder DSL** - Type-safe query builder
6. **Test Framework DSL** - describe/it/expect
7. **Config DSL** - Application configuration
8. **Workflow DSL** - Multi-step process definition
9. **State Machine DSL** - Declarative state transitions
10. **Tokenizer** - Lexical analysis
11. **Recursive Descent Parser** - Expression parsing
12. **AST + Visitor Pattern** - Code generation from AST
13. **Animation DSL** - Declarative animations
14. **Pipeline DSL** - Data processing pipelines
15. **Mini Language Interpreter** - Complete toy language

DSL ช่วยให้ code อ่านง่าย, expressive, และ domain-expert สามารถเข้าใจได้โดยไม่ต้องรู้ implementation details
