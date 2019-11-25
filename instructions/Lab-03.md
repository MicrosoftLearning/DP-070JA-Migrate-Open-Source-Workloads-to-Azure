---
lab:
    title: 'オンプレミス PostgreSQL データベースの Azure への移行'
    module: 'モジュール 3: オンプレミスの PostgreSQL を Azure Database for PostgreSQL に移行する'
---

# ラボ: オンプレミス PostgreSQL データベースの Azure への移行

## 概要

このラボでは、このモジュールで学習した情報を使用して、PostgreSQL データベースを Azure に移行します。完全なカバレッジを提供するために、受講者は 2 つの移行を実行します。1 つ目はオフライン移行で、オンプレミスの PostgreSQL データベースを Azure 上で実行されている仮想マシンに転送します。2 つ目はオンライン移行で、仮想マシン上で実行されているデータベースを Azure Database for PostgreSQL に転送します。

また、データベースを使用するサンプル アプリケーションを再構成して実行し、移行のたびにデータベースが正しく動作することを確認する必要があります。

## 目標

このラボを完了すると、次のことができるようになります。

1. オンプレミスの PostgreSQL データベースを Azure 仮想マシンにオフラインで移行します。

1. 2 つ目はオンライン移行で、仮想マシン上で実行されている PostgreSQL データベースを Azure Database for PostgreSQL へオンライン移行します。

## シナリオ

あなたは AdventureWorks 組織のデータベース開発者として働いているとします。AdventureWorks は、10年以上にわたり、エンド コンシューマーやディストリビューターに自転車や自転車部品を直接販売してきました。AdventureWorks のシステムは、オンプレミスのデータセンターにある PostgreSQL で実行されているデータベースに情報を格納します。ハードウェアの合理化の一環として、AdventureWorks はデータベースを Azure に移動したいと考えています。あなたは、この移行の実行を求められました。

初めにあなたは Azure 仮想マシンで実行されている PostgreSQL データベースにデータをすばやく再配置しようと考えました。これは、データベースに変更を加える必要がほとんどないため、リスクの低いアプローチと見なされます。ただし、この方法では、データベースに関連付けられた日次監視および管理タスクのほとんどを引き続き実行することになります。また、AdventureWorks の顧客ベースがどのように変化したかを考慮する必要があります。当初、AdventureWorks は地域の顧客をターゲットにしていましたが、現在は世界的な事業に拡大しています。顧客は世界中に存在するため、データベースにクエリしている顧客が経験する待機時間を最小限にすることが最優先事項となります。他のリージョンにある仮想マシンに PostgreSQL レプリケーションを実装することもできますが、これもまた管理オーバーヘッドです。

代わりに、仮想マシンでシステムを実行したら、データベースをより長期的なソリューションである Azure Database for PostgreSQL に移します。この PaaS 戦略では、システムの保守に関連する作業の多くが削除されます。これで、システムを簡単に拡張し、読み取り専用レプリカを追加して、世界中のどこからでも顧客をサポートできます。さらに、Microsoft は可用性を保証する SLA を提供します。

## セットアップ

Azure に移行するデータを含む既存の PostgreSQL データベースを持つオンプレミス環境があります。ラボを開始する前に、最初の移行のターゲットとして機能する PostgreSQL を実行する Azure 仮想マシンを作成する必要があります。この仮想マシンを作成して構成するスクリプトをここに提供します。このスクリプトをダウンロードして実行するには、次の手順を実行します。

1. 教室環境で実行されている **LON-DEV-01** 仮想マシンにサインインします。ユーザー名は **azureuser**、パスワードは **Pa55w.rd**  です。

2. ブラウザーを使用して、Azure portal にサインインします。

3. Azure Cloud Shell ウィンドウを開く **Bash** シェルが実行していることを確認します。

4. スクリプトとサンプル データベースを保持するリポジトリのクローンを作成します (まだ作成していない場合)。

    ```bash
    git clone https://github.com/MicrosoftLearning/DP-070-Migrate-Open-Source-Workloads-to-Azure ~/workshop
    ```

5. *workshop/migration_samples/setup* フォルダーに移動します。

    ```bash
    cd ~/workshop/migration_samples/setup
    ```

