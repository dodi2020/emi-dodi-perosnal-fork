# EMI Minecraft 1.21.11 Update Guide

## Overview
This document outlines the systematic approach to update EMI from Minecraft 1.21.1 to 1.21.11.

## Research Summary

### Version Timeline
- 1.21.1 → 1.21.2 (Bundles of Bravery - Part 1)
- 1.21.2 → 1.21.3 (Bundles of Bravery - Part 2)  
- 1.21.3 → 1.21.4 (Garden Awakens)
- 1.21.4 → 1.21.5 through 1.21.11 (incremental updates)

### Major Breaking Changes by Version

#### Minecraft 1.21.2/1.21.3 Changes
**Fabric Tooling:**
- Loom 1.8+ required
- Fabric Loader 0.16.7+
- Java 21 recommended (Java 23 supported)

**API Changes:**
- MixinExtras annotations: `@WrapMethod`, `@Cancellable`, namespace sharing in `@Share`
- New Loot API v3
- Creative inventory API extensions
- Item API extensions for enchantments and components
- Renderer API: `ShadeMode`, quads overload for joml
- More convention tags and entity events

#### Minecraft 1.21.4 Changes (MAJOR)
**Fabric Tooling:**
- Loom 1.9+ required
- Fabric Loader 0.16.9+

**Removed/Deprecated APIs:**
1. `fabric-rendering-v0` module - REMOVED
2. `FabricModelPredicateProviderRegistry` - REMOVED
3. `BuiltinItemRenderer` and `BuiltinItemRendererRegistry` - Replaced with transitive access widener via `SpecialModelTypes.ID_MAPPER`
4. `AttachmentRegistry#builder` - Deprecated, use new create methods
5. `BlockPickInteractionAware` - Removed, use new pick item events
6. `CustomIngredient.getMatchingStacks` - **BREAKING**: Must return `Stream` instead of `List`

**New Features:**
- Data Attachments: new registration and sync API for mod data
- Transfer API extended for item-containing items
- New chunk/world lifecycle events

#### Minecraft 1.21.5+ Changes
**Removed APIs:**
1. `VillagerInteractionsRegistries#registerGiftLootTable` (Identifier overload)
2. `VillagerProfessionBuilder` and `VillagerTypeHelper` - Removed
3. `TradeOfferHelper#refreshOffers` - Removed
4. `HudRenderCallback` - Deprecated (use `HudLayerRegistrationCallback`)

**Changed APIs:**
1. Dynamic registries must use namespaced directories
2. `BiomeModificationContext#addSpawn` now includes weight parameter

#### Minecraft 1.21.11 Final Changes
**Identifier Changes (CRITICAL):**
- Constructor is now `protected`
- Must use `Identifier.of()` or `Identifier.ofVanilla()`
- All `new Identifier(...)` calls will break

**Registry Changes:**
- Registry Sync now freezes registries earlier in mod loading
- All registrations must be done in `ModInitializer`

**Item API:**
- `EquipmentSlotProvider#getPreferredEquipmentSlot` receives `LivingEntity` parameter
- `FabricItem#getAttributeModifiers` removed - use Default Components API

**Required Changes:**
- Add English tag translations for all custom tags
- Update resource conditions for better registry support

## Dependency Updates

### gradle.properties
```properties
# Minecraft and Mappings
minecraft_version=1.21.11
yarn_mappings=1.21.11+build.3

# Fabric
fabric_loader_version=0.18.4
fabric_api_version=0.140.2+1.21.11

# Optional Dependencies
jei_version=jei-1.21.11-fabric:27.3.0.12
```

### build.gradle (Root)
```gradle
plugins {
    id "architectury-plugin" version "3.4-SNAPSHOT"
    id "dev.architectury.loom" version "1.7-SNAPSHOT" apply false
}
```

**Note:** Loom 1.14-SNAPSHOT would be ideal for 1.21.11 but requires plugin compatibility testing.

### fabric/build.gradle
```gradle
dependencies {
    // Update ModMenu
    modCompileOnly "com.terraformersmc:modmenu:12.0.0"
    
    // JEI already updated via gradle.properties
    modCompileOnly("mezz.jei:${rootProject.jei_version}") {
        transitive = false
    }
}
```

## Code Changes Required

### 1. Identifier Usage
**Search Pattern:** `new Identifier(`

**Change From:**
```java
new Identifier("emi", "path")
new Identifier("minecraft:item")
```

**Change To:**
```java
Identifier.of("emi", "path")
Identifier.ofVanilla("item")
```

### 2. CustomIngredient (if used)
**Change From:**
```java
@Override
public List<ItemStack> getMatchingStacks() {
    return stackList;
}
```

**Change To:**
```java
@Override
public Stream<ItemStack> getMatchingStacks() {
    return stackList.stream();
}
```

### 3. Equipment Slot Provider (if used)
**Change From:**
```java
EquipmentSlot getPreferredEquipmentSlot(ItemStack stack);
```

