---
lab:
  topic: Instrument and observe apps
  title: OpenTelemetry SDK を使用してアプリをインストルメント化する
  description: OpenTelemetry を使用してアプリケーションをインストルメント化する方法、カスタム スパンと属性の作成方法、テレメトリを Application Insights にエクスポートする方法、トランザクション検索やログ クエリを使用してパフォーマンスの問題を診断する方法を学習します。
  level: 300
  duration: 25
  islab: true
---

# OpenTelemetry を使ってアプリをインストルメント化する

OpenTelemetry は、オープンソースの監視フレームワークであり、アプリケーションからトレース、メトリック、ログを収集するための標準化された方法を提供します。 Azure Monitor OpenTelemetry Distro では、OpenTelemetry SDK と Azure Monitor エクスポーターがパッケージ化されるため、Python アプリケーションは最小限の設定でテレメトリを Application Insights に送信できます。 カスタム スパンを使用すると、アプリケーション固有の操作をトレースし、トレース データをビジネス コンテキストでエンリッチする属性を追加できます。

この演習では、Application Insights リソースをデプロイし、ドキュメント処理パイプライン向けの OpenTelemetry インストルメンテーションを示す Python Flask Web アプリケーションをビルドします。 Azure Monitor OpenTelemetry Distro を構成し、パイプライン ステージごとにカスタムの親および子スパンを作成し、スパン属性を追加してドキュメント メタデータをキャプチャし、Azure portal でトランザクション検索とログ クエリを使用してテレメトリを検証し、シミュレートされた待機時間のボトルネックを診断します。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードする
- Application Insights リソースを作成する
- スターター ファイルにコードを追加してアプリを完成させる
- アプリを実行し、Application Insights でパフォーマンスの問題を診断する

