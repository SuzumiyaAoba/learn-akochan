# Akochan の手牌探索と最終意思決定

この文書は、Akochan の AI が「入力された MJAI のイベント列」から「次に返す行動」を決めるまでを、ソースコードを読まずに再現できる粒度でまとめたものです。対象は akochan/ai_src の手牌解析、候補生成、候補グラフ、動的計画法（DP）、ベタオリ、和了・放銃・流局の確率統合、行動 JSON 化です。

説明は現在の実装の挙動を基準にしています。理想的な麻雀ルール、学習済みモデルの精度、外部 MJAI サーバーの合法性検査とは区別してください。固定重みファイルの作り方は本リポジトリからは分からないため、ここでは「重みを読む場所」と「重みをどう使うか」までを記します。

## 1. 全体像

AI の入力は可変の盤面オブジェクトではなく、Moves = vector<Json> の時系列です。selector.cpp の Selector::set_selector() が直近イベントを入口にし、局面を再構成して候補を比較します。

~~~text
Moves（start_kyoku 以降のイベント列）
  │
  ├─ get_game_state()                 局面を再生
  ├─ Tehai_Analyzer                   シャンテン、待ち、和了形
  ├─ Tenpai_Estimator / 放銃推定       他家の聴牌率・危険度・打点
  ├─ Tehai_Calculator                 手牌候補と遷移グラフ
  │    ├─ 高シャンテンならルールベース
  │    ├─ それ以外なら候補を列挙
  │    └─ DP または非DPの和了確率計算
  ├─ cal_exp / 順位点期待値            和了・放銃・流局を同じ尺度へ
  ├─ Hai_Choice / Fuuro_Choice         行動候補へ戻す
  └─ pt_exp_total 最大を MJAI JSON 化
~~~

外部から使う主な入口は次の3つです。

| 入口 | 戻り値 | 用途 |
|---|---|---|
| ai(record, pid, out_console) | Moves | 最良候補1件を返す |
| calc_moves_score(record, pid) | (Moves, score) の配列 | 全候補と pt_exp_total を返す |
| ai_review(record, pid) | JSON 配列 | 全候補の moves と review を返す |

ai() は Selector の候補を降順に並べ、先頭だけを out_moves() で出力します。候補が空の場合は none を返します。

## 2. 入力イベントと牌の表現

### 2.1 AI を呼べる直近イベント

Selector::set_selector() は末尾イベントが次のいずれかであることをアサートします。

~~~text
自分の tsumo
他家の dahai
他家の kakan
~~~

直近イベントの pai を hai_str_to_int() で数値化します。牌番号は次の38要素配列です。添字0は未使用です。

| 数値 | 牌 |
|---:|---|
| 1..9 | 1..9萬 |
| 11..19 | 1..9筒 |
| 21..29 | 1..9索 |
| 31..37 | 東・南・西・北・白・發・中 |
| 10, 20, 30 | 赤5萬・赤5筒・赤5索 |

haikind(10)=5、haikind(20)=15、haikind(30)=25 です。手牌形の比較は原則として黒5へ正規化しますが、赤5が手牌内にあるか、鳴きへ出たかは Tehai_State2 のフラグに残します。

局面復元後の重要な差は次の通りです。

* 自分の tsumo では Game_State にツモ牌が既に加わっているため、リーチ中だけ解析用手牌から直近ツモ牌を1枚削除します。リーチ後はツモ切りしか通常候補にならないためです。
* 他家の dahai では自分の手牌は13枚のままです。直近捨て牌は fuuro_cand_hai として鳴き候補の入口になります。
* 他家の kakan では槍槓のロン判定を行えますが、鳴き候補の列挙は行いません。

### 2.2 見えている枚数と残り枚数

get_hai_visible_all() は、4人の河、全副露の consumed、ドラ表示牌を合計します。赤5が2枚以上見えている場合は、2枚目以降を黒5の見え枚数へ移し、赤5の見え枚数を1に保ちます。

~~~text
visible[h] = 河 + 副露の consumed + ドラ表示牌
visible[黒5] += max(visible[赤5] - 1, 0)
visible[赤5] = min(visible[赤5], 1)
~~~

get_nokori_hai_num() は次を返します。

~~~text
136 - 全員の河・副露・ドラ表示牌の見え枚数 - 自分の手牌枚数
~~~

get_hai_visible_wo_tehai() は、自分の副露を見え枚数から差し引き、他家とドラ表示だけが見える配列を作ります。DP と非DPの残り牌計算ではこの配列を牌種へ正規化して使います。

## 3. 局結果の価値を先に作る

手牌の受け入れ枚数だけでは最終判断にならないため、探索の前に「この局の結果が終わった時点の順位点期待値」を作ります。デフォルトの順位点は次です。

~~~text
1着 +90、2着 +30、3着 -30、4着 -90
~~~

### 3.1 和了時テーブル

cal_kyoku_end_pt_exp() は、和了者 pid1、放銃者またはツモの対象 pid2、翻 1..13、符インデックス 1..11 ごとに、局終了後の自分の順位点期待値を保存します。

実際の計算は次の順です。

1. ten_move_hora() で親子、本場、供託を含む点棒移動を求める。
2. リーチ宣言済みだが未受理の1000点を引き、和了者へ戻す補正を行う。
3. 局が続くか親が連荘するかを決め、局終了後の4人の点数を作る。
4. calc_jun_prob() で4着順の確率を求め、jun_pt と内積を取る。

実装上の条件は次です。

* fu > 2 かつ han >= 5 は符が点数に影響しないため、符インデックス2へコピーします。
* ツモ側のピンフ符、七対子符、1翻七対子ロンなど、あり得ない翻符の組合せは0にします。
* 自分がこれからリーチする候補を評価する時は reach_mode=true の別テーブルを作り、未受理1000点を含めます。

### 3.2 流局時テーブル

cal_ryuukyoku_pt_exp() は4人の形式聴牌フラグを全16通り列挙します。各パターンについて ten_move_ryukyoku() の点棒移動、リーチ未受理補正、親の連荘、順位点期待値を計算します。DP の終端はこの表を使い、自分の聴牌・ノーテンを分けます。

## 4. 手牌解析

### 4.1 内部状態

Tehai_Analyzer_Basic は次を持ちます。

* Bit_Hai_Num tehai_bit: 牌種ごとの枚数。
* Tehai_State2 tehai_state: 副露、リーチ、赤5の内外。
* num_and_flags: 手牌枚数、副露数、暗槓数、通常形・七対子・総合シャンテン、聴牌・フリテン等をビットフィールドに詰めた値。

