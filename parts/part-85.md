# บทที่ 85: Advanced C API กับ Lua

## บทนำ

Lua ถูกออกแบบมาให้สามารถฝัง (embed) เข้ากับภาษา C ได้อย่างง่ายดาย **C API** ของ Lua ให้เราสร้าง Extension ที่เชื่อม Lua กับ C Library เพื่อประสิทธิภาพสูงสุด ในบทนี้เราจะศึกษาการทำงานลึกของ C API รวมถึงการสร้าง Extension ที่ซับซ้อนอย่าง Fast CSV Parser

---

## 85.1 lua_State Lifecycle และ Threading Model

### 85.1.1 lua_State คืออะไร

`lua_State` เป็น struct ที่เก็บสถานะทั้งหมดของ Lua VM หนึ่งตัว รวมถึง stack, global table, garbage collector state และ runtime information

```c
/* lua_State lifecycle */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>

int main(void) {
    /* สร้าง Lua state ใหม่ */
    lua_State *L = luaL_newstate();
    if (!L) {
        fprintf(stderr, "Cannot create Lua state: out of memory\n");
        return 1;
    }
    
    /* โหลด standard libraries */
    luaL_openlibs(L);
    
    /* รัน Lua code */
    int status = luaL_dostring(L, "print('Hello from Lua!')");
    if (status != LUA_OK) {
        /* ดึง error message จาก stack */
        fprintf(stderr, "Error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
    }
    
    /* ปิด state และ free memory ทั้งหมด */
    lua_close(L);
    return 0;
}
```

### 85.1.2 Main Thread vs Coroutines

ใน Lua 5.4 แต่ละ Coroutine มี `lua_State` ของตัวเอง แต่ share global environment

```c
/* สร้างและรัน coroutine จาก C */
int run_coroutine_example(lua_State *L) {
    /* สร้าง coroutine thread */
    lua_State *co = lua_newthread(L);
    /* co ถูก push บน stack ของ L ด้วย */
    
    /* โหลด function สำหรับ coroutine */
    luaL_loadstring(co, 
        "local i = 0\n"
        "while true do\n"
        "    i = i + 1\n"
        "    coroutine.yield(i)\n"
        "end\n"
    );
    
    /* Resume coroutine 3 ครั้ง */
    for (int i = 0; i < 3; i++) {
        int nresults;
        int status = lua_resume(co, L, 0, &nresults);
        
        if (status == LUA_YIELD) {
            printf("Yielded: %lld\n", lua_tointeger(co, -1));
            lua_pop(co, nresults);
        } else if (status == LUA_OK) {
            printf("Coroutine finished\n");
            break;
        } else {
            fprintf(stderr, "Coroutine error: %s\n", lua_tostring(co, -1));
            break;
        }
    }
    
    return 0;
}
```

### 85.1.3 Thread-Safe Lua: lua_newstate per Thread

```c
#include <pthread.h>

/* แต่ละ POSIX thread ควรมี lua_State ของตัวเอง */
typedef struct {
    int thread_id;
    const char *script;
    int result;
} ThreadArgs;

static void *lua_thread_worker(void *arg) {
    ThreadArgs *args = (ThreadArgs *)arg;
    
    /* สร้าง independent Lua state */
    lua_State *L = luaL_newstate();
    if (!L) {
        args->result = -1;
        return NULL;
    }
    
    luaL_openlibs(L);
    
    /* Push thread id เป็น global variable */
    lua_pushinteger(L, args->thread_id);
    lua_setglobal(L, "THREAD_ID");
    
    /* รัน script */
    int status = luaL_dostring(L, args->script);
    args->result = (status == LUA_OK) ? 0 : -1;
    
    if (status != LUA_OK) {
        fprintf(stderr, "Thread %d error: %s\n",
            args->thread_id, lua_tostring(L, -1));
    }
    
    lua_close(L);
    return NULL;
}

int run_parallel_lua(const char *script, int num_threads) {
    pthread_t threads[num_threads];
    ThreadArgs args[num_threads];
    
    for (int i = 0; i < num_threads; i++) {
        args[i].thread_id = i;
        args[i].script    = script;
        args[i].result    = 0;
        
        pthread_create(&threads[i], NULL, lua_thread_worker, &args[i]);
    }
    
    int all_ok = 1;
    for (int i = 0; i < num_threads; i++) {
        pthread_join(threads[i], NULL);
        if (args[i].result != 0) all_ok = 0;
    }
    
    return all_ok ? 0 : -1;
}
```

---

## 85.2 Stack Manipulation

### 85.2.1 เข้าใจ Lua Stack

Lua ใช้ Virtual Stack เป็นสื่อกลางในการส่งข้อมูลระหว่าง C และ Lua Stack Index สามารถเป็นบวก (จากล่าง) หรือลบ (จากบน)

