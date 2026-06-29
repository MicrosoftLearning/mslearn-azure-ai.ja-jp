---
lab:
  topic: Azure Container Apps
  title: 機能しなくなったデプロイを診断し修正する
  description: 環境変数の欠落やイングレスの構成の誤りの診断、および過去のトラブルシューティングに関する Log Analytics へのクエリの実行により、Azure Container Apps のトラブルシューティングを行う方法を学びます。
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Container Apps
---

# 機能しなくなったデプロイを診断し修正する

この演習では、機能しなくなったコンテナー アプリをトラブルシューティングし、的を絞った修正プログラムを適用します。 リビジョンの状態、ログ、Azure CLI を使用して、デプロイの問題を切り分けます。 このワークフローは AI ソリューションで一般的です。モデルや依存関係を更新すると起動の動作が頻繁に変わるためです。

この演習で実行されるタスク:

- モック AI ドキュメント処理 API をコンテナー アプリとしてデプロイする
- 欠落している環境変数を導入し診断する
- イングレス構成の問題の導入と診断
- 過去のトラブルシューティング データについて Log Analytics にクエリを実行する

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
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/aca-manage-python.zip
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

1. 次のコマンドを実行して、Azure CLI 用の **containerapp** 拡張機能を持っていることを確認します。

    ```azurecli
    az extension add --name containerapp
    ```

1. 次のコマンドを実行して、演習に必要なリソース プロバイダーがご自分のサブスクリプションにあることを確認します。

    ```azurecli
    az provider register --namespace Microsoft.App
    az provider register --namespace Microsoft.OperationalInsights
    az provider register --namespace Microsoft.ContainerRegistry
    ```

### Azure でリソースを作成する

このセクションでは、必要なサービスを Azure サブスクリプションにデプロイするためのデプロイ スクリプトを実行します。

1. プロジェクトのルート ディレクトリにいることを確認し、ターミナルで適切なコマンドを実行してデプロイ スクリプトを起動します。 デプロイ スクリプトによって ACR がデプロイされ、演習に必要な環境変数を含むファイルが作成されます。

    **Bash**
    ```bash
    bash azdeploy.sh
    ```

    **PowerShell**
    ```powershell
    ./azdeploy.ps1
    ```

1. スクリプトの実行中に、「**1**」と入力して **Create Azure Container Registry and build container image** オプションを起動します。 このオプションは ACR サービスを作成し、ACR タスクを使ってイメージを構築し、レジストリにプッシュします。

1. 前の操作が終わったら、「**2**」と入力して **Create Container Apps environment** オプションを起動します。 コンテナーをデプロイする前に環境を作成する必要があります。

1. 前の操作が完了したら、「**3**」と入力して **Deploy the container app and configure secrets** オプションを起動します。

    >**注:** コンテナー アプリの作成後に、環境変数を含むファイルが作成されます。 これらの変数を演習全体で活用します。

1. 前の操作が完了したら、「**5**」と入力してデプロイ スクリプトを終了します。

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

1. 次のコマンドを実行して、既定のエンドポイントを呼び出してアプリが動作していることを確認します。 このコマンドにより、いくつかの JSON が返されます。 **[model.name]** フィールドを探します。**gpt-4o-mini** に設定されているはずです。

    **Bash**
    ```bash
    curl -s "https://$FQDN/"
    ```

    **PowerShell**
    ```powershell
    Invoke-RestMethod -Uri "https://$FQDN/"
    ```

## 欠落している環境変数を診断する

コンテナー アプリが設定されていない環境変数に依存している場合、アプリが起動しなかったり、予期せぬ挙動を起こしたりすることがあります。 このセクションでは、必要な環境変数を削除し、症状を観察します。

