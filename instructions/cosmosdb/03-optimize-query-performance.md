---
lab:
  topic: Azure Cosmos DB for NoSQL
  title: Azure Cosmos DB for NoSQL でベクトル インデックスを使用してクエリ パフォーマンスを最適化する
  description: Azure Cosmos DB for NoSQL でベクトル インデックス作成戦略を比較および調整して、クエリのパフォーマンスを最適化し、RU コストを削減する方法について説明します
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Cosmos DB
---

# Azure Cosmos DB for NoSQL でベクトル インデックスを使用してクエリ パフォーマンスを最適化する

この演習では、Azure Cosmos DB for NoSQL でクエリのパフォーマンスを最適化するために、ベクトル インデックス作成戦略を比較および調整します。 ベクトル インデックスは、検索品質と要求ユニット (RU) の消費の両方に大きな影響を与えます。 3 種類の異なるインデックス (flat、quantizedFlat、diskANN) でコンテナーを作成し、同じサンプル データを読み込み、比較検索を実行してパフォーマンスの違いを測定します。 この実践的な練習は、AI アプリケーションの要件に適したインデックス作成戦略を選ぶのに役立ちます。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードし、デプロイ スクリプトを構成する
- ベクトル検索機能を含む Azure Cosmos DB for NoSQL アカウントをデプロイする
- ベクトル インデックスのパフォーマンスを比較するための Python 関数を作成する
- flat、quantizedFlat、diskANN インデックスを含むコンテナーを作成する
- Flask Web アプリケーションを使ってインデックスのパフォーマンスをテストして比較する

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
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/cosmosdb-optimize-query-python.zip
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

## インデックス比較関数を完成させる

このセクションでは、ベクトル インデックスのパフォーマンスを比較するための Python コードを完成させて、コンテナー セットアップ スクリプトをレビューします。 類似検索を実行し、RU の消費と実行時間を追跡する関数を追加します。 また、ベクトル インデックス作成のさまざまな構成を調べて、各インデックスの種類がどのように作成されるかを理解します。

### ベクトル類似検索スクリプトを完成させる

このセクションでは、ベクトル類似検索とパフォーマンス追跡を実行する関数を追加して、*index_functions.py* ファイルを完成させます。 この関数をコンテナーごとに呼び出して、インデックスの種類が異なると同じクエリがどのように処理されるかを比較します。

1. VS Codeで *client/index_functions.py* ファイルを開きます。

1. **BEGIN VECTOR SIMILARITY SEARCH FUNCTION** というコメントを検索し、次のコードをこのコメントの直後に追加します。 この関数は、クエリに似たドキュメントを検索し、パフォーマンス メトリックを追跡します。

    ```python
    def vector_similarity_search(
        container_name: str,
        query_embedding: list,
        top_n: int = 5
    ) -> dict:
        """
        Find documents most similar to the query using vector distance.

        This function performs a vector similarity search using the VectorDistance
        function and tracks the RU consumption and execution time. Results are
        ordered by distance (lowest = most similar).

        Args:
            container_name: Name of the container to search
            query_embedding: 256-dimensional query vector
            top_n: Number of results to return

        Returns:
            Dictionary containing results, ru_charge, and execution_time_ms
        """
        container = get_container(container_name)

        # Track execution time for performance comparison
        start_time = time.time()

        # The VectorDistance function calculates distance between vectors
        # Using cosine distance: 0 = identical, 2 = opposite
        # Results ordered by distance ascending (most similar first)
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

        items = list(container.query_items(
            query=query,
            parameters=[
                {"name": "@topN", "value": top_n},
                {"name": "@queryVector", "value": query_embedding}
            ],
            enable_cross_partition_query=True
        ))

        end_time = time.time()
        execution_time_ms = (end_time - start_time) * 1000

        # Get RU charge from the query - note: this is approximate for multi-page results
        # For accurate RU tracking in production, use Azure Monitor
        ru_charge = 0.0
        try:
            # The last_response_headers contains the RU charge
            ru_charge = float(container.client_connection.last_response_headers.get(
                'x-ms-request-charge', 0
            ))
        except Exception:
            pass  # RU tracking may not be available in all scenarios

        results = [
            {
                "chunk_id": item["id"],
                "document_id": item["documentId"],
                "content": item["content"],
                "metadata": item["metadata"],
                "similarity_score": item["similarityScore"]
            }
            for item in items
        ]

        return {
            "results": results,
            "ru_charge": ru_charge,
            "execution_time_ms": round(execution_time_ms, 2)
        }
    ```

