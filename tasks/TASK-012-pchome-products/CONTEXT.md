# CONTEXT — TASK-012

## 環境
- 伺服器: 52.76.231.27
- MySQL: docker container `mall-mysql`, port 3306, root/root, database `mall`
- Python3 已安裝
- pip3 可用

## DB Schema 重點

### pms_product_category
```sql
id, parent_id, name, level(0=一級,1=二級), product_count, product_unit, nav_status, show_status, sort, icon, keywords, description
```

### pms_brand
```sql
id, name, first_letter, sort, factory_status(1=品牌製造商), show_status(1=顯示), product_count, product_comment_count, logo, big_pic, brand_story
```

### pms_product (重要欄位)
```sql
id, brand_id, product_category_id, name, pic, product_sn, 
delete_status(0), publish_status(1), new_status(1), recommend_status(0), verify_status(1),
sort(0), sale, price, original_price, stock(999), unit('件'),
sub_title, description, album_pics, detail_html
```

### pms_sku_stock
```sql
id, product_id, sku_code, price, stock(999), low_stock(0), pic, sale, sp_data
```

## 圖片存放
- 路徑: `/home/ubuntu/product-images/{category_name}/{product_sn}/`
- 主圖: `main.jpg`
- 相簿: `album_1.jpg`, `album_2.jpg`, ...
- DB 中存相對路徑: `/product-images/{category_name}/{product_sn}/main.jpg`

## PChome API 參考
- 搜尋: `https://ecshweb.pchome.com.tw/search/v4.3/all/results?q=筆電&page=1&sort=sale/dc`
- 商品詳情: `https://ecapi.pchome.com.tw/ecshop/prodapi/v2/prod/{prodId}&fields=Seq,Id,Name,Price,Imgs,Store`
- 圖片 URL 格式: `https://cs-{x}.ecimg.tw/{path}`
- 注意：API 可能有變動，需要先測試確認可用的 endpoint

## 安裝依賴
```bash
pip3 install pymysql requests
```
