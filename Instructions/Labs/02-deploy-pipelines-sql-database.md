---
lab:
  title: Azure SQL Database プロジェクト用の CI/CD パイプラインを構成してデプロイする
  module: Develop for an Azure SQL Database
---

# Azure SQL Database プロジェクト用の CI/CD パイプラインを構成してデプロイする

この演習では、Visual Studio Code と GitHub Actions を使って、Azure SQL Database プロジェクトのための CI/CD パイプラインを作成、構成、デプロイします。 これにより、Azure SQL Database プロジェクト用の CI/CD パイプラインを設定するプロセスを理解できます。

この演習の所要時間は約 **30** 分です。

## 開始する前に

この演習を開始する前に、以下を行う必要があります。

- リソースを作成および管理するための適切なアクセス許可がある Azure サブスクリプション。
- お使いのコンピューターに [Visual Studio Code](https://code.visualstudio.com/download) と次の拡張機能がインストールされていること。
  - [SQL Database プロジェクト](https://marketplace.visualstudio.com/items?itemName=ms-mssql.mssql)。
  - [GitHub pull request](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-pull-request-github)。
- GitHub アカウント。
- *GitHub Actions* パイプラインに関する基本的な知識

## Azure SQL Database の作成

まず、新しい Azure SQL Database を作成する必要があります。

1. [Azure portal](https://portal.azure.com?azure-portal=true) にサインインします。 
1. **[Azure SQL]** ページに移動し、**[+ 作成]** を選択します。
1. **[SQL Database]**、*[単一データベース]*、**[作成]** ボタンの順に選択します。
1. **[SQL データベースの作成]** ダイアログに必要な情報を入力し、**[OK]** を選択し、他のオプションはすべて既定値のままにします。

    | 設定 | Value |
    | --- | --- |
    | 無料のサーバーレス オファー | *オファーの適用* |
    | サブスクリプション | 該当するサブスクリプション |
    | リソース グループ | *新しいリソース グループを選択または作成します* |
    | データベース名 | *MyDB* |
    | [サーバー] | ***[新規作成]** リンクを選択します* |
    | サーバー名 | *一意の名前を選択する* |
    | Location | *場所を選択してください* |
    | 認証方法 | *SQL 認証を使用する* |
    | サーバー管理者のログイン | *sqladmin* |
    | パスワード | *[パスワード] を入力します* |
    | [パスワードの確認入力] | *パスワードを確認します* |

1. **[確認および作成]**、**[作成]** の順に選択します。
1. デプロイが完了したら、作成した Azure SQL Database *サーバー*に移動します。
1. 左側のペインの **[セキュリティ]** で、**[ネットワーク]** を選択します。 IP アドレスをファイアウォール規則に追加します。
1. **[Azure サービスおよびリソースにこのサーバーへのアクセスを許可する]** オプションを選択します。 このオプションを使用すると、GitHub Actions からデータベースにアクセスできます。

    > **注:** 運用環境では、必要な IP アドレスのみにアクセスを制限します。 さらに、SQL 認証ではなく、GitHub Action のマネージド ID を使用して、データベースにアクセスすることを検討してください。 詳しくは、「[Azure SQL 用の Microsoft Entra でのマネージド ID](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true)」をご覧ください。

1. **[保存]** を選択します。

## GitHub リポジトリを設定する

次に、新しい GitHub リポジトリを設定する必要があります。

1. [GitHub](https://github.com) Web サイトを開きます。
1. GitHub アカウントにサインインします。
1. アカウントの **[Repositories]** に移動し、**[New]** を選択します。
1. **Owner** のアカウントを選択します。 「**my-sql-db-repo**」という名前を入力します。
1. リポジトリを **[Private]** に設定します。
1. **[Create repository]** を選択します。

### Visual Studio Code 拡張機能をインストールし、リポジトリを複製する

リポジトリを複製する前に、必要な **Visual Studio Code** 拡張機能がインストールされていることを確認します。 ガイダンスについては、「**開始する前に**」セクションをご覧ください。

1. Visual Studio Code で、 **[表示]** > **[コマンド パレット]** の順に選択します。
1. コマンド パレットに「`Git: Clone`」と入力し、それを選択します。
1. 前の手順で作成したリポジトリの URL を入力して、**[複製]** を選択します。 URL は、次の形式に従う必要があります。*https://github.com/<your_account>/<your_repository>.git*
1. リポジトリ ファイルを格納するフォルダーを選択または作成します。

## Azure SQL Database プロジェクトの作成と構成

Visual Studio の Azure SQL Database プロジェクトを使用すると、データベース スキーマとデータを開発、ビルド、テスト、発行できます。 このセクションでは、プロジェクトを作成し、前に設定した Azure SQL Database に接続するように構成します。

1. Visual Studio Code で、 **[表示]** > **[コマンド パレット]** の順に選択します。
1. コマンド パレットに「`Database projects: New`」と入力し、それを選択します。
    > **注:** mssql 拡張機能用の SQL ツール サービスのインストールには数分かかる場合があります。
1. **[Azure SQL Database]** を選択します。
1. 「**MyDBProj**」という名前を入力し、**Enter** キーを押して確定します。
1. 複製した GitHub リポジトリ フォルダーを選択して、プロジェクトを保存します。
1. **[SDK スタイルのプロジェクト]** で、**[はい (推奨)]** を選択します。
    > **注:** **MyDBProj** という名前の新しいプロジェクトが作成されることに注目してください。

### プロジェクトに新しい .SQL ファイルを作成する

Azure SQL Database プロジェクトを作成したので、新しい SQL ファイルをプロジェクトに追加して新しいテーブルを作成しましょう。

1. Visual Studio Code で、左側のアクティビティ バーにある **[データベース プロジェクト]** アイコンを選択します。
1. プロジェクト名を右クリックし、**[テーブルの追加]** を選択します。
1. テーブルに **Employees** という名前を付け、**Enter** キーを押します。
1. 既存のスクリプトを次のコードに置き換えます。

    ```sql
    CREATE TABLE [dbo].[Employees]
    (
        EmployeeID INT PRIMARY KEY,
        FirstName NVARCHAR(50),
        LastName NVARCHAR(50),
        Department NVARCHAR(50)
    );
    ```

1. エディターを閉じます。 `Employees.sql` ファイルがプロジェクトに保存されていることに注目してください。

## 変更をリポジトリにコミットする

Azure SQL Database プロジェクトを作成し、テーブル スクリプトをプロジェクトに追加したので、リポジトリに変更をコミットしましょう。

1. Visual Studio Code で、左側のアクティビティ バーにある **[ソース管理]** アイコンを選択します。
1. 「*Created project and added a create table script*」というメッセージを入力します。
1. **[コミット]** を選択して、変更をコミットします。
1. 省略記号の下にある **[プッシュ]** を選択して、変更をリポジトリにプッシュします。

## リポジトリで変更を確認します。

変更をプッシュしたので、GitHub リポジトリでそれらを確認しましょう。

1. [GitHub](https://github.com) Web サイトを開きます。
1. **my-sql-db-repo** リポジトリに移動します。
1. **[<> Code]** タブで、**MyDBProj** フォルダーを開きます。
1. **Employees.sql** ファイルの変更が最新かどうかを確認します。

## GitHub Actions を使用して継続的インテグレーション (CI) を設定する

GitHub Actions を使用すると、お使いの GitHub リポジトリ内でソフトウェア開発ワークフローを自動化、カスタマイズ、実行できます。 このセクションでは、データベースに新しいテーブルを作成して、Azure SQL Database プロジェクトをビルドしてテストするように GitHub Actions ワークフローを構成します。

### サービス プリンシパルを作成する

1. Azure portal の右上隅にある **[Cloud Shell]** アイコンを選択します。 `>_` 記号のように表示されます。 メッセージが表示されたら、シェルの種類として **Bash** を選択します。

1. Cloud Shell ターミナルで次のコマンドを実行します。 `<your_subscription_id>` と `<your_resource_group_name>` の値を、実際の値に置き換えます。 これらの値は、Azure portal の **[サブスクリプション]** および **[リソース グループ]** ページで取得できます。

    ```azurecli
    az ad sp create-for-rbac --name "MyDBProj" --role contributor --scopes /subscriptions/<your_subscription_id>/resourceGroups/<your_resource_group_name>
    ```

    テキスト エディターを開き、前のコマンドの出力を使用して、次のような資格情報スニペットを作成します。
    
    ```
    {
    "clientId": <your_service_principal_appId>,
    "clientSecret": <your_service_principal_password>,
    "tenantId": <your_service_principal_tenant>,
    "subscriptionId": <your_subscription_id>
    }
    ```

1. テキスト エディターは開いたままにしておきます。 これは、次のセクションで参照します。

### シークレットをリポジトリに追加する

1. GitHub リポジトリで、**[設定]** を選択します。
1. **[シークレットと変数]** を選択し、**[アクション]** を選択します。
1. **[シークレット]** タブで、**[新しいリポジトリ シークレット]** を選択し、次の情報を入力します。

    | 名前 | 値 |
    | --- | --- |
    | AZURE_CREDENTIALS | 前のセクションでコピーしたサービス プリンシパルの出力。|
    | AZURE_CONN_STRING | 接続文字列。 |
   
    接続文字列は次のようになります。

    ```Server=<your_sqldb_server>.database.windows.net;Initial Catalog=MyDB;Persist Security Info=False;User ID=sqladmin;Password=<your_password>;Encrypt=True;Connection Timeout=30;```

### GitHub Actions ワークフローを作成する

1. GitHub リポジトリで、**[Actions] (アクション)** タブを選択します。
1. **[set up a workflow yourself]** オプションを選択します。
1. 次のコードを **main.yml** ファイルにコピーします。 このコードには、データベース プロジェクトをビルドしてデプロイする手順が含まれています。

    {% raw %}
    ```yaml
    name: Build and Deploy SQL Database Project
    on:
      push:
        branches:
          - main
    jobs:
      build:
        permissions:
          contents: 'read'
          id-token: 'write'
          
        runs-on: ubuntu-latest  # Can also use windows-latest depending on your environment
        steps:
          - name: Checkout repository
            uses: actions/checkout@v3

        # Install the SQLpackage tool
          - name: sqlpack install
            run: dotnet tool install -g microsoft.sqlpackage
    
          - name: Login to Azure
            uses: azure/login@v1
            with:
              creds: ${{ secrets.AZURE_CREDENTIALS }}
    
          # Build and Deploy SQL Project
          - name: Build and Deploy SQL Project
            uses: azure/sql-action@v2.3
            with:
              connection-string: ${{ secrets.AZURE_CONN_STRING }}
              path: './MyDBProj/MyDBProj.sqlproj'
              action: 'publish'
              build-arguments: '-c Release'
              arguments: '/p:DropObjectsNotInSource=true'  # Optional: Customize as needed
    ```
    {% endraw %}
   
      YAML ファイルの **SQL プロジェクトをビルドしてデプロイする**手順は、`AZURE_CONN_STRING` シークレットに格納されている接続文字列を使用して Azure SQL Database に接続します。 このアクションは、SQL プロジェクト ファイルへのパスを指定し、プロジェクトをデプロイするために発行するアクションを設定し、リリース モードでコンパイルするビルド引数を含めます。 さらに、`/p:DropObjectsNotInSource=true` 引数を使用して、ソースに存在しないオブジェクトがデプロイ中にターゲット データベースから削除されるようにします。

1. 変更をコミットします。

### GitHub Actions ワークフローをテストする

1. GitHub リポジトリで、**[Actions] (アクション)** タブを選択します。
1. **SQL Database プロジェクトをビルドしてデプロイする**ワークフローを選択します。
    > **注:** ワークフローが進行中であることがわかります。 完了するまで待ちます。 既に完了している場合は、最新の実行を選択して詳細を表示します。

### Azure SQL Database の変更を確認する

GitHub Actions ワークフローを設定して、Azure SQL Database プロジェクトをビルドしてデプロイしたので、Azure SQL Database の変更を確認します。

1. [Azure portal](https://portal.azure.com?azure-portal=true) にサインインします。 
1. **MyDB** SQL Database に移動します。
1. **[クエリ エディター]** を選択します。
1. **sqladmin** 資格情報を使用してデータベースに接続します。
1. **Tables** セクションで、**Employees** テーブルが作成されていることを確認します。 必要に応じて更新します。

Azure SQL Database プロジェクトをビルドしてデプロイするための GitHub Actions ワークフローが正常に設定されました。

## クリーンアップ

独自のサブスクリプションを使用している場合は、プロジェクトの最後に、作成したリソースがまだ必要かどうかを確認してください。 

リソースを不必要に実行したままにしておくと、追加コストが発生する可能性があります。 [Azure portal](https://portal.azure.com?azure-portal=true) でリソースを個別に削除することも、リソースのセット全体を削除することもできます。

## 詳細

Azure SQL Database 用の SQL Database プロジェクトの拡張機能の詳細については、「[SQL Database プロジェクトの拡張機能をお使いになる前に](https://learn.microsoft.com/azure-data-studio/extensions/sql-database-project-extension-getting-started?azure-portal=true)」をご覧ください。
