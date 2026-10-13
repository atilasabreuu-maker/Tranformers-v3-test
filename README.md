--[[
===============================================================
 TRANSFORMER ENGINE — ULTIMATE MOBILE V3 / AUTÔNOMO
 Um único LocalScript. Não exige modelos, animações, sons,
 pastas, RemoteEvents, módulos ou IDs preparados pelo criador.

 INSTALAÇÃO:
 StarterPlayer > StarterPlayerScripts > LocalScript
 Cole este arquivo inteiro.

 O script constrói um mecha e um caminhão usando Parts nativas,
 esconde/restaura localmente o avatar, anima as placas por CFrame,
 e cria UI, partículas, luzes e controles mobile em tempo de execução.

 LIMITAÇÕES IMPORTANTES:
 • Os modelos gerados são mecânicos/estilizados, não malhas 3D
   cinematográficas modeladas à mão.
 • É uma transformação visual local. Um LocalScript sozinho não
   replica o mecha para os outros jogadores nem controla autoridade
   física do servidor.
 • O Roblox não fornece síntese de efeitos sonoros arbitrários em
   Luau puro sem fonte de áudio; esta versão não finge possuir SFX.
===============================================================
]]

---------------------------
-- 1. SERVIÇOS E CONFIGURAÇÃO
---------------------------
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local CFG = {
    TRANSFORM_TIME = 3.15,
    HUMANOID_SPEED = 16,
    MECHA_SPEED = 27,
    VEHICLE_SPEED = 58,
    MECHA_JUMP_POWER = 58,
    MECHA_JUMP_HEIGHT = 8,
    WALK_SWAY = 0.52,
    RUN_SWAY = 0.72,
    MECHA_FOV = 77,
    VEHICLE_FOV = 80,
    HUMANOID_FOV = 70,
    FX_PARTICLE_RATE = 16,
    FX_LOW_PARTICLE_RATE = 8,
    SHOW_KEYBOARD_SHORTCUTS = true,
    -- O modo automático baixa os efeitos quando o dispositivo começa a atrasar.
    AUTO_QUALITY = true,
    MOBILE_TARGET_FPS = 45,
    LOW_FPS_THRESHOLD = 32,
    HIGH_FPS_RECOVERY = 52,
    DRIVE_ACCELERATION = 1.0,
    DRIVE_BRAKE = 1.35,
    DRIVE_MAX_SPEED = 74,
    DRIVE_BOOST_SPEED = 92,
    DRIVE_REVERSE_SPEED = 22,
    DRIVE_TURN_RATE = 1.55,
    MECHA_DASH_SPEED = 58,
    MECHA_DASH_TIME = 0.34,
    MECHA_PUNCH_TIME = 0.42,
    CAMERA_SHAKE_MECHA = 0.018,
    CAMERA_SHAKE_VEHICLE = 0.012,
    TELEMETRY_INTERVAL = 0.12,
    DETAIL_PART_BUDGET_MOBILE = 320,
    TRANSFORM_STAGE_ANTICIPATION = 0.14,
    TRANSFORM_STAGE_UNFOLD = 0.30,
    TRANSFORM_STAGE_ROTATE = 0.34,
    TRANSFORM_STAGE_LOCK = 0.22,
}

local COLORS = {
    Red = Color3.fromRGB(190, 39, 43),
    RedLight = Color3.fromRGB(255, 75, 61),
    Blue = Color3.fromRGB(28, 93, 190),
    BlueLight = Color3.fromRGB(0, 190, 255),
    BlueGlass = Color3.fromRGB(20, 70, 112),
    Steel = Color3.fromRGB(87, 105, 122),
    Silver = Color3.fromRGB(170, 186, 198),
    Dark = Color3.fromRGB(20, 28, 39),
    Black = Color3.fromRGB(7, 10, 16),
    Gold = Color3.fromRGB(255, 181, 48),
    White = Color3.fromRGB(225, 240, 255),
    Tire = Color3.fromRGB(14, 17, 22),
    Glass = Color3.fromRGB(9, 27, 44),
}

local FORMS = {
    HUMANOIDE = "HUMANOIDE",
    MECHA = "MECHA",
    VEICULO = "VEÍCULO",
}

local form = FORMS.HUMANOIDE
local transition = nil
-- Encaminhador local: os botões chamam esta função sem poluir _G.
local requestForm = nil
local canRequestTransformation = nil
local emitAdaptiveBurst = nil
local stopLocalMovement = nil
local canApplyVelocity = nil
local ANIMATION_PROFILES = nil
local character = nil
local humanoid = nil
local hrp = nil
local avatarParts = {}
local descendantConnection = nil
local normalProperties = nil
local camera = Workspace.CurrentCamera
local initialFOV = camera and camera.FieldOfView or CFG.HUMANOID_FOV
CFG.HUMANOID_FOV = initialFOV
local savedCameraSubject = nil
local savedCameraType = camera and camera.CameraType or Enum.CameraType.Custom
local savedCameraMinZoom = player.CameraMinZoomDistance
local savedCameraMaxZoom = player.CameraMaxZoomDistance
local cameraFocusMode = nil
local lastGrounded = false
local lastFootstepAt = 0
local lastRootCF = nil
local destroyed = false
local walkPhase = 0
local wheelPhase = 0
local effectTick = 0

-- Estado runtime centralizado. Separar os comandos de toque do estado do
-- personagem evita que uma soltura de botão deixe o veículo acelerando.
local Runtime = {
    drive = {
        forward = false,
        reverse = false,
        left = false,
        right = false,
        boost = false,
        braking = false,
        throttle = 0,
        steering = 0,
        smoothedThrottle = 0,
        smoothedSteering = 0,
        currentSpeed = 0,
        maxObservedSpeed = 0,
        lastCommand = 0,
    },
    action = {
        name = "none",
        startedAt = 0,
        progress = 0,
        side = 1,
        cooldownUntil = 0,
    },
    quality = {
        level = 2,
        averageFrameTime = 1 / 60,
        samples = 0,
        sampleAccumulator = 0,
        lowFrameTime = 0,
        highFrameTime = 0,
        particleScale = 1,
        label = "BALANCEADO",
    },
    telemetry = {
        updateAccumulator = 0,
        displayedSpeed = 0,
        displayedHeading = 0,
        lastRootPosition = nil,
        recentDistance = 0,
        statusLine = "INICIALIZANDO",
    },
    transform = {
        stage = "IDLE",
        lastStage = "IDLE",
        lastCompletedAt = 0,
        sequenceId = 0,
        stageProgress = 0,
        pulse = 0,
    },
    camera = {
        shakeEnvelope = 0,
        shakePhase = 0,
        previousSubject = nil,
        desiredOffset = Vector3.zero,
    },
    cleanup = {
        connections = {},
        instances = {},
        controls = {},
        registered = false,
    },
}

---------------------------
-- 2. FUNÇÕES UTILITÁRIAS
---------------------------
local function safeCall(fn)
    local ok, result = pcall(fn)
    if not ok then
        warn("[Transformer Engine] " .. tostring(result))
    end
    return ok, result
end

local function clamp01(value)
    return math.clamp(value, 0, 1)
end

local function smoothstep(value)
    value = clamp01(value)
    return value * value * (3 - 2 * value)
end

local function angleCF(x, y, z)
    return CFrame.Angles(math.rad(x or 0), math.rad(y or 0), math.rad(z or 0))
end

-- Comparação textual simples usada tanto na criação quanto na animação das peças.
-- Declarada cedo para que o mapa semântico funcione já durante a inicialização.
local function nameContains(name, fragment)
    return string.find(string.lower(name), string.lower(fragment), 1, true) ~= nil
end

local function setPartBasics(part, name, size, color, material, transparency)
    part.Name = name
    part.Size = size
    part.Color = color or COLORS.Steel
    part.Material = material or Enum.Material.Metal
    part.Transparency = transparency or 0
    part.Anchored = true
    part.CanCollide = false
    part.CanTouch = false
    part.CanQuery = false
    part.CastShadow = part.Material ~= Enum.Material.Neon
    part.TopSurface = Enum.SurfaceType.Smooth
    part.BottomSurface = Enum.SurfaceType.Smooth
end

local function createVisualModel(name)
    local old = Workspace:FindFirstChild(name)
    if old then
        old:Destroy()
    end

    local model = Instance.new("Model")
    model.Name = name
    model.Parent = Workspace

    local root = Instance.new("Part")
    setPartBasics(root, "VisualRoot", Vector3.new(0.2, 0.2, 0.2), COLORS.Black, Enum.Material.SmoothPlastic, 1)
    root.CFrame = CFrame.new(0, -1000, 0)
    root.Parent = model
    model.PrimaryPart = root

    return model
end

local function newPart(model, list, name, size, cf, color, material, shape, group, pivot, transparency, className)
    local part
    if className == "WedgePart" then
        part = Instance.new("WedgePart")
    else
        part = Instance.new("Part")
        if shape then
            part.Shape = shape
        end
    end

    setPartBasics(part, name, size, color, material, 1)
    part.CFrame = cf
    part.Parent = model

    local index = #list + 1
    -- Dispersão determinística: não usa math.random dentro do loop de renderização.
    local scatterX = math.sin(index * 12.37 + #name) * (1.6 + (index % 3) * 0.55)
    local scatterY = math.cos(index * 8.13 + #name) * (1.2 + (index % 4) * 0.40)
    local scatterZ = math.sin(index * 5.71 + #name * 0.7) * (1.6 + (index % 5) * 0.35)
    local scatter = CFrame.new(scatterX, scatterY, scatterZ)
        * CFrame.Angles(math.rad((index * 31) % 130 - 65), math.rad((index * 47) % 180 - 90), math.rad((index * 23) % 100 - 50))

    local item = {
        part = part,
        name = name,
        baseCF = cf,
        baseTransparency = transparency or 0,
        group = group or "body",
        pivot = pivot or Vector3.zero,
        scatter = scatter,
        index = index,
    }
    table.insert(list, item)
    return part, item
end

local function addSparkEmitter(part, colorSequence, rate)
    local attachment = Instance.new("Attachment")
    attachment.Name = "GeneratedSparkAttachment"
    attachment.Parent = part

    local emitter = Instance.new("ParticleEmitter")
    emitter.Name = "GeneratedSparks"
    emitter.Texture = "rbxasset://textures/particles/sparkles_main.dds"
    emitter.Color = colorSequence or ColorSequence.new(COLORS.BlueLight, COLORS.Gold)
    emitter.LightEmission = 0.85
    emitter.LightInfluence = 0
    emitter.Lifetime = NumberRange.new(0.18, 0.55)
    emitter.Speed = NumberRange.new(3, 9)
    emitter.Rate = rate or CFG.FX_LOW_PARTICLE_RATE
    emitter.SpreadAngle = Vector2.new(180, 180)
    emitter.Rotation = NumberRange.new(0, 360)
    emitter.RotSpeed = NumberRange.new(-180, 180)
    emitter.Size = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0.19),
        NumberSequenceKeypoint.new(0.55, 0.10),
        NumberSequenceKeypoint.new(1, 0),
    })
    emitter.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0.05),
        NumberSequenceKeypoint.new(1, 1),
    })
    emitter.Enabled = false
    emitter.Parent = attachment
    return emitter
end

local function addSmokeEmitter(part)
    local attachment = Instance.new("Attachment")
    attachment.Name = "GeneratedSmokeAttachment"
    attachment.Position = Vector3.new(0, -0.25, 0)
    attachment.Parent = part

    local emitter = Instance.new("ParticleEmitter")
    emitter.Name = "GeneratedLandingSmoke"
    emitter.Texture = "rbxasset://textures/particles/smoke_main.dds"
    emitter.Color = ColorSequence.new(Color3.fromRGB(95, 108, 122), Color3.fromRGB(30, 36, 44))
    emitter.LightInfluence = 0.2
    emitter.Lifetime = NumberRange.new(0.45, 1.15)
    emitter.Speed = NumberRange.new(2, 5)
    emitter.Rate = 0
    emitter.SpreadAngle = Vector2.new(35, 35)
    emitter.Size = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0.6),
        NumberSequenceKeypoint.new(1, 2.1),
    })
    emitter.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0.25),
        NumberSequenceKeypoint.new(1, 1),
    })
    emitter.Enabled = false
    emitter.Parent = attachment
    return emitter
end

local function createTrail(part, color)
    local a0 = Instance.new("Attachment")
    a0.Name = "TrailAttachmentA"
    a0.Position = Vector3.new(0, 0.2, -0.35)
    a0.Parent = part

    local a1 = Instance.new("Attachment")
    a1.Name = "TrailAttachmentB"
    a1.Position = Vector3.new(0, -0.2, -0.35)
    a1.Parent = part

    local trail = Instance.new("Trail")
    trail.Name = "GeneratedMotionTrail"
    trail.Attachment0 = a0
    trail.Attachment1 = a1
    trail.Color = ColorSequence.new(color or COLORS.BlueLight)
    trail.LightEmission = 0.9
    trail.Lifetime = 0.16
    trail.MinLength = 0.05
    trail.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0.1),
        NumberSequenceKeypoint.new(1, 1),
    })
    trail.Enabled = false
    trail.Parent = part
    return trail
end

local function buildPointLight(parent, color, range, brightness)
    local light = Instance.new("PointLight")
    light.Color = color
    light.Range = range
    light.Brightness = brightness
    light.Shadows = false
    light.Parent = parent
    return light
end

---------------------------
-- 3. CONSTRUÇÃO PROCEDURAL DO MECHA
---------------------------
local mechaModel = createVisualModel("__TransformerEngine_Mecha_" .. player.UserId)
local mechaParts = {}
local mechaEmitters = {}
local mechaTrails = {}
local mechaCoreLight = nil
local mechaEyeParts = {}
local mechaSmokeEmitters = {}

