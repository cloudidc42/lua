# บทที่ 94: Advanced Profiling และ Performance Analysis

## บทนำ

Profiling คือกระบวนการวัดประสิทธิภาพของโปรแกรมอย่างละเอียด ช่วยให้เราระบุ bottleneck และ optimize ได้อย่างถูกจุด

---

## 94.1 หลักการ Profiling

```
"Premature optimization is the root of all evil" - Donald Knuth

วิธีที่ถูกต้อง:
1. ทำให้ code ถูกต้องก่อน (Make it correct)
2. วัดประสิทธิภาพ (Measure)
3. ระบุ bottleneck (Find hotspots)
4. Optimize เฉพาะส่วนที่ช้า (Optimize)
5. วัดซ้ำ (Re-measure)
```

---

## 94.2 Basic Timing

```lua
-- Simple timing with os.clock() (CPU time)
local function time_function(fn, iterations)
    iterations = iterations or 1000
    
    -- Warm up
    for i = 1, 10 do fn() end
    
    -- Actual measurement
    local start = os.clock()
    for i = 1, iterations do
        fn()
    end
    local elapsed = os.clock() - start
    
    return {
        total_ms = elapsed * 1000,
        per_call_us = elapsed * 1000000 / iterations,
        iterations = iterations
    }
end

-- Compare two implementations
local function bench_compare(name1, fn1, name2, fn2, iterations)
    local r1 = time_function(fn1, iterations)
    local r2 = time_function(fn2, iterations)
    
    print(string.format("Benchmark (%d iterations):", iterations))
    print(string.format("  %-30s %10.2f μs/call", name1, r1.per_call_us))
    print(string.format("  %-30s %10.2f μs/call", name2, r2.per_call_us))
    
    local ratio = r1.per_call_us / r2.per_call_us
    if ratio > 1 then
        print(string.format("  %s is %.1fx SLOWER than %s", name1, ratio, name2))
    else
        print(string.format("  %s is %.1fx FASTER than %s", name1, 1/ratio, name2))
    end
end

-- Example: String building comparison
bench_compare(
    "string concat (..)",
    function()
        local s = ""
        for i = 1, 100 do
            s = s .. tostring(i)
        end
        return s
    end,
    "table.concat",
    function()
        local t = {}
        for i = 1, 100 do
            t[i] = tostring(i)
        end
        return table.concat(t)
    end,
    1000
)
```

---

## 94.3 Statistical Benchmarking

```lua
-- Statistical benchmark (multiple samples)
local function benchmark(fn, options)
    options = options or {}
    local warmup = options.warmup or 5
    local samples = options.samples or 10
    local iterations = options.iterations or 1000
    
    -- Warm up
    for i = 1, warmup do fn() end
    
    local results = {}
    
    for s = 1, samples do
        collectgarbage("collect")  -- Clean GC before each sample
        
        local mem_before = collectgarbage("count")
        local start = os.clock()
        
        for i = 1, iterations do
            fn()
        end
        
        local elapsed = os.clock() - start
        local mem_after = collectgarbage("count")
        
        table.insert(results, {
            time = elapsed,
            memory = mem_after - mem_before
        })
    end
    
    -- Calculate statistics
    local times = {}
    local memories = {}
    for _, r in ipairs(results) do
        table.insert(times, r.time * 1000000 / iterations)  -- μs per call
        table.insert(memories, r.memory)
    end
    
    table.sort(times)
    table.sort(memories)
    
    local sum = 0
    for _, t in ipairs(times) do sum = sum + t end
    local mean = sum / #times
    
    local variance = 0
    for _, t in ipairs(times) do
        variance = variance + (t - mean)^2
    end
    local stddev = math.sqrt(variance / #times)
    
    return {
        mean_us = mean,
        stddev_us = stddev,
        min_us = times[1],
        max_us = times[#times],
        p50_us = times[math.floor(#times * 0.5)],
        p90_us = times[math.floor(#times * 0.9)],
        p99_us = times[#times],  -- with 10 samples, last is ~p99
        samples = samples,
        iterations = iterations,
        cv_percent = (stddev / mean) * 100
    }
end

local function print_benchmark(name, result)
    print(string.format("\n=== %s ===", name))
    print(string.format("  Mean:    %8.2f μs", result.mean_us))
    print(string.format("  StdDev:  %8.2f μs (CV: %.1f%%)", 
        result.stddev_us, result.cv_percent))
    print(string.format("  Min:     %8.2f μs", result.min_us))
    print(string.format("  P50:     %8.2f μs", result.p50_us))
    print(string.format("  P90:     %8.2f μs", result.p90_us))
    print(string.format("  Max:     %8.2f μs", result.max_us))
end

-- Run benchmarks
print_benchmark("Table lookup", benchmark(function()
    local t = { a=1, b=2, c=3, d=4, e=5 }
    return t.c
end))

print_benchmark("Function call overhead", benchmark(function()
    local function noop() end
    noop()
end))

print_benchmark("String format", benchmark(function()
    return string.format("Hello, %s! You are %d years old.", "Alice", 30)
end))
```