6. *create_postgresql_vm.sh* スクリプトを次のように実行します。リソース グループの名前と、仮想マシンをパラメーターとして保持する場所を指定します。リソース グループがまだ存在しない場合は、この時点で作成されます。*eastus* や *uksouth* など、お近くの場所を指定します。

    ```bash
    bash create_postgresql_vm.sh [resource group name] [location]
    ```

    スクリプトの実行には約 10 分かかります。実行時に多くの出力が生成され、新しい仮想マシンの IP アドレスで終了し、**セットアップ完了** のメッセージが表示されます。IP アドレスをメモしてください。
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

1. 教室環境で実行されている **LON-DEV-01** 仮想マシンにサインインします。ユーザー名は **azureuser**、パスワードは **Pa55w.rd**  です。

    この仮想マシンは、オンプレミス環境をシミュレートします。また、移行する AdventureWorks データベースをホストしている PostgreSQL サーバーを実行しています。

2. 教室環境で実行されている **LON-DEV-01** 仮想マシンで、画面の左側にある **お気に入り** バーで、**pgAdmin4** ユーティリティをクリック します。

3. 「**保存済みパスワードのロック解除**」 ダイアログ ボックスに、パスワード **Pa55w.rd** を入力し、「**OK**」 をクリックします。 

4. **pgAdmin4** ウィンドウで、**サーバー** を展開 し、**LON-DEV-01** を展開し、** データベース** を展開し、**adventureworks** を展開し、 スキーマを展開します。

5. **販売** スキーマで、**テーブル** を展開します。

6. **salesorderheader** テーブルを右クリック し、「**スクリプト**」 をクリックし、「**スクリプトを選択**」 をクリックします。

7. **F5** キーを押してクエリを実行します。31465 行を返す必要があります。

8. 表の一覧 で、「**salesorderdetail**」 テーブルを右クリック し、「**スクリプト**」 をクリックし、「**スクリプトの選択**」 をクリック します。 

9. **F5** キーを押してクエリを実行します。121317 行を返す必要があります。

10. データベース内のさまざまなスキーマ内の他のテーブルのデータを数分間参照します。

### タスク 3: データベースを照会するサンプル アプリケーションを実行する

1. **LON-DEV-01** 仮想マシンでお気に入りバー で、「**ターミナル**」 をクリックしてターミナル ウィンドウを開きます。

1. ターミナル ウィンドウで、デモのサンプル コードをダウンロードします。プロンプトが表示されたら、パスワードを **Pa55w.rd** と入力します。 

   ```bash
   sudo rm -rf ~/workshop
   git clone  https://github.com/MicrosoftLearning/DP-070-Migrate-Open-Source-Workloads-to-Azure ~/workshop
   ```

