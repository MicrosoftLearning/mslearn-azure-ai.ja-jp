---
lab:
  topic: App secrets and configuration
  title: Azure App Configuration から設定とシークレットを取得する
  description: Python SDK を使って、Azure App Configuration と Key Vault を使って構成設定を読み込み、一覧表示し、動的に更新する方法を学びます。
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure App Configuration
---

# Azure App Configuration から設定とシークレットを取得する

AI アプリケーションは、モデル エンドポイントやバッチ サイズのような非機密構成と API キーのような機密性の高い認証情報の両方に依存しています。 Azure App Configuration は、ラベルベースの環境のオーバーライド、シークレットの Key Vault 参照、センチネルベースの動的更新を使用してこれらの設定を管理するための一元的なストアを提供します。これにより、アプリケーションは再起動せずに構成の変更を取得できます。

この演習では、サンプル設定が事前に読み込まれた Azure App Configuration ストアと Key Vault をデプロイし、Azure SDK を使用して主要な構成管理パターンを実演する Python Flask Web アプリケーションを構築します。 ラベル スタックと Key Vault (キー コンテナー) 参照の自動解決を使用して設定を読み込み、すべての設定プロパティとメタデータを一覧表示し、動的に変更を検出するために Sentinel ベースの更新をトリガーします。

この演習で実行されるタスク:

- プロジェクトのスターター ファイルをダウンロードする
- サンプル設定で Azure App Configuration ストアと Key Vault を作成する
- スターター ファイルにコードを追加してアプリを完成させる
- 構成操作を行うためにアプリを実行する

