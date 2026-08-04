-- =====================================================================
-- 👑【小木HUB v6.6 - 完美直显版】
-- =====================================================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local TweenService = game:GetService("TweenService")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")

local plr = Players.LocalPlayer
local camera = Workspace.CurrentCamera

-- ==========================================
-- 【配置文件系统】
-- ==========================================
local Config = {}
local ConfigFile = "Ye_UI_Settings.json"

local function loadConfig()
    if writefile and readfile then
        local success, data = pcall(function()
            return readfile(ConfigFile)
        end)
        if success and data and data ~= "" then
            local success2, decoded = pcall(function()
                return HttpService:JSONDecode(data)
            end)
            if success2 and type(decoded) == "table" then
                Config = decoded
                return true
            end
        end
    end
    return false
end

local function saveConfig()
    if writefile then
        local success, encoded = pcall(function()
            return HttpService:JSONEncode(Config)
        end)
        if success then
            pcall(function()
                writefile(ConfigFile, encoded)
            end)
        end
    end
end

-- 默认配置
local defaultConfig = {
    SpeedEnabled = false,
    SpeedValue = 60,
    JumpEnabled = false,
    JumpValue = 50,
    NoclipEnabled = false,
    FlightEnabled = false,
    AimbotEnabled = false,
    TeamCheck = true,
    FOVValue = 120,
    ESPEnabled = false,
    FlyCarEnabled = false,
    FlyCarSpeed = 80
}

-- 加载配置，如果没加载成功则使用默认
if not loadConfig() then
    Config = defaultConfig
    saveConfig()
end

-- ==========================================
-- 【模块状态（使用配置初始化）】
-- ==========================================
local FlightModule = { Enabled = Config.FlightEnabled, Speed = 60, MovingUp = false, MovingDown = false }
local FlyCarModule = { Enabled = Config.FlyCarEnabled, Speed = Config.FlyCarSpeed }
local SpeedModule = { Enabled = Config.SpeedEnabled, CustomSpeed = Config.SpeedValue }
local JumpModule = { Enabled = Config.JumpEnabled, CustomJump = Config.JumpValue }
local MiscModule = { AntiStun = true, ESPEnabled = Config.ESPEnabled, Noclip = Config.NoclipEnabled }
local AimbotModule = { 
    Enabled = Config.AimbotEnabled, 
    PredictionEnabled = true, 
    FOV = Config.FOVValue, 
    BulletSpeed = 1600.0, 
    Smoothing = 0.3, 
    TeamCheck = Config.TeamCheck, 
    PlayerWhitelist = {}, 
    PlayerBlacklist = {},
    AimPart = "UpperTorso",
    AimPartName = "胸口"
}
local TeamWhitelist = { Locked = false, TargetTeam = nil }
local teleportDistance = 50

local character, rootPart, humanoid
local noclipApplied = false
local renderConnection = nil
local espCache = {}
local currentTarget = nil
local isClosing = false
local showFOVCircle = true
local carSeat = nil
local currentVehicle = nil
local carUpP, carDownP = false, false

-- 快速保存配置的函数
local function updateConfig(key, value)
    Config[key] = value
    saveConfig()
end

-- ==========================================
-- 【清理】
-- ==========================================
local function cleanupState()
    FlightModule.MovingUp = false; FlightModule.MovingDown = false; FlightModule.Enabled = false; FlyCarModule.Enabled = false
    if humanoid then pcall(function() humanoid.PlatformStand = false; humanoid:ChangeState(Enum.HumanoidStateType.GettingUp) end) end
    if character then pcall(function() for _, part in ipairs(character:GetDescendants()) do if part:IsA("BasePart") then part.CanCollide = true end end end) end
    noclipApplied = false; currentTarget = nil; carSeat = nil; currentVehicle = nil
    for userId, esp in pairs(espCache) do if esp and esp.Parent then pcall(function() esp:Destroy() end) end end
    espCache = {}
end
local function safeDestroy(obj) if obj and obj.Parent then pcall(function() obj:Destroy() end) end end

local function setupCharacter(char)
    if isClosing then return end
    character = char; cleanupState(); noclipApplied = false
    task.spawn(function() rootPart = char:WaitForChild("HumanoidRootPart", 5); humanoid = char:WaitForChild("Humanoid", 5) end)
end
if plr.Character then setupCharacter(plr.Character) end
plr.CharacterAdded:Connect(setupCharacter)

-- ==========================================
-- 【UI 构建】
-- ==========================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "XiaoMuHubV2"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
local success, parent = pcall(function() return CoreGui end)
if success and parent then screenGui.Parent = CoreGui else screenGui.Parent = plr:WaitForChild("PlayerGui") end

