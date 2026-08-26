---
lab:
  topic: Container hosting
  title: ACR タスクを使用して、コンテナー イメージをビルドおよび実行する
  description: Azure Container Registry (ACR) タスクを使用して、ローカルの Docker インストールを必要とせずに、完全にクラウド内でコンテナー イメージをビルドおよび管理する方法について学習します。
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Container Registry
---

# ACR タスクを使用して、コンテナー イメージをビルドおよび実行する

この演習では、Azure Container Registry (ACR) タスクを使用して、ローカルの Docker インストールを必要とせずに、完全にクラウド内でコンテナー イメージをビルドおよび管理します。

この演習で実行されるタスク:

- プロジェクトのスターター ファイルをダウンロードする
- Azure Container Registry をデプロイする
- ACR タスクを使用してコンテナー イメージをビルドし、確認する
- イメージのバージョンを管理し、本番イメージを保護する

この演習の所要時間は約 **30** 分です。

>**重要:** Azure の無料クレジットを使用すると、Azure Container Registry タスクの実行が一時停止されます。 この演習には、従量課金制または別の有料プランが必要です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- 必要な Azure サービスをデプロイする権限を持つ Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- [Python 3.12](https://www.python.org/downloads/) 以上。

## プロジェクト スターター ファイルをダウンロードして Azure サービスをデプロイする

このセクションでは、プロジェクト スターター ファイルをダウンロードし、スクリプトを使用して必要なサービスを Azure サブスクリプションにデプロイします。 Azure Container Registry のデプロイは、完了までに数分かかります。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/acr-tasks-python.zip
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

1. 次のコマンドを実行して、サブスクリプションに Azure Container Registry (ACR) をインストールするために必要なリソース プロバイダーがあることを確認します。

    ```
    az provider register --namespace Microsoft.ContainerRegistry
    ```

1. プロジェクトのルート ディレクトリにいることを確認し、ターミナルで次のコマンドを実行してデプロイ スクリプトを起動します。 デプロイ スクリプトによって ACR がデプロイされ、演習に必要な環境変数のファイルが作成されます。

    ```
    python azdeploy.py
    ```

1. 適切なコマンドを実行して、環境変数をターミナル セッションに読み込みます。

    **Bash**
    ```bash
    source .env
    ```

    **PowerShell**
    ```powershell
    . .\.env.ps1
    ```

    >**注:** ターミナルは開いたままにします。 閉じて新しいターミナルを作成すると、環境変数を再度作成するコマンドを実行しなければならない場合があります。

## イメージを ACR タスクでビルドする

このセクションでは、ローカル コンピューターで Docker を必要とせずに Azure でイメージをビルドするための簡単なタスクを使用します。 **az acr build** コマンドによってソース ファイルがアップロードされ、クラウドにイメージがビルドされ、レジストリにプッシュされます。

1. 次のコマンドを実行してビルドし、レジストリにプッシュします。 ビルドは Azure 内で完結します。 ローカルの Docker インストールは必要ありません。

    **Bash**
    ```bash
    az acr build \
        --registry $ACR_NAME \
        --image inference-api:v1.0.0 \
        ./api
    ```

   **PowerShell**
    ```powershell
    az acr build `
        --registry $env:ACR_NAME `
        --image inference-api:v1.0.0 `
        ./api
    ```

1. ACR タスクとして出力をご覧ください。

    - ソース コンテキストが Azure にパッケージ化され、アップロードされます
    - ビルド タスクがキューに入り、開始します
    - 各レイヤーを示す Docker ビルド出力がストリーミングされます
    - 完成したイメージがレジストリにプッシュされます
    - イメージのダイジェストとタスクの状況が報告されます


## レジストリ内のイメージを確認する

このセクションでは、リポジトリやタグを一覧表示して、イメージがレジストリに存在していることを確認します。

1. 次のコマンドを実行して、レジストリ内のすべてのリポジトリを一覧表示します。

    **Bash**
    ```bash
    az acr repository list --name $ACR_NAME --output table
    ```

    **PowerShell**
    ```powershell
    az acr repository list --name $env:ACR_NAME --output table
    ```

    出力は作成した **inference-api** リポジトリを示しています。

1. **inference-api** リポジトリのタグを一覧表示するコマンドを実行します。

    **Bash**
    ```bash
    az acr repository show-tags \
        --name $ACR_NAME \
        --repository inference-api \
        --output table
    ```

    **PowerShell**
    ```powershell
    az acr repository show-tags `
        --name $env:ACR_NAME `
        --repository inference-api `
        --output table
    ```

    出力にはビルド時に割り当てた **v1.0.0** のタグが表示されます。

1. 次のコマンドを実行して、詳細なマニフェスト情報 (ダイジェストを含む) を表示します。

    **Bash**
    ```bash
    az acr manifest list-metadata \
        --registry $ACR_NAME \
        --name inference-api \
        --output table
    ```

    **PowerShell**
    ```powershell
    az acr manifest list-metadata `
        --registry $env:ACR_NAME `
        --name inference-api `
        --output table
    ```

    ダイジェスト値に注目してください。 この SHA-256 ハッシュはタグに関係なくイメージを一意に識別します。

## イメージを ACR タスクで実行する

このセクションでは、**az acr run** コマンドを使用してビルドしたイメージ内でコマンドを実行し、正しく動作することを確認します。

1. 次のコマンドを実行して、Flask アプリケーションがコンテナーに正常に読み込まれることを確認します。

    **Bash**
    ```bash
    az acr run \
        --registry $ACR_NAME \
        --cmd "$ACR_NAME.azurecr.io/inference-api:v1.0.0 python -c 'from app import app'" \
        /dev/null
    ```

    **PowerShell**
    ```powershell
    az acr run `
        --registry $env:ACR_NAME `
        --cmd "$env:ACR_NAME.azurecr.io/inference-api:v1.0.0 python -c 'from app import app'" `
        /dev/null
    ```

    出力には、イメージをダウンロードする docker pull の進行状況が含まれます。 実行が成功すると、**Run ID: xxx was successful after xxx** で終了します。 これによりコンテナーが正しく実行され、Flask アプリケーションがエラーなくインポートされることが確認されます。

## 異なるタグでビルドする

このセクションでは、新しいバージョンのイメージを異なるタグで作成し、レジストリで複数のバージョンがどのように管理されるかを確認します。

1. 次のコマンドを実行して、新しいバージョン タグでイメージを再度ビルドします。

    **Bash**
    ```bash
    az acr build \
        --registry $ACR_NAME \
        --image inference-api:v1.1.0 \
        ./api
    ```

    **PowerShell**
    ```powershell
    az acr build `
        --registry $env:ACR_NAME `
        --image inference-api:v1.1.0 `
        ./api
    ```

1. 次のコマンドを実行して、すべてのタグを一覧表示し、両方のバージョンを表示します。

    **Bash**
    ```bash
    az acr repository show-tags \
        --name $ACR_NAME \
        --repository inference-api \
        --output table
    ```

    **PowerShell**
    ```powershell
    az acr repository show-tags `
        --name $env:ACR_NAME `
        --repository inference-api `
        --output table
    ```

    出力には **v1.0.0** と **v1.1.0** の両方が表示され、レジストリで複数のバージョンがどのように管理されるかがわかります。

## ビルド履歴を表示し、本番イメージをロックする

このセクションでは、ACR タスクの実行履歴をレビューし、イメージをロックして誤った変更から守ります。

1. 次のコマンドを実行して、ACR タスクの実行履歴をレビューし、これまでに実行したすべてのビルドを確認します。

    **Bash**
    ```bash
    az acr task list-runs \
        --registry $ACR_NAME \
        --output table
    ```

    **PowerShell**
    ```powershell
    az acr task list-runs `
        --registry $env:ACR_NAME `
        --output table
    ```

    出力には、各ビルド タスクの実行 ID、状態、トリガー タイプ、期間が表示されます。 この履歴は、ビルドの追跡や問題の診断に役立ちます。

1. 誤った削除や改変を防ぐために、v1.0.0 のイメージをロックする次のコマンドを実行します。

    **Bash**
    ```bash
    az acr repository update \
        --name $ACR_NAME \
        --image inference-api:v1.0.0 \
        --write-enabled false
    ```

    **PowerShell**
    ```powershell
    az acr repository update `
        --name $env:ACR_NAME `
        --image inference-api:v1.0.0 `
        --write-enabled false
    ```

1. 次のコマンドを実行して、ロックが適用されていることを確認します。

    **Bash**
    ```bash
    az acr repository show \
        --name $ACR_NAME \
        --image inference-api:v1.0.0
    ```

    **PowerShell**
    ```powershell
    az acr repository show `
        --name $env:ACR_NAME `
        --image inference-api:v1.0.0
    ```

    **[writeEnabled]** フィールドには **[False]** が表示され、イメージが保護されていることが示されています。

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

**ビルドの失敗をトラブルシューティングする**
- ビルド出力にエラー メッセージがないかどうかを確認します。よくある問題は Dockerfile の欠如やファイル パスの誤りです。
- プロジェクトのルート ディレクトリ (*api* フォルダーがある場所) からコマンドを実行していることを確認します。
- 「**az acr task list-runs --registry $ACR_NAME --output table**」を実行して、最近のビルドの状態を確認します。

