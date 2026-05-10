# 行銷通路歸因與 App 推播成效評估
### 大數據與商業分析 × 91APP｜期末專題 Proposal｜第二十組

---

## 專題概述

本專題以 91APP 美妝保健業態資料集為基礎，探討以下兩個核心商業問題：

1. **廣告通路歸因**：不同廣告來源（UTM Source/Medium）對引發購買行為的影響力，並比較現行各種歸因模型（Last-Click、First-Click、線性、時間衰退）的差異。
2. **App 推播成效評估**：會員安裝 App 及開啟推播通知的狀態，是否顯著提升其回購率與消費金額。

---

## 組員分工

| 組員 | 負責內容 |
|------|----------|
| 亭穎 | 背景與動機、預期成果與可能貢獻 |
| 若涵 | EDA + 預計使用的方法與工具 |
| 芝伶 | 書面文字整理、資料欄位說明（本文件） |

---

## 資料集說明

本專題使用 91APP 提供之美妝保健業態資料集，共六張資料表。以下列出**本專題實際使用的資料表與欄位**。

### 1. 會員資料（Member）

| 欄位名稱 | 欄位定義 | 使用目的 |
|----------|----------|----------|
| `ShopMemberId` | 會員中心編碼 | 跨表 JOIN 的主鍵 |
| `RegisterSourceTypeDef` | 會員註冊來源（AndroidApp / iOSApp / Web / Store） | 區分線上 vs 線下會員 |
| `IsAppInstalled` | 是否曾下載 App | App 安裝狀態分析 |
| `IsEnablePushNotification` | 允許推播行銷通知 | 推播允許狀態分析 |
| `IsEnableEmail` | 允許 Email 行銷通知 | 行銷接觸偏好對照 |
| `IsEnableShortMessage` | 允許簡訊行銷通知 | 行銷接觸偏好對照 |
| `FirstAppOpenDateTime` | 首次開啟 App 時間 | App 使用行為時間錨 |
| `LastAppOpenDateTime` | 最後開啟 App 時間 | App 活躍度衡量 |
| `MemberCardLevel` | 會員卡等級（10–90） | 控制變數（會員價值層級） |

---

### 2. 主單資料（Order）

| 欄位名稱 | 欄位定義 | 使用目的 |
|----------|----------|----------|
| `ShopMemberId` | 會員中心編碼 | 與 Member 表 JOIN |
| `TradesGroupCode` | 主單編號 | 與 Behavior 表 JOIN（追蹤購買事件） |
| `OrderDateTime` | 訂單成立時間 | 購買時間點，計算回購間隔 |
| `ChannelType` | 購買通路（OfficialECom / Pos） | 區分線上線下消費 |
| `TotalSalesAmount` | 訂單總金額 | 廣告貢獻金額的目標值 |
| `TotalDiscount` | 折扣總金額 | 控制折扣影響 |
| `StatusDef` | 訂單狀態 | 篩選有效完成訂單（排除 Fail） |
| `PaymentType` | 付款方式 | 輔助分析消費偏好 |

---

### 3. 行為資料（Behavior）

> ⚠️ 資料量：248,257,353 筆 × 28 欄，建議取樣或以 DuckDB / Polars 處理。

**共有欄位（所有行為事件皆有）**

| 欄位名稱 | 欄位定義 | 使用目的 |
|----------|----------|----------|
| `ShopMemberId` | 會員中心編碼 | 與 Member、Order 表 JOIN |
| `FullvisitorId` | 訪客編號 | 識別未登入訪客的 Session |
| `Tunnel` | 管道（Web / App） | 區分 App 與 Web 行為 |
| `Device` | 裝置（iOS / Android / Desktop / MobileWeb） | 裝置層級分析 |
| `UTMSource` | 廣告活動來源（google / FB / SMS 等） | 廣告歸因核心欄位 |
| `UTMMedium` | 廣告活動媒介（cpc / organic / referral 等） | 廣告歸因核心欄位 |
| `UTMName` | 廣告活動名稱 | 廣告活動層級細分 |
| `HitTime` | Session 時間戳記（Unix ms） | Session 切分基準 |

**事件專屬欄位（本專題重點行為）**

| 欄位名稱 | 對應行為 | 使用目的 |
|----------|----------|----------|
| `Behavior` | 八大行為事件名稱 | 轉換漏斗各階段識別 |
| `TradesGroupCode` | Purchase 事件 | 與 Order 表串聯，確認實際購買 |
| `TotalSalesAmount` | Purchase 事件 | 廣告帶來的訂單金額 |
| `SalePageId` | ViewProduct / AddToCart / Checkout / Purchase | 商品層級行為追蹤 |
| `EventTime` | 所有事件 | 事件時間序列排序 |

**Session 切分規則**（依資料說明文件定義）
- 閒置超過 30 分鐘 → 新 Session
- 每天過 23:59:59 → 新 Session
- 換流量來源（UTM 改變）→ 新 Session

---

### 4. 轉換漏斗定義

本專題的廣告歸因分析以下列行為順序定義轉換漏斗：

```
UTM 觸及（Session 開始）
    → ViewProduct（瀏覽商品頁）
        → AddToCart（加入購物車）
            → Checkout（開始結帳）
                → Purchase（完成購買）✓
```

各漏斗階段的流失率將以各 UTM Source 為分組單位分別計算。

---

### 5. 資料表關聯圖

```
Member (ShopMemberId)
    ├── Order (ShopMemberId, TradesGroupCode)
    └── Behavior (ShopMemberId, TradesGroupCode @ Purchase)
```

---

## 資料前處理注意事項

**訂單篩選**：計算成立訂單需排除 `StatusDef = 'Fail'`；計算訂單淨額需同時排除 `Overdue`。

**Behavior 表時間轉換**：`HitTime` 與 `EventTime` 均為 Unix Timestamp（13 碼毫秒），需轉換後加 8 小時調整至台灣時區。
```python
import pandas as pd
df['HitDateTime'] = pd.to_datetime(df['HitTime'], unit='ms') + pd.Timedelta(hours=8)
```

**推播欄位限制**：`IsEnablePushNotification = True` 僅代表會員「允許」推播通知，不代表實際接收到推播內容，分析結論需注意此因果推斷限制。

**App 安裝欄位限制**：`IsAppInstalled = True` 僅代表曾經下載，可能已卸載，需搭配 `LastAppOpenDateTime` 輔助判斷實際活躍狀態。

**UTM 資料完整性**：直接流量（`UTMSource = '(direct)'`）比例通常較高，建議在歸因分析中單獨列為一個管道類別，而非排除。

---

## 專案結構（預計）

```
bda2026-g20-attribution/
├── README.md               # 本文件
├── data/                   # 資料說明（不含原始資料）
│   └── schema.md
├── eda/                    # 探索性資料分析
│   ├── 01_member_profile.ipynb
│   ├── 02_utm_funnel.ipynb
│   └── 03_push_notification.ipynb
├── model/                  # 模型建構
│   └── attribution_model.ipynb
├── report/                 # 期末報告素材
└── proposal/               # Proposal 文件連結
```

---

## 參考資源

- 91APP 資料集說明文件（2024.05，Zola Hsieh）
- [Proposal Google Doc](https://docs.google.com/document/d/1M3a99B8Klwvlzba9LoxkKq314rMbMrTioO3xYkbtlAc/edit)
- G20、G22 過往 Proposal 範例（廣告歸因方向）