-- ================= 窗口主体 =================
local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"; mainFrame.Parent = screenGui
mainFrame.Size = UDim2.new(0, 520, 0, 400)
mainFrame.Position = UDim2.new(0.5, 0, 0.5, 0); mainFrame.AnchorPoint = Vector2.new(0.5, 0.5)
mainFrame.BackgroundColor3 = Color3.fromRGB(20, 22, 30); mainFrame.BackgroundTransparency = 0.02
mainFrame.BorderSizePixel = 0; mainFrame.Visible = false
Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 8)
local mainStroke = Instance.new("UIStroke"); mainStroke.Color = Color3.fromRGB(45, 48, 60); mainStroke.Thickness = 1.5; mainStroke.Parent = mainFrame

local titleBar = Instance.new("Frame")
titleBar.Name = "TitleBar"; titleBar.Parent = mainFrame; titleBar.Size = UDim2.new(1, 0, 0, 40)
titleBar.BackgroundColor3 = Color3.fromRGB(28, 30, 40); titleBar.BackgroundTransparency = 0.2; titleBar.BorderSizePixel = 0
Instance.new("UICorner", titleBar).CornerRadius = UDim.new(0, 8)
local cornerFix = Instance.new("Frame"); cornerFix.Parent = titleBar; cornerFix.Size = UDim2.new(1, 0, 0, 8); cornerFix.Position = UDim2.new(0, 0, 0, 32); cornerFix.BackgroundColor3 = Color3.fromRGB(20, 22, 30); cornerFix.BorderSizePixel = 0
local titleText = Instance.new("TextLabel"); titleText.Parent = titleBar; titleText.Size = UDim2.new(0.6, 0, 1, 0); titleText.Position = UDim2.new(0.05, 0, 0, 0); titleText.BackgroundTransparency = 1; titleText.Font = Enum.Font.GothamBold; titleText.Text = " 小木HUB v6.6"; titleText.TextColor3 = Color3.fromRGB(80, 160, 255); titleText.TextSize = 16; titleText.TextXAlignment = Enum.TextXAlignment.Left
local closeBtn = Instance.new("TextButton"); closeBtn.Parent = titleBar; closeBtn.Size = UDim2.new(0, 30, 0, 30); closeBtn.Position = UDim2.new(1, -36, 0, 5); closeBtn.BackgroundTransparency = 1; closeBtn.Text = "✕"; closeBtn.TextColor3 = Color3.fromRGB(180, 180, 180); closeBtn.Font = Enum.Font.GothamBold; closeBtn.TextSize = 14
closeBtn.MouseButton1Click:Connect(function() if isClosing then return end; isClosing = true; cleanupState(); if renderConnection then renderConnection:Disconnect() end; safeDestroy(screenGui) end)

-- ================= 分类与UI =================
local tabScroll = Instance.new("ScrollingFrame"); tabScroll.Parent = mainFrame; tabScroll.BackgroundTransparency = 1; tabScroll.Position = UDim2.new(0.05, 0, 0.12, 0); tabScroll.Size = UDim2.new(0.9, 0, 0, 40); tabScroll.ScrollBarThickness = 0; tabScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
local currentTab = "移动"; local tabContentFrames = {}
local function createTabBtn(txt, icon)
    local btn = Instance.new("TextButton"); btn.Parent = tabScroll; btn.BackgroundTransparency = 1; btn.Size = UDim2.new(0, 80, 1, 0); btn.Text = icon .. txt; btn.TextColor3 = Color3.fromRGB(160, 160, 180); btn.Font = Enum.Font.GothamBold; btn.TextSize = 13; Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 8); return btn
end
local function createContentPanel()
    local sf = Instance.new("ScrollingFrame"); sf.Parent = mainFrame; sf.BackgroundTransparency = 1; sf.Position = UDim2.new(0.05, 0, 0.20, 0); sf.Size = UDim2.new(0.9, 0, 0.70, 0); sf.CanvasSize = UDim2.new(0, 0, 0, 350); sf.ScrollBarThickness = 3; sf.Visible = false; return sf
