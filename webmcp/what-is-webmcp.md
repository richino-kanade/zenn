---
title: "WebMCPとは何か"
emoji: "🌐"
type: "tech"
topics: ["webmcp", "mcp", "ai", "chrome", "agent"]
published: false
---

「MCP[^1]」という言葉は、AIをなんとなく使うようになってからいつのまにか定着していてかなり時間が経ったように感じる。Anthropicが提唱したMCPは、LLMにローカルファイルやデータベース、APIなどのツールをつなぎ込む仕組みとして急速に普及している[^2]。

そんな中、2026年3月に提案されたW3C[^2+1]に始まり、2026年8月に何やらCloudflareが騒ぎ立てている[^2+2]「**WebMCP**」というものが何なのかを追っかける。

## 🌐 WebMCPとは何か

WebMCPは、 **「Webページ自身がJavascriptの functionを公開し、AIエージェントがそれをtypedに呼び出せるようにする仕組み」** だ。

従来のBrowser Use[^2+3], Computer Use[^2+4]では、ページのスクショなどの描画を用い画像認識する手法、DOMツリーを探索してクリックするといえ手法などを手段として実行していた。しかし、WebMCPではページ側があらかじめ「この関数を実行すればカートに商品が入る」「この関数で検索ができる」といったツールを登録しておく。エージェントはそれをAPIのように直接実行することで目的の操作を完了できる。

MCPとの違いは、名前こそ似ているものの、**WebMCPはMCPとは全く別物**だ。いや、Model Context Protocolという英語の意味としては正しいのかもしれない。しかし、MCPとして想像するサーバーを立ち上げてJSON-RPCなどのプロトコルで通信するような仕組みとは全く異なる。

WebMCPは、ブラウザで開いているWebページ上のJavaScript関数（function）をそのまま叩くだけの仕組みだ。開発者ツールのデバッグコンソールを開いて、ページ上の関数を直接ポチポチ叩いていることと何ら変わりない。つまり、「クライアント側／ページ内」完結のツール公開インターフェースだ。

なお、仕様のステータスについては正確に把握しておく必要がある。WebMCPはW3Cの正式な勧告（Recommendation）ではなく、W3C Web Machine Learning Community Group（WebML CG）で議論されている提案段階のレポート（Draft Community Group Report / proposed web standard）だ[^3]。

現行のドラフト仕様（WebML CG draft）では、以下のように `document.modelContext` を通じてツールを登録・取得・実行するAPIが規定されている[^4]。

- `document.modelContext.registerTool(...)`
- `document.modelContext.getTools(...)`
- `document.modelContext.executeTool(...)`

Google Chromeの公式ドキュメント（2026年5月公開、8月更新）によると、Chrome 149からOrigin Trialが開始されており、ローカル環境でも `chrome://flags/#enable-webmcp-testing` を有効にすることでテストできる[^5]。

ちなみに、APIの命名には変遷がある。初期の提案や過去のドキュメントでは `navigator.modelContext` というエントリーポイントが使われていた。例えばCloudflare Browser Runのドキュメント（2026年4月更新）では `navigator.modelContextTesting.listTools` / `executeTool` と記載されている[^6]。ドラフトの改訂に伴ってエントリーポイントが `navigator` から `document` へ移行している過渡期であるため、「どちらが正式か」というよりは仕様策定の進展に伴う差分として理解するのが正確だ。

## 🧩 なぜWebMCPが必要なのか

なぜわざわざWebページ側に関数を登録させる必要があるのか。

最大の理由は、**「人間向けUIの逆エンジニアリングの脆さ」**を解消するためだ。

Webページの見た目やHTML構造は頻繁に変わる。エージェントがスクリーンショットの画像認識やDOMのヒューリスティクスに頼っていると、ボタンの色や配置が少し変わっただけで操作に失敗してしまう。

Chrome公式ドキュメントでも指摘されている通り、「エージェントが画面要素を観察して目的を推測する」よりも、「Webサイト自身がツールの目的（purpose）や入力スキーマを明示的に宣言する」ほうが圧倒的に堅牢だ[^7]。

さらに大きなメリットが、**「人間とエージェントのセッション共有」**だ。

