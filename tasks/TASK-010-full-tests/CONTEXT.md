# CONTEXT.md — TASK-010

## General Mall
- Admin API: port 8080
- Portal API: port 8085
- 現有測試: `/home/ubuntu/general-mall/tests/`（5 個 test-*.sh，26 tests）
- Admin 登入: username=admin, password=123456
- API Token (claw): 36ae430b6120992e5ac779cd8713342e

## Entertainment Mall
- Admin API: port 8090
- Portal API: port 8095
- 現有測試: `/home/ubuntu/entertainment-mall/tests/`（6 個 test-*.sh，23 tests）
- Admin 登入: username=admin, password=123456
- Portal 需帶 Host header: `Host: ent.homely-go.com`

## 注意
- 先看現有的 run-tests.sh 結構，確保新增測試能被自動載入
- 如果 run-tests.sh 是硬編碼的，改成動態掃描 test-*.sh
- CRUD 測試要建一個假資料→驗證→刪掉，不留殘留
- 商品 create 需要的欄位比較多，參考現有的 Controller 或 Swagger
- Portal API 有些需要會員 JWT（購物車、訂單），沒登入就只測公開端點
