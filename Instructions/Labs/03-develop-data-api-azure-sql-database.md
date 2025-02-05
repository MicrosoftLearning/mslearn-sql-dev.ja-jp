---
lab:
  title: Azure SQL Database 用のデータ API を開発する
  module: Develop a Data API for Azure SQL Database
---

# Azure SQL Database 用のデータ API を開発する

この演習では、Azure Static Web Apps を使用して、Azure SQL Database 用の Data API を開発してデプロイします。 これにより、データ API ビルダー構成を設定し、Azure Static Web App 環境内にデプロイする際の実践的な体験が提供されます。

## 前提条件

この演習を開始する前に、以下を必ず取得してください。

- アクティブな Azure サブスクリプション。
- Azure SQL Database、Azure Static Web Apps、GitHub に関する基本的な知識。
- 必要な拡張機能がインストールされている Visual Studio Code。
- リポジトリを管理するための GitHub アカウント。

## 環境を設定する

この演習の環境を設定するには、いくつかの手順を実行する必要があります。

### Visual Studio Code 拡張機能をインストールする

この演習を開始する前に、Visual Studio Code 拡張機能をインストールする必要があります。

1. Visual Studio Code を開きます。
1. Visual Studio Code でターミナル ウィンドウを開きます。
1. 次のコマンドを実行して Static Web Apps CLI をインストールします。

    ```bash
    npm install -g @azure/static-web-apps-cli
    ```

1. 次のコマンドを実行してデータ API ビルダー CLI をインストールします。

    ```bash
    dotnet tool install --global Microsoft.DataApiBuilder
    ```

これで、必要な拡張機能を使用して Visual Studio Code が設定されました。

### Azure SQL Database の作成

まだ作成していない場合は、Azure SQL Database を作成する必要があります。

