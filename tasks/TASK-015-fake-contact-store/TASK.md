# TASK-015: 假人資料關聯 7-11 門市

## 目標
讓每個假人綁定一個常用 7-11 門市，代收代付建單時自動帶入門市地址資訊。

## 步驟

### 1. 修改 oms_fake_contact 表結構

新增欄位：
```sql
ALTER TABLE oms_fake_contact ADD COLUMN store_id VARCHAR(20) DEFAULT NULL COMMENT '常用7-11門市代碼';
```

### 2. 按人口分布分配門市

寫 Python 腳本 `/home/ubuntu/7eleven-scraper/assign_stores.py`：

台灣各縣市人口比例（大約）：
```
新北市: 17.0%, 台北市: 10.8%, 桃園市: 9.8%, 台中市: 12.0%, 台南市: 8.0%, 高雄市: 11.8%,
基隆市: 1.6%, 新竹市: 1.9%, 新竹縣: 2.5%, 苗栗縣: 2.3%, 彰化縣: 5.3%, 南投縣: 2.1%,
雲林縣: 2.8%, 嘉義市: 1.1%, 嘉義縣: 2.1%, 屏東縣: 3.4%, 宜蘭縣: 1.9%, 花蓮縣: 1.3%,
台東縣: 0.9%, 澎湖縣: 0.5%, 金門縣: 0.6%, 連江縣: 0.1%
```

邏輯：
1. 從 DB 讀 store_711 全部門市（按 city 分組）
2. 按人口比例算每個縣市分配多少假人（10000 人）
3. 每個縣市內，隨機把假人分配到該縣市的門市（平均分散，部分門市可多一些）
4. UPDATE oms_fake_contact SET store_id = ? WHERE id = ?

### 3. 修改 OmsFakeContact Model

檔案: `/home/ubuntu/general-mall/mall/mall-mbg/src/main/java/com/macro/mall/model/OmsFakeContact.java`

新增欄位：
```java
private String storeId;    // 常用門市代碼

// getter/setter
```

### 4. 修改 OmsFakeContactMapper

檔案: `/home/ubuntu/general-mall/mall/mall-mbg/src/main/java/com/macro/mall/mapper/OmsFakeContactMapper.java`

新增查詢（用 JOIN 一次取出假人+門市資訊）：
```java
@Select("SELECT fc.*, s.store_name, s.city as store_city, s.district as store_district, " +
        "s.address as store_address, s.phone as store_phone " +
        "FROM oms_fake_contact fc " +
        "LEFT JOIN store_711 s ON fc.store_id = s.store_id " +
        "WHERE fc.id >= (SELECT FLOOR(1 + RAND() * (SELECT MAX(id) FROM oms_fake_contact))) LIMIT 1")
OmsFakeContact selectRandomWithStore();
```

OmsFakeContact Model 加 transient 欄位：
```java
private String storeName;     // 門市名稱（非DB欄位）
private String storeCity;     // 縣市
private String storeDistrict; // 區域
private String storeAddress;  // 門市地址
private String storePhone;    // 門市電話
// 全部加 getter/setter
```

### 5. 修改 AutoOrderServiceImpl - 代收訂單

檔案: `/home/ubuntu/general-mall/mall/mall-portal/src/main/java/com/macro/mall/portal/service/impl/AutoOrderServiceImpl.java`

找到代收（generateCollectOrder）設定 receiver 的地方（約 line 119-127）：

原本：
```java
OmsFakeContact collectContact = fakeContactMapper.selectRandom();
...
order.setReceiverName(collectContact.getName());
order.setReceiverPhone(collectContact.getPhone());
```

改為：
```java
OmsFakeContact collectContact = fakeContactMapper.selectRandomWithStore();
...
order.setReceiverName(collectContact.getName());
order.setReceiverPhone(collectContact.getPhone());
// 帶入門市地址
if (collectContact.getStoreAddress() != null) {
    order.setReceiverProvince(collectContact.getStoreCity());
    order.setReceiverCity(collectContact.getStoreDistrict());
    order.setReceiverDetailAddress("7-11 " + collectContact.getStoreName() + " (" + collectContact.getStoreAddress() + ")");
}
```

### 6. 修改 AutoOrderServiceImpl - 代付退貨

找到代付（generatePayOrder）設定 return info 的地方（約 line 207-255）：

原本：
```java
OmsFakeContact payContact = fakeContactMapper.selectRandom();
...
returnApply.setReturnName(payContact.getName());
returnApply.setReturnPhone(payContact.getPhone());
```

改為：
```java
OmsFakeContact payContact = fakeContactMapper.selectRandomWithStore();
...
returnApply.setReturnName(payContact.getName());
returnApply.setReturnPhone(payContact.getPhone());
```

注意：退貨申請表目前沒有地址欄位，暫不需要改。但 description 可以加上門市資訊（可選）。

### 7. Build + Deploy（開發環境）

```bash
cd /home/ubuntu/general-mall/mall && mvn clean package -DskipTests -q
cd mall-admin/target && docker build -t mall/mall-admin:1.0-SNAPSHOT -f /tmp/Dockerfile-admin .
cd ../../mall-portal/target && docker build -t mall/mall-portal:1.0-SNAPSHOT -f /tmp/Dockerfile-portal .
cd /home/ubuntu/general-mall && docker compose up -d mall-admin mall-portal
```

### 8. 測試驗證

```bash
# 驗證假人都有門市
docker exec mall-mysql mysql -u root -proot mall -e "
  SELECT COUNT(*) as total, COUNT(store_id) as with_store FROM oms_fake_contact;
  SELECT s.city, COUNT(*) as contacts FROM oms_fake_contact fc JOIN store_711 s ON fc.store_id = s.store_id GROUP BY s.city ORDER BY contacts DESC;
"

# 測試代收 API
curl -s -X POST http://localhost:8085/autoOrder/collect \
  -H "Content-Type: application/json" \
  -H "X-Api-Token: 550711ffb944efb498366a45940f736e" \
  -d '{"employeeId":"test01","deviceId":"D001","amount":300}' | python3 -m json.tool

# 確認訂單有地址
docker exec mall-mysql mysql -u root -proot mall -e "
  SELECT receiver_name, receiver_province, receiver_city, receiver_detail_address 
  FROM oms_order ORDER BY id DESC LIMIT 3;
"
```

## 驗收標準
1. 10000 假人全部有 store_id
2. 各縣市假人數量符合人口比例（±5%）
3. 代收訂單自動帶入 7-11 門市地址
4. 代付退貨正常運作
5. `mvn package` 無錯誤

## ⚠️ 注意
- Dockerfile 路徑: `/tmp/Dockerfile-admin`, `/tmp/Dockerfile-portal`
- 只操作開發環境
- `selectRandomWithStore` 的隨機邏輯跟原本 `selectRandom` 相同
- receiver_province = 縣市, receiver_city = 區域（這是現有欄位設計，雖然名稱有點怪但保持一致）
- Git commit message 用中文
