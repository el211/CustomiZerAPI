# CustomiZer Developer API

This guide is synchronized from the main CustomiZer repository. Its Maven setup
section describes that multi-module checkout (`fr.elias:customizer-api`). For this
standalone repository use the [README installation instructions](README.md):
local Maven coordinates are `fr.elias:CustomiZerAPI:BETA-2.1`, or use the documented
JitPack coordinates. Only the main repository contains the plugin implementation.

A guide for plugin developers who want to integrate with CustomiZer.

---

## Setup

### 1. Add CustomiZer as a dependency

In your `plugin.yml`:
```yaml
depend: [CustomiZer]
# or if it's optional:
softdepend: [CustomiZer]
```

### 2. Add CustomiZer to your build path

Build and install the current reactor from the repository root:
```bash
mvn install
```
Then add to your `pom.xml`:
```xml
<dependency>
    <groupId>fr.elias</groupId>
    <artifactId>customizer-api</artifactId>
    <version>BETA-2.1</version>
    <scope>provided</scope>
</dependency>
```

### 3. Get the API instance

Use the API dependency with `provided` scope; **do not shade or relocate it**.
The CustomiZer plugin supplies it at runtime. Do not install the API-only JAR as a
server plugin. Both artifacts retain the repository's BETA-2.1 version; this update
does not raise the minimum Paper dependency just to add facade methods.

Call instance methods on the server thread. Registration happens after handler
initialization and is cleared before shutdown. For optional integrations use
`CustomiZerAPI.getOrNull()` or `isAvailable()`; reacquire after reload rather than
keeping a stale instance. No asynchronous safety is implied.

```java
import fr.elias.customiZer.api.CustomiZerAPI;

public class MyPlugin extends JavaPlugin {
    private CustomiZerAPI customizer;

    @Override
    public void onEnable() {
        if (getServer().getPluginManager().isPluginEnabled("CustomiZer")) {
            customizer = CustomiZerAPI.get();
        }
    }
}
```

> **Note:** `CustomiZerAPI.get()` throws `IllegalStateException` if called before CustomiZer has enabled. Since `depend` guarantees load order, calling it in your own `onEnable()` is always safe.

---

## Current pack, armor and cosmetic support (BETA-2.1)

These methods use the existing plugin factories; they do not construct imitation
items or spawn alternate renderers. Pack-local armor YAML is loaded through the
same merged configuration as the plugin.

```java
CustomiZerAPI api = CustomiZerAPI.get();
Set<String> yamlKeys = api.getItemKeys();
Set<String> packs = api.getPackNames();
Set<String> armors = api.getArmorKeys();
ItemStack helmet = api.buildArmorItem("void_wolf_helmet");
ItemStack wings = api.buildCosmeticItem("valentines_wings");
List<FunctionalItemInfo> entries = api.getFunctionalItems("fenrir_armor");
ItemStack chest = api.buildFunctionalItem("fenrir_armor", "armor", "fenrir_chestplate");
```

Builders return null for unresolved entries. Functional catalog entries are the
configured `/zitems` list, not an exhaustive registry of every internal subsystem.
Their builders use the GUI's underlying item factory but do not apply GUI-only
name/lore overlays. Items are returned, not given or equipped. Permissions and
equipment behavior remain controlled by the existing plugin flows.

- `getAllPackItems()` lists visual pack items across categories.
- `buildPackItem(pack, id)` retains modern item/equipment metadata.
- `getPackModelInfo(pack, id)` exposes item model, resolved model, equipment model,
  equipment sound and tooltip style; fields may be null when not configured.
- `getCosmeticKeys()` lists the currently loaded cosmetic IDs.
- `getItemNames()` retains its old display-name lookup IDs; `getItemKeys()` returns
  the original YAML keys instead.

Existing API method signatures and the six-field `PackItemInfo` record are retained.
New methods have concrete unsupported defaults for older third-party subclasses;
the bundled plugin overrides all of them. Updated name/key sets and the rewards
map are read-only snapshots (reward values themselves retain their existing API).
Furniture locations are cloned on construction and access.

