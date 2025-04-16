---
lab:
  title: Azure SQL Database での開発用にデータをインポートまたはエクスポートする
  module: Import and export data for development in Azure SQL Database
---

# Azure SQL Database での開発用にデータをインポートまたはエクスポートする

この演習では、データを外部 REST エンドポイントからインポートし (Azure Static Web App を使用してシミュレート)、Azure 関数を使用してエクスポートします。 このラボでは、REST API と Azure Functions を統合してデータのインポート/エクスポート操作を処理することに重点を置いて、開発目的で Azure SQL Database を操作する際の実用的な経験を提供します。

## 前提条件

このラボを開始する前に、以下を必ず取得してください。

- リソースを作成および管理するためのアクセス許可があるアクティブな Azure サブスクリプション。
- Azure SQL Database、REST API、Azure Functions に関する基本的な知識。
- 次の拡張機能がインストールされた Visual Studio Code:
      - Azure Functions 拡張機能。
- リポジトリを複製するための Git。
- データベースを管理するための SQL Server Management Studio (SSMS) または Azure Data Studio。

## 環境を設定する

まず、Azure SQL Database やデータのインポートとエクスポートに必要なツールなど、このラボに必要なリソースを設定します。

### Azure SQL Database の作成

この手順では、Azure にデータベースを作成します。

1. Azure portal で、**[SQL データベース]** ページに移動します。
1. **［作成］** を選択します
1. 必須フィールドに入力します。

    | 設定 | Value |
    |---|---|
    | 無料のサーバーレス オファー | オファーの適用 |
    | サブスクリプション | 該当するサブスクリプション |
    | リソース グループ | 新しいリソース グループを選択または作成します |
    | データベース名 | **MyDB** |
    | [サーバー] | 新しいサーバーを選択または作成します |
    | 認証方法 | SQL 認証 |
    | サーバー管理者のログイン | **sqladmin** |
    | Password | セキュリティで保護されたパスワードを入力します |
    | [パスワードの確認入力] | パスワードを確認します |

1. **[確認および作成]** 、 **[作成]** の順に選択します。
1. デプロイが完了したら、***[Azure SQL サーバー]*** (Azure SQL Database ではありません) の **[ネットワーク]** セクションに移動します。
    1. IP アドレスをファイアウォール規則に追加します。 これにより、SQL Server Management Studio (SSMS) または Azure Data Studio を使用してデータベースを管理できるようになります。
    1. **[Azure サービスおよびリソースにこのサーバーへのアクセスを許可する]** チェックボックスをオンにします。 これにより、Azure 関数アプリからデータベース サーバーにアクセスできるようになります。
    1. 変更を保存。
1. **[Azure SQL サーバー]** の **[Microsoft Entra ID]** セクションに移動し、**[このサーバーの Microsoft Entra 専用認証をサポートする]** を必ず*選択解除*し、選択した場合は変更を **[保存]** します。 この例では SQL 認証を使用するため、Entra のみのサポートを無効にする必要があります。

> [!NOTE]
> 運用環境では、どの種類のアクセスに、どこからのアクセスを許可するかを決定する必要があります。 Entra 認証のみを選択した場合、関数は若干変更されますが、Azure 関数アプリがサーバーにアクセスできるようにするには、*[Azure サービスおよびリソースにこのサーバーへのアクセスを許可する]* を有効にする必要があることに注目してください。

### GitHub リポジトリをクローンする

1. **Visual Studio Code** を開きます。

1. GitHub リポジトリをクローンし、プロジェクトの準備をします。

    1. **Visual Studio Code** で、**Ctrl + Shift + P キー** (Windows) または **Cmd + Shift + P キー** (Mac) を押して、**コマンド パレット**を開きます。
    1. 「**Git: Clone**」と入力し、**[Git: Clone]** を選択します。
    1. プロンプトで、次の URL を入力してリポジトリをクローンします。
        ```bash
        https://github.com/MicrosoftLearning/mslearn-sql-dev.git
        ```

    1. リポジトリをクローンする宛先フォルダーを選択します。

### JSON データ用の Azure Blob Storage を設定する

次に、**employees.json** ファイルをホストするように **Azure Blob Storage** を設定します。 Azure portal および **Visual Studio Code** 内で次の手順に従います。

まず、Azure ストレージ アカウントを作成します。

1. **Azure portal** で、**[ストレージ アカウント]** ページに移動します。
1. **［作成］** を選択します
1. 必須フィールドに入力します。

    | 設定 | 値 |
    |---|---|
    | サブスクリプション | 該当するサブスクリプション |
    | リソース グループ | 新しいリソース グループを選択または作成します |
    | ストレージ アカウント名 | グローバル一意識別子を選択します |
    | リージョン | 自分に最も近いリージョンを選択します |
    | プライマリ サービス | **Azure Blob Storage または Azure Data Lake Storage Gen2** |
    | パフォーマンス | Standard |
    | 冗長性 | ローカル冗長ストレージ (LRS) |

