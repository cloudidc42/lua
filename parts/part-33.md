# บทที่ 33: Testing ด้วย LuaUnit

## บทนำ

**LuaUnit** เป็น testing framework อีกตัวเลือกหนึ่งสำหรับ Lua ได้รับแรงบันดาลใจจาก xUnit (JUnit, NUnit) ซึ่งเป็น pattern ที่นิยมในโลกของ Java และ .NET โดยใช้รูปแบบ class-based testing ต่างจาก Busted ที่ใช้ BDD style

LuaUnit เหมาะสำหรับโปรเจกต์ที่:
- ต้องการ JUnit XML output สำหรับ CI/CD
- ชอบรูปแบบการเขียนแบบ OOP
- ต้องการ integration กับ tools เดิมที่รองรับ xUnit

---

## 33.1 ติดตั้ง LuaUnit

```bash
# ติดตั้งผ่าน LuaRocks
luarocks install luaunit

# ตรวจสอบการติดตั้ง
lua -e "require('luaunit'); print('LuaUnit installed!')"

# ดู version
lua -e "local lu = require('luaunit'); print(lu.VERSION)"
```

---

## 33.2 โครงสร้างพื้นฐาน

### ตัวอย่างที่ 1: Test พื้นฐาน

```lua
-- test_basic.lua

-- นำเข้า LuaUnit
local luaunit = require("luaunit")

-- TestCase class
TestBasic = {}

-- test methods ต้องขึ้นต้นด้วย "test" (case insensitive)
function TestBasic:testAddition()
    luaunit.assertEquals(4, 2 + 2)
end

function TestBasic:testSubtraction()
    luaunit.assertEquals(1, 3 - 2)
end

function TestBasic:testMultiplication()
    luaunit.assertEquals(6, 2 * 3)
end

function TestBasic:testDivision()
    luaunit.assertAlmostEquals(3.333, 10 / 3, 0.001)
end

-- รัน tests
os.exit(luaunit.LuaUnit.run())
```

```bash
# รัน test
lua test_basic.lua
```

---

## 33.3 assertEquals และ assertNotEquals

### ตัวอย่างที่ 2: Equality Assertions

```lua
-- test_equality.lua
local luaunit = require("luaunit")

TestEquality = {}

function TestEquality:testNumbers()
    luaunit.assertEquals(42, 42)
    luaunit.assertEquals(3.14, 3.14)
    luaunit.assertEquals(-1, -1)
end

function TestEquality:testStrings()
    luaunit.assertEquals("hello", "hello")
    luaunit.assertEquals("", "")
    luaunit.assertEquals("สวัสดี", "สวัสดี")
end

function TestEquality:testBooleans()
    luaunit.assertEquals(true, true)
    luaunit.assertEquals(false, false)
end

function TestEquality:testNotEqual()
    luaunit.assertNotEquals(1, 2)
    luaunit.assertNotEquals("foo", "bar")
    luaunit.assertNotEquals(true, false)
end

function TestEquality:testTableEquality()
    local t1 = {1, 2, 3}
    local t2 = {1, 2, 3}
    -- assertEquals สำหรับ tables ตรวจสอบ deep equality
    luaunit.assertEquals(t1, t2)
end

function TestEquality:testNestedTableEquality()
    local t1 = {name = "Alice", scores = {90, 85, 92}}
    local t2 = {name = "Alice", scores = {90, 85, 92}}
    luaunit.assertEquals(t1, t2)
end

os.exit(luaunit.LuaUnit.run())
```

---

## 33.4 assertNil และ assertNotNil

### ตัวอย่างที่ 3: Nil Checks

```lua
-- test_nil.lua
local luaunit = require("luaunit")

TestNilChecks = {}

function TestNilChecks:testNilValue()
    local x = nil
    luaunit.assertNil(x)
end

function TestNilChecks:testTableLookup()
    local t = {a = 1, b = 2}
    luaunit.assertNil(t["c"])       -- key ไม่มี
    luaunit.assertNotNil(t["a"])    -- key มี
end

function TestNilChecks:testFunctionReturn()
    local function mayReturnNil(x)
        if x > 0 then
            return x
        end
        return nil
    end
    
    luaunit.assertNotNil(mayReturnNil(5))
    luaunit.assertNil(mayReturnNil(-1))
end

function TestNilChecks:testOptionalParameters()
    local function greet(name, greeting)
        greeting = greeting or "Hello"
        return greeting .. ", " .. (name or "World") .. "!"
    end
    
    -- ฟังก์ชันควร return string ไม่ใช่ nil
    luaunit.assertNotNil(greet("Alice"))
    luaunit.assertNotNil(greet())
    luaunit.assertNotNil(greet("Bob", "Hi"))
end

os.exit(luaunit.LuaUnit.run())
```

---

## 33.5 assertTrue และ assertFalse

### ตัวอย่างที่ 4: Boolean Assertions

