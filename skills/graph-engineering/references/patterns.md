# 構造設計パターン集

lecture-14 の基礎概念と、game-proposal-studio の実設計で検証したパターン。
Phase 1（初版）と Phase 3（多重化・失敗・境界）で対象ワークフローに当てる。
**全部を使うことが目的ではない。** 該当しないパターンの不採用も人間に諮り、決定台帳へ残す。

## 基礎（lecture-14）

- グラフの4要素: **ノード**（仕事の単位。決定論的コードまたは完全な agent）、
  **エッジ**（次に何をするかの受け渡し。並列・条件・失敗/retry・rollback を表現）、
  **共有状態**（ノードが読み書きする共通ワークスペース）、
  **ルーティング規則**（条件に基づき次ノードを決める明示的な制御フロー）。
- workflow との違いはノードの型: workflow のノードは決定論的関数、グラフのノードは
  自前の loop を持つ agent。「グラフは workflow の一般化」であり、全決定論は特殊ケース。
- **loop は問題を内側に隠し、グラフは紙の上に出す。** グラフを描くことは、暗黙だった
  構造判断を前倒しで宣言させる行為である。
- **人間のレビュー帯域が常にボトルネック**（orchestration tax）。agent の起動は無料でも
  検証は無料でない。並列化しても人間の判断は並列化されない。ノードを増やす提案は
  常に「誰がいつ検証するか」とセットで書く。
- **文脈隔離は構造的独立性。** 検証ノードは fresh context で、生成側の推論を見ない。
  これは prompt の注意書きではなく設計プロパティ（物理隔離）として保証する。
- 形（トポロジー）は本質ではない。**replayability・observability・recoverability** が本質。

## エッジ設計

### 矢印の意味を固定する
矢印は「作業順の推奨」ではなく「**実際に起こりうる遷移**」。理想パスだけを描いた図は
実行時に必ず裏切られる。rollback・失敗・保留を含めて描く。

### 禁止エッジ（forbidden edge）を明示する
「〜へは戻らない」「〜は一度だけ」という文章規約は、グラフ上「エッジが存在しないこと」
として表現する。F 番号を振り、理由と、紛らわしい正規エッジとの区別を書く
（例:「proposal 改訂へは戻れるが blind 発散へは戻れない — 両立する」）。
違反検出が機械化でき、レビューで「このエッジは無いのか、書き忘れか」の曖昧さが消える。

### rollback は欠陥の発生層へ戻す
失敗を一律にスタートへ戻さない。欠陥がどの層の前提を崩したかで戻り先を分ける。
実例の三分類:

1. 末端成果物（検証仮説）だけが崩壊 → その成果物の設計ノードへ
2. 中間成果物（proposal の主張）だけが崩壊 → その成果物の改訂ループへ
3. 根本前提（brief・解空間・中核コンセプト）が崩壊 → **同一 run を巻き戻さず、
   旧 run を superseded で閉じて successor run を新規起票する**

3 が重要: 上流のブラインド工程や一度きり工程を「巻き戻し再実行」すると独立性・一回性の
規約が壊れる。新インスタンスの起票なら規約を保ったまま根本からやり直せる。
戻り先の選択自体が創造的判断なら、選ぶのは人間。

### failure edge は業務分岐と分けて宣言する
keep/kill/pivot のような業務ルーティングと、agent 失敗・外部ツール unavailable・
retry 枯渇・invalid output は別レイヤ。主図に重ねず overlay（別図・別表）で宣言する。
基本形:

- agent ノード: retryable → 同一 idempotency scope で retry。枯渇 → human interrupt
- 決定論ノード: 副作用を reconcile して idempotent retry。枯渇 → human interrupt
- checker/peer unavailable: **自動素通りさせない。** 再試行・代替・停止を人間が選ぶ

## 多重化

### fan-out はグラフのインスタンス化で表現する
「1テーマに成果物が複数」を受けるとき、検討順は:

| 案 | 内容 | 典型的評価 |
|---|---|---|
| A. 強制収束 | 常に1本へ絞らせる | 独立検証の機会を捨てる。多様性規律と矛盾しがち |
| B. スキーマ多重化 | 1インスタンス内にファイルを並べる | 下流の全契約が壊れ、checkpoint が交錯する |
| C. インスタンス化 | 単位ごとに同一グラフ（または部分グラフ）の新インスタンスを起動 | **下流契約を一切変えない。多重性はインスタンス数で表現** |

C を既定とする。fan-out 点は**人間ノード**にする — 並行本数 k の決定は下流コスト
（k 本ぶんの検証・レビュー・人間判断）の**コスト承認**であり、承認内容を記録する。

### fan-in は人間の創造判断 + 記録義務で閉じる
合流の併合規則を機械化しない（「全 pass なら merge」が成立するのは機械検査可能な
成果物だけ）。創造的成果物の合流は: 招集は全インスタンスの結果が出揃った後
（早期招集は残る実行中インスタンスの中止を明示記録）、融合・取捨・改善は人間の自由、
構造が課す義務は「各入力の処遇（採用/棄却/改善種）の記録」と「正史（下流への出力）は1つ」だけ。

