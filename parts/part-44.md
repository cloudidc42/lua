# บทที่ 44: Embedding Lua in C Applications

## บทนำ

Embedding Lua หมายถึงการใช้ Lua เป็นส่วนหนึ่งของ C application ของเรา แทนที่จะรัน Lua interpreter แบบ standalone เราจะสร้าง application ที่มี Lua interpreter อยู่ข้างใน ทำให้ users สามารถเขียน Lua scripts เพื่อ customize พฤติกรรมของ application ได้

---

## 44.1 ทำไมต้อง Embed Lua

### ตัวอย่างที่ 1: Use Cases

```lua
-- use_cases.lua
-- ตัวอย่าง use cases สำหรับ embedded Lua

--[[
1. GAME SCRIPTING
   - เกม RPG ที่ให้ modders เขียน scripts
   - AI behaviors สำหรับ NPCs
   - Event handlers และ quest logic
   Example: World of Warcraft, Garry's Mod, Roblox

2. APPLICATION PLUGINS
   - Text editors: Neovim, LightTable
   - Web servers: nginx (OpenResty), Apache
   - Database: Redis scripting

3. CONFIGURATION FILES
   - Complex config logic (if/else, loops)
   - ดีกว่า JSON/YAML สำหรับ dynamic config

4. EMBEDDED SYSTEMS
   - IoT devices ที่ต้องการ scripting
   - เช่น NodeMCU ใช้ Lua บน ESP8266/ESP32

5. SANDBOX SCRIPTING
   - Execute user-provided code อย่างปลอดภัย
   - Restrict access to dangerous functions
--]]

-- ตัวอย่าง: Game config เป็น Lua
local gameConfig = [[
    -- Game settings
    player = {
        maxHealth = 100,
        speed = 5.0,
        jumpHeight = 10,
        startLevel = 1
    }
    
    enemies = {
        {name = "Goblin",  health = 20, damage = 5,  reward = 10},
        {name = "Orc",     health = 50, damage = 15, reward = 30},
        {name = "Dragon",  health = 200, damage = 50, reward = 100},
    }
    
    function calculateDamage(attacker, defender)
        local base = attacker.damage
        local defense = defender.defense or 0
        return math.max(1, base - defense)
    end
]]

-- parse config (simulation)
print("Use case: Game Configuration")
print("Config would be loaded from external Lua file")
print("Players/designers can modify without recompiling the game")
```

---

## 44.2 Creating และ Destroying lua_State

### ตัวอย่างที่ 2: Complete State Lifecycle

```c
/* state_lifecycle.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <stdlib.h>

/* Error handler สำหรับ lua_pcall */
static int errorHandler(lua_State *L) {
    const char *msg = lua_tostring(L, 1);
    if (msg) {
        /* เพิ่ม stack trace */
        luaL_traceback(L, L, msg, 1);
    } else {
        lua_pushliteral(L, "(error object is not a string)");
    }
    return 1;
}

/* Safe execution wrapper */
static int safeExec(lua_State *L, const char *code) {
    /* Push error handler */
    int base = lua_gettop(L);
    lua_pushcfunction(L, errorHandler);
    
    /* Load code */
    if (luaL_loadstring(L, code) != LUA_OK) {
        const char *err = lua_tostring(L, -1);
        fprintf(stderr, "Syntax error: %s\n", err);
        lua_pop(L, 2);  /* remove error + handler */
        return 0;
    }
    
    /* Execute */
    int status = lua_pcall(L, 0, LUA_MULTRET, base + 1);
    lua_remove(L, base + 1);  /* remove error handler */
    
    if (status != LUA_OK) {
        const char *err = lua_tostring(L, -1);
        fprintf(stderr, "Runtime error:\n%s\n", err);
        lua_pop(L, 1);
        return 0;
    }
    
    return 1;
}

int main(void) {
    /* === Create State === */
    lua_State *L = luaL_newstate();
    if (!L) {
        fprintf(stderr, "Failed to create Lua state\n");
        return 1;
    }
    printf("Lua state created\n");
    
    /* === Load Libraries === */
    /* luaL_openlibs: opens all standard libs */
    luaL_openlibs(L);
    printf("Standard libraries loaded\n");
    
    /* หรือ load เฉพาะ libs ที่ต้องการ */
    /* luaopen_base(L);    -- basic funcs: print, error, etc. */
    /* luaopen_math(L);    -- math library */
    /* luaopen_string(L);  -- string library */
    /* luaopen_table(L);   -- table library */
    /* luaopen_io(L);      -- I/O library */
    
    /* === Use the State === */
    safeExec(L, "print('Hello from embedded Lua!')");
    safeExec(L, "for i=1,3 do print('Line', i) end");
    
    /* Test error handling */
    safeExec(L, "error('intentional error')");
    safeExec(L, "this is invalid syntax {{{{");
    
    /* === Destroy State === */
    lua_close(L);
    printf("Lua state closed\n");
    
    return 0;
}
```

### ตัวอย่างที่ 3: State Manager

```c
/* state_manager.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* State Manager: จัดการ multiple Lua states */
#define MAX_STATES 16

typedef struct {
    lua_State *L;
    char name[64];
    int active;
} LuaStateInfo;

static LuaStateInfo states[MAX_STATES] = {0};

static LuaStateInfo* findState(const char *name) {
    for (int i = 0; i < MAX_STATES; i++) {
        if (states[i].active && strcmp(states[i].name, name) == 0) {
            return &states[i];
        }
    }
    return NULL;
}

static LuaStateInfo* createState(const char *name, int openLibs) {
    /* หา slot ว่าง */
    for (int i = 0; i < MAX_STATES; i++) {
        if (!states[i].active) {
            states[i].L = luaL_newstate();
            if (!states[i].L) return NULL;
            
            if (openLibs) luaL_openlibs(states[i].L);
            
            strncpy(states[i].name, name, 63);
            states[i].active = 1;
            return &states[i];
        }
    }
    return NULL;  /* no free slot */
}

static void destroyState(const char *name) {
    LuaStateInfo *info = findState(name);
    if (info) {
        lua_close(info->L);
        memset(info, 0, sizeof(LuaStateInfo));
    }
}

static int runInState(const char *name, const char *code) {
    LuaStateInfo *info = findState(name);
    if (!info) {
        fprintf(stderr, "State '%s' not found\n", name);
        return 0;
    }
    
    if (luaL_dostring(info->L, code) != LUA_OK) {
        fprintf(stderr, "[%s] Error: %s\n", name, lua_tostring(info->L, -1));
        lua_pop(info->L, 1);
        return 0;
    }
    return 1;
}

int main(void) {
    /* สร้าง states ต่างๆ */
    createState("game", 1);
    createState("ui", 1);
    createState("sandbox", 0);  /* sandbox: no standard libs */
    
    printf("Created 3 Lua states\n\n");
    
    /* รัน code ใน states ต่างๆ */
    runInState("game", "x = 100; print('Game state: x =', x)");
    runInState("ui",   "x = 200; print('UI state: x =', x)");
    
    /* States แยกกัน - x ไม่ข้าม */
    runInState("game", "print('Game x still:', x)");
    runInState("ui",   "print('UI x still:', x)");
    
    /* Sandbox ไม่มี print */
    runInState("sandbox", "return 1 + 2");
    
    /* Cleanup */
    destroyState("game");
    destroyState("ui");
    destroyState("sandbox");
    printf("\nAll states destroyed\n");
    
    return 0;
}
```

---

## 44.3 Running Lua Scripts

### ตัวอย่างที่ 4: Load from File

```c
/* run_scripts.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

/* รัน Lua file */
static int runFile(lua_State *L, const char *filename) {
    /* luaL_loadfile: compile file */
    if (luaL_loadfile(L, filename) != LUA_OK) {
        fprintf(stderr, "Load error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
        return 0;
    }
    
    /* รัน compiled chunk */
    if (lua_pcall(L, 0, 0, 0) != LUA_OK) {
        fprintf(stderr, "Runtime error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
        return 0;
    }
    
    return 1;
}

/* รัน Lua string */
static int runString(lua_State *L, const char *code) {
    return luaL_dostring(L, code) == LUA_OK;
}

/* รัน Lua buffer */
static int runBuffer(lua_State *L, const char *buf, size_t size, const char *name) {
    if (luaL_loadbuffer(L, buf, size, name) != LUA_OK) {
        fprintf(stderr, "Buffer load error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
        return 0;
    }
    
    if (lua_pcall(L, 0, 0, 0) != LUA_OK) {
        fprintf(stderr, "Buffer runtime error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
        return 0;
    }
    
    return 1;
}

int main(void) {
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    
    /* รัน string */
    printf("=== Running string ===\n");
    runString(L, "print('Hello from string!')");
    
    /* รัน buffer (เหมือน string แต่กำหนด size ได้) */
    printf("\n=== Running buffer ===\n");
    const char *code = "for i = 1, 3 do print('Buffer line', i) end";
    runBuffer(L, code, strlen(code), "@buffer_example");
    
    /* รัน file (ถ้ามี) */
    printf("\n=== Trying to run file ===\n");
    if (!runFile(L, "config.lua")) {
        printf("File not found, that's OK for this demo\n");
    }
    
    lua_close(L);
    return 0;
}
```

