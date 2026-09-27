# บทที่ 43: C Extension - Lua C API Basics

## บทนำ

Lua เป็นภาษาที่ออกแบบมาเพื่อทำงานร่วมกับ C ได้อย่างดีเยี่ยม C API ของ Lua ช่วยให้เราขยายความสามารถของ Lua ด้วยโค้ด C ซึ่งทำให้ได้ performance ที่ดีขึ้น และสามารถเข้าถึง system libraries ได้

ในบทนี้เราจะเรียนรู้การใช้งาน Lua C API ตั้งแต่พื้นฐาน การสร้าง C modules ที่ Lua สามารถ require() ได้

---

## 43.1 ทำไมต้องขยาย Lua ด้วย C

### เหตุผลที่ควรใช้ C Extension:

1. **Performance** - C เร็วกว่า Lua มากสำหรับงาน compute-intensive
2. **System Access** - เข้าถึง OS APIs, hardware interfaces
3. **Existing Libraries** - ใช้ C/C++ libraries ที่มีอยู่แล้ว
4. **Bit Operations** - ก่อน Lua 5.3 ไม่มี bitwise operators
5. **Binary Data** - จัดการ raw bytes ได้ดีกว่า

### ตัวอย่างที่ 1: Lua code ที่ slow (candidate for C extension)

```lua
-- slow_lua.lua
-- Operations ที่ควรทำใน C

-- CRC32 ใน Lua (ช้า)
local function crc32_lua(data)
    local table = {}
    for i = 0, 255 do
        local crc = i
        for _ = 1, 8 do
            if crc & 1 == 1 then
                crc = (crc >> 1) ~ 0xEDB88320
            else
                crc = crc >> 1
            end
        end
        table[i] = crc
    end
    
    local crc = 0xFFFFFFFF
    for i = 1, #data do
        crc = (crc >> 8) ~ table[(crc ~ data:byte(i)) & 0xFF]
    end
    return ~crc & 0xFFFFFFFF
end

-- Benchmark
local testData = string.rep("Hello World! ", 10000)
local start = os.clock()
for _ = 1, 100 do
    crc32_lua(testData)
end
local elapsed = os.clock() - start
print(string.format("Lua CRC32: %.4fs for 100 iterations", elapsed))
print("A C implementation would be 10-50x faster")

-- SHA-like hash (demo ว่า C จะเหมาะกว่า)
local function simpleMix(data)
    local h = 0x811c9dc5
    for i = 1, #data do
        h = h ~ data:byte(i)
        h = (h * 16777619) & 0xFFFFFFFF
    end
    return h
end

local hash = simpleMix(testData)
print(string.format("Hash: 0x%08X", hash))
```

---

## 43.2 Lua C API Overview

### ตัวอย่างที่ 2: Hello World C Extension

```c
/* hello.c - บทเรียนแรกของ Lua C API */

/*
 * compile:
 *   Linux:   gcc -shared -fPIC -o hello.so hello.c $(lua-5.4-config --cflags --libs)
 *   macOS:   gcc -bundle -undefined dynamic_lookup -o hello.so hello.c $(lua-5.4-config --cflags)
 *   Windows: gcc -shared -o hello.dll hello.c -I"lua/include" -L"lua" -llua54
 */

#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

/* C function ที่ Lua สามารถเรียกได้
 * ทุก C function ต้องมี signature: int function_name(lua_State *L)
 * return value คือจำนวน values ที่ push ลง stack (return values)
 */
static int lua_hello(lua_State *L) {
    /* ดึง argument แรกจาก stack (string) */
    const char *name = luaL_optstring(L, 1, "World");
    
    /* สร้าง greeting string */
    lua_pushfstring(L, "Hello, %s! from C", name);
    
    /* return 1 value (string ที่เพิ่ง push) */
    return 1;
}

static int lua_add(lua_State *L) {
    /* ดึง arguments */
    double a = luaL_checknumber(L, 1);  /* arg 1 */
    double b = luaL_checknumber(L, 2);  /* arg 2 */
    
    /* push result */
    lua_pushnumber(L, a + b);
    return 1;
}

/* Registration table: map Lua function name -> C function */
static const luaL_Reg hello_lib[] = {
    {"hello", lua_hello},
    {"add",   lua_add},
    {NULL, NULL}  /* sentinel: end of list */
};

/* Entry point เมื่อ require("hello") ถูกเรียก */
int luaopen_hello(lua_State *L) {
    luaL_newlib(L, hello_lib);
    return 1;
}
```

```lua
-- test_hello.lua (ถ้า compile แล้ว)
-- local hello = require("hello")
-- print(hello.hello("Lua"))      -- Hello, Lua! from C
-- print(hello.hello())           -- Hello, World! from C
-- print(hello.add(3.14, 2.71))   -- 5.85
```

---

## 43.3 lua_State

### ตัวอย่างที่ 3: Creating and Using lua_State

```c
/* lua_state_demo.c */

#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    /* สร้าง Lua state ใหม่
     * lua_State เป็น struct ที่เก็บสถานะทั้งหมดของ Lua interpreter
     * ทุก Lua state เป็น independent กัน (no global state)
     */
    lua_State *L = luaL_newstate();
    if (L == NULL) {
        fprintf(stderr, "Cannot create Lua state\n");
        return EXIT_FAILURE;
    }
    
    /* เปิด standard libraries (io, math, string, etc.) */
    luaL_openlibs(L);
    
    /* รัน Lua string */
    const char *code = "print('Hello from embedded Lua!')";
    if (luaL_dostring(L, code) != LUA_OK) {
        fprintf(stderr, "Error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
    }
    
    /* ปิด Lua state และ free memory */
    lua_close(L);
    
    printf("Lua state closed successfully\n");
    return EXIT_SUCCESS;
}

/* 
 * Compile: gcc -o demo lua_state_demo.c -llua -lm -ldl
 * Run:     ./demo
 */
```

### ตัวอย่างที่ 4: Multiple lua_States

```c
/* multiple_states.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

void runScript(lua_State *L, const char *script, const char *name) {
    printf("=== Running in %s ===\n", name);
    if (luaL_dostring(L, script) != LUA_OK) {
        fprintf(stderr, "Error in %s: %s\n", name, lua_tostring(L, -1));
        lua_pop(L, 1);
    }
}

int main(void) {
    /* สร้าง 2 Lua states แยกกัน - แต่ละอันมี global scope ของตัวเอง */
    lua_State *L1 = luaL_newstate();
    lua_State *L2 = luaL_newstate();
    
    luaL_openlibs(L1);
    luaL_openlibs(L2);
    
    /* ตั้งค่า global variables แยกกัน */
    luaL_dostring(L1, "x = 100; name = 'State A'");
    luaL_dostring(L2, "x = 200; name = 'State B'");
    
    /* แต่ละ state มี globals ของตัวเอง */
    runScript(L1, "print('x =', x, 'name =', name)", "L1");
    runScript(L2, "print('x =', x, 'name =', name)", "L2");
    
    /* States ไม่แชร์ globals */
    runScript(L1, "print('x in L1 still:', x)", "L1");
    
    lua_close(L1);
    lua_close(L2);
    
    printf("Both states closed\n");
    return 0;
}
```

---

## 43.4 The Lua Stack

Stack เป็น mechanism หลักที่ C และ Lua ใช้แลกเปลี่ยนข้อมูลกัน

### ตัวอย่างที่ 5: Stack Basics

```c
/* stack_demo.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

void printStack(lua_State *L, const char *label) {
    int n = lua_gettop(L);
    printf("\n--- Stack: %s (size=%d) ---\n", label, n);
    for (int i = 1; i <= n; i++) {
        int type = lua_type(L, i);
        const char *typeName = lua_typename(L, type);
        printf("  [%d] %s: ", i, typeName);
        
        switch (type) {
            case LUA_TNIL:
                printf("nil");
                break;
            case LUA_TBOOLEAN:
                printf("%s", lua_toboolean(L, i) ? "true" : "false");
                break;
            case LUA_TNUMBER:
                if (lua_isinteger(L, i)) {
                    printf("%lld", (long long)lua_tointeger(L, i));
                } else {
                    printf("%g", lua_tonumber(L, i));
                }
                break;
            case LUA_TSTRING:
                printf("%q", lua_tostring(L, i));
                break;
            case LUA_TTABLE:
                printf("table at %p", lua_topointer(L, i));
                break;
            default:
                printf("?");
        }
        printf("\n");
    }
}

int main(void) {
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    
    printStack(L, "empty");
    
    /* Push values */
    lua_pushnil(L);
    lua_pushboolean(L, 1);
    lua_pushnumber(L, 3.14);
    lua_pushstring(L, "hello");
    lua_pushinteger(L, 42);
    
    printStack(L, "after pushes");
    
    /* lua_gettop: จำนวน elements ใน stack */
    printf("\nStack size: %d\n", lua_gettop(L));
    
    /* Negative indices: -1 = top, -2 = second from top */
    printf("Top element (-1): %s\n", lua_tostring(L, -1));
    printf("Second (-2): %s\n", lua_typename(L, lua_type(L, -2)));
    
    /* lua_pop: ลบ elements จาก top */
    lua_pop(L, 2);
    printStack(L, "after pop(2)");
    
    /* lua_settop: set stack size (truncate or pad with nil) */
    lua_settop(L, 2);
    printStack(L, "after settop(2)");
    
    /* Clear stack */
    lua_settop(L, 0);
    printStack(L, "cleared");
    
    lua_close(L);
    return 0;
}
```