1. **BEGIN COMPARE INDEX PERFORMANCE FUNCTION** というコメントを検索し、次のコードをこのコメントの直後に追加します。 この関数は、3 つのコンテナーすべてに対して同じクエリを実行し、比較結果を返します。

    ```python
    def compare_index_performance(
        query_embedding: list,
        top_n: int = 5
    ) -> dict:
        """
        Run the same vector search query against all three containers and compare performance.

        This function executes identical vector similarity searches against containers
        with different indexing strategies (flat, quantizedFlat, diskANN) to demonstrate
        the performance characteristics of each approach.

        Args:
            query_embedding: 256-dimensional query vector
            top_n: Number of results to return from each container

        Returns:
            Dictionary with results from each container including RU costs and timing
        """
        comparison = {}

        # Test each container with the same query
        for index_type, container_name in [
            ("flat", CONTAINER_FLAT),
            ("quantizedFlat", CONTAINER_QUANTIZED),
            ("diskANN", CONTAINER_DISKANN)
        ]:
            try:
                result = vector_similarity_search(container_name, query_embedding, top_n)
                comparison[index_type] = {
                    "container": container_name,
                    "results": result["results"],
                    "ru_charge": result["ru_charge"],
                    "execution_time_ms": result["execution_time_ms"],
                    "result_count": len(result["results"]),
                    "status": "success"
                }
            except Exception as e:
                comparison[index_type] = {
                    "container": container_name,
                    "results": [],
                    "ru_charge": 0,
                    "execution_time_ms": 0,
                    "result_count": 0,
                    "status": "error",
                    "error": str(e)
                }

        return comparison
    ```

1. *index_functions.py* ファイルへの変更を保存します。

1. 少し時間をかけて、スクリプト内のすべてのコードをレビューします。

### コンテナーのセットアップ コードをレビューする

このセクションでは、異なるベクトル インデックス作成戦略でコンテナーを作成する *setup_containers.py* スクリプトをレビューします。 ベクトル埋め込みポリシー (パス、ディメンション、データ型、距離関数) はコンテナーの作成時に設定され、後で変更することはできません。 一方、ベクトル インデックスの種類はインデックス作成ポリシーの一部であり、既存のコンテナーで更新できます。 インデックスの種類を変更すると、Cosmos DB によってバックグラウンドでインデックスの変換が実行されます。 このように柔軟性はありますが、それでもさまざまなインデックスの種類を前もってテストするのはがスト プラクティスです。一般的なアプローチとしては、運用構成にコミットする前に、インデックスの種類ごとにテスト コンテナーを作成し、代表的なサンプル データを読み込み、ベンチマーク クエリを実行して RU のコストと待ち時間を測定します。

1. VS Codeで *client/setup_containers.py* ファイルを開きます。

1. **BEGIN CREATE FLAT CONTAINER FUNCTION** というコメントを検索して、コードを確認します。 flat インデックスがどのように構成されているかに注目します。

    ```python
    # Flat index: exact search, compares query against all vectors
    # Higher RU cost for large datasets but guaranteed best results
    indexing_policy = {
        "indexingMode": "consistent",
        "automatic": True,
        "includedPaths": [
            {"path": "/*"}
        ],
        "excludedPaths": [
            {"path": "/embedding/*"}
        ],
        "vectorIndexes": [
            {
                "path": "/embedding",
                "type": "flat"
            }
        ]
    }
    ```

1. **BEGIN CREATE QUANTIZED CONTAINER FUNCTION** というコメントを検索し、quantizedFlat の構成を確認します。

    ```python
    # QuantizedFlat index: compressed vectors for memory efficiency
    # Lower memory footprint with slight accuracy trade-off
    indexing_policy = {
        ...
        "vectorIndexes": [
            {
                "path": "/embedding",
                "type": "quantizedFlat"
            }
        ]
    }
    ```

1. **BEGIN CREATE DISKANN CONTAINER FUNCTION** というコメントを検索し、diskANN の構成を確認します。

    ```python
    # DiskANN index: approximate nearest neighbor with graph-based search
    # Best performance for large datasets, slight accuracy trade-off
    indexing_policy = {
        ...
        "vectorIndexes": [
            {
                "path": "/embedding",
                "type": "diskANN"
            }
        ]
    }
    ```

