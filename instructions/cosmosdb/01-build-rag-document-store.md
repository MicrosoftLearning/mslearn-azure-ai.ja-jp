---
lab:
  topic: Azure Cosmos DB for NoSQL
  title: Azure Cosmos DB for NoSQL で RAG ドキュメント ストアを構築する
  description: Azure Cosmos DB for NoSQL を使って検索拡張生成 (RAG) 用のドキュメント ストレージ バックエンドを構築する方法を学びます
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Cosmos DB
---

# Azure Cosmos DB for NoSQL で RAG ドキュメント ストアを構築する

この演習では、取得拡張生成 (RAG) アプリケーションのドキュメント ストアとして機能する Azure Cosmos DB for NoSQL データベースを作成します。 このデータベースには、AI アプリケーションが言語モデルにコンテキストを提供するために取得できるメタデータを含む、チャンクにされたドキュメントが格納されます。 ドキュメントの取得用に最適化されたスキーマを設計して、ドキュメント チャンクの格納とクエリを実行する Python 関数を作成し、Flask Web アプリケーションを使って完成したワークフローをテストします。 このパターンでは、組織のドキュメントを言語モデルの応答の典拠とする AI アプリケーションを構築するための基盤が提供されます。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードし、デプロイ スクリプトを構成する
- データベースとコンテナーを含む Azure Cosmos DB for NoSQL アカウントをデプロイする
- ドキュメントのチャンクの格納と取得を行うための Python 関数を作成する
- Flask Web アプリケーションを使って RAG 関数をテストする
- Cosmos DB SQL API を使ってドキュメントのコンテキストのクエリを実行する