### ตัวอย่างที่ 5: Hot Reload Scripts

```c
/* hot_reload.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <time.h>
#include <sys/stat.h>

/* ตรวจสอบเวลา modify ของไฟล์ */
static time_t getFileModTime(const char *filename) {
    struct stat st;
    if (stat(filename, &st) != 0) return 0;
    return st.st_mtime;
}

typedef struct {
    lua_State *L;
    char filename[256];
    time_t lastModTime;
} HotReloader;

static HotReloader* createHotReloader(const char *filename) {
    HotReloader *hr = (HotReloader*)malloc(sizeof(HotReloader));
    hr->L = luaL_newstate();
    luaL_openlibs(hr->L);
    strncpy(hr->filename, filename, 255);
    hr->lastModTime = 0;
    return hr;
}

static int loadOrReload(HotReloader *hr) {
    time_t modTime = getFileModTime(hr->filename);
    if (modTime == hr->lastModTime) return 0;  /* no change */
    
    printf("Loading/Reloading: %s\n", hr->filename);
    
    if (luaL_dofile(hr->L, hr->filename) != LUA_OK) {
        fprintf(stderr, "Error loading %s: %s\n",
            hr->filename, lua_tostring(hr->L, -1));
        lua_pop(hr->L, 1);
        return -1;
    }
    
    hr->lastModTime = modTime;
    return 1;  /* reloaded */
}

/* ใน loop หลักของ game/app */
static void mainLoop(HotReloader *hr, int iterations) {
    for (int i = 0; i < iterations; i++) {
        /* ตรวจสอบ reload ทุก iteration */
        int result = loadOrReload(hr);
        if (result > 0) {
            printf("Script reloaded successfully (iteration %d)\n", i);
        }
        
        /* เรียก update function ถ้ามี */
        lua_getglobal(hr->L, "update");
        if (lua_isfunction(hr->L, -1)) {
            lua_pushinteger(hr->L, i);
            if (lua_pcall(hr->L, 1, 0, 0) != LUA_OK) {
                fprintf(stderr, "update error: %s\n", lua_tostring(hr->L, -1));
                lua_pop(hr->L, 1);
            }
        } else {
            lua_pop(hr->L, 1);
        }
    }
}

/* Simulation: สร้าง test script ก่อน */
static void createTestScript(const char *filename) {
    FILE *f = fopen(filename, "w");
    if (!f) return;
    fprintf(f, "-- Auto-generated test script\n");
    fprintf(f, "function update(frame)\n");
    fprintf(f, "  if frame %% 5 == 0 then\n");
    fprintf(f, "    print('Frame:', frame)\n");
    fprintf(f, "  end\n");
    fprintf(f, "end\n");
    fclose(f);
}

int main(void) {
    const char *scriptFile = "/tmp/hot_reload_test.lua";
    createTestScript(scriptFile);
    
    HotReloader *hr = createHotReloader(scriptFile);
    mainLoop(hr, 10);
    
    lua_close(hr->L);
    free(hr);
    
    printf("Hot reload demo complete\n");
    return 0;
}
```

---

## 44.4 Calling Lua Functions from C

### ตัวอย่างที่ 6: Basic Function Calls

```c
/* call_lua_funcs.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

/* เรียก Lua function ที่ return single value */
static double callLuaMath(lua_State *L, const char *funcName, double a, double b) {
    lua_getglobal(L, funcName);
    
    if (!lua_isfunction(L, -1)) {
        lua_pop(L, 1);
        fprintf(stderr, "'%s' is not a function\n", funcName);
        return 0.0;
    }
    
    lua_pushnumber(L, a);
    lua_pushnumber(L, b);
    
    if (lua_pcall(L, 2, 1, 0) != LUA_OK) {
        fprintf(stderr, "Error calling %s: %s\n", funcName, lua_tostring(L, -1));
        lua_pop(L, 1);
        return 0.0;
    }
    
    double result = lua_tonumber(L, -1);
    lua_pop(L, 1);
    return result;
}

/* เรียก Lua function ที่ return string */
static const char* callLuaString(lua_State *L, const char *funcName, const char *arg) {
    lua_getglobal(L, funcName);
    lua_pushstring(L, arg);
    
    if (lua_pcall(L, 1, 1, 0) != LUA_OK) {
        fprintf(stderr, "Error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
        return NULL;
    }
    
    /* IMPORTANT: string จะ invalid หลัง lua_pop */
    /* ต้อง copy ถ้าจะใช้ต่อ */
    static char buffer[1024];
    const char *s = lua_tostring(L, -1);
    if (s) strncpy(buffer, s, 1023);
    else buffer[0] = '\0';
    lua_pop(L, 1);
    
    return buffer;
}

/* เรียก Lua function ที่ return table */
static void callLuaGetStats(lua_State *L, int *data, int n) {
    /* สร้าง Lua array จาก C array */
    lua_getglobal(L, "computeStats");
    lua_newtable(L);
    for (int i = 0; i < n; i++) {
        lua_pushinteger(L, data[i]);
        lua_seti(L, -2, i + 1);
    }
    
    if (lua_pcall(L, 1, 1, 0) != LUA_OK) {
        fprintf(stderr, "computeStats error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
        return;
    }
    
    /* อ่าน result table */
    if (lua_istable(L, -1)) {
        lua_getfield(L, -1, "avg");
        lua_getfield(L, -2, "min");
        lua_getfield(L, -3, "max");
        
        printf("Stats: avg=%.2f, min=%g, max=%g\n",
            lua_tonumber(L, -3),
            lua_tonumber(L, -2),
            lua_tonumber(L, -1));
        lua_pop(L, 3);
    }
    lua_pop(L, 1);  /* remove table */
}

int main(void) {
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    
    /* Define Lua functions */
    luaL_dostring(L,
        "function add(a, b) return a + b end\n"
        "function mul(a, b) return a * b end\n"
        "function greet(name) return 'Hello, ' .. name .. '!' end\n"
        "function computeStats(t)\n"
        "  local sum, mn, mx = 0, t[1], t[1]\n"
        "  for _, v in ipairs(t) do\n"
        "    sum = sum + v\n"
        "    if v < mn then mn = v end\n"
        "    if v > mx then mx = v end\n"
        "  end\n"
        "  return {avg=sum/#t, min=mn, max=mx}\n"
        "end\n"
    );
    
    printf("=== Calling Lua math functions ===\n");
    printf("add(3, 4) = %.0f\n", callLuaMath(L, "add", 3, 4));
    printf("mul(5, 7) = %.0f\n", callLuaMath(L, "mul", 5, 7));
    
    printf("\n=== Calling Lua string function ===\n");
    printf("%s\n", callLuaString(L, "greet", "World"));
    printf("%s\n", callLuaString(L, "greet", "สวัสดี"));
    
    printf("\n=== Calling Lua stats function ===\n");
    int data[] = {5, 2, 8, 1, 9, 3, 7, 4, 6};
    callLuaGetStats(L, data, 9);
    
    lua_close(L);
    return 0;
}
```

---

## 44.5 Passing Data Between C and Lua

### ตัวอย่างที่ 7: Data Exchange Patterns