---

## 94.4 Call Graph Profiler

```lua
-- Instrumentation-based call graph profiler
local CallProfiler = {}
CallProfiler.__index = CallProfiler

function CallProfiler.new()
    return setmetatable({
        calls = {},      -- function_name -> {count, total_time, children}
        call_stack = {},
        enabled = false
    }, CallProfiler)
end

-- Instrument a module
function CallProfiler:instrument(module_table, module_name)
    for name, fn in pairs(module_table) do
        if type(fn) == "function" then
            local full_name = (module_name or "") .. "." .. name
            module_table[name] = self:wrap(fn, full_name)
        end
    end
    return module_table
end

function CallProfiler:wrap(fn, name)
    local profiler = self
    return function(...)
        if not profiler.enabled then return fn(...) end
        
        local entry = profiler.calls[name]
        if not entry then
            entry = { name=name, count=0, total_ns=0, self_ns=0 }
            profiler.calls[name] = entry
        end
        
        -- Track call stack
        local parent = profiler.call_stack[#profiler.call_stack]
        table.insert(profiler.call_stack, name)
        
        local start = os.clock()
        local results = { fn(...) }
        local elapsed_ns = (os.clock() - start) * 1e9
        
        table.remove(profiler.call_stack)
        
        entry.count = entry.count + 1
        entry.total_ns = entry.total_ns + elapsed_ns
        
        return table.unpack(results)
    end
end

function CallProfiler:enable()
    self.enabled = true
end

function CallProfiler:disable()
    self.enabled = false
end

function CallProfiler:reset()
    self.calls = {}
    self.call_stack = {}
end

function CallProfiler:report(top_n)
    top_n = top_n or 20
    
    -- Sort by total time
    local sorted = {}
    for _, entry in pairs(self.calls) do
        table.insert(sorted, entry)
    end
    table.sort(sorted, function(a, b)
        return a.total_ns > b.total_ns
    end)
    
    print(string.format("\n%-40s %8s %12s %12s",
        "Function", "Calls", "Total (ms)", "Avg (μs)"))
    print(string.rep("-", 76))
    
    for i = 1, math.min(top_n, #sorted) do
        local e = sorted[i]
        print(string.format("%-40s %8d %12.3f %12.3f",
            e.name, e.count,
            e.total_ns / 1e6,
            e.total_ns / e.count / 1000))
    end
end

-- ตัวอย่างการใช้งาน
local profiler = CallProfiler.new()

-- Create a test module
local MyMath = {
    factorial = function(n)
        if n <= 1 then return 1 end
        return n * MyMath.factorial(n - 1)
    end,
    
    fibonacci = function(n)
        if n <= 1 then return n end
        return MyMath.fibonacci(n-1) + MyMath.fibonacci(n-2)
    end,
    
    is_prime = function(n)
        if n < 2 then return false end
        for i = 2, math.sqrt(n) do
            if n % i == 0 then return false end
        end
        return true
    end
}

-- Instrument the module
profiler:instrument(MyMath, "MyMath")
profiler:enable()

-- Run some operations
for i = 1, 10 do
    MyMath.factorial(10)
end
for i = 1, 5 do
    MyMath.fibonacci(15)
end
for i = 2, 100 do
    MyMath.is_prime(i)
end

profiler:disable()
profiler:report()
```

---

## 94.5 Memory Profiler

