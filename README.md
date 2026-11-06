--[[
=========================================================
  SORIANO AIMBOT — v3.1 (fechar/abrir painel)
  LocalScript — coloque em StarterPlayerScripts
  Teclas: N=Aim | M=ESP | C=Circulo | F=Fly | G=SuperTiro | H=Painel
=========================================================
]]

--===================== SERVIÇOS =====================--
local Players       = game:GetService("Players")
local RunService    = game:GetService("RunService")
local UIS           = game:GetService("UserInputService")
local VIM           = game:GetService("VirtualInputManager")
local TweenService  = game:GetService("TweenService")

local Camera = workspace.CurrentCamera
local LP     = Players.LocalPlayer
local PG     = LP:WaitForChild("PlayerGui")

--===================== ESTADO =====================--
local lockedTarget = nil
local camActive    = false
local yaw, pitch   = 0, 0
local bursting     = false
local flying       = false
local flyObjects   = {}
local panelOpen    = true

--===================== CONFIG =====================--
local Config = {
    Aim = {
        Enabled    = false,
        FOV        = 260,
        Smoothness = 0.28,
        Part       = "Head",
        MaxDist    = 2000,
    },
    Cam = {
        Sens     = 0.003,
        Distance = 12,
        Height   = 1.4,
    },
    Circle = {
        Enabled   = true,
        Color     = Color3.fromRGB(0, 255, 140),
        Thickness = 2,
    },
    ESP = {
        Enabled     = false,
        Color       = Color3.fromRGB(0, 255, 140),
        MaxDistance = 2500,
    },
    SuperShot = {
        Enabled = false,
        Mult    = 500,
        Batch   = 20,
    },
    Fly = {
        Speed = 65,
    },
    TeamCheck  = true,
    SkipFriend = true,
}

--===================== HELPERS =====================--
local function getMyRoot()
    local char = LP.Character
    return char and char:FindFirstChild("HumanoidRootPart")
end

--===================== FILTROS (AMIGO / TIME) =====================--
local friendCache = {}
local teamCache   = {}
local espData     = {}

local function removeESP(plr)
    local d = espData[plr]
    if not d then return end
    if d.highlight then d.highlight:Destroy() end
    if d.billboard then d.billboard:Destroy() end
    espData[plr] = nil
end

local function clearAllESP()
    local keys = {}
    for plr in pairs(espData) do table.insert(keys, plr) end
    for _, plr in ipairs(keys) do removeESP(plr) end
end

local buildESP

local function hookTeam(plr)
    teamCache[plr] = plr.Team
    plr.Changed:Connect(function(prop)
        if prop == "Team" then
            teamCache[plr] = plr.Team
            if espData[plr] then
                task.spawn(function()
                    removeESP(plr)
                    if buildESP then buildESP(plr) end
                end)
            end
        end
    end)
end

for _, plr in ipairs(Players:GetPlayers()) do hookTeam(plr) end
Players.PlayerAdded:Connect(hookTeam)

local function isFriend(plr)
    if plr == LP then return true end
    if not Config.SkipFriend then return false end
    local cached = friendCache[plr.UserId]
    if cached == true  then return true  end
    if cached == false then return false end
    friendCache[plr.UserId] = false
    task.spawn(function()
        local ok, res = pcall(function() return plr:IsFriendsWith(LP.UserId) end)
        if ok then friendCache[plr.UserId] = res else friendCache[plr.UserId] = nil end
    end)
    return false
end

local function isAlly(plr)
    if not Config.TeamCheck then return false end
    if plr == LP then return true end
    local myTeam    = LP.Team
    local theirTeam = teamCache[plr] or plr.Team
    if myTeam == nil or theirTeam == nil then return false end
    return theirTeam == myTeam
end

local function validTarget(plr)
    if not plr or plr == LP or not plr.Parent then return false end
    if isFriend(plr) or isAlly(plr) then return false end
    local char = plr.Character
    if not char then return false end
    local hum  = char:FindFirstChildOfClass("Humanoid")
    local part = char:FindFirstChild(Config.Aim.Part)
    if not hum or not part or hum.Health <= 0 then return false end
    local myRoot = getMyRoot()
    if not myRoot then return false end
    if (part.Position - myRoot.Position).Magnitude > Config.Aim.MaxDist then return false end
    return true
end

