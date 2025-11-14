# 系統設計文檔

## 執行摘要

本文檔描述輔大失物招領應用的完整系統設計。該設計採用分層架構（5層）和 MVVM 模式，確保代碼質量、可維護性和性能。

---

## 1. 系統架構總體設計

### 1.1 架構模式

```
┌────────────────────────────────────┐
│  4-Layer MVVM + Repository Pattern │
│                                    │
│  Layer 1: Presentation (Compose)   │
│  Layer 2: ViewModel (State)        │
│  Layer 3: Repository (Data Access)│
│  Layer 4: Remote (Firebase)        │
└────────────────────────────────────┘
```

### 1.2 數據流原則

**單向數據流 (Unidirectional Data Flow)**:
```
Firebase
    ↓ (read)
Repository
    ↓ (process)
ViewModel
    ↓ (expose as StateFlow)
Compose UI
    ↓ (user action)
ViewModel
    ↓ (command)
Repository
    ↓ (write)
Firebase
```

### 1.3 核心組件

| 組件 | 職責 | 檔案 |
|------|------|------|
| **Repository** | 數據訪問層 | `repository/LostFoundRepository.kt` |
| **ViewModel** | 狀態管理層 | `ui/viewmodel/MainViewModel.kt` |
| **Screens** | UI 展示層 | `ui/screens/*.kt` |
| **Models** | 數據模型 | `models/FirebaseModels.kt` |
| **Services** | 業務服務 | `services/` |

---

## 2. 核心模塊設計

### 2.1 Repository 層 (LostFoundRepository.kt)

**目的**: 封裝所有 Firebase 操作，提供統一的數據訪問接口

**設計特點**:
```kotlin
// 1. 統一的返回類型 (Result<T>)
suspend fun getAllBuildings(): Result<List<Building>>

// 2. 協程支持
withContext(Dispatchers.IO)

// 3. 完整的錯誤處理
.onSuccess { ... }
.onFailure { ... }

// 4. 日誌記錄
Log.d(TAG, "✓ 成功獲取 ${items.size} 件物品")
```

**主要操作**:

1. **建築物管理**
   - `getAllBuildings()` - 獲取所有建築物
   - `getBuildingById(id)` - 獲取單個建築物
   - `initializeBuildings(list)` - 初始化 41 棟建築物

2. **物品管理**
   - `getAllItems()` - 獲取所有物品
   - `getItemsByBuilding(id)` - 獲取特定建築物的物品
   - `getItemCountByBuilding()` ⭐ - 獲取物品計數（只返回有物品的）
   - `addItem(item)` - 新增物品
   - `updateItemStatus(id, status)` - 更新狀態

3. **搜尋**
   - `searchItems(query)` - 全文搜尋

4. **圖片**
   - `uploadImage(uri)` - 上傳圖片到 Storage
   - `deleteImage(url)` - 刪除圖片

**關鍵 API: getItemCountByBuilding()**

這是解決「圖釘不顯示」的關鍵:

```kotlin
suspend fun getItemCountByBuilding(): Result<Map<String, Int>> {
    val buildings = getAllBuildings()  // 獲取所有建築物
    val itemCounts = mutableMapOf<String, Int>()

    buildings.forEach { building ->
        val items = getItemsByBuilding(building.id)
        if (items.isNotEmpty()) {  // 只加入有物品的
            itemCounts[building.id] = items.size
        }
    }

    return Result.success(itemCounts)  // 只包含有物品的建築物
}
```

**優勢**:
- ✅ Repository 負責過濾邏輯
- ✅ UI 層無需修改，自動只顯示有物品的圖釘
- ✅ 減少 UI 層複雜性

### 2.2 ViewModel 層 (MainViewModel.kt)

**目的**: 管理 UI 狀態，與 Repository 通信

**設計特點**:

1. **不可變狀態流**
   ```kotlin
   private val _buildings = MutableStateFlow<List<Building>>(emptyList())
   val buildings: StateFlow<List<Building>> = _buildings.asStateFlow()
   ```

2. **單一真實來源**
   - 所有狀態都來自 Repository
   - 避免狀態重複

