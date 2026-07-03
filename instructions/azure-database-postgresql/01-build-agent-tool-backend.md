---
lab:
  topic: Azure Database for PostgreSQL
  title: Azure Database for PostgreSQL でエージェント ツール バックエンドを構築する
  description: AI エージェント用の永続メモリ ストレージを Azure Database for PostgreSQL を使って構築する方法を学ぶ
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
    - Azure Database for PostgreSQL
---

# Azure Database for PostgreSQL でエージェント ツール バックエンドを構築する

この演習では、AI エージェントのツール バックエンドとして機能する Azure Database for PostgreSQL インスタンスを作成します。 データベースには、操作中にエージェントが読み書きできる会話コンテキストとタスクの状態が格納されます。 あなたはエージェント メモリ用のスキーマを設計し、エージェントのツールとなる Python 関数を構築し、ワークフロー全体をテストします。 このパターンを基盤として AI エージェントを構築すると、永続メモリがセッション間で維持され、中断されたタスクの再開ができるようになります。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードし、デプロイ スクリプトを構成する
- Azure Database for PostgreSQL フレキシブル サーバーを Microsoft Entra 認証付きでデプロイする
- 会話とタスク状態管理のための Python ツール関数を構築する
- エージェント メモリ用のデータベース スキーマ (会話、メッセージ、タスク チェックポイントのテーブルで構成される) を作成する
- 用意されているテスト スクリプトを使ってエージェント メモリ ワークフローをテストする
- SQL を使ってクエリを実行して会話コンテキストを取り出す

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
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/postgresql-build-agent-python.zip
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

このセクションでは、デプロイ スクリプトを実行して PostgreSQL をデプロイします。

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

## ツール関数アプリを完成させる

このセクションでは、AI エージェントが永続化や状態取得のために呼び出せる関数を *agent_tools.py* に追加してこのファイルを完成させます。 これらの関数は、エージェントからデータベースへのインターフェイスとなります。 この演習で後ほど実行する *test_workflow.py* スクリプトでこれらの関数をインポートし、どのようにエージェントに使用されるかを見ていきます。

1. *agent-backend/agent_tools.py* ファイルを VS Code で開きます。

1. **BEGIN CREATE CONVERSATION FUNCTION** というコメントを検索し、次に示すコードをこのコメントの直後に追加します。 この関数は、新しい会話レコードを一意のセッション ID 付きで作成するとともに、オプションのメタデータを JSONB として格納します。

    ```python
    def create_conversation(user_id: str, metadata: dict = None) -> dict:
        """Create a new conversation and return its details."""
        session_id = uuid.uuid4()
        with get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute(
                    """
                    INSERT INTO conversations (session_id, user_id, metadata)
                    VALUES (%s, %s, %s)
                    RETURNING id, session_id, started_at
                    """,
                    (str(session_id), user_id, psycopg.types.json.Json(metadata or {}))
                )
                row = cur.fetchone()
                conn.commit()
                return {
                    "conversation_id": row[0],
                    "session_id": str(row[1]),
                    "started_at": row[2].isoformat()
                }
    ```

1. **BEGIN RETRIEVE CONVERSATION HISTORY FUNCTION** というコメントを検索し、次に示すコードをこのコメントの直後に追加します。 この関数は、1 つの会話からメッセージを時系列順に取り出します。

    ```python
    def get_conversation_history(conversation_id: int, limit: int = 50) -> list:
        """Retrieve recent messages from a conversation."""
        with get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute(
                    """
                    SELECT id, role, content, created_at, metadata
                    FROM messages
                    WHERE conversation_id = %s
                    ORDER BY created_at DESC
                    LIMIT %s
                    """,
                    (conversation_id, limit)
                )
                rows = cur.fetchall()
                return [
                    {
                        "id": row[0],
                        "role": row[1],
                        "content": row[2],
                        "created_at": row[3].isoformat(),
                        "metadata": row[4]
                    }
                    for row in reversed(rows)  # Return in chronological order
                ]
    ```

