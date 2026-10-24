--[[
    BLOX FRUITS SECRET QUEST TRACKER
    Version: 1.0
    Usage:  loadstring(game:HttpGet("YOUR_RAW_GITHUB_URL"))()
    
    Notes:
      - Works in executors that support writefile/readfile (Delta, Solara, Xeno, etc.)
      - Falls back to in-memory (no save) if file API is unavailable
      - Prints a clean checklist to the console
--]]

local SAVE_FILE = "BF_SecretQuestTracker.json"

-- ============================================================
-- MASTER QUEST LIST (40 total)
-- ============================================================
local QUESTS = {
    ["Middle Town"]     = { "Early Access", "Lookout Duty", "X Marks the Spot", "The Finale" },
    ["Jungle"]          = { "Zipline Repair", "The Thieving Monkey", "Awakened Gorilla King" },
    ["Pirate Village"]  = { "Free the Windmill", "Tavern Brawl", "Awakened Chef" },
    ["Desert"]          = { "Save Hasan", "Rune Cleanup", "Prickly Harvest" },
    ["Frozen Village"]  = { "Breaking the Ice", "Build a Snowman", "Awakened Yeti" },
    ["Marine Fortress"] = { "Raise the Flag", "Pirate Raid", "Awakened Vice Admiral" },
    ["Lower Skylands"]  = { "Missing Lightning Bolt", "Return the Treasure", "Unexpected Guest" },
    ["Prison"]          = { "Prison Break", "Mysterious Key", "Awakened Warden" },
    ["Colosseum"]       = { "Gladiatorial Combat", "Previous Champions", "Crowded Colosseum" },
    ["Magma Village"]   = { "Magma Nest", "Magma Ore Extraction", "Awakened Magma General" },
    ["Underwater City"] = { "Ride the Bubble", "Scattered Light", "Awakened Fishman Lord" },
    ["Upper Skylands"]  = { "The Old Temple", "Awakened Sky Warlord", "Awakened Lightning God" },
    ["Fountain City"]   = { "Pipe Repairs", "Sewer Gang", "Awakened Cyborg" },
}

-- Order of islands for display
local ISLAND_ORDER = {
    "Middle Town","Jungle","Pirate Village","Desert","Frozen Village",
    "Marine Fortress","Lower Skylands","Prison","Colosseum",
    "Magma Village","Underwater City","Upper Skylands","Fountain City",
}

-- Your current progress (29/40) — applied on first run only
local DEFAULT_DONE = {
    ["Middle Town"]     = { "Early Access", "Lookout Duty" },
    ["Jungle"]          = { "Zipline Repair", "Awakened Gorilla King" },
    ["Pirate Village"]  = { "Free the Windmill", "Tavern Brawl", "Awakened Chef" },
    ["Desert"]          = { "Save Hasan", "Rune Cleanup" },
    ["Frozen Village"]  = { "Breaking the Ice", "Awakened Yeti" },
    ["Marine Fortress"] = { "Raise the Flag" },
    ["Lower Skylands"]  = { "Missing Lightning Bolt" },
    ["Prison"]          = { "Prison Break", "Mysterious Key", "Awakened Warden" },
    ["Colosseum"]       = { "Gladiatorial Combat", "Previous Champions" },
    ["Magma Village"]   = { "Magma Nest", "Magma Ore Extraction", "Awakened Magma General" },
    ["Underwater City"] = { "Ride the Bubble", "Scattered Light", "Awakened Fishman Lord" },
    ["Upper Skylands"]  = { "The Old Temple", "Awakened Sky Warlord", "Awakened Lightning God" },
    ["Fountain City"]   = { "Pipe Repairs", "Sewer Gang" },
}

