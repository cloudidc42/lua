# บทที่ 16: Closures เชิงลึก

## บทนำ

**Closure** คือฟังก์ชันที่ "จดจำ" environment ที่มันถูกสร้างขึ้น หรือพูดให้ถูกต้องกว่านั้น closure คือ **function + upvalues** รวมกัน

ในบทก่อน เราเห็นตัวอย่าง upvalues พื้นฐานแล้ว บทนี้จะลงลึกถึงเทคนิคการใช้ closures ในรูปแบบต่างๆ ที่นักพัฒนา Lua ระดับสูงใช้จริง

---

## 16.1 Closure คืออะไร

```lua
-- ตัวอย่างที่ 1: closure พื้นฐาน
local x = 10  -- x จะเป็น upvalue

local function add_x(n)
    return n + x  -- x ถูก "close over" (captured)
end

print(add_x(5))   -- 15
x = 20            -- เปลี่ยน x
print(add_x(5))   -- 25  (closure เห็นการเปลี่ยนแปลง!)
```

```lua
-- ตัวอย่างที่ 2: anatomy ของ closure
local function make_closure()
    local private_state = 0          -- upvalue

    -- นี่คือ closure: function + upvalue (private_state)
    return function()
        private_state = private_state + 1
        return private_state
    end
end

local fn = make_closure()

-- ตรวจสอบว่า fn มี upvalue
local name, value = debug.getupvalue(fn, 1)
print("upvalue name:", name)    -- private_state
print("upvalue value:", value)  -- 0

fn()
fn()

local name2, value2 = debug.getupvalue(fn, 1)
print("after calls:", value2)   -- 2
```

```lua
-- ตัวอย่างที่ 3: closures หลายตัว share upvalue เดียวกัน
local function make_shared_state()
    local shared = 0

    -- ทั้งสองฟังก์ชัน share ตัวแปร shared เดียวกัน
    local function increment() shared = shared + 1 end
    local function get() return shared end

    return increment, get
end

local inc, get = make_shared_state()
inc(); inc(); inc()
print(get())  -- 3
```

---

## 16.2 Counter Factory

```lua
-- ตัวอย่างที่ 4: counter factory พื้นฐาน
function make_counter(start, step)
    start = start or 0
    step = step or 1
    local count = start

    return {
        next = function()
            local current = count
            count = count + step
            return current
        end,
        reset = function()
            count = start
        end,
        peek = function()
            return count
        end
    }
end

local c1 = make_counter()
print(c1.next(), c1.next(), c1.next())  -- 0  1  2

local c2 = make_counter(10, 5)
print(c2.next(), c2.next(), c2.next())  -- 10  15  20

c1.reset()
print(c1.peek())  -- 0  (reset แล้ว)
```

```lua
-- ตัวอย่างที่ 5: counter ที่ครบครัน
function make_full_counter(options)
    options = options or {}
    local start     = options.start or 0
    local step      = options.step or 1
    local max_val   = options.max
    local min_val   = options.min
    local on_change = options.on_change

    local count = start
    local history = {}

    local function notify(old, new)
        if on_change then on_change(old, new) end
    end

    return {
        increment = function(n)
            local old = count
            count = count + (n or step)
            if max_val and count > max_val then count = max_val end
            notify(old, count)
            history[#history+1] = count
        end,

        decrement = function(n)
            local old = count
            count = count - (n or step)
            if min_val and count < min_val then count = min_val end
            notify(old, count)
            history[#history+1] = count
        end,

        reset = function()
            local old = count
            count = start
            history = {}
            notify(old, count)
        end,

        get = function() return count end,
        get_history = function() return {table.unpack(history)} end
    }
end

local c = make_full_counter({
    start = 0,
    step = 1,
    max = 5,
    on_change = function(old, new)
        print(string.format("changed: %d -> %d", old, new))
    end
})

c.increment()   -- changed: 0 -> 1
c.increment()   -- changed: 1 -> 2
c.increment(3)  -- changed: 2 -> 5
c.increment()   -- changed: 5 -> 5  (capped at max)

print("history:", table.concat(c.get_history(), ", "))
-- history: 1, 2, 5, 5
```

---

## 16.3 Accumulator

```lua
-- ตัวอย่างที่ 6: simple accumulator
function make_accumulator(initial)
    local total = initial or 0
    return function(n)
        total = total + n
        return total
    end
end

local acc = make_accumulator(0)
print(acc(10))  -- 10
print(acc(20))  -- 30
print(acc(5))   -- 35
```

```lua
-- ตัวอย่างที่ 7: generic accumulator
function make_reducer(initial, reducer_fn)
    local state = initial
    return function(value)
        state = reducer_fn(state, value)
        return state
    end
end

-- sum accumulator
local sum = make_reducer(0, function(acc, x) return acc + x end)
print(sum(1), sum(2), sum(3), sum(4))  -- 1  3  6  10

-- product accumulator
local product = make_reducer(1, function(acc, x) return acc * x end)
print(product(2), product(3), product(4))  -- 2  6  24

-- max accumulator
local running_max = make_reducer(-math.huge, math.max)
print(running_max(3), running_max(7), running_max(2), running_max(9))  -- 3  7  7  9
```