local function buildMecha()
    local m = mechaModel
    local parts = mechaParts

    local function B(name, size, x, y, z, color, material, group, pivot, rx, ry, rz, shape, transparency, className)
        local cf = CFrame.new(x, y, z) * angleCF(rx, ry, rz)
        local p = newPart(m, parts, name, size, cf, color, material, shape, group, pivot, transparency, className)
        return p
    end

    local BODY = Vector3.zero
    local LSH = Vector3.new(-1.48, 2.30, 0)
    local RSH = Vector3.new(1.48, 2.30, 0)
    local LHIP = Vector3.new(-0.65, -0.68, 0)
    local RHIP = Vector3.new(0.65, -0.68, 0)
    local HEAD = Vector3.new(0, 3.85, 0)

    -- Coluna central, torso e placas abdominais.
    B("Torso_Core", Vector3.new(2.05, 2.15, 1.05), 0, 1.95, 0.05, COLORS.Dark, Enum.Material.Metal)
    B("Chest_Outer_Armor", Vector3.new(2.60, 1.28, 1.18), 0, 2.72, -0.03, COLORS.Blue, Enum.Material.Metal)
    B("Chest_Center_Plate", Vector3.new(0.72, 1.03, 0.18), 0, 2.63, -0.69, COLORS.Silver, Enum.Material.DiamondPlate)
    B("Chest_Left_Red_Panel", Vector3.new(0.77, 1.05, 0.22), -0.82, 2.70, -0.69, COLORS.Red, Enum.Material.Metal, "body", BODY, 0, 0, -7)
    B("Chest_Right_Red_Panel", Vector3.new(0.77, 1.05, 0.22), 0.82, 2.70, -0.69, COLORS.Red, Enum.Material.Metal, "body", BODY, 0, 0, 7)
    B("Windshield_Left", Vector3.new(0.47, 0.72, 0.09), -0.82, 2.78, -0.83, COLORS.BlueGlass, Enum.Material.Glass, "body", BODY, 0, 0, -8)
    B("Windshield_Right", Vector3.new(0.47, 0.72, 0.09), 0.82, 2.78, -0.83, COLORS.BlueGlass, Enum.Material.Glass, "body", BODY, 0, 0, 8)
    B("Chest_Upper_Brow", Vector3.new(2.35, 0.20, 0.20), 0, 3.36, -0.62, COLORS.Silver, Enum.Material.Metal)
    B("Abdominal_Plate_01", Vector3.new(1.55, 0.34, 0.83), 0, 1.27, -0.52, COLORS.Steel, Enum.Material.DiamondPlate)
    B("Abdominal_Plate_02", Vector3.new(1.38, 0.28, 0.78), 0, 0.91, -0.55, COLORS.Dark, Enum.Material.Metal)
    B("Abdominal_Plate_03", Vector3.new(1.18, 0.22, 0.70), 0, 0.62, -0.55, COLORS.Silver, Enum.Material.Metal)
    B("Waist_Belt", Vector3.new(1.92, 0.35, 1.03), 0, 0.39, 0.02, COLORS.Blue, Enum.Material.Metal)
    B("Belt_Core", Vector3.new(0.38, 0.25, 0.10), 0, 0.39, -0.57, COLORS.Gold, Enum.Material.Neon)

    local core = B("Energon_Core", Vector3.new(0.50, 0.50, 0.23), 0, 2.52, -0.91, COLORS.BlueLight, Enum.Material.Neon, "body", BODY, 0, 0, 0, Enum.PartType.Ball)
    mechaCoreLight = buildPointLight(core, COLORS.BlueLight, 13, 2.3)
    table.insert(mechaEmitters, addSparkEmitter(core, ColorSequence.new(COLORS.BlueLight, COLORS.White), 2))

    -- Costas e propulsores.
    B("Backpack_Main", Vector3.new(1.75, 1.90, 0.72), 0, 2.02, 0.82, COLORS.Dark, Enum.Material.Metal)
    B("Backpack_Left_Armor", Vector3.new(0.63, 1.45, 0.65), -0.67, 2.12, 1.04, COLORS.Blue, Enum.Material.Metal)
    B("Backpack_Right_Armor", Vector3.new(0.63, 1.45, 0.65), 0.67, 2.12, 1.04, COLORS.Blue, Enum.Material.Metal)
    B("Thruster_Left", Vector3.new(0.42, 1.15, 0.48), -0.67, 1.37, 1.13, COLORS.Steel, Enum.Material.Metal)
    B("Thruster_Right", Vector3.new(0.42, 1.15, 0.48), 0.67, 1.37, 1.13, COLORS.Steel, Enum.Material.Metal)
    B("Thruster_Left_Glow", Vector3.new(0.24, 0.15, 0.26), -0.67, 0.77, 1.13, COLORS.BlueLight, Enum.Material.Neon)
    B("Thruster_Right_Glow", Vector3.new(0.24, 0.15, 0.26), 0.67, 0.77, 1.13, COLORS.BlueLight, Enum.Material.Neon)
    B("Back_Spine", Vector3.new(0.36, 1.45, 0.28), 0, 1.95, 1.28, COLORS.Silver, Enum.Material.Metal)

    -- Cabeça: capacete, máscara, olhos e antenas.
    B("Head_Helmet", Vector3.new(0.88, 0.86, 0.74), 0, 3.98, 0, COLORS.Blue, Enum.Material.Metal, "head", HEAD)
    B("Head_Crown", Vector3.new(0.65, 0.26, 0.61), 0, 4.44, 0.01, COLORS.Silver, Enum.Material.Metal, "head", HEAD)
    B("Head_Faceplate", Vector3.new(0.62, 0.43, 0.20), 0, 3.78, -0.43, COLORS.Silver, Enum.Material.Metal, "head", HEAD)
    local eyeL = B("Eye_Left", Vector3.new(0.23, 0.10, 0.08), -0.22, 4.03, -0.405, COLORS.BlueLight, Enum.Material.Neon, "head", HEAD)
    local eyeR = B("Eye_Right", Vector3.new(0.23, 0.10, 0.08), 0.22, 4.03, -0.405, COLORS.BlueLight, Enum.Material.Neon, "head", HEAD)
    table.insert(mechaEyeParts, eyeL)
    table.insert(mechaEyeParts, eyeR)
    B("Head_Cheek_Left", Vector3.new(0.17, 0.45, 0.25), -0.44, 3.88, -0.22, COLORS.Red, Enum.Material.Metal, "head", HEAD)
    B("Head_Cheek_Right", Vector3.new(0.17, 0.45, 0.25), 0.44, 3.88, -0.22, COLORS.Red, Enum.Material.Metal, "head", HEAD)
    B("Head_Antenna_Left", Vector3.new(0.13, 0.56, 0.13), -0.39, 4.51, 0.04, COLORS.Gold, Enum.Material.Metal, "head", HEAD, 0, 0, -9)
    B("Head_Antenna_Right", Vector3.new(0.13, 0.56, 0.13), 0.39, 4.51, 0.04, COLORS.Gold, Enum.Material.Metal, "head", HEAD, 0, 0, 9)
    B("Head_Back_Panel", Vector3.new(0.60, 0.48, 0.25), 0, 4.00, 0.41, COLORS.Dark, Enum.Material.Metal, "head", HEAD)

    -- Ombros e braços articulados.
    B("Shoulder_Joint_L", Vector3.new(0.74, 0.74, 0.74), -1.47, 2.29, 0, COLORS.Silver, Enum.Material.Metal, "leftArm", LSH, 0, 0, 0, Enum.PartType.Ball)
    B("Shoulder_Joint_R", Vector3.new(0.74, 0.74, 0.74), 1.47, 2.29, 0, COLORS.Silver, Enum.Material.Metal, "rightArm", RSH, 0, 0, 0, Enum.PartType.Ball)
    B("Shoulder_Armor_L", Vector3.new(1.18, 1.06, 1.12), -1.76, 2.55, 0.03, COLORS.Red, Enum.Material.Metal, "leftArm", LSH, 0, 0, -6)
    B("Shoulder_Armor_R", Vector3.new(1.18, 1.06, 1.12), 1.76, 2.55, 0.03, COLORS.Red, Enum.Material.Metal, "rightArm", RSH, 0, 0, 6)
    B("Shoulder_Cap_L", Vector3.new(0.91, 0.22, 0.86), -1.79, 3.11, -0.02, COLORS.Silver, Enum.Material.DiamondPlate, "leftArm", LSH)
    B("Shoulder_Cap_R", Vector3.new(0.91, 0.22, 0.86), 1.79, 3.11, -0.02, COLORS.Silver, Enum.Material.DiamondPlate, "rightArm", RSH)
    B("Shoulder_Stripe_L", Vector3.new(0.17, 0.74, 0.12), -2.14, 2.55, -0.55, COLORS.Gold, Enum.Material.Neon, "leftArm", LSH)
    B("Shoulder_Stripe_R", Vector3.new(0.17, 0.74, 0.12), 2.14, 2.55, -0.55, COLORS.Gold, Enum.Material.Neon, "rightArm", RSH)
    B("UpperArm_L", Vector3.new(0.80, 1.13, 0.83), -1.89, 1.67, 0, COLORS.Blue, Enum.Material.Metal, "leftArm", LSH, 0, 0, -3)
    B("UpperArm_R", Vector3.new(0.80, 1.13, 0.83), 1.89, 1.67, 0, COLORS.Blue, Enum.Material.Metal, "rightArm", RSH, 0, 0, 3)
    B("UpperArm_OuterPlate_L", Vector3.new(0.22, 0.83, 0.69), -2.30, 1.68, -0.03, COLORS.Silver, Enum.Material.Metal, "leftArm", LSH)
    B("UpperArm_OuterPlate_R", Vector3.new(0.22, 0.83, 0.69), 2.30, 1.68, -0.03, COLORS.Silver, Enum.Material.Metal, "rightArm", RSH)
    B("Elbow_L", Vector3.new(0.62, 0.38, 0.66), -1.93, 1.00, 0, COLORS.Dark, Enum.Material.Metal, "leftArm", LSH, 0, 0, 0, Enum.PartType.Ball)
    B("Elbow_R", Vector3.new(0.62, 0.38, 0.66), 1.93, 1.00, 0, COLORS.Dark, Enum.Material.Metal, "rightArm", RSH, 0, 0, 0, Enum.PartType.Ball)
    B("Forearm_L", Vector3.new(0.91, 1.05, 0.94), -1.98, 0.50, -0.02, COLORS.Red, Enum.Material.Metal, "leftArm", LSH, 0, 0, -2)
    B("Forearm_R", Vector3.new(0.91, 1.05, 0.94), 1.98, 0.50, -0.02, COLORS.Red, Enum.Material.Metal, "rightArm", RSH, 0, 0, 2)
    B("Forearm_Guard_L", Vector3.new(0.96, 0.20, 0.88), -1.98, 0.81, -0.50, COLORS.Silver, Enum.Material.DiamondPlate, "leftArm", LSH)
    B("Forearm_Guard_R", Vector3.new(0.96, 0.20, 0.88), 1.98, 0.81, -0.50, COLORS.Silver, Enum.Material.DiamondPlate, "rightArm", RSH)
    B("Wrist_L", Vector3.new(0.62, 0.30, 0.62), -1.98, -0.07, -0.03, COLORS.Silver, Enum.Material.Metal, "leftArm", LSH)
    B("Wrist_R", Vector3.new(0.62, 0.30, 0.62), 1.98, -0.07, -0.03, COLORS.Silver, Enum.Material.Metal, "rightArm", RSH)
    local handL = B("Hand_L", Vector3.new(0.72, 0.55, 0.74), -1.98, -0.43, -0.08, COLORS.Dark, Enum.Material.Metal, "leftArm", LSH)
    local handR = B("Hand_R", Vector3.new(0.72, 0.55, 0.74), 1.98, -0.43, -0.08, COLORS.Dark, Enum.Material.Metal, "rightArm", RSH)
    for _, fingerSpec in ipairs({
        {"L", -1.98, "leftArm", LSH, -1},
        {"R", 1.98, "rightArm", RSH, 1},
    }) do
        local side, x, groupName, pivot, sign = fingerSpec[1], fingerSpec[2], fingerSpec[3], fingerSpec[4], fingerSpec[5]
        for finger = 1, 3 do
            local fx = x + (finger - 2) * 0.20
            B("Finger_" .. side .. "_" .. finger, Vector3.new(0.14, 0.29, 0.18), fx, -0.79, -0.12, COLORS.Silver, Enum.Material.Metal, groupName, pivot, 0, 0, 0)
        end
        B("Thumb_" .. side, Vector3.new(0.17, 0.30, 0.20), x + sign * 0.40, -0.45, -0.16, COLORS.Silver, Enum.Material.Metal, groupName, pivot, 0, 0, sign * 22)
    end
    table.insert(mechaEmitters, addSparkEmitter(handL, ColorSequence.new(COLORS.BlueLight, COLORS.Gold), 3))
    table.insert(mechaEmitters, addSparkEmitter(handR, ColorSequence.new(COLORS.BlueLight, COLORS.Gold), 3))
    table.insert(mechaTrails, createTrail(handL, COLORS.BlueLight))
    table.insert(mechaTrails, createTrail(handR, COLORS.BlueLight))
    local shoulderL = B("Shoulder_FX_Node_L", Vector3.new(0.18, 0.18, 0.18), -2.17, 2.50, -0.45, COLORS.BlueLight, Enum.Material.Neon, "leftArm", LSH, 0, 0, 0, Enum.PartType.Ball, 0.15)
    local shoulderR = B("Shoulder_FX_Node_R", Vector3.new(0.18, 0.18, 0.18), 2.17, 2.50, -0.45, COLORS.BlueLight, Enum.Material.Neon, "rightArm", RSH, 0, 0, 0, Enum.PartType.Ball, 0.15)
    table.insert(mechaEmitters, addSparkEmitter(shoulderL, ColorSequence.new(COLORS.Gold, COLORS.White), CFG.FX_PARTICLE_RATE))
    table.insert(mechaEmitters, addSparkEmitter(shoulderR, ColorSequence.new(COLORS.Gold, COLORS.White), CFG.FX_PARTICLE_RATE))

    -- Pernas: coxa, joelho, canela, pistões e pés grandes.
    B("Hip_Joint_L", Vector3.new(0.78, 0.78, 0.82), -0.65, -0.68, 0, COLORS.Silver, Enum.Material.Metal, "leftLeg", LHIP, 0, 0, 0, Enum.PartType.Ball)
    B("Hip_Joint_R", Vector3.new(0.78, 0.78, 0.82), 0.65, -0.68, 0, COLORS.Silver, Enum.Material.Metal, "rightLeg", RHIP, 0, 0, 0, Enum.PartType.Ball)
    B("Thigh_L", Vector3.new(0.98, 1.10, 1.00), -0.70, -1.34, 0.03, COLORS.Red, Enum.Material.Metal, "leftLeg", LHIP, 0, 0, -2)
    B("Thigh_R", Vector3.new(0.98, 1.10, 1.00), 0.70, -1.34, 0.03, COLORS.Red, Enum.Material.Metal, "rightLeg", RHIP, 0, 0, 2)
    B("Thigh_FrontPlate_L", Vector3.new(0.66, 0.70, 0.16), -0.70, -1.31, -0.54, COLORS.Silver, Enum.Material.DiamondPlate, "leftLeg", LHIP)
    B("Thigh_FrontPlate_R", Vector3.new(0.66, 0.70, 0.16), 0.70, -1.31, -0.54, COLORS.Silver, Enum.Material.DiamondPlate, "rightLeg", RHIP)
    B("Knee_L", Vector3.new(0.76, 0.52, 0.78), -0.70, -1.94, -0.04, COLORS.Dark, Enum.Material.Metal, "leftLeg", LHIP, 0, 0, 0, Enum.PartType.Ball)
    B("Knee_R", Vector3.new(0.76, 0.52, 0.78), 0.70, -1.94, -0.04, COLORS.Dark, Enum.Material.Metal, "rightLeg", RHIP, 0, 0, 0, Enum.PartType.Ball)
    B("Knee_Cap_L", Vector3.new(0.52, 0.36, 0.18), -0.70, -1.94, -0.48, COLORS.Gold, Enum.Material.Metal, "leftLeg", LHIP)
    B("Knee_Cap_R", Vector3.new(0.52, 0.36, 0.18), 0.70, -1.94, -0.48, COLORS.Gold, Enum.Material.Metal, "rightLeg", RHIP)
    B("Shin_L", Vector3.new(1.00, 1.22, 1.06), -0.73, -2.56, 0.05, COLORS.Blue, Enum.Material.Metal, "leftLeg", LHIP, 0, 0, -2)
    B("Shin_R", Vector3.new(1.00, 1.22, 1.06), 0.73, -2.56, 0.05, COLORS.Blue, Enum.Material.Metal, "rightLeg", RHIP, 0, 0, 2)
    B("Shin_Armor_L", Vector3.new(0.74, 0.74, 0.15), -0.73, -2.51, -0.55, COLORS.Silver, Enum.Material.DiamondPlate, "leftLeg", LHIP)
    B("Shin_Armor_R", Vector3.new(0.74, 0.74, 0.15), 0.73, -2.51, -0.55, COLORS.Silver, Enum.Material.DiamondPlate, "rightLeg", RHIP)
    B("Shin_Side_L", Vector3.new(0.20, 0.82, 0.65), -1.25, -2.51, 0.05, COLORS.Red, Enum.Material.Metal, "leftLeg", LHIP)
    B("Shin_Side_R", Vector3.new(0.20, 0.82, 0.65), 1.25, -2.51, 0.05, COLORS.Red, Enum.Material.Metal, "rightLeg", RHIP)
    B("Ankle_L", Vector3.new(0.67, 0.34, 0.70), -0.73, -3.20, 0.04, COLORS.Steel, Enum.Material.Metal, "leftLeg", LHIP)
    B("Ankle_R", Vector3.new(0.67, 0.34, 0.70), 0.73, -3.20, 0.04, COLORS.Steel, Enum.Material.Metal, "rightLeg", RHIP)
    B("Foot_L", Vector3.new(1.20, 0.48, 1.48), -0.78, -3.39, -0.34, COLORS.Dark, Enum.Material.Metal, "leftLeg", LHIP)
    B("Foot_R", Vector3.new(1.20, 0.48, 1.48), 0.78, -3.39, -0.34, COLORS.Dark, Enum.Material.Metal, "rightLeg", RHIP)
    B("Foot_Toe_Plate_L", Vector3.new(0.94, 0.22, 0.55), -0.78, -3.31, -0.85, COLORS.Red, Enum.Material.Metal, "leftLeg", LHIP)
    B("Foot_Toe_Plate_R", Vector3.new(0.94, 0.22, 0.55), 0.78, -3.31, -0.85, COLORS.Red, Enum.Material.Metal, "rightLeg", RHIP)
    B("Foot_Light_L", Vector3.new(0.22, 0.10, 0.08), -0.78, -3.31, -1.14, COLORS.BlueLight, Enum.Material.Neon, "leftLeg", LHIP)
    B("Foot_Light_R", Vector3.new(0.22, 0.10, 0.08), 0.78, -3.31, -1.14, COLORS.BlueLight, Enum.Material.Neon, "rightLeg", RHIP)
    B("Piston_L", Vector3.new(0.20, 0.95, 0.20), -0.34, -2.51, 0.58, COLORS.Silver, Enum.Material.Metal, "leftLeg", LHIP, 0, 0, -8, Enum.PartType.Cylinder)
    B("Piston_R", Vector3.new(0.20, 0.95, 0.20), 0.34, -2.51, 0.58, COLORS.Silver, Enum.Material.Metal, "rightLeg", RHIP, 0, 0, 8, Enum.PartType.Cylinder)

    -- Pneus nas laterais dos ombros para reforçar a leitura veículo/robô.
    B("Shoulder_Wheel_L", Vector3.new(0.95, 0.48, 0.95), -2.24, 2.23, 0.14, COLORS.Tire, Enum.Material.SmoothPlastic, "leftArm", LSH, 0, 0, 90, Enum.PartType.Cylinder)
    B("Shoulder_Wheel_R", Vector3.new(0.95, 0.48, 0.95), 2.24, 2.23, 0.14, COLORS.Tire, Enum.Material.SmoothPlastic, "rightArm", RSH, 0, 0, 90, Enum.PartType.Cylinder)
    B("Shoulder_Wheel_Hub_L", Vector3.new(0.43, 0.12, 0.43), -2.51, 2.23, 0.14, COLORS.Silver, Enum.Material.Metal, "leftArm", LSH, 0, 0, 90, Enum.PartType.Cylinder)
    B("Shoulder_Wheel_Hub_R", Vector3.new(0.43, 0.12, 0.43), 2.51, 2.23, 0.14, COLORS.Silver, Enum.Material.Metal, "rightArm", RSH, 0, 0, 90, Enum.PartType.Cylinder)

    -- Linhas de painel, parafusos e entradas de ar: muitos detalhes pequenos, mas baratos.
    for i = 1, 4 do
        local y = 2.98 - i * 0.12
        B("Chest_Vent_L_" .. i, Vector3.new(0.28, 0.045, 0.04), -1.04, y, -0.82, COLORS.Dark, Enum.Material.Metal)
        B("Chest_Vent_R_" .. i, Vector3.new(0.28, 0.045, 0.04), 1.04, y, -0.82, COLORS.Dark, Enum.Material.Metal)
    end
    for i = 1, 3 do
        B("Belt_Detail_" .. i, Vector3.new(0.13, 0.16, 0.05), (i - 2) * 0.24, 0.12, -0.55, COLORS.Gold, Enum.Material.Neon)
    end

    -- Trilhas curtas nos pulsos, habilitadas apenas em movimento intenso.
    local wristLeft = nil
    local wristRight = nil
    for _, item in ipairs(parts) do
        if item.name == "Wrist_L" then wristLeft = item.part end
        if item.name == "Wrist_R" then wristRight = item.part end
    end
    if wristLeft then table.insert(mechaTrails, createTrail(wristLeft, COLORS.BlueLight)) end
    if wristRight then table.insert(mechaTrails, createTrail(wristRight, COLORS.RedLight)) end

    -- Fumaça de pouso reutilizável nos pés.
    local footL = nil
    local footR = nil
    for _, item in ipairs(parts) do
        if item.name == "Foot_L" then footL = item.part end
        if item.name == "Foot_R" then footR = item.part end
    end
    if footL then table.insert(mechaSmokeEmitters, addSmokeEmitter(footL)) end
    if footR then table.insert(mechaSmokeEmitters, addSmokeEmitter(footR)) end
end

---------------------------
-- 4. CONSTRUÇÃO PROCEDURAL DO VEÍCULO
---------------------------
local vehicleModel = createVisualModel("__TransformerEngine_Vehicle_" .. player.UserId)
local vehicleParts = {}
local vehicleEmitters = {}
local vehicleLights = {}
local vehicleWheelPivots = {}

