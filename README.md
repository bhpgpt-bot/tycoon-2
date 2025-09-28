-- Serviços
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

-- Estados PvP
local savedPosition = nil
local teleportKey = Enum.KeyCode.F
local triggerBotKey = Enum.KeyCode.G
local aimAssistKey = Enum.KeyCode.H
local triggerBotEnabled = false
local aimAssistEnabled = false
local autoHealEnabled = false
local maxDistance = 250
local fovLimit = 60
local aimPart = "HumanoidRootPart"
local healThreshold = 40
local healItemName = "SmallMedkit"

-- FOV Circle
local fovCircle = Drawing.new("Circle")
fovCircle.Visible = false
fovCircle.Transparency = 1
fovCircle.Thickness = 2
fovCircle.Color = Color3.new(1, 1, 1)
fovCircle.Filled = false
fovCircle.Radius = fovLimit
fovCircle.Position = Vector2.new(workspace.CurrentCamera.ViewportSize.X / 2, workspace.CurrentCamera.ViewportSize.Y / 2)

RunService.RenderStepped:Connect(function()
    fovCircle.Position = Vector2.new(workspace.CurrentCamera.ViewportSize.X / 2, workspace.CurrentCamera.ViewportSize.Y / 2)
    fovCircle.Radius = fovLimit
end)

-- Funções PvP
local function isInFOV(targetPosition)
    local camera = workspace.CurrentCamera
    local direction = (targetPosition - camera.CFrame.Position).Unit
    local look = camera.CFrame.LookVector.Unit
    local angle = math.deg(math.acos(look:Dot(direction)))
    return angle <= fovLimit
end

local function getClosestTarget()
    local closest, shortest = nil, maxDistance
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Team ~= LocalPlayer.Team then
            local char = player.Character
            if char and char:FindFirstChild("Humanoid") then
                local part = char:FindFirstChild(aimPart)
                if part and char.Humanoid.Health > 0 then
                    local dist = (LocalPlayer.Character.HumanoidRootPart.Position - part.Position).Magnitude
                    if dist < shortest and isInFOV(part.Position) then
                        shortest = dist
                        closest = part
                    end
                end
            end
        end
    end
    return closest
end

RunService.Heartbeat:Connect(function()
    if triggerBotEnabled then
        local target = getClosestTarget()
        local tool = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Tool")
        if target and tool then tool:Activate() end
    end
end)

RunService.RenderStepped:Connect(function()
    if aimAssistEnabled then
        local target = getClosestTarget()
        if target then
            workspace.CurrentCamera.CFrame = CFrame.new(workspace.CurrentCamera.CFrame.Position, target.Position)
        end
    end
end)

RunService.Heartbeat:Connect(function()
    if autoHealEnabled and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("Humanoid") then
        local humanoid = LocalPlayer.Character.Humanoid
        if humanoid.Health <= healThreshold then
            local backpack = LocalPlayer:FindFirstChild("Backpack")
            local tool = backpack and backpack:FindFirstChild(healItemName) or LocalPlayer.Character:FindFirstChild(healItemName)
            if tool and tool:IsA("Tool") then
                if tool.Parent == backpack then tool.Parent = LocalPlayer.Character task.wait(0.1) end
                pcall(function() tool:Activate() end)
            end
        end
    end
end)

UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.KeyCode == teleportKey and savedPosition then
        LocalPlayer.Character.HumanoidRootPart.CFrame = CFrame.new(savedPosition)
    elseif input.KeyCode == triggerBotKey then
        triggerBotEnabled = not triggerBotEnabled
    elseif input.KeyCode == aimAssistKey then
        aimAssistEnabled = not aimAssistEnabled
        fovCircle.Visible = aimAssistEnabled
    end
end)
-- Locais onde Tools podem estar
local locais = {
    workspace,
    ReplicatedStorage,
    game:GetService("Lighting"),
    game:GetService("StarterPack"),
    game:GetService("StarterGear")
}

-- Função para clonar Tools
local function clonarTool(tool)
    if tool:IsA("Tool") and not LocalPlayer.Backpack:FindFirstChild(tool.Name) then
        local clone = tool:Clone()
        clone.Parent = LocalPlayer.Backpack
        print("🧬 Clonado:", tool.Name)
    end
end

-- Scanner geral
local function escanearEClonar()
    for _, localBase in ipairs(locais) do
        for _, obj in pairs(localBase:GetDescendants()) do
            if obj:IsA("Tool") then
                clonarTool(obj)
            end
        end
    end
end

-- Tentar salvar arma atual no armário
local function tentarSalvarArma()
    local tool = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Tool")
    local saveEvent = ReplicatedStorage:FindFirstChild("SaveWeaponEvent")
    if tool and saveEvent then
        saveEvent:FireServer(tool)
        print("💾 Tentativa de salvar arma:", tool.Name)
    else
        print("⚠️ Evento SaveWeaponEvent não encontrado ou sem arma equipada")
    end
