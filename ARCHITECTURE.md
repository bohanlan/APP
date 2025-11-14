# 失物招領系統 - 完整架構設計文檔

## 📋 目錄
1. [系統概述](#系統概述)
2. [架構層級](#架構層級)
3. [核心模組](#核心模組)
4. [數據流設計](#數據流設計)
5. [類別關係圖](#類別關係圖)
6. [API 設計](#api-設計)

---

## 系統概述

### 應用名稱
輔大失物招領系統（FJU Lost & Found）

### 核心功能
- 📱 遺失物品報告與管理
- 🗺️ 地圖搜尋與可視化
- 🔍 全文搜尋功能
- 📍 建築物位置管理
- 🖼️ 物品圖片上傳
- 📊 配額管理與監控

### 技術棧
```
前端: Jetpack Compose
後端: Firebase (Firestore + Storage + Auth)
地圖: Google Maps Android API
狀態管理: ViewModel + StateFlow
架構模式: MVVM + Repository
```

---

## 架構層級

### 分層架構（5層設計）

```
┌─────────────────────────────────────┐
│         UI Layer (Screens)          │  ← AddItemScreen, MapScreen, etc.
├─────────────────────────────────────┤
│       ViewModel Layer               │  ← MainViewModel
├─────────────────────────────────────┤
│      Repository Layer               │  ← LostFoundRepository
├─────────────────────────────────────┤
│      Service Layer                  │  ← Firebase, QuotaManager
├─────────────────────────────────────┤
│       Model Layer                   │  ← Building, LostFoundItem
└─────────────────────────────────────┘
```

### 各層職責

#### 1️⃣ UI 層 (Presentation Layer)
**位置**: `ui/screens/`

**檔案**:
- `AddItemScreen.kt` - 新增物品表單
- `MapScreen.kt` - 地圖顯示
- `BuildingItemsScreen.kt` - 建築物物品列表
- `HomeScreen.kt` - 首頁
- `SearchScreen.kt` - 搜尋頁面
- `DetailScreen.kt` - 物品詳情

**職責**:
- 顯示 UI 組件
- 捕獲用戶輸入
- 調用 ViewModel 方法
- 訂閱 StateFlow 顯示數據

**範例**:
```kotlin
@Composable
fun MapScreen(navController: NavController) {
    val viewModel: MainViewModel = remember { ... }
    val buildings by viewModel.buildings.collectAsState()
    val itemCountByBuilding by viewModel.itemCountByBuilding.collectAsState()

    // UI 邏輯...
}
```

#### 2️⃣ ViewModel 層 (State Management)
**位置**: `ui/viewmodel/`

**檔案**:
- `MainViewModel.kt` - 主應用狀態容器
- `ViewModelFactory.kt` - ViewModel 工廠

**職責**:
- 管理 UI 狀態（通過 StateFlow）
- 調用 Repository 方法
- 協程生命週期管理
- 錯誤處理與日誌記錄

**重要概念**:
```kotlin
class MainViewModel(private val repository: LostFoundRepository) : ViewModel() {
    // 狀態流
    private val _buildings = MutableStateFlow<List<Building>>(emptyList())
    val buildings: StateFlow<List<Building>> = _buildings.asStateFlow()

    // 業務邏輯
    fun loadBuildings() {
        viewModelScope.launch {
            val result = repository.getAllBuildings()
            result.onSuccess { buildings ->
                _buildings.value = buildings
            }
        }
    }
}
```

**關鍵特性**:
- ✅ 單一真實來源 (Single Source of Truth)
- ✅ 不可變狀態流
- ✅ 自動生命週期感知
- ✅ 協程作用域管理

#### 3️⃣ Repository 層 (Data Access)
**位置**: `repository/`

**檔案**:
- `LostFoundRepository.kt` - 統一數據訪問接口

**職責**:
- 封裝 Firebase 操作
- 提供數據查詢/修改接口
- 統一錯誤處理（使用 Result<T>）
- 實現業務邏輯

**架構優勢**:
```kotlin
class LostFoundRepository(
    private val firestore: FirebaseFirestore,
    private val storage: FirebaseStorage
) {
    // 統一的返回類型
    suspend fun getItemCountByBuilding(): Result<Map<String, Int>>

    // 只返回有物品的建築物計數
    // 這確保地圖只顯示有物品的圖釘！
}
```

#### 4️⃣ Service 層 (業務服務)
**位置**: `services/`

**檔案**:
- `FirebaseService.kt` - Firebase 配置與初始化
- `QuotaManager.kt` - Google Maps 配額管理

**職責**:
- Firebase 連接管理
- 配額追蹤與限制
- 跨應用服務

#### 5️⃣ Model 層 (數據模型)
**位置**: `models/`

**檔案**:
- `FirebaseModels.kt` - Building, LostFoundItem, User

**關鍵模型**:

```kotlin
// 建築物模型
data class Building(
    @DocumentId
    val id: String = "",
    val name: String = "",
    val latitude: Double = 0.0,
    val longitude: Double = 0.0,
    val color: String = "#FF5733",
    val description: String = "",
    val zone: String = ""
)

// 物品模型（統一使用此模型）
data class LostFoundItem(
    @DocumentId
    val id: String = "",
    val name: String = "",
    val description: String = "",
    val buildingId: String = "",  // 關鍵：與建築物的連結
    val location: String = "",
    val imageUrl: String = "",
    val tags: List<String> = emptyList(),
    val timestamp: Long = 0L,
    val status: String = "available",  // available/claimed/returned
    val contact: String = "",
    val uploadedBy: String = "",
    val category: String = "found"  // found/lost
)
```

---

## 核心模組

### 🏢 建築物管理 Module

**流程**:
```
1. 應用啟動
   ↓
2. FirebaseService.initializeSampleData()
   - 刪除所有舊建築物
   - 寫入所有 41 棟建築物
   ↓
3. MainViewModel.loadBuildings()
   ↓
4. Repository.getAllBuildings()
   ↓
5. Firestore 查詢
   ↓
6. StateFlow 更新 → UI 刷新
```

**檔案**:
- `repository/LostFoundRepository.kt:43-88` - 建築物查詢
- `ui/viewmodel/MainViewModel.kt:113-130` - 狀態管理

### 📦 物品管理 Module

**流程**:
```
1. 用戶在 AddItemScreen 填寫表單
   - 選擇建築物
   - 拍照
   - 輸入物品名稱和描述
   ↓
2. 點擊「新增」按鈕
   ↓
3. 上傳圖片到 Storage
   ↓
4. 保存物品信息到 Firestore
   ↓
5. MainViewModel.addItem() 更新狀態
   ↓
6. 所有頁面 (Home, Map, Search) 自動更新
```

**關鍵文件**:
- `repository/LostFoundRepository.kt:135-165` - 物品新增
- `ui/screens/AddItemScreen.kt` - 表單 UI
- `ui/viewmodel/MainViewModel.kt:179-199` - 狀態更新

### 🗺️ 地圖顯示 Module

**核心邏輯**:
```kotlin
// 只顯示有物品的圖釘的流程

1. Repository.getItemCountByBuilding()
   ↓
   for each building:
       - 查詢該建築物的物品
       - if (count > 0):
           - itemCountMap.put(buildingId, count)
   ↓
   return Map<buildingId, itemCount>  // 只包含有物品的建築物
   ↓

2. MapScreen 訂閱 itemCountByBuilding StateFlow
   ↓

3. GoogleMap 展示
   itemCountByBuilding.forEach { (buildingId, count) ->
       // 只創建有物品的標記
       Marker(...)
   }
```

**重要改變**:
- ✅ 之前: 遍歷所有建築物 → 導致大量空圖釘
- ✅ 現在: Repository 只返回有物品的計數 → 自動只顯示有物品的圖釘

**檔案**:
- `repository/LostFoundRepository.kt:176-224` - 物品計數查詢
- `ui/screens/MapScreen.kt:200-222` - 圖釘渲染

### 🔍 搜尋 Module

**流程**:
```
1. 用戶輸入搜尋詞
   ↓
2. MainViewModel.searchItems(query)
   ↓
3. Repository.searchItems(query)
   ↓
4. 在內存中搜尋：
   - 物品名稱
   - 物品描述
   - 物品標籤
   ↓
5. StateFlow 更新 → 搜尋結果顯示
```

**檔案**:
- `repository/LostFoundRepository.kt:238-259` - 搜尋實現
- `ui/viewmodel/MainViewModel.kt:224-250` - 搜尋狀態管理

---

## 數據流設計

### 📍 新增物品完整流程

```
┌─────────────────────────────────┐
│  AddItemScreen                  │
│  - 選擇建築物                    │
│  - 拍照                          │
│  - 輸入信息                      │
└──────────┬──────────────────────┘
           │ 點擊「新增」
           ▼
┌─────────────────────────────────┐
│  MainViewModel.addItem()         │
│  - 調用 Repository.addItem()    │
│  - 更新 _items StateFlow         │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  LostFoundRepository.addItem()  │
│  - 生成 itemId                  │
│  - 添加時間戳                    │
│  - 保存到 Firestore             │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│  Firebase Firestore             │
│  collection("lostItems")        │
│  document(itemId)               │
│  {                              │
│    id, name, buildingId,        │
│    timestamp, ...               │
│  }                              │
└─────────────────────────────────┘
```

### 🗺️ 地圖加載完整流程

```
┌─────────────────────────────────┐
│  MapScreen LaunchedEffect       │
└──────────┬──────────────────────┘
           │
           ├─→ viewModel.loadBuildings()
           │   ↓
           │   Repository.getAllBuildings()
           │   ↓
           │   _buildings.value = buildings
           │
           └─→ viewModel.loadItemCountByBuilding()
               ↓
               Repository.getItemCountByBuilding()
               ↓
               for each building:
                   count = getItemsByBuilding(buildingId)
                   if count > 0:
                       map.put(buildingId, count)
               ↓
               _itemCountByBuilding.value = map
               ↓
               GoogleMap 重新渲染
               ↓
               只顯示有物品的圖釘 ✓
```

---

## 類別關係圖

### 依賴關係

```
UI Screen
    ↓ (uses)
MainViewModel
    ↓ (calls)
LostFoundRepository
    ↓ (calls)
Firebase (Firestore + Storage)
    ↓
Models (Building, LostFoundItem)
```

### 數據流向

```
Firebase
    ↓
Repository (Result<T>)
    ↓
ViewModel (StateFlow)
    ↓
UI Screen (Composable)
```

---

## API 設計

### 📚 LostFoundRepository API

#### 建築物操作
```kotlin
// 獲取所有建築物
suspend fun getAllBuildings(): Result<List<Building>>

// 獲取單個建築物
suspend fun getBuildingById(buildingId: String): Result<Building?>

// 初始化建築物
suspend fun initializeBuildings(buildings: List<Building>): Result<Unit>
```

#### 物品操作
```kotlin
// 獲取所有物品
suspend fun getAllItems(): Result<List<LostFoundItem>>

// 獲取特定建築物的物品
suspend fun getItemsByBuilding(buildingId: String): Result<List<LostFoundItem>>

// ⭐ 獲取物品計數（只返回有物品的建築物）
suspend fun getItemCountByBuilding(): Result<Map<String, Int>>

// 獲取單個物品
suspend fun getItemById(itemId: String): Result<LostFoundItem?>

// 新增物品
suspend fun addItem(item: LostFoundItem): Result<String>

// 更新物品狀態
suspend fun updateItemStatus(itemId: String, status: String): Result<Unit>

// 搜尋物品
suspend fun searchItems(query: String): Result<List<LostFoundItem>>
```

#### 圖片操作
```kotlin
// 上傳圖片
suspend fun uploadImage(
    imageUri: Uri,
    itemId: String = UUID.randomUUID().toString()
): Result<String>

// 刪除圖片
suspend fun deleteImage(imageUrl: String): Result<Unit>
```

### 🎮 MainViewModel API

```kotlin
// 初始化應用
fun initializeApp()

// 建築物操作
fun loadBuildings()
fun initializeBuildings(buildings: List<Building>)

// 物品操作
fun loadItems()
fun loadItemsByBuilding(buildingId: String)
fun loadItemCountByBuilding()  // ⭐ 用於地圖
fun addItem(item: LostFoundItem)
fun updateItemStatus(itemId: String, status: String)

// 搜尋操作
fun searchItems(query: String)
fun clearSearch()

// 調試
fun debugPrintState()
```

---

## 🔧 系統設計原則

### 1. 單一責任原則 (SRP)
```
Repository   → 只負責數據訪問
ViewModel    → 只負責狀態管理
UI Screen    → 只負責顯示
```

### 2. 依賴反轉原則 (DIP)
```kotlin
// ✅ 好的設計
class MainViewModel(
    private val repository: LostFoundRepository  // 依賴抽象
)

// ❌ 不好的設計
class MainViewModel {
    private val firestore = FirebaseFirestore.getInstance()  // 直接依賴
}
```

### 3. DRY (Don't Repeat Yourself)
- ✅ 統一使用 `LostFoundItem` 模型
- ✅ Repository 統一返回 `Result<T>`
- ✅ ViewModel 統一管理狀態

### 4. 明確的數據流
```
數據單向流動：
Firebase → Repository → ViewModel → UI

用戶交互反向流動：
UI → ViewModel → Repository → Firebase
```

---

## 📊 關鍵改進

### 問題 1: 圖釘不顯示
**根本原因**: `itemCountByBuilding` 為空
**解決方案**:
- Repository 實現 `getItemCountByBuilding()`
- 只返回有物品的建築物計數
- MapScreen 只遍歷有物品的圖釘

### 問題 2: 數據模型混亂
**根本原因**: 使用多個不同的數據模型
**解決方案**:
- 統一使用 `LostFoundItem`
- 所有屏幕共用一個模型
- 降低維護成本

### 問題 3: 數據訪問分散
**根本原因**: Firebase 調用遍佈在各個屏幕
**解決方案**:
- 建立 Repository 層
- 集中管理所有數據訪問
- 便於測試和維護

---

## 🚀 部署架構

### Firebase 數據庫結構

```
Firestore
├── buildings/
│   ├── LI (文華樓)
│   ├── LF (文友樓)
│   └── ... (41 棟)
│
├── lostItems/
│   ├── {itemId1}
│   │   ├── id: "..."
│   │   ├── name: "黑色錢包"
│   │   ├── buildingId: "ES"  ← 建築物連結
│   │   ├── timestamp: 1234567890
│   │   ├── status: "available"
│   │   └── ...
│   └── {itemId2}
│       └── ...
│
└── users/  (未來用)
    └── ...
```

### 配額管理

```
DataStore (本地)
├── api_call_count: Long (當日 API 呼叫次數)
├── last_reset_date: Long (上次重置日期)
└── is_quota_exceeded: Boolean (配額是否超出)

限制: 25,000 API 呼叫/天
警告: 20,000 呼叫 (80%)
```

---

## 📝 總結

此架構設計提供：

1. **清晰的分層** - 5 層架構，職責分明
2. **統一的數據流** - 單向數據流，易於追蹤
3. **強大的可維護性** - Repository 模式，易於測試
4. **完整的狀態管理** - ViewModel + StateFlow，自動同步
5. **健壯的錯誤處理** - Result<T> 類型，統一異常管理

**關鍵成功因素**:
- ✅ Repository 層統一數據訪問
- ✅ ViewModel 層統一狀態管理
- ✅ `getItemCountByBuilding()` 只返回有物品的建築物
- ✅ MapScreen 智能渲染只有物品的圖釘

---

**文檔更新**: 2024-11-13
**架構版本**: v2.0 (MVVM + Repository)
