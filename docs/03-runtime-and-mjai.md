# Akochan の実行環境・対局進行・MJAI 連携

この文書は、Akochan を実際にビルドして起動し、自己対局・MJAI 対局・ログ解析を行うときの処理を、ソースコードを読まずに追える形にまとめたものです。AI の候補評価や確率計算そのものは [`akochan-architecture.md`](./akochan-architecture.md) を参照してください。ここでは、それらを呼び出す実行入口、イベント列の進み方、外部プロセスとの通信を中心に説明します。

根拠とした主なファイルは次の通りです。

- [`akochan/main.cpp`](../akochan/main.cpp)、[`main.hpp`](../akochan/main.hpp): CLI の分岐と入出力
- [`akochan/mjai_manager.cpp`](../akochan/mjai_manager.cpp)、[`mjai_manager.hpp`](../akochan/mjai_manager.hpp): 自己対局と一手 API の状態遷移
- [`akochan/mjai_client.cpp`](../akochan/mjai_client.cpp)、[`mjai_client.hpp`](../akochan/mjai_client.hpp): TCP/MJAI クライアント
- [`akochan/analyze.cpp`](../akochan/analyze.cpp)、[`stats.cpp`](../akochan/stats.cpp): レビューと統計
- [`akochan/Makefile*`](../akochan/Makefile)、[`ai_src/Makefile*`](../akochan/ai_src/Makefile): ビルド構成
- [`akochan/setup_match.json`](../akochan/setup_match.json)、[`setup_mjai.json`](../akochan/setup_mjai.json)、[`mjai.sh`](../akochan/mjai.sh): 実行設定と起動例

## 1. まず押さえる実行モデル

Akochan の局面は、可変の盤面を直接渡すのではなく、MJAI 形式の JSON オブジェクトを時系列に積んだ `Moves`（JSON 配列）で渡します。典型的には次のような列です。

```text
start_game
  → start_kyoku
  → tsumo → dahai
  → tsumo → dahai …
  → hora または ryukyoku
  → end_kyoku
  → 次の start_kyoku、または end_game
```

AI は列の中から直近の `start_kyoku` を探して局面を再構築し、その末尾が自分の `tsumo` なら打牌側、他家の `dahai`/`kakan` なら鳴き・ロン側の候補を作ります。自己対局エンジン、ファイルレビュー、標準入力の `pipe`、TCP クライアントは、入力経路は異なってもこのイベント列と `ai()`/`ai_review()` を共有します。

実行時の大まかな責務分担は次の通りです。

| 部品 | 役割 |
| --- | --- |
| `system.exe` | CLI の入口。自己対局、ログ解析、標準入出力、TCP クライアントを選ぶ |
| `libai.so` / `ai.dll` | `ai()`、`ai_review()`、合法手の評価などを含む AI 動的ライブラリ |
| `mjai_manager` | 山、配牌、ツモ、鳴き、和了、流局、次局をイベントとして追加 |
| `MJAI_Interface` | TCP から受けたイベントを AI 用の記録へ正規化し、二段階行動を保持 |
| `setup_*.json` | 対局数・起家・出力先、または AI の戦術 JSON |
| `params/` | AI が実行時に読む固定重み。学習処理は含まれない |

`setup_*.json` と `params/` は相対パスで読むため、通常は `akochan/` をカレントディレクトリにして起動します。

## 2. ビルドと生成物

### 2.1 二段階ビルド

AI 部分を先に動的ライブラリへビルドし、その後にルートの実行ファイルをビルドします。ルート側は `ai_src` を実行時にロードする `dlopen` 方式ではなく、リンク時に `-lai` で接続します。

| OS | AI 側の実行場所とコマンド | 生成物 | ルート側 | 最終実行ファイル |
| --- | --- | --- | --- | --- |
| Linux | `ai_src` で `make -f Makefile_Linux` | `libai.so`（親ディレクトリへコピー） | `make -f Makefile_Linux` | `system.exe` |
| macOS | `ai_src` で `make -f Makefile_MacOS` | `libai.so`（親ディレクトリへコピー） | `make -f Makefile_MacOS` | `system.exe` |
| Windows/MinGW | `ai_src` で `make` | `ai.dll`（親ディレクトリへコピー） | `make` | `system.exe` |