```lua
-- Memory allocation profiler
local MemoryProfiler = {}
MemoryProfiler.__index = MemoryProfiler

function MemoryProfiler.new()
    return setmetatable({
        snapshots = {},
        allocations_by_type = {}
    }, MemoryProfiler)
end

function MemoryProfiler:snapshot(label)
    collectgarbage("collect")
    
    local mem = collectgarbage("count")
    table.insert(self.snapshots, {
        label = label,
        kb = mem,
        time = os.clock()
    })
    return mem
end

function MemoryProfiler:profile_allocations(fn, label)
    -- Disable GC during measurement
    collectgarbage("stop")
    collectgarbage("collect")
    
    local before = collectgarbage("count")
    local before_time = os.clock()
    
    fn()
    
    local after = collectgarbage("count")
    local elapsed = os.clock() - before_time
    
    -- Resume GC
    collectgarbage("restart")
    
    local alloc_kb = after - before
    
    local result = {
        label = label or "unnamed",
        allocated_kb = alloc_kb,
        time_ms = elapsed * 1000
    }
    
    table.insert(self.allocations_by_type, result)
    return result
end

function MemoryProfiler:report()
    print("\n=== Memory Profile ===")
    
    -- Show snapshots
    if #self.snapshots > 0 then
        print("\nMemory Snapshots:")
        local baseline = self.snapshots[1].kb
        for i, s in ipairs(self.snapshots) do
            local delta = s.kb - (i > 1 and self.snapshots[i-1].kb or baseline)
            print(string.format("  %-30s %8.2f KB  (Δ %+.2f KB)", 
                s.label, s.kb, delta))
        end
    end
    
    -- Show allocations
    if #self.allocations_by_type > 0 then
        print("\nAllocation Profile:")
        table.sort(self.allocations_by_type, 
            function(a,b) return a.allocated_kb > b.allocated_kb end)
        
        for _, a in ipairs(self.allocations_by_type) do
            print(string.format("  %-30s %8.2f KB  %6.2f ms",
                a.label, a.allocated_kb, a.time_ms))
        end
    end
end

-- ตัวอย่างการใช้งาน
local mem_prof = MemoryProfiler.new()

mem_prof:snapshot("Start")

mem_prof:profile_allocations(function()
    -- Create many tables
    local tables = {}
    for i = 1, 10000 do
        tables[i] = { x=i, y=i*2 }
    end
end, "10k table allocation")

mem_prof:profile_allocations(function()
    -- String concatenation
    local s = ""
    for i = 1, 1000 do
        s = s .. string.rep("x", 10)
    end
end, "string concat 1k")

mem_prof:profile_allocations(function()
    -- Closure creation
    local closures = {}
    for i = 1, 5000 do
        local x = i
        closures[i] = function() return x end
    end
end, "5k closures")

mem_prof:snapshot("After allocations")

collectgarbage("collect")
mem_prof:snapshot("After GC")

mem_prof:report()
```

---

## 94.6 debug.sethook Profiler

```lua
-- Sampling profiler using debug.sethook
local SamplingProfiler = {}
SamplingProfiler.__index = SamplingProfiler

function SamplingProfiler.new()
    return setmetatable({
        samples = {},
        active = false
    }, SamplingProfiler)
end

function SamplingProfiler:start()
    self.active = true
    local profiler = self
    
    debug.sethook(function(event)
        if not profiler.active then return end
        
        local info = debug.getinfo(2, "Snl")
        if info then
            local key = string.format("%s:%s:%d",
                info.short_src or "?",
                info.name or "?",
                info.currentline or 0)
            
            profiler.samples[key] = (profiler.samples[key] or 0) + 1
        end
    end, "l", 1)  -- Hook every line
end

function SamplingProfiler:stop()
    self.active = false
    debug.sethook()
end

function SamplingProfiler:report(top_n)
    top_n = top_n or 20
    
    local sorted = {}
    local total = 0
    
    for key, count in pairs(self.samples) do
        total = total + count
        table.insert(sorted, { key=key, count=count })
    end
    
    table.sort(sorted, function(a, b) return a.count > b.count end)
    
    print(string.format("\n=== Sampling Profile (top %d) ===", top_n))
    print(string.format("Total samples: %d", total))
    print(string.format("%-60s %8s %8s", "Location", "Samples", "%"))
    print(string.rep("-", 80))
    
    for i = 1, math.min(top_n, #sorted) do
        local s = sorted[i]
        print(string.format("%-60s %8d %7.1f%%",
            s.key, s.count, s.count / total * 100))
    end
end

-- ตัวอย่าง
local sp = SamplingProfiler.new()
sp:start()

-- Code to profile
local function hot_path()
    local sum = 0
    for i = 1, 10000 do
        sum = sum + math.sqrt(i)
    end
    return sum
end

local function cold_path()
    return "rarely called"
end

for i = 1, 100 do
    hot_path()
    if i % 20 == 0 then cold_path() end
end

sp:stop()
sp:report(10)
```

---

## 94.7 Flamegraph Generation

```lua
-- Flamegraph data generation
-- Output format compatible with flamegraph.pl

local FlameGraph = {}
FlameGraph.__index = FlameGraph

function FlameGraph.new()
    return setmetatable({
        stacks = {},  -- "frame1;frame2;..." -> count
        call_stack = {}
    }, FlameGraph)
end

function FlameGraph:start()
    local fg = self
    
    debug.sethook(function(event)
        local info = debug.getinfo(2, "Sn")
        if not info then return end
        
        local frame = string.format("%s`%s",
            info.short_src or "?",
            info.name or "(anonymous)")
        
        if event == "call" then
            table.insert(fg.call_stack, frame)
        elseif event == "return" then
            -- Record current stack
            if #fg.call_stack > 0 then
                local stack_key = table.concat(fg.call_stack, ";")
                fg.stacks[stack_key] = (fg.stacks[stack_key] or 0) + 1
                table.remove(fg.call_stack)
            end
        end
    end, "cr")