```c
/* data_exchange.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <string.h>

/* C struct ที่ต้องการส่งไป Lua */
typedef struct {
    char name[64];
    int level;
    float health;
    float maxHealth;
    int skills[8];
    int skillCount;
} Player;

/* แปลง C Player -> Lua table */
static void pushPlayer(lua_State *L, const Player *p) {
    lua_newtable(L);
    
    lua_pushstring(L, p->name);
    lua_setfield(L, -2, "name");
    
    lua_pushinteger(L, p->level);
    lua_setfield(L, -2, "level");
    
    lua_pushnumber(L, p->health);
    lua_setfield(L, -2, "health");
    
    lua_pushnumber(L, p->maxHealth);
    lua_setfield(L, -2, "maxHealth");
    
    /* Push skills array */
    lua_newtable(L);
    for (int i = 0; i < p->skillCount; i++) {
        lua_pushinteger(L, p->skills[i]);
        lua_seti(L, -2, i + 1);
    }
    lua_setfield(L, -2, "skills");
    
    /* Computed field: healthPercent */
    lua_pushnumber(L, p->maxHealth > 0 ? p->health / p->maxHealth * 100.0f : 0);
    lua_setfield(L, -2, "healthPercent");
}

/* แปลง Lua table -> C Player */
static void pullPlayer(lua_State *L, int idx, Player *p) {
    memset(p, 0, sizeof(Player));
    
    lua_getfield(L, idx, "name");
    if (lua_isstring(L, -1)) {
        strncpy(p->name, lua_tostring(L, -1), 63);
    }
    lua_pop(L, 1);
    
    lua_getfield(L, idx, "level");
    p->level = (int)lua_tointeger(L, -1);
    lua_pop(L, 1);
    
    lua_getfield(L, idx, "health");
    p->health = (float)lua_tonumber(L, -1);
    lua_pop(L, 1);
    
    lua_getfield(L, idx, "maxHealth");
    p->maxHealth = (float)lua_tonumber(L, -1);
    lua_pop(L, 1);
    
    /* Read skills array */
    lua_getfield(L, idx, "skills");
    if (lua_istable(L, -1)) {
        int n = (int)luaL_len(L, -1);
        p->skillCount = n < 8 ? n : 8;
        for (int i = 0; i < p->skillCount; i++) {
            lua_geti(L, -1, i + 1);
            p->skills[i] = (int)lua_tointeger(L, -1);
            lua_pop(L, 1);
        }
    }
    lua_pop(L, 1);
}

int main(void) {
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    
    /* สร้าง Player ใน C */
    Player hero = {
        .name = "Aragorn",
        .level = 10,
        .health = 85.5f,
        .maxHealth = 100.0f,
        .skills = {1, 3, 5, 7},
        .skillCount = 4
    };
    
    /* ส่ง Player ไป Lua */
    printf("=== C -> Lua ===\n");
    pushPlayer(L, &hero);
    lua_setglobal(L, "player");
    
    /* ใน Lua: อ่านและแก้ไข player */
    luaL_dostring(L,
        "print('Player:', player.name)\n"
        "print('Level:', player.level)\n"
        "print('HP:', player.health .. '/' .. player.maxHealth)\n"
        "print('HP%:', string.format('%.1f%%', player.healthPercent))\n"
        "print('Skills:', table.concat(player.skills, ', '))\n"
        "\n"
        "-- Lua modifies player\n"
        "player.level = player.level + 1\n"
        "player.health = 100.0\n"
        "player.skills[5] = 9\n"
    );
    
    /* ดึง Player กลับจาก Lua */
    printf("\n=== Lua -> C ===\n");
    lua_getglobal(L, "player");
    Player modified;
    pullPlayer(L, -1, &modified);
    lua_pop(L, 1);
    
    printf("Modified level: %d\n", modified.level);
    printf("Modified health: %.1f\n", modified.health);
    printf("Modified skills: ");
    for (int i = 0; i < modified.skillCount; i++) {
        printf("%d ", modified.skills[i]);
    }
    printf("\n");
    
    lua_close(L);
    return 0;
}
```

---

## 44.6 Lua เป็น Config Language

### ตัวอย่างที่ 8: Application Config ใน Lua

```c
/* lua_config.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

typedef struct {
    char host[256];
    int  port;
    int  maxConnections;
    int  timeout;
    char logLevel[16];
    char logFile[256];
    int  debugMode;
    char allowedHosts[16][64];
    int  allowedHostCount;
} ServerConfig;

/* Helper: อ่าน string field */
static void getStringField(lua_State *L, int idx, const char *field,
                            char *dest, size_t maxLen, const char *defaultVal) {
    lua_getfield(L, idx, field);
    const char *val = lua_isstring(L, -1) ? lua_tostring(L, -1) : defaultVal;
    if (val) strncpy(dest, val, maxLen - 1);
    lua_pop(L, 1);
}

/* Helper: อ่าน int field */
static int getIntField(lua_State *L, int idx, const char *field, int defaultVal) {
    lua_getfield(L, idx, field);
    int val = lua_isnumber(L, -1) ? (int)lua_tointeger(L, -1) : defaultVal;
    lua_pop(L, 1);
    return val;
}

/* Helper: อ่าน bool field */
static int getBoolField(lua_State *L, int idx, const char *field, int defaultVal) {
    lua_getfield(L, idx, field);
    int val = lua_isboolean(L, -1) ? lua_toboolean(L, -1) : defaultVal;
    lua_pop(L, 1);
    return val;
}

/* Load config จาก Lua */
static int loadConfig(lua_State *L, const char *configCode, ServerConfig *cfg) {
    memset(cfg, 0, sizeof(ServerConfig));
    
    /* รัน config script */
    if (luaL_dostring(L, configCode) != LUA_OK) {
        fprintf(stderr, "Config error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
        return 0;
    }
    
    /* ดึง config table */
    lua_getglobal(L, "config");
    if (!lua_istable(L, -1)) {
        fprintf(stderr, "Config: 'config' must be a table\n");
        lua_pop(L, 1);
        return 0;
    }
    
    int configIdx = lua_gettop(L);
    
    /* Server settings */
    lua_getfield(L, configIdx, "server");
    if (lua_istable(L, -1)) {
        getStringField(L, -1, "host", cfg->host, 256, "localhost");
        cfg->port = getIntField(L, -1, "port", 8080);
        cfg->maxConnections = getIntField(L, -1, "maxConnections", 100);
        cfg->timeout = getIntField(L, -1, "timeout", 30);
    }
    lua_pop(L, 1);
    
    /* Logging settings */
    lua_getfield(L, configIdx, "logging");
    if (lua_istable(L, -1)) {
        getStringField(L, -1, "level", cfg->logLevel, 16, "INFO");
        getStringField(L, -1, "file", cfg->logFile, 256, "app.log");
    }
    lua_pop(L, 1);
    
    /* Debug mode */
    cfg->debugMode = getBoolField(L, configIdx, "debug", 0);
    
    /* Allowed hosts array */
    lua_getfield(L, configIdx, "allowedHosts");
    if (lua_istable(L, -1)) {
        int n = (int)luaL_len(L, -1);
        cfg->allowedHostCount = n < 16 ? n : 16;
        for (int i = 0; i < cfg->allowedHostCount; i++) {
            lua_geti(L, -1, i + 1);
            const char *host = lua_tostring(L, -1);
            if (host) strncpy(cfg->allowedHosts[i], host, 63);
            lua_pop(L, 1);
        }
    }
    lua_pop(L, 1);
    
    /* Validate */
    lua_getfield(L, configIdx, "validate");
    if (lua_isfunction(L, -1)) {
        lua_pushvalue(L, configIdx);
        if (lua_pcall(L, 1, 1, 0) == LUA_OK) {
            int valid = lua_toboolean(L, -1);
            if (!valid) {
                fprintf(stderr, "Config validation failed\n");
                lua_pop(L, 2);
                return 0;
            }
        }
        lua_pop(L, 1);
    } else {
        lua_pop(L, 1);
    }
    
    lua_pop(L, 1);  /* pop config table */
    return 1;
}

int main(void) {
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    
    /* Config ที่เขียนเป็น Lua */
    const char *configCode =
        "-- Application Configuration\n"
        "local ENV = os.getenv('APP_ENV') or 'development'\n"
        "\n"
        "config = {\n"
        "  server = {\n"
        "    host = ENV == 'production' and '0.0.0.0' or 'localhost',\n"
        "    port = 8080,\n"
        "    maxConnections = ENV == 'production' and 1000 or 10,\n"
        "    timeout = 30\n"
        "  },\n"
        "  logging = {\n"
        "    level = ENV == 'production' and 'WARN' or 'DEBUG',\n"
        "    file = '/var/log/app.log'\n"
        "  },\n"
        "  debug = ENV ~= 'production',\n"
        "  allowedHosts = {'localhost', '127.0.0.1', '::1'},\n"
        "\n"
        "  -- Validation function\n"
        "  validate = function(cfg)\n"
        "    if cfg.server.port < 1 or cfg.server.port > 65535 then\n"
        "      return false, 'Invalid port'\n"
        "    end\n"
        "    return true\n"
        "  end\n"
        "}\n";
    
    ServerConfig cfg;
    if (loadConfig(L, configCode, &cfg)) {
        printf("=== Server Configuration ===\n");
        printf("Host: %s\n", cfg.host);
        printf("Port: %d\n", cfg.port);
        printf("Max connections: %d\n", cfg.maxConnections);
        printf("Timeout: %ds\n", cfg.timeout);
        printf("Log level: %s\n", cfg.logLevel);
        printf("Log file: %s\n", cfg.logFile);
        printf("Debug mode: %s\n", cfg.debugMode ? "yes" : "no");
        printf("Allowed hosts (%d):\n", cfg.allowedHostCount);
        for (int i = 0; i < cfg.allowedHostCount; i++) {
            printf("  - %s\n", cfg.allowedHosts[i]);
        }
    }
    
    lua_close(L);
    return 0;
}
```

