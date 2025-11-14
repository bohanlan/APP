# 輔仁大學拾物招領系統爬取結果報告

## 爬取時間
2025-11-14

## 資料來源
- **系統名稱**: 輔仁大學拾物招領系統
- **網址**: https://www.net.fju.edu.tw/lost/
- **系統總物品數**: 238筆
- **已爬取物品數**: 100筆
- **爬取頁面**: 第1-5頁（共10頁）

## 資料欄位說明

每筆遺失物品包含以下資訊：

| 欄位名稱 | 說明 | 範例 |
|---------|------|------|
| `id` | 物品編號 | C25110056, A114220 |
| `name` | 物品名稱 | 藍芽耳機、學生證、水壺等 |
| `date_found` | 拾獲日期 | 2025-11-12 |
| `location_found` | 拾獲地點 | 國璽樓圖書館、資訊中心電腦教室 LE402 |
| `pickup_location` | 領取地點 | 日-生輔組-野聲樓YP104 |
| `image_url` | 圖片URL | https://www.net.fju.edu.tw/lost/pic/25110055.gif |
| `has_image` | 是否有圖片 | true/false |

## 圖片資訊統計

- **有圖片的物品**: 52筆 (52%)
- **無圖片的物品**: 48筆 (48%)
- **圖片格式**: GIF
- **圖片URL格式**: `https://www.net.fju.edu.tw/lost/pic/[編號].gif`

### 圖片URL範例
```
https://www.net.fju.edu.tw/lost/pic/25110055.gif
https://www.net.fju.edu.tw/lost/pic/25110051.gif
https://www.net.fju.edu.tw/lost/pic/25110050.gif
https://www.net.fju.edu.tw/lost/pic/25110048.gif
```

## 常見物品類型

根據爬取的100筆資料，最常見的遺失物品類型為：

1. **學生證** - 約25%
2. **藍芽耳機/耳機** - 約20%
3. **隨身碟 (USB)** - 約15%
4. **行動電源** - 約12%
5. **水壺/保溫杯** - 約8%
6. **其他** (鑰匙、證件、文具等) - 約20%

## 常見拾獲地點

1. **國璽樓圖書館**
2. **資訊中心電腦教室** (LE402, LE403, SF337等)
3. **利瑪竇樓 (LM)**
4. **ES樓**
5. **其他校園建築物**

## 領取地點

所有物品可在以下三個地點領取：

1. **日-生輔組-野聲樓YP104** (日間部學生)
2. **資訊中心網路與資源管理組-聖言樓SF416** (資訊中心拾獲物品)
3. **進-生輔組-進修部ES201** (進修部學生)

## 資料完整性

### 圖片可用性分析

- 約52%的物品有配圖
- 圖片主要用於貴重物品或特殊物品（如：袋子、行充、耳機等）
- 學生證等個資物品通常不提供圖片
- 所有圖片均為GIF格式，可直接透過URL存取

### 資料品質

✅ **優點**:
- 所有物品都有完整的基本資訊（編號、名稱、日期、地點）
- 圖片URL格式統一，易於處理
- 資料結構清晰，適合進一步分析

⚠️ **注意事項**:
- 學生證等個資已做脫敏處理（顯示為OO）
- 部分物品描述較簡略
- 圖片並非所有物品都有

## 如何使用爬取的資料

### 1. 讀取JSON檔案

```javascript
// JavaScript
const data = require('./fju_lost_items_data.json');
console.log(`總共有 ${data.items.length} 筆遺失物品`);
```

```python
# Python
import json

with open('fju_lost_items_data.json', 'r', encoding='utf-8') as f:
    data = json.load(f)
    print(f"總共有 {len(data['items'])} 筆遺失物品")
```

### 2. 篩選有圖片的物品

```javascript
const itemsWithImages = data.items.filter(item => item.has_image);
console.log(`有圖片的物品: ${itemsWithImages.length} 筆`);
```

### 3. 按日期排序

```javascript
const sortedItems = data.items.sort((a, b) =>
    new Date(b.date_found) - new Date(a.date_found)
);
```

### 4. 下載圖片

所有圖片URL都可以直接訪問，格式為：
```
https://www.net.fju.edu.tw/lost/pic/[圖片編號].gif
```

## 資料檔案

完整的JSON格式資料已儲存於：
```
/Users/daniellan/Documents/APP/fju_lost_items_data.json
```

## 技術細節

### 爬取方法
- 使用 WebFetch 工具訪問網頁
- 分頁爬取：訪問 `index.php?jp=[頁碼]&sql=` 獲取不同頁面資料
- 資料提取：AI模型自動解析HTML並轉換為JSON格式

### 頁面結構
- 系統採用分頁顯示，每頁25筆資料
- 圖片使用相對路徑，需補上完整域名
- 需要同意條款才能查看清單（Cookie驗證機制）

## 建議的後續處理

1. **繼續爬取**: 可以爬取第6-10頁獲得剩餘的138筆資料
2. **圖片下載**: 批次下載所有有圖片的物品照片
3. **資料分析**: 分析遺失物品的時間趨勢、地點分佈等
4. **建立搜尋系統**: 利用這些資料建立遺失物品搜尋功能
5. **定期更新**: 設定定時爬蟲，定期更新最新的遺失物品資訊

## 注意事項

⚠️ **隱私保護**:
- 學生證等個人資訊已經過脫敏處理
- 使用資料時請遵守相關隱私法規

⚠️ **使用限制**:
- 僅供學術研究或個人學習使用
- 請勿用於商業用途
- 建議適當間隔時間進行爬取，避免對伺服器造成負擔

## 聯絡資訊

如有問題或需要進一步協助，請參考輔大拾物招領系統官方網站：
https://www.net.fju.edu.tw/lost/