### ตัวอย่างที่ 6: Stack Manipulation Functions

```c
/* stack_manipulation.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

void showStack(lua_State *L) {
    int n = lua_gettop(L);
    printf("[");
    for (int i = 1; i <= n; i++) {
        if (i > 1) printf(", ");
        int t = lua_type(L, i);
        if (t == LUA_TNUMBER) printf("%g", lua_tonumber(L, i));
        else if (t == LUA_TSTRING) printf("%q", lua_tostring(L, i));
        else if (t == LUA_TBOOLEAN) printf("%s", lua_toboolean(L, i) ? "true" : "false");
        else if (t == LUA_TNIL) printf("nil");
        else printf("<%s>", lua_typename(L, t));
    }
    printf("] (size=%d)\n", n);
}

int main(void) {
    lua_State *L = luaL_newstate();
    
    /* Build initial stack */
    lua_pushinteger(L, 1);
    lua_pushinteger(L, 2);
    lua_pushinteger(L, 3);
    printf("Initial: "); showStack(L);
    
    /* lua_insert: insert element (popped from top) at position */
    lua_pushinteger(L, 99);
    lua_insert(L, 2);  /* insert 99 at position 2 */
    printf("After insert(2): "); showStack(L);
    
    /* lua_remove: remove element at position */
    lua_remove(L, 1);
    printf("After remove(1): "); showStack(L);
    
    /* lua_replace: replace element at position with top */
    lua_pushinteger(L, 77);
    lua_replace(L, 2);
    printf("After replace(2): "); showStack(L);
    
    /* lua_copy: copy element from source to dest */
    lua_copy(L, 1, 3);  /* copy index 1 to index 3 */
    printf("After copy(1,3): "); showStack(L);
    
    /* lua_rotate: rotate stack elements */
    /* lua_rotate(L, idx, n): rotate elements from idx to top by n */
    lua_rotate(L, 1, 1);  /* rotate all elements up by 1 */
    printf("After rotate(1,1): "); showStack(L);
    
    lua_close(L);
    return 0;
}
```

---

## 43.5 Pushing Values onto the Stack

### ตัวอย่างที่ 7: All Push Functions

```c
/* push_values.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

int main(void) {
    lua_State *L = luaL_newstate();
    
    /* === lua_pushnil === */
    lua_pushnil(L);
    printf("nil pushed, type: %s\n", lua_typename(L, lua_type(L, -1)));
    lua_pop(L, 1);
    
    /* === lua_pushboolean === */
    lua_pushboolean(L, 1);  /* true */
    lua_pushboolean(L, 0);  /* false */
    printf("booleans: %d %d\n", lua_toboolean(L, -2), lua_toboolean(L, -1));
    lua_pop(L, 2);
    
    /* === lua_pushnumber === */
    lua_pushnumber(L, 3.14159);
    lua_pushnumber(L, -2.71828);
    printf("numbers: %g %g\n", lua_tonumber(L, -2), lua_tonumber(L, -1));
    lua_pop(L, 2);
    
    /* === lua_pushinteger === */
    lua_pushinteger(L, 42);
    lua_pushinteger(L, -100);
    lua_pushinteger(L, 9007199254740992LL);  /* 2^53 */
    printf("integers: %lld %lld %lld\n",
        (long long)lua_tointeger(L, -3),
        (long long)lua_tointeger(L, -2),
        (long long)lua_tointeger(L, -1));
    lua_pop(L, 3);
    
    /* === lua_pushstring === */
    lua_pushstring(L, "Hello, World!");
    lua_pushstring(L, "สวัสดี Lua");
    printf("strings: %s | %s\n", lua_tostring(L, -2), lua_tostring(L, -1));
    lua_pop(L, 2);
    
    /* === lua_pushlstring === (string with length) */
    const char *raw = "Hello\0World";  /* contains null byte */
    lua_pushlstring(L, raw, 11);  /* 11 bytes including null */
    size_t len;
    lua_tolstring(L, -1, &len);
    printf("lstring length: %zu\n", len);
    lua_pop(L, 1);
    
    /* === lua_pushfstring === (formatted string) */
    lua_pushfstring(L, "Value: %d, Pi: %.4f", 42, 3.14159);
    printf("fstring: %s\n", lua_tostring(L, -1));
    lua_pop(L, 1);
    
    /* === lua_pushlightuserdata === */
    int myVar = 42;
    lua_pushlightuserdata(L, &myVar);
    void *ptr = lua_touserdata(L, -1);
    printf("lightuserdata: ptr=%p, value=%d\n", ptr, *(int*)ptr);
    lua_pop(L, 1);
    
    /* === lua_pushvalue === (duplicate value) */
    lua_pushstring(L, "original");
    lua_pushvalue(L, -1);  /* duplicate top */
    printf("duplicated: %s %s\n", lua_tostring(L, -2), lua_tostring(L, -1));
    lua_pop(L, 2);
    
    /* === lua_pushcfunction === */
    lua_pushcfunction(L, luaopen_base);
    printf("cfunction pushed: %s\n", lua_typename(L, lua_type(L, -1)));
    lua_pop(L, 1);
    
    lua_close(L);
    return 0;
}
```

---

## 43.6 Reading Values from the Stack

### ตัวอย่างที่ 8: All Tovalue Functions

```c
/* read_values.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

int main(void) {
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    
    /* Push test values */
    lua_pushnil(L);         /* index 1 */
    lua_pushboolean(L, 1);  /* index 2 */
    lua_pushinteger(L, 42); /* index 3 */
    lua_pushnumber(L, 3.14);/* index 4 */
    lua_pushstring(L, "hello"); /* index 5 */
    
    /* === lua_type และ lua_typename === */
    printf("Types:\n");
    for (int i = 1; i <= 5; i++) {
        int t = lua_type(L, i);
        printf("  [%d] = %s\n", i, lua_typename(L, t));
    }
    
    /* === lua_isXXX check functions === */
    printf("\nType checks for index 3 (integer 42):\n");
    printf("  isnil:      %d\n", lua_isnil(L, 3));
    printf("  isboolean:  %d\n", lua_isboolean(L, 3));
    printf("  isnumber:   %d\n", lua_isnumber(L, 3));
    printf("  isinteger:  %d\n", lua_isinteger(L, 3));
    printf("  isstring:   %d\n", lua_isstring(L, 3));
    printf("  istable:    %d\n", lua_istable(L, 3));
    
    /* === lua_toboolean === */
    /* In Lua: only nil and false are falsy */
    printf("\nBoolean conversions:\n");
    printf("  nil -> %d\n", lua_toboolean(L, 1));        /* false (nil) */
    printf("  true -> %d\n", lua_toboolean(L, 2));       /* true */
    printf("  42 -> %d\n", lua_toboolean(L, 3));         /* true (any number) */
    printf("  \"hello\" -> %d\n", lua_toboolean(L, 5));  /* true (any string) */
    
    /* === lua_tonumber และ lua_tointeger === */
    printf("\nNumber conversions:\n");
    lua_pop(L, 5);
    lua_pushstring(L, "3.14");   /* string ที่เป็นตัวเลข */
    lua_pushinteger(L, 100);
    lua_pushnumber(L, 1.5);
    
    printf("  \"3.14\" as number: %g\n", lua_tonumber(L, 1));
    printf("  100 as number: %g\n", lua_tonumber(L, 2));
    printf("  1.5 as integer: %lld\n", (long long)lua_tointeger(L, 3));
    
    /* tointegerx: returns success flag */
    int isInt;
    lua_Integer intVal = lua_tointegerx(L, 1, &isInt);
    printf("  \"3.14\" tointegerx: value=%lld, success=%d\n",
        (long long)intVal, isInt);
    
    /* === lua_tostring === */
    printf("\nString conversions:\n");
    lua_pop(L, 3);
    lua_pushinteger(L, 42);
    lua_pushnumber(L, 3.14);
    lua_pushboolean(L, 1);
    lua_pushnil(L);
    
    /* lua_tostring() converts numbers to strings */
    printf("  42 as string: %s\n", lua_tostring(L, 1));   /* "42" */
    printf("  3.14 as string: %s\n", lua_tostring(L, 2)); /* "3.14" */
    /* lua_tostring ไม่แปลง boolean หรือ nil */
    printf("  true as string: %s\n",
        lua_tostring(L, 3) ? lua_tostring(L, 3) : "(NULL)");
    printf("  nil as string: %s\n",
        lua_tostring(L, 4) ? lua_tostring(L, 4) : "(NULL)");
    
    lua_close(L);
    return 0;
}
```

