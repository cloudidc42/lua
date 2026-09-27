# บทที่ 32: Testing ด้วย Busted

## บทนำ

การทดสอบซอฟต์แวร์ (Software Testing) เป็นส่วนสำคัญของการพัฒนาที่มีคุณภาพ **Busted** เป็น testing framework ยอดนิยมสำหรับ Lua ที่ได้รับแรงบันดาลใจจาก RSpec (Ruby) และ Jasmine (JavaScript) มีรูปแบบการเขียนที่อ่านง่ายและเข้าใจได้ตามธรรมชาติ

---

## 32.1 หลักการ TDD (Test-Driven Development)

TDD คือการพัฒนาซอฟต์แวร์โดยเขียน test ก่อนเขียน code จริง

### วงจร TDD: Red-Green-Refactor

```
1. Red   → เขียน test ที่ fail (code ยังไม่มี)
2. Green → เขียน code ให้ test ผ่าน
3. Refactor → ปรับปรุง code โดยไม่ให้ test fail
```

### ตัวอย่างที่ 1: TDD Workflow ง่ายๆ

```lua
-- Step 1: เขียน test (จะ fail เพราะยังไม่มี calculator.lua)
-- spec/calculator_spec.lua

local Calculator = require("calculator")

describe("Calculator", function()
    it("ควรบวกเลขได้ถูกต้อง", function()
        local calc = Calculator.new()
        assert.are.equal(5, calc:add(2, 3))
    end)
    
    it("ควรลบเลขได้ถูกต้อง", function()
        local calc = Calculator.new()
        assert.are.equal(1, calc:subtract(3, 2))
    end)
end)
```

```lua
-- Step 2: เขียน code ให้ test ผ่าน
-- calculator.lua

local Calculator = {}
Calculator.__index = Calculator

function Calculator.new()
    return setmetatable({}, Calculator)
end

function Calculator:add(a, b)
    return a + b
end

function Calculator:subtract(a, b)
    return a - b
end

return Calculator
```

---

## 32.2 ติดตั้งและเริ่มต้น Busted

```bash
# ติดตั้ง busted
luarocks install busted

# ตรวจสอบการติดตั้ง
busted --version

# รัน tests
busted

# รัน tests แบบ verbose
busted --verbose

# รัน test file เฉพาะ
busted spec/calculator_spec.lua
```

### โครงสร้าง Project ที่แนะนำ

```
myproject/
├── src/
│   ├── calculator.lua
│   └── utils.lua
├── spec/
│   ├── calculator_spec.lua
│   └── utils_spec.lua
└── .busted          (configuration file)
```

```lua
-- .busted configuration
return {
    default = {
        verbose = true,
        coverage = false,
        ["auto-insulate"] = true,
        lua = "lua",
        pattern = "_spec",  -- pattern สำหรับ test files
    }
}
```

---

## 32.3 describe() และ it()

`describe()` ใช้จัดกลุ่ม tests และ `it()` ใช้เขียน test แต่ละตัว

### ตัวอย่างที่ 2: describe และ it พื้นฐาน

```lua
-- spec/basic_spec.lua

describe("ทดสอบ string operations", function()
    
    it("ควร concatenate strings", function()
        local result = "Hello" .. " " .. "World"
        assert.are.equal("Hello World", result)
    end)
    
    it("ควร convert to uppercase", function()
        local result = string.upper("hello")
        assert.are.equal("HELLO", result)
    end)
    
    it("ควร find substring", function()
        local str = "Hello, World!"
        local pos = string.find(str, "World")
        assert.is_not_nil(pos)
        assert.are.equal(8, pos)
    end)
    
    it("ควร measure string length", function()
        assert.are.equal(5, #"Hello")
        assert.are.equal(0, #"")
    end)
    
end)
```

### ตัวอย่างที่ 3: Nested describe

```lua
-- spec/nested_spec.lua

describe("ทดสอบ table operations", function()
    
    describe("array operations", function()
        
        it("ควร insert elements", function()
            local t = {1, 2, 3}
            table.insert(t, 4)
            assert.are.equal(4, #t)
            assert.are.equal(4, t[4])
        end)
        
        it("ควร remove elements", function()
            local t = {1, 2, 3, 4, 5}
            table.remove(t, 3)
            assert.are.equal(4, #t)
            assert.are.equal(4, t[3])
        end)
        
        it("ควร sort elements", function()
            local t = {3, 1, 4, 1, 5, 9, 2, 6}
            table.sort(t)
            assert.are.equal(1, t[1])
            assert.are.equal(9, t[#t])
        end)
        
    end)
    
    describe("hash table operations", function()
        
        it("ควร set and get values", function()
            local t = {}
            t["key"] = "value"
            assert.are.equal("value", t["key"])
        end)
        
        it("ควร count keys", function()
            local t = {a = 1, b = 2, c = 3}
            local count = 0
            for _ in pairs(t) do
                count = count + 1
            end
            assert.are.equal(3, count)
        end)
        
    end)
    
end)
```

