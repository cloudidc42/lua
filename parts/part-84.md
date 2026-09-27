# บทที่ 84: Distributed Systems Concepts

## บทนำ

Distributed Systems คือระบบที่ประกอบด้วย components หลายตัวที่ทำงานบนเครือข่าย โดยแต่ละ component ไม่รู้ว่า component อื่นๆ อยู่ที่ไหน แต่ทำงานร่วมกันเพื่อให้ผู้ใช้เห็นเป็นระบบเดียว การออกแบบ distributed systems เป็นเรื่องยากมากเพราะต้องจัดการกับ partial failures, network partitions, clock synchronization และ concurrency

ในบทนี้เราจะเรียนรู้ concepts สำคัญและ implement ตัวอย่างด้วย Lua พร้อม simulation ของ distributed behaviors

---

## 84.1 CAP Theorem

CAP Theorem บอกว่าใน distributed system คุณไม่สามารถมีทั้งสามสิ่งนี้พร้อมกันได้:

```lua
-- CAP Theorem:
-- C = Consistency: ทุก node เห็น data เดียวกันตลอดเวลา
-- A = Availability: ทุก request ได้รับ response (ไม่มี timeout/error)
-- P = Partition Tolerance: ระบบทำงานได้แม้ network แบ่งออกเป็นส่วนๆ

-- ในทางปฏิบัติ P ต้อง handle เสมอ (network failures เกิดขึ้นแน่นอน)
-- ดังนั้นต้องเลือกระหว่าง C และ A

-- CA systems (ไม่ handle partition): Single-node databases, RDBMS
-- CP systems (เลือก Consistency): HBase, Zookeeper, etcd, MongoDB
-- AP systems (เลือก Availability): CouchDB, Cassandra, DynamoDB

local function cap_theorem_demo()
    -- Simulation: สองส่วนของ network แบ่งออกจากกัน
    
    -- Node A (ฝั่งซ้าย)
    local node_a = {data = "version1", version = 1}
    
    -- Node B (ฝั่งขวา)
    local node_b = {data = "version1", version = 1}
    
    -- Partition เกิดขึ้น: A และ B คุยกันไม่ได้
    local partition = true
    
    -- Client ส่ง write ไปที่ Node A
    local function write_to_a(new_data)
        node_a.data = new_data
        node_a.version = node_a.version + 1
        
        if not partition then
            -- Replicate ไปยัง B
            node_b.data = new_data
            node_b.version = node_a.version
        end
        -- ถ้า partition อยู่: B จะ stale
    end
    
    -- CP behavior: ปฏิเสธ write ถ้า majority ไม่ available
    local function cp_write(new_data)
        if partition then
            return false, "Cannot write: partition detected, maintaining consistency"
        end
        write_to_a(new_data)
        return true
    end
    
    -- AP behavior: รับ write แม้จะมี partition
    local function ap_write(new_data)
        write_to_a(new_data)
        return true  -- accept แม้ B อาจ stale
    end
    
    -- ทดสอบ
    write_to_a("initial")
    partition = true
    
    local ok, err = cp_write("update_during_partition")
    print("CP write:", ok, err)  -- false, error
    
    ok = ap_write("update_during_partition")
    print("AP write:", ok)  -- true
    
    -- เมื่อ partition หาย
    partition = false
    print("After partition heal:")
    print("  Node A:", node_a.data, "v" .. node_a.version)
    print("  Node B:", node_b.data, "v" .. node_b.version)
    -- ต้อง reconcile/merge ความแตกต่าง!
end

cap_theorem_demo()
```

---

## 84.2 Consistency Models

```lua
-- Strong Consistency: ทุกคนเห็น data เดียวกันพร้อมกันเสมอ
-- Eventual Consistency: ในที่สุดทุกคนจะเห็นเหมือนกัน
-- Causal Consistency: operations ที่มีความสัมพันธ์กันจะเห็นตามลำดับที่ถูกต้อง

-- Simulation: eventually consistent key-value store

local EventuallyConsistentKV = {}
EventuallyConsistentKV.__index = EventuallyConsistentKV

function EventuallyConsistentKV.new(node_id)
    local self = setmetatable({}, EventuallyConsistentKV)
    self.node_id = node_id
    self.data = {}          -- {key → {value, timestamp, version}}
    self.peers = {}         -- รายชื่อ peer nodes
    self.message_queue = {} -- รอส่ง
    return self
end

function EventuallyConsistentKV:set(key, value)
    local timestamp = os.time() * 1000 + math.random(0, 999)
    local entry = {
        value = value,
        timestamp = timestamp,
        node = self.node_id,
    }
    self.data[key] = entry
    
    -- Queue for replication to peers
    for _, peer in ipairs(self.peers) do
        self.message_queue[#self.message_queue+1] = {
            to = peer,
            key = key,
            entry = entry,
        }
    end
    
    return true
end

function EventuallyConsistentKV:get(key)
    local entry = self.data[key]
    if entry then
        return entry.value, entry.timestamp
    end
    return nil, nil
end

-- รับ replication message
function EventuallyConsistentKV:receive_update(key, remote_entry)
    local local_entry = self.data[key]
    
    if not local_entry then
        -- ไม่มี local: accept
        self.data[key] = remote_entry
    elseif remote_entry.timestamp > local_entry.timestamp then
        -- Remote ใหม่กว่า: accept
        self.data[key] = remote_entry
    elseif remote_entry.timestamp == local_entry.timestamp then
        -- Conflict: ใช้ node_id เป็น tiebreaker (LWW - Last Write Wins)
        if remote_entry.node > local_entry.node then
            self.data[key] = remote_entry
        end
        -- มิฉะนั้นเก็บ local ไว้
    end
    -- Remote เก่ากว่า: ignore
end

-- Propagate updates
function EventuallyConsistentKV:flush(all_nodes)
    local sent = #self.message_queue
    for _, msg in ipairs(self.message_queue) do
        local target = all_nodes[msg.to]
        if target then
            target:receive_update(msg.key, msg.entry)
        end
    end
    self.message_queue = {}
    return sent
end

-- ทดสอบ eventual consistency
local function test_eventual_consistency()
    local nodes = {}
    for i = 1, 3 do
        nodes[i] = EventuallyConsistentKV.new("node" .. i)
    end
    
    -- เชื่อม peers
    for i = 1, 3 do
        for j = 1, 3 do
            if i ~= j then
                nodes[i].peers[#nodes[i].peers+1] = j
            end
        end
    end
    
    local all = {}
    for i, n in ipairs(nodes) do all[i] = n end
    
    -- Write ไปที่ node 1
    nodes[1]:set("user:1", {name="Alice", age=30})
    
    -- ณ จุดนี้ node 2 และ 3 ยังไม่รู้
    print("Before propagation:")
    print("  node1.user:1 =", tostring(nodes[1]:get("user:1")))
    print("  node2.user:1 =", tostring(nodes[2]:get("user:1")))
    
    -- Propagate
    nodes[1]:flush(all)
    
    print("After propagation:")
    for i, n in ipairs(nodes) do
        local val = n:get("user:1")
        print(string.format("  node%d.user:1 = %s", i, val and val.name or "nil"))
    end
    
    -- Concurrent writes (conflict scenario)
    print("\n--- Conflict Scenario ---")
    nodes[1]:set("counter", {value = 10})
    nodes[2]:set("counter", {value = 20})  -- ไม่รู้เรื่อง node1's write
    
    -- Propagate node1 ก่อน
    nodes[1]:flush(all)
    -- แล้ว node2
    nodes[2]:flush(all)
    
    -- ดูผล (LWW จะเลือกอันล่าสุด)
    for i, n in ipairs(nodes) do
        local val = n:get("counter")
        print(string.format("  node%d.counter = %s", i, val and tostring(val.value) or "nil"))
    end
end

test_eventual_consistency()
```