`givePackItem` still fires its cancellable event; inventory overflow is now dropped
at the player's feet instead of discarded. `giveBlockItem` uses the same overflow
handling. `PackItemGiveEvent` is an API give-path event, not a universal `/zitems`
GUI event. No new event guarantee is implied for functional-item builders.

Verification: API lifecycle/legacy-subclass/snapshot tests and a plugin-side test
requiring every public API instance method to have a concrete implementation.
The full reactor was compiled and tested; live third-party plugin integrations
have not been exercised here.

## Custom Items

Custom items are defined in `items.yml`. They are special use-items with actions like HEAL, SPEED, COMMAND, etc.

### List legacy lookup IDs or original configuration keys

```java
Set<String> keys = customizer.getItemNames();
Set<String> configKeys = customizer.getItemKeys();
// e.g. ["blood_ingot", "speed_flask", "jump_potion"]
```

### Get item data

```java
import fr.elias.customiZer.CustomItemData;

CustomItemData data = customizer.getItemData("blood_ingot");
if (data != null) {
    String name       = data.getName();        // MiniMessage display name
    String actionType = data.getActionType();  // "HEAL", "SPEED", "COMMAND", etc.
    int modelData     = data.getModelData();   // custom model data value
    int cooldown      = data.getCooldown();    // cooldown in seconds
    int uses          = data.getUses();        // max uses (-1 = unlimited)
}
```

> Both the config key (e.g. `"blood_ingot"`) and the stripped display name work as the lookup argument.

### Build an ItemStack

```java
ItemStack stack = customizer.buildItemStack("blood_ingot");
if (stack != null) {
    // stack is ready — has display name, lore, model data, and a unique UUID tag
}
```

### Give an item to a player

```java
customizer.giveItem(player, "blood_ingot");
// Adds the item to the player's inventory and sends them a confirmation message
```

### Create a drop-style ItemStack

Useful for custom drop tables. Accepts a config key, a `packName:itemId` pack reference, or a vanilla material name:

```java
ItemStack drop = customizer.createDropItem("blood_ingot", 3); // 3x blood_ingot
ItemStack drop = customizer.createDropItem("blossom_studios:ruby", 1); // pack item
ItemStack drop = customizer.createDropItem("DIAMOND", 2); // vanilla fallback
```

### Check if an ItemStack is a CustomiZer item

```java
if (customizer.isCustomItem(stack)) {
    // stack has the custom_item_uuid PDC tag set by CustomiZer
}
```

---

## Custom Blocks

Custom blocks are defined in `blocks.yml`. They are placeable blocks with custom model data, custom drops, and optional sounds/particles.

### List all registered block keys

```java
Set<String> keys = customizer.getBlockKeys();
// e.g. ["ruby_ore", "amethyst_node", "mossy_log"]
```

### Build a custom block ItemStack

```java
ItemStack blockItem = customizer.buildBlockItem("ruby_ore");
if (blockItem != null) {
    // ready to place, has the custom_block PDC tag
}
```

### Give a custom block to a player

```java
customizer.giveBlockItem(player, "ruby_ore");
// Adds the block item directly to the player's inventory
```

### Check if an ItemStack is a custom block

```java
if (customizer.isCustomBlockItem(stack)) {
    // this item was created by CustomiZer's block system
}
```

---

## Furniture

Furniture pieces are pack items with `furniture:` config in a pack's `config.yml`. When placed in the world, they become hidden ItemFrames. You can look them up by world location or iterate over all placed pieces.

### Get furniture at a specific location

```java
import fr.elias.customiZer.api.PlacedFurnitureInfo;

PlacedFurnitureInfo info = customizer.getFurnitureAt(block.getLocation());
if (info != null) {
    String packName    = info.packName();      // e.g. "blossom_studios"
    String itemId      = info.itemId();        // e.g. "oak_chair"
    int modelData      = info.modelData();     // custom model data value
    Location baseLoc   = info.baseLocation();  // block where it sits
    UUID frameUuid     = info.frameUuid();     // UUID of the backing ItemFrame entity
}
```

> The lookup is block-precise. Any location inside the furniture's footprint returns the same result.

### Get all placed furniture in the world

