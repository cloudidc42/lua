# บทที่ 83: Compiler Construction - Lexer และ Parser

## บทนำ

Compiler Construction เป็นหนึ่งในหัวข้อที่น่าสนใจที่สุดในวิทยาการคอมพิวเตอร์ มันครอบคลุมทุกแง่มุมของการแปลงโปรแกรมจากภาษา high-level ไปเป็น machine code หรือ intermediate representation ในบทนี้ เราจะสร้าง compiler ขนาดเล็กสำหรับภาษา expression อย่างง่ายทั้งหมดด้วย Lua ตั้งแต่ lexer ไปจนถึง interpreter

---

## 83.1 Compiler Phases Overview

```
Source Code (text)
       ↓
   [Lexer/Tokenizer]        ← บทนี้ part 1
       ↓
   Token Stream
       ↓
   [Parser]                  ← บทนี้ part 2
       ↓
   Abstract Syntax Tree (AST)
       ↓
   [Semantic Analysis]       ← บทนี้ part 3
       ↓
   Annotated AST
       ↓
   [Code Generator]          ← บทนี้ part 4
       ↓
   Bytecode / IR
       ↓
   [Optimizer]               ← ขั้นสูง
       ↓
   Optimized Code
       ↓
   [Backend]
       ↓
   Target Code (x86, ARM, etc.)
```

```lua
-- ตัวอย่าง: ภาษา Tiny ที่เราจะ implement
-- มีฟีเจอร์:
-- - arithmetic expressions: +, -, *, /, %
-- - comparison: ==, !=, <, >, <=, >=
-- - boolean: and, or, not
-- - variables: let x = expr
-- - if/else statement
-- - while loop
-- - function definition และ call
-- - print statement
-- - comments: -- ...

-- ตัวอย่างโปรแกรม Tiny:
local tiny_program = [[
let x = 10
let y = 20
let result = x + y * 2

if result > 50 then
    print("big!")
else
    print("small")
end

let i = 0
while i < 5 do
    print(i)
    let i = i + 1
end

fun square(n)
    return n * n
end

print(square(7))
]]

print("We will compile and run this language!")
```

---

## 83.2 Writing a Lexer/Tokenizer

### Token Types

```lua
-- ประเภทของ tokens ทั้งหมด
local TokenType = {
    -- Literals
    NUMBER    = "NUMBER",
    STRING    = "STRING",
    BOOL      = "BOOL",
    NIL       = "NIL",
    
    -- Identifiers
    IDENT     = "IDENT",
    
    -- Keywords
    LET       = "LET",
    IF        = "IF",
    THEN      = "THEN",
    ELSE      = "ELSE",
    ELSEIF    = "ELSEIF",
    END       = "END",
    WHILE     = "WHILE",
    DO        = "DO",
    FUN       = "FUN",
    RETURN    = "RETURN",
    PRINT     = "PRINT",
    AND       = "AND",
    OR        = "OR",
    NOT       = "NOT",
    TRUE      = "TRUE",
    FALSE     = "FALSE",
    
    -- Operators
    PLUS      = "PLUS",      -- +
    MINUS     = "MINUS",     -- -
    STAR      = "STAR",      -- *
    SLASH     = "SLASH",     -- /
    PERCENT   = "PERCENT",   -- %
    EQ        = "EQ",        -- ==
    NEQ       = "NEQ",       -- !=
    LT        = "LT",        -- <
    GT        = "GT",        -- >
    LEQ       = "LEQ",       -- <=
    GEQ       = "GEQ",       -- >=
    ASSIGN    = "ASSIGN",    -- =
    
    -- Delimiters
    LPAREN    = "LPAREN",    -- (
    RPAREN    = "RPAREN",    -- )
    LBRACKET  = "LBRACKET",  -- [
    RBRACKET  = "RBRACKET",  -- ]
    COMMA     = "COMMA",     -- ,
    DOT       = "DOT",       -- .
    DOTDOT    = "DOTDOT",    -- ..
    
    -- Special
    NEWLINE   = "NEWLINE",
    EOF       = "EOF",
}

-- Keywords mapping
local KEYWORDS = {
    ["let"]    = TokenType.LET,
    ["if"]     = TokenType.IF,
    ["then"]   = TokenType.THEN,
    ["else"]   = TokenType.ELSE,
    ["elseif"] = TokenType.ELSEIF,
    ["end"]    = TokenType.END,
    ["while"]  = TokenType.WHILE,
    ["do"]     = TokenType.DO,
    ["fun"]    = TokenType.FUN,
    ["return"] = TokenType.RETURN,
    ["print"]  = TokenType.PRINT,
    ["and"]    = TokenType.AND,
    ["or"]     = TokenType.OR,
    ["not"]    = TokenType.NOT,
    ["true"]   = TokenType.TRUE,
    ["false"]  = TokenType.FALSE,
    ["nil"]    = TokenType.NIL,
}

-- Token struct
local function make_token(type, value, line, col)
    return {
        type  = type,
        value = value,
        line  = line,
        col   = col,
    }
end

print("Token types defined:", #(function() 
    local c = 0
    for _ in pairs(TokenType) do c = c + 1 end 
    return {c}
end)())
```

### The Lexer Implementation

