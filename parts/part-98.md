# บทที่ 98: Real Project 2 - Multiplayer Game Server

## บทนำ

บทนี้สร้าง Multiplayer Game Server สำหรับเกม Dungeon Explorer แบบ Real-time ด้วย Lua + TCP Sockets

---

## 98.1 Game Architecture

```
Architecture:
┌─────────────────────────────────────┐
│         Game Server (Lua)            │
│  ┌────────────────────────────────┐  │
│  │  Network Layer (TCP)           │  │
│  │  - Accept connections          │  │
│  │  - Binary protocol parsing     │  │
│  │  - Connection management       │  │
│  └──────────────┬─────────────────┘  │
│                 │                    │
│  ┌──────────────▼─────────────────┐  │
│  │  Game Engine                   │  │
│  │  - World state                 │  │
│  │  - Physics (simple AABB)       │  │
│  │  - Combat system               │  │
│  │  - ECS (Entity Component)      │  │
│  └──────────────┬─────────────────┘  │
│                 │                    │
│  ┌──────────────▼─────────────────┐  │
│  │  Persistence (Redis/SQLite)     │  │
│  │  - Player data                 │  │
│  │  - World state                 │  │
│  │  - Leaderboard                 │  │
│  └────────────────────────────────┘  │
└─────────────────────────────────────┘
       ↑           ↑          ↑
   Player1     Player2    Player3
```

---

## 98.2 Protocol Design

```lua
-- Binary protocol definition
-- Header: 4 bytes (2 bytes: message type, 2 bytes: payload length)
-- Body: variable length

local Protocol = {}

-- Message types
Protocol.MSG = {
    -- Client -> Server
    LOGIN        = 0x01,
    MOVE         = 0x02,
    ATTACK       = 0x03,
    USE_ITEM     = 0x04,
    CHAT         = 0x05,
    PING         = 0x06,
    
    -- Server -> Client
    LOGIN_OK     = 0x81,
    LOGIN_FAIL   = 0x82,
    WORLD_STATE  = 0x83,
    PLAYER_UPDATE = 0x84,
    PLAYER_LEAVE  = 0x85,
    COMBAT       = 0x86,
    CHAT_MSG     = 0x87,
    PONG         = 0x88,
    ERROR        = 0xFF
}

-- Encode message to binary
function Protocol.encode(msg_type, payload)
    local payload_str = payload or ""
    local header = string.pack(">I2I2", msg_type, #payload_str)
    return header .. payload_str
end

-- Decode message from binary
function Protocol.decode(data)
    if #data < 4 then return nil, "Incomplete header" end
    
    local msg_type, length = string.unpack(">I2I2", data)
    
    if #data < 4 + length then
        return nil, "Incomplete payload"
    end
    
    local payload = data:sub(5, 4 + length)
    local remaining = data:sub(5 + length)
    
    return { type = msg_type, payload = payload }, remaining
end

-- Pack player position
function Protocol.pack_position(x, y, facing)
    return string.pack(">fff B", x, y, 0, facing)
end

-- Unpack player position
function Protocol.unpack_position(data)
    local x, y, z, facing = string.unpack(">fff B", data)
    return x, y, facing
end

-- Pack player stats
function Protocol.pack_stats(stats)
    return string.pack(">I2 I2 I2 I2 I1",
        stats.hp, stats.max_hp,
        stats.mp, stats.max_mp,
        stats.level)
end

-- Unpack player stats
function Protocol.unpack_stats(data)
    local hp, max_hp, mp, max_mp, level = string.unpack(">I2 I2 I2 I2 I1", data)
    return { hp=hp, max_hp=max_hp, mp=mp, max_mp=max_mp, level=level }
end

-- Chat message packing
function Protocol.pack_chat(sender, message)
    local sender_bytes = sender:sub(1, 20)  -- max 20 char name
    return string.pack(">I1", #sender_bytes) .. sender_bytes .. message
end

function Protocol.unpack_chat(data)
    local name_len = string.unpack(">I1", data)
    local sender = data:sub(2, 1 + name_len)
    local message = data:sub(2 + name_len)
    return sender, message
end

return Protocol
```

