local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer

-- load beacon: if you DON'T see this when you land in the instance, the script
-- is not being re-run after Matcha's environment restart (autoexecute issue).
pcall(function() print("[Archived] ===== SCRIPT START =====") end)
pcall(function() notify("Archived script loaded", "Archived", 3) end)

local noFall = {
    enabled = false,
    triggerSpeed = -45,
    safeSpeed = -8,
    delay = 0.03
}

local speedChanger = {
    enabled = false,
    speed = 24,
    delay = 0.08
}

local spaceBoost = {
    enabled = false,
    boostPower = 55,
    maxUpwardVelocity = 180,
    holdInterval = 0.08,
    held = false
}

local noFog = {
    enabled = false,
    delay = 0.25,
    saved = false,
    fogStart = nil,
    fogEnd = nil,
    atmosphereValues = {}
}

local weaponTweaks = {
    enabled = false,
    noM1Delay = false,
    noStaminaDrain = false,
    delay = 0.15,
    originalValues = setmetatable({}, { __mode = "k" })
}

local autoLibrary = {
    enabled = false,
    delay = 0.05,            -- loop interval; ~20 clicks/sec when spamming
    attachDistance = 4,      -- studs to sit behind the target
    heightOffset = 0,        -- vertical nudge when attaching
    aliveFolder = "Alive",   -- Workspace.Alive (confirmed)
    rankBy = "MaxHealth",    -- "MaxHealth" (stable, boss first) or "Health" (current HP)
    aimAtTarget = false,     -- also slide the cursor onto the NPC via WorldToScreen
    spamClick = true,

    -- entry sequence: TP to the library NPC, press E, wait, click dialogue
    entryPosition = Vector3.new(300.92, 64.50, -145.47), -- Library Room TP (next to NPC)
    tpSettleDelay = 0.5,     -- wait for the teleport to settle before pressing E
    uiCloseDelay = 4.0,      -- extra wait after TP so you can close the Matcha UI
    useStartPrompt = true,
    promptKey = 0x45,        -- E (Windows virtual-key code)
    promptHold = 0.12,       -- how long E is held (too short = game ignores it)
    promptWait = 2.0,        -- wait after pressing E before clicking (your 2s)
    dialogDelay = 2.0,       -- wait between the two dialogue clicks (double dialogue)
    dialog1X = 906, dialog1Y = 824,   -- first dialogue choice
    dialog2X = 906, dialog2Y = 824,   -- second dialogue choice
    idleDelay = 0.5,         -- pause while no NPC is alive

    -- ===== inside the teleported "Library" instance =====
    instancePlaceId = 99831550635699,                        -- the library instance PlaceId
    instanceLoadWait = 25,   -- wait this long after detecting the instance before part 2
    part1Enabled = false,     -- auto-run overworld part 1 (TP to NPC -> enter library)
    part2Enabled = false,     -- auto-run instance part 2 (dialogue -> equip -> farm)
    overworldPlaceId = nil,  -- optional: set your overworld PlaceId to restrict part 1
    instancePosition = Vector3.new(730.11, 525.69, 1011.97), -- start spot for part 2
    equipWait = 10.0,        -- wait before the final E that equips your weapon
    loadTimeout = 120,       -- max seconds to wait for the instance to load
    autoNoM1InInstance = true, -- auto-enable No M1 Delay once inside the instance

    -- ===== attach + grip =====
    attaching = true,        -- master attach switch (Stop Attaching button sets false)
    attachHeight = 4,        -- studs ABOVE the NPC (facing down so the hitbox hits)
    gripThreshold = 1,       -- when the target's HP is <= this, run the grip sequence
    gripMinMaxHealth = 0,    -- only grip targets whose MaxHealth >= this (0 = any mob)
    gripSafePosition = Vector3.new(4726.75, 338.90, -1208.81), -- <<< SET YOUR SAFE SPOT (or use the button)
    gripPickupKey = 0x56,    -- V  (pick up the body)
    gripDropKey = 0x56,      -- V  (drop the body)
    gripKey = 0x42,          -- B  (grip)
    gripPickupWait = 0.8,    -- after V pickup, before teleporting
    gripTpWait = 0.8,        -- after teleport, before dropping
    gripDropWait = 0.8,      -- after V drop, before B grip
    gripGripWait = 5.0,      -- wait while gripping before re-attaching (5s)

    -- ===== damage retreat (TP to safe for a moment when you take damage) =====
    damageRetreat = true,    -- when your HP drops, retreat to the safe spot
    damageRetreatTime = 1.0, -- seconds to stay safe after a hit
    damageThreshold = 1,     -- min HP lost to count as a hit (ignore tiny chip)
    retreatUnattachDelay = 2.0 -- unattach, wait this long, THEN teleport to safe
}

local animSpeed = {
    enabled = false,
    speed = 1.0,             -- 1 = normal, 2 = double speed, etc.
    delay = 0.1
}

local locations = {
    Subway = Vector3.new(64.72, 0.50, 714.79),
    Darius = Vector3.new(134.44, 30.57, 846.82),
    Hana = Vector3.new(299.76, 27.50, 564.97),
    Sentenza = Vector3.new(422.82, -7.50, 448.98),
    Workshop = Vector3.new(156.65, 64.50, 121.05),
    LibraryTP = Vector3.new(298.92, 64.50, -158.47),
    Docks = Vector3.new(-1240.93, -10.50, 870.84),
    MarkTower = Vector3.new(-981.13, 697.53, 1447.19),
    BasketBall = Vector3.new(1030.96, 27.59, 912.24),
    TowerOfAss = Vector3.new(732.88, 770.22, -77.86)
}

local function getAliveFolder()
    return Workspace:FindFirstChild("Alive")
end

