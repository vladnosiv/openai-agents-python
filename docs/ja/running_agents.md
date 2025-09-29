---
search:
  exclude: true
---
# エージェントの実行

エージェントは [`Runner`][agents.run.Runner] クラスで実行できます。選択肢は 3 つあります。

1. [`Runner.run()`][agents.run.Runner.run]: 非同期で実行され、[`RunResult`][agents.result.RunResult] を返します。
2. [`Runner.run_sync()`][agents.run.Runner.run_sync]: 同期メソッドで、内部的には `.run()` を実行します。
3. [`Runner.run_streamed()`][agents.run.Runner.run_streamed]: 非同期で実行され、[`RunResultStreaming`][agents.result.RunResultStreaming] を返します。LLM をストリーミングモードで呼び出し、受信したイベントをそのままストリーミングします。

```python
from agents import Agent, Runner

async def main():
    agent = Agent(name="Assistant", instructions="You are a helpful assistant")

    result = await Runner.run(agent, "Write a haiku about recursion in programming.")
    print(result.final_output)
    # Code within the code,
    # Functions calling themselves,
    # Infinite loop's dance
```

詳細は [結果ガイド](results.md) をご覧ください。

## エージェントのループ

`Runner` の run メソッドを使うときは、開始するエージェントと入力を渡します。入力は文字列（ユーザー メッセージとして扱われます）か、OpenAI Responses API のアイテムからなる入力アイテムのリストのいずれかです。

Runner は次のループを実行します。

1. 現在のエージェントに対して、現在の入力で LLM を呼び出します。
2. LLM が出力を生成します。
    1. LLM が `final_output` を返した場合、ループを終了し結果を返します。
    2. LLM が ハンドオフ を行った場合、現在のエージェントと入力を更新し、ループを再実行します。
    3. LLM が ツール呼び出し を生成した場合、それらを実行し、結果を追加して、ループを再実行します。
3. 渡された `max_turns` を超えた場合、[`MaxTurnsExceeded`][agents.exceptions.MaxTurnsExceeded] 例外を送出します。

!!! note

    LLM の出力が「最終出力」と見なされるルールは、所望の型のテキスト出力を生成し、かつツール呼び出しがない場合です。

## ストリーミング

ストリーミングを使うと、LLM の実行と同時にストリーミングイベントも受け取れます。ストリーム完了後、[`RunResultStreaming`][agents.result.RunResultStreaming] には、実行中に生成された新しい出力を含む、実行の完全な情報が含まれます。ストリーミングイベントは `.stream_events()` を呼び出して受け取れます。詳細は [ストリーミング ガイド](streaming.md) をご覧ください。

## 実行設定

`run_config` パラメーターで、エージェント実行のグローバル設定を構成できます。

-   [`model`][agents.run.RunConfig.model]: 各 Agent の `model` に関係なく、使用するグローバルな LLM モデルを設定します。
-   [`model_provider`][agents.run.RunConfig.model_provider]: モデル名を解決するためのモデルプロバイダーで、デフォルトは OpenAI です。
-   [`model_settings`][agents.run.RunConfig.model_settings]: エージェント固有の設定を上書きします。たとえば、グローバルな `temperature` や `top_p` を設定できます。
-   [`input_guardrails`][agents.run.RunConfig.input_guardrails], [`output_guardrails`][agents.run.RunConfig.output_guardrails]: すべての実行に含める入力/出力 ガードレール のリストです。
-   [`handoff_input_filter`][agents.run.RunConfig.handoff_input_filter]: ハンドオフに既定のフィルタがない場合に適用される、すべてのハンドオフに対するグローバルな入力フィルタです。入力フィルタは、新しいエージェントに送信される入力を編集できます。詳細は [`Handoff.input_filter`][agents.handoffs.Handoff.input_filter] のドキュメントをご覧ください。
-   [`tracing_disabled`][agents.run.RunConfig.tracing_disabled]: 実行全体の [トレーシング](tracing.md) を無効化します。
-   [`trace_include_sensitive_data`][agents.run.RunConfig.trace_include_sensitive_data]: トレースに、LLM やツール呼び出しの入出力など、機微なデータを含めるかどうかを設定します。
-   [`workflow_name`][agents.run.RunConfig.workflow_name], [`trace_id`][agents.run.RunConfig.trace_id], [`group_id`][agents.run.RunConfig.group_id]: 実行のトレーシング ワークフロー名、トレース ID、トレース グループ ID を設定します。少なくとも `workflow_name` を設定することを推奨します。グループ ID は任意で、複数の実行にまたがるトレースを関連付けできます。
-   [`trace_metadata`][agents.run.RunConfig.trace_metadata]: すべてのトレースに含めるメタデータです。

## 会話/チャットスレッド