```lua
-- test_boolean.lua
local luaunit = require("luaunit")

TestBooleanAssertions = {}

function TestBooleanAssertions:testIsEven()
    local function isEven(n)
        return n % 2 == 0
    end
    
    luaunit.assertTrue(isEven(4))
    luaunit.assertTrue(isEven(0))
    luaunit.assertTrue(isEven(-6))
    luaunit.assertFalse(isEven(3))
    luaunit.assertFalse(isEven(7))
end

function TestBooleanAssertions:testStringStartsWith()
    local function startsWith(str, prefix)
        return str:sub(1, #prefix) == prefix
    end
    
    luaunit.assertTrue(startsWith("Hello World", "Hello"))
    luaunit.assertTrue(startsWith("Lua 5.4", "Lua"))
    luaunit.assertFalse(startsWith("Hello", "World"))
    luaunit.assertFalse(startsWith("", "prefix"))
end

function TestBooleanAssertions:testContainsKey()
    local function hasKey(t, key)
        return t[key] ~= nil
    end
    
    local config = {
        host = "localhost",
        port = 8080
    }
    
    luaunit.assertTrue(hasKey(config, "host"))
    luaunit.assertTrue(hasKey(config, "port"))
    luaunit.assertFalse(hasKey(config, "password"))
    luaunit.assertFalse(hasKey(config, "debug"))
end

function TestBooleanAssertions:testRangeCheck()
    local function inRange(value, min, max)
        return value >= min and value <= max
    end
    
    luaunit.assertTrue(inRange(50, 0, 100))
    luaunit.assertTrue(inRange(0, 0, 100))
    luaunit.assertTrue(inRange(100, 0, 100))
    luaunit.assertFalse(inRange(-1, 0, 100))
    luaunit.assertFalse(inRange(101, 0, 100))
end

os.exit(luaunit.LuaUnit.run())
```

---

## 33.6 assertError

### ตัวอย่างที่ 5: Error Assertions

```lua
-- test_errors.lua
local luaunit = require("luaunit")

TestErrorAssertions = {}

function TestErrorAssertions:testDivisionByZero()
    local function safeDivide(a, b)
        if b == 0 then
            error("Division by zero!")
        end
        return a / b
    end
    
    -- ตรวจสอบว่า function throw error
    luaunit.assertError(safeDivide, 10, 0)
    
    -- ตรวจสอบว่า function ไม่ throw error
    local ok, result = pcall(safeDivide, 10, 2)
    luaunit.assertTrue(ok)
    luaunit.assertEquals(5, result)
end

function TestErrorAssertions:testInvalidInput()
    local function validateEmail(email)
        if type(email) ~= "string" then
            error("email ต้องเป็น string")
        end
        if not email:find("@") then
            error("email ต้องมี @")
        end
        return true
    end
    
    luaunit.assertError(validateEmail, 123)
    luaunit.assertError(validateEmail, "invalid-email")
    
    -- valid email ไม่ควร throw
    local ok = pcall(validateEmail, "test@example.com")
    luaunit.assertTrue(ok)
end

function TestErrorAssertions:testAssertErrorWithMessage()
    local function riskyFunction()
        error("specific error message")
    end
    
    -- assertErrorMsgEquals ตรวจสอบ error message ด้วย
    luaunit.assertErrorMsgEquals(
        "specific error message",
        riskyFunction
    )
end

function TestErrorAssertions:testAssertErrorMsgContains()
    local function anotherRiskyFunction(x)
        if x < 0 then
            error("ค่า x (" .. x .. ") ต้องไม่ติดลบ")
        end
    end
    
    -- ตรวจสอบว่า error message มี substring
    luaunit.assertErrorMsgContains(
        "ต้องไม่ติดลบ",
        anotherRiskyFunction,
        -5
    )
end

os.exit(luaunit.LuaUnit.run())
```

---

## 33.7 assertAlmostEquals

### ตัวอย่างที่ 6: Floating Point Comparisons

```lua
-- test_float.lua
local luaunit = require("luaunit")

TestFloatAssertions = {}

function TestFloatAssertions:testBasicFloat()
    -- floating point ไม่ควรเปรียบเทียบด้วย == โดยตรง
    local result = 1/3
    luaunit.assertAlmostEquals(0.333, result, 0.001)
end

function TestFloatAssertions:testTrigonometry()
    -- sin(π/2) ≈ 1
    luaunit.assertAlmostEquals(1.0, math.sin(math.pi/2), 1e-10)
    
    -- cos(0) = 1
    luaunit.assertAlmostEquals(1.0, math.cos(0), 1e-10)
    
    -- tan(π/4) ≈ 1
    luaunit.assertAlmostEquals(1.0, math.tan(math.pi/4), 1e-10)
end

function TestFloatAssertions:testSquareRoot()
    luaunit.assertAlmostEquals(1.41421, math.sqrt(2), 0.00001)
    luaunit.assertAlmostEquals(1.73205, math.sqrt(3), 0.00001)
    luaunit.assertAlmostEquals(2.23607, math.sqrt(5), 0.00001)
end

function TestFloatAssertions:testCurrencyCalculation()
    -- ทดสอบการคำนวณเงิน (มักมีปัญหา floating point)
    local price = 19.99
    local quantity = 3
    local total = price * quantity  -- อาจได้ 59.97000000001
    
    luaunit.assertAlmostEquals(59.97, total, 0.001)
end

function TestFloatAssertions:testNotAlmostEqual()
    luaunit.assertNotAlmostEquals(1.0, 2.0, 0.5)
    luaunit.assertNotAlmostEquals(3.14, 2.72, 0.1)
end

os.exit(luaunit.LuaUnit.run())
```

