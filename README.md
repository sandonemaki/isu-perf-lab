# isu-perf-lab

## 学習終了時（課金を止める）

```bash
make aws.down       # EC2インスタンス（web・bench）を停止
docker compose down # ローカルのDockerも停止（任意）
```

`make aws.down` でEC2の2台（webサーバー・ベンチマークサーバー）が停止します。停止状態は課金されません。

---

## 学習開始時

EC2を再起動すると**IPアドレスが変わる**ので、SSHの設定も更新する必要があります。

```bash
docker compose up -d        # ローカルのDocker起動
make aws.up                 # EC2インスタンスを起動
make aws.setup-ssh-config   # SSH設定を新しいIPで更新
```

| コマンド | やること |
| --- | --- |
| `make aws.up` | EC2を起動して自分のIPをセキュリティグループに追加 |
| `make aws.setup-ssh-config` | `.ssh/config` のIPを新しいIPに書き換えてSSH接続確認 |

---

## 通常の作業フロー

```bash
make bench      # ① ログ洗い替え → ベンチマーク実行 → ログダウンロード
make analyze    # ② ログ解析 → ClickHouseに保存
make perf.open  # ③ Grafana・ClickHouseをブラウザで開く
```

Grafanaで表を表示するには `make bench` → `make analyze` でClickHouseにデータを入れる必要があります。
`make perf.open-ch` はClickHouseのWeb UIを開くだけで、データを作る操作ではありません。

| コマンド | データへの影響 |
| --- | --- |
| `make bench` | ベンチマーク実行・ログ取得（`results/` にファイルが増える） |
| `make analyze` | ログを解析してClickHouseにデータを保存 ← **ここでデータが入る** |
| `make perf.open-ch` | ClickHouse Web UIを開くだけ（データの確認・手動クエリ用） |
| `make perf.open-grafana` | GrafanaをブラウザW（ClickHouseにデータがあれば表示される） |

---

## webサーバーに ClickHouse・Grafana をデプロイして使う場合

ローカルのDockerを使わず、webサーバー上で ClickHouse・Grafana を動かす構成です。

### ① ローカルのコンテナを停止

```bash
docker compose down   # ローカルの ClickHouse・Grafana を停止
```

### ② webサーバーにデプロイ

```bash
make perf.deploy      # ClickHouse・Grafana を webサーバーに起動
```

### ③ 通常の作業フロー（webサーバー向け）

```bash
make bench                # ベンチマーク実行 → ログダウンロード
make analyze              # ログ解析 → webサーバーの ClickHouse に保存
make perf.open-grafana    # webサーバーの Grafana をブラウザで開く
make perf.open-ch         # webサーバーの ClickHouse Web UI をブラウザで開く
```

`make perf.open-grafana` / `make perf.open-ch` は `perf.define-variables` によって接続先を自動判定します。

| 状態 | 接続先 |
| --- | --- |
| ローカルのDockerが起動中 | `localhost` |
| ローカルのDockerが停止中 | webサーバーのIP |

---

## `make perf.deploy` でローカル起動中のメッセージが出た場合

`make perf.deploy` を実行したときに以下のメッセージが出た場合：

```
パフォーマンス計測環境はローカルにデプロイされています
```

ローカルのDockerで ClickHouse と Grafana がすでに起動しています。
ローカルのコンテナを停止してから、再度 `make perf.deploy` を実行してください。

```bash
docker compose down   # ローカルのコンテナを停止
make perf.deploy      # webサーバーにデプロイ
```

---

## SQLを更新した場合（テーブル定義の変更）

`etc/clickhouse-server/docker-entrypoint-initdb.d/` 配下のSQLを変更したときは、
ClickHouseのボリュームを削除して再起動する必要があります。

```bash
docker compose down -v  # コンテナ停止 + ボリューム削除（テーブルがリセットされる）
docker compose up -d    # 再起動（SQLが自動実行されてテーブルが再作成される）
```

確認：
```bash
make perf.open-ch
# ClickHouse Web UIで以下を実行
SHOW TABLES;
```

---

## 現在の状態確認

```bash
make aws.status   # EC2の状態とIPを確認
```

---

## Makefile コマンド一覧

```bash
make help   # 全コマンドの一覧を表示
```

### よく使うコマンド

| コマンド | 内容 |
| --- | --- |
| `make bench` | ログ洗い替え → ベンチマーク → ログダウンロード（一括） |
| `make analyze` | ログ解析 → ClickHouseへ保存（一括） |
| `make deploy` | アプリ・Nginx・MySQL設定をwebサーバーにデプロイ |
| `make setup` | webサーバーの初回セットアップ |
| `make perf.open` | GrafanaとClickHouseをブラウザで開く |
| `make perf.open-ch` | ClickHouse Web UIをブラウザで開く |
| `make perf.open-grafana` | GrafanaをブラウザWeで開く |
| `make ssh.web` | webサーバーにSSH接続 |
| `make ssh.bench` | benchサーバーにSSH接続 |
| `make aws.up` | EC2インスタンスを起動 |
| `make aws.down` | EC2インスタンスを停止 |
| `make aws.setup-ssh-config` | SSH設定を更新 |
| `make aws.status` | EC2の状態確認 |
| `make aws.create-cfn` | AWSのCloudFormationスタックを作成 |
| `make aws.delete-cfn` | AWSのCloudFormationスタックを削除 |

---

## ディレクトリ構造

