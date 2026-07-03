---
lab:
  topic: Azure Database for PostgreSQL
  title: Azure Database for PostgreSQL でベクトル検索を実装する
  description: ベクトル類似性検索を Azure Database for PostgreSQL と pgvector 拡張機能を使用して実装する方法を学ぶ
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Database for PostgreSQL
---

# Azure Database for PostgreSQL でベクトル検索を実装する

この演習では、Azure Database for PostgreSQL と pgvector 拡張機能を使用して、製品の類似性検索アプリケーションをビルドします。 ベクトル ストレージ機能を有効化し、製品と埋め込み用のデータベース スキーマを作成し、サンプル データを Flask Web アプリケーションを通じて読み込み、類似性検索を実行して関連製品を見つけます。 このパターンは、レコメンデーション システムやセマンティック検索機能、およびその他の AI を活用するアプリケーションの構築の基盤となります。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードし、デプロイ スクリプトを構成する
- Azure Database for PostgreSQL フレキシブル サーバーを Microsoft Entra 認証付きでデプロイする
- サーバーをデプロイしている間に Flask アプリケーション コードを完成させる
- pgvector 拡張機能を有効にして製品テーブル スキーマを作成する
- Flask アプリケーションを実行して製品を読み込み、類似性検索を実行する
- 新しい製品を追加して類似性の結果の変化を観察する

