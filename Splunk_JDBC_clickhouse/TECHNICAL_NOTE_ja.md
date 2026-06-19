# Technical Note: Splunk DB Connect 向け ClickHouse JDBC ドライバーの導入に関する技術的課題と解決策

本ドキュメントは、ClickHouse JDBC ドライバーを Splunk DB Connect に組み込む際に発生した技術的な問題と、それを解決するために適用した回避策（ワークアラウンド）の詳細を記録したものです。

## 1. 発生した問題
Splunk DB Connect の仕様上、公式の ClickHouse JDBC ドライバー（例: `clickhouse-jdbc-0.9.8.jar` とその依存ライブラリ群）をそのまま `lib/dbxdrivers/` ディレクトリに配置するだけでは、正常に動作しませんでした。

具体的には、Splunk のタスクサーバー（Task Server）起動時や接続テスト時に以下のようなエラーが発生し、ドライバーのロードに失敗します。
- **`java.lang.NoClassDefFoundError: org/slf4j/LoggerFactory`**
- **`SecurityException: Invalid signature file digest for Manifest main attributes`**

## 2. 根本原因 (Root Cause)
1. **クラスローダーの依存関係分断 (Classloader Isolation Issues):**
   Splunk DB Connect は、`lib/dbxdrivers/` 内の JAR ファイルを独自のクラスローダーで動的に読み込みます。ClickHouse ドライバー本体と、依存するライブラリ（SLF4J, LZ4, Jackson など）が別々の JAR ファイルに分かれていると、クラスパスの解決順序や名前空間の衝突により `NoClassDefFoundError` が発生します。
2. **署名検証エラー (Signature Verification Failure):**
   複数の JAR ファイルが干渉し合う問題を防ぐために JAR の中身を結合・編集しようとした場合、JAR 内部の `META-INF/` に含まれる署名ファイル（`.SF`, `.RSA`, `.DSA`）のダイジェスト値と中身が不一致となり、Javaのセキュリティ機構によってロードが拒否されます。
3. **冗長なファイルの存在:**
   `lib/dbxdrivers/` 内に複数のバージョンの JAR や、役割の被る JAR（`all.jar` と `dependencies.jar` など）が混在していると、Splunk 側が誤ったクラスを読み込んでしまい致命的なエラーを引き起こします。

## 3. 適用したパッチ・回避策 (Workaround / Fixes)
これらの問題を回避するため、本アドオンでは以下の対応を行っています。

### A. Shaded (Fat) JAR の単一配置
別々の依存 JAR を配置するアプローチを放棄し、すべての依存関係（SLF4J等）が1つのファイル内にカプセル化（Shaded）された **`clickhouse-jdbc-all-x.x.x.jar`** アーティファクトを Maven Central から直接取得して採用しました。
これにより、Splunk DB Connect のクラスローダーが外部の依存関係を探しに行く必要がなくなり、`NoClassDefFoundError` が完全に解消されました。

### B. ディレクトリの厳密なクリーンアップ
`lib/dbxdrivers/` ディレクトリには **必ず1つの JAR ファイルのみ** を配置する仕様としました。
過去の検証時に存在した `clickhouse-jdbc-xxx-dependencies.jar` などの不要なファイルはすべて削除し、クラスの重複や署名の競合が発生しないようにクリーンアップを徹底しました。

### C. Splunk DB Connect UI バリデーションの突破
JDBCのロード問題とは別に、Splunk DB Connect の UI 側で Connection を保存する際、「The connection is invalid」というエラーで保存が弾かれる問題がありました。
これはドライバーの問題ではなく Splunk 側の仕様であり、`db_connection_types.conf` に以下の2つの必須属性を追加することでパッチを当てています。
- `testQuery = SELECT 1` (接続時の死活監視用クエリ)
- `port = 443` (デフォルトポートの明示)

### D. Splunk Cloud でのポート互換性
本アドオンを **Splunk Cloud** にデプロイする場合、非標準ポート（`8443` など）への送信接続はデフォルトでブロックされます。Splunk Cloud の送信ファイアウォールは、`443`（HTTPS）のような既知のポートへのトラフィックのみ許可します。ACS（Admin Config Service）API を使用してカスタム送信ポートを開放できますが、この機能は**シングルインスタンスまたはトライアル版の Splunk Cloud では利用できません**。

この問題の回避策として、本アドオンではデフォルトポートを `443` に設定しています。ClickHouse Cloud はポート `8443` と同様に `443` でも HTTPS 接続を受け付けるため、追加のネットワーク設定なしに Splunk Enterprise と Splunk Cloud の両方で動作します。

| 環境 | ポート `443` | ポート `8443` |
|------|-------------|--------------|
| Splunk Enterprise（オンプレミス） | ✅ 動作 | ✅ 動作 |
| Splunk Cloud（標準） | ✅ 動作 | ❌ デフォルトでブロック |
| Splunk Cloud（ACS 送信ルール設定済み） | ✅ 動作 | ✅ 動作（ルール追加時） |

