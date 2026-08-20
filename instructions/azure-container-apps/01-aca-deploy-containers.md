---
lab:
  topic: Azure Container Apps
  title: コンテナー化されたバックエンド API を Container Apps にデプロイする
  description: 安全なイメージ プル用のマネージド ID を使用して、Azure Container Registry (ACR) から Azure Container Apps にコンテナー イメージをデプロイし、デプロイを確認してログを表示する方法について説明します。
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Container Apps
    - Azure Container Registry
---

# コンテナ化されたバックエンド API を Azure Container Apps にデプロイする

この演習では、コンテナ化されたバックエンド API を Azure Container Apps にデプロイします。 マネージド ID を使用して、Azure Container Registry からイメージを安全にプルし、環境変数としてシークレットを構成します。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードして Azure サービスをデプロイする
- マネージド ID 認証を使用してコンテナー アプリをデプロイする
- シークレットを構成し、環境変数から参照する
- API エンドポイントを呼び出し、ログを確認してデプロイを検証する

この演習の所要時間は約 **30** 分です。

>**重要:** Azure の無料クレジットを使用すると、Azure Container Registry タスクの実行が一時停止されます。 この演習には、従量課金制または別の有料プランが必要です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- 必要な Azure サービスをデプロイする権限を持つ Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- オプション: [Python 3.12](https://www.python.org/downloads/) 以上。

## プロジェクト スターター ファイルをダウンロードして Azure サービスをデプロイする

このセクションでは、プロジェクト スターター ファイルをダウンロードし、スクリプトを使用してこの演習に必要なサービスをご自分の Azure サブスクリプションにデプロイします。 Azure Container Registry および Container Apps 環境のデプロイは数分で完了します。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/aca-deploy-python.zip
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

1. 次のコマンドを実行して、Azure CLI 用の **containerapp** 拡張機能を持っていることを確認します。

    ```azurecli
    az extension add --name containerapp
    az extension add --name log-analytics
    ```

1. 次のコマンドを実行して、演習に必要なリソース プロバイダーがご自分のサブスクリプションにあることを確認します。

    ```azurecli
    az provider register --namespace Microsoft.App
    az provider register --namespace Microsoft.OperationalInsights
    az provider register --namespace Microsoft.ContainerRegistry
    ```

### Azure でリソースを作成する

このセクションでは、必要なサービスを Azure サブスクリプションにデプロイするためのデプロイ スクリプトを実行します。

1. プロジェクトのルート ディレクトリにいることを確認し、ターミナルで次のコマンドを実行してデプロイ スクリプトを起動します。 デプロイ スクリプトによって ACR がデプロイされ、演習に必要な環境変数のファイルが作成されます。

    ```
    python azdeploy.py
    ```

1. スクリプトの実行中に、「**1**」と入力して **Create Azure Container Registry and build container image** オプションを起動します。 このオプションは ACR サービスを作成し、ACR タスクを使ってイメージを構築し、レジストリにプッシュします。

1. 前の操作が終わったら、「**2**」と入力して **Create Container Apps environment** オプションを起動します。 コンテナーをデプロイする前に環境を作成する必要があります。

    >**注:** Container Apps 環境の作成後に、環境変数を含むファイルが作成されます。 これらの変数を演習全体で活用します。

1. 前の操作が完了したら、「**4**」と入力してデプロイ スクリプトを終了します。

1. 適切なコマンドを実行して、前の手順で作成したファイルから環境変数をターミナル セッションに読み込みます。

    **Bash**
    ```bash
    source .env
    ```

    **PowerShell**
    ```powershell
    . .\.env.ps1
    ```

    >**注:** ターミナルは開いたままにします。 閉じてから新しいターミナルを作成する場合、環境変数を再び作成するコマンドを実行する必要があることがあります。

## コンテナー アプリをデプロイし、シークレットを設定する

このセクションでは、外部イングレスを含むコンテナー アプリとして API をデプロイします。 イメージはプライベート レジストリにあるため、最初のリビジョンでイメージをプルできるように、作成時にレジストリ認証を構成する必要があります。 その後、シークレットを構成し、環境変数から参照します。 このパターンは、AI アプリがプロバイダー API キーを保存する方法を反映しています

1. システムに割り当てられたマネージド ID でコンテナー アプリを作成し、作成時にレジストリ認証を構成します。 **--registry-identity** フラグは、Container Apps にアプリのマネージド ID を使って指定されたレジストリからイメージを取得するよう指示します。 Azure Container Registry でこのフラグを使うと、CLI により自動的に **AcrPull** ロールが割り当てられます。

    **Bash**
    ```azurecli
    az containerapp create \
        --name $CONTAINER_APP_NAME \
        --resource-group $RESOURCE_GROUP \
        --environment $ACA_ENVIRONMENT \
        --image "$ACR_SERVER/$CONTAINER_IMAGE" \
        --ingress external \
        --target-port $TARGET_PORT \
        --env-vars MODEL_NAME=$MODEL_NAME \
        --registry-server "$ACR_SERVER" \
        --registry-identity system
    ```

    **PowerShell**
    ```powershell
    az containerapp create `
        --name $env:CONTAINER_APP_NAME `
        --resource-group $env:RESOURCE_GROUP `
        --environment $env:ACA_ENVIRONMENT `
        --image "$env:ACR_SERVER/$env:CONTAINER_IMAGE" `
        --ingress external `
        --target-port $env:TARGET_PORT `
        --env-vars MODEL_NAME=$env:MODEL_NAME `
        --registry-server "$env:ACR_SERVER" `
        --registry-identity system
    ```

1. シークレットを作成し、環境変数から参照します。

    **Bash**
    ```azurecli
    az containerapp secret set -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
        --secrets embeddings-api-key=$EMBEDDINGS_API_KEY
    ```

    **PowerShell**
    ```powershell
    az containerapp secret set -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP `
        --secrets embeddings-api-key=$env:EMBEDDINGS_API_KEY
    ```

1. 環境変数からシークレットを参照します。 このコマンドで新しいリビジョンが作成され、アプリが再起動されてシークレットの変更が反映されます。

    **Bash**
    ```azurecli
    az containerapp update -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
        --set-env-vars EMBEDDINGS_API_KEY=secretref:embeddings-api-key
    ```

    **PowerShell**
    ```powershell
    az containerapp update -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP `
        --set-env-vars EMBEDDINGS_API_KEY=secretref:embeddings-api-key
    ```

1. 次のコマンドを実行してリビジョンを一覧表示し、新しいリビジョンが作成されたことを確認します。

    **Bash**
    ```azurecli
    az containerapp revision list -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP -o table
    ```

    **PowerShell**
    ```powershell
    az containerapp revision list -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP -o table
    ```

    リビジョン名は `--0000002` のようなサフィックスで終わり、これが 2 番目のリビジョンであることを示します。 Container Apps では環境変数やシークレットを変更するたびに新しいリビジョンが作成され、更新された設定でアプリが再起動されます。 以前の非アクティブなリビジョンは、時間が経つと削除されることがあります。

## デプロイを検証する

アプリが起動し、イングレスが正常に動作していることを確認する必要があります。 また、ログを使ってアプリが期待どおりに動作していることも確認します。

1. 次のコマンドを実行してアプリの FQDN を取得し、結果を変数に保存します。

    **Bash**
    ```bash
    FQDN=$(az containerapp show -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
        --query properties.configuration.ingress.fqdn -o tsv)

    echo "$FQDN"
    ```

    **PowerShell**
    ```powershell
    $FQDN = az containerapp show -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP `
        --query properties.configuration.ingress.fqdn -o tsv

    Write-Output $FQDN
    ```

1. 次のコマンドを実行して、正常性エンドポイントを呼び出します。 コマンドで、**{"status": "healthy"}** が返されるはずです。

    **Bash**
    ```bash
    curl -s "https://$FQDN/health"
    ```

    **PowerShell**
    ```powershell
    Invoke-RestMethod -Uri "https://$FQDN/health"
    ```

1. 次のコマンドを実行し、ルート エンドポイントを呼び出してシークレットが構成されていることを確認します。 エンドポイントから、構成されたモデル名、および API キー シークレットが構成されているかどうかを含めたアプリ情報を含む JSON が返されます。

    **Bash**
    ```bash
    curl -s "https://$FQDN/"
    ```

    **PowerShell**
    ```powershell
    Invoke-RestMethod -Uri "https://$FQDN/"
    ```

1. 次のコマンドを実行してドキュメント処理エンドポイントをテストします。 コマンドにより *document.txt* ファイルがエンドポイントに送信されます。 この操作は、モック データ分析情報を含む JSON を返します。

    **Bash**
    ```bash
    curl -s -X POST "https://$FQDN/process" \
        -H "Content-Type: text/plain" \
        -d @document.txt
    ```

    **PowerShell**
    ```powershell
    Invoke-RestMethod -Uri "https://$FQDN/process" `
        -Method Post `
        -ContentType "text/plain" `
        -Body (Get-Content -Raw document.txt)
    ```

1. 次のコマンドを実行して、スタートアップおよびランタイム シグナルのログを確認します。 このコマンドは最近のコンソール出力のみを表示します。 過去のログと詳細なトラブルシューティングについて、ログは Container Apps 環境に関連付けられた Log Analytics ワークスペースに保存されます。

    **Bash**
    ```azurecli
    az containerapp logs show -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP
    ```

    **Powershell**
    ```powershell
    az containerapp logs show -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP
    ```

    ワーカーが生成されて、ポート 8000 でリッスンしていることを示す、**gunicorn** スタートアップ メッセージを探します。 また、curl コマンド (GET/health、POST/process など) からの HTTP リクエスト ログも確認できます。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループと、そのグループ内のすべてのリソースを削除します。 **\<rg-name>** は、この演習で選択した名前に置き換えてください。 このコマンドを実行すると Azure の中でバックグラウンド タスクが起動されてリソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の実行中に問題が発生した場合は、次のトラブルシューティング手順をお試しください。

**スクリプトでデプロイの状態を確認する**

- デプロイ スクリプトを実行し、オプション **3** を選択して、ACR および Container Apps 環境の状態を確認します。 これにより、ベース インフラストラクチャがデプロイされ、コンテナー イメージが存在することを確認できます。

**Azure 認証と環境変数を確認する**

- **az account show** を実行して、正しい Azure サブスクリプションにログインしていることを確認します。
- **echo $ACR_NAME** (Bash)、または **$env:ACR_NAME** (PowerShell) を実行して、環境変数が設定されていることを確認します。
- 変数が空の場合は、**source .env** (Bash) または **. .\.env.ps1** (PowerShell) をもう一度実行します。

**ACR のデプロイを確認する**

- [Azure portal](https://portal.azure.com) に移動してリソース グループを見つけます。
- Azure Container Registry が存在し、**[プロビジョニングの状態]** が **[成功]** と表示されていることを確認します。
- **az acr list --output table** を実行して、レジストリがアクセス可能であることを確認します。

**ビルドの失敗のトラブルシューティングを行う**

- デプロイ スクリプトでは、詳細な **az acr build** 出力が抑制されます。 失敗のトラブルシューティングを行うには、直近の ACR タスク実行のステータスとログを確認してください。
- デプロイ スクリプトがプロジェクトのルート ディレクトリ (*api* フォルダーがある場所) から実行されていることを確認します。
- 最近の ACR タスク実行を一覧表示します。
    - **Bash:** **az acr task list-runs --registry $ACR_NAME --output table**
    - **PowerShell:** **az acr task list-runs --registry $env:ACR_NAME --output table**
- 特定の実行のログを表示します (**\<run-id>** を前のコマンドの値に置き換えます)。
    - **Bash:** **az acr task logs --registry $ACR_NAME --run-id \<run-id>**
    - **PowerShell:** **az acr task logs --registry $env:ACR_NAME --run-id \<run-id>**

**コンテナーのプルの失敗のトラブルシューティング (ImagePullBackOff / unauthorized / 403)**

- コンテナー アプリでシステム割り当てマネージド ID が有効になっていることを確認します。
    - **Bash:** **az containerapp identity show -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP**
    - **PowerShell:** **az containerapp identity show -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP**
- コンテナー アプリの **AcrPull** ロール割り当ての範囲がレジストリに設定されていることを確認します。 ロール割り当ては作成から伝達までに 1、2 分かかることがあります。

**コンテナーのスタートアップとアプリケーション エラーのトラブルシューティング**

- スタートアップの問題を診断するためにコンテナーのログをストリーミングします。
    - **Bash:** **az containerapp logs show -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP --follow**
    - **PowerShell:** **az containerapp logs show -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP --follow**
- デプロイ直後にアプリから 502/503 が返された場合は、1 分待ってからもう一度試してみてください。 最初の起動では、Container Apps がコンテナーをプルして起動するまでに時間がかかることがあります。
- プロビジョニング エラーのリビジョンの状態を確認します。
    - **Bash:** **az containerapp revision list -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP -o table**
    - **PowerShell:** **az containerapp revision list -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP -o table**

**シークレット構成のトラブルシューティング**

- シークレットが作成されたことを確認します。
    - **Bash:** **az containerapp secret list -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP -o table**
    - **PowerShell:** **az containerapp secret list -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP -o table**
- API キーが構成されているかどうかを示すルート エンドポイント (**/**) を呼び出して、環境変数がシークレットを正しく参照していることを確認します。

