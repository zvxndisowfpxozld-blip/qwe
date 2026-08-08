```
script_name("Advance RVANKA")
script_version("3.4")
script_author("Zhenya Prigozhin | blast.hk")
script_description("ACTIVATION -> /prez (inCar) | /prz [id] | /bps (Bypass)")

local sampev = require("lib.samp.events")
local ffi = require("ffi")
local BYPASS = true
local IS_ATTACKING = false

function main()
    while not isSampAvailable() do wait(0) end
    sampAddChatMessage("[AdvRvanka] /prez - порвать жопы ерпешников в машинах", -1)
    sampAddChatMessage("[AdvRvanka] /prz [id] - порвать конкретного ерпешника (даже на земле)", -1)
    sampAddChatMessage("[AdvRvanka] /bps - чтобы не кикало (включен)", -1)
    
    sampRegisterChatCommand("prez", function()
        if not isCharOnFoot(PLAYER_PED) then
            massAttack()
        else
            sampAddChatMessage("[AdvRvanka] Для набутыливания нужен ТС", -1)
        end
    end)

    sampRegisterChatCommand("prz", function(args)
        local targetId = tonumber(args)
        if not targetId then
            sampAddChatMessage("[AdvRvanka] Используй: /prz [id игрока]", -1)
            return
        end
        
        if not sampIsPlayerConnected(targetId) then
            sampAddChatMessage("[AdvRvanka] Игрок с ID " .. targetId .. " не найден", -1)
            return
        end
        
        if targetId == sampGetPlayerIdByCharHandle(PLAYER_PED) then
            sampAddChatMessage("[AdvRvanka] Нельзя бутылить самого себя!", -1)
            return
        end
        
        attackPlayerById(targetId)
    end)

    sampRegisterChatCommand('bps', function()
        BYPASS = not BYPASS
        sampAddChatMessage("[AdvRvanka] Bypass " .. (BYPASS and "включен" or "выключен"), -1)
    end)

    while true do wait(0) end
end

function sampev.onSendPlayerSync(data)
    if BYPASS then
        sendSpectator(data.position.x, data.position.y, data.position.z, data.keysData)
    end
end

function sampev.onSendVehicleSync(data)
    if IS_ATTACKING then return false end
end

function onSendPacket(ID) 
    if BYPASS and (ID == 204 or ID == 203) then return false end
end

function onReceiveRpc(ID) 
    if BYPASS and (ID == 22 or ID == 21 or ID == 67 or ID == 86 or ID == 87 or ID == 15) then return false end
end

function sendSpectator(x, y, z, keysData)
    local bs = raknetNewBitStream()
    raknetBitStreamWriteInt8(bs, 212)
    raknetBitStreamWriteInt16(bs, 0)
    raknetBitStreamWriteInt16(bs, 0)
    raknetBitStreamWriteInt16(bs, keysData)
    raknetBitStreamWriteFloat(bs, x)
    raknetBitStreamWriteFloat(bs, y)
    raknetBitStreamWriteFloat(bs, z)
    raknetSendBitStream(bs)
    raknetDeleteBitStream(bs)
end

function samp_create_sync_data(sync_type, copy_from_player)
    local sampfuncs = require('sampfuncs')
    local raknet = require('samp.raknet')
    
    copy_from_player = (copy_from_player == nil) and true or copy_from_player
    local sync_traits = {
        player = {'PlayerSyncData', raknet.PACKET.PLAYER_SYNC, sampStorePlayerOnfootData},
        vehicle = {'VehicleSyncData', raknet.PACKET.VEHICLE_SYNC, sampStorePlayerIncarData},
        passenger = {'PassengerSyncData', raknet.PACKET.PASSENGER_SYNC, sampStorePlayerPassengerData},
        aim = {'AimSyncData', raknet.PACKET.AIM_SYNC, sampStorePlayerAimData},
        trailer = {'TrailerSyncData', raknet.PACKET.TRAILER_SYNC, sampStorePlayerTrailerData},
        unoccupied = {'UnoccupiedSyncData', raknet.PACKET.UNOCCUPIED_SYNC, nil},
        bullet = {'BulletSyncData', raknet.PACKET.BULLET_SYNC, nil},
        spectator = {'SpectatorSyncData', raknet.PACKET.SPECTATOR_SYNC, nil}
    }
    local sync_info = sync_traits[sync_type]
    local data_type = 'struct ' .. sync_info[1]
    local data = ffi.new(data_type, {})
    local raw_data_ptr = tonumber(ffi.cast('uintptr_t', ffi.new(data_type .. '*', data)))
    
    if copy_from_player then
        local copy_func = sync_info[3]
        if copy_func then
            local _, player_id
            if copy_from_player == true then
                _, player_id = sampGetPlayerIdByCharHandle(PLAYER_PED)
            else
                player_id = tonumber(copy_from_player)
            end
            if player_id then
                copy_func(player_id, raw_data_ptr)
            end
        end
    end
    
    local func_send = function()
        local bs = raknetNewBitStream()
        raknetBitStreamWriteInt8(bs, sync_info[2])
        raknetBitStreamWriteBuffer(bs, raw_data_ptr, ffi.sizeof(data))
        raknetSendBitStreamEx(bs, sampfuncs.HIGH_PRIORITY, sampfuncs.UNRELIABLE_SEQUENCED, 1)
        raknetDeleteBitStream(bs)
    end
    
    local mt = {
        __index = function(t, index) return data[index] end,
        __newindex = function(t, index, value) data[index] = value end
    }
    return setmetatable({send = func_send}, mt)
end

function getMyCar()
    local myCar = 0
    if isCharInAnyCar(PLAYER_PED) then
        myCar = storeCarCharIsInNoSave(PLAYER_PED)
    end
    return myCar
end

function getPlayerName(playerId)
    local name = sampGetPlayerNickname(playerId)
    if name then
        return name
    end
    return "ID " .. playerId
end

function getAllPlayersInCars()
    local players = {}
    local myX, myY, myZ = getCharCoordinates(PLAYER_PED)
    local maxDist = 75.0
    local myCar = getMyCar()
    
    for i = 0, sampGetMaxPlayerId(true) do
        if sampIsPlayerConnected(i) then
            local find, handle = sampGetCharHandleBySampPlayerId(i)
            if find and handle ~= PLAYER_PED and not isCharDead(handle) and isCharInAnyCar(handle) then
                local targetCar = storeCarCharIsInNoSave(handle)
                if targetCar and targetCar ~= myCar then
                    local x, y, z = getCharCoordinates(handle)
                    local dist = getDistanceBetweenCoords3d(myX, myY, myZ, x, y, z)
                    if dist <= maxDist then
                        table.insert(players, {id = i, handle = handle, dist = dist})
                    end
                end
            end
        end
    end
    
    table.sort(players, function(a, b) return a.dist < b.dist end)
    return players
end

function attackPlayerInCar(player_data)
    local handle = player_data.handle
    if handle == PLAYER_PED then return end
    
    local car = storeCarCharIsInNoSave(handle)
    if not car then return end
    
    local name = getPlayerName(player_data.id)
    local myCar = getMyCar()
    
    if car == myCar then return end
    
    local x, y, z = getCarCoordinates(car)
    local heading = getCarHeading(car)
    
    for i = 1, 25 do
        local offset = (i % 2 == 0) and 1.5 or -1.5
        local rad = math.rad(heading)
        local cutX = x - math.sin(rad) * offset
        local cutY = y + math.cos(rad) * offset
        
        local data = samp_create_sync_data("vehicle", true)
        if data then
            local speedMult = (i % 2 == 0) and 6.0 or -6.0
            data.moveSpeed = { -math.sin(rad) * speedMult, math.cos(rad) * speedMult, 3.5 }
            data.position = {cutX, cutY, z + 0.3}
            data.send()
        end
        wait(10)
    end
    
    sampAddChatMessage("[AdvRvanka] У " .. name .. " порвана жопа", -1)
end

function attackPlayerById(playerId)
    local find, handle = sampGetCharHandleBySampPlayerId(playerId)
    if not find then
        sampAddChatMessage("[AdvRvanka] Игрок не найден", -1)
        return
    end
    
    local name = getPlayerName(playerId)
    
    if isCharInAnyCar(handle) then
        local car = storeCarCharIsInNoSave(handle)
        if car then
            local x, y, z = getCarCoordinates(car)
            local heading = getCarHeading(car)
            
            lua_thread.create(function()
                IS_ATTACKING = true
                sampAddChatMessage("[AdvRvanka] Бутылю " .. name .. " (в машине)", -1)
                
                for i = 1, 25 do
                    local offset = (i % 2 == 0) and 1.5 or -1.5
                    local rad = math.rad(heading)
                    local cutX = x - math.sin(rad) * offset
                    local cutY = y + math.cos(rad) * offset
                    
                    local data = samp_create_sync_data("vehicle", true)
                    if data then
                        local speedMult = (i % 2 == 0) and 6.0 or -6.0
                        data.moveSpeed = { -math.sin(rad) * speedMult, math.cos(rad) * speedMult, 3.5 }
                        data.position = {cutX, cutY, z + 0.3}
                        data.send()
                    end
                    wait(10)
                end
                
                IS_ATTACKING = false
                sampAddChatMessage("[AdvRvanka] У " .. name .. " порвана жопа", -1)
            end)
        end
    else
        lua_thread.create(function()
            IS_ATTACKING = true
            sampAddChatMessage("[AdvRvanka] Бутылю " .. name .. " (на земле)", -1)
            
            local x, y, z = getCharCoordinates(handle)
            
            for i = 1, 30 do
                local data = samp_create_sync_data("player", true)
                if data then
                    local offsetX = (math.random() - 0.5) * 2
                    local offsetY = (math.random() - 0.5) * 2
                    data.position = {x + offsetX, y + offsetY, z + 0.5}
                    data.moveSpeed = {offsetX * 2, offsetY * 2, 2.0}
                    data.health = 100
                    data.armour = 0
                    data.send()
                end
                wait(8)
            end
            
            IS_ATTACKING = false
            sampAddChatMessage("[AdvRvanka] У " .. name .. " порвана жопа", -1)
        end)
    end
end

function massAttack()
    local players = getAllPlayersInCars()
    if #players == 0 then 
        sampAddChatMessage("[AdvRvanka] Не нашел бомжей в тачках (75м)", -1)
        return 
    end

    lua_thread.create(function()
        IS_ATTACKING = true
        
        for idx, player in ipairs(players) do
            local name = getPlayerName(player.id)
            sampAddChatMessage("[AdvRvanka] Бутылю " .. name .. " (" .. idx .. "/" .. #players .. ")", -1)
            attackPlayerInCar(player)
            wait(50)
        end
        
        IS_ATTACKING = false
        
        local data = samp_create_sync_data("vehicle", true)
        if data then data.send() end
    end)
end
```
