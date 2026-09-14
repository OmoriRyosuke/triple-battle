# トリプルバトル（ポケモン風バトルシミュレーター）

## このプロジェクトについて

ブラウザで動く1ファイル完結のHTMLゲーム。ポケモンのトリプルバトル（3vs3）を再現した個人用・非営利のファンプロジェクト。
ユーザーはプログラミングをせず、日本語で仕様を伝えてClaudeが実装する進め方をしている。

## ファイル

- `triple_battle.html` … 本体。HTML/CSS/JSすべてが1ファイルに入っている
- `index.html` … GitHub Pages公開用（中身は triple_battle.html と同じ）
- `triple_battle_backup_v1.html` 〜 `v10.html` … 各時点のバックアップ。v10が直前の安定版
- 公開URL: https://omoriryosuke.github.io/triple-battle/
- リポジトリ: OmoriRyosuke/triple-battle（`index.html` を差し替えて更新する）

## 実装済みの主な仕様

- **位置ルール**: 左・中・右の3体が同時に出る。端のポケモンは対角の相手に届かない。中央は全員に届く。ひこう技・波動技は位置無視
- **ムーブ**: 端のポケモンが中央と入れ替わる。素早さ順で処理
- **自動リセットムーブ**: お互いの攻撃が届かない配置になったらターン終了時に両者中央へ（合体ヘイラッシャも考慮）
- **6体選出**: 手持ち6体から3体を場に出し、残りは交代で出す。先に6体倒したほうが勝ち
- **育成**: 特性・技（最大4つ）・持ち物・能力ポイント（合計66、各32まで）・性格補正（↑1.1倍/↓0.9倍、小数切り捨て）
- **ステータス**: HP = 種族値+75+ポイント、他 = 種族値+20+ポイント
- **メガシンカ**: ターンごとに選択。処理順は 交代 → メガシンカ → 技。メガ後の特性は即時発動
- **しれいとう**: シャリタツとヘイラッシャが揃うと合体（ヘイラッシャ全能力+2、シャリタツは狙われず行動不可、交代不可）
- 天候（晴れ/雨/砂/雪）、フィールド（サイコ/グラス/ミスト/エレキ）、壁、状態異常、混乱、もうどく
- カスタムポケモン・カスタム技の作成機能（localStorageに保存）
- データの書き出し／読み込み（チーム・カスタムデータをJSONで退避）
- 対戦は2人交互入力。相手の持ち物とHP実数値を隠すモードあり

## 規模（目安）

ポケモン78体 / 技190前後 / 持ち物44 / 実装済み特性100超

## 開発の進め方（重要）

1. `triple_battle.html` を直接編集する
2. **必ず自動テストを書いて検証してから完了とする**。テスト方法は下記
3. 大きめの変更のたびに `triple_battle_backup_vN.html` としてバックアップを作る
4. GitHubへの反映は `index.html` を同じ内容にしてコミット・プッシュ

## テストの書き方

HTMLから `<script>` の中身を取り出し、DOMをスタブしてNodeで実行する方式。

```js
// harness.js
const _elems={};
function mkEl(){return {innerHTML:"",appendChild(){},scrollTop:0,scrollHeight:0,id:"",value:"",checked:false,set textContent(v){}};}
global.document={getElementById:(id)=>{ if(!_elems[id])_elems[id]=mkEl(); return _elems[id]; },createElement:()=>mkEl()};
global.window=global;
const _store={};
global.localStorage={getItem:k=>_store[k]||null,setItem:(k,v)=>{_store[k]=String(v);},removeItem:k=>{delete _store[k];}};
global.alert=(m)=>console.log("[alert]",m);
global.confirm=()=>true;
const fs=require("fs");
const html=fs.readFileSync(process.argv[2],"utf-8");
const m=html.match(/<script>([\s\S]*?)<\/script>/);
eval(m[1]+"\n"+fs.readFileSync(process.argv[3],"utf-8"));
```

実行: `node harness.js triple_battle.html tests.js`

テスト側では以下のような流れでバトルを組み立てられる。

```js
const gi=n=>DEX.findIndex(d=>d.n===n);
const teamA={name:"A", mons:[gi("ガブリアス"),1,2,3,4,5].map(defaultCfg)};
const teamB={name:"B", mons:[0,2,8,9,21,24].map(defaultCfg)};
saveTeams([teamA,teamB]);
S={mode:"setup", step:"team1", teams:{}, orders:{}, orderTmp:[]};
pickTeam(1,0); pickTeam(2,1);
[0,1,2,3,4,5].forEach(i=>toggleOrder(i)); confirmOrder();
[0,1,2,3,4,5].forEach(i=>toggleOrder(i)); confirmOrder();
// B.sides[1].field[0] などで場のポケモンを取得し、execMove / execStatus / calcDamage を直接呼べる
```

**必ず確認すること**
- DEXの全ポケモンの技・持ち物がMOVES/ITEMSに存在するか（未定義があるとバトル中に落ちる）
- 通しプレイ（ターンを自動で回して最後まで進むか）
- 確率が絡む仕様は数千〜2万回のサンプリングで理論値と比較

## コードの主要関数

- `calcStats(dexIdx, ev, mega, nature)` … 実数値計算
- `effAbility(mon)` … かがくへんかガス等を考慮した実効特性。**特性判定は必ずこれを通す**
- `reachable(myPos, foePos)` … 射程判定（`Math.abs((2-myPos)-foePos)<=1`）
- `actPrio(x)` … 実行時の優先度（いたずらごころ・はやてのつばさ・グラススライダー等を毎回再評価）
- `execAction / execMove / execStatus` … 行動実行
- `damage(mon, amt, label, ctx)` … ダメージ処理。ログ順序のため `ctx.hold` に発動ログを退避できる
- `continueResolution()` … 行動順を毎回その時点の実効素早さで決めて1体ずつ処理（途中交代で中断・再開できる）

## 注意点

- チームは**ポケモン名でも保存**している（`loadTeams` で名前から番号を引き直す）。DEXに追加するときは**末尾に足す**こと
- ポケモン名・技名はゲーム内文字列として直接使っている。個人用・非営利の前提
- 使用率データの出典: ポケモン徹底攻略（ポケモンチャンピオンズ ダブル使用率）。★印のポケモンは未実装のため推定値