README の手順を操作にすると、Linux の場合は次の順です。

```text
cd akochan/ai_src
make -f Makefile_Linux
cd ..
make -f Makefile_Linux
```

`ai_src` 側の Makefile は `*.cpp` と `learn/*.cpp`（存在する場合）、`../share/*.cpp` をまとめてコンパイルし、`-shared` でライブラリにします。Linux 版は `-fPIC`、`-pthread`、OpenMP、実行環境の CPU 数から決める `-DNPROCS` を使います。Windows 版は `NPROCS` を Makefile に手で設定し、`WINSTD` を定義します。ルート側はルート直下の `*.cpp` と `share/*.cpp` をオブジェクト化し、生成済み AI ライブラリと Boost をリンクします。

### 2.2 OS ごとの差

- ルートの [`Makefile`](../akochan/Makefile) は `-DWINSTD`、`-lws2_32`、MinGW 用 Boost 名、`-lai` を使う Windows 寄りの既定設定です。
- [`Makefile_Linux`](../akochan/Makefile_Linux) は `g++`、`-pthread`、`-lboost_system`、`-L./ -lai` を使います。
- [`Makefile_MacOS`](../akochan/Makefile_MacOS) は Homebrew の `llvm` と `boost` の場所を `brew --prefix` で求め、Apple Silicon なら `-mcpu=apple-m1`、それ以外は `-march=native` を使います。ルート実行ファイルには `-Wl,-rpath,./` が付き、カレントディレクトリのライブラリを探索します。
- AI 側の Linux/MacOS Makefile のターゲット名はどちらも `libai.so` です。macOS の一般的な拡張子とは異なりますが、リポジトリのリンク名はこの名前で固定されています。
- AI 側の Linux/MacOS Makefile の `all` には `copy` が依存先として書かれていますが、同じファイル内に `copy` ターゲットはありません。README の通り、まず既定ターゲット（`make -f Makefile_Linux`）または `libai.so` ターゲットを明示して生成物を確認するのが安全です。
- ルート側の `import.hpp` は `WINSTD` の有無で `__declspec(dllimport)` を切り替えます。Linux/MacOS では通常の関数宣言、Windows では DLL の import 宣言になります。

ライブラリが見つからない、Boost/OpenMP が無い、またはライブラリと実行ファイルの OS・ABI が合わない場合、AI の呼び出し以前にリンクまたは起動ローダで停止します。

## 3. CLI の入口

`main.cpp` の `usage()` に表示される入口に加えて、実装には `game_server`、`legal_action`、`legal_action_log_all` もあります。引数個数の判定は厳密なので、README の表記と実装を分けて確認してください。

### 3.1 自己対局

```text
system.exe test [seed_init seed_end]
system.exe initial_condition_match [seed_init seed_end]
```

`test` はカレントディレクトリの `setup_match.json` を読みます。

1. `result_dir` を作成し、そこへ `setup_match.json` のコピーを保存する。
2. `tactics` 配列が4要素であることを確認し、`set_tactics()` でプレイヤーごとの設定を登録する。
3. seed は既定で `[1000,1001)`。引数がちょうど2個の追加引数（`argc == 4`）のときだけ指定値を使う。
4. 各 seed について `chicha` 配列の各起家を順に処理する。したがって seed 2個、起家 `[0,1]` なら4半荘を実行する。
5. `seed_mt19937(seed)`、空の記録と山を用意し、`game_loop()` を呼ぶ。
6. 結果を `result_dir/haifu_log_<seed>_<chicha>.json` に保存する。

自己対局では `player_id = -1` なので、4人とも `ai()` が選びます。進行中のイベントが標準出力へ表示されるため、自己対局の標準出力は MJAI の純粋な応答ストリームではありません。

`initial_condition_match` は、`setup_match.json` の `initial_dir` と `kyoku_init` を使い、`initial_dir/haifu_log_<seed>_0.json` 内の該当局から点数を取り出します。その点数と局番号を `start_game.first_kyoku` に入れてから `game_loop()` を開始します。通常の `test` と異なり、コード上は `std::srand(seed)` を呼びますが、自己対局の山を管理する `std::mt19937 gen` を `seed_mt19937()` では再シードしません。完全な seed 再現を期待する場合はこの差を考慮してください。

