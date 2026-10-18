--[[
THE POOLROOMS - geração infinita (client-side)
-- Primeira pessoa, sem monstros
-- Piscinas rasas (andáveis), salas, corredores, colunas, arcos, bancos, escadas, etc.
-- SFX: reverb, ambiente, passos, respingos, pingos distantes, lâmpadas piscando
-- VFX: névoa, bloom, partículas de poeira, ondas na água, respingos, brilho da água, vinheta
Controles: WASD andar | SHIFT correr | P sair (volta pro mapa principal) | PULO DESATIVADO
]]

local env = (getgenv and getgenv()) or _G
if env.__POOLROOMS_STOP then pcall(env.__POOLROOMS_STOP) end

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")
local TweenService = game:GetService("TweenService")
local SoundService = game:GetService("SoundService")
local UserInputService = game:GetService("UserInputService")
local StarterGui = game:GetService("StarterGui")
local ContextActionService = game:GetService("ContextActionService")
local Debris = game:GetService("Debris")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera


---

-- CONFIG

local CFG = {
Radius = 3,            -- chunks ao redor (diminua p/ celular: 2)
PerFrame = 1,          -- chunks gerados por frame
Base = Vector3.new(20000, 5000, 20000), -- onde as Poolrooms ficam (longe do mapa)
Seed = math.random(1, 999999),
WalkSpeed = 14,
SprintSpeed = 22,
ExitKey = Enum.KeyCode.P,
Exposure = -1.05,                         -- bem mais escuro, luzes do teto continuam dominantes
Ambient = Color3.fromRGB(16, 22, 25),     -- ambiente quase preto; o teto é a fonte de luz
CameraBob = 0.16,     -- balanço natural da câmera
WaterSounds = false,  -- som de splash da água (o do Roblox é alto/chato)
Scares = true,         -- sustos visuais; o rig não causa dano
ScareChance = 0.12,    -- aparições bem mais raras
ScareTriggerChance = 0.006,
}

-- Sons (embutidos do Roblox). Troque por rbxassetid://ID se quiser outros.
local S = {
step = "rbxasset://sounds/action_footsteps_plastic.mp3",
splash = "rbxasset://sounds/impact_water.mp3",
swim = "rbxasset://sounds/action_swim.mp3",
hum = "rbxasset://sounds/bass.wav",
click = "rbxasset://sounds/snap.mp3",
}

local CS, H, CELL = 48, 18, 8
local BASE = CFG.Base
local BY = BASE.Y


---

-- Materiais / cores

local okTile, tileMat = pcall(function() return Enum.Material.CeramicTiles end)
local TILE = okTile and tileMat or Enum.Material.Marble

local C_TILE = Color3.fromRGB(200, 214, 212)
local C_DARK = Color3.fromRGB(170, 192, 192)
local C_TEAL = Color3.fromRGB(110, 165, 172)
local C_STEP = Color3.fromRGB(184, 205, 204)
local C_WATER1 = Color3.fromRGB(45, 150, 165)
local C_WATER2 = Color3.fromRGB(70, 180, 190)


---

-- Estado

