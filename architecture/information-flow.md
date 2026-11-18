# 資訊流與時序

快照：2026-11-18。圖中為元件角色，不含運行網址或帳號資料。

## 收件與動作判定

```mermaid
sequenceDiagram
    participant L as LINE
    participant E as Cloud Run edge
    participant Q as Firestore／Cloud Tasks
    participant W as 私有 worker
    participant A as GAS
    L->>E: 個人文字或群組真正提及 Bot
    E->>E: 驗原始簽章
    E->>Q: 持久化合法事件及收件順序
    Q->>W: 喚醒處理
    W->>W: 驗入口、識別使用者與聊天室
    W->>A: 具 HMAC 的業務判定請求
    A-->>W: 加入／提問／離開／狀態
    W->>Q: 更新席位及等待狀態
```

此為主要路徑摘要，不表示整條流程是一個分散式交易；失敗、重試與去重各有證據界線。未通過入口條件的事件不應成為 NotebookLM 提問。

## 席位內提問、回覆及交接

```mermaid
sequenceDiagram
    participant U as LINE 使用者
    participant W as worker
    participant F as Firestore
    participant N as NotebookLM
    participant S as Scheduler
    U->>W: 排隊
    W->>F: 入隊／分配
    alt 席位有人使用
        W-->>U: 等待通知
    else 可分配
        W-->>U: 輪到通知
        U->>W: 重新提問
        W->>F: 記錄當輪及工作
        W->>N: 核對歷史後提問
        N-->>W: 答案與可取得引用
        W->>F: 保存工作／通知狀態
        W-->>U: 透過 LINE push 回原聊天室
    end
    S->>W: 定期檢查到期
    U->>W: 或主動結束
    W->>F: 停止接受本輪新題
    alt 上游結果不明或清除失敗
        W->>F: PAUSED，不分配下一人
    else 已知工作完成且可清除
        W->>N: 清除可識別的當輪測試聊天
        N-->>W: 核對可見歷史為空
        W->>F: IDLE，分配下一位
    end
```

圖中的到期與主動結束是替代觸發，不要求兩者都發生。未使用的邀請在歷史本來為空時可直接釋放。等待時的原問題沒有保存供日後自動重問。

## 身分與回覆目的地

| 情境 | 席位識別 | 回覆位置 |
|---|---|---|
| 個人 | 頻道＋使用者 | 原個人聊天室 |
| 群組 | 頻道＋群組＋使用者 | 原群組；內容公開 |
| 同人跨入口 | 視為不同情境 | 各自原入口，不把個人答案搬到群組 |

資料流成功只能證明觀察到的通路；引用支持、內容品質及多人隔離需另行驗證。
