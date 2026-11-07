--[[
    OA FINAL | PLANETA DOS GUARDIOES
    Reescala urbana, arquitetura coerente e refinamento visual
    Tipo: Script Luau para execucao local pelo Xeno

    ESCOPO:
    Todo o mapa, a interface, as luzes, as portas e as interacoes sao
    criados no cliente local. Nada e enviado ao servidor e outros jogadores
    nao recebem estas alteracoes. O codigo nao usa APIs exclusivas de
    executor, chamadas remotas ou mecanismos de contorno de protecoes.

    REVISAO VISUAL DO GLOBO:
    - Visto de fora, o globo e um planeta verde opaco e luminoso, com
      nevoa verde, nuvens e destrocos ao redor. A cidade nao aparece.
    - Ao entrar no globo, a casca e a nevoa desaparecem suavemente e OA
      fica visivel por dentro.
    - Estruturas externas (orbitas, estacoes, satelites, emissores e
      berco inferior) foram refinadas.
]]

--========================================================
-- SERVICOS E CONFIGURACAO
--========================================================
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")

local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")

local VERSION = "FINAL"
local MAP_NAME = "OA_LIVING_FINAL"
local LEGACY_MAP_NAME = "OA_LIVING_V10"
local PREVIOUS_MINOR_PATCH_MAP_NAME = "OA_LIVING_V9_3_5"
local PREVIOUS_MINOR_PATCH_3_MAP_NAME = "OA_LIVING_V9_3_4"
local PREVIOUS_MINOR_PATCH_OLD_MAP_NAME = "OA_LIVING_V9_3_3"
local PREVIOUS_MINOR_PATCH_2_MAP_NAME = "OA_LIVING_V9_3_2"
local PREVIOUS_MAP_NAME = "OA_LIVING_V9_3"
local PREVIOUS_PATCH_MAP_NAME = "OA_LIVING_V9_3_1"
local PREVIOUS_MINOR_MAP_NAME = "OA_LIVING_V9_2"
local OLDER_MAP_NAME = "OA_LIVING_V9_1"
local GUI_NAME = "OA_CONTROL_PANEL_FINAL"
local LEGACY_GUI_NAME = "OA_CONTROL_PANEL_V10"
local PREVIOUS_MINOR_PATCH_GUI_NAME = "OA_CONTROL_PANEL_V9_3_5"
local PREVIOUS_MINOR_PATCH_3_GUI_NAME = "OA_CONTROL_PANEL_V9_3_4"
local PREVIOUS_MINOR_PATCH_OLD_GUI_NAME = "OA_CONTROL_PANEL_V9_3_3"
local PREVIOUS_MINOR_PATCH_2_GUI_NAME = "OA_CONTROL_PANEL_V9_3_2"
local PREVIOUS_GUI_NAME = "OA_CONTROL_PANEL_V9_3"
local PREVIOUS_PATCH_GUI_NAME = "OA_CONTROL_PANEL_V9_3_1"
local PREVIOUS_MINOR_GUI_NAME = "OA_CONTROL_PANEL_V9_2"
local OLDER_GUI_NAME = "OA_CONTROL_PANEL_V9_1"

local CENTER = Vector3.new(0, 6000, 0)
local GLOBE_RADIUS = 950
-- Escala urbana global: reduz estruturas e deslocamentos mantendo o planeta intacto.
local CITY_SCALE = 0.72
local CITY_ROOTS = {
    ["02_CENTRO_PLANETARIO"] = true, ["02B_MAPA_HOLOGRAFICO"] = true,
    ["03_CITADEL_DOS_GUARDIOES"] = true, ["04_ARQUIVOS_CENTRAIS"] = true,
    ["05_PRISAO_DE_CONTENCAO"] = true, ["06_BATERIA_CENTRAL"] = true, ["06_JARDINS"] = true, ["07_DISTRITOS"] = true,
    ["08_SKYLINE"] = true, ["09_TRANSPORTE"] = true,
    ["09B_HANGAR_ORBITAL"] = true, ["10_DETALHES_URBANOS"] = true,
    ["11_CENTRO_COMANDO"] = true, ["12_MUSEU_GUARDIOES"] = true,
}

local BUILDING = false
local PART_COUNT = 0
local BUILDING_COUNT = 0
local Map = nil
local CELL_DOORS = {}
local DOOR_STATES = {}
local LOCKDOWN = false
local BUILD_YIELD_INTERVAL = 140
local STUD_TO_METERS = 0.28 -- escala aproximada padrao do Roblox; ajuste se o mapa usar outra escala.
-- A Bateria Central ocupa o eixo geometrico do planeta.
local BATTERY_POSITION = CENTER
local PRISON_POSITION = CENTER + Vector3.new(365, 0, -180)
local runtimeConnections = {}
local atmosphereActive = false
local gpsDestinationIndex = 1

-- Pecas da "pele externa" do planeta (casca, nevoa, nuvens, campo).
-- Somem suavemente quando a camera entra no globo.
local OUTER_PARTS = {}
local outerFade = 0
local lastAppliedFade = -1

-- Esferas de energia que orbitam a Bateria Central e luz pulsante do nucleo.
local batteryOrbs = {}
local batteryCoreLight = nil

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
    -- Paleta do planeta visto de fora (inspirada na referencia).
    PlanetLime = Color3.fromRGB(165, 232, 82),
    PlanetPale = Color3.fromRGB(222, 255, 160),
    HaloGreen = Color3.fromRGB(120, 255, 70),
    DebrisGreen = Color3.fromRGB(38, 92, 52),
}

local function folder(name, parent)
    local f = Instance.new("Folder")
    f.Name = name
    f.Parent = parent or Map
    return f
end

local function tagOuter(p, baseTransparency)
    OUTER_PARTS[#OUTER_PARTS + 1] = {part = p, base = baseTransparency or p.Transparency}
    return p
end

local function applyOuterFade(fade)
    for _, entry in ipairs(OUTER_PARTS) do
        local p = entry.part
        if p and p.Parent then
            p.Transparency = entry.base + (1 - entry.base) * fade
        end
    end
end

-- Todos os objetos sao ancorados. Por padrao, pecas decorativas nao
-- participam de consultas/toques; addPrompt reativa CanQuery no suporte.
local function part(name, size, cf, color, material, parent, transparency, collidable)
    local p = Instance.new("Part")
    p.Name = name
    local targetParent = parent or Map
    local ancestor = targetParent
    while ancestor and Map and ancestor.Parent and ancestor.Parent ~= Map do
        ancestor = ancestor.Parent
    end
    local isCityGeometry = ancestor and CITY_ROOTS[ancestor.Name] == true
    if isCityGeometry then
        size = size * CITY_SCALE
        local scaledPosition = CENTER + (cf.Position - CENTER) * CITY_SCALE
        cf = CFrame.new(scaledPosition) * (cf - cf.Position)
    end
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
    p.Parent = targetParent

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
    -- Placas elevadas recebem montantes discretos ligados ao elemento abaixo.
    local postHeight = math.max(8, height * 0.55)
    for side = -1, 1, 2 do
        part("Suporte_placa", Vector3.new(1.4, postHeight, 1.4),
            CFrame.new(position + Vector3.new(side * width * 0.38, -height/2 - postHeight/2 + 1, 0)),
            C.Silver, Enum.Material.Metal, parent, 0, true)
    end

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
            if not ok then warn("[OA V9.3] Falha em interacao:", err) end
        end
    end)
    return prompt
end

local function randomDirection(rng)
    local v = Vector3.new(rng:NextNumber(-1, 1), rng:NextNumber(-1, 1), rng:NextNumber(-1, 1))
    if v.Magnitude < 0.05 then return Vector3.new(0, 1, 0) end
    return v.Unit
end

--========================================================
-- GLOBO, ATMOSFERA, ORBITAS E ESTRUTURAS EXTERNAS
--========================================================
local function buildGlobe()
    local f = folder("01_PLANETA_E_ATMOSFERA")
    local rng = Random.new(906)

    -- Casca opaca: de fora, o planeta e uma esfera verde-limao luminosa.
    -- A cidade dentro dela fica totalmente oculta ate o jogador entrar.
    local shell = ball("Globo_de_OA", GLOBE_RADIUS * 2, CFrame.new(CENTER),
        C.PlanetLime, Enum.Material.Neon, f, 0, false)
    shell.CastShadow = false
    tagOuter(shell, 0)

    -- Faixas de nuvem claras sobre a superficie, como na referencia.
    local function band(latDeg, height, color, alpha, segments)
        local lat = math.rad(latDeg)
        local radius = GLOBE_RADIUS * math.cos(lat) + 1
        local y = GLOBE_RADIUS * math.sin(lat)
        local arc = (2 * math.pi * radius / segments) * 1.02
        for i = 1, segments do
            local mid = ((i - 0.5) / segments) * math.pi * 2
            local strip = part("Faixa_de_nuvem", Vector3.new(3, height, arc),
                CFrame.new(CENTER + Vector3.new(math.cos(mid) * radius, y, math.sin(mid) * radius))
                    * CFrame.Angles(0, -mid, 0),
                color, Enum.Material.Neon, f, alpha, false)
            strip.CastShadow = false
            tagOuter(strip, alpha)
        end
    end
    band(34, 5, C.PlanetPale, 0.62, 72)
    band(22, 4, C.PlanetPale, 0.70, 72)
    band(8, 6, C.PlanetPale, 0.58, 72)
    band(-12, 5, C.PlanetPale, 0.66, 72)
    band(-30, 4, C.PlanetPale, 0.72, 72)

    -- Camadas de brilho: halo verde que cria o resplendor da imagem.
    local haloLayers = {
        {1.05, 0.905}, {1.09, 0.945}, {1.15, 0.975},
    }
    for index, layer in ipairs(haloLayers) do
        local halo = ball("Halo_verde_" .. index, GLOBE_RADIUS * 2 * layer[1],
            CFrame.new(CENTER), C.HaloGreen, Enum.Material.Neon, f, layer[2], false)
        halo.CastShadow = false
        tagOuter(halo, layer[2])
    end

    -- Nevoa gasosa: blobs translucidos espalhados em volta do planeta.
    for i = 1, 14 do
        local dir = randomDirection(rng)
        local dist = GLOBE_RADIUS * rng:NextNumber(1.06, 1.22)
        local size = rng:NextNumber(150, 300)
        local alpha = rng:NextNumber(0.93, 0.972)
        local cloud = ball("Nevoa_gasosa", size, CFrame.new(CENTER + dir * dist),
            i % 3 == 0 and C.PlanetPale or C.HaloGreen, Enum.Material.Neon, f, alpha, false)
        cloud.CastShadow = false
        tagOuter(cloud, alpha)
    end

    -- Destrocos e fragmentos flutuando ao redor, como na referencia.
    for i = 1, 70 do
        local dir = randomDirection(rng)
        local dist = GLOBE_RADIUS * rng:NextNumber(1.13, 1.55)
        local s = rng:NextNumber(4, 16)
        local shard = part("Fragmento_orbital",
            Vector3.new(s, s * rng:NextNumber(0.5, 1.4), s * rng:NextNumber(0.6, 1.2)),
            CFrame.new(CENTER + dir * dist)
                * CFrame.Angles(rng:NextNumber(0, 6.28), rng:NextNumber(0, 6.28), rng:NextNumber(0, 6.28)),
            i % 5 == 0 and C.Green or C.DebrisGreen, Enum.Material.Slate, f, 0, false)
        shard.CastShadow = false
    end

    -- Globo sem feicoes: rosto e acessorios removidos.

    -- Orbitas (mantidas), agora com anel luminoso fino por cima.
    buildRing("Orbita_principal", CENTER, 1160, 5, 0, C.Green, 80, f)
    buildRing("Orbita_luminosa", CENTER, 1160, 1.6, 3, C.Bright, 80, f, Enum.Material.Neon)
    buildRing("Orbita_secundaria", CENTER, 1220, 2, 100, C.Silver, 80, f)
    buildRing("Orbita_dourada", CENTER, 1090, 3, -110, C.Gold, 80, f)

    -- Estacoes orbitais sobre a orbita principal.
    for i = 1, 6 do
        local a = (i - 1) * math.pi / 3 + math.pi / 6
        local pos = CENTER + Vector3.new(math.cos(a) * 1160, 0, math.sin(a) * 1160)
        local face = CFrame.new(pos) * CFrame.Angles(0, -a, 0)
        cylinder("Estacao_orbital_modulo", 34, 22, CFrame.new(pos),
            C.Metal, Enum.Material.Metal, f, false)
        local ring = cylinder("Estacao_orbital_anel", 40, 3,
            CFrame.new(pos + Vector3.new(0, 7, 0)), C.Bright, Enum.Material.Neon, f, false)
        ring.CastShadow = false
        for side = -1, 1, 2 do
            part("Estacao_painel_solar", Vector3.new(34, 1, 12),
                face * CFrame.new(0, 0, side * 32), C.DarkGreen, Enum.Material.Glass, f, 0.15, false)
            part("Estacao_braco", Vector3.new(2, 2, 20),
                face * CFrame.new(0, 0, side * 16), C.Silver, Enum.Material.Metal, f, 0, false)
        end
        local lamp = neon("Estacao_luz", Vector3.new(6, 4, 6),
            CFrame.new(pos + Vector3.new(0, 14, 0)), C.Bright, f)
        makeLight(lamp, C.Bright, 0.9, 45)
    end

    -- Satelites (mantidos), agora com paineis solares e antena.
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
        local face = CFrame.new(pos) * CFrame.Angles(0, -a, 0)
        for side = -1, 1, 2 do
            part("Painel_solar_satelite", Vector3.new(2, 1, 22),
                face * CFrame.new(side * 0, 0, side * 24), C.DarkGreen, Enum.Material.Glass, f, 0.18, false)
            part("Haste_painel", Vector3.new(1.2, 1.2, 8),
                face * CFrame.new(0, 0, side * 13), C.Silver, Enum.Material.Metal, f, 0, false)
        end
        cylinder("Antena_satelite", 7, 1.5,
            face * CFrame.new(0, 14, 0), C.Silver, Enum.Material.Metal, f, false)
    end

    for i = 1, 4 do
        local a = i * math.pi / 2
        local pos = CENTER + Vector3.new(math.cos(a) * 960, 0, math.sin(a) * 960)
        local anchor = ball("Ancora_atmosferica", 15, CFrame.new(pos),
            C.Bright, Enum.Material.Neon, f, 0, false)
        tagOuter(anchor, 0)
        if i == 1 then makeLight(anchor, C.Bright, 1.5, 40) end
    end