local function getLocalCharacter()
    if LocalPlayer and LocalPlayer.Character then
        return LocalPlayer.Character
    end

    local alive = getAliveFolder()
    if alive and LocalPlayer and LocalPlayer.Name then
        local character = alive:FindFirstChild(LocalPlayer.Name)
        if character then
            return character
        end
    end

    if LocalPlayer and LocalPlayer.Name then
        return Workspace:FindFirstChild(LocalPlayer.Name)
    end

    return nil
end

local function getRootPart(character)
    if not character then
        return nil
    end

    for _, name in ipairs({"HumanoidRootPart", "Torso", "UpperTorso", "LowerTorso", "Head"}) do
        local part = character:FindFirstChild(name)
        if part then
            return part
        end
    end

    for _, object in ipairs(character:GetDescendants()) do
        local ok, hasPosition = pcall(function()
            return object.Position ~= nil
        end)

        if ok and hasPosition then
            return object
        end
    end

    return nil
end

local function getHumanoid(character)
    if not character then
        return nil
    end

    return character:FindFirstChildOfClass("Humanoid")
end

local function getWalkSpeedValue(character)
    if not character then
        return nil
    end

    for _, name in ipairs({"WalkSpeed", "Speed", "WS"}) do
        local value = character:FindFirstChild(name)
        if value then
            return value
        end
    end

    for _, object in ipairs(character:GetDescendants()) do
        if object.Name == "WalkSpeed" or object.Name == "Speed" or object.Name == "WS" then
            return object
        end
    end

    return nil
end

local function getRootVelocity(root)
    if not root then
        return nil
    end

    local ok, value = pcall(function()
        return root.AssemblyLinearVelocity
    end)

    if ok and value then
        return value
    end

    ok, value = pcall(function()
        return root.Velocity
    end)

    if ok and value then
        return value
    end

    return nil
end

local function setRootVelocity(root, velocity)
    if not root then
        return false
    end

    pcall(function()
        root.AssemblyLinearVelocity = velocity
    end)

    pcall(function()
        root.Velocity = velocity
    end)

    return true
end

local function getWeaponInfoFolder()
    return ReplicatedStorage:FindFirstChild("WeaponINFO")
end

local function getEquippedWeaponName(character)
    if not character then
        return nil
    end

    local weaponValue = character:FindFirstChild("Weapon")
    if weaponValue and weaponValue:IsA("StringValue") and weaponValue.Value ~= "" then
        return weaponValue.Value
    end

    local weaponInfo = getWeaponInfoFolder()
    for _, child in ipairs(character:GetChildren()) do
        local isCandidate = false
        pcall(function()
            isCandidate = child:IsA("Model") or child:IsA("Folder") or child:IsA("Tool")
        end)

        if isCandidate then
            if weaponInfo and weaponInfo:FindFirstChild(child.Name) then
                return child.Name
            end

            if child:FindFirstChild("WeaponState") or child:FindFirstChild("SwingSpeed") or child:FindFirstChild("M1Delay") then
                return child.Name
            end
        end
    end

    return nil
end

local function getWeaponRoots(character)
    local roots = {}
    local weaponName = getEquippedWeaponName(character)

    if not weaponName then
        return roots, nil
    end

    local weaponInfo = getWeaponInfoFolder()
    if weaponInfo then
        local info = weaponInfo:FindFirstChild(weaponName)
        if info then
            table.insert(roots, info)
        end
    end

    local liveWeapon = character and character:FindFirstChild(weaponName)
    if liveWeapon then
        table.insert(roots, liveWeapon)
    end

    return roots, weaponName
end

local function getNumberValueObjectsByName(root, valueName)
    local results = {}

    if not root then
        return results
    end

    local function consider(object)
        local isValue = false
        pcall(function()
            isValue = object:IsA("NumberValue") or object:IsA("IntValue")
        end)

        if isValue and object.Name == valueName then
            table.insert(results, object)
        end
    end

    consider(root)

    local ok, descendants = pcall(function()
        return root:GetDescendants()
    end)

    if ok and descendants then
        for _, object in ipairs(descendants) do
            consider(object)
        end
    end

    return results
end

local function cacheWeaponOriginalValue(object)
    if weaponTweaks.originalValues[object] ~= nil then
        return
    end

    local ok, value = pcall(function()
        return object.Value
    end)

    if ok then
        weaponTweaks.originalValues[object] = value
    end
end

local function setWeaponValue(object, value)
    cacheWeaponOriginalValue(object)
    pcall(function()
        object.Value = value
    end)
end

local function restoreWeaponValue(object)
    local original = weaponTweaks.originalValues[object]
    if original == nil then
        return
    end

    pcall(function()
        object.Value = original
    end)
end

local function applyWeaponTweaksToRoot(root)
    if not root then
        return false
    end

    local applied = false
    local targets = {
        { name = "TimeUntilHitbox", enabled = weaponTweaks.noM1Delay, value = 0 },
        { name = "SwingSpeed", enabled = weaponTweaks.noM1Delay, value = 1000 },
        { name = "M1Delay", enabled = weaponTweaks.noM1Delay, value = -1000 },
        { name = "Stamina", enabled = weaponTweaks.noStaminaDrain, value = 0 }
    }

    for _, target in ipairs(targets) do
        for _, object in ipairs(getNumberValueObjectsByName(root, target.name)) do
            applied = true
            if target.enabled then
                setWeaponValue(object, target.value)
            else
                restoreWeaponValue(object)
            end
        end
    end

    return applied
end

local function applyWeaponTweaks()
    local character = getLocalCharacter()
    local roots = getWeaponRoots(character)
    local applied = false

    for _, root in ipairs(roots) do
        applied = applyWeaponTweaksToRoot(root) or applied
    end

    return applied
end

