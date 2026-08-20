---
lab:
  topic: Integrate backend services
  title: Azure Service Bus を使ってメッセージを処理する
  description: Python SDK で Azure Service Bus のキュー、トピック、サブスクリプションを使用してメッセージを送信、受信、ルーティングする方法を学習します。
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Service Bus
---

# Azure Service Bus を使ってメッセージを処理する

AI ワークフローは、多くの場合、メッセージングに依存して、要求の取り込みをモデル推論から切り離し、結果を複数のダウンストリーム コンシューマーにルーティングします。 Azure Service Bus では、これらのコンポーネントを接続する信頼性とルーティング レイヤーが提供されるため、各コンポーネントを個別にスケーリングでき、障害を個々にとどめることができます。

この演習では、Azure Service Bus 名前空間を作成し、AI 推論シナリオを使用したコア メッセージング パターンを実際に示す Python Flask Web アプリケーションを構築します。 peek-lock 配信を使用してキューで推論リクエストの送受信を行い、処理に失敗した不正なペイロードを確認するためにデッドレター キューを調査し、さらに優先度レベル別に推論結果を配信するため、フィルター付きサブスクリプションを持つトピックを使用します。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードする
- Azure Service Bus の名前空間とメッセージング エンティティを作成する
- スターター ファイルにコードを追加してアプリを完成する
- アプリを実行してメッセージング操作を実行する

