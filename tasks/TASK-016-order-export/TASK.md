# TASK-016: 訂單列表時間篩選+導出

## 目標
1. 提交時間改為日期區間篩選
2. 新增導出按鈕，導出篩選結果為 Excel
3. 導出最後一欄顯示該訂單所有商品×數量

## 步驟

### 1. 後端 - OmsOrderQueryParam 加日期區間

檔案: `/home/ubuntu/general-mall/mall/mall-admin/src/main/java/com/macro/mall/dto/OmsOrderQueryParam.java`

```java
// 移除 createTime，改為：
@ApiModelProperty(value = "開始時間")
private String startTime;
@ApiModelProperty(value = "結束時間")
private String endTime;
```

### 2. 後端 - OmsOrderDao.xml 修改查詢

檔案: `/home/ubuntu/general-mall/mall/mall-admin/src/main/resources/dao/OmsOrderDao.xml`

把原有的：
```xml
<if test="queryParam.createTime!=null and queryParam.createTime!=''">
    AND create_time LIKE concat(#{queryParam.createTime},"%")
</if>
```

改為：
```xml
<if test="queryParam.startTime!=null and queryParam.startTime!=''">
    AND create_time &gt;= concat(#{queryParam.startTime}," 00:00:00")
</if>
<if test="queryParam.endTime!=null and queryParam.endTime!=''">
    AND create_time &lt;= concat(#{queryParam.endTime}," 23:59:59")
</if>
```

### 3. 後端 - 新增導出 API

檔案: `/home/ubuntu/general-mall/mall/mall-admin/src/main/java/com/macro/mall/controller/OmsOrderController.java`

新增方法：
```java
@ApiOperation("導出訂單Excel")
@RequestMapping(value = "/exportExcel", method = RequestMethod.GET)
public void exportExcel(OmsOrderQueryParam queryParam, HttpServletResponse response) {
    // 1. 驗證必須有篩選條件（至少一個非空）
    // 2. 如果有日期區間，驗證不超過31天
    // 3. 查詢所有符合條件的訂單（不分頁）
    // 4. 對每個訂單查詢 order_items
    // 5. 輸出 Excel
}
```

需要加 import:
```java
import javax.servlet.http.HttpServletResponse;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
```

**注意**: 項目已有 poi 依賴（hutool 包含），若沒有則用 hutool 的 ExcelWriter。

先檢查依賴：
```bash
grep -r "poi\|hutool" /home/ubuntu/general-mall/mall/mall-admin/pom.xml
```

如果沒有 POI，使用 hutool-poi（hutool-all 已包含）：
```java
import cn.hutool.poi.excel.ExcelUtil;
import cn.hutool.poi.excel.ExcelWriter;
```

導出邏輯（使用 hutool）：
```java
@ApiOperation("導出訂單Excel")
@RequestMapping(value = "/exportExcel", method = RequestMethod.GET)
public void exportExcel(OmsOrderQueryParam queryParam, HttpServletResponse response) throws Exception {
    // 驗證：至少一個篩選條件
    if (isAllEmpty(queryParam)) {
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("{\"code\":400,\"message\":\"請至少選擇一個篩選條件\"}");
        return;
    }
    // 驗證日期區間不超過 31 天
    if (queryParam.getStartTime() != null && queryParam.getEndTime() != null) {
        // parse and check
        java.time.LocalDate start = java.time.LocalDate.parse(queryParam.getStartTime());
        java.time.LocalDate end = java.time.LocalDate.parse(queryParam.getEndTime());
        if (java.time.temporal.ChronoUnit.DAYS.between(start, end) > 31) {
            response.setContentType("application/json;charset=UTF-8");
            response.getWriter().write("{\"code\":400,\"message\":\"日期範圍不能超過一個月\"}");
            return;
        }
    }
    
    applySourceFilter(queryParam);
    // 查全部（不分頁）
    List<OmsOrder> orderList = orderService.listAll(queryParam);
    
    // 查每個訂單的商品
    ExcelWriter writer = ExcelUtil.getWriter(true);
    writer.addHeaderAlias("orderSn", "訂單編號");
    writer.addHeaderAlias("createTime", "提交時間");
    writer.addHeaderAlias("receiverName", "收貨人");
    writer.addHeaderAlias("receiverPhone", "手機號碼");
    writer.addHeaderAlias("totalAmount", "訂單金額");
    writer.addHeaderAlias("payAmount", "應付金額");
    writer.addHeaderAlias("sourceName", "站台名稱");
    writer.addHeaderAlias("statusText", "訂單狀態");
    writer.addHeaderAlias("receiverDetailAddress", "收貨地址");
    writer.addHeaderAlias("products", "商品明細");
    
    List<Map<String, Object>> rows = new ArrayList<>();
    for (OmsOrder order : orderList) {
        Map<String, Object> row = new LinkedHashMap<>();
        row.put("orderSn", order.getOrderSn());
        row.put("createTime", order.getCreateTime());
        row.put("receiverName", order.getReceiverName());
        row.put("receiverPhone", order.getReceiverPhone());
        row.put("totalAmount", order.getTotalAmount());
        row.put("payAmount", order.getPayAmount());
        row.put("sourceName", order.getSourceName());
        row.put("statusText", formatStatus(order.getStatus()));
        row.put("receiverDetailAddress", order.getReceiverDetailAddress());
        // 查商品
        List<OmsOrderItem> items = orderItemMapper.selectByOrderId(order.getId());
        StringBuilder sb = new StringBuilder();
        for (OmsOrderItem item : items) {
            if (sb.length() > 0) sb.append("\n");
            sb.append(item.getProductName()).append(" × ").append(item.getProductQuantity());
        }
        row.put("products", sb.toString());
        rows.add(row);
    }
    writer.write(rows, true);
    // 自動列寬
    writer.autoSizeColumnAll();
    
    response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;charset=utf-8");
    String fileName = java.net.URLEncoder.encode("訂單匯出_" + java.time.LocalDate.now() + ".xlsx", "UTF-8");
    response.setHeader("Content-Disposition", "attachment;filename=" + fileName);
    writer.flush(response.getOutputStream(), true);
    writer.close();
}

private boolean isAllEmpty(OmsOrderQueryParam q) {
    return (q.getOrderSn() == null || q.getOrderSn().isEmpty())
        && (q.getReceiverKeyword() == null || q.getReceiverKeyword().isEmpty())
        && q.getStatus() == null
        && q.getOrderType() == null
        && q.getSourceType() == null
        && (q.getStartTime() == null || q.getStartTime().isEmpty())
        && (q.getEndTime() == null || q.getEndTime().isEmpty())
        && (q.getSourceName() == null || q.getSourceName().isEmpty());
}

private String formatStatus(Integer status) {
    if (status == null) return "未知";
    switch (status) {
        case 0: return "待付款";
        case 1: return "待發貨";
        case 2: return "已發貨";
        case 3: return "已完成";
        case 4: return "已關閉";
        case 5: return "無效訂單";
        default: return "未知";
    }
}
```

