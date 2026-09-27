# บทที่ 81: Lua VM Internals

## บทนำ

Lua เป็นภาษาโปรแกรมที่ถูกออกแบบมาให้มีขนาดเล็กและรวดเร็ว หัวใจสำคัญของ Lua คือ Virtual Machine (VM) ที่ทำงานบน register-based architecture ซึ่งแตกต่างจาก stack-based VM อย่าง JVM หรือ CPython บทนี้จะพาคุณดำดิ่งลงไปใน Lua VM internals เพื่อทำความเข้าใจว่า Lua ทำงานอย่างไรในระดับต่ำสุด

การเข้าใจ VM internals มีประโยชน์อย่างมากสำหรับ:
- การ optimize โค้ดให้ทำงานเร็วขึ้น
- การ debug ปัญหาที่ซับซ้อน
- การเขียน C extension ที่มีประสิทธิภาพ
- การเข้าใจ trade-off ในการออกแบบภาษา

---

## 81.1 สถาปัตยกรรม Lua VM

### Register-Based vs Stack-Based

VM มีสองแบบหลัก:

**Stack-Based VM** (เช่น JVM, CPython):
- ทุก operation ทำงานกับ stack
- `a + b` → push a, push b, add (pop 2, push result)
- โค้ดสั้นกว่าแต่ต้องมี instruction มากกว่า

**Register-Based VM** (เช่น Lua, Dalvik):
- operation ทำงานกับ register โดยตรง
- `a + b → r` → `ADD r, a, b`
- instruction น้อยกว่าแต่แต่ละ instruction ใหญ่กว่า

```lua
-- ตัวอย่าง: เข้าใจ register model
-- โค้ด Lua นี้:
local a = 10
local b = 20
local c = a + b
print(c)

-- จะถูกแปลงเป็น pseudo-bytecode ประมาณนี้:
-- LOADK   R0, 10      ; R0 = 10 (a)
-- LOADK   R1, 20      ; R1 = 20 (b)
-- ADD     R2, R0, R1  ; R2 = R0 + R1 (c)
-- GETTABUP R3, ENV, "print"
-- MOVE    R4, R2
-- CALL    R3, 2, 1
-- RETURN  0, 1
```

### Lua VM Components

```
┌─────────────────────────────────────────┐
│              Lua State (lua_State)       │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  Stack   │  │ Call Info│  │   GC   │ │
│  └──────────┘  └──────────┘  └────────┘ │
│  ┌──────────────────────────────────────┤
│  │           Global State               │
│  │  ┌────────┐  ┌────────┐  ┌────────┐ │
│  │  │Registry│  │Strings │  │  GC    │ │
│  │  └────────┘  └────────┘  └────────┘ │
└─────────────────────────────────────────┘
```

```lua
-- ดูขนาด Lua state
-- ใช้ C API ดู internal structure
-- lua_State มี fields หลักๆ ดังนี้:

--[[
struct lua_State {
  CommonHeader;           -- GC header
  lu_byte status;         -- thread status
  StkId top;             -- first free slot in stack
  global_State *l_G;     -- pointer to global state
  CallInfo *ci;          -- call info for current function
  const Instruction *oldpc; -- last pc traced
  StkId stack_last;      -- last free slot in stack
  StkId stack;           -- stack base
  UpVal *openupval;      -- list of open upvalues
  GCObject *gclist;      -- GC list
  struct lua_State *twups; -- list of threads with open upvalues
  struct lua_longjmp *errorJmp; -- current error recover point
  CallInfo base_ci;      -- CallInfo for first level (C calling Lua)
  volatile lua_Hook hook;
  ptrdiff_t errfunc;     -- current error handling function (stack index)
  int stacksize;
  int basehookcount;
  int hookcount;
  lu_byte hookmask;
  lu_byte allowhook;
  unsigned short nCcalls; -- number of nested C calls
  lu_byte nny;           -- number of non-yieldable calls in stack
};
]]
```

---

## 81.2 Lua Bytecode Format

### การ compile โค้ด Lua

เมื่อ Lua อ่านซอร์สโค้ด มันจะผ่านขั้นตอนเหล่านี้:

```
Source Code → Lexer → Tokens → Parser → AST → Code Generator → Bytecode
```

```lua
-- ตัวอย่าง: compile และดู bytecode ด้วย luac
-- สร้างไฟล์ test.lua:

local function add(a, b)
    return a + b
end

local result = add(10, 20)
print(result)
```

### โครงสร้าง Function Prototype

Function prototype (Proto) เป็น struct ที่เก็บข้อมูลทั้งหมดของ function:

```c
// จาก lobject.h ใน Lua source
typedef struct Proto {
  CommonHeader;
  lu_byte numparams;    // จำนวน fixed parameters
  lu_byte is_vararg;    // เป็น vararg function หรือไม่
  lu_byte maxstacksize; // จำนวน registers ที่ต้องการ
  int sizeupvalues;     // ขนาดของ upvalue list
  int sizek;            // จำนวน constants
  int sizecode;         // จำนวน instructions
  int sizelineinfo;     // ขนาด line info
  int sizep;            // จำนวน nested functions
  int sizelocvars;      // จำนวน local variables
  int linedefined;      // บรรทัดที่ define function
  int lastlinedefined;  // บรรทัดสุดท้าย
  TValue *k;            // constants array
  Instruction *code;    // bytecode array
  struct Proto **p;     // nested functions
  int *lineinfo;        // line number per instruction
  LocVar *locvars;      // local variable info
  Upvaldesc *upvalues;  // upvalue descriptors
  struct LClosure *cache; // last-created closure
  TString *source;      // source name
  GCObject *gclist;
} Proto;
```

```lua
-- ตัวอย่างการดู function prototype ด้วย debug library
local function inspect_proto(f)
    local info = debug.getinfo(f, "Slnu")
    print("=== Function Prototype ===")
    print("Source:", info.source)
    print("Lines:", info.linedefined, "-", info.lastlinedefined)
    print("Params:", info.nparams)
    print("Is vararg:", info.isvararg)
    
    -- ดู upvalues
    print("\n--- Upvalues ---")
    local i = 1
    while true do
        local name, val = debug.getupvalue(f, i)
        if not name then break end
        print(string.format("  [%d] %s = %s", i, name, tostring(val)))
        i = i + 1
    end
    
    -- ดู local variables (ต้องเรียกจาก running function)
    print("\n--- Info ---")
    print("MaxStack:", info.nups, "upvalues")
end

local x = 10
local function closure_example(a, b)
    return a + b + x
end

inspect_proto(closure_example)
```

---

## 81.3 การใช้ luac Compiler

### การ compile ด้วย luac