end
local function createToggle(parent, posY, txt, initialState, cb)
    local card = Instance.new("Frame"); card.Parent = parent; card.BackgroundColor3 = Color3.fromRGB(24, 26, 35); card.Position = UDim2.new(0, 0, 0, posY); card.Size = UDim2.new(1, 0, 0, 40); Instance.new("UICorner", card).CornerRadius = UDim.new(0, 6)
    local lbl = Instance.new("TextLabel"); lbl.Parent = card; lbl.BackgroundTransparency = 1; lbl.Position = UDim2.new(0.05, 0, 0, 0); lbl.Size = UDim2.new(0.6, 0, 1, 0); lbl.Font = Enum.Font.GothamBold; lbl.Text = txt; lbl.TextColor3 = Color3.fromRGB(220, 220, 230); lbl.TextSize = 12; lbl.TextXAlignment = Enum.TextXAlignment.Left
    local bg = Instance.new("TextButton"); bg.Parent = card; bg.BackgroundColor3 = initialState and Color3.fromRGB(46, 204, 113) or Color3.fromRGB(45, 48, 60); bg.Position = UDim2.new(0.82, 0, 0.2, 0); bg.Size = UDim2.new(0, 40, 0, 22); bg.Text = ""; Instance.new("UICorner", bg).CornerRadius = UDim.new(1, 0)
    local dot = Instance.new("Frame"); dot.Parent = bg; dot.BackgroundColor3 = Color3.fromRGB(255, 255, 255); dot.Position = initialState and UDim2.new(0.52, 0, 0.1, 0) or UDim2.new(0.08, 0, 0.1, 0); dot.Size = UDim2.new(0, 16, 0, 16); Instance.new("UICorner", dot).CornerRadius = UDim.new(1, 0)
    local state = initialState
    bg.MouseButton1Click:Connect(function() state = not state; local targetColor = state and Color3.fromRGB(46, 204, 113) or Color3.fromRGB(45, 48, 60); local targetPos = state and UDim2.new(0.52, 0, 0.1, 0) or UDim2.new(0.08, 0, 0.1, 0); TweenService:Create(bg, TweenInfo.new(0.2), {BackgroundColor3 = targetColor}):Play(); TweenService:Create(dot, TweenInfo.new(0.2), {Position = targetPos}):Play(); cb(state) end)
end
local function createSlider(parent, posY, txt, mn, mx, cur, unit, cb)
    local card = Instance.new("Frame"); card.Parent = parent; card.BackgroundColor3 = Color3.fromRGB(24, 26, 35); card.Position = UDim2.new(0, 0, 0, posY); card.Size = UDim2.new(1, 0, 0, 40); Instance.new("UICorner", card).CornerRadius = UDim.new(0, 6)
    local lbl = Instance.new("TextLabel"); lbl.Parent = card; lbl.BackgroundTransparency = 1; lbl.Position = UDim2.new(0.05, 0, 0, 0); lbl.Size = UDim2.new(0.6, 0, 1, 0); lbl.Font = Enum.Font.GothamBold; lbl.Text = txt; lbl.TextColor3 = Color3.fromRGB(220, 220, 230); lbl.TextSize = 12; lbl.TextXAlignment = Enum.TextXAlignment.Left
    local val = Instance.new("TextLabel"); val.Parent = card; val.BackgroundTransparency = 1; val.Position = UDim2.new(0.52, 0, 0, 0); val.Size = UDim2.new(0.12, 0, 1, 0); val.Font = Enum.Font.GothamBold; val.Text = tostring(cur); val.TextColor3 = Color3.fromRGB(80, 160, 255); val.TextSize = 12
    local bar = Instance.new("Frame"); bar.Parent = card; bar.BackgroundColor3 = Color3.fromRGB(45, 48, 60); bar.Position = UDim2.new(0.66, 0, 0.45, 0); bar.Size = UDim2.new(0.3, 0, 0, 6); Instance.new("UICorner", bar).CornerRadius = UDim.new(1, 0)
    local fill = Instance.new("Frame"); fill.Parent = bar; fill.BackgroundColor3 = Color3.fromRGB(80, 160, 255); fill.Size = UDim2.new((cur - mn)/(mx - mn), 0, 1, 0); Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)
    local knob = Instance.new("Frame"); knob.Parent = bar; knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255); knob.AnchorPoint = Vector2.new(0.5, 0.5); knob.Position = UDim2.new((cur - mn)/(mx - mn), 0, 0.5, 0); knob.Size = UDim2.new(0, 14, 0, 14); Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)
    local dragging = false
    knob.InputBegan:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = true end end)
    UserInputService.InputEnded:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = false end end)
    UserInputService.InputChanged:Connect(function(input) if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then local posX = input.Position.X - bar.AbsolutePosition.X; local pos = math.clamp(posX / bar.AbsoluteSize.X, 0, 1); local v = math.floor(mn + (mx - mn) * pos); val.Text = tostring(v); fill.Size = UDim2.new(pos, 0, 1, 0); knob.Position = UDim2.new(pos, 0, 0.5, 0); cb(v) end end)
end

local tabBtns = {}
local function setupTabs()
    local names = {"移动", "战斗", "视觉", "飞车", "传送"}; local icons = {"⚡", "🎯", "👁️", "🚗", "🌀"}
    local posX = 0
    for i, name in ipairs(names) do
        local btn = createTabBtn(name, icons[i]); btn.Position = UDim2.new(0, posX, 0, 0); posX = posX + 90; tabBtns[name] = btn
        local panel = createContentPanel(); panel.Name = name .. "Panel"; tabContentFrames[name] = panel
        btn.MouseButton1Click:Connect(function() currentTab = name; for _, b in pairs(tabBtns) do b.TextColor3 = Color3.fromRGB(160, 160, 180) end; btn.TextColor3 = Color3.fromRGB(80, 160, 255); for n, p in pairs(tabContentFrames) do p.Visible = (n == name) end end)
    end
    tabScroll.CanvasSize = UDim2.new(0, posX, 0, 0)
    if tabBtns["移动"] then tabBtns["移动"].TextColor3 = Color3.fromRGB(80, 160, 255) end
    if tabContentFrames["移动Panel"] then tabContentFrames["移动Panel"].Visible = true end
