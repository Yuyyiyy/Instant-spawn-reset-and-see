-- HYBRID Ultra-Performance Instant Respawn System
-- Combining speed optimization with bulletproof reliability

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer

-- Performance optimizations
local taskSpawn = task.spawn
local taskWait = task.wait
local tick = tick

-- Advanced state management
local RespawnState = {
    isRespawning = false,
    lastDeath = 0,
    connectionId = 0,
    guideRemote = nil
}

-- Connection management
local ConnectionCache = {}

-- Pre-cache Guide remote for maximum speed
local function CacheGuideRemote()
    if not RespawnState.guideRemote then
        RespawnState.guideRemote = ReplicatedStorage:FindFirstChild("Guide")
        if not RespawnState.guideRemote then
            taskSpawn(function()
                RespawnState.guideRemote = ReplicatedStorage:WaitForChild("Guide", 10)
            end)
        end
    end
end

-- Ultra-optimized respawn function with smart fallbacks
local function InstantRespawn()
    local currentTime = tick()
    
    -- Prevent spam calls (frame-perfect protection)
    if RespawnState.isRespawning or (currentTime - RespawnState.lastDeath) < 0.05 then
        return
    end
    
    RespawnState.isRespawning = true
    RespawnState.lastDeath = currentTime
    
    taskSpawn(function()
        -- Primary method: Cached Guide remote (fastest)
        if RespawnState.guideRemote and RespawnState.guideRemote:IsA("RemoteEvent") then
            RespawnState.guideRemote:FireServer()
        else
            -- Fallback: Search for Guide remote
            local guide = ReplicatedStorage:FindFirstChild("Guide")
            if guide and guide:IsA("RemoteEvent") then
                guide:FireServer()
                RespawnState.guideRemote = guide -- Cache it
            else
                -- Emergency fallback: LoadCharacter
                pcall(function()
                    LocalPlayer:LoadCharacter()
                end)
            end
        end
        
        -- Reset respawn flag
        taskWait(0.05)
        RespawnState.isRespawning = false
    end)
end

-- Hybrid connection system (multiple detection + single management)
local function SetupHybridConnections(character)
    if not character then return end
    
    -- Increment connection ID for bulletproof tracking
    RespawnState.connectionId = RespawnState.connectionId + 1
    local connectionId = RespawnState.connectionId
    
    -- Clean previous connections
    if ConnectionCache[character] then
        for _, conn in pairs(ConnectionCache[character]) do
            if conn and conn.Connected then
                conn:Disconnect()
            end
        end
    end
    ConnectionCache[character] = {}
    
    local function setupConnections()
        local humanoid = character:FindFirstChildWhichIsA("Humanoid")
        if humanoid then
            -- Primary: Died event (most reliable)
            local deathConn = humanoid.Died:Connect(function()
                if RespawnState.connectionId == connectionId then
                    InstantRespawn()
                end
            end)
            
            -- Secondary: Health monitoring (backup detection)
            local healthConn = humanoid.HealthChanged:Connect(function(health)
                if health <= 0 and RespawnState.connectionId == connectionId and not RespawnState.isRespawning then
                    InstantRespawn()
                end
            end)
            
            ConnectionCache[character] = {deathConn, healthConn}
            return true
        end
        return false
    end
    
    -- Immediate setup or wait for humanoid
    if not setupConnections() then
        local attempts = 0
        local setupLoop
        setupLoop = RunService.Heartbeat:Connect(function()
            attempts = attempts + 1
            if setupConnections() or attempts > 150 then -- 2.5 second timeout
                setupLoop:Disconnect()
            end
        end)
    end
end

-- Streamlined character monitoring
local function OnCharacterAdded(character)
    -- Cache Guide remote on first character spawn
    CacheGuideRemote()
    
    -- Wait for essential components
    character:WaitForChild("HumanoidRootPart", 5)
    
    -- Minimal delay for stability
    taskWait()
    
    -- Setup hybrid connections
    SetupHybridConnections(character)
end

-- Lightweight watchdog (reduced overhead)
local function CreateLightWatchdog()
    taskSpawn(function()
        while true do
            taskWait(1) -- Check every second instead of every frame
            
            -- Verify current character has connections
            if LocalPlayer.Character then
                local character = LocalPlayer.Character
                if not ConnectionCache[character] or #ConnectionCache[character] == 0 then
                    SetupHybridConnections(character)
                end
            end
            
            -- Re-cache Guide remote if lost
            if not RespawnState.guideRemote then
                CacheGuideRemote()
            end
        end
    end)
end

-- Initialize system
LocalPlayer.CharacterAdded:Connect(OnCharacterAdded)

-- Handle current character
if LocalPlayer.Character then
    OnCharacterAdded(LocalPlayer.Character)
end

-- Cleanup on character removal
LocalPlayer.CharacterRemoving:Connect(function(character)
    if ConnectionCache[character] then
        for _, conn in pairs(ConnectionCache[character]) do
            if conn and conn.Connected then
                conn:Disconnect()
            end
        end
        ConnectionCache[character] = nil
    end
    RespawnState.isRespawning = false
end)

-- Start lightweight monitoring
CreateLightWatchdog()

-- Global emergency respawn function
_G.ForceRespawn = InstantRespawn

print("HYBRID Ultra-Performance Instant Respawn System Active - Maximum Speed + Reliability")