local running = true
local conns = {}
local chunks = {}
local queue = {}
local saved = { lighting = {}, removed = {}, disabled = {} }
local obfAlphabet = "abcdefghijklmnopqrstuvwxyz0123456789"
local function obfName()
local length = math.random(16, 24)
local chars = table.create(length)
for i = 1, length do
local index = math.random(1, #obfAlphabet)
chars[i] = obfAlphabet:sub(index, index)
end
return table.concat(chars)
end

local function inst(className)
local obj = Instance.new(className)
obj.Name = obfName()
return obj
end

local root = inst("Folder")
root.Name = obfName()
root.Parent = Workspace

local sfxFolder = inst("Folder")
sfxFolder.Name = obfName()
sfxFolder.Parent = SoundService

local gui, effects = nil, {}
local inventoryGui, lanternTool, lanternLights = nil, nil, {}
local savedBackpackEnabled = true
local flashlightOn = true
local savedCF, savedWalk
local savedHum = {}
local savedCam = {}
local camState = { moving = false, sprint = false }
local sprinting, runOn, shiftHeld = false, false, false
local updateRunBtn = function() end
local function refreshRun()
sprinting = runOn or shiftHeld
updateRunBtn()
end
local lastVy, landDip, camAmp, camRoll = 0, 0, 0, 0
local bobT = 0
local blurFx
local scareRegistry = {}
local nextStepSfx = 0

-- Anticheat local: camada de reação no cliente (reverte, pune, kicka). Não substitui validação no servidor.
local AC = {
Enabled = true,
KickOnDetection = true,   -- kicka ao acumular MaxStrikes detecções dentro de StrikeWindow
MaxStrikes = 4,
StrikeWindow = 15,
FlagCooldown = 1,         -- cooldown por tipo de detecção (segundos)
SpeedFactor = 1.15,       -- tolerância sobre a velocidade esperada
SpeedSlack = 1.5,
SpeedWindow = 0.6,        -- janela p/ média de velocidade (pega tpwalk em passos pequenos)
TeleportDistance = 20,
AirGrace = 0.45,          -- tempo no ar sem pulo antes de checar voo
MinGap = 1.1,             -- distância dos pés ao chão p/ contar como "no ar"
MaxFeetHeight = 4.5,      -- altura máxima legítima (prancha de mergulho ~3)
MaxUpVel = 14,            -- sem pulo, velocidade vertical pra cima não existe
LagSpikeThreshold = 0.35,
LagRecovery = 1.0,
}
table.freeze(AC)

local HST = Enum.HumanoidStateType
local BAD_STATES = {
[HST.Jumping] = true, [HST.Flying] = true, [HST.Climbing] = true, [HST.Swimming] = true,
[HST.Physics] = true, [HST.PlatformStanding] = true, [HST.Seated] = true, [HST.Ragdoll] = true,
}
local LOCKED_STATES = { HST.Jumping, HST.Flying, HST.Climbing, HST.Swimming }
local WATCHED_PROPS = { "WalkSpeed", "JumpPower", "JumpHeight", "UseJumpPower", "HipHeight",
"MaxSlopeAngle", "PlatformStand", "AutoRotate", "Sit" }
local MOVER_CLASSES = { "BodyMover", "RocketPropulsion", "LinearVelocity", "AngularVelocity", "AlignPosition",
"AlignOrientation", "VectorForce", "LineForce", "Torque" }

local acState = {
lastPos = nil, lastTime = 0, strikes = {}, lastFlags = {}, lastReason = "", lagUntil = 0,
safeCF = nil, samples = {}, airT = 0, noclipT = 0, upT = 0, idleT = 0, fastT = 0, jumpF = 0,
gravity = Workspace.Gravity,
expected = { WalkSpeed = 16, JumpPower = 0, JumpHeight = 0, UseJumpPower = true, HipHeight = 2,
MaxSlopeAngle = 89, PlatformStand = false, AutoRotate = true, Sit = false },
}
local acGraceUntil = 0

-- só enxerga a geometria das Poolrooms que realmente colide
local acRay = RaycastParams.new()
acRay.FilterType = Enum.RaycastFilterType.Include
acRay.FilterDescendantsInstances = { root }
acRay.RespectCanCollide = true
local acOverlap = OverlapParams.new()
acOverlap.FilterType = Enum.RaycastFilterType.Include
acOverlap.FilterDescendantsInstances = { root }
acOverlap.RespectCanCollide = true


---

-- Helpers

local function hash(a, b, c)
return bit32.bxor(
(a * 73856093) % 4294967296,
(b * 19349663) % 4294967296,
((c or 0) * 83492791 + CFG.Seed * 2654435761) % 4294967296
)
end

local function mk(parent, size, cf, color, mat, extra)
local p = inst("Part")
p.Anchored = true
p.Size = size
p.CFrame = cf
p.Color = color
p.Material = mat or TILE
p.TopSurface = Enum.SurfaceType.Smooth
p.BottomSurface = Enum.SurfaceType.Smooth
if extra then
for k, v in pairs(extra) do p[k] = v end
end
p.Parent = parent
return p
end

local GHOST = { CanCollide = false, CanQuery = false, CanTouch = false }
local GHOST_T = { Transparency = 1, CanCollide = false, CanQuery = false, CanTouch = false }

local function playUI(id, vol, pitch)
local s = inst("Sound")
s.SoundId = id
s.Volume = vol
s.PlaybackSpeed = pitch or 1
s.Parent = sfxFolder
s:Play()
Debris:AddItem(s, 5)
end

local function play3D(id, pos, vol, pitch, maxd)
local p = mk(root, Vector3.one * 0.2, CFrame.new(pos), C_TILE, Enum.Material.SmoothPlastic,
{ Transparency = 1, CanCollide = false, CanQuery = false, CanTouch = false })
local s = inst("Sound")
s.SoundId = id
s.Volume = vol
s.PlaybackSpeed = pitch or 1
s.RollOffMode = Enum.RollOffMode.InverseTapered
s.RollOffMinDistance = 6
s.RollOffMaxDistance = maxd or 90
s.Parent = p
s:Play()
Debris:AddItem(p, 5)
end

local function ripple(pos, size, life)
local p = mk(root, Vector3.new(0.05, 0.5, 0.5),
CFrame.new(pos) * CFrame.Angles(0, 0, math.pi / 2),
Color3.fromRGB(235, 255, 255), Enum.Material.SmoothPlastic,
{ Transparency = 0.45, CanCollide = false, CanQuery = false, CanTouch = false, CastShadow = false })
p.Shape = Enum.PartType.Cylinder
p.Size = Vector3.new(0.05, 0.5, 0.5)
TweenService:Create(p, TweenInfo.new(life or 1.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
{ Size = Vector3.new(0.05, size, size), Transparency = 1 }):Play()
Debris:AddItem(p, (life or 1.2) + 0.2)
end

-- Rig Blocky de susto: usa Humanoid:MoveTo(), sem TweenService e sem dano.
local function createScare(parent, pos, submerged, rng)
local model = Instance.new("Model")
model.Name = obfName()
model.Parent = parent
local sc = { parent = model, model = model, pos = pos, home = pos, active = false,
visible = false, mode = rng:NextNumber() < 0.52 and "static" or "runner",
cooldown = 0, moveClock = 0, submerged = submerged }
local dark = Color3.fromRGB(8, 16, 19)
local function part(name, size, at, transparency)
local q = mk(model, size, CFrame.new(at), dark, Enum.Material.SmoothPlastic,
{ Transparency = transparency or 1, CanCollide = false, CanQuery = false, CanTouch = false, CastShadow = false, Massless = true })
q.Name = name
return q
end
local hrp = part("HumanoidRootPart", Vector3.new(2, 2, 1), pos + Vector3.new(0, 2, 0), 1)
local torso = part("Torso", Vector3.new(2, 2, 1), pos + Vector3.new(0, 3.5, 0), 0.08)
local head = part("Head", Vector3.new(2, 1, 1), pos + Vector3.new(0, 5.15, 0), 0.02)
local leftArm = part("Left Arm", Vector3.new(1, 2, 1), pos + Vector3.new(-1.5, 3.5, 0), 0.08)
local rightArm = part("Right Arm", Vector3.new(1, 2, 1), pos + Vector3.new(1.5, 3.5, 0), 0.08)
local leftLeg = part("Left Leg", Vector3.new(1, 2, 1), pos + Vector3.new(-0.5, 1, 0), 0.08)
local rightLeg = part("Right Leg", Vector3.new(1, 2, 1), pos + Vector3.new(0.5, 1, 0), 0.08)
local hum = Instance.new("Humanoid")
hum.Name = "ScareHumanoid"
hum.WalkSpeed = 11
hum.JumpPower = 0
hum.JumpHeight = 0
hum.AutoRotate = true
hum.DisplayDistanceType = Enum.HumanoidDisplayDistanceType.None
hum.HealthDisplayType = Enum.HumanoidHealthDisplayType.AlwaysOff
hum.NameDisplayDistance = 0
hum.BreakJointsOnDeath = false
hum.RequiresNeck = false
hum.Parent = model
for _, state in ipairs({ Enum.HumanoidStateType.Jumping, Enum.HumanoidStateType.FallingDown, Enum.HumanoidStateType.Ragdoll }) do
pcall(function() hum:SetStateEnabled(state, false) end)
end
local function joint(name, p0, p1, c0, c1)
local m = Instance.new("Motor6D")
m.Name = name
m.Part0, m.Part1, m.C0, m.C1 = p0, p1, c0, c1
m.Parent = p0
end
joint("RootJoint", hrp, torso, CFrame.new(0, 1.5, 0), CFrame.new())
joint("Neck", torso, head, CFrame.new(0, 1.5, 0), CFrame.new(0, -0.5, 0))
joint("Left Shoulder", torso, leftArm, CFrame.new(-1.5, 0.5, 0), CFrame.new(0, 0.5, 0))
joint("Right Shoulder", torso, rightArm, CFrame.new(1.5, 0.5, 0), CFrame.new(0, 0.5, 0))
joint("Left Hip", torso, leftLeg, CFrame.new(-0.5, -1, 0), CFrame.new(0, 1, 0))
joint("Right Hip", torso, rightLeg, CFrame.new(0.5, -1, 0), CFrame.new(0, 1, 0))
model.PrimaryPart = hrp
if submerged then sc.pos = pos end
table.insert(scareRegistry, sc)
return sc
end

local function scareVisible(sc, on)
sc.visible = on
for _, obj in ipairs(sc.model:GetDescendants()) do
if obj:IsA("BasePart") then
if obj.Name == "HumanoidRootPart" then obj.Transparency = 1 else obj.Transparency = on and 0.08 or 1 end
end
end
end
---

-- Formas das piscinas (6x6 células de 8 studs; borda sempre seca)

local function wetSfx(vol, pitch)
if CFG.WaterSounds then
playUI(S.splash, vol, pitch)
else
playUI(S.step, vol * 0.45, pitch * 0.55)
end
end
local function wetStep()
if CFG.WaterSounds then
playUI(S.splash, 0.2, math.random(120, 150) / 100)
else
playUI(S.step, 0.2, math.random(60, 75) / 100)
end
end

local RGB = Color3.fromRGB
local THEMES = {
{ w = 3, tile = RGB(200, 214, 212), dark = RGB(170, 192, 192), teal = RGB(110, 165, 172), step = RGB(184, 205, 204), w1 = RGB(45, 150, 165), w2 = RGB(70, 180, 190), lamp = RGB(255, 246, 225), dim = 1 },
{ w = 1.2, tile = RGB(170, 200, 215), dark = RGB(130, 165, 185), teal = RGB(90, 130, 170), step = RGB(150, 185, 200), w1 = RGB(35, 110, 170), w2 = RGB(60, 140, 195), lamp = RGB(225, 238, 255), dim = 0.95 },
{ w = 1.2, tile = RGB(190, 212, 200), dark = RGB(150, 180, 165), teal = RGB(100, 160, 130), step = RGB(170, 200, 185), w1 = RGB(40, 150, 125), w2 = RGB(70, 180, 150), lamp = RGB(240, 255, 225), dim = 1 },
{ w = 1.2, tile = RGB(205, 198, 176), dark = RGB(180, 170, 140), teal = RGB(150, 165, 140), step = RGB(192, 184, 160), w1 = RGB(60, 160, 160), w2 = RGB(90, 190, 185), lamp = RGB(255, 236, 200), dim = 0.9 },
{ w = 1, tile = RGB(205, 190, 195), dark = RGB(180, 160, 168), teal = RGB(150, 130, 150), step = RGB(192, 176, 182), w1 = RGB(70, 155, 170), w2 = RGB(100, 185, 195), lamp = RGB(255, 235, 240), dim = 0.9 },
{ w = 1, tile = RGB(140, 152, 155), dark = RGB(105, 118, 122), teal = RGB(70, 105, 115), step = RGB(125, 138, 140), w1 = RGB(25, 95, 110), w2 = RGB(40, 120, 135), lamp = RGB(220, 245, 255), dim = 0.55 },
}
local function pickTheme(rng)
local total = 0
for _, t in ipairs(THEMES) do total += t.w end
local r = rng:NextNumber() * total
for _, t in ipairs(THEMES) do
r -= t.w
if r <= 0 then return t end
end
return THEMES[1]
end

local SHAPES = {
big = { "######", "#....#", "#....#", "#....#", "#....#", "######" },
L = { "######", "#..###", "#..###", "#....#", "#....#", "######" },
cross = { "######", "##..##", "#....#", "#....#", "##..##", "######" },
lanes = { "######", "#.#.##", "#.#.##", "#.#.##", "#.#.##", "######" },
ring = { "######", "#....#", "#.##.#", "#.##.#", "#....#", "######" },
snake = { "######", "#....#", "####.#", "#....#", "#.####", "######" },
dry = { "######", "######", "######", "######", "######", "######" },
pods = { "######", "#..#.#", "#..###", "###..#", "#.#..#", "######" },
olympic = { "######", "#....#", "#....#", "#....#", "######", "######" },
U = { "######", "#.##.#", "#.##.#", "#.##.#", "#....#", "######" },
H = { "######", "#.##.#", "#....#", "#....#", "#.##.#", "######" },
diag = { "######", "#.####", "##.###", "###.##", "####.#", "######" },
}
local PICK = {
{ "big", 2 }, { "L", 2 }, { "cross", 2 }, { "lanes", 2 }, { "ring", 2 },
{ "snake", 1 }, { "dry", 1.5 }, { "corridor", 2 }, { "halls", 1.5 },
{ "pods", 1.5 }, { "olympic", 1.5 }, { "U", 1.2 }, { "H", 1.2 }, { "diag", 1 }, { "split", 2 }, { "forest", 1.2 },
}

local function pickShape(rng)
local total = 0
for _, e in ipairs(PICK) do total += e[2] end
local r = rng:NextNumber() * total
for _, e in ipairs(PICK) do
r -= e[2]
if r <= 0 then return e[1] end
end
return "big"
end

local function buildMask(shape, rot)
local src = SHAPES[shape] or SHAPES.dry
local pool = {}
for r = 1, 6 do
pool[r] = {}
for c = 1, 6 do pool[r][c] = src[r]:sub(c, c) == "." end
end
for _ = 1, rot do
local new = {}
for r = 1, 6 do
new[r] = {}
for c = 1, 6 do new[r][c] = pool[7 - c][r] end
end
pool = new
end
return pool
end

local SIGNS = { "NO RUNNING", "DEPTH 0.5 m", "NO DIVING", "LEVEL 37", "POOL", "SWIM AT YOUR OWN RISK" }


---

-- Geração de chunk

local function genChunk(cx, cz)
local key = cx .. "," .. cz
if chunks[key] then return end
local rng = Random.new(hash(cx, cz, 1))
local TH = pickTheme(rng)
local C_TILE, C_DARK, C_TEAL, C_STEP = TH.tile, TH.dark, TH.teal, TH.step
local C_WATER1, C_WATER2 = TH.w1, TH.w2
local folder = inst("Folder")
folder.Name = obfName()
folder.Parent = root
local ox, oz = BASE.X + cx * CS, BASE.Z + cz * CS
local info = { cx = cx, cz = cz, folder = folder, water = {}, floaters = {}, flick = {}, suns = {}, caustics = {}, scares = {} }

local function B(x0, x1, y0, y1, z0, z1, color, mat, extra)
return mk(folder, Vector3.new(x1 - x0, y1 - y0, z1 - z0),
CFrame.new(ox + (x0 + x1) / 2, BY + (y0 + y1) / 2, oz + (z0 + z1) / 2),
color or C_TILE, mat or TILE, extra)
end

local shape = pickShape(rng)
local rot = rng:NextInteger(0, 3)
local pool = buildMask(shape, rot)
local function P(r, c) return pool[r] ~= nil and pool[r][c] == true end

-- PISO + ÁGUA ------------------------------------------------
for r = 1, 6 do
local c = 1
while c <= 6 do
local isPool = P(r, c)
local c2 = c
while c2 < 6 and P(r, c2 + 1) == isPool do c2 += 1 end
local x0, x1, z0, z1 = (c - 1) * CELL, c2 * CELL, (r - 1) * CELL, r * CELL
if isPool then
B(x0, x1, -4, -2.4, z0, z1, C_TILE, TILE)
B(x0 + 1, x1 - 1, -2.4, -2.37, (z0 + z1) / 2 - 0.25, (z0 + z1) / 2 + 0.25, C_TEAL, TILE)
local w = B(x0, x1, -2.4, -0.5, z0, z1, C_WATER1, Enum.Material.Glass,
{ Transparency = 0.5, CanCollide = false, CanQuery = false, CanTouch = false, Reflectance = 0.22 })
local lp = B((x0 + x1) / 2 - 0.1, (x0 + x1) / 2 + 0.1, 1.1, 1.3, (z0 + z1) / 2 - 0.1, (z0 + z1) / 2 + 0.1,
C_WATER2, Enum.Material.SmoothPlastic, { Transparency = 1, CanCollide = false, CanQuery = false, CanTouch = false })
local pl = inst("PointLight")
pl.Color = Color3.fromRGB(110, 235, 240)
pl.Brightness = 0.22
pl.Range = 24
pl.Shadows = true
pl.Parent = lp
local sl = inst("SpotLight")
sl.Color = Color3.fromRGB(90, 220, 255)
sl.Brightness = 0.8
sl.Range = 20
sl.Angle = 95
sl.Face = Enum.NormalId.Top
sl.Parent = lp
table.insert(info.water, { part = w, light = pl, ph = rng:NextNumber() * 6.28, c1 = C_WATER1, c2 = C_WATER2 })
else
B(x0, x1, -4, 0, z0, z1, C_TILE, TILE)
end
c = c2 + 1
end
end

-- Desgaste procedural: manchas, rachaduras estilizadas e pequenos detritos.
for _ = 1, rng:NextInteger(2, 5) do
local rr, rc = rng:NextInteger(1, 6), rng:NextInteger(1, 6)
local x, z = (rc - 1) * CELL + rng:NextNumber() * 6 + 1, (rr - 1) * CELL + rng:NextNumber() * 6 + 1
if P(rr, rc) then
local stain = B(x - 1.1, x + 1.1, -0.47, -0.43, z - 0.45, z + 0.45, Color3.fromRGB(42, 105, 112), Enum.Material.SmoothPlastic,
{ Transparency = 0.52, CanCollide = false, CanQuery = false, CanTouch = false, CastShadow = false })
stain.CFrame *= CFrame.Angles(0, rng:NextNumber() * 6.28, 0)
else
local crack = B(x - 0.85, x + 0.85, 0.015, 0.035, z - 0.045, z + 0.045, Color3.fromRGB(82, 100, 101), Enum.Material.SmoothPlastic,
{ Transparency = 0.28, CanCollide = false, CanQuery = false, CanTouch = false, CastShadow = false })
crack.CFrame *= CFrame.Angles(0, rng:NextNumber() * 6.28, 0)
end
end
for _ = 1, rng:NextInteger(1, 3) do
local x, z = rng:NextNumber() * 46 + 1, rng:NextNumber() * 46 + 1
if not P(math.clamp(math.floor(z / CELL) + 1, 1, 6), math.clamp(math.floor(x / CELL) + 1, 1, 6)) then
local debris = B(x - 0.12, x + 0.12, 0.04, 0.08, z - 0.65, z + 0.65, Color3.fromRGB(35, 48, 50), Enum.Material.SmoothPlastic,
{ CanCollide = false, CanQuery = false, CanTouch = false, CastShadow = false })
debris.CFrame *= CFrame.Angles(0, rng:NextNumber() * 6.28, rng:NextNumber() * 0.22 - 0.11)
end
end

-- DEGRAUS ----------------------------------------------------
local function step(x0, x1, z0, z1, top)
B(x0, x1, -2.4, top, z0, z1, C_STEP, TILE)
end
local function edgeRuns(alongRows, dr, dc, emit)
for o = 1, 6 do
local i = 1
while i <= 6 do
local r, c = o, i
if not alongRows then r, c = i, o end
if P(r, c) and not P(r + dr, c + dc) then
local j = i
while j < 6 do
local r2, c2 = o, j + 1
if not alongRows then r2, c2 = j + 1, o end
if P(r2, c2) and not P(r2 + dr, c2 + dc) then j += 1 else break end
end
emit(o, i, j)
i = j + 1
else
i += 1
end
end
end
end

local ladderDone = false
local function diving(x, z)
B(x - 1.6, x + 1.6, 0, 2.8, z - 6.5, z - 2.5, C_TILE, TILE)
B(x + 1.6, x + 3.2, 0, 1.87, z - 6.5, z - 3.5, C_STEP, TILE)
B(x + 3.2, x + 4.8, 0, 0.93, z - 6.5, z - 3.5, C_STEP, TILE)
B(x - 1, x + 1, 2.8, 3.05, z - 2.5, z + 4.5, Color3.fromRGB(232, 236, 236), Enum.Material.SmoothPlastic)
end
local function ladder(x, z)
local chrome = Color3.fromRGB(210, 218, 222)
for _, dx in ipairs({ -1.1, 1.1 }) do
local v = mk(folder, Vector3.new(5.2, 0.25, 0.25),
CFrame.new(ox + x + dx, BY + 0.1, oz + z + 0.5) * CFrame.Angles(0, 0, math.pi / 2),
chrome, Enum.Material.Metal, { CanCollide = false, Reflectance = 0.55 })
v.Shape = Enum.PartType.Cylinder
v.Size = Vector3.new(5.2, 0.25, 0.25)
local top = mk(folder, Vector3.new(1.4, 0.25, 0.25),
CFrame.new(ox + x + dx, BY + 2.6, oz + z - 0.1) * CFrame.Angles(0, math.pi / 2, 0),
chrome, Enum.Material.Metal, { CanCollide = false, Reflectance = 0.55 })
top.Shape = Enum.PartType.Cylinder
top.Size = Vector3.new(1.4, 0.25, 0.25)
end
for _, y in ipairs({ -0.9, -1.7 }) do
local rung = mk(folder, Vector3.new(2.2, 0.2, 0.2), CFrame.new(ox + x, BY + y, oz + z + 0.5),
chrome, Enum.Material.Metal, { CanCollide = false })
rung.Shape = Enum.PartType.Cylinder
rung.Size = Vector3.new(2.2, 0.2, 0.2)
end
end

edgeRuns(true, -1, 0, function(o, i, j)
local x0, x1, z = (i - 1) * CELL, j * CELL, (o - 1) * CELL
step(x0, x1, z, z + 2.5, -0.8)
step(x0, x1, z + 2.5, z + 5, -1.6)
if not ladderDone and rng:NextNumber() < 0.6 then
ladderDone = true
if rng:NextNumber() < 0.4 then
diving((x0 + x1) / 2, z)
else
ladder((x0 + x1) / 2, z + 0.4)
end
end
end)
edgeRuns(true, 1, 0, function(o, i, j)
local x0, x1, z = (i - 1) * CELL, j * CELL, o * CELL
step(x0, x1, z - 2.5, z, -0.8)
step(x0, x1, z - 5, z - 2.5, -1.6)
end)
edgeRuns(false, 0, -1, function(o, i, j)
local z0, z1, x = (i - 1) * CELL, j * CELL, (o - 1) * CELL
step(x, x + 2.5, z0, z1, -0.8)
step(x + 2.5, x + 5, z0, z1, -1.6)
end)
edgeRuns(false, 0, 1, function(o, i, j)
local z0, z1, x = (i - 1) * CELL, j * CELL, o * CELL
step(x - 2.5, x, z0, z1, -0.8)
step(x - 5, x - 2.5, z0, z1, -1.6)
end)

-- PAREDES / ARCOS --------------------------------------------
local T = 2
local wstyle = rng:NextInteger(1, 4)
local function wall(axis, fixed, u0, u1, c, w, kind)
local function seg(a, b, y0, y1, col, mat, thick)
thick = thick or T
if b - a <= 0.01 then return end
if axis == "x" then
B(a, b, y0, y1, fixed - thick / 2, fixed + thick / 2, col or C_TILE, mat or TILE)
else
B(fixed - thick / 2, fixed + thick / 2, y0, y1, a, b, col or C_TILE, mat or TILE)
end
end
local lo, hi = c - w / 2, c + w / 2
local function solid(a, b)
seg(a, b, -4, H)
if wstyle == 1 then
seg(a, b, 0, 1.1, C_DARK, TILE, T + 0.3)
seg(a, b, 8.6, 9.8, C_TEAL, TILE, T + 0.2)
elseif wstyle == 2 then
seg(a, b, 0, 6.5, C_TEAL, TILE, T + 0.2)
seg(a, b, 6.5, 7.1, C_DARK, TILE, T + 0.3)
elseif wstyle == 3 then
seg(a, b, 0, 1.1, C_DARK, TILE, T + 0.3)
else
seg(a, b, 11, 12, C_DARK, TILE, T + 0.25)
seg(a, b, 3, 3.6, C_TEAL, TILE, T + 0.2)
end
end
solid(u0, lo)
solid(hi, u1)
if kind == "arch" then
local n = math.floor(w)
local sl = w / n
for i = 0, n - 1 do
local ua = lo + i * sl
local dx = (ua + sl / 2) - c
local yy = 8 + math.sqrt(math.max(0, (w / 2) ^ 2 - dx * dx))
seg(ua, ua + sl, math.min(yy, H - 0.5), H)
end
else
seg(lo, hi, 12.5, H)
seg(lo, hi, 11.8, 12.5, C_DARK, TILE, T + 0.3)
end
end

local function sign(axis, u)
local txt = SIGNS[rng:NextInteger(1, #SIGNS)]
local p
if axis == "x" then
p = mk(folder, Vector3.new(4, 1.5, 0.1), CFrame.new(ox + u, BY + 6, oz + 1.05),
Color3.fromRGB(60, 150, 160), Enum.Material.SmoothPlastic, { CanCollide = false })
else
p = mk(folder, Vector3.new(0.1, 1.5, 4), CFrame.new(ox + 1.05, BY + 6, oz + u),
Color3.fromRGB(60, 150, 160), Enum.Material.SmoothPlastic, { CanCollide = false })
end
local sg = inst("SurfaceGui")
sg.Face = (axis == "x") and Enum.NormalId.Back or Enum.NormalId.Right
sg.SizingMode = Enum.SurfaceGuiSizingMode.PixelsPerStud
sg.PixelsPerStud = 50
sg.LightInfluence = 0.6
sg.Adornee = p
sg.Parent = p
local tl = inst("TextLabel")
tl.Size = UDim2.fromScale(1, 1)
tl.BackgroundTransparency = 1
tl.Text = txt
tl.TextScaled = true
tl.Font = Enum.Font.GothamBold
tl.TextColor3 = Color3.fromRGB(245, 255, 255)
tl.Parent = sg
end

local function outerWall(axis, seed)
local er = Random.new(hash(cx, cz, seed))
local roll = er:NextNumber()
local kind = roll < 0.55 and "arch" or (roll < 0.85 and "wide" or "none")
if kind == "none" then return end
local c = 24 + er:NextInteger(-6, 6)
wall(axis, 0, 0, CS, c, kind == "arch" and 14 or 22, kind)
if er:NextNumber() < 0.45 then sign(axis, er:NextNumber() < 0.5 and 4 or 44) end
end
outerWall("x", 2) -- parede norte
outerWall("z", 3) -- parede oeste

-- Pilar de canto
local roundCols = (shape == "forest") or rng:NextNumber() < 0.3
local function cyl(x, y0, y1, z, d, col)
local p = mk(folder, Vector3.new(y1 - y0, d, d),
CFrame.new(ox + x, BY + (y0 + y1) / 2, oz + z) * CFrame.Angles(0, 0, math.pi / 2), col, TILE)
p.Shape = Enum.PartType.Cylinder
p.Size = Vector3.new(y1 - y0, d, d)
return p
end
local function column(x, z, s, forceSq)
if roundCols and not forceSq then
cyl(x, -4, H, z, s, C_TILE)
cyl(x, -4, 1, z, s + 0.5, C_DARK)
cyl(x, 8.6, 9.8, z, s + 0.2, C_TEAL)
cyl(x, H - 1.2, H, z, s + 0.6, C_DARK)
return
end
B(x - s / 2, x + s / 2, -4, H, z - s / 2, z + s / 2, C_TILE, TILE)
B(x - s / 2 - 0.25, x + s / 2 + 0.25, -4, 1, z - s / 2 - 0.25, z + s / 2 + 0.25, C_DARK, TILE)
B(x - s / 2 - 0.1, x + s / 2 + 0.1, 8.6, 9.8, z - s / 2 - 0.1, z + s / 2 + 0.1, C_TEAL, TILE)
B(x - s / 2 - 0.3, x + s / 2 + 0.3, H - 1.2, H, z - s / 2 - 0.3, z + s / 2 + 0.3, C_DARK, TILE)
end
column(0, 0, 3, true)

-- Corredores internos
if shape == "corridor" then
local ax = rng:NextNumber() < 0.5 and "z" or "x"
wall(ax, 16, 0, CS, 24, 14, "arch")
wall(ax, 32, 0, CS, 24, 14, "arch")
end

if shape == "split" then
local ax = rng:NextNumber() < 0.5 and "z" or "x"
local pos = 18 + rng:NextInteger(0, 12)
local dc = 24 + rng:NextInteger(-8, 8)
local kd = rng:NextNumber() < 0.5 and "arch" or "wide"
wall(ax, pos, 0, CS, dc, kd == "arch" and 14 or 22, kd)
end

-- Colunas
if shape == "halls" then
for r = 2, 5 do
for c = 2, 5 do
if (r + c) % 2 == 0 then column((c - 1) * CELL + 4, (r - 1) * CELL + 4, 3.6) end
end
end
elseif shape == "forest" then
for r = 2, 5 do
for c = 2, 5 do
if rng:NextNumber() < 0.6 then
column((c - 1) * CELL + 4 + rng:NextNumber() * 3 - 1.5, (r - 1) * CELL + 4 + rng:NextNumber() * 3 - 1.5, 2.8)
end
end
end
elseif shape ~= "corridor" and shape ~= "split" then
local n = rng:NextInteger(2, 4)
for _ = 1, n do
local r, c = rng:NextInteger(2, 5), rng:NextInteger(2, 5)
column((c - 1) * CELL + 4, (r - 1) * CELL + 4, 3.6)
end
end

-- ELEMENTOS DE PISCINA ----------------------------------------
-- Props ambientais recorrentes, sem NPCs, entidades ou monstros.
local function poolRail(x, z, alongX)
local metal = Color3.fromRGB(205, 220, 220)
local yaw = alongX and 0 or math.pi / 2
local function tube(cf, length)
local q = mk(folder, Vector3.new(length, 0.22, 0.22), cf, metal, Enum.Material.Metal,
{ CanCollide = false, Reflectance = 0.6 })
q.Shape = Enum.PartType.Cylinder
return q
end
for _, dx in ipairs({ -2.4, 2.4 }) do
tube(CFrame.new(ox + x + (alongX and dx or 0), BY + 0.8, oz + z + (alongX and 0 or dx)) * CFrame.Angles(0, yaw, 0), 3.8)
end
tube(CFrame.new(ox + x, BY + 1.55, oz + z) * CFrame.Angles(0, yaw, 0), 5.1)
end

local function swimRing(x, z)
-- Segmentos simples formam uma boia sem depender de mesh externo.
for i = 1, 12 do
local a = (i / 12) * math.pi * 2
local q = mk(folder, Vector3.new(0.72, 0.28, 0.72),
CFrame.new(ox + x + math.cos(a) * 1.8, BY - 0.28, oz + z + math.sin(a) * 1.8),
Color3.fromRGB(245, 138, 82), Enum.Material.SmoothPlastic,
{ CanCollide = false, CanQuery = false, CanTouch = false, CastShadow = false })
q.Shape = Enum.PartType.Ball
end
end

local function woodWalkway(x, z, alongX)
for i = -2, 2 do
local px = x + (alongX and i * 1.7 or 0)
local pz = z + (alongX and 0 or i * 1.7)
mk(folder, alongX and Vector3.new(1.45, 0.22, 5.5) or Vector3.new(5.5, 0.22, 1.45),
CFrame.new(ox + px, BY + 0.12, oz + pz), Color3.fromRGB(116, 82, 58), Enum.Material.Wood,
{ Reflectance = 0.12 })
end
end

-- TETO ---------------------------------------------------------
local cr = rng:NextNumber()
local style = (cr < 0.4 and "coffer") or (cr < 0.65 and "strips") or (cr < 0.85 and "grid") or "vault"
B(0, CS, H, H + 2, 0, CS, C_TILE, TILE)
local flickN = 0
local function lamp(panel, flick, mult)
local sl = inst("SurfaceLight")
sl.Face = Enum.NormalId.Bottom
sl.Brightness = 0.85 * TH.dim * (mult or 1)
sl.Range = 24
sl.Angle = 170
sl.Color = TH.lamp
sl.Shadows = true
sl.Parent = panel
if rng:NextNumber() < 0.42 then
local hum = inst("Sound")
hum.SoundId = S.hum
hum.Volume = 0.018
hum.PlaybackSpeed = rng:NextNumber(0.92, 1.06)
hum.Looped = true
hum.RollOffMode = Enum.RollOffMode.InverseTapered
hum.RollOffMaxDistance = 38
hum.Parent = panel
hum:Play()
end
if flick and flickN < 1 and rng:NextNumber() < 0.2 then
flickN += 1
table.insert(info.flick, { panel = panel, light = sl, next = 0, on = true })
end
end
local function neon(x0, x1, z0, z1)
return B(x0, x1, H - 0.25, H, z0, z1, TH.lamp, Enum.Material.Neon)
end

if style == "coffer" then
for _, i in ipairs({ 12, 24, 36 }) do
B(0, CS, H - 1.6, H, i - 1, i + 1, C_TILE, TILE)
B(i - 1, i + 1, H - 1.6, H, 0, CS, C_TILE, TILE)
end
local coffer = {}
for i = 0, 3 do
for j = 0, 3 do table.insert(coffer, { 6 + 12 * i, 6 + 12 * j }) end
end
for k = 1, rng:NextInteger(3, 5) do
local cp = table.remove(coffer, rng:NextInteger(1, #coffer))
lamp(neon(cp[1] - 4, cp[1] + 4, cp[2] - 4, cp[2] + 4), k == 1)
end
elseif style == "strips" then
for _, i in ipairs({ 12, 24, 36 }) do
if rng:NextNumber() < 0.8 then lamp(neon(5, CS - 5, i - 1, i + 1), true) end
end
for _, i in ipairs({ 6, 18, 30, 42 }) do
B(0, CS, H - 0.5, H, i - 0.3, i + 0.3, C_DARK, TILE)
end
elseif style == "grid" then
for i = 0, 2 do
for j = 0, 2 do
if rng:NextNumber() < 0.75 then
local px, pz = 8 + 16 * i, 8 + 16 * j
lamp(neon(px - 2.5, px + 2.5, pz - 2.5, pz + 2.5), true, 0.8)
end
end
end
for _, i in ipairs({ 16, 32 }) do
B(0, CS, H - 0.6, H, i - 0.4, i + 0.4, C_DARK, TILE)
B(i - 0.4, i + 0.4, H - 0.6, H, 0, CS, C_DARK, TILE)
end
else
lamp(B(14, 34, H - 0.3, H, 14, 34, TH.lamp, Enum.Material.Neon), false, 1.5)
for _, e in ipairs({ { 13, 35, 13, 14 }, { 13, 35, 34, 35 }, { 13, 14, 13, 35 }, { 34, 35, 13, 35 } }) do
B(e[1], e[2], H - 1.2, H, e[3], e[4], C_DARK, TILE)
end
end

-- Caústicas no teto: placas neon discretas que oscilam como reflexos da água.
for _ = 1, rng:NextInteger(3, 6) do
local x, z = rng:NextNumber() * 44 + 2, rng:NextNumber() * 44 + 2
local q = B(x - 2.2, x + 2.2, H - 0.035, H - 0.015, z - 0.35, z + 0.35,
Color3.fromRGB(75, 195, 210), Enum.Material.Neon,
{ Transparency = 0.82, CanCollide = false, CanQuery = false, CanTouch = false, CastShadow = false })
table.insert(info.caustics, { part = q, ph = rng:NextNumber() * 6.28 })
end

-- JANELAS DE SOL + FEIXES DE LUZ ---------------------------------
local first = (cx == 0 and cz == 0)
if first or rng:NextNumber() < 0.4 then
for _ = 1, (first and 1 or rng:NextInteger(1, 2)) do
local wx, wz = 6 + 12 * rng:NextInteger(1, 3), 6 + 12 * rng:NextInteger(1, 3)
local sx, sz = rng:NextNumber() * 10 - 5, rng:NextNumber() * 10 - 5
if first then wx, wz, sx, sz = 30, 18, 5, 3 end
local gx = math.clamp(wx + sx * 2.2, 3, 45)
local gz = math.clamp(wz + sz * 2.2, 3, 45)

B(wx - 4.5, wx + 4.5, H - 0.3, H, wz - 4.5, wz + 4.5, Color3.fromRGB(255, 236, 190), Enum.Material.Neon)

local top = B(wx - 0.1, wx + 0.1, H - 0.6, H - 0.4, wz - 0.1, wz + 0.1, C_TILE, TILE, GHOST_T)
local bot = B(gx - 0.1, gx + 0.1, 0.2, 0.4, gz - 0.1, gz + 0.1, C_TILE, TILE, GHOST_T)
for k = 1, 3 do
local off = (k - 2) * 2.2
local a0 = inst("Attachment")
a0.Position = Vector3.new(off, 0, off * 0.5)
a0.Parent = top
local a1 = inst("Attachment")
a1.Position = Vector3.new(off * 1.5, 0, off * 0.7)
a1.Parent = bot
local bm = inst("Beam")
bm.Attachment0 = a0
bm.Attachment1 = a1
bm.Width0 = 2.6
bm.Width1 = 4.5
bm.FaceCamera = true
bm.LightEmission = 1
bm.LightInfluence = 0
bm.Color = ColorSequence.new(Color3.fromRGB(255, 232, 180))
bm.Transparency = NumberSequence.new({
NumberSequenceKeypoint.new(0, 0.86),
NumberSequenceKeypoint.new(0.5, 0.92),
NumberSequenceKeypoint.new(1, 0.97),
})
bm.Parent = top
end

local pr = math.clamp(math.floor(gz / CELL) + 1, 1, 6)
local pc = math.clamp(math.floor(gx / CELL) + 1, 1, 6)
local py = P(pr, pc) and -0.44 or 0.04
local panes = {}
for i = -1, 1, 2 do
for j = -1, 1, 2 do
local px, pz = gx + i * 1.55, gz + j * 1.55
table.insert(panes, B(px - 1.3, px + 1.3, py, py + 0.03, pz - 1.3, pz + 1.3,
Color3.fromRGB(255, 238, 190), Enum.Material.Neon,
{ Transparency = 0.55, CanCollide = false, CanQuery = false, CanTouch = false, CastShadow = false }))
end
end
local sl = inst("PointLight")
sl.Color = Color3.fromRGB(255, 225, 160)
sl.Brightness = 0.12
sl.Range = 14
sl.Shadows = true
sl.Parent = bot
table.insert(info.suns, { light = sl, panes = panes, ph = rng:NextNumber() * 6.28 })
end
end

-- PROPS ----------------------------------------------------------
if shape ~= "corridor" and shape ~= "split" and not (cx == 0 and cz == 0) and rng:NextNumber() < 0.45 then
for _ = 1, rng:NextInteger(1, 3) do
local r, c = rng:NextInteger(2, 5), rng:NextInteger(2, 5)
if not P(r, c) then
local x, z = (c - 1) * CELL + 4, (r - 1) * CELL + 4
if rng:NextNumber() < 0.5 then
B(x - 5, x + 5, 0, 3, z - 0.4, z + 0.4, C_DARK, TILE)
else
B(x - 0.4, x + 0.4, 0, 3, z - 5, z + 5, C_DARK, TILE)
end
end
end
end

if shape ~= "corridor" and not (cx == 0 and cz == 0) then
local spots = { { 7.5, 4.2 }, { 40.5, 4.2 }, { 7.5, 43.8 }, { 40.5, 43.8 } }
for _, sp in ipairs(spots) do
if rng:NextNumber() < 0.3 then
local x, z = sp[1], sp[2]
B(x - 4, x + 4, 1.2, 1.7, z - 1.1, z + 1.1, C_DARK, TILE)
B(x - 3.6, x - 2.6, 0, 1.2, z - 0.9, z + 0.9, C_TILE, TILE)
B(x + 2.6, x + 3.6, 0, 1.2, z - 0.9, z + 0.9, C_TILE, TILE)
end
end
end

for _ = 1, rng:NextInteger(1, 3) do
local r, c = rng:NextInteger(1, 6), rng:NextInteger(1, 6)
if not P(r, c) then
local x, z = (c - 1) * CELL + rng:NextNumber() * 4 + 2, (r - 1) * CELL + rng:NextNumber() * 4 + 2
B(x - 0.8, x + 0.8, 0, 0.06, z - 0.8, z + 0.8, Color3.fromRGB(70, 86, 90), Enum.Material.Metal)
end
end

-- Mesa de escritório impossível no corredor, com monitor ligado e mensagem curta.
if rng:NextNumber() < 0.16 then
local x, z = rng:NextNumber(10, 38), rng:NextNumber(10, 38)
local rr, rc = math.clamp(math.floor(z / CELL) + 1, 1, 6), math.clamp(math.floor(x / CELL) + 1, 1, 6)
if not P(rr, rc) then
B(x - 4, x + 4, 1.8, 2.15, z - 1.8, z + 1.8, Color3.fromRGB(80, 86, 86), Enum.Material.Wood)
for _, dx in ipairs({ -3.3, 3.3 }) do
B(x + dx - 0.18, x + dx + 0.18, 0, 1.8, z - 1.2, z - 0.85, Color3.fromRGB(52, 62, 63), Enum.Material.Metal)
B(x + dx - 0.18, x + dx + 0.18, 0, 1.8, z + 0.85, z + 1.2, Color3.fromRGB(52, 62, 63), Enum.Material.Metal)
end
local screen = B(x - 2.0, x + 2.0, 2.15, 4.55, z - 0.18, z + 0.18, Color3.fromRGB(8, 28, 32), Enum.Material.Neon,
{ CanCollide = false, CanQuery = false, CanTouch = false, Reflectance = 0.18 })
local sg = inst("SurfaceGui")
sg.Face = Enum.NormalId.Front
sg.SizingMode = Enum.SurfaceGuiSizingMode.PixelsPerStud
sg.PixelsPerStud = 40
sg.Adornee = screen
sg.Parent = screen
local label = inst("TextLabel")
label.Size = UDim2.fromScale(1, 1)
label.BackgroundTransparency = 1
label.Text = "NO SIGNAL"
label.TextScaled = true
label.Font = Enum.Font.Code
label.TextColor3 = Color3.fromRGB(120, 245, 232)
label.Parent = sg
local monitorLight = inst("PointLight")
monitorLight.Color = Color3.fromRGB(60, 220, 210)
monitorLight.Brightness = 0.45
monitorLight.Range = 10
monitorLight.Parent = screen
end
end

if #info.water > 0 and rng:NextNumber() < 0.7 then
for _ = 1, rng:NextInteger(1, 2) do
local w = info.water[rng:NextInteger(1, #info.water)]
local sp = w.part
local x = sp.Position.X + (rng:NextNumber() - 0.5) * math.max(0, sp.Size.X - 6)
local z = sp.Position.Z + (rng:NextNumber() - 0.5) * math.max(0, sp.Size.Z - 4)
local cf = CFrame.new(x, BY - 0.42, z) * CFrame.Angles(0, rng:NextNumber() * 6.28, 0)
local board = mk(folder, Vector3.new(3, 0.25, 1.7), cf, Color3.fromRGB(245, 250, 250),
Enum.Material.SmoothPlastic, { CanCollide = false, CanQuery = false })
table.insert(info.floaters, { part = board, base = cf, ph = rng:NextNumber() * 6.28 })
end
end

-- Variações arquitetônicas esparsas, sempre sem criaturas.
if rng:NextNumber() < 0.28 then
local rr, rc = rng:NextInteger(2, 5), rng:NextInteger(2, 5)
local ax, az = (rc - 1) * CELL + 4, (rr - 1) * CELL + 4
if P(rr, rc) then
swimRing(ax, az)
elseif rng:NextNumber() < 0.55 then
poolRail(ax, az, rng:NextNumber() < 0.5)
else
woodWalkway(ax, az, rng:NextNumber() < 0.5)
end
end
-- Aparições: algumas ficam no fundo da piscina, outras atravessam corredores.
if CFG.Scares and rng:NextNumber() < CFG.ScareChance then
local submerged = #info.water > 0 and rng:NextNumber() < 0.58
local at
if submerged then
local w = info.water[rng:NextInteger(1, #info.water)].part
at = Vector3.new(w.Position.X + rng:NextNumber(-2, 2), BY - 2.1, w.Position.Z + rng:NextNumber(-2, 2))
else
at = Vector3.new(ox + rng:NextNumber(5, 43), BY, oz + rng:NextNumber(5, 43))
end
table.insert(info.scares, createScare(folder, at, submerged, rng))
end

chunks[key] = info
end

local function chunkOf(pos)
return math.floor((pos.X - BASE.X) / CS), math.floor((pos.Z - BASE.Z) / CS)
end

local function refreshQueue(pcx, pcz)
local R = CFG.Radius
queue = {}
for dx = -R, R do
for dz = -R, R do
if not chunks[(pcx + dx) .. "," .. (pcz + dz)] then
table.insert(queue, { pcx + dx, pcz + dz, dx * dx + dz * dz })
end
end
end
table.sort(queue, function(a, b) return a[3] < b[3] end)
for k, ch in pairs(chunks) do
if math.max(math.abs(ch.cx - pcx), math.abs(ch.cz - pcz)) > R + 1 then
ch.folder:Destroy()
chunks[k] = nil
end
end
end

local function nearChunks(pcx, pcz, rad)
local list = {}
for dx = -rad, rad do
for dz = -rad, rad do
local ch = chunks[(pcx + dx) .. "," .. (pcz + dz)]
if ch then table.insert(list, ch) end
end
end
return list
end


---

-- Ambiente (Lighting / efeitos)
-- Iluminação principal: luminárias no teto; água e janelas funcionam apenas como preenchimento sutil.

local applyLighting_cc
local function setL(name, val) pcall(function() Lighting[name] = val end) end

local function applyLighting()
for _, p in ipairs({ "Ambient", "OutdoorAmbient", "Brightness", "ClockTime", "FogColor", "FogStart", "FogEnd",
"ExposureCompensation", "EnvironmentDiffuseScale", "EnvironmentSpecularScale", "Technology", "GlobalShadows", "ShadowSoftness" }) do
pcall(function() saved.lighting[p] = Lighting[p] end)
end
for _, o in ipairs(Lighting:GetChildren()) do
if o:IsA("PostEffect") then
table.insert(saved.disabled, { o, o.Enabled })
o.Enabled = false
elseif o:IsA("Atmosphere") or o:IsA("Sky") or o:IsA("Clouds") then
table.insert(saved.removed, o)
o.Parent = nil
end
end
setL("Ambient", CFG.Ambient)
setL("OutdoorAmbient", CFG.Ambient)
setL("Brightness", 0.03)
setL("Technology", Enum.Technology.Future)
setL("GlobalShadows", true)
setL("ShadowSoftness", 0.18)
setL("ClockTime", 12)
setL("FogColor", Color3.fromRGB(22, 32, 36))
setL("FogStart", 8)
setL("FogEnd", CFG.Radius * CS * 0.75)
setL("ExposureCompensation", CFG.Exposure)
setL("EnvironmentDiffuseScale", 0.3)
setL("EnvironmentSpecularScale", 0.7)

local atm = inst("Atmosphere")
atm.Density = 0.42
atm.Offset = 0.1
atm.Color = Color3.fromRGB(55, 80, 86)
atm.Decay = Color3.fromRGB(35, 55, 60)
atm.Glare = 0
atm.Haze = 0.7
atm.Parent = Lighting
table.insert(effects, atm)

local bloom = inst("BloomEffect")
bloom.Intensity = 0.35
bloom.Size = 22
bloom.Threshold = 1
bloom.Parent = Lighting
table.insert(effects, bloom)

local rays = inst("SunRaysEffect")
rays.Intensity = 0.035
rays.Spread = 0.65
rays.Parent = Lighting
table.insert(effects, rays)

local cc = inst("ColorCorrectionEffect")
cc.Brightness = -0.16
cc.Contrast = 0.28
cc.Saturation = -0.12
cc.TintColor = Color3.fromRGB(190, 215, 220)
cc.Parent = Lighting
table.insert(effects, cc)
applyLighting_cc = cc

local dof = inst("DepthOfFieldEffect")
dof.FarIntensity = 0.18
dof.NearIntensity = 0
dof.FocusDistance = 45
dof.InFocusRadius = 40
dof.Parent = Lighting
table.insert(effects, dof)

local blur = inst("BlurEffect")
blur.Size = 0
blur.Parent = Lighting
table.insert(effects, blur)
blurFx = blur

pcall(function() SoundService.AmbientReverb = Enum.ReverbType.Auditorium end)
end


---

-- GUI (intro + vinheta)

local function buildGui()
gui = inst("ScreenGui")
gui.Name = obfName()
gui.IgnoreGuiInset = true
gui.ResetOnSpawn = false
gui.DisplayOrder = 999
pcall(function() gui.Parent = (gethui and gethui()) or game:GetService("CoreGui") end)
if not gui.Parent then gui.Parent = player:WaitForChild("PlayerGui") end

local hint = inst("TextLabel")
hint.BackgroundTransparency = 1
hint.Position = UDim2.new(0, 12, 1, -28)
hint.Size = UDim2.new(0, 400, 0, 20)
hint.TextXAlignment = Enum.TextXAlignment.Left
hint.Font = Enum.Font.Gotham
hint.TextSize = 13
hint.TextTransparency = 0.4
hint.TextColor3 = Color3.fromRGB(235, 255, 255)
hint.Text = "RUN: botão / R / SHIFT  •  P sair"
hint.Parent = gui

-- BOTÃO RUN -----------------------------------------------------
-- Interface minimalista: apenas o texto RUN.
local btn = inst("TextButton")
btn.Name = obfName()
btn.AnchorPoint = Vector2.new(1, 1)
btn.Position = UDim2.new(1, -24, 1, -105)
btn.Size = UDim2.fromOffset(128, 48)
btn.AutoButtonColor = false
btn.Text = "RUN"
btn.Font = Enum.Font.GothamBold
btn.TextSize = 20
btn.TextColor3 = Color3.fromRGB(235, 255, 255)
btn.BackgroundColor3 = Color3.fromRGB(12, 30, 36)
btn.BackgroundTransparency = 0.12
btn.BorderSizePixel = 0
btn.ZIndex = 5
btn.Parent = gui
local bc = inst("UICorner")
bc.CornerRadius = UDim.new(0, 12)
bc.Parent = btn
local bs = inst("UIStroke")
bs.Thickness = 2
bs.Color = Color3.fromRGB(100, 190, 205)
bs.Transparency = 0.15
bs.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
bs.Parent = btn
local sc = inst("UIScale")
sc.Parent = btn

local pulse
updateRunBtn = function()
local on = runOn or shiftHeld
btn.TextColor3 = on and Color3.fromRGB(145, 255, 210) or Color3.fromRGB(235, 255, 255)
TweenService:Create(bs, TweenInfo.new(0.2), {
Color = on and Color3.fromRGB(80, 240, 255) or Color3.fromRGB(100, 190, 205),
Thickness = on and 3 or 2,
}):Play()
TweenService:Create(btn, TweenInfo.new(0.2), {
BackgroundColor3 = on and Color3.fromRGB(15, 92, 84) or Color3.fromRGB(12, 30, 36),
}):Play()
if pulse then pulse:Cancel() pulse = nil end
if on then
bs.Transparency = 0.05
pulse = TweenService:Create(bs, TweenInfo.new(0.7, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true),
{ Transparency = 0.65 })
pulse:Play()
else
bs.Transparency = 0.15
end
end

table.insert(conns, btn.Activated:Connect(function()
runOn = not runOn
refreshRun()
end))
table.insert(conns, btn.MouseButton1Down:Connect(function()
TweenService:Create(sc, TweenInfo.new(0.08), { Scale = 0.94 }):Play()
end))
table.insert(conns, btn.MouseButton1Up:Connect(function()
TweenService:Create(sc, TweenInfo.new(0.12, Enum.EasingStyle.Back), { Scale = 1 }):Play()
end))
table.insert(conns, btn.MouseEnter:Connect(function()
TweenService:Create(btn, TweenInfo.new(0.15), { BackgroundTransparency = 0 }):Play()
end))
table.insert(conns, btn.MouseLeave:Connect(function()
TweenService:Create(btn, TweenInfo.new(0.15), { BackgroundTransparency = 0.12 }):Play()
TweenService:Create(sc, TweenInfo.new(0.12), { Scale = 1 }):Play()
end))

local cover = inst("Frame")
cover.Name = obfName()
cover.Size = UDim2.fromScale(1, 1)
cover.BackgroundColor3 = Color3.new(0, 0, 0)
cover.BorderSizePixel = 0
cover.ZIndex = 50
cover.Parent = gui
local title = inst("TextLabel")
title.BackgroundTransparency = 1
title.Size = UDim2.fromScale(1, 1)
title.Font = Enum.Font.Gotham
title.TextSize = 38
title.Text = "T H E   P O O L R O O M S"
title.TextColor3 = Color3.fromRGB(215, 245, 245)
title.ZIndex = 51
title.Parent = cover
return cover, title
end

-- Inventário exclusivo da experiência: um único slot visual com preview 3D da lanterna.
local function createFlashlightInventory()
local backpack = player:WaitForChild("Backpack")
local tool = Instance.new("Tool")
tool.Name = "Lanterna"
tool.ToolTip = "Lanterna"
tool.RequiresHandle = true
tool.CanBeDropped = false
tool.ManualActivationOnly = false
tool.Grip = CFrame.new(0, -0.18, -1.15) * CFrame.Angles(math.rad(-8), math.rad(180), 0)
tool.Parent = backpack
lanternTool = tool

local function piece(name, size, cf, color, material, transparency)
local q = Instance.new("Part")
q.Name = name
q.Size = size
q.CFrame = cf
q.Color = color
q.Material = material or Enum.Material.SmoothPlastic
q.Transparency = transparency or 0
q.CanCollide = false
q.CanTouch = false
q.CanQuery = false
q.Massless = true
q.CastShadow = true
q.Parent = tool
return q
end
local black = Color3.fromRGB(13, 18, 23)
local metal = Color3.fromRGB(88, 104, 112)
local cyan = Color3.fromRGB(115, 245, 255)
local handle = piece("Handle", Vector3.new(0.62, 1.65, 0.62), CFrame.new(0, 0, 0) * CFrame.Angles(0, 0, math.rad(90)), black, Enum.Material.Metal)
handle.Shape = Enum.PartType.Cylinder
local cap = piece("TopCap", Vector3.new(0.72, 0.16, 0.72), CFrame.new(0, 0.86, 0), metal, Enum.Material.Metal)
cap.Shape = Enum.PartType.Cylinder
local bezel = piece("Bezel", Vector3.new(0.86, 0.22, 0.86), CFrame.new(0, 0.62, 0), metal, Enum.Material.Metal)
bezel.Shape = Enum.PartType.Cylinder
local lens = piece("Lens", Vector3.new(0.68, 0.08, 0.68), CFrame.new(0, 0.76, 0), cyan, Enum.Material.Neon)
lens.Shape = Enum.PartType.Cylinder
lens.CastShadow = false
local bottom = piece("BottomCap", Vector3.new(0.7, 0.14, 0.7), CFrame.new(0, -0.88, 0), metal, Enum.Material.Metal)
bottom.Shape = Enum.PartType.Cylinder
for _, y in ipairs({ -0.64, 0.38 }) do
local ring = piece("GripRing", Vector3.new(0.7, 0.08, 0.7), CFrame.new(0, y, 0), metal, Enum.Material.Metal)
ring.Shape = Enum.PartType.Cylinder
end
local side = piece("SideGrip", Vector3.new(0.18, 1.15, 0.32), CFrame.new(0.43, -0.04, 0), black, Enum.Material.Rubber)
local glow = Instance.new("PointLight")
glow.Name = "LanternGlow"
glow.Color = cyan
glow.Brightness = 2.4
glow.Range = 14
glow.Shadows = true
glow.Enabled = true
glow.Parent = lens
local spot = Instance.new("SpotLight")
spot.Name = "LanternBeam"
spot.Color = Color3.fromRGB(210, 250, 255)
spot.Brightness = 8
spot.Range = 70
spot.Angle = 58
spot.Face = Enum.NormalId.Top
spot.Shadows = true
spot.Enabled = true
spot.Parent = lens
table.insert(lanternLights, glow)
table.insert(lanternLights, spot)
for _, q in ipairs({ cap, bezel, lens, bottom, side }) do
local weld = Instance.new("WeldConstraint")
weld.Part0 = handle
weld.Part1 = q
weld.Parent = handle
end

local function setFlashlight(on)
flashlightOn = on
for _, light in ipairs(lanternLights) do light.Enabled = on end
if inventoryGui then
local slot = inventoryGui:FindFirstChild("Slot", true)
local stroke = slot and slot:FindFirstChildOfClass("UIStroke")
if stroke then stroke.Color = on and Color3.fromRGB(100, 240, 255) or Color3.fromRGB(55, 65, 70) end
end
end
tool.Activated:Connect(function() setFlashlight(not flashlightOn) end)
tool.Equipped:Connect(function() setFlashlight(true) end)

inventoryGui = Instance.new("ScreenGui")
inventoryGui.Name = obfName()
inventoryGui.IgnoreGuiInset = true
inventoryGui.ResetOnSpawn = false
inventoryGui.DisplayOrder = 1000
inventoryGui.Parent = (gethui and gethui()) or player:WaitForChild("PlayerGui")
local slot = Instance.new("ImageButton")
slot.Name = "Slot"
slot.AnchorPoint = Vector2.new(0.5, 1)
slot.Position = UDim2.new(0.5, 0, 1, -24)
slot.Size = UDim2.fromOffset(92, 92)
slot.BackgroundColor3 = Color3.fromRGB(8, 19, 24)
slot.BackgroundTransparency = 0.08
slot.AutoButtonColor = false
slot.Image = ""
slot.Parent = inventoryGui
local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 14)
corner.Parent = slot
local outline = Instance.new("UIStroke")
outline.Thickness = 2
outline.Color = Color3.fromRGB(100, 240, 255)
outline.Transparency = 0.08
outline.Parent = slot
local viewport = Instance.new("ViewportFrame")
viewport.Name = "LanternModel"
viewport.Size = UDim2.fromScale(0.86, 0.86)
viewport.Position = UDim2.fromScale(0.07, 0.05)
viewport.BackgroundTransparency = 1
viewport.Parent = slot
local world = Instance.new("WorldModel")
world.Parent = viewport
for _, q in ipairs(tool:GetChildren()) do
if q:IsA("BasePart") then
local clone = q:Clone()
clone.Anchored = true
clone.CFrame = CFrame.new(0, 0, 0) * q.CFrame
clone.Parent = world
end
end
local cam = Instance.new("Camera")
cam.CFrame = CFrame.lookAt(Vector3.new(3.1, 2.1, 4.1), Vector3.new(0, 0, 0))
cam.Parent = viewport
viewport.CurrentCamera = cam
local key = Instance.new("TextLabel")
key.BackgroundTransparency = 1
key.AnchorPoint = Vector2.new(1, 1)
key.Position = UDim2.new(1, -7, 1, -5)
key.Size = UDim2.fromOffset(24, 18)
key.Text = "F"
key.Font = Enum.Font.GothamBold
key.TextSize = 14
key.TextColor3 = Color3.fromRGB(220, 255, 255)
key.Parent = slot
slot.Activated:Connect(function()
if lanternTool.Parent == backpack then
local ch = player.Character
		local hum = ch and ch:FindFirstChildOfClass("Humanoid")
if hum then hum:EquipTool(lanternTool) end
end
setFlashlight(not flashlightOn)
end)
pcall(function()
savedBackpackEnabled = StarterGui:GetCoreGuiEnabled(Enum.CoreGuiType.Backpack)
StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.Backpack, false)
end)
local ch = player.Character
		local hum = ch and ch:FindFirstChildOfClass("Humanoid")
if hum then hum:EquipTool(tool) end
end


---

-- Personagem

local function getChar()
local ch = player.Character
if not ch then return end
return ch, ch:FindFirstChildOfClass("Humanoid"), ch:FindFirstChild("HumanoidRootPart")
end

-- O Roblox pode recriar Running/Footsteps no personagem; silencia todos para evitar áudio duplicado.
local function muteDefaultFootsteps(ch)
if not ch then return end
local function mute(obj)
if obj:IsA("Sound") then
local n = string.lower(obj.Name)
if string.find(n, "running", 1, true) or string.find(n, "foot", 1, true) or string.find(n, "step", 1, true) then
obj.Volume = 0
end
end
end
for _, obj in ipairs(ch:GetDescendants()) do mute(obj) end
table.insert(conns, ch.DescendantAdded:Connect(mute))
end

local function showBodyInFirstPerson(ch)
if not ch then return end
for _, obj in ipairs(ch:GetDescendants()) do
if obj:IsA("BasePart") then
-- Esconde apenas a cabeça; corpo e braços ficam visíveis ao olhar para baixo.
obj.LocalTransparencyModifier = (obj.Name == "Head") and 1 or 0
end
end
end

local function hookBodyVisibility(ch)
showBodyInFirstPerson(ch)
local c = ch.DescendantAdded:Connect(function(obj)
if obj:IsA("BasePart") then
obj.LocalTransparencyModifier = (obj.Name == "Head") and 1 or 0
end
end)
table.insert(conns, c)
end

local function applyCam()
pcall(function() player.CameraMode = Enum.CameraMode.LockFirstPerson end)
pcall(function()
player.CameraMinZoomDistance = 0.5
player.CameraMaxZoomDistance = 0.5
end)
end

local function tpToStart()
local ch, hum, rp = getChar()
if not rp then return end
local pos = Vector3.new(BASE.X + 4.5, BY + 4.5, BASE.Z + 20)
ch:PivotTo(CFrame.lookAt(pos, pos + Vector3.new(1, 0, 0)))
rp.AssemblyLinearVelocity = Vector3.zero
acGraceUntil = os.clock() + 3
acState.lastPos = pos
acState.lastTime = os.clock()
acState.safeCF = CFrame.lookAt(pos, pos + Vector3.new(1, 0, 0))
acState.samples = {}
end

local function feetOff(hum, rp)
return (hum.RigType == Enum.HumanoidRigType.R6) and 3 or (hum.HipHeight + rp.Size.Y / 2)
end

local function feetInWater(hum, rp)
local fy = rp.Position.Y - feetOff(hum, rp)
return fy < BY - 0.6 and fy > BY - 3
end


---

-- ANTICHEAT (cliente)
-- Detecta: fly, float/hover, tpwalk, speed, noclip, swim-fly, pulo forçado/infjump, body movers,
-- troca de propriedades/estados do Humanoid, gravidade, freecam e remoção do próprio loop.
-- Reação: reverte pra última posição segura, registra strike e kicka ao acumular strikes.
-- Limite: tudo roda no cliente. Quem controla o cliente pode tentar mexer no próprio detector.

local function acFlag(reason, detail)
local now = os.clock()
if acState.lastFlags[reason] and now - acState.lastFlags[reason] < AC.FlagCooldown then return end
acState.lastFlags[reason] = now
table.insert(acState.strikes, now)
while #acState.strikes > 0 and now - acState.strikes[1] > AC.StrikeWindow do table.remove(acState.strikes, 1) end
acState.lastReason = reason
warn(("[Poolrooms AC] %s%s"):format(reason, detail and (" | " .. detail) or ""))
if AC.KickOnDetection and #acState.strikes >= AC.MaxStrikes then
pcall(function() player:Kick("Movimento inválido detectado: " .. reason) end)
end
end

local function propOk(v, e)
if type(v) == "number" and type(e) == "number" then return math.abs(v - e) <= 0.1 end
return v == e
end

local function isMover(o)
for _, c in ipairs(MOVER_CLASSES) do
if o:IsA(c) then return true end
end
return false
end

local function lockStates(hum)
for _, s in ipairs(LOCKED_STATES) do
pcall(function() hum:SetStateEnabled(s, false) end)
end
end

-- distância dos pés até o chão que colide (99 = sem chão embaixo)
local function acGap(hum, rp)
local hit = Workspace:Raycast(rp.Position, Vector3.new(0, -60, 0), acRay)
if not hit then return 99 end
return rp.Position.Y - feetOff(hum, rp) - hit.Position.Y
end

local function installACWatchers(ch)
if not ch then return end
local hum = ch:FindFirstChildOfClass("Humanoid")
local rp = ch:FindFirstChild("HumanoidRootPart")
if not hum or not rp then return end
local ex = acState.expected
-- valores esperados primeiro, depois aplica (evita falso positivo nos próprios sets)
ex.WalkSpeed = hum.WalkSpeed
ex.HipHeight = hum.HipHeight
ex.MaxSlopeAngle = hum.MaxSlopeAngle
ex.JumpPower, ex.JumpHeight, ex.UseJumpPower = 0, 0, true
ex.PlatformStand, ex.AutoRotate, ex.Sit = false, true, false
pcall(function()
hum.UseJumpPower = true
hum.JumpPower = 0
hum.JumpHeight = 0
end)
lockStates(hum)

for _, prop in ipairs(WATCHED_PROPS) do
table.insert(conns, hum:GetPropertyChangedSignal(prop):Connect(function()
local now = os.clock()
if now < acGraceUntil or now < acState.lagUntil then return end
local v, e = hum[prop], acState.expected[prop]
if propOk(v, e) then return end
acFlag("property_changed", ("%s=%s"):format(prop, tostring(v)))
pcall(function() hum[prop] = e end)
end))
end

table.insert(conns, hum.StateChanged:Connect(function(_, new)
local now = os.clock()
if now < acGraceUntil or now < acState.lagUntil then return end
if BAD_STATES[new] then
acFlag(new == HST.Jumping and "forced_jump" or ("state_" .. new.Name))
pcall(function() hum:ChangeState(HST.Running) end)
end
end))

-- qualquer BodyVelocity/LinearVelocity/etc. colocado no personagem é removido na hora
table.insert(conns, ch.DescendantAdded:Connect(function(o)
if isMover(o) then
acFlag("body_mover", o.ClassName)
task.defer(function() pcall(function() o:Destroy() end) end)
end
end))
end

local function clientAntiCheatStep(dt)
if not AC.Enabled then return end
local ch, hum, rp = getChar()
if not ch or not hum or not rp then return end
local now = os.clock()
local pos = rp.Position
local ex = acState.expected

local function resetTrack()
acState.lastPos, acState.lastTime = pos, now
acState.samples = {}
acState.airT, acState.noclipT, acState.upT, acState.idleT, acState.fastT, acState.jumpF = 0, 0, 0, 0, 0, 0
end

if now < acGraceUntil then
-- durante a tolerância, adota HipHeight/slope reais (rig pode escalar logo após o spawn)
ex.HipHeight = hum.HipHeight
ex.MaxSlopeAngle = hum.MaxSlopeAngle
resetTrack()
return
end
if dt > AC.LagSpikeThreshold then
-- Não transforma um frame congelado em falso tpwalk/noclip.
acState.lagUntil = now + AC.LagRecovery
resetTrack()
return
end
if now < acState.lagUntil then
resetTrack()
return
end

local bad = false
local function violation(reason, detail, keep)
if not keep then bad = true end
acFlag(reason, detail)
end

-- física inválida / fling
local vel = rp.AssemblyLinearVelocity
if vel.X ~= vel.X or vel.Y ~= vel.Y or vel.Z ~= vel.Z or vel.Magnitude > 180 then
violation("physics_nan_or_fling", tostring(vel.Magnitude))
rp.AssemblyLinearVelocity = Vector3.zero
vel = Vector3.zero
end

-- gravidade e root ancorado
if Workspace.Gravity ~= acState.gravity then
violation("gravity_changed", tostring(Workspace.Gravity), true)
pcall(function() Workspace.Gravity = acState.gravity end)
end
if rp.Anchored then
violation("anchored_root", nil, true)
pcall(function() rp.Anchored = false end)
end

-- propriedades do Humanoid: qualquer desvio é revertido e flagado
for _, p in ipairs(WATCHED_PROPS) do
local v, e = hum[p], ex[p]
if not propOk(v, e) then
violation("humanoid_property", ("%s=%s"):format(p, tostring(v)), true)
pcall(function() hum[p] = e end)
end
end

-- PULO: desativado. Qualquer tentativa que chegue até aqui é pulo forçado.
if hum.Jump then acState.jumpF += 1 else acState.jumpF = 0 end
if acState.jumpF >= 2 then violation("forced_jump", "Humanoid.Jump", true) end
if hum.Jump then hum.Jump = false end
for _, s in ipairs(LOCKED_STATES) do
local okS, en = pcall(function() return hum:GetStateEnabled(s) end)
if okS and en then
violation("state_unlocked", s.Name, true)
pcall(function() hum:SetStateEnabled(s, false) end)
end
end
local st = hum:GetState()
if BAD_STATES[st] then
violation(st == HST.Jumping and "forced_jump" or ("state_" .. st.Name))
pcall(function() hum:ChangeState(HST.Running) end)
end

-- altitude (sem pulo, nada legítimo passa de ~3 studs acima do piso)
local feetY = pos.Y - feetOff(hum, rp)
if feetY > BY + AC.MaxFeetHeight then
violation("altitude", ("%.1f"):format(feetY - BY))
end

-- voo / hover: no ar sem queda livre compatível com a gravidade
local gap = acGap(hum, rp)
if gap > AC.MinGap and hum.FloorMaterial == Enum.Material.Air then
acState.airT += dt
else
acState.airT = 0
end
if acState.airT > AC.AirGrace and vel.Y > -0.4 * acState.gravity * acState.airT then
violation("fly", ("gap=%.1f vy=%.1f"):format(math.min(gap, 99), vel.Y))
end
if vel.Y > AC.MaxUpVel then acState.upT += dt else acState.upT = 0 end
if acState.upT > 0.12 then violation("upward_velocity", ("%.1f"):format(vel.Y)) end

-- velocidade horizontal (média numa janela: pega tpwalk de passos pequenos)
local hs = Vector3.new(vel.X, 0, vel.Z).Magnitude
table.insert(acState.samples, { t = now, p = pos, ws = ex.WalkSpeed })
while #acState.samples > 1 and now - acState.samples[1].t > AC.SpeedWindow do table.remove(acState.samples, 1) end
local firstS = acState.samples[1]
local span = now - firstS.t
if span >= AC.SpeedWindow * 0.7 then
local maxWs = 0
for _, s in ipairs(acState.samples) do
if s.ws > maxWs then maxWs = s.ws end
end
local avg = Vector3.new(pos.X - firstS.p.X, 0, pos.Z - firstS.p.Z).Magnitude / span
if avg > maxWs * AC.SpeedFactor + AC.SpeedSlack then
violation("speed_or_tpwalk", ("avg=%.1f max=%.1f"):format(avg, maxWs * AC.SpeedFactor + AC.SpeedSlack))
end
end
if hs > ex.WalkSpeed * 1.4 + 4 then acState.fastT += dt else acState.fastT = 0 end
if acState.fastT > 0.25 then violation("velocity_speed", ("%.1f"):format(hs)) end
-- movimento sem input (BodyVelocity/slide)
if hum.MoveDirection.Magnitude < 0.1 and hs > 5 and gap <= AC.MinGap then acState.idleT += dt else acState.idleT = 0 end
if acState.idleT > 0.5 then violation("no_input_motion", ("%.1f"):format(hs)) end

-- teleporte por frame + atravessar parede
if acState.lastPos then
local elapsed = math.max(now - acState.lastTime, 1 / 60)
local delta = pos - acState.lastPos
local horiz = Vector3.new(delta.X, 0, delta.Z)
local allowed = ex.WalkSpeed * 1.5 * elapsed + 2
if horiz.Magnitude > allowed or delta.Magnitude > AC.TeleportDistance then
violation("tpwalk_or_teleport", ("distance=%.1f allowed=%.1f"):format(delta.Magnitude, allowed))
end
if horiz.Magnitude > 0.35 then
local wall = Workspace:Raycast(acState.lastPos, horiz, acRay)
if wall then violation("noclip", wall.Instance:GetFullName()) end
end
end

-- noclip: corpo dentro de geometria sólida
local inside = Workspace:GetPartBoundsInBox(rp.CFrame, Vector3.new(0.6, 2, 0.6), acOverlap)
if #inside > 0 then acState.noclipT += dt else acState.noclipT = 0 end
if acState.noclipT > 0.2 then violation("noclip", inside[1]:GetFullName()) end

-- freecam / câmera longe do personagem
local cam = Workspace.CurrentCamera
if cam then
local head = ch:FindFirstChild("Head")
if cam.CameraType == Enum.CameraType.Scriptable
or (cam.CameraSubject and cam.CameraSubject ~= hum)
or (head and (cam.CFrame.Position - head.Position).Magnitude > 8) then
violation("freecam", nil, true)
end
end

if bad then
if acState.safeCF then
pcall(function() ch:PivotTo(acState.safeCF) end)
rp.AssemblyLinearVelocity = Vector3.zero
pos = rp.Position
end
resetTrack()
return
end
if gap <= AC.MinGap then acState.safeCF = rp.CFrame end
acState.lastPos, acState.lastTime = pos, now
end


---

-- FX globais

local fxPart = mk(root, Vector3.one * 0.2, CFrame.new(BASE), C_TILE, Enum.Material.SmoothPlastic,
{ Transparency = 1, CanCollide = false, CanQuery = false, CanTouch = false })
local fxAtt = inst("Attachment")
fxAtt.Parent = fxPart
local splashEm = inst("ParticleEmitter")
splashEm.Rate = 0
splashEm.Lifetime = NumberRange.new(0.5, 0.9)
splashEm.Speed = NumberRange.new(5, 10)
splashEm.SpreadAngle = Vector2.new(0, 28)
splashEm.EmissionDirection = Enum.NormalId.Top
splashEm.Acceleration = Vector3.new(0, -35, 0)
splashEm.Size = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0.28), NumberSequenceKeypoint.new(1, 0.04) })
splashEm.Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0.25), NumberSequenceKeypoint.new(1, 1) })
splashEm.LightEmission = 0.6
splashEm.Color = ColorSequence.new(Color3.fromRGB(215, 250, 255))
splashEm.Parent = fxAtt

local function splashFx(pos, n)
fxAtt.WorldPosition = pos
splashEm:Emit(n)
end

local motes = mk(root, Vector3.new(90, 26, 90), CFrame.new(BASE), C_TILE, Enum.Material.SmoothPlastic,
{ Transparency = 1, CanCollide = false, CanQuery = false, CanTouch = false })
local moteEm = inst("ParticleEmitter")
moteEm.Shape = Enum.ParticleEmitterShape.Box
moteEm.ShapeStyle = Enum.ParticleEmitterShapeStyle.Volume
moteEm.Rate = 30
moteEm.Lifetime = NumberRange.new(8, 12)
moteEm.Speed = NumberRange.new(0.05, 0.25)
moteEm.SpreadAngle = Vector2.new(180, 180)
moteEm.Size = NumberSequence.new(0.14)
moteEm.Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(0.2, 0.7),
NumberSequenceKeypoint.new(0.8, 0.7), NumberSequenceKeypoint.new(1, 1) })
moteEm.LightEmission = 0.8
moteEm.LightInfluence = 0
moteEm.Color = ColorSequence.new(Color3.fromRGB(230, 255, 255))
moteEm.Parent = motes

