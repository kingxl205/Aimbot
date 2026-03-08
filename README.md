--[[
    ============================================================
    SISTEMA DE MIRA (CROSSHAIR) - FPS Roblox
    ============================================================
    Autor: CrosshairSystem
    Descrição: Crosshair dinâmica para jogos de tiro em primeira
               pessoa. Expande ao mover e ao atirar (recoil).
    Local: StarterPlayerScripts > LocalScript
    ============================================================
--]]

-- ============================================================
-- SERVIÇOS
-- ============================================================
local Players           = game:GetService("Players")
local UserInputService  = game:GetService("UserInputService")
local RunService        = game:GetService("RunService")
local TweenService      = game:GetService("TweenService")

-- ============================================================
-- REFERÊNCIAS DO JOGADOR
-- ============================================================
local LocalPlayer   = Players.LocalPlayer
local PlayerGui     = LocalPlayer:WaitForChild("PlayerGui")
local Character     = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
local Humanoid      = Character:WaitForChild("Humanoid")
local RootPart      = Character:WaitForChild("HumanoidRootPart")

-- ============================================================
-- CONFIGURAÇÕES DA CROSSHAIR
-- ============================================================
local CONFIG = {
    -- Tamanho base da crosshair (em pixels, distância do centro)
    BaseSize        = 14,

    -- Tamanho máximo ao mover
    MoveSize        = 28,

    -- Tamanho máximo ao atirar (recoil)
    RecoilSize      = 48,

    -- Espessura das linhas da crosshair
    Thickness       = 2,

    -- Comprimento de cada linha
    LineLength      = 10,

    -- Gap central (espaço vazio no centro)
    CenterGap       = 4,

    -- Cor padrão da crosshair
    Color           = Color3.fromRGB(255, 255, 255),

    -- Cor ao atirar (recoil)
    RecoilColor     = Color3.fromRGB(255, 80, 80),

    -- Transparência (0 = opaco, 1 = invisível)
    Transparency    = 0,

    -- Velocidade de retorno ao tamanho base (segundos)
    ShrinkSpeed     = 0.12,

    -- Velocidade de expansão ao atirar (segundos)
    RecoilSpeed     = 0.04,

    -- Velocidade de expansão ao mover (segundos)
    MoveExpandSpeed = 0.08,

    -- Ponto central do ponto (dot) no meio da mira
    ShowDot         = true,
    DotSize         = 3,
}

-- ============================================================
-- CRIAÇÃO DA SCREENGUI
-- ============================================================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name              = "CrosshairGui"
ScreenGui.ResetOnSpawn      = false   -- Mantém a GUI ao respawnar
ScreenGui.IgnoreGuiInset    = true    -- Ocupa toda a tela sem offset
ScreenGui.ZIndexBehavior    = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent            = PlayerGui

-- Frame container central
local Container = Instance.new("Frame")
Container.Name              = "Container"
Container.Size              = UDim2.new(0, 0, 0, 0)
Container.Position          = UDim2.new(0.5, 0, 0.5, 0)
Container.AnchorPoint       = Vector2.new(0.5, 0.5)
Container.BackgroundTransparency = 1
Container.Parent            = ScreenGui

-- ============================================================
-- FUNÇÃO: Criar uma linha da crosshair
-- ============================================================
-- direction: "Horizontal" ou "Vertical"
-- side:      "Positive" (+X/+Y) ou "Negative" (-X/-Y)
local function CreateLine(direction, side)
    local frame = Instance.new("Frame")
    frame.BackgroundColor3      = CONFIG.Color
    frame.BackgroundTransparency = CONFIG.Transparency
    frame.BorderSizePixel       = 0

    if direction == "Horizontal" then
        frame.Size = UDim2.new(0, CONFIG.LineLength, 0, CONFIG.Thickness)
    else
        frame.Size = UDim2.new(0, CONFIG.Thickness, 0, CONFIG.LineLength)
    end

    frame.AnchorPoint = Vector2.new(0.5, 0.5)
    frame.Parent = Container
    return frame
end

-- ============================================================
-- CRIAÇÃO DAS 4 LINHAS + DOT CENTRAL
-- ============================================================
local Lines = {
    Right  = CreateLine("Horizontal", "Positive"),
    Left   = CreateLine("Horizontal", "Negative"),
    Down   = CreateLine("Vertical",   "Positive"),
    Up     = CreateLine("Vertical",   "Negative"),
}

-- Dot central opcional
local Dot
if CONFIG.ShowDot then
    Dot = Instance.new("Frame")
    Dot.Name                    = "Dot"
    Dot.Size                    = UDim2.new(0, CONFIG.DotSize, 0, CONFIG.DotSize)
    Dot.Position                = UDim2.new(0.5, 0, 0.5, 0)
    Dot.AnchorPoint             = Vector2.new(0.5, 0.5)
    Dot.BackgroundColor3        = CONFIG.Color
    Dot.BackgroundTransparency  = CONFIG.Transparency
    Dot.BorderSizePixel         = 0
    Dot.Parent                  = Container

    -- Bordas arredondadas no dot
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(1, 0)
    corner.Parent = Dot