3. **協程管理**
   - 使用 `viewModelScope`
   - 自動取消未完成的協程

4. **完整的錯誤處理**
   ```kotlin
   val itemsError: StateFlow<String?> = _itemsError.asStateFlow()
   ```

**主要方法**:

| 方法 | 用途 |
|------|------|
| `loadBuildings()` | 加載建築物列表 |
| `loadItems()` | 加載所有物品 |
| `loadItemsByBuilding(id)` | 加載特定建築物的物品 |
| `loadItemCountByBuilding()` | 加載物品計數（用於地圖） |
| `addItem(item)` | 新增物品 |
| `updateItemStatus(id, status)` | 更新物品狀態 |
| `searchItems(query)` | 搜尋物品 |

**狀態分類**:

```kotlin
// 建築物相關
val buildings: StateFlow<List<Building>>
val buildingsLoading: StateFlow<Boolean>
val buildingsError: StateFlow<String?>

// 物品相關
val items: StateFlow<List<LostFoundItem>>
val itemsLoading: StateFlow<Boolean>
val itemsError: StateFlow<String?>

// 地圖物品計數（關鍵）
val itemCountByBuilding: StateFlow<Map<String, Int>>
val itemCountLoading: StateFlow<Boolean>
val itemCountError: StateFlow<String?>

// 搜尋
val searchResults: StateFlow<List<LostFoundItem>>
val searchQuery: StateFlow<String>
val searchLoading: StateFlow<Boolean>
```

### 2.3 UI 層 (Screens)

**設計原則**:

1. **聲明式**
   - 描述 UI 應該是什麼樣子
   - 不是如何改變它

2. **響應式**
   - 訂閱 StateFlow
   - 自動重組

3. **簡潔**
   - 最小化 UI 邏輯
   - 複雜邏輯放在 ViewModel

**MapScreen 示例**:

```kotlin
@Composable
fun MapScreen(navController: NavController) {
    // 1. 創建 ViewModel
    val viewModel: MainViewModel = remember {
        val repository = LostFoundRepository(firestore, storage)
        MainViewModel(repository)
    }

    // 2. 訂閱狀態
    val buildings by viewModel.buildings.collectAsState()
    val itemCountByBuilding by viewModel.itemCountByBuilding.collectAsState()
    val itemCountLoading by viewModel.itemCountLoading.collectAsState()

    // 3. 初始化
    LaunchedEffect(Unit) {
        viewModel.loadBuildings()
        viewModel.loadItemCountByBuilding()
    }

    // 4. 基於狀態渲染 UI
    if (itemCountLoading) {
        LoadingScreen()
    } else {
        GoogleMap {
            // 只遍歷有物品的建築物
            itemCountByBuilding.forEach { (buildingId, count) ->
                val building = buildings.find { it.id == buildingId }
                Marker(position = LatLng(building.latitude, building.longitude))
            }
        }
    }
}
```

**關鍵流程**:

1. **狀態訂閱** → 自動監聽 StateFlow 變化
2. **初始化加載** → LaunchedEffect 觸發數據加載
3. **自動重組** → StateFlow 更新時自動刷新 UI
4. **智能渲染** → 只显示有物品的圖釘

---

## 3. 地圖圖釘問題分析

### 3.1 問題描述

**現象**: 地圖上沒有顯示任何圖釘

**原因分析**:

| 層級 | 問題 |
|------|------|
| Repository | `itemCountByBuilding` 方法返回空 Map |
| ViewModel | 沒有正確調用 Repository 方法 |
| UI | 使用了錯誤的狀態或邏輯 |
| Firebase | 沒有物品數據 |

### 3.2 解決方案

**第 1 步: Repository 層修復**

```kotlin
suspend fun getItemCountByBuilding(): Result<Map<String, Int>> {
    // 1. 獲取所有建築物
    val buildings = getAllBuildings()

    // 2. 對每個建築物查詢物品
    val itemCounts = mutableMapOf<String, Int>()
    buildings.forEach { building ->
        val items = getItemsByBuilding(building.id)
        // 3. 只在有物品時加入計數
        if (items.isNotEmpty()) {
            itemCounts[building.id] = items.size
        }
    }

    // 4. 返回結果（只包含有物品的建築物）
    return Result.success(itemCounts)
}
```

