# FJU Lost & Found App - Quick Reference Guide

## File Locations

**All files located in:** `/Users/daniellan/Documents/APP/app/src/main/java/tw/edu/fju/myapplication/`

### By Category

**Models (Data Classes)**
- `/models/LostItem.kt` - Legacy local model (36 LOC)
- `/models/FirebaseModels.kt` - Building, LostFoundItem, User (55 LOC)

**Services**
- `/services/FirebaseService.kt` - All Firebase operations (266 LOC)
- `/services/QuotaManager.kt` - Google Maps quota tracking (152 LOC)

**Screens (UI)**
- `/ui/screens/HomeScreen.kt` - Main landing page (140 LOC)
- `/ui/screens/SearchScreen.kt` - Search results (101 LOC)
- `/ui/screens/DetailScreen.kt` - Item details (99 LOC)
- `/ui/screens/AddItemScreen.kt` - Report item (431 LOC) ⚠ Largest
- `/ui/screens/MapScreen.kt` - Google Maps view (282 LOC)
- `/ui/screens/BuildingItemsScreen.kt` - Building items (130 LOC)

**Components**
- `/ui/components/LostItemCard.kt` - Item card (68 LOC)
- `/ui/components/SortButtons.kt` - Sort controls (68 LOC)
- `/ui/components/BuildingMarkerInfo.kt` - Map marker (17 LOC)

**Utilities**
- `/utils/SearchUtils.kt` - Search logic (43 LOC)
- `/utils/TimeUtils.kt` - Time parsing (62 LOC)
- `/utils/StringUtils.kt` - String utilities (58 LOC)
- `/utils/LocationUtils.kt` - GPS/location (126 LOC)
- `/utils/CameraHelper.kt` - Camera integration (114 LOC)
- `/utils/CameraUtils.kt` - Camera utils (58 LOC)
- `/utils/SearchHelper.kt` - Search helpers (50 LOC)

**Entry Point**
- `/MainActivity.kt` - App entry + navigation (90 LOC)
- `/MainViewModel.kt` - Global state (31 LOC)

**Styling**
- `/ui/theme/Color.kt`, `Theme.kt`, `Type.kt`

---

## Key Architecture Points

### MVVM Pattern
- **View:** Jetpack Compose screens
- **ViewModel:** MainViewModel (minimal)
- **Model:** Firebase data models + sampleItems
- **Backend:** Firebase Firestore + Storage + Auth

### Navigation Routes
```
home → HomePage
search/{query} → SearchResultScreen
detail/{index} → DetailScreen ⚠ Should be {itemId}
map → MapScreen
addItem → AddItemScreen
buildingItems/{buildingId}/{buildingName} → BuildingItemsScreen
```

### Data Sources
```
Local (In-Memory):
├─ sampleItems (9 hardcoded LostItems)
└─ MainViewModel (sort state)

Firebase:
├─ Firestore/buildings/ (41 documents)
├─ Firestore/lostItems/ (user submissions)
├─ Storage/items/ (photos)
└─ Auth (anonymous)

Local Storage:
└─ DataStore (quota tracking)
```

---

## Critical Issues Summary

### Issue #1: Data Model Duplication
**Severity:** CRITICAL ⚠⚠
**Problem:** Two separate item models
- `LostItem` - Used in HomeScreen, SearchScreen, DetailScreen
- `LostFoundItem` - Used in MapScreen, AddItemScreen, BuildingItemsScreen

**Impact:** 
- HomeScreen never shows Firebase items
- SearchScreen can't find Firebase items
- DetailScreen crashes with Firebase items

**Location:** `/models/LostItem.kt` vs `/models/FirebaseModels.kt`

### Issue #2: Search Only Uses Local Data
**Severity:** CRITICAL ⚠⚠
**Problem:** SearchUtils only searches sampleItems
**Impact:** Users can't find items added via AddItemScreen
**Location:** `/utils/SearchUtils.kt` line 12-19

### Issue #3: DetailScreen Index-Based Navigation
**Severity:** CRITICAL ⚠⚠
**Problem:** DetailScreen uses sampleItems[index] instead of itemId
**Impact:** Can't view Firebase item details
**Location:** `/ui/screens/DetailScreen.kt` line 33

### Issue #4: No Repository Pattern
**Severity:** CRITICAL ⚠⚠
**Problem:** Screens directly call FirebaseService
**Impact:** No code reuse, scattered error handling, testability issues
**Locations:** 
- `/ui/screens/MapScreen.kt` line 87-117
- `/ui/screens/AddItemScreen.kt` line 106-113
- `/ui/screens/BuildingItemsScreen.kt` line 52-57

### Issue #5: AddItemScreen Monolithic
**Severity:** HIGH ⚠
**Problem:** 431 LOC with 11 mutable states in single Composable
**Impact:** Hard to test, maintain, and understand
**Location:** `/ui/screens/AddItemScreen.kt`

---

## Data Flow Breakdown

### Working Flows
1. **Add Item** - AddItemScreen → Firebase ✓
   - Photo upload → Storage
   - Item save → Firestore
   - Path: AddItemScreen.kt (371-408)

2. **Map Display** - Firebase → MapScreen ✓
   - Load buildings → Display markers
   - Path: MapScreen.kt (87-117)

### Broken Flows
1. **Display Items** - HomeScreen ✗
   - Only shows hardcoded sampleItems
   - Never fetches Firebase items
   - Path: HomeScreen.kt (48-55)