---

## 16.4 Private State ผ่าน Closure

```lua
-- ตัวอย่างที่ 8: object ที่มี private state อย่างแท้จริง
function make_bank_account(owner, initial_balance)
    -- private state - ไม่มีทางเข้าถึงจากภายนอกโดยตรง
    local balance = initial_balance or 0
    local transactions = {}
    local frozen = false

    local function record_transaction(type, amount, note)
        transactions[#transactions+1] = {
            type = type,
            amount = amount,
            balance_after = balance,
            note = note or "",
            time = os.time()
        }
    end

    -- public interface
    return {
        owner = owner,  -- public (read-only ได้)

        deposit = function(amount, note)
            if frozen then return false, "account is frozen" end
            if amount <= 0 then return false, "amount must be positive" end
            balance = balance + amount
            record_transaction("deposit", amount, note)
            return true, balance
        end,

        withdraw = function(amount, note)
            if frozen then return false, "account is frozen" end
            if amount <= 0 then return false, "amount must be positive" end
            if amount > balance then return false, "insufficient funds" end
            balance = balance - amount
            record_transaction("withdrawal", amount, note)
            return true, balance
        end,

        get_balance = function()
            return balance
        end,

        freeze = function()
            frozen = true
        end,

        get_statement = function()
            local lines = {string.format("Account: %s", owner)}
            for _, t in ipairs(transactions) do
                lines[#lines+1] = string.format(
                    "  [%s] %s: %.2f (balance: %.2f) %s",
                    os.date("%H:%M:%S", t.time),
                    t.type, t.amount, t.balance_after, t.note
                )
            end
            lines[#lines+1] = string.format("Current Balance: %.2f", balance)
            return table.concat(lines, "\n")
        end
    }
end

local acct = make_bank_account("Alice", 1000)
local ok, bal = acct.deposit(500, "salary")
print("deposit:", ok, bal)  -- deposit: true  1500

local ok2, bal2 = acct.withdraw(200, "groceries")
print("withdraw:", ok2, bal2)  -- withdraw: true  1300

local ok3, err = acct.withdraw(2000)
print("overdraft:", ok3, err)  -- overdraft: false  insufficient funds

print(acct.get_statement())
```

---

## 16.5 Memoization

```lua
-- ตัวอย่างที่ 9: memoize พื้นฐาน
function memoize(fn)
    local cache = {}
    return function(...)
        local key = table.concat({...}, ",")
        if cache[key] == nil then
            cache[key] = fn(...)
        end
        return cache[key]
    end
end

-- Fibonacci โดยไม่มี memoize: O(2^n) 
-- ด้วย memoize: O(n)
local fib = memoize(function(n)
    if n <= 1 then return n end
    -- เรียกตัวเองโดยตรงไม่ได้ตอน memoize (ต้องใช้ recursive ref)
    return n  -- placeholder
end)

-- วิธีที่ถูกต้องสำหรับ recursive memoize:
local fib_cache = {}
local function fib(n)
    if fib_cache[n] ~= nil then return fib_cache[n] end
    if n <= 1 then
        fib_cache[n] = n
    else
        fib_cache[n] = fib(n-1) + fib(n-2)
    end
    return fib_cache[n]
end

local t = os.clock()
print(fib(40))  -- 102334155
print(string.format("time: %.6f sec", os.clock() - t))
```

```lua
-- ตัวอย่างที่ 10: memoize ที่รองรับ multiple arguments อย่างถูกต้อง
function memoize_v2(fn)
    local cache = {}

    return function(...)
        local args = table.pack(...)
        local node = cache

        -- Navigate/create nested tables as trie
        for i = 1, args.n do
            local k = args[i]
            if node[k] == nil then node[k] = {} end
            node = node[k]
        end

        -- Use sentinel to distinguish nil result from "not cached"
        local CACHED = node._cached
        if not CACHED then
            node._result = table.pack(fn(...))
            node._cached = true
        end

        return table.unpack(node._result, 1, node._result.n)
    end
end

local expensive = memoize_v2(function(x, y)
    print(string.format("computing %d + %d", x, y))
    return x + y, x * y
end)

local s1, p1 = expensive(3, 4)
print(s1, p1)  -- computing 3 + 4 / 7  12

local s2, p2 = expensive(3, 4)  -- จาก cache
print(s2, p2)  -- 7  12 (ไม่พิมพ์ "computing")
```

```lua
-- ตัวอย่างที่ 11: memoize ที่มี TTL (Time-To-Live)
function memoize_ttl(fn, ttl_seconds)
    local cache = {}

    return function(...)
        local key = table.concat({...}, "\0")
        local entry = cache[key]
        local now = os.time()

        if entry and (now - entry.time) < ttl_seconds then
            return table.unpack(entry.results, 1, entry.results.n)
        end

        local results = table.pack(fn(...))
        cache[key] = {results = results, time = now}
        return table.unpack(results, 1, results.n)
    end
end

local call_count = 0
local cached_fn = memoize_ttl(function(x)
    call_count = call_count + 1
    return x * x
end, 60)  -- cache 60 วินาที

print(cached_fn(5))  -- 25 (call 1)
print(cached_fn(5))  -- 25 (from cache)
print(cached_fn(5))  -- 25 (from cache)
print("calls made:", call_count)  -- 1
```

