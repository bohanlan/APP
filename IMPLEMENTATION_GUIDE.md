# 实现指南 - 快速参考

本指南提供了实现分页、分类、图片显示的具体代码示例。

## 快速导航

- [分页实现](#分页实现)
- [分类实现](#分类实现) 
- [图片显示实现](#图片显示实现)

---

## 分页实现

### Step 1: 更新 Repository

**文件**: `/Users/daniellan/Documents/APP/app/src/main/java/tw/edu/fju/myapplication/repository/LostFoundRepository.kt`

添加以下数据类和方法：

```kotlin
// 在 LostFoundRepository.kt 顶部添加
data class PaginationResult(
    val items: List<LostFoundItem>,
    val nextCursor: DocumentSnapshot?
)

// 在 LostFoundRepository 类中添加方法
/**
 * 分页获取所有物品 (使用 cursor)
 * @param pageSize 每页大小 (默认20)
 * @param lastDocumentSnapshot 上一页的最后一个文档 (首页为 null)
 */
suspend fun getItemsPaginated(
    pageSize: Int = 20,
    lastDocumentSnapshot: DocumentSnapshot? = null
): Result<PaginationResult> = withContext(Dispatchers.IO) {
    try {
        var query: Query = firestore.collection(ITEMS_COLLECTION)
            .whereEqualTo("status", "available")
            .orderBy("timestamp", Query.Direction.DESCENDING)
            .limit((pageSize + 1).toLong())
        
        if (lastDocumentSnapshot != null) {
            query = query.startAfter(lastDocumentSnapshot)
        }
        
        val snapshot = query.get().await()
        val items = snapshot.documents
            .take(pageSize)
            .map { it.toObject(LostFoundItem::class.java)!! }
        
        val nextCursor = if (snapshot.documents.size > pageSize) {
            snapshot.documents[pageSize]
        } else null
        
        Log.d(TAG, "✓ 分页查询成功: ${items.size} 件物品")
        Result.success(PaginationResult(items, nextCursor))
    } catch (e: Exception) {
        Log.e(TAG, "✗ 分页查询失败: ${e.message}", e)
        Result.failure(e)
    }
}
```

### Step 2: 更新 ViewModel

**文件**: `/Users/daniellan/Documents/APP/app/src/main/java/tw/edu/fju/myapplication/ui/viewmodel/MainViewModel.kt`

```kotlin
// 在 MainViewModel 类中添加
private val _paginatedItems = MutableStateFlow<List<LostFoundItem>>(emptyList())
val paginatedItems: StateFlow<List<LostFoundItem>> = _paginatedItems.asStateFlow()

private val _paginationCursor = MutableStateFlow<DocumentSnapshot?>(null)
val paginationCursor: StateFlow<DocumentSnapshot?> = _paginationCursor.asStateFlow()

private val _hasMoreItems = MutableStateFlow(true)
val hasMoreItems: StateFlow<Boolean> = _hasMoreItems.asStateFlow()

private val _paginationLoading = MutableStateFlow(false)
val paginationLoading: StateFlow<Boolean> = _paginationLoading.asStateFlow()

/**
 * 加载第一页
 */
fun loadFirstPage() {
    viewModelScope.launch {
        _paginationLoading.value = true
        val result = repository.getItemsPaginated(pageSize = 20, lastDocumentSnapshot = null)
        result
            .onSuccess { paginationResult ->
                _paginatedItems.value = paginationResult.items
                _paginationCursor.value = paginationResult.nextCursor
                _hasMoreItems.value = paginationResult.nextCursor != null
                Log.d(TAG, "✓ 第一页加载成功: ${paginationResult.items.size} 件")
            }
            .onFailure { error ->
                Log.e(TAG, "✗ 第一页加载失败: ${error.message}")
            }
        _paginationLoading.value = false
    }
}

/**
 * 加载下一页
 */
fun loadNextPage() {
    val cursor = _paginationCursor.value ?: return
    
    viewModelScope.launch {
        _paginationLoading.value = true
        val result = repository.getItemsPaginated(pageSize = 20, lastDocumentSnapshot = cursor)
        result
            .onSuccess { paginationResult ->
                _paginatedItems.value = _paginatedItems.value + paginationResult.items
                _paginationCursor.value = paginationResult.nextCursor
                _hasMoreItems.value = paginationResult.nextCursor != null
                Log.d(TAG, "✓ 下一页加载成功: ${paginationResult.items.size} 件")
            }
            .onFailure { error ->
                Log.e(TAG, "✗ 下一页加载失败: ${error.message}")
            }
        _paginationLoading.value = false
    }
}
```

### Step 3: 更新 UI (HomeScreen)

**文件**: `/Users/daniellan/Documents/APP/app/src/main/java/tw/edu/fju/myapplication/ui/screens/HomeScreen.kt`

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun HomePage(navController: NavController, vm: MainViewModel = viewModel()) {
    var searchQuery by remember { mutableStateOf("") }
    
    // 订阅分页状态
    val paginatedItems by vm.paginatedItems.collectAsState()
    val hasMoreItems by vm.hasMoreItems.collectAsState()
    val isLoading by vm.paginationLoading.collectAsState()
    
    // 初始化加载第一页
    LaunchedEffect(Unit) {
        vm.loadFirstPage()
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("輔大失物招領") },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primary,
                    titleContentColor = Color.White
                )
            )
        }
    ) { innerPadding ->
        Column(modifier = Modifier.padding(innerPadding)) {
            // 搜尋框和其他 UI...
            
            // 物品列表 - 改为分页列表
            LazyColumn(verticalArrangement = Arrangement.spacedBy(8.dp)) {
                itemsIndexed(paginatedItems) { index, item ->
                    LostItemCard(
                        item = LostItem(
                            name = item.name,
                            location = item.location,
                            time = item.timestamp.toString(),
                            buildingId = item.buildingId
                        ),
                        onClick = { /* 导航到详情 */ }
                    )
                    
                    // 当到达列表末尾时，加载下一页
                    if (index == paginatedItems.size - 1 && hasMoreItems && !isLoading) {
                        LaunchedEffect(Unit) {
                            vm.loadNextPage()
                        }
                    }
                }
                
                // 加载更多提示
                if (isLoading) {
                    item {
                        Box(
                            modifier = Modifier
                                .fillMaxWidth()
                                .padding(16.dp),
                            contentAlignment = Alignment.Center
                        ) {
                            CircularProgressIndicator()
                        }
                    }
                }
                
                // 没有更多数据
                if (!hasMoreItems && paginatedItems.isNotEmpty()) {
                    item {
                        Box(
                            modifier = Modifier
                                .fillMaxWidth()
                                .padding(16.dp),
                            contentAlignment = Alignment.Center
                        ) {
                            Text("已加载全部内容", color = Color.Gray)
                        }
                    }
                }
            }
        }
    }
}
```

---

## 分类实现

### Step 1: 添加 Repository 方法

**文件**: `/Users/daniellan/Documents/APP/app/src/main/java/tw/edu/fju/myapplication/repository/LostFoundRepository.kt`

```kotlin
/**
 * 按分类获取物品
 */
