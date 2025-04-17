local defaults = {
	ESP = {
		["Money BagESP"] = false,
		["MonsterESP"] = false,
		["Dead MonsterESP"] = false,
		["AnimalESP"] = false,
		["Dead AnimalESP"] = false,
		["ItemESP"] = false,
		["Vault CodeESP"] = false,
		["Train ESP"] = false
	},
	ExtraPP = 1,
	AutoCollectBags = false,
	AutoPickTools = false,
	AutoPickOther = false,
	AutoPickArmor = false,
	AutoPickBonds = false,
	Noclip = false,
	NC = false,
	ShowTime = false,
	ShowDistance = false,
	ShowSpeed = false,
	ShowFuel = false,
	II = false,
	GKA = false,
	MA = false,
	ARG = false,
	Raycast = false,
	Aimbot = false,
	Mode = "Distance",
	NoVoid = false,
	SaveBulltets = false,

	BandageUse = 0,
	OilUse = 0,
	OilUseCooldown = 5,
	KAR = 500,

	ThrowPower = 100
}

local vals = table.clone(defaults)
vals.ESP = table.clone(defaults.ESP)

local function getGlobalTable()
	return typeof(getfenv().getgenv) == "function" and typeof(getfenv().getgenv()) == "table" and getfenv().getgenv() or _G
end

getGlobalTable().FireHubLoaded = true

local WindUI = loadstring(game:HttpGet("https://tree-hub.vercel.app/api/UI/WindUI"))()
local espLib = loadstring(game:HttpGet("https://raw.githubusercontent.com/MITUMAxDev/Glitch/refs/heads/main/loads/libs/esp.lua", true))()

local plr = game:GetService("Players").LocalPlayer

if game.PlaceId == 116495829188952 then
	warn("PLEASE ONLY EXECUTE IN GAME")
	return
end

espLib.Values = vals.ESP

local function rs(times)
	local times = math.max(math.round(tonumber(times) or 1), 1)
	local dt = 0
	for i=1, times do
		dt = dt + game:GetService("RunService").RenderStepped:Wait()
	end
	return dt / times
end
local espFunc = espLib.ApplyESP

local closed = false
local cons = {}
local prompts = {}
local oprompts = {}
local hooks = {}

local cd = {}
local fppn = false
local fpp = getfenv().fireproximityprompt

local probablyDead = {}
local deathAmmo = {}

local function isDead(hum)
	if probablyDead[hum] then
		return true
	end

	if hum and hum.Parent then
		if not hum:IsA("Humanoid") then
			hum = hum:FindFirstChild("Humanoid")
		end

		if hum then
			if probablyDead[hum] then
				return true
			end

			local dead = hum.Health <= 0.01 and hum.PlatformStand
			if dead then
				probablyDead[hum] = true
			end

			return dead
		end
	end

	return true
end

local myGuns = {}
local melee = {}
local heals = {
	Bandage = {},
	["Snake Oil"] = {}
}

local function bp(v)
	if v and v:IsA("Tool") then
		if v:FindFirstChild("WeaponConfiguration") and not myGuns[v] then
			myGuns[v] = true
		elseif v:FindFirstChild("SwingEvent") and not melee[v] then
			melee[v] = true
		elseif heals[v.Name] and not heals[v.Name][v] then
			heals[v.Name][v] = true
		end
	end
end

local toolsMt = setmetatable({}, {
	__index = function(self, value)
		if value == "GetChildren" then
			local tools = plr.Backpack:GetChildren()

			if plr.Character then
				for i,v in plr.Character:GetChildren() do
					if v and v:IsA("Tool") then
						table.insert(tools, 1, v)
					end
				end
			end

			return tools
		end
		if plr and plr.Character and plr.Character:FindFirstChildOfClass("Tool") and plr.Character:FindFirstChildOfClass("Tool").Name == value then
			return plr.Character:FindFirstChildOfClass("Tool")
		end
		return plr.Backpack:FindFirstChild(value)
	end
})

for i,v in toolsMt.GetChildren do
	bp(v)