**第 2 步: ViewModel 層修復**

```kotlin
fun loadItemCountByBuilding() {
    viewModelScope.launch {
        _itemCountLoading.value = true

        val result = repository.getItemCountByBuilding()
        result
            .onSuccess { counts ->
                _itemCountByBuilding.value = counts
                Log.d(TAG, "✓ 加載物品計數：${counts.size} 棟建築物有物品")
            }
            .onFailure { error ->
                _itemCountError.value = error.message
            }

        _itemCountLoading.value = false
    }
}
```

**第 3 步: UI 層修復**

```kotlin
// MapScreen 中
LaunchedEffect(Unit) {
    viewModel.loadBuildings()
    viewModel.loadItemCountByBuilding()  // ← 必須調用
}

GoogleMap {
    // 遍歷 itemCountByBuilding（只包含有物品的）
    itemCountByBuilding.forEach { (buildingId, count) ->
        val building = buildings.find { it.id == buildingId }
        if (building != null && count > 0) {
            Marker(...)  // 顯示圖釘
        }
    }
}
```

### 3.3 驗證步驟

1. **檢查 Firebase 數據**
   ```
   Firestore → lostItems → 應該有文檔
   ```

2. **檢查日誌**
   ```
   D/MainViewModel: ✓ 加載物品計數：3 棟建築物有物品
   D/MapScreen: ✓ 地圖上顯示 3 個圖釘
   ```

3. **調試應用狀態**
   ```kotlin
   viewModel.debugPrintState()
   // 應輸出: 有物品的建築物: 3
   ```

---

## 4. 數據流設計

### 4.1 新增物品流程

```
1. AddItemScreen
   ├─ 用戶選擇建築物
   ├─ 用戶拍照
   └─ 用戶輸入信息
         ↓
2. 點擊「新增」
         ↓
3. MainViewModel.addItem(item)
   ├─ 調用 Repository.addItem()
   └─ 更新 _items StateFlow
         ↓
4. LostFoundRepository.addItem()
   ├─ 生成 itemId
   ├─ 添加時間戳
   └─ 保存到 Firestore
         ↓
5. Firestore
   ├─ 保存文檔
   └─ 返回成功
         ↓
6. HomeScreen / MapScreen / SearchScreen
   ├─ 訂閱 items StateFlow
   ├─ 自動接收新物品
   └─ UI 重組並刷新
```

### 4.2 搜尋流程

```
1. SearchScreen
   ├─ 用戶輸入搜尋詞
         ↓
2. MainViewModel.searchItems(query)
   ├─ 調用 Repository.searchItems(query)
   └─ 更新 _searchResults StateFlow
         ↓
3. LostFoundRepository.searchItems(query)
   ├─ 獲取所有物品
   ├─ 在內存中搜尋
   │  ├─ 物品名稱包含
   │  ├─ 物品描述包含
   │  └─ 物品標籤包含
   └─ 返回搜尋結果
         ↓
4. Compose UI
   ├─ 訂閱 searchResults StateFlow
   ├─ 自動重組
   └─ 顯示搜尋結果
```

---

## 5. 性能設計

### 5.1 查詢優化

1. **建築物列表**
   - 單次加載全部 (41 棟)
   - 結果存儲在 ViewModel
   - 不需要重複查詢

2. **物品計數**
   - 並行查詢多個建築物
   - 使用 Dispatcher.IO
   - 結果緩存在 StateFlow

3. **搜尋**
   - 在內存中搜尋 (不查詢 Firestore)
   - 支持模糊匹配
   - 實時搜尋結果

### 5.2 資源管理

1. **協程**
   - 使用 viewModelScope（自動取消）
   - 避免內存洩漏

2. **網絡**
   - 初始化時加載數據
   - 用戶操作時更新
   - 避免過度查詢

3. **存儲**
   - 圖片存儲在 Firebase Storage
   - 使用 JPEG 格式壓縮
   - 按建築物和日期組織

---

## 6. 錯誤處理策略

### 6.1 Repository 層