Bit_Hai_Num は色ごとに32bit整数を持ち、各牌を3bit（0..7枚）で表します。赤5は黒5の枚数へ入れず、Tehai_State2 の赤5内側フラグで表します。Tehai_Change の等価判定も id を無視してこの牌枚数だけを比較するため、同じ正規化手牌は重複候補になりません。

牌種 h の使用枚数は次です。

~~~text
using(h) = 手牌中の h + 副露状態が消費した h
~~~

副露状態の消費数は、ポン3、明槓4、暗槓4、該当する順子の出現数で加算します。

### 4.2 通常形の再帰分解

analyze_tehai() は正規化手牌を次の順で再帰分解します。

1. 各牌を雀頭候補として2枚取り除く。
2. cut_kotu() で刻子を1組ずつ取り除く。
3. cut_syuntu() で順子を1組ずつ取り除く。
4. cut_tatu() で対子、両面、嵌張のターツを取り除く。
5. 余った牌を孤立牌として記録し、通常形シャンテンを更新する。

基本値は次です。

~~~text
normal = 8 - 2 × (完成面子数 + 副露数) - ターツ数 - 雀頭数
~~~

4面子（手牌の完成面子＋副露）が揃ったとき、残りに「まだ4枚見えていない単騎候補」がある場合は上の値をそのまま使い、それ以外の余り1枚条件では実装の補正により normal += 1 となる場合があります。これは cut_tatu() の mentu_num + fuuro_num == 4 部分で、余りの1枚が tehai_kcp[hai] == 4 かどうかを検査しているためです。

tehai_kcp は赤5を黒5へ移した配列です。解析の途中で tehai_tmp の残り枚数を調べ、次の待ち形を検査します。

* 残り0枚: 完成形。
* 残り1枚: 単騎待ち。
* 残り2枚: シャンポン、両面、嵌張、辺張を列挙。

待ち牌が4枚使い切られている場合は待ちから除外します。1つでも待ちが見つかると聴牌フラグを立て、通常形シャンテンを0にします。

### 4.3 七対子

副露がない時だけ七対子を別ルートで解析します。牌種単位の対子種類数を pair、1枚だけ存在する牌種の数を isolated とすると、次です。

~~~text
chiitoi = max(0, 13 - 2 × pair - min(7 - pair, isolated))
~~~

対子種類が6で孤立牌がある時は、孤立牌ごとに単騎待ちの和了情報も生成します。最終的な総合シャンテンは次です。

~~~text
shanten = min(normal, chiitoi)
~~~

### 4.4 和了情報の生成

各待ちについて agari_push_func() が calc_agari() を呼び、翻・符・待ち牌・待ち形を Agari_Info にします。

* リーチ状態ならツモ・ロン双方へ1翻を加算します。
* 手牌、非暗槓副露、ドラ表示牌、赤5、和了牌自身がドラの場合を数えます。
* 役が0翻の形にはドラを足しません。
* Agari_Info_Bit はツモ翻、ロン翻、ツモ符インデックス、ロン符インデックス、和了牌、ドラ数を32bitへ詰めます。ただし現在の agari_info_to_agari_calc() 経路ではドラ数フィールドは設定されません。

同じ待ち牌について10件目に達した時は、より点数の高いツモ・ロン情報を残して重複を整理します。候補グラフ側の Agari_Calc 格納上限は Windows で100,000件、それ以外で200,000件です。

和了時の順位点期待値は Agari_Basic::get_ten_exp() で求めます。リーチしていない場合の裏ドラ分布は0枚だけです。リーチ済みの場合、見えていない牌の枚数を重みとして裏ドラ枚数0..12の分布を作り、翻13を上限に和了テーブルへ畳み込みます。戻り値4要素は概念的に次です。

~~~text
[通常牌ツモ, 通常牌ロン平均, 赤5ツモ, 赤5ロン平均]
~~~

ロン値は3人の相手について平均します。直近ツモの赤5、またはロンされた赤5は赤ドラ1枚分を含む要素を選びます。get_ten_exp_direct() は対象者を固定し、通常牌・赤5の2要素を返します。

## 5. 役の見込みと手替わりパターン

### 5.1 Tehai_Pattern の表現

探索用のパターンは、完成面子、雀頭、ターツ、孤立牌を Tehai_Pattern_Source に分けます。Tehai_Pattern は最大4面子と雀頭1つを目標として、各ブロックを完成させるための入力牌列を作ります。

空の面子ブロックには、各色の123..789順子、111..999刻子、字牌刻子を候補として入れます。1枚・2枚の不完全ブロックには、順子の不足牌、対子化する同牌、両面・嵌張の不足牌を入れます。雀頭ブロックが空なら全牌種の対子、1枚だけなら同牌1枚を候補にします。

cal_hai_in_pattern() は5ブロックの不足牌候補の直積を列挙し、次を除きます。

* remain（今の形で余る牌）と同じ牌を入力に要求するもの。
* 現在の手牌・副露と合わせて牌種4枚を超えるもの。
* シャンテン3以上で、雀頭ブロックが2枚必要になるもの。

各パターンは hai_in_pattern（将来引き入れる牌）と hai_out_pattern（外へ出る牌）へソートして保存します。

七対子パターンは、既存の対子を最大7組まで選び、1枚牌を将来の対子候補へ変換します。欠けた対子は互いに異なる牌種の組合せとして列挙され、同じ候補は Tehai_Change の集合で重複除去されます。

### 5.2 手替わり上限と優先度

非リーチ時の通常形手替わり上限は次です。

~~~text
mentu_change_num_max = normal_shanten + tegawari_num[normal_shanten]
~~~

デフォルトの tegawari_num は [2,2,1,1,0,0,0] です。七対子側の上限は cal_titoi_change_num_max(titoi, normal) で決まり、概念的には次です。

~~~text
if titoi <= normal:
    titoi <= 2 なら titoi+1、それ以外は titoi
else if titoi+1 <= normal:
    titoi
else:
    0
~~~

パターンの優先度は cal_priority() が計算します。

~~~text
priority
 = 2 × honitsu候補
 + 2 × toitoi候補
 + 1 × tanyao候補
 + 役牌候補数
 + ∏h C(4 - visible_kind[h], input_count[h]) / 4^(input枚数)
~~~

honitsu_check_pattern() は字牌を許す一色、tanyao_check_pattern() は全ブロックと入力が中張牌だけ、toitoi_check_pattern() はブロック・雀頭・入力に現れる牌種数がちょうど5か、で判定します。yakuhai_check_pattern() は場風、自風、三元牌の刻子を数え、場風と自風が同じ場合などは二重に加点され得ます。

yaku_dist.cpp の calc_yaku_dist() は役までの距離を別の尺度で計算する補助関数です。役牌、混一色、断么九、混全帯么九、対々和、三色同刻、三色同順、一気通貫を計算します。調査した現行 selector・候補生成経路からこの関数への呼び出しはなく、上記の手替わり優先度とは別系統です。再実装時に calc_yaku_dist() を暗黙に足すと現行 AI と一致しません。

