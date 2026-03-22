---
title: 东方夜雀食堂-Mod-制作笔记
date: 2025-12-10 15:33:41
tags:
---

## 前言

该日记主要随着分析和开发进度推进而更新，因此一般来说靠后的研究更有参考价值，靠前的研究大多为实验

## 安装 BepInEx

## hello world

### 获取字体列表

在 `PluginManager.Awake` 执行

```c#
// 获取并记录所有加载的字体
Font[] fonts = Resources.FindObjectsOfTypeAll<Font>();
Log.LogMessage("Loaded Fonts:");
foreach (var font in fonts)
{
    Log.LogMessage("Font: " + font.name);
}
```

获取到如下字体

```shell
Loaded Fonts:
Font: JYHPHZ
Font: SongMyung-Regular
Font: SiYuan
Font: KaiseiTokumin-ExtraBold
Font: jiangxizhuokai
```

### 想到哪分析到哪

移动交互类(`DayScene`)

```c#
DayScene.Input.DayScenePlayerInputGenerator.OnMovePerformed
DayScene.Input.DayScenePlayerInputGenerator.OnMoveCanceled
DayScene.Input.DayScenePlayerInputGenerator.OnSprintPerformed
DayScene.Input.DayScenePlayerInputGenerator.OnSprintCanceled
DayScene.Input.DayScenePlayerInputGenerator.TryInteract
```

玩家按下方向键时，`OnMovePerformed` 被调用，切换方向键时，`OnMovePerformed` 被调用，松开全部方向键时`OnMoveCanceled` 被调用。按下 `小跑` 键后，`OnSprintPerformed` 被调用，直到松开后 `OnSprintCanceled`被调用。玩家按下 `交互` 键时，无论有无实际可交互对象，都会执行 `TryInteract`

`OnMovePerformed` 会在处理后将方向向量传入 `Common.CharacterUtility.CharacterControllerInputGeneratorComponent.UpdateInputDirection`

```c
v23 = DEYU_Utils_VectorMatrix__Matrix_6446669568(v17, _9__16_0, 0LL);
x_low = (__m128)LODWORD(v23.fields.x);
y_low = (__m128)LODWORD(v23.fields.y);
x_low.m128_f32[0] = v23.fields.x * this->fields.moveSpeed;
y_low.m128_f32[0] = v23.fields.y * this->fields.moveSpeed;
Common_CharacterUtility_CharacterControllerInputGeneratorComponent__UpdateInputDirection(
    (Common_CharacterUtility_CharacterControllerInputGeneratorComponent_o *)this,
    (UnityEngine_Vector2_o)*(_OWORD *)&_mm_unpacklo_ps(x_low, y_low),
    0LL);
```

该 `Common.CharacterUtility.CharacterControllerInputGeneratorComponent.UpdateInputDirection` 的入参 `Vector2 inputDirection` 有且仅有以下⑨个可能的值 $\{(0, 0), (1, 0), (0, 1), (-1, 0), (0, -1), (\frac{\sqrt{2}}{2}, \frac{\sqrt{2}}{2}), (\frac{\sqrt{2}}{2}, -\frac{\sqrt{2}}{2}), (-\frac{\sqrt{2}}{2}, \frac{\sqrt{2}}{2}), (-\frac{\sqrt{2}}{2}, -\frac{\sqrt{2}}{2})\}$



尝试打印出了 `DayScene.DaySceneMap.allCharacters` 的各个坐标，大部分是指示地图上角色的位置，但是玩家操纵的小碎骨不在此列

```c#
var characters = DayScene.DaySceneMap.allCharacters;
if (characters != null) {
    foreach (var kvp in characters) {
        var id = kvp.Key;
        var component = kvp.Value;
        if (component != null && component.gameObject != null) {
            var pos = component.gameObject.transform.position;
            Log.LogMessage($"Character {id}: {component.gameObject.name} at ({pos.x}, {pos.y}, {pos.z})");
        } else {
            Log.LogMessage($"Character {id}: null or no gameObject");
        }
    }
} else {
    Log.LogMessage("allCharacters is null");
}
```

对实际小碎骨的位置的 `getter` 和 `setter` 

```c#
public static Vector2? GetPlayerPosition()
{
    var characters = UnityEngine.Object.FindObjectsOfType<DayScene.Input.DayScenePlayerInputGenerator>();
    if (characters == null || characters.Length == 0) {
        Log.LogMessage("未找到 DayScenePlayerInputGenerator 实例");
        return null;
    }
    if (characters.Length > 1) {
        Log.LogWarning($"找到 {characters.Length} 个 DayScenePlayerInputGenerator 实例，使用第一个");
    }

    var character = characters[0];

    var characterUnit = character.Character;
    if (characterUnit == null) {
        Log.LogMessage("CharacterControllerUnit 为空");
        return null;
    }

    var rb = characterUnit.rb2d;
    if (rb == null) {
        Log.LogMessage("Rigidbody2D 为空");
        return null;
    }

    return rb.position;
}

public static void SetPlayerPosition(float x, float y)
{
    var characters = UnityEngine.Object.FindObjectsOfType<DayScene.Input.DayScenePlayerInputGenerator>();
    if (characters == null || characters.Length == 0) {
        Log.LogMessage("未找到 DayScenePlayerInputGenerator 实例");
        return;
    }
    if (characters.Length > 1) {
        Log.LogWarning($"找到 {characters.Length} 个 DayScenePlayerInputGenerator 实例，使用第一个");
    }

    var character = characters[0];

    var characterUnit = character.Character;
    if (characterUnit == null) {
        Log.LogMessage("CharacterControllerUnit 为空");
        return;
    }

    var rb = characterUnit.rb2d;
    if (rb == null) {
        Log.LogMessage("Rigidbody2D 为空");
        return;
    }

    rb.position = new Vector2(x, y);
    Log.LogMessage($"已设置玩家位置到 ({x}, {y})");
}
```

由于是单机，小碎骨本身不存放地图数据，地图由 `DayScene.SceneManager` 管理，可以读取 `DayScene.SceneManager.CurrentActiveMapLabel` 来获取当前地图名