この演習の所要時間は約 **30** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- [Python 3.12](https://www.python.org/downloads/) 以上。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。

## プロジェクト スターター ファイルをダウンロードして Azure App Configuration をデプロイする

このセクションでは、アプリのスターター ファイルをダウンロードし、スクリプトを使用してサンプル設定を含む Azure App Configuration ストアと Key Vault を、サブスクリプションにデプロイします。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/app-config-python.zip
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

1. 次のコマンドを実行して、演習に必要なリソース プロバイダーがご自分のサブスクリプションにあることを確認します。

    ```
    az provider register --namespace Microsoft.AppConfiguration
    az provider register --namespace Microsoft.KeyVault
    ```

1. ターミナルで次のコマンドを実行して、デプロイ スクリプトを起動します。

    ```
    python azdeploy.py
    ```

1. スクリプトの実行中に、「**1**」と入力して、**1. Create App Configuration** オプションを起動します。

    このオプションは、リソース グループがまだ存在していなければ作成し、Azure App Configuration ストアをデプロイします。 App Configuration は、コードとは別にアプリケーション設定を管理するための一元化されたサービスを提供します。

1. 「**2**」と入力して、**2. Create Key Vault** オプションを実行します。 これにより、RBAC 認証が有効な Azure Key Vault が作成されます。 Key Vault は、App Configuration が参照する API キーなどの機密値を安全に保存します。

1. 「**3**」と入力して、**3. Assign roles** オプションを実行します。 これにより、App Configuration Data Owner ロールと Key Vault Secrets Officer があなたのアカウントに割り当てられるので、Microsoft Entra 認証を使って設定やシークレットの読み書き、作成、更新を行うことができます。

1. 「**4**」と入力して、**4. Store settings** オプションを実行します。 これにより、環境固有の設定の既定の (ラベルなし) 値や運用ラベル付きオーバーライドを含む構成設定が App Configuration ストアに保存されます。 また、シークレットを Key Vault に保存し、シークレットを指す App Configuration に Key Vault 参照を作成します。 最後に、動的更新に使用されるセンチネル キーを作成します。

1. 「**5**」と入力して、**5. Check deployment status** オプションを実行します。 App Configuration ストアと Key Vault の両方に **Succeeded** と表示され、ロールが割り当てられ、設定が保存されていることを確認してから続行します。 リソースがまだプロビジョニング中の場合は、少し待ってからもう一度試してみてください。

1. 「**6**」と入力して、**6. Retrieve connection info** オプションを実行します。 これにより、アプリに必要な App Configuration エンドポイント URL を持つ環境変数ファイルが作成されます。

1. 「**7**」と入力して、デプロイ スクリプトを終了します。

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

このセクションでは、App Configuration 管理機能を完成させるコードを *appconfig_functions.py* ファイルに追加します。 *app.py* 内の Flask アプリは、これらの関数を呼び出し、結果をブラウザーに表示します。 このアプリは、演習の後半で実行します。

1. *client/appconfig_functions.py* ファイルを開き、コードの追加を開始します。

>**注:** アプリケーションに追加するコード ブロックは、コードのそのセクションのコメントと一致する必要があります。

### 設定を読み込むコードを追加する

このセクションでは、ラベルの積み重ねと自動 Key Vault 参照解決を使用して、App Configuration ストアからすべての構成設定を読み込むコードを追加します。 この関数は、ラベルなしの既定値を運用ラベル付きのオーバーライドとマージし、透過的に Key Vault 参照を解決するプロバイダーを作成します。

この関数は **、2 つの** SettingSelector **エントリを持つ load()** を呼び出します。1 つ目の関数は、(null ラベル フィルター **\0**を使用して) すべてのラベルのない設定を選択し、2 つ目の設定が運用ラベル付けされたすべての設定を選択します。 運用セレクターが 2 番目に表示されるため、その値が一致するキーの既定値を上書きします。 **AzureAppConfigurationKeyVaultOptions** パラメーターは、同じ資格情報を使用して Key Vault 参照を自動的に解決するようにプロバイダーに指示します。これにより、アプリケーションは参照 URI ではなく実際のシークレット値を受け取ります。

>**ヒント:** コードの適切なインデントを維持するには、左余白 (列 1) のコード揃えを貼り付け、貼り付けられた行をすべて選択して、**Tab** キーを押し、ブロックを**開始/終了**のマーカーに合わせます。 必要に応じて **Shift + Tab** キーを押してインデントを戻してください。

1. **# BEGIN LOAD SETTINGS FUNCTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def load_settings():
        """Load all settings with label stacking and Key Vault reference resolution."""
        provider = get_provider()
        results = []

        # The provider resolves Key Vault references automatically and
        # applies label stacking: Production-labeled values override
        # unlabeled defaults for matching keys
        known_keys = [
            "OpenAI:Endpoint",
            "OpenAI:DeploymentName",
            "OpenAI:ApiKey",
            "Pipeline:BatchSize",
            "Pipeline:RetryCount",
            "Sentinel"
        ]

        for key in known_keys:
            try:
                value = provider[key]
                is_secret = key == "OpenAI:ApiKey"
                display_value = value[:10] + "..." if is_secret and len(value) > 10 else value
                results.append({
                    "key": key,
                    "value": display_value,
                    "type": "Key Vault reference" if is_secret else "configuration",
                    "status": "loaded"
                })
            except KeyError:
                results.append({
                    "key": key,
                    "value": None,
                    "type": "unknown",
                    "status": "not found"
                })

        return results
    ```

1. 少し時間をかけて  コードを確認しましょう。

### 設定プロパティを一覧表示するコードを追加する

このセクションでは、App Configuration ストア内のすべての設定のプロパティを一覧表示するコードを追加します。 ラベルをマージして Key Vault 参照を解決する **load()** 関数とは異なり、この関数は、すべてのラベルとコンテンツ タイプを含め、個々の設定エントリをすべて表示した生のストレージ ビューを表示します。

この関数は、管理クライアントで **list_configuration_settings()** を呼び出します。これにより、キー、ラベル、コンテンツ タイプ、最終更新日時のタイムスタンプ、読み取り専用の状態などのメタデータを持つ設定オブジェクトのイテラブルが返されます。 これは、個別のラベル付けされていないエントリや運用ラベル付きのエントリを含めて、保存されている内容を正確に確認する必要がある在庫管理や監査業務に役立ちます。

1. **# BEGIN LIST SETTINGS FUNCTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def list_setting_properties():
        """List all setting properties from the App Configuration store."""
        client = get_client()
        results = []

        # list_configuration_settings returns every setting in the store
        # including all labels, showing the raw storage view rather than
        # the merged view that load() provides
        for setting in client.list_configuration_settings():
            results.append({
                "key": setting.key,
                "label": setting.label or "(no label)",
                "content_type": setting.content_type or "—",
                "last_modified": str(setting.last_modified) if setting.last_modified else "—",
                "read_only": setting.read_only
            })

        return results
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### 動的更新用のコードを追加する

このセクションでは、センチネルベースの動的更新を示すコードを追加します。 この関数は設定を更新し、新しいセンチネル値を設定し、その後プロバイダーで **refresh()** を呼び出して、アプリケーションを再起動せずに設定を再度読み込みます。

この関数は現在のプロバイダー値をキャプチャし、管理クライアントを使って **Pipeline:BatchSize** を新しいランダム値で更新し、**センチネル** キーを新しいタイムスタンプ値に設定します。 センチネルは変更シグナルとして機能します。プロバイダーはそれを監視し、その値が変更されると、**refresh()** の呼び出しによってすべての設定の再読み込みがトリガーされます。 関数は更新間隔が経過するのを短時間待機した後、**refresh()** を呼び出し、前後の値を比較して更新が伝達されたことを確認します。

1. **# BEGIN REFRESH CONFIGURATION FUNCTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def refresh_configuration():
        """Demonstrate sentinel-based dynamic refresh of configuration settings."""
        provider = get_provider()
        client = get_client()

        # Capture current values before the change
        tracked_keys = ["Pipeline:BatchSize"]
        before = {}
        for key in tracked_keys:
            try:
                before[key] = provider[key]
            except KeyError:
                before[key] = "—"

        # Update Pipeline:BatchSize with a new value to simulate a
        # configuration change, then increment the Sentinel key to
        # signal the provider that settings have changed
        import random
        new_batch = str(random.randint(100, 999))

        setting = ConfigurationSetting(
            key="Pipeline:BatchSize",
            value=new_batch,
            label="Production",
            content_type="text/plain"
        )
        client.set_configuration_setting(setting)

        # Update the Sentinel to signal the provider that settings have changed.
        # Using a timestamp ensures the value is always different from whatever
        # the provider has cached, even if settings were reset externally.
        new_sentinel = str(int(time.time()))

        sentinel_setting = ConfigurationSetting(
            key="Sentinel",
            value=new_sentinel
        )
        client.set_configuration_setting(sentinel_setting)

        # Wait briefly for the refresh interval to elapse, then call
        # refresh() to reload settings from the store
        time.sleep(2)
        provider.refresh()

        # Capture values after the refresh
        after = {}
        for key in tracked_keys:
            try:
                after[key] = provider[key]
            except KeyError:
                after[key] = "—"

        settings = []
        for key in tracked_keys:
            settings.append({
                "key": key,
                "before": before[key],
                "after": after[key],
                "changed": before[key] != after[key]
            })

        return {
            "settings": settings,
            "sentinel_value": new_sentinel,
            "new_batch_size": new_batch,
            "batch_size_updated": after["Pipeline:BatchSize"] == new_batch
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

このセクションでは、完成した Flask アプリケーションを実行してさまざまな App Configuration の管理操作を行います。 このアプリは設定の読み込み、プロパティの一覧表示、動的更新のテストを行うことができる Web インターフェイスを提供します。

1. ターミナルで次のコマンドを実行して、アプリを起動します。 コマンドを実行する前に、演習の前半のコマンドを参照して、必要に応じて環境をアクティブ化します。 *client* ディレクトリから移動した場合は、まず **cd client** を実行します。

    ```
    python app.py
    ```

1. ブラウザーを開き、`http://localhost:5000` に移動してアプリにアクセスします。

1. 左側のパネルで、**[Load Settings]** を選択します。 これにより、ラベルの積み重ねと Key Vault の参照解決を含むすべての構成設定が読み込まれます。 結果には各設定のキー、値、タイプが表示されます。 **configuration**とラベル付けされた設定は通常の App Configuration 値であり、**Key Vault reference** は、Key Vault のシークレットから解決されたことを示します。

1. **[List Setting Properties]** を選択します。 これにより、ラベル付けされていない既定値と運用ラベル付きのオーバーライドの両方を含めて、ストア内のすべての個々の設定エントリが個々の行で覧表示されます。 結果には各設定のキー、ラベル、コンテンツ タイプ、最終更新のタイムスタンプ、読み取り専用の状態が表示されます。 **Pipeline:BatchSize** が 2 回表示されることに注目します。1 回はラベルなし (値 10)、もう 1 回は**運用**ラベル (値 200) です。 **[Load Settings]** の結果には、運用ラベル付きのオーバーライドがラベルなしの既定値よりも優先されたため、200 と表示されました。

1. **[Refresh Configuration]** を選択します。 これは、センチネルベースの動的更新を示します。 この関数は、**Pipeline:BatchSize** を新しいランダムな値で更新し、**Sentinel** キーを新しいタイムスタンプに設定し、短時間待機してから、プロバイダーで **refresh()** を呼び出します。 結果は追跡設定の前後の値を示しており、プロバイダーがアプリケーションを再起動せずに変更を検出したことを確認できます。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループと、そのグループ内のすべてのリソースを削除します。 **\<rg-name>** は、この演習で選択した名前に置き換えてください。 このコマンドを実行すると Azure の中でバックグラウンド タスクが起動されてリソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の実行中に問題が発生した場合は、次のトラブルシューティング手順をお試しください。

**Azure App Configuration のデプロイを確認する**
- [Azure portal](https://portal.azure.com) に移動してリソース グループを見つけます。
- App Configuration ストアの **[プロビジョニングの状態]** が **[成功]** と表示されていることを確認します。
- Key Vault の **[プロビジョニングの状態]** が **[成功]** と表示され、RBAC 認証が有効になっていることを確認します。

**設定を確認する**
- デプロイ スクリプトの **Check deployment status** オプションを実行して、設定が正常に保存されていることを確認します。
- 設定が不足している場合は、もう一度 **Store settings** オプションを実行します。

**コードの完全性とインデントを確認する**
- すべてのコード ブロックが、*appconfig_functions.py* 内の適切な BEGIN/END コメント マーカーの間にある正しいセクションに追加されていることを確認します。
- Python のインデントが一貫していること (タブではなくスペースを使用していること)、およびすべてのコードが関数内で正しく配置されていることを確認します。
- 指定されたセクションの外部でコードが誤って削除または変更されていないことを確認します。

**環境変数を確認する**
- *.env* ファイルがプロジェクトのルートに存在し、**AZURE_APPCONFIG_ENDPOINT**値が含まれていることを確認します。
- **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行して環境変数をターミナル セッションに読み込んだことを確認します。
- 変数が空の場合は、**source .env** (Bash) または **. .\.env.ps1** (PowerShell) をもう一度実行します。

**認証を確認する**
- **az account show** を実行して、Azure CLI にログインしていることを確認します。
- Azure portal でロールの割り当てを確認するか、デプロイ スクリプトのオプションを実行してロールをもう一度割り当てて、App Configuration Data Owner ロールと Key Vault Secrets Officer ロールがアカウントに割り当てられていることを確認します。

**Python 環境と依存関係を確認する**
- アプリを実行する前に、仮想環境がアクティブになっていることを確認します。
- **pip list** を実行して、*requirements.txt* のすべてのパッケージが正常にインストールされたことを確認します。