--===================== CÍRCULO FOV =====================--
local overlay = Instance.new("ScreenGui")
overlay.IgnoreGuiInset = true
overlay.ResetOnSpawn   = false
overlay.DisplayOrder   = 20
overlay.Parent         = PG

local circle = Instance.new("Frame")
circle.AnchorPoint            = Vector2.new(0.5, 0.5)
circle.BackgroundTransparency = 1
circle.BorderSizePixel        = 0
circle.Visible                = false
circle.Parent                 = overlay

local circleCorner = Instance.new("UICorner")
circleCorner.CornerRadius = UDim.new(1, 0)
circleCorner.Parent       = circle

local circleStroke = Instance.new("UIStroke")
circleStroke.Color        = Config.Circle.Color
circleStroke.Thickness    = Config.Circle.Thickness
circleStroke.Transparency = 0.1
circleStroke.Parent       = circle

local function refreshCircle()
    local d = Config.Aim.FOV * 2
    circle.Size     = UDim2.fromOffset(d, d)
    circle.Position = UDim2.fromOffset(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    circleStroke.Color     = Config.Circle.Color
    circleStroke.Thickness = Config.Circle.Thickness
end
refreshCircle()

--===================== SUPER TIRO =====================--
local function fireOnce(tool)
    if tool and tool.Parent and tool:IsA("Tool") then
        pcall(function() tool:Activate() end)
        return
    end
    local ok = pcall(function()
        VIM:SendMouseButtonEvent(math.floor(Camera.ViewportSize.X/2), math.floor(Camera.ViewportSize.Y/2), 0, true,  game, 1)
        VIM:SendMouseButtonEvent(math.floor(Camera.ViewportSize.X/2), math.floor(Camera.ViewportSize.Y/2), 0, false, game, 1)
    end)
    if not ok and not fireOnce._warned then
        fireOnce._warned = true
        warn("[SuperShot] Sem tool equipada e VIM indisponível (Studio).")
    end
end

local function superShot()
    if bursting then return end
    local char = LP.Character
    if not char then return end
    local tool  = char:FindFirstChildOfClass("Tool")
    local total = math.max(1, math.floor(Config.SuperShot.Mult))
    local batch = math.max(1, math.floor(Config.SuperShot.Batch))
    bursting = true
    task.spawn(function()
        local sucesso = pcall(function()
            for i = 1, total do
                if not Config.SuperShot.Enabled then break end
                if not LP.Character then break end
                if i % batch == 1 then tool = LP.Character:FindFirstChildOfClass("Tool") end
                fireOnce(tool)
                if i % batch == 0 then task.wait() end
            end
        end)
        if not sucesso then warn("[SuperShot] erro no burst, resetando estado") end
        bursting = false
    end)
end

--===================== ESP =====================--
function buildESP(plr)
    removeESP(plr)
    if not Config.ESP.Enabled or plr == LP then return end
    local char = plr.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hrp or not hum then return end

    local hl = Instance.new("Highlight")
    hl.Adornee          = char
    hl.FillColor        = Config.ESP.Color
    hl.OutlineColor     = Color3.new(1, 1, 1)
    hl.FillTransparency = 0.55
    hl.DepthMode        = Enum.HighlightDepthMode.AlwaysOnTop
    hl.Parent           = char

    local bb = Instance.new("BillboardGui")
    bb.Size        = UDim2.new(0, 220, 0, 56)
    bb.StudsOffset = Vector3.new(0, 3.2, 0)
    bb.AlwaysOnTop = true
    bb.Parent      = hrp

    local name = Instance.new("TextLabel")
    name.Size                   = UDim2.new(1, 0, 0.45, 0)
    name.BackgroundTransparency = 1
    name.Font                   = Enum.Font.GothamBold
    name.TextSize               = 14
    name.TextColor3             = Config.ESP.Color
    name.TextStrokeTransparency = 0.4
    name.Text                   = plr.Name
    name.Parent                 = bb

    local info = Instance.new("TextLabel")
    info.Size                   = UDim2.new(1, 0, 0.35, 0)
    info.Position               = UDim2.new(0, 0, 0.45, 0)
    info.BackgroundTransparency = 1
    info.Font                   = Enum.Font.Gotham
    info.TextSize               = 12
    info.TextColor3             = Color3.new(1, 1, 1)
    info.TextStrokeTransparency = 0.4
    info.Parent                 = bb

    espData[plr] = { highlight = hl, billboard = bb, info = info }
end

task.spawn(function()
    while task.wait(0.15) do
        if not Config.ESP.Enabled then clearAllESP() continue end
        local myRoot = getMyRoot()
        for plr, d in pairs(espData) do
            local char = plr.Character
            local hrp  = char and char:FindFirstChild("HumanoidRootPart")
            local hum  = char and char:FindFirstChildOfClass("Humanoid")
            if not hrp or not hum or hum.Health <= 0 or not myRoot then
                removeESP(plr)
            else
                local dist = math.floor((hrp.Position - myRoot.Position).Magnitude)
                local show = dist <= Config.ESP.MaxDistance and not isFriend(plr) and not isAlly(plr)
                if d.highlight then d.highlight.Enabled = show end
                if d.billboard then d.billboard.Enabled = show end
                if show then d.info.Text = string.format("%d HP | %dm", math.floor(hum.Health), dist) end
            end
        end
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LP and not espData[plr] and plr.Character then buildESP(plr) end
        end
    end
end)

local function hookPlayer(plr)
    if plr == LP then return end
    plr.CharacterAdded:Connect(function()
        task.spawn(function() task.wait(0.5) buildESP(plr) end)
    end)
    if plr.Character then
        task.spawn(function() task.wait(0.5) buildESP(plr) end)
    end
end

for _, plr in ipairs(Players:GetPlayers()) do hookPlayer(plr) end
Players.PlayerAdded:Connect(hookPlayer)
Players.PlayerRemoving:Connect(function(plr)
    removeESP(plr)
    teamCache[plr] = nil
    if lockedTarget == plr then lockedTarget = nil end
end)

--===================== AIMBOT =====================--
local function acquireTarget()
    local best, bestDist = nil, Config.Aim.FOV
    local center = Camera.ViewportSize / 2
    for _, plr in ipairs(Players:GetPlayers()) do
        if validTarget(plr) then
            local part = plr.Character[Config.Aim.Part]
            local sp, onScreen = Camera:WorldToViewportPoint(part.Position)
            if onScreen then
                local d = (Vector2.new(sp.X, sp.Y) - center).Magnitude
                if d <= bestDist then best, bestDist = plr, d end
            end
        end
    end
    return best
end

local function syncAngles()
    local look = Camera.CFrame.LookVector
    yaw   = math.atan2(-look.X, -look.Z)
    pitch = math.asin(math.clamp(look.Y, -1, 1))
end

local function setAim(state)
    if state then
        local char = LP.Character
        local hum  = char and char:FindFirstChildOfClass("Humanoid")
        if not hum then return end
        syncAngles()
        Camera.CameraType  = Enum.CameraType.Scriptable
        camActive          = true
        Config.Aim.Enabled = true
    else
        Config.Aim.Enabled = false
        camActive          = false
        lockedTarget       = nil
        local char = LP.Character
        local hum  = char and char:FindFirstChildOfClass("Humanoid")
        Camera.CameraType = Enum.CameraType.Custom
        if hum then Camera.CameraSubject = hum end
    end
end

local function aimStep()
    if not camActive then circle.Visible = false return end
    local char = LP.Character
    local hum  = char and char:FindFirstChildOfClass("Humanoid")
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not hum or not root or hum.Health <= 0 then return end

    circle.Visible = Config.Circle.Enabled
    if Config.Circle.Enabled then
        circle.Position = UDim2.fromOffset(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    end

    if not validTarget(lockedTarget) then lockedTarget = acquireTarget() end
    local originPos = root.Position + Vector3.new(0, Config.Cam.Height, 0)

    if lockedTarget then
        local part = lockedTarget.Character and lockedTarget.Character[Config.Aim.Part]
        if part then
            local dir = part.Position - originPos
            if dir.Magnitude > 0.5 then
                local wantYaw   = math.atan2(-dir.X, -dir.Z)
                local wantPitch = math.asin(math.clamp(dir.Unit.Y, -1, 1))
                local dy = (wantYaw - yaw + math.pi) % (2 * math.pi) - math.pi
                local a  = math.clamp(1 - Config.Aim.Smoothness, 0.05, 1)
                yaw   += dy * a
                pitch += (wantPitch - pitch) * a
            end
        end
    end

    pitch = math.clamp(pitch, -1.5, 1.5)
    local rot = CFrame.fromOrientation(pitch, yaw, 0)
    local pos = originPos - rot.LookVector * Config.Cam.Distance
    Camera.CFrame = CFrame.new(pos) * rot
end

pcall(function() RunService:UnbindFromRenderStep("sorianoAim") end)
RunService:BindToRenderStep("sorianoAim", Enum.RenderPriority.Camera.Value + 1, aimStep)

UIS.InputChanged:Connect(function(input)
    if not camActive then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        yaw   -= input.Delta.X * Config.Cam.Sens
        pitch  = math.clamp(pitch - input.Delta.Y * Config.Cam.Sens, -1.5, 1.5)
    elseif input.UserInputType == Enum.UserInputType.Touch then
        yaw   -= input.Delta.X * Config.Cam.Sens * 1.8
        pitch  = math.clamp(pitch - input.Delta.Y * Config.Cam.Sens * 1.8, -1.5, 1.5)
    end
end)

LP.CharacterAdded:Connect(function()
    if camActive then
        task.wait(0.8)
        if camActive then
            local hum = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
            if hum then
                Camera.CameraType = Enum.CameraType.Scriptable
                syncAngles()
            end
        end
    end
end)

--===================== FLY =====================--
local function setFly(state)
    flying = state
    local char = LP.Character
    local hrp  = char and char:FindFirstChild("HumanoidRootPart")
    local hum  = char and char:FindFirstChildOfClass("Humanoid")
    for _, obj in ipairs(flyObjects) do obj:Destroy() end
    flyObjects = {}
    if not flying then
        if hum then hum.PlatformStand = false end
        return
    end
    if not hrp or not hum then flying = false return end

    local att = Instance.new("Attachment")
    att.Parent = hrp

    local lv = Instance.new("LinearVelocity")
    lv.Attachment0            = att
    lv.MaxForce               = math.huge
    lv.VelocityConstraintMode = Enum.VelocityConstraintMode.Vector
    lv.VectorVelocity         = Vector3.zero
    lv.Parent                 = hrp

    local ao = Instance.new("AlignOrientation")
    ao.Mode        = Enum.OrientationAlignmentMode.OneAttachment
    ao.Attachment0 = att
    ao.MaxTorque   = math.huge
    ao.CFrame      = Camera.CFrame
    ao.Parent      = hrp

    flyObjects = { lv, ao, att }
    hum.PlatformStand = true
end

RunService.RenderStepped:Connect(function()
    if not flying then return end
    local lv, ao = flyObjects[1], flyObjects[2]
    if not lv or not ao then return end
    local dir = Vector3.zero
    if UIS:IsKeyDown(Enum.KeyCode.W)           then dir += Camera.CFrame.LookVector  end
    if UIS:IsKeyDown(Enum.KeyCode.S)           then dir -= Camera.CFrame.LookVector  end
    if UIS:IsKeyDown(Enum.KeyCode.A)           then dir -= Camera.CFrame.RightVector end
    if UIS:IsKeyDown(Enum.KeyCode.D)           then dir += Camera.CFrame.RightVector end
    if UIS:IsKeyDown(Enum.KeyCode.Space)       then dir += Vector3.new(0, 1, 0)     end
    if UIS:IsKeyDown(Enum.KeyCode.LeftControl) then dir -= Vector3.new(0, 1, 0)     end
    lv.VectorVelocity = dir.Magnitude > 0 and dir.Unit * Config.Fly.Speed or Vector3.zero
    ao.CFrame = Camera.CFrame
end)

LP.CharacterAdded:Connect(function()
    if flying then
        task.wait(0.6)
        setFly(false)
        setFly(true)
    end
end)

--===================== PAINEL UI =====================--
local panelGui = Instance.new("ScreenGui")
panelGui.ResetOnSpawn = false
panelGui.DisplayOrder = 50
panelGui.Parent       = PG

local panel = Instance.new("Frame")
panel.Size                   = UDim2.new(0, 232, 0, 494)
panel.Position               = UDim2.new(0, 18, 0, 70)
panel.BackgroundColor3       = Color3.fromRGB(12, 14, 18)
panel.BackgroundTransparency = 0.28
panel.BorderSizePixel        = 0
panel.Parent                 = panelGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 10)
corner.Parent       = panel

local stroke = Instance.new("UIStroke")
stroke.Color        = Config.Circle.Color
stroke.Transparency = 0.55
stroke.Parent       = panel

-- Botão de fechar (✕)
local closeBtn = Instance.new("TextButton")
closeBtn.Size                   = UDim2.new(0, 26, 0, 26)
closeBtn.Position               = UDim2.new(1, -30, 0, 4)
closeBtn.BackgroundColor3       = Color3.fromRGB(150, 45, 45)
closeBtn.BackgroundTransparency = 0.15
closeBtn.BorderSizePixel        = 0
closeBtn.Font                   = Enum.Font.GothamBold
closeBtn.TextSize               = 14
closeBtn.TextColor3             = Color3.new(1, 1, 1)
closeBtn.Text                   = "✕"
closeBtn.Parent                 = panel

local closeCorner = Instance.new("UICorner")
closeCorner.CornerRadius = UDim.new(0, 6)
closeCorner.Parent       = closeBtn

-- Título
local title = Instance.new("TextLabel")
title.Size                   = UDim2.new(1, -70, 0, 34)
title.BackgroundColor3       = Color3.new(0, 0, 0)
title.BackgroundTransparency = 0.45
title.BorderSizePixel        = 0
title.Font                   = Enum.Font.GothamBold
title.TextSize               = 13
title.TextColor3             = Config.Circle.Color
title.Text                   = "SORIANO AIMBOT v3.1"
title.Active                 = true
title.Parent                 = panel

local titleCorner = Instance.new("UICorner")
titleCorner.CornerRadius = UDim.new(0, 10)
titleCorner.Parent       = title

local status = Instance.new("TextLabel")
status.Size                   = UDim2.new(1, -20, 0, 16)
status.Position               = UDim2.new(0, 10, 0, 36)
status.BackgroundTransparency = 1
status.Font                   = Enum.Font.Code
status.TextSize               = 10
status.TextXAlignment         = Enum.TextXAlignment.Left
status.TextColor3             = Color3.fromRGB(120, 220, 170)
status.Text                   = "alvo: nenhum"
status.Parent                 = panel

-- Drag
local dragging, dragStart, startPos = false, nil, nil
title.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
    or input.UserInputType == Enum.UserInputType.Touch then
        dragging  = true
        dragStart = input.Position
        startPos  = panel.Position
    end
end)
UIS.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
                     or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        panel.Position = UDim2.new(
            startPos.X.Scale, startPos.X.Offset + delta.X,
            startPos.Y.Scale, startPos.Y.Offset + delta.Y
        )
    end