付属の `setup_match.json` には `initial_dir` と `kyoku_init` が無いため、このコマンドを使うには両キーを追加した設定と、対応する `initial_dir/haifu_log_<seed>_0.json` のログ群が別途必要です。

### 3.2 ログ・レビュー・検査

| コマンド | 動作 |
| --- | --- |
| `system.exe check` | `haifu_log.json` を1行ずつ読み、`proceed_game()` で進行検査する。イベントの進捗を標準出力へ表示する |
| `system.exe mjai_log <file> <id>` | ログ末尾の局面を `ai_review()` し、全候補とレビュー情報を表示する |
| `system.exe mjai_log <file> <id> <line_index>` | 0始まりの指定行までを読み、その時点で `ai_review()` する |
| `system.exe full_analyze <file> <id>` | ログ中の各判断局面を実際の次イベントと照合し、`<元名>_full_analyze.<拡張子>` を作る |
| `system.exe para_check` | OpenMP の各スレッド番号を表示する。AI の動作確認ではなく並列環境の確認用 |
| `system.exe legal_action <JSON文字列>` | `record` に対する合法な単一行動を列挙する |
| `system.exe legal_action_log_all <file>` | ログの各行までに対する合法単一行動の配列を出す |

`mjai_log` は読み込んだ記録を先に表示し、その後 `calculating review` と `ai_review()` の配列を出します。`ai_review()` の各要素は、おおむね `moves`（候補イベント列）と `review`（評価値など）です。

`full_analyze` は自分のツモ、または他家の打牌の直後だけを判断点とします。合法手が2個未満のルールベース局面、他家の邪魔ポン・ロン、槓、嶺上和了などは比較対象から外します。比較できた場合は、AI の最良候補のスコアと実際に選ばれた候補の差を `err` として元イベントへ追加します。比較対象の次行が不足する末尾ログや想定外のイベント列は、安全にレビューできる入力形式ではありません。

### 3.3 統計

```text
system.exe stats <dir_name>
system.exe stats_mjai [dir_name [player_name_prefix]]
```

`stats` は指定ディレクトリのログを読み、`setup_match.json` を除いてプレイヤー0〜3ごとの順位、平均順位、リーチ、鳴き、和了、放銃の回数・局あたり率を出します。1局中に一度でも `chi`/`pon` があれば `fuuro_prob`、その回数は `fuuro_num` として数えます。`stats_mjai` は `*.mjson` だけを対象にし、`start_game.names` で既定値 `Akochan` に一致するプレイヤーを1人選びます。指定した名前が無いファイルはエラーになります。

`stats_mjai` は内部で `./<dir_name>/*.mjson` を `ls` し、一時ファイル `tmp.txt` に保存します。後始末の `rm` はコメントアウトされているため、実行後に `tmp.txt` が残ります。どちらの統計も空ディレクトリ・不完全ログでは0除算やエラーになり得ます。

## 4. 自己対局の初期化と山

`game_loop()` は記録の末尾が `end_game` になるまで `proceed_game()` を繰り返します。`proceed_game()` は現在の末尾イベントを見て、次に必要なイベントを追加する状態機械です。

### 4.1 山と配牌

`prepare_haiyama()` は物理牌を0〜135で作り、`std::shuffle` した後、38種類の内部番号へ変換します。赤5は通常の5とは別番号で保持されます。最初の52枚を4人へ4人順に13枚ずつ割り当て、`chicha` が起家になるように座席を回転します。初期ドラ表示牌は山の後ろから6枚目（`haiyama[130]`）です。

- 通常ツモ: `52 + 通常ツモ回数` の位置から取る
- 嶺上ツモ: 山の末尾から嶺上ツモ回数分だけ逆向きに取る
- 通常ツモと嶺上ツモの合計上限: 70回

自己対局ログの `start_kyoku` には、4人の配牌、点数、ドラ、さらに山全体が入ります。`get_masked_log()` という他家ツモを `?` に置き換える補助関数はありますが、自己対局・TCP クライアントで自動的に適用されるわけではありません。

### 4.2 最初の局

記録が空なら `start_game` を追加します。末尾が `start_game` なら `add_first_kyoku()` が呼ばれます。

