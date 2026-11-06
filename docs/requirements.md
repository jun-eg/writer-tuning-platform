# 要件定義書: Zenn 記事執筆スタイル チューニング基盤

| 項目 | 内容 |
|---|---|
| 作成者 | jun-eg |
| 作成日 | 2026-11-06 |
| 状態 | 草案（未決事項あり） |

## 1. 目的

自分が過去に書いた Zenn 記事 1 本を教師データとし、同じ文体・構成で記事を書ける「生成環境」を自動反復で作り上げる。

チューニング対象はモデルの重みではなく、生成 AI に渡す環境（system prompt、スタイルガイド、ツール設定）である。記事本文は毎回ゼロから生成し直し、修正は環境側にのみ加える。

## 2. 用語

| 用語 | 意味 |
|---|---|
| 教師記事 | 自分が執筆した Zenn 記事。1 本のみ |
| 環境 | generator に渡す system prompt、スタイルガイド、ツール設定の束。git で版管理する |
| ルーブリック | 採点軸ごとに 0〜4 点の状態を文章で定義した採点表 |
| run | 環境のチューニング 1 サイクル。合格または反復上限で終了する |
| iter | run 内の 1 反復。生成、採点、環境修正の 1 周 |
| 題材 | generator に与える「何について書くか」の指示 |

## 3. スコープ

### 3.1 含む

- 教師記事からのルーブリック抽出とレビュー
- 生成、採点、環境修正の自動ループ
- 合格した環境の汎用性評価
- ループ全体の定期監視と異常通知
- 全サービスを 1 リポジトリに収め、Docker で分離して実行

### 3.2 含まない

- モデルのファインチューニング
- Zenn への自動投稿
- 複数ユーザーへの対応

## 4. 前提条件

| 項目 | 確定内容 |
|---|---|
| 教師記事 | 1 本。作成者自身の Zenn 記事 |
| 教師記事の題材 | 自分の GitHub リポジトリ（public） |
| 情報源 | 上記リポジトリのみ。Issue、ドキュメント、ADR が整備されている |
| 情報源へのアクセス | 外部 API サーバーは作らない。generator から GitHub REST API を直接読む |
| web 検索 | generator に許可する |
| 合格基準 | ルーブリック合計の 80% 以上 |
| 反復上限 | 1 run あたり 20 iter |
| コスト上限 | 設けない |
| 監視 | Haiku が定期起動し、異常を Grok に報告する |
| 通知の目的 | 主に作成者への通知 |

## 5. システム構成

### 5.1 サービス一覧

| サービス | 役割 | モデル |
|---|---|---|
| orchestrator | run と iter の進行管理。LLM を使わない | なし |
| generator | 環境を読み、題材から記事を生成する | Claude Opus |
| judge | 生成記事を教師記事とルーブリックで採点する | Claude Opus |
| refiner | judge の結果を元に環境を書き換える | Claude Fable |
| env_evaluator | 合格した環境を別題材で評価し、汎用性と漏洩を点検する | Claude Fable |
| monitor | 定期起動でログと指標を読み、異常を Grok に報告する | Claude Haiku |
| 報告先 | monitor からの報告を受け、作成者へ通知する | Grok（xAI API） |

### 5.2 アクセス境界

各サービスは Docker コンテナとして分離し、必要最小限のマウントとネットワークだけを与える。

| サービス | マウント | 外部ネットワーク | 環境変数 |
|---|---|---|---|
| orchestrator | jobs 読み書き | なし | なし |
| generator | env 読み取り、jobs 書き込み | GitHub、web 検索、Anthropic | ANTHROPIC_API_KEY、GITHUB_TOKEN |
| judge | teacher 読み取り、jobs 読み書き | Anthropic | ANTHROPIC_API_KEY |
| refiner | env 読み書き、jobs 読み取り | Anthropic | ANTHROPIC_API_KEY |
| env_evaluator | env 読み取り、teacher 読み取り、jobs 書き込み | GitHub、Anthropic | ANTHROPIC_API_KEY、GITHUB_TOKEN |
| monitor | jobs 読み取り、env の git log 読み取り | Anthropic、xAI | ANTHROPIC_API_KEY、XAI_API_KEY |

generator と refiner は教師記事を読めない。judge と refiner は情報源と web に出られない。

### 5.3 リポジトリ構成

```
writer-tuning-platform/
├── compose.yml
├── docs/
│   └── requirements.md
├── teacher/                 # 教師記事と確定済みルーブリック
│   ├── article.md
│   └── rubric.yml
├── env/                     # チューニング対象。git で版管理
│   ├── system_prompt.md
│   ├── style_guide.md
│   └── tools.yml
├── jobs/                    # 共有ボリューム。1 iter = 1 ディレクトリ
│   └── run-0001/
│       └── iter-01/
│           ├── article.md
│           ├── score.json
│           └── env_diff.md
├── services/
│   ├── generator/
│   ├── judge/
│   ├── refiner/
│   ├── env_evaluator/
│   └── monitor/
└── orchestrator/
```

## 6. 機能要件

### 6.1 ルーブリック抽出（初回のみ）