---

## 84.3 Vector Clocks

Vector Clocks ช่วยติดตาม causal ordering ของ events:

```lua
-- Vector Clock implementation
local VectorClock = {}
VectorClock.__index = VectorClock

function VectorClock.new(node_id, all_nodes)
    local self = setmetatable({}, VectorClock)
    self.node_id = node_id
    self.clock = {}
    for _, n in ipairs(all_nodes) do
        self.clock[n] = 0
    end
    return self
end

function VectorClock:tick()
    self.clock[self.node_id] = self.clock[self.node_id] + 1
    return self:copy()
end

function VectorClock:copy()
    local c = {}
    for k, v in pairs(self.clock) do
        c[k] = v
    end
    return c
end

function VectorClock:merge(other_clock)
    for node, time in pairs(other_clock) do
        self.clock[node] = math.max(self.clock[node] or 0, time)
    end
end

function VectorClock:receive(other_clock)
    self:merge(other_clock)
    self.clock[self.node_id] = self.clock[self.node_id] + 1
end

-- Comparison
function VectorClock:happened_before(vc_a, vc_b)
    -- A happened before B iff:
    -- ทุก component ของ A <= B และ อย่างน้อย 1 component < B
    local less_than = false
    for node, time_a in pairs(vc_a) do
        local time_b = vc_b[node] or 0
        if time_a > time_b then
            return false  -- A ไม่ได้ happen before B
        end
        if time_a < time_b then
            less_than = true
        end
    end
    return less_than
end

function VectorClock:concurrent(vc_a, vc_b)
    -- Concurrent ถ้า A ไม่ happen before B และ B ไม่ happen before A
    return not self:happened_before(vc_a, vc_b) and 
           not self:happened_before(vc_b, vc_a)
end

function VectorClock:to_string(vc)
    local parts = {}
    local keys = {}
    for k in pairs(vc) do keys[#keys+1] = k end
    table.sort(keys)
    for _, k in ipairs(keys) do
        parts[#parts+1] = k .. ":" .. vc[k]
    end
    return "{" .. table.concat(parts, ", ") .. "}"
end

-- ทดสอบ vector clocks
local function test_vector_clocks()
    local nodes = {"A", "B", "C"}
    local vc_a = VectorClock.new("A", nodes)
    local vc_b = VectorClock.new("B", nodes)
    local vc_c = VectorClock.new("C", nodes)
    
    print("\n=== Vector Clock Demo ===")
    
    -- A ส่ง message ไปยัง B
    local msg1_clock = vc_a:tick()
    print("A sends msg1 with clock:", vc_c:to_string(msg1_clock))
    -- {A:1, B:0, C:0}
    
    -- B receives message
    vc_b:receive(msg1_clock)
    print("B receives msg1, B's clock:", vc_c:to_string(vc_b.clock))
    -- {A:1, B:1, C:0}
    
    -- B ส่ง message ไปยัง C
    local msg2_clock = vc_b:tick()
    print("B sends msg2 with clock:", vc_c:to_string(msg2_clock))
    -- {A:1, B:2, C:0}
    
    -- C receives message
    vc_c:receive(msg2_clock)
    print("C receives msg2, C's clock:", vc_c:to_string(vc_c.clock))
    -- {A:1, B:2, C:1}
    
    -- Causal ordering check
    print("\nCausal checks:")
    print("msg1 happened before msg2:", vc_c:happened_before(msg1_clock, msg2_clock))
    print("msg2 happened before msg1:", vc_c:happened_before(msg2_clock, msg1_clock))
    
    -- Concurrent events
    local msg3_clock = vc_a:tick()  -- A ส่งต่อจาก msg1
    local msg4_clock = vc_c:tick()  -- C ส่ง independently
    
    print("msg3 concurrent with msg4:", vc_c:concurrent(msg3_clock, msg4_clock))
end

test_vector_clocks()
```

---

## 84.4 Lamport Timestamps

Lamport timestamps ง่ายกว่า vector clocks แต่ให้ข้อมูลน้อยกว่า:

```lua
-- Lamport Clock
local LamportClock = {}
LamportClock.__index = LamportClock

function LamportClock.new()
    return setmetatable({time = 0}, LamportClock)
end

function LamportClock:tick()
    self.time = self.time + 1
    return self.time
end

function LamportClock:receive(remote_time)
    -- เลือก max(local, remote) + 1
    self.time = math.max(self.time, remote_time) + 1
    return self.time
end

-- Distributed event log ด้วย Lamport timestamps
local DistributedLog = {}
DistributedLog.__index = DistributedLog

function DistributedLog.new(node_id)
    local self = setmetatable({}, DistributedLog)
    self.node_id = node_id
    self.clock = LamportClock.new()
    self.events = {}
    return self
end

function DistributedLog:record(event_type, data)
    local ts = self.clock:tick()
    local event = {
        timestamp = ts,
        node = self.node_id,
        type = event_type,
        data = data,
    }
    self.events[#self.events+1] = event
    return event
end

function DistributedLog:receive_event(event)
    self.clock:receive(event.timestamp)
    self.events[#self.events+1] = event
end

function DistributedLog:get_sorted_events()
    local all = {}
    for _, e in ipairs(self.events) do
        all[#all+1] = e
    end
    -- Sort by (timestamp, node_id) สำหรับ total ordering
    table.sort(all, function(a, b)
        if a.timestamp ~= b.timestamp then
            return a.timestamp < b.timestamp
        end
        return a.node < b.node
    end)
    return all
end

-- ทดสอบ
local function test_lamport()
    local nodeA = DistributedLog.new("A")
    local nodeB = DistributedLog.new("B")
    
    -- Events ที่เกิดขึ้นแบบ concurrent
    local e1 = nodeA:record("user_login", {user="alice"})
    local e2 = nodeB:record("user_login", {user="bob"})
    local e3 = nodeA:record("item_added", {item="laptop"})
    
    -- B รับ event จาก A
    nodeB:receive_event(e1)
    local e4 = nodeB:record("order_created", {items={"laptop"}})
    
    -- A รับ events จาก B
    nodeA:receive_event(e2)
    nodeA:receive_event(e4)
    
    print("\n=== Lamport Ordered Events ===")
    local sorted = nodeA:get_sorted_events()
    for _, e in ipairs(sorted) do
        print(string.format("  [T=%d, N=%s] %s: %s",
            e.timestamp, e.node, e.type,
            type(e.data) == "table" and e.data.user or e.data.item or "..."))
    end
end

test_lamport()
```

---

## 84.5 Distributed Locks ด้วย Redis (Simulation)

