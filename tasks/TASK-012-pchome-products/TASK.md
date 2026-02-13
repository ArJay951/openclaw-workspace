# TASK-012: PChome 商品資料爬取與匯入

## 目標
從 PChome 24h (https://24h.pchome.com.tw/) 爬取商品資料，下載圖片到本地，清空現有商品後匯入通用商城 DB。

## 分類（7 大類，每類 50-80 筆商品）
1. 3C數位
2. 家電
3. 食品
4. 生活
5. 居家
6. 保健
7. 美妝

## 步驟

### 1. 分析 PChome API
- PChome 有 JSON API，不需要渲染 HTML
- 常見 API 格式：`https://ecshweb.pchome.com.tw/search/v4.3/all/results?q=關鍵字&page=1&sort=sale/dc`
- 或分類 API：`https://ecapi.pchome.com.tw/ecshop/prodapi/v2/store/{storeId}/prod&offset=0&limit=40`
- 先測試找到可用的 API endpoint

### 2. 爬取商品資料
每個商品需要：
- 商品名稱 (name)
- 商品圖片 URL (pic) — 主圖
- 價格 (price)
- 原價 (original_price)
- 副標題/描述 (sub_title)
- 品牌名稱
- 商品相簿圖 (album_pics) — 多張

### 3. 下載圖片到本地
- 存放路徑：`/home/ubuntu/product-images/`
- 結構：`/home/ubuntu/product-images/{category}/{product_sn}/main.jpg`
- 相簿圖：`/home/ubuntu/product-images/{category}/{product_sn}/album_1.jpg` 等
- 圖片 URL 格式先用本地路徑，之後會上傳到 S3

### 4. 清空現有商品相關資料
```sql
-- 順序很重要，先刪關聯表
DELETE FROM pms_sku_stock;
DELETE FROM pms_product_attribute_value;
DELETE FROM pms_member_price;
DELETE FROM pms_product_full_reduction;
DELETE FROM pms_product_ladder;
DELETE FROM pms_product_verify_record;
DELETE FROM pms_product_operate_log;
DELETE FROM pms_comment;
DELETE FROM pms_comment_replay;
DELETE FROM pms_album_pic;
DELETE FROM pms_album;
DELETE FROM pms_product;
DELETE FROM pms_product_category WHERE 1=1;
DELETE FROM pms_brand WHERE 1=1;
DELETE FROM pms_product_attribute WHERE 1=1;
DELETE FROM pms_product_attribute_category WHERE 1=1;
-- 重置 AUTO_INCREMENT
ALTER TABLE pms_product AUTO_INCREMENT = 1;
ALTER TABLE pms_product_category AUTO_INCREMENT = 1;
ALTER TABLE pms_brand AUTO_INCREMENT = 1;
ALTER TABLE pms_sku_stock AUTO_INCREMENT = 1;
```

### 5. 建立分類結構
- 7 個一級分類（parent_id=0, level=0）
- 每個一級分類下建 3-5 個二級分類（level=1），根據 PChome 實際子分類

### 6. 建立品牌
- 從爬取的商品中提取品牌名稱
- 去重後插入 pms_brand

### 7. 匯入商品
每筆商品需插入：
- `pms_product` — 主表
- `pms_sku_stock` — 至少一筆 SKU（price, stock=999, sku_code=product_sn）

欄位對應：
```
pms_product:
- brand_id: 對應品牌
- product_category_id: 二級分類 ID
- name: 商品名稱
- pic: 主圖路徑
- product_sn: 自動生成（P + 6位流水號）
- delete_status: 0
- publish_status: 1
- new_status: 1
- recommend_status: 0
- verify_status: 1
- sort: 0
- sale: 隨機 10-999
- price: PChome 售價
- original_price: PChome 原價（沒有就等於 price）
- stock: 999
- unit: '件'
- weight: 0
- sub_title: 副標題
- description: 簡短描述
- album_pics: 逗號分隔的相簿圖路徑
- detail_html: 簡單 HTML（可放商品圖）
```

## 驗收標準
1. 7 個一級分類 + 對應二級分類
2. 每個一級分類下有 50-80 筆商品（總計 350-560 筆）
3. 每筆商品有主圖 + 至少 1 張相簿圖，圖片已下載到本地
4. 品牌正確關聯
5. SKU stock 正確建立
6. 後台 (dev.homely-go.com/admin/) 能正常瀏覽商品列表和詳情

## 注意事項
- PChome 可能有反爬蟲機制，加 delay 和 User-Agent
- 圖片下載要注意並發控制，不要太快
- 使用 python3 腳本執行
- MySQL 連線：`docker exec -i mall-mysql mysql -u root -proot mall`
- 或用 pymysql 直接連 `127.0.0.1:3306 root/root mall`
