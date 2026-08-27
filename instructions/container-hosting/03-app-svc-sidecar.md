---
lab:
  topic: Container hosting
  title: ローカル モデルを提供するサイドカーを備えた AI API をデプロイする
  description: チャット API とローカルの Phi-3 モデルのサイドカーを Azure App Service にデプロイしてから、ローカルの Flask クライアントを使用してアプリケーションをテストします。
  level: 300
  duration: 30
---

# ローカル モデルを提供するサイドカーを備えた AI API をデプロイする

この演習では、Python のチャット API をメインの App Service コンテナーとして、ローカル モデル サーバーをサイドカーとしてデプロイします。 デプロイ スクリプトは両方のイメージを Azure Container Registry でビルドします。 別の Flask クライアントが開発用コンピューター上で動作して、公開チャット API を呼び出します。 マネージド ID によるイメージのプル、`localhost` 通信、共有一時ボリューム、コンテナー固有の診断を構成します。

この演習で実行されるタスク:

- プロジェクトのスターター ファイルをダウンロードする
- Azure Container Registry をデプロイし、ACR タスクを使用してチャット API とモデルサーバー イメージをビルドする
- App Service プランと、マネージド ID によるイメージのプルを備えたサイドカー対応の Web アプリをデプロイする
- メインとサイドカーのコンテナー構成を定義して適用する
- モデルサイドカーの準備状況と共有ボリュームへのアクセスを検証する
- ローカルの Flask クライアント用に Python 環境を構成する
- チャット クライアントを実行し、エンドツーエンドのモデル推論をテストする