local function buildVehicle()
    local m = vehicleModel
    local parts = vehicleParts

    local function B(name, size, x, y, z, color, material, group, pivot, rx, ry, rz, shape, transparency, className)
        local cf = CFrame.new(x, y, z) * angleCF(rx, ry, rz)
        local p = newPart(m, parts, name, size, cf, color, material, shape, group, pivot, transparency, className)
        return p
    end

    B("Vehicle_Underframe", Vector3.new(2.85, 0.45, 5.60), 0, -0.55, 0.05, COLORS.Black, Enum.Material.Metal)
    B("Vehicle_Chassis", Vector3.new(3.20, 0.62, 6.05), 0, -0.30, 0, COLORS.Dark, Enum.Material.Metal)
    B("Vehicle_Cab_Main", Vector3.new(3.15, 1.25, 3.05), 0, 0.46, -0.73, COLORS.Red, Enum.Material.Metal)
    B("Vehicle_Hood", Vector3.new(3.08, 0.45, 1.92), 0, 0.51, -2.48, COLORS.Blue, Enum.Material.Metal)
    B("Vehicle_Hood_Center", Vector3.new(0.72, 0.13, 1.55), 0, 0.77, -2.45, COLORS.Silver, Enum.Material.DiamondPlate)
    B("Vehicle_Hood_Stripe_L", Vector3.new(0.16, 0.07, 1.45), -0.95, 0.76, -2.47, COLORS.Gold, Enum.Material.Neon)
    B("Vehicle_Hood_Stripe_R", Vector3.new(0.16, 0.07, 1.45), 0.95, 0.76, -2.47, COLORS.Gold, Enum.Material.Neon)
    B("Vehicle_Roof", Vector3.new(2.75, 0.34, 1.85), 0, 1.31, -0.54, COLORS.Blue, Enum.Material.Metal)
    B("Vehicle_Windshield", Vector3.new(2.55, 0.77, 0.10), 0, 0.92, -1.66, COLORS.Glass, Enum.Material.Glass, "body", Vector3.zero, -22, 0, 0, nil, 0.14)
    B("Vehicle_Rear_Window", Vector3.new(2.45, 0.55, 0.10), 0, 0.94, 0.36, COLORS.Glass, Enum.Material.Glass, "body", Vector3.zero, 14, 0, 0, nil, 0.20)
    B("Vehicle_Side_Window_L", Vector3.new(0.09, 0.56, 1.12), -1.59, 0.96, -0.62, COLORS.Glass, Enum.Material.Glass, "body", Vector3.zero, 0, 0, -5, nil, 0.16)
    B("Vehicle_Side_Window_R", Vector3.new(0.09, 0.56, 1.12), 1.59, 0.96, -0.62, COLORS.Glass, Enum.Material.Glass, "body", Vector3.zero, 0, 0, 5, nil, 0.16)

    -- Para-brisa com bordas e carroceria lateral.
    B("Vehicle_A_Pillar_L", Vector3.new(0.15, 0.91, 0.17), -1.35, 0.91, -1.52, COLORS.Silver, Enum.Material.Metal, "body", Vector3.zero, -20, 0, -12)
    B("Vehicle_A_Pillar_R", Vector3.new(0.15, 0.91, 0.17), 1.35, 0.91, -1.52, COLORS.Silver, Enum.Material.Metal, "body", Vector3.zero, -20, 0, 12)
    B("Vehicle_Door_L", Vector3.new(0.18, 0.68, 1.55), -1.61, 0.28, -0.15, COLORS.Blue, Enum.Material.Metal)
    B("Vehicle_Door_R", Vector3.new(0.18, 0.68, 1.55), 1.61, 0.28, -0.15, COLORS.Blue, Enum.Material.Metal)
    B("Vehicle_Door_Handle_L", Vector3.new(0.07, 0.07, 0.30), -1.73, 0.36, -0.20, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Door_Handle_R", Vector3.new(0.07, 0.07, 0.30), 1.73, 0.36, -0.20, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Front_Bumper", Vector3.new(3.25, 0.30, 0.32), 0, -0.18, -3.40, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Front_Grille", Vector3.new(1.12, 0.46, 0.10), 0, 0.05, -3.34, COLORS.Black, Enum.Material.Metal)
    B("Vehicle_Grille_Bar_1", Vector3.new(0.93, 0.055, 0.04), 0, 0.17, -3.405, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Grille_Bar_2", Vector3.new(0.93, 0.055, 0.04), 0, 0.04, -3.405, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Grille_Bar_3", Vector3.new(0.93, 0.055, 0.04), 0, -0.09, -3.405, COLORS.Silver, Enum.Material.Metal)
    local headL = B("Vehicle_Headlight_L", Vector3.new(0.52, 0.24, 0.12), -1.07, 0.15, -3.35, COLORS.BlueLight, Enum.Material.Neon)
    local headR = B("Vehicle_Headlight_R", Vector3.new(0.52, 0.24, 0.12), 1.07, 0.15, -3.35, COLORS.BlueLight, Enum.Material.Neon)
    table.insert(vehicleLights, buildPointLight(headL, COLORS.BlueLight, 9, 1.3))
    table.insert(vehicleLights, buildPointLight(headR, COLORS.BlueLight, 9, 1.3))
    B("Vehicle_Tail_L", Vector3.new(0.47, 0.25, 0.10), -1.22, 0.12, 3.06, COLORS.RedLight, Enum.Material.Neon)
    B("Vehicle_Tail_R", Vector3.new(0.47, 0.25, 0.10), 1.22, 0.12, 3.06, COLORS.RedLight, Enum.Material.Neon)
    B("Vehicle_Rear_Bumper", Vector3.new(3.14, 0.25, 0.30), 0, -0.20, 3.17, COLORS.Steel, Enum.Material.Metal)
    B("Vehicle_Side_Skirt_L", Vector3.new(0.18, 0.20, 2.55), -1.62, -0.33, 0.08, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Side_Skirt_R", Vector3.new(0.18, 0.20, 2.55), 1.62, -0.33, 0.08, COLORS.Silver, Enum.Material.Metal)
    B("Vehicle_Roof_Lightbar", Vector3.new(1.28, 0.10, 0.18), 0, 1.54, -0.45, COLORS.BlueLight, Enum.Material.Neon)
    B("Vehicle_Mirror_L", Vector3.new(0.30, 0.20, 0.24), -1.79, 0.80, -1.13, COLORS.Black, Enum.Material.Metal)
    B("Vehicle_Mirror_R", Vector3.new(0.30, 0.20, 0.24), 1.79, 0.80, -1.13, COLORS.Black, Enum.Material.Metal)
    B("Vehicle_Exhaust_L", Vector3.new(0.22, 0.62, 0.22), -1.00, -0.48, 2.75, COLORS.Silver, Enum.Material.Metal, "body", Vector3.zero, 90, 0, 0, Enum.PartType.Cylinder)
    B("Vehicle_Exhaust_R", Vector3.new(0.22, 0.62, 0.22), 1.00, -0.48, 2.75, COLORS.Silver, Enum.Material.Metal, "body", Vector3.zero, 90, 0, 0, Enum.PartType.Cylinder)
    local exhaustL = B("Vehicle_Exhaust_Glow_L", Vector3.new(0.12, 0.12, 0.12), -1.00, -0.78, 2.75, COLORS.BlueLight, Enum.Material.Neon, "body", Vector3.zero, 0, 0, 0, Enum.PartType.Ball)
    local exhaustR = B("Vehicle_Exhaust_Glow_R", Vector3.new(0.12, 0.12, 0.12), 1.00, -0.78, 2.75, COLORS.BlueLight, Enum.Material.Neon, "body", Vector3.zero, 0, 0, 0, Enum.PartType.Ball)
    table.insert(vehicleEmitters, addSparkEmitter(exhaustL, ColorSequence.new(COLORS.BlueLight, COLORS.White), 2))
    table.insert(vehicleEmitters, addSparkEmitter(exhaustR, ColorSequence.new(COLORS.BlueLight, COLORS.White), 2))

    -- Quatro rodas geradas em código, com cubos centrais e parafusos.
    local wheelSpecs = {
        {"wheelFL", -1.48, -0.82, -2.02, -1},
        {"wheelFR", 1.48, -0.82, -2.02, 1},
        {"wheelRL", -1.48, -0.82, 2.02, -1},
        {"wheelRR", 1.48, -0.82, 2.02, 1},
    }
    for _, spec in ipairs(wheelSpecs) do
        local group, x, y, z, side = spec[1], spec[2], spec[3], spec[4], spec[5]
        local pivot = Vector3.new(x, y, z)
        vehicleWheelPivots[group] = pivot
        B("Tire_" .. group, Vector3.new(1.30, 0.60, 1.30), x, y, z, COLORS.Tire, Enum.Material.SmoothPlastic, group, pivot, 0, 0, 90, Enum.PartType.Cylinder)
        B("Hub_" .. group, Vector3.new(0.72, 0.14, 0.72), x + side * 0.34, y, z, COLORS.Silver, Enum.Material.Metal, group, pivot, 0, 0, 90, Enum.PartType.Cylinder)
        B("Hub_Core_" .. group, Vector3.new(0.28, 0.16, 0.28), x + side * 0.43, y, z, COLORS.BlueLight, Enum.Material.Neon, group, pivot, 0, 0, 90, Enum.PartType.Cylinder)
        for bolt = 1, 6 do
            local a = (bolt / 6) * math.pi * 2
            local by = y + math.cos(a) * 0.24
            local bz = z + math.sin(a) * 0.24
            B("WheelBolt_" .. group .. "_" .. bolt, Vector3.new(0.09, 0.09, 0.09), x + side * 0.45, by, bz, COLORS.Gold, Enum.Material.Metal, group, pivot, 0, 0, 0, Enum.PartType.Ball)
        end
    end

    -- Iluminação inferior discreta.
    local underGlowL = B("Vehicle_UnderGlow_L", Vector3.new(0.38, 0.05, 2.10), -1.18, -0.70, 0.10, COLORS.BlueLight, Enum.Material.Neon, "body", Vector3.zero, 0, 0, 0, nil, 0.35)
    local underGlowR = B("Vehicle_UnderGlow_R", Vector3.new(0.38, 0.05, 2.10), 1.18, -0.70, 0.10, COLORS.BlueLight, Enum.Material.Neon, "body", Vector3.zero, 0, 0, 0, nil, 0.35)
    table.insert(vehicleLights, buildPointLight(underGlowL, COLORS.BlueLight, 7, 0.6))
    table.insert(vehicleLights, buildPointLight(underGlowR, COLORS.BlueLight, 7, 0.6))
end

safeCall(buildMecha)
safeCall(buildVehicle)

---------------------------
-- 4.1 PASSAGEM DE DETALHES PROCEDURAIS
-- Pequenos componentes são criados uma única vez. Eles não são instanciados
-- dentro dos loops de renderização; todos entram nas mesmas listas de pose.
---------------------------
local generatedDetailCount = 0
local function registerDetailPart(model, partList, name, size, localCF, color, material, group, pivot, transparency, shape)
    if generatedDetailCount >= CFG.DETAIL_PART_BUDGET_MOBILE then
        return nil
    end
    generatedDetailCount += 1
    local part = Instance.new("Part")
    part.Name = name
    part.Size = size
    part.Color = color
    part.Material = material or Enum.Material.Metal
    part.Shape = shape or Enum.PartType.Block
    part.Anchored = true
    part.CanCollide = false
    part.CanTouch = false
    part.CanQuery = false
    part.CastShadow = false
    part.TopSurface = Enum.SurfaceType.Smooth
    part.BottomSurface = Enum.SurfaceType.Smooth
    part.Transparency = 1
    part.CFrame = localCF
    part.Parent = model
    local index = #partList + 1
    local scatter = CFrame.new(
        math.sin(index * 7.13) * 2.0,
        math.cos(index * 9.71) * 1.65,
        math.sin(index * 3.31) * 2.0
    ) * CFrame.Angles(index * 0.11, index * 0.07, index * 0.09)
    local item = {
        part = part,
        name = name,
        baseCF = localCF,
        baseTransparency = transparency or 0,
        group = group or "body",
        pivot = pivot or Vector3.zero,
        scatter = scatter,
        index = index,
        detail = true,
    }
    table.insert(partList, item)
    return part
end

-- Cria uma fileira de parafusos com espaçamento uniforme em um painel.
local function buildBoltStrip(model, partList, prefix, origin, axis, count, spacing, boltSize, color, group, pivot)
    local created = {}
    count = math.max(1, math.floor(count))
    for i = 1, count do
        local offset = (i - (count + 1) * 0.5) * spacing
        local position = origin + axis * offset
        local cf = CFrame.new(position) * CFrame.Angles(0, 0, math.rad(0))
        local part = registerDetailPart(
            model,
            partList,
            prefix .. "_Bolt_" .. string.format("%02d", i),
            boltSize,
            cf,
            color,
            Enum.Material.Metal,
            group,
            pivot,
            0,
            Enum.PartType.Ball
        )
        table.insert(created, part)
    end
    return created
end

-- Painéis de armadura extra. Os pontos estão no espaço local dos modelos.
local function buildMechaDetailPass()
    local m = mechaModel
    local p = mechaParts
    local metal = COLORS.Silver
    local dark = COLORS.Dark
    local blue = COLORS.Blue
    local red = COLORS.Red
    local gold = COLORS.Gold

    -- Respiradores superiores: lâminas curtas, em vez de uma única caixa lisa.
    for row = 1, 3 do
        local z = -0.641 - row * 0.013
        for col = 1, 5 do
            local x = (col - 3) * 0.215
            registerDetailPart(m, p, "Chest_HeatVent_" .. row .. "_" .. col,
                Vector3.new(0.115, 0.045, 0.035),
                CFrame.new(x, 3.15 - row * 0.105, z) * CFrame.Angles(0, 0, math.rad((col - 3) * 2)),
                dark, Enum.Material.Metal, "body", Vector3.zero, 0)
        end
    end

    -- Quatro fileiras de parafusos no peitoral, em coordenadas assimétricas.
    for side = -1, 1, 2 do
        for row = 1, 4 do
            local x = side * (0.59 + (row % 2) * 0.06)
            local y = 2.38 + row * 0.19
            registerDetailPart(m, p, "Chest_Fastener_" .. side .. "_" .. row,
                Vector3.new(0.075, 0.075, 0.045), CFrame.new(x, y, -0.80),
                gold, Enum.Material.Metal, "body", Vector3.zero, 0, Enum.PartType.Ball)
        end
    end

    -- Costelas laterais com material alternado para separar volumes.
    for side = -1, 1, 2 do
        for row = 1, 5 do
            local y = 1.32 + row * 0.235
            registerDetailPart(m, p, "Torso_Rib_" .. side .. "_" .. row,
                Vector3.new(0.16, 0.12, 0.72),
                CFrame.new(side * (1.02 + (row % 2) * 0.025), y, -0.015)
                    * CFrame.Angles(0, 0, math.rad(side * 8)),
                row % 2 == 0 and metal or dark, Enum.Material.DiamondPlate,
                "body", Vector3.zero, 0)
        end
    end

    -- Placas de braço: sulcos, faixas e encaixes para que os braços não pareçam cilindros simples.
    for side = -1, 1, 2 do
        local label = side < 0 and "L" or "R"
        for row = 1, 5 do
            local y = 1.30 + row * 0.145
            registerDetailPart(m, p, "UpperArm_Groove_" .. label .. "_" .. row,
                Vector3.new(0.51, 0.035, 0.035),
                CFrame.new(side * 2.00, y, -0.424),
                row % 2 == 0 and COLORS.BlueLight or dark,
                row % 2 == 0 and Enum.Material.Neon or Enum.Material.Metal,
                side < 0 and "leftArm" or "rightArm",
                side < 0 and Vector3.new(-1.48, 2.30, 0) or Vector3.new(1.48, 2.30, 0),
                0)
        end
        for row = 1, 4 do
            local y = 0.18 + row * 0.145
            registerDetailPart(m, p, "Forearm_Ridge_" .. label .. "_" .. row,
                Vector3.new(0.59, 0.06, 0.065),
                CFrame.new(side * 1.98, y, -0.505)
                    * CFrame.Angles(0, 0, math.rad(side * 2)),
                row == 2 and red or metal, Enum.Material.Metal,
                side < 0 and "leftArm" or "rightArm",
                side < 0 and Vector3.new(-1.48, 2.30, 0) or Vector3.new(1.48, 2.30, 0), 0)
        end
    end

    -- Marcadores de junta nos cotovelos e pulsos.
    for side = -1, 1, 2 do
        local label = side < 0 and "L" or "R"
        for bolt = 1, 6 do
            local angle = bolt * math.pi / 3
            local px = side * 1.93
            local py = 1.0 + math.sin(angle) * 0.23
            local pz = math.cos(angle) * 0.23 - 0.35
            registerDetailPart(m, p, "Elbow_Rivet_" .. label .. "_" .. bolt,
                Vector3.new(0.075, 0.075, 0.075), CFrame.new(px, py, pz),
                gold, Enum.Material.Metal, side < 0 and "leftArm" or "rightArm",
                side < 0 and Vector3.new(-1.48, 2.30, 0) or Vector3.new(1.48, 2.30, 0),
                0, Enum.PartType.Ball)
        end
    end

    -- Blindagem de quadril, pernas e joelhos com geometria sobreposta.
    for side = -1, 1, 2 do
        local label = side < 0 and "L" or "R"
        local hip = side * 0.66
        for row = 1, 5 do
            registerDetailPart(m, p, "Thigh_Armor_Slat_" .. label .. "_" .. row,
                Vector3.new(0.47, 0.105, 0.12),
                CFrame.new(hip, -0.95 - row * 0.155, -0.45)
                    * CFrame.Angles(0, 0, math.rad(side * (row - 3) * 2)),
                row % 2 == 0 and blue or red, Enum.Material.Metal,
                side < 0 and "leftLeg" or "rightLeg",
                side < 0 and Vector3.new(-0.65, -0.68, 0) or Vector3.new(0.65, -0.68, 0), 0)
        end
        for bolt = 1, 4 do
            registerDetailPart(m, p, "Knee_Lock_" .. label .. "_" .. bolt,
                Vector3.new(0.095, 0.095, 0.06),
                CFrame.new(hip + side * 0.22, -2.20, -0.47),
                gold, Enum.Material.Metal,
                side < 0 and "leftLeg" or "rightLeg",
                side < 0 and Vector3.new(-0.65, -0.68, 0) or Vector3.new(0.65, -0.68, 0),
                0, Enum.PartType.Ball)
        end
    end

    -- Detalhe das costas: caixas de cabos e dissipadores pequenos.
    for side = -1, 1, 2 do
        for row = 1, 6 do
            registerDetailPart(m, p, "Back_HeatSink_" .. side .. "_" .. row,
                Vector3.new(0.34, 0.09, 0.10),
                CFrame.new(side * 0.43, 2.60 - row * 0.17, 1.20),
                row % 2 == 0 and metal or blue,
                Enum.Material.Metal,
                "body", Vector3.zero, 0)
        end
    end

    -- Antenas laterais pequenas e marcações de inspeção no capacete.
    for side = -1, 1, 2 do
        for row = 1, 3 do
            registerDetailPart(m, p, "Helmet_Insignia_" .. side .. "_" .. row,
                Vector3.new(0.07, 0.18, 0.035),
                CFrame.new(side * 0.36, 4.15 - row * 0.14, -0.405),
                row == 2 and gold or red, Enum.Material.Metal,
                "head", Vector3.new(0, 3.85, 0), 0)
        end
    end

    -- Placas de blindagem periféricas: este laço é executado uma vez na criação.
    for side = -1, 1, 2 do
        for row = 1, 4 do
            local z = (row - 2.5) * 0.26
            registerDetailPart(m, p, "Waist_Side_Plate_" .. side .. "_" .. row,
                Vector3.new(0.16, 0.23, 0.20),
                CFrame.new(side * 1.02, 0.31, z),
                row % 2 == 0 and red or metal, Enum.Material.DiamondPlate,
                "body", Vector3.zero, 0)
        end
    end
end

-- Detalhes do caminhão: grelhas, fechos, parafusos, reforços de chassi e eixos.
local function buildVehicleDetailPass()
    local m = vehicleModel
    local p = vehicleParts
    local dark = COLORS.Dark
    local steel = COLORS.Silver
    local red = COLORS.Red
    local blue = COLORS.Blue

    -- Lâminas internas na grade frontal, separadas para dar profundidade.
    for row = 1, 7 do
        local x = (row - 4) * 0.135
        registerDetailPart(m, p, "Grille_Vertical_" .. row,
            Vector3.new(0.045, 0.35, 0.025), CFrame.new(x, 0.045, -3.414),
            row % 2 == 0 and steel or dark, Enum.Material.Metal, "body", Vector3.zero, 0)
    end
    for side = -1, 1, 2 do
        local label = side < 0 and "L" or "R"
        -- Parafusos ao redor do farol.
        for bolt = 1, 4 do
            local x = side * (1.07 + ((bolt % 2) * 0.17))
            local y = 0.03 + (bolt > 2 and 0.22 or -0.08)
            registerDetailPart(m, p, "Headlight_Fastener_" .. label .. "_" .. bolt,
                Vector3.new(0.055, 0.055, 0.045), CFrame.new(x, y, -3.425),
                steel, Enum.Material.Metal, "body", Vector3.zero, 0, Enum.PartType.Ball)
        end
        -- Fileiras de ventilação laterais da cabine.
        for vent = 1, 6 do
            registerDetailPart(m, p, "Cab_Side_Vent_" .. label .. "_" .. vent,
                Vector3.new(0.035, 0.055, 0.48),
                CFrame.new(side * 1.622, 0.25 + vent * 0.085, 0.45)
                    * CFrame.Angles(math.rad(0), 0, math.rad(side * 1.5)),
                vent % 2 == 0 and steel or dark, Enum.Material.Metal,
                "body", Vector3.zero, 0)
        end
        -- Trilhos da parte inferior e reforços de chassi.
        for row = 1, 5 do
            registerDetailPart(m, p, "Chassis_Rail_" .. label .. "_" .. row,
                Vector3.new(0.12, 0.12, 0.74),
                CFrame.new(side * 1.32, -0.62, -1.55 + row * 0.72),
                row % 2 == 0 and steel or dark, Enum.Material.Metal,
                "body", Vector3.zero, 0)
        end
    end

    -- Travas e linhas do teto.
    for row = 1, 5 do
        registerDetailPart(m, p, "Roof_Rib_" .. row,
            Vector3.new(2.35, 0.055, 0.08),
            CFrame.new(0, 1.505, -1.15 + row * 0.25),
            row % 2 == 0 and steel or blue,
            Enum.Material.Metal, "body", Vector3.zero, 0)
    end
    -- Dobradiças das portas: os marcadores ficam alinhados com cada lateral.
    for side = -1, 1, 2 do
        local label = side < 0 and "L" or "R"
        for hinge = 1, 3 do
            registerDetailPart(m, p, "Door_Hinge_" .. label .. "_" .. hinge,
                Vector3.new(0.12, 0.15, 0.12),
                CFrame.new(side * 1.72, 0.06 + hinge * 0.20, -0.63),
                steel, Enum.Material.Metal, "body", Vector3.zero, 0)
        end
    end
    -- Peças de eixo estilizadas para dar leitura mecânica entre as rodas.
    for side = -1, 1, 2 do
        for axle = 1, 2 do
            registerDetailPart(m, p, "Axle_Bracket_" .. side .. "_" .. axle,
                Vector3.new(0.24, 0.25, 0.38),
                CFrame.new(side * 1.14, -0.79, axle == 1 and -2.02 or 2.02),
                red, Enum.Material.Metal, "body", Vector3.zero, 0)
        end
    end
    -- Pequenas marcações de segurança e emblemas sem texturas externas.
    for side = -1, 1, 2 do
        for stripe = 1, 4 do
            registerDetailPart(m, p, "Cab_Safety_Mark_" .. side .. "_" .. stripe,
                Vector3.new(0.10, 0.035, 0.16),
                CFrame.new(side * 1.60, -0.01 + stripe * 0.09, -1.35 + stripe * 0.13),
                stripe % 2 == 0 and COLORS.Gold or red,
                Enum.Material.Neon, "body", Vector3.zero, 0)
        end
    end
end

safeCall(buildMechaDetailPass)
safeCall(buildVehicleDetailPass)

---------------------------
-- 4.2 MAPA SEMÂNTICO ENTRE PEÇAS DE ROBÔ E VEÍCULO
-- As peças visuais são ligadas por função mecânica. Isso faz os componentes
-- começarem na posição de uma peça da forma anterior antes de encaixar na forma
-- nova, em vez de depender só de um aparecimento por transparência.
---------------------------
local function getSideFromName(name)
    if string.find(name, "_L", 1, true) or string.find(name, "Left", 1, true) then
        return -1
    end
    if string.find(name, "_R", 1, true) or string.find(name, "Right", 1, true) then
        return 1
    end
    return 0
end

local function sourcePoolForMechaItem(item)
    local name = item.name
    local group = item.group
    local pool = {}
    local function pushMatching(predicate)
        for _, source in ipairs(vehicleParts) do
            if source.part and source.part.Parent and predicate(source) then
                table.insert(pool, source)
            end
        end
    end

    if group == "head" or nameContains(name, "Head_") or nameContains(name, "Helmet") then
        pushMatching(function(source)
            return nameContains(source.name, "Roof")
                or nameContains(source.name, "Windshield")
                or nameContains(source.name, "Cab_Main")
        end)
    elseif group == "leftArm" or nameContains(name, "LeftArm") then
        pushMatching(function(source)
            return nameContains(source.name, "Door_L")
                or nameContains(source.name, "Mirror_L")
                or nameContains(source.name, "wheelFL")
                or nameContains(source.name, "wheelRL")
        end)
    elseif group == "rightArm" or nameContains(name, "RightArm") then
        pushMatching(function(source)
            return nameContains(source.name, "Door_R")
                or nameContains(source.name, "Mirror_R")
                or nameContains(source.name, "wheelFR")
                or nameContains(source.name, "wheelRR")
        end)
    elseif group == "leftLeg" then
        pushMatching(function(source)
            return nameContains(source.name, "wheelRL")
                or nameContains(source.name, "Chassis_Rail_L")
                or nameContains(source.name, "Axle_Bracket_-1_2")
        end)
    elseif group == "rightLeg" then
        pushMatching(function(source)
            return nameContains(source.name, "wheelRR")
                or nameContains(source.name, "Chassis_Rail_R")
                or nameContains(source.name, "Axle_Bracket_1_2")
        end)
    else
        pushMatching(function(source)
            return nameContains(source.name, "Cab_Main")
                or nameContains(source.name, "Chassis")
                or nameContains(source.name, "Hood")
                or nameContains(source.name, "Underframe")
        end)
    end
    if #pool == 0 then
        for _, source in ipairs(vehicleParts) do table.insert(pool, source) end
    end
    return pool
end

local function sourcePoolForVehicleItem(item)
    local name = item.name
    local pool = {}
    local function pushMatching(predicate)
        for _, source in ipairs(mechaParts) do
            if source.part and source.part.Parent and predicate(source) then
                table.insert(pool, source)
            end
        end
    end

    if nameContains(name, "Tire_") or nameContains(name, "Hub_") or nameContains(name, "WheelBolt") then
        pushMatching(function(source)
            return source.group == "leftLeg" or source.group == "rightLeg"
                or source.group == "leftArm" or source.group == "rightArm"
        end)
    elseif nameContains(name, "Roof") or nameContains(name, "Windshield")
        or nameContains(name, "Cab_Main") or nameContains(name, "Mirror") then
        pushMatching(function(source)
            return source.group == "head"
                or nameContains(source.name, "Chest_")
                or nameContains(source.name, "Shoulder_Armor")
        end)
    elseif nameContains(name, "Door") or nameContains(name, "Side_Window") then
        local side = getSideFromName(name)
        pushMatching(function(source)
            return side < 0 and source.group == "leftArm"
                or side > 0 and source.group == "rightArm"
                or source.group == "leftArm" or source.group == "rightArm"
        end)
    elseif nameContains(name, "Chassis") or nameContains(name, "Underframe")
        or nameContains(name, "Axle") then
        pushMatching(function(source)
            return source.group == "leftLeg" or source.group == "rightLeg"
                or nameContains(source.name, "Torso")
                or nameContains(source.name, "Abdominal")
        end)
    else
        pushMatching(function(source)
            return source.group == "body"
                or nameContains(source.name, "Chest_")
                or nameContains(source.name, "Waist")
        end)
    end
    if #pool == 0 then
        for _, source in ipairs(mechaParts) do table.insert(pool, source) end
    end
    return pool
end

local function chooseMigrationMate(item, candidatePool)
    if #candidatePool == 0 then return nil end
    local name = item.name or ""
    local hash = 0
    for index = 1, #name do
        hash = (hash * 31 + string.byte(name, index)) % 65521
    end
    local selectedIndex = (hash % #candidatePool) + 1
    return candidatePool[selectedIndex]
end

local function linkTransformationParts()
    for _, item in ipairs(mechaParts) do
        item.transformMate = chooseMigrationMate(item, sourcePoolForMechaItem(item))
        item.migrationTarget = FORMS.MECHA
    end
    for _, item in ipairs(vehicleParts) do
        item.transformMate = chooseMigrationMate(item, sourcePoolForVehicleItem(item))
        item.migrationTarget = FORMS.VEICULO
    end
end

safeCall(linkTransformationParts)

---------------------------
-- 5. EFEITOS DE TELA E INTERFACE MOBILE
---------------------------
local oldGui = playerGui:FindFirstChild("TransformerEngineUI")
if oldGui then oldGui:Destroy() end

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "TransformerEngineUI"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = false
screenGui.DisplayOrder = 35
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.Parent = playerGui

local blur = Lighting:FindFirstChild("TransformerEngine_Blur")
if blur then blur:Destroy() end
blur = Instance.new("BlurEffect")
blur.Name = "TransformerEngine_Blur"
blur.Size = 0
blur.Parent = Lighting

local hud = Instance.new("Frame")
hud.Name = "StatusPanel"
hud.AnchorPoint = Vector2.new(0.5, 0)
hud.Position = UDim2.new(0.5, 0, 0, 10)
hud.Size = UDim2.new(0, 285, 0, 62)
hud.BackgroundColor3 = Color3.fromRGB(8, 14, 24)
hud.BackgroundTransparency = 0.18
hud.BorderSizePixel = 0
hud.Parent = screenGui
local hudCorner = Instance.new("UICorner")
hudCorner.CornerRadius = UDim.new(0, 10)
hudCorner.Parent = hud
local hudStroke = Instance.new("UIStroke")
hudStroke.Color = COLORS.BlueLight
hudStroke.Thickness = 1.2
hudStroke.Transparency = 0.18
hudStroke.Parent = hud

local title = Instance.new("TextLabel")
title.Name = "EngineTitle"
title.BackgroundTransparency = 1
title.Position = UDim2.new(0, 12, 0, 5)
title.Size = UDim2.new(1, -24, 0, 22)
title.Font = Enum.Font.GothamBlack
title.Text = "TRANSFORMER ENGINE"
title.TextColor3 = COLORS.White
title.TextSize = 15
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = hud

local status = Instance.new("TextLabel")
status.Name = "StatusText"
status.BackgroundTransparency = 1
status.Position = UDim2.new(0, 12, 0, 27)
status.Size = UDim2.new(1, -24, 0, 18)
status.Font = Enum.Font.GothamMedium
status.Text = "SISTEMA ONLINE  •  HUMANOIDE"
status.TextColor3 = COLORS.BlueLight
status.TextSize = 10
status.TextXAlignment = Enum.TextXAlignment.Left
status.Parent = hud

local barBack = Instance.new("Frame")
barBack.Name = "ProgressBack"
barBack.Position = UDim2.new(0, 12, 1, -10)
barBack.Size = UDim2.new(1, -24, 0, 3)
barBack.BackgroundColor3 = Color3.fromRGB(38, 49, 64)
barBack.BorderSizePixel = 0
barBack.Visible = false
barBack.Parent = hud
local barFill = Instance.new("Frame")
barFill.Name = "ProgressFill"
barFill.Size = UDim2.new(0, 0, 1, 0)
barFill.BackgroundColor3 = COLORS.BlueLight
barFill.BorderSizePixel = 0
barFill.Parent = barBack

local buttonInfo = {
    {form = FORMS.MECHA, text = "ROBÔ", color = COLORS.Blue, key = Enum.KeyCode.T},
    {form = FORMS.VEICULO, text = "VEÍCULO", color = COLORS.Red, key = Enum.KeyCode.V},
    {form = FORMS.HUMANOIDE, text = "HUMANO", color = COLORS.Steel, key = Enum.KeyCode.H},
}
local buttons = {}

local function makeButton(info, index)
    local button = Instance.new("TextButton")
    button.Name = "TransformTo_" .. info.form
    button.AnchorPoint = Vector2.new(1, 0)
    button.Position = UDim2.new(1, -12, 0.39, (index - 1) * 55)
    button.Size = UDim2.new(0, 104, 0, 46)
    button.BackgroundColor3 = Color3.fromRGB(9, 17, 29)
    button.BackgroundTransparency = 0.10
    button.BorderSizePixel = 0
    button.AutoButtonColor = true
    button.Font = Enum.Font.GothamBlack
    button.Text = info.text
    button.TextColor3 = COLORS.White
    button.TextSize = 12
    button.Selectable = false
    button.Parent = screenGui

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 9)
    corner.Parent = button

    local stroke = Instance.new("UIStroke")
    stroke.Name = "ButtonStroke"
    stroke.Color = info.color
    stroke.Thickness = 1.4
    stroke.Transparency = 0.15
    stroke.Parent = button

    local line = Instance.new("Frame")
    line.Name = "Accent"
    line.BorderSizePixel = 0
    line.BackgroundColor3 = info.color
    line.Size = UDim2.new(0, 4, 0.70, 0)
    line.Position = UDim2.new(0, 6, 0.15, 0)
    line.Parent = button
    local lineCorner = Instance.new("UICorner")
    lineCorner.CornerRadius = UDim.new(1, 0)
    lineCorner.Parent = line

    button.Activated:Connect(function()
        if not destroyed then
            -- Botões Activated funcionam por toque e por mouse.
            if requestForm then
                requestForm(info.form)
            end
        end
    end)

    buttons[info.form] = {button = button, stroke = stroke, info = info}
end

for i, info in ipairs(buttonInfo) do
    makeButton(info, i)
end

local hint = Instance.new("TextLabel")
hint.Name = "TouchHint"
hint.AnchorPoint = Vector2.new(1, 0)
hint.Position = UDim2.new(1, -12, 0.39, 166)
hint.Size = UDim2.new(0, 150, 0, 22)
hint.BackgroundTransparency = 1
hint.Font = Enum.Font.Gotham
hint.Text = "TOQUE PARA TRANSFORMAR"
hint.TextColor3 = Color3.fromRGB(172, 192, 211)
hint.TextSize = 8
hint.TextXAlignment = Enum.TextXAlignment.Right
hint.Parent = screenGui

---------------------------
-- 5.1 HUD DE TELEMETRIA E PAINEL DE CONTROLE MOBILE
-- Todos os botões abaixo usam Activated/InputBegan/InputEnded, portanto
-- aceitam toque sem exigir teclado, mouse ou gamepad.
---------------------------
local telemetryPanel = Instance.new("Frame")
telemetryPanel.Name = "TelemetryPanel"
telemetryPanel.AnchorPoint = Vector2.new(0.5, 0)
telemetryPanel.Position = UDim2.new(0.5, 0, 0, 80)
telemetryPanel.Size = UDim2.new(0, 244, 0, 38)
telemetryPanel.BackgroundColor3 = Color3.fromRGB(6, 11, 19)
telemetryPanel.BackgroundTransparency = 0.28
telemetryPanel.BorderSizePixel = 0
telemetryPanel.Parent = screenGui
local telemetryCorner = Instance.new("UICorner")
telemetryCorner.CornerRadius = UDim.new(0, 8)
telemetryCorner.Parent = telemetryPanel
local telemetryStroke = Instance.new("UIStroke")
telemetryStroke.Color = COLORS.Steel
telemetryStroke.Thickness = 1
telemetryStroke.Transparency = 0.38
telemetryStroke.Parent = telemetryPanel

local telemetrySpeed = Instance.new("TextLabel")
telemetrySpeed.Name = "SpeedReadout"
telemetrySpeed.BackgroundTransparency = 1
telemetrySpeed.Position = UDim2.new(0, 9, 0, 3)
telemetrySpeed.Size = UDim2.new(0, 92, 0, 16)
telemetrySpeed.Font = Enum.Font.GothamBold
telemetrySpeed.Text = "SPD 000"
telemetrySpeed.TextColor3 = COLORS.White
telemetrySpeed.TextSize = 11
telemetrySpeed.TextXAlignment = Enum.TextXAlignment.Left
telemetrySpeed.Parent = telemetryPanel

local telemetryHeading = Instance.new("TextLabel")
telemetryHeading.Name = "HeadingReadout"
telemetryHeading.BackgroundTransparency = 1
telemetryHeading.Position = UDim2.new(1, -88, 0, 3)
telemetryHeading.Size = UDim2.new(0, 79, 0, 16)
telemetryHeading.Font = Enum.Font.GothamBold
telemetryHeading.Text = "HDG 000"
telemetryHeading.TextColor3 = COLORS.BlueLight
telemetryHeading.TextSize = 10
telemetryHeading.TextXAlignment = Enum.TextXAlignment.Right
telemetryHeading.Parent = telemetryPanel

local telemetryTrack = Instance.new("Frame")
telemetryTrack.Name = "SpeedTrack"
telemetryTrack.Position = UDim2.new(0, 9, 0, 23)
telemetryTrack.Size = UDim2.new(1, -18, 0, 5)
telemetryTrack.BackgroundColor3 = Color3.fromRGB(32, 42, 56)
telemetryTrack.BorderSizePixel = 0
telemetryTrack.Parent = telemetryPanel
local telemetryTrackCorner = Instance.new("UICorner")
telemetryTrackCorner.CornerRadius = UDim.new(1, 0)
telemetryTrackCorner.Parent = telemetryTrack
local telemetryFill = Instance.new("Frame")
telemetryFill.Name = "SpeedFill"
telemetryFill.Size = UDim2.new(0, 0, 1, 0)
telemetryFill.BackgroundColor3 = COLORS.BlueLight
telemetryFill.BorderSizePixel = 0
telemetryFill.Parent = telemetryTrack
local telemetryFillCorner = Instance.new("UICorner")
telemetryFillCorner.CornerRadius = UDim.new(1, 0)
telemetryFillCorner.Parent = telemetryFill

local stageReadout = Instance.new("TextLabel")
stageReadout.Name = "SystemReadout"
stageReadout.AnchorPoint = Vector2.new(0.5, 0)
stageReadout.Position = UDim2.new(0.5, 0, 0, 121)
stageReadout.Size = UDim2.new(0, 244, 0, 16)
stageReadout.BackgroundTransparency = 1
stageReadout.Font = Enum.Font.GothamMedium
stageReadout.Text = "SENSORES ONLINE  /  QUALIDADE BALANCEADA"
stageReadout.TextColor3 = Color3.fromRGB(142, 164, 184)
stageReadout.TextSize = 8
stageReadout.Parent = screenGui

local controlButtons = {}
local function makeControlButton(name, label, offsetX, offsetY, color, holdName, actionName)
    local button = Instance.new("TextButton")
    button.Name = "Control_" .. name
    button.AnchorPoint = Vector2.new(1, 0)
    button.Position = UDim2.new(1, -offsetX, 0.67, offsetY)
    button.Size = UDim2.new(0, 47, 0, 43)
    button.BackgroundColor3 = Color3.fromRGB(7, 14, 24)
    button.BackgroundTransparency = 0.12
    button.BorderSizePixel = 0
    button.AutoButtonColor = false
    button.Font = Enum.Font.GothamBlack
    button.Text = label
    button.TextColor3 = COLORS.White
    button.TextSize = 10
    button.Visible = false
    button.Selectable = false
    button.Parent = screenGui

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 9)
    corner.Parent = button

    local stroke = Instance.new("UIStroke")
    stroke.Color = color
    stroke.Thickness = 1.3
    stroke.Transparency = 0.2
    stroke.Parent = button

    local function pressed()
        button.BackgroundColor3 = Color3.fromRGB(26, 43, 60)
        button.Position = UDim2.new(1, -offsetX, 0.67, offsetY + 2)
        if holdName then
            Runtime.drive[holdName] = true
        end
        -- Ações de toque único são disparadas em Activated, abaixo, uma só vez.
        -- Os controles de manter pressionado atualizam apenas o estado de entrada.
    end

    local function released()
        button.BackgroundColor3 = Color3.fromRGB(7, 14, 24)
        button.Position = UDim2.new(1, -offsetX, 0.67, offsetY)
        if holdName then
            Runtime.drive[holdName] = false
        end
    end

    button.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch
            or input.UserInputType == Enum.UserInputType.MouseButton1 then
            pressed()
        end
    end)
    button.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch
            or input.UserInputType == Enum.UserInputType.MouseButton1 then
            released()
        end
    end)
    button.Activated:Connect(function()
        -- Activated serve como fallback para acessibilidade e interfaces
        -- em que InputBegan/Ended não foram encaminhados como esperado.
        if actionName then
            local fn = Runtime.actionHandlers and Runtime.actionHandlers[actionName]
            if fn then fn() end
        end
    end)

    controlButtons[name] = {
        button = button,
        stroke = stroke,
        setPressed = pressed,
        setReleased = released,
        holdName = holdName,
        actionName = actionName,
    }
    table.insert(Runtime.cleanup.controls, controlButtons[name])
    return button