end)
UIS.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
    or input.UserInputType == Enum.UserInputType.Touch then
        dragging = false
    end
end)

-- Fábricas
local refreshers = {}

local function makeToggle(text, y, getter, setter)
    local btn = Instance.new("TextButton")
    btn.Size                   = UDim2.new(1, -20, 0, 28)
    btn.Position               = UDim2.new(0, 10, 0, y)
    btn.BackgroundColor3       = Color3.fromRGB(40, 44, 52)
    btn.BackgroundTransparency = 0.2
    btn.BorderSizePixel        = 0
    btn.AutoButtonColor        = false
    btn.Font                   = Enum.Font.GothamBold
    btn.TextSize               = 12
    btn.TextColor3             = Color3.new(1, 1, 1)
    btn.Parent                 = panel

    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, 6)
    c.Parent       = btn

    local function refresh()
        local on = getter()
        btn.BackgroundColor3 = on and Color3.fromRGB(0, 170, 95) or Color3.fromRGB(40, 44, 52)
        btn.Text             = text .. (on and "  [ON]" or "  [OFF]")
        btn.TextColor3       = on and Color3.new(1, 1, 1) or Color3.fromRGB(190, 190, 190)
    end

    btn.MouseButton1Click:Connect(function()
        setter(not getter())
        refresh()
    end)
    refreshers[text] = refresh
    refresh()
    return refresh
