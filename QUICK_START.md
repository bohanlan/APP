# 快速開發指南

## 🎯 核心概念

### Repository 層 (數據訪問)
```kotlin
// 位置: repository/LostFoundRepository.kt
// 職責: 封裝所有 Firebase 操作

// 建築物相關
getAllBuildings(): Result<List<Building>>
getBuildingById(id): Result<Building?>

// 物品相關
getAllItems(): Result<List<LostFoundItem>>
getItemsByBuilding(buildingId): Result<List<LostFoundItem>>
getItemCountByBuilding(): Result<Map<String, Int>>  // ⭐ 關鍵
addItem(item): Result<String>
updateItemStatus(itemId, status): Result<Unit>

// 搜尋
searchItems(query): Result<List<LostFoundItem>>

// 圖片
uploadImage(uri): Result<String>
```

### ViewModel 層 (狀態管理)
```kotlin
// 位置: ui/viewmodel/MainViewModel.kt
// 職責: 管理 UI 狀態，調用 Repository

// 狀態流 (UI 訂閱)
val buildings: StateFlow<List<Building>>
val items: StateFlow<List<LostFoundItem>>
val itemCountByBuilding: StateFlow<Map<String, Int>>
val searchResults: StateFlow<List<LostFoundItem>>

// 方法 (UI 調用)
loadBuildings()
loadItems()
loadItemCountByBuilding()  // 用於地圖
addItem(item)
searchItems(query)
```

### UI 層 (Screens)
```kotlin
// MapScreen 例子
val viewModel = remember { MainViewModel(repository) }

// 訂閱狀態
val buildings by viewModel.buildings.collectAsState()
val itemCountByBuilding by viewModel.itemCountByBuilding.collectAsState()

// 加載數據
LaunchedEffect(Unit) {
    viewModel.loadBuildings()
    viewModel.loadItemCountByBuilding()
}

// 渲染 UI (自動同步)
itemCountByBuilding.forEach { (buildingId, count) ->
    Marker(...)  // 只顯示有物品的圖釘
}
```

---

## 📝 常見開發任務

### 任務 1: 添加新的查詢方法

**在 Repository 中添加**:
```kotlin
suspend fun getItemsByCategory(category: String): Result<List<LostFoundItem>> {
    return withContext(Dispatchers.IO) {
        try {
            val snapshot = firestore.collection(ITEMS_COLLECTION)
                .whereEqualTo("category", category)
                .get()
                .await()
            val items = snapshot.toObjects(LostFoundItem::class.java)
            Result.success(items)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

**在 ViewModel 中添加狀態和方法**:
```kotlin
private val _itemsByCategory = MutableStateFlow<List<LostFoundItem>>(emptyList())
val itemsByCategory: StateFlow<List<LostFoundItem>> = _itemsByCategory.asStateFlow()

fun loadItemsByCategory(category: String) {
    viewModelScope.launch {
        val result = repository.getItemsByCategory(category)
        result.onSuccess { items ->
            _itemsByCategory.value = items
        }
    }
}
```

**在 UI 中使用**:
```kotlin
@Composable
fun CategoryScreen() {
    val viewModel = remember { MainViewModel(repository) }
    val items by viewModel.itemsByCategory.collectAsState()

    LaunchedEffect(Unit) {
        viewModel.loadItemsByCategory("found")
    }

    LazyColumn {
        items(items) { item ->
            ItemCard(item)
        }
    }
}
```

### 任務 2: 顯示錯誤信息

**ViewModel 中已有錯誤狀態**:
```kotlin
val itemsError: StateFlow<String?> = _itemsError.asStateFlow()
```

**UI 中使用**:
```kotlin
@Composable
fun ItemsScreen() {
    val viewModel = remember { MainViewModel(repository) }
    val error by viewModel.itemsError.collectAsState()

    if (error != null) {
        ErrorDialog(message = error)
    }
}
```

### 任務 3: 添加新的數據模型

**在 models/FirebaseModels.kt 中**:
```kotlin
@DocumentId
data class Review(
    val id: String = "",
    val itemId: String = "",
    val rating: Int = 0,
    val comment: String = "",
    val timestamp: Long = 0L
)
```

**在 Repository 中添加操作**:
```kotlin
suspend fun addReview(review: Review): Result<String> {
    // 實現...
}
```

---

## 🐛 調試技巧

### 查看應用狀態
```kotlin
// 在 ViewModel 中調用
viewModel.debugPrintState()