---

## 43.7 C Functions Callable from Lua

### ตัวอย่างที่ 9: Basic C Function

```c
/* math_module.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <math.h>
#include <stdio.h>
#include <string.h>

/*
 * lua_CFunction signature: int f(lua_State *L)
 * - Access arguments via stack indices 1, 2, 3, ...
 * - Push return values onto stack
 * - Return the count of return values
 */

/* ฟังก์ชัน clamp: clamp(value, min, max) */
static int lua_clamp(lua_State *L) {
    /* luaL_checknumber: ต้องเป็น number (หรือ error) */
    double value = luaL_checknumber(L, 1);
    double minVal = luaL_checknumber(L, 2);
    double maxVal = luaL_checknumber(L, 3);
    
    if (value < minVal) value = minVal;
    if (value > maxVal) value = maxVal;
    
    lua_pushnumber(L, value);
    return 1;
}

/* ฟังก์ชัน lerp: lerp(a, b, t) */
static int lua_lerp(lua_State *L) {
    double a = luaL_checknumber(L, 1);
    double b = luaL_checknumber(L, 2);
    double t = luaL_checknumber(L, 3);
    
    lua_pushnumber(L, a + (b - a) * t);
    return 1;
}

/* ฟังก์ชัน distance: distance(x1, y1, x2, y2) */
static int lua_distance(lua_State *L) {
    double x1 = luaL_checknumber(L, 1);
    double y1 = luaL_checknumber(L, 2);
    double x2 = luaL_checknumber(L, 3);
    double y2 = luaL_checknumber(L, 4);
    
    double dx = x2 - x1;
    double dy = y2 - y1;
    lua_pushnumber(L, sqrt(dx*dx + dy*dy));
    return 1;
}

/* ฟังก์ชัน multiple return values */
static int lua_divmod(lua_State *L) {
    lua_Integer a = luaL_checkinteger(L, 1);
    lua_Integer b = luaL_checkinteger(L, 2);
    
    if (b == 0) {
        /* luaL_error: throws Lua error */
        return luaL_error(L, "division by zero");
    }
    
    lua_pushinteger(L, a / b);  /* quotient */
    lua_pushinteger(L, a % b);  /* remainder */
    return 2;  /* 2 return values */
}

/* ฟังก์ชันที่รับ optional argument */
static int lua_round(lua_State *L) {
    double n = luaL_checknumber(L, 1);
    /* luaL_optinteger: optional, default = 0 */
    int places = (int)luaL_optinteger(L, 2, 0);
    
    double factor = pow(10.0, places);
    lua_pushnumber(L, round(n * factor) / factor);
    return 1;
}

/* ฟังก์ชันที่รับ string */
static int lua_strlen_utf8(lua_State *L) {
    size_t len;
    const char *s = luaL_checklstring(L, 1, &len);
    
    /* นับจำนวน UTF-8 characters (ไม่นับ continuation bytes) */
    int count = 0;
    for (size_t i = 0; i < len; i++) {
        unsigned char c = (unsigned char)s[i];
        if ((c & 0xC0) != 0x80) count++;  /* not a continuation byte */
    }
    
    lua_pushinteger(L, count);
    lua_pushinteger(L, (lua_Integer)len);  /* byte length */
    return 2;
}

/* Registration */
static const luaL_Reg math_ext[] = {
    {"clamp",       lua_clamp},
    {"lerp",        lua_lerp},
    {"distance",    lua_distance},
    {"divmod",      lua_divmod},
    {"round",       lua_round},
    {"strlen_utf8", lua_strlen_utf8},
    {NULL, NULL}
};

int luaopen_mathext(lua_State *L) {
    luaL_newlib(L, math_ext);
    
    /* เพิ่ม constants */
    lua_pushnumber(L, M_PI);
    lua_setfield(L, -2, "PI");
    
    lua_pushnumber(L, M_E);
    lua_setfield(L, -2, "E");
    
    return 1;
}
```

```lua
-- test_mathext.lua
-- local m = require("mathext")
--
-- print(m.clamp(150, 0, 100))     -- 100
-- print(m.clamp(-10, 0, 100))     -- 0
-- print(m.clamp(50, 0, 100))      -- 50
--
-- print(m.lerp(0, 100, 0.5))      -- 50.0
-- print(m.lerp(0, 100, 0.25))     -- 25.0
--
-- print(m.distance(0, 0, 3, 4))   -- 5.0
--
-- local q, r = m.divmod(17, 5)
-- print(q, r)                      -- 3  2
--
-- print(m.round(3.14159, 2))       -- 3.14
-- print(m.round(3.14159, 4))       -- 3.1416
--
-- local chars, bytes = m.strlen_utf8("สวัสดี")
-- print(chars, bytes)              -- 6  18 (UTF-8)
--
-- print(m.PI)                      -- 3.14159...
```

---

## 43.8 Error Handling ใน C

### ตัวอย่างที่ 10: Error Handling Patterns

```c
/* error_handling.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <string.h>

/* 1. luaL_error: ส่ง error message ธรรมดา */
static int lua_safeDivide(lua_State *L) {
    double a = luaL_checknumber(L, 1);
    double b = luaL_checknumber(L, 2);
    
    if (b == 0.0) {
        /* luaL_error ไม่ return - มัน longjmp */
        return luaL_error(L, "division by zero: %g / %g", a, b);
    }
    
    lua_pushnumber(L, a / b);
    return 1;
}

/* 2. Argument type checking */
static int lua_stringify(lua_State *L) {
    /* ตรวจสอบ argument type */
    int t = lua_type(L, 1);
    
    switch (t) {
        case LUA_TNIL:
            lua_pushstring(L, "nil");
            break;
        case LUA_TBOOLEAN:
            lua_pushstring(L, lua_toboolean(L, 1) ? "true" : "false");
            break;
        case LUA_TNUMBER:
            lua_pushfstring(L, "%g", lua_tonumber(L, 1));
            break;
        case LUA_TSTRING:
            lua_pushvalue(L, 1);  /* push copy */
            break;
        default:
            /* luaL_argerror: error ที่ระบุ argument number */
            return luaL_argerror(L, 1,
                lua_pushfstring(L, "nil/bool/number/string expected, got %s",
                    lua_typename(L, t)));
    }
    return 1;
}

/* 3. Protected calls ใน C */
static int lua_safeEval(lua_State *L) {
    const char *code = luaL_checkstring(L, 1);
    
    /* luaL_loadstring: compile แต่ไม่รัน */
    if (luaL_loadstring(L, code) != LUA_OK) {
        /* syntax error */
        lua_pushnil(L);
        lua_pushvalue(L, -2);  /* error message already on stack */
        return 2;
    }
    
    /* lua_pcall: เรียก function แบบ protected */
    /* lua_pcall(L, nargs, nresults, msgh) */
    int status = lua_pcall(L, 0, LUA_MULTRET, 0);
    if (status != LUA_OK) {
        lua_pushnil(L);
        lua_insert(L, -2);  /* nil before error message */
        return 2;
    }
    
    /* Success: return true + all results */
    int nresults = lua_gettop(L) - 1;  /* -1 for the code string */
    lua_pushboolean(L, 1);
    lua_insert(L, 2);  /* insert true before results */
    return nresults + 1;
}

/* 4. Custom error object */
static int lua_riskyOp(lua_State *L) {
    int failMode = (int)luaL_optinteger(L, 1, 0);
    
    if (failMode == 1) {
        /* สร้าง error table */
        lua_newtable(L);
        lua_pushstring(L, "NETWORK_ERROR");
        lua_setfield(L, -2, "code");
        lua_pushstring(L, "Connection refused");
        lua_setfield(L, -2, "message");
        lua_pushinteger(L, 503);
        lua_setfield(L, -2, "status");
        
        return lua_error(L);  /* throw table as error */
    } else if (failMode == 2) {
        return luaL_error(L, "Simple string error");
    }
    
    lua_pushstring(L, "Operation succeeded");
    return 1;
}

static const luaL_Reg errlib[] = {
    {"safeDivide", lua_safeDivide},
    {"stringify",  lua_stringify},
    {"safeEval",   lua_safeEval},
    {"riskyOp",    lua_riskyOp},
    {NULL, NULL}
};

int luaopen_errlib(lua_State *L) {
    luaL_newlib(L, errlib);
    return 1;
}
```

