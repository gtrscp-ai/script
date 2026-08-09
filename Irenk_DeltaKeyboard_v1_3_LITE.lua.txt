--[[
    Irenk Delta Keyboard v1.3 LITE
    ONE FILE LocalScript - Delta/mobile friendly

    Focus v1.3 Lite:
    - Stable D-Pad movement only
    - Basic action buttons
    - Simple coloring/transparency
    - Edit mode drag
    - Scale +5% / -5%
    - Mouse click offset calibration
    - Save / Reset / Kill

    Removed from v1.2 Advanced for stability:
    - Custom button creator
    - HUD customizer
    - Canvas/custom shapes
    - Multi-language heavy dictionary
    - Complex tabs
]]

---------------------------------------------------------------------
-- SERVICES
---------------------------------------------------------------------

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local HttpService = game:GetService("HttpService")
local StatsService = game:GetService("Stats")

local VIM = nil
pcall(function()
    VIM = game:GetService("VirtualInputManager")
end)

local LocalPlayer = Players.LocalPlayer

---------------------------------------------------------------------
-- CONFIG
---------------------------------------------------------------------

local Config = {
    Version = "1.3-LITE",
    SaveFileName = "Irenk_DeltaKeyboard_v13_Lite_Save.json",

    Scale = 1,
    ButtonTransparency = 0.20,
    DPadTransparency = 0.26,

    DPadPosition = {0, 34, 1, -230},
    DPadSize = {0, 190, 0, 190},

    MouseOffsets = {
        Global = {X = 0, Y = 0},
        Left = {X = 0, Y = 0},
        Right = {X = 0, Y = 0}
    },

    Colors = {
        Button = {255, 88, 105},
        DPad = {255, 145, 155},
        Stick = {255, 70, 82},
        Text = {255, 255, 255},
        Outline = {0, 0, 0}
    },

    Buttons = {
        {Id="SHIFT", Title="SHIFT", Type="Key", KeyName="LeftShift", Mode="Toggle", Position={0, 250, 1, -136}, Size={0, 150, 0, 58}},
        {Id="SPACE", Title="space", Type="Key", KeyName="Space", Mode="Tap", Position={1, -165, 1, -84}, Size={0, 145, 0, 60}},
        {Id="LEFT", Title="LEFT", Type="Mouse", MouseButton=0, Mode="Hold", Position={0.70, 0, 1, -145}, Size={0, 160, 0, 70}},
        {Id="RIGHT", Title="right", Type="Mouse", MouseButton=1, Mode="Hold", Position={1, -150, 0.42, 0}, Size={0, 120, 0, 65}},
        {Id="R", Title="R", Type="Key", KeyName="R", Mode="Tap", Position={0.60, 0, 1, -210}, Size={0, 82, 0, 62}},
        {Id="E", Title="E", Type="Key", KeyName="E", Mode="Tap", Position={0.66, 0, 1, -82}, Size={0, 88, 0, 58}},
        {Id="C", Title="C", Type="Key", KeyName="C", Mode="Hold", Position={0.56, 0, 1, -130}, Size={0, 82, 0, 62}}
    }
}

local Theme = {
    Panel = Color3.fromRGB(28, 18, 42),
    Panel2 = Color3.fromRGB(42, 26, 66),
    Pink = Color3.fromRGB(255, 88, 105),
    Pink2 = Color3.fromRGB(255, 130, 140),
    Green = Color3.fromRGB(90, 255, 170),
    Yellow = Color3.fromRGB(255, 225, 110),
    Red = Color3.fromRGB(255, 72, 95),
    Blue = Color3.fromRGB(75, 145, 255),
    White = Color3.fromRGB(255, 255, 255),
    Black = Color3.fromRGB(0, 0, 0),
    Muted = Color3.fromRGB(210, 198, 224)
}

---------------------------------------------------------------------
-- STATE
---------------------------------------------------------------------

local CAN_FILES = type(writefile) == "function" and type(readfile) == "function" and type(isfile) == "function"
local Alive = true
local EditMode = false
local SettingsOpen = false
local SelectedId = "DPAD"
local MouseTarget = "Global"

local Connections = {}
local RenderConnections = {}
local ActiveInputs = {}
local ToggleStates = {}
local DPadKeys = {W=false, A=false, S=false, D=false}
local DPadActiveInput = nil

local ScreenGui, ButtonLayer, DPadFrame, StickFrame, SettingsButton, SettingsPanel, Notice
local StatsFrame, FpsLabel, PingLabel, StatusLabel

---------------------------------------------------------------------
-- HELPERS
---------------------------------------------------------------------

local function add(list, conn)
    table.insert(list or Connections, conn)
    return conn
end