いずれの run メソッドを呼び出しても、1 つ以上のエージェントが実行される（つまり 1 回以上の LLM 呼び出しが行われる）可能性がありますが、これはチャット会話における 1 つの論理的なターンを表します。例:

1. ユーザーのターン: ユーザーがテキストを入力
2. Runner の実行: 最初のエージェントが LLM を呼び出し、ツールを実行し、2 つ目のエージェントへハンドオフ、2 つ目のエージェントがさらにツールを実行し、その後出力を生成

エージェントの実行が終わったら、ユーザーに何を表示するかを選べます。たとえば、エージェントが生成したすべての新規アイテムを表示するか、最終出力のみを表示するかです。いずれの場合でも、ユーザーが追質問をするかもしれず、その場合は再度 run メソッドを呼び出せます。

### 手動の会話管理

次のターンの入力を得るために、[`RunResultBase.to_input_list()`][agents.result.RunResultBase.to_input_list] メソッドを使って、会話履歴を手動で管理できます。

```python
async def main():
    agent = Agent(name="Assistant", instructions="Reply very concisely.")

    thread_id = "thread_123"  # Example thread ID
    with trace(workflow_name="Conversation", group_id=thread_id):
        # First turn
        result = await Runner.run(agent, "What city is the Golden Gate Bridge in?")
        print(result.final_output)
        # San Francisco

        # Second turn
        new_input = result.to_input_list() + [{"role": "user", "content": "What state is it in?"}]
        result = await Runner.run(agent, new_input)
        print(result.final_output)
        # California
```

### Sessions を使った自動会話管理

より簡単な方法として、[Sessions](sessions.md) を使うと、`.to_input_list()` を手動で呼び出さずに会話履歴を自動的に扱えます。

```python
from agents import Agent, Runner, SQLiteSession

async def main():
    agent = Agent(name="Assistant", instructions="Reply very concisely.")

    # Create session instance
    session = SQLiteSession("conversation_123")

    thread_id = "thread_123"  # Example thread ID
    with trace(workflow_name="Conversation", group_id=thread_id):
        # First turn
        result = await Runner.run(agent, "What city is the Golden Gate Bridge in?", session=session)
        print(result.final_output)
        # San Francisco

        # Second turn - agent automatically remembers previous context
        result = await Runner.run(agent, "What state is it in?", session=session)
        print(result.final_output)
        # California
```

Sessions は自動的に次を行います。

-   各実行前に会話履歴を取得
-   各実行後に新しいメッセージを保存
-   セッション ID ごとに独立した会話を維持

詳細は [Sessions のドキュメント](sessions.md) をご覧ください。

## 長時間稼働エージェントとヒューマンインザループ

Agents SDK の [Temporal](https://temporal.io/) 連携を使うと、ヒューマンインザループのタスクを含む、永続的で長時間稼働するワークフローを実行できます。Temporal と Agents SDK が長時間タスクを完了する様子のデモは [この動画](https://www.youtube.com/watch?v=fFBZqzT4DD8) を、ドキュメントは [こちら](https://github.com/temporalio/sdk-python/tree/main/temporalio/contrib/openai_agents) をご覧ください。

## 例外

SDK は特定のケースで例外を送出します。完全な一覧は [`agents.exceptions`][] にあります。概要は以下のとおりです。

-   [`AgentsException`][agents.exceptions.AgentsException]: SDK 内で送出されるすべての例外の基底クラスです。その他の特定の例外はここから派生します。
-   [`MaxTurnsExceeded`][agents.exceptions.MaxTurnsExceeded]: エージェントの実行が `Runner.run`、`Runner.run_sync`、`Runner.run_streamed` メソッドに渡した `max_turns` 制限を超えたときに送出されます。指定されたインタラクション回数内にタスクを完了できなかったことを示します。
-   [`ModelBehaviorError`][agents.exceptions.ModelBehaviorError]: 基盤となるモデル（LLM）が予期しない、または無効な出力を生成したときに発生します。これには次が含まれます。
    -   不正な JSON: 特定の `output_type` が定義されている場合に特に、ツール呼び出しや直接の出力で不正な JSON 構造を返す場合。
    -   予期しないツール関連の失敗: モデルが想定どおりにツールを使用できない場合
-   [`UserError`][agents.exceptions.UserError]:（SDK を使うコード記述者である）あなたが SDK の使用中に誤りを犯したときに送出されます。これは通常、不正なコード実装、無効な構成、または SDK の API の誤用が原因です。
-   [`InputGuardrailTripwireTriggered`][agents.exceptions.InputGuardrailTripwireTriggered], [`OutputGuardrailTripwireTriggered`][agents.exceptions.OutputGuardrailTripwireTriggered]: それぞれ、入力 ガードレール または出力 ガードレール の条件が満たされたときに送出されます。入力 ガードレール は処理前に受信メッセージをチェックし、出力 ガードレール は配信前にエージェントの最終応答をチェックします。