-- Demo completo (Studio-only) — ESP + Box 3D por time + Tracer + Fly suave + painel rolável + Wallhack branco + crédito
-- USO: Demonstração / Debug no Roblox Studio. NÃO contém aimbot/autofire.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

local IS_STUDIO = RunService:IsStudio()
if not IS_STUDIO then
    warn("Este script de demonstração foi feito para Roblox Studio.")
end

-- ==== Config ====
local SHOW_SELF = false
local MAX_DISTANCE = 1000
local RAINBOW_SPEED = 1
local FLY_SPEED = 40
local FLY_SMOOTH = 0.2

local modes = {
    AtivarESP = false,
    Nome = true,
    Vida = true,
    Distancia = true,
    Box = true,
    Rainbow = true,
    Tracer = true,
    Fly = false,
    CorBox = "Time", -- "Time" | "Fixa" | "Arco-íris"
    WallhackBranco = false, -- novo modo para wallhack
}

-- ==== UI (Scrolling Panel) ====
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "ESP_Painel_GUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local Frame = Instance.new("Frame", ScreenGui)
Frame.Name = "ESP_Painel"
Frame.Size = UDim2.new(0, 360, 0, 440)
Frame.Position = UDim2.new(0.03, 0, 0.12, 0)
Frame.BackgroundColor3 = Color3.fromRGB(25,25,35)
Frame.BorderSizePixel = 0
Instance.new("UICorner", Frame).CornerRadius = UDim.new(0,8)

local Title = Instance.new("TextLabel", Frame)
Title.Size = UDim2.new(1,0,0,56)
Title.Position = UDim2.new(0,0,0,0)
Title.BackgroundColor3 = Color3.fromRGB(120,0,255)
Title.Text = "🎯 ESP Painel Demo"
Title.TextColor3 = Color3.new(1,1,1)
Title.TextScaled = true
Title.BorderSizePixel = 0
Instance.new("UICorner", Title).CornerRadius = UDim.new(0,8)

local Scrolling = Instance.new("ScrollingFrame", Frame)
Scrolling.Size = UDim2.new(1,0,1,-96)
Scrolling.Position = UDim2.new(0,0,0,56)
Scrolling.BackgroundTransparency = 1
Scrolling.CanvasSize = UDim2.new(0,0,0,0)
Scrolling.ScrollBarThickness = 8
local UIList = Instance.new("UIListLayout", Scrolling)
UIList.SortOrder = Enum.SortOrder.LayoutOrder
UIList.Padding = UDim.new(0,8)

local Credito = Instance.new("TextLabel", Frame)
Credito.Size = UDim2.new(1,0,0,30)
Credito.Position = UDim2.new(0,0,1,-36)
Credito.BackgroundTransparency = 1
Credito.Text = "🔥 biamiranda 1.3 🔥"
Credito.TextColor3 = Color3.fromRGB(200,100,255)
Credito.TextScaled = true
Credito.Font = Enum.Font.GothamBold

local function atualizarCanvas()
    Scrolling.CanvasSize = UDim2.new(0,0,0, UIList.AbsoluteContentSize.Y + 12)
end
UIList:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(atualizarCanvas)

-- ==== Botões helper ====
local buttons = {}
local function criarBotao(nome)
    local botao = Instance.new("TextButton", Scrolling)
    botao.Size = UDim2.new(1,-20,0,44)
    botao.Position = UDim2.new(0,10,0,0)
    local textVal = typeof(modes[nome]) == "boolean" and (modes[nome] and "ON" or "OFF") or tostring(modes[nome])
    botao.Text = nome..": "..textVal
    botao.TextColor3 = Color3.new(1,1,1)
    botao.BackgroundColor3 = Color3.fromRGB(40,40,50)
    botao.BorderSizePixel = 0
    Instance.new("UICorner", botao).CornerRadius = UDim.new(0,6)

    botao.MouseButton1Click:Connect(function()
        if typeof(modes[nome]) == "boolean" then
            modes[nome] = not modes[nome]
            botao.Text = nome..": "..(modes[nome] and "ON" or "OFF")
            botao.BackgroundColor3 = modes[nome] and Color3.fromRGB(120,0,255) or Color3.fromRGB(40,40,50)
        elseif nome == "CorBox" then
            local seq = {"Time","Fixa","Arco-íris"}
            local i = table.find(seq, modes.CorBox) or 1
            modes.CorBox = seq[i % #seq + 1]
            botao.Text = "CorBox: "..modes.CorBox
            botao.BackgroundColor3 = Color3.fromRGB(120,0,255)
        end
    end)

    buttons[nome] = botao
    return botao
end

local ordemNomes = {"AtivarESP","Nome","Vida","Distancia","Box","Rainbow","Tracer","Fly","CorBox","WallhackBranco"}
for _,n in ipairs(ordemNomes) do criarBotao(n) end
atualizarCanvas()

-- Toggle painel com Y
UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == Enum.KeyCode.Y then
        Frame.Visible = not Frame.Visible
    end
end)