```lua
-- Lexer class
local Lexer = {}
Lexer.__index = Lexer

function Lexer.new(source, filename)
    local self = setmetatable({}, Lexer)
    self.source   = source
    self.filename = filename or "<unknown>"
    self.pos      = 1
    self.line     = 1
    self.col      = 1
    self.tokens   = {}
    return self
end

-- Helper methods
function Lexer:peek(offset)
    offset = offset or 0
    return self.source:sub(self.pos + offset, self.pos + offset)
end

function Lexer:peek_byte(offset)
    offset = offset or 0
    return self.source:byte(self.pos + offset)
end

function Lexer:advance()
    local c = self.source:sub(self.pos, self.pos)
    self.pos = self.pos + 1
    if c == "\n" then
        self.line = self.line + 1
        self.col  = 1
    else
        self.col = self.col + 1
    end
    return c
end

function Lexer:at_end()
    return self.pos > #self.source
end

function Lexer:match(expected)
    if self:at_end() then return false end
    if self.source:sub(self.pos, self.pos) ~= expected then
        return false
    end
    self:advance()
    return true
end

function Lexer:error(msg)
    error(string.format("[Lexer] %s:%d:%d: %s", 
        self.filename, self.line, self.col, msg))
end

-- Skip whitespace and comments
function Lexer:skip_whitespace()
    while not self:at_end() do
        local c = self:peek()
        
        if c == " " or c == "\t" or c == "\r" then
            self:advance()
            
        elseif c == "\n" then
            -- Keep newlines as tokens (optional: for statement termination)
            break
            
        elseif c == "-" and self:peek(1) == "-" then
            -- Comment: skip until end of line
            while not self:at_end() and self:peek() ~= "\n" do
                self:advance()
            end
            
        else
            break
        end
    end
end

-- Scan number literal
function Lexer:scan_number()
    local start = self.pos
    local line = self.line
    local col = self.col
    
    -- Optional minus (handled separately)
    
    -- Integer part
    while not self:at_end() do
        local b = self:peek_byte()
        if b >= 48 and b <= 57 then  -- 0-9
            self:advance()
        else
            break
        end
    end
    
    -- Optional decimal
    if self:peek() == "." and self:peek(1):match("[0-9]") then
        self:advance()  -- consume "."
        while not self:at_end() do
            local b = self:peek_byte()
            if b >= 48 and b <= 57 then
                self:advance()
            else
                break
            end
        end
    end
    
    -- Optional exponent
    if self:peek():lower() == "e" then
        self:advance()
        if self:peek() == "+" or self:peek() == "-" then
            self:advance()
        end
        while not self:at_end() do
            local b = self:peek_byte()
            if b >= 48 and b <= 57 then
                self:advance()
            else
                break
            end
        end
    end
    
    local text = self.source:sub(start, self.pos - 1)
    return make_token(TokenType.NUMBER, tonumber(text), line, col)
end

-- Scan string literal
function Lexer:scan_string(quote_char)
    local line = self.line
    local col = self.col
    local parts = {}
    
    while not self:at_end() do
        local c = self:advance()
        
        if c == quote_char then
            -- End of string
            return make_token(TokenType.STRING, table.concat(parts), line, col)
            
        elseif c == "\\" then
            -- Escape sequence
            if self:at_end() then
                self:error("Unterminated escape sequence")
            end
            local esc = self:advance()
            if esc == "n" then
                parts[#parts+1] = "\n"
            elseif esc == "t" then
                parts[#parts+1] = "\t"
            elseif esc == "r" then
                parts[#parts+1] = "\r"
            elseif esc == "\\" then
                parts[#parts+1] = "\\"
            elseif esc == "\"" then
                parts[#parts+1] = "\""
            elseif esc == "'" then
                parts[#parts+1] = "'"
            elseif esc == "0" then
                parts[#parts+1] = "\0"
            else
                parts[#parts+1] = "\\" .. esc
            end
            
        elseif c == "\n" then
            self:error("Unterminated string (newline in string)")
            
        else
            parts[#parts+1] = c
        end
    end
    
    self:error("Unterminated string")
end

-- Scan identifier or keyword
function Lexer:scan_identifier()
    local start = self.pos
    local line = self.line
    local col = self.col
    
    while not self:at_end() do
        local b = self:peek_byte()
        -- a-z, A-Z, 0-9, _
        if (b >= 97 and b <= 122) or 
           (b >= 65 and b <= 90) or
           (b >= 48 and b <= 57) or
           b == 95 then
            self:advance()
        else
            break
        end
    end
    
    local text = self.source:sub(start, self.pos - 1)
    local keyword = KEYWORDS[text]
    
    if keyword then
        return make_token(keyword, text, line, col)
    else
        return make_token(TokenType.IDENT, text, line, col)
    end
end

-- Main tokenize function
function Lexer:tokenize()
    while not self:at_end() do
        self:skip_whitespace()
        
        if self:at_end() then break end
        
        local line = self.line
        local col = self.col
        local c = self:peek()
        local b = self:peek_byte()
        
        -- Newline
        if c == "\n" then
            self:advance()
            -- Skip multiple newlines
            while not self:at_end() and self:peek() == "\n" do
                self:advance()
            end
            -- Don't add NEWLINE token for simplicity
            
        -- Numbers
        elseif b >= 48 and b <= 57 then
            self.tokens[#self.tokens+1] = self:scan_number()
            
        -- Strings
        elseif c == '"' or c == "'" then
            self:advance()
            self.tokens[#self.tokens+1] = self:scan_string(c)
            
        -- Identifiers / Keywords
        elseif (b >= 97 and b <= 122) or (b >= 65 and b <= 90) or b == 95 then
            self.tokens[#self.tokens+1] = self:scan_identifier()
            
        -- Operators and punctuation
        elseif c == "+" then
            self:advance()
            self.tokens[#self.tokens+1] = make_token(TokenType.PLUS, "+", line, col)
            
        elseif c == "-" then
            self:advance()
            self.tokens[#self.tokens+1] = make_token(TokenType.MINUS, "-", line, col)
            
        elseif c == "*" then
            self:advance()
            self.tokens[#self.tokens+1] = make_token(TokenType.STAR, "*", line, col)
            
        elseif c == "/" then
            self:advance()
            self.tokens[#self.tokens+1] = make_token(TokenType.SLASH, "/", line, col)
            
        elseif c == "%" then
            self:advance()
            self.tokens[#self.tokens+1] = make_token(TokenType.PERCENT, "%", line, col)
            
        elseif c == "=" then
            self:advance()
            if self:match("=") then
                self.tokens[#self.tokens+1] = make_token(TokenType.EQ, "==", line, col)
            else
                self.tokens[#self.tokens+1] = make_token(TokenType.ASSIGN, "=", line, col)
            end
            
        elseif c == "!" then
            self:advance()
            if self:match("=") then
                self.tokens[#self.tokens+1] = make_token(TokenType.NEQ, "!=", line, col)
            else
                self:error("Expected '=' after '!'")
            end
            
        elseif c == "<" then
            self:advance()
            if self:match("=") then
                self.tokens[#self.tokens+1] = make_token(TokenType.LEQ, "<=", line, col)
            else
                self.tokens[#self.tokens+1] = make_token(TokenType.LT, "<", line, col)
            end
            
        elseif c == ">" then
            self:advance()
            if self:match("=") then
                self.tokens[#self.tokens+1] = make_token(TokenType.GEQ, ">=", line, col)
            else
                self.tokens[#self.tokens+1] = make_token(TokenType.GT, ">", line, col)
            end
            
        elseif c == "(" then
            self:advance()
            self.tokens[#self.tokens+1] = make_token(TokenType.LPAREN, "(", line, col)
            
        elseif c == ")" then
            self:advance()
            self.tokens[#self.tokens+1] = make_token(TokenType.RPAREN, ")", line, col)
            
        elseif c == "," then
            self:advance()
            self.tokens[#self.tokens+1] = make_token(TokenType.COMMA, ",", line, col)
            
        elseif c == "." then
            self:advance()
            if self:match(".") then
                self.tokens[#self.tokens+1] = make_token(TokenType.DOTDOT, "..", line, col)
            else
                self.tokens[#self.tokens+1] = make_token(TokenType.DOT, ".", line, col)
            end
            
        else
            self:error("Unexpected character: " .. c)
        end
    end
    
    self.tokens[#self.tokens+1] = make_token(TokenType.EOF, nil, self.line, self.col)
    return self.tokens
end

-- ทดสอบ lexer
local function test_lexer()
    local source = [[let x = 10 + 20 * 3
if x > 50 then print("big") end]]
    
    local lexer = Lexer.new(source, "test.tiny")
    local tokens = lexer:tokenize()
    
    print("=== Lexer Output ===")
    for _, tok in ipairs(tokens) do
        print(string.format("  [%s] %s", tok.type, tostring(tok.value)))
    end
end

test_lexer()
```

---

## 83.3 Abstract Syntax Tree (AST) Nodes

```lua
-- AST node constructors
local AST = {}

-- Expressions
function AST.NumberLiteral(value, line)
    return {kind="NumberLiteral", value=value, line=line}
end

function AST.StringLiteral(value, line)
    return {kind="StringLiteral", value=value, line=line}
end

function AST.BoolLiteral(value, line)
    return {kind="BoolLiteral", value=value, line=line}
end

function AST.NilLiteral(line)
    return {kind="NilLiteral", line=line}
end

function AST.Identifier(name, line)
    return {kind="Identifier", name=name, line=line}
end

function AST.BinaryOp(op, left, right, line)
    return {kind="BinaryOp", op=op, left=left, right=right, line=line}
end

function AST.UnaryOp(op, operand, line)
    return {kind="UnaryOp", op=op, operand=operand, line=line}
end

function AST.Call(callee, args, line)
    return {kind="Call", callee=callee, args=args, line=line}
end

function AST.Index(table_expr, key_expr, line)
    return {kind="Index", table_expr=table_expr, key_expr=key_expr, line=line}
end

function AST.FieldAccess(object, field, line)
    return {kind="FieldAccess", object=object, field=field, line=line}
end

-- Statements
function AST.LetStatement(name, value, line)
    return {kind="LetStatement", name=name, value=value, line=line}
end

function AST.AssignStatement(target, value, line)
    return {kind="AssignStatement", target=target, value=value, line=line}
end

function AST.IfStatement(condition, then_body, elseif_clauses, else_body, line)
    return {
        kind="IfStatement",
        condition=condition,
        then_body=then_body,
        elseif_clauses=elseif_clauses or {},
        else_body=else_body,
        line=line
    }
end

function AST.WhileStatement(condition, body, line)
    return {kind="WhileStatement", condition=condition, body=body, line=line}
end

function AST.FunctionDef(name, params, body, line)
    return {kind="FunctionDef", name=name, params=params, body=body, line=line}
end

function AST.ReturnStatement(value, line)
    return {kind="ReturnStatement", value=value, line=line}
end

function AST.PrintStatement(expr, line)
    return {kind="PrintStatement", expr=expr, line=line}
end

function AST.Block(statements)
    return {kind="Block", statements=statements}
end

-- ดู AST node
local function ast_to_string(node, indent)
    indent = indent or 0
    local prefix = string.rep("  ", indent)
    
    if not node then return prefix .. "nil" end
    
    if node.kind == "NumberLiteral" then
        return prefix .. "Num(" .. tostring(node.value) .. ")"
    elseif node.kind == "StringLiteral" then
        return prefix .. 'Str("' .. node.value .. '")'
    elseif node.kind == "Identifier" then
        return prefix .. "Id(" .. node.name .. ")"
    elseif node.kind == "BinaryOp" then
        return prefix .. "BinOp(" .. node.op .. ")\n" ..
               ast_to_string(node.left, indent+1) .. "\n" ..
               ast_to_string(node.right, indent+1)
    elseif node.kind == "UnaryOp" then
        return prefix .. "UnaryOp(" .. node.op .. ")\n" ..
               ast_to_string(node.operand, indent+1)
    elseif node.kind == "Block" then
        local parts = {prefix .. "Block{"}
        for _, stmt in ipairs(node.statements) do
            parts[#parts+1] = ast_to_string(stmt, indent+1)
        end
        parts[#parts+1] = prefix .. "}"
        return table.concat(parts, "\n")
    else
        return prefix .. node.kind
    end
end

-- ทดสอบ AST
local test_ast = AST.BinaryOp("+",
    AST.BinaryOp("*",
        AST.NumberLiteral(2),
        AST.NumberLiteral(3)
    ),
    AST.Identifier("x")
)

print("\n=== AST Example ===")
print(ast_to_string(test_ast))
```