- `request.haiyama` が無ければ山を新しくシャッフルする。
- `start_game.first_kyoku` があれば、その局番号と点数を使う。
- それ以外は東1局・本場0・供託0・指定された起家・4人25000点で開始する。
- `start_kyoku` の直後は必ず起家の `tsumo` を追加する。

通常の `test` では `make_start_game()` が `kyoku_first=4`、`aka_flag=true` を設定します。

## 5. 1局の進行

### 5.1 `proceed_game()` の状態遷移

| 末尾イベント | 次に行う処理 |
| --- | --- |
| なし | `start_game` を追加 |
| `start_game` | 配牌して `start_kyoku` を追加 |
| `start_kyoku` | 起家が `tsumo` |
| `tsumo` | そのプレイヤーの和了・九種九牌・打牌/リーチ/槓を選び、イベントを追加 |
| `dahai` / `kakan` | 他家のロン・鳴き、流局、嶺上ツモ、次のツモを判定 |
| `ankan` | カンドラを追加し、嶺上ツモ |
| `daiminkan` | 嶺上ツモ |
| `reach` / `chi` / `pon` | 同じプレイヤーから後続の打牌を受け、合法なら追加 |
| `hora` / `ryukyoku` | 次局または `end_kyoku`/`end_game` |

外部の一手を `request` として受ける場合、`tsumo`・`dahai`・`kakan` の局面では単一行動の合法性を確認してから記録に反映します。自分以外のプレイヤーの行動は `ai_assign()` で自動選択されます。自己対局では `request` が空なので、4人すべてが自動選択です。

### 5.2 ツモ後の選択

ツモしたプレイヤーの候補列の先頭が次のいずれかなら、それを直ちに確定します。

- `hora`: ツモ和了。点数を計算した `hora` を追加
- `ryukyoku`（`reason=kyushukyuhai`）: 九種九牌流局
- それ以外: 候補列をそのまま追加。通常打牌は1イベント、リーチ宣言は `reach`→`dahai`、暗槓/加槓は各1イベント

和了時は、ツモ牌を含む手牌、リーチ・一発、役の翻符、表ドラ・裏ドラを使って点数を計算し、`hora.scores` を作ります。九種九牌では自分の手牌だけを公開し、他家の手牌は `?`、点数は変えずに `ryukyoku` を追加します。

### 5.3 打牌・加槓後の優先順位

`add_move_after_dahai()` は直前の `dahai` または `kakan` に対し、次の順で処理します。座席順の走査は、放銃者または加槓者を `target` とした相対位置1、2、3の順です。

1. 他家のロン (`hora`)
2. 通常ツモ＋嶺上ツモの合計が70回なら流局
3. 直前が `kakan` なら嶺上ツモ
4. ポン。複数候補があれば座席走査で最初のプレイヤー
5. 大明槓。同じく座席走査で最初の候補
6. 下家だけのチー
7. リーチ受理・必要なカンドラを追加し、次プレイヤーをツモ

ロン候補は3人全員を調べます。複数人がロン候補の場合、複数の `hora` イベントが同じ呼び出しで追加されることがあります。どれか1件でもロンがあれば、その後の鳴きや次ツモには進みません。

リーチ宣言後の打牌の次には `reach_accepted` を追加します。暗槓は暗槓直後にカンドラと嶺上ツモを追加し、大明槓・加槓は嶺上ツモ後の打牌に続けてカンドラが追加されます。

### 5.4 次局・半荘の判定

和了または流局の末尾で、直前の点数を使って `cal_next_bakaze_kyoku()`、`cal_next_honba()`、`cal_next_oya()` を計算します。

- 飛び（点数が0未満）があれば `end_kyoku` の後に `end_game`。
- 親の和了、または流局で親がテンパイなら同じ親が連荘する。
- それ以外は親を次へ回し、4局目の次は次の場の1局目へ進む。
- 九種九牌は同じ局をやり直し、親・局番号を維持する。
- 継続局では本場を増やし、和了で親が連荘しない場合は本場を0にする。
- 終局時は必ず `end_kyoku` を挿入し、続行なら新しい `start_kyoku`、終了なら `end_game` を挿入する。