而实际执行传送机制的方法为 `DayScene.SceneManager.SwapMap`，如果需要检测是否执行传送，应考虑 hook 后者

```c#
public unsafe void SwapMap(string targetMapLabel, string targetMarkerName, int travelCount, bool shouldFadeIn = true, bool shouldFadeOut = true, bool triggerEnterMapEvent = true, Action onSwapFinish = null)
```

其中，`travelCount` 为花费 `??` 数



`DayScene.SceneManager$$SwapMap`

```c
public unsafe void SwapMap(string targetMapLabel, string targetMarkerName, int travelCount, bool shouldFadeIn = true, bool shouldFadeOut = true, bool triggerEnterMapEvent = true, Action onSwapFinish = null)
void DayScene_SceneManager__SwapMap(
        DayScene_SceneManager_o *this,
        System_String_o *targetMapLabel,
        System_String_o *targetMarkerName,
        int32_t travelCount,
        bool shouldFadeIn,
        bool shouldFadeOut,
        bool triggerEnterMapEvent,
        System_Action_o *onSwapFinish,
        const MethodInfo *method)
```

执行地图切换的实际逻辑，有 `DayScene_SceneManager$$SwapMapAsync` 辅助



`DayScene.DaySceneMap$$RefreshNPCs`

```c#
public unsafe void RefreshNPCs(bool rotateCharacter = false)
void DayScene_DaySceneMap__RefreshNPCs(DayScene_DaySceneMap_o *this, bool rotateCharacter, const MethodInfo *method)
```

刷新当前地图上的所有NPC，处理角色实例化、位置更新和状态同步。会用到 `DayScene.DaySceneMap$$SolveAndUpdateCharacterPositionInternal`



`DayScene.DaySceneMap$$SolveAndUpdateCharacterPositionInternal`

`public unsafe void SolveAndUpdateCharacterPositionInternal(Dictionary<string, TrackedNPC> npcs, TrackedNPC npc, CharacterConditionComponent character, out bool isNPCOnMap, bool changeRotation = false)`

当地图更新时候加载，负责解决并更新角色在地图上的位置、旋转、碰撞器和可见性。处理正常位置更新、覆盖位置和特殊客人逻辑。传出 `isNPCOnMap`

### 研究 NPC 生命周期

>  主要需要解决问题：`Kyouko` 正常只会在兽道才会被加载，在 `Home` 中不会被加载，因此需要提前加载 `Kyouko`

观察到 `DayScene.DaySceneMap.allCharacters` 存储着当前的全部 NPC 角

```c#
// DayScene.DaySceneMap
public unsafe static Dictionary<string, CharacterConditionComponent> allCharacters;
```

在 `Mystia` 刚刚进入 `Home` 时，不会加载任何 NPC，`allCharacters` 为空，当 `Mystia` 进入户外时，`allCharacter` 被加入了多个对象。全局搜索发现只在 `DayScene.DaySceneMap$$RefreshNPCs` 被引用。`RefreshNPCs` 调用了 `GameData.RunTime.DaySceneUtility.RunTimeDayScene$$GetMapNPCs` 获取全部 NPC 对象。

```c#
// GameData.RunTime.DaySceneUtility.RunTimeDayScene
public unsafe static Dictionary<string, TrackedNPC> GetMapNPCs(string mapLabel);
```

`Home` 地图下没有任何 NPC，`BeastForest` 地图下有非常多 NPC，考虑一个可能的尝试方案：hook `GetMapNPCs` 如果入参为 `Home` 则代理访问 `GetMapNPCs("BeastForest")` 把 `Kyouko` 拿过来用作为

```c#

[HarmonyPatch(nameof(RunTimeDayScene.GetMapNPCs))]
[HarmonyPrefix]
public static bool GetMapNPCs_Prefix(string mapLabel, ref Dictionary<string, TrackedNPC> __result)
{
    if (mapLabel != "Home")
    {
        return true;
    }

    Log.LogInfo("[RunTimeDayScenePatch] GetMapNPCs called for 'Home' map. Intercepting...");

    var beastForestNPCs = RunTimeDayScene.GetMapNPCs("BeastForest");
    foreach (var kvp in beastForestNPCs)
    {
        Log.LogInfo($"[RunTimeDayScenePatch] NPC in 'BeastForest': {kvp.Key}");
    }

    var kyouko = beastForestNPCs.ContainsKey("Kyouko") ? beastForestNPCs["Kyouko"] : null;
    if (kyouko != null)
    {
        Log.LogInfo("[RunTimeDayScenePatch] 'Kyouko' NPC found in 'BeastForest'.");

        var result = new Dictionary<string, TrackedNPC>();
        result.Add("Kyouko", kyouko);

        __result = result;
        return false;
    }
    else 
    {
        Log.LogInfo("[RunTimeDayScenePatch] 'Kyouko' NPC NOT found in 'BeastForest'.");
        __result = new Dictionary<string, TrackedNPC>();
        return false;
    }
}
```

```log
[Warning:MetaMystia] [RunTimeDayScenePatch] NPC in 'BeastForest': Mike
[Warning:MetaMystia] [RunTimeDayScenePatch] 'Kyouko' NPC found in 'BeastForest'.
[Error  :     Unity] NPC = Kyouko
[Error  :     Unity] Key = Kyouko
[Error  :Il2CppInterop] During invoking native->managed trampoline
Exception: Il2CppInterop.Runtime.Il2CppException: System.Collections.Generic.KeyNotFoundException: The given key 'Kyouko' was not present in the dictionary.
--- BEGIN IL2CPP STACK TRACE ---
System.Collections.Generic.KeyNotFoundException: The given key 'Kyouko' was not present in the dictionary.
  at System.Collections.Generic.Dictionary`2[TKey,TValue].get_Item (TKey key) [0x00000] in <00000000000000000000000000000000>:0 
  at DayScene.DaySceneMap.SolveAndUpdateCharacterPositionInternal (System.Collections.Generic.Dictionary`2[TKey,TValue] npcs, GameData.RunTime.DaySceneUtility.Collection.TrackedNPC npc, DayScene.Interactables.Collections.ConditionComponents.CharacterConditionComponent character, System.Boolean& isNPCOnMap, System.Boolean changeRotation) [0x00000] in <00000000000000000000000000000000>:0 
--- END IL2CPP STACK TRACE ---

   at Il2CppInterop.Runtime.Il2CppException.RaiseExceptionIfNecessary(IntPtr returnedException) in /home/runner/work/Il2CppInterop/Il2CppInterop/Il2CppInterop.Runtime/Il2CppException.cs:line 36
   at DMD<DayScene.DaySceneMap::SolveAndUpdateCharacterPositionInternal>(DaySceneMap this, Dictionary`2 npcs, TrackedNPC npc, CharacterConditionComponent character, Boolean& isNPCOnMap, Boolean changeRotation)
   at (il2cpp -> managed) SolveAndUpdateCharacterPositionInternal(IntPtr , IntPtr , IntPtr , IntPtr , Byte& , Byte , Il2CppMethodInfo* )
