# C Data

C Data is a simple wrapper for LuaJIT FFI's cast to C String. This is primarily used to serialize Lua tables into network-transferable data.

It is worth noting that if you encode an incomplete table (missing a key) then when you decode the packet, it will have the missing key with a value of 0.


## Examples

### Register Packets

```lua
local cdata   = require "cdata"
local packets = {}

-- all structs get a type field so we don't lose our minds.
function add_struct(name, fields, map)
    local struct = string.format("typedef struct { uint8_t type; %s } %s;", fields, name)
    cdata:new_struct(name, struct)

    -- the packet_type struct isn't a real packet, so don't index it.
    if map then
        map.name = name
        table.insert(packets, map)
        packets[name] = #packets
    end
end

-- Slightly special, I guess.
add_struct("packet_type", "")

add_struct(
    "player_whois", [[
        uint16_t id;
    ]], {
        "id",
    }
)

add_struct(
    "player_create", [[
        uint16_t id;
        uint8_t flags;
        float position_x, position_y, position_z;
        float orientation_x, orientation_y, orientation_z;
        unsigned char name[64];
    ]], {
        "id",
        "flags",
        "position_x", "position_y", "position_z",
        "orientation_x", "orientation_y", "orientation_z",
        "name",
    }
)

add_struct(
    "player_update", [[
        uint16_t id;
        float position_x, position_y, position_z;
        float orientation_x, orientation_y, orientation_z;
    ]], {
        "id",
        "position_x", "position_y", "position_z",
        "orientation_x", "orientation_y", "orientation_z",
    }
)

add_struct(
    "player_action", [[
        uint16_t id;
        uint16_t action;
    ]], {
        "id",
        "action",
    }
)
```


### Encode Data

```lua
local player = self.players[1]
local data   = {
    type          = packets["player_update"],
    id            = player.id,
    position_x    = player.position.x,
    position_y    = player.position.y,
    position_z    = player.position.z,
    orientation_x = player.orientation.x,
    orientation_y = player.orientation.y,
    orientation_z = player.orientation.z,
}

local struct  = cdata:set_struct("player_update", data)
local encoded = cdata:encode(struct)
```


### Decode Data

```lua
-- We assume we have a variable named data that we received from the network
local header = cdata:decode("packet_type", data)
local map    = packets[header.type]

if not map then
    error(string.format("Invalid packet type (%s) received!", header.type))
    return
end

local decoded = cdata:decode(map.name, data)
```