local function disconnectList(list)
    for _, conn in ipairs(list) do
        pcall(function() conn:Disconnect() end)
    end
    for i = #list, 1, -1 do
        table.remove(list, i)
    end
end

local function safeSet(obj, prop, value)
    pcall(function()
        obj[prop] = value
    end)
end

local function new(className, props)
    local obj = Instance.new(className)
    for k, v in pairs(props or {}) do
        safeSet(obj, k, v)
    end
    return obj
end

local function corner(obj, radius)
    local c = new("UICorner", {CornerRadius = UDim.new(0, radius or 10)})
    c.Parent = obj
    return c
end

local function stroke(obj, color, thickness, transparency)
    local s = new("UIStroke", {
        Color = color or Theme.Black,
        Thickness = thickness or 2,
        Transparency = transparency or 0
    })
    s.Parent = obj
    return s
end

local function tween(obj, props, time)
    pcall(function()
        TweenService:Create(obj, TweenInfo.new(time or 0.14, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), props):Play()
    end)
end

local function clamp(n, a, b)
    n = tonumber(n) or a
    if n < a then return a end
    if n > b then return b end
    return n
end

local function colorFromTable(t, fallback)
    fallback = fallback or Theme.White
    if type(t) ~= "table" then return fallback end
    return Color3.fromRGB(clamp(t[1], 0, 255), clamp(t[2], 0, 255), clamp(t[3], 0, 255))
end

local function colorToTable(c)
    return {math.floor(c.R * 255 + 0.5), math.floor(c.G * 255 + 0.5), math.floor(c.B * 255 + 0.5)}
end

local function tableToUDim2(t)
    if type(t) ~= "table" then return UDim2.new(0,0,0,0) end
    return UDim2.new(tonumber(t[1]) or 0, tonumber(t[2]) or 0, tonumber(t[3]) or 0, tonumber(t[4]) or 0)
end

local function udim2ToTable(u)
    return {u.X.Scale, u.X.Offset, u.Y.Scale, u.Y.Offset}
end

local function scaleUDim2(u)
    local s = Config.Scale or 1
    return UDim2.new(u.X.Scale, math.floor(u.X.Offset * s), u.Y.Scale, math.floor(u.Y.Offset * s))
end

local function getParentGui()
    local p = nil
    pcall(function()
        if gethui then p = gethui() end
    end)
    if not p then pcall(function() p = game:GetService("CoreGui") end) end
    if not p and LocalPlayer then p = LocalPlayer:WaitForChild("PlayerGui") end
    return p
end

local function notify(text, bad)
    if not Notice or not Notice.Parent then return end
    Notice.Text = tostring(text or "")
    Notice.TextColor3 = bad and Theme.Red or Theme.Green
    Notice.Visible = true
    Notice.Position = UDim2.new(0.5, 0, 0, -42)
    tween(Notice, {Position = UDim2.new(0.5, 0, 0, 18)}, 0.16)
    task.spawn(function()
        task.wait(1.2)
        if Alive and Notice and Notice.Parent then
            tween(Notice, {Position = UDim2.new(0.5, 0, 0, -42)}, 0.16)
            task.wait(0.2)
            if Notice and Notice.Parent then Notice.Visible = false end
        end
    end)
end

local function findButton(id)
    for _, b in ipairs(Config.Buttons) do
        if b.Id == id then return b end
    end
    return nil
end

local function releaseAll()
    for _, b in ipairs(Config.Buttons) do
        if ActiveInputs[b.Id] or ToggleStates[b.Id] then
            ActiveInputs[b.Id] = nil
            ToggleStates[b.Id] = nil
        end
    end
    for _, k in ipairs({"W","A","S","D","LeftShift","Space","R","E","C"}) do
        if VIM then pcall(function() VIM:SendKeyEvent(false, Enum.KeyCode[k], false, game) end) end
    end
    if VIM then
        pcall(function()
            VIM:SendMouseButtonEvent(0,0,0,false,game,0)
            VIM:SendMouseButtonEvent(0,0,1,false,game,0)
        end)
    end
end

---------------------------------------------------------------------
-- SAVE / LOAD
---------------------------------------------------------------------

local function saveConfig(show)
    if not CAN_FILES then
        if show then notify("Save unsupported", true) end
        return false
    end

    local ok = pcall(function()
        writefile(Config.SaveFileName, HttpService:JSONEncode(Config))
    end)

    if show then notify(ok and "Saved" or "Save failed", not ok) end
    return ok
end

