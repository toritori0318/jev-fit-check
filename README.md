# jev-fit-check

「この判断に Jev を使うべきか」を 8 つの問いで決めるエージェントスキル。Claude Code と Codex で使える。

[Jev](https://typesafe.ai/) は TypeSafe AI の判断専用モデルで、文章を生成せず、渡した選択肢の中から確率付きで選ぶ。速くて安いが、使いどころを外すと自信ありげに間違える。

このスキルは、分類・ルーティング・スコアリングの処理を **コード / Jev 単独 / Jev → LLM・人のカスケード / LLM** のどれで作るべきかを判定する。Jev と出た場合は、質問の分解や閾値、データ保護まで含めた設計テンプレも出す。

非公式のコミュニティ製で、TypeSafe AI とは無関係。本文の数字はすべて公式ドキュメントと第三者の公開検証から取り、出典を `SKILL.md` の末尾に載せている。

## できること

- **使うべきかの判定**: 4 つの問いで、コード・LLM・Jev のどれにするかを決める
- **使う形の設計**: さらに 4 つの問いで、カスケードの形や注入対策、個人情報の扱いを決める
- **精度の改善**: 質問・選択肢・state・閾値の直し方。効果が数字で確かめられたものだけを載せている
- **既存の Jev 利用箇所のレビュー**: 11 の観点で抜けを指摘する
- **セキュリティが絡む判断の制約**: Jev を唯一の門にしない、エラー時は止める側に倒す、など

## インストール

入れ方は 3 つあり、どれでも中身は同じ。**どれか 1 つだけ**にする。両方入れると同じスキルが 2 つ出てきて、二重に反応する。

### Claude Code（プラグインとして。更新も受け取れる）

```bash
claude plugin marketplace add toritori0318/jev-fit-check
claude plugin install jev-fit-check@toritori0318
```

セッション内なら `/plugin marketplace add toritori0318/jev-fit-check` のあと `/plugin` から入れる。スキルは **`/jev-fit-check:jev-fit-check`** で呼ぶ（プラグインのスキルにはプラグイン名の接頭辞が付く）。更新は `claude plugin update jev-fit-check@toritori0318`。

### Claude Code（ファイルをコピーするだけ）

```bash
tmp=$(mktemp -d)
git clone --depth 1 https://github.com/toritori0318/jev-fit-check.git "$tmp"
mkdir -p ~/.claude/skills && cp -r "$tmp/skills/jev-fit-check" ~/.claude/skills/
```

Claude Code を起動し直すと **`/jev-fit-check`** で呼べる。プロジェクト単位で入れるなら `~/.claude/skills/` の代わりに `<プロジェクト>/.claude/skills/` にコピーする。更新は手動でコピーし直す。

### Codex

```bash
tmp=$(mktemp -d)
git clone --depth 1 https://github.com/toritori0318/jev-fit-check.git "$tmp"
mkdir -p ~/.agents/skills && cp -r "$tmp/skills/jev-fit-check" ~/.agents/skills/
```

ユーザー全体なら `~/.agents/skills/`、リポジトリ単位なら `<リポジトリ>/.agents/skills/`。Codex では **`$jev-fit-check`** で呼ぶか、`/skills` の一覧から選ぶ。frontmatter の `allowed-tools` は Claude Code 用の設定で、Codex では無視される。

## 使い方

判断の内容を一言添えて呼ぶ。件数とレイテンシの要件が分かっていれば一緒に書く。

```
/jev-fit-check 問い合わせを返品・配送・在庫・解約に振り分けたい。月 3,000 件、解約の兆候も拾いたい
```

```
/jev-fit-check PR をマージしてよいか判定したい
```

```
/jev-fit-check src/triage/rerank.py の Jev の使い方をレビューして
```

```
/jev-fit-check Jev の分類精度が 70% で頭打ち。質問はこう書いている: 「この問い合わせの深刻度は？」
```

返ってくるのは、判定（コード / Jev 単独 / Jev → LLM・人 / LLM）と、どの問いで決まったか。Jev と出た場合は設計テンプレが続く。

Claude Code では「これ Jev でできる？」「Jev の精度が出ない」のような言い方でも起動する。Codex では `$jev-fit-check` で明示的に呼ぶのが確実。

### 判定の例

「EC の問い合わせを返品・配送・在庫・定期解約・その他に振り分け、解約の兆候を拾う。月 3,000 件」なら、こう進む（全文は `SKILL.md` の「判定の例」）。

- Q0〜Q3: 間接表現が多く正規表現では拾えない。候補は列挙でき、根拠は本文にあり、件数も多い → すべて通過
- Q4: 振り分けの誤りは人が直せる。解約兆候は見逃しが痛いので、「兆候なし」だけ Jev で確定し、残りは人が見る → **Jev → 人のカスケード**
- Q7: 本文に名前・住所が入るので、送る前に伏せる

## 使わない場面

- Jev の API や SDK の書き方そのもの → [公式ドキュメント](https://docs.typesafe.ai/)
- Jev 以外の判断モデル、LLM 一般の相談

## 注意

Jev は 2026-09-15 公開の early access で、料金・上限（255 択、約 64K トークン）・モデル版（jev-1.13）は 2026-09 時点の公開情報。日本語での検証は小規模なものしかないため、材料が日本語なら自社データで閾値を決め直すことを前提にしている。

## リポジトリの構成

```
.claude-plugin/
  plugin.json        プラグインの定義
  marketplace.json   マーケットプレイスの定義（このリポジトリ自身を 1 プラグインとして載せている）
skills/
  jev-fit-check/
    SKILL.md         スキル本体。Claude Code・Codex 共通
```

## ライセンス

MIT