```lua
-- test_errors.lua (Lua side)
-- local e = require("errlib")
--
-- -- safeDivide
-- print(e.safeDivide(10, 2))  -- 5.0
-- local ok, err = pcall(e.safeDivide, 10, 0)
-- print(ok, err)  -- false "division by zero: 10 / 0"
--
-- -- stringify
-- print(e.stringify(42))       -- "42"
-- print(e.stringify("hello"))  -- "hello"
-- print(e.stringify(true))     -- "true"
-- local ok2, err2 = pcall(e.stringify, {})
-- print(ok2, err2)  -- false "bad argument #1..."
--
-- -- safeEval
-- local ok3, result = e.safeEval("return 1 + 2")
-- print(ok3, result)  -- true  3
-- local ok4, err4 = e.safeEval("invalid syntax {{{{")
-- print(ok4, err4)    -- nil  "[string...]:1: ..."
```

---

## 43.9 Working with Tables ใน C

### ตัวอย่างที่ 11: Table Creation and Access

```c
/* table_ops.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

/* สร้าง table และ return กลับ Lua */
static int lua_makePoint(lua_State *L) {
    double x = luaL_optnumber(L, 1, 0.0);
    double y = luaL_optnumber(L, 2, 0.0);
    
    /* สร้าง table ใหม่ */
    lua_newtable(L);
    
    /* ตั้งค่า fields */
    lua_pushnumber(L, x);
    lua_setfield(L, -2, "x");  /* table.x = x */
    
    lua_pushnumber(L, y);
    lua_setfield(L, -2, "y");  /* table.y = y */
    
    /* เพิ่ม method "distance" */
    lua_pushcfunction(L, (lua_CFunction)NULL);  /* placeholder */
    
    return 1;  /* return the table */
}

/* อ่านค่าจาก table */
static int lua_sumTable(lua_State *L) {
    luaL_checktype(L, 1, LUA_TTABLE);
    
    double sum = 0.0;
    int count = 0;
    
    /* วนรอบ table ด้วย lua_next */
    lua_pushnil(L);  /* first key */
    while (lua_next(L, 1) != 0) {
        /* key ที่ index -2, value ที่ index -1 */
        if (lua_isnumber(L, -1)) {
            sum += lua_tonumber(L, -1);
            count++;
        }
        lua_pop(L, 1);  /* remove value, keep key for next */
    }
    
    lua_pushnumber(L, sum);
    lua_pushinteger(L, count);
    return 2;
}

/* อ่าน array table */
static int lua_arrayStats(lua_State *L) {
    luaL_checktype(L, 1, LUA_TTABLE);
    
    int n = (int)luaL_len(L, 1);  /* length operator */
    
    if (n == 0) {
        lua_pushnil(L);
        lua_pushstring(L, "empty array");
        return 2;
    }
    
    double sum = 0, minVal = 0, maxVal = 0;
    int first = 1;
    
    for (int i = 1; i <= n; i++) {
        lua_geti(L, 1, i);  /* table[i] */
        double v = lua_tonumber(L, -1);
        lua_pop(L, 1);
        
        sum += v;
        if (first) {
            minVal = maxVal = v;
            first = 0;
        } else {
            if (v < minVal) minVal = v;
            if (v > maxVal) maxVal = v;
        }
    }
    
    /* return stats as table */
    lua_newtable(L);
    
    lua_pushnumber(L, sum / n);
    lua_setfield(L, -2, "avg");
    
    lua_pushnumber(L, minVal);
    lua_setfield(L, -2, "min");
    
    lua_pushnumber(L, maxVal);
    lua_setfield(L, -2, "max");
    
    lua_pushnumber(L, sum);
    lua_setfield(L, -2, "sum");
    
    lua_pushinteger(L, n);
    lua_setfield(L, -2, "count");
    
    return 1;
}

/* สร้าง nested table */
static int lua_makeMatrix(lua_State *L) {
    int rows = (int)luaL_checkinteger(L, 1);
    int cols = (int)luaL_checkinteger(L, 2);
    double initVal = luaL_optnumber(L, 3, 0.0);
    
    lua_createtable(L, rows, 0);  /* pre-allocate array part */
    
    for (int i = 1; i <= rows; i++) {
        lua_createtable(L, cols, 0);  /* row table */
        for (int j = 1; j <= cols; j++) {
            lua_pushnumber(L, initVal);
            lua_seti(L, -2, j);  /* row[j] = initVal */
        }
        lua_seti(L, -2, i);  /* matrix[i] = row */
    }
    
    return 1;
}

static const luaL_Reg tablelib[] = {
    {"makePoint",  lua_makePoint},
    {"sumTable",   lua_sumTable},
    {"arrayStats", lua_arrayStats},
    {"makeMatrix", lua_makeMatrix},
    {NULL, NULL}
};

int luaopen_tableops(lua_State *L) {
    luaL_newlib(L, tablelib);
    return 1;
}
```

```lua
-- test_tableops.lua
-- local t = require("tableops")
--
-- local pt = t.makePoint(3, 4)
-- print(pt.x, pt.y)           -- 3.0  4.0
--
-- local mixed = {a=1, b=2.5, c="str", d=3}
-- local sum, count = t.sumTable(mixed)
-- print(sum, count)            -- 6.5  3
--
-- local arr = {5, 2, 8, 1, 9, 3}
-- local stats = t.arrayStats(arr)
-- print(stats.avg, stats.min, stats.max)  -- 4.67  1  9
--
-- local m = t.makeMatrix(3, 3, 0)
-- m[1][1] = 1; m[2][2] = 1; m[3][3] = 1  -- identity matrix
-- print(m[1][1], m[1][2], m[2][2])        -- 1  0  1
```

---

## 43.10 Userdata

### ตัวอย่างที่ 12: Full Userdata

```c
/* counter_module.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <string.h>

/* ชื่อ metatable สำหรับ type checking */
#define COUNTER_MT "Counter"

/* Data structure */
typedef struct {
    int value;
    int step;
    int minVal;
    int maxVal;
} Counter;

/* ดึง Counter* จาก stack */
static Counter* checkCounter(lua_State *L, int idx) {
    return (Counter*)luaL_checkudata(L, idx, COUNTER_MT);
}

/* Constructor: Counter.new(initial, step, min, max) */
static int counter_new(lua_State *L) {
    int initial = (int)luaL_optinteger(L, 1, 0);
    int step    = (int)luaL_optinteger(L, 2, 1);
    int minVal  = (int)luaL_optinteger(L, 3, INT_MIN);
    int maxVal  = (int)luaL_optinteger(L, 4, INT_MAX);
    
    /* สร้าง userdata */
    Counter *c = (Counter*)lua_newuserdata(L, sizeof(Counter));
    c->value = initial;
    c->step  = step;
    c->minVal = minVal;
    c->maxVal = maxVal;
    
    /* set metatable */
    luaL_setmetatable(L, COUNTER_MT);
    
    return 1;
}

/* Methods */
static int counter_increment(lua_State *L) {
    Counter *c = checkCounter(L, 1);
    int n = (int)luaL_optinteger(L, 2, 1);
    
    c->value += c->step * n;
    if (c->value > c->maxVal) c->value = c->maxVal;
    
    lua_pushinteger(L, c->value);
    return 1;
}

static int counter_decrement(lua_State *L) {
    Counter *c = checkCounter(L, 1);
    int n = (int)luaL_optinteger(L, 2, 1);
    
    c->value -= c->step * n;
    if (c->value < c->minVal) c->value = c->minVal;
    
    lua_pushinteger(L, c->value);
    return 1;
}

static int counter_reset(lua_State *L) {
    Counter *c = checkCounter(L, 1);
    c->value = (int)luaL_optinteger(L, 2, 0);
    return 0;
}

static int counter_get(lua_State *L) {
    Counter *c = checkCounter(L, 1);
    lua_pushinteger(L, c->value);
    return 1;
}

/* __tostring metamethod */
static int counter_tostring(lua_State *L) {
    Counter *c = checkCounter(L, 1);
    lua_pushfstring(L, "Counter{value=%d, step=%d}", c->value, c->step);
    return 1;
}

/* __index metamethod */
static int counter_index(lua_State *L) {
    Counter *c = checkCounter(L, 1);
    const char *key = luaL_checkstring(L, 2);
    
    if (strcmp(key, "value") == 0) {
        lua_pushinteger(L, c->value);
        return 1;
    } else if (strcmp(key, "step") == 0) {
        lua_pushinteger(L, c->step);
        return 1;
    }
    
    /* Look in methods */
    luaL_getmetatable(L, COUNTER_MT);
    lua_getfield(L, -1, key);
    return 1;
}

/* Methods table */
static const luaL_Reg counter_methods[] = {
    {"increment",  counter_increment},
    {"decrement",  counter_decrement},
    {"reset",      counter_reset},
    {"get",        counter_get},
    {"__tostring", counter_tostring},
    {"__index",    counter_index},
    {NULL, NULL}
};

static const luaL_Reg counter_lib[] = {
    {"new", counter_new},
    {NULL, NULL}
};

int luaopen_counter(lua_State *L) {
    /* สร้าง metatable */
    luaL_newmetatable(L, COUNTER_MT);
    luaL_setfuncs(L, counter_methods, 0);
    lua_pop(L, 1);
    
    /* สร้าง module table */
    luaL_newlib(L, counter_lib);
    return 1;
}
```