---

## 32.4 before_each() และ after_each()

### ตัวอย่างที่ 4: Setup และ Teardown

```lua
-- spec/database_spec.lua

describe("ทดสอบ Database Module", function()
    local db
    local testData
    
    -- รันก่อนแต่ละ test
    before_each(function()
        -- สร้าง mock database ใหม่สำหรับแต่ละ test
        db = {
            records = {},
            nextId = 1
        }
        
        testData = {
            {name = "Alice", age = 30},
            {name = "Bob", age = 25},
            {name = "Charlie", age = 35}
        }
    end)
    
    -- รันหลังแต่ละ test
    after_each(function()
        -- ล้างข้อมูล
        db = nil
        testData = nil
    end)
    
    it("ควร insert record", function()
        local record = {name = "Test", age = 20}
        db.records[db.nextId] = record
        db.nextId = db.nextId + 1
        
        assert.are.equal(1, #db.records)
    end)
    
    it("ควร insert หลาย records", function()
        for _, data in ipairs(testData) do
            db.records[db.nextId] = data
            db.nextId = db.nextId + 1
        end
        
        assert.are.equal(3, db.nextId - 1)
    end)
    
    it("ควรเริ่มต้น db ใหม่ในแต่ละ test", function()
        -- db ควรว่างเปล่า เพราะ before_each reset ไว้
        assert.are.equal(1, db.nextId)
        assert.are.equal(0, #db.records)
    end)
    
end)
```

### ตัวอย่างที่ 5: before_all และ after_all

```lua
-- spec/resource_spec.lua

describe("ทดสอบที่ต้องการ Resource ร่วม", function()
    local sharedResource
    local setupCount = 0
    
    -- รันครั้งเดียวก่อน tests ทั้งหมด
    setup(function()
        sharedResource = {
            connection = "fake_connection",
            isOpen = true
        }
        print("เปิด shared resource แล้ว")
    end)
    
    -- รันครั้งเดียวหลัง tests ทั้งหมด
    teardown(function()
        sharedResource.isOpen = false
        sharedResource = nil
        print("ปิด shared resource แล้ว")
    end)
    
    before_each(function()
        setupCount = setupCount + 1
    end)
    
    it("test 1 - ใช้ shared resource", function()
        assert.is_true(sharedResource.isOpen)
        assert.are.equal("fake_connection", sharedResource.connection)
    end)
    
    it("test 2 - ใช้ shared resource", function()
        assert.is_true(sharedResource.isOpen)
    end)
    
    it("test 3 - ตรวจ setup count", function()
        assert.are.equal(3, setupCount)
    end)
    
end)
```

---

## 32.5 Assertions

### ตัวอย่างที่ 6: assert พื้นฐาน

```lua
-- spec/assertions_spec.lua

describe("ทดสอบ Assertions ต่างๆ", function()
    
    it("equality assertions", function()
        -- เปรียบเทียบค่า
        assert.are.equal(42, 42)
        assert.are.equal("hello", "hello")
        assert.are.equal(true, true)
        
        -- ไม่เท่ากัน
        assert.are_not.equal(1, 2)
        assert.are_not.equal("foo", "bar")
    end)
    
    it("truthy และ falsy assertions", function()
        -- truthy: ค่าที่ไม่ใช่ false และ nil
        assert.is_truthy(1)
        assert.is_truthy("string")
        assert.is_truthy({})
        assert.is_truthy(true)
        
        -- falsy: nil และ false
        assert.is_falsy(false)
        assert.is_falsy(nil)
    end)
    
    it("nil assertions", function()
        local x = nil
        local y = 5
        
        assert.is_nil(x)
        assert.is_not_nil(y)
    end)
    
    it("boolean assertions", function()
        assert.is_true(true)
        assert.is_false(false)
        
        -- เหล่านี้จะ fail:
        -- assert.is_true(1)   -- 1 ไม่ใช่ true
        -- assert.is_false(nil)  -- nil ไม่ใช่ false
    end)
    
end)
```

### ตัวอย่างที่ 7: assert.are.same สำหรับ Tables

