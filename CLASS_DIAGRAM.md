# 類別關係圖

## 1. 完整系統類別圖

```
┌────────────────────────────────────────────────────────────────┐
│                        UI Layer (Screens)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   Home   │  │   Map    │  │  AddItem │  │  Search  │ ...  │
│  │ Screen   │  │ Screen   │  │ Screen   │  │ Screen   │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │             │             │             │              │
│       └─────────────┴─────────────┴─────────────┘              │
│              ▲                                                  │
│              │ 訂閱 StateFlow                                   │
│              │ 調用方法                                        │
└──────────────┼──────────────────────────────────────────────────┘
               │
┌──────────────┼──────────────────────────────────────────────────┐
│              │        ViewModel Layer                           │
│              │                                                  │
│         ┌────▼─────────────────────────────┐                   │
│         │     MainViewModel                │                   │
│         ├───────────────────────────────────┤                   │
│         │ - buildings: StateFlow            │                   │
│         │ - items: StateFlow                │                   │
│         │ - itemCountByBuilding: StateFlow  │ ⭐              │
│         │ - searchResults: StateFlow        │                   │
│         │ - errors: StateFlow               │                   │
│         ├───────────────────────────────────┤                   │
│         │ + loadBuildings()                 │                   │
│         │ + loadItems()                     │                   │
│         │ + loadItemCountByBuilding()       │ ⭐              │
│         │ + addItem()                       │                   │
│         │ + searchItems()                   │                   │
│         │ + updateItemStatus()              │                   │
│         └────┬──────────────────────────────┘                   │
│              │                                                  │
│              │ 使用                                            │
└──────────────┼──────────────────────────────────────────────────┘
               │
┌──────────────┼──────────────────────────────────────────────────┐
│              │      Repository Layer                            │
│              │                                                  │
│         ┌────▼────────────────────────────────────┐             │
│         │  LostFoundRepository                    │             │
│         ├────────────────────────────────────────┤             │
│         │ - firestore: FirebaseFirestore         │             │
│         │ - storage: FirebaseStorage             │             │
│         ├────────────────────────────────────────┤             │
│         │ Building Operations:                   │             │
│         │ + getAllBuildings()                    │             │
│         │ + getBuildingById()                    │             │
│         │ + initializeBuildings()                │             │
│         │                                        │             │
│         │ Item Operations:                       │             │
│         │ + getAllItems()                        │             │
│         │ + getItemsByBuilding()                 │             │
│         │ + getItemCountByBuilding() ⭐         │             │
│         │ + addItem()                            │             │
│         │ + updateItemStatus()                   │             │
│         │ + searchItems()                        │             │
│         │                                        │             │
│         │ Image Operations:                      │             │
│         │ + uploadImage()                        │             │
│         │ + deleteImage()                        │             │
│         └────┬─────────────────────────────────┘              │
│              │                                                  │
│              │ 調用                                            │
└──────────────┼──────────────────────────────────────────────────┘
               │
┌──────────────┼──────────────────────────────────────────────────┐
│              │         Firebase Services                        │
│              │                                                  │
│    ┌─────────▼─────┐         ┌──────────────┐                 │
│    │   Firestore   │         │ Firebase     │                 │
│    │  (Database)   │         │ Storage      │                 │
│    └───────────────┘         │ (Images)     │                 │
│                              └──────────────┘                  │
└──────────────────────────────────────────────────────────────────┘
```

## 2. 數據模型圖

```
┌─────────────────────────────────┐
│      Building                   │
├─────────────────────────────────┤
│ - id: String                    │
│ - name: String                  │
│ - latitude: Double              │
│ - longitude: Double             │
│ - color: String                 │
│ - description: String           │
│ - zone: String                  │
└─────────────────────────────────┘
         △
         │ 1
         │
    ┌────┴──────────────────┐
    │  1:Many Relationship  │
    │                       │
    │                       │
    │                    ┌──▼──────────────────────────────┐
    │                    │  LostFoundItem                  │
    │                    ├─────────────────────────────────┤
    │                    │ - id: String                    │
    │                    │ - name: String                  │
    │                    │ - description: String           │
    │                    │ - buildingId: String ⭐ (FK)   │
    │                    │ - location: String              │
    │                    │ - imageUrl: String              │
    │                    │ - thumbnailUrl: String          │
    │                    │ - tags: List<String>            │
    │                    │ - timestamp: Long               │
    │                    │ - status: String                │
    │                    │ - contact: String               │
    │                    │ - phone: String                 │
    │                    │ - uploadedBy: String            │
    │                    │ - category: String              │
    │                    └─────────────────────────────────┘
```

## 3. 狀態流依賴圖