-- Névoa de cloro baixa, densa e esbranquiçada junto ao piso/água.
local chlorine = mk(root, Vector3.new(120, 2.2, 120), CFrame.new(BASE + Vector3.new(0, 0.7, 0)), C_TILE, Enum.Material.SmoothPlastic,
{ Transparency = 1, CanCollide = false, CanQuery = false, CanTouch = false })
local chlorineAtt = inst("Attachment")
chlorineAtt.Parent = chlorine
local chlorineEm = inst("ParticleEmitter")
chlorineEm.Rate = 9
chlorineEm.Lifetime = NumberRange.new(5, 9)
chlorineEm.Speed = NumberRange.new(0.08, 0.35)
chlorineEm.SpreadAngle = Vector2.new(180, 180)
chlorineEm.Size = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1.4), NumberSequenceKeypoint.new(0.7, 3.5), NumberSequenceKeypoint.new(1, 4.2) })
chlorineEm.Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(0.12, 0.82), NumberSequenceKeypoint.new(0.75, 0.88), NumberSequenceKeypoint.new(1, 1) })
chlorineEm.Color = ColorSequence.new(Color3.fromRGB(224, 244, 239))
chlorineEm.LightEmission = 0.35
chlorineEm.Parent = chlorineAtt

-- Sons ambiente
local function loopSound(id, vol, pitch)
local s = inst("Sound")
s.SoundId = id
s.Volume = vol
s.PlaybackSpeed = pitch
s.Looped = true
s.Parent = sfxFolder
s:Play()
return s
end
loopSound(S.hum, 0.05, 0.6)


