# TASK-019: 修復+補充 API 測試

## 目標
修復壞掉的測試 + 補上今天新功能的測試覆蓋，最終全部 PASS。

## 測試路徑
- `/home/ubuntu/general-mall/tests/run-tests.sh`
- 各 test-*.sh 在同目錄

## 現況
- 原有 67 tests，現在壞了（來源管理 create API 參數變了）
- 今天新功能無測試覆蓋

## 修復項目

### 1. test-source-mgmt.sh — 修復站台建立參數
現在 `/orderSource/create` 需要 `adminUsername` 和 `adminPassword`：
```json
{"name":"test_auto_api","status":1,"adminUsername":"test_auto_api","adminPassword":"Test123456"}
```
回傳格式: `{"code":200,"data":{"source":{...},"username":"...","rawPassword":"..."}}`

注意：站台名稱有 UNIQUE 約束，測試前先確認沒有殘留資料。建議先嘗試刪除同名站台。

### 2. test-auto-order.sh — 補充驗證

在現有測試後加：

#### 代收訂單驗證門市地址
```bash
# 驗證 receiverProvince 非空（門市所在縣市）
RESP=$(curl -s "$ADMIN_API/order/list?pageNum=1&pageSize=1&orderSn=$TEST_ORDER_SN" -H "$AUTH")
PROVINCE=$(echo "$RESP" | jq -r '.data.list[0].receiverProvince')
DETAIL=$(echo "$RESP" | jq -r '.data.list[0].receiverDetailAddress')
HAS_711=$(echo "$DETAIL" | grep -c "7-11" || true)

if [ -n "$PROVINCE" ] && [ "$PROVINCE" != "null" ] && [ "$HAS_711" -gt 0 ]; then
  pass "代收訂單帶入門市地址 → $PROVINCE, 含7-11"
else
  fail "代收訂單門市地址" "province=$PROVINCE, detail=$DETAIL"
fi
```

#### 代付驗證單筆退貨+productAttr
```bash
# 查代付退貨記錄
RESP=$(curl -s "$ADMIN_API/returnApply/list?pageNum=1&pageSize=5" -H "$AUTH")
# 找到剛建的退貨（orderSn = TEST_RETURN_ORDER_SN）
RETURN_APPLY=$(echo "$RESP" | jq --arg sn "$TEST_RETURN_ORDER_SN" '.data.list[] | select(.orderSn == $sn)')
RETURN_COUNT=$(echo "$RESP" | jq --arg sn "$TEST_RETURN_ORDER_SN" '[.data.list[] | select(.orderSn == $sn)] | length')

if [ "$RETURN_COUNT" = "1" ]; then
  pass "代付只建一筆退貨申請 (orderSn=$TEST_RETURN_ORDER_SN)"
else
  fail "代付退貨數量" "expected 1, got $RETURN_COUNT"
fi

# 驗 returnAmount = 請求金額 (200)
RETURN_AMOUNT=$(echo "$RETURN_APPLY" | jq -r '.returnAmount')
if [ "$RETURN_AMOUNT" = "200.00" ] || [ "$RETURN_AMOUNT" = "200" ]; then
  pass "代付 returnAmount = 200"
else
  fail "代付 returnAmount" "expected 200, got $RETURN_AMOUNT"
fi
```

### 3. 新增 test-export.sh — 導出 Excel 測試