## 6. 候補手牌グラフの構築

### 6.1 根候補

Tehai_Calculator::set_candidates3_single_thread() は正規化した現在手牌を根にします。

| 現在の有効枚数（手牌＋副露×3） | 根候補 |
|---:|---|
| 14 | 手牌の各牌を1枚抜いた「打牌後」候補 |
| 13 | 現状維持、および1枚引いて1枚切る候補 |

13枚状態の1枚交換は、未リーチなら牌種ごとの入力上限4枚を確認してから、手牌中の全打牌牌と組み合わせます。リーチ中の13枚状態は現状維持の根だけです。14枚状態の根候補は in0num として数えられ、最終の自分ツモ時候補の起点になります。

パターン経由の候補は、入力枚数 nin=1..sn と出力枚数

~~~text
nout = tehai_all_num - 13 + nin
~~~

の組合せを列挙します。したがって13枚状態なら nout=nin、14枚状態なら nout=nin+1 です。候補の正規化牌集合を unordered_set に入れ、同一手牌を1つにします。

fuuro_cand_hai がある他家打牌時は、現在の捨て牌を入力パターンへ一時追加する形も試します。入力と出力に同じ牌種があるパターンは鳴き替えの不整合になるため除外します。

### 6.2 列挙制限

デフォルト設定では、次の上限・段階があります。

* 全候補数 MAX_CANDIDATES_NUM = 200,000。
* 1つの Tehai_Group が持つ解析状態 ta_loc は最大128。
* 1スレッドの解析状態は最大100,000。副露展開時は10,000件をリーチ用に残します。
* DP用のツモ回数配列はパリティ2要素で保存し、MAX_TSUMO_NUM=20 を想定します。
* デフォルトのツモ候補数制限は20,000。tsumo_enumerate_always=-1 なので、候補がこの制限に達するまでは各シャンテンのパターンを通常列挙します。
* enumerate_restriction=-1、enumerate_restriction_fp=-1 なので、副露ノード側も候補数制限を設定しないのがデフォルトです。

上限に達した場合、該当関数はアラートを出さず、その先の候補を追加しない実装です。cn_max_addition による追加列挙はデフォルト0です。

### 6.3 グループと赤5

1つの Tehai_Group は同じ正規化手牌を表し、ta_loc に赤5の内外や副露・リーチ状態が異なる Tehai_Analyzer_Basic を持ちます。set_tav_init() は根との差分を入力・出力へ分け、赤5について次を選びます。

* 黒5入力があり、鳴き牌が赤5なら黒5入力と赤5入力の両方。
* 赤5を手牌から出せる時、黒5を出す場合と赤5を出す場合の両方（元の黒5が残る場合）。
* 赤5しか出せない時は出力を赤5へ固定。

状態の重複は Tehai_State2 をキーにした map で抑えます。

### 6.4 副露・槓ノード

add_fuuro_node() は現在ノードから相対的に最大3副露（デフォルト）まで、ポン・チーを再帰追加します。副露展開では次の3種類を順番を変えて試します。

1. 現在の有効牌を必ず鳴く経路。
2. fuuro_must を強制しないが、有効牌を優先する経路。
3. 有効牌条件を付けない経路。

consider_kan=true の時は、暗槓・加槓・大明槓もグラフへ追加します。暗槓・大明槓は元の手牌で対象牌種が3枚かつ副露状態を含めた使用数が3枚の時だけ、加槓は既存ポンと手牌の1枚が対応する時だけ作ります。暗槓の追加は fuuro_num_max を超えて現れることがあります（コードコメントにもある実装上の都合です）。

全候補状態について一度聴牌解析、次に全和了解析を行います。自分の河に既に捨てた牌、または候補の変換で自分が使用している待ち牌については、候補状態のフリテンフラグを立てます。

### 6.5 遷移辺

set_tsumo_edge() は、ある候補から牌 ho を切って牌 hi を入れる辺を作ります。最初は hi > ho の片方向だけを保存し、後で逆方向も登録するため、最終的には両方向の交換として参照できます。in_flag[hi]、または根から現在候補を作る過程で入れ替えた牌だけを入力側に許します。

Tehai_Action の主要フィールドは次です。

~~~text
hai          引いた牌・鳴いた牌
hai_out      その後の打牌
action_type  ツモ交換、ポン、チー、暗槓、加槓など
dst_group    移動先の候補グループ
dst_group_sub 移動先グループ内の状態
~~~

主要な Action_Type の数値は次です。赤牌を含むポン・チーには100番台の派生値がありますが、候補グラフ内部では通常のポン・チー値と状態内の赤5フラグを使います。

| 値 | Action_Type |
|---:|---|
| 0 | AT_TSUMO |
| -1 | AT_DAHAI |
| 2 | AT_PON |
| 3 | AT_DAIMINKAN |
| 4 | AT_ANKAN |
| 5 | AT_KAKAN |
| 11, 12, 13 | AT_CHI_LOW, AT_CHI_MIDDLE, AT_CHI_HIGH |
| 19 | AT_REACH_DECLARE |
| 50, 51 | AT_TSUMO_AGARI, AT_RON_AGARI |
| 53, 54 | AT_FUURO_PASS, AT_KYUSHUKYUHAI |

遷移作成時に以下を拒否します。

* 現在状態がリーチ中で、暗槓以外の行動。
* 暗槓・加槓以外で、鳴き牌と同じ牌種を直後に捨てる行動。
* チーの低位・中位・高位に対する食い替え（例: 1-2-3を1で鳴いて4を切る等、牌種差で判定されるもの）。
* 鳴きに使った赤5を直後に赤5として出す行動。

遷移先が聴牌、かつ暗槓を除く副露が0、未リーチの場合は、同じツモ・暗槓辺からリーチフラグ付き状態への辺も追加します。これにより「この牌を引いた後にリーチする」という選択を別状態として評価します。

### 6.6 Tehai_State2 の符号化

Tehai_State2 は、解析状態を次の整数群へ圧縮します。

| フィールド | 内容 |
|---|---|
| chi_int[3] | 各色の順子開始位置ごとに3bitで個数 |
| pon_kan_int[4] | 各牌種のビット0=ポン、1=暗槓、2=明槓（各3bit幅） |
| reach_aka_int bit0 | リーチ |
| 同 bit1..3 | 萬・筒・索の赤5が手牌内 |
| 同 bit4..6 | 萬・筒・索の赤5が副露外側 |

get_fuuro() は、現在状態と比較対象状態の差分から Fuuro_Vector を復元します。赤5外側フラグがある場合は、最初に見つけた該当副露牌へ赤5を割り当てます。