suspend fun getItemsByCategory(category: String): Result<List<LostFoundItem>> = 
    withContext(Dispatchers.IO) {
    try {
        val snapshot = firestore.collection(ITEMS_COLLECTION)
            .whereEqualTo("category", category)
            .whereEqualTo("status", "available")
            .orderBy("timestamp", Query.Direction.DESCENDING)
            .get()
            .await()
        
        val items = snapshot.toObjects(LostFoundItem::class.java)
        Log.d(TAG, "✓ 获取 '$category' 分类物品: ${items.size} 件")
        Result.success(items)
    } catch (e: Exception) {
        Log.e(TAG, "✗ 获取分类物品失败: ${e.message}", e)
        Result.failure(e)
    }
}
```

### Step 2: 更新 ViewModel

**文件**: `/Users/daniellan/Documents/APP/app/src/main/java/tw/edu/fju/myapplication/ui/viewmodel/MainViewModel.kt`

```kotlin
// 添加分类相关状态
private val _selectedCategory = MutableStateFlow("found")  // "found" 或 "lost"
val selectedCategory: StateFlow<String> = _selectedCategory.asStateFlow()

private val _itemsByCategory = MutableStateFlow<List<LostFoundItem>>(emptyList())
val itemsByCategory: StateFlow<List<LostFoundItem>> = _itemsByCategory.asStateFlow()

