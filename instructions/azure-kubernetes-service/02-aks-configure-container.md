---
lab:
  topic: Azure Kubernetes Service
  title: Azure Kubernetes Service 上のアプリを構成する
  description: '永続ストレージを使用して Kubernetes デプロイを構成し、機密設定と非機密設定を格納する方法について学習します。 '
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
---

# Azure Kubernetes Service 上のアプリを構成する

この演習では、機密性の高い設定用の ConfigMaps、機密性の高い資格情報用のシークレット、永続ストレージ用の PersistentVolumeClaims を使用して Kubernetes デプロイを構成する方法について説明します。 コンテナー化された API を Azure Kubernetes Service (AKS) にデプロイし、さまざまな Kubernetes リソースで構成し、Python クライアント アプリケーションを使用して操作します。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードする
- Azure にリソースをデプロイする (ACR、AKS クラスター)
- コンテナー イメージをビルドして Azure Container Registry にプッシュする
- AKS クラスター アクセス用に kubectl 資格情報を構成する
- 更新された YAML ファイルを AKS に適用してポッドを作成し、LoadBalancer を使用して API を公開する
- クライアント アプリを実行して API エンドポイントをテストする
- 永続ボリュームに格納されている API ログを表示する
- Azure リソースをクリーンアップする

この演習の所要時間は約 **30** 分です。

>**重要:** この演習の永続ストレージの実装は、デモンストレーションのみを目的としています。 ログ記録の場合、実稼働アプリケーションでは、永続的なボリュームにログを格納するのではなく、Azure Monitor や Application Insights などの一元的なログ記録ソリューションを使用する必要があります。 永続ストレージが必要な場合は、ログ ローテーション ポリシーを実装してストレージがいっぱいにならないようにします。これにより、コンテナーの障害やポッドの削除が発生する可能性があります。