local function loadConfig()
    if not CAN_FILES then return false end
    local exists = false
    pcall(function() exists = isfile(Config.SaveFileName) end)
    if not exists then return false end

    local raw
    local okRead = pcall(function() raw = readfile(Config.SaveFileName) end)
    if not okRead or not raw then return false end

    local data
    local okDecode = pcall(function() data = HttpService:JSONDecode(raw) end)
    if not okDecode or type(data) ~= "table" then return false end

    if type(data.Scale) == "number" then Config.Scale = clamp(data.Scale, 0.5, 2.5) end
    if type(data.ButtonTransparency) == "number" then Config.ButtonTransparency = clamp(data.ButtonTransparency, 0, 0.8) end
    if type(data.DPadTransparency) == "number" then Config.DPadTransparency = clamp(data.DPadTransparency, 0, 0.8) end
    if type(data.DPadPosition) == "table" then Config.DPadPosition = data.DPadPosition end
    if type(data.DPadSize) == "table" then Config.DPadSize = data.DPadSize end
    if type(data.MouseOffsets) == "table" then Config.MouseOffsets = data.MouseOffsets end
    if type(data.Colors) == "table" then Config.Colors = data.Colors end
    if type(data.Buttons) == "table" then Config.Buttons = data.Buttons end

    return true
end

---------------------------------------------------------------------
-- INPUT
---------------------------------------------------------------------

local function sendKey(keyName, down)
    if not VIM then return false end
    local key = Enum.KeyCode[keyName]
    if not key then return false end
    return pcall(function()
        VIM:SendKeyEvent(down, key, false, game)
    end)
end

local function getMousePos(mouseButton)
    local cam = workspace.CurrentCamera
    local x, y = 400, 300
    if cam then
        local vp = cam.ViewportSize
        x = math.floor(vp.X / 2)
        y = math.floor(vp.Y / 2)
    end
    local g = Config.MouseOffsets.Global or {X=0,Y=0}
    local l = Config.MouseOffsets.Left or {X=0,Y=0}
    local r = Config.MouseOffsets.Right or {X=0,Y=0}
    local s = mouseButton == 1 and r or l
    return x + (tonumber(g.X) or 0) + (tonumber(s.X) or 0), y + (tonumber(g.Y) or 0) + (tonumber(s.Y) or 0)
end

local function sendMouse(button, down)
    if not VIM then return false end
    local x, y = getMousePos(button)
    return pcall(function()
        if VIM.SendMouseMoveEvent then VIM:SendMouseMoveEvent(x, y, game) end
        VIM:SendMouseButtonEvent(x, y, button, down, game, 0)
    end)
end

local function sendButton(btn, down)
    if btn.Type == "Mouse" then
        return sendMouse(tonumber(btn.MouseButton) or 0, down)
    else
        return sendKey(btn.KeyName, down)
    end
end

local function pressButton(btn, input)
    SelectedId = btn.Id
    if btn.Mode == "Tap" then
        local ok = sendButton(btn, true)
        task.delay(0.07, function()
            if Alive then sendButton(btn, false) end
        end)
        return ok
    elseif btn.Mode == "Toggle" then
        local state = not ToggleStates[btn.Id]
        ToggleStates[btn.Id] = state
        ActiveInputs[btn.Id] = state and true or nil
        sendButton(btn, state)
        return true
    else
        if ActiveInputs[btn.Id] then return true end
        ActiveInputs[btn.Id] = input or true
        return sendButton(btn, true)
    end
end

local function releaseButton(btn, input)
    if btn.Mode ~= "Hold" then return end
    local active = ActiveInputs[btn.Id]
    if not active then return end
    if input and active ~= true and active ~= input then return end
    ActiveInputs[btn.Id] = nil
    sendButton(btn, false)
end

---------------------------------------------------------------------
-- DPAD
---------------------------------------------------------------------

local function setDPadKeys(keys)
    for _, k in ipairs({"W","A","S","D"}) do
        local want = keys[k] == true
        if DPadKeys[k] ~= want then
            DPadKeys[k] = want
            sendKey(k, want)
        end
    end
end

local function resetDPad()
    setDPadKeys({})
    if StickFrame then
        tween(StickFrame, {Position = UDim2.new(0.5, 0, 0.5, 0)}, 0.08)
    end
end

local function updateDPad(input)
    if not DPadFrame then return end
    local center = DPadFrame.AbsolutePosition + (DPadFrame.AbsoluteSize / 2)
    local delta = input.Position - center
    local radius = math.max(1, math.min(DPadFrame.AbsoluteSize.X, DPadFrame.AbsoluteSize.Y) / 2)
    local x = clamp(delta.X / radius, -1, 1)
    local y = clamp(delta.Y / radius, -1, 1)
    local dead = 0.18
    local keys = {}
    if y < -dead then keys.W = true end
    if y > dead then keys.S = true end
    if x < -dead then keys.A = true end
    if x > dead then keys.D = true end
    setDPadKeys(keys)

    if StickFrame then
        local maxMove = radius * 0.42
        StickFrame.Position = UDim2.new(0.5, x * maxMove, 0.5, y * maxMove)
    end