- Fable が教師記事を読み、採点軸と各軸の 0〜4 点の状態定義を `teacher/rubric.yml` に書き出す。
- 作成者がレビューし、確定する。確定後は run の途中で変更しない。
- 軸の定義は、確定後は教師記事を参照しなくても採点できる記述（数値、型）に寄せる。別題材でも同じルーブリックを使うため。

採点軸の候補:

| 軸 | 0 点 | 4 点 |
|---|---|---|
| 記事の骨格 | 導入なしで本題に入る | 教師記事と同じ順序で進む |
| 見出しの粒度 | 階層が単一、または乱立 | 教師記事と同じ階層と数の範囲 |
| 語尾と人称 | 文体が混在 | 教師記事と一致 |
| 一文の長さ | 平均が教師記事から大きく離れる | 平均と分散が教師記事の範囲内 |
| コードブロック | なし、または説明なし | 教師記事と同じ頻度で前後に説明がある |
| 情報源の示し方 | 出典なし | Issue や ADR へのリンクの置き方が同じ |
| 冒頭と結び | 定型なし | 教師記事と同じ型の導入とまとめがある |

### 6.2 チューニングループ

1. orchestrator が `jobs/run-NNNN/iter-MM/` を作り、generator を起動する。
2. generator は `env/` と題材指示を読み、GitHub ツールと web 検索で記事を書き、`article.md` を置く。
3. judge は `article.md`、`teacher/article.md`、`teacher/rubric.yml` を読み、`score.json` を置く。
4. 合計が 80% 以上なら run を合格として終了し、env_evaluator に進む。
5. 未満なら refiner が `score.json` と `env/` を読み、環境を書き換えて commit する。次の iter へ進む。
6. 20 iter で合格しなければ run を失敗として閉じる。

`score.json` の内容:

- 軸ごとの点数
- 軸ごとの根拠（生成記事からの引用）
- 軸ごとの不足している特徴
- 合計と合格判定

### 6.3 環境評価

- 合格した環境に、同じリポジトリの別の Issue 群を題材として与え、記事を生成させる。
- 同じルーブリックで採点する。
- 合格なら環境に git タグを打ち、完了とする。
- 不合格なら理由を refiner に渡し、新しい run を開始する。

### 6.4 監視

monitor は cron で定期起動し、以下を異常として Grok に報告する。

- 反復上限に達した run の失敗
- スコアが 5 iter 連続で上がらない
- 同じ環境差分の繰り返し
- API エラーの連続
- 1 iter が一定時間を超えて終わらない

### 6.5 generator のツール

GitHub REST API を内部で呼ぶ読み取り専用のツール層を generator に置く。

| ツール | 入力 | 出力 |
|---|---|---|
| search | query | 種別、タイトル、URL、冒頭 200 字の一覧 |
| get_issue | number | タイトル、本文、ラベル、作成日、コメント一覧 |
| list_files | path | ファイル一覧 |
| read_file | path | ファイル本文 |

認証は権限なしの PAT を generator と env_evaluator にだけ渡す。未認証のレート制限を避けるため。

## 7. 非機能要件

### 7.1 教師データの漏洩防止

- generator と refiner は `teacher/` をマウントしない。
- generator の web 検索から作成者の Zenn アカウント配下を除外する。
- 対象リポジトリ内に教師記事への URL がある場合、ツール層でマスクする。
- judge のフィードバックは軸、点数、不足している特徴のみとし、教師記事の文を引用しない。

### 7.2 再現性

- 環境は iter ごとに git commit し、スコアと対応づける。
- `jobs/` は run と iter の単位で残し、削除しない。

### 7.3 分離

- 各サービスは別コンテナで動かし、5.2 の境界を compose で強制する。
- サービス間の通信は共有ボリューム経由とし、サービス間 HTTP は設けない。

## 8. 未決事項

| 番号 | 項目 | 選択肢 | 決定に必要なこと |
|---|---|---|---|
| 1 | 作成者への通知経路 | メール、Discord、Slack、GitHub Issue | Grok は文章を返すだけで通知手段を持たないため、monitor が最終的にどこへ流すかを決める |
| 2 | 題材指示の形 | タイトルのみ、または参照すべき Issue 番号と ADR の一覧 | 前者は情報収集の仕方まで環境に含まれ、後者は文体のみに絞れる |
| 3 | 実行環境 | WSL2 上で compose を回す、または別ホスト | 常時稼働させる場合は WSL2 の停止に注意が必要 |
| 4 | ルーブリックの軸の確定 | 6.1 の候補を元に Fable が抽出し作成者がレビュー | 教師記事の投入後に実施 |
| 5 | monitor の起動間隔 | 未定 | iter の所要時間が分かってから決める |
| 6 | 「1 iter が一定時間を超えて終わらない」の閾値 | 未定 | 同上 |
| 7 | 対象リポジトリの README 等に教師記事 URL があるか | 要確認 | 7.1 のマスク対象を決める |
| 8 | env_evaluator に与える別題材の選び方 | 未定 | 教師記事と重ならない Issue 群の抽出基準 |
