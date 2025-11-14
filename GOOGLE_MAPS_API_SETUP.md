# Google Maps API 免費申請指南

## ⚠️ 重要：配額保護機制

此應用已內建**配額監控系統**，當 Google Maps API 配額用盡時，**地圖功能會自動關閉**，不會產生任何費用。

## 📊 配額資訊

```
免費日配額：25,000 次 API 呼叫/天
預期虛擬機演示成本：< $0.01
安全保護：自動限制，不會超支
```

---

## 📋 申請步驟

### **步驟 1️⃣：你的應用信息**

```
套件名稱：tw.edu.fju.myapplication
SHA-1 簽名：75:10:82:18:2A:EC:DD:AD:3C:F2:9A:56:55:38:9A:E9:54:F6:D1:B0
應用類型：Android
```

### **步驟 2️⃣：訪問 Google Cloud Console**

1. 訪問 https://console.cloud.google.com
2. 使用你的 Google 帳號登入

### **步驟 3️⃣：建立新專案**

1. 點擊頂部的「Select a Project」
2. 點擊「NEW PROJECT」
3. 專案名稱：`fju-lost-found`
4. 組織：保持空白
5. 點擊「CREATE」
6. 等待 1-2 分鐘完成

### **步驟 4️⃣：啟用 Maps SDK for Android**

1. 在左側菜單，搜尋「Maps SDK for Android」
2. 點擊搜尋結果中的「Maps SDK for Android」
3. 點擊「ENABLE」（啟用）
4. 等待啟用完成

### **步驟 5️⃣：建立 API Key**

1. 左側菜單點擊「Credentials」（憑證）
2. 點擊「+ CREATE CREDENTIALS」
3. 選擇「API Key」
4. 系統會自動為你建立一個 API Key
5. 複製你的 API Key（形式為：`AIzaSyD...`）

### **步驟 6️⃣：限制 API Key（重要！防止誤用）**

1. 在「Credentials」頁面找到剛才建立的 API Key
2. 點擊 API Key 進入編輯頁面
3. 找到「API restrictions」（API 限制）
4. 選擇「Restrict to specific APIs」（限制為特定 API）
5. 在下拉菜單中選擇「Maps SDK for Android」
6. 點擊「Save」

### **步驟 7️⃣：設定應用限制**

1. 在「Application restrictions」（應用程式限制）部分
2. 選擇「Android apps」
3. 點擊「Add an item」
4. **Package name**：`tw.edu.fju.myapplication`
5. **SHA-1 certificate fingerprint**：`75:10:82:18:2A:EC:DD:AD:3C:F2:9A:56:55:38:9A:E9:54:F6:D1:B0`
6. 點擊「Save」

### **步驟 8️⃣：設定配額上限（可選但推薦）**

1. 左側菜單找到「Quotas」（配額）
2. 找到「Maps SDK for Android」
3. 點擊它，選擇「Requests per day」
4. 設定為 1,000（遠低於免費額度，但足夠演示）
5. 點擊「Change Quotas」

---

## 🔑 複製 API Key

你的 API Key 看起來像這樣：

```
AIzaSyD_4h0k7h9x8c2k3j9x8c2k3j9x8c2k3j9
```

---

## 💻 配置到應用

### **步驟 1：編輯 AndroidManifest.xml**

1. 打開 `/Users/daniellan/Documents/APP/app/src/main/AndroidManifest.xml`
2. 找到這一行：
   ```xml
   android:value="YOUR_GOOGLE_MAPS_API_KEY_HERE"
   ```
3. 將 `YOUR_GOOGLE_MAPS_API_KEY_HERE` 替換為你的實際 API Key
4. 保存文件

### **步驟 2：重新構建應用**

```bash
./gradlew clean build
```

### **步驟 3：運行應用**

在 Android Studio 中點擊「Run」或按 Shift + F10

---

## 📱 驗證地圖是否正常工作

1. 打開應用
2. 點擊「地圖搜尋」按鈕
3. 應該看到輔仁大學校園地圖
4. 可以縮放和拖拽地圖
5. 點擊建築物標記查看該區域的物品

---

## ✅ 配額監控

應用會自動：

1. ✅ 記錄每次 API 呼叫
2. ✅ 每天午夜重置計數器
3. ✅ 達到 80% 時在日誌中警告
4. ✅ 達到 100% 時自動關閉地圖功能
5. ✅ 永遠不會產生任何費用

### **查看配額狀態**

配額狀態會在應用日誌中顯示：

```
D/QuotaManager: API Call Count: 1234/25000 (4%)
W/QuotaManager: 警告：Google Maps API 配額已達 80%
E/QuotaManager: 錯誤：Google Maps API 配額已用盡！
```

---

## 🛡️ 安全保護機制

```
安全層級 1：設定 API Key 限制
├─ 只限用於 Maps SDK for Android
└─ 只限特定應用

安全層級 2：設定配額上限
├─ 日配額限制為 1,000 次
└─ 防止意外超支

安全層級 3：應用內監控
├─ 追蹤每日呼叫次數
├─ 達到配額時自動關閉功能
└─ 不會自動扣款
```

---

## 常見問題

### **Q: 配額用完會怎樣？**
A: 地圖功能會自動關閉，但應用會繼續正常運行。不會產生任何費用。

### **Q: 虛擬機演示會用掉多少配額？**
A: 1-2 小時的演示大約使用 50-200 次呼叫，遠遠在免費配額內。

### **Q: 如何重置配額計數？**
A: 每天午夜自動重置。如需手動重置，運行：
```
./gradlew run_quota_reset
```

### **Q: 怎樣知道配額快用完了？**
A: 應用會在達到 80% 時在日誌中警告，達到 100% 時自動關閉地圖。

### **Q: 可以升級為付費方案嗎？**
A: 可以，但無需升級。免費配額已足夠學校報告使用。

---

## 📞 需要幫助？

如果在申請過程中遇到問題：

1. 確保你的 Google 帳號有效
2. 確保 SHA-1 簽名完全匹配
3. 確保套件名稱完全匹配
4. 嘗試清除瀏覽器快取並重新申請

---

**完成上述步驟後，你就可以在應用中使用真實的 Google 地圖了！🎉**