```lua
-- spec/table_assertions_spec.lua

describe("ทดสอบ Table Assertions", function()
    
    it("are.equal เปรียบเทียบ reference", function()
        local t = {1, 2, 3}
        local t2 = t  -- reference เดียวกัน
        
        assert.are.equal(t, t2)  -- ผ่าน (reference เดียวกัน)
    end)
    
    it("are.same เปรียบเทียบ content", function()
        local t1 = {1, 2, 3}
        local t2 = {1, 2, 3}  -- table ต่างกัน แต่ content เหมือนกัน
        
        -- are.equal จะ fail เพราะเป็น object ต่างกัน
        -- assert.are.equal(t1, t2)  -- FAIL!
        
        -- are.same จะ pass เพราะ content เหมือนกัน
        assert.are.same(t1, t2)  -- PASS!
    end)
    
    it("same กับ nested tables", function()
        local t1 = {
            name = "Alice",
            scores = {90, 85, 92},
            address = {
                city = "Bangkok",
                country = "Thailand"
            }
        }
        
        local t2 = {
            name = "Alice",
            scores = {90, 85, 92},
            address = {
                city = "Bangkok",
                country = "Thailand"
            }
        }
        
        assert.are.same(t1, t2)
    end)
    
    it("ตรวจสอบ table ไม่เหมือนกัน", function()
        local t1 = {1, 2, 3}
        local t2 = {1, 2, 4}  -- ต่างกันที่ index 3
        
        assert.are_not.same(t1, t2)
    end)
    
end)
```

### ตัวอย่างที่ 8: assert.has_error

```lua
-- spec/error_spec.lua

describe("ทดสอบ Error Handling", function()
    
    local function divideNumbers(a, b)
        if b == 0 then
            error("ไม่สามารถหารด้วยศูนย์ได้!")
        end
        return a / b
    end
    
    local function validateAge(age)
        if type(age) ~= "number" then
            error("age ต้องเป็นตัวเลข", 2)
        end
        if age < 0 or age > 150 then
            error("age ต้องอยู่ระหว่าง 0-150", 2)
        end
        return true
    end
    
    it("ควร throw error เมื่อหารด้วยศูนย์", function()
        assert.has_error(function()
            divideNumbers(10, 0)
        end)
    end)
    
    it("ควร throw error พร้อม message เฉพาะ", function()
        assert.has_error(function()
            divideNumbers(5, 0)
        end, "ไม่สามารถหารด้วยศูนย์ได้!")
    end)
    
    it("ไม่ควร throw error เมื่อหารปกติ", function()
        assert.has_no.errors(function()
            divideNumbers(10, 2)
        end)
    end)
    
    it("ควร throw error เมื่อ age ไม่ถูกต้อง", function()
        assert.has_error(function()
            validateAge("ยี่สิบห้า")
        end)
        
        assert.has_error(function()
            validateAge(-1)
        end)
        
        assert.has_error(function()
            validateAge(200)
        end)
    end)
    
    it("ไม่ควร throw error เมื่อ age ถูกต้อง", function()
        assert.has_no.errors(function()
            validateAge(25)
        end)
    end)
    
end)
```

---

## 32.6 Spies และ Stubs

### ตัวอย่างที่ 9: spy.on() - ติดตามการเรียก function

```lua
-- spec/spy_spec.lua

describe("ทดสอบ Spies", function()
    
    it("spy ติดตามการเรียก function", function()
        local myObj = {
            greet = function(self, name)
                return "Hello, " .. name
            end
        }
        
        -- ติดตาม greet method
        spy.on(myObj, "greet")
        
        -- เรียก function
        myObj:greet("Alice")
        myObj:greet("Bob")
        
        -- ตรวจสอบว่าถูกเรียกกี่ครั้ง
        assert.spy(myObj.greet).was.called(2)
        
        -- ตรวจสอบว่าถูกเรียกด้วย argument อะไร
        assert.spy(myObj.greet).was.called_with(myObj, "Alice")
        assert.spy(myObj.greet).was.called_with(myObj, "Bob")
    end)
    
    it("spy ตรวจสอบว่าไม่ถูกเรียก", function()
        local myObj = {
            doSomething = function(self)
                return "done"
            end
        }
        
        spy.on(myObj, "doSomething")
        
        -- ไม่ได้เรียก doSomething
        
        assert.spy(myObj.doSomething).was_not.called()
    end)
    
    it("spy บน standalone function", function()
        local logger = {
            log = function(msg)
                -- log to file or console
            end
        }
        
        spy.on(logger, "log")
        
        -- ทำ operation ที่ควรเรียก log
        local function processData(data, lgr)
            lgr.log("Processing: " .. tostring(data))
            return data * 2
        end
        
        local result = processData(5, logger)
        
        assert.are.equal(10, result)
        assert.spy(logger.log).was.called(1)
        assert.spy(logger.log).was.called_with("Processing: 5")
    end)
    
end)
```

