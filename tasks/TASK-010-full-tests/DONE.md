# TASK-010: DONE

## 結果
✅ 全部完成，兩個商城測試全 PASS

## General Mall
- **Total: 67 tests, 67 PASS, 0 FAIL**
- 新增 7 個測試檔案：test-product-admin.sh, test-product-category.sh, test-brand.sh, test-coupon.sh, test-home-content.sh, test-ums.sh, test-portal.sh
- 擴充 test-order-admin.sh（+3 tests: 訂單詳情、更新備註、退貨原因列表）
- run-tests.sh 改為動態載入（auth + 依賴順序的先載，其餘 test-*.sh 自動掃描）

## Entertainment Mall
- **Total: 41 tests, 41 PASS, 0 FAIL**
- 新增 6 個測試檔案：test-product-admin.sh, test-product-category.sh, test-brand.sh, test-coupon.sh, test-ums.sh, test-portal-full.sh
- 擴充 test-ums.sh（+2: role/listAll, resource/listAll）
- run-tests.sh 改為動態載入

## 注意事項
- Brand delete 在 General Mall 是 GET（非 POST）
- Coupon update 需要完整欄位（含 id、所有必填欄位）
- Order update/note 用 query params（非 JSON body）
- ProductCategory update 需要 id + level + productUnit 等完整欄位
- CRUD 測試資料皆自動清理，無殘留