```lua
-- Redlock algorithm simulation
-- (ในทางปฏิบัติต้องใช้ redis client จริงๆ)

-- Simulated Redis node
local RedisNode = {}
RedisNode.__index = RedisNode

function RedisNode.new(id, is_alive)
    return setmetatable({
        id = id,
        alive = is_alive ~= false,
        data = {},
        latency = math.random(1, 5),  -- simulate network latency ms
    }, RedisNode)
end

function RedisNode:set_nx(key, value, ttl_ms)
    if not self.alive then
        return false, "node down"
    end
    
    -- SET if Not eXists
    if self.data[key] and self.data[key].expires_at > os.time() * 1000 then
        return false, "key exists"
    end
    
    self.data[key] = {
        value = value,
        expires_at = os.time() * 1000 + ttl_ms,
    }
    return true
end

function RedisNode:del_if_equals(key, value)
    if not self.alive then return false end
    
    local entry = self.data[key]
    if entry and entry.value == value then
        self.data[key] = nil
        return true
    end
    return false
end

-- Redlock implementation
local Redlock = {}
Redlock.__index = Redlock

function Redlock.new(redis_nodes)
    return setmetatable({
        nodes = redis_nodes,
        quorum = math.floor(#redis_nodes / 2) + 1,  -- majority
    }, Redlock)
end

function Redlock:acquire(resource, ttl_ms)
    local value = string.format("%s-%d-%d", 
        resource, os.time(), math.random(1000000))
    
    local start_time = os.clock() * 1000
    local acquired_count = 0
    local acquired_nodes = {}
    
    -- ลอง acquire จากทุก node
    for _, node in ipairs(self.nodes) do
        local ok = node:set_nx(resource, value, ttl_ms)
        if ok then
            acquired_count = acquired_count + 1
            acquired_nodes[#acquired_nodes+1] = node
        end
    end
    
    local elapsed = os.clock() * 1000 - start_time
    local validity = ttl_ms - elapsed - 2  -- 2ms drift factor
    
    -- ต้องได้ majority และยังมีเวลาพอ
    if acquired_count >= self.quorum and validity > 0 then
        return {
            value = value,
            resource = resource,
            validity = validity,
            acquired_nodes = acquired_nodes,
        }
    else
        -- ไม่สำเร็จ: release ทั้งหมดที่ได้
        for _, node in ipairs(acquired_nodes) do
            node:del_if_equals(resource, value)
        end
        return nil
    end
end

function Redlock:release(lock)
    for _, node in ipairs(self.nodes) do
        node:del_if_equals(lock.resource, lock.value)
    end
end

-- ทดสอบ Redlock
local function test_redlock()
    -- สร้าง Redis cluster
    local nodes = {}
    for i = 1, 5 do
        nodes[i] = RedisNode.new("redis" .. i)
    end
    
    -- Simulate node failure
    nodes[3].alive = false
    nodes[5].alive = false
    
    local redlock = Redlock.new(nodes)
    
    print("\n=== Redlock Demo ===")
    print("Nodes:", #nodes)
    print("Quorum needed:", redlock.quorum)
    print("Alive nodes:", 3)
    
    -- Client 1 acquire
    local lock1 = redlock:acquire("shared_resource", 10000)
    print("\nClient1 acquire:", lock1 ~= nil)
    
    -- Client 2 ลอง acquire เดียวกัน
    local lock2 = redlock:acquire("shared_resource", 10000)
    print("Client2 acquire:", lock2 ~= nil)  -- false!
    
    -- Release lock1
    if lock1 then
        redlock:release(lock1)
        print("Client1 released")
    end
    
    -- ตอนนี้ client2 ควรได้
    lock2 = redlock:acquire("shared_resource", 10000)
    print("Client2 re-acquire:", lock2 ~= nil)  -- true
    
    if lock2 then redlock:release(lock2) end
end

test_redlock()
```

---

## 84.6 Raft Consensus Algorithm (Simplified)

```lua
-- Raft overview:
-- Leader election:
--   1. Nodes เริ่มเป็น Follower
--   2. ถ้าไม่ได้ยิน heartbeat: เป็น Candidate
--   3. Candidate ขอ vote จากทุกคน
--   4. ถ้าได้ majority: เป็น Leader
-- Log replication:
--   1. Clients ส่ง commands ไปที่ Leader
--   2. Leader append ลง log
--   3. Leader replicate ไปยัง Followers
--   4. เมื่อ majority confirm: commit

local RaftNode = {}
RaftNode.__index = RaftNode

local FOLLOWER  = "follower"
local CANDIDATE = "candidate"
local LEADER    = "leader"

function RaftNode.new(id, all_nodes)
    return setmetatable({
        id = id,
        state = FOLLOWER,
        current_term = 0,
        voted_for = nil,
        log = {},           -- {term, command}
        commit_index = 0,
        last_applied = 0,
        all_nodes = all_nodes,
        votes_received = {},
        leader_id = nil,
        election_timeout = math.random(150, 300),  -- ms
        last_heartbeat = os.clock() * 1000,
    }, RaftNode)
end

function RaftNode:start_election()
    self.current_term = self.current_term + 1
    self.state = CANDIDATE
    self.voted_for = self.id
    self.votes_received = {[self.id] = true}
    
    print(string.format("Node %s starts election for term %d",
        self.id, self.current_term))
    
    -- Request votes from all other nodes
    local votes = 1  -- vote for self
    for _, node_id in ipairs(self.all_nodes) do
        if node_id ~= self.id then
            -- Simulate request vote RPC
            local granted = self:request_vote_from(node_id)
            if granted then
                votes = votes + 1
                self.votes_received[node_id] = true
            end
        end
    end
    
    -- Check if won
    local quorum = math.floor(#self.all_nodes / 2) + 1
    if votes >= quorum then
        self:become_leader()
    else
        self.state = FOLLOWER
        print(string.format("Node %s lost election (got %d/%d votes)",
            self.id, votes, quorum))
    end
end

function RaftNode:request_vote_from(node_id)
    -- Simplified: ใน Raft จริงๆ ต้องส่ง term, last_log_index, last_log_term
    -- ที่นี่ simulate ว่า node ยอมรับ term ที่สูงกว่า
    return math.random() > 0.3  -- 70% chance grant
end

function RaftNode:become_leader()
    self.state = LEADER
    self.leader_id = self.id
    print(string.format("Node %s became LEADER for term %d",
        self.id, self.current_term))
end

function RaftNode:append_entry(command)
    if self.state ~= LEADER then
        return false, "not leader"
    end
    
    -- Append to log
    local entry = {
        term = self.current_term,
        command = command,
        index = #self.log + 1,
    }
    self.log[#self.log+1] = entry
    
    -- Replicate to followers (simplified)
    local replicated = 1  -- leader itself
    for _, node_id in ipairs(self.all_nodes) do
        if node_id ~= self.id then
            -- Simulate AppendEntries RPC
            if math.random() > 0.2 then  -- 80% success
                replicated = replicated + 1
            end
        end
    end
    
    -- Commit if majority
    local quorum = math.floor(#self.all_nodes / 2) + 1
    if replicated >= quorum then
        self.commit_index = entry.index
        return true
    else
        -- ต้อง retry (simplified: just return false)
        self.log[#self.log] = nil  -- rollback
        return false, "failed to replicate"
    end
end

-- ทดสอบ Raft
local function test_raft()
    print("\n=== Raft Consensus Demo ===")
    
    local node_ids = {"node1", "node2", "node3", "node4", "node5"}
    local nodes = {}
    for _, id in ipairs(node_ids) do
        nodes[id] = RaftNode.new(id, node_ids)
    end
    
    -- Election
    math.randomseed(42)
    nodes["node1"]:start_election()
    
    -- ถ้าชนะ: ส่ง commands
    if nodes["node1"].state == LEADER then
        print("\nAppending entries:")
        
        local ok, err = nodes["node1"]:append_entry("set x = 1")
        print("  'set x = 1':", ok, err)
        
        ok, err = nodes["node1"]:append_entry("set y = 2")
        print("  'set y = 2':", ok, err)
        
        ok, err = nodes["node1"]:append_entry("set z = x + y")
        print("  'set z = x + y':", ok, err)
        
        print("Committed log entries:", nodes["node1"].commit_index)
    end
end

test_raft()
```

