if not game:IsLoaded() then
   game.Loaded:Wait()
end




-- ========================================
-- CONFIGURATION
-- ========================================




-- Player Hitbox Settings (MODERATE INCREASE)
_G.Disabled = false  
_G.NearSize = 5       -- Moderate increase
_G.MiddleSize = 15    -- Moderate increase
_G.FarSize = 30       -- Moderate increase
_G.NearDistance = 20  
_G.FarDistance = 170  




-- NPC Settings
local weaponNames = {"DMR", "Gift of Fire", "Rifle", "Burst Rifle", "Medical Bow", "Vitabow"}
local excludedEnemies = {"Jetpacker", "APU", "Ranger", "Platform"}




-- ========================================
-- SERVICES & VARIABLES
-- ========================================




local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local lp = Players.LocalPlayer




local enemyNames = ReplicatedStorage.Units.Noobs:GetChildren()
local mapFolder = Workspace.Map
local modifiedEnemies = {}
local excludedEnemyInstances = {}




-- ========================================
-- INFINITE AMMO FUNCTIONALITY
-- ========================================




local function SetupInfiniteAmmo()
    local character = lp.Character or lp.CharacterAdded:Wait()
   
    for _, weaponName in ipairs(weaponNames) do
        local weapon = character:WaitForChild(weaponName, 10)
        if weapon then
            weapon:GetAttributeChangedSignal("Ammo"):Connect(function()
                weapon:SetAttribute("Ammo", 999)
            end)
            weapon:SetAttribute("Ammo", 999)
        end
    end
end




-- Run on initial spawn
if lp.Character then
    SetupInfiniteAmmo()
end




-- Run when character respawns
lp.CharacterAdded:Connect(function()
    SetupInfiniteAmmo()
end)




-- Constantly maintain infinite ammo
task.spawn(function()
    while true do
        task.wait(0.1)
       
        local character = lp.Character
        if character then
            for _, weaponName in ipairs(weaponNames) do
                local weapon = character:FindFirstChild(weaponName)
                if weapon then
                    if weapon:GetAttribute("Ammo") then
                        weapon:SetAttribute("Ammo", 999)
                    end
                    if weapon:GetAttribute("MaxAmmo") then
                        weapon:SetAttribute("MaxAmmo", 999)
                    end
                end
            end
        end
    end
end)




-- ========================================
-- NPC HITBOX EXPANSION
-- ========================================




local function CreateHighlight(instance)
    if instance:FindFirstChildOfClass("Highlight") then
        return
    end
   
    local highlight = Instance.new("Highlight")
    highlight.FillColor = Color3.fromRGB(255, 0, 0)
    highlight.OutlineColor = Color3.fromRGB(255, 0, 0)
    highlight.FillTransparency = 0.5
    highlight.OutlineTransparency = 0
    highlight.Parent = instance
end




local function IsExcluded(enemyName)
    for _, excluded in ipairs(excludedEnemies) do
        if enemyName == excluded then
            return true
        end
    end
    return false
end




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
    for _, x in ipairs(enemyNames) do
        if tostring(x) == instance.Name and instance ~= nil then
            if IsExcluded(instance.Name) then
                excludedEnemyInstances[instance] = true
                CreateHighlight(instance)
                return
            end
           
            if modifiedEnemies[instance] then
                return
            end
           
            local head = instance:WaitForChild("Head", 5)
            if head then
                head.Transparency = 0.9
                head.Size = Vector3.new(50.0, 50.0, 50.0)
                head.Massless = true
                head.CanCollide = false
               
                modifiedEnemies[instance] = {head = head}
            end
           
            CreateHighlight(instance)
           
            local humanoid = instance:FindFirstChildOfClass("Humanoid")
            if humanoid then
                humanoid.Died:Connect(function()
                    HandleEnemyDeath(instance, head)
                end)
               
                humanoid:GetPropertyChangedSignal("Health"):Connect(function()
                    if humanoid.Health <= 0 then
                        HandleEnemyDeath(instance, head)
                    end
                end)
            end
        end
    end
end




for _, v in ipairs(Workspace:GetChildren()) do
    task.wait()
    PutOnExtend(v)
end




Workspace.ChildAdded:Connect(function(v)
    task.wait()
    PutOnExtend(v)
end)




task.spawn(function()
    while true do
        task.wait(0.5)
       
        for _, instance in ipairs(Workspace:GetChildren()) do
            for _, x in ipairs(enemyNames) do
                if tostring(x) == instance.Name and instance ~= nil then
                    if excludedEnemyInstances[instance] then
                        local humanoid = instance:FindFirstChildOfClass("Humanoid")
                        if humanoid and humanoid.Health > 0 then
                            CreateHighlight(instance)
                        end
                    else
                        local humanoid = instance:FindFirstChildOfClass("Humanoid")
                        if humanoid and humanoid.Health > 0 then
                            CreateHighlight(instance)
                            PutOnExtend(instance)
                        else
                            if modifiedEnemies[instance] then
                                HandleEnemyDeath(instance, modifiedEnemies[instance].head)
                            end
                        end
                    end
                end
            end
        end
    end
end)




-- ========================================
-- PLAYER HITBOX EXPANSION
-- ========================================




local function updatePlayers()
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
end




task.spawn(function()
    while true do
        updatePlayers()
        task.wait(1)  
    end
end)




-- ========================================
-- UI NOTIFICATION
-- ========================================




local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Parent = game:GetService("CoreGui")




local Frame = Instance.new("Frame")
Frame.Size = UDim2.new(0, 300, 0, 100)
Frame.Position = UDim2.new(0.5, -150, 0.1, 0)
Frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
Frame.BackgroundTransparency = 0.2
Frame.BorderSizePixel = 0
Frame.Parent = ScreenGui




local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0, 10)
UICorner.Parent = Frame




local Title = Instance.new("TextLabel")
Title.Text = "✅ Script Executed!"
Title.Size = UDim2.new(1, 0, 0, 40)
Title.BackgroundTransparency = 1
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.TextScaled = true
Title.Font = Enum.Font.GothamBold
Title.Parent = Frame




local Subtitle = Instance.new("TextLabel")
Subtitle.Text = "Enhanced Hitboxes Active"
Subtitle.Size = UDim2.new(1, 0, 0, 30)
Subtitle.Position = UDim2.new(0, 0, 0.4, 0)
Subtitle.BackgroundTransparency = 1
Subtitle.TextColor3 = Color3.fromRGB(200, 200, 200)
Subtitle.TextScaled = true
Subtitle.Font = Enum.Font.Gotham
Subtitle.Parent = Frame




task.delay(5, function()
    ScreenGui:Destroy()
end)