---

## 33.8 assertStrContains

### ตัวอย่างที่ 7: String Assertions

```lua
-- test_strings.lua
local luaunit = require("luaunit")

TestStringAssertions = {}

function TestStringAssertions:testContains()
    local message = "Hello, World! Welcome to Lua."
    
    luaunit.assertStrContains(message, "World")
    luaunit.assertStrContains(message, "Lua")
    luaunit.assertStrContains(message, "Hello")
end

function TestStringAssertions:testNotContains()
    local message = "Hello, World!"
    
    luaunit.assertNotStrContains(message, "Python")
    luaunit.assertNotStrContains(message, "JavaScript")
end

function TestStringAssertions:testMatchesPattern()
    local email = "user@example.com"
    
    -- ตรวจสอบ pattern ด้วย assertStrMatches
    luaunit.assertStrMatches(email, "%w+@%w+%.%w+")
end

function TestStringAssertions:testErrorMessage()
    -- ตรวจสอบว่า error message มี substring
    local function failingFunction()
        error("ข้อผิดพลาด: ไม่พบไฟล์ config.lua")
    end
    
    luaunit.assertErrorMsgContains("ไม่พบไฟล์", failingFunction)
end

function TestStringAssertions:testStringFormat()
    local function formatName(first, last)
        return string.format("%s %s", first, last)
    end
    
    local result = formatName("John", "Doe")
    luaunit.assertStrContains(result, "John")
    luaunit.assertStrContains(result, "Doe")
    luaunit.assertEquals("John Doe", result)
end

os.exit(luaunit.LuaUnit.run())
```

---

## 33.9 assertTableEquals

### ตัวอย่างที่ 8: Table Assertions

```lua
-- test_tables.lua
local luaunit = require("luaunit")

TestTableAssertions = {}

function TestTableAssertions:testSimpleTable()
    local expected = {1, 2, 3, 4, 5}
    local actual = {}
    for i = 1, 5 do
        table.insert(actual, i)
    end
    
    luaunit.assertEquals(expected, actual)
end

function TestTableAssertions:testHashTable()
    local expected = {
        name = "Alice",
        age = 30,
        city = "Bangkok"
    }
    
    local actual = {
        city = "Bangkok",
        name = "Alice",
        age = 30
    }
    
    -- ลำดับ key ไม่สำคัญ
    luaunit.assertEquals(expected, actual)
end

function TestTableAssertions:testNestedTable()
    local expected = {
        user = {
            id = 1,
            profile = {
                name = "Bob",
                email = "bob@example.com"
            }
        },
        settings = {
            theme = "dark",
            language = "th"
        }
    }
    
    local actual = {
        user = {
            id = 1,
            profile = {
                name = "Bob",
                email = "bob@example.com"
            }
        },
        settings = {
            theme = "dark",
            language = "th"
        }
    }
    
    luaunit.assertEquals(expected, actual)
end

function TestTableAssertions:testArrayOrdering()
    -- array ordering สำคัญ
    local t1 = {1, 2, 3}
    local t2 = {3, 2, 1}  -- ลำดับต่างกัน
    
    luaunit.assertNotEquals(t1, t2)
end

function TestTableAssertions:testTableLength()
    local t = {10, 20, 30, 40, 50}
    luaunit.assertEquals(5, #t)
end

os.exit(luaunit.LuaUnit.run())
```

---

## 33.10 setUp และ tearDown

### ตัวอย่างที่ 9: setUp และ tearDown Method

```lua
-- test_setup_teardown.lua
local luaunit = require("luaunit")

TestWithSetup = {}

-- รันก่อนแต่ละ test method
function TestWithSetup:setUp()
    -- สร้าง test data ใหม่สำหรับแต่ละ test
    self.database = {
        users = {},
        nextId = 1
    }
    
    self.sampleUsers = {
        {name = "Alice", email = "alice@example.com"},
        {name = "Bob", email = "bob@example.com"},
        {name = "Charlie", email = "charlie@example.com"},
    }
end

-- รันหลังแต่ละ test method
function TestWithSetup:tearDown()
    -- ล้างข้อมูล
    self.database = nil
    self.sampleUsers = nil
end

function TestWithSetup:testInsertUser()
    local user = {name = "Dave", email = "dave@example.com"}
    self.database.users[self.database.nextId] = user
    self.database.nextId = self.database.nextId + 1
    
    luaunit.assertEquals(1, #self.database.users)
end

function TestWithSetup:testInsertMultipleUsers()
    for _, user in ipairs(self.sampleUsers) do
        self.database.users[self.database.nextId] = user
        self.database.nextId = self.database.nextId + 1
    end
    
    luaunit.assertEquals(3, #self.database.users)
end

function TestWithSetup:testDatabaseStartsEmpty()
    -- setUp เรียกใหม่ทุก test ดังนั้น database ควรว่าง
    luaunit.assertEquals(0, #self.database.users)
    luaunit.assertEquals(1, self.database.nextId)
end

os.exit(luaunit.LuaUnit.run())
```

