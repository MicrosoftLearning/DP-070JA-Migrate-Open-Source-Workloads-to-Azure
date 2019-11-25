---
lab:
    title: 'オンプレミス MySQL データベースの Azure への移行'
    module: 'モジュール 2: オンプレミスの MySQL を Azure Database for MySQL に移行する'
---

# ラボ: オンプレミス MySQL データベースの Azure への移行

## 概要

このラボでは、このモジュールで学習した情報を使用して、MySQL データベースを Azure に移行します。完全なカバレッジを提供するために、受講者は 2 つの移行を実行します。1 つ目はオフライン移行で、オンプレミスの MySQL データベースを Azure 上で実行されている仮想マシンに転送します。2 つ目はオンライン移行で、仮想マシン上で実行されているデータベースを Azure Database for MySQL に転送します。

また、データベースを使用するサンプル アプリケーションを再構成して実行し、移行のたびにデータベースが正しく動作することを確認する必要があります。

## 目標

このラボを完了すると、次のことができるようになります。

1. オンプレミスの MySQL データベースを Azure 仮想マシンにオフラインで移行します。

1. 2 つ目はオンライン移行で、仮想マシン上で実行されているデータベースを Azure Database for MySQL へオンライン移行します。

## シナリオ

あなたは AdventureWorks 組織のデータベース開発者として働いているとします。AdventureWorks は、10年以上にわたり、エンド コンシューマーやディストリビューターに自転車や自転車部品を直接販売してきました。AdventureWorks のシステムは、オンプレミスのデータセンターにある MySQL で実行されているデータベースに情報を格納します。ハードウェアの合理化の一環として、AdventureWorks はデータベースを Azure に移動したいと考えています。あなたは、この移行の実行を求められました。

初めにあなたは Azure 仮想マシンで実行されている MySQL データベースにデータをすばやく再配置しようと考えました。これは、データベースに変更を加える必要がほとんどないため、リスクの低いアプローチと見なされます。ただし、この方法では、データベースに関連付けられた日次監視および管理タスクのほとんどを引き続き実行することになります。また、AdventureWorks の顧客ベースがどのように変化したかを考慮する必要があります。当初、AdventureWorks は地域の顧客をターゲットにしていましたが、現在は世界的な事業に拡大しています。顧客は世界中に存在するため、データベースにクエリしている顧客が経験する待機時間を最小限にすることが最優先事項となります。他のリージョンにある仮想マシンに MySQL レプリケーションを実装することもできますが、これもまた管理オーバーヘッドです。

代わりに、仮想マシンでシステムを実行したら、データベースをより長期的なソリューションである Azure Database for MySQL に移します。この PaaS 戦略では、システムの保守に関連する作業の多くが削除されます。これで、システムを簡単に拡張し、読み取り専用レプリカを追加して、世界中のどこからでも顧客をサポートできます。さらに、Microsoft は可用性を保証する SLA を提供します。

## セットアップ

Azure に移行するデータを含む既存の MySQL データベースを持つオンプレミス環境があります。ラボを開始する前に、最初の移行のターゲットとして機能する MySQL を実行する Azure 仮想マシンを作成する必要があります。この仮想マシンを作成して構成するスクリプトをここに提供します。このスクリプトをダウンロードして実行するには、次の手順を実行します。

1. 教室環境で実行されている **LON-DEV-01** 仮想マシンにサインインします。ユーザー名は **azureuser**、パスワードは **Pa55w.rd**  です。

    この仮想マシンは、オンプレミス環境をシミュレートします。また、移行する AdventureWorks データベースをホストしている MySQL サーバーを実行しています。

2. ブラウザーを使用して、Azure portal にサインインします。

3. Azure Cloud Shell ウィンドウを開く **Bash** シェルが実行していることを確認します。

