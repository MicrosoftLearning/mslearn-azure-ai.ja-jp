---
lab:
  topic: Azure Container Apps
  title: KEDA を使って API の自動スケーリングを構成する
  description: HTTP の同時処理トリガーを使って、Azure Container Apps で KEDA ベースのオートスケーリングを構成する方法を学びます。
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Container Apps
---

# KEDA トリガーを使用して自動スケーリングを構成する

AI アプリケーションは、しばしば、推論要求の急増、バッチ ジョブ、エージェントベースのワークフローからの突発的なスパイクなどの予測不能なワークロードに直面します。 Azure Container Apps の KEDA ベースのオートスケーリングにより、アイドル時はワークロードをゼロまでスケーリングし (コスト削減)、需要が増えれば迅速にスケールアウトできます。

この演習では、シンプルなモック エージェント API をデプロイし、**HTTP 同時要求数**に基づいて自動スケーリングを構成します。 その後、同時負荷を生成し、設定変更が適用される際にアプリのスケールアウトや新しいリビジョンが作成されるしくみを観察します。

この演習で実行されるタスク:

- Azure Container Registry と Container Apps リソースを作成する
- モック エージェント API コンテナー アプリをデプロイする
- KEDA を使って HTTP コンカレンシー スケーリング ルールを設定する
- スケールアウトをトリガーし、レプリカ数の変化をリアルタイムで監視する同時要求を生成する
- YAML を使ってスケーリング ルールを設定する

この演習の所要時間は約 **30** 分です。

