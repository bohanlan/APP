# FJU Lost & Found Application - Architecture Analysis

**Total LOC:** 2,540 lines of Kotlin code across 25 files
**Architecture Pattern:** MVVM (Model-View-ViewModel) with Jetpack Compose
**Backend:** Firebase (Firestore + Storage + Authentication)

---

## 1. PROJECT STRUCTURE

```
tw.edu.fju.myapplication/
├── MainActivity.kt                    [90 lines]    - App entry, Navigation routing
├── MainViewModel.kt                   [31 lines]    - Global state (sort option)
├── models/
│   ├── LostItem.kt                    [36 lines]    - Local item model (legacy)
│   └── FirebaseModels.kt              [55 lines]    - Firebase data models (Building, LostFoundItem, User)
├── services/
│   ├── FirebaseService.kt             [266 lines]   - Firebase operations (CRUD, search, auth)
│   └── QuotaManager.kt                [152 lines]   - Google Maps API quota management
├── ui/
│   ├── screens/
│   │   ├── HomeScreen.kt              [140 lines]   - Main landing page
│   │   ├── SearchScreen.kt            [101 lines]   - Search results display
│   │   ├── DetailScreen.kt            [99 lines]    - Item detail view
│   │   ├── AddItemScreen.kt           [431 lines]   - Item reporting form
│   │   ├── MapScreen.kt               [282 lines]   - Google Maps with quota protection
│   │   └── BuildingItemsScreen.kt     [130 lines]   - Building-specific items
│   ├── components/
│   │   ├── LostItemCard.kt            [68 lines]    - Item card UI component
│   │   ├── SortButtons.kt             [68 lines]    - Sort UI control
│   │   └── BuildingMarkerInfo.kt      [17 lines]    - Map marker info
│   ├── theme/
│   │   ├── Color.kt                   [8 lines]
│   │   ├── Theme.kt                   [22 lines]
│   │   └── Type.kt                    [33 lines]
└── utils/
    ├── SearchUtils.kt                 [43 lines]    - Search logic
    ├── SearchHelper.kt                [50 lines]    - Search helpers
    ├── StringUtils.kt                 [58 lines]    - String processing
    ├── TimeUtils.kt                   [62 lines]    - Time parsing/sorting
    ├── LocationUtils.kt               [126 lines]   - GPS/location handling
    ├── CameraHelper.kt                [114 lines]   - Camera integration
    └── CameraUtils.kt                 [58 lines]    - Camera utilities
```

---

## 2. COMPONENT BREAKDOWN

### **2.1 CORE MODELS**

#### LostItem (Legacy Local Model)
```kotlin
data class LostItem(
    val name: String,              // "黑色錢包"
    val location: String,          // "在文華樓 3F 走廊"
    val time: String,              // "2 小時前"
    val buildingId: String = ""    // "building_001"
)
```
**Purpose:** Used for displaying items on HomeScreen from `sampleItems` list
**Status:** Legacy - being replaced by Firebase models

#### Firebase Models
```kotlin
data class Building(
    val id: String,              // "LI", "LF", "LE" (41 unique IDs)
    val name: String,            // "文華樓"
    val latitude: Double,
    val longitude: Double,
    val color: String,           // "#FF5733"
    val description: String,
    val zone: String
)

data class LostFoundItem(
    val id: String,
    val name: String,
    val description: String,
    val buildingId: String,      // KEY: Links to Building
    val location: String,
    val imageUrl: String,        // Firebase Storage
    val thumbnailUrl: String,
    val tags: List<String>,      // Gemini-generated tags
    val timestamp: Long,
    val status: String,          // "available", "claimed", "returned"
    val contact: String,
    val phone: String,
    val uploadedBy: String,      // User ID
    val category: String         // "found" or "lost"
)

data class User(
    val id: String,
    val email: String,
    val name: String,
    val uploadedItems: List<String>,
    val createdAt: Long
)
```
**Status:** Production Firebase Firestore documents

