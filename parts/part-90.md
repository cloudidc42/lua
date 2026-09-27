# บทที่ 90: Game Server Architecture

## บทนำ: Game Server Types

Game server มีหลายประเภท แต่ละประเภทมีวัตถุประสงค์และ trade-offs ที่แตกต่างกัน:

- **Authoritative Server**: server เป็นผู้ตัดสินใจขั้นสุดท้ายทั้งหมด ป้องกัน cheating
- **Relay Server**: ส่งต่อ messages ระหว่าง clients โดยไม่ validate state
- **Dedicated Server**: server แยกต่างหากที่ไม่มี player
- **Listen Server**: player หนึ่งคนทำหน้าที่เป็น server ด้วย

```lua
-- ตัวอย่างที่ 1: Game Server Base Class
local GameServer = {}
GameServer.__index = GameServer

GameServer.TYPE = {
    AUTHORITATIVE = "authoritative",
    RELAY = "relay",
    DEDICATED = "dedicated"
}

function GameServer.new(config)
    config = config or {}
    return setmetatable({
        server_type = config.type or GameServer.TYPE.AUTHORITATIVE,
        tick_rate = config.tick_rate or 64,  -- ticks per second
        max_players = config.max_players or 64,
        game_state = {
            tick = 0,
            entities = {},
            players = {}
        },
        pending_inputs = {},  -- inputs waiting to be processed
        snapshots = {},       -- game state snapshots
        config = config
    }, GameServer)
end

function GameServer:start()
    print(string.format("[Server] Starting %s server (tick_rate=%d, max_players=%d)",
                       self.server_type, self.tick_rate, self.max_players))
    self.running = true
    self:game_loop()
end

function GameServer:game_loop()
    local tick_duration = 1.0 / self.tick_rate  -- seconds per tick
    
    while self.running do
        local tick_start = os.clock()
        
        -- Process inputs
        self:process_inputs()
        
        -- Update game state
        self:update(tick_duration)
        
        -- Take snapshot
        self:take_snapshot()
        
        -- Broadcast state to clients
        self:broadcast_state()
        
        self.game_state.tick = self.game_state.tick + 1
        
        -- Sleep for remainder of tick (simulated)
        local tick_elapsed = os.clock() - tick_start
        local sleep_time = tick_duration - tick_elapsed
        if sleep_time < 0 then
            print(string.format("[WARNING] Tick %d took too long: %.2fms (budget: %.2fms)",
                               self.game_state.tick,
                               tick_elapsed * 1000,
                               tick_duration * 1000))
        end
        
        -- In real implementation: actual sleep/timer here
        break  -- For demonstration, just run once
    end
end

function GameServer:process_inputs()
    -- Sort inputs by tick number
    table.sort(self.pending_inputs, function(a, b)
        return a.client_tick < b.client_tick
    end)
    
    for _, input in ipairs(self.pending_inputs) do
        self:apply_input(input)
    end
    self.pending_inputs = {}
end

function GameServer:apply_input(input)
    local player = self.game_state.players[input.player_id]
    if not player then return end
    
    -- Validate input (anti-cheat)
    if not self:validate_input(input, player) then
        print(string.format("[AntiCheat] Invalid input from player %s", input.player_id))
        return
    end
    
    -- Apply movement
    if input.move_x then
        player.x = player.x + input.move_x * player.speed
    end
    if input.move_y then
        player.y = player.y + input.move_y * player.speed
    end
    
    -- Clamp to world bounds
    player.x = math.max(0, math.min(1000, player.x))
    player.y = math.max(0, math.min(1000, player.y))
end

function GameServer:validate_input(input, player)
    -- Check movement speed doesn't exceed max
    local max_speed = player.speed or 10
    local dx = math.abs(input.move_x or 0)
    local dy = math.abs(input.move_y or 0)
    
    if dx > max_speed + 1 or dy > max_speed + 1 then
        return false
    end
    
    -- Check input rate (anti-spam)
    if input.timestamp and player.last_input_time then
        local min_interval = 1.0 / self.tick_rate * 0.8  -- 80% of tick time
        if input.timestamp - player.last_input_time < min_interval then
            return false
        end
    end
    
    return true
end

function GameServer:update(dt)
    -- Update all entities
    for id, entity in pairs(self.game_state.entities) do
        if entity.update then
            entity:update(dt)
        end
    end
end

function GameServer:take_snapshot()
    -- Deep copy current state
    local snapshot = {
        tick = self.game_state.tick,
        timestamp = os.clock(),
        players = {}
    }
    
    for pid, player in pairs(self.game_state.players) do
        snapshot.players[pid] = {
            id = player.id,
            x = player.x,
            y = player.y,
            health = player.health
        }
    end
    
    -- Keep last 64 snapshots (1 second at 64 tick)
    table.insert(self.snapshots, snapshot)
    if #self.snapshots > 64 then
        table.remove(self.snapshots, 1)
    end
end

function GameServer:broadcast_state()
    -- In real implementation: serialize and send to all clients
    -- Here we just simulate
end
```

## State Synchronization

```lua
-- ตัวอย่างที่ 2: State Synchronization System
local StateSyncManager = {}
StateSyncManager.__index = StateSyncManager

function StateSyncManager.new(server)
    return setmetatable({
        server = server,
        client_acks = {},      -- last ack'd tick per client
        client_states = {},    -- last sent state per client
        dirty_entities = {}    -- entities that changed since last sync
    }, StateSyncManager)
end

function StateSyncManager:mark_dirty(entity_id)
    self.dirty_entities[entity_id] = true
end

function StateSyncManager:get_update_for_client(client_id)
    local last_ack = self.client_acks[client_id] or 0
    local current_tick = self.server.game_state.tick
    
    -- Find relevant snapshot
    local baseline_snapshot = nil
    for _, snap in ipairs(self.server.snapshots) do
        if snap.tick == last_ack then
            baseline_snapshot = snap
            break
        end
    end
    
    local update = {
        tick = current_tick,
        baseline_tick = last_ack,
        players = {}
    }
    
    -- Delta encode: only send changed fields
    for pid, player in pairs(self.server.game_state.players) do
        local player_delta = {}
        
        if baseline_snapshot then
            local old_player = baseline_snapshot.players[pid]
            if old_player then
                -- Only send changed values
                if player.x ~= old_player.x then
                    player_delta.x = player.x
                end
                if player.y ~= old_player.y then
                    player_delta.y = player.y
                end
                if player.health ~= old_player.health then
                    player_delta.health = player.health
                end
            else
                -- New player, send all
                player_delta = {x = player.x, y = player.y, health = player.health}
                player_delta.new = true
            end
        else
            -- No baseline, send all
            player_delta = {x = player.x, y = player.y, health = player.health}
        end
        
        if next(player_delta) then
            update.players[pid] = player_delta
        end
    end
    
    return update
end

function StateSyncManager:acknowledge(client_id, tick)
    self.client_acks[client_id] = math.max(
        self.client_acks[client_id] or 0,
        tick
    )
end
```