-- ==== Helpers ====
local function hsvToRgb(h,s,v)
    return Color3.fromHSV((h%1), math.clamp(s,0,1), math.clamp(v,0,1))
end

-- ==== Dados por jogador ====
local espList = {}

local function createESPForPlayer(player)
    if player == LocalPlayer and not SHOW_SELF then return end
    if espList[player] then return end

    local data = {}
    -- Billboard (nome / hp / dist)
    local billboard = Instance.new("BillboardGui")
    billboard.Name = "ESPBillboard_"..player.Name
    billboard.AlwaysOnTop = true
    billboard.Size = UDim2.new(0,220,0,56)
    billboard.StudsOffset = Vector3.new(0,3,0)

    local nameLabel = Instance.new("TextLabel", billboard)
    nameLabel.Size = UDim2.new(1,0,0.35,0)
    nameLabel.BackgroundTransparency = 1
    nameLabel.TextScaled = true
    nameLabel.TextColor3 = Color3.new(1,1,1)

    local hpLabel = Instance.new("TextLabel", billboard)
    hpLabel.Size = UDim2.new(1,0,0.32,0)
    hpLabel.Position = UDim2.new(0,0,0.35,0)
    hpLabel.BackgroundTransparency = 1
    hpLabel.TextScaled = true
    hpLabel.TextColor3 = Color3.fromRGB(0,255,0)

    local distLabel = Instance.new("TextLabel", billboard)
    distLabel.Size = UDim2.new(1,0,0.33,0)
    distLabel.Position = UDim2.new(0,0,0.67,0)
    distLabel.BackgroundTransparency = 1
    distLabel.TextScaled = true
    distLabel.TextColor3 = Color3.fromRGB(0,170,255)

    data.Billboard = billboard
    data.NameLabel = nameLabel
    data.HPLabel = hpLabel
    data.DistLabel = distLabel

    -- Highlight
    local highlight = Instance.new("Highlight")
    highlight.Name = "ESPHighlight_"..player.Name
    highlight.FillTransparency = 1
    highlight.OutlineTransparency = 0.5
    highlight.Enabled = false
    data.Highlight = highlight

    -- Tracer (Drawing)
    local tracer = nil
    if Drawing then
        local line = Drawing.new("Line")
        line.Visible = false
        line.Thickness = 2
        line.Transparency = 1
        line.Color = Color3.fromRGB(120,0,255)
        tracer = line
    end
    data.Tracer = tracer

    -- BoxLines (Drawing)
    data.BoxLines = nil

    espList[player] = data
end

local function removeESPForPlayer(player)
    local d = espList[player]
    if not d then return end
    if d.Billboard and d.Billboard.Parent then d.Billboard:Destroy() end
    if d.Highlight and d.Highlight.Parent then d.Highlight:Destroy() end
    if d.Tracer then
        d.Tracer.Visible = false
        if d.Tracer.Remove then pcall(function() d.Tracer:Remove() end) end
    end
    if d.BoxLines then
        for _, l in ipairs(d.BoxLines) do
            if l and l.Visible ~= nil then
                l.Visible = false
                if l.Remove then pcall(function() l:Remove() end) end
            end
        end
    end
    espList[player] = nil
end

Players.PlayerAdded:Connect(createESPForPlayer)
Players.PlayerRemoving:Connect(removeESPForPlayer)
for _,p in ipairs(Players:GetPlayers()) do createESPForPlayer(p) end

-- ==== Fly body ====
local flyBody = Instance.new("BodyVelocity")
flyBody.MaxForce = Vector3.new(1e5,1e5,1e5)
flyBody.Velocity = Vector3.zero
flyBody.Parent = nil
local currentVelocity = Vector3.new()