## 7. ルールベース分岐

探索量を抑えるため、Tehai_Analyzer_Basic::rule_base_decision() は次の時に真を返します。

~~~text
未リーチ かつ
  総合シャンテン >= 4
  または 通常形シャンテン >= 5 かつ 七対子シャンテン >= 3
~~~

Tactics::use_rule_base_at_mentu5_titoi3_shanten という設定フィールドは存在しますが、この判定箇所では条件がコメントアウトされており、現行コードでは値を見ません。

### 7.1 自分のツモ

九種九牌設定が有効なら、合法性を確認して九種九牌を最優先します。そうでなければ、他家の誰かがリーチ中、または推定聴牌率が0.5を超える場合にベタオリ候補を全打牌牌について計算します。

それ以外は koritu_most_needless() の最初の牌を1枚切ります。選択順位は概ね次の通りです（各段階で最初に見つかった牌を返します）。

1. 見え3枚または2枚の孤立字牌。
2. 場風・自風・ドラでない、見え1枚または0枚の孤立字牌。
3. 自風・ドラをなるべく残した孤立字牌。
4. 端牌の孤立牌（隣接牌がない形）。
5. 孤立2・8。
6. 最後は牌番号37から1へ逆順に、手牌にある牌。

ドラ判定は表示牌から dora_marker_to_dora() で得た牌種を使います。ベタオリ候補の pt_exp_total は、直近打牌の放銃価値を含めて次で計算します。

~~~text
total_houjuu_hai_prob_now[h] × total_houjuu_hai_value_now[h]
 + (1 - total_houjuu_hai_prob_now[h]) × pt_exp_after_ori[h]
~~~

### 7.2 他家の捨て牌・槓

高シャンテンの他家打牌応答では、鳴かず AT_FUURO_PASS を1件だけ返します。以降の DP/非DP分岐では、他家の捨て牌時に鳴き候補を評価します。

## 8. 他家推定、放銃リスク、ベースライン

Selector は他家4人（自分を含む）の Tenpai_Estimator_Simple を作り、次の配列を得ます。

* tenpai_prob[pid]: 現在の聴牌確率。
* houjuu_hai_prob[pid][h]: 相手 pid に牌種 h で放銃する確率。
* houjuu_hai_value[pid][h]: その牌で放銃した時の順位点期待値。
* 翻符分布: ツモ用・ロン用の han × fu 分布。

全相手の牌別リスクは、相手の聴牌率を掛けた後に相手間で合算します。相手 pid の牌 h について、han・fu 分布から次を作ります。

~~~text
q_pid(h) = Σhan,fu P(pid が h で和了, han, fu)
v_pid(h) = Σ P(pid が h で和了, han, fu) × 局終了後順位点
            / q_pid(h)       （q_pid(h)>0 の時）
~~~

現在時点の合算は次です。分母が0なら total_value[h] は0です。

~~~text
raw_p[h] = Σpid≠me tenpai_prob[pid] × q_pid(h)
total_p[h] = min(1, raw_p[h])
total_value[h] = Σpid≠me tenpai_prob[pid] × q_pid(h) × v_pid(h)
                 / raw_p[h]
~~~

total_value は、確率を1へ丸める前の raw_p を分母に使う点に注意してください。DP の将来時点では同じ q_pid(h)、v_pid(h) に tenpai_prob_other[pid][tn]（リーチ経路では reach_tenpai_prob_other）を掛け、houjuu_p_hai[tn][h] と houjuu_e_hai[tn][h] を作ります。相手間の独立性による「少なくとも1人」の補正や、DP内の将来リスクの上限処理はありません。

### 8.1 他家の将来聴牌率と局終了率

set_tenpai_prob_other() は DP の残りツモ数ごとに他家聴牌率を作ります。

* tenpai_after_est="instant": 現在値を全時点で維持。
* "ako": 現在河枚数と将来河枚数でパラメータファイルを選び、[1, logit(現在聴牌率)] をロジスティック回帰へ入力。
* リーチ済み相手は常に1.0。
* 他家に1人でもリーチ者がいる場合、現行コードは非リーチ相手にも reach_tenpai_prob_other を使用する。

set_other_end_prob() は「その巡までに他家が和了して局が終わる」確率を作ります。

* other_end_prob_est="instant": 全時点0.1。
* "ako": 非リーチ他家の少なくとも1人が聴牌する確率

~~~text
1 - ∏pid≠me,非リーチ (1 - tenpai_prob[pid])
~~~

と他家リーチ人数、現在・将来河枚数を3特徴としてロジスティック回帰します。

DP のこの経路では Tactics::other_end_prob_max の上限処理は実装されていません。

### 8.2 ベタオリの局価値

cal_exp() は非DP候補のベースラインを5結果（自分の和了、他家3人の和了、流局）で計算します。概念的には次です。

~~~text
my_agari_prob = 補正後の自分の和了確率
my_agari_value = (value_sol - (1 - agari_prob_sol) × value_not_agari) / agari_prob_sol
ryuukyoku_value = 4人の形式聴牌パターンの期待値
E = Σ5結果 P(result) × Value(result)
~~~

agari_prob_sol==0 の時は my_agari_value=0 です。序盤の門前ツモで他家リーチも副露もない条件では、生の agari_prob_sol をそのまま使う条件分岐があります。それ以外では agari_prob_est に応じて、固定0.5倍またはロジスティック回帰で補正します。

## 9. DP を使うかどうか

cal_dp_flag() は現在の総合シャンテンだけでなく、「直近捨て牌を鳴いた後の最良シャンテン」も判定します。デフォルト閾値は次です。

| 条件 | シャンテン以下ならDP |
|---|---:|
| 常時 | 1 |
| 他家リーチ中 | 2 |
| 他家打牌への応答（フーロ局面） | 1 |
| 他家リーチ中かつフーロ局面 | 2 |

現在手には4条件を適用します。fuuro_agari_shanten_num には常時の閾値、他家リーチ時の閾値、フーロ閾値、他家リーチかつフーロ時の閾値を順に適用します（実装ではフーロ閾値のチェックにフーロ局面フラグを要求しません）。fuuro_agari_shanten_num は、直近牌を鳴く各遷移先の和了シャンテンの最小値です。候補がない場合の初期値は8です。

### 9.1 DP の残りツモ回数

cal_tsumo_num_DP() は山の進行を固定70ツモとして、次を計算します。

~~~text
all = count_tsumo_num_all(record)
base = (70 - all) / 4          // 整数除算
next = last action が dahai なら next_player(actor,1)
       それ以外なら next_player(my_pid,1)
if ((70-all)%4 > (4 + my_pid - next)%4): base += 1
~~~