4. スクリプトとサンプル データベースを保持するリポジトリのクローンを作成します (まだ作成していない場合)。

    ```bash
    git clone https://github.com/MicrosoftLearning/DP-070-Migrate-Open-Source-Workloads-to-Azure workshop 
    ```

5. *migration_samples/setup* フォルダーに移動します。

    ```bash
    cd ~/workshop/migration_samples/setup
    ```

6. *create_mysql_vm.sh* スクリプトを次のように実行します。リソース グループの名前と、仮想マシンをパラメーターとして保持する場所を指定します。リソース グループがまだ存在しない場合は、この時点で作成されます。*eastus* や *uksouth* など、お近くの場所を指定します。

    ```bash
    bash create_mysql_vm.sh 「resource group name」 「location」
    ```

    スクリプトの実行には約 10 分かかります。実行時に多くの出力が生成され、新しい仮想マシンの IP アドレスで終了し、**セットアップ完了** のメッセージが表示されます。 
7. IP アドレスをメモしてください。

> [!NOTE]
> この IP アドレスは演習に必要です。

## エクササイズ 1: オンプレミス データベースを Azure 仮想マシンに移行する

このエクササイズでは、次のタスクを実行します。

1. オンプレミス データベースを確認する。
2. データベースを照会するサンプル アプリケーションを実行する。
3. データベースを Azure 仮想マシンにオフラインで移行する。
4. Azure 仮想マシン上のデータベースを確認する。
5. Azure 仮想マシンのデータベースに対してサンプル アプリケーションを再構成してテストする。

### タスク 3: オンプレミス データベースを確認する

1. 教室環境で実行されている **LON-DEV-01** 仮想マシンで、画面の左側にある **お気に入り** バーで、**MySQLWorkbench** をクリック します。 

1. **MySQLWorkbench** ウィンドウで、「**LON-DEV-01**」 をクリックし、「**OK**」をクリックします。

1. **adventureworks** を展開し、**テーブル** を展開 します。

1. **コンタクト** テーブルを右クリック し、「**1000 行制限を選択する**」 をクリックし、**コンタクト** クエリ ウィンドウで 「**1000 行に制限する**」 をクリック し、「**制限しない**」 をクリックします。

1. 「**実行**」 をクリックしてクエリを実行します。  19972 行を返す必要があります。

1. テーブルの一覧 で、「**従業員**」 テーブルを右クリック し、「**行の選択**」 をクリック します。

1. 「**実行**」 をクリックしてクエリを実行します。290 行を返す必要があります。

1. データベース内のさまざまなテーブル内の他のテーブルのデータを数分間参照します。

### タスク 3: データベースを照会するサンプル アプリケーションを実行する

1. **LON-DEV-01** 仮想マシンでお気に入りバー で、「**ターミナル**」 をクリックしてターミナル ウィンドウを開きます。

1. ターミナル ウィンドウで、ラボのサンプル コードをダウンロードします。プロンプトが表示されたら、パスワードを **Pa55w.rd** と入力します。 

   ```bash
   sudo rm -rf ~/workshop
   git clone  https://github.com/MicrosoftLearning/DP-070-Migrate-Open-Source-Workloads-to-Azure ~/workshop
   ```

1. *~/workshop/migration_samples/code/mysql/AdventureWorksQueries* フォルダーに移動します。

   ```bash
   cd ~/workshop/migration_samples/code/mysql/AdventureWorksQueries
   ```

    このフォルダーには、*adventureworks* データベース内の複数のテーブルの行数をカウントするクエリを実行するサンプル アプリが含まれています。

1. クライアント アプリを実行します。

    ```bash
    dotnet run
    ```

    アプリは、次の出力を生成する必要があります。

    ```text
    Querying AdventureWorks database
    SELECT COUNT(*) FROM production.vproductanddescription
    1764

    SELECT COUNT(*) FROM purchasing.vendor
    104

    SELECT COUNT(*) FROM sales.specialoffer
    16

    SELECT COUNT(*) FROM sales.salesorderheader
    31465

    SELECT COUNT(*) FROM sales.salesorderdetail
    121317

    SELECT COUNT(*) FROM person.person
    19972
    ```

