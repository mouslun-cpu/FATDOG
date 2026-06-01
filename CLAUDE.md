# Fat Dog Rush — 專案說明

## Zeabur 部署資訊
- Project ID: `6a1c64da4853e1f02a136300`
- Service ID: `6a1c64e37a8a8f2f6021511b`
- Environment ID: `6a1c64dab764eebf4f53b580`
- **學生端網址**: https://fatdog-rush.zeabur.app/index.html
- **老師端網址**: https://fatdog-rush.zeabur.app/admin.html

## GitHub
- Repo: https://github.com/mouslun-cpu/FATDOG.git
- Branch: main

## Firebase
- Project: fatdog-adf25
- DB URL: https://fatdog-adf25-default-rtdb.asia-southeast1.firebasedatabase.app

## 使用說明
1. 老師開啟 admin.html → 等候學生掃 QR Code 加入
2. 學生掃 QR → 輸入姓名 → 等候畫面（自動分流 A/B）
3. 老師按「🚀 開始搶購！」→ 全班同時搶 10 個名額
4. 搶完 → SOLD OUT → 老師點「🎊 揭曉真相！」→ A vs B 比分板
5. 老師「🔄 重置遊戲」→ 回到步驟 1

---

## 技術架構

### 檔案結構
```
Fatdog/
├── index.html      ← 學生手機端（A/B 分流）
├── admin.html      ← 老師大螢幕戰情室
├── fatdog.png      ← 主背景圖（Admin 使用）
├── image/fatdog.png← 備份（同一張）
├── Firebase.txt    ← Firebase SDK 設定參考
└── .claude/
    └── launch.json ← 本地 dev server (npx serve, port 5500)
```

### 本地開發
```bash
# 啟動本地 server（在 .claude/launch.json 已設定）
npx serve -p 5500 .
# 學生端: http://localhost:5500/index.html
# 老師端: http://localhost:5500/admin.html
```

### 部署流程（每次更新）
```bash
cd "C:/Users/weilu/Desktop/SideProject/Fatdog"
git add index.html admin.html fatdog.png
git commit -m "說明更新內容"
git push
# 然後在 Zeabur 觸發 redeploy：
npx zeabur@latest deploy --project-id 6a1c64da4853e1f02a136300 --service-id 6a1c64e37a8a8f2f6021511b --json
```

---

## 遊戲機制設計

### Firebase Schema
- `/gameStatus`: `'waiting'` | `'racing'` | `'finished'`
- `/inventory/total`: 數字（預設 10）
- `/groups/{groupId}`: `{ variant, status, displayId, name, ts }`
- `/transactions/{id}`: `{ groupId, variant, displayId, name, ts }`
- `/results`: `{ A_count, B_count }`

### 關鍵技術問題（已修復）
- **Firebase transaction current=null bug**：在 `init()` 加 `db.ref('/inventory/total').on('value', () => {})` 持久暖快取，transaction 才能讀到真實值
- **重置後仍顯示售完**：重置改設 `gameStatus = 'waiting'`（不是 'racing'），讓學生端 listener 正確重新渲染

### A/B 分流邏輯
- 學生第一次進入 → 隨機分配 A 或 B（50/50）
- `variant` 存入 localStorage (`fg_groupId`) + Firebase `/groups`
- 重置後 localStorage groupId 仍在，但 Firebase group 被刪 → 重新隨機分配

---

## A/B 介面差異（當前版本）

### 目標課程
**健康科技學院 → 健康事業管理系 → 創意品管課**（在第 46 位，共 49 選項）

### A 組（高阻力）
| 欄位 | 設計 |
|------|------|
| 識別碼 | 小輸入框，**禁止複製貼上** |
| 課程 | 44 門課 + 4 學院標題（共 49 選項），全平鋪無縮排，目標藏在 **第 46 位** |
| 學期 | 10 個學期（111-1 到 115-2），正確答案 **114-2**（第 8 位）|
| 申請理由 | 手動輸入，**至少 10 字** |
| 送出按鈕 | **52×22px**，極小 |

### B 組（低阻力）
| 欄位 | 設計 |
|------|------|
| 識別碼 | 大舒適輸入框（可貼上）+ 下方 3 個快捷鍵（**FATDOG666** 正確答案在第一個、FATCAT999、HOTDOG888）|
| 課程 | 4 學院色塊 → 點選後顯示 12 張課程卡片 |
| 申請理由 | 3 個罐頭文字按鈕（📌學業目標 / 💼職涯規劃 / 🙏神狗保佑），**一鍵帶入** |
| 送出按鈕 | **橫跨全畫面大紅按鈕** |

### 識別碼
全班共用：**`FATDOG666`**

### 學院資料（基於國立臺北護理健康大學）
```javascript
const DEPT_COURSES = {
  '護理學院': [...12 門課],
  '健康科技學院': [...12 門課，含「創意品管課」在第 10 位],
  '人康學院': [...10 門課],  // B 版簡稱；A 版顯示「人類發展與健康學院」
  '通識教育中心': [...10 門課],
};
// A 版課程選單順序：護理學院 → 人類發展與健康學院 → 通識教育中心 → 健康科技學院（目標在最底部）
```

---

## Admin 戰情室功能

### 三種狀態
1. **等候模式**（waiting）：大 QR Code + 即時人數 + 🚀 開始按鈕
2. **搶購模式**（racing）：fatdog.png 全屏 + 霓虹閃爍剩餘名額 + 跑馬燈
3. **完售模式**（finished）：SOLD OUT → 揭曉真相 → A vs B 比分板

### 按鈕功能
- **🚀 開始搶購**：gameStatus → racing
- **🎊 揭曉真相**：顯示 A vs B 結果比分板
- **🏁 手動結束遊戲**：強制 gameStatus → finished（庫存設 0）
- **🔄 重置遊戲**：inventory=10, gameStatus=waiting, 清除 groups/transactions/results
- **QR Code**：點擊可放大全屏顯示

### 跑馬燈
顯示格式：`🎉 [姓名] 搶購成功！`（不顯示 A/B 組別，維持盲測）