1. 少し時間をとって、インデックスの種類ごとの主な違いを理解してください。

    | [インデックスの種類] | Search メソッド | 最適な用途 | トレードオフ |
    |------------|---------------|----------|------------|
    | **f**lat | 厳密最近傍 | 小規模のデータセット、最高の正確性 | 大規模のデータセットでは RU が高くなる |
    | **quantizedFlat** | 圧縮された完全検索 | 中規模のデータセット、メモリの効率 | 正確性がわずかに失われる |
    | **diskANN** | 近似グラフ検索 | 大規模なデータセット、運用 | 最大 95% のリコール、最高のパフォーマンス |

次に、Azure リソースのデプロイを完了します。

## Azureリソースのデプロイを完了する

このセクションでは、デプロイ スクリプトに戻ってコンテナーを作成し、Entra ID のアクセスを構成して、接続情報を取得します。

1. **Create Cosmos DB account** の操作が完了したら、「**2**」と入力して **[Create containers]** オプションを開始します。 これにより、ベクトル インデックス作成戦略が異なる 3 つのコンテナーが作成されます (flat、quantizedFlat、diskANN)。

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

    >**注:** ターミナルは開いたままにします。 閉じて新しいターミナルを作成する場合は、このコマンドをもう一度実行して環境変数を再度作成することが必要になる可能性があります。

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

次に、Flask アプリケーションを使ってベクトル インデックスのパフォーマンスをテストします。

## Flask アプリを使ってベクトル インデックスのパフォーマンスをテストする

このセクションでは、Flask Web アプリケーションを開始し、そのインターフェイスを使って、3 つのインデックス作成戦略のベクトル検索のパフォーマンスを比較します。 このアプリは、すべてのコンテナーに対して同じクエリを実行し、結果を並べて表示します。

1. 仮想環境がアクティブにされた "クライアント" ディレクトリにまだいることを確認します。** ターミナルのプロンプトに **(.venv)** と表示されているはずです。

1. 次のコマンドを実行して、Flask アプリケーションを開始します。

    ```bash
    python app.py
    ```

1. ブラウザーを開き、`http://127.0.0.1:5000` に移動してアプリケーションを表示します。

### サンプル データを読み込む

このセクションでは、このアプリを使って、事前に計算された埋め込みを含むサンプル サポート チケットを 3 つのコンテナーすべてに読み込みます。 同じデータを読み込むと、インデックスのパフォーマンスを公平に比較できます。

1. ページの上部にある **[コンテナーの状態]** セクションを確認します。 最初は、3 つのコンテナーすべてが 0 ドキュメントと示されているはずです。

1. **[サンプル データの読み込み]** セクションで、**[すべてのコンテナーにデータを読み込む]** を選びます。 これにより、事前に計算された埋め込みを含む 500 個のサポート チケットが、*sample_vectors.json* ファイルから各コンテナーに挿入されます。 アップロードでは並列処理を使ってデータが効率的に読み込まれ、通常は 30 から 45 秒で完了します。

1. 成功メッセージが表示され、コンテナーごとに読み込まれたドキュメントの数と RU のコストが示されることを確認します。 インデックスの種類によって書き込み RU コストが若干異なる場合があることに注意してください。

### ベクトル検索のパフォーマンスを比較する

このセクションでは、ベクトル類似検索を実行し、各インデックスの種類によって同じクエリがどのように処理されるかを比較します。 アプリで、RU コストと実行時間が並べて表示されます。