---

## 84.7 Two-Phase Commit (2PC)

```lua
-- Two-Phase Commit Protocol

local TwoPhaseCoordinator = {}
TwoPhaseCoordinator.__index = TwoPhaseCoordinator

function TwoPhaseCoordinator.new(transaction_id)
    return setmetatable({
        txn_id = transaction_id,
        participants = {},
        state = "INIT",
        votes = {},
    }, TwoPhaseCoordinator)
end

function TwoPhaseCoordinator:add_participant(participant)
    self.participants[#self.participants+1] = participant
end

-- Phase 1: Prepare
function TwoPhaseCoordinator:prepare()
    self.state = "PREPARING"
    print(string.format("\n[%s] Phase 1: PREPARE", self.txn_id))
    
    for _, p in ipairs(self.participants) do
        local vote = p:prepare(self.txn_id)
        self.votes[p.id] = vote
        print(string.format("  Participant %s votes: %s", p.id, vote))
    end
    
    -- ตรวจสอบ votes
    local all_yes = true
    for _, vote in pairs(self.votes) do
        if vote ~= "YES" then
            all_yes = false
            break
        end
    end
    
    return all_yes
end

-- Phase 2: Commit or Abort
function TwoPhaseCoordinator:commit_or_abort(should_commit)
    if should_commit then
        self.state = "COMMITTING"
        print(string.format("[%s] Phase 2: COMMIT", self.txn_id))
    else
        self.state = "ABORTING"
        print(string.format("[%s] Phase 2: ABORT", self.txn_id))
    end
    
    for _, p in ipairs(self.participants) do
        if should_commit then
            p:commit(self.txn_id)
        else
            p:abort(self.txn_id)
        end
    end
    
    self.state = should_commit and "COMMITTED" or "ABORTED"
end

function TwoPhaseCoordinator:execute()
    local all_yes = self:prepare()
    self:commit_or_abort(all_yes)
    return self.state == "COMMITTED"
end

-- Participant
local TwoPhaseParticipant = {}
TwoPhaseParticipant.__index = TwoPhaseParticipant

function TwoPhaseParticipant.new(id, fail_probability)
    return setmetatable({
        id = id,
        fail_prob = fail_probability or 0.1,
        pending = {},
        data = {},
    }, TwoPhaseParticipant)
end

function TwoPhaseParticipant:prepare(txn_id)
    -- ตรวจสอบว่า transaction สามารถ execute ได้
    if math.random() < self.fail_prob then
        return "NO"  -- จะ abort
    end
    
    -- Lock resources, write to WAL (Write-Ahead Log)
    self.pending[txn_id] = {
        state = "PREPARED",
        locked_at = os.time(),
    }
    
    return "YES"
end

function TwoPhaseParticipant:commit(txn_id)
    local pending = self.pending[txn_id]
    if pending then
        -- Apply changes
        self.pending[txn_id] = nil
        print(string.format("  Participant %s COMMITTED %s", self.id, txn_id))
    end
end

function TwoPhaseParticipant:abort(txn_id)
    local pending = self.pending[txn_id]
    if pending then
        -- Rollback changes
        self.pending[txn_id] = nil
        print(string.format("  Participant %s ABORTED %s", self.id, txn_id))
    end
end

-- ทดสอบ 2PC
local function test_2pc()
    print("\n=== Two-Phase Commit Demo ===")
    math.randomseed(42)
    
    -- สร้าง transaction
    local coord = TwoPhaseCoordinator.new("TXN-001")
    
    -- เพิ่ม participants (databases)
    coord:add_participant(TwoPhaseParticipant.new("DB-A", 0.1))
    coord:add_participant(TwoPhaseParticipant.new("DB-B", 0.1))
    coord:add_participant(TwoPhaseParticipant.new("DB-C", 0.1))
    
    local success = coord:execute()
    print(string.format("\nTransaction result: %s", 
        success and "COMMITTED" or "ABORTED"))
    
    -- Test with high failure rate
    print("\n--- High failure scenario ---")
    local coord2 = TwoPhaseCoordinator.new("TXN-002")
    coord2:add_participant(TwoPhaseParticipant.new("DB-X", 0.8))
    coord2:add_participant(TwoPhaseParticipant.new("DB-Y", 0.8))
    coord2:add_participant(TwoPhaseParticipant.new("DB-Z", 0.8))
    
    success = coord2:execute()
    print(string.format("Transaction result: %s",
        success and "COMMITTED" or "ABORTED"))
end

test_2pc()
```

---

## 84.8 Saga Pattern

```lua
-- Saga Pattern: สำหรับ distributed transactions ที่ไม่ใช้ 2PC
-- ใช้ sequence ของ local transactions + compensating transactions

local Saga = {}
Saga.__index = Saga

function Saga.new(name)
    return setmetatable({
        name = name,
        steps = {},
        completed_steps = {},
        state = "PENDING",
    }, Saga)
end

function Saga:add_step(name, action, compensate)
    self.steps[#self.steps+1] = {
        name = name,
        action = action,
        compensate = compensate,
    }
end

function Saga:execute()
    self.state = "RUNNING"
    print(string.format("\n=== Saga: %s ===", self.name))
    
    for i, step in ipairs(self.steps) do
        print(string.format("  Step %d: %s", i, step.name))
        
        local ok, err = pcall(step.action)
        
        if ok then
            self.completed_steps[#self.completed_steps+1] = step
            print(string.format("    ✓ Success"))
        else
            print(string.format("    ✗ Failed: %s", tostring(err)))
            print("    Starting compensation...")
            
            -- Compensate in reverse order
            for j = #self.completed_steps, 1, -1 do
                local completed = self.completed_steps[j]
                print(string.format("    Compensating: %s", completed.name))
                pcall(completed.compensate)
            end
            
            self.state = "FAILED"
            return false, err
        end
    end
    
    self.state = "COMPLETED"
    return true
end

-- ตัวอย่าง: Order fulfillment saga
local function order_saga_example()
    local saga = Saga.new("Order Fulfillment")
    
    -- Order state (shared)
    local order = {
        id = "ORD-123",
        reserved_inventory = false,
        payment_processed = false,
        shipped = false,
    }
    
    -- Step 1: Reserve inventory
    saga:add_step(
        "Reserve Inventory",
        function()
            -- Simulate inventory check
            if math.random() < 0.1 then
                error("Inventory not available")
            end
            order.reserved_inventory = true
            print("      Inventory reserved for order " .. order.id)
        end,
        function()
            -- Compensate: release inventory
            order.reserved_inventory = false
            print("      Inventory released for order " .. order.id)
        end
    )
    
    -- Step 2: Process payment
    saga:add_step(
        "Process Payment",
        function()
            if math.random() < 0.2 then
                error("Payment declined")
            end
            order.payment_processed = true
            print("      Payment processed for order " .. order.id)
        end,
        function()
            -- Compensate: refund
            order.payment_processed = false
            print("      Payment refunded for order " .. order.id)
        end
    )
    
    -- Step 3: Ship order
    saga:add_step(
        "Ship Order",
        function()
            if math.random() < 0.1 then
                error("Shipping service unavailable")
            end
            order.shipped = true
            print("      Order shipped: " .. order.id)
        end,
        function()
            -- Compensate: cancel shipment
            order.shipped = false
            print("      Shipment cancelled for order " .. order.id)
        end
    )
    
    -- Execute saga
    math.randomseed(42)
    local ok, err = saga:execute()
    
    print(string.format("\nSaga result: %s", ok and "SUCCESS" or ("FAILED: " .. tostring(err))))
    print(string.format("Order state: inventory=%s payment=%s shipped=%s",
        tostring(order.reserved_inventory),
        tostring(order.payment_processed),
        tostring(order.shipped)))
end

order_saga_example()
```