## Delta Compression

```lua
-- ตัวอย่างที่ 3: Delta Compression for Network Packets
local DeltaCompressor = {}
DeltaCompressor.__index = DeltaCompressor

function DeltaCompressor.new()
    return setmetatable({
        baseline = nil,
        compression_stats = {
            total_bytes = 0,
            compressed_bytes = 0,
            packets = 0
        }
    }, DeltaCompressor)
end

function DeltaCompressor:compress(current_state, baseline_state)
    if not baseline_state then
        -- Full state
        return {
            type = "full",
            data = current_state
        }
    end
    
    -- Compute delta
    local delta = {
        type = "delta",
        baseline_tick = baseline_state.tick,
        changes = {}
    }
    
    -- Compare players
    for pid, player in pairs(current_state.players) do
        local old_player = baseline_state.players[pid]
        
        if not old_player then
            -- New player
            delta.changes[pid] = {op = "add", data = player}
        else
            -- Find changed fields using quantization
            local changed = {}
            
            -- Quantize positions to reduce noise
            local qx = math.floor(player.x * 10) / 10
            local qy = math.floor(player.y * 10) / 10
            local old_qx = math.floor(old_player.x * 10) / 10
            local old_qy = math.floor(old_player.y * 10) / 10
            
            if qx ~= old_qx then changed.x = qx end
            if qy ~= old_qy then changed.y = qy end
            if player.health ~= old_player.health then
                changed.health = player.health
            end
            if player.rotation ~= old_player.rotation then
                -- Quantize rotation to 1-degree precision
                changed.rotation = math.floor(player.rotation)
            end
            
            if next(changed) then
                delta.changes[pid] = {op = "update", data = changed}
            end
        end
    end
    
    -- Check for removed players
    if baseline_state.players then
        for pid in pairs(baseline_state.players) do
            if not current_state.players[pid] then
                delta.changes[pid] = {op = "remove"}
            end
        end
    end
    
    return delta
end

function DeltaCompressor:decompress(delta, baseline_state)
    if delta.type == "full" then
        return delta.data
    end
    
    -- Apply delta to baseline
    local state = {
        tick = delta.tick or (baseline_state and baseline_state.tick + 1 or 0),
        players = {}
    }
    
    -- Copy baseline players
    if baseline_state then
        for pid, player in pairs(baseline_state.players) do
            state.players[pid] = {}
            for k, v in pairs(player) do
                state.players[pid][k] = v
            end
        end
    end
    
    -- Apply changes
    for pid, change in pairs(delta.changes) do
        if change.op == "add" then
            state.players[pid] = change.data
        elseif change.op == "update" then
            if not state.players[pid] then
                state.players[pid] = {}
            end
            for k, v in pairs(change.data) do
                state.players[pid][k] = v
            end
        elseif change.op == "remove" then
            state.players[pid] = nil
        end
    end
    
    return state
end

-- ตัวอย่างการใช้งาน Delta Compression
local compressor = DeltaCompressor.new()

local state1 = {
    tick = 100,
    players = {
        p1 = {x = 100.0, y = 200.0, health = 100, rotation = 45},
        p2 = {x = 300.0, y = 400.0, health = 80, rotation = 180}
    }
}

local state2 = {
    tick = 101,
    players = {
        p1 = {x = 105.3, y = 200.0, health = 100, rotation = 47},  -- p1 moved
        p2 = {x = 300.0, y = 400.0, health = 70, rotation = 180},  -- p2 took damage
        p3 = {x = 500.0, y = 500.0, health = 100, rotation = 0}    -- new player
    }
}

local delta = compressor:compress(state2, state1)
print("\n=== Delta Compression ===")
print("Full state players:", 2)
print("Delta changes:", (function()
    local n = 0
    for _ in pairs(delta.changes) do n = n + 1 end
    return n
end)())

-- Show what changed
for pid, change in pairs(delta.changes) do
    print(string.format("  Player %s: %s", pid, change.op))
    if change.data then
        for k, v in pairs(change.data) do
            print(string.format("    %s = %s", k, tostring(v)))
        end
    end
end

-- Reconstruct state
local reconstructed = compressor:decompress(delta, state1)
print("\nReconstructed state player count:", (function()
    local n = 0
    for _ in pairs(reconstructed.players) do n = n + 1 end
    return n
end)())
```

## Lag Compensation