### タスク 3: データベースを Azure 仮想マシンにオフラインで移行する

adventureworks データベース内のデータについて理解したら、次に Azure の仮想マシンで実行されている MySQL サーバーにデータを移行します。この操作は、バックアップ コマンドと復元コマンドを使用してオフライン タスクとして実行します。

> [!NOTE]
> データをオンラインで移行する場合は、オンプレミス データベースから Azure 仮想マシンで実行されているデータベースへのレプリケーションを構成することが可能です。

1. ターミナル ウィンドウから、次のコマンドを実行して *adventureworks* データベースのバックアップを実行します。  LON-DEV-01 仮想マシン上の MySQL サーバーは、ポート 3306 を使用していることに注意してください。

    ```bash
    mysqldump -u azureuser -pPa55w.rd adventureworks > aw_mysql_backup.sql
    ```

1. Azure Cloud Shell を使用して、MySQL サーバーとデータベースを含む仮想マシンに接続します。\<*nn.nn.nn.nn*\> を仮想マシンの IP アドレスに置き換えます。続行するかどうかを確認するメッセージが表示されたら、「**yes**」と入力し、Enter キーを押します。 

    ```bash
    ssh azureuser@nn.nn.nn.nn
    ```

1. **「Pa55w.rdDemo」** と入力し、Enter キーを押します。
1. MySQL サーバーに接続します。

    ```bash
    mysql -u azureuser -pPa55w.rd
    ```

1. Azure 仮想マシンでターゲット データベースを作成します。

    ```azurecli
    create database adventureworks;
    ```

1. MySQL を終了します。

    ```bash
    quit
    ```

1. SSH セッションを終了します。

    ```bash
    exit
    ```

1. mysql コマンドを使用して、バックアップを新しいデータベースに復元します。

    ```bash
    mysql -h [nn.nn.nn.nn] -u azureuser -pPa55w.rd adventureworks < aw_mysql_backup.sql
    ```

    このコマンドの実行には数分かかります。

### タスク 3: Azure 仮想マシン上のデータベースを確認する

1. 次のコマンドを実行して、Azure 仮想マシンのデータベースに接続します。仮想マシンで実行されている MySQL サーバーの *azureuser* ユーザーのパスワードは **Pa55w.rd** です。

    ```bash
    mysql -h [nn.nn.nn.nn] -u azureuser -pPa55w.rd adventureworks
    ```

1. 次のクエリを実行します。

    ```SQL
    SELECT COUNT(*) FROM specialoffer;
    ```

    このクエリが 16 行を返していることを確認します。これは、オンプレミス データベース内の行数と同じです。

1. *sales.vendor* テーブルの行数を照会します。

    ```SQL
    SELECT COUNT(*) FROM sales.vendor;
    ```

    このテーブルは 104 行含まれている必要があります。

1. **quit** コマンドを使用して *mysql* ユーティリティーを閉 じます。
1. **MySQL Workbench** ツールに切り替えます。
1. 「**データベース**」 メニューで、「**接続の管理**」 をクリックし、「**新規**」 をクリック します。 
1. 「**接続**」 タブをクリックします。
1. **接続名** タイプの **MySQL on Azure** で、
1. 次の詳細を入力し、「**テスト接続**」 をクリックします。

    | プロパティ  | 値  |
    |---|---|
    | Hostname | [nn.nn.nn.nn] |
    | ポート | 3306 |
    | ユーザー名 | azureuser |
    | 既定のスキーマ | adventureworks |

1. 「**パスワード**」 に **Pa55w.rd** と入力 し、「**OK**」 をクリックします。 
1. 「**OK**」 を クリックし、「**閉じる**」 をクリックします。
1. **MySQL on Azure** をクリックします。
1. **adventureworks** では、データベース内のテーブルを参照します。  テーブルは、オンプレミス データベースのテーブルと同じである必要があります。

