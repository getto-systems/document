# database に接続する

- 経緯 : keycloak を本番運用したい
- 目的 : 外部データベースに接続する
- 状況 : 他プロジェクトは全て mysql である
- 決定 : mysql を使用してセットアップする


###### Table of Contents

- [データベースの準備](#user-content-データベースの準備)
- [JDBC connector の用意](#user-content-JDBC connector の用意)
- [standalone-ha.xml の設定](#user-content-standalone-ha.xml の設定)
- [ssl 証明書の設定](#user-content-ssl 証明書の設定)
- [参考資料](#user-content-参考資料)



### データベースの準備

ユーザーを作成する。

```sql
create user 'keycloak_dev'@'%' identified by 'keycloak_dev';
grant all on keycloak_dev.* to 'keycloak_dev'@'%';
```

データベースの文字コードは `utf8` を選択する。
テーブル定義の関係で `utf8mb4` は選択できない。

### JDBC connector の用意

[Download Connector/J | MySQL](https://dev.mysql.com/downloads/connector/j/) にアクセスして jdbc connector をダウンロードする。
「Platform Independent」を選択して zip か tar でダウンロードする。

解凍した `mysql-connector-java-*.jar` を `modules/system/layers/keycloak/com/mysql/main/` に配置する。
`modules/system/layers/keycloak/com/mysql/main/module.xml` を以下の内容で設置する。

```xml
<?xml version="1.0" ?>
<module xmlns="urn:jboss:module:1.3" name="com.mysql">
    <resources>
        <resource-root path="mysql-connector-java-*.jar" />
    </resources>
    <dependencies>
        <module name="javax.api"/>
        <module name="javax.transaction.api"/>
    </dependencies>
</module>
```

### standalone-ha.xml の設定

`standalone/configuration/standalone-ha.xml` の `drivers` に mysql 用の設定を追加する。

```xml
<subsystem xmlns="urn:jboss:domain:datasources:5.0">
    <datasources>
        ...
        <drivers>
            <driver name="mysql" module="com.mysql">
                <driver-class>com.mysql.jdbc.Driver</driver-class>
            </driver>      
        </drivers>
    </datasources>
</subsystem>
```

h2 の方は削除してしまって良い。

続いて datasource の `KeycloakDS` の項目を探して設定する。

```xml
<datasource jndi-name="java:/jboss/datasources/KeycloakDS" pool-name="KeycloakDS" enabled="true">
    <connection-url>jdbc:mysql://localhost:3306/keycloak?useSSL=true&amp;characterEncoding=UTF-8</connection-url>
    <driver>mysql</driver>
    <pool>
        <min-pool-size>5</min-pool-size>
        <max-pool-size>15</max-pool-size>
    </pool>
    <security>
        <user-name>keycloak</user-name>
        <password>keycloak</password>
    </security>
    <validation>
        <valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLValidConnectionChecker"/>
        <validate-on-match>true</validate-on-match>
        <exception-sorter class-name="org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLExceptionSorter"/>
    </validation>
</datasource>
```

`ExampleDS` の方は削除してしまって良い。

本番環境では proxy で接続するので `useSSL=false` で良い。


### ssl 証明書の設定

- 本番環境では proxy で接続するので必要ない

ssl のキー `server-ca.pem`, `client-key.pem`, `client-cert.pem` を用意しておく。

以下のコマンドで `server-ca.pem` から `truststore` ファイルを作成する。

```
keytool -importcert -alias MySQLCACert -file path/to/server-ca.pem -keystore truststore -storepass MySQLCACertPass
```

MySQLCACertPass の部分は適当なパスワードにすること。

さらに、以下のコマンドで `client-key.pem`, `client-cert.pem` から `keystore` ファイルを作成する。

```
openssl pkcs12 -export -in path/to/client-cert.pem -inkey path/to/client-key.pem -name "mysqlclient" -passout pass:mysqlclientpass -out client-keystore.p12

keytool -importkeystore -srckeystore client-keystore.p12 -srcstoretype pkcs12 -srcstorepass mysqlclientpass -destkeystore keystore -deststoretype JKS -deststorepass mysqlclientpass
```

mysqlclientpass の部分は適当なパスワードにすること。

これで `truststore` と `keystore` が用意できた。
以下のオプションで ssl を使用して接続できるようになる。

```
-Djavax.net.ssl.trustStore=path/to/truststore
-Djavax.net.ssl.trustStorePassword=MySQLCACertPass
-Djavax.net.ssl.keyStore=path/to/keystore
-Djavax.net.ssl.keyStorePassword=mysqlclientpass
```

MySQLCACertPass と mysqlclientpass は適宜変更すること。


#### 参考資料

- [Keycloak MySQL Setup](https://stackoverflow.com/questions/49087006/how-to-connect-to-mysql-server-using-ssl-jdbc)
- [Connecting Securely Using SSL | Connector/J](https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-using-ssl.html)
