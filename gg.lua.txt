-- =====================================================================
-- 👑【小木HUB · 终极夜脚本风格版】
-- =====================================================================
-- 基于 Patriot-Key-System UI 库重写
-- 完美复刻夜脚本紫金霓虹、磨砂半透、侧边栏样式
-- 功能：飞行/自瞄/飞车/ESP/传送/甩飞
-- =====================================================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")

local plr = Players.LocalPlayer
local camera = Workspace.CurrentCamera

-- ==========================================
-- 【核心功能状态】
-- ==========================================
local Flight = { Enabled = false, Speed = 60, Up = false, Down = false }
local FlyCar = { Enabled = false, Speed = 80 }
local Speed = { Enabled = false, Value = 60 }
local Jump = { Enabled = false, Value = 50 }
local Misc = { AntiStun = true, ESP = true, Noclip = false }
local Aimbot = { Enabled = false, FOV = 120, TeamCheck = true, AimPart = "UpperTorso" }

local character, rootPart, humanoid
local espCache = {}
local currentTarget = nil
local renderConnection = nil
local carUpP, carDownP = false, false
local noclipApplied = false

-- ==========================================
-- 【角色监听】
-- ==========================================
local function setupChar(char)
    character = char
    task.spawn(function()
        rootPart = char:WaitForChild("HumanoidRootPart", 5)
        humanoid = char:WaitForChild("Humanoid", 5)
    end)
end
if plr.Character then setupChar(plr.Character) end
plr.CharacterAdded:Connect(setupChar)

local function cleanup()
    Flight.Enabled = false; FlyCar.Enabled = false
    if humanoid then pcall(function() humanoid.PlatformStand = false end) end
    if character then pcall(function() for _, p in ipairs(character:GetDescendants()) do if p:IsA("BasePart") then p.CanCollide = true end end end) end
    noclipApplied = false
    for _, v in pairs(espCache) do pcall(function() v:Destroy() end) end
    espCache = {}
end

-- ==========================================
-- 【加载 Patriot UI 库】
-- ==========================================
local Library = loadstring(game:HttpGet("https://raw.githubusercontent.com/SyndromeXph/Patriot-Key-System-Ui-Library/main/Patriot.lua"))()

local Window = Library:CreateWindow({
    Title = "小木HUB",
    SubTitle = "顶级 UI 版",
    Icon = "rbxassetid://135577550284336", -- 大司马头像
    Theme = "Purple" -- 紫金霓虹风格
})

-- ==========================================
-- 【创建分类标签】
-- ==========================================
local Tabs = {
    Move = Window:CreateTab("移动", "⚡"),
    Combat = Window:CreateTab("战斗", "🎯"),
    Visual = Window:CreateTab("视觉", "👁️"),
    Car = Window:CreateTab("飞车", "🚗"),
    Teleport = Window:CreateTab("传送", "🌀"),
}

local MoveGroup = Tabs.Move:CreateLeftGroupbox("人物控制")
local CombatGroup = Tabs.Combat:CreateLeftGroupbox("自瞄设置")
local VisualGroup = Tabs.Visual:CreateLeftGroupbox("视觉辅助")
local CarGroup = Tabs.Car:CreateLeftGroupbox("飞车系统")
local TeleportGroup = Tabs.Teleport:CreateLeftGroupbox("传送操作")

-- ==========================================
-- 【功能绑定】
-- ==========================================
-- 移动
local SpeedToggle = MoveGroup:CreateToggle("速度加快", Speed.Enabled, function(v) Speed.Enabled = v end)
local SpeedSlider = MoveGroup:CreateSlider("移动速度", 16, 200, Speed.Value, function(v) Speed.Value = v end)
local JumpToggle = MoveGroup:CreateToggle("跳跃加强", Jump.Enabled, function(v) Jump.Enabled = v end)
local JumpSlider = MoveGroup:CreateSlider("跳跃高度", 50, 300, Jump.Value, function(v) Jump.Value = v end)
local NoclipToggle = MoveGroup:CreateToggle("穿墙 (Noclip)", Misc.Noclip, function(v) Misc.Noclip = v; noclipApplied = false end)
local FlightToggle = MoveGroup:CreateToggle("人物飞行", Flight.Enabled, function(v) Flight.Enabled = v; if not v then cleanup() end end)

