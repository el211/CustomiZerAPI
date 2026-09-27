# CustomiZer API

The official developer API for the **CustomiZer** Minecraft plugin.  
Add it as a `provided` dependency to hook into CustomiZer from your own plugin.

---

## Installation

### Maven

```xml
<repositories>
    <repository>
        <id>jitpack.io</id>
        <url>https://jitpack.io</url>
    </repository>
</repositories>

<dependencies>
    <dependency>
        <groupId>com.github.el211</groupId>
        <artifactId>CustomiZerAPI</artifactId>
        <version>main-SNAPSHOT</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

### Gradle

```groovy
repositories {
    maven { url 'https://jitpack.io' }
}

dependencies {
    compileOnly 'com.github.el211:CustomiZerAPI:main-SNAPSHOT'
}
```

> **Note:** Use `provided` / `compileOnly` — CustomiZer ships the API classes at runtime. Do not shade them.

The current source version is BETA-2.1. This repository is the new home of the standalone CustomiZer API.
`main-SNAPSHOT` follows the branch; pin a commit for reproducible JitPack builds.
JitPack publication is separate from a successful local Maven build.

## BETA-2.1 additions

- Lifecycle checks: `isAvailable()` and `getOrNull()`; reacquire after reload.
- Pack, armor, cosmetic and functional item discovery and factories.
- Modern pack model metadata through `PackModelInfo`.
- Native custom block lookup: `getBlockData("customizer:ambre_ore")`.
- Main-thread live placement: `placeBlock(location, "customizer:ambre_ore")`.

Read the [complete integration guide](API_GUIDE.md), including threading and placement limitations.
Existing abstract API methods remain available. The new methods have defaults for
older implementations; the updated CustomiZer plugin implements them.

Terrain generators must retain `BlockData`, not `buildBlockItem(...).getType()`:
the latter discards the custom state. Resolve on the server thread, cache the
state, then use `ChunkData.setBlock(x, y, z, data)` with worker-local clones.
The configured block must exist. Entity-backed/display and emissive blocks are
not supported by the native block API. WorldEdit/FAWE command integration is not
included; see the guide before attempting bulk replacement of managed blocks.

---

## Getting the API instance

```java
CustomiZerAPI api = CustomiZerAPI.get();
```

Call this after CustomiZer has been enabled. A good place is inside your plugin's `onEnable()`, after declaring CustomiZer as a `depend` or `softdepend` in `plugin.yml`.

```yaml
# plugin.yml
softdepend:
  - CustomiZer
```

---

## Custom Items

### CustomStack — get an item by id (like ItemsAdder)

```java
CustomStack stack = CustomStack.getInstance("blood_ingot");
if (stack != null) {
    ItemStack item = stack.getItemStack().clone();
    item.setAmount(amount);
    player.getInventory().addItem(item);
}
```

### Other item methods

```java
CustomiZerAPI api = CustomiZerAPI.get();

// Build an ItemStack directly
ItemStack item = api.buildItemStack("blood_ingot");

// Give an item to a player
api.giveItem(player, "blood_ingot");

// Check if an ItemStack is a CustomiZer item
boolean isCustom = api.isCustomItem(itemStack);

// List all registered item keys
Set<String> keys = api.getItemNames();

// Get full item configuration
CustomItemData data = api.getItemData("blood_ingot");
```

---

## Pack Items

```java
CustomiZerAPI api = CustomiZerAPI.get();

// Get a pack item by pack name + item id
PackItemInfo info = api.getPackItem("blossom_studios", "ruby_sword");

// Give a pack item to a player (fires PackItemGiveEvent)
api.givePackItem(player, "blossom_studios", "ruby_sword");

// Get all items in a category
List<PackItemInfo> items = api.getPackItems("weapons");

// Get by custom model data value
PackItemInfo info = api.getPackItemByModelData(1042);
```

---

## Custom Blocks

```java
// Check if an ItemStack is a CustomiZer block
boolean isBlock = api.isCustomBlockItem(itemStack);

// Build the block's ItemStack
ItemStack blockItem = api.buildBlockItem("my_block");

// Give a block item to a player
api.giveBlockItem(player, "my_block");

// List all block keys
Set<String> blockKeys = api.getBlockKeys();
```

---

## Furniture

```java
// Get furniture at a location
PlacedFurnitureInfo furniture = api.getFurnitureAt(location);
if (furniture != null) {
    String pack = furniture.packName();
    String id   = furniture.itemId();
}

// Get all placed furniture in the world
Collection<PlacedFurnitureInfo> all = api.getAllPlacedFurniture();

// Check if an entity is a furniture frame
boolean isFurniture = api.isFurnitureEntity(entity);
```

---

## Glyphs (Font Images)

```java
// Get a glyph by pack + font id
Glyph glyph = api.getGlyph("gem_ranks", "ba_i");

// Get the rendered character (use in chat / lore)
String character = api.getGlyphChar("gem_ranks", "ba_i");

// Search glyphs
List<Glyph> results = api.searchGlyphs("coin");

// Get all glyphs for a pack
List<Glyph> packGlyphs = api.getPackGlyphs("gem_ranks");
```

---

## Player Progression

```java
UUID uuid = player.getUniqueId();

// Mastery XP
long xp    = api.getMasteryXP(uuid, "mining");
int  level = api.getMasteryLevel(uuid, "mining");
api.setMasteryXP(uuid, "mining", xp + 500);

// Prestige
int prestige = api.getPrestigeLevel(uuid);

// Missions
int  progress  = api.getMissionProgress(uuid, "mine_100_diamonds");
boolean done   = api.isMissionCompleted(uuid, "mine_100_diamonds");
```

---

## Block Rewards

```java
// Get reward config for a block material
RewardData reward = api.getRewardData("diamond_ore");
if (reward != null) {
    double xp    = reward.getJobsxp();
    double money = reward.getMoney();
    List<DropData> drops = reward.getDrops();
}

// Get all reward configs
Map<String, RewardData> allRewards = api.getAllRewardData();
```

---

## Events

Listen to CustomiZer events in your plugin:

```java
@EventHandler
public void onCustomItemUse(CustomItemUseEvent event) {
    Player player       = event.getPlayer();
    CustomItemData data = event.getItemData();
    // cancel to prevent the action (cooldown still applies)
    event.setCancelled(true);
}

@EventHandler
public void onBlockReward(CustomBlockRewardEvent event) {
    // Modify reward amounts before they are paid out
    event.setMoney(event.getMoney() * 2);
    event.setJobsXP(event.getJobsXP() * 1.5);
}

@EventHandler
public void onPackItemGive(PackItemGiveEvent event) {
    PackItemInfo item = event.getPackItem();
    // Replace the item that gets given
    event.setItemStack(myCustomStack);
    // Or cancel entirely
    event.setCancelled(true);
}
```

---

## API Classes

| Class | Description |
|---|---|
| `CustomiZerAPI` | Main entry point — call `CustomiZerAPI.get()` |
| `CustomStack` | Resolve a custom item by id |
| `PackItemInfo` | Immutable snapshot of a pack item |
| `CustomItemData` | Full configuration of a custom item |
| `Glyph` | Registered font image / unicode character |
| `PlacedFurnitureInfo` | Snapshot of placed furniture |
| `RewardData` | Block reward configuration |
| `DropData` | Single drop entry in a reward |
| `ClickTypes` | Interaction type enum for item actions |
| `event/CustomItemUseEvent` | Fired when a player uses a CustomiZer item |
| `event/CustomBlockRewardEvent` | Fired when a block reward is distributed |
| `event/PackItemGiveEvent` | Fired before a pack item is given to a player |