---

## 98.3 Entity Component System (ECS)

```lua
-- ecs.lua - Entity Component System

local ECS = {}
ECS.__index = ECS

function ECS.new()
    return setmetatable({
        entities = {},      -- entity_id -> { comp_type -> component }
        next_id = 1,
        systems = {},       -- ordered list of systems
        component_index = {} -- comp_type -> { entity_id -> component }
    }, ECS)
end

-- Entity management
function ECS:create_entity()
    local id = self.next_id
    self.next_id = self.next_id + 1
    self.entities[id] = {}
    return id
end

function ECS:destroy_entity(id)
    local comps = self.entities[id]
    if not comps then return end
    
    for comp_type in pairs(comps) do
        if self.component_index[comp_type] then
            self.component_index[comp_type][id] = nil
        end
    end
    self.entities[id] = nil
end

-- Component management
function ECS:add_component(entity_id, comp_type, data)
    if not self.entities[entity_id] then return end
    
    self.entities[entity_id][comp_type] = data
    
    if not self.component_index[comp_type] then
        self.component_index[comp_type] = {}
    end
    self.component_index[comp_type][entity_id] = data
    
    return data
end

function ECS:get_component(entity_id, comp_type)
    local e = self.entities[entity_id]
    if not e then return nil end
    return e[comp_type]
end

function ECS:remove_component(entity_id, comp_type)
    local e = self.entities[entity_id]
    if not e then return end
    
    e[comp_type] = nil
    if self.component_index[comp_type] then
        self.component_index[comp_type][entity_id] = nil
    end
end

-- Query entities with specific components
function ECS:query(...)
    local comp_types = { ... }
    if #comp_types == 0 then return {} end
    
    -- Start with smallest component set
    local smallest_type = comp_types[1]
    local smallest_count = math.huge
    
    for _, ct in ipairs(comp_types) do
        local idx = self.component_index[ct]
        if not idx then return {} end
        local count = 0
        for _ in pairs(idx) do count = count + 1 end
        if count < smallest_count then
            smallest_count = count
            smallest_type = ct
        end
    end
    
    local results = {}
    local base_index = self.component_index[smallest_type]
    if not base_index then return {} end
    
    for entity_id in pairs(base_index) do
        local has_all = true
        for _, ct in ipairs(comp_types) do
            if not self.entities[entity_id] or 
               not self.entities[entity_id][ct] then
                has_all = false
                break
            end
        end
        if has_all then
            table.insert(results, entity_id)
        end
    end
    
    return results
end

-- System registration
function ECS:add_system(system)
    table.insert(self.systems, system)
end

-- Update all systems
function ECS:update(dt)
    for _, system in ipairs(self.systems) do
        if system.update then
            system:update(self, dt)
        end
    end
end

return ECS
```

---

## 98.4 Game World