end

-- Painel do veículo fica deslocado para a esquerda dos botões padrão de salto.
makeControlButton("SteerLeft", "◀", 198, 0, COLORS.BlueLight, "left")
makeControlButton("SteerRight", "▶", 146, 0, COLORS.BlueLight, "right")
makeControlButton("Throttle", "GÁS", 198, 49, COLORS.RedLight, "forward")
makeControlButton("Brake", "FREIO", 146, 49, COLORS.Gold, "braking")
makeControlButton("Boost", "BOOST", 198, 98, COLORS.BlueLight, "boost")
makeControlButton("Reverse", "RÉ", 146, 98, COLORS.Steel, "reverse")

-- Painel de ações do mecha. Não dá armas nem altera combate do jogo; são
-- movimentos cosméticos locais que também funcionam em experiências solo.
makeControlButton("Dash", "DASH", 198, 0, COLORS.BlueLight, nil, "dash")
makeControlButton("Punch", "SOCAR", 146, 0, COLORS.RedLight, nil, "punch")
makeControlButton("Stomp", "IMPACTO", 198, 49, COLORS.Gold, nil, "stomp")
makeControlButton("Jump", "SALTO", 146, 49, COLORS.Silver, nil, "jump")

local function setControlPanelVisibility(currentForm, transitioning)
    local vehicleVisible = currentForm == FORMS.VEICULO and not transitioning
    local mechaVisible = currentForm == FORMS.MECHA and not transitioning
    for name, entry in pairs(controlButtons) do
        local vehicleButton = name == "SteerLeft" or name == "SteerRight"
            or name == "Throttle" or name == "Brake" or name == "Boost" or name == "Reverse"
        entry.button.Visible = vehicleButton and vehicleVisible
            or (not vehicleButton and mechaVisible)
        if not entry.button.Visible then
            if entry.holdName then Runtime.drive[entry.holdName] = false end
            entry.setReleased()
        end
    end
