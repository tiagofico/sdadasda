-- Aura Battles - Auto Fly Kill Script
-- Voa automaticamente até o jogador mais próximo e ataca

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local rootPart = character:WaitForChild("HumanoidRootPart")

-- Configurações principais
local config = {
    autoFlyKill = false,
    flySpeed = 80,
    attackRange = 15,
    attackSpeed = 0.1,
    useAbilities = true,
    targetClosest = true,
    godMode = true,
    antiKick = true
}

-- Variáveis de controle
local bodyVelocity
local bodyPosition
local currentTarget = nil
local lastAttack = 0
local lastAbility = 0
local kills = 0
local connections = {}

-- GUI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "AutoFlyKill"
screenGui.ResetOnSpawn = false
screenGui.Parent = player:WaitForChild("PlayerGui")

local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 400, 0, 300)
mainFrame.Position = UDim2.new(0.5, -200, 0, 50)
mainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
mainFrame.BorderSizePixel = 0
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.Parent = screenGui

-- Estilo
local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 12)
corner.Parent = mainFrame

local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(255, 0, 0)
stroke.Thickness = 3
stroke.Parent = mainFrame

-- Título
local titleLabel = Instance.new("TextLabel")
titleLabel.Size = UDim2.new(1, 0, 0, 50)
titleLabel.Position = UDim2.new(0, 0, 0, 0)
titleLabel.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
titleLabel.BorderSizePixel = 0
titleLabel.Text = "🚁 AUTO FLY KILL - AURA BATTLES 🚁"
titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
titleLabel.TextSize = 16
titleLabel.Font = Enum.Font.GothamBold
titleLabel.Parent = mainFrame

local titleCorner = Instance.new("UICorner")
titleCorner.CornerRadius = UDim.new(0, 12)
titleCorner.Parent = titleLabel

-- Botão principal
local mainButton = Instance.new("TextButton")
mainButton.Size = UDim2.new(0, 300, 0, 60)
mainButton.Position = UDim2.new(0.5, -150, 0, 70)
mainButton.BackgroundColor3 = Color3.fromRGB(0, 150, 0)
mainButton.BorderSizePixel = 0
mainButton.Text = "INICIAR AUTO FLY KILL"
mainButton.TextColor3 = Color3.fromRGB(255, 255, 255)
mainButton.TextSize = 18
mainButton.Font = Enum.Font.GothamBold
mainButton.Parent = mainFrame

local buttonCorner = Instance.new("UICorner")
buttonCorner.CornerRadius = UDim.new(0, 10)
buttonCorner.Parent = mainButton

-- Status
local statusLabel = Instance.new("TextLabel")
statusLabel.Size = UDim2.new(1, -20, 0, 80)
statusLabel.Position = UDim2.new(0, 10, 0, 150)
statusLabel.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
statusLabel.BorderSizePixel = 0
statusLabel.Text = "Status: Aguardando...\nAlvo: Nenhum\nKills: 0\nDistância: 0"
statusLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
statusLabel.TextSize = 12
statusLabel.Font = Enum.Font.Gotham
statusLabel.TextYAlignment = Enum.TextYAlignment.Top
statusLabel.TextXAlignment = Enum.TextXAlignment.Left
statusLabel.Parent = mainFrame

local statusCorner = Instance.new("UICorner")
statusCorner.CornerRadius = UDim.new(0, 8)
statusCorner.Parent = statusLabel

-- Configurações rápidas
local configFrame = Instance.new("Frame")
configFrame.Size = UDim2.new(1, -20, 0, 40)
configFrame.Position = UDim2.new(0, 10, 0, 240)
configFrame.BackgroundTransparency = 1
configFrame.Parent = mainFrame

local speedLabel = Instance.new("TextLabel")
speedLabel.Size = UDim2.new(0, 100, 1, 0)
speedLabel.Position = UDim2.new(0, 0, 0, 0)
speedLabel.BackgroundTransparency = 1
speedLabel.Text = "Velocidade: 80"
speedLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
speedLabel.TextSize = 10
speedLabel.Font = Enum.Font.Gotham
speedLabel.Parent = configFrame

