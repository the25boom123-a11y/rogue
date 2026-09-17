-- Rogue village attack event. This stays server-side since spawning, rewards and damage credit all need to be authoritative.
local RS = game:GetService('ReplicatedStorage')
local SS = game:GetService('ServerStorage')
local SSS = game:GetService('ServerScriptService')
local Players = game:GetService('Players')

local RewardEXP = require(SSS.Services.RewardEXP)

local SpawnFolder = workspace['Automated Events']

-- These tables are kept up top so the event can be rebalanced without touching the event loop.
local ATTACKABLE_VILLAGES = {'Leaf', 'Mist', 'Sand', 'Cloud', 'Stone'}

local ROGUE_JUTSU_POOLS = {
    Fire = {'Fireball', 'Great Fireball', 'Phoenix Flower', 'Flamethrower', 'Fire Burst'},
    Lightning = {'Chidori', 'Lightning Bolt', 'Lightning Strike', 'Lightning Balls', 'Chidori Senbon'},
    Water = {'Water Dart', 'Waterfall', 'Water Dragon', 'Great Waterfall', 'Water Wall'},
    Wind = {'Wind Bullet', 'Breakthrough', 'Wind Scythe', 'Violent Whirlwind', 'Gale Palm'},
    Earth = {'Earth Spikes', 'Earth Pillar', 'Earth Rampart', 'Mud Wall', 'Rock Gun'},
}

local KAGE_CHAIR_VILLAGES = {
    KageChairLeaf = 'Leaf',
    KageChairMist = 'Mist',
    KageChairSand = 'Sand',
    KageChairCloud = 'Cloud',
    KageChairStone = 'Stone',
}

local REWARD_XP = {500, 350, 250, 150}
local REWARD_CURRENCY = {100, 75, 50, 30}

local ROGUE_COUNT = 4
local EVENT_TIMEOUT = 300
local ROGUE_HEALTH = 450
local ROGUE_RANK = 20

local isActive = false
local damageTracker = {} -- {[player] = damageDealt}
local spawnedRogues = {}

-- I use each village's Kage chair as a reliable point to build the attack around.
local function getVillageKageChairPosition(villageName)
    for _, chairModel in ipairs(workspace.KageChairs:GetChildren()) do
        if not chairModel:IsA('Model') then continue end
        for _, child in ipairs(chairModel:GetChildren()) do
            if child:IsA('MeshPart') and KAGE_CHAIR_VILLAGES[child.Name] == villageName then
                return chairModel:GetPivot().Position
            end
        end
    end
    return nil
end

