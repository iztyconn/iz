local WindUI = loadstring(game:HttpGet("https://raw.githubusercontent.com/Footagesus/WindUI/main/dist/main.lua"))()

local AIMBOT_ENABLED = false
local FOV_ENABLED = true
local FOV_RADIUS = 120
local HITBOX_ENABLED = false
local HITBOX_SIZE = 10
local WEAPON_REQUIRED = true
local CHECK_WALL = true
local AIMBOT_SMOOTH = 0.35
local AUTO_FIRE = false

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local VirtualInput = game:GetService("VirtualInputManager")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

local isAiming = false
local hitboxConnection = nil
local aimbotConnection = nil
local fovCircle = nil

local function hasWeapon()
    local char = LocalPlayer.Character
    return char and char:FindFirstChildOfClass("Tool") ~= nil
end

local function isVisible(part)
    local origin = Camera.CFrame.Position
    local direction = (part.Position - origin).Unit
    local rayParams = RaycastParams.new()
    rayParams.FilterType = Enum.RaycastFilterType.Blacklist
    rayParams.FilterDescendantsInstances = {LocalPlayer.Character, Camera}
    local result = Workspace:Raycast(origin, direction * (origin - part.Position).Magnitude, rayParams)
    return not result or result.Instance:IsDescendantOf(part.Parent)
end

local function getBestTarget()
    if not FOV_ENABLED then return nil end
    local best, bestAngle = nil, FOV_RADIUS
    local center = Camera.ViewportSize / 2
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local head = player.Character:FindFirstChild("Head")
            if head then
                local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
                if onScreen then
                    local angle = (Vector2.new(screenPos.X, screenPos.Y) - center).Magnitude
                    if angle < bestAngle and (not CHECK_WALL or isVisible(head)) then
                        best, bestAngle = head, angle
                    end
                end
            end
        end
    end
    return best
end

local function applySmoothAimbot(target)
    if not target then return end
    if AIMBOT_SMOOTH > 0 then
        local currentCF = Camera.CFrame
        local targetCF = CFrame.new(currentCF.Position, target.Position)
        local newCF = currentCF:Lerp(targetCF, AIMBOT_SMOOTH)
        Camera.CFrame = newCF
    else
        Camera.CFrame = CFrame.new(Camera.CFrame.Position, target.Position)
    end
end

local function autoFire()
    if AUTO_FIRE then
        VirtualInput:SendMouseButtonEvent(Enum.UserInputType.MouseButton1, Enum.UserInputState.Begin, Camera, nil)
        task.wait(0.05)
        VirtualInput:SendMouseButtonEvent(Enum.UserInputType.MouseButton1, Enum.UserInputState.End, Camera, nil)
    end
end

local function startAimbotLoop()
    if aimbotConnection then aimbotConnection:Disconnect() end
    aimbotConnection = RunService.RenderStepped:Connect(function()
        if not AIMBOT_ENABLED then return end
        if WEAPON_REQUIRED and not (hasWeapon() or isAiming) then return end
        local target = getBestTarget()
        if target then
            applySmoothAimbot(target)
            autoFire()
        end
    end)
end

local function updateHitboxes()
    if not HITBOX_ENABLED then return end
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local head = player.Character:FindFirstChild("Head")
            if head then
                head.Size = Vector3.new(HITBOX_SIZE, HITBOX_SIZE, HITBOX_SIZE)
                head.Transparency = 0.65
                head.CanCollide = false
                head.Massless = true
                head.Color = Color3.fromRGB(255, 70, 110)
            end
        end
    end
end

local function resetHitboxes()
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local head = player.Character:FindFirstChild("Head")
            if head then
                head.Size = Vector3.new(2, 1, 1)
                head.Transparency = 1
                head.CanCollide = true
                head.Massless = false
            end
        end
    end
end

local function setHitboxEnabled(enabled)
    HITBOX_ENABLED = enabled
    if HITBOX_ENABLED then
        if not hitboxConnection then
            hitboxConnection = RunService.RenderStepped:Connect(updateHitboxes)
        end
        updateHitboxes()
    else
        if hitboxConnection then
            hitboxConnection:Disconnect()
            hitboxConnection = nil
        end
        resetHitboxes()
    end
end

local function setupFOVCircle()
    if not Drawing or not Drawing.new then return false end
    local success, circle = pcall(function() return Drawing.new("Circle") end)
    if success and circle then
        fovCircle = circle
        fovCircle.Visible = FOV_ENABLED
        fovCircle.Radius = FOV_RADIUS
        fovCircle.Thickness = 2
        fovCircle.Color = Color3.fromRGB(255, 100, 150)
        fovCircle.Filled = false
        fovCircle.NumSides = 64
        fovCircle.Transparency = 0.7
        return true
    end
    return false
end
setupFOVCircle()

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isAiming = true
    end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isAiming = false
    end
end)

