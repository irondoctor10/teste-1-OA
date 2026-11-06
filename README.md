--[[
    OA V9.1 | PLANETA DOS GUARDIOES
    Revisao estrutural e estabilizacao
    Tipo: LocalScript / StarterPlayerScripts

    IMPORTANTE:
    Esta versao preserva o modo local da V10.0. O mapa e as interacoes
    criados por este LocalScript existem apenas para o cliente local.
    Para uma experiencia multiplayer compartilhada, a construcao e as
    interacoes precisam ser migradas para Scripts/RemoteEvents no servidor.
]]

--========================================================
-- SERVICOS E CONFIGURACAO
--========================================================
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")

local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")

local VERSION = "9.1"
local MAP_NAME = "OA_LIVING_V9_1"
local LEGACY_MAP_NAME = "OA_LIVING_V10"
local GUI_NAME = "OA_CONTROL_PANEL_V9_1"
local LEGACY_GUI_NAME = "OA_CONTROL_PANEL_V10"

local CENTER = Vector3.new(0, 6000, 0)
local GLOBE_RADIUS = 950

local BUILDING = false
local PART_COUNT = 0
local BUILDING_COUNT = 0
local Map = nil
local CELL_DOORS = {}
local DOOR_STATES = {}
local LOCKDOWN = false
local BUILD_YIELD_INTERVAL = 140

local C = {
    Green = Color3.fromRGB(35, 190, 105),
    Bright = Color3.fromRGB(90, 255, 155),
    DarkGreen = Color3.fromRGB(10, 55, 38),
    DeepGreen = Color3.fromRGB(5, 23, 20),
    Stone = Color3.fromRGB(85, 95, 89),
    LightStone = Color3.fromRGB(155, 165, 155),
    DarkStone = Color3.fromRGB(35, 43, 40),
    Metal = Color3.fromRGB(45, 60, 55),
    Silver = Color3.fromRGB(175, 190, 185),
    Gold = Color3.fromRGB(205, 170, 90),
    Glass = Color3.fromRGB(110, 205, 175),
    White = Color3.fromRGB(225, 240, 230),
    Floor = Color3.fromRGB(62, 75, 68),
    Road = Color3.fromRGB(25, 34, 31),
    Soil = Color3.fromRGB(45, 65, 46),
    Red = Color3.fromRGB(180, 45, 45),
}

local function folder(name, parent)
    local f = Instance.new("Folder")
    f.Name = name
    f.Parent = parent or Map
    return f
end

-- Todos os objetos sao ancorados. Por padrao, pecas decorativas nao
-- participam de consultas/toques; addPrompt reativa CanQuery no suporte.
local function part(name, size, cf, color, material, parent, transparency, collidable)
    local p = Instance.new("Part")
    p.Name = name
    p.Size = size
    p.CFrame = cf
    p.Anchored = true
    p.CanCollide = collidable == true
    p.CanTouch = false
    p.CanQuery = false
    p.CastShadow = true
    p.Color = color or C.Stone
    p.Material = material or Enum.Material.SmoothPlastic
    p.Transparency = transparency or 0
    p.TopSurface = Enum.SurfaceType.Smooth
    p.BottomSurface = Enum.SurfaceType.Smooth
    p.Parent = parent or Map

    PART_COUNT += 1
    if PART_COUNT % BUILD_YIELD_INTERVAL == 0 then
        task.wait()
    end
    return p
end

local function ball(name, diameter, cf, color, material, parent, transparency, collidable)
    local p = part(name, Vector3.new(diameter, diameter, diameter), cf,
        color, material, parent, transparency, collidable)
    p.Shape = Enum.PartType.Ball
    return p
end

local function cylinder(name, diameter, height, cf, color, material, parent, collidable)
    -- Cylinder usa o eixo X como comprimento; a rotacao coloca o eixo em Y.
    local p = part(name, Vector3.new(height, diameter, diameter),
        cf * CFrame.Angles(0, 0, math.rad(90)), color, material, parent, nil, collidable)
    p.Shape = Enum.PartType.Cylinder
    return p
end

local function neon(name, size, cf, color, parent)
    return part(name, size, cf, color or C.Bright, Enum.Material.Neon, parent, nil, false)
end

local function makeLight(parent, color, brightness, range)
    local light = Instance.new("PointLight")
    light.Color = color or C.Bright
    light.Brightness = brightness or 1
    light.Range = range or 25
    light.Shadows = false
    light.Parent = parent
    return light
end

local function beam(name, a, b, width, color, parent)
    local delta = b - a
    if delta.Magnitude < 0.01 then return nil end
    local mid = (a + b) / 2
    return part(name, Vector3.new(width, width, delta.Magnitude),
        CFrame.lookAt(mid, b), color or C.Silver, Enum.Material.Metal, parent, nil, false)
end