### 4. 後端 - OmsOrderService 加 listAll 方法

檔案: `/home/ubuntu/general-mall/mall/mall-admin/src/main/java/com/macro/mall/service/OmsOrderService.java`
新增: `List<OmsOrder> listAll(OmsOrderQueryParam queryParam);`

實作: `/home/ubuntu/general-mall/mall/mall-admin/src/main/java/com/macro/mall/service/impl/OmsOrderServiceImpl.java`
```java
@Override
public List<OmsOrder> listAll(OmsOrderQueryParam queryParam) {
    // 不用 PageHelper，直接查全部
    return orderDao.getList(queryParam);
}
```

### 5. 後端 - OmsOrderItemMapper 加 selectByOrderId

檢查是否已有，如果沒有，在 Mapper interface 加：
```java
@Select("SELECT * FROM oms_order_item WHERE order_id = #{orderId}")
List<OmsOrderItem> selectByOrderId(@Param("orderId") Long orderId);
```

### 6. 後端 - 確認 hutool-poi 依賴

```bash
grep -r "hutool" /home/ubuntu/general-mall/mall/mall-admin/pom.xml
grep -r "hutool" /home/ubuntu/general-mall/mall/pom.xml
```

如果 mall-admin 沒有 hutool-poi，需要加依賴（或直接用已有的 hutool-all）。
如果完全沒有 hutool-poi，在 mall-admin/pom.xml 加：
```xml
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-poi</artifactId>
    <version>5.8.25</version>
</dependency>
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>5.2.5</version>
</dependency>
```

### 7. 前端 - 修改提交時間篩選器

檔案: `/home/ubuntu/mall-admin-web/src/views/oms/order/index.vue`

把提交時間 el-date-picker 改為日期區間：
```html
<el-form-item label="提交時間：">
  <el-date-picker
    v-model="listQuery.dateRange"
    class="input-width"
    value-format="yyyy-MM-dd"
    type="daterange"
    range-separator="至"
    start-placeholder="開始日期"
    end-placeholder="結束日期"
    style="width: 280px">
  </el-date-picker>
</el-form-item>
```

修改 defaultListQuery：
```js
// 移除 createTime，改為
dateRange: null,
```

修改 getList()，在 fetchList 之前處理日期：
```js
getList() {
  this.listLoading = true;
  let query = Object.assign({}, this.listQuery);
  if (query.dateRange && query.dateRange.length === 2) {
    query.startTime = query.dateRange[0];
    query.endTime = query.dateRange[1];
  }
  delete query.dateRange;
  fetchList(query).then(response => {
    this.listLoading = false;
    this.list = response.data.list;
    this.total = response.data.total;
  });
},
```

### 8. 前端 - 新增導出按鈕

在查詢搜索按鈕旁加導出按鈕：
```html
<el-button
  style="float:right;margin-right: 15px"
  type="warning"
  @click="handleExport()"
  size="small">
  導出Excel
</el-button>
```

