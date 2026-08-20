---
lab:
  topic: Azure Managed Redis
  title: Azure Managed Redis でセマンティック検索を実装する
  description: Azure Managed Redis で redis-py や RediSearch を使用して、埋め込みを含む製品ベクトルの保存、セマンティック検索インデックスの作成、類似性検索を行う方法を学びます。
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Managed Redis
---

# Azure Managed Redis でセマンティック検索を実装する

この演習では、Azure Managed Redis をデプロイし、Python Flask の Web アプリを完成させて、製品の埋め込みやメタデータを保存し、ベクトル インデックスを作成し、コサイン距離を使用した類似性検索を実行します。 Microsoft Entra ID に接続し、インデックスを作成し、製品ベクトルを保存し、ブラウザーベースのインターフェイスから類似製品のクエリを実行するコードを追加します。

この演習で実行されるタスク:

- プロジェクトのスターター ファイルをダウンロードする
- Azure Managed Redis リソースを作成する
- スターター ファイルにコードを追加してアプリを完成させる
- 製品を読み込み、ベクトルを保存し、類似性検索を実行するアプリを実行する

この演習の所要時間は約 **30** 分です。

## 開始する前に

このセクションでは、演習に必要な前提条件をレビューします。

演習を最後まで行うには、次のものが必要です。

- Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- [Python 3.12](https://www.python.org/downloads/) 以上。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- Azure CLI **redisenterprise** 拡張機能、バージョン 2.75.0 以上。 後のステップでこの拡張機能のインストールやアップグレードが行われます。

## プロジェクト スターター ファイルをダウンロードして Azure Managed Redis をデプロイする

このセクションでは、アプリのスターターファイルをダウンロードし、スクリプトを使って Azure Managed Redis のサブスクリプションへのデプロイを初期化します。 Azure Managed Redis のデプロイは完了までに 5 から 10 分かかるため、まずデプロイを開始し、プロビジョニング中にアプリにコードを追加します。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/amr-vector-query-python.zip
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

1. 次のコマンドを実行して、Azure Managed Redis に必要なリソース プロバイダーがご自分のサブスクリプションにあることを確認します。

    ```
    az provider register --namespace Microsoft.Cache
    ```

1. 次のコマンドを実行して、Azure CLI 用の **redisenterprise** 拡張機能をインストールまたはアップグレードします。 データベースで Microsoft Entra ID アクセスを構成するには、バージョン 2.75.0 以上が必要です。

    ```
    az extension add --upgrade --name redisenterprise
    ```

1. ターミナルで次のコマンドを実行して、デプロイ スクリプトを起動します。

    ```
    python azdeploy.py
    ```

1. スクリプトの実行中に、「**1**」と入力して **1. Create Azure Managed Redis resource** オプションを起動します。

    このオプションは、リソースグループがまだ存在していなければ作成して、Azure Managed Redis をデプロイします。 スクリプトは、5 から 10 分間かかるデプロイの完了を待機し、ターミナルにその結果を報告します。 スクリプトを実行したまま、次のセクションに進み、デプロイのプロビジョニング中にコードを追加します。 定期的に端末を確認してエラーをチェックします。

    デプロイが成功すると、次のような確認メッセージが表示され、メニューが返されます。

    Azure Managed Redis リソースが正常に作成されました: amr-exercise-\<hash>**

## アプリの仕上げ

このセクションでは、ベクトルの保存と検索操作を完了するコードを *client/vector_functions.py* ファイルに追加します。 *client/app.py* の Flask アプリはこれらの関数を呼び出してブラウザーからワークフローを実行します。 *client/app.py* を編集する必要はありません。 このアプリは、演習の後半で実行します。

1. *client/vector_functions.py* ファイルを開き、コードの追加を開始します。

>**注:** アプリケーションに追加するコード ブロックは、コードのそのセクションのコメントと一致する必要があります。

### Azure Managed Redis に接続するコードを追加する

このセクションでは、Microsoft Entra ID で認証する Redis クライアントを作成するコードを追加します。 Entra ID を使うと、アプリではアクセス キーが処理されません。

**get_client()** 関数は **REDIS_HOST** 環境変数から Redis エンドポイントを読み取り、**create_from_default_azure_credential()** を呼び出して資格情報プロバイダーを構築します。 プロバイダーは **DefaultAzureCredential** を使用して Microsoft Entra トークンを取得し、バックグラウンドで自動的に更新します。

>**ヒント:** コードの適切なインデントを維持するには、左余白 (列 1) のコード揃えを貼り付け、貼り付けられた行をすべて選択して、**Tab** キーを押し、ブロックを**開始/終了**のマーカーに合わせます。 必要に応じて **Shift + Tab** キーを押してインデントを戻してください。

1. **# BEGIN CONNECTION CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def get_client() -> redis.Redis:
        """Create a Redis client for Azure Managed Redis using Microsoft Entra ID."""
        redis_host = os.environ.get("REDIS_HOST")

        if not redis_host:
            raise ValueError("REDIS_HOST environment variable must be set")

        credential_provider = create_from_default_azure_credential(
            ("https://redis.azure.com/.default",),
        )

        return redis.Redis(
            host=redis_host,
            port=10000,
            ssl=True,
            decode_responses=False,
            credential_provider=credential_provider,
            socket_timeout=30,
            socket_connect_timeout=30,
        )
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### ベクトル インデックスを作成するコードを追加する

このセクションでは、類似性検索で使用する RediSearch インデックスを作成するコードを追加します。

**_create_vector_index()** 関数はテキスト フィールドと **embedding** というベクトル フィールドを定義します。 ベクトル フィールドは、HNSW アルゴリズムとコサイン距離および埋め込み次元 8 を使用してサンプル データをマッチングします。

1. **# BEGIN CREATE VECTOR INDEX CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def _create_vector_index(self):
        """Create a RediSearch index for product semantic search."""
        try:
            schema = (
                TextField("name"),
                TextField("category"),
                TextField("product_id"),
                VectorField(
                    "embedding",
                    "HNSW",
                    {
                        "TYPE": "FLOAT32",
                        "DIM": VECTOR_DIM,
                        "DISTANCE_METRIC": "COSINE",
                    },
                ),
            )

            definition = IndexDefinition(
                prefix=["product:"],
                index_type=IndexType.HASH,
            )

            self.r.ft(VECTOR_INDEX_NAME).create_index(
                fields=schema,
                definition=definition,
            )
        except redis.ResponseError as e:
            if "already exists" not in str(e):
                raise Exception(f"Error creating vector index: {e}")
        except Exception as e:
            raise Exception(f"Error creating vector index: {e}")
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### 製品ベクトルを保存するコードを追加する

このセクションでは、Redis で製品埋め込みとメタデータを保存するコードを追加します。

**store_product()** 関数は埋め込みリストを numpy を使用して **float32** バイトに変換し、埋め込みとメタデータを **hset()** を使用して Redis のハッシュに書き込みます。

1. **# BEGIN STORE PRODUCT CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def store_product(
        self,
        vector_key: str,
        vector: list[float],
        metadata: dict[str, str] | None = None,
    ) -> tuple[bool, str]:
        """Store or update a product hash containing embedding and metadata."""
        try:
            embedding = np.array(vector, dtype=np.float32)
            data: dict[str, Any] = {"embedding": embedding.tobytes()}

            if metadata:
                for key, value in metadata.items():
                    data[key] = str(value)

            result = self.r.hset(vector_key, mapping=data)
            if result > 0:
                return True, f"Product stored successfully under key '{vector_key}'"
            return True, f"Product updated successfully under key '{vector_key}'"
        except Exception as e:
            return False, f"Error storing product: {e}"
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

### 類似製品を検索するコードを追加する

このセクションでは、ベクトル インデックスに対して KNN 類似性検索を実行するコードを追加します。

**search_similar_products()** 関数はクエリ埋め込みをバイトに変換し、RediSearch の KNN クエリを構築し、スコア順に最も近い製品マッチを返します。