>**重要:** Azure の無料クレジットを使用すると、Azure Container Registry タスクの実行が一時停止されます。 この演習には、従量課金制または別の有料プランが必要です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- 必要な Azure サービスをデプロイする権限を含む Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- [Python 3.12](https://www.python.org/downloads/) 以上。

## プロジェクト スターター ファイルをダウンロードして Azure サービスをデプロイする

このセクションでは、プロジェクト スターター ファイルをダウンロードし、スクリプトを使用してこの演習に必要なサービスをご自分の Azure サブスクリプションにデプロイします。 Azure Container Registry および Container Apps 環境のデプロイは数分で完了します。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/aca-scale-python.zip
    ```

1. プロジェクトで作業するシステム内の場所にファイルをコピーまたは移動します。 その後、ファイルをフォルダーに解凍します。

1. Visual Studio Code (VS Code) を起動し、メニューで **[ファイル] > [フォルダーを開く...]** を選択してから、プロジェクト ファイルを含むフォルダーを選びます。

1. プロジェクトには Bash (*azdeploy.sh*) と PowerShell (*azdeploy.ps1*) の両方のデプロイ スクリプトが含まれています。 お使いの環境に適したファイルを開き、スクリプトの先頭の 2 つの値をご自分のニーズに合わせて変更してから、変更を保存します。 **注:** スクリプトの他の部分は変更しないでください。

    ```
    "<your-resource-group-name>" # Resource Group name
    "<your-azure-region>" # Azure region for the resources
    ```

1. メニュー バーで、**[ターミナル] > [新しいターミナル]** を選択して、VS Code でターミナル ウィンドウを開きます。

1. 次のコマンドを実行して、Azure アカウントにログインします。 画面の指示に従って、演習用の Azure アカウントとサブスクリプションを選択します。

    ```
    az login
    ```

1. 次のコマンドを実行して、Azure CLI 用の **containerapp** 拡張機能を持っていることを確認します。

    ```azurecli
    az extension add --name containerapp
    ```

1. 次のコマンドを実行して、演習に必要なリソース プロバイダーがご自分のサブスクリプションにあることを確認します。

    ```azurecli
    az provider register --namespace Microsoft.App
    az provider register --namespace Microsoft.OperationalInsights
    ```

### Azure でリソースを作成する

このセクションでは、必要なサービスを Azure サブスクリプションにデプロイするためのデプロイ スクリプトを実行します。

1. プロジェクトのルート ディレクトリにいることを確認し、ターミナルで適切なコマンドを実行してデプロイ スクリプトを起動します。 このスクリプトは、ACR、Container Apps 環境、イングレスが有効になっているコンテナー アプリをデプロイします。 また、演習中に使う環境変数のファイルも作成します。

    **Bash**
    ```bash
    bash azdeploy.sh
    ```

    **PowerShell**
    ```powershell
    ./azdeploy.ps1
    ```

1. スクリプトの実行中に、「**1**」と入力して **Create Azure Container Registry and build container image** を起動します。

1. 前の操作が終わったら、「**2**」と入力して **Create Container Apps environment** を起動します。

1. 前の操作が終わったら、「**3**」と入力して **Create Container App** を起動します。

    >**注:** コンテナー アプリの作成後に、環境変数を含むファイルが作成されます。 これらの変数を演習全体で活用します。

1. デプロイが完了したら、「**5**」と入力してデプロイ スクリプトを終了します。

1. 適切なコマンドを実行して、前の手順で作成したファイルから環境変数をターミナル セッションに読み込みます。

    **Bash**
    ```bash
    source .env
    ```

    **PowerShell**
    ```powershell
    . .\.env.ps1
    ```

1. アプリのエンドポイントが利用可能なことを確認します。

   **Bash**
    ```bash
    curl -sS "$CONTAINER_APP_URL/" | head
    ```

    **PowerShell**
    ```powershell
    Invoke-RestMethod "$env:CONTAINER_APP_URL/"
    ```

    >**注:** ターミナルは、開いたままにします。 閉じてから新しいターミナルを作成する場合、環境変数を再び作成するコマンドを実行する必要があることがあります。

## 自動スケーリングを構成する

このセクションでは、**同時要求数**に基づいてスケーリングをトリガーする HTTP スケーリング ルールを構成します。 これは、他の Azure サービスを追加しない" 進行中のエージェント要求" の便利なプロキシです。

>**注:** 構成の更新 (スケーリングの変更を含む) を適用すると**新しいリビジョン**が作成されます。

1. 次のコマンドを実行して、HTTP スケーリング ルールを使ってコンテナー アプリを更新します。 このルールは同時処理中の要求を監視し、需要が増加した場合にアプリをスケーリングします。

    **Bash**
    ```bash
    az containerapp update \
        --name $CONTAINER_APP_NAME \
        --resource-group $RESOURCE_GROUP \
        --min-replicas 0 \
        --max-replicas 10 \
        --scale-rule-name http-scaling \
        --scale-rule-type http \
        --scale-rule-http-concurrency 10
    ```

    **PowerShell**
    ```powershell
    az containerapp update `
        --name $env:CONTAINER_APP_NAME `
        --resource-group $env:RESOURCE_GROUP `
        --min-replicas 0 `
        --max-replicas 10 `
        --scale-rule-name http-scaling `
        --scale-rule-type http `
        --scale-rule-http-concurrency 10
    ```

1. 次のコマンドを実行して、スケーリング ルールが構成されていることを確認します。 出力で **minReplicas** が **0** に設定され、**maxReplicas** が **10** に設定されている **http-scaling** ルールを探します。

    **Bash**
    ```bash
    az containerapp show \
        --name $CONTAINER_APP_NAME \
        --resource-group $RESOURCE_GROUP \
        --query "properties.template.scale"
    ```

    **PowerShell**
    ```powershell
    az containerapp show `
        --name $env:CONTAINER_APP_NAME `
        --resource-group $env:RESOURCE_GROUP `
        --query "properties.template.scale"
    ```

## 負荷を生成し、スケーリングを観察する

このセクションでは、同時要求を生成し、コンテナー アプリのリビジョンやレプリカを表示することができるローカルの Flask ダッシュボードを実行します。

1. 次のコマンドを実行して、*client* ディレクトリに移動します。

    ```
    cd client
    ```

1. 次のコマンドを入力して、クライアント アプリ用の仮想環境を作成します。 お使いの環境によって、**python** または **python3** のコマンドになることがあります。

    ```python
    python -m venv .venv
    ```

1. 次のコマンドを使用して、Python 環境をアクティブ化します。 **注:** Linux/macOS では Bash コマンドを使用します。 Windows では、PowerShell コマンドを使用します。 Windows で Git Bash を使っている場合は、**source .venv/Scripts/activate** を使用します。

    **Bash**
    ```bash
    source .venv/bin/activate
    ```

    **PowerShell**
    ```powershell
    .\.venv\Scripts\Activate.ps1
    ```

1. 次のコマンドを実行して、クライアント アプリの依存関係をインストールします。

    ```bash
    pip install -r requirements.txt
    ```

1. 次のコマンドを実行し、ダッシュボードを起動します。

    ```
    python app.py
    ```

1. ブラウザーを開き、次の URL に移動します。`http://127.0.0.1:5000`

1. アプリの左側ペインで **[Refresh Revisions & Replicas]** を選択します。 アプリの右上を見ると、**1** または **0** のレプリカが動作していることがわかります。

    アプリをデプロイしたときは、既定で **1** レプリカが実行中でした。 前のステップで KEDA スケーリング ルールを適用し、ワークロードがアイドルになってからゼロにスケールダウンされるまでにさらに**最大 5 分**かかる場合があります。これは、既定のクールダウン期間が **300 秒 (5 分)** あるためです。

1. **[Load Generator]** セクションで、コンテナー アプリにデータを送信する **[Start]** を選択します。

1. **[Refresh Revisions & Replicas]** を 5 から 10 秒ごとに選択すると、レプリカの数が増えるのが確認できるはずです。 **Load Generator** の停止後にもう一度実行すると、トラフィックとレプリカ数を増やすことができます。

終わったらブラウザー ウィンドウを閉じて、ターミナルで **Ctrl + C** を入力してクライアント アプリを終了します。

## YAML を使ってスケーリング ルールを設定する

このセクションでは、コンテナー アプリの YAML を編集して自動スケーリングを設定します。 これはスケーリング ルールを管理するための繰り返し可能な方法であり、複数のルールがある場合に不可欠です。

1. 次のコマンドを実行して、アプリの構成を YAML ファイルにエクスポートします。

    **Bash**
    ```bash
    az containerapp show \
        --name $CONTAINER_APP_NAME \
        --resource-group $RESOURCE_GROUP \
        --output yaml > app-config.yaml
    ```

    **PowerShell**
    ```powershell
    az containerapp show `
        --name $env:CONTAINER_APP_NAME `
        --resource-group $env:RESOURCE_GROUP `
        --output yaml > app-config.yaml
    ```

1. VS Code で *app-config.yaml* ファイルを開きます。 **[テンプレート] > [プロパティ]** から **[スケーリング]** セクションを見つけます。 スケーリング設定を変更して、**cooldownPeriod** を **200** 秒に短縮し (より速いスケールダウン)、**maxReplicas** を **5** に設定し、**minReplicas** を **1** に設定して、アプリで常に少なくとも 1 つのレプリカが稼働しているようにします。 **[スケーリング]** セクションは次の例のようになります。

    ```yaml
    scale:
      cooldownPeriod: 200
      maxReplicas: 5
      minReplicas: 1
      pollingInterval: 30
    ```

1. ファイルを保存し、次のコマンドを実行して、更新された構成を適用します。

    **Bash**
    ```bash
    az containerapp update \
        --name $CONTAINER_APP_NAME \
        --resource-group $RESOURCE_GROUP \
        --yaml app-config.yaml
    ```

    **PowerShell**
    ```powershell
    az containerapp update `
        --name $env:CONTAINER_APP_NAME `
        --resource-group $env:RESOURCE_GROUP `
        --yaml app-config.yaml
    ```

1. 次のコマンドを実行して、実装した変更を確認します。

    **Bash**
    ```bash
    az containerapp show \
        --name $CONTAINER_APP_NAME \
        --resource-group $RESOURCE_GROUP \
        --query "properties.template.scale"
    ```

    **PowerShell**
    ```powershell
    az containerapp show `
        --name $env:CONTAINER_APP_NAME `
        --resource-group $env:RESOURCE_GROUP `
        --query "properties.template.scale"
    ```

# リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループとグループ内のすべてのリソースを削除します。 **\<rg-name>** を、演習の前半で選択した名前に置き換えます。 このコマンドにより、Azure でバックグラウンド タスクが起動され、リソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この作業中に問題が見つかった場合は、次のステップを試してみてください。

**負荷の発生時にアプリがスケーリングされない**
- HTTP スケーリング ルールが設定されていることを確認します: **az containerapp show --query "properties.template.scale"**
- 同時要求を生成していることを確認します (ダッシュボードで delayMs > 0 を使用します)
- 要求が重複し、同時処理が蓄積するように **delayMs** を 500 から 1500 ミリ秒に増やします
- スケーリング イベントのシステム ログを確認します: **az containerapp logs show --type system --tail 50**

**ダッシュボードが起動しない、または、リビジョンやレプリカを一覧表示できない**
- Python の仮想環境がアクティブであることを確認します (ターミナル プロンプトに **(.venv)** が表示されます)
- 依存関係がインストールされていることを確認します: **pip install -r client/requirements.txt**
- Azure CLI がインストールされていることを確認し、**az login** を実行します
- **containerapp** 拡張機能がインストールされていることを確認します: **az extension add --name containerapp**
- **.env**が読み込まれ、かつ **RESOURCE_GROUP** と **CONTAINER_APP_NAME** が含まれていることを確認します

**Python venv のアクティベーションに関する問題**
- Linux/macOS では、**source client/.venv/bin/activate** を使います
- Windows の PowerShell では、**.\client\.venv\Scripts\Activate.ps1** を使います
- **activate** スクリプトが欠けている場合は、**python3-venv**パッケージを再インストールして venv を再作成します

**YAML アップデートが失敗する**
- YAML ファイルの構文が有効であることを確認します (インデントを確認します)
- **id**、**systemData**、**type**のような一部の読み取り専用プロパティはエラーを起こすことがあるため、必要に応じて削除してください
- **[プロパティ] > [テンプレート] > [スケーリング]** で、[スケーリング] セクションが正しい構造に従っていることを確認します