### ตัวอย่างที่ 10: Stubs - แทนที่ Function

```lua
-- spec/stub_spec.lua

describe("ทดสอบ Stubs", function()
    
    it("stub แทนที่ function ด้วย fake implementation", function()
        local database = {
            getUser = function(id)
                -- ปกติจะเรียก database จริง
                return nil  -- ไม่มี database จริง
            end
        }
        
        -- แทนที่ด้วย stub ที่ return ค่าที่กำหนด
        stub(database, "getUser").returns({
            id = 1,
            name = "Test User",
            email = "test@example.com"
        })
        
        -- เรียก stub
        local user = database.getUser(1)
        
        -- ตรวจสอบผลลัพธ์
        assert.is_not_nil(user)
        assert.are.equal("Test User", user.name)
        assert.are.equal("test@example.com", user.email)
        
        -- ตรวจสอบว่าถูกเรียก
        assert.stub(database.getUser).was.called_with(1)
    end)
    
    it("stub ด้วย function", function()
        local api = {
            fetchData = function(url)
                -- ปกติจะเรียก HTTP request
                return nil
            end
        }
        
        -- stub ด้วย function
        stub(api, "fetchData").invokes(function(url)
            if url:find("users") then
                return {status = 200, body = '{"users": []}'}
            else
                return {status = 404, body = "Not Found"}
            end
        end)
        
        local response1 = api.fetchData("https://api.example.com/users")
        local response2 = api.fetchData("https://api.example.com/other")
        
        assert.are.equal(200, response1.status)
        assert.are.equal(404, response2.status)
    end)
    
end)
```

### ตัวอย่างที่ 11: Mock Objects

```lua
-- spec/mock_spec.lua

describe("ทดสอบ Mock Objects", function()
    
    it("mock object ครบชุด", function()
        -- สร้าง mock email service
        local emailService = mock({
            send = function(to, subject, body)
                return true
            end,
            validate = function(email)
                return email:find("@") ~= nil
            end
        })
        
        -- ใช้ mock ใน code ที่ test
        local function registerUser(user, emailSvc)
            if not emailSvc.validate(user.email) then
                return false, "Invalid email"
            end
            
            emailSvc.send(
                user.email,
                "Welcome!",
                "ยินดีต้อนรับสู่ระบบ"
            )
            
            return true
        end
        
        local ok, err = registerUser({
            name = "Alice",
            email = "alice@example.com"
        }, emailService)
        
        assert.is_true(ok)
        assert.stub(emailService.send).was.called(1)
        assert.stub(emailService.validate).was.called(1)
    end)
    
end)
```

---

## 32.7 Test Organization

### ตัวอย่างที่ 12: File Structure ขนาดใหญ่

```lua
-- spec/user_spec.lua

local User = require("src.user")

describe("User Module", function()
    
    local validUserData
    
    before_each(function()
        validUserData = {
            username = "alice",
            email = "alice@example.com",
            age = 25,
            password = "secret123"
        }
    end)
    
    describe("User.new()", function()
        
        it("ควรสร้าง user ได้สำเร็จ", function()
            local user = User.new(validUserData)
            assert.is_not_nil(user)
            assert.are.equal("alice", user.username)
        end)
        
        it("ควร error เมื่อไม่มี username", function()
            validUserData.username = nil
            assert.has_error(function()
                User.new(validUserData)
            end)
        end)
        
        it("ควร error เมื่อ email ไม่ถูกต้อง", function()
            validUserData.email = "invalid-email"
            assert.has_error(function()
                User.new(validUserData)
            end)
        end)
        
    end)
    
    describe("user:authenticate()", function()
        local user
        
        before_each(function()
            user = User.new(validUserData)
        end)
        
        it("ควร authenticate ด้วย password ถูกต้อง", function()
            assert.is_true(user:authenticate("secret123"))
        end)
        
        it("ควร fail เมื่อ password ผิด", function()
            assert.is_false(user:authenticate("wrongpassword"))
        end)
        
        it("ควร fail เมื่อ password ว่าง", function()
            assert.is_false(user:authenticate(""))
        end)
        
    end)
    
    describe("user:updateProfile()", function()
        local user
        
        before_each(function()
            user = User.new(validUserData)
        end)
        
        it("ควรอัปเดต email ได้", function()
            user:updateProfile({email = "newemail@example.com"})
            assert.are.equal("newemail@example.com", user.email)
        end)
        
        it("ควร error เมื่ออัปเดต email ไม่ถูกต้อง", function()
            assert.has_error(function()
                user:updateProfile({email = "notvalid"})
            end)
        end)
        
    end)
    
end)
```

