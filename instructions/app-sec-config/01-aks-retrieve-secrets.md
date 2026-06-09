---
lab:
  topic: App secrets and configuration
  title: Azure Key Vault を使用してシークレットを管理する
  description: Python SDK で Azure Key Vault を使用してシークレットを保存、取得、バージョン管理、キャッシュする方法について説明します。
  level: 300
  duration: 20
  islab: true
  primarytopics:
    - Azure
    - Azure Key Vault
---

# Azure Key Vault を使用してシークレットを管理する

AI アプリケーションは通常、API キー、接続文字列、証明書などの機密性の高い資格情報に依存して、モデル エンドポイントとデータ ストアにアクセスします。 Azure Key Vault では、RBAC アクセス制御、自動バージョン管理、監査ログを使用して、これらのシークレット用の一元化された安全なストアが提供されるため、アプリケーションはコードまたは構成ファイルに資格情報を埋め込む必要はありません。

この演習では、サンプル シークレットを事前に読み込んだ Azure Key Vault をデプロイし、Azure SDK を使用して中核的シークレット管理パターンを示す Python Flask Web アプリケーションをビルドします。 シークレットを取得し、そのメタデータを検査して、すべてのシークレット プロパティを値を公開せずにリストアップし、認証情報のローテーションをシミュレートするために新しいシークレット バージョンを作成し、Key Vault の API 呼び出しを減らすために時間ベースのキャッシュを実装します。

この演習で実行されるタスク:

- プロジェクトのスターター ファイルをダウンロードする
- Azure Key Vault を作成してサンプル シークレットを保存する
- スターター ファイルにコードを追加してアプリを完成させる
- アプリを実行してシークレットを操作する

