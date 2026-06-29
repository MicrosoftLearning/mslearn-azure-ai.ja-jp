---
lab:
  topic: Azure Kubernetes Service
  title: Azure Kubernetes Service でアプリのトラブルシューティングを行う
  description: ラベルの不一致、CrashLoopBackOff エラー、readiness probe の失敗など、Kubernetes の一般的な問題を診断して解決する方法を学びます。
  level: 300
  duration: 30
  islab: true
  primarytopics:
    - Azure
---

# Azure Kubernetes Service でアプリのトラブルシューティングを行う

この演習では、コンテナー化された API を Azure Kubernetes Service (AKS) にデプロイし、Kubernetes の一般的な問題を診断して解決します。 **kubectl** コマンド使用して、問題の特定、ポッドの状態の検査、ログの確認、イベントの表示を行います。 その後、**kubectl edit** を使用して、サービス セレクターの不一致、環境変数の欠落、無効な readiness probe パスなどの構成ミスを修正します。

この演習で実行されるタスク:

- プロジェクト スターター ファイルをダウンロードする
- Azure にリソースをデプロイする (ACR、AKS クラスター)
- いくつかの一般的な問題を診断して解決する
- Azure リソースをクリーンアップする

この演習の所要時間は約 **30** 分です。

>**重要:** Azure の無料クレジットを使用すると、Azure Container Registry タスクの実行が一時停止されます。 この演習には、従量課金制または別の有料プランが必要です。

## 開始する前に

演習を最後まで行うには、次のものが必要です。

