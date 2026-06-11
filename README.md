# Animated 3D Models on World Tiles
## Project Zomboid Build 42

How to attach a `boned up` animated 3D model to a tile object and play its
animation, that renders through the chunk render pipeline, from Lua.
No pack or tiles files required

## How does this shit work?

1. We're going to use `common/media/spriteModels.txt`. At startup,
   `SpriteModelManager` loads this file from the `common` directory of every
   active mod. Remember, **the common directory**.
2. Each entry binds a tile to a `modelScript`, plus translate/rotate/scale, an 
   `animation` clip name, and an `animationTime`. Each entry is registered in
   the ScriptManager as `<tileset>_<index>`
3. The tileset name does **not** have to be a real tilesheet! A made-up name
   still registers the entry, and you attach it to any object with `obj:setSpriteModelName("my_fakeass_tileset_0")`. The 3D model then renders
   *instead of* the object's 2D sprite.
4. When the object has no active animation player, the renderer `poses` the
   model at `spriteModel:getAnimationTime()` - a **normalized** 0-1 position in
   the clip - every time the object's render chunk redraws. We'll use Lua to 
   animate the model.

### Why not use `obj:setAnimating(true)`? It's not like I didn't try!

`setAnimating` that only sets a flag. Real playback starts with
`IsoObjectAnimations.getInstance().addObject(obj, spriteModel, clipName)` but 
`IsoObjectAnimations` is **not exposed to Lua**. Hence the `animationTime` driver below. If TIS exposes it, we get smoove per-object playback and can delete the driver!

## 1. The model (Enter Boss: Blender)

Do it exactly like vanilla's animated doors (check out `media/models_X/IsoObject/door_n_sw.glb` and the `.blend` sources in `media/models_X/IsoObject/blender/`):

- An **armature** with one or more bones and the mesh **skinned** to it. One bone is enough for simple motion.
- One **action** animating the *bones* (not the object). The action name
  becomes the clip name.
- **Keyframes must start at frame 0.** Blender's glTF exporter does not slide
  the action to time 0.

Export as a **glb** with armature + mesh only, animation mode "Actions". Use .glb, not FBX - I had trouble with FBX and it frequently "`Failed to load asset`".

 Animation file goes to: `common/media/models_X/IsoObject/MyThing.glb`  
 Texture goes to: `common/media/textures/MyThing.png`

## 2. Model scripts

`common/media/scripts/MyMod_Models.txt`:

```
module MyMod
{
    /* loads the clips embedded in the glb */
    animationsMesh MyThingMesh
    {
        meshFile = IsoObject/MyThing,
        keepMeshAnimations = true,
    }

    model MyThing
    {
        mesh = IsoObject/MyThing,
        animationsMesh = MyThingMesh,
        texture = MyThing,
        static = false,
        scale = 1.0,
        undoCoreScale = true,
    }
}
```

## 3. The sprite model entries

`common/media/spriteModels.txt`. Two entries, same model. One whose
`animationTime` we drive (animating), one fixed at 0 (off). Toggling an object
between them is how we start/stop it without affecting the others.

```
spriteModel
{
    VERSION = 1,

    tileset
    {
        name = mymod_things_01,

        /* access as "mymod_things_01_0" - the running one */
        tile
        {
            xy = 0 0,
            modelScript = MyMod.MyThing,
            translate = 0.0000 0.0000 0.0000,
            rotate = 0.0000 0.0000 0.0000,
            scale = 1.0000,
            animation = MyClipName,
            animationTime = 0.0000,
        }

        /* access as "mymod_things_01_1" - the idle one */
        tile
        {
            xy = 1 0,
            modelScript = MyMod.MyThing,
            translate = 0.0000 0.0000 0.0000,
            rotate = 0.0000 0.0000 0.0000,
            scale = 1.0000,
            animation = MyClipName,
            animationTime = 0.0000,
        }
    }
}
```

You can tune `translate`/`rotate`/`scale` with the debug **Sprite Model Editor**

## 4. The Lua
 **This is not optimized and not the way I would do it. It's for demo only**

`42/media/lua/client/MyMod_AnimatedThing.lua`