/**
 * 设置分类并加载数据
 */
fun setCategory(category: String) {
    _selectedCategory.value = category
    loadItemsByCategory(category)
}

/**
 * 按分类加载物品
 */
fun loadItemsByCategory(category: String) {
    viewModelScope.launch {
        _itemsLoading.value = true
        
        val result = repository.getItemsByCategory(category)
        result
            .onSuccess { items ->
                _itemsByCategory.value = items
                Log.d(TAG, "✓ 加载 '$category' 分类: ${items.size} 件")
            }
            .onFailure { error ->
                _itemsError.value = error.message
                Log.e(TAG, "✗ 加载分类失败: ${error.message}")
            }
        
        _itemsLoading.value = false
    }
}
```

### Step 3: 更新 AddItemScreen

**文件**: `/Users/daniellan/Documents/APP/app/src/main/java/tw/edu/fju/myapplication/ui/screens/AddItemScreen.kt`

```kotlin
// 添加分类选择状态 (在 @Composable 函数中)
var selectedCategory by remember { mutableStateOf("found") }

// 在 UI 中添加分类选择按钮 (在第2步和第3步之间)
Spacer(modifier = Modifier.height(24.dp))

Text(
    "第 2.5 步：選擇分類",
    style = MaterialTheme.typography.titleMedium,
    modifier = Modifier
        .fillMaxWidth()
        .padding(bottom = 8.dp)
)

Row(
    modifier = Modifier
        .fillMaxWidth()
        .padding(bottom = 16.dp),
    horizontalArrangement = Arrangement.SpaceEvenly
) {
    Button(
        onClick = { selectedCategory = "lost" },
        modifier = Modifier.weight(1f),
        colors = ButtonDefaults.buttonColors(
            containerColor = if (selectedCategory == "lost") 
                MaterialTheme.colorScheme.primary 
            else 
                MaterialTheme.colorScheme.surfaceVariant
        )
    ) {
        Text(
            "遺失",
            color = if (selectedCategory == "lost") Color.White else Color.Black
        )
    }
    
    Spacer(modifier = Modifier.width(8.dp))
    
    Button(
        onClick = { selectedCategory = "found" },
        modifier = Modifier.weight(1f),
        colors = ButtonDefaults.buttonColors(
            containerColor = if (selectedCategory == "found") 
                MaterialTheme.colorScheme.primary 
            else 
                MaterialTheme.colorScheme.surfaceVariant
        )
    ) {
        Text(
            "拾獲",
            color = if (selectedCategory == "found") Color.White else Color.Black
        )
    }
}