```bash
# compile ไฟล์ Lua เป็น bytecode
luac -o output.luac input.lua

# disassemble bytecode
luac -l input.lua

# disassemble แบบ verbose (ทุก instruction)
luac -l -l input.lua

# ไม่บันทึก output (แค่ disassemble)
luac -l -p input.lua
```

### ตัวอย่าง Disassembly

```lua
-- สร้างไฟล์ example.lua
local function factorial(n)
    if n <= 1 then
        return 1
    end
    return n * factorial(n - 1)
end

print(factorial(10))
```

Output จาก `luac -l example.lua`:

```
main <example.lua:0,0> (8 instructions at 0x...)
0+ params, 2 slots, 1 upvalue, 0 locals, 1 constant, 1 function
    1	[9]	GETTABUP	0 0 -1	; _ENV "print"
    2	[9]	CLOSURE	1 0	; 0x... (factorial)
    3	[9]	LOADK	2 -2	; 10
    4	[9]	CALL	1 2 2
    5	[9]	CALL	0 2 1
    6	[9]	RETURN	0 1

function <example.lua:1,7> (11 instructions at 0x...)
1 param, 3 slots, 1 upvalue, 1 local, 2 constants, 0 functions
    1	[2]	LE	0 -1 0	; - 1
    2	[2]	JMP	0 1	; to 4
    3	[3]	RETURN	0 2
    4	[3]	LOADK	1 -1	; 1
    5	[3]	RETURN	1 2
    6	[5]	GETTABUP	1 0 -3	; _ENV "factorial"  
    7	[5]	SUB	2 0 -1	; - 1
    8	[5]	CALL	1 2 2
    9	[5]	MUL	1 0 1
   10	[5]	RETURN	1 2
   11	[7]	RETURN	0 1
```

```lua
-- อ่าน bytecode จาก string
local function compile_and_show(code)
    -- load เป็น function
    local f = load(code)
    if not f then
        print("Compile error")
        return
    end
    
    -- ใช้ string.dump เพื่อได้ bytecode
    local bytecode = string.dump(f)
    print("Bytecode length:", #bytecode, "bytes")
    
    -- แสดง hex dump ของ 64 bytes แรก
    local hex = {}
    for i = 1, math.min(64, #bytecode) do
        hex[i] = string.format("%02x", bytecode:byte(i))
    end
    print("First 64 bytes:", table.concat(hex, " "))
    
    return bytecode
end

local code = [[
local x = 10
local y = 20
return x + y
]]

compile_and_show(code)
```

---

## 81.4 Opcodes ทั้งหมดใน Lua 5.4

Lua 5.4 มี opcodes ประมาณ 83 ตัว แบ่งเป็นหมวดหมู่:

### Arithmetic Opcodes

```lua
-- ตัวอย่าง arithmetic operations และ bytecode ที่ได้

-- ADDI - add immediate (Lua 5.4 ใหม่)
-- ปรับปรุงประสิทธิภาพการบวกกับ constant integer
local function add_demo()
    local a = 5
    local b = a + 1    -- ADDI R1, R0, 1
    local c = a + 10   -- ADDI R2, R0, 10
    return b + c       -- ADD R3, R1, R2
end

-- ADDK - add with constant from k table
local function addk_demo()
    local a = 3.14
    local b = a + 2.71828  -- ADDK (float constant)
    return b
end

-- ทุก arithmetic operations:
-- ADD  - addition
-- SUB  - subtraction  
-- MUL  - multiplication
-- DIV  - division (float)
-- IDIV - integer division
-- MOD  - modulo
-- POW  - power
-- UNM  - unary minus
-- BAND - bitwise AND
-- BOR  - bitwise OR
-- BXOR - bitwise XOR
-- BNOT - bitwise NOT
-- SHL  - shift left
-- SHR  - shift right

-- ตัวอย่างครบทุกอย่าง
local function all_arithmetic(a, b)
    local r1 = a + b    -- ADD
    local r2 = a - b    -- SUB
    local r3 = a * b    -- MUL
    local r4 = a / b    -- DIV
    local r5 = a // b   -- IDIV
    local r6 = a % b    -- MOD
    local r7 = a ^ b    -- POW
    local r8 = -a       -- UNM
    local r9 = a & b    -- BAND
    local r10 = a | b   -- BOR
    local r11 = a ~ b   -- BXOR
    local r12 = ~a      -- BNOT
    local r13 = a << 2  -- SHL
    local r14 = a >> 2  -- SHR
    return r1,r2,r3,r4,r5,r6,r7,r8,r9,r10,r11,r12,r13,r14
end
```

### Load Opcodes

```lua
-- MOVE - copy register to register
local function move_demo()
    local a = 10
    local b = a    -- MOVE R1, R0
    return b
end

-- LOADK - load constant
local function loadk_demo()
    local a = 42          -- LOADK R0, K(42)
    local b = "hello"     -- LOADK R1, K("hello")
    local c = 3.14        -- LOADK R2, K(3.14)
    return a, b, c
end

-- LOADKX - load large constant (index > 255)
-- ใช้เมื่อ constant index เกิน 18 bits

-- LOADBOOL - load boolean
local function loadbool_demo()
    local t = true      -- LOADBOOL R0, 1, 0
    local f = false     -- LOADBOOL R1, 0, 0
    return t, f
end

-- LOADINT - load integer (Lua 5.4)
local function loadint_demo()
    local i = 1000000   -- LOADINT R0, 1000000
    return i
end

-- LOADFLT - load float (Lua 5.4)
local function loadflt_demo()
    local f = 3.14      -- LOADFLT R0, 3.14
    return f
end

-- LOADNIL - load nil
local function loadnil_demo()
    local a, b, c  -- LOADNIL R0, 2 (load nil into R0, R1, R2)
    return a, b, c
end
```

### Table Opcodes

```lua
-- NEWTABLE - create new table
local function newtable_demo()
    local t = {}        -- NEWTABLE R0, 0, 0
    local t2 = {1,2,3} -- NEWTABLE R1, 3, 0 (hint: 3 array items)
    local t3 = {a=1}   -- NEWTABLE R2, 0, 1 (hint: 1 hash item)
end

-- GETTABLE - get from table with register key
local function gettable_demo(t, k)
    return t[k]         -- GETTABLE R2, R0, R1
end

-- SETTABLE - set in table with register key
local function settable_demo(t, k, v)
    t[k] = v            -- SETTABLE R0, R1, R2
end

-- GETFIELD - get from table with string key (constant)
local function getfield_demo(t)
    return t.name       -- GETFIELD R1, R0, K("name")
end

-- SETFIELD - set in table with string key
local function setfield_demo(t, v)
    t.name = v          -- SETFIELD R0, K("name"), R1
end

-- GETI - get from table with integer key (Lua 5.4)
local function geti_demo(t)
    return t[1]         -- GETI R1, R0, 1
end

-- SETI - set in table with integer key (Lua 5.4)
local function seti_demo(t, v)
    t[1] = v            -- SETI R0, 1, R1
end

-- GETFIELD vs GETTABLE performance:
local function benchmark_access(t, n)
    local sum = 0
    -- GETFIELD ใช้ constant key - เร็วกว่า
    for i = 1, n do
        sum = sum + t.value  -- GETFIELD
    end
    return sum
end
```