### タスク 3: Azure 仮想マシンのデータベースに対してサンプル アプリケーションを再構成してテストする。

1. **ターミナル** ウィンドウに戻 ります。 

1. *ナノ* エディタを使用してテストアプリケーションのApp.configファイルを開きます。 

    ```bash
    nano App.config
    ```

1. **ConnectionString** 設定の値を変更 し、**127.0.0.1** を Azure 仮想マシンの IP アドレスに置き換えます。  ファイルは次のようになります。

    ```XML
    <?xml version="1.0" encoding="utf-8" ?>
    <configuration>
    <appSettings>
        <add key="ConnectionString" value="Server=nn.nn.nn.nn;Database=adventureworks;Uid=azureuser;Pwd=Pa55w.rd;" />
    </appSettings>
    </configuration>
    ```

    これで、アプリケーションは Azure 仮想マシンで実行されているデータベースに接続するはずです。

1. エディタを閉じるには、ESC キーを押してから、Ctrl + X を押します。ファイル保存のプロンプトが表示される場合は、Enter キーを押します。

1. アプリケーションのビルドと実行:

    ```bash
    dotnet run
    ```

    アプリケーションが正常に実行され、各テーブルの行数が以前と同じであることを確認します。

    これで、オンプレミス データベースを Azure 仮想マシンに移行し、新しいデータベースを使用するようにアプリケーションを再構成しました。

## エクササイズ 2: Azure Database for MySQL へのオンライン移行を実行する

このエクササイズでは、次のタスクを実行します。

1. Azure 仮想マシンで実行されている MySQL サーバーを構成し、スキーマをエクスポートします。
2. Azure Database for MySQL サーバーとデータベースを作成する。
3. スキーマをターゲット データベースにインポートします。
4. Database Migration Service を使用してオンライン移行を実行する。
5. データを変更し、新しいデータベースに一括で移行する
6. Azure Database for MySQL のデータベースを確認する
7. Azure Database for MySQL のデータベースに対してサンプル アプリケーションを再構成してテストします。

### タスク 3: Azure 仮想マシンで実行されている MySQL サーバーを構成し、スキーマをエクスポートする

1. Web ブラウザーを使用して、Azure portal に戻ります。

1. Azure Cloud Shell ウィンドウを開く **Bash** シェルが実行していることを確認します。

1. MySQL サーバーを実行している仮想マシンに戻ります。次のコマンドを使用して、*[nn.nn.nn.nn]* を仮想マシンの IP アドレスに置き換えます。プロンプトが表示された場合は、パスワード **「Pa55w.rdDemo」** を入力します。

    ```bash
    ssh azureuser@nn.nn.nn.nn
    ```

1. MySQL が正しく開始されていることを確認します。

    ```bash
    service mysql status
    ```

    サービスが実行されている場合、次のようなメッセージが表示されます。

    ```text
         mysql.service - MySQL Community Server
           Loaded: loaded (/lib/systemd/system/mysql.service; enabled; vendor preset: enabled)
           Active: active (running) since Mon 2019-09-02 14:45:42 UTC; 21h ago
          Process: 15329 ExecStart=/usr/sbin/mysqld --daemonize --pid-file=/run/mysqld/mysqld.pid (code=exited, status=0/SUCCESS)
          Process: 15306 ExecStartPre=/usr/share/mysql/mysql-systemd-start pre (code=exited, status=0/SUCCESS)
         Main PID: 15331 (mysqld)
            Tasks: 30 (limit: 4070)
           CGroup: /system.slice/mysql.service
                   └─15331 /usr/sbin/mysqld --daemonize --pid-file=/run/mysqld/mysqld.pid
        
            Sep 02 14:45:41 mysqlvm systemd[1]: Starting MySQL Community Server...
            Sep 02 14:45:42 mysqlvm systemd[1]: Started MySQL Community Server.
    ```