Webサイト側がツールを提供してくれれば、人間がブラウザ上でログインしたセッションや入力中の画面状態を保ったまま、エージェントに横から作業を手伝ってもらうことができる。裏でヘッドレスブラウザを新しく立ち上げてログインし直す必要がなく、人間とAIが同じタブを見ながら協調できるのがWebMCPの大きな魅力だ。

## 🔍 最近の実装差とそれぞれの立ち位置

WebMCPやブラウザ連携エージェントの動きを見ると、各社のアプローチに興味深い違いが見えてきた。一次資料をもとに3つの軸で比較してみる。

### 1. Cloudflare: 供給側としての橋渡し
Cloudflareは、同社のBrowser Run機能においてWebMCPのサポートを提供している（2026年4月更新のドキュメントより）[^8]。
Chrome betaのlab sessionを活用し、`navigator.modelContextTesting` を用いて既存のWebサイトをエージェントが操作できるようにする手順が示されている。

Cloudflareの立ち位置は、どちらかといえば「ツール供給側」だ。サーバーサイドで動作するマネージドブラウザ上でWebMCPのエンドポイントを提供し、外部のエージェントから既存サイトの機能を呼び出せるようにするインフラ側の橋渡しを担っている。

### 2. Claude: 既存Chromeセッションに乗る使いやすさ
Anthropicは「Claude in Chrome」を提供している（公式ヘルプおよび公式ブログより）[^9][^10]。
公式ヘルプ「Get started with Claude in Chrome」によると、ユーザーが自分のChromeで、すでに開いている／ログインしたページにおいて画面の読み取り（read）、クリックや入力（click / navigate）、スクリーンショット撮影などを行う設計になっている。

普段使っているChromeをそのまま使うため、既存タブのセッション（ログイン状態やCookie）を引き継げるのが大きな強みだ。
例えば、Zero Trust（Cloudflare Access 等）で認証制限されている環境でも、「日常のChromeの既存セッションを使うためCookieが残っていればそのまま作業できる」という一般論としては言える。ただし、Zero Trust製品の公式一次情報として「デバイス信頼（Device Trust）まで必ず通る」と確認されたわけではない（この点は未確認だ）[^11]。

なお、Claude in ChromeがWebMCPの型付き関数呼び出し（typed call）を公式に解釈・実行するかどうかは現時点でドキュメント上未確認だ（GitHubの `anthropics/claude-code#30645` ではWebMCPサポートの要望が挙げられている）。したがって、Claude in Chromeの利便性は「すでにログインしている普段のChromeセッションをそのまま活用できる」という点に基づいている。

### 3. ChatGPT desktop: 独自内蔵ブラウザによるセッションの分断
OpenAIのChatGPTデスクトップアプリでは、「Site tools」としてWebMCPが導入されている（公式ヘルプ「Using site tools in the ChatGPT desktop app」より）[^12]。

ヘルプ内でも以下のように明記されている。
- 「Site tools use WebMCP, a proposed web standard（Site toolsは提案中のWeb標準であるWebMCPを使用している）」
- 「available only in the ChatGPT desktop app’s built-in browser, not in Chrome（Chromeではなく、ChatGPTデスクトップアプリの内蔵ブラウザでのみ利用可能）」
- 「The built-in browser has its own browser state. If you are already signed in to the same website in Chrome, you may need to sign in again in the built-in browser（内蔵ブラウザは独自のブラウザ状態を持つため、Chromeですでにログインしていても、内蔵ブラウザ側で再ログインが必要になる場合がある）」

このように、ChatGPTデスクトップアプリの内蔵ブラウザは独自のブラウザ状態を持つため、普段使っているChromeのログインセッションが引き継がれず、内蔵ブラウザ側で個別にログインし直さなければならない場面がある。普段のChromeを使えば既存タブのセッションをそのまま引き継げるのに対し、独自内蔵ブラウザを使うアプローチではセッションが分断されてしまうという使い勝手の差が生じている。

なお、Codexに関して一次資料（ChatGPT Learn）では「ChatGPTデスクトップアプリの内蔵ブラウザ上で、ChatGPT WorkやCodexがこれらのツールを利用できる」と記載されている[^13]。これはCodexがChatGPTデスクトップアプリの内蔵ブラウザ上でSite toolsを使えるという話であり、Codexアプリ独自の内蔵ブラウザである一次情報ではない。Codexアプリが独自の内蔵ブラウザによって日常のChromeとセッションが切れるかについては、一次情報としては未確認だ[^14]。