local speedSlider = Instance.new("TextButton")
speedSlider.Size = UDim2.new(0, 200, 0, 20)
speedSlider.Position = UDim2.new(0, 110, 0, 0)
speedSlider.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
speedSlider.BorderSizePixel = 0
speedSlider.Text = ""
speedSlider.Parent = configFrame

local sliderCorner = Instance.new("UICorner")
sliderCorner.CornerRadius = UDim.new(0, 10)
sliderCorner.Parent = speedSlider

-- Botão fechar
local closeButton = Instance.new("TextButton")
closeButton.Size = UDim2.new(0, 30, 0, 30)
closeButton.Position = UDim2.new(1, -35, 0, 10)
closeButton.BackgroundColor3 = Color3.fromRGB(200, 0, 0)
closeButton.BorderSizePixel = 0
closeButton.Text = "X"
closeButton.TextColor3 = Color3.fromRGB(255, 255, 255)
closeButton.TextSize = 14
closeButton.Font = Enum.Font.GothamBold
closeButton.Parent = mainFrame

local closeCorner = Instance.new("UICorner")
closeCorner.CornerRadius = UDim.new(1, 0)
closeCorner.Parent = closeButton

-- Funções principais
local function getClosestPlayer()
    local closest = nil
    local shortestDistance = math.huge
    
    for _, otherPlayer in pairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character and otherPlayer.Character:FindFirstChild("HumanoidRootPart") and otherPlayer.Character:FindFirstChild("Humanoid") then
            if otherPlayer.Character.Humanoid.Health > 0 then
                local distance = (rootPart.Position - otherPlayer.Character.HumanoidRootPart.Position).Magnitude
                if distance < shortestDistance then
                    closest = otherPlayer
                    shortestDistance = distance
                end
            end
        end
    end
    
    return closest, shortestDistance
end

local function setupFly()
    -- Body Velocity para movimento suave
    bodyVelocity = Instance.new("BodyVelocity")
    bodyVelocity.MaxForce = Vector3.new(4000, 4000, 4000)
    bodyVelocity.Velocity = Vector3.new(0, 0, 0)
    bodyVelocity.Parent = rootPart
    
    -- Body Position para posicionamento preciso
    bodyPosition = Instance.new("BodyPosition")
    bodyPosition.MaxForce = Vector3.new(4000, 4000, 4000)
    bodyPosition.Position = rootPart.Position
    bodyPosition.D = 2000
    bodyPosition.P = 10000
    bodyPosition.Parent = rootPart
end

local function removeFly()
    if bodyVelocity then
        bodyVelocity:Destroy()
        bodyVelocity = nil
    end
    if bodyPosition then
        bodyPosition:Destroy()
        bodyPosition = nil
    end
end

local function flyToTarget(target)
    if not target or not target.Character or not target.Character:FindFirstChild("HumanoidRootPart") then
        return
    end
    
    local targetPos = target.Character.HumanoidRootPart.Position
    local direction = (targetPos - rootPart.Position).Unit
    local distance = (targetPos - rootPart.Position).Magnitude
    
    if distance > config.attackRange then
        -- Voar em direção ao alvo
        local flyPosition = targetPos + Vector3.new(0, 5, 0) -- Voar um pouco acima
        if bodyPosition then
            bodyPosition.Position = flyPosition
        end
        if bodyVelocity then
            bodyVelocity.Velocity = direction * config.flySpeed
        end
    else
        -- Parar quando estiver próximo
        if bodyVelocity then
            bodyVelocity.Velocity = Vector3.new(0, 0, 0)
        end
    end
end

local function attackTarget()
    local currentTime = tick()
    if currentTime - lastAttack < config.attackSpeed then return end
    
    -- Simular M1 (clique esquerdo)
    mouse1click()
    lastAttack = currentTime
    
    -- Usar habilidades
    if config.useAbilities and currentTime - lastAbility > 2 then
        -- Pressionar E (habilidade especial)
        keypress(0x45)
        wait(0.05)
        keyrelease(0x45)
        
        wait(0.1)
        
        -- Pressionar F (aura)
        keypress(0x46)
        wait(0.05)
        keyrelease(0x46)
        
        lastAbility = currentTime
    end