```lua
-- test_counter.lua
-- local Counter = require("counter")
--
-- local c = Counter.new(0, 5, 0, 100)  -- start=0, step=5, min=0, max=100
-- print(c)                     -- Counter{value=0, step=5}
-- print(c:increment())         -- 5
-- print(c:increment(3))        -- 20 (5 + 3*5)
-- print(c:decrement())         -- 15
-- print(c:get())               -- 15
-- c:reset(50)
-- print(c:get())               -- 50
-- print(c.value)               -- 50
-- print(c.step)                -- 5
```

---

## 43.11 Upvalues และ Closures ใน C

### ตัวอย่างที่ 13: C Closures

```c
/* closures.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

/* C closure ที่มี upvalue */
static int closureIncrement(lua_State *L) {
    /* อ่าน upvalue ที่ 1 (counter) */
    int counter = (int)lua_tointeger(L, lua_upvalueindex(1));
    int step = (int)lua_tointeger(L, lua_upvalueindex(2));
    
    counter += step;
    
    /* อัพเดท upvalue */
    lua_pushinteger(L, counter);
    lua_replace(L, lua_upvalueindex(1));
    
    lua_pushinteger(L, counter);
    return 1;
}

/* Factory function สร้าง closure */
static int makeCounter(lua_State *L) {
    int start = (int)luaL_optinteger(L, 1, 0);
    int step  = (int)luaL_optinteger(L, 2, 1);
    
    /* Push upvalues */
    lua_pushinteger(L, start);  /* upvalue 1: current value */
    lua_pushinteger(L, step);   /* upvalue 2: step */
    
    /* สร้าง closure ด้วย 2 upvalues */
    lua_pushcclosure(L, closureIncrement, 2);
    
    return 1;
}

/* Memoization ด้วย C closure */
static int memoized(lua_State *L) {
    /* upvalue 1: cache table */
    /* upvalue 2: original function */
    
    /* ดูใน cache ก่อน */
    lua_pushvalue(L, 1);  /* key = argument */
    lua_gettable(L, lua_upvalueindex(1));
    
    if (!lua_isnil(L, -1)) {
        /* Cache hit */
        return 1;
    }
    lua_pop(L, 1);
    
    /* Cache miss: เรียก original function */
    lua_pushvalue(L, lua_upvalueindex(2));  /* function */
    lua_pushvalue(L, 1);  /* argument */
    lua_call(L, 1, 1);
    
    /* บันทึกใน cache */
    lua_pushvalue(L, 1);   /* key */
    lua_pushvalue(L, -2);  /* value */
    lua_settable(L, lua_upvalueindex(1));
    
    return 1;
}

static int lua_memoize(lua_State *L) {
    luaL_checktype(L, 1, LUA_TFUNCTION);
    
    lua_newtable(L);           /* upvalue 1: cache */
    lua_pushvalue(L, 1);       /* upvalue 2: original fn */
    lua_pushcclosure(L, memoized, 2);
    
    return 1;
}

static const luaL_Reg closurelib[] = {
    {"makeCounter", makeCounter},
    {"memoize",     lua_memoize},
    {NULL, NULL}
};

int luaopen_closures(lua_State *L) {
    luaL_newlib(L, closurelib);
    return 1;
}
```

```lua
-- test_closures.lua
-- local cl = require("closures")
--
-- -- Counter closures
-- local c1 = cl.makeCounter(0, 1)
-- local c2 = cl.makeCounter(100, -10)
-- print(c1(), c1(), c1())   -- 1  2  3
-- print(c2(), c2(), c2())   -- 90  80  70
--
-- -- Memoization
-- local callCount = 0
-- local function expensive(n)
--     callCount = callCount + 1
--     return n * n
-- end
--
-- local memo_expensive = cl.memoize(expensive)
-- print(memo_expensive(5))   -- 25
-- print(memo_expensive(5))   -- 25 (cached)
-- print(memo_expensive(10))  -- 100
-- print("Total calls:", callCount)  -- 2 (not 3)
```

---

## 43.12 สร้าง C Module สมบูรณ์

### ตัวอย่างที่ 14: String Utils Module

```c
/* strutils.c - String utility functions */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <string.h>
#include <ctype.h>
#include <stdlib.h>

/* trim whitespace */
static int lua_trim(lua_State *L) {
    size_t len;
    const char *s = luaL_checklstring(L, 1, &len);
    
    /* Find start */
    size_t start = 0;
    while (start < len && isspace((unsigned char)s[start])) start++;
    
    /* Find end */
    size_t end = len;
    while (end > start && isspace((unsigned char)s[end - 1])) end--;
    
    lua_pushlstring(L, s + start, end - start);
    return 1;
}

/* split string */
static int lua_split(lua_State *L) {
    size_t strLen, sepLen;
    const char *str = luaL_checklstring(L, 1, &strLen);
    const char *sep = luaL_checklstring(L, 2, &sepLen);
    int limit = (int)luaL_optinteger(L, 3, -1);
    
    if (sepLen == 0) {
        /* Split into characters */
        lua_newtable(L);
        for (size_t i = 0; i < strLen; i++) {
            lua_pushlstring(L, str + i, 1);
            lua_seti(L, -2, (lua_Integer)i + 1);
        }
        return 1;
    }
    
    lua_newtable(L);
    int count = 0;
    const char *p = str;
    const char *end = str + strLen;
    
    while (p <= end) {
        const char *found = NULL;
        if (p < end && (limit < 0 || count < limit - 1)) {
            found = strstr(p, sep);
        }
        
        if (found == NULL || found > end) {
            /* Last piece */
            lua_pushlstring(L, p, end - p);
            lua_seti(L, -2, (lua_Integer)count + 1);
            count++;
            break;
        }
        
        lua_pushlstring(L, p, found - p);
        lua_seti(L, -2, (lua_Integer)count + 1);
        count++;
        p = found + sepLen;
    }
    
    return 1;
}

/* string padding */
static int lua_pad(lua_State *L) {
    size_t strLen;
    const char *str = luaL_checklstring(L, 1, &strLen);
    int width = (int)luaL_checkinteger(L, 2);
    const char *padChar = luaL_optstring(L, 3, " ");
    const char *align = luaL_optstring(L, 4, "left");
    
    if ((int)strLen >= width) {
        lua_pushvalue(L, 1);
        return 1;
    }
    
    int padLen = width - (int)strLen;
    char *result = (char*)malloc(width + 1);
    if (!result) return luaL_error(L, "out of memory");
    
    if (strcmp(align, "right") == 0) {
        for (int i = 0; i < padLen; i++) result[i] = padChar[0];
        memcpy(result + padLen, str, strLen);
    } else if (strcmp(align, "center") == 0) {
        int leftPad = padLen / 2;
        int rightPad = padLen - leftPad;
        for (int i = 0; i < leftPad; i++) result[i] = padChar[0];
        memcpy(result + leftPad, str, strLen);
        for (int i = 0; i < rightPad; i++) result[leftPad + (int)strLen + i] = padChar[0];
    } else {
        /* left */
        memcpy(result, str, strLen);
        for (int i = 0; i < padLen; i++) result[strLen + i] = padChar[0];
    }
    result[width] = '\0';
    
    lua_pushlstring(L, result, width);
    free(result);
    return 1;
}

/* count occurrences */
static int lua_countOccurrences(lua_State *L) {
    size_t strLen, patLen;
    const char *str = luaL_checklstring(L, 1, &strLen);
    const char *pat = luaL_checklstring(L, 2, &patLen);
    
    if (patLen == 0) {
        lua_pushinteger(L, (lua_Integer)strLen + 1);
        return 1;
    }
    
    int count = 0;
    const char *p = str;
    while ((p = strstr(p, pat)) != NULL) {
        count++;
        p += patLen;
    }
    
    lua_pushinteger(L, count);
    return 1;
}

/* starts_with / ends_with */
static int lua_startsWith(lua_State *L) {
    size_t strLen, prefLen;
    const char *str = luaL_checklstring(L, 1, &strLen);
    const char *pref = luaL_checklstring(L, 2, &prefLen);
    
    if (prefLen > strLen) {
        lua_pushboolean(L, 0);
    } else {
        lua_pushboolean(L, memcmp(str, pref, prefLen) == 0);
    }
    return 1;
}

static int lua_endsWith(lua_State *L) {
    size_t strLen, sufLen;
    const char *str = luaL_checklstring(L, 1, &strLen);
    const char *suf = luaL_checklstring(L, 2, &sufLen);
    
    if (sufLen > strLen) {
        lua_pushboolean(L, 0);
    } else {
        lua_pushboolean(L, memcmp(str + strLen - sufLen, suf, sufLen) == 0);
    }
    return 1;
}

static const luaL_Reg strutils_lib[] = {
    {"trim",             lua_trim},
    {"split",            lua_split},
    {"pad",              lua_pad},
    {"countOccurrences", lua_countOccurrences},
    {"startsWith",       lua_startsWith},
    {"endsWith",         lua_endsWith},
    {NULL, NULL}
};

int luaopen_strutils(lua_State *L) {
    luaL_newlib(L, strutils_lib);
    return 1;
}
```