---

-- Cleanup

local function cleanup()
if not running then return end
running = false
pcall(function() RunService:UnbindFromRenderStep("PoolroomsCam") end)
pcall(function() RunService:UnbindFromRenderStep("PoolroomsNoJump") end)
pcall(function() ContextActionService:UnbindAction("PoolroomsNoJump") end)
for _, c in ipairs(conns) do pcall(function() c:Disconnect() end) end
pcall(function() gui:Destroy() end)
pcall(function() inventoryGui:Destroy() end)
pcall(function() lanternTool:Destroy() end)
pcall(function() StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.Backpack, savedBackpackEnabled) end)
pcall(function() root:Destroy() end)
for i = #scareRegistry, 1, -1 do scareRegistry[i] = nil end
pcall(function() sfxFolder:Destroy() end)
for _, e in ipairs(effects) do pcall(function() e:Destroy() end) end
for k, v in pairs(saved.lighting) do pcall(function() Lighting[k] = v end) end
for _, d in ipairs(saved.disabled) do pcall(function() d[1].Enabled = d[2] end) end
for _, o in ipairs(saved.removed) do pcall(function() o.Parent = Lighting end) end
pcall(function() SoundService.AmbientReverb = Enum.ReverbType.NoReverb end)
pcall(function()
player.CameraMode = savedCam.mode or Enum.CameraMode.Classic
player.CameraMinZoomDistance = savedCam.min or 0.5
player.CameraMaxZoomDistance = savedCam.max or 128
end)
pcall(function() camera.FieldOfView = savedCam.fov or 70 end)
local ch, hum, rp = getChar()
if hum then
pcall(function()
hum.CameraOffset = Vector3.zero
hum.WalkSpeed = savedWalk or 16
end)
-- devolve o pulo e os estados bloqueados
pcall(function()
if savedHum.use ~= nil then hum.UseJumpPower = savedHum.use end
hum.JumpPower = savedHum.power or 50
hum.JumpHeight = savedHum.height or 7.2
end)
for _, s in ipairs(LOCKED_STATES) do
pcall(function() hum:SetStateEnabled(s, true) end)
end
end
if ch and savedCF then pcall(function() ch:PivotTo(savedCF) end) end
env.__POOLROOMS_STOP = nil
end
env.__POOLROOMS_STOP = cleanup