```c
/* การ Push ค่าต่างๆ ขึ้น Stack */
void demonstrate_push(lua_State *L) {
    /* lua_push* functions */
    lua_pushnil(L);              /* stack: [nil]             index: 1, -1 */
    lua_pushboolean(L, 1);       /* stack: [nil, true]       index: 2, -1 */
    lua_pushinteger(L, 42);      /* stack: [nil, true, 42]   index: 3, -1 */
    lua_pushnumber(L, 3.14);     /* stack: [..., 3.14]       index: 4, -1 */
    lua_pushstring(L, "hello");  /* stack: [..., "hello"]    index: 5, -1 */
    lua_pushlstring(L, "hi", 2); /* stack: [..., "hi"]       index: 6, -1 */
    
    /* ตรวจสอบ stack size */
    int top = lua_gettop(L);
    printf("Stack has %d elements\n", top);
    
    /* Print แต่ละ element */
    for (int i = 1; i <= top; i++) {
        int type = lua_type(L, i);
        printf("  [%d] type=%s value=",
            i, lua_typename(L, type));
        
        switch (type) {
            case LUA_TNIL:
                printf("nil\n");
                break;
            case LUA_TBOOLEAN:
                printf("%s\n", lua_toboolean(L, i) ? "true" : "false");
                break;
            case LUA_TNUMBER:
                if (lua_isinteger(L, i))
                    printf("%lld\n", lua_tointeger(L, i));
                else
                    printf("%g\n", lua_tonumber(L, i));
                break;
            case LUA_TSTRING:
                printf("'%s'\n", lua_tostring(L, i));
                break;
            default:
                printf("(other)\n");
        }
    }
    
    /* ล้าง stack */
    lua_settop(L, 0);
}
```

### 85.2.2 lua_to* Functions

```c
/* ดึงค่าจาก stack ด้วย lua_to* */
void demonstrate_get(lua_State *L) {
    /* สมมติ stack มี: [42, 3.14, "hello", true, {key="val"}] */
    
    /* lua_toboolean: แปลงเป็น C int (0/1) */
    int b = lua_toboolean(L, 4);  /* true -> 1 */
    
    /* lua_tointeger: แปลงเป็น lua_Integer */
    lua_Integer n = lua_tointeger(L, 1);  /* 42 */
    
    /* lua_tonumber: แปลงเป็น lua_Number (double) */
    lua_Number d = lua_tonumber(L, 2);  /* 3.14 */
    
    /* lua_tostring: แปลงเป็น C string (อย่า free!) */
    const char *s = lua_tostring(L, 3);  /* "hello" */
    
    /* lua_tolstring: string พร้อม length (สำคัญสำหรับ binary data) */
    size_t len;
    const char *raw = lua_tolstring(L, 3, &len);
    
    /* lua_topointer: ดึง pointer (สำหรับ userdata/table) */
    const void *ptr = lua_topointer(L, 5);
    
    printf("integer=%lld number=%.2f string='%s' bool=%d\n",
        n, d, s, b);
}
```

### 85.2.3 lua_check* Functions (Type-safe)

```c
/* luaL_check* - throw error ถ้า type ไม่ถูกต้อง */
static int my_function(lua_State *L) {
    /* ตรวจสอบและดึง arguments */
    
    /* luaL_checkinteger: ต้องเป็น integer */
    lua_Integer id = luaL_checkinteger(L, 1);
    
    /* luaL_checknumber: ต้องเป็น number */
    lua_Number amount = luaL_checknumber(L, 2);
    
    /* luaL_checkstring: ต้องเป็น string */
    size_t name_len;
    const char *name = luaL_checklstring(L, 3, &name_len);
    
    /* luaL_checktype: ต้องเป็น type ที่ระบุ */
    luaL_checktype(L, 4, LUA_TTABLE);
    
    /* luaL_optinteger: optional, ใช้ default ถ้าไม่มี */
    lua_Integer page = luaL_optinteger(L, 5, 1);
    
    /* luaL_optstring: optional string */
    const char *format = luaL_optstring(L, 6, "json");
    
    printf("id=%lld amount=%.2f name=%s page=%lld format=%s\n",
        id, amount, name, page, format);
    
    lua_pushboolean(L, 1);
    return 1;
}
```

---

## 85.3 Creating Lua Userdata กับ __gc Finalizer

### 85.3.1 Full Userdata

Userdata เป็น block of memory ที่ Lua จัดการ Garbage Collection ให้ แต่เนื้อหาข้างในเราจัดการเอง

```c
/* ตัวอย่าง: File handle ที่ wrapped ด้วย userdata */

typedef struct {
    FILE *fp;
    int  closed;
    char filename[256];
} FileHandle;

#define FILE_HANDLE_MT "mylib.FileHandle"

/* สร้าง FileHandle userdata */
static int filehandle_open(lua_State *L) {
    const char *filename = luaL_checkstring(L, 1);
    const char *mode     = luaL_optstring(L, 2, "r");
    
    /* Allocate userdata */
    FileHandle *fh = (FileHandle *)lua_newuserdata(L, sizeof(FileHandle));
    fh->closed = 1;
    fh->fp     = NULL;
    strncpy(fh->filename, filename, sizeof(fh->filename) - 1);
    
    /* ติด metatable */
    luaL_setmetatable(L, FILE_HANDLE_MT);
    
    /* เปิดไฟล์ */
    fh->fp = fopen(filename, mode);
    if (!fh->fp) {
        /* ถ้าเปิดไม่ได้ userdata ยังคงอยู่แต่ __gc จะ handle */
        return luaL_error(L, "cannot open '%s': %s",
            filename, strerror(errno));
    }
    fh->closed = 0;
    
    return 1;  /* return userdata */
}

/* __gc finalizer - เรียกเมื่อ GC เก็บ userdata */
static int filehandle_gc(lua_State *L) {
    FileHandle *fh = (FileHandle *)luaL_checkudata(L, 1, FILE_HANDLE_MT);
    
    if (!fh->closed && fh->fp) {
        fclose(fh->fp);
        fh->fp     = NULL;
        fh->closed = 1;
    }
    
    return 0;
}

/* __tostring สำหรับ debugging */
static int filehandle_tostring(lua_State *L) {
    FileHandle *fh = (FileHandle *)luaL_checkudata(L, 1, FILE_HANDLE_MT);
    
    lua_pushfstring(L, "FileHandle(%s, %s)",
        fh->filename,
        fh->closed ? "closed" : "open");
    
    return 1;
}

/* Read method */
static int filehandle_read(lua_State *L) {
    FileHandle *fh = (FileHandle *)luaL_checkudata(L, 1, FILE_HANDLE_MT);
    
    if (fh->closed || !fh->fp) {
        return luaL_error(L, "attempt to read from closed file");
    }
    
    size_t n = (size_t)luaL_optinteger(L, 2, 4096);
    
    /* Allocate buffer */
    luaL_Buffer buf;
    char *p = luaL_buffinitsize(L, &buf, n);
    
    size_t bytes_read = fread(p, 1, n, fh->fp);
    
    if (bytes_read == 0) {
        lua_pushnil(L);
        return 1;
    }
    
    luaL_pushresultsize(&buf, bytes_read);
    return 1;
}

/* Close method */
static int filehandle_close(lua_State *L) {
    FileHandle *fh = (FileHandle *)luaL_checkudata(L, 1, FILE_HANDLE_MT);
    
    if (!fh->closed && fh->fp) {
        if (fclose(fh->fp) != 0) {
            return luaL_error(L, "error closing file: %s", strerror(errno));
        }
        fh->fp     = NULL;
        fh->closed = 1;
    }
    
    lua_pushboolean(L, 1);
    return 1;
}
```