local function startWeaponTweakLoop()
    if _G.matchaWeaponTweakLoopRunning then
        return
    end

    _G.matchaWeaponTweakLoopRunning = true

    spawn(function()
        while weaponTweaks.noM1Delay or weaponTweaks.noStaminaDrain do
            applyWeaponTweaks()
            task.wait(weaponTweaks.delay)
        end

        applyWeaponTweaks()
        _G.matchaWeaponTweakLoopRunning = false
    end)
end

local function isSpaceInput(input)
    local keyCode = input and input.KeyCode

    if Enum and Enum.KeyCode and keyCode == Enum.KeyCode.Space then
        return true
    end

    local keyName = tostring(keyCode)
    return keyName == "Space" or keyName == "Enum.KeyCode.Space"
end

local function boostSpaceOnce()
    local root = getRootPart(getLocalCharacter())
    if not root then
        return false
    end

    local velocity = getRootVelocity(root) or Vector3.new(0, 0, 0)
    local upwardVelocity = velocity.Y + spaceBoost.boostPower

    if upwardVelocity > spaceBoost.maxUpwardVelocity then
        upwardVelocity = spaceBoost.maxUpwardVelocity
    end

    return setRootVelocity(root, Vector3.new(velocity.X, upwardVelocity, velocity.Z))
end

local function startSpaceBoostLoop()
    if _G.matchaSpaceBoostLoopRunning then
        return
    end

    _G.matchaSpaceBoostLoopRunning = true

    spawn(function()
        while spaceBoost.enabled and spaceBoost.held do
            boostSpaceOnce()
            task.wait(spaceBoost.holdInterval)
        end

        _G.matchaSpaceBoostLoopRunning = false
    end)
end

local function softenFallOnce()
    local root = getRootPart(getLocalCharacter())
    if not root then
        return false
    end

    local velocity = getRootVelocity(root)
    if not velocity then
        return false
    end

    if velocity.Y < noFall.triggerSpeed then
        setRootVelocity(root, Vector3.new(velocity.X, noFall.safeSpeed, velocity.Z))
        return true
    end

    return false
end

local function startNoFallLoop()
    if _G.matchaNoFallLoopRunning then
        return
    end

    _G.matchaNoFallLoopRunning = true

    spawn(function()
        while noFall.enabled do
            softenFallOnce()
            task.wait(noFall.delay)
        end

        _G.matchaNoFallLoopRunning = false
    end)
end

local function applySpeedOnce()
    local character = getLocalCharacter()
    local humanoid = getHumanoid(character)
    local walkSpeedValue = getWalkSpeedValue(character)
    local changed = false

    if humanoid then
        local ok = pcall(function()
            humanoid.WalkSpeed = speedChanger.speed
        end)
        changed = changed or ok
    end

    if walkSpeedValue then
        local ok = pcall(function()
            walkSpeedValue.Value = speedChanger.speed
        end)
        changed = changed or ok
    end

    return changed
end

local function startSpeedLoop()
    if _G.matchaSpeedLoopRunning then
        return
    end

    _G.matchaSpeedLoopRunning = true

    spawn(function()
        while speedChanger.enabled do
            applySpeedOnce()
            task.wait(speedChanger.delay)
        end

        _G.matchaSpeedLoopRunning = false
    end)
end

local function setAllCharacterCollision(canCollide)
    local character = getLocalCharacter()
    if not character then
        return
    end

    for _, part in ipairs(character:GetDescendants()) do
        if part:IsA("BasePart") then
            pcall(function()
                part.CanCollide = canCollide
            end)
        end
    end
end

local function teleportTo(pos)
    local root = getRootPart(getLocalCharacter())
    if not root then
        return false
    end

    local ok = pcall(function()
        setRootVelocity(root, Vector3.new(0, 0, 0))
        root.Position = pos
    end)

    return ok
end

local function teleportReliable(pos, attempts, tolerance)
    -- right after loading the root can be briefly unmovable, so a single
    -- teleportTo silently fails. Retry until the character actually arrives.
    attempts = attempts or 25
    tolerance = tolerance or 8

    for _ = 1, attempts do
        teleportTo(pos)
        task.wait(0.2)

        local root = getRootPart(getLocalCharacter())
        if root then
            local ok, dist = pcall(function()
                return (root.Position - pos).Magnitude
            end)
            if ok and dist and dist <= tolerance then
                return true
            end
        end
    end

    return false
end

local function setPropertyIfPossible(object, property, value)
    if not object then
        return false
    end

    local ok = pcall(function()
        object[property] = value
    end)

    return ok
end

local function getSafeDescendants(root)
    if not root then
        return {}
    end

    local ok, descendants = pcall(function()
        return root:GetDescendants()
    end)

    if ok and descendants then
        return descendants
    end

    return {}
end

local function getPropertyIfPossible(object, property)
    if not object then
        return nil
    end

    local ok, value = pcall(function()
        return object[property]
    end)

    if ok then
        return value
    end

    return nil
end

local function isClass(object, className)
    local ok, result = pcall(function()
        return object:IsA(className)
    end)

    return ok and result
end

local function saveNoFogOriginals()
    if noFog.saved or not Lighting then
        return
    end

    noFog.fogStart = getPropertyIfPossible(Lighting, "FogStart")
    noFog.fogEnd = getPropertyIfPossible(Lighting, "FogEnd")

    for _, object in ipairs(getSafeDescendants(Lighting)) do
        if isClass(object, "Atmosphere") then
            noFog.atmosphereValues[object] = {
                Density = getPropertyIfPossible(object, "Density"),
                Haze = getPropertyIfPossible(object, "Haze"),
                Glare = getPropertyIfPossible(object, "Glare")
            }
        end
    end

    noFog.saved = true
end

