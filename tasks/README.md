# Tasks — 工單系統

## 結構

```
tasks/
├── TASK-XXX-簡述/
│   ├── TASK.md        # 任務說明（目標、步驟、驗收標準）
│   ├── CONTEXT.md     # 只放 sub 需要的 context（路徑、API、DB）
│   ├── REFERENCES/    # 參考檔案（可選）
│   └── DONE.md        # Sub 完成後寫回報
└── archive/           # 完成的任務歸檔
```

## 流程

1. Main 收斂需求 → 寫 TASK.md + CONTEXT.md
2. Main spawn sub-agent，prompt 指向 task 資料夾
3. Sub 讀檔 → 執行 → 寫 DONE.md → build/test
4. Main 驗收 → 通過則歸檔，不通過則補充 context 再派

## Sub-Agent 標準 Prompt

```
你是一個執行者。請先閱讀以下檔案：
1. /home/ubuntu/.openclaw/workspace/tasks/TASK-XXX/TASK.md
2. /home/ubuntu/.openclaw/workspace/tasks/TASK-XXX/CONTEXT.md

嚴格按照 TASK.md 執行。完成後：
1. 把結果摘要寫入 DONE.md
2. 執行驗收步驟（build / test / deploy）
3. 如遇到無法解決的問題，寫入 DONE.md 說明卡住的原因
```

## 命名規則
- TASK-001, TASK-002 ... 遞增
- 簡述用中文，如 `TASK-001-前端配色調整`