### Control Flow Opcodes

```lua
-- JMP - unconditional jump
local function jmp_demo()
    goto skip
    local x = 1  -- never executed
    ::skip::
    return 2     -- JMP to here
end

-- TEST - conditional test (ไม่มี register destination)
local function test_demo(a)
    if a then       -- TEST R0, 0 + JMP
        return 1
    end
    return 2
end

-- TESTSET - test and set
local function testset_demo(a, b)
    return a or b   -- TESTSET R2, R0, 0 (if R0 is truthy, R2 = R0, else...)
end

-- EQ, LT, LE - comparison (ไม่มี register destination, เป็น conditional JMP)
local function compare_demo(a, b)
    if a == b then return 1 end  -- EQ + JMP
    if a < b then return 2 end   -- LT + JMP
    if a <= b then return 3 end  -- LE + JMP
    return 4
end

-- FORPREP - prepare numeric for loop
-- FORLOOP - numeric for loop step
local function for_demo()
    local sum = 0
    for i = 1, 10 do  -- FORPREP, then FORLOOP
        sum = sum + i
    end
    return sum
end

-- ดู bytecode ของ for loop
--[[
LOADK    R0, 0       ; sum = 0
LOADK    R1, 1       ; i = 1 (start)
LOADK    R2, 10      ; limit = 10
LOADK    R3, 1       ; step = 1
FORPREP  R1, +4      ; prepare, jump to FORLOOP
ADD      R0, R0, R1  ; sum = sum + i  (R1 is 'i' inside loop)
FORLOOP  R1, -2      ; increment and check, jump back if continue
RETURN   R0, 2       ; return sum
]]

-- TFORPREP / TFORCALL / TFORLOOP - generic for loop
local function generic_for_demo(t)
    local result = {}
    for k, v in pairs(t) do  -- TFORPREP/TFORCALL/TFORLOOP
        result[k] = v
    end
    return result
end
```

### Function Call Opcodes

```lua
-- CALL - function call
-- CALL A B C:
--   A = register with function
--   B = number of args + 1 (0 = variable)
--   C = number of results + 1 (0 = variable)
local function call_demo()
    local f = print
    f("hello")      -- CALL R0, 2, 1 (1 arg, 0 results)
    
    local a, b = math.modf(3.7)  -- CALL ..., 2, 3 (1 arg, 2 results)
end

-- CALLK - call with immediate argument count
-- (Lua 5.4 optimization)

-- TAILCALL - tail call optimization
local function tail_demo(n)
    if n <= 0 then return 0 end
    return tail_demo(n - 1)  -- TAILCALL (TCO!)
end

-- RETURN - return from function
-- RETURN A B:
--   return R(A) to R(A+B-2)
--   B = 0 means variable return count

local function return_demo(a, b, c)
    return a, b, c  -- RETURN R0, 4 (returns 3 values)
end

-- RETURN0 - return no values (Lua 5.4)
local function return0_demo()
    -- implicit return
end  -- RETURN0

-- RETURN1 - return 1 value (Lua 5.4)  
local function return1_demo(x)
    return x * 2  -- RETURN1 R1 (เร็วกว่า RETURN สำหรับ 1 value)
end
```

---

## 81.5 Upvalues และ Closures

### การทำงานของ Upvalue

```lua
-- Upvalue คือ variable ที่ closure capture มาจาก outer scope
-- มี 2 สถานะ: open (อยู่ใน stack) และ closed (ย้ายออกจาก stack)

local function make_counter()
    local count = 0  -- จะกลายเป็น upvalue เมื่อ closure ถูกสร้าง
    
    return function()
        count = count + 1  -- access upvalue
        return count
    end
end

-- ดู upvalue structure
--[[
เมื่อ make_counter() ทำงาน:
1. สร้าง local 'count' ใน stack slot
2. สร้าง closure function
3. closure มี upvalue descriptor ที่ชี้ไป stack slot ของ count
4. เมื่อ make_counter() return, count ถูก "close" 
   (ย้ายจาก stack ไปยัง heap)
5. closure ยังชี้ไปยัง count ที่ heap
]]

-- ตัวอย่าง shared upvalue
local function shared_upvalue_demo()
    local x = 0
    
    local inc = function() x = x + 1 end
    local get = function() return x end
    
    -- inc และ get share upvalue เดียวกัน (x)
    -- เปลี่ยนผ่าน inc จะเห็นการเปลี่ยนแปลงใน get
    
    inc()
    inc()
    print(get())  -- 2
    
    return inc, get
end

-- ดู bytecode สำหรับ upvalue access
--[[
GETUPVAL R0, 0     ; R0 = upvalue[0] (อ่าน upvalue)
SETUPVAL R0, 0     ; upvalue[0] = R0 (เขียน upvalue)
]]
```

### Upvalue Opcodes

```lua
-- GETUPVAL - get upvalue
-- SETUPVAL - set upvalue
-- GETTABUP - get from table in upvalue (common for global access)
-- SETTABUP - set in table in upvalue

local x = 100  -- global ใน _ENV upvalue

local function upvalue_ops()
    -- GETTABUP ใช้สำหรับ global access
    local y = x          -- GETTABUP R0, _ENV, K("x")
    x = y + 1           -- SETTABUP _ENV, K("x"), R1
    
    return y
end

-- ดู upvalue descriptions ด้วย debug
local function show_upvalues(f)
    local i = 1
    while true do
        local name = debug.getupvalue(f, i)
        if not name then break end
        print(string.format("upvalue[%d] = %s", i, name))
        i = i + 1
    end
end

local counter = (function()
    local n = 0
    return function()
        n = n + 1
        return n
    end
end)()

show_upvalues(counter)  -- แสดง upvalue[1] = n
```

---

## 81.6 Constants Table

Constants table เก็บค่าที่ไม่เปลี่ยนแปลงของ function:

```lua
-- ตัวอย่าง constants ที่จะอยู่ใน k table
local function constants_demo()
    -- Integer constants
    local a = 42         -- K[0] = 42
    local b = 100        -- K[1] = 100
    
    -- Float constants
    local c = 3.14       -- K[2] = 3.14
    local d = 2.71828    -- K[3] = 2.71828
    
    -- String constants
    local e = "hello"    -- K[4] = "hello"
    local f = "world"    -- K[5] = "world"
    
    -- Boolean และ nil ไม่อยู่ใน k table
    -- ใช้ LOADBOOL / LOADNIL แทน
    
    return a, b, c, d, e, f
end

-- ตรวจสอบ constants ด้วย bytecode dump
local function analyze_constants(f)
    local bytecode = string.dump(f)
    -- Parse bytecode header...
    -- (ในทางปฏิบัติต้องใช้ parser library)
    print("Bytecode size:", #bytecode)
end

-- Constants sharing: Lua จะ intern strings
-- "hello" ใน 2 ที่จะใช้ pointer เดียวกัน
local s1 = "hello"
local s2 = "hello"
print(s1 == s2)  -- true
-- ใน memory: s1 และ s2 ชี้ไปยัง string object เดียวกัน
```

---

## 81.7 การ Execute ทีละ Instruction

### Main Execution Loop

```lua
-- Pseudo-code ของ Lua VM main loop (จาก lvm.c)
--[[
function luaV_execute(L):
    while true:
        instruction = *pc++
        opcode = GET_OPCODE(instruction)
        A = GETARG_A(instruction)
        B = GETARG_B(instruction)
        C = GETARG_C(instruction)
        
        switch opcode:
            case OP_MOVE:
                R(A) = R(B)
            
            case OP_LOADK:
                R(A) = K(Bx)
            
            case OP_ADD:
                if is_integer(R(B)) and is_integer(R(C)):
                    R(A) = integer_add(R(B), R(C))
                else:
                    R(A) = float_add(coerce(R(B)), coerce(R(C)))
            
            case OP_CALL:
                -- ซับซ้อนกว่า ต้อง adjust stack ฯลฯ
                luaD_call(L, R(A), C-1)
            
            case OP_RETURN:
                -- cleanup and return
                return
            
            ...
]]
```

### Instruction Encoding

```lua
-- Lua 5.4 instruction format
-- ทุก instruction เป็น 32-bit integer

-- Format iABC (ส่วนใหญ่):
-- Bits: 7(opcode) + 8(A) + 8(B) + 9(C)
-- [31..25|24..17|16..9|8..0]
-- [  C   |  B   |  A  | op ]

-- Format iABx:
-- Bits: 7(opcode) + 8(A) + 17(Bx)
-- [31..15|14..7|6..0]
-- [  Bx  |  A  | op ]

-- Format iAsBx:
-- Bits: 7(opcode) + 8(A) + 17(sBx, signed)
-- sBx = Bx - MAXARG_sBx/2 (bias encoding)

-- Format iAx:
-- Bits: 7(opcode) + 25(Ax)

-- ดู encoding
local function decode_instruction(ins)
    local op = ins & 0x7F          -- bits 0-6
    local a  = (ins >> 7) & 0xFF   -- bits 7-14
    local b  = (ins >> 23) & 0x1FF -- bits 23-31 (9 bits)
    local c  = (ins >> 14) & 0x1FF -- bits 14-22 (9 bits)
    local bx = (ins >> 14) & 0x3FFFF -- bits 14-31 (18 bits)
    
    return {op=op, a=a, b=b, c=c, bx=bx}
end

-- ตัวอย่าง instruction values
-- LOADK 0, 1  (load constant K[1] into R[0])
-- instruction = OP_LOADK | (0 << 7) | (1 << 14)
local LOADK_op = 3  -- opcode number for LOADK
local loadk_ins = LOADK_op | (0 << 7) | (1 << 14)
print(string.format("LOADK instruction: 0x%08x", loadk_ins))
```

---

## 81.8 Hot Path Optimization

Lua VM มีการ optimize hot paths หลายวิธี:

### Inline Caching

```lua
-- Table access ใช้ inline cache
-- ครั้งแรก: ต้อง hash lookup
-- ครั้งต่อไป: ใช้ cached position

local function inline_cache_demo()
    local t = {x = 1, y = 2, z = 3}
    
    -- ครั้งแรก: hash lookup สำหรับ "x"
    -- หลังจากนั้น: ตรวจสอบว่า table ยังมี shape เดิมหรือไม่
    -- ถ้าใช่: direct access
    -- ถ้าไม่: full lookup
    
    local sum = 0
    for i = 1, 1000000 do
        sum = sum + t.x + t.y + t.z
    end
    return sum
end

-- Metatable caching
local mt = {
    __index = function(t, k)
        -- ถูก cache หลังจาก first access
        return rawget(t, k)
    end
}
```

### Numeric For Loop Optimization

```lua
-- Numeric for loop ถูก optimize เป็นพิเศษ
-- ไม่มี iterator function call
-- ทำงานโดยตรงบน registers

local function optimized_for()
    local sum = 0
    -- นี่คือ tight loop ที่ Lua optimize ได้ดี
    for i = 1, 1000000 do
        sum = sum + i
    end
    return sum
end

-- เทียบกับ generic for (ช้ากว่า)
local function generic_for_version()
    local sum = 0
    local function range(n)
        local i = 0
        return function()
            i = i + 1
            if i <= n then return i end
        end
    end
    
    for i in range(1000000) do
        sum = sum + i
    end
    return sum
end

-- Benchmark
local function benchmark(f, name, n)
    n = n or 5
    local times = {}
    for i = 1, n do
        local t1 = os.clock()
        f()
        local t2 = os.clock()
        times[i] = t2 - t1
    end
    table.sort(times)
    print(string.format("%s: min=%.4f avg=%.4f max=%.4f", 
        name, times[1], 
        (function() local s=0; for _,v in ipairs(times) do s=s+v end; return s/#times end)(),
        times[#times]))
end

benchmark(optimized_for, "numeric for")
benchmark(generic_for_version, "generic for")
```

### Type Specialization

```lua
-- Lua 5.4 แยก integer และ float operations
-- ทำให้ integer arithmetic เร็วขึ้นมาก

local function integer_ops(n)
    local sum = 0
    for i = 1, n do
        sum = sum + i    -- integer ADD
        sum = sum * 2    -- integer MUL
        sum = sum // 3   -- integer IDIV
    end
    return sum
end

local function float_ops(n)
    local sum = 0.0
    for i = 1, n do
        sum = sum + i    -- float ADD (slower)
        sum = sum * 2.0  -- float MUL
        sum = sum / 3.0  -- float DIV
    end
    return sum
end

-- Integer ops จะเร็วกว่าเพราะไม่ต้อง FPU
```

---

## 81.9 Bytecode Portability