- 必要な Azure サービスをデプロイする権限を持つ Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- [サポートされているプラットフォーム](https://code.visualstudio.com/docs/supporting/requirements#_platforms)のいずれかにインストールされた [Visual Studio Code](https://code.visualstudio.com/)。
- 最新バージョンの [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)。
- Kubernetes コマンドライン ツール [kubectl](https://kubernetes.io/docs/tasks/tools/)。
- オプション: [Python 3.12](https://www.python.org/downloads/) 以上。

## プロジェクト スターター ファイルをダウンロードして Azure サービスをデプロイする

このセクションでは、コンソール アプリのスターター ファイルをダウンロードし、スクリプトを使用して必要なサービスを Azure サブスクリプションにデプロイします。 Azure Managed Redis のデプロイが完了するまでに 5 分から 10 分かかります。

1. ブラウザーを開き、次の URL を入力してスターター ファイルをダウンロードします。 ファイルはユーザーの既定のダウンロード場所に保存されます。

    ```
    https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/aks-troubleshoot-python.zip
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

### Azure にリソースをデプロイする

デプロイ スクリプトを実行した状態で、次の手順に従って Azure に必要なリソースを作成します。

1. モデルがデプロイされた後、「**1**」と入力して、**[Create Azure Container Registry (ACR)]** を起動します。 これにより、API コンテナーが格納されるリソースが作成され、後で AKS リソースにプルされます。

1. ACR リソースが作成された後、「**2**」と入力して **[Build and push API image to ACR]** を起動します。 このオプションでは、ACR タスクを使用してイメージをビルドし、ACR リポジトリに追加します。 この操作は、完了するまで 3 分から 5 分かかることがあります。

1. イメージがビルドされ、ACR にプッシュされた後、「**3**」と入力して、**[Create AKS cluster]** オプションを起動します。 これにより、マネージド ID で構成された AKS リソースが作成され、ACR リソースからイメージをプルするアクセス許可がサービスに付与されます。

1. AKS クラスターのデプロイが完了した後、「**4**」と入力して、**[Get AKS credentials for kubectl]** オプションを起動します。 これにより、**az aks get-credentials** コマンドを使用して資格情報が取得され、**kubectl** が構成されます。

1. 資格情報が設定された後、「**5**」と入力して **[Deploy applications to AKS]** オプションを起動します。 これにより、API が AKS クラスターにデプロイされます。

1. アプリがデプロイされた後、「**6**」と入力して **[Check deployment stats]** オプションを起動します。 このオプションでは、各リソースが正常にデプロイされたかどうかが報告されます。

    すべてのサービスで**成功**メッセージが返された場合は、「**7**」と入力してデプロイ スクリプトを終了します。

>**注:** ターミナルを開いたままにしておくと、演習のすべての手順がターミナルで実行されます。

## デプロイのトラブルシューティング

デプロイ スクリプトでは、**aks-troubleshoot** という**名前空間**に Kubernetes リソースをすべて作成しました。 名前空間は、Kubernetes クラスター内のリソースを整理および分離する方法です。 関連するリソースをグループ化したり、リソース クォータを適用したり、アクセス制御を管理したりできます。 名前空間を指定しない場合、リソースは**既定の**名前空間で作成されます。 この演習では、すべての **kubectl** コマンドに **-n aks-troubleshoot** が含まれており、正しい名前空間をターゲットにします。

デプロイを確認した後、3 つのトラブルシューティング シナリオを実行します。 各シナリオでは、特定のエラーを発生させるマニフェスト ファイルをデプロイに適用します。 その後、**kubectl** コマンドを使用して問題を診断し、構成を編集して解決します。

### デプロイを検証する

このセクションでは、エラーが発生する前に、セットアップ スクリプトによってデプロイされたアプリケーションが正しく実行されていることを確認します。

1. 次のコマンドを実行して、ポッドが名前空間で実行されていることを確認します。 このコマンドでは、**Running** 状態で READY 列に **1/1** と示された 1 つのポッドが返されるはずです。

    ```
    kubectl get pods -n aks-troubleshoot
    ```

1. 次のコマンドを実行して、Service にエンドポイントがあることを確認します。 このコマンドでは、IP アドレスと共に一覧表示された 1 つのエンドポイント スライスが返されるはずです。

    ```
    kubectl get endpointslices -l kubernetes.io/service-name=api-service -n aks-troubleshoot
    ```

1. 次のコマンドを実行し、port-forward を使用して接続をテストします。 このコマンドでは、ローカル コンピューターからクラスターで実行されている Service へのトンネルが作成され、**http://localhost:8080** でアクセスできるようにします。

    ```
    kubectl port-forward service/api-service 8080:80 -n aks-troubleshoot
    ```

1. メニューバーで **[ターミナル] > [新しいターミナル]** を選択して、VS Code で 2 つ目のターミナル ウィンドウを開きます。 次のコマンドを実行して接続をテストします。 **"status": "healthy"** の JSON 応答を受け取るはずです。

    ```bash
    # Bash
    curl http://localhost:8080/healthz
    ```

    ```powershell
    # PowerShell
    Invoke-RestMethod http://localhost:8080/healthz
    ```

1. **port-forward** が実行されているターミナルに戻り、**ctrl + c** キーを押してコマンドを終了します。

デプロイが正しく動作していることを確認しました。次はラベルの不一致の問題を診断します。

### ラベルの不一致を診断する

Service では、ラベル セレクターに基づいてトラフィックをポッドにルーティングします。 ラベルが一致しない場合、Service にはエンドポイントがなく、要求は失敗します。 API は、**app: api** というラベルが付いたポッドと共にデプロイされました。Service セレクターは **app: api** と一致しています。 このセクションでは、セレクターを **app: api-v2** に変更する Service 構成を適用し、接続を切断します。

1. 次のコマンドを実行して、ラベルの不一致エラーが発生する Service 構成を適用します。

    ```
    kubectl apply -f k8s/label-mismatch-service.yaml -n aks-troubleshoot
    ```

1. 次のコマンドを実行して、ポッドがまだ実行されていることを確認します。 ポッドには、状態が **Running** で、準備完了状態が **1/1** であること、および **app=api** を示すラベルが表示されます。

    ```
    kubectl get pods --show-labels -n aks-troubleshoot
    ```

1. 以下のコマンドを実行して、Service エンドポイントのスライスを確認します。 このコマンドでは、ENDPOINTS 列に **\<unset>** のエンドポイント スライスが返されるはずです。これは、Service セレクターと一致するポッドがないことを示します。

    ```
    kubectl get endpointslices -l kubernetes.io/service-name=api-service -n aks-troubleshoot
    ```

1. 次のコマンドを実行して、Service の詳細を表示します。 出力で **Selector** フィールドを探します。現在、**app=api-v2** と表示されています。

    ```
    kubectl describe service api-service -n aks-troubleshoot
    ```

    これによりラベルの不一致が確認されました。 Service セレクターは **app=api-v2** ですが、ポッド ラベルは **app=api** です。

1. 以下のコマンドを実行して、エディターで Service 構成を開きます。

    ```
    kubectl edit service api-service -n aks-troubleshoot
    ```

    **注:** **kubectl edit** コマンドでは、クラスターからライブ リソース構成をフェッチし、ローカルのテキスト エディターで開きます。 エディターを保存して閉じると、kubectl により自動的に Kubernetes API サーバーに変更が送り返され、それらが検証されて実行中のクラスターに適用されます。 エディターは環境によって異なります。

    - **Bash:** 既定で **vi** を開きます。 **i** キーを押して挿入モードに入り、変更を行い、**Esc** キーを押してから「**:wq**」と入力し、**Enter** キーを押して保存して終了します。 「**:q!**」と入力し、 保存せずに終了します。
    - **PowerShell (Windows):** 既定で**メモ帳**を開きます。 変更を行い、**[ファイル] > [保存]** を選択し (または **Ctrl + S**)、ウィンドウを閉じます。 保存せずに閉じると、編集が取り消されます。

1. エディターで **selector** セクションを見つけて、**app: api-v2** を **app: api** に変更します。 変更を保存してエディターを閉じます。

1. 以下のコマンドを実行して、エンドポイント スライス アドレスが復元されていることを確認します。 このコマンドでは、IP アドレスが一覧表示されたエンドポイント スライスが返されるはずです。

    ```
    kubectl get endpointslices -l kubernetes.io/service-name=api-service -n aks-troubleshoot
    ```

ラベルの不一致問題を修正したので、次は CrashLoopBackOff を診断します。

### CrashLoopBackOff を診断する

コンテナーの起動に失敗すると、Kubernetes によって繰り返し再起動され、**CrashLoopBackOff** 状態になります。 ログを読むことで、アプリケーションがクラッシュした理由がわかります。

1. 次のコマンドを実行して、必要な **API_KEY** 環境変数を削除するデプロイ構成を適用します。

    ```
    kubectl apply -f k8s/crashloop-deployment.yaml -n aks-troubleshoot
    ```

1. 次のコマンドを実行して、ポッドの状態を監視します。 しばらくすると、ポッドは **CrashLoopBackOff** に入ります。 **ctrl + c** キーを押してコマンドを終了します。

    ```
    kubectl get pods -n aks-troubleshoot -w
    ```

1. 以下のコマンドを実行して、ポッド ログでエラー メッセージを確認します。 環境変数が欠落していることを示すエラーが表示されるはずです。

    ```
    kubectl logs -l app=api -n aks-troubleshoot
    ```

1. Deployment を編集して **API_KEY** 環境変数を追加することで問題を解決します。

    ```
    kubectl edit deployment api-deployment -n aks-troubleshoot
    ```

    エディターで、**spec.template.spec** の下にある **containers** セクションを見つけます。**name: api** 行を見つけ、そのすぐ下に **env** ブロックを追加し、**name** のインデントと一致させます。

    ```yaml
        name: api
        env:
        - name: API_KEY
          value: "demo-api-key-12345"
        ports:
    ```

    変更を保存してエディターを閉じます。

1. 次のコマンドを実行して、ポッドの状態を監視します。 しばらくすると、ポッドが **Running** に入ります。 **ctrl + c** キーを押してコマンドを終了します。

    ```
    kubectl get pods -n aks-troubleshoot -w
    ```

CrashLoopBackOff の問題が解決しました。次は readiness probe の失敗を診断します。

### readiness probe の失敗を診断する

readiness probe が失敗すると、ポッドには **Running** と表示されますが、コンテナーの準備状態は **0/1** となります。 Kubernetes では、準備チェックに合格するまで Service エンドポイントにポッドが追加されません。 ローリング更新戦略では、古い作業ポッドでトラフィックの処理が続行されますが、新しいポッドは準備未完了状態のままです。

1. 次のコマンドを実行して、readiness probe の失敗を発生させるデプロイ構成を適用します。 これにより、準備チェックに無効なパスが適用されます。

    ```
    kubectl apply -f k8s/probe-failure-deployment.yaml -n aks-troubleshoot
    ```

1. 次のコマンドを実行して、ポッドの状態を確認します。 2 つのポッドが見えるはずです。新しいポッドでは **Running** と表示されますが、READY 列には **0/1** と示され、古いポッドでは **1/1** 準備完了のままです。 新しいポッドの準備が整わないため、ローリング更新はブロックされます。

    ```
    kubectl get pods -n aks-troubleshoot
    ```

1. 次のコマンドを実行して、プローブの失敗イベントを確認します。 このコマンドでは、readiness probe と liveness probe の失敗の両方を含む、すべての**異常な**イベントが返されます。 readiness probe が 404 状態コードで失敗したことを示すメッセージを探します。

    ```
    kubectl get events -n aks-troubleshoot --field-selector reason=Unhealthy
    ```

1. 次のコマンドを実行し、Deployment を編集してパスを修正することで readiness probe を修正します。

    ```
    kubectl edit deployment api-deployment -n aks-troubleshoot
    ```

    エディターで **readinessProbe** セクションを見つけて、**path: /invalid-path** を **path: /healthz** に変更します。 変更を保存してエディターを閉じます。

1. 次のコマンドを実行して、新しいポッドの準備が整い、古いポッドが終了することを確認します。 **Running** 状態で READY 列に **1/1** と示されたポッドが 1 つのみが表示されるはずです。

    ```
    kubectl get pods -n aks-troubleshoot
    ```

readiness probe エラーを診断して解決しました。次は、エンドツーエンドの接続を確認します。

### エンドツーエンド接続を確認する

すべてのトラブルシューティング シナリオを完了した後、アプリケーションが完全に機能していることを確認します。

1. 次のコマンドを実行し、port-forward を使用して Service にアクセスします。

    ```
    kubectl port-forward service/api-service 8080:80 -n aks-troubleshoot
    ```

1. メニューバーで **[ターミナル] > [新しいターミナル]** を選択して、VS Code で 2 つ目のターミナル ウィンドウを開きます。 次のコマンドを実行して、すべてのエンドポイントをテストします。

    ```bash
    # Bash
    curl http://localhost:8080/healthz
    curl http://localhost:8080/readyz
    curl http://localhost:8080/api/info
    ```

    ```powershell
    # PowerShell
    Invoke-RestMethod http://localhost:8080/healthz
    Invoke-RestMethod http://localhost:8080/readyz
    Invoke-RestMethod http://localhost:8080/api/info
    ```

1. 次のコマンドを実行し、ポッド ログを確認して要求を確かめます。

    ```
    kubectl logs -l app=api -n aks-troubleshoot
    ```

アプリケーションが完全に機能していることを確認しました。次はリソースをクリーンアップします。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. VS Code ターミナルで次のコマンドを実行し、リソース グループと、そのグループ内のすべてのリソースを削除します。 **\<rg-name>** は、この演習で選択した名前に置き換えてください。 このコマンドを実行すると Azure の中でバックグラウンド タスクが起動されてリソース グループが削除されます。

    ```
    az group delete --name <rg-name> --no-wait --yes
    ```

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

## トラブルシューティング

この演習の設定中に問題が発生した場合は、次のトラブルシューティング手順をお試しください。

**デプロイ スクリプトでデプロイ状態を確認する**
- デプロイ スクリプトを実行し、オプションの **[6. Check deployment status]** を選択して、デプロイされたすべてのリソースの状態を確認します。
- このコマンドでは、ACR プロビジョニングの状態、AKS クラスターのプロビジョニング状態、Kubernetes リソースの可用性を確認します。
- この出力を使用して、問題の原因となっている可能性のあるコンポーネントを特定します。

**ACR イメージのプル エラー**
- ポッドに **ImagePullBackOff** または **ErrImagePull** の状態が表示される場合は、ACR リソースが作成され、イメージが正常にプッシュされたことを確認します。
- 必要に応じて、もう一度デプロイ スクリプト オプションの **[2. Build and push API image to ACR]** を実行します。
- デプロイ スクリプトの出力でロールの割り当てが成功したことを確認して、AKS クラスターに ACR からプルするアクセス許可があることを確認します。

**kubectl 接続の問題**
- kubectl コマンドが接続エラーで失敗した場合は、デプロイ スクリプト オプションの **[4. Get AKS credentials for kubectl]** を実行して資格情報を更新します。
- Azure portal を確認するか、**az aks show --resource-group \<rg-name> --name \<aks-name> --query provisioningState** を実行して、AKS クラスターが実行されていることを確認します。

**演習のリセット**
- トラブルシューティング シナリオを最初からやり直す必要がある場合は、デプロイ スクリプト オプションの **[5. Deploy applications to AKS]** を実行して元の作業構成を再デプロイします。
- これにより基本デプロイ ファイルとサービス ファイルが再適用され、演習中に行われた変更がリセットされます。