この判定は `cal_next_bakaze_kyoku()` が文字列 `"tonpu"` を固定で渡して行います。東南戦を設定 JSON で選ぶ機能はこの自己対局経路にはありません。`mjai.sh` も外部サーバーへ `--game_type=tonpu` を渡す構成です。

## 6. 合法手と二段階行動

AI の返り値は「一つの JSON」ではなく、通常は `Moves`、すなわち1〜2イベントの配列です。

| 局面 | 返り値の例 | クライアントが次にすること |
| --- | --- | --- |
| 自分の `tsumo` | `[dahai]`、`[hora]`、`[ankan]`、`[kakan]` | そのまま1イベント送る |
| 自分の `tsumo` でリーチ | `[reach, dahai]` | まず `reach`、受理通知後に `dahai` |
| 他家の `dahai` | `[none]`、`[hora]`、`[daiminkan]` | その場の応答を送る |
| 他家の `dahai` でポン | `[pon, dahai]` | まず `pon`、受理通知後に `dahai` |
| 他家の `dahai` でチー | `[chi, dahai]` | まず `chi`、受理通知後に `dahai` |

`Hai_Choice::out_moves()` と `Fuuro_Choice::out_moves()` がこの配列を作ります。`ai()` は候補を評価値の降順に並べ、先頭候補だけ返します。`ai_review()` は候補をすべて返し、`calc_moves_score()` は候補とスコアの組を返します。

`reach`、`chi`、`pon` は宣言と直後の打牌が別イベントです。サーバーから `reach`/`chi`/`pon` の受理イベントが返るまで2番目の打牌を送ってはいけません。`mjai_client` は直前の `best_moves` を保存し、受理イベントを受けると `best_moves[1]` を送ります。

## 7. MJAI TCP クライアント

### 7.1 起動

`mjai_client` は `TcpClient("127.0.0.1", port)` でローカル TCP サーバーへ接続します。既定ポートは11600です。設定 JSON は引数が無ければ `setup_mjai.json`、引数が4個以上なら第4引数（`argv[3]`）を読みます。

リポジトリの [`mjai.sh`](../akochan/mjai.sh) は次の構成です。

```text
LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH
mjai server --port=11602 --game_type=tonpu --room=default \
  --games=5 --log_dir='./mjai_result' \
  './system.exe mjai_client 11602 setup_mjai.json' \
  mjai-manue mjai-manue mjai-manue
```

つまり、外部の `mjai server` が1人の Akochan と3人の `mjai-manue` を同じルームで5ゲーム実行し、`mjai_result` にログを置く想定です。`LD_LIBRARY_PATH` はカレントディレクトリの `libai.so` を見つけるためです。

### 7.2 1行JSONの往復

通信単位は改行区切りのJSONです。`ReadOneLine()` が `\n` まで同期的に読み、`json11::Json::parse()` した後、応答を `response.dump() + "\n"` で返します。概念的な往復は次の通りです。

```text
サーバー → {"type":"hello", ...}
Akochan → {"type":"join","name":"Akochan","room":"default"}
サーバー → {"type":"start_game", "id": 0, ...}
Akochan → {"type":"none"}
サーバー → {"type":"start_kyoku", ...}
Akochan → {"type":"none"}
サーバー → {"type":"tsumo", "actor":0, "pai":"..."}
Akochan → 最良候補の1行JSON
```

受信イベントと応答は次の規則です。

| 受信イベント | `MJAI_Interface` への処理 | 応答 |
| --- | --- | --- |
| `hello` | 記録へは追加しない | `join`（名前Akochan、room default） |
| `start_game` | `kyoku_first=4`、`aka_flag=true` を上書きして記録 | `none` |
| `start_kyoku` | 点数を保持し、AI 用の標準フィールドだけを記録 | `none` |
| 自分の `tsumo` | そのまま記録し、`ai()` | `best_moves[0]` |
| 他家の `dahai` かつ `possible_actions` が空でない | 記録し、`ai()` | `best_moves[0]` |
| 他家の `kakan` | そのまま記録 | `none`（この分岐では `ai()` を呼ばない） |
| 自分の `reach`/`pon`/`chi` | そのまま記録 | 保存済み `best_moves[1]` |
| `hora`/`ryukyoku` | 記録 | `none` |
| その他 | 記録 | `none` |