```lua
-- Lua bytecode ไม่ portable ระหว่าง:
-- - Lua versions ต่างกัน
-- - 32-bit vs 64-bit systems
-- - Big-endian vs little-endian

-- Bytecode format header:
--[[
Byte 0: 0x1B (ESC)
Byte 1: 0x4C ('L')
Byte 2: 0x75 ('u')
Byte 3: 0x61 ('a')
Byte 4: Version (0x54 = Lua 5.4)
Byte 5: Format (0 = official)
Byte 6-11: LUAC_DATA (0x19 0x93 0x0D 0x0A 0x1A 0x0A)
Byte 12: Size of instruction (4)
Byte 13: Size of lua_Integer (8 on 64-bit)
Byte 14: Size of lua_Number (8)
Byte 15-22: Integer test value (0x5678)
Byte 23-30: Float test value (370.5)
]]

local function check_bytecode_header(bytecode)
    if #bytecode < 31 then
        return false, "too short"
    end
    
    -- ตรวจสอบ magic
    if bytecode:sub(1, 4) ~= "\x1BLua" then
        return false, "invalid magic"
    end
    
    -- ตรวจสอบ version
    local version = bytecode:byte(5)
    if version ~= 0x54 then
        return false, string.format("wrong version: %02x", version)
    end
    
    return true, "valid"
end

-- ตัวอย่างการ load bytecode
local function load_bytecode_example()
    local source = [[
        local x = 42
        return x * 2
    ]]
    
    -- Compile
    local f = load(source)
    local bytecode = string.dump(f)
    
    -- ตรวจสอบ
    local ok, msg = check_bytecode_header(bytecode)
    print("Valid:", ok, msg)
    
    -- Load กลับ
    local f2 = load(bytecode)
    print("Result:", f2())  -- 84
end

load_bytecode_example()
```

---

## 81.10 การแก้ไข Bytecode (ระวัง!)

```lua
-- WARNING: การแก้ไข bytecode โดยตรงเป็นเรื่องที่อันตรายมาก
-- ใช้เพื่อการศึกษาเท่านั้น

-- ตัวอย่าง: แก้ไข constant ใน bytecode
-- (ต้องใช้ library พิเศษ เช่น luac parser)

-- แนวทาง safe กว่า: ใช้ debug hooks
local function bytecode_inspection()
    local function target(x)
        return x * 2
    end
    
    -- Dump bytecode
    local bc = string.dump(target)
    
    -- ดู hex
    local lines = {}
    for i = 1, #bc, 16 do
        local hex = {}
        local chars = {}
        for j = i, math.min(i + 15, #bc) do
            local b = bc:byte(j)
            hex[#hex+1] = string.format("%02x", b)
            chars[#chars+1] = (b >= 32 and b < 127) and string.char(b) or "."
        end
        lines[#lines+1] = string.format("%04x: %-48s  %s",
            i-1, table.concat(hex, " "), table.concat(chars))
    end
    
    print(table.concat(lines, "\n"))
end

bytecode_inspection()
```

---

## 81.11 Source Information และ Debug Info

```lua
-- Lua เก็บ source info สำหรับ error messages และ debugger

local function source_info_demo()
    -- debug.getinfo ให้ข้อมูล source
    local function get_caller_info()
        local info = debug.getinfo(2, "Sln")
        return info
    end
    
    local info = get_caller_info()
    print("Source:", info.source)
    print("Short source:", info.short_src)
    print("Current line:", info.currentline)
    print("What:", info.what)  -- "Lua", "C", "main"
    print("Name:", info.name)
    print("Name what:", info.namewhat)
end

source_info_demo()

-- ลด debug info เพื่อ performance
-- luac -s ลบ debug info ออก
-- string.dump(f, true) ลบ debug info

local function strip_debug_info()
    local f = load([[
        local function foo(x)
            return x * 2
        end
        return foo(21)
    ]])
    
    -- พร้อม debug info (ใหญ่กว่า)
    local with_debug = string.dump(f)
    
    -- ไม่มี debug info (เล็กกว่า)
    local without_debug = string.dump(f, true)
    
    print("With debug:", #with_debug, "bytes")
    print("Without debug:", #without_debug, "bytes")
    
    -- Load กลับทั้งสอง
    local f2 = load(without_debug)
    print("Result:", f2())  -- ยังทำงานได้
end

strip_debug_info()
```

---

## 81.12 Local Variable Information

```lua
-- LocVar structure เก็บข้อมูล local variable
--[[
struct LocVar {
    TString *varname;  -- ชื่อ variable
    int startpc;       -- instruction แรกที่ variable มีชีวิต
    int endpc;         -- instruction สุดท้าย
};
]]

local function locvar_demo()
    local function show_locals(f, pc)
        -- debug.getlocal ให้ข้อมูล local ณ position ปัจจุบัน
        local i = 1
        while true do
            local name, val = debug.getlocal(f, i)
            if not name then break end
            -- Variables ที่เริ่มด้วย "(" คือ temp variables
            if name:sub(1,1) ~= "(" then
                print(string.format("  local[%d] %s = %s", i, name, tostring(val)))
            end
            i = i + 1
        end
    end
    
    -- ดู locals ระหว่าง execution
    debug.sethook(function()
        local info = debug.getinfo(2)
        if info.what == "Lua" then
            -- show_locals(2, info.currentline)
        end
    end, "l")
    
    local a = 10
    local b = 20
    local c = a + b
    
    debug.sethook()  -- remove hook
    return c
end

-- ตัวอย่าง dead variable elimination
local function dead_variable_example()
    local x = 10    -- used
    local y = 20    -- NOT used (dead)
    local z = x + 5 -- used
    return z
    -- compiler จะ optimize y ออก
end
```

---

## 81.13 Proto List (Nested Functions)

```lua
-- Function ที่ define ภายใน function อื่นจะอยู่ใน p[] array

local function outer()
    -- inner functions อยู่ใน p[] ของ outer
    local function inner1(x)
        return x + 1
    end
    
    local function inner2(x)
        return x * 2
    end
    
    -- closure ต่างกัน แต่ใช้ proto เดียวกัน
    local c1 = inner1
    local c2 = inner1  -- same proto, different closure
    
    return inner1(5), inner2(5)
end

-- ดู nested protos
local function analyze_nested(f, depth)
    depth = depth or 0
    local prefix = string.rep("  ", depth)
    local info = debug.getinfo(f, "Snu")
    
    print(prefix .. "Function at " .. info.source .. ":" .. info.linedefined)
    print(prefix .. "  params=" .. info.nparams .. " upvalues=" .. info.nups)
end

-- CLOSURE opcode สร้าง closure จาก proto
--[[
CLOSURE R(A), Bx
  R(A) = closure(proto[Bx])
  
  Pseudo-code:
  function luaV_execute_CLOSURE(A, Bx):
    proto = current_proto->p[Bx]
    closure = luaF_newLclosure(L, proto)
    for each upvalue in proto:
        if upvalue.instack:
            closure->upvals[i] = findupval(L, base + upvalue.idx)
        else:
            closure->upvals[i] = enclosing_closure->upvals[upvalue.idx]
    R(A) = closure
]]
```

---

## 81.14 Decompilation