---

## 44.7 Sandboxed Execution

### ตัวอย่างที่ 9: Sandbox Environment

```c
/* sandbox.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <string.h>

/* สร้าง sandbox environment */
static void createSandbox(lua_State *L) {
    /* สร้าง table ที่จะเป็น environment */
    lua_newtable(L);
    
    /* อนุญาต functions เหล่านี้ */
    const char *allowed[] = {
        "print", "tostring", "tonumber", "type",
        "pairs", "ipairs", "next", "select", "unpack",
        "error", "assert", "pcall", "xpcall",
        "setmetatable", "getmetatable",
        "rawget", "rawset", "rawequal", "rawlen",
        NULL
    };
    
    for (int i = 0; allowed[i]; i++) {
        lua_getglobal(L, allowed[i]);
        if (!lua_isnil(L, -1)) {
            lua_setfield(L, -2, allowed[i]);
        } else {
            lua_pop(L, 1);
        }
    }
    
    /* อนุญาต math library */
    lua_getglobal(L, "math");
    lua_setfield(L, -2, "math");
    
    /* อนุญาต string library */
    lua_getglobal(L, "string");
    lua_setfield(L, -2, "string");
    
    /* อนุญาต table library */
    lua_getglobal(L, "table");
    lua_setfield(L, -2, "table");
    
    /* ไม่อนุญาต: io, os, dofile, loadfile, require, debug */
    
    /* เพิ่ม restricted print ที่จำกัด output */
    lua_pushcfunction(L, (lua_CFunction)(lua_CFunction)NULL);
    /* สำหรับ demo เราไม่สร้าง custom print */
}

/* รัน code ใน sandbox */
static int runInSandbox(lua_State *L, const char *code, int maxInstructions) {
    /* Load code */
    if (luaL_loadstring(L, code) != LUA_OK) {
        fprintf(stderr, "Syntax error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
        return 0;
    }
    
    /* สร้าง sandbox environment */
    lua_newtable(L);  /* sandbox env */
    
    /* Allow safe globals */
    const char *safeFuncs[] = {
        "print", "tostring", "tonumber", "type",
        "pairs", "ipairs", "math", "string", "table",
        "select", "error", "pcall", "assert",
        NULL
    };
    
    for (int i = 0; safeFuncs[i]; i++) {
        lua_getglobal(L, safeFuncs[i]);
        lua_setfield(L, -2, safeFuncs[i]);
    }
    
    /* Set chunk environment (Lua 5.2+) */
    /* In Lua 5.4, we use upvalue _ENV */
    lua_setupvalue(L, -2, 1);
    
    /* รัน */
    if (lua_pcall(L, 0, LUA_MULTRET, 0) != LUA_OK) {
        fprintf(stderr, "Sandbox error: %s\n", lua_tostring(L, -1));
        lua_pop(L, 1);
        return 0;
    }
    
    return 1;
}

int main(void) {
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    
    printf("=== Sandbox Tests ===\n");
    
    /* Safe code - ควรทำงานได้ */
    printf("\nTest 1: Safe code\n");
    runInSandbox(L,
        "local result = {}\n"
        "for i = 1, 5 do\n"
        "  result[i] = i * i\n"
        "end\n"
        "print('Squares:', table.concat(result, ', '))\n",
        10000);
    
    /* Dangerous code - ควร fail */
    printf("\nTest 2: Dangerous code (io access)\n");
    runInSandbox(L,
        "local f = io.open('/etc/passwd', 'r')\n"
        "if f then print(f:read('*all')) end\n",
        1000);
    
    /* Dangerous code - require */
    printf("\nTest 3: Dangerous code (require)\n");
    runInSandbox(L,
        "local os = require('os')\n"
        "os.execute('ls -la')\n",
        1000);
    
    /* Math heavy - should work */
    printf("\nTest 4: Math computation\n");
    runInSandbox(L,
        "local function fib(n)\n"
        "  if n <= 1 then return n end\n"
        "  return fib(n-1) + fib(n-2)\n"
        "end\n"
        "print('fib(10) =', fib(10))\n",
        100000);
    
    lua_close(L);
    return 0;
}
```

### ตัวอย่างที่ 10: Instruction Count Limiting

```c
/* instruction_limit.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

/* Hook สำหรับจำกัด instructions */
static void instructionCountHook(lua_State *L, lua_Debug *ar) {
    (void)ar;
    /* ยกเว้น error เพื่อหยุดการทำงาน */
    luaL_error(L, "Script execution timeout: too many instructions");
}

/* รัน code พร้อม instruction limit */
static int runWithLimit(lua_State *L, const char *code, int maxInstructions) {
    /* ตั้ง debug hook */
    lua_sethook(L, instructionCountHook, LUA_MASKCOUNT, maxInstructions);
    
    int result = 1;
    if (luaL_dostring(L, code) != LUA_OK) {
        const char *err = lua_tostring(L, -1);
        if (err && strstr(err, "timeout")) {
            fprintf(stderr, "TIMEOUT: Code exceeded %d instructions\n", maxInstructions);
        } else {
            fprintf(stderr, "Error: %s\n", err);
        }
        lua_pop(L, 1);
        result = 0;
    }
    
    /* ลบ hook */
    lua_sethook(L, NULL, 0, 0);
    return result;
}

int main(void) {
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    
    printf("=== Instruction Limit Tests ===\n");
    
    /* Code ที่รันเร็ว - ควรผ่าน */
    printf("\nTest 1: Fast code (limit=100000)\n");
    runWithLimit(L,
        "local sum = 0\n"
        "for i = 1, 100 do sum = sum + i end\n"
        "print('Sum:', sum)\n",
        100000);
    
    /* Infinite loop - ควร timeout */
    printf("\nTest 2: Infinite loop (limit=1000)\n");
    runWithLimit(L,
        "while true do end\n",
        1000);
    
    /* Recursive function - ควร timeout */
    printf("\nTest 3: Deep recursion (limit=5000)\n");
    runWithLimit(L,
        "local function inf(n)\n"
        "  return inf(n + 1)\n"
        "end\n"
        "inf(0)\n",
        5000);
    
    /* Normal code after timeout test */
    printf("\nTest 4: Normal code after timeout\n");
    runWithLimit(L,
        "print('Still working after timeout test!')\n",
        100000);
    
    lua_close(L);
    return 0;
}
```

---

## 44.8 Lua Coroutines from C

### ตัวอย่างที่ 11: Coroutines in C