```lua
local MODEL_RUNNING = "mymod_things_01_0"
local MODEL_IDLE = "mymod_things_01_1"

local CYCLE_SECONDS = 2.0  -- real time for one full clip cycle
local CYCLE_STEPS = 60     -- distinct 'poses' per cycle (see note below)

-- dirty bit for invalidateRenderChunkLevel
local CHUNK_DIRTY_FLAG = 256

MyAnimatedThing = MyAnimatedThing or {}

-- objects currently animating
local running = {}
local runningCount = 0
local cycleTime = 0.0
local lastQuantized = -1

local function setRunningFlag(obj, on)
    if on and not running[obj] then
        running[obj] = true
        runningCount = runningCount + 1
    elseif not on and running[obj] then
        running[obj] = nil
        runningCount = runningCount - 1
    end
end

-- sprite model assignment is render state and not saved with the object
-- reapply whenever the object is created or the chunk loads
local function applySpriteModel(obj)
    local on = obj:getModData().myThingRunning
    obj:setSpriteModelName(on and MODEL_RUNNING or MODEL_IDLE)
end

--- spawn one on a square. the sprite name is just an identifier here
--- the 3D model replaces it
function MyAnimatedThing.place(square)
    local obj = IsoObject.new(getCell(), square, MODEL_IDLE)
    obj:getModData().isMyThing = true
    obj:getModData().myThingRunning = false
    square:AddTileObject(obj)
    applySpriteModel(obj)
    obj:transmitCompleteItemToServer()
    return obj
end

--- start/stop animation
function MyAnimatedThing.setRunning(obj, on)
    obj:getModData().myThingRunning = on
    applySpriteModel(obj)
    setRunningFlag(obj, on)
    obj:invalidateRenderChunkLevel(CHUNK_DIRTY_FLAG)
end

-- the animation driver
-- tldr CYCLE_STEPS poses get cached
-- quantization matters because the engine caches one pose per distinct
-- float. Sweeping through raw time would grow that cache forever
-- quantized values are reused after the first cycle
-- =============================================================================

local function OnTick()
    if runningCount == 0 then return end

    cycleTime = (cycleTime + getGameTime():getRealworldSecondsSinceLastUpdate() / CYCLE_SECONDS) % 1.0
    local quantized = math.floor(cycleTime * CYCLE_STEPS) / CYCLE_STEPS
    if quantized == lastQuantized then return end
    lastQuantized = quantized

    local sm = getScriptManager():getSpriteModel(MODEL_RUNNING)
    if not sm then return end
    sm:setAnimationTime(quantized)

    local notLoaded = nil
    for obj in pairs(running) do
        if obj:getSquare() == nil then
            notLoaded = notLoaded or {}
            table.insert(notLoaded, obj)
        else
            obj:invalidateRenderChunkLevel(CHUNK_DIRTY_FLAG)
        end
    end
    if notLoaded then
        for _, obj in ipairs(notLoaded) do setRunningFlag(obj, false) end
    end
end

Events.OnTick.Add(OnTick)

-- reapply the render state when chunks load/grid square loads and
-- resume anything saved as running
local function OnLoadGridsquare(square)
    local objects = square:getObjects()
    for i = 0, objects:size() - 1 do
        local obj = objects:get(i)
        if obj:hasModData() and obj:getModData().isMyThing then
            applySpriteModel(obj)
            if obj:getModData().myThingRunning then
                setRunningFlag(obj, true)
            end
        end
    end
end

Events.LoadGridsquare.Add(OnLoadGridsquare)

-- right-click ground to place / toggle for testing

local function findMyThing(square)
    if not square then return nil end
    local objects = square:getObjects()
    for i = 0, objects:size() - 1 do
        local obj = objects:get(i)
        if obj:hasModData() and obj:getModData().isMyThing then return obj end
    end
end

local function onContext(player, context, worldobjects, test)
    if test and ISWorldObjectContextMenu.Test then return true end
    local square = nil
    for _, v in ipairs(worldobjects) do square = v:getSquare() break end
    if not square then return end

    local obj = findMyThing(square)
    if obj then
        local label = obj:getModData().myThingRunning and "Debug: Stop Thing" or "Debug: Start Thing"
        context:addOption(label, obj, function(o) MyAnimatedThing.setRunning(o, not o:getModData().myThingRunning) end)
    else
        context:addOption("Debug: Place Animated Thing", square, MyAnimatedThing.place)
    end
end

Events.OnPreFillWorldObjectContextMenu.Add(onContext)
```

## File layout summary

```
MyMod/
├── 42/
│   ├── mod.info
│   └── media/lua/client/MyMod_AnimatedThing.lua
└── common/
    └── media/
        ├── spriteModels.txt
        ├── scripts/MyMod_Models.txt
        ├── models_X/IsoObject/MyThing.glb
        └── textures/MyThing.png
```

## Stupid shit

- **All running objects share one pose.** The lua driver sets one 
  `animationTime` on one shared SpriteModel script object, so everything using 
  it animates in sync. For independent phases, register more spriteModel 
  entries and drive each at an offset.
- **`CYCLE_STEPS` trade-off:** pose changes per second = CYCLE_STEPS /
  CYCLE_SECONDS. Each step caches one pose forever (a few KB each).
  60 steps over 2s = 30 pose updates/sec.
- **MP:** the driver is client-side render state. Sync the on/off state
  your own self (mod data + server commands) and call `setRunning` on each client.
- **Console check:** `Failed to load asset: AssetPath{ "IsoObject/MyThing" }`
  at startup means the mesh didn't load (wrong path, or FBX - I told you to use a .glb).
- **The 2D sprite is hidden once the model renders.** If the model fails to
  load, the object falls back to drawing whatever sprite name you created it
  with.