1. *~/workshop/migration_samples/code/postgresql/AdventureWorksQueries* フォルダーに移動します。

   ```bash
   cd ~/workshop/migration_samples/code/postgresql/AdventureWorksQueries
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

adventureworks データベース内のデータについて理解したら、次に Azure の仮想マシンで実行されている Postgresql サーバーにデータを移行します。この操作は、バックアップ コマンドと復元コマンドを使用してオフライン タスクとして実行します。

> [!NOTE]
> データをオンラインで移行する場合は、オンプレミス データベースから Azure 仮想マシンで実行されているデータベースへのレプリケーションを構成することが可能です。

1. ターミナル ウィンドウから、次のコマンドを実行して *adventureworks* データベースのバックアップを実行します。

    ```bash
    pg_dump adventureworks -U azureuser -Fc > adventureworks_backup.bak
    ```

1. Azure 仮想マシンでターゲット データベースを作成します。[nn.nn.nn.nn]を、この実習ラボのセットアップ段階で作成された仮想マシンの IP アドレスに置き換えます。**azureuser** のパスワードの入力を求められたら、パスワード **Pa55w.rd** を入力 します。

    ```azurecli
    createdb -h [nn.nn.nn.nn] -U azureuser --password adventureworks
    ```

1. pg_restore コマンドを使用して、バックアップを新しいデータベースに復元します。

    ```bash
    pg_restore -d adventureworks -h [nn.nn.nn.nn] -Fc -U azureuser --password adventureworks_backup.bak
    ```

    このコマンドの実行には数分間かかることがあります。

### タスク 3: Azure 仮想マシン上のデータベースを確認する

1. 次のコマンドを実行して、Azure 仮想マシンのデータベースに接続します。仮想マシンで実行されている PostgreSQL サーバーの *azureuser* ユーザーのパスワードは **Pa55w.rd** です。

    ```bash
    psql -h [nn.nn.nn.nn] -U azureuser adventureworks
    ```

1. 次のクエリを実行します。

    ```SQL
    SELECT COUNT(*) FROM sales.salesorderheader;
    ```

    このクエリが 31465 行を返していることを確認します。これは、オンプレミス データベース内の行数と同じです。

1. *sales.salesorderdetail* テーブルの行数を照会します。

    ```SQL
    SELECT COUNT(*) FROM sales.salesorderdetail;
    ```

    このテーブルは 121317 行含まれている必要があります。

1. **\q** コマンドを使用して *psql* ユーティリティーを閉じます。

1. **pgAdmin4** ツールに切り替えます。

1. 左側のペインで、「**サーバー**」 を右クリックし、「**作成**」 をクリックし、「**サーバー**」 をクリックします。 

1. 「**サーバーの作成**」 ダイアログ ボックスで、「**全般**」 タブの 「**名前**」 ボックスに **Virtual Machine** と入力し、「**接続**」 タブをクリックします。

1. 次の詳細を入力し、「**保存**」をクリックします。

    | プロパティ  | 値  |
    |---|---|
    | ホスト名/アドレス | *[nn.nn.nn.nn]* |
    | ポート | 5432 |
    | メンテナンス データベース | postgres |
    | ユーザー名 | azureuser |
    | パスワード | Pa55w.rdDemo |
    | パスワードの保存 | 選択済み |
    | 役割 | *空白のままにする* |
    | サービス | *空白のままにする* |

1. **pgAdmin4** ウィンドウの左側のペインで、「**サーバー**」 の下に **仮想マシン** を展開 します。 

1. **データベース** を展開し、**adventureworks** を展開 し、データベース内のスキーマとテーブルを参照します。テーブルは、オンプレミス データベースのテーブルと同じである必要があります。

### タスク 3: Azure 仮想マシンのデータベースに対してサンプル アプリケーションを再構成してテストする。

1. **ターミナル** ウィンドウに戻 ります。 

1. *ナノ* エディタを使用してテストアプリケーションのApp.configファイルを開きます。

    ```bash
    nano App.config
    ```

1. **ConnectionString** 設定の値を変更 し、**localhost** を Azure 仮想マシンの IP アドレスに置き換えます。ファイルは次のようになります。

    ```XML
    <?xml version="1.0" encoding="utf-8" ?>
    <configuration>
      <appSettings>
        <add key="ConnectionString" value="Server=nn.nn.nn.nn;Database=adventureworks;Port=5432;User Id=azureuser;Password=Pa55w.rd;" />
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

## エクササイズ 2: Azure Database for PostgreSQL へのオンライン移行を実行する

このエクササイズでは、次のタスクを実行します。

1. Azure 仮想マシンで実行されている PostgreSQL サーバーを構成し、スキーマをエクスポートします。
2. Azure Database for PostgreSQL サーバーとデータベースを作成する。
3. スキーマをターゲット データベースにインポートします。
4. Database Migration Service を使用してオンライン移行を実行する。
5. データを変更し、新しいデータベースに一括で移行する
6. Azure Database for PostgreSQL のデータベースを確認する
7. Azure Database for PostgreSQL のデータベースに対してサンプル アプリケーションを再構成してテストする

### タスク 3: Azure 仮想マシンで実行されている PostgreSQL サーバーを構成し、スキーマをエクスポートします。

1. Web ブラウザーを使用して、Azure portal に戻ります。

1. Azure Cloud Shell ウィンドウを開く **Bash** シェルが実行していることを確認します。

1. スクリプトとサンプル データベースを保持するリポジトリのクローンを作成します (まだ作成していない場合)。

    ```bash
    git clone https://github.com/MicrosoftLearning/DP-070-Migrate-Open-Source-Workloads-to-Azure ~/workshop
    ```