end
setupTabs()

-- ================= 功能填充 =================
local mP = tabContentFrames["移动Panel"]
createToggle(mP, 0, "速度加快", Config.SpeedEnabled, function(s) SpeedModule.Enabled = s; updateConfig("SpeedEnabled", s) end)
createSlider(mP, 50, "移动速度", 16, 200, Config.SpeedValue, "速", function(v) SpeedModule.CustomSpeed = v; updateConfig("SpeedValue", v) end)
createToggle(mP, 100, "跳跃加强", Config.JumpEnabled, function(s) JumpModule.Enabled = s; updateConfig("JumpEnabled", s) end)
createSlider(mP, 150, "跳跃高度", 50, 300, Config.JumpValue, "高", function(v) JumpModule.CustomJump = v; updateConfig("JumpValue", v) end)
createToggle(mP, 200, "穿墙 (Noclip)", Config.NoclipEnabled, function(s) MiscModule.Noclip = s; noclipApplied = false; updateConfig("NoclipEnabled", s) end)
createToggle(mP, 250, "人物飞行", Config.FlightEnabled, function(s) FlightModule.Enabled = s; if not s then cleanupState() end; updateConfig("FlightEnabled", s) end)

local bP = tabContentFrames["战斗Panel"]
createToggle(bP, 0, "超维自瞄", Config.AimbotEnabled, function(s) AimbotModule.Enabled = s; if not s then currentTarget = nil; fovCircle.Visible = false end; updateConfig("AimbotEnabled", s) end)
createToggle(bP, 50, "战队保护", Config.TeamCheck, function(s) AimbotModule.TeamCheck = s; updateConfig("TeamCheck", s) end)
createSlider(bP, 100, "FOV范围", 50, 300, Config.FOVValue, "度", function(v) AimbotModule.FOV = v; updateConfig("FOVValue", v); if camera then local r = v * 2.2; fovCircle.Size = UDim2.new(0, r * 2, 0, r * 2); fovCircle.Position = UDim2.new(0, camera.ViewportSize.X / 2, 0, camera.ViewportSize.Y / 2) end end)

local vP = tabContentFrames["视觉Panel"]
createToggle(vP, 0, "全图透视 ESP", Config.ESPEnabled, function(s) MiscModule.ESPEnabled = s; updateConfig("ESPEnabled", s) end)

local cP = tabContentFrames["飞车Panel"]
createToggle(cP, 0, "🚗 飞车模式", Config.FlyCarEnabled, function(s) FlyCarModule.Enabled = s; if not s then carSeat = nil; currentVehicle = nil end; updateConfig("FlyCarEnabled", s) end)
createSlider(cP, 50, "飞车速度", 20, 200, Config.FlyCarSpeed, "速", function(v) FlyCarModule.Speed = v; updateConfig("FlyCarSpeed", v) end)
local carCtrl = Instance.new("Frame"); carCtrl.Parent = cP; carCtrl.BackgroundColor3 = Color3.fromRGB(24, 26, 35); carCtrl.Position = UDim2.new(0, 0, 0, 100); carCtrl.Size = UDim2.new(1, 0, 0, 50); Instance.new("UICorner", carCtrl).CornerRadius = UDim.new(0, 6)
local carLbl = Instance.new("TextLabel"); carLbl.Parent = carCtrl; carLbl.BackgroundTransparency = 1; carLbl.Position = UDim2.new(0.05, 0, 0.1, 0); carLbl.Size = UDim2.new(0.5, 0, 1, 0); carLbl.Font = Enum.Font.GothamBold; carLbl.Text = "飞车升降"; carLbl.TextColor3 = Color3.fromRGB(220, 220, 230); carLbl.TextSize = 12
local carUp = Instance.new("TextButton"); carUp.Parent = carCtrl; carUp.BackgroundColor3 = Color3.fromRGB(46, 204, 113); carUp.Position = UDim2.new(0.65, 0, 0.1, 0); carUp.Size = UDim2.new(0.15, 0, 0, 32); carUp.Text = "▲"; carUp.TextColor3 = Color3.fromRGB(255, 255, 255); carUp.TextSize = 14; Instance.new("UICorner", carUp).CornerRadius = UDim.new(0, 4)
local carDown = Instance.new("TextButton"); carDown.Parent = carCtrl; carDown.BackgroundColor3 = Color3.fromRGB(231, 76, 60); carDown.Position = UDim2.new(0.82, 0, 0.1, 0); carDown.Size = UDim2.new(0.15, 0, 0, 32); carDown.Text = "▼"; carDown.TextColor3 = Color3.fromRGB(255, 255, 255); carDown.TextSize = 14; Instance.new("UICorner", carDown).CornerRadius = UDim.new(0, 4)
local function bindCar(btn, setter) btn.MouseButton1Down:Connect(function() setter(true) end); btn.MouseButton1Up:Connect(function() setter(false) end); btn.InputEnded:Connect(function(i) if i.UserInputType == Enum.UserInputType.Touch then setter(false) end end) end
bindCar(carUp, function(s) carUpP = s end); bindCar(carDown, function(s) carDownP = s end)