local function applyNoFogOnce()
    if not Lighting then
        return false
    end

    saveNoFogOriginals()
    setPropertyIfPossible(Lighting, "FogStart", 0)
    setPropertyIfPossible(Lighting, "FogEnd", 1000000)

    for _, object in ipairs(getSafeDescendants(Lighting)) do
        if isClass(object, "Atmosphere") then
            if not noFog.atmosphereValues[object] then
                noFog.atmosphereValues[object] = {
                    Density = getPropertyIfPossible(object, "Density"),
                    Haze = getPropertyIfPossible(object, "Haze"),
                    Glare = getPropertyIfPossible(object, "Glare")
                }
            end

            setPropertyIfPossible(object, "Density", 0)
            setPropertyIfPossible(object, "Haze", 0)
            setPropertyIfPossible(object, "Glare", 0)
        end
    end

    return true
end

local function restoreNoFogOriginals()
    if not Lighting or not noFog.saved then
        return
    end

    if noFog.fogStart ~= nil then
        setPropertyIfPossible(Lighting, "FogStart", noFog.fogStart)
    end

    if noFog.fogEnd ~= nil then
        setPropertyIfPossible(Lighting, "FogEnd", noFog.fogEnd)
    end

    for atmosphere, values in pairs(noFog.atmosphereValues) do
        if atmosphere and values then
            if values.Density ~= nil then
                setPropertyIfPossible(atmosphere, "Density", values.Density)
            end

            if values.Haze ~= nil then
                setPropertyIfPossible(atmosphere, "Haze", values.Haze)
            end

            if values.Glare ~= nil then
                setPropertyIfPossible(atmosphere, "Glare", values.Glare)
            end
        end
    end
end

local function startNoFogLoop()
    if _G.matchaNoFogLoopRunning then
        return
    end

    _G.matchaNoFogLoopRunning = true

    spawn(function()
        while noFog.enabled do
            applyNoFogOnce()
            task.wait(noFog.delay)
        end

        _G.matchaNoFogLoopRunning = false
    end)
end

local function roundValue(value)
    return math.floor((tonumber(value) or 0) + 0.5)
end

-- ===========================================================================
-- Auto Library Farm  (uses Matcha input globals: mouse1click, mousemoveabs,
-- keypress/keyrelease with Windows VK codes, WorldToScreen, isrbxactive)
-- ===========================================================================

local function robloxIsActive()
    -- mouse/keyboard input only lands when Roblox is the focused window.
    local ok, active = pcall(function()
        return isrbxactive()
    end)

    if ok then
        return active
    end

    return true -- assume focused if the executor doesn't expose isrbxactive
end

local function waitForRobloxFocus(timeout)
    -- external executors steal focus; keypress/mouse only land in the focused
    -- window, so block until Roblox is the active window again.
    local elapsed = 0
    while not robloxIsActive() do
        if timeout and elapsed >= timeout then
            return false
        end
        task.wait(0.1)
        elapsed = elapsed + 0.1
    end
    return true
end

local function clickOnce()
    if not robloxIsActive() then
        return false
    end

    pcall(function()
        mouse1click()
    end)

    return true
end

local function clickAtScreenPos(x, y, nudge)
    if not robloxIsActive() then
        return false
    end

    x, y = roundValue(x), roundValue(y)
    nudge = nudge or 4

    -- the UI only registers a click after a real mouse-move delta. If the cursor
    -- is already on the target, moving to the same spot does nothing, so nudge
    -- it sideways a little (horizontal only, small) so we don't drift onto a
    -- different option, then back onto the target to fire the hover.
    pcall(function()
        mousemoveabs(x + nudge, y)
    end)
    task.wait(0.03)

    pcall(function()
        mousemoveabs(x, y)
    end)
    task.wait(0.03)

    pcall(function()
        mouse1click()
    end)

    return true
end

local function clickDialogue(x, y, times, nudge)
    -- the first click is often dropped, so click the same spot a few times.
    times = times or 3
    for _ = 1, times do
        clickAtScreenPos(x, y, nudge)
        task.wait(0.15)
    end
end

local function tapKey(vk, holdTime)
    pcall(function()
        keypress(vk)
    end)

    task.wait(holdTime or 0.05)

    pcall(function()
        keyrelease(vk)
    end)
end

local function aimAtPart(part)
    if not part then
        return false
    end

    local ok, screenPos, onScreen = pcall(function()
        return WorldToScreen(part.Position)
    end)

    if ok and screenPos and onScreen then
        pcall(function()
            mousemoveabs(roundValue(screenPos.X), roundValue(screenPos.Y))
        end)
        return true
    end

    return false
end

local function getPlayerNameSet()
    local names = {}

    local ok, players = pcall(function()
        return Players:GetPlayers()
    end)

    if ok and players then
        for _, plr in ipairs(players) do
            pcall(function()
                names[plr.Name] = true
            end)
        end
    end

    return names
end

local function getHealthInfo(npc)
    local humanoid = getHumanoid(npc)
    local health, maxHealth

    if humanoid then
        health = getPropertyIfPossible(humanoid, "Health")
        maxHealth = getPropertyIfPossible(humanoid, "MaxHealth")
    end

    if health == nil then -- fallback: a Health NumberValue on the model
        local hv = npc:FindFirstChild("Health")
        if hv then
            health = getPropertyIfPossible(hv, "Value")
        end
    end

    return health, maxHealth
end

local function getAliveNpcs()
    local results = {}
    local folder = Workspace:FindFirstChild(autoLibrary.aliveFolder) or Workspace
    local localChar = getLocalCharacter()
    local localName = LocalPlayer and LocalPlayer.Name
    local playerNames = getPlayerNameSet()

    for _, child in ipairs(folder:GetChildren()) do
        if child ~= localChar and child.Name ~= localName and not playerNames[child.Name] then
            if getHumanoid(child) or child:FindFirstChild("Health") then
                table.insert(results, child)
            end
        end
    end

    return results
end

