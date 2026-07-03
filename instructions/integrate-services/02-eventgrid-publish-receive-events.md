---
lab:
  topic: Integrate backend services
  title: Azure Event Grid を使用してイベントを発行および受信する
  description: Azure Event Grid 名前空間でプル配信とフィルター処理されたサブスクリプションを使用して、コンテンツ モデレーション イベントを発行、受信、ルーティングする方法を学習します。
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Event Grid
---

# Azure Event Grid を使用してイベントを発行および受信する

AI コンテンツ モデレーション システムは、送信内容を分類およびレビューする際に、大量のイベントを生成します。 Azure Event Grid には、イベントの種類に基づいてこれらのイベントを適切なダウンストリーム コンシューマーに送信するルーティング レイヤーが用意されているため、各ハンドラーはポーリングや手動フィルター処理なしで必要なイベントのみを受信します。

この演習では、名前空間トピックとフィルター処理されたイベント サブスクリプションを含む Event Grid 名前空間をデプロイし、プル配信を使用してコンテンツ モデレーション イベントの発行と受信を行う Python Flask アプリケーションをビルドします。 Event Grid サブスクリプションは、フラグ付きコンテンツ、承認済みコンテンツ、すべてのイベントを個別のサブスクリプションにルーティングするため、実際のフィルター処理のしくみを確認できます。 また、アプリケーションがイベントを処理する方法を制御するために、プル配信によって提供される受信操作、確認操作、拒否操作も使用します。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードする
- 名前空間トピックを使用して Event Grid 名前空間をデプロイする
- 種類フィルターを使用してイベント サブスクリプションを作成する
- スターター ファイルにコードを追加してアプリを完成させる
- アプリを実行してモデレーション イベントを発行、受信、処理する

