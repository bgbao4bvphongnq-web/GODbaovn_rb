--//  Client // --

-- Services --
local replicatedStorage = game:GetService("ReplicatedStorage")
local userInputService = game:GetService("UserInputService")
local players = game:GetService("Players")

-- Modules --
local modules = replicatedStorage:WaitForChild("Modules")

local globalModules = require(modules.globalModules)
local clientModules = require(modules.clientModules)

local animationService = require(modules.animationService)

local hitboxService = require(globalModules.hitboxService)

 -- Some variables -- 
local player = players.LocalPlayer
local character = player.Character
local humanoid = character.Humanoid
local rootPart = character.HumanoidRootPart

local events = replicatedStorage:WaitForChild("Communication").Events
local functions = replicatedStorage:WaitForChild("Communication").Functions

local animations = animationService:LoadAnimations(character).animations

local combatEvent = events.combat
local cooldownEvent = events.cooldown

-- combat system variables --
local currentTrack = nil
local resetCooldown = 1
local comboDelay = 1.5
local attackCooldown = 0.45

local combo = 1
local maxCombo = 5
local canAttack = true
local canBlock = true

userInputService.InputBegan:Connect(function(input, typing)
	if typing then -- if the player is writing in the chat, the code will ignore his input
		return
	end
	
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		if not character:GetAttribute("blocking") and not character:GetAttribute("comboCooldown") then -- running the block of code if player isn't blocking or isn't on comboCooldown
			if canAttack then -- running the part of code that gives life to the combatSystem(if canAttack is true)
				canAttack = false
				canBlock = false
				
				character:SetAttribute("running", false) -- changing the running attribute to prevent player from running while attacking
				-- different attack types depending on player key and attribute state
				local airCombo = (userInputService:IsKeyDown(Enum.KeyCode.Space) and not character:GetAttribute("aerialCombo") and combo == 4) or false 
				local downSlam = (combo == 5 and character:GetAttribute("aerialCombo"))
				-- getting the respective animationTrack relative to the attackType
				local track = airCombo and animations.launcher or downSlam and animations.downSlam
					or animations[tostring(combo)]
				
				if not track then
					warn("track does not exist")
						
					return -- exiting the function if the track doesn't exists
				end
				
				humanoid.JumpPower = 0 -- preventing player from jumping for better immersive combat
				
				task.delay(track.Length + 0.45, function()
					if currentTrack == track then
						humanoid.JumpPower = 50
					end
				end) -- enabling jumppower after short cooldown
				
 				-- stoping previus animation(if its looped)
				if currentTrack then
					if currentTrack.IsPlaying then
						currentTrack:Stop()
					end
				end
				
				local currentCombo = combo
				 -- playing the currentAnimation obtained previously in the track variable
				currentTrack = track
				currentTrack:Play()
				
				track:GetMarkerReachedSignal("ConnectionPoint"):Once(function()-- using animationSignalMarker to run some hitbox and serverTrigger scripts
					local foundTargets = hitboxService:spatialQuery(character, track.Name == "launcher" and "single" or "multi")
					
					--print(currentCombo .. ":",foundTargets)

					combatEvent:FireServer("registerHit", {foundTargets, currentCombo, airCombo})
				end)
				 -- reseting attack and block once the attack has ended
				track:GetMarkerReachedSignal("AttackEnded"):Once(function()
					canAttack = true
					canBlock = true
				end)
				-- creating a server sided cooldown and creating a soundEffect
				cooldownEvent:FireServer({["attacking"] = track.Length + 0.35, ["noJump"] = track.Length + 0.45})
				combatEvent:FireServer("soundEffects", {airCombo})
				
				-- creating the logic of the combat algorithm
				if combo >= 5 then
					
					canAttack = false
					-- reseting the combo once the combo has ended
					task.delay(comboDelay, function()
						combo = 1
						canAttack = true
						humanoid.JumpPower = 50
					end)
				else
					local old = combo
					--reseting the combo if the player takes too long to attack
					task.delay(resetCooldown, function()
						if combo == old + 1 then
							combo = 1
							
							canAttack = true
						end
					end)
				end
				
				combo += 1
			end
		end
	end
end)
-- another mechanism to reset the combo
combatEvent.OnClientEvent:Connect(function(request)
	if request == "resetCombo" then
		combo = 1
	end
end)