```
┌─────────────────────────────────────────────────────────────┐
│              MainViewModel State Flows                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Buildings:                                                 │
│  ┌─────────────────────────────────────┐                  │
│  │ _buildings: MutableStateFlow         │                  │
│  │ buildings: StateFlow (public)        │                  │
│  │ _buildingsLoading: StateFlow         │                  │
│  │ _buildingsError: StateFlow           │                  │
│  └─────────────────────────────────────┘                  │
│                                                              │
│  Items:                                                     │
│  ┌─────────────────────────────────────┐                  │
│  │ _items: MutableStateFlow             │                  │
│  │ items: StateFlow (public)            │                  │
│  │ _itemsLoading: StateFlow             │                  │
│  │ _itemsError: StateFlow               │                  │
│  └─────────────────────────────────────┘                  │
│                                                              │
│  Map Item Count (⭐ Key):                                   │
│  ┌─────────────────────────────────────┐                  │
│  │ _itemCountByBuilding: MutableState   │                  │
│  │ itemCountByBuilding: StateFlow       │                  │
│  │ _itemCountLoading: StateFlow         │                  │
│  │ _itemCountError: StateFlow           │                  │
│  └─────────────────────────────────────┘                  │
│                                                              │
│  Search:                                                    │
│  ┌─────────────────────────────────────┐                  │
│  │ _searchResults: MutableStateFlow     │                  │
│  │ searchResults: StateFlow (public)    │                  │
│  │ _searchQuery: StateFlow              │                  │
│  │ _searchLoading: StateFlow            │                  │
│  └─────────────────────────────────────┘                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 4. 數據流交互序列圖

### 新增物品流程

```
User          UI              ViewModel        Repository       Firebase
│             │                   │                │              │
├─ Click      │                   │                │              │
│  "Add"      │                   │                │              │
│             │─ addItem(item)──→ │                │              │
│             │                   │─ addItem()──→  │              │
│             │                   │                │─ upload────→ │
│             │                   │                │  to Storage  │
│             │                   │                │  & Firestore │
│             │                   │                │              │
│             │                   │                │ ◄─ success ─│
│             │                   │ ◄─ itemId ────│              │
│             │                   │                              │
│             │                   │─ loadItems()─→ │              │
│             │                   │                │─ query ───→ │
│             │                   │                │              │
│             │                   │                │ ◄─ items ──│
│             │                   │ ◄─ result ────│              │
│             │                   │                              │
│             │ ◄ items updated ─│                              │
│             │                   │                              │
│ ◄─ UI        │                   │                              │
│  refreshed ─│                   │                              │
```

### 地圖加載流程

```
MapScreen      ViewModel         Repository        Firebase
│              │                     │                 │
├─ LaunchEffect
│  triggered   │                     │                 │
│              │─ loadBuildings()──→  │                │
│              │                     │─ query ──────→ │
│              │                     │                │
│              │                     │ ◄─ buildings──│
│              │ ◄─ buildings result ─│               │
│              │                     │               │
│              │─ loadItemCountBy
│              │  Building()──────→  │               │
│              │                     │               │
│              │                ┌─→ For each building:
│              │                │    │─ getItems()─→│
│              │                │    │          query items
│              │                │    │        ◄─────│
│              │                │    if count > 0:
│              │                │      add to map
│              │                │
│              │ ◄─ itemCount ──│
│              │    Map          │               │
│              │                 │               │
│ ◄─ GoogleMap
│  Composable │
│  recomposes │
```

## 5. 方法調用依賴圖

```
┌─────────────────────────────────────────────────────┐
│  loadItemCountByBuilding()  ⭐ 關鍵方法              │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. repository.getItemCountByBuilding()             │
│     ├─ repository.getAllBuildings()                 │
│     │  └─ Firebase Query: "SELECT * FROM buildings"│
│     │                                               │
│     └─ For each building:                           │
│        ├─ repository.getItemsByBuilding(id)         │
│        │  └─ Firebase Query:                        │
│        │     "SELECT * FROM lostItems               │
│        │      WHERE buildingId = id"                │
│        │                                             │
│        └─ if items.isNotEmpty():                    │
│           └─ itemCount[buildingId] = items.size     │
│                                                     │
│  2. _itemCountByBuilding.value = itemCounts         │
│     └─ Notify all subscribers                       │
│        └─ MapScreen receives update                 │
│           └─ GoogleMap recomposes                   │
│              └─ Only renders markers for            │
│                 buildingIds in itemCounts           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## 6. Repository 方法分類