local tP = tabContentFrames["传送Panel"]
local tpTarget = nil
local tpList = Instance.new("ScrollingFrame"); tpList.Parent = tP; tpList.BackgroundTransparency = 1; tpList.Position = UDim2.new(0, 0, 0, 0); tpList.Size = UDim2.new(1, 0, 0, 150); tpList.CanvasSize = UDim2.new(0, 0, 0, 0); tpList.ScrollBarThickness = 2
local function refreshTP() for _, c in ipairs(tpList:GetChildren()) do if c:IsA("TextButton") then c:Destroy() end end; local y = 0; for _, p in ipairs(Players:GetPlayers()) do if p ~= plr then local btn = Instance.new("TextButton"); btn.Parent = tpList; btn.BackgroundColor3 = Color3.fromRGB(30, 32, 42); btn.Position = UDim2.new(0, 0, 0, y); btn.Size = UDim2.new(1, 0, 0, 28); btn.Text = "  " .. p.Name; btn.TextColor3 = Color3.fromRGB(200, 200, 210); btn.TextSize = 12; btn.TextXAlignment = Enum.TextXAlignment.Left; Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4); btn.MouseButton1Click:Connect(function() tpTarget = p; for _, c in ipairs(tpList:GetChildren()) do if c:IsA("TextButton") then c.BackgroundColor3 = Color3.fromRGB(30, 32, 42) end end; btn.BackgroundColor3 = Color3.fromRGB(50, 55, 70) end); y = y + 34 end end; tpList.CanvasSize = UDim2.new(0, 0, 0, math.max(y, 10)) end
task.spawn(function() while screenGui and screenGui.Parent do refreshTP() task.wait(3) end end); refreshTP()
local function teleportPlayer(mode) if not tpTarget or not tpTarget.Character then return end; local hrp = tpTarget.Character:FindFirstChild("HumanoidRootPart"); if not hrp or not rootPart then return end; if mode == "direct" then rootPart.CFrame = hrp.CFrame; elseif mode == "front" then rootPart.CFrame = hrp.CFrame + hrp.CFrame.LookVector * 0.5; elseif mode == "back" then rootPart.CFrame = hrp.CFrame - hrp.CFrame.LookVector * 0.5 end end
local btnCard = Instance.new("Frame"); btnCard.Parent = tP; btnCard.BackgroundColor3 = Color3.fromRGB(24, 26, 35); btnCard.Position = UDim2.new(0, 0, 0, 160); btnCard.Size = UDim2.new(1, 0, 0, 45); Instance.new("UICorner", btnCard).CornerRadius = UDim.new(0, 6)
local dBtn = Instance.new("TextButton"); dBtn.Parent = btnCard; dBtn.BackgroundColor3 = Color3.fromRGB(46, 204, 113); dBtn.Position = UDim2.new(0.05, 0, 0.1, 0); dBtn.Size = UDim2.new(0.28, 0, 0, 32); dBtn.Text = "直接"; dBtn.TextColor3 = Color3.fromRGB(255, 255, 255); dBtn.TextSize = 12; Instance.new("UICorner", dBtn).CornerRadius = UDim.new(0, 4); dBtn.MouseButton1Click:Connect(function() teleportPlayer("direct") end)
local fBtn = Instance.new("TextButton"); fBtn.Parent = btnCard; fBtn.BackgroundColor3 = Color3.fromRGB(52, 152, 219); fBtn.Position = UDim2.new(0.36, 0, 0.1, 0); fBtn.Size = UDim2.new(0.28, 0, 0, 32); fBtn.Text = "前面"; fBtn.TextColor3 = Color3.fromRGB(255, 255, 255); fBtn.TextSize = 12; Instance.new("UICorner", fBtn).CornerRadius = UDim.new(0, 4); fBtn.MouseButton1Click:Connect(function() teleportPlayer("front") end)
local bBtn = Instance.new("TextButton"); bBtn.Parent = btnCard; bBtn.BackgroundColor3 = Color3.fromRGB(231, 76, 60); bBtn.Position = UDim2.new(0.67, 0, 0.1, 0); bBtn.Size = UDim2.new(0.28, 0, 0, 32); bBtn.Text = "后面"; bBtn.TextColor3 = Color3.fromRGB(255, 255, 255); bBtn.TextSize = 12; Instance.new("UICorner", bBtn).CornerRadius = UDim.new(0, 4); bBtn.MouseButton1Click:Connect(function() teleportPlayer("back") end)
local flingBtn = Instance.new("TextButton"); flingBtn.Parent = tP; flingBtn.BackgroundColor3 = Color3.fromRGB(255, 165, 0); flingBtn.Position = UDim2.new(0, 0, 0, 220); flingBtn.Size = UDim2.new(1, 0, 0, 40); flingBtn.Text = "💥 甩飞一次"; flingBtn.TextColor3 = Color3.fromRGB(255, 255, 255); flingBtn.TextSize = 13; Instance.new("UICorner", flingBtn).CornerRadius = UDim.new(0, 6); flingBtn.MouseButton1Click:Connect(function() if tpTarget and tpTarget.Character then local hrp = tpTarget.Character:FindFirstChild("HumanoidRootPart"); if hrp then hrp.AssemblyLinearVelocity = Vector3.new(0, 120, 0) end end end)