```lua
-- test_strutils.lua
-- local su = require("strutils")
--
-- print(su.trim("  hello  "))        -- "hello"
-- print(su.trim("\t\n test \n\t"))   -- "test"
--
-- local parts = su.split("a,b,c,d", ",")
-- for i, v in ipairs(parts) do print(i, v) end
-- -- 1  a / 2  b / 3  c / 4  d
--
-- local parts2 = su.split("a,b,c", ",", 2)
-- print(#parts2, parts2[2])   -- 2  "b,c"
--
-- print(su.pad("hello", 10))          -- "hello     "
-- print(su.pad("hello", 10, ".", "right"))  -- ".....hello"
-- print(su.pad("hi", 10, "-", "center"))    -- "----hi----"
--
-- print(su.countOccurrences("banana", "an"))  -- 2
--
-- print(su.startsWith("Hello World", "Hello"))  -- true
-- print(su.endsWith("Hello World", "World"))    -- true
```

---

## 43.13 Registering Functions

### ตัวอย่างที่ 15: Different Registration Methods

```c
/* registration_demo.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

static int f1(lua_State *L) { lua_pushstring(L, "method 1"); return 1; }
static int f2(lua_State *L) { lua_pushstring(L, "method 2"); return 1; }
static int f3(lua_State *L) { lua_pushstring(L, "method 3"); return 1; }

/* Method 1: luaL_newlib (recommended for Lua 5.2+) */
static const luaL_Reg lib1[] = {
    {"func1", f1},
    {"func2", f2},
    {NULL, NULL}
};

/* Method 2: Manual registration */
int luaopen_regdemo(lua_State *L) {
    /* === Method 1: luaL_newlib === */
    luaL_newlib(L, lib1);
    /* stack: [module_table] */
    
    /* === Method 2: เพิ่ม function แยก === */
    lua_pushcfunction(L, f3);
    lua_setfield(L, -2, "func3");
    
    /* === Method 3: เพิ่ม string constant === */
    lua_pushstring(L, "v1.0.0");
    lua_setfield(L, -2, "VERSION");
    
    /* === Method 4: เพิ่ม nested table === */
    lua_newtable(L);
    
    lua_pushcfunction(L, f1);
    lua_setfield(L, -2, "sub_f1");
    
    lua_pushcfunction(L, f2);
    lua_setfield(L, -2, "sub_f2");
    
    lua_setfield(L, -2, "sub");  /* module.sub = sub_table */
    
    /* === Method 5: เพิ่ม to global environment === */
    /* (usually not recommended) */
    lua_pushcfunction(L, f1);
    lua_setglobal(L, "globalFunc1");
    
    return 1;
}
```

---

## 43.14 Compiling และ Loading

### ตัวอย่างที่ 16: Makefile

```makefile
# Makefile สำหรับ compile Lua C modules

# Lua config (ใช้ pkg-config)
LUA_CFLAGS = $(shell pkg-config --cflags lua5.4 2>/dev/null || echo "-I/usr/include/lua5.4")
LUA_LIBS   = $(shell pkg-config --libs lua5.4 2>/dev/null || echo "-llua5.4")

CC     = gcc
CFLAGS = -Wall -Wextra -O2 -fPIC $(LUA_CFLAGS)
LDFLAGS = -shared

# Targets
all: hello.so mathext.so strutils.so counter.so

hello.so: hello.c
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $< $(LUA_LIBS) -lm

mathext.so: mathext.c
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $< $(LUA_LIBS) -lm

strutils.so: strutils.c
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $< $(LUA_LIBS)

counter.so: counter.c
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $< $(LUA_LIBS)

clean:
	rm -f *.so

.PHONY: all clean
```

### ตัวอย่างที่ 17: Loading Module ใน Lua

```lua
-- loading_module.lua
-- วิธีต่างๆ ในการ load C module

-- 1. require() (standard way)
-- local mymod = require("mymodule")

-- 2. package.loadlib (manual)
-- local loader = package.loadlib("./mymodule.so", "luaopen_mymodule")
-- local mymod = loader()

-- 3. เพิ่ม search path
package.cpath = package.cpath .. ";./?.so;./lib/?.so"
-- ตอนนี้ require("mymodule") จะค้นหาใน ./mymodule.so ด้วย

-- 4. Load จาก specific path
local function loadFromPath(path, funcName)
    local loader, err = package.loadlib(path, funcName)
    if not loader then
        return nil, err
    end
    local ok, result = pcall(loader)
    if not ok then
        return nil, result
    end
    return result
end

-- ตัวอย่าง: ตรวจสอบว่า module มีอยู่ไหม
local function requireSafe(name)
    local ok, mod = pcall(require, name)
    if not ok then
        return nil, mod  -- mod is error message
    end
    return mod
end

local ffi, err = requireSafe("ffi")
if ffi then
    print("ffi module available (LuaJIT)")
else
    print("ffi not available:", err and err:sub(1, 50))
end

-- ตรวจสอบ package.cpath
print("\nC module search paths:")
for path in package.cpath:gmatch("[^;]+") do
    print("  " .. path)
end

-- ตรวจสอบ package.path
print("\nLua module search paths:")
for path in package.path:gmatch("[^;]+") do
    print("  " .. path)
end
```

---

## 43.15 ตัวอย่างเพิ่มเติม

### ตัวอย่างที่ 18: Bit Operations Module

```c
/* bitops.c - Bitwise operations สำหรับ Lua 5.1/5.2 */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

typedef unsigned int uint32;

static int bit_band(lua_State *L) {
    uint32 a = (uint32)luaL_checknumber(L, 1);
    uint32 b = (uint32)luaL_checknumber(L, 2);
    lua_pushnumber(L, (lua_Number)(a & b));
    return 1;
}

static int bit_bor(lua_State *L) {
    uint32 a = (uint32)luaL_checknumber(L, 1);
    uint32 b = (uint32)luaL_checknumber(L, 2);
    lua_pushnumber(L, (lua_Number)(a | b));
    return 1;
}

static int bit_bxor(lua_State *L) {
    uint32 a = (uint32)luaL_checknumber(L, 1);
    uint32 b = (uint32)luaL_checknumber(L, 2);
    lua_pushnumber(L, (lua_Number)(a ^ b));
    return 1;
}

static int bit_bnot(lua_State *L) {
    uint32 a = (uint32)luaL_checknumber(L, 1);
    lua_pushnumber(L, (lua_Number)(~a));
    return 1;
}

static int bit_lshift(lua_State *L) {
    uint32 a = (uint32)luaL_checknumber(L, 1);
    int n    = (int)luaL_checknumber(L, 2);
    lua_pushnumber(L, (lua_Number)(a << n));
    return 1;
}

static int bit_rshift(lua_State *L) {
    uint32 a = (uint32)luaL_checknumber(L, 1);
    int n    = (int)luaL_checknumber(L, 2);
    lua_pushnumber(L, (lua_Number)(a >> n));
    return 1;
}

static int bit_tobin(lua_State *L) {
    uint32 n = (uint32)luaL_checknumber(L, 1);
    int width = (int)luaL_optinteger(L, 2, 32);
    
    char buf[33];
    for (int i = width - 1; i >= 0; i--) {
        buf[width - 1 - i] = (n >> i) & 1 ? '1' : '0';
    }
    buf[width] = '\0';
    
    lua_pushstring(L, buf);
    return 1;
}

static int bit_popcount(lua_State *L) {
    uint32 n = (uint32)luaL_checknumber(L, 1);
    int count = 0;
    while (n) {
        count += n & 1;
        n >>= 1;
    }
    lua_pushinteger(L, count);
    return 1;
}

static const luaL_Reg bitlib[] = {
    {"band",     bit_band},
    {"bor",      bit_bor},
    {"bxor",     bit_bxor},
    {"bnot",     bit_bnot},
    {"lshift",   bit_lshift},
    {"rshift",   bit_rshift},
    {"tobin",    bit_tobin},
    {"popcount", bit_popcount},
    {NULL, NULL}
};

int luaopen_bitops(lua_State *L) {
    luaL_newlib(L, bitlib);
    return 1;
}
```

