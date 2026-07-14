---
lab:
  topic: Integrate backend services
  title: Azure Functions を使用して MCP サーバーを作成する
  description: Azure Functions を使用して、AI エージェントや言語モデルが検出と呼び出しを行うためのツール トリガー関数を公開する MCP サーバーを作成してテストする方法を学習します。
  level: 300
  duration: 25
  islab: true
  primarytopics:
    - Azure
    - Azure Functions
---

# Azure Functions を使用して MCP サーバーを作成する

モデル コンテキスト プロトコル (MCP) は、AI エージェントや言語モデルが外部ツールを検出して呼び出す方法を定義するオープン標準です。 Azure Functions には MCP 拡張機能が含まれており、関数アプリを MCP サーバーとして公開できます。これにより、各関数は、MCP クライアントから呼び出すことができるツールになります。

この演習では、MCP 拡張機能を使用して Azure Functions プロジェクトを作成し、ドキュメント処理用のツール トリガー関数を定義して、MCP サーバー設定を構成し、エージェント モードの GitHub Copilot から接続してサーバーをローカルでテストします。

>**注:** この演習では Azure Functions MCP 拡張機能を使用しますが、この拡張機能は現在も活発に進化しています。 最新のセットアップ手順、API サーフェス、構成オプションについては、[Azure Functions MCP 拡張機能のドキュメント](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-mcp)を参照してください。

この演習で実行されるタスク:

- MCP 拡張機能を使用して新しい Azure Functions プロジェクトを作成する
- *host.json* で MCP サーバーの設定を構成する
- *function_app.py* で MCP ツール トリガー関数を定義する
- Python 環境を確認する
- エージェント モードで GitHub Copilot を使用して MCP サーバーをローカルでテストする