1. **# BEGIN SEARCH SIMILAR PRODUCTS CODE SECTION** というコメントを見つけ、コメントの下に次のコードを追加します。 コードの配置が適切かどうかを必ず確認してください。

    ```python
    def search_similar_products(
        self,
        query_vector: list[float],
        top_k: int = 3,
    ) -> tuple[bool, list[dict[str, Any]] | str]:
        """Run a KNN vector query against product embeddings."""
        try:
            query_bytes = np.array(query_vector, dtype=np.float32).tobytes()

            knn_query = (
                Query(f"*=>[KNN {top_k} @embedding $query_vec AS score]")
                .return_fields("name", "category", "product_id", "score")
                .sort_by("score")
                .dialect(2)
            )

            results = self.r.ft(VECTOR_INDEX_NAME).search(
                knn_query,
                query_params={"query_vec": query_bytes},
            )

            if results.total == 0:
                return False, "No products found in Redis. Load sample products first."

            similarities: list[dict[str, Any]] = []
            for doc in results.docs:
                similarities.append(
                    {
                        "key": doc.id,
                        "similarity": float(doc.score),
                        "product_id": doc.product_id.decode() if isinstance(doc.product_id, bytes) else doc.product_id,
                        "name": doc.name.decode() if isinstance(doc.name, bytes) else doc.name,
                        "category": doc.category.decode() if isinstance(doc.category, bytes) else doc.category,
                    }
                )

            return True, similarities
        except Exception as e:
            return False, f"Error searching products: {e}"
    ```

1. 変更を保存し、少し時間を取ってコードをレビューします。

## リソースのデプロイを確認する

このセクションでは、実行中のデプロイ スクリプトに戻って、ベクトル データベースを作成し、Microsoft Entra ID アクセスを構成し、Redis エンドポイントで環境変数ファイルを生成します。

1. デプロイ スクリプトが実行中のターミナルに戻ります。 スクリプトで Azure Managed Redis リソースが正常に作成されたと報告されたら、**Enter** キーを選択してデプロイ メニューに戻ります。

1. **2. Create database and configure access** (データベースを作成してアクセスを構成する) オプションを実行するには、「**2**」を入力します。 これにより、RediSearch モジュールでベクトル対応データベースが作成され、アカウントにデータ アクセス ポリシーが割り当てられ、*.env* と *.env.ps1* のファイルが **REDIS_HOST** を使用して作成されます。

1. 「**3**」と入力して、**[3 デプロイの状態を確認する]** オプションを最終確認として実行します。

1. 「**4**」を入力して、デプロイ スクリプトを終了します。

1. 適切なコマンドを実行して、前の手順で作成したファイルからターミナル セッションに環境変数を読み込みます。

    **Bash**
    ```bash
    source .env
    ```

    **PowerShell**
    ```powershell
    . .\.env.ps1
    ```

    >**注:** ターミナルは、開いたままにします。 ターミナルを閉じて新しいターミナルを作成する場合は、このコマンドをもう一度実行して、環境変数をもう一度読み込む必要があります。

## Python 環境を構成する

このセクションでは、クライアント ディレクトリに移動し、Python 環境を作成し、依存関係をインストールします。

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

このセクションでは、完成した Flask アプリケーションを実行し、単一の Web ページからベクトルの保存と類似性検索を実行します。

1. ターミナルで次のコマンドを実行して、アプリを起動します。 コマンドを実行する前に、演習の前半のコマンドを参照して、必要に応じて環境をアクティブ化し環境変数を読み込みます。 *client* ディレクトリから移動した場合は、まず **cd client** を実行します。

    ```
    python app.py
    ```

1. ブラウザーを開き、`http://localhost:5000` に移動してアプリにアクセスします。

### サンプル データを読み込み、類似性検索を実行する

このセクションでは、サンプルの製品埋め込みを読み込み、最初の類似性検索を実行します。

1. **[データ操作]** で **[サンプル製品を読み込む]** を選択します。

1. **[すべての製品のリスト]** を選択し、**[操作結果]** にプロダクト キーが表示されているかを確認します。

1. **[類似性検索]** で「**product:001**」と入力し、**[top_k]** を **[5]** に設定したままにしてから **[類似検索]** を選択します。