// 修改新增按钮中的分类字段
val lostItem = LostFoundItem(
    name = itemName,
    description = itemDescription,
    buildingId = selectedBuilding!!.id,
    location = selectedBuilding!!.name,
    imageUrl = imageUrl,
    tags = listOf(itemName),
    status = "available",
    contact = "user@example.com",
    category = selectedCategory  // ✅ 使用用户选择的分类
)
```

### Step 4: 更新 HomeScreen (添加分类标签页)

**文件**: `/Users/daniellan/Documents/APP/app/src/main/java/tw/edu/fju/myapplication/ui/screens/HomeScreen.kt`

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun HomePage(navController: NavController, vm: MainViewModel = viewModel()) {
    val selectedCategory by vm.selectedCategory.collectAsState()
    val itemsByCategory by vm.itemsByCategory.collectAsState()
    
    Column {
        // TopAppBar...
        
        // 分类标签页
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
            horizontalArrangement = Arrangement.Center
        ) {
            Button(
                onClick = { vm.setCategory("found") },
                modifier = Modifier.weight(1f),
                colors = ButtonDefaults.buttonColors(
                    containerColor = if (selectedCategory == "found")
                        MaterialTheme.colorScheme.primary
                    else
                        MaterialTheme.colorScheme.surfaceVariant
                )
            ) {
                Text("拾獲", color = if (selectedCategory == "found") Color.White else Color.Black)
            }
            
            Spacer(modifier = Modifier.width(8.dp))
            
            Button(
                onClick = { vm.setCategory("lost") },
                modifier = Modifier.weight(1f),
                colors = ButtonDefaults.buttonColors(
                    containerColor = if (selectedCategory == "lost")
                        MaterialTheme.colorScheme.primary
                    else
                        MaterialTheme.colorScheme.surfaceVariant
                )
            ) {
                Text("遺失", color = if (selectedCategory == "lost") Color.White else Color.Black)
            }
        }
        
        // 物品列表
        LazyColumn {
            itemsIndexed(itemsByCategory) { _, item ->
                LostItemCard(
                    item = LostItem(
                        name = item.name,
                        location = item.location,
                        time = item.timestamp.toString(),
                        buildingId = item.buildingId
                    ),
                    onClick = { /* 导航 */ }
                )
            }
        }
    }
}
```

---

## 图片显示实现

### Step 1: 添加依赖

**文件**: `build.gradle.kts`

```kotlin
// 添加 Coil 图片加载库
implementation("io.coil-kt:coil-compose:2.4.0")
```

### Step 2: 创建图片显示组件

**新文件**: `/Users/daniellan/Documents/APP/app/src/main/java/tw/edu/fju/myapplication/ui/components/ItemImage.kt`

```kotlin
package tw.edu.fju.myapplication.ui.components

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.unit.dp
import coil.compose.AsyncImage

/**
 * 显示物品图片的 Composable
 */
@Composable
fun ItemImage(
    imageUrl: String?,
    contentDescription: String = "物品图片",
    modifier: Modifier = Modifier
        .fillMaxWidth()
        .height(200.dp)
) {
    if (imageUrl.isNullOrEmpty()) {
        // 无图片时显示占位符
        Box(
            modifier = modifier
                .background(Color.LightGray)
                .clip(RoundedCornerShape(8.dp)),
            contentAlignment = Alignment.Center
        ) {
            Text("無照片", color = Color.Gray)
        }
    } else {
        // 有图片时使用 AsyncImage 显示
        AsyncImage(
            model = imageUrl,
            contentDescription = contentDescription,
            modifier = modifier.clip(RoundedCornerShape(8.dp)),
            contentScale = ContentScale.Crop,
            loading = {
                Box(
                    modifier = Modifier.fillMaxSize(),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator()
                }
            },
            error = {
                Box(
                    modifier = Modifier
                        .fillMaxSize()
                        .background(Color.LightGray),
                    contentAlignment = Alignment.Center
                ) {
                    Text("圖片載入失敗", color = Color.Gray)
                }
            }
        )
    }
}
```

### Step 3: 更新 DetailScreen

**文件**: `/Users/daniellan/Documents/APP/app/src/main/java/tw/edu/fju/myapplication/ui/screens/DetailScreen.kt`

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DetailScreen(navController: NavController, index: Int) {
    val item = sampleItems.getOrNull(index)

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("失物詳情") },
                navigationIcon = {
                    IconButton(onClick = { navController.popBackStack() }) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, "返回", tint = Color.White)
                    }
                }
            )
        }
    ) { innerPadding ->
        if (item == null) {
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .padding(innerPadding),
                contentAlignment = Alignment.Center
            ) {
                Text("找不到此物品")
            }
        } else {
            Column(
                modifier = Modifier
                    .padding(innerPadding)
                    .padding(16.dp)
                    .fillMaxSize()
            ) {
                // ✅ 使用新的 ItemImage 组件显示图片
                ItemImage(
                    imageUrl = item.imageUrl,  // 注意：LostItem 没有 imageUrl
                    contentDescription = item.name
                )

                Spacer(modifier = Modifier.height(16.dp))
                Text(
                    item.name,
                    style = MaterialTheme.typography.titleLarge,
                    fontWeight = FontWeight.Bold
                )

                Spacer(modifier = Modifier.height(8.dp))
                Text(
                    "地點: ${StringUtils.displayLocation(item.location)}",
                    style = MaterialTheme.typography.bodyMedium
                )

                Spacer(modifier = Modifier.height(4.dp))
                Text(
                    "時間: ${item.time}",
                    style = MaterialTheme.typography.bodySmall,
                    color = Color.Gray
                )
            }
        }
    }
}

