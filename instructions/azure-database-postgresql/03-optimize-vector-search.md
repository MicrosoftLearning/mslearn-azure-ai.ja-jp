---
lab:
  topic: Azure Database for PostgreSQL
  title: Azure Database for PostgreSQL でベクトル検索のパフォーマンスを最適化する
  description: Azure Database for PostgreSQL でのベクトル検索のパフォーマンスをインデックスとパラメーター チューニングを使って最適化する方法を学ぶ
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Database for PostgreSQL
---

# Azure Database for PostgreSQL でベクトル検索のパフォーマンスを最適化する

この演習では、Azure Database for PostgreSQL インスタンスをデプロイし、ベクトル検索ワークロード用に最適化します。 テスト データをベクトル埋め込み付きで作成し、ベースライン パフォーマンスを分析し、IVFFlat と HNSW のインデックスを構築して比較し、検索パラメーターをチューニングします。 これらの手法は、実際の AI アプリケーションで高速の類似性検索を大規模なデータセット全体にわたって行う必要がある場合に不可欠です。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードし、デプロイ スクリプトを構成する
- Azure Database for PostgreSQL フレキシブル サーバーを Microsoft Entra 認証付きでデプロイする
- テスト データセットをベクトル埋め込み付きで作成する
- インデックスなしのベースライン ベクトル検索のパフォーマンスを分析する
- IVFFlat と HNSW のベクトル インデックスを作成して比較する
- 速度とリコールのバランスを取るようにインデックス パラメーターをチューニングする