---

## 84.9 Event Sourcing

```lua
-- Event Sourcing: เก็บ events แทนที่จะเก็บ current state

local EventStore = {}
EventStore.__index = EventStore

function EventStore.new()
    return setmetatable({
        events = {},
        subscribers = {},
    }, EventStore)
end

function EventStore:append(stream_id, event_type, data, metadata)
    local event = {
        id = #self.events + 1,
        stream_id = stream_id,
        type = event_type,
        data = data,
        metadata = metadata or {},
        timestamp = os.time(),
        sequence = #self:get_stream(stream_id) + 1,
    }
    
    self.events[#self.events+1] = event
    
    -- Notify subscribers
    for _, sub in ipairs(self.subscribers) do
        if not sub.stream_id or sub.stream_id == stream_id then
            sub.handler(event)
        end
    end
    
    return event
end

function EventStore:get_stream(stream_id)
    local stream = {}
    for _, e in ipairs(self.events) do
        if e.stream_id == stream_id then
            stream[#stream+1] = e
        end
    end
    return stream
end

function EventStore:subscribe(handler, stream_id)
    self.subscribers[#self.subscribers+1] = {
        handler = handler,
        stream_id = stream_id,
    }
end

-- Aggregate (domain object rebuilt from events)
local BankAccount = {}
BankAccount.__index = BankAccount

function BankAccount.new(account_id)
    return setmetatable({
        id = account_id,
        balance = 0,
        status = "closed",
        owner = nil,
        version = 0,
    }, BankAccount)
end

-- Event handlers
local event_handlers = {
    AccountOpened = function(account, data)
        account.status = "open"
        account.owner = data.owner
        account.balance = data.initial_balance or 0
    end,
    
    MoneyDeposited = function(account, data)
        account.balance = account.balance + data.amount
    end,
    
    MoneyWithdrawn = function(account, data)
        account.balance = account.balance - data.amount
    end,
    
    AccountClosed = function(account, data)
        account.status = "closed"
    end,
}

function BankAccount:apply(event)
    local handler = event_handlers[event.type]
    if handler then
        handler(self, event.data)
    end
    self.version = self.version + 1
end

function BankAccount.load_from_events(account_id, events)
    local account = BankAccount.new(account_id)
    for _, event in ipairs(events) do
        account:apply(event)
    end
    return account
end

-- Command handlers
local AccountService = {}
AccountService.__index = AccountService

function AccountService.new(event_store)
    return setmetatable({store = event_store}, AccountService)
end

function AccountService:open_account(account_id, owner, initial_balance)
    local account = self:load(account_id)
    
    if account.status == "open" then
        return false, "Account already open"
    end
    
    self.store:append(account_id, "AccountOpened", {
        owner = owner,
        initial_balance = initial_balance or 0,
    })
    
    return true
end

function AccountService:deposit(account_id, amount)
    local account = self:load(account_id)
    
    if account.status ~= "open" then
        return false, "Account not open"
    end
    
    if amount <= 0 then
        return false, "Amount must be positive"
    end
    
    self.store:append(account_id, "MoneyDeposited", {amount = amount})
    return true
end

function AccountService:withdraw(account_id, amount)
    local account = self:load(account_id)
    
    if account.status ~= "open" then
        return false, "Account not open"
    end
    
    if amount > account.balance then
        return false, "Insufficient funds"
    end
    
    self.store:append(account_id, "MoneyWithdrawn", {amount = amount})
    return true
end

function AccountService:load(account_id)
    local events = self.store:get_stream(account_id)
    return BankAccount.load_from_events(account_id, events)
end

function AccountService:get_balance(account_id)
    return self:load(account_id).balance
end

-- ทดสอบ Event Sourcing
local function test_event_sourcing()
    print("\n=== Event Sourcing Demo ===")
    
    local store = EventStore.new()
    
    -- Subscribe to events
    store:subscribe(function(e)
        print(string.format("  Event: [%s] %s - %s",
            e.stream_id, e.type,
            type(e.data) == "table" and 
                (e.data.owner or tostring(e.data.amount) or "{}") or
                tostring(e.data)))
    end)
    
    local service = AccountService.new(store)
    
    -- Operations
    local ok, err
    ok, err = service:open_account("ACC-001", "Alice", 1000)
    print("Open:", ok, err)
    
    ok, err = service:deposit("ACC-001", 500)
    print("Deposit 500:", ok, err)
    
    ok, err = service:withdraw("ACC-001", 200)
    print("Withdraw 200:", ok, err)
    
    ok, err = service:withdraw("ACC-001", 2000)  -- ไม่พอ!
    print("Withdraw 2000:", ok, err)
    
    -- ดู current balance
    print("\nFinal balance:", service:get_balance("ACC-001"))
    
    -- Replay events เพื่อ rebuild state
    print("\n--- Replaying events ---")
    local events = store:get_stream("ACC-001")
    print("Total events:", #events)
    local account = BankAccount.load_from_events("ACC-001", events)
    print("Rebuilt balance:", account.balance)
    print("Account owner:", account.owner)
end

test_event_sourcing()
```

---

## 84.10 CQRS Pattern

