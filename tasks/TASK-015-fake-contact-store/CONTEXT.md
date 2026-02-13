# CONTEXT - TASK-015

## 路徑
- 後端: `/home/ubuntu/general-mall/mall/`
- Model: `mall-mbg/src/main/java/com/macro/mall/model/OmsFakeContact.java`
- Mapper: `mall-mbg/src/main/java/com/macro/mall/mapper/OmsFakeContactMapper.java`
- Service: `mall-portal/src/main/java/com/macro/mall/portal/service/impl/AutoOrderServiceImpl.java`
- 門市分配腳本: `/home/ubuntu/7eleven-scraper/assign_stores.py`

## DB
- MySQL: `docker exec -i mall-mysql mysql -u root -proot mall`
- 表: `oms_fake_contact` (10000筆), `store_711` (7316筆)

## 現有 oms_fake_contact 結構
```
id bigint PK AUTO_INCREMENT
name varchar(10)
phone varchar(20)
```

## store_711 結構
```
id, store_id(UNIQUE), store_name, city, district, address, phone, 
longitude, latitude, services, op_day, op_time, create_time
```

## AutoOrderServiceImpl 關鍵位置
- 代收: ~line 119 `fakeContactMapper.selectRandom()` → 設定 order 的 receiver 欄位
- 代付: ~line 207 `fakeContactMapper.selectRandom()` → 設定 returnApply 的 return 欄位

## oms_order receiver 欄位
- receiver_name, receiver_phone, receiver_post_code
- receiver_province (縣市), receiver_city (區域), receiver_region
- receiver_detail_address (詳細地址)

## Build & Deploy
```bash
cd /home/ubuntu/general-mall/mall && mvn clean package -DskipTests -q
cd mall-portal/target && docker build -t mall/mall-portal:1.0-SNAPSHOT -f /tmp/Dockerfile-portal .
cd ../../mall-admin/target && docker build -t mall/mall-admin:1.0-SNAPSHOT -f /tmp/Dockerfile-admin .
cd /home/ubuntu/general-mall && docker compose up -d mall-admin mall-portal
```

## API Token (測試用)
- 本地測試站台: `550711ffb944efb498366a45940f736e`

## Python
- `/home/ubuntu/7eleven-scraper/` 已有 scrape.py, stores.json
- 可直接從 DB 讀 store_711 資料
- pip3 install pymysql（可能需要安裝）
