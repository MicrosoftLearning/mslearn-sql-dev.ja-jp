---
lab:
  title: Azure SQL Database のマネージド ID を構成する
  module: Explore Azure SQL Database safety practices for development
---

# Azure SQL Database のマネージド ID を構成する

この演習では、コードに資格情報を格納せずに、サンプル Web アプリにマネージド ID を追加します。

Azure App Service は、高度にスケーラブルで自己管理性の高い Web ホスティング ソリューションを提供します。 その主な機能の 1 つは、アプリケーション用のマネージド ID のプロビジョニングで、Azure SQL Database やその他の Azure サービスへのアクセスをセキュリティで保護するのが簡単になります。 マネージド ID を使用すると、資格情報などの機密情報を接続文字列内に格納する必要がなくなることで、アプリケーションのセキュリティを強化できます。 

この演習の所要時間は約 **30** 分です。

## 開始する前に

この演習を開始する前に、以下を行う必要があります。

- リソースを作成および管理するための適切なアクセス許可がある Azure サブスクリプション。
- [**SQL Server Management Studio (SSMS)**](https://learn.microsoft.com/en-us/ssms/install/install) がコンピューターにインストールされている。
- 次の拡張機能を備えてコンピューターにインストールされた [**Visual Studio Code**](https://code.visualstudio.com/download?azure-portal=true)。
    - [Azure App Service](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azureappservice?azure-portal=true)。

## Web アプリケーションと Azure SQL データベースを作成する

まず、Web アプリケーションと Azure SQL データベースを作成します。

1. [Azure portal](https://portal.azure.com?azure-portal=true) にサインインします。
1. **[サブスクリプション]** を検索して選択します。
1. **[設定]** の **[リソース プロバイダー]** に移動し、**Microsoft.Sql** プロバイダーを検索して **[登録]** を選択します。
1. Azure portal のメイン ページに戻り、**[リソースの作成]** を選択します。
1. **Web App** を検索して選択します。
1. **[+ 作成]** を選択し、必須の詳細を入力します。

    | グループ | 設定 | 値 |
    | --- | --- | --- |
    | **プロジェクトの詳細** | **サブスクリプション** | Azure サブスクリプションを選択します。 |
    | **プロジェクトの詳細** | **リソース グループ** | 新しいリソース グループを選択または作成します |
    | **インスタンスの詳細** | **名前** | Web アプリの一意の名前を入力します。 |
    | **インスタンスの詳細** | **ランタイム スタック** | .NET 8 (LTS) |
    | **インスタンスの詳細** | **リージョン** | Web アプリをホストするリージョンを選択します。 |
    | **価格プラン** | **料金プラン** | 基本 |
    | **データベース** | **エンジン** | SQLAzure |
    | **データベース** | **サーバー名** | 使用する SQL サーバーの一意の名前を入力します。 |
    | **データベース** | **データベース名** | 使用するデータベースの一意の名前を入力します。 |
    

    > **注:** 実稼働ワークロードの場合は、**[Standard : 汎用の運用アプリの場合]** を選択してください。 新しいデータベースのユーザー名とパスワードは自動的に生成されます。 デプロイ後にこれらの値を取得するには、アプリの **[環境変数]** ページにある **[接続文字列]** に移動します。 

1. **[確認および作成]** 、 **[作成]** の順に選択します。 デプロイが完了するまでに数分かかる場合があります。
1. SSMS でデータベースに接続し、次のコードを実行します。

    >**ヒント**: Web アプリ リソースの [サービス コネクタ] ページで接続文字列を確認すると、サーバーに割り当てられたユーザー ID とパスワードを取得できます
 
    ```sql
    CREATE TABLE Products (
        ProductID INT PRIMARY KEY,
        ProductName NVARCHAR(100),
        Category NVARCHAR(50),
        Price DECIMAL(10, 2),
        Stock INT
    );
    
    INSERT INTO Products (ProductID, ProductName, Category, Price, Stock) VALUES
    (1, 'Laptop', 'Electronics', 999.99, 50),
    (2, 'Smartphone', 'Electronics', 699.99, 150),
    (3, 'Desk Chair', 'Furniture', 89.99, 200),
    (4, 'Coffee Maker', 'Appliances', 49.99, 100),
    (5, 'Book', 'Books', 19.99, 300);
    ```

## SQL 管理者としてアカウントを追加する

次に、データベースへのアカウント アクセスを追加します。 Microsoft Entra を通じて認証されたアカウントのみが、この演習の後続の手順の前提条件である他の Microsoft Entra ID ユーザーを作成できるため、これが必要です。

1. 先ほど作成した Azure SQL サーバーに移動します。
1. 左側の **[設定]** メニューで **[Microsoft Entra ID]** を選択します。
1. **[管理者の設定]** を選びます。
1. 自分のアカウントを検索して選択します。
1. **[保存]** を選択します。

## マネージド ID の有効化

次に、Azure Web アプリのシステム割り当てマネージド ID を有効にします。これは、資格情報の自動管理が可能になるセキュリティのベスト プラクティスです。

1. Azure portal で Web App に移動します。
1. 左側メニューの **[設定]** で、**[ID]** を選択します。
1. **[システム割り当て済み]** タブで、**[状態]** を **[オン]** に切り替えて、**[保存]** を選択します。 Web アプリのシステム割り当てマネージド ID を有効にするかどうかを確認するメッセージが表示された場合は、**[はい]** を選択します。

## Azure SQL データベースへのアクセス権を付与する

1. SSMS を使用して、Azure SQL データベースにもう一度接続します。 **Microsoft Entra MFA** を選択し、ユーザー名を指定します。
1. 使用するデータベースを選択し、新しいクエリ エディターを開きます。
1. 次の SQL コマンドを実行して、マネージド ID のユーザーを作成し、必要なアクセス許可を割り当てます。 Web アプリ名を指定してスクリプトを編集します。

    ```sql
    CREATE USER [your-web-app-name] FROM EXTERNAL PROVIDER;
    ALTER ROLE db_datareader ADD MEMBER [your-web-app-name];
    ALTER ROLE db_datawriter ADD MEMBER [your-web-app-name];
    ```

##  Web アプリケーションを作成する

次に、Entity Framework Core と Azure SQL Database を使用して製品テーブルからの製品の一覧を表示する ASP.NET アプリケーションを作成します。

### プロジェクトを作成する

1. VS Code で、新しいフォルダーを作成します。 フォルダーの名前をプロジェクト名に設定します。
1. ターミナルを開き、次のコマンドを実行して、新しい MVC プロジェクトを作成します。
    
    ```dos
   dotnet new mvc
    ```
    こうすることで、選択したフォルダーに新しい ASP.NET MVC プロジェクトが作成され、Visual Studio Code に読み込まれます。

1. 次のコマンドを実行して、アプリケーションを実行します。 

    ```dos
   dotnet run
    ```
1. ターミナルに *Now listening on: http://localhost:<port>* と出力されます。 Web ブラウザーで URL に移動して、アプリケーションにアクセスします。 

1. Web ブラウザーを閉じて、アプリケーションを停止します。 または、VS Code ターミナルで `Ctrl+C` キーを押して、アプリケーションを停止することもできます。

### Azure SQL Database に接続するようにプロジェクトを更新する

次に、マネージド ID を使用して Azure SQL データベースに正常に接続できる構成をいくつか更新します。

1. プロジェクトで、SQL Server に必要な NuGet パッケージを追加します。
    ```dos
    dotnet add package Microsoft.EntityFrameworkCore.SqlServer
    ```
1. プロジェクトのルート フォルダーで、**appsettings.json** ファイルを開き、`ConnectionStrings` セクションを挿入します。 ここでは、`<server-name>` と `<db-name>` をサーバーおよびデータベースの実際の名前に置き換えます。 この接続文字列は、データベースへの接続を確立するために、`Models/MyDbContext.cs` ファイルの既定のコンストラクターによって使用されます。

    ```json
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning"
        }
      },
      "AllowedHosts": "*",
      "ConnectionStrings": {
        "DefaultConnection": "Server=<server-name>.database.windows.net,1433;Initial Catalog=<db-name>;Authentication=Active Directory Default;"
      }
    }
    ```
1. ファイルを保存して閉じます。

### コードを追加する

1. プロジェクトの **Models** フォルダーで、次のコードを使用して製品エンティティの **Product.cs** ファイルを作成します。 `<app name>` を自分のアプリケーションの名前と置き換えます。

    ```csharp
    namespace <app name>.Models;
    
    public class Product
    {
        public int ProductId { get; set; }
        public string ProductName { get; set; }
        public string Category { get; set; }
        public decimal Price { get; set; }
        public int Stock { get; set; }
    }
    ```
1. プロジェクトのルート フォルダーで **Database** フォルダーを作成します。
1. プロジェクトの **Database** フォルダーで、次のコードを使用して製品エンティティの **MyDbContext.cs** ファイルを作成します。 `<app name>` を自分のアプリケーションの名前と置き換えます。

    ```csharp
    using <app name>.Models;
    
    namespace <app name>.Database;
    
    using Microsoft.EntityFrameworkCore;
    
    public class MyDbContext : DbContext
    {
        public MyDbContext(DbContextOptions<MyDbContext> options) : base(options)
        {
        }
    
        public DbSet<Product> Products { get; set; }
    }    
    ```
1. プロジェクトの **Controllers** フォルダーで、**HomeController.cs** ファイルのクラス `HomeController` と `IActionResult` を編集し、次のコードを使用して`_context` 変数を追加します。

    ```csharp
    private MyDbContext _context;

    public HomeController(ILogger<HomeController> logger, MyDbContext context)
    {
        _logger = logger;
        _context = context;
    }

    public IActionResult Index()
    {
        var data = _context.Products.ToList();
        return View(data);
    }
    ```

1. また、ファイルの先頭に `using.<app name>.Database` を追加します。
1. プロジェクトの **Views -> Home** フォルダーで、**Index.cshtml** ファイルを更新し、次のコードを追加します。

    ```html
    <table class="table">
        <thead>
            <tr>
                <th>Product Id</th>
                <th>Product Name</th>
                <th>Category</th>
                <th>Price</th>
                <th>Stock</th>
            </tr>
        </thead>
        <tbody>
            @foreach(var item in Model)
            {
                <tr>
                    <td>@item.ProductId</td>
                    <td>@item.ProductName</td>
                    <td>@item.Category</td>
                    <td>@item.Price</td>
                    <td>@item.Stock</td>
                </tr>
            }
        </tbody>
    </table>
    ```

1. **Program.cs** ファイルを編集し、指定されたコード スニペットを `var app = builder.Build();` 行のすぐ上に挿入します。 この変更により、アプリケーションのスタートアップ シーケンス中にコードが確実に実行されます。 `<app name>` を自分のアプリケーションの名前と置き換えます。

    ```csharp
    using Microsoft.EntityFrameworkCore;
    using <app name>.Database;

    var builder = WebApplication.CreateBuilder(args);

    // Add services to the container.
    builder.Services.AddControllersWithViews();
    builder.Services.AddDbContext<MyDbContext>(options =>
        options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
    ```

    > **注:** デプロイ前にアプリケーションを実行する場合は、SQL ユーザー資格情報を使用して接続文字列を更新します。 新しいデータベースのユーザー名とパスワードが自動的に生成されました。 デプロイ後にこれらの値を取得するには、アプリの **[環境変数]** ページにある **[接続文字列]** に移動します。 アプリケーションが想定どおりに実行されていることを確認したら、マネージド ID を使用してセキュリティで保護されたデプロイ プロセスに戻ります。

### コードのデプロイ

1. `Ctrl+Shift+P` を押して、**コマンド パレット**を開きます。
1. 「**Azure App Service: Web アプリにデプロイ...**」と入力して選択します。
1. Web アプリケーション コードが含まれるフォルダーを選択します。
1. 前の手順で作成した Web アプリを選択します。
    > 注: 「デプロイに必要な構成が "アプリ" にありません」というメッセージが表示される場合があります。 **[構成の追加]** を選択します。次に、指示に従ってサブスクリプションを選択し、App Service リソースを選択します。
1. メッセージが表示されたら、デプロイを確認します。

## アプリケーションのテスト

Web アプリケーションを実行し、保存されている資格情報なしで Azure SQL Database に接続できることを確認します。

1. ブラウザーを開き、Azure Web アプリの URL (たとえば、https://your-web-app-name.azurewebsites.net)) に移動します。
1. Web アプリケーションが実行されていてアクセス可能であることを確認します。
1. 以下のような Web ページが表示されます。

    ![デプロイ後の Web アプリケーションのスクリーンショット。](./Media/01-app-page.png)

## 継続的デプロイの設定 (オプション)

1. `Ctrl+Shift+P` を押して、**コマンド パレット**を開きます。
1. **Azure App Service: 継続的デリバリーを構成する...** と入力して選択します。
1. プロンプトに従って、GitHub リポジトリまたは Azure DevOps から継続的デプロイを設定します。

**System 割り当てマネージド ID** の代わりに、**User 割り当てマネージド ID** を使用すると役立つシナリオを検討します。

## クリーンアップ

独自のサブスクリプションを使用している場合は、プロジェクトの最後に、作成したリソースがまだ必要かどうかを確認してください。 

リソースを不必要に実行したままにしておくと、追加コストが発生する可能性があります。 [Azure portal](https://portal.azure.com?azure-portal=true) でリソースを個別に削除することも、リソースのセット全体を削除することもできます。

## 詳細

Azure SQL Database の詳細については、「[Azure SQL 用 Microsoft Entra のマネージド ID](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true)」を参照してください。