end

local function updateStatus()
    local target, distance = getClosestPlayer()
    local targetName = target and target.Name or "Nenhum"
    local distanceText = target and math.floor(distance) or 0
    
    statusLabel.Text = string.format(
        "Status: %s\nAlvo: %s\nKills: %d\nDistância: %d",
        config.autoFlyKill and "ATIVO" or "INATIVO",
        targetName,
        kills,
        distanceText
    )
end

-- Anti-kick
local function setupAntiKick()
    connections.antiKick = player.Idled:Connect(function()
        game:GetService("VirtualUser"):Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
        game:GetService("VirtualUser"):Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
    end)
end

-- God Mode
local function setupGodMode()
    connections.godMode = RunService.Heartbeat:Connect(function()
        if config.godMode and humanoid then
            humanoid.Health = humanoid.MaxHealth
        end
    end)
end

-- Loop principal
local function mainLoop()
    connections.mainLoop = RunService.Heartbeat:Connect(function()
        if not config.autoFlyKill then return end
        if not character or not rootPart or not humanoid then return end
        
        local target, distance = getClosestPlayer()
        
        if target then
            currentTarget = target
            
            -- Voar até o alvo
            flyToTarget(target)
            
            -- Atacar se estiver próximo
            if distance <= config.attackRange then
                attackTarget()
            end
        else
            currentTarget = nil
            -- Parar movimento se não houver alvo
            if bodyVelocity then
                bodyVelocity.Velocity = Vector3.new(0, 0, 0)
            end
        end
        
        updateStatus()
    end)
end

-- Event handlers
mainButton.MouseButton1Click:Connect(function()
    config.autoFlyKill = not config.autoFlyKill
    
    if config.autoFlyKill then
        mainButton.Text = "PARAR AUTO FLY KILL"
        mainButton.BackgroundColor3 = Color3.fromRGB(200, 0, 0)
        setupFly()
        mainLoop()
    else
        mainButton.Text = "INICIAR AUTO FLY KILL"
        mainButton.BackgroundColor3 = Color3.fromRGB(0, 150, 0)
        removeFly()
        if connections.mainLoop then
            connections.mainLoop:Disconnect()
        end
    end
end)

-- Slider de velocidade
local dragging = false
speedSlider.MouseButton1Down:Connect(function()
    dragging = true
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = false
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local mousePos = UserInputService:GetMouseLocation()
        local sliderPos = speedSlider.AbsolutePosition
        local sliderSize = speedSlider.AbsoluteSize
        
        local relativeX = math.clamp((mousePos.X - sliderPos.X) / sliderSize.X, 0, 1)
        config.flySpeed = math.floor(20 + (relativeX * 180)) -- 20 a 200
        speedLabel.Text = "Velocidade: " .. config.flySpeed
    end
end)

closeButton.MouseButton1Click:Connect(function()
    -- Limpar conexões
    for _, connection in pairs(connections) do
        if connection then
            connection:Disconnect()
        end
    end
    
    removeFly()
    screenGui:Destroy()
end)

-- Monitorar mortes para contar kills
local function onCharacterAdded(newCharacter)
    character = newCharacter
    humanoid = newCharacter:WaitForChild("Humanoid")
    rootPart = newCharacter:WaitForChild("HumanoidRootPart")
    
    humanoid.Died:Connect(function()
        if currentTarget then
            kills = kills + 1
        end
    end)
end

-- Reconectar quando o personagem respawnar
player.CharacterAdded:Connect(onCharacterAdded)

-- Setup inicial
if config.antiKick then
    setupAntiKick()
end

if config.godMode then
    setupGodMode()
end

-- Notificação
game.StarterGui:SetCore("SendNotification", {
    Title = "Auto Fly Kill";
    Text = "Script carregado! Clique no botão para iniciar.";
    Duration = 5;
})

print("Auto Fly Kill Script carregado!")
print("- Voa automaticamente até o jogador mais próximo")
print("- Ataca automaticamente quando próximo")
print("- Usa habilidades E e F automaticamente")
print("- God Mode e Anti-Kick inclusos")
print("- Ajuste a velocidade com o slider")