```lua
-- ตัวอย่างที่ 4: Server-Side Lag Compensation
local LagCompensator = {}
LagCompensator.__index = LagCompensator

function LagCompensator.new(server)
    return setmetatable({
        server = server,
        history = {},  -- Historical positions per player
        max_history_ticks = 128,
        max_compensation_ms = 150  -- Max lag to compensate for
    }, LagCompensator)
end

function LagCompensator:record_positions(tick, players)
    self.history[tick] = {
        tick = tick,
        timestamp = os.clock(),
        positions = {}
    }
    
    for pid, player in pairs(players) do
        self.history[tick].positions[pid] = {
            x = player.x,
            y = player.y,
            bbox = player.bbox or {w = 30, h = 60}  -- hitbox
        }
    end
    
    -- Cleanup old history
    local oldest_kept = tick - self.max_history_ticks
    for t in pairs(self.history) do
        if t < oldest_kept then
            self.history[t] = nil
        end
    end
end

function LagCompensator:rewind_world(target_tick)
    -- Return the world state at a specific tick
    return self.history[target_tick]
end

function LagCompensator:compensated_hit_check(
    shooter_player_id, target_player_id, 
    ray_start, ray_dir, 
    shooter_latency_ms)
    
    -- Calculate which tick the shooter "saw" when they fired
    local current_tick = self.server.game_state.tick
    local ticks_of_lag = math.floor(
        (shooter_latency_ms / 1000) * self.server.tick_rate
    )
    local shoot_tick = current_tick - ticks_of_lag
    
    -- Clamp to available history
    shoot_tick = math.max(
        current_tick - self.max_history_ticks,
        math.min(current_tick, shoot_tick)
    )
    
    -- Get historical position of target
    local historical_state = self.history[shoot_tick]
    if not historical_state then
        -- No history, use current position
        historical_state = self.history[current_tick]
    end
    
    if not historical_state then
        return false, "No history available"
    end
    
    local target_pos = historical_state.positions[target_player_id]
    if not target_pos then
        return false, "Target not in historical state"
    end
    
    -- Check ray vs AABB (Axis-Aligned Bounding Box)
    local hit = self:ray_aabb_intersect(
        ray_start, ray_dir,
        target_pos, target_pos.bbox
    )
    
    if hit then
        print(string.format(
            "[LagComp] Hit confirmed: tick %d (compensated %d ticks, %dms lag)",
            shoot_tick, ticks_of_lag, shooter_latency_ms
        ))
    end
    
    return hit
end

function LagCompensator:ray_aabb_intersect(ray_start, ray_dir, box_center, box_size)
    -- Simple 2D ray vs AABB intersection
    local box_min = {
        x = box_center.x - box_size.w / 2,
        y = box_center.y - box_size.h / 2
    }
    local box_max = {
        x = box_center.x + box_size.w / 2,
        y = box_center.y + box_size.h / 2
    }
    
    -- Parametric ray intersection
    local t_min_x = (box_min.x - ray_start.x) / (ray_dir.x + 0.0001)
    local t_max_x = (box_max.x - ray_start.x) / (ray_dir.x + 0.0001)
    local t_min_y = (box_min.y - ray_start.y) / (ray_dir.y + 0.0001)
    local t_max_y = (box_max.y - ray_start.y) / (ray_dir.y + 0.0001)
    
    if t_min_x > t_max_x then t_min_x, t_max_x = t_max_x, t_min_x end
    if t_min_y > t_max_y then t_min_y, t_max_y = t_max_y, t_min_y end
    
    local t_enter = math.max(t_min_x, t_min_y)
    local t_exit = math.min(t_max_x, t_max_y)
    
    return t_enter <= t_exit and t_exit >= 0 and t_enter <= 1000
end

-- ตัวอย่างที่ 5: Client-Side Prediction
local ClientPredictor = {}
ClientPredictor.__index = ClientPredictor

function ClientPredictor.new()
    return setmetatable({
        pending_inputs = {},  -- inputs not yet confirmed by server
        predicted_state = nil,
        server_state = nil,
        input_sequence = 0
    }, ClientPredictor)
end

function ClientPredictor:apply_input(input)
    self.input_sequence = self.input_sequence + 1
    input.sequence = self.input_sequence
    
    -- Apply input to local prediction
    if self.predicted_state then
        self:simulate_input(self.predicted_state, input)
    end
    
    -- Store as pending (waiting for server confirmation)
    table.insert(self.pending_inputs, input)
    
    return input.sequence
end

function ClientPredictor:simulate_input(state, input)
    -- Simulate exact same physics as server
    local player = state.local_player
    if not player then return end
    
    if input.move_x then
        player.x = player.x + input.move_x * player.speed
    end
    if input.move_y then
        player.y = player.y + input.move_y * player.speed
    end
    
    player.x = math.max(0, math.min(1000, player.x))
    player.y = math.max(0, math.min(1000, player.y))
    
    player.last_input_sequence = input.sequence
end

function ClientPredictor:reconcile(server_update)
    -- Server sent authoritative state with last processed input sequence
    local server_seq = server_update.last_processed_sequence
    
    -- Remove inputs that server has processed
    local i = 1
    while i <= #self.pending_inputs do
        if self.pending_inputs[i].sequence <= server_seq then
            table.remove(self.pending_inputs, i)
        else
            i = i + 1
        end
    end
    
    -- Reset to server state
    self.predicted_state = {
        local_player = {
            x = server_update.player_x,
            y = server_update.player_y,
            speed = server_update.player_speed or 10
        }
    }
    
    -- Re-apply pending (unconfirmed) inputs
    for _, input in ipairs(self.pending_inputs) do
        self:simulate_input(self.predicted_state, input)
    end
    
    print(string.format("[Prediction] Reconciled at seq %d, %d inputs replayed",
                       server_seq, #self.pending_inputs))
end

function ClientPredictor:get_position()
    if not self.predicted_state then return nil end
    return self.predicted_state.local_player.x, self.predicted_state.local_player.y
end
```

## Server Reconciliation

```lua
-- ตัวอย่างที่ 6: Server Reconciliation with Error Correction
local ServerReconciler = {}
ServerReconciler.__index = ServerReconciler

function ServerReconciler.new(options)
    options = options or {}
    return setmetatable({
        correction_speed = options.correction_speed or 0.1,  -- lerp factor
        snap_threshold = options.snap_threshold or 50,  -- pixels
        corrections = 0,
        snaps = 0
    }, ServerReconciler)
end

function ServerReconciler:compute_correction(predicted_pos, server_pos)
    local dx = server_pos.x - predicted_pos.x
    local dy = server_pos.y - predicted_pos.y
    local distance = math.sqrt(dx*dx + dy*dy)
    
    if distance > self.snap_threshold then
        -- Too far off, snap immediately
        self.snaps = self.snaps + 1
        return {
            x = server_pos.x,
            y = server_pos.y,
            type = "snap",
            distance = distance
        }
    elseif distance > 0.1 then
        -- Smooth correction via lerp
        self.corrections = self.corrections + 1
        return {
            x = predicted_pos.x + dx * self.correction_speed,
            y = predicted_pos.y + dy * self.correction_speed,
            type = "lerp",
            distance = distance
        }
    end
    
    return {
        x = predicted_pos.x,
        y = predicted_pos.y,
        type = "none",
        distance = distance
    }
end

function ServerReconciler:report()
    print(string.format("Reconciler: %d corrections, %d snaps",
                       self.corrections, self.snaps))
end
```

## Lockstep Simulation