end

--========================================================
-- ESTRUTURA INFERIOR DO PLANETA (BERCO DE SUSTENTACAO)
--========================================================
local function buildPlanetaryUnderside()
    local f = folder("01A_ESTRUTURA_INFERIOR_DO_PLANETA")
    local baseY = -GLOBE_RADIUS - 62

    -- Aneis do berco, abaixo do hemisferio inferior.
    for index, radius in ipairs({640, 500, 360}) do
        buildRing("Aro_inferior_estrutural_" .. index, CENTER, radius,
            index == 1 and 9 or 5, baseY - (index - 1) * 38,
            index == 2 and C.Bright or C.Green, 72, f,
            index == 2 and Enum.Material.Neon or Enum.Material.Metal)
    end

    -- Disco-base largo e coluna central de sustentacao.
    cylinder("Disco_base_inferior", 300, 12,
        CFrame.new(CENTER + Vector3.new(0, baseY - 118, 0)), C.DarkStone, Enum.Material.Metal, f, false)
    cylinder("Coluna_inferior_central", 70, 150,
        CFrame.new(CENTER + Vector3.new(0, -GLOBE_RADIUS - 52, 0)), C.DarkGreen, Enum.Material.Metal, f, false)
    cylinder("Colar_coluna_inferior", 92, 8,
        CFrame.new(CENTER + Vector3.new(0, -GLOBE_RADIUS - 118, 0)), C.Silver, Enum.Material.Metal, f, false)
    local core = ball("Nucleo_inferior", 62,
        CFrame.new(CENTER + Vector3.new(0, baseY - 150, 0)),
        C.Bright, Enum.Material.Neon, f, 0, false)
    makeLight(core, C.Bright, 1.8, 120)

    -- Nervuras: do anel externo ate pontos na superficie do globo (latitude -55).
    local lat = math.rad(-55)
    local surfaceRadius = GLOBE_RADIUS * math.cos(lat)
    local surfaceY = GLOBE_RADIUS * math.sin(lat)
    for i = 1, 8 do
        local a = (i - 1) * math.pi / 4
        local outer = CENTER + Vector3.new(math.cos(a) * 640, baseY, math.sin(a) * 640)
        local onSurface = CENTER + Vector3.new(math.cos(a) * surfaceRadius, surfaceY, math.sin(a) * surfaceRadius)
        beam("Nervura_inferior", outer, onSurface, 9, C.Silver, f)
        local inner = CENTER + Vector3.new(math.cos(a) * 140, baseY - 118, math.sin(a) * 140)
        beam("Tirante_inferior", outer - Vector3.new(0, 4, 0), inner, 4, C.Metal, f)
        local node = ball("No_energetico_inferior", 20, CFrame.new(onSurface), C.Green, Enum.Material.Neon, f, 0, false)
        if i % 2 == 0 then makeLight(node, C.Green, 0.7, 40) end
        ball("Junta_berco", 16, CFrame.new(outer), C.Silver, Enum.Material.Metal, f, 0, false)
    end
end

--========================================================
-- CAMPO DE PROTECAO VISUAL E IDENTIFICADOR DISTANTE DA OA
--========================================================
local function buildProtectionField()
    local f = folder("01B_CAMPO_DE_PROTECAO")
    local outerDiameter = GLOBE_RADIUS * 2.075
    local shell = ball("Membrana_externa_do_campo", outerDiameter,
        CFrame.new(CENTER), C.Bright, Enum.Material.ForceField, f, 0.945, false)
    shell.CastShadow = false
    tagOuter(shell, 0.945)

    local innerShell = ball("Membrana_interna_do_campo", GLOBE_RADIUS * 2.045,
        CFrame.new(CENTER), C.Green, Enum.Material.Neon, f, 0.978, false)
    innerShell.CastShadow = false
    tagOuter(innerShell, 0.978)

    -- Linhas de energia em tres planos, sem criar uma malha excessivamente pesada.
    for _, piece in ipairs(buildRing("Linha_equatorial_do_escudo", CENTER, GLOBE_RADIUS * 1.035,
        2.2, 0, C.Bright, 64, f, Enum.Material.Neon)) do tagOuter(piece, 0) end
    for _, piece in ipairs(buildRing("Linha_polar_norte", CENTER, GLOBE_RADIUS * 0.82,
        1.5, GLOBE_RADIUS * 0.48, C.Green, 56, f, Enum.Material.Neon)) do tagOuter(piece, 0) end
    for _, piece in ipairs(buildRing("Linha_polar_sul", CENTER, GLOBE_RADIUS * 0.82,
        1.5, -GLOBE_RADIUS * 0.48, C.Green, 56, f, Enum.Material.Neon)) do tagOuter(piece, 0) end

    -- Emissores externos: pilar com base, anel e nucleo luminoso.
    local capPositions = {}
    for i = 1, 8 do
        local angle = (i - 1) * math.pi / 4
        local emitterPos = CENTER + Vector3.new(math.cos(angle) * 1000, 0, math.sin(angle) * 1000)
        cylinder("Base_do_emissor", 40, 12,
            CFrame.new(emitterPos + Vector3.new(0, 6, 0)), C.DarkStone, Enum.Material.Metal, f, false)
        cylinder("Emissor_do_campo", 22, 56,
            CFrame.new(emitterPos + Vector3.new(0, 36, 0)), C.DarkGreen, Enum.Material.Metal, f, false)
        cylinder("Colar_do_emissor", 30, 5,
            CFrame.new(emitterPos + Vector3.new(0, 52, 0)), C.Silver, Enum.Material.Metal, f, false)
        local capPos = emitterPos + Vector3.new(0, 66, 0)
        local cap = neon("Nucleo_emissor", Vector3.new(10, 14, 10), CFrame.new(capPos), C.Bright, f)
        makeLight(cap, C.Bright, 1.15, 42)
        capPositions[i] = capPos
        beam("Ligacao_emissor_ao_campo", capPos,
            CENTER + Vector3.new(math.cos(angle) * 970, 0, math.sin(angle) * 970),
            2.2, C.Bright, f)
    end
    -- Arcos de energia ligando emissores vizinhos.
    for i = 1, 8 do
        local nextIndex = (i % 8) + 1
        local arc = beam("Arco_de_energia_do_campo", capPositions[i], capPositions[nextIndex], 1.6, C.Bright, f)
        if arc then
            arc.Material = Enum.Material.Neon
            arc.CastShadow = false
        end
    end
end

local function buildDistantPlanetBeacon()
    local f = folder("01C_IDENTIFICADOR_DISTANTE")
    local anchor = part("OA_Beacon_Anchor", Vector3.new(2, 2, 2),
        CFrame.new(CENTER + Vector3.new(0, GLOBE_RADIUS * 1.18, 0)),
        C.Bright, Enum.Material.Neon, f, 1, false)
    anchor.CastShadow = false

    local billboard = Instance.new("BillboardGui")
    billboard.Name = "OA_Distant_Nameplate"
    billboard.Adornee = anchor
    billboard.AlwaysOnTop = true
    billboard.LightInfluence = 0
    billboard.MaxDistance = math.huge
    billboard.Size = UDim2.fromOffset(260, 76)
    billboard.StudsOffset = Vector3.new(0, 0, 0)
    billboard.Parent = anchor

    local title = Instance.new("TextLabel")
    title.Name = "PlanetName"
    title.Size = UDim2.new(1, 0, 0, 48)
    title.BackgroundTransparency = 1
    title.Text = "OA"
    title.TextColor3 = Color3.fromRGB(145, 255, 180)
    title.TextStrokeColor3 = Color3.fromRGB(5, 30, 18)
    title.TextStrokeTransparency = 0.12
    title.Font = Enum.Font.GothamBlack
    title.TextScaled = true
    title.Parent = billboard

    local subtitle = Instance.new("TextLabel")
    subtitle.Name = "PlanetSubtitle"
    subtitle.Position = UDim2.new(0, 0, 0, 45)
    subtitle.Size = UDim2.new(1, 0, 0, 23)
    subtitle.BackgroundTransparency = 1
    subtitle.Text = "PLANETA DOS GUARDIOES"
    subtitle.TextColor3 = C.White
    subtitle.TextStrokeColor3 = Color3.fromRGB(5, 30, 18)
    subtitle.TextStrokeTransparency = 0.18
    subtitle.Font = Enum.Font.GothamBold
    subtitle.TextScaled = true
    subtitle.Parent = billboard
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

    -- O centro fica reservado para a Bateria Central. O anel dourado
    -- delimita a praca sem criar um segundo nucleo que competiria com ela.
    label(f, "OA | NUCLEO PLANETARIO", CENTER + Vector3.new(0, 265, -135), 110, 22)

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

    -- Linguagem arquitetonica: base pesada, estrutura legivel e fachada em modulos.
    local isCommercial = style == "commercial"
    local isCitadel = style == "citadel"
    local wallColor = isCommercial and C.Metal or (isCitadel and C.DarkStone or C.Stone)
    local accentColor = isCitadel and C.Gold or (isCommercial and C.Silver or C.Green)
    local glassColor = isCitadel and C.Gold or C.Glass
    local wallThickness = math.max(3, math.min(5, width * 0.055))
    local floorHeight = 20
    local floorCount = math.max(1, math.floor(height / floorHeight))
    local innerWidth = width - wallThickness * 2
    local innerDepth = depth - wallThickness * 2
    local entryWidth = math.max(12, width * 0.28)
    local frontZ = position.Z - depth / 2 + wallThickness / 2
    local wallY = position.Y + 5 + height / 2
    local facadeY = position.Y + 5

    -- Fundacao e embasamento: a construcao parece apoiada no terreno.
    part("Fundacao", Vector3.new(width + 10, 5, depth + 10),
        CFrame.new(position + Vector3.new(0, 2.5, 0)),
        C.DarkStone, Enum.Material.Metal, f, 0, true)
    part("Podio_inferior", Vector3.new(width + 4, 3, depth + 4),
        CFrame.new(position + Vector3.new(0, 6.5, 0)),
        accentColor, Enum.Material.Metal, f, 0, true)

    -- Lajes independentes mantem o interior aberto e caminhavel.
    for level = 0, floorCount - 1 do
        local y = position.Y + 8 + level * floorHeight
        part("Laje_" .. (level + 1), Vector3.new(innerWidth, 1.4, innerDepth),
            CFrame.new(position.X, y, position.Z),
            level == 0 and C.Floor or C.DarkStone,
            Enum.Material.Concrete, f, 0, true)
    end

    -- Paredes principais, com vao frontal de entrada.
    part("Parede_esquerda", Vector3.new(wallThickness, height, depth),
        CFrame.new(position.X - width / 2 + wallThickness / 2, wallY, position.Z),
        wallColor, Enum.Material.Concrete, f, 0, true)
    part("Parede_direita", Vector3.new(wallThickness, height, depth),
        CFrame.new(position.X + width / 2 - wallThickness / 2, wallY, position.Z),
        wallColor, Enum.Material.Concrete, f, 0, true)
    part("Parede_traseira", Vector3.new(width, height, wallThickness),
        CFrame.new(position.X, wallY, position.Z + depth / 2 - wallThickness / 2),
        wallColor, Enum.Material.Concrete, f, 0, true)

    local sideWidth = (width - entryWidth) / 2
    part("Fachada_frontal_esquerda", Vector3.new(sideWidth, height, wallThickness),
        CFrame.new(position.X - (entryWidth + sideWidth) / 2, wallY, frontZ),
        wallColor, Enum.Material.Concrete, f, 0, true)
    part("Fachada_frontal_direita", Vector3.new(sideWidth, height, wallThickness),
        CFrame.new(position.X + (entryWidth + sideWidth) / 2, wallY, frontZ),
        wallColor, Enum.Material.Concrete, f, 0, true)
    part("Verga_da_entrada", Vector3.new(entryWidth, 5, wallThickness + 1),
        CFrame.new(position.X, position.Y + 5 + height - 2.5, frontZ),
        C.DarkStone, Enum.Material.Metal, f, 0, true)

    -- Pilares de fachada: ritmo estrutural regular em vez de pecas aleatorias.
    local bayCount = math.clamp(math.floor(width / 18), 2, 5)
    for bay = 0, bayCount do
        local x = position.X - width / 2 + (width / bayCount) * bay
        if math.abs(x - position.X) > entryWidth * 0.55 then
            part("Montante_fachada", Vector3.new(1.4, height, 1.6),
                CFrame.new(x, wallY, position.Z - depth / 2 - 0.8),
                accentColor, Enum.Material.Metal, f, 0, false)
        end
    end

    -- Modulos de janela por pavimento. Vidro recuado e peitoris criam profundidade.
    for level = 1, floorCount do
        local y = position.Y + 5 + math.min(height - 9, level * floorHeight - 8)
        local windowWidth = math.max(7, width / math.max(2, bayCount) * 0.62)
        for bay = 1, bayCount - 1 do
            local x = position.X - width / 2 + (width / bayCount) * bay
            if math.abs(x - position.X) > entryWidth * 0.48 then
                local pane = part("Modulo_de_vidro",
                    Vector3.new(windowWidth, 6.5, 0.8),
                    CFrame.new(x, y, position.Z - depth / 2 - 0.35),
                    glassColor, Enum.Material.Glass, f, 0.38, false)
                pane.CastShadow = false
                part("Peitoril", Vector3.new(windowWidth + 1.5, 0.8, 1.8),
                    CFrame.new(x, y - 3.8, position.Z - depth / 2 - 0.8),
                    accentColor, Enum.Material.Metal, f, 0, false)
            end
        end

        -- Faixas horizontais discretas marcam os pavimentos.
        part("Friso_pavimento_" .. level, Vector3.new(width + 2, 1.1, 1.8),
            CFrame.new(position.X, position.Y + 5 + level * (height / floorCount),
                position.Z - depth / 2 - 0.9),
            C.Silver, Enum.Material.Metal, f, 0, false)
    end

    -- Entrada legivel: moldura, cobertura e identificacao.
    part("Moldura_entrada_superior", Vector3.new(entryWidth + 5, 2, 3),
        CFrame.new(position.X, position.Y + 13, frontZ - 1.2),
        accentColor, Enum.Material.Metal, f, 0, false)
    part("Marquise", Vector3.new(entryWidth + 8, 2, 9),
        CFrame.new(position.X, position.Y + 10, frontZ - 4),
        C.DarkStone, Enum.Material.Metal, f, 0, true)
    local sign = neon("Identidade_visual", Vector3.new(math.min(width * 0.55, 26), 1.2, 0.6),
        CFrame.new(position.X, position.Y + height + 1, frontZ - 1.2),
        accentColor, f)
    sign.CastShadow = false

    -- Cobertura tecnica com parapeito e equipamentos compactos.
    part("Cobertura", Vector3.new(width + 5, 4, depth + 5),
        CFrame.new(position.X, position.Y + height + 7, position.Z),
        C.DarkStone, Enum.Material.Metal, f, 0, true)
    for side = -1, 1, 2 do
        part("Parapeito", Vector3.new(width + 5, 3, 2),
            CFrame.new(position.X, position.Y + height + 10, position.Z + side * (depth / 2 + 1.5)),
            accentColor, Enum.Material.Metal, f, 0, false)
    end
    for unit = 1, 2 do
        part("Equipamento_de_cobertura", Vector3.new(7, 3, 6),
            CFrame.new(position.X + (unit == 1 and -1 or 1) * width * 0.22,
                position.Y + height + 11.5, position.Z + depth * 0.18),
            C.Metal, Enum.Material.Metal, f, 0, false)
    end

    -- Escada compacta de servico; mantida simples para limitar pecas.
    if floorCount > 1 then
        local steps = math.floor(floorHeight / 2.5)
        for level = 0, floorCount - 2 do
            for step = 1, steps do
                local y = position.Y + 9 + level * floorHeight + step * 2
                local z = position.Z - innerDepth * 0.28 + step * 0.55
                part("Degrau_escada", Vector3.new(7, 1.2, 4.5),
                    CFrame.new(position.X + innerWidth * 0.32, y, z),
                    C.Silver, Enum.Material.Metal, f, 0, true)
            end
        end
    end

    -- Interior funcional em escala moderada.
    part("Balcao_recepcao", Vector3.new(width * 0.36, 4, 5),
        CFrame.new(position + Vector3.new(0, 10, depth * 0.18)),
        C.Metal, Enum.Material.Metal, f, 0, true)
    for i = 1, 3 do
        part("Assento_recepcao", Vector3.new(7, 2, 4),
            CFrame.new(position + Vector3.new(-width * 0.24 + i * width * 0.14, 9, -depth * 0.2)),
            C.DarkGreen, Enum.Material.Metal, f, 0, true)
    end

    -- Reforma da fachada: colunas de canto, janelas laterais e faixas luminosas.
    for sx = -1, 1, 2 do
        for sz = -1, 1, 2 do
            part("Coluna_de_canto", Vector3.new(4, height + 8, 4),
                CFrame.new(position.X + sx * (width / 2 - 1), wallY + 2, position.Z + sz * (depth / 2 - 1)),
                C.DarkStone, Enum.Material.Metal, f, 0, false)
        end
    end
    for level = 1, floorCount do
        local y = position.Y + 5 + math.min(height - 9, level * floorHeight - 8)
        for side = -1, 1, 2 do
            for k = -1, 1, 2 do
                local pane = part("Janela_lateral", Vector3.new(0.8, 6.5, math.max(7, depth * 0.22)),
                    CFrame.new(position.X + side * (width / 2 + 0.35), y, position.Z + k * depth * 0.22),
                    glassColor, Enum.Material.Glass, f, 0.38, false)
                pane.CastShadow = false
            end
        end
    end
    for side = -1, 1, 2 do
        local strip = neon("Faixa_da_entrada", Vector3.new(0.8, 18, 0.8),
            CFrame.new(position.X + side * (entryWidth / 2 + 1.5), position.Y + 15, frontZ - 1.8),
            accentColor, f)
        strip.CastShadow = false
    end
    -- Coroa escalonada nos edificios altos.
    local crownLift = 0
    if height >= 90 then
        crownLift = 16
        part("Coroa_escalonada", Vector3.new(width * 0.6, 16, depth * 0.6),
            CFrame.new(position.X, position.Y + height + 17, position.Z),
            wallColor, Enum.Material.Metal, f, 0, true)
        local band = part("Faixa_de_vidro_coroa", Vector3.new(width * 0.6 + 0.6, 5, depth * 0.6 + 0.6),
            CFrame.new(position.X, position.Y + height + 18, position.Z),
            glassColor, Enum.Material.Glass, f, 0.4, false)
        band.CastShadow = false
        neon("Aro_luminoso_coroa", Vector3.new(width * 0.6 + 2, 1, depth * 0.6 + 2),
            CFrame.new(position.X, position.Y + height + 25.5, position.Z), accentColor, f)
    end

    local interior = neon("Iluminacao_interna", Vector3.new(width * 0.28, 0.5, 2),
        CFrame.new(position + Vector3.new(0, 5 + math.min(17, height - 5), 0)),
        C.Bright, f)
    makeLight(interior, C.Bright, 0.65, math.min(26, width * 0.4))

    local beacon = neon("Sinalizador_tecnico", Vector3.new(3, 6, 3),
        CFrame.new(position + Vector3.new(0, height + 14 + crownLift, 0)), accentColor, f)
    if BUILDING_COUNT % 4 == 0 then makeLight(beacon, accentColor, 0.55, 18) end
    return f