1. *~/workshop/migration_samples/setup/postgresql/adventureworks* フォルダーに移動します。

    ```bash
    cd ~/workshop/migration_samples/setup/postgresql/adventureworks

1. PostgreSQL サーバーを実行している Azure 仮想マシンに接続します。次のコマンドを使用して、*[nn.nn.nn.nn]* を仮想マシンの IP アドレスに置き換えます。プロンプトが表示された場合は、パスワード **「Pa55w.rdDemo」** を入力します。

    ```bash
    ssh azureuser@nn.nn.nn.nn
    ```

1. 仮想マシンで、*ルート* アカウントに切り替えます。プロンプトが表示された場合は、*azureuser* ユーザーのパスワードを入力します (**Pa55w.rdDemo**)。

    ```bash
    sudo bash
    ```

1. ディレクトリ */etc/postgresql/10/main* に移動します。 

    ```bash
    cd /etc/postgresql/10/main
    ```

1. *ナノ* エディタを使用して *postgresql.conf* ファイルを開きます。

    ```bash
    nano postgresql.conf
    ```

1. ファイルの一番下までスクロールし、次のパラメーターが構成されていることを確認します。

    ```text
    listen-addresses = '*'
    wal_level = logical
    max_replication_slots = 5
    max_wal_senders = 10
    ```

1. エディタを閉じるには、ESC キーを押してから、Ctrl + X を押します。ファイル保存のプロンプトが表示される場合は、Enter キーを押します。

1. PostgreSQL サービスを再起動します。

    ```bash
    service postgresql restart
    ```

1. Postgres が正しく開始されていることを確認します。

    ```bash
    service postgresql status
    ```

    サービスが実行されている場合、次のようなメッセージが表示されます。

    ```text
     postgresql.service - PostgreSQL RDBMS
      Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
       Active: active (exited) since Fri 2019-08-23 12:47:02 UTC; 2min 3s ago
       Process: 115562 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
      Main PID: 115562 (code=exited, status=0/SUCCESS)

    Aug 23 12:47:02 postgreSQLVM systemd[1]: Starting PostgreSQL RDBMS...
    Aug 23 12:47:02 postgreSQLVM systemd[1]: Started PostgreSQL RDBMS.
    ```

1. *root* アカウントから退出し、*azureuser* アカウントに戻ります。

    ```bash
    exit
    ```

1. bash プロンプトで、次のコマンドを実行して、 **adventureworks** データベースのスキーマを **adventureworks_schema.sql** と名付けられたファイルにエクスポートします。

    ```bash
    pg_dump -o  -d adventureworks -s > adventureworks_schema.sql
    ```

1. 仮想マシンへの接続を終了し、Cloud Shell に戻ります。

    ```bash
    exit
    ```

1. Cloud Shell で、仮想マシンからスキーマ ファイルをコピーします。*[nn.nn.nn.nn]* を仮想マシンの IP アドレスに置き換えます。プロンプトが表示された場合は、パスワード **「Pa55w.rdDemo」** を入力します。

    ```bash
    scp azureuser@nn.nn.nn.nn:~/adventureworks_schema.sql adventureworks_schema.sql
    ```

## タスク 2: Azure Database for PostgreSQL サーバーとデータベースを作成する。

1. Azure portal に切り替えます

2. 「**+ リソースの作成**」 をクリックします。

3. 「**マーケットプレースを検索**」 ボックス に「**Azure Database for PostgreSQL**」と入力し、Enter キーを押します。

4. **Azure Database for PostgreSQL** ページで、「**作成**」 をクリックします。

5. 「**Azure Database for PostgreSQL デプロイ オプションを選択**」 (Select Azure Database for PostgreSQL deployment option) ページにある 「**単一サーバー**」 ボックスで 「**作成**」 をクリック します。

6. 「**単一サーバー**」 ページで、次の詳細を入力し、「**レビュー + 作成**」 をクリックします。

    | プロパティ  | 値  |
    |---|---|
    | リソース グループ | このラボの *セットアップ* タスクの前半で Azure 仮想マシンを作成したときに指定したのと同じリソース グループを使用します。 |
    | サーバー名 | **adventureworks*nnn*** の *nnn* は、サーバー名を一意にするためのサフィックスで、任意に選択していただけます。 |
    | データ ソース | 無し |
    | 管理者ユーザー名 | awadmin |
    | パスワード | Pa55w.rdDemo |
    | パスワードを確定する | Pa55w.rdDemo |
    | 場所 | 該当地域を選択する |
    | バージョン | 10 |
    | コンピューティング + ストレージ:  | 「**サーバーを構成**」 をクリックし、 **「Basic」** 価格レベルを選択し、「**OK**」 をクリックします。 |

7. 「**レビュー + 作成**」 ページで、「**作成**」 をクリックします。次へ進む前に、サービスが作成されるのを待ちます。

8. サービスが作成されたら、ポータルのサービスのページに移動し、「**接続セキュリティ**」 をクリックします。

9. **接続セキュリティ ページ** で、「**Azure サービスへのアクセスを許可する**」 (Allow access to Azure services) を **ON** に設定 します。

10. ファイアウォール ルールの一覧で、**VM** という名前のルールを追加し、以前作成した PostgreSQL サーバーを実行している仮想マシンの IP アドレスに **START IP アドレス** と **END IP アドレス** を設定します。

11. オンプレミス サーバーとして機能する **LON-DEV-01** 仮想マシンが Azure Database for PostgreSQL に接続できるようにするには、「**クライアント IP を追加**」 をクリックします。再構成されたクライアント アプリケーションを実行するときに、後でこのアクセスが必要になります。

12. **保存**し、ファイアウォール ルールが更新されるのを待ちます。

13. Cloud Shell プロンプトで次のコマンドを実行して、Azure Database for PostgreSQL サービスに新しいデータベースを作成します。*[nnn]* を Azure Database for PostgreSQL サービスの作成時に使用したサフィックスに置き換えます。*[resource group]* を、サービス用に指定したリソース グループの名前に置き換えます。

    ```bash
    az postgres db create \
      --name azureadventureworks \
      --server-name adventureworks[nnn] \
      --resource-group [resource group]
    ```

    データベースが正常に作成されると、次のようなメッセージが表示されます。

    ```text
    {
      "charset": "UTF8",
      "collation": "English_United States.1252",
      "id": "/subscriptions/nnnnnnnnnnnnnnnnnnnnnn/resourceGroups/nnnnnnnn/providers/Microsoft.DBforPostgreSQL/servers/adventureworksnnn/databases/azureadventureworks",
      "name": "azureadventureworks",
      "resourceGroup": "nnnnnnnn",
      "type": "Microsoft.DBforPostgreSQL/servers/databases"
    }
    ```

### タスク 3: スキーマをターゲット データベースにインポートする

1. Cloud Shell で、次のコマンドを実行して、azureadventureworks[nnn] サーバーに接続します。*[nnn]* の 2 つのインスタンスをサービスのサフィックスに置き換えます。ユーザー名 には *@adventureworks[nnn]* のサフィックスがあることに注意してください。パスワードプロンプトで、**「Pa55w.rdDemo」** と入力します。

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U awadmin@adventureworks[nnn] -d postgres
    ```

