-- =====================================================================
-- 👑【小木HUB · 夜脚本风格版 (Patriot UI)】
-- =====================================================================
-- 外观：紫金霓虹、磨砂玻璃、侧边菜单
-- 功能：飞行、自瞄、ESP、飞车、传送、甩飞
-- 警告：依赖网络加载UI库，网不好可能黑屏
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
local FlyCar = { Enabled = false, Speed = 80, Up = false, Down = false }
local Speed = { Enabled = false, Value = 60 }
local Jump = { Enabled = false, Value = 50 }
local Misc = { AntiStun = true, ESP = true, Noclip = false }
local Aimbot = { Enabled = false, FOV = 120, Smooth = 0.7, TeamCheck = true }

local character, rootPart, humanoid
local espCache = {}
local currentTarget = nil
local renderConnection = nil
local noclipApplied = false
local tpTarget = nil

-- ==========================================
-- 【角色监听】
-- ==========================================
local function setupChar(char)
    character = char
    for _, v in pairs(espCache) do
        pcall(function()
            if v and v.Frame then v.Frame:Destroy() end
            if v and v.Label then v.Label:Destroy() end
        end)
    end
    espCache = {}
    Flight.Enabled = false
    FlyCar.Enabled = false
    Aimbot.Enabled = false
    currentTarget = nil
    Misc.ESP = false
    noclipApplied = false
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
    for _, v in pairs(espCache) do
        pcall(function()
            if v and v.Frame then v.Frame:Destroy() end
            if v and v.Label then v.Label:Destroy() end
        end)
    end
    espCache = {}
    if renderConnection then renderConnection:Disconnect() end
end

-- ==========================================
-- 【加载 Patriot UI 库】
-- ==========================================
local Library = loadstring(game:HttpGet("https://raw.githubusercontent.com/SyndromeXph/Patriot-Key-System-Ui-Library/main/Patriot.lua"))()

