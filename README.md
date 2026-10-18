-- ModoBatataFPS.client.lua (versão anti-trava)
-- LocalScript em StarterPlayer > StarterPlayerScripts
-- Modo batata + céu + HUD + limitador de efeitos de frutas/espadas

local Lighting = game:GetService("Lighting")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")
local Stats = game:GetService("Stats")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

--------------------------------------------------
-- CONFIG
--------------------------------------------------
local SKY_ID = "rbxassetid://82868551497078"
local TIKTOK = "zinc.infinitx"
local TEMPO_AVISO = 5
local REMOVER_TEXTURAS = true
local DESLIGAR_PARTICULAS = true
local DESLIGAR_LUZES = true
local OCULTAR_OUTROS_JOGADORES = false
local REMOVER_ACESSORIOS_OUTROS = true
local RAM_MAX_MB = 4096
local FRAME_BUDGET_MS = 1000 / 60
local LIMITADOR_ATIVO = true -- false = desliga o limitador de efeitos (use para testar se o giro some)
local APAGAR_EFEITOS = false -- false = só esconde os efeitos, nunca apaga (mais seguro para o personagem)

--------------------------------------------------
-- LISTA DE FRUTAS E ESPADAS (efeitos reduzidos ao máximo)
-- Valor = máximo de partes de efeito por segundo.
-- Menor número = menos efeito = menos lag.
-- Control e Rumble estão no mínimo porque são as que mais travam.
--------------------------------------------------
local LIMITE_PADRAO = 40

local LIMITES = {
	-- Frutas
	["Control"] = 4,
	["Rumble"] = 4,
	["Magma"] = 15,
	["Quake"] = 15,
	["Leopard"] = 15,
	["Spirit"] = 15,
	["T-Rex"] = 15,
	["Gas"] = 15,
	["Mammoth"] = 15,
	["Portal"] = 15,
	["Flame"] = 15,
	["Blizzard"] = 15,
	["Shadow"] = 15,
	["Phoenix"] = 15,
	["Gravity"] = 15,

	-- Espadas
	["Cursed Dual Katana"] = 15,
	["CDK"] = 15,
	["True Triple Katana"] = 15,
	["TTK"] = 15,
	["Dark Blade"] = 15,
	["Yoru"] = 15,
	["Shark Anchor"] = 15,
	["Dragon Trident"] = 15,
	["Hallow Scythe"] = 15,
	["Dark Dagger"] = 15,
	["Tushita"] = 15,
	["Yama"] = 15,
	["Rengoku"] = 15,
	["Saber"] = 15,
	["Midnight Blade"] = 15,
	["Canvander"] = 15,
	["Buddy Sword"] = 15,
	["Spikey Trident"] = 15,
	["Fox Lamp"] = 15,
	["Twin Hooks"] = 15,
	["Gravity Cane"] = 15,
	["Pole"] = 15,
	["Shisui"] = 15,
}

-- Pastas do Workspace onde os efeitos costumam aparecer
local PASTAS_EFEITO = {
	["Effects"] = true,
	["_WorldOrigin"] = true,
	["FX"] = true,
	["Debris"] = true,
}

--------------------------------------------------
-- 1) MODO BATATA: ILUMINAÇÃO
--------------------------------------------------
Lighting.GlobalShadows = false
Lighting.Brightness = 2
Lighting.Ambient = Color3.fromRGB(170, 170, 170)
Lighting.OutdoorAmbient = Color3.fromRGB(170, 170, 170)
Lighting.EnvironmentDiffuseScale = 0
Lighting.EnvironmentSpecularScale = 0
Lighting.ShadowSoftness = 0
Lighting.FogEnd = 1e6
Lighting.FogStart = 1e6

pcall(function()
	Lighting.Technology = Enum.Technology.Compatibility
end)

local function limparLighting(obj)
	if obj:IsA("PostEffect") then
		obj.Enabled = false
	elseif obj:IsA("Atmosphere") then
		obj:Destroy()
	end