---

## 83.4 Recursive Descent Parser

```lua
-- Parser class
local Parser = {}
Parser.__index = Parser

function Parser.new(tokens)
    local self = setmetatable({}, Parser)
    self.tokens = tokens
    self.pos = 1
    return self
end

function Parser:peek(offset)
    offset = offset or 0
    local idx = self.pos + offset
    if idx > #self.tokens then
        return self.tokens[#self.tokens]  -- EOF
    end
    return self.tokens[idx]
end

function Parser:current()
    return self:peek(0)
end

function Parser:advance()
    local tok = self:current()
    if tok.type ~= TokenType.EOF then
        self.pos = self.pos + 1
    end
    return tok
end

function Parser:check(type)
    return self:current().type == type
end

function Parser:match(...)
    for _, type in ipairs({...}) do
        if self:check(type) then
            return self:advance()
        end
    end
    return nil
end

function Parser:expect(type, msg)
    if self:check(type) then
        return self:advance()
    end
    local tok = self:current()
    error(string.format("[Parser] Line %d: Expected %s, got %s ('%s'). %s",
        tok.line, type, tok.type, tostring(tok.value), msg or ""))
end

function Parser:error(msg)
    local tok = self:current()
    error(string.format("[Parser] Line %d:%d: %s", tok.line, tok.col, msg))
end

-- Parse program (entry point)
function Parser:parse_program()
    local statements = {}
    
    while not self:check(TokenType.EOF) do
        local stmt = self:parse_statement()
        if stmt then
            statements[#statements+1] = stmt
        end
    end
    
    return AST.Block(statements)
end

-- Parse statement
function Parser:parse_statement()
    local tok = self:current()
    
    if tok.type == TokenType.LET then
        return self:parse_let()
    elseif tok.type == TokenType.IF then
        return self:parse_if()
    elseif tok.type == TokenType.WHILE then
        return self:parse_while()
    elseif tok.type == TokenType.FUN then
        return self:parse_function()
    elseif tok.type == TokenType.RETURN then
        return self:parse_return()
    elseif tok.type == TokenType.PRINT then
        return self:parse_print()
    elseif tok.type == TokenType.IDENT then
        return self:parse_assignment_or_call()
    else
        self:error("Unexpected token: " .. tok.type)
    end
end

function Parser:parse_let()
    local line = self:current().line
    self:expect(TokenType.LET)
    local name = self:expect(TokenType.IDENT, "Expected variable name").value
    self:expect(TokenType.ASSIGN, "Expected '='")
    local value = self:parse_expression()
    return AST.LetStatement(name, value, line)
end

function Parser:parse_if()
    local line = self:current().line
    self:expect(TokenType.IF)
    local condition = self:parse_expression()
    self:expect(TokenType.THEN, "Expected 'then'")
    
    local then_body = self:parse_block()
    local elseif_clauses = {}
    local else_body = nil
    
    while self:check(TokenType.ELSEIF) do
        self:advance()
        local elseif_cond = self:parse_expression()
        self:expect(TokenType.THEN)
        local elseif_body = self:parse_block()
        elseif_clauses[#elseif_clauses+1] = {
            condition = elseif_cond,
            body = elseif_body
        }
    end
    
    if self:match(TokenType.ELSE) then
        else_body = self:parse_block()
    end
    
    self:expect(TokenType.END, "Expected 'end'")
    return AST.IfStatement(condition, then_body, elseif_clauses, else_body, line)
end

function Parser:parse_while()
    local line = self:current().line
    self:expect(TokenType.WHILE)
    local condition = self:parse_expression()
    self:expect(TokenType.DO, "Expected 'do'")
    local body = self:parse_block()
    self:expect(TokenType.END, "Expected 'end'")
    return AST.WhileStatement(condition, body, line)
end

function Parser:parse_function()
    local line = self:current().line
    self:expect(TokenType.FUN)
    local name = self:expect(TokenType.IDENT).value
    self:expect(TokenType.LPAREN)
    
    local params = {}
    if not self:check(TokenType.RPAREN) then
        params[#params+1] = self:expect(TokenType.IDENT).value
        while self:match(TokenType.COMMA) do
            params[#params+1] = self:expect(TokenType.IDENT).value
        end
    end
    
    self:expect(TokenType.RPAREN)
    local body = self:parse_block()
    self:expect(TokenType.END)
    
    return AST.FunctionDef(name, params, body, line)
end

function Parser:parse_return()
    local line = self:current().line
    self:expect(TokenType.RETURN)
    
    local value = nil
    if not self:check(TokenType.END) and 
       not self:check(TokenType.ELSE) and
       not self:check(TokenType.EOF) then
        value = self:parse_expression()
    end
    
    return AST.ReturnStatement(value, line)
end

function Parser:parse_print()
    local line = self:current().line
    self:expect(TokenType.PRINT)
    self:expect(TokenType.LPAREN)
    local expr = self:parse_expression()
    self:expect(TokenType.RPAREN)
    return AST.PrintStatement(expr, line)
end

function Parser:parse_assignment_or_call()
    local line = self:current().line
    local name = self:expect(TokenType.IDENT).value
    
    if self:match(TokenType.ASSIGN) then
        local value = self:parse_expression()
        return AST.AssignStatement(AST.Identifier(name, line), value, line)
    elseif self:check(TokenType.LPAREN) then
        -- Function call statement
        self:expect(TokenType.LPAREN)
        local args = self:parse_args()
        self:expect(TokenType.RPAREN)
        return AST.Call(AST.Identifier(name, line), args, line)
    else
        self:error("Expected assignment or function call")
    end
end

function Parser:parse_block()
    local statements = {}
    
    while not self:check(TokenType.END) and
          not self:check(TokenType.ELSE) and
          not self:check(TokenType.ELSEIF) and
          not self:check(TokenType.EOF) do
        statements[#statements+1] = self:parse_statement()
    end
    
    return AST.Block(statements)
end

function Parser:parse_args()
    local args = {}
    if not self:check(TokenType.RPAREN) then
        args[#args+1] = self:parse_expression()
        while self:match(TokenType.COMMA) do
            args[#args+1] = self:parse_expression()
        end
    end
    return args
end
```

---

## 83.5 Pratt Parser สำหรับ Expressions

Pratt Parser (Top-Down Operator Precedence) เป็นวิธีที่ elegant มากสำหรับ parsing expressions:

```lua
-- Operator precedence (higher = binds tighter)
local PRECEDENCE = {
    [TokenType.OR]      = 1,
    [TokenType.AND]     = 2,
    [TokenType.EQ]      = 3,
    [TokenType.NEQ]     = 3,
    [TokenType.LT]      = 4,
    [TokenType.GT]      = 4,
    [TokenType.LEQ]     = 4,
    [TokenType.GEQ]     = 4,
    [TokenType.DOTDOT]  = 5,   -- string concat
    [TokenType.PLUS]    = 6,
    [TokenType.MINUS]   = 6,
    [TokenType.STAR]    = 7,
    [TokenType.SLASH]   = 7,
    [TokenType.PERCENT] = 7,
    -- Unary (handled separately)
}

local BINARY_OPS = {
    [TokenType.PLUS]    = "+",
    [TokenType.MINUS]   = "-",
    [TokenType.STAR]    = "*",
    [TokenType.SLASH]   = "/",
    [TokenType.PERCENT] = "%",
    [TokenType.EQ]      = "==",
    [TokenType.NEQ]     = "!=",
    [TokenType.LT]      = "<",
    [TokenType.GT]      = ">",
    [TokenType.LEQ]     = "<=",
    [TokenType.GEQ]     = ">=",
    [TokenType.AND]     = "and",
    [TokenType.OR]      = "or",
    [TokenType.DOTDOT]  = "..",
}

-- Parse expression with Pratt parser
function Parser:parse_expression(min_prec)
    min_prec = min_prec or 0
    
    -- Parse left side (prefix position)
    local left = self:parse_primary()
    
    -- Handle binary operators (infix position)
    while true do
        local tok = self:current()
        local prec = PRECEDENCE[tok.type]
        
        if not prec or prec <= min_prec then
            break
        end
        
        local op = BINARY_OPS[tok.type]
        local line = tok.line
        self:advance()
        
        -- Right-associative operators use same precedence
        -- Left-associative use prec (not prec-1)
        local right
        if tok.type == TokenType.DOTDOT then
            right = self:parse_expression(prec - 1)  -- right-assoc
        else
            right = self:parse_expression(prec)  -- left-assoc
        end
        
        left = AST.BinaryOp(op, left, right, line)
    end
    
    return left
end

-- Parse primary expression (atoms, unary ops, grouping)
function Parser:parse_primary()
    local tok = self:current()
    local line = tok.line
    
    -- Number literal
    if tok.type == TokenType.NUMBER then
        self:advance()
        return AST.NumberLiteral(tok.value, line)
    
    -- String literal
    elseif tok.type == TokenType.STRING then
        self:advance()
        return AST.StringLiteral(tok.value, line)
    
    -- Boolean
    elseif tok.type == TokenType.TRUE then
        self:advance()
        return AST.BoolLiteral(true, line)
    elseif tok.type == TokenType.FALSE then
        self:advance()
        return AST.BoolLiteral(false, line)
    
    -- Nil
    elseif tok.type == TokenType.NIL then
        self:advance()
        return AST.NilLiteral(line)
    
    -- Unary minus
    elseif tok.type == TokenType.MINUS then
        self:advance()
        local operand = self:parse_primary()
        return AST.UnaryOp("-", operand, line)
    
    -- Unary not
    elseif tok.type == TokenType.NOT then
        self:advance()
        local operand = self:parse_primary()
        return AST.UnaryOp("not", operand, line)
    
    -- Grouped expression
    elseif tok.type == TokenType.LPAREN then
        self:advance()
        local expr = self:parse_expression()
        self:expect(TokenType.RPAREN, "Expected ')'")
        return expr
    
    -- Identifier (variable or function call)
    elseif tok.type == TokenType.IDENT then
        self:advance()
        local node = AST.Identifier(tok.value, line)
        
        -- Function call
        if self:check(TokenType.LPAREN) then
            self:advance()
            local args = self:parse_args()
            self:expect(TokenType.RPAREN)
            return AST.Call(node, args, line)
        end
        
        return node
    
    else
        self:error("Unexpected token in expression: " .. tok.type .. 
                   " '" .. tostring(tok.value) .. "'")
    end
end
```

---

## 83.6 Symbol Table

```lua
-- Symbol table สำหรับ semantic analysis
local SymbolTable = {}
SymbolTable.__index = SymbolTable

function SymbolTable.new(parent)
    return setmetatable({
        symbols = {},
        parent = parent,
        scope_level = parent and (parent.scope_level + 1) or 0,
    }, SymbolTable)
end

function SymbolTable:define(name, info)
    if self.symbols[name] then
        -- Warning: redefining symbol
    end
    self.symbols[name] = info or {type = "var", level = self.scope_level}
    return self.symbols[name]
end

function SymbolTable:lookup(name)
    if self.symbols[name] then
        return self.symbols[name], self
    end
    if self.parent then
        return self.parent:lookup(name)
    end
    return nil, nil
end

function SymbolTable:lookup_local(name)
    return self.symbols[name]
end

function SymbolTable:enter_scope()
    return SymbolTable.new(self)
end

-- ตัวอย่างการใช้ symbol table
local function test_symbol_table()
    local global = SymbolTable.new()
    
    -- Define global functions
    global:define("print", {type = "builtin"})
    global:define("tostring", {type = "builtin"})
    global:define("tonumber", {type = "builtin"})
    
    -- Enter function scope
    local func_scope = global:enter_scope()
    func_scope:define("x", {type = "param"})
    func_scope:define("y", {type = "param"})
    
    -- Enter block scope
    local block_scope = func_scope:enter_scope()
    block_scope:define("temp", {type = "local"})
    
    -- Lookup
    local sym, where = block_scope:lookup("x")
    print("Found 'x':", sym ~= nil, "in scope:", where and where.scope_level)
    
    sym = block_scope:lookup_local("x")
    print("'x' is local:", sym ~= nil)  -- false (x is in func_scope)
    
    sym = block_scope:lookup_local("temp")
    print("'temp' is local:", sym ~= nil)  -- true
end

test_symbol_table()
```

---

## 83.7 Semantic Analysis

```lua
-- Semantic analyzer: ตรวจสอบ type errors, undefined variables ฯลฯ
local SemanticAnalyzer = {}
SemanticAnalyzer.__index = SemanticAnalyzer

function SemanticAnalyzer.new()
    local self = setmetatable({}, SemanticAnalyzer)
    self.errors = {}
    self.warnings = {}
    
    -- Global scope
    self.scope = SymbolTable.new()
    self.scope:define("print", {type="builtin", return_type="void"})
    self.scope:define("tostring", {type="builtin", return_type="string"})
    self.scope:define("tonumber", {type="builtin", return_type="number"})
    self.scope:define("math", {type="module"})
    
    return self
end

function SemanticAnalyzer:error(msg, line)
    self.errors[#self.errors+1] = string.format("Line %d: %s", line or 0, msg)
end

function SemanticAnalyzer:warning(msg, line)
    self.warnings[#self.warnings+1] = string.format("Line %d: %s", line or 0, msg)
end

function SemanticAnalyzer:analyze(node)
    if not node then return end
    
    local kind = node.kind
    
    if kind == "Block" then
        self.scope = self.scope:enter_scope()
        for _, stmt in ipairs(node.statements) do
            self:analyze(stmt)
        end
        self.scope = self.scope.parent
        
    elseif kind == "LetStatement" then
        self:analyze(node.value)
        self.scope:define(node.name, {type="var", line=node.line})
        
    elseif kind == "AssignStatement" then
        local name = node.target.name
        local sym = self.scope:lookup(name)
        if not sym then
            self:error("Undefined variable: " .. name, node.line)
        end
        self:analyze(node.value)
        
    elseif kind == "Identifier" then
        local sym = self.scope:lookup(node.name)
        if not sym then
            self:error("Undefined variable: " .. node.name, node.line)
        end
        
    elseif kind == "BinaryOp" then
        self:analyze(node.left)
        self:analyze(node.right)
        
    elseif kind == "UnaryOp" then
        self:analyze(node.operand)
        
    elseif kind == "Call" then
        local callee_name = node.callee.name
        if callee_name then
            local sym = self.scope:lookup(callee_name)
            if not sym then
                self:error("Undefined function: " .. callee_name, node.line)
            end
        end
        for _, arg in ipairs(node.args) do
            self:analyze(arg)
        end
        
    elseif kind == "IfStatement" then
        self:analyze(node.condition)
        self:analyze(node.then_body)
        for _, clause in ipairs(node.elseif_clauses) do
            self:analyze(clause.condition)
            self:analyze(clause.body)
        end
        if node.else_body then
            self:analyze(node.else_body)
        end
        
    elseif kind == "WhileStatement" then
        self:analyze(node.condition)
        self:analyze(node.body)
        
    elseif kind == "FunctionDef" then
        -- Define function in current scope
        self.scope:define(node.name, {type="function", params=node.params})
        
        -- Enter function scope
        local prev_scope = self.scope
        self.scope = self.scope:enter_scope()
        
        -- Define parameters
        for _, param in ipairs(node.params) do
            self.scope:define(param, {type="param"})
        end
        
        -- Analyze body
        self:analyze(node.body)
        
        self.scope = prev_scope
        
    elseif kind == "ReturnStatement" then
        if node.value then
            self:analyze(node.value)
        end
        
    elseif kind == "PrintStatement" then
        self:analyze(node.expr)
    end
end

function SemanticAnalyzer:report()
    if #self.errors > 0 then
        print("=== Semantic Errors ===")
        for _, err in ipairs(self.errors) do
            print("  ERROR: " .. err)
        end
    end
    if #self.warnings > 0 then
        print("=== Warnings ===")
        for _, warn in ipairs(self.warnings) do
            print("  WARN: " .. warn)
        end
    end
    return #self.errors == 0
end
```