**Change To:**
```java
EquipmentSlot getPreferredEquipmentSlot(ItemStack stack, LivingEntity entity);
```

### 4. HUD Rendering (if used)
**Change From:**
```java
HudRenderCallback.EVENT.register((matrices, tickDelta) -> {
    // render code
});
```

**Change To:**
```java
HudLayerRegistrationCallback.EVENT.register((context) -> {
    // register layers
});
```

### 5. Villager Trades (if used)
**Remove usage of:**
- `VillagerProfessionBuilder`
- `VillagerTypeHelper`
- `TradeOfferHelper#refreshOffers`

**Use instead:**
- `VillagerProfession` constructors directly
- `VillagerType#create`

### 6. Data Attachments (if adding new features)
**Change From:**
```java
AttachmentRegistry.builder()
    .type(type)
    .build();
```

**Change To:**
```java
AttachmentRegistry.create(type);
```

### 7. Rendering APIs
**If using `fabric-rendering-v0`:**
- Module is removed - must migrate to newer APIs
- Use transitive access widener for built-in item rendering
- Check for `FabricModelPredicateProviderRegistry` usage and remove

### 8. Tag Translations
Add to language files (e.g., `en_us.json`):
```json
{
  "tag.item.c.your_tag": "Your Tag Display Name",
  "tag.block.c.your_tag": "Your Tag Display Name"
}
```

## Build and Test Strategy

### Phase 1: Dependency Updates (Current Status)
- [x] Update gradle.properties
- [x] Update build.gradle versions
- [x] Update fabric/build.gradle dependencies
- [⚠️] Resolve Architectury plugin repository access

### Phase 2: Code Migration
1. Search for all Identifier constructors
2. Search for CustomIngredient implementations
3. Search for equipment slot providers
4. Search for HUD render callbacks
5. Search for villager trade registrations
6. Search for rendering v0 API usage

### Phase 3: Build Testing
1. Run `./gradlew clean build`
2. Fix compilation errors
3. Run `./gradlew :fabric:build`
4. Test mod loading in Minecraft 1.21.11
5. Verify all features work correctly

### Phase 4: Validation
1. Test with JEI integration
2. Test with ModMenu integration
3. Test recipe viewing
4. Test item tooltips
5. Test all custom EMI features

## GitHub Actions Workflow

Created: `.github/workflows/build-1.21.11.yml`

Features:
- Java 21 (Temurin distribution)
- Modern Gradle caching
- Fabric build with stacktrace
- Artifact upload (excluding dev-shadow jars)
- Build logs on failure

Triggers:
- Push to main or update branch
- Pull requests to main
- Manual workflow dispatch

## Known Issues

### 🚫 BLOCKING: Maven Repository Access
**Issue:** `maven.architectury.dev` cannot be resolved in the build environment.

**Error:**
```
curl: (6) Could not resolve host: maven.architectury.dev
```

**Impact:** Cannot resolve Architectury plugin, blocking all builds.

**Potential Solutions:**
1. Unblock maven.architectury.dev in environment
2. Use alternative Architectury plugin repository
3. Provide local plugin artifacts
4. Use GitHub Actions with network access

**Status:** Awaiting resolution

## Migration Checklist for Each Version

### Testing for 1.21.4 (Major Version)
- [ ] Verify no usage of `fabric-rendering-v0`
- [ ] Check CustomIngredient returns Stream
- [ ] Test attachment registry API
- [ ] Build and run mod

### Testing for 1.21.10 (Required Build)
- [ ] Update to 1.21.10 dependencies
- [ ] Build successfully
- [ ] Test all features
- [ ] Verify no regressions

### Testing for 1.21.11 (Highest Priority)
- [ ] All Identifier usage updated
- [ ] All deprecated APIs removed
- [ ] Tag translations added
- [ ] Build successfully
- [ ] Full feature testing
- [ ] Performance testing

## Resources

### Official Documentation
- [Fabric for 1.21.1](https://fabricmc.net/2024/05/31/121.html)
- [Fabric for 1.21.2 & 1.21.3](https://fabricmc.net/2024/10/14/1212.html)
- [Fabric for 1.21.4](https://fabricmc.net/2024/12/02/1214.html)
- [Fabric Documentation](https://docs.fabricmc.net/)
- [Architectury Documentation](https://docs.architectury.dev/)

### Dependency Repositories
- [Fabric API - Modrinth](https://modrinth.com/mod/fabric-api)
- [JEI - CurseForge](https://www.curseforge.com/minecraft/mc-mods/jei)
- [ModMenu - Modrinth](https://modrinth.com/mod/modmenu)

## Notes

- This is a systematic guide based on official changelogs and community documentation
- Always backup before updating
- Test incrementally at each major version
- Some breaking changes may not be documented and require testing to discover
- Performance may vary between versions