end

for _, v in ipairs(Lighting:GetChildren()) do
	limparLighting(v)
end
Lighting.ChildAdded:Connect(limparLighting)

--------------------------------------------------
-- 2) CÉU
--------------------------------------------------
local sky = Lighting:FindFirstChildOfClass("Sky")
if not sky then
	sky = Instance.new("Sky")
	sky.Parent = Lighting
end
for _, face in ipairs({"SkyboxBk", "SkyboxDn", "SkyboxFt", "SkyboxLf", "SkyboxRt", "SkyboxUp"}) do
	sky[face] = SKY_ID
end
sky.StarCount = 0
sky.CelestialBodiesShown = false

--------------------------------------------------
-- 3) TERRENO / ÁGUA
--------------------------------------------------
pcall(function()
	local t = Workspace.Terrain
	t.Decoration = false
	t.WaterWaveSize = 0
	t.WaterWaveSpeed = 0
	t.WaterReflectance = 0
	t.WaterTransparency = 1
end)

--------------------------------------------------
-- 4) LIMITADOR DE EFEITOS (frutas e espadas)
--------------------------------------------------
-- Descobre qual item da lista alguém está segurando
local itensLower = {}
for nome, limite in pairs(LIMITES) do
	itensLower[string.lower(nome)] = limite
end

local function limiteDoTool(nomeTool)
	local n = string.lower(nomeTool)
	local melhor = nil
	for item, limite in pairs(itensLower) do
		if n == item or string.sub(n, 1, #item) == item then
			if not melhor or limite < melhor then
				melhor = limite
			end
		end
	end
	return melhor
end

-- limite atual de cada jogador (nil = nenhum item da lista equipado)
local limitePorJogador = {}

local function observarPersonagem(p, char)
	local function checar()
		local menor = nil
		for _, v in ipairs(char:GetChildren()) do
			if v:IsA("Tool") then
				local l = limiteDoTool(v.Name)
				if l and (not menor or l < menor) then
					menor = l
				end
			end
		end
		limitePorJogador[p] = menor
	end
	checar()
	char.ChildAdded:Connect(checar)
	char.ChildRemoved:Connect(checar)
end

local function observarJogador(p)
	if p.Character then
		observarPersonagem(p, p.Character)
	end
	p.CharacterAdded:Connect(function(c)
		observarPersonagem(p, c)
	end)
end

for _, p in ipairs(Players:GetPlayers()) do
	observarJogador(p)
end
Players.PlayerAdded:Connect(observarJogador)
Players.PlayerRemoving:Connect(function(p)
	limitePorJogador[p] = nil
end)

-- Modo anti-freeze: enquanto estiver ativo, nenhum efeito novo passa
local emergenciaAte = 0

local function limiteAtual()
	if os.clock() < emergenciaAte then
		return 0
	end
	local menor = LIMITE_PADRAO
	for _, l in pairs(limitePorJogador) do
		if l < menor then
			menor = l
		end
	end
	return menor
end

local function emPersonagem(obj)
	local m = obj:FindFirstAncestorOfClass("Model")
	return m ~= nil and m:FindFirstChildOfClass("Humanoid") ~= nil
end

local function emPastaDeEfeito(obj)
	local pai = obj.Parent
	if pai == Workspace then
		return true
	end
	while pai and pai ~= Workspace do
		if PASTAS_EFEITO[pai.Name] then
			return true
		end
		pai = pai.Parent
	end
	return false
end

-- Partes que o jogo usa para mover o personagem (solda, BodyMover, etc.)
-- nunca devem ser tocadas, senão o golpe pode bugar (ex.: ficar girando).
local function temFisica(obj)
	if obj.Parent and (obj.Parent:IsA("Tool") or obj.Parent:IsA("Accessory")) then
		return true
	end
	for _, v in ipairs(obj:GetChildren()) do
		if v:IsA("BodyMover") or v:IsA("Constraint") or v:IsA("JointInstance")
			or v:IsA("WeldConstraint") or v:IsA("AlignOrientation") or v:IsA("AlignPosition") then
			return true
		end
	end
	return false
end

local contador, janela = 0, os.clock()

local function limitarEfeito(obj)
	if not obj:IsA("BasePart") or obj:IsA("Terrain") then
		return
	end
	if not LIMITADOR_ATIVO or obj.CanCollide or emPersonagem(obj) or not emPastaDeEfeito(obj) or temFisica(obj) then
		return
	end

	local agora = os.clock()
	if agora - janela >= 1 then
		janela = agora
		contador = 0
	end
	contador += 1

	-- invisível sempre; acima do limite, remove de vez
	obj.Transparency = 1
	obj.CastShadow = false
	if APAGAR_EFEITOS and contador > limiteAtual() then
		task.defer(function()
			pcall(function()
				obj:Destroy()
			end)
		end)
	end
end

--------------------------------------------------
-- 5) WORKSPACE
--------------------------------------------------
local function otimizar(obj)
	if obj:IsA("BasePart") then
		obj.CastShadow = false
		obj.Reflectance = 0
		if not obj:IsA("Terrain") then
			obj.Material = Enum.Material.SmoothPlastic
		end
		if obj:IsA("MeshPart") then
			obj.RenderFidelity = Enum.RenderFidelity.Performance
			obj.TextureID = ""
		end
	elseif DESLIGAR_PARTICULAS and obj:IsA("ParticleEmitter") then
		obj.Enabled = false
		obj.Rate = 0
		obj.Lifetime = NumberRange.new(0)
	elseif DESLIGAR_PARTICULAS and (
		obj:IsA("Trail") or obj:IsA("Beam")
		or obj:IsA("Smoke") or obj:IsA("Fire") or obj:IsA("Sparkles")
	) then
		obj.Enabled = false
	elseif DESLIGAR_LUZES and obj:IsA("Light") then
		obj.Enabled = false
	elseif REMOVER_TEXTURAS and (obj:IsA("Decal") or obj:IsA("Texture")) then
		obj:Destroy()
	elseif obj:IsA("SurfaceAppearance") or obj:IsA("Clothing") or obj:IsA("ShirtGraphic") then
		obj:Destroy()
	elseif obj:IsA("Highlight") then
		obj.Enabled = false
	elseif obj:IsA("Explosion") then
		obj.Visible = false
		obj.BlastPressure = 0
	elseif obj:IsA("ForceField") then
		obj.Visible = false
	end
