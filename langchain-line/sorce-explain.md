**LINEのWebhookイベントを受け取り、AWS Bedrockの基盤モデルを使って応答を生成し、LINEに返信するLambda関数**の実装。会話履歴をDynamoDBに保存し、LangChainを使って過去のやり取りを踏まえた応答を返す仕組み。

***

### **1. 主要なライブラリと役割**

*   **os**: 環境変数の取得。
*   **boto3**: AWS SDK。Bedrock Runtimeクライアントを作成。
*   **langchain**: 会話チェーンやメモリ管理を提供。
*   **langchain\_aws.ChatBedrock**: AWS BedrockモデルをLangChainのLLMとして利用。
*   **linebot.v3**: LINE Messaging API SDK。

***

### **2. 環境変数**

```python
LINE_CHANNEL_ACCESS_TOKEN = os.getenv("LINE_CHANNEL_ACCESS_TOKEN")
LINE_CHANNEL_SECRET = os.getenv("LINE_CHANNEL_SECRET")
NUM_OF_HISTORY = os.getenv("NUM_OF_HISTORY", 10)
FOUNDATION_MODEL = os.getenv("FOUNDATION_MODEL")
DYNAMODB_TABLE_NAME = os.getenv("DYNAMODB_TABLE_NAME")
```

*   **LINE\_CHANNEL\_ACCESS\_TOKEN / SECRET**: LINE Bot認証情報。
*   **NUM\_OF\_HISTORY**: 会話履歴の保持数（デフォルト10）。
*   **FOUNDATION\_MODEL**: Bedrockで使うモデルID（例: `anthropic.claude-v2`）。
*   **DYNAMODB\_TABLE\_NAME**: 会話履歴を保存するDynamoDBテーブル名。

***

### **3. LINEハンドラーとAPIクライアント**

```python
line_configuration = Configuration(access_token=LINE_CHANNEL_ACCESS_TOKEN)
line_handler = WebhookHandler(channel_secret=LINE_CHANNEL_SECRET)
```

*   LINEのWebhookイベントを受け取るための設定。

***

### **4. Bedrockモデルの設定**

```python
llm = ChatBedrock(
    model_id=FOUNDATION_MODEL,
    model_kwargs={"max_tokens": 4096},
    client=boto3.client("bedrock-runtime"),
)
```

*   Bedrockの基盤モデルをLangChainの`llm`として利用。
*   `max_tokens`で応答の最大トークン数を指定。

***

### **5. メッセージ受信時の処理**

```python
@line_handler.add(MessageEvent, message=TextMessageContent)
def handle_message(event: MessageEvent):
    user_id = event.source.user_id
    text = event.message.text

    response = conversation(input=text, session_id=user_id)

    with ApiClient(line_configuration) as api_client:
        line_bot_api = MessagingApi(api_client)
        line_bot_api.reply_message_with_http_info(
            ReplyMessageRequest(
                reply_token=event.reply_token, messages=[TextMessage(text=response)]
            )
        )
```

*   LINEからメッセージを受け取ると`conversation()`で応答を生成。
*   その応答をLINEに返信。

***

### **6. 会話処理（LangChain + DynamoDB）**

```python
def conversation(input: str, session_id: str, num_of_history: int = NUM_OF_HISTORY):
    message_history = DynamoDBChatMessageHistory(
        table_name=DYNAMODB_TABLE_NAME,
        session_id=session_id,
    )

    memory = ConversationBufferWindowMemory(
        memory_key="history",
        chat_memory=message_history,
        return_messages=True,
        k=num_of_history,
    )

    conversation = ConversationChain(llm=llm, verbose=True, memory=memory)

    return conversation.predict(input=input)
```

*   **DynamoDBChatMessageHistory**: DynamoDBに会話履歴を保存。
*   **ConversationBufferWindowMemory**: 最新の履歴`k`件を保持。
*   **ConversationChain**: LLMとメモリを組み合わせて応答生成。

***

### **7. Lambdaハンドラー**

```python
def lambda_handler(event, context):
    signature = event["headers"]["x-line-signature"]
    body = event["body"]
    line_handler.handle(body, signature)
    return {"statusCode": 200, "body": "OK"}
```

*   API Gateway経由でLINEのWebhookイベントを受け取り、LINE SDKで処理。

***

✅ **このコードの役割まとめ**

*   LINE Bot → Lambda → Bedrock → 応答 → LINE返信。
*   会話履歴はDynamoDBで管理。
*   LangChainで過去の履歴を踏まえた自然な応答。

***

次にどうしますか？

*   **① アーキテクチャ図を作成**して視覚的に理解しますか？
*   \*\*② このコードの改善ポイント（例: エラーハンドリング、非同期化）\*\*を提案しますか？
*   \*\*③ デプロイ手順（Lambda + API Gateway + LINE設定）\*\*をまとめますか？