end

---------------------------------------------------------------------
-- GUI BASE
---------------------------------------------------------------------

local function createBaseGui()
    local parent = getParentGui()
    pcall(function()
        local old = parent:FindFirstChild("IrenkDeltaKeyboardV13Lite")
        if old then old:Destroy() end
    end)

    ScreenGui = new("ScreenGui", {
        Name = "IrenkDeltaKeyboardV13Lite",
        ResetOnSpawn = false,
        IgnoreGuiInset = true,
        DisplayOrder = 9999999,
        ZIndexBehavior = Enum.ZIndexBehavior.Global
    })
    ScreenGui.Parent = parent

    ButtonLayer = new("Frame", {
        Parent = ScreenGui,
        BackgroundTransparency = 1,
        Position = UDim2.new(0,0,0,0),
        Size = UDim2.new(1,0,1,0),
        ZIndex = 10
    })

    StatsFrame = new("Frame", {
        Parent = ScreenGui,
        Position = UDim2.new(0, 12, 0, 78),
        Size = UDim2.new(0, 150, 0, 56),
        BackgroundColor3 = Theme.Panel,
        BackgroundTransparency = 0.08,
        BorderSizePixel = 0,
        ZIndex = 70
    })
    corner(StatsFrame, 12); stroke(StatsFrame, Theme.Pink, 1, 0.25)

    FpsLabel = new("TextLabel", {
        Parent = StatsFrame,
        BackgroundTransparency = 1,
        Position = UDim2.new(0,10,0,6),
        Size = UDim2.new(1,-20,0,20),
        Text = "FPS: ...",
        TextColor3 = Theme.White,
        TextSize = 13,
        Font = Enum.Font.GothamBlack,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 71
    })

    PingLabel = new("TextLabel", {
        Parent = StatsFrame,
        BackgroundTransparency = 1,
        Position = UDim2.new(0,10,0,29),
        Size = UDim2.new(1,-20,0,20),
        Text = "PING: ...",
        TextColor3 = Theme.Muted,
        TextSize = 12,
        Font = Enum.Font.GothamBlack,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 71
    })

    SettingsButton = new("TextButton", {
        Parent = ScreenGui,
        AnchorPoint = Vector2.new(0.5, 0),
        Position = UDim2.new(0.5, 0, 0, 18),
        Size = UDim2.new(0, 110, 0, 52),
        BackgroundColor3 = Theme.Pink,
        BorderSizePixel = 0,
        Text = "CFG v1.3",
        TextColor3 = Theme.White,
        TextStrokeColor3 = Theme.Black,
        TextStrokeTransparency = 0.06,
        TextSize = 15,
        Font = Enum.Font.GothamBlack,
        AutoButtonColor = true,
        Active = true,
        ZIndex = 90
    })
    corner(SettingsButton, 26); stroke(SettingsButton, Theme.Black, 4, 0)

    Notice = new("TextLabel", {
        Parent = ScreenGui,
        AnchorPoint = Vector2.new(0.5, 0),
        Position = UDim2.new(0.5, 0, 0, -42),
        Size = UDim2.new(0, 250, 0, 34),
        BackgroundColor3 = Theme.Panel,
        BorderSizePixel = 0,
        Text = "Notice",
        TextColor3 = Theme.Green,
        TextSize = 13,
        Font = Enum.Font.GothamBlack,
        Visible = false,
        ZIndex = 130
    })
    corner(Notice, 12); stroke(Notice, Theme.Black, 3, 0)
end

---------------------------------------------------------------------
-- RENDER KEYBOARD
---------------------------------------------------------------------

local function makeText(parent, txt, pos, size, ts)
    return new("TextLabel", {
        Parent = parent,
        AnchorPoint = Vector2.new(0.5,0.5),
        Position = pos,
        Size = size,
        BackgroundTransparency = 1,
        Text = txt,
        TextColor3 = colorFromTable(Config.Colors.Text, Theme.White),
        TextStrokeColor3 = Theme.Black,
        TextStrokeTransparency = 0.05,
        TextSize = ts or 24,
        Font = Enum.Font.GothamBlack,
        ZIndex = 22
    })
end