end

-- Teste de dano manual
local function dispararDanoManual()
    local target = nil
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Humanoid") then
            target = player
            break
        end
    end

    local damageEvent = ReplicatedStorage:FindFirstChild("DealDamage")
    if target and damageEvent then
        damageEvent:FireServer(target, 100)
        print("🔫 Dano manual enviado para:", target.Name)
    else
        print("⚠️ Evento DealDamage não encontrado ou sem alvo")
    end
end
-- Interface Rayfield
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
    Name = "R3TH Painel Completo",
    LoadingTitle = "Sistema Restaurado",
    LoadingSubtitle = "by Lost",
    ShowText = "Clonador + PvP",
    Theme = "Default",
    ToggleUIKeybind = "K",
})

-- Aba PvP
local PvPTab = Window:CreateTab("Main", 4483362458)

PvPTab:CreateButton({
    Name = "Salvar Local Atual",
    Callback = function()
        local hrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if hrp then savedPosition = hrp.Position end
    end,
})

PvPTab:CreateInput({
    Name = "Tecla para Teleporte",
    PlaceholderText = "Ex: F",
    RemoveTextAfterFocusLost = true,
    Callback = function(input)
        local key = Enum.KeyCode[input:upper()]
        if key then teleportKey = key end
    end,
})

PvPTab:CreateInput({
    Name = "Tecla para TriggerBot",
    PlaceholderText = "Ex: G",
    RemoveTextAfterFocusLost = true,
    Callback = function(input)
        local key = Enum.KeyCode[input:upper()]
        if key then triggerBotKey = key end
    end,
})

PvPTab:CreateInput({
    Name = "Tecla para Aim Assist",
    PlaceholderText = "Ex: H",
    RemoveTextAfterFocusLost = true,
    Callback = function(input)
        local key = Enum.KeyCode[input:upper()]
        if key then aimAssistKey = key end
    end,
})

PvPTab:CreateSlider({
    Name = "FOV (Campo de Visão)",
    Range = {10, 180},
    Increment = 5,
    Suffix = "°",
    CurrentValue = fovLimit,
    Callback = function(Value)
        fovLimit = Value
        fovCircle.Radius = Value
    end,
})

PvPTab:CreateSlider({
    Name = "Alcance Máximo",
    Range = {50, 500},
    Increment = 10,
    Suffix = " studs",
    CurrentValue = maxDistance,
    Callback = function(Value)
        maxDistance = Value
    end,
})

PvPTab:CreateDropdown({
    Name = "Parte para mirar",
    Options = {"Head", "HumanoidRootPart"},
    CurrentOption = "HumanoidRootPart",
    Callback = function(Option)
        aimPart = Option
    end,
})

PvPTab:CreateToggle({
    Name = "AutoHeal",
    CurrentValue = false,
    Flag = "AutoHealToggle",
    Callback = function(Value)
        autoHealEnabled = Value
    end,
})

PvPTab:CreateSlider({
    Name = "Vida mínima para curar",
    Range = {10, 100},
    Increment = 5,
    Suffix = "%",
    CurrentValue = healThreshold,
    Callback = function(Value)
        healThreshold = Value
    end,
})

PvPTab:CreateButton({
    Name = "Ativar ESP",
    Callback = function()
        for _, player in pairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Head") then
                if not player.Character.Head:FindFirstChild("ESPLabel") then
                    local esp = Instance.new("BillboardGui")
                    esp.Name = "ESPLabel"
                    esp.Size = UDim2.new(0, 100, 0, 40)
                    esp.Adornee = player.Character.Head
                    esp.AlwaysOnTop = true
                    esp.Parent = player.Character.Head

                    local label = Instance.new("TextLabel", esp)
                    label.Size = UDim2.new(1, 0, 1, 0)
                    label.Text = "🎯 " .. player.Name
                    label.TextColor3 = Color3.new(1, 0, 0)
                    label.BackgroundTransparency = 1
                    label.Font = Enum.Font.Code
                    label.TextScaled = true
                end
            end
        end
    end,
})

-- Aba Scanner
local ScannerTab = Window:CreateTab("Scanner", 4483362458)

ScannerTab:CreateButton({
    Name = "🔍 Clonar Todas as Armas e Kits",
    Callback = escanearEClonar,
})

ScannerTab:CreateButton({
    Name = "💾 Tentar Salvar Arma no Armário",
    Callback = tentarSalvarArma,
})

ScannerTab:CreateButton({
    Name = "🔫 Testar Dano Manual",
    Callback = dispararDanoManual,
})

ScannerTab:CreateParagraph({
    Title = "Como funciona",
    Content = "Este painel escaneia o mapa e clona qualquer Tool visível (armas, kits médicos, etc) direto pra sua mochila. Também tenta salvar a arma atual no armário e disparar dano manual, se o jogo permitir."
})