2. 次のコマンドを実行して *azureuser* という名前のユーザーを作成し、このユーザーのパスワードを *Pa55w.rd* に設定します。3 番目のステートメントは、*azureadventureworks* データベースでオブジェクトを作成および管理するために必要な特権を *azureuser* ユーザーに与えます。*azure_pg_admin* ロールを使用すると、*azureuser* ユーザーはデータベースに拡張機能をインストールして使用できるようになります。

    ```SQL
    CREATE ROLE azureuser WITH LOGIN;
    ALTER ROLE azureuser PASSWORD 'Pa55w.rd';
    GRANT ALL PRIVILEGES ON DATABASE azureadventureworks TO azureuser;
    GRANT azure_pg_admin TO azureuser;
    ```

3. **\q** コマンドを使用して *psql* ユーティリティーを閉じます。

4. **adventureworks** データベースのスキーマ を、Azure Database for PostgreSQL サービスで実行されている **azureadventureworks** データベースにインポートします。*azureuser* としてインポートを実行しているので、 プロンプトが表示されたらパスワード **Pa55w.rd** を入力します。

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f adventureworks_schema.sql
    ```

    各項目が作成されると、一連のメッセージが表示されます。スクリプトはエラーなしで完了するはずです。

5. 次のコマンドを実行します。*findkeys.sql* スクリプトは、*dropkeys.sql* という名前の別の SQL スクリプトを生成します。このスクリプトは、**azureadventureworks** データベース内のテーブルからすべての外部キーを削除します。少し待つと、*dropkeys.sql* スクリプトが実行されます。

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f findkeys.sql -o dropkeys.sql -t
    ```

    時間がある場合は、テキストエディタを使用して *dropkeys.sql* スクリプトを調べることも可能です。