end

task.spawn(function()
	local n = 0
	for _, obj in ipairs(Workspace:GetDescendants()) do
		otimizar(obj)
		n += 1
		if n % 300 == 0 then
			task.wait()
		end
	end
end)

-- ANTI-FREEZE
-- Em vez de processar tudo no mesmo frame (o que causa a "congelada" quando
-- Control/Rumble soltam centenas de partes), os objetos entram numa fila e
-- poucos são processados por frame, dentro de um orçamento de tempo.
local fila, inicio, fim = {}, 1, 0
local MAX_FILA = 3000
local ORCAMENTO_MS = 1.5

Workspace.DescendantAdded:Connect(function(obj)
	if fim - inicio >= MAX_FILA then
		-- fila cheia (explosão de efeitos): corta na hora
		if APAGAR_EFEITOS and obj:IsA("BasePart") and not obj:IsA("Terrain") and not obj.CanCollide
			and not emPersonagem(obj) and emPastaDeEfeito(obj) then
			task.defer(function()
				pcall(function()
					obj:Destroy()
				end)
			end)
		end
		return
	end
	fim += 1
	fila[fim] = obj
end)

local mediaDt = 1 / 60
RunService.Heartbeat:Connect(function(dt)
	-- Detector de congelada: frame muito longo ou FPS médio baixo
	-- ativa o modo emergência por alguns segundos.
	mediaDt = mediaDt * 0.9 + dt * 0.1
	if dt > 0.12 then
		emergenciaAte = os.clock() + 3
	elseif mediaDt > 1 / 20 then
		emergenciaAte = os.clock() + 2
	end

	local t0 = os.clock()
	while inicio <= fim do
		local obj = fila[inicio]
		fila[inicio] = nil
		inicio += 1
		if obj and obj.Parent then
			pcall(otimizar, obj)
			pcall(limitarEfeito, obj)
		end
		if (os.clock() - t0) * 1000 > ORCAMENTO_MS then
			break
		end
	end
	if inicio > fim then
		inicio, fim = 1, 0
	end
end)