local function pickStrongestNpc()
    local best, bestScore

    for _, npc in ipairs(getAliveNpcs()) do
        local health, maxHealth = getHealthInfo(npc)
        -- include HP 0/1 mobs so a dying boss stays the target until we grip it
        if health ~= nil then
            local score = (autoLibrary.rankBy == "Health") and health or (maxHealth or health)
            if not bestScore or score > bestScore then
                bestScore = score
                best = npc
            end
        end
    end

    return best
end

local function attachToTarget(npc)
    -- sit ABOVE the NPC and face straight down so the melee hitbox still lands.
    local myRoot = getRootPart(getLocalCharacter())
    local npcRoot = getRootPart(npc)

    if not myRoot or not npcRoot then
        return false
    end

    local ok = pcall(function()
        local above = npcRoot.Position + Vector3.new(0, autoLibrary.attachHeight, 0)
        myRoot.CFrame = CFrame.lookAt(above, npcRoot.Position) -- look down at the NPC
    end)

    if not ok then -- no CFrame support: position-only fallback
        pcall(function()
            myRoot.Position = npcRoot.Position + Vector3.new(0, autoLibrary.attachHeight, 0)
        end)
    end

    setRootVelocity(myRoot, Vector3.new(0, 0, 0))
    return true
end

local function gripSequence(npc)
    print("[Archived] Grip sequence")
    pcall(function() notify("Gripping target", "Auto Library", 3) end)

    local function tpToBody()
        local bodyRoot = getRootPart(npc)
        if bodyRoot then
            local ok, pos = pcall(function() return bodyRoot.Position end)
            if ok and pos then
                teleportTo(pos)
            end
        end
    end

    -- 1) TP onto the body, pick it up (V)
    tpToBody()
    task.wait(0.2)
    tapKey(autoLibrary.gripPickupKey, autoLibrary.promptHold)
    task.wait(autoLibrary.gripPickupWait)

    -- 2) TP to the safe spot immediately, drop the body (V)
    teleportTo(autoLibrary.gripSafePosition)
    task.wait(autoLibrary.gripTpWait)
    tapKey(autoLibrary.gripDropKey, autoLibrary.promptHold)
    task.wait(autoLibrary.gripDropWait)

    -- 3) TP back onto the body, grip it (B), then wait while gripping
    tpToBody()
    task.wait(0.2)
    tapKey(autoLibrary.gripKey, autoLibrary.promptHold)
    task.wait(autoLibrary.gripGripWait) -- ~5s while gripping; then loop re-attaches
end

local function enterLibrary()
    -- 1) teleport to the library NPC
    teleportTo(autoLibrary.entryPosition)
    task.wait(autoLibrary.tpSettleDelay)

    -- 2) input only lands when Roblox is the focused window. Matcha's UI steals
    --    focus, so give time to close the Matcha UI / click back into the game.
    pcall(function()
        notify("Close the Matcha UI - starting soon", "Auto Library", 4)
    end)
    task.wait(autoLibrary.uiCloseDelay)
    if not robloxIsActive() then
        waitForRobloxFocus() -- block until Roblox is active again
        task.wait(0.3)
    end

    -- 3) press E to open the dialogue
    if autoLibrary.useStartPrompt then
        tapKey(autoLibrary.promptKey, autoLibrary.promptHold) -- keypress(0x45)/keyrelease(0x45)
    end

    -- 4) wait for the dialogue to appear, then click through the options (3x each)
    task.wait(autoLibrary.promptWait) -- ~2 seconds
    clickDialogue(autoLibrary.dialog1X, autoLibrary.dialog1Y)
    task.wait(autoLibrary.dialogDelay)
    -- final click before the teleport is the flaky one: more clicks (small move)
    clickDialogue(autoLibrary.dialog2X, autoLibrary.dialog2Y, 6, 6)
    -- after this you get teleported to the other instance (handled next)
end

local function autoLibraryStep()
    if not autoLibrary.attaching then -- Stop Attaching pressed
        _G.matchaCurrentTarget = nil
        task.wait(autoLibrary.idleDelay)
        return
    end

    local npc = pickStrongestNpc()

    if not npc then -- no enemies right now: idle (wave spawning or cleared)
        _G.matchaCurrentTarget = nil
        _G.matchaGripDone = false
        task.wait(autoLibrary.idleDelay)
        return
    end

    local health, maxHealth = getHealthInfo(npc)
    local bossOk = (maxHealth or 0) >= autoLibrary.gripMinMaxHealth

    -- target is (near) dead: pick up -> TP to safe -> drop -> grip, once.
    if health and health <= autoLibrary.gripThreshold and bossOk then
        if not _G.matchaGripDone then
            _G.matchaGripDone = true
            _G.matchaCurrentTarget = nil   -- stop pinning
            autoLibrary.attaching = false  -- attach OFF while Roland is dead (no flying)
            gripSequence(npc)
            autoLibrary.attaching = true   -- resume attaching after the grip
        end
        return
    end

    _G.matchaGripDone = false
    -- the dedicated attach loop keeps us pinned above this target every frame
    _G.matchaCurrentTarget = npc

    if autoLibrary.aimAtTarget then
        aimAtPart(getRootPart(npc))
    end

    if autoLibrary.spamClick then
        clickOnce()
    end
end

local function startAutoLibraryLoop()
    if _G.matchaAutoLibraryLoopRunning then
        return
    end

    _G.matchaAutoLibraryLoopRunning = true

    spawn(function()
        while autoLibrary.enabled do
            autoLibraryStep()
            task.wait(autoLibrary.delay)
        end

        _G.matchaAutoLibraryLoopRunning = false
    end)
end

-- Dedicated high-frequency attach: re-pins you above the target every frame so
-- the M1 lunge (which drags you down into the body) is corrected instantly.
local function startAttachLoop()
    if _G.matchaAttachLoopRunning then
        return
    end
    _G.matchaAttachLoopRunning = true

    spawn(function()
        while autoLibrary.enabled do
            if autoLibrary.attaching and _G.matchaCurrentTarget then
                attachToTarget(_G.matchaCurrentTarget)
            end
            task.wait() -- every frame
        end
        _G.matchaAttachLoopRunning = false
    end)
