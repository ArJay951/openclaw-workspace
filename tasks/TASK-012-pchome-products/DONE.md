# TASK-012: mall.sql 加入商品種類與屬性資料 - 完成

## 完成內容

1. **pms_product_attribute_category** - 新增 7 筆屬性分類（3C數位、家電、食品、生活、居家、保健、美妝）
2. **pms_product_attribute** - 新增 41 筆商品屬性（每個分類 5-7 個屬性，含規格和參數）
3. **pms_product** - 560 筆商品的 `product_attribute_category_id` 從 NULL 更新為對應的屬性分類變數引用

## 驗證結果
- mall_test2 測試資料庫匯入成功
- 屬性分類: 7 ✅
- 屬性: 41 ✅  
- 商品: 560 ✅

## Git
- Commit: `update: mall.sql 加入商品種類與屬性資料`
- Pushed to main branch
