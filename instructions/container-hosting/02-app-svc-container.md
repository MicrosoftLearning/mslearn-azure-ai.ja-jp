---
lab:
  topic: Container hosting
  title: コンテナーを Azure App Service にデプロイする
  description: マネージド ID を使用して Azure Container Registry (ACR) から Azure App Service へコンテナー イメージをデプロイし、その後実行中のコンテナーを検証し、トラブルシューティングする方法について学習します。
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure App Service
    - Azure Container Registry
---

# コンテナーを Azure App Service にデプロイする

この演習では、Azure Container Registry (ACR) から Azure App Service に Linux コンテナー イメージをデプロイします。 Web アプリでシステム割り当てマネージド ID と **AcrPull** ロールが使用されるように設定すれば、アプリ設定にレジストリの認証情報を保存することなく、App Service でプライベート レジストリからイメージをプルできるようになります。

この演習で実行されるタスク:

- プロジェクトのスターター ファイルをダウンロードする
- Azure Container Registry をデプロイし、ACR タスクを使用してコンテナー イメージをビルドする
- Linux コンテナー向けの App Service プランをデプロイする
- マネージド ID を使用して ACR からプルされるように Web App for Containers の作成と構成を行う
- ランタイム設定を構成し、コンテナー ログを有効にする
- デプロイを確認して、ドキュメント処理エンドポイントをテストする

この演習の所要時間は約 **30** 分です。