### ตัวอย่างที่ 10: setUpClass และ tearDownClass (Class-level)

```lua
-- test_class_setup.lua
local luaunit = require("luaunit")

TestWithClassSetup = {}
TestWithClassSetup.sharedData = nil
TestWithClassSetup.connectionCount = 0

-- รันครั้งเดียวก่อน tests ทั้งหมดในชั้น
-- (LuaUnit ไม่มี setUpClass โดยตรง แต่ใช้ pattern นี้)
local sharedConnection = nil

-- ใช้ setUp ปกติแต่ตรวจสอบว่าสร้างแล้วหรือยัง
function TestWithClassSetup:setUp()
    if not sharedConnection then
        -- สร้างครั้งเดียว
        sharedConnection = {
            host = "localhost",
            port = 5432,
            isConnected = true
        }
        print("สร้าง shared connection")
    end
    self.connection = sharedConnection
end

function TestWithClassSetup:testConnectionExists()
    luaunit.assertNotNil(self.connection)
    luaunit.assertTrue(self.connection.isConnected)
end

function TestWithClassSetup:testConnectionHost()
    luaunit.assertEquals("localhost", self.connection.host)
end

function TestWithClassSetup:testConnectionPort()
    luaunit.assertEquals(5432, self.connection.port)
end

os.exit(luaunit.LuaUnit.run())
```

---

## 33.11 Test Suites

### ตัวอย่างที่ 11: จัดการ Test Suites

```lua
-- test_suite.lua
local luaunit = require("luaunit")

-- Test Class 1: Math Operations
TestMathOperations = {}

function TestMathOperations:testAdd()
    luaunit.assertEquals(10, 3 + 7)
end

function TestMathOperations:testPower()
    luaunit.assertEquals(8, 2^3)
end

function TestMathOperations:testModulo()
    luaunit.assertEquals(1, 7 % 3)
end

function TestMathOperations:testAbsoluteValue()
    luaunit.assertEquals(5, math.abs(-5))
    luaunit.assertEquals(5, math.abs(5))
end

-- Test Class 2: String Operations
TestStringOperations = {}

function TestStringOperations:testLength()
    luaunit.assertEquals(5, #"hello")
    luaunit.assertEquals(0, #"")
end

function TestStringOperations:testConcatenation()
    luaunit.assertEquals("Hello World", "Hello" .. " " .. "World")
end

function TestStringOperations:testRepeat()
    luaunit.assertEquals("aaa", string.rep("a", 3))
    luaunit.assertEquals("ab-ab-ab", string.rep("ab", 3, "-"))
end

function TestStringOperations:testReverse()
    luaunit.assertEquals("olleH", string.reverse("Hello"))
end

-- Test Class 3: Table Operations
TestTableOperations = {}

function TestTableOperations:testInsert()
    local t = {}
    table.insert(t, "a")
    table.insert(t, "b")
    table.insert(t, "c")
    luaunit.assertEquals(3, #t)
    luaunit.assertEquals("c", t[3])
end

function TestTableOperations:testSort()
    local t = {5, 3, 8, 1, 9, 2}
    table.sort(t)
    luaunit.assertEquals({1, 2, 3, 5, 8, 9}, t)
end

function TestTableOperations:testConcat()
    local t = {"Hello", "World", "Lua"}
    luaunit.assertEquals("Hello, World, Lua", table.concat(t, ", "))
end

-- รัน test suite เฉพาะ
-- lua test_suite.lua TestMathOperations  -- รันเฉพาะ class
-- lua test_suite.lua TestMathOperations.testAdd  -- รันเฉพาะ method
os.exit(luaunit.LuaUnit.run())
```

---

## 33.12 รัน Tests จาก Command Line

### ตัวอย่างที่ 12: Command Line Options

```bash
# รัน tests ทั้งหมด
lua test_suite.lua

# รัน tests แบบ verbose
lua test_suite.lua -v

# รัน test class เฉพาะ
lua test_suite.lua TestMathOperations

# รัน test method เฉพาะ
lua test_suite.lua TestMathOperations.testAdd

# รัน พร้อม output format
lua test_suite.lua --output text
lua test_suite.lua --output tap
lua test_suite.lua --output junit

# รัน พร้อมบันทึกผล
lua test_suite.lua --output junit > test_results.xml
```

---

## 33.13 Output Formats

### ตัวอย่างที่ 13: Text Output

```lua
-- test_output_demo.lua
local luaunit = require("luaunit")

TestOutputDemo = {}

function TestOutputDemo:testPass()
    luaunit.assertEquals(2, 1 + 1)
end

function TestOutputDemo:testAnother()
    luaunit.assertTrue(1 < 2)
end

-- Text output (default)
-- lua test_output_demo.lua -v
-- Output:
-- TestOutputDemo
--   testAnother ... OK
--   testPass ... OK
-- Ran 2 tests in 0.001 seconds, 2 successes, 0 failures

os.exit(luaunit.LuaUnit.run())
```

### ตัวอย่างที่ 14: TAP Output

```bash
# TAP (Test Anything Protocol) output
# lua test_output_demo.lua --output tap

# Output:
# ok 1 - TestOutputDemo.testAnother
# ok 2 - TestOutputDemo.testPass
# 1..2
```