// 輸出示例:
// D/MainViewModel: ========== 應用狀態調試 ==========
// D/MainViewModel: 建築物數量: 41
// D/MainViewModel: 物品數量: 15
// D/MainViewModel: 有物品的建築物: 8
// D/MainViewModel: 搜尋結果: 0
```

### 檢查 Firebase 數據
```kotlin
// 在 Repository 方法中添加日誌
Log.d(TAG, "✓ 成功獲取 ${buildings.size} 棟建築物")
Log.d(TAG, "✓ 成功獲取 $buildingId 的 ${items.size} 件物品")
```

### 監控地圖圖釘
```kotlin
// MapScreen 中自動添加日誌
Log.d(TAG, "✓ 顯示圖釘: ${building.name} ($itemCount 件物品)")
Log.d(TAG, "✓ 地圖上顯示 ${itemCountByBuilding.size} 個圖釘")
```

---

## 🔗 文件導航

### Repository 層
- `repository/LostFoundRepository.kt` - 所有數據操作

### ViewModel 層
- `ui/viewmodel/MainViewModel.kt` - 主應用狀態
- `ui/viewmodel/ViewModelFactory.kt` - 工廠模式

### UI 層
- `ui/screens/MapScreen.kt` - 地圖顯示 ⭐
- `ui/screens/AddItemScreen.kt` - 新增物品
- `ui/screens/HomeScreen.kt` - 首頁
- `ui/screens/SearchScreen.kt` - 搜尋

### Models
- `models/FirebaseModels.kt` - 所有數據模型

### Services
- `services/FirebaseService.kt` - Firebase 初始化
- `services/QuotaManager.kt` - 配額管理

---

## ⚠️ 常見問題

### Q1: 為什麼地圖上沒有圖釘?
**A**: 檢查 `itemCountByBuilding` 是否為空
```kotlin
// 調試
Log.d(TAG, "itemCountByBuilding: ${itemCountByBuilding.size}")  // 應該 > 0
Log.d(TAG, "有物品的建築物: ${itemCountByBuilding.keys}")
```

**可能原因**:
1. Firebase 中沒有物品 → 添加物品
2. `getItemCountByBuilding()` 失敗 → 檢查網絡
3. 配額超出 → 檢查 QuotaManager

### Q2: 新增物品後沒有出現?
**A**: 確保調用了 `viewModel.loadItems()`
```kotlin
// 正確的流程
val success = addItem(item)  // 新增
if (success) {
    viewModel.loadItems()  // 重新加載列表
}
```

### Q3: 搜尋結果為空?
**A**: 檢查搜尋詞和物品內容是否匹配
```kotlin
// 搜尋使用模糊匹配
item.name.lowercase().contains(query.lowercase())
item.tags.any { it.lowercase().contains(query.lowercase()) }
```

---

## 📚 架構決策

### 為什麼使用 Repository 模式?
✅ 隔離 Firebase 實現細節
✅ 便於單元測試（可以 mock Repository）
✅ 代碼重用（多個 ViewModel 可以共用 Repository）

### 為什麼使用 StateFlow?
✅ 自動處理 Composable 重組
✅ 觀察者模式，自動同步
✅ 協程友好

### 為什麼 getItemCountByBuilding() 只返回有物品的?
✅ 減少地圖渲染的圖釘數量
✅ 自動過濾空建築物
✅ 更好的性能和 UX

---

## 🎓 學習資源

### Repository 模式
- 位置: `repository/LostFoundRepository.kt` (371 行)
- 特點: Result<T> 類型，統一錯誤處理

### MVVM 模式
- 位置: `ui/viewmodel/MainViewModel.kt` (331 行)
- 特點: StateFlow 狀態管理，完整的業務邏輯

### Compose 最佳實踐
- 位置: `ui/screens/` (所有屏幕)
- 特點: 聲明式 UI，狀態驅動

---

## ✅ 檢查清單 (新功能開發)

- [ ] 在 Repository 中添加數據操作
- [ ] 在 ViewModel 中添加狀態和方法
- [ ] 在 UI 中訂閱狀態並調用方法
- [ ] 添加適當的錯誤處理
- [ ] 編寫日誌語句用於調試
- [ ] 測試新功能
- [ ] 更新相關文檔

---

**快速開發指南 v1.0**
**最後更新**: 2024-11-13