```lua
-- world.lua - Game World Management

local ECS = require("ecs")
local Protocol = require("protocol")

local World = {}
World.__index = World

-- Component types
local COMP = {
    POSITION  = "position",
    VELOCITY  = "velocity",
    STATS     = "stats",
    PLAYER    = "player",
    MONSTER   = "monster",
    ITEM      = "item",
    COLLIDABLE = "collidable",
    COMBAT    = "combat"
}

function World.new()
    local w = setmetatable({
        ecs = ECS.new(),
        map = nil,
        players = {},    -- username -> entity_id
        tick = 0,
        tick_rate = 20   -- 20 ticks per second
    }, World)
    
    w:init_map()
    w:setup_systems()
    w:spawn_monsters()
    
    return w
end

function World:init_map()
    -- Simple 50x50 grid map
    -- 0 = floor, 1 = wall
    self.map = {}
    local W, H = 50, 50
    
    for y = 1, H do
        self.map[y] = {}
        for x = 1, W do
            -- Border walls
            if x == 1 or x == W or y == 1 or y == H then
                self.map[y][x] = 1
            else
                self.map[y][x] = 0
            end
        end
    end
    
    -- Add some walls
    for i = 10, 20 do
        self.map[15][i] = 1
    end
    for i = 5, 15 do
        self.map[i][25] = 1
    end
    
    self.map_width = W
    self.map_height = H
end

function World:is_walkable(x, y)
    local gx = math.floor(x)
    local gy = math.floor(y)
    if gx < 1 or gx > self.map_width or gy < 1 or gy > self.map_height then
        return false
    end
    return self.map[gy][gx] == 0
end

function World:setup_systems()
    -- Movement system
    self.ecs:add_system({
        update = function(system, ecs, dt)
            local entities = ecs:query(COMP.POSITION, COMP.VELOCITY)
            for _, eid in ipairs(entities) do
                local pos = ecs:get_component(eid, COMP.POSITION)
                local vel = ecs:get_component(eid, COMP.VELOCITY)
                
                local new_x = pos.x + vel.vx * dt
                local new_y = pos.y + vel.vy * dt
                
                -- Collision check
                if self:is_walkable(new_x, pos.y) then
                    pos.x = new_x
                end
                if self:is_walkable(pos.x, new_y) then
                    pos.y = new_y
                end
            end
        end
    })
    
    -- Combat system
    self.ecs:add_system({
        update = function(system, ecs, dt)
            local combatants = ecs:query(COMP.COMBAT, COMP.STATS)
            for _, eid in ipairs(combatants) do
                local combat = ecs:get_component(eid, COMP.COMBAT)
                local stats = ecs:get_component(eid, COMP.STATS)
                
                combat.attack_cooldown = (combat.attack_cooldown or 0) - dt
                
                if combat.attack_cooldown <= 0 then
                    combat.attack_cooldown = combat.attack_speed
                    
                    -- Check for nearby enemies
                    local pos = ecs:get_component(eid, COMP.POSITION)
                    if pos then
                        local is_player = ecs:get_component(eid, COMP.PLAYER)
                        local target_comp = is_player and COMP.MONSTER or COMP.PLAYER
                        
                        local targets = ecs:query(COMP.POSITION, target_comp, COMP.STATS)
                        for _, tid in ipairs(targets) do
                            local tpos = ecs:get_component(tid, COMP.POSITION)
                            local dist = math.sqrt((tpos.x - pos.x)^2 + (tpos.y - pos.y)^2)
                            
                            if dist <= combat.attack_range then
                                local tstats = ecs:get_component(tid, COMP.STATS)
                                local damage = combat.attack_damage - (tstats.defense or 0)
                                damage = math.max(1, damage)
                                tstats.hp = tstats.hp - damage
                                
                                if tstats.hp <= 0 then
                                    self:on_entity_death(tid, eid)
                                end
                                
                                break  -- Only attack first target
                            end
                        end
                    end
                end
            end
        end
    })
    
    -- Monster AI system
    self.ecs:add_system({
        update = function(system, ecs, dt)
            local monsters = ecs:query(COMP.MONSTER, COMP.POSITION, COMP.VELOCITY)
            for _, mid in ipairs(monsters) do
                local mpos = ecs:get_component(mid, COMP.POSITION)
                local mvel = ecs:get_component(mid, COMP.VELOCITY)
                local monster = ecs:get_component(mid, COMP.MONSTER)
                
                -- Find nearest player
                local nearest_player = nil
                local nearest_dist = math.huge
                
                local players = ecs:query(COMP.PLAYER, COMP.POSITION)
                for _, pid in ipairs(players) do
                    local ppos = ecs:get_component(pid, COMP.POSITION)
                    local dist = math.sqrt((ppos.x - mpos.x)^2 + (ppos.y - mpos.y)^2)
                    if dist < nearest_dist then
                        nearest_dist = dist
                        nearest_player = pid
                    end
                end
                
                if nearest_player and nearest_dist < (monster.aggro_range or 10) then
                    -- Chase player
                    local ppos = ecs:get_component(nearest_player, COMP.POSITION)
                    local dx = ppos.x - mpos.x
                    local dy = ppos.y - mpos.y
                    local len = math.sqrt(dx*dx + dy*dy)
                    if len > 0.1 then
                        mvel.vx = (dx / len) * (monster.speed or 2)
                        mvel.vy = (dy / len) * (monster.speed or 2)
                    end
                else
                    -- Random wander
                    monster.wander_timer = (monster.wander_timer or 0) - dt
                    if monster.wander_timer <= 0 then
                        local angle = math.random() * math.pi * 2
                        mvel.vx = math.cos(angle) * (monster.speed or 1) * 0.5
                        mvel.vy = math.sin(angle) * (monster.speed or 1) * 0.5
                        monster.wander_timer = 2 + math.random() * 3
                    end
                end
            end
        end
    })
end

function World:spawn_monsters()
    math.randomseed(os.time())
    
    local monster_types = {
        { name="Goblin",  hp=30,  attack=5,  defense=2, speed=3, aggro=8, reward=10 },
        { name="Orc",     hp=60,  attack=10, defense=5, speed=2, aggro=10, reward=25 },
        { name="Dragon",  hp=200, attack=25, defense=10, speed=4, aggro=15, reward=100 }
    }
    
    for i = 1, 20 do  -- Spawn 20 monsters
        local mt = monster_types[math.random(#monster_types)]
        local mid = self.ecs:create_entity()
        
        -- Find open spawn point
        local x, y
        repeat
            x = math.random(2, self.map_width - 1)
            y = math.random(2, self.map_height - 1)
        until self:is_walkable(x, y)
        
        self.ecs:add_component(mid, COMP.POSITION, { x=x, y=y })
        self.ecs:add_component(mid, COMP.VELOCITY, { vx=0, vy=0 })
        self.ecs:add_component(mid, COMP.STATS, {
            hp = mt.hp, max_hp = mt.hp, level = 1, defense = mt.defense
        })
        self.ecs:add_component(mid, COMP.MONSTER, {
            name = mt.name, speed = mt.speed, aggro_range = mt.aggro, reward = mt.reward
        })
        self.ecs:add_component(mid, COMP.COMBAT, {
            attack_damage = mt.attack, attack_speed = 1.5, attack_range = 1.5,
            attack_cooldown = math.random()
        })
    end
end

function World:add_player(username)
    local pid = self.ecs:create_entity()
    
    -- Spawn in safe area
    local x, y = 5, 5
    while not self:is_walkable(x, y) do
        x = x + 1
    end
    
    self.ecs:add_component(pid, COMP.POSITION, { x=x, y=y })
    self.ecs:add_component(pid, COMP.VELOCITY, { vx=0, vy=0 })
    self.ecs:add_component(pid, COMP.STATS, {
        hp=100, max_hp=100, mp=50, max_mp=50, level=1, defense=3
    })
    self.ecs:add_component(pid, COMP.PLAYER, { username=username, score=0 })
    self.ecs:add_component(pid, COMP.COMBAT, {
        attack_damage=15, attack_speed=1.0, attack_range=2.0, attack_cooldown=0
    })
    
    self.players[username] = pid
    return pid
end

function World:remove_player(username)
    local pid = self.players[username]
    if pid then
        self.ecs:destroy_entity(pid)
        self.players[username] = nil
    end
end

function World:move_player(username, direction)
    local pid = self.players[username]
    if not pid then return end
    
    local vel = self.ecs:get_component(pid, COMP.VELOCITY)
    if not vel then return end
    
    local speed = 5.0
    vel.vx, vel.vy = 0, 0
    
    if direction == "up"    then vel.vy = -speed
    elseif direction == "down"  then vel.vy =  speed
    elseif direction == "left"  then vel.vx = -speed
    elseif direction == "right" then vel.vx =  speed
    end
end

function World:on_entity_death(entity_id, killer_id)
    local is_monster = self.ecs:get_component(entity_id, COMP.MONSTER)
    
    if is_monster then
        -- Give reward to killer
        local killer_player = self.ecs:get_component(killer_id, COMP.PLAYER)
        if killer_player then
            killer_player.score = killer_player.score + is_monster.reward
        end
        
        -- Respawn monster later
        -- (simplified: just destroy)
        self.ecs:destroy_entity(entity_id)
    else
        -- Player died - respawn at start
        local pos = self.ecs:get_component(entity_id, COMP.POSITION)
        local stats = self.ecs:get_component(entity_id, COMP.STATS)
        if pos then pos.x, pos.y = 5, 5 end
        if stats then stats.hp = stats.max_hp end
    end
end

function World:get_state_snapshot()
    local snapshot = { players = {}, monsters = {} }
    
    -- Players
    for username, pid in pairs(self.players) do
        local pos = self.ecs:get_component(pid, COMP.POSITION)
        local stats = self.ecs:get_component(pid, COMP.STATS)
        local player = self.ecs:get_component(pid, COMP.PLAYER)
        
        if pos and stats then
            table.insert(snapshot.players, {
                id = pid, username = username,
                x = pos.x, y = pos.y,
                hp = stats.hp, max_hp = stats.max_hp,
                level = stats.level, score = player.score
            })
        end
    end
    
    -- Monsters (visible range)
    local monsters = self.ecs:query("monster", "position", "stats")
    for _, mid in ipairs(monsters) do
        local pos = self.ecs:get_component(mid, "position")
        local stats = self.ecs:get_component(mid, "stats")
        local monster = self.ecs:get_component(mid, "monster")
        
        table.insert(snapshot.monsters, {
            id = mid, name = monster.name,
            x = pos.x, y = pos.y,
            hp = stats.hp, max_hp = stats.max_hp
        })
    end
    
    return snapshot
end

function World:update(dt)
    self.tick = self.tick + 1
    self.ecs:update(dt)
end

return World
```