1. mysqldump ユーティリティを使用してソース データベースのスキーマをエクスポートします。

    ```bash
    mysqldump -u azureuser -pPa55w.rd adventureworks --no-data > adventureworks_mysql_schema.sql
    ```

1. bash プロンプトで、次のコマンドを実行して、 **adventureworks** データベースのスキーマを **adventureworks_mysql.sql** と名付けられたファイルにエクスポートします。

    ```bash
    mysqldump -u azureuser -pPa55w.rd adventureworks > adventureworks_mysql.sql
    ```

1. 仮想マシンへの接続を終了し、Cloud Shell に戻ります。

    ```bash
    exit
    ```

1. Cloud Shell で、仮想マシンからスキーマ ファイルをコピーします。*[nn.nn.nn.nn]* を仮想マシンの IP アドレスに置き換えます。プロンプトが表示された場合は、パスワード **「Pa55w.rdDemo」** を入力します。

    ```bash
    scp azureuser@nn.nn.nn.nn:~/adventureworks_mysql_schema.sql adventureworks_mysql_schema.sql
    ```
### タスク 2: Azure Database for MySQL サーバーとデータベースを作成する

1. Azure portal に切り替えます

1. 「**リソースの作成**」 をクリックします。

1. 「**マーケットプレースを検索**」 ボックスに、**Azure Database for MySQL** と入力し、Enter キーを押します。

1. **Azure Database for MySQL** ページで、「**作成**」 をクリックします。

1. 「**MySQL サーバーの作成**」 ページで、次の詳細を入力 し、「**レビュー + 作成**」 をクリックします。

    | プロパティ  | 値  |
    |---|---|
    | リソース グループ | このラボの *セットアップ* タスクの前半で Azure 仮想マシンを作成したときに指定したのと同じリソース グループを使用します。  |
    | サーバー名 | **adventureworks*nnn*** の *nnn* は、サーバー名を一意にするためのサフィックスで、任意に選択していただけます。 |
    | データ ソース | 無し |
    | 管理者ユーザー名 | awadmin |
    | パスワード | Pa55w.rdDemo |
    | パスワードを確定する | Pa55w.rdDemo |
    | 場所 | 該当地域を選択する |
    | バージョン | 5.7 |
    | コンピューティング + ストレージ:  | 「**サーバーを構成**」 をクリックし、 **「Basic」** 価格レベルを選択し、「**OK**」 をクリックします。 |

1. 「**レビュー + 作成**」 ページで、「**作成**」 をクリックします。次へ進む前に、サービスが作成されるのを待ちます。

1. サービスが作成されたら、ポータルのサービスのページに移動し、「**接続セキュリティ**」 をクリックします。


1. **接続セキュリティ ページ** で、「**Azure サービスへのアクセスを許可する**」 (Allow access to Azure services) を **ON** に設定 します。

1. ファイアウォール ルールの一覧で、**VM** という名前のルールを追加し、MySQL サーバーを実行している仮想マシンの IP アドレスに **START IP アドレス** と **END IP アドレス** を設定します。

1. オンプレミス サーバーとして機能する **LON-DEV-01** 仮想マシンが Azure Database for MySQL に接続できるようにするには、「**クライアント IP を追加**」 をクリックします。再構成されたクライアント アプリケーションを実行するときに、後でこのアクセスが必要になります。

1. **保存**し、ファイアウォール ルールが更新されるのを待ちます。