6. 次のコマンドを実行します。*createkeys.sql* スクリプトは、すべての外部キーを再作成する *addkeys.sql* という名前の SQL スクリプトを生成します。データベースを移行した後、*addkeys.sql* スクリプトを実行します。

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f createkeys.sql -o addkeys.sql -t
    ```

7. *dropkeys.sql* スクリプトを実行します。 

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f dropkeys.sql
    ```

    外部キーがドロップされると、一連の **ALTER TABLE** メッセージが表示されます。

8. psql ユーティリティをもう一度統計し、*azureadventureworks* データベースに接続します。 

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks
    ```

9. 次のクエリを実行して、残りの外部キーの詳細を検索します。

    ```SQL
    SELECT constraint_type, table_schema, table_name, constraint_name
    FROM information_schema.table_constraints
    WHERE constraint_type = 'FOREIGN KEY';
    ```

    このクエリは、空の結果セットを返すはずです。ただし、外部キーがまだ存在する場合は、外部キーごとに次のコマンドを実行します。

    ```SQL
    ALTER TABLE [table_schema].[table_name] DROP CONSTRAINT [constraint_name];
    ```

10. 残りの外部キーを削除したら、次の SQL ステートメントを実行して、データベースにトリガーを表示します。

    ```bash
    SELECT trigger_name
    FROM information_schema.triggers;
    ```

    このクエリは、データベースにトリガーが含まれていることを示す空の結果セットも返します。データベースにトリガーが含まれている場合は、データを移行する前にトリガーを無効にし、後で再度有効にする必要があります。

11. **\q** コマンドを使用して *psql* ユーティリティーを閉じます。

## タスク 4:  Database Migration Service を使用してオンライン移行を実行する。

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
    | リソース グループを選択する | Azure Database for PostgreSQL サービスと Azure 仮想マシンに使用したのと同じリソース グループを指定する |
    | 場所 | 該当地域を選択する |
    | 仮想ネットワーク | **postgresqlvnet/posgresqlvmSubnet** 仮想ネットワークを選択します。このネットワークは、セットアップの一部として作成されました。 |
    | 定価層 | Premium、4つの vCore を搭載 |

9. Database Migration Service が作成されるまで待ちます。これには数分かかります。

10. Azure portal で、Database Migration Service のページに移動します。

11. 「**新規移行プロジェクト**」 をクリックします。

12. 「**新規移行プロジェクト**」 ページで、次の詳細を入力し、「**アクティビティの作成と実行**」 をクリックします。

    | プロパティ  | 値  |
    |---|---|
    | プロジェクト名 | adventureworks_migration_project |
    | ソース サーバー タイプ | PostgreSQL |
    | Target Database for PostgreSQL | Azure Database for PostgreSQL |
    | アクティビティ タイプの選択 | オンライン データの移行 |

13. **Migration Wizard** が起動したら、「**ソースの詳細の追加**」 (Add Source Details) ページで次の詳細を入力し、「**保存**」 をクリックします。

    | プロパティ  | 値  |
    |---|---|
    | ソース サーバー名 | nn.nn.nn.nn *PostgreSQL を実行している仮想マシンの IP アドレス* |
    | サーバー ポート | 5432 |
    | データ ベース | adventureworks |
    | ユーザー名 | azureuser |
    | パスワード | Pa55w.rdDemo |

14. 「**ターゲットの詳細**」 ページで、次の詳細を入力し、「**保存**」 をクリックします。

    | プロパティ  | 値  |
    |---|---|
    | ターゲット サーバー名 | adventureworks[nnn].postgres.database.azure.com |
    | データ ベース | azureadventureworks |
    | ユーザー名 | azureuser@adventureworks[nnn] |
    | パスワード | Pa55w.rdDemo |

15. 「**ターゲット データベースにマッピングする**」 (Map to target databases) のページで、 **adventureworks** データベースを選択し、**azureadventureworks** にマッピングします。**postgres** データベースを選択解除します。「**保存**」 をクリックします。

16. 「**移行設定**」 ページで、「**adventureworks**」 ドロップダウンを展開し、「**高度なオンライン移行**」 設定のドロップダウンを展開し、 **並列に読み込むインスタンスの最大数** (aximum number of instances to load in parallel) が 5 に設定されていることを確認し、「**保存**」 をクリック します。

17. 「**移行の概要**」 ページで、「**アクティビティ名**」 ボックスに **AdventureWorks_Migration_Activity** と入力し、「**移行の実行**」 をクリックします。

18. 「**AdventureWorks_Migration_Activity**」 ページ で、15 秒間隔で **更新** を選択します。移行操作の進行状況が表示されます。**移行の詳細** 列が **カットオーバー準備完了** に変わるまで待ちます。

19. Cloud Shell に切り替えます。

20. 次のコマンドを実行して、**azureadventureworks** データベースの外部キーを再作成します。前回生成した **addkeys.sql** スクリプトを使用します。

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks -f addkeys.sql
    ```

    外部キーが追加されると、一連の **ALTER TABLE** ステートメントが表示されます。*SpecialOfferProduct* テーブルに関するエラーが表示される場合がありますが、今は無視して問題ありません。これは、正しく転送されていない UNIQUE 制約が原因です。これが現実に起こった場合は、次のクエリを使用して、ソース データベースからこの制約の詳細を取得する必要があります。

    ```SQL
    SELECT constraint_type, table_schema, table_name, constraint_name
    FROM information_schema.table_constraints
    WHERE constraint_type = 'UNIQUE';
    ```

    その後、Azure Database for PostgreSQL のターゲット データベースでこの制約を手動で復元できます。

    他のエラーは発生しないはずです。