```java
Collection<PlacedFurnitureInfo> all = customizer.getAllPlacedFurniture();
for (PlacedFurnitureInfo info : all) {
    // iterate over every placed furniture piece across all worlds
}
```

### Check if an entity is a furniture frame

```java
@EventHandler
public void onEntityClick(PlayerInteractEntityEvent event) {
    if (customizer.isFurnitureEntity(event.getRightClicked())) {
        // player clicked a CustomiZer furniture ItemFrame
    }
}
```

### Give a furniture item to a player

Furniture is a pack item. Use `givePackItem`:

```java
customizer.givePackItem(player, "blossom_studios", "oak_chair");
```

---

## Pack Items

Pack items are any items registered from a resource pack folder under `packs/`.

### Get a specific pack item

```java
import fr.elias.customiZer.handler.PackManager.PackItem;

PackItem item = customizer.getPackItem("blossom_studios", "ruby_sword");
if (item != null) {
    String name   = item.displayName;
    int modelData = item.modelData;
    ItemStack stack = item.buildItemStack();
}
```

### Get a pack item by custom model data

```java
PackItem item = customizer.getPackItemByModelData(30105);
```

### Get all pack items in a category

```java
List<PackItem> swords = customizer.getPackItems("swords");
```

### Give a pack item (fires `PackItemGiveEvent`)

```java
boolean given = customizer.givePackItem(player, "blossom_studios", "ruby_sword");
// returns false if the item doesn't exist or PackItemGiveEvent was cancelled
```

---

## Events

Listen to these events from your plugin to hook into CustomiZer's flows.

### `CustomItemUseEvent`

Fired when a player activates a custom item's action. **Cancellable** — cancelling prevents the action.

```java
import fr.elias.customiZer.api.event.CustomItemUseEvent;

@EventHandler
public void onCustomItemUse(CustomItemUseEvent event) {
    Player player = event.getPlayer();
    CustomItemData data = event.getItemData();

    if (data.getActionType().equals("FLY") && !player.hasPermission("myplugin.canfly")) {
        event.setCancelled(true);
        player.sendMessage("You don't have permission to use that.");
    }
}
```

### `CustomBlockRewardEvent`

Fired when a player breaks a block that has CustomiZer reward data. **Cancellable** — cancelling skips the rewards. You can also modify the payout values.

```java
import fr.elias.customiZer.api.event.CustomBlockRewardEvent;

@EventHandler
public void onBlockReward(CustomBlockRewardEvent event) {
    // Double money in a specific world
    if (event.getBlock().getWorld().getName().equals("mining_world")) {
        event.setMoney(event.getMoney() * 2);
    }

    // Cancel rewards if player is in a specific region
    if (isInSafeZone(event.getPlayer())) {
        event.setCancelled(true);
    }
}
```

### `PackItemGiveEvent`

Fired when `givePackItem()` is called. **Cancellable** — cancelling prevents the item from being given. You can also swap the ItemStack.

```java
import fr.elias.customiZer.api.event.PackItemGiveEvent;

@EventHandler
public void onPackItemGive(PackItemGiveEvent event) {
    if (event.getPackItem().itemId.equals("legendary_blade")) {
        // log legendary item grants
        getLogger().info(event.getPlayer().getName() + " received a legendary blade.");
    }
}
```

---

## Player Progression

### Mastery XP and level

```java
UUID uuid = player.getUniqueId();

long xp    = customizer.getMasteryXP(uuid, "mining");
int  level = customizer.getMasteryLevel(uuid, "mining");

customizer.setMasteryXP(uuid, "mining", xp + 500);
```

### Prestige level

```java
int prestige = customizer.getPrestigeLevel(uuid);
```

### Mission progress

```java
int progress      = customizer.getMissionProgress(uuid, "mine_ruby_ore");
boolean completed = customizer.isMissionCompleted(uuid, "mine_ruby_ore");
```

---

## Reward Data

### Get reward data for a block type

```java
import fr.elias.customiZer.rewards.RewardData;

RewardData data = customizer.getRewardData("diamond_ore");
if (data != null) {
    double xp     = data.getJobsxp();
    double money  = data.getMoney();
    double points = data.getPoints();
    String job    = data.getJob();
}
```

### Get all reward data