---

### **2.2 SERVICES LAYER**

#### FirebaseService (266 lines)
**Responsibility:** All Firebase Firestore & Storage operations

**Key Methods:**
- `initializeAuth()` - Anonymous Firebase auth
- `getCurrentUserId()` - Get current user
- `uploadImage(Uri, itemId)` - Upload to Storage, return URL
- `getBuildings()` - Fetch all 41 buildings
- `getItemsByBuilding(buildingId)` - Fetch items by building
- `getAllItems()` - Fetch all items
- `addItem(LostFoundItem)` - Create new item
- `updateItemStatus(itemId, status)` - Mark as claimed/returned
- `searchItems(query)` - Full-text search
- `initializeSampleData()` - Seed 41 buildings

**Data Flow:**
```
Firestore Collections:
├── buildings/
│   ├── LI (Building doc)
│   ├── LF (Building doc)
│   └── ... (41 buildings)
└── lostItems/
    ├── {uuid1} (LostFoundItem doc)
    ├── {uuid2} (LostFoundItem doc)
    └── ... (user-submitted items)

Firebase Storage:
└── items/
    ├── {itemId1}/image_*.jpg
    ├── {itemId2}/image_*.jpg
    └── ...
```

#### QuotaManager (152 lines)
**Responsibility:** Google Maps API quota tracking (25,000 calls/day)

**Features:**
- Daily quota reset at midnight
- Call counting via DataStore
- Warning at 80% (20,000 calls)
- Auto-disable map at limit
- Quota percentage UI indicator

**Data Storage:** Android DataStore (local encrypted)

---

### **2.3 VIEW MODELS**

#### MainViewModel (31 lines)
**Responsibility:** Global app state
```kotlin
class MainViewModel(private val state: SavedStateHandle) : ViewModel() {
    val sortOption: StateFlow<String>  // "最新" or "最舊"
    
    init {
        FirebaseService.initializeAuth()
        FirebaseService.initializeSampleData()
    }
    
    fun setSort(option: String)
}
```
**Issues:** 
- Minimal ViewModel - sort state is the only global state
- Firebase initialization in ViewModel (not ideal)
- No error handling or loading states

---

### **2.4 NAVIGATION & SCREENS**

#### Navigation Routes (MainActivity.kt)
```kotlin
"home"                                    → HomePage
"map"                                     → MapScreen
"addItem"                                 → AddItemScreen
"search/{query}"                          → SearchResultScreen
"detail/{index}"                          → DetailScreen
"buildingItems/{buildingId}/{buildingName}" → BuildingItemsScreen
```

#### Screen Hierarchy

**HomePage (140 lines)**
- Displays: sampleItems (local hardcoded data)
- Features: Search box, Sort buttons, Map button, FAB (Add Item)
- State: searchQuery, sortOption (from ViewModel)
- Issue: Uses old `LostItem` model, not Firebase data

**SearchResultScreen (101 lines)**
- Displays: Search results from sampleItems
- Features: Sort buttons, result cards
- Issue: Searches only local sampleItems, not Firebase items

**AddItemScreen (431 lines) - LARGEST**
- 3-Step workflow:
  1. Select building from 41 buildings (Firebase)
  2. Take photo (Camera + CameraHelper)
  3. Enter item details
- Upload flow: Photo → Firebase Storage → Create LostFoundItem → Save to Firestore
- Features: ModalBottomSheet building selector, real-time feedback
- Issues:
  - Very large component (431 LOC)
  - Complex state management (11 mutable states)
  - All logic in single Composable

**MapScreen (282 lines)**
- Displays: Google Maps with building markers
- Shows: Item count per building
- Features: 
  - Quota indicator (red/yellow/green)
  - Automatic map disable if quota exceeded
  - Click markers to see building items
- Calls: getBuildings() + getItemsByBuilding() for each building