[Message:MetaMystia] Kyouko visibility updated to False (Kyouko map: '', Mystia map: 'Home')
```

> DEYU：@MetaMiku SpecialGuests无法Copy，但是如果你只是想要场景里面存在一个不能互动的小人的话，可以用 SceneDirector.Instance.SpawnCharacter

```c#
// Common.SceneDirector
// Token: 0x06005062 RID: 20578 RVA: 0x00002053 File Offset: 0x00000253
[Token(Token = "0x6005062")]
[Address(RVA = "0x7E39F0", Offset = "0x7E23F0", VA = "0x1807E39F0")]
public void SpawnCharacter(SceneDirector.Identity characterType, int characterId, Vector2 startPosition, string label)
{
}

```

该方法可以很方便地直接创建角色，并可以通过相同的方式进行控制，但是该方法所创建的角色是使用世界坐标，这意味着切换地图时，角色不会消失。可以考虑修改 z 值到 -10 及以下

## 启用 EnableDebugCosole

> DEYU：@MetaMiku 你有Hook的技术的话你优先Hook SplashScene.SceneManager.EnableDebugCosole 让其返回true，并且Hook SplashScene.SceneManager.CurrentConsoleMode 让其返回 ConsoleMode.Full (0)

### Hook

```C#
// SplashScene.SceneManager
// Token: 0x17000203 RID: 515
// (get) Token: 0x060005B5 RID: 1461 RVA: 0x000B0CCC File Offset: 0x000AEECC
public unsafe static bool EnableDebugCosole
{
	[CallerCount(299)]
	[CachedScanResults(RefRangeStart = 25582, RefRangeEnd = 25881, XrefRangeStart = 25582, XrefRangeEnd = 25582, MetadataInitTokenRva = 0L, MetadataInitFlagRva = 0L)]
	get
	{
		IntPtr* ptr = null;
		IntPtr intPtr2;
		IntPtr intPtr = IL2CPP.il2cpp_runtime_invoke(SceneManager.NativeMethodInfoPtr_get_EnableDebugCosole_Public_Static_get_Boolean_0, 0, (void**)ptr, ref intPtr2);
		Il2CppException.RaiseExceptionIfNecessary(intPtr2);
		return *IL2CPP.il2cpp_object_unbox(intPtr);
	}
}

```

该字段只有 getter 无 setter，优先考虑 hook getter

```C#
[HarmonyPatch(typeof(SplashScene.SceneManager))]
public class SplashSceneSceneManagerPatch
{
    private static ManualLogSource Log => Plugin.Instance.Log;

    [HarmonyPatch("EnableDebugCosole", MethodType.Getter)]
    [HarmonyPostfix]
    public static void EnableDebugCosole_Postfix(ref bool __result)
    {
        __result = true;
    }

    [HarmonyPatch("CurrentConsoleMode", MethodType.Getter)]
    [HarmonyPostfix]
    public static void CurrentConsoleMode_Postfix(ref GamePlatform.Systems.ConsoleMode __result)
    {
        __result = GamePlatform.Systems.ConsoleMode.Full;
    }
}
```

无法启动，游戏报错

```log
[Error  :     Unity] Unable to read the setting file at
Memory\Mystia.setting
[Error  :     Unity] NotSupportedException: Collection is read-only.
[Message:     Unity] UnivGameMana: Awake
[Error  :     Unity] Unable to read the setting file at
Memory\Mystia.setting
[Error  :     Unity] NotSupportedException: Collection is read-only.
```

如果强行删掉 `Memory\Mystia.setting` 文件会报错

```log
NotSupportedException: Collection is read-only.
  at System.Collections.ObjectModel.Collection`1[T].Insert (System.Int32 index, T item) [0x00000] in <00000000000000000000000000000000>:0 
  at Newtonsoft.Json.JsonSerializer.ApplySerializerSettings (Newtonsoft.Json.JsonSerializer serializer, Newtonsoft.Json.JsonSerializerSettings settings) [0x00000] in <00000000000000000000000000000000>:0 
  at Newtonsoft.Json.JsonSerializer.CreateDefault () [0x00000] in <00000000000000000000000000000000>:0 
  at Newtonsoft.Json.JsonConvert.SerializeObject (System.Object value, Newtonsoft.Json.Formatting formatting) [0x00000] in <00000000000000000000000000000000>:0 
  at GameData.Utils.SaveManagement.SaveSettingData (GameData.Utils.PlayerSettings playerSettings) [0x00000] in <00000000000000000000000000000000>:0 
  at GameData.Utils.SaveManagement.LoadSettingData () [0x00000] in <00000000000000000000000000000000>:0 
  at Common.UI.EscapeUtility.EscConfigPannel.get_CurrentSettings () [0x00000] in <00000000000000000000000000000000>:0 
  at Common.UI.EscapeUtility.EscConfigPannel.get_CurrentLanguage () [0x00000] in <00000000000000000000000000000000>:0 
  at SplashScene.SceneManager.Awake () [0x00000] in <00000000000000000000000000000000>:0
```

出问题的是这几行