```c
/* coroutines_from_c.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>

/* สร้างและรัน coroutine */
static void demonstrateCoroutine(lua_State *L) {
    /* Define Lua generator function */
    luaL_dostring(L,
        "function counter(start, stop, step)\n"
        "  for i = start, stop, step do\n"
        "    coroutine.yield(i)\n"
        "  end\n"
        "  return 'done'\n"
        "end\n"
    );
    
    /* ดึง counter function */
    lua_getglobal(L, "counter");
    
    /* สร้าง coroutine */
    lua_State *co = lua_newthread(L);
    lua_xmove(L, co, 1);  /* move function to coroutine thread */
    
    /* ส่ง arguments */
    lua_pushinteger(co, 1);    /* start */
    lua_pushinteger(co, 10);   /* stop */
    lua_pushinteger(co, 2);    /* step */
    
    printf("=== Coroutine Generator ===\n");
    
    /* Resume coroutine ซ้ำๆ */
    int status;
    do {
        status = lua_resume(co, L, lua_gettop(co) - (co == L ? 0 : 0), &(int){0});
        
        if (status == LUA_YIELD) {
            /* coroutine yielded - ดึงค่าที่ yield */
            int n = lua_gettop(co);
            printf("Yielded %d value(s):", n);
            for (int i = 1; i <= n; i++) {
                printf(" %s", lua_tostring(co, i));
            }
            printf("\n");
            lua_pop(co, n);  /* clear yielded values */
        } else if (status == LUA_OK) {
            /* coroutine completed */
            int n = lua_gettop(co);
            if (n > 0) {
                printf("Completed with: %s\n", lua_tostring(co, -1));
            } else {
                printf("Completed\n");
            }
        } else {
            /* error */
            fprintf(stderr, "Coroutine error: %s\n", lua_tostring(co, -1));
        }
    } while (status == LUA_YIELD);
    
    lua_pop(L, 1);  /* pop thread */
}

/* Producer-Consumer pattern */
static void producerConsumer(lua_State *L) {
    luaL_dostring(L,
        "function producer(items)\n"
        "  for _, item in ipairs(items) do\n"
        "    coroutine.yield(item)\n"
        "  end\n"
        "end\n"
    );
    
    lua_getglobal(L, "producer");
    lua_State *co = lua_newthread(L);
    lua_xmove(L, co, 1);
    
    /* สร้าง items array */
    lua_newtable(co);
    const char *items[] = {"apple", "banana", "cherry", "date", "elderberry"};
    for (int i = 0; i < 5; i++) {
        lua_pushstring(co, items[i]);
        lua_seti(co, -2, i + 1);
    }
    
    printf("\n=== Producer-Consumer ===\n");
    
    int narg = 1;
    int status;
    do {
        int nres;
        status = lua_resume(co, L, narg, &nres);
        narg = 0;  /* no more args after first resume */
        
        if (status == LUA_YIELD && nres > 0) {
            /* "Consume" the item */
            const char *item = lua_tostring(co, -1);
            printf("Consumed: %s\n", item);
            lua_pop(co, nres);
        }
    } while (status == LUA_YIELD);
    
    lua_pop(L, 1);
}

int main(void) {
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    
    demonstrateCoroutine(L);
    producerConsumer(L);
    
    lua_close(L);
    return 0;
}
```

---

## 44.9 Custom Memory Allocators

### ตัวอย่างที่ 12: Custom Allocator

```c
/* custom_allocator.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* Tracking allocator */
typedef struct {
    size_t totalAllocated;
    size_t totalFreed;
    size_t currentUsage;
    size_t peakUsage;
    size_t allocationCount;
    size_t freeCount;
    size_t maxMemory;  /* 0 = unlimited */
} AllocStats;

static AllocStats gStats = {0};

/* Custom allocator function */
static void* trackingAlloc(void *ud, void *ptr, size_t osize, size_t nsize) {
    AllocStats *stats = (AllocStats*)ud;
    
    if (nsize == 0) {
        /* Free */
        if (ptr) {
            stats->totalFreed += osize;
            stats->currentUsage -= osize;
            stats->freeCount++;
            free(ptr);
        }
        return NULL;
    }
    
    if (stats->maxMemory > 0 && 
        stats->currentUsage + (nsize - osize) > stats->maxMemory) {
        /* Reject: would exceed memory limit */
        fprintf(stderr, "[ALLOC] Memory limit exceeded: limit=%zu, current=%zu, requested=%zu\n",
            stats->maxMemory, stats->currentUsage, nsize);
        return NULL;
    }
    
    void *newPtr = realloc(ptr, nsize);
    
    if (newPtr) {
        if (ptr == NULL) {
            /* New allocation */
            stats->totalAllocated += nsize;
            stats->currentUsage += nsize;
            stats->allocationCount++;
        } else {
            /* Resize */
            stats->totalAllocated += (nsize > osize) ? (nsize - osize) : 0;
            stats->totalFreed     += (osize > nsize) ? (osize - nsize) : 0;
            stats->currentUsage   += nsize - osize;
        }
        
        if (stats->currentUsage > stats->peakUsage) {
            stats->peakUsage = stats->currentUsage;
        }
    }
    
    return newPtr;
}

/* สร้าง Lua state ด้วย custom allocator */
static lua_State* createTrackedState(size_t maxMemory) {
    gStats = (AllocStats){.maxMemory = maxMemory};
    
    /* lua_newstate: สร้างด้วย custom allocator */
    lua_State *L = lua_newstate(trackingAlloc, &gStats);
    if (L) {
        luaL_openlibs(L);
    }
    return L;
}

static void printStats(const char *label) {
    printf("\n=== Memory Stats: %s ===\n", label);
    printf("  Current usage:  %8zu bytes\n", gStats.currentUsage);
    printf("  Peak usage:     %8zu bytes\n", gStats.peakUsage);
    printf("  Total allocated:%8zu bytes\n", gStats.totalAllocated);
    printf("  Total freed:    %8zu bytes\n", gStats.totalFreed);
    printf("  Allocations:    %8zu\n", gStats.allocationCount);
    printf("  Frees:          %8zu\n", gStats.freeCount);
    if (gStats.maxMemory > 0) {
        printf("  Memory limit:   %8zu bytes\n", gStats.maxMemory);
        printf("  Usage %%:        %.1f%%\n",
            (double)gStats.currentUsage / gStats.maxMemory * 100);
    }
}

int main(void) {
    /* สร้าง state พร้อม memory tracking (limit 10MB) */
    lua_State *L = createTrackedState(10 * 1024 * 1024);
    if (!L) {
        fprintf(stderr, "Failed to create state\n");
        return 1;
    }
    
    printStats("After init");
    
    /* รัน code */
    luaL_dostring(L, "local t = {}; for i=1,10000 do t[i] = i*i end");
    printStats("After allocating 10000 numbers");
    
    luaL_dostring(L, "local s = string.rep('x', 100000)");
    printStats("After allocating 100KB string");
    
    /* Garbage collect */
    lua_gc(L, LUA_GCCOLLECT, 0);
    printStats("After GC");
    
    lua_close(L);
    printStats("After close");
    
    return 0;
}
```

---

## 44.10 Production Embedding Patterns

### ตัวอย่างที่ 13: Plugin System

