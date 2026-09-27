# บทที่ 93: Language Server Protocol (LSP) กับ Lua

## บทนำ

LSP (Language Server Protocol) คือ protocol มาตรฐานสำหรับการสื่อสารระหว่าง editor และ language server ทำให้ IDE features เช่น autocomplete, go-to-definition, diagnostics ทำงานได้กับทุก editor

---

## 93.1 LSP Overview

```
Editor (Client)          Language Server
     |                         |
     |-- initialize ---------->|
     |<-- initialized ---------|
     |                         |
     |-- textDocument/         |
     |   didOpen ------------> |
     |                         |
     |-- textDocument/         |
     |   completion ---------->|
     |<-- CompletionList -------|
     |                         |
     |-- textDocument/         |  
     |   hover -------------> |
     |<-- Hover response -------|
     |                         |
     |-- shutdown ------------>|
     |<-- (null) --------------|
     |-- exit ---------------->|
```

---

## 93.2 lua-language-server

lua-language-server (LuaLS) คือ official LSP สำหรับ Lua:

```bash
# ติดตั้งผ่าน Mason (Neovim)
# :MasonInstall lua-language-server

# ติดตั้งด้วย Homebrew (macOS)
brew install lua-language-server

# ติดตั้งจาก release
# Download จาก https://github.com/LuaLS/lua-language-server/releases
```

---

## 93.3 Configuring lua-language-server

```lua
-- .luarc.json (JSON config for LuaLS)
--[[
{
    "runtime": {
        "version": "Lua 5.4",
        "pathStrict": true,
        "path": [
            "?.lua",
            "?/init.lua",
            "lib/?.lua"
        ]
    },
    "diagnostics": {
        "enable": true,
        "globals": ["vim", "love", "ngx"],
        "neededFileStatus": {
            "codestyle-check": "Any"
        },
        "groupSeverity": {
            "strong": "Warning",
            "strict": "Hint"
        },
        "groupFileStatus": {
            "ambiguity": "Opened",
            "await": "Opened"
        },
        "unusedLocalExclude": ["_*"]
    },
    "workspace": {
        "library": [
            "${3rd}/love2d/library",
            "${3rd}/OpenResty/library"
        ],
        "checkThirdParty": false,
        "ignoreDir": [
            ".git",
            ".vscode",
            "node_modules"
        ]
    },
    "completion": {
        "enable": true,
        "autoRequire": true,
        "callSnippet": "Replace",
        "keywordSnippet": "Replace",
        "displayContext": 6
    },
    "hover": {
        "enable": true,
        "enumsLimit": 5
    },
    "hint": {
        "enable": true,
        "setType": false,
        "paramType": true,
        "paramName": "Disable",
        "semicolon": "SameLine",
        "arrayIndex": "Disable"
    },
    "format": {
        "enable": true,
        "defaultConfig": {
            "indent_style": "space",
            "indent_size": "4",
            "continuation_indent_size": "4",
            "max_line_length": "100",
            "trailing_table_separator": "smart",
            "call_arg_parentheses": "keep"
        }
    }
}
]]
```

---

## 93.4 LSP สำหรับ Neovim

```lua
-- ~/.config/nvim/lua/lsp/lua.lua
-- Neovim LSP config for Lua development

local lspconfig = require('lspconfig')

-- Setup lua-language-server
lspconfig.lua_ls.setup({
    settings = {
        Lua = {
            runtime = {
                version = 'Lua 5.4',
                path = {
                    '?.lua',
                    '?/init.lua',
                    vim.fn.expand('~/.luarocks/share/lua/5.4/?.lua'),
                    vim.fn.expand('~/.luarocks/share/lua/5.4/?/init.lua'),
                }
            },
            diagnostics = {
                globals = { 'vim', 'love', 'ngx' }
            },
            workspace = {
                library = vim.api.nvim_get_runtime_file("", true),
                checkThirdParty = false
            },
            telemetry = {
                enable = false
            },
            completion = {
                callSnippet = "Replace"
            }
        }
    },
    on_attach = function(client, bufnr)
        -- เปิด format on save
        if client.supports_method("textDocument/formatting") then
            vim.api.nvim_create_autocmd("BufWritePre", {
                buffer = bufnr,
                callback = function()
                    vim.lsp.buf.format({ async = false })
                end
            })
        end
    end
})

-- Keymaps
vim.keymap.set('n', 'gd', vim.lsp.buf.definition)
vim.keymap.set('n', 'gr', vim.lsp.buf.references)
vim.keymap.set('n', 'K', vim.lsp.buf.hover)
vim.keymap.set('n', '<leader>rn', vim.lsp.buf.rename)
vim.keymap.set('n', '<leader>ca', vim.lsp.buf.code_action)
vim.keymap.set('n', '[d', vim.diagnostic.goto_prev)
vim.keymap.set('n', ']d', vim.diagnostic.goto_next)
```