### ตัวอย่างที่ 19: C Module ทดสอบใน Pure Lua (Simulation)

```lua
-- simulate_c_module.lua
-- จำลองการทำงาน C module ด้วย Lua เพื่อเข้าใจ API

-- จำลอง "lua_State" ด้วย table
local function createFakeState()
    local state = {
        stack = {},
        globals = {}
    }
    
    function state:push(v)
        table.insert(self.stack, v)
    end
    
    function state:pop(n)
        n = n or 1
        for _ = 1, n do
            table.remove(self.stack)
        end
    end
    
    function state:get(idx)
        if idx > 0 then
            return self.stack[idx]
        else
            return self.stack[#self.stack + idx + 1]
        end
    end
    
    function state:top()
        return #self.stack
    end
    
    function state:setglobal(name, val)
        self.globals[name] = val
    end
    
    function state:getglobal(name)
        self.push(self, self.globals[name])
    end
    
    return state
end

-- จำลองการทำงาน C function
local function simulate_c_function(L, func_name, ...)
    local args = {...}
    for _, v in ipairs(args) do
        L:push(v)
    end
    
    print(string.format("=== Calling %s(%s) ===",
        func_name, table.concat(
            (function()
                local strs = {}
                for _, v in ipairs(args) do
                    table.insert(strs, tostring(v))
                end
                return strs
            end)(), ", ")))
    print("Stack before:", #L.stack, "items")
end

-- Simulate pushes
local L = createFakeState()
L:push(nil)
L:push(true)
L:push(42)
L:push("hello")

print("Stack simulation:")
for i, v in ipairs(L.stack) do
    print(string.format("  [%d] %s: %s", i, type(v), tostring(v)))
end

print("\nNegative indexing:")
print("  [-1] =", tostring(L:get(-1)))  -- "hello"
print("  [-2] =", tostring(L:get(-2)))  -- 42
print("  [-3] =", tostring(L:get(-3)))  -- true

L:pop(2)
print("\nAfter pop(2):")
for i, v in ipairs(L.stack) do
    print(string.format("  [%d] %s", i, tostring(v)))
end
```

### ตัวอย่างที่ 20: Checking C Module Availability

```lua
-- check_modules.lua
-- ตรวจสอบ C modules ที่ available

local function checkModule(name)
    local ok, mod = pcall(require, name)
    if ok then
        return true, mod
    else
        return false, mod
    end
end

local modules_to_check = {
    "bit32",     -- Lua 5.2 bit library (built-in)
    "io",        -- I/O library (built-in)
    "math",      -- Math library (built-in)
    "os",        -- OS library (built-in)
    "string",    -- String library (built-in)
    "table",     -- Table library (built-in)
    "utf8",      -- UTF-8 library (Lua 5.3+)
    "coroutine", -- Coroutine library (built-in)
    "package",   -- Package library (built-in)
    "debug",     -- Debug library (built-in)
    "ffi",       -- LuaJIT FFI
    "jit",       -- LuaJIT JIT control
    "socket",    -- LuaSocket
    "lfs",       -- LuaFileSystem
    "ssl",       -- LuaSec
    "cjson",     -- lua-cjson
    "serpent",   -- Serpent serializer
}

print("Module availability check:")
print(string.format("%-15s %s", "Module", "Status"))
print(string.rep("-", 40))

for _, name in ipairs(modules_to_check) do
    local ok, mod = checkModule(name)
    if ok then
        local version = ""
        if type(mod) == "table" then
            version = mod._VERSION or mod.version or mod.VERSION or ""
            if version ~= "" then version = " (v" .. tostring(version) .. ")" end
        end
        print(string.format("%-15s [OK]%s", name, version))
    else
        print(string.format("%-15s [NOT FOUND]", name))
    end
end

-- Lua version info
print("\nLua version: " .. (_VERSION or "unknown"))
if jit then
    print("LuaJIT version: " .. jit.version)
end
```

### ตัวอย่างที่ 21: Writing Tests สำหรับ C Module

```lua
-- test_framework_for_c_modules.lua
-- Framework ง่ายๆ สำหรับ test C modules

local TestRunner = {}
TestRunner.__index = TestRunner

function TestRunner.new(moduleName)
    local self = setmetatable({}, TestRunner)
    self.moduleName = moduleName
    self.tests = {}
    self.passed = 0
    self.failed = 0
    return self
end

function TestRunner:test(name, fn)
    table.insert(self.tests, {name = name, fn = fn})
end

function TestRunner:assertEqual(actual, expected, msg)
    if actual ~= expected then
        error(string.format("assertEqual failed: expected %s, got %s%s",
            tostring(expected), tostring(actual),
            msg and (" (" .. msg .. ")") or ""), 2)
    end
end

function TestRunner:assertAlmostEqual(actual, expected, epsilon, msg)
    epsilon = epsilon or 1e-10
    if math.abs(actual - expected) > epsilon then
        error(string.format("assertAlmostEqual failed: expected ~%g, got %g%s",
            expected, actual,
            msg and (" (" .. msg .. ")") or ""), 2)
    end
end

function TestRunner:assertTrue(val, msg)
    if not val then
        error("assertTrue failed: " .. (msg or "value is not truthy"), 2)
    end
end

function TestRunner:assertError(fn, pattern, msg)
    local ok, err = pcall(fn)
    if ok then
        error("assertError failed: no error was raised" .. 
              (msg and (" (" .. msg .. ")") or ""), 2)
    end
    if pattern and not tostring(err):find(pattern) then
        error(string.format("assertError failed: error '%s' doesn't match pattern '%s'",
            tostring(err), pattern), 2)
    end
end

function TestRunner:run()
    print(string.format("\n=== Testing module: %s ===", self.moduleName))
    
    for _, test in ipairs(self.tests) do
        local ok, err = pcall(test.fn, self)
        if ok then
            print(string.format("  [PASS] %s", test.name))
            self.passed = self.passed + 1
        else
            print(string.format("  [FAIL] %s: %s", test.name, tostring(err)))
            self.failed = self.failed + 1
        end
    end
    
    print(string.format("\nResults: %d passed, %d failed, %d total",
        self.passed, self.failed, self.passed + self.failed))
    
    return self.failed == 0
end

-- ทดสอบ math module (built-in C)
local runner = TestRunner.new("math")

runner:test("math.sqrt", function(t)
    t:assertAlmostEqual(math.sqrt(4), 2.0)
    t:assertAlmostEqual(math.sqrt(9), 3.0)
    t:assertAlmostEqual(math.sqrt(2), 1.41421356, 1e-7)
end)

runner:test("math.floor and ceil", function(t)
    t:assertEqual(math.floor(3.7), 3)
    t:assertEqual(math.ceil(3.2), 4)
    t:assertEqual(math.floor(-3.7), -4)
    t:assertEqual(math.ceil(-3.2), -3)
end)

runner:test("math.max and min", function(t)
    t:assertEqual(math.max(1, 5, 3, 2, 4), 5)
    t:assertEqual(math.min(1, 5, 3, 2, 4), 1)
end)

runner:test("math.random seeded", function(t)
    math.randomseed(42)
    local r1 = math.random(1, 100)
    math.randomseed(42)
    local r2 = math.random(1, 100)
    t:assertEqual(r1, r2, "same seed should give same result")
end)

runner:test("math error handling", function(t)
    -- math.sqrt ของค่าลบ return NaN (ไม่ error)
    local result = math.sqrt(-1)
    t:assertTrue(result ~= result, "sqrt(-1) should be NaN")
end)

runner:run()
```

### ตัวอย่างที่ 22: Calling Lua from C