---

## 85.4 luaL_newmetatable และ Method Registration

### 85.4.1 Metatable Setup

```c
/* ลงทะเบียน metatable และ methods */
static void register_filehandle_metatable(lua_State *L) {
    /* สร้าง/หา metatable */
    if (luaL_newmetatable(L, FILE_HANDLE_MT)) {
        /* ถ้าสร้างใหม่ (ยังไม่มี) ให้ตั้งค่า */
        
        /* __gc: finalizer */
        lua_pushcfunction(L, filehandle_gc);
        lua_setfield(L, -2, "__gc");
        
        /* __close: for-to-be-closed (Lua 5.4+) */
        lua_pushcfunction(L, filehandle_close);
        lua_setfield(L, -2, "__close");
        
        /* __tostring */
        lua_pushcfunction(L, filehandle_tostring);
        lua_setfield(L, -2, "__tostring");
        
        /* __index = method table (OOP style) */
        lua_newtable(L);  /* create method table */
        
        static const luaL_Reg methods[] = {
            { "read",  filehandle_read  },
            { "close", filehandle_close },
            { NULL,    NULL             },
        };
        luaL_setfuncs(L, methods, 0);
        
        lua_setfield(L, -2, "__index");  /* mt.__index = methods */
    }
    lua_pop(L, 1);  /* pop metatable */
}

/* ตัวอย่างการใช้งานจาก Lua:
local f = mylib.open("/tmp/test.txt", "r")
local data = f:read(1024)
f:close()
-- หรือใช้ to-be-closed variable (Lua 5.4):
local f <close> = mylib.open("/tmp/test.txt", "r")
local data = f:read()
-- f:close() ถูกเรียกอัตโนมัติเมื่อออกจาก scope
*/
```

### 85.4.2 Registration Pattern ที่สมบูรณ์

```c
/* Module registration */
static const luaL_Reg mylib_funcs[] = {
    { "open",   filehandle_open   },
    { NULL,     NULL              },
};

int luaopen_mylib(lua_State *L) {
    /* ลงทะเบียน metatables */
    register_filehandle_metatable(L);
    
    /* สร้าง module table */
    luaL_newlib(L, mylib_funcs);
    
    /* เพิ่ม constants */
    lua_pushstring(L, "1.0.0");
    lua_setfield(L, -2, "VERSION");
    
    return 1;  /* return module table */
}
```

---

## 85.5 Error Handling: lua_pcall และ lua_error

### 85.5.1 lua_pcall - Protected Call

```c
/* lua_pcall: เรียก function ใน protected mode */
int call_lua_function(lua_State *L, const char *fname,
                      int nargs, int nresults) {
    /* ดึง function จาก global */
    int type = lua_getglobal(L, fname);
    if (type != LUA_TFUNCTION) {
        lua_pop(L, 1);
        fprintf(stderr, "'%s' is not a function\n", fname);
        return -1;
    }
    
    /* Stack ตอนนี้: [args..., function] */
    /* ต้อง move function ไว้ก่อน args */
    lua_insert(L, -(nargs + 1));
    
    /* เรียก function ด้วย protection */
    int status = lua_pcall(L, nargs, nresults, 0);
    
    if (status != LUA_OK) {
        const char *msg = lua_tostring(L, -1);
        fprintf(stderr, "Error calling '%s': %s\n", fname, msg);
        lua_pop(L, 1);
        return -1;
    }
    
    return 0;
}

/* lua_pcall กับ error handler (msgh) */
static int traceback_handler(lua_State *L) {
    const char *msg = lua_tostring(L, 1);
    if (msg) {
        luaL_traceback(L, L, msg, 1);
    } else {
        lua_pushliteral(L, "(error object is not a string)");
    }
    return 1;
}

int call_with_traceback(lua_State *L, const char *code) {
    /* Push error handler */
    lua_pushcfunction(L, traceback_handler);
    int handler_idx = lua_gettop(L);
    
    /* Load code */
    if (luaL_loadstring(L, code) != LUA_OK) {
        lua_remove(L, handler_idx);
        return -1;
    }
    
    /* Call with error handler */
    int status = lua_pcall(L, 0, LUA_MULTRET, handler_idx);
    
    if (status != LUA_OK) {
        fprintf(stderr, "Error:\n%s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
    }
    
    lua_remove(L, handler_idx);
    return status == LUA_OK ? 0 : -1;
}
```