この演習の所要時間は約 **30** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- 必要な Azure サービスをデプロイする権限を持つ Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- [Python 3.12](https://www.python.org/downloads/) 以上。
- [PostgreSQL コマンドライン ツール](https://www.postgresql.org/download/) (**psql**)

## プロジェクト スターター ファイルをダウンロードして Azure サービスをデプロイする

このセクションでは、プロジェクト スターター ファイルをダウンロードし、スクリプトを使用してこの演習に必要なサービスを自分の Azure サブスクリプションにデプロイします。 PostgreSQL サーバーのデプロイは完了まで数分かかります。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/postgresql-vector-search-python.zip
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

    > **注:** PowerShell がデジタル署名されていないためにスクリプトをブロックした場合は、同じターミナル セッション内で次のコマンドを実行し、再度配置スクリプトを実行してください。 このコマンドは、現在の PowerShell プロセスの実行ポリシーのみを変更します。

    ```powershell
    Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
    ```

1. スクリプト メニューが表示されたら、「**1**」と入力して **[Create PostgreSQL server with Entra authentication]** オプションを起動します。 これで、Entra 認証のみが有効化された状態でサーバーが作成されます。 **注:** デプロイが完了するまで 5 分から 10 分ほどかかります。

    >**重要:** デプロイを実行するターミナルは、演習が終了するまで開いたままにしてください。 ターミナルでのデプロイの進行中に、演習の次のセクションに進んでもかまいません。

## クライアント アプリケーションを完成させる

このセクションでは、*app.py* ファイルを完成させるために、PostgreSQL データベースと相互作用するルート ハンドラーを追加します。 これらのルートが、サンプル製品の読み込み、類似性検索の実行、新しい製品の追加を処理します。 この Flask アプリケーションは、ベクトル類似性検索をテストするための Web インターフェイスとなります。

1. VS Code で *client/app.py* ファイルを開きます。

1. **BEGIN LOAD DATA SECTION** というコメントを検索し、次に示すコードをこのコメントの直後に追加します。 このルートでは、製品を JSON ファイルから読み込んで、その製品の埋め込みとともにデータベースに挿入します。

    ```python
    @app.route("/load-data", methods=["POST"])
    def load_data():
        """Load sample products into the database."""
        try:
            products = load_json_file("sample_products.json")

            with get_connection() as conn:
                with conn.cursor() as cur:
                    for product in products:
                        # Check if product already exists
                        cur.execute("SELECT id FROM products WHERE name = %s", (product["name"],))
                        if cur.fetchone():
                            continue

                        # Format embedding as PostgreSQL array (pgvector expects bracket notation)
                        embedding_str = "[" + ",".join(str(x) for x in product["embedding"]) + "]"

                        cur.execute("""
                            INSERT INTO products (name, category, description, price, embedding)
                            VALUES (%s, %s, %s, %s, %s)
                        """, (
                            product["name"],
                            product["category"],
                            product["description"],
                            product["price"],
                            embedding_str
                        ))
                    # Commit all inserts in a single transaction
                    conn.commit()

            flash(f"Successfully loaded {len(products)} sample products!", "success")
        except Exception as e:
            flash(f"Error loading data: {str(e)}", "error")

        return redirect(url_for("index"))
    ```

1. **BEGIN SEARCH SECTION** というコメントを検索し、次に示すコードをこのコメントの直後に追加します。 このルートでは、選択された製品の埋め込みを取り出し、コサイン距離を使用して類似の製品を見つけます。

    ```python
    @app.route("/search", methods=["POST"])
    def search():
        """Find products similar to the selected product using vector similarity."""
        product_id = request.form.get("product_id")

        if not product_id:
            flash("Please select a product", "error")
            return redirect(url_for("index"))

        try:
            with get_connection() as conn:
                with conn.cursor() as cur:
                    # Get the embedding for the selected product
                    cur.execute("SELECT embedding FROM products WHERE id = %s", (product_id,))
                    row = cur.fetchone()

                    if not row:
                        flash("Product not found", "error")
                        return redirect(url_for("index"))

                    # Find similar products using cosine distance
                    # The <=> operator is pgvector's cosine distance operator
                    # Lower distance = more similar (0 = identical, 2 = opposite)
                    cur.execute("""
                        SELECT id, name, category, description, price, embedding <=> %s AS distance
                        FROM products
                        WHERE id != %s
                        ORDER BY distance
                        LIMIT 5
                    """, (row[0], product_id))

                    results = [
                        {"id": r[0], "name": r[1], "category": r[2], "description": r[3], "price": r[4], "distance": r[5]}
                        for r in cur.fetchall()
                    ]

            products = get_products()
            new_products = get_new_products()
            return render_template("index.html", products=products, new_products=new_products, results=results)

        except Exception as e:
            flash(f"Error searching: {str(e)}", "error")
            return redirect(url_for("index"))
    ```

1. **BEGIN ADD PRODUCT SECTION** というコメントを検索し、次に示すコードをこのコメントの直後に追加します。 このルートでは、*new_products.json* ファイルからの製品の 1 つをデータベースに追加します。

    ```python
    @app.route("/add-product", methods=["POST"])
    def add_product():
        """Add a new product from the new_products.json file."""
        product_index = request.form.get("product_index")

        if product_index is None or product_index == "":
            flash("Please select a product to add", "error")
            return redirect(url_for("index"))

        try:
            new_products = load_json_file("new_products.json")
            product = new_products[int(product_index)]

            with get_connection() as conn:
                with conn.cursor() as cur:
                    # Check if product already exists
                    cur.execute("SELECT id FROM products WHERE name = %s", (product["name"],))
                    if cur.fetchone():
                        flash(f"Product '{product['name']}' already exists", "error")
                        return redirect(url_for("index"))

                    # Format embedding as PostgreSQL array
                    embedding_str = "[" + ",".join(str(x) for x in product["embedding"]) + "]"

                    cur.execute("""
                        INSERT INTO products (name, category, description, price, embedding)
                        VALUES (%s, %s, %s, %s, %s)
                    """, (
                        product["name"],
                        product["category"],
                        product["description"],
                        product["price"],
                        embedding_str
                    ))
                    conn.commit()

            flash(f"Successfully added '{product['name']}'!", "success")
        except Exception as e:
            flash(f"Error adding product: {str(e)}", "error")

        return redirect(url_for("index"))
    ```

1. 変更内容を *app.py* ファイルに保存します。

1. 少し時間をかけて、このアプリのコード全体をレビューします。 各ルートでどのように **get_connection()** 関数を使って PostgreSQL に Microsoft Entra 認証で接続しているか、およびどのように **\<=>** 演算子で類似性検索のためのコサイン距離計算を実行しているかに注目してください。

## Azure リソースのデプロイを完了してスキーマを作成する

このセクションでは、pgvector 拡張機能を有効にするとともに、製品テーブルを作成します。このテーブルには埋め込みを格納するためのベクトル列があります。 このスキーマには、製品詳細のためのさまざまな列と、類似性検索に使われる 384 次元埋め込みベクトルが含まれています。

1. PostgreSQL サーバーのデプロイ完了とターミナルに表示されるまで待ちます。

1. デプロイ スクリプトのメニューで「**2**」と入力して Microsoft Entra 認証を構成します。 これで自分の Azure アカウントがデータベース管理者として設定されます。

1. 前の操作が完了したら、「**3**」を入力して **[Check deployment status]** オプションを起動します。 これで、サーバーの準備が整っていることが確認されます。

1. 「**4**」を入力して **[Retrieve connection info and access token]** オプションを起動します。 これで、必要な環境変数が記録されたファイルが作成されます。

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

1. 次のコマンドを実行して PostgreSQL サーバーに **psql** を使用して接続します。 このコマンドでは、前のステップで読み込んだ環境変数が使用されます。

    **Bash**
    ```bash
    psql "host=$DB_HOST dbname=$DB_NAME user=$DB_USER sslmode=require"
    ```

    **PowerShell**
    ```powershell
    psql "host=$env:DB_HOST port=5432 dbname=$env:DB_NAME user=$env:DB_USER sslmode=require"
    ```

1. pgvector 拡張機能を有効にします。 ベクトル データ型を使用するには、事前にこの拡張機能を有効にする必要があります。

    ```sql
    CREATE EXTENSION IF NOT EXISTS vector;
    ```

1. 埋め込み用のベクトル列付きで製品テーブルを作成します。 この埋め込み列には 384 次元が使用されますが、これは一般的なセンテンス トランスフォーマー モデルに合わせたものです。

    ```sql
    CREATE TABLE products (
        id SERIAL PRIMARY KEY,
        name TEXT NOT NULL,
        category TEXT,
        description TEXT,
        price NUMERIC(10, 2),
        embedding vector(330)
    );
    ```

1. 高速の類似性検索を可能にするために HNSW インデックスを作成します。 この種類のインデックスは、ベクトル データに対する近似最近傍クエリに合わせて最適化されています。

    ```sql
    CREATE INDEX products_embedding_idx
    ON products USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
    ```

1. テーブルとインデックスが作成されたことを確認するために、テーブル構造のリストを表示します。

    ```sql
    \d products
    ```

    このテーブル構造には id、name、category、description、price、embedding の列があり、さらに HNSW インデックスがあるはずです。

1. 「**quit**」と入力してセッションを終了します。


## Flask アプリケーションを設定して実行する

このセクションでは、Python の依存関係をインストールし、Flask Web アプリケーションを実行します。 このアプリケーションがブラウザー インターフェイスとなって、製品の読み込み、類似品の検索、データベースへの新しい製品の追加を行うことができます。

1. 次のコマンドを実行して *client* フォルダーに移動します。

    ```
    cd client
    ```

1. 次のコマンドを実行して Python 仮想環境を作成します。 この演習に使う環境に応じて、コマンドは **python** または **python3** となります。

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

1. 次のコマンドを実行して、必要な Python パッケージをインストールします。 *requirements.txt* ファイルには、Web フレームワークのための **flask**、PostgreSQL 接続のための **psycopg**、認証のための **azure-identity** が含まれています。

    ```bash
    pip install -r requirements.txt
    ```

1. Flask アプリケーションを実行します。 このアプリケーションはポート 5000 で起動し、ブラウザーからアクセス可能です。

    ```bash
    python app.py
    ```

    次のように、Flask サーバーが稼働していることを示す出力が表示されるはずです。
    ```
    * Running on all addresses (0.0.0.0)
    * Running on http://127.0.0.1:5000
    ```

1. Web ブラウザーを開き、`http://127.0.0.1:5000` に移動します。 ベクトル検索デモのページに空の製品リストが表示されるはずです。

## 製品を読み込んで類似性探索を実行する

このセクションでは、Web アプリケーションを使ってサンプル製品をデータベースに読み込み、類似性検索を実行します。 これらの製品には事前計算された埋め込みが含まれており、これは各製品のセマンティック的意味を表すものであるため、このアプリケーションは類似品をそれぞれの説明に基づいて見つけることができます。

1. Web ページ上の **[Load Sample Products]** を選択します。 これで製品 10 件とそれぞれの埋め込みがデータベースに挿入されます。

    成功のメッセージが表示され、製品リストにはさまざまな製品、たとえば "Wireless Bluetooth Headphones"、"Gaming Laptop"、"Running Shoes" が含まれるようになったはずです。

1. **[Find Similar Products]** セクションで、**[Wireless Bluetooth Headphones]** をドロップダウンから選択し、**[Find Similar]** を選択します。

    このアプリケーションはベクトル類似度を使用してデータベースに対するクエリを実行し、製品を返しますが、この順序は選択された製品とのセマンティックな近さの順となります。 "Noise Cancelling Earbuds" が結果の先頭近くにありますが、セマンティック的に類似していることが理由です (どちらもオーディオ デバイスです)。

1. さまざまな製品を選んで、その製品のカテゴリや説明に応じてどのように類似製品が変化するかを観察してみてください。

## 新しい製品を追加して変化を観察する

このセクションでは、新しい製品をデータベースに追加して、その製品がどのように類似性検索の結果に現れるかを観察します。 この目的は、ベクトル検索がどのようにデータの変化に適応するかを見ることです。

1. Web ブラウザーの Flask アプリケーションに戻ります。

1. **[Add New Produc]** セクションで、**[Espresso Machine]** をドロップダウンから選択して **[Add Product]** を選びます。

1. **Coffee Maker** に似た製品を検索します。この製品を **[Find Similar Products]** ドロップダウンから選択して **[Find Similar]** を選んでください。

    "Espresso Machine" が検索結果に表示され、距離スコアは低いことに注目してください。これは両製品がセマンティック的に関連している (コーヒー用家電である) からです。

1. ドロップダウンにある残りの製品 (**[Wireless Gaming Mouse]** と **[Fitness Tracker Band]**) を追加して、これらが関連製品の類似性検索の結果にどのように表示されるかを観察します。

## まとめ

この演習では、製品類似性検索アプリケーションを Azure Database for PostgreSQL と pgvector 拡張機能を使用して構築しました。 PostgreSQL フレキシブル サーバーを Microsoft Entra 認証付きでデプロイし、pgvector 拡張機能を有効にし、製品テーブルを 384 次元のベクトル列付きで作成して埋め込みを格納できるようにしました。 類似性クエリを最適化するための HNSW インデックスを追加してから、Flask Web アプリケーションを使用してサンプル製品を読み込み、ベクトル類似性検索をコサイン距離演算子 (**\<=>**) を使って実行しました。 このパターンは、レコメンデーション システムとセマンティック検索機能を構築するときに関連製品を正確なキーワード一致ではなくセマンティック的意味に基づいて発見できるようにする方法を示すものです。

## リソースをクリーンアップする

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

**Flask アプリケーションが起動しない**
- Python 仮想環境がアクティブになっていることを確認します (ターミナル プロンプトに **(.venv)** が表示されるはずです)
- 依存関係がインストールされていることを確認します: **pip install -r requirements.txt**
- 3 つのルート関数すべてが *app.py* に正しく追加されていることを確認してください

**Flask でのデータベース接続エラー**
- Flask を実行するターミナルに環境変数が読み込まれていることを確認します
- 製品テーブルが存在することを確認するために **psql** に接続して **\d products** を実行します
- pgvector 拡張機能が有効化されていることを確認します: **CREATE EXTENSION IF NOT EXISTS vector;**

**読み込み後に製品が何も表示されない**
- Flask ターミナルにエラー メッセージがあるかどうかを調べます
- *sample_products.json* ファイルが *client* フォルダーに存在していることを確認します
- 製品テーブルが正しいスキーマで作成されていることを確認します

**Python venv アクティブ化の問題**
- Linux/macOS では **source .venv/bin/activate** を使用します
- Windows PowerShell では **.\.venv\Scripts\Activate.ps1** を使用します
- Windows で Git Bash を使っている場合は、**source .venv/Scripts/activate** を使用します
