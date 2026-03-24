---
title: "Notionのページをいい感じのPDFにしてくれるClaude Codeスキルを作った話"
emoji: "📄"
type: "tech"
topics: ["ClaudeCode", "Notion", "PDF", "skill"]
published: false
---

## はじめに

会社で契約しているNotionは、セキュリティ対策でエクスポート機能が無効化されています。
普段はそれで困ることもないんですが、ドキュメントをPDFで納品しないといけない場面が出てきて、さすがに手段がないと困る。

ブラウザの印刷機能でPDFにすることもできなくはないですが、余計なUIが入ったり、テーブルが崩れたり、mermaid図がそのまま出なかったりで、正直なところ微妙です。

で、どうせ作るならClaude Codeのスキルにしておけば今後も使い回せるので、スキルとして作ることにしました。

https://github.com/knktkc/notion-pdf

こんな感じのPDFが出ます。テーブル、コードブロック、callout、トグルなど主要なNotion要素に対応しています。

![サンプルPDF](/images/notion-pdf-sample.png)

## 使い方

Claudeに話しかけるだけです。

```
このNotionページをPDFにして: https://www.notion.so/your-page-id
```

URLを渡せば勝手にやってくれます。ページ名で検索してもらうこともできます。

インストールはこれだけ。

```bash
npx skills add knktkc/notion-pdf -g -y
```