```
┌──────────────────────────────────────────────────────┐
│         LostFoundRepository API                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│ Building Operations:                                │
│ ├─ suspend fun getAllBuildings()                    │
│ │  Result<List<Building>>                          │
│ ├─ suspend fun getBuildingById(id)                  │
│ │  Result<Building?>                               │
│ └─ suspend fun initializeBuildings(list)            │
│    Result<Unit>                                     │
│                                                      │
│ Item Operations:                                    │
│ ├─ suspend fun getAllItems()                        │
│ │  Result<List<LostFoundItem>>                     │
│ ├─ suspend fun getItemsByBuilding(id)               │
│ │  Result<List<LostFoundItem>>                     │
│ ├─ suspend fun getItemCountByBuilding() ⭐         │
│ │  Result<Map<String, Int>>                        │
│ ├─ suspend fun getItemById(id)                      │
│ │  Result<LostFoundItem?>                          │
│ ├─ suspend fun addItem(item)                        │
│ │  Result<String>  (itemId)                        │
│ ├─ suspend fun updateItemStatus(id, status)         │
│ │  Result<Unit>                                     │
│ └─ suspend fun searchItems(query)                   │
│    Result<List<LostFoundItem>>                     │
│                                                      │
│ Image Operations:                                   │
│ ├─ suspend fun uploadImage(uri)                     │
│ │  Result<String>  (imageUrl)                      │
│ └─ suspend fun deleteImage(url)                     │
│    Result<Unit>                                     │
│                                                      │
└──────────────────────────────────────────────────────┘
```

## 7. ViewModel 方法分類

```
┌────────────────────────────────────────────────────┐
│       MainViewModel API                            │
├────────────────────────────────────────────────────┤
│                                                    │
│ Initialization:                                   │
│ └─ fun initializeApp()                            │
│                                                    │
│ Building Operations:                              │
│ ├─ fun loadBuildings()                            │
│ └─ fun initializeBuildings(list)                  │
│                                                    │
│ Item Operations:                                  │
│ ├─ fun loadItems()                                │
│ ├─ fun loadItemsByBuilding(id)                    │
│ ├─ fun loadItemCountByBuilding() ⭐              │
│ ├─ fun addItem(item)                              │
│ └─ fun updateItemStatus(id, status)               │
│                                                    │
│ Search Operations:                                │
│ ├─ fun searchItems(query)                         │
│ └─ fun clearSearch()                              │
│                                                    │
│ Debug:                                            │
│ └─ fun debugPrintState()                          │
│                                                    │
└────────────────────────────────────────────────────┘
```

## 8. Firestore 數據庫結構

```
Firestore (Data)
│
├── /buildings/ (Collection)
│   ├── LI/ (Document)
│   │   ├── id: "LI"
│   │   ├── name: "文華樓"
│   │   ├── latitude: 25.0352
│   │   ├── longitude: 121.4322
│   │   ├── color: "#FF5733"
│   │   ├── description: ""
│   │   └── zone: "LI"
│   ├── LF/ (Document)
│   │   └── ... (同上)
│   └── ... (41 棟)
│
├── /lostItems/ (Collection)
│   ├── {itemId1}/ (Document)
│   │   ├── id: "{itemId1}"
│   │   ├── name: "黑色錢包"
│   │   ├── description: "皮革，有卡片"
│   │   ├── buildingId: "ES"  ⭐
│   │   ├── location: "進修部大樓 1F"
│   │   ├── imageUrl: "gs://..."
│   │   ├── tags: ["錢包", "皮革", "黑色"]
│   │   ├── timestamp: 1234567890000
│   │   ├── status: "available"
│   │   ├── contact: "user@email.com"
│   │   ├── phone: "0912345678"
│   │   ├── uploadedBy: "anonymous"
│   │   └── category: "found"
│   ├── {itemId2}/ (Document)
│   │   └── ... (同上)
│   └── ...
│
└── /users/ (Collection)
    └── ... (Future)
```

## 9. 狀態更新流程圖

```
Repository        ViewModel              UI
   │                  │                  │
   │                  │                  │
   │ getAllItems()    │                  │
   │◄─────────────────┤                  │
   │                  │                  │
   │─ Result.success()─→ _items.value    │
   │                  │     = items      │
   │                  │                  │
   │                  │   items         │
   │                  │ StateFlow       │
   │                  │    │            │
   │                  │    │            │
   │                  │    └───────────→│ collectAsState()
   │                  │                  │
   │                  │                  │ LazyColumn {
   │                  │                  │   items(items) {
   │                  │                  │     ItemCard()
   │                  │                  │   }
   │                  │                  │ }
   │                  │                  │
   │                  │                  │ Recompose ✓
```

---

## 10. 核心架構總結

```
                    Data Flow

Firebase Firestore
        │
        │ Query Results
        ▼
Repository.getAllItems()
    Result<List<LostFoundItem>>
        │
        │ onSuccess
        ▼
ViewModel._items.value = items
        │
        │ StateFlow Emission
        ▼
UI: collectAsState()
        │
        │ State Change
        ▼
Compose Recomposition
        │
        │
        ▼
UI Rendered ✓
```

---

**類別關係圖 v1.0**
**最後更新**: 2024-11-13