1. 次のコマンドを実行してコンテナー アプリを更新し、`MODEL_NAME` 環境変数を削除します。

    **Bash**
    ```bash
    az containerapp update -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
        --remove-env-vars MODEL_NAME
    ```

    **PowerShell**
    ```powershell
    az containerapp update -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP `
        --remove-env-vars MODEL_NAME
    ```

1. 次のコマンドを実行してリビジョンを一覧表示し、新しいリビジョンが作成されたことを確認します。 サフィックス番号がより高く (例: **ai-api--0000002**)、**TrafficWeight** が **100** の新しいリビジョンを探します。これは、すべてのトラフィックを受信していることを示します。

    **Bash**
    ```bash
    az containerapp revision list -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP -o table
    ```

    **PowerShell**
    ```powershell
    az containerapp revision list -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP -o table
    ```

1. 次のコマンドを実行してルート エンドポイントを確認し、API 利用者の視点から症状を観察します。 **[model.name]** フィールドには設定値の代わりに既定値の **not-configured** が表示されています。

    **Bash**
    ```bash
    curl -s "https://$FQDN/" | jq .model
    ```

    **PowerShell**
    ```powershell
    (Invoke-RestMethod -Uri "https://$FQDN/").model

1. Run the following command to diagnose the root cause by viewing the container app's configuration. Run the following command to confirm the **MODEL_NAME** environment variable is missing.

    **Bash**
    ```bash
    az containerapp show -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
        --query "properties.template.containers[0].env" -o table
    ```

    **PowerShell**
    ```powershell
    az containerapp show -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP `
        --query "properties.template.containers[0].env" -o table
    ```

1. 次のコマンドを実行して、`MODEL_NAME` 環境変数をもう一度追加して問題を解決します。

    **Bash**
    ```bash
    az containerapp update -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
        --set-env-vars MODEL_NAME=$MODEL_NAME
    ```

    **PowerShell**
    ```powershell
    az containerapp update -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP `
        --set-env-vars MODEL_NAME=$env:MODEL_NAME
    ```

1. 次のコマンドを実行して、ルート エンドポイントをもう一度確認して修正を検証します。 これにより、API 利用者の視点からアプリケーションが正しく動作することを確認します。 応答には設定済みのモデル名が表示されるはずです。

    **Bash**
    ```bash
    curl -s "https://$FQDN/" | jq .model
    ```

    **PowerShell**
    ```powershell
    (Invoke-RestMethod -Uri "https://$FQDN/").model
    ```

欠落していた環境変数を診断し、修正しました。 次に、シークレットのイングレスの問題を診断します。

## イングレス構成の問題の診断

Container Apps は **target-port** 設定を使ってトラフィックをコンテナーにルーティングします。 ポートがアプリケーションがリッスンしているものと一致しない場合、要求は失敗します。 このセクションでは、ポートの不一致について説明します。

1. 次のコマンドを実行して、コンテナー アプリが誤ったターゲット ポートを使用するように更新します。

    **Bash**
    ```bash
    az containerapp ingress update -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
        --target-port 3000
    ```

    **PowerShell**
    ```powershell
    az containerapp ingress update -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP `
        --target-port 3000
    ```

1. 次のコマンドを実行して、API 利用者の視点から症状を観察するために、正常性エンドポイントへのアクセスを試みます。

    **Bash**
    ```bash
    curl -s "https://$FQDN/health"
    ```

    **PowerShell**
    ```powershell
    Invoke-RestMethod -Uri "https://$FQDN/health"
    ```

    リクエストが失敗またはタイムアウトするのは、Container Apps がトラフィックをポート 3000 にルーティングしているのに、アプリケーションはポート 8000 でリッスンしているためです。

1. 次のコマンドを実行して、現在のイングレス構成を確認して根本原因を診断します。 **targetPort** が 3000 に設定されていることに注目してください。

    **Bash**
    ```bash
    az containerapp show -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
        --query "properties.configuration.ingress" -o yaml
    ```

    **PowerShell**
    ```powershell
    az containerapp show -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP `
        --query "properties.configuration.ingress" -o yaml
    ```

1. 次のコマンドを実行して、コンテナー ログを調べて、アプリケーションが実行されているかどうかを確認します。 Gunicorn の起動メッセージで、アプリがポート 8000 でリッスンしていることが表示されるので、不一致を確認できます。

    **Bash**
    ```bash
    az containerapp logs show -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP
    ```

    **PowerShell**
    ```powershell
    az containerapp logs show -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP
    ```