Claude Desktop（claude.ai）でも使えます。[Releases](https://github.com/knktkc/notion-pdf/releases)からZIPをダウンロードして、Settings > Features > Skills からアップロードするだけです。

## Claude Codeのスキルとは

ざっくり言うと、Claudeが読むマニュアルのようなものです。
`SKILL.md`というファイルに「こういうときはこの手順でやってね」と書いておくと、Claudeがそれに従って作業してくれます。

https://docs.anthropic.com/ja/docs/claude-code/skills

スキルの中にはスクリプトや参照ファイルも含められるので、「Notion MCPでページを取得して→HTMLに変換して→PuppeteerでPDF化する」みたいな複雑なワークフローも、SKILL.mdに書いておけばClaudeが勝手にやってくれます。

## 全体の流れ

処理の流れはこんな感じです。

1. **Notion MCP** でページ内容を取得（Enhanced Markdown形式で返ってくる）
2. Notion固有のタグを**クリーンアップ**して標準的なMarkdownにする
3. Primer CSSのテンプレートに流し込んで**HTML生成**
4. puppeteer-coreで**PDF化**

この流れをSKILL.mdに書いておくと、Claudeが各ステップを順番に実行してくれます。
人間がやるのは「このページをPDFにして」と言うだけ。

## Notionの Enhanced Markdown をどうにかする

ここが一番面倒だったところです。

Notion MCPの`notion-fetch`でページを取得すると、Enhanced Markdownという形式で返ってきます。普通のMarkdownかと思いきや、Notion固有のタグがいろいろ混ざってます。

```
<page-title>プロジェクト計画書</page-title>
<page-icon>📋</page-icon>
<page-properties>
ステータス: 進行中
担当: <mention type="user" id="xxx">田中</mention>
</page-properties>

<data-source url="collection://xxx">
| 項目 | 内容 |
|------|------|
| 開始日 | 2025-04-01 |
</data-source>
```

`<page-title>`とか`<mention>`とか`<data-source>`とか、当然ブラウザでは描画できないので、これらを標準的なHTMLやテキストに変換してあげる必要があります。

で、ここが今回のスキル設計で一番面白いところなんですが、この変換処理をコードでは書いてないです。
代わりに、変換ルールをまとめたドキュメント（`references/notion-cleanup.md`）を用意して、Claude自身にクリーンアップさせています。

```
# クリーンアップルール（抜粋）

<page-title>...</page-title>  → 削除（h1として別途使用）
<mention type="page" id="...">ページ名</mention> → テキスト「ページ名」に置換
<data-source url="collection://...">...</data-source> → 削除
```

要するに、正規表現で頑張るスクリプトを書く代わりに、「このタグはこう変換してね」というルール集をClaudeに読ませているだけです。
Notionの仕様が変わってもルール集を更新するだけで対応できるので、メンテナンスも楽です。（たぶん）

## Primer CSSでスタイリングする

CSSフレームワークにはGitHubの[Primer CSS](https://primer.style/css/)を使いました。
Markdownのレンダリング用スタイルが最初から揃っていて、`class="markdown-body"`をつけるだけでGitHubのREADMEみたいな見た目になります。テーブルもコードブロックもblockquoteもいい感じ。

ただ、Primer CSSはテーマ変数（CSS Custom Properties）に依存しているところが多くて、環境によってはテーブルのボーダーが消えたりします。
`data-color-mode="light"`を`<html>`タグにつけないとCSS変数が未定義になるので、テンプレートの設計時にちょっとハマりました。

結局、CSS変数に頼らないフォールバックを直接書いてます。

```css
/* Primer CSSの変数が未定義の環境でも動くように */
.markdown-body table th,
.markdown-body table td {
  border: 1px solid #d0d7de;
  padding: 6px 13px;
}
.markdown-body blockquote {
  border-left: .25em solid #d0d7de;
  color: #656d76;
}
```

テーブルボーダー、blockquote、見出しの下線、コードブロック背景など、PDF出力に影響するスタイルは一通りフォールバックを入れてあります。

## PDF化のスクリプト

HTML→PDFの変換には`puppeteer-core`を使っています。
`puppeteer`ではなく`puppeteer-core`にしているのは、Chromiumバイナリを同梱しないためです。Chrome/Chromiumは環境から自動検出するようにしてあります。

```javascript
function findChrome() {
  // 1. 環境変数 CHROME_PATH
  // 2. which chromium-browser, google-chrome 等
  // 3. ~/.cache/puppeteer/ 内を検索
  // 4. macOS の /Applications/Google Chrome.app
}
```

あと、CDNにアクセスできない環境（Claude DesktopのVMとか）のために、HTML内のCDN URLをローカルファイルに自動置換する仕組みも入ってます。
`vendor/primer.css`にPrimer CSSを同梱してあるので、オフラインでもPDFが生成できます。

## Claude Desktop対応でハマったこと

Claude Desktopに対応させるときに、いくつかハマりポイントがありました。

### スキルディレクトリが読み取り専用

Claude DesktopのVMでは、スキルが`/mnt/skills/user/notion-pdf/`にマウントされるんですが、これが読み取り専用です。
`npm install`しようとしても書き込めなくて失敗します。
対策として、scriptsディレクトリを作業ディレクトリにコピーしてからインストールする手順をSKILL.mdに書きました。

### CDNにアクセスできない

VMからのネットワークアクセスはドメインが制限されていて、`cdn.jsdelivr.net`にはアクセスできませんでした。
CSSの読み込みでタイムアウトして止まります。
Primer CSSを`vendor/primer.css`として同梱し、スクリプト側でCDN URLをローカルパスに自動置換することで対応しました。

### 日本語フォントが微妙

VMの環境にはmacOSのフォント（ヒラギノとか）が入ってないので、日本語の表示が微妙でした。
`fc-list :lang=ja family`で調べたところ`Noto Sans CJK JP`が使えることが分かったので、font-familyに追加しました。

```css
font-family: -apple-system, BlinkMacSystemFont, "Segoe UI",
             "Noto Sans CJK JP", "Noto Sans JP", "Hiragino Sans",
             Helvetica, Arial, sans-serif;
```

## SKILL.mdの設計で意識したこと

スキルを作るにあたって、いくつか意識したことがあります。

### ワークフローを明確にステップ分けする

SKILL.mdにはステップ1〜6のワークフローを書いています。
「ページ特定→内容取得→クリーンアップ→HTML生成→PDF変換→完了報告」という流れを明確にしておくと、Claudeが迷わず処理してくれます。

### 詳細ルールはreferencesに分離する

クリーンアップルールやHTMLテンプレートの詳細はSKILL.mdには書かず、`references/`ディレクトリに分離しています。
SKILL.mdが肥大化すると全体の見通しが悪くなるので、参照すべき詳細は別ファイルにしておくのがよさそうです。

### descriptionフィールドはしっかり書く

スキルがいつ発動するかは`description`フィールドで決まります。
「NotionをPDFにして」だけでなく「このページを印刷したい」「ドキュメントをPDFで共有したい」「Notionのページを保存したい」など、ユーザーが使いそうな表現を網羅的に書いておくと、発動率が上がります。

### エラーハンドリングも書いておく

「Notion MCPが未接続のときはこう案内して」「Chromeが見つからないときはこうして」といったエラーケースもSKILL.mdに書いておくと、Claudeが自律的にリカバリしてくれます。
ユーザーに「npm installしてください」って提案するところまでやってくれるのは助かりますね。

## まとめ

ここまで書いてきた内容ですが、実は全部Claude Codeにお願いして実装してもらいました。
SKILL.mdの設計もスクリプトもClaude Desktop対応もCSSフォールバックも、やったのは方針を伝えてレビューしたくらいです。

普段はスキルを自分で作るというより、誰かが作ったものを探してきて使うことが多いんですが、今回は探しても見つからなかったので、試しにClaude Codeに作ってもらいました。
「NotionをPDFにするスキルを作って」から始まって、ここまでできてしまうのは便利な時代になったなぁと思います。

https://github.com/knktkc/notion-pdf