```c
void SplashScene_SceneManager__Awake(SplashScene_SceneManager_o *this, const MethodInfo *method)
{
  // ...
  CurrentLanguage = Common_UI_EscapeUtility_EscConfigPannel__get_CurrentLanguage(0LL);
  if ( !GameData_MultiLanguageTextMeshCore_TypeInfo->_2.cctor_finished )
    il2cpp_runtime_class_init();
  GameData_MultiLanguageTextMeshCore__SetLanguageType(CurrentLanguage, 0LL);
  Common_UI_EscapeUtility_EscConfigPannel__MountCurrentSettings(0LL);
  Config = GamePlatform_Systems_Launch__GetConfig(0LL);
  // ...
}
```

配置/设置文件无法加载

### 延迟 hook

尝试修改 hook 方法，在进入游戏后再改该字段

```C#
[HarmonyPatch(typeof(SplashScene.SceneManager))]
public class SplashSceneSceneManagerPatch
{
    private static ManualLogSource Log => Plugin.Instance.Log;
    private static readonly string LOG_TAG = "[SplashSceneSceneManagerPatch]";
    public static SplashScene.SceneManager instanceRef = null;

    [HarmonyPatch("EnableDebugCosole", MethodType.Getter)]
    [HarmonyPostfix]
    public static void EnableDebugCosole_Postfix(ref bool __result)
    {
        __result = PluginManager.EnableDebugCosole; // EnableDebugCosole 初始为 false，会在进入游戏后根据需要随时更改
    }

    [HarmonyPatch("CurrentConsoleMode", MethodType.Getter)]
    [HarmonyPostfix]
    public static void CurrentConsoleMode_Postfix(ref GamePlatform.Systems.ConsoleMode __result)
    {
        __result = GamePlatform.Systems.ConsoleMode.Full;
    }
}
```

启动游戏后，再改 `PluginManager.EnableDebugCosole` 实现对 `SplashScene.SceneManager.EnableDebugCosole` 的伪修改

结果导致资源全部无法加载，如果再改回 `false`，游戏会处于一种诡异的半加载状态

 ![image-20251204030256959](/image/articles/东方夜雀食堂-Mod-制作笔记.assets/image-20251204030256959.png)

![image-20251204030331857](/image/articles/东方夜雀食堂-Mod-制作笔记.assets/image-20251204030331857.png)

### Patch-1

再次推测可能是由于 hook 时间不够早，尝试从源头直接修改游戏 dll 文件

首先需要定位 `EnableDebugCosole`，诡异的是无法定位，如图

![image-20251204030622722](/image/articles/东方夜雀食堂-Mod-制作笔记.assets/image-20251204030622722.png)

同样是 readonly 的 `CurrentConsoleMode` 却能看到对应的 getter，而 `EnableDebugCosole`

回看 dnspy，用 il2cppdumper 导出的 Assembly-CSharp 可能更简单一点

```C#
// SplashScene.SceneManager
// Token: 0x1700005E RID: 94
// (get) Token: 0x06000325 RID: 805 RVA: 0x00002DD8 File Offset: 0x00000FD8
[Token(Token = "0x1700005E")]
public static bool EnableDebugCosole
{
	[Token(Token = "0x6000325")]
	[Address(RVA = "0x42E0B0", Offset = "0x42CAB0", VA = "0x18042E0B0")]
	get
	{
		return default(bool);
	}
}

```

可以利用文件偏移量 Offset 在 010 中定位到代码 

![image-20251204032935555](/image/articles/东方夜雀食堂-Mod-制作笔记.assets/image-20251204032935555.png)

或者更直接的用基址+偏移量即 `VA` 获取虚拟地址，在 IDA 中可以发现他指向了一个 `il2cpp` 段的方法

```asm
il2cpp:000000018042E0B0
il2cpp:000000018042E0B0                         ; =============== S U B R O U T I N E =======================================
il2cpp:000000018042E0B0
il2cpp:000000018042E0B0
il2cpp:000000018042E0B0                         ; bool DEYU_AssetHandleUtility_TempAssetHandle_object___get_IsPersistentAsset(DEYU_AssetHandleUtility_TempAssetHandle_T__o *this, const MethodInfo_42E0B0 *method)
il2cpp:000000018042E0B0                         DEYU_AssetHandleUtility_TempAssetHandle_object_$$get_IsPersistentAsset proc near
il2cpp:000000018042E0B0                                                                 ; CODE XREF: sub_180261430+C↑p
il2cpp:000000018042E0B0                                                                 ; DEYU_Utils_UnityEngineExtensionStatic__ReadAllFromFolderAsync_d__68$$MoveNext+123↑p ...
il2cpp:000000018042E0B0
il2cpp:000000018042E0B0                         method          = qword ptr  28h
il2cpp:000000018042E0B0
il2cpp:000000018042E0B0 32 C0                                   xor     al, al
il2cpp:000000018042E0B2 C3                                      retn
il2cpp:000000018042E0B2                         ; ---------------------------------------------------------------------------
il2cpp:000000018042E0B3 CC CC CC CC CC CC CC CC                 align 20h
il2cpp:000000018042E0B3 CC CC CC CC CC          DEYU_AssetHandleUtility_TempAssetHandle_object_$$get_IsPersistentAsset endp
il2cpp:000000018042E0B3
```

`DEYU_AssetHandleUtility_TempAssetHandle_object_$$get_IsPersistentAsset`

看交叉引用也可以发现 `SplashScene.SceneManager.Awake` 确实有调用 `get_IsPersistentAsset`

因此作 Patch，将 `(32 C0)  xor al, al` 改成 `(BO 01)  mov al, 1` 即可

但是运行发现一切与 hook 方法别无二样

### 忽略异常

试图在抛出异常的源头给跳过

