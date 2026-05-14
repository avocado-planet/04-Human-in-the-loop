# Human-In-The-Loop Middleware について

## 概要

`HumanInTheLoopMiddleware` は、エージェントが危険なツールを呼び出そうとしたときに**実行を一時停止**し、人間の判断（承認・拒否・編集）を待ってから処理を再開する仕組みです。

---

## 全体フロー

```
ユーザー: 「test_data を削除して」
    │
    ▼
エージェント: delete_data(target="test_data") を呼ぼうとする
    │
    ▼
ミドルウェア: 「delete_data は承認が必要」→ interrupt 発生!
    │
    ├─ State を checkpointer に保存
    └─ result.interrupts に中断情報を入れて返す
         │
         ▼
    ┌─────────────────────────┐
    │  人間が判断する           │
    │                         │
    │  approve → そのまま実行   │
    │  reject  → 拒否+理由     │
    │  edit    → 引数修正+実行  │
    └─────────────────────────┘
         │
         ▼
Command(resume={...}) でエージェントを再開
    │
    ▼
最終応答
```

---

## 構成要素の説明

### checkpointer

```python
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    ...,
    checkpointer=InMemorySaver(),
)
```

エージェントが interrupt で一時停止したとき、その時点の **State**（メッセージ履歴、ツール呼び出し情報、カスタムフィールドなど）を**まるごと保存**する仕組みです。

なぜ必要かというと、人間が判断を返すまでには時間がかかります（秒〜時間単位）。その間エージェントの実行は止まっているので、状態がどこかに保存されていなければ再開できません。

| checkpointer | 用途 |
|---|---|
| `InMemorySaver()` | メモリ上に保存。開発・デモ用。プロセスが落ちたら消える |
| DB ベース（PostgreSQL 等） | 本番用。永続化される |

checkpointer がない状態で HITL を使おうとすると、エラーになるか、再開できません。

### thread_id

```python
config = {"configurable": {"thread_id": "session-123"}}
```

checkpointer が State を保存するとき、**どのセッション（会話）の State か**を識別するための ID です。

HITL のフローでは2回の `invoke` を行います:

```
1回目: agent.invoke({...}, config=config)      → interrupt で中断
2回目: agent.invoke(Command(resume=...), config=config)  → 再開
```

この2回で**同じ `thread_id`** を使うことで、checkpointer が「これは中断した会話の続きだ」と認識し、State を復元できます。

別の `thread_id` を使うと、別の会話として扱われるため再開できません。

### version="v2"

```python
result = agent.invoke(..., version="v2")
```

`version="v2"` を指定すると、戻り値が `GraphOutput` オブジェクトになります。このオブジェクトには `.interrupts` 属性があり、中断情報にアクセスできます。

```python
result.interrupts    # 中断情報のリスト（なければ空）
result.messages      # メッセージ履歴（従来通り）
```

`version="v2"` を付けないと、戻り値は普通の辞書になり、interrupt の有無を判定しにくくなります。

### interrupt_on

```python
HumanInTheLoopMiddleware(
    interrupt_on={
        "delete_data": {
            "allowed_decisions": ["approve", "edit", "reject"],
        },
        "get_weather": False,
    }
)
```

ツールごとに中断するかどうかを指定します:

| 設定値 | 意味 |
|---|---|
| `True` | 中断する。全判断（approve/edit/reject）を許可 |
| `False` | 中断しない。自動的に実行される |
| `{"allowed_decisions": [...]}` | 中断する。許可する判断を細かく指定 |

---

## 3つの判断パターン

### approve（承認）

ツール呼び出しを**そのまま実行**します。引数は変更されません。

```python
result = agent.invoke(
    Command(resume={
        "decisions": [{"type": "approve"}]
    }),
    config=config,
    version="v2",
)
```

使い所: ツール呼び出しの内容を確認して問題なしと判断した場合。

### reject（拒否）

ツール呼び出しを**拒否**します。`message` で理由を伝えると、エージェントはその理由を踏まえて応答を生成します。

```python
result = agent.invoke(
    Command(resume={
        "decisions": [{
            "type": "reject",
            "message": "ログは削除しないでください。保持が必要です。",
        }]
    }),
    config=config,
    version="v2",
)
```

使い所: ツール呼び出し自体が不適切な場合。理由を伝えることでエージェントが代替案を提示してくれることもあります。

### edit（編集）

ツール呼び出しの**引数を書き換えてから実行**します。

```python
result = agent.invoke(
    Command(resume={
        "decisions": [{
            "type": "edit",
            "edited_action": {
                "name": "delete_data",
                "args": {"target": "expired_cache_only"},
            },
        }]
    }),
    config=config,
    version="v2",
)
```

使い所: ツール呼び出し自体は正しいが、引数を微調整したい場合。例えばエージェントが広範囲の削除を提案したが、対象を限定したい場合など。

---

## decisions がリストである理由

```python
"decisions": [
    {"type": "approve"},      # 1つ目のツール呼び出しへの判断
    {"type": "reject", ...},  # 2つ目のツール呼び出しへの判断
]
```

モデルが1回の応答で**複数のツールを同時に呼ぶ**場合があります。その場合、`decisions` リストの順番は `result.interrupts` の順番と対応します。各ツール呼び出しに個別の判断を返せます。

---

## メッセージ履歴の流れ

### approve の場合

```
[0] HumanMessage: 「test_data を削除して」
[1] AIMessage: (tool_calls: delete_data)  ← モデルがツール呼び出しを提案
--- ここで interrupt ---
--- approve で再開 ---
[2] ToolMessage: 「'test_data' を削除しました」 ← ツール実行結果
[3] AIMessage: 「test_data を削除しました。」   ← 最終応答
```

### reject の場合

```
[0] HumanMessage: 「important_logs を削除して」
[1] AIMessage: (tool_calls: delete_data)
--- ここで interrupt ---
--- reject で再開 ---
[2] ToolMessage: 「拒否されました: ログは削除しないでください...」 ← 拒否理由
[3] AIMessage: 「承知しました。ログは保持します。」               ← 拒否を踏まえた応答
```

### edit の場合

```
[0] HumanMessage: 「old_cache を削除して」
[1] AIMessage: (tool_calls: delete_data(target="old_cache"))
--- ここで interrupt ---
--- edit で再開（target を "expired_cache_only" に変更）---
[2] ToolMessage: 「'expired_cache_only' を削除しました」 ← 修正された引数で実行
[3] AIMessage: 「expired_cache_only を削除しました。」    ← 最終応答
```

---

## よくあるエラーと対処

### checkpointer がない

```
エラー: Human-in-the-loop requires a checkpointer
```

→ `create_agent(..., checkpointer=InMemorySaver())` を追加。

### thread_id が Step 1 と Step 2 で違う

```
再開しても中断前の状態が見つからない
```

→ 同じ `config = {"configurable": {"thread_id": "..."}}` を使う。

### interrupt_on に対象ツール名がない

→ `interrupt_on` に含まれないツールは自動実行される（`False` と同じ扱い）。明示的に `False` を書くのがベストプラクティス。

---

## 参考リンク

- [Human-in-the-loop 公式ドキュメント](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)
- [HumanInTheLoopMiddleware API リファレンス](https://reference.langchain.com/python/langchain/agents/middleware/human_in_the_loop/HumanInTheLoopMiddleware)