この演習の所要時間は約 **20** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- [Python 3.12](https://www.python.org/downloads/) 以上。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。

## プロジェクト スターター ファイルをダウンロードして Azure Key Vault をデプロイする

このセクションでは、アプリのスターター ファイルをダウンロードし、スクリプトを使ってサンプル シークレット含む Azure Key Vault を、サブスクリプションにデプロイします。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/key-vault-python.zip
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

1. 次のコマンドを実行して、演習に必要なリソース プロバイダーがご自分のサブスクリプションにあることを確認します。

    ```
    az provider register --namespace Microsoft.KeyVault
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

1. スクリプトの実行中に、「**1**」と入力して **1. Create Key Vault** オプションを起動します。

    このオプションは、リソースグループがまだ存在していなければ作成し、RBAC 認証が有効な Azure Key Vault をデプロイします。 RBAC 認証は、従来のアクセス ポリシーではなく、コンテナー シークレットへのアクセスを制御するための推奨モデルです。

1. 「**2**」と入力して、**2. Assign role** オプションを実行します。 これにより、Key Vault Secrets Officer ロールがアカウントに割り当てられるため、Microsoft Entra 認証を使用してシークレットの読み取り、作成、更新、削除を行うことができます。

1. 「**3**」と入力して、**3. Store secrets** オプションを実行します。 これにより、コンテナー内に 2 つのサンプル シークレットが保存されます。モデル エンドポイントの API キー (**openai-api-key**) とデータベース接続文字列 (**cosmosdb-connection-string**) です。 両方とも、環境とサービスの識別のためのメタデータがタグ付けされます。

1. 「**4**」と入力して、**4. Check deployment status** オプションを実行します。 コンテナーの状態が **Succeeded** と表示され、ロールが割り当てられ、シークレットが保存されていることを確認したら続行します。 コンテナーのプロビジョニングがまだ完了していない場合は、しばらく待ってからもう一度試してみてください。

1. 「**5**」と入力して、**5. Retrieve connection info** オプションを実行します。 これにより、アプリが必要とする Key Vault の URL を含む環境変数ファイルが作成されます。

1. 「**6**」と入力して、デプロイ スクリプトを終了します。

1. 適切なコマンドを実行して、前の手順で作成したファイルから環境変数をターミナル セッションに読み込みます。

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

このセクションでは、*keyvault_functions.py* ファイルにコードを追加して Key Vault のシークレット管理機能を完成させます。 *app.py* 内の Flask アプリは、これらの関数を呼び出し、結果をブラウザーに表示します。 このアプリは、演習の後半で実行します。

1. *client/keyvault_functions.py* ファイルを開き、コードの追加を開始します。

>**注:** アプリケーションに追加するコード ブロックは、コードのそのセクションのコメントと一致する必要があります。

### シークレットを取得するコードを追加する

このセクションでは、コンテナーから 2 つのシークレットを取得し、そのメタデータを返すコードを追加します。 この関数は、シークレット値、バージョン識別子、コンテンツ タイプ、作成日、カスタム タグへのアクセス方法を示します。

この関数は、シークレット名ごとに **get_secret()** を呼び出し、これにより、シークレット値とメタデータを含むプロパティ オブジェクトの両方が返されます。 これは、シークレットの欠落の場合は **ResourceNotFoundError**、認可またはネットワークの問題の場合は **HttpResponseError** を処理します。 切り捨てられた値により、完全な資格情報が UI に表示されることを防ぎつつ、シークレットが取得されたことを確認できます。

1. **# BEGIN RETRIEVE SECRETS FUNCTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def retrieve_secrets():
        """Retrieve secrets and display their metadata."""
        client = get_client()
        results = []

        secret_names = ["openai-api-key", "cosmosdb-connection-string"]

        for name in secret_names:
            try:
                # get_secret returns the secret value and its properties,
                # including version, content type, creation date, and tags
                secret = client.get_secret(name)
                results.append({
                    "name": secret.name,
                    "value": secret.value[:20] + "..." if len(secret.value) > 20 else secret.value,
                    "version": secret.properties.version,
                    "content_type": secret.properties.content_type,
                    "created_on": str(secret.properties.created_on),
                    "tags": secret.properties.tags or {},
                    "status": "retrieved"
                })
            except ResourceNotFoundError:
                results.append({
                    "name": name,
                    "value": None,
                    "version": None,
                    "content_type": None,
                    "created_on": None,
                    "tags": {},
                    "status": "not found"
                })
            except HttpResponseError as e:
                results.append({
                    "name": name,
                    "value": None,
                    "version": None,
                    "content_type": None,
                    "created_on": None,
                    "tags": {},
                    "status": f"error: {e.message}"
                })

        return results
    ```

1. 少し時間をかけて  コードを確認しましょう。

### シークレット プロパティを一覧表示するコードを追加する

このセクションでは、コンテナー内のすべてのシークレットのプロパティを、値を取得せずに一覧表示するコードを追加します。 これは、名前、有効な状態、コンテンツ タイプ、タイムスタンプなどのメタデータのみを公開して、最小限の特権の原則に従います。

この関数は、シークレット プロパティ オブジェクトのイテラブルを返す **list_properties_of_secrets()** を呼び出します。 **get_secret()** とは異なり、このメソッドはシークレット値を返さないため、内容にアクセスせずに存在するシークレットを知る必要がある在庫管理や監査操作がより安全になります。

1. **# BEGIN LIST SECRETS FUNCTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def list_secret_properties():
        """List all secret properties without retrieving values."""
        client = get_client()
        results = []

        # list_properties_of_secrets returns metadata for every secret
        # in the vault without exposing the secret values, which follows
        # the principle of least privilege
        for prop in client.list_properties_of_secrets():
            results.append({
                "name": prop.name,
                "enabled": prop.enabled,
                "content_type": prop.content_type,
                "created_on": str(prop.created_on),
                "updated_on": str(prop.updated_on)
            })

        return results
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### 新しいシークレット バージョンを作成するコードを追加する

このセクションでは、認証情報のローテーションをシミュレートする新しいシークレットのバージョンを作成するためのコードを追加します。 この関数は現在のバージョンを取得し、**set_secret()** を使って新しい値を書き込み、その後もう一度シークレットを取得して更新を確認します。

この関数は、**set_secret()** を使用して既存のシークレット名の新しい値を書き込みます。これにより、前のシークレット名を保持しつつ新しいバージョンが自動的に作成されます。 以前のバージョンは、バージョン ID で引き続きアクセス可能ですが、バージョン パラメーターのない **get_secret()** では常に最新のバージョンが返されます。 また、この関数は、ローテーション メタデータを追跡するために、更新されたタグを新しいバージョンにアタッチします。

1. **# BEGIN CREATE SECRET VERSION FUNCTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def create_secret_version(secret_name, new_value):
        """Create a new version of a secret and verify the update."""
        client = get_client()

        # Retrieve the current version before updating
        try:
            current = client.get_secret(secret_name)
            old_version = current.properties.version
            old_value = current.value[:20] + "..." if len(current.value) > 20 else current.value
        except ResourceNotFoundError:
            old_version = None
            old_value = None

        # set_secret creates a new version of the secret. The previous
        # version is preserved and can still be retrieved by version ID.
        client.set_secret(
            secret_name,
            new_value,
            content_type="text/plain",
            tags={"environment": "development", "rotated": "true"}
        )

        # Confirm the update by retrieving the secret again —
        # get_secret always returns the latest version
        confirmed = client.get_secret(secret_name)

        return {
            "name": secret_name,
            "old_version": old_version,
            "old_value": old_value,
            "new_version": confirmed.properties.version,
            "new_value": confirmed.value[:20] + "..." if len(confirmed.value) > 20 else confirmed.value,
            "created_on": str(confirmed.properties.created_on),
            "tags": confirmed.properties.tags or {}
        }
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### キャッシュされたシークレット取得のためのコードを追加する

このセクションでは、時間ベースのキャッシュを実装するコードを追加して、シークレットが頻繁にアクセスされる場合に Key Vault API 呼び出しの数を減らします。 キャッシュは、構成可能な Time to Live (TTL) とともにシークレット値をメモリに保存し、キャッシュのヒットとミスを追跡します。

この関数は、経過時間の追跡に **time.monotonic()** を使用して、30 秒の TTL を持つディクショナリ ベースのキャッシュを作成します。 2 つのシークレットにアクセスするための 5 回のラウンドをシミュレートします。 最初のラウンドではキャッシュ ミスが発生します。これは、キャッシュが空から始まり、返すべきエントリがないためであり、コードは Key Vault から各シークレットを取得して保存します。 TTL 内の次のラウンドでは、キャッシュされたエントリが検索され、それらが API を呼び出さずに返されます。 アクセス ログには各ヒットまたはミスが表示され、サマリーに API 呼び出しの合計数と合計アクセス数が報告されて、効率の向上が示されます。

1. **# BEGIN CACHED RETRIEVAL FUNCTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def cached_retrieval():
        """Demonstrate time-based caching to reduce Key Vault API calls."""
        client = get_client()
        cache = {}
        cache_ttl = 30
        vault_calls = 0
        access_log = []

        secret_names = ["openai-api-key", "cosmosdb-connection-string"]

        # Simulate five rounds of secret access. The first round fetches
        # from Key Vault (cache miss), and subsequent rounds return the
        # cached value if the TTL has not expired.
        for i in range(5):
            for name in secret_names:
                cached = cache.get(name)
                now = time.monotonic()

                if cached and (now - cached["timestamp"]) < cache_ttl:
                    access_log.append({
                        "round": i + 1,
                        "secret": name,
                        "result": "cache hit",
                        "value": cached["value"]
                    })
                else:
                    secret = client.get_secret(name)
                    vault_calls += 1
                    truncated = secret.value[:20] + "..." if len(secret.value) > 20 else secret.value
                    cache[name] = {
                        "value": truncated,
                        "timestamp": now
                    }
                    access_log.append({
                        "round": i + 1,
                        "secret": name,
                        "result": "cache miss",
                        "value": truncated
                    })

        return {
            "access_log": access_log,
            "vault_calls": vault_calls,
            "total_accesses": len(access_log),
            "cache_ttl_seconds": cache_ttl
        }
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

1. 次のコマンドを使用して、Python 環境をアクティブ化します。 **注:** Linux/macOS では Bash コマンドを使用します。 Windows では、PowerShell コマンドを使用します。 Windows で Git Bash を使っている場合は、**source .venv/Scripts/activate** を使用します。

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

このセクションでは、完成した Flask アプリケーションを実行してさまざまな Key Vault シークレットの管理操作を行います。 このアプリには、シークレットの取得、プロパティの一覧表示、新しいバージョンの作成、キャッシュ取得のテストを行う Web インターフェイスが用意されています。

1. ターミナルで次のコマンドを実行して、アプリを起動します。 コマンドを実行する前に、演習の前半のコマンドを参照して、必要に応じて環境をアクティブ化します。 *client* ディレクトリから移動した場合は、まず **cd client** を実行します。

    ```
    python app.py
    ```

1. ブラウザーを開き、`http://localhost:5000` に移動してアプリにアクセスします。

1. 左側のパネルで、**[Retrieve Secrets]** を選択します。 これにより、コンテナーに保存されている 2 つのシークレットが取得され、そのメタデータが右側のパネルに表示されます。これには、シークレット名、切り捨てられた値、バージョン識別子、コンテンツ タイプ、作成日、カスタム タグが含まれます。 両方のシークレットの状態が **retrieved** になります。

1. **[List Secret Properties]** を選択します。 これは、コンテナー内のすべてのシークレットの値を公開せずにプロパティを一覧表示します。 結果には、各シークレットの名前、有効な状態、コンテンツ タイプ、作成日、最終更新日が表示されます。 この操作は在庫管理や監査のシナリオに役立ちます。

1. **[Create New Version]** を選択します。 これにより、資格情報のローテーションをシミュレートしてランダムに生成された値を持つ **openai-api-key** シークレットの新しいバージョンが作成されます。 結果には、以前のバージョンと値が新しいバージョンと値と共に表示され、**set_secret()** が古いバージョンを保持しながら新しいバージョンを作成することを確認できます。

1. 左パネルで **[Retrieve Secrets]** を選択して、シークレットが更新されたことを確認します。

1. **[Run Cached Retrieval]** を選択します。 これにより、30 秒の TTL キャッシュを使用して両方のシークレットにアクセスする 5 回のラウンドがシミュレートされます。 最初のラウンドでは、値が Key Vault からフェッチされるため、2 つのキャッシュ ミス (シークレットごとに 1 つ) が表示されます。 残りのラウンドでは、TTL の期限が切れていないためキャッシュ ヒットが表示されます。 サマリーによると、Key Vault API 呼び出しは合計 10 回のアクセスで 2 回のみでした。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループとグループ内のすべてのリソースを削除します。 **\<rg-name>** を、演習の前半で選択した名前に置き換えます。 このコマンドにより、Azure でバックグラウンド タスクが起動され、リソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の実行中に問題が発生した場合は、次のトラブルシューティング手順をお試しください。

**Azure Key Vault のデプロイを確認する**
- [Azure portal](https://portal.azure.com) に移動してリソース グループを見つけます。
- Key Vault の **[プロビジョニングの状態]** が **[成功]** と表示されていることを確認します。
- コンテナーで RBAC 認可 (アクセス ポリシー モードではないもの) が有効になっていることを確認します。

**シークレットを確認する**
- デプロイ スクリプトの **Check deployment status** オプションを実行して、シークレットが正常に保存されていることを確認します。
- シークレットが欠落している場合は、もう一度 **Store secrets** オプションを実行してみてください。

**コードの完全性とインデントを確認する**
- すべてのコード ブロックが、*keyvault_functions.py* 内の適切な BEGIN/END コメント マーカーの間にある正しいセクションに追加されていることを確認します。
- Python のインデントが一貫していること (タブではなくスペースを使用していること)、およびすべてのコードが関数内で正しく配置されていることを確認します。
- 指定されたセクションの外部でコードが誤って削除または変更されていないことを確認します。

**環境変数を確認する**
- *.env* ファイルがプロジェクトのルートに存在し、**KEY_VAULT_URL** 値が含まれていることを確認します。
- **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行して環境変数をターミナル セッションに読み込んだことを確認します。
- 変数が空の場合は、**source .env** (Bash) または **. .\.env.ps1** (PowerShell) をもう一度実行します。

**認証を確認する**
- **az account show** を実行して、Azure CLI にログインしていることを確認します。
- Azure portal でロールの割り当てを確認するか、デプロイ スクリプトのオプションを実行してロールをもう一度割り当てて、Key Vault Secrets Officer ロールがアカウントに割り当てられていることを確認します。

**Python 環境と依存関係を確認する**
- アプリを実行する前に、仮想環境がアクティブになっていることを確認します。
- **pip list** を実行して、*requirements.txt* のすべてのパッケージが正常にインストールされたことを確認します。