この演習の所要時間は約 **30** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- 必要な Azure サービスをデプロイするためのアクセス許可がある Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- [Python 3.12](https://www.python.org/downloads/) 以上。

## プロジェクト スターター ファイルをダウンロードして Azure サービスをデプロイする

このセクションでは、プロジェクト スターター ファイルをダウンロードし、スクリプトを使用してこの演習に必要なサービスをご自分の Azure サブスクリプションにデプロイします。 Cosmos DB アカウントのデプロイが完了するまで数分かかる場合があります。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/cosmosdb-build-query-python.zip
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

1. 次のコマンドを実行し、演習に必要なリソース プロバイダーがサブスクリプションにあることを確認します。

    ```azurecli
    az provider register --namespace Microsoft.DocumentDB
    ```

### Azure でリソースを作成する

このセクションでは、デプロイ スクリプトを実行して Cosmos DB アカウントをデプロイします。

1. プロジェクトのルート ディレクトリにいることを確認し、ターミナルで適切なコマンドを実行してデプロイ スクリプトを起動します。

    **Bash**
    ```bash
    bash azdeploy.sh
    ```

    **PowerShell**
    ```powershell
    ./azdeploy.ps1
    ```

1. スクリプト メニューが表示されたら、「**1**」と入力して **[Create Cosmos DB account]** オプションを開始します。 これにより、データベースとコンテナーを含む Cosmos DB for NoSQL アカウントが作成されます。 **注:** デプロイが完了するまで 5 分から 10 分ほどかかります。

    >**重要:** デプロイを実行するターミナルは、演習が終了するまで開いたままにしてください。 ターミナルでのデプロイの進行中に、演習の次のセクションに進んでもかまいません。

## RAG ドキュメント関数を完成させる

このセクションでは、AI アプリケーションがドキュメント チャンクの格納と取得を行うために呼び出すことができる関数を追加して、*rag_functions.py* ファイルを完成させます。 これらの関数は、ドキュメント ストアへのアプリケーションのインターフェイスとして機能します。 この演習で後ほど実行する *test_workflow.py* スクリプトでは、これらの関数がインポートされ、AI アプリケーションでそれらがどのように使われるかが示されます。

1. VS Code で *rag-backend/rag_functions.py* ファイルを開きます。

1. **BEGIN STORE DOCUMENT CHUNK FUNCTION** というコメントを検索し、次に示すコードをこのコメントの直後に追加します。 この関数は、アップサートを使って挿入と更新の両方を処理し、ドキュメントのチャンクとそのメタデータを格納します。

    ```python
    def store_document_chunk(
        document_id: str,
        chunk_id: str,
        content: str,
        metadata: dict = None,
        embedding: list = None
    ) -> dict:
        """Store a document chunk with metadata and optional embedding placeholder."""
        container = get_container()

        # Build the document structure following our RAG schema
        # The 'id' field is required by Cosmos DB and must be unique within the partition
        # The 'documentId' field is our partition key - chunks from the same source document
        # are stored together for efficient retrieval
        chunk = {
            "id": chunk_id,
            "documentId": document_id,
            "content": content,
            "metadata": metadata or {},
            "embedding": embedding or [],  # Placeholder for vector embeddings
            "createdAt": datetime.utcnow().isoformat(),
            "chunkIndex": metadata.get("chunkIndex", 0) if metadata else 0
        }

        # upsert_item inserts if new, updates if exists (based on id + partition key)
        # This is idempotent - safe to call multiple times with the same data
        response = container.upsert_item(body=chunk)

        # Request Units (RUs) measure the cost of database operations in Cosmos DB
        # Tracking RU consumption helps optimize queries and estimate costs
        ru_charge = response.get_response_headers()['x-ms-request-charge']

        return {
            "chunk_id": chunk_id,
            "document_id": document_id,
            "ru_charge": float(ru_charge)
        }
    ```

1. **BEGIN GET CHUNKS BY DOCUMENT FUNCTION** というコメントを検索し、次に示すコードをこのコメントの直後に追加します。 この関数は、順次読み取りのためにチャンク インデックスで並べ替えて、特定のドキュメントのチャンクをすべて取得します。

    ```python
    def get_chunks_by_document(document_id: str, limit: int = 100) -> list:
        """Retrieve all chunks for a specific document, ordered by chunk index."""
        container = get_container()

        # SQL query using parameterized values (@documentId, @limit) to prevent injection
        # The 'c' alias represents each document in the container
        query = """
            SELECT c.id, c.content, c.metadata, c.chunkIndex, c.createdAt
            FROM c
            WHERE c.documentId = @documentId
            ORDER BY c.chunkIndex
            OFFSET 0 LIMIT @limit
        """

        # Single-partition query: providing partition_key limits the query to one partition
        # This is more efficient than cross-partition queries because Cosmos DB only
        # needs to read from one physical partition instead of fanning out to all partitions
        items = container.query_items(
            query=query,
            parameters=[
                {"name": "@documentId", "value": document_id},
                {"name": "@limit", "value": limit}
            ],
            partition_key=document_id  # Scopes query to a single partition
        )

        # Transform Cosmos DB items into a consistent response format
        return [
            {
                "chunk_id": item["id"],
                "content": item["content"],
                "metadata": item["metadata"],
                "chunk_index": item["chunkIndex"],
                "created_at": item["createdAt"]
            }
            for item in items
        ]
    ```

1. **BEGIN SEARCH CHUNKS BY METADATA FUNCTION** というコメントを検索し、次に示すコードをこのコメントの直後に追加します。 この関数は、メタデータ フィルターを使ってドキュメントをまたいでチャンクを検索します。これは、タグ、カテゴリ、または他の属性に基づいて関連するコンテキストを検索するのに役立ちます。

    ```python
    def search_chunks_by_metadata(
        filters: dict,
        limit: int = 10
    ) -> list:
        """Search for chunks across documents using metadata filters."""
        container = get_container()

        # Build WHERE clauses dynamically based on provided filters
        # This allows flexible querying by any combination of metadata fields
        where_clauses = []
        parameters = []

        if "source" in filters and filters["source"]:
            where_clauses.append("c.metadata.source = @source")
            parameters.append({"name": "@source", "value": filters["source"]})

        if "category" in filters and filters["category"]:
            where_clauses.append("c.metadata.category = @category")
            parameters.append({"name": "@category", "value": filters["category"]})

        if "tags" in filters and filters["tags"]:
            # ARRAY_CONTAINS checks if a value exists within an array field
            # This is useful for searching tags, keywords, or other list-based metadata
            where_clauses.append("ARRAY_CONTAINS(c.metadata.tags, @tag)")
            parameters.append({"name": "@tag", "value": filters["tags"][0]})

        # Default to "1=1" (always true) if no filters provided
        where_clause = " AND ".join(where_clauses) if where_clauses else "1=1"
        parameters.append({"name": "@limit", "value": limit})

        query = f"""
            SELECT c.id, c.documentId, c.content, c.metadata, c.chunkIndex
            FROM c
            WHERE {where_clause}
            OFFSET 0 LIMIT @limit
        """

        # Cross-partition query: searches across ALL partitions in the container
        # Required when you don't know which partition contains the data you need
        # More expensive than single-partition queries but necessary for metadata searches
        items = container.query_items(
            query=query,
            parameters=parameters,
            enable_cross_partition_query=True  # Fan out to all partitions
        )

        return [
            {
                "chunk_id": item["id"],
                "document_id": item["documentId"],
                "content": item["content"],
                "metadata": item["metadata"],
                "chunk_index": item["chunkIndex"]
            }
            for item in items
        ]
    ```

1. **BEGIN GET CHUNK BY ID FUNCTION** というコメントを検索し、次に示すコードをこのコメントの直後に追加します。 この関数は、効率的なポイント読み取りを実行し、特定のチャンクをその ID とドキュメント ID で取得します。

    ```python
    def get_chunk_by_id(document_id: str, chunk_id: str) -> dict:
        """Retrieve a specific chunk using a point read (most efficient)."""
        container = get_container()

        try:
            # Point read: the most efficient Cosmos DB operation
            # By providing both the item ID and partition key, Cosmos DB can go
            # directly to the exact location of the document without any query execution
            # This results in the lowest latency and RU cost (typically 1 RU for small docs)
            item = container.read_item(
                item=chunk_id,         # The unique ID within the partition
                partition_key=document_id  # The partition where this item lives
            )
            return {
                "chunk_id": item["id"],
                "document_id": item["documentId"],
                "content": item["content"],
                "metadata": item["metadata"],
                "chunk_index": item["chunkIndex"],
                "created_at": item["createdAt"],
                "embedding": item.get("embedding", [])
            }
        except exceptions.CosmosResourceNotFoundError:
            # Return None if the item doesn't exist rather than raising an exception
            # This allows the caller to handle missing items gracefully
            return None
    ```

1. 変更を *rag_functions.py* ファイルに保存します。

1. 少し時間をかけて、アプリ内のすべてのコードをレビューします。

次に、Azure リソースのデプロイを完了します。

## Azure リソースのデプロイを完了する

このセクションでは、デプロイ スクリプトに戻って Entra ID のアクセスを構成し、Cosmos DB アカウントの接続情報を取得します。

1. **Create Cosmos DB account** の操作が完了したら、「**2**」と入力して **[Configure Entra ID access]** オプションを開始します。 これにより、お使いのユーザー アカウントに、Cosmos DB データ プレーンにアクセスするために必要なロールが割り当てられます。

1. 「**3**」と入力して、**[Check deployment status]** オプションを開始します。 これにより、すべてのリソースの準備が完了していることが確認されます。

1. 「**4**」と入力して、**[Retrieve connection info]** オプションを開始します。 これにより、必要な環境変数を含むファイルが作成されます。

1. 「**5**」と入力して、デプロイ スクリプトを終了します。

1. 次のコマンドを実行して、前のステップで作成したファイルからターミナル セッションに環境変数を読み込みます。

    **Bash**
    ```bash
    source .env
    ```

    **PowerShell**
    ```powershell
    . .\.env.ps1
    ```

    >**注:** ターミナルは開いたままにします。 閉じて新しいターミナルを作成する場合は、このコマンドをもう一度実行して環境変数を再度作成することが必要になる可能性があります。

次に、RAG ドキュメント ストアに使われるドキュメント スキーマを調べます。

## RAG ドキュメント スキーマを理解する

このセクションでは、RAG アプリケーション用に設計されたドキュメント スキーマについて学びます。 リレーショナル データベースとは異なり、Cosmos DB for NoSQL では柔軟な JSON ドキュメント モデルが使われます。 このコンテナーは、**documentId** をパーティション キーとして作成されました。これにより、効率的な取得のため、同じソース ドキュメントのすべてのチャンクがグループ化されます。

各チャンクのドキュメント スキーマには、次のものが含まれます。

| フィールド | 説明 |
|-------|-------------|
| **id** | チャンクの一意識別子 (Cosmos DB で必要) |
| **documentId** | ソース ドキュメント識別子 (パーティション キー) |
| **content** | チャンクの実際のテキスト コンテンツ |
| **metadata** | ソース、カテゴリ、タグ、カスタム属性のための柔軟なオブジェクト |
| **embedding (埋め込み)** | ベクトル埋め込み用の配列プレースホルダー (ベクトル検索のシナリオで使用) |
| **chunkIndex** | ソース ドキュメント内でのチャンクの位置 |
| **createdAt** | チャンクが格納されたときのタイムスタンプ |

このスキーマでは、一般的な RAG パターンがサポートされています。

- **ポイント読み取り**: ID とドキュメント ID で特定のチャンクを取得します (最短の待ち時間)
- **単一パーティション クエリ**: ドキュメントのすべてのチャンクを効率的に取得します
- **パーティション間クエリ**: メタデータを使ってドキュメントをまたいで検索します
- **ベクトル検索**: ベクトル インデックス作成と組み合わせた場合

## Flask アプリを使って RAG 関数をテストする

このセクションでは、Flask Web アプリケーションを開始し、そのインターフェイスを使って、作成した RAG 関数をテストします。 このアプリは、データの読み込み、テストの実行、チャンクのクエリ、カスタム SQL クエリの実行を視覚的に行う方法を提供します。

1. 次のコマンドを実行して、"クライアント" ディレクトリに移動します。**

    ```
    cd client
    ```

1. 次のコマンドを入力して、Flask アプリ用の仮想環境を作成します。 実際に使用する環境に応じて、コマンドは **python** または **python3** となります。

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

1. 次のコマンドを実行してアプリの Python 依存関係をインストールします。 これにより、**flask** と **azure-cosmos** ライブラリがインストールされます。

    ```bash
    pip install -r requirements.txt
    ```

1. 次のコマンドを実行して、Flask アプリケーションを開始します。

    ```bash
    python app.py
    ```

1. ブラウザーを開き、`http://127.0.0.1:5000` に移動してアプリケーションを表示します。

### サンプル データを読み込む

このセクションでは、アプリを使ってサンプル ドキュメントのチャンクを Cosmos DB コンテナーに読み込みます。 アプリは、*rag_functions.py* で作成した **store_document_chunk()** 関数を呼び出して、各チャンクを挿入します。

1. **[サンプル データの読み込み]** セクションで、**[サンプル チャンクの読み込み]** を選びます。 これにより、架空の Azure ドキュメント記事のコンテンツを表す 12 個のサンプル チャンクが 4 つのドキュメントから挿入されます。

1. 読み込まれたチャンクの数と RU (要求ユニット) の合計料金を示す成功メッセージが **[結果]** セクションに表示されることを確認します。

### テスト ワークフローを実行する

このセクションでは、*rag_functions.py* で作成した RAG 関数が正しく動作することを確認する自動テストを実行します。

1. **[テスト ワークフローの実行]** セクションで、**[テストの実行]** を選びます。 これにより、各関数を使用する 5 つのテストが実行されます。

1. **[結果]** パネルでテスト結果を確認します。 各テストで**合格**状態が示される必要があります。
    - ドキュメント チャンクを格納する
    - ドキュメント ID でチャンクを取得する
    - カテゴリーで検索
    - タグで検索する
    - ID でポイント読み取りを行う

### ドキュメントでチャンクを取得する

このセクションでは、特定のドキュメントのすべてのチャンクを取得します。 アプリは、*rag_functions.py* で作成した **get_chunks_by_document()** 関数を呼び出します。

1. **[ドキュメント別のチャンクの取得]** セクションで、ドロップダウンからドキュメントを選びます (例: **doc-azure-overview**)。

1. **[チャンクの取得]** を選んで、そのドキュメントのすべてのチャンクを取得します。

1. インデックスで並べ替えられたチャンクおよびそのコンテンツとメタデータ タグを示す結果を確認します。

### メタデータで検索する

このセクションでは、メタデータ フィルターを使って、すべてのドキュメントでチャンクを検索します。 アプリは、*rag_functions.py* で作成した **search_chunks_by_metadata()** 関数を呼び出します。 フィルターの組み合わせによって結果がどのように絞り込まれるかを観察します。

1. **[メタデータによる検索]** セクションで、**[カテゴリ]** ドロップダウンから **[ai-applications]** を選びます。 **[タグ]** フィールドは空のままにします。

1. **[検索]** を選んで、そのカテゴリのすべてのチャンクを検索します。

1. **[結果]** パネルで結果を確認します。 4 個のチャンクが返され、それぞれに異なるタグ (**rag**、**embeddings**、**chunking**、**metadata** など) が付いているはずです。

1. 次に、タグ フィルターを追加して結果を絞り込みます。 **[タグ]** フィールドに「**embeddings**」と入力して、**[検索]** をもう一度選びます。

1. **ai-applications** カテゴリが一致し、かつ、**embeddings** タグが含まれるチャンクのみが返され、結果が少なくなることに注意してください。 これはメタデータ フィルターを組み合わせると、RAG アプリケーションでより対象を絞ったコンテキストを取得するのに役立つことを示しています。

## ドキュメント コンテキストのクエリを実行する

このセクションでは、クエリ エクスプローラーを使って Cosmos DB コンテナーに対する SQL クエリを記述する練習をします。 これらのクエリを見ると、RAG アプリケーションでドキュメント コンテキストを取得するためによく使われるパターンがわかります。

1. **[クエリ エクスプローラー]** セクションで、次のクエリを **[SQL クエリ]** フィールドに入力して、特定のドキュメントのすべてのチャンクを検索します。 このクエリは、順次読み取りのためにインデックス順に並べ替えられたチャンクを取得します。

    ```sql
    SELECT c.id, c.chunkIndex, c.content, c.metadata
    FROM c
    WHERE c.documentId = 'doc-azure-overview'
    ORDER BY c.chunkIndex
    ```

1. **[クエリの実行]** を選んで、結果を確認します。

1. 次のクエリを **[SQL クエリ]** フィールドに入力して、すべてのドキュメントで特定のカテゴリを持つチャンクを検索します。 これは、メタデータを検索するパーティションをまたいだクエリを示しています。

    ```sql
    SELECT c.documentId, c.id, c.content, c.metadata.category
    FROM c
    WHERE c.metadata.category = 'cloud-services'
    ```

1. **[クエリの実行]** を選んで、結果を確認します。

1. **[SQL クエリ]** フィールドに次のクエリを入力し、コンテナーに格納されているドキュメントと共に、そのカテゴリおよびソース メタデータを表示します。 これは、RAG の取得に使用できるコンテンツを理解するのに役立ちます。

    ```sql
    SELECT DISTINCT c.documentId, c.metadata.category, c.metadata.source
    FROM c
    ```

1. **[クエリの実行]** を選んで、結果を確認します。

1. **[SQL クエリ]** フィールドに次のクエリを入力し、メタデータに特定のタグが含まれるチャンクを検索します。 これは **ARRAY_CONTAINS** を用いた配列内の探索を示しています。

    ```sql
    SELECT c.documentId, c.id, c.content, c.metadata.tags
    FROM c
    WHERE ARRAY_CONTAINS(c.metadata.tags, 'compute')
    ```

1. **[クエリの実行]** を選んで、結果を確認します。

1. ターミナルに戻り、**Ctrl + C** キーを押して Flask アプリケーションを停止します。

## まとめ

この演習では、RAG アプリケーション用に Cosmos DB ベースのドキュメント ストアを構築しました。 ドキュメント取得パターン用に最適化されたデータベースとコンテナーを含む Azure Cosmos DB for NoSQL アカウントをデプロイしました。 メタデータを使ってドキュメント チャンクを格納し、ドキュメント ID でチャンクを取得し、メタデータ フィルターを使ってドキュメントをまたいで検索を行い、効率的なポイント読み取りを実行する Python 関数を作成しました。 各関数を呼び出す Flask Web アプリケーションを使ってワークフローをテストした後、Cosmos DB SQL API を使って格納されたデータのクエリを実行しました。 このパターンを AI アプリケーションで使うと、チャンクになったドキュメントを格納し、関連するコンテキストを取得して言語モデルの応答の典拠とすることができます。

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
- デプロイ スクリプトのオプション **2** を実行して Entra ID アクセスが構成されたことを確認します
- ユーザーが**共同作成者**ロールと **Cosmos DB 組み込みデータ共同作成者**ロールの両方を持っていることを確認します
- ターミナル セッションで **COSMOS_ENDPOINT** が正しく設定されていることを確認します

**Cosmos DB の操作が失敗する**
- デプロイ スクリプトのオプション **3** を実行して Cosmos DB アカウントが準備されていることを確認します
- デプロイの間にデータベースとコンテナーが作成されたことを確認します
- コンテナーでパーティション キーとして **/documentId** が使われていることを調べます

**環境変数に関する問題**
- デプロイ スクリプトのオプション **4** を実行して *.env* ファイルが作成されたことを確認します
- 新しいターミナルを作成した後で **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行します
- **echo $COSMOS_ENDPOINT** (Bash) または **$env:COSMOS_ENDPOINT** (PowerShell) を実行して、変数が設定されていることを確認します

**Python venv のアクティブ化に関する問題**
- Linux/macOS では **source .venv/bin/activate** を使用します
- Windows PowerShell では **.\venv\Scripts\Activate.ps1** を使用します
- **activate** スクリプトがない場合は、**python3-venv** パッケージを再インストールして venv を再作成します
