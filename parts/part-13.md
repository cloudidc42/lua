# บทที่ 13: การจัดการข้อผิดพลาด (Error Handling)

## สารบัญ
1. [บทนำ Error Handling](#บทนำ)
2. [Runtime Errors](#runtime-errors)
3. [error() function](#error-function)
4. [pcall() Protected Call](#pcall-protected-call)
5. [xpcall() กับ Message Handler](#xpcall-กับ-message-handler)
6. [Error Objects](#error-objects)
7. [Stack Traceback](#stack-traceback)
8. [Custom Error Types](#custom-error-types)
9. [Error Propagation](#error-propagation)
10. [assert()](#assert)
11. [Rethrowing Errors](#rethrowing-errors)
12. [Error ใน Coroutines](#error-ใน-coroutines)
13. [Defensive Programming](#defensive-programming)
14. [Input Validation Patterns](#input-validation-patterns)
15. [Contracts Pattern](#contracts-pattern)
16. [Result Type Pattern](#result-type-pattern)
17. [Error Wrapping](#error-wrapping)
18. [HTTP-style Error Codes](#http-style-error-codes)
19. [Logging Errors](#logging-errors)
20. [Production Error Handling](#production-error-handling)

---

## บทนำ

Error handling ที่ดีเป็นหัวใจของ production-ready software ใน Lua มีกลไก error handling หลักๆ คือ `pcall`, `xpcall`, `error`, และ `assert`

```lua
-- ตัวอย่างที่ 1: ประเภทของ errors ใน Lua
print("=== Types of Errors ===")
print("1. Syntax errors  - ตรวจพบตอน compile")
print("2. Runtime errors - เกิดขึ้นขณะรัน")
print("3. Logic errors   - โปรแกรมทำงานผิด (ไม่มี error message)")
print("")
print("Runtime error sources:")
print("  - Index nil value: local t = nil; t.x")
print("  - Call non-function: local f = 1; f()")
print("  - Stack overflow: infinite recursion")
print("  - Memory error: out of memory")
print("  - error() function: ที่เราเรียกเอง")
```

```lua
-- ตัวอย่างที่ 2: Runtime errors พื้นฐาน
-- ถ้าไม่มี pcall จะทำให้โปรแกรมหยุดทำงาน

-- 1. Attempt to index a nil value
local ok, err = pcall(function()
    local t = nil
    return t.field  -- ERROR!
end)
print("Index nil:", ok, err)

-- 2. Attempt to call a nil value
ok, err = pcall(function()
    local f = nil
    f()  -- ERROR!
end)
print("Call nil:", ok, err)

-- 3. Attempt to perform arithmetic on string
ok, err = pcall(function()
    return "hello" + 1  -- ERROR!
end)
print("Arithmetic on string:", ok, err)

-- 4. Stack overflow
ok, err = pcall(function()
    local function recurse() return recurse() end
    recurse()
end)
print("Stack overflow:", ok, err and err:sub(1, 50))
```

---

## Runtime Errors

```lua
-- ตัวอย่างที่ 3: Common runtime errors และวิธีป้องกัน
-- nil indexing
local function safe_index(t, key, default)
    if type(t) ~= "table" then
        return default
    end
    local v = t[key]
    return v ~= nil and v or default
end

local user = nil
print("Safe index:", safe_index(user, "name", "Unknown"))

user = {name = "Alice", age = 30}
print("Safe index:", safe_index(user, "name", "Unknown"))
print("Safe index:", safe_index(user, "email", "No email"))
```

```lua
-- ตัวอย่างที่ 4: Arithmetic errors
local function safe_divide(a, b)
    if type(a) ~= "number" or type(b) ~= "number" then
        return nil, "Arguments must be numbers"
    end
    if b == 0 then
        return nil, "Division by zero"
    end
    return a / b
end

local result, err = safe_divide(10, 2)
print("10/2:", result)

result, err = safe_divide(10, 0)
print("10/0:", result, err)

result, err = safe_divide("10", 2)
print('"10"/2:', result, err)

-- ใน Lua: 10/0 = inf (ไม่ใช่ error)
print("Direct 10/0 =", 10/0)   -- math.huge
print("Direct 0/0 =", 0/0)     -- -nan
print("Is nan:", 0/0 ~= 0/0)   -- nan != nan
```

```lua
-- ตัวอย่างที่ 5: Type checking เพื่อป้องกัน runtime errors
local function type_check(value, expected_type, name)
    if type(value) ~= expected_type then
        error(string.format(
            "bad argument '%s' (expected %s, got %s)",
            name, expected_type, type(value)
        ), 2)  -- level 2 = ชี้ไปที่ caller
    end
    return value
end

local function greet(name)
    type_check(name, "string", "name")
    return "Hello, " .. name
end

-- ทดสอบ
local ok, err = pcall(greet, "Alice")
print("Valid:", ok, ok and greet("Alice") or err)

ok, err = pcall(greet, 42)
print("Invalid:", ok, err)
```

---

## error() function

```lua
-- ตัวอย่างที่ 6: error() กับ level
-- error(message, level)
-- level 0 = ไม่เพิ่ม position info
-- level 1 = ชี้ไปที่บรรทัดที่เรียก error() (default)
-- level 2 = ชี้ไปที่ caller ของ function ที่เรียก error()

local function check_positive(n, name)
    if n <= 0 then
        error(string.format("'%s' must be positive, got %d", name, n), 2)
    end
    return n
end

local function calculate_area(width, height)
    check_positive(width, "width")
    check_positive(height, "height")
    return width * height
end

-- ทดสอบ
local ok, err = pcall(calculate_area, -5, 10)
print("Negative width error:")
print(err)  -- ชี้ไปที่ calculate_area (caller ของ check_positive)

ok, err = pcall(function()
    error("simple error message")  -- level 1
end)
print("\nLevel 1:", err)

ok, err = pcall(function()
    error("no position info", 0)  -- level 0
end)
print("Level 0:", err)
```

```lua
-- ตัวอย่างที่ 7: error() กับ message types
-- error() รับ value ใดก็ได้ ไม่ใช่แค่ string

-- error กับ string
local ok, err = pcall(function()
    error("something went wrong")
end)
print("String error:", type(err), err)

-- error กับ table
ok, err = pcall(function()
    error({code = 404, message = "Not Found"})
end)
print("Table error:", type(err), err.code, err.message)

-- error กับ number
ok, err = pcall(function()
    error(42)
end)
print("Number error:", type(err), err)

-- error กับ nil (ไม่ค่อยใช้)
ok, err = pcall(function()
    error(nil)  -- ไม่ดี - ทำให้ยากในการ handle
end)
print("Nil error:", ok, err)
```

```lua
-- ตัวอย่างที่ 8: สร้าง error factory
local function make_error(code, message, data)
    return {
        code = code,
        message = message,
        data = data,
        timestamp = os.time(),
    }
end

local function throw(code, message, data)
    error(make_error(code, message, data), 2)
end

-- ทดสอบ
local ok, err = pcall(function()
    throw(400, "Bad Request", {field = "email", reason = "invalid format"})
end)

if not ok then
    if type(err) == "table" then
        print(string.format("Error %d: %s", err.code, err.message))
        if err.data then
            print("Field:", err.data.field)
            print("Reason:", err.data.reason)
        end
    else
        print("Error:", err)
    end
end
```

---

## pcall() Protected Call

```lua
-- ตัวอย่างที่ 9: pcall() พื้นฐาน
-- pcall(func, arg1, arg2, ...)
-- คืน: true + results ถ้าสำเร็จ
--       false + error message ถ้าล้มเหลว

local function divide(a, b)
    if b == 0 then error("division by zero") end
    return a / b
end

-- สำเร็จ
local ok, result = pcall(divide, 10, 2)
print("Success:", ok, result)

-- ล้มเหลว
ok, result = pcall(divide, 10, 0)
print("Failure:", ok, result)

-- ส่ง multiple return values
local function multi_return(x)
    return x, x*2, x*3
end

ok, a, b, c = pcall(multi_return, 5)
print("Multiple returns:", ok, a, b, c)
```

```lua
-- ตัวอย่างที่ 10: pcall กับ methods
local Calculator = {}
Calculator.__index = Calculator

function Calculator.new()
    return setmetatable({history = {}}, Calculator)
end

function Calculator:compute(op, a, b)
    local result
    if op == "+" then result = a + b
    elseif op == "-" then result = a - b
    elseif op == "*" then result = a * b
    elseif op == "/" then
        if b == 0 then error("division by zero", 2) end
        result = a / b
    elseif op == "sqrt" then
        if a < 0 then error("cannot sqrt negative number", 2) end
        result = math.sqrt(a)
    else
        error("unknown operator: " .. tostring(op), 2)
    end
    
    table.insert(self.history, {op=op, a=a, b=b, result=result})
    return result
end

local calc = Calculator.new()

local operations = {
    {"+", 10, 5}, {"-", 10, 3}, {"*", 4, 5},
    {"/", 10, 0}, {"sqrt", -4, nil}, {"^", 2, 3}
}

print("=== Calculator with pcall ===")
for _, op in ipairs(operations) do
    local ok, result = pcall(calc.compute, calc, op[1], op[2], op[3])
    if ok then
        print(string.format("  %s(%s,%s) = %s", op[1], op[2], op[3] or "nil", result))
    else
        print(string.format("  %s(%s,%s) ERROR: %s", op[1], op[2], op[3] or "nil", 
              result:match(": (.+)$") or result))
    end
end
```

```lua
-- ตัวอย่างที่ 11: pcall สำหรับ file operations
local function safe_read_file(filename)
    return pcall(function()
        local f, err = io.open(filename, "r")
        if not f then error(err) end
        local content = f:read("a")
        f:close()
        return content
    end)
end

local ok, content = safe_read_file("/etc/hostname")
if ok then
    print("Hostname:", content:match("(.-)%s*$"))
else
    print("Cannot read /etc/hostname:", content)
end

ok, content = safe_read_file("/nonexistent/file.txt")
print("Nonexistent file:", ok, content)
```

```lua
-- ตัวอย่างที่ 12: pcall chain - nested protected calls
local function step1(x)
    if x < 0 then error("negative input in step1") end
    return x * 2
end

local function step2(x)
    if x > 100 then error("too large in step2") end
    return x + 10
end

local function step3(x)
    if x % 2 ~= 0 then error("odd number in step3") end
    return x / 2
end

local function pipeline(x)
    local ok, v = pcall(step1, x)
    if not ok then return nil, "step1: " .. v end
    
    ok, v = pcall(step2, v)
    if not ok then return nil, "step2: " .. v end
    
    ok, v = pcall(step3, v)
    if not ok then return nil, "step3: " .. v end
    
    return v
end

print("=== Pipeline ===")
local tests = {5, -1, 50, 30, 3}
for _, x in ipairs(tests) do
    local result, err = pipeline(x)
    if result then
        print(string.format("  input=%d -> %d", x, result))
    else
        print(string.format("  input=%d -> ERROR: %s", x, err))
    end
end
```

---

## xpcall() กับ Message Handler

```lua
-- ตัวอย่างที่ 13: xpcall() พื้นฐาน
-- xpcall(func, handler, arg1, arg2, ...)
-- handler รับ error message และสามารถแก้ไขได้ก่อน return

local function error_handler(err)
    -- เพิ่ม stack traceback
    return {
        message = err,
        traceback = debug.traceback("", 2),
        time = os.date("%Y-%m-%d %H:%M:%S"),
    }
end

local function risky_function()
    local t = nil
    return t.field  -- error!
end

local ok, result = xpcall(risky_function, error_handler)
if not ok then
    print("=== xpcall Error ===")
    print("Message:", result.message)
    print("Time:", result.time)
    print("Traceback:")
    print(result.traceback)
end
```

```lua
-- ตัวอย่างที่ 14: xpcall กับ message handler ที่ซับซ้อน
local ErrorInfo = {}

local function create_error_handler(context)
    return function(err)
        local info = {
            error = err,
            context = context,
            traceback = debug.traceback("", 2),
            timestamp = os.time(),
            thread = tostring(coroutine.running()),
        }
        
        -- Log ทันที
        io.stderr:write(string.format(
            "[ERROR][%s] %s\n", 
            context, 
            type(err) == "string" and err or "error object"
        ))
        
        return info
    end
end

local function do_db_operation(query)
    if not query:match("^SELECT") then
        error("Only SELECT queries allowed: " .. query)
    end
    return {rows = 0}  -- simulate result
end

local ok, result = xpcall(
    do_db_operation,
    create_error_handler("database"),
    "DELETE FROM users"
)

if not ok then
    print("Operation failed:")
    print("  Error:", result.error)
    print("  Context:", result.context)
    print("  Time:", os.date("%c", result.timestamp))
end
```

```lua
-- ตัวอย่างที่ 15: Global error handler pattern
local GlobalErrorHandler = {
    handlers = {},
    default_handler = nil,
}

function GlobalErrorHandler.add(category, handler)
    GlobalErrorHandler.handlers[category] = handler
end

function GlobalErrorHandler.set_default(handler)
    GlobalErrorHandler.default_handler = handler
end

function GlobalErrorHandler.handle(err)
    if type(err) == "table" and err.category then
        local handler = GlobalErrorHandler.handlers[err.category]
        if handler then return handler(err) end
    end
    
    if GlobalErrorHandler.default_handler then
        return GlobalErrorHandler.default_handler(err)
    end
    
    -- Fallback
    print("Unhandled error:", tostring(err))
    return err
end

-- ลงทะเบียน handlers
GlobalErrorHandler.add("network", function(err)
    print(string.format("[NETWORK ERROR] Code %d: %s", err.code, err.message))
    -- อาจ retry หรือ fallback
    return err
end)

GlobalErrorHandler.add("validation", function(err)
    print(string.format("[VALIDATION] Field '%s': %s", err.field, err.message))
    return err
end)

GlobalErrorHandler.set_default(function(err)
    print("[ERROR]", tostring(err))
    return err
end)

-- ทดสอบ
local function protected_run(func, ...)
    return xpcall(func, GlobalErrorHandler.handle, ...)
end

protected_run(function()
    error({category = "network", code = 503, message = "Service Unavailable"})
end)

protected_run(function()
    error({category = "validation", field = "email", message = "Invalid format"})
end)

protected_run(function()
    error("plain error string")
end)
```

---

## Error Objects

```lua
-- ตัวอย่างที่ 16: สร้าง Error class
local Error = {}
Error.__index = Error

function Error.new(message, code, data)
    return setmetatable({
        message = message or "Unknown error",
        code = code or 0,
        data = data,
        traceback = debug.traceback("", 2),
        timestamp = os.time(),
    }, Error)
end

function Error:__tostring()
    return string.format("Error[%d]: %s", self.code, self.message)
end

function Error:throw()
    error(self, 2)
end

function Error:is_a(class)
    return getmetatable(self) == class
end

-- ทดสอบ
local ok, err = pcall(function()
    Error.new("File not found", 404, {path = "/tmp/missing.txt"}):throw()
end)

if not ok then
    if type(err) == "table" and getmetatable(err) == Error then
        print("Error:", tostring(err))
        print("Code:", err.code)
        print("Data:", err.data and err.data.path)
    else
        print("Raw error:", err)
    end
end
```

```lua
-- ตัวอย่างที่ 17: Error hierarchy
local BaseError = {}
BaseError.__index = BaseError

function BaseError.new(class, message, code)
    local instance = setmetatable({
        message = message,
        code = code or 0,
        class = class.__name or "Error",
    }, class)
    instance.__index = class
    return instance
end

function BaseError:__tostring()
    return string.format("%s(%d): %s", self.class, self.code, self.message)
end

function BaseError:is_instance(class)
    local mt = getmetatable(self)
    while mt do
        if mt == class then return true end
        mt = getmetatable(mt)
    end
    return false
end

-- สร้าง error types
local NetworkError = setmetatable({__name = "NetworkError"}, {__index = BaseError})
NetworkError.__index = NetworkError

local TimeoutError = setmetatable({__name = "TimeoutError"}, {__index = NetworkError})
TimeoutError.__index = TimeoutError

local ValidationError = setmetatable({__name = "ValidationError"}, {__index = BaseError})
ValidationError.__index = ValidationError

-- constructors
function NetworkError.new(msg, code)
    return BaseError.new(NetworkError, msg, code or 500)
end

function TimeoutError.new(timeout_seconds)
    local err = BaseError.new(TimeoutError, 
        "Request timed out after " .. timeout_seconds .. "s", 408)
    err.timeout = timeout_seconds
    return err
end

function ValidationError.new(field, msg)
    local err = BaseError.new(ValidationError, msg, 400)
    err.field = field
    return err
end

-- ทดสอบ hierarchy
local errors = {
    NetworkError.new("Connection refused", 503),
    TimeoutError.new(30),
    ValidationError.new("email", "Invalid email format"),
}

print("=== Error Hierarchy ===")
for _, err in ipairs(errors) do
    print(tostring(err))
    print("  is NetworkError:", err:is_instance(NetworkError))
    print("  is TimeoutError:", err:is_instance(TimeoutError))
    print("  is BaseError:", err:is_instance(BaseError))
end
```

---

## Stack Traceback

```lua
-- ตัวอย่างที่ 18: debug.traceback()
local function level_3()
    error("error at level 3")
end

local function level_2()
    level_3()
end

local function level_1()
    level_2()
end

-- ด้วย xpcall
local ok, err = xpcall(level_1, function(err)
    return err .. "\n" .. debug.traceback("", 2)
end)

print("=== Stack Traceback ===")
print("Error:", err)
```

```lua
-- ตัวอย่างที่ 19: Custom traceback formatter
local function format_traceback(err, level)
    level = level or 2
    local tb = debug.traceback("", level)
    
    -- Parse traceback
    local lines = {}
    for line in tb:gmatch("[^\n]+") do
        table.insert(lines, line)
    end
    
    local formatted = {
        "Error: " .. tostring(err),
        "Stack trace:",
    }
    
    for i = 2, #lines do
        local line = lines[i]:match("^%s*(.+)$")
        if line and line ~= "" then
            table.insert(formatted, "  " .. i-1 .. ". " .. line)
        end
    end
    
    return table.concat(formatted, "\n")
end

local function risky()
    local t = {}
    return t.missing.value  -- nested nil access
end

local ok, err = xpcall(risky, function(e)
    return format_traceback(e, 2)
end)

print("=== Formatted Traceback ===")
print(err)
```

```lua
-- ตัวอย่างที่ 20: debug.getinfo() สำหรับ error location
local function get_caller_info(level)
    level = level or 2
    local info = debug.getinfo(level, "Snl")
    if not info then return nil end
    
    return {
        source = info.source,
        short_src = info.short_src,
        line = info.currentline,
        name = info.name,
        what = info.what,
    }
end

local function check_input(value, msg)
    if value == nil then
        local caller = get_caller_info(2)
        local location = caller and 
            string.format("%s:%d", caller.short_src, caller.line) or "unknown"
        error(string.format("[%s] %s", location, msg), 3)
    end
    return value
end

local function process_user(user)
    check_input(user, "user cannot be nil")
    check_input(user.name, "user.name cannot be nil")
    return "Processing: " .. user.name
end

-- ทดสอบ
local ok, err = pcall(process_user, nil)
print("nil user:", err)

ok, err = pcall(process_user, {name = nil})
print("nil name:", err)

ok, result = pcall(process_user, {name = "Alice"})
print("valid:", result)
```

---

## Custom Error Types

```lua
-- ตัวอย่างที่ 21: Domain-specific errors
local AppError = {}
AppError.__index = AppError
AppError.__name = "AppError"

-- Error codes
AppError.codes = {
    -- HTTP-inspired
    BAD_REQUEST       = {code = 400, name = "Bad Request"},
    UNAUTHORIZED      = {code = 401, name = "Unauthorized"},
    FORBIDDEN         = {code = 403, name = "Forbidden"},
    NOT_FOUND         = {code = 404, name = "Not Found"},
    CONFLICT          = {code = 409, name = "Conflict"},
    INTERNAL_ERROR    = {code = 500, name = "Internal Error"},
    NOT_IMPLEMENTED   = {code = 501, name = "Not Implemented"},
    SERVICE_UNAVAIL   = {code = 503, name = "Service Unavailable"},
    -- Custom
    VALIDATION        = {code = 422, name = "Validation Error"},
    DATABASE          = {code = 510, name = "Database Error"},
    PERMISSION        = {code = 511, name = "Permission Error"},
}

function AppError.new(error_code, message, context)
    local ec = AppError.codes[error_code] or {code = 0, name = error_code}
    return setmetatable({
        error_code = error_code,
        code = ec.code,
        name = ec.name,
        message = message or ec.name,
        context = context,
        timestamp = os.time(),
    }, AppError)
end

function AppError:__tostring()
    return string.format("AppError[%d %s]: %s", 
           self.code, self.name, self.message)
end

function AppError:is(code)
    return self.error_code == code
end

function AppError:throw()
    error(self, 2)
end

-- ทดสอบ
local function find_user(id)
    if type(id) ~= "number" then
        AppError.new("BAD_REQUEST", "User ID must be a number", {id = id}):throw()
    end
    if id <= 0 then
        AppError.new("NOT_FOUND", "User not found", {id = id}):throw()
    end
    return {id = id, name = "User #" .. id}
end

local ids = {5, -1, "abc", 100}
print("=== Custom Error Types ===")
for _, id in ipairs(ids) do
    local ok, result = pcall(find_user, id)
    if ok then
        print(string.format("  ID %-5s -> Found: %s", tostring(id), result.name))
    else
        if type(result) == "table" and getmetatable(result) == AppError then
            print(string.format("  ID %-5s -> %s", tostring(id), tostring(result)))
        else
            print(string.format("  ID %-5s -> Raw error: %s", tostring(id), result))
        end
    end
end
```

---

## Error Propagation

```lua
-- ตัวอย่างที่ 22: Error propagation patterns
-- Pattern 1: Re-raise (ส่งต่อ error)
local function fetch_data(url)
    if not url:match("^https?://") then
        error("Invalid URL: " .. url)
    end
    return "data from " .. url
end

local function process_api(url)
    -- ไม่ catch error แค่ส่งต่อ
    local data = fetch_data(url)
    return data .. " (processed)"
end

local function run_job(url)
    local ok, result = pcall(process_api, url)
    if not ok then
        -- Log แล้ว re-raise
        io.stderr:write("Job failed: " .. tostring(result) .. "\n")
        error("Job failed: " .. tostring(result), 2)
    end
    return result
end

-- ทดสอบ
print("=== Error Propagation ===")
local ok, err = pcall(run_job, "ftp://invalid.com")
print("Error reached top:", ok, err and err:sub(1, 50))
```

```lua
-- ตัวอย่างที่ 23: Error wrapping
local function wrap_error(original_err, context)
    if type(original_err) == "table" and original_err.wrapped then
        -- ไม่ wrap ซ้ำ
        table.insert(original_err.context_chain, context)
        return original_err
    end
    
    return {
        wrapped = true,
        original = original_err,
        message = context .. ": " .. tostring(original_err),
        context_chain = {context},
    }
end

local function step_a()
    error("database connection failed")
end

local function step_b()
    local ok, err = pcall(step_a)
    if not ok then
        error(wrap_error(err, "step_b failed"), 2)
    end
end

local function step_c()
    local ok, err = pcall(step_b)
    if not ok then
        error(wrap_error(err, "step_c failed"), 2)
    end
end

local ok, err = pcall(step_c)
if not ok then
    print("=== Wrapped Error Chain ===")
    if type(err) == "table" and err.wrapped then
        print("Final message:", err.message)
        print("Context chain:")
        for i, ctx in ipairs(err.context_chain) do
            print(string.format("  %d. %s", i, ctx))
        end
        print("Original error:", tostring(err.original))
    else
        print("Error:", err)
    end
end
```

---

## assert()

```lua
-- ตัวอย่างที่ 24: assert() พื้นฐาน
-- assert(value, message)
-- ถ้า value เป็น false หรือ nil จะ error ด้วย message
-- ถ้า value เป็น truthy จะคืนค่า value (pass-through)

-- assert ป้องกัน nil
local function get_user(id)
    if id == 1 then return {name = "Alice", id = 1} end
    return nil
end

-- ใช้ assert เพื่อ ensure ไม่ nil
local ok, err = pcall(function()
    local user = assert(get_user(99), "User not found: " .. 99)
    print("User:", user.name)
end)
print("assert failed:", err)

-- ใช้ assert เป็น pass-through
local user = assert(get_user(1), "User 1 not found")
print("assert passed:", user.name)
```

```lua
-- ตัวอย่างที่ 25: assert กับ io.open
local function read_config(filename)
    -- assert ด้วย error message จาก io.open
    local file = assert(io.open(filename, "r"), "Cannot open config: " .. filename)
    local content = file:read("a")
    file:close()
    return content
end

-- สร้างไฟล์ก่อน
io.open("/tmp/test_assert.txt", "w"):write("test"):close()

local ok, result = pcall(read_config, "/tmp/test_assert.txt")
print("Valid file:", ok, result)

ok, result = pcall(read_config, "/tmp/missing.txt")
print("Missing file:", ok, result)
```

```lua
-- ตัวอย่างที่ 26: assert-style validation
local function validate_user_input(data)
    assert(type(data) == "table", "data must be a table")
    assert(type(data.name) == "string", "name must be a string")
    assert(#data.name >= 2, "name too short (min 2 chars)")
    assert(#data.name <= 100, "name too long (max 100 chars)")
    assert(type(data.age) == "number", "age must be a number")
    assert(data.age >= 0 and data.age <= 150, "age must be 0-150")
    assert(type(data.email) == "string", "email must be a string")
    assert(data.email:match("^[%w%.]+@[%w%.]+%.[%a]+$"), "invalid email format")
    return true
end

local test_inputs = {
    {name = "Alice", age = 25, email = "alice@example.com"},
    {name = "A", age = 25, email = "alice@example.com"},     -- name too short
    {name = "Bob", age = -5, email = "bob@example.com"},     -- invalid age
    {name = "Charlie", age = 30, email = "not-an-email"},    -- invalid email
    "not a table",
}

print("=== Assert Validation ===")
for _, input in ipairs(test_inputs) do
    local ok, err = pcall(validate_user_input, input)
    if ok then
        print("  VALID:", type(input) == "table" and input.name or input)
    else
        print("  INVALID:", err)
    end
end
```

```lua
-- ตัวอย่างที่ 27: assert vs explicit error
-- assert: สั้น แต่ level อาจไม่ถูกต้อง
-- error: ควบคุม level และ message ได้ดีกว่า

-- Custom assert ที่ส่ง error ที่ level ถูกต้อง
local function check(cond, msg, level)
    if not cond then
        error(msg or "assertion failed", (level or 1) + 1)
    end
    return cond
end

local function create_matrix(rows, cols)
    check(type(rows) == "number" and rows > 0, "rows must be positive", 2)
    check(type(cols) == "number" and cols > 0, "cols must be positive", 2)
    
    local matrix = {}
    for i = 1, rows do
        matrix[i] = {}
        for j = 1, cols do
            matrix[i][j] = 0
        end
    end
    return matrix
end

local ok, err = pcall(create_matrix, -3, 4)
print("Invalid matrix:", err)

local m = create_matrix(3, 3)
print("Valid matrix:", #m, "x", #m[1])
```

---

## Rethrowing Errors

```lua
-- ตัวอย่างที่ 28: Rethrow pattern
local function parse_number(s)
    local n = tonumber(s)
    if not n then
        error("Cannot parse as number: '" .. s .. "'", 2)
    end
    return n
end

local function process_value(s)
    local ok, result = pcall(parse_number, s)
    if not ok then
        -- เพิ่ม context แล้ว rethrow
        error("process_value failed: " .. result, 2)
    end
    return result * 2
end

-- ทดสอบ
local ok, err = pcall(process_value, "abc")
print("Rethrow:", err)
```

```lua
-- ตัวอย่างที่ 29: ตัดสินใจว่าจะ handle หรือ rethrow
local RETRYABLE_ERRORS = {
    "connection timeout",
    "connection refused",
    "service unavailable",
}

local function is_retryable(err)
    local msg = type(err) == "string" and err or tostring(err)
    for _, pattern in ipairs(RETRYABLE_ERRORS) do
        if msg:find(pattern, 1, true) then
            return true
        end
    end
    return false
end

local function retry_with_backoff(func, max_retries, base_delay)
    max_retries = max_retries or 3
    base_delay = base_delay or 1
    
    local last_err
    for attempt = 1, max_retries do
        local ok, result = pcall(func)
        if ok then
            return result
        end
        
        last_err = result
        
        if not is_retryable(result) then
            -- ไม่ retryable: rethrow ทันที
            error(result, 2)
        end
        
        print(string.format("Attempt %d/%d failed: %s", attempt, max_retries, result))
        
        if attempt < max_retries then
            local delay = base_delay * (2 ^ (attempt - 1))
            print(string.format("Retrying in %.1f seconds...", delay))
            -- os.execute("sleep " .. delay)  -- ใน production
        end
    end
    
    error("Max retries exceeded: " .. tostring(last_err), 2)
end

-- จำลอง flaky service
local attempt_count = 0
local function flaky_service()
    attempt_count = attempt_count + 1
    if attempt_count <= 2 then
        error("connection timeout")  -- fail สองครั้งแรก
    end
    return "success!"
end

print("=== Retry with Backoff ===")
local ok, result = pcall(retry_with_backoff, flaky_service, 3, 0.1)
if ok then
    print("Result:", result)
else
    print("Failed:", result)
end
```

---

## Error ใน Coroutines

```lua
-- ตัวอย่างที่ 30: Error propagation ใน coroutines
local function coroutine_with_error()
    local co = coroutine.create(function()
        coroutine.yield(1)
        coroutine.yield(2)
        error("error in coroutine!")  -- error ภายใน coroutine
        coroutine.yield(3)  -- ไม่ถึงที่นี่
    end)
    
    local results = {}
    while true do
        local ok, value = coroutine.resume(co)
        if not ok then
            print("Coroutine error:", value)
            break
        end
        if coroutine.status(co) == "dead" then break end
        table.insert(results, value)
    end
    
    return results
end

print("=== Coroutine Errors ===")
local values = coroutine_with_error()
print("Values before error:", table.concat(values, ", "))
```

```lua
-- ตัวอย่างที่ 31: Protected coroutine wrapper
local function protected_coroutine(func)
    local co = coroutine.create(func)
    
    return {
        resume = function(...)
            local ok, result = coroutine.resume(co, ...)
            if not ok then
                return false, result  -- error
            end
            if coroutine.status(co) == "dead" then
                return true, result, true  -- completed
            end
            return true, result, false  -- yielded
        end,
        status = function()
            return coroutine.status(co)
        end,
    }
end

local gen = protected_coroutine(function()
    for i = 1, 3 do
        coroutine.yield(i)
        if i == 2 then
            error("intentional error at i=2... just kidding")  
            -- ถ้า uncomment จะ error
        end
    end
    return "done"
end)

print("=== Protected Coroutine ===")
while true do
    local ok, value, done = gen.resume()
    if not ok then
        print("Error:", value)
        break
    end
    if done then
        print("Completed with:", value)
        break
    end
    print("Yielded:", value)
end
```

---

## Defensive Programming

```lua
-- ตัวอย่างที่ 32: Defensive programming patterns
local function safe_get(t, ...)
    local current = t
    for _, key in ipairs({...}) do
        if type(current) ~= "table" then return nil end
        current = current[key]
    end
    return current
end

-- ทดสอบ deep access
local config = {
    database = {
        primary = {
            host = "localhost",
            port = 5432,
        }
    }
}

print("=== Safe Get ===")
print(safe_get(config, "database", "primary", "host"))    -- localhost
print(safe_get(config, "database", "secondary", "host"))  -- nil (ไม่ error)
print(safe_get(config, "cache", "host"))                  -- nil (ไม่ error)
print(safe_get(nil, "any", "key"))                        -- nil (ไม่ error)
```

```lua
-- ตัวอย่างที่ 33: Null object pattern
local NullObject = {}
NullObject.__index = function(_, key)
    -- คืน function ที่คืน nil หรือ NullObject
    return function(...) return NullObject end
end
NullObject.__tostring = function() return "NullObject" end
NullObject.__concat = function(a, b)
    if a == NullObject then return tostring(b) end
    return tostring(a)
end
NullObject.__call = function() return NullObject end
NullObject.is_null = true

setmetatable(NullObject, {
    __index = NullObject,
    __tostring = function() return "NullObject" end,
})

-- ทดสอบ
local function find_user_or_null(id)
    if id == 1 then
        return {name = "Alice", email = "alice@example.com"}
    end
    return NullObject
end

local user = find_user_or_null(99)
print("=== Null Object Pattern ===")
print("is_null:", user == NullObject)
-- Safe to call methods without nil check
if user == NullObject then
    print("No user found")
end

user = find_user_or_null(1)
print("Found:", user.name)
```

```lua
-- ตัวอย่างที่ 34: Guard clauses
local function process_order(order)
    -- Guard clauses - fail fast
    if not order then
        return nil, "order is required"
    end
    if type(order) ~= "table" then
        return nil, "order must be a table"
    end
    if not order.items or #order.items == 0 then
        return nil, "order must have at least one item"
    end
    if not order.customer_id then
        return nil, "customer_id is required"
    end
    
    -- Main logic (รู้ว่า input valid แล้ว)
    local total = 0
    for _, item in ipairs(order.items) do
        if not item.price or not item.quantity then
            return nil, "each item must have price and quantity"
        end
        total = total + item.price * item.quantity
    end
    
    return {
        order_id = math.random(10000, 99999),
        customer_id = order.customer_id,
        total = total,
        status = "pending",
    }
end

-- ทดสอบ
local test_orders = {
    nil,
    "not a table",
    {customer_id = 1},  -- no items
    {customer_id = 1, items = {}},  -- empty items
    {items = {{price = 10, quantity = 2}}},  -- no customer
    {
        customer_id = 1,
        items = {{price = 10, quantity = 2}, {price = 5.5, quantity = 3}},
    },
}

print("=== Guard Clauses ===")
math.randomseed(42)
for _, order in ipairs(test_orders) do
    local result, err = process_order(order)
    if result then
        print(string.format("  OK: order #%d total=%.2f", result.order_id, result.total))
    else
        print(string.format("  ERROR: %s", err))
    end
end
```

---

## Input Validation Patterns

```lua
-- ตัวอย่างที่ 35: Validation framework
local Validator = {}
Validator.__index = Validator

function Validator.new()
    return setmetatable({errors = {}}, Validator)
end

function Validator:add_error(field, message)
    table.insert(self.errors, {field = field, message = message})
    return self
end

function Validator:is_valid()
    return #self.errors == 0
end

function Validator:first_error()
    return self.errors[1]
end

function Validator:all_errors()
    return self.errors
end

function Validator:__tostring()
    local msgs = {}
    for _, e in ipairs(self.errors) do
        table.insert(msgs, e.field .. ": " .. e.message)
    end
    return table.concat(msgs, "; ")
end

-- Validation rules
local function validate_create_user(data)
    local v = Validator.new()
    
    if type(data.username) ~= "string" then
        v:add_error("username", "must be a string")
    elseif #data.username < 3 then
        v:add_error("username", "minimum 3 characters")
    elseif #data.username > 30 then
        v:add_error("username", "maximum 30 characters")
    elseif not data.username:match("^[%a%d_]+$") then
        v:add_error("username", "only letters, numbers, underscores allowed")
    end
    
    if type(data.email) ~= "string" then
        v:add_error("email", "must be a string")
    elseif not data.email:match("^[%w%.%+%-]+@[%w%.%-]+%.[%a]+$") then
        v:add_error("email", "invalid email format")
    end
    
    if type(data.age) ~= "number" then
        v:add_error("age", "must be a number")
    elseif data.age < 13 then
        v:add_error("age", "must be at least 13")
    elseif data.age > 120 then
        v:add_error("age", "must be at most 120")
    end
    
    if data.password then
        if #data.password < 8 then
            v:add_error("password", "minimum 8 characters")
        end
        if not data.password:match("%u") then
            v:add_error("password", "must contain uppercase")
        end
        if not data.password:match("%d") then
            v:add_error("password", "must contain a digit")
        end
    else
        v:add_error("password", "required")
    end
    
    return v
end

-- ทดสอบ
local test_users = {
    {username = "alice", email = "alice@example.com", age = 25, password = "Secret1!"},
    {username = "ab", email = "bad-email", age = 10, password = "weak"},
    {username = "valid_user_123", email = "valid@test.org", age = 30, password = "Valid1Pass"},
}

print("=== Validation Framework ===")
for _, user in ipairs(test_users) do
    local v = validate_create_user(user)
    print(string.format("\nUser '%s':", user.username or "?"))
    if v:is_valid() then
        print("  VALID!")
    else
        for _, err in ipairs(v:all_errors()) do
            print(string.format("  - %s: %s", err.field, err.message))
        end
    end
end
```

---

## Contracts Pattern

```lua
-- ตัวอย่างที่ 36: Design by Contract
local Contract = {}

function Contract.require(condition, message)
    -- Precondition
    if not condition then
        error("Precondition violated: " .. (message or "condition failed"), 2)
    end
end

function Contract.ensure(condition, message)
    -- Postcondition
    if not condition then
        error("Postcondition violated: " .. (message or "condition failed"), 2)
    end
end

function Contract.invariant(obj, check_func, message)
    -- Invariant check
    if not check_func(obj) then
        error("Invariant violated: " .. (message or "invariant check failed"), 2)
    end
end

-- ตัวอย่าง: BankAccount กับ contracts
local BankAccount = {}
BankAccount.__index = BankAccount

function BankAccount.new(initial_balance)
    Contract.require(
        type(initial_balance) == "number" and initial_balance >= 0,
        "Initial balance must be non-negative number"
    )
    
    local self = setmetatable({
        balance = initial_balance,
        transaction_count = 0,
    }, BankAccount)
    
    return self
end

function BankAccount:deposit(amount)
    -- Preconditions
    Contract.require(type(amount) == "number", "Amount must be a number")
    Contract.require(amount > 0, "Deposit amount must be positive")
    
    local old_balance = self.balance
    self.balance = self.balance + amount
    self.transaction_count = self.transaction_count + 1
    
    -- Postconditions
    Contract.ensure(self.balance == old_balance + amount, "Balance not updated correctly")
    Contract.ensure(self.balance > old_balance, "Balance should increase after deposit")
    
    return self.balance
end

function BankAccount:withdraw(amount)
    -- Preconditions
    Contract.require(type(amount) == "number", "Amount must be a number")
    Contract.require(amount > 0, "Withdrawal amount must be positive")
    Contract.require(amount <= self.balance, 
        string.format("Insufficient funds: have %.2f, need %.2f", self.balance, amount))
    
    local old_balance = self.balance
    self.balance = self.balance - amount
    self.transaction_count = self.transaction_count + 1
    
    -- Postconditions
    Contract.ensure(self.balance >= 0, "Balance cannot be negative")
    Contract.ensure(self.balance == old_balance - amount, "Balance not updated correctly")
    
    return self.balance
end

-- ทดสอบ
print("=== Contracts Pattern ===")
local account = BankAccount.new(1000)
print("Initial balance:", account.balance)

local ok, result = pcall(function()
    account:deposit(500)
    print("After deposit 500:", account.balance)
    account:withdraw(200)
    print("After withdraw 200:", account.balance)
    account:withdraw(2000)  -- ไม่พอ!
end)

if not ok then
    print("Contract violated:", result:match(": (.+)$") or result)
end
print("Final balance:", account.balance)
print("Transactions:", account.transaction_count)
```

---

## Result Type Pattern

```lua
-- ตัวอย่างที่ 37: Result/Either pattern
local Result = {}
Result.__index = Result

function Result.ok(value)
    return setmetatable({
        _ok = true,
        _value = value,
    }, Result)
end

function Result.err(message, code)
    return setmetatable({
        _ok = false,
        _error = message,
        _code = code or 0,
    }, Result)
end

function Result:is_ok()
    return self._ok
end

function Result:is_err()
    return not self._ok
end

function Result:value()
    if not self._ok then
        error("Cannot get value of error result: " .. tostring(self._error), 2)
    end
    return self._value
end

function Result:error()
    return self._error
end

function Result:or_else(default)
    if self._ok then return self._value end
    return default
end

function Result:map(func)
    if not self._ok then return self end
    local ok, val = pcall(func, self._value)
    if ok then
        return Result.ok(val)
    else
        return Result.err(val)
    end
end

function Result:flat_map(func)
    if not self._ok then return self end
    local ok, val = pcall(func, self._value)
    if ok and type(val) == "table" and val._ok ~= nil then
        return val  -- Return the Result from func
    elseif ok then
        return Result.ok(val)
    else
        return Result.err(val)
    end
end

function Result:__tostring()
    if self._ok then
        return "Ok(" .. tostring(self._value) .. ")"
    else
        return "Err(" .. tostring(self._error) .. ")"
    end
end

-- ทดสอบ
local function parse_age(s)
    local n = tonumber(s)
    if not n then
        return Result.err("'" .. s .. "' is not a number")
    end
    if n < 0 or n > 150 then
        return Result.err("Age must be between 0 and 150")
    end
    return Result.ok(math.floor(n))
end

local function get_age_group(age)
    if age < 18 then return Result.ok("minor")
    elseif age < 65 then return Result.ok("adult")
    else return Result.ok("senior")
    end
end

print("=== Result Type Pattern ===")
local inputs = {"25", "abc", "-5", "30", "200", "17"}

for _, input in ipairs(inputs) do
    local result = parse_age(input)
        :flat_map(get_age_group)
    
    print(string.format("  %-5s -> %s", input, tostring(result)))
end

-- Method chaining
local computation = Result.ok(10)
    :map(function(x) return x * 2 end)
    :map(function(x) return x + 5 end)
    :map(function(x) return "Result: " .. x end)

print("\nChained computation:", tostring(computation))
```

```lua
-- ตัวอย่างที่ 38: Go-style error handling (return value, err)
local function divide(a, b)
    if b == 0 then
        return nil, "division by zero"
    end
    return a / b, nil
end

local function sqrt(n)
    if n < 0 then
        return nil, "cannot sqrt negative number"
    end
    return math.sqrt(n), nil
end

local function chain_operations(a, b)
    local result, err = divide(a, b)
    if err then return nil, "divide: " .. err end
    
    result, err = sqrt(result)
    if err then return nil, "sqrt: " .. err end
    
    return result, nil
end

print("=== Go-style Error Handling ===")
local tests = {{16, 4}, {-9, 3}, {100, 0}}
for _, t in ipairs(tests) do
    local val, err = chain_operations(t[1], t[2])
    if val then
        print(string.format("  chain(%d, %d) = %.4f", t[1], t[2], val))
    else
        print(string.format("  chain(%d, %d) = ERROR: %s", t[1], t[2], err))
    end
end
```

---

## Error Wrapping

```lua
-- ตัวอย่างที่ 39: Error context chain
local ContextError = {}
ContextError.__index = ContextError

function ContextError.wrap(err, context)
    if type(err) == "table" and err._is_context_error then
        table.insert(err._chain, context)
        return err
    end
    
    return setmetatable({
        _is_context_error = true,
        _original = err,
        _chain = {context},
        _message = context .. ": " .. tostring(err),
    }, ContextError)
end

function ContextError:unwrap()
    return self._original
end

function ContextError:chain()
    return self._chain
end

function ContextError:__tostring()
    return self._message
end

-- ตัวอย่างการใช้
local function read_user_from_db(id)
    if id > 100 then
        error("record not found: id=" .. id)
    end
    return {id = id, name = "User " .. id}
end

local function get_user_profile(id)
    local ok, result = pcall(read_user_from_db, id)
    if not ok then
        error(ContextError.wrap(result, "get_user_profile"), 2)
    end
    return result
end

local function handle_request(user_id)
    local ok, result = pcall(get_user_profile, user_id)
    if not ok then
        error(ContextError.wrap(result, "handle_request"), 2)
    end
    return result
end

print("=== Error Context Chain ===")
local ok, err = pcall(handle_request, 150)
if not ok then
    print("Error:", tostring(err))
    if type(err) == "table" and err._is_context_error then
        print("Context chain:")
        for i, ctx in ipairs(err:chain()) do
            print(string.format("  %d. %s", i, ctx))
        end
        print("Original:", tostring(err:unwrap()))
    end
end
```

---

## HTTP-style Error Codes

```lua
-- ตัวอย่างที่ 40: HTTP-style Error system
local HttpError = {}
HttpError.__index = HttpError

-- Standard HTTP status codes
local STATUS = {
    OK                    = 200,
    CREATED               = 201,
    BAD_REQUEST           = 400,
    UNAUTHORIZED          = 401,
    FORBIDDEN             = 403,
    NOT_FOUND             = 404,
    METHOD_NOT_ALLOWED    = 405,
    CONFLICT              = 409,
    UNPROCESSABLE_ENTITY  = 422,
    TOO_MANY_REQUESTS     = 429,
    INTERNAL_SERVER_ERROR = 500,
    NOT_IMPLEMENTED       = 501,
    SERVICE_UNAVAILABLE   = 503,
    GATEWAY_TIMEOUT       = 504,
}

local STATUS_MESSAGES = {
    [200] = "OK",
    [201] = "Created",
    [400] = "Bad Request",
    [401] = "Unauthorized",
    [403] = "Forbidden",
    [404] = "Not Found",
    [405] = "Method Not Allowed",
    [409] = "Conflict",
    [422] = "Unprocessable Entity",
    [429] = "Too Many Requests",
    [500] = "Internal Server Error",
    [501] = "Not Implemented",
    [503] = "Service Unavailable",
    [504] = "Gateway Timeout",
}

function HttpError.new(status_code, message, details)
    local status_msg = STATUS_MESSAGES[status_code] or "Unknown"
    return setmetatable({
        status = status_code,
        status_text = status_msg,
        message = message or status_msg,
        details = details,
        timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ"),
    }, HttpError)
end

function HttpError:__tostring()
    return string.format("HTTP %d %s: %s", 
           self.status, self.status_text, self.message)
end

function HttpError:is_client_error()
    return self.status >= 400 and self.status < 500
end

function HttpError:is_server_error()
    return self.status >= 500
end

function HttpError:to_response()
    return {
        error = {
            status = self.status,
            message = self.message,
            details = self.details,
            timestamp = self.timestamp,
        }
    }
end

-- Router simulation
local function simulate_api(method, path, body)
    if method ~= "GET" and method ~= "POST" and method ~= "DELETE" then
        error(HttpError.new(STATUS.METHOD_NOT_ALLOWED, 
              "Method " .. method .. " not allowed"))
    end
    
    -- Route matching
    local user_id = path:match("^/users/(%d+)$")
    if user_id then
        user_id = tonumber(user_id)
        if method == "GET" then
            if user_id > 100 then
                error(HttpError.new(STATUS.NOT_FOUND, 
                      "User not found", {user_id = user_id}))
            end
            return {status = 200, body = {id = user_id, name = "User " .. user_id}}
        end
    end
    
    if path == "/users" and method == "POST" then
        if not body or not body.name then
            error(HttpError.new(STATUS.BAD_REQUEST, 
                  "name is required", {missing_fields = {"name"}}))
        end
        return {status = 201, body = {id = math.random(1000), name = body.name}}
    end
    
    error(HttpError.new(STATUS.NOT_FOUND, "Route not found: " .. method .. " " .. path))
end

-- ทดสอบ
print("=== HTTP-style Errors ===")
local requests = {
    {"GET", "/users/50", nil},
    {"GET", "/users/200", nil},
    {"POST", "/users", {name = "Alice"}},
    {"POST", "/users", {}},
    {"DELETE", "/users/1", nil},
    {"PATCH", "/users/1", nil},
    {"GET", "/unknown", nil},
}

math.randomseed(42)
for _, req in ipairs(requests) do
    local ok, result = pcall(simulate_api, req[1], req[2], req[3])
    if ok then
        print(string.format("  %s %-20s -> %d: %s", 
              req[1], req[2], result.status, 
              type(result.body) == "table" and 
              (result.body.name and "name=" .. result.body.name or "id=" .. (result.body.id or "?")) 
              or "ok"))
    else
        if type(result) == "table" and getmetatable(result) == HttpError then
            print(string.format("  %s %-20s -> ERROR: %s", req[1], req[2], tostring(result)))
            if result.details then
                -- แสดง details
                for k, v in pairs(result.details) do
                    print(string.format("    %s: %s", k, tostring(v)))
                end
            end
        else
            print(string.format("  ERROR: %s", result))
        end
    end
end
```

---

## Logging Errors

```lua
-- ตัวอย่างที่ 41: Error Logger
local Logger = {}
Logger.__index = Logger

Logger.LEVELS = {
    DEBUG = 1,
    INFO  = 2,
    WARN  = 3,
    ERROR = 4,
    FATAL = 5,
}

Logger.LEVEL_NAMES = {[1]="DEBUG",[2]="INFO",[3]="WARN",[4]="ERROR",[5]="FATAL"}

function Logger.new(options)
    options = options or {}
    local self = setmetatable({
        level = options.level or Logger.LEVELS.INFO,
        outputs = options.outputs or {io.stdout},
        format = options.format or "[{level}][{time}] {message}",
        include_traceback = options.include_traceback or false,
    }, Logger)
    return self
end

function Logger:_write(level, message, context)
    if level < self.level then return end
    
    local formatted = self.format
        :gsub("{level}", self.LEVEL_NAMES[level] or "?")
        :gsub("{time}", os.date("%Y-%m-%d %H:%M:%S"))
        :gsub("{message}", tostring(message))
    
    for _, output in ipairs(self.outputs) do
        output:write(formatted .. "\n")
        if context then
            for k, v in pairs(context) do
                output:write(string.format("  %s: %s\n", k, tostring(v)))
            end
        end
        if self.include_traceback and level >= Logger.LEVELS.ERROR then
            output:write(debug.traceback("", 3) .. "\n")
        end
    end
end

function Logger:debug(msg, ctx) self:_write(Logger.LEVELS.DEBUG, msg, ctx) end
function Logger:info(msg, ctx)  self:_write(Logger.LEVELS.INFO, msg, ctx) end
function Logger:warn(msg, ctx)  self:_write(Logger.LEVELS.WARN, msg, ctx) end
function Logger:error(msg, ctx) self:_write(Logger.LEVELS.ERROR, msg, ctx) end
function Logger:fatal(msg, ctx) self:_write(Logger.LEVELS.FATAL, msg, ctx) end

function Logger:log_error(err, context)
    local msg, ctx
    if type(err) == "table" then
        msg = err.message or tostring(err)
        ctx = context or {}
        if err.code then ctx.code = err.code end
        if err.data then
            for k, v in pairs(err.data) do ctx[k] = v end
        end
    else
        msg = tostring(err)
        ctx = context
    end
    self:error(msg, ctx)
end

-- ทดสอบ
print("=== Error Logger ===")
local log = Logger.new({
    level = Logger.LEVELS.DEBUG,
    format = "[{level}] {time} - {message}",
})

log:info("Application started")
log:debug("Connecting to database...", {host = "localhost", port = 5432})

local ok, err = pcall(function()
    error({code = 500, message = "Database connection failed", 
           data = {host = "localhost", retry_count = 3}})
end)

if not ok then
    log:log_error(err, {operation = "startup"})
    log:fatal("Application cannot start")
end
```

---

## Production Error Handling

```lua
-- ตัวอย่างที่ 42: Production-ready error handler
local ProductionErrorHandler = {}
ProductionErrorHandler.__index = ProductionErrorHandler

function ProductionErrorHandler.new(config)
    config = config or {}
    return setmetatable({
        log_file = config.log_file or "/tmp/app_errors.log",
        max_errors = config.max_errors or 1000,
        error_count = 0,
        errors = {},
        alert_threshold = config.alert_threshold or 10,
        alert_callback = config.alert_callback,
    }, ProductionErrorHandler)
end

function ProductionErrorHandler:handle(err, context)
    self.error_count = self.error_count + 1
    
    local error_info = {
        id = self.error_count,
        timestamp = os.time(),
        message = type(err) == "string" and err or 
                  (type(err) == "table" and err.message) or tostring(err),
        error_obj = err,
        context = context,
    }
    
    -- เก็บใน memory (จำกัด)
    if #self.errors >= self.max_errors then
        table.remove(self.errors, 1)
    end
    table.insert(self.errors, error_info)
    
    -- Log ลงไฟล์
    self:_log_to_file(error_info)
    
    -- Alert ถ้าเกิน threshold
    if self.error_count % self.alert_threshold == 0 then
        self:_send_alert(error_info)
    end
    
    return error_info
end

function ProductionErrorHandler:_log_to_file(info)
    local f = io.open(self.log_file, "a")
    if f then
        f:write(string.format(
            "[%s][#%d] %s\n",
            os.date("%Y-%m-%d %H:%M:%S", info.timestamp),
            info.id,
            info.message
        ))
        if info.context then
            for k, v in pairs(info.context) do
                f:write(string.format("  %s=%s\n", k, tostring(v)))
            end
        end
        f:close()
    end
end

function ProductionErrorHandler:_send_alert(info)
    if self.alert_callback then
        self.alert_callback(info, self.error_count)
    else
        io.stderr:write(string.format(
            "[ALERT] %d errors occurred. Latest: %s\n",
            self.error_count, info.message
        ))
    end
end

function ProductionErrorHandler:stats()
    return {
        total = self.error_count,
        in_memory = #self.errors,
        recent = self.errors[#self.errors],
    }
end

-- ใช้งานใน production
local handler = ProductionErrorHandler.new({
    log_file = "/tmp/prod_errors.log",
    alert_threshold = 3,
    alert_callback = function(info, count)
        print(string.format("🚨 ALERT: %d errors! Latest: %s", count, info.message))
    end,
})

-- Wrap function ด้วย production handler
local function safe_execute(func, context, ...)
    local ok, result = xpcall(func, function(err)
        return handler:handle(err, context)
    end, ...)
    
    if not ok then
        return nil, result
    end
    return result
end

-- จำลอง production errors
print("=== Production Error Handling ===")
local operations = {
    function() return 10 / 2 end,
    function() error("Database timeout") end,
    function() return 20 / 4 end,
    function() error("Network unreachable") end,
    function() local t = nil return t.field end,
    function() return 30 / 6 end,
    function() error("Out of memory") end,
}

for i, op in ipairs(operations) do
    local result, err = safe_execute(op, {operation = "op_" .. i})
    if result then
        print(string.format("  op_%d: SUCCESS = %s", i, tostring(result)))
    else
        print(string.format("  op_%d: FAILED - %s", i, 
              type(err) == "table" and err.message or tostring(err)))
    end
end

local stats = handler:stats()
print(string.format("\nError stats: %d total, %d in memory", 
      stats.total, stats.in_memory))
```

```lua
-- ตัวอย่างที่ 43: Circuit Breaker pattern
local CircuitBreaker = {}
CircuitBreaker.__index = CircuitBreaker

CircuitBreaker.STATES = {
    CLOSED   = "closed",    -- ปกติ
    OPEN     = "open",      -- หยุดรับ requests
    HALF_OPEN = "half_open", -- ทดสอบว่าพร้อมแล้วหรือยัง
}

function CircuitBreaker.new(options)
    options = options or {}
    return setmetatable({
        state = CircuitBreaker.STATES.CLOSED,
        failure_count = 0,
        success_count = 0,
        failure_threshold = options.failure_threshold or 5,
        success_threshold = options.success_threshold or 2,
        timeout = options.timeout or 60,  -- seconds
        last_failure_time = 0,
        name = options.name or "circuit",
    }, CircuitBreaker)
end

function CircuitBreaker:call(func, ...)
    local now = os.time()
    
    if self.state == CircuitBreaker.STATES.OPEN then
        -- Check ว่าถึงเวลา half-open หรือยัง
        if now - self.last_failure_time >= self.timeout then
            self.state = CircuitBreaker.STATES.HALF_OPEN
            self.success_count = 0
            print(string.format("[CB:%s] -> HALF_OPEN", self.name))
        else
            return nil, "Circuit breaker is OPEN"
        end
    end
    
    local ok, result = pcall(func, ...)
    
    if ok then
        if self.state == CircuitBreaker.STATES.HALF_OPEN then
            self.success_count = self.success_count + 1
            if self.success_count >= self.success_threshold then
                self.state = CircuitBreaker.STATES.CLOSED
                self.failure_count = 0
                print(string.format("[CB:%s] -> CLOSED (recovered)", self.name))
            end
        else
            self.failure_count = 0  -- reset on success
        end
        return result
    else
        self.failure_count = self.failure_count + 1
        self.last_failure_time = now
        
        if self.failure_count >= self.failure_threshold then
            self.state = CircuitBreaker.STATES.OPEN
            print(string.format("[CB:%s] -> OPEN (failures=%d)", 
                  self.name, self.failure_count))
        end
        
        return nil, result
    end
end

function CircuitBreaker:status()
    return {
        state = self.state,
        failures = self.failure_count,
        last_failure = self.last_failure_time,
    }
end

-- ทดสอบ
print("=== Circuit Breaker ===")
local cb = CircuitBreaker.new({
    name = "api",
    failure_threshold = 3,
    success_threshold = 2,
    timeout = 1,  -- 1 second สำหรับทดสอบ
})

-- Service ที่จะ fail บางส่วน
local fail_count = 0
local function unreliable_service()
    fail_count = fail_count + 1
    if fail_count <= 4 then
        error("Service temporarily unavailable")
    end
    return "success response"
end

-- ทำ requests
for i = 1, 8 do
    local result, err = cb:call(unreliable_service)
    local status = cb:status()
    print(string.format("  Request %d: state=%-10s result=%s", 
          i, status.state, result or ("ERR: " .. tostring(err):sub(1, 30))))
    
    -- จำลองการรอ (เพื่อ trigger half-open)
    if i == 6 then
        fail_count = 10  -- service recovered
        -- จำลอง timeout ผ่านด้วยการ set last_failure_time
        cb.last_failure_time = os.time() - 2
    end
end
```

```lua
-- ตัวอย่างที่ 44: สรุป - Complete error handling system
print("\n=== Complete Error Handling Summary ===")
print([[
Lua Error Handling Patterns:

1. pcall(func, ...) 
   - Protected call, คืน ok, result/error
   - ใช้เมื่อต้องการ catch error ทั่วไป

2. xpcall(func, handler, ...)
   - Protected call + custom error handler
   - ใช้เมื่อต้องการ stack trace หรือ logging

3. error(msg, level)
   - Throw error
   - level 0: no location, 1: here, 2: caller

4. assert(cond, msg)
   - Error ถ้า condition false
   - ดีสำหรับ quick validation

5. Error objects (table)
   - เก็บข้อมูลเพิ่มเติมเช่น code, context
   - ใช้ type checking เพื่อ handle แยก

6. Result type pattern
   - return value, error
   - Explicit error handling ไม่ throw

7. Contract pattern
   - precondition/postcondition
   - สำหรับ defensive programming

8. Circuit Breaker
   - ป้องกัน cascade failures
   - Production resilience pattern
]])
```

```lua
-- ตัวอย่างที่ 45: Error handling best practices
-- เป็นตัวอย่างสุดท้ายที่รวบรวม patterns ทั้งหมด

local function robust_operation(input)
    -- 1. Validate input ก่อน
    if type(input) ~= "table" then
        return nil, "input must be a table"
    end
    
    if not input.value then
        return nil, "input.value is required"
    end
    
    -- 2. ทำงานหลักใน pcall
    local ok, result = pcall(function()
        -- 3. Assert preconditions
        assert(type(input.value) == "number", "value must be a number")
        assert(input.value >= 0, "value must be non-negative")
        
        -- 4. ทำงานจริง
        local processed = math.sqrt(input.value) * 100
        
        -- 5. Assert postconditions
        assert(processed >= 0, "result must be non-negative")
        
        return math.floor(processed) / 100
    end)
    
    -- 6. Handle errors
    if not ok then
        -- Wrap error กับ context
        return nil, string.format("robust_operation failed: %s", 
               type(result) == "string" and result or tostring(result))
    end
    
    return result, nil
end

print("=== Best Practices Summary ===")
local test_inputs = {
    {value = 16},
    {value = -4},
    {value = "abc"},
    {},
    "not a table",
    {value = 100},
}

for i, input in ipairs(test_inputs) do
    local result, err = robust_operation(input)
    if result then
        print(string.format("  Test %d: OK -> %.2f", i, result))
    else
        print(string.format("  Test %d: ERR -> %s", i, err))
    end
end
```

---

## สรุปบทที่ 13

### Functions สำคัญ

| Function | การใช้งาน |
|---------|-----------|
| `error(msg, level)` | Throw error |
| `pcall(f, ...)` | Protected call |
| `xpcall(f, handler, ...)` | Protected call + handler |
| `assert(cond, msg)` | Error if false |
| `debug.traceback()` | Stack trace string |
| `debug.getinfo()` | Location info |

### Patterns สรุป

| Pattern | เหมาะกับ |
|---------|---------|
| `pcall` | catch errors ทั่วไป |
| `xpcall` + handler | logging, tracing |
| Error objects (table) | structured errors |
| Result type (val, err) | Go-style, explicit |
| assert() | fast fail, preconditions |
| Contract pattern | domain logic |
| Circuit Breaker | production resilience |

### Error Level Guide
```
level 0 = ไม่มี position info
level 1 = บรรทัดที่เรียก error() เอง  (default)
level 2 = caller ของ function ที่เรียก error()
level 3 = caller ของ caller
```

> **Golden Rule:** ใช้ `pcall` รอบการเรียก external resources เสมอ (file I/O, network, database) และ propagate errors ด้วยการเพิ่ม context แทนที่จะ swallow errors โดยเงียบๆ