1. **BEGIN TASK CHECKPOINT FUNCTIONS** というコメントを検索し、次に示すコードをこのコメントの直後に追加します。 この関数はアップサート パターンを使ってタスク状態を保存または更新するものであり、これでエージェントは中断されたタスクを再開できるようになります。

    ```python
    def save_task_state(conversation_id: int, task_name: str, status: str, checkpoint_data: dict) -> dict:
        """Save or update a task checkpoint."""
        with get_connection() as conn:
            with conn.cursor() as cur:
                cur.execute(
                    """
                    INSERT INTO task_checkpoints (conversation_id, task_name, status, checkpoint_data)
                    VALUES (%s, %s, %s, %s)
                    ON CONFLICT (conversation_id, task_name)
                    DO UPDATE SET
                        status = EXCLUDED.status,
                        checkpoint_data = EXCLUDED.checkpoint_data,
                        updated_at = CURRENT_TIMESTAMP
                    RETURNING id, updated_at
                    """,
                    (conversation_id, task_name, status, psycopg.types.json.Json(checkpoint_data))
                )
                row = cur.fetchone()
                conn.commit()
                return {
                    "checkpoint_id": row[0],
                    "updated_at": row[1].isoformat()
                }
    ```

1. 変更内容を *agent_tools.py* ファイルに保存します。

1. 少し時間をかけて、このアプリのコード全体をレビューします。

次に、Azure リソースのデプロイを完了します。

## Azure リソースのデプロイを完了する

このセクションでは、デプロイ スクリプトに戻って Microsoft Entra 管理者を構成し、PostgreSQL サーバーの接続情報を取得します。

1. **[Create PostgreSQL server with Entra authentication]** 操作が完了したら、「**2**」を入力して **[Configure Microsoft Entra administrator]** オプションを起動します。 これで自分の Azure アカウントがデータベース管理者として設定されます。

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

    >**注:** ターミナルは開いたままにします。 閉じて新しいターミナルを作成する場合は、このコマンドをもう一度実行して環境変数を再度作成することが必要になる可能性があります。

    >**注:** アクセス トークンは約 1 時間後に失効します。 後で再接続が必要な場合は、このスクリプトをもう一度実行し、オプション **4** を選択して新しいトークンを生成してから、もう一度変数をエクスポートしてください。

次に、エージェントをサポートするスキーマを作成します。

## エージェント メモリ スキーマを **psql** で作成する

このセクションでは、**psql** コマンドライン ツールを使って PostgreSQL サーバーに接続し、エージェント メモリ用のデータベース スキーマを作成します。 このスキーマには 3 つのテーブルが含まれます。1 つは会話 (エージェント セッション) 用、2 つめはその会話の中のメッセージ用、3 つめはエージェントが中断した作業を再開できるようにするためのタスク チェックポイント用です。

1. 次のコマンドを実行して、環境変数を使用してサーバーに接続します。 **PGPASSWORD** 環境変数が自動的に認証に使用されます。

    **Bash**
    ```bash
    psql "host=$DB_HOST port=5432 dbname=$DB_NAME user=$DB_USER sslmode=require"
    ```

    **PowerShell**
    ```powershell
    psql "host=$env:DB_HOST port=5432 dbname=$env:DB_NAME user=$env:DB_USER sslmode=require"
    ```

1. 接続を確認するために、次のコマンドを実行して PostgreSQL のバージョンを調べます。

    ```sql
    SELECT version();
    ```
1. 次のコマンドを実行してエージェント バックエンド用のデータベースを作成します。 **\c** コマンドで新しいデータベースに接続します。

    ```sql
    CREATE DATABASE agent_memory;
    \c agent_memory
    ```

1. 次のコマンドを実行して会話 (エージェント セッション) 用のテーブルを作成します。 このテーブルにセッション メタデータが格納され、メッセージが特定の会話にリンクされます。

    ```sql
    CREATE TABLE conversations (
        id BIGSERIAL PRIMARY KEY,
        session_id UUID NOT NULL UNIQUE,
        user_id VARCHAR(255) NOT NULL,
        started_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
        ended_at TIMESTAMP WITH TIME ZONE,
        metadata JSONB DEFAULT '{}'::jsonb
    );
    ```