---

## 93.5 สร้าง Simple LSP Server ใน Lua

```lua
-- simple_lsp.lua
-- A minimal LSP server implementation

local json = require("dkjson")  -- หรือ cjson

local LSPServer = {}
LSPServer.__index = LSPServer

function LSPServer.new()
    return setmetatable({
        handlers = {},
        initialized = false,
        documents = {},  -- uri -> content
        next_id = 1
    }, LSPServer)
end

-- JSON-RPC message reading
function LSPServer:read_message()
    local header = ""
    local content_length = 0
    
    -- Read headers
    while true do
        local line = io.read("l")
        if line == nil or line == "" then break end
        
        local length = line:match("Content%-Length: (%d+)")
        if length then
            content_length = tonumber(length)
        end
    end
    
    if content_length == 0 then return nil end
    
    -- Read body
    local body = io.read(content_length)
    return json.decode(body)
end

-- JSON-RPC message writing
function LSPServer:write_message(msg)
    local body = json.encode(msg)
    local header = string.format("Content-Length: %d\r\n\r\n", #body)
    io.write(header .. body)
    io.flush()
end

-- Send response
function LSPServer:respond(id, result, error_info)
    local msg = {
        jsonrpc = "2.0",
        id = id
    }
    if error_info then
        msg.error = error_info
    else
        msg.result = result
    end
    self:write_message(msg)
end

-- Send notification
function LSPServer:notify(method, params)
    self:write_message({
        jsonrpc = "2.0",
        method = method,
        params = params
    })
end

-- Handler registration
function LSPServer:on(method, handler)
    self.handlers[method] = handler
end

-- Handle initialize
function LSPServer:handle_initialize(id, params)
    self.initialized = true
    self:respond(id, {
        capabilities = {
            textDocumentSync = {
                openClose = true,
                change = 1,  -- Full sync
            },
            completionProvider = {
                triggerCharacters = { ".", ":" },
                resolveProvider = false
            },
            hoverProvider = true,
            definitionProvider = true,
            referencesProvider = true,
            renameProvider = true,
            documentFormattingProvider = true,
            diagnosticProvider = {
                interFileDependencies = false,
                workspaceDiagnostics = false
            }
        },
        serverInfo = {
            name = "simple-lua-lsp",
            version = "0.1.0"
        }
    })
end

-- Handle hover
function LSPServer:handle_hover(id, params)
    local uri = params.textDocument.uri
    local line = params.position.line
    local char = params.position.character
    
    local doc = self.documents[uri]
    if not doc then
        self:respond(id, nil)
        return
    end
    
    -- Simple hover: show word under cursor
    local lines = {}
    for l in doc:gmatch("[^\n]+") do
        table.insert(lines, l)
    end
    
    local current_line = lines[line + 1] or ""
    
    -- Find word at character position
    local word_start = char
    while word_start > 0 and current_line:sub(word_start, word_start):match("[%w_]") do
        word_start = word_start - 1
    end
    local word = current_line:sub(word_start + 1):match("[%w_]+")
    
    if word then
        -- Look up documentation
        local docs = {
            print = "Prints values to stdout. Multiple values are separated by tabs.",
            pairs = "Returns an iterator function, the table, and nil.",
            ipairs = "Returns an iterator function for arrays (integer keys).",
            type = "Returns the type of the value as a string.",
            tostring = "Converts the value to a string.",
            tonumber = "Converts the value to a number.",
            error = "Raises an error with the given message.",
            pcall = "Calls a function in protected mode.",
            require = "Loads the given module.",
            setmetatable = "Sets the metatable for a table.",
            getmetatable = "Returns the metatable of a table."
        }
        
        local doc_text = docs[word] or ("Symbol: `" .. word .. "`")
        
        self:respond(id, {
            contents = {
                kind = "markdown",
                value = "```lua\n" .. word .. "\n```\n\n" .. doc_text
            }
        })
    else
        self:respond(id, nil)
    end
end

-- Handle completion
function LSPServer:handle_completion(id, params)
    local uri = params.textDocument.uri
    local line = params.position.line
    local char = params.position.character
    
    -- Built-in completions
    local completions = {}
    
    local builtins = {
        "print", "pairs", "ipairs", "type", "tostring", "tonumber",
        "error", "pcall", "xpcall", "assert", "require",
        "setmetatable", "getmetatable", "rawget", "rawset",
        "select", "unpack", "table", "string", "math", "io", "os",
        "coroutine", "debug", "collectgarbage", "load", "dofile"
    }
    
    for _, name in ipairs(builtins) do
        table.insert(completions, {
            label = name,
            kind = 3,  -- Function
            detail = "Built-in function",
        })
    end
    
    -- String library completions
    local string_methods = {
        "format", "len", "sub", "upper", "lower", "rep", "reverse",
        "find", "match", "gmatch", "gsub", "byte", "char"
    }
    for _, m in ipairs(string_methods) do
        table.insert(completions, {
            label = "string." .. m,
            kind = 3,
            detail = "string library"
        })
    end
    
    self:respond(id, {
        isIncomplete = false,
        items = completions
    })
end

-- Handle textDocument/didOpen
function LSPServer:handle_did_open(params)
    local uri = params.textDocument.uri
    local text = params.textDocument.text
    self.documents[uri] = text
    
    -- Run diagnostics
    self:run_diagnostics(uri, text)
end

-- Handle textDocument/didChange
function LSPServer:handle_did_change(params)
    local uri = params.textDocument.uri
    -- Full sync - take last change
    local changes = params.contentChanges
    if #changes > 0 then
        self.documents[uri] = changes[#changes].text
        self:run_diagnostics(uri, self.documents[uri])
    end
end

-- Simple diagnostics
function LSPServer:run_diagnostics(uri, text)
    local diagnostics = {}
    local line_num = 0
    
    for line in text:gmatch("[^\n]*") do
        -- Check for common issues
        
        -- Warn about print in production
        local col = line:find("print%(")
        if col then
            table.insert(diagnostics, {
                range = {
                    start = { line = line_num, character = col - 1 },
                    ["end"] = { line = line_num, character = col + 4 }
                },
                severity = 3,  -- Information
                code = "no-print",
                message = "Consider using a logger instead of print()",
                source = "simple-lua-lsp"
            })
        end
        
        -- Check for TODO comments
        local todo_col = line:find("TODO")
        if todo_col then
            table.insert(diagnostics, {
                range = {
                    start = { line = line_num, character = todo_col - 1 },
                    ["end"] = { line = line_num, character = todo_col + 3 }
                },
                severity = 4,  -- Hint
                message = "TODO comment found",
                source = "simple-lua-lsp"
            })
        end
        
        line_num = line_num + 1
    end
    
    -- Publish diagnostics
    self:notify("textDocument/publishDiagnostics", {
        uri = uri,
        diagnostics = diagnostics
    })
end

-- Main loop
function LSPServer:run()
    while true do
        local msg = self:read_message()
        if not msg then break end
        
        local method = msg.method
        local id = msg.id
        local params = msg.params or {}
        
        if method == "initialize" then
            self:handle_initialize(id, params)
        elseif method == "initialized" then
            -- No response needed
        elseif method == "shutdown" then
            self:respond(id, nil)
        elseif method == "exit" then
            os.exit(0)
        elseif method == "textDocument/hover" then
            self:handle_hover(id, params)
        elseif method == "textDocument/completion" then
            self:handle_completion(id, params)
        elseif method == "textDocument/didOpen" then
            self:handle_did_open(params)
        elseif method == "textDocument/didChange" then
            self:handle_did_change(params)
        elseif method == "$/cancelRequest" then
            -- Cancel not implemented
        else
            if id then
                self:respond(id, nil, {
                    code = -32601,
                    message = "Method not found: " .. tostring(method)
                })
            end
        end
    end
end

-- Run the server
-- local server = LSPServer.new()
-- server:run()
```