1. **[ベクトル検索の比較]** セクションで、**[クエリの選択]** ドロップダウンから **[I can't login to my account]** を選びます。

1. 既定の **[上位 5 件]** の結果をそのまま使用し、**[インデックス パフォーマンスの比較]** を選びます。

1. **[インデックス パフォーマンスの比較]** テーブルの表示を確認します。
    - 各コンテナーの **[結果]** の数
    - 各クエリの **[RU コスト]**
    - **[時間 (ミリ秒)]** の実行時間

1. 下にスクロールし、左右に並べて表示されている各コンテナーから返されたデータの結果を確認します。 注意:
    - 3 つのインデックスすべてから、この小さいデータセットに対する似た結果が返されるはずです
    - RU コストは、インデックスの種類によって異なる場合があります
    - 通常、規模が大きいと、diskANN インデックスでは低い RU 消費量が示されます

1. **[My payment was charged twice]** や **[Package hasn't arrived yet]** などの異なるクエリを試し、検索が違っても同じパターンになることを確認します。

### フィルター処理された検索のパフォーマンスを比較する

このセクションでは、メタデータのフィルター処理とベクトル類似検索を組み合わせます。 フィルター処理を行うと、ベクトル ランク付けを適用する前に検索空間が狭くなり、パフォーマンスに対する影響がインデックスの種類ごとに異なる可能性があります。

1. **[フィルター処理されたベクトル検索の比較]** セクションで、**[クエリの選択]** ドロップダウンから **[Protect my account from hackers]** を選びます。

1. **[カテゴリでフィルター]** ドロップダウンから **[account]** を選びます。

1. **[フィルター処理された検索の比較]** を選び、すべてのコンテナーでフィルター処理された検索を実行します。

1. 結果を確認します。 注意:
    - **account** カテゴリのドキュメントのみが返されます
    - フィルター処理とベクトル検索を組み合わせると、異なる RU パターンが示される場合があります
    - すべてのインデックスの種類で、ベクトル ランク付けの前にフィルターが適用されます

1. **technical** カテゴリで同じクエリを試し、異なるフィルター処理結果を確認します。

### 結果を分析する

テストに基づき、インデックスの種類の選択に関して次のガイドラインを検討します。

| シナリオ | 推奨インデックス | 理由 |
|----------|-------------------|--------|
| 小規模なデータセット (10,000 ベクトル未満) | flat | 正確な結果、許容される RU コスト |
| 中規模なデータセット、メモリが制約される | quantizedFlat | メモリは減り、良い正確性 |
| 大規模なデータセット、運用ワークロード | diskANN | 最高の RU 効率、最大 95% のリコール |

1. ターミナルに戻り、**Ctrl + C** キーを押して Flask アプリケーションを停止します。

## まとめ

この演習では、Azure Cosmos DB for NoSQL でベクトル インデックス作成戦略を比較しました。 **EnableNoSQLVectorSearch** 機能を使って Azure Cosmos DB アカウントをデプロイし、Entra ID 認証を構成しました。 Python SDK を使って異なるベクトル インデックスの種類で 3 つのコンテナーを作成しました: 正確な検索のための flat、メモリ効率のための quantizedFlat、運用規模の近似検索のための diskANN。 RU の消費と実行時間を追跡しながらベクトル類似検索を実行する Python 関数を作成しました。 Flask Web アプリケーションを使って比較検索を実行し、パフォーマンスの違いを分析しました。 このパターンは、データセットのサイズ、正確性の要件、コストの制約に基づいて、AI アプリケーションに最適なインデックス作成戦略を選ぶのに役立ちます。

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

**setup_containers.py が失敗する**
- Python の仮想環境がアクティブになっていることを確認します
- 環境変数が設定されていることを確認します (**COSMOS_ENDPOINT**、**COSMOS_DATABASE**)
- コンテナーが既に存在する場合、スクリプトは既存のコンテナーを使います

**ベクトル検索がエラーを返す**
- デプロイ スクリプトのオプション **2** を実行してコンテナーが作成されたことを確認します
- 検索を実行する前にサンプル データが読み込まれたことを確認します
- コンテナーでベクトル埋め込みポリシーが構成されていることを確認します

**Cosmos DB の操作が失敗する**
- デプロイ スクリプトのオプション **4** を実行して Cosmos DB アカウントが準備されていることを確認します
- デプロイの間にデータベースが作成されたことを確認します
- アカウントに **EnableNoSQLVectorSearch** 機能があることを確認します

**環境変数に関する問題**
- デプロイ スクリプトのオプション **5** を実行して *.env* ファイルが作成されたことを確認します
- 新しいターミナルを作成した後で **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行します
- **echo $COSMOS_ENDPOINT** (Bash) または **$env:COSMOS_ENDPOINT** (PowerShell) を実行して、変数が設定されていることを確認します

**Python venv のアクティブ化に関する問題**
- Linux/macOSでは **source .venv/bin/activate** を使用します
- Windows PowerShell では **.\.venv\Scripts\Activate.ps1** を使用します
- **activate** スクリプトがない場合は、**python3-venv** パッケージを再インストールして venv を再作成します