```java
Map<String, RewardData> all = customizer.getAllRewardData();
// keyed by lower-case material name, e.g. "diamond_ore"
```

---

## Quick Reference

| What you want | Method |
|---|---|
| All custom item keys | `getItemNames()` |
| Item definition | `getItemData(key)` |
| Build item ItemStack | `buildItemStack(key)` |
| Give item to player | `giveItem(player, key)` |
| Check if stack is custom item | `isCustomItem(stack)` |
| All custom block keys | `getBlockKeys()` |
| Build block ItemStack | `buildBlockItem(key)` |
| Give block to player | `giveBlockItem(player, key)` |
| Check if stack is custom block | `isCustomBlockItem(stack)` |
| Furniture at location | `getFurnitureAt(location)` |
| All placed furniture | `getAllPlacedFurniture()` |
| Check if entity is furniture | `isFurnitureEntity(entity)` |
| Give pack item | `givePackItem(player, pack, id)` |
| Get pack item | `getPackItem(pack, id)` |
| Player mastery XP | `getMasteryXP(uuid, track)` |
| Player prestige level | `getPrestigeLevel(uuid)` |
| Reward data for block | `getRewardData(material)` |
# Terrain generation and programmatic blocks (BETA-2.1)

`buildBlockItem(key)` produces an inventory item, **not** a world block state.
Calling `getType()` on that item loses the custom block identity. In particular,
a PAPER item with custom model data must never become the terrain material.

Resolve native, non-emissive blocks on the server thread after CustomiZer is ready:

```java
BlockData stone = CustomiZerAPI.get().getBlockData("customizer:ambre_ore");
if (stone == null) throw new IllegalArgumentException("Unknown custom block: ambre_ore");
```

Accepted identifiers: a `blocks:` YAML key, `customizer:<key>`, `pack:item`,
or `customizer:pack:item`. The alias must point to an existing registered block;
this API does not create a new ore definition or texture. DISPLAY-backed blocks,
non-block pack items, and blocks with positive `light_level` are unsupported and
throw `UnsupportedOperationException`, rather than silently losing visuals or light.

In a chunk generator, retain **BlockData** in the profile fields and use the
`ChunkData.setBlock(x, y, z, blockData)` overload. Cache resolved states before
generation starts, safely publish the profile, and use a clone per worker. Do
not call the API, load chunks, or mutate live world blocks on generator threads.
Resolve vanilla entries such as SAND through `Material.SAND.createBlockData()`.
Do not convert the custom state back to Material anywhere along the pipeline.
Resource packs and CustomiZer's stable note-state registry must match the world.

For a single live-world placement on the main server thread:

```java
boolean placed = CustomiZerAPI.get().placeBlock(location, "customizer:ambre_ore");
```

The chunk must already be loaded. Unknown keys and protected replacement targets
return false. Tile entities, furniture anchors, barriers, light blocks, note blocks
and tracked pack blocks are not overwritten. This is deliberately **not** a generic
custom-block removal/replacement API. It does not fire player placement events,
check region permissions, charge items or generate drops; the caller owns those checks.
Existing player-placement behavior is unchanged.

## PlanetariaGen migration

The supplied PlanetariaGen.jar resolves `buildBlockItem` and then `getType()` in
`BlockRef`. Its `PlanetProfile` terrain fields are Material, so changing YAML alone
cannot fix it. Its developer must change `BlockRef`, profile layer/ore types and
generator writes to preserve BlockData. Recompile against this API and declare
CustomiZer as a dependency (or explicitly defer custom profile resolution until it
is enabled). The third-party JAR has not been modified.

## WorldEdit / FAWE status

There is **no registered `customizer:` WorldEdit command parser in this update**.
For an integration operating on ordinary terrain, the developer can adapt a resolved
BlockData with WorldEdit's BukkitAdapter and pass it to their EditSession, retaining
WorldEdit's normal history handling. Resolve before entering asynchronous work.
Do not use this technique to overwrite existing CustomiZer-managed blocks: display
entities, emissive packet states and persisted location tracking also need cleanup,
and undo/redo must reconcile that metadata. A complete command bridge needs that
additional lifecycle integration; a pattern parser alone would not be safe.