```lua
-- Command Query Responsibility Segregation
-- Write side (Commands) และ Read side (Queries) แยกกัน

local CQRS = {}

-- Command side: ใช้ normalized data model
local CommandStore = {}
CommandStore.__index = CommandStore

function CommandStore.new()
    return setmetatable({
        users = {},
        orders = {},
        products = {},
    }, CommandStore)
end

function CommandStore:execute_command(cmd)
    if cmd.type == "CreateUser" then
        if self.users[cmd.data.id] then
            return false, "User already exists"
        end
        self.users[cmd.data.id] = {
            id = cmd.data.id,
            name = cmd.data.name,
            email = cmd.data.email,
            created_at = os.time(),
        }
        return true
        
    elseif cmd.type == "PlaceOrder" then
        local user = self.users[cmd.data.user_id]
        if not user then
            return false, "User not found"
        end
        
        local order_id = "ORD-" .. (#self.orders + 1)
        self.orders[order_id] = {
            id = order_id,
            user_id = cmd.data.user_id,
            items = cmd.data.items,
            total = cmd.data.total,
            status = "pending",
            created_at = os.time(),
        }
        return true, order_id
    end
end

-- Query side: ใช้ denormalized data สำหรับ fast reads
local QueryStore = {}
QueryStore.__index = QueryStore

function QueryStore.new()
    return setmetatable({
        -- Denormalized views
        user_profiles = {},     -- user + stats
        order_summaries = {},   -- order + user info
    }, QueryStore)
end

-- Update query store จาก events
function QueryStore:handle_event(event_type, data)
    if event_type == "UserCreated" then
        self.user_profiles[data.id] = {
            id = data.id,
            name = data.name,
            email = data.email,
            order_count = 0,
            total_spent = 0,
        }
        
    elseif event_type == "OrderPlaced" then
        -- Update user stats
        local profile = self.user_profiles[data.user_id]
        if profile then
            profile.order_count = profile.order_count + 1
            profile.total_spent = profile.total_spent + (data.total or 0)
        end
        
        -- Store order summary
        self.order_summaries[data.order_id] = {
            id = data.order_id,
            user_id = data.user_id,
            user_name = profile and profile.name or "Unknown",
            total = data.total,
            status = "pending",
        }
    end
end

-- Query methods
function QueryStore:get_user_profile(user_id)
    return self.user_profiles[user_id]
end

function QueryStore:get_user_orders(user_id)
    local orders = {}
    for _, order in pairs(self.order_summaries) do
        if order.user_id == user_id then
            orders[#orders+1] = order
        end
    end
    return orders
end

function QueryStore:get_top_customers(limit)
    local profiles = {}
    for _, p in pairs(self.user_profiles) do
        profiles[#profiles+1] = p
    end
    table.sort(profiles, function(a, b) 
        return a.total_spent > b.total_spent 
    end)
    local result = {}
    for i = 1, math.min(limit, #profiles) do
        result[i] = profiles[i]
    end
    return result
end

-- ทดสอบ CQRS
local function test_cqrs()
    print("\n=== CQRS Demo ===")
    
    local cmd_store = CommandStore.new()
    local query_store = QueryStore.new()
    
    -- Event bus (simplified)
    local function publish_event(event_type, data)
        query_store:handle_event(event_type, data)
    end
    
    -- Commands
    cmd_store:execute_command({
        type = "CreateUser",
        data = {id = "U1", name = "Alice", email = "alice@example.com"}
    })
    publish_event("UserCreated", {id="U1", name="Alice", email="alice@example.com"})
    
    cmd_store:execute_command({
        type = "CreateUser",
        data = {id = "U2", name = "Bob", email = "bob@example.com"}
    })
    publish_event("UserCreated", {id="U2", name="Bob", email="bob@example.com"})
    
    local ok, order_id = cmd_store:execute_command({
        type = "PlaceOrder",
        data = {
            user_id = "U1",
            items = {"laptop", "mouse"},
            total = 1500,
        }
    })
    
    if ok then
        publish_event("OrderPlaced", {
            order_id = order_id,
            user_id = "U1",
            total = 1500,
        })
    end
    
    -- ทำ orders หลายรายการ
    for i = 1, 3 do
        ok, order_id = cmd_store:execute_command({
            type = "PlaceOrder",
            data = {user_id = "U2", items = {"item" .. i}, total = 100 * i}
        })
        if ok then
            publish_event("OrderPlaced", {
                order_id = order_id,
                user_id = "U2",
                total = 100 * i,
            })
        end
    end
    
    -- Queries (fast reads from denormalized data)
    print("\n--- Queries ---")
    
    local profile = query_store:get_user_profile("U1")
    if profile then
        print(string.format("Alice: %d orders, $%.0f spent",
            profile.order_count, profile.total_spent))
    end
    
    local bob_orders = query_store:get_user_orders("U2")
    print(string.format("Bob's orders: %d", #bob_orders))
    
    local top = query_store:get_top_customers(2)
    print("\nTop customers:")
    for i, c in ipairs(top) do
        print(string.format("  %d. %s ($%.0f)", i, c.name, c.total_spent))
    end
end

test_cqrs()
```

---

## 84.11 Implementing Distributed Cache ใน Lua

```lua
-- Distributed cache ด้วย consistent hashing

-- Hash function
local function fnv1a_hash(key)
    local hash = 2166136261  -- FNV offset basis
    for i = 1, #key do
        hash = hash ~ key:byte(i)
        hash = (hash * 16777619) & 0xFFFFFFFF  -- FNV prime
    end
    return hash
end

-- Consistent Hash Ring
local HashRing = {}
HashRing.__index = HashRing

function HashRing.new(replicas)
    return setmetatable({
        replicas = replicas or 150,  -- virtual nodes per real node
        ring = {},
        nodes = {},
    }, HashRing)
end

function HashRing:add_node(node_id)
    self.nodes[node_id] = true
    
    -- เพิ่ม virtual nodes
    for i = 1, self.replicas do
        local key = node_id .. ":" .. i
        local hash = fnv1a_hash(key)
        self.ring[hash] = node_id
    end
end

function HashRing:remove_node(node_id)
    self.nodes[node_id] = nil
    
    for hash, nid in pairs(self.ring) do
        if nid == node_id then
            self.ring[hash] = nil
        end
    end
end

function HashRing:get_node(key)
    if not next(self.ring) then return nil end
    
    local hash = fnv1a_hash(key)
    
    -- หา hash ที่ใกล้ที่สุดใน ring (clockwise)
    local sorted_hashes = {}
    for h in pairs(self.ring) do
        sorted_hashes[#sorted_hashes+1] = h
    end
    table.sort(sorted_hashes)
    
    for _, h in ipairs(sorted_hashes) do
        if h >= hash then
            return self.ring[h]
        end
    end
    
    -- Wrap around
    return self.ring[sorted_hashes[1]]
end

function HashRing:get_nodes_for_key(key, n)
    local nodes_list = {}
    local seen = {}
    
    -- ใช้ virtual nodes เพื่อหา n unique real nodes
    local hash = fnv1a_hash(key)
    
    local sorted_hashes = {}
    for h in pairs(self.ring) do
        sorted_hashes[#sorted_hashes+1] = h
    end
    table.sort(sorted_hashes)
    
    -- Find starting position
    local start_idx = 1
    for i, h in ipairs(sorted_hashes) do
        if h >= hash then
            start_idx = i
            break
        end
    end
    
    -- Collect n unique nodes
    local count = 0
    local idx = start_idx
    while count < n do
        local h = sorted_hashes[idx]
        local node = self.ring[h]
        
        if not seen[node] then
            seen[node] = true
            nodes_list[#nodes_list+1] = node
            count = count + 1
        end
        
        idx = (idx % #sorted_hashes) + 1
        if idx == start_idx then break end  -- full circle
    end
    
    return nodes_list
end

-- Distributed Cache Node
local CacheNode = {}
CacheNode.__index = CacheNode

function CacheNode.new(id, capacity)
    return setmetatable({
        id = id,
        capacity = capacity or 1000,
        data = {},
        count = 0,
        hits = 0,
        misses = 0,
    }, CacheNode)
end

function CacheNode:set(key, value, ttl)
    if self.count >= self.capacity then
        self:evict()
    end
    
    self.data[key] = {
        value = value,
        expires_at = ttl and (os.time() + ttl) or math.huge,
        accessed_at = os.time(),
        created_at = os.time(),
    }
    self.count = self.count + 1
end

function CacheNode:get(key)
    local entry = self.data[key]
    if not entry then
        self.misses = self.misses + 1
        return nil
    end
    
    if entry.expires_at < os.time() then
        self.data[key] = nil
        self.count = self.count - 1
        self.misses = self.misses + 1
        return nil
    end
    
    entry.accessed_at = os.time()
    self.hits = self.hits + 1
    return entry.value
end

function CacheNode:delete(key)
    if self.data[key] then
        self.data[key] = nil
        self.count = self.count - 1
        return true
    end
    return false
end

function CacheNode:evict()
    -- LRU eviction
    local oldest_key = nil
    local oldest_time = math.huge
    
    for k, entry in pairs(self.data) do
        if entry.accessed_at < oldest_time then
            oldest_time = entry.accessed_at
            oldest_key = k
        end
    end
    
    if oldest_key then
        self.data[oldest_key] = nil
        self.count = self.count - 1
    end
end

function CacheNode:stats()
    local total = self.hits + self.misses
    return {
        id = self.id,
        count = self.count,
        hits = self.hits,
        misses = self.misses,
        hit_rate = total > 0 and (self.hits / total * 100) or 0,
    }
end

-- Distributed Cache Cluster
local DistributedCache = {}
DistributedCache.__index = DistributedCache

function DistributedCache.new(replication_factor)
    return setmetatable({
        ring = HashRing.new(150),
        nodes = {},
        replication_factor = replication_factor or 2,
    }, DistributedCache)
end

function DistributedCache:add_node(node)
    self.nodes[node.id] = node
    self.ring:add_node(node.id)
end

function DistributedCache:remove_node(node_id)
    self.nodes[node_id] = nil
    self.ring:remove_node(node_id)
end

function DistributedCache:set(key, value, ttl)
    local node_ids = self.ring:get_nodes_for_key(key, self.replication_factor)
    
    for _, node_id in ipairs(node_ids) do
        local node = self.nodes[node_id]
        if node then
            node:set(key, value, ttl)
        end
    end
end

function DistributedCache:get(key)
    local node_ids = self.ring:get_nodes_for_key(key, self.replication_factor)
    
    for _, node_id in ipairs(node_ids) do
        local node = self.nodes[node_id]
        if node then
            local val = node:get(key)
            if val ~= nil then
                return val
            end
        end
    end
    
    return nil
end

function DistributedCache:delete(key)
    local node_ids = self.ring:get_nodes_for_key(key, self.replication_factor)
    
    for _, node_id in ipairs(node_ids) do
        local node = self.nodes[node_id]
        if node then
            node:delete(key)
        end
    end
end

function DistributedCache:print_stats()
    print("\n--- Cache Stats ---")
    for _, node in pairs(self.nodes) do
        local s = node:stats()
        print(string.format("  %s: count=%d hits=%d misses=%d rate=%.1f%%",
            s.id, s.count, s.hits, s.misses, s.hit_rate))
    end
end

-- ทดสอบ
local function test_distributed_cache()
    print("\n=== Distributed Cache Demo ===")
    
    local cache = DistributedCache.new(2)
    
    -- เพิ่ม nodes
    for i = 1, 4 do
        cache:add_node(CacheNode.new("cache" .. i, 500))
    end
    
    -- Test key distribution
    local node_counts = {}
    for i = 1, 1000 do
        local key = "user:" .. i
        local primary = cache.ring:get_node(key)
        node_counts[primary] = (node_counts[primary] or 0) + 1
    end
    
    print("Key distribution (1000 keys):")
    for node, count in pairs(node_counts) do
        print(string.format("  %s: %d keys (%.1f%%)", node, count, count/10))
    end
    
    -- Set/Get operations
    for i = 1, 100 do
        cache:set("key:" .. i, {value = i, data = "test"}, 3600)
    end
    
    -- Access some
    local hits = 0
    for i = 1, 200 do
        local v = cache:get("key:" .. i)
        if v then hits = hits + 1 end
    end
    
    print(string.format("\nAccess test: %d/200 hits", hits))
    
    cache:print_stats()
    
    -- Node failure simulation
    print("\n--- Node Failure Simulation ---")
    cache:remove_node("cache2")
    
    -- Data on cache2 ที่ไม่ได้ replicate จะหาย
    hits = 0
    for i = 1, 100 do
        local v = cache:get("key:" .. i)
        if v then hits = hits + 1 end
    end
    print(string.format("After node failure: %d/100 hits (replication=%d)",
        hits, cache.replication_factor))
end

test_distributed_cache()
```

