---
lab:
  topic: Instrument and observe apps
  title: KQL を使用してログを照会する
  description: Application Insights で KQL を使用して要求、例外、依存関係をクエリする方法と、Azure CLI を使用して、スケジュールされたクエリ アラート ルールを作成する方法を学習します。
  level: 300
  duration: 20
  islab: true
  primarytopics:
    - Azure
---

# KQL を使用してログを照会する

Kusto 照会言語 (KQL) は、Application Insights のログ データを分析するために使用されるクエリ言語です。 KQL クエリを使用すると、要求、依存関係、例外などのテレメトリ テーブルのフィルター処理、集計、結合を行ってアプリケーションの正常性とパフォーマンスを診断することができます。 Application Insights の [ログ] ブレードには、オートコンプリート、視覚的な結果、時間範囲の制御を備えた対話型クエリ エディターが用意されており、テレメトリを調査するための主要なツールとなります。 Azure CLI を使用して作成されたスケジュール済みクエリ アラート ルールと組み合わせて KQL を使用すると、プロアクティブな監視が可能になり、失敗率や待機時間が許容可能なしきい値を超えたときにチームに通知されます。

この演習では、Application Insights リソースをデプロイし、OpenTelemetry を使用してサンプル要求、依存関係、および例外テレメトリを生成する Python スクリプトを実行し、Azure portal の [ログ] ブレードで KQL クエリを記述して、アプリケーションの正常性を調査します。 要求テーブルのクエリを実行して失敗を特定し、例外を要求と結合してエラーを関連付け、依存関係の待機時間をパーセンタイル計算で分析し、Azure CLI を使用してアクション グループとログ検索アラート ルールを作成します。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードする
- Application Insights リソースを作成する
- テレメトリ ジェネレーターを実行してサンプルデータを作成する
- Azure portal で KQL を使用してテレメトリのクエリを実行する
- Azure CLI を使用してアクション グループとアラート ルールを作成する

この演習の所要時間は約 **20** 分です。

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
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/analyze-logs-python.zip
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