※なお、今回の比較は各社の公開ヘルプや公式ドキュメントなどの一次資料に基づいており、本ローカル環境でClaudeやChatGPTデスクトップアプリの全挙動を実地検証したわけではない。一次資料で確認できる事実関係を中心に整理している。

## 🚀 まとめとこれからの展望

WebMCPを取り巻く現状を整理すると、以下のようになる。

- **標準化の方向性**: Webページ自身がJavaScript関数としてエージェント向けツールを宣言・公開する（W3C WebML CGのドラフト標準）。
- **実装アプローチの分岐**: 「どのブラウザ環境で動かすか」によって使い勝手と設計が分かれている。
  - **Cloudflare**: 供給側として、サーバーサイドのマネージドブラウザから既存サイトの機能をエージェントに橋渡しする。
  - **Claude (Claude in Chrome)**: 普段のChromeセッションに乗ることで既存のログイン状態を引き継ぐ（現時点の公式ドキュメントではUI操作中心）。
  - **ChatGPT (Site tools)**: WebMCP標準を採用しているが、アプリ独自の内蔵ブラウザ限定のため日常のChromeセッションと分断される（※Codex単体での独自内蔵ブラウザ動作は一次未確認）。

Webページが人間だけでなくAIエージェントにとっても「操作しやすい窓口」を開放していく流れは確実に進んでいるし、開発をする上でAIに操作させる流れは来るだろう。今後はなんとかかんとかで締める。

[^1]: Model Context Protocol の略。サーバーとして JSON を返す。
[^2]: サーバー側MCPの動向およびセキュリティ脅威に関する研究については、Hou, X., Zhao, Y., Wang, S., & Wang, H. (2025). "Model Context Protocol (MCP): Landscape, Security Threats, and Future Research Directions." *arXiv preprint* arXiv:2503.23278 ( https://doi.org/10.48550/arXiv.2503.23278 ) の摘要などを参照（※本論文はサーバー側MCPに関する研究であり、WebMCPそのものの論文ではない）。
[^3]: W3C Web Machine Learning Community Group. "Web Model Context Protocol (WebMCP) - Draft Community Group Report". https://webmachinelearning.github.io/webmcp/ (GitHub: https://github.com/webmachinelearning/webmcp)
[^4]: 命令的APIの詳細については Google Chrome. "WebMCP Imperative API". https://developer.chrome.com/docs/ai/webmcp/imperative-api を参照。
[^5]: Google Chrome. "WebMCP: Web Model Context Protocol". https://developer.chrome.com/docs/ai/webmcp (Published: 2026-05-18, Updated: 2026-08-07)
[^6]: Cloudflare. "WebMCP in Browser Run". https://developers.cloudflare.com/browser-run/features/webmcp/ (Last updated: Apr 23, 2026)
[^7]: Google Chrome. "WebMCP: Web Model Context Protocol". https://developer.chrome.com/docs/ai/webmcp
[^8]: Cloudflare. "WebMCP in Browser Run". https://developers.cloudflare.com/browser-run/features/webmcp/
[^9]: Anthropic. "Get started with Claude in Chrome". https://support.claude.com/en/articles/12012173-get-started-with-claude-in-chrome
[^10]: Anthropic. "Claude in Chrome is now generally available". https://claude.com/blog/claude-in-chrome-generally-available
[^11]: Zero Trust（Cloudflare Access 等）環境において、Claude in Chrome 等がデバイス証明書やポスチャ確認などのデバイス信頼（Device Trust）要件を透過的に満たせるかどうかは各社公式一次情報でも未確認。
[^12]: OpenAI. "Using site tools in the ChatGPT desktop app". https://help.openai.com/en/articles/20001423-using-site-tools-in-the-chatgpt-desktop-app
[^13]: ChatGPT Learn. "WebMCP". https://learn.chatgpt.com/docs/webmcp （「In the built-in browser in the ChatGPT desktop app, ChatGPT Work and Codex can discover and use these tools when they are available.」とあり、CodexがChatGPT desktopの内蔵ブラウザ上でSite toolsを使えるという記述であり、Codexアプリ独自の内蔵ブラウザの存在を示す一次情報ではない）。
[^14]: Codexアプリが独自の内蔵ブラウザを備えており日常のChromeとセッションが切れるかについては、公式一次資料で未確認。