```lua
-- ตัวอย่างที่ 7: Deterministic Lockstep
local LockstepGame = {}
LockstepGame.__index = LockstepGame

function LockstepGame.new(player_count)
    return setmetatable({
        player_count = player_count,
        tick = 0,
        inputs = {},  -- inputs[tick][player_id]
        ready_players = {},
        state = {
            players = {},
            entities = {}
        },
        checksum_log = {}
    }, LockstepGame)
end

function LockstepGame:submit_input(player_id, tick, input)
    if not self.inputs[tick] then
        self.inputs[tick] = {}
    end
    self.inputs[tick][player_id] = input
    
    -- Check if all players have submitted for this tick
    if self:all_inputs_ready(tick) then
        return true  -- Ready to advance
    end
    return false
end

function LockstepGame:all_inputs_ready(tick)
    local inputs = self.inputs[tick]
    if not inputs then return false end
    
    local count = 0
    for _ in pairs(inputs) do count = count + 1 end
    return count >= self.player_count
end

function LockstepGame:advance()
    if not self:all_inputs_ready(self.tick) then
        return false, "Not all inputs received"
    end
    
    -- Process all inputs for this tick deterministically
    -- Sort by player_id to ensure deterministic order
    local inputs = self.inputs[self.tick]
    local sorted_players = {}
    for pid in pairs(inputs) do
        table.insert(sorted_players, pid)
    end
    table.sort(sorted_players)
    
    for _, pid in ipairs(sorted_players) do
        self:process_player_input(pid, inputs[pid])
    end
    
    -- Update simulation
    self:update_simulation()
    
    -- Compute and store checksum for desync detection
    local checksum = self:compute_state_checksum()
    self.checksum_log[self.tick] = checksum
    
    self.tick = self.tick + 1
    
    -- Clear old inputs
    self.inputs[self.tick - 10] = nil
    
    return true
end

function LockstepGame:process_player_input(player_id, input)
    local player = self.state.players[player_id]
    if not player then return end
    
    -- Deterministic movement (integer math preferred)
    if input.dx then
        player.x = player.x + math.floor(input.dx * 100) / 100
    end
    if input.dy then
        player.y = player.y + math.floor(input.dy * 100) / 100
    end
end

function LockstepGame:update_simulation()
    -- Deterministic physics/game logic
    for _, entity in pairs(self.state.entities) do
        if entity.vx then
            entity.x = entity.x + entity.vx
        end
        if entity.vy then
            entity.y = entity.y + entity.vy
        end
    end
end

function LockstepGame:compute_state_checksum()
    -- Simple checksum for desync detection
    local sum = 0
    
    for pid, player in pairs(self.state.players) do
        sum = sum + math.floor(player.x * 100) + math.floor(player.y * 100)
        -- Add player id contribution
        for i = 1, #pid do
            sum = sum + pid:byte(i)
        end
    end
    
    return sum % (2^32)
end

function LockstepGame:verify_checksum(tick, other_checksum)
    local our_checksum = self.checksum_log[tick]
    if not our_checksum then
        return true  -- Can't verify
    end
    
    if our_checksum ~= other_checksum then
        print(string.format("[DESYNC] Tick %d: our=%d, other=%d",
                           tick, our_checksum, other_checksum))
        return false
    end
    return true
end

-- ตัวอย่างการใช้งาน Lockstep
local game = LockstepGame.new(2)

game.state.players = {
    p1 = {x = 100, y = 100, speed = 5},
    p2 = {x = 400, y = 400, speed = 5}
}

-- Simulate 3 ticks
for tick = 0, 2 do
    game:submit_input("p1", tick, {dx = 5, dy = 0})
    game:submit_input("p2", tick, {dx = -3, dy = 2})
    
    local ok = game:advance()
    if ok then
        print(string.format("Tick %d: p1=(%.0f,%.0f) p2=(%.0f,%.0f) checksum=%d",
                           tick,
                           game.state.players.p1.x, game.state.players.p1.y,
                           game.state.players.p2.x, game.state.players.p2.y,
                           game.checksum_log[tick] or 0))
    end
end
```

## Area of Interest Management

```lua
-- ตัวอย่างที่ 8: Area of Interest (AoI) System
local AoIManager = {}
AoIManager.__index = AoIManager

function AoIManager.new(options)
    options = options or {}
    return setmetatable({
        world_width = options.world_width or 10000,
        world_height = options.world_height or 10000,
        cell_size = options.cell_size or 500,  -- Grid cell size
        player_radius = options.player_radius or 1000,  -- AoI radius
        grid = {},
        entity_cells = {}  -- Which cell each entity is in
    }, AoIManager)
end

function AoIManager:cell_coords(x, y)
    local cx = math.floor(x / self.cell_size) + 1
    local cy = math.floor(y / self.cell_size) + 1
    return cx, cy
end

function AoIManager:cell_key(cx, cy)
    return cx .. ":" .. cy
end

function AoIManager:update_entity(entity_id, x, y)
    -- Remove from old cell
    local old_cell = self.entity_cells[entity_id]
    if old_cell then
        local cell = self.grid[old_cell]
        if cell then
            cell[entity_id] = nil
        end
    end
    
    -- Add to new cell
    local cx, cy = self:cell_coords(x, y)
    local new_key = self:cell_key(cx, cy)
    
    if not self.grid[new_key] then
        self.grid[new_key] = {}
    end
    
    self.grid[new_key][entity_id] = {x = x, y = y, id = entity_id}
    self.entity_cells[entity_id] = new_key
end

function AoIManager:get_entities_in_range(x, y, radius)
    radius = radius or self.player_radius
    local result = {}
    
    -- Calculate which grid cells to check
    local min_cx = math.floor((x - radius) / self.cell_size)
    local max_cx = math.ceil((x + radius) / self.cell_size)
    local min_cy = math.floor((y - radius) / self.cell_size)
    local max_cy = math.ceil((y + radius) / self.cell_size)
    
    for cx = min_cx, max_cx do
        for cy = min_cy, max_cy do
            local key = self:cell_key(cx, cy)
            local cell = self.grid[key]
            if cell then
                for eid, entity in pairs(cell) do
                    -- Distance check
                    local dx = entity.x - x
                    local dy = entity.y - y
                    local dist = math.sqrt(dx*dx + dy*dy)
                    
                    if dist <= radius then
                        table.insert(result, {
                            id = eid,
                            x = entity.x,
                            y = entity.y,
                            distance = dist
                        })
                    end
                end
            end
        end
    end
    
    return result
end

function AoIManager:get_observer_sets(players)
    -- Return which players each player should receive updates about
    local observer_sets = {}
    
    for pid, player in pairs(players) do
        observer_sets[pid] = {}
        local nearby = self:get_entities_in_range(player.x, player.y)
        for _, entity in ipairs(nearby) do
            observer_sets[pid][entity.id] = entity
        end
    end
    
    return observer_sets
end

-- ตัวอย่างการใช้งาน AoI
local aoi = AoIManager.new({
    world_width = 10000,
    world_height = 10000,
    cell_size = 500,
    player_radius = 1500
})

-- Place some players
local players = {
    p1 = {x = 1000, y = 1000},
    p2 = {x = 1200, y = 1100},  -- Near p1
    p3 = {x = 5000, y = 5000},  -- Far from p1
    p4 = {x = 1500, y = 900}    -- Near p1
}

for pid, pos in pairs(players) do
    aoi:update_entity(pid, pos.x, pos.y)
end

print("\n=== Area of Interest ===")
local nearby_p1 = aoi:get_entities_in_range(players.p1.x, players.p1.y)
print("Players visible to p1:")
for _, entity in ipairs(nearby_p1) do
    print(string.format("  %s at (%.0f, %.0f) distance: %.0f",
                       entity.id, entity.x, entity.y, entity.distance))
end
```

## Spatial Hashing