>**重要:** Azure の無料クレジットを使用すると、Azure Container Registry タスクの実行が一時停止されます。 この演習には、従量課金制または別の有料プランが必要です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- 必要な Azure サービスをデプロイする権限を持つ Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- Kubernetes コマンドライン ツール [kubectl](https://kubernetes.io/docs/tasks/tools/)。
- オプション: [Python 3.12](https://www.python.org/downloads/) 以上。

## プロジェクト スターター ファイルをダウンロードして Azure サービスをデプロイする

このセクションでは、コンソール アプリのスターター ファイルをダウンロードし、スクリプトを使用して必要なサービスを Azure サブスクリプションにデプロイします。 Azure リソースのデプロイが完了するまで 10 分から 15 分かかる場合があります。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/aks-configure-python.zip
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

1. 次のコマンドを実行して、AKS と ACR をインストールするために必要なリソース プロバイダーがサブスクリプションにあることを確かめます。 **Microsoft.Compute**、**Microsoft.Network**、**Microsoft.Storage** プロバイダーは、Azure が通常自動的に登録する AKS の依存関係ですが、明示的に登録することで、新しいサブスクリプションでクラスターの作成エラーが発生する場合に回避できます。

    ```
    az provider register --namespace Microsoft.ContainerService
    az provider register --namespace Microsoft.ContainerRegistry
    az provider register --namespace Microsoft.Compute
    az provider register --namespace Microsoft.Network
    az provider register --namespace Microsoft.Storage
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

    > **注:** PowerShell がデジタル署名されていないためにスクリプトをブロックした場合は、同じターミナル セッション内で次のコマンドを実行し、再度配置スクリプトを実行してください。 このコマンドは、現在の PowerShell プロセスの実行ポリシーのみを変更します。

    ```powershell
    Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
    ```

### Azure にリソースをデプロイする

デプロイ スクリプトを実行した状態で、次の手順に従って Azure に必要なリソースを作成します。

1. 「**1**」と入力して、**[Azure Container Registry (ACR) の作成]** を起動します。 これにより、API コンテナーが格納されるリソースが作成され、後で AKS リソースにプルされます。

    操作が完了すると、ACR エンドポイントが返されます。 情報をコピーします。この演習の後半で必要になります。

1. ACR リソースが作成された後、「**2**」と入力して **[Build and push API image to ACR]** を起動します。 このオプションでは、ACR タスクを使用してイメージをビルドし、ACR リポジトリに追加します。 この操作は、完了するまで 3 分から 5 分かかることがあります。

1. イメージがビルドされ、ACR にプッシュされた後、「**3**」と入力して、**[Create AKS cluster]** オプションを起動します。 これにより、マネージド ID で構成された AKS リソースが作成され、ACR リソースからイメージをプルするためのアクセス許可がサービスに付与され、永続ストレージに書き込むのに必要な RBAC ロールが割り当てられます。 この操作は、完了するまで 5 分から 10 分かかることがあります。

1. AKS クラスターのデプロイが完了した後、「**4**」と入力して、**[Get AKS credentials for kubectl]** オプションを起動します。 これにより、**az aks get-credentials** コマンドを使用して資格情報が取得され、**kubectl** が構成されます。

1. 認証情報が構成された後、「**5**」と入力して **[デプロイの状態の確認]** オプションを起動します。 このオプションでは、各リソースが正常にデプロイされたかどうかが報告されます。

    すべてのサービスで**成功**メッセージが返された場合は、「**7**」と入力してデプロイ スクリプトを終了します。

    AKS クラスターが **[失敗]** または **[キャンセル済み]** の状態にある場合は、報告された問題を修正し、「**6**」と入力して **[失敗した AKS デプロイの削除]** オプションを起動し、オプション **3** を再度実行します。 この保護されたオプションでは、正常な、または進行中のクラスターは削除されません。

次に、API を AKS にデプロイするために必要な YAML ファイルを完成させます。

## YAML デプロイ ファイルを完成させ、AKS にデプロイする

このセクションでは、*k8s* フォルダーにある YAML ファイルを完成させます。これは、永続ストレージを使用して Kubernetes デプロイを構成し、機密設定と非機密設定を格納するために必要です。

### ConfigMap YAML ファイルを完成させる

ConfigMaps では、ポッドで消費できるキーと値のペアとして非機密構成データを格納します。 このセクションでは、学生名、API バージョン、ログ パスなどのアプリケーション設定を格納する ConfigMap を作成します。

1. *k8s/configmap.yaml* ファイルを開き、次のコードをファイルに追加します。 必要に応じて、**STUDENT_NAME** の値を自分の名前で更新できます。

    ```yml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: api-config
      labels:
        app: aks-config-api # Label for the AKS configuration API
    data:
      # Store non-sensitive configuration values
      STUDENT_NAME: "YourNameHere"
      API_VERSION: "1.0.0"
      LOG_PATH: "/var/log/api" # Path for API logs
    ```

1. コード内のコメントをレビューしてから、変更を保存します。

### Secrets YAML ファイルを完成させる

Secrets ではパスワード、トークン、キーなどの機密情報を base64 エンコード形式で格納します。 このセクションでは、実行時に API がアクセスする機密資格情報を格納する Secret を作成します。

1. *k8s/secrets.yaml* ファイルを開き、次のコードをファイルに追加します。

    ```yml
    apiVersion: v1
    kind: Secret
    metadata:
      name: api-secrets
      labels:
        app: aks-config-api
    type: Opaque
    stringData:
      # Store sensitive credentials as base64-encoded values
      secret-endpoint: "SecretEndpointValue"
      secret-access-key: "SecretAccessKey123456"
    ```

1. 少し時間を取ってコード内のコメントをレビューしてから、変更を保存します。

### PVC YAML ファイルを完成させる

PersistentVolumeClaim (PVC) では、ポッドにマウントできるストレージ リソースを Azure から要求します。 このセクションでは、Azure Disk Storage を使用してポッドの再起動間で API ログ ファイルを保持する PVC を作成します。

1. *k8s/pvc.yaml* ファイルを開き、次のコードをファイルに追加します。

    ```yml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: api-logs-pvc
      labels:
        app: aks-config-api # Label for the AKS configuration API
    spec:
      accessModes:
        - ReadWriteOnce  # Allow single pod to mount volume for read/write
      resources:
        requests:
          storage: 1Gi  # Request minimum Azure Disk size
      storageClassName: managed-csi  # Use Azure Disk CSI driver (default)
      volumeMode: Filesystem  # Default mode
    ```

1. 少し時間を取ってコード内のコメントをレビューしてから、変更を保存します。

### Deployment YAML ファイルを更新する

デプロイ マニフェストは、環境変数、ボリューム マウント、プローブを使用して既に部分的に構成されています。 コンテナー イメージの参照を特定の ACR エンドポイントで更新するだけで済みます。

1. *k8s/deployment.yaml* ファイルを開き、**image: \<YOUR_ACR_ENDPOINT>/aks-config-api:latest** 行を見つけます。

1. **\<YOUR_ACR_ENDPOINT>** は、演習で前に記録した値に置き換えます。

1. 少し時間を取ってコード内のコメントをレビューしてから、変更を保存します。


## マニフェストを AKS に適用する

このセクションでは、マニフェストを AKS に適用します。 VS Code ターミナルで次の手順を実行します。 コマンドを実行する前に、プロジェクトのルートにいることを確かめます。

1. 次のコマンドを実行して、ConfigMap を適用します。

    ```
    kubectl apply -f k8s/configmap.yaml
    ```

1. 次のコマンドを実行して、Secrets を適用します。

    ```
    kubectl apply -f k8s/secrets.yaml
    ```

1. 次のコマンドを実行して、PersistentVolumeClaim を適用します。

    ```
    kubectl apply -f k8s/pvc.yaml
    ```

1. 次のコマンドを実行して、Deployment を適用します。

    ```
    kubectl apply -f k8s/deployment.yaml
    ```

1. 次のコマンドを実行して、Service を作成します。

    ```
    kubectl apply -f k8s/service.yaml
    ```

1. Service を作成した後、デプロイが完了するまで数分かかる場合があります。 次のコマンドでは、サービスを監視し、ポッドが使用可能になったときにポッドの外部 IP アドレスを更新します。 外部 IP アドレスはメモしておいてください。この演習の後半で必要になります。 IP アドレスが表示された後、**ctrl + c** キーを押してコマンドを終了します。

    ```
    kubectl get svc aks-config-api-service -w
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

1. クライアント ディレクトリに *.env* ファイルを作成し、次のコードを追加します。 **\<API_IP_address>** は、演習で前にコピーした値に置き換えます。

    ```
    # API endpoint - update this with the external IP from the LoadBalancer service
    # Get the IP with: kubectl get services
    API_ENDPOINT=http://<API_IP_address>
    ```
### アプリで操作を実行する

Python 環境が構成され、依存関係がインストールされたので、クライアント アプリケーションを実行して、デプロイされた API をテストできるようになりました。 API ではすべての操作が永続ボリュームに記録され、クライアントはさまざまなエンドポイントと対話するためのメニュー駆動型インターフェイスを提供します。

1. ターミナルで次のコマンドを実行して、コンソール アプリを起動します。 コマンドを実行する前に、演習の前半のコマンドを参照して、必要に応じて環境をアクティブ化してください。

    ```
    python main.py
    ```

1. 「**1**」と入力して、**[Check API Health (Liveness)]** オプションを起動します。 これにより、API コンテナーが実行中であることが確認され、正常性チェックに応答します。これは、Kubernetes liveness probe で使用されるエンドポイントと同じです。 返される情報には、ConfigMap に設定された非機密の学生名が含まれていることに注意してください。

    ```
    [*] Checking API health...
    ✓ API is healthy
      Service: aks-config-api
      Version: 1.0.0
      Student: YourNameHere
    ```
1. 「**2**」と入力して、**[Check API Readiness (Foundry Connectivity)]** オプションを起動します。 これにより、API が Foundry モデル エンドポイントに正常に接続でき、推論要求を処理する準備ができていることが確認されます。

1. 「**3**」と入力して、**[View Secrets Information]** オプションを起動します。 この機能は、シークレットがポッドで設定されたことを確認するためにのみ存在し、デモンストレーションのみを目的としています。 出力で、シークレットに関する情報を表示できます。この出力はマスクされます。

    ```
    Secret Details:

      secret_endpoint:
        Loaded: True
        Value: SecretEndp...
        Length: 19 characters

      secret_access_key:
        Loaded: True
        Value: ***3456
        Length: 21 characters
    ```

1. 「**5**」と入力して、**[List All Products]** オプションを起動します。 これにより、API に含まれているモック データが表示されます。

1. API で複数の異なるエンドポイントの操作がログに記録されたので、次はログを表示します。 「**6**」と入力して、**[View Log Summary]** オプションを起動し、さまざまな操作の概要を表示します。 要求の合計数と、**/readyz** および **/healthz** エンドポイントへの要求に注意してください。 これら 2 つの操作は、*deployment.yaml* ファイルで設定されたスケジュールに基づいて自動的に実行されます。

    ```
    ✓ Log summary retrieved

    Log file: /var/log/api/api-requests-2025-12-21.log
    Total requests: 220
    Student: YourNameHere

    First request: 2025-12-21T02:19:48.953308
    Last request: 2025-12-21T02:46:08.952772

    Requests by endpoint:
      /readyz: 160
      /healthz: 56
      /secrets: 1
      /products: 1
    ```

1. ログ情報の生成を続け、終了したら「**7**」と入力してアプリを終了できます。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループと、そのグループ内のすべてのリソースを削除します。 **\<rg-name>** は、この演習で選択した名前に置き換えてください。 このコマンドを実行すると Azure の中でバックグラウンド タスクが起動されてリソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の実行中に問題が発生した場合は、次のトラブルシューティング手順をお試しください。

**デプロイ スクリプトでデプロイ状態を確認する**
- デプロイ スクリプトを実行し、オプションの **[5. Check deployment status]** を選択して、デプロイされたすべてのリソースの状態を確認します。
- このコマンドでは以下を確認します。
  - ACR プロビジョニングの状態と準備
  - AKS クラスターのプロビジョニング状態
  - Kubernetes リソース (ConfigMap、Secrets、PVC、Deployment の可用性、Service LoadBalancer IP)
- この出力を使用して、問題の原因となっている可能性のあるコンポーネントを特定します。

**AKS クラスターの作成エラーを解決する**
- クォータ検証は AKS リソースが作成される前に失敗することがあり、プロビジョニングでの失敗では、クラスターが **[失敗]** または **[キャンセル済み]** 状態のままになることがあります。
- エラーで **[Standard_D2s_v5]** が利用できないか、リージョンの容量が不足していると報告された場合は、オプション **7** で終了し、デプロイ スクリプトの上部近くにある **[場所]** を変更し、オプション **[3 AKS クラスターの作成]** を再度実行します。
- エラーでクォータ不足と報告された場合は、サブスクリプションに利用可能な [Dsv5-family] クォータがあるリージョンを選択するか、クォータの増量を申請してください。 リージョンの変更が有効なのは、他のリージョンに十分なクォータがある場合に限られます。
- AKS クラスターは、リソース グループが既に別のリージョンに存在しても、スクリプトで構成された**場所**を使用します。
- オプション **5** で **[失敗]** または **[キャンセル済み]** が報告された場合は、根本的な問題を修正し、オプション **[6 失敗した AKS デプロイの削除]** を実行してからオプション **3** を再試行してください。 AKS リソースが作成されていない場合、削除は不要です。

**YAML ファイルの完全性を確認する**
- すべての YAML コンテンツが *configmap.yaml*、*secrets.yaml*、*pvc.yaml* に正しく追加され、そのインデントが正しいことを確かめます。
- *deployment.yaml* で ACR エンドポイントが正しく更新されたことを確認します (**\<YOUR_ACR_ENDPOINT>** を実際の ACR エンドポイントに置き換えます)。
- 初期デプロイ後に YAML ファイルに変更を加える場合は、**kubectl apply -f k8s/\<filename>.yaml** を使用してファイルを再適用します。
- ConfigMap または Secret ファイルを更新した後、ローリング再起動を実行して構成を再度読み込みます: **kubectl rollout restart deployment aks-config-api**。

**クライアント構成を確認する**
- *client* フォルダーに *.env* ファイルを作成し、**API_ENDPOINT** を LoadBalancer の外部 IP (例: **http://20.xxx.xxx.xxx**) に設定していることを確かめます。
- ターミナルから **curl http://\<external-ip>/healthz** を実行して、API エンドポイントに到達できることを確認します。

**Python 環境と依存関係を確認する**
- クライアント アプリを実行する前に、仮想環境がアクティブ化されていることを確認します。
- **pip list** を実行して、*requirements.txt* のすべてのパッケージが正常にインストールされたことを確認します。
- *client* ディレクトリからクライアントを実行していることを確かめてください。