### 85.5.2 lua_error - Throwing Errors

```c
/* lua_error:던지다 error จาก C function */
static int divide(lua_State *L) {
    lua_Number a = luaL_checknumber(L, 1);
    lua_Number b = luaL_checknumber(L, 2);
    
    if (b == 0.0) {
        /* สร้าง error object (อาจเป็น string หรือ table) */
        lua_newtable(L);
        lua_pushstring(L, "division_by_zero");
        lua_setfield(L, -2, "code");
        lua_pushstring(L, "Cannot divide by zero");
        lua_setfield(L, -2, "message");
        
        /* lua_error ไม่ return (longjmp) */
        lua_error(L);
        /* ไม่มีทางมาถึงบรรทัดนี้ */
    }
    
    lua_pushnumber(L, a / b);
    return 1;
}

/* luaL_error: สะดวกกว่า, รับ format string */
static int validate_age(lua_State *L) {
    lua_Integer age = luaL_checkinteger(L, 1);
    
    if (age < 0 || age > 150) {
        return luaL_error(L, "invalid age: %lld (must be 0-150)", age);
    }
    
    lua_pushboolean(L, 1);
    return 1;
}
```

---

## 85.6 lua_CFunction Implementation Patterns

### 85.6.1 String Manipulation Function

```c
/* C function ที่ process string */
static int string_reverse_words(lua_State *L) {
    size_t len;
    const char *str = luaL_checklstring(L, 1, &len);
    
    /* Copy string to mutable buffer */
    char *buf = (char *)malloc(len + 1);
    if (!buf) return luaL_error(L, "out of memory");
    
    memcpy(buf, str, len + 1);
    
    /* Reverse word order */
    char *words[1024];
    int word_count = 0;
    char *token = strtok(buf, " ");
    
    while (token && word_count < 1024) {
        words[word_count++] = token;
        token = strtok(NULL, " ");
    }
    
    /* Build result */
    luaL_Buffer result;
    luaL_buffinit(L, &result);
    
    for (int i = word_count - 1; i >= 0; i--) {
        luaL_addstring(&result, words[i]);
        if (i > 0) luaL_addchar(&result, ' ');
    }
    
    luaL_pushresult(&result);
    free(buf);
    
    return 1;
}

/* C function ที่ return หลายค่า */
static int string_split_at(lua_State *L) {
    size_t len;
    const char *str = luaL_checklstring(L, 1, &len);
    lua_Integer pos  = luaL_checkinteger(L, 2);
    
    /* Validate position */
    if (pos < 1 || (size_t)pos > len) {
        return luaL_error(L, "position %lld out of range [1, %zu]", pos, len);
    }
    
    /* Return สอง strings */
    lua_pushlstring(L, str, (size_t)(pos - 1));     /* ส่วนแรก */
    lua_pushlstring(L, str + pos - 1, len - (size_t)(pos - 1));  /* ส่วนหลัง */
    
    return 2;  /* return 2 values */
}
```

### 85.6.2 Table Manipulation

```c
/* เข้าถึงและแก้ไข Lua table จาก C */
static int table_sum(lua_State *L) {
    luaL_checktype(L, 1, LUA_TTABLE);
    
    lua_Number sum = 0;
    int count = 0;
    
    /* วนลูป table (ใช้ lua_next สำหรับ generic table) */
    lua_pushnil(L);  /* first key */
    
    while (lua_next(L, 1) != 0) {
        /* key อยู่ที่ -2, value อยู่ที่ -1 */
        if (lua_type(L, -1) == LUA_TNUMBER) {
            sum += lua_tonumber(L, -1);
            count++;
        }
        lua_pop(L, 1);  /* ลบ value, เก็บ key สำหรับ next iteration */
    }
    
    lua_pushnumber(L, sum);
    lua_pushinteger(L, count);
    
    return 2;  /* sum, count */
}

/* สร้าง table ใน C และ return ไปยัง Lua */
static int make_range(lua_State *L) {
    lua_Integer from = luaL_checkinteger(L, 1);
    lua_Integer to   = luaL_checkinteger(L, 2);
    lua_Integer step = luaL_optinteger(L, 3, 1);
    
    if (step == 0)
        return luaL_error(L, "step cannot be zero");
    
    lua_newtable(L);
    int idx = 1;
    
    for (lua_Integer i = from;
         step > 0 ? i <= to : i >= to;
         i += step) {
        lua_pushinteger(L, i);
        lua_rawseti(L, -2, idx++);
    }
    
    return 1;
}
```

---

## 85.7 Light Userdata vs Full Userdata

### 85.7.1 ความแตกต่าง