-- Rotaciona os nomes dos objetos gerados, sem renomear classes Roblox ou personagens.
-- Isso é apenas higiene de estrutura; a proteção real fica no servidor.
local function rotateGeneratedNames()
if not running then return end
root.Name = obfName()
for _, obj in ipairs(root:GetDescendants()) do
obj.Name = obfName()
end
end

task.spawn(function()
while running do
task.wait(300)
rotateGeneratedNames()
end
end)


---

-- Início

local char0, hum0, root0 = getChar()
if not root0 then
player.CharacterAdded:Wait()
char0, hum0, root0 = getChar()
task.wait(0.5)
char0, hum0, root0 = getChar()
end
savedCF = root0 and root0.CFrame
savedWalk = hum0 and hum0.WalkSpeed
if hum0 then
savedHum.power, savedHum.height, savedHum.use = hum0.JumpPower, hum0.JumpHeight, hum0.UseJumpPower
end
pcall(function()
savedCam.mode = player.CameraMode
savedCam.min = player.CameraMinZoomDistance
savedCam.max = player.CameraMaxZoomDistance
savedCam.fov = camera.FieldOfView
end)

local cover, title = buildGui()
createFlashlightInventory()
applyLighting()
applyCam()
hookBodyVisibility(char0)
muteDefaultFootsteps(char0)
installACWatchers(char0)