end
--========================================================
-- CITADEL DOS GUARDIOES | COMPLEXO ABERTO E SALAS INTERNAS
--========================================================
local function buildCitadel()
    local f = folder("03_CITADEL_DOS_GUARDIOES")
    local pos = CENTER + Vector3.new(0, 0, -440)
    local floorY = pos.Y + 18
    local frontZ = pos.Z - 112
    local wallY = pos.Y + 77

    -- Praca e plataforma em camadas. O eixo frontal permanece livre.
    cylinder("Plataforma_inferior", 430, 18,
        CFrame.new(pos + Vector3.new(0, 9, 0)), C.DarkStone, Enum.Material.Metal, f, true)
    cylinder("Plataforma_superior", 390, 7,
        CFrame.new(pos + Vector3.new(0, 21, 0)), C.Stone, Enum.Material.Marble, f, true)
    buildRing("Aro_perimetral_dourado", pos, 207, 5, 25, C.Gold, 72, f)
    buildRing("Aro_interno_verde", pos, 178, 3, 26, C.Bright, 64, f, Enum.Material.Neon)

    -- Escadaria ampla e baixa, sem bloco atravessando a passagem.
    for step = 1, 10 do
        local h = step * 1.8
        part("Escadaria_frontal_degrau_" .. step, Vector3.new(112, h, 7),
            CFrame.new(pos + Vector3.new(0, 18 + h/2, -116 - step*6.5)),
            step % 2 == 0 and C.LightStone or C.Stone, Enum.Material.Marble, f, 0, true)
    end

    -- Grande sala principal: piso, laterais e fundo. Frente em tres segmentos
    -- com vao central de 58 studs, sem parede, porta ou marquise bloqueando.
    part("Piso_grande_sala", Vector3.new(204, 3, 190),
        CFrame.new(pos + Vector3.new(0, 19.5, 0)), C.Floor, Enum.Material.Marble, f, 0, true)
    part("Parede_lateral_esquerda_sala", Vector3.new(8, 118, 190),
        CFrame.new(pos + Vector3.new(-102, wallY, 0)), C.DarkStone, Enum.Material.Concrete, f, 0, true)
    part("Parede_lateral_direita_sala", Vector3.new(8, 118, 190),
        CFrame.new(pos + Vector3.new(102, wallY, 0)), C.DarkStone, Enum.Material.Concrete, f, 0, true)
    part("Parede_fundo_sala", Vector3.new(204, 118, 8),
        CFrame.new(pos + Vector3.new(0, wallY, 95)), C.DarkStone, Enum.Material.Concrete, f, 0, true)
    part("Fachada_frontal_esquerda", Vector3.new(72, 118, 8),
        CFrame.new(pos + Vector3.new(-66, wallY, -95)), C.DarkStone, Enum.Material.Concrete, f, 0, true)
    part("Fachada_frontal_direita", Vector3.new(72, 118, 8),
        CFrame.new(pos + Vector3.new(66, wallY, -95)), C.DarkStone, Enum.Material.Concrete, f, 0, true)
    -- Moldura monumental alta; o vao inferior central continua totalmente aberto.
    part("Viga_superior_entrada", Vector3.new(60, 12, 12),
        CFrame.new(pos + Vector3.new(0, pos.Y*0 + 132, -95)), C.Gold, Enum.Material.Metal, f, 0, false)
    for side = -1, 1, 2 do
        part("Pilar_portal_entrada", Vector3.new(10, 112, 12),
            CFrame.new(pos + Vector3.new(side*30, 75, -95)), C.LightStone, Enum.Material.Marble, f, 0, true)
        local strip = neon("Filete_luminoso_entrada", Vector3.new(1.2, 96, 1.2),
            CFrame.new(pos + Vector3.new(side*35, 75, -101)), C.Bright, f)
        makeLight(strip, C.Bright, 0.35, 16)
    end

    -- Cobertura integral da Citadel: laje apoiada nas paredes e nos pilares.
    part("Cobertura_principal_citadel", Vector3.new(216, 8, 202),
        CFrame.new(pos + Vector3.new(0, 139, 0)), C.DarkStone, Enum.Material.Metal, f, 0, true)
    part("Beiral_frontal_citadel", Vector3.new(220, 4, 18),
        CFrame.new(pos + Vector3.new(0, 135, -101)), C.Gold, Enum.Material.Metal, f, 0, false)
    for side = -1, 1, 2 do
        part("Calha_lateral_citadel", Vector3.new(4, 3, 194),
            CFrame.new(pos + Vector3.new(side*106, 135, 0)), C.Silver, Enum.Material.Metal, f, 0, false)
    end
    -- Claraboias estreitas iluminam o salao sem deixar a cobertura aberta.
    for i = -1, 1 do
        part("Claraboia_citadel", Vector3.new(34, 1.2, 22),
            CFrame.new(pos + Vector3.new(i*58, 143.5, 12)), C.Glass, Enum.Material.Glass, f, 0.28, false)
    end

    -- Colunas internas laterais, mantendo corredor central livre.
    for side = -1, 1, 2 do
        for row = 1, 3 do
            local z = -48 + row*42
            part("Coluna_interior", Vector3.new(8, 105, 8),
                CFrame.new(pos + Vector3.new(side*83, 73, z)), C.LightStone, Enum.Material.Marble, f, 0, true)
            part("Capitel_coluna", Vector3.new(13, 4, 13),
                CFrame.new(pos + Vector3.new(side*83, 128, z)), C.Gold, Enum.Material.Metal, f, 0, false)
        end
    end

    -- Salão de recepção: balcões laterais e assentos, sem obstruir o eixo central.
    for side = -1, 1, 2 do
        part("Balcao_recepcao_lateral", Vector3.new(34, 5, 12),
            CFrame.new(pos + Vector3.new(side*65, 23, -43)), C.Metal, Enum.Material.Metal, f, 0, true)
        for seat = 1, 3 do
            local z = -15 + seat*17
            part("Banco_salao_assento", Vector3.new(20, 3, 7),
                CFrame.new(pos + Vector3.new(side*67, 22, z)), C.DarkGreen, Enum.Material.Fabric, f, 0, true)
            part("Banco_salao_encosto", Vector3.new(20, 9, 2),
                CFrame.new(pos + Vector3.new(side*67, 27, z+3.5)), C.DarkGreen, Enum.Material.Fabric, f, 0, true)
        end
    end

    -- Núcleo simbólico ao fundo do salão, fora do corredor de entrada.
    cylinder("Pedestal_esfera_conselho", 42, 12,
        CFrame.new(pos + Vector3.new(0, 27, 65)), C.Metal, Enum.Material.Metal, f, true)
    local core = ball("Esfera_dos_guardioes", 27,
        CFrame.new(pos + Vector3.new(0, 66, 65)), C.Bright, Enum.Material.Neon, f, 0, false)
    makeLight(core, C.Bright, 2.2, 58)

    -- Sala de conferência fechada no fundo do complexo, com entrada ampla frontal.
    local conf = folder("Sala_de_conferencia", f)
    local cx, cz = 0, 57
    part("Piso_conferencia", Vector3.new(126, 2, 76),
        CFrame.new(pos + Vector3.new(cx, 22, cz)), C.DarkStone, Enum.Material.Marble, conf, 0, true)
    part("Parede_conferencia_fundo", Vector3.new(126, 54, 4),
        CFrame.new(pos + Vector3.new(cx, 49, cz+38)), C.Stone, Enum.Material.Concrete, conf, 0, true)
    part("Parede_conferencia_esquerda", Vector3.new(4, 54, 76),
        CFrame.new(pos + Vector3.new(cx-63, 49, cz)), C.Stone, Enum.Material.Concrete, conf, 0, true)
    part("Parede_conferencia_direita", Vector3.new(4, 54, 76),
        CFrame.new(pos + Vector3.new(cx+63, 49, cz)), C.Stone, Enum.Material.Concrete, conf, 0, true)
    -- Parede frontal dividida: porta central livre.
    part("Parede_conferencia_frontal_esq", Vector3.new(43, 54, 4),
        CFrame.new(pos + Vector3.new(-41.5, 49, cz-38)), C.Stone, Enum.Material.Concrete, conf, 0, true)
    part("Parede_conferencia_frontal_dir", Vector3.new(43, 54, 4),
        CFrame.new(pos + Vector3.new(41.5, 49, cz-38)), C.Stone, Enum.Material.Concrete, conf, 0, true)
    part("Viga_conferencia_porta", Vector3.new(40, 5, 4),
        CFrame.new(pos + Vector3.new(0, 73, cz-38)), C.Gold, Enum.Material.Metal, conf, 0, false)

    -- Teto fechado da sala de conferencia, com claraboia central.
    part("Teto_sala_conferencia", Vector3.new(130, 4, 80),
        CFrame.new(pos + Vector3.new(cx, 77, cz)), C.DarkStone, Enum.Material.Metal, conf, 0, true)
    part("Claraboia_sala_conferencia", Vector3.new(42, 1, 18),
        CFrame.new(pos + Vector3.new(cx, 79.6, cz)), C.Glass, Enum.Material.Glass, conf, 0.25, false)

    -- Mesa modular de conselho.
    part("Mesa_conferencia_centro", Vector3.new(66, 4, 22),
        CFrame.new(pos + Vector3.new(0, 25, cz+1)), C.Metal, Enum.Material.Metal, conf, 0, true)
    part("Superficie_mesa_conferencia", Vector3.new(64, 0.8, 20),
        CFrame.new(pos + Vector3.new(0, 27.4, cz+1)), C.DarkGreen, Enum.Material.Glass, conf, 0.12, false)
    local holo = neon("Projetor_holografico_conferencia", Vector3.new(15, 0.5, 10),
        CFrame.new(pos + Vector3.new(0, 31, cz+1)), C.Bright, conf)
    makeLight(holo, C.Bright, 0.8, 22)

    -- Cadeiras com assento, encosto e quatro pernas. 12 lugares ao redor da mesa.
    local function conferenceChair(x, z, facing)
        local chair = folder("Cadeira_conselho", conf)
        local base = pos + Vector3.new(x, 0, z)
        part("Assento_cadeira", Vector3.new(9, 2, 9), CFrame.new(base + Vector3.new(0, 27, 0)),
            C.DarkGreen, Enum.Material.Fabric, chair, 0, true)
        part("Encosto_cadeira", Vector3.new(9, 11, 2),
            CFrame.new(base + Vector3.new(0, 32.5, facing*4)), C.DarkGreen, Enum.Material.Fabric, chair, 0, true)
        for lx = -1, 1, 2 do
            for lz = -1, 1, 2 do
                part("Perna_cadeira", Vector3.new(1.2, 7, 1.2),
                    CFrame.new(base + Vector3.new(lx*3.4, 23.5, lz*3.4)), C.Silver, Enum.Material.Metal, chair, 0, true)
            end
        end
    end
    for i = -2, 2 do
        conferenceChair(i*13, cz-17, -1)
        conferenceChair(i*13, cz+19, 1)
    end
    conferenceChair(-47, cz+1, 0)
    conferenceChair(47, cz+1, 0)

    -- Luzes de teto e painéis acústicos decorativos.
    for i = -1, 1 do
        local panel = neon("Painel_luz_conferencia", Vector3.new(28, 0.8, 4),
            CFrame.new(pos + Vector3.new(i*36, 76, cz)), C.Bright, conf)
        makeLight(panel, C.Bright, 0.55, 30)
    end
    label(conf, "SALA DE CONFERENCIA | CONSELHO", pos + Vector3.new(0, 78, cz-41), 100, 14)

    -- Alas laterais administrativas com fachadas recuadas e portas desobstruidas.
    for side = -1, 1, 2 do
        local wing = folder(side < 0 and "Ala_administrativa" or "Ala_de_estrategia", f)
        part("Piso_ala", Vector3.new(62, 2, 112),
            CFrame.new(pos + Vector3.new(side*143, 22, 25)), C.Floor, Enum.Material.Marble, wing, 0, true)
        part("Parede_fundo_ala", Vector3.new(62, 48, 4),
            CFrame.new(pos + Vector3.new(side*143, 47, 81)), C.DarkStone, Enum.Material.Concrete, wing, 0, true)
        for edge = -1, 1, 2 do
            part("Parede_lateral_ala", Vector3.new(4, 48, 112),
                CFrame.new(pos + Vector3.new(side*143 + edge*31, 47, 25)), C.DarkStone, Enum.Material.Concrete, wing, 0, true)
        end
        part("Cobertura_ala", Vector3.new(66, 4, 116),
            CFrame.new(pos + Vector3.new(side*143, 73, 25)), C.DarkStone, Enum.Material.Metal, wing, 0, true)
        part("Console_ala", Vector3.new(34, 5, 8),
            CFrame.new(pos + Vector3.new(side*143, 25, 55)), C.Metal, Enum.Material.Metal, wing, 0, true)
        local screen = neon("Painel_ala", Vector3.new(25, 10, 1),
            CFrame.new(pos + Vector3.new(side*143, 34, 77)), C.Bright, wing)
        makeLight(screen, C.Bright, 0.45, 20)
    end

    -- Torre posterior e sinalizacao.
    local spireBase = pos + Vector3.new(0, 150, 104)
    cylinder("Pedestal_torre_citadel", 42, 16, CFrame.new(spireBase), C.DarkGreen, Enum.Material.Metal, f, true)
    cylinder("Torre_sinalizacao_citadel", 14, 76,
        CFrame.new(spireBase + Vector3.new(0, 44, 0)), C.Metal, Enum.Material.Metal, f, true)
    local spire = ball("Nucleo_sinalizador_citadel", 23,
        CFrame.new(spireBase + Vector3.new(0, 91, 0)), C.Bright, Enum.Material.Neon, f, 0, false)
    makeLight(spire, C.Bright, 1.8, 54)
    label(f, "CITADEL | CONSELHO DOS GUARDIOES", pos + Vector3.new(0, 145, -110), 150, 22)
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
        print("[OA V9.3] Arquivos centrais acessados.")
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
    print("[OA V9.3] Lockdown:", enabled and "ATIVADO" or "DESATIVADO")
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
                print("[OA V9.3] Prisao em lockdown.")
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
    -- Cobertura hibrida: telhado modular com patio tecnico central parcialmente aberto.
    local roofY = pos.Y + 51
    part("Telhado_modulo_esquerdo", Vector3.new(94, 5, 170),
        CFrame.new(pos + Vector3.new(-68, 51, 0)), C.DarkStone, Enum.Material.Metal, f, 0, true)
    part("Telhado_modulo_direito", Vector3.new(94, 5, 170),
        CFrame.new(pos + Vector3.new(68, 51, 0)), C.DarkStone, Enum.Material.Metal, f, 0, true)
    part("Telhado_frontal", Vector3.new(42, 5, 58),
        CFrame.new(pos + Vector3.new(0, 51, -56)), C.Metal, Enum.Material.Metal, f, 0, true)
    part("Telhado_traseiro", Vector3.new(42, 5, 58),
        CFrame.new(pos + Vector3.new(0, 51, 56)), C.Metal, Enum.Material.Metal, f, 0, true)
    -- Parapeitos perimetrais e faixa luminosa de identificacao.
    part("Parapeito_traseiro", Vector3.new(230, 5, 5),
        CFrame.new(pos + Vector3.new(0, 55, 82)), C.Silver, Enum.Material.Metal, f, 0, true)
    part("Parapeito_esquerdo", Vector3.new(5, 5, 170),
        CFrame.new(pos + Vector3.new(-115, 55, 0)), C.Silver, Enum.Material.Metal, f, 0, true)
    part("Parapeito_direito", Vector3.new(5, 5, 170),
        CFrame.new(pos + Vector3.new(115, 55, 0)), C.Silver, Enum.Material.Metal, f, 0, true)
    neon("Faixa_de_sinalizacao_telhado", Vector3.new(180, 1, 2),
        CFrame.new(pos + Vector3.new(0, 54, -82)), C.Bright, f)

    -- Quatro postos de observacao estilizados nos cantos.
    for sx = -1, 1, 2 do
        for sz = -1, 1, 2 do
            local towerPos = pos + Vector3.new(sx * 101, 61, sz * 70)
            part("Torre_observacao_base", Vector3.new(22, 12, 22),
                CFrame.new(towerPos), C.DarkGreen, Enum.Material.Metal, f, 0, true)
            part("Torre_observacao_corpo", Vector3.new(15, 18, 15),
                CFrame.new(towerPos + Vector3.new(0, 14, 0)), C.Metal, Enum.Material.Metal, f, 0, true)
            local beacon = neon("Torre_observacao_sinal", Vector3.new(9, 2, 9),
                CFrame.new(towerPos + Vector3.new(0, 24, 0)), C.Bright, f)
            makeLight(beacon, C.Bright, 0.8, 22)
        end
    end

    -- Equipamentos tecnicos no patio aberto central.
    for i = -1, 1, 2 do
        part("Antena_prisional", Vector3.new(2, 22, 2),
            CFrame.new(pos + Vector3.new(i * 14, 62, 0)), C.Silver, Enum.Material.Metal, f, 0, false)
        local antennaTip = neon("Ponta_antena", Vector3.new(4, 2, 4),
            CFrame.new(pos + Vector3.new(i * 14, 74, 0)), C.Bright, f)
        makeLight(antennaTip, C.Bright, 0.5, 16)
    end
    label(f, "OA | CONTENCAO", pos + Vector3.new(0, 82, -85), 90, 18)