```c
/*
 * Full userdata:
 *   - Lua manages memory (GC)
 *   - Can have metatable
 *   - Can have __gc finalizer
 *   - ใช้ lua_newuserdata()
 *
 * Light userdata:
 *   - เป็นแค่ pointer (C void*)
 *   - Lua ไม่ manage memory
 *   - ไม่มี metatable
 *   - ไม่มี __gc
 *   - ใช้ lua_pushlightuserdata()
 *   - เปรียบเทียบด้วย pointer equality
 */

/* Light userdata - สำหรับ pass C pointer ไปยัง Lua */
static void demonstrate_light_userdata(lua_State *L) {
    /* มี C object ที่ Lua ไม่ต้อง manage memory */
    static int MY_REGISTRY_KEY;  /* ใช้ address ของ static variable เป็น key */
    
    /* Push pointer เป็น light userdata */
    void *ptr = malloc(100);
    lua_pushlightuserdata(L, ptr);
    
    /* ใช้เป็น registry key */
    lua_pushlightuserdata(L, &MY_REGISTRY_KEY);  /* key */
    lua_pushstring(L, "stored value");            /* value */
    lua_settable(L, LUA_REGISTRYINDEX);
    
    /* ดึงกลับ */
    lua_pushlightuserdata(L, &MY_REGISTRY_KEY);
    lua_gettable(L, LUA_REGISTRYINDEX);
    printf("Retrieved: %s\n", lua_tostring(L, -1));
    lua_pop(L, 1);
    
    /* ต้อง free เอง! */
    free(ptr);
}

/* Full userdata - สำหรับ objects ที่ต้องการ lifecycle management */
typedef struct {
    int    *data;
    size_t  size;
    size_t  capacity;
} IntArray;

#define INT_ARRAY_MT "mylib.IntArray"

static int intarray_new(lua_State *L) {
    size_t capacity = (size_t)luaL_optinteger(L, 1, 16);
    
    IntArray *arr = (IntArray *)lua_newuserdata(L, sizeof(IntArray));
    arr->data     = NULL;
    arr->size     = 0;
    arr->capacity = 0;
    
    /* ติด metatable ก่อน allocate ข้างใน
       ถ้า malloc fail, __gc จะเรียกแต่ data=NULL ซึ่งเราต้องจัดการ */
    luaL_setmetatable(L, INT_ARRAY_MT);
    
    arr->data = (int *)malloc(capacity * sizeof(int));
    if (!arr->data) {
        return luaL_error(L, "out of memory");
    }
    arr->capacity = capacity;
    
    return 1;
}

static int intarray_gc(lua_State *L) {
    IntArray *arr = (IntArray *)luaL_checkudata(L, 1, INT_ARRAY_MT);
    if (arr->data) {
        free(arr->data);
        arr->data = NULL;
    }
    return 0;
}

static int intarray_push(lua_State *L) {
    IntArray *arr = (IntArray *)luaL_checkudata(L, 1, INT_ARRAY_MT);
    lua_Integer val = luaL_checkinteger(L, 2);
    
    /* Grow if needed */
    if (arr->size >= arr->capacity) {
        size_t new_cap = arr->capacity * 2;
        int *new_data = (int *)realloc(arr->data, new_cap * sizeof(int));
        if (!new_data) {
            return luaL_error(L, "out of memory growing array");
        }
        arr->data     = new_data;
        arr->capacity = new_cap;
    }
    
    arr->data[arr->size++] = (int)val;
    lua_pushinteger(L, (lua_Integer)arr->size);
    return 1;
}
```

---

## 85.8 Weak References จาก C

### 85.8.1 Weak Table ใน Registry

```c
/* สร้าง weak reference table ใน Lua registry */
static int create_weak_table(lua_State *L, const char *key,
                              const char *weakness) {
    /* สร้าง table */
    lua_newtable(L);
    
    /* สร้าง metatable พร้อม __mode */
    lua_newtable(L);
    lua_pushstring(L, weakness);  /* "v" = weak values, "k" = weak keys */
    lua_setfield(L, -2, "__mode");
    lua_setmetatable(L, -2);
    
    /* เก็บใน registry */
    lua_setfield(L, LUA_REGISTRYINDEX, key);
    
    return 0;
}

/* เก็บ object ด้วย weak reference */
static int store_weak_ref(lua_State *L, const char *table_key, lua_Integer id) {
    /* ดึง weak table จาก registry */
    lua_getfield(L, LUA_REGISTRYINDEX, table_key);
    
    if (lua_isnil(L, -1)) {
        lua_pop(L, 1);
        return -1;
    }
    
    /* เก็บ object: weak_table[id] = object */
    /* object อยู่บน stack ก่อนเรียก function นี้ */
    lua_pushvalue(L, -2);  /* copy object */
    lua_rawseti(L, -2, id);
    lua_pop(L, 1);  /* pop weak table */
    
    return 0;
}

/* ดึง object จาก weak reference */
static int get_weak_ref(lua_State *L, const char *table_key, lua_Integer id) {
    lua_getfield(L, LUA_REGISTRYINDEX, table_key);
    
    if (lua_isnil(L, -1)) {
        return 0;
    }
    
    lua_rawgeti(L, -1, id);
    lua_remove(L, -2);  /* remove weak table */
    
    return !lua_isnil(L, -1);  /* 1 ถ้า object ยังอยู่ */
}
```

---

## 85.9 Complete C Extension: Fast CSV Parser

นี่คือ Example ที่สมบูรณ์ของ C Extension สำหรับ parse CSV files อย่างรวดเร็ว