```c
/* call_lua_from_c.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

/* เรียก Lua function จาก C */
static void callLuaFunction(lua_State *L) {
    /* ตั้งค่า Lua function */
    const char *luaCode = 
        "function greet(name, times)\n"
        "  local result = {}\n"
        "  for i = 1, times do\n"
        "    result[i] = 'Hello ' .. name .. ' #' .. i\n"
        "  end\n"
        "  return table.concat(result, ', ')\n"
        "end\n";
    
    if (luaL_dostring(L, luaCode) != LUA_OK) {
        fprintf(stderr, "Compile error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
        return;
    }
    
    /* ดึง function จาก global */
    lua_getglobal(L, "greet");
    
    /* Push arguments */
    lua_pushstring(L, "World");
    lua_pushinteger(L, 3);
    
    /* เรียก: 2 args, 1 return value */
    if (lua_pcall(L, 2, 1, 0) != LUA_OK) {
        fprintf(stderr, "Runtime error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
        return;
    }
    
    /* ดึง result */
    const char *result = lua_tostring(L, -1);
    printf("Lua result: %s\n", result);
    lua_pop(L, 1);
}

/* เรียก Lua function ที่ return หลายค่า */
static void callLuaMultiReturn(lua_State *L) {
    luaL_dostring(L,
        "function stats(t)\n"
        "  local sum, min, max = 0, t[1], t[1]\n"
        "  for _, v in ipairs(t) do\n"
        "    sum = sum + v\n"
        "    if v < min then min = v end\n"
        "    if v > max then max = v end\n"
        "  end\n"
        "  return sum/#t, min, max\n"
        "end\n");
    
    lua_getglobal(L, "stats");
    
    /* สร้าง array table */
    lua_newtable(L);
    int data[] = {5, 2, 8, 1, 9, 3, 7, 4, 6};
    int n = sizeof(data) / sizeof(data[0]);
    for (int i = 0; i < n; i++) {
        lua_pushinteger(L, data[i]);
        lua_seti(L, -2, i + 1);
    }
    
    /* เรียก: 1 arg, 3 return values */
    if (lua_pcall(L, 1, 3, 0) != LUA_OK) {
        fprintf(stderr, "Error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
        return;
    }
    
    /* ดึง 3 return values (bottom-to-top) */
    double avg = lua_tonumber(L, -3);
    double min = lua_tonumber(L, -2);
    double max = lua_tonumber(L, -1);
    lua_pop(L, 3);
    
    printf("Stats: avg=%.2f, min=%.0f, max=%.0f\n", avg, min, max);
}

int main(void) {
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    
    printf("=== Calling Lua from C ===\n");
    callLuaFunction(L);
    callLuaMultiReturn(L);
    
    lua_close(L);
    return 0;
}
```

### ตัวอย่างที่ 23: Metamethods ใน C

```c
/* vector_module.c - Vector2D ด้วย metamethods */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <math.h>
#include <stdio.h>
#include <string.h>

#define VEC2_MT "Vector2D"

typedef struct { double x, y; } Vec2;

static Vec2* checkVec2(lua_State *L, int idx) {
    return (Vec2*)luaL_checkudata(L, idx, VEC2_MT);
}

static int vec2_new(lua_State *L) {
    double x = luaL_optnumber(L, 1, 0.0);
    double y = luaL_optnumber(L, 2, 0.0);
    Vec2 *v = (Vec2*)lua_newuserdata(L, sizeof(Vec2));
    v->x = x; v->y = y;
    luaL_setmetatable(L, VEC2_MT);
    return 1;
}

/* __add metamethod: v1 + v2 */
static int vec2_add(lua_State *L) {
    Vec2 *a = checkVec2(L, 1);
    Vec2 *b = checkVec2(L, 2);
    Vec2 *r = (Vec2*)lua_newuserdata(L, sizeof(Vec2));
    r->x = a->x + b->x;
    r->y = a->y + b->y;
    luaL_setmetatable(L, VEC2_MT);
    return 1;
}

/* __sub metamethod: v1 - v2 */
static int vec2_sub(lua_State *L) {
    Vec2 *a = checkVec2(L, 1);
    Vec2 *b = checkVec2(L, 2);
    Vec2 *r = (Vec2*)lua_newuserdata(L, sizeof(Vec2));
    r->x = a->x - b->x;
    r->y = a->y - b->y;
    luaL_setmetatable(L, VEC2_MT);
    return 1;
}

/* __mul metamethod: v * scalar or scalar * v */
static int vec2_mul(lua_State *L) {
    Vec2 *r = (Vec2*)lua_newuserdata(L, sizeof(Vec2));
    if (lua_isuserdata(L, 1)) {
        Vec2 *v = checkVec2(L, 1);
        double s = luaL_checknumber(L, 2);
        r->x = v->x * s; r->y = v->y * s;
    } else {
        double s = luaL_checknumber(L, 1);
        Vec2 *v = checkVec2(L, 2);
        r->x = v->x * s; r->y = v->y * s;
    }
    luaL_setmetatable(L, VEC2_MT);
    return 1;
}

/* __eq metamethod */
static int vec2_eq(lua_State *L) {
    Vec2 *a = checkVec2(L, 1);
    Vec2 *b = checkVec2(L, 2);
    lua_pushboolean(L, a->x == b->x && a->y == b->y);
    return 1;
}

/* __len metamethod: #v = magnitude */
static int vec2_len(lua_State *L) {
    Vec2 *v = checkVec2(L, 1);
    lua_pushnumber(L, sqrt(v->x * v->x + v->y * v->y));
    return 1;
}

/* __tostring */
static int vec2_tostring(lua_State *L) {
    Vec2 *v = checkVec2(L, 1);
    lua_pushfstring(L, "Vec2(%g, %g)", v->x, v->y);
    return 1;
}

/* __index: get fields and methods */
static int vec2_index(lua_State *L) {
    Vec2 *v = checkVec2(L, 1);
    const char *key = luaL_checkstring(L, 2);
    
    if (strcmp(key, "x") == 0) { lua_pushnumber(L, v->x); return 1; }
    if (strcmp(key, "y") == 0) { lua_pushnumber(L, v->y); return 1; }
    
    /* Lookup in methods */
    luaL_getmetatable(L, VEC2_MT);
    lua_getfield(L, -1, key);
    return 1;
}

/* Methods */
static int vec2_dot(lua_State *L) {
    Vec2 *a = checkVec2(L, 1);
    Vec2 *b = checkVec2(L, 2);
    lua_pushnumber(L, a->x * b->x + a->y * b->y);
    return 1;
}

static int vec2_normalize(lua_State *L) {
    Vec2 *v = checkVec2(L, 1);
    double mag = sqrt(v->x * v->x + v->y * v->y);
    Vec2 *r = (Vec2*)lua_newuserdata(L, sizeof(Vec2));
    if (mag > 0) { r->x = v->x / mag; r->y = v->y / mag; }
    else { r->x = 0; r->y = 0; }
    luaL_setmetatable(L, VEC2_MT);
    return 1;
}

static const luaL_Reg vec2_meta[] = {
    {"__add",      vec2_add},
    {"__sub",      vec2_sub},
    {"__mul",      vec2_mul},
    {"__eq",       vec2_eq},
    {"__len",      vec2_len},
    {"__tostring", vec2_tostring},
    {"__index",    vec2_index},
    {"dot",        vec2_dot},
    {"normalize",  vec2_normalize},
    {NULL, NULL}
};

static const luaL_Reg vec2_lib[] = {
    {"new", vec2_new},
    {NULL, NULL}
};

int luaopen_vector2d(lua_State *L) {
    luaL_newmetatable(L, VEC2_MT);
    luaL_setfuncs(L, vec2_meta, 0);
    lua_pop(L, 1);
    
    luaL_newlib(L, vec2_lib);
    return 1;
}
```

```lua
-- test_vector2d.lua
-- local Vec2 = require("vector2d")
--
-- local a = Vec2.new(3, 4)
-- local b = Vec2.new(1, 2)
--
-- print(tostring(a))           -- Vec2(3, 4)
-- print(#a)                    -- 5.0 (magnitude)
-- print(tostring(a + b))       -- Vec2(4, 6)
-- print(tostring(a - b))       -- Vec2(2, 2)
-- print(tostring(a * 2))       -- Vec2(6, 8)
-- print(a.x, a.y)              -- 3  4
-- print(a:dot(b))              -- 11 (3*1 + 4*2)
-- print(tostring(a:normalize()))  -- Vec2(0.6, 0.8)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Lua C API:

1. **lua_State** - struct ที่เก็บ Lua interpreter state
2. **The Stack** - mechanism หลักในการแลกเปลี่ยนข้อมูลระหว่าง C และ Lua
3. **Pushing Values** - lua_pushnil, lua_pushnumber, lua_pushstring, etc.
4. **Reading Values** - lua_tonumber, lua_tostring, lua_toboolean
5. **Stack Manipulation** - lua_gettop, lua_settop, lua_pop, lua_insert
6. **C Functions** - lua_CFunction signature และ registration
7. **Error Handling** - luaL_error, luaL_argerror, lua_pcall
8. **Tables** - สร้างและเข้าถึง tables ใน C
9. **Userdata** - full userdata สำหรับ C structures
10. **Closures** - C closures ด้วย upvalues
11. **Metamethods** - สร้าง metamethods ใน C
12. **Module System** - สร้างและ load C modules

การเขียน C extensions ช่วยให้ Lua สามารถทำงาน compute-intensive ได้เร็วขึ้น และเข้าถึง system resources ที่ Lua ไม่มี built-in support