end

--========================================================
-- BATERIA CENTRAL DA OA | INSPIRACAO VISUAL DOS QUADRINHOS
--========================================================
local function buildCentralBattery()
    local f = folder("06_BATERIA_CENTRAL")
    local p = BATTERY_POSITION

    cylinder("Plataforma_bateria", 190, 10, CFrame.new(p + Vector3.new(0, 5, 0)),
        C.DarkStone, Enum.Material.Metal, f, true)
    buildRing("Aro_externo_bateria", p, 91, 5, 11, C.Green, 56, f)
    buildRing("Aro_interno_bateria", p, 73, 3, 13, C.Bright, 48, f, Enum.Material.Neon)

    -- Base inferior alargada e garganta metalica sob a esfera.
    cylinder("Base_inferior_alargada", 88, 30,
        CFrame.new(p + Vector3.new(0, 27, 0)), C.Metal, Enum.Material.Metal, f, true)
    cylinder("Aro_base_verde", 94, 5,
        CFrame.new(p + Vector3.new(0, 13, 0)), C.DarkGreen, Enum.Material.Metal, f, true)
    cylinder("Colar_superior", 65, 18,
        CFrame.new(p + Vector3.new(0, 49, 0)), C.Silver, Enum.Material.Metal, f, true)

    -- Esfera externa verde com nucleo claro visivel na frente.
    local shell = ball("Carcaca_esferica", 116,
        CFrame.new(p + Vector3.new(0, 112, 0)), C.Green, Enum.Material.Metal, f, 0, false)
    shell.Reflectance = 0.08
    local core = ball("Nucleo_luminoso", 82,
        CFrame.new(p + Vector3.new(0, 112, -39)), C.White, Enum.Material.Neon, f, 0, false)
    batteryCoreLight = makeLight(core, Color3.fromRGB(190, 255, 190), 3.2, 95)
    local coreRim = cylinder("Aro_frontal_nucleo", 94, 5,
        CFrame.new(p + Vector3.new(0, 112, -40)) * CFrame.Angles(math.rad(90), 0, 0),
        C.DarkGreen, Enum.Material.Metal, f, false)
    coreRim.Transparency = 0.12

    -- Bracos laterais de sustentacao, semelhantes ao desenho de referencia.
    for side = -1, 1, 2 do
        local armPos = p + Vector3.new(side * 66, 105, 0)
        part("Braco_lateral_bateria", Vector3.new(9, 72, 9),
            CFrame.new(armPos) * CFrame.Angles(0, 0, math.rad(side * -18)),
            C.Silver, Enum.Material.Metal, f, 0, true)
        ball("Junta_lateral", 17, CFrame.new(p + Vector3.new(side * 59, 139, 0)),
            C.DarkGreen, Enum.Material.Metal, f, 0, false)
    end

    -- Gargalo superior e pequena chama geometrica verde (sem textura externa).
    cylinder("Gargalo_superior", 48, 24,
        CFrame.new(p + Vector3.new(0, 176, 0)), C.DarkGreen, Enum.Material.Metal, f, true)
    cylinder("Aro_gargalo", 54, 5,
        CFrame.new(p + Vector3.new(0, 190, 0)), C.Green, Enum.Material.Metal, f, false)
    for i = 1, 5 do
        local angle = (i - 1) * (math.pi * 2 / 5)
        local flamePos = p + Vector3.new(math.cos(angle) * 15, 207 + (i % 2) * 8, math.sin(angle) * 15)
        local flame = ball("Chama_energetica", 13 - (i % 2) * 3,
            CFrame.new(flamePos), C.Bright, Enum.Material.Neon, f, 0.12, false)
        if i == 1 then makeLight(flame, C.Bright, 1.1, 35) end
    end

    local statusPanel = neon("Painel_status_bateria", Vector3.new(38, 12, 2),
        CFrame.new(p + Vector3.new(0, 25, -52)), C.Bright, f)
    makeLight(statusPanel, C.Bright, 0.7, 25)

    -- Degraus alargados sob a plataforma.
    cylinder("Degrau_base_externo", 270, 4, CFrame.new(p + Vector3.new(0, 2, 0)),
        C.DarkStone, Enum.Material.Marble, f, true)
    cylinder("Degrau_base_interno", 235, 7, CFrame.new(p + Vector3.new(0, 3.5, 0)),
        C.Stone, Enum.Material.Marble, f, true)
    buildRing("Aro_dourado_degrau", p, 133, 3, 4.6, C.Gold, 64, f)

    -- Raios luminosos no piso da plataforma.
    for i = 1, 12 do
        local a = (i - 1) * math.pi / 6
        neon("Raio_da_plataforma", Vector3.new(70, 0.5, 2.4),
            CFrame.new(p + Vector3.new(math.cos(a) * 52, 10.3, math.sin(a) * 52)) * CFrame.Angles(0, -a, 0),
            C.Bright, f)
    end

    -- Postes de luz ao redor da plataforma.
    for i = 1, 8 do
        local a = (i - 1) * math.pi / 4 + math.pi / 8
        local base = p + Vector3.new(math.cos(a) * 106, 0, math.sin(a) * 106)
        part("Poste_da_bateria", Vector3.new(2.4, 20, 2.4), CFrame.new(base + Vector3.new(0, 17, 0)),
            C.Silver, Enum.Material.Metal, f, 0, true)
        local lamp = ball("Lampada_da_bateria", 7, CFrame.new(base + Vector3.new(0, 29, 0)),
            C.Bright, Enum.Material.Neon, f, 0, false)
        makeLight(lamp, C.Bright, 0.9, 34)
    end

    -- Cone de chama em camadas e coluna de energia ate o alto do globo.
    for i = 1, 6 do
        local layer = ball("Chama_em_camadas", 34 - i * 4.5,
            CFrame.new(p + Vector3.new(0, 214 + i * 13, 0)),
            i % 2 == 0 and C.White or C.Bright, Enum.Material.Neon, f, 0.15 + i * 0.05, false)
        layer.CastShadow = false
    end
    local column = cylinder("Coluna_de_energia", 22, 420,
        CFrame.new(p + Vector3.new(0, 440, 0)), C.Bright, Enum.Material.Neon, f, false)
    column.Transparency = 0.8
    column.CastShadow = false
    local columnCore = cylinder("Nucleo_coluna_de_energia", 8, 420,
        CFrame.new(p + Vector3.new(0, 440, 0)), C.White, Enum.Material.Neon, f, false)
    columnCore.Transparency = 0.55
    columnCore.CastShadow = false

    -- Fagulhas subindo da chama.
    local fire = part("Emissor_de_chama", Vector3.new(14, 2, 14),
        CFrame.new(p + Vector3.new(0, 210, 0)), C.Bright, Enum.Material.Neon, f, 1, false)
    fire.CastShadow = false
    local sparks = Instance.new("ParticleEmitter")
    sparks.Name = "Fagulhas_da_bateria"
    sparks.Texture = "rbxasset://textures/particles/sparkles_main.dds"
    sparks.Color = ColorSequence.new(Color3.fromRGB(235, 255, 225), Color3.fromRGB(70, 255, 130))
    sparks.LightEmission = 1
    sparks.Size = NumberSequence.new({NumberSequenceKeypoint.new(0, 5), NumberSequenceKeypoint.new(1, 0)})
    sparks.Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 0.1), NumberSequenceKeypoint.new(1, 1)})
    sparks.Lifetime = NumberRange.new(2, 4)
    sparks.Rate = 35
    sparks.Speed = NumberRange.new(10, 20)
    sparks.SpreadAngle = Vector2.new(20, 20)
    sparks.EmissionDirection = Enum.NormalId.Top
    sparks.Acceleration = Vector3.new(0, 6, 0)
    sparks.Parent = fire

    -- Esferas de energia orbitando a bateria (animadas em tempo real).
    for i = 1, 6 do
        local orb = ball("Orbe_orbital_bateria", 14, CFrame.new(p),
            i % 2 == 0 and C.White or C.Bright, Enum.Material.Neon, f, 0, false)
        orb.CastShadow = false
        if i % 2 == 0 then makeLight(orb, C.Bright, 0.8, 30) end
        batteryOrbs[#batteryOrbs + 1] = {
            part = orb,
            phase = i * math.pi / 3,
            speed = 0.55 + (i % 3) * 0.12,
            radius = 92 + (i % 2) * 16,
            tiltX = (i - 3.5) * 0.22,
            tiltZ = (i % 3 - 1) * 0.35,
            center = (BATTERY_POSITION - CENTER) + Vector3.new(0, 112, 0),
        }
    end

    label(f, "BATERIA CENTRAL | OA", p + Vector3.new(0, 222, -65), 110, 18)
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
-- NOVOS EQUIPAMENTOS CIVICOS: COMANDO E MUSEU
--========================================================
local function buildCivicFacilities()
    -- Centro de Comando no setor noroeste, com entrada frontal voltada ao sul.
    do
        local f = folder("11_CENTRO_COMANDO")
        local pos = CENTER + Vector3.new(-650, 0, -80)
        local b = buildBuilding("Centro_de_Comando", pos, 132, 116, 104, "citadel", f)
        part("Mesa_comando", Vector3.new(58, 4, 24),
            CFrame.new(pos + Vector3.new(0, 12, 10)), C.Metal, Enum.Material.Metal, b, 0, true)
        local holo = neon("Mesa_holografica_comando", Vector3.new(38, 1, 18),
            CFrame.new(pos + Vector3.new(0, 15, 10)), C.Bright, b)
        makeLight(holo, C.Bright, 1, 34)
        for side = -1, 1, 2 do
            local console = part("Console_operacional", Vector3.new(22, 5, 8),
                CFrame.new(pos + Vector3.new(side*42, 11, 28)), C.DarkGreen, Enum.Material.Metal, b, 0, true)
            local screen = neon("Tela_operacional", Vector3.new(18, 9, 1),
                CFrame.new(pos + Vector3.new(side*42, 20, 32)), C.Bright, b)
            makeLight(screen, C.Bright, 0.4, 18)
            part("Banco_operador", Vector3.new(9, 3, 8),
                CFrame.new(pos + Vector3.new(side*42, 9, 10)), C.DarkGreen, Enum.Material.Fabric, b, 0, true)
        end
        label(b, "CENTRO DE COMANDO", pos + Vector3.new(0, 111, -61), 88, 14)
        local terminal = ball("Terminal_de_comando", 9,
            CFrame.new(pos + Vector3.new(0, 18, -25)), C.Bright, Enum.Material.Neon, b, 0, false)
        addPrompt(terminal, "Acessar", "Centro de comando", function()
            print("[OA FINAL] Centro de comando acessado.")
        end)
    end

    -- Museu no setor nordeste, com vitrines baixas e percurso central desimpedido.
    do
        local f = folder("12_MUSEU_GUARDIOES")
        local pos = CENTER + Vector3.new(650, 0, -80)
        local b = buildBuilding("Museu_dos_Guardioes", pos, 144, 122, 88, "commercial", f)
        for i = 1, 4 do
            local x = -48 + (i-1)*32
            local plinth = part("Pedestal_exposicao", Vector3.new(18, 5, 18),
                CFrame.new(pos + Vector3.new(x, 10, 8)), C.DarkStone, Enum.Material.Metal, b, 0, true)
            local artifact = ball("Artefato_historico", 9,
                CFrame.new(pos + Vector3.new(x, 19, 8)), i%2==0 and C.Gold or C.Bright,
                Enum.Material.Neon, b, 0.08, false)
            makeLight(artifact, artifact.Color, 0.45, 16)
        end
        part("Balcao_museu", Vector3.new(34, 4, 7),
            CFrame.new(pos + Vector3.new(0, 10, 43)), C.Metal, Enum.Material.Metal, b, 0, true)
        for side = -1, 1, 2 do
            part("Banco_museu", Vector3.new(24, 3, 6),
                CFrame.new(pos + Vector3.new(side*43, 9, -35)), C.DarkGreen, Enum.Material.Fabric, b, 0, true)
        end
        label(b, "MUSEU DOS GUARDIOES", pos + Vector3.new(0, 95, -64), 96, 14)
        local archivePrompt = part("Totem_informativo_museu", Vector3.new(5, 12, 5),
            CFrame.new(pos + Vector3.new(0, 13, -43)), C.Metal, Enum.Material.Metal, b, 0, true)
        addPrompt(archivePrompt, "Explorar", "Museu dos Guardioes", function()
            print("[OA FINAL] Museu dos Guardioes acessado.")
        end)
    end