1. 次のコマンドを実行して会話内のメッセージ用のテーブルを作成します。 このテーブルには各メッセージのロール (ユーザー、アシスタント、システム、またはツール) と内容が格納されます。

    ```sql
    CREATE TABLE messages (
        id BIGSERIAL PRIMARY KEY,
        conversation_id BIGINT NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
        role VARCHAR(50) NOT NULL CHECK (role IN ('user', 'assistant', 'system', 'tool')),
        content TEXT NOT NULL,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
        metadata JSONB DEFAULT '{}'::jsonb
    );
    ```

1. 次のコマンドを実行してタスク チェックポイント用のテーブルを作成します。 このテーブルでエージェント状態の永続化が可能になるため、エージェントは中断したタスクを再開できるようになります。

    ```sql
    CREATE TABLE task_checkpoints (
        id BIGSERIAL PRIMARY KEY,
        conversation_id BIGINT REFERENCES conversations(id) ON DELETE CASCADE,
        task_name VARCHAR(255) NOT NULL,
        status VARCHAR(50) NOT NULL CHECK (status IN ('pending', 'in_progress', 'completed', 'failed')),
        checkpoint_data JSONB NOT NULL DEFAULT '{}'::jsonb,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
    );
    ```

1. 次のコマンドを実行して、一般的なクエリを最適化するインデックスを作成します。 これらのインデックスを作成しておくと、会話またはタイムスタンプを指定してメッセージを取り出すときのパフォーマンスが向上します。

    ```sql
    CREATE INDEX idx_messages_conversation_id ON messages(conversation_id);
    CREATE INDEX idx_messages_created_at ON messages(created_at);
    CREATE INDEX idx_task_checkpoints_conversation_id ON task_checkpoints(conversation_id);
    CREATE INDEX idx_conversations_session_id ON conversations(session_id);
    ```

1. この演習で完成させたアプリでは **ON CONFLICT** が使用されていますが、これには一意制約が必要です。 これを追加するために、次のコマンドをターミナル内の **psql** セッションで実行します。

    ```sql
    ALTER TABLE task_checkpoints
    ADD CONSTRAINT unique_conversation_task
    UNIQUE (conversation_id, task_name);
    ```

1. スキーマが正しく作成されたことを確認するために、次のコマンドを実行します。

    ```sql
    \dt
    ```

    3 つのテーブルが一覧表示されるはずです。

1. 「`exit`」と入力して **psql** セッションを閉じ、ターミナルに戻ります。

## エージェント メモリ ワークフローをテストする

このセクションでは、ツール関数が正しく動作することを、テスト スクリプトを実行して確認します。 *test_workflow.py* スクリプトがプロジェクト ファイルに含まれており、これを実行すると会話の作成、メッセージの格納、タスク チェックポイントの管理を実際に行うことができます。

1. 次のコマンドを実行して *agent-backend* ディレクトリに移動します。

    ```
    cd agent-backend
    ```

1. 次のコマンドを実行して *test_workflow.py* アプリ用の仮想環境を作成します。 この演習に使う環境に応じて、コマンドは **python** または **python3** となります。

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

1. 次のコマンドを実行してこのアプリ用の Python 依存関係をインストールします。 これで PostgreSQL 接続用の **psycopg** ライブラリと Microsoft Entra 認証用の **azure-identity** がインストールされます。

    ```bash
    pip install -r requirements.txt
    ```

1. 次のコマンドを実行してテスト スクリプトを実行します。 これまでに作成したすべてのエージェント ツール関数がこのスクリプトで実行されます。

    ```bash
    python test_workflow.py
    ```

1. 出力には、各ステップが正常に完了したことが表示されるはずです。これはこのエージェントで会話の作成、メッセージの格納、タスク状態の保存、履歴の取得ができることを示しています。

1. 省略可能: *test_workflow.py* ファイルを開いてコードを確認します。

## クエリを実行して会話コンテキストを取り出す

このセクションでは、エージェントが意思決定に使うであろうデータを、実際にクエリを実行して取り出してみます。