```c
// local variable allocation has failed, the output may be wrong!
void System_Collections_ObjectModel_Collection_object___Insert(
        System_Collections_ObjectModel_Collection_T__o *this,
        int32_t index,
        Il2CppObject *item,
        const MethodInfo_1BC3E10 *method)
{
  struct System_Collections_Generic_IList_T__o *items; // rdi
  System_Collections_ObjectModel_Collection_T__RGCTXs *rgctx_data; // r10
  __int64 _2_System_Collections_Generic_ICollection_T; // rax
  struct System_Collections_Generic_IList_T__o *v11; // rdi
  System_Collections_ObjectModel_Collection_T__RGCTXs *v12; // rcx
  __int64 v13; // rax

  items = this->fields.items;
  if ( !items )
    goto LABEL_12;
  rgctx_data = method->klass->rgctx_data;
  _2_System_Collections_Generic_ICollection_T = (__int64)rgctx_data->_2_System_Collections_Generic_ICollection_T_;
  if ( (*(_BYTE *)(_2_System_Collections_Generic_ICollection_T + 306) & 1) == 0 )
    _2_System_Collections_Generic_ICollection_T = sub_180365730(rgctx_data->_2_System_Collections_Generic_ICollection_T_);
  if ( (unsigned __int8)sub_180002C50(1LL, _2_System_Collections_Generic_ICollection_T, items) )
    System_ThrowHelper__ThrowNotSupportedException_6478897136(28, 0LL);
  v11 = this->fields.items;
  if ( !v11 )
LABEL_12:
    sub_18033D420(this, *(_QWORD *)&index);
  v12 = method->klass->rgctx_data;
  v13 = (__int64)v12->_2_System_Collections_Generic_ICollection_T_;
  if ( (*(_BYTE *)(v13 + 306) & 1) == 0 )
    v13 = sub_180365730(v12->_2_System_Collections_Generic_ICollection_T_);
  if ( index > (unsigned int)sub_180002C50(0LL, v13, v11) )
    System_ThrowHelper__ThrowArgumentOutOfRange_IndexException(0LL);
  ((void (__fastcall *)(System_Collections_ObjectModel_Collection_T__o *, _QWORD, Il2CppObject *, const MethodInfo *))this->klass->vtable._36_InsertItem.methodPtr)(
    this,
    (unsigned int)index,
    item,
    this->klass->vtable._36_InsertItem.method);
}
```

即 `NOP` 掉或动调时改 `RIP` 来跳过 `line 23` 的 `Throw`

不出意外的没用，该炸还是炸

### 分析

观察 `get_IsPersistentAsset` 的交叉引用，发现有大量的调用

![image-20251204034549874](/image/articles/东方夜雀食堂-Mod-制作笔记.assets/image-20251204034549874.png)

这很可能已经超出了 `EnableDebugCosole` 本身的作用。结合名字，推测 `get_IsPersistentAsset` 是原本管理(指定)资源是否是持久化资源，可能将他改为 `true` 会使得大量资源被只读限制而无法正常加载卸载，因此在游戏加载时无法正常加载游戏配置，游戏中无法正常加载卸载游戏资源。

不清楚为什么 `EnableDebugCosole` 的 getter 会变成这样一个风马牛不相及的 `get_IsPersistentAsset`，猜测可能是由于都是硬编码的 false，在编译 IL 和 il2cpp 时被莫名“链接”到一起，出现一 hook 俱 hook 的惨案。

### 强制调用

既然 `SplashScene.SceneManager.Awake` 会被调用，那么尝试 hook 他获取 instance 并在之后手动调用 `SplashScene.SceneManager.Start`

```c#
[HarmonyPatch(typeof(SplashScene.SceneManager))]
public class SplashSceneSceneManagerPatch
{
    private static ManualLogSource Log => Plugin.Instance.Log;
    private static readonly string LOG_TAG = "[SplashSceneSceneManagerPatch]";
    public static SplashScene.SceneManager instanceRef = null;

    [HarmonyPatch("EnableDebugCosole", MethodType.Getter)]
    [HarmonyPostfix]
    public static void EnableDebugCosole_Postfix(ref bool __result)
    {
        __result = PluginManager.EnableDebugCosole;
    }

    [HarmonyPatch("CurrentConsoleMode", MethodType.Getter)]
    [HarmonyPostfix]
    public static void CurrentConsoleMode_Postfix(ref GamePlatform.Systems.ConsoleMode __result)
    {
        __result = GamePlatform.Systems.ConsoleMode.Full;
    }
    
    // SplashScene.SceneManager$$Awake
    [HarmonyPatch(nameof(SplashScene.SceneManager.Awake))]
    [HarmonyPrefix]
    public static void Awake_Prefix(ref SplashScene.SceneManager __instance)
    {
        instanceRef = __instance;
        Log.LogWarning($"{LOG_TAG} SplashScene SceneManager Awake called");
    }
}


public class PluginManager : MonoBehaviour
{
    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.F1))
        {
            SplashSceneSceneManagerPatch.instanceRef.Init();
            SplashSceneSceneManagerPatch.instanceRef.Start();
        }
    }
}
```

结果更惨，实验两次，出现了两种不同报错

![image-20251204035627389](/image/articles/东方夜雀食堂-Mod-制作笔记.assets/image-20251204035627389.png)

![image-20251204035644043](/image/articles/东方夜雀食堂-Mod-制作笔记.assets/image-20251204035644043.png)

*~~这是什么？kk 的 qq 号！加一下（~~*

### 绕过

再次回看 `SplashScene.SceneManager$$Awake`，除了会在 `Common.UI.EscapeUtility.EscConfigPannel$$get_CurrentLanguage` 内会因为修改 `get_IsPersistentAsset` 而崩溃，还在后面有且只有一次直接调用了 `get_IsPersistentAsset`

```C
  if ( GamePlatform_MonoScripts_GamePlatformManager_TypeInfo->static_fields->_GamePlatformType_k__BackingField == 1
    || DEYU_AssetHandleUtility_TempAssetHandle_object___get_IsPersistentAsset(0LL, v19) )
  {
LABEL_56:
    if ( !DEYU_Utils_UnityEngineExtensionStatic_TypeInfo->_2.cctor_finished )
      il2cpp_runtime_class_init();
    DEYU_Utils_UnityEngineExtensionStatic__Log((Il2CppObject *)this, StringLiteral_9170, 0LL);
    v24 = (UnityEngine_GameObject_o *)sub_18033D3C0(UnityEngine_GameObject_TypeInfo);
    v22 = v24;
    if ( !v24 )
      goto LABEL_52;
    UnityEngine_GameObject___ctor(v24, StringLiteral_9163, 0LL);
    v23 = Method_UnityEngine_GameObject_AddComponent_GlobalDebugConsole___;
    goto LABEL_21;
  }
```

