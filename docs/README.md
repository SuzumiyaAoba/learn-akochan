# Akochan ドキュメント案内

このディレクトリの文書は、Akochan が MJAI のイベント列を受け取り、局面を復元し、候補手を評価して次の行動を返すまでを、ソースコードを読まずに追えるように整理したものです。

## 推奨読順

目的に応じて部分的に読めますが、初めて読む場合は次の順番を推奨します。

1. **全体把握** — [akochan-architecture.md](./akochan-architecture.md)

   ゲーム進行、手牌探索、他家推定、動的計画法、順位点期待値を一枚の地図として把握します。

2. **運用** — [03-runtime-and-mjai.md](./03-runtime-and-mjai.md)

   ビルド、生成物、自己対局、CLI、標準入出力、MJAI TCP 連携、ログ確認の手順を読みます。

3. **AI 探索** — [04-ai-search-and-decision.md](./04-ai-search-and-decision.md)

   手牌候補グラフ、手替わり、DP、押し引き、候補の比較、最終行動選択の順に読みます。

4. **統計モデル** — [05-models-evaluation-and-parameters.md](./05-models-evaluation-and-parameters.md)

   他家の聴牌・待ち・打点、流局や和了の確率、固定パラメータ、順位確率、`pt_exp_total` の作り方を確認します。

5. **ルール実装** — [02-state-rules-scoring.md](./02-state-rules-scoring.md)

   牌表現、イベントからの状態復元、待ち・役・符・点数、合法手、フリテンや立直などの実装上の境界を確認します。

## 入力から行動までの短い地図

```text
MJAI/自己対局の JSON イベント列
  → 最新 start_kyoku から Game_State を復元
  → 手牌・副露から合法な打牌/鳴き/和了候補を列挙
  → 自分の和了・流局と、他家の聴牌/待ち/打点を確率化
  → 点数移動後の4人の順位確率を計算
  → 順位点期待値 pt_exp_total で候補を比較
  → 先頭の行動を MJAI 応答または対局イベントとして出力
```

この流れで、`Game_State` と合法性は [ルール実装文書](./02-state-rules-scoring.md)、候補グラフと探索は [AI探索文書](./04-ai-search-and-decision.md)、確率と順位価値は [統計モデル文書](./05-models-evaluation-and-parameters.md)、外部との入出力は [運用文書](./03-runtime-and-mjai.md) が担当します。

## 各文書の役割

| 文書 | 読むと分かること |
|---|---|
| [akochan-architecture.md](./akochan-architecture.md) | システム全体、主要コンポーネント、AIの処理順、候補評価から順位点期待値まで |
| [03-runtime-and-mjai.md](./03-runtime-and-mjai.md) | ビルド、実行ファイル、自己対局、ログ、`pipe`/`pipe_detailed`、`game_server`、MJAI TCP |
| [04-ai-search-and-decision.md](./04-ai-search-and-decision.md) | 手牌の候補生成、探索グラフ、DP、リーチ/鳴き/降りの選択、行動評価 |
| [05-models-evaluation-and-parameters.md](./05-models-evaluation-and-parameters.md) | ロジスティック回帰・softmax、他家推定、和了/流局確率、順位確率、固定パラメータ |
| [02-state-rules-scoring.md](./02-state-rules-scoring.md) | 牌番号、`Moves`→`Game_State`、シャンテン/待ち、役/符/点数、合法手と例外 |

## 用語集

| 用語 | このリポジトリでの意味 |
|---|---|
| MJAI | 対局サーバーとAIが交換するJSONイベント形式。1イベントを1行で送る経路もある |
| `Moves` | `json11::Json` の時系列配列。`start_kyoku` 以降を再生して現在局面を作る |
| `Game_State` | 場風・局・本場・供託・ドラ表示牌と4人の `Player_State` を持つ現在局面 |
| `Player_State` | 点数、自風、`Hai_Array` の手牌、副露、河、立直フラグを持つプレイヤー状態 |
| `Hai_Array` | 牌の種類ごとの枚数配列。赤5は別添字で持ち、`haikind` で通常5へ合算できる |
| 副露（`Fuuro`） | チー、ポン、大明槓、暗槓、加槓。鳴いた牌と手牌から出した `consumed` を記録する |
| シャンテン | 和了までの距離。共有の待ち計算では通常形の聴牌を0として検出し、七対子は専用式で計算する |
| 待ち（`machi`） | 単騎、シャンポン、両面、嵌張、辺張など、1枚加えると和了形になる牌種 |
| 翻（`han`）/役 | 和了の価値を表す単位と、その条件。`Agari_Info` はツモ用/ロン用を分けて持つ |
| 符（`fu`） | 面子、雀頭、待ち、門前ロンなどから加算する点数計算の単位 |
| 合法手 | 直前イベント、手牌枚数、立直・副露・ツモ回数、和了役、フリテンを満たす行動 |
| 候補グラフ / DP | ツモ・打牌・鳴き・槓で到達できる手牌状態をつなぎ、将来価値を比較する探索 |
| `tenpai_prob` など | 他家の聴牌、将来状態、流局、和了などを推定する固定統計モデル |
| `pt_exp_total` | 各結果の確率に順位点を掛けて合算した候補の期待値。最終行動の比較軸 |

## 実装仕様と一般ルールの区別

文書中の「平和」「フリテン」「海底」などの用語は一般的な麻雀の意味を持つが、条件・境界・例外はAkochanの現在のソースを基準にしている。一般的なルール説明と実装が一致しない場合は、次を優先して読む。

- 状態、合法手、役、符、点数の具体的な条件は [02-state-rules-scoring.md](./02-state-rules-scoring.md) の実装仕様。
- `instant`/`ako` の確率や順位点は実戦の真の確率ではなく、[05-models-evaluation-and-parameters.md](./05-models-evaluation-and-parameters.md) に記載したモデル出力。
- ビルド・CLI・MJAIの挙動は、ルール文書ではなく [03-runtime-and-mjai.md](./03-runtime-and-mjai.md) の実行仕様。
- 04のAI探索文書でも、探索の前提となる牌・合法性は02、確率評価は05を参照する。

例えば、共有層の通常形シャンテン値、立直後の槓、海底付近の槓、和了後の `Game_State` の点数反映には実装上の制約がある。これらを「標準ルールだから必ずそうなる」と解釈せず、02の例外欄を確認する。

## 対象外

この案内と各設計文書は、Akochanのドメインロジックを理解するためのものです。次は詳細説明の対象にしません。

- `json11` の汎用JSONパーサー、文字列化、型変換などの一般実装。必要なイベント項目だけを各文書で説明します。
- `params/` の重みを作る学習手順、学習データ、再学習環境。リポジトリには実行時に読む固定重みがあり、学習処理は含まれません。
- 実質的に空の接続・プレースホルダーファイル。例として [`import.cpp`](../akochan/import.cpp) と [`combination.cpp`](../akochan/ai_src/combination.cpp) はドメインロジックの根拠にしません。
- この案内に掲載していない外部MJAIサーバー固有の仕様。

実装を変更するときは、まず全体把握文書で影響範囲を確認し、運用・探索・統計モデル・ルール実装の該当文書を横断して、イベント形式、候補、確率、点数の単位がずれないことを確認してください。