end

function FlameGraph:stop()
    debug.sethook()
end

function FlameGraph:to_folded()
    local lines = {}
    for stack, count in pairs(self.stacks) do
        table.insert(lines, string.format("%s %d", stack, count))
    end
    table.sort(lines)
    return table.concat(lines, "\n")
end

function FlameGraph:save(filename)
    local f = io.open(filename, "w")
    if f then
        f:write(self:to_folded())
        f:close()
        print("Flamegraph data saved to: " .. filename)
        print("Generate with: flamegraph.pl " .. filename .. " > flame.svg")
    end
end

-- Quick profiling function
local function profile_with_flamegraph(fn, output_file)
    local fg = FlameGraph.new()
    fg:start()
    fn()
    fg:stop()
    
    if output_file then
        fg:save(output_file)
    else
        print("Flamegraph data (first 10 stacks):")
        local count = 0
        for stack, n in pairs(fg.stacks) do
            if count < 10 then
                print(string.format("  [%3d] %s", n, stack:sub(1, 80)))
                count = count + 1
            end
        end
    end
    
    return fg
end
```

---

## 94.8 Profiling Real-World Scenarios

```lua
-- Web server request profiling simulation
local RequestProfiler = {}
RequestProfiler.__index = RequestProfiler

function RequestProfiler.new()
    return setmetatable({
        requests = {},
        current = nil
    }, RequestProfiler)
end

function RequestProfiler:start_request(path, method)
    local req = {
        path = path,
        method = method,
        start_time = os.clock(),
        spans = {}
    }
    self.current = req
    table.insert(self.requests, req)
    return req
end

function RequestProfiler:span(name)
    if not self.current then return function() end end
    
    local span = {
        name = name,
        start = os.clock()
    }
    table.insert(self.current.spans, span)
    
    return function()
        span.duration = os.clock() - span.start
    end
end

function RequestProfiler:end_request()
    if self.current then
        self.current.duration = os.clock() - self.current.start_time
        self.current = nil
    end
end

function RequestProfiler:summary()
    local by_path = {}
    
    for _, req in ipairs(self.requests) do
        local key = req.method .. " " .. req.path
        local entry = by_path[key] or {count=0, total=0, max=0, min=math.huge}
        entry.count = entry.count + 1
        entry.total = entry.total + (req.duration or 0)
        entry.max = math.max(entry.max, req.duration or 0)
        entry.min = math.min(entry.min, req.duration or 0)
        by_path[key] = entry
    end
    
    print("\n=== Request Profile Summary ===")
    print(string.format("%-40s %6s %10s %10s %10s",
        "Endpoint", "Count", "Avg (ms)", "Min (ms)", "Max (ms)"))
    print(string.rep("-", 80))
    
    local sorted = {}
    for path, entry in pairs(by_path) do
        table.insert(sorted, { path=path, entry=entry })
    end
    table.sort(sorted, function(a, b)
        return (a.entry.total / a.entry.count) > (b.entry.total / b.entry.count)
    end)
    
    for _, item in ipairs(sorted) do
        local e = item.entry
        print(string.format("%-40s %6d %10.2f %10.2f %10.2f",
            item.path, e.count,
            e.total / e.count * 1000,
            e.min * 1000,
            e.max * 1000))
    end
end

-- Simulate requests
local rp = RequestProfiler.new()

local function simulate_db_query(ms)
    local start = os.clock()
    while os.clock() - start < ms / 1000 do end
end

for i = 1, 20 do
    -- GET /api/users
    rp:start_request("/api/users", "GET")
    local end_auth = rp:span("authenticate")
    simulate_db_query(1)
    end_auth()
    local end_query = rp:span("db_query")
    simulate_db_query(5 + math.random(10))
    end_query()
    rp:end_request()
    
    -- POST /api/users
    if i <= 5 then
        rp:start_request("/api/users", "POST")
        local end_valid = rp:span("validate")
        simulate_db_query(0.5)
        end_valid()
        local end_insert = rp:span("db_insert")
        simulate_db_query(10 + math.random(5))
        end_insert()
        rp:end_request()
    end
end

rp:summary()
```

---

## แบบฝึกหัด

1. **Benchmark Suite**: สร้าง benchmark suite สำหรับ data structure ต่างๆ
2. **Continuous Profiling**: สร้าง profiler ที่ทำงาน background ใน production
3. **Profile Comparison**: วัดผลก่อนและหลัง optimization
4. **Async Profiling**: Profile coroutine-based async code
5. **Memory Timeline**: สร้าง visualization ของ memory usage ตามเวลา

---

*ต่อไป: [Part 95 - Code Generation และ Macros](part-95.md)*
