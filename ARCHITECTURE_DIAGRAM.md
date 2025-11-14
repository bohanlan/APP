# FJU Lost & Found App - Architecture Diagram

## System Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              USER INTERFACE LAYER                          │
│                                 (Jetpack Compose)                          │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  HomeScreen  │  │ SearchScreen │  │ DetailScreen │  │  MapScreen   │  │
│  │   (140 LOC)  │  │  (101 LOC)   │  │  (99 LOC)    │  │ (282 LOC)    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
│         │                  │                 │                 │           │
│         │                  │                 │                 │           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ AddItemScreen│  │BuildingItems │  │  Theme &     │  │ Components   │  │
│  │ (431 LOC ⚠)  │  │   Screen     │  │  Navigation  │  │  (Cards, UI) │  │
│  │              │  │ (130 LOC)    │  │              │  │              │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                          STATE MANAGEMENT LAYER                            │
│                          (ViewModels & State)                              │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  MainViewModel (31 LOC)                                             │  │
│  │  ├─ sortOption: StateFlow<String>  ("最新" / "最舊")                │  │
│  │  ├─ init { initializeAuth() + initializeSampleData() }  ⚠          │  │
│  │  └─ Missing: items, loading, error states                          │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ⚠ Issue: Firebase init should be in Application class, not ViewModel     │
│  ⚠ Too minimal: Only manages sort, should manage all app state            │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                         SERVICE & REPOSITORY LAYER                         │
│                                                                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │  FirebaseService (266 LOC) - Direct calls from screens ⚠            │ │
│  │  ├─ initializeAuth()               (Anonymous login)                │ │
│  │  ├─ uploadImage(Uri) → String      (→ Firebase Storage)             │ │
│  │  ├─ getBuildings() → List<Building>                                 │ │
│  │  ├─ getItemsByBuilding(id)                                          │ │
│  │  ├─ getAllItems()                  (Loads ALL into memory ⚠)        │ │
│  │  ├─ addItem(LostFoundItem)         (Create - works ✓)              │ │
│  │  ├─ updateItemStatus()             (Update - works ✓)              │ │
│  │  ├─ searchItems(query)             (Search - not used ⚠)           │ │
│  │  └─ initializeSampleData()         (Seed 41 buildings)             │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ⚠ Issues:                                                                 │
│    - No repository pattern - screens call directly                         │
│    - No caching or state persistence                                       │
│    - getAllItems() loads everything - no pagination                        │
│    - searchItems() never called (search uses local data)                   │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │  QuotaManager (152 LOC) - Google Maps API quota tracking            │ │
│  │  ├─ Daily limit: 25,000 calls/day                                    │ │
│  │  ├─ Warning: 80% (20,000 calls)                                      │ │
│  │  ├─ Auto-disable: Map at 25,000 calls                                │ │
│  │  └─ Storage: Android DataStore                                       │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                            UTILITIES LAYER                                 │
│                                                                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SearchUtils (43 LOC)          TimeUtils (62 LOC)                          │
│  ├─ searchItems()              ├─ parseTimeToMillis()                      │
│  ├─ sortItems()                └─ Parse Chinese dates                      │
│  └─ searchAndSort()                                                        │
│     ⚠ Issue: Only searches sampleItems, not Firebase                       │
│                                                                             │
│  StringUtils (58 LOC)          LocationUtils (126 LOC)                     │
│  ├─ displayLocation()          ├─ getCurrentLocation()                     │
│  └─ expandedQueries()          └─ Default: FJU Continuing Ed Bldg          │
│                                                                             │
│  CameraHelper (114 LOC)        CameraUtils (58 LOC)                        │
│  └─ takePicture()              └─ Camera utilities                         │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                              DATA LAYER                                    │
│                                                                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─── LOCAL DATA ──────────────────┐   ┌─── FIREBASE BACKEND ──────────┐ │
│  │                                  │   │                               │ │
│  │  sampleItems (hardcoded)        │   │  Firestore Database:          │ │
│  │  └─ 9 LostItem objects          │   │  ├─ collections/              │ │
│  │     (used in HomeScreen only)   │   │  │   ├─ buildings/ (41 docs)  │ │
│  │     ⚠ NEVER UPDATED             │   │  │   │   └─ {id, name, ...}   │ │
│  │                                  │   │  │   └─ lostItems/ (user add) │ │
│  │                                  │   │  │       └─ {uuid, ...}      │ │
│  │  Model: LostItem (legacy)        │   │  │                           │ │
│  │  ├─ name: String                 │   │  │  Firebase Storage:        │ │
│  │  ├─ location: String             │   │  │  └─ items/{itemId}/*.jpg  │ │
│  │  ├─ time: String (Chinese)       │   │  │                           │ │
│  │  └─ buildingId: String           │   │  │  Firebase Auth:           │ │
│  │                                  │   │  │  └─ Anonymous users       │ │
│  └─────────────────────────────────┘   │                               │
│                                         │  DataStore (Local):           │
│                                         │  └─ Quota tracking           │
│                                         └───────────────────────────────┘
│                                                                             │
│  ⚠ Issue: Two separate data stores                                        │
│    - Local sampleItems never updates                                      │
│    - Firebase data isolated from local data                               │
│    - Manual conversion needed (LostFoundItem → LostItem)                  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Model Mismatch

```
┌─────────────────────────────────────┐     ┌─────────────────────────────────────┐
│         LOCAL DATA MODEL            │     │      FIREBASE DATA MODEL            │
├─────────────────────────────────────┤     ├─────────────────────────────────────┤
│                                     │     │                                     │
│  LostItem (used in HomeScreen):     │     │  LostFoundItem (from Firestore):    │
│  ├─ name: String                    │     │  ├─ id: String  (UUID)              │
│  ├─ location: String                │     │  ├─ name: String                    │
│  ├─ time: String ("2 小時前")        │     │  ├─ description: String             │
│  └─ buildingId: String              │     │  ├─ buildingId: String              │
│                                     │     │  ├─ location: String                │
│  ✓ Used: HomeScreen                 │     │  ├─ imageUrl: String (Storage URL)  │
│  ✓ Used: SearchScreen               │     │  ├─ thumbnailUrl: String            │
│  ✓ Used: DetailScreen               │     │  ├─ tags: List<String>              │
│                                     │     │  ├─ timestamp: Long (epoch millis)  │
│                                     │     │  ├─ status: String                  │
│                                     │     │  ├─ contact: String                 │
│                                     │     │  ├─ phone: String                   │
│                                     │     │  ├─ uploadedBy: String (User ID)    │
│                                     │     │  └─ category: String ("found"/"lost")│
│                                     │     │                                     │
│                                     │     │  ✓ Used: MapScreen                  │
│                                     │     │  ✓ Used: AddItemScreen              │
│                                     │     │  ✓ Used: BuildingItemsScreen        │
│                                     │     │  ✗ NOT Used: HomeScreen             │
│                                     │     │  ✗ NOT Used: SearchScreen           │
│                                     │     │                                     │
│                                     │     │  Firestore Doc Path:                │
│                                     │     │  lostItems/{uuid}                   │
│                                     │     │                                     │
└─────────────────────────────────────┘     └─────────────────────────────────────┘

                                    ⚠ CONFLICT ⚠
                                    
    HomeScreen/SearchScreen only use LostItem from sampleItems
    → Firebase items are NEVER displayed in these screens
    → Data inconsistency: old vs new items
```

---

## Current Data Flows

### Flow 1: Add Item (CREATE) - Works ✓
```
AddItemScreen
    │
    ├─ User selects building from 41 options
    │  └─ FirebaseService.getBuildings() [Firebase]
    │
    ├─ User takes photo
    │  └─ CameraHelper.takePicture() → Uri
    │
    ├─ User enters name & description
    │
    └─ Submit button clicked
         │
         ▼
    FirebaseService.uploadImage(Uri)
         │
         ├─ Firebase Storage.putFile()
         │  └─ items/{itemId}/image_*.jpg
         │
         └─ Returns: download URL
              │
              ▼
         Create LostFoundItem object
              │
              ├─ id: UUID
              ├─ imageUrl: from Storage
              ├─ buildingId: from selected building
              ├─ timestamp: now
              ├─ uploadedBy: FirebaseService.getCurrentUserId()
              └─ ... other fields
              │
              ▼
         FirebaseService.addItem(LostFoundItem)
              │
              └─ Firestore.lostItems/{uuid}.set()
                   │
                   ✓ Successfully saved!
```

### Flow 2: Display Items (READ) - BROKEN ✗

#### 2a. HomeScreen Display
```
HomePage composable
    │
    ├─ sampleItems = hardcoded list of 9 LostItems
    │
    └─ Display items
         │
         ⚠ PROBLEM: Firebase items are NEVER fetched
         ⚠ Users only see old hardcoded items
         ⚠ New items added via AddItemScreen are invisible
```

#### 2b. Search - BROKEN ✗
```
User enters search query in HomeScreen
    │
    ▼
SearchResultScreen(query)
    │
    ├─ SearchUtils.searchAndSort(sampleItems, query)
    │  └─ Searches ONLY in sampleItems (9 hardcoded items)
    │
    └─ Display results
         │
         ⚠ PROBLEM: Firebase items are NEVER searched
         ⚠ searchItems() method in FirebaseService is unused
         ⚠ Users can't find items added via AddItemScreen
```

#### 2c. Map Display - PARTIALLY WORKS ✓
```
MapScreen composable
    │
    ├─ FirebaseService.getBuildings()
    │  └─ Fetches 41 Building objects from Firestore
    │
    ├─ For each building:
    │  └─ FirebaseService.getItemsByBuilding(buildingId)
    │     └─ Count items in this building
    │
    ├─ Display markers (only if itemCount > 0)
    │
    └─ On marker click
         │
         ├─ Navigate to BuildingItemsScreen
         │
         ▼
    BuildingItemsScreen(buildingId)
         │
         ├─ FirebaseService.getItemsByBuilding(buildingId)
         │  └─ Returns: List<LostFoundItem> from Firestore
         │
         ├─ Convert: LostFoundItem → LostItem
         │  (manual mapping to compatible UI model)
         │
         ├─ Merge with local sampleItems (same buildingId)
         │
         └─ Display combined list
              │
              ⚠ ISSUE: Navigation returns index, not itemId
              ⚠ DetailScreen expects sampleItems[index]
              ⚠ Firebase items will show "找不到此物品"
```

#### 2d. Detail View - BROKEN ✗
```
DetailScreen(index)
    │
    ├─ item = sampleItems.getOrNull(index)
    │  └─ Only works for hardcoded items!
    │
    ├─ If item == null
    │  └─ Show "找不到此物品"
    │
    ⚠ PROBLEM: Navigation should pass itemId, not index
    ⚠ PROBLEM: Can't display Firebase items
    ⚠ PROBLEM: Can't show images (DetailScreen hardcodes "無照片")
    ⚠ PROBLEM: Can't show contact info or status
```

---

## Component Dependencies

```
MainActivity (Entry point)
    │
    └─ AppNavigation (NavHost setup)
         │
         ├─ HomeScreen
         │  ├─ MainViewModel (sort state)
         │  ├─ SearchUtils (search local data)
         │  └─ LostItemCard component
         │
         ├─ SearchResultScreen
         │  ├─ MainViewModel
         │  ├─ SearchUtils (⚠ local data only)
         │  └─ LostItemCard
         │
         ├─ DetailScreen
         │  ├─ sampleItems (⚠ hardcoded, index-based)
         │  └─ StringUtils
         │
         ├─ MapScreen
         │  ├─ FirebaseService
         │  ├─ QuotaManager
         │  ├─ LocationUtils
         │  └─ Google Maps Compose library
         │
         ├─ AddItemScreen
         │  ├─ FirebaseService (upload + add)
         │  ├─ CameraHelper (take photo)
         │  └─ ModalBottomSheet (building selector)
         │
         └─ BuildingItemsScreen
            ├─ FirebaseService (get items by building)
            ├─ sampleItems (merge with Firebase)
            └─ LostItemCard
```

---

## Issue Summary with Visual Markers

| # | Issue | Severity | Component | Impact |
|---|-------|----------|-----------|--------|
| 1 | Data model duplication | CRITICAL ⚠⚠ | Models | Data inconsistency across app |
| 2 | Search uses local only | CRITICAL ⚠⚠ | SearchScreen | Users can't find Firebase items |
| 3 | DetailScreen broken | CRITICAL ⚠⚠ | DetailScreen | Can't view Firebase item details |
| 4 | No Repository pattern | CRITICAL ⚠⚠ | Services | Screens too smart, no reusability |
| 5 | AddItemScreen monolithic | HIGH ⚠ | AddItemScreen | 431 LOC, 11 states, hard to test |
| 6 | No error handling | HIGH ⚠ | Services/UI | Network failures silently fail |
| 7 | No pagination | HIGH ⚠ | Services | getAllItems() loads everything |
| 8 | No image display | MEDIUM | DetailScreen | Photos uploaded but never shown |
| 9 | Limited ViewModel | MEDIUM | MainViewModel | Doesn't manage app state properly |
| 10 | Buildings hardcoded | LOW | FirebaseService | Maintenance burden, not scalable |

---

## Navigation Graph

```
┌──────────────┐
│   home       │◄────────────────────────────────────────┐
└──────┬───────┘                                         │
       │ search/{query}                                  │
       │                                                 │
       ├─►┌────────────────────┐                         │
       │  │  search            │─────────────────────────┤
       │  └────────────────────┘  detail/{index}  (broken)
       │                              ▲
       │                              │
       ├─►┌────────────────────┐      │
       │  │  detail/{index}    │◄─────┘ (broken: index-based)
       │  └────────────────────┘
       │
       │ map
       │
       ├─►┌────────────────────┐
       │  │  map               │
       │  └────────┬───────────┘
       │           │ buildingItems/{buildingId}/{name}
       │           │
       │           ├─►┌────────────────────┐
       │              │ buildingItems      │
       │              └────────────────────┘
       │
       │ addItem
       │
       ├─►┌────────────────────┐
          │  addItem           │───────┐
          └────────────────────┘       │
                                       │ (on success)
                                       └─► home

⚠ Navigation Issues:
  - detail/{index} should be detail/{itemId}
  - Search only works with local data
  - No deep linking
  - No saved navigation state
```

