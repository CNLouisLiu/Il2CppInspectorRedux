# Il2CppInspectorRedux — Arknights Endfield CN 1.2 适配记录

## 目标

支持明日方舟：终末地 CN 1.2 的 `GameAssembly.dll` + `global-metadata.dat` 解析。

- Metadata 版本：v29.0 Tag2022（Unity 2022.x IL2CPP）
- 架构：x86_64 PE (Windows)
- Header 大小：0x108

---

## 修改文件及内容

### 1. `Il2CppInspector.Common/Next/BinaryMetadata/Il2CppCodeRegistration.cs`

**问题**：`UnresolvedInstanceCallWrappers` 和 `UnresolvedStaticCallPointers` 被错误地标记为 v29/v31 Tag2022 存在，导致 `StructSize` 多算 16 字节，CodeRegistration 起始地址计算错误，字段全部读偏。

**修复**：移除这两个字段的 `[VersionCondition(EqualTo = "29.0", IncludingTag = "2022")]` 和 `[VersionCondition(EqualTo = "31.0", IncludingTag = "2022")]`，仅保留 `[VersionCondition(GreaterThanOrEqual = "35.0")]`。

### 2. `Il2CppInspector.Common/Next/Metadata/Il2CppTypeDefinition.cs`

**问题**：v29-2022 的 `Il2CppTypeDefinition` 比标准 v29 多 4 字节（92 vs 88），导致 `metadata.Types.Length` 计算错误（63000 vs 60261），MetadataRegistration 搜索失败。

**修复**：在 `InterfaceOffsetsStart` 和 `MethodCount` 之间添加 `UnknownIndex2022` 字段（int, 4字节），仅在 v29/v31 Tag2022 时存在。该字段的值为小整数（0, 2, 15 等），推测与 packing/native size 相关。

```csharp
[VersionCondition(EqualTo = "29.0", IncludingTag = "2022")]
[VersionCondition(EqualTo = "31.0", IncludingTag = "2022")]
public int UnknownIndex2022 { get; private set; }
```

### 3. `Il2CppInspector.Common/IL2CPP/ImageScan.cs`

**问题**：`FindCodeRegistration()` 中 `expectedImageCount == imagesCount` 精确匹配失败。该二进制的 CodeGenModules 指针指向数组中间（array[99]），而非 array[0]，导致 count=165 ≠ imagesCount=264。

**修复**：将精确匹配改为范围检查：

```csharp
// 旧：if (expectedImageCount == imagesCount)
// 新：
if (expectedImageCount > 0 && (long)expectedImageCount <= imagesCount)
```

### 4. `Il2CppInspector.Common/IL2CPP/Il2CppInspector.cs`

**问题**：`FieldDefaultValue`、`ParameterDefaultValue` 和 field offset 字典在遇到重复 key 时抛出 `ArgumentException`。

**修复**：将 `.Add()` 改为 `.TryAdd()`，忽略重复项。

### 5. `Il2CppInspector.Common/Reflection/TypeModel.cs`

**问题**：构建类型时，方法泛型参数的 `MethodsByDefinitionIndex[container.OwnerIndex]` 返回 null（方法尚未创建），导致 `NullReferenceException`。

**修复**：添加 null 检查，owner 为 null 时返回 null。

### 6. `Il2CppInspector.Common/Reflection/TypeInfo.cs`

**问题**：部分类型的 `InterfacesIndex`、`NestedTypeIndex`、`FieldIndex`、`MethodIndex` 为 -1（无数据），但 count > 0 时会越界访问数组。

**修复**：
- 接口、嵌套类型循环添加边界检查
- 字段、方法循环添加 `index >= 0 && index + count <= array.Length` 前置条件
- 泛型参数名称拼接时过滤 null 项

---

## 之前会话已完成的修改

### `Il2CppInspector.Common/Next/Metadata/Il2CppGlobalMetadataHeader.cs`

添加 `CodeGenModulesOffset` / `CodeGenModulesSize` 字段（v29/v31 Tag2022），使 header 大小从 0x100 增加到 0x108。

### `Il2CppInspector.Common/IL2CPP/Metadata.cs`

添加 v29/v31 Tag2022 子版本检测逻辑，当 header 大小为 0x108 时设置 Tag2022。

---

## 关键数据（IDA 验证）

| 项目 | 值 |
|---|---|
| Image base (Global offset) | `0x7FF9A4F90C00` |
| CodeRegistration RVA | `0x0A31FAC0` |
| MetadataRegistration RVA | `0x0A31FCD0` |
| CodeGenModulesCount | 165 |
| ImagesCount (metadata) | 264 |
| TypeDefinitionsCount | 60261 |
| FieldsCount | 278566 |
| Il2CppTypeDefinition size | 92 bytes |
| Il2CppFieldDefinition size | 12 bytes |