---

## 16.6 Currying

```lua
-- ตัวอย่างที่ 12: curry สำหรับ 2 arguments
function curry2(fn)
    return function(a)
        return function(b)
            return fn(a, b)
        end
    end
end

local add = curry2(function(a, b) return a + b end)
local add5 = add(5)

print(add5(3))   -- 8
print(add5(10))  -- 15
print(add(2)(3)) -- 5
```

```lua
-- ตัวอย่างที่ 13: curry สำหรับ n arguments
function curry(fn, n)
    n = n or debug.getinfo(fn, "u").nparams

    local function helper(collected)
        if #collected >= n then
            return fn(table.unpack(collected, 1, n))
        end
        return function(...)
            local new_collected = {table.unpack(collected)}
            for i = 1, select('#', ...) do
                new_collected[#new_collected+1] = select(i, ...)
            end
            return helper(new_collected)
        end
    end

    return helper({})
end

local add3 = curry(function(a, b, c) return a + b + c end, 3)

print(add3(1)(2)(3))    -- 6
print(add3(1, 2)(3))    -- 6
print(add3(1)(2, 3))    -- 6
print(add3(1, 2, 3))    -- 6

-- Practical example: curried string operations
local format_string = curry(function(fmt, value)
    return string.format(fmt, value)
end, 2)

local format_percent = format_string("%.1f%%")
local format_dollars = format_string("$%.2f")

print(format_percent(95.5))   -- 95.5%
print(format_dollars(1234.5)) -- $1234.50
```

```lua
-- ตัวอย่างที่ 14: auto-curry ด้วย __call metamethod
function auto_curry(fn, arity)
    arity = arity or debug.getinfo(fn, "u").nparams or 1

    local function make_curried(collected_args)
        if #collected_args >= arity then
            return fn(table.unpack(collected_args, 1, arity))
        end

        -- Return a new function that collects more args
        local curried = setmetatable({}, {
            __call = function(self, ...)
                local new_args = {table.unpack(collected_args)}
                for i = 1, select('#', ...) do
                    new_args[#new_args+1] = select(i, ...)
                end
                return make_curried(new_args)
            end
        })
        return curried
    end

    return make_curried({})
end

local multiply = auto_curry(function(a, b, c) return a * b * c end, 3)
print(multiply(2)(3)(4))   -- 24
print(multiply(2, 3)(4))   -- 24
```

---

## 16.7 Partial Application

```lua
-- ตัวอย่างที่ 15: partial application
function partial(fn, ...)
    local pre_args = table.pack(...)
    return function(...)
        local args = {}
        for i = 1, pre_args.n do args[#args+1] = pre_args[i] end
        for i = 1, select('#', ...) do args[#args+1] = select(i, ...) end
        return fn(table.unpack(args))
    end
end

local function log(level, module, message)
    print(string.format("[%s][%s] %s", level, module, message))
end

local info_log = partial(log, "INFO")
local error_log = partial(log, "ERROR")
local db_info = partial(info_log, "Database")

info_log("API", "Request received")       -- [INFO][API] Request received
error_log("Auth", "Invalid token")        -- [ERROR][Auth] Invalid token
db_info("Query executed in 5ms")          -- [INFO][Database] Query executed in 5ms
```

```lua
-- ตัวอย่างที่ 16: partial application กับ placeholder
local _ = {}  -- placeholder สำหรับ "ยังไม่ได้ใส่"

function partial_with_placeholder(fn, ...)
    local template = table.pack(...)

    return function(...)
        local fill = table.pack(...)
        local args = {}
        local fill_idx = 1

        for i = 1, template.n do
            if template[i] == _ then
                args[i] = fill[fill_idx]
                fill_idx = fill_idx + 1
            else
                args[i] = template[i]
            end
        end

        -- Append remaining fill args
        while fill_idx <= fill.n do
            args[#args+1] = fill[fill_idx]
            fill_idx = fill_idx + 1
        end

        return fn(table.unpack(args, 1, #args))
    end
end

local divide = function(a, b) return a / b end
local invert = partial_with_placeholder(divide, _, 1)  -- 1/x
local half_of = partial_with_placeholder(divide, _, 2) -- x/2

print(invert(4))    -- 0.25  (1/4)
print(half_of(10))  -- 5.0   (10/2)

local greet = function(greeting, name, punct)
    return greeting .. ", " .. name .. punct
end
local exclaim = partial_with_placeholder(greet, _, _, "!")
print(exclaim("Hello", "World"))   -- Hello, World!
print(exclaim("Goodbye", "Alice")) -- Goodbye, Alice!
```

---

## 16.8 Function Composition

```lua
-- ตัวอย่างที่ 17: compose 2 functions
function compose2(f, g)
    return function(...)
        return f(g(...))
    end
end

local double = function(x) return x * 2 end
local inc = function(x) return x + 1 end

local double_then_inc = compose2(inc, double)  -- inc(double(x))
local inc_then_double = compose2(double, inc)  -- double(inc(x))

print(double_then_inc(5))  -- 11  (5*2+1)
print(inc_then_double(5))  -- 12  ((5+1)*2)
```