1. 次のコマンドを実行して、演習に必要なリソース プロバイダーが自分のサブスクリプションにあることを確認します。

    ```
    az provider register --namespace Microsoft.Insights
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

1. スクリプトの実行中、「**1**」と入力して、**[1. Application Insights を作成する]** オプションを起動します。

    このオプションは、リソース グループがまだ存在していない場合にそれを作成し、Application Insights リソースを作成します。

1. 「**2**」と入力して、**[2. ロールを割り当てる]** オプションを実行します。 これにより、アカウントに Monitoring Metrics Publisher ロールが割り当てられるため、アプリで Microsoft Entra 認証を使用して Application Insights にテレメトリを発行できるようになります。

1. 「**3**」と入力して、**[3. デプロイ状態を確認する]** オプションを実行します。 続行する前に、Application Insights リソースに **Succeeded** と表示され、ロールが割り当てられていることを確認します。 リソースのプロビジョニングがまだ完了していない場合は、しばらく待ってからもう一度試してください。

1. 「**4**」と入力して、**[4. 接続情報を取得する]** オプションを実行します。 これにより、アプリと CLI コマンドで必要な Application Insights 接続文字列、リソース グループ名、Application Insights 名、リソース ID を含む環境変数ファイルが作成されます。

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

    >**注:** ターミナルは開いたままにします。 ターミナルを閉じて新しいターミナルを作成する場合は、このコマンドをもう一度実行して、環境変数を再度読み込む必要があります。

## テレメトリ データを生成する

このセクションでは、Python 環境を設定し、事前に作成されたテレメトリ ジェネレーターを実行してて Application Insights でサンプルの要求、依存関係、例外データを作成し、データが到着するのを待ってからクエリを実行します。 このジェネレーター スクリプトは、OpenTelemetry を使用して、Application Insights の**要求**、**依存関係**、**例外**の各テーブルにマッピングするスパンを作成します。

1. VS Code ターミナルで次のコマンドを実行して、*client* ディレクトリに移動します。

    ```
    cd client
    ```

1. 次のコマンドを実行して、Python 環境を作成します。

    ```
    python -m venv .venv
    ```

1. 次のコマンドを実行して、Python 環境をアクティブにします。 **注:** Linux/macOS では Bash コマンドを使用します。 Windows では、PowerShell コマンドを使用します。 Windows で Git Bash を使っている場合は、**source .venv/Scripts/activate** を使用します。

    **Bash**
    ```bash
    source .venv/bin/activate
    ```

    **PowerShell**
    ```powershell
    .\.venv\Scripts\Activate.ps1
    ```

1. 次のコマンドを実行して、依存関係をインストールします。

    ```
    pip install -r requirements.txt
    ```

1. 次のコマンドを実行して、テレメトリ ジェネレーターを起動します。

    ```
    python app.py
    ```

1. このスクリプトにより、15 個の要求スパン、12 個の依存関係スパン、5 個の例外スパンが生成され、概要が出力されます。 スクリプトが完了するのを待ちます。

1. このスクリプトをさらに 2 回実行して、追加のテレメトリ データを生成します。

    ```
    python app.py
    ```

1. テレメトリが Application Insights に到着するまで 2 から 3 分待ちます。 テレメトリはバッチ処理され、定期的に送信されるため、データが表示されるまでに短い遅延があります。

## Azure portal でテレメトリのクエリを実行する

このセクションでは、Application Insights の [ログ] ブレードを使用して、生成したテレメトリに対して KQL クエリを実行します。 [ログ] ブレードには、オートコンプリート、表形式の結果、チャート レンダリングを備えた対話型エディターが用意されています。

1. [Azure portal](https://portal.azure.com) に移動し、先ほど作成したリソース グループ内で Application Insights リソースを見つけます。

1. Application Insights リソースで、左側のナビゲーションの **[監視]** の下にある **[ログ]** を選択します。 表示されたクエリ テンプレート ダイアログを閉じます。 **注意:** クエリ バーのドロップダウン セレクターで、必ず **[KQL モード]** を選択してください。

### 失敗した要求のクエリを実行する

1. 次のクエリをコピーしてクエリ エディターに貼り付け、**[実行]** を選択します。 このクエリは、**where** 演算子を使用して、**要求**テーブルを失敗した要求のみにフィルター処理し、**summarize** を使用してサービス名と HTTP 状態コード別にグループ化します。 **count()** 集計は各組み合わせの失敗回数を集計し、**order by** は、最も頻繁に発生した失敗が最初に表示されるように結果を並べ替えます。

    ```kusto
    requests
    | where success == false
    | summarize failedCount = count() by cloud_RoleName, resultCode
    | order by failedCount desc
    ```

1. 結果を確認します。 api-gateway、doc-processor、auth-service サービスの行に、状態コード 500 と 429 が表示されます。 **failedCount** 列は、各組み合わせで発生した失敗の回数を示します。

### クエリ要求の量とパフォーマンス

1. 次のクエリをコピーしてクエリ エディターに貼り付け、**[実行]** を選択します。 このクエリは、**bin(timestamp, 1h)** を使用して要求を 1 時間間隔にバケット化し、サービス名別にグループ化します。 バケットごとに、要求の総数、平均 (**avg()** を使用)、95 パーセンタイル期間 (**percentile()** を使用) が計算されます。 95 パーセンタイルは、要求の 95% が完了する応答時間を示しており、テール待機時間の検出に役立ちます。

    ```kusto
    requests
    | summarize requestCount = count(),
        avgDuration = avg(duration),
        p95Duration = percentile(duration, 95)
        by bin(timestamp, 1h), cloud_RoleName
    | order by timestamp desc
    ```

1. 結果を確認します。 **p95Duration** 列には、最も遅い上位 5% の要求によって発生した応答時間が表示されます。 平均処理時間を 95 パーセンタイルと比較します。大きな差は、ほとんどの要求は高速であるが、一部の要求で大幅な遅延が発生するサービスを強調しています。

### 例外を要求と結合する

1. 次のクエリをコピーしてクエリ エディターに貼り付け、**[実行]** を選択します。 このクエリは、**例外**テーブルから開始し、**join kind=inner** を使用し、共有の **operation_Id** フィールドを介して各例外を、それをトリガーした要求と一致させます。 両方のテーブルに **name** 列があるため、サブクエリ内の **project** は、結合前に **name** を **requestName** に名前変更して列名の衝突を回避します。 結合後、**summarize** は、結果を要求名、例外の種類、サービス別にグループ化し、それぞれの組み合わせで発生した例外の数をカウントします。 **Take 15** 演算子は、出力を上位 15 行に制限します。

    ```kusto
    exceptions
    | join kind=inner (requests | project requestName = name, operation_Id, cloud_RoleName) on operation_Id
    | summarize exceptionCount = count()
        by requestName, exceptionType = type, cloud_RoleName
    | order by exceptionCount desc
    | take 15
    ```

1. 結果を確認します。 各行は、要求名、例外の種類、サービスの組み合わせを示します。 このビューは、最も多くのエラーが発生する操作と、それに関係する例外の種類を特定するのに役立ちます。

### 依存関係の待機時間を分析する

1. 次のクエリをコピーしてクエリ エディターに貼り付け、**[実行]** を選択します。 このクエリは、**依存関係**テーブルを読み取ります。このテーブルには、アプリケーションがデータベース、API、その他のサービスに対して行う送信呼び出しが記録されています。 **target** (依存エンドポイント) と **type** (HTTP や SQL など) 別にグループ化され、呼び出し回数と平均期間、および 50 パーセンタイル、95 パーセンタイル、99 パーセンタイルの待機時間が計算されます。 複数のパーセンタイルを比較すると、遅い応答が孤立した外れ値であるか、より広範なパターンであるかが明らかになります。

    ```kusto
    dependencies
    | summarize callCount = count(),
        avgDuration = avg(duration),
        p50 = percentile(duration, 50),
        p95 = percentile(duration, 95),
        p99 = percentile(duration, 99)
        by target, type
    | order by p95 desc
    ```

1. 結果を確認します。 p50 と p95 の値の間に大きな差がある場合は、パフォーマンスに一貫性がないことを示しており、ほとんどの呼び出しは高速であるが、かなりの割合の呼び出しに非常に時間がかかることを示します。

## アクション グループとアラート ルールを作成する

このセクションでは、Azure CLI を使用して、アクション グループと、失敗率がしきい値を超えた場合を検出するログ検索アラート ルールを作成します。

1. VS Code ターミナルで次のコマンドを実行して、メール通知付きのアクション グループを作成します。 **ALERT_EMAIL** 変数は、デプロイ スクリプトで **[接続情報の取得]** オプションを実行したときに Azure アカウントから設定されました。

    **Bash**
    ```bash
    az monitor action-group create \
        --resource-group $RESOURCE_GROUP \
        --name pipeline-alerts-ag \
        --short-name PipeAlert \
        --action email oncall-email $ALERT_EMAIL
    ```

    **PowerShell**
    ```powershell
    az monitor action-group create `
        --resource-group $env:RESOURCE_GROUP `
        --name pipeline-alerts-ag `
        --short-name PipeAlert `
        --action email oncall-email $env:ALERT_EMAIL
    ```

    このコマンドは、トリガーされたときにメール通知を送信する **pipeline-alerts-ag** という名前のアクション グループを作成します。

1. 次のコマンドを実行して、Application Insights リソースで失敗した要求を監視するログ検索アラート ルールを作成します。 このルールは、5 分間の時間枠でクエリを評価し、10 を超える要求が失敗した場合に発動されます。

    **Bash**
    ```bash
    az monitor scheduled-query create \
        --resource-group $RESOURCE_GROUP \
        --name high-failure-rate-alert \
        --scopes $APPINSIGHTS_RESOURCE_ID \
        --condition "count 'FailedRequests' > 10" \
        --condition-query FailedRequests="requests | where success == false" \
        --evaluation-frequency 5m \
        --window-size 5m \
        --severity 1 \
        --description "Alert when more than 10 requests fail in a 5-minute window"
    ```

    **PowerShell**
    ```powershell
    az monitor scheduled-query create `
        --resource-group $env:RESOURCE_GROUP `
        --name high-failure-rate-alert `
        --scopes $env:APPINSIGHTS_RESOURCE_ID `
        --condition "count 'FailedRequests' > 10" `
        --condition-query FailedRequests="requests | where success == false" `
        --evaluation-frequency 5m `
        --window-size 5m `
        --severity 1 `
        --description "Alert when more than 10 requests fail in a 5-minute window"
    ```

    重大度は、重大な問題ではあるが、停止レベルの問題ではないことを示す 1 (エラー) に設定されます。 アラートが発動されると、アクション グループがトリガーされ、メールが送信されます。