-- ================= 核心改动：直接显示按钮 =================
local openBtn = Instance.new("TextButton")
openBtn.Name = "OpenButton"
openBtn.Parent = screenGui
openBtn.Size = UDim2.new(0, 55, 0, 55)
openBtn.Position = UDim2.new(0.08, 0, 0.5, 0)
openBtn.AnchorPoint = Vector2.new(0.5, 0.5)
openBtn.BackgroundColor3 = Color3.fromRGB(40, 50, 80)
openBtn.Text = "⚡"
openBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
openBtn.TextSize = 28
openBtn.ZIndex = 9999
Instance.new("UICorner", openBtn).CornerRadius = UDim.new(0, 12)
local btnStroke = Instance.new("UIStroke")
btnStroke.Color = Color3.fromRGB(80, 160, 255)
btnStroke.Thickness = 2
btnStroke.Parent = openBtn

openBtn.MouseButton1Click:Connect(function()
    mainFrame.Visible = not mainFrame.Visible
end)

-- ================= 拖拽绑定 =================
local dragging = false; local dragInput, mousePos, framePos
titleBar.InputBegan:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = true; mousePos = input.Position; framePos = mainFrame.Position; input.Changed:Connect(function() if input.UserInputState == Enum.UserInputState.End then dragging = false end end) end end)
titleBar.InputChanged:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then dragInput = input end end)
UserInputService.InputChanged:Connect(function(input) if input == dragInput and dragging then local delta = input.Position - mousePos; mainFrame.Position = UDim2.new(framePos.X.Scale, framePos.X.Offset + delta.X, framePos.Y.Scale, framePos.Y.Offset + delta.Y) end end)

-- ================= FOV圈与自瞄 =================
local fovCircle = Instance.new("Frame"); fovCircle.Name = "FOVCircle"; fovCircle.Parent = screenGui; fovCircle.BackgroundTransparency = 1; fovCircle.AnchorPoint = Vector2.new(0.5, 0.5); fovCircle.Visible = showFOVCircle and AimbotModule.Enabled; local fovStroke = Instance.new("UIStroke"); fovStroke.Color = Color3.fromRGB(255, 180, 0); fovStroke.Thickness = 1.5; fovStroke.Parent = fovCircle; local fovCorner = Instance.new("UICorner"); fovCorner.CornerRadius = UDim.new(1, 0); fovCorner.Parent = fovCircle
local function getClosestTarget() local closestDist = AimbotModule.FOV; local closestTarget = nil; local center = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2); for _, player in ipairs(Players:GetPlayers()) do if player ~= plr then if TeamWhitelist.Locked and TeamWhitelist.TargetTeam then if player.TeamColor ~= TeamWhitelist.TargetTeam then continue end else if AimbotModule.TeamCheck and player.TeamColor == plr.TeamColor then continue end end; if AimbotModule.PlayerBlacklist[player.UserId] then continue end; if #AimbotModule.PlayerWhitelist > 0 and not AimbotModule.PlayerWhitelist[player.UserId] then continue end; local char = player.Character; if char and char:FindFirstChild("HumanoidRootPart") and char:FindFirstChild("Humanoid") and char.Humanoid.Health > 0 then local screenPos, onScreen = camera:WorldToViewportPoint(char.HumanoidRootPart.Position); if onScreen then local dist = (Vector2.new(screenPos.X, screenPos.Y) - center).Magnitude; if dist < closestDist then closestDist = dist; closestTarget = player end end end end end; return closestTarget end
local function performAimbot() if not AimbotModule.Enabled then return end; if not character or not rootPart or not humanoid then return end; local target = getClosestTarget(); if target then currentTarget = target; local targetChar = target.Character; if targetChar and targetChar:FindFirstChild("HumanoidRootPart") then local targetHrp = targetChar.HumanoidRootPart; local targetPart = targetChar:FindFirstChild(AimbotModule.AimPart) or targetHrp; local targetPos = targetPart.Position; if AimbotModule.PredictionEnabled then local targetVel = targetHrp.AssemblyLinearVelocity; local dist = (rootPart.Position - targetPos).Magnitude; local timeToHit = dist / AimbotModule.BulletSpeed; targetPos = targetPos + targetVel * timeToHit end; local dist3D = (rootPart.Position - targetHrp.Position).Magnitude; local dynamicSmooth = AimbotModule.Smoothing; if dist3D < 30 then dynamicSmooth = 0.1 elseif dist3D > 150 then dynamicSmooth = 0.5 end; local lookDir = (targetPos - camera.CFrame.Position).Unit; local targetCFrame = CFrame.lookAt(camera.CFrame.Position, camera.CFrame.Position + lookDir); local lerpAmount = 1 - dynamicSmooth; camera.CFrame = camera.CFrame:Lerp(targetCFrame, lerpAmount) end else currentTarget = nil end end