```lua
-- ตัวอย่างที่ 18: compose หลายฟังก์ชัน
function compose(...)
    local fns = table.pack(...)
    return function(x)
        -- apply ฟังก์ชันจากขวาไปซ้าย
        for i = fns.n, 1, -1 do
            x = fns[i](x)
        end
        return x
    end
end

local function square(x) return x * x end
local function negate(x) return -x end
local function add_100(x) return x + 100 end

-- compose(f, g, h)(x) = f(g(h(x)))
local transform = compose(add_100, negate, square)
print(transform(3))  -- add_100(negate(square(3))) = add_100(negate(9)) = add_100(-9) = 91
```

```lua
-- ตัวอย่างที่ 19: pipe (compose จากซ้ายไปขวา)
function pipe(...)
    local fns = table.pack(...)
    return function(x)
        for i = 1, fns.n do
            x = fns[i](x)
        end
        return x
    end
end

-- pipe(f, g, h)(x) = h(g(f(x)))
local process = pipe(
    function(x) return x * 2 end,    -- step 1: double
    function(x) return x + 10 end,   -- step 2: add 10
    function(x) return x / 2 end     -- step 3: halve
)

print(process(5))   -- ((5*2)+10)/2 = 10.0
print(process(10))  -- ((10*2)+10)/2 = 15.0
```

```lua
-- ตัวอย่างที่ 20: compose กับ multiple return values
function compose_multi(...)
    local fns = table.pack(...)
    return function(...)
        local results = table.pack(...)
        for i = fns.n, 1, -1 do
            results = table.pack(fns[i](table.unpack(results, 1, results.n)))
        end
        return table.unpack(results, 1, results.n)
    end
end

local function parse_num(s)
    return tonumber(s), type(s)
end

local function double_with_type(n, t)
    return n * 2, t .. "_doubled"
end

local transform = compose_multi(double_with_type, parse_num)
print(transform("21"))  -- 42.0  string_doubled
```

---

## 16.9 Pipeline/Chain Functions

```lua
-- ตัวอย่างที่ 21: data pipeline
local Pipeline = {}
Pipeline.__index = Pipeline

function Pipeline.new(value)
    return setmetatable({_value = value, _steps = {}}, Pipeline)
end

function Pipeline:map(fn)
    self._steps[#self._steps+1] = {type="map", fn=fn}
    return self
end

function Pipeline:filter(predicate)
    self._steps[#self._steps+1] = {type="filter", fn=predicate}
    return self
end

function Pipeline:reduce(fn, initial)
    self._steps[#self._steps+1] = {type="reduce", fn=fn, initial=initial}
    return self
end

function Pipeline:run()
    local data = self._value
    for _, step in ipairs(self._steps) do
        if step.type == "map" then
            local result = {}
            for i, v in ipairs(data) do result[i] = step.fn(v) end
            data = result
        elseif step.type == "filter" then
            local result = {}
            for _, v in ipairs(data) do
                if step.fn(v) then result[#result+1] = v end
            end
            data = result
        elseif step.type == "reduce" then
            local acc = step.initial
            for _, v in ipairs(data) do acc = step.fn(acc, v) end
            data = acc
        end
    end
    return data
end

local result = Pipeline.new({1, 2, 3, 4, 5, 6, 7, 8, 9, 10})
    :filter(function(x) return x % 2 == 0 end)  -- เลขคู่
    :map(function(x) return x * x end)          -- ยกกำลัง 2
    :reduce(function(acc, x) return acc + x end, 0)  -- รวม
    :run()

print("Sum of squares of even numbers:", result)  -- 2²+4²+6²+8²+10² = 220
```

---

## 16.10 Iterator Factories

```lua
-- ตัวอย่างที่ 22: range iterator
function range(from, to, step)
    step = step or 1
    local current = from - step
    return function()
        current = current + step
        if (step > 0 and current <= to) or (step < 0 and current >= to) then
            return current
        end
    end
end

print("range(1,5):")
for i in range(1, 5) do io.write(i .. " ") end
print()  -- 1 2 3 4 5

print("range(0, 10, 2):")
for i in range(0, 10, 2) do io.write(i .. " ") end
print()  -- 0 2 4 6 8 10

print("range(5, 1, -1):")
for i in range(5, 1, -1) do io.write(i .. " ") end
print()  -- 5 4 3 2 1
```

```lua
-- ตัวอย่างที่ 23: filtered iterator
function filter_iter(iter, predicate)
    return function()
        while true do
            local v = iter()
            if v == nil then return nil end
            if predicate(v) then return v end
        end
    end
end

-- เลขคี่ใน range 1-20
for v in filter_iter(range(1, 20), function(x) return x % 2 ~= 0 end) do
    io.write(v .. " ")
end
print()  -- 1 3 5 7 9 11 13 15 17 19
```