1. 次のコマンドを実行して、アラート ルールが作成され、有効になっていることを確認します。

    **Bash**
    ```bash
    az monitor scheduled-query show \
        --resource-group $RESOURCE_GROUP \
        --name high-failure-rate-alert \
        --query "{Name:name, Severity:severity, Enabled:enabled, Window:windowSize, Frequency:evaluationFrequency, Threshold:condition.failingPeriods}" \
        --output table
    ```

    **PowerShell**
    ```powershell
    az monitor scheduled-query show `
        --resource-group $env:RESOURCE_GROUP `
        --name high-failure-rate-alert `
        --query "{Name:name, Severity:severity, Enabled:enabled, Window:windowSize, Frequency:evaluationFrequency, Threshold:condition.failingPeriods}" `
        --output table
    ```

    **Enabled** 列に **True** が表示されており、重大度、時間枠、頻度の値が前の手順で指定した値と一致していることを確認します。

1. 次のコマンドを実行して、リソース グループ内のすべてのスケジュール済みクエリ アラート ルールを一覧表示します。

    **Bash**
    ```bash
    az monitor scheduled-query list \
        --resource-group $RESOURCE_GROUP \
        --output table
    ```

    **PowerShell**
    ```powershell
    az monitor scheduled-query list `
        --resource-group $env:RESOURCE_GROUP `
        --output table
    ```

    **high-failure-rate-alert** という名前のルールが表示されます。 失敗した要求に関するテレメトリは既に生成されているため、ルールは 5 分のサイクルで即座に評価を開始します。 時間枠内で失敗した要求の数がしきい値を超えると、Azure Monitor によってアラートが発動され、アクション グループによって、指定されたメール アドレスにメール通知が送信されます。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループとグループ内のすべてのリソースを削除します。 **\<rg-name>** を、演習で先ほど選択した名前に置き換えます。 このコマンドにより、Azure でバックグラウンド タスクが起動され、リソース グループが削除されます。

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

