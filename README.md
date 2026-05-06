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

## デプロイの種類と使いどころ

| コマンド | 何をデプロイするか | いつ使うか |
| --- | --- | --- |
| `make deploy` | Goアプリ・Nginx・MySQL設定 | **チューニング中に毎回**（コードや設定を変えたとき） |
| `make perf.deploy` | ClickHouse・Grafana・OTel Collector | **初回セットアップ時** または監視設定を変えたとき |
| `make setup` | apt・mysqldef・Docker・OTel Collector本体 | **サーバー作り直し時のみ**（EC2を再作成したとき） |

### チューニング中の基本サイクル

```
app.go を編集
    ↓
make deploy       # Goアプリをwebサーバーに反映
    ↓
make bench        # ベンチマーク実行（トレース・ログが生成される）
    ↓
make analyze      # ログ解析 → ClickHouseに保存
    ↓
make perf.open    # Grafanaで結果を確認
```

### `otel_traces` にデータを入れるには

Grafanaのトレース系クエリ（`otel_traces` テーブル）にデータが入るのは、**OTel対応済みのGoアプリでベンチマークを実行した後**です。

```bash
make deploy   # OTel対応のGoアプリをデプロイ（初回または app.go 変更後）
make bench    # ベンチ実行 → トレースが otel_traces に保存される
```

`make analyze` は不要です（トレースはGoアプリから直接OTel Collector経由でClickHouseに送られます）。

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

---

## perf インスタンスと web インスタンスの関係

```
┌───────────────────────────────────────────────────────────────────┐
│  web インスタンス                                                   │
│                                                                   │
│  ┌──────────────────────────────┐                                 │
│  │  Goアプリ (isu-go :8080)     │──── プロファイリング(HTTP) ──┐  │
│  │  + pyroscope-go SDK          │                             │  │
│  └──────────────┬───────────────┘                             │  │
│                 │ OTLP HTTP                                    │  │
│                 ▼                                             │  │
│  ┌──────────────────────────────┐                             │  │
│  │  OTel Collector Agent        │                             │  │
│  │  :4317/:4318                 │                             │  │
│  │  ・traces 受信               │                             │  │
│  │  ・hostmetrics 収集          │                             │  │
│  └──────────────┬───────────────┘                             │  │
│                 │ OTLP HTTP (traces + metrics)                 │  │
└─────────────────┼─────────────────────────────────────────────┼──┘
                  │                                             │
                  ▼                                             ▼
┌───────────────────────────────────────────────────────────────────┐
│  perf インスタンス (192.168.1.30)                                   │
│                                                                   │
│  ┌──────────────────────────┐   ┌──────────────────────────────┐  │
│  │  OTel Collector Gateway  │   │  Pyroscope :4040             │  │
│  │  :4318                   │   │  (継続的プロファイリング受信)  │  │
│  │  ・traces + metrics 受信 │   └──────────────┬───────────────┘  │
│  └──────────────┬───────────┘                  │                  │
│                 │ OTLP                          │                  │
│                 ▼                               │                  │
│  ┌──────────────────────────────────────────┐   │                  │
│  │  ClickHouse :9000/:8123                  │   │                  │
│  │  otel_traces         ← トレース          │   │                  │
│  │  otel_metrics_*      ← メトリクス        │   │                  │
│  │  results             ← ベンチスコア      │   │                  │
│  │  nginx_access_runs   ← Nginxログ         │◀──┼── make analyze   │
│  │  slow_runs / slow_queries ← スロークエリ │   │   (ローカルPC)   │
│  └──────────────┬───────────────────────────┘   │                  │
│                 │                                │                  │
│                 ▼                                │                  │
│  ┌──────────────────────────────────────────┐   │                  │
│  │  Grafana :3000                           │◀──┘                  │
│  │  トレース・メトリクス・プロファイリング    │                      │
│  │  ・ベンチスコア等を一元可視化             │                      │
│  └──────────────────────────────────────────┘                      │
└───────────────────────────────────────────────────────────────────┘
```

### データ種別ごとの経路

| データ種別 | 送信元 | 経路 | 保存先 |
|---|---|---|---|
| トレース | Goアプリ | OTLP HTTP → OTel Agent → OTel Gateway | ClickHouse (`otel_traces`) |
| メトリクス (webEC2) | OTel Agent (hostmetrics) | OTLP HTTP → OTel Gateway | ClickHouse (`otel_metrics_*`) |
| プロファイリング | Goアプリ (pyroscope-go SDK) | HTTP → Pyroscope :4040 | Pyroscope |
| Nginxログ | Nginx `access.log` | `make bench` → `make analyze` (TSV) | ClickHouse (`nginx_access_runs`) |
| スロークエリログ | MySQL `slow.log` | `make bench` → `make analyze` (TSV) | ClickHouse (`slow_runs`, `slow_queries`) |
| ベンチスコア | ベンチマーク `result.json` | `make analyze` | ClickHouse (`results`) |

### ポイント

- **トレース・メトリクス** は Goアプリ起動後にリアルタイムで自動送信される（`make bench` 不要）
- **プロファイリング** は起動時に Pyroscope へ接続確認し、応答があれば継続的に送信し続ける
- **ログ系** は `make bench` でログ取得 → `make analyze` で解析・保存の2ステップが必要