1. 返された製品とその距離スコアを **[操作結果]** で確認します。

### 新しい製品を保存して再度検索する

このセクションでは、新しい製品埋め込みを保存し、類似性検索を再度実行してニアレストネイバーがどのように変化するかを確認します。

1. **[製品の保存]** に次の値を入力し、**[製品の保存]** を選択します。

    プロダクト キー:

    ```
    product:011
    ```

    Embedding:

    ```
    [0.53, 0.63, 0.58, 0.37, 0.68, 0.47, 0.73, 0.57]
    ```

    メタデータ:

    ```
    product_id=011
    name=Gym Bag
    category=Sports
    ```

1. **[類似性検索]** で「**product:009**」と入力し、**[類似検索]** を選択します。

1. 結果をレビューし、製品の類似性の順序が新たに保存されたベクトルを反映しているかを確認してください。

### 製品を削除する

このセクションでは、プロダクト キーを 1 つ削除して、削除動作を検証します。

1. **[製品の削除]** で「**product:011**」と入力し、**[削除]** を選択します。

1. **[すべての製品のリスト]** を選択し、**[product:011]** が製品リストに表示されなくなったことを確認します。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループと、そのグループ内のすべてのリソースを削除します。 **\<rg-name>** は、この演習で選択した名前に置き換えてください。 このコマンドを実行すると Azure の中でバックグラウンド タスクが起動されてリソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の実行中に問題が発生した場合は、次のトラブルシューティング手順をお試しください。

**Azure Managed Redis リソースのデプロイを確認する**
- [Azure portal](https://portal.azure.com) に移動してリソース グループを見つけます。
- Azure Managed Redis リソースの **[プロビジョニングの状態]** の表示が **[成功]** であることを確認します。
- デプロイ スクリプトの **[デプロイの状態を確認する]** オプションを実行し、クラスターとデータベースの準備ができていることを確認してから、アプリを実行します。

**デプロイの失敗を解決する**
- デプロイが失敗した場合、多くの場合は選択した Azure リージョンの SKU の容量が一時的に不足していることが原因です。
- 画面上の指示に従ってスクリプトを終了し、スクリプト上部近くの **location** 変数を eastus2、australiaeast、canadacentral など別の地域に変更し、もう一度スクリプトを実行してオプション 1 を選択します。
- 失敗したリソースは次の試行の前に自動的に削除されます。

**認証とアクセスをチェックする**
- **az account show** を実行して、Azure CLI にログインしていることを確認します。
- デプロイ スクリプトの **Create database and configure access** オプションが正常に完了し、アカウントにデータベースに対するデータ アクセス ポリシーが適用されていることを確認します。
- アプリで認証エラーが報告された場合は、アクセス ポリシーの割り当てが有効になるまでに少し時間がかかる可能性があるため、しばらく待ってからもう一度試してみてください。

**コードの完全性とインデントを確認する**
- すべてのコード ブロックが、*client/vector_functions.py* 内の正しいセクションの、適切な BEGIN/END コメント マーカーの間に追加されていることを確認します。
- Python のインデントが一貫している (タブではなくスペースを使用している) ことを確認します。
- 指定されたセクションの外部でコードが誤って削除または変更されていないことを確認します。

**環境変数を確認する**
- *.env* と *.env.ps1* の両方のファイルがプロジェクトのルートに存在し、**REDIS_HOST** の値を含んでいることを確認します。
- **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行して環境変数をターミナル セッションに読み込みます。

**Python 環境と依存関係を確認する**
- アプリを実行する前に、仮想環境がアクティブになっていることを確認します。
- **pip list** を実行して、*client/requirements.txt* のすべてのパッケージが正常にインストールされたことを確認します。

**検索結果や製品の不足はありません**
- 類似性検索を実行する前に、サンプル製品を読み込んだことを確認します。
- **[すべての製品のリスト]** を選択してクエリ プロダクト キーがあることを確認します。
- インデックス構成とマッチングするための 8 つの数値が埋め込みに含まれていることを確認してください。