end

---------------------------
-- 5.2 FUNÇÕES DO HUD, TELEMETRIA E CONTROLE DE TOQUE
---------------------------
local function formatSpeed(speed)
    local rounded = math.floor(math.max(0, speed) + 0.5)
    if rounded < 10 then return "00" .. tostring(rounded) end
    if rounded < 100 then return "0" .. tostring(rounded) end
    return tostring(math.min(999, rounded))
end

local function getHorizontalDirection(vector)
    local flat = Vector3.new(vector.X, 0, vector.Z)
    if flat.Magnitude < 0.001 then
        return Vector3.new(0, 0, -1)
    end
    return flat.Unit
end

local function getHeadingDegrees(cf)
    local look = cf.LookVector
    local heading = math.deg(math.atan2(-look.X, -look.Z))
    heading = heading % 360
    return math.floor(heading + 0.5)
end

local function updateTelemetry(dt)
    Runtime.telemetry.updateAccumulator += dt
    if Runtime.telemetry.updateAccumulator < CFG.TELEMETRY_INTERVAL then
        return
    end
    Runtime.telemetry.updateAccumulator = 0

    if not hrp or not hrp.Parent then
        telemetrySpeed.Text = "SPD ---"
        telemetryHeading.Text = "HDG ---"
        telemetryFill.Size = UDim2.new(0, 0, 1, 0)
        return
    end

    local measured = hrp.AssemblyLinearVelocity.Magnitude
    if not transition and form == FORMS.HUMANOIDE then
        measured = humanoid and humanoid.MoveDirection.Magnitude * humanoid.WalkSpeed or measured
    end
    if form == FORMS.VEICULO then
        measured = math.max(measured, humanoid and humanoid.MoveDirection.Magnitude * humanoid.WalkSpeed or 0)
    elseif form == FORMS.MECHA then
        measured = math.max(measured, humanoid and humanoid.MoveDirection.Magnitude * humanoid.WalkSpeed * 0.75 or 0)
    end

    Runtime.telemetry.displayedSpeed += (measured - Runtime.telemetry.displayedSpeed) * 0.38
    Runtime.telemetry.displayedHeading = getHeadingDegrees(hrp.CFrame)
    telemetrySpeed.Text = "SPD " .. formatSpeed(Runtime.telemetry.displayedSpeed)
    telemetryHeading.Text = "HDG " .. string.format("%03d", Runtime.telemetry.displayedHeading)

    local ceiling = form == FORMS.VEICULO and CFG.DRIVE_BOOST_SPEED
        or (form == FORMS.MECHA and CFG.MECHA_SPEED or CFG.HUMANOID_SPEED)
    local ratio = math.clamp(Runtime.telemetry.displayedSpeed / math.max(1, ceiling), 0, 1)
    telemetryFill.Size = UDim2.new(ratio, 0, 1, 0)
    if form == FORMS.VEICULO and Runtime.drive.boost then
        telemetryFill.BackgroundColor3 = COLORS.Gold
    elseif form == FORMS.MECHA then
        telemetryFill.BackgroundColor3 = COLORS.RedLight
    else
        telemetryFill.BackgroundColor3 = COLORS.BlueLight
    end
end

local function updateControlsVisibility()
    setControlPanelVisibility(form, transition ~= nil)
end

---------------------------
-- 6. AVATAR ORIGINAL: CAPTURA E VISIBILIDADE LOCAL
---------------------------
local function capturePart(part)
    if part:IsA("BasePart") and avatarParts[part] == nil then
        avatarParts[part] = part.LocalTransparencyModifier
    end
end

local function captureCharacter(char)
    if descendantConnection then
        descendantConnection:Disconnect()
        descendantConnection = nil
    end

    character = char
    humanoid = char:WaitForChild("Humanoid", 12)
    hrp = char:WaitForChild("HumanoidRootPart", 12)
    avatarParts = {}

    for _, descendant in ipairs(char:GetDescendants()) do
        capturePart(descendant)
    end

    descendantConnection = char.DescendantAdded:Connect(function(descendant)
        capturePart(descendant)
        if form ~= FORMS.HUMANOIDE and descendant:IsA("BasePart") then
            descendant.LocalTransparencyModifier = 1
        end
    end)

    if humanoid then
        if not savedCameraSubject then
            local currentCamera = Workspace.CurrentCamera
            savedCameraSubject = currentCamera and currentCamera.CameraSubject or humanoid
        end
        normalProperties = {
            WalkSpeed = humanoid.WalkSpeed,
            JumpPower = humanoid.JumpPower,
            JumpHeight = humanoid.JumpHeight,
            UseJumpPower = humanoid.UseJumpPower,
            AutoRotate = humanoid.AutoRotate,
            CameraOffset = humanoid.CameraOffset,
        }
    end
end

local function setAvatarVisible(visibleAmount)
    visibleAmount = clamp01(visibleAmount)
    for part, original in pairs(avatarParts) do
        if part and part.Parent then
            part.LocalTransparencyModifier = 1 - (1 - original) * visibleAmount
        end
    end
end

local function saveMovementIfNeeded()
    if humanoid and not normalProperties then
        normalProperties = {
            WalkSpeed = humanoid.WalkSpeed,
            JumpPower = humanoid.JumpPower,
            JumpHeight = humanoid.JumpHeight,
            UseJumpPower = humanoid.UseJumpPower,
            AutoRotate = humanoid.AutoRotate,
            CameraOffset = humanoid.CameraOffset,
        }
    end
end

local function applyFormMovement(target)
    if not humanoid then return end
    saveMovementIfNeeded()

    if target == FORMS.HUMANOIDE then
        if normalProperties then
            humanoid.WalkSpeed = normalProperties.WalkSpeed or CFG.HUMANOID_SPEED
            humanoid.UseJumpPower = normalProperties.UseJumpPower
            humanoid.JumpPower = normalProperties.JumpPower or 50
            humanoid.JumpHeight = normalProperties.JumpHeight or 7.2
            humanoid.AutoRotate = normalProperties.AutoRotate ~= false
            humanoid.CameraOffset = normalProperties.CameraOffset or Vector3.zero
        else
            humanoid.WalkSpeed = CFG.HUMANOID_SPEED
            humanoid.JumpPower = 50
            humanoid.JumpHeight = 7.2
            humanoid.AutoRotate = true
        end
    elseif target == FORMS.MECHA then
        humanoid.WalkSpeed = CFG.MECHA_SPEED
        humanoid.JumpPower = CFG.MECHA_JUMP_POWER
        humanoid.JumpHeight = CFG.MECHA_JUMP_HEIGHT
        humanoid.AutoRotate = true
        humanoid.CameraOffset = Vector3.new(0, 1.0, 0)
    elseif target == FORMS.VEICULO then
        humanoid.WalkSpeed = CFG.VEHICLE_SPEED
        humanoid.JumpPower = 0
        humanoid.JumpHeight = 0
        humanoid.AutoRotate = true
        humanoid.CameraOffset = Vector3.new(0, 0.15, 0)
    end
end

---------------------------
-- 7. ANIMAÇÃO PROCEDURAL DAS PEÇAS
---------------------------
local function groupTransform(item, mode, phase, moving, cameraLocalLook, now)
    local group = item.group
    local pivot = item.pivot or Vector3.zero
    local rx, ry, rz = 0, 0, 0
    local action = Runtime.action
    local actionProgress = action.progress or 0
    local actionPulse = math.sin(actionProgress * math.pi)
    local profiles = ANIMATION_PROFILES or {}
    local idleProfile = profiles.MechaIdle or {}
    local walkProfile = profiles.MechaWalk or {}
    local runProfile = profiles.MechaRun or {}
    local commandedSpeed = humanoid and humanoid.MoveDirection.Magnitude * humanoid.WalkSpeed or 0
    local isRunning = commandedSpeed >= CFG.MECHA_SPEED * 0.72
    local armAmplitude = isRunning and (runProfile.armAmplitude or CFG.RUN_SWAY)
        or (walkProfile.armAmplitude or CFG.WALK_SWAY)
    local legAmplitude = isRunning and (runProfile.legAmplitude or 0.61)
        or (walkProfile.legAmplitude or 0.46)
    local idleFrequency = idleProfile.frequency or 1.15

    if mode == FORMS.MECHA then
        if group == "leftArm" then
            rx = moving and math.sin(phase + math.pi) * armAmplitude or math.sin(now * idleFrequency) * 0.025
            rz = moving and -0.06 or math.sin(now * 0.8) * 0.018
            if action.name == "PUNCH" and action.side < 0 then
                rx = rx - actionPulse * 1.65
                rz = rz - actionPulse * 0.18
            elseif action.name == "DASH" then
                rx = rx + actionPulse * 0.22
            elseif action.name == "STOMP" then
                rz = rz - actionPulse * 0.10
            end
        elseif group == "rightArm" then
            rx = moving and math.sin(phase) * armAmplitude or math.sin(now * idleFrequency + 1) * 0.025
            rz = moving and 0.06 or math.sin(now * 0.8 + 1) * 0.018
            if action.name == "PUNCH" and action.side > 0 then
                rx = rx - actionPulse * 1.65
                rz = rz + actionPulse * 0.18
            elseif action.name == "DASH" then
                rx = rx + actionPulse * 0.22
            elseif action.name == "STOMP" then
                rz = rz + actionPulse * 0.10
            end
        elseif group == "leftLeg" then
            rx = moving and math.sin(phase) * legAmplitude or 0
            if action.name == "STOMP" then rx = rx - actionPulse * 0.62 end
            if action.name == "DASH" then rx = rx + actionPulse * 0.18 end
        elseif group == "rightLeg" then
            rx = moving and math.sin(phase + math.pi) * legAmplitude or 0
            if action.name == "STOMP" then rx = rx - actionPulse * 0.62 end
            if action.name == "DASH" then rx = rx + actionPulse * 0.18 end
        elseif group == "head" then
            ry = math.clamp(-cameraLocalLook.X * 0.43, -0.52, 0.52)
            rx = math.clamp(cameraLocalLook.Y * 0.20, -0.18, 0.18)
        elseif group == "body" then
            -- Respiração sutil no repouso e inclinação para frente quando acelera.
            local breath = moving and 0 or math.sin(now * 1.65) * 0.012
            rx = breath + (moving and 0.035 or 0)
            ry = math.sin(now * 0.57) * 0.008
            if action.name == "DASH" then rx = rx - actionPulse * 0.11 end
            if action.name == "STOMP" then rx = rx + actionPulse * 0.08 end
        end
    elseif mode == FORMS.VEICULO then
        if string.sub(group, 1, 5) == "wheel" then
            local spin = wheelPhase
            local steer = 0
            if group == "wheelFL" or group == "wheelFR" then
                steer = math.clamp(Runtime.drive.smoothedSteering * 0.30, -0.35, 0.35)
                if not moving then
                    steer = math.clamp(-cameraLocalLook.X * 0.12, -0.24, 0.24)
                end
            end
            local base = CFrame.new(pivot)
            return base * CFrame.Angles(spin, steer, 0) * CFrame.new(-pivot)
        elseif group == "body" then
            -- Pequeno balanço visual de suspensão; a posição vertical é aplicada
            -- no root no loop principal, enquanto estas rotações são locais.
            rx = moving and math.sin(phase * 0.5) * 0.008 or math.sin(now * 1.7) * 0.003
            rz = Runtime.drive.smoothedSteering * 0.025
            if Runtime.drive.boost then rx = rx - 0.012 end
        end
    end

    return CFrame.new(pivot) * CFrame.Angles(rx, ry, rz) * CFrame.new(-pivot)
end

---------------------------
-- 7.1 COREOGRAFIA MECÂNICA DA TRANSFORMAÇÃO
-- Em vez de somente alternar transparência, este estágio acrescenta movimento
-- nominal às peças selecionadas. O espalhamento global é mantido como uma
-- camada de montagem, e as articulações fazem pequenos movimentos de dobradiça.
---------------------------
local function segment01(value, first, last)
    if last <= first then return value >= last and 1 or 0 end
    return clamp01((value - first) / (last - first))
end

local function easeOutCubic(value)
    value = clamp01(value)
    return 1 - (1 - value) ^ 3
end

local function easeInOutSine(value)
    value = clamp01(value)
    return -(math.cos(math.pi * value) - 1) * 0.5
end

local function isArmoredPanelName(name)
    return nameContains(name, "armor")
        or nameContains(name, "plate")
        or nameContains(name, "panel")
        or nameContains(name, "guard")
        or nameContains(name, "rib")
        or nameContains(name, "groove")
        or nameContains(name, "vent")
end

local function getMechanicalPivot(item)
    if item.group == "leftArm" then
        return Vector3.new(-1.48, 2.30, 0)
    end
    if item.group == "rightArm" then
        return Vector3.new(1.48, 2.30, 0)
    end
    if item.group == "leftLeg" then
        return Vector3.new(-0.65, -0.68, 0)
    end
    if item.group == "rightLeg" then
        return Vector3.new(0.65, -0.68, 0)
    end
    if item.group == "head" then
        return Vector3.new(0, 3.85, 0)
    end
    return item.baseCF.Position
end

local function getVehiclePanelMotion(name, progress, direction)
    local open = math.sin(progress * math.pi)
    local rotation = Vector3.zero
    local displacement = Vector3.zero

    if nameContains(name, "Door") or nameContains(name, "Mirror") then
        local sign = string.find(name, "_L", 1, true) and -1 or 1
        rotation = Vector3.new(0, math.rad(sign * 24 * open), math.rad(sign * 7 * open))
        displacement = Vector3.new(sign * 0.12 * open, 0, 0)
    elseif nameContains(name, "Hood") then
        rotation = Vector3.new(math.rad(-13 * open), 0, 0)
        displacement = Vector3.new(0, 0.10 * open, -0.03 * open)
    elseif nameContains(name, "Roof") then
        rotation = Vector3.new(math.rad(6 * open), math.rad(2 * open), 0)
        displacement = Vector3.new(0, 0.04 * open, 0)
    elseif nameContains(name, "Bumper") then
        rotation = Vector3.new(math.rad(7 * open), 0, 0)
        displacement = Vector3.new(0, -0.06 * open, -0.03 * open)
    elseif nameContains(name, "Chassis") or nameContains(name, "Underframe") then
        rotation = Vector3.new(0, 0, math.rad(2.4 * open))
    end

    rotation *= direction
    displacement *= direction
    return CFrame.new(displacement) * CFrame.Angles(rotation.X, rotation.Y, rotation.Z)
end

local function getMechaPanelMotion(item, progress, direction)
    local name = item.name
    local open = math.sin(progress * math.pi)
    local rotate = Vector3.zero
    local shift = Vector3.zero
    local side = string.find(name, "_L", 1, true) and -1
        or (string.find(name, "_R", 1, true) and 1 or 0)

    if nameContains(name, "Chest_Left") or nameContains(name, "Chest_Right") then
        rotate = Vector3.new(0, math.rad(14 * open), math.rad((side == 0 and 1 or side) * 8 * open))
        shift = Vector3.new((side == 0 and 0 or side) * 0.10 * open, 0.02 * open, 0)
    elseif nameContains(name, "Shoulder_Armor") or nameContains(name, "Shoulder_Cap") then
        rotate = Vector3.new(0, math.rad(side * 12 * open), math.rad(side * 10 * open))
        shift = Vector3.new(side * 0.08 * open, 0.05 * open, 0)
    elseif nameContains(name, "UpperArm") or nameContains(name, "Forearm") then
        rotate = Vector3.new(math.rad(7 * open), 0, math.rad(side * 5 * open))
    elseif nameContains(name, "Backpack") or nameContains(name, "Thruster") then
        rotate = Vector3.new(math.rad(8 * open), math.rad(side * 4 * open), 0)
        shift = Vector3.new(0, 0, 0.08 * open)
    elseif nameContains(name, "Head_Crown") or nameContains(name, "Head_Antenna") then
        rotate = Vector3.new(math.rad(-4 * open), 0, math.rad(side * 5 * open))
        shift = Vector3.new(0, 0.035 * open, 0)
    elseif nameContains(name, "Thigh") or nameContains(name, "Knee") then
        rotate = Vector3.new(math.rad(5 * open), 0, math.rad(side * 3 * open))
    elseif nameContains(name, "Finger") or nameContains(name, "Thumb") then
        rotate = Vector3.new(math.rad(8 * open), 0, math.rad(side * 3 * open))
    elseif nameContains(name, "Chest_HeatVent") or nameContains(name, "Back_HeatSink") then
        shift = Vector3.new(0, 0, 0.035 * open)
    elseif isArmoredPanelName(name) then
        rotate = Vector3.new(math.rad(2 * open), math.rad(side * 3 * open), 0)
    end

    rotate *= direction
    shift *= direction
    local pivot = getMechanicalPivot(item)
    return CFrame.new(shift)
        * CFrame.new(pivot)
        * CFrame.Angles(rotate.X, rotate.Y, rotate.Z)
        * CFrame.new(-pivot)
end

local function transitionArticulation(item, mode, now)
    if not transition then
        return CFrame.new()
    end
    local p = clamp01(transition.progress or 0)
    local isTarget = mode == transition.to
    local t = isTarget and p or (1 - p)
    local direction = isTarget and 1 or -1
    local selected = item.part
    if not selected or not selected.Parent then
        return CFrame.new()
    end

    -- Onda de destravamento e de encaixe. O pequeno atraso por índice simula
    -- uma transmissão de movimento ao longo das placas sem usar Tween por peça.
    local lag = (item.index % 7) * 0.012
    local localProgress = easeOutCubic(segment01(t, lag, 1))
    local openPhase = math.sin(localProgress * math.pi)
    if mode == FORMS.MECHA then
        return getMechaPanelMotion(item, openPhase, direction)
    elseif mode == FORMS.VEICULO then
        local pivot = item.baseCF.Position
        local motion = getVehiclePanelMotion(item.name, openPhase, direction)
        return CFrame.new(pivot) * motion * CFrame.new(-pivot)
    end
    return CFrame.new()
end

local function getTransitionStage(progress)
    if progress < CFG.TRANSFORM_STAGE_ANTICIPATION then
        return "ANTECIPAÇÃO"
    elseif progress < CFG.TRANSFORM_STAGE_ANTICIPATION + CFG.TRANSFORM_STAGE_UNFOLD then
        return "DESTRAVANDO PLACAS"
    elseif progress < CFG.TRANSFORM_STAGE_ANTICIPATION + CFG.TRANSFORM_STAGE_UNFOLD + CFG.TRANSFORM_STAGE_ROTATE then
        return "RECONFIGURANDO EIXOS"
    elseif progress < 1 then
        return "TRAVANDO ARTICULAÇÕES"
    end
    return "CONCLUÍDO"
end

local function updateModel(parts, rootCF, visible, assemble, mode, phase, moving, cameraLocalLook, now, bob)
    visible = clamp01(visible)
    assemble = clamp01(assemble)
    if visible <= 0.002 then
        -- Continua oculto sem recalcular centenas de CFrames.
        for _, item in ipairs(parts) do
            if item.part and item.part.Parent and item.part.Transparency < 0.999 then
                item.part.Transparency = 1
            end
        end
        return
    end

    local rootMotion = CFrame.new(0, bob or 0, 0)
    for _, item in ipairs(parts) do
        local p = item.part
        if p and p.Parent then
            local spread = item.scatter:Lerp(CFrame.new(), assemble)
            local pose = groupTransform(item, mode, phase, moving, cameraLocalLook, now)
            local mechanical = transitionArticulation(item, mode, now)
            local targetCF = rootCF * rootMotion * spread * pose * mechanical * item.baseCF
            local migrating = transition ~= nil
                and transition.to == mode
                and transition.from ~= FORMS.HUMANOIDE
                and item.transformMate ~= nil
                and transition.sourceSnapshots ~= nil
            if migrating then
                local migrationStart = transition.sourceSnapshots[item.transformMate]
                if migrationStart then
                    local migrationAlpha = easeInOutSine(assemble)
                    p.CFrame = migrationStart:Lerp(targetCF, migrationAlpha)
                else
                    p.CFrame = targetCF
                end
            else
                p.CFrame = targetCF
            end
            p.Transparency = 1 - visible * (1 - item.baseTransparency)
        end
    end