end

local function getLocalHealth()
    local char = getLocalCharacter()
    local hum = getHumanoid(char)
    if hum then
        local h = getPropertyIfPossible(hum, "Health")
        if h ~= nil then
            return h
        end
    end
    if char then
        local hv = char:FindFirstChild("Health")
        if hv then
            return getPropertyIfPossible(hv, "Value")
        end
    end
    return nil
end

-- When your HP drops (you got hit): stop attacking, TP to safe for a moment,
-- then resume. Server-authoritative HP is a rock-solid "I'm in danger" signal.
local function startDamageWatcher()
    if _G.matchaDamageWatcherRunning then
        return
    end
    _G.matchaDamageWatcherRunning = true

    spawn(function()
        local lastHealth = getLocalHealth()

        while autoLibrary.enabled do
            local hp = getLocalHealth()

            -- don't interrupt a grip; only react to real damage (a drop)
            if autoLibrary.damageRetreat and not _G.matchaGripDone
                and hp and lastHealth and hp < (lastHealth - autoLibrary.damageThreshold) then

                print("[Archived] Took damage -> retreating to safe spot")
                pcall(function() notify("Hit - retreating", "Auto Library", 1) end)

                -- 1) unattach first, 2) wait, 3) THEN teleport to safe
                autoLibrary.attaching = false
                _G.matchaCurrentTarget = nil
                task.wait(autoLibrary.retreatUnattachDelay) -- ~2s before TP
                teleportTo(autoLibrary.gripSafePosition)
                task.wait(autoLibrary.damageRetreatTime)
                autoLibrary.attaching = true

                lastHealth = getLocalHealth() -- refresh so we don't re-trigger
            else
                if hp then
                    lastHealth = hp
                end
            end

            task.wait(0.1)
        end
        _G.matchaDamageWatcherRunning = false
    end)
end

-- ===== teleport / instance handling =====
local function getJobId()
    return getPropertyIfPossible(game, "JobId")
end

local function getPlaceId()
    return getPropertyIfPossible(game, "PlaceId")
end

local function inLibraryInstance()
    local pid = getPlaceId()
    return pid ~= nil and tonumber(pid) == tonumber(autoLibrary.instancePlaceId)
end

local function isGameLoaded()
    local ok, loaded = pcall(function()
        return game:IsLoaded()
    end)
    if ok then
        return loaded
    end
    return true
end

local function refreshServices()
    -- after a server teleport the old DataModel is destroyed; re-resolve the
    -- service references our helpers rely on so they point at the new instance.
    pcall(function()
        LocalPlayer = game:GetService("Players").LocalPlayer
        Workspace = game:GetService("Workspace")
    end)
end

local function waitForLibraryInstance(prevJobId, prevPlaceId, timeout)
    -- block until we are in a new server (ideally the known library PlaceId).
    local elapsed = 0
    while true do
        local jid = getJobId()
        local pid = getPlaceId()
        local inInstance = pid and pid == autoLibrary.instancePlaceId
        local changed = inInstance or (jid and jid ~= prevJobId) or (pid and pid ~= prevPlaceId)

        if changed then
            refreshServices()
            return true
        end

        if timeout and elapsed >= timeout then
            return false
        end
        task.wait(0.5)
        elapsed = elapsed + 0.5
    end
end

local function ensureFocus()
    if not robloxIsActive() then
        pcall(function()
            notify("Click back into Roblox - waiting for focus", "Auto Library", 4)
        end)
        waitForRobloxFocus()
        task.wait(0.3)
    end
end

local function runLibraryInstance()
    refreshServices()
    autoLibrary.enabled = true -- ensure the fight loop is allowed to run

    -- 1) TELEPORT FIRST -- must be the very first action. Retry until the
    --    character actually arrives (the root can be unmovable right after load).
    print("[Archived] Part 2: teleporting to start spot")
    pcall(function() notify("Teleporting to start", "Auto Library", 3) end)
    local tpOk = teleportReliable(autoLibrary.instancePosition)
    print("[Archived] Teleport success = " .. tostring(tpOk))
    task.wait(autoLibrary.tpSettleDelay)

    -- make sure Roblox is focused before any key/mouse input
    ensureFocus()

    -- auto-enable No M1 Delay for the instance (you asked for this)
    if autoLibrary.autoNoM1InInstance then
        weaponTweaks.noM1Delay = true
        applyWeaponTweaks()
        startWeaponTweakLoop()
    end

    -- 2) E -> 2s -> click -> 2s -> click -> 2s  (each click 3x)
    tapKey(autoLibrary.promptKey, autoLibrary.promptHold)
    task.wait(2)
    clickDialogue(autoLibrary.dialog1X, autoLibrary.dialog1Y)
    task.wait(2)
    clickDialogue(autoLibrary.dialog1X, autoLibrary.dialog1Y)
    task.wait(2)

    -- 3) E again -> 2s -> click -> ~10s -> TELEPORT -> E (equip weapon) -> fight
    tapKey(autoLibrary.promptKey, autoLibrary.promptHold)
    task.wait(2)
    clickDialogue(autoLibrary.dialog1X, autoLibrary.dialog1Y)
    task.wait(autoLibrary.equipWait) -- ~10s

    -- the intro dialogue/cutscene can move you, so teleport back BEFORE equipping
    teleportTo(autoLibrary.instancePosition)
    task.wait(autoLibrary.tpSettleDelay)

    tapKey(autoLibrary.promptKey, autoLibrary.promptHold) -- equip weapon

    task.wait(0.5)
    startAttachLoop()      -- keep us pinned above the target every frame
    startDamageWatcher()   -- retreat to safe for a moment when you take damage
    startAutoLibraryLoop() -- pick target + spam M1 + grip on kill
