---
lab:
  topic: Azure Kubernetes Service
  title: AI 推論 API を Azure Kubernetes Service にデプロイする
  description: Kubernetes デプロイとサービス マニフェストを作成して、AI 推論 API コンテナーを Azure Kubernetes Service にデプロイする方法を学びます。
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
---

# AI 推論 API を Azure Kubernetes Service にデプロイする

この演習では、Microsoft Foundry AI モデル、Azure Container Registry (ACR)、Azure Kubernetes Service (AKS) クラスターといった Azure リソースをデプロイします。 その後、Kubernetes マニフェスト ファイルを完成させ、コンテナーの仕様、正常性プローブ、リソース制限、負荷分散を定義します。 コンテナー化された API を AKS にデプロイした後、Python クライアント アプリケーションを使用して、正常性チェック、準備検証、AI モデル推論要求など、デプロイされた API エンドポイントをテストします。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードする
- Azure にリソースをデプロイする
- *deployment.yaml* および *service.yaml* ファイルを完成させ、コンテナーを AKS にデプロイする
- クライアント アプリを実行して API をテストする

この演習の所要時間は約 **30** 分です。

>**重要:** Azure の無料クレジットを使用すると、Azure Container Registry タスクの実行が一時停止されます。 この演習には、従量課金制または別の有料プランが必要です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- 必要な Azure サービスをデプロイするためのアクセス許可がある Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- Kubernetes コマンドライン ツール [kubectl](https://kubernetes.io/docs/tasks/tools/)。
- オプション: [Python 3.12](https://www.python.org/downloads/) 以上。

## プロジェクト スターター ファイルをダウンロードして Azure サービスをデプロイする

このセクションでは、コンソール アプリのスターター ファイルをダウンロードし、スクリプトを使用して必要なサービスを Azure サブスクリプションにデプロイします。 Azure Managed Redis のデプロイが完了するまでに 5 分から 10 分かかります。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/aks-deploy-python.zip
    ```

1. プロジェクトで作業するシステム内の場所にファイルをコピーまたは移動します。 その後、ファイルをフォルダーに解凍します。

1. Visual Studio Code (VS Code) を起動し、メニューで **[ファイル] > [フォルダーを開く...]** を選択してから、プロジェクト ファイルを含むフォルダーを選びます。

1. プロジェクトには Bash (*azdeploy.sh*) と PowerShell (*azdeploy.ps1*) の両方のデプロイ スクリプトが含まれています。 自分の環境に適したファイルを開き、スクリプトの先頭の 2 つの値を自分のニーズに合わせて変更してから、変更を保存します。 **注:** スクリプトの他の部分は変更しないでください。

    ```
    "<your-resource-group-name>" # Resource Group name
    "<your-azure-region>" # Azure region for the resources
    ```

    > **注:** **eastus2**、**swedencentral**、または **australiaeast** の 3 つの Azure リージョンのいずれかをデプロイに使用することをお勧めします。 これらのリージョンでは、演習で使用される AI 推論モデルのデプロイがサポートされています。

1. メニュー バーで、**[ターミナル] > [新しいターミナル]** を選択して、VS Code でターミナル ウィンドウを開きます。

1. 次のコマンドを実行して、Azure アカウントにログインします。 画面の指示に従って、演習用の Azure アカウントとサブスクリプションを選択します。

    ```
    az login
    ```

1. 次のコマンドを実行して、AKS、ACR、Foundry AI モデルをインストールするために必要なリソース プロバイダーがサブスクリプションにあることを確かめます。

    ```
    az provider register --namespace Microsoft.CognitiveServices
    az provider register --namespace Microsoft.ContainerService
    az provider register --namespace Microsoft.ContainerRegistry
    ```

1. プロジェクトのルート ディレクトリにいることを確認し、ターミナルで適切なコマンドを実行してデプロイ スクリプトを起動します。

    **Bash**
    ```bash
    bash azdeploy.sh
    ```

    **PowerShell**
    ```powershell
    ./azdeploy.ps1
    ```

### Azure にリソースをデプロイする

デプロイ スクリプトを実行した状態で、次の手順に従って Azure に必要なリソースを作成します。

1. 「**1**」と入力して、**[1. Provision gpt-5-mini model in Microsoft Foundry]** オプションを起動します。 このオプションでは、リソース グループがまだ存在しない場合は作成され、MIcrosoft Foundry にリソースが作成されて、**gpt-5-mini** モデルがリソースにデプロイされます。

    > **重要:** モデルのデプロイ中にエラーが発生した場合は、「**2**」と入力して **[2. Delete/Purge Foundry deployment]** オプションを起動します。 これによりデプロイが削除され、リソース名が消去されます。 メニューを終了し、デプロイ スクリプトのリージョンを他の推奨リージョンのいずれかに変更します。 その後、デプロイ スクリプトを再起動し、モデル プロビジョニング オプションをもう一度実行します。

1. モデルがデプロイされた後、「**3**」と入力して、**[3. Create Azure Container Registry (ACR)]** を起動します。 これにより、API コンテナーが格納されるリソースが作成され、後で AKS リソースにプルされます。

1. ACR リソースが作成された後、「**4**」と入力して **[Build and push API image to ACR]** を起動します。 このオプションでは、ACR タスクを使用してイメージをビルドし、ACR リポジトリに追加します。 この操作は、完了するまで 3 分から 5 分かかることがあります。

1. イメージがビルドされ、ACR にプッシュされた後、「**5**」と入力して、**[5. Create AKS cluster]** オプションを起動します。 これにより、マネージド ID で構成された AKS リソースが作成され、ACR リソースからイメージをプルするアクセス許可がサービスに付与されます。 この操作は、完了するまで 5 分から 10 分かかることがあります。

1. AKS リソースがデプロイされた後、「**6**」と入力して **[6. Check deployment status]** オプションを起動します。 このオプションでは、3 つの各リソースが正常にデプロイされたかどうかが報告されます。

    すべてのサービスで**成功**メッセージが返された場合は、「**8**」と入力してデプロイ スクリプトを終了します。

次に、API を AKS にデプロイするために必要な YAML ファイルを完成させます。

## YAML デプロイ ファイルを完成させ、AKS にデプロイする

このセクションでは、*deployment.yaml* と *service.yaml* の両方のファイルを完成させます。 デプロイ マニフェストでは、API コンテナーを AKS でデプロイおよび管理する方法を定義しますが、サービス マニフェストでは、ロード バランサーを介して外部トラフィックに API を公開します。

1. *k8s/deployment.yaml* ファイルを開き、ファイルを完成させる作業を始めます。

1. **# BEGIN: Container specification** というコメントを見つけ、そのコメントの下のマニフェストに次の YAML セクションを追加します。 YAML インデントが正しいことを確かめてください。

    ```yml
    containers:  # List of containers to run in the pod
    - name: api
      image: ACR_ENDPOINT/aks-api:latest  # Container image from ACR
      imagePullPolicy: Always  # Always pull the latest image from registry
      ports:  # Ports exposed by the container
      - name: http
        containerPort: 8000
        protocol: TCP
    ```

    このセクションでは、ACR から使用するコンテナー イメージ、プル ポリシー、HTTP トラフィック用にコンテナーで公開されるポートなど、コンテナーの仕様を定義します。

1. **# BEGIN: Liveness Probe Configuration** というコメントを見つけ、そのコメントの下のマニフェストに次の YAML セクションを追加します。 YAML インデントが正しいことを確かめてください。

    ```yml
    livenessProbe:  # Detects if container is alive or needs restart
      httpGet:
        path: /healthz  # Health check endpoint path
        port: http
      initialDelaySeconds: 10  # Seconds to wait before first check
      periodSeconds: 30
      timeoutSeconds: 5
      failureThreshold: 3  # Consecutive failures before restarting container
    ```

    このセクションでは、liveness probe を構成し、**/healthz** エンドポイントに HTTP 要求を行うことで、コンテナーが正常かどうかを定期的に確認します。 probe が 3 回連続して失敗した場合、Kubernetes によってコンテナーが自動的に再起動されます。

1. **# BEGIN: Resource Limits Configuration** というコメントを見つけ、そのコメントの下のマニフェストに次の YAML セクションを追加します。 YAML インデントが正しいことを確かめてください。

    ```yml
    resources:  # CPU and memory resource specifications
      requests:  # Minimum resources guaranteed to the container
        memory: "256Mi"
        cpu: "250m"
      limits:  # Maximum resources the container can use
        memory: "512Mi"
        cpu: "500m"
    ```

    このセクションでは、コンテナーの CPU およびメモリ リソースを定義します。 要求では保証される最小リソースを指定しますが、制限ではコンテナーで消費できる最大リソースを設定します。 これにより、Kubernetes でポッドを効率的にスケジュールし、リソース不足を防ぐことができます。

1. 変更を保存し、少し時間を取って、完成した *deployment.yaml* ファイルをレビューします。

次に、*service.yaml* ファイルを更新します。

1. ファイルを完成させるために *k8s/service.yaml* を開きます。

1. 以下の YAML をマニフェストに追加します。 YAML インデントが正しいことを確かめてください。

    ```yml
    apiVersion: v1
    kind: Service  # Service: exposes pods on a network and provides load balancing
    metadata:
      name: aks-api-service  # Unique name for the service
      labels:
        app: aks-api # Matches deployment and pod labels
      annotations:
        service.beta.kubernetes.io/azure-load-balancer-internal: "false"  # Use public load balancer
    spec:  # Service specification
      type: LoadBalancer  # Exposes service externally
      selector:  # Selects which pods to route traffic to based on labels
        app: aks-api
        version: v1
      ports:  # Port mappings between service and pods
      - name: http
        port: 80  # Service port exposed externally
        targetPort: http  # Pod container port to forward traffic to
        protocol: TCP
      sessionAffinity: None  # Client requests not pinned to specific pods
    ```

    このマニフェストでは、Azure Load Balancer を介して API ポッドを外部に公開する LoadBalancer Service を作成します。 ラベル セレクターを使用してトラフィックを受信するポッドを特定して、ポート 80 の着信トラフィックをコンテナーのポート 8000 にルーティングします。

1. 変更を保存し、少し時間を取ってファイルをレビューします。

### マニフェストを AKS に適用する

このセクションでは、デプロイ スクリプトを使用してマニフェストを AKS に適用します。

1. プロジェクトのルート ディレクトリにいることを確認し、ターミナルで適切なコマンドを実行してデプロイ スクリプトを起動します。

    **Bash**
    ```bash
    bash azdeploy.sh
    ```

    **PowerShell**
    ```powershell
    ./azdeploy.ps1
    ```

1. 「**7**」と入力し、**[7. Deploy to AKS]** オプションを起動します。 このオプションではいくつかの操作を実行します。AKS 資格情報を取得して kubectl を構成し、**Cognitive Services OpenAI ユーザー** ロールを AKS kubelet マネージド ID に割り当てて API で Microsoft Entra ID を使用して Foundry に対して認証できるようにします。また、ACR エンドポイントと Foundry エンドポイントを使用してデプロイ マニフェストを更新し、**kubectl apply** を使って両方のマニフェストを AKS クラスターにデプロイします。 操作が完了したら、「**8**」と入力してデプロイ スクリプトを終了します。

1. ターミナルで以下のコマンドを実行して、デプロイを確認します。 **kubectl get deploy,svc** では、Deployment **READY** が **1/1** (またはレプリカ数) として表示され、Service **EXTERNAL-IP** はパブリック IP (**\<pending>** ではない) を持っていることを想定します。 ロールアウト コマンドでは、更新の完了時に **deployment "aks-api" successfully rolled out** と出力されるはずです。

    ```
    kubectl get deploy,svc
    kubectl rollout status deploy/aks-api
    ```

## クライアント アプリの実行

このセクションでは、Python 環境を構成し、クライアント アプリを使用して API に対して操作を実行します

### Python 環境を構成する

このセクションでは、Python 環境を作成し、依存関係をインストールします。

1. ターミナルのプロジェクトの *client* フォルダーにいることを確かめてください。

1. VS Code ターミナルで次のコマンドを実行して、Python 環境を作成します。

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

1. VS Code ターミナルで次のコマンドを実行して、依存関係をインストールします。

    ```
    pip install -r requirements.txt
    ```

### アプリで操作を実行する

次は、クライアント アプリケーションを実行し、API に対してさまざまな操作を実行します。 アプリには、メニュー駆動型インターフェイスが用意されています。

1. ターミナルで次のコマンドを実行して、コンソール アプリを起動します。 コマンドを実行する前に、演習の前半のコマンドを参照して、必要に応じて環境をアクティブ化してください。

    ```
    python main.py
    ```

1. 「**1**」と入力して、**[1. Check API Health (Liveness)]** オプションを起動します。 これにより、API コンテナーが実行中であることが確認され、正常性チェックに応答します。これは、Kubernetes liveness probe で使用されるエンドポイントと同じです。

1. 「**2**」と入力して、**[2. Check API Readiness (Foundry Connectivity)]** オプションを起動します。 これにより、API が Foundry モデル エンドポイントに正常に接続でき、推論要求を処理する準備ができていることが確認されます。

1. 「**3**」と入力して、**[3. Send Inference Request]** オプションを起動します。 これにより、API に単一のプロンプトが送信され、デプロイされたモデルから完全な応答が受信されます。 単一の推論要求は、バッチ処理、自動化されたタスク、またはさらなる処理のために一度に応答全体が必要な場合に役立ちます。

1. 「**4**」と入力して、**[4. Start Chat Session (Streaming)]** オプションを起動します。 これにより、モデルからの応答が生成されるとリアルタイムでストリーミングされる対話型チャット セッションが開始されます。

終了したら、「**5**」と入力してアプリを終了します。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループとグループ内のすべてのリソースを削除します。 **\<rg-name>** は、演習で前に選択した名前に置き換えます。 このコマンドにより、Azure でバックグラウンド タスクが起動され、リソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の実行中に問題が発生した場合は、次のトラブルシューティング手順をお試しください。

**Azure リソースのデプロイを確認する**
- [Azure portal](https://portal.azure.com) に移動して、リソース グループを見つけます。
- Microsoft Foundry リソースに、**Succeeded** の **Provisioning State** が表示され、**gpt-5-mini** モデルがデプロイされていることを確認します。
- Azure Container Registry (ACR) が存在し、**aks-api** イメージが含まれていることを確認します。
- AKS クラスターが **Succeeded** 状態であり、ノードが実行されていることを確認します。

**AKS のデプロイ状態を確認する**
- **kubectl get pods** を実行して、API ポッドが実行されているかどうかを確認します。 **Running** 状態を探します。
- **kubectl get svc** を実行して、LoadBalancer サービスに外部 IP が割り当てられていることを確認します (**\<pending>** ではない)。
- **kubectl describe pod\<pod-name>** を実行して、問題が発生した場合のポッドの状態とイベントの詳細を確認します。
- **kubectl logs\<pod-name>** でポッド ログを確認し、コンテナーのスタートアップ エラーまたはランタイムの問題を確認します。

**YAML ファイルの完全性を確認する**
- すべての YAML セクションが、適切なコメント マーカーの間に正しく *deployment.yaml* および *service.yaml* に追加されていることを確かめます。
- YAML のインデントが正しいことを確認します (タブではなくスペースを使用します)。インデントが正しくないと、デプロイ エラーが発生します。
- デプロイ スクリプトによってデプロイ マニフェストで ACR エンドポイントが適切に置き換えられたことを確認します。

**クライアント構成を確認する**
- *.env* ファイルが *client* フォルダーに存在し、有効な **API_ENDPOINT** 値が含まれていることを確認します。
- API エンドポイントで LoadBalancer サービスの正しい外部 IP が使用されていることを確かめます。
- ターミナルから **curl http://\<external-ip>/healthz** を実行して、API エンドポイントに到達できることを確認します。

**Python 環境と依存関係を確認する**
- クライアント アプリを実行する前に、仮想環境がアクティブ化されていることを確認します。
- **pip list** を実行して、*requirements.txt* のすべてのパッケージが正常にインストールされたことを確認します。
- *client* ディレクトリからクライアントを実行していることを確かめてください。