for dx = -1, 1 do
for dz = -1, 1 do genChunk(dx, dz) end
end
tpToStart()
refreshQueue(0, 0)

task.delay(2.2, function()
if not running then return end
TweenService:Create(cover, TweenInfo.new(2.2), { BackgroundTransparency = 1 }):Play()
TweenService:Create(title, TweenInfo.new(2.2), { TextTransparency = 1 }):Play()
Debris:AddItem(cover, 3)
end)


---

-- Entrada

table.insert(conns, UserInputService.InputBegan:Connect(function(i, gpe)
if gpe then return end
if i.KeyCode == Enum.KeyCode.F then
if lanternTool then lanternTool:Activate() end
elseif i.KeyCode == Enum.KeyCode.LeftShift then
shiftHeld = true
refreshRun()
elseif i.KeyCode == Enum.KeyCode.R then
runOn = not runOn
refreshRun()
elseif i.KeyCode == CFG.ExitKey then
cleanup()
end
end))
table.insert(conns, UserInputService.InputEnded:Connect(function(i)
if i.KeyCode == Enum.KeyCode.LeftShift then
shiftHeld = false
refreshRun()
end
end))

-- PULO DESATIVADO: o espaço é consumido antes do controle padrão e o Jump é zerado a cada frame.
ContextActionService:BindActionAtPriority("PoolroomsNoJump", function()
return Enum.ContextActionResult.Sink
end, false, Enum.ContextActionPriority.High.Value + 100, Enum.KeyCode.Space, Enum.KeyCode.ButtonA)
RunService:BindToRenderStep("PoolroomsNoJump", Enum.RenderPriority.Input.Value + 1, function()
if not running then return end
local _, h = getChar()
if h and h.Jump then h.Jump = false end
end)