1. 次のコマンドを実行して、環境変数を使用して **agent_memory** データベースに接続します。

    **Bash**
    ```bash
    psql "host=$DB_HOST port=5432 dbname=agent_memory user=$DB_USER sslmode=require"
    ```

    **PowerShell**
    ```powershell
    psql "host=$env:DB_HOST port=5432 dbname=agent_memory user=$env:DB_USER sslmode=require"
    ```

    >**ヒント:** psql でのクエリ結果が表示されるときに、結果が現在のターミナル ウィンドウに収まらない場合はページャーが使用されます。 このようになった場合は、**q** キーを押すとページャーが終了して psql プロンプトに戻ります。 ターミナル ウィンドウを最大化しておくと、これが発生することが減り、コマンドの結果の確認もしやすくなります。

1. 次のクエリを実行して、ある 1 人のユーザーのすべての会話を見つけます。 テスト スクリプトで作成された会話の中に、**user_id** が **user_123** に設定されたものが 1 件あります。

    ```sql
    SELECT id, session_id, started_at, metadata
    FROM conversations
    WHERE user_id = 'user_123'
    ORDER BY started_at DESC;
    ```

1. 次のクエリを実行して、すべての会話から最近のメッセージを取得します。 これを実行すると、テスト スクリプトによって格納されたメッセージが返されます。

    ```sql
    SELECT c.session_id, m.role, m.content, m.created_at
    FROM messages m
    JOIN conversations c ON m.conversation_id = c.id
    ORDER BY m.created_at DESC
    LIMIT 10;
    ```

1. 次のクエリを実行して、完了したタスクを見つけます。 テスト スクリプトによって、タスクの状態はワークフロー終了時に **completed** に更新されています。

    ```sql
    SELECT
        c.session_id,
        t.task_name,
        t.status,
        t.checkpoint_data,
        t.updated_at
    FROM task_checkpoints t
    JOIN conversations c ON t.conversation_id = c.id
    WHERE t.status = 'completed';
    ```

1. 次のクエリを実行して各会話内のメッセージをロール別に数えます。 これはユーザー、アシスタント、システム、ツールのメッセージの分布を理解するのに役立ちます。

    ```sql
    SELECT
        c.id AS conversation_id,
        m.role,
        COUNT(*) AS message_count
    FROM conversations c
    JOIN messages m ON c.id = m.conversation_id
    GROUP BY c.id, m.role
    ORDER BY c.id, m.role;
    ```

1. psql プロンプトで「**quit**」と入力して終了します。

## まとめ

この演習では、AI エージェントのための PostgreSQL ベースのツール バックエンドを構築しました。 Azure Database for PostgreSQL フレキシブル サーバーを Microsoft Entra 認証付きでデプロイし、エージェントが会話やタスク状態を管理するために呼び出せる Python 関数を作成し、会話、メッセージ、タスク チェックポイント用のテーブルから成るデータベース スキーマを設計しました。 ワークフローをテストするために、エージェントのオペレーションをシミュレートするスクリプトを実行してから、SQL を使用してクエリを実行して格納済みのデータを取り出しました。 このパターンを使用すると、AI エージェントが永続メモリをセッション間で維持できるため、中断されたタスクを再開できるようになります。

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

**Python テスト スクリプトが失敗する**
- Python 仮想環境がアクティブになっていることを確認します (ターミナル プロンプトに **(.venv)** が表示されるはずです)
- 依存関係がインストールされていることを確認します: **pip install -r requirements.txt**
- **agent_memory** データベースとすべてのテーブルを **psql** で作成したことを確認します
- **task_checkpoints** テーブルに対する一意制約を追加したことを確認します

**データベースまたはテーブルが見つからないというエラー**
- **psql** で **\c agent_memory** を使用して **agent_memory** データベースに接続したことを確認します
- テーブルが存在していることを **psql** で **\dt** を実行して確認します
- テーブルが存在しない場合は CREATE TABLE 文をもう一度実行します

**Python venv アクティブ化の問題**
- Linux/macOS では **source .venv/bin/activate** を使用します
- Windows PowerShell では **.\.venv\Scripts\Activate.ps1** を使用します
- **activate** スクリプトがない場合は、**python3-venv** パッケージを再インストールして venv を再作成します
