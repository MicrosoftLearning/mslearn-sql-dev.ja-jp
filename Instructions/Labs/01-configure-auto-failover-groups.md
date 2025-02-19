---
lab:
  title: Azure SQL Database の自動フェールオーバー グループでアプリケーションの回復性を実現する
  module: Get started with Azure SQL Database for cloud-native application development
---

# Azure SQL Database の自動フェールオーバー グループでアプリケーションの回復性を実現する

この演習では、プライマリとセカンダリとして機能する 2 つの Azure SQL データベースを作成します。 自動フェールオーバー グループを構成してアプリケーション データベースの高可用性とディザスター リカバリーを確保し、アプリケーション上のレプリケーションの状態を検証します。

この演習の所要時間は約 **30** 分です。

## 開始する前に

この演習を開始する前に、以下を行う必要があります。

- リソースを作成および管理するための適切なアクセス許可がある Azure サブスクリプション。
- 次の拡張機能を備えてコンピューターにインストールされた [**Visual Studio Code**](https://code.visualstudio.com/download?azure-portal=true)。
    - [C# 開発キット](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit?azure-portal=true)

## プライマリとセカンダリの Azure SQL サーバーを作成する

最初に、プライマリ サーバーとセカンダリ サーバーの両方を設定し、**AdventureWorksLT** サンプル データベースを使用します。

1. [Azure portal](https://portal.azure.com?azure-portal=true) にサインインします。

1. Azure portal の右上隅にある [Cloud Shell] アイコンを選択します。 `>_` 記号のように表示されます。 メッセージが表示されたら、シェルの種類として **[Bash]** を選択します。

1. Cloud Shell ターミナルで次のコマンドを実行します。 `<your_resource_group>`、`<your_primary_server>`、`<your_location>`、`<your_secondary_server>`、`<your_admin_password>` の値を、実際の値に置き換えます。

    * リソース グループを作成する
    ```azurecli
    az group create --name <your_resource_group> --location <your_location>
    ```

    * プライマリ SQL サーバーを作成する
    ```azurecli        
    az sql server create --name <your_primary_server> --resource-group <your_resource_group> --location <your_location> --admin-user "sqladmin" --admin-password <your_admin_password>
    ```

    * セカンダリ SQL サーバーを作成します。 スクリプトは同じで、サーバー名と場所のみを変更します
    ```azurecli
    az sql server create --name <your_secondary_server> --resource-group <your_resource_group> --location <your_location> --admin-user "sqladmin" --admin-password <your_admin_password>    
    ```

    * 指定した価格レベルのプライマリ サーバーにサンプル データベースを作成する
    ```azurecli
    az sql db create --resource-group <your_resource_group> --server <your_primary_server> --name AdventureWorksLT --sample-name AdventureWorksLT --service-objective "S0"    
    ```
    
1. デプロイが完了したら、作成したプライマリ Azure SQL サーバーに移動します。
1. 左側のペインの **[セキュリティ]** で、**[ネットワーク]** を選択します。 IP アドレスをファイアウォール規則に追加します。
1. **[Azure サービスおよびリソースにこのサーバーへのアクセスを許可する]** オプションを選択します。
1. **[保存]** を選択します。
1. セカンダリ サーバーに対して上記の手順を繰り返します。

    これらの手順により、構造化された冗長な Azure SQL Database 環境を使用できるようになります。

## 自動フェールオーバー グループを構成する

次に、前に設定した Azure SQL Database の自動フェールオーバー グループを作成します。 これには、2 つのサーバー間でフェールオーバー グループを確立し、設定が正常に動作していることを確認する必要があります。

1. Cloud Shell ターミナルで次のコマンドを実行します。 `<your_failover_group>`、`<your_resource_group>`、`<your_primary_server>`、`<your_secondary_server>` の値を、実際の値に置き換えます。

    * フェールオーバー グループを作成する
    ```azurecli
    az sql failover-group create -n <your_failover_group> -g <your_resource_group> -s <your_primary_server> --partner-server <your_secondary_server> --failover-policy Automatic --grace-period 1 --add-db AdventureWorksLT
    ```

    * フェールオーバー グループを確認する
    ```azurecli    
    az sql failover-group show -n <your_failover_group> -g <your_resource_group> -s <your_primary_server>
    ```

    > 少し時間を取って、結果と `partnerServers` の値を確認してください。 これは、なぜ重要ですか。

    > 各パートナー サーバー内の `role` 属性を確認することで、サーバーが現在プライマリとセカンダリのどちらとして機能しているかを判断できます。 この情報は、フェールオーバー グループの現在の構成と準備状況を理解するために重要です。 これは、フェールオーバー シナリオ中にアプリケーションに与える可能性のある影響を評価し、高可用性とディザスター リカバリーのために設定が正しく構成されていることを確認するのに役立ちます。
    
## アプリケーション コードと統合する

.NET アプリケーションを Azure SQL Database エンドポイントに接続するには、次の手順に従う必要があります。

1. Visual Studio Code で、ターミナルを開き、次のコマンドを実行して `Microsoft.Data.SqlClient` パッケージをインストールし、新しい .NET コンソール アプリケーションを作成します。

    ```bash
    dotnet new console -n AdventureWorksLTApp
    cd AdventureWorksLTApp 
    dotnet add package Microsoft.Data.SqlClient --version 5.2.1
    ```

1. **Visual Studio Code** で、前の手順で作成した `AdventureWorksLTApp` フォルダーを開きます。

1. 自分のプロジェクトのルート ディレクトリに、`appsettings.json` ファイルを作成します。 この構成ファイルには、データベース接続文字列が格納されます。 接続文字列内の `<your_failover_group>` と `<your_password>` の値を、必ず実際の詳細に置き換えてください。

    ```json
    {
      "ConnectionStrings": {
        "FailoverGroupConnection": "Server=tcp:<your_failover_group>.database.windows.net,1433;Initial Catalog=AdventureWorksLT;Persist Security Info=False;User ID=sqladmin;Password=<your_password>;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
      }
    }
    ```

1. **Visual Studio Code** で `.csproj` ファイルを開き、`</PropertyGroup>` タグのすぐ下に次の内容を追加します。

    ```xml
    <ItemGroup>
        <PackageReference Include="Microsoft.Data.SqlClient" Version="5.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration" Version="6.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="6.0.0" />
    </ItemGroup>
    ```

    完成した `.csproj` ファイルは次のようになるはずです。

    ```xml
    <Project Sdk="Microsoft.NET.Sdk">

      <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>net8.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
      </PropertyGroup>
    
      <ItemGroup>
        <PackageReference Include="Microsoft.Data.SqlClient" Version="5.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration" Version="6.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="6.0.0" />
      </ItemGroup>
    
    </Project>
    ```

1. **Visual Studio Code** で `Program.cs` ファイルを開きます。 エディターで、既存のすべてのコードを次のコードに置き換えます。

    > **注:** 少し時間を取って、コードを見直し、自動フェールオーバー グループ内のプライマリ サーバーとセカンダリ サーバーに関する情報がどのように出力されるかを確認してください。

    ```csharp
    using System;
    using Microsoft.Data.SqlClient;
    using Microsoft.Extensions.Configuration;
    using System.IO;
    
    namespace AdventureWorksLTApp
    {
        class Program
        {
            static void Main(string[] args)
            {
                var configuration = new ConfigurationBuilder()
                    .SetBasePath(Directory.GetCurrentDirectory())
                    .AddJsonFile("appsettings.json")
                    .Build();
    
                string connectionString = configuration.GetConnectionString("FailoverGroupConnection");
    
                ExecuteQuery(connectionString);
            }
    
            static void ExecuteQuery(string connectionString)
            {
                using (SqlConnection connection = new SqlConnection(connectionString))
                {
                    try
                    {
                        connection.Open();
                        string query = @"
                            SELECT 
                                @@SERVERNAME AS [Primary_Server],
                                partner_server AS [Secondary_Server],
                                partner_database AS [Database],
                                replication_state_desc
                            FROM 
                                sys.dm_geo_replication_link_status";
    
                        using (SqlCommand command = new SqlCommand(query, connection))
                        {
                            using (SqlDataReader reader = command.ExecuteReader())
                            {
                                while (reader.Read())
                                {
                                    Console.WriteLine($"Primary Server: {reader["Primary_Server"]}");
                                    Console.WriteLine($"Secondary Server: {reader["Secondary_Server"]}");
                                    Console.WriteLine($"Database: {reader["Database"]}");
                                    Console.WriteLine($"Replication State: {reader["replication_state_desc"]}");
                                }
                            }
                        }
                    }
                    catch (Exception ex)
                    {
                        Console.WriteLine($"Error executing query: {ex.Message}");
                    }
                    finally
                    {
                        connection.Close();
                    }
                }
            }
        }
    }
    ```

1. メニューから **[実行]** > **[デバッグの開始]** の順に選択してコードを実行するか、単に **F5** キーを押します。 上部ツール バーの再生ボタンを選択して、アプリケーションを起動することもできます。

    > **重要:** *"You don't have an extension for debugging C#. Should we find a C# extension in the Marketplace?"* というメッセージが表示された場合は、**C# 開発キット**拡張機能がインストールされていることを確認してください。

1. コードを実行すると、Visual Studio Code の **[デバッグ コンソール]** タブに出力が表示されるはずです。

    ```
    Primary Server: <your_server_name>
    Secondary Server: <your_server_name>
    Database: AdventureWorksLT
    Replication State: CATCH_UP
    ```
    
    レプリケーションの状態 `CATCH_UP` は、データベースがパートナーと完全に同期され、フェールオーバーの準備ができていることを意味します。 レプリケーションの状態を監視すると、パフォーマンスのボトルネックを特定し、データ レプリケーションが効率的に行われるようにするのに役立ちます。

## セカンダリ リージョンにフェールオーバーする

リージョンの障害が原因でプライマリ Azure SQL Database で問題が発生しているシナリオを想像してください。 サービスの継続性を維持し、ダウンタイムを最小限に抑えるには、強制フェールオーバーを実行してアプリケーションをセカンダリ レプリカに切り替える必要があります。

強制フェールオーバー中は、すべての新しい TDS セッションがセカンダリ サーバーに自動的に再ルーティングされ、プライマリ サーバーになります。 最も良い点は、エンドポイントが変わらないので、アプリケーションの接続文字列を変更する必要がないことです。

フェールオーバーを開始し、アプリケーションを実行して、プライマリ サーバーとセカンダリ サーバーの状態を確認しましょう。

1. Azure portal に戻り、Cloud Shell ターミナルの新しいインスタンスを開きます。 次のコードを実行します。 `<your_failover_group>`、`<your_resource_group>`、`<your_primary_server>` の値は、実際の値に置き換えます。 `--server` のパラメーター値は、現在のセカンダリである必要があります。

    ```azurecli
    az sql failover-group set-primary --name <your_failover_group> --resource-group <your_resource_group> --server <your_server_name>
    ```

    > **注**:この操作には数分かかる場合があります。

1. フェールオーバーが完了したら、もう一度アプリケーションを実行してレプリケーションの状態を確認します。 セカンダリ サーバーがプライマリを引き継ぎ、元のプライマリ サーバーがセカンダリになっているはずです。

プライマリとセカンダリのアプリケーション データベースを同じリージョンに配置する理由と、どのような場合に異なるリージョンにすると役に立つのかについて検討してください。

## クリーンアップ

独自のサブスクリプションを使用している場合は、プロジェクトの最後に、作成したリソースがまだ必要かどうかを確認してください。 

リソースを不必要に実行したままにしておくと、追加コストが発生する可能性があります。 [Azure portal](https://portal.azure.com?azure-portal=true) でリソースを個別に削除することも、リソースのセット全体を削除することもできます。

## 詳細

Azure SQL Database の自動フェールオーバー グループの詳細については、「[フェールオーバー グループの概要とベスト プラクティス (Azure SQL データベース)](https://learn.microsoft.com/azure/azure-sql/database/failover-group-sql-db?azure-portal=true)」をご覧ください。