-- Anel parametrizado: nunca consulta CENTER global.
local function buildRing(name, origin, radius, thickness, yOffset, color, segments, parent, material)
    segments = math.max(12, math.floor(segments or 48))
    local pieces = {}
    local arcLength = (2 * math.pi * radius / segments) * 1.015

    for i = 1, segments do
        local angle = ((i - 1) / segments) * math.pi * 2
        local mid = angle + math.pi / segments
        local offset = Vector3.new(math.cos(mid) * radius, yOffset, math.sin(mid) * radius)
        local piece = part(name, Vector3.new(thickness, 1.2, arcLength),
            CFrame.new(origin + offset) * CFrame.Angles(0, -mid, 0),
            color, material or Enum.Material.Metal, parent, nil, false)
        pieces[#pieces + 1] = piece
    end
    return pieces
end

local function label(parent, text, position, width, height)
    local p = part("Identificacao", Vector3.new(width, height, 1),
        CFrame.new(position), C.DeepGreen, Enum.Material.SmoothPlastic, parent, 0, false)
    p.CanQuery = false

    local surface = Instance.new("SurfaceGui")
    surface.Face = Enum.NormalId.Front
    surface.AlwaysOnTop = true
    surface.SizingMode = Enum.SurfaceGuiSizingMode.PixelsPerStud
    surface.PixelsPerStud = 40
    surface.Parent = p

    local txt = Instance.new("TextLabel")
    txt.Size = UDim2.fromScale(1, 1)
    txt.BackgroundTransparency = 1
    txt.Text = text
    txt.TextColor3 = C.Bright
    txt.TextScaled = true
    txt.Font = Enum.Font.GothamBold
    txt.Parent = surface
    return p
end

local function addPrompt(parent, action, object, callback)
    parent.CanQuery = true
    local prompt = Instance.new("ProximityPrompt")
    prompt.ActionText = action
    prompt.ObjectText = object
    prompt.KeyboardKeyCode = Enum.KeyCode.E
    prompt.HoldDuration = 0.3
    prompt.MaxActivationDistance = 14
    prompt.RequiresLineOfSight = false
    prompt.Parent = parent
    prompt.Triggered:Connect(function(player)
        if player == Player and parent.Parent then
            local ok, err = pcall(callback)
            if not ok then warn("[OA V9.1] Falha em interacao:", err) end
        end
    end)
    return prompt
end

--========================================================
-- GLOBO, ATMOSFERA, MERIDIANOS E ORBITAS
--========================================================
local function buildGlobe()
    local f = folder("01_PLANETA_E_ATMOSFERA")

    local shell = ball("Globo_de_OA", GLOBE_RADIUS * 2, CFrame.new(CENTER),
        C.Glass, Enum.Material.Glass, f, 0.965, false)
    shell.CastShadow = false

    local atmosphere = ball("Atmosfera_externa", GLOBE_RADIUS * 2.025,
        CFrame.new(CENTER), C.Bright, Enum.Material.ForceField, f, 0.985, false)
    atmosphere.CastShadow = false

    for degree = -75, 75, 15 do
        local angle = math.rad(degree)
        local radius = GLOBE_RADIUS * math.cos(angle)
        local y = GLOBE_RADIUS * math.sin(angle)
        if radius > 0 then
            buildRing("Latitude_" .. degree, CENTER, radius, 1.4, y,
                C.Green, 40, f)
        end
    end

    for meridian = 0, 7 do
        local rotation = math.rad(meridian * 22.5)
        local previous = nil
        for step = 0, 40 do
            local angle = (step / 40) * math.pi * 2
            local pos = CENTER + Vector3.new(
                math.sin(angle) * GLOBE_RADIUS * math.cos(rotation),
                math.cos(angle) * GLOBE_RADIUS,
                math.sin(angle) * GLOBE_RADIUS * math.sin(rotation)
            )
            if previous then beam("Meridiano", previous, pos, 1.2, C.Bright, f) end
            previous = pos
        end
    end

    buildRing("Orbita_principal", CENTER, 1160, 5, 0, C.Green, 80, f)
    buildRing("Orbita_secundaria", CENTER, 1220, 2, 100, C.Silver, 80, f)
    buildRing("Orbita_dourada", CENTER, 1090, 3, -110, C.Gold, 80, f)

    for i = 1, 8 do
        local a = i * math.pi / 4
        local pos = CENTER + Vector3.new(math.cos(a) * 1260,
            math.sin(a * 2) * 100, math.sin(a) * 1260)
        ball("Satelite_" .. i, 22, CFrame.new(pos), C.Silver,
            Enum.Material.Metal, f, 0, false)
        local core = ball("Nucleo_satelite", 8, CFrame.new(pos),
            C.Bright, Enum.Material.Neon, f, 0, false)
        if i % 2 == 0 then makeLight(core, C.Bright, 0.8, 22) end
        beam("Braco_satelite", pos, CENTER + (pos - CENTER).Unit * 1160, 2, C.Silver, f)
    end

    for i = 1, 4 do
        local a = i * math.pi / 2
        local pos = CENTER + Vector3.new(math.cos(a) * 960, 0, math.sin(a) * 960)
        local anchor = ball("Ancora_atmosferica", 15, CFrame.new(pos),
            C.Bright, Enum.Material.Neon, f, 0, false)
        if i == 1 then makeLight(anchor, C.Bright, 1.5, 40) end
    end
end

--========================================================
-- CENTRO PLANETARIO
--========================================================
local function buildCentralHub()
    local f = folder("02_CENTRO_PLANETARIO")

    cylinder("Fundacao_principal", 1450, 32, CFrame.new(CENTER - Vector3.new(0, 36, 0)),
        C.DarkStone, Enum.Material.Metal, f, true)
    cylinder("Anel_estrutural", 1400, 8, CFrame.new(CENTER - Vector3.new(0, 17, 0)),
        C.Green, Enum.Material.Metal, f, true)
    cylinder("Piso_inferior", 1340, 10, CFrame.new(CENTER - Vector3.new(0, 8, 0)),
        C.Floor, Enum.Material.Concrete, f, true)
    cylinder("Piso_superior", 1180, 3, CFrame.new(CENTER),
        C.LightStone, Enum.Material.Marble, f, true)

    buildRing("Borda_externa", CENTER, 700, 7, 2, C.Bright, 80, f)
    buildRing("Borda_interna", CENTER, 570, 4, 3, C.Green, 80, f)
    buildRing("Circuito_central", CENTER, 170, 4, 5, C.Gold, 64, f)

    for i = 1, 12 do
        local a = i * math.pi / 6
        local x, z = math.cos(a), math.sin(a)
        part("Avenida_radial", Vector3.new(36, 1, 700),
            CFrame.new(CENTER + Vector3.new(x * 360, 2.5, z * 360))
                * CFrame.Angles(0, -a, 0),
            C.Road, Enum.Material.Asphalt, f, 0, false)

        for j = 1, 8 do
            local distance = j * 75
            neon("Marcacao", Vector3.new(2, 0.25, 13),
                CFrame.new(CENTER + Vector3.new(x * distance, 3.2, z * distance))
                    * CFrame.Angles(0, -a, 0), C.Gold, f)
        end
    end

    cylinder("Plataforma_do_nucleo", 230, 16,
        CFrame.new(CENTER + Vector3.new(0, 12, 0)),
        C.DarkGreen, Enum.Material.Metal, f, true)

    for i = 1, 5 do
        cylinder("Anel_de_energia", 210 - i * 25, 2,
            CFrame.new(CENTER + Vector3.new(0, 22 + i * 5, 0)),
            C.Green, Enum.Material.Neon, f, false)
    end

    local core = ball("Nucleo_central_de_OA", 65,
        CFrame.new(CENTER + Vector3.new(0, 75, 0)),
        C.Bright, Enum.Material.Neon, f, 0, false)
    makeLight(core, C.Bright, 4, 110)

    for i = 1, 8 do
        local a = i * math.pi / 4
        local pos = CENTER + Vector3.new(math.cos(a) * 115, 35, math.sin(a) * 115)
        part("Pilar_nuclear", Vector3.new(9, 55, 9), CFrame.new(pos),
            C.Silver, Enum.Material.Metal, f, 0, true)
    end

    label(f, "OA", CENTER + Vector3.new(0, 125, -125), 90, 28)

    for i = 1, 8 do
        local a = i * math.pi / 4
        local pos = CENTER + Vector3.new(math.cos(a) * 240, 0, math.sin(a) * 240)
        part("Coluna_da_praca", Vector3.new(7, 38, 7),
            CFrame.new(pos + Vector3.new(0, 20, 0)),
            C.DarkStone, Enum.Material.Metal, f, 0, true)
        local lamp = neon("Luz_da_praca", Vector3.new(9, 3, 9),
            CFrame.new(pos + Vector3.new(0, 41, 0)), C.Bright, f)
        if i % 2 == 0 then makeLight(lamp, C.Bright, 0.7, 24) end
    end
end

--========================================================
-- EDIFICIOS MODULARES: PAREDES, PISOS, ABERTURAS E TELHADO
--========================================================
local function buildBuilding(name, position, width, depth, height, style, parent)
    assert(width >= 24 and depth >= 24 and height >= 24, "Dimensoes invalidas para " .. name)
    local f = folder(name, parent)
    BUILDING_COUNT += 1

    local wallColor = style == "commercial" and C.Metal or C.Stone
    local windowColor = style == "citadel" and C.Gold or C.Bright
    local wallThickness = math.max(3, math.min(5, width * 0.055))
    local floorHeight = 20
    local floorCount = math.max(1, math.floor(height / floorHeight))
    local innerWidth = width - wallThickness * 2
    local innerDepth = depth - wallThickness * 2
    local entryWidth = math.max(12, width * 0.28)
    local frontZ = position.Z - depth / 2 + wallThickness / 2
    local wallY = position.Y + 5 + height / 2

    part("Fundacao", Vector3.new(width + 8, 5, depth + 8),
        CFrame.new(position + Vector3.new(0, 2.5, 0)),
        C.DarkStone, Enum.Material.Metal, f, 0, true)

    -- Lajes independentes: o volume interno permanece vazio.
    for level = 0, floorCount - 1 do
        local y = position.Y + 5 + level * floorHeight
        part("Piso_" .. (level + 1), Vector3.new(innerWidth, 1.2, innerDepth),
            CFrame.new(position.X, y, position.Z),
            level == 0 and C.Floor or C.DarkStone,
            Enum.Material.Concrete, f, 0, true)
    end

    -- Paredes laterais e traseira.
    part("Parede_esquerda", Vector3.new(wallThickness, height, depth),
        CFrame.new(position.X - width / 2 + wallThickness / 2, wallY, position.Z),
        wallColor, Enum.Material.Concrete, f, 0, true)
    part("Parede_direita", Vector3.new(wallThickness, height, depth),
        CFrame.new(position.X + width / 2 - wallThickness / 2, wallY, position.Z),
        wallColor, Enum.Material.Concrete, f, 0, true)
    part("Parede_traseira", Vector3.new(width, height, wallThickness),
        CFrame.new(position.X, wallY, position.Z + depth / 2 - wallThickness / 2),
        wallColor, Enum.Material.Concrete, f, 0, true)

    -- Fachada frontal dividida para deixar uma entrada real.
    local sideWidth = (width - entryWidth) / 2
    part("Fachada_frontal_esquerda", Vector3.new(sideWidth, height, wallThickness),
        CFrame.new(position.X - (entryWidth + sideWidth) / 2, wallY, frontZ),
        wallColor, Enum.Material.Concrete, f, 0, true)
    part("Fachada_frontal_direita", Vector3.new(sideWidth, height, wallThickness),
        CFrame.new(position.X + (entryWidth + sideWidth) / 2, wallY, frontZ),
        wallColor, Enum.Material.Concrete, f, 0, true)
    part("Verga_da_entrada", Vector3.new(entryWidth, 5, wallThickness),
        CFrame.new(position.X, position.Y + 5 + height - 2.5, frontZ),
        C.DarkStone, Enum.Material.Metal, f, 0, true)

    -- Janelas decorativas por pavimento, sem bloquear a circulacao.
    for level = 1, floorCount do
        local y = position.Y + 5 + math.min(height - 8, level * floorHeight - 8)
        local pane = part("Janela_frontal_" .. level,
            Vector3.new(math.max(8, width * 0.32), 5, 0.7),
            CFrame.new(position.X, y, position.Z - depth / 2 - 0.25),
            windowColor, Enum.Material.Glass, f, 0.42, false)
        pane.CastShadow = false

        part("Friso_" .. level, Vector3.new(width + 2, 1, depth + 2),
            CFrame.new(position.X, position.Y + 5 + level * (height / floorCount), position.Z),
            C.Silver, Enum.Material.Metal, f, 0, false)
    end

    part("Cobertura", Vector3.new(width + 5, 4, depth + 5),
        CFrame.new(position.X, position.Y + height + 7, position.Z),
        C.DarkStone, Enum.Material.Metal, f, 0, true)

    -- Escada compacta em um canto para conectar os pisos.
    if floorCount > 1 then
        local steps = math.floor(floorHeight / 2)
        for level = 0, floorCount - 2 do
            for step = 1, steps do
                local y = position.Y + 6 + level * floorHeight + step * 1.8
                local z = position.Z - innerDepth * 0.28 + step * 0.65
                part("Degrau_escada", Vector3.new(7, 1.5, 5),
                    CFrame.new(position.X + innerWidth * 0.32, y, z),
                    C.Silver, Enum.Material.Metal, f, 0, true)
            end
        end
    end

    -- Mobiliario inicial mantido em escala moderada e fora do vao da porta.
    part("Balcao", Vector3.new(width * 0.38, 4, 5),
        CFrame.new(position + Vector3.new(0, 8, depth * 0.18)),
        C.Metal, Enum.Material.Metal, f, 0, true)
    for i = 1, 3 do
        part("Assento", Vector3.new(8, 2, 4),
            CFrame.new(position + Vector3.new(-width * 0.25 + i * width * 0.15, 7, -depth * 0.2)),
            C.DarkGreen, Enum.Material.Metal, f, 0, true)
    end

    local interior = neon("Luz_interna", Vector3.new(width * 0.32, 0.5, 2),
        CFrame.new(position + Vector3.new(0, 5 + math.min(17, height - 5), 0)), C.Bright, f)
    makeLight(interior, C.Bright, 0.8, math.min(30, width * 0.45))

    local beacon = neon("Sinalizador", Vector3.new(4, 8, 4),
        CFrame.new(position + Vector3.new(0, height + 13, 0)), C.Bright, f)
    if BUILDING_COUNT % 3 == 0 then makeLight(beacon, C.Bright, 0.7, 22) end
    return f
end

--========================================================
-- CITADEL DOS GUARDIOES
--========================================================
local function buildCitadel()
    local f = folder("03_CITADEL_DOS_GUARDIOES")
    local pos = CENTER + Vector3.new(0, 0, -440)

    cylinder("Plataforma", 330, 16, CFrame.new(pos + Vector3.new(0, 9, 0)),
        C.DarkStone, Enum.Material.Metal, f, true)
    buildRing("Anel_dourado", pos, 155, 5, 18, C.Gold, 64, f)

    part("Parede_traseira", Vector3.new(240, 100, 12),
        CFrame.new(pos + Vector3.new(0, 65, 85)), C.Stone, Enum.Material.Concrete, f, 0, true)
    part("Parede_esquerda", Vector3.new(12, 100, 170),
        CFrame.new(pos + Vector3.new(-115, 65, 0)), C.DarkStone, Enum.Material.Concrete, f, 0, true)
    part("Parede_direita", Vector3.new(12, 100, 170),
        CFrame.new(pos + Vector3.new(115, 65, 0)), C.DarkStone, Enum.Material.Concrete, f, 0, true)

    for i = 1, 9 do
        local x = -100 + (i - 1) * 25
        part("Coluna", Vector3.new(9, 85, 9),
            CFrame.new(pos + Vector3.new(x, 55, -75)), C.LightStone, Enum.Material.Marble, f, 0, true)
        part("Capitel", Vector3.new(14, 5, 14),
            CFrame.new(pos + Vector3.new(x, 100, -75)), C.Gold, Enum.Material.Metal, f, 0, true)
    end

    part("Arco_superior", Vector3.new(260, 12, 18),
        CFrame.new(pos + Vector3.new(0, 115, -75)), C.Gold, Enum.Material.Metal, f, 0, true)
    part("Piso_da_sala", Vector3.new(150, 2, 115),
        CFrame.new(pos + Vector3.new(0, 20, 10)), C.Floor, Enum.Material.Marble, f, 0, true)
    cylinder("Altar", 65, 14, CFrame.new(pos + Vector3.new(0, 30, 15)),
        C.Metal, Enum.Material.Metal, f, true)

    local core = ball("Esfera_dos_guardioes", 32,
        CFrame.new(pos + Vector3.new(0, 80, 15)), C.Bright, Enum.Material.Neon, f, 0, false)
    makeLight(core, C.Bright, 3, 75)

    for i = 1, 4 do
        part("Console", Vector3.new(14, 7, 10),
            CFrame.new(pos + Vector3.new(-50 + i * 30, 25, 55)),
            C.Metal, Enum.Material.Metal, f, 0, true)
    end

    buildBuilding("Ala_administrativa", pos + Vector3.new(-185, 0, 30),
        65, 65, 100, "citadel", f)
    buildBuilding("Ala_treinamento", pos + Vector3.new(185, 0, 30),
        65, 65, 100, "citadel", f)
    label(f, "CITADEL", pos + Vector3.new(0, 130, -95), 110, 22)
end

--========================================================
-- ARQUIVOS CENTRAIS
--========================================================
local function buildArchives()
    local f = folder("04_ARQUIVOS_CENTRAIS")
    local pos = CENTER + Vector3.new(-370, 0, -180)
    local building = buildBuilding("Biblioteca", pos, 110, 100, 130, "citadel", f)

    for row = 1, 5 do
        for col = 1, 5 do
            local x = pos.X - 45 + col * 18
            local z = pos.Z - 35 + row * 13
            part("Estante", Vector3.new(3, 26, 9), CFrame.new(x, pos.Y + 20, z),
                C.DarkGreen, Enum.Material.Metal, building, 0, true)
            for shelf = 1, 3 do
                part("Prateleira", Vector3.new(4, 1, 10),
                    CFrame.new(x, pos.Y + 12 + shelf * 6, z),
                    C.Gold, Enum.Material.Metal, building, 0, true)
            end
        end
    end

    part("Mesa_de_consulta", Vector3.new(45, 3, 18),
        CFrame.new(pos + Vector3.new(0, 10, 0)), C.Metal, Enum.Material.Metal, building, 0, true)
    local dataCore = ball("Nucleo_de_dados", 18,
        CFrame.new(pos + Vector3.new(0, 45, 0)), C.Bright, Enum.Material.Neon, building, 0, false)
    makeLight(dataCore, C.Bright, 1.5, 35)
    addPrompt(dataCore, "Consultar", "Arquivo central", function()
        print("[OA V9.1] Arquivos centrais acessados.")
    end)
end

--========================================================
-- PRISAO, PORTAS E LOCKDOWN
--========================================================
local function toggleCellDoor(door, forceClose)
    local state = DOOR_STATES[door]
    if not state or not door or not door.Parent then return end
    if state.tween then state.tween:Cancel() end

    local shouldOpen = not state.opened
    if forceClose then shouldOpen = false end
    local target = shouldOpen and state.openCFrame or state.closedCFrame
    state.opened = shouldOpen

    state.tween = TweenService:Create(door,
        TweenInfo.new(0.8, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
        {CFrame = target})
    state.tween:Play()
end

local function setLockdown(enabled)
    LOCKDOWN = enabled
    if enabled then
        for _, door in ipairs(CELL_DOORS) do
            local state = DOOR_STATES[door]
            if state and state.opened then toggleCellDoor(door, true) end
        end
    end
    print("[OA V9.1] Lockdown:", enabled and "ATIVADO" or "DESATIVADO")
end

local function buildPrison()
    local f = folder("05_PRISAO_DE_CONTENCAO")
    local pos = CENTER + Vector3.new(365, 0, -180)

    part("Fundacao_prisional", Vector3.new(230, 6, 170),
        CFrame.new(pos + Vector3.new(0, 3, 0)), C.DarkStone, Enum.Material.Concrete, f, 0, true)
    part("Piso_interno", Vector3.new(218, 2, 158),
        CFrame.new(pos + Vector3.new(0, 7, 0)), C.Floor, Enum.Material.Concrete, f, 0, true)

    part("Parede_traseira", Vector3.new(230, 42, 7),
        CFrame.new(pos + Vector3.new(0, 28, 82)), C.Metal, Enum.Material.Metal, f, 0, true)
    part("Parede_esquerda", Vector3.new(7, 42, 170),
        CFrame.new(pos + Vector3.new(-115, 28, 0)), C.Metal, Enum.Material.Metal, f, 0, true)
    part("Parede_direita", Vector3.new(7, 42, 170),
        CFrame.new(pos + Vector3.new(115, 28, 0)), C.Metal, Enum.Material.Metal, f, 0, true)
    part("Parede_frontal_superior", Vector3.new(230, 42, 7),
        CFrame.new(pos + Vector3.new(0, 28, -82)), C.Metal, Enum.Material.Metal, f, 0, true)
    part("Corredor", Vector3.new(220, 1, 35),
        CFrame.new(pos + Vector3.new(0, 8, -18)), C.Road, Enum.Material.Asphalt, f, 0, true)

    for i = 1, 6 do
        local x = -85 + (i - 1) * 34
        local cell = folder("Cela_" .. i, f)
        part("Divisoria_esquerda", Vector3.new(2, 30, 48),
            CFrame.new(pos + Vector3.new(x - 15, 23, 32)), C.Silver, Enum.Material.Metal, cell, 0, true)
        part("Divisoria_direita", Vector3.new(2, 30, 48),
            CFrame.new(pos + Vector3.new(x + 15, 23, 32)), C.Silver, Enum.Material.Metal, cell, 0, true)
        part("Parede_fundo", Vector3.new(32, 30, 3),
            CFrame.new(pos + Vector3.new(x, 23, 56)), C.DarkStone, Enum.Material.Metal, cell, 0, true)

        for bar = -2, 2 do
            part("Barra_grade", Vector3.new(1.5, 30, 1.5),
                CFrame.new(pos + Vector3.new(x + bar * 5, 23, 8)),
                C.Silver, Enum.Material.Metal, cell, 0, true)
        end

        local door = part("Porta_cela_" .. i, Vector3.new(28, 28, 2),
            CFrame.new(pos + Vector3.new(x, 22, 8)),
            C.Metal, Enum.Material.Metal, cell, 0, true)
        DOOR_STATES[door] = {
            closedCFrame = door.CFrame,
            openCFrame = door.CFrame + Vector3.new(0, 32, 0),
            opened = false,
            tween = nil,
        }
        CELL_DOORS[#CELL_DOORS + 1] = door

        addPrompt(door, "Abrir / fechar", "Cela " .. i, function()
            if LOCKDOWN then
                print("[OA V9.1] Prisao em lockdown.")
                return
            end
            toggleCellDoor(door)
        end)

        part("Cama", Vector3.new(20, 3, 10),
            CFrame.new(pos + Vector3.new(x, 9, 40)),
            C.DarkGreen, Enum.Material.Metal, cell, 0, true)
        local cellLamp = neon("Iluminacao", Vector3.new(8, 1, 2),
            CFrame.new(pos + Vector3.new(x, 36, 30)), C.Bright, cell)
        if i == 1 or i == 4 then makeLight(cellLamp, C.Bright, 0.5, 18) end
        label(cell, "CELA " .. i, pos + Vector3.new(x, 41, 4), 20, 5)
    end

    local control = folder("Sala_de_controle", f)
    part("Piso_controle", Vector3.new(75, 2, 35),
        CFrame.new(pos + Vector3.new(0, 9, -58)), C.DarkGreen, Enum.Material.Metal, control, 0, true)
    part("Console_principal", Vector3.new(45, 8, 10),
        CFrame.new(pos + Vector3.new(0, 14, -65)), C.Metal, Enum.Material.Metal, control, 0, true)
    local terminal = neon("Terminal_de_seguranca", Vector3.new(24, 8, 2),
        CFrame.new(pos + Vector3.new(0, 20, -58)), C.Bright, control)
    makeLight(terminal, C.Bright, 1, 28)
    addPrompt(terminal, "Alternar lockdown", "Controle prisional", function()
        setLockdown(not LOCKDOWN)
    end)
    label(f, "OA | CONTENCAO", pos + Vector3.new(0, 52, -85), 90, 18)
end

--========================================================
-- JARDINS E PRACAS
--========================================================
local function buildGarden(position, radius, parent)
    local f = folder("Jardim", parent)
    cylinder("Canteiro", radius, 3, CFrame.new(position + Vector3.new(0, 2, 0)),
        C.Soil, Enum.Material.Ground, f, true)

    for i = 1, 12 do
        local a = i * math.pi / 6
        local x, z = math.cos(a) * radius * 0.3, math.sin(a) * radius * 0.3
        part("Tronco", Vector3.new(3, 14, 3),
            CFrame.new(position + Vector3.new(x, 10, z)),
            C.DarkStone, Enum.Material.Wood, f, 0, true)
        ball("Copa", 17, CFrame.new(position + Vector3.new(x, 20, z)),
            C.DarkGreen, Enum.Material.Grass, f, 0, false)
    end

    cylinder("Fonte", radius * 0.3, 5, CFrame.new(position + Vector3.new(0, 4, 0)),
        C.Silver, Enum.Material.Marble, f, true)
    local water = cylinder("Agua", radius * 0.23, 1,
        CFrame.new(position + Vector3.new(0, 7, 0)),
        C.Bright, Enum.Material.Neon, f, false)
    water.Transparency = 0.35
end

local function buildGardens()
    local f = folder("06_JARDINS")
    buildGarden(CENTER + Vector3.new(-270, 0, 280), 95, f)
    buildGarden(CENTER + Vector3.new(270, 0, 280), 95, f)
    buildGarden(CENTER + Vector3.new(0, 0, 420), 115, f)

    for i = 1, 12 do
        local a = i * math.pi / 6
        local pos = CENTER + Vector3.new(math.cos(a) * 330, 7, math.sin(a) * 330)
        part("Banco", Vector3.new(20, 3, 5),
            CFrame.new(pos) * CFrame.Angles(0, -a, 0),
            C.DarkGreen, Enum.Material.Wood, f, 0, true)
    end
end

--========================================================
-- DISTRITOS HABITACIONAIS, COMERCIAIS E TECNOLOGICOS
--========================================================
local function buildDistricts()
    local f = folder("07_DISTRITOS")
    local residential = folder("Residencial", f)
    local commercial = folder("Comercial", f)
    local technology = folder("Tecnologico", f)

    local homes = {
        {-470, 120, 48, 45, 95}, {-420, 210, 50, 48, 115},
        {-540, 250, 44, 42, 85}, {-320, 180, 55, 48, 105},
        {-490, 350, 50, 50, 125}, {-360, 350, 45, 45, 90},
    }
    for i, d in ipairs(homes) do
        buildBuilding("Residencia_" .. i, CENTER + Vector3.new(d[1], 0, d[2]),
            d[3], d[4], d[5], "residential", residential)
    end

    local shops = {
        {420, 110, 65, 55, 60}, {500, 190, 70, 55, 75},
        {380, 250, 60, 55, 65}, {520, 320, 75, 60, 85},
        {350, 350, 60, 50, 70},
    }
    for i, d in ipairs(shops) do
        buildBuilding("Comercio_" .. i, CENTER + Vector3.new(d[1], 0, d[2]),
            d[3], d[4], d[5], "commercial", commercial)
    end

    for i = 1, 4 do
        local x = -160 + i * 80
        buildBuilding("Complexo_tecnologico_" .. i,
            CENTER + Vector3.new(x, 0, 490), 60, 65, 100 + i * 10,
            "commercial", technology)
    end
end

--========================================================
-- TORRES E SKYLINE
--========================================================
local function buildTower(position, height, index, parent)
    local f = folder("Torre_" .. index, parent)
    cylinder("Base", 65, 14, CFrame.new(position + Vector3.new(0, 7, 0)),
        C.DarkStone, Enum.Material.Metal, f, true)
    part("Corpo", Vector3.new(30, height, 30),
        CFrame.new(position + Vector3.new(0, height / 2 + 15, 0)),
        C.Stone, Enum.Material.Concrete, f, 0, true)

    for i = 1, 4 do
        local a = i * math.pi / 2
        part("Coluna", Vector3.new(4, height + 12, 4),
            CFrame.new(position + Vector3.new(math.cos(a) * 18, height / 2 + 15, math.sin(a) * 18)),
            C.Silver, Enum.Material.Metal, f, 0, true)
    end

    for y = 30, height, 35 do
        buildRing("Anel_luminoso", position, 25, 2, y, C.Green, 24, f, Enum.Material.Neon)
    end

    local crown = ball("Nucleo_superior", 15,
        CFrame.new(position + Vector3.new(0, height + 28, 0)),
        C.Bright, Enum.Material.Neon, f, 0, false)
    if index % 3 == 0 then makeLight(crown, C.Bright, 1.2, 38) end
end

local function buildSkyline()
    local f = folder("08_SKYLINE")
    for i = 1, 12 do
        local a = i * math.pi / 6
        local pos = CENTER + Vector3.new(math.cos(a) * 570, 0, math.sin(a) * 570)
        buildTower(pos, 180 + (i % 4) * 65, i, f)
    end
    buildTower(CENTER, 330, 99, f)
end

--========================================================
-- TRANSPORTE E DETALHES URBANOS
--========================================================
local function buildTransport()
    local f = folder("09_TRANSPORTE")
    local pos = CENTER + Vector3.new(0, 0, 545)

    cylinder("Estacao", 120, 9, CFrame.new(pos + Vector3.new(0, 9, 0)),
        C.DarkStone, Enum.Material.Metal, f, true)
    cylinder("Plataforma", 90, 4, CFrame.new(pos + Vector3.new(0, 17, 0)),
        C.Silver, Enum.Material.Metal, f, true)

    local core = ball("Nucleo", 24, CFrame.new(pos + Vector3.new(0, 38, 0)),
        C.Bright, Enum.Material.Neon, f, 0, false)
    makeLight(core, C.Bright, 2, 55)
    addPrompt(core, "Acessar", "Estacao de transporte", function()
        print("[OA V9.1] Estacao de transporte acessada.")
    end)

    for i = 1, 4 do
        local a = i * math.pi / 2
        part("Pilar", Vector3.new(6, 35, 6),
            CFrame.new(pos + Vector3.new(math.cos(a) * 40, 20, math.sin(a) * 40)),
            C.Silver, Enum.Material.Metal, f, 0, true)
    end

    for i = 1, 4 do
        local a = i * math.pi / 2
        local p = CENTER + Vector3.new(math.cos(a) * 390, 18, math.sin(a) * 390)
        part("Passarela", Vector3.new(20, 3, 180),
            CFrame.new(p) * CFrame.Angles(0, -a, 0),
            C.Metal, Enum.Material.Metal, f, 0, true)
        for side = -1, 1, 2 do
            part("Guarda_corpo", Vector3.new(2, 8, 180),
                CFrame.new(p + Vector3.new(side * 10, 5, 0)) * CFrame.Angles(0, -a, 0),
                C.Silver, Enum.Material.Metal, f, 0, true)
        end
    end
end

local function buildUrbanDetails()
    local f = folder("10_DETALHES_URBANOS")
    for i = 1, 24 do
        local a = i * math.pi / 12
        local pos = CENTER + Vector3.new(math.cos(a) * 480, 0, math.sin(a) * 480)
        part("Poste", Vector3.new(2, 22, 2), CFrame.new(pos + Vector3.new(0, 11, 0)),
            C.Metal, Enum.Material.Metal, f, 0, true)
        part("Braco", Vector3.new(9, 1.5, 1.5), CFrame.new(pos + Vector3.new(4, 21, 0)),
            C.Silver, Enum.Material.Metal, f, 0, false)
        neon("Luminaria", Vector3.new(5, 1, 4),
            CFrame.new(pos + Vector3.new(8, 20, 0)), C.Bright, f)
    end

    for i = 1, 16 do
        local a = i * math.pi / 8
        local pos = CENTER + Vector3.new(math.cos(a) * 165, 0, math.sin(a) * 165)
        neon("Balizador", Vector3.new(2, 7, 2),
            CFrame.new(pos + Vector3.new(0, 5, 0)), C.Bright, f)
    end
end

--========================================================
-- RECONSTRUCAO TRANSACIONAL
--========================================================
local BUILD_STEPS = {
    {"Globo e atmosfera", buildGlobe},
    {"Centro planetario", buildCentralHub},
    {"Citadel", buildCitadel},
    {"Arquivos centrais", buildArchives},
    {"Prisao", buildPrison},
    {"Jardins", buildGardens},
    {"Distritos", buildDistricts},
    {"Skyline", buildSkyline},
    {"Transporte", buildTransport},
    {"Detalhes urbanos", buildUrbanDetails},
}

local function buildOA(onProgress)
    if BUILDING then return false, "Construcao em andamento." end
    BUILDING = true

    local oldMap = workspace:FindFirstChild(MAP_NAME)
    local staging = Instance.new("Folder")
    staging.Name = MAP_NAME .. "_STAGING"
    staging.Parent = workspace

    local previousMap = Map
    Map = staging
    PART_COUNT = 0
    BUILDING_COUNT = 0
    CELL_DOORS = {}
    DOOR_STATES = {}
    LOCKDOWN = false

    local success, err = xpcall(function()
        for index, step in ipairs(BUILD_STEPS) do
            if onProgress then onProgress(step[1], index, #BUILD_STEPS) end
            step[2]()
            task.wait()
        end
    end, debug.traceback)

    if success then
        -- Commit: a nova arvore ja existe antes de remover a anterior.
        staging.Name = MAP_NAME
        if oldMap and oldMap.Parent then oldMap:Destroy() end
        local legacyMap = workspace:FindFirstChild(LEGACY_MAP_NAME)
        if legacyMap and legacyMap.Parent then legacyMap:Destroy() end
        Map = staging
        BUILDING = false
        return true, string.format("OA V%s pronta | %d pecas | %d edificios.",
            VERSION, PART_COUNT, BUILDING_COUNT)
    end

    -- Rollback: preserva a versao anterior se a construcao falhar.
    if staging and staging.Parent then staging:Destroy() end
    Map = oldMap or previousMap
    CELL_DOORS = {}
    DOOR_STATES = {}
    LOCKDOWN = false
    BUILDING = false
    warn("[OA V9.1] Reconstrucao cancelada; rollback executado:\n" .. tostring(err))
    return false, tostring(err)
end

--========================================================
-- INTERFACE
--========================================================
local oldGui = PlayerGui:FindFirstChild(GUI_NAME)
if oldGui then oldGui:Destroy() end
local legacyGui = PlayerGui:FindFirstChild(LEGACY_GUI_NAME)
if legacyGui then legacyGui:Destroy() end

local GUI = Instance.new("ScreenGui")
GUI.Name = GUI_NAME
GUI.ResetOnSpawn = false
GUI.IgnoreGuiInset = true
GUI.DisplayOrder = 20
GUI.Parent = PlayerGui

local Main = Instance.new("Frame")
Main.Name = "Painel"
Main.Size = UDim2.new(0, 300, 0, 410)
Main.Position = UDim2.new(0, 20, 0.5, -205)
Main.BackgroundColor3 = Color3.fromRGB(10, 23, 19)
Main.BackgroundTransparency = 0.05
Main.BorderSizePixel = 0
Main.Active = true
Main.Parent = GUI

local mainCorner = Instance.new("UICorner")
mainCorner.CornerRadius = UDim.new(0, 12)
mainCorner.Parent = Main
local mainStroke = Instance.new("UIStroke")
mainStroke.Color = C.Green
mainStroke.Thickness = 2
mainStroke.Parent = Main

local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 50)
Header.BackgroundColor3 = Color3.fromRGB(13, 62, 42)
Header.BorderSizePixel = 0
Header.Parent = Main
local headerCorner = Instance.new("UICorner")
headerCorner.CornerRadius = UDim.new(0, 12)
headerCorner.Parent = Header

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -55, 1, 0)
Title.Position = UDim2.new(0, 12, 0, 0)
Title.BackgroundTransparency = 1
Title.Text = "OA | CONTROLE PLANETARIO"
Title.TextColor3 = C.White
Title.TextSize = 13
Title.Font = Enum.Font.GothamBold
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = Header

local Minimize = Instance.new("TextButton")
Minimize.Size = UDim2.new(0, 36, 0, 34)
Minimize.Position = UDim2.new(1, -43, 0, 8)
Minimize.BackgroundColor3 = C.DarkGreen
Minimize.Text = "_"
Minimize.TextColor3 = C.White
Minimize.TextSize = 20
Minimize.Font = Enum.Font.GothamBold
Minimize.Parent = Header
local minCorner = Instance.new("UICorner")
minCorner.CornerRadius = UDim.new(0, 7)
minCorner.Parent = Minimize

local Content = Instance.new("ScrollingFrame")
Content.Size = UDim2.new(1, -16, 1, -62)
Content.Position = UDim2.new(0, 8, 0, 56)
Content.BackgroundTransparency = 1
Content.BorderSizePixel = 0
Content.ScrollBarThickness = 4
Content.ScrollBarImageColor3 = C.Green
Content.CanvasSize = UDim2.new(0, 0, 0, 0)
Content.AutomaticCanvasSize = Enum.AutomaticSize.Y
Content.Parent = Main

local Layout = Instance.new("UIListLayout")
Layout.Padding = UDim.new(0, 8)
Layout.SortOrder = Enum.SortOrder.LayoutOrder
Layout.Parent = Content

local Status = Instance.new("TextLabel")
Status.Size = UDim2.new(1, -4, 0, 42)
Status.BackgroundColor3 = Color3.fromRGB(20, 45, 35)
Status.Text = "OA V9.1 | Inicializando"
Status.TextColor3 = C.Bright
Status.TextSize = 12
Status.Font = Enum.Font.Gotham
Status.LayoutOrder = 0
Status.Parent = Content
local statusCorner = Instance.new("UICorner")
statusCorner.CornerRadius = UDim.new(0, 7)
statusCorner.Parent = Status

local function setStatus(message)
    Status.Text = message
end

local function makeButton(text, order, callback)
    local button = Instance.new("TextButton")
    button.Name = text
    button.Size = UDim2.new(1, -4, 0, 42)
    button.LayoutOrder = order
    button.BackgroundColor3 = Color3.fromRGB(20, 72, 48)
    button.Text = text
    button.TextColor3 = C.White
    button.TextSize = 12
    button.Font = Enum.Font.GothamBold
    button.AutoButtonColor = true
    button.Parent = Content

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = button
    local stroke = Instance.new("UIStroke")
    stroke.Color = C.Green
    stroke.Transparency = 0.55
    stroke.Parent = button

    button.Activated:Connect(callback)
    return button
end

local function teleportToOA()
    local character = Player.Character
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    if not character or not humanoid or humanoid.Health <= 0 then
        setStatus("Personagem indisponivel.")
        return
    end
    character:PivotTo(CFrame.new(CENTER + Vector3.new(0, 35, 0)))
    setStatus("Voce chegou ao centro de OA.")
end

makeButton("TELEPORTAR PARA OA", 1, teleportToOA)

makeButton("RECONSTRUIR PLANETA", 2, function()
    if BUILDING then setStatus("Construcao ja em andamento.") return end
    setStatus("Preparando reconstrucao...")
    task.spawn(function()
        local ok, message = buildOA(function(stepName, index, total)
            setStatus(string.format("Construindo: %s (%d/%d)", stepName, index, total))
        end)
        setStatus(ok and message or ("Falha: " .. message))
    end)
end)

makeButton("LIMPAR PLANETA", 3, function()
    if BUILDING then setStatus("Aguarde a construcao terminar.") return end
    local current = workspace:FindFirstChild(MAP_NAME)
    if current then current:Destroy() end
    local legacy = workspace:FindFirstChild(LEGACY_MAP_NAME)
    if legacy then legacy:Destroy() end
    Map = nil
    CELL_DOORS = {}
    DOOR_STATES = {}
    LOCKDOWN = false
    setStatus("OA removida deste cliente.")
end)

makeButton("ATIVAR LOCKDOWN", 4, function()
    if BUILDING then setStatus("Aguarde a construcao.") return end
    if not Map or not Map.Parent then setStatus("Construa OA primeiro.") return end
    setLockdown(not LOCKDOWN)
    setStatus(LOCKDOWN and "LOCKDOWN ATIVADO." or "LOCKDOWN DESATIVADO.")
end)

makeButton("RESTAURAR PORTAS", 5, function()
    if LOCKDOWN then setStatus("Desative o lockdown primeiro.") return end
    for _, door in ipairs(CELL_DOORS) do
        local state = DOOR_STATES[door]
        if state and state.opened then toggleCellDoor(door, true) end
    end
    setStatus("Portas restauradas.")
end)

makeButton("VOLTAR AO CENTRO", 6, teleportToOA)

local Info = Instance.new("TextLabel")
Info.Size = UDim2.new(1, -4, 0, 45)
Info.LayoutOrder = 7
Info.BackgroundTransparency = 1
Info.Text = "PLANETA DOS GUARDIOES\nOA V9.1 | SISTEMA LOCAL"
Info.TextColor3 = C.LightStone
Info.TextSize = 11
Info.Font = Enum.Font.Gotham
Info.Parent = Content

--========================================================
-- MINIMIZAR E ARRASTAR (PC / MOBILE)
--========================================================
local minimized = false
Minimize.Activated:Connect(function()
    minimized = not minimized
    Content.Visible = not minimized
    Main.Size = minimized and UDim2.new(0, 300, 0, 50) or UDim2.new(0, 300, 0, 410)
    Minimize.Text = minimized and "+" or "_"
end)

local dragging = false
local dragInput, dragStart, startPosition

Header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPosition = Main.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)

Header.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement
        or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and input == dragInput then
        local delta = input.Position - dragStart
        Main.Position = UDim2.new(startPosition.X.Scale, startPosition.X.Offset + delta.X,
            startPosition.Y.Scale, startPosition.Y.Offset + delta.Y)
    end
end)

--========================================================
-- INICIALIZACAO
--========================================================
setStatus("Construindo OA V9.1...")
task.spawn(function()
    local ok, message = buildOA(function(stepName, index, total)
        setStatus(string.format("Construindo: %s (%d/%d)", stepName, index, total))
    end)
    if ok then
        setStatus(message)
    else
        setStatus("Falha na construcao. Consulte o Output.")
        warn(message)
    end
end)

print("========================================")
print(" OA V9.1 | PLANETA DOS GUARDIOES")
print(" Reconstrucao transacional habilitada.")
print(" Anéis relativos a origem local.")
print(" Edificios com interiores modulares.")
print(" Lockdown e portas interativas.")
print(" Lighting preservado.")
print("========================================")