end

local function makeStepper(label, y, getter, setter, step, min, max, fmt, onChange)
    local box = Instance.new("Frame")
    box.Size                   = UDim2.new(1, -20, 0, 26)
    box.Position               = UDim2.new(0, 10, 0, y)
    box.BackgroundTransparency = 1
    box.Parent                 = panel

    local lbl = Instance.new("TextLabel")
    lbl.Size                   = UDim2.new(0.6, 0, 1, 0)
    lbl.BackgroundTransparency = 1
    lbl.Font                   = Enum.Font.Gotham
    lbl.TextSize               = 11
    lbl.TextColor3             = Color3.fromRGB(200, 200, 200)
    lbl.TextXAlignment         = Enum.TextXAlignment.Left
    lbl.Parent                 = box

    local minus = Instance.new("TextButton")
    minus.Size             = UDim2.new(0, 26, 1, 0)
    minus.Position         = UDim2.new(0.6, 0, 0, 0)
    minus.BackgroundColor3 = Color3.fromRGB(40, 44, 52)
    minus.BorderSizePixel  = 0
    minus.Font             = Enum.Font.GothamBold
    minus.TextColor3       = Color3.new(1, 1, 1)
    minus.Text             = "-"
    minus.Parent           = box

    local plus = minus:Clone()
    plus.Position = UDim2.new(1, -26, 0, 0)
    plus.Text     = "+"
    plus.Parent   = box

    for _, b in ipairs({ minus, plus }) do
        local c = Instance.new("UICorner")
        c.CornerRadius = UDim.new(0, 6)
        c.Parent       = b
    end

    local function refresh()
        lbl.Text = string.format("%s: %s", label, fmt(getter()))
    end

    minus.MouseButton1Click:Connect(function()
        setter(math.clamp(getter() - step, min, max))
        refresh()
        if onChange then onChange() end
    end)
    plus.MouseButton1Click:Connect(function()
        setter(math.clamp(getter() + step, min, max))
        refresh()
        if onChange then onChange() end
    end)
    refresh()