この演習の所要時間は約 **30** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- 必要な Azure サービスをデプロイする権限を持つ Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- [Python 3.12](https://www.python.org/downloads/) 以上。

## プロジェクトのスタート ファイルをダウンロードし、Azure リソースをデプロイする

このセクションではプロジェクトのスタート ファイルをダウンロードし、デプロイ スクリプトを実行します。 このスクリプトでは、リソース グループ、Azure Container Registry、両方のコンテナー イメージ、レジストリ上のユーザー割り当てマネージド ID で付与された **AcrPull**、App Service プラン、そして、作成時に ID が付与されたサイドカー対応の Web アプリが作成されます。

1. ブラウザーを開き、次の URL を入力してスタート ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/app-svc-sidecar-python.zip
    ```

1. ダウンロードしたファイルを作業フォルダーにコピーするか移動し、その内容を抽出します。

1. 抽出されたフォルダーを Visual Studio Code で開きます。

1. 1. *azdeploy.py* デプロイ スクリプトを開き、スクリプト上部の 2 つの値を必要に応じて変更して、変更を保存します。 **注:** スクリプトの他の部分は変更しないでください。

1. Visual Studio Code で新しいターミナルを開きます。

1. 次のコマンドを実行して、Azure にサインインします。 これにより Azure CLI が認証され、演習のリソースが作成されるサブスクリプションを選択できるようになります。

    ```
    az login
    ```

1. 次のコマンドを実行して、演習で使用される Azure のリソース プロバイダーを登録します。 登録により、サブスクリプションで Azure Container Registry および App Service リソースを作成できるようになります。

    ```
    az provider register --namespace Microsoft.ContainerRegistry
    az provider register --namespace Microsoft.Web
    ```

1. 次のコマンドを実行して、デプロイ スクリプトを開始します。 スクリプトによって、演習のリソースを必要な順序でプロビジョニングするためのメニューが提供されます。

    ```
    python azdeploy.py
    ```

1. 「**1**」を入力して、**[Azure Container Registry を作成して両方のイメージをビルドする]** を選択します。 このオプションにより、レジストリが作成され、その ARM 認証ポリシーがマネージド ID によるイメージのプルをサポートしていることを検証され、ACR タスクを使用してチャット API と Phi-3 モデルサーバー イメージがビルドされ、プッシュされます。

    最初のモデルサーバー ビルドでは、約 2.7 GB の Phi-3 CPU INT4 モデルがダウンロードされ、5 〜10 分かかることがあります。 両方のビルドが終わるまでターミナルは開いたままにしてください。 デプロイが失敗した場合は、「**トラブルシューティング**」セクションを確認してください。

1. 「**2**」を入力して、**[ユーザー割り当てマネージド ID を作成し、AcrPull を割り当てる]** を選択します。 このオプションにより、ユーザー割り当てマネージド ID が作成され、レジストリ上で **AcrPull** ロールが付与されるため、App Service が非公開のイメージをプルできるようになります。

1. 「**3**」を入力して、**[マネージド ID を付与した App Service リソースを作成する]** を選択します。 このオプションにより、App Service プランと、作成時にユーザー割り当てマネージド ID を付与したサイドカー対応 Web アプリが作成され、ウォームアップ中にモデル準備操作を待つように App Service が構成され、レジストリ名と ID のクライアント ID を使用して *sitecontainers-spec.json* が生成され、リソース値が *.env* および *.env.ps1* に書き込まれます。

1. 「**4**」を入力して、**[デプロイの状態を確認する]** を選択します。 レジストリ、両方のイメージ、マネージド ID、AcrPull の割り当て、プラン、Web アプリがすべて利用可能であることを確認します。

1. 「**5**」と入力して、デプロイ スクリプトを終了します。

1. 次のコマンドを実行して、Bash でリソース値を読み込みます。 このコマンドは *.env* から値をエクスポートするため、残りの Azure CLI コマンドとローカル クライアントがそれらを使用できます。

    **Bash**
    ```bash
    source .env
    ```

    **PowerShell**
    ```powershell
    . .\.env.ps1
    ```

演習のイメージは、メインの API にポート **8080** を、モデル サーバーにポート **11434** を使用します。 メインの API は **MODEL_ENDPOINT** を読み取り、**http://localhost:11434** を通じてサイドカーに推論リクエストを送信します。 **isMain** が **true** に設定されたコンテナーを定義して適用するまで、Web アプリはチャット API を提供しません。

## メインとサイドカーのコンテナーを定義する

このセクションでは、メインのチャット API コンテナーと、デプロイ スクリプトで生成された*sitecontainers-spec.json* ファイルに定義された Phi-3 モデルのサイドカーを確認し、その仕様をサイドカー対応の Web アプリに適用します。 メインの API はポート **8080** で外部トラフィックを受け取り、モデル サーバーはポート **11434** で内部に留まります。 どちらのコンテナーも、共有されたユーザー割り当てマネージド ID を使用して非公開のイメージをプルします。

プロジェクトには、レジストリ名と ID のクライアント ID のプレースホルダーを含む *sitecontainers-spec.template.json* が含まれています。 デプロイ スクリプトはそのテンプレートを保持し、レジストリ名とクライアント ID を使用して *sitecontainers-spec.json* を生成しますが、仕様は適用しません。 このセクションでは生成された構成を確認し、両方のコンテナーをデプロイします。

1. Visual Studio Code で *sitecontainers-spec.json* を開きます。

1. **chat-api** コンテナーの定義を確認し、次の設定になっていることを確認します。

    - **image** が Azure Container Registry の **chat-api:v1** イメージを指している。
    - **targetPort** が外部の App Service トラフィックを受信するサポート対象のポートである **8080** になっている。
    - **isMain** が **true** であり、このコンテナーを公開アプリケーションとして指定している。
    - **authType** が **UserAssigned** であり、App Service にユーザー割り当てマネージド ID を使用してイメージをプルするように指示している。
    - **userManagedIdentityClientId** が、レジストリ上で **AcrPull** ロールを持つ共有のユーザー割り当てマネージド ID のクライアント ID である。

1. **model-server** コンテナーの定義を確認し、次の設定になっていることを確認します。

    - **image** が Azure Container Registry 内の **model-server:v1** イメージを指している。
    - **targetPort** が **11434**、**isMain** が **false** で、モデル サーバーが内部サイドカーのままである。
    - コンテナーはメインの API と同じユーザー割り当てマネージド ID を使用してイメージをプルします。

1. 次のコマンドを実行して、メインとサイドカーのコンテナーの定義を適用します。 これにより、公開チャット API とその内部モデルのサイドカーとの間にランタイム関係が生まれ、初期コンテナーのプルが始まります。

    **Bash**
    ```bash
    az webapp sitecontainers create \
        --name "$APP_NAME" \
        --resource-group "$RESOURCE_GROUP" \
        --sitecontainers-spec-file ./sitecontainers-spec.json
    ```

    **PowerShell**
    ```powershell
    az webapp sitecontainers create `
        --name $env:APP_NAME `
        --resource-group $env:RESOURCE_GROUP `
        --sitecontainers-spec-file ./sitecontainers-spec.json
    ```

1. 次のコマンドを実行して、保存されたコンテナー定義を確認します。 これにより、App Service が割り当てられたロールとターゲット ポートで両方のコンテナーを保存したことを確認できます。

    **Bash**
    ```bash
    az webapp sitecontainers list \
        --name "$APP_NAME" \
        --resource-group "$RESOURCE_GROUP" \
        --output table
    ```

    **PowerShell**
    ```powershell
    az webapp sitecontainers list `
        --name $env:APP_NAME `
        --resource-group $env:RESOURCE_GROUP `
        --output table
    ```

1. **chat-api** がメイン コンテナーで、**model-server** がサイドカーであること、ターゲット ポートが異なること、両方の定義が同じ **userManagedIdentityClientId** で **UserAssigned** 認証を使用していることを確認します。

## モデルのサイドカーが準備完了であることを確認する

このセクションでは、ローカルのチャット クライアントを起動する前に、App Service が両方のイメージをプルし、Phi-3 モデルのサイドカーの読み込みが完了したことを確認します。 最初のコンテナーの起動には数分かかることがあります。

1. 次のコマンドを実行して、モデルサーバーのログを取得します。 コンテナー固有のログで、モデルの読み込みや起動時の問題と、メインの API のエラーを区別します。

    **Bash**
    ```bash
    az webapp sitecontainers log \
      --name "$APP_NAME" \
      --resource-group "$RESOURCE_GROUP" \
      --container-name model-server
    ```

    **PowerShell**
    ```powershell
    az webapp sitecontainers log `
      --name $env:APP_NAME `
      --resource-group $env:RESOURCE_GROUP `
      --container-name model-server
    ```

1. ログがモデルの読み込み正常に報告し、モデル サーバーがポート **11434** でリッスンしていることを確認します。

1. 次のコマンドを実行して、API 準備操作を呼び出します。 これにより、メインの API が共有のネットワーク名前空間を通じてモデルのサイドカーに到達できることが確認されます。 応答は、モデル構成や内部パスを公開せずにローカル モデル依存性が利用可能であることを報告するはずです。

    **Bash**
    ```bash
    curl --fail-with-body "${CHAT_API_URL}/health/ready"
    ```

    **PowerShell**
    ```powershell
    Invoke-RestMethod -Uri "$env:CHAT_API_URL/health/ready"
    ```

## 共有ボリュームを確認する

このセクションでは、メインの API とモデルのサイドカーが App Service の既定の **/home** 共有ボリュームにアクセスできることを確認します。 Linux の Web アプリ内のすべてのサイトコンテナーが自動的に **/home** ボリュームを共有するため、モデル サーバーはモデルを読み込んだ後に **/home/models/manifest.json** に小さなマニフェストを書き込み、メインの API が同じファイルを読み取ります。

1. 次のコマンドを実行して、非機密のモデル マニフェスト フィールド要求します。 応答が成功すると、サイドカーがマニフェストを書き、メインの API が共有の **/home** ボリュームを通じてそれを読み取ったことが証明されます。

    **Bash**
    ```bash
    curl --fail-with-body "${CHAT_API_URL}/model-info"
    ```

    **PowerShell**
    ```powershell
    Invoke-RestMethod -Uri "$env:CHAT_API_URL/model-info"
    ```

1. 応答が Microsoft Phi-3 Mini モデル、Microsoft ONNX Runtime GenAI、CPU INT4 量子化、そして準備完了状態を特定していることを確認します。

App Service はすべてのサイトコンテナーで **/home** ボリュームを自動的に共有するため、仕様で **volumeMounts** を定義する必要はありませんでした。 サイドカーは起動するたびにマニフェストを再作成できるため、ボリュームは非永続的です。 再起動後も残す必要があるデータや、スケールアウトされたインスタンス間で共有する必要があるデータは、非消費型ストレージに保管されます。

## Python 環境を設定する

このセクションでは、Python 仮想環境を作成し、ローカルの Flask クライアントに必要な依存関係をインストールします。 クライアントは App Service アプリケーションに 3 台目のコンテナーを追加せずにブラウザーによるチャット体験を提供します。

1. 次のコマンドを実行して、"クライアント" ディレクトリに移動します。**

    ```
    cd client
    ```

1. 次のコマンドを実行して、Python アプリケーション用の仮想環境を作成します。 使用する環境に応じて、コマンドは **python** または **python3** となります。

    ```
    python -m venv .venv
    ```

1. 次のコマンドを実行して、Python 環境をアクティブにします。

    > **注:** Linux/macOS では、Bash コマンドを使用してください。 Windows では、PowerShell コマンドを使用します。 Windows で Git Bash を使っている場合は、**source .venv/Scripts/activate** を使用します。

    **Bash**
    ```bash
    source .venv/bin/activate
    ```

    **PowerShell**
    ```powershell
    .\.venv\Scripts\Activate.ps1
    ```

1. 次のコマンドを実行して、Python の依存関係をインストールします。 これにより、**flask** と **requests** ライブラリがインストールされます。

    ```
    pip install -r requirements.txt
    ```

次は、ローカルの Flask アプリケーションを起動し、それを使って Azure でチャット API およびモデルのサイドカーと通信します。

## チャット クライアントを実行する

このセクションでは、ローカルの Flask Web アプリケーションを起動し、チャット API と Phi-3 モデルのサイドカーを通じてエンドツーエンドの推論を検証します。 クライアントが *.env* または *.env.ps1* から読み込んだ **CHAT_API_URL** の値を読み取ります。

1. 仮想環境がアクティブにされた "クライアント" ディレクトリにまだいることを確認します。** ターミナルのプロンプトに **(.venv)** と表示されているはずです。

1. 次のコマンドを実行して、Flask アプリケーションを開始します。

    ```
    python app.py
    ```

1. ブラウザーを開き、 `http://127.0.0.1:5000` に移動します。

1. ページで **[モデルの準備完了]** と報告されていることを確認してから、ショート メッセージを送信してください。 ブラウザーは最近のユーザーおよびアシスタント メッセージを最大 8 件まで現在のタブに保持し、制限付きの履歴をローカルの Flask クライアントを通じてチャット API に送信します。 履歴はどちらのサーバーにも保存されず、ページを再度読み込むとクリアされます。

応答では、いくつかの制限が確認されます。 App Service は外部トラフィックをメイン コンテナーにルーティングし、チャット API は **localhost:11434** に接続し、モデル サーバーは完了を返します。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループと、そのグループ内のすべてのリソースを削除します。 **\<rg-name>** は、この演習で選択した名前に置き換えてください。 このコマンドを実行すると Azure の中でバックグラウンド タスクが起動されてリソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の実行中に問題が発生した場合は、次のトラブルシューティング手順をお試しください。

**リソースのデプロイを確認する**
- [Azure portal](https://portal.azure.com) に移動してリソース グループを見つけます。
- Azure Container Registry、App Service プラン、Web アプリで、**[プロビジョニングの状態]** が **[成功]** と表示されていることを確認します。
- デプロイ スクリプトの **[デプロイの状態を確認する]** オプションを実行し、レジストリ、両方のコンテナー イメージ、プラン、Web アプリ、マネージド ID がすべて利用可能であることを確認してから、サイトコンテナーの仕様を適用します。

**デプロイの失敗を解決する**
- オプション 1 またはオプション 3 が失敗した場合、その多くは、選択した Azure リージョンでコンテナー レジストリや App Service プランの SKU の容量が一時的に不足していることが原因です。
- スクリプトを終了し、*azdeploy.py* の上部の **location** 変数を eastus2、australiaeast、canadacentral などの別の Azure リージョンに変更し、もう一度スクリプトを実行して、失敗したオプションを選択します。
- 失敗したリソースは次の試行の前に自動的に削除されます。

**モデルサーバーのビルドがタイムアウトするか、失敗する**
- 最初のモデルサーバー ビルドでは、約 2.7 GB の Phi-3 CPU INT4 モデルがダウンロードされ、5 〜10 分かかることがあります。
- ビルドが途中で失敗した場合、モデル ダウンロード中のネットワークが不安定であることが最も一般的な原因です。 オプション 1 をもう一度実行して再試行してください。
- 2 回目のビルドが常に失敗する場合は、Azure portal でレジストリの **[サービス]** > **[タスク]** > **[実行]** ブレードの下にある ACR ビルド ログで特定のエラーがないかどうかを確認します。

**マネージド ID によるイメージのプルでトークン検証の失敗が報告される**
- App Service のマネージド ID によるイメージのプルでは、レジストリの ARM 認証ポリシーを有効にする必要があります。 オプション 1 はイメージのビルド前にこのポリシーを確認します。
- ポリシーが無効化されているとスクリプトで報告された場合、次のコマンドを実行して有効にし、再度オプション 1 を実行してください。

    **Bash**
    ```bash
    az acr config authentication-as-arm update \
        --registry "$ACR_NAME" \
        --resource-group "$RESOURCE_GROUP" \
        --status enabled
    ```

    **PowerShell**
    ```powershell
    az acr config authentication-as-arm update `
        --registry $env:ACR_NAME `
        --resource-group $env:RESOURCE_GROUP `
        --status enabled
    ```

**環境変数を確認する**
- *.env* と *.env.ps1* の両方のファイルがプロジェクトのルートに存在し、**RESOURCE_GROUP**、**APP_NAME**、**ACR_NAME**、および **CHAT_API_URL** の値を含んでいることを確認します。
- **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行して環境変数をターミナル セッションに読み込んでから、Azure CLI のコマンドまたはローカル クライアントを実行します。

**サイトコンテナーの仕様を確認する**
- *azdeploy.py* の横に *sitecontainers-spec.json* が存在すること、そして各 **image** フィールドのレジストリ名がお使いのレジストリ (**$ACR_NAME.azurecr.io**) と一致していることを確認してください。
- ファイルが見つからないか、**\<registry-name>** や **\<managed-identity-client-id>** のプレースホルダーがまだ含まれている場合は、デプロイ スクリプトでオプション 3 を再度実行して再生成してください。
- **az webapp sitecontainers create** が失敗した場合、**az webapp sitecontainers list --name $APP_NAME --resource-group $RESOURCE_GROUP --output table** を実行して現在保存されている定義を確認します。

**チャット API の準備状況確認で、モデルが利用できないと報告される**
- **/health/ready** の操作では、モデルサーバーのサイドカーが Phi-3 の読み込みを終え、共有マニフェストを書き込んだ後にのみ、モデル依存関係を利用可能と報告します。
- **az webapp sitecontainers log --container-name model-server** でモデルサーバー ログを取得し、モデルがロードされたことをログが報告したことを確認し、モデル サーバーがポート **11434** でリッスンしていることを確認します。
- モデルがまだ読み込み中であることがログに表示される場合は、数分待ってから再度 **/health/ready** を呼び出してください。

**Python 環境と依存関係を確認する**
- アプリを実行する前に仮想環境が有効化されていることを確認してください。ターミナルのプロンプトに **(.venv)** が表示されるはずです。
- **pip list** を実行して、*requirements.txt* のすべてのパッケージが正常にインストールされたことを確認します。
- ローカルの Flask アプリがチャット API に到達できない場合は、**CHAT_API_URL** が **https://\<your-app-name>.azurewebsites.net** を指し、準備操作が成功したという応答を返すことを確認してください。