```lua
-- ตัวอย่างที่ 24: map iterator
function map_iter(iter, transform)
    return function()
        local v = iter()
        if v == nil then return nil end
        return transform(v)
    end
end

-- กำลังสองของเลข 1-5
for v in map_iter(range(1, 5), function(x) return x*x end) do
    io.write(v .. " ")
end
print()  -- 1 4 9 16 25
```

```lua
-- ตัวอย่างที่ 25: take iterator
function take(n, iter)
    local count = 0
    return function()
        if count >= n then return nil end
        count = count + 1
        return iter()
    end
end

-- เลขคี่แรก 5 ตัว
local odd_numbers = filter_iter(range(1, math.huge), function(x) return x%2~=0 end)
for v in take(5, odd_numbers) do
    io.write(v .. " ")
end
print()  -- 1 3 5 7 9
```

```lua
-- ตัวอย่างที่ 26: zip iterator
function zip_iter(...)
    local iters = {...}
    return function()
        local vals = {}
        for i, iter in ipairs(iters) do
            local v = iter()
            if v == nil then return nil end
            vals[i] = v
        end
        return table.unpack(vals)
    end
end

for a, b, c in zip_iter(range(1,3), range(10,30,10), range(100,300,100)) do
    print(a, b, c)
end
-- 1  10  100
-- 2  20  200
-- 3  30  300
```

---

## 16.11 Event Handlers ด้วย Closures

```lua
-- ตัวอย่างที่ 27: event system
local EventSystem = {}

function EventSystem.new()
    local handlers = {}
    local once_handlers = {}

    return {
        on = function(event, handler)
            if not handlers[event] then handlers[event] = {} end
            table.insert(handlers[event], handler)
            -- return unsubscribe function (closure!)
            return function()
                local t = handlers[event]
                for i, h in ipairs(t) do
                    if h == handler then
                        table.remove(t, i)
                        return true
                    end
                end
                return false
            end
        end,

        once = function(event, handler)
            local wrapper
            wrapper = function(...)
                handler(...)
                -- remove self after first call
                local t = handlers[event]
                for i, h in ipairs(t) do
                    if h == wrapper then
                        table.remove(t, i)
                        return
                    end
                end
            end
            if not handlers[event] then handlers[event] = {} end
            table.insert(handlers[event], wrapper)
        end,

        emit = function(event, ...)
            local event_handlers = handlers[event]
            if event_handlers then
                -- copy เพื่อ prevent mutation during iteration
                local copy = {table.unpack(event_handlers)}
                for _, handler in ipairs(copy) do
                    handler(...)
                end
            end
        end,

        off = function(event)
            handlers[event] = nil
        end
    }
end

local events = EventSystem.new()

-- Subscribe
local unsub = events.on("click", function(x, y)
    print(string.format("clicked at (%d, %d)", x, y))
end)

events.once("connect", function(host)
    print("connected to:", host)  -- พิมพ์แค่ครั้งเดียว
end)

-- Emit
events.emit("click", 10, 20)    -- clicked at (10, 20)
events.emit("click", 30, 40)    -- clicked at (30, 40)
events.emit("connect", "server.example.com")  -- connected to: server.example.com
events.emit("connect", "other.example.com")   -- ไม่พิมพ์! (once)

-- Unsubscribe
unsub()
events.emit("click", 50, 60)  -- ไม่พิมพ์! (unsubscribed)
```

---

## 16.12 Module ที่มี Private State

```lua
-- ตัวอย่างที่ 28: module pattern ด้วย closure
local Cache = (function()
    -- Private state ที่ module ทั้งหมดเข้าถึงได้แต่ภายนอกไม่ได้
    local store = {}
    local hit_count = 0
    local miss_count = 0
    local max_size = 100

    -- Private functions
    local function evict()
        -- Simple eviction: ลบครึ่งหนึ่ง
        local keys = {}
        for k in pairs(store) do keys[#keys+1] = k end
        for i = 1, math.floor(#keys / 2) do
            store[keys[i]] = nil
        end
    end

    -- Public interface
    return {
        get = function(key)
            local entry = store[key]
            if entry then
                hit_count = hit_count + 1
                entry.accessed = os.time()
                return entry.value
            end
            miss_count = miss_count + 1
            return nil
        end,

        set = function(key, value)
            local size = 0
            for _ in pairs(store) do size = size + 1 end
            if size >= max_size then evict() end
            store[key] = {value = value, set_at = os.time()}
        end,

        delete = function(key)
            store[key] = nil
        end,

        stats = function()
            local size = 0
            for _ in pairs(store) do size = size + 1 end
            local total = hit_count + miss_count
            return {
                size = size,
                hits = hit_count,
                misses = miss_count,
                hit_rate = total > 0 and hit_count/total or 0
            }
        end,

        clear = function()
            store = {}
            hit_count = 0
            miss_count = 0
        end
    }
end)()  -- IIFE: Immediately Invoked Function Expression

Cache.set("user:1", {name="Alice", age=30})
Cache.set("user:2", {name="Bob", age=25})

local u = Cache.get("user:1")
print(u.name)  -- Alice

Cache.get("user:999")  -- miss

local s = Cache.stats()
print(string.format("size=%d hits=%d misses=%d rate=%.0f%%",
    s.size, s.hits, s.misses, s.hit_rate * 100))
-- size=2 hits=1 misses=1 rate=50%
```