この値は0以上であることをアサートします。dahai 以外の直近イベントは一律に my_pid を基準にするため、他家の kakan 直後などは実際の嶺上ツモ順と異なる可能性があります。

## 10. DP の状態、終端、再帰

### 10.1 状態配列

DP の手牌状態は Tehai_Group のグループ番号 cn、グループ内状態番号 gn で指定します。値は Tehai_Calculator_Work に保存されます。

| 値 | 意味 |
|---|---|
| agari_prob | 残りツモで自分が和了する確率の推定 |
| agari_exp | 和了に関する順位点期待値成分 |
| agari_han_prob[5] | 翻分布（4翻以上は添字4へまとめる） |
| tenpai_prob | 残り時点で聴牌している確率 |
| ten_exp | 和了・放銃・他家和了・流局を混ぜた総合期待値 |
| ori_exp | その状態からベタオリした期待値 |

[2] の2要素しか持たず、ツモ回数 tn の偶奇（tn % 2）で再利用します。to_work は次の時点を計算する一時配列です。

### 10.2 終端 tn=0

各状態を次で初期化します。

~~~text
agari_prob[0] = 0
agari_exp[0] = 0
tenpai_prob[0] = 状態の聴牌フラグ
~~~

ten_exp[0] は、リーチ状態ならリーチ後流局期待値、通常の聴牌なら聴牌流局期待値、ノーテンならノーテン流局期待値です。DP はここから tn=1..tsumo_num へ値を作りますが、各 tn の遷移参照は tn-1 の状態なので、意味としては「残り回数を1つずつ増やす後退解析」です。

### 10.3 1巡目のツモ・打牌選択

各牌 h について、最初に「何もしない現状態」と「候補グラフのツモ辺」を比較します。相手に放銃する牌を切る場合は、次の値を使います。

~~~text
discard_value(h_out)
 = (1 - p[h_out]) × next_ten_exp
   + e[h_out]
~~~

ここで p[h_out] はその時点の総放銃確率、e[h_out] は p × 放銃時順位点です。リーチ状態で reach_regression_mode_default==1 なら、リーチ者用の reach_houjuu_* 配列を使います。

直接和了辺がある場合は、agari.tsumo_exp が現在の牌に対する最大値を超えた時に、その牌の値を和了値へ置き換えます。通常のツモ辺は、引いた h、切った h_out、遷移先 dst を使い、

~~~text
(1 - p[h_out]) × ten_exp[dst, tn-1] + e[h_out]
~~~

を比較します。遷移先が neg_flag なら候補にしません。各 h について最も ten_exp が高い行動を残すため、agari_prob は「最適な打牌を選び続けた時の和了確率」です。

### 10.4 牌山での平均

状態 s における牌 h の残り枚数は、コード上次です。

~~~text
remain_s(h) = 4 - using_s(h) - visible_without_my_hand[h]
if using_now(h) > using_s(h):
    remain_s(h) -= using_now(h) - using_s(h)
~~~

分母は次です。

~~~text
denom = nokori_hai_num - (tsumo_num - tn)
~~~

ツモ辺を処理する第一段階では、各牌の最良値そのものを remain_s(h)/denom で加重平均します。ロン・鳴き辺を処理する第二段階では、第一段階の平均値との差分だけを同じ重みと係数で加えます。実装は remain_s(h) を明示的に0以上へ丸めていません。候補生成が4枚制約を守ることを前提にしています。

### 10.5 ベタオリ置換

各 tn の平均後、次を満たす状態では攻撃値をベタオリ値へ置き換えます。

~~~text
ori_choice_mode > 0
未リーチ
ori_exp > ten_exp
かつ
  use_ori_exp_at_dp_fuuro == true
  または 状態の副露数 == 現在の副露数
~~~

置換時は agari_prob、agari_exp、tenpai_prob を0、ten_exp=ori_exp とします。セレクタは DP 時の ori_choice_mode を2に固定しています。

### 10.6 鳴き・ロン選択

ツモ処理後、各状態についてロンと鳴きの辺を同じ方式で比較します。

* フリテン状態ではロン辺を作らない。
* 和了情報のロン翻が正で、和了牌の使用残数条件を満たす時だけロン辺を作る。
* ポン、チー、大明槓は遷移先の tn-1 の値を使い、直後に切る牌の放銃リスクを掛ける。
* ポンまたはロンを選べる牌の差分には pon_ron_flag に応じた係数を付ける。

デフォルト設定の ron_ratio_est="ako" では、門前ダマの牌種別固定比、またはリーチ状態・自分の聴牌率・他家聴牌率・牌別放銃率を入力にしたロジスティック回帰のオッズを使います。ron_ratio_est="instant" の場合の係数は次です。

~~~text
coeff = 1 + 2 × pon_ron_flag
~~~

直接ロン候補では pon_ron_flag と ron_flag を1にし、ポンまたは大明槓の改善辺では pon_ron_flag を1にします。

最後に他家和了終了を混ぜます。

~~~text
ten_exp = other_end_prob × other_end_value
          + (1 - other_end_prob) × ten_exp_without_other_end
agari_prob *= (1 - other_end_prob)
tenpai_prob *= (1 - other_end_prob)
agari_exp *= (1 - other_end_prob)
~~~

状態がリーチ中なら other_end_value_ar、暗槓後なら other_end_value_kan を選びます。

### 10.7 流局終端の組合せ

set_exp_ryuukyoku_DP() は、他家3人の聴牌・ノーテンフラグを8通り列挙し、独立積で重み付けします。

~~~text
P(flag) = ∏pid≠me
  (flag[pid] ? tenpai_prob_other[pid][0]
             : 1 - tenpai_prob_other[pid][0])
~~~

自分のフラグが0/1ごとに exp_ryuukyoku[0/1] へ足し、リーチ状態で自分のフラグが1なら reach_tenpai_prob_other と ryuukyoku_pt_exp_ar を使って exp_ryuukyoku_ar を計算します。

## 11. 非DPの和了確率計算

DP条件を満たさない場合も候補グラフは使いますが、calc_agari_prob() は放銃と他家和了の巡中リスクを直接混ぜません。

1. tn=0 で聴牌フラグと exp_min を初期化。
2. 各 tn で、直接ツモ和了の点数が現在値を超えれば、その牌の和了確率を1へ置く。
3. ツモ辺の遷移先がより高い聴牌率・期待値なら、その牌の値を更新。
4. remain_s(h) / denom で牌山平均を取り、次の tn へコピー。

rn_para は初期化値1.0で、最終ツモ以外の残り枚数へ掛かります。現行の指定ファイル内では値を変更する経路がありません。

非DPのセレクタは、候補ごとの agari_prob、ten_exp、tenpai_prob を cal_exp() へ渡します。cal_exp() は自分の和了、他家和了、流局、放銃者分配、順位点を再計算して pt_exp_after を作るため、単純な ten_exp 比較ではありません。