この演習の所要時間は約 **25** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- [Python 3.12](https://www.python.org/downloads/) 以上。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。

## プロジェクトのスターター ファイルをダウンロードし、Application Insights をデプロイする

このセクションでは、アプリのスターター ファイルをダウンロードし、スクリプトを使用して Application Insights リソースをサブスクリプションにデプロイします。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/instrument-app-python.zip
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

1. 次のコマンドを実行して、演習に必要なリソース プロバイダーがご自分のサブスクリプションにあることを確認します。

    ```
    az provider register --namespace Microsoft.Insights
    az provider register --namespace Microsoft.OperationalInsights
    ```

1. 次のコマンドを実行して、Application Insights CLI 拡張機能を追加します。 この拡張機能は、デプロイ スクリプトで Application Insights リソースを作成して管理するために使用するコマンドを提供します。

    ```
    az extension add --name application-insights
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

1. スクリプトの実行中、「**1**」と入力して、**[1. Application Insights を作成する]** オプションを起動します。

    このオプションは、リソース グループがまだ存在していない場合にそれを作成し、Application Insights リソースを作成します。

1. 「**2**」と入力して、**[2. ロールを割り当てる]** オプションを実行します。 これにより、アカウントに Monitoring Metrics Publisher ロールが割り当てられるため、アプリで Microsoft Entra 認証を使用して Application Insights にテレメトリを発行できるようになります。

1. 「**3**」と入力して、**[3. デプロイ状態を確認する]** オプションを実行します。 続行する前に、Application Insights リソースに **Succeeded** と表示され、ロールが割り当てられていることを確認します。 リソースのプロビジョニングがまだ完了していない場合は、しばらく待ってからもう一度試してください。

1. 「**4**」と入力して、**[4. 接続情報を取得する]** オプションを実行します。 これにより、Application Insights 接続文字列と、アプリに必要な **OTEL_SERVICE_NAME** 変数を含む環境変数ファイルが作成されます。

1. 「**5**」と入力して、デプロイ スクリプトを終了します。

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

このセクションでは、*telemetry_functions.py* ファイルにコードを追加して、OpenTelemetry インストルメンテーション関数を完成します。 *app.py* 内の Flask アプリは、これらの関数を呼び出し、結果をブラウザーに表示します。 このアプリは、演習の後半で実行します。

1. *client/telemetry_functions.py* ファイルを開き、コードの追加を開始します。

>**注:** アプリケーションに追加するコード ブロックは、コードのそのセクションのコメントと一致する必要があります。

### テレメトリを構成するコードを追加する

このセクションでは、Azure Monitor OpenTelemetry Distro を構成するコードを追加して、アプリケーションでトレースを Application Insights にエクスポートするようにします。 この関数は環境変数から接続文字列を読み取り、Microsoft Entra 認証用の **DefaultAzureCredential** を作成し、Azure Monitor エクスポーターを構成します。 アプリがローカルで実行されるため、この資格情報では、マネージド ID プロバイダーは除外されます。この設定がない場合、資格情報チェーンはテレメトリのエクスポートごとに Azure Instance Metadata Service にアクセスしようとし、失敗した HTTP 呼び出しがノイズとしてアプリケーション マップに表示されます。

この関数は、Azure Monitor OpenTelemetry Distro パッケージから **configure_azure_monitor()** を呼び出します。 この 1 回の呼び出しで、Azure Monitor トレース エクスポーターと共に OpenTelemetry SDK が構成され、Flask 要求の自動インストルメンテーションが設定されます。 **credential** パラメーターにより、Entra ベースの認証が有効になるため、アプリでは、インストルメンテーション キーではなく Monitoring Metrics Publisher ロールを使用してテレメトリを発行します。 **OTEL_SERVICE_NAME** 環境変数は、デプロイ スクリプトによって *.env* ファイル内に設定され、アプリケーション マップに表示される **cloud.role.name** を制御します。

1. コメント **# BEGIN CONFIGURE TELEMETRY FUNCTION** を見つけ、そのコメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def configure_telemetry():
        """Configure the Azure Monitor OpenTelemetry Distro."""
        connection_string = os.environ.get("APPLICATIONINSIGHTS_CONNECTION_STRING")

        if not connection_string:
            raise ValueError(
                "APPLICATIONINSIGHTS_CONNECTION_STRING environment variable must be set"
            )

        from azure.monitor.opentelemetry import configure_azure_monitor

        credential = DefaultAzureCredential(
            exclude_managed_identity_credential=True
        )

        configure_azure_monitor(
            connection_string=connection_string,
            credential=credential,
        )
    ```

1. 少し時間をかけて  コードを確認しましょう。

### ドキュメントを処理するコードを追加する

このセクションでは、バッチ ドキュメント処理操作の親スパンを作成するコードを追加します。 この関数は、構成可能な数のドキュメントをループ処理し、それぞれのドキュメントに対して、3 つの子スパン関数 (検証、エンリッチ、格納) を呼び出します。

この関数は、**start_as_current_span()** を使用して "process-documents" という名前の親スパンを作成し、バッチ全体をラップします。 各子関数は独自のスパンを作成し、それが自動的に現在のスパンの子となり、階層トレース ツリーが構築されます。 span 属性は、バッチ サイズと正常に処理されたドキュメントの数を記録します。

1. コメント **# BEGIN PROCESS DOCUMENTS FUNCTION** を見つけ、そのコメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def process_documents(doc_count):
        """Process a batch of documents through the pipeline with tracing."""
        tracer = get_tracer()
        results = []

        with tracer.start_as_current_span("process-documents") as parent_span:
            parent_span.set_attribute("document.count", doc_count)
            parent_span.set_attribute("pipeline.name", "document-processing")

            for i in range(1, doc_count + 1):
                doc_id = f"DOC-{i:04d}"

                validate_result = validate_document(doc_id)
                enrich_result = enrich_document(doc_id)
                store_result = store_document(doc_id)

                results.append({
                    "doc_id": doc_id,
                    "validate": validate_result,
                    "enrich": enrich_result,
                    "store": store_result
                })

            parent_span.set_attribute("document.processed", len(results))

        return results
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### パイプライン ステージをトレースするコードを追加する

このセクションでは、ドキュメント パイプラインの各ステージ (検証、エンリッチ、格納) ごとに子スパンを作成する 3 つの関数を追加します。 これら 3 つはすべて同じパターンに従います。**start_as_current_span()** を呼び出して自動的にアクティブな親の子となるスパンを作成し、**set_attribute()** を呼び出して検索可能なメタデータを付加し、**set_status()** を呼び出して結果をマークします。

**enrich_document** 関数には、意図的に待機時間を発生させる問題も含まれています。 ドキュメント **DOC-0003** および **DOC-0005** では 1.5 秒から 3 秒の遅延が発生し、外部サービスのボトルネックをシミュレートします。 **enrichment.slow** 属性は、影響を受けたスパンにフラグを付けて、Application Insights でそれらをフィルター処理できるようにします。 後でエンド ツー エンドのトランザクション ビューを調べると、これらのスパンがパイプラインの遅延の原因として際立ちます。

1. コメント **# BEGIN PIPELINE STAGE FUNCTIONS** を見つけ、そのコメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def validate_document(doc_id):
        """Validate a document and record a traced span."""
        tracer = get_tracer()

        with tracer.start_as_current_span("validate-document") as span:
            span.set_attribute("document.id", doc_id)
            span.set_attribute("document.stage", "validate")

            # Simulate validation work
            time.sleep(random.uniform(0.05, 0.15))
            is_valid = True

            span.set_attribute("document.valid", is_valid)
            span.set_status(StatusCode.OK)

        return {"status": "valid", "duration_ms": round(random.uniform(50, 150))}


    def enrich_document(doc_id):
        """Enrich a document with metadata and record a traced span."""
        tracer = get_tracer()

        with tracer.start_as_current_span("enrich-document") as span:
            span.set_attribute("document.id", doc_id)
            span.set_attribute("document.stage", "enrich")

            # Simulated latency issue: documents DOC-0003 and DOC-0005
            # experience high latency during enrichment, representing
            # a bottleneck for the student to diagnose in Application Insights
            if doc_id in ("DOC-0003", "DOC-0005"):
                delay = random.uniform(1.5, 3.0)
                span.set_attribute("enrichment.slow", True)
            else:
                delay = random.uniform(0.05, 0.2)
                span.set_attribute("enrichment.slow", False)

            time.sleep(delay)
            span.set_attribute("enrichment.duration_s", round(delay, 3))
            span.set_status(StatusCode.OK)

        return {
            "status": "enriched",
            "duration_ms": round(delay * 1000),
            "slow": doc_id in ("DOC-0003", "DOC-0005")
        }


    def store_document(doc_id):
        """Store a document and record a traced span."""
        tracer = get_tracer()

        with tracer.start_as_current_span("store-document") as span:
            span.set_attribute("document.id", doc_id)
            span.set_attribute("document.stage", "store")
            span.set_attribute("storage.type", "blob")

            # Simulate storage write
            time.sleep(random.uniform(0.05, 0.2))

            span.set_status(StatusCode.OK)

        return {"status": "stored", "duration_ms": round(random.uniform(50, 200))}
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

このセクションでは、完成した Flask アプリケーションを実行してテレメトリを生成し、その後 Azure portal でトランザクション検索とログ クエリを使用してスパンを検証し、シミュレーションされたパフォーマンス ボトルネックを診断します。

1. ターミナルで次のコマンドを実行して、アプリを起動します。 コマンドを実行する前に、演習の前半のコマンドを参照して、必要に応じて環境をアクティブ化します。 *client* ディレクトリから移動した場合は、まず **cd client** を実行します。

    ```
    python app.py
    ```

1. ブラウザーを開き、`http://localhost:5000` に移動してアプリにアクセスします。

1. 左側のパネルで、**[テレメトリの状態の確認]** を選択します。 テレメトリの状態が **active** であり、リソース属性に **service.name** が含まれており、その値が **document-pipeline-app** であることを確認します。 これにより、Azure Monitor OpenTelemetry Distro が構成され、テレメトリがエクスポートされていることが確認できます。

1. 左側のパネルで、**[ドキュメントの処理]** を選択します。 これにより、パイプラインを介して 5 つのドキュメントが処理され、その結果がテーブルに表示されます。 他のドキュメントは短時間で完了しますが、ドキュメント **DOC-0003** と **DOC-0005** は、エンリッチメント期間が大幅に長く、**SLOW** タグが表示されていることに注意してください。

1. **[ドキュメントの処理]** をさらに 2 回選択して、追加のテレメトリ データを生成します。 実行ごとに、親スパンと子スパンを持つ新しいトレースが作成されます。

1. テレメトリが Application Insights に到着するまで 2 から 3 分待ちます。 テレメトリはバッチ処理され、定期的に送信されるため、ポータルにデータが表示されるまでに短い遅延があります。

1. [Azure portal](https://portal.azure.com) に移動し、先ほど作成したリソース グループ内で Application Insights リソースを見つけます。

### エンド ツー エンドのトランザクションを表示する

1. Application Insights リソースで、左側のナビゲーションの **[調査]** の下にある **[検索]** を選択します。 このビューには、すべての着信要求とそれに関連するテレメトリが一覧表示されます。

1. 結果リストで、**POST /process-documents** エントリの 1 つを見つけて選択します。 エンド ツー エンドのトランザクションのビューが開き、ルート HTTP 要求スパン、"process-documents" 親スパン、各パイプライン ステージ (検証、強化、保存) の子スパンのすべてのスパンが表示されます。

1. トランザクション タイムラインで、期間が 1.5 秒以上の "enrich-document" スパンを特定します。 これらは、シミュレートされた待機時間を示すドキュメント **DOC-0003** および **DOC-0005** のスパンです。 これらのスパンのいずれかを選択すると、**document.id**、**document.stage**、**enrichment.slow = True** などの属性が表示されます。

### KQL を使用してテレメトリのクエリを実行する

1. Application Insights リソースで、左側のナビゲーションの **[監視]** の下にある **[ログ]** を選択します。 表示されたクエリ テンプレート ダイアログを閉じます。 **注意:** クエリ バーのドロップダウン セレクターで、必ず **[KQL モード]** を選択してください。

1. 次のクエリをコピーしてクエリ エディターに貼り付け、**[実行]** を選択します。 このクエリは、コードで作成されたカスタム スパンと設定したスパン属性を取得します。

    ```kusto
    dependencies
    | where timestamp > ago(1h)
    | project timestamp, name, duration,
        documentId = customDimensions["document.id"],
        stage = customDimensions["document.stage"],
        slow = customDimensions["enrichment.slow"]
    | order by timestamp desc
    ```

1. 結果を確認します。 各パイプライン ステージ (validate-document、enrich-document、store-document) ごとに、コードに追加した **documentId** 属性と **stage** 属性を含む行が表示されます。 DOC-0003 および DOC-0005 の行の **slow** 列には **True** が表示されます。

1. 次のクエリをコピーして貼り付け、低速と高速のエンリッチメント スパンの平均期間を比較します。

    ```kusto
    dependencies
    | where timestamp > ago(1h) and name == "enrich-document"
    | extend slow = tostring(customDimensions["enrichment.slow"])
    | summarize avgDuration = round(avg(duration), 0) by slow
    ```

1. 結果を確認します。 **True** 行は、平均期間が 1,500 ミリ秒以上を示し、**False** 行は平均が 200 ミリ秒未満を示します。 これにより、これは、エンリッチメント ステージがボトルネックであり、スパン属性によって、影響を受けるドキュメントが明確に特定されることを確認できます。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループと、そのグループ内のすべてのリソースを削除します。 **\<rg-name>** は、この演習で選択した名前に置き換えてください。 このコマンドを実行すると Azure の中でバックグラウンド タスクが起動されてリソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の実行中に問題が発生した場合は、次のトラブルシューティング手順を試してください。

**Application Insights のデプロイを検証する**
- [Azure portal](https://portal.azure.com) に移動して、リソース グループを見つけます。
- Application Insights リソースの **[プロビジョニングの状態]** に **Succeeded** が表示されていることを確認します。

**接続文字列を確認します**
- デプロイ スクリプトの **[デプロイ状態の確認]** オプションを実行して、リソースが正常に作成されたことを確認します。
- *.env* ファイルに **APPLICATIONINSIGHTS_CONNECTION_STRING** の値が含まれていることを確認します。
- 接続文字列がない場合は、**[接続情報の取得]** オプションをもう一度実行します。

**コードの完全性とインデントを確認する**
- すべてのコード ブロックが、*telemetry_functions.py* 内の適切な BEGIN/END コメント マーカーの間にある正しいセクションに追加されていることを確認します。
- Python のインデントが一貫していること (タブではなくスペースを使用していること)、およびすべてのコードが関数内で正しく配置されていることを確認します。
- 指定されたセクションの外部でコードが誤って削除または変更されていないことを確認します。

**環境変数を検証する**
- *.env* ファイルがプロジェクトのルートに存在し、**APPLICATIONINSIGHTS_CONNECTION_STRING**の値が含まれていることを確認します。
- **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行して環境変数をターミナル セッションに読み込んだことを確認します。
- 変数が空の場合は、**source .env** (Bash) または **. .\.env.ps1** (PowerShell) をもう一度実行します。

**認証を確認する**
- **az account show** を実行して、Azure CLI にログインしていることを確認します。
- Azure portal でロールの割り当てを確認するか、デプロイ スクリプトのオプションを実行してロールをもう一度割り当てて、Metrics Publisher ロールがアカウントに割り当てられていることを確認します。

**Python 環境と依存関係を確認する**
- アプリを実行する前に、仮想環境がアクティブになっていることを確認します。
- **pip list** を実行して、*requirements.txt* のすべてのパッケージが正常にインストールされたことを確認します。
- **azure-monitor-opentelemetry** がインストールされていない場合は、**pip install -r requirements.txt**をもう一度実行します。

**Application Insights にテレメトリが表示されない**
- テレメトリがアプリで送信された後、ポータルに表示されるまでに 2 から 5 分かかる場合があります。 しばらく待ってから、クエリ結果を更新してください。
- 接続文字列が正しいことを確認するには、Azure portal の Application Insights リソースの **[概要]** ページに表示されている値と比較します。
- VS Code ターミナルの出力で、テレメトリのエクスポートに関連するエラーがないかどうかを確認します。
- KQL クエリで結果が返されない場合は、**where** 句の時間の範囲を広げます (たとえば、**ago(1h)** を **ago(4h)** に変更します)。