local Window = Library:CreateWindow({
    Title = "小木HUB",
    SubTitle = "夜脚本风格版",
    Icon = "rbxassetid://135577550284336",
    Theme = "Purple"
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
local CarToggle = CarGroup:CreateToggle("飞车模式", FlyCar.Enabled, function(v) FlyCar.Enabled = v; FlyCar.Up = false; FlyCar.Down = false end)
local CarSpeed = CarGroup:CreateSlider("飞车速度", 20, 200, FlyCar.Speed, function(v) FlyCar.Speed = v end)

local CarUpBtn = CarGroup:CreateButton("▲ 上升 (长按)", function() FlyCar.Up = true end)
local CarDownBtn = CarGroup:CreateButton("▼ 下降 (长按)", function() FlyCar.Down = true end)

-- 传送
local TargetDropdown = TeleportGroup:CreateDropdown("选择目标", {}, function(v) end)
local DirectBtn = TeleportGroup:CreateButton("直接传送", function() end)
local FrontBtn = TeleportGroup:CreateButton("前面传送", function() end)
local BackBtn = TeleportGroup:CreateButton("后面传送", function() end)
local FlingBtn = TeleportGroup:CreateButton("💥 甩飞一次", function() end)

task.spawn(function()
    while true do
        local list = {}
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= plr then table.insert(list, p.Name) end
        end
        pcall(function() TargetDropdown:SetValues(list) end)
        task.wait(3)
    end
end)

TargetDropdown:OnSelect(function(name)
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
                if Aimbot.TeamCheck and p.Team == plr.Team then continue end
                local c = p.Character
                if c and c:FindFirstChild("HumanoidRootPart") and c:FindFirstChild("Humanoid") and c.Humanoid.Health > 0 then
                    local hrp = c.HumanoidRootPart
                    local s, on = camera:WorldToViewportPoint(hrp.Position)
                    if on then
                        local params = RaycastParams.new()
                        params.FilterDescendantsInstances = {plr.Character}
                        params.FilterType = Enum.RaycastFilterType.Exclude
                        local result = workspace:Raycast(camera.CFrame.Position, (hrp.Position - camera.CFrame.Position).Unit * 999, params)
                        if not result then
                            local d = (Vector2.new(s.X, s.Y) - center).Magnitude
                            if d < closestDist then closestDist = d; closestTarget = p end
                        end
                    end
                end
            end
        end
        if closestTarget then
            currentTarget = closestTarget
            local c = closestTarget.Character
            if c and c:FindFirstChild("HumanoidRootPart") then
                local tPos = c.HumanoidRootPart.Position
                local lDir = (tPos - camera.CFrame.Position).Unit
                camera.CFrame = camera.CFrame:Lerp(CFrame.lookAt(camera.CFrame.Position, camera.CFrame.Position + lDir), Aimbot.Smooth)
            end
        else
            currentTarget = nil
        end
    end

    -- ESP
    if Misc.ESP then
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= plr and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                local hrp = p.Character.HumanoidRootPart
                local s, on = camera:WorldToViewportPoint(hrp.Position)
                if on and s.Z > 0 and s.Z < 1000 then
                    local size = 50 + (200 - s.Z) * 0.3
                    if not espCache[p.UserId] then
                        local f = Instance.new("Frame")
                        f.Parent = CoreGui
                        f.BackgroundTransparency = 1
                        f.Size = UDim2.new(0, size, 0, size*2)
                        f.Position = UDim2.new(0, s.X - size/2, 0, s.Y - size)
                        f.ZIndex = 20
                        local st = Instance.new("UIStroke")
                        st.Color = Color3.fromRGB(255, 255, 255)
                        st.Thickness = 1
                        st.Parent = f
                        local l = Instance.new("TextLabel")
                        l.Parent = f
                        l.BackgroundTransparency = 1
                        l.Size = UDim2.new(0, 100, 0, 16)
                        l.Position = UDim2.new(0, -25, 0, -20)
                        l.Text = p.Name
                        l.TextColor3 = Color3.fromRGB(255, 255, 255)
                        l.TextSize = 10
                        l.ZIndex = 21
                        espCache[p.UserId] = {Frame=f, Label=l}
                    else
                        local e = espCache[p.UserId]
                        e.Frame.Size = UDim2.new(0, size, 0, size*2)
                        e.Frame.Position = UDim2.new(0, s.X - size/2, 0, s.Y - size)
                        e.Frame.Visible = true
                        e.Label.Visible = true
                    end
                else
                    if espCache[p.UserId] then
                        espCache[p.UserId].Frame.Visible = false
                        espCache[p.UserId].Label.Visible = false
                    end
                end
            end
        end
    else
        for _, v in pairs(espCache) do
            pcall(function()
                if v and v.Frame then v.Frame:Destroy() end
                if v and v.Label then v.Label:Destroy() end
            end)
        end
        espCache = {}
    end

    -- 飞车
    if FlyCar.Enabled then
        local seat = plr.Character and (plr.Character:FindFirstChild("Seat") or plr.Character:FindFirstChild("VehicleSeat"))
        if seat and seat.Occupant and seat.Occupant.Parent == plr.Character then
            local vr = seat.Parent:FindFirstChild("VehicleRootPart") or seat.Parent:FindFirstChild("PrimaryPart") or seat.Parent:FindFirstChild("Body")
            if vr then
                local v = Vector3.new(0,0,0)
                local h = plr.Character:FindFirstChild("Humanoid")
                if h then
                    local d = h.MoveDirection
                    if d.Magnitude > 0.1 then
                        local r = camera.CFrame:VectorToObjectSpace(d)
                        v = (camera.CFrame.LookVector * -r.Z + camera.CFrame.RightVector * r.X) * FlyCar.Speed
                    end
                end
                if FlyCar.Up then v = v + Vector3.new(0, FlyCar.Speed, 0) end
                if FlyCar.Down then v = v - Vector3.new(0, FlyCar.Speed, 0) end
                vr.AssemblyLinearVelocity = v
            end
        end
    end

    -- 基础属性
    pcall(function()
        if character and humanoid and humanoid.Parent then
            if Speed.Enabled then humanoid.WalkSpeed = Speed.Value
            elseif humanoid.WalkSpeed ~= 16 then humanoid.WalkSpeed = 16 end
            if Jump.Enabled then humanoid.JumpPower = Jump.Value; humanoid.UseJumpPower = true
            else humanoid.UseJumpPower = false end
            if Misc.AntiStun and humanoid.PlatformStand and not Flight.Enabled then humanoid.PlatformStand = false end
        end
    end)

    -- Noclip
    if Misc.Noclip and not noclipApplied and character then
        pcall(function() for _, p in ipairs(character:GetDescendants()) do if p:IsA("BasePart") and p.Name ~= "HumanoidRootPart" then p.CanCollide = false end end; noclipApplied = true end)
    elseif not Misc.Noclip and noclipApplied and character then
        pcall(function() for _, p in ipairs(character:GetDescendants()) do if p:IsA("BasePart") then p.CanCollide = true end end; noclipApplied = false end)
    end

    -- 飞行
    if Flight.Enabled then
        pcall(function()
            if not character or not rootPart or not humanoid then return end
            humanoid.PlatformStand = true
            local moveDir = humanoid.MoveDirection
            local velocity = Vector3.new(0,0,0)
            if moveDir.Magnitude > 0 then
                local rel = camera.CFrame:VectorToObjectSpace(moveDir)
                velocity = (camera.CFrame.LookVector * -rel.Z + camera.CFrame.RightVector * rel.X) * Flight.Speed
            end
            if Flight.Up then velocity = velocity + Vector3.new(0, Flight.Speed, 0) end
            if Flight.Down then velocity = velocity - Vector3.new(0, Flight.Speed, 0) end
            rootPart.AssemblyLinearVelocity = velocity
        end)
    end
end)

print("✅【小木HUB · 夜脚本风格版】加载成功！")
