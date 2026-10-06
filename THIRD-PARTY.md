# Third-party code and dependencies

**TagEventBus 1.0.0** — last verified September 7, 2026.

TagEventBus contains no third-party code and depends on no marketplace plugins. Every
module it links is part of Unreal Engine, and the plugin ships no content assets
(`CanContainContent` is `false`).

Use of Unreal Engine itself is governed by the
[Unreal Engine End User License Agreement](https://www.unrealengine.com/eula).

## Engine modules linked

### Runtime modules

`TagEventBus`, `TagEventBusRequests`, `TagEventBusContracts`

| Module | Used by |
| --- | --- |
| Core | all three |
| CoreUObject | all three |
| GameplayTags | all three |
| Engine | all three |
| DeveloperSettings | `TagEventBus`, `TagEventBusContracts` |
| UnrealEd | `TagEventBusContracts`, editor builds only (`bBuildEditor`) |

### Editor module

`TagEventBusEditor`

| Module | Used by |
| --- | --- |
| Core | `TagEventBusEditor` |
| CoreUObject | `TagEventBusEditor` |
| Engine | `TagEventBusEditor` |
| GameplayTags | `TagEventBusEditor` |
| GameplayTagsEditor | `TagEventBusEditor` |
| Slate | `TagEventBusEditor` |
| SlateCore | `TagEventBusEditor` |
| UnrealEd | `TagEventBusEditor` |
| ToolMenus | `TagEventBusEditor` |
| InputCore | `TagEventBusEditor` |
| PropertyEditor | `TagEventBusEditor` |

### Engine plugins

| Plugin | Note |
| --- | --- |
| GameplayTagsEditor | Ships with Unreal Engine; enabled in `TagEventBus.uplugin` |