local function hookHum(ch)
local hum = ch:WaitForChild("Humanoid", 5)
if not hum then return end
table.insert(conns, hum.StateChanged:Connect(function(_, new)
local _, h, rp = getChar()
if not rp or not h or not running then return end
if new == Enum.HumanoidStateType.Landed then landDip = math.clamp(-lastVy * 0.006, 0.04, 0.22) end
if feetInWater(h, rp) then
local p = Vector3.new(rp.Position.X, BY - 0.47, rp.Position.Z)
if new == Enum.HumanoidStateType.Landed then
wetSfx(0.7, 1.0)
splashFx(p, 14)
ripple(p, 9, 1.6)
end
end
end))
end
if char0 then task.spawn(hookHum, char0) end
table.insert(conns, player.CharacterAdded:Connect(function(ch)
if not running then return end
ch:WaitForChild("HumanoidRootPart", 5)
task.wait(0.3)
tpToStart()
applyCam()
hookBodyVisibility(ch)
muteDefaultFootsteps(ch)
installACWatchers(ch)
hookHum(ch)
end))


---

-- Loop principal

local rngFx = Random.new()
local lastCX, lastCZ
local stepDist, waterAmt, fovNow = 0, 0, 70
local nextDrip, nextDistant, nextScare, shimmerT = 1, 8, 12, 0

