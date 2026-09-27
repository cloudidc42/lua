# บทที่ 96: Contributing to Lua - Open Source

## บทนำ

การมีส่วนร่วมกับโปรเจกต์ open source คือหนึ่งในทักษะที่ดีที่สุดที่นักพัฒนาสามารถพัฒนาได้ บทนี้จะแนะนำวิธีการมีส่วนร่วมกับ Lua และ ecosystem ของมัน

---

## 96.1 Lua Development Process

Lua ถูกพัฒนาโดยทีมเล็กๆ ที่ PUC-Rio, Brazil:
- **Roberto Ierusalimschy** - ผู้ออกแบบหลัก
- **Waldemar Celes** - co-designer
- **Luiz Henrique de Figueiredo** - co-designer

```
Lua Development Model:
- Email-based development (lua-l@lua.org mailing list)
- No public issue tracker (unlike GitHub projects)
- Changes discussed on mailing list first
- Small, conservative team
- "Lua is not for everyone" philosophy - intentional minimalism
```

---

## 96.2 Building Lua จาก Source

```bash
# Clone the Lua source
git clone https://github.com/lua/lua.git
cd lua

# Build (Linux/macOS)
make linux-readline
# หรือ
make macosx

# Build (Windows - MinGW)
make mingw

# Install
sudo make install

# Run tests
cd test
lua all.lua
```

---

## 96.3 Lua Source Structure

```
lua/
├── lapi.c      - The Public API (lua_*)
├── lauxlib.c   - Auxiliary library (luaL_*)
├── lbaselib.c  - Basic functions (print, pairs, etc.)
├── lcode.c     - Code generator
├── lcorolib.c  - Coroutine library
├── lctype.c    - Character classes
├── ldblib.c    - Debug library
├── ldebug.c    - Debug interface
├── ldo.c       - Stack and Call Management
├── ldump.c     - Binary serializer
├── lfunc.c     - Function support
├── lgc.c       - Garbage collector
├── linit.c     - Library initialization
├── liolib.c    - I/O library
├── llex.c      - Lexer
├── lmathlib.c  - Math library
├── lmem.c      - Memory management
├── loadlib.c   - Package loading
├── lobject.c   - Object operations
├── lopcodes.c  - Opcode definitions
├── loslib.c    - OS library
├── lparser.c   - Parser
├── lstate.c    - Global state
├── lstring.c   - String table
├── lstrlib.c   - String library
├── ltable.c    - Table operations
├── ltablib.c   - Table library
├── ltm.c       - Tag methods (metamethods)
├── lundump.c   - Binary deserializer
├── lutf8lib.c  - UTF-8 library
├── lvm.c       - Virtual machine
├── lzio.c      - Buffered I/O
├── lua.c       - Lua standalone interpreter
├── luac.c      - Lua compiler
└── lua.h       - Main header
```

---

## 96.4 Reading Lua Source Code

```c
/* lstate.h - Global State structure */
typedef struct global_State {
    lua_Alloc frealloc;    /* function to reallocate memory */
    void *ud;              /* auxiliary data to 'frealloc' */
    l_mem totalbytes;      /* number of bytes currently allocated */
    l_mem GCdebt;          /* bytes allocated not yet compensated by GC */
    lu_mem GCestimate;     /* an estimate of the non-garbage memory in use */
    lu_mem lastatomic;     /* see function 'genstep' in lgc.c */
    stringtable strt;      /* hash table for strings */
    TValue l_registry;     /* registry */
    TValue nilvalue;       /* a nil value */
    unsigned int seed;     /* randomized seed for hashes */
    lu_byte currentwhite;
    lu_byte gcstate;       /* state of garbage collector */
    lu_byte gckind;        /* kind of GC running */
    ...
} global_State;
```

```lua
-- Lua-level exploration of VM
-- ดู bytecode ของ function

local function example(x, y)
    local result = x + y
    return result * 2
end

-- Dump bytecode (requires luac)
-- luac -l -l -p example.lua

-- Alternative: use debug library
local info = debug.getinfo(example)
print("Function info:")
print("  Source:", info.source)
print("  Lines:", info.linedefined, "-", info.lastlinedefined)
print("  Params:", info.nparams)
print("  Is vararg:", info.isvararg)
print("  Upvalues:", info.nups)
```

---

## 96.5 Writing Your First Bug Report