```c
/* plugin_system.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#define MAX_PLUGINS 32
#define PLUGIN_API_VERSION 1

typedef struct {
    char name[64];
    char version[16];
    int  apiVersion;
    int  enabled;
    lua_State *state;
} Plugin;

static Plugin plugins[MAX_PLUGINS] = {0};
static int pluginCount = 0;

/* C API ที่ expose ให้ plugins */
static int api_log(lua_State *L) {
    const char *level = luaL_optstring(L, 1, "INFO");
    const char *msg   = luaL_checkstring(L, 2);
    
    /* ดึง plugin name จาก registry */
    lua_getfield(L, LUA_REGISTRYINDEX, "_PLUGIN_NAME");
    const char *name = lua_isstring(L, -1) ? lua_tostring(L, -1) : "unknown";
    lua_pop(L, 1);
    
    printf("[%s] [%s] %s\n", level, name, msg);
    return 0;
}

static int api_getConfig(lua_State *L) {
    const char *key = luaL_checkstring(L, 1);
    
    /* Simple mock config */
    if (strcmp(key, "maxItems") == 0) lua_pushinteger(L, 100);
    else if (strcmp(key, "debug") == 0) lua_pushboolean(L, 0);
    else lua_pushnil(L);
    
    return 1;
}

static const luaL_Reg apiLib[] = {
    {"log",       api_log},
    {"getConfig", api_getConfig},
    {NULL, NULL}
};

/* สร้าง plugin state */
static Plugin* loadPlugin(const char *name, const char *code) {
    if (pluginCount >= MAX_PLUGINS) {
        fprintf(stderr, "Plugin limit reached\n");
        return NULL;
    }
    
    Plugin *p = &plugins[pluginCount];
    strncpy(p->name, name, 63);
    p->state = luaL_newstate();
    if (!p->state) return NULL;
    
    /* Load safe libraries only */
    luaL_requiref(p->state, "base", luaopen_base, 1); lua_pop(p->state, 1);
    luaL_requiref(p->state, "math", luaopen_math, 1); lua_pop(p->state, 1);
    luaL_requiref(p->state, "string", luaopen_string, 1); lua_pop(p->state, 1);
    luaL_requiref(p->state, "table", luaopen_table, 1); lua_pop(p->state, 1);
    
    /* เพิ่ม plugin API */
    luaL_newlib(p->state, apiLib);
    lua_setglobal(p->state, "api");
    
    /* บันทึก plugin name ใน registry */
    lua_pushstring(p->state, name);
    lua_setfield(p->state, LUA_REGISTRYINDEX, "_PLUGIN_NAME");
    
    /* โหลด plugin code */
    if (luaL_dostring(p->state, code) != LUA_OK) {
        fprintf(stderr, "Plugin '%s' load error: %s\n",
            name, lua_tostring(p->state, -1));
        lua_pop(p->state, 1);
        lua_close(p->state);
        p->state = NULL;
        return NULL;
    }
    
    /* ดึง plugin info */
    lua_getglobal(p->state, "PLUGIN_VERSION");
    if (lua_isstring(p->state, -1)) {
        strncpy(p->version, lua_tostring(p->state, -1), 15);
    }
    lua_pop(p->state, 1);
    
    p->enabled = 1;
    p->apiVersion = PLUGIN_API_VERSION;
    pluginCount++;
    
    printf("Plugin loaded: %s v%s\n", p->name, p->version);
    return p;
}

/* เรียก plugin event */
static void callPluginEvent(Plugin *p, const char *eventName, const char *arg) {
    if (!p || !p->enabled) return;
    
    lua_getglobal(p->state, "on_" + (eventName[0] == 'o' ? 0 : 0));
    lua_getglobal(p->state, eventName);
    
    if (!lua_isfunction(p->state, -1)) {
        lua_pop(p->state, 1);
        return;
    }
    
    if (arg) lua_pushstring(p->state, arg);
    int nargs = arg ? 1 : 0;
    
    if (lua_pcall(p->state, nargs, 0, 0) != LUA_OK) {
        fprintf(stderr, "Plugin '%s' event '%s' error: %s\n",
            p->name, eventName, lua_tostring(p->state, -1));
        lua_pop(p->state, 1);
    }
}

int main(void) {
    printf("=== Plugin System Demo ===\n\n");
    
    /* Plugin 1: Logger */
    Plugin *logPlugin = loadPlugin("logger-plugin",
        "PLUGIN_VERSION = '1.0.0'\n"
        "\n"
        "function on_start()\n"
        "  api.log('INFO', 'Logger plugin started')\n"
        "end\n"
        "\n"
        "function on_request(path)\n"
        "  api.log('INFO', 'Request: ' .. path)\n"
        "end\n"
        "\n"
        "function on_stop()\n"
        "  api.log('INFO', 'Logger plugin stopped')\n"
        "end\n"
    );
    
    /* Plugin 2: Statistics */
    Plugin *statsPlugin = loadPlugin("stats-plugin",
        "PLUGIN_VERSION = '2.1.0'\n"
        "local requests = 0\n"
        "\n"
        "function on_start()\n"
        "  api.log('DEBUG', 'Stats plugin initialized')\n"
        "  requests = 0\n"
        "end\n"
        "\n"
        "function on_request(path)\n"
        "  requests = requests + 1\n"
        "  if requests % 5 == 0 then\n"
        "    api.log('INFO', 'Request #' .. requests .. ': ' .. path)\n"
        "  end\n"
        "end\n"
        "\n"
        "function on_stop()\n"
        "  api.log('INFO', 'Total requests: ' .. requests)\n"
        "end\n"
    );
    
    /* Fire events */
    printf("\n--- start event ---\n");
    callPluginEvent(logPlugin, "on_start", NULL);
    callPluginEvent(statsPlugin, "on_start", NULL);
    
    printf("\n--- request events ---\n");
    const char *paths[] = {"/api/users", "/api/products", "/api/orders",
                            "/api/search", "/api/cart", "/api/checkout"};
    for (int i = 0; i < 6; i++) {
        callPluginEvent(logPlugin, "on_request", paths[i]);
        callPluginEvent(statsPlugin, "on_request", paths[i]);
    }
    
    printf("\n--- stop event ---\n");
    callPluginEvent(logPlugin, "on_stop", NULL);
    callPluginEvent(statsPlugin, "on_stop", NULL);
    
    /* Cleanup */
    for (int i = 0; i < pluginCount; i++) {
        if (plugins[i].state) {
            lua_close(plugins[i].state);
            plugins[i].state = NULL;
        }
    }
    
    return 0;
}
```

---

## 44.11 ตัวอย่างเพิ่มเติม

### ตัวอย่างที่ 14: C ที่ควบคุม Game Loop

```c
/* game_loop.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <math.h>

typedef struct {
    double x, y;
    double vx, vy;
    double health;
    int alive;
    char name[32];
} Entity;

/* Push Entity ไป Lua */
static void pushEntity(lua_State *L, const Entity *e) {
    lua_newtable(L);
    lua_pushnumber(L, e->x); lua_setfield(L, -2, "x");
    lua_pushnumber(L, e->y); lua_setfield(L, -2, "y");
    lua_pushnumber(L, e->vx); lua_setfield(L, -2, "vx");
    lua_pushnumber(L, e->vy); lua_setfield(L, -2, "vy");
    lua_pushnumber(L, e->health); lua_setfield(L, -2, "health");
    lua_pushboolean(L, e->alive); lua_setfield(L, -2, "alive");
    lua_pushstring(L, e->name); lua_setfield(L, -2, "name");
}

/* Pull Entity จาก Lua */
static void pullEntity(lua_State *L, int idx, Entity *e) {
    lua_getfield(L, idx, "x"); e->x = lua_tonumber(L, -1); lua_pop(L, 1);
    lua_getfield(L, idx, "y"); e->y = lua_tonumber(L, -1); lua_pop(L, 1);
    lua_getfield(L, idx, "vx"); e->vx = lua_tonumber(L, -1); lua_pop(L, 1);
    lua_getfield(L, idx, "vy"); e->vy = lua_tonumber(L, -1); lua_pop(L, 1);
    lua_getfield(L, idx, "health"); e->health = lua_tonumber(L, -1); lua_pop(L, 1);
    lua_getfield(L, idx, "alive"); e->alive = lua_toboolean(L, -1); lua_pop(L, 1);
}

int main(void) {
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    
    /* Define game logic ใน Lua */
    luaL_dostring(L,
        "function updateEntity(entity, dt)\n"
        "  -- Move\n"
        "  entity.x = entity.x + entity.vx * dt\n"
        "  entity.y = entity.y + entity.vy * dt\n"
        "\n"
        "  -- Bounce off walls\n"
        "  if entity.x < 0 or entity.x > 100 then\n"
        "    entity.vx = -entity.vx\n"
        "    entity.x = math.max(0, math.min(100, entity.x))\n"
        "  end\n"
        "  if entity.y < 0 or entity.y > 100 then\n"
        "    entity.vy = -entity.vy\n"
        "    entity.y = math.max(0, math.min(100, entity.y))\n"
        "  end\n"
        "\n"
        "  -- Drain health\n"
        "  entity.health = entity.health - 0.1 * dt\n"
        "  entity.alive = entity.health > 0\n"
        "\n"
        "  return entity\n"
        "end\n"
    );
    
    /* สร้าง entities ใน C */
    Entity entities[] = {
        {10, 20, 5, 3, 100, 1, "Hero"},
        {50, 50, -2, 4, 80,  1, "Enemy1"},
        {80, 10, 1, -2, 60,  1, "Enemy2"},
    };
    int numEntities = 3;
    
    /* Game loop simulation */
    double dt = 0.1;  /* 100ms per frame */
    int frame = 0;
    int maxFrames = 20;
    
    printf("=== Game Loop ===\n");
    
    while (frame < maxFrames) {
        int aliveCount = 0;
        
        for (int i = 0; i < numEntities; i++) {
            if (!entities[i].alive) continue;
            
            /* ส่ง entity ไป Lua สำหรับ update */
            lua_getglobal(L, "updateEntity");
            pushEntity(L, &entities[i]);
            lua_pushnumber(L, dt);
            
            if (lua_pcall(L, 2, 1, 0) == LUA_OK) {
                pullEntity(L, -1, &entities[i]);
                lua_pop(L, 1);
            } else {
                fprintf(stderr, "Update error: %s\n", lua_tostring(L, -1));
                lua_pop(L, 1);
            }
            
            if (entities[i].alive) aliveCount++;
        }
        
        /* Print status every 5 frames */
        if (frame % 5 == 0) {
            printf("Frame %2d: alive=%d\n", frame, aliveCount);
            for (int i = 0; i < numEntities; i++) {
                if (entities[i].alive) {
                    printf("  %s: pos=(%.1f,%.1f) hp=%.1f\n",
                        entities[i].name,
                        entities[i].x, entities[i].y,
                        entities[i].health);
                }
            }
        }
        
        if (aliveCount == 0) {
            printf("All entities dead at frame %d\n", frame);
            break;
        }
        
        frame++;
    }
    
    lua_close(L);
    return 0;
}
```