`MJAI_Interface::push()` は `start_kyoku` について、`bakaze`、`dora_marker`、本場、供託、局、親、点数、配牌、`type` を再構成します。受信イベントに `scores` があれば `scores_` を更新し、次の `start_kyoku` で点数が欠けた場合は保持値を使います。`start_game` で受信したプレイヤーIDは TCP ループ側で保存します。

### 7.3 TCP 版の運用上の注意

- 接続先は `127.0.0.1` 固定です。接続・読み込み・書き込みエラーはメッセージを出して `exit(1)` になります。
- TCP の受信 JSON の構文エラーは詳細なリカバリをせず、その後の `type` 判定に進みます。
- 実装は `best_moves for tsumo: ...` と `response: ...` を標準出力へ表示します。`end_game` では記録全体も標準出力へ出します。標準出力を厳密なMJAI応答だけにするサーバーでは、これらの診断行がプロトコルを壊す可能性があります。
- `argc > 2` のとき受信ログを `argv[2]` に開きます。一方、ポートも `argv[2]` から読むため、`mjai_client 11602 setup_mjai.json` ではファイル名 `11602` をログとして開く挙動になります。これは `mjai.sh` の起動例にも存在する実装上の癖です。
- `reach`/`pon`/`chi` を受けた時に保存済み配列へ2番目の要素が無いと範囲外アクセスになります。二段階行動の受理通知が、直前の判断と対応していることが前提です。
- AI本体は他家の `kakan` を入力に槍槓候補として扱えますが、TCPループのAI呼び出し条件は他家の `dahai` に限定されています。TCP版では他家 `kakan` を受けても `ai()` を呼ばず、通常の `none` を返す経路になります。

## 8. 標準入力 `pipe` と `pipe_detailed`

```text
system.exe pipe <tactics.json> <id>
system.exe pipe_detailed <tactics.json> <id>
```

どちらも1行ずつMJAIイベントを標準入力から読み、指定プレイヤー `id` の判断時だけ標準出力へ1行返します。

1. `<tactics.json>` を読み、`set_tactics_one()` で4人全員へ同じ設定を割り当てる。
2. `error` イベントは捨てる。
3. `start_kyoku` を受けたら、記録を最初の `start_game` だけ残して新しい局を始める。
4. `actor` が無いイベントでは応答しない。
5. 自分の `tsumo`、または他家の `dahai`/`kakan` で AI を呼ぶ。

`pipe` の出力は `ai()` の `Moves` 配列、`pipe_detailed` の出力は `ai_review()` の候補・レビュー配列です。`can_act` が存在して false のときに処理を打ち切る検査は `pipe` にだけあり、`pipe_detailed` はその値を見ずに以後の `actor` とイベント種別を判定します。TCP 版と違い、`pipe` は他家の `dahai` に `possible_actions` があるかを確認しません。入力側が不要なタイミングを送らないことが前提です。また、このプログラム自身は返した行動を `game_record` へ追加しないため、継続的に使う側が返答を対局記録へ反映して次の入力を送ります。

## 9. `game_server` 一手API

```text
system.exe game_server '<JSON文字列>'
```

これは TCP サーバーを起動するものではなく、1回のJSON入力で `proceed_game()` を1ステップ進めるアダプタです。`setup_mjai.json` を読み、`set_tactics_one()` を行った後、入力の `record` と `request` を使います。入力は概ね次の形です。

```json
{
  "record": [
    {"type":"start_game", "kyoku_first":4, "aka_flag":true}
  ],
  "request": {"type":"dahai", "actor":0, "pai":"1m", "tsumogiri":false},
  "chicha": 0,
  "haiyama": ["...", "..."]
}
```

仕様上の重要点は次の通りです。

- 対局者は `my_pid=0` 固定です。`request` はプレイヤー0の行動として扱われ、他の3人はAIが自動選択します。
- `record` が空なら、まず `start_game` だけが追加されるため、呼び出しを分けて次の状態を送ります。
- 初局では `chicha` が必要です。入力に `haiyama` が無い場合、直近の `start_kyoku.haiyama` を探します。外部MJAIログのように山が省略された記録では、必要な山を復元できずエラーになります。
- `tsumo`/`dahai`/`kakan` の局面で送った行動が合法か検査します。ただし `request.type == "pass"` は検査を迂回します。
- `reach`、`chi`、`pon` の直後は、次の呼び出しでプレイヤー0の打牌を送る二段階です。