end

-- Controles
makeToggle("AIMBOT (N)", 58,
    function() return Config.Aim.Enabled end,
    function(v) setAim(v) end)

local refreshEsp = makeToggle("ESP (M)", 92,
    function() return Config.ESP.Enabled end,
    function(v)
        Config.ESP.Enabled = v
        if v then
            for _, plr in ipairs(Players:GetPlayers()) do buildESP(plr) end
        else
            clearAllESP()
        end
    end)

makeToggle("CIRCULO (C)", 126,
    function() return Config.Circle.Enabled end,
    function(v) Config.Circle.Enabled = v end)

makeToggle("VOAR (F)", 160,
    function() return flying end,
    function(v) setFly(v) end)

makeToggle("SUPER TIRO (G)", 194,
    function() return Config.SuperShot.Enabled end,
    function(v)
        Config.SuperShot.Enabled = v
        if v and (not LP.Character or not LP.Character:FindFirstChildOfClass("Tool")) then
            warn("[SuperShot] Equipe uma Tool para disparar!")
        end
    end)

makeToggle("IGNORAR AMIGOS", 228,
    function() return Config.SkipFriend end,
    function(v) Config.SkipFriend = v; if v then friendCache = {} end end)

makeToggle("IGNORAR TIME", 262,
    function() return Config.TeamCheck end,
    function(v) Config.TeamCheck = v end)