---

## 93.6 LSP Testing

```lua
-- test_lsp.lua - Test LSP server functionality

-- Simulate LSP client
local LSPClient = {}
LSPClient.__index = LSPClient

function LSPClient.new(server_input, server_output)
    return setmetatable({
        server_in = server_input,
        server_out = server_output,
        next_id = 1
    }, LSPClient)
end

function LSPClient:send(method, params, callback)
    local id = self.next_id
    self.next_id = self.next_id + 1
    
    local msg = {
        jsonrpc = "2.0",
        id = id,
        method = method,
        params = params
    }
    
    -- Would write to server stdin
    return id
end

-- Unit test for diagnostic runner
local function test_diagnostics()
    local server = LSPServer.new()
    local diagnostics_received = {}
    
    -- Override notify to capture
    local orig_notify = server.notify
    server.notify = function(self, method, params)
        if method == "textDocument/publishDiagnostics" then
            table.insert(diagnostics_received, params)
        end
    end
    
    -- Test Lua with print statement
    local test_code = [[
local function greet(name)
    print("Hello, " .. name)  -- TODO: use logger
    return "Hello, " .. name
end
]]
    
    server:handle_did_open({
        textDocument = {
            uri = "file:///test.lua",
            languageId = "lua",
            version = 1,
            text = test_code
        }
    })
    
    assert(#diagnostics_received == 1, "Should receive diagnostics")
    local diags = diagnostics_received[1].diagnostics
    assert(#diags >= 2, "Should find print and TODO")
    
    print("LSP diagnostics test passed!")
    return true
end

local ok, err = pcall(test_diagnostics)
if ok then
    print("All LSP tests passed!")
else
    print("Test failed:", err)
end
```