RunService.RenderStepped:Connect(function()
    if fovCircle then
        fovCircle.Visible = FOV_ENABLED
        if FOV_ENABLED and Camera and Camera.ViewportSize then
            fovCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
            fovCircle.Radius = FOV_RADIUS
        end
    end
end)

startAimbotLoop()

local Window = WindUI:CreateWindow({
    Title = "Aura + (ego)",
    Author = "by Tycoon",
    Folder = "aimbot_hub",
    Icon = "",
    Size = UDim2.fromOffset(500, 450),
    MinimizeButton = true,
    CloseButton = true,
    Topbar = {
        Height = 44,
        ButtonsType = "Windows"
    }
})

Window:Tag({
    Title = "v2.0.0",
    Color = Color3.fromHex("#30ff6a"),
    Radius = 20,
})

local AimbotTab = Window:Tab({
    Title = "aimbot",
    Icon = "solar:target-bold",
    IconColor = Color3.fromRGB(255, 100, 150)
})

local AimbotSection = AimbotTab:Section({ Title = "Configurações do Aimbot" })

AimbotSection:Toggle({
    Flag = "aimbot_enabled",
    Title = "aimbot (legit)",
    Value = AIMBOT_ENABLED,
    Callback = function(state) AIMBOT_ENABLED = state end
})

AimbotSection:Toggle({
    Title = "anti-parede",
    Value = CHECK_WALL,
    Callback = function(state) CHECK_WALL = state end
})

AimbotSection:Toggle({
    Title = "grudar apenas (arma em moes/mira)",
    Value = WEAPON_REQUIRED,
    Callback = function(state) WEAPON_REQUIRED = state end
})

AimbotSection:Slider({
    Flag = "fov_radius",
    Title = "raio do FOV (pixels)",
    Step = 5,
    Value = { Min = 40, Max = 400, Default = FOV_RADIUS },
    Callback = function(value) FOV_RADIUS = value end
})

AimbotSection:Slider({
    Title = "suavidade (smooth)",
    Step = 0.05,
    Value = { Min = 0, Max = 1, Default = AIMBOT_SMOOTH },
    Callback = function(value) AIMBOT_SMOOTH = value end
})

AimbotSection:Toggle({
    Title = "auto-firing",
    Value = AUTO_FIRE,
    Callback = function(state) AUTO_FIRE = state end
})

local FOVSection = AimbotTab:Section({ Title = "Círculo FOV Visual" })
FOVSection:Toggle({
    Flag = "fov_enabled",
    Title = "mostrar circulo",
    Value = FOV_ENABLED,
    Callback = function(state) FOV_ENABLED = state end
})

local HitboxTab = Window:Tab({
    Title = "hitbox",
    Icon = "solar:box-bold",
    IconColor = Color3.fromRGB(255, 100, 150)
})

local HitboxSection = HitboxTab:Section({ Title = "Configurações da Hitbox" })
HitboxSection:Toggle({
    Flag = "hitbox_enabled",
    Title = "hitbox",
    Value = HITBOX_ENABLED,
    Callback = function(state) setHitboxEnabled(state) end
})

HitboxSection:Slider({
    Flag = "hitbox_size",
    Title = "tamanho da hitbox (studs)",
    Step = 1,
    Value = { Min = 5, Max = 20, Default = HITBOX_SIZE },
    Callback = function(value)
        HITBOX_SIZE = value
        if HITBOX_ENABLED then updateHitboxes() end
    end
})

local InfoTab = Window:Tab({
    Title = "Info",
    Icon = "solar:info-square-bold",
    IconColor = Color3.fromRGB(255, 100, 150)
})

local InfoSection = InfoTab:Section({ Title = "Informações" })
InfoSection:Label({ Title = "aimbot + hitbox" })
InfoSection:Label({ Title = "by Tycoon" })
InfoSection:Label({ Title = "" })
InfoSection:Label({ Title = "Funcionalidades:" })
InfoSection:Label({ Title = "• Aimbot ajustável (FOV, smooth)" })
InfoSection:Label({ Title = "• Verificação de parede (raycast)" })
InfoSection:Label({ Title = "• Hitbox expansível 5~20" })
InfoSection:Label({ Title = "• Funciona apenas com arma/mira" })
InfoSection:Label({ Title = "• FOV visual" })

InfoSection:Button({
    Title = "Testar Notificação",
    Justify = "Center",
    Callback = function()
        WindUI:Notify({
            Title = "Configurações atuais",
            Content = "Aimbot: " .. (AIMBOT_ENABLED and "ON" or "OFF") .. "\nHitbox: " .. (HITBOX_ENABLED and "ON" or "OFF"),
            Duration = 3
        })
    end
})

Window.ConfigManager:Config("default"):Load()

Players.PlayerAdded:Connect(function(p)
    p.CharacterAdded:Connect(function()
        task.wait(0.5)
        if HITBOX_ENABLED then updateHitboxes() end
    end)
end)