### ตัวอย่างที่ 15: JUnit XML Output

```bash
# JUnit XML สำหรับ CI/CD
# lua test_suite.lua --output junit > test_results.xml
```

```xml
<!-- ตัวอย่าง JUnit XML output -->
<?xml version="1.0" encoding="UTF-8"?>
<testsuites>
  <testsuite name="TestMathOperations" tests="4" failures="0" errors="0">
    <testcase classname="TestMathOperations" name="testAdd" time="0.001"/>
    <testcase classname="TestMathOperations" name="testPower" time="0.001"/>
    <testcase classname="TestMathOperations" name="testModulo" time="0.001"/>
    <testcase classname="TestMathOperations" name="testAbsoluteValue" time="0.001"/>
  </testsuite>
</testsuites>
```

---

## 33.14 CI Integration

### ตัวอย่างที่ 16: GitHub Actions Integration

```yaml
# .github/workflows/test.yml
name: Lua Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        lua-version: ['5.3', '5.4']
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Lua ${{ matrix.lua-version }}
      uses: leafo/gh-actions-lua@v9
      with:
        luaVersion: ${{ matrix.lua-version }}
    
    - name: Setup LuaRocks
      uses: leafo/gh-actions-luarocks@v4
    
    - name: Install LuaUnit
      run: luarocks install luaunit
    
    - name: Run Tests
      run: lua test_suite.lua --output junit > test_results.xml
    
    - name: Publish Test Results
      uses: dorny/test-reporter@v1
      if: always()
      with:
        name: Lua Tests (Lua ${{ matrix.lua-version }})
        path: test_results.xml
        reporter: java-junit
```

### ตัวอย่างที่ 17: GitLab CI Integration

```yaml
# .gitlab-ci.yml
stages:
  - test

lua-test:
  stage: test
  image: nickblah/lua:5.4
  before_script:
    - luarocks install luaunit
  script:
    - lua test_suite.lua --output junit > test_results.xml
  artifacts:
    when: always
    reports:
      junit: test_results.xml
```

---

## 33.15 Property-Based Testing Concepts

### ตัวอย่างที่ 18: Property-Based Testing แบบง่าย

```lua
-- test_property.lua
-- Property-based testing: ทดสอบ properties ที่ควรเป็นจริงเสมอ

local luaunit = require("luaunit")

-- Helper: สร้าง random integers
local function randomInt(min, max)
    return math.random(min, max)
end

-- Helper: รัน property test หลายครั้ง
local function forAll(generator, count, property)
    math.randomseed(42)  -- seed คงที่สำหรับ reproducibility
    for i = 1, count do
        local value = generator()
        if not property(value) then
            return false, value
        end
    end
    return true
end

TestProperties = {}

function TestProperties:testCommutativity()
    -- a + b == b + a ต้องเป็นจริงเสมอ
    math.randomseed(42)
    for i = 1, 100 do
        local a = randomInt(-1000, 1000)
        local b = randomInt(-1000, 1000)
        luaunit.assertEquals(a + b, b + a)
    end
end

function TestProperties:testAssociativity()
    -- (a + b) + c == a + (b + c)
    math.randomseed(42)
    for i = 1, 100 do
        local a = randomInt(-100, 100)
        local b = randomInt(-100, 100)
        local c = randomInt(-100, 100)
        luaunit.assertEquals((a + b) + c, a + (b + c))
    end
end

function TestProperties:testReverseReverse()
    -- reverse(reverse(s)) == s
    local strings = {
        "hello", "world", "lua", "test",
        "", "a", "ab", "abc"
    }
    
    for _, s in ipairs(strings) do
        local reversed = string.reverse(s)
        local doubleReversed = string.reverse(reversed)
        luaunit.assertEquals(s, doubleReversed)
    end
end

function TestProperties:testSortIdempotent()
    -- sort(sort(t)) == sort(t)
    math.randomseed(42)
    for i = 1, 10 do
        local t = {}
        for j = 1, 10 do
            table.insert(t, randomInt(1, 100))
        end
        
        -- สร้าง copies
        local t1 = {table.unpack(t)}
        local t2 = {table.unpack(t)}
        
        table.sort(t1)
        table.sort(t2)
        table.sort(t2)  -- sort ซ้ำ
        
        luaunit.assertEquals(t1, t2)
    end
end

function TestProperties:testTableLength()
    -- หลัง insert n elements, #t == n
    math.randomseed(42)
    for trial = 1, 20 do
        local n = randomInt(1, 50)
        local t = {}
        
        for i = 1, n do
            table.insert(t, i)
        end
        
        luaunit.assertEquals(n, #t)
    end
end

os.exit(luaunit.LuaUnit.run())
```

---

## 33.16 ตัวอย่าง Test Suite สมบูรณ์

### ตัวอย่างที่ 19: Module ที่จะ Test