```lua
-- ตัวอย่างที่ 9: Spatial Hash for Collision Detection
local SpatialHash = {}
SpatialHash.__index = SpatialHash

function SpatialHash.new(cell_size)
    return setmetatable({
        cell_size = cell_size or 100,
        cells = {},
        entity_count = 0
    }, SpatialHash)
end

function SpatialHash:hash(x, y)
    local cx = math.floor(x / self.cell_size)
    local cy = math.floor(y / self.cell_size)
    -- Cantor pairing function for unique hash
    return (cx + cy) * (cx + cy + 1) / 2 + cy
end

function SpatialHash:insert(entity)
    -- Entity has: id, x, y, radius
    local cells_covered = self:get_cells(entity)
    
    entity._spatial_cells = cells_covered
    
    for _, cell_hash in ipairs(cells_covered) do
        if not self.cells[cell_hash] then
            self.cells[cell_hash] = {}
        end
        self.cells[cell_hash][entity.id] = entity
    end
    
    self.entity_count = self.entity_count + 1
end

function SpatialHash:remove(entity)
    if not entity._spatial_cells then return end
    
    for _, cell_hash in ipairs(entity._spatial_cells) do
        if self.cells[cell_hash] then
            self.cells[cell_hash][entity.id] = nil
        end
    end
    
    entity._spatial_cells = nil
    self.entity_count = self.entity_count - 1
end

function SpatialHash:update(entity, new_x, new_y)
    self:remove(entity)
    entity.x = new_x
    entity.y = new_y
    self:insert(entity)
end

function SpatialHash:get_cells(entity)
    local cells = {}
    local r = entity.radius or 0
    
    local x_min = math.floor((entity.x - r) / self.cell_size)
    local x_max = math.floor((entity.x + r) / self.cell_size)
    local y_min = math.floor((entity.y - r) / self.cell_size)
    local y_max = math.floor((entity.y + r) / self.cell_size)
    
    for cx = x_min, x_max do
        for cy = y_min, y_max do
            local h = (cx + cy) * (cx + cy + 1) / 2 + cy
            table.insert(cells, h)
        end
    end
    
    return cells
end

function SpatialHash:get_nearby(entity)
    local nearby = {}
    local seen = {}
    
    local cells = self:get_cells(entity)
    for _, cell_hash in ipairs(cells) do
        local cell = self.cells[cell_hash]
        if cell then
            for eid, other in pairs(cell) do
                if eid ~= entity.id and not seen[eid] then
                    seen[eid] = true
                    table.insert(nearby, other)
                end
            end
        end
    end
    
    return nearby
end

function SpatialHash:check_collisions(callback)
    local checked_pairs = {}
    
    for _, cell in pairs(self.cells) do
        local entities_in_cell = {}
        for _, entity in pairs(cell) do
            table.insert(entities_in_cell, entity)
        end
        
        -- Check all pairs in this cell
        for i = 1, #entities_in_cell do
            for j = i + 1, #entities_in_cell do
                local e1 = entities_in_cell[i]
                local e2 = entities_in_cell[j]
                
                -- Ensure we don't check same pair twice
                local pair_key = math.min(e1.id, e2.id) .. ":" .. math.max(e1.id, e2.id)
                if not checked_pairs[pair_key] then
                    checked_pairs[pair_key] = true
                    
                    -- Distance check
                    local dx = e1.x - e2.x
                    local dy = e1.y - e2.y
                    local dist = math.sqrt(dx*dx + dy*dy)
                    local min_dist = (e1.radius or 0) + (e2.radius or 0)
                    
                    if dist < min_dist then
                        callback(e1, e2, dist, min_dist - dist)
                    end
                end
            end
        end
    end
end

-- ตัวอย่างการใช้งาน
local spatial_hash = SpatialHash.new(200)

-- Add some bullets and enemies
for i = 1, 10 do
    spatial_hash:insert({
        id = "bullet_" .. i,
        x = math.random(0, 1000),
        y = math.random(0, 1000),
        radius = 5
    })
end

for i = 1, 5 do
    spatial_hash:insert({
        id = "enemy_" .. i,
        x = math.random(0, 1000),
        y = math.random(0, 1000),
        radius = 30
    })
end

print("\n=== Spatial Hash Collision Detection ===")
print(string.format("Entities in hash: %d", spatial_hash.entity_count))

spatial_hash:check_collisions(function(e1, e2, dist, overlap)
    print(string.format("Collision: %s <-> %s (overlap: %.1f)",
                       e1.id, e2.id, overlap))
end)
```

## Entity Streaming

```lua
-- ตัวอย่างที่ 10: Entity Streaming System
local EntityStreamer = {}
EntityStreamer.__index = EntityStreamer

function EntityStreamer.new(options)
    options = options or {}
    return setmetatable({
        streaming_radius = options.streaming_radius or 2000,
        despawn_radius = options.despawn_radius or 2500,
        max_entities_per_client = options.max_entities or 200,
        
        -- All entities in the world
        world_entities = {},
        
        -- Per-client tracked entities
        client_entities = {},
        
        -- Streaming events queue
        events = {}
    }, EntityStreamer)
end

function EntityStreamer:add_entity(entity)
    self.world_entities[entity.id] = entity
end

function EntityStreamer:remove_entity(entity_id)
    self.world_entities[entity_id] = nil
    
    -- Notify clients tracking this entity
    for client_id, tracked in pairs(self.client_entities) do
        if tracked[entity_id] then
            self:queue_event(client_id, "despawn", {entity_id = entity_id})
            tracked[entity_id] = nil
        end
    end
end

function EntityStreamer:update_client_view(client_id, client_x, client_y)
    if not self.client_entities[client_id] then
        self.client_entities[client_id] = {}
    end
    
    local tracked = self.client_entities[client_id]
    local now_visible = {}
    
    -- Find entities within streaming radius
    for eid, entity in pairs(self.world_entities) do
        local dx = entity.x - client_x
        local dy = entity.y - client_y
        local dist = math.sqrt(dx*dx + dy*dy)
        
        if dist <= self.streaming_radius then
            now_visible[eid] = {entity = entity, dist = dist}
        end
    end
    
    -- Limit to max entities (prioritize by proximity)
    if self:count_table(now_visible) > self.max_entities_per_client then
        local sorted = {}
        for eid, info in pairs(now_visible) do
            table.insert(sorted, {id = eid, dist = info.dist, entity = info.entity})
        end
        table.sort(sorted, function(a, b) return a.dist < b.dist end)
        
        now_visible = {}
        for i = 1, self.max_entities_per_client do
            if sorted[i] then
                now_visible[sorted[i].id] = {entity = sorted[i].entity, dist = sorted[i].dist}
            end
        end
    end
    
    -- Spawn new entities
    for eid, info in pairs(now_visible) do
        if not tracked[eid] then
            tracked[eid] = true
            self:queue_event(client_id, "spawn", info.entity)
        end
    end
    
    -- Despawn removed entities
    for eid in pairs(tracked) do
        if not now_visible[eid] then
            tracked[eid] = nil
            self:queue_event(client_id, "despawn", {entity_id = eid})
        end
    end
end

function EntityStreamer:queue_event(client_id, event_type, data)
    if not self.events[client_id] then
        self.events[client_id] = {}
    end
    table.insert(self.events[client_id], {type = event_type, data = data})
end

function EntityStreamer:flush_events(client_id)
    local events = self.events[client_id] or {}
    self.events[client_id] = {}
    return events
end

function EntityStreamer:count_table(t)
    local n = 0
    for _ in pairs(t) do n = n + 1 end
    return n
end
```

## Matchmaking Algorithm