根据上一段的猜测，可能只有这里的 `get_IsPersistentAsset` 才是真正的 `EnableDebugCosole`，因此可以修改这里的返回值，由于正常情况下永远不会进入 `if` 即 `||` 两段恒为 `false`，因此采用一种最取巧的方法 `(74)  jz` 改 `(75)  jnz`

![image-20251204041437878](/image/articles/东方夜雀食堂-Mod-制作笔记.assets/image-20251204041437878.png)

运行，右面多了个小东西，点开

![image-20251204041821200](/image/articles/东方夜雀食堂-Mod-制作笔记.assets/image-20251204041821200.png)

win！



20251204 04:19 By MetaMiku

### Patch-2

`SplashScene.SceneManager$$Awake+15F`

```asm
il2cpp:000000018042AEFD
il2cpp:000000018042AEFD                         loc_18042AEFD:                          ; CODE XREF: SplashScene_SceneManager$$Awake+138↑j
il2cpp:000000018042AEFD 48 8B 05 0C 0E E1 03                    mov     rax, cs:GamePlatform_MonoScripts_GamePlatformManager_TypeInfo ; GamePlatform.MonoScripts.GamePlatformManager_TypeInfo
il2cpp:000000018042AF04 48 8B 88 B8 00 00 00                    mov     rcx, [rax+0B8h]
il2cpp:000000018042AF0B 83 79 04 01                             cmp     dword ptr [rcx+4], 1
il2cpp:000000018042AF0F 75 7F                                   jnz     short loc_18042AF90
il2cpp:000000018042AF11 33 C9                                   xor     ecx, ecx        ; this
il2cpp:000000018042AF13 E8 98 31 00 00                          call    DEYU_AssetHandleUtility_TempAssetHandle_object_$$get_IsPersistentAsset
il2cpp:000000018042AF18 84 C0                                   test    al, al
il2cpp:000000018042AF1A 75 74                                   jnz     short loc_18042AF90
```

`(74)  jz` 改 `(75)  jnz`

(由于逻辑短路，改 `000000018042AF0F` 的即可)



`PrototypingManagers.DebugConsoleBase$$Start+31`

```asm
il2cpp:00000001806E6CE5
il2cpp:00000001806E6CE5                         loc_1806E6CE5:                          ; CODE XREF: PrototypingManagers_DebugConsoleBase$$Start+10↑j
il2cpp:00000001806E6CE5 33 C9                                   xor     ecx, ecx        ; this
il2cpp:00000001806E6CE7 E8 C4 73 D4 FF                          call    DEYU_AssetHandleUtility_TempAssetHandle_object_$$get_IsPersistentAsset
il2cpp:00000001806E6CEC 48 8B CB                                mov     rcx, rbx        ; this
il2cpp:00000001806E6CEF 84 C0                                   test    al, al
il2cpp:00000001806E6CF1 74 16                                   jz     short loc_1806E6D09
il2cpp:00000001806E6CF3 48 8B 03                                mov     rax, [rbx]
il2cpp:00000001806E6CF6 48 8B 90 80 01 00 00                    mov     rdx, [rax+180h]
il2cpp:00000001806E6CFD 48 83 C4 20                             add     rsp, 20h
il2cpp:00000001806E6D01 5B                                      pop     rbx
il2cpp:00000001806E6D02 48 FF A0 78 01 00 00                    jmp     qword ptr [rax+178h]
```

`(74)  jz` 改 `(75)  jnz`



`PrototypingManagers.MainSceneDebugConsole$$OnStart+12A`

```asm
il2cpp:00000001806EF45D
il2cpp:00000001806EF45D                         loc_1806EF45D:                          ; CODE XREF: PrototypingManagers_MainSceneDebugConsole$$OnStart+10↑j
il2cpp:00000001806EF45D                                                                 ; DATA XREF: .rdata:0000000183F0B654↓o ...
il2cpp:00000001806EF45D 48 89 5C 24 30                          mov     [rsp+28h+arg_0], rbx
il2cpp:00000001806EF462 33 C9                                   xor     ecx, ecx        ; this
il2cpp:00000001806EF464 48 89 6C 24 38                          mov     [rsp+28h+arg_8], rbp
il2cpp:00000001806EF469 48 89 7C 24 40                          mov     [rsp+28h+arg_10], rdi
il2cpp:00000001806EF46E 4C 89 74 24 48                          mov     [rsp+28h+arg_18], r14
il2cpp:00000001806EF473 E8 38 EC D3 FF                          call    DEYU_AssetHandleUtility_TempAssetHandle_object_$$get_IsPersistentAsset
il2cpp:00000001806EF478 84 C0                                   test    al, al
il2cpp:00000001806EF47A 74 47                                   jz     short loc_1806EF4C3
il2cpp:00000001806EF47C 48 8B 05 7D 1E B1 03                    mov     rax, cs:GameData_Utils_SaveManagement_TypeInfo ; GameData.Utils.SaveManagement_TypeInfo
il2cpp:00000001806EF483 83 B8 E0 00 00 00 00                    cmp     dword ptr [rax+0E0h], 0
il2cpp:00000001806EF48A 75 0F                                   jnz     short loc_1806EF49B
il2cpp:00000001806EF48C 48 8B C8                                mov     rcx, rax
il2cpp:00000001806EF48F E8 2C E0 C4 FF                          call    il2cpp_runtime_class_init
il2cpp:00000001806EF494 48 8B 05 65 1E B1 03                    mov     rax, cs:GameData_Utils_SaveManagement_TypeInfo ; GameData.Utils.SaveManagement_TypeInfo
```

`(74)  jz` 改 `(75)  jnz`



`Common_DialogUtility_DialogPannel$$OnGUI+46`