## タスク 5: データを変更し、新しいデータベースに一括で移行する

1. Azure portal の **AdventureWorks_Migration_Activity** ページに戻ります。

1. **adventureworks** データベースをクリックします。

1. **adventureworks** ページで、すべてのテーブルのステータスが **完了** としてマークされていることを確認 します。

1. **増分データ同期** をクリックします。すべてのテーブルのステータスが **同期** としてマークされていることを確認します。

1. Cloud Shell に切り替えます。

1. 次のコマンドを実行して、仮想マシン上で PostgreSQL を使用して実行されている **adventureworks** データベースに接続します。

    ```bash
    psql -h nn.nn.nn.nn -U azureuser -d adventureworks
    ```

1. 次の SQL ステートメントを実行して表示し、データベースから 43659、43660、43661 の注文を削除します。データベースが *salesorderheader* テーブルでカスケード削除を実装することに注意してください。これにより、*salesorderdetail* テーブルから対応する行が自動的に削除されます。

    ```SQL
    SELECT * FROM sales.salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
    SELECT * FROM sales.salesorderdetail WHERE salesorderid IN (43659, 43660, 43661);
    DELETE FROM sales.salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
    ```

1. **\q** コマンドを使用して *psql* ユーティリティーを閉じます。

1. Azure portal の **adventureworks** ページに戻り、 *salesorderheader* と *salesorderdetail* テーブルのページにスクロールします。  「**更新**」 をクリックします。  *salesorderheader* テーブルで、3 行が削除され、29 行が **sales.salesorderdetail** テーブルから消えたことを確認 します。

1. 「**カットオーバーを開始**」 をクリックします。

1. 「**カットオーバーの完了**」 ページで、「**確認**」 を選択し、「**適用**」 をクリックします。ステータスが 「**完了**」 に変わるまで待ちます。

1. Cloud Shell に戻ります。