---

## 16.13 Lazy Evaluation และ Thunks

```lua
-- ตัวอย่างที่ 29: thunk พื้นฐาน
function thunk(fn, ...)
    local args = table.pack(...)
    return function()
        return fn(table.unpack(args, 1, args.n))
    end
end

-- สร้าง computation แต่ยังไม่รัน
local expensive_calc = thunk(function(n)
    print("calculating...")
    local sum = 0
    for i = 1, n do sum = sum + i end
    return sum
end, 1000000)

print("before computation")
-- calculation ยังไม่เกิด
print("computing:", expensive_calc())  -- calculating... / computing: 500000500000
```

```lua
-- ตัวอย่างที่ 30: lazy value ที่คำนวณครั้งเดียว
function lazy(fn)
    local computed = false
    local value

    return function()
        if not computed then
            value = fn()
            computed = true
        end
        return value
    end
end

local lazy_pi = lazy(function()
    print("computing pi approximation...")
    local pi = 0
    for i = 0, 10000 do
        pi = pi + ((-1)^i) / (2*i + 1)
    end
    return pi * 4
end)

print("pi =", lazy_pi())  -- computing pi approximation... / pi = 3.1415...
print("pi =", lazy_pi())  -- pi = 3.1415... (no recompute)
```

```lua
-- ตัวอย่างที่ 31: lazy sequence
function lazy_seq(gen_fn)
    local cache = {}
    local generator = gen_fn()

    return function(n)
        while #cache < n do
            cache[#cache+1] = generator()
        end
        return cache[n]
    end
end

-- Fibonacci sequence (lazy)
local fibs = lazy_seq(function()
    local a, b = 0, 1
    return function()
        local val = a
        a, b = b, a + b
        return val
    end
end)

for i = 1, 10 do
    io.write(fibs(i) .. " ")
end
print()  -- 0 1 1 2 3 5 8 13 21 34
print(fibs(10))  -- 34 (จาก cache)
```

---

## 16.14 Once() Function

```lua
-- ตัวอย่างที่ 32: once - รันแค่ครั้งเดียว
function once(fn)
    local called = false
    local result

    return function(...)
        if not called then
            called = true
            result = table.pack(fn(...))
        end
        return table.unpack(result, 1, result.n)
    end
end

local init = once(function()
    print("initializing... (should only see this once)")
    return "initialized"
end)

print(init())  -- initializing... / initialized
print(init())  -- initialized  (no re-init)
print(init())  -- initialized  (no re-init)
```

```lua
-- ตัวอย่างที่ 33: once กับ reset
function resettable_once(fn)
    local called = false
    local result

    return {
        call = function(...)
            if not called then
                called = true
                result = table.pack(fn(...))
            end
            return table.unpack(result, 1, result.n)
        end,
        reset = function()
            called = false
            result = nil
        end,
        was_called = function()
            return called
        end
    }
end

local setup = resettable_once(function(x)
    print("setup with:", x)
    return x * 2
end)

print(setup.call(5))        -- setup with: 5 / 10
print(setup.call(99))       -- 10 (ignored, already called)
print(setup.was_called())   -- true

setup.reset()
print(setup.call(20))       -- setup with: 20 / 40
```

---

## 16.15 Debounce และ Throttle

```lua
-- ตัวอย่างที่ 34: debounce
function debounce(fn, delay_ms)
    local last_call_time = -math.huge
    local scheduled = false

    -- ในสภาพแวดล้อมจริงจะใช้ timer หรือ event loop
    -- ตัวอย่างนี้ simulate ด้วย time check
    return function(...)
        local args = table.pack(...)
        local now = os.clock() * 1000  -- milliseconds

        if now - last_call_time >= delay_ms then
            last_call_time = now
            return fn(table.unpack(args, 1, args.n))
        end
        -- ถ้าเรียกบ่อยเกินไป ไม่ทำอะไร
    end
end

local call_count = 0
local debounced = debounce(function(x)
    call_count = call_count + 1
    print("called with:", x, "count:", call_count)
end, 100)  -- 100ms

debounced(1)   -- เรียกได้ (ครั้งแรก)
debounced(2)   -- อาจถูก debounce (ขึ้นกับ timing)
debounced(3)   -- อาจถูก debounce
```

```lua
-- ตัวอย่างที่ 35: throttle
function throttle(fn, interval_ms)
    local last_call = -math.huge

    return function(...)
        local now = os.clock() * 1000
        if now - last_call >= interval_ms then
            last_call = now
            return fn(...)
        end
    end
end

local throttled_log = throttle(function(msg)
    print(os.date("%H:%M:%S"), msg)
end, 1000)  -- maximum 1 call per second

-- Rapid calls
for i = 1, 5 do
    throttled_log("event " .. i)
end
-- จะเห็นแค่ event ที่ผ่าน throttle (ไม่เกิน 1 ต่อวินาที)
```

---

## 16.16 Rate Limiter

