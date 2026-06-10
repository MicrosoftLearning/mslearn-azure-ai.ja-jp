---
lab:
  topic: Azure Cosmos DB for NoSQL
  title: Azure Cosmos DB for NoSQL を使用して、セマンティック検索アプリケーションを構築します
  description: Azure Cosmos DB for NoSQL にベクトル類似検索を実装して、サポート チケット データのセマンティック検索を有効にする方法を学びます
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Cosmos DB
---

# Azure Cosmos DB for NoSQL を使用して、セマンティック検索アプリケーションを構築します

この演習では、Azure Cosmos DB for NoSQL を使用してベクトル類似性検索を実装します。 ベクトル検索では、テキストの高次元ベクトル表現を比較してセマンティック マッチングを行い、用語が完全に一致しない場合でも関連する結果を見つけることができます。 ベクトル埋め込みとインデックス作成ポリシーでコンテナーを構成し、事前に計算された埋め込みを含むサポート チケットを読み込み、**VectorDistance** 関数を使って類似性クエリを実行します。 このパターンでは、セマンティック検索を実行する AI アプリケーションを作成するための基盤が提供されます。たとえば、顧客の問題をより速やかに解決するために似たサポート ケースを見つける場合などです。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードし、デプロイ スクリプトを構成する
- ベクトル検索機能を含む Azure Cosmos DB for NoSQL アカウントをデプロイする
- ベクトル類似検索用の Python 関数を作成する
- ベクトル埋め込みとインデックス作成ポリシーを使ってコンテナーを作成する
- Flask Web アプリケーションを使ってベクトル検索をテストする