---

## 93.7 VS Code Extension สำหรับ Custom LSP

```json
// package.json สำหรับ VS Code extension
{
    "name": "my-lua-lsp",
    "displayName": "My Lua LSP",
    "description": "Custom Lua Language Server",
    "version": "0.1.0",
    "engines": { "vscode": "^1.70.0" },
    "categories": ["Programming Languages"],
    "activationEvents": ["onLanguage:lua"],
    "contributes": {
        "languages": [{
            "id": "lua",
            "extensions": [".lua"],
            "configuration": "./language-configuration.json"
        }],
        "grammars": [{
            "language": "lua",
            "scopeName": "source.lua",
            "path": "./syntaxes/lua.tmLanguage.json"
        }]
    },
    "main": "./out/extension.js"
}
```

```javascript
// extension.js (TypeScript/JavaScript)
const { LanguageClient, TransportKind } = require('vscode-languageclient/node');
const path = require('path');
const { workspace } = require('vscode');

let client;

function activate(context) {
    const serverOptions = {
        run: {
            command: 'lua',
            args: [path.join(__dirname, 'simple_lsp.lua')],
            transport: TransportKind.stdio
        }
    };

    const clientOptions = {
        documentSelector: [{ scheme: 'file', language: 'lua' }],
        synchronize: {
            fileEvents: workspace.createFileSystemWatcher('**/*.lua')
        }
    };

    client = new LanguageClient(
        'my-lua-lsp',
        'My Lua LSP',
        serverOptions,
        clientOptions
    );

    client.start();
}

function deactivate() {
    if (client) return client.stop();
}

module.exports = { activate, deactivate };
```

---

## 93.8 Semantic Tokens สำหรับ Syntax Highlighting

