# TASK-018: 退貨申請列表導出 Excel

## 目標
退貨申請列表加入導出功能，邏輯同訂單列表（TASK-016）。

## 參考
訂單列表導出已完成，程式碼在：
- 後端: `/home/ubuntu/general-mall/mall/mall-admin/src/main/java/com/macro/mall/controller/OmsOrderController.java` 的 `exportExcel` 方法
- 前端: `/home/ubuntu/mall-admin-web/src/views/oms/order/index.vue` 的 `handleExport` 方法

## 步驟

### 1. 後端 - OmsReturnApplyQueryParam 加日期區間

檔案: `/home/ubuntu/general-mall/mall/mall-admin/src/main/java/com/macro/mall/dto/OmsReturnApplyQueryParam.java`

- 移除 `createTime`，新增 `startTime` / `endTime`
- 同理移除 `handleTime`，新增 `handleStartTime` / `handleEndTime`（或保留 handleTime 不改，看現有篩選需求）

實際上只改 createTime → startTime/endTime 即可（跟訂單一致），handleTime 保留單日。

### 2. 後端 - OmsOrderReturnApplyDao.xml 修改查詢

檔案: `/home/ubuntu/general-mall/mall/mall-admin/src/main/resources/dao/OmsOrderReturnApplyDao.xml`

把 createTime LIKE 改為日期區間：
```xml
<if test="queryParam.startTime!=null and queryParam.startTime!=''">
    AND ra.create_time &gt;= concat(#{queryParam.startTime}," 00:00:00")
</if>
<if test="queryParam.endTime!=null and queryParam.endTime!=''">
    AND ra.create_time &lt;= concat(#{queryParam.endTime}," 23:59:59")
</if>
```

### 3. 後端 - OmsOrderReturnApplyController 新增導出 API

檔案: `/home/ubuntu/general-mall/mall/mall-admin/src/main/java/com/macro/mall/controller/OmsOrderReturnApplyController.java`

新增 `exportExcel` 方法，參考 OmsOrderController：
- 驗證至少一個篩選條件
- 日期不超過 31 天
- 站台帳號過濾（applySourceFilter）
- 不分頁查全部

Excel 欄位：
| 欄位 | 來源 |
|------|------|
| 服務單號 | id |
| 訂單編號 | orderSn |
| 申請時間 | createTime |
| 退貨人 | returnName |
| 手機號碼 | returnPhone |
| 退款金額 | returnAmount (fallback: productRealPrice * productCount) |
| 申請狀態 | status (格式化) |
| 退貨原因 | reason |
| 問題描述 | description |
| 商品明細 | 解析 productAttr JSON（如有），格式「商品名 × 數量」；否則用 productName × productCount |

### 4. 後端 - ReturnApplyService 加 listAll 方法

Service interface + impl 加 listAll（不分頁版本）。

### 5. 前端 - 申請時間改日期區間

檔案: `/home/ubuntu/mall-admin-web/src/views/oms/apply/index.vue`

把申請時間 el-date-picker 改為 daterange：
```html
<el-form-item label="申請時間：">
  <el-date-picker
    v-model="listQuery.dateRange"
    value-format="yyyy-MM-dd"
    type="daterange"
    range-separator="至"
    start-placeholder="開始日期"
    end-placeholder="結束日期"
    style="width: 280px">
  </el-date-picker>
</el-form-item>
```

修改 defaultListQuery：移除 createTime，加 dateRange: null

修改 getList()：在 fetchList 前處理 dateRange → startTime/endTime

### 6. 前端 - 新增導出按鈕

在查詢搜索按鈕旁加：
```html
<el-button
  style="float:right;margin-right: 15px"
  type="warning"
  @click="handleExport()"
  size="small">
  導出Excel
</el-button>
```

handleExport 邏輯同訂單列表（驗證條件、31天限制、blob 下載）。

### 7. 前端 - API

檔案: `/home/ubuntu/mall-admin-web/src/api/returnApply.js`

新增：
```js
export function exportExcel(params) {
  return request({
    url: '/returnApply/exportExcel',
    method: 'get',
    params: params,
    responseType: 'blob'
  })
}
```

### 8. Build & Deploy

```bash
# 後端
cd /home/ubuntu/general-mall/mall && mvn clean package -DskipTests -q
cd mall-admin/target && docker build -t mall/mall-admin:1.0-SNAPSHOT -f /tmp/Dockerfile-admin .
cd /home/ubuntu/general-mall && docker compose up -d mall-admin

# 前端
cd /home/ubuntu/mall-admin-web && npm run build

# Git
cd /home/ubuntu/general-mall/mall && git add -A && git commit -m "feat: 退貨申請列表時間區間篩選+導出Excel" && git push
cd /home/ubuntu/mall-admin-web && git add -A && git commit -m "feat: 退貨申請列表時間區間篩選+導出Excel" && git push
```

## 驗收標準
1. 申請時間改為日期區間選擇器
2. 導出按鈕 + 條件驗證（至少一個、日期≤31天）
3. Excel 含所有欄位（特別是商品明細）
4. 站台帳號只能導出自己的退貨申請
5. mvn build + npm build 成功

## ⚠️ 注意
- Dockerfile: `/tmp/Dockerfile-admin`
- 只操作開發環境
- 退貨申請的商品明細：代付(orderId=0)從 productAttr JSON 解析，一般退貨用 productName × productCount
- request.js 已支援 blob（TASK-016 改過）
- 狀態格式化：0待處理、1退貨中、2已完成、3已拒絕