**BuildingItemsScreen (130 lines)**
- Displays: Items from both Firebase + local sampleItems
- Data merge: Converts LostFoundItem → LostItem for display
- Flow: Building ID → getItemsByBuilding() → Merge with local items

**DetailScreen (99 lines)**
- Shows: Item details (name, location, time)
- Issue: Only accesses sampleItems[index], not Firebase items
- Missing: Image display, contact info, status

---

### **2.5 UI COMPONENTS**

#### LostItemCard (68 lines)
```kotlin
@Composable
fun LostItemCard(item: LostItem, onClick: () -> Unit)
```
- Displays: Name, Location, Time
- Missing: Image, Status, Tags
- Reusable: Used in HomeScreen, SearchScreen, BuildingItemsScreen

#### SortButtons (68 lines)
- Options: "最新" (newest) / "最舊" (oldest)
- Used in: HomeScreen, SearchScreen

---

### **2.6 UTILITIES**

#### SearchUtils (43 lines)
```kotlin
fun searchItems(items, query) → filters by name/location
fun sortItems(items, sortOption) → by timestamp
fun searchAndSort(items, query, sortOption) → combined
```

#### TimeUtils (62 lines)
- Parses Chinese time strings: "X 分鐘前", "今天", "昨天"
- Converts to milliseconds for sorting
- Example: "2 小時前" → now - 2*3600*1000

#### StringUtils (58 lines)
- Location display formatting
- Query expansion (synonyms)

#### LocationUtils (126 lines)
- `getCurrentLocation(context)` - Gets device GPS
- Default: FJU (Continuing Education Building)
- Latitude: 25.035, Longitude: 121.432

#### CameraHelper (114 lines)
- `takePicture(context, lifecycleOwner)` - Intent-based camera
- Returns: Uri of saved photo
- Used in: AddItemScreen

---

## 3. DATA FLOW ARCHITECTURE