1. **[確認および作成]** 、 **[作成]** の順に選択します。
1. ストレージ アカウントが作成されるまで待ちます。

アカウントが作成されたので、**employees.json** を Blob Storage にアップロードしましょう。

1. Azure portal の **[ストレージ アカウント]** ページに移動します。
1. 使うストレージ アカウントを選びます。
1. **[コンテナー]** セクションに移動します。
1. **jsonfiles** という名前の新しいコンテナーを作成します。
1. コンテナー内で、**[アップロード]** をクリックしてクローンされたディレクトリの **/Allfiles/Labs/04/blob-storage** にある **employees.json** ファイルをアップロードします。

ファイルへの匿名アクセスを許可することもできますが、ここでは、アクセスをセキュリティで保護するために、このファイルの *Shared Access Signature (SAS)* を生成してみましょう。

1. **jsonfiles** コンテナーで、**employees.json** ファイルを選択します。
1. ファイルのコンテキスト メニューから **[SAS の生成]** を選択します。
1. 設定を確認し、**[SAS トークンおよび URL を生成]** を選択します。
1. Blob SAS トークンと Blob SAS URL が生成されます。 次の手順で使用するために **Blob SAS トークン**と **Blob SAS URL** をコピーします。 そのウィンドウを閉じると、二度とトークン値にアクセスできなくなります。

これで、**employees.json** ファイルにアクセスするための、セキュリティで保護された URL が作成されました。それでは、次に進んでテストしましょう。

1. 新しいブラウザー タブを開いて、**Blob SAS URL** を貼り付けます。
1. **employees.json** ファイルの内容がブラウザーに表示され、それは次のようになるはずです。

    ```json
    {
        "employees": [
            {
                "EmployeeID": 1,
                "FirstName": "John",
                "LastName": "Doe",
                "Department": "HR"
            },
            {
                "EmployeeID": 2,
                "FirstName": "Jane",
                "LastName": "Smith",
                "Department": "Engineering"
            }
        ]
    }
    ```

### Blob Storage から Azure SQL Database にデータをインポートする

これで、Azure Blob Storage でホストされている **employees.json** ファイルから Azure SQL Database にデータをインポートする準備ができました。

まず、Azure SQL Database で**マスター キー**と**データベース スコープ資格情報**を作成する必要があります。

1. **SQL Server Management Studio** (SSMS) または **Azure Data Studio** を使用して Azure SQL Database に接続します。
1. *まだ作成していない場合*は、次の SQL コマンドを実行してマスター キーを作成します。

    ```sql
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'YourStrongPassword!';
    ```

1. 次に、次の SQL コマンドを実行して、Azure Blob Storage にアクセスするための**データベース スコープ資格情報**を作成します。

    ```sql
    CREATE DATABASE SCOPED CREDENTIAL MyBlobCredential
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE', 
    SECRET = '<your-sas-token>';
    ```

    \<your-sas-token\> を先ほど生成された Blob SAS トークンに置き換えます。

1. 最後に、Azure Blob Storage にアクセスするための**データ ソース**が必要です。 次の SQL コマンドを実行して、**データ ソース**を作成します。

    ```sql
    CREATE EXTERNAL DATA SOURCE MyBlobDataSource
    WITH (
        TYPE = BLOB_STORAGE,
        LOCATION = 'https://<your-storage-account-name>.blob.core.windows.net',
        CREDENTIAL = MyBlobCredential
    );
    ```

    \<your-storage-account-name\> を Azure ストレージ アカウントの名前に置き換えます。

これで、**employees.json** ファイルから *Azure SQL Database* にデータをインポートするようにすべてが設定されました。

次の SQL コマンドを使用して、*Azure Blob Storage* でホストされている**employees.json** ファイルからデータをインポートします。

```sql
SELECT EmployeeID
    , FirstName
    , LastName
    , Department
INTO dbo.employee_data
FROM OPENROWSET(
    BULK 'jsonfiles/employees.json',
    DATA_SOURCE = 'MyBlobDataSource',
    SINGLE_CLOB
) AS JSONData
CROSS APPLY OPENJSON(JSONData.BulkColumn, '$.employees')
WITH (
    EmployeeID INT '$.EmployeeID',
    FirstName NVARCHAR(50) '$.FirstName',
    LastName NVARCHAR(50) '$.LastName',
    Department NVARCHAR(50) '$.Department'
) AS EmployeeData;
```

このコマンドは、*Azure Blob Storage* の **jsonfiles** コンテナーから **employees.json** ファイルを読み取り、Azure SQL Database の **employee_data** テーブルにデータをインポートします。

