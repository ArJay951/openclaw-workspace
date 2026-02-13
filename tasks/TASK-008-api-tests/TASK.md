# TASK-008: General Mall API 自動測試腳本

## 目標
建立一組 shell 測試腳本，覆蓋 General Mall 所有自定義 API，部署後一鍵執行驗證。

## 產出
- `/home/ubuntu/general-mall/tests/run-tests.sh` — 主執行腳本
- `/home/ubuntu/general-mall/tests/test-*.sh` — 各模組測試
- 輸出格式：每個測試 PASS/FAIL，最後統計

---

## 測試覆蓋範圍

### 1. test-auth.sh（後台登入）
- ✅ 正確帳密登入 → 200 + token
- ❌ 錯誤密碼 → 非 200
- 存下 admin JWT token 供後續測試用

### 2. test-auto-order.sh（代收代付 API）
**前置**: 需要有效的 API token + 員工設備資料

認證測試：
- ❌ 無 X-Api-Token header → 非 200
- ❌ 錯誤 token → 403
- ❌ 正確 token + 不屬於該來源的 employeeId → 403
- ❌ 正確 token + 正確 employeeId + 不屬於的 deviceId → 403

代收測試：
- ✅ 正確參數 → 200，回傳 orderId
- ✅ 驗證回傳的 orderId 格式正確
- ❌ amount = 0 → 失敗
- ❌ 缺少 employeeId → 400

代付測試：
- ✅ 正確參數 → 200，回傳 returnOrderId（R 開頭）
- ❌ amount = 0 → 失敗

訂單驗證：
- ✅ 用 admin JWT 查詢剛建立的訂單，確認 receiverName 是中文、receiverPhone 是 09 開頭

### 3. test-source-mgmt.sh（來源管理 — Admin API）
需要 admin JWT token

- ✅ GET /orderSource/list → 200，有資料
- ✅ POST /orderSource/create → 200，回傳含 apiToken
- ✅ GET /orderSource/{id} → 200
- ✅ POST /orderSource/regenerateToken/{id} → 200，新 token 不同於舊的
- ✅ POST /orderSource/updateStatus → 200
- ✅ POST /orderSource/delete/{id} → 200
- 清理：刪除測試建立的來源

### 4. test-device-mgmt.sh（設備管理 — Admin API）
需要 admin JWT token + 一個來源 ID

- ✅ POST /orderSource/{sourceId}/devices/create → 200
- ✅ GET /orderSource/{sourceId}/devices → 200，包含剛建立的
- ✅ POST /orderSource/{sourceId}/devices/update/{id} → 200
- ✅ POST /orderSource/{sourceId}/devices/delete/{id} → 200

### 5. test-order-admin.sh（後台訂單管理）
- ✅ 訂單列表 API 可存取
- ✅ 修改 receiverInfo API → 200
- ✅ 退貨申請列表可存取

---

## 腳本規範

### run-tests.sh
```bash
#!/bin/bash
# 用法: ./run-tests.sh [base_url]
# 預設: http://127.0.0.1

BASE_URL=${1:-"http://127.0.0.1"}
ADMIN_API="$BASE_URL:8080"
PORTAL_API="$BASE_URL:8085"

PASS=0
FAIL=0
TOTAL=0

# 輸出函數
pass() { PASS=$((PASS+1)); TOTAL=$((TOTAL+1)); echo "  ✅ PASS: $1"; }
fail() { FAIL=$((FAIL+1)); TOTAL=$((TOTAL+1)); echo "  ❌ FAIL: $1 — $2"; }

# 執行各模組...

# 最後統計
echo "================================"
echo "Total: $TOTAL | Pass: $PASS | Fail: $FAIL"
if [ $FAIL -gt 0 ]; then exit 1; fi
```

### 每個 test 檔
- 可獨立執行，也可被 run-tests.sh source
- 用 curl + jq 解析回傳
- 建立的測試資料在測試結束後清理
- 測試用的 API token/員工/設備用現有的測試資料（claw / emp001 / dev001）

### 安裝依賴
- 確認 `jq` 已安裝，沒有就 `apt install jq`

---

## 驗收標準
1. `./run-tests.sh` 一鍵執行，全部 PASS
2. 覆蓋至少 20 個測試 case
3. 有清楚的 PASS/FAIL 輸出 + 最終統計
4. 測試資料自動清理（不留垃圾）
5. 腳本有註解，易讀