返り値は `new_moves`、`msg_type`、必要なら `legal_moves` です。

| 状況 | `msg_type` | 意味 |
| --- | --- | --- |
| 新しい末尾が `reach`/`pon`/`chi` | `update_and_dahai` | 宣言を反映したので、同じプレイヤーの打牌を要求 |
| 新しい末尾が自分の `tsumo` | `update_and_dahai` | 打牌候補を `legal_moves` と共に返す |
| 新しい末尾が他家の `dahai` などで自分に鳴き機会がある | `update_and_fuuro` | ロン/鳴き/パスを `legal_moves` と共に返す |
| 新しいイベントはあるが合法手がない | `update` | 状態更新だけ |
| 不合法行動でイベントが増えない。直前が宣言または自分のツモ | `dahai_again` | 打牌を再送 |
| 不合法行動でイベントが増えない。それ以外 | `fuuro_again` | 鳴き/ロン/パスを再送 |

`new_moves` は入力時点の `record` との差分であり、AI が応答して進んだイベントも含みます。

## 10. 設定の渡り方

### 10.1 対局設定とMJAI設定

`setup_match.json` は `result_dir`、`chicha` 配列、4人分の `tactics` 配列を持ちます。`test` と `initial_condition_match` は `set_tactics()` を呼び、4人のJSONをそのまま `tactics_json[0..3]` へ保存します。

`setup_mjai.json` は1つの `tactics` オブジェクトを持ちます。`mjai_client`、`mjai_log`、`full_analyze`、`game_server`、`pipe` は `set_tactics_one()` を呼び、同じJSONを4人分へ複製します。`pipe` の `<tactics.json>` も同じ形式です。

AI が `ai()` を呼ぶと、対象プレイヤーの `tactics_json[pid]` から `Tactics` を構築します。`base` は `default`、`light`、`minimum` のいずれかで、候補列挙量やDP閾値などの既定値を選びます。`jun_pt`、カン、降り、翻符分布などは `Tactics::set_from_json()` が構造体へ読み込みます。

一方、`jun_est`、`tenpai_prob_est`、`agari_prob_est`、`ron_ratio_est`、`other_end_prob_est` などは、AI内部の各計算が共有の `tactics_json` を直接参照します。したがって、設定を追加・変更するときは「`Tactics` が読む項目」と「AI各所が直接読む項目」の両方にキーが存在するか確認します。リポジトリ付属の設定は、これらを `ako` 系へ明示的に設定しています。

### 10.2 設定が実際に効く境界

JSON にキーを書いただけでは効果は保証されません。コードが分岐でそのキーを読む場合だけ有効です。例えば、順位点は `Tactics` の `jun_pt`、順位推定方式は `tactics_json` の `jun_est`、将来ツモ数は `tsumo_num_est` から別々に読まれます。存在しない `base`、重み配列の要素数不正、確率合計が1でない設定は `assert` で停止します。

## 11. 障害時の挙動と入力前提

通常運用で問題を切り分けるときは、次の順に確認します。

1. カレントディレクトリが `akochan/` か。設定・パラメータの相対パスが解決できるか。
2. `system.exe` と同じ場所に `libai.so`（Linux/MacOS）または `ai.dll`（Windows）があるか。
3. `setup_*.json` の `tactics`、`base`、重み配列、4人分の要素数が正しいか。
4. ログの先頭が `start_game`、局ごとに `start_kyoku` があり、必須フィールドがそろっているか。
5. MJAI TCP の場合、外部サーバーのポート、改行付き1行JSON、二段階受理イベントが対応しているか。

実装は例外を使った回復より `assert` とプロセス終了に依存しています。パラメータファイルの欠落・要素不足、設定の不整合、山や配牌の欠落、想定外イベント順序では停止または未定義動作になり得ます。TCP のソケットエラーも再接続せず終了します。

## 12. ルール上の差と既知の制約

「一般的な麻雀ルール」と「このリポジトリの処理」は同じとは限りません。運用上影響が大きいものを列挙します。