この演習の所要時間は約 **30** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- [Python 3.12](https://www.python.org/downloads/) 以上。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。

## プロジェクト スターター ファイルをダウンロードして Azure Service Bus をデプロイする

このセクションでは、アプリのスターター ファイルをダウンロードし、スクリプトを使用して Azure Service Bus の名前空間とメッセージング エンティティをサブスクリプションにデプロイします。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/service-bus-python.zip
    ```

1. プロジェクトで作業するシステム内の場所にファイルをコピーまたは移動します。 その後、ファイルをフォルダーに解凍します。

1. Visual Studio Code (VS Code) を起動し、メニューで **[ファイル] > [フォルダーを開く...]** を選択してから、プロジェクト ファイルを含むフォルダーを選びます。

1. *azdeploy.py* デプロイ スクリプトを開き、スクリプト上部の 2 つの値を必要に応じて変更して、変更を保存します。 **注:** スクリプトの他の部分は変更しないでください。

    ```
    "<your-resource-group-name>" # Resource Group name
    "<your-azure-region>" # Azure region for the resources
    ```

1. メニュー バーで、**[ターミナル] > [新しいターミナル]** を選択して、VS Code でターミナル ウィンドウを開きます。

1. 次のコマンドを実行して、Azure アカウントにログインします。 画面の指示に従って、演習用の Azure アカウントとサブスクリプションを選択します。

    ```
    az login
    ```

1. 次のコマンドを実行して、演習に必要なリソース プロバイダーが自分のサブスクリプションにあることを確認します。

    ```
    az provider register --namespace Microsoft.ServiceBus
    ```

1. ターミナルで次のコマンドを実行して、デプロイ スクリプトを起動します。

    ```
    python azdeploy.py
    ```

1. スクリプトの実行中に、「**1**」と入力して **[1. Service Bus の名前空間を作成する]** オプションを起動します。

    このオプションは、リソース グループがまだ存在していない場合にそれを作成し、Standard レベルの Azure Service Bus 名前空間をデプロイします。 名前空間は、演習中に作成するすべてのメッセージング エンティティ用のコンテナーです。

1. 「**2**」と入力して、**[2. メッセージング エンティティを作成する]** オプションを実行します。 これにより、アプリで使用するキュー、トピック、サブスクリプション、SQL フィルターが作成されます。 **inference-requests** キューでは、最大配信数が 5 に構成され、メッセージの有効期限切れによる配信不能が構成されています。 **inference-results** トピックには、**notifications** サブスクリプション (すべてのメッセージを受信する) と **high-priority** サブスクリプション (**priority** プロパティが **high** と等しいメッセージのみを受信するようにフィルター処理される) の 2 つのサブスクリプションがあります。

1. 「**3**」と入力して、**[3. ロールを割り当てる]** オプションを実行します。 これにより、アカウントに Azure Service Bus Data Owner ロールが割り当てられるため、Microsoft Entra 認証を使用してメッセージを送受信できるようになります。

1. 「**4**」と入力して、**[4. デプロイ状態を確認する]** オプションを実行します。 続行する前に、名前空間の状態に **Succeeded** と表示され、メッセージング エンティティが作成され、ロールが割り当てられていることを確認します。 名前空間のプロビジョニングがまだ完了していない場合は、しばらく待ってからもう一度試してください。

1. 「**5**」と入力して、**[5. 接続情報を取得する]** オプションを実行します。 これにより、アプリで必要な完全修飾ドメイン名 (FQDN) を含む環境変数ファイルが作成されます。

1. 「**6**」と入力して、デプロイ スクリプトを終了します。

1. 適切なコマンドを実行して、前の手順で作成したファイルから環境変数をターミナル セッションに読み込みます。

    **Bash**
    ```bash
    source .env
    ```

    **PowerShell**
    ```powershell
    . .\.env.ps1
    ```

    >**注:** ターミナルは、開いたままにします。 ターミナルを閉じて新しいターミナルを作成する場合は、このコマンドをもう一度実行して、環境変数をもう一度読み込む必要があります。

## アプリの仕上げ

このセクションでは、*service_bus_functions.py* ファイルにコードを追加して、Service Bus メッセージング関数を完成させます。 *app.py* 内の Flask アプリは、これらの関数を呼び出し、結果をブラウザーに表示します。 このアプリは、演習の後半で実行します。

1. *client/service_bus_functions.py* ファイルを開き、コードの追加を開始します。

>**注:** アプリケーションに追加するコード ブロックは、コードのそのセクションのコメントと一致する必要があります。

### キューにメッセージを送信するコードを追加する

このセクションでは、3 つのメッセージをキューに送信するコードを追加します。 2 つのメッセージには、推論要求を表す有効な JSON ペイロードが含まれており、1 つのメッセージには、処理の失敗をシミュレートして配信不能キューを実証するために意図的に誤った形式にした JSON が含まれます。

この関数は、**DefaultAzureCredential** を使用して **ServiceBusClient** を開き、**get_queue_sender()** を使用してキュー センダーを作成します。 3 つの **ServiceBusMessage** オブジェクトが構築され、それぞれが重複除去用の **message_id**、追跡用の **correlation_id**、カスタム メタデータ用の **application_properties** を持ちます。 2 つのメッセージには有効な JSON ペイロードが含まれ、1 つのメッセージには意図的に誤った形式にした JSON が含まれます。 **send_messages()** メソッドは各メッセージを個別にキューに送信します。

>**ヒント:** コードの適切なインデントを維持するには、左余白 (列 1) のコード揃えを貼り付け、貼り付けられた行をすべて選択して、**Tab** キーを押し、ブロックを**開始/終了**のマーカーに合わせます。 必要に応じて **Shift + Tab** キーを押してインデントを戻してください。

1. コメント **# BEGIN SEND MESSAGES FUNCTION** を見つけ、そのコメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def send_messages():
        """Send messages to the queue including one malformed message."""
        # Create a ServiceBusClient using DefaultAzureCredential
        client = get_client()
        results = []

        with client:
            # Open a sender bound to the queue
            with client.get_queue_sender(QUEUE_NAME) as sender:
                # Valid message 1 — message_id enables deduplication,
                # correlation_id links related messages for tracking,
                # and application_properties carry custom metadata for routing
                msg1 = ServiceBusMessage(
                    body=json.dumps({
                        "prompt": "Extract parties and effective date.",
                        "model": "gpt-4o",
                        "document_id": "doc-001"
                    }),
                    content_type="application/json",
                    message_id=str(uuid.uuid4()),
                    correlation_id="req-doc-001",
                    application_properties={"priority": "standard", "document_type": "contract"}
                )
                sender.send_messages(msg1)
                results.append({
                    "correlation_id": msg1.correlation_id,
                    "type": "valid",
                    "status": "sent"
                })

                # Valid message 2 — priority is set to "high" so it matches
                # the SQL filter on the high-priority topic subscription
                msg2 = ServiceBusMessage(
                    body=json.dumps({
                        "prompt": "Summarize the key terms.",
                        "model": "gpt-4o",
                        "document_id": "doc-002"
                    }),
                    content_type="application/json",
                    message_id=str(uuid.uuid4()),
                    correlation_id="req-doc-002",
                    application_properties={"priority": "high", "document_type": "contract"}
                )
                sender.send_messages(msg2)
                results.append({
                    "correlation_id": msg2.correlation_id,
                    "type": "valid",
                    "status": "sent"
                })

                # Invalid message — intentionally malformed JSON body to
                # demonstrate dead-lettering during processing
                msg3 = ServiceBusMessage(
                    body="not valid json: [broken",
                    content_type="application/json",
                    message_id=str(uuid.uuid4()),
                    correlation_id="req-doc-003",
                    application_properties={"priority": "standard"}
                )
                sender.send_messages(msg3)
                results.append({
                    "correlation_id": msg3.correlation_id,
                    "type": "malformed",
                    "status": "sent"
                })

        return results
    ```

1. 少し時間をかけて  コードを確認しましょう。

### ピークロックを使用してメッセージを処理するコードを追加する

このセクションでは、ピークロックを使用してキューからメッセージを受信するコードを追加します。 プロセッサは、JSON ペイロードを検証し、有効なメッセージを完了させ、無効な JSON を含むメッセージについては、理由とエラーの説明を提供して配信不能処理を行います。

この関数は、ピークロック モード (既定値) を使用して **get_queue_receiver()** でキュー レシーバーを作成します。このモードでは、各メッセージは処理中にロックされますが、解決されるまでキュー内に保持されます。 メッセージごとに、JSON 本文の解析が試みられます。 有効なメッセージは、**complete_message()** によりキューから削除されます。 無効なメッセージは **dead_letter_message()** により配信不能サブキューに移動され、診断のために **reason** と **error_description** が提供されます。

1. コメント **# BEGIN PROCESS MESSAGES FUNCTION** を見つけ、そのコメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def process_messages():
        """Receive and process messages from the queue using peek-lock."""
        client = get_client()
        results = []

        with client:
            # Peek-lock is the default receive mode — the message is locked
            # but stays in the queue until explicitly completed or dead-lettered.
            # max_wait_time sets how long the receiver waits for new messages.
            with client.get_queue_receiver(
                queue_name=QUEUE_NAME,
                max_wait_time=5
            ) as receiver:
                for msg in receiver:
                    try:
                        payload = json.loads(str(msg))
                        # Complete removes the message from the queue
                        receiver.complete_message(msg)
                        results.append({
                            "correlation_id": msg.correlation_id,
                            "document_id": payload.get("document_id"),
                            "model": payload.get("model"),
                            "prompt": payload.get("prompt", "")[:50],
                            "status": "completed"
                        })
                    except json.JSONDecodeError:
                        # Dead-letter moves the message to the dead-letter
                        # sub-queue with a reason and description for diagnostics
                        receiver.dead_letter_message(
                            msg,
                            reason="MalformedPayload",
                            error_description="Message body is not valid JSON"
                        )
                        results.append({
                            "correlation_id": msg.correlation_id,
                            "document_id": None,
                            "model": None,
                            "prompt": str(msg)[:50],
                            "status": "dead-lettered"
                        })

        return results
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### 配信不能キューを調べるコードを追加する

このセクションでは、配信不能キューからメッセージを読み取り、診断情報を表示するコードを追加します。 配信不能キューは、処理できなかったメッセージと、トラブルシューティングのためにその理由とエラーの説明を取り込みます。

この関数は、**sub_queue=ServiceBusSubQueue.DEAD_LETTER** を **get_queue_receiver()** に渡して、配信不能サブキューをターゲットとするレシーバーを作成します。 レシーバーは、配信不能処理されたメッセージごとに、診断プロパティ (**dead_letter_reason**、**dead_letter_error_description**、**delivery_count**) を読み取ります。 読み取った後、**complete_message()** を呼び出して、メッセージを配信不能キューから削除します。

1. コメント **# BEGIN INSPECT DLQ FUNCTION** を見つけ、そのコメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def inspect_dead_letter_queue():
        """Inspect and remove messages from the dead-letter queue."""
        client = get_client()
        results = []

        with client:
            # ServiceBusSubQueue.DEAD_LETTER targets the dead-letter sub-queue,
            # which holds messages that failed processing
            with client.get_queue_receiver(
                queue_name=QUEUE_NAME,
                sub_queue=ServiceBusSubQueue.DEAD_LETTER,
                max_wait_time=5
            ) as dlq_receiver:
                for msg in dlq_receiver:
                    # Dead-lettered messages include diagnostic properties:
                    # dead_letter_reason, dead_letter_error_description,
                    # and delivery_count (number of delivery attempts)
                    results.append({
                        "message_id": msg.message_id,
                        "correlation_id": msg.correlation_id,
                        "dead_letter_reason": msg.dead_letter_reason,
                        "error_description": msg.dead_letter_error_description,
                        "delivery_count": msg.delivery_count,
                        "body": str(msg)[:100]
                    })
                    # Complete removes the message from the dead-letter queue
                    dlq_receiver.complete_message(msg)

        return results
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### フィルター処理されたサブスクリプションを使用してトピック メッセージングのコードを追加する

このセクションでは、異なる優先度レベルでトピックにメッセージを送信し、各サブスクリプションからメッセージを受信するコードを追加して、フィルター処理が正しく機能することを確認します。 **notifications** サブスクリプションはすべてのメッセージを受信し、**high-priority** サブスクリプションは、**priority** アプリケーション プロパティが **high** と等しいメッセージのみを受信します。

この関数は、**get_topic_sender()** を使用して、トピックにバインドされたセンダーを開き、**application_properties** 内の **priority** 値が異なる 5 つのメッセージを送信します。 その後、**get_subscription_receiver()** で 2 つのサブスクリプション レシーバーを開きます。 **notifications** サブスクリプションにはフィルターがなく、5 個のメッセージをすべて受信します。 **high-priority** サブスクリプションは、SQL フィルターと一致するメッセージのみを配信します。 AMQP エンコードのために、アプリケーション プロパティがバイトとして到着する場合があるため、このコードは文字列キーとバイト キーの両方を処理することに注意してください。

1. コメント **# BEGIN TOPIC MESSAGING FUNCTION** を見つけ、そのコメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def topic_messaging():
        """Send messages to a topic and receive from filtered subscriptions."""
        client = get_client()
        sent = []
        notifications = []
        high_priority = []

        with client:
            # get_topic_sender opens a sender bound to a topic instead of a queue.
            # Each message is broadcast to all matching subscriptions.
            with client.get_topic_sender(TOPIC_NAME) as sender:
                for i, priority in enumerate(["standard", "high", "standard", "high", "low"]):
                    result = {
                        "document_id": f"doc-{i+1:03d}",
                        "status": "completed",
                        "confidence": 0.95
                    }
                    # The "priority" application property is what the SQL
                    # filter on the high-priority subscription evaluates
                    msg = ServiceBusMessage(
                        body=json.dumps(result),
                        content_type="application/json",
                        message_id=str(uuid.uuid4()),
                        application_properties={"priority": priority}
                    )
                    sender.send_messages(msg)
                    sent.append({
                        "document_id": f"doc-{i+1:03d}",
                        "priority": priority
                    })

            # Receive from the notifications subscription, which has no filter
            # and therefore receives all messages sent to the topic
            with client.get_subscription_receiver(
                topic_name=TOPIC_NAME,
                subscription_name="notifications",
                max_wait_time=5
            ) as receiver:
                for msg in receiver:
                    body = json.loads(str(msg))
                    # Application properties may arrive as bytes depending
                    # on the AMQP encoding, so handle both str and bytes keys
                    props = msg.application_properties or {}
                    priority_val = props.get("priority") or props.get(b"priority", b"unknown")
                    if isinstance(priority_val, bytes):
                        priority_val = priority_val.decode("utf-8")
                    notifications.append({
                        "document_id": body["document_id"],
                        "priority": priority_val
                    })
                    receiver.complete_message(msg)

            # Receive from the high-priority subscription, which only delivers
            # messages where the SQL filter "priority = 'high'" matches
            with client.get_subscription_receiver(
                topic_name=TOPIC_NAME,
                subscription_name="high-priority",
                max_wait_time=5
            ) as receiver:
                for msg in receiver:
                    body = json.loads(str(msg))
                    props = msg.application_properties or {}
                    priority_val = props.get("priority") or props.get(b"priority", b"unknown")
                    if isinstance(priority_val, bytes):
                        priority_val = priority_val.decode("utf-8")
                    high_priority.append({
                        "document_id": body["document_id"],
                        "priority": priority_val
                    })
                    receiver.complete_message(msg)

        return {
            "sent": sent,
            "notifications": notifications,
            "high_priority": high_priority
        }
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

## Python 環境を構成する

このセクションでは、クライアント アプリ ディレクトリに移動し、Python 環境を作成し、依存関係をインストールします。

1. VS Code ターミナルで次のコマンドを実行して、*client* ディレクトリに移動します。

    ```
    cd client
    ```

1. 次のコマンドを実行して、Python 環境を作成します。

    ```
    python -m venv .venv
    ```

1. 次のコマンドを使用して、Python 環境をアクティブ化します。 **注:** Linux/macOS では、Bash コマンドを使用してください。 Windows では、PowerShell コマンドを使用します。 Windows で Git Bash を使っている場合は、**source .venv/Scripts/activate** を使用します。

    **Bash**
    ```bash
    source .venv/bin/activate
    ```

    **PowerShell**
    ```powershell
    .\.venv\Scripts\Activate.ps1
    ```

1. VS Code ターミナルで次のコマンドを実行して、依存関係をインストールします。

    ```
    pip install -r requirements.txt
    ```

## アプリを実行する

このセクションでは、完成した Flask アプリケーションを実行して、さまざまな Service Bus メッセージング操作を実行します。 このアプリは、メッセージの送信、ピークロック配信によるメッセージの処理、配信不能キューの検査、フィルター処理されたサブスクリプションによるトピック メッセージングのテストを行うことができる Web インターフェイスを提供します。

1. ターミナルで次のコマンドを実行して、アプリを起動します。 コマンドを実行する前に、演習の前半のコマンドを参照して、必要に応じて環境をアクティブ化します。 *client* ディレクトリから移動した場合は、まず **cd client** を実行します。

    ```
    python app.py
    ```

1. ブラウザーを開き、`http://localhost:5000` に移動してアプリにアクセスします。

1. 左側のパネルで **[メッセージの送信]** を選択します。 これにより、キューに 3 つのメッセージが送信されます。2 つには推論要求を表す有効な JSON ペイロードが含まれ、1 つには意図的に誤った形式にした JSON が含まれます。 右側のパネルの結果を見ると、各メッセージが、その関連付け ID と種類と共に送信されたことを確認できます。

1. **[メッセージの処理]** を選択します。 これにより、ピークロック配信を使用して 3 つのメッセージがキューから受信されます。 プロセッサは各メッセージの JSON ペイロードを検証し、2 つの有効なメッセージを完了させ、形式に誤りがあるメッセージは、**MalformedPayload** という理由で配信不能処理されます。 結果は、処理後の各メッセージの状態を示します。

1. **[配信不能キューの検査]** を選択します。 これにより、配信不能処理されたメッセージが読み取られ、その診断情報 (配信不能の理由、エラーの説明、配信回数など) が表示されます。 配信不能キューは、処理できなかったメッセージをキャプチャして、エラーを調査できるようにします。

1. **[トピック メッセージの送受信]** を選択します。 これにより、**inference-results** トピックに、5 つのメッセージがそれぞれ異なる優先度で送信されます。 その後、両方のサブスクリプションからデータを読み取り、フィルター処理された配信を実証します。 **notifications** サブスクリプションは 5 つのメッセージすべてを受信し、**high-priority** サブスクリプションは、**priority** プロパティが **high** と等しいメッセージのみを受信します。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループと、そのグループ内のすべてのリソースを削除します。 **\<rg-name>** は、この演習で選択した名前に置き換えてください。 このコマンドを実行すると Azure の中でバックグラウンド タスクが起動されてリソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の実行中に問題が発生した場合は、次のトラブルシューティング手順をお試しください。

**Azure Service Bus 名前空間のデプロイを確認する**
- [Azure portal](https://portal.azure.com) に移動して、リソース グループを見つけます。
- Service Bus 名前空間の **[プロビジョニングの状態]** が **Succeeded** であることを確認します。
- 名前空間のレベルが **Standard** (トピックやサブスクリプションに必須) であることを確認します。

**メッセージング エンティティを確認する**
- デプロイ スクリプトの **[デプロイ状態を確認する]** オプションを実行し、キュー、トピック、サブスクリプション、SQL フィルターがすべて正常に作成されていることを確認します。
- エンティティがない場合は、**[メッセージング エンティティを作成する]** オプションをもう一度実行します。

**コードの完全性とインデントを確認する**
- すべてのコード ブロックが、*service_bus_functions.py* 内の適切な BEGIN/END コメント マーカーの間にある正しいセクションに追加されていることを確認します。
- Python のインデントが一貫していること (タブではなくスペースを使用していること)、およびすべてのコードが関数内で正しく配置されていることを確認します。
- 指定されたセクションの外部でコードが誤って削除または変更されていないことを確認します。

**環境変数を確認する**
- *.env* と *.env.ps1* の両方のファイルがプロジェクトのルートに存在し、**SERVICE_BUS_FQDN** の値を含んでいることを確認します。
- **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行して環境変数をターミナル セッションに読み込みます。

**認証を確認する**
- **az account show** を実行して、Azure CLI にログインしていることを確認します。
- Azure portal でロールの割り当てを確認するか、デプロイ スクリプトのオプションを実行してロールをもう一度割り当てて、Azure Service Bus Data Owner ロールがアカウントに割り当てられていることを確認します。

**Python 環境と依存関係を確認する**
- アプリを実行する前に、仮想環境がアクティブになっていることを確認します。
- **pip list** を実行して、*requirements.txt* のすべてのパッケージが正常にインストールされたことを確認します。