```lua
-- src/stack.lua
-- Stack data structure

local Stack = {}
Stack.__index = Stack

function Stack.new()
    return setmetatable({
        data = {},
        size = 0
    }, Stack)
end

function Stack:push(value)
    if value == nil then
        error("ไม่สามารถ push nil เข้า stack ได้")
    end
    self.size = self.size + 1
    self.data[self.size] = value
end

function Stack:pop()
    if self:isEmpty() then
        error("Stack ว่าง ไม่สามารถ pop ได้")
    end
    local value = self.data[self.size]
    self.data[self.size] = nil
    self.size = self.size - 1
    return value
end

function Stack:peek()
    if self:isEmpty() then
        return nil
    end
    return self.data[self.size]
end

function Stack:isEmpty()
    return self.size == 0
end

function Stack:getSize()
    return self.size
end

function Stack:clear()
    self.data = {}
    self.size = 0
end

function Stack:toArray()
    local result = {}
    for i = 1, self.size do
        result[i] = self.data[i]
    end
    return result
end

return Stack
```

### ตัวอย่างที่ 20: Test Suite สมบูรณ์สำหรับ Stack

```lua
-- test_stack.lua
local luaunit = require("luaunit")
local Stack = require("src.stack")

TestStack = {}

function TestStack:setUp()
    self.stack = Stack.new()
end

function TestStack:tearDown()
    self.stack = nil
end

-- ทดสอบ new stack
function TestStack:testNewStackIsEmpty()
    luaunit.assertTrue(self.stack:isEmpty())
    luaunit.assertEquals(0, self.stack:getSize())
end

function TestStack:testNewStackPeekReturnsNil()
    luaunit.assertNil(self.stack:peek())
end

-- ทดสอบ push
function TestStack:testPushSingleElement()
    self.stack:push(1)
    luaunit.assertFalse(self.stack:isEmpty())
    luaunit.assertEquals(1, self.stack:getSize())
end

function TestStack:testPushMultipleElements()
    self.stack:push(1)
    self.stack:push(2)
    self.stack:push(3)
    luaunit.assertEquals(3, self.stack:getSize())
end

function TestStack:testPushNilThrowsError()
    luaunit.assertError(
        function() self.stack:push(nil) end
    )
end

function TestStack:testPushDifferentTypes()
    self.stack:push(42)
    self.stack:push("string")
    self.stack:push(true)
    self.stack:push({key = "value"})
    
    luaunit.assertEquals(4, self.stack:getSize())
end

-- ทดสอบ pop
function TestStack:testPopSingleElement()
    self.stack:push("hello")
    local value = self.stack:pop()
    
    luaunit.assertEquals("hello", value)
    luaunit.assertTrue(self.stack:isEmpty())
end

function TestStack:testPopEmptyThrowsError()
    luaunit.assertError(
        function() self.stack:pop() end
    )
end

function TestStack:testPopReturnsLIFOOrder()
    self.stack:push(1)
    self.stack:push(2)
    self.stack:push(3)
    
    luaunit.assertEquals(3, self.stack:pop())
    luaunit.assertEquals(2, self.stack:pop())
    luaunit.assertEquals(1, self.stack:pop())
end

-- ทดสอบ peek
function TestStack:testPeekReturnsTopWithoutRemoving()
    self.stack:push("top")
    self.stack:push("middle")  
    -- wait, this is wrong order - top should be last pushed
    
    -- ลำดับ: bottom=top, then middle added = middle is now top
    -- Let me re-push
    self.stack:clear()
    self.stack:push("bottom")
    self.stack:push("top_element")
    
    luaunit.assertEquals("top_element", self.stack:peek())
    luaunit.assertEquals(2, self.stack:getSize())  -- ไม่ถูก remove
end

-- ทดสอบ isEmpty
function TestStack:testIsEmptyOnNewStack()
    luaunit.assertTrue(self.stack:isEmpty())
end

function TestStack:testIsEmptyAfterPushPop()
    self.stack:push(1)
    self.stack:pop()
    luaunit.assertTrue(self.stack:isEmpty())
end

function TestStack:testIsNotEmptyWithElements()
    self.stack:push(1)
    luaunit.assertFalse(self.stack:isEmpty())
end

-- ทดสอบ clear
function TestStack:testClear()
    self.stack:push(1)
    self.stack:push(2)
    self.stack:push(3)
    
    self.stack:clear()
    
    luaunit.assertTrue(self.stack:isEmpty())
    luaunit.assertEquals(0, self.stack:getSize())
end

-- ทดสอบ toArray
function TestStack:testToArray()
    self.stack:push(10)
    self.stack:push(20)
    self.stack:push(30)
    
    local arr = self.stack:toArray()
    
    luaunit.assertEquals({10, 20, 30}, arr)
end

function TestStack:testToArrayEmpty()
    local arr = self.stack:toArray()
    luaunit.assertEquals({}, arr)
end

-- Property-based tests
function TestStack:testPushPopSymmetry()
    -- push n elements แล้ว pop n elements ควรได้ order กลับกัน
    local values = {1, 2, 3, 4, 5}
    
    for _, v in ipairs(values) do
        self.stack:push(v)
    end
    
    local popped = {}
    while not self.stack:isEmpty() do
        table.insert(popped, self.stack:pop())
    end
    
    -- popped ควรเป็น reverse ของ values
    local reversed = {}
    for i = #values, 1, -1 do
        table.insert(reversed, values[i])
    end
    
    luaunit.assertEquals(reversed, popped)
end

os.exit(luaunit.LuaUnit.run())
```

---

## 33.17 Advanced LuaUnit Features

### ตัวอย่างที่ 21: assertItemsEquals

```lua
-- test_advanced.lua
local luaunit = require("luaunit")

TestAdvanced = {}

function TestAdvanced:testItemsEquals()
    -- assertItemsEquals ไม่สนใจลำดับ
    local expected = {1, 2, 3, 4, 5}
    local actual = {5, 3, 1, 4, 2}  -- ลำดับต่างกัน
    
    luaunit.assertItemsEquals(expected, actual)
end

function TestAdvanced:testTableContains()
    local function contains(t, value)
        for _, v in ipairs(t) do
            if v == value then
                return true
            end
        end
        return false
    end
    
    local fruits = {"apple", "banana", "cherry", "date"}
    luaunit.assertTrue(contains(fruits, "banana"))
    luaunit.assertFalse(contains(fruits, "grape"))
end

function TestAdvanced:testTypeChecks()
    luaunit.assertIsString("hello")
    luaunit.assertIsNumber(42)
    luaunit.assertIsBoolean(true)
    luaunit.assertIsTable({})
    luaunit.assertIsFunction(print)
    luaunit.assertIsNil(nil)
end

function TestAdvanced:testIs()
    -- assertIs ตรวจสอบว่าเป็น object เดียวกัน (reference)
    local obj = {}
    local ref = obj
    luaunit.assertIs(obj, ref)
    
    local different = {}
    luaunit.assertIsNot(obj, different)
end

os.exit(luaunit.LuaUnit.run())
```

### ตัวอย่างที่ 22: SkipTest Pattern

```lua
-- test_skip.lua
local luaunit = require("luaunit")

TestWithSkip = {}

function TestWithSkip:testNormalTest()
    luaunit.assertEquals(2 + 2, 4)
end

function TestWithSkip:testRequiresDatabase()
    -- ถ้าไม่มี database ให้ skip
    local hasDatabase = false  -- เปลี่ยนเป็น true ถ้ามี db
    
    if not hasDatabase then
        -- LuaUnit ไม่มี built-in skip
        -- แต่ทำได้ด้วย condition
        print("SKIP: ต้องการ database connection")
        return
    end
    
    -- test ที่ต้องการ database
    luaunit.assertEquals(1, 1)
end

function TestWithSkip:testRequiresNetwork()
    local hasNetwork = os.execute("ping -c 1 8.8.8.8 > /dev/null 2>&1") == 0
    
    if not hasNetwork then
        print("SKIP: ต้องการ network connection")
        return
    end
    
    -- test ที่ต้องการ network
    luaunit.assertTrue(true)
end

os.exit(luaunit.LuaUnit.run())
```

---

## 33.18 Test Organization ขนาดใหญ่

### ตัวอย่างที่ 23: Multiple Test Files

```lua
-- run_all_tests.lua
-- รัน tests จากหลายไฟล์

local luaunit = require("luaunit")

-- นำเข้า test modules
require("test_math")
require("test_strings")
require("test_tables")
require("test_stack")

-- รัน tests ทั้งหมด
os.exit(luaunit.LuaUnit.run())
```

```bash
# รัน tests ทั้งหมดในครั้งเดียว
lua run_all_tests.lua

# หรือรัน tests ทีละไฟล์
for f in test_*.lua; do
    echo "Running $f..."
    lua "$f"
done
```

### ตัวอย่างที่ 24: Test Tagging Pattern

```lua
-- test_tagged.lua
local luaunit = require("luaunit")

-- จำลอง tag system ด้วย naming convention
TestUnit = {}  -- Unit tests
TestIntegration = {}  -- Integration tests
TestPerformance = {}  -- Performance tests

-- Unit tests
function TestUnit:testSimpleCalculation()
    luaunit.assertEquals(100, 10 * 10)
end

function TestUnit:testStringManipulation()
    luaunit.assertEquals("HELLO", string.upper("hello"))
end

-- Integration tests
function TestIntegration:testModuleInteraction()
    -- ทดสอบการทำงานร่วมกัน
    local t = {}
    for i = 1, 10 do
        table.insert(t, i)
    end
    local sum = 0
    for _, v in ipairs(t) do
        sum = sum + v
    end
    luaunit.assertEquals(55, sum)
end

-- Performance tests
function TestPerformance:testLargeLoop()
    local start = os.clock()
    local sum = 0
    for i = 1, 1000000 do
        sum = sum + i
    end
    local elapsed = os.clock() - start
    
    luaunit.assertEquals(500000500000, sum)
    -- ควรเสร็จภายใน 1 วินาที
    luaunit.assertTrue(elapsed < 1.0,
        string.format("ใช้เวลา %.3f วินาที (ควรน้อยกว่า 1.0)", elapsed))
end

-- รัน เฉพาะ unit tests
-- lua test_tagged.lua TestUnit

-- รัน ทั้งหมด
os.exit(luaunit.LuaUnit.run())
```

---

## 33.19 Comparing Busted vs LuaUnit

### ตัวอย่างที่ 25: test เดียวกันใน 2 frameworks

