---
title: "WebMCPとは何か"
emoji: "🌐"
type: "tech"
topics: ["webmcp", "mcp", "ai", "chrome", "agent"]
published: false
---

「MCP[^1]」という言葉は、AIをなんとなく使うようになってからいつのまにか定着した。Anthropicが提唱したMCPは、LLMにローカルファイルやデータベース、APIなどのツールを繋ぎ込む仕組みとして急速に普及しています。

そんな中、ブラウザの標準化コミュニティや各社のプロダクトから「**WebMCP**」という新しい動きが出てきました。

「MCPと何が違うの？」「ブラウザ自動化とどう違うの？」と気になったので、仕様ドラフトや各社の一次ドキュメントを調べてみました。今回はその仕組みと背景、そして各社の実装アプローチの違いについて整理してお伝えします。

## 🌐 WebMCPとは何か

ひとことで言うと、WebMCPは**「Webページ自身がJavaScriptの関数（ツール）を公開し、AIエージェントがそれを直接型付き（typed）で呼び出せるようにする仕組み」**です。

従来のブラウザ操作エージェントといえば、ページのスクリーンショットを撮ってボタンの位置を画像認識し、DOMツリーを探索して「ここをクリックする」といった手順を踏んでいました。しかし、WebMCPではページ側があらかじめ「この関数を実行すればカートに商品が入る」「この関数で検索ができる」といったツールを登録しておきます。エージェントはそれをAPIのように直接実行できるわけです。

ここで注意したいのが、通常のMCPとの違いです。
MCPは通常、クライアントとツールの間にローカルサーバーやプロセスを立ち上げ、JSON-RPCなどのプロトコルを介してやりとりします。一方のWebMCPは、サーバーを立ててプロトコル通信を行うものではありません。

WebMCPは、ブラウザで開いているWebページ上のJavaScript関数をそのまま叩くだけの仕組みです。開発者ツールのデバッグコンソールを開いて、ページ上の関数を直接ポチポチ実行する感覚に非常に近いと言えます。まさに「クライアント側／ページ内」完結のツール公開インターフェースなのです。

なお、仕様のステータスについては正確に把握しておく必要があります。WebMCPはW3Cの正式な勧告（Recommendation）ではなく、W3C Web Machine Learning Community Group（WebML CG）で議論されている提案段階のレポート（Draft Community Group Report / proposed web standard）です。

現行のドラフト仕様（WebML CG draft）では、以下のように `document.modelContext` を通じてツールを登録・取得・実行するAPIが規定されています。

- `document.modelContext.registerTool(...)`
- `document.modelContext.getTools(...)`
- `document.modelContext.executeTool(...)`

Google Chromeの公式ドキュメント（2026年5月公開、8月更新）によると、Chrome 149からOrigin Trialが開始されており、ローカル環境でも `chrome://flags/#enable-webmcp-testing` を有効にすることでテスト可能です。

ちなみに、APIの命名には変遷があります。初期の提案や過去のドキュメントでは `navigator.modelContext` というエントリーポイントが使われていました。例えばCloudflare Browser Runのドキュメント（2026年4月更新）では `navigator.modelContextTesting.listTools` / `executeTool` と記載されています。ドラフトの改訂に伴ってエントリーポイントが `navigator` から `document` へ移行している過渡期であるため、「どちらが正式か」というよりは仕様策定の進展に伴う差分として理解するのが正確です。

## 🧩 なぜWebMCPが必要なのか

なぜわざわざWebページ側に関数を登録させる必要があるのでしょうか。

最大の理由は、**「人間向けUIの逆エンジニアリングの脆さ」**を解消するためです。

Webページの見た目やHTML構造は頻繁に変わります。エージェントがスクリーンショットの画像認識やDOMのヒューリスティクスに頼っていると、ボタンの色や配置が少し変わっただけで操作に失敗してしまいます。

Chrome公式ドキュメントでも指摘されている通り、「エージェントが画面要素を観察して目的を推測する」よりも、「Webサイト自身がツールの目的（purpose）や入力スキーマを明示的に宣言する」ほうが圧倒的に堅牢です。

さらに大きなメリットが、**「人間とエージェントのセッション共有」**です。

Webサイト側がツールを提供してくれれば、人間がブラウザ上でログインしたセッションや入力中の画面状態を保ったまま、エージェントに横から作業を手伝ってもらうことができます。裏でヘッドレスブラウザを新しく立ち上げてログインし直す必要がなく、人間とAIが同じタブを見ながら協調できるのがWebMCPの大きな魅力です。

## 🔍 最近の実装差とそれぞれの立ち位置

WebMCPやブラウザ連携エージェントの動きを見ると、各社のアプローチに興味深い違いが見えてきました。一次資料をもとに3つの軸で比較してみます。

### 1. Cloudflare: 供給側としての橋渡し
Cloudflareは、同社のBrowser Run機能においてWebMCPのサポートを提供しています（2026年4月更新のドキュメントより）。
Chrome betaのlab sessionを活用し、`navigator.modelContextTesting` を用いて既存のWebサイトをエージェントが操作できるようにする手順が示されています。

Cloudflareの立ち位置は、どちらかといえば「ツール供給側」です。サーバーサイドで動作するマネージドブラウザ上でWebMCPのエンドポイントを提供し、外部のエージェントから既存サイトの機能を呼び出せるようにするインフラ側の橋渡しを担っています。