### 分岐原理の対称性を確認する
分岐が複数の層に現れるなら（案の分岐、インスタンスの分岐、仮説の分岐…）、
全層で同じ契約に従わせる: **分岐は人間決定・記録必須・正史は混ぜない。**
fan-in も対で: **合流も人間決定・各入力の処遇を記録・正史は1つ。**

## 状態と正本

### 三グラフ分離
一枚の図に重ねがちな三つは別の設計対象:

| graph | 単位 | 扱うもの |
|---|---|---|
| Execution graph | 1 run + branch | ノード、routing、interrupt、retry、rollback、fan-out/fan-in |
| Artifact dependency graph | 成果物 | ID、revision、depends_on、stale、repair 対象 |
| 組織学習グラフ（portfolio learning） | run 間・時間軸 | 結果の一般化、次回起票が読む学習 |

「終端ノード → 次の run の開始」は execution edge ではなく、新 run が過去の学習を読む
data dependency。外部からの実績・観測も execution ノードに数えない（外部 event）。

### 正本の分離（三正本）
- **machine run ledger** = 実行位置の正。append-only event（node start/complete/fail/
  interrupt/route/external observation）を記録し、reducer が branch ごとの現在位置
  = **active frontier の集合**を導出する。並行分岐がある限り「現在位置は1ノード」は嘘になる。
- **artifact envelope**（revision / depends_on / stale） = 成果物鮮度の正。
- **人間判断ログ**（decision log） = 判断・理由・採否の正。実行 checkpoint に流用しない。

初版で「人間ログの末尾 = 現在位置」と置くのは動くが、並行 frontier を表現できず
superseded になった（実地の教訓）。並行化の予定が少しでもあるなら最初から ledger 設計にする。
外部 executor の状態ファイルは「観測 event」として ledger へ取り込み、例外的な別正本にしない。

### stale repair interrupt
required input が stale のノードは開始できない（block）。ただし stale が実行位置を黙って
巻き戻すことはない。router が owner ノードへ repair token を生成し、revalidate 後に
待機 branch へ fan-in して元の router を再評価する。修復不能・前提崩壊は rollback /
successor run へ escalate。

## ノード契約と文脈

### node contract
各ノードは表の6列（ID・責務・実行主体・入力・出力）に加えて宣言する:
role、input selector、private context packet、tool/read/write policy、output schema、
validator、completion criteria、timeout、retry budget、partial-result policy、
idempotency key、failure route。

### private context packet と物理隔離
ノード間で共有するのは revision 付き artifact と machine state だけ。会話履歴を暗黙入力に
しない。ブラインド生成・独立検証・auditor には許可入力だけを置いた run-scoped directory /
worktree を渡す — **「読まない」という prompt 指示だけに依存しない。**
canonical へ書ける主体は1つに限定し、variant/auditor は read-only または candidate 出力のみ。

### evidence snapshot
外部ツール（Web / MCP）由来の事実は、取得した会話を正本にせず、source URL・取得日時・
query・provenance 付きの snapshot artifact に固定してから下流へ渡す。並列の独立生成には
**同じ事実 packet** を渡す — 解釈の独立性と事実の取得差を混同しない。

### 人間ノードの規律
human interrupt は自動承認・自動却下しない。期限超過は overdue として可視化して現ノードに
留まる。判断要求は定型フォーマット（文脈・選択肢・各選択の帰結）で提示する。
全ルーター共通の catch-all: どの条件にも該当しない応答（保留等）は現ノードに留まり、
保留と理由を記録する。

## 境界と上位構造

### call-subgraph + typed boundary event
別ハーネス・別リポのワークフローは、ノードに畳んだ **call-subgraph** として接続する。
親は parent/child run ID・handoff revision・子 graph version を記録し、子の内部 frontier は
子が所有する。親が受理するのは**型付き boundary event の有限集合だけ**
（例: awaiting-human / completed / blocked / repair-requested）。
stale・状態変更は境界を自動で越えない — 境界を越える変更は handoff 契約の改訂のみ。
双方向になる場合は handoff に purpose と相関フィールド（どの request への応答か）を持たせ、
「新規起動」と「既存 child への結果返却」を型で分離し、request ID を idempotency key にする
（同一再配送は no-op、競合は human interrupt）。

### spec 正本化と生成投影
設計が安定したら、構造の正本を宣言的 spec（例: `graph.spec.yaml`）へ一本化する計画を書く。
guard/reducer/side effect の実装コードから graph 構造を再定義させない。Mermaid・ノード表・
エッジ表は spec から生成し、CI が spec・runtime・生成文書の edge 集合一致を検査する。
移行期の手書き文書には「実装後は生成物になる」旨を冒頭に明記する。

### skill / command は操作 API、node は実行契約
既存の skill / command とノードの一対一対応を要求しない。skill は `covers_nodes` と
参照 graph version を宣言して複数ノードを orchestrate してよいが、制御フロー自体を
本文へ複製しない。責務の分担例: 原則と役割 = 最上位規約、構造契約 = spec、
形式契約 = rules、手順と人間の判断点 = skills、実行主体への束縛 = commands。
