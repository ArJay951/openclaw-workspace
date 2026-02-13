# TASK-005: 假人資料表 + 代收代付自動帶入 + 後台可修改

## 目標
1. 建立 1 萬筆假人資料（台灣中文名 + 手機號）
2. 代收代付訂單建立時隨機挑一筆填入姓名/電話
3. 後台訂單明細可修改姓名和電話

---

## Step 1: 建表 + 塞資料

### 1a. 建表
```sql
CREATE TABLE oms_fake_contact (
  id BIGINT(20) NOT NULL AUTO_INCREMENT,
  name VARCHAR(10) NOT NULL COMMENT '姓名（2~3個中文字）',
  phone VARCHAR(20) NOT NULL COMMENT '手機號碼（09開頭）',
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='假人資料';
```

### 1b. 用程式或 SQL 生成 1 萬筆資料
- **姓氏**：常見台灣姓氏（陳林黃張李王吳劉蔡楊許鄭謝洪郭邱曾廖賴徐周葉蘇莊呂江何蕭羅高潘簡朱鍾彭游詹胡施沈余盧梁趙顏柯翁魏孫馬）
- **名字**：1~2 個字，從常用字池隨機組合（如：志明淑芬雅婷家豪怡君宗翰美玲建宏欣怡俊傑麗華...等）
- **手機**：`09` + 隨機 8 位數字
- 用 Java、Python 或純 SQL 腳本都行，能塞進 DB 就好

---

## Step 2: Model + Mapper (mall-mbg)

### 2a. 新增 `OmsFakeContact.java`
- 欄位: id, name, phone

### 2b. 新增 `OmsFakeContactMapper.java`
- `selectRandom()` → OmsFakeContact（隨機取一筆，用 `ORDER BY RAND() LIMIT 1` 或更高效的方式）
- `selectCount()` → int

---

## Step 3: Portal — 自動帶入

### 修改 `AutoOrderServiceImpl.java`
- `generateCollectOrder()` 和 `generatePayOrder()` 中：
  - 呼叫 `OmsFakeContactMapper.selectRandom()` 取得一筆假人
  - `order.setReceiverName(contact.getName())`
  - `order.setReceiverPhone(contact.getPhone())`
- 目前 receiverName 是用 employeeId，改為用假人姓名
- 退貨申請的 returnName / returnPhone 也要用假人

**⚠️ 效能注意**: `ORDER BY RAND()` 在 1 萬筆不是問題，但如果追求效能可以用 `SELECT * FROM oms_fake_contact WHERE id >= (SELECT FLOOR(RAND() * (SELECT MAX(id) FROM oms_fake_contact))) LIMIT 1` 的方式。

---

## Step 4: Admin — 後台修改姓名電話

### 確認 OmsOrderController 是否已有 update 端點
- 如果有：確保 receiverName, receiverPhone 可以被更新
- 如果沒有：新增 `POST /order/updateReceiver` API
  - 參數: orderId, receiverName, receiverPhone
  - 只更新這兩個欄位

### 退貨單也要能改
- 確認退貨申請是否有 update 端點
- 如果有：確保 returnName, returnPhone 可更新
- 如果沒有：新增對應 API

---

## 驗收標準
1. `mvn clean package -DskipTests` 成功
2. DB 確認 `oms_fake_contact` 有 10000 筆資料，姓名 2~3 字中文，手機 09 開頭
3. curl 代收 API → 訂單的 receiverName 是中文姓名，receiverPhone 是 09 手機
4. curl 代付 API → 同上
5. 後台 update API 可修改姓名電話
6. Docker 容器重啟成功