この演習の所要時間は約 **30** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- Azure サブスクリプション (必要な Azure サービスをデプロイするためのアクセス許可が付与されていること)。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- [PostgreSQL コマンドライン ツール](https://www.postgresql.org/download/) (**psql**)

## プロジェクト スターター ファイルをダウンロードして Azure サービスをデプロイする

このセクションでは、プロジェクト スターター ファイルをダウンロードし、スクリプトを使用してこの演習に必要なサービスを自分の Azure サブスクリプションにデプロイします。 PostgreSQL サーバーのデプロイは完了まで数分かかります。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/postgresql-optimize-vector-search-python.zip
    ```

1. このファイルを自分のシステム内の、このプロジェクトの作業に使用する場所にコピーするか移動します。 その後で、ファイルの圧縮を解除して任意のフォルダーに出力します。

1. Visual Studio Code (VS Code) を起動し、メニューで **[ファイル] > [フォルダーを開く...]** を選択してから、プロジェクト ファイルが含まれているフォルダーを選択します。

1. このプロジェクトのデプロイ スクリプトは、Bash (*azdeploy.sh*) と PowerShell (*azdeploy.ps1*) の両方があります。 自分の環境に適したファイルを開き、スクリプトの先頭の 2 つの値を自分のニーズに合わせて変更してから、変更を保存します。 **注:** スクリプトの他の部分は変更しないでください。

    ```
    "<your-resource-group-name>" # Resource Group name
    "<your-azure-region>" # Azure region for the resources
    ```

1. メニュー バーで、**[ターミナル] > [新しいターミナル]** を選択して VS Code の中にターミナル ウィンドウを開きます。

    >**ヒント:** この演習の作業はすべてターミナルの中で行います。 パネル サイズを最大化すると、コマンドの結果が見やすくなります。

1. 次のコマンドを実行して自分の Azure アカウントにログインします。 画面の指示に従って、演習用の Azure アカウントとサブスクリプションを選択します。

    ```
    az login
    ```

1. 次のコマンドを実行して、演習に必要なリソース プロバイダーが自分のサブスクリプションに確実に存在する状態にします。

    ```azurecli
    az provider register --namespace Microsoft.DBforPostgreSQL
    ```

### Azure でリソースを作成する

このセクションでは、デプロイ スクリプトを実行して PostgreSQL サーバーをデプロイし、認証を構成します。

1. 現在の場所がプロジェクトのルート ディレクトリであることを確認してから、該当するコマンドをターミナルで実行してデプロイ スクリプトを起動します。

    **Bash**
    ```bash
    bash azdeploy.sh
    ```

    **PowerShell**
    ```powershell
    ./azdeploy.ps1
    ```

1. スクリプト メニューが表示されたら、「**1**」と入力して **[Create PostgreSQL server with Entra authentication]** オプションを起動します。 これで、Entra 認証のみが有効化された状態でサーバーが作成されます。 **注:** デプロイが完了するまで 5 分から 10 分ほどかかります。

    >**重要:** デプロイを実行するターミナルは、演習が終了するまで開いたままにしてください。 ターミナルでのデプロイの進行中に、演習の次のセクションに進んでもかまいません。

## ベクトル インデックスの概念を復習する

このセクションでは、演習で後ほど応用する、ベクトル インデックス作成に関する主要な概念を復習します。 これらのトレードオフを理解しておくと、ベクトル検索を最適化するときに情報に基づいた判断ができるようになります。

### IVFFlat インデックス

IVFFlat (Inverted File with Flat compression) とは、ベクトルの集合を**リスト**と呼ばれるクラスターに分割するものです。 これで、検索時にスキャンされるのはデータセット全体ではなく、近隣のクラスター内のベクトルのみとなります。

主要パラメーター:

- **lists**: 作成するクラスターの数。 初期設定として、行数が 100 万以下の場合は `rows / 1000` とするとよいでしょう。 リストが多いほど検索は速くなりますが、インデックスの構築に時間がかかります。
- **probes**: クエリ時に検索されるクラスターの数。 値が大きいほどリコール (つまり真の最近傍を見つける能力) が向上しますが、待ち時間が長くなります。

### HNSW インデックス

HNSW (Hierarchical Navigable Small World) とは、多層グラフ構造を構築するものです。 上の層ほどノード数が少ないため速くナビゲーションでき、下の層ほどノード数が多くなるため正確な検索ができます。

主要パラメーター:

- **m**: ノードあたりの最大接続数。 値が大きいほどリコールが向上しますが、メモリ使用量とビルド時間が増加します。 既定値は 16 です。
- **ef_construction**: インデックス構築時の動的候補リストのサイズ。 値が大きいほど良質のグラフが作られますが、構築に要する時間が長くなります。 既定値は 64 です。
- **ef_search**: 検索時の動的候補リストのサイズ。 値が大きいほどリコールが向上しますが、待ち時間が長くなります。 既定値は 40 です。

### それぞれをいつ使用するか

| 考慮事項 | IVFFlat | HNSW |
|---------------|---------|------|
| ビルド時 | より高速 | 低速 |
| クエリ速度 | 速い | より高速 |
| メモリ使用量 | 削減 | 増加 |
| リコール正確性 | チューニングすれば良好 | 初期設定のままでも優れている |
| 更新パフォーマンス | 再構築が必要 | 増分をサポート |

この演習では、両方の種類のインデックスをテストしてトレードオフを実際に体験します。

## Azure リソースのデプロイを完了する

このセクションでは、デプロイ スクリプトに戻って Microsoft Entra 管理者を構成し、PostgreSQL サーバーの接続情報を取得します。

1. **[Create PostgreSQL server with Entra authentication]** 操作が完了したら、「**2**」を入力して **[Configure Microsoft Entra administrator]** オプションを起動します。 これで自分の Azure アカウントがデータベース管理者として設定されます。

1. 前の操作が完了したら、「**3**」を入力して **[Check deployment status]** オプションを起動します。 これで、サーバーの準備が整っていることが確認されます。

1. 「**4**」を入力して **[Retrieve connection info and access token]** オプションを起動します。 これで、必要な環境変数が記録されたファイルが作成されます。

1. 「**5**」を入力してデプロイ スクリプトを終了します。

1. 次のコマンドを実行して、前のステップで作成したファイルからターミナル セッションに環境変数を読み込みます。

    **Bash**
    ```bash
    source .env
    ```

    **PowerShell**
    ```powershell
    . .\.env.ps1
    ```

    >**注:** ターミナルを開いたままにしてください。 閉じて新しいターミナルを作成する場合は、このコマンドをもう一度実行して環境変数を読み込むことが必要になる可能性があります。

    >**注:** アクセス トークンは約 1 時間後に失効します。 後で再接続が必要な場合は、このスクリプトをもう一度実行し、オプション **4** を選択して新しいトークンを生成してから、もう一度変数をエクスポートしてください。

## データベース スキーマとテスト データを作成する

このセクションでは、PostgreSQL サーバーに接続してテーブルを作成し、テスト用の製品データとベクトル埋め込みを用意します。

1. 次のコマンドを実行して、環境変数を使用してサーバーに接続します。 **PGPASSWORD** 環境変数が自動的に認証に使用されます。

    **Bash**
    ```bash
    psql "host=$DB_HOST port=5432 dbname=$DB_NAME user=$DB_USER sslmode=require"
    ```

    **PowerShell**
    ```powershell
    psql "host=$env:DB_HOST port=5432 dbname=$env:DB_NAME user=$env:DB_USER sslmode=require"
    ```

    >**ヒント:** psql でのクエリ結果が表示されるときに、結果が現在のターミナル ウィンドウに収まらない場合はページャーが使用されます。 このようになった場合は、**q** キーを押すとページャーが終了して psql プロンプトに戻ります。 ターミナル ウィンドウを最大化しておくと、これが発生することが減り、コマンドの結果の確認もしやすくなります。

1. 次のコマンドを実行して pgvector 拡張機能を有効にします。 PostgreSQL の拡張機能を使用するには、事前に明示的に有効化する必要があります。 pgvector 拡張機能を有効にすると、この演習で使用する **vector** データ型と **<=>** (コサイン距離) などの演算子が追加されます。 Azure Database for PostgreSQL には pgvector が含まれていますが、既定では有効化されません。

    ```sql
    CREATE EXTENSION IF NOT EXISTS vector;
    ```

1. 次のコマンドを実行して、ベクトル列を持つ製品テーブルを作成します。 **vector(384)** というデータ型には 384 次元の埋め込みが格納されますが、これはセンテンス埋め込みモデル用に一般的なサイズです。

    ```sql
    CREATE TABLE products (
        id BIGSERIAL PRIMARY KEY,
        name TEXT NOT NULL,
        category_id INTEGER NOT NULL,
        price NUMERIC(10,2) NOT NULL,
        in_stock BOOLEAN DEFAULT true,
        embedding vector(384)
    );
    ```

1. 次のコマンドを実行して、ランダムな埋め込みから成るテスト データを生成します。 これで、ランダムな 384 次元のベクトルを持つ製品 100,000 件が作成されます。

    ```sql
    INSERT INTO products (name, category_id, price, in_stock, embedding)
    SELECT
        'Product ' || i,
        (random() * 20)::int + 1,
        (random() * 1000)::numeric(10,2),
        random() > 0.1,
        ('[' || array_to_string(ARRAY(
            SELECT (random() * 2 - 1)::float4
            FROM generate_series(1, 384)
        ), ',') || ']')::vector
    FROM generate_series(1, 100000) AS i;
    ```

1. データが作成されたことを確認するために、次のコマンドを実行します。 100,000 行と表示されるはずです。

    ```sql
    SELECT COUNT(*) FROM products;
    ```

1. テストの条件をそろえるために、次のコマンドを実行してクエリ ベクトルを作成します。 この一時テーブルにランダムな埋め込みが 1 件格納され、これをこの演習で全体で使用します。

    ```sql
    CREATE TEMP TABLE query_vectors AS
    SELECT ('[' || array_to_string(ARRAY(
        SELECT (random() * 2 - 1)::float4
        FROM generate_series(1, 384)
    ), ',') || ']')::vector AS embedding;
    ```

## ベースライン パフォーマンスを分析する

このセクションでは、インデックスを一切使わないベクトル検索のパフォーマンスを測定することによってベースラインを確立します。

1. 次のコマンドを実行します。これはベクトル類似性クエリを実行して実行プランをキャプチャするものです。

    ```sql
    EXPLAIN ANALYZE
    SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance
    FROM products
    ORDER BY embedding <=> (SELECT embedding FROM query_vectors)
    LIMIT 10;
    ```

1. 出力を調べます。 プランの中に **Seq Scan** があるはずです。これは PostgreSQL が 100,000 行すべてをスキャンしていることを示しています。 一番下の **Execution Time** の値に注目してください。

1. 一貫性のある測定値を得るために、このクエリをさらに 2 回実行します。 初回の実行は、コールド キャッシュが理由で時間がかかる可能性があります。 最後の実行時間をベースラインとして記録してください。

## IVFFlat と HNSW のインデックスを作成して比較する

このセクションでは、両方の種類のインデックスを作成して両者のパフォーマンスを比較します。

### IVFFlat インデックスを作成する

1. 次のコマンドを使用して IVFFlat インデックスを作成します。 100,000 行の場合は、リスト数の初期値として 100 が妥当です (`rows / 1000` をガイドラインとして使用)。 インデックス構築に要する時間に注目してください。

    ```sql
    CREATE INDEX idx_products_embedding_ivfflat
    ON products USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
    ```

1. 次のコマンドを実行して、同じクエリを IVFFlat インデックスありの状態で実行します。 プランに **Index Scan using idx_products_embedding_ivfflat** と表示されていることを確認します。 実行時間を記録します。

    ```sql
    EXPLAIN ANALYZE
    SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance
    FROM products
    ORDER BY embedding <=> (SELECT embedding FROM query_vectors)
    LIMIT 10;
    ```

1. 次のコマンドを実行して probes が低い状態でテストします。 このようにすると速くなりますが、フル スキャンに比べると真の最近傍を見逃す可能性があります。その理由は、検索するクラスターが 1 つだけであるからです。 リコールへの影響は、時間に関する出力には現れないため、測定するには実際の結果を比較する必要があります。

    ```sql
    SET ivfflat.probes = 1;
    EXPLAIN ANALYZE
    SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance
    FROM products
    ORDER BY embedding <=> (SELECT embedding FROM query_vectors)
    LIMIT 10;
    ```

1. 次のコマンドを実行して probes が高い状態 (時間がかかるが、リコールが上昇する) でテストします。 実行時間を記録します。

    ```sql
    SET ivfflat.probes = 50;
    EXPLAIN ANALYZE
    SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance
    FROM products
    ORDER BY embedding <=> (SELECT embedding FROM query_vectors)
    LIMIT 10;
    ```

### HNSW インデックスを作成する

1. 次のコマンドを実行して IVFFlat インデックスを削除します。これは、HNSW を単独でテストできるようにするためです。

    ```sql
    DROP INDEX idx_products_embedding_ivfflat;
    ```

1. 次のコマンドを実行して、インデックス構築に使用できるメモリを増やします。 HNSW インデックスの構築には IVFFlat に比べて多くのメモリが必要になります。

    ```sql
    SET maintenance_work_mem = '256MB';
    ```

1. 次のコマンドを実行して HNSW インデックスを作成します。 構築時間に注目してください。一般的には IVFFlat よりも長くなります。

    ```sql
    CREATE INDEX idx_products_embedding_hnsw
    ON products USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
    ```

1. 次のコマンドを実行してクエリを HNSW インデックスありの状態で実行します。

    ```sql
    EXPLAIN ANALYZE
    SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance
    FROM products
    ORDER BY embedding <=> (SELECT embedding FROM query_vectors)
    LIMIT 10;
    ```

1. 次のコマンドを実行して ef_search が低い状態でテストします。 このようにすると速くなりますが、真の最近傍がいくつか見逃される可能性があります。これは、検索時に探索される候補パスの数が少なくなるからです。 リコールへの影響は、時間に関する出力には現れません。

    ```sql
    SET hnsw.ef_search = 20;
    EXPLAIN ANALYZE
    SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance
    FROM products
    ORDER BY embedding <=> (SELECT embedding FROM query_vectors)
    LIMIT 10;
    ```

1. 次のコマンドを実行して ef_search が高い状態 (リコールが上昇する) でテストします。 実行時間を記録します。

    ```sql
    SET hnsw.ef_search = 100;
    EXPLAIN ANALYZE
    SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance
    FROM products
    ORDER BY embedding <=> (SELECT embedding FROM query_vectors)
    LIMIT 10;
    ```

### 結果を比較する

これまでの、さまざまな構成での実行時間を比較します。 次のことがわかるはずです。

- **シーケンシャル スキャン**が最も時間がかかりますが、これは 100,000 行すべてを調べるからです
- **IVFFlat と probes=1** はインデックス使用時に最も速くなる選択肢ですが、真の最近傍がいくつか見逃される可能性があります
- **HNSW** はリコール レベルが同程度の場合に IVFFlat と比較してクエリが速くなります
- **probes** (IVFFlat) または **ef_search** (HNSW) を大きくすると正確性が向上しますが、待ち時間が長くなります

## インデックスを用いたメタデータ フィルタリングを実装する

このセクションでは、ベクトル類似性とメタデータ フィルターを組み合わせたクエリをテストします。 実際のアプリケーションでは、純粋なベクトル検索はまれであり、一般的にはカテゴリ、日付範囲、価格、またはその他の属性でフィルタリングしてから類似品を見つけることになります。 このような複合クエリを最適化するには、PostgreSQL で複数の種類のインデックスの組み合わせがどのように使用されるかを理解する必要があります。

1. 次のコマンドを実行して B ツリー インデックスをカテゴリ列に作成します。

    ```sql
    CREATE INDEX idx_products_category ON products (category_id);
    ```

1. 次のコマンドを実行してフィルター付きベクトル検索を実行します。 実行プランを調べると、PostgreSQL が HNSW インデックスを使ってベクトル類似性を見つけてからカテゴリ フィルターを適用していることがわかるはずです。 スクリプトの実行時間を書き留めておきます。

    ```sql
    EXPLAIN ANALYZE
    SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance
    FROM products
    WHERE category_id = 5
    ORDER BY embedding <=> (SELECT embedding FROM query_vectors)
    LIMIT 10;
    ```

1. さらに選択を絞り込むフィルターを使ってテストするために、次のコマンドを実行します。 複数のフィルター条件があるときは、B ツリー インデックスに対する**ビットマップ インデックス スキャン**と事後フィルタリングの組み合わせが使用されることも、別の戦略でプランが作成されることもあります。 実行時間を前のクエリと比較してください。

    ```sql
    EXPLAIN ANALYZE
    SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance
    FROM products
    WHERE category_id = 5 AND price BETWEEN 100 AND 200
    ORDER BY embedding <=> (SELECT embedding FROM query_vectors)
    LIMIT 10;
    ```

1. 次のコマンドを実行して、このフィルターの組み合わせのための複合インデックスを作成します。

    ```sql
    CREATE INDEX idx_products_category_price ON products (category_id, price);
    ```

1. 前のクエリをもう一度実行して実行プランを比較します。 この複合インデックスが存在すると、PostgreSQL がベクトル検索の実行前または実行中に効率的にフィルタリングできるようになり、その結果として実行時間が短縮する可能性があります。

    ```sql
    EXPLAIN ANALYZE
    SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance
    FROM products
    WHERE category_id = 5 AND price BETWEEN 100 AND 200
    ORDER BY embedding <=> (SELECT embedding FROM query_vectors)
    LIMIT 10;
    ```

## まとめ

この演習では、以下のことを行います。

- Azure Database for PostgreSQL フレキシブル サーバーを Microsoft Entra 認証付きでデプロイしました
- ベクトル埋め込み 100,000 件から成るテスト データセットを作成しました
- インデックスなしの状態でのベクトル クエリのベースライン パフォーマンスを確立しました
- IVFFlat と HNSW のインデックスを作成して比較しました
- インデックス パラメーター (**probes** と **ef_search**) を、正確性と速度のバランスを取るようにチューニングしました
- B ツリー インデックスを使用するメタデータ フィルタリングを実装しました

これらの手法を使用すると、Azure Database for PostgreSQL を実際のベクトル検索ワークロードに合わせて最適化できます。

# リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループと、そのグループ内のすべてのリソースを削除します。 **\<rg-name>** は、この演習で選択した名前に置き換えてください。 このコマンドを実行すると Azure の中でバックグラウンド タスクが起動されてリソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の中で問題が発生した場合は、次のステップを試してみてください。

**psql の接続に失敗する**
- *.env* ファイルが作成されたことを確認します (デプロイ スクリプトを実行してオプション **4** を選択する)
- **source .env** (Bash) または **. .\.env.ps1** (PowerShell) を実行して環境変数を読み込んだことを確認します
- アクセス トークンは約 1 時間後に失効するため、デプロイ スクリプトをもう一度実行し、オプション **4** を選択して新しいトークンを生成します
- サーバーの準備ができていることを確認します (デプロイ スクリプトを実行してオプション **3** を選択する)

**アクセス拒否または認証エラー**
- Microsoft Entra 管理者が構成されたことを確認します (デプロイ スクリプトを実行してオプション **2** を選択する)
- ターミナル セッションで **PGPASSWORD** が正しく設定されていることを確認します
- 使用している **DB_USER** の値 (Azure アカウントのメールアドレス) が正しいことを確認します

**インデックス構築に時間がかかりすぎるか失敗する**
- HNSW インデックスの構築は IVFFlat よりも時間がかかります (ベクトル 100,000 件の場合は 1 分から 2 分)
- 構築がタイムアウトした場合は、CPU とメモリのメトリックを Azure Monitor で調べてください
- テスト用のデータセットのサイズ縮小を検討します