```
isu-perf-lab/
│
├── Makefile                       ← よく使うコマンドをまとめた「コマンド集」
├── compose.yaml                   ← ローカルDocker環境（ClickHouse・Grafana）
├── compose.tool.yaml              ← 分析ツール用Docker（alp・pt・ch クライアント）
├── alp.yaml                       ← alpの設定ファイル（Nginxログ解析ルール）
├── .envrc                         ← 環境変数の設定（direnvで自動読み込み）
├── .envrc.override                ← 個人設定の上書き用（GitHubユーザー名など）
├── private-isu.yaml               ← AWSのEC2を作るCloudFormationの定義
├── schema.sql                     ← MySQLのテーブル定義
├── dummy-mysql-slow.log.tmpl      ← MySQLスロークエリログのダミーテンプレート
│
├── makefiles/                     ← Makefileを機能別に分割したファイル群
│   ├── aws.mk                     ←   AWSのEC2起動・停止・SSH設定
│   ├── perf.mk                    ←   ブラウザでGrafana/ClickHouseを開く
│   ├── ssh.mk                     ←   SSH接続コマンド
│   └── tool.mk                    ←   ツール確認コマンド
│
├── scripts/                       ← 実際の処理を行うシェルスクリプト
│   ├── 00-setup.sh                ←   webサーバーの初回セットアップ
│   ├── 01-deploy-app.sh           ←   Goアプリのデプロイ
│   ├── 01-deploy-mysql.sh         ←   MySQL設定のデプロイ
│   ├── 01-deploy-nginx.sh         ←   Nginx設定のデプロイ
│   ├── 02-bench.sh                ←   ベンチマーク実行
│   ├── 02-log-rotate.sh           ←   ログの洗い替えと再起動
│   ├── 02-prepare-analyze.sh      ←   ログをresults/にダウンロード
│   ├── 03-analyze.sh              ←   ログ解析（alp・pt-query-digest）→ TSV生成
│   ├── 04-store-nginx-access-runs.sh ← NginxログのTSVをClickHouseに保存
│   ├── 04-store-results.sh        ←   ベンチスコアをClickHouseに保存
│   ├── 04-store-slow-queries.sh   ←   スロークエリのTSVをClickHouseに保存
│   └── 99-util.sh                 ←   共通関数（log_info・start_timerなど）
│
├── etc/                           ← 各サービスの設定ファイル
│   ├── clickhouse-server/
│   │   └── docker-entrypoint-initdb.d/
│   │       ├── 001_results.sql    ←   resultsテーブル（ベンチスコア）
│   │       ├── 002_nginx_access_logs.sql ← nginx_access_runsテーブル
│   │       └── 003_slow_query_logs.sql   ← slow_runs・slow_queriesテーブル
│   ├── grafana/provisioning/
│   │   ├── datasources/
│   │   │   └── clickhouse.yaml    ←   GrafanaとClickHouseの接続設定
│   │   └── dashboards/
│   │       └── default.yaml       ←   Grafanaのダッシュボード読み込み先
│   ├── mysql/
│   │   └── mysql.conf.d/
│   │       └── mysqld.cnf         ←   MySQL設定（スロークエリログ有効化など）
│   └── nginx/
│       ├── nginx.conf             ←   Nginx設定
│       └── sites-available/
│           └── isucon.conf        ←   バーチャルホスト設定
│
├── dockerfiles/
│   └── Dockerfile.alp             ← alp（Nginxログ解析ツール）のDockerイメージ
│
├── private_isu/                   ← ベンチマーク対象のWebアプリ本体
│   └── webapp/golang/
│       ├── app.go                 ←   GoのWebアプリ（チューニング対象）
│       └── templates/             ←   HTMLテンプレート
│
├── results/                       ← ベンチマーク結果が溜まるディレクトリ
│   └── 2026-05-06T00:00:00Z_score:1234/
│       ├── result.json            ←   スコア・開始終了時刻
│       ├── var/log/nginx/access.log  ← Nginxアクセスログ
│       ├── var/log/mysql/mysql-slow.log ← MySQLスロークエリログ
│       ├── analyzed_nginx_access.tsv  ← alp解析結果
│       ├── analyzed_slowquery         ← pt-query-digest解析結果（テキスト）
│       ├── analyzed_slowquery.json    ← pt-query-digest解析結果（JSON）
│       ├── analyzed_slow_run.tsv      ← slow_runs用TSV
│       └── analyzed_slow_queries.tsv  ← slow_queries用TSV
│
├── var/lib/grafana/dashboards/    ← Grafanaダッシュボード定義ファイルの保存先
│
└── .ssh/
    ├── config                     ← web・benchサーバーのSSH設定（自動生成）
    └── ssh_config.tmpl            ← SSH設定のテンプレート
```

---

## 全体のデータフロー

```
[AWS EC2 webサーバー]
  - Goアプリ (private_isu)
  - MySQL  → mysql-slow.log
  - Nginx  → access.log
       |
       | make bench
       | (02-log-rotate.sh → 02-bench.sh → 02-prepare-analyze.sh)
       ↓
[ローカルPC] results/<タイムスタンプ>_score:<スコア>/
  ├── result.json
  ├── var/log/nginx/access.log
  └── var/log/mysql/mysql-slow.log
       |
       | make analyze
       | (03-analyze.sh)
       | alp          → analyzed_nginx_access.tsv
       | pt-query-digest → analyzed_slowquery / analyzed_slowquery.json
       |              → analyzed_slow_run.tsv / analyzed_slow_queries.tsv
       ↓
[ClickHouse] (Docker)
  ├── results          ← ベンチスコア
  ├── nginx_access_runs ← Nginxアクセスログ統計
  ├── slow_runs        ← スロークエリ全体統計
  └── slow_queries     ← クエリごとの統計
       |
       ↓
[Grafana] (Docker)
  グラフで可視化
```
オンプレ環境構築