```lua
-- ตัวอย่างที่ 11: ELO-based Matchmaking
local Matchmaker = {}
Matchmaker.__index = Matchmaker

function Matchmaker.new(options)
    options = options or {}
    return setmetatable({
        pool = {},
        matches = {},
        target_match_size = options.match_size or 10,
        max_skill_diff = options.max_skill_diff or 200,
        max_wait_seconds = options.max_wait_time or 60,
        expand_rate = options.expand_rate or 50,  -- expand skill range per 10 seconds
    }, Matchmaker)
end

function Matchmaker:add_player(player)
    player.join_time = os.time()
    player.skill_range = {
        min = player.elo - self.max_skill_diff,
        max = player.elo + self.max_skill_diff
    }
    table.insert(self.pool, player)
    print(string.format("[MM] Player %s joined pool (ELO: %d)", 
                       player.id, player.elo))
end

function Matchmaker:expand_ranges()
    -- Expand skill range for players who've waited too long
    local now = os.time()
    for _, player in ipairs(self.pool) do
        local wait_time = now - player.join_time
        local expansion = math.floor(wait_time / 10) * self.expand_rate
        
        player.skill_range.min = player.elo - self.max_skill_diff - expansion
        player.skill_range.max = player.elo + self.max_skill_diff + expansion
    end
end

function Matchmaker:find_compatible(player, candidates)
    local compatible = {}
    
    for _, candidate in ipairs(candidates) do
        if candidate.id ~= player.id then
            -- Check mutual skill compatibility
            local elo_diff = math.abs(player.elo - candidate.elo)
            local compatible_range = math.min(
                player.skill_range.max - player.skill_range.min,
                candidate.skill_range.max - candidate.skill_range.min
            ) / 2
            
            if elo_diff <= compatible_range then
                table.insert(compatible, candidate)
            end
        end
    end
    
    return compatible
end

function Matchmaker:calculate_match_quality(players)
    if #players < 2 then return 0 end
    
    local elos = {}
    for _, p in ipairs(players) do
        table.insert(elos, p.elo)
    end
    table.sort(elos)
    
    local range = elos[#elos] - elos[1]
    local quality = math.max(0, 1 - range / 400)  -- 0-1 score
    
    return quality
end

function Matchmaker:try_create_match()
    self:expand_ranges()
    
    if #self.pool < self.target_match_size then
        return nil
    end
    
    -- Sort by wait time (oldest first)
    table.sort(self.pool, function(a, b)
        return a.join_time < b.join_time
    end)
    
    -- Try to build a good match starting from longest-waiting player
    local seed_player = self.pool[1]
    local match_players = {seed_player}
    
    -- Find compatible players
    local compatible = self:find_compatible(seed_player, self.pool)
    
    -- Sort by skill closeness
    table.sort(compatible, function(a, b)
        return math.abs(a.elo - seed_player.elo) < math.abs(b.elo - seed_player.elo)
    end)
    
    for _, p in ipairs(compatible) do
        if #match_players >= self.target_match_size then break end
        table.insert(match_players, p)
    end
    
    if #match_players >= self.target_match_size then
        local quality = self:calculate_match_quality(match_players)
        
        -- Remove matched players from pool
        for _, mp in ipairs(match_players) do
            for i, pp in ipairs(self.pool) do
                if pp.id == mp.id then
                    table.remove(self.pool, i)
                    break
                end
            end
        end
        
        local match = {
            id = string.format("MATCH_%d", #self.matches + 1),
            players = match_players,
            quality = quality,
            created_at = os.time()
        }
        table.insert(self.matches, match)
        
        return match
    end
    
    return nil
end

-- ELO Rating Update
local function update_elo(winner_elo, loser_elo, k_factor)
    k_factor = k_factor or 32
    
    local expected_win = 1 / (1 + 10^((loser_elo - winner_elo) / 400))
    local new_winner_elo = winner_elo + k_factor * (1 - expected_win)
    local new_loser_elo = loser_elo + k_factor * (0 - (1 - expected_win))
    
    return math.floor(new_winner_elo), math.floor(new_loser_elo)
end

-- ตัวอย่างการใช้งาน Matchmaking
local mm = Matchmaker.new({match_size = 4})  -- 4 player match for demo

math.randomseed(42)

-- Add players with various ELO ratings
for i = 1, 8 do
    mm:add_player({
        id = "player_" .. i,
        elo = 1000 + math.random(-300, 300),
        name = "Player " .. i
    })
end

print("\n=== Matchmaking ===")
local match = mm:try_create_match()
if match then
    print(string.format("Match created: %s (quality: %.0f%%)",
                       match.id, match.quality * 100))
    print("Players:")
    for _, p in ipairs(match.players) do
        print(string.format("  %s (ELO: %d)", p.name, p.elo))
    end
else
    print("Not enough players for a match")
end

-- ELO update example
print("\n=== ELO Update ===")
local winner_elo, loser_elo = 1200, 1100
local new_winner, new_loser = update_elo(winner_elo, loser_elo)
print(string.format("Winner: %d -> %d (+%d)", winner_elo, new_winner, new_winner - winner_elo))
print(string.format("Loser:  %d -> %d (%d)", loser_elo, new_loser, new_loser - loser_elo))
```

## Game State Persistence