この演習の所要時間は約 **30** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- [Python 3.12](https://www.python.org/downloads/) 以上。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。

## プロジェクト スターター ファイルをダウンロードし、リソースをデプロイする

このセクションでは、アプリのスターター ファイルをダウンロードし、スクリプトを使用して Event Grid 名前空間をサブスクリプションにデプロイします。 名前空間には、アプリケーションがモデレーション イベントと、プル配信のためにイベントをフィルター処理して保持するイベント サブスクリプションを発行するトピックが含まれています。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/event-grid-python.zip
    ```

1. プロジェクトで作業するシステム内の場所にファイルをコピーまたは移動します。 その後、ファイルをフォルダーに解凍します。

1. Visual Studio Code (VS Code) を起動し、メニューで **[ファイル] > [フォルダーを開く...]** を選択してから、プロジェクト ファイルを含むフォルダーを選びます。

1. プロジェクトには Bash (*azdeploy.sh*) と PowerShell (*azdeploy.ps1*) の両方のデプロイ スクリプトが含まれています。 自分の環境に適したファイルを開き、スクリプトの先頭の 2 つの値を自分のニーズに合わせて変更してから、変更を保存します。 **注:** スクリプトの他の部分は変更しないでください。

    ```
    "<your-resource-group-name>" # Resource Group name
    "<your-azure-region>" # Azure region for the resources
    ```

1. メニュー バーで、**[ターミナル] > [新しいターミナル]** を選択して、VS Code でターミナル ウィンドウを開きます。

1. 次のコマンドを実行して、Azure アカウントにログインします。 画面の指示に従って、演習用の Azure アカウントとサブスクリプションを選択します。

    ```
    az login
    ```

1. 次のコマンドを実行し、演習に必要なリソース プロバイダーがサブスクリプションにあることを確認します。

    ```
    az provider register --namespace Microsoft.EventGrid
    ```

1. 次のコマンドを実行して、Event Grid CLI 拡張機能をインストールします。 デプロイ スクリプトで使用される名前空間コマンドには、この拡張機能が必要です。

    ```
    az extension add --name eventgrid --yes
    ```

1. ターミナルで適切なコマンドを実行して、スクリプトを起動します。

    **Bash**
    ```bash
    bash azdeploy.sh
    ```

    **PowerShell**
    ```powershell
    ./azdeploy.ps1
    ```

    > **注:** PowerShell がデジタル署名されていないためにスクリプトをブロックした場合は、同じターミナル セッション内で次のコマンドを実行し、再度配置スクリプトを実行してください。 このコマンドは、現在の PowerShell プロセスの実行ポリシーのみを変更します。

    ```powershell
    Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
    ```

1. スクリプトの実行中に、「**1**」と入力して **[1. Event Grid の名前空間とトピックを作成する]** オプションを起動します。

    このオプションは、リソースグループがまだ存在していない場合にそれを作成し、Standard SKU を使用して Event Grid 名前空間をデプロイし、CloudEvents v1.0 入力用に構成された **moderation-events** という名前の名前空間トピックを作成します。 名前空間はトピックとイベント サブスクリプション用のコンテナーであり、プル配信を使用すると、アプリケーションは個別のメッセージング サービスを必要とせずに、Event Grid に直接接続してイベントを受信できます。

1. 「**2**」と入力して、**[2. イベント サブスクリプションを作成する]** オプションを実行します。

    このオプションは、名前空間トピックに 3 つのイベント サブスクリプションを作成します。 **sub-flagged** サブスクリプションは、**com.contoso.ai.ContentFlagged** イベントのみを配信するイベントの種類フィルターを使用します。 **sub-approved** サブスクリプションは、**com.contoso.ai.ContentApproved**のイベントのみを配信します。 **sub-all-events** サブスクリプションにはフィルターがなく、トピックに発行されたすべてのイベントを配信し、監査ログとして機能します。 各サブスクリプションは、プル配信モード、60 秒の受信ロック期間、最大配信回数 10 回、1 日間のイベント有効期限で構成されています。

1. 「**3**」と入力して、**[3. ユーザー ロールを割り当てる]** オプションを実行します。 これにより、名前空間に EventGrid Data Sender ロールと EventGrid Data Receiver ロールが割り当てられるため、アカウントは Microsoft Entra 認証を使用してイベントを発行および受信できるようになります。

1. 「**4**」と入力して、**[4. 接続情報を取得する]** オプションを実行します。 これにより、リソースグループ名、名前空間名、トピック名、および名前空間エンドポイントを含む環境変数ファイルが作成されます。

1. 「**6**」と入力して、デプロイ スクリプトを終了します。

    > **注:** 演習の後半で問題が発生した場合は、スクリプトを再実行し、「**5**」と入力して **[5. デプロイ状態を確認する]** を実行します。 このトラブルシューティング オプションは、名前空間に **Succeeded** ​​と表示されていること、トピックが作成されていること、ロールが割り当てられていること、すべてのイベント サブスクリプションがプロビジョニングされていることを確認します。

1. 適切なコマンドを実行して、前の手順で作成したファイルからターミナル セッションに環境変数を読み込みます。

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

このセクションでは、*event_grid_functions.py* ファイルにコードを追加して、Event Grid 発行関数とプル配信関数を完成します。 *app.py* 内の Flask アプリは、これらの関数を呼び出し、結果をブラウザーに表示します。 このアプリは、演習の後半で実行します。

1. *client/event_grid_functions.py* ファイルを開き、コードの追加を開始します。

>**注:** アプリケーションに追加するコード ブロックは、コードのそのセクションのコメントと一致する必要があります。

### モデレーション イベントを発行するコードを追加する

このセクションでは、5 つのコンテンツ モデレーション イベントを Event Grid 名前空間トピックに発行するコードを追加します。 これらのイベントは、CloudEvents v1.0 スキーマを使用し、フラグ付きコンテンツ、承認済みコンテンツ、エスカレーションされたレビューなど、さまざまなモデレーション結果を表します。そのため、各サブスクリプションのイベントの種類フィルターによって、配信されるイベントがどのように決定されるかを確認できます。

この関数は、*moderation_events.json* ファイルからイベント定義を読み込みます。このファイルには、各イベントの CloudEvent エンベロープ フィールド (**type**、**source**、**subject**) と **data** ペイロードが含まれています。 発行時に、この関数は、各イベントに一意の **id** と現在の UTC **timestamp** を追加し、**CloudEvent** オブジェクトを作成し、**send()** メソッドを使用してそれらのオブジェクトを単一の要求で発行します。 **EventGridPublisherClient** は、名前空間トピック エンドポイントをターゲットとする **namespace_topic** パラメーターを使用して構築され、Microsoft Entra 認証に **DefaultAzureCredential** を使用します。

1. コメント **# BEGIN PUBLISH EVENTS FUNCTION** を見つけ、そのコメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def publish_moderation_events():
        """Publish content moderation events to the Event Grid namespace topic."""
        client = get_publisher_client()
        results = []

        # Load event definitions from the JSON file. Each entry contains the
        # CloudEvent envelope fields (type, source, subject) and the data
        # payload that mirrors a realistic AI content moderation pipeline.
        json_path = os.path.join(os.path.dirname(__file__), "moderation_events.json")
        with open(json_path, "r") as f:
            event_definitions = json.load(f)

        # Build CloudEvent objects from the definitions, adding a unique id
        # and a current UTC timestamp to each event at publish time.
        events = []
        for defn in event_definitions:
            defn["data"]["timestamp"] = datetime.now(timezone.utc).isoformat()
            events.append(
                CloudEvent(
                    type=defn["type"],
                    source=defn["source"],
                    subject=defn["subject"],
                    data=defn["data"],
                    id=str(uuid.uuid4())
                )
            )

        # send() publishes all events to the Event Grid namespace topic in a
        # single request. Event Grid then evaluates each subscription's
        # filters and routes matching events to the configured subscriptions.
        client.send(events)

        for event in events:
            results.append({
                "content_id": event.data["contentId"],
                "event_type": event.type.split(".")[-1],
                "category": event.data["category"],
                "confidence": event.data["confidence"],
                "status": "published"
            })

        return results
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### イベントを受信して確認するコードを追加する

このセクションでは、各サブスクリプションからイベントを受信し、確認するコードを追加して、フィルター処理が機能することを確認します。 プル配信とは、Event Grid がエンドポイントにイベントをプッシュするのではなく、アプリケーションで Event Grid に接続してイベントを要求することを意味します。 受信した各イベントにはロック トークンが含まれており、イベントをサブスクリプションから完全に削除するには、これを確認する必要があります。そうしなければ、ロック期間が切れた後にイベントが再配信されます。

この関数は 3 つのサブスクリプションそれぞれに対して **EventGridConsumerClient** を作成します。 **receive()** メソッドは、**ReceiveDetails** オブジェクトの一覧を返します。各オブジェクトには、**CloudEvent** (**.event**) と **lock_token** を含むブローカー プロパティが含まれています。 処理後、この関数は、収集されたロック トークンを使用して **acknowledge()** を呼び出し、イベントが正常に処理されたことを確認します。

1. コメント **# BEGIN CHECK DELIVERY FUNCTION** を見つけ、そのコメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def check_filtered_delivery():
        """Receive and acknowledge events from each subscription to verify filtering."""
        flagged = []
        approved = []
        all_events = []

        # Receive from the sub-flagged subscription, which only delivers
        # events where the event type is com.contoso.ai.ContentFlagged.
        # receive() returns a list of ReceiveDetails, each containing
        # the CloudEvent and a lock token for acknowledgment.
        consumer = get_consumer_client(SUB_FLAGGED)
        details = consumer.receive(max_events=10, max_wait_time=10)
        tokens = []
        for detail in details:
            event = detail.event
            flagged.append({
                "content_id": event.data.get("contentId"),
                "category": event.data.get("category"),
                "severity": event.data.get("severity"),
                "confidence": event.data.get("confidence")
            })
            tokens.append(detail.broker_properties.lock_token)
        # acknowledge() removes the events from the subscription so they
        # are not delivered again on the next receive call.
        if tokens:
            consumer.acknowledge(lock_tokens=tokens)

        # Receive from the sub-approved subscription, which only delivers
        # events where the event type is com.contoso.ai.ContentApproved.
        consumer = get_consumer_client(SUB_APPROVED)
        details = consumer.receive(max_events=10, max_wait_time=10)
        tokens = []
        for detail in details:
            event = detail.event
            approved.append({
                "content_id": event.data.get("contentId"),
                "category": event.data.get("category"),
                "severity": event.data.get("severity"),
                "confidence": event.data.get("confidence")
            })
            tokens.append(detail.broker_properties.lock_token)
        if tokens:
            consumer.acknowledge(lock_tokens=tokens)

        # Receive from the sub-all-events subscription, which has no filter
        # and delivers every event published to the topic (audit log).
        consumer = get_consumer_client(SUB_ALL)
        details = consumer.receive(max_events=10, max_wait_time=10)
        tokens = []
        for detail in details:
            event = detail.event
            all_events.append({
                "content_id": event.data.get("contentId"),
                "event_type": event.data.get("modelName", "unknown"),
                "category": event.data.get("category"),
                "confidence": event.data.get("confidence")
            })
            tokens.append(detail.broker_properties.lock_token)
        if tokens:
            consumer.acknowledge(lock_tokens=tokens)

        return {
            "flagged": flagged,
            "approved": approved,
            "all_events": all_events
        }
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### イベントを検査し拒否するコードを追加する

このセクションでは、単一のテスト イベントを発行し、それを受信し、CloudEvent エンベロープ全体を検査して拒否するコードを追加します。 イベントを拒否すると、そのイベントは処理できないことが Event Grid に通知されます。 これは、処理が成功したことを確認する確認とは異なります。 拒否されたイベントは破棄されるか、配信不能の宛先が構成されている場合はそこに移動されます。

この関数はまず **EventGridPublisherClient** を使用してテスト イベントを発行するため、以前のイベントが既に確認されているかどうかに関係なく、常に使用可能なイベントが存在します。 その後、**sub-flagged** サブスクリプションからイベントを受信し、CloudEvent 属性とブローカー プロパティ (**delivery_count** など) を抽出し、ロック トークンを使用して **reject()** を呼び出します。

1. コメント **# BEGIN INSPECT AND REJECT FUNCTION** を見つけ、そのコメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def inspect_and_reject():
        """Publish one event, receive it, inspect the CloudEvent envelope, then reject it."""
        publisher = get_publisher_client()

        # Publish a single test event so there is always something to inspect,
        # regardless of whether the student already acknowledged earlier events.
        test_event = CloudEvent(
            type="com.contoso.ai.ContentFlagged",
            source="/services/content-moderation",
            subject="/content/text/test-inspect",
            data={
                "contentId": "test-inspect",
                "contentType": "text",
                "modelName": "text-moderator-v2",
                "modelVersion": "2.4.0",
                "confidence": 0.76,
                "category": "misinformation",
                "severity": "medium",
                "reviewRequired": True,
                "timestamp": datetime.now(timezone.utc).isoformat()
            },
            id=str(uuid.uuid4())
        )
        publisher.send([test_event])

        # Receive from the sub-flagged subscription to pick up the test event.
        consumer = get_consumer_client(SUB_FLAGGED)
        details = consumer.receive(max_events=1, max_wait_time=10)

        if not details:
            return None

        detail = details[0]
        event = detail.event
        lock_token = detail.broker_properties.lock_token
        delivery_count = detail.broker_properties.delivery_count

        # Capture the full CloudEvent envelope before rejecting.
        result = {
            "specversion": "1.0",
            "type": event.type,
            "source": event.source,
            "subject": event.subject,
            "id": event.id,
            "time": str(event.time) if event.time else "",
            "data": event.data,
            "delivery_count": delivery_count,
            "action": "rejected"
        }

        # reject() tells Event Grid this event cannot be processed. The event
        # is moved to the dead-letter location if configured, or discarded
        # if max delivery count has been reached.
        consumer.reject(lock_tokens=[lock_token])

        return result
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

このセクションでは、完成した Flask アプリケーションを実行してコンテンツ モデレーション イベントを発行し、フィルター処理されたサブスクリプションによってそれらが正しく配信されることを確認します。 このアプリは、CloudEvent 構造全体とプル配信操作を探索するために、イベントの発行、フィルター処理されたサブスクリプションからのイベントの受信と確認、イベントの検査と拒否を行うことができる Web インターフェイスを提供します。

1. ターミナルで次のコマンドを実行して、アプリを起動します。 コマンドを実行する前に、演習の前半のコマンドを参照して、必要に応じて環境をアクティブ化します。 *client* ディレクトリから移動した場合は、まず **cd client** を実行します。

    ```
    python app.py
    ```

1. ブラウザーを開き、`http://localhost:5000` に移動してアプリにアクセスします。

1. 左側のパネルで **[モデレーションイベントの発行]** を選択します。 これにより、5 つのコンテンツ モデレーション イベントが Event Grid 名前空間トピックに発行されます。フラグ付きコンテンツ イベントが 2 つ、承認済みコンテンツ イベントが 2 つ、エスカレーションされたレビューが 1 つです。 右側のパネルの結果を見ると、各イベントがコンテンツ ID、イベントの種類、カテゴリと共に発行されたことを確認できます。

1. 左側のパネルで **[イベントの受信と確認]** を選択します。 これはプル配信を使用して 3 つのすべてのサブスクリプションからイベントを受信し、処理後にそれらを確認します。 各サブスクリプションで構成されているフィルターに基づいて、次の配信動作を確認します。

    - **フラグ付きサブスクリプション:** 2 つのイベントが含まれ、どちらもポリシー違反 (暴力やヘイトスピーチ) を示すカテゴリ値が含まれています。 これらは **ContentFlagged** イベントです。
    - **承認済みサブスクリプション:** 2 つのイベントが含まれ、両方ともカテゴリは **safe** です。 これらは **ContentApproved** イベントです。
    - **すべてのイベント サブスクリプション:** 種類に関わらず 5 つのイベントすべてが含まれ、監査ログとして機能します。

    エスカレーションされたレビュー イベント (**ReviewEscalated**) は、そのイベントの種類がフラグ付きまたは承認済みのサブスクリプションのいずれのフィルターにも含まれていないため、all-events サブスクリプションにのみ表示されます。 イベントが確認されたため、このボタンをもう一度選択しても、追加のイベントを発行するまでイベントは表示されません。

1. 左側のパネルで **[イベントの検査と拒否]** を選択します。 これにより、新しいテスト イベントが発行され、フラグ付きサブスクリプションから受信され、ブローカー プロパティからの **delivery_count** を含む完全な CloudEvent エンベロープが表示され、その後イベントが拒否されます。 拒否により、このイベントは処理できないことが Event Grid に通知されるため、Event Grid はイベントを破棄するか、配信不能の宛先が構成されている場合はそこに移動します。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループと、そのグループ内のすべてのリソースを削除します。 **\<rg-name>** は、この演習で選択した名前に置き換えてください。 このコマンドを実行すると Azure の中でバックグラウンド タスクが起動されてリソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の実行中に問題が発生した場合は、次のトラブルシューティング手順をお試しください。

**Event Grid 名前空間のデプロイを確認する**
- [Azure portal](https://portal.azure.com) に移動して、リソース グループを見つけます。
- Event Grid 名前空間の **[プロビジョニングの状態]** が **Succeeded** であることを確認します。
- 名前空間トピック **moderation-events** が名前空間内に存在することを確認します。

**イベント サブスクリプションを確認する**
- デプロイ スクリプトの状態チェック (オプション 5) を実行して、3 つのイベント サブスクリプションすべてが作成されていることを確認します。
- サブスクリプション (**sub-flaagged**、**sub-approved**、**sub-all-events**) の状態が **Succeeded** であることを確認します。
- 発行後にイベントが受信されない場合は、トピックの後にサブスクリプションが作成されたかどうかを確認します。 必要に応じてデプロイ スクリプトのオプション 2 を再実行します。

**コードの完全性とインデントを確認する**
- すべてのコード ブロックが、*event_grid_functions.py* 内の適切な BEGIN/END コメント マーカーの間にある正しいセクションに追加されていることを確認します。
- Python のインデントが一貫していること (タブではなくスペースを使用していること)、およびすべてのコードが関数内で正しく配置されていることを確認します。
- 指定されたセクションの外部でコードが誤って削除または変更されていないことを確認します。

**環境変数を確認する**
- *.env*ファイルがプロジェクトのルートに存在し、**EVENTGRID_ENDPOINT**、**EVENTGRID_TOPIC_NAME**、**RESOURCE_GROUP**、**NAMESPACE_NAME**の値が含まれていることを確認します。
- **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行して環境変数をターミナル セッションに読み込んだことを確認します。
- 変数が空の場合は、**source .env** (Bash) または **. .\.env.ps1** (PowerShell) をもう一度実行します。

**認証を確認する**
- **az account show** を実行して、Azure CLI にログインしていることを確認します。
- 名前空間に EventGrid Data Sender ロールと EventGrid Data Receiver ロールが割り当てられていることを確認します。 必要に応じて、デプロイ スクリプトのロールの割り当てオプション (オプション 3) をもう一度実行します。

**Python 環境と依存関係を確認する**
- アプリを実行する前に、仮想環境がアクティブになっていることを確認します。
- **pip list** を実行して、*requirements.txt* のすべてのパッケージが正常にインストールされたことを確認します。