```bash
#!/bin/bash
# test-export.sh — 導出 Excel

echo ""
echo "📋 test-export: 導出 Excel"

AUTH="Authorization: Bearer $ADMIN_TOKEN"
TODAY=$(date +%Y-%m-%d)

# 1. 訂單導出 — 無條件 → 應回 400
RESP=$(curl -s "$ADMIN_API/order/exportExcel" -H "$AUTH")
# blob 回傳如果是 json 就是錯誤
HAS_ERROR=$(echo "$RESP" | jq -r '.code' 2>/dev/null || echo "")
if [ "$HAS_ERROR" = "400" ]; then
  pass "訂單導出無條件 → 400 拒絕"
else
  fail "訂單導出無條件" "expected 400, got response"
fi

# 2. 訂單導出 — 有條件 → 應回 xlsx (Content-Type 檢查)
HTTP_CODE=$(curl -s -o /tmp/test_order_export.xlsx -w "%{http_code}" \
  "$ADMIN_API/order/exportExcel?startTime=$TODAY&endTime=$TODAY" -H "$AUTH")
FILE_SIZE=$(wc -c < /tmp/test_order_export.xlsx)

if [ "$HTTP_CODE" = "200" ] && [ "$FILE_SIZE" -gt 100 ]; then
  pass "訂單導出有條件 → 200, 檔案 ${FILE_SIZE} bytes"
else
  fail "訂單導出有條件" "http=$HTTP_CODE, size=$FILE_SIZE"
fi

# 3. 退貨申請導出 — 無條件 → 400
RESP=$(curl -s "$ADMIN_API/returnApply/exportExcel" -H "$AUTH")
HAS_ERROR=$(echo "$RESP" | jq -r '.code' 2>/dev/null || echo "")
if [ "$HAS_ERROR" = "400" ]; then
  pass "退貨導出無條件 → 400 拒絕"
else
  fail "退貨導出無條件" "expected 400"
fi

# 4. 退貨申請導出 — 有條件 → xlsx
HTTP_CODE=$(curl -s -o /tmp/test_return_export.xlsx -w "%{http_code}" \
  "$ADMIN_API/returnApply/exportExcel?startTime=$TODAY&endTime=$TODAY" -H "$AUTH")
FILE_SIZE=$(wc -c < /tmp/test_return_export.xlsx)

if [ "$HTTP_CODE" = "200" ] && [ "$FILE_SIZE" -gt 100 ]; then
  pass "退貨導出有條件 → 200, 檔案 ${FILE_SIZE} bytes"
else
  fail "退貨導出有條件" "http=$HTTP_CODE, size=$FILE_SIZE"
fi

# 5. 日期超過31天 → 400
RESP=$(curl -s "$ADMIN_API/order/exportExcel?startTime=2026-01-01&endTime=2026-03-01" -H "$AUTH")
HAS_ERROR=$(echo "$RESP" | jq -r '.code' 2>/dev/null || echo "")
if [ "$HAS_ERROR" = "400" ]; then
  pass "訂單導出超過31天 → 400 拒絕"
else
  fail "訂單導出31天限制" "expected 400"
fi

# 清理
rm -f /tmp/test_order_export.xlsx /tmp/test_return_export.xlsx
```

### 4. 新增 test-store711.sh — 7-11 門市資料驗證

```bash
#!/bin/bash
# test-store711.sh — 7-11 門市資料

echo ""
echo "📋 test-store711: 7-11 門市資料"

# 用 mysql 直接查
TOTAL=$(docker exec mall-mysql mysql -u root -proot mall -sN -e "SELECT COUNT(*) FROM store_711" 2>/dev/null)
CITIES=$(docker exec mall-mysql mysql -u root -proot mall -sN -e "SELECT COUNT(DISTINCT city) FROM store_711" 2>/dev/null)

if [ "$TOTAL" -gt 6000 ] 2>/dev/null; then
  pass "7-11 門市總數 = $TOTAL (>6000)"
else
  fail "7-11 門市總數" "got $TOTAL"
fi

if [ "$CITIES" -ge 22 ] 2>/dev/null; then
  pass "7-11 涵蓋 $CITIES 個縣市 (>=22)"
else
  fail "7-11 縣市覆蓋" "got $CITIES"
fi

# 假人門市關聯
WITH_STORE=$(docker exec mall-mysql mysql -u root -proot mall -sN -e "SELECT COUNT(*) FROM oms_fake_contact WHERE store_id IS NOT NULL" 2>/dev/null)
TOTAL_CONTACTS=$(docker exec mall-mysql mysql -u root -proot mall -sN -e "SELECT COUNT(*) FROM oms_fake_contact" 2>/dev/null)

if [ "$WITH_STORE" = "$TOTAL_CONTACTS" ] && [ "$TOTAL_CONTACTS" -gt 0 ]; then
  pass "全部假人都有綁定門市 ($WITH_STORE/$TOTAL_CONTACTS)"
else
  fail "假人門市關聯" "with_store=$WITH_STORE, total=$TOTAL_CONTACTS"
fi
```

## 執行驗證
```bash
cd /home/ubuntu/general-mall/tests && bash run-tests.sh
```

## 驗收標準
1. 全部測試 PASS（0 FAIL）
2. 測試數量 >= 75（原 67 + 新增 ~10）
3. 覆蓋今天的功能：站台建立、代收門市地址、代付單筆退貨、導出 Excel、7-11 資料

## ⚠️ 注意
- 不要改後端或前端程式碼，只改測試腳本
- test-source-mgmt.sh 建立的測試站台名必須唯一，用 `test_auto_$(date +%s)` 避免衝突也行
- API Token 用 claw 站台: `36ae430b6120992e5ac779cd8713342e`，device: `dev001`
- 退貨申請列表查詢現在用 startTime/endTime 不用 createTime
- 導出 API 走 GET，直接 curl 下載