end


local function setEffectsEnabled(enabled, vehicleEnabled)
    for _, emitter in ipairs(mechaEmitters) do
        if emitter and emitter.Parent then emitter.Enabled = enabled end
    end
    for _, trail in ipairs(mechaTrails) do
        if trail and trail.Parent then trail.Enabled = enabled and vehicleEnabled == false end
    end
    for _, emitter in ipairs(mechaSmokeEmitters) do
        if emitter and emitter.Parent then emitter.Enabled = false end
    end
    for _, emitter in ipairs(vehicleEmitters) do
        if emitter and emitter.Parent then emitter.Enabled = enabled and vehicleEnabled == true end
    end
end

local function emitTransformationBurst()
    for _, emitter in ipairs(mechaEmitters) do
        if emitter and emitter.Parent then safeCall(function() emitter:Emit(7) end) end
    end
    for _, emitter in ipairs(vehicleEmitters) do
        if emitter and emitter.Parent then safeCall(function() emitter:Emit(5) end) end
    end
    for _, emitter in ipairs(mechaSmokeEmitters) do
        if emitter and emitter.Parent then safeCall(function() emitter:Emit(10) end) end
    end
end

---------------------------
-- 8. CONTROLE DE ESTADO E TRANSFORMAÇÃO
---------------------------
local function updateButtonStates()
    for target, entry in pairs(buttons) do
        local selected = (not transition and form == target)
        entry.stroke.Thickness = selected and 2.6 or 1.2
        entry.stroke.Transparency = selected and 0 or 0.22
        entry.button.BackgroundColor3 = selected and Color3.fromRGB(21, 40, 60) or Color3.fromRGB(9, 17, 29)
        entry.button.TextTransparency = transition and 0.28 or 0
    end
    updateControlsVisibility()
end

requestForm = function(target)
    if canRequestTransformation then
        local allowed, reason = canRequestTransformation(target)
        if not allowed then
            if reason == "AVATAR_NÃO_PRONTO" then
                status.Text = "AGUARDANDO AVATAR..."
                status.TextColor3 = COLORS.Gold
            end
            return
        end
    else
        if destroyed or transition or not buttons[target] or target == form then return end
        if not humanoid or not hrp or humanoid.Health <= 0 then
            status.Text = "AGUARDANDO AVATAR..."
            status.TextColor3 = COLORS.Gold
            return
        end
    end

    saveMovementIfNeeded()
    local sourceParts = form == FORMS.MECHA and mechaParts
        or (form == FORMS.VEICULO and vehicleParts or nil)
    local sourceSnapshots = {}
    if sourceParts then
        for _, sourceItem in ipairs(sourceParts) do
            if sourceItem.part and sourceItem.part.Parent then
                sourceSnapshots[sourceItem] = sourceItem.part.CFrame
            end
        end
    end
    transition = {
        from = form,
        to = target,
        startTime = os.clock(),
        duration = CFG.TRANSFORM_TIME,
        progress = 0,
        sourceSnapshots = sourceSnapshots,
        sequenceId = Runtime.transform.sequenceId + 1,
        lastFxStage = "ANTECIPAÇÃO",
    }
    Runtime.transform.sequenceId += 1
    Runtime.transform.stage = "ANTECIPAÇÃO"
    Runtime.transform.stageProgress = 0
    Runtime.action.name = "none"
    Runtime.action.progress = 0
    for _, key in ipairs({"forward", "reverse", "left", "right", "boost", "braking"}) do
        Runtime.drive[key] = false
    end

    -- Imobilização suave durante o processo, sem ancorar o personagem.
    humanoid.WalkSpeed = 0
    humanoid.JumpPower = 0
    humanoid.JumpHeight = 0
    if stopLocalMovement then
        stopLocalMovement()
    else
        humanoid:Move(Vector3.zero, true)
    end
    if hrp then
        hrp.AssemblyLinearVelocity = Vector3.zero
        hrp.AssemblyAngularVelocity = Vector3.zero
        lastRootCF = hrp.CFrame
    end

    status.Text = "SISTEMA DE TRANSFORMAÇÃO ATIVADO..."
    status.TextColor3 = COLORS.Gold
    barBack.Visible = true
    barFill.Size = UDim2.new(0, 0, 1, 0)
    updateButtonStates()
    setEffectsEnabled(true, target == FORMS.VEICULO)
    emitTransformationBurst()
end

local function completeTransition()
    if not transition then return end
    local target = transition.to
    form = target
    transition = nil
    Runtime.transform.stage = "IDLE"
    Runtime.transform.lastCompletedAt = os.clock()
    Runtime.action.name = "none"
    Runtime.action.progress = 0
    applyFormMovement(form)

    if form == FORMS.HUMANOIDE then
        setAvatarVisible(1)
        status.Text = "SISTEMA ONLINE  •  HUMANOIDE"
        status.TextColor3 = COLORS.BlueLight
    elseif form == FORMS.MECHA then
        setAvatarVisible(0)
        status.Text = "MODO MECHA  •  NÚCLEO ENERGON ATIVO"
        status.TextColor3 = COLORS.BlueLight
        for _, emitter in ipairs(mechaSmokeEmitters) do
            if emitter and emitter.Parent then safeCall(function() emitter:Emit(8) end) end
        end
    else
        setAvatarVisible(0)
        status.Text = "MODO VEÍCULO  •  PROPULSÃO ONLINE"
        status.TextColor3 = COLORS.RedLight
    end

    barBack.Visible = false
    barFill.Size = UDim2.new(0, 0, 1, 0)
    blur.Size = 0
    setEffectsEnabled(false, form == FORMS.VEICULO)
    updateButtonStates()
end

---------------------------
-- 8.1 AÇÕES DO MECHA E ROTINA DE CONTROLE
-- As ações não criam instâncias por quadro. Os efeitos usam emitters que já
-- existem, e os impulsos alteram apenas a física local do avatar original.
---------------------------
local function actionIsAvailable()
    if destroyed or transition or form ~= FORMS.MECHA then
        return false
    end
    if not humanoid or not hrp or not humanoid.Parent or humanoid.Health <= 0 then
        return false
    end
    return os.clock() >= (Runtime.action.cooldownUntil or 0)
end

local function beginMechaAction(name, duration, cooldown)
    if not actionIsAvailable() then
        return false
    end
    Runtime.action.name = name
    Runtime.action.startedAt = os.clock()
    Runtime.action.progress = 0
    Runtime.action.cooldownUntil = os.clock() + (cooldown or duration)
    Runtime.action.side = Runtime.action.side or 1
    return true
end

local function getFacingVector()
    if not hrp then return Vector3.new(0, 0, -1) end
    return getHorizontalDirection(hrp.CFrame.LookVector)
end

local function performDash()
    if canApplyVelocity and not canApplyVelocity() then return end
    if not beginMechaAction("DASH", CFG.MECHA_DASH_TIME, 1.05) then
        return
    end
    local facing = getFacingVector()
    local oldVelocity = hrp.AssemblyLinearVelocity
    local dashVelocity = facing * CFG.MECHA_DASH_SPEED
    hrp.AssemblyLinearVelocity = Vector3.new(
        dashVelocity.X,
        math.max(oldVelocity.Y, 2.0),
        dashVelocity.Z
    )
    for _, emitter in ipairs(mechaEmitters) do
        if emitter and emitter.Parent then
            safeCall(function() emitter:Emit(5) end)
        end
    end
    Runtime.camera.shakeEnvelope = math.max(Runtime.camera.shakeEnvelope, 0.42)
end

local function performPunch()
    if not beginMechaAction("PUNCH", CFG.MECHA_PUNCH_TIME, 0.62) then
        return
    end
    Runtime.action.side = -Runtime.action.side
    for _, trail in ipairs(mechaTrails) do
        if trail and trail.Parent then trail.Enabled = true end
    end
    Runtime.camera.shakeEnvelope = math.max(Runtime.camera.shakeEnvelope, 0.18)
end

local function performStomp()
    if not beginMechaAction("STOMP", 0.72, 1.30) then
        return
    end
    if humanoid and humanoid.FloorMaterial ~= Enum.Material.Air then
        humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
    end
    for _, emitter in ipairs(mechaSmokeEmitters) do
        if emitter and emitter.Parent then
            safeCall(function() emitter:Emit(5) end)
        end
    end
    Runtime.camera.shakeEnvelope = math.max(Runtime.camera.shakeEnvelope, 0.32)
end

local function performMechaJump()
    if not actionIsAvailable() then return end
    if humanoid then
        humanoid.Jump = true
        humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
    end
    Runtime.camera.shakeEnvelope = math.max(Runtime.camera.shakeEnvelope, 0.12)
end

Runtime.actionHandlers = {
    dash = performDash,
    punch = performPunch,
    stomp = performStomp,
    jump = performMechaJump,
}

local function isDirectVehicleControlActive()
    local drive = Runtime.drive
    return drive.forward or drive.reverse or drive.left or drive.right
        or drive.boost or drive.braking
end

local function clearVehicleControls()
    local drive = Runtime.drive
    drive.forward = false
    drive.reverse = false
    drive.left = false
    drive.right = false
    drive.boost = false
    drive.braking = false
    drive.throttle = 0
    drive.steering = 0
    drive.smoothedThrottle = 0
    drive.smoothedSteering = 0
end

local function calculateDriveInputs()
    local drive = Runtime.drive
    local forward = drive.forward and 1 or 0
    local reverse = drive.reverse and -1 or 0
    local brake = drive.braking and 1 or 0
    local left = drive.left and -1 or 0
    local right = drive.right and 1 or 0
    drive.throttle = math.clamp(forward + reverse, -1, 1)
    drive.steering = math.clamp(left + right, -1, 1)
    if brake > 0 then drive.throttle = 0 end
end

local function smoothDriveInputs(dt)
    local drive = Runtime.drive
    local acceleration = math.clamp(dt * 7.5, 0, 1)
    local steeringAcceleration = math.clamp(dt * 5.5, 0, 1)
    drive.smoothedThrottle += (drive.throttle - drive.smoothedThrottle) * acceleration
    drive.smoothedSteering += (drive.steering - drive.smoothedSteering) * steeringAcceleration
end

local function horizontalCameraBasis()
    local currentCamera = Workspace.CurrentCamera
    local look = currentCamera and currentCamera.CFrame.LookVector or getFacingVector()
    local forward = getHorizontalDirection(look)
    local right = Vector3.new(-forward.Z, 0, forward.X)
    return forward, right
end

local function updateVehicleMovement(dt)
    if not humanoid or not hrp or form ~= FORMS.VEICULO or transition then
        return
    end

    local drive = Runtime.drive
    calculateDriveInputs()
    smoothDriveInputs(dt)

    local directInput = isDirectVehicleControlActive()
    local targetSpeed = CFG.DRIVE_MAX_SPEED
    if drive.boost then
        targetSpeed = CFG.DRIVE_BOOST_SPEED
    end
    if drive.reverse then
        targetSpeed = CFG.DRIVE_REVERSE_SPEED
    end
    if drive.braking then
        targetSpeed = 7
    end

    if directInput then
        local cameraForward, cameraRight = horizontalCameraBasis()
        local throttle = drive.smoothedThrottle
        local steering = drive.smoothedSteering
        local moveVector = Vector3.zero
        if math.abs(throttle) > 0.03 then
            -- Direção só altera o vetor quando há aceleração ou ré. Segurar
            -- apenas esquerda/direita não deve fazer o caminhão disparar de lado.
            moveVector = cameraForward * throttle
                + cameraRight * steering * math.abs(throttle) * 0.55
        end
        if moveVector.Magnitude > 0.03 then
            humanoid:Move(moveVector.Unit, false)
        else
            humanoid:Move(Vector3.zero, false)
        end
        humanoid.WalkSpeed = targetSpeed
    else
        -- O joystick padrão segue funcionando sem botão alternativo. Quando o
        -- jogador move para trás, o sistema escolhe a velocidade de ré menor.
        local moveDirection = humanoid.MoveDirection
        local facing = hrp.CFrame.LookVector
        local longitudinal = moveDirection:Dot(facing)
        if drive.boost then
            humanoid.WalkSpeed = CFG.DRIVE_BOOST_SPEED
        elseif longitudinal < -0.25 then
            humanoid.WalkSpeed = CFG.DRIVE_REVERSE_SPEED
        else
            humanoid.WalkSpeed = CFG.DRIVE_MAX_SPEED
        end
    end

    local velocity = hrp.AssemblyLinearVelocity
    local planarSpeed = Vector3.new(velocity.X, 0, velocity.Z).Magnitude
    drive.currentSpeed += (planarSpeed - drive.currentSpeed) * math.clamp(dt * 5.0, 0, 1)
    drive.maxObservedSpeed = math.max(drive.maxObservedSpeed, drive.currentSpeed)
    drive.lastCommand = os.clock()
end

local function updateActionRuntime(now)
    local action = Runtime.action
    if action.name == "none" then return end
    local duration = CFG.MECHA_PUNCH_TIME
    if action.name == "DASH" then duration = CFG.MECHA_DASH_TIME end
    if action.name == "STOMP" then duration = 0.72 end
    if duration <= 0 then duration = 0.01 end
    action.progress = math.clamp((now - action.startedAt) / duration, 0, 1)
    if action.progress >= 1 then
        action.name = "none"
        action.progress = 0
        for _, trail in ipairs(mechaTrails) do
            if trail and trail.Parent then
                trail.Enabled = form == FORMS.MECHA
                    and humanoid ~= nil
                    and humanoid.MoveDirection.Magnitude > 0.45
            end
        end
    end
end

local function resetActionsAndInputs()
    clearVehicleControls()
    Runtime.action.name = "none"
    Runtime.action.progress = 0
    Runtime.action.cooldownUntil = 0
    for _, entry in pairs(controlButtons) do
        entry.setReleased()
    end
end

---------------------------
-- 8.2 SUPORTE AO TECLADO E GAMEPAD (OPCIONAL)
-- O mobile continua sendo o caminho principal; os atalhos são conveniência.
---------------------------
local keyboardDrive = {
    W = Enum.KeyCode.W,
    S = Enum.KeyCode.S,
    A = Enum.KeyCode.A,
    D = Enum.KeyCode.D,
    Shift = Enum.KeyCode.LeftShift,
}

local function setKeyboardDrive(keyCode, isDown)
    if keyCode == keyboardDrive.W then
        Runtime.drive.forward = isDown
    elseif keyCode == keyboardDrive.S then
        Runtime.drive.reverse = isDown
    elseif keyCode == keyboardDrive.A then
        Runtime.drive.left = isDown
    elseif keyCode == keyboardDrive.D then
        Runtime.drive.right = isDown
    elseif keyCode == keyboardDrive.Shift then
        Runtime.drive.boost = isDown
    end
end

local function releaseInputsForNonVehicleForm()
    if form ~= FORMS.VEICULO then
        clearVehicleControls()
    end
end

---------------------------
-- 8.3 MONITOR DE INTEGRIDADE DO MODELO
-- O modelo pode ser destruído por uma experiência ou por um reset; o guardião
-- detecta isso e evita que o loop visual tente atualizar referências mortas.
---------------------------
local function isModelHealthy(model, parts)
    if not model or not model.Parent or not model.PrimaryPart or not model.PrimaryPart.Parent then
        return false
    end
    local activeParts = 0
    for _, item in ipairs(parts) do
        if item.part and item.part.Parent == model then
            activeParts += 1
        end
    end
    return activeParts > 4
end

local function repairVisualRoot(model, modelName)
    if model and model.Parent and model.PrimaryPart and model.PrimaryPart.Parent then
        return true
    end
    warn("[Transformer Engine] Raiz do modelo ausente: " .. modelName)
    return false
end

local function getActiveVisualModel()
    if form == FORMS.MECHA then return mechaModel end
    if form == FORMS.VEICULO then return vehicleModel end
    return nil
end

local function inspectRuntimeState()
    local activeModel = getActiveVisualModel()
    local report = {
        activeModelReady = activeModel == nil or activeModel.Parent ~= nil,
        characterReady = character ~= nil and character.Parent ~= nil,
        humanoidReady = humanoid ~= nil and humanoid.Parent ~= nil,
        rootReady = hrp ~= nil and hrp.Parent ~= nil,
        mechaReady = isModelHealthy(mechaModel, mechaParts),
        vehicleReady = isModelHealthy(vehicleModel, vehicleParts),
        uiReady = screenGui ~= nil and screenGui.Parent ~= nil,
        currentForm = form,
        transforming = transition ~= nil,
        destroyed = destroyed,
    }
    return report
end

---------------------------
-- 9. INICIALIZAÇÃO DO AVATAR E RESPAWN
---------------------------
local function bindCharacter(char)
    local ok = pcall(function()
        -- Um respawn invalida o rig visual anterior; recomeça no avatar original
        -- para não deixar velocidade, câmera ou transparência presas no estado antigo.
        transition = nil
        form = FORMS.HUMANOIDE
        Runtime.transform.stage = "IDLE"
        Runtime.transform.stageProgress = 0
        cameraFocusMode = nil
        resetActionsAndInputs()
        captureCharacter(char)
    end)
    if not ok then return end

    if humanoid and hrp then
        if form == FORMS.HUMANOIDE then
            setAvatarVisible(1)
        else
            setAvatarVisible(0)
            applyFormMovement(form)
        end
    end
end

if player.Character then
    task.spawn(bindCharacter, player.Character)
end
local characterAddedConnection = player.CharacterAdded:Connect(function(char)
    task.spawn(function()
        bindCharacter(char)
    end)
end)

---------------------------
-- 10. ENTRADA: TECLADO OPCIONAL E TOQUE MOBILE
---------------------------
local inputBeganConnection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed or destroyed or not CFG.SHOW_KEYBOARD_SHORTCUTS then return end
    if input.KeyCode == Enum.KeyCode.T then
        requestForm(FORMS.MECHA)
    elseif input.KeyCode == Enum.KeyCode.V then
        requestForm(FORMS.VEICULO)
    elseif input.KeyCode == Enum.KeyCode.H then
        requestForm(FORMS.HUMANOIDE)
    end
    if form == FORMS.VEICULO then
        setKeyboardDrive(input.KeyCode, true)
    end
end)

local inputEndedConnection = UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if not CFG.SHOW_KEYBOARD_SHORTCUTS then return end
    setKeyboardDrive(input.KeyCode, false)
end)