```lua
-- ด้วย Busted (BDD style)
-- spec/calculator_spec.lua

describe("Calculator", function()
    local calc
    
    before_each(function()
        calc = {
            add = function(a, b) return a + b end,
            sub = function(a, b) return a - b end,
        }
    end)
    
    it("ควรบวกเลขได้", function()
        assert.are.equal(5, calc.add(2, 3))
    end)
    
    it("ควรลบเลขได้", function()
        assert.are.equal(1, calc.sub(3, 2))
    end)
end)
```

```lua
-- ด้วย LuaUnit (xUnit style)
-- test_calculator.lua
local luaunit = require("luaunit")

TestCalculator = {}

function TestCalculator:setUp()
    self.calc = {
        add = function(a, b) return a + b end,
        sub = function(a, b) return a - b end,
    }
end

function TestCalculator:testAdd()
    luaunit.assertEquals(5, self.calc.add(2, 3))
end

function TestCalculator:testSubtract()
    luaunit.assertEquals(1, self.calc.sub(3, 2))
end

os.exit(luaunit.LuaUnit.run())
```

---

## 33.20 Best Practices

### ตัวอย่างที่ 26: Test Naming Convention

```lua
-- test_naming_best_practices.lua
local luaunit = require("luaunit")

TestUserRegistration = {}

-- ชื่อ test ควรอธิบายว่าทดสอบอะไร ภายใต้เงื่อนไขอะไร คาดหวังอะไร
-- pattern: test_[scenario]_[expected_result]

function TestUserRegistration:testValidUser_shouldSucceed()
    local result = {success = true, userId = 1}
    luaunit.assertTrue(result.success)
    luaunit.assertNotNil(result.userId)
end

function TestUserRegistration:testDuplicateEmail_shouldFail()
    local result = {success = false, error = "Email already exists"}
    luaunit.assertFalse(result.success)
    luaunit.assertStrContains(result.error, "already exists")
end

function TestUserRegistration:testInvalidEmail_shouldFail()
    local result = {success = false, error = "Invalid email format"}
    luaunit.assertFalse(result.success)
end

function TestUserRegistration:testEmptyPassword_shouldFail()
    local result = {success = false, error = "Password cannot be empty"}
    luaunit.assertFalse(result.success)
end

os.exit(luaunit.LuaUnit.run())
```

### ตัวอย่างที่ 27: Test Data Builders

```lua
-- test_with_builders.lua
local luaunit = require("luaunit")

-- Builder pattern สำหรับ test data
local UserBuilder = {}
UserBuilder.__index = UserBuilder

function UserBuilder.new()
    return setmetatable({
        _data = {
            id = 1,
            username = "testuser",
            email = "test@example.com",
            age = 25,
            role = "user",
            isActive = true
        }
    }, UserBuilder)
end

function UserBuilder:withUsername(name)
    self._data.username = name
    return self
end

function UserBuilder:withEmail(email)
    self._data.email = email
    return self
end

function UserBuilder:withAge(age)
    self._data.age = age
    return self
end

function UserBuilder:withRole(role)
    self._data.role = role
    return self
end

function UserBuilder:inactive()
    self._data.isActive = false
    return self
end

function UserBuilder:build()
    -- return copy
    local data = {}
    for k, v in pairs(self._data) do
        data[k] = v
    end
    return data
end

-- ใช้ builder ใน tests
TestUserBuilder = {}

function TestUserBuilder:testDefaultUser()
    local user = UserBuilder.new():build()
    
    luaunit.assertEquals("testuser", user.username)
    luaunit.assertEquals("user", user.role)
    luaunit.assertTrue(user.isActive)
end

function TestUserBuilder:testAdminUser()
    local admin = UserBuilder.new()
        :withUsername("admin")
        :withEmail("admin@example.com")
        :withRole("admin")
        :build()
    
    luaunit.assertEquals("admin", admin.username)
    luaunit.assertEquals("admin", admin.role)
end

function TestUserBuilder:testInactiveUser()
    local user = UserBuilder.new()
        :withUsername("olduser")
        :inactive()
        :build()
    
    luaunit.assertFalse(user.isActive)
end

os.exit(luaunit.LuaUnit.run())
```

---

## สรุปบทที่ 33

LuaUnit เป็น testing framework แบบ xUnit สำหรับ Lua:

| Method | การใช้งาน |
|--------|----------|
| `assertEquals(a, b)` | a == b |
| `assertNotEquals(a, b)` | a ~= b |
| `assertNil(x)` | x == nil |
| `assertNotNil(x)` | x ~= nil |
| `assertTrue(x)` | x == true |
| `assertFalse(x)` | x == false |
| `assertError(f, ...)` | f throw error |
| `assertAlmostEquals(a, b, delta)` | \|a-b\| < delta |
| `assertStrContains(s, sub)` | s contains sub |
| `assertItemsEquals(t1, t2)` | same items |
| `assertIsString(x)` | type(x) == "string" |
| `assertIsNumber(x)` | type(x) == "number" |

### Output Formats

| Format | Command |
|--------|---------|
| Text (default) | `lua test.lua` |
| Verbose | `lua test.lua -v` |
| TAP | `lua test.lua --output tap` |
| JUnit XML | `lua test.lua --output junit` |

---

*บทต่อไป: บทที่ 34 - Performance Optimization*