---

## 32.8 Fixtures

### ตัวอย่างที่ 13: ใช้ Fixtures

```lua
-- spec/fixtures/users.lua
-- Fixture file สำหรับ test data

return {
    alice = {
        id = 1,
        username = "alice",
        email = "alice@example.com",
        age = 30,
        role = "admin"
    },
    bob = {
        id = 2,
        username = "bob",
        email = "bob@example.com",
        age = 25,
        role = "user"
    },
    charlie = {
        id = 3,
        username = "charlie",
        email = "charlie@example.com",
        age = 35,
        role = "user"
    }
}
```

```lua
-- spec/user_fixture_spec.lua

local fixtures = require("spec.fixtures.users")

describe("ทดสอบด้วย Fixtures", function()
    
    it("ตรวจสอบ admin user", function()
        local alice = fixtures.alice
        assert.are.equal("admin", alice.role)
        assert.are.equal(1, alice.id)
    end)
    
    it("ตรวจสอบ regular users", function()
        assert.are.equal("user", fixtures.bob.role)
        assert.are.equal("user", fixtures.charlie.role)
    end)
    
    it("users ทั้งหมดมี email", function()
        for name, user in pairs(fixtures) do
            assert.is_not_nil(user.email, 
                name .. " ควรมี email")
            assert.is_truthy(user.email:find("@"),
                name .. " ควรมี @ ใน email")
        end
    end)
    
end)
```

---

## 32.9 Parametric Tests

### ตัวอย่างที่ 14: Test ด้วยหลาย Parameters

```lua
-- spec/parametric_spec.lua

describe("ทดสอบแบบ Parametric", function()
    
    -- ชุดข้อมูลทดสอบ
    local testCases = {
        {input = 0, expected = true,  desc = "ศูนย์เป็นจำนวนคู่"},
        {input = 2, expected = true,  desc = "2 เป็นจำนวนคู่"},
        {input = 4, expected = true,  desc = "4 เป็นจำนวนคู่"},
        {input = 1, expected = false, desc = "1 ไม่ใช่จำนวนคู่"},
        {input = 3, expected = false, desc = "3 ไม่ใช่จำนวนคู่"},
        {input = -2, expected = true, desc = "-2 เป็นจำนวนคู่"},
        {input = -3, expected = false, desc = "-3 ไม่ใช่จำนวนคู่"},
    }
    
    for _, case in ipairs(testCases) do
        it(case.desc, function()
            local isEven = case.input % 2 == 0
            assert.are.equal(case.expected, isEven)
        end)
    end
    
end)
```

### ตัวอย่างที่ 15: Parametric Tests สำหรับ String Functions

```lua
-- spec/string_parametric_spec.lua

describe("ทดสอบ String Functions แบบ Parametric", function()
    
    -- ทดสอบ string.upper
    local upperCases = {
        {"hello", "HELLO"},
        {"world", "WORLD"},
        {"lua", "LUA"},
        {"mixed CASE", "MIXED CASE"},
        {"", ""},
    }
    
    describe("string.upper()", function()
        for _, case in ipairs(upperCases) do
            local input, expected = case[1], case[2]
            it(string.format('upper("%s") == "%s"', input, expected), function()
                assert.are.equal(expected, string.upper(input))
            end)
        end
    end)
    
    -- ทดสอบ string.len
    local lenCases = {
        {"", 0},
        {"a", 1},
        {"hello", 5},
        {"สวัสดี", 18},  -- UTF-8 bytes
    }
    
    describe("string.len()", function()
        for _, case in ipairs(lenCases) do
            local input, expected = case[1], case[2]
            it(string.format('len("%s") == %d', input, expected), function()
                assert.are.equal(expected, #input)
            end)
        end
    end)
    
end)
```

---

## 32.10 Async Tests

### ตัวอย่างที่ 16: Async Tests ด้วย Busted

