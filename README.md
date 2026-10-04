# jev-fit-check

「この判断に Jev を使うべきか」を 8 つの問いで決めるエージェントスキル。Claude Code と Codex で使える。

[Jev](https://typesafe.ai/) は TypeSafe AI の判断専用モデルで、文章を生成せず、渡した選択肢の中から確率付きで選ぶ。速くて安いが、使いどころを外すと「自信ありげに間違える」。このスキルは、ある分類・ルーティング・スコアリング処理について、**コード / Jev 単独 / Jev → LLM・人のカスケード / LLM** のどれが適切かを判定し、Jev と出た場合は設計テンプレ（質問の分解・選択肢・state・閾値・正解ラベル・フォールバック・データ保護・監査）まで出す。

非公式のコミュニティ製で、TypeSafe AI とは無関係。本文の数字はすべて公開されている第三者の検証から取り、出典を `SKILL.md` の末尾に載せている。

## できること

- **使うべきかの判定**: Q0〜Q3 のゲート（ルールで書けるか／候補を列挙できるか／根拠が入力の中にあるか／量か速さが価値になるか）で、外れたらそこで結論
- **使う形の設計**: Q4〜Q7（確信度で損害を抑えられるか／正解ラベルを作れるか／敵対的な入力が混ざるか／個人情報が入るか）で、カスケードの形・shadow mode・注入対策・データ保護を決める
- **精度を上げるコツ**: 質問・選択肢・state・閾値の 4 層。効果が数字で出たものだけ（例: 大きな問いを 5 問に分解して 62.6% → 95.0%、閾値を 0.5 から 0.80 に直して 76% → 87%）
- **既存の Jev 利用箇所のレビュー**: 11 の観点で抜けを指摘
- **セキュリティが絡む判断の制約**: Jev を唯一の門にしない、止める側に倒す、監査ログ、モデル版の固定

## インストール

スキル本体は `skills/jev-fit-check/SKILL.md` の 1 ファイル。入れ方は 3 つあり、どれでも中身は同じ。

### Claude Code（プラグインとして。更新も受け取れる）

```bash
claude plugin marketplace add toritori0318/jev-fit-check
claude plugin install jev-fit-check@toritori0318
```

セッション内なら `/plugin marketplace add toritori0318/jev-fit-check` のあと `/plugin` から入れる。スキルは **`/jev-fit-check:jev-fit-check`** で呼ぶ（プラグインのスキルにはプラグイン名の接頭辞が付く）。更新は `claude plugin update jev-fit-check@toritori0318`。

### Claude Code（ファイルをコピーするだけ）

```bash
git clone --depth 1 https://github.com/toritori0318/jev-fit-check.git /tmp/jev-fit-check
cp -r /tmp/jev-fit-check/skills/jev-fit-check ~/.claude/skills/
```

Claude Code を起動し直すと **`/jev-fit-check`** で呼べる。プロジェクト単位で入れるなら `~/.claude/skills/` の代わりに `<プロジェクト>/.claude/skills/` にコピーする。更新は手動でコピーし直す。

### Codex

```bash
git clone --depth 1 https://github.com/toritori0318/jev-fit-check.git /tmp/jev-fit-check
cp -r /tmp/jev-fit-check/skills/jev-fit-check ~/.agents/skills/
```

ユーザー全体なら `~/.agents/skills/`、リポジトリ単位なら `<リポジトリ>/.agents/skills/`。Codex では **`$jev-fit-check`** で呼ぶか、`/skills` の一覧から選ぶ。Codex が読むのは frontmatter の `name` と `description` で、`allowed-tools` は Claude Code 向けの指定なので無視される（Codex はツールを制限せずに動く）。

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

返ってくるのは、判定（コード / Jev 単独 / Jev → LLM・人 / LLM）と、どの問いで決まったか。Jev と出た場合は設計テンプレが続く。「これ Jev でできる？」「Jev の精度が出ない」のような言い方でも起動する（Claude Code の場合。Codex は `$jev-fit-check` で明示的に呼ぶのが確実）。

### 判定の例

「問い合わせを 4 カテゴリに振り分け、解約の兆候を拾う。月 3,000 件」なら、こう進む。

- Q0: 「お金を戻していただけないでしょうか」のような間接表現は正規表現で拾えない → 通過
- Q1: 4 カテゴリ＋「不明」で列挙できる。解約兆候は はい／いいえ → 通過
- Q2: 根拠は問い合わせ本文にある → 通過
- Q3: 月 3,000 件を全件見たい → 通過
- Q4: 振り分けの誤りは人が直せる。解約兆候は見逃しが痛いので、否定側だけ Jev で確定し、残りは人が見る → **Jev → 人のカスケード**
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