この演習の所要時間は約 **25** 分です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- Visual Studio Code 用 [Azure Functions 拡張機能](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurefunctions)。
- [Azure Functions Core Tools](https://learn.microsoft.com/azure/azure-functions/functions-run-local) v4 以降。
- [Python 3.9](https://www.python.org/downloads/) 以降。
- Visual Studio Code の [GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot) 拡張機能。

## MCP 拡張機能を使用して新しい Functions プロジェクトを作成する

このセクションでは、Azure Functions 拡張機能を使用して Visual Studio Code で新しい Azure Functions プロジェクトを作成し、*host.json* で MCP サーバーの設定を構成します。 この拡張機能は、プロジェクト構造をスキャフォールディングし、統合デバッグ用の *.vscode* 構成ファイルを生成し、Python v2 プログラミング モデルを設定します。

1. プロジェクト用のフォルダー (例: *mcp-server-functions*) を作成し、メニューで **[ファイル] > [フォルダーを開く...]** を選択して Visual Studio Code でそのフォルダーを開きます。

1. **Ctrl + Shift + P** キーを押して、コマンド パレットを開きます。 **Azure Functions: Create Function...** コマンドを実行し、メッセージが表示されたら、次のオプションを選択します。

    | 回答内容 | アクション |
    |--|--|
    | Select the folder... | 前の手順で開いたフォルダーを選択します |
    | プロジェクトの種類を選択する | **[Python]** を選択します |
    | Select a Python interpreter... | **[Python 3.12]** を選択します |
    | Select a template... | **[HTTP トリガー]** を選択します。 |
    | 関数名 | 既定の **[http_trigger]** をそのまま使用します |
    | 承認レベル | **[ANONYMOUS]** を選択します |

    この拡張機能により、*function_app.py*、*host.json*、*local.settings.json*、*requirements.txt*、*.vscode* フォルダー (*launch.json*、*tasks.json*、*extensions.json* を含む) などのプロジェクト構造が作成されます。 *function_app.py* 内のスキャフォールディングされた HTTP トリガー関数は、次のセクションで置き換えられます。

1. *host.json* を開き、その内容を次のコードに置き換えて、変更を保存します。 バインディング タイプの **mcpToolTrigger** はプレビュー機能であり、安定した拡張機能バンドルには含まれていないため、**extensionBundle** は **プレビュー** バンドルを使用する必要があります。 **mcpToolTrigger** セクションでは、MCP クライアントが接続時に表示する MCP サーバー名、バージョン、および命令を定義します。

    ```json
    {
        "version": "2.0",
        "extensionBundle": {
            "id": "Microsoft.Azure.Functions.ExtensionBundle.Preview",
            "version": "[4.*, 5.0.0)"
        },
        "extensions": {
            "mcpToolTrigger": {
                "serverName": "document-tools",
                "serverVersion": "1.0.0",
                "serverInstructions": "Tools for document processing and classification"
            }
        }
    }
    ```

1. *local.settings.json* を開き、その内容を次のコードに置き換えます。

    ```json
    {
      "IsEncrypted": false,
      "Values": {
        "AzureWebJobsStorage": "",
        "FUNCTIONS_WORKER_RUNTIME": "python",
        "AzureWebJobsSecretStorageType": "Files"
      }
    }
    ```

1. プロジェクト内に *.vscode/mcp.json* ファイルを作成し、ローカル MCP エンドポイントを Visual Studio Code に登録します。 このファイルは、Visual Studio Code に MCP サーバーの場所と接続方法を指示します。 このファイルに次のコードを追加し、変更を保存します。

    ```json
    {
        "servers": {
            "document-tools-local": {
                "type": "sse",
                "url": "http://localhost:7071/runtime/webhooks/mcp/sse"
            }
        }
    }
    ```

## MCP ツール トリガー関数を定義する

このセクションでは、MCP クライアントが検出可能なツールとなる 2 つの MCP ツール トリガー関数を定義します。 各関数は、type が **mcpToolTrigger** の **@app.generic_trigger()** デコレーターを使用します。 トリガー構成を使用してツール名、説明、入力プロパティを定義し、関数は接続された MCP クライアントからツール呼び出し要求を受け取ります。

1. Visual Studio Code エクスプローラーのサイド バーで *function_app.py* を開き、その内容を、2 つの MCP ツール トリガー関数を定義する次のコードに置き換えます。

    ```python
    import azure.functions as func
    import json
    import logging

    # Initialize the FunctionApp instance that registers all trigger functions
    app = func.FunctionApp()

    # Define an MCP tool trigger that exposes "summarize_text" to MCP clients.
    # toolProperties defines the input schema: a single "text" string parameter.
    @app.generic_trigger(
        arg_name="context",
        type="mcpToolTrigger",
        toolName="summarize_text",
        description="Summarize a block of text into key points",
        toolProperties='[{"propertyName": "text", "propertyType": "string", "description": "The text to summarize"}]'
    )
    def summarize_text(context: str) -> str:
        # Log the raw payload for debugging
        logging.info(f"summarize_text raw context: {context}")
        # Parse the outer request envelope sent by the MCP client
        request = json.loads(context)
        logging.info(f"summarize_text parsed request: {request}")
        # Extract the tool arguments from the request
        arguments = request.get("arguments", {})
        # Retrieve the "text" property defined in toolProperties
        text = arguments.get("text", "")

        # In a real implementation, call an Azure AI service here
        summary = f"Summary of {len(text.split())} words: {text[:100]}..."

        # Return a JSON response; the "content" field is displayed to the MCP client
        return json.dumps({"content": summary})

    # Define a second MCP tool trigger that exposes "classify_document" to MCP clients.
    # toolProperties defines two input parameters: "text" and "categories".
    @app.generic_trigger(
        arg_name="context",
        type="mcpToolTrigger",
        toolName="classify_document",
        description="Classify a document into a category",
        toolProperties='[{"propertyName": "text", "propertyType": "string", "description": "The document text to classify"}, {"propertyName": "categories", "propertyType": "string", "description": "Comma-separated list of possible categories"}]'
    )
    def classify_document(context: str) -> str:
        # Log the raw payload for debugging
        logging.info(f"classify_document raw context: {context}")
        # Parse the outer request envelope sent by the MCP client
        request = json.loads(context)
        logging.info(f"classify_document parsed request: {request}")
        # Extract the tool arguments from the request
        arguments = request.get("arguments", {})
        # Retrieve the "text" and "categories" properties defined in toolProperties
        text = arguments.get("text", "")
        categories = arguments.get("categories", "general")

        # In a real implementation, call an Azure AI service here
        # Split the comma-separated categories string into a list
        category_list = [c.strip() for c in categories.split(",")]
        # Select the first category as the classification result
        selected_category = category_list[0] if category_list else "unknown"

        # Return a JSON response with the classification result
        return json.dumps({
            "content": f"Classification: {selected_category}",
            "category": selected_category
        })
    ```

1. ファイルを保存し、少し時間を取ってコードをレビューします。 各関数は、type が **mcpToolTrigger** の **@app.generic_trigger()** デコレーターを使用します。 **toolName** は MCP クライアントのツール リストに表示され、**description** は、それぞれのツールをいつ使用すべきかを言語モデルが理解するのに役立ちます。 **toolProperties** パラメーターは、入力スキーマをプロパティ定義の JSON 配列として定義します。

## Python 環境を確認する

このセクションでは、Visual Studio Code が、プロジェクトのセットアップ時に Azure Functions 拡張機能によって作成された仮想環境の Python インタープリターを使用していることを確認します。

1. **Ctrl + Shift + P** キーを押してコマンド パレットを開き、**Python: Select Interpreter** コマンドを実行します。 プロジェクトディレクトリ内の *.venv* フォルダーからインタープリター (例: *./.venv/bin/python*) を選択します。 これにより、**F5** キーで Functions ランタイムを起動したときに、デバッガーとターミナルで確実に正しい環境が使用されます。

## MCP サーバーをローカルでテストする

このセクションでは、ローカルの Functions ランタイムを起動し、エージェント モードで GitHub Copilot から MCP サーバーに接続して、ツールが検出可能であり、期待される結果を返すことを確認します。

1. **F5** キーを押して、デバッガーがアタッチされた状態で Functions ランタイムを起動します。 必要なストレージ アカウントに関する警告が表示された場合は、**[後で確認する]** を選択します。 Visual Studio Code は Core Tools を起動し、デバッガーをアタッチして、関数エンドポイントを表示したターミナル パネルを開きます。

    >**注:** 統合ターミナルで `func start` を実行して、デバッガーを使用せずにランタイムを起動することもできます。

    ターミナル出力は、登録済みの MCP ツール トリガー関数を示します。 次のような結果が出力されることを確認します。

    ```
    Functions:
        classify_document: mcpToolTrigger
        summarize_text: mcpToolTrigger
    ```

    両方の機能が表示されている場合、MCP サーバーは稼働しており、接続の準備ができています。

1. Visual Studio Code は、前に作成した *.vscode/mcp.json* ファイルを検出し、MCP サーバーに接続します。 GitHub Copilot Chat を開き、**[エージェント]** モードに切り替えます。 ツール アイコン (レンチ) を選択し、**document-tools-local** グループを探します。これは、*.vscode/mcp.json* のサーバー キー名と一致します。 そのグループの下に、**summarize_text** と **classify_document** の両方がそれぞれの説明と共に表示されていることを確認します。

### 明示的なプロンプトを使用してテストする

ツールに直接名前を付ける明示的なプロンプトは、最も確実に MCP ツールをトリガーする方法です。 モデルは通常ツールを呼び出しますが、その応答でツールの生の出力を言い換えたり、要約したりすることもあります。 ツールが呼び出されない場合は、ターミナル出力を調べて確認し、プロンプトをもう一度送信してみてください。

1. Copilot Chat に次のプロンプトを入力して、**classify_document** ツールをテストします。 **注:** Copilot が初めて MCP ツールを呼び出すときに、アクセス許可プロンプトが表示される場合があります。 Copilot がツールを呼び出すようにするには、**[許可]** を選択します。

    ```
    Use the classify_document tool to classify this text: 'This agreement is entered into by Party A and Party B'  with categories: contract, invoice, memo
    ```

    Copilot がツールを呼び出して応答を返します。 結果に、**contract** の分類が含まれていることを確認します。 スタブ実装では、コンマ区切りの一覧から最初のカテゴリが選択されるため、結果はプロンプトで指定した最初のカテゴリと一致します。 ターミナル出力で関数呼び出しのログ エントリを確認します。

1. Copilot Chat に次のプロンプトを入力して、**summarize_text** ツールをテストします。

    ```
    Use the summarize_text tool to summarize this text: 'Azure Functions is a serverless compute service that lets you run event-triggered code without having to explicitly provision or manage infrastructure.'
    ```

    結果に、**Summary of** で始まる要約文字列と、その後にワード数、入力テキストの切り詰められたプレビューが含まれていることを確認します。

### 自然言語プロンプトを使用してテストする

自然言語プロンプトは、ツールを呼び出さずにプロンプトに直接応答する可能性が高くなります。これは、ツールが関連するかどうかをモデルが独自に判断する必要があるためです。 ターミナル出力でログ エントリを確認して、ツールが実際に呼び出されたかどうかを確認します。 呼び出されていない場合は、プロンプトを言い換えるか、ツールに明示的に名前を付けてみてください。

1. Copilot Chat に次のプロンプトを入力します。

    ```
    Is the following text an invoice, contract, or memo?
    'Invoice B1234 for services rendered in January 2026. Total amount due: $5,000.'
    ```

    Copilot は、**classify_document** ツールがこの要求と一致することを認識し、このツールを自動的に呼び出します。 結果が分類を返していることを確認します。 ターミナル出力を調べて、関数が呼び出されたことを確認します。

1. 次のプロンプトを入力して、**summarize_text** ツールの自然なツール検出をテストします。

    ```
    Give me a brief summary of this text: 'Machine learning models require large datasets for training. The quality of the training data directly impacts model accuracy. Data preprocessing steps include cleaning, normalization, and feature extraction.'
    ```

    Copilot は、**summarize_text** ツールを呼び出して要約を返します。 ターミナル出力に関数呼び出しが示されていることを確認します。

1. **Shift + F5** キーを押して、デバッガーを停止し、Functions ランタイムをシャットダウンします。

## 次のステップ

運用環境では、Flex 従量課金プランを使用して関数アプリを Azure にデプロイし、**mcp_extension** システム キーを使用して MCP クライアント接続を認証し、各ツール関数内のプレースホルダー ロジックを **DefaultAzureCredential** と関数アプリのマネージド ID を使用した Azure AI サービスへの呼び出しに置き換えます。 詳細については [Azure Functions MCP 拡張機能のドキュメント](/azure/azure-functions/functions-bindings-mcp)を参照してください。

## トラブルシューティング

この演習の実行中に問題が発生した場合は、次のトラブルシューティング手順をお試しください。

**Azure Functions Core Tools が起動しない**
- ターミナルで **func --version** を実行して、Azure Functions Core Tools v4 以降がインストールされていることを確認してください。
- ポート 7071 が別のプロセスで使用されていないことを確認します。
- プロジェクトのルートに *local.settings.json* が存在することを確認します。 Azure Functions 拡張機能は、プロジェクトの作成時にこのファイルを生成します。

**MCP ツールが Copilot に表示されない**
- *.vscode/mcp.json* が保存されており、URL がローカル エンドポイント (`http://localhost:7071/runtime/webhooks/mcp/sse`) と一致していることを確認します。
- Functions ランタイムが実行されており、ターミナル出力に両方のツール トリガー関数が表示されていることを確認します。
- ファイルを保存した後に MCP サーバーの構成が検出されない場合は、Visual Studio Code を再起動してみてください。

**関数呼び出しでエラーが返される**
- ターミナル出力で Python の例外やスタック トレースを確認します。
- *function_app.py* のインデントが正しいことと、すべてのコードが示されたとおりに入力されていることを確認します。
- 仮想環境がアクティブであることと、*requirements.txt* のすべての依存関係がインストールされていることを確認してください。

**Python 環境と依存関係を確認する**
- **Python: Select Interpreter** コマンドを実行して、Visual Studio Code で *.venv* フォルダーの Python インタープリターが使用されていることを確認します。
- 統合ターミナルで **pip list** を実行して、*requirements.txt* のすべてのパッケージが正常にインストールされていることを確認します。