local function randomWaterPos(pcx, pcz, rad)
local list = {}
for _, ch in ipairs(nearChunks(pcx, pcz, rad)) do
for _, w in ipairs(ch.water) do table.insert(list, w) end
end
if #list == 0 then return end
local w = list[rngFx:NextInteger(1, #list)]
local sp = w.part
return Vector3.new(
sp.Position.X + (rngFx:NextNumber() - 0.5) * math.max(0, sp.Size.X - 4),
BY - 0.47,
sp.Position.Z + (rngFx:NextNumber() - 0.5) * math.max(0, sp.Size.Z - 4)
)
end

local mainConn
local function mainStep(dt)
if not running then return end
local ch, hum, rp = getChar()
if not rp or not hum then return end
clientAntiCheatStep(dt)
local pos = rp.Position
local pcx, pcz = chunkOf(pos)

if pcx ~= lastCX or pcz ~= lastCZ then
lastCX, lastCZ = pcx, pcz
refreshQueue(pcx, pcz)
end
for _ = 1, CFG.PerFrame do
local q = table.remove(queue, 1)
if q then genChunk(q[1], q[2]) end
end

if pos.Y < BY - 12 then
tpToStart()
return
end

-- primeira pessoa
if player.CameraMode ~= Enum.CameraMode.LockFirstPerson then applyCam() end

-- velocidade / água
local inW = feetInWater(hum, rp)
local speed = sprinting and CFG.SprintSpeed or CFG.WalkSpeed
if inW then speed *= 0.8 end
acState.expected.WalkSpeed = speed
hum.WalkSpeed = speed

local vel = rp.AssemblyLinearVelocity
local hs = Vector3.new(vel.X, 0, vel.Z).Magnitude
local moving = hum.MoveDirection.Magnitude > 0.1 and hum.FloorMaterial ~= Enum.Material.Air

-- silencia passos padrão (usamos somente o SFX abaixo)
for _, obj in ipairs(ch:GetDescendants()) do
if obj:IsA("Sound") then
local n = string.lower(obj.Name)
if string.find(n, "running", 1, true) or string.find(n, "foot", 1, true) or string.find(n, "step", 1, true) then obj.Volume = 0 end
end
end

-- passos
if moving then
stepDist += hs * dt
local stride = 5.7
if stepDist >= stride and os.clock() >= nextStepSfx then
stepDist = 0
nextStepSfx = os.clock() + 0.24
if inW then
wetStep()
local p = Vector3.new(pos.X, BY - 0.47, pos.Z)
ripple(p, 4.5, 1.2)
splashFx(p, 3)
else
playUI(S.step, 0.4, rngFx:NextNumber(0.85, 1.05))
end
end
bobT += dt * hs * 0.55
else
stepDist = 0
end

-- estado p/ câmera
camState.moving = moving
camState.sprint = sprinting and moving
lastVy = vel.Y

-- FOV (corrida)
local fovT = (sprinting and moving) and 75 or 70
fovNow += (fovT - fovNow) * math.min(1, dt * 6)
camera.FieldOfView = fovNow

-- tom de água
waterAmt += ((inW and 1 or 0) - waterAmt) * math.min(1, dt * 4)
if applyLighting_cc then
applyLighting_cc.TintColor = Color3.fromRGB(190, 215, 220):Lerp(Color3.fromRGB(130, 200, 215), waterAmt)
end

-- poeira segue a câmera
motes.CFrame = CFrame.new(camera.CFrame.Position + Vector3.new(0, 2, 0))

-- pingos / respingos distantes
nextDrip -= dt
if nextDrip <= 0 then
nextDrip = rngFx:NextNumber(1.2, 4)
local p = randomWaterPos(pcx, pcz, 2)
if p then
if CFG.WaterSounds then play3D(S.splash, p, 0.28, rngFx:NextNumber(1.8, 2.4), 70) end
ripple(p, 3, 1.6)
end
end
nextDistant -= dt
if nextDistant <= 0 then
nextDistant = rngFx:NextNumber(9, 20)
local p = randomWaterPos(pcx, pcz, 3)
if p then
if CFG.WaterSounds then play3D(S.splash, p, 0.5, rngFx:NextNumber(0.55, 0.75), 170) end
ripple(p, 6, 2)
end
end

-- Diretor de sustos: aproximação, corrida breve e desaparecimento sem dano.
nextScare -= dt
local t = os.clock()
if CFG.Scares and nextScare <= 0 then
nextScare = rngFx:NextNumber(8, 18)
end
for _, sc in ipairs(scareRegistry) do
if sc.parent and sc.parent.Parent and sc.cooldown <= t then
local current = sc.model.PrimaryPart
local d = current and (Vector3.new(pos.X, current.Position.Y, pos.Z) - current.Position).Magnitude or 999
if not sc.active and d < 62 and (rngFx:NextNumber() < CFG.ScareTriggerChance or d < 18) then
sc.active = true
sc.mode = (rngFx:NextNumber() < 0.42) and "runner" or "static"
sc.model:PivotTo(CFrame.lookAt(sc.home, sc.home + Vector3.new(1, 0, 0)))
scareVisible(sc, true)
play3D(S.swim, sc.home, 0.32, rngFx:NextNumber(0.82, 1.05), 90)
ripple(sc.home + Vector3.new(0, 1.7, 0), 5, 1.1)
end
if sc.active then
if sc.mode == "runner" then
sc.moveClock -= dt
if sc.moveClock <= 0 then
sc.moveClock = 0.28
-- Movimento real do Humanoid; não há TweenService nem teleporte por frame.
sc.model.ScareHumanoid:MoveTo(Vector3.new(pos.X, current.Position.Y, pos.Z))
end
if d < 8 then
scareVisible(sc, false)
sc.active, sc.cooldown = false, t + 45
play3D(S.click, pos, 0.22, 0.55, 50)
end
else
if d < 10 then
scareVisible(sc, false)
sc.active, sc.cooldown = false, t + 55
end
end
end
end
end
end

-- animações dos chunks

shimmerT -= dt
local doShimmer = shimmerT <= 0
if doShimmer then shimmerT = 0.12 end

for _, c in ipairs(nearChunks(pcx, pcz, 2)) do
if doShimmer then
for _, w in ipairs(c.water) do
local a = (math.sin(t * 0.8 + w.ph) + 1) / 2
w.part.Color = w.c1:Lerp(w.c2, a)
w.light.Brightness = 0.05 + 0.025 * math.sin(t * 1.7 + w.ph)
end
for _, ca in ipairs(c.caustics) do
local k = (math.sin(t * 1.8 + ca.ph) + 1) / 2
ca.part.Transparency = 0.76 + k * 0.16
ca.part.Color = Color3.fromRGB(65 + math.floor(k * 35), 170 + math.floor(k * 45), 195 + math.floor(k * 45))
end
for _, sn in ipairs(c.suns) do
local k = math.sin(t * 0.7 + sn.ph)
sn.light.Brightness = 0.35 + 0.1 * k
for _, pn in ipairs(sn.panes) do pn.Transparency = 0.52 + 0.07 * k end
end
end
for _, f in ipairs(c.floaters) do
local s1 = math.sin(t * 1.3 + f.ph)
f.part.CFrame = f.base * CFrame.new(0, s1 * 0.04, 0)
* CFrame.Angles(math.sin(t * 0.9 + f.ph) * 0.03, math.sin(t * 0.2 + f.ph) * 0.4, s1 * 0.03)
end
for _, fl in ipairs(c.flick) do
if t >= fl.next then
fl.on = not fl.on
if fl.on then
fl.next = t + rngFx:NextNumber(2, 8)
else
fl.next = t + rngFx:NextNumber(0.04, 0.25)
end
fl.panel.Material = fl.on and Enum.Material.Neon or Enum.Material.SmoothPlastic
fl.light.Enabled = fl.on
if (fl.panel.Position - pos).Magnitude < 50 then
play3D(S.click, fl.panel.Position, 0.12, rngFx:NextNumber(1.4, 2), 50)
end
end
end
end
end

local function connectMain()
mainConn = RunService.Heartbeat:Connect(mainStep)
table.insert(conns, mainConn)
end
connectMain()

-- Vigia: se o loop principal (que contém o anticheat) for desconectado, flaga e reconecta.
task.spawn(function()
while running do
task.wait(1)
if running and AC.Enabled and mainConn and not mainConn.Connected then
acFlag("ac_tamper", "loop desconectado")
connectMain()
end
end
end)

-- Câmera suave (balanço leve, ajustável em CFG.CameraBob)
RunService:BindToRenderStep("PoolroomsCam", Enum.RenderPriority.Camera.Value + 1, function(dt)
if not running then return end
local _, hum = getChar()
local t = os.clock()
local k = CFG.CameraBob
camAmp += ((camState.moving and 1 or 0) - camAmp) * math.min(1, dt * 7)
local sp = camState.sprint and 1.15 or 1
local strafe = 0
if hum then strafe = hum.MoveDirection:Dot(camera.CFrame.RightVector) end
camRoll += (-strafe * 0.02 * camAmp * k - camRoll) * math.min(1, dt * 6)
landDip += (0 - landDip) * math.min(1, dt * 6)
local breathe = math.sin(t * 1.15)
local x = math.sin(bobT) * 0.05 * camAmp * sp * k
local y = -math.cos(bobT * 2) * 0.05 * camAmp * sp * k + breathe * 0.006 * k - landDip * k * 2
local pitch = math.cos(bobT * 2) * 0.006 * camAmp * sp * k + landDip * 0.04 * k
local roll = math.sin(bobT) * 0.013 * camAmp * sp * k + camRoll
camera.CFrame = camera.CFrame * CFrame.new(x, y, 0) * CFrame.Angles(pitch, 0, roll)
if blurFx then blurFx.Size = (camState.sprint and 1 or 0) + landDip * 3 end
end)

print("[Poolrooms] Ativo. RUN = botão / R / SHIFT | P = sair | pulo desativado")
/ R / SHIFT | P = sair | pulo desativado")

```