local function selectRandomVillage()
    return ATTACKABLE_VILLAGES[math.random(1, #ATTACKABLE_VILLAGES)]
end

-- Rogues get one element theme per spawn instead of pulling random moves from every element.
local function selectRogueJutsu()
    local poolNames = {}
    for name, _ in pairs(ROGUE_JUTSU_POOLS) do
        table.insert(poolNames, name)
    end
    local poolName = poolNames[math.random(1, #poolNames)]
    local pool = ROGUE_JUTSU_POOLS[poolName]

    local selected = {}
    local count = math.random(2, 3)
    local available = {table.unpack(pool)}
    for i = 1, math.min(count, #available) do
        local idx = math.random(1, #available)
        table.insert(selected, available[idx])
        table.remove(available, idx)
    end
    return selected
end

-- Most of the setup happens here so every NPC enters the normal combat systems in a predictable state.
local function createRogueNPC(villageName, spawnPosition, rogueIndex)
    local aiTemplate = SS.Assets:FindFirstChild('AI Test')
    if not aiTemplate then
        warn('[RogueAttack] AI Test template not found!')
        return nil
    end

    local rogue = aiTemplate:Clone()
    rogue.Name = 'Rogue ' .. rogueIndex

    local stats = rogue:FindFirstChild('Stats')
    if stats then
        local nameVal = stats:FindFirstChild('Name')
        if nameVal and nameVal:IsA('StringValue') then
            nameVal.Value = 'Rogue Ninja'
        end

        local gender = stats:FindFirstChild('Gender')
        if gender and gender:IsA('StringValue') then
            gender.Value = math.random(1, 2) == 1 and 'Male' or 'Female'
        end
    end

    -- The combat scripts expect these values to exist, so I build the same state container players use.
    local effects = Instance.new('Folder')
    effects.Name = 'Effects'
    effects.Parent = rogue

    local chakra = Instance.new('NumberValue')
    chakra.Name = 'Chakra'
    chakra.Value = 99999
    chakra:SetAttribute('MaxValue', 99999)
    chakra.Parent = effects

    -- Fast hand signs make the NPC responsive without changing the jutsu modules themselves.
    local handSignBoost = Instance.new('NumberValue')
    handSignBoost.Name = 'HandSignBoost'
    handSignBoost.Value = 0.3
    handSignBoost.Parent = effects

    local dontAttack = Instance.new('ObjectValue')
    dontAttack.Name = 'DontAttack'
    dontAttack.Value = rogue
    dontAttack.Parent = effects

    local hit = Instance.new('IntValue')
    hit.Name = 'Hit'
    hit.Value = 0
    hit.Parent = effects

    local displayName = Instance.new('StringValue')
    displayName.Name = 'DisplayName'
    displayName.Value = 'Rogue Ninja'
    displayName.Parent = effects

    local clan = Instance.new('StringValue')
    clan.Name = 'Clan'
    clan.Value = 'Rogue'
    clan.Parent = effects

    if stats then
        local handsignExp = stats:FindFirstChild('HandsignExperience')
        if handsignExp and handsignExp:IsA('NumberValue') then
            handsignExp.Value = 2500
        elseif not handsignExp then
            local newExp = Instance.new('NumberValue')
            newExp.Name = 'HandsignExperience'
            newExp.Value = 2500
            newExp.Parent = stats
        end
    end

    -- Health is set here and again after parenting because older character handlers can overwrite it on spawn.
    local humanoid = rogue:FindFirstChildOfClass('Humanoid')
    if humanoid then
        humanoid.MaxHealth = ROGUE_HEALTH
        humanoid.Health = ROGUE_HEALTH
    end

    -- Rogues borrow the target village outfit so the event does not need a separate character set for every village.
    local villageClothes = SS.Assets.Clothes.Villages:FindFirstChild(villageName)
    if villageClothes then
        for _, clothing in ipairs(rogue:GetChildren()) do
            if clothing:IsA('Shirt') or clothing:IsA('Pants') then
                clothing.Parent = nil
            end
        end

        if stats then
            for _, clothing in ipairs(stats:GetChildren()) do
                if clothing:IsA('Shirt') or clothing:IsA('Pants') then
                    clothing.Parent = nil
                end
            end
        end

        local shirt = villageClothes:FindFirstChildOfClass('Shirt')
        local pants = villageClothes:FindFirstChildOfClass('Pants')
        if shirt then
            local newShirt = shirt:Clone()
            newShirt.Name = 'Clothing'
            newShirt.Parent = rogue
        end
        if pants then
            local newPants = pants:Clone()
            newPants.Name = 'Clothing'
            newPants.Parent = rogue
        end
    end

    -- The overhead display is made in code because these NPCs are cloned at runtime.
    local head = rogue:FindFirstChild('Head')
    if head and head:IsA('BasePart') then
        local bbGui = Instance.new('BillboardGui')
        bbGui.Name = 'RogueNameTag'
        bbGui.Adornee = head
        bbGui.Size = UDim2.new(8, 0, 2.5, 0)
        bbGui.StudsOffset = Vector3.new(0, 3, 0)
        bbGui.AlwaysOnTop = true
        bbGui.MaxDistance = 150
        bbGui.Parent = head

        local nameLabel = Instance.new('TextLabel')
        nameLabel.Name = 'NameLabel'
        nameLabel.Size = UDim2.new(1, 0, 0.5, 0)
        nameLabel.Position = UDim2.new(0, 0, 0, 0)
        nameLabel.BackgroundTransparency = 1
        nameLabel.Text = 'Rogue Ninja'
        nameLabel.TextColor3 = Color3.fromRGB(255, 50, 50)
        nameLabel.TextStrokeTransparency = 0.5
        nameLabel.TextStrokeColor3 = Color3.new(0, 0, 0)
        nameLabel.Font = Enum.Font.GothamBold
        nameLabel.TextScaled = true
        nameLabel.Parent = bbGui

        -- The tag is intentionally simple: name plus health is enough information during a crowded event.
        local healthBg = Instance.new('Frame')
        healthBg.Name = 'HealthBg'
        healthBg.Size = UDim2.new(0.8, 0, 0.25, 0)
        healthBg.Position = UDim2.new(0.1, 0, 0.6, 0)
        healthBg.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
        healthBg.BorderSizePixel = 0
        healthBg.Parent = bbGui

        local healthCorner = Instance.new('UICorner')
        healthCorner.CornerRadius = UDim.new(0, 4)
        healthCorner.Parent = healthBg

        local healthFill = Instance.new('Frame')
        healthFill.Name = 'HealthFill'
        healthFill.Size = UDim2.new(1, 0, 1, 0)
        healthFill.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
        healthFill.BorderSizePixel = 0
        healthFill.Parent = healthBg

        local fillCorner = Instance.new('UICorner')
        fillCorner.CornerRadius = UDim.new(0, 4)
        fillCorner.Parent = healthFill

        -- Keep the bar tied to the humanoid instead of polling it every frame.
        if humanoid then
            humanoid.HealthChanged:Connect(function(newHealth)
                if healthFill and healthFill.Parent then
                    local ratio = math.clamp(newHealth / humanoid.MaxHealth, 0, 1)
                    healthFill.Size = UDim2.new(ratio, 0, 1, 0)
                    healthFill.BackgroundColor3 = Color3.fromRGB(
                        255,
                        math.floor(50 + 200 * ratio),
                        50
                    )
                end
            end)
        end
    end

    -- The AI reads a comma-separated move list, which keeps the event script separate from the actual casting code.
    local jutsuList = selectRogueJutsu()
    local jutsuTypes = Instance.new('StringValue')
    jutsuTypes.Name = 'JutsuTypes'
    jutsuTypes.Value = table.concat(jutsuList, ',')
    jutsuTypes.Parent = rogue

    local offset = Vector3.new(math.random(-20, 20), 5, math.random(-20, 20))
    rogue:PivotTo(CFrame.new(spawnPosition + offset))

    local aiScript = rogue:FindFirstChild('AI')
    if aiScript and aiScript:IsA('Script') then
        aiScript.Disabled = true
    end

    local handleChar = rogue:FindFirstChild('Handle_Character')
    if handleChar and handleChar:IsA('Script') then
        handleChar.Disabled = true
    end

    -- The event owns setup/spawning, while RogueAI stays responsible for movement and fighting.
    local rogueAITemplate = SS.Assets:FindFirstChild('RogueAI')
    if rogueAITemplate and rogueAITemplate:IsA('Script') then
        local rogueAIClone = rogueAITemplate:Clone()
        rogueAIClone.Name = 'RogueAI'
        rogueAIClone.Disabled = true -- Will enable after parenting

        local skillLevel = Instance.new('IntValue')
        skillLevel.Name = 'Skill Level'
        skillLevel.Value = 2
        skillLevel.Parent = rogueAIClone

        rogueAIClone.Parent = rogue
    else
        warn('[RogueAttack] RogueAI template not found in ServerStorage.Assets!')
    end

    -- Parent last. Some of the existing character systems start listening as soon as something enters workspace.Alive.
    rogue.Parent = workspace.Alive

    local rogueAI = rogue:FindFirstChild('RogueAI')
    if rogueAI and rogueAI:IsA('Script') then
        rogueAI.Disabled = false
    end

    if effects:FindFirstChild('DontAttack') then
        effects.DontAttack.Value = rogue
    end

    if humanoid then
        humanoid.MaxHealth = ROGUE_HEALTH
        humanoid.Health = ROGUE_HEALTH
    end

    return rogue
end

-- Damage credit is tracked from health changes and only counts nearby players from the village being attacked.
local function trackDamageOnRogue(rogue, villageName)
    local humanoid = rogue:FindFirstChildOfClass('Humanoid')
    if not humanoid then return end

    local previousHealth = humanoid.Health

    humanoid.HealthChanged:Connect(function(newHealth)
        local damageDealt = previousHealth - newHealth
        previousHealth = newHealth

        -- Healing or health resets should never count backward against a player's event score.
        if damageDealt > 0 then
            local hrp = rogue:FindFirstChild('HumanoidRootPart')
            if hrp then
                local closestPlayer = nil
                local closestDist = 40
                for _, char in ipairs(workspace.Alive:GetChildren()) do
                    if char ~= rogue and char:FindFirstChild('HumanoidRootPart') then
                        local dist = (char.HumanoidRootPart.Position - hrp.Position).Magnitude
                        if dist < closestDist then
                            local player = Players:GetPlayerFromCharacter(char)
                            if player then
                                local data = SS.PlayerData:FindFirstChild(player.Name)
                                if data and data:FindFirstChild('Village') and data.Village.Value == villageName then
                                    closestDist = dist
                                    closestPlayer = player
                                end
                            end
                        end
                    end
                end
                if closestPlayer then
                    damageTracker[closestPlayer] = (damageTracker[closestPlayer] or 0) + damageDealt
                end
            end
        end
    end)
end

-- Sort once at the end of the event instead of maintaining a leaderboard every time damage changes.
local function getTopDefenders(villageName, count)
    local defenders = {}
    for player, damage in pairs(damageTracker) do
        if player and player.Parent == Players then
            local data = SS.PlayerData:FindFirstChild(player.Name)
            if data and data:FindFirstChild('Village') and data.Village.Value == villageName then
                table.insert(defenders, {player = player, damage = damage})
            end
        end
    end

    -- Damage is the ranking metric here, but the reward table is independent so placements are easy to tune.
    table.sort(defenders, function(a, b)
        return a.damage > b.damage
    end)

    local result = {}
    for i = 1, math.min(count, #defenders) do
        table.insert(result, defenders[i])
    end
    return result
end

-- Rewards respect the existing skill-point cap so the event cannot bypass normal progression rules.
local function rewardDefenders(villageName)
    local topDefenders = getTopDefenders(villageName, #REWARD_XP)

    if #topDefenders == 0 then
        RS.Events.Notify:FireAllClients('ROGUE ATTACK', 'Title')
        RS.Events.Notify:FireAllClients('No defenders stepped up for ' .. villageName .. '...', 'Description')
        return
    end

    RS.Events.Notify:FireAllClients('ROGUE ATTACK', 'Title')
    RS.Events.Notify:FireAllClients('The rogues attacking ' .. villageName .. ' have been defeated!', 'Description')

    task.wait(2)

    for i, defender in ipairs(topDefenders) do
        local player = defender.player
        local xpReward = REWARD_XP[i] or REWARD_XP[#REWARD_XP]
        local currencyReward = REWARD_CURRENCY[i] or REWARD_CURRENCY[#REWARD_CURRENCY]

        local data = SS.PlayerData:FindFirstChild(player.Name)
        if data then
            local skillPoints = data:FindFirstChild('SkillPoints')
            local spentSkillPoints = data:FindFirstChild('SpentSkillPoints')
            local rank = data:FindFirstChild('Rank')

            -- Rank acts as the progression ceiling in this project, so I compare earned + spent points against it.
            local totalSP = 0
            if skillPoints then totalSP = totalSP + skillPoints.Value end
            if spentSkillPoints then totalSP = totalSP + spentSkillPoints.Value end

            local maxSP = rank and rank.Value or 0

            if totalSP < maxSP then
                RewardEXP.MissionEXP(player, xpReward, currencyReward)

                RS.Events.Notify:FireAllClients('ROGUE ATTACK', 'Title')
                RS.Events.Notify:FireClient(player,
                    '#' .. i .. ' Defender: ' .. player.Name .. ' (+' .. xpReward .. ' XP)',
                    'Description')
            else
                if data:FindFirstChild('Currency') then
                    data.Currency.Value = data.Currency.Value + currencyReward
                end
                RS.Events.Notify:FireClient(player,
                    '#' .. i .. ' Defender! (SP cap reached - currency only)',
                    'Description')
            end
        end
    end
end

-- Shut the NPC scripts down before removing the models so their loops do not keep references alive.
local function cleanupRogues()
    for _, rogue in ipairs(spawnedRogues) do
        if rogue then
            local rogueAI = rogue:FindFirstChild('RogueAI')
            if rogueAI and rogueAI:IsA('Script') then
                rogueAI.Disabled = true
            end
            local aiScript = rogue:FindFirstChild('AI')
            if aiScript and aiScript:IsA('Script') then
                aiScript.Disabled = true
            end
            local handleChar = rogue:FindFirstChild('Handle_Character')
            if handleChar and handleChar:IsA('Script') then
                handleChar.Disabled = true
            end
            if rogue.Parent then
                rogue.Parent = nil
            end
        end
    end
    spawnedRogues = {}
end

-- Reset all event-owned state in one place so manual stops and normal finishes clean up the same way.
local function cleanupEvent(eventFolder)
    cleanupRogues()
    damageTracker = {}
    isActive = false
    if eventFolder and eventFolder.Parent then
        eventFolder.Parent = nil
    end
end

-- I recalculate this from the humanoids so a missed death connection cannot leave the event stuck.
local function countAliveRogues()
    local count = 0
    for _, rogue in ipairs(spawnedRogues) do
        if rogue and rogue.Parent then
            local hum = rogue:FindFirstChildOfClass('Humanoid')
            if hum and hum.Health > 0 then
                count = count + 1
            end
        end
    end
    return count
end

-- Only one copy of this event is allowed at a time; the scheduler and admin command both come through here.
local function runRogueAttack(allPlayers)
    if isActive then
        warn('[RogueAttack] Event already in progress, skipping.')
        return
    end

    isActive = true
    damageTracker = {}
    spawnedRogues = {}

    -- Pick the target first, then bail cleanly if that village is missing its anchor in the map.
    local targetVillage = selectRandomVillage()
    local kageChairPos = getVillageKageChairPosition(targetVillage)

    if not kageChairPos then
        warn('[RogueAttack] Could not find Kage Chair for ' .. targetVillage)
        isActive = false
        return
    end

    local eventFolder = Instance.new('Folder')
    eventFolder.Name = 'RogueAttack'
    eventFolder.Parent = SpawnFolder

    local activeMarker = Instance.new('Folder')
    activeMarker.Name = 'Active'
    activeMarker.Parent = eventFolder

    local villageValue = Instance.new('StringValue')
    villageValue.Name = 'TargetVillage'
    villageValue.Value = targetVillage
    villageValue.Parent = eventFolder

    RS.Events.Notify:FireAllClients('ROGUE ATTACK', 'Title')
    RS.Events.Notify:FireAllClients(
        'Rogues are attacking ' .. targetVillage .. ' Village! Defend your home!',
        'Description')

    -- Audio is optional so a missing sound does not stop the event from running.
    local eventSound = workspace:FindFirstChild('EventScroll')
    if eventSound then
        eventSound:Play()
    end

    task.wait(3)

    -- Small delays between clones avoid dumping every NPC setup task into the same frame.
    for i = 1, ROGUE_COUNT do
        local rogue = createRogueNPC(targetVillage, kageChairPos, i)
        if rogue then
            table.insert(spawnedRogues, rogue)
            trackDamageOnRogue(rogue, targetVillage)
        end
        task.wait(0.5)
    end

    RS.Events.Notify:FireAllClients('ROGUE ATTACK', 'Title')
    RS.Events.Notify:FireAllClients(
        #spawnedRogues .. ' rogue ninjas have infiltrated ' .. targetVillage .. '!',
        'Description')

    -- The loop checks once a second. There is no reason to run event-state checks every frame.
    local startTime = tick()
    local lastAnnouncement = 0

    repeat
        task.wait(1)

        local aliveRogues = countAliveRogues()
        local elapsed = tick() - startTime

        -- Status messages are spaced out so the event stays readable instead of constantly spamming the HUD.
        if elapsed - lastAnnouncement >= 60 and aliveRogues > 0 then
            lastAnnouncement = elapsed
            RS.Events.Notify:FireAllClients('ROGUE ATTACK', 'Title')
            RS.Events.Notify:FireAllClients(
                aliveRogues .. ' rogues remain in ' .. targetVillage .. '! (' ..
                math.floor(EVENT_TIMEOUT - elapsed) .. 's left)',
                'Description')
        end

        if not eventFolder:FindFirstChild('Active') then
            warn('[RogueAttack] Event manually stopped.')
            cleanupEvent(eventFolder)
            return
        end
    until aliveRogues == 0 or elapsed >= EVENT_TIMEOUT

    -- A win pays normal rewards; a timeout still gives partial credit for damage already done.
    if countAliveRogues() == 0 then
        task.wait(2)
        rewardDefenders(targetVillage)
    else
        RS.Events.Notify:FireAllClients('ROGUE ATTACK', 'Title')
        RS.Events.Notify:FireAllClients(
            'The rogues have escaped from ' .. targetVillage .. '...',
            'Description')

        -- Timeout rewards are intentionally reduced, but still use the same defender ranking.
        local topDefenders = getTopDefenders(targetVillage, #REWARD_XP)
        for i, defender in ipairs(topDefenders) do
            local player = defender.player
            local reducedXP = math.floor((REWARD_XP[i] or REWARD_XP[#REWARD_XP]) * 0.5)
            local reducedCurrency = math.floor((REWARD_CURRENCY[i] or REWARD_CURRENCY[#REWARD_CURRENCY]) * 0.5)
            local data = SS.PlayerData:FindFirstChild(player.Name)
            if data then
                local skillPoints = data:FindFirstChild('SkillPoints')
                local spentSkillPoints = data:FindFirstChild('SpentSkillPoints')
                local rank = data:FindFirstChild('Rank')
                local totalSP = 0
                if skillPoints then totalSP = totalSP + skillPoints.Value end
                if spentSkillPoints then totalSP = totalSP + spentSkillPoints.Value end
                local maxSP = rank and rank.Value or 0
                if totalSP < maxSP then
                    RewardEXP.MissionEXP(player, reducedXP, reducedCurrency)
                elseif data:FindFirstChild('Currency') then
                    data.Currency.Value = data.Currency.Value + reducedCurrency
                end
            end
        end
    end

    task.wait(3)
    cleanupEvent(eventFolder)
end

-- Both the automatic scheduler and the staff command use the same entry point.
script.Parent['Automated Events'].Event:Connect(function(eventType, allPlayers)
    if eventType == script.Name or eventType == 'ROGUES' then
        warn('[RogueAttack] Event triggered via: ' .. tostring(eventType))
        runRogueAttack(allPlayers)
    end
end)

warn('[RogueAttack] Script loaded.')