-- ============================================================
-- TINY JSON ENCODER / DECODER (so we don't depend on HttpService)
-- ============================================================
local function encodeJSON(t)
    local function esc(s)
        return '"' .. s:gsub('\\','\\\\'):gsub('"','\\"'):gsub('\n','\\n') .. '"'
    end
    local function enc(v)
        local ty = type(v)
        if ty == "table" then
            local isArray = true
            local n = 0
            for k in pairs(v) do
                if type(k) ~= "number" then isArray = false break end
                n = math.max(n, k)
            end
            if isArray and n > 0 then
                local out = {}
                for i = 1, n do out[#out+1] = enc(v[i]) end
                return "[" .. table.concat(out, ",") .. "]"
            else
                local out = {}
                for k, val in pairs(v) do
                    out[#out+1] = esc(tostring(k)) .. ":" .. enc(val)
                end
                return "{" .. table.concat(out, ",") .. "}"
            end
        elseif ty == "string" then
            return esc(v)
        elseif ty == "number" then
            return tostring(v)
        elseif ty == "boolean" then
            return tostring(v)
        elseif ty == "nil" then
            return "null"
        end
        return "null"
    end
    return enc(t)
end

-- Minimal JSON decoder (handles our own output: nested objects of true/false)
local function decodeJSON(str)
    local pos = 1
    local function skipWS()
        while pos <= #str and str:sub(pos,pos):match("%s") do pos = pos + 1 end
    end
    local parseValue
    local function parseString()
        local quote = str:sub(pos,pos)
        assert(quote == '"', "expected string at " .. pos)
        pos = pos + 1
        local buf = {}
        while pos <= #str do
            local c = str:sub(pos,pos)
            if c == '\\' then
                local nxt = str:sub(pos+1,pos+1)
                if nxt == '"' then buf[#buf+1] = '"'
                elseif nxt == '\\' then buf[#buf+1] = '\\'
                elseif nxt == 'n' then buf[#buf+1] = '\n'
                else buf[#buf+1] = nxt end
                pos = pos + 2
            elseif c == '"' then
                pos = pos + 1
                return table.concat(buf)
            else
                buf[#buf+1] = c
                pos = pos + 1
            end
        end
        error("unterminated string")
    end
    local function parseNumber()
        local s = str:match("^-?%d+%.?%d*[eE]?[-+]?%d*", pos)
        if not s then error("bad number at " .. pos) end
        pos = pos + #s
        return tonumber(s)
    end
    local function parseObject()
        pos = pos + 1 -- {
        local obj = {}
        skipWS()
        if str:sub(pos,pos) == '}' then pos = pos + 1 return obj end
        while true do
            skipWS()
            local key = parseString()
            skipWS()
            assert(str:sub(pos,pos) == ':', "expected :")
            pos = pos + 1
            skipWS()
            obj[key] = parseValue()
            skipWS()
            local c = str:sub(pos,pos)
            if c == ',' then pos = pos + 1
            elseif c == '}' then pos = pos + 1 break
            else error("expected , or } got " .. c) end
        end
        return obj
    end
    local function parseArray()
        pos = pos + 1 -- [
        local arr = {}
        skipWS()
        if str:sub(pos,pos) == ']' then pos = pos + 1 return arr end
        while true do
            skipWS()
            arr[#arr+1] = parseValue()
            skipWS()
            local c = str:sub(pos,pos)
            if c == ',' then pos = pos + 1
            elseif c == ']' then pos = pos + 1 break
            else error("expected , or ]") end
        end
        return arr
    end
    function parseValue()
        skipWS()
        local c = str:sub(pos,pos)
        if c == '{' then return parseObject() end
        if c == '[' then return parseArray() end
        if c == '"' then return parseString() end
        if c == 't' then pos = pos + 4 return true end
        if c == 'f' then pos = pos + 5 return false end
        if c == 'n' then pos = pos + 4 return nil end
        return parseNumber()
    end
    return parseValue()
end

-- ============================================================
-- STATE MANAGEMENT
-- ============================================================
local state = {}

local function buildDefault()
    local s = {}
    for island in pairs(QUESTS) do s[island] = {} end
    for island, list in pairs(DEFAULT_DONE) do
        for _, q in ipairs(list) do s[island][q] = true end
    end
    return s
end

local function load()
    if typeof and typeof(writefile) == "function" and typeof(readfile) == "function" then
        local ok, data = pcall(readfile, SAVE_FILE)
        if ok and data and #data > 0 then
            local ok2, parsed = pcall(decodeJSON, data)
            if ok2 and type(parsed) == "table" then
                -- ensure every island exists
                for island in pairs(QUESTS) do
                    parsed[island] = parsed[island] or {}
                end
                return parsed
            end
        end
    end
    return buildDefault()
end

local function save()
    if typeof and typeof(writefile) == "function" then
        pcall(writefile, SAVE_FILE, encodeJSON(state))
    end
end

local function countDone()
    local total = 0
    for island, quests in pairs(QUESTS) do
        for _, q in ipairs(quests) do
            if state[island] and state[island][q] then total = total + 1 end
        end
    end
    return total
end

-- ============================================================
-- PUBLIC API
-- ============================================================
local Tracker = {}

function Tracker.mark(island, questName, completed)
    if not QUESTS[island] then
        warn("[Tracker] Unknown island: " .. tostring(island)); return false
    end
    local valid = false
    for _, q in ipairs(QUESTS[island]) do
        if q == questName then valid = true break end
    end
    if not valid then
        warn("[Tracker] Unknown quest: " .. tostring(questName)); return false
    end
    state[island] = state[island] or {}
    state[island][questName] = (completed ~= false) and true or nil
    save()
    return true
end

function Tracker.unmark(island, questName)
    return Tracker.mark(island, questName, false)
end

function Tracker.print()
    print("=============================================")
    print("  BLOX FRUITS SECRET QUEST TRACKER")
    print("=============================================")
    for _, island in ipairs(ISLAND_ORDER) do
        print("\n" .. island)
        for _, q in ipairs(QUESTS[island]) do
            local done = state[island] and state[island][q]
            print("  " .. (done and "[x]" or "[ ]") .. " " .. q)
        end
    end
    print("\n---------------------------------------------")
    print(string.format("  Progress: %d/%d", countDone(), 40))
    print("=============================================")
end

function Tracker.reset()
    state = buildDefault()
    save()
    print("[Tracker] Reset to default 29/40 progress.")
end

function Tracker.getState()
    return state
end

function Tracker.getProgress()
    return countDone(), 40
end

-- ============================================================
-- BOOT
-- ============================================================
state = load()
Tracker.print()

print("\n[Tracker] Commands:")
print("  _G.BFTracker.print()                                  -- print checklist")
print("  _G.BFTracker.mark(\"Middle Town\", \"X Marks the Spot\")   -- mark done")
print("  _G.BFTracker.unmark(\"Desert\", \"Prickly Harvest\")       -- unmark")
print("  _G.BFTracker.reset()                                  -- reset to default")

_G.BFTracker = Tracker