makeStepper("RAIO DO CIRCULO", 298,
    function() return Config.Aim.FOV end,
    function(v) Config.Aim.FOV = v end,
    20, 40, 500,
    function(v) return tostring(v) end,
    refreshCircle)

makeStepper("SUAVIDADE", 328,
    function() return Config.Aim.Smoothness end,
    function(v) Config.Aim.Smoothness = v end,
    0.05, 0.05, 0.95,
    function(v) return string.format("%.2f", v) end)

makeStepper("ALCANCE (STUDS)", 358,
    function() return Config.Aim.MaxDist end,
    function(v) Config.Aim.MaxDist = v end,
    100, 100, 2500,
    function(v) return tostring(v) end)

makeStepper("DISTANCIA CAM", 388,
    function() return Config.Cam.Distance end,
    function(v) Config.Cam.Distance = v end,
    2, 4, 12,
    function(v) return tostring(v) end)

makeStepper("TIROS POR CLIQUE", 418,
    function() return Config.SuperShot.Mult end,
    function(v) Config.SuperShot.Mult = v end,
    50, 10, 1000,
    function(v) return tostring(v) end)

-- Reset total
local reset = Instance.new("TextButton")
reset.Size                   = UDim2.new(1, -20, 0, 26)
reset.Position               = UDim2.new(0, 10, 0, 452)
reset.BackgroundColor3       = Color3.fromRGB(150, 45, 45)
reset.BackgroundTransparency = 0.15
reset.BorderSizePixel        = 0
reset.Font                   = Enum.Font.GothamBold
reset.TextSize               = 11
reset.TextColor3             = Color3.new(1, 1, 1)
reset.Text                   = "PARAR TUDO"
reset.Parent                 = panel

local rc = Instance.new("UICorner")
rc.CornerRadius = UDim.new(0, 6)
rc.Parent       = reset