1. [Azure portal](https://portal.azure.com?azure-portal=true) にサインインします。 
1. **[Azure SQL]** ページに移動し、**[+ 作成]** を選択します。
1. **[SQL Database]**、*[単一データベース]*、**[作成]** ボタンの順に選択します。
1. **[SQL データベースの作成]** ダイアログに必要な情報を入力し、**[OK]** を選択します (他のオプションはすべて既定値のままにします)。

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
1. **[保存]** を選択します。

### データベースにサンプル データを追加する

Azure SQL Database が作成されたので、いくつかのサンプル データを追加する必要があります。 これは、API を稼働させた後にテストするのに役立ちます。

1. 新しく作成した Azure SQL Database に移動します。
1. Azure portal で**クエリ エディター**を使用して、次の SQL スクリプトを実行します。

    ```sql
    CREATE TABLE [dbo].[Employees]
    (
        EmployeeID INT PRIMARY KEY,
        FirstName NVARCHAR(50),
        LastName NVARCHAR(50),
        Department NVARCHAR(50)
    );
    
    INSERT INTO [dbo].[Employees] VALUES (1,'John', 'Smith', 'Accounting');
    INSERT INTO [dbo].[Employees] VALUES (2,'Michelle', 'Sanchez', 'IT');
    
    SELECT * FROM [dbo].[Employees];
    ```

### GitHub で基本的な Web アプリを作成する

Azure Static Web Apps を作成する前に、GitHub で基本的な Web アプリを作成する必要があります。

1. GitHub で基本的な Web アプリを作成するには、「[バニラ Web サイトを生成する](https://github.com/staticwebdev/vanilla-basic/generate)」に移動します。
1. Repostiory テンプレートが **staticwebdev/vanilla-basic** に設定されていることを確認します。
1. ***[所有者]*** で自分のアカウントを選択します。
1. ***[リポジトリ名]*** に「**my-sql-repo**」という名前を入力します。
1. リポジトリを **[プライベート]** にします。
1. **[リポジトリの作成]** ボタンを選択します。

## Azure Static Web アプリを作成する

最初に静的 Web アプリを作成してから、データ API ビルダーの構成を追加します。

1. Azure portal で、**[静的 Web アプリ]** ページに移動します。
1. **[+ 作成]** を選択します。
1. **[静的 Web アプリの作成]** ダイアログに次の情報を入力します (他のオプションはすべて既定値のままにします)。

    | 設定 | 値 |
    | --- | --- |
    | サブスクリプション | 該当するサブスクリプション |
    | リソース グループ | *新しいリソース グループを選択または作成します* |
    | Name | *一意の名前* |
    | ホスティング プランのソース | *GitHub* |
    | GitHub アカウント | *アカウントを選択します* |
    | 組織 | *ほとんどの場合、GitHub ユーザー名* |
    | リポジトリ | *前のステップで作成したリポジトリを選択します* |
    | [Branch]\(ブランチ) | *main* |

1. **[確認および作成]** 、 **[作成]** の順に選択します。
1. デプロイされたら、リソースに移動します。
1. **[ブラウザーでアプリを表示]** ボタンをクリックすると、ウェルカム メッセージが表示されたシンプルな Web ページが表示されます。 このタブは閉じてもかまいません。

## データ API ビルダーの構成ファイルを追加する

データ API ビルダーの構成を Azure Static Web アプリに追加します。 データ API ビルダーの構成を追加するには、GitHub リポジトリに新しいファイルを作成する必要があります。

1. Visual Studio Code で、前に作成した GitHub リポジトリをクローンします。
1. ターミナル ウィンドウを Visual Studio Code で開きます。
1. 次のコマンドを実行して、新しいデータ API ビルダーの構成ファイルを作成します。

    ```bash
    swa db init --database-type "mssql"
    ```

    これにより、*swa-db-connections* という名前の新しいフォルダーと、そのフォルダー内に *staticwebapp.database.config.json* という名前のファイルが作成されます。

1. 次のコマンドを実行して、データベース エンティティを構成ファイルに追加します。

    ```bash
    dab add "Employees" --source "dbo.Employees" --permissions "anonymous:*" --config "swa-db-connections/staticwebapp.database.config.json"
    ```

1. *staticwebapp.database.config.json* ファイルの内容を確認します。 
1. 変更をコミットし、GitHub リポジトリにプッシュします。

## データベース接続を構成する

1. Azure portal で、作成した Azure Static Web アプリに移動します。
1. **[設定]** で、**[データベース接続]** を選択します。
1. **[既存のデータベースをリンクする]** を選択します。
1. **[データベースをリンクする]* ダイアログで、前に作成した Azure SQL Database を次の追加設定で選択します。

    | 設定 | Value |
    | --- | --- |
    | データベースの種類 | *Azure SQL Database* |
    | 認証の種類 | *接続文字列* |
    | ユーザー名 | *管理者ユーザー名* |
    | Password | *管理者ユーザーに与えたパスワード* |
    | [受信確認] チェック ボックス | *オン* |

   > **注:** 運用環境では、必要な IP アドレスのみにアクセスを制限します。 さらに、SQL 認証ではなく、静的 Web アプリのマネージド ID を使用して、データベースにアクセスすることを検討してください。 詳しくは、「[Azure SQL 用の Microsoft Entra でのマネージド ID](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true)」をご覧ください。
1. **リンク**を選択します。

## Data API エンドポイントをテストする

ここで必要なのは、Data API エンドポイントをテストすることだけです。

1. Azure portal で、作成した Azure Static Web アプリに移動します。
1. [概要] ページで、Web アプリの URL をコピーします。
1. 新しいブラウザー タブを開いて、この URL を貼り付けます。 **Vanilla JavaScript アプリ**というメッセージが表示されたシンプルな Web ページが表示されます。
1. URL の末尾に **/data-api** を追加し、**Enter** キーを押します。 Data API が動作していることを示すために、**[正常]** が表示されます。
1. URL の末尾に **/data-api/rest/Employees** を追加し、**Enter** キーを押します。 Azure SQL Database に前に追加したサンプル データが表示されます。

Azure Static Web Apps を使用して、Azure SQL Database 用のデータ API を正常に開発してデプロイしました。

## クリーンアップ

独自のサブスクリプションを使用している場合は、プロジェクトの最後に、作成したリソースがまだ必要かどうかを確認してください。 

リソースを不必要に実行したままにしておくと、追加コストが発生する可能性があります。 [Azure portal](https://portal.azure.com?azure-portal=true) でリソースを個別に削除することも、リソースのセット全体を削除することもできます。

## 詳細

Azure SQL Database のデータ API ビルダーの詳細については、「[Azure Database 用のデータ API ビルダーとは](https://learn.microsoft.com/azure/data-api-builder/overview?azure-portal=true)」をご覧ください。