2. **Search** - SearchScreen ✗
   - Only searches sampleItems
   - Ignores Firebase items
   - Path: SearchScreen.kt (40-46)

3. **View Details** - DetailScreen ✗
   - Only works with sampleItems[index]
   - Crashes with Firebase items
   - Path: DetailScreen.kt (32-33)

---

## Component Connection Map

```
HomeScreen ──────► sampleItems (local)
    ↓
    └─► SearchUtils.searchAndSort()
         └─► SearchResultScreen

AddItemScreen ──────► FirebaseService
    ├─► getBuildings() → building selector
    ├─► uploadImage() → Storage
    └─► addItem() → Firestore

MapScreen ──────► FirebaseService
    ├─► getBuildings() → Display markers
    ├─► getItemsByBuilding() → Count items
    └─► QuotaManager → Track API calls

BuildingItemsScreen ──────► FirebaseService
    ├─► getItemsByBuilding() → Firebase items
    └─► sampleItems → Merge local items

DetailScreen ──────► sampleItems[index]
    └─► ⚠ Only works for local items
```

---

## Total Codebase Stats

| Metric | Value |
|--------|-------|
| **Total Files** | 25 Kotlin files |
| **Total LOC** | 2,540 lines |
| **Largest File** | AddItemScreen.kt (431 LOC) |
| **Average LOC per File** | 102 LOC |
| **Firebase Operations** | 10 methods |
| **Screens** | 6 screens |
| **UI Components** | 3 components |
| **Utilities** | 7 utilities |

---

## Recommended Reading Order

1. **Start here:** `/ARCHITECTURE_ANALYSIS.md` (comprehensive analysis)
2. **Visual guide:** `/ARCHITECTURE_DIAGRAM.md` (data flows + diagrams)
3. **Code walkthrough:**
   - `MainActivity.kt` - Navigation setup
   - `MainViewModel.kt` - State management
   - `FirebaseService.kt` - Backend operations
   - `HomeScreen.kt` - Main UI
   - `AddItemScreen.kt` - Form handling
   - `MapScreen.kt` - Map integration

---

## Quick Fixes (Priority Order)

### Priority 1: Unify Data Models
Replace `LostItem` with `LostFoundItem` everywhere
- Estimated effort: 2-3 hours
- Impact: Fixes HomeScreen, SearchScreen data consistency

### Priority 2: Implement Repository Pattern
Create `ItemRepository` interface + implementation
- Estimated effort: 3-4 hours
- Impact: Centralizes Firebase calls, enables caching

### Priority 3: Fix Navigation
Change DetailScreen to use itemId instead of index
- Estimated effort: 1-2 hours
- Impact: Enables Firebase item details viewing

### Priority 4: Search Firebase Items
Integrate `FirebaseService.searchItems()` into SearchScreen
- Estimated effort: 1 hour
- Impact: Users can find all items

### Priority 5: Extract AddItemScreen Components
Break into BuildingSelector, PhotoCapture, ItemForm
- Estimated effort: 2-3 hours
- Impact: Improved maintainability, testability

---

## Firebase Collections Structure

```
Firestore Database (projectId: fju-[censored])

📦 buildings/ (41 documents)
   ├─ LI: Building(name="文華樓", lat=25.0352, lng=121.4322, ...)
   ├─ LF: Building(name="文友樓", ...)
   ├─ LE: Building(name="文開樓", ...)
   └─ ... 38 more buildings

📦 lostItems/ (user submissions)
   ├─ {uuid1}: LostFoundItem(name="黑色錢包", buildingId="LI", ...)
   ├─ {uuid2}: LostFoundItem(name="藍色水壺", buildingId="LF", ...)
   └─ ... (new items added via AddItemScreen)

📦 users/ (optional, not used yet)
   └─ {uid}: User(email="...", uploadedItems=[...], ...)

Firebase Storage (gs://bucket)

📦 items/
   ├─ {itemId1}/
   │  └─ image_1700000000.jpg
   ├─ {itemId2}/
   │  └─ image_1700000100.jpg
   └─ ... (photos from AddItemScreen)
```

---

## Environment & Setup

**Android Version:** Target 34 (Android 14)
**Kotlin Version:** Latest (1.9.x)
**Build System:** Gradle (Kotlin DSL)

**Key Dependencies:**
- Jetpack Compose
- Firebase (Auth, Firestore, Storage)
- Google Maps Android Compose
- Coroutines & Flow
- DataStore Preferences

**Permissions:**
- CAMERA (for photo capture)
- INTERNET (for Firebase)
- ACCESS_FINE_LOCATION (for GPS)

---

## Testing Considerations

### Currently Missing Tests
- No unit tests for SearchUtils, TimeUtils, etc.
- No UI tests for Composables
- No integration tests for Firebase operations
- No repository pattern → harder to mock

### Suggested Test Strategy
1. Create `ItemRepository` interface (mockable)
2. Add unit tests for utils (SearchUtils, TimeUtils)
3. Add ViewModel tests with MockRepository
4. Add integration tests with Firebase Emulator

---

## Next Steps

1. **Read** `/ARCHITECTURE_ANALYSIS.md` for complete context
2. **Review** `/ARCHITECTURE_DIAGRAM.md` for visual understanding
3. **Identify** which issues to fix first based on priority
4. **Implement** Repository Pattern (Phase 1)
5. **Unify** data models (Phase 2)
6. **Fix** navigation and search (Phase 3)