// 添加导入
import tw.edu.fju.myapplication.ui.components.ItemImage
```

### Step 4: 更新 LostItemCard (显示缩略图)

**文件**: `/Users/daniellan/Documents/APP/app/src/main/java/tw/edu/fju/myapplication/ui/components/LostItemCard.kt`

```kotlin
@Composable
fun LostItemCard(
    item: LostItem,
    imageUrl: String? = null,  // 新增参数
    onClick: () -> Unit = {}
) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .clickable { onClick() }
    ) {
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .height(120.dp)
                .clickable { onClick() }
        ) {
            // 左侧：图片缩略图
            if (!imageUrl.isNullOrEmpty()) {
                AsyncImage(
                    model = imageUrl,
                    contentDescription = item.name,
                    modifier = Modifier
                        .width(100.dp)
                        .fillMaxHeight()
                        .clip(RoundedCornerShape(8.dp)),
                    contentScale = ContentScale.Crop,
                    loading = {
                        Box(
                            modifier = Modifier
                                .fillMaxSize()
                                .background(Color.LightGray),
                            contentAlignment = Alignment.Center
                        ) {
                            CircularProgressIndicator(
                                modifier = Modifier.size(30.dp),
                                strokeWidth = 2.dp
                            )
                        }
                    }
                )
                Spacer(modifier = Modifier.width(12.dp))
            } else {
                Box(
                    modifier = Modifier
                        .width(100.dp)
                        .fillMaxHeight()
                        .background(Color.LightGray)
                        .clip(RoundedCornerShape(8.dp)),
                    contentAlignment = Alignment.Center
                ) {
                    Text("無圖", color = Color.Gray, fontSize = 12.sp)
                }
                Spacer(modifier = Modifier.width(12.dp))
            }

            // 右侧：物品信息
            Column(
                modifier = Modifier
                    .weight(1f)
                    .padding(vertical = 8.dp)
            ) {
                Text(
                    item.name,
                    style = MaterialTheme.typography.titleMedium,
                    fontWeight = FontWeight.Bold,
                    maxLines = 1
                )
                Spacer(modifier = Modifier.height(4.dp))
                Text(
                    "地點: ${StringUtils.displayLocation(item.location)}",
                    style = MaterialTheme.typography.bodySmall,
                    maxLines = 1
                )
                Spacer(modifier = Modifier.height(4.dp))
                Text(
                    "時間: ${item.time}",
                    style = MaterialTheme.typography.bodySmall,
                    color = Color.Gray,
                    maxLines = 1
                )
            }
        }
    }
}

// 添加导入
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.width
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.ui.unit.sp
import coil.compose.AsyncImage
```

---

## 测试清单

### 分页测试
- [ ] 首页加载第一页 (20 条)
- [ ] 滚动到底部时自动加载下一页
- [ ] 显示 "已加载全部内容" 提示
- [ ] 加载中显示进度指示器

### 分类测试
- [ ] 点击"拾獲"标签显示拾獲物品
- [ ] 点击"遺失"标签显示遺失物品
- [ ] 新增物品时可以选择分类
- [ ] Firebase 中的分类字段正确保存

### 图片测试
- [ ] 详情页面显示图片
- [ ] 列表卡片显示缩略图
- [ ] 无图片时显示占位符
- [ ] 图片加载中显示加载动画
- [ ] 图片加载失败显示错误提示

---

**最后更新**: 2025-11-14  
**版本**: 1.0