これで、次の SQL コマンドを実行して、データ インポートを検証することができます。 

```sql
SELECT * FROM dbo.employee_data;
```

**employee_data** テーブルにインポートされた **employees.json** ファイルのデータが表示されるはずです。

---

## Azure 関数アプリを使用してデータをエクスポートする

ラボのこの部分では、C# で Azure 関数アプリを作成し、Azure SQL Database からデータをエクスポートします。 この関数は、データを取得し、そのデータを JSON 応答として返します。

### Visual Studio Code で Azure 関数アプリを作成する

まず、Visual Studio Code で Azure 関数アプリを作成します。

1. **Visual Studio Code** を開きます。
1. エクスプローラー ウィンドウで、**/Allfiles/Labs/04/azure-functions** フォルダーに移動します。
1. **azure-functions** フォルダーを右クリックし、**[統合ターミナルで開く]** を選択します。
1. VS Code ターミナルで、次のコマンドを使用して Azure にログインします。

    ```bash
    az login
    ```

1. (任意) サブスクリプションが複数ある場合は、アクティブなサブスクリプションを設定します。

    ```bash
    az account set --subscription <your-subscription-id>
    ```

1. 次のコマンドを実行して Azure 関数アプリを作成します。

    ```bash
    $functionappname = "YourUniqueFunctionAppName"
    $resourcegroup = "YourResourceGroupName"
    $location = "YourLocation"
    # NOTE - The following should be a new storage account name where your Azure function will resided.
    # It should not be the same Storage Account name used to store the JSON file
    $storageaccount = "YourStorageAccountName"

    az storage account create --name $storageaccount --location $location --resource-group $resourcegroup --sku Standard_LRS
    
    az functionapp create --resource-group $resourcegroup --consumption-plan-location $location --runtime dotnet --name  $functionappname --os-type Linux --storage-account $storageaccount --functions-version 4
    
    ```

    ***プレースホルダーは自分の値に置き換えます。json ファイルに使用されるストレージ アカウント名は使用しないでください。このスクリプトでは、Azure 関数アプリを格納する新しいストレージ アカウントを作成する必要があります***。


### Visual Studio Code で新しい関数アプリを作成する

Visual Studio Code で新しい関数を作成して、Azure SQL Database からデータをエクスポートしましょう。

まだ行っていない場合は、Visual Studio Code に Azure Functions 拡張機能を追加することが必要になる場合があります。 拡張機能ペインで **Azure Functions** を検索してインストールすることで、これを行うことができます。