---------- // Server // ----

-- Services --
local replicatedStorage = game:GetService("ReplicatedStorage")
local players = game:GetService("Players")
-- Modules --
local modules = replicatedStorage:WaitForChild("Modules")

local globalModules = require(modules.globalModules)
local cooldownService = require(modules.cooldownService)

local hitboxService = require(globalModules.hitboxService)
 -- Events -- 
local Events = replicatedStorage:WaitForChild("Communication").Events
local Functions = replicatedStorage:WaitForChild("Communication").Functions

local combatEvent = Events.combat
local cooldownEvent = Events.cooldown

local information = require(modules.information)
 -- some combat Functions that affects the gameplay
local COMBAT_FUNCTIONS = {
	registerHit = function(player, combatInformations, foundTargets, currentCombo, airCombo)
		if not cooldownService:getActiveCooldowns(player, {"comboCooldown", "block"}) then -- running the code if "comboCooldown" and "block" isnt on cooldown
			if cooldownService:getActiveCooldowns(player, "soundPlayed") then -- creating a newSound cooldown if it doesnt exists
				cooldownService:setCooldown(player, "soundPlayed", false)
			end
			 -- changing combatInformations elements based on combo value/ creating a new cooldown when the combo is finished
			if currentCombo == 5 then
				combatInformations.knockback = true
				combatInformations.canBlockBreak = true
				
				cooldownService:setCooldown(player, "comboCooldown"):extend(1.5)
			end
			-- changing the aerial element if aerial is true
			if airCombo then
				combatInformations.aerial = true
			end
			
			-- running hitboxService function to start the algoritm that checks if the damage is legal
			hitboxService:registerDamage(foundTargets, player, currentCombo, airCombo, combatInformations)
		end
	end,
	
	block = function(player, combatInformations, active) -- server side block functionabilities
		local character = player.Character
		local rootPart = character:FindFirstChild("HumanoidRootPart")
		local humanoid = character:FindFirstChild("Humanoid")
		
		if not rootPart or not humanoid then
			return
		end
		-- part of code that checks if the player has the attribute block as true and, if the player does, creates a intConstrained value that i personaly think that works good in this situation.
		if not character:GetAttribute("block") then
			if not cooldownService:getActiveCooldowns(player, {"block", "blockCooldown",  "stunned"}) then
				print("Blocking")
				
				local blockBarValue = Instance.new("IntConstrainedValue")
				blockBarValue.MaxValue = 5
				blockBarValue.Value = 5
				blockBarValue.Name = "blockBar"
				blockBarValue.Parent = character
				-- creating a new cooldown called block. Toggle cooldown type
				cooldownService:setCooldown(character, "block")
			end
		else -- part of the code that runs if the player is currently blocking
			print("stopped blocking")
			
			local blockBarValue = character:FindFirstChild("blockBarValue")
			
			if blockBarValue then
				blockBarValue:Destroy()
			end
			 -- removing block cooldown, with a delay of 1 second
			cooldownService:setCooldown(character, {["block"] = false, ["blockCooldown"] = 1})
		end
	end,
	
	soundEffects = function(player) -- creating sound effects with cooldown creating
		local character = player.Character
		
		if not cooldownService:getActiveCooldowns(player, {"comboCooldown", "soundPlayed", "block"}) then
			local swingSound = replicatedStorage.Sounds.Combat.swing:Clone()

			swingSound.Parent = character.HumanoidRootPart
			swingSound:Play()
			
			cooldownService:setCooldown(player, "soundPlayed"):extend(0.5, false)
		end
	end,
}

combatEvent.OnServerEvent:Connect(function(player, actionName, arguments) -- listening and calling the function of the COMBAT_FUNCTIONS with the respective key
	local character = player.Character
	
	--print("action: " .. actionName)
	local combatInformations = information("combat")
	COMBAT_FUNCTIONS[actionName](player, combatInformations, unpack(arguments))
end)

-- creating a cooldown and reseting after length(in seconds) (similar to Debris, but with cooldowns)
cooldownEvent.OnServerEvent:Connect(function(player, arguments)
	for name, length in next, arguments do
		cooldownService:setCooldown(player, name):extend(length, false)
	end
end)
