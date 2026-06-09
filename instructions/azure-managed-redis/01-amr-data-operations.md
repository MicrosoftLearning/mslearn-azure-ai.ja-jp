---
lab:
  topic: Azure Managed Redis
  title: Azure Managed Redis でデータ操作を実行する
  description: Python の redis-py ライブラリを使って Azure Managed Redis でデータ操作を行う方法を学びます。
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Managed Redis
---

# Azure Managed Redis でデータ操作を実行する

この演習では、Azure Managed Redis リソースを作成し、**redis-py** ライブラリを使用して一般的なデータ操作を実行する Python コンソール アプリケーションをビルドします。 Redis のハッシュデータ構造を使って、キーと値のペアを保存および取得し、Time to Live (TTL) 設定でキーの有効期限を管理し、キャッシュからキーを削除します。

この演習で実行されるタスク:

- プロジェクトのスターター ファイルをダウンロードする
- Azure Managed Redis リソースを作成する
- スターター ファイルにコードを追加してコンソール アプリを完成させる
- コンソールアプリを実行してデータ操作を行う

この演習の所要時間は約 **30** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- [Python 3.12](https://www.python.org/downloads/) 以上。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- Azure CLI **redisenterprise** 拡張機能。 **az extension add --name redisenterprise** コマンドを実行するとインストールできます。

## プロジェクト スターター ファイルをダウンロードして Azure Managed Redis をデプロイする

このセクションでは、コンソール アプリのスターター ファイルをダウンロードし、スクリプトを使って Azure Managed Redis のサブスクリプションへのデプロイを初期化します。 Azure Managed Redis のデプロイが完了するまでに 5 分から 10 分かかります。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/amr-data-operations-python.zip
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

1. 次のコマンドを実行して、Azure Managed Redis に必要なリソース プロバイダーがご自分のサブスクリプションにあることを確認します。

    ```
    az provider register --namespace Microsoft.Cache
    ```

1. 次のコマンドを実行して、Azure CLI 用の **redisenterprise** 拡張機能をインストールします。

    ```
    az extension add --name redisenterprise
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

1. スクリプトの実行中に、「**1**」と入力して **1. Create Azure Managed Redis resource** オプションを起動します。

    このオプションは、リソースグループがまだ存在していなければ作成して、Azure Managed Redis のデプロイを開始します。 このプロセスは Azure のバックグラウンドタスクとして完了します。

1. コンソールに次のメッセージが表示されたら、**Enter** を選択してメニューに戻り、次に **[4]** を選択してスクリプトを終了します。 後でもう一度スクリプトを実行して、デプロイ状況を確認し、プロジェクト用の *.env* ファイルを作成します。

    Azure Managed Redis リソースが作成されており、完了までに 5 分から 10 分かかります。**

    演習の後半でメニューからデプロイ状況を確認できます。**


## Python 環境を構成する

このセクションでは、Python 環境を作成し、依存関係をインストールします。

1. VS Code ターミナルで次のコマンドを実行して、Python 環境を作成します。

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

## アプリの仕上げ

このセクションでは、コンソールアプリを完成させるために *main.py* スクリプトにコードを追加します。 Azure Managed Redis リソースが完全にデプロイされていることを確認し、*.env* ファイルを作成してから、演習の後半でアプリを起動します。

1. *main.py* ファイルを開き、コードの追加を開始します。

>**注:** アプリケーションに追加するコード ブロックは、コードのそのセクションのコメントと一致する必要があります。

### client connection を追加する

このセクションでは、redis-py ライブラリを使って Azure Managed Redis への接続を確立するコードを追加します。 このコードは環境変数から接続認証情報を取得し、安全な SSL 通信のために構成された Redis クライアント インスタンスを作成します。

1. **# BEGIN CONNECTION CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    try:
        # Azure Managed Redis with Non-Clustered policy uses standard Redis connection
        redis_host = os.getenv("REDIS_HOST")
        redis_key = os.getenv("REDIS_KEY")

        # Non-clustered policy uses standard Redis client connection
        r = redis.Redis(
            host=redis_host,
            port=10000,  # Azure Managed Redis uses port 10000
            ssl=True,
            decode_responses=True, # Decode responses to strings
            password=redis_key,
            socket_timeout=30,  # Add timeout for better reliability
            socket_connect_timeout=30,
        )

        print(f"Connected to Redis at {redis_host}")
        input("\nPress Enter to continue...")
        return r
    ```

### データを保存および取得するコードを追加する

このセクションでは、**hset** および **hgetall** コマンドを使って Redis のハッシュ データ構造を扱うコードを追加します。 **hset** メソッドは 1 つのキーの下に複数のフィールド値ペアを保存します。一方、**hgetall** は指定されたキーのすべてのフィールドと値を取得します。

1. **# BEGIN STORE AND RETRIEVE CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def store_hash_data(r, key, value) -> None:
        """Store hash data in Redis"""
        clear_screen()
        print(f"Storing hash data for key: {key}")
        result = r.hset(key, mapping=value) # Store hash data
        if result > 0: # New fields were added
            print(f"Data stored successfully under key '{key}' ({result} new fields added)")
        else:
            print(f"Data updated successfully under key '{key}' (all fields already existed)")
        input("\nPress Enter to continue...")

    def retrieve_hash_data(r, key) -> None:
        """Retrieve hash data from Redis"""
        clear_screen()
        print(f"Retrieving hash data for key: {key}")
        retrieved_value = r.hgetall(key) # Retrieve hash data
        if retrieved_value:
            print("\nRetrieved hash data:")
            for field, value in retrieved_value.items():
                print(f"  {field}: {value}")
        else:
            print(f"Key '{key}' does not exist.")

        input("\nPress Enter to continue...")
    ```

### 有効期限を設定し取得するためのコードを追加する

このセクションでは、**expire** および **ttl** コマンドを使ってキーの有効期限を管理するコードを追加します。 **expire** メソッドはキーに Time to Live (TTL) を設定し、指定された秒数後に自動的に有効期限切れにします。一方、**ttl** はキーの期限までの残りの時間を取得します。

1. **# BEGIN EXPIRATION CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def set_expiration(r, key) -> None:
        """Set an expiration time for a key"""
        clear_screen()
        print("Set expiration time for a key")
        # Set expiration time, 1 hour equals 3600 seconds
        expiration = int(input("Enter expiration time in seconds (default 3600): ") or 3600)
        result = r.expire(key, expiration) # Set expiration time
        if result:
            print(f"Expiration time of {expiration} seconds set for key '{key}'")
        else:
            print(f"Key '{key}' does not exist. Expiration not set.")

        input("\nPress Enter to continue...")

    def retrieve_expiration(r, key) -> None:
        """Retrieve the TTL of a key"""
        clear_screen()
        print(f"Retrieving the current TTL of {key}...")
        ttl = r.ttl(key) # Get current TTL
        if ttl == -2: # Key does not exist
            print(f"\nKey '{key}' does not exist.")
        elif ttl == -1: # No expiration set
            print(f"\nKey '{key}' has no expiration set (persists indefinitely).")
        else:
            print(f"\nCurrent TTL for '{key}': {ttl} seconds")
        input("\nPress Enter to continue...")
    ```

### データを削除するコードを追加する

このセクションでは、**delete** コマンドを使って Redis からキーを削除するコードを追加します。 **delete** メソッドは、キーとそれに付随する値をキャッシュから永久に削除し、メモリを解放し、データへのアクセスを停止させます。

1. **# BEGIN DELETE CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def delete_key(r, key) -> None:
        """Delete a key"""
        clear_screen()
        print(f"Deleting key: {key}...")
        result = r.delete(key) # Delete the key
        if result == 1:
            print(f"Key '{key}' deleted successfully.")
        else:
            print(f"Key '{key}' does not exist.")
        input("\nPress Enter to continue...")
    ```

1. 変更内容を *main.py* ファイルに保存します。

## リソースのデプロイを確認する

このセクションでは、デプロイ スクリプトをもう一度実行して、Azure Managed Redis のデプロイが完了したかどうかを確認し、エンドポイントとアクセス キーの値を持つ *.env* ファイルを作成します。

1. ターミナルで適切なコマンドを実行してデプロイ スクリプトを起動します。 前のターミナルを閉じた場合は、メニューの **[ターミナル] > [新しいターミナル]** を選択して新しいターミナルを開きます。

    **Bash**
    ```bash
    bash azdeploy.sh
    ```

    **PowerShell**
    ```powershell
    ./azdeploy.ps1
    ```

1. デプロイ メニューが表示されたら、「**2**」と入力して、**2. Check deployment status** オプションを実行します。 状態に **Succeeded** と表示されている場合は、次の手順に進みます。 そうでない場合は、数分待ってから、オプションをもう一度試してみてください。

1. デプロイの完了後、「**3**」と入力して **3. Create database and retrieve endpoint and access key** オプションを実行します。 これによりデータベースが作成され、アクセス キー認証が可能になり、エンド ポイントとアクセス キーが取得されます。 その後、それらの値を含めた *.env* ファイルが作成されます。

1. *.env* ファイルを確認して値が存在していることを確認し、「**4**」と入力してデプロイ スクリプトを終了します。

## コンソール アプリの実行

このセクションでは、完成したコンソールアプリケーションを実行して Redis のさまざまなデータ操作を行います。 このアプリには、メニュー駆動のインターフェイスが用意されており、ハッシュデータの保存、値の取得、キーの有効期限管理、キーの削除を行うことができます。

1. ターミナルで次のコマンドを実行して、コンソール アプリを起動します。 コマンドを実行する前に、演習の前半のコマンドを参照して、必要に応じて環境をアクティブ化します。

    ```
    python main.py
    ```

1. このアプリには、次のオプションがあります。 **1. Store hash data** を選択して開始します。

    ```
    1. Store hash data
    2. Retrieve hash data
    3. Set expiration
    4. Retrieve expiration (TTL)
    5. Delete key
    6. Exit
    ```

1. さまざまな操作を実行するには、残りのオプションを選択します。

>**注:** オプションは、任意の順番で実行できます。 たとえば、ハッシュ データを保存した後、有効期限情報を取得すると、キーに有効期限が設定されていないことがわかります。

アプリで使用されるモック ハッシュ データは **main()** 関数の冒頭で定義されています。 別のキーを使用するようにコードを更新したり、ハッシュ データに値を追加したりすることができます。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループとグループ内のすべてのリソースを削除します。 **\<rg-name>** を、演習の前半で選択した名前に置き換えます。 このコマンドにより、Azure でバックグラウンド タスクが起動され、リソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の実行中に問題が発生した場合は、次のトラブルシューティング手順をお試しください。

**Azure Managed Redis リソースのデプロイを確認する**
- [Azure portal](https://portal.azure.com) に移動してリソース グループを見つけます。
- Azure Managed Redis リソースの **[プロビジョニングの状態]** の表示が **[成功]** であることを確認します。
- リソースの **[公衆ネットワーク アクセス]** が有効で **[アクセス キー認証]** が **[有効]** に設定されているか確認します。

**コードの完全性とインデントを確認する**
- すべてのコード ブロックが、*main.py* 内の適切な BEGIN/END コメント マーカーの間にある正しいセクションに追加されていることを確認します。
- Python のインデントが一貫していること (タブではなくスペースを使用していること)、およびすべてのコードが関数内で正しく配置されていることを確認します。
- 指定されたセクションの外部でコードが誤って削除または変更されていないことを確認します。

**環境変数を確認する**
- *.env* ファイルがプロジェクト フォルダーに存在し、有効な **REDIS_HOST** と **REDIS_KEY** の数値が含まれていることを確認します。
- *.env*ファイルが*main.py*と同じディレクトリにあることを確認します。

**Python 環境と依存関係を確認する**
- アプリを実行する前に、仮想環境がアクティブになっていることを確認します。
- **pip list** を実行して、*requirements.txt* のすべてのパッケージが正常にインストールされたことを確認します。