---

## 84.12 Circuit Breaker Pattern

```lua
-- Circuit Breaker: ป้องกัน cascading failures ใน distributed systems

local CircuitBreaker = {}
CircuitBreaker.__index = CircuitBreaker

local CB_CLOSED   = "CLOSED"    -- ปกติ: allow requests
local CB_OPEN     = "OPEN"      -- ปัญหา: reject requests
local CB_HALF_OPEN = "HALF_OPEN" -- ทดสอบ: allow 1 request

function CircuitBreaker.new(options)
    options = options or {}
    return setmetatable({
        state = CB_CLOSED,
        failure_count = 0,
        success_count = 0,
        last_failure_time = nil,
        
        -- Config
        failure_threshold = options.failure_threshold or 5,
        success_threshold = options.success_threshold or 2,
        timeout_ms = options.timeout_ms or 60000,
        
        -- Stats
        total_requests = 0,
        total_failures = 0,
        total_successes = 0,
        rejected_requests = 0,
    }, CircuitBreaker)
end

function CircuitBreaker:call(fn)
    self.total_requests = self.total_requests + 1
    
    -- Check state
    if self.state == CB_OPEN then
        -- Check if timeout has passed
        local elapsed = (os.clock() * 1000) - (self.last_failure_time or 0)
        if elapsed >= self.timeout_ms then
            self.state = CB_HALF_OPEN
            self.success_count = 0
        else
            self.rejected_requests = self.rejected_requests + 1
            return false, "Circuit breaker OPEN: service unavailable"
        end
    end
    
    -- Execute
    local ok, result = pcall(fn)
    
    if ok then
        self:on_success()
        return true, result
    else
        self:on_failure(result)
        return false, result
    end
end

function CircuitBreaker:on_success()
    self.total_successes = self.total_successes + 1
    self.failure_count = 0
    
    if self.state == CB_HALF_OPEN then
        self.success_count = self.success_count + 1
        if self.success_count >= self.success_threshold then
            self.state = CB_CLOSED
            print("Circuit breaker: HALF_OPEN → CLOSED (recovered!)")
        end
    end
end

function CircuitBreaker:on_failure(err)
    self.total_failures = self.total_failures + 1
    self.failure_count = self.failure_count + 1
    self.last_failure_time = os.clock() * 1000
    
    if self.state == CB_HALF_OPEN then
        self.state = CB_OPEN
        print("Circuit breaker: HALF_OPEN → OPEN (still failing)")
    elseif self.state == CB_CLOSED then
        if self.failure_count >= self.failure_threshold then
            self.state = CB_OPEN
            print(string.format("Circuit breaker: CLOSED → OPEN (failures=%d)",
                self.failure_count))
        end
    end
end

function CircuitBreaker:get_stats()
    return {
        state = self.state,
        failure_count = self.failure_count,
        total_requests = self.total_requests,
        total_failures = self.total_failures,
        total_successes = self.total_successes,
        rejected_requests = self.rejected_requests,
        error_rate = self.total_requests > 0 and 
            (self.total_failures / self.total_requests * 100) or 0,
    }
end

-- ทดสอบ Circuit Breaker
local function test_circuit_breaker()
    print("\n=== Circuit Breaker Demo ===")
    
    local cb = CircuitBreaker.new({
        failure_threshold = 3,
        success_threshold = 2,
        timeout_ms = 100,  -- 100ms สำหรับ demo
    })
    
    -- Simulate unstable service
    local fail_count = 0
    local function flaky_service()
        fail_count = fail_count + 1
        if fail_count <= 5 then
            error("Service temporarily unavailable")
        end
        return "OK"
    end
    
    -- Make requests
    for i = 1, 10 do
        local ok, result = cb:call(flaky_service)
        print(string.format("Request %d: %s [CB: %s]",
            i, ok and result or "FAILED: " .. tostring(result), cb.state))
    end
    
    print("\n--- Waiting for circuit timeout ---")
    -- Simulate waiting (in real code: actual sleep)
    cb.last_failure_time = 0  -- หลอก timeout
    
    -- Try again (should be HALF_OPEN)
    for i = 11, 14 do
        local ok, result = cb:call(flaky_service)
        print(string.format("Request %d: %s [CB: %s]",
            i, ok and result or "FAILED: " .. tostring(result), cb.state))
    end
    
    -- Stats
    local stats = cb:get_stats()
    print(string.format("\nStats: total=%d success=%d fail=%d rejected=%d rate=%.1f%%",
        stats.total_requests, stats.total_successes,
        stats.total_failures, stats.rejected_requests,
        stats.error_rate))
end

test_circuit_breaker()
```