```asm
il2cpp:00000001807DC30D
il2cpp:00000001807DC30D                         loc_1807DC30D:                          ; CODE XREF: Common_DialogUtility_DialogPannel$$OnGUI+10↑j
il2cpp:00000001807DC30D 33 C9                                   xor     ecx, ecx        ; this
il2cpp:00000001807DC30F E8 9C 1D C5 FF                          call    DEYU_AssetHandleUtility_TempAssetHandle_object_$$get_IsPersistentAsset
il2cpp:00000001807DC314 84 C0                                   test    al, al
il2cpp:00000001807DC316 0F 84 2E 01 00 00                       jz      loc_1807DC44A
il2cpp:00000001807DC31C 48 8B 0D ED 0E A6 03                    mov     rcx, cs:PrototypingManagers_GlobalDebugConsole_TypeInfo ; PrototypingManagers.GlobalDebugConsole_TypeInfo
il2cpp:00000001807DC323 83 B9 E0 00 00 00 00                    cmp     dword ptr [rcx+0E0h], 0
il2cpp:00000001807DC32A 75 05                                   jnz     short loc_1807DC331
il2cpp:00000001807DC32C E8 8F 11 B6 FF                          call    il2cpp_runtime_class_init
```

`(0F 84)  jz` 改 `(0F 85)  jnz`



`DLC3_MusicGameStartPannel$$OnGUI+8E`

```asm
il2cpp:00000001804090A5
il2cpp:00000001804090A5                         loc_1804090A5:                          ; CODE XREF: DLC3_MusicGameStartPannel$$OnGUI+10↑j
il2cpp:00000001804090A5 33 C9                                   xor     ecx, ecx        ; this
il2cpp:00000001804090A7 E8 04 50 02 00                          call    DEYU_AssetHandleUtility_TempAssetHandle_object_$$get_IsPersistentAsset
il2cpp:00000001804090AC 84 C0                                   test    al, al
il2cpp:00000001804090AE 0F 84 DB 01 00 00                       jz      loc_18040928F
il2cpp:00000001804090B4 48 8B 0D 55 41 E3 03                    mov     rcx, cs:PrototypingManagers_GlobalDebugConsole_TypeInfo ; PrototypingManagers.GlobalDebugConsole_TypeInfo
il2cpp:00000001804090BB 83 B9 E0 00 00 00 00                    cmp     dword ptr [rcx+0E0h], 0
il2cpp:00000001804090C2 75 05                                   jnz     short loc_1804090C9
il2cpp:00000001804090C4 E8 F7 43 F3 FF                          call    il2cpp_runtime_class_init
```

`(0F 84)  jz` 改 `(0F 85)  jnz`



`DayScene.UI.RogueLike.DLC5_RogueLikePurchasePanel$$OnGUI+31B`

```asm
il2cpp:000000018045BB42
il2cpp:000000018045BB42                         loc_18045BB42:                          ; CODE XREF: DayScene_UI_RogueLike_DLC5_RogueLikePurchasePanel$$OnGUI+21↑j
il2cpp:000000018045BB42 45 33 F6                                xor     r14d, r14d
il2cpp:000000018045BB45 44 89 B4 24 00 01 00 00                 mov     [rsp+0E8h+arg_10], r14d
il2cpp:000000018045BB4D 0F 57 C0                                xorps   xmm0, xmm0
il2cpp:000000018045BB50 0F 11 44 24 58                          movups  xmmword ptr [rsp+0E8h+var_90], xmm0
il2cpp:000000018045BB55 0F 11 44 24 68                          movups  xmmword ptr [rsp+0E8h+var_90+10h], xmm0
il2cpp:000000018045BB5A 4C 89 B4 24 98 00 00 00                 mov     [rsp+0E8h+successAdd], r14
il2cpp:000000018045BB62 33 C9                                   xor     ecx, ecx        ; this
il2cpp:000000018045BB64 E8 47 25 FD FF                          call    DEYU_AssetHandleUtility_TempAssetHandle_object_$$get_IsPersistentAsset
il2cpp:000000018045BB69 84 C0                                   test    al, al
il2cpp:000000018045BB6B 0F 84 B5 11 00 00                       jz      loc_18045CD26
il2cpp:000000018045BB71 48 8B 0D 98 16 DE 03                    mov     rcx, cs:PrototypingManagers_GlobalDebugConsole_TypeInfo ; PrototypingManagers.GlobalDebugConsole_TypeInfo
il2cpp:000000018045BB78 44 39 B1 E0 00 00 00                    cmp     [rcx+0E0h], r14d
il2cpp:000000018045BB7F 75 05                                   jnz     short loc_18045BB86
```

`(0F 84)  jz` 改 `(0F 85)  jnz`



`NightScene.PartnerUtility.PartnerManager$$OnGUI+52`

```asm
il2cpp:00000001804E7069
il2cpp:00000001804E7069                         loc_1804E7069:                          ; CODE XREF: NightScene_PartnerUtility_PartnerManager$$OnGUI+10↑j
il2cpp:00000001804E7069 33 C9                                   xor     ecx, ecx        ; this
il2cpp:00000001804E706B E8 40 70 F4 FF                          call    DEYU_AssetHandleUtility_TempAssetHandle_object_$$get_IsPersistentAsset
il2cpp:00000001804E7070 84 C0                                   test    al, al
il2cpp:00000001804E7072 0F 84 95 01 00 00                       jz      loc_1804E720D
il2cpp:00000001804E7078 48 8B 0D 91 61 D5 03                    mov     rcx, cs:PrototypingManagers_GlobalDebugConsole_TypeInfo ; PrototypingManagers.GlobalDebugConsole_TypeInfo
il2cpp:00000001804E707F 83 B9 E0 00 00 00 00                    cmp     dword ptr [rcx+0E0h], 0
il2cpp:00000001804E7086 75 05                                   jnz     short loc_1804E708D
il2cpp:00000001804E7088 E8 33 64 E5 FF                          call    il2cpp_runtime_class_init
```

`(0F 84)  jz` 改 `(0F 85)  jnz`

### LaunchMode

修改 `Touhou Mystia Izakaya\Touhou Mystia Izakaya_Data\StreamingAssets\LaunchMode.txt`  `LaunchMode` 改为 `Full` 也可以启动控制台，但启动前需要登录测试人员账号，验证服务器已无效

## Characters

### 角色生成