```lua
-- spec/async_spec.lua

describe("ทดสอบ Async Operations", function()
    
    -- Busted รองรับ async tests แบบง่ายๆ
    it("ทดสอบ operation ที่ใช้เวลา", function()
        local completed = false
        
        -- simulate async operation
        local function asyncOperation(callback)
            -- ใน Lua ปกติไม่มี real async
            -- นี่คือ simulation
            local result = {status = "done", data = 42}
            callback(result)
        end
        
        asyncOperation(function(result)
            completed = true
            assert.are.equal("done", result.status)
            assert.are.equal(42, result.data)
        end)
        
        assert.is_true(completed)
    end)
    
    it("ทดสอบ coroutine-based async", function()
        local results = {}
        
        -- สร้าง coroutine-based tasks
        local function task(id, callback)
            return coroutine.create(function()
                -- simulate work
                local value = id * 10
                coroutine.yield()
                callback(value)
            end)
        end
        
        local tasks = {}
        for i = 1, 3 do
            local co = task(i, function(value)
                table.insert(results, value)
            end)
            table.insert(tasks, co)
        end
        
        -- run tasks
        for _, co in ipairs(tasks) do
            coroutine.resume(co)
        end
        
        for _, co in ipairs(tasks) do
            coroutine.resume(co)
        end
        
        -- ตรวจสอบผลลัพธ์
        assert.are.equal(3, #results)
        table.sort(results)
        assert.are.same({10, 20, 30}, results)
    end)
    
end)
```

---

## 32.11 Test Coverage

### ตัวอย่างที่ 17: รัน Test Coverage

```bash
# ติดตั้ง luacov
luarocks install luacov

# รัน tests พร้อม coverage
busted --coverage

# หรือผ่าน luacov โดยตรง
lua -lluacov src/mymodule.lua

# ดูรายงาน coverage
luacov

# เปิดไฟล์ luacov.report.out
cat luacov.report.out
```

```lua
-- .luacov configuration
return {
    -- ไฟล์ที่ต้องการ track
    include = {
        "src/.*",
    },
    
    -- ไฟล์ที่ไม่ต้องการ track
    exclude = {
        "spec/.*",
        "test_.*",
    },
    
    -- threshold ขั้นต่ำ (%)
    threshold = 80,
}
```

---

## 32.12 Custom Assertions

### ตัวอย่างที่ 18: เพิ่ม Custom Assertions

```lua
-- spec/helpers/custom_assertions.lua

-- เพิ่ม custom assertions ใน busted
local assert = require("luassert")
local say = require("say")

-- custom assertion: ตรวจสอบว่า table มี key
say:set("assertion.has_key.positive", "Expected table to have key '%s'")
say:set("assertion.has_key.negative", "Expected table to NOT have key '%s'")

local function hasKey(state, arguments)
    local t = arguments[1]
    local key = arguments[2]
    return t[key] ~= nil
end

assert:register(
    "assertion",
    "has_key",
    hasKey,
    "assertion.has_key.positive",
    "assertion.has_key.negative"
)

-- custom assertion: ตรวจสอบว่า number อยู่ใน range
say:set("assertion.in_range.positive", 
    "Expected %s to be between %s and %s")
say:set("assertion.in_range.negative", 
    "Expected %s to NOT be between %s and %s")

local function inRange(state, arguments)
    local value = arguments[1]
    local min = arguments[2]
    local max = arguments[3]
    return value >= min and value <= max
end

assert:register(
    "assertion",
    "in_range",
    inRange,
    "assertion.in_range.positive",
    "assertion.in_range.negative"
)
```

```lua
-- spec/custom_assertion_spec.lua
require("spec.helpers.custom_assertions")

describe("ใช้ Custom Assertions", function()
    
    it("has_key assertion", function()
        local config = {
            host = "localhost",
            port = 8080,
            debug = true
        }
        
        assert.has_key(config, "host")
        assert.has_key(config, "port")
        assert.not_has_key(config, "missing_key")
    end)
    
    it("in_range assertion", function()
        local score = 85
        assert.in_range(score, 0, 100)
        
        local temperature = 36.5
        assert.in_range(temperature, 36.0, 37.5)
    end)
    
end)
```

---

## 32.13 ตัวอย่าง Test Suite สมบูรณ์

### ตัวอย่างที่ 19: Test Suite สำหรับ Shopping Cart

```lua
-- src/cart.lua
local Cart = {}
Cart.__index = Cart

function Cart.new()
    return setmetatable({
        items = {},
        discount = 0
    }, Cart)
end

function Cart:addItem(product, quantity)
    assert(type(product) == "table", "product ต้องเป็น table")
    assert(type(quantity) == "number" and quantity > 0, 
        "quantity ต้องเป็นจำนวนบวก")
    
    local existing = self.items[product.id]
    if existing then
        existing.quantity = existing.quantity + quantity
    else
        self.items[product.id] = {
            product = product,
            quantity = quantity
        }
    end
end

function Cart:removeItem(productId)
    if not self.items[productId] then
        error("ไม่พบสินค้า id: " .. tostring(productId))
    end
    self.items[productId] = nil
end

function Cart:setDiscount(percent)
    assert(percent >= 0 and percent <= 100, 
        "discount ต้องอยู่ระหว่าง 0-100")
    self.discount = percent
end

function Cart:getTotal()
    local total = 0
    for _, item in pairs(self.items) do
        total = total + (item.product.price * item.quantity)
    end
    return total * (1 - self.discount / 100)
end

function Cart:getItemCount()
    local count = 0
    for _, item in pairs(self.items) do
        count = count + item.quantity
    end
    return count
end

function Cart:clear()
    self.items = {}
    self.discount = 0
end

return Cart
```