---

## 98.5 Server Main Loop

```lua
-- server.lua - Game Server Entry Point
local socket = require("socket")
local World = require("world")
local Protocol = require("protocol")
local json = require("dkjson")

local Server = {}
Server.__index = Server

function Server.new(host, port)
    local server = setmetatable({
        host = host or "0.0.0.0",
        port = port or 8888,
        world = World.new(),
        clients = {},    -- socket -> { username, buffer }
        running = false,
        tick_rate = 20,  -- 20 ticks per second
    }, Server)
    return server
end

function Server:start()
    local tcp = socket.tcp()
    tcp:setoption("reuseaddr", true)
    tcp:bind(self.host, self.port)
    tcp:listen(128)
    tcp:settimeout(0)  -- non-blocking
    
    self.server_socket = tcp
    self.running = true
    
    print(string.format("Game server started on %s:%d", self.host, self.port))
    
    local last_tick = socket.gettime()
    local tick_interval = 1.0 / self.tick_rate
    
    while self.running do
        -- Accept new connections
        local client, err = tcp:accept()
        if client then
            client:settimeout(0)
            self.clients[client] = { 
                socket = client, 
                buffer = "", 
                username = nil,
                authenticated = false
            }
            print("New connection from:", client:getpeername())
        end
        
        -- Read from all clients
        for sock, info in pairs(self.clients) do
            local data, err, partial = sock:receive(4096)
            local received = data or partial
            
            if received and #received > 0 then
                info.buffer = info.buffer .. received
                self:process_buffer(sock, info)
            end
            
            if err == "closed" then
                self:disconnect_client(sock, info)
            end
        end
        
        -- Game tick
        local now = socket.gettime()
        local dt = now - last_tick
        
        if dt >= tick_interval then
            self.world:update(dt)
            self:broadcast_state()
            last_tick = now
        end
        
        -- Small sleep to avoid 100% CPU
        socket.sleep(0.001)
    end
end

function Server:process_buffer(sock, info)
    while #info.buffer >= 4 do
        local msg, remaining = Protocol.decode(info.buffer)
        if not msg then break end
        
        info.buffer = remaining
        self:handle_message(sock, info, msg)
    end
end

function Server:handle_message(sock, info, msg)
    local MT = Protocol.MSG
    
    if msg.type == MT.LOGIN then
        -- Extract username (first 20 bytes max)
        local username = msg.payload:sub(1, 20):gsub("\0", "")
        
        if self.world.players[username] then
            sock:send(Protocol.encode(MT.LOGIN_FAIL, "Username taken"))
            return
        end
        
        info.username = username
        info.authenticated = true
        
        self.world:add_player(username)
        
        sock:send(Protocol.encode(MT.LOGIN_OK, username))
        
        print("Player logged in:", username)
        
    elseif not info.authenticated then
        sock:send(Protocol.encode(MT.ERROR, "Not authenticated"))
        return
        
    elseif msg.type == MT.MOVE then
        local directions = { "up", "down", "left", "right", "stop" }
        local dir_idx = string.byte(msg.payload, 1) or 5
        local direction = directions[dir_idx] or "stop"
        self.world:move_player(info.username, direction)
        
    elseif msg.type == MT.CHAT then
        local chat_msg = msg.payload
        self:broadcast_to_all(Protocol.encode(MT.CHAT_MSG,
            Protocol.pack_chat(info.username, chat_msg)))
        
    elseif msg.type == MT.PING then
        sock:send(Protocol.encode(MT.PONG, ""))
    end
end

function Server:broadcast_state()
    local snapshot = self.world:get_state_snapshot()
    local state_json = json.encode(snapshot)
    local state_msg = Protocol.encode(Protocol.MSG.WORLD_STATE, state_json)
    
    for sock, info in pairs(self.clients) do
        if info.authenticated then
            local ok, err = sock:send(state_msg)
            if not ok then
                -- Will be cleaned up on next receive error
            end
        end
    end
end

function Server:broadcast_to_all(data)
    for sock, info in pairs(self.clients) do
        if info.authenticated then
            sock:send(data)
        end
    end
end

function Server:disconnect_client(sock, info)
    if info.username then
        self.world:remove_player(info.username)
        print("Player disconnected:", info.username)
    end
    self.clients[sock] = nil
    sock:close()
end

-- Start server
local server = Server.new("0.0.0.0", 8888)
server:start()
```