end

--========================================================
-- TORRES E SKYLINE
--========================================================
local function buildTower(position, height, index, parent)
    local f = folder("Torre_" .. index, parent)
    local profile = index % 3
    local baseWidth = profile == 0 and 38 or (profile == 1 and 32 or 28)
    local topWidth = profile == 0 and 24 or (profile == 1 and 27 or 20)

    cylinder("Plinto_circular", 68, 12, CFrame.new(position + Vector3.new(0, 6, 0)),
        C.DarkStone, Enum.Material.Metal, f, true)
    part("Base_estrutural", Vector3.new(baseWidth + 12, 12, baseWidth + 12),
        CFrame.new(position + Vector3.new(0, 12, 0)),
        C.DarkStone, Enum.Material.Concrete, f, 0, true)

    -- Corpo em dois trechos para criar uma silhueta de torre projetada.
    part("Corpo_inferior", Vector3.new(baseWidth, height * 0.58, baseWidth),
        CFrame.new(position + Vector3.new(0, 18 + height * 0.29, 0)),
        C.Stone, Enum.Material.Concrete, f, 0, true)
    part("Corpo_superior", Vector3.new(topWidth, height * 0.42, topWidth),
        CFrame.new(position + Vector3.new(0, 18 + height * 0.79, 0)),
        C.Metal, Enum.Material.Metal, f, 0, true)

    -- Fachadas envidracadas em cada face e antena de topo.
    for face = 0, 3 do
        local faceCF = CFrame.new(position) * CFrame.Angles(0, face * math.pi / 2, 0)
        local lowerGlass = part("Faixa_de_vidro_inferior", Vector3.new(baseWidth * 0.5, height * 0.46, 0.8),
            faceCF * CFrame.new(0, 18 + height * 0.29, -(baseWidth / 2 + 0.3)),
            C.Glass, Enum.Material.Glass, f, 0.35, false)
        lowerGlass.CastShadow = false
        local upperGlass = part("Faixa_de_vidro_superior", Vector3.new(topWidth * 0.5, height * 0.32, 0.8),
            faceCF * CFrame.new(0, 18 + height * 0.79, -(topWidth / 2 + 0.3)),
            C.Glass, Enum.Material.Glass, f, 0.3, false)
        upperGlass.CastShadow = false
    end
    cylinder("Antena_da_torre", 3, 34,
        CFrame.new(position + Vector3.new(0, height + 40, 0)), C.Silver, Enum.Material.Metal, f, false)
    local spireTip = ball("Ponta_da_antena", 6,
        CFrame.new(position + Vector3.new(0, height + 58, 0)), C.Bright, Enum.Material.Neon, f, 0, false)
    if index % 2 == 0 then makeLight(spireTip, C.Bright, 0.8, 30) end

    -- Quatro montantes externos revelam a logica estrutural.
    for i = 1, 4 do
        local a = (i - 1) * math.pi / 2 + math.pi / 4
        local radius = baseWidth * 0.52
        part("Montante_estrutural", Vector3.new(3.2, height + 10, 3.2),
            CFrame.new(position + Vector3.new(math.cos(a) * radius, height / 2 + 23,
                math.sin(a) * radius)),
            C.Silver, Enum.Material.Metal, f, 0, false)
    end

    -- Faixas de pavimento em intervalos regulares, com poucas luzes de destaque.
    for y = 32, height, 42 do
        buildRing("Faixa_tecnica", position, math.max(topWidth, baseWidth) * 0.53,
            1.4, y, C.Green, 20, f, Enum.Material.Metal)
    end

    -- Coroa tecnica mais contida, sem esfera luminosa gigante.
    part("Coroamento", Vector3.new(topWidth + 8, 5, topWidth + 8),
        CFrame.new(position + Vector3.new(0, height + 21, 0)),
        C.DarkStone, Enum.Material.Metal, f, 0, false)
    local beacon = neon("Farol_de_navegacao", Vector3.new(4, 5, 4),
        CFrame.new(position + Vector3.new(0, height + 26, 0)),
        C.Bright, f)
    if index % 3 == 0 then makeLight(beacon, C.Bright, 0.8, 28) end
end
local function buildSkyline()
    local f = folder("08_SKYLINE")
    for i = 1, 12 do
        local a = i * math.pi / 6
        local pos = CENTER + Vector3.new(math.cos(a) * 570, 0, math.sin(a) * 570)
        buildTower(pos, 105 + (i % 4) * 22, i, f)
    end
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
        print("[OA V9.3] Estacao de transporte acessada.")
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
        -- Pilares sob a passarela eliminam o aspecto de placas suspensas.
        for span = -1, 1 do
            local localOffset = Vector3.new(0, -9, span * 58)
            local supportPos = p + CFrame.Angles(0, -a, 0):VectorToWorldSpace(localOffset)
            part("Pilar_sustentacao_passarela", Vector3.new(5, 18, 5),
                CFrame.new(supportPos), C.DarkStone, Enum.Material.Concrete, f, 0, true)
            part("Base_pilar_passarela", Vector3.new(10, 2, 10),
                CFrame.new(supportPos - Vector3.new(0, 9, 0)), C.Metal, Enum.Material.Metal, f, 0, true)
        end
        for side = -1, 1, 2 do
            part("Guarda_corpo", Vector3.new(2, 8, 180),
                CFrame.new(p + Vector3.new(side * 10, 5, 0)) * CFrame.Angles(0, -a, 0),
                C.Silver, Enum.Material.Metal, f, 0, true)
        end
    end
end

local function buildOrbitalHangar()
    local f = folder("09B_HANGAR_ORBITAL")
    local pos = CENTER + Vector3.new(650, 0, 500)
    -- Hangar elevado de perfil baixo, conectado ao distrito por uma passarela.
    cylinder("Hangar_plataforma", 250, 16, CFrame.new(pos + Vector3.new(0, 8, 0)), C.DarkStone, Enum.Material.Metal, f, true)
    buildRing("Hangar_anel_externo", pos, 116, 5, 18, C.Bright, 64, f, Enum.Material.Neon)
    part("Hangar_piso", Vector3.new(210, 5, 170), CFrame.new(pos + Vector3.new(0, 20, 0)), C.Metal, Enum.Material.Metal, f, 0, true)
    -- Arcos laterais sugerem uma cobertura de doca sem fechar o interior.
    for side = -1, 1, 2 do
        part("Hangar_coluna_lateral", Vector3.new(12, 82, 12), CFrame.new(pos + Vector3.new(side * 94, 61, -8)), C.Silver, Enum.Material.Metal, f, 0, true)
        part("Hangar_viga_superior", Vector3.new(12, 12, 180), CFrame.new(pos + Vector3.new(side * 94, 104, 0)), C.DarkGreen, Enum.Material.Metal, f, 0, true)
        local strip = neon("Hangar_faixa_luminosa", Vector3.new(2, 2, 160), CFrame.new(pos + Vector3.new(side * 87, 96, 0)), C.Bright, f)
        makeLight(strip, C.Bright, 0.7, 32)
    end
    part("Hangar_trave_traseira", Vector3.new(200, 12, 12), CFrame.new(pos + Vector3.new(0, 104, 82)), C.Silver, Enum.Material.Metal, f, 0, true)
    for i = -2, 2 do
        local marker = neon("Hangar_guia_de_pouso", Vector3.new(5, 0.6, 26), CFrame.new(pos + Vector3.new(i * 28, 23, 0)), C.Green, f)
        if i == 0 then makeLight(marker, C.Bright, 1.2, 42) end
    end
    local beacon = ball("Hangar_sinalizador", 20, CFrame.new(pos + Vector3.new(0, 119, 82)), C.Bright, Enum.Material.Neon, f, 0, false)
    makeLight(beacon, C.Bright, 2, 70)
    label(f, "OA | HANGAR ORBITAL", pos + Vector3.new(0, 128, -104), 130, 22)
    local walkwayStart = CENTER + Vector3.new(430, 16, 390)
    local walkwayEnd = pos + Vector3.new(-90, 16, 0)
    beam("Passarela_ao_hangar", walkwayStart, walkwayEnd, 12, C.Metal, f)
    beam("Guia_luminosa_passarela", walkwayStart + Vector3.new(0, 3, 0), walkwayEnd + Vector3.new(0, 3, 0), 2, C.Bright, f)
    local bridgeDelta = walkwayEnd - walkwayStart
    for i = 1, 3 do
        local supportPos = walkwayStart + bridgeDelta * (i / 4) - Vector3.new(0, 8, 0)
        part("Pilar_passarela_hangar", Vector3.new(7, 16, 7),
            CFrame.new(supportPos), C.DarkStone, Enum.Material.Concrete, f, 0, true)
    end