---------------------------
-- 11. LOOP VISUAL ÚNICO (OTIMIZADO)
---------------------------
local renderConnection
renderConnection = RunService.RenderStepped:Connect(function(deltaTime)
    if destroyed then return end
    camera = Workspace.CurrentCamera or camera

    if not character or not character.Parent or not humanoid or not hrp or not hrp.Parent then
        status.Text = "AGUARDANDO PERSONAGEM..."
        return
    end

    local now = os.clock()
    local moveMagnitude = humanoid.MoveDirection.Magnitude
    local moving = moveMagnitude > 0.05 and not transition
    for _, trail in ipairs(mechaTrails) do
        if trail and trail.Parent then
            trail.Enabled = (not transition) and form == FORMS.MECHA and moving and moveMagnitude > 0.45
        end
    end
    local speedScale = 1
    if form == FORMS.VEICULO then
        speedScale = 1.65
    elseif form == FORMS.MECHA then
        speedScale = 1.05
    end
    if moving then
        local gaitRate = 7.3
        if form == FORMS.VEICULO then
            gaitRate = 11
        elseif form == FORMS.MECHA and humanoid.WalkSpeed >= CFG.MECHA_SPEED * 0.72 then
            gaitRate = 10.2
        end
        walkPhase += deltaTime * gaitRate * math.clamp(moveMagnitude, 0, 1)
        wheelPhase += deltaTime * (form == FORMS.VEICULO and 7.5 or 0)
    end
    effectTick += deltaTime

    local rootCF = hrp.CFrame
    -- Os roots invisíveis funcionam como pontos de foco para a câmera Roblox.
    -- Eles acompanham o avatar, mas não participam da colisão do mundo.
    if mechaModel and mechaModel.PrimaryPart then
        mechaModel.PrimaryPart.CFrame = rootCF * CFrame.new(0, 2.05, 0)
    end
    if vehicleModel and vehicleModel.PrimaryPart then
        vehicleModel.PrimaryPart.CFrame = rootCF * CFrame.new(0, 0.45, 0)
    end

    local look = camera and camera.CFrame.LookVector or rootCF.LookVector
    local cameraLocalLook = rootCF:VectorToObjectSpace(look)

    local mVis, mAssembly = 0, 1
    local vVis, vAssembly = 0, 1
    local avatarVisible = form == FORMS.HUMANOIDE and 1 or 0
    local blurAmount = 0
    local progress = 0
    local bob = moving and math.abs(math.sin(walkPhase * 2)) * (form == FORMS.MECHA and 0.08 or 0.025) or 0

    if transition then
        progress = clamp01((now - transition.startTime) / transition.duration)
        transition.progress = progress
        Runtime.transform.stageProgress = progress
        local currentStage = getTransitionStage(progress)
        Runtime.transform.stage = currentStage
        if currentStage ~= transition.lastFxStage then
            transition.lastFxStage = currentStage
            Runtime.transform.lastStage = currentStage
            stageReadout.Text = "TRANSFORMAÇÃO  /  " .. currentStage
            -- O efeito forte ocorre durante a reconfiguração e no travamento,
            -- não em cada frame, para evitar saturar o celular com partículas.
            if currentStage == "RECONFIGURANDO EIXOS" then
                for _, emitter in ipairs(mechaEmitters) do
                    if emitter and emitter.Parent then safeCall(function() emitter:Emit(9) end) end
                end
                for _, emitter in ipairs(vehicleEmitters) do
                    if emitter and emitter.Parent then safeCall(function() emitter:Emit(7) end) end
                end
            elseif currentStage == "TRAVANDO ARTICULAÇÕES" then
                if emitAdaptiveBurst then emitAdaptiveBurst() else emitTransformationBurst() end
                Runtime.camera.shakeEnvelope = math.max(Runtime.camera.shakeEnvelope, 0.22)
            end
        end
        local eased = smoothstep(progress)
        if transition.from == FORMS.MECHA then
            mVis = 1 - eased
            mAssembly = 1 - eased
        elseif transition.to == FORMS.MECHA then
            mVis = eased
            mAssembly = eased
        end
        if transition.from == FORMS.VEICULO then
            vVis = 1 - eased
            vAssembly = 1 - eased
        elseif transition.to == FORMS.VEICULO then
            vVis = eased
            vAssembly = eased
        end

        if transition.from == FORMS.HUMANOIDE then
            avatarVisible = 1 - eased
        elseif transition.to == FORMS.HUMANOIDE then
            avatarVisible = eased
        else
            avatarVisible = 0
        end

        blurAmount = math.sin(progress * math.pi) * 7
        barFill.Size = UDim2.new(progress, 0, 1, 0)
        status.Text = "TRANSFORMANDO  •  " .. tostring(math.floor(progress * 100 + 0.5)) .. "%"
        if progress >= 1 then
            completeTransition()
            mVis = form == FORMS.MECHA and 1 or 0
            mAssembly = 1
            vVis = form == FORMS.VEICULO and 1 or 0
            vAssembly = 1
            avatarVisible = form == FORMS.HUMANOIDE and 1 or 0
            blurAmount = 0
        end
    else
        if form == FORMS.MECHA then mVis = 1 end
        if form == FORMS.VEICULO then vVis = 1 end
        if form == FORMS.HUMANOIDE then avatarVisible = 1 end
    end

    setAvatarVisible(avatarVisible)

    -- Muda o ponto seguido pela câmera em cada forma. No modo humano restaura
    -- o Humanoid original, evitando que a câmera continue presa ao modelo falso.
    if camera then
        local focusMode = form
        if transition then
            focusMode = (progress < 0.52) and transition.from or transition.to
        end
        if focusMode ~= cameraFocusMode then
            cameraFocusMode = focusMode
            if focusMode == FORMS.MECHA and mechaModel and mechaModel.PrimaryPart then
                camera.CameraSubject = mechaModel.PrimaryPart
            elseif focusMode == FORMS.VEICULO and vehicleModel and vehicleModel.PrimaryPart then
                camera.CameraSubject = vehicleModel.PrimaryPart
            elseif humanoid then
                camera.CameraSubject = humanoid
            end
        end
    end

    -- Detecta pouso e emite partículas já criadas (nada de instâncias no loop).
    local grounded = humanoid.FloorMaterial ~= Enum.Material.Air
    if form == FORMS.MECHA and grounded and not lastGrounded and now - lastFootstepAt > 0.22 then
        lastFootstepAt = now
        for _, emitter in ipairs(mechaSmokeEmitters) do
            if emitter and emitter.Parent then safeCall(function() emitter:Emit(7) end) end
        end
        Runtime.camera.shakeEnvelope = math.max(Runtime.camera.shakeEnvelope, 0.08)
    end
    lastGrounded = grounded

    local mechaRootCF = rootCF * CFrame.new(0, 0.62, 0)
    local vehicleSuspension = form == FORMS.VEICULO and moving
        and math.sin(now * 8.5) * 0.025 * math.clamp(moveMagnitude, 0, 1) or 0
    local vehicleRootCF = rootCF * CFrame.new(0, -1.45 + vehicleSuspension, 0)
    local mechaBob = form == FORMS.MECHA and bob or (transition and (transition.to == FORMS.MECHA or transition.from == FORMS.MECHA) and math.sin(progress * math.pi * 6) * 0.035 or 0)

    updateModel(mechaParts, mechaRootCF, mVis, mAssembly, FORMS.MECHA, walkPhase, moving and form == FORMS.MECHA, cameraLocalLook, now, mechaBob)
    updateModel(vehicleParts, vehicleRootCF, vVis, vAssembly, FORMS.VEICULO, wheelPhase, moving and form == FORMS.VEICULO, cameraLocalLook, now, 0)

    if mechaCoreLight and mechaCoreLight.Parent then
        local active = form == FORMS.MECHA or (transition and (transition.from == FORMS.MECHA or transition.to == FORMS.MECHA))
        local pulse = 2.0 + (math.sin(now * 3.2) + 1) * 0.5
        mechaCoreLight.Brightness = active and pulse or 0
        mechaCoreLight.Range = active and (11 + math.sin(now * 2.0) * 1.3) or 0
    end
    for _, eye in ipairs(mechaEyeParts) do
        if eye and eye.Parent then
            local active = form == FORMS.MECHA or (transition and transition.to == FORMS.MECHA)
            eye.Color = active and COLORS.BlueLight or COLORS.Steel
        end
    end
    for _, light in ipairs(vehicleLights) do
        if light and light.Parent then
            light.Brightness = (form == FORMS.VEICULO or (transition and transition.to == FORMS.VEICULO)) and (1.0 + math.sin(now * 4) * 0.18) or 0
        end
    end

    local desiredFOV = CFG.HUMANOID_FOV
    if form == FORMS.MECHA then desiredFOV = CFG.MECHA_FOV end
    if form == FORMS.VEICULO then desiredFOV = CFG.VEHICLE_FOV end
    if transition then
        local fromFOV = transition.from == FORMS.MECHA and CFG.MECHA_FOV or (transition.from == FORMS.VEICULO and CFG.VEHICLE_FOV or CFG.HUMANOID_FOV)
        local toFOV = transition.to == FORMS.MECHA and CFG.MECHA_FOV or (transition.to == FORMS.VEICULO and CFG.VEHICLE_FOV or CFG.HUMANOID_FOV)
        desiredFOV = fromFOV + (toFOV - fromFOV) * smoothstep(progress)
    end
    if camera then
        camera.FieldOfView = camera.FieldOfView + (desiredFOV - camera.FieldOfView) * math.clamp(deltaTime * 3.5, 0, 1)
    end
    if blur then
        blur.Size = blur.Size + (blurAmount - blur.Size) * math.clamp(deltaTime * 9, 0, 1)
    end

    if not transition then
        Runtime.transform.stage = "IDLE"
        if form == FORMS.HUMANOIDE then
            stageReadout.Text = "SISTEMA ONLINE  /  HUMANOIDE"
        elseif form == FORMS.MECHA then
            stageReadout.Text = "SERVO ONLINE  /  QUALIDADE " .. Runtime.quality.label
        else
            stageReadout.Text = "TRAÇÃO ONLINE  /  QUALIDADE " .. Runtime.quality.label
        end
        updateButtonStates()
    end
    updateTelemetry(deltaTime)
end)

---------------------------
-- 12. SISTEMA ADAPTATIVO DE QUALIDADE E ROTINA DE CONTROLE
-- Evita criar objetos por frame. O monitor usa média exponencial do tempo de
-- quadro e só altera taxas de partículas quando a qualidade muda de patamar.
---------------------------
local BASE_PARTICLE_RATES = {}
for _, emitter in ipairs(mechaEmitters) do
    if emitter and emitter.Parent then BASE_PARTICLE_RATES[emitter] = emitter.Rate end
end
for _, emitter in ipairs(vehicleEmitters) do
    if emitter and emitter.Parent then BASE_PARTICLE_RATES[emitter] = emitter.Rate end
end
for _, emitter in ipairs(mechaSmokeEmitters) do
    if emitter and emitter.Parent then BASE_PARTICLE_RATES[emitter] = emitter.Rate end
end

local function setQualityLevel(level)
    level = math.clamp(math.floor(level), 0, 2)
    if Runtime.quality.level == level then return end
    Runtime.quality.level = level
    if level == 0 then
        Runtime.quality.particleScale = 0.18
        Runtime.quality.label = "LEVE"
    elseif level == 1 then
        Runtime.quality.particleScale = 0.48
        Runtime.quality.label = "ECONÔMICA"
    else
        Runtime.quality.particleScale = 1
        Runtime.quality.label = "BALANCEADA"
    end
    for emitter, originalRate in pairs(BASE_PARTICLE_RATES) do
        if emitter and emitter.Parent then
            emitter.Rate = math.max(0, originalRate * Runtime.quality.particleScale)
        end
    end
end

local function sampleFrameQuality(dt)
    local quality = Runtime.quality
    quality.samples += 1
    quality.averageFrameTime += (dt - quality.averageFrameTime) * 0.07
    quality.sampleAccumulator += dt
    if quality.averageFrameTime > 1 / CFG.LOW_FPS_THRESHOLD then
        quality.lowFrameTime += dt
        quality.highFrameTime = 0
    elseif quality.averageFrameTime < 1 / CFG.HIGH_FPS_RECOVERY then
        quality.highFrameTime += dt
        quality.lowFrameTime = math.max(0, quality.lowFrameTime - dt * 0.5)
    else
        quality.lowFrameTime = math.max(0, quality.lowFrameTime - dt * 0.15)
        quality.highFrameTime = math.max(0, quality.highFrameTime - dt * 0.15)
    end
    if not CFG.AUTO_QUALITY then return end
    if quality.lowFrameTime >= 2.5 and quality.level > 0 then
        setQualityLevel(quality.level - 1)
        quality.lowFrameTime = 0
        quality.highFrameTime = 0
    elseif quality.highFrameTime >= 9 and quality.level < 2 then
        setQualityLevel(quality.level + 1)
        quality.highFrameTime = 0
        quality.lowFrameTime = 0
    end
end

local function updateCameraShake(dt)
    if destroyed then return end
    local currentCamera = Workspace.CurrentCamera
    if not currentCamera then return end
    Runtime.camera.shakeEnvelope = math.max(0, Runtime.camera.shakeEnvelope - dt * 1.65)
    local envelope = Runtime.camera.shakeEnvelope
    local planarSpeed = Runtime.telemetry.displayedSpeed
    if not transition and form == FORMS.MECHA and planarSpeed > 3 then
        envelope = math.max(envelope, CFG.CAMERA_SHAKE_MECHA * math.clamp(planarSpeed / CFG.MECHA_SPEED, 0, 1))
    elseif not transition and form == FORMS.VEICULO and planarSpeed > 10 then
        envelope = math.max(envelope, CFG.CAMERA_SHAKE_VEHICLE * math.clamp(planarSpeed / CFG.DRIVE_MAX_SPEED, 0, 1))
    end
    Runtime.camera.shakePhase += dt * (form == FORMS.VEICULO and 17 or 11)
    if envelope <= 0.0002 then return end
    local phase = Runtime.camera.shakePhase
    local x = math.sin(phase * 1.37) * envelope
    local y = math.cos(phase * 1.91) * envelope * 0.55
    local pitch = math.sin(phase * 1.17) * envelope * 0.012
    local yaw = math.cos(phase * 1.29) * envelope * 0.009
    currentCamera.CFrame = currentCamera.CFrame
        * CFrame.new(x, y, 0)
        * CFrame.Angles(pitch, yaw, 0)
end

local heartbeatConnection
heartbeatConnection = RunService.Heartbeat:Connect(function(dt)
    if destroyed then return end
    local safeDt = math.clamp(dt, 0, 0.1)
    sampleFrameQuality(safeDt)
    updateVehicleMovement(safeDt)
    updateActionRuntime(os.clock())
    releaseInputsForNonVehicleForm()
    if form ~= FORMS.VEICULO and humanoid and normalProperties and not transition then
        if form == FORMS.MECHA then
            humanoid.WalkSpeed = CFG.MECHA_SPEED
        elseif form == FORMS.HUMANOIDE then
            local originalSpeed = normalProperties.WalkSpeed or CFG.HUMANOID_SPEED
            if math.abs(humanoid.WalkSpeed - originalSpeed) > 0.01 then
                humanoid.WalkSpeed = originalSpeed
            end
        end
    end
end)

-- Camera effects são executados depois da câmera padrão. O shake é puramente
-- visual e não altera o vetor de movimento ou a posição do personagem.
RunService:BindToRenderStep(
    "TransformerEngineCameraEffects_" .. tostring(player.UserId),
    Enum.RenderPriority.Camera.Value + 1,
    updateCameraShake
)

---------------------------
-- 12.1 PAINEL DE DIAGNÓSTICO LOCAL
-- O botão DIAG pode ser usado quando algo parecer não iniciar: ele revela
-- se o avatar, as peças visuais e a interface foram construídos com sucesso.
---------------------------
local diagnosticToggle = Instance.new("TextButton")
diagnosticToggle.Name = "DiagnosticToggle"
diagnosticToggle.AnchorPoint = Vector2.new(0, 0)
diagnosticToggle.Position = UDim2.new(0, 10, 0, 10)
diagnosticToggle.Size = UDim2.new(0, 50, 0, 28)
diagnosticToggle.BackgroundColor3 = Color3.fromRGB(7, 14, 24)
diagnosticToggle.BackgroundTransparency = 0.1
diagnosticToggle.BorderSizePixel = 0
diagnosticToggle.Font = Enum.Font.GothamBlack
diagnosticToggle.Text = "DIAG"
diagnosticToggle.TextSize = 9
diagnosticToggle.TextColor3 = COLORS.BlueLight
diagnosticToggle.Parent = screenGui
local diagnosticToggleCorner = Instance.new("UICorner")
diagnosticToggleCorner.CornerRadius = UDim.new(0, 7)
diagnosticToggleCorner.Parent = diagnosticToggle
local diagnosticToggleStroke = Instance.new("UIStroke")
diagnosticToggleStroke.Color = COLORS.BlueLight
diagnosticToggleStroke.Thickness = 1
diagnosticToggleStroke.Transparency = 0.22
diagnosticToggleStroke.Parent = diagnosticToggle

local diagnosticPanel = Instance.new("Frame")
diagnosticPanel.Name = "DiagnosticPanel"
diagnosticPanel.AnchorPoint = Vector2.new(0, 0)
diagnosticPanel.Position = UDim2.new(0, 10, 0, 44)
diagnosticPanel.Size = UDim2.new(0, 226, 0, 192)
diagnosticPanel.BackgroundColor3 = Color3.fromRGB(4, 9, 16)
diagnosticPanel.BackgroundTransparency = 0.08
diagnosticPanel.BorderSizePixel = 0
diagnosticPanel.Visible = false
diagnosticPanel.Parent = screenGui
local diagnosticPanelCorner = Instance.new("UICorner")
diagnosticPanelCorner.CornerRadius = UDim.new(0, 8)
diagnosticPanelCorner.Parent = diagnosticPanel
local diagnosticPanelStroke = Instance.new("UIStroke")
diagnosticPanelStroke.Color = COLORS.Steel
diagnosticPanelStroke.Thickness = 1
diagnosticPanelStroke.Transparency = 0.15
diagnosticPanelStroke.Parent = diagnosticPanel

local diagnosticTitle = Instance.new("TextLabel")
diagnosticTitle.Name = "DiagnosticTitle"
diagnosticTitle.BackgroundTransparency = 1
diagnosticTitle.Position = UDim2.new(0, 9, 0, 6)
diagnosticTitle.Size = UDim2.new(1, -18, 0, 18)
diagnosticTitle.Font = Enum.Font.GothamBlack
diagnosticTitle.Text = "TRANSFORMER / DIAGNÓSTICO"
diagnosticTitle.TextColor3 = COLORS.White
diagnosticTitle.TextSize = 10
diagnosticTitle.TextXAlignment = Enum.TextXAlignment.Left
diagnosticTitle.Parent = diagnosticPanel

local diagnosticText = Instance.new("TextLabel")
diagnosticText.Name = "DiagnosticText"
diagnosticText.BackgroundTransparency = 1
diagnosticText.Position = UDim2.new(0, 9, 0, 27)
diagnosticText.Size = UDim2.new(1, -18, 1, -34)
diagnosticText.Font = Enum.Font.Code
diagnosticText.Text = "INICIALIZANDO..."
diagnosticText.TextColor3 = Color3.fromRGB(169, 194, 214)
diagnosticText.TextSize = 9
diagnosticText.TextWrapped = false
diagnosticText.TextXAlignment = Enum.TextXAlignment.Left
diagnosticText.TextYAlignment = Enum.TextYAlignment.Top
diagnosticText.Parent = diagnosticPanel

local diagnosticUpdateAccumulator = 0
local getDebugSummary
local function collectDiagnosticLines()
    local state = inspectRuntimeState()
    local fps = Runtime.quality.averageFrameTime > 0
        and math.floor(1 / Runtime.quality.averageFrameTime + 0.5) or 0
    local mechaCount = #mechaParts
    local vehicleCount = #vehicleParts
    local rootPosition = hrp and hrp.Parent and hrp.Position or Vector3.zero
    local positionLine = string.format("XYZ %d / %d / %d",
        math.floor(rootPosition.X), math.floor(rootPosition.Y), math.floor(rootPosition.Z))
    local lines = {
        "ESTADO: " .. tostring(state.currentForm),
        "AVATAR: " .. (state.characterReady and "OK" or "AGUARDANDO"),
        "HUMANOID: " .. (state.humanoidReady and "OK" or "AUSENTE"),
        "ROOT: " .. (state.rootReady and "OK" or "AUSENTE"),
        "MECHA: " .. (state.mechaReady and "OK" or "ERRO") .. " / " .. mechaCount .. " peças",
        "VEÍCULO: " .. (state.vehicleReady and "OK" or "ERRO") .. " / " .. vehicleCount .. " peças",
        "DETALHES: " .. tostring(generatedDetailCount) .. "/" .. tostring(CFG.DETAIL_PART_BUDGET_MOBILE),
        "UI: " .. (state.uiReady and "OK" or "ERRO"),
        "MODELO ATIVO: " .. (state.activeModelReady and "OK" or "ERRO"),
        "TRANSIÇÃO: " .. (state.transforming and Runtime.transform.stage or "NÃO"),
        "FPS MÉDIO: " .. tostring(fps) .. " / QUALIDADE " .. Runtime.quality.label,
        "AÇÃO: " .. tostring(Runtime.action.name),
        "BOOST: " .. (Runtime.drive.boost and "ATIVO" or "INATIVO"),
        positionLine,
        getDebugSummary(),
    }
    return table.concat(lines, "\n")