---

## 84.13 Bulkhead Pattern

```lua
-- Bulkhead Pattern: แยก resources เพื่อป้องกัน cascading failures

local ThreadPool = {}
ThreadPool.__index = ThreadPool

-- Simulate thread pool ด้วย coroutines
function ThreadPool.new(name, size)
    local self = setmetatable({}, ThreadPool)
    self.name = name
    self.size = size
    self.active = 0
    self.queue = {}
    self.rejected = 0
    self.completed = 0
    self.failed = 0
    return self
end

function ThreadPool:submit(task)
    if self.active >= self.size then
        self.rejected = self.rejected + 1
        return false, "Thread pool '" .. self.name .. "' is full"
    end
    
    self.active = self.active + 1
    
    -- Execute task
    local ok, err = pcall(task)
    
    self.active = self.active - 1
    
    if ok then
        self.completed = self.completed + 1
    else
        self.failed = self.failed + 1
    end
    
    return ok, err
end

function ThreadPool:stats()
    return {
        name = self.name,
        active = self.active,
        size = self.size,
        completed = self.completed,
        failed = self.failed,
        rejected = self.rejected,
    }
end

-- Bulkhead สำหรับ services ต่างๆ
local function test_bulkhead()
    print("\n=== Bulkhead Pattern Demo ===")
    
    -- แยก thread pools สำหรับ services ต่างๆ
    local payment_pool = ThreadPool.new("payments", 10)
    local inventory_pool = ThreadPool.new("inventory", 20)
    local notification_pool = ThreadPool.new("notifications", 5)
    
    -- Simulate high load บน notifications
    -- ไม่ทำให้ payments/inventory ได้รับผลกระทบ
    
    local function process_payment(order_id)
        -- Simulate some work
        local sum = 0
        for i = 1, 1000 do sum = sum + i end
        return true
    end
    
    local function send_notification(user_id)
        -- Simulate slow service
        local sum = 0
        for i = 1, 100000 do sum = sum + i end
        return true
    end
    
    -- ส่ง 50 requests พร้อมกัน (simulate)
    local results = {payments=0, inventory=0, notifications=0}
    local rejected = {payments=0, inventory=0, notifications=0}
    
    for i = 1, 50 do
        local ok, err = payment_pool:submit(function()
            process_payment("order-" .. i)
        end)
        if ok then results.payments = results.payments + 1
        else rejected.payments = rejected.payments + 1 end
        
        ok, err = inventory_pool:submit(function()
            local sum = 0
            for j = 1, 500 do sum = sum + j end
        end)
        if ok then results.inventory = results.inventory + 1
        else rejected.inventory = rejected.inventory + 1 end
        
        ok, err = notification_pool:submit(function()
            send_notification("user-" .. i)
        end)
        if ok then results.notifications = results.notifications + 1
        else rejected.notifications = rejected.notifications + 1 end
    end
    
    print("Results:")
    for service, count in pairs(results) do
        print(string.format("  %s: %d completed, %d rejected",
            service, count, rejected[service]))
    end
    
    print("\nEven if notifications are overloaded, payments work fine!")
end

test_bulkhead()
```

---

## 84.14 สรุป Distributed Systems Patterns

```lua
-- สรุป patterns ที่เราเรียนในบทนี้

local patterns_summary = {
    {
        name = "CAP Theorem",
        description = "Cannot have C, A, P simultaneously. Choose 2.",
        use_when = "Designing distributed databases"
    },
    {
        name = "Eventual Consistency",
        description = "All nodes converge eventually",
        use_when = "High availability > strong consistency (shopping cart, DNS)"
    },
    {
        name = "Vector Clocks",
        description = "Track causal ordering across nodes",
        use_when = "Need to detect causality and conflicts"
    },
    {
        name = "Distributed Locks (Redlock)",
        description = "Distributed mutual exclusion",
        use_when = "Prevent concurrent access to shared resources"
    },
    {
        name = "Raft Consensus",
        description = "Leader election + log replication",
        use_when = "Need strong consistency and fault tolerance"
    },
    {
        name = "Two-Phase Commit",
        description = "Atomic distributed transactions",
        use_when = "ACID transactions across multiple databases"
    },
    {
        name = "Saga Pattern",
        description = "Long-running transactions with compensation",
        use_when = "Business transactions across microservices"
    },
    {
        name = "Event Sourcing",
        description = "Store events not current state",
        use_when = "Audit trails, time travel, event replay"
    },
    {
        name = "CQRS",
        description = "Separate read and write models",
        use_when = "Complex queries or high read/write ratio difference"
    },
    {
        name = "Circuit Breaker",
        description = "Prevent cascading failures",
        use_when = "Calling unreliable external services"
    },
    {
        name = "Bulkhead",
        description = "Isolate failures with resource limits",
        use_when = "Protecting critical services from noisy neighbors"
    },
    {
        name = "Consistent Hashing",
        description = "Distribute load with minimal reshuffling",
        use_when = "Distributed caches, load balancers"
    },
}

print("\n=== Distributed Systems Patterns Summary ===")
for i, p in ipairs(patterns_summary) do
    print(string.format("\n%d. %s", i, p.name))
    print("   " .. p.description)
    print("   Use when: " .. p.use_when)
end
```

---

## แบบฝึกหัด

1. Implement Gossip Protocol สำหรับ membership detection ใน cluster

2. เพิ่ม anti-entropy mechanism ใน `EventuallyConsistentKV` เพื่อ sync ข้อมูลที่ขาดหาย

3. Implement Read Repair ใน distributed cache เมื่อ node หนึ่งมีข้อมูลเก่า

4. สร้าง timeout และ retry mechanism ที่มี exponential backoff

5. Implement leader election ด้วย Bully algorithm (ง่ายกว่า Raft)

6. เพิ่ม snapshot mechanism ใน Event Store เพื่อ speed up replay

7. Implement rate limiter แบบ Token Bucket สำหรับ distributed system

8. สร้าง service registry ที่ใช้ consistent hashing สำหรับ load balancing

---

## สรุป

ในบทนี้เราได้เรียนรู้ concepts สำคัญของ distributed systems ตั้งแต่:

- **CAP Theorem**: fundamental trade-offs
- **Consistency Models**: strong, eventual, causal
- **Vector Clocks & Lamport Timestamps**: causality tracking
- **Distributed Locks (Redlock)**: mutual exclusion ใน cluster
- **Raft**: consensus algorithm
- **Two-Phase Commit**: distributed transactions
- **Saga Pattern**: long-running transaction compensation
- **Event Sourcing**: immutable event log
- **CQRS**: separate read/write models
- **Distributed Cache**: consistent hashing
- **Circuit Breaker**: failure isolation
- **Bulkhead**: resource isolation

ทุก patterns มี trade-offs ที่ต้องพิจารณาตาม use case ไม่มี one-size-fits-all solution ใน distributed systems

ในบทต่อไป (บทที่ 85) เราจะไปดู High-Performance Lua - การ optimize โค้ดให้ทำงานได้เร็วสูงสุด