> DEYU：@MetaMiku SpecialGuests无法Copy，但是如果你只是想要场景里面存在一个不能互动的小人的话，可以用 SceneDirector.Instance.SpawnCharacter

```c#
// Common.SceneDirector
// Token: 0x06005062 RID: 20578 RVA: 0x00002053 File Offset: 0x00000253
[Token(Token = "0x6005062")]
[Address(RVA = "0x7E39F0", Offset = "0x7E23F0", VA = "0x1807E39F0")]
public void SpawnCharacter(SceneDirector.Identity characterType, int characterId, Vector2 startPosition, string label)
{
}

```

使用例：

```c#
Common.SceneDirector.Instance.SpawnCharacter(Common.SceneDirector.Identity.Special, 14, new Vector2(0, 0), "test");
Common.SceneDirector.Instance.characterCollection["test"].UpdateInputVelocity(new Vector2(1, 0));
```

### 添加碰撞体

`SpawnCharacter` 在 `Common.SceneDirector$$SpawnCharacter+295` 处调用了 `Common.CharacterUtility.CharacterControllerUnit$$Initialize`

```c
Common_CharacterUtility_CharacterControllerUnit__Initialize(v29, v32, v32->fields.moveSpeedMultiplier, 0, 0LL);
```

其中，

```c#
// Common.CharacterUtility.CharacterControllerUnit
// Token: 0x06006023 RID: 24611 RVA: 0x00002053 File Offset: 0x00000253
[Token(Token = "0x6006023")]
[Address(RVA = "0x8C47F0", Offset = "0x8C31F0", VA = "0x1808C47F0")]
public void Initialize(CharacterSpriteSetCompact compactOrFullvisual, float moveSpeedMultiplier, bool shouldTurnOnCollider)
{
}

```

```c
void Common_CharacterUtility_CharacterControllerUnit__Initialize(
        Common_CharacterUtility_CharacterControllerUnit_o *this,
        GameData_Core_Collections_CharacterUtility_CharacterSpriteSetCompact_o *compactOrFullvisual,
        float moveSpeedMultiplier,
        bool shouldTurnOnCollider,
        const MethodInfo *method)
```

不难发现正常逻辑下 `shouldTurnOnCollider` 被写死为 `false`，因此如果需要考虑添加碰撞体，可以 hook `Common.CharacterUtility.CharacterControllerUnit.Initialize` 改入参 `shouldTurnOnCollider` 以加入碰撞体，并在之后通过 `Common.CharacterUtility.CharacterControllerUnit.UpdateColliderStatus` 修改碰撞状态

```c#
// Common.CharacterUtility.CharacterControllerUnit
// Token: 0x06006028 RID: 24616 RVA: 0x00002053 File Offset: 0x00000253
[Token(Token = "0x6006028")]
[Address(RVA = "0x8C6650", Offset = "0x8C5050", VA = "0x1808C6650")]
public void UpdateColliderStatus(bool shouldOpen)
{
}

```

```c
void Common_CharacterUtility_CharacterControllerUnit__UpdateColliderStatus(
        Common_CharacterUtility_CharacterControllerUnit_o *this,
        bool shouldOpen,
        const MethodInfo *method)
```

### 屏蔽部分碰撞

使用 `Physics.IgnoreCollision` 可以轻松实现

```c#
public static void IgnoreCollision(Collider collider1, Collider collider2, bool ignore = true);
```

例

```c#
UnityEngine.Physics2D.IgnoreCollision(
    Common.SceneDirector.Instance.characterCollection[NightKyoukoManager.KYOUKO_ID].cl2d,
    Common.SceneDirector.Instance.characterCollection["Self"].cl2d, // 小碎骨
    false);
```

[Unity - Scripting API_ Physics.IgnoreCollision](https://docs.unity3d.com/2021.3/Documentation/ScriptReference/Physics.IgnoreCollision.html)

### 添加高度处理器

角色在经过部分场景，如上桥、上楼等地方，有一种高度效果。经测试，该高度效果实打实地作用在了 `Common.CharacterUtility.CharacterControllerUnit.rb2d.position` 并非只是视觉效果

可以注意到有下面几个地方注册了高度输入处理器 `HeightBlendedInputProcessorComponent`

`DayScene.Input.DayScenePlayerInputGenerator$$UpdateCharacter+AD8`

`DayScene.Input.DayScenePlayerInputGenerator$$UpdateCharacter+C04`

`NightScene.Input.WorkScenePlayerInputGenerator$$Initialize+890`

例如

```c
v94 = Common_CharacterUtility_CharacterControllerUnit__AddInputProcessor_object_(
    Character_k__BackingField,
    Method_Common_CharacterUtility_CharacterControllerUnit_AddInputProcessor_HeightBlendedInputProcessorComponent___);
if ( !v94 )
    goto LABEL_58;
Common_CharacterUtility_HeightBlendedInputProcessorComponent__Initialize(
    (Common_CharacterUtility_HeightBlendedInputProcessorComponent_o *)v94,
    heightMap,
    0LL);
```

因此仿照此先注册高度输入处理器，再执行初始化，其中 `heightMap` 可以考虑从 `Self` (小碎骨) 的高度输入处理器中取

```c#
var characterUnit = Common.SceneDirector.Instance.characterCollection[KYOUKO_ID];
var heightProcessor = characterUnit.AddInputProcessor<Common.CharacterUtility.HeightBlendedInputProcessorComponent>();

var selfUnit = Common.SceneDirector.Instance.characterCollection["Self"];
var selfHeightProcessor = selfUnit.gameObject.GetComponent<Common.CharacterUtility.HeightBlendedInputProcessorComponent>();
heightProcessor.Initialize(selfHeightProcessor.heightMap);
```

也可以不通过 `Self` 获取 heightMap

```c#
var characterUnit = Common.SceneDirector.Instance.characterCollection[KyoukoManager.KYOUKO_ID];
var heightProcessor = characterUnit.AddInputProcessor<Common.CharacterUtility.HeightBlendedInputProcessorComponent>();
heightProcessor.Initialize(DayScene.SceneManager.Instance.CurrentActiveMap.height); // 白天
heightProcessor.Initialize(NightScene.MapManager.Instance.height); // 夜晚
```