```lua
-- ตัวอย่างที่ 36: rate limiter แบบ token bucket
function make_rate_limiter(rate, burst)
    local tokens = burst
    local last_refill = os.clock()
    local max_tokens = burst

    return function()
        local now = os.clock()
        local elapsed = now - last_refill
        last_refill = now

        -- เติม tokens ตาม rate
        tokens = math.min(max_tokens, tokens + elapsed * rate)

        if tokens >= 1 then
            tokens = tokens - 1
            return true  -- allowed
        end
        return false  -- rate limited
    end
end

local limiter = make_rate_limiter(2, 5)  -- 2 req/sec, burst of 5

-- ทดสอบด้วยการ simulate rapid requests
local allowed = 0
local denied = 0
for i = 1, 10 do
    if limiter() then
        allowed = allowed + 1
    else
        denied = denied + 1
    end
end

print(string.format("allowed=%d, denied=%d", allowed, denied))
-- burst ของ 5 ให้ผ่านก่อน แล้วค่อยๆ rate limit
```

---

## 16.17 Observable Pattern

```lua
-- ตัวอย่างที่ 37: observable value
function make_observable(initial_value)
    local value = initial_value
    local subscribers = {}

    return {
        get = function() return value end,

        set = function(new_value)
            local old_value = value
            value = new_value
            if old_value ~= new_value then
                for _, handler in ipairs(subscribers) do
                    handler(new_value, old_value)
                end
            end
        end,

        subscribe = function(handler)
            subscribers[#subscribers+1] = handler
            -- Return unsubscribe function
            return function()
                for i, h in ipairs(subscribers) do
                    if h == handler then
                        table.remove(subscribers, i)
                        return
                    end
                end
            end
        end,

        -- computed value
        computed = function(transform)
            local computed_obs = make_observable(transform(value))
            -- Subscribe to this observable and update computed
            local function update(v)
                computed_obs.set(transform(v))
            end
            -- Add subscriber manually
            subscribers[#subscribers+1] = update
            return computed_obs
        end
    }
end

local score = make_observable(0)
local grade = score.computed(function(s)
    if s >= 90 then return "A"
    elseif s >= 80 then return "B"
    elseif s >= 70 then return "C"
    elseif s >= 60 then return "D"
    else return "F"
    end
end)

score.subscribe(function(new, old)
    print(string.format("score changed: %d -> %d (grade: %s)",
        old, new, grade.get()))
end)

score.set(75)   -- score changed: 0 -> 75 (grade: C)
score.set(85)   -- score changed: 75 -> 85 (grade: B)
score.set(92)   -- score changed: 85 -> 92 (grade: A)
score.set(92)   -- ไม่มี notification (ค่าไม่เปลี่ยน)
```

---

## 16.18 Stateful Parsers

```lua
-- ตัวอย่างที่ 38: parser combinator พื้นฐาน
-- Parser คือ function ที่รับ string + position และคืน value + new position

local function literal(expected)
    return function(input, pos)
        pos = pos or 1
        if input:sub(pos, pos + #expected - 1) == expected then
            return expected, pos + #expected
        end
        return nil, pos, "expected: " .. expected
    end
end

local function digits()
    return function(input, pos)
        pos = pos or 1
        local result = input:match("^(%d+)", pos)
        if result then
            return tonumber(result), pos + #result
        end
        return nil, pos, "expected digits"
    end
end

local function sequence(...)
    local parsers = {...}
    return function(input, pos)
        pos = pos or 1
        local results = {}
        for _, parser in ipairs(parsers) do
            local result, new_pos, err = parser(input, pos)
            if result == nil then
                return nil, pos, err
            end
            results[#results+1] = result
            pos = new_pos
        end
        return results, pos
    end
end

-- Parse "42+17"
local parse_addition = sequence(digits(), literal("+"), digits())
local result, pos = parse_addition("42+17")
if result then
    print(string.format("%d + %d = %d", result[1], result[3], result[1]+result[3]))
    -- 42 + 17 = 59
end
```

```lua
-- ตัวอย่างที่ 39: stateful tokenizer
function make_tokenizer(source)
    local pos = 1
    local tokens = {}
    local current_token = nil

    local function skip_whitespace()
        pos = source:match("^%s*()", pos)
    end

    local function next_token()
        skip_whitespace()
        if pos > #source then return nil end

        -- Number
        local num = source:match("^%-?%d+%.?%d*", pos)
        if num then
            pos = pos + #num
            return {type="number", value=tonumber(num)}
        end

        -- Identifier/keyword
        local id = source:match("^[%a_][%w_]*", pos)
        if id then
            pos = pos + #id
            return {type="identifier", value=id}
        end

        -- Operator
        local op = source:match("^[%+%-%*/%(%)%=]", pos)
        if op then
            pos = pos + 1
            return {type="operator", value=op}
        end

        -- Unknown
        local ch = source:sub(pos, pos)
        pos = pos + 1
        return {type="unknown", value=ch}
    end

    return {
        peek = function()
            if current_token == nil then
                current_token = next_token()
            end
            return current_token
        end,

        consume = function()
            local tok = current_token or next_token()
            current_token = nil
            return tok
        end,

        has_more = function()
            skip_whitespace()
            return pos <= #source or current_token ~= nil
        end
    }
end

local tokenizer = make_tokenizer("x = 42 + y * 3.14")
while tokenizer.has_more() do
    local tok = tokenizer.consume()
    if tok then
        print(string.format("%-12s %s", tok.type, tostring(tok.value)))
    end
end
```