導出方法：
```js
handleExport() {
  // 驗證至少一個條件
  let query = Object.assign({}, this.listQuery);
  if (query.dateRange && query.dateRange.length === 2) {
    query.startTime = query.dateRange[0];
    query.endTime = query.dateRange[1];
  }
  delete query.dateRange;
  
  let hasCondition = query.orderSn || query.receiverKeyword || query.status !== null 
    || query.orderType !== null || query.sourceName || query.startTime || query.endTime;
  
  if (!hasCondition) {
    this.$message({
      message: '請至少選擇一個篩選條件',
      type: 'warning'
    });
    return;
  }
  
  // 日期區間驗證（最多31天）
  if (query.startTime && query.endTime) {
    let start = new Date(query.startTime);
    let end = new Date(query.endTime);
    let diff = (end - start) / (1000 * 60 * 60 * 24);
    if (diff > 31) {
      this.$message({
        message: '日期範圍不能超過一個月',
        type: 'warning'
      });
      return;
    }
  }
  
  // 下載
  let params = new URLSearchParams();
  for (let key in query) {
    if (query[key] !== null && query[key] !== undefined && query[key] !== '') {
      params.append(key, query[key]);
    }
  }
  
  // 取得 token
  let token = this.$store.state.user.token || localStorage.getItem('token') || '';
  window.open(process.env.VUE_APP_BASE_API + '/order/exportExcel?' + params.toString() + '&token=' + token);
},
```

**注意**: 下載 Excel 需要認證。由於是 GET 請求用 window.open，token 需要用 URL 參數傳。

**更好的方式**: 用 axios 下載 blob：
```js
import { exportExcel } from '@/api/order'

handleExport() {
  // ...驗證同上...
  
  this.$message({ message: '正在導出...', type: 'info', duration: 2000 });
  
  const config = {
    url: '/order/exportExcel',
    method: 'get',
    params: query,
    responseType: 'blob'
  };
  
  import('@/utils/request').then(module => {
    module.default(config).then(response => {
      const url = window.URL.createObjectURL(new Blob([response]));
      const link = document.createElement('a');
      link.href = url;
      link.setAttribute('download', '訂單匯出_' + new Date().toISOString().slice(0, 10) + '.xlsx');
      document.body.appendChild(link);
      link.click();
      link.remove();
    }).catch(() => {
      this.$message({ message: '導出失敗', type: 'error' });
    });
  });
},
```

但要注意 request.js 的攔截器可能會處理 blob response 有問題。**建議用第一種方式（window.open + token param）比較簡單。**

如果後端攔截器要支援 URL token 參數認證，需要在 Spring Security 配置中放行 `/order/exportExcel` 或者支援 token query param。

**最簡單**: 在前端用 axios 配 responseType: 'blob'，在 api/order.js 加：
```js
export function exportExcel(params) {
  return request({
    url: '/order/exportExcel',
    method: 'get',
    params: params,
    responseType: 'blob'
  })
}
```

前端 handleExport 用這個 function 下載。

**但要注意**: request.js 攔截器可能對 blob response 做 JSON parse 會出錯。
檢查 `/home/ubuntu/mall-admin-web/src/utils/request.js` 的 response interceptor，如果有問題需要加判斷。

### 9. Build & Deploy

```bash
# 後端
cd /home/ubuntu/general-mall/mall && mvn clean package -DskipTests -q
cd mall-admin/target && docker build -t mall/mall-admin:1.0-SNAPSHOT -f /tmp/Dockerfile-admin .
cd /home/ubuntu/general-mall && docker compose up -d mall-admin

# 前端
cd /home/ubuntu/mall-admin-web && npm run build
```

### 10. Git commit

```bash
cd /home/ubuntu/general-mall/mall && git add -A && git commit -m "feat: 訂單列表時間區間篩選+導出Excel（含商品明細）" && git push
cd /home/ubuntu/mall-admin-web && git add -A && git commit -m "feat: 訂單列表時間區間篩選+導出Excel" && git push
```

## 驗收標準
1. 提交時間改為日期區間選擇器（起始～結束）
2. 沒選任何條件點導出 → 提示「請至少選擇一個篩選條件」
3. 日期超過 31 天 → 提示「日期範圍不能超過一個月」
4. 導出 Excel 包含欄位: 訂單編號、提交時間、收貨人、手機號碼、訂單金額、應付金額、站台名稱、訂單狀態、收貨地址、商品明細
5. 商品明細格式: `商品A × 2\n商品B × 1`
6. 站台帳號只能導出自己的訂單

## ⚠️ 注意
- Dockerfile 路徑: `/tmp/Dockerfile-admin`, `/tmp/Dockerfile-portal`
- 只操作開發環境
- request.js 攔截器可能需要對 blob responseType 特殊處理
- hutool-poi 依賴要確認（可能需要額外加 poi-ooxml）
- Excel 中文檔名需要 URLEncode
- `listAll` 方法不要用 PageHelper（避免分頁）