## 12. ベタオリの計算

### 12.1 危険牌の並べ替え

Betaori は手牌を牌種ごとにまとめ、牌種 h の所持枚数を n として危険度を作ります。houjuu_decay=0.9 固定です。

放銃確率 p、放銃時価値 Vh、降りたままの価値 Vo に対し、牌種の並べ替え係数は次です。

~~~text
beta = 1 - 0.9^n
risk_coeff = p × (Vo - Vh) / (p + beta - p×beta)
~~~

係数の昇順（同値なら所持枚数の昇順）に、安全そうな牌から切ります。必要な ori_num 枚数へ達するまで順に処理し、途中まで放銃しない確率を更新します。

~~~text
P(放銃) += P(ここまで無事) × p[h]
E(放銃) += P(ここまで無事) × p[h] × Vh
P(ここまで無事) *= (1-p[h]) × 0.9^n
~~~

最後に、残った非放銃確率へ other_value を掛けます。

### 12.2 流局補正

betaori_est="ako" では、河枚数を2..18へ丸めて3重みのロジスティック回帰を読み、次から放銃確率を補正します。

~~~text
[1, 副露数, logit(betaori_houjuu_prob)]
~~~

補正後の非放銃部分には、次を掛けます。

~~~text
ryuukyoku_prob × noten_ryuukyoku_value
 + (1-ryuukyoku_prob) × other_value
~~~

betaori_est="instant" は補正を行わず、other_value=not_agari_value を使います。

DP 中は ori_exp としてこの計算を状態ごとに行い、非DPの自分ツモでは他家リーチが1人以上、または他家の最大副露数が2以上なら、攻撃値とベタオリ値の大きい方を採用します。高シャンテンのルールベース分岐でも、他家の攻撃がある場合は各打牌のベタオリ値を比較します。

## 13. 自分のツモ時の候補と採点

### 13.1 DP 時

in0num の根候補と、現在副露と同じ副露状態の gn について次を作ります。

1. 現在ツモ牌が和了待ちに一致し、翻（ハイテイ1翻を含む）が正なら AT_TSUMO_AGARI。
2. 候補状態との差分を find_hai_out_ta() で求め、通常打牌を作る。
3. その状態がリーチフラグ付きなら AT_REACH_DECLARE にする。
4. ツモ辺から暗槓・加槓を探す。

ツモ和了候補は「現在ツモ牌の牌種が和了情報の牌種と一致」「和了翻＋ハイテイ翻が正」「候補状態のリーチフラグと現在のリーチ受理状態が一致」の全てを満たす必要があります。赤5ツモなら get_ten_exp()[2] を使います。

暗槓・加槓は総ツモ数70未満で、DP内の do_ankan_inclusive / do_kakan_inclusive が有効な時だけ候補へ入れます。リーチ受理後の暗槓は Ankan_After_Reach::check() を通します。

通常打牌・リーチ・加槓の採点は直近の牌別放銃リスクを混ぜます。

~~~text
pt_exp_total
 = total_houjuu_hai_prob_now[h] × total_houjuu_hai_value_now[h]
 + (1-total_houjuu_hai_prob_now[h]) × pt_exp_after
~~~

暗槓とツモ和了は切る牌がないため pt_exp_after をそのまま pt_exp_total にします。

### 13.2 非DP 時

候補の agari_prob 等を cal_exp() へ渡して pt_exp_after を計算し、暗槓を除く候補に同じ直近放銃リスクを掛けます。設定 do_kan_ordinary が有効な場合だけ、通常探索でも暗槓・加槓を比較します。

他家リーチがある、または他家最大副露数が2以上なら、各候補の pt_exp_after_ori と攻撃側 pt_exp_after の大きい方を先に採用します。

## 14. 他家の捨て牌・加槓時の候補と採点

### 14.1 DP 時

現在の手牌と同じ cn/gn を見つけ、次を比較します。

* ロン: フリテンでなく、現在牌が待ち、ロン翻または槍槓・ハイテイの加算翻が正。
* パス: max(DP ten_exp, not_agari_value)。
* ポン・チー・大明槓: 現在牌に対応する候補グラフ辺。

ロンは同巡フリテンも調べます。待ちのいずれかが get_furiten_flags(..., true) で真なら、現在牌が当たりでもロン候補を出しません。赤5ロンは get_ten_exp_direct()[1] を使います。

直近が kakan の場合は incident_han に槍槓1翻を足し、ロンとパスだけを比較します。直近の山の最後（count_tsumo_num_all()==70）ならハイテイ1翻も足します。

直近の山の最後でない dahai なら、鳴き辺を現在牌で検索します。チーは、捨てた相手の下家（my_pid == next_player(actor,1)）の時だけ残します。鳴き後の残りツモ数を max(tsumo_num_DP-1,0) として pt_exp_after を計算し、鳴き前の残りツモ数による pt_exp_after_prev も計算します。

~~~text
after_now = p(hai_out) × V_houjuu(hai_out)
          + (1-p(hai_out)) × pt_exp_after
after_prev = p(hai_out) × V_houjuu(hai_out)
           + (1-p(hai_out)) × pt_exp_after_prev
pt_exp_total = min(after_now, after_prev)
~~~

### 14.2 非DP 時

パスは cal_exp() で評価します。直近が dahai で総ツモ数70未満なら、ポン・チー・大明槓を候補にし、鳴き後は fuuro_inc=1、dahai_inc=1 で cal_exp() を呼びます。鳴き候補の pt_exp_total は DP と同じく after と after_prev の小さい方です。

非DP分岐では明示的なロン候補を生成しません。直近 kakan の非DP分岐もパスだけです。これは高シャンテン側を非DPにする探索設計による限定です。

## 15. リーチ後暗槓の条件

Ankan_After_Reach::check() は、ツモ牌を一時的に加えて暗槓可否を判定します。

1. 字牌なら真。
2. 数牌なら対象牌が4枚でなければ偽。
3. 周囲に牌がない単純な場合は真。
4. それ以外は暗槓前後の面子・雀頭・ターツ分解を再帰列挙し、構成コードが不変なら真。
5. 九蓮宝燈リーチ後に1・9を暗槓する特殊形は偽。

暗槓前後の構成コードは、刻子・順子・雀頭・ターツに加えた番号の和で比較します。チェッカー単体は字牌を無条件可としており、呼び出し側が候補グラフで「4枚そろっている」ことを先に確認する設計です。

## 16. 行動の JSON 化

候補の内部型と外部イベントは次の対応です。