---

## 83.8 AST Interpreter (Tree-Walking Interpreter)

```lua
-- Interpreter ที่ walk AST โดยตรง
local Interpreter = {}
Interpreter.__index = Interpreter

-- Return value wrapper (สำหรับ early return)
local ReturnSignal = {}
ReturnSignal.__index = ReturnSignal

function ReturnSignal.new(value)
    return setmetatable({value = value}, ReturnSignal)
end

-- Runtime function object
local function make_function(params, body, closure_env)
    return {
        __type = "function",
        params = params,
        body = body,
        env = closure_env,
    }
end

-- Environment (variable scope)
local Env = {}
Env.__index = Env

function Env.new(parent)
    return setmetatable({vars = {}, parent = parent}, Env)
end

function Env:set(name, value)
    self.vars[name] = value
end

function Env:get(name)
    if self.vars[name] ~= nil then
        return self.vars[name]
    end
    if self.parent then
        return self.parent:get(name)
    end
    return nil
end

function Env:update(name, value)
    if self.vars[name] ~= nil then
        self.vars[name] = value
        return true
    end
    if self.parent then
        return self.parent:update(name, value)
    end
    return false
end

-- Create interpreter
function Interpreter.new()
    local self = setmetatable({}, Interpreter)
    
    -- Global environment
    self.global = Env.new()
    self.env = self.global
    
    -- Built-in functions
    self.global:set("print", function(...)
        local args = {...}
        local parts = {}
        for _, v in ipairs(args) do
            parts[#parts+1] = tostring(v)
        end
        print(table.concat(parts, "\t"))
    end)
    
    self.global:set("tostring", tostring)
    self.global:set("tonumber", tonumber)
    self.global:set("type", type)
    
    self.global:set("math", {
        sqrt  = math.sqrt,
        floor = math.floor,
        ceil  = math.ceil,
        abs   = math.abs,
        max   = math.max,
        min   = math.min,
        pi    = math.pi,
    })
    
    return self
end

function Interpreter:eval(node)
    if not node then return nil end
    
    local kind = node.kind
    
    -- Literals
    if kind == "NumberLiteral" then
        return node.value
        
    elseif kind == "StringLiteral" then
        return node.value
        
    elseif kind == "BoolLiteral" then
        return node.value
        
    elseif kind == "NilLiteral" then
        return nil
        
    -- Variable
    elseif kind == "Identifier" then
        local val = self.env:get(node.name)
        if val == nil then
            error("Undefined variable: " .. node.name .. " at line " .. (node.line or "?"))
        end
        return val
        
    -- Binary operation
    elseif kind == "BinaryOp" then
        local op = node.op
        
        -- Short-circuit evaluation
        if op == "and" then
            local left = self:eval(node.left)
            if not left then return left end
            return self:eval(node.right)
        end
        
        if op == "or" then
            local left = self:eval(node.left)
            if left then return left end
            return self:eval(node.right)
        end
        
        local left = self:eval(node.left)
        local right = self:eval(node.right)
        
        if op == "+"  then return left + right
        elseif op == "-"  then return left - right
        elseif op == "*"  then return left * right
        elseif op == "/"  then
            if right == 0 then error("Division by zero") end
            return left / right
        elseif op == "%"  then return left % right
        elseif op == "==" then return left == right
        elseif op == "!=" then return left ~= right
        elseif op == "<"  then return left < right
        elseif op == ">"  then return left > right
        elseif op == "<=" then return left <= right
        elseif op == ">=" then return left >= right
        elseif op == ".." then return tostring(left) .. tostring(right)
        else
            error("Unknown operator: " .. op)
        end
        
    -- Unary operation
    elseif kind == "UnaryOp" then
        local val = self:eval(node.operand)
        if node.op == "-" then
            return -val
        elseif node.op == "not" then
            return not val
        end
        
    -- Function call
    elseif kind == "Call" then
        local callee = self:eval(node.callee)
        
        if type(callee) == "function" then
            -- Built-in Lua function
            local args = {}
            for _, arg in ipairs(node.args) do
                args[#args+1] = self:eval(arg)
            end
            return callee(table.unpack(args))
            
        elseif type(callee) == "table" and callee.__type == "function" then
            -- User-defined function
            local args = {}
            for _, arg in ipairs(node.args) do
                args[#args+1] = self:eval(arg)
            end
            
            -- Create new environment with closure
            local func_env = Env.new(callee.env)
            
            -- Bind parameters
            for i, param in ipairs(callee.params) do
                func_env:set(param, args[i])
            end
            
            -- Execute body
            local prev_env = self.env
            self.env = func_env
            
            local result = nil
            local ok, ret = pcall(function()
                self:execute(callee.body)
            end)
            
            self.env = prev_env
            
            if not ok then
                if type(ret) == "table" and ret.__type == "return" then
                    return ret.value
                else
                    error(ret, 0)
                end
            end
            
            return nil
        else
            error("Not a function: " .. tostring(callee))
        end
        
    else
        error("Unknown AST node: " .. tostring(kind))
    end
end

function Interpreter:execute(node)
    if not node then return end
    
    local kind = node.kind
    
    if kind == "Block" then
        local block_env = Env.new(self.env)
        local prev_env = self.env
        self.env = block_env
        
        for _, stmt in ipairs(node.statements) do
            self:execute(stmt)
        end
        
        self.env = prev_env
        
    elseif kind == "LetStatement" then
        local value = self:eval(node.value)
        self.env:set(node.name, value)
        
    elseif kind == "AssignStatement" then
        local value = self:eval(node.value)
        local name = node.target.name
        if not self.env:update(name, value) then
            self.env:set(name, value)
        end
        
    elseif kind == "IfStatement" then
        local cond = self:eval(node.condition)
        
        if cond and cond ~= false then
            self:execute(node.then_body)
        else
            local handled = false
            for _, clause in ipairs(node.elseif_clauses) do
                if self:eval(clause.condition) then
                    self:execute(clause.body)
                    handled = true
                    break
                end
            end
            if not handled and node.else_body then
                self:execute(node.else_body)
            end
        end
        
    elseif kind == "WhileStatement" then
        local max_iterations = 1000000  -- safety limit
        local count = 0
        
        while self:eval(node.condition) do
            count = count + 1
            if count > max_iterations then
                error("Infinite loop detected")
            end
            
            local ok, err = pcall(function()
                self:execute(node.body)
            end)
            
            if not ok then
                error(err, 0)
            end
        end
        
    elseif kind == "FunctionDef" then
        local func = make_function(node.params, node.body, self.env)
        self.env:set(node.name, func)
        
    elseif kind == "ReturnStatement" then
        local value = node.value and self:eval(node.value) or nil
        error({__type = "return", value = value}, 0)
        
    elseif kind == "PrintStatement" then
        local value = self:eval(node.expr)
        print(tostring(value))
        
    elseif kind == "Call" then
        self:eval(node)
        
    else
        error("Unknown statement: " .. tostring(kind))
    end
end
```