### **3.1 CURRENT DATA FLOW**

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER INTERACTION                         │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ HomeScreen (UI)      │
                    │ SearchScreen (UI)    │
                    │ DetailScreen (UI)    │
                    │ MapScreen (UI)       │
                    └──────────────────────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
                ▼              ▼              ▼
        ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
        │ sampleItems  │ │ MainViewModel│ │ Firebase     │
        │ (hardcoded)  │ │ (sort state) │ │ Service      │
        └──────────────┘ └──────────────┘ └──────────────┘
                                               │
                ┌──────────────┬──────────────┬┴──────────────┐
                │              │              │               │
                ▼              ▼              ▼               ▼
        ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
        │ Firestore    │ │ Firebase Auth│ │ Firebase Stor│ │ Quota Manager│
        │ lostItems/   │ │ (anonymous)  │ │ items/*.jpg  │ │ (DataStore)  │
        │ buildings/   │ │              │ │              │ │              │
        └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
```

### **3.2 USER INPUT → FIREBASE → UI CYCLE**

#### Path 1: Add Item (Create)
```
User Input (AddItemScreen)
  ├─ Select Building (41 options from Firebase)
  ├─ Take Photo (Camera API)
  ├─ Enter details (name, description)
  └─ Submit
           │
           ▼
FirebaseService.uploadImage(Uri)
           │
           ├─→ Firebase Storage: items/{itemId}/image_*.jpg
           │
           └─→ Returns: download URL
           │
           ▼
FirebaseService.addItem(LostFoundItem)
           │
           └─→ Firestore: lostItems/{uuid} = {LostFoundItem doc}
                          with imageUrl from Storage
```

#### Path 2: Search & Display (Read)
```
User Input (HomeScreen search)
  └─ Query: "錢包"
           │
           ▼
SearchUtils.searchAndSort(sampleItems, query)
           │
           ├─ Searches: name, location from sampleItems (HARDCODED)
           │ Issue: Does NOT search Firebase items
           │
           └─→ SearchResultScreen displays results

User Action (MapScreen click building)
  └─ Click marker on building "文華樓"
           │
           ▼
FirebaseService.getItemsByBuilding("LI")
           │
           └─→ Firestore query: lostItems where buildingId == "LI"
                (Returns LostFoundItem docs)
                │
                ▼
           BuildingItemsScreen
           Merge: Firebase items + local sampleItems by buildingId
           Display as LostItemCard
```

#### Path 3: Real-time Map Display (Read)
```
MapScreen Init
  └─ LaunchedEffect
           │
           ├─ FirebaseService.getBuildings() → List<Building>
           │
           └─ For each building:
              └─ FirebaseService.getItemsByBuilding(id) → count
                        │
                        ▼
                   Show marker only if count > 0
                   Marker label: "物品數: {count}"
                        │
                        ▼
                   On marker click:
                   - QuotaManager.recordApiCall()
                   - Navigate to BuildingItemsScreen
```

---

## 4. ARCHITECTURAL ISSUES & PROBLEMS

### **CRITICAL ISSUES**

1. **Data Model Duplication & Mismatch**
   - `LostItem` (local) vs `LostFoundItem` (Firebase) - two different models
   - HomeScreen uses local `sampleItems`, not Firebase data
   - Manual conversion in BuildingItemsScreen: `LostFoundItem → LostItem`
   - Issue: Local data & Firebase data never sync
   - Impact: Updates to Firebase don't reflect in HomeScreen

2. **Search Only Uses Local Data**
   - SearchScreen searches `sampleItems` only (hardcoded)
   - Firebase items are never searched
   - Path: `SearchUtils.searchAndSort(sampleItems, query)`
   - Missing: `FirebaseService.searchItems(query)` integration
   - Impact: Users can't find items added via Firebase

3. **DetailScreen Broken for Firebase Items**
   - DetailScreen: `val item = sampleItems.getOrNull(index)`
   - Only works for hardcoded items
   - If user clicks Firebase item → index doesn't match → shows "找不到此物品"
   - Missing: Item ID-based routing (should be "detail/{itemId}")

4. **No Global Repository Pattern**
   - Firebase calls scattered across screens (Composables)
   - AddItemScreen, MapScreen, BuildingItemsScreen all call FirebaseService directly
   - No data caching or state management layer
   - No error handling or retry logic
   - Issue: Violates MVVM, screens are too smart

5. **MainViewModel Too Simple**
   - Only manages sort state
   - Should manage: items list, loading state, errors, user auth
   - Firebase init in ViewModel not ideal (should be in Application class)

### **MODERATE ISSUES**

6. **AddItemScreen is Monolithic**
   - 431 lines, 11 mutable states in single Composable
   - Should be broken into smaller components:
     - BuildingSelector (already ModalBottomSheet but could be extracted)
     - PhotoCapture
     - ItemForm
   - No ViewModel for AddItemScreen state

7. **No Error Handling**
   - FirebaseService methods catch exceptions but only log them
   - No error states in UI (except uploadMessage string in AddItemScreen)
   - No retry mechanisms
   - Network failures silently fail

8. **No Loading States**
   - Most screens don't show loading indicators while fetching
   - MapScreen has one, but HomeScreen doesn't
   - Confusing UX if Firebase is slow

9. **Quota Management Underutilized**
   - QuotaManager tracks API calls
   - Only used in MapScreen
   - AddItemScreen could track storage uploads
   - QuotaStatus never displayed in UI

10. **Image Handling Issues**
    - DetailScreen never displays images (placeholder: "無照片")
    - AddItemScreen uploads but no preview
    - No caching - every image fetch is new request

11. **No Pagination**
    - getAllItems() fetches ALL items into memory
    - Firestore query: `.get()` (complete fetch)
    - Should: Use `.limit(20)` + `.startAfter()`

12. **Authentication Gap**
    - Anonymous auth only - no user accounts
    - Can't claim items or get notifications
    - Contact info is "user@example.com" hardcoded

13. **41 Buildings Hardcoded in Code**
    - InitializeSampleData() duplicates all 41 buildings
    - Should be fetched from backend API or config

---

## 5. DATA FLOW GAPS

| Operation | Current Implementation | Issues |
|-----------|----------------------|--------|
| **Display Items** | HomeScreen → sampleItems | Doesn't fetch from Firebase |
| **Search Items** | SearchUtils.searchItems(sampleItems, query) | Only searches hardcoded items |
| **View Item Detail** | DetailScreen(index) with sampleItems[index] | Breaks with Firebase items |
| **Add Item** | AddItemScreen → FirebaseService.addItem() | Works correctly ✓ |
| **List by Building** | MapScreen → getItemsByBuilding() | Works, merges with local items |
| **Update Status** | No UI for claiming/returning items | Not implemented |
| **User Tracking** | Anonymous auth only | No user state |
| **Notifications** | Not implemented | No real-time updates |

---

## 6. RECOMMENDED ARCHITECTURE IMPROVEMENTS

### **Phase 1: Repository Pattern & State Management**
```kotlin
// Create Repository layer
interface ItemRepository {
    suspend fun getItems(): Flow<List<LostFoundItem>>
    suspend fun getItemById(id: String): LostFoundItem?
    suspend fun searchItems(query: String): List<LostFoundItem>
    suspend fun addItem(item: LostFoundItem): Result<String>
    suspend fun getItemsByBuilding(buildingId: String): List<LostFoundItem>
}

class ItemRepositoryImpl(
    private val firebaseService: FirebaseService
) : ItemRepository {
    // Implementation with caching, error handling
}

// Improve ViewModels
class HomeViewModel(private val itemRepository: ItemRepository) : ViewModel() {
    val items: StateFlow<List<LostFoundItem>> = ...
    val isLoading: StateFlow<Boolean> = ...
    val error: StateFlow<String?> = ...
    val sortOption: StateFlow<String> = ...
}
```

### **Phase 2: Unified Data Model**
- Remove `LostItem` - use only `LostFoundItem`
- Add image lazy loading
- Add Parcelable for navigation

### **Phase 3: Fix Navigation**
- DetailScreen: `"detail/{itemId}"` instead of `"detail/{index}"`
- SearchScreen: Search Firebase items, not just local
- Share search results across screens

### **Phase 4: Component Extraction**
- Break AddItemScreen into smaller components
- Extract BuildingSelector as reusable
- Create ItemListViewModel for list screens

### **Phase 5: Error Handling**
- Add Result wrapper to all async operations
- Show error states in UI
- Implement retry logic

---

## 7. TECHNOLOGY STACK

| Layer | Technology |
|-------|-----------|
| UI Framework | Jetpack Compose |
| Architecture | MVVM |
| Navigation | Jetpack Navigation |
| State | StateFlow, SavedStateHandle |
| Backend | Firebase (Firestore, Storage, Auth) |
| Images | Google Maps API, Firebase Storage |
| Location | Location API |
| Camera | Intent-based Camera |
| Data Storage | DataStore (Preferences), Firestore |
| Async | Coroutines, Flow |

---

## 8. SUMMARY

**Architecture Type:** MVVM with Jetpack Compose + Firebase

**Strengths:**
- Clean UI with Compose
- Firebase integration works for write operations
- Google Maps integration with quota protection
- Responsive camera capture

**Weaknesses:**
- Dual data model (local + Firebase) causes inconsistency
- No repository pattern - screens directly call Firebase
- Search only works on hardcoded data
- DetailScreen navigation broken with Firebase items
- Large components (AddItemScreen 431 LOC)
- Limited error handling
- No pagination
- Missing image display

**Next Priority:** Implement Repository Pattern + unify data model to fix data consistency issues