local function renderKeyboard()
    disconnectList(RenderConnections)
    if not ButtonLayer then return end
    for _, child in ipairs(ButtonLayer:GetChildren()) do child:Destroy() end
    resetDPad()

    local outline = colorFromTable(Config.Colors.Outline, Theme.Black)
    local dpadColor = colorFromTable(Config.Colors.DPad, Theme.Pink2)
    local stickColor = colorFromTable(Config.Colors.Stick, Theme.Pink)

    local dPos = scaleUDim2(tableToUDim2(Config.DPadPosition))
    local dSize = scaleUDim2(tableToUDim2(Config.DPadSize))

    DPadFrame = new("Frame", {
        Parent = ButtonLayer,
        Position = dPos,
        Size = dSize,
        BackgroundColor3 = dpadColor,
        BackgroundTransparency = Config.DPadTransparency,
        BorderSizePixel = 0,
        Active = true,
        ZIndex = 20
    })
    corner(DPadFrame, 999); stroke(DPadFrame, outline, 4, 0)

    makeText(DPadFrame, "W", UDim2.new(0.5,0,0.12,0), UDim2.new(0,60,0,28), 28)
    makeText(DPadFrame, "A", UDim2.new(0.14,0,0.5,0), UDim2.new(0,60,0,28), 34)
    makeText(DPadFrame, "D", UDim2.new(0.86,0,0.5,0), UDim2.new(0,60,0,28), 34)
    makeText(DPadFrame, "S", UDim2.new(0.5,0,0.86,0), UDim2.new(0,60,0,28), 34)

    StickFrame = new("Frame", {
        Parent = DPadFrame,
        AnchorPoint = Vector2.new(0.5, 0.5),
        Position = UDim2.new(0.5, 0, 0.5, 0),
        Size = UDim2.new(0, math.floor(58 * Config.Scale), 0, math.floor(58 * Config.Scale)),
        BackgroundColor3 = stickColor,
        BackgroundTransparency = 0.08,
        BorderSizePixel = 0,
        ZIndex = 23
    })
    corner(StickFrame, 999); stroke(StickFrame, outline, 2, 0)

    add(RenderConnections, DPadFrame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            SelectedId = "DPAD"
            if EditMode then
                local dragStart = input.Position
                local startPos = DPadFrame.Position
                local dragging = true
                input.Changed:Connect(function()
                    if input.UserInputState == Enum.UserInputState.End then
                        dragging = false
                        Config.DPadPosition = udim2ToTable(UDim2.new(DPadFrame.Position.X.Scale, math.floor(DPadFrame.Position.X.Offset / Config.Scale), DPadFrame.Position.Y.Scale, math.floor(DPadFrame.Position.Y.Offset / Config.Scale)))
                        saveConfig(false)
                    end
                end)
                local conn
                conn = UserInputService.InputChanged:Connect(function(changed)
                    if dragging and changed == input then
                        local delta = changed.Position - dragStart
                        DPadFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
                    end
                    if not dragging then pcall(function() conn:Disconnect() end) end
                end)
            else
                DPadActiveInput = input
                updateDPad(input)
            end
        end
    end))

    add(RenderConnections, DPadFrame.InputChanged:Connect(function(input)
        if not EditMode and input == DPadActiveInput then updateDPad(input) end
    end))

    add(RenderConnections, DPadFrame.InputEnded:Connect(function(input)
        if input == DPadActiveInput then
            DPadActiveInput = nil
            resetDPad()
        end
    end))

    local btnColor = colorFromTable(Config.Colors.Button, Theme.Pink)
    local textColor = colorFromTable(Config.Colors.Text, Theme.White)

    for _, btn in ipairs(Config.Buttons) do
        local pos = scaleUDim2(tableToUDim2(btn.Position))
        local size = scaleUDim2(tableToUDim2(btn.Size))
        local b = new("TextButton", {
            Parent = ButtonLayer,
            Position = pos,
            Size = size,
            BackgroundColor3 = btnColor,
            BackgroundTransparency = Config.ButtonTransparency,
            BorderSizePixel = 0,
            Text = btn.Title,
            TextColor3 = textColor,
            TextStrokeColor3 = Theme.Black,
            TextStrokeTransparency = 0.04,
            TextSize = math.clamp(math.floor(size.Y.Offset * 0.42), 13, 34),
            Font = Enum.Font.GothamBlack,
            AutoButtonColor = false,
            Active = true,
            ZIndex = 20
        })
        corner(b, btn.Id == "SHIFT" or btn.Id == "SPACE" and 999 or 12)
        stroke(b, outline, 3, 0)

        local dragging = false
        local dragInput = nil
        local dragStart = nil
        local startPos = nil

        add(RenderConnections, b.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
                SelectedId = btn.Id
                if EditMode then
                    dragging = true
                    dragInput = input
                    dragStart = input.Position
                    startPos = b.Position
                    input.Changed:Connect(function()
                        if input.UserInputState == Enum.UserInputState.End then
                            dragging = false
                            btn.Position = udim2ToTable(UDim2.new(b.Position.X.Scale, math.floor(b.Position.X.Offset / Config.Scale), b.Position.Y.Scale, math.floor(b.Position.Y.Offset / Config.Scale)))
                            saveConfig(false)
                        end
                    end)
                else
                    local ok = pressButton(btn, input)
                    if not ok then notify("Input unavailable", true) end
                    if btn.Mode ~= "Toggle" then b.BackgroundColor3 = Theme.Pink2 end
                    if btn.Mode == "Toggle" then b.BackgroundColor3 = ToggleStates[btn.Id] and Theme.Green or btnColor end
                end
            end
        end))

        add(RenderConnections, b.InputChanged:Connect(function(input)
            if dragging and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
                dragInput = input
            end
        end))

        add(RenderConnections, b.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
                if EditMode then
                    dragging = false
                    saveConfig(false)
                else
                    releaseButton(btn, input)
                    if btn.Mode ~= "Toggle" then b.BackgroundColor3 = btnColor end
                end
            end
        end))

        add(RenderConnections, UserInputService.InputChanged:Connect(function(input)
            if dragging and input == dragInput and EditMode then
                local delta = input.Position - dragStart
                b.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
            end
        end))
    end