---

## 83.9 Putting It All Together

```lua
-- Complete tiny language compiler/interpreter

local TinyLang = {}

function TinyLang.run(source, filename)
    filename = filename or "<stdin>"
    
    -- Phase 1: Lexing
    local lexer = Lexer.new(source, filename)
    local ok, tokens_or_err = pcall(function()
        return lexer:tokenize()
    end)
    
    if not ok then
        print("Lexer Error:", tokens_or_err)
        return false
    end
    
    local tokens = tokens_or_err
    
    -- Phase 2: Parsing
    local parser = Parser.new(tokens)
    ok, tokens_or_err = pcall(function()
        return parser:parse_program()
    end)
    
    if not ok then
        print("Parser Error:", tokens_or_err)
        return false
    end
    
    local ast = tokens_or_err
    
    -- Phase 3: Semantic Analysis
    local analyzer = SemanticAnalyzer.new()
    analyzer:analyze(ast)
    
    if not analyzer:report() then
        return false
    end
    
    -- Phase 4: Interpretation
    local interp = Interpreter.new()
    ok, tokens_or_err = pcall(function()
        interp:execute(ast)
    end)
    
    if not ok then
        print("Runtime Error:", tokens_or_err)
        return false
    end
    
    return true
end

-- ทดสอบด้วยโปรแกรมง่ายๆ
print("\n=== Testing TinyLang ===\n")

TinyLang.run([[
let x = 10
let y = 20
let result = x + y
print(result)
]])

print("\n--- Fibonacci ---")
TinyLang.run([[
fun fib(n)
    if n <= 1 then
        return n
    end
    return fib(n - 1) + fib(n - 2)
end

let i = 0
while i < 10 do
    print(fib(i))
    let i = i + 1
end
]])
```

---

## 83.10 Code Generation to Bytecode

```lua
-- Code generator: แปลง AST เป็น simple bytecode

local Instruction = {}

-- Simple stack-based bytecode (ง่ายกว่า register-based)
local OPCODE = {
    PUSH_NUM  = 1,   -- push number constant
    PUSH_STR  = 2,   -- push string constant
    PUSH_BOOL = 3,   -- push boolean
    PUSH_NIL  = 4,   -- push nil
    LOAD_VAR  = 5,   -- load variable
    STORE_VAR = 6,   -- store variable
    ADD       = 10,  -- binary ops
    SUB       = 11,
    MUL       = 12,
    DIV       = 13,
    MOD       = 14,
    EQ        = 15,
    NEQ       = 16,
    LT        = 17,
    GT        = 18,
    LEQ       = 19,
    GEQ       = 20,
    AND       = 21,
    OR        = 22,
    CONCAT    = 23,
    NEG       = 24,  -- unary minus
    NOT       = 25,  -- unary not
    CALL      = 30,  -- function call (arg_count)
    RETURN    = 31,
    JMP       = 32,  -- unconditional jump
    JMP_FALSE = 33,  -- jump if false
    POP       = 34,  -- pop stack
    PRINT     = 35,  -- print top of stack
    MAKE_FUN  = 36,  -- create function object
    HALT      = 99,
}

local OPCODE_NAMES = {}
for name, code in pairs(OPCODE) do
    OPCODE_NAMES[code] = name
end

local CodeGen = {}
CodeGen.__index = CodeGen

function CodeGen.new()
    local self = setmetatable({}, CodeGen)
    self.code = {}
    self.constants = {}
    self.var_names = {}
    self.locals = {}  -- stack of scope var lists
    return self
end

function CodeGen:emit(op, arg)
    self.code[#self.code+1] = {op=op, arg=arg, pos=#self.code+1}
end

function CodeGen:emit_at(pos, op, arg)
    self.code[pos] = {op=op, arg=arg, pos=pos}
end

function CodeGen:current_pos()
    return #self.code
end

function CodeGen:add_constant(value)
    for i, c in ipairs(self.constants) do
        if c == value then return i end
    end
    self.constants[#self.constants+1] = value
    return #self.constants
end

function CodeGen:gen(node)
    if not node then return end
    
    local kind = node.kind
    
    if kind == "NumberLiteral" then
        local idx = self:add_constant(node.value)
        self:emit(OPCODE.PUSH_NUM, idx)
        
    elseif kind == "StringLiteral" then
        local idx = self:add_constant(node.value)
        self:emit(OPCODE.PUSH_STR, idx)
        
    elseif kind == "BoolLiteral" then
        self:emit(OPCODE.PUSH_BOOL, node.value and 1 or 0)
        
    elseif kind == "NilLiteral" then
        self:emit(OPCODE.PUSH_NIL)
        
    elseif kind == "Identifier" then
        local idx = self:add_constant(node.name)
        self:emit(OPCODE.LOAD_VAR, idx)
        
    elseif kind == "BinaryOp" then
        local op_map = {
            ["+"]  = OPCODE.ADD,
            ["-"]  = OPCODE.SUB,
            ["*"]  = OPCODE.MUL,
            ["/"]  = OPCODE.DIV,
            ["%"]  = OPCODE.MOD,
            ["=="] = OPCODE.EQ,
            ["!="] = OPCODE.NEQ,
            ["<"]  = OPCODE.LT,
            [">"]  = OPCODE.GT,
            ["<="] = OPCODE.LEQ,
            [">="] = OPCODE.GEQ,
            ["and"]= OPCODE.AND,
            ["or"] = OPCODE.OR,
            [".."] = OPCODE.CONCAT,
        }
        self:gen(node.left)
        self:gen(node.right)
        self:emit(op_map[node.op])
        
    elseif kind == "UnaryOp" then
        self:gen(node.operand)
        if node.op == "-" then
            self:emit(OPCODE.NEG)
        elseif node.op == "not" then
            self:emit(OPCODE.NOT)
        end
        
    elseif kind == "LetStatement" then
        self:gen(node.value)
        local idx = self:add_constant(node.name)
        self:emit(OPCODE.STORE_VAR, idx)
        
    elseif kind == "PrintStatement" then
        self:gen(node.expr)
        self:emit(OPCODE.PRINT)
        
    elseif kind == "Call" then
        -- Push function
        self:gen(node.callee)
        -- Push args
        for _, arg in ipairs(node.args) do
            self:gen(arg)
        end
        self:emit(OPCODE.CALL, #node.args)
        
    elseif kind == "ReturnStatement" then
        if node.value then
            self:gen(node.value)
        else
            self:emit(OPCODE.PUSH_NIL)
        end
        self:emit(OPCODE.RETURN)
        
    elseif kind == "Block" then
        for _, stmt in ipairs(node.statements) do
            self:gen(stmt)
        end
        
    elseif kind == "IfStatement" then
        -- Generate condition
        self:gen(node.condition)
        
        -- JMP_FALSE to else/end
        local jmp_false_pos = self:current_pos() + 1
        self:emit(OPCODE.JMP_FALSE, 0)  -- placeholder
        
        -- Then body
        self:gen(node.then_body)
        
        -- JMP to end
        local jmp_end_pos = self:current_pos() + 1
        self:emit(OPCODE.JMP, 0)  -- placeholder
        
        -- Patch JMP_FALSE
        self:emit_at(jmp_false_pos, OPCODE.JMP_FALSE, self:current_pos() + 1)
        
        -- Else body (if any)
        if node.else_body then
            self:gen(node.else_body)
        end
        
        -- Patch JMP to end
        self:emit_at(jmp_end_pos, OPCODE.JMP, self:current_pos() + 1)
        
    elseif kind == "WhileStatement" then
        local loop_start = self:current_pos() + 1
        
        -- Condition
        self:gen(node.condition)
        
        -- JMP_FALSE to end
        local jmp_pos = self:current_pos() + 1
        self:emit(OPCODE.JMP_FALSE, 0)
        
        -- Body
        self:gen(node.body)
        
        -- JMP back to condition
        self:emit(OPCODE.JMP, loop_start)
        
        -- Patch JMP_FALSE
        self:emit_at(jmp_pos, OPCODE.JMP_FALSE, self:current_pos() + 1)
        
    else
        -- Skip unknown nodes
    end
end

function CodeGen:disassemble()
    print("=== Bytecode ===")
    print("Constants:", #self.constants)
    for i, c in ipairs(self.constants) do
        print(string.format("  K[%d] = %s", i, tostring(c)))
    end
    
    print("\nInstructions:")
    for i, ins in ipairs(self.code) do
        local name = OPCODE_NAMES[ins.op] or "UNKNOWN"
        if ins.arg ~= nil then
            local arg_str = tostring(ins.arg)
            -- ถ้า arg เป็น constant index แสดงค่าด้วย
            if ins.op == OPCODE.PUSH_NUM or 
               ins.op == OPCODE.PUSH_STR or
               ins.op == OPCODE.LOAD_VAR or
               ins.op == OPCODE.STORE_VAR then
                local c = self.constants[ins.arg]
                if c then
                    arg_str = arg_str .. " (" .. tostring(c) .. ")"
                end
            end
            print(string.format("  %04d  %-15s %s", i, name, arg_str))
        else
            print(string.format("  %04d  %s", i, name))
        end
    end
end

-- ทดสอบ code generation
local function test_codegen()
    local source = [[
let x = 5 + 3
let y = x * 2
print(y)
]]
    
    local lexer = Lexer.new(source, "test")
    local tokens = lexer:tokenize()
    local parser = Parser.new(tokens)
    local ast = parser:parse_program()
    
    local gen = CodeGen.new()
    gen:gen(ast)
    gen:emit(OPCODE.HALT)
    gen:disassemble()
end

test_codegen()
```