reset.MouseButton1Click:Connect(function()
    setAim(false)
    Config.SuperShot.Enabled = false
    bursting = false
    Config.ESP.Enabled = false
    clearAllESP()
    if flying then setFly(false) end
    friendCache = {}
    teamCache   = {}
    for _, r in pairs(refreshers) do r() end
end)

--===================== BOTÃO FLUTUANTE (ABRIR) =====================--
local openBtn = Instance.new("TextButton")
openBtn.Size                   = UDim2.new(0, 42, 0, 42)
openBtn.Position               = UDim2.new(0, 18, 0, 70)
openBtn.BackgroundColor3       = Color3.fromRGB(12, 14, 18)
openBtn.BackgroundTransparency = 0.15
openBtn.BorderSizePixel        = 0
openBtn.Font                   = Enum.Font.GothamBold
openBtn.TextSize               = 20
openBtn.TextColor3             = Config.Circle.Color
openBtn.Text                   = "☰"
openBtn.Visible                = false
openBtn.Parent                 = panelGui

local openCorner = Instance.new("UICorner")
openCorner.CornerRadius = UDim.new(0, 10)
openCorner.Parent       = openBtn

local openStroke = Instance.new("UIStroke")
openStroke.Color        = Config.Circle.Color
openStroke.Transparency = 0.45
openStroke.Parent       = openBtn

--===================== TOGGLE FECHAR/ABRIR =====================--
local function setPanelOpen(state)
    panelOpen = state
    panel.Visible   = state
    openBtn.Visible = not state
end

closeBtn.MouseButton1Click:Connect(function() setPanelOpen(false) end)
openBtn.MouseButton1Click:Connect(function() setPanelOpen(true) end)

-- Status em loop
task.spawn(function()
    while task.wait(0.1) do
        if lockedTarget and lockedTarget.Parent and validTarget(lockedTarget) then
            local ok, dist = pcall(function()
                local part = lockedTarget.Character and lockedTarget.Character[Config.Aim.Part]
                if not part then return nil end
                return math.floor((part.Position - Camera.CFrame.Position).Magnitude)
            end)
            if ok and dist then
                status.Text       = "TRAVADO: " .. lockedTarget.Name .. " | " .. dist
                status.TextColor3 = Color3.fromRGB(120, 255, 150)
            else
                status.Text       = "alvo perdido: " .. lockedTarget.Name
                status.TextColor3 = Color3.fromRGB(255, 180, 120)
            end
        else
            status.Text       = Config.Aim.Enabled and "alvo: procurando no circulo..." or "alvo: nenhum"
            status.TextColor3 = Color3.fromRGB(180, 180, 120)
        end
    end
end)

--===================== INPUT =====================--
UIS.InputBegan:Connect(function(input, processed)
    if input.UserInputType == Enum.UserInputType.MouseButton1
    and Config.SuperShot.Enabled and not processed then
        superShot()
    end

    if processed then return end

    if input.KeyCode == Enum.KeyCode.M then
        Config.ESP.Enabled = not Config.ESP.Enabled
        if Config.ESP.Enabled then
            for _, plr in ipairs(Players:GetPlayers()) do buildESP(plr) end
        else
            clearAllESP()
        end
        if refreshEsp then refreshEsp() end
    elseif input.KeyCode == Enum.KeyCode.N then
        setAim(not Config.Aim.Enabled)
        if refreshers["AIMBOT (N)"] then refreshers["AIMBOT (N)"]() end
    elseif input.KeyCode == Enum.KeyCode.C then
        Config.Circle.Enabled = not Config.Circle.Enabled
        if refreshers["CIRCULO (C)"] then refreshers["CIRCULO (C)"]() end
    elseif input.KeyCode == Enum.KeyCode.F then
        setFly(not flying)
        if refreshers["VOAR (F)"] then refreshers["VOAR (F)"]() end
    elseif input.KeyCode == Enum.KeyCode.G then
        Config.SuperShot.Enabled = not Config.SuperShot.Enabled
        if refreshers["SUPER TIRO (G)"] then refreshers["SUPER TIRO (G)"]() end
    elseif input.KeyCode == Enum.KeyCode.H then
        setPanelOpen(not panelOpen)
    end
end)

print("[SORIANO AIMBOT v3.1] carregado | N=Aim M=ESP C=Circulo F=Fly G=SuperTiro H=Painel")