1. 次のコマンドを実行 して、Azure Databasefor PostgreSQL サービスを使用して実行されている **azureadventureworks** データベースに接続します。

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks
    ```

1. 次の SQL ステートメントを実行して、データベースに注文と注文の詳細を表示します。各テーブルの最初のページの後に終了します。これらのクエリの目的は、データが転送されたことを示すことです。

    ```SQL
    SELECT * FROM sales.salesorderheader;
    SELECT * FROM sales.salesorderdetail;
    ```

1. 次の SQL ステートメントを実行して、43659、43660、43661 の注文と詳細を表示します。

    ```SQL
    SELECT * FROM sales.salesorderheader WHERE salesorderid IN (43659, 43660, 43661);
    SELECT * FROM sales.salesorderdetail WHERE salesorderid IN (43659, 43660, 43661);
    ```

    どちらのクエリも 0 行を返す必要があります。

1. **\q** コマンドを使用して *psql* ユーティリティーを閉じます。

### タスク 3: Azure Database for PostgreSQL のデータベースを確認する

1. オンプレミス コンピュータとして機能する仮想マシンに戻る

1. **pgAdmin4** ツールに切り替えます。

1. 左側のペインで、「**サーバー**」 を右クリックし、「**作成**」 をクリックし、「**サーバー**」 をクリックします。

1. 「**サーバーの作成**」 ダイアログ ボックスで、「**全般**」 タブの 「**名前**」 ボックスに **Azure Database for PostgreSQL** と入力し、「**接続**」 タブをクリックします。

1. 次の詳細を入力し、「**SSL**」 タブをクリックします。

    | プロパティ  | 値  |
    |---|---|
    | ホスト名/アドレス | adventureworks*[nnn]*.postgres.database.azure.com |
    | ポート | 5432 |
    | メンテナンス データベース | postgres |
    | ユーザー名 | azureuser@adventureworks*[nnn]* |
    | パスワード | Pa55w.rdDemo |
    | パスワードの保存 | 選択済み |
    | 役割 | *空白のままにする* |
    | サービス | *空白のままにする* |

1. 「**SSL**」 タブ で、**SSL モード** を 「**必須**」 に設定し、「**保存**」 をクリック します。 

1. **pgAdmin4** ウィンドウの左側のペイン で、「**サーバー**」 の下に、**Azure Database for PostgreSQL** を展開します。 

1. **データベース** を展開し、**azureadventureworks** を展開 し、データベース内のテーブルを参照します。テーブルは、オンプレミス データベースのテーブルと同じである必要があります。

### タスク 3: Azure Database for PostgreSQL のデータベースに対してサンプル アプリケーションを再構成してテストする

1. **ターミナル** ウィンドウに戻 ります。 

2. *ナノ* エディタを使用してテストアプリケーションのApp.configファイルを開きます。

    ```bash
    nano App.config
    ```

3. **ConnectionString** 設定の値を変更し、Azure 仮想マシンの IP アドレスを **adventureworks[nnn].postgres.database.azure.com** に置き換えます。  **データベース** を **azureadventureworks** に変更します。**ユーザー ID** を **azureuser@adventureworks[nnn]** に変更します。テキスト **Ssl Mode=Require;** を接続文字列の末端にアペンドします。  ファイルは次のようになります。

    ```XML
    <?xml version="1.0" encoding="utf-8" ?>
    <configuration>
      <appSettings>
        <add key="ConnectionString" value="Server=adventureworks[nnn].postgres.database.azure.com;Database=azureadventureworks;Port=5432;User Id=azureuser@adventureworks[nnn];Password=Pa55w.rd;Ssl Mode=Require;" />
      </appSettings>
    </configuration>
    ```

    これで、アプリケーションは Azure 仮想マシンで実行されているデータベースに接続するはずです。

4. エディタを閉じるには、ESC キーを押してから、Ctrl + X を押します。ファイル保存のプロンプトが表示される場合は、Enter キーを押します。

5. アプリケーションのビルドと実行:

    ```bash
    dotnet run
    ```

    アプリケーションはデータベースに接続しますが、**具体化ビュー "vproductanddescription" が設定されていません** というメッセージが表示され、失敗します。  データベース内の具体化ビューを更新する必要があります。

6. ターミナル ウィンドウ で、*psql* ユーティリティを使用して azureadventureworks データベースに接続します。 

    ```bash
    psql -h adventureworks[nnn].postgres.database.azure.com -U azureuser@adventureworks[nnn] -d azureadventureworks
    ```

7. 次のクエリを実行して、データベース内のすべての具体化ビューを一覧表示します。

    ```SQL
    SELECT schemaname, matviewname, matviewowner, ispopulated
    FROM pg_matviews;
    ```

    このクエリ は、*person.vstateprovincecountryregion* および *production.vproductanddescription* の 2 行を返します。両方のビューの *ispopulated* 列は *f* で、まだ設定されていないことを示します。

8. 次のステートメントを実行して、両方のビューを設定します。

   ```SQL
   REFRESH MATERIALIZED VIEW person.vstateprovincecountryregion;
   REFRESH MATERIALIZED VIEW production.vproductanddescription;
   ```

9.  **\q** コマンドを使用して *psql* ユーティリティーを閉じます。

10. アプリケーションを再度実行します。

    ```bash
    dotnet run
    ```

    アプリケーションが正常に実行されていることを確認します。

    これで、データベースが Azure Database for PostgreSQL に移行され、新しいデータベースを使用するようにアプリケーションが再構成されました。