end

-- ============================================================
-- ESTADO INTERNO DA CROSSHAIR
-- ============================================================
local State = {
    CurrentSize = CONFIG.BaseSize,  -- Tamanho atual (interpolado)
    TargetSize  = CONFIG.BaseSize,  -- Tamanho alvo
    IsRecoiling = false,            -- Se está em recoil
}

-- ============================================================
-- FUNÇÃO: Atualizar posição das linhas com base no tamanho atual
-- ============================================================
local function UpdateCrosshairVisuals(size, color)
    local gap = CONFIG.CenterGap

    -- Linha Direita
    Lines.Right.Position = UDim2.new(0.5, gap + CONFIG.LineLength / 2 + (size - CONFIG.BaseSize), 0.5, 0)
    Lines.Right.BackgroundColor3 = color or CONFIG.Color

    -- Linha Esquerda
    Lines.Left.Position  = UDim2.new(0.5, -(gap + CONFIG.LineLength / 2 + (size - CONFIG.BaseSize)), 0.5, 0)
    Lines.Left.BackgroundColor3 = color or CONFIG.Color

    -- Linha Baixo
    Lines.Down.Position  = UDim2.new(0.5, 0, 0.5, gap + CONFIG.LineLength / 2 + (size - CONFIG.BaseSize))
    Lines.Down.BackgroundColor3 = color or CONFIG.Color

    -- Linha Cima
    Lines.Up.Position    = UDim2.new(0.5, 0, 0.5, -(gap + CONFIG.LineLength / 2 + (size - CONFIG.BaseSize)))
    Lines.Up.BackgroundColor3 = color or CONFIG.Color

    -- Dot central
    if Dot then
        Dot.BackgroundColor3 = color or CONFIG.Color
    end
end

-- ============================================================
-- FUNÇÃO: Disparar evento de recoil (chame ao atirar)
-- ============================================================
local function TriggerRecoil()
    State.IsRecoiling = true
    State.TargetSize  = CONFIG.RecoilSize

    -- Retorna ao tamanho base após o recoil
    task.delay(CONFIG.RecoilSpeed + 0.05, function()
        State.IsRecoiling = false
        -- O loop RenderStepped irá encolher automaticamente
    end)
end

-- ============================================================
-- DETECÇÃO DE DISPARO — Botão esquerdo do mouse / toque
-- ============================================================
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    -- Ignora se a UI está processando o input (ex: chat aberto)
    if gameProcessed then return end

    if input.UserInputType == Enum.UserInputType.MouseButton1
    or input.UserInputType == Enum.UserInputType.Touch then
        TriggerRecoil()
    end
end)

-- ============================================================
-- LOOP PRINCIPAL — RenderStepped (roda a cada frame)
-- ============================================================
RunService.RenderStepped:Connect(function(deltaTime)

    -- ── 1. Calcular o tamanho alvo com base no movimento ──
    local velocity    = RootPart.Velocity
    local speed       = Vector3.new(velocity.X, 0, velocity.Z).Magnitude
    local isMoving    = speed > 1.5   -- limiar de velocidade

    if not State.IsRecoiling then
        if isMoving then
            State.TargetSize = CONFIG.MoveSize
        else
            State.TargetSize = CONFIG.BaseSize
        end
    end

    -- ── 2. Interpolar suavemente o tamanho atual ──
    local lerpSpeed
    if State.IsRecoiling then
        lerpSpeed = CONFIG.RecoilSpeed
    elseif State.CurrentSize < State.TargetSize then
        lerpSpeed = CONFIG.MoveExpandSpeed
    else
        lerpSpeed = CONFIG.ShrinkSpeed
    end

    -- Lerp: aproxima CurrentSize → TargetSize
    local t = math.min(1, deltaTime / lerpSpeed)
    State.CurrentSize = State.CurrentSize + (State.TargetSize - State.CurrentSize) * t

    -- ── 3. Definir cor da crosshair ──
    local color = State.IsRecoiling and CONFIG.RecoilColor or CONFIG.Color

    -- ── 4. Atualizar visuais ──
    UpdateCrosshairVisuals(State.CurrentSize, color)
end)

-- ============================================================
-- RECRIAR REFERÊNCIAS AO RESPAWNAR
-- ============================================================
LocalPlayer.CharacterAdded:Connect(function(newCharacter)
    Character  = newCharacter
    Humanoid   = newCharacter:WaitForChild("Humanoid")
    RootPart   = newCharacter:WaitForChild("HumanoidRootPart")
end)

-- ============================================================
-- API PÚBLICA — outros scripts podem usar via RemoteEvent ou
-- ModuleScript para disparar o recoil externamente:
--
--   Exemplo no servidor / outro LocalScript:
--   _G.CrosshairRecoil = TriggerRecoil  (apenas LocalScript)
-- ============================================================
_G.CrosshairRecoil = TriggerRecoil

print("[CrosshairSystem] Crosshair carregada com sucesso!")