```lua
-- Semantic token provider
local TokenTypes = {
    "namespace", "class", "enum", "interface", "struct",
    "typeParameter", "type", "parameter", "variable", "property",
    "enumMember", "decorator", "event", "function", "method",
    "macro", "label", "comment", "string", "keyword", "number",
    "regexp", "operator"
}

local TokenModifiers = {
    "declaration", "definition", "readonly", "static", "deprecated",
    "abstract", "async", "modification", "documentation", "defaultLibrary"
}

-- Simple Lua lexer for semantic tokens
local function tokenize_lua(source)
    local tokens = {}
    local pos = 1
    local line = 0
    local char = 0
    
    local keywords = {
        "and", "break", "do", "else", "elseif", "end",
        "false", "for", "function", "goto", "if", "in",
        "local", "nil", "not", "or", "repeat", "return",
        "then", "true", "until", "while"
    }
    local keyword_set = {}
    for _, kw in ipairs(keywords) do keyword_set[kw] = true end
    
    while pos <= #source do
        local c = source:sub(pos, pos)
        
        if c == '\n' then
            line = line + 1
            char = 0
            pos = pos + 1
        elseif c:match('%s') then
            char = char + 1
            pos = pos + 1
        elseif c == '-' and source:sub(pos, pos+1) == '--' then
            -- Comment
            local end_pos = source:find('\n', pos) or #source + 1
            table.insert(tokens, {
                line = line, char = char,
                length = end_pos - pos,
                type_idx = 11,  -- comment
                modifiers = 0
            })
            char = char + (end_pos - pos)
            pos = end_pos
        elseif c == '"' or c == "'" then
            -- String
            local quote = c
            local start = pos
            pos = pos + 1
            while pos <= #source and source:sub(pos, pos) ~= quote do
                if source:sub(pos, pos) == '\\' then pos = pos + 1 end
                pos = pos + 1
            end
            pos = pos + 1
            local len = pos - start
            table.insert(tokens, {
                line = line, char = char,
                length = len,
                type_idx = 17,  -- string
                modifiers = 0
            })
            char = char + len
        elseif c:match('%d') then
            -- Number
            local num = source:match('^[%d%.]+[eE]?[%d]*', pos)
            if num then
                table.insert(tokens, {
                    line = line, char = char,
                    length = #num,
                    type_idx = 19,  -- number
                    modifiers = 0
                })
                char = char + #num
                pos = pos + #num
            else
                pos = pos + 1
                char = char + 1
            end
        elseif c:match('[%a_]') then
            -- Identifier or keyword
            local ident = source:match('^[%a_%d]+', pos)
            if ident then
                local type_idx
                if keyword_set[ident] then
                    type_idx = 18  -- keyword
                elseif source:sub(pos + #ident, pos + #ident) == '(' then
                    type_idx = 13  -- function
                else
                    type_idx = 9   -- variable
                end
                table.insert(tokens, {
                    line = line, char = char,
                    length = #ident,
                    type_idx = type_idx,
                    modifiers = 0
                })
                char = char + #ident
                pos = pos + #ident
            else
                pos = pos + 1
                char = char + 1
            end
        else
            pos = pos + 1
            char = char + 1
        end
    end
    
    return tokens
end

-- Test tokenizer
local test_source = [[
local function greet(name)
    local msg = "Hello, " .. name
    print(msg)
    return msg
end

greet("World")
]]

local tokens = tokenize_lua(test_source)
print(string.format("Found %d tokens", #tokens))
for i, tok in ipairs(tokens) do
    local type_name = TokenTypes[tok.type_idx + 1] or "unknown"
    if i <= 10 then  -- Show first 10
        print(string.format("  L%d:C%d len=%d type=%s",
            tok.line, tok.char, tok.length, type_name))
    end
end
```

---

## แบบฝึกหัด

1. **LSP Feature**: เพิ่ม go-to-definition ให้กับ simple LSP
2. **Formatter**: สร้าง Lua code formatter ที่ใช้ผ่าน LSP
3. **Diagnostics**: เพิ่ม rule ตรวจสอบ code quality เพิ่มเติม
4. **Snippet Provider**: เพิ่ม code snippets ให้กับ completion
5. **Workspace Symbols**: ใช้งาน workspace symbol search

---

*ต่อไป: [Part 94 - Advanced Profiling](part-94.md)*