```lua
-- ไม่มี official decompiler สำหรับ Lua
-- แต่มี tools เช่น unluac, luadec

-- Decompilation challenges:
-- 1. ชื่อ local variable ถูกลบ (ถ้า strip debug info)
-- 2. Goto ถูก reconstruct เป็น control flow
-- 3. Short-circuit evaluation ซับซ้อน
-- 4. Multiple return values ติดตามยาก

-- แนวทางทำ simple decompiler

local opcodes = {
    [0]  = "MOVE",
    [1]  = "LOADI",
    [2]  = "LOADF",
    [3]  = "LOADK",
    [4]  = "LOADKX",
    [5]  = "LOADBOOL", -- Lua 5.3
    [6]  = "LOADNIL",
    [7]  = "GETUPVAL",
    [8]  = "SETUPVAL",
    [9]  = "GETTABUP",
    [10] = "GETTABLE",
    [11] = "GETI",
    [12] = "GETFIELD",
    -- ... และอีกมาก
}

local function simple_disasm(bytecode)
    -- Skip header (ประมาณ 31+ bytes)
    -- Parse instructions
    -- นี่เป็น simplified version
    
    local function read_int32(data, pos)
        local b1, b2, b3, b4 = data:byte(pos, pos+3)
        return b1 | (b2 << 8) | (b3 << 16) | (b4 << 24)
    end
    
    -- จริงๆ ต้องใช้ proper bytecode parser
    -- เช่น luadbg หรือ write bytecode parser เอง
    print("Simple disassembler (educational)")
end
```

---

## 81.15 Understanding Compilation Flow

```lua
-- Compilation phases ใน Lua

-- Phase 1: Lexical Analysis (llex.c)
-- - แปลง source text เป็น tokens
-- - ตัดสตริง, ตัวเลข, keywords ออก

-- Phase 2: Parsing (lparser.c)
-- - Recursive descent parser
-- - สร้าง AST ไปพร้อมกับ code generation

-- Phase 3: Code Generation (lcode.c)
-- - แปลง AST nodes เป็น bytecode
-- - Register allocation
-- - Constant folding

-- ตัวอย่าง: constant folding
local function constant_folding_demo()
    -- Lua compiler ทำ constant folding ณ compile time
    local x = 2 + 3      -- คำนวณเป็น 5 ณ compile time
    local y = 10 * 20    -- คำนวณเป็น 200
    local z = "hello" .. " " .. "world"  -- จะ fold หรือไม่?
    
    -- String concatenation ไม่ได้ fold เสมอไป
    -- แต่ตัวเลขจะ fold
    
    return x, y, z
end

-- Constant folding ตรวจสอบได้ด้วย luac -l
-- ถ้า fold แล้ว จะเห็น LOADK 5 แทนที่จะเป็น ADD 2, 3

-- Phase 4: Register Allocation
--[[
Lua ใช้ simple linear scan allocation:
- Register ต่ำๆ สำหรับ local variables
- Register สูงๆ สำหรับ temporaries
- เมื่อ temporary ไม่ใช้แล้ว register ถูก reclaim

ตัวอย่าง:
    local a = 1  -- R0
    local b = 2  -- R1
    local c = a + b + a * b
    
    R2 = a + b    (temp)
    R3 = a * b    (temp)
    R2 = R2 + R3  (temp, recycle R2)
    local c = R2  -- R2 (แต่ c ก็คือ R2)
]]
```

---

## 81.16 Performance Profiling ด้วย VM Hooks

```lua
-- ใช้ debug hooks เพื่อ profile code

local function profiler()
    local counts = {}
    local call_stack = {}
    
    local function hook(event, line)
        local info = debug.getinfo(2, "Sn")
        local key = (info.source or "?") .. ":" .. (info.name or "?")
        
        if event == "call" then
            call_stack[#call_stack + 1] = {key=key, start=os.clock()}
        elseif event == "return" then
            if #call_stack > 0 then
                local frame = table.remove(call_stack)
                local elapsed = os.clock() - frame.start
                counts[frame.key] = (counts[frame.key] or 0) + elapsed
            end
        end
    end
    
    return {
        start = function()
            debug.sethook(hook, "cr")
        end,
        stop = function()
            debug.sethook()
        end,
        report = function()
            local sorted = {}
            for k, v in pairs(counts) do
                sorted[#sorted+1] = {name=k, time=v}
            end
            table.sort(sorted, function(a,b) return a.time > b.time end)
            
            print("\n=== Profile Report ===")
            for i, entry in ipairs(sorted) do
                if i > 20 then break end
                print(string.format("  %8.4fs  %s", entry.time, entry.name))
            end
        end
    }
end

-- ใช้ profiler
local p = profiler()
p.start()

-- code to profile
local sum = 0
for i = 1, 10000 do
    sum = sum + math.sqrt(i)
end

p.stop()
p.report()
```

---

## 81.17 VM Register Allocation Details

```lua
-- ทำความเข้าใจการ allocate registers

-- Rule 1: Local variables ได้ registers ที่เล็กที่สุดที่ว่าง
-- Rule 2: Temporary ได้ registers หลัง locals
-- Rule 3: Function call: function+args ต้องอยู่ติดกัน

local function register_layout_demo()
    -- Stack layout:
    --   R0: a (local)
    --   R1: b (local)
    --   R2: c (local)
    --   R3: f (temp function for print)
    --   R4: arg (temp for first arg)
    
    local a = 1
    local b = 2
    local c = 3
    
    print(a, b, c)
    -- R3 = _ENV["print"]  (GETTABUP)
    -- R4 = a              (MOVE)
    -- R5 = b              (MOVE)
    -- R6 = c              (MOVE)
    -- CALL R3, 4, 1       (call with 3 args, 0 results)
end

-- Multiple assignment register tricks
local function multi_assign_demo()
    -- a, b = b, a  (swap ต้องระวัง register)
    local a = 1
    local b = 2
    
    -- Lua รับประกันว่า right side evaluate ก่อน
    -- ดังนั้น a, b = b, a ทำงานถูกต้องเสมอ
    a, b = b, a
    
    -- bytecode:
    -- MOVE T0, R1  (T0 = b)
    -- MOVE T1, R0  (T1 = a)
    -- MOVE R0, T0  (a = T0)
    -- MOVE R1, T1  (b = T1)
    
    print(a, b)  -- 2, 1
end

multi_assign_demo()
```

---

## 81.18 การวิเคราะห์ Performance จาก Bytecode