pcall(function()
	settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
end)

pcall(function()
	local ugs = UserSettings():GetService("UserGameSettings")
	ugs.SavedQualityLevel = Enum.SavedQualitySetting.QualityLevel1
end)

--------------------------------------------------
-- 6) INTERFACE (HUD + AVISO TIKTOK)
--------------------------------------------------
local gui = Instance.new("ScreenGui")
gui.Name = "InterfaceBatata"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.DisplayOrder = 999
gui.Parent = playerGui

local hud = Instance.new("Frame")
hud.AnchorPoint = Vector2.new(0.5, 0)
hud.Position = UDim2.new(0.5, 0, 0, 4)
hud.Size = UDim2.fromOffset(250, 34)
hud.BackgroundColor3 = Color3.new(0, 0, 0)
hud.BackgroundTransparency = 0.15
hud.BorderSizePixel = 0
hud.Parent = gui

local hudCorner = Instance.new("UICorner")
hudCorner.CornerRadius = UDim.new(0, 8)
hudCorner.Parent = hud

local layout = Instance.new("UIListLayout")
layout.FillDirection = Enum.FillDirection.Horizontal
layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
layout.VerticalAlignment = Enum.VerticalAlignment.Center
layout.Padding = UDim.new(0, 2)
layout.Parent = hud

local function criarStat(titulo)
	local f = Instance.new("Frame")
	f.BackgroundTransparency = 1
	f.Size = UDim2.new(0, 46, 1, 0)
	f.Parent = hud

	local t = Instance.new("TextLabel")
	t.BackgroundTransparency = 1
	t.Position = UDim2.fromOffset(0, 3)
	t.Size = UDim2.new(1, 0, 0, 10)
	t.Font = Enum.Font.Gotham
	t.TextSize = 8
	t.TextColor3 = Color3.fromRGB(150, 150, 160)
	t.Text = titulo
	t.Parent = f

	local v = Instance.new("TextLabel")
	v.BackgroundTransparency = 1
	v.Position = UDim2.fromOffset(0, 13)
	v.Size = UDim2.new(1, 0, 0, 18)
	v.Font = Enum.Font.GothamMedium
	v.TextSize = 15
	v.TextColor3 = Color3.new(1, 1, 1)
	v.Text = "--"
	v.Parent = f
	return v
end

local fpsL = criarStat("FPS")
local cpuL = criarStat("CPU %")
local gpuL = criarStat("GPU %")
local ramL = criarStat("RAM %")
local pingL = criarStat("PING")

local function pct(ms)
	return math.clamp(math.floor(ms / FRAME_BUDGET_MS * 100 + 0.5), 0, 100)
end

-- Heartbeat é mais leve que RenderStepped para um HUD
local frames, tempo = 0, 0
RunService.Heartbeat:Connect(function(dt)
	frames += 1
	tempo += dt
	if tempo < 1 then
		return
	end

	fpsL.Text = tostring(math.floor(frames / tempo + 0.5))
	frames, tempo = 0, 0

	pcall(function()
		local ps = Stats.PerformanceStats
		cpuL.Text = tostring(pct(ps.CPU:GetValue()))
		gpuL.Text = tostring(pct(ps.GPU:GetValue()))
	end)

	pcall(function()
		local ping = Stats.Network.ServerStatsItem["Data Ping"]:GetValue()
		pingL.Text = tostring(math.floor(ping + 0.5))
	end)

	pcall(function()
		ramL.Text = tostring(math.clamp(math.floor(Stats:GetTotalMemoryUsageMb() / RAM_MAX_MB * 100), 0, 100))
	end)
end)

