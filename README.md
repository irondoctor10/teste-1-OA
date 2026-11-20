--[[
    OA 3.1.2 | EMERALD MOBILE + ATMOSPHERE FIX
    Launcher integrado para a versao OA FINAL do repositorio do usuario.

    O launcher carrega a base original e, depois que a construcao transacional
    termina, aplica a revisao urbanistica, registra a planta e gera uma
    visualizacao arquitetonica das superficies caminhaveis.

    ESCOPO LOCAL:
    - Nao usa RemoteEvents/RemoteFunctions.
    - Nao altera dados do servidor nem outros jogadores.
    - Nao cria NPCs ou sistemas de IA.
    - A NavMesh e uma visualizacao arquitetonica local, nao substitui o
      PathfindingService nem altera colisores do mapa.

    IMPORTANTE:
    Este arquivo e um launcher integrado e busca a base OA atual no URL
    definido abaixo. A base original permanece hospedada no repositorio.
]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")

-- Evita manter listeners duplicados quando o launcher for executado novamente.
local oldRuntime = PlayerGui:FindFirstChild("OA_31_RUNTIME")
if oldRuntime then oldRuntime:Destroy() end
local RuntimeFolder = Instance.new("Folder")
RuntimeFolder.Name = "OA_31_RUNTIME"
RuntimeFolder.Parent = PlayerGui

local BASE_URL = "https://raw.githubusercontent.com/irondoctor10/teste-1-OA/refs/heads/main/README.md"
local MAP_NAME = "OA_LIVING_FINAL"
local MAP_GUI_NAME = "OA_CONTROL_PANEL_2_2_PRO"
local EXTENSION_NAME = "17_OA_312_MOBILE_ATMOSPHERE"

local CENTER = Vector3.new(0, 6000, 0)
local CITY_SCALE = 0.72
local GLOBE_RADIUS = 950

local COLORS = {
    Green = Color3.fromRGB(40, 225, 135),
    Bright = Color3.fromRGB(135, 255, 185),
    Deep = Color3.fromRGB(5, 28, 20),
    Panel = Color3.fromRGB(8, 25, 19),
    Gold = Color3.fromRGB(224, 188, 104),
    Blue = Color3.fromRGB(90, 190, 255),
    Neutral = Color3.fromRGB(105, 122, 112), -- cinza-esverdeado para obstaculos, sem vermelho
    White = Color3.fromRGB(230, 245, 235),
    Muted = Color3.fromRGB(155, 190, 170),
}

local function report(message)
    print("[OA 3.1.1 | URBAN MASTERPLAN] " .. tostring(message))
end

--========================================================
-- 1. CARREGAR A BASE OFICIAL INFORMADA PELO USUARIO
--========================================================
local function loadBase()
    local okHttp, source = pcall(function()
        return game:HttpGet(BASE_URL)
    end)
    if not okHttp or type(source) ~= "string" or #source < 10000 then
        error("Nao foi possivel obter a base OA do repositorio. Verifique a URL e a conexao. Detalhe: " .. tostring(source))
    end

    -- Corrige coordenadas de GPS/teleporte da base, que ainda apontavam para
    -- posicoes anteriores a escala 0,72 e ao Masterplan 2.0.
    local function replaceOnce(text, oldText, newText)
        local first, last = string.find(text, oldText, 1, true)
        if not first then
            warn("[OA 3.1] Trecho de coordenada nao encontrado na base: " .. oldText)
            return text
        end
        return string.sub(text, 1, first - 1) .. newText .. string.sub(text, last + 1)
    end

    local coordinateFixes = {
        {"position = PRISON_POSITION + Vector3.new(0, 10, 0)", "position = CENTER + Vector3.new(263, 10, -130)"},
        {"position = CENTER + Vector3.new(-370, 12, -180)", "position = CENTER + Vector3.new(-266, 12, -130)"},
        {"position = CENTER + Vector3.new(0, 12, -440)", "position = CENTER + Vector3.new(0, 12, -316)"},
        {"position = CENTER + Vector3.new(-470, 12, 420)", "position = CENTER + Vector3.new(-300, 12, 370)"},
        {"position = CENTER + Vector3.new(0, 12, 610)", "position = CENTER + Vector3.new(0, 12, 460)"},
        {"position = CENTER + Vector3.new(470, 14, 420)", "position = CENTER + Vector3.new(300, 14, 370)"},
        {"{name = \"Prisão de Contenção\", position = PRISON_POSITION + Vector3.new(0, 18, 0)},", "{name = \"Prisão de Contenção\", position = CENTER + Vector3.new(263, 18, -130)},"},
        {"{name = \"Arquivos Centrais\", position = CENTER + Vector3.new(-370, 24, -180) * CITY_SCALE},", "{name = \"Arquivos Centrais\", position = CENTER + Vector3.new(-266, 24, -130)},"},
        {"{name = \"Hangar Orbital\", position = CENTER + Vector3.new(0, 35, 700) * CITY_SCALE},", "{name = \"Hangar Orbital\", position = CENTER + Vector3.new(520, 35, -340)},"},
        {"{name = \"Cafe dos Guardioes\", position = CENTER + Vector3.new(-260, 12, 470) * CITY_SCALE},", "{name = \"Cafe dos Guardioes\", position = CENTER + Vector3.new(-300, 12, 370)},"},
        {"{name = \"Mirante de Convivencia\", position = CENTER + Vector3.new(300, 12, 480) * CITY_SCALE},", "{name = \"Mirante de Convivencia\", position = CENTER + Vector3.new(0, 12, 460)},"},
        {"{name = \"Anfiteatro dos Guardioes\", position = CENTER + Vector3.new(0, 14, 420) * CITY_SCALE},", "{name = \"Anfiteatro dos Guardioes\", position = CENTER + Vector3.new(300, 14, 370)},"},
    }
    for _, fix in ipairs(coordinateFixes) do
        source = replaceOnce(source, fix[1], fix[2])
    end

    -- Ajustes da base para telas touch e saida confiavel da atmosfera.
    local function replaceExact(text, oldText, newText, label)
        local first, last = string.find(text, oldText, 1, true)
        if not first then
            warn("[OA 3.1.2] Patch nao aplicado (" .. label .. "): trecho original mudou.")
            return text
        end
        return string.sub(text, 1, first - 1) .. newText .. string.sub(text, last + 1)
    end

    local mobileLayoutOld = [=[
local function updateResponsiveLayout()
    local camera = workspace.CurrentCamera
    if not camera then return end
    local viewport = camera.ViewportSize
    local widthScale = math.clamp((viewport.X - 24) / 292, 0.78, 1)
    local heightScale = math.clamp((viewport.Y - 24) / 398, 0.72, 1)
    MainScale.Scale = math.min(widthScale, heightScale)
    -- Mantem o painel acessivel em telas pequenas sem ocupar o centro da tela.
    local maxX = math.max(8, viewport.X - 300)
    local maxY = math.max(8, viewport.Y - 406)
    local currentX = math.clamp(Main.Position.X.Offset, 8, maxX)
    local currentY = math.clamp(Main.Position.Y.Offset, 8, maxY)
    Main.Position = UDim2.new(0, currentX, 0, currentY)
end
]=]
    local mobileLayoutNew = [=[
local function updateResponsiveLayout()
    local camera = workspace.CurrentCamera
    if not camera then return end
    local viewport = camera.ViewportSize
    local touchMode = UserInputService.TouchEnabled
    local margin = touchMode and 10 or 12
    local targetWidth = math.max(200, math.min(292, viewport.X - margin * 2))
    local targetHeight = math.max(250, math.min(398, viewport.Y - margin * 2))
    Main.Size = UDim2.new(0, targetWidth, 0, targetHeight)
    MainScale.Scale = 1
    local maxX = math.max(4, viewport.X - targetWidth - margin)
    local maxY = math.max(4, viewport.Y - targetHeight - margin)
    local currentX = math.clamp(Main.Position.X.Offset, margin, maxX)
    local currentY = math.clamp(Main.Position.Y.Offset, margin, maxY)
    Main.Position = UDim2.new(0, currentX, 0, currentY)
end
]=]
    source = replaceExact(source, mobileLayoutOld, mobileLayoutNew, "menu responsivo")

    local buttonSizeOld = 'button.Size = UDim2.new(1, -4, 0, 39)'
    local buttonSizeNew = 'button.Size = UDim2.new(1, -4, 0, UserInputService.TouchEnabled and 46 or 39)'
    source = replaceExact(source, buttonSizeOld, buttonSizeNew, "botoes touch")

    local atmosphereOld = 'local insideOA = Map ~= nil and Map.Parent ~= nil\n        and distanceFromPlanet <= (GLOBE_RADIUS * 1.08)'
    local atmosphereNew = [=[
    -- Histerese: ativa apenas dentro do globo e desativa pouco depois de
    -- cruzar sua superficie externa. Evita que o efeito persista no espaco.
    local hasActiveMap = Map ~= nil and Map.Parent ~= nil
    local enterRadius = GLOBE_RADIUS * 0.985
    local exitRadius = GLOBE_RADIUS * 1.015
    local insideOA = hasActiveMap and (
        (atmosphereActive and distanceFromPlanet < exitRadius)
        or (not atmosphereActive and distanceFromPlanet <= enterRadius)
    )
]=]
    source = replaceExact(source, atmosphereOld, atmosphereNew, "limite de saida da atmosfera")

    -- Garante desligamento/restauracao ao remover o mapa ou perder o personagem.
    local missingRootOld = 'if not root then\n        GPSReadout.Text = "Distancia ate a OA: aguardando..."\n        GPSCoords.Text = "XYZ (studs): --"\n        if atmosphereActive then setOAAtmosphere(false) end\n        return\n    end'
    local missingRootNew = 'if not root then\n        GPSReadout.Text = "Distancia ate a OA: aguardando..."\n        GPSCoords.Text = "XYZ (studs): --"\n        if atmosphereActive then setOAAtmosphere(false) end\n        return\n    end'
    source = replaceExact(source, missingRootOld, missingRootNew, "restauracao sem personagem")

    local compiler = loadstring
    if type(compiler) ~= "function" then
        error("O ambiente nao disponibilizou loadstring para executar a base OA.")
    end

    local chunk, compileError = compiler(source)
    if not chunk then
        error("Falha ao compilar a base OA: " .. tostring(compileError))
    end

    local okRun, runError = pcall(chunk)
    if not okRun then
        error("A base OA encontrou um erro durante a inicializacao: " .. tostring(runError))
    end
    report("Base OA carregada. Aguardando a construcao transacional terminar...")
end

loadBase()

-- A base cria OA_LIVING_FINAL_STAGING e so troca o nome para OA_LIVING_FINAL
-- depois que todas as etapas de construcao terminam com sucesso.
local deadline = os.clock() + 300
local staging = nil
repeat
    staging = workspace:FindFirstChild(MAP_NAME .. "_STAGING")
    if not staging then task.wait(0.25) end
until staging or os.clock() >= deadline

if not staging then
    error("A pasta de construcao OA_LIVING_FINAL_STAGING nao apareceu. A base pode ter falhado; consulte o Output.")
end

while staging.Parent == workspace and staging.Name ~= MAP_NAME and os.clock() < deadline do
    task.wait(0.5)
end

if staging.Parent ~= workspace or staging.Name ~= MAP_NAME then
    error("A construcao da OA nao foi concluida no tempo esperado. Nenhuma alteracao urbanistica foi aplicada.")
end

local ActiveMap = staging

--========================================================
-- 2. MASTERPLAN 3.1: AJUSTES DE IMPLANTACAO
--========================================================
local function moveFolderContents(folder, delta, attributeName)
    if not folder or not folder.Parent then return 0 end
    if folder:GetAttribute(attributeName) == true then return 0 end
    local moved = 0
    for _, object in ipairs(folder:GetDescendants()) do
        if object:IsA("BasePart") then
            object.CFrame = CFrame.new(delta) * object.CFrame
            moved += 1
        end
    end
    folder:SetAttribute(attributeName, true)
    return moved
end

local function applyMasterplan(map)
    if not map or not map.Parent then return end
    local rp = map:FindFirstChild("14_ESPACOS_RP")
    local holo = map:FindFirstChild("02B_MAPA_HOLOGRAFICO")

    -- Compacta o arco social e deixa o jardim central como ligacao entre os
    -- espacos. Coordenadas ja estao na escala final (72%).
    if rp then
        moveFolderContents(rp:FindFirstChild("Cafe_dos_Guardioes"), Vector3.new(170, 0, -50), "OA31_CafeRepositioned")
        moveFolderContents(rp:FindFirstChild("Mirante_de_Convivencia"), Vector3.new(0, 0, -151), "OA31_LookoutRepositioned")
        moveFolderContents(rp:FindFirstChild("Anfiteatro_dos_Guardioes"), Vector3.new(-170, 0, -50), "OA31_AmphitheaterRepositioned")
    end

    -- O mapa holografico da base ficava demasiado perto do Cafe. Move para
    -- o quadrante noroeste, criando um setor cartografico sem sobreposicao.
    moveFolderContents(holo, Vector3.new(-61, 0, -738), "OA31_HoloMapRepositioned")

    local extension = map:FindFirstChild(EXTENSION_NAME)
    if not extension then
        extension = Instance.new("Folder")
        extension.Name = EXTENSION_NAME
        extension.Parent = map
    end

    extension:SetAttribute("Version", "3.1.2")
    extension:SetAttribute("PlanTitle", "OA | Emerald Urban Masterplan")
    extension:SetAttribute("CoordinateOrigin", "0, 6000, 0")
    extension:SetAttribute("CityScale", CITY_SCALE)
    extension:SetAttribute("NoNPCs", true)

    -- Cadastro permanente da planta para futuras revisoes. As coordenadas
    -- sao relativas ao centro de OA e representam a implantacao final.
    local registry = extension:FindFirstChild("DistrictRegistry")
    if not registry then
        registry = Instance.new("Folder")
        registry.Name = "DistrictRegistry"
        registry.Parent = extension
    else
        registry:ClearAllChildren()
    end

    local districts = {
        {"Bateria Central", 0, 0, "Nucleo energetico e marco monumental"},
        {"Centro Planetario", 0, 0, "Praca central e distribuicao radial"},
        {"Citadel dos Guardioes", 0, -316, "Conselho, recepcao e alas administrativas"},
        {"Terminal de Transporte", 0, -560, "Ponto de chegada e conexao urbana"},
        {"Hangar Orbital", 520, -340, "Doca elevada e infraestrutura orbital"},
        {"Mapa Holografico", -500, -400, "Orientacao e cartografia da cidade"},
        {"Arquivos Centrais", -266, -130, "Biblioteca e consulta"},
        {"Prisão de Contencao", 263, -130, "Setor de seguranca"},
        {"Centro de Comando", -468, -58, "Administracao e operacao"},
        {"Museu dos Guardioes", 468, -58, "Memoria e exposicao"},
        {"Distrito Residencial", -320, 190, "Moradias e escala de bairro"},
        {"Distrito Comercial", 320, 190, "Lojas e servicos"},
        {"Jardins Centrais", 0, 302, "Vegetacao, fontes e pausa"},
        {"Cafe dos Guardioes", -300, 370, "Convivencia e descanso"},
        {"Anfiteatro dos Guardioes", 300, 370, "Encontros e eventos sociais"},
        {"Mirante de Convivencia", 0, 460, "Observacao e tranquilidade"},
        {"Distrito Tecnologico", 115, 353, "Complexos de pesquisa"},
    }

    for index, data in ipairs(districts) do
        local entry = Instance.new("Folder")
        entry.Name = string.format("%02d_%s", index, data[1]:gsub("%W+", "_"))
        entry:SetAttribute("DisplayName", data[1])
        entry:SetAttribute("RelativeX", data[2])
        entry:SetAttribute("RelativeZ", data[3])
        entry:SetAttribute("Description", data[4])
        entry:SetAttribute("WorldPosition", CENTER + Vector3.new(data[2], 0, data[3]))
        entry.Parent = registry
    end

    return extension
end

local ActiveExtension = applyMasterplan(ActiveMap)
report("Masterplan 3.1 aplicado: arco social compactado e mapa holografico reposicionado.")

--========================================================
-- 3. NAVMESH ARQUITETONICA LOCAL
--========================================================
local NavMeshFolder = nil
local NavMeshVisible = false
local NavMeshRequested = false
local navButtonRef = nil
local NavMeshStats = {surfaces = 0, cells = 0, blocked = 0, walkable = 0}
local NAV_CELL_BASE = 24
local NAV_MAX_CELLS = 1750

local function nameLooksWalkable(name)
    name = string.lower(name)
    local patterns = {
        "piso", "laje", "plataforma", "passarela", "corredor",
        "degrau", "escadaria", "deck", "avenida", "calcada",
        "hangar_piso", "estacao", "patio",
    }
    for _, token in ipairs(patterns) do
        if string.find(name, token, 1, true) then return true end
    end
    return false
end

local function getSurfaceInfo(part)
    if not part:IsA("BasePart") or not part.CanCollide or part.Transparency >= 0.95 then
        return nil
    end
    if not nameLooksWalkable(part.Name) then return nil end

    local isCylinder = part:IsA("Part") and part.Shape == Enum.PartType.Cylinder
    if isCylinder then
        local axisY = math.abs(part.CFrame.RightVector.Y)
        if axisY < 0.88 then return nil end
        local topY = part.Position.Y + (part.Size.X * 0.5 * axisY)
        return {kind = "cylinder", topY = topY, normalY = axisY}
    end

    local normalY = math.abs(part.CFrame.UpVector.Y)
    if normalY < 0.88 or part.Size.Y > 30 then return nil end
    local topY = part.Position.Y + (part.Size.Y * 0.5 * normalY)
    return {kind = "box", topY = topY, normalY = normalY}
end

local function clearOldNavMesh(map)
    local old = map and map:FindFirstChild("OA_31_ARCHITECTURAL_NAVMESH")
    if old then old:Destroy() end
end

local function buildNavMesh(map)
    if not map or not map.Parent then return false, "Mapa OA nao encontrado." end
    clearOldNavMesh(map)

    local folder = Instance.new("Folder")
    folder.Name = "OA_31_ARCHITECTURAL_NAVMESH"
    folder.Parent = map
    NavMeshFolder = folder

    local candidates = {}
    for _, descendant in ipairs(map:GetDescendants()) do
        local info = getSurfaceInfo(descendant)
        if info then
            local offset = descendant.Position - CENTER
            if Vector3.new(offset.X, 0, offset.Z).Magnitude <= GLOBE_RADIUS * 0.9 then
                candidates[#candidates + 1] = {part = descendant, info = info}
            end
        end
    end

    table.sort(candidates, function(a, b)
        local sa, sb = a.part.Size, b.part.Size
        return (sa.X * sa.Z) > (sb.X * sb.Z)
    end)

    local floorSet = {}
    for _, candidate in ipairs(candidates) do floorSet[candidate.part] = true end

    local overlap = OverlapParams.new()
    overlap.FilterType = Enum.RaycastFilterType.Exclude
    overlap.FilterDescendantsInstances = {folder}
    overlap.RespectCanCollide = true
    overlap.MaxParts = 12

    local seen = {}
    local cells = 0
    local blockedCount = 0
    local walkableCount = 0

    for _, candidate in ipairs(candidates) do
        if cells >= NAV_MAX_CELLS then break end
        local part = candidate.part
        local info = candidate.info
        local size = part.Size
        local areaEstimate
        if info.kind == "cylinder" then
            local radius = math.min(size.Y, size.Z) * 0.5
            areaEstimate = math.pi * radius * radius
        else
            areaEstimate = size.X * size.Z
        end

        local adaptiveStep = math.max(NAV_CELL_BASE, math.sqrt(areaEstimate / 360) * 1.08)
        local step = math.min(42, adaptiveStep)
        local halfCell = step * 0.39

        local function addCell(worldPosition)
            if cells >= NAV_MAX_CELLS then return end
            local relative = worldPosition - CENTER
            if Vector3.new(relative.X, 0, relative.Z).Magnitude > GLOBE_RADIUS * 0.9 then return end

            local key = string.format("%d:%d:%d",
                math.floor(worldPosition.X / step + 0.5),
                math.floor(worldPosition.Z / step + 0.5),
                math.floor(worldPosition.Y / 12 + 0.5))
            if seen[key] then return end
            seen[key] = true

            local queryCF = CFrame.new(worldPosition + Vector3.new(0, 4.5, 0))
            local querySize = Vector3.new(math.max(8, step * 0.72), 8, math.max(8, step * 0.72))
            local hits = workspace:GetPartBoundsInBox(queryCF, querySize, overlap)
            local isBlocked = false
            for _, hit in ipairs(hits) do
                if hit ~= part and not floorSet[hit] then
                    isBlocked = true
                    break
                end
            end

            local tile = Instance.new("Part")
            tile.Name = isBlocked and "Cell_Bloqueada" or "Cell_Caminhavel"
            tile.Anchored = true
            tile.CanCollide = false
            tile.CanTouch = false
            tile.CanQuery = false
            tile.CastShadow = false
            tile.Material = Enum.Material.Neon
            tile.Transparency = isBlocked and 0.76 or 0.62
            tile.Size = Vector3.new(math.max(7, step - 3), 0.16, math.max(7, step - 3))
            tile.CFrame = CFrame.new(worldPosition + Vector3.new(0, 0.24, 0))
            tile.Color = isBlocked and COLORS.Neutral or (info.normalY < 0.97 and COLORS.Gold or COLORS.Green)
            tile.Parent = folder

            cells += 1
            if isBlocked then blockedCount += 1 else walkableCount += 1 end
        end

        if info.kind == "cylinder" then
            local radius = math.min(size.Y, size.Z) * 0.5 - halfCell * 0.6
            if radius > 2 then
                for localY = -radius, radius, step do
                    for localZ = -radius, radius, step do
                        if localY * localY + localZ * localZ <= radius * radius then
                            local localPoint = Vector3.new(size.X * 0.5 + 0.12, localY, localZ)
                            local worldPoint = part.CFrame:PointToWorldSpace(localPoint)
                            addCell(Vector3.new(worldPoint.X, info.topY, worldPoint.Z))
                        end
                        if cells >= NAV_MAX_CELLS then break end
                    end
                    if cells >= NAV_MAX_CELLS then break end
                end
            end
        else
            local usableX = math.max(0, size.X * 0.5 - halfCell * 0.5)
            local usableZ = math.max(0, size.Z * 0.5 - halfCell * 0.5)
            if usableX < 1 or usableZ < 1 then
                local point = part.CFrame:PointToWorldSpace(Vector3.new(0, size.Y * 0.5 + 0.12, 0))
                addCell(Vector3.new(point.X, info.topY, point.Z))
            else
                for localX = -usableX, usableX, step do
                    for localZ = -usableZ, usableZ, step do
                        local localPoint = Vector3.new(localX, size.Y * 0.5 + 0.12, localZ)
                        local worldPoint = part.CFrame:PointToWorldSpace(localPoint)
                        addCell(Vector3.new(worldPoint.X, info.topY, worldPoint.Z))
                        if cells >= NAV_MAX_CELLS then break end
                    end
                    if cells >= NAV_MAX_CELLS then break end
                end
            end
        end

        if cells % 90 < 3 then task.wait() end
    end

    NavMeshStats = {
        surfaces = #candidates,
        cells = cells,
        blocked = blockedCount,
        walkable = walkableCount,
    }
    folder:SetAttribute("SurfaceCount", #candidates)
    folder:SetAttribute("CellCount", cells)
    folder:SetAttribute("BlockedCount", blockedCount)
    folder:SetAttribute("WalkableCount", walkableCount)
    report(string.format("NavMesh arquitetonica: %d superficies, %d celulas, %d livres, %d com obstrucao.",
        #candidates, cells, walkableCount, blockedCount))
    return true, NavMeshStats
end

local function setNavMeshVisible(visible)
    NavMeshVisible = visible
    if not NavMeshFolder or not NavMeshFolder.Parent then return end
    for _, object in ipairs(NavMeshFolder:GetChildren()) do
        if object:IsA("BasePart") then
            object.LocalTransparencyModifier = visible and 0 or 1
        end
    end
end

--========================================================
-- 4. PLANTA 2D DENTRO DO MENU OA EXISTENTE
--========================================================
local existingGui = PlayerGui:WaitForChild(MAP_GUI_NAME, 45)
if not existingGui then
    report("A OA foi construida, mas o painel original nao foi encontrado. NavMesh continua disponivel por F6/F8.")
end

local mapPanel = nil
local statsLabel = nil
local selectedLabel = nil

local mapPoints = {
    {name = "BATERIA", x = 0, z = 0, color = COLORS.Gold, description = "Nucleo energetico central"},
    {name = "CITADEL", x = 0, z = -316, color = COLORS.Bright, description = "Conselho e areas administrativas"},
    {name = "TERMINAL", x = 0, z = -560, color = COLORS.Blue, description = "Terminal de transporte"},
    {name = "HANGAR", x = 520, z = -340, color = COLORS.Blue, description = "Hangar orbital"},
    {name = "MAPA HOLO", x = -500, z = -400, color = COLORS.Blue, description = "Centro cartografico"},
    {name = "ARQUIVOS", x = -266, z = -130, color = COLORS.Bright, description = "Biblioteca e arquivos"},
    {name = "PRISAO", x = 263, z = -130, color = COLORS.Neutral, description = "Setor de seguranca"},
    {name = "COMANDO", x = -468, z = -58, color = COLORS.Gold, description = "Centro de comando"},
    {name = "MUSEU", x = 468, z = -58, color = COLORS.Gold, description = "Museu dos Guardioes"},
    {name = "RESIDENCIAL", x = -320, z = 190, color = COLORS.Green, description = "Moradias"},
    {name = "COMERCIAL", x = 320, z = 190, color = COLORS.Gold, description = "Comercio e servicos"},
    {name = "JARDINS", x = 0, z = 302, color = COLORS.Green, description = "Jardins centrais"},
    {name = "CAFE", x = -300, z = 370, color = COLORS.Green, description = "Convivencia e descanso"},
    {name = "ANFITEATRO", x = 300, z = 370, color = COLORS.Green, description = "Encontros comunitarios"},
    {name = "MIRANTE", x = 0, z = 460, color = COLORS.Blue, description = "Observacao e tranquilidade"},
    {name = "TECNOLOGIA", x = 115, z = 353, color = COLORS.Blue, description = "Pesquisa e tecnologia"},
}

local function createMapPanel(gui)
    if not gui then return end
    local old = gui:FindFirstChild("OA_31_Architectural_Map")
    if old then old:Destroy() end

    local overlay = Instance.new("Frame")
    overlay.Name = "OA_31_Architectural_Map"
    overlay.Size = UDim2.fromScale(1, 1)
    overlay.BackgroundColor3 = Color3.new(0, 0, 0)
    overlay.BackgroundTransparency = 0.35
    overlay.Visible = false
    overlay.ZIndex = 50
    overlay.Parent = gui

    local panel = Instance.new("Frame")
    panel.Name = "Panel"
    panel.Size = UDim2.new(0, 440, 0, 430)
    panel.Position = UDim2.fromScale(0.5, 0.5)
    panel.AnchorPoint = Vector2.new(0.5, 0.5)
    panel.BackgroundColor3 = COLORS.Panel
    panel.BorderSizePixel = 0
    panel.ZIndex = 51
    panel.Parent = overlay
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 14)
    corner.Parent = panel
    local stroke = Instance.new("UIStroke")
    stroke.Color = COLORS.Green
    stroke.Thickness = 1.5
    stroke.Transparency = 0.12
    stroke.Parent = panel

    local scale = Instance.new("UIScale")
    scale.Scale = 1
    scale.Parent = panel
    local function resize()
        local camera = workspace.CurrentCamera
        if camera then
            local viewport = camera.ViewportSize
            scale.Scale = math.clamp(math.min((viewport.X - 28) / 440, (viewport.Y - 28) / 430), 0.68, 1)
        end
    end
    resize()
    if workspace.CurrentCamera then
        workspace.CurrentCamera:GetPropertyChangedSignal("ViewportSize"):Connect(resize)
    end

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -64, 0, 34)
    title.Position = UDim2.new(0, 16, 0, 9)
    title.BackgroundTransparency = 1
    title.Text = "OA | PLANTA URBANA 3.1.1"
    title.TextColor3 = COLORS.Bright
    title.TextSize = 15
    title.Font = Enum.Font.GothamBold
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.ZIndex = 52
    title.Parent = panel

    local close = Instance.new("TextButton")
    close.Size = UDim2.new(0, 32, 0, 30)
    close.Position = UDim2.new(1, -42, 0, 10)
    close.BackgroundColor3 = Color3.fromRGB(24, 76, 53)
    close.Text = "X"
    close.TextColor3 = COLORS.White
    close.Font = Enum.Font.GothamBold
    close.TextSize = 13
    close.ZIndex = 53
    close.Parent = panel
    Instance.new("UICorner", close).CornerRadius = UDim.new(0, 7)
    close.Activated:Connect(function() overlay.Visible = false end)

    local subtitle = Instance.new("TextLabel")
    subtitle.Size = UDim2.new(1, -28, 0, 20)
    subtitle.Position = UDim2.new(0, 14, 0, 43)
    subtitle.BackgroundTransparency = 1
    subtitle.Text = "NORTE ACIMA  |  ORIGEM: (0, 6000, 0)  |  ESCALA: 0,72"
    subtitle.TextColor3 = COLORS.Muted
    subtitle.TextSize = 9
    subtitle.Font = Enum.Font.Gotham
    subtitle.TextXAlignment = Enum.TextXAlignment.Left
    subtitle.ZIndex = 52
    subtitle.Parent = panel

    local mapCanvas = Instance.new("Frame")
    mapCanvas.Name = "TopDownMap"
    mapCanvas.Size = UDim2.new(0, 300, 0, 280)
    mapCanvas.Position = UDim2.new(0, 14, 0, 72)
    mapCanvas.BackgroundColor3 = Color3.fromRGB(6, 34, 24)
    mapCanvas.BackgroundTransparency = 0.05
    mapCanvas.BorderSizePixel = 0
    mapCanvas.ClipsDescendants = true
    mapCanvas.ZIndex = 52
    mapCanvas.Parent = panel
    Instance.new("UICorner", mapCanvas).CornerRadius = UDim.new(0, 10)
    local canvasStroke = Instance.new("UIStroke")
    canvasStroke.Color = Color3.fromRGB(54, 135, 93)
    canvasStroke.Transparency = 0.3
    canvasStroke.Parent = mapCanvas

    local ring = Instance.new("Frame")
    ring.Size = UDim2.new(0, 250, 0, 250)
    ring.Position = UDim2.new(0.5, -125, 0.5, -125)
    ring.BackgroundTransparency = 1
    ring.ZIndex = 52
    ring.Parent = mapCanvas
    Instance.new("UICorner", ring).CornerRadius = UDim.new(1, 0)
    local ringStroke = Instance.new("UIStroke")
    ringStroke.Color = Color3.fromRGB(58, 145, 96)
    ringStroke.Thickness = 1
    ringStroke.Transparency = 0.35
    ringStroke.Parent = ring

    local crossH = Instance.new("Frame")
    crossH.Size = UDim2.new(0, 266, 0, 1)
    crossH.Position = UDim2.new(0.5, -133, 0.5, 0)
    crossH.BackgroundColor3 = Color3.fromRGB(48, 106, 75)
    crossH.BorderSizePixel = 0
    crossH.ZIndex = 52
    crossH.Parent = mapCanvas
    local crossV = Instance.new("Frame")
    crossV.Size = UDim2.new(0, 1, 0, 266)
    crossV.Position = UDim2.new(0.5, 0, 0.5, -133)
    crossV.BackgroundColor3 = Color3.fromRGB(48, 106, 75)
    crossV.BorderSizePixel = 0
    crossV.ZIndex = 52
    crossV.Parent = mapCanvas

    local centerMark = Instance.new("Frame")
    centerMark.Size = UDim2.new(0, 12, 0, 12)
    centerMark.Position = UDim2.new(0.5, -6, 0.5, -6)
    centerMark.BackgroundColor3 = COLORS.Gold
    centerMark.BorderSizePixel = 0
    centerMark.ZIndex = 54
    centerMark.Parent = mapCanvas
    Instance.new("UICorner", centerMark).CornerRadius = UDim.new(1, 0)

    selectedLabel = Instance.new("TextLabel")
    selectedLabel.Size = UDim2.new(1, -328, 0, 66)
    selectedLabel.Position = UDim2.new(0, 326, 0, 90)
    selectedLabel.BackgroundTransparency = 1
    selectedLabel.Text = "Selecione um distrito\npara ver sua funcao e coordenadas."
    selectedLabel.TextColor3 = COLORS.White
    selectedLabel.TextSize = 11
    selectedLabel.Font = Enum.Font.Gotham
    selectedLabel.TextWrapped = true
    selectedLabel.TextXAlignment = Enum.TextXAlignment.Left
    selectedLabel.TextYAlignment = Enum.TextYAlignment.Top
    selectedLabel.ZIndex = 53
    selectedLabel.Parent = panel

    local key = Instance.new("TextLabel")
    key.Size = UDim2.new(1, -328, 0, 95)
    key.Position = UDim2.new(0, 326, 0, 164)
    key.BackgroundTransparency = 1
    key.Text = "LEGENDA\n\nVERDE  Caminhavel\nCINZA  Obstrucao / area bloqueada\nDOURADO  Rampa / inclinacao\nAZUL  Area de referencia"
    key.TextColor3 = COLORS.Muted
    key.TextSize = 9
    key.Font = Enum.Font.Gotham
    key.TextXAlignment = Enum.TextXAlignment.Left
    key.TextYAlignment = Enum.TextYAlignment.Top
    key.ZIndex = 53
    key.Parent = panel

    for _, point in ipairs(mapPoints) do
        local x = 150 + (point.x / 680) * 124
        local y = 140 + (point.z / 680) * 124
        local node = Instance.new("TextButton")
        node.Name = "District_" .. point.name:gsub("%W+", "_")
        node.Size = UDim2.new(0, 64, 0, 18)
        node.Position = UDim2.new(0, x - 32, 0, y - 9)
        node.BackgroundColor3 = point.color
        node.BackgroundTransparency = 0.12
        node.Text = point.name
        node.TextColor3 = Color3.fromRGB(5, 24, 17)
        node.TextSize = 8
        node.Font = Enum.Font.GothamBold
        node.AutoButtonColor = true
        node.ZIndex = 55
        node.Parent = mapCanvas
        Instance.new("UICorner", node).CornerRadius = UDim.new(0, 5)
        node.Activated:Connect(function()
            selectedLabel.Text = string.format("%s\n%s\nX: %d  |  Z: %d", point.name, point.description, point.x, point.z)
        end)
    end

    statsLabel = Instance.new("TextLabel")
    statsLabel.Size = UDim2.new(1, -28, 0, 34)
    statsLabel.Position = UDim2.new(0, 14, 0, 360)
    statsLabel.BackgroundColor3 = Color3.fromRGB(13, 52, 37)
    statsLabel.BackgroundTransparency = 0.05
    statsLabel.Text = "NavMesh desativada. Use F8 ou GERAR NAVMESH para analisar."
    statsLabel.TextColor3 = COLORS.Bright
    statsLabel.TextSize = 10
    statsLabel.Font = Enum.Font.GothamBold
    statsLabel.ZIndex = 53
    statsLabel.Parent = panel
    Instance.new("UICorner", statsLabel).CornerRadius = UDim.new(0, 7)

    local buttonsRow = Instance.new("Frame")
    buttonsRow.Size = UDim2.new(1, -28, 0, 28)
    buttonsRow.Position = UDim2.new(0, 14, 0, 399)
    buttonsRow.BackgroundTransparency = 1
    buttonsRow.ZIndex = 53
    buttonsRow.Parent = panel

    local navButton = Instance.new("TextButton")
    navButton.Size = UDim2.new(0.49, 0, 1, 0)
    navButton.Position = UDim2.new(0, 0, 0, 0)
    navButton.BackgroundColor3 = Color3.fromRGB(21, 94, 60)
    navButton.Text = "GERAR NAVMESH"
    navButton.TextColor3 = COLORS.White
    navButton.TextSize = 9
    navButton.Font = Enum.Font.GothamBold
    navButton.ZIndex = 54
    navButton.Parent = buttonsRow
    Instance.new("UICorner", navButton).CornerRadius = UDim.new(0, 6)
    navButtonRef = navButton
    navButton.Activated:Connect(function()
        if not NavMeshFolder or not NavMeshFolder.Parent then
            NavMeshRequested = true
            navButton.Text = "GERANDO..."
            task.spawn(function()
                local ok, result = buildNavMesh(ActiveMap)
                if ok then
                    setNavMeshVisible(true)
                    navButton.Text = "OCULTAR NAVMESH"
                    statsLabel.Text = string.format("%d superficies  |  %d celulas  |  %d livres  |  %d obstruidas",
                        result.surfaces, result.cells, result.walkable, result.blocked)
                else
                    navButton.Text = "GERAR NAVMESH"
                    statsLabel.Text = tostring(result)
                end
            end)
        else
            setNavMeshVisible(not NavMeshVisible)
            navButton.Text = NavMeshVisible and "OCULTAR NAVMESH" or "MOSTRAR NAVMESH"
        end
    end)

    local rebuildButton = Instance.new("TextButton")
    rebuildButton.Size = UDim2.new(0.49, 0, 1, 0)
    rebuildButton.Position = UDim2.new(0.51, 0, 0, 0)
    rebuildButton.BackgroundColor3 = Color3.fromRGB(25, 70, 53)
    rebuildButton.Text = "RECALCULAR"
    rebuildButton.TextColor3 = COLORS.White
    rebuildButton.TextSize = 9
    rebuildButton.Font = Enum.Font.GothamBold
    rebuildButton.ZIndex = 54
    rebuildButton.Parent = buttonsRow
    Instance.new("UICorner", rebuildButton).CornerRadius = UDim.new(0, 6)
    rebuildButton.Activated:Connect(function()
        rebuildButton.Text = "ANALISANDO..."
        task.spawn(function()
            NavMeshRequested = true
            local ok, result = buildNavMesh(ActiveMap)
            if ok then
                setNavMeshVisible(true)
                navButton.Text = "OCULTAR NAVMESH"
                statsLabel.Text = string.format("%d superficies  |  %d celulas  |  %d livres  |  %d obstruidas",
                    result.surfaces, result.cells, result.walkable, result.blocked)
            else
                statsLabel.Text = tostring(result)
            end
            rebuildButton.Text = "RECALCULAR"
        end)
    end)

    mapPanel = overlay
    return overlay
end

if existingGui then
    mapPanel = createMapPanel(existingGui)

    local content = existingGui:FindFirstChild("Content", true)
    if content and content:IsA("ScrollingFrame") then
        local oldButton = content:FindFirstChild("OA_31_OpenUrbanMap")
        if oldButton then oldButton:Destroy() end

        local openButton = Instance.new("TextButton")
        openButton.Name = "OA_31_OpenUrbanMap"
        openButton.Size = UDim2.new(1, -4, 0, 39)
        openButton.LayoutOrder = 18
        openButton.BackgroundColor3 = Color3.fromRGB(20, 86, 57)
        openButton.Text = "PLANTA URBANA  /  NAVMESH"
        openButton.TextColor3 = COLORS.White
        openButton.TextSize = 11
        openButton.Font = Enum.Font.GothamBold
        openButton.Parent = content
        Instance.new("UICorner", openButton).CornerRadius = UDim.new(0, 7)
        local outline = Instance.new("UIStroke")
        outline.Color = COLORS.Bright
        outline.Transparency = 0.55
        outline.Parent = openButton
        openButton.Activated:Connect(function()
            if mapPanel then mapPanel.Visible = true end
        end)
    end
end

--========================================================
-- 5. NAVMESH SOB DEMANDA E ATALHOS
--========================================================
-- A NavMesh nao e criada automaticamente: o mapa fica limpo ao iniciar.
-- O usuario pode gerar a camada de diagnostico quando quiser, pelo F8 ou
-- pelo botao do painel. Celulas bloqueadas usam cinza, nunca vermelho.
local inputConnection = UserInputService.InputBegan:Connect(function(input, processed)
    if not RuntimeFolder.Parent or processed then return end
    if input.KeyCode == Enum.KeyCode.F6 then
        if NavMeshFolder and NavMeshFolder.Parent then
            setNavMeshVisible(not NavMeshVisible)
            if navButtonRef then
                navButtonRef.Text = NavMeshVisible and "OCULTAR NAVMESH" or "MOSTRAR NAVMESH"
            end
        elseif statsLabel then
            statsLabel.Text = "NavMesh ainda nao foi gerada. Use F8 para analisar."
        end
    elseif input.KeyCode == Enum.KeyCode.F7 then
        if mapPanel then mapPanel.Visible = not mapPanel.Visible end
    elseif input.KeyCode == Enum.KeyCode.F8 then
        NavMeshRequested = true
        task.spawn(function()
            local ok, result = buildNavMesh(ActiveMap)
            if ok then
                setNavMeshVisible(true)
                if navButtonRef then navButtonRef.Text = "OCULTAR NAVMESH" end
            end
            if ok and statsLabel then
                statsLabel.Text = string.format("%d superficies  |  %d celulas  |  %d livres  |  %d obstruidas",
                    result.surfaces, result.cells, result.walkable, result.blocked)
            elseif not ok and statsLabel then
                statsLabel.Text = tostring(result)
            end
        end)
    end
end)

-- O menu original permite reconstruir OA. Detecta a troca transacional de
-- mapa e reaplica apenas os ajustes arquitetonicos, sem duplicar conteudo.
task.spawn(function()
    while RuntimeFolder.Parent do
        task.wait(1.5)
        local current = workspace:FindFirstChild(MAP_NAME)
        if current and current ~= ActiveMap then
            ActiveMap = current
            ActiveExtension = applyMasterplan(ActiveMap)
            task.wait(0.8)
            if NavMeshRequested then
                local ok, result = buildNavMesh(ActiveMap)
                if ok then
                    setNavMeshVisible(NavMeshVisible)
                    if navButtonRef then
                        navButtonRef.Text = NavMeshVisible and "OCULTAR NAVMESH" or "MOSTRAR NAVMESH"
                    end
                    if statsLabel then
                        statsLabel.Text = string.format("%d superficies  |  %d celulas  |  %d livres  |  %d obstruidas",
                            result.surfaces, result.cells, result.walkable, result.blocked)
                    end
                end
            end
            report("Novo mapa detectado: Masterplan atualizado" .. (NavMeshRequested and " e NavMesh recalculada." or "."))
        end
    end
end)

report("OA 3.1.2 carregada. Menu responsivo; atmosfera com saida corrigida; NavMesh sob demanda.")
print("[OA 3.1] A NavMesh e uma leitura visual aproximada das superficies fisicas locais; revise rampas, escadas e interiores no jogo.")