1. Cloud Shell プロンプトで、次のコマンドを実行して、Azure Database for MySQL サービスに新しいデータベースを作成します。*[nnn]* を Azure Database for MySQL サービスの作成時に使用したサフィックスに置き換えます。*リソース グループ* を、サービス用に作成したリソース グループの名前に置き換えます。

    ```bash
    az MySQL db create \
    --name azureadventureworks \
    --server-name adventureworks[nnn] \
    --resource-group [resource group]
    ```

    データベースが正常に作成されると、次のようなメッセージが表示されます。

    ```text
    {
          "charset": "latin1",
          "collation": "latin1_swedish_ci",
          "id": "/subscriptions/nnnnnnnnnnnnnnnnnnnnnnnnnnnnn/resourceGroups/nnnnnn/providers/Microsoft.DBforMySQL/servers/adventureworksnnnn/databases/azureadventureworks",
          "name": "azureadventureworks",
          "resourceGroup": "nnnnn",
          "type": "Microsoft.DBforMySQL/servers/databases"
    }
    ```

### タスク 3: スキーマをターゲット データベースにインポートする

1. Cloud Shell で、次のコマンドを実行して、azureadventureworks[nnn] サーバーに接続します。*[nnn]* の 2 つのインスタンスをサービスのサフィックスに置き換えます。ユーザー名 には *adventureworks[nnn]* のサフィックスがあることに注意してください。パスワードプロンプトで、**「Pa55w.rdDemo」** と入力します。

    ```bash
    mysql -h adventureworks[nnn].MySQL.database.azure.com -u awadmin@adventureworks[nnn] -pPa55w.rdDemo
    ```

1. 次のコマンドを実行して *azureuser* という名前のユーザーを作成し、このユーザーのパスワードを *Pa55w.rd* に設定します。2 番目のステートメントは、*azureadventureworks* データベースでオブジェクトを作成するために必要な特権を *azureuser* ユーザーに与えます。 

    ```SQL
    GRANT SELECT ON *.* TO 'azureuser'@'localhost' IDENTIFIED BY 'Pa55w.rd';
    GRANT CREATE ON *.* TO 'azureuser'@'localhost';
    ```

1. 次のコマンドを実行して、*adventureworks* データベースを作成します。

    ```SQL
    CREATE DATABASE adventureworks;
    ```

1. **quit** コマンドを使用して *mysql* ユーティリティーを閉 じます。

1. **adventureworks** スキーマを Azure Database for MySQL サービスにインポートします。  *azureuser* としてインポートを実行しているので、 プロンプトが表示されたらパスワード **Pa55w.rd** を入力します。

    ```bash
    mysql -h adventureworks「nnnn」.MySQL.database.azure.com -u awadmin@adventureworks[nnn] -pPa55w.rdDemo adventureworks < adventureworks_mysql_schema.sql
    ```

### タスク 4:  Database Migration Service を使用してオンライン移行を実行する。

1. Azure portal に再ログインします。

2. 「**すべてのサービス**」 をクリックし、「**サブスクリプション**」 をクリックし、サブスクリプションをクリックします。

3. 「サブスクリプション」 ページで 「**設定**」 下の 「**リソース プロバイダー**」 をクリックします。

4. 「**名前でフィルターする**」 ボックスに「**DataMigration**」と入力 し、「**Microsoft.DataMigration**」 をクリックします。

5. **Microsoft.DataMigration** が登録されていない場合は、「**登録**」 をクリックし、**ステータス** が 「**登録済み**」 に変わるまで待ちます。変更状態を確認するには、「**更新**」 をクリックします。

6. 「**マーケットプレースの検索**」 ボックスの 「**Azure Database Migration Service**」 で、「**リソースを作成**」 をクリックし、Enter を押します。

7. 「**Azure Database Migration Service**」 ページで、「**作成**」 をクリック します。

8. 「**移行サービスの作成**」 ページで、次の詳細を入力し、「**作成**」 をクリックします。

    | プロパティ  | 値  |
    |---|---|
    | サービス名 | adventureworks_migration_service |
    | サブスクリプション | 自分のサブスクリプションを選択する |
    | リソース グループを選択する | Azure Database for MySQL サービスと Azure 仮想マシンに使用したのと同じリソース グループを指定する |
    | 場所 | 該当地域を選択する |
    | 仮想ネットワーク |  **MySQLvnet/mysqlvmSubnet** 仮想ネットワークを選択します。  このネットワークは、セットアップの一部として作成されました。 |
    | 定価層 | Premium、4つの vCore を搭載 |