end

local function buildHolographicMap()
    local f = folder("02B_MAPA_HOLOGRAFICO")
    local pos = CENTER + Vector3.new(-610, 0, 470)
    cylinder("Base_mapa_holografico", 126, 12, CFrame.new(pos + Vector3.new(0, 6, 0)), C.DarkStone, Enum.Material.Metal, f, true)
    cylinder("Aro_base_mapa", 112, 3, CFrame.new(pos + Vector3.new(0, 13, 0)), C.Bright, Enum.Material.Neon, f, false)
    local tableTop = part("Mesa_holografica", Vector3.new(104, 8, 104), CFrame.new(pos + Vector3.new(0, 20, 0)), C.Metal, Enum.Material.Metal, f, 0, true)
    local mapDisc = cylinder("Projecao_mapa_OA", 82, 2, CFrame.new(pos + Vector3.new(0, 28, 0)), C.Green, Enum.Material.Neon, f, false)
    mapDisc.Transparency = 0.48
    -- Miniatura esquematica em escala reduzida: anel, centro e marcadores dos setores.
    buildRing("Mapa_linha_externa", pos + Vector3.new(0, 30, 0), 36, 1.4, 0, C.Bright, 40, f, Enum.Material.Neon)
    buildRing("Mapa_linha_interna", pos + Vector3.new(0, 30, 0), 20, 1, 0, C.White, 32, f, Enum.Material.Neon)
    local centerNode = ball("Mapa_nucleo_central", 8, CFrame.new(pos + Vector3.new(0, 34, 0)), C.White, Enum.Material.Neon, f, 0, false)
    makeLight(centerNode, C.Bright, 1.3, 34)
    local mapPoints = {
        Vector3.new(0, 0, -29), Vector3.new(24, 0, -13), Vector3.new(-25, 0, -10),
        Vector3.new(-24, 0, 18), Vector3.new(0, 0, 30), Vector3.new(25, 0, 17),
    }
    for i, offset in ipairs(mapPoints) do
        local marker = ball("Marcador_setor_mapa_" .. i, 5, CFrame.new(pos + Vector3.new(offset.X, 34, offset.Z)), i == 1 and C.Gold or C.Bright, Enum.Material.Neon, f, 0, false)
        if i == 1 or i == 3 then makeLight(marker, C.Bright, 0.45, 18) end
    end
    for side = -1, 1, 2 do
        local support = part("Suporte_mesa_holografica", Vector3.new(8, 28, 8), CFrame.new(pos + Vector3.new(side * 34, 14, 0)), C.Silver, Enum.Material.Metal, f, 0, true)
        support.CastShadow = false
    end
    label(f, "OA | MAPA HOLOGRAFICO", pos + Vector3.new(0, 52, -65), 145, 22)
    makeLight(tableTop, C.Green, 0.45, 30)
end

local function buildUrbanDetails()
    local f = folder("10_DETALHES_URBANOS")
    -- Postes no anel externo: luminaria real com PointLight, nao apenas Neon.
    for i = 1, 24 do
        local a = i * math.pi / 12
        local pos = CENTER + Vector3.new(math.cos(a) * 480, 0, math.sin(a) * 480)
        part("Poste", Vector3.new(2.4, 22, 2.4), CFrame.new(pos + Vector3.new(0, 11, 0)),
            C.Metal, Enum.Material.Metal, f, 0, true)
        part("Braco", Vector3.new(9, 1.5, 1.5), CFrame.new(pos + Vector3.new(4, 21, 0)),
            C.Silver, Enum.Material.Metal, f, 0, false)
        local lamp = neon("Luminaria", Vector3.new(5, 1.4, 4),
            CFrame.new(pos + Vector3.new(8, 20, 0)), C.Bright, f)
        makeLight(lamp, Color3.fromRGB(155, 255, 190), 1.25, 38)
    end

    -- Segundo circuito de postes menores para iluminar as avenidas internas.
    for i = 1, 12 do
        local a = i * math.pi / 6
        local pos = CENTER + Vector3.new(math.cos(a) * 300, 0, math.sin(a) * 300)
        part("Poste_interno", Vector3.new(1.8, 16, 1.8),
            CFrame.new(pos + Vector3.new(0, 8, 0)), C.DarkStone, Enum.Material.Metal, f, 0, true)
        local lamp = neon("Luminaria_interna", Vector3.new(4, 1.2, 3),
            CFrame.new(pos + Vector3.new(0, 16, 0)), C.Bright, f)
        makeLight(lamp, Color3.fromRGB(170, 255, 195), 0.9, 30)
    end

    for i = 1, 16 do
        local a = i * math.pi / 8
        local pos = CENTER + Vector3.new(math.cos(a) * 165, 0, math.sin(a) * 165)
        neon("Balizador", Vector3.new(2, 7, 2),
            CFrame.new(pos + Vector3.new(0, 5, 0)), C.Bright, f)
    end

    -- Mobiliario urbano discreto, sempre apoiado no piso.
    for i = 1, 8 do
        local a = i * math.pi / 4
        local pos = CENTER + Vector3.new(math.cos(a) * 250, 0, math.sin(a) * 250)
        local yaw = -a
        part("Banco_avenida", Vector3.new(18, 2.5, 5),
            CFrame.new(pos + Vector3.new(0, 3, 0)) * CFrame.Angles(0, yaw, 0),
            C.DarkGreen, Enum.Material.Wood, f, 0, true)
        part("Lixeira_urbana", Vector3.new(4, 8, 4),
            CFrame.new(pos + Vector3.new(15, 4, 0)),
            C.Metal, Enum.Material.Metal, f, 0, true)
    end
end

--========================================================
-- RECONSTRUCAO TRANSACIONAL
--========================================================
--========================================================
-- ATMOSFERA INTERNA: PARTICULAS E FEIXES DE LUZ
--========================================================
local function buildInteriorAtmosphere()
    local f = folder("01D_ATMOSFERA_INTERNA")
    local volume = part("Volume_de_particulas", Vector3.new(1100, 260, 1100),
        CFrame.new(CENTER + Vector3.new(0, 150, 0)), C.Bright, Enum.Material.SmoothPlastic, f, 1, false)
    volume.CastShadow = false

    local motes = Instance.new("ParticleEmitter")
    motes.Name = "Particulas_de_energia"
    motes.Texture = "rbxasset://textures/particles/sparkles_main.dds"
    motes.Color = ColorSequence.new(Color3.fromRGB(120, 255, 170), Color3.fromRGB(200, 255, 120))
    motes.LightEmission = 1
    motes.Size = NumberSequence.new({NumberSequenceKeypoint.new(0, 0),
        NumberSequenceKeypoint.new(0.5, 2.2), NumberSequenceKeypoint.new(1, 0)})
    motes.Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 1),
        NumberSequenceKeypoint.new(0.2, 0.25), NumberSequenceKeypoint.new(0.8, 0.25),
        NumberSequenceKeypoint.new(1, 1)})
    motes.Lifetime = NumberRange.new(6, 11)
    motes.Rate = 40
    motes.Speed = NumberRange.new(1, 4)
    motes.SpreadAngle = Vector2.new(180, 180)
    motes.Acceleration = Vector3.new(0, 1.2, 0)
    motes.RotSpeed = NumberRange.new(-30, 30)
    motes.ShapeStyle = Enum.ParticleEmitterShapeStyle.Volume
    motes.Parent = volume

    local pollen = Instance.new("ParticleEmitter")
    pollen.Name = "Poeira_luminosa"
    pollen.Texture = "rbxasset://textures/particles/sparkles_main.dds"
    pollen.Color = ColorSequence.new(Color3.fromRGB(235, 255, 235))
    pollen.LightEmission = 0.6
    pollen.Size = NumberSequence.new(1.1)
    pollen.Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 1),
        NumberSequenceKeypoint.new(0.5, 0.5), NumberSequenceKeypoint.new(1, 1)})
    pollen.Lifetime = NumberRange.new(10, 16)
    pollen.Rate = 25
    pollen.Speed = NumberRange.new(0.5, 2)
    pollen.SpreadAngle = Vector2.new(180, 180)
    pollen.ShapeStyle = Enum.ParticleEmitterShapeStyle.Volume
    pollen.Parent = volume

    -- Feixes de luz verticais, bem translucidos, em volta da cidade.
    for i = 1, 8 do
        local a = (i - 1) * math.pi / 4 + math.pi / 8
        local radius = i % 2 == 0 and 330 or 470
        local shaft = cylinder("Feixe_de_luz", 38, 520,
            CFrame.new(CENTER + Vector3.new(math.cos(a) * radius, 260, math.sin(a) * radius)),
            C.Bright, Enum.Material.Neon, f, false)
        shaft.Transparency = 0.94
        shaft.CastShadow = false
    end
end


--========================================================
-- OA FINAL | FUNDACAO, ORBITA E SUPERFICIE EXPANDIDAS
--========================================================
local function buildUnderCityMegaStructure()
    local f = folder("02A_FUNDACAO_COLOSSAL_DA_CIDADE")

    -- Camadas escalonadas sob a cidade: casco, anel tecnico e quilha.
    cylinder("Casco_inferior_escalonado", 1160, 22,
        CFrame.new(CENTER + Vector3.new(0, -66, 0)), C.DarkStone, Enum.Material.Metal, f, false)
    cylinder("Disco_tecnico_inferior", 930, 14,
        CFrame.new(CENTER + Vector3.new(0, -96, 0)), C.Metal, Enum.Material.Metal, f, false)
    cylinder("Plataforma_de_distribuicao", 690, 16,
        CFrame.new(CENTER + Vector3.new(0, -128, 0)), C.DarkGreen, Enum.Material.Metal, f, false)
    cylinder("Quilha_central", 280, 20,
        CFrame.new(CENTER + Vector3.new(0, -166, 0)), C.DarkStone, Enum.Material.Metal, f, false)

    local levels = {
        {r=560, y=-82, t=8, c=C.Gold},
        {r=430, y=-184, t=7, c=C.Silver},
        {r=300, y=-286, t=6, c=C.Green},
        {r=160, y=-390, t=5, c=C.Bright},
    }
    for i, v in ipairs(levels) do
        buildRing("Trelica_inferior_" .. i, CENTER, v.r, v.t, v.y, v.c, 64, f,
            i == 4 and Enum.Material.Neon or Enum.Material.Metal)
    end

    -- Nervuras inclinadas e contraventamentos radiais.
    for i = 1, 16 do
        local a = (i - 1) * math.pi * 2 / 16
        local b = a + math.pi / 16
        local p1 = CENTER + Vector3.new(math.cos(a)*560, -82, math.sin(a)*560)
        local p2 = CENTER + Vector3.new(math.cos(b)*430, -184, math.sin(b)*430)
        local p3 = CENTER + Vector3.new(math.cos(a)*300, -286, math.sin(a)*300)
        local p4 = CENTER + Vector3.new(math.cos(b)*160, -390, math.sin(b)*160)
        beam("Nervura_estrutural_externa", p1, p2, 12, C.DarkStone, f)
        beam("Nervura_estrutural_media", p2, p3, 9, C.Silver, f)
        beam("Nervura_estrutural_interna", p3, p4, 7, C.Metal, f)
        beam("Contravento_radial", p1, p3, 4, C.Gold, f)
        beam("Contravento_interno", p2, p4, 4, C.Green, f)
    end

    -- Doze pilares diagonais e docas técnicas suspensas.
    for i = 1, 12 do
        local a = (i - 1) * math.pi * 2 / 12
        local upper = CENTER + Vector3.new(math.cos(a)*610, -62, math.sin(a)*610)
        local lower = CENTER + Vector3.new(math.cos(a)*245, -330, math.sin(a)*245)
        beam("Pilar_colossal_inclinado", upper, lower, 23, C.DarkGreen, f)
        ball("Junta_mecanica_colossal", 28, CFrame.new(lower), C.Gold,
            Enum.Material.Metal, f, 0, false)
        local lamp = neon("Sinalizador_de_carga", Vector3.new(6, 18, 6),
            CFrame.new(lower + Vector3.new(0, 20, 0)), C.Bright, f)
        if i % 2 == 0 then makeLight(lamp, C.Bright, 0.5, 28) end
    end

    for i = 1, 8 do
        local a = (i - 1) * math.pi / 4
        local pos = CENTER + Vector3.new(math.cos(a)*440, -224, math.sin(a)*440)
        cylinder("Doca_de_manutencao", 70, 18, CFrame.new(pos),
            C.DarkStone, Enum.Material.Metal, f, false)
        cylinder("Aro_luminoso_doca", 78, 4, CFrame.new(pos + Vector3.new(0, 12, 0)),
            C.Bright, Enum.Material.Neon, f, false)
        part("Modulo_de_servico", Vector3.new(62, 34, 82),
            CFrame.new(pos + Vector3.new(0, -28, 0)), C.Metal, Enum.Material.Metal, f, true)
    end

    cylinder("Coluna_do_reator_inferior", 110, 170,
        CFrame.new(CENTER + Vector3.new(0, -390, 0)), C.DarkStone, Enum.Material.Metal, f, false)
    cylinder("Aro_do_reator_inferior", 170, 10,
        CFrame.new(CENTER + Vector3.new(0, -472, 0)), C.Gold, Enum.Material.Metal, f, false)
    local core = ball("Nucleo_do_reator_inferior", 76,
        CFrame.new(CENTER + Vector3.new(0, -520, 0)), C.Bright, Enum.Material.Neon, f, 0, false)
    makeLight(core, C.Bright, 1.5, 100)
end