---

## 98.6 Leaderboard System

```lua
-- leaderboard.lua
local M = {}

-- In-memory leaderboard (use Redis in production)
local scores = {}

function M.update_score(username, score)
    scores[username] = score
end

function M.get_top(n)
    local sorted = {}
    for username, score in pairs(scores) do
        table.insert(sorted, { username=username, score=score })
    end
    
    table.sort(sorted, function(a, b)
        return a.score > b.score
    end)
    
    local top = {}
    for i = 1, math.min(n, #sorted) do
        table.insert(top, {
            rank = i,
            username = sorted[i].username,
            score = sorted[i].score
        })
    end
    return top
end

function M.print_top(n)
    print("\n=== Leaderboard (Top " .. n .. ") ===")
    for _, entry in ipairs(M.get_top(n)) do
        print(string.format("#%d  %-20s  %d pts",
            entry.rank, entry.username, entry.score))
    end
end

return M
```

---

## แบบฝึกหัด

1. **Inventory System**: เพิ่ม item collection และ inventory management
2. **Map Generation**: Procedural dungeon generation algorithm
3. **Spell System**: Magic spells ด้วย cooldown และ mana cost
4. **Guild System**: สร้าง player guilds ที่ share rewards
5. **Anti-Cheat**: เพิ่ม server-side validation สำหรับ player movements

---

*ต่อไป: [Part 99 - Real Project: API Gateway](part-99.md)*