end
cons[#cons+1] = plr.Backpack.ChildAdded:Connect(bp)

local cooldown = {}
local function setCooldown(gun)
	cooldown[gun] = true
	task.wait((gun.WeaponConfiguration.FireDelay.Value * 1.5) + 0.25)
	cooldown[gun] = false
end

local function addFunction(t,v)
	if v == nil or typeof(t) ~= "table" then return end
	local i = 1
	while true do
		if v == nil or typeof(v) == "Instance" and v.Parent == nil then
			return -1
		end
		if t[i] == nil or typeof(t[i]) == "Instance" and t[i].Parent == nil then
			t[i] = v
			return i
		end
		i = i + 1
	end
end
local function add(t,v)
	task.spawn(addFunction, t, v)
end
local function remove(t,v)
	task.spawn(pcall, table.remove, t, table.find(t, v))
end
local function count(t)
	local amnt = 0
	for i,v in t do
		if typeof(v) == "Instance" and v.Parent ~= nil or typeof(v) ~= "Instance" and v ~= nil then
			amnt = amnt + 1
		end
	end
	return amnt
end
local function getFirst(t)
	for v,i in t do
		if typeof(v) == "Instance" and (v.Parent == plr.Character or v.Parent == plr.Backpack) or typeof(v) ~= "Instance" and v ~= nil then
			return v
		else
			remove(t, v)
		end
	end
end

local function fuseTables(t1, t2)
	for i,v in t2 do
		add(t1, v)
	end

	return t1
end

local function raycast(from, to, ignore)
	local raycastParams = RaycastParams.new()

	raycastParams.IgnoreWater = true
	raycastParams.FilterDescendantsInstances = fuseTables(plr.Character and plr.Character:GetDescendants() or {}, ignore or {})

	local result = workspace:Raycast(from, (to - from).Unit * (to - from).Magnitude, raycastParams)
	return result and result.Instance
end

local s = game:GetService("ReplicatedStorage").Remotes.Weapon.Shoot
local r = game:GetService("ReplicatedStorage").Remotes.Weapon.Reload
local function shoot(gun, target)
	if not isDead(target) and (vals.Raycast and not raycast(workspace.CurrentCamera.CFrame.Position, target:GetPivot().Position, target:GetDescendants()) or not vals.Raycast) and (workspace.CurrentCamera.CFrame.Position - target:GetPivot().Position).Magnitude <= vals.KAR then
		local head = target:FindFirstChild("Head") or target:GetPivot()

		local hits = {}
		for i=1, gun.WeaponConfiguration.PelletsPerBullet.Value do
			hits[tostring(i)] = target.Humanoid
		end

		if target.Humanoid.Health - gun.WeaponConfiguration.BulletDamage.Value < 0 and gun.ServerWeaponState.CurrentAmmo.Value >= 1 and not cooldown[gun] then
			deathAmmo[target.Humaoind] = (tonumber(deathAmmo[target.Humaoind]) or 3) - 1
			if deathAmmo[target.Humaoind] <= 0 then
				probablyDead[target.Humanoid] = true
			end
			task.spawn(setCooldown, gun)
		end

		s:FireServer(workspace:GetServerTimeNow(), gun, CFrame.lookAt(head.Position + (head.CFrame.LookVector * 10), head.Position), hits)
	end
end
local function reload(gun)
	r:FireServer(workspace:GetServerTimeNow(), gun)
end

if fpp then
	pcall(function()
		task.spawn(function()
			local pp = Instance.new("ProximityPrompt", workspace)
			local con; con = pp.Triggered:Connect(function()
				con:Disconnect()
				fppn = true
				task.wait(0.1)
				pp.Parent = nil
				task.wait(0.1)
				pp:Destroy()
			end)
			task.wait(0.1)
			fpp(pp)
			task.wait(1.5)
			if pp and pp.Parent then
				con:Disconnect()
				task.wait(0.1)
				pp.Parent = nil
				task.wait(0.1)
				pp:Destroy()
			end
		end)
	end)
end

local function fppFunc(pp)
	cd[pp] = true
	local a,b,c,d,e = pp.MaxActivationDistance, pp.Enabled, pp.Parent, pp.HoldDuration, pp.RequiresLineOfSight
	local obj = Instance.new("Part", workspace)
	obj.Transparency = 1
	obj.CanCollide = false
	obj.Size = Vector3.new(0.1, 0.1, 0.1)
	obj.Anchored = true
	pp.Parent = obj
	pp.MaxActivationDistance = math.huge
	pp.Enabled = true
	pp.HoldDuration = 0
	pp.RequiresLineOfSight = false
	if not pp or not pp.Parent then
		obj:Destroy()
		return
	end
	obj:PivotTo(workspace.CurrentCamera.CFrame + (workspace.CurrentCamera.CFrame.LookVector / 5))
	rs()
	obj:PivotTo(workspace.CurrentCamera.CFrame + (workspace.CurrentCamera.CFrame.LookVector / 5))
	rs()
	obj:PivotTo(workspace.CurrentCamera.CFrame + (workspace.CurrentCamera.CFrame.LookVector / 5))
	pp:InputHoldBegin()
	rs()
	pp:InputHoldEnd()
	rs()
	if pp.Parent == obj then
		pp.Parent = c
		pp.MaxActivationDistance = a
		pp.Enabled = b
		pp.HoldDuration = d
		pp.RequiresLineOfSight = e
	end
	obj:Destroy()
	cd[pp] = false
end
local function canGetPivot(pp)
	return pp.Parent.GetPivot
end
local fireproximityprompt = function(pp, i)
	if not i and (typeof(pp) ~= "Instance" or not pp:IsA("ProximityPrompt") or not pcall(canGetPivot, pp) or cd[pp] or not workspace.CurrentCamera or ((game.Players.LocalPlayer and game.Players.LocalPlayer.Character and game.Players.LocalPlayer.Character:FindFirstChild("HumanoidRootPart") and game.Players.LocalPlayer.Character.HumanoidRootPart or workspace.CurrentCamera).CFrame.Position - pp.Parent:GetPivot().Position).Magnitude > pp.MaxActivationDistance * 2) then return end
	if fppn then
		return fpp(pp)
	end
	task.spawn(fppFunc, pp)
end

local function insertCum(str)
	local new = str:gsub("(%u)", " %1")
	if new:sub(1, 1) == " " then
		new = new:sub(2)
	end

	return new:gsub("  ", " "):gsub("_", "") .. ""
end

local function getSelectedObject()
	return game:GetService("ReplicatedStorage").Client.Handlers.DraggableItemHandlers.ClientDraggableObjectHandler.DragHighlight.Adornee
end

local function throwObject(object)
	if (object:GetPivot().Position - plr.Character:GetPivot().Position).Magnitude > 20 then
		return
	end

	game:GetService("ReplicatedStorage").Shared.Remotes.RequestStartDrag:FireServer(object)

	local par

	while true do
		local drag1 = object:FindFirstChild("DragAttachment", math.huge)
		local drag2 = object:FindFirstChild("DragAlignPosition", math.huge)
		local drag3 = object:FindFirstChild("DragAlignOrientation", math.huge)

		if not drag1 and not drag2 and not drag3 and par then
			break
		end

		if drag1 then
			par = drag1.Parent
			drag1:Destroy()
			continue
		end
		if drag2 then
			par = drag2.Parent
			drag2:Destroy()
			continue
		end
		if drag3 then
			par = drag3.Parent
			drag3:Destroy()
			continue
		end

		task.wait()
	end

	task.wait()

	if par then
		par.AssemblyLinearVelocity = CFrame.lookAt(workspace.CurrentCamera.CFrame.Position, par:GetPivot().Position + Vector3.new(0, ((10000 - vals.ThrowPower)/10000) * 5 - 0.25)).LookVector * vals.ThrowPower
		task.wait()
	end

	game:GetService("ReplicatedStorage").Shared.Remotes.RequestStopDrag:FireServer()
end

local function throw()
	local obj = getSelectedObject()
	if not obj then
		local noti = WindUI:Notify({
			Title = "No!",
			Content = "Look at object you want to throw",
			Duration = 5,
		})
		return
	end

	throwObject(obj)
end

local esps = {}
local desps = {}
local monsters = {}

local tools = {}
local bonds = {}
local other = {}
local equippables = {}

local pickupable = { "Consumable", "Gun", "Weapon", "Melee", "Playable", "Tool" }
local activateable = { "Ammo" }
local armor = { "Equippable" }

local infoStored = {}

local function getInfo(object)
	rs(2)
	if not object or not object.Parent then return end
	if infoStored[object.Name] then return infoStored[object.Name] end

	local info = {}
	for i,v in object:WaitForChild("ObjectInfo", 9e9):GetChildren() do
		if v.Name ~= "Title" and v:IsA("TextLabel") then
			add(info, v.Text)
		end
	end

	infoStored[object.Name] = info
	return info
end

local function hasProperty(object, prop)
	if not object:FindFirstChild("ObjectInfo") then return false end

	local info = getInfo(object)

	if not info then return false end

	for i,v in info do
		if v == prop then
			return true
		end
	end

	return false
end

local function getColor(v)
	local val = v and v:GetAttribute("Value")
	if v.Name == "Bond" then
		return Color3.fromRGB(255, 170)
	elseif v.Name == "Coal" then
		return Color3.new(0.2, 0.2, 0.2)
	elseif v.Name == "Unicorn" then
		return Color3.fromRGB(85, 255, 255)
	elseif v.Name == "Bandage" then
		return Color3.fromRGB(255, 85, 255)
	elseif v.Name == "Snake Oil" then
		return Color3.fromRGB(0, 170)
	elseif hasProperty(v, "Ammo") then
		return Color3.fromRGB(255, 170, 125)
	elseif hasProperty(v, "Weapon") or hasProperty(v, "Gun") or hasProperty(v, "Melee") then
		return Color3.new(0.75, 0.5, 0.5)
	elseif val then
		if val <= 50 then
			return Color3.new(0.8, 0.8, 0.8):Lerp(Color3.fromRGB(255, 255, 75), val / 50)
		elseif val <= 175 then
			return Color3.fromRGB(255, 255, 75):Lerp(Color3.fromRGB(75, 255, 255), (val - 50) / 175)
		else
			return Color3.fromRGB(75, 255, 255):Lerp(Color3.fromRGB(255, 125, 255), (val - 175) / 325)
		end
	end
	return Color3.new(0.8, 0.8, 0.8)
end

local function getText(obj)
	local n = obj.Name
	local l = n:lower()

	if l:match("vase") then
		return "Vase"
	elseif l:match("outlaw") then
		return "Outlaw"
	elseif l:match("zombie") then
		return "Zombie"
	elseif l:match("nikola") then
		return "OMFG, IT IS THE MOTHERFUCKER"
	end

	return insertCum(n)
end

local function main(v)
	rs(2)
	if v and v.Parent then
		if v:IsA("ProximityPrompt") and not oprompts[v] then
			oprompts[v] = v.MaxActivationDistance
			v.MaxActivationDistance *= vals.ExtraPP
		elseif v:IsA("Model") and not game:GetService("Players"):GetPlayerFromCharacter(v) then
			if v.Name ~= "Moneybag" and v.Name ~= "Vault" then
				if v:GetAttribute("DangerScore") then
					local hum = v:WaitForChild("Humanoid", 9e9)

					local monster = esps[v.Name] or {HighlightEnabled = true, Color = Color3.new(0.35):Lerp(Color3.new(1), v:GetAttribute("DangerScore") / 100), Text = getText(v), ESPName = "MonsterESP"}
					esps[v.Name] = monster

					espFunc(v, monster)
					add(monsters, v)

					repeat task.wait() until not hum or not hum.Parent or isDead(hum)

					pcall(espLib.DeapplyESP, v)

					local dead = desps[v.Name] or {HighlightEnabled = true, Color = Color3.fromRGB(200, 150, 50):Lerp(Color3.fromRGB(255, 75, 0), v:GetAttribute("DangerScore") / 100), Text = getText(v), ESPName = "Dead MonsterESP"}
					desps[v.Name] = dead

					espFunc(v, dead)

					return
				end

				if v:GetAttribute("BloodColor") then
					local hum = v:WaitForChild("Humanoid", 9e9)

					local animal = esps[v.Name] or {HighlightEnabled = true, Color = Color3.new(0.8, 0.8, 0.8), Text = getText(v) .. (v:GetAttribute("Value") and " (" .. v:GetAttribute("Value") .. "$)" or ""), ESPName = "AnimalESP"}
					esps[v.Name] = animal

					espFunc(v, animal)

					repeat task.wait() until not hum or not hum.Parent or isDead(hum)

					pcall(espLib.DeapplyESP, v)

					local dead = desps[v.Name] or {HighlightEnabled = true, Color = Color3.new(1, 0.7, 0.7), Text = getText(v) .. (v:GetAttribute("Value") and " (" .. v:GetAttribute("Value") .. "$)" or ""), ESPName = "Dead AnimalESP"}
					desps[v.Name] = dead

					espFunc(v, dead)

					return
				end

				if v:FindFirstChild("ObjectInfo") and (v:FindFirstChild("Base") and v.Base:IsA("BasePart") and not v.Base.Anchored or not v:FindFirstChild("Base") or not v.Base:IsA("BasePart")) --[[or v.Parent == workspace.RuntimeItems]] then
					local tool = esps[v.Name] or {HighlightEnabled = true, Color = getColor(v), Text = getText(v) .. (v:GetAttribute("Value") and " (" .. v:GetAttribute("Value") .. "$)" or ""), ESPName = "ItemESP"}

					esps[v.Name] = tool

					espFunc(v, tool)

					for i,va in pickupable do
						if hasProperty(v, va) then
							add(tools, v)
							return
						end
					end

					for i,va in activateable do
						if hasProperty(v, va) then
							add(other, v)
							return
						end
					end

					for i,va in armor do
						if hasProperty(v, va) then
							add(equippables, v)
							return
						end
					end

					if hasProperty(v, "Currency") then
						add(bonds, v)
						return
					end
				end
			elseif v.Name == "Moneybag" then
				local price = tonumber(v:WaitForChild("MoneyBag", 9e9):WaitForChild("BillboardGui", 9e9):WaitForChild("TextLabel", 9e9).Text:gsub("%$", "") .. "")
				local bag = esps[price] or {HighlightEnabled = true, Color = Color3.fromRGB(85, 170, 0), Text = price .. "$", ESPName = "Money BagESP"}

				esps[price] = bag

				espFunc(v, bag)
				add(prompts, v.MoneyBag:WaitForChild("CollectPrompt", 9e9))
			elseif v.Name == "Vault" and v:FindFirstChild("Combination") then
				espFunc(v, {HighlightEnabled = true, Color = Color3.fromRGB(85, 170, 0), Text = "Code: [" .. tostring(v.Combination.Value):gsub("", " ") .. "]", ESPName = "Vault CodeESP"})
			end
		end
	end
end

local function getClosestMonster(mode)
	mode = mode or vals.Mode
	if mode == "Angle" and workspace.CurrentCamera then
		local a, d, m = math.huge, math.huge, nil
		for i,v in monsters do
			if v and v.Parent and not isDead(v) then
				if vals.Raycast and raycast(workspace.CurrentCamera.CFrame.Position, v:GetPivot().Position, v:GetDescendants()) or (workspace.CurrentCamera.CFrame.Position - v:GetPivot().Position).Magnitude > vals.KAR then
					continue
				end

				local di = (plr.Character:GetPivot().Position - v:GetPivot().Position).Magnitude
				local an = ((workspace.CurrentCamera.CFrame.Position + (workspace.CurrentCamera.CFrame.LookVector * di)) - v:GetPivot().Position).Magnitude

				if an <= a then
					d = di
					a = an
					m = v
				end
			end
		end

		return m, d
	elseif mode == "Random" then
		local allowedMonsters = {}
		for i,v in monsters do
			if v and v.Parent and not isDead(v) then
				if vals.Raycast and raycast(workspace.CurrentCamera.CFrame.Position, v:GetPivot().Position, v:GetDescendants()) or (workspace.CurrentCamera.CFrame.Position - v:GetPivot().Position).Magnitude > vals.KAR then
					continue
				end

				add(allowedMonsters, v)
			end
		end

		if #allowedMonsters > 0 then
			local monster = allowedMonsters[math.random(1, #allowedMonsters)]
			return monster, monster and (plr.Character:GetPivot().Position - monster:GetPivot().Position).Magnitude
		end
	else
		local d, m = math.huge, nil
		for i,v in monsters do
			if v and v.Parent and not isDead(v) then
				if vals.Raycast and raycast(workspace.CurrentCamera.CFrame.Position, v:GetPivot().Position, v:GetDescendants()) or (workspace.CurrentCamera.CFrame.Position - v:GetPivot().Position).Magnitude > vals.KAR then
					continue
				end

				local di = (plr.Character:GetPivot().Position - v:GetPivot().Position).Magnitude

				if di <= d then
					d = di
					m = v
				end
			end
		end

		return m, d
	end
end

local gncm, hmm = getfenv().getnamecallmethod, getfenv().hookmetamethod
if hmm and gncm then
	local old; old = hmm(game, "__namecall", function(self, ...)
		if vals.Aimbot and self == s and gncm() == "FireServer" then
			local args = { ... }

			local m = getClosestMonster()

			if m then
				local hits = {}
				for i=1, args[2].WeaponConfiguration.PelletsPerBullet.Value do
					hits[tostring(i)] = m.Humanoid
				end

				local head = m:FindFirstChild("Head") or m:GetPivot()

				args[3] = CFrame.lookAt(head.Position + Vector3.new(0, 1), head.Position)
				args[4] = hits
			elseif vals.SaveBullets then
				args[2].ClientWeaponState.CurrentAmmo.Value += 1
				error("Cancel shoot", 0)
			end

			return s.FireServer(s, unpack(args))
		elseif vals.SaveBullets and self == s and gncm() == "FireServer" and not getClosestMonster() then
			({ ... })[2].ClientWeaponState.CurrentAmmo.Value += 1
			error("Cancel shoot", 0)
		end

		return old(self, ...)
	end)

	hooks[#hooks + 1] = function()
		hmm(game, "__namecall", old)
	end
end

task.spawn(function()
	while not closed and task.wait(0.1) do
		if vals.GKA and plr.Character then
			local m = getClosestMonster()
			if m then
				for v in myGuns do
					if v and v.Parent and v:FindFirstChild("WeaponConfiguration") then
						pcall(shoot, v, m)
						task.wait(0.01)
					end
				end
			end
		end
	end
end)
task.spawn(function()
	while not closed and task.wait(0.1) do
		if vals.ARG and plr.Character then
			for v in myGuns do
				if v and v.Parent and v:FindFirstChild("WeaponConfiguration") then
					pcall(reload, v)
				end
			end
		end
	end
end)

local ad = 30
local farEvents = {}

local function equipUntilNoZombie(tool, zombie)
	tool.Parent = plr.Character

	if not farEvents[zombie] then
		farEvents[zombie] = Instance.new("BindableEvent")
		repeat task.wait() until not vals.MA or isDead(zombie)

		farEvents[zombie]:Fire()

		farEvents[zombie]:Destroy()
		farEvents[zombie] = nil
	else
		farEvents[zombie].Event:Wait()
	end

	tool.Parent = plr.Backpack
end

task.spawn(function()
	while not closed and task.wait(0.1) do
		if vals.MA then
			local m, d = getClosestMonster("Distance")
			if m and d <= ad then
				for v in melee do
					if v and v.Parent and v:FindFirstChild("SwingEvent") then
						if v.Parent == plr.Backpack then
							task.spawn(equipUntilNoZombie, v, m)
						end

						v.SwingEvent:FireServer(CFrame.lookAt(plr.Character:GetPivot().Position, m:GetPivot().Position + Vector3.new(0, 2)).LookVector)
					end
				end
			end
		end
	end
end)

for i,v in workspace:GetDescendants() do
	task.spawn(main, v)
end
cons[#cons+1] = workspace.DescendantAdded:Connect(main)

local void = pcall(function()
	workspace.FallenPartsDestroyHeight = workspace.FallenPartsDestroyHeight
end)

local oilCooldown = false
cons[#cons+1] = game:GetService("RunService").RenderStepped:Connect(function()
	if vals.FB then
		game.Lighting.Ambient = Color3.new(1, 1, 1)
		game.Lighting.Brightness = 1.5
	end
	if vals.NC then
		plr.CameraMode = Enum.CameraMode.Classic
	end
	if void then
		workspace.FallenPartsDestroyHeight = vals.NoVoid and 0/0 or -500
	end
	plr.DevCameraOcclusionMode = vals.NC and Enum.DevCameraOcclusionMode.Invisicam or Enum.DevCameraOcclusionMode.Zoom
	game.Lighting.GlobalShadows = not vals.FB
	if vals.AutoCollectBags then
		for i,v in prompts do
			if v and v.Parent then
				fireproximityprompt(v)
			else
				remove(prompts, v)
			end
		end
	end
	if plr.Character then
		if vals.Noclip then
			for i,v in plr.Character:GetDescendants() do
				if v and v:IsA("BasePart") then
					v.CanCollide = false
				end
			end
		elseif plr.Character:FindFirstChild("HumanoidRootPart") then
			plr.Character.HumanoidRootPart.CanCollide = true
		end

		if vals.AutoPickTools then
			for i,v in tools do
				if v and v.Parent then
					if (v:GetPivot().Position - plr.Character:GetPivot().Position).Magnitude <= 30 then
						game:GetService("ReplicatedStorage").Remotes.Tool.PickUpTool:FireServer(v)
					end
				else
					remove(prompts, v)
				end
			end
		end
		if vals.AutoPickOther then
			for i,v in other do
				if v and v.Parent then
					if (v:GetPivot().Position - plr.Character:GetPivot().Position).Magnitude <= 30 then
						game:GetService("ReplicatedStorage").Packages.RemotePromise.Remotes.C_ActivateObject:FireServer(v)
					end
				else
					remove(prompts, v)
				end
			end
		end
		if vals.AutoPickBonds then
			for i,v in bonds do
				if v and v.Parent then
					if (v:GetPivot().Position - plr.Character:GetPivot().Position).Magnitude <= 30 then
						game:GetService("ReplicatedStorage").Packages.RemotePromise.Remotes.C_ActivateObject:FireServer(v)
					end
				else
					remove(prompts, v)
				end
			end
		end
		if vals.AutoPickArmor then
			for i,v in equippables do
				if v and v.Parent then
					if (v:GetPivot().Position - plr.Character:GetPivot().Position).Magnitude <= 30 then
						game:GetService("ReplicatedStorage").Remotes.Object.EquipObject:FireServer(v)
					end
				else
					remove(prompts, v)
				end
			end
		end

		local bandage = getFirst(heals.Bandage)
		if bandage and bandage.Parent and plr.Character:FindFirstChildOfClass("Humanoid").Health <= vals.BandageUse then
			return bandage.Use:FireServer(bandage)
		end

		local oil = getFirst(heals["Snake Oil"])
		if not oilCooldown and oil and oil.Parent and plr.Character:FindFirstChildOfClass("Humanoid").Health <= vals.OilUse then
			oilCooldown = true
			oil.Use:FireServer(oil)
			task.wait(vals.OilUseCooldown)
			oilCooldown = false
		end
	end
end)

cons[#cons+1] = game:GetService("ProximityPromptService").PromptButtonHoldBegan:Connect(function(pp)
	if vals.II then
		fireproximityprompt(pp, true)
	end
end)

local Window = WindUI:CreateWindow({
	Title = "Snow", -- UI Title
	Icon = "https://cdn-pic.xncly.xyz/upload/O2FTGkMtIUQNaQc80Ldv.png", -- Url or rbxassetid or lucide
	Author = "Dead Rails", -- Author & Creator
	Folder = "Snow", -- Folder name for saving data (And key)
	Size = UDim2.fromOffset(380, 260), -- UI Size
	Transparent = true,-- UI Transparency
	Theme = "Dark", -- UI Theme
	SideBarWidth = 170, -- UI Sidebar Width (number)
	HasOutline = true, -- Adds Outlines to the window
})

Window:EditOpenButton({
	Title = "Snow", -- Title
	Icon = "rbxassetid://71315952129083", -- Icon
	CornerRadius = UDim.new(0,5), -- Radius
	StrokeThickness = 1, -- Stroke Thickness
	Color = ColorSequence.new( -- Gradient
		Color3.fromHex("FF0F7B"), 
		Color3.fromHex("F89B29")
	),
	Position = UDim2.new(0.5,0,0.5,0), -- Position
	Enabled = true,   -- true or false
	Draggable = true, -- true or false
})





local General = Window:Tab({Title = "General", Icon = "globe"})

General:Section({ 
	Title = "ตัวละคร", 
	TextXAlignment = "Center",
	TextSize = 17, 
})

General:Toggle({
	Title = "ทะลุกำแพง (No Clip)",
	Icon = "check",
	Value = false,
	Callback = function(state)
		vals.Noclip = state
	end,
})

General:Section({ 
	Title = "ไอเทม",
	TextXAlignment = "Center",
	TextSize = 17,
})

General:Toggle({
	Title = "เก็บถุงเงินอัตโนมัติ",
	Icon = "check",
	Value = false,
	Callback = function(state)
		vals.AutoCollectBags = state
	end,
})

General:Toggle({
	Title = "เก็บอุปกรณ์อัตโนมัติ",
	Icon = "check",
	Value = false,
	Callback = function(state)
		vals.AutoPickTools = state
	end,
})

General:Toggle({
	Title = "เกราะอัตโนมัติ", 
	Icon = "check",
	Value = false,
	Callback = function(state)
		vals.AutoPickArmor = state
	end,
})

General:Toggle({
	Title = "เก็บพันธบัตรอัตโนมัติ",
	Icon = "check",
	Value = false,
	Callback = function(state)
		vals.AutoPickBonds = state
	end,
})

General:Toggle({
	Title = "เก็บของอื่น ๆ อัตโนมัติ", 
	Icon = "check",
	Value = false,
	Callback = function(state)
		vals.AutoPickOther = state
	end,
})

General:Section({ 
	Title = "ปรับแต่งเกม", 
	TextXAlignment = "Center",
	TextSize = 17,
})

General:Toggle({
	Title = "โต้ตอบทันที",
	Icon = "check",
	Value = false,
	Callback = function(state)
		vals.II = state
	end,
})


General:Slider({
	Title = "ระยะการกดปุ่มโต้ตอบ (เพิ่ม/ลด)",
	Step = 0.01,
	Value = {
		Min = 1,
		Max = 2,
		Default = 1,
	},
	Callback = function(value)
		vals.ExtraPP = value
		for i,v in oprompts do
			if i and i.Parent and v then
				i.MaxActivationDistance = v * value
			end
		end
	end
})

if void then
	General:Toggle({
		Title = "กันตกแมพ (No Void)",
		Desc = "(ป้องกันการตายเมื่อร่วงลงใต้แมพ)",
		Icon = "check",
		Value = false,
		Callback = function(state)
			vals.NoVoid = state
		end,
	})
end

General:Section({ 
	Title = "การฟื้นฟูพลังชีวิต", -- Healing
	TextXAlignment = "Center",
	TextSize = 17,
})

General:Slider({
	Title = "ใช้ผ้าพันแผลอัตโนมัติเมื่อ HP น้อยกว่า:",
	Step = 0.5,
	Value = {
		Min = 0,
		Max = 99.5,
		Default = 0,
	},
	Callback = function(value)
		vals.BandageUse = value
	end
})

General:Slider({
	Title = "ใช้น้ำมันงูอัตโนมัติเมื่อ HP น้อยกว่า:",
	Step = 0.5,
	Value = {
		Min = 0,
		Max = 100,
		Default = 0,
	},
	Callback = function(value)
		vals.OilUse = value
	end
})

General:Slider({
	Title = "คูลดาวน์การใช้น้ำมันงู (วินาที):",
	Step = 0.1,
	Value = {
		Min = 0,
		Max = 10,
		Default = 5,
	},
	Callback = function(value)
		vals.OilUseCooldown = value
	end
})




local Visual = Window:Tab({Title = "Visual", Icon = "scan-eye"})


Visual:Section({ 
	Title = "ข้อมูล", -- Info
	TextXAlignment = "Center",
	TextSize = 17,
})

local ShowDistance = Visual:Paragraph({
	Title = "ระยะทาง: N/A",
	Image = "train-front",
	ImageSize = 20,
})

local ShowTime = Visual:Paragraph({
	Title = "เวลา: N/A",
	Image = "alarm-clock",
	ImageSize = 20,
})

local ShowSpeed = Visual:Paragraph({
	Title = "ความเร็ว: N/A",
	Image = "clock-arrow-up",
	ImageSize = 20,
})

local ShowFuel = Visual:Paragraph({
	Title = "น้ำมัน: N/A",
	Image = "flame",
	ImageSize = 20,
})

Visual:Section({ 
	Title = "การแสดงผล", -- Visual
	TextXAlignment = "Center",
	TextSize = 17,
})

Visual:Toggle({
	Title = "แสงสว่างสูงสุด (Full Bright)",
	Icon = "check",
	Value = false,
	Callback = function(state)
		vals.FB = state
	end,
})

Visual:Toggle({
	Title = "กล้องปกติ (Normal Camera)",
	Icon = "check",
	Value = false,
	Callback = function(state)
		vals.NC = state
		rs(2)
		plr.CameraMode = vals.NC and Enum.CameraMode.Classic or Enum.CameraMode.LockFirstPerson
		if state then
			local noti = WindUI:Notify({
				Title = "เปิดกล้องปกติ",
				Content = "ตอนนี้คุณสามารถซูมออกได้",
				Duration = 5,
			})
		end
	end,
})

Visual:Section({ 
	Title = "ESP (ไฮไลต์วัตถุ/ศัตรู)",
	TextXAlignment = "Center",
	TextSize = 17,
})


local activated = false


for i,v in vals.ESP do
	Visual:Toggle({Title = i:gsub("ESP", " ESP"), Icon = "check", Value = v, Callback = function(state)
		espLib.ESPValues[i] = state
	end})
end




local Weapon = Window:Tab({Title = "Weapon", Icon = "target"})


Weapon:Section({ 
	Title = "วงโจมตีอัตโนมัติ", -- Kill Aura
	TextXAlignment = "Center",
	TextSize = 17,
})

Weapon:Toggle({
	Title = "ปืนยิงอัตโนมัติ (Gun Aura)",
	Icon = "check",
	Value = false,
	Callback = function(state)
		vals.GKA = state
	end,
})

Weapon:Toggle({
	Title = "โจมตีระยะใกล้อัตโนมัติ (Melee Aura)",
	Icon = "check",
	Value = false,
	Callback = function(state)
		vals.MA = state
	end,
})

Weapon:Section({ 
	Title = "การตั้งค่า", -- Settings
	TextSize = 17,
})

Weapon:Slider({
	Title = "ระยะปืน (Gun Radius)",
	Step = 1,
	Value = {
		Min = 10,
		Max = 2501,
		Default = vals.KAR,
	},
	Callback = function(value)
		vals.KAR = value >= 2751 and 228_1488 or value >= 2501 and 2500 or value
	end
})



local t = {"Distance", "Angle", "Random"}

Weapon:Dropdown({
	Title = "เลือกเป้าหมายโดย", -- Target by
	Value = "Distance",
	Multi = false,
	AllowNone = true,
	Values = t,
	Callback = function(Tab)
		vals.Mode = t[Tab]
	end
})

Weapon:Toggle({
	Title = "รีโหลดกระสุนอัตโนมัติ", -- Auto Reload
	Icon = "check",
	Value = false,
	Callback = function(state)
		vals.ARG = state
	end,
})

if hmm and gncm then

	Weapon:Toggle({
		Title = "เล็งอัตโนมัติ (Aimbot)", -- Aimbot
		Icon = "check",
		Value = false,
		Callback = function(state)
			vals.Aimbot = state
		end,
	})

	Weapon:Toggle({
		Title = "ประหยัดกระสุน (ไม่ยิงถ้าไม่มีเป้า)", -- Save Bullet
		Icon = "check", 
		Value = false,
		Callback = function(state)
			vals.SaveBullets = state
		end,
	})
end


espFunc(workspace:WaitForChild("Train", 9e9), {HighlightEnabled = false, Color = Color3.fromRGB(55, 65, 65), Text = "Train", ESPName = "Train ESP"})

cons[#cons+1] = game:GetService("RunService").RenderStepped:Connect(function()
	if workspace.Train.TrainControls:FindFirstChild("TimeDial") then
		ShowTime:SetTitle("Time: " .. workspace.Train.TrainControls.TimeDial.SurfaceGui.TextLabel.Text)

		ShowDistance:SetTitle("Traveled: " .. workspace.Train.TrainControls.DistanceDial.SurfaceGui.TextLabel.Text)

		ShowSpeed:SetTitle("Speed: " .. (math.round((workspace.Train.TrainControls.Spedometer.SurfaceGui.ImageLabel.Gauge.Rotation - 120) / 163 * 65 * 10) / 10) .. " s/s")

		ShowFuel:SetTitle("Fuel: " .. (math.round((workspace.Train.TrainControls.Fuel.SurfaceGui.ImageLabel.Gauge.Rotation - 120) / 300 * 1000) / 10) .. "%")
	end
end)

local Utility = Window:Tab({Title = "Utility", Icon = "wrench"})

Utility:Section({
    Title = "วาปไปที่รถไฟ",
    TextXAlignment = "Center",
    TextSize = 17,
})

Utility:Button({
    Title = "Teleport to Train",
    Icon = "chair",
    Callback = function()
        -- โค้ด Teleport to Train ที่มีอยู่เดิม
        local train = workspace:FindFirstChild("Train")
        if not train then
            WindUI:Notify({
                Title = "Error",
                Content = "Train not found in workspace!",
                Duration = 5,
            })
            return
        end

        local chair
        for _,part in pairs(train:GetDescendants()) do
            if part:IsA("Seat") or part:IsA("VehicleSeat") then
                chair = part
                break
            end
        end

        if not chair then
            WindUI:Notify({
                Title = "Error",
                Content = "No chair found in train!",
                Duration = 5,
            })
            return
        end

        local character = plr.Character
        if not character then return end

        local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        
        if humanoidRootPart and humanoid then
            humanoidRootPart.CFrame = chair.CFrame * CFrame.new(0, 0.5, 0)
            wait(0.5)
            humanoid.Sit = true
            
            local success = false
            for _=1, 10 do  
                if humanoid.Sit and humanoid.SeatPart == chair then
                    success = true
                    break
                end
                wait(0.1)
            end
            
            if success then
                WindUI:Notify({
                    Title = "Success",
                    Content = "Teleported to train chair!",
                    Duration = 5,
                })
            else
                WindUI:Notify({
                    Title = "Warning",
                    Content = "Teleported but couldn't sit automatically",
                    Duration = 5,
                })
            end
        end
    end
})

Window:SelectTab(1) 