```lua
-- ตัวอย่างที่ 12: Game State Persistence Layer
local GamePersistence = {}
GamePersistence.__index = GamePersistence

function GamePersistence.new(backend)
    return setmetatable({
        backend = backend or "memory",
        store = {},  -- In-memory store (simulate DB)
        save_queue = {},
        dirty_players = {}
    }, GamePersistence)
end

function GamePersistence:save_player(player_id, data, priority)
    priority = priority or "normal"
    
    -- Mark as dirty
    self.dirty_players[player_id] = {
        data = data,
        priority = priority,
        queued_at = os.time()
    }
    
    -- Immediate save for critical data
    if priority == "critical" then
        self:flush_player(player_id)
    end
end

function GamePersistence:flush_player(player_id)
    local queued = self.dirty_players[player_id]
    if not queued then return end
    
    -- Simulate database save
    self.store[player_id] = {
        data = queued.data,
        saved_at = os.time(),
        version = (self.store[player_id] and self.store[player_id].version or 0) + 1
    }
    
    self.dirty_players[player_id] = nil
    
    print(string.format("[Persist] Saved player %s (v%d)",
                       player_id, self.store[player_id].version))
end

function GamePersistence:flush_all()
    local saved = 0
    for pid in pairs(self.dirty_players) do
        self:flush_player(pid)
        saved = saved + 1
    end
    if saved > 0 then
        print(string.format("[Persist] Batch saved %d players", saved))
    end
end

function GamePersistence:load_player(player_id)
    local stored = self.store[player_id]
    if not stored then
        return nil, "Player not found"
    end
    return stored.data
end

-- ตัวอย่างที่ 13: Anti-Cheat System
local AntiCheat = {}
AntiCheat.__index = AntiCheat

AntiCheat.VIOLATION_TYPES = {
    SPEED_HACK = "speed_hack",
    TELEPORT = "teleport",
    IMPOSSIBLE_ACTION = "impossible_action",
    PACKET_MANIPULATION = "packet_manipulation",
    WALLHACK_INDICATOR = "wallhack_indicator"
}

function AntiCheat.new()
    return setmetatable({
        player_history = {},
        violations = {},
        ban_thresholds = {
            speed_hack = 5,
            teleport = 3,
            impossible_action = 10
        }
    }, AntiCheat)
end

function AntiCheat:check_movement(player_id, new_x, new_y, dt)
    local history = self.player_history[player_id]
    
    if not history then
        self.player_history[player_id] = {
            x = new_x, y = new_y, last_update = os.clock()
        }
        return true
    end
    
    -- Calculate actual speed
    local dx = new_x - history.x
    local dy = new_y - history.y
    local distance = math.sqrt(dx*dx + dy*dy)
    local elapsed = os.clock() - history.last_update
    
    if elapsed <= 0 then elapsed = 0.001 end
    
    local actual_speed = distance / elapsed
    local max_allowed_speed = 500  -- units per second
    
    -- Update history
    history.x = new_x
    history.y = new_y
    history.last_update = os.clock()
    
    -- Teleport check (moved too far in one step)
    if distance > 1000 then
        self:record_violation(player_id, AntiCheat.VIOLATION_TYPES.TELEPORT, {
            distance = distance,
            from = {x = history.x, y = history.y},
            to = {x = new_x, y = new_y}
        })
        return false
    end
    
    -- Speed check
    if actual_speed > max_allowed_speed * 1.5 then
        self:record_violation(player_id, AntiCheat.VIOLATION_TYPES.SPEED_HACK, {
            actual_speed = actual_speed,
            max_speed = max_allowed_speed
        })
        return false
    end
    
    return true
end

function AntiCheat:record_violation(player_id, violation_type, details)
    if not self.violations[player_id] then
        self.violations[player_id] = {}
    end
    
    local violation = {
        type = violation_type,
        timestamp = os.time(),
        details = details,
        count = (self.violations[player_id][violation_type] or 0) + 1
    }
    
    self.violations[player_id][violation_type] = violation.count
    table.insert(self.violations[player_id], violation)
    
    print(string.format("[AntiCheat] Violation: player=%s type=%s count=%d",
                       player_id, violation_type, violation.count))
    
    -- Check if should ban
    local threshold = self.ban_thresholds[violation_type]
    if threshold and violation.count >= threshold then
        self:ban_player(player_id, violation_type)
    end
end

function AntiCheat:ban_player(player_id, reason)
    print(string.format("[AntiCheat] BANNING player %s: %s", player_id, reason))
    -- In real implementation: kick from server and add to ban database
end

function AntiCheat:get_player_report(player_id)
    local violations = self.violations[player_id]
    if not violations then
        return "No violations"
    end
    
    local counts = {}
    for k, v in pairs(violations) do
        if type(k) == "string" then
            table.insert(counts, k .. ":" .. v)
        end
    end
    
    return table.concat(counts, ", ")
end

-- ตัวอย่างการใช้งาน Anti-Cheat
local anticheat = AntiCheat.new()

print("\n=== Anti-Cheat System ===")

-- Normal movement
for i = 1, 5 do
    anticheat:check_movement("player1", i * 5, i * 3, 0.016)
end
print("Player1 (normal):", anticheat:get_player_report("player1"))

-- Speedhack attempt
anticheat:check_movement("cheater", 100, 100, 0.016)
anticheat:check_movement("cheater", 5000, 5000, 0.016)  -- teleport!
anticheat:check_movement("cheater", 5500, 5500, 0.001)  -- too fast!
print("Cheater:", anticheat:get_player_report("cheater"))
```

## Session Management

```lua
-- ตัวอย่างที่ 14: Game Session Manager
local GameSessionManager = {}
GameSessionManager.__index = GameSessionManager

function GameSessionManager.new(options)
    options = options or {}
    return setmetatable({
        sessions = {},
        player_sessions = {},  -- player_id -> session_id
        max_sessions = options.max_sessions or 1000,
        session_timeout = options.timeout or 3600,  -- 1 hour
        config = options
    }, GameSessionManager)
end

function GameSessionManager:create_session(host_player, game_mode, config)
    if self:count_sessions() >= self.max_sessions then
        return nil, "Server full"
    end
    
    local session_id = string.format("SESS_%06d", math.random(0, 999999))
    
    local session = {
        id = session_id,
        host = host_player,
        game_mode = game_mode,
        config = config or {},
        state = "lobby",
        players = {[host_player] = {ready = false, joined_at = os.time()}},
        spectators = {},
        created_at = os.time(),
        started_at = nil,
        ended_at = nil,
        max_players = config and config.max_players or 10,
        min_players = config and config.min_players or 2
    }
    
    self.sessions[session_id] = session
    self.player_sessions[host_player] = session_id
    
    print(string.format("[Session] Created: %s (mode: %s, host: %s)",
                       session_id, game_mode, host_player))
    
    return session_id
end

function GameSessionManager:join_session(session_id, player_id, as_spectator)
    local session = self.sessions[session_id]
    if not session then
        return false, "Session not found"
    end
    
    if session.state ~= "lobby" then
        if not as_spectator then
            return false, "Game already in progress"
        end
    end
    
    if not as_spectator then
        local player_count = 0
        for _ in pairs(session.players) do player_count = player_count + 1 end
        
        if player_count >= session.max_players then
            return false, "Session full"
        end
        
        session.players[player_id] = {ready = false, joined_at = os.time()}
        self.player_sessions[player_id] = session_id
        print(string.format("[Session] Player %s joined %s", player_id, session_id))
    else
        session.spectators[player_id] = {joined_at = os.time()}
        print(string.format("[Session] Spectator %s joined %s", player_id, session_id))
    end
    
    return true
end

function GameSessionManager:player_ready(session_id, player_id)
    local session = self.sessions[session_id]
    if not session or not session.players[player_id] then
        return false
    end
    
    session.players[player_id].ready = true
    
    -- Check if all players ready
    local all_ready = true
    local count = 0
    for _, player in pairs(session.players) do
        count = count + 1
        if not player.ready then
            all_ready = false
        end
    end
    
    if all_ready and count >= session.min_players then
        self:start_session(session_id)
    end
    
    return true
end

function GameSessionManager:start_session(session_id)
    local session = self.sessions[session_id]
    if not session then return false end
    
    session.state = "in_progress"
    session.started_at = os.time()
    
    print(string.format("[Session] Started: %s (%d players)",
                       session_id, (function()
                           local n = 0
                           for _ in pairs(session.players) do n = n + 1 end
                           return n
                       end)()))
    return true
end

function GameSessionManager:end_session(session_id, results)
    local session = self.sessions[session_id]
    if not session then return false end
    
    session.state = "ended"
    session.ended_at = os.time()
    session.results = results
    
    -- Clean up player session references
    for pid in pairs(session.players) do
        self.player_sessions[pid] = nil
    end
    
    local duration = session.ended_at - (session.started_at or session.created_at)
    print(string.format("[Session] Ended: %s (duration: %ds)", 
                       session_id, duration))
    
    -- Archive or delete after some time
    -- self.sessions[session_id] = nil  -- immediate cleanup
    
    return true
end

function GameSessionManager:count_sessions()
    local n = 0
    for _ in pairs(self.sessions) do n = n + 1 end
    return n
end

function GameSessionManager:get_active_sessions()
    local active = {}
    for id, session in pairs(self.sessions) do
        if session.state == "lobby" or session.state == "in_progress" then
            table.insert(active, session)
        end
    end
    return active
end

function GameSessionManager:status_report()
    print("\n=== Session Manager Status ===")
    local total = self.count_sessions and self:count_sessions() or 0
    local active = #self:get_active_sessions()
    
    print(string.format("Total sessions: %d", total))
    print(string.format("Active sessions: %d", active))
    
    for id, session in pairs(self.sessions) do
        local player_count = 0
        for _ in pairs(session.players) do player_count = player_count + 1 end
        print(string.format("  %s: %s (%d players, mode: %s)",
                           id, session.state, player_count, session.game_mode))
    end
end

-- ตัวอย่างการใช้งาน
local sm = GameSessionManager.new({max_sessions = 100})

local session_id = sm:create_session("player1", "team_deathmatch", {
    max_players = 4,
    min_players = 2,
    map = "de_dust2"
})

sm:join_session(session_id, "player2")
sm:join_session(session_id, "player3")

sm:player_ready(session_id, "player1")
sm:player_ready(session_id, "player2")
sm:player_ready(session_id, "player3")

-- After game ends
sm:end_session(session_id, {
    winner = "team_a",
    score = {team_a = 16, team_b = 10},
    mvp = "player1"
})

sm:status_report()
```