### ตัวอย่างที่ 15: Event System

```c
/* event_system.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <string.h>

/* C-side event system ที่ Lua สามารถ subscribe ได้ */

/* Event registry key */
#define EVENT_REGISTRY_KEY "EventSystem"

/* Register event handler ใน Lua */
static int lua_on(lua_State *L) {
    const char *eventName = luaL_checkstring(L, 1);
    luaL_checktype(L, 2, LUA_TFUNCTION);
    
    /* ดึง event table จาก registry */
    lua_getfield(L, LUA_REGISTRYINDEX, EVENT_REGISTRY_KEY);
    if (!lua_istable(L, -1)) {
        lua_pop(L, 1);
        lua_newtable(L);
        lua_pushvalue(L, -1);
        lua_setfield(L, LUA_REGISTRYINDEX, EVENT_REGISTRY_KEY);
    }
    
    /* events[eventName] = events[eventName] or {} */
    lua_getfield(L, -1, eventName);
    if (!lua_istable(L, -1)) {
        lua_pop(L, 1);
        lua_newtable(L);
        lua_pushvalue(L, -1);
        lua_setfield(L, -3, eventName);
    }
    
    /* handlers[#handlers+1] = function */
    lua_pushvalue(L, 2);  /* function */
    lua_seti(L, -2, (lua_Integer)luaL_len(L, -2) + 1);
    
    lua_pop(L, 2);  /* pop handlers table + events table */
    return 0;
}

/* Emit event จาก C */
static void emitEvent(lua_State *L, const char *eventName, int nargs) {
    /* ดึง events จาก registry */
    lua_getfield(L, LUA_REGISTRYINDEX, EVENT_REGISTRY_KEY);
    if (!lua_istable(L, -1)) {
        lua_pop(L, 1);
        /* pop original args */
        lua_pop(L, nargs);
        return;
    }
    
    /* ดึง handlers */
    lua_getfield(L, -1, eventName);
    lua_remove(L, -2);  /* remove events table */
    
    if (!lua_istable(L, -1)) {
        lua_pop(L, 1);
        lua_pop(L, nargs);
        return;
    }
    
    int handlersIdx = lua_gettop(L);
    int n = (int)luaL_len(L, handlersIdx);
    
    /* เรียก handlers แต่ละตัว */
    for (int i = 1; i <= n; i++) {
        lua_geti(L, handlersIdx, i);
        
        /* Push arguments (copies) */
        for (int j = -(nargs + 1); j < -1; j++) {
            lua_pushvalue(L, j - 1);
        }
        
        if (lua_pcall(L, nargs, 0, 0) != LUA_OK) {
            fprintf(stderr, "Event handler error: %s\n", lua_tostring(L, -1));
            lua_pop(L, 1);
        }
    }
    
    lua_pop(L, 1);  /* pop handlers table */
    lua_pop(L, nargs);  /* pop original args */
}

static const luaL_Reg eventsLib[] = {
    {"on", lua_on},
    {NULL, NULL}
};

int main(void) {
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    
    /* เพิ่ม event API */
    luaL_newlib(L, eventsLib);
    lua_setglobal(L, "events");
    
    /* Lua code subscribe events */
    luaL_dostring(L,
        "-- Subscribe to events\n"
        "events.on('user.login', function(userId, username)\n"
        "  print('[Lua] User logged in:', userId, username)\n"
        "end)\n"
        "\n"
        "events.on('user.login', function(userId, username)\n"
        "  print('[Lua] Welcome,', username .. '!')\n"
        "end)\n"
        "\n"
        "events.on('data.saved', function(table, count)\n"
        "  print('[Lua] Saved', count, 'records to', table)\n"
        "end)\n"
    );
    
    /* Emit events จาก C */
    printf("=== Event System ===\n\n");
    
    printf("Emitting user.login...\n");
    lua_pushinteger(L, 42);
    lua_pushstring(L, "Alice");
    emitEvent(L, "user.login", 2);
    
    printf("\nEmitting data.saved...\n");
    lua_pushstring(L, "users");
    lua_pushinteger(L, 150);
    emitEvent(L, "data.saved", 2);
    
    printf("\nEmitting unknown event...\n");
    lua_pushstring(L, "test data");
    emitEvent(L, "unknown.event", 1);
    printf("(no handlers - silently ignored)\n");
    
    lua_close(L);
    return 0;
}
```

### ตัวอย่างที่ 16: Lua State Pool

```c
/* state_pool.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <time.h>

/* State Pool สำหรับ reuse Lua states */
#define POOL_SIZE 8

typedef struct {
    lua_State *state;
    int inUse;
    time_t lastUsed;
} StatePoolEntry;

typedef struct {
    StatePoolEntry entries[POOL_SIZE];
    const char *initCode;  /* code ที่รันใน state ใหม่ */
} StatePool;

static StatePool* createPool(const char *initCode) {
    StatePool *pool = (StatePool*)calloc(1, sizeof(StatePool));
    pool->initCode = initCode;
    return pool;
}

static lua_State* acquireState(StatePool *pool) {
    /* หา state ที่ว่าง */
    for (int i = 0; i < POOL_SIZE; i++) {
        if (pool->entries[i].state && !pool->entries[i].inUse) {
            pool->entries[i].inUse = 1;
            pool->entries[i].lastUsed = time(NULL);
            printf("[Pool] Acquired state #%d (reused)\n", i);
            return pool->entries[i].state;
        }
    }
    
    /* หา slot ว่างสำหรับสร้างใหม่ */
    for (int i = 0; i < POOL_SIZE; i++) {
        if (!pool->entries[i].state) {
            lua_State *L = luaL_newstate();
            luaL_openlibs(L);
            if (pool->initCode) {
                luaL_dostring(L, pool->initCode);
            }
            pool->entries[i].state = L;
            pool->entries[i].inUse = 1;
            pool->entries[i].lastUsed = time(NULL);
            printf("[Pool] Acquired state #%d (new)\n", i);
            return L;
        }
    }
    
    fprintf(stderr, "[Pool] Pool exhausted!\n");
    return NULL;
}

static void releaseState(StatePool *pool, lua_State *L) {
    for (int i = 0; i < POOL_SIZE; i++) {
        if (pool->entries[i].state == L) {
            /* Reset state สำหรับ reuse */
            lua_settop(L, 0);  /* clear stack */
            
            /* Re-run init code */
            if (pool->initCode) {
                luaL_dostring(L, pool->initCode);
            }
            
            pool->entries[i].inUse = 0;
            printf("[Pool] Released state #%d\n", i);
            return;
        }
    }
}

static void destroyPool(StatePool *pool) {
    for (int i = 0; i < POOL_SIZE; i++) {
        if (pool->entries[i].state) {
            lua_close(pool->entries[i].state);
            pool->entries[i].state = NULL;
        }
    }
    free(pool);
}

/* Simulation: Process requests using pooled states */
static void processRequest(StatePool *pool, int requestId, const char *code) {
    lua_State *L = acquireState(pool);
    if (!L) {
        fprintf(stderr, "No available state for request %d\n", requestId);
        return;
    }
    
    /* รัน request code */
    lua_pushinteger(L, requestId);
    lua_setglobal(L, "REQUEST_ID");
    
    if (luaL_dostring(L, code) != LUA_OK) {
        fprintf(stderr, "Request %d error: %s\n", requestId, lua_tostring(L, -1));
        lua_pop(L, 1);
    }
    
    releaseState(pool, L);
}

int main(void) {
    StatePool *pool = createPool(
        "-- Init code: run in every new state\n"
        "function processData(data)\n"
        "  local result = 0\n"
        "  for _, v in ipairs(data) do result = result + v end\n"
        "  return result\n"
        "end\n"
    );
    
    printf("=== State Pool Demo ===\n\n");
    
    /* Process multiple requests */
    for (int i = 1; i <= 5; i++) {
        char code[256];
        snprintf(code, 256,
            "local sum = processData({%d, %d, %d})\n"
            "print('Request ' .. REQUEST_ID .. ': sum =', sum)",
            i*10, i*20, i*30);
        processRequest(pool, i, code);
        printf("\n");
    }
    
    destroyPool(pool);
    printf("Pool destroyed\n");
    return 0;
}
```

### ตัวอย่างที่ 17: Lua as Template Engine

