if not game:IsLoaded() then
   game.Loaded:Wait()
end

-- Player Hitbox Settings (MODERATE INCREASE)
_G.Disabled = false  
_G.NearSize = 5
_G.MiddleSize = 15
_G.FarSize = 30
_G.NearDistance = 20  
_G.FarDistance = 170

-- Services
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")

-- Configuration
local weaponNames = {"DMR", "Gift of Fire", "Rifle", "Burst Rifle", "Medical Bow", "Vitabow", "Akimbo", "Voltaic Impact"}
local excludedEnemies = {"Jetpacker", "APU", "Ranger", "Platform"}
local bossNames = {"Boss", "BossNoob", "Giant", "Titan"}

-- References
local lp = Players.LocalPlayer
local enemyNames = ReplicatedStorage.Units.Noobs:GetChildren()
local modifiedEnemies = {}
local excludedEnemyInstances = {}
local processedWeapons = {}

-- Utility Functions
local function IsBoss(enemyName)
    for _, bossName in ipairs(bossNames) do
        if string.find(enemyName, bossName) then
            return true
        end
    end
    return false
end

local function IsExcluded(enemyName)
    for _, excluded in ipairs(excludedEnemies) do
        if enemyName == excluded then
            return true
        end
    end
    return false
end

local function CreateHighlight(instance)
    if instance:FindFirstChildOfClass("Highlight") then
        return
    end
   
    local highlight = Instance.new("Highlight")
    
    if IsBoss(instance.Name) then
        highlight.FillColor = Color3.fromRGB(0, 100, 255)
        highlight.OutlineColor = Color3.fromRGB(0, 150, 255)
    else
        highlight.FillColor = Color3.fromRGB(255, 0, 0)
        highlight.OutlineColor = Color3.fromRGB(255, 0, 0)
    end
    
    highlight.FillTransparency = 0.5
    highlight.OutlineTransparency = 0
    highlight.Parent = instance
end

-- Weapon/Ammo Management
local function SetupWeaponAmmo(weapon)
    if processedWeapons[weapon] then return end
    
    processedWeapons[weapon] = true
    
    -- Set initial ammo
    weapon:SetAttribute("Ammo", 999)
    weapon:SetAttribute("MaxAmmo", 999)
    
    -- Listen for changes
    weapon:GetAttributeChangedSignal("Ammo"):Connect(function()
        weapon:SetAttribute("Ammo", 999)
    end)
end

local function SetupInfiniteAmmo()
    local character = lp.Character
    if not character then return end
   
    for _, weaponName in ipairs(weaponNames) do
        local weapon = character:FindFirstChild(weaponName)
        if weapon and not processedWeapons[weapon] then
            SetupWeaponAmmo(weapon)
        end
    end
end

-- Character setup
lp.CharacterAdded:Connect(function(character)
    processedWeapons = {} -- Reset on respawn
    task.wait(0.5) -- Small delay for character to load
    SetupInfiniteAmmo()
end)

-- Initial setup
if lp.Character then
    SetupInfiniteAmmo()
end

-- Enemy Management
local function HandleEnemyDeath(instance, head)
    local highlight = instance:FindFirstChildOfClass("Highlight")
    if highlight then
        highlight:Destroy()
    end
   
    if head and head.Parent then
        head.Size = Vector3.new(2, 1, 1)
        head.Transparency = 1
        head.CanCollide = false
    end
   
    modifiedEnemies[instance] = nil
end

local function PutOnExtend(instance)
    -- Quick name check first
    local isValidEnemy = false
    for _, x in ipairs(enemyNames) do
        if tostring(x) == instance.Name then
            isValidEnemy = true
            break
        end
    end
    
    if not isValidEnemy then return end
    
    -- Handle excluded enemies
    if IsExcluded(instance.Name) then
        excludedEnemyInstances[instance] = true
        CreateHighlight(instance)
        return
    end
    
    -- Skip if already modified
    if modifiedEnemies[instance] then return end
    
    -- Find head without waiting
    local head = instance:FindFirstChild("Head")
    if not head then return end
    
    -- Modify head
    head.Transparency = 0.9
    head.Size = Vector3.new(50.0, 50.0, 50.0)
    head.Massless = true
    head.CanCollide = false
    
    modifiedEnemies[instance] = {head = head}
    CreateHighlight(instance)
    
    -- Setup death handler
    local humanoid = instance:FindFirstChildOfClass("Humanoid")
    if humanoid then
        humanoid.Died:Once(function()
            HandleEnemyDeath(instance, head)
        end)
    end
end

-- Initial enemy processing
for _, v in ipairs(Workspace:GetChildren()) do
    PutOnExtend(v)
end

-- New enemy detection
Workspace.ChildAdded:Connect(function(v)
    task.defer(function()
        PutOnExtend(v)
    end)
end)

-- Player Hitbox Updates (using RenderStepped for smooth updates)
local lastPlayerUpdate = 0
RunService.RenderStepped:Connect(function()
    local now = tick()
    if now - lastPlayerUpdate < 0.5 then return end -- Update every 0.5 seconds
    lastPlayerUpdate = now
    
    if _G.Disabled then return end
    
    local myRoot = lp.Character and lp.Character:FindFirstChild("HumanoidRootPart")
    if not myRoot then return end
    
    for _, v in ipairs(Players:GetPlayers()) do
        if v ~= lp and v.Character then
            local rootPart = v.Character:FindFirstChild("HumanoidRootPart")
            
            if rootPart then
                local distance = (rootPart.Position - myRoot.Position).Magnitude
                
                pcall(function()
                    if distance <= _G.NearDistance then
                        rootPart.Size = Vector3.new(_G.NearSize, _G.NearSize, _G.NearSize)
                        rootPart.Transparency = 0.9  
                        rootPart.BrickColor = BrickColor.new("Bright blue")  
                        rootPart.Material = Enum.Material.Plastic
                    elseif distance >= _G.FarDistance then
                        rootPart.Size = Vector3.new(_G.FarSize, _G.FarSize, _G.FarSize)
                        rootPart.Transparency = 0.6  
                        rootPart.BrickColor = BrickColor.new("Lavender")  
                        rootPart.Material = Enum.Material.Neon
                        rootPart.CanCollide = false
                    else
                        rootPart.Size = Vector3.new(_G.MiddleSize, _G.MiddleSize, _G.MiddleSize)
                        rootPart.Transparency = 0.7  
                        rootPart.BrickColor = BrickColor.new("Lavender")
                        rootPart.Material = Enum.Material.Neon
                        rootPart.CanCollide = false
                    end
                end)
            end
        end
    end
end)