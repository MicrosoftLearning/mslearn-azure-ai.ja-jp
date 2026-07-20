---
lab:
  topic: Azure Managed Redis
  title: Azure Managed Redis でイベントを発行してサブスクライブする
  description: redis-py Python ライブラリと Microsoft Entra ID を使用して、Azure Managed Redis で pub/sub メッセージング パターンを実装する Flask Web アプリを構築する方法について学びます。
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Managed Redis
---

# Azure Managed Redis でイベントを発行してサブスクライブする

この演習では、Azure Managed Redis リソースをデプロイし、単一ページから Redis チャネルを発行して購読する Python Flask Web アプリを完成させます。 Microsoft Entra ID を使用して Redis に接続するコードの追加、イベント メッセージの発行、すべてのチャネルへのブロードキャスト、受信済みメッセージの書式設定、バックグラウンド スレッドでのメッセージのリッスンを行い、チャネルとパターンを購読します。 その後、アプリを起動して、発行したメッセージがリアルタイムで届くことを確認します。

この演習で実行されるタスク:

- プロジェクトのスターター ファイルをダウンロードする
- Azure Managed Redis リソースを作成する
- スターター ファイルにコードを追加してアプリを完成させる
- アプリを実行してメッセージを発行および購読する

この演習の所要時間は約 **30** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- [Python 3.12](https://www.python.org/downloads/) 以上。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- Azure CLI **redisenterprise** 拡張機能、バージョン 2.75.0 以上。 後のステップでこの拡張機能のインストールやアップグレードが行われます。

## プロジェクト スターター ファイルをダウンロードして Azure Managed Redis をデプロイする

このセクションでは、アプリのスターターファイルをダウンロードし、スクリプトを使って Azure Managed Redis のサブスクリプションへのデプロイを初期化します。 Azure Managed Redis のデプロイは完了までに 5 から 10 分かかるため、まずデプロイを開始し、プロビジョニング中にアプリにコードを追加します。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/amr-pub-sub-python.zip
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

1. 次のコマンドを実行して、Azure Managed Redis に必要なリソース プロバイダーがご自分のサブスクリプションにあることを確認します。

    ```
    az provider register --namespace Microsoft.Cache
    ```

1. 次のコマンドを実行して、Azure CLI 用の **redisenterprise** 拡張機能をインストールまたはアップグレードします。 データベースで Microsoft Entra ID アクセスを構成するには、バージョン 2.75.0 以上が必要です。

    ```
    az extension add --upgrade --name redisenterprise
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

1. スクリプトの実行中に、「**1**」と入力して **1. Create Azure Managed Redis resource** オプションを起動します。

    このオプションは、リソースグループがまだ存在していなければ作成して、Azure Managed Redis をデプロイします。 スクリプトは、5 から 10 分間かかるデプロイの完了を待機し、ターミナルにその結果を報告します。 スクリプトを実行したまま、次のセクションに進み、デプロイのプロビジョニング中にコードを追加します。 定期的に端末を確認してエラーをチェックします。

    デプロイが成功すると、次のような確認メッセージが表示され、メニューが返されます。

    Azure Managed Redis リソースが正常に作成されました: amr-exercise-\<hash>**

    > **注:** デプロイが失敗した場合、多くの場合は選択したリージョンの SKU の容量が一時的に不足していることが原因です。 画面上の指示に従ってスクリプトを終了し、スクリプト上部近くの **location** 変数を eastus2、australiaeast、canadacentral など別の地域に変更し、もう一度スクリプトを実行してオプション 1 を選択します。 失敗したリソースは次の試行の前に自動的に削除されます。

## アプリの仕上げ

このセクションでは、pub/sub 関数を完了するためのコードを *pubsub_functions.py* ファイルに追加します。 *app.py* の Flask アプリはこれらの機能を呼び出して、メッセージを発行し、サブスクリプションを管理し、受信済みメッセージをブラウザーにストリーミングします。 *app.py* を編集する必要はありません。 このアプリは、演習の後半で実行します。

1. *client/pubsub_functions.py* ファイルを開き、コードの追加を開始します。

>**注:** アプリケーションに追加するコード ブロックは、コードのそのセクションのコメントと一致する必要があります。

### Azure Managed Redis に接続するコードを追加する

このセクションでは、Microsoft Entra ID で認証する Redis クライアントを作成するコードを追加します。 Entra ID を使うと、アプリではアクセス キーが処理されません。

**get_client()** 関数は **REDIS_HOST** 環境変数から Redis エンドポイントを読み取り、**create_from_default_azure_credential()** を呼び出して資格情報プロバイダーを構築します。 プロバイダーは **DefaultAzureCredential** を使用して Microsoft Entra トークンを取得し、これをバックグラウンドで自動的に更新するため、長時間維持されるリスナー接続が認証されたままになります。 クライアントは TLS 経由でポート 10000 に接続し、文字列への応答をデコードします。

1. **# BEGIN CONNECTION CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def get_client() -> redis.Redis:
        """Create a Redis client for Azure Managed Redis using Microsoft Entra ID."""
        redis_host = os.environ.get("REDIS_HOST")

        if not redis_host:
            raise ValueError("REDIS_HOST environment variable must be set")

        # create_from_default_azure_credential uses DefaultAzureCredential to
        # acquire a Microsoft Entra token for Redis. The credential provider
        # refreshes the token automatically in the background so long-lived
        # connections (like the pub/sub listener) stay authenticated.
        credential_provider = create_from_default_azure_credential(
            ("https://redis.azure.com/.default",),
        )

        return redis.Redis(
            host=redis_host,
            port=10000,
            ssl=True,
            decode_responses=True,
            credential_provider=credential_provider,
            socket_timeout=30,
            socket_connect_timeout=30,
        )
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### イベントを発行するコードを追加する

このセクションでは、注文生成イベントを発行するコードを追加します。 これは、メッセージを単一のチャネルに送信する中心的な発行操作を示しています。

**publish_order_created()** 関数は、イベントを記述するディクショナリを作成し、それを JSON にシリアル化し、**orders:created** チャネルで **publish()** を呼び出します。 **publish()** メソッドはメッセージを受け取ったサブスクライバー数を返し、アプリでそれが表示され、メッセージが配信されたことを確認できるようにします。

1. **# BEGIN PUBLISH MESSAGE CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def publish_order_created(r: redis.Redis) -> dict:
        """Publish an order created event to the 'orders:created' channel."""
        order_data = {
            "event": "order_created",
            "order_id": f"ORD-{datetime.now().strftime('%Y%m%d%H%M%S')}",
            "customer": "Jane Doe",
            "total": 129.99,
            "timestamp": datetime.now().isoformat(),
        }
        channel = "orders:created"

        # publish() sends the message to every subscriber of the channel and
        # returns the number of subscribers that received it.
        subscribers = r.publish(channel, json.dumps(order_data))

        return {"channel": channel, "subscribers": subscribers, "message": order_data}
    ```

    > **注:** スターター ファイルには、**publish_order_shipped()**、**publish_inventory_alert()**、**publish_notification()** 関数が既に含まれているため、扱うことができるイベントの種類がいくつかあります。 少し時間を取って、これらを確認してみてください。

1. 変更を保存し、少し時間を取ってコードをレビューします。

### すべてのチャンネルにブロードキャストするコードを追加する

このセクションでは、すべてのチャンネルに単一のメッセージをブロードキャストするコードを追加します。 ブロードキャストは、チャンネルに関係なくすべての加入者が受け取る必要があるシステム全体のアナウンスに適しています。

**broadcast_to_all()** 関数は、**AVAILABLE_CHANNELS** をループし、それぞれに対して同じメッセージで **publish()** を呼び出します。 すべてのチャンネルのサブスクライバー数を合計してアプリがサブスクライバー数を報告できるようにします。

1. **# BEGIN BROADCAST CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def broadcast_to_all(r: redis.Redis) -> dict:
        """Broadcast the same message to every channel using publish() in a loop."""
        announcement = {
            "event": "system_announcement",
            "message": "System maintenance scheduled for 2 AM",
            "priority": "high",
            "timestamp": datetime.now().isoformat(),
        }
        message = json.dumps(announcement)

        results = []
        total_subscribers = 0
        for channel in AVAILABLE_CHANNELS:
            # Send the same message to multiple channels for multi-channel delivery.
            count = r.publish(channel, message)
            total_subscribers += count
            results.append({"channel": channel, "subscribers": count})

        return {
            "channels": results,
            "total_subscribers": total_subscribers,
            "message": announcement,
        }
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### 受信したメッセージをフォーマットするコードを追加する

このセクションでは、受信メッセージを表示するために書式設定するコードを追加します。 バックグラウンド リスナーは、受信するすべてのメッセージに対してこの関数を呼び出して、Web ページにクリーンな概要を表示できるようにします。

**format_message()** 関数は、未加工の pub/sub メッセージからチャネル、パターン、データを読み取ります。 これは、JSON ペイロードを解析し、イベントの種類とともに **order_id** や **customer** などの一連の既知のフィールドを抽出します。 ペイロードが有効な JSON でない場合は、代わりに生の値が返されるため、何も失われません。

1. **# BEGIN MESSAGE FORMATTING CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def format_message(message: dict) -> dict:
        """Parse a pub/sub message and extract relevant fields for display."""
        timestamp = datetime.now().strftime("%H:%M:%S")
        channel = message.get("channel", "unknown")
        pattern = message.get("pattern")

        try:
            data = json.loads(message["data"])
        except (json.JSONDecodeError, TypeError):
            # Non-JSON payloads are returned as-is under a "raw" key.
            return {
                "timestamp": timestamp,
                "channel": channel,
                "pattern": pattern,
                "event": None,
                "details": {"raw": message.get("data")},
            }

        # Pull out the fields that the demo events include so the UI can
        # display a clean summary of each message.
        field_names = [
            "order_id", "customer", "total", "tracking_number",
            "product_name", "current_stock", "message",
        ]
        details = {name: data[name] for name in field_names if name in data}

        return {
            "timestamp": timestamp,
            "channel": channel,
            "pattern": pattern,
            "event": data.get("event", "unknown"),
            "details": details,
        }
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### メッセージをリッスンするコードを追加する

このセクションでは、**PubSubManager** クラスの **listen_messages()** メソッドにコードを追加します。 このメソッドはバックグラウンド スレッドで実行されるため、アプリは Web 要求に応答しながらメッセージを継続的に受信できます。

メソッドは **pubsub.listen()** を反復処理し、発行時にメッセージをブロックして生成します。 これは、**message** (ダイレクト チャネル) と **pmessage** (パターン) タイプの両方を扱い、それぞれを **format_message()** でフォーマットし、Web ページがポーリングするスレッドセーフ バッファーに追加します。 エラーはキャプチャされ、システム メッセージとして表示されるため、UI にエラーが表示されます。

1. **# BEGIN MESSAGE LISTENER CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def listen_messages(self) -> None:
        """Background thread that reads messages from subscribed channels."""
        self.listener_active = True
        try:
            # listen() blocks and yields messages as they are published.
            for message in self.pubsub.listen():
                if not self.listening:
                    break

                # Handle both direct channel messages and pattern messages.
                if message["type"] in ("message", "pmessage"):
                    self._add_message(format_message(message))

        except Exception as e:
            if self.listening:
                self._add_message({
                    "timestamp": datetime.now().strftime("%H:%M:%S"),
                    "channel": "system",
                    "pattern": None,
                    "event": "listener_error",
                    "details": {"error": str(e)},
                })
        finally:
            self.listener_active = False
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### チャネルとパターンを購読するコードを追加する

このセクションでは、**subscribe_to_channel()** および **subscribe_to_pattern()** メソッドにコードを追加します。 これらは、Redis pub/sub の 2 つの主なサブスクリプション戦略です。特定のイベント用のダイレクト チャネル サブスクリプションとワイルドカード照合用のパターン サブスクリプションです。

**subscribe_to_channel()** メソッドは **subscribe()** を呼び出して 1 つのチャネルに関心を登録します。一方、**subscribe_to_pattern()** は、**orders:*** などのパターンを持つ複数のチャネルと一致するように **psubscribe()** を呼び出します。 各サブスクリプションが変更されるたびに、コードによりリスナーが再起動され、新しいチャネルでメッセージの受信が開始されます。

1. **# BEGIN SUBSCRIBE CHANNEL/PATTERN CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def subscribe_to_channel(self, channel: str) -> str:
        """Subscribe to a specific channel using subscribe()."""
        self.pubsub.subscribe(channel)  # Register interest in the channel.
        self.restart_listener()
        return f"Subscribed to channel: {channel}"

    def subscribe_to_pattern(self, pattern: str) -> str:
        """Subscribe using a pattern with psubscribe() (e.g. 'orders:*')."""
        self.pubsub.psubscribe(pattern)  # Register interest in matching channels.
        self.restart_listener()
        return f"Subscribed to pattern: {pattern}"
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

## リソースのデプロイを確認する

このセクションでは、実行中のデプロイ スクリプトに戻り、Azure Managed Redis のデプロイが完了したことを確認し、データベースを作成し、Microsoft Entra ID アクセスを構成し、エンドポイントで *.env* ファイルを生成します。

1. デプロイ スクリプトが実行中のターミナルに戻ります。 デプロイが成功すると、確認メッセージとメニューが表示されます。 スクリプトを終了した場合は、適切なコマンドを実行してもう一度起動します。

    **Bash**
    ```bash
    bash azdeploy.sh
    ```

    **PowerShell**
    ```powershell
    ./azdeploy.ps1
    ```

1. **2. Create database and configure access** (データベースを作成してアクセスを構成する) オプションを実行するには、「**2**」を入力します。 これにより Microsoft Entra ID 認証付きのデータベースが作成され、アカウントにデータアクセス ポリシーが割り当てられてアプリがあなたの ID で接続できるようになり、**REDIS_HOST** エンドポイントで *.env* ファイルが作成されます。

1. (省略可能) 「**3**」を入力して、**3 Check deployment status** (デプロイの状態をチェックする) オプションを最後のチェックとして確認します。

1. 「**4**」を入力して、デプロイ スクリプトを終了します。

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

このセクションでは、完成した Flask アプリケーションを実行し、単一の Web ページからメッセージを発行および購読します。 左パネルはイベントの発行とサブスクリプションの管理を行い、右パネルは直近の発行結果と受信したメッセージのライブ ストリームを表示します。

1. ターミナルで次のコマンドを実行して、アプリを起動します。 コマンドを実行する前に、演習の前半のコマンドを参照して、必要に応じて環境をアクティブ化し環境変数を読み込みます。 *client* ディレクトリから移動した場合は、まず **cd client** を実行します。

    ```
    python app.py
    ```

1. ブラウザーを開き、`http://localhost:5000` に移動してアプリにアクセスします。

1. 左側パネルの **[サブスクリプション]** エリアで、チャンネル ボックスを選択してドロップダウン リストを開き、**[通知]** を選択して **[購読]** を選択します。 成功のメッセージによりサブスクリプションが確認され、**[アクティブなサブスクリプション]** リストが更新されて、そのチャネルが含まれます。 チャネルのメッセージを受け取るには、まずチャネルを登録する必要があります。

1. **[イベント発行]** エリアで **[通知]** を選択します。 右のパネルは、チャンネルとサブスクライバーの数を含む発行の結果を示しています。 1、2 秒以内に、バックグラウンド リスナーからサブスクリプションに送られたメッセージが**受信済みメッセージ** リストに表示されます。

1. **[在庫アラート]** を選択します。 発行の結果ではメッセージが **inventory:alerts** チャネルに送信されたことが示されていますが、**受信済みメッセージ**には表示されません。これは、**[通知]** だけを購読したためです。

1. **[サブスクリプション]** 領域でパターン ボックスに「**orders:\***」と入力し、**[パターンを購読する]** を選択します。 これにより **orders:** で始まるすべてのチャンネルが購読され、そのパターンと**通知**が**アクティブなサブスクリプション** リストに表示されます。

1. **[作成済みの注文]** を選択し、次に **[発送済みの注文]** を選択します。 両方のメッセージが **受信済みメッセージ** リストに表示されます。**orders:***パターンが **orders:created** および **orders:shipped** チャネルの両方に一致しているためです。

1. **[すべてにブロードキャスト]** を選択して、すべてのチャネルに単一のアナウンスを送信します。 現在のサブスクリプションに一致するメッセージが届くたびに**受信済みメッセージ** リストがリアルタイムで更新されるようすをご覧ください。

1. **[すべての購読を中止]** を選択して購読を解除し、もう一度 **[すべてにブロードキャスト]** を選択します。 現在、アクティブなサブスクリプションがまったくないため、**受信済みメッセージ**に新しいメッセージが届いていないことを確認します。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループと、そのグループ内のすべてのリソースを削除します。 **\<rg-name>** は、この演習で選択した名前に置き換えてください。 このコマンドを実行すると Azure の中でバックグラウンド タスクが起動されてリソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の実行中に問題が発生した場合は、次のトラブルシューティング手順をお試しください。

**Azure Managed Redis リソースのデプロイを確認する**
- [Azure portal](https://portal.azure.com) に移動してリソース グループを見つけます。
- Azure Managed Redis リソースの **[プロビジョニングの状態]** の表示が **[成功]** であることを確認します。
- デプロイ スクリプトの **Check deployment status** オプションを実行し、クラスターとデータベースの準備ができていることを確認してから、データベースを作成しアクセスを構成します。

**認証とアクセスをチェックする**
- **az account show** を実行して、Azure CLI にログインしていることを確認します。
- デプロイ スクリプトの **Create database and configure access** オプションが正常に完了し、アカウントにデータベースに対するデータ アクセス ポリシーが適用されていることを確認します。
- アプリで認証エラーが報告された場合は、アクセス ポリシーの割り当てが有効になるまでに少し時間がかかる可能性があるため、しばらく待ってからもう一度試してみてください。

**コードの完全性とインデントを確認する**
- すべてのコード ブロックが、*pubsub_functions.py* 内の適切な BEGIN/END コメント マーカーの間にある正しいセクションに追加されていることを確認します。
- Python のインデントが一貫している (タブではなくスペースを使用している) ことを確認します。 **listen_messages()**、**subscribe_to_channel()**、**subscribe_to_pattern()** メソッドは **PubSubManager** クラス内にあるため、コードは 1 レベルだけインデントする必要があります。
- 指定されたセクションの外部でコードが誤って削除または変更されていないことを確認します。

**環境変数を確認する**
- *.env* ファイルがプロジェクトのルートに存在し、**REDIS_HOST** 値が含まれていることを確認します。
- **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行して環境変数をターミナル セッションに読み込んだことを確認します。
- 変数が空の場合は、**source .env** (Bash) または **. .\.env.ps1** (PowerShell) をもう一度実行します。

**メッセージが表示されない**
- 発行するチャンネルを購読したことを確認します。 メッセージは、購読しているチャンネルやパターンにのみ届きます。
- ブラウザーで **[アクティブなサブスクリプション]** リストをチェックして、現在のサブスクリプションを確認します。
- 端末でアプリがまだ実行中であることと、ページにメッセージ ストリームの更新が表示されていることを確認します。

**Python 環境と依存関係を確認する**
- アプリを実行する前に、仮想環境がアクティブになっていることを確認します。
- **pip list** を実行して、*requirements.txt* のすべてのパッケージが正常にインストールされたことを確認します。