local function buildOrbitalInfrastructure()
    local f = folder("01E_INFRAESTRUTURA_ORBITAL")
    local radius = GLOBE_RADIUS * 1.52

    -- Tres orbitas inclinadas em torno de OA.
    for ringIndex = 1, 3 do
        local tilt = math.rad((ringIndex - 2) * 22)
        local r = radius + (ringIndex - 2) * 105
        local previous = nil
        for i = 0, 63 do
            local a = i * math.pi * 2 / 64
            local point = CENTER + Vector3.new(math.cos(a)*r,
                math.sin(a)*r*math.sin(tilt), math.sin(a)*r*math.cos(tilt))
            if previous then
                local segment = beam("Segmento_anel_orbital", previous, point,
                    ringIndex == 2 and 5 or 3,
                    ringIndex == 2 and C.Bright or C.Gold, f)
                if segment then segment.Material = Enum.Material.Neon end
            end
            previous = point
        end
    end

    -- Quatro estações de vigilância e dez satélites decorativos.
    for i = 1, 4 do
        local a = (i - 1) * math.pi / 2 + math.pi / 4
        local direction = Vector3.new(math.cos(a), (i % 2 == 0) and 0.18 or -0.18, math.sin(a)).Unit
        local pos = CENTER + direction * (GLOBE_RADIUS * 1.72)
        ball("Nucleo_estacao_orbital", 72, CFrame.new(pos), C.DarkGreen,
            Enum.Material.Metal, f, 0, false)
        cylinder("Aro_estacao_orbital", 126, 8, CFrame.new(pos),
            C.Silver, Enum.Material.Metal, f, false)
        local beacon = neon("Farol_estacao", Vector3.new(18, 18, 18),
            CFrame.new(pos + direction * 48), C.Bright, f)
        makeLight(beacon, C.Bright, 1, 75)
        beam("Braco_estacao_orbital", pos, CENTER + direction * (GLOBE_RADIUS * 1.12),
            9, C.Metal, f)
    end

    for i = 1, 10 do
        local a = (i - 1) * math.pi * 2 / 10
        local y = (i % 2 == 0) and 250 or -250
        local pos = CENTER + Vector3.new(math.cos(a)*GLOBE_RADIUS*1.9, y,
            math.sin(a)*GLOBE_RADIUS*1.9)
        local body = part("Satelite_de_patrulha", Vector3.new(34, 16, 48),
            CFrame.lookAt(pos, CENTER), C.Silver, Enum.Material.Metal, f, 0, false)
        part("Painel_solar_esquerdo", Vector3.new(48, 2, 22),
            body.CFrame * CFrame.new(-42, 0, 0), C.DarkGreen, Enum.Material.Metal, f, false)
        part("Painel_solar_direito", Vector3.new(48, 2, 22),
            body.CFrame * CFrame.new(42, 0, 0), C.DarkGreen, Enum.Material.Metal, f, false)
    end
end

local function buildGlobeSurfaceEnhancements()
    local f = folder("01D_DETALHES_DE_SUPERFICIE")
    -- Pequenos marcadores em quatro latitudes reforçam a leitura de cartografia planetária.
    for band = 1, 4 do
        local latitude = math.rad(-48 + band * 24)
        local ringRadius = GLOBE_RADIUS * math.cos(latitude)
        local y = GLOBE_RADIUS * math.sin(latitude)
        for i = 1, 48 do
            local a = (i - 1) * math.pi * 2 / 48
            local pos = CENTER + Vector3.new(math.cos(a)*ringRadius, y, math.sin(a)*ringRadius)
            local marker = ball("Marco_cartografico", 5, CFrame.new(pos),
                C.Bright, Enum.Material.Neon, f, 0.35, false)
            marker.CastShadow = false
            tagOuter(marker, 0.35)
        end
    end

    -- Pequenos pontos de relevo visual espalhados na superfície esférica.
    local rng = Random.new(1942)
    for i = 1, 28 do
        local y = rng:NextNumber(-0.78, 0.78)
        local a = rng:NextNumber(0, math.pi * 2)
        local h = math.sqrt(1 - y*y)
        local direction = Vector3.new(math.cos(a)*h, y, math.sin(a)*h)
        local pos = CENTER + direction * (GLOBE_RADIUS + 2)
        local mark = ball("Marca_de_relevo", rng:NextNumber(55, 105),
            CFrame.lookAt(pos, pos + direction), C.Green, Enum.Material.SmoothPlastic, f, 0.4, false)
        mark.Size = Vector3.new(rng:NextNumber(75, 145), rng:NextNumber(35, 85), 3)
        mark.CastShadow = false
        tagOuter(mark, 0.4)
    end
end



--========================================================
-- SISTEMA ESTELAR DE OA | DOIS SOIS DISTANTES
--========================================================
local function buildTwinSuns()
    local f = folder("01F_SISTEMA_ESTELAR")

    local function createSun(name, offset, diameter, color, coronaColor, brightness, range)
        local center = CENTER + offset
        local sun = ball(name, diameter, CFrame.new(center), color, Enum.Material.Neon, f, 0, false)
        sun.CastShadow = false
        local corona = ball(name .. "_Corona", diameter * 1.24, CFrame.new(center),
            coronaColor, Enum.Material.Neon, f, 0.82, false)
        corona.CastShadow = false
        local halo = ball(name .. "_Halo", diameter * 1.62, CFrame.new(center),
            coronaColor, Enum.Material.Neon, f, 0.94, false)
        halo.CastShadow = false
        buildRing(name .. "_Anel_Equatorial", center, diameter * 0.78, 3, 0,
            coronaColor, 48, f, Enum.Material.Neon)
        buildRing(name .. "_Anel_Inclinado", center, diameter * 0.92, 2, diameter * 0.12,
            color, 48, f, Enum.Material.Neon)
        makeLight(sun, color, brightness, range)
    end

    createSun("Sol_Principal_de_Oa", Vector3.new(-4300, 1850, -5200), 520,
        Color3.fromRGB(255, 224, 145), Color3.fromRGB(255, 174, 85), 2.2, 180)
    createSun("Sol_Secundario_de_Oa", Vector3.new(6100, 2850, 4300), 350,
        Color3.fromRGB(190, 225, 255), Color3.fromRGB(105, 175, 255), 1.35, 135)

    local markers = {
        Vector3.new(-2600, 1050, -3150), Vector3.new(3600, 1750, 2500),
        Vector3.new(-1700, -450, -2050), Vector3.new(2500, 650, 1850),
    }
    for i, offset in ipairs(markers) do
        local beacon = ball("Baliza_de_navegacao_estelar_" .. i, 22,
            CFrame.new(CENTER + offset), i % 2 == 0 and C.Bright or C.Gold,
            Enum.Material.Neon, f, 0.08, false)
        beacon.CastShadow = false
        if i % 2 == 0 then makeLight(beacon, C.Bright, 0.55, 45) end
    end
end