```lua
-- เทคนิคการอ่าน bytecode เพื่อ optimize

-- ตัวอย่าง 1: การ access global ช้ากว่า local
local sqrt = math.sqrt  -- cache ไว้ใน local

local function benchmark_global_vs_local(n)
    local t1 = os.clock()
    local sum1 = 0
    for i = 1, n do
        sum1 = sum1 + math.sqrt(i)  -- GETTABUP ทุก iteration
    end
    local t2 = os.clock()
    
    local sum2 = 0
    for i = 1, n do
        sum2 = sum2 + sqrt(i)  -- MOVE เพียงครั้งเดียว ใน register
    end
    local t3 = os.clock()
    
    print(string.format("global: %.3f, local: %.3f", t2-t1, t3-t2))
end

benchmark_global_vs_local(1000000)

-- ตัวอย่าง 2: Table access pattern
local function table_access_patterns()
    local t = {}
    for i = 1, 1000 do t[i] = i end
    
    local t1 = os.clock()
    local sum1 = 0
    for i = 1, 1000000 do
        -- Sequential access (cache friendly)
        sum1 = sum1 + t[i % 1000 + 1]
    end
    local t2 = os.clock()
    
    print(string.format("Sequential: %.3f", t2-t1))
end

-- ตัวอย่าง 3: String vs integer keys
local function key_type_benchmark()
    local t1 = {a=1, b=2, c=3}
    local t2 = {[1]=1, [2]=2, [3]=3}
    
    local n = 1000000
    local start
    
    start = os.clock()
    local s = 0
    for i = 1, n do
        s = s + t1.a + t1.b + t1.c
    end
    print(string.format("String keys: %.3f", os.clock()-start))
    
    start = os.clock()
    s = 0
    for i = 1, n do
        s = s + t2[1] + t2[2] + t2[3]
    end
    print(string.format("Integer keys: %.3f", os.clock()-start))
end
```

---

## 81.19 ตัวอย่างการ Debug ด้วย VM Knowledge

```lua
-- การใช้ความรู้ VM เพื่อ debug

-- ตัวอย่าง: เข้าใจ nil error
local function nil_error_debug()
    local t = nil
    
    -- pcall catch error
    local ok, err = pcall(function()
        return t.x  -- attempt to index a nil value
    end)
    
    print("Error:", err)
    -- Lua บอก "attempt to index a nil value (local 't')"
    -- เพราะ debug info บอกชื่อ variable
end

-- ตัวอย่าง: Stack overflow
local function stack_overflow_demo()
    local function recurse(n)
        return recurse(n + 1)  -- ไม่มี base case
    end
    
    local ok, err = pcall(recurse, 0)
    print("Error:", err)  -- "stack overflow"
    
    -- Lua มี stack limit (default 200 levels)
    -- แต่ละ call ใช้ CallInfo struct
end

-- ตัวอย่าง: Upvalue lifetime
local function upvalue_lifetime()
    local closures = {}
    
    for i = 1, 5 do
        -- Bug: ทุก closure capture ตัวแปรเดียวกัน!
        closures[i] = function() return i end
    end
    
    -- ทุก closure return ค่าเดียวกัน (5)
    for _, f in ipairs(closures) do
        io.write(f() .. " ")
    end
    print()
    
    -- Fix: ใช้ IIFE
    for i = 1, 5 do
        closures[i] = (function(captured_i)
            return function() return captured_i end
        end)(i)
    end
    
    for _, f in ipairs(closures) do
        io.write(f() .. " ")
    end
    print()
end

upvalue_lifetime()
```

---

## 81.20 Advanced Bytecode Analysis Tools

```lua
-- เขียน bytecode analyzer ใน Lua

local BytecodeAnalyzer = {}
BytecodeAnalyzer.__index = BytecodeAnalyzer

function BytecodeAnalyzer.new(f)
    local self = setmetatable({}, BytecodeAnalyzer)
    self.func = f
    self.bytecode = string.dump(f)
    return self
end

function BytecodeAnalyzer:get_size()
    return #self.bytecode
end

function BytecodeAnalyzer:get_header_info()
    local bc = self.bytecode
    return {
        magic = bc:sub(1, 4),
        version = bc:byte(5),
        format = bc:byte(6),
        size_instruction = bc:byte(13),
        size_integer = bc:byte(14),
        size_number = bc:byte(15),
    }
end

function BytecodeAnalyzer:get_function_info()
    local info = debug.getinfo(self.func, "Snu")
    return {
        source = info.source,
        line_start = info.linedefined,
        line_end = info.lastlinedefined,
        num_params = info.nparams,
        is_vararg = info.isvararg,
        num_upvalues = info.nups,
    }
end

function BytecodeAnalyzer:count_upvalues()
    local count = 0
    while debug.getupvalue(self.func, count + 1) do
        count = count + 1
    end
    return count
end

function BytecodeAnalyzer:report()
    print("=== Bytecode Analysis ===")
    local info = self:get_function_info()
    print(string.format("Source: %s:%d-%d", 
        info.source, info.line_start, info.line_end))
    print(string.format("Parameters: %d%s", 
        info.num_params, info.is_vararg and " (vararg)" or ""))
    print(string.format("Upvalues: %d", info.num_upvalues))
    print(string.format("Bytecode size: %d bytes", self:get_size()))
end

-- ตัวอย่างการใช้
local function test_function(a, b, ...)
    local x = a + b
    return x * 2
end

local analyzer = BytecodeAnalyzer.new(test_function)
analyzer:report()
```

---

## 81.21 Lua VM และ Coroutines

```lua
-- Coroutines มีผลต่อ VM อย่างไร

-- แต่ละ coroutine มี lua_State แยกกัน
-- แต่ share global_State เดียวกัน

local function coroutine_vm_demo()
    -- สร้าง coroutine
    local co = coroutine.create(function(x)
        print("coroutine started with:", x)
        local y = coroutine.yield(x * 2)
        print("resumed with:", y)
        return y + x
    end)
    
    -- resume ครั้งแรก
    local ok, val = coroutine.resume(co, 10)
    print("yielded:", val)  -- 20
    
    -- resume ครั้งที่สอง
    ok, val = coroutine.resume(co, 100)
    print("returned:", val)  -- 110
    
    -- ใน VM:
    -- YIELD opcode บันทึก state และ return ไป caller
    -- RESUME ฟื้นคืน state และ continue
end

coroutine_vm_demo()

-- Coroutine pool สำหรับ performance
local coroutine_pool = {}
local pool_size = 10

local function get_coroutine(f)
    -- reuse coroutine ถ้ามีใน pool
    local co = table.remove(coroutine_pool)
    if not co then
        co = coroutine.create(f)
    end
    return co
end

-- Note: coroutine ที่ dead แล้ว ไม่สามารถ reuse ได้
-- ต้อง wrap ใน loop เพื่อ reuse จริงๆ
local function recyclable_coroutine(f)
    return coroutine.create(function(...)
        while true do
            local args = {...}
            local results = {f(table.unpack(args))}
            -- yield results แล้วรับ args ใหม่
            local new_args = {coroutine.yield(table.unpack(results))}
            -- ... (ซับซ้อนกว่านี้จริงๆ)
        end
    end)
end
```