| 内部候補 | out_moves() のイベント |
|---|---|
| AT_DAHAI | dahai 1件 |
| AT_REACH_DECLARE | reach → dahai |
| AT_TSUMO_AGARI | hora(actor=my_pid,target=my_pid) |
| AT_KYUSHUKYUHAI | type=ryukyoku, reason=kyushukyuhai |
| AT_ANKAN | ankan（赤5があれば 5mr/5pr/5sr を含む） |
| AT_KAKAN | kakan |
| AT_RON_AGARI | hora(actor=my_pid,target=捨て牌者) |
| AT_FUURO_PASS | none |
| ポン | pon → dahai |
| チー | chi → dahai |
| 大明槓 | daiminkan 1件 |

ポン・チーの sarashi_hai は、Tehai_State2 の現在副露との差分から、直近牌以外の2枚を復元します。JSON は牌文字列へ戻します。

~~~json
[
  {"type":"reach", "actor":0},
  {"type":"dahai", "actor":0, "pai":"5p", "tsumogiri":false}
]
~~~

~~~json
[
  {"type":"pon", "actor":0, "target":2, "pai":"7s", "consumed":["7s","7s"]},
  {"type":"dahai", "actor":0, "pai":"1m", "tsumogiri":false}
]
~~~

九種九牌は専用の type ではなく、流局イベントの reason で表します。

~~~json
{"type":"ryukyoku", "reason":"kyushukyuhai", "actor":0}
~~~

リーチ・ポン・チーが2イベントになるのは MJAI 側が1件目を受理してから次の打牌を受け取るプロトコルを前提にしているためです。

ai_review() の戻り値は、候補ごとに次の形の JSON 配列です。

~~~json
[
  {
    "moves": [{"type":"dahai", "actor":0, "pai":"1m", "tsumogiri":false}],
    "review": {
      "total_houjuu_hai_prob_now": 0.12,
      "total_houjuu_hai_value_now": -35.0,
      "pt_exp_after": 18.4,
      "pt_exp_total": 12.0
    }
  }
]
~~~

放銃確率が0の候補では total_houjuu_hai_value_now と pt_exp_after を review に入れません。calc_moves_score() は同じ候補を JSON ではなく (Moves, pt_exp_total) の配列で返します。

## 17. 最小再実装手順（擬似コード）

以下で、現行の分岐順と候補の絞り込みを再現できます。

~~~text
function decide(record, me):
    state = replay_from_last_start_kyoku(record)
    action = record[-1]
    assert action is me.tsumo or other.dahai or other.kakan
    current = parse_hai(action.pai)

    build kyoku_end_pt_exp and ryuukyoku_pt_exp
    ta = Analyzer(state.me.tehai, state.me.reach, state.me.fuuro)
    if state.me.reach and action.type == tsumo:
        ta.remove(current)
    ta.analyze_tenpai(state)

    if ta.rule_base_decision():
        if action.type == tsumo:
            if legal_kyushukyuhai and setting.do_kyushukyuhai:
                return kyushukyuhai
            if other_reach or any(other.tenpai_prob > .5):
                return argmax_over_discards(betaori_with_current_tile_risk)
            return first_koritu_most_needless()
        return none

    set hand-change maxima
    ta.generate_inout_patterns()
    calc = Calculator(me, aka_flag)
    calc.make_root_candidates(13/14 state)
    calc.make_tsumo_candidates_and_titoi_candidates()
    calc.make_fuuro_kan_reach_states()
    calc.analyze_tenpai_and_agari_all_states()
    calc.make_tsumo_edges_and_fuuro_edges()

    fuuro_shanten = best_shanten_after_current_discard_or_8
    if dp_enabled(ta.shanten, fuuro_shanten, other_reach, action.type == dahai):
        tn = remaining_draw_count(record, me)
        calc.dp_with_discard_risk_other_end_and_betaori(tn)
        if action.type == tsumo:
            candidates = make_discard_reach_tsumo_agari_kan_choices()
        else:
            candidates = make_ron_pass_pon_chi_daiminkan_choices()
    else:
        tn = min(statistical_draw_count, remaining_draw_count)
        calc.non_dp_agari_prob(tn)
        if action.type == tsumo:
            candidates = make_discard_reach_and_optional_kan_choices()
        else:
            candidates = make_pass_and_optional_fuuro_choices()

    for c in candidates:
        c.pt_exp_total = combine_current_discard_houjuu(c)
    sort descending by pt_exp_total
    return c[0].out_moves()
~~~

## 18. 具体例

### 18.1 通常形シャンテン

次の13枚を考えます。

~~~text
123萬 456萬 789萬 12筒 5索 東
~~~

完成面子3組、両面ターツ1組、孤立牌2枚なので、通常形の式は

~~~text
8 - 2 × (3 + 0) - 0 - 1 = 1
~~~

となり1シャンテンです。例えば 3筒を引いて東を切ると123筒が完成し、4面子＋孤立牌となる0シャンテンです。この交換は13枚状態の nin=1、nout=1 の候補になります。見えている3筒が2枚なら、牌山重みは「4 - 2 - 現在の使用枚数」で決まり、同じ最終手牌を作る別経路は Tehai_Change の集合で1件にまとめます。

### 18.2 七対子との比較

牌種単位で対子が4種類、孤立牌が5種類なら、七対子式は

~~~text
13 - 2 × 4 - min(3, 5) = 2
~~~

です。通常形が1シャンテンなら総合値は min(1,2)=1 となり、通常形が優先されます。七対子パターンを追加するのは、副露がなく、titoi_change_num_max の範囲に入る場合だけです。

### 18.3 DP の1牌比較

ある残り時点で、牌 A を引いた後に切る牌の放銃確率が0.10、放銃時価値が -80、無放銃時の遷移先 ten_exp が55だとします。houjuu_e_hai は確率込みで -8 なので、

~~~text
0.90 × 55 - 8 = 41.5
~~~

です。別の安全な遷移の値が40なら、Aについては前者を残します。その後、他家和了終了率が0.20、他家和了時価値が-20なら、

~~~text
0.20 × (-20) + 0.80 × 41.5 = 29.2
~~~

となります。最終的な状態値は、このような牌ごとの最良値を残り枚数で平均し、次巡の状態へ渡します。

### 18.4 ベタオリの1牌係数

同じ牌を2枚持ち、放銃確率0.15、放銃時価値-60、降りた価値0、減衰0.9なら、

~~~text
beta = 1 - 0.9^2 = 0.19
risk_coeff = 0.15 × (0 - (-60)) / (0.15 + 0.19 - 0.15×0.19)
           ≈ 28.89
~~~

となります。各牌種についてこの値を作り、係数の小さい順に必要枚数まで切ります。

## 19. 実装固有の制約・注意点

再実装・検証時に結果がずれやすい点を列挙します。