1. 次のコマンドを実行して、正しいターゲット ポートを設定してイングレス構成を修正します。

    **Bash**
    ```bash
    az containerapp ingress update -n $CONTAINER_APP_NAME -g $RESOURCE_GROUP \
        --target-port 8000
    ```

    **PowerShell**
    ```powershell
    az containerapp ingress update -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP `
        --target-port 8000
    ```

1. 次のコマンドを実行して、正常性エンドポイントを呼び出して修正を確認します。 これにより、API 利用者の視点からアプリケーションがアクセス可能であることを確認します。 **{"status":healthy"}** が表示されます。

    **Bash**
    ```bash
    curl -s "https://$FQDN/health"
    ```

    **PowerShell**
    ```powershell
    Invoke-RestMethod -Uri "https://$FQDN/health"
    ```

イングレスの構成の問題を診断し修正しました。 次に、履歴ログにクエリを実行する方法について説明します。

## 過去のトラブルシューティングについて Log Analytics にクエリを実行する

**az containerapp logs** では最近のコンソールログのみが表示されます。 過去のトラブルシューティングについては、ログは Container Apps 環境に関連付けられた Log Analytics ワークスペースに保存されます。

1. 次のコマンドを実行して、Container Apps 環境から Log Analytics ワークスペース ID を取得します。

    **Bash**
    ```bash
    WORKSPACE_ID=$(az containerapp env show -n $ACA_ENVIRONMENT -g $RESOURCE_GROUP \
        --query properties.appLogsConfiguration.logAnalyticsConfiguration.customerId -o tsv)

    echo "Workspace ID: $WORKSPACE_ID"
    ```

    **PowerShell**
    ```powershell
    $WORKSPACE_ID = az containerapp env show -n $env:ACA_ENVIRONMENT -g $env:RESOURCE_GROUP `
        --query properties.appLogsConfiguration.logAnalyticsConfiguration.customerId -o tsv

    Write-Output "Workspace ID: $WORKSPACE_ID"
    ```

1. 次のコマンドを実行して、コンテナー アプリにコンソールログのクエリを実行します。 これにより、タイムスタンプとメッセージを示す直近 20 個のログ エントリが返されます。

    **Bash**
    ```bash
    az monitor log-analytics query -w $WORKSPACE_ID \
        --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s == '$CONTAINER_APP_NAME' | project TimeGenerated, Log_s | order by TimeGenerated desc | take 20" \
        -o table
    ```

    **PowerShell**
    ```powershell
    az monitor log-analytics query -w $WORKSPACE_ID `
        --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s == '$env:CONTAINER_APP_NAME' | project TimeGenerated, Log_s | order by TimeGenerated desc | take 20" `
        -o table
    ```

    > [!NOTE]
    > イベント発生後に Log Analytics データが表示されるまで、数分かかることがあります。 最近のログが表示されない場合は、数分待ってからもう一度やり直してみてください。

1. 次のコマンドを実行して、具体的にエラー レベルのログのクエリを実行します。

    **Bash**
    ```bash
    az monitor log-analytics query -w $WORKSPACE_ID \
        --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s == '$CONTAINER_APP_NAME' and Log_s contains 'error' | order by TimeGenerated desc | take 20" \
        -o table
    ```

    **PowerShell**
    ```powershell
    az monitor log-analytics query -w $WORKSPACE_ID `
        --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s == '$env:CONTAINER_APP_NAME' and Log_s contains 'error' | order by TimeGenerated desc | take 20" `
        -o table
    ```

これらのクエリは、コンテナーの再起動やリビジョンの変更後でも、過去に発生した問題を調査するのに役立ちます。

## リソースをクリーンアップする

クリーンアップは継続的なコストを回避します。 リソースグループを削除すると、Container Apps 環境、コンテナー アプリ、レジストリも削除されます。

```bash
az group delete --name $RESOURCE_GROUP --no-wait --yes
```

## トラブルシューティング

この作業中に問題が見つかった場合は、次のステップを試してみてください。

**コンテナー アプリが応答しない**
- **az containerapp revision list** を使ってリビジョンが有効かどうかを確認します
- **az containerapp show** を使ってイングレスが構成されていることを確認します

**ログが表示されない**
- コンソール ログは最近のもののみです。 過去のデータには Log Analytics を使用します。
- Log Analytics のデータが表示されるには、2 から 5 分かかることがあります。

**環境変数が有効にならない**
- Container Apps では環境変数を変更すると新しいリビジョンが作成されます。 新しいリビジョンがアクティブであることを確認します。
- **--replace-env-vars** を慎重に使います。これにより、指定した変数だけでなく、すべての環境変数が置き換えられます。