**環境変数を検証する**
- *.env* ファイルがプロジェクトのルートに存在し、**APPLICATIONINSIGHTS_CONNECTION_STRING**、**RESOURCE_GROUP**、**APPINSIGHTS_NAME**、**APPINSIGHTS_RESOURCE_ID**、**ALERT_EMAIL** の値が含まれていることを確認します。
- **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行して環境変数をターミナル セッションに読み込んだことを確認します。
- 変数が空の場合は、**source .env** (Bash) または **. .\.env.ps1** (PowerShell) を再実行します。

**認証を確認する**
- **az account show** を実行して、Azure CLI にログインしていることを確認します。
- Azure portal でロールの割り当てを確認するか、デプロイ スクリプトのオプションを実行してロールをもう一度割り当てて、Metrics Publisher ロールがアカウントに割り当てられていることを確認します。

**Python 環境と依存関係を確認する**
- スクリプトを実行する前に、仮想環境がアクティブになっていることを確認します。
- **pip list** を実行して、*requirements.txt* のすべてのパッケージが正常にインストールされたことを確認します。
- **azure-monitor-opentelemetry** がインストールされていない場合は、**pip install -r requirements.txt**をもう一度実行します。

**クエリ結果にテレメトリが表示されない**
- テレメトリがスクリプトで送信された後、表示されるまでに 2 から 5 分かかる場合があります。 しばらく待ってから、[ログ] ブレードで **[実行]** をもう一度選択します。
- 接続文字列が正しいことを確認するには、Azure portal の Application Insights リソースの **[概要]** ページに表示されている値と比較します。
- VS Code ターミナルの出力で、テレメトリのエクスポートに関連するエラーがないかどうかを確認します。
- クエリで結果が返されない場合は、[ログ] ブレードのタイム ピッカーを使用して時間の範囲を広げます (たとえば、**[過去 1 時間]** から **[過去 4 時間]** に変更します)。

**アラート ルール作成エラー**
- **az extension add --name scheduled-query** を実行して、**scheduled-query** 拡張機能がインストールされていることを確認します。
- **echo $APPINSIGHTS_RESOURCE_ID** を実行して、**APPINSIGHTS_RESOURCE_ID** 環境変数に有効なリソース ID が含まれていることを確認します。
- コマンドにプレビュー拡張機能が必要な場合は、まず **az config set extension.dynamic_install_allow_preview=true** を実行します。