```kotlin
suspend fun getAllItems(): Result<List<LostFoundItem>> {
    return withContext(Dispatchers.IO) {
        try {
            val snapshot = firestore.collection(ITEMS_COLLECTION).get().await()
            val items = snapshot.toObjects(LostFoundItem::class.java)
            Result.success(items)  // ✓ 成功
        } catch (e: Exception) {
            Result.failure(e)      // ✗ 失敗
        }
    }
}
```

### 6.2 ViewModel 層

```kotlin
fun loadItems() {
    viewModelScope.launch {
        val result = repository.getAllItems()

        result.onSuccess { items ->
            _items.value = items
            Log.d(TAG, "✓ 成功加載 ${items.size} 件物品")
        }

        result.onFailure { error ->
            _itemsError.value = error.message  // 保存錯誤信息
            Log.e(TAG, "✗ 加載物品失敗: ${error.message}")
        }
    }
}
```

### 6.3 UI 層

```kotlin
@Composable
fun ItemsScreen() {
    val viewModel = remember { MainViewModel(repository) }
    val error by viewModel.itemsError.collectAsState()

    if (error != null) {
        ErrorDialog(
            title = "錯誤",
            message = error,
            onDismiss = { /* 清除錯誤 */ }
        )
    }
}
```

---

## 7. 測試策略

### 7.1 單元測試

```kotlin
// 測試 Repository
@Test
fun testGetAllBuildings() = runTest {
    val repository = LostFoundRepository(firestore, storage)
    val result = repository.getAllBuildings()

    assert(result.isSuccess)
    assert(result.getOrNull()?.size == 41)
}
```

### 7.2 集成測試

```kotlin
// 測試 ViewModel + Repository
@Test
fun testLoadBuildings() = runTest {
    val viewModel = MainViewModel(repository)
    viewModel.loadBuildings()

    advanceUntilIdle()
    assert(viewModel.buildings.value.size == 41)
}
```

---

## 8. 部署架構

### 8.1 Firebase 配置

```
Project ID: fju-lost-found
Database: Firestore
Storage: Firebase Storage

Collections:
├── buildings/ (41 文檔)
├── lostItems/ (N 文檔)
└── users/ (待實現)
```

### 8.2 本地存儲

```
SharedPreferences (via DataStore):
├── api_call_count (Int)
├── last_reset_date (Long)
└── is_quota_exceeded (Boolean)
```

---

## 9. 最佳實踐

### 9.1 代碼組織

```
✅ DO:
- Repository 只負責數據訪問
- ViewModel 只負責狀態管理
- UI 只負責展示

❌ DON'T:
- 在 UI 中直接調用 Firebase
- 在 Repository 中處理 UI 邏輯
- 混合職責
```

### 9.2 狀態管理

```
✅ DO:
- 使用 StateFlow 表示狀態
- 使用 MutableStateFlow 更新
- 訂閱 collectAsState()

❌ DON'T:
- 使用全局變量
- 直接修改 UI 組件狀態
- 多個真實來源
```

### 9.3 錯誤處理

```
✅ DO:
- 使用 Result<T> 類型
- .onSuccess / .onFailure 模式
- 保存錯誤信息到狀態

❌ DON'T:
- 忽略異常
- 直接拋出異常
- 不記錄錯誤
```

---

## 10. 總結

### 核心成功要素

1. **分層設計** ✅
   - Repository 封裝數據訪問
   - ViewModel 管理狀態
   - UI 展示數據

2. **單一真實來源** ✅
   - 所有狀態在 ViewModel
   - StateFlow 自動同步
   - 避免狀態不一致

3. **統一數據流** ✅
   - Firebase → Repository → ViewModel → UI
   - 用戶交互反向流動
   - 清晰可追蹤

4. **強大的錯誤處理** ✅
   - Result<T> 類型
   - 統一異常管理
   - 用戶友好的錯誤信息

5. **完整的日誌** ✅
   - 追蹤所有重要操作
   - 便於調試和監控
   - 性能分析

---

**系統設計文檔 v2.0**
**最後更新**: 2024-11-13
**架構模式**: MVVM + Repository Pattern
**狀態管理**: ViewModel + StateFlow