* get_shanten_num() は通常形と七対子の小さい方であり、国士無双はこの解析対象ではありません。
* rule_base_decision() は未リーチかつ総合4シャンテン以上、または通常5・七対子3以上で即時終了します。設定フィールド use_rule_base_at_mentu5_titoi3_shanten はこの箇所で参照されません。
* Tehai_Change の重複判定は id でなく牌枚数だけです。id は候補配列へ挿入する時に振り直されます。
* MAX_CANDIDATES_NUM や各 static vector の上限に達した時は、後続候補を黙って捨てます。
* DP の値はツモ回数の偶奇2スロットで上書きされます。全 tn の履歴を保持する実装では数値は一致しません。
* DP の牌残数は候補状態の使用枚数と、現在状態から候補へ移す差分を二重に調整します。単純な 4-visible-using_candidate だけでは一致しません。
* 現在時点の他家放銃確率は聴牌率を掛けた相手別値を合算し、確率だけ1以下へ丸めます。DP 内の将来値は tenpai_prob_other を掛けた相手別値を単純加算し、牌ごとの総和を1以下へ丸める処理はありません。
* ron_ratio_est="ako" の回帰ファイルがない、または必要重み数に満たない場合は assert で停止します。read_parameters() はファイル不在と個数不足を補完しません。
* my_logit() は確率0以下を-10、1以上を+10へ丸めます。通常の無限大ロジットではありません。
* Tactics::set_from_json() が読むのは base、一部の槓・DP降り設定、順位点、翻符分布、ベタオリ方式などです。jun_est、tenpai_after_est、agari_prob_est、ron_ratio_est などは tactics_json を各関数が直接参照します。
* デフォルトは fuuro_num_max=3、do_ankan_inclusive=true、do_kakan_inclusive=true、do_kan_ordinary=false、use_ori_exp_at_dp_fuuro=true です。
* 他家の kakan 直後は DP でロン・パスだけを比較します。槍槓の加算翻はロンの incident_han にだけ入ります。
* 総ツモ数70では鳴き候補を生成しません。ハイテイは AI の和了候補やロン判定では1翻として加算されますが、別の局進行側の点数計算と一致するとは限りません。
* Hai_Choice::out_moves() の最後の条件は if (action_type == AT_DAHAI || AT_REACH_DECLARE) で、列挙値を直接ORしているため常に真です。ただし既知の槓・和了・九種九牌はその前に return するため、通常の生成候補では打牌を追加する結果になります。不正な action_type を手作りで渡す API では注意が必要です。
* set_selector() は他家の打牌に対して chi_flag を座順の (actor+1)%4 == my_pid で判定します。鳴き候補の最終出力でも同じく下家条件を使います。
* yaku_dist.cpp の役距離関数は現行の Selector 主経路から呼ばれていません。役の候補優先度を再現する場合は Tehai_Pattern::cal_priority() の式を使います。

## 20. 根拠ファイル

この文書の主要な断定は次の実装を突き合わせて確認しました。

| 領域 | 根拠 |
|---|---|
| 最終分岐・行動候補・JSON | [selector.cpp](../akochan/ai_src/selector.cpp)、[selector.hpp](../akochan/ai_src/selector.hpp) |
| シャンテン・待ち・和了情報 | [tehai_ana.cpp](../akochan/ai_src/tehai_ana.cpp)、[tehai_ana.hpp](../akochan/ai_src/tehai_ana.hpp)、[agari.cpp](../akochan/ai_src/agari.cpp)、[agari.hpp](../akochan/ai_src/agari.hpp) |
| 候補列挙・グラフ・DP | [tehai_cal.cpp](../akochan/ai_src/tehai_cal.cpp)、[tehai_cal.hpp](../akochan/ai_src/tehai_cal.hpp)、[tehai_cal_supl.cpp](../akochan/ai_src/tehai_cal_supl.cpp)、[tehai_cal_supl.hpp](../akochan/ai_src/tehai_cal_supl.hpp)、[tehai_cal_work.cpp](../akochan/ai_src/tehai_cal_work.cpp)、[tehai_cal_work.hpp](../akochan/ai_src/tehai_cal_work.hpp)、[tehai_group.cpp](../akochan/ai_src/tehai_group.cpp)、[tehai_group.hpp](../akochan/ai_src/tehai_group.hpp) |
| 牌数・状態・行動型 | [bit_hai_num.cpp](../akochan/ai_src/bit_hai_num.cpp)、[bit_hai_num.hpp](../akochan/ai_src/bit_hai_num.hpp)、[tehai_state2.cpp](../akochan/ai_src/tehai_state2.cpp)、[tehai_state2.hpp](../akochan/ai_src/tehai_state2.hpp)、[tehai_action.cpp](../akochan/ai_src/tehai_action.cpp)、[tehai_action.hpp](../akochan/ai_src/tehai_action.hpp)、[tehai_change.cpp](../akochan/ai_src/tehai_change.cpp)、[tehai_change.hpp](../akochan/ai_src/tehai_change.hpp) |
| 手替わり・役候補 | [tehai_pat.cpp](../akochan/ai_src/tehai_pat.cpp)、[tehai_pat.hpp](../akochan/ai_src/tehai_pat.hpp)、[tehai_inout_pattern.cpp](../akochan/ai_src/tehai_inout_pattern.cpp)、[tehai_inout_pattern.hpp](../akochan/ai_src/tehai_inout_pattern.hpp)、[yaku_pattern.cpp](../akochan/ai_src/yaku_pattern.cpp)、[yaku_pattern.hpp](../akochan/ai_src/yaku_pattern.hpp)、[yaku_dist.cpp](../akochan/ai_src/yaku_dist.cpp)、[yaku_dist.hpp](../akochan/ai_src/yaku_dist.hpp) |
| リーチ後暗槓 | [ankan_after_reach.cpp](../akochan/ai_src/ankan_after_reach.cpp)、[ankan_after_reach.hpp](../akochan/ai_src/ankan_after_reach.hpp) |
| DP閾値・設定 | [tactics.cpp](../akochan/ai_src/tactics.cpp)、[tactics.hpp](../akochan/ai_src/tactics.hpp) |
| ベタオリ | [betaori.cpp](../akochan/ai_src/betaori.cpp)、[betaori.hpp](../akochan/ai_src/betaori.hpp) |
| 確率・牌山・赤牌補助 | [mjutil.cpp](../akochan/ai_src/mjutil.cpp)、[mjutil.hpp](../akochan/ai_src/mjutil.hpp) |

局面イベントの復元と MJAI の基本 JSON フィールドは [share/types.cpp](../akochan/share/types.cpp)、[share/make_move.cpp](../akochan/share/make_move.cpp) が補助的な根拠です。全体の位置付けは [akochan-architecture.md](./akochan-architecture.md) と併読してください。