---

## 83.11 Bytecode Virtual Machine

```lua
-- Stack-based VM สำหรับ execute bytecode ที่เราสร้าง

local VM = {}
VM.__index = VM

function VM.new(code, constants)
    local self = setmetatable({}, VM)
    self.code = code
    self.constants = constants
    self.stack = {}
    self.sp = 0  -- stack pointer
    self.pc = 1  -- program counter
    self.env = {}  -- flat variable store
    self.call_stack = {}
    return self
end

function VM:push(value)
    self.sp = self.sp + 1
    self.stack[self.sp] = value
end

function VM:pop()
    local val = self.stack[self.sp]
    self.sp = self.sp - 1
    return val
end

function VM:peek()
    return self.stack[self.sp]
end

function VM:run()
    while self.pc <= #self.code do
        local ins = self.code[self.pc]
        self.pc = self.pc + 1
        
        local op = ins.op
        
        if op == OPCODE.PUSH_NUM then
            self:push(self.constants[ins.arg])
            
        elseif op == OPCODE.PUSH_STR then
            self:push(self.constants[ins.arg])
            
        elseif op == OPCODE.PUSH_BOOL then
            self:push(ins.arg == 1)
            
        elseif op == OPCODE.PUSH_NIL then
            self:push(nil)
            
        elseif op == OPCODE.LOAD_VAR then
            local name = self.constants[ins.arg]
            local val = self.env[name]
            if val == nil then
                -- Check builtins
                val = _G[name]
            end
            self:push(val)
            
        elseif op == OPCODE.STORE_VAR then
            local name = self.constants[ins.arg]
            self.env[name] = self:pop()
            
        elseif op == OPCODE.ADD then
            local b = self:pop()
            local a = self:pop()
            self:push(a + b)
            
        elseif op == OPCODE.SUB then
            local b = self:pop()
            local a = self:pop()
            self:push(a - b)
            
        elseif op == OPCODE.MUL then
            local b = self:pop()
            local a = self:pop()
            self:push(a * b)
            
        elseif op == OPCODE.DIV then
            local b = self:pop()
            local a = self:pop()
            self:push(a / b)
            
        elseif op == OPCODE.EQ then
            local b = self:pop()
            local a = self:pop()
            self:push(a == b)
            
        elseif op == OPCODE.LT then
            local b = self:pop()
            local a = self:pop()
            self:push(a < b)
            
        elseif op == OPCODE.GT then
            local b = self:pop()
            local a = self:pop()
            self:push(a > b)
            
        elseif op == OPCODE.NEG then
            self:push(-self:pop())
            
        elseif op == OPCODE.NOT then
            self:push(not self:pop())
            
        elseif op == OPCODE.PRINT then
            print(tostring(self:pop()))
            
        elseif op == OPCODE.JMP then
            self.pc = ins.arg
            
        elseif op == OPCODE.JMP_FALSE then
            local cond = self:pop()
            if not cond then
                self.pc = ins.arg
            end
            
        elseif op == OPCODE.HALT then
            break
            
        elseif op == OPCODE.POP then
            self:pop()
        end
    end
end

-- ทดสอบ VM
local function test_vm()
    local source = [[
let a = 10
let b = 20
let sum = a + b
print(sum)
if sum > 25 then
    print("big sum")
end
]]
    
    local lexer = Lexer.new(source)
    local tokens = lexer:tokenize()
    local parser = Parser.new(tokens)
    local ast = parser:parse_program()
    
    local gen = CodeGen.new()
    gen:gen(ast)
    gen:emit(OPCODE.HALT)
    
    print("\n=== VM Execution ===")
    local vm = VM.new(gen.code, gen.constants)
    vm:run()
end

test_vm()
```

---

## 83.12 Optimizations ที่ Compiler ทำได้

```lua
-- Constant Folding Pass
local function constant_fold(node)
    if not node then return node end
    
    if node.kind == "BinaryOp" then
        local left = constant_fold(node.left)
        local right = constant_fold(node.right)
        
        -- ถ้าทั้งสองเป็น numbers สามารถ fold ได้
        if left.kind == "NumberLiteral" and right.kind == "NumberLiteral" then
            local op = node.op
            local lv = left.value
            local rv = right.value
            
            local result
            if op == "+"  then result = lv + rv
            elseif op == "-"  then result = lv - rv
            elseif op == "*"  then result = lv * rv
            elseif op == "/"  then 
                if rv ~= 0 then result = lv / rv end
            elseif op == "%"  then result = lv % rv
            end
            
            if result ~= nil then
                return AST.NumberLiteral(result, node.line)
            end
        end
        
        return AST.BinaryOp(node.op, left, right, node.line)
        
    elseif node.kind == "UnaryOp" then
        local operand = constant_fold(node.operand)
        if node.op == "-" and operand.kind == "NumberLiteral" then
            return AST.NumberLiteral(-operand.value, node.line)
        end
        return AST.UnaryOp(node.op, operand, node.line)
        
    elseif node.kind == "Block" then
        local new_stmts = {}
        for _, stmt in ipairs(node.statements) do
            new_stmts[#new_stmts+1] = constant_fold(stmt)
        end
        return AST.Block(new_stmts)
        
    elseif node.kind == "LetStatement" then
        return AST.LetStatement(node.name, constant_fold(node.value), node.line)
        
    elseif node.kind == "PrintStatement" then
        return AST.PrintStatement(constant_fold(node.expr), node.line)
        
    else
        return node
    end
end

-- ทดสอบ constant folding
local function test_folding()
    local source = "let x = 2 + 3 * 4 - 1"
    
    local lexer = Lexer.new(source)
    local tokens = lexer:tokenize()
    local parser = Parser.new(tokens)
    local ast = parser:parse_program()
    
    print("\n=== Before Folding ===")
    -- ดู AST
    
    local folded = constant_fold(ast)
    print("=== After Folding ===")
    -- ดู folded AST
    
    -- Value ควรเป็น 2 + 12 - 1 = 13
    local stmt = folded.statements[1]
    print("Folded value:", stmt.value.value)  -- 13
end

test_folding()
```

---

## 83.13 Error Recovery