```lua
-- spec/cart_spec.lua
local Cart = require("src.cart")

describe("Shopping Cart", function()
    
    local cart
    local apple = {id = 1, name = "Apple", price = 10}
    local banana = {id = 2, name = "Banana", price = 5}
    local orange = {id = 3, name = "Orange", price = 8}
    
    before_each(function()
        cart = Cart.new()
    end)
    
    describe("Cart.new()", function()
        it("ควรสร้าง cart ว่างเปล่า", function()
            assert.are.equal(0, cart:getItemCount())
            assert.are.equal(0, cart:getTotal())
        end)
    end)
    
    describe("cart:addItem()", function()
        
        it("ควรเพิ่มสินค้าได้", function()
            cart:addItem(apple, 3)
            assert.are.equal(3, cart:getItemCount())
        end)
        
        it("ควรเพิ่มสินค้าหลายรายการ", function()
            cart:addItem(apple, 2)
            cart:addItem(banana, 5)
            assert.are.equal(7, cart:getItemCount())
        end)
        
        it("ควรรวม quantity เมื่อเพิ่มสินค้าเดิม", function()
            cart:addItem(apple, 2)
            cart:addItem(apple, 3)
            assert.are.equal(5, cart:getItemCount())
        end)
        
        it("ควร error เมื่อ product ไม่ถูกต้อง", function()
            assert.has_error(function()
                cart:addItem("not a table", 1)
            end)
        end)
        
        it("ควร error เมื่อ quantity <= 0", function()
            assert.has_error(function()
                cart:addItem(apple, 0)
            end)
            assert.has_error(function()
                cart:addItem(apple, -1)
            end)
        end)
        
    end)
    
    describe("cart:removeItem()", function()
        
        before_each(function()
            cart:addItem(apple, 2)
            cart:addItem(banana, 3)
        end)
        
        it("ควรลบสินค้าได้", function()
            cart:removeItem(apple.id)
            assert.are.equal(3, cart:getItemCount())
        end)
        
        it("ควร error เมื่อลบสินค้าที่ไม่มี", function()
            assert.has_error(function()
                cart:removeItem(999)
            end)
        end)
        
    end)
    
    describe("cart:getTotal()", function()
        
        it("ควรคำนวณยอดรวมถูกต้อง", function()
            cart:addItem(apple, 2)   -- 10 * 2 = 20
            cart:addItem(banana, 3)  -- 5 * 3 = 15
            assert.are.equal(35, cart:getTotal())
        end)
        
        it("ควรคำนวณยอดรวมพร้อม discount", function()
            cart:addItem(apple, 10)  -- 10 * 10 = 100
            cart:setDiscount(20)    -- 20% off
            assert.are.equal(80, cart:getTotal())
        end)
        
        it("ควร return 0 เมื่อ cart ว่าง", function()
            assert.are.equal(0, cart:getTotal())
        end)
        
    end)
    
    describe("cart:setDiscount()", function()
        
        it("ควร set discount ได้", function()
            cart:setDiscount(10)
            cart:addItem(apple, 10)  -- 100 total, 10% off = 90
            assert.are.equal(90, cart:getTotal())
        end)
        
        it("ควร error เมื่อ discount < 0", function()
            assert.has_error(function()
                cart:setDiscount(-5)
            end)
        end)
        
        it("ควร error เมื่อ discount > 100", function()
            assert.has_error(function()
                cart:setDiscount(101)
            end)
        end)
        
        it("ควรรับ discount 100% ได้ (ฟรี!)", function()
            cart:addItem(apple, 5)
            cart:setDiscount(100)
            assert.are.equal(0, cart:getTotal())
        end)
        
    end)
    
    describe("cart:clear()", function()
        
        it("ควรล้าง cart ได้", function()
            cart:addItem(apple, 5)
            cart:addItem(banana, 3)
            cart:setDiscount(10)
            
            cart:clear()
            
            assert.are.equal(0, cart:getItemCount())
            assert.are.equal(0, cart:getTotal())
        end)
        
    end)
    
end)
```

---

## 32.14 การรันและดู Output

### ตัวอย่างที่ 20: รัน Tests แบบต่างๆ