```c
/* csvparser.c - Fast CSV Parser C extension for Lua 5.4 */

#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

#define CSV_MT "csvparser.Parser"
#define CSV_MAX_FIELDS 4096
#define CSV_BUFFER_SIZE (64 * 1024)  /* 64KB buffer */

/* ===================== Parser State ===================== */

typedef struct {
    FILE   *fp;
    char   *buffer;
    size_t  buf_size;
    size_t  buf_pos;
    size_t  buf_len;
    char    delim;
    char    quote;
    int     has_header;
    int     closed;
    long    line_count;
    char  **headers;
    int     header_count;
} CSVParser;

/* ===================== Internal Helpers ===================== */

static int csv_fill_buffer(CSVParser *p) {
    if (p->buf_pos < p->buf_len) return 1;  /* still data */
    
    p->buf_len = fread(p->buffer, 1, p->buf_size, p->fp);
    p->buf_pos = 0;
    
    return p->buf_len > 0;
}

static int csv_next_char(CSVParser *p) {
    if (!csv_fill_buffer(p)) return -1;  /* EOF */
    return (unsigned char)p->buffer[p->buf_pos++];
}

/* Parse one field, push result onto Lua stack */
static int csv_parse_field(lua_State *L, CSVParser *p) {
    luaL_Buffer buf;
    luaL_buffinit(L, &buf);
    
    int c = csv_next_char(p);
    if (c < 0) return 0;  /* EOF */
    
    if (c == p->quote) {
        /* Quoted field */
        while (1) {
            c = csv_next_char(p);
            if (c < 0) {
                return luaL_error(L, "unterminated quoted field at line %ld",
                    p->line_count + 1);
            }
            
            if (c == p->quote) {
                /* ดู peek ahead */
                int next = csv_next_char(p);
                if (next == p->quote) {
                    /* escaped quote ("") */
                    luaL_addchar(&buf, (char)p->quote);
                } else if (next == p->delim || next == '\n' || 
                           next == '\r' || next < 0) {
                    /* end of field */
                    if (next == '\r') {
                        /* consume LF */
                        int lf = csv_next_char(p);
                        if (lf != '\n' && lf >= 0) p->buf_pos--;
                    }
                    if (next == '\n' || next == '\r') {
                        p->line_count++;
                        luaL_pushresult(&buf);
                        return -1;  /* end of line */
                    }
                    break;
                } else {
                    luaL_addchar(&buf, (char)c);
                    luaL_addchar(&buf, (char)next);
                }
            } else {
                luaL_addchar(&buf, (char)c);
            }
        }
    } else {
        /* Unquoted field */
        while (c != p->delim && c != '\n' && c != '\r' && c >= 0) {
            luaL_addchar(&buf, (char)c);
            c = csv_next_char(p);
        }
        
        if (c == '\r') {
            int lf = csv_next_char(p);
            if (lf != '\n' && lf >= 0) p->buf_pos--;
            c = '\n';
        }
        
        if (c == '\n') {
            p->line_count++;
            luaL_pushresult(&buf);
            return -1;  /* end of line */
        }
        
        if (c < 0) {
            luaL_pushresult(&buf);
            return 0;  /* EOF */
        }
    }
    
    luaL_pushresult(&buf);
    return 1;  /* more fields on this line */
}

/* Parse one complete row, return as Lua table */
static int csv_parse_row(lua_State *L, CSVParser *p) {
    lua_newtable(L);
    int field_idx = 1;
    
    while (1) {
        int status = csv_parse_field(L, p);
        
        if (status == 0 && field_idx == 1) {
            /* EOF ตั้งแต่ต้น = ไม่มีข้อมูล */
            lua_pop(L, 2);  /* pop table and field */
            return 0;
        }
        
        /* stack: [table, field_string] */
        
        if (p->has_header && p->headers && field_idx <= p->header_count) {
            /* ใช้ header เป็น key */
            lua_setfield(L, -2, p->headers[field_idx - 1]);
        } else {
            /* ใช้ index เป็น key */
            lua_rawseti(L, -2, field_idx);
        }
        
        field_idx++;
        
        if (status <= 0) {
            /* end of line or EOF */
            break;
        }
        
        if (field_idx > CSV_MAX_FIELDS) {
            return luaL_error(L, "too many fields (max %d)", CSV_MAX_FIELDS);
        }
    }
    
    return 1;  /* row table on stack */
}

/* ===================== Lua API ===================== */

/* csvparser.open(filename, [opts]) -> parser */
static int csv_open(lua_State *L) {
    const char *filename = luaL_checkstring(L, 1);
    
    /* Options table */
    char delim = ',';
    char quote = '"';
    int has_header = 0;
    
    if (lua_istable(L, 2)) {
        lua_getfield(L, 2, "delimiter");
        if (!lua_isnil(L, -1)) {
            const char *d = lua_tostring(L, -1);
            if (d && d[0]) delim = d[0];
        }
        lua_pop(L, 1);
        
        lua_getfield(L, 2, "quote");
        if (!lua_isnil(L, -1)) {
            const char *q = lua_tostring(L, -1);
            if (q && q[0]) quote = q[0];
        }
        lua_pop(L, 1);
        
        lua_getfield(L, 2, "header");
        has_header = lua_toboolean(L, -1);
        lua_pop(L, 1);
    }
    
    /* Allocate parser userdata */
    CSVParser *p = (CSVParser *)lua_newuserdata(L, sizeof(CSVParser));
    memset(p, 0, sizeof(CSVParser));
    p->delim      = delim;
    p->quote      = quote;
    p->has_header = has_header;
    p->closed     = 1;
    
    luaL_setmetatable(L, CSV_MT);
    
    /* Allocate buffer */
    p->buffer = (char *)malloc(CSV_BUFFER_SIZE);
    if (!p->buffer) {
        return luaL_error(L, "out of memory");
    }
    p->buf_size = CSV_BUFFER_SIZE;
    
    /* Open file */
    p->fp = fopen(filename, "rb");
    if (!p->fp) {
        free(p->buffer);
        p->buffer = NULL;
        return luaL_error(L, "cannot open '%s': %s", filename, strerror(errno));
    }
    p->closed = 0;
    
    /* Parse header row ถ้าต้องการ */
    if (has_header) {
        lua_newtable(L);  /* temp table for header row */
        
        /* Parse เป็น indexed table ก่อน */
        int saved_header = p->has_header;
        p->has_header = 0;
        
        if (!csv_parse_row(L, p)) {
            lua_pop(L, 2);
            return luaL_error(L, "empty CSV file");
        }
        
        p->has_header = saved_header;
        
        /* แปลงเป็น array ของ strings */
        int n = (int)lua_rawlen(L, -1);
        p->headers = (char **)malloc(n * sizeof(char *));
        if (!p->headers) {
            return luaL_error(L, "out of memory");
        }
        p->header_count = n;
        
        for (int i = 1; i <= n; i++) {
            lua_rawgeti(L, -1, i);
            const char *h = lua_tostring(L, -1);
            p->headers[i-1] = h ? strdup(h) : strdup("");
            lua_pop(L, 1);
        }
        
        lua_pop(L, 1);  /* pop temp header table */
    }
    
    return 1;  /* return parser userdata */
}

/* parser:read() -> row_table or nil */
static int csv_read(lua_State *L) {
    CSVParser *p = (CSVParser *)luaL_checkudata(L, 1, CSV_MT);
    
    if (p->closed || !p->fp) {
        return luaL_error(L, "attempt to read from closed parser");
    }
    
    if (!csv_parse_row(L, p)) {
        lua_pushnil(L);
        return 1;
    }
    
    return 1;
}

/* parser:lines() -> iterator */
static int csv_lines_iter(lua_State *L) {
    /* Upvalue 1 = parser */
    CSVParser *p = (CSVParser *)luaL_checkudata(L, lua_upvalueindex(1), CSV_MT);
    
    if (p->closed || !p->fp) {
        lua_pushnil(L);
        return 1;
    }
    
    if (!csv_parse_row(L, p)) {
        lua_pushnil(L);
        return 1;
    }
    
    return 1;
}

static int csv_lines(lua_State *L) {
    luaL_checkudata(L, 1, CSV_MT);
    
    /* Push parser as upvalue */
    lua_pushvalue(L, 1);
    lua_pushcclosure(L, csv_lines_iter, 1);
    
    return 1;
}

/* parser:close() */
static int csv_close(lua_State *L) {
    CSVParser *p = (CSVParser *)luaL_checkudata(L, 1, CSV_MT);
    
    if (!p->closed) {
        fclose(p->fp);
        p->fp     = NULL;
        p->closed = 1;
    }
    
    lua_pushboolean(L, 1);
    return 1;
}

/* __gc finalizer */
static int csv_gc(lua_State *L) {
    CSVParser *p = (CSVParser *)luaL_checkudata(L, 1, CSV_MT);
    
    if (!p->closed && p->fp) {
        fclose(p->fp);
        p->fp     = NULL;
        p->closed = 1;
    }
    
    if (p->buffer) {
        free(p->buffer);
        p->buffer = NULL;
    }
    
    if (p->headers) {
        for (int i = 0; i < p->header_count; i++) {
            free(p->headers[i]);
        }
        free(p->headers);
        p->headers = NULL;
    }
    
    return 0;
}

/* parser:stats() -> table */
static int csv_stats(lua_State *L) {
    CSVParser *p = (CSVParser *)luaL_checkudata(L, 1, CSV_MT);
    
    lua_newtable(L);
    lua_pushinteger(L, p->line_count);
    lua_setfield(L, -2, "lines_read");
    lua_pushboolean(L, p->closed);
    lua_setfield(L, -2, "closed");
    lua_pushinteger(L, p->header_count);
    lua_setfield(L, -2, "header_count");
    
    return 1;
}

/* ===================== Module Registration ===================== */

static void register_parser_mt(lua_State *L) {
    if (luaL_newmetatable(L, CSV_MT)) {
        static const luaL_Reg methods[] = {
            { "read",   csv_read   },
            { "lines",  csv_lines  },
            { "close",  csv_close  },
            { "stats",  csv_stats  },
            { NULL,     NULL       },
        };
        
        lua_newtable(L);
        luaL_setfuncs(L, methods, 0);
        lua_setfield(L, -2, "__index");
        
        lua_pushcfunction(L, csv_gc);
        lua_setfield(L, -2, "__gc");
        
        lua_pushcfunction(L, csv_close);
        lua_setfield(L, -2, "__close");
        
        lua_pushstring(L, CSV_MT);
        lua_setfield(L, -2, "__name");
    }
    lua_pop(L, 1);
}

int luaopen_csvparser(lua_State *L) {
    register_parser_mt(L);
    
    static const luaL_Reg funcs[] = {
        { "open", csv_open },
        { NULL,   NULL     },
    };
    
    luaL_newlib(L, funcs);
    
    /* Constants */
    lua_pushstring(L, "1.0.0");
    lua_setfield(L, -2, "_VERSION");
    
    return 1;
}
```