end

-- Single always-on controller. It looks at the current PlaceId every second and
-- runs whichever part fits: overworld -> part 1, instance -> part 2. Because the
-- Matcha environment restarts on every teleport, this is re-created fresh each
-- load (via autoexecute) and simply does the right thing for wherever you are.
local function startFarmController()
    if _G.matchaFarmControllerRunning then
        return
    end
    _G.matchaFarmControllerRunning = true

    print("[Archived] Farm controller started. PlaceId = " .. tostring(getPlaceId()))

    spawn(function()
        task.wait(3) -- small grace on initial load

        while true do
            if inLibraryInstance() then
                -- ===== inside the library instance: PART 2 =====
                _G.matchaPart1Ran = false -- re-arm part 1 for when we return

                if not _G.matchaInstanceNotified then
                    _G.matchaInstanceNotified = true
                    print("[Archived] In library instance (PlaceId " .. tostring(getPlaceId()) .. ")")
                    pcall(function()
                        notify("In library instance", "Auto Library", 4)
                    end)
                end

                if autoLibrary.part2Enabled and not _G.matchaPart2Ran then
                    _G.matchaPart2Ran = true

                    for i = autoLibrary.instanceLoadWait, 1, -1 do
                        pcall(function()
                            notify("Part 2 starting in " .. tostring(i) .. "s", "Auto Library", 1)
                        end)
                        task.wait(1)
                    end

                    if inLibraryInstance() then
                        print("[Archived] Running part 2")
                        runLibraryInstance()
                    end
                end
            else
                -- ===== overworld (not the instance): PART 1 =====
                _G.matchaPart2Ran = false        -- re-arm part 2 for next instance visit
                _G.matchaInstanceNotified = false

                local placeOk = (autoLibrary.overworldPlaceId == nil)
                    or (tonumber(getPlaceId()) == tonumber(autoLibrary.overworldPlaceId))
                local haveChar = getRootPart(getLocalCharacter()) ~= nil

                if autoLibrary.part1Enabled and placeOk and haveChar and not _G.matchaPart1Ran then
                    _G.matchaPart1Ran = true
                    print("[Archived] In overworld -> running part 1")
                    spawn(function()
                        enterLibrary()
                    end)
                end
            end

            task.wait(1)
        end
    end)
end

local function getPlayingTracks(humanoid)
    local tracks = {}

    if not humanoid then
        return tracks
    end

    pcall(function()
        for _, t in ipairs(humanoid:GetPlayingAnimationTracks()) do
            table.insert(tracks, t)
        end
    end)

    local animator = humanoid:FindFirstChildOfClass("Animator")
    if animator then
        pcall(function()
            for _, t in ipairs(animator:GetPlayingAnimationTracks()) do
                table.insert(tracks, t)
            end
        end)
    end

    return tracks
end

local function applyAnimSpeedOnce()
    local hum = getHumanoid(getLocalCharacter())

    for _, t in ipairs(getPlayingTracks(hum)) do
        pcall(function()
            t:AdjustSpeed(animSpeed.speed)
        end)
    end
end

local function startAnimSpeedLoop()
    if _G.matchaAnimSpeedLoopRunning then
        return
    end

    _G.matchaAnimSpeedLoopRunning = true

    spawn(function()
        while animSpeed.enabled do
            applyAnimSpeedOnce()
            task.wait(animSpeed.delay)
        end

        _G.matchaAnimSpeedLoopRunning = false
    end)
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed or not spaceBoost.enabled or not isSpaceInput(input) then
        return
    end

    spaceBoost.held = true
    boostSpaceOnce()
    startSpaceBoostLoop()
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if gameProcessed or not isSpaceInput(input) then
        return
    end

    spaceBoost.held = false
end)

print("[Archived] Script loaded. PlaceId = " .. tostring(getPlaceId()) .. " | inInstance = " .. tostring(inLibraryInstance()))
startFarmController() -- always-on: drives part 1 / part 2 by current PlaceId

if not UI then
    return
end

