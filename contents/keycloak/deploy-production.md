# 本番環境にデプロイする

- 経緯 : 開発環境で keycloak の動作確認が完了した
- 目的 : keycloak を本番環境で稼働させる
- 状況 : GKE にクラスタは用意済み
- 決定 : GKE の k8s クラスタに keycloak をデプロイする


###### Table of Contents

- [proxy 経由のアクセス用設定](#user-content-proxy 経由のアクセス用設定)
- [jboss クラスタのディスカバリー](#user-content-jboss クラスタのディスカバリー)
- [deployment](#user-content-deployment)
- [Reference](#user-content-Reference)


### proxy 経由のアクセス用設定

GKE で k8s クラスタにデプロイするので、keycloak にはクラスタ経由でアクセスすることになる。
このため、`X-Forwarded-For` などからアクセス元の情報を取得するように設定する必要がある。

```xml
<subsystem xmlns="urn:jboss:domain:undertow:6.0">
  <buffer-cache name="default"/>
  <server name="default-server">
    <ajp-listener name="ajp" socket-binding="ajp"/>
    <http-listener name="default" socket-binding="http" redirect-socket="https" proxy-address-forwarding="true" redirect-socket="proxy-https"/>
    ...
  </server>
  ...
</subsystem>
```

subsystem `urn:jboss:domain:undertow:6.0` を探して `http-listener` の設定を変更する。

- `proxy-address-forwarding="true"` : `X-Forwarded-For` などから情報を取得
- `redirect-socket="proxy-https"` : リダイレクトするときのポートを指定

さらに `redirect-socket="proxy-https"` の設定で参照される `socket-binding` を追加する。

```xml
<socket-binding-group name="standard-sockets" default-interface="public" port-offset="${jboss.socket.binding.port-offset:0}">
  ...
  <socket-binding name="proxy-https" port="443"/>
  ...
</socket-binding-group>
```

ただ、ログに記載される `ipAddress` がクライアントのものではなくロードバランサーのものになってしまう・・・（未解決）


### jboss クラスタのディスカバリー

pod は問題なく起動するが、このままではログイン後にリダイレクトループになった。
おそらく、セッションを保存している infinispan がうまく参照できないため、セッションが保存されているノードにアクセスした時はログイン後画面にリダイレクトして、そうでないときはログイン前画面にリダイレクトして、を繰り返したのだと思う。

これを解決するために、クラスタに参加しているノードを発見するためのディスカバリーを追加する必要がある。

まず、下記 extension がロードされているか確認しておく。

```xml
<extension module="org.jboss.as.clustering.infinispan"/>
<extension module="org.jboss.as.clustering.jgroups"/>
```

`modcluster` extension は必要ないのでコメントアウトして良い。
その時は `modcluster` subsystem もあわせて削除する必要があるかも。


#### データソースの設定

```xml
<datasource jndi-name="java:jboss/datasources/ClusteringDS" pool-name="ClusteringDS" enabled="true" use-java-context="true" use-ccm="true">
  <connection-url>${env.DB_CONNECTIONURL}&amp;characterEncoding=UTF-8</connection-url>
  <driver>mysql</driver>
  <security>
    <user-name>${env.DB_USERNAME}</user-name>
    <password>${env.DB_PASSWORD}</password>
  </security>
  <validation>
    <valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLValidConnectionChecker"/>
    <validate-on-match>true</validate-on-match>
    <exception-sorter class-name="org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLExceptionSorter"/>
  </validation>
</datasource>
```

mysql を使用する設定。
`DB_CONNECTIONURL` とかはよしなに設定する。
データベースの作成等も忘れずに。


#### jgroups subsystem の設定

```xml
<subsystem xmlns="urn:jboss:domain:jgroups:6.0">
  <channels default="ee">
    <channel name="ee" stack="tcp"/>
  </channels>
  <stacks>
    <stack name="tcp">
      <transport type="TCP" socket-binding="jgroups-tcp">
        <property name="external_addr">${env.HOST_IP}</property>
        <property name="use_ip_addrs">true</property>
      </transport>
      <protocol type="org.jgroups.protocols.JDBC_PING">
        <property name="datasource_jndi_name">java:jboss/datasources/ClusteringDS</property>

        <property name="initialize_sql">
          CREATE TABLE IF NOT EXISTS JGROUPSPING (
            own_addr varchar(200) NOT NULL,
            bind_addr varchar(200) NOT NULL,
            created timestamp NOT NULL,
            cluster_name varchar(200) NOT NULL,
            ping_data BLOB,
            PRIMARY KEY (own_addr, cluster_name)
          )
        </property>
        <property name="insert_single_sql">
          INSERT INTO JGROUPSPING (own_addr, bind_addr, created, cluster_name, ping_data) values (?,'${env.HOST_IP}',NOW(), ?, ?)
        </property>
        <property name="delete_single_sql">
          DELETE FROM JGROUPSPING WHERE own_addr=? AND cluster_name=?
        </property>
        <property name="select_all_pingdata_sql">
          SELECT ping_data FROM JGROUPSPING WHERE cluster_name=?
        </property>
      </protocol>
      <protocol type="MERGE3"/>
      <protocol type="FD_SOCK"/>
      <protocol type="FD_ALL"/>
      <protocol type="VERIFY_SUSPECT"/>
      <protocol type="pbcast.NAKACK2"/>
      <protocol type="UNICAST3"/>
      <protocol type="pbcast.STABLE"/>
      <protocol type="pbcast.GMS">
        <property name="print_physical_addrs">true</property>
        <property name="print_local_addr">true</property>
      </protocol>
      <protocol type="MFC"/>
      <protocol type="FRAG3"/>
    </stack>
  </stacks>
</subsystem>
```

GKE の k8s クラスタにデプロイするため、UDP を用いたディスカバリは使用できない。
TCP でノードを見つけて、データベースに記録する、という手法でノードの管理を行うもの。

さらに `socket-binding` の `jgroups-tcp` の設定を `public` に変更する。

```xml
<socket-binding name="jgroups-tcp" interface="public" port="7600"/>
```


#### deployment

```yaml
- name: keycloak
  image: gcr.io/getto-union/github.com/getto-systems/getto-keycloak:0.9.3
  livenessProbe:
    failureThreshold: 8
    initialDelaySeconds: 120
    periodSeconds: 60
    httpGet:
      path: /auth/
      port: 8080
      scheme: HTTP
  readinessProbe:
    failureThreshold: 8
    initialDelaySeconds: 120
    periodSeconds: 60
    httpGet:
      path: /auth/
      port: 8080
      scheme: HTTP
  resources:
    limits:
      cpu: "300m"
  ports:
  - containerPort: 8080
  env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: getto-keycloak-db
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: getto-keycloak-db
          key: password
    - name: HOST_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
```

こんな設定でデプロイする。
`fieldRef` を使用するためには `apiVersion: extensions/v1beta1` を指定する必要がある。

起動には３分くらいかかるので、`livenessProbe` と `readinessProbe` を設定しておく。


### Reference

- [How-to deploy a Highly Available JBoss cluster on Kubernetes with dynamic node discovery | Medium](https://medium.com/@ltearno/how-to-deploy-a-highly-available-jboss-cluster-on-kubernetes-with-dynamic-node-discovery-part-1-23cc6cede88c)