### 85.9.1 การใช้งาน CSV Parser จาก Lua

```lua
-- ตัวอย่างการใช้งาน CSV Parser C extension
local csv = require("csvparser")

-- อ่าน CSV พร้อม header
local parser = csv.open("data.csv", {
    delimiter = ",",
    header    = true,
})

-- อ่านทีละ row
local row = parser:read()
while row do
    print(row.name, row.age, row.email)
    row = parser:read()
end
parser:close()

-- ใช้ iterator
local parser2 = csv.open("large_file.csv", { header = true })
for row in parser2:lines() do
    process_row(row)
end
-- parser2:close() ถูกเรียกอัตโนมัติถ้าใช้ <close>

-- to-be-closed (Lua 5.4)
local total = 0
do
    local p <close> = csv.open("numbers.csv")
    for row in p:lines() do
        total = total + tonumber(row[1])
    end
end
-- p ถูก close อัตโนมัติ

print("Total:", total)
```

### 85.9.2 Makefile สำหรับ Build

```makefile
# Makefile สำหรับ build CSV Parser extension

LUA_INC := $(shell pkg-config --cflags lua5.4)
LUA_LIB := $(shell pkg-config --libs lua5.4)

CC      := gcc
CFLAGS  := -O2 -Wall -Wextra -fPIC $(LUA_INC)
LDFLAGS := -shared $(LUA_LIB)

TARGET := csvparser.so

$(TARGET): csvparser.c
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $<

test: $(TARGET)
	lua5.4 test_csv.lua

clean:
	rm -f $(TARGET)

.PHONY: test clean
```