UI.AddTab("Archived", function(tab)
    local movement = tab:Section("Movement", "Left", {"Main"}, 460)
    if movement.page == 0 then
        movement:Toggle("pm_noclip", "Noclip", false, function(value)
            _G.matchaNoclip = value
            if value then
                spawn(function()
                    while _G.matchaNoclip do
                        setAllCharacterCollision(false)
                        task.wait(0.1)
                    end
                end)
            else
                setAllCharacterCollision(true)
            end
        end)

        movement:Toggle("pm_nofall", "No Fall", false, function(value)
            noFall.enabled = value
            if value then
                startNoFallLoop()
            end
        end)

        movement:SliderInt("pm_fall_trigger", "Fall Trigger", 40, 140, math.abs(noFall.triggerSpeed), function(value)
            noFall.triggerSpeed = -math.abs(roundValue(value))
        end)

        movement:SliderInt("pm_fall_cap", "Fall Cap", 1, 70, math.abs(noFall.safeSpeed), function(value)
            noFall.safeSpeed = -math.abs(roundValue(value))
        end)

        movement:Button("Soften Fall Now", function()
            softenFallOnce()
        end)

        movement:Toggle("pm_speed_enabled", "Speed", false, function(value)
            speedChanger.enabled = value
            if value then
                applySpeedOnce()
                startSpeedLoop()
            end
        end)

        movement:SliderInt("pm_speed", "Walk Speed", 16, 250, speedChanger.speed, function(value)
            speedChanger.speed = roundValue(value)
            if speedChanger.enabled then
                applySpeedOnce()
            end
        end)

        movement:Toggle("pm_space_boost", "Space Boost", false, function(value)
            spaceBoost.enabled = value
            if not value then
                spaceBoost.held = false
            end
        end)

        movement:SliderInt("pm_space_boost_power", "Space Boost Power", 10, 150, spaceBoost.boostPower, function(value)
            spaceBoost.boostPower = roundValue(value)
        end)

        movement:SliderInt("pm_space_boost_cap", "Space Boost Cap", 40, 350, spaceBoost.maxUpwardVelocity, function(value)
            spaceBoost.maxUpwardVelocity = roundValue(value)
        end)

        movement:Toggle("pm_no_fog", "No Fog", false, function(value)
            noFog.enabled = value
            if value then
                applyNoFogOnce()
                startNoFogLoop()
            else
                restoreNoFogOriginals()
            end
        end)

        movement:Toggle("pm_no_m1_delay", "No M1 Delay", false, function(value)
            weaponTweaks.noM1Delay = value
            applyWeaponTweaks()
            if value then
                startWeaponTweakLoop()
            end
        end)

        movement:Toggle("pm_no_stamina_drain", "No Stamina Drain", false, function(value)
            weaponTweaks.noStaminaDrain = value
            applyWeaponTweaks()
            if value then
                startWeaponTweakLoop()
            end
        end)
    end

    local library = tab:Section("Library", "Right", {"Main"}, 460)
    if library.page == 0 then
        library:Toggle("lib_part1", "Auto Part 1 (Overworld)", autoLibrary.part1Enabled, function(value)
            autoLibrary.part1Enabled = value
        end)

        library:Toggle("lib_part2", "Auto Part 2 (Instance)", autoLibrary.part2Enabled, function(value)
            autoLibrary.part2Enabled = value
        end)

        library:Button("Stop Attaching", function()
            autoLibrary.attaching = false
            pcall(function() notify("Attaching stopped", "Auto Library", 3) end)
        end)

        library:Button("Resume Attaching", function()
            autoLibrary.attaching = true
            pcall(function() notify("Attaching resumed", "Auto Library", 3) end)
        end)

        -- prints your current world position (and PlaceId/JobId) to the console
        library:SliderInt("lib_attach_height", "Attach Height (above)", 1, 20, autoLibrary.attachHeight, function(value)
            autoLibrary.attachHeight = roundValue(value)
        end)

        library:Toggle("lib_dmgretreat", "Damage Retreat (TP safe when hit)", autoLibrary.damageRetreat, function(value)
            autoLibrary.damageRetreat = value
        end)

        library:SliderInt("lib_dmgtime", "Retreat Time (x100ms)", 1, 30, roundValue(autoLibrary.damageRetreatTime * 10), function(value)
            autoLibrary.damageRetreatTime = roundValue(value) / 10
        end)

        library:Button("Enter Library Now", function()
            spawn(function()
                enterLibrary()
            end)
        end)

        library:Button("Run Instance Sequence (manual)", function()
            autoLibrary.enabled = true
            spawn(function()
                runLibraryInstance()
            end)
        end)

        library:Toggle("lib_spam", "Spam Click", true, function(value)
            autoLibrary.spamClick = value
        end)

        library:Toggle("lib_aim", "Aim Cursor At Target", false, function(value)
            autoLibrary.aimAtTarget = value
        end)

        library:Toggle("lib_prompt", "Press E To Start Run", true, function(value)
            autoLibrary.useStartPrompt = value
        end)

        library:SliderInt("lib_dist", "Attach Distance", 1, 20, autoLibrary.attachDistance, function(value)
            autoLibrary.attachDistance = roundValue(value)
        end)

        library:SliderInt("lib_height", "Height Offset", -10, 10, autoLibrary.heightOffset, function(value)
            autoLibrary.heightOffset = roundValue(value)
        end)

        library:SliderInt("lib_rate", "Click Delay (ms)", 20, 500, roundValue(autoLibrary.delay * 1000), function(value)
            autoLibrary.delay = roundValue(value) / 1000
        end)

        library:SliderInt("lib_d1x", "Dialogue 1 X", 0, 1920, autoLibrary.dialog1X, function(value)
            autoLibrary.dialog1X = roundValue(value)
        end)

        library:SliderInt("lib_d1y", "Dialogue 1 Y", 0, 1080, autoLibrary.dialog1Y, function(value)
            autoLibrary.dialog1Y = roundValue(value)
        end)

        library:SliderInt("lib_d2x", "Dialogue 2 X", 0, 1920, autoLibrary.dialog2X, function(value)
            autoLibrary.dialog2X = roundValue(value)
        end)

        library:SliderInt("lib_d2y", "Dialogue 2 Y", 0, 1080, autoLibrary.dialog2Y, function(value)
            autoLibrary.dialog2Y = roundValue(value)
        end)

        library:Button("Test Dialogue Clicks", function()
            clickAtScreenPos(autoLibrary.dialog1X, autoLibrary.dialog1Y)
            task.wait(autoLibrary.dialogDelay)
            clickAtScreenPos(autoLibrary.dialog2X, autoLibrary.dialog2Y)
        end)
    end
end)

UI.AddTab("Archived Teleports", function(tab)
    local teleport = tab:Section("Teleports", "Left", {"Places"}, 460)
    if teleport.page == 0 then
        teleport:Button("Subway", function() teleportTo(locations.Subway) end)
        teleport:Button("Darius", function() teleportTo(locations.Darius) end)
        teleport:Button("Hana", function() teleportTo(locations.Hana) end)
        teleport:Button("Sentenza", function() teleportTo(locations.Sentenza) end)
        teleport:Button("Workshop", function() teleportTo(locations.Workshop) end)
        teleport:Button("Library TP", function() teleportTo(locations.LibraryTP) end)
        teleport:Button("Mark Tower", function() teleportTo(locations.MarkTower) end)
        teleport:Button("BasketBall", function() teleportTo(locations.BasketBall) end)
        teleport:Button("TowerOfAss", function() teleportTo(locations.TowerOfAss) end)
    end
end)