local aviso = Instance.new("Frame")
aviso.Size = UDim2.fromOffset(170, 40)
aviso.AnchorPoint = Vector2.new(0.5, 1)
aviso.Position = UDim2.new(0.5, 0, 1, -10)
aviso.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
aviso.BackgroundTransparency = 0.2
aviso.BorderSizePixel = 0
aviso.Parent = gui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 8)
corner.Parent = aviso

local titulo = Instance.new("TextLabel")
titulo.Size = UDim2.new(1, 0, 0, 14)
titulo.Position = UDim2.fromOffset(0, 3)
titulo.BackgroundTransparency = 1
titulo.Font = Enum.Font.GothamBold
titulo.TextSize = 11
titulo.TextColor3 = Color3.fromRGB(255, 255, 255)
titulo.Text = "TikTok (toque no nome)"
titulo.Parent = aviso

local nome = Instance.new("TextBox")
nome.Size = UDim2.new(1, -10, 0, 18)
nome.Position = UDim2.fromOffset(5, 18)
nome.BackgroundTransparency = 1
nome.Font = Enum.Font.GothamBold
nome.TextSize = 14
nome.TextColor3 = Color3.fromRGB(0, 220, 255)
nome.Text = TIKTOK
nome.TextEditable = false
nome.ClearTextOnFocus = false
nome.Parent = aviso

local focado = false
nome.Focused:Connect(function()
	focado = true
	nome.CursorPosition = #TIKTOK + 1
	nome.SelectionStart = 1
end)
nome.FocusLost:Connect(function()
	focado = false
	task.delay(2, function()
		if not focado and aviso.Parent then
			aviso:Destroy()
		end
	end)
end)

task.delay(TEMPO_AVISO, function()
	if focado or not aviso.Parent then
		return
	end
	local tween = TweenService:Create(aviso, TweenInfo.new(0.4), {BackgroundTransparency = 1})
	tween:Play()
	titulo.Visible = false
	nome.Visible = false
	tween.Completed:Wait()
	if not focado and aviso.Parent then
		aviso:Destroy()
	end
end)

--------------------------------------------------
-- 7) EXTRA: MENOS COISA PARA DESENHAR
--------------------------------------------------
if REMOVER_ACESSORIOS_OUTROS then
	local function limparPersonagem(char)
		for _, v in ipairs(char:GetChildren()) do
			if v:IsA("Accessory") then
				v:Destroy()
			end
		end
		char.ChildAdded:Connect(function(v)
			if v:IsA("Accessory") then
				task.defer(function()
					v:Destroy()
				end)
			end
		end)
	end

	local function aoEntrar(p)
		if p == player then
			return
		end
		if p.Character then
			limparPersonagem(p.Character)
		end
		p.CharacterAdded:Connect(limparPersonagem)
	end

	for _, p in ipairs(Players:GetPlayers()) do
		aoEntrar(p)
	end
	Players.PlayerAdded:Connect(aoEntrar)
end

if OCULTAR_OUTROS_JOGADORES then
	local function esconder(char)
		for _, v in ipairs(char:GetDescendants()) do
			if v:IsA("BasePart") then
				v.LocalTransparencyModifier = 1
			end
		end
		char.DescendantAdded:Connect(function(v)
			if v:IsA("BasePart") then
				v.LocalTransparencyModifier = 1
			end
		end)
	end

	local function aoEntrarOculto(p)
		if p == player then
			return
		end
		if p.Character then
			esconder(p.Character)
		end
		p.CharacterAdded:Connect(esconder)
	end

	for _, p in ipairs(Players:GetPlayers()) do
		aoEntrarOculto(p)
	end
	Players.PlayerAdded:Connect(aoEntrarOculto)
end

print("[ModoBatataFPS] ativo")