9. Database Migration Service が作成されるまで待ちます。これには数分かかります。

10. Azure portal で、Database Migration Service のページに移動します。

11. 「**新規移行プロジェクト**」 をクリックします。

12. 「**新規移行プロジェクト**」 ページで、次の詳細を入力し、「**アクティビティの作成と実行**」 をクリックします。

    | プロパティ  | 値  |
    |---|---|
    | プロジェクト名 | adventureworks_migration_project |
    | ソース サーバー タイプ | MySQL |
    | Target Database for MySQL | Azure Database for MySQL |
    | アクティビティ タイプの選択 | オンライン データの移行 |

13. **Migration Wizard** が起動したら、「**ソースの詳細の追加**」 (Add Source Details) ページで次の詳細を入力し、「**保存**」 をクリックします。

    | プロパティ  | 値  |
    |---|---|
    | ソース サーバー名 | nn.nn.nn.nn *MySQL を実行している仮想マシンの IP アドレス* |
    | サーバー ポート | 3306 |
    | ユーザー名 | azureuser |
    | パスワード | Pa55w.rdDemo |

14. 「**ターゲットの詳細**」 ページで、次の詳細を入力し、「**保存**」 をクリックします。

    | プロパティ  | 値  |
    |---|---|
    | ターゲット サーバー名 | adventureworks[nnn].MySQL.database.azure.com |
    | ユーザー名 | awadmin@adventureworks[nnn] |
    | パスワード | Pa55w.rdDemo |

15. 「**ターゲット データベースにマッピングする**」 (Map to target databases) のページで、 **adventureworks** データベースを選択し、**adventureworks** にマッピングします。「**保存**」 をクリックします。

16. 「**移行設定**」 ページで、「**adventureworks**」 ドロップダウンを展開し、「**高度なオンライン移行**」 設定のドロップダウンを展開し、 **並列に読み込むインスタンスの最大数** (aximum number of instances to load in parallel) が 5 に設定されていることを確認し、「**保存**」 をクリック します。

17. 「**移行の概要**」 ページで、「**アクティビティ名**」 ボックスに **AdventureWorks_Migration_Activity** と入力し、「**移行の実行**」 をクリックします。

18. 「**AdventureWorks_Migration_Activity**」 ページ で、15 秒間隔で **更新** を選択します。移行操作の進行状況が表示されます。**移行の詳細** 列が **カットオーバー準備完了** に変わるまで待ちます。


### タスク 5: データを変更し、新しいデータベースに一括で移行する

1. Azure portal の **AdventureWorks_Migration_Activity** ページに戻ります。

1. **adventureworks** データベースをクリックします。

1. **adventureworks** ページで、すべてのテーブルのステータスが **完了** としてマークされていることを確認 します。

1. **増分データ同期** をクリックします。すべてのテーブルのステータスが **同期** としてマークされていることを確認します。

1. Cloud Shell に切り替えます。

1. 次のコマンドを実行して、仮想マシン上で MySQL を使用して実行されている **adventureworks** データベースに接続します。

    ```bash
    mysql -h nn.nn.nn.nn -u azureuser -pPa55w.rd adventureworks
    ```

1. 次の SQL ステートメントを実行して表示し、データベースから 43659、43660、43661 の注文を削除します。データベースが *salesorderheader* テーブルでカスケード削除を実装することに注意してください。これにより、*salesorderdetail* テーブルから対応する行が自動的に削除されます。 

    ```SQL
    SELECT * FROM salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
    SELECT * FROM salesorderdetail WHERE salesorderid IN (43659, 43660, 43661);
    DELETE FROM salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
    ```