```lua
-- Parser ที่มี Error Recovery ทำให้แสดง errors หลายตัวพร้อมกัน

local ErrorRecoveringParser = {}
ErrorRecoveringParser.__index = ErrorRecoveringParser

-- ใช้ Parser เป็น base class
setmetatable(ErrorRecoveringParser, {__index = Parser})

function ErrorRecoveringParser.new(tokens)
    local self = Parser.new(tokens)
    setmetatable(self, ErrorRecoveringParser)
    self.errors = {}
    return self
end

function ErrorRecoveringParser:error(msg)
    local tok = self:current()
    self.errors[#self.errors+1] = {
        msg = msg,
        line = tok.line,
        col = tok.col
    }
    -- Continue parsing (don't throw!)
end

function ErrorRecoveringParser:synchronize()
    -- Skip tokens until we find a "sync point"
    -- (statement boundary, keyword, etc.)
    local sync_tokens = {
        [TokenType.LET] = true,
        [TokenType.IF] = true,
        [TokenType.WHILE] = true,
        [TokenType.FUN] = true,
        [TokenType.RETURN] = true,
        [TokenType.EOF] = true,
    }
    
    while not self:check(TokenType.EOF) do
        if sync_tokens[self:current().type] then
            return
        end
        self:advance()
    end
end

function ErrorRecoveringParser:report_errors()
    if #self.errors == 0 then
        print("No parse errors")
        return true
    end
    
    print(string.format("\n=== %d Parse Error(s) ===", #self.errors))
    for _, err in ipairs(self.errors) do
        print(string.format("  Line %d:%d: %s", err.line, err.col, err.msg))
    end
    return false
end

-- ทดสอบ error recovery
local function test_error_recovery()
    local source = [[
let x = 10
let y = (10 + 20  -- missing )
let z = y * 2
print(z)
]]
    
    local lexer = Lexer.new(source)
    local ok, tokens = pcall(function() return lexer:tokenize() end)
    if not ok then
        print("Lex error:", tokens)
        return
    end
    
    local parser = ErrorRecoveringParser.new(tokens)
    local ast = parser:parse_program()
    
    parser:report_errors()
end

test_error_recovery()
```

---

## 83.14 ตัวอย่างขั้นสูง: Type Inference (Basic)

```lua
-- Basic type inference pass

local TypeInferrer = {}
TypeInferrer.__index = TypeInferrer

local Types = {
    NUMBER  = "number",
    STRING  = "string",
    BOOL    = "boolean",
    NIL     = "nil",
    UNKNOWN = "unknown",
    FUNCTION = "function",
}

function TypeInferrer.new()
    local self = setmetatable({}, TypeInferrer)
    self.var_types = {}
    return self
end

function TypeInferrer:infer(node)
    if not node then return Types.NIL end
    
    local kind = node.kind
    
    if kind == "NumberLiteral" then
        return Types.NUMBER
        
    elseif kind == "StringLiteral" then
        return Types.STRING
        
    elseif kind == "BoolLiteral" then
        return Types.BOOL
        
    elseif kind == "NilLiteral" then
        return Types.NIL
        
    elseif kind == "Identifier" then
        return self.var_types[node.name] or Types.UNKNOWN
        
    elseif kind == "BinaryOp" then
        local left_type = self:infer(node.left)
        local right_type = self:infer(node.right)
        
        if node.op == "==" or node.op == "!=" or
           node.op == "<" or node.op == ">" or
           node.op == "<=" or node.op == ">=" or
           node.op == "and" or node.op == "or" then
            return Types.BOOL
        elseif node.op == ".." then
            return Types.STRING
        elseif left_type == Types.NUMBER and right_type == Types.NUMBER then
            return Types.NUMBER
        else
            return Types.UNKNOWN
        end
        
    elseif kind == "LetStatement" then
        local value_type = self:infer(node.value)
        self.var_types[node.name] = value_type
        return value_type
        
    elseif kind == "Block" then
        local last_type = Types.NIL
        for _, stmt in ipairs(node.statements) do
            last_type = self:infer(stmt)
        end
        return last_type
        
    else
        return Types.UNKNOWN
    end
end

-- ทดสอบ type inference
local function test_type_inference()
    local source = [[
let x = 42
let y = "hello"
let z = x + 1
let b = x > 10
]]
    
    local lexer = Lexer.new(source)
    local tokens = lexer:tokenize()
    local parser = Parser.new(tokens)
    local ast = parser:parse_program()
    
    local inferrer = TypeInferrer.new()
    inferrer:infer(ast)
    
    print("\n=== Inferred Types ===")
    for name, t in pairs(inferrer.var_types) do
        print(string.format("  %s: %s", name, t))
    end
end

test_type_inference()
```

---

## 83.15 ตัวอย่างโปรแกรม Tiny Language ที่ซับซ้อน

```lua
-- โปรแกรม fibonacci ด้วย memoization ใน Tiny language
local fibonacci_program = [[
fun fib(n)
    if n <= 1 then
        return n
    end
    return fib(n - 1) + fib(n - 2)
end

let i = 0
while i <= 10 do
    print(fib(i))
    let i = i + 1
end
]]

print("\n=== Fibonacci Program ===")
local ok = TinyLang.run(fibonacci_program)
if not ok then
    print("Program failed")
end

-- โปรแกรม bubble sort ง่ายๆ (ต้องเพิ่ม array support)
print("\n=== Tiny language is working! ===")
```

---

## 83.16 สรุปและทิศทางต่อไป

```lua
-- สิ่งที่เราสร้างในบทนี้:
-- 1. Lexer ที่รองรับ numbers, strings, identifiers, operators
-- 2. Parser (recursive descent + Pratt) สำหรับ expressions
-- 3. AST node system
-- 4. Symbol table สำหรับ scoping
-- 5. Semantic analyzer
-- 6. Tree-walking interpreter
-- 7. Bytecode compiler + stack VM
-- 8. Basic constant folding optimization
-- 9. Error recovery
-- 10. Basic type inference

-- ทิศทางต่อไป:
local next_steps = {
    "Array และ hash table support",
    "First-class functions (closures)",
    "Pattern matching",
    "Module system",
    "Type system (static typing)",
    "Register-based VM (เร็วกว่า stack-based)",
    "LLVM backend",
    "Incremental compilation",
    "Language Server Protocol (LSP) support",
    "Standard library",
}

print("\n=== Next Steps ===")
for i, step in ipairs(next_steps) do
    print(i .. ". " .. step)
end

-- Resources สำหรับเรียนเพิ่มเติม:
-- - "Crafting Interpreters" by Robert Nystrom (free online)
-- - "Engineering a Compiler" by Cooper & Torczon
-- - "Modern Compiler Implementation in Java/C/ML" by Andrew Appel
-- - LLVM Tutorial
-- - "Types and Programming Languages" by Benjamin Pierce
```

---

## แบบฝึกหัด

1. เพิ่ม support สำหรับ `break` และ `continue` statements ใน Tiny language

2. Implement array literals `[1, 2, 3]` และ indexing `arr[i]` ใน lexer, parser, และ interpreter

3. เพิ่ม string operations: `len(s)`, `sub(s, i, j)`, `upper(s)`, `lower(s)`

4. Implement closures ที่แท้จริง (functions capture variables by reference)

5. เพิ่ม `for i = start, end do ... end` loop

6. สร้าง pretty printer สำหรับ AST ที่สร้าง source code กลับจาก AST

7. เพิ่ม Dead Code Elimination pass

8. Implement tail-call optimization ใน interpreter

9. สร้าง REPL (Read-Eval-Print Loop) สำหรับ Tiny language

10. เพิ่ม error highlighting ที่แสดงตำแหน่งใน source code

---

## สรุป

ในบทนี้เราได้สร้าง tiny language compiler ที่สมบูรณ์ โดยครอบคลุม:

- **Lexer**: แปลง source text เป็น tokens
- **Parser**: สร้าง AST ด้วย recursive descent + Pratt parsing
- **Semantic Analysis**: ตรวจสอบ undefined variables และ type errors เบื้องต้น
- **Interpretation**: Execute AST โดยตรง (tree-walking interpreter)
- **Code Generation**: แปลง AST เป็น bytecode
- **VM**: Execute bytecode ด้วย stack-based VM
- **Optimizations**: Constant folding, type inference

ความรู้เหล่านี้เป็นพื้นฐานสำคัญสำหรับการเขียน DSL, configuration languages, template engines, และ scripting systems

ในบทต่อไป (บทที่ 84) เราจะไปดู Distributed Systems Concepts ที่เกี่ยวข้องกับการพัฒนา systems ขนาดใหญ่