>**重要:** Azure の無料クレジットを使用すると、Azure Container Registry タスクの実行が一時停止されます。 この演習には、従量課金制または別の有料プランが必要です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- 必要な Azure サービスをデプロイする権限を持つ Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- [Python 3.12](https://www.python.org/downloads/) 以上。


## プロジェクト スターター ファイルをダウンロードして Azure サービスをデプロイする

このセクションでは、プロジェクト スターター ファイルをダウンロードし、スクリプトを使用して必要なサービスを Azure サブスクリプションにデプロイします。 Azure Container Registry および App Service プランのデプロイは数分で完了します。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/app-svc-container-python.zip
    ```

1. プロジェクトで作業するシステム内の場所にファイルをコピーまたは移動します。 その後、ファイルをフォルダーに解凍します。

1. Visual Studio Code (VS Code) を起動し、メニューで **[ファイル] > [フォルダーを開く...]** を選択してから、プロジェクト ファイルを含むフォルダーを選びます。

1. *azdeploy.py* デプロイ スクリプトを開き、スクリプト上部の 2 つの値を必要に応じて変更して、変更を保存します。 **注:** スクリプトの他の部分は変更しないでください。

    ```
    "<your-resource-group-name>" # Resource Group name
    "<your-azure-region>" # Azure region for the resources
    ```

1. メニュー バーで、**[ターミナル] > [新しいターミナル]** を選択して、VS Code でターミナル ウィンドウを開きます。

1. 次のコマンドを実行して、Azure アカウントにログインします。 プロンプトに答えて、演習用の Azure アカウントとサブスクリプションを選択してください。

    ```
    az login
    ```

1. 次のコマンドを実行して、サブスクリプションに Azure Container Registry (ACR) および Azure App Service に必要なリソース プロバイダーが含まれていることを確認します。

    ```
    az provider register --namespace Microsoft.ContainerRegistry
    az provider register --namespace Microsoft.Web
    ```

### Azure でリソースを作成する

このセクションでは、必要なサービスを Azure サブスクリプションにデプロイするためのデプロイ スクリプトを実行します。

1. プロジェクトのルート ディレクトリにいることを確認し、ターミナルで次のコマンドを実行してデプロイ スクリプトを起動します。 デプロイ スクリプトによって ACR がデプロイされ、演習に必要な環境変数のファイルが作成されます。

    ```
    python azdeploy.py
    ```

1. スクリプトの実行中に、「**1**」と入力して **[1. Azure Container Registry を作成してコンテナー イメージをビルドする]** オプションを起動します。 このオプションは ACR サービスを作成し、ACR タスクを使ってイメージをビルドし、レジストリにプッシュします。

1. 前の操作が終わったら、「**2**」と入力して **[App Service プランを作成]** オプションを起動します。 このオプションは Web アプリに必要な App Service プランを作成します。

    >**注:** 環境変数を含むファイルは、App Service プランの作成後に作成されます。 これらの変数は演習全体で使用します。

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

    >**注:** ターミナルは開いたままにします。 閉じて新しいターミナルを作成すると、環境変数を再度作成するコマンドを実行しなければならない場合があります。

## Web アプリの作成

このセクションでは、CLI コマンドを使用して Web アプリを作成します。 その後、システム割り当てマネージド ID を使用して Web アプリを設定し、アプリが ACR のイメージにアクセスできるようにします。

1. 次のコマンドを実行して、コンテナー レジストリからプルするように設定した Web App for Containers を作成します。

    **Bash**
    ```bash
    az webapp create \
        --resource-group $RESOURCE_GROUP \
        --plan $APP_PLAN \
        --name $APP_NAME \
        --container-image-name $ACR_NAME.azurecr.io/docprocessor:v1
    ```

    **PowerShell**
    ```powershell
    az webapp create `
        --resource-group $env:RESOURCE_GROUP `
        --plan $env:APP_PLAN `
        --name $env:APP_NAME `
        --container-image-name "$($env:ACR_NAME).azurecr.io/docprocessor:v1"
    ```

    既定では、Azure Container Registry は非公開です。 App Service では、イメージをプルする前に ACR を認証する方法が必要です。

    その認証は、アプリ設定にレジストリの認証情報を保存する代わりに、システム割り当てマネージド ID (推奨) を使って設定します。

1. 次のコマンドを使用して、Web アプリでシステム割り当てマネージド ID を有効にします。

    **Bash**
    ```bash
    az webapp identity assign \
        --resource-group $RESOURCE_GROUP \
        --name $APP_NAME
    ```

    **PowerShell**
    ```powershell
    az webapp identity assign `
        --resource-group $env:RESOURCE_GROUP `
        --name $env:APP_NAME
    ```

### AcrPull ロールを Web アプリに割り当てる

このセクションでは、Web アプリにプライベート レジストリからイメージをプルする権限を与えます。 マネージド ID とは、Azure が作成し、管理する Microsoft Entra でバックアップされる ID です。 Web アプリでシステム割り当て ID を有効にすると、App Service はその ID としてトークンを要求できます。

Web アプリがその ID を使ってイメージをプルできるように、組み込みの **AcrPull** ロールをレジストリに割り当てます。 これでアクセス許可が最小限になります。Web アプリはイメージをダウンロードできますが、レジストリのプッシュや管理はできません。

1. 次のコマンドを実行して、Web アプリのプリンシパル ID を取得します。

    **Bash**
    ```bash
    PRINCIPAL_ID=$(az webapp identity show \
        --resource-group $RESOURCE_GROUP \
        --name $APP_NAME \
        --query principalId \
        --output tsv)
    ```

    **PowerShell**
    ```powershell
    $PRINCIPAL_ID = az webapp identity show `
        --resource-group $env:RESOURCE_GROUP `
        --name $env:APP_NAME `
        --query principalId `
        --output tsv
    ```
1. 次のコマンドを実行して、ACR の ID を取得します。

    **Bash**
    ```bash
    ACR_ID=$(az acr show \
        --resource-group $RESOURCE_GROUP \
        --name $ACR_NAME \
        --query id \
        --output tsv)
    ```

    **PowerShell**
    ```powershell
    $ACR_ID = az acr show `
        --resource-group $env:RESOURCE_GROUP `
        --name $env:ACR_NAME `
        --query id `
        --output tsv
    ```

1. 次のコマンドを実行して、Web アプリに AcrPull ロールを割り当てます。

    **Bash**
    ```bash
    az role assignment create \
        --assignee $PRINCIPAL_ID \
        --scope $ACR_ID \
        --role AcrPull
    ```

    **PowerShell**
    ```powershell
    az role assignment create `
        --assignee $PRINCIPAL_ID `
        --scope $ACR_ID `
        --role AcrPull
    ```

    >**注:** ロールの割り当ては伝達に 1 〜 2 分かかることがあります。 このステップが終わってもアプリですぐにイメージをプルできない場合は、少し待ってから再試行してください。

1. 次のコマンドを実行して、レジストリ認証にマネージド ID を使用するように Web アプリを設定します。 この設定により、App Service にはコンテナー レジストリにアクセスする際に (レジストリ管理者の認証情報ではなく) Web アプリのマネージド ID を使用するように指示されます。

    **Bash**
    ```bash
    az webapp config set \
        --resource-group $RESOURCE_GROUP \
        --name $APP_NAME \
        --acr-use-identity true \
        --acr-identity [system]
    ```

    **PowerShell**
    ```powershell
    az webapp config set `
        --resource-group $env:RESOURCE_GROUP `
        --name $env:APP_NAME `
        --acr-use-identity true `
        --acr-identity [system]
    ```

1. 次のコマンドを実行して、マネージド ID 付きのレジストリを使用するようにコンテナー設定を更新します。 このステップで、Web アプリが使用すべきイメージとレジストリの URL が明示的に設定されます。 後でイメージ タグを更新した場合は、ここで Web アプリを新しいバージョンにポイントします。

    **Bash**
    ```bash
    az webapp config container set \
        --resource-group $RESOURCE_GROUP \
        --name $APP_NAME \
        --container-image-name $ACR_NAME.azurecr.io/docprocessor:v1 \
        --container-registry-url https://$ACR_NAME.azurecr.io
    ```

    **PowerShell**
    ```powershell
    az webapp config container set `
        --resource-group $env:RESOURCE_GROUP `
        --name $env:APP_NAME `
        --container-image-name "$($env:ACR_NAME).azurecr.io/docprocessor:v1" `
        --container-registry-url "https://$($env:ACR_NAME).azurecr.io"
    ```

## ランタイム設定を構成し、コンテナー ログを有効にする

このセクションでは、ランタイム設定を構成し、ログを有効にしてコンテナー実行の信頼性を高め、問題のトラブルシューティングを支援します。

1. 次のコマンドを実行して、コンテナー ポートを設定します。 サンプル イメージはポート 80 (既定) でリスンしているため、このステップでは動作が変わることなく設定が示されます。

    **Bash**
    ```bash
    az webapp config appsettings set \
        --resource-group $RESOURCE_GROUP \
        --name $APP_NAME \
        --settings WEBSITES_PORT=80
    ```

    **PowerShell**
    ```powershell
    az webapp config appsettings set `
        --resource-group $env:RESOURCE_GROUP `
        --name $env:APP_NAME `
        --settings WEBSITES_PORT=80
    ```

1. 次のコマンドを実行して、処理済みドキュメントの永続ストレージを有効にします。 この設定により、App Service のストレージ マウント (たとえば、Linux コンテナーの **/home** パス) が有効になります。

    **Bash**
    ```bash
    az webapp config appsettings set \
        --resource-group $RESOURCE_GROUP \
        --name $APP_NAME \
        --settings WEBSITES_ENABLE_APP_SERVICE_STORAGE=true
    ```

    **PowerShell**
    ```powershell
    az webapp config appsettings set `
        --resource-group $env:RESOURCE_GROUP `
        --name $env:APP_NAME `
        --settings WEBSITES_ENABLE_APP_SERVICE_STORAGE=true
    ```

1. 次のコマンドを実行して、常時接続を有効にします。 常時接続では、アプリを起動したままにすることでコールドスタートの待ち時間を減らします。

    **Bash**
    ```bash
    az webapp config set \
        --resource-group $RESOURCE_GROUP \
        --name $APP_NAME \
        --always-on true
    ```

    **PowerShell**
    ```powershell
    az webapp config set `
        --resource-group $env:RESOURCE_GROUP `
        --name $env:APP_NAME `
        --always-on true
    ```

1. 次のコマンドを実行して、コンテナー ログを有効にします。 これにより、コンテナーから stdout/stderr がキャプチャされるため、CLI からログを表示できます。

    **Bash**
    ```bash
    az webapp log config \
        --resource-group $RESOURCE_GROUP \
        --name $APP_NAME \
        --docker-container-logging filesystem
    ```

    **PowerShell**
    ```powershell
    az webapp log config `
        --resource-group $env:RESOURCE_GROUP `
        --name $env:APP_NAME `
        --docker-container-logging filesystem
    ```

## デプロイを検証する

このセクションでは、Web アプリが動作し、応答していることを確認します。

1. 次のコマンドを実行して、Web アプリのホスト名を取得します。

    **Bash**
    ```bash
    APP_URL=$(az webapp show \
        --resource-group $RESOURCE_GROUP \
        --name $APP_NAME \
        --query defaultHostName \
        --output tsv)

    echo "Application URL: https://$APP_URL"
    ```

    **PowerShell**
    ```powershell
    $APP_URL = az webapp show `
        --resource-group $env:RESOURCE_GROUP `
        --name $env:APP_NAME `
        --query defaultHostName `
        --output tsv

    Write-Host "Application URL: https://$APP_URL"
    ```

1. ブラウザーで URL を開き、アプリケーションが応答していることを確認します。 ブラウザーは演習の後半で使用するので、開いたままにしておいてください。 アプリケーションは実行中を示す応答を返すはずです。 最初の要求は、App Service がコンテナー イメージをプルし、アプリケーションを起動するため、時間がかかることがあります。

## ドキュメント処理をテストする

このセクションでは、API に要求を送信し、アプリが動作していること、およびその結果が永続ストレージに書き込まれていることを確認します。

1. 次のコマンドを実行して、プロジェクトに含まれる *document.txt* ファイルを処理エンドポイントに送信します。

    **Bash**
    ```bash
    curl -X POST "https://$APP_URL/process" \
        -H "Content-Type: text/plain" \
        --data-binary @document.txt
    ```

    **PowerShell**
    ```powershell
    $body = Get-Content -Raw -Path "document.txt"
    Invoke-RestMethod -Method Post -Uri "https://$APP_URL/process" -ContentType "text/plain" -Body $body | ConvertTo-Json -Depth 10
    ```

    API は抽出されたエンティティ、キー フレーズ、感情分析などのモック分析結果を返します。 応答には、結果が永続ストレージに保存されたかどうかが示されています。

1. 次のコマンドを実行して、処理済みのすべてのドキュメントを一覧表示します。

    **Bash**
    ```bash
    curl https://$APP_URL/documents
    ```

    **PowerShell**
    ```powershell
    Invoke-RestMethod -Uri "https://$APP_URL/documents" | ConvertTo-Json -Depth 10
    ```

    永続ストレージが正しく有効になっていれば、今処理したドキュメントが一覧に表示されるはずです。

## コンテナー ログをストリーミングする

このセクションでは、起動や要求処理のトラブルシューティングに役立つコンテナー ログのストリーミングを行います。

1. 次のコマンドを実行して、コンテナーからリアルタイムのログを表示します。

    **Bash**
    ```bash
    az webapp log tail \
        --resource-group $RESOURCE_GROUP \
        --name $APP_NAME
    ```

    **PowerShell**
    ```powershell
    az webapp log tail `
        --resource-group $env:RESOURCE_GROUP `
        --name $env:APP_NAME
    ```

1. ブラウザーを更新することで、アプリケーションへの要求がさらに増えます。 ストリームにログ エントリーが表示されるはずです。 ストリーミングを停止するには、Ctrl キーを押しながら C キーを押します。

## 診断コンソールを調べる

このセクションでは、SCM (Kudu) サイトを開き、設定ビューや共通ログの場所を調べます。

1. 次のコマンドを実行して、SCM (Kudu) の URL を出力します。

    **Bash**
    ```bash
    echo "Kudu URL: https://$APP_NAME.scm.azurewebsites.net"
    ```

    **PowerShell**
    ```powershell
    Write-Host "Kudu URL: https://$($env:APP_NAME).scm.azurewebsites.net"
    ```

1. ブラウザーでこの URL を開きます。 ページ上部のメニューで、次の場所へ移動します。

    1. **[環境]** は環境変数を表示し、アプリの設定があることを確認します。
    1. **[SSH]** を展開し、**[Kudu]** を選択すると、ブラウザーベースのシェルが開きます。
    1. **[ファイル マネージャー]** で、**/home/LogFiles/** に移動してログ ファイルを表示します。

    >**ヒント:** 上部のメニューの **[ログ ストリーム]** を使用してブラウザーでログを表示したり、**[SSH]** オプションを使用してアプリ コンテナーに接続したりすることもできます。

    SCM サイトはアプリ コンテナーとは別なので、コンテナーのファイル システムや実行中のプロセス全体を表示できるわけではありません。

## アプリケーション設定を表示する

このセクションでは、設定したアプリ設定が存在していることを確認します。

1. 次のコマンドを実行して、アプリケーション設定を一覧表示します。

    **Bash**
    ```bash
    az webapp config appsettings list \
        --resource-group $RESOURCE_GROUP \
        --name $APP_NAME \
        --output table
    ```

    **PowerShell**
    ```powershell
    az webapp config appsettings list `
        --resource-group $env:RESOURCE_GROUP `
        --name $env:APP_NAME `
        --output table
    ```

    設定がシステム提供の設定とともに一覧に表示されていることを確認します。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループと、そのグループ内のすべてのリソースを削除します。 **\<rg-name>** は、この演習で選択した名前に置き換えてください。 このコマンドを実行すると Azure の中でバックグラウンド タスクが起動されてリソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習中に問題が発生した場合は、次のトラブルシューティングのステップをお試しください。

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
- 特定の実行のログを表示します (**<run-id>** は前のコマンドの値に置き換えます)。
    - **Bash:** **az acr task logs --registry $ACR_NAME --run-id <run-id>**
    - **PowerShell:** **az acr task logs --registry $env:ACR_NAME --run-id <run-id>**

**コンテナーのプルの失敗のトラブルシューティング (ImagePullBackOff / unauthorized / 403)**
- **az webapp identity show** を実行して、Web アプリのシステム割り当てマネージド ID が有効になっていることを確認します。
- Web アプリの **AcrPull** ロール割り当ての範囲がレジストリに設定されていることを確認します。 ロール割り当ては作成から伝達までに 1 〜 2 分かかることがあります。
- コンテナー設定ステップを再度実行して、イメージ名とレジストリ URL が正しいことを確認してください。

**コンテナー設定とアプリケーション エラーのトラブルシューティング**
- コンテナー ログが有効になっていることを確認して、ログをストリーミングします。
    - **Bash:** **az webapp log tail --resource-group $RESOURCE_GROUP --name $APP_NAME**
    - **PowerShell:** **az webapp log tail --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME**
- デプロイ直後にアプリが 502/503 を返した場合は、1 分待ってから再試行してください。 最初の起動では、App Service がコンテナーをプルして起動するまでに時間がかかることがあります。

**永続ストレージが有効であることを確認する**

- **WEBSITES_ENABLE_APP_SERVICE_STORAGE** 設定が存在し、**[True]** に設定されていることを確認します。
- ドキュメントを送信した後、**/documents** エンドポイントを呼び出して、結果が永続ストレージに書き込まれていることを確認します。