1. **quit** コマンドを使用して *mysql* ユーティリティーを閉 じます。

2. Azure portal の **adventureworks** ページに戻り、「**更新**」 をクリックします。*salesorderheader* と *salesorderdetail* のページにスクロールします。*salesorderheader* テーブルで、3 行が削除され、29 行が **sales.salesorderdetail** テーブルから消えたことを確認 します。更新プログラムが適用されていない場合は、データベースに **保留中の変更** があることを確認します。

1. 「**カットオーバーを開始**」 をクリックします。

1. 「**カットオーバーの完了**」 ページで、「**確認**」 を選択し、「**適用**」 をクリックします。ステータスが 「**完了**」 に変わるまで待ちます。

1. Cloud Shell に戻ります。

1. 次のコマンドを実行 して、Azure Databasefor MySQL サービスを使用して実行されている **azureadventureworks** データベースに接続します。

    ```bash
    mysql -h adventureworks[nnn].MySQL.database.azure.com -u awadmin@adventureworks[nnn] -pPa55w.rdDemo adventureworks
    ```

1. 次の SQL ステートメントを実行して、43659、43660、43661 の注文と詳細を表示します。これらのクエリの目的は、データが転送されたことを示すことです。

    ```SQL
    SELECT * FROM salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
    SELECT * FROM salesorderdetail WHERE salesorderid IN (43659, 43660, 43661);
    ```

    最初のクエリは 3 行を返します。2 番目のクエリは 29 行を返します。

1. **quit** コマンドを使用して *mysql* ユーティリティーを閉 じます。

### タスク 3: Azure Database for MySQL のデータベースを確認する

1. オンプレミス コンピュータとして機能する仮想マシンに戻る

1. **MySQL Workbench** ツールに切り替えます。
1. 「**データベース**」 メニューで、「**データベースに接続**」 をクリックします。 

1. 次の詳細を入力し、「**OK**」をクリックします。

    | プロパティ  | 値  |
    |---|---|
    | Hostname | adventureworks*[nnn]*.MySQL.database.azure.com |
    | ポート | 3306 |
    | ユーザー名 | awadmin@adventureworks*[nnn]* |
    | パスワード | Pa55w.rdDemo |

1. **データベース** を展開し、**adventureworks** を展開 し、データベース内のテーブルを参照します。テーブルは、オンプレミス データベースのテーブルと同じである必要があります。

### タスク 3: Azure Database for MySQL のデータベースに対してサンプル アプリケーションを再構成してテストする

1. **ターミナル** ウィンドウに戻 ります。 
1. *code/mysql/AdventureWorksQueries* フォルダーに移動します。

   ```bash
   cd code/mysql/AdventureWorksQueries
   ```

1. コード エディタを使用して App.config ファイルを開きます。

    ```bash
    code App.config
    ```

1. **ConnectionString** 設定の値を変更し、Azure 仮想マシンの IP アドレスを **adventureworks[nnn].MySQL.database.azure.com** に置き換えます。**ユーザー ID** を **awadmin@adventureworks[nnn]** に変更します。**パスワード** を **「Pa55w.rdDemo」** に変更します。ファイルは次のようになります。

    ```XML
    <?xml version="1.0" encoding="utf-8" ?>
        <configuration>
          <appSettings>
            <add key="ConnectionString" value="Server=adventureworks[nnn].MySQL.database.azure.com;Database=adventureworks;Port=3306;User Id=awadmin@adventureworks[nnn];Password=Pa55w.rdDemo" />
          </appSettings>
        </configuration>
    ```

    これで、アプリケーションは Azure 仮想マシンで実行されているデータベースに接続するはずです。

1. ファイルを保存し、エディタを閉じます。

1. アプリケーションのビルドと実行:

    ```bash
    dotnet run
    ```

   これで、データベースが Azure Database for MySQL に移行され、新しいデータベースを使用するようにアプリケーションが再構成されました。