```bash
# รัน tests ทั้งหมด
busted

# รัน พร้อม verbose output
busted --verbose

# รัน test file เฉพาะ
busted spec/cart_spec.lua

# รัน test ที่มี pattern ในชื่อ
busted --pattern "cart"

# รัน พร้อม TAP output (สำหรับ CI)
busted --output tap

# รัน พร้อม JUnit XML output
busted --output junit

# รัน พร้อม coverage
busted --coverage

# รัน แบบ shuffle tests
busted --shuffle
```

### ตัวอย่างที่ 21: ตัวอย่าง Output

```
ทดสอบ Shopping Cart
  Cart.new()
    ✓ ควรสร้าง cart ว่างเปล่า
  cart:addItem()
    ✓ ควรเพิ่มสินค้าได้
    ✓ ควรเพิ่มสินค้าหลายรายการ
    ✓ ควรรวม quantity เมื่อเพิ่มสินค้าเดิม
    ✓ ควร error เมื่อ product ไม่ถูกต้อง
    ✓ ควร error เมื่อ quantity <= 0
  ...

12 successes / 0 failures / 0 errors / 0 pending
```

---

## 32.15 Test Coverage กับ LuaCov

### ตัวอย่างที่ 22: ตั้งค่า Coverage

```lua
-- .busted
return {
    default = {
        coverage = true,
        verbose = true,
    }
}
```

```lua
-- luacov.stats.out (ตัวอย่าง output)
-- luacov.report.out

--[[
File         Hits  Missed  Coverage
---------------------------------------
src/cart.lua   45     5    90.00%
src/user.lua   30     2    93.75%
--]]
```

---

## 32.16 Pending Tests

### ตัวอย่างที่ 23: Pending Tests

```lua
-- spec/pending_spec.lua

describe("Features ที่ยังไม่ implement", function()
    
    -- pending test - ยังไม่มี implementation
    pending("ควรรองรับ OAuth login")
    
    pending("ควรส่ง email notification", function()
        -- TODO: implement email service
    end)
    
    it("feature ที่ implement แล้ว", function()
        assert.is_true(true)
    end)
    
end)
```

---

## 32.17 Integration Test

### ตัวอย่างที่ 24: Integration Test

```lua
-- spec/integration/api_spec.lua

-- ทดสอบ integration ระหว่าง modules
local Cart = require("src.cart")
local Inventory = require("src.inventory")
local Order = require("src.order")

describe("Integration: Cart + Inventory + Order", function()
    
    local inventory, cart
    
    before_each(function()
        -- setup inventory
        inventory = Inventory.new({
            {id = 1, name = "Widget", price = 25, stock = 100},
            {id = 2, name = "Gadget", price = 50, stock = 50},
        })
        
        cart = Cart.new()
    end)
    
    it("ควร checkout และ update inventory", function()
        -- เพิ่มสินค้าใน cart
        local widget = inventory:getProduct(1)
        cart:addItem(widget, 3)
        
        -- ตรวจสอบ stock ก่อน checkout
        assert.are.equal(100, inventory:getStock(1))
        
        -- สร้าง order จาก cart
        local order = Order.fromCart(cart, inventory)
        
        -- ตรวจสอบ stock หลัง checkout
        assert.are.equal(97, inventory:getStock(1))
        
        -- ตรวจสอบ order
        assert.is_not_nil(order)
        assert.are.equal(75, order:getTotal())  -- 25 * 3
    end)
    
end)
```

---

## สรุปบทที่ 32

Busted เป็น testing framework ที่ทรงพลังและใช้งานง่าย:

| Function | การใช้งาน |
|----------|----------|
| `describe()` | จัดกลุ่ม tests |
| `it()` | กำหนด test case |
| `before_each()` | รันก่อนแต่ละ test |
| `after_each()` | รันหลังแต่ละ test |
| `setup()` | รันครั้งเดียวก่อน group |
| `teardown()` | รันครั้งเดียวหลัง group |
| `pending()` | mark test เป็น pending |

| Assertion | การใช้งาน |
|-----------|----------|
| `assert.are.equal` | เปรียบเทียบค่าเท่ากัน |
| `assert.are.same` | เปรียบเทียบ table content |
| `assert.is_truthy` | ค่าเป็น truthy |
| `assert.is_falsy` | ค่าเป็น falsy |
| `assert.is_nil` | ค่าเป็น nil |
| `assert.is_true` | ค่าเป็น true |
| `assert.is_false` | ค่าเป็น false |
| `assert.has_error` | function ควร throw error |

---

*บทต่อไป: บทที่ 33 - Testing ด้วย LuaUnit*