end

---------------------------------------------------------------------
-- SETTINGS
---------------------------------------------------------------------

local function makeSmall(parent, text, pos, size, callback, color)
    local b = new("TextButton", {
        Parent = parent,
        Position = pos,
        Size = size,
        BackgroundColor3 = color or Theme.Panel2,
        BorderSizePixel = 0,
        Text = text,
        TextColor3 = Theme.White,
        TextStrokeColor3 = Theme.Black,
        TextStrokeTransparency = 0.16,
        TextSize = 11,
        Font = Enum.Font.GothamBlack,
        AutoButtonColor = true,
        ZIndex = 112
    })
    corner(b, 9); stroke(b, Theme.Pink, 1, 0.55)
    add(Connections, b.MouseButton1Click:Connect(function()
        if callback then callback() end
    end))
    return b
end

local function refreshStatus()
    if StatusLabel then
        StatusLabel.Text = "Selected: " .. tostring(SelectedId) .. " | Scale: " .. tostring(math.floor(Config.Scale * 100 + 0.5)) .. "%"
    end
end

local function createSettings()
    if SettingsPanel then SettingsPanel:Destroy() end

    SettingsPanel = new("Frame", {
        Parent = ScreenGui,
        AnchorPoint = Vector2.new(0.5, 0.5),
        Position = UDim2.new(0.5, 0, 0.5, 0),
        Size = UDim2.new(0, 380, 0, 430),
        BackgroundColor3 = Theme.Panel,
        BackgroundTransparency = 0.02,
        BorderSizePixel = 0,
        Visible = SettingsOpen,
        Active = true,
        ZIndex = 110
    })
    corner(SettingsPanel, 16); stroke(SettingsPanel, Theme.Pink, 2, 0.2)

    new("TextLabel", {
        Parent = SettingsPanel,
        Position = UDim2.new(0, 16, 0, 10),
        Size = UDim2.new(1, -58, 0, 26),
        BackgroundTransparency = 1,
        Text = "Irenk Delta Keyboard v1.3 Lite",
        TextColor3 = Theme.White,
        TextSize = 16,
        Font = Enum.Font.GothamBlack,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 111
    })

    makeSmall(SettingsPanel, "X", UDim2.new(1, -44, 0, 10), UDim2.new(0, 30, 0, 28), function()
        SettingsOpen = false
        SettingsPanel.Visible = false
    end, Theme.Red)

    StatusLabel = new("TextLabel", {
        Parent = SettingsPanel,
        Position = UDim2.new(0, 16, 0, 42),
        Size = UDim2.new(1, -32, 0, 22),
        BackgroundTransparency = 1,
        Text = "Selected: DPAD",
        TextColor3 = Theme.Muted,
        TextSize = 11,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 111
    })

    makeSmall(SettingsPanel, "Edit: OFF", UDim2.new(0, 16, 0, 72), UDim2.new(0, 108, 0, 34), function()
        EditMode = not EditMode
        releaseAll()
        notify(EditMode and "Edit ON" or "Edit OFF", false)
        createSettings()
        renderKeyboard()
    end, EditMode and Theme.Green or Theme.Panel2).Text = EditMode and "Edit: ON" or "Edit: OFF"

    makeSmall(SettingsPanel, "+5 Scale", UDim2.new(0, 136, 0, 72), UDim2.new(0, 108, 0, 34), function()
        Config.Scale = clamp(Config.Scale + 0.05, 0.5, 2.5)
        renderKeyboard(); refreshStatus(); saveConfig(false)
    end)

    makeSmall(SettingsPanel, "-5 Scale", UDim2.new(0, 256, 0, 72), UDim2.new(0, 108, 0, 34), function()
        Config.Scale = clamp(Config.Scale - 0.05, 0.5, 2.5)
        renderKeyboard(); refreshStatus(); saveConfig(false)
    end)

    new("TextLabel", {Parent=SettingsPanel, Position=UDim2.new(0,16,0,118), Size=UDim2.new(1,-32,0,20), BackgroundTransparency=1, Text="Colors", TextColor3=Theme.White, TextSize=14, Font=Enum.Font.GothamBlack, TextXAlignment=Enum.TextXAlignment.Left, ZIndex=111})

    local palette = {
        {"Pink", Color3.fromRGB(255,88,105)},
        {"Blue", Color3.fromRGB(75,145,255)},
        {"Green", Color3.fromRGB(90,255,170)},
        {"Purple", Color3.fromRGB(165,90,255)},
        {"Yellow", Color3.fromRGB(255,225,110)},
        {"White", Color3.fromRGB(255,255,255)}
    }

    local x, y = 16, 146
    for _, p in ipairs(palette) do
        makeSmall(SettingsPanel, p[1], UDim2.new(0,x,0,y), UDim2.new(0,108,0,32), function()
            Config.Colors.Button = colorToTable(p[2])
            Config.Colors.DPad = colorToTable(p[2]:Lerp(Color3.fromRGB(255,255,255), 0.25))
            Config.Colors.Stick = colorToTable(p[2])
            renderKeyboard(); saveConfig(false)
        end, p[2])
        x = x + 120
        if x > 270 then x = 16; y = y + 38 end
    end

    new("TextLabel", {Parent=SettingsPanel, Position=UDim2.new(0,16,0,228), Size=UDim2.new(1,-32,0,20), BackgroundTransparency=1, Text="Transparency", TextColor3=Theme.White, TextSize=14, Font=Enum.Font.GothamBlack, TextXAlignment=Enum.TextXAlignment.Left, ZIndex=111})

    makeSmall(SettingsPanel, "Solid", UDim2.new(0,16,0,254), UDim2.new(0,80,0,32), function()
        Config.ButtonTransparency = 0
        Config.DPadTransparency = 0.08
        renderKeyboard(); saveConfig(false)
    end)
    makeSmall(SettingsPanel, "25%", UDim2.new(0,104,0,254), UDim2.new(0,80,0,32), function()
        Config.ButtonTransparency = 0.25
        Config.DPadTransparency = 0.25
        renderKeyboard(); saveConfig(false)
    end)
    makeSmall(SettingsPanel, "50%", UDim2.new(0,192,0,254), UDim2.new(0,80,0,32), function()
        Config.ButtonTransparency = 0.50
        Config.DPadTransparency = 0.50
        renderKeyboard(); saveConfig(false)
    end)
    makeSmall(SettingsPanel, "+ Clear", UDim2.new(0,280,0,254), UDim2.new(0,80,0,32), function()
        Config.ButtonTransparency = clamp(Config.ButtonTransparency + 0.10, 0, 0.80)
        Config.DPadTransparency = clamp(Config.DPadTransparency + 0.10, 0, 0.80)
        renderKeyboard(); saveConfig(false)
    end)

    new("TextLabel", {Parent=SettingsPanel, Position=UDim2.new(0,16,0,298), Size=UDim2.new(1,-32,0,20), BackgroundTransparency=1, Text="Mouse Offset: " .. MouseTarget, TextColor3=Theme.White, TextSize=14, Font=Enum.Font.GothamBlack, TextXAlignment=Enum.TextXAlignment.Left, ZIndex=111})

    makeSmall(SettingsPanel, "Global", UDim2.new(0,16,0,324), UDim2.new(0,80,0,30), function() MouseTarget="Global"; createSettings() end, MouseTarget=="Global" and Theme.Green or Theme.Panel2)
    makeSmall(SettingsPanel, "Left", UDim2.new(0,104,0,324), UDim2.new(0,80,0,30), function() MouseTarget="Left"; createSettings() end, MouseTarget=="Left" and Theme.Green or Theme.Panel2)
    makeSmall(SettingsPanel, "Right", UDim2.new(0,192,0,324), UDim2.new(0,80,0,30), function() MouseTarget="Right"; createSettings() end, MouseTarget=="Right" and Theme.Green or Theme.Panel2)

    local function off(dx, dy)
        Config.MouseOffsets[MouseTarget] = Config.MouseOffsets[MouseTarget] or {X=0,Y=0}
        Config.MouseOffsets[MouseTarget].X = (Config.MouseOffsets[MouseTarget].X or 0) + dx
        Config.MouseOffsets[MouseTarget].Y = (Config.MouseOffsets[MouseTarget].Y or 0) + dy
        saveConfig(false)
        notify(MouseTarget .. " X:" .. Config.MouseOffsets[MouseTarget].X .. " Y:" .. Config.MouseOffsets[MouseTarget].Y, false)
    end

    makeSmall(SettingsPanel, "X-", UDim2.new(0,16,0,360), UDim2.new(0,60,0,28), function() off(-10,0) end)
    makeSmall(SettingsPanel, "X+", UDim2.new(0,84,0,360), UDim2.new(0,60,0,28), function() off(10,0) end)
    makeSmall(SettingsPanel, "Y-", UDim2.new(0,152,0,360), UDim2.new(0,60,0,28), function() off(0,-10) end)
    makeSmall(SettingsPanel, "Y+", UDim2.new(0,220,0,360), UDim2.new(0,60,0,28), function() off(0,10) end)
    makeSmall(SettingsPanel, "Reset", UDim2.new(0,288,0,360), UDim2.new(0,72,0,28), function()
        Config.MouseOffsets[MouseTarget] = {X=0,Y=0}
        saveConfig(false)
        notify("Offset reset", false)
    end)

    makeSmall(SettingsPanel, "Save", UDim2.new(0,16,0,396), UDim2.new(0,80,0,30), function() saveConfig(true) end, Theme.Green)
    makeSmall(SettingsPanel, "Reset All", UDim2.new(0,104,0,396), UDim2.new(0,100,0,30), function()
        pcall(function() if delfile then delfile(Config.SaveFileName) end end)
        notify("Re-execute script after reset", false)
    end, Theme.Yellow)
    makeSmall(SettingsPanel, "Kill", UDim2.new(0,212,0,396), UDim2.new(0,80,0,30), function()
        Alive = false
        releaseAll()
        disconnectList(Connections)
        disconnectList(RenderConnections)
        if ScreenGui then ScreenGui:Destroy() end
    end, Theme.Red)

    refreshStatus()
