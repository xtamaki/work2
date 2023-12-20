---
date: "2023-12-12"
lastmod: "2023-12-12"
---

## DGCP共通モジュール
1. TomcatのBaseImageに設定している内容をEmbededTomcat用にカスタマイズしたJarファイルです。  
アプリ開発にEmbededTomcat使用環境での利用を想定した仕様となっています。 

【※想定経路図※】  
DGCP共通モジュールを利用すると、DGCP標準として配布しているスタンドアロンTomcatの既定値と同様の設定が反映されるようになっています。
-  アプリへのメインアクセス経路のポート番号（Tomcat既定値：8080）を31100、（アプリ担当利用想定）管理ポート番号を31300の既定値で設定します。  
-  上記メインアクセス経路のポート、及び管理ポートのconnector設定もDGCP標準値を設定します。（※各設定値の詳細はDGCP共通モジュールのインターフェイス参照）  
  ![想定経路図01](./files/コンテナ想定構成図.png)  


## 提供している主な機能
DGCP共通モジュールを利用すると以下の設定を反映する事が可能となります。  
-  ポート31100、31300をリスンするようにコネクタを設定。  
-  最大スレッド数やKeepAliveタイムアウトをアプリに不具合が出ないよう設定。 
-  アクセスログをJSON形式で出力するように設定。
-  Tomcatのエラーレポートにスタックトレースやサーバ情報を出力しないよう設定
-  一定時間以上スタックしている場合の検知ログ出力するよう設定
-  Proxyを環境変数より標準設定
-  SpringActuatorのinfoエンドポイントを利用し、メタデータを取得し組み込みを行う