- 自己対局の半荘終了判定は `tonpu` 固定です。`bakaze` が南になった後の最終局判定、飛び、親連荘条件もこの固定ルールを前提にします。
- 山は136枚をシャッフルしますが、嶺上牌の取り方は通常の物理的な王牌進行を厳密に再現せず、山末尾から逆向きに取る簡略化です。
- ツモ・嶺上ツモの総数は70回で打ち切ります。合法性検査では70回目をハイテイの1翻として扱う箇所がありますが、自己対局の `hora` 点数計算ではハイテイ分を `han_add` に加えていません。
- 複数ロンはイベント列へ複数の `hora` を追加できますが、後続の点数計算が直前の `hora` の支払いを累積する仕組みではありません。一般的な同時和了の精算と同一とは限りません。
- `get_game_state()` は `hora` の `scores` を状態へ反映せず、`hora` に遭遇すると供託を0へ戻します。自己対局の結果点数は各 `hora` 作成時に別途計算されます。
- `is_valid_*` の一部はイベントの `type` と一部フィールドだけを確認し、完全なMJAI JSON Schema検証ではありません。`chi`/`pon`/`daiminkan` の合法性検査には、直前牌の文字列へ `int_value()` を呼ぶ比較があり、意図した牌比較にならないケースがあります。
- `is_legal_hora()` は入力行動の `target` を直前打牌者と直接照合するのではなく、待ち全体とフリテン状態を走査します。AIが生成する候補と外部から手入力する行動で検査の強さが異なります。
- `request.type == "pass"` は `proceed_game()` の単一行動検査を迂回します。相手の打牌に対するパスを想定した処理であり、自分のツモ後に送ると記録が停滞する可能性があります。
- TCP版には診断文字列が標準出力へ混ざります。MJAIサーバーが1行ごとに必ずJSONを要求する場合は、そのままでは互換性が保証されません。
- AI本体・`pipe`・`game_server` が扱える他家 `kakan`（槍槓）と異なり、TCP `mjai_client` のAI呼び出し条件は他家 `dahai` のみです。他家の `kakan` 通知では `ai()` を呼ばず、槍槓判断を行いません。
- `mjai_client` の引数第2番目はポートと受信ログファイル名の両方に使われます。標準起動例ではポート番号名のファイルが作られます。
- `full_analyze` は次のイベント、槓・カンドラ、末尾行の存在を前提に添字を進めます。短いログ・不完全ログを一般的なレビュー入力として扱う設計ではありません。
- `stats` は `chi`/`pon` のみを鳴きとして数え、槓を `fuuro_num` に加えません。`stats_mjai` は名前プレフィックスで1人を選ぶため、同じ接頭辞のプレイヤーが複数いると最初の1人だけが対象です。
- `initial_condition_match` は通常の `start_game` が持つ `kyoku_first`/`aka_flag` と異なる初期記録を作り、乱数再シード経路も `test` と異なります。AIを含む全経路でそのまま再現できるとは限りません。
- 学習データ、重みの生成手順、モデルの精度、外部 `mjai`/`mjai-manue` の実装はこのリポジトリのソースからは分かりません。外部コマンドが無い場合、`mjai.sh` は開始できません。

## 13. 運用時の最短チェックリスト

自己対局なら、`ai_src` 側のライブラリを先に生成し、ルートで `system.exe` を作ってから `setup_match.json` の `result_dir` と `chicha` を確認します。まず seed 範囲を1つにして `system.exe test 1000 1001` を実行し、`haifu_log_1000_<chicha>.json` と `end_game` が生成されることを確認します。

MJAIなら、`libai.so`/`ai.dll` の探索パス、外部サーバーのポート、Akochan の `id`、`setup_mjai.json`、そして標準出力へ診断文字列が混ざる実装を確認します。標準入力連携だけが必要なら `pipe`、候補と評価値まで必要なら `pipe_detailed`、1回の状態更新と合法手一覧をAPIとして扱うなら `game_server` を選びます。

ログの最終判断だけを見たい場合は `mjai_log`、全判断を実際の選択と照合したい場合は `full_analyze`、対局数・順位・和了などを集計したい場合は `stats`/`stats_mjai` を使います。