---

## 16.19 Advanced: Closure-Based OOP

```lua
-- ตัวอย่างที่ 40: OOP ด้วย closures (ไม่ใช้ metatables)
function Animal(name, sound)
    -- Private state
    local _name = name
    local _sound = sound
    local _energy = 100

    -- Public interface
    local self = {}

    function self.getName() return _name end
    function self.getEnergy() return _energy end

    function self.speak()
        if _energy <= 0 then
            return _name .. " is too tired to speak"
        end
        _energy = _energy - 10
        return _name .. " says: " .. _sound
    end

    function self.eat(amount)
        _energy = math.min(100, _energy + (amount or 20))
        return _name .. " ate and has " .. _energy .. " energy"
    end

    function self.toString()
        return string.format("Animal(%s, energy=%d)", _name, _energy)
    end

    return self
end

function Dog(name)
    local base = Animal(name, "Woof")
    local self = {}

    -- Inherit all methods
    for k, v in pairs(base) do self[k] = v end

    -- Override speak
    function self.speak()
        return base.speak() .. "!"  -- Dogs are more enthusiastic
    end

    -- Add new method
    function self.fetch(item)
        return name .. " fetches the " .. item .. "!"
    end

    return self
end

local cat = Animal("Whiskers", "Meow")
local dog = Dog("Rex")

print(cat.speak())   -- Whiskers says: Meow
print(dog.speak())   -- Rex says: Woof!
print(dog.fetch("ball"))  -- Rex fetches the ball!
print(cat.eat(30))   -- Whiskers ate and has 100 energy
```

---

## แบบฝึกหัด (Exercises)

### ระดับพื้นฐาน

**แบบฝึกหัดที่ 1:** เขียน `make_adder(n)` factory

```lua
function make_adder(n)
    -- TODO: คืน function ที่บวก n เข้ากับ argument
end

local add10 = make_adder(10)
print(add10(5))   -- 15
print(add10(20))  -- 30
```

**แบบฝึกหัดที่ 2:** เขียน `toggle()` ที่สลับ true/false

```lua
function toggle(initial)
    -- TODO: คืน function ที่สลับค่า
end

local switch = toggle(false)
print(switch())  -- true
print(switch())  -- false
print(switch())  -- true
```

**แบบฝึกหัดที่ 3:** เขียน `make_multiplier_table(n)` ที่คืน table ของ closures

### ระดับกลาง

**แบบฝึกหัดที่ 4:** เขียน `compose_n(fns)` รับ array ของ functions

**แบบฝึกหัดที่ 5:** เขียน `make_validator(rules)` ที่ closure เก็บ rules

```lua
local validator = make_validator({
    min_length = 3,
    max_length = 20,
    must_start_with_letter = true
})

print(validator("ab"))        -- false, "too short"
print(validator("hello"))     -- true
print(validator("123invalid"))  -- false, "must start with letter"
```

**แบบฝึกหัดที่ 6:** เขียน `retry_with_backoff(fn, max_retries)` ที่ใช้ exponential backoff

### ระดับยาก

**แบบฝึกหัดที่ 7:** เขียน `promise` system แบบ simplistic

```lua
-- Template:
local function promise(executor)
    -- TODO: implement Promise-like async pattern
    -- .then(fn) -> chain transformations
    -- .catch(fn) -> handle errors
    -- .resolve(value) -> resolve
end
```

**แบบฝึกหัดที่ 8:** เขียน generator ที่ใช้ coroutines กับ closures

**แบบฝึกหัดที่ 9:** เขียน `reactive table` ที่ notify เมื่อค่าเปลี่ยน

**แบบฝึกหัดที่ 10:** implement สถาปัตยกรรม Observer pattern ครบถ้วน

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Closure คืออะไร** - function + upvalues ที่ captured
2. **Counter Factory** - สร้าง stateful counters
3. **Accumulator** - สะสมค่าผ่าน closure
4. **Private State** - ข้อมูล private ที่แท้จริง
5. **Memoization** - cache ผลลัพธ์ด้วย closure
6. **Currying** - แปลงฟังก์ชัน n-args เป็น chain ของ 1-arg
7. **Partial Application** - pre-fill บาง arguments
8. **Function Composition** - รวมฟังก์ชันเป็น pipeline
9. **Iterator Factories** - สร้าง custom iterators
10. **Event Handlers** - event system ด้วย closures
11. **Lazy Evaluation** - คำนวณเมื่อจำเป็น
12. **Once/Debounce/Throttle** - control execution
13. **Observable Pattern** - reactive programming
14. **Stateful Parsers** - closures สำหรับ parsing

Closures เป็นหนึ่งในเครื่องมือที่ทรงพลังที่สุดใน Lua ทำให้สามารถสร้าง abstractions ที่ elegant และ flexible ได้โดยไม่ต้องพึ่ง class-based OOP