```lua
-- template_simulation.lua
-- จำลอง C application ที่ใช้ Lua เป็น template engine

-- Template engine ที่รัน Lua code ใน templates
local function renderTemplate(template, context)
    -- แปลง template ที่มี <%= expr %> และ <% code %> เป็น Lua code
    local parts = {}
    local pos = 1
    
    while pos <= #template do
        local exprStart = template:find("<%=", pos, true)
        local codeStart = template:find("<%", pos, true)
        
        -- หา tag ที่ใกล้ที่สุด
        local nextTag
        local tagType
        
        if exprStart and (not codeStart or exprStart < codeStart) then
            nextTag = exprStart
            tagType = "expr"
        elseif codeStart then
            nextTag = codeStart
            tagType = "code"
        end
        
        if not nextTag then
            -- ส่วนที่เหลือเป็น literal text
            table.insert(parts, string.format('__out[#__out+1] = %q', template:sub(pos)))
            break
        end
        
        -- Literal text ก่อน tag
        if nextTag > pos then
            table.insert(parts, string.format('__out[#__out+1] = %q', template:sub(pos, nextTag - 1)))
        end
        
        local tagEnd = template:find("%>", nextTag, true)
        if not tagEnd then
            error("Unclosed template tag at position " .. nextTag)
        end
        
        if tagType == "expr" then
            local expr = template:sub(nextTag + 3, tagEnd - 1):match("^%s*(.-)%s*$")
            table.insert(parts, string.format('__out[#__out+1] = tostring(%s)', expr))
        else
            local code = template:sub(nextTag + 2, tagEnd - 1)
            table.insert(parts, code)
        end
        
        pos = tagEnd + 2
    end
    
    -- Build Lua function
    local code = "local __out = {}\n" .. table.concat(parts, "\n") .. "\nreturn table.concat(__out)"
    
    -- สร้าง environment จาก context
    local env = setmetatable(context or {}, {__index = _G})
    
    local fn, err = load(code, "template", "t", env)
    if not fn then
        return nil, "Template compile error: " .. err
    end
    
    local ok, result = pcall(fn)
    if not ok then
        return nil, "Template render error: " .. result
    end
    
    return result
end

-- ทดสอบ
local template = [[
<!DOCTYPE html>
<html>
<head><title><%= title %></title></head>
<body>
<h1>Hello, <%= name %>!</h1>
<p>Items:</p>
<ul>
<% for _, item in ipairs(items) do %>
  <li><%= item.name %> - $<%= string.format("%.2f", item.price) %></li>
<% end %>
</ul>
<p>Total: $<%= string.format("%.2f", total) %></p>
</body>
</html>
]]

local context = {
    title = "Shopping Cart",
    name = "Alice",
    items = {
        {name = "Apple", price = 1.99},
        {name = "Banana", price = 0.99},
        {name = "Cherry", price = 3.49},
    },
    total = 1.99 + 0.99 + 3.49
}

local result, err = renderTemplate(template, context)
if err then
    print("Error:", err)
else
    print(result)
end
```

### ตัวอย่างที่ 18: Debug Interface

```c
/* debug_interface.c */
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdio.h>
#include <string.h>

/* Debug hook callback */
static void debugHook(lua_State *L, lua_Debug *ar) {
    lua_getinfo(L, "nSlf", ar);
    
    const char *source = ar->source ? ar->source : "?";
    const char *funcName = ar->name ? ar->name : "?";
    int line = ar->currentline;
    
    /* ตัดชื่อ source ถ้ายาวเกิน */
    if (source[0] == '@') source++;
    if (strlen(source) > 30) source = source + strlen(source) - 30;
    
    switch (ar->event) {
        case LUA_HOOKCALL:
            printf("[CALL]   %s:%d -> %s\n", source, line, funcName);
            break;
        case LUA_HOOKRET:
            printf("[RETURN] %s:%d <- %s\n", source, line, funcName);
            break;
        case LUA_HOOKLINE:
            printf("[LINE]   %s:%d\n", source, line);
            break;
    }
}

/* Enable/disable debug tracing */
static int lua_setTrace(lua_State *L) {
    int enable = lua_toboolean(L, 1);
    
    if (enable) {
        lua_sethook(L, debugHook,
            LUA_MASKCALL | LUA_MASKRET | LUA_MASKLINE, 0);
        printf("[Debug] Tracing enabled\n");
    } else {
        lua_sethook(L, NULL, 0, 0);
        printf("[Debug] Tracing disabled\n");
    }
    return 0;
}

/* Stack dump */
static int lua_dumpStack(lua_State *L) {
    int n = lua_gettop(L);
    printf("[Stack Dump] %d items:\n", n - 1);  /* -1 for this function */
    for (int i = 1; i < n; i++) {
        int t = lua_type(L, i);
        printf("  [%d] %s: ", i, lua_typename(L, t));
        switch (t) {
            case LUA_TNIL: printf("nil"); break;
            case LUA_TBOOLEAN: printf("%s", lua_toboolean(L, i) ? "true" : "false"); break;
            case LUA_TNUMBER: printf("%g", lua_tonumber(L, i)); break;
            case LUA_TSTRING: printf("%q", lua_tostring(L, i)); break;
            default: printf("<%s>", lua_typename(L, t)); break;
        }
        printf("\n");
    }
    return 0;
}

/* Inspect global variables */
static int lua_dumpGlobals(lua_State *L) {
    const char *filter = lua_tostring(L, 1);  /* optional filter */
    
    lua_pushglobaltable(L);
    lua_pushnil(L);
    
    int count = 0;
    printf("[Globals]:\n");
    
    while (lua_next(L, -2) != 0) {
        const char *key = lua_tostring(L, -2);
        if (key && (!filter || strstr(key, filter))) {
            int t = lua_type(L, -1);
            printf("  %s = (%s)", key, lua_typename(L, t));
            if (t == LUA_TNUMBER) printf(" %g", lua_tonumber(L, -1));
            else if (t == LUA_TSTRING) printf(" %q", lua_tostring(L, -1));
            else if (t == LUA_TBOOLEAN) printf(" %s", lua_toboolean(L, -1) ? "true" : "false");
            printf("\n");
            count++;
        }
        lua_pop(L, 1);
    }
    lua_pop(L, 1);
    
    printf("Total: %d globals\n", count);
    return 0;
}

static const luaL_Reg debugLib[] = {
    {"setTrace",   lua_setTrace},
    {"dumpStack",  lua_dumpStack},
    {"dumpGlobals", lua_dumpGlobals},
    {NULL, NULL}
};

int main(void) {
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    
    luaL_newlib(L, debugLib);
    lua_setglobal(L, "dbg");
    
    printf("=== Debug Interface Demo ===\n\n");
    
    /* Dump globals */
    luaL_dostring(L,
        "myVar = 42\n"
        "myStr = 'hello'\n"
        "myBool = true\n"
        "dbg.dumpGlobals('my')\n"
    );
    
    printf("\n--- Tracing ---\n");
    luaL_dostring(L,
        "dbg.setTrace(true)\n"
        "local function greet(name)\n"
        "  return 'Hello, ' .. name\n"
        "end\n"
        "local result = greet('World')\n"
        "dbg.setTrace(false)\n"
        "print(result)\n"
    );
    
    lua_close(L);
    return 0;
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Embedding Lua in C Applications:

1. **ทำไมต้อง Embed Lua** - game scripting, plugins, config, sandboxing
2. **State Lifecycle** - สร้าง, ใช้งาน, และทำลาย lua_State
3. **Running Scripts** - dostring, dofile, loadbuffer
4. **Calling Lua from C** - เรียก functions, ดึง return values
5. **Data Exchange** - แปลง C structs เป็น Lua tables และกลับ
6. **Config Language** - ใช้ Lua สำหรับ dynamic config
7. **Sandboxed Execution** - จำกัด environment ของ scripts
8. **Instruction Limits** - ป้องกัน infinite loops
9. **Coroutines from C** - ใช้ lua_resume สำหรับ generators
10. **Custom Allocators** - ควบคุม memory allocation
11. **Plugin System** - สร้าง plugin architecture
12. **State Pool** - reuse Lua states สำหรับ performance
13. **Event System** - subscribe/emit events ระหว่าง C และ Lua
14. **Debug Interface** - tools สำหรับ debugging embedded Lua

Embedding Lua ช่วยให้ application มีความยืดหยุ่นสูง สามารถ customize พฤติกรรมได้โดยไม่ต้องแก้ไขและ compile code ใหม่