end

diagnosticToggle.Activated:Connect(function()
    diagnosticPanel.Visible = not diagnosticPanel.Visible
    diagnosticToggle.TextColor3 = diagnosticPanel.Visible and COLORS.Gold or COLORS.BlueLight
end)

local function updateDiagnosticPanel(dt)
    if not diagnosticPanel.Visible then return end
    diagnosticUpdateAccumulator += dt
    if diagnosticUpdateAccumulator < 0.3 then return end
    diagnosticUpdateAccumulator = 0
    diagnosticText.Text = collectDiagnosticLines()
end

---------------------------
-- 12.2 TABELAS DE ANIMAÇÃO, LIMITES E VALIDAÇÃO
-- Os perfis permitem alterar valores sem espalhar números mágicos em handlers.
---------------------------
ANIMATION_PROFILES = {
    MechaIdle = {
        frequency = 1.15,
        amplitude = 0.025,
        chestPulse = 0.012,
        headLimitYaw = 0.52,
        headLimitPitch = 0.18,
    },
    MechaWalk = {
        cycle = 7.3,
        armAmplitude = 0.52,
        legAmplitude = 0.46,
        torsoBob = 0.08,
        footstepInterval = 0.22,
    },
    MechaRun = {
        cycle = 10.2,
        armAmplitude = 0.72,
        legAmplitude = 0.61,
        torsoBob = 0.12,
        footstepInterval = 0.16,
    },
    VehicleIdle = {
        suspensionFrequency = 1.7,
        suspensionAmplitude = 0.003,
        steeringRoll = 0.025,
    },
    VehicleMove = {
        wheelAngularVelocity = 7.5,
        suspensionFrequency = 8.5,
        suspensionAmplitude = 0.025,
        steeringRoll = 0.06,
    },
    Transform = {
        phaseAnticipation = CFG.TRANSFORM_STAGE_ANTICIPATION,
        phaseUnfold = CFG.TRANSFORM_STAGE_UNFOLD,
        phaseRotate = CFG.TRANSFORM_STAGE_ROTATE,
        phaseLock = CFG.TRANSFORM_STAGE_LOCK,
    },
}

local function validateAnimationProfiles()
    local valid = true
    for profileName, profile in pairs(ANIMATION_PROFILES) do
        if type(profile) ~= "table" then
            valid = false
            warn("[Transformer Engine] Perfil de animação inválido: " .. tostring(profileName))
        else
            for key, value in pairs(profile) do
                if type(value) == "number" and (value ~= value or value == math.huge or value == -math.huge) then
                    valid = false
                    warn("[Transformer Engine] Número inválido no perfil: " .. tostring(profileName) .. "/" .. tostring(key))
                end
            end
        end
    end
    return valid
end

local function validateConfiguration()
    local valid = true
    local numericKeys = {
        "TRANSFORM_TIME", "HUMANOID_SPEED", "MECHA_SPEED", "VEHICLE_SPEED",
        "MECHA_JUMP_POWER", "MECHA_JUMP_HEIGHT", "WALK_SWAY", "RUN_SWAY",
        "MECHA_FOV", "VEHICLE_FOV", "HUMANOID_FOV", "FX_PARTICLE_RATE",
        "FX_LOW_PARTICLE_RATE", "DRIVE_MAX_SPEED", "DRIVE_BOOST_SPEED",
        "DRIVE_REVERSE_SPEED", "DRIVE_TURN_RATE", "MECHA_DASH_SPEED",
    }
    for _, key in ipairs(numericKeys) do
        local value = CFG[key]
        if type(value) ~= "number" or value ~= value or value < 0 then
            valid = false
            warn("[Transformer Engine] Configuração inválida: " .. tostring(key))
        end
    end
    if CFG.TRANSFORM_TIME < 1.0 then
        warn("[Transformer Engine] TRANSFORM_TIME muito curto; pode parecer instantâneo.")
    end
    if CFG.DRIVE_BOOST_SPEED < CFG.DRIVE_MAX_SPEED then
        warn("[Transformer Engine] BOOST menor que velocidade máxima padrão.")
    end
    return valid
end

local function runBootDiagnostics()
    local configOkay = validateConfiguration()
    local profilesOkay = validateAnimationProfiles()
    local robotOkay = isModelHealthy(mechaModel, mechaParts)
    local truckOkay = isModelHealthy(vehicleModel, vehicleParts)
    if not configOkay then
        warn("[Transformer Engine] Uma ou mais configurações falharam na validação.")
    end
    if not profilesOkay then
        warn("[Transformer Engine] Um ou mais perfis de animação falharam na validação.")
    end
    if not robotOkay then
        warn("[Transformer Engine] A montagem do mecha está incompleta.")
    end
    if not truckOkay then
        warn("[Transformer Engine] A montagem do veículo está incompleta.")
    end
    return configOkay and profilesOkay and robotOkay and truckOkay
end

---------------------------
-- 12.3 WATCHDOG DE RESPAWN E ESTADO
---------------------------
local watchdogAccumulator = 0
local function watchdogStep(dt)
    watchdogAccumulator += dt
    if watchdogAccumulator < 1.25 then return end
    watchdogAccumulator = 0
    if destroyed then return end
    if not character or not character.Parent then
        status.Text = "AGUARDANDO AVATAR..."
        status.TextColor3 = COLORS.Gold
        return
    end
    if not humanoid or not humanoid.Parent or not hrp or not hrp.Parent then
        status.Text = "AVATAR INCOMPLETO / AGUARDANDO RESPAWN"
        status.TextColor3 = COLORS.RedLight
        clearVehicleControls()
        return
    end
    if not repairVisualRoot(mechaModel, "MechaRig") then
        status.Text = "ERRO NO ROOT DO MECHA"
        status.TextColor3 = COLORS.RedLight
    elseif not repairVisualRoot(vehicleModel, "VehicleRig") then
        status.Text = "ERRO NO ROOT DO VEÍCULO"
        status.TextColor3 = COLORS.RedLight
    end
end

-- Um atualizador central pequeno compartilha os ticks entre diagnóstico e
-- watchdog para não criar conexões separadas para cada rótulo/elemento da UI.
local serviceAccumulator = 0
local serviceConnection
serviceConnection = RunService.Heartbeat:Connect(function(dt)
    if destroyed then return end
    serviceAccumulator += dt
    if serviceAccumulator >= 0.10 then
        local elapsed = serviceAccumulator
        serviceAccumulator = 0
        updateDiagnosticPanel(elapsed)
        watchdogStep(elapsed)
    end
end)

---------------------------
-- 12.4 GUIA INTERNO DE ESTADOS E CONTRATOS
-- Estes predicados são usados na inicialização, diagnóstico e futuros ajustes.
---------------------------
local function isKnownForm(value)
    return value == FORMS.HUMANOIDE
        or value == FORMS.MECHA
        or value == FORMS.VEICULO
end

canRequestTransformation = function(target)
    if destroyed then return false, "SCRIPT_REMOVIDO" end
    if not isKnownForm(target) then return false, "FORMA_DESCONHECIDA" end
    if transition then return false, "TRANSFORMAÇÃO_EM_CURSO" end
    if target == form then return false, "FORMA_JÁ_ATIVA" end
    if not humanoid or not hrp or humanoid.Health <= 0 then
        return false, "AVATAR_NÃO_PRONTO"
    end
    return true, "OK"
end

local function getFormDisplayName(value)
    if value == FORMS.HUMANOIDE then return "HUMANOIDE" end
    if value == FORMS.MECHA then return "MECHA" end
    if value == FORMS.VEICULO then return "VEÍCULO" end
    return "DESCONHECIDO"
end

local function getTransitionSummary()
    if not transition then
        return "SEM TRANSIÇÃO"
    end
    local percent = math.floor((transition.progress or 0) * 100 + 0.5)
    return getFormDisplayName(transition.from) .. " -> "
        .. getFormDisplayName(transition.to) .. " / " .. tostring(percent) .. "%"
end

local function getQualitySummary()
    return Runtime.quality.label .. " / média "
        .. tostring(math.floor(1 / math.max(Runtime.quality.averageFrameTime, 0.001))) .. " FPS"
end

local function getVehicleSummary()
    local drive = Runtime.drive
    if form ~= FORMS.VEICULO then return "VEÍCULO INATIVO" end
    if drive.boost then return "BOOST / " .. formatSpeed(drive.currentSpeed) end
    if drive.braking then return "FREANDO / " .. formatSpeed(drive.currentSpeed) end
    if drive.reverse then return "RÉ / " .. formatSpeed(drive.currentSpeed) end
    if drive.throttle > 0 then return "ACELERANDO / " .. formatSpeed(drive.currentSpeed) end
    return "TRAÇÃO PRONTA / " .. formatSpeed(drive.currentSpeed)
end

local function recordLastKnownRoot()
    if hrp and hrp.Parent then
        Runtime.telemetry.lastRootPosition = hrp.Position
        return hrp.Position
    end
    return Runtime.telemetry.lastRootPosition
end

local function getActivePartCount()
    if form == FORMS.MECHA then return #mechaParts end
    if form == FORMS.VEICULO then return #vehicleParts end
    return 0
end

local function getActionSummary()
    local action = Runtime.action
    if action.name == "none" then return "PRONTO" end
    return action.name .. " / " .. tostring(math.floor(action.progress * 100 + 0.5)) .. "%"
end

getDebugSummary = function()
    return table.concat({
        getTransitionSummary(),
        getQualitySummary(),
        getVehicleSummary(),
        "AÇÃO " .. getActionSummary(),
        "PEÇAS VISÍVEIS " .. tostring(getActivePartCount()),
    }, "\n")
end

---------------------------
-- 12.5 PASSO DE SERVIÇO UNIFICADO
---------------------------
local serviceRevision = 3
local function runServiceStep(dt)
    if destroyed then return end
    local now = os.clock()
    Runtime.transform.pulse = (math.sin(now * 3.0) + 1) * 0.5
    if transition then
        Runtime.transform.stageProgress = transition.progress or 0
    else
        Runtime.transform.stageProgress = 0
    end
    if form ~= FORMS.MECHA then
        for _, trail in ipairs(mechaTrails) do
            if trail and trail.Parent and not transition then trail.Enabled = false end
        end
    end
    if form ~= FORMS.VEICULO and not transition then
        Runtime.drive.currentSpeed *= math.max(0, 1 - dt * 2.4)
    end
    recordLastKnownRoot()
end

local finalServiceConnection
finalServiceConnection = RunService.Heartbeat:Connect(runServiceStep)

---------------------------
-- 12.6 AUTO-TESTES SEM ALTERAR A PARTIDA
-- Não teletransportam o jogador e não acionam transformações; só verificam
-- contratos simples, então podem rodar na inicialização sem quebrar a sessão.
---------------------------
local function runNonDestructiveSelfTests()
    local results = {}
    local function test(name, callback)
        local ok, value = pcall(callback)
        table.insert(results, {
            name = name,
            passed = ok and value == true,
            detail = ok and (value == true and "OK" or tostring(value)) or tostring(value),
        })
    end

    test("enum de formas", function()
        return isKnownForm(FORMS.HUMANOIDE)
            and isKnownForm(FORMS.MECHA)
            and isKnownForm(FORMS.VEICULO)
    end)
    test("tempo de transformação", function()
        return CFG.TRANSFORM_TIME >= 1
    end)
    test("corpo do mecha", function()
        return #mechaParts >= 50
    end)
    test("carroceria do veículo", function()
        return #vehicleParts >= 40
    end)
    test("UI presente", function()
        return screenGui ~= nil and screenGui.Parent ~= nil
    end)
    test("qualidade dentro dos limites", function()
        return Runtime.quality.level >= 0 and Runtime.quality.level <= 2
    end)
    test("controles de veículo registrados", function()
        return controlButtons.Throttle ~= nil
            and controlButtons.Brake ~= nil
            and controlButtons.SteerLeft ~= nil
            and controlButtons.SteerRight ~= nil
    end)
    test("handlers do mecha registrados", function()
        return Runtime.actionHandlers.dash ~= nil
            and Runtime.actionHandlers.punch ~= nil
            and Runtime.actionHandlers.stomp ~= nil
    end)

    local failed = 0
    for _, result in ipairs(results) do
        if not result.passed then
            failed += 1
            warn("[Transformer Engine][SELFTEST] " .. result.name .. ": " .. result.detail)
        end
    end
    print("[Transformer Engine] Auto-test: " .. tostring(#results - failed)
        .. "/" .. tostring(#results) .. " verificações passaram.")
    return failed == 0, results
end

---------------------------
-- 12.7 LIMITADOR DE PRESSÃO DE EFEITOS
---------------------------
local function throttleParticleBursts()
    local scale = Runtime.quality.particleScale
    local mechaAmount = math.floor(7 * scale + 0.5)
    local vehicleAmount = math.floor(5 * scale + 0.5)
    return math.max(1, mechaAmount), math.max(1, vehicleAmount)
end

emitAdaptiveBurst = function()
    local mechaAmount, vehicleAmount = throttleParticleBursts()
    for _, emitter in ipairs(mechaEmitters) do
        if emitter and emitter.Parent then
            safeCall(function() emitter:Emit(mechaAmount) end)
        end
    end
    for _, emitter in ipairs(vehicleEmitters) do
        if emitter and emitter.Parent then
            safeCall(function() emitter:Emit(vehicleAmount) end)
        end
    end
end

---------------------------
-- 12.8 REGISTRO DE EVENTOS E ESTADO DE LIMPEZA
---------------------------
local function registerConnection(connection)
    if connection then
        table.insert(Runtime.cleanup.connections, connection)
    end
    return connection
end

local function registerInstance(instance)
    if instance then
        table.insert(Runtime.cleanup.instances, instance)
    end
    return instance
end

local function disconnectRegisteredConnections()
    for _, connection in ipairs(Runtime.cleanup.connections) do
        if connection and connection.Connected then
            pcall(function() connection:Disconnect() end)
        end
    end
    table.clear(Runtime.cleanup.connections)
end

local function destroyRegisteredInstances()
    for _, instance in ipairs(Runtime.cleanup.instances) do
        if instance and instance.Parent then
            pcall(function() instance:Destroy() end)
        end
    end
    table.clear(Runtime.cleanup.instances)
end

---------------------------
-- 12.9 PREDICADOS DE SEGURANÇA DE MOVIMENTO
---------------------------
local function isHumanoidAlive()
    return humanoid ~= nil and humanoid.Parent ~= nil and humanoid.Health > 0
end

local function isRootAvailable()
    return hrp ~= nil and hrp.Parent ~= nil and hrp:IsA("BasePart")
end

canApplyVelocity = function()
    return isHumanoidAlive() and isRootAvailable() and not transition
end

stopLocalMovement = function()
    if humanoid and humanoid.Parent then
        pcall(function()
            humanoid:Move(Vector3.zero, false)
        end)
    end
    if hrp and hrp.Parent then
        pcall(function()
            hrp.AssemblyAngularVelocity = Vector3.zero
        end)
    end
end

local function enforceFormSpeed()
    if not humanoid or not humanoid.Parent or transition then return end
    if form == FORMS.MECHA then
        humanoid.WalkSpeed = CFG.MECHA_SPEED
        humanoid.JumpPower = CFG.MECHA_JUMP_POWER
    elseif form == FORMS.VEICULO then
        if not Runtime.drive.braking then
            humanoid.WalkSpeed = Runtime.drive.boost and CFG.DRIVE_BOOST_SPEED or CFG.DRIVE_MAX_SPEED
        end
        humanoid.JumpPower = 0
    end
end

---------------------------
-- 12.10 STATUS DE INICIALIZAÇÃO
---------------------------
local function finishInitialization()
    Runtime.cleanup.registered = true
    registerInstance(mechaModel)
    registerInstance(vehicleModel)
    registerInstance(screenGui)
    registerInstance(blur)
    registerConnection(heartbeatConnection)
    registerConnection(serviceConnection)
    registerConnection(finalServiceConnection)
    setQualityLevel(2)
    updateControlsVisibility()
    runBootDiagnostics()
    runNonDestructiveSelfTests()
    if isHumanoidAlive() and isRootAvailable() then
        enforceFormSpeed()
    end
    local state = inspectRuntimeState()
    if state.characterReady and state.humanoidReady and state.rootReady then
        stageReadout.Text = "SISTEMA ONLINE  /  " .. getFormDisplayName(form)
    else
        stageReadout.Text = "AGUARDANDO AVATAR  /  SISTEMA PRONTO"
    end
    print("[Transformer Engine] V3 iniciada; serviço " .. tostring(serviceRevision)
        .. "; peças mecha=" .. tostring(#mechaParts)
        .. "; peças veículo=" .. tostring(#vehicleParts) .. ".")
end

---------------------------
-- 12. ELEGÂNCIA: LIMPEZA QUANDO O SCRIPT É REMOVIDO
---------------------------
local function cleanup()
    if destroyed then return end
    destroyed = true
    if renderConnection then renderConnection:Disconnect() end
    if heartbeatConnection then heartbeatConnection:Disconnect() end
    if serviceConnection then serviceConnection:Disconnect() end
    if finalServiceConnection then finalServiceConnection:Disconnect() end
    if descendantConnection then descendantConnection:Disconnect() end
    if characterAddedConnection then characterAddedConnection:Disconnect() end
    if inputBeganConnection then inputBeganConnection:Disconnect() end
    if inputEndedConnection then inputEndedConnection:Disconnect() end
    pcall(function()
        RunService:UnbindFromRenderStep("TransformerEngineCameraEffects_" .. tostring(player.UserId))
    end)
    resetActionsAndInputs()
    disconnectRegisteredConnections()
    destroyRegisteredInstances()
    if humanoid and normalProperties then
        pcall(function()
            humanoid.WalkSpeed = normalProperties.WalkSpeed or CFG.HUMANOID_SPEED
            humanoid.UseJumpPower = normalProperties.UseJumpPower
            humanoid.JumpPower = normalProperties.JumpPower or 50
            humanoid.JumpHeight = normalProperties.JumpHeight or 7.2
            humanoid.AutoRotate = normalProperties.AutoRotate ~= false
            humanoid.CameraOffset = normalProperties.CameraOffset or Vector3.zero
        end)
    end
    for part, original in pairs(avatarParts) do
        if part and part.Parent then
            pcall(function() part.LocalTransparencyModifier = original end)
        end
    end
    if camera then
        pcall(function()
            camera.CameraSubject = savedCameraSubject or humanoid
            camera.CameraType = savedCameraType or Enum.CameraType.Custom
            camera.FieldOfView = CFG.HUMANOID_FOV
        end)
    end
    pcall(function()
        player.CameraMinZoomDistance = savedCameraMinZoom
        player.CameraMaxZoomDistance = savedCameraMaxZoom
    end)
    if mechaModel and mechaModel.Parent then mechaModel:Destroy() end
    if vehicleModel and vehicleModel.Parent then vehicleModel:Destroy() end
    if screenGui and screenGui.Parent then screenGui:Destroy() end
    if blur and blur.Parent then blur:Destroy() end
    requestForm = nil
end

script.AncestryChanged:Connect(function(_, parent)
    if parent == nil then cleanup() end
end)

updateButtonStates()
finishInitialization()
print("[Transformer Engine] Autônomo iniciado. Toque em ROBÔ, VEÍCULO ou HUMANO.")

