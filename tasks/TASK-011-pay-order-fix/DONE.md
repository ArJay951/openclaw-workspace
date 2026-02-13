# TASK-011 DONE

## 完成內容
修改 `AutoOrderServiceImpl.generatePayOrder()`，移除建立 `oms_order` 和 `oms_order_item` 的程式碼，代付只建立 `oms_order_return_apply`。

### 具體改動
- **檔案**: `AutoOrderServiceImpl.java`
- **刪除**: 建立 OmsOrder 物件、orderMapper.insert()、orderItem 迴圈插入
- **調整**: `returnApply.setOrderId(0L)` （無對應訂單）
- **保留**: 商品選擇算法、退貨申請建立、假人資料

## 驗收結果
1. ✅ `mvn clean package -DskipTests` — BUILD SUCCESS
2. ✅ Docker 容器重啟成功（docker cp jar + restart）
3. ✅ curl 代付 API → 200，回傳 returnOrderId
4. ✅ DB 確認：`oms_order_return_apply` 新增一筆（order_id=0），`oms_order` 無新增
5. ✅ 代收 API 不受影響
6. ✅ `run-tests.sh` — 67/67 PASS