### 2. Claude: 既存Chromeセッションに乗る使いやすさ
Anthropicは「Claude in Chrome」を提供しています（公式ヘルプおよび公式ブログより）。
ユーザーが普段使っているGoogle Chrome上で拡張機能として動作するため、Googleアカウントや各Webサービスにすでにログイン済みの状態でそのまま作業を依頼できるのが大きな強みです。

公式ヘルプによると、現在のClaude in Chromeの主要機能は画面の読み取り（read）、クリックや入力（click / navigate）、スクリーンショット撮影など、人間の操作を模倣するブラウジングが中心となっています。

なお、Claude in ChromeがWebMCPの型付き関数呼び出し（typed call）を公式に解釈・実行するかどうかは現時点でドキュメント上未確認です（GitHubの `anthropics/claude-code#30645` ではWebMCPサポートの要望が挙げられています）。したがって、Claude in Chromeの利便性は「すでにログインしている既存のChromeセッションをそのまま活用できる」という点に基づいています。

### 3. ChatGPT / Codex desktop: 独自内蔵ブラウザによるセッションの分断
OpenAIのChatGPTデスクトップアプリでは、「Site tools」としてWebMCPが導入されています（公式ヘルプ記事「Using site tools in the ChatGPT desktop app」より）。

ヘルプ内でも以下のように明記されています。
- 「Site tools use WebMCP, a proposed web standard（Site toolsは提案中のWeb標準であるWebMCPを使用している）」
- 「available only in the ChatGPT desktop app’s built-in browser, not in Chrome（Chromeではなく、ChatGPTデスクトップアプリの内蔵ブラウザでのみ利用可能）」
- 「The built-in browser has its own browser state. If you are already signed in to the same website in Chrome, you may need to sign in again in the built-in browser（内蔵ブラウザは独自のブラウザ状態を持つため、Chromeですでにログインしていても、内蔵ブラウザ側で再ログインが必要になる場合がある）」

この仕様のため、ChatGPTデスクトップアプリのWebMCPは「普段使っているChromeのログインセッションが引き継がれず、内蔵ブラウザ側で個別にログインし直さなければならない」という点で使いにくさや手間に直面しやすい構造になっています。

※なお、今回の比較は各社の公開ヘルプや公式ドキュメントなどの一次資料に基づいており、本ローカル環境でClaudeやChatGPTデスクトップアプリの全挙動を実地検証したわけではありません。一次資料で確認できる事実関係を中心に整理しています。

## 🚀 まとめとこれからの展望

WebMCPを取り巻く現状を整理すると、以下のようになります。

- **標準化の方向性**: Webページ自身がJavaScript関数としてエージェント向けツールを宣言・公開する（W3C WebML CGのドラフト標準）。
- **実装アプローチの分岐**: 「どのブラウザ環境で動かすか」によって使い勝手と設計が分かれている。
  - **Cloudflare**: 供給側として、サーバーサイドのマネージドブラウザから既存サイトの機能をエージェントに橋渡しする。
  - **Claude (Claude in Chrome)**: 既存のChromeセッションに乗ることでログインの手間を省く（現時点の公式ドキュメントではUI操作中心）。
  - **ChatGPT / Codex (Site tools)**: WebMCP標準を採用しているが、アプリ独自の内蔵ブラウザ限定のため日常のChromeセッションと分断される。

Webページが人間だけでなくAIエージェントにとっても「操作しやすい窓口」を開放していく流れは確実に進んでいます。今後、各ブラウザやエージェント製品がどのエントリーポイントに収束していくのか、引き続き注目していきたいところです。

## 📚 参考文献

本稿の執筆にあたり参照・確認した一次仕様および関連文献です（書籍に関しては直接該当する刊行物が確認されていないため、公式仕様・ドキュメントおよび論文を参照しています）。

- W3C Web Machine Learning Community Group. "Web Model Context Protocol (WebMCP) - Draft Community Group Report". https://webmachinelearning.github.io/webmcp/ (GitHub: https://github.com/webmachinelearning/webmcp)
- Google Chrome. "WebMCP: Web Model Context Protocol". https://developer.chrome.com/docs/ai/webmcp (Published: 2026-05-18, Updated: 2026-08-07)
- Google Chrome. "WebMCP Imperative API". https://developer.chrome.com/docs/ai/webmcp/imperative-api
- Cloudflare. "WebMCP in Browser Run". https://developers.cloudflare.com/browser-run/features/webmcp/ (Last updated: Apr 23, 2026)
- OpenAI. "Using site tools in the ChatGPT desktop app". https://help.openai.com/en/articles/20001423-using-site-tools-in-the-chatgpt-desktop-app
- Anthropic. "Get started with Claude in Chrome". https://support.claude.com/en/articles/12012173-get-started-with-claude-in-chrome
- Anthropic. "Claude in Chrome is now generally available". https://claude.com/blog/claude-in-chrome-generally-available
- Hou, X., Zhao, Y., Wang, S., & Wang, H. (2025). "Model Context Protocol (MCP): Landscape, Security Threats, and Future Research Directions." *arXiv preprint* arXiv:2503.23278. https://doi.org/10.48550/arXiv.2503.23278
  （※本論文はサーバー側MCPの動向およびセキュリティ脅威に関する研究であり、WebMCPそのものの論文ではありません。サーバー側MCPとの対比・背景整理のために摘要を確認しています）

  ^1 Model Context Protocol の略。サーバーとしjsonを返却する