1. Visual Studio Code で、**Ctrl + Shift + P** キー (Windows) または **Cmd + Shift + P** キー (Mac) を押してコマンド パレットを開きます。
1. 「**Azure Functions: Create New Project**」を入力して選択します。
1. **Function App** ディレクトリを選択します。 GitHub クローン リポジトリの **/Allfiles/Labs/04/azure-functions** フォルダーを選択します。
1. 言語として **[C#]** を選択します。
1. ランタイムとして **[.Net 8.0 LTS]** を選択します。
1. テンプレートとして **[HTTP trigger]** を選択します。
1. 関数 **ExportDataFunction** を呼び出します。
1. 名前空間を **Contoso.ExportFunction** にします。
1. 関数に**匿名**アクセス レベルを付与します。

### データをエクスポートするための C# コードを記述する

1. Azure 関数アプリでは、最初にいくつかのパッケージをインストールすることが必要になる場合があります。 次のコマンドを実行することで、それらをインストールできます。

    ```bash
    dotnet add package Microsoft.Data.SqlClient
    dotnet add package Newtonsoft.Json
    dotnet restore
    
    npm install -g azure-functions-core-tools@4 --unsafe-perm true

    ```

1. プレースホルダー関数コードを次の C# コードに置き換えて、Azure SQL Database にクエリを実行し、結果を JSON として返します。

    ```csharp
    using System;
    using System.IO;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Azure.WebJobs;
    using Microsoft.Azure.WebJobs.Extensions.Http;
    using Microsoft.AspNetCore.Http;
    using Microsoft.Extensions.Logging;
    using Microsoft.Data.SqlClient;
    using Newtonsoft.Json;
    using System.Collections.Generic;
    using System.Threading.Tasks;
    
    public static class ExportDataFunction
    {
        [FunctionName("ExportDataFunction")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "get", Route = null)] HttpRequest req,
            ILogger log)
        {
            // Connection string to the database
            // NOTE: REPLACE THIS CONNECTION STRING WITH THE CONNECTION STRING OF YOUR AZURE SQL DATABASE
            string connectionString = "Server=tcp:yourserver.database.windows.net;Database=DataLabDatabase;User ID=youruserid;Password=yourpassword;Encrypt=True;";
            
            // List to hold employee data
            List<Employee> employees = new List<Employee>();
            
            try
            {
                // Establishing connection to the database
                using (SqlConnection conn = new SqlConnection(connectionString))
                {
                    await conn.OpenAsync();
                    var query = "SELECT EmployeeID, FirstName, LastName, Department FROM employee_data";
                    
                    // Executing the query
                    using (SqlCommand cmd = new SqlCommand(query, conn))
                    {
                        // Adding parameters to the query (if needed)
                        // cmd.Parameters.AddWithValue("@ParameterName", parameterValue);

                        using (SqlDataReader reader = await cmd.ExecuteReaderAsync())
                        {
                            // Reading data from the database
                            while (await reader.ReadAsync())
                            {
                                employees.Add(new Employee
                                {
                                    EmployeeID = (int)reader["EmployeeID"],
                                    FirstName = reader["FirstName"].ToString(),
                                    LastName = reader["LastName"].ToString(),
                                    Department = reader["Department"].ToString()
                                });
                            }
                        }
                    }
                }
            }
            catch (SqlException ex)
            {
                // Logging SQL errors
                log.LogError($"SQL Error: {ex.Message}");
                return new StatusCodeResult(500);
            }
            catch (System.Exception ex)
            {
                // Logging unexpected errors
                log.LogError($"Unexpected Error: {ex.Message}");
                return new StatusCodeResult(500);
            }
    
            // Returning the list of employees as a JSON response
            return new OkObjectResult(JsonConvert.SerializeObject(employees, Formatting.Indented));
        }
    
        // Employee class to hold employee data
        public class Employee
        {
            public int EmployeeID { get; set; }
            public string FirstName { get; set; }
            public string LastName { get; set; }
            public string Department { get; set; }
        }
    }
    ```

    *必ず **connectionString** は Azure SQL Database への接続文字列に置き換え、接続文字列には sqladmin パスワードも入力してください。*

    > **注:** 運用環境では、必要な IP アドレスのみにアクセスを制限します。 さらに、SQL 認証ではなく、Azure 関数アプリのマネージド ID を使用して、データベースにアクセスすることを検討してください。 詳しくは、「[Azure SQL 用の Microsoft Entra でのマネージド ID](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true)」をご覧ください。

1. 関数コードを保存し、オブジェクトを JSON にシリアル化するための **Newtonsoft.Json** パッケージが **.csproj** ファイルに含まれていることを確認します。 含まれていない場合は、追加します。

    ```xml
    <PackageReference Include="Newtonsoft.Json" Version="13.X.X" />
    ```

次は Azure 関数アプリを Azure にデプロイします。

### Azure 関数アプリを Azure にデプロイする

1. **Visual Studio Code** 統合ターミナルで、次のコマンドを実行して Azure 関数を Azure にデプロイします。

    ```bash
    func azure functionapp publish <your-function-app-name>
    ```

    ***<your-function-app-name>*** を Azure 関数アプリの名前に置き換えます。

1. デプロイが完了するまで待ちます。

### Azure 関数アプリの URL を取得する

1. Azure portal を開き、Azure 関数アプリに移動します。
1. *[概要]* セクションの *[関数]* タブに、関数が一覧表示され、それを選択します。
1. **[コードとテスト]** タブで、**[関数の URL の取得]** を選択します。
1. **[既定 (ファンクション キー)]** をコピーします。それはすぐに必要になります。 この URL は次のようになります。
   
   ```url
   https://YourFunctionAppName.azurewebsites.net/api/ExportDataFunction?code=2pjO0HqRyz_13DHQg8ga-ysdDWbDU_eHdtlixbAHLVEGAzFuplomUg%3D%3D
   ```

### Azure 関数アプリをテストする

1. デプロイが完了したら、Visual Studio Code ターミナルから先ほどコピーしたファンクション キーのURL に HTTP 要求を送信することで、関数をテストできます。

    ```bash
    curl https://<your-function-app-name>.azurewebsites.net/api/ExportDataFunction?code=<the function key embedded to your function URL>
    ```

1. 応答には、***employee_data*** テーブルからエクスポートされたデータが JSON 形式で含まれている必要があります。

この関数は簡単な例ですが、データのフィルター処理、並べ替え、集計など、より複雑なロジックやデータ処理を含むように拡張できます。  コードを拡張して、エラー処理、ログ、セキュリティなどの機能を含めることもできます。

### リソースのクリーンアップ

ラボを完了したら、この演習で作成したリソースを削除して、追加コストが発生しないようにすることができます。

- Azure SQL Database を削除します。
- Azure ストレージ アカウントを削除します。
- Azure 関数アプリを削除します。
- リソースを含むリソース グループを削除します。