## Complete Game Server Example

```lua
-- ตัวอย่างที่ 15: Putting It All Together
local function create_game_server(config)
    config = config or {}
    
    local server = {
        -- Components
        tick_rate = config.tick_rate or 64,
        max_players = config.max_players or 100,
        
        -- State
        tick = 0,
        players = {},
        
        -- Systems
        aoi = AoIManager.new({player_radius = 1500}),
        anticheat = AntiCheat.new(),
        matchmaker = Matchmaker.new({match_size = config.match_size or 10}),
        session_manager = GameSessionManager.new({max_sessions = 1000}),
        persistence = GamePersistence.new(),
        reconciler = ServerReconciler.new()
    }
    
    function server:add_player(player_id, elo)
        local player = {
            id = player_id,
            x = math.random(100, 900),
            y = math.random(100, 900),
            health = 100,
            speed = 200,  -- units/second
            elo = elo or 1000
        }
        
        self.players[player_id] = player
        self.aoi:update_entity(player_id, player.x, player.y)
        
        print(string.format("[Server] Player %s spawned at (%.0f, %.0f)",
                           player_id, player.x, player.y))
        
        return player
    end
    
    function server:handle_input(player_id, input)
        local player = self.players[player_id]
        if not player then return false end
        
        -- Anti-cheat check
        if input.move_x or input.move_y then
            local new_x = player.x + (input.move_x or 0)
            local new_y = player.y + (input.move_y or 0)
            
            local ok = self.anticheat:check_movement(player_id, new_x, new_y, 0.016)
            if not ok then return false end
            
            player.x = new_x
            player.y = new_y
            
            -- Update spatial systems
            self.aoi:update_entity(player_id, player.x, player.y)
        end
        
        return true
    end
    
    function server:get_visible_players(player_id)
        local player = self.players[player_id]
        if not player then return {} end
        
        return self.aoi:get_entities_in_range(player.x, player.y)
    end
    
    function server:tick_update()
        self.tick = self.tick + 1
        
        -- Periodic saves
        if self.tick % (self.tick_rate * 30) == 0 then  -- every 30 seconds
            self.persistence:flush_all()
        end
    end
    
    function server:status()
        local player_count = 0
        for _ in pairs(self.players) do player_count = player_count + 1 end
        
        print(string.format([[
=== Game Server Status ===
Tick: %d
Players online: %d/%d
Tick rate: %d Hz]],
            self.tick, player_count, self.max_players, self.tick_rate
        ))
    end
    
    return server
end

-- Run demo
print("\n=== Game Server Demo ===")
math.randomseed(100)

local game_server = create_game_server({
    tick_rate = 64,
    max_players = 100,
    match_size = 4
})

-- Add some players
for i = 1, 5 do
    game_server:add_player("player_" .. i, 1000 + math.random(-200, 200))
end

-- Simulate some ticks
for _ = 1, 10 do
    for i = 1, 5 do
        local pid = "player_" .. i
        game_server:handle_input(pid, {
            move_x = math.random(-5, 5),
            move_y = math.random(-5, 5)
        })
    end
    game_server:tick_update()
end

game_server:status()

-- Show AoI
print("\nPlayers visible to player_1:")
local visible = game_server:get_visible_players("player_1")
for _, entity in ipairs(visible) do
    print(string.format("  %s at distance %.0f", entity.id, entity.distance))
end
```

## สรุปบทที่ 90

ในบทนี้เราได้เรียนรู้ Game Server Architecture ครอบคลุม:

1. **Game Server Types** - Authoritative vs Relay servers
2. **State Synchronization** - Delta encoding และ baseline compression
3. **Delta Compression** - Efficient state updates
4. **Lag Compensation** - Server-side hit detection with history rewind
5. **Client-Side Prediction** - Local movement prediction
6. **Server Reconciliation** - Correcting prediction errors
7. **Lockstep Simulation** - Deterministic simulation with checksums
8. **Area of Interest** - Grid-based entity visibility management
9. **Spatial Hashing** - Efficient collision detection
10. **Entity Streaming** - Dynamic entity loading/unloading
11. **Matchmaking** - ELO-based player matching
12. **Game Persistence** - Saving and loading game state
13. **Anti-Cheat** - Movement validation, speed detection
14. **Session Management** - Game session lifecycle
15. **Complete Integration** - All systems working together

Key game server principles:
- Server เป็น authoritative source of truth เสมอ
- Client prediction ทำให้ game รู้สึก responsive แม้มี latency
- Lag compensation ช่วยให้การต่อสู้รู้สึกยุติธรรมสำหรับทุก player
- Deterministic simulation จำเป็นสำหรับ lockstep และ replay features
- AoI management ลด bandwidth โดยส่งเฉพาะข้อมูลที่ player ต้องเห็น
- Anti-cheat ต้องทำทั้ง client (ป้องกัน obvious cheats) และ server (authoritative validation)