end

---------------------------------------------------------------------
-- STATS / LOOPS
---------------------------------------------------------------------

local function startStats()
    task.spawn(function()
        local frames, elapsed, last = 0, 0, os.clock()
        while Alive do
            RunService.RenderStepped:Wait()
            local now = os.clock()
            elapsed = elapsed + (now - last)
            last = now
            frames = frames + 1
            if elapsed >= 0.5 then
                if FpsLabel then FpsLabel.Text = "FPS: " .. tostring(math.floor(frames / elapsed + 0.5)) end
                frames, elapsed = 0, 0
            end
        end
    end)

    task.spawn(function()
        while Alive do
            local result = "N/A"
            pcall(function()
                local net = StatsService:FindFirstChild("Network")
                local server = net and net:FindFirstChild("ServerStatsItem")
                local ping = server and server:FindFirstChild("Data Ping")
                if ping then result = ping:GetValueString() end
            end)
            if PingLabel then PingLabel.Text = "PING: " .. result end
            task.wait(1)
        end
    end)

    task.spawn(function()
        while Alive do
            task.wait(0.35)
            for _, btn in ipairs(Config.Buttons) do
                if ActiveInputs[btn.Id] and btn.Type == "Key" and btn.Mode == "Hold" then
                    sendButton(btn, true)
                end
                if ToggleStates[btn.Id] and btn.Type == "Key" then
                    sendButton(btn, true)
                end
            end
        end
    end)
end

---------------------------------------------------------------------
-- BOOT
---------------------------------------------------------------------

loadConfig()
createBaseGui()
createSettings()
SettingsPanel.Visible = false

local lastPress = 0
local function toggleSettings()
    local now = os.clock()
    if now - lastPress < 0.25 then return end
    lastPress = now
    SettingsOpen = not SettingsOpen
    if not SettingsPanel or not SettingsPanel.Parent then createSettings() end
    SettingsPanel.Visible = SettingsOpen
    if SettingsOpen then refreshStatus() end
end

add(Connections, SettingsButton.MouseButton1Click:Connect(toggleSettings))
add(Connections, SettingsButton.MouseButton1Down:Connect(toggleSettings))
add(Connections, SettingsButton.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch then toggleSettings() end
end))

pcall(function()
    add(Connections, UserInputService.WindowFocusReleased:Connect(function()
        releaseAll()
        resetDPad()
    end))
end)

renderKeyboard()
startStats()
notify("Irenk Keyboard v1.3 Lite loaded", false)