```
Good bug report template for lua-l:

Subject: [BUG] Unexpected behavior with string.format and %q

Environment:
- Lua 5.4.6
- Linux x86_64 (or Windows, macOS)

Steps to reproduce:
```
local s = string.format("%q", "\0")
print(#s)  -- Expected 4 ("\0"), got 2
```

Expected behavior:
%q should produce "\0" (4 chars) for null bytes

Actual behavior:
Returns "" (empty string, 2 chars)

Additional notes:
This worked correctly in Lua 5.3

---

What NOT to do:
- Don't post "I found a bug" without code
- Don't ask "is this a bug?" - show the code
- Don't expect immediate response - be patient
- Don't post to lua-l AND github simultaneously
```

---

## 96.6 Writing a Lua Library

```lua
-- Example: Writing a well-structured Lua library

--[[
mylib.lua - A demonstration library

API:
  mylib.greet(name) - Returns greeting
  mylib.version     - Library version
  mylib.Config:new(opts) - Config object

INSTALL:
  luarocks install mylib (when published)

LICENSE: MIT
]]

local M = {}

-- Version
M._VERSION = "1.0.0"
M._NAME = "mylib"

-- Private state
local _config = {
    debug = false,
    prefix = "[mylib]"
}

-- Private helper (not exported)
local function _log(msg)
    if _config.debug then
        io.stderr:write(_config.prefix .. " " .. msg .. "\n")
    end
end

-- Public API
function M.greet(name)
    assert(type(name) == "string", "name must be a string")
    _log("greet called with: " .. name)
    return "Hello, " .. name .. "!"
end

function M.configure(opts)
    if type(opts) ~= "table" then return end
    for k, v in pairs(opts) do
        if _config[k] ~= nil then
            _config[k] = v
        end
    end
end

-- Class within library
M.Config = {}
M.Config.__index = M.Config

function M.Config.new(opts)
    local obj = setmetatable({}, M.Config)
    obj._data = {}
    if opts then
        for k, v in pairs(opts) do
            obj._data[k] = v
        end
    end
    return obj
end

function M.Config:get(key, default)
    local val = self._data[key]
    return val ~= nil and val or default
end

function M.Config:set(key, value)
    self._data[key] = value
    return self
end

function M.Config:__tostring()
    local parts = {}
    for k, v in pairs(self._data) do
        table.insert(parts, tostring(k) .. "=" .. tostring(v))
    end
    return "Config{" .. table.concat(parts, ", ") .. "}"
end

return M
```

---

## 96.7 Creating a Rockspec

```lua
-- mylib-1.0.0-1.rockspec

package = "mylib"
version = "1.0.0-1"

source = {
    url = "git+https://github.com/yourname/mylib.git",
    tag = "v1.0.0"
}

description = {
    summary = "A demonstration Lua library",
    detailed = [[
        mylib provides useful utilities for Lua developers.
        See documentation at https://github.com/yourname/mylib
    ]],
    homepage = "https://github.com/yourname/mylib",
    license = "MIT"
}

dependencies = {
    "lua >= 5.1",
    -- optional dependencies:
    -- "luasocket >= 3.0",
    -- "dkjson >= 2.5"
}

build = {
    type = "builtin",
    modules = {
        ["mylib"] = "src/mylib.lua",
        ["mylib.config"] = "src/mylib/config.lua",
        ["mylib.utils"] = "src/mylib/utils.lua"
    }
}
```

```bash
# Test rockspec locally
luarocks make mylib-1.0.0-1.rockspec

# Pack into rock
luarocks pack mylib-1.0.0-1.rockspec

# Upload to LuaRocks
luarocks upload mylib-1.0.0-1.rockspec --api-key=YOUR_KEY
```

---

## 96.8 Contributing to LuaRocks

```bash
# Fork LuaRocks on GitHub
git clone https://github.com/YOUR_FORK/luarocks.git
cd luarocks

# Create a feature branch
git checkout -b fix/my-bug-fix

# Make changes
# ... edit files ...

# Run tests
cd test
lua run_tests.lua

# Submit PR with clear description:
# - What problem does this fix?
# - How was it tested?
# - Is this a breaking change?
```

---

## 96.9 Open Source Best Practices

```
1. Code Style
   - Follow existing conventions in the project
   - Use consistent naming
   - Add tests for new features
   - Don't break existing tests

2. Documentation  
   - Update README.md if behavior changes
   - Add LuaDoc comments to public APIs
   - Include example code

3. Version Control
   - Small, atomic commits
   - Clear commit messages
   - Reference issues when applicable

4. Communication
   - Be respectful and patient
   - Accept feedback gracefully
   - Ask questions before large PRs
   - Be specific about what you're changing

5. Testing
   - Write tests BEFORE or WITH the code
   - Cover edge cases
   - Don't just test the happy path
```

---

## 96.10 Notable Lua Projects to Contribute To

```lua
-- Projects accepting contributions:

local projects = {
    {
        name = "LuaRocks",
        url = "https://github.com/luarocks/luarocks",
        desc = "Package manager for Lua"
    },
    {
        name = "lua-language-server",
        url = "https://github.com/LuaLS/lua-language-server",
        desc = "LSP server for Lua"
    },
    {
        name = "Lapis",
        url = "https://github.com/leafo/lapis",
        desc = "Web framework for OpenResty"
    },
    {
        name = "Busted",
        url = "https://github.com/lunarmodules/busted",
        desc = "Testing framework"
    },
    {
        name = "Penlight",
        url = "https://github.com/lunarmodules/Penlight",
        desc = "Utility library"
    },
    {
        name = "Teal",
        url = "https://github.com/teal-language/tl",
        desc = "Typed Lua language"
    },
    {
        name = "LÖVE",
        url = "https://github.com/love2d/love",
        desc = "2D game framework"
    },
    {
        name = "Neovim plugins",
        url = "https://github.com/rockerBOO/awesome-neovim",
        desc = "Many plugins are written in Lua"
    }
}

for _, p in ipairs(projects) do
    print(string.format("• %-30s - %s", p.name, p.desc))
    print(string.format("  %s", p.url))
end
```

---

## แบบฝึกหัด

1. **Bug Report**: เขียน bug report ที่ดีสำหรับ issue ที่คุณพบ
2. **First Contribution**: ส่ง PR แก้ไข documentation หรือ typo
3. **Library**: สร้าง Lua library เล็กๆ และ publish บน LuaRocks
4. **Test**: เพิ่ม test cases ให้กับ library ที่มีอยู่
5. **Translation**: แปล documentation เป็นภาษาไทย

---

*ต่อไป: [Part 97 - Real Project: Chat Application](part-97.md)*