local BUILD_STEPS = {
    {"Globo e atmosfera", buildGlobe},
    {"Estrutura inferior do planeta", buildPlanetaryUnderside},
    {"Campo de protecao visual", buildProtectionField},
    {"Identificador distante OA", buildDistantPlanetBeacon},
    {"Detalhes de superficie do globo", buildGlobeSurfaceEnhancements},
    {"Infraestrutura orbital", buildOrbitalInfrastructure},
    {"Sistema estelar de dois sois", buildTwinSuns},
    {"Centro planetario", buildCentralHub},
    {"Fundacao colossal da cidade", buildUnderCityMegaStructure},
    {"Citadel", buildCitadel},
    {"Arquivos centrais", buildArchives},
    {"Prisao e telhado hibrido", buildPrison},
    {"Bateria Central", buildCentralBattery},
    {"Jardins", buildGardens},
    {"Distritos", buildDistricts},
    {"Skyline", buildSkyline},
    {"Transporte", buildTransport},
    {"Hangar orbital", buildOrbitalHangar},
    {"Mapa holografico", buildHolographicMap},
    {"Centro de comando e museu", buildCivicFacilities},
    {"Atmosfera interna", buildInteriorAtmosphere},
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
    local previousOuter = OUTER_PARTS
    local previousOrbs = batteryOrbs
    local previousCoreLight = batteryCoreLight
    OUTER_PARTS = {}
    batteryOrbs = {}
    batteryCoreLight = nil
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
        local previousMinorPatchMap = workspace:FindFirstChild(PREVIOUS_MINOR_PATCH_MAP_NAME)
        if previousMinorPatchMap and previousMinorPatchMap.Parent then previousMinorPatchMap:Destroy() end
        local previousMinorPatch3Map = workspace:FindFirstChild(PREVIOUS_MINOR_PATCH_3_MAP_NAME)
        if previousMinorPatch3Map and previousMinorPatch3Map.Parent then previousMinorPatch3Map:Destroy() end
        local previousMinorPatchOldMap = workspace:FindFirstChild(PREVIOUS_MINOR_PATCH_OLD_MAP_NAME)
        if previousMinorPatchOldMap and previousMinorPatchOldMap.Parent then previousMinorPatchOldMap:Destroy() end
        local previousMinorPatch2Map = workspace:FindFirstChild(PREVIOUS_MINOR_PATCH_2_MAP_NAME)
        if previousMinorPatch2Map and previousMinorPatch2Map.Parent then previousMinorPatch2Map:Destroy() end
        local previousPatchMap = workspace:FindFirstChild(PREVIOUS_PATCH_MAP_NAME)
        if previousPatchMap and previousPatchMap.Parent then previousPatchMap:Destroy() end
        local previousVersionMap = workspace:FindFirstChild(PREVIOUS_MAP_NAME)
        if previousVersionMap and previousVersionMap.Parent then previousVersionMap:Destroy() end
        local previousMinorMap = workspace:FindFirstChild(PREVIOUS_MINOR_MAP_NAME)
        if previousMinorMap and previousMinorMap.Parent then previousMinorMap:Destroy() end
        local olderMap = workspace:FindFirstChild(OLDER_MAP_NAME)
        if olderMap and olderMap.Parent then olderMap:Destroy() end
        Map = staging
        lastAppliedFade = -1 -- forca reaplicar a transparencia da pele externa
        BUILDING = false
        return true, string.format("OA V%s pronta | %d pecas | %d edificios.",
            VERSION, PART_COUNT, BUILDING_COUNT)
    end

    -- Rollback: preserva a versao anterior se a construcao falhar.
    if staging and staging.Parent then staging:Destroy() end
    Map = oldMap or previousMap
    OUTER_PARTS = previousOuter
    batteryOrbs = previousOrbs
    batteryCoreLight = previousCoreLight
    CELL_DOORS = {}
    DOOR_STATES = {}
    LOCKDOWN = false
    BUILDING = false
    warn("[OA FINAL] Reconstrucao cancelada; rollback executado:\n" .. tostring(err))
    return false, tostring(err)
end

--========================================================
-- INTERFACE
--========================================================
local oldGui = PlayerGui:FindFirstChild(GUI_NAME)
if oldGui then oldGui:Destroy() end
local legacyGui = PlayerGui:FindFirstChild(LEGACY_GUI_NAME)
if legacyGui then legacyGui:Destroy() end
local previousMinorPatchGui = PlayerGui:FindFirstChild(PREVIOUS_MINOR_PATCH_GUI_NAME)
if previousMinorPatchGui then previousMinorPatchGui:Destroy() end
local previousMinorPatch3Gui = PlayerGui:FindFirstChild(PREVIOUS_MINOR_PATCH_3_GUI_NAME)
if previousMinorPatch3Gui then previousMinorPatch3Gui:Destroy() end
local previousMinorPatchOldGui = PlayerGui:FindFirstChild(PREVIOUS_MINOR_PATCH_OLD_GUI_NAME)
if previousMinorPatchOldGui then previousMinorPatchOldGui:Destroy() end
local previousMinorPatch2Gui = PlayerGui:FindFirstChild(PREVIOUS_MINOR_PATCH_2_GUI_NAME)
if previousMinorPatch2Gui then previousMinorPatch2Gui:Destroy() end
local previousGui = PlayerGui:FindFirstChild(PREVIOUS_GUI_NAME)
if previousGui then previousGui:Destroy() end
local previousPatchGui = PlayerGui:FindFirstChild(PREVIOUS_PATCH_GUI_NAME)
if previousPatchGui then previousPatchGui:Destroy() end
local previousMinorGui = PlayerGui:FindFirstChild(PREVIOUS_MINOR_GUI_NAME)
if previousMinorGui then previousMinorGui:Destroy() end
local olderGui = PlayerGui:FindFirstChild(OLDER_GUI_NAME)
if olderGui then olderGui:Destroy() end

local GUI = Instance.new("ScreenGui")
GUI.Name = GUI_NAME
GUI.ResetOnSpawn = false
GUI.IgnoreGuiInset = true
GUI.DisplayOrder = 20
GUI.Parent = PlayerGui

local Main = Instance.new("Frame")
Main.Name = "Painel"
Main.Size = UDim2.new(0, 320, 0, 430)
Main.Position = UDim2.new(0, 12, 0, 12)
Main.BackgroundColor3 = Color3.fromRGB(10, 23, 19)
Main.BackgroundTransparency = 0.05
Main.BorderSizePixel = 0
Main.Active = true
Main.Parent = GUI

local MainScale = Instance.new("UIScale")
MainScale.Scale = 1
MainScale.Parent = Main

local function updateResponsiveLayout()
    local camera = workspace.CurrentCamera
    if not camera then return end
    local viewport = camera.ViewportSize
    local widthScale = math.max(0.72, math.min(1, (viewport.X - 24) / 320))
    local heightScale = math.max(0.72, math.min(1, (viewport.Y - 24) / 430))
    MainScale.Scale = math.min(widthScale, heightScale)
    Main.Position = UDim2.new(0, 12, 0, 12)
end

updateResponsiveLayout()
if workspace.CurrentCamera then
    runtimeConnections[#runtimeConnections + 1] = workspace.CurrentCamera:GetPropertyChangedSignal("ViewportSize"):Connect(updateResponsiveLayout)
end

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
Minimize.Size = UDim2.new(0, 40, 0, 38)
Minimize.Position = UDim2.new(1, -47, 0, 6)
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
Status.Text = "OA FINAL | Inicializando"
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

-- Navegacao local: destinos fixos conhecidos da construcao V9.3.
local GPS_DESTINATIONS = {
    {name = "OA | Centro do planeta", position = CENTER + Vector3.new(0, 10, 0)},
    {name = "OA-01 | Bateria Central", position = BATTERY_POSITION + Vector3.new(0, 112, 0)},
    {name = "OA-02 | Prisao", position = PRISON_POSITION + Vector3.new(0, 10, 0)},
    {name = "OA-03 | Arquivos centrais", position = CENTER + Vector3.new(-370, 12, -180)},
    {name = "OA-04 | Citadel", position = CENTER + Vector3.new(0, 12, -440)},
}

local GPSPanel = Instance.new("Frame")
GPSPanel.Name = "Navegacao_OA"
GPSPanel.Size = UDim2.new(1, -4, 0, 98)
GPSPanel.BackgroundColor3 = Color3.fromRGB(15, 52, 37)
GPSPanel.BorderSizePixel = 0
GPSPanel.LayoutOrder = 1
GPSPanel.Parent = Content
local gpsCorner = Instance.new("UICorner")
gpsCorner.CornerRadius = UDim.new(0, 8)
gpsCorner.Parent = GPSPanel
local GPSName = Instance.new("TextLabel")
GPSName.Size = UDim2.new(1, -12, 0, 22)
GPSName.Position = UDim2.new(0, 8, 0, 5)
GPSName.BackgroundTransparency = 1
GPSName.Text = "OA | Centro do planeta"
GPSName.TextColor3 = C.Bright
GPSName.TextSize = 12
GPSName.Font = Enum.Font.GothamBold
GPSName.TextXAlignment = Enum.TextXAlignment.Left
GPSName.Parent = GPSPanel
local GPSReadout = Instance.new("TextLabel")
GPSReadout.Size = UDim2.new(1, -12, 0, 22)
GPSReadout.Position = UDim2.new(0, 8, 0, 29)
GPSReadout.BackgroundTransparency = 1
GPSReadout.Text = "Distancia ate a OA: calculando..."
GPSReadout.TextColor3 = C.White
GPSReadout.TextSize = 11
GPSReadout.Font = Enum.Font.Gotham
GPSReadout.TextXAlignment = Enum.TextXAlignment.Left
GPSReadout.Parent = GPSPanel
local GPSCoords = Instance.new("TextLabel")
GPSCoords.Size = UDim2.new(1, -96, 0, 20)
GPSCoords.Position = UDim2.new(0, 8, 0, 52)
GPSCoords.BackgroundTransparency = 1
GPSCoords.Text = "XYZ (studs): --"
GPSCoords.TextColor3 = C.LightStone
GPSCoords.TextSize = 10
GPSCoords.Font = Enum.Font.Code
GPSCoords.TextXAlignment = Enum.TextXAlignment.Left
GPSCoords.Parent = GPSPanel
local GPSNext = Instance.new("TextButton")
GPSNext.Size = UDim2.new(0, 74, 0, 22)
GPSNext.Position = UDim2.new(1, -82, 1, -27)
GPSNext.BackgroundColor3 = C.DarkGreen
GPSNext.Text = "LOCAL >"
GPSNext.TextColor3 = C.White
GPSNext.TextSize = 9
GPSNext.Font = Enum.Font.GothamBold
GPSNext.Parent = GPSPanel
GPSNext.Activated:Connect(function()
    gpsDestinationIndex = (gpsDestinationIndex % #GPS_DESTINATIONS) + 1
    GPSName.Text = GPS_DESTINATIONS[gpsDestinationIndex].name
end)

local function makeButton(text, order, callback)
    local button = Instance.new("TextButton")
    button.Name = text
    button.Size = UDim2.new(1, -4, 0, 46)
    button.LayoutOrder = order + 1
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

local function focusOACamera()
    local camera = workspace.CurrentCamera
    if not camera then
        setStatus("Camera indisponivel.")
        return
    end
    camera.CameraType = Enum.CameraType.Scriptable
    local cameraPosition = CENTER + Vector3.new(0, 520, 2350)
    camera.CFrame = CFrame.lookAt(cameraPosition, CENTER)
    camera.Focus = CFrame.new(CENTER)
    setStatus("Camera focada no planeta OA.")
end

local function restorePlayerCamera()
    local camera = workspace.CurrentCamera
    local character = Player.Character
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    if not camera then
        setStatus("Camera indisponivel.")
        return
    end
    camera.CameraType = Enum.CameraType.Custom
    if humanoid then camera.CameraSubject = humanoid end
    setStatus("Camera do jogador restaurada.")
end

makeButton("FOCAR CAMERA EM OA", 1, focusOACamera)

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
    local previousPatchMinor = workspace:FindFirstChild(PREVIOUS_MINOR_PATCH_MAP_NAME)
    if previousPatchMinor then previousPatchMinor:Destroy() end
    local previousPatchMinor2 = workspace:FindFirstChild(PREVIOUS_MINOR_PATCH_2_MAP_NAME)
    if previousPatchMinor2 then previousPatchMinor2:Destroy() end
    local previous = workspace:FindFirstChild(PREVIOUS_MAP_NAME)
    if previous then previous:Destroy() end
    local previousMinor = workspace:FindFirstChild(PREVIOUS_MINOR_MAP_NAME)
    if previousMinor then previousMinor:Destroy() end
    local older = workspace:FindFirstChild(OLDER_MAP_NAME)
    if older then older:Destroy() end
    Map = nil
    OUTER_PARTS = {}
    batteryOrbs = {}
    batteryCoreLight = nil
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

makeButton("RESTAURAR CAMERA", 6, restorePlayerCamera)

-- Atalho local para atravessar a casca e entrar no planeta.
makeButton("ENTRAR EM OA", 7, function()
    if BUILDING then setStatus("Aguarde a construcao.") return end
    if not Map or not Map.Parent then setStatus("Construa OA primeiro.") return end
    local character = Player.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")
    if not root then setStatus("Personagem indisponivel.") return end
    root.AssemblyLinearVelocity = Vector3.zero
    root.CFrame = CFrame.new(CENTER + Vector3.new(0, 8, -150))
    restorePlayerCamera()
    setStatus("Voce entrou em OA.")
end)

local Info = Instance.new("TextLabel")
Info.Size = UDim2.new(1, -4, 0, 45)
Info.LayoutOrder = 9
Info.BackgroundTransparency = 1
Info.Text = "PLANETA DOS GUARDIOES\nOA FINAL | SISTEMA LOCAL"
Info.TextColor3 = C.LightStone
Info.TextSize = 11
Info.Font = Enum.Font.Gotham
Info.Parent = Content

--========================================================
-- ATMOSFERA VERDE LOCAL, PELE EXTERNA E GPS EM TEMPO REAL
--========================================================
local greenColor = Instance.new("ColorCorrectionEffect")
greenColor.Name = "OA_V93_GreenAtmosphere"
greenColor.Enabled = false
greenColor.TintColor = Color3.fromRGB(210, 255, 222)
greenColor.Saturation = 0.12
greenColor.Contrast = 0.025
greenColor.Brightness = 0.075
greenColor.Parent = Lighting

local greenBloom = Instance.new("BloomEffect")
greenBloom.Name = "OA_V93_GreenBloom"
greenBloom.Enabled = false
greenBloom.Intensity = 0.32
greenBloom.Size = 38
greenBloom.Threshold = 1.05
greenBloom.Parent = Lighting

-- Brilho do planeta visto de fora (resplendor verde da referencia).
local planetBloom = Instance.new("BloomEffect")
planetBloom.Name = "OA_V93_PlanetGlow"
planetBloom.Enabled = true
planetBloom.Intensity = 0.42
planetBloom.Size = 48
planetBloom.Threshold = 0.9
planetBloom.Parent = Lighting

-- Raios de sol suaves, ativos apenas dentro do globo.
local sunRays = Instance.new("SunRaysEffect")
sunRays.Name = "OA_V93_SunRays"
sunRays.Enabled = false
sunRays.Intensity = 0.11
sunRays.Spread = 0.72
sunRays.Parent = Lighting

-- Atmosfera volumetrica verde. Se o jogo ja tem uma, ela e ajustada
-- temporariamente e restaurada ao sair; horario e clima nao sao alterados.
local atmosphereObject = Lighting:FindFirstChildOfClass("Atmosphere")
local ownAtmosphere = false
if not atmosphereObject then
    atmosphereObject = Instance.new("Atmosphere")
    atmosphereObject.Name = "OA_V93_Atmosphere"
    atmosphereObject.Density = 0
    atmosphereObject.Parent = Lighting
    ownAtmosphere = true
end
local savedAtmosphere = nil

local function setOAAtmosphere(enabled)
    atmosphereActive = enabled
    greenColor.Enabled = enabled
    greenBloom.Enabled = enabled
    sunRays.Enabled = enabled
    -- O brilho forte so existe do lado de fora do globo.
    planetBloom.Enabled = not enabled

    if enabled then
        if not savedAtmosphere then
            savedAtmosphere = {
                Density = atmosphereObject.Density, Offset = atmosphereObject.Offset,
                Color = atmosphereObject.Color, Decay = atmosphereObject.Decay,
                Glare = atmosphereObject.Glare, Haze = atmosphereObject.Haze,
                Ambient = Lighting.Ambient, OutdoorAmbient = Lighting.OutdoorAmbient,
            }
        end
        atmosphereObject.Density = 0.28
        atmosphereObject.Offset = 0.12
        atmosphereObject.Color = Color3.fromRGB(150, 255, 190)
        atmosphereObject.Decay = Color3.fromRGB(40, 140, 90)
        atmosphereObject.Glare = 0.32
        atmosphereObject.Haze = 1.9
        Lighting.Ambient = savedAtmosphere.Ambient:Lerp(Color3.fromRGB(40, 110, 70), 0.3)
        Lighting.OutdoorAmbient = savedAtmosphere.OutdoorAmbient:Lerp(Color3.fromRGB(70, 150, 100), 0.3)
    elseif savedAtmosphere then
        atmosphereObject.Density = savedAtmosphere.Density
        atmosphereObject.Offset = savedAtmosphere.Offset
        atmosphereObject.Color = savedAtmosphere.Color
        atmosphereObject.Decay = savedAtmosphere.Decay
        atmosphereObject.Glare = savedAtmosphere.Glare
        atmosphereObject.Haze = savedAtmosphere.Haze
        Lighting.Ambient = savedAtmosphere.Ambient
        Lighting.OutdoorAmbient = savedAtmosphere.OutdoorAmbient
        savedAtmosphere = nil
    end
end

local function updateLocalSystems()
    local character = Player.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")
    if not root then
        GPSReadout.Text = "Distancia ate a OA: aguardando..."
        GPSCoords.Text = "XYZ (studs): --"
        if atmosphereActive then setOAAtmosphere(false) end
        return
    end

    local distanceFromPlanet = (root.Position - CENTER).Magnitude
    local insideOA = Map ~= nil and Map.Parent ~= nil
        and distanceFromPlanet <= (GLOBE_RADIUS * 1.08)
    if insideOA ~= atmosphereActive then setOAAtmosphere(insideOA) end

    local destination = GPS_DESTINATIONS[gpsDestinationIndex]
    local distance = (destination.position - root.Position).Magnitude
    GPSName.Text = destination.name
    GPSReadout.Text = string.format("Distancia: %.0f metros", distance * STUD_TO_METERS)
    GPSCoords.Text = string.format("X:%.0f Y:%.0f Z:%.0f", destination.position.X, destination.position.Y, destination.position.Z)
end

runtimeConnections[#runtimeConnections + 1] = RunService.RenderStepped:Connect(updateLocalSystems)

-- Pele externa: opaca de fora, dissolve suavemente quando a camera entra no globo.
runtimeConnections[#runtimeConnections + 1] = RunService.RenderStepped:Connect(function(dt)
    local camera = workspace.CurrentCamera
    if not camera then return end
    local cameraInside = (camera.CFrame.Position - CENTER).Magnitude < GLOBE_RADIUS * 0.985
    local target = cameraInside and 1 or 0
    outerFade = outerFade + (target - outerFade) * math.min(1, dt * 6)
    if math.abs(target - outerFade) < 0.01 then outerFade = target end
    if outerFade ~= lastAppliedFade then
        lastAppliedFade = outerFade
        applyOuterFade(outerFade)
    end

end)

-- Bateria Central: orbes em orbita e nucleo pulsante.
runtimeConnections[#runtimeConnections + 1] = RunService.RenderStepped:Connect(function()
    if not Map or not Map.Parent then return end
    local now = os.clock()
    if batteryCoreLight and batteryCoreLight.Parent then
        batteryCoreLight.Brightness = 3.2 + math.sin(now * 2.2) * 0.9
    end
    for _, orb in ipairs(batteryOrbs) do
        if orb.part.Parent then
            local angle = now * orb.speed + orb.phase
            local offset = CFrame.Angles(orb.tiltX, 0, orb.tiltZ)
                * Vector3.new(math.cos(angle) * orb.radius, 0, math.sin(angle) * orb.radius)
            orb.part.Position = CENTER + (orb.center + offset) * CITY_SCALE
        end
    end
end)

GUI.Destroying:Connect(function()
    for _, connection in ipairs(runtimeConnections) do
        if connection and connection.Connected then connection:Disconnect() end
    end
    setOAAtmosphere(false)
    if greenColor then greenColor:Destroy() end
    if greenBloom then greenBloom:Destroy() end
    if planetBloom then planetBloom:Destroy() end
    if sunRays then sunRays:Destroy() end
    if ownAtmosphere and atmosphereObject then atmosphereObject:Destroy() end
end)

--========================================================
-- MINIMIZAR E ARRASTAR (PC / MOBILE)
--========================================================
local minimized = false
Minimize.Activated:Connect(function()
    minimized = not minimized
    Content.Visible = not minimized
    Main.Size = minimized and UDim2.new(0, 320, 0, 50) or UDim2.new(0, 320, 0, 430)
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
setStatus("Construindo OA FINAL...")
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
print(" OA FINAL | PLANETA DOS GUARDIOES")
print(" Execucao local independente habilitada.")
print(" Reconstrucao transacional habilitada.")
print(" GPS sem seta, distancia em metros e atmosfera verde local.")
print(" Globo opaco verde visto de fora; cidade visivel apenas ao entrar.")
print(" Nevoa, nuvens, destrocos, estacoes orbitais e berco inferior adicionados.")
print(" Arquitetura urbana revisada, coberturas e placas apoiadas.")
print(" Centro de Comando e Museu dos Guardioes adicionados.")
print(" Bateria Central, telhado hibrido e Citadel monumental adicionados.")
print(" Edificios com interiores modulares.")
print(" Citadel, alas e sala de conferencia com coberturas fechadas.")
print(" Placas com suportes, passarelas com pilares e mobiliario urbano.")
print(" Globo cartoon: olhos seguem a camera, piscam, sorriso e chapeu de festa.")
print(" Atmosfera interna volumetrica, particulas e feixes de luz.")
print(" Bateria Central com orbes, coluna de energia e nucleo pulsante.")
print(" Edificios com colunas de canto, janelas laterais e coroa; torres com antena.")
print(" Lockdown e portas interativas locais.")
print(" Horario e clima do jogo preservados.")
print("========================================")