この演習の所要時間は約 **30** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- 必要な Azure サービスをデプロイする権限を持つ Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- [Python 3.12](https://www.python.org/downloads/) 以上。

## プロジェクト スターター ファイルをダウンロードして Azure サービスをデプロイする

このセクションでは、プロジェクト スターター ファイルをダウンロードし、スクリプトを使用して必要なサービスを Azure サブスクリプションにデプロイします。 Cosmos DB アカウントのデプロイが完了するまで数分かかる場合があります。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/cosmosdb-implement-vector-python.zip
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

1. 次のコマンドを実行して、演習に必要なリソース プロバイダーが自分のサブスクリプションにあることを確認します。

    ```azurecli
    az provider register --namespace Microsoft.DocumentDB
    ```

### Azure でリソースを作成する

このセクションでは、デプロイ スクリプトを実行して、ベクトル検索機能を備えた Cosmos DB アカウントをデプロイします。

1. プロジェクトのルート ディレクトリにいることを確認し、ターミナルで適切なコマンドを実行してデプロイ スクリプトを起動します。

    **Bash**
    ```bash
    bash azdeploy.sh
    ```

    **PowerShell**
    ```powershell
    ./azdeploy.ps1
    ```

1. スクリプト メニューが表示されたら、「**1**」と入力して **[Create Cosmos DB account]** オプションを開始します。 これにより、**EnableNoSQLVectorSearch** 機能とデータベースを含む Cosmos DB for NoSQL アカウントが作成されます。 **注:** デプロイが完了するまで 5 分から 10 分ほどかかります。

    >**重要:** デプロイを実行するターミナルは、演習が終了するまで開いたままにしてください。 ターミナルでのデプロイの進行中に、演習の次のセクションに進んでもかまいません。

## アプリを完成させる

このセクションでは、ベクトル検索関数とコンテナー セットアップ スクリプトの両方の Python コードを完成させます。 ベクトル関数は **VectorDistance** 関数を使って類似検索を実行し、セットアップ スクリプトは必要なベクトル ポリシーを使ってコンテナーを作成します。

### ベクトル検索関数を完成させる

このセクションでは、ベクトル類似検索を実行する関数を追加して、*vector_functions.py* ファイルを完成させます。 これらの関数は、**VectorDistance** 関数を使って、クエリ ベクトルとチケット埋め込みの類似度を計算します。 サポート アプリケーションでは、新しい問題が報告されたときに、これらの関数を使って似たチケットを検索できます。

1. VS Code で *client/vector_functions.py* ファイルを開きます。

1. **BEGIN STORE VECTOR DOCUMENT FUNCTION** というコメントを検索し、次に示すコードをこのコメントの直後に追加します。 この関数は、類似検索のためのベクトル埋め込みを含むサポート チケットを格納します。

    ```python
    def store_vector_document(
        document_id: str,
        chunk_id: str,
        content: str,
        embedding: list,
        metadata: dict = None
    ) -> dict:
        """Store a document with its vector embedding for similarity search."""
        container = get_container()

        # Build the document structure with embedding for vector search
        # The 'id' field is required by Cosmos DB and must be unique within the partition
        # The 'documentId' field is our partition key - chunks from the same source document
        # are stored together for efficient retrieval
        # The 'embedding' field contains the vector that will be used for similarity search
        document = {
            "id": chunk_id,
            "documentId": document_id,
            "content": content,
            "embedding": embedding,  # 256-dimensional vector for similarity search
            "metadata": metadata or {},
            "createdAt": datetime.utcnow().isoformat(),
            "chunkIndex": metadata.get("chunkIndex", 0) if metadata else 0
        }

        # upsert_item inserts if new, updates if exists (based on id + partition key)
        # This is idempotent - safe to call multiple times with the same data
        response = container.upsert_item(body=document)

        # Request Units (RUs) measure the cost of database operations in Cosmos DB
        # Tracking RU consumption helps optimize queries and estimate costs
        ru_charge = response.get_response_headers()['x-ms-request-charge']

        return {
            "chunk_id": chunk_id,
            "document_id": document_id,
            "ru_charge": float(ru_charge)
        }
    ```

1. **BEGIN VECTOR SIMILARITY SEARCH FUNCTION** というコメントを検索し、次のコードをこのコメントの直後に追加します。 この関数は、ベクトル距離を使ってクエリに最も似ているチケットを検索します。

    ```python
    def vector_similarity_search(
        query_embedding: list,
        top_n: int = 5
    ) -> list:
        """
        Find documents most similar to the query using vector distance.

        Uses the VectorDistance function to calculate cosine similarity between
        the query embedding and document embeddings stored in Cosmos DB.
        Results are ordered by similarity (lowest distance = most similar).
        """
        container = get_container()

        # The VectorDistance function calculates the distance between two vectors
        # Using cosine distance: 0 = identical, 2 = opposite
        # We order by distance ascending so most similar results come first
        # The @queryVector parameter contains our 256-dimensional query embedding
        query = """
            SELECT TOP @topN
                c.id,
                c.documentId,
                c.content,
                c.metadata,
                VectorDistance(c.embedding, @queryVector) AS similarityScore
            FROM c
            ORDER BY VectorDistance(c.embedding, @queryVector)
        """

        items = container.query_items(
            query=query,
            parameters=[
                {"name": "@topN", "value": top_n},
                {"name": "@queryVector", "value": query_embedding}
            ],
            enable_cross_partition_query=True
        )

        return [
            {
                "chunk_id": item["id"],
                "document_id": item["documentId"],
                "content": item["content"],
                "metadata": item["metadata"],
                "similarity_score": item["similarityScore"]
            }
            for item in items
        ]
    ```

1. **BEGIN FILTERED VECTOR SEARCH FUNCTION** というコメントを検索し、次のコードをこのコメントの直後に追加します。 この関数は、ハイブリッド クエリのためにベクトル類似検索とメタデータ フィルター処理を組み合わせます。

    ```python
    def filtered_vector_search(
        query_embedding: list,
        category: str = None,
        top_n: int = 5
    ) -> list:
        """
        Combine vector similarity search with metadata filtering.

        This hybrid approach first filters documents by category (or other metadata),
        then ranks the filtered results by vector similarity. This is useful for
        narrowing results to a specific domain before applying semantic search.
        """
        container = get_container()

        # Build WHERE clause for metadata filtering
        # The filter is applied BEFORE vector ranking, reducing the search space
        where_clause = ""
        parameters = [
            {"name": "@topN", "value": top_n},
            {"name": "@queryVector", "value": query_embedding}
        ]

        if category:
            where_clause = "WHERE c.metadata.category = @category"
            parameters.append({"name": "@category", "value": category})

        # Filtered vector search: apply metadata filter, then rank by similarity
        query = f"""
            SELECT TOP @topN
                c.id,
                c.documentId,
                c.content,
                c.metadata,
                VectorDistance(c.embedding, @queryVector) AS similarityScore
            FROM c
            {where_clause}
            ORDER BY VectorDistance(c.embedding, @queryVector)
        """

        items = container.query_items(
            query=query,
            parameters=parameters,
            enable_cross_partition_query=True
        )

        return [
            {
                "chunk_id": item["id"],
                "document_id": item["documentId"],
                "content": item["content"],
                "metadata": item["metadata"],
                "similarity_score": item["similarityScore"]
            }
            for item in items
        ]
    ```

1. 変更を *vector_functions.py* ファイルに保存します。

1. 少し時間をかけて、このファイルのコード全体をレビューします。

### セットアップ コンテナーのコードをレビューする

このセクションでは、ベクトル埋め込みとインデックス作成ポリシーを使って Cosmos DB コンテナーを作成するために使われる *setup_container.py* スクリプトをレビューします。 これらのポリシーを使用するコンテナーはデプロイ スクリプトによって既に作成されていますが、コードをレビューすると構成を理解するのに役立ちます。

1. VS Code で *client/setup_container.py* ファイルを開きます。

1. **BEGIN CREATE VECTOR CONTAINER FUNCTION** というコメントを検索して、コードを確認します。 次の 2 つの重要なポリシー構成に注目してください。

    ```python
    # Define the vector embedding policy
    # This tells Cosmos DB how to handle vector data at the /embedding path
    vector_embedding_policy = {
        "vectorEmbeddings": [
            {
                "path": "/embedding",
                "dataType": "float32",
                "distanceFunction": "cosine",
                "dimensions": 256
            }
        ]
    }

    # Define the indexing policy with vector index
    # - DiskANN provides efficient approximate nearest neighbor search
    # - Exclude /embedding/* from standard indexing (vectors use their own index)
    indexing_policy = {
        "indexingMode": "consistent",
        "automatic": True,
        "includedPaths": [{"path": "/*"}],
        "excludedPaths": [{"path": "/embedding/*"}],
        "vectorIndexes": [
            {"path": "/embedding", "type": "diskANN"}
        ]
    }

    # Create the container with vector policies
    # partition_key determines how data is distributed across physical partitions
    container = database.create_container_if_not_exists(
        id=container_name,
        partition_key=PartitionKey(path="/documentId"),
        indexing_policy=indexing_policy,
        vector_embedding_policy=vector_embedding_policy
    )
    ```

1. 少し時間をとって、重要な構成要素を理解してください。

    | ポリシー | 設定 | パーパス |
    |--------|---------|---------|
    | **vectorEmbeddings** | path: /embedding | ベクトル データが格納されている場所 |
    | **vectorEmbeddings** | dimensions: 256 | 埋め込みモデルの出力と一致する必要があります |
    | **vectorEmbeddings** | distanceFunction: cosine | VectorDistance の類似性メトリック |
    | **vectorIndexes** | type: diskANN | 効率的な近似最近傍アルゴリズム |
    | **excludedPaths** | /embedding/* | ベクトルは、標準ではなく特殊なインデックスを使います |


次に、Azure リソースのデプロイを完了します。

## Azure リソースのデプロイを完了する

このセクションでは、デプロイ スクリプトに戻ってコンテナーを作成し、Entra ID のアクセスを構成して、接続情報を取得します。

1. **Create Cosmos DB account** の操作が完了したら、「**2**」と入力して **[Create container]** オプションを開始します。 これにより、類似検索に必要な埋め込みとインデックス作成ポリシーを含むベクトル コンテナーが作成されます。

1. 「**3**」と入力して、**[Configure Entra ID access]** オプションを開始します。 これにより、お使いのユーザー アカウントに、Cosmos DB データ プレーンにアクセスするために必要なロールが割り当てられます。

1. 「**4**」と入力して、**[Check deployment status]** オプションを開始します。 ベクトル検索機能が有効な状態で、Cosmos DB アカウントが準備完了と表示されることを確認します。

1. 「**5**」と入力して、**[Retrieve connection info]** オプションを開始します。 これにより、必要な環境変数を含むファイルが作成されます。

1. 「**6**」と入力して、デプロイ スクリプトを終了します。

1. 次のコマンドを実行して、前のステップで作成したファイルからターミナル セッションに環境変数を読み込みます。

    **Bash**
    ```bash
    source .env
    ```

    **PowerShell**
    ```powershell
    . .\.env.ps1
    ```

    >**注:** ターミナルは開いたままにしてください。 閉じて新しいターミナルを作成する場合は、このコマンドをもう一度実行して環境変数を再度作成することが必要になる可能性があります。

次に、Python 環境を設定し、アプリケーションを実行します。

## Python 環境を設定する

このセクションでは、Python 仮想環境を作成し、コンテナー セットアップ スクリプトと Flask アプリケーションの両方に必要な依存関係をインストールします。

1. 次のコマンドを実行して、"クライアント" ディレクトリに移動します。**

    ```
    cd client
    ```

1. 次のコマンドを実行して、Python スクリプト用の仮想環境を作成します。 実際に使用する環境に応じて、コマンドは **python** または **python3** となります。

    ```
    python -m venv .venv
    ```

1. 次のコマンドを実行して、Python 環境をアクティブにします。 **注:** Linux/macOS では、Bash コマンドを使用してください。 Windows では、PowerShell コマンドを使用します。 Windows で Git Bash を使っている場合は、**source .venv/Scripts/activate** を使用します。

    **Bash**
    ```bash
    source .venv/bin/activate
    ```

    **PowerShell**
    ```powershell
    .\.venv\Scripts\Activate.ps1
    ```

1. 次のコマンドを実行して、Python の依存関係をインストールします。 これにより、**flask**、**azure-cosmos**、**azure-identity** の各ライブラリがインストールされます。

    ```bash
    pip install -r requirements.txt
    ```

次に、Flask アプリケーションを使ってベクトル検索関数をテストします。

## Flask アプリを使ってベクトル検索関数をテストする

このセクションでは、Flask Web アプリケーションを開始し、そのインターフェイスを使って、作成したベクトル検索関数をテストします。 このアプリでは、サンプル サポート チケットを読み込んでベクトル類似検索を実行する視覚的な方法が提供されます。

1. 仮想環境がアクティブにされた "クライアント" ディレクトリにまだいることを確認します。** ターミナルのプロンプトに **(.venv)** と表示されているはずです。

1. 次のコマンドを実行して、Flask アプリケーションを開始します。

    ```bash
    python app.py
    ```

1. ブラウザーを開き、`http://127.0.0.1:5000` に移動してアプリケーションを表示します。

### サンプル データを読み込む

このセクションでは、このアプリを使って、事前に計算された埋め込みを含むサンプル サポート チケットを Cosmos DB コンテナーに読み込みます。 サンプル データには、それぞれが 256 次元の埋め込みベクトルを含む、さまざまなカテゴリ (billing、technical、account、shipping) に関する 12 個のサポート チケットが含まれています。 アプリは、*vector_functions.py* で作成した **store_vector_document()** 関数を呼び出します。

1. **[サンプル データの読み込み]** セクションで、**[ベクトル データの読み込み]** を選びます。 これにより、事前に計算された埋め込みを含むチケットが、*sample_vectors.json* ファイルから挿入されます。

1. 読み込まれたチケットの数と RU (要求ユニット) の合計料金を示す成功メッセージが **[結果]** セクションに表示されることを確認します。

### ベクトル類似性検索

このセクションでは、事前に計算されたクエリ ベクトルを使ってセマンティック検索を実行します。 アプリは、*vector_functions.py* で作成した **vector_similarity_search()** 関数を呼び出します。

1. **[ベクトル類似検索]** セクションで、**[クエリの選択]** ドロップダウンから **[I can't login to my account]** を選びます。

1. 既定の **[上位 5 件]** の結果をそのまま使用し、**[検索]** を選びます。

1. 類似度スコアでランク付けされたチケットを示す結果を確認します。 認証とアカウント アクセスに関するチケットが、クエリとは異なる用語を使っている場合でも、最初に表示されることに注意してください。

1. **[My payment was charged twice]** や **[Package hasn't arrived yet]** などの異なるクエリを選んで、セマンティック検索が関連するサポート ケースをどのように見つけるかを確認してください。

### フィルター処理されるベクトル検索

このセクションでは、メタデータのフィルター処理とベクトル類似度ランク付けを組み合わせます。 アプリは、*vector_functions.py* で作成した **filtered_vector_search()** 関数を呼び出します。 フィルター処理によって結果が特定のカテゴリにどのように絞り込まれるかを確認します。

1. **[フィルター処理されたベクトル検索]** セクションで、**[クエリの選択]** ドロップダウンから **[I can't login to my account]** を選びます。

1. **[カテゴリでフィルター]** ドロップダウンから **[technical]** を選びます。

1. **[フィルターを使用して検索]** を選び、フィルター処理された検索を実行します。

1. 結果を確認します。 **technical** カテゴリのチケットのみが返され、クエリへの類似度によってランク付けされていることに注意してください。

1. **account** カテゴリで同じクエリを試し、セマンティック的にはまだ関連しているもののアカウント関連の問題に限定されている異なる結果を確認します。

1. ターミナルに戻り、**Ctrl + C** キーを押して Flask アプリケーションを停止します。

## まとめ

この演習では、Azure Cosmos DB for NoSQL を使ってベクトル類似検索を実装しました。 **EnableNoSQLVectorSearch** 機能を使って Azure Cosmos DB アカウントをデプロイし、Entra ID 認証を構成しました。 **VectorDistance** 関数を使用できるようにするベクトル埋め込みとインデックス作成ポリシーを含むコンテナーを、Python SDK を使って作成しました。 埋め込みを含むサポート チケットを格納し、ベクトル類似検索を実行し、ベクトル検索とメタデータ フィルターを組み合わせる Python 関数を作成しました。 Flask Web アプリケーションを使ってワークフローをテストしました。 アプリケーションでこのパターンを使うと、サポート データに対してセマンティック検索を実行し、正確なキーワード一致ではなく、意味に基づいて似たチケットを検索できます。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループとグループ内のすべてのリソースを削除します。 **\<rg-name>** は、演習で前に選択した名前に置き換えます。 このコマンドにより、Azure でバックグラウンド タスクが起動され、リソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の中で問題が発生した場合は、次のステップを試してみてください。

**Flask アプリを開始できない**
- Python 仮想環境がアクティブになっていることを確認します (ターミナル プロンプトに **(.venv)** が表示されるはずです)
- 依存関係がインストールされていることを確認します: **pip install -r requirements.txt**
- **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行して、環境変数が確実に読み込まれるようにします
- **python app.py** を実行するときに、"クライアント" のディレクトリにいることを確認します**

**認証またはアクセス拒否エラー**
- デプロイ スクリプトのオプション **3** を実行して Entra ID アクセスが構成されたことを確認します
- ユーザーが**共同作成者**ロールと **Cosmos DB 組み込みデータ共同作成者**ロールの両方を持っていることを確認します
- ターミナル セッションで **COSMOS_ENDPOINT** が正しく設定されていることを確認します

**ベクトル検索で結果やエラーが返されない**
- デプロイ スクリプトのオプション **2** を実行してベクトル コンテナーが作成されたことを確認します
- コンテナーでベクトル埋め込みポリシーが構成されていることを確認します (デプロイ スクリプトのオプション **4** で状態を調べます)
- 検索を実行する前にサンプル チケットが読み込まれたことを確認します

**setup_container.py が失敗する**
- Python の仮想環境がアクティブになっていることを確認します
- 環境変数が設定されていることを確認します (**COSMOS_ENDPOINT**、**COSMOS_DATABASE**、**COSMOS_CONTAINER**)
- コンテナーが既に存在する場合、スクリプトは既存のコンテナーを使います

**Cosmos DB の操作が失敗する**
- デプロイ スクリプトのオプション **4** を実行して Cosmos DB アカウントが準備されていることを確認します
- デプロイの間にデータベースが作成されたことを確認します
- アカウントに **EnableNoSQLVectorSearch** 機能があることを確認します

**環境変数に関する問題**
- デプロイ スクリプトのオプション **5** を実行して *.env* ファイルが作成されたことを確認します
- 新しいターミナルを作成した後で **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行します
- **echo $COSMOS_ENDPOINT** (Bash) または **$env:COSMOS_ENDPOINT** (PowerShell) を実行して、変数が設定されていることを確認します

**Python venv のアクティブ化に関する問題**
- Linux/macOS では **source .venv/bin/activate** を使用します
- Windows PowerShell では **.\.venv\Scripts\Activate.ps1** を使用します
- **activate** スクリプトがない場合は、**python3-venv** パッケージを再インストールして venv を再作成します