---

## 81.22 String Interning

```lua
-- Lua intern (deduplicate) strings โดยอัตโนมัติ
-- Short strings (< 40 bytes): always interned
-- Long strings: interned lazily

-- ผลกระทบ: string comparison เป็น O(1) สำหรับ interned strings

local function string_intern_demo()
    -- ทุก "hello" ในโปรแกรม Lua จะเป็น pointer เดียวกัน
    local a = "hello"
    local b = "hell" .. "o"  -- สร้างใหม่แต่ intern ไว้
    local c = string.format("%s", "hello")  -- อาจ intern
    
    -- ไม่มีทางดู pointer โดยตรงใน Lua
    -- แต่ comparison เร็วมาก
    print(a == b)  -- true (pointer comparison!)
    print(a == c)  -- true
end

-- String interning มีผลต่อ GC
local function string_gc_demo()
    -- สร้าง strings จำนวนมาก
    local strings = {}
    for i = 1, 10000 do
        strings[i] = "prefix_" .. i  -- สร้าง string ใหม่ทุกครั้ง
    end
    
    -- Force GC
    collectgarbage("collect")
    print("GC count:", collectgarbage("count"), "KB")
    
    -- Clear references
    strings = nil
    collectgarbage("collect")
    print("After GC:", collectgarbage("count"), "KB")
end
```

---

## 81.23 ตัวอย่าง VM Internals ขั้นสูง

```lua
-- ดู CallInfo structure ผ่าน debug

local function callinfo_demo()
    local function level3()
        -- ดู call stack
        local level = 1
        while true do
            local info = debug.getinfo(level, "Sln")
            if not info then break end
            print(string.format("Level %d: %s @ line %d (%s)", 
                level,
                info.name or "?",
                info.currentline or -1,
                info.what))
            level = level + 1
        end
    end
    
    local function level2()
        level3()
    end
    
    local function level1()
        level2()
    end
    
    level1()
end

callinfo_demo()

-- VM Hooks ทุกประเภท
local function hook_types_demo()
    local function all_hook(event, line)
        local info = debug.getinfo(2, "n")
        print(string.format("[%s] %s line=%s", 
            event, 
            info and info.name or "?",
            tostring(line)))
    end
    
    -- "c" = call events
    -- "r" = return events  
    -- "l" = line events
    -- "count" = every N instructions
    
    debug.sethook(all_hook, "crl")
    
    local function small_func(x)
        return x * 2
    end
    
    small_func(5)
    
    debug.sethook()
end

-- ไม่ run ถ้าจะ flood output
-- hook_types_demo()
```

---

## 81.24 GC Integration

```lua
-- VM integration กับ Garbage Collector

-- GC modes ใน Lua 5.4:
-- incremental GC (default)
-- generational GC (experimental)

local function gc_demo()
    -- ดู GC stats
    local before = collectgarbage("count")
    
    -- สร้าง objects จำนวนมาก
    local t = {}
    for i = 1, 100000 do
        t[i] = {value = i, name = "item_" .. i}
    end
    
    local after_alloc = collectgarbage("count")
    print(string.format("Allocated: %.1f KB", after_alloc - before))
    
    -- Release
    t = nil
    collectgarbage("collect")
    
    local after_gc = collectgarbage("count")
    print(string.format("After GC: %.1f KB", after_gc - before))
end

gc_demo()

-- GC tuning
collectgarbage("setpause", 100)      -- default: 200 (เมื่อไหร่จะ run GC ครั้งต่อไป)
collectgarbage("setstepmul", 100)    -- default: 100 (GC speed vs allocation speed)

-- Generational GC (Lua 5.4+)
-- collectgarbage("generational", 0, 0)  -- minor_mul, major_mul
-- ดีสำหรับ apps ที่ objects ส่วนมาก short-lived
```

---

## 81.25 สรุป VM Internals

```lua
-- สรุปสิ่งที่ควรรู้เกี่ยวกับ Lua VM

-- 1. Register-based: efficient instruction execution
-- 2. Bytecode portability: ไม่ portable ระหว่าง versions/architectures
-- 3. Upvalues: mechanism สำหรับ closures
-- 4. Constants table: เก็บ literals ใน function
-- 5. Stack ใน VM: ทั้ง data และ call stack

-- Performance tips จาก VM knowledge:
local performance_tips = {
    "cache globals ใน locals",
    "ใช้ numeric for แทน generic for ถ้าทำได้",
    "pre-allocate tables ด้วย {} hint",
    "ใช้ integer arithmetic แทน float",
    "หลีก closures ใน hot loops",
    "ใช้ local references สำหรับ frequently-used functions",
    "ทำความเข้าใจ upvalue overhead",
}

for i, tip in ipairs(performance_tips) do
    print(i .. ". " .. tip)
end

-- Tools ที่ควรรู้:
-- luac -l : disassemble bytecode
-- debug library: runtime introspection
-- string.dump: access bytecode
-- collectgarbage: GC control
```

---

## แบบฝึกหัด

1. ใช้ `luac -l` วิเคราะห์ bytecode ของ fibonacci function และอธิบายแต่ละ instruction

2. เขียน profiler ที่นับ instruction execution count โดยใช้ debug hooks

3. สร้างฟังก์ชันที่ตรวจสอบว่าฟังก์ชันสองตัวมี prototype เดียวกัน

4. วิเคราะห์ว่าการใช้ `table.insert` vs direct index assignment มี bytecode แตกต่างกันอย่างไร

5. เขียนโปรแกรมที่ dump bytecode header และตรวจสอบ Lua version

6. วิเคราะห์ upvalue sharing ระหว่าง closures หลายตัวที่สร้างจาก loop เดียวกัน

7. เปรียบเทียบ bytecode ของ `if x > 0 then` กับ `if not (x <= 0) then`

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Lua VM internals อย่างลึกซึ้ง ตั้งแต่:
- โครงสร้าง register-based VM
- Bytecode format และ function prototype
- Opcodes ต่างๆ และความหมาย
- การทำงานของ upvalues และ closures
- Hot path optimization
- Bytecode portability issues
- Debug info และ source information
- การ profile ด้วย debug hooks
- GC integration

ความรู้เหล่านี้ช่วยให้เราเขียนโค้ด Lua ที่มีประสิทธิภาพสูงขึ้น เข้าใจ error messages ได้ดีขึ้น และสามารถ debug ปัญหาซับซ้อนได้อย่างมีประสิทธิภาพ

ในบทต่อไป (บทที่ 82) เราจะไปดู LuaJIT ซึ่งเป็น Just-In-Time compiler สำหรับ Lua ที่สามารถทำให้โค้ดทำงานได้เร็วขึ้นหลายเท่าตัว