-- ================= ESP骨骼 =================
local boneJoints = {"Head", "UpperTorso", "LowerTorso", "LeftUpperArm", "LeftLowerArm", "LeftHand", "RightUpperArm", "RightLowerArm", "RightHand", "LeftUpperLeg", "LeftLowerLeg", "LeftFoot", "RightUpperLeg", "RightLowerLeg", "RightFoot"}
local function createSkeletonESP(player) if not player or player == plr then return end; local char = player.Character; if not char then return end; if espCache[player.UserId] then return end; local esp = Instance.new("Folder"); esp.Name = "Skeleton_"..player.Name; esp.Parent = screenGui; local skeletonLines = {}; for i = 1, #boneJoints do local line = Instance.new("Frame"); line.Name = "Bone_"..i; line.Parent = esp; line.BackgroundColor3 = Color3.fromRGB(255,255,255); line.Size = UDim2.new(0,2,0,2); line.BorderSizePixel = 0; line.ZIndex = 20; table.insert(skeletonLines, line) end; local nameLabel = Instance.new("TextLabel"); nameLabel.Parent = esp; nameLabel.BackgroundTransparency = 1; nameLabel.Size = UDim2.new(0,80,0,16); nameLabel.Font = Enum.Font.Gotham; nameLabel.Text = player.Name; nameLabel.TextColor3 = Color3.fromRGB(255,255,255); nameLabel.TextSize = 10; nameLabel.TextStrokeTransparency = 0.5; nameLabel.ZIndex = 19; espCache[player.UserId] = esp; return esp end
local function updateSkeletonESP(esp, char) local function getBonePos(name) local part = char:FindFirstChild(name); if part and part:IsA("BasePart") then return part.Position end return nil end; local bonePositions = {}; for _, boneName in ipairs(boneJoints) do local pos = getBonePos(boneName); table.insert(bonePositions, pos) end; local lines = esp:GetChildren(); local humanoid = char:FindFirstChild("Humanoid"); if not humanoid or humanoid.Health <= 0 then for _, line in ipairs(lines) do if line:IsA("Frame") then line.Visible = false end end return end; local lineIdx = 1; local connections = {{1,2},{2,3},{2,4},{4,5},{5,6},{2,7},{7,8},{8,9},{3,10},{10,11},{11,12},{3,13},{13,14},{14,15}}; for _, conn in ipairs(connections) do local p1 = bonePositions[conn[1]]; local p2 = bonePositions[conn[2]]; if p1 and p2 then local screenP1, onScreen1 = camera:WorldToViewportPoint(p1); local screenP2, onScreen2 = camera:WorldToViewportPoint(p2); if onScreen1 and onScreen2 then local midX = (screenP1.X + screenP2.X)/2; local midY = (screenP1.Y + screenP2.Y)/2; local dx = screenP1.X - screenP2.X; local dy = screenP1.Y - screenP2.Y; local length = math.sqrt(dx*dx + dy*dy); local angle = math.atan2(dy, dx); local line = lines[lineIdx]; if line and line:IsA("Frame") then line.Position = UDim2.new(0, midX - length/2, 0, midY - 1); line.Size = UDim2.new(0, length, 0, 2); line.Rotation = math.deg(angle); line.Visible = true end else local line = lines[lineIdx]; if line and line:IsA("Frame") then line.Visible = false end end else local line = lines[lineIdx]; if line and line:IsA("Frame") then line.Visible = false end end; lineIdx = lineIdx + 1 end; local hrp = char:FindFirstChild("HumanoidRootPart"); if hrp then local screenPos, onScreen = camera:WorldToViewportPoint(hrp.Position); if onScreen then local nameLabel = esp:FindFirstChild("TextLabel"); if nameLabel then nameLabel.Position = UDim2.new(0, screenPos.X - 40, 0, screenPos.Y - 50); nameLabel.Visible = true; nameLabel.Text = player.Name; if player.TeamColor == plr.TeamColor then nameLabel.TextColor3 = Color3.fromRGB(46,204,113) else nameLabel.TextColor3 = Color3.fromRGB(255,80,80) end end end end end
local function updateESP() for userId, esp in pairs(espCache) do local player = Players:GetPlayerByUserId(userId); if not player or not player.Character then pcall(function() esp:Destroy() end); espCache[userId] = nil end end; for _, player in ipairs(Players:GetPlayers()) do if player ~= plr and player.Character then if not espCache[player.UserId] then createSkeletonESP(player) end; local esp = espCache[player.UserId]; if esp then updateSkeletonESP(esp, player.Character) end end end end