---

## 85.10 Performance Tips สำหรับ C Extensions

```c
/* Tips สำหรับการเขียน C extension ที่มีประสิทธิภาพ */

/* 1. ใช้ lua_rawget/rawset แทน lua_gettable/settable เมื่อไม่ต้องการ metamethod */
static int fast_table_access(lua_State *L) {
    luaL_checktype(L, 1, LUA_TTABLE);
    
    /* ช้ากว่า: lua_gettable (ตรวจสอบ __index) */
    /* lua_pushstring(L, "key"); */
    /* lua_gettable(L, 1); */
    
    /* เร็วกว่า: lua_rawget (ข้าม metamethod) */
    lua_rawgetf(L, 1, "key");  /* Lua 5.4 */
    
    return 1;
}

/* 2. เก็บ function reference ใน upvalue แทน global lookup ทุกครั้ง */
static int setup_cached_func(lua_State *L) {
    /* ค้นหาครั้งเดียวตอน module load */
    lua_getglobal(L, "string");
    lua_getfield(L, -1, "format");
    lua_remove(L, -2);
    
    /* เก็บ reference ใน registry */
    int ref = luaL_ref(L, LUA_REGISTRYINDEX);
    
    /* เก็บ ref number ไว้ใช้ทีหลัง */
    lua_pushinteger(L, ref);
    return 1;
}

/* 3. luaL_Buffer สำหรับ build strings */
static int efficient_string_build(lua_State *L) {
    luaL_Buffer b;
    luaL_buffinit(L, &b);
    
    for (int i = 0; i < 1000; i++) {
        char tmp[32];
        int len = snprintf(tmp, sizeof(tmp), "item%d,", i);
        luaL_addlstring(&b, tmp, len);
    }
    
    luaL_pushresult(&b);
    return 1;
}

/* 4. lua_checkstack ก่อน push หลายค่า */
static int push_many_values(lua_State *L, int n) {
    if (!lua_checkstack(L, n + 5)) {
        return luaL_error(L, "stack overflow");
    }
    
    for (int i = 0; i < n; i++) {
        lua_pushinteger(L, i);
    }
    
    return n;
}
```

---

## 85.11 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Stack Manipulation
เขียน C function `lua_dump_stack(L)` ที่:
- แสดง type และ value ของทุก element บน stack
- Handle ทุก Lua types รวมถึง function, table, userdata
- ใช้ได้เป็น debugging helper ใน development

### แบบฝึกหัดที่ 2: Userdata ด้วย Finalizer
สร้าง C extension `timer` ที่:
- `timer.new()` สร้าง timer userdata
- `:start()` บันทึกเวลาเริ่มต้น
- `:elapsed()` คืนค่าเวลาที่ผ่านไปเป็น milliseconds
- `:__gc` finalizer ที่ทำ cleanup อย่างถูกต้อง
- รองรับ `tostring()` ที่แสดงสถานะ

### แบบฝึกหัดที่ 3: Error Handling
เขียน `safe_divide(a, b)` C function ที่:
- Return error table ถ้า b == 0 (ไม่ใช้ lua_error)
- Return result, nil ถ้าสำเร็จ
- Handle กรณีที่ argument ไม่ใช่ number
- Test ด้วย pcall จาก Lua

### แบบฝึกหัดที่ 4: Thread-Safe Extension
สร้าง C extension `counter` สำหรับ multi-threaded environment:
- ใช้ atomic operations (C11 `_Atomic` หรือ mutex)
- `counter.new()` สร้าง counter ใหม่ (independent ต่อแต่ละ Lua state)
- `:increment()`, `:decrement()`, `:get()`
- `:reset()` reset เป็น 0
- ทดสอบด้วย pthread ว่า thread-safe จริงๆ

### แบบฝึกหัดที่ 5: JSON Parser Extension
ออกแบบและ implement Fast JSON Parser C extension:
- รองรับ objects, arrays, strings, numbers, booleans, null
- Streaming parser สำหรับ large JSON files
- Error reporting พร้อม line/column number
- Benchmark เปรียบเทียบกับ lua-cjson

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **lua_State lifecycle** - การสร้าง, ใช้งาน และปิด Lua states อย่างถูกต้อง
2. **Threading model** - ทำไมต้องมี lua_State แยกต่างหากต่อ thread
3. **Stack manipulation** - lua_push*, lua_to*, lua_check* ที่ใช้บ่อย
4. **Userdata** - Full vs Light userdata และการสร้าง __gc finalizer
5. **Metatable** - luaL_newmetatable, method registration, OOP pattern
6. **Error handling** - lua_pcall, lua_error, luaL_error อย่างถูกต้อง
7. **lua_CFunction patterns** - Multiple returns, table manipulation, closures
8. **Weak references** - การจัดการ object lifetime จาก C
9. **Complete extension** - Fast CSV Parser พร้อม production-grade code

การเขียน C extension ที่ดีต้องให้ความสำคัญกับ:
- Memory management ที่รัดกุม (ไม่มี leak)
- Error handling ที่สมบูรณ์ (ไม่ crash)
- Thread safety (ถ้าใช้ multi-threading)
- Performance (ใช้ raw operations เมื่อเหมาะสม)