-- 战斗
local AimbotToggle = CombatGroup:CreateToggle("超维自瞄", Aimbot.Enabled, function(v) Aimbot.Enabled = v; if not v then currentTarget = nil end end)
local TeamToggle = CombatGroup:CreateToggle("战队保护", Aimbot.TeamCheck, function(v) Aimbot.TeamCheck = v end)
local FovSlider = CombatGroup:CreateSlider("FOV 范围", 50, 300, Aimbot.FOV, function(v) Aimbot.FOV = v end)

-- 视觉
local EspToggle = VisualGroup:CreateToggle("全图透视 ESP", Misc.ESP, function(v) Misc.ESP = v end)

-- 飞车
local CarToggle = CarGroup:CreateToggle("飞车模式", FlyCar.Enabled, function(v) FlyCar.Enabled = v; if not v then carUpP = false; carDownP = false end end)
local CarSpeed = CarGroup:CreateSlider("飞车速度", 20, 200, FlyCar.Speed, function(v) FlyCar.Speed = v end)
local CarUp = CarGroup:CreateButton("▲ 上升", function() carUpP = true; task.wait(0.2); carUpP = false end)
local CarDown = CarGroup:CreateButton("▼ 下降", function() carDownP = true; task.wait(0.2); carDownP = false end)

-- 传送
local TargetDropdown = TeleportGroup:CreateDropdown("选择目标", {}, function(v) end)
local DirectBtn = TeleportGroup:CreateButton("直接传送", function() end)
local FrontBtn = TeleportGroup:CreateButton("前面传送", function() end)
local BackBtn = TeleportGroup:CreateButton("后面传送", function() end)
local FlingBtn = TeleportGroup:CreateButton("💥 甩飞一次", function() end)

-- ==========================================
-- 【玩家列表自动刷新】
-- ==========================================
local tpTarget = nil
task.spawn(function()
    while true do
        local list = {}
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= plr then table.insert(list, p.Name) end
        end
        TargetDropdown:SetValues(list)
        task.wait(3)
    end
end)

TargetDropdown.OnSelect:Connect(function(name)
    for _, p in ipairs(Players:GetPlayers()) do
        if p.Name == name then tpTarget = p; break end
    end
end)

local function teleportPlayer(mode)
    if not tpTarget or not tpTarget.Character then return end
    local hrp = tpTarget.Character:FindFirstChild("HumanoidRootPart")
    if not hrp or not rootPart then return end
    if mode == "direct" then rootPart.CFrame = hrp.CFrame
    elseif mode == "front" then rootPart.CFrame = hrp.CFrame + hrp.CFrame.LookVector * 0.5
    elseif mode == "back" then rootPart.CFrame = hrp.CFrame - hrp.CFrame.LookVector * 0.5 end
end

DirectBtn:OnClick(function() teleportPlayer("direct") end)
FrontBtn:OnClick(function() teleportPlayer("front") end)
BackBtn:OnClick(function() teleportPlayer("back") end)
FlingBtn:OnClick(function()
    if tpTarget and tpTarget.Character then
        local hrp = tpTarget.Character:FindFirstChild("HumanoidRootPart")
        if hrp then hrp.AssemblyLinearVelocity = Vector3.new(0, 120, 0) end
    end
end)

-- ==========================================
-- 【主循环：物理与逻辑】
-- ==========================================
renderConnection = RunService.RenderStepped:Connect(function()
    if not Window then return end

    -- 自瞄
    if Aimbot.Enabled then
        local closestDist = Aimbot.FOV
        local closestTarget = nil
        local center = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= plr then
                if Aimbot.TeamCheck and p.TeamColor == plr.TeamColor then continue end
                local c = p.Character
                if c and c:FindFirstChild("HumanoidRootPart") and c:FindFirstChild("Humanoid") and c.Humanoid.Health > 0 then
                    local s, on = camera:WorldToViewportPoint(c.HumanoidRootPart.Position)
                    if on then
                        local d = (Vector2.new(s.X, s.Y) - center).Magnitude
                        if d < closestDist then closestDist = d; closestTarget = p end
                    end
                end
            end
        end
        if closestTarget then
            currentTarget = closestTarget
            local c =