## DGCP共通モジュールの組み込み方法について
1. [コンテナ共通モジュール配布ページ](https://scaling-couscous-ovzol19.pages.github.io/)からjarファイルを取得します。  
    
2. Mavenを使用し、項番1で取得したjarファイルをローカルリポジトリに配置します。  
```実行コマンド
set MAVEN_OPTS=-Dhttps.proxyHost=proxy.isid.co.jp  -Dhttp.proxyHost=proxy.isid.co.jp -Dhttp.proxyPort=8080 -Dhttps.proxyPort=8080
mvnw install:install-file -Dfile=e:\dgcputil-1.0.0.jar -Dversion=1.0.0 -DgroupId=jp.co.isid.dgcp -DartifactId=dgcputil  -Dpackaging=jar -DgeneratePom=true
```  

3. アプリのpom.xmlに下記を記載します。
```pom.xml
<dependency>
  <groupId>jp.co.isid.dgcp</groupId>
  <artifactId>dgcputil</artifactId>
  <version>{version}</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

4. @SpringBootApplicationのスキャン対象を変更します。  
```Application.class
  @SpringBootApplication
  ⇒
  @SpringBootApplication(scanBasePackages={"jp.co.isid.{your_application},jp.co.isid.dgcp.springboot"})
```
（例）  
```Application.class
package jp.co.isid.dgcp.test;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication(scanBasePackages={"jp.co.isid.dgcp.test,jp.co.isid.dgcp.springboot"})
public class DgcpTest1Application {

	public static void main(String[] args) {
		SpringApplication.run(DgcpTest1Application.class, args);
	}
}
```


5. アプリ要件により、application.propertiesに共通モジュール設定値のカスタム項目を記載します。    
  ※カスタムの必要となる項目については事項インターフェース説明参照。  
  （例）スレッド数を既定値 30 → 45 に変更する場合
アプリ配下のapplication.propertiesに下記を記載します。
```application.properties
  dgcp.server.tomcat.threads.max=45
```

## DGCP共通モジュールのインターフェイスについて

※カスタム要否について … 該当項目はアプリ要件により、個別に設定を行ってください。  
1. Connector要素  
TomcatのConnector要素に対応したインターフェースです。  
※項番1～6がメインポート、7～12が管理ポートの設定です  

| NO | プロパティ名 | プロパティ既定値 | カスタム要否 | 説明 |  
| ----- | ----- | ----- | ----- | ----- |
| 1 | dgcp.server.port | 31100 | -- | コネクタがサーバソケットを作成し、着信接続を待つTCPポート番号<br> コンテナではサーバーとTomcatが1対1となるので、「31100」を規定値として設定する。 |  
| 2 | dgcp.server.tomcat.max-swallow-size | - | -- | アップロードが中止された場合に Tomcat によって読み込まれるリクエスト本文の最大バイト数 (転送エンコーディングのオーバーヘッドを除く)。 |  
| 3 | dgcp.server.tomcat.max-http-form-post-size | -1 | 要 | コンテナのFORM URLパラメータの解析によって処理されたPOSTの最大サイズ(単位：バイト)。 |  
| 4 | dgcp.server.tomcat.max-http-form-parameter-count | -1 | 要 | クエリ文字列、および POSTリクエストのリクエストパラメータの最大数。 |  
| 5 | dgcp.server.tomcat.threads.max | 30 | 要 | コネクタがリクエスト処理対して作成するスレツドの最大数(最大同時実行数)。<br> 現行のEC2（Tomcat）サーバーの規定値も30であり、特に問題はないと判断してこの値を採用する。<br> またアプリ側で要件があれば変更可能。  |  
| 6 | dgcp.server.tomcat.threads.connection-timeout | 601000 | 要 | コネクタが接続を受け入れた後、要求 URI 行が表示されるまで待機するミリ秒数。 |  
| 7 | dgcp.server.mngport | 31300 | -- | コネクタがサーバソケットを作成し、着信接続を待つTCPポート番号<br> コンテナではサーバーとTomcatが1対1となるので、「31300」を既定値として設定する。 |  
| 8 | dgcp.server.tomcat.mngmax-swallow-size | - | -- | アップロードが中止された場合に Tomcat によって読み込まれるリクエスト本文の最大バイト数 (転送エンコーディングのオーバーヘッドを除く)。 |  
| 9 | dgcp.server.tomcat.mngmax-http-form-post-size | -1 | 要 | コンテナのFORM URLパラメータの解析によって処理されたPOSTの最大サイズ(単位：バイト)。 |  
| 10 | dgcp.server.tomcat.mngmax-http-form-parameter-count | -1 | 要 | クエリ文字列、および POSTリクエストのリクエストパラメータの最大数。 |  
| 11 | dgcp.server.tomcat.threads.vmax | 30 | 要 | コネクタがリクエスト処理対して作成するスレツドの最大数(最大同時実行数)。<br> 現行のEC2（Tomcat）サーバーの規定値も30であり、特に問題はないと判断してこの値を採用する。<br> またアプリ側で要件があれば変更可能。  |  
| 12 | dgcp.server.tomcat.threads.mngconnection-timeout | 601000 | 要 | コネクタが接続を受け入れた後、要求 URI 行が表示されるまで待機するミリ秒数。 |  
| 13 | dgcp.server.tomcat.connector-enabled | 601000 | 要 | 項番1～12の反映ロジックをすべて無効にする設定。<br> アプリ要件で独自のコネクタ設定を用いたい場合に「FALSE」を設定する。 |  


<br><br>
2. Valve要素（AccessLogValve）  
TomcatのValve要素のAccessLogValveに対応したインターフェースです。  

| NO | プロパティ名 | プロパティ既定値 | カスタム要否 | 説明 |  
| ----- | ----- | ----- | ----- | ----- |
| 1 | dgcp.server.tomcat.accesslog.enabled | TRUE | -- | アクセスログの標準出力に伴う設定。 |  
| 2 | dgcp.server.tomcat.accesslog.directory | /dev/stdout | -- | アクセスログの標準出力に伴う設定。 |  
| 3 | dgcp.server.tomcat.accesslog.pattern | {"logName":"tomcat_access", "remoteHost":"%h", "remoteUser":"%u", "timestamp":"%{begin:yyyy-MM-dd HH:mm:ss}t", "request":"%r", "httpStatus":"%s", "bodyBytesSent":"%b", "requestTime":"%T", "http-x-forwarded-for":"%{X-Forwarded-For}i"} | -- | アクセスログの標準出力に伴う設定。 |  
| 4 | dgcp.server.tomcat.accesslog.request-attributes-enabled  | TRUE | -- | アクセスログの標準出力に伴う設定。 |  
| 5 | dgcp.server.tomcat.accesslog.rotate | TRUE | -- | アクセスログの標準出力に伴う設定。 |  
| 6 | dgcp.server.tomcat.valve-accesslog-enabled | TRUE | 要 | 項番1～5の反映ロジックをすべて無効にする設定。<br> 開発環境などでのデバッグ時に一時的にログ出力を無効にした場合に「FALSE」を設定する。<br>  ※本番環境では必ずTRUEにしてください※ |  


<br><br>
3. Valve要素（ErrorReportValve）  
TomcatのValve要素のErrorReportValveに対応したインターフェースです。  

| NO | プロパティ名 | プロパティ既定値 | カスタム要否 | 説明 |  
| ----- | ----- | ----- | ----- | ----- |
| 1 | dgcp.server.tomcat.valve-errorreport-enabled | TRUE | -- | セキュリティーに伴うエラーレポートの表示設定。<br> 既定値ではセキュリティの観点からエラーレポートにスタックトレースやサーバ情報を出力しない設定としている。 |


<br><br>
4. Valve要素（StuckThreadDetectionValve）  
TomcatのValve要素のStuckThreadDetectionValveに対応したインターフェースです。  

| NO | プロパティ名 | プロパティ既定値 | カスタム要否 | 説明 |  
| ----- | ----- | ----- | ----- | ----- |
| 1 | dgcp.server.tomcat.valve-stuckthread-enabled | TRUE | -- | ログ管理（一定時間以上スタックしている場合の検知ログ出力）設定。<br> 既定値で出力する設定とするため、出力不要な場合に「FALSE」を設定する。 |


<br><br>

## さらなるDGCP共通モジュールのカスタマイズについて
DGCP共通モジュールまたはspring Bootのapplication.propertiesで対応していない設定を行うには
DGCP共通モジュールのクラスを継承したロジックを実装する事で対応可能となります。

※ロジック例を記載。。。