-- ================= 主循环 =================
renderConnection = RunService.RenderStepped:Connect(function(dt)
    if not screenGui or not screenGui.Parent then if renderConnection then renderConnection:Disconnect() end return end
    pcall(function() if camera then local r = AimbotModule.FOV * 2.2; fovCircle.Size = UDim2.new(0, r * 2, 0, r * 2); fovCircle.Position = UDim2.new(0, camera.ViewportSize.X / 2, 0, camera.ViewportSize.Y / 2); fovCircle.Visible = showFOVCircle and AimbotModule.Enabled end end)
    pcall(performAimbot)
    if MiscModule.ESPEnabled then pcall(updateESP) else for _, v in pairs(espCache) do pcall(function() v:Destroy() end) end; espCache = {} end
    pcall(function() if FlyCarModule.Enabled then local seat = plr.Character and (plr.Character:FindFirstChild("Seat") or plr.Character:FindFirstChild("VehicleSeat")); if seat and seat.Occupant and seat.Occupant.Parent == plr.Character then local vr = seat.Parent:FindFirstChild("VehicleRootPart") or seat.Parent:FindFirstChild("PrimaryPart") or seat.Parent:FindFirstChild("Body"); if vr then local v = Vector3.new(0,0,0); local h = plr.Character:FindFirstChild("Humanoid"); if h then local d = h.MoveDirection; if d.Magnitude > 0.1 then local r = camera.CFrame:VectorToObjectSpace(d); v = (camera.CFrame.LookVector * -r.Z + camera.CFrame.RightVector * r.X) * FlyCarModule.Speed end end; if carUpP then v = v + Vector3.new(0, FlyCarModule.Speed, 0) end; if carDownP then v = v - Vector3.new(0, FlyCarModule.Speed, 0) end; vr.AssemblyLinearVelocity = v end end end end)
    pcall(function() if not character or not humanoid or not humanoid.Parent then return end; if SpeedModule.Enabled then humanoid.WalkSpeed = SpeedModule.CustomSpeed elseif humanoid.WalkSpeed ~= 16 then humanoid.WalkSpeed = 16 end; if JumpModule.Enabled then humanoid.JumpPower = JumpModule.CustomJump; humanoid.UseJumpPower = true else humanoid.UseJumpPower = false end; if MiscModule.AntiStun and humanoid.PlatformStand and not FlightModule.Enabled then humanoid.PlatformStand = false end end)
    if MiscModule.Noclip and not noclipApplied and character then pcall(function() for _, part in ipairs(character:GetDescendants()) do if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then part.CanCollide = false end end; noclipApplied = true end) elseif not MiscModule.Noclip and noclipApplied and character then pcall(function() for _, part in ipairs(character:GetDescendants()) do if part:IsA("BasePart") then part.CanCollide = true end end; noclipApplied = false end) end
    if FlightModule.Enabled then pcall(function() if not character or not rootPart or not humanoid then return end; if not rootPart.Parent then return end; humanoid.PlatformStand = true; if not camera then return end; local moveDir = humanoid.MoveDirection; local velocity = Vector3.new(0,0,0); if moveDir.Magnitude > 0 then local rel = camera.CFrame:VectorToObjectSpace(moveDir); velocity = (camera.CFrame.LookVector * -rel.Z + camera.CFrame.RightVector * rel.X) * FlightModule.Speed end; if FlightModule.MovingUp then velocity = velocity + Vector3.new(0, FlightModule.Speed, 0) end; if FlightModule.MovingDown then velocity = velocity - Vector3.new(0, FlightModule.Speed, 0) end; rootPart.AssemblyLinearVelocity = velocity end) end
end)

print("✅【小木HUB v6.6 直显版】加载完成！点击左侧⚡按钮打开")