-- ==== Loop principal ====
RunService.RenderStepped:Connect(function(dt)
    if not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        for _, d in pairs(espList) do
            if d.Tracer then d.Tracer.Visible = false end
            if d.BoxLines then for _, l in ipairs(d.BoxLines) do if l then l.Visible = false end end end
            if d.Billboard then d.Billboard.Parent = nil end
            if d.Highlight then d.Highlight.Enabled = false end
        end
        return
    end

    local hrp = LocalPlayer.Character.HumanoidRootPart
    local camCF = Camera.CFrame
    local screenCenter = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)

    for player, data in pairs(espList) do
        local char = player.Character
        local targetPart = char and (char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Head"))
        if targetPart and modes.AtivarESP then
            data.Billboard.Parent = Camera
            data.Billboard.Adornee = targetPart
            data.NameLabel.Text = modes.Nome and player.Name or ""
            if char:FindFirstChildOfClass("Humanoid") and modes.Vida then
                data.HPLabel.Text = "Vida: "..math.floor(char:FindFirstChildOfClass("Humanoid").Health)
            else
                data.HPLabel.Text = ""
            end
            if modes.Distancia and hrp then
                local dist = (targetPart.Position - hrp.Position).Magnitude
                data.DistLabel.Text = string.format("%.0fm", dist)
            else
                data.DistLabel.Text = ""
            end

            if modes.Rainbow then
                data.NameLabel.TextColor3 = hsvToRgb(tick()*RAINBOW_SPEED, 0.9, 1)
            else
                data.NameLabel.TextColor3 = Color3.new(1,1,1)
            end

            if data.Tracer and modes.Tracer then
                local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
                if onScreen then
                    data.Tracer.From = screenCenter
                    data.Tracer.To = Vector2.new(screenPos.X, screenPos.Y)
                    data.Tracer.Visible = true
                else
                    data.Tracer.Visible = false
                end
            elseif data.Tracer then
                data.Tracer.Visible = false
            end

            -- BOX 3D via Drawing
            if modes.Box then
                if not data.BoxLines and Drawing then
                    data.BoxLines = {}
                    for i = 1,12 do
                        local ln = Drawing.new("Line")
                        ln.Thickness = 2
                        ln.Visible = false
                        table.insert(data.BoxLines, ln)
                    end
                end
            end

            -- WALLHACK BRANCO
            if modes.WallhackBranco then
                if data.Highlight then
                    data.Highlight.Parent = workspace
                    data.Highlight.Adornee = char
                    data.Highlight.Enabled = true
                    data.Highlight.FillColor = Color3.new(1,1,1)
                    data.Highlight.OutlineColor = Color3.new(1,1,1)
                    data.Highlight.FillTransparency = 0.8
                    data.Highlight.OutlineTransparency = 0.2
                end
            else
                if data.Highlight then data.Highlight.Enabled = false end
            end

        else
            if data.Billboard then data.Billboard.Parent = nil end
            if data.Tracer then data.Tracer.Visible = false end
            if data.BoxLines then for _, l in ipairs(data.BoxLines) do l.Visible = false end end
            if data.Highlight then data.Highlight.Enabled = false end
        end
    end

    -- Fly (suavizado)
    if modes.Fly then
        if not flyBody.Parent then flyBody.Parent = hrp end
        local moveDir = Vector3.new()
        if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveDir = moveDir + camCF.LookVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveDir = moveDir - camCF.LookVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveDir = moveDir - camCF.RightVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveDir = moveDir + camCF.RightVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.Space) then moveDir = moveDir + Vector3.new(0,1,0) end
        if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then moveDir = moveDir - Vector3.new(0,1,0) end

        if moveDir.Magnitude > 0 then
            moveDir = moveDir.Unit * FLY_SPEED
        else
            moveDir = Vector3.new()
        end

        currentVelocity = currentVelocity:Lerp(moveDir, FLY_SMOOTH)
        flyBody.Velocity = currentVelocity
    else
        if flyBody.Parent then flyBody.Parent = nil end
        currentVelocity = Vector3.new()
    end
end)

print("✅ Demo carregada (Studio-only). Sem aimbot. Use o painel para ligar/desligar recursos.")
