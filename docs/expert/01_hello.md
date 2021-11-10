# Hello Language Server Protocol

## はじめに

本ハンズオンでは[**L**anguage **S**erver **P**rotocol](https://microsoft.github.io/language-server-protocol/) (以下，**LSP**)を用いたエディタの拡張機能開発を行います．
LSPとは，コード補完や，変数参照，スタイル修正といった機能実装を[あらゆるエディタ/IDE](https://microsoft.github.io/language-server-protocol/implementors/tools/) へ提供するプロトコルです．

本ハンズオンでは[Visual Studio Code](https://code.visualstudio.com/)を用いて拡張機能開発を行いますが，作成した機能は[Vim](https://www.vim.org/)や[Emacs](https://www.gnu.org/software/emacs/)でも使えます．

## 対象読者

* エディタ開発に興味がある人
* エディタの構造を知りたい人
* 色んなエディタをいじってみたい人

もしこの記事の内容が難しすぎる場合，以下のリンクから基礎的な学習を行えます．

* [TypeScript](http://js.studio-kingdom.com/typescript/)
* [Npm パッケージ](https://qiita.com/dondoko-susumu/items/cf252bd6494412ed7847)

この記事ではTypeScriptを利用していますが，あまり高度な機能は利用しないため，JavaScript，もしくは他の言語にふれたことがあれば大丈夫です．

## 今回行うこと

* Hello Worldを表示 (本ドキュメント)
* Linterコース
  * [特定の文章に対して警告を出す]()
  * [警告に対して自動修正を行う]()
* 補完機能コース
  * ["VS Code", "Visual Studio Code"を補完入力する]()

### 行わないこと

* 言語依存の機能(補完, 参照)開発
* リモート環境・マルチルートワークスペースを考慮した機能開発

## 開発環境

* [Visual Studio Code](https://code.visualstudio.com/)
* [Node JS](https://nodejs.org/) x64, version >= 10.x, <= 12.x

## Hello World

コンピュータープログラミングにおける古来の伝統に従い，最初に作るアプリケーションは「Hello World」を表示するプログラムにしましょう．

[簡素なテンプレート](https://github.com/Ikuyadeu/vscode-language-server-template)を用意しましたのでそれをクローンしてVS Codeで開きましょう．

```sh
git clone https://github.com/Ikuyadeu/vscode-language-server-template.git
code vscode-language-server-template
```

### ビルド

1: ターミナルを起動
[表示] -> [統合ターミナル]
または，`Control` + \` (Macなら `^` + \`)

2: 以下のコマンドを実行し，必要なパッケージのインストールを行います．

```sh
npm install
```

3: `F5` キーを入力することで拡張機能をインストールしたVS Codeを立ち上げます．
4: 開いたエディタ上で`.txt`ファイル，もしくは`.md`ファイルを開いてみましょう．
画像では`test.txt`の１行目に波線を表示させ，その上にマウスを置くと`Hello World`と表示させています．

<img width="518" alt="Screen Shot 2019-12-22 at 0.27.56.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/221488/ece4c4c2-158b-c36a-9376-a025f80c1b73.png">


### ファイル構成

以下がVS CodeでLSPを実装するときの一般的なファイルの構成です．
本記事ではサーバー側のメインである`server/src/server.ts`を編集していきます．

```
.
├── client               // Language Serverのクライアントサイド
│   ├── src
│   │   └── extension.ts // クライアント側拡張機能のソースコード
│   └── package.json     // クライアント側として利用するパッケージ情報
│
├── server               // Language Serverのサーバーサイド
│   │── src
│   │   └── server.ts    // Language Serverのメインソースコード
│   │── package.json     // クライアント側として利用するパッケージ情報
│   └── README.md        // サーバーとしてのREADME.md (説明書)
│
├── package.json         // VS Codeプラグインとしてのパッケージ情報 
└── README.md            // VS CodeプラグインとしてのREADME.md(説明書)
```

### ソースコード解説

<details><summary>サーバー側のソースコード</summary><div>

```ts:server/src/server.ts
'use strict';

import {
	createConnection,
	Diagnostic,
	DiagnosticSeverity,
	ProposedFeatures,
	Range,
	TextDocuments,
	TextDocumentSyncKind,
} from 'vscode-languageserver/node';
import { TextDocument } from 'vscode-languageserver-textdocument';

// サーバー接続オブジェクトを作成する。この接続にはNodeのIPC(プロセス間通信)を利用する
// LSPの全機能を提供する
const connection = createConnection(ProposedFeatures.all);
connection.console.info(`Sample server running in node ${process.version}`);
// 初期化ハンドルでインスタンス化する
let documents!: TextDocuments<TextDocument>;

// 接続の初期化
connection.onInitialize((_params, _cancel, progress) => {
	// サーバーの起動を進捗表示する
	progress.begin('Initializing Sample Server');
	// テキストドキュメントを監視する
	documents = new TextDocuments(TextDocument);
	setupDocumentsListeners();
	// 起動進捗表示の終了
	progress.done();

	return {
		// サーバー仕様
		capabilities: {
			// ドキュメントの同期
			textDocumentSync: {
				openClose: true,
				change: TextDocumentSyncKind.Incremental,
				willSaveWaitUntil: false,
				save: {
					includeText: false,
				}
			}
		},
	};
});

/**
 * テキストドキュメントを検証する
 * @param doc 検証対象ドキュメント
 */
function validate(doc: TextDocument) {
	// 警告などの状態を管理するリスト
	const diagnostics: Diagnostic[] = [];
	// 0行目(エディタ上の行番号は1から)の端から端までに警告
	const range: Range = {start: {line: 0, character: 0},
		end: {line: 0, character: Number.MAX_VALUE}};
	// 警告を追加する
	const diagnostic: Diagnostic = {
		// 警告範囲
		range: range,
		// 警告メッセージ
		message: 'Hello world',
		// 警告の重要度、Error, Warning, Information, Hintのいずれかを選ぶ
		severity: DiagnosticSeverity.Warning,
		// 警告コード、警告コードを識別するために使用する
		code: '',
		// 警告を発行したソース、例: eslint, typescript
		source: 'sample',
	};
	diagnostics.push(diagnostic);
	//接続に警告を通知する
	connection.sendDiagnostics({ uri: doc.uri, diagnostics });
}


/**
 * ドキュメントの動作を監視する
 */
function setupDocumentsListeners() {
	// ドキュメントを作成、変更、閉じる作業を監視するマネージャー
	documents.listen(connection);

	// 開いた時
	documents.onDidOpen((event) => {
		validate(event.document);
	});

	// 変更した時
	documents.onDidChangeContent((change) => {
		validate(change.document);
	});

	// 保存した時
	documents.onDidSave((change) => {
		validate(change.document);
	});

	// 閉じた時
	documents.onDidClose((close) => {
		// ドキュメントのURI(ファイルパス)を取得する
		const uri = close.document.uri;
		// 警告を削除する
		connection.sendDiagnostics({ uri: uri, diagnostics: []});
	});
}

// Listen on the connection
connection.listen();
```
</div></details>
<details><summary>クライアント側のソースコード</summary><div>

```ts:client/src/extension.ts
'use strict';

import { ExtensionContext, window as Window, Uri } from 'vscode';
import {
	LanguageClient,
	LanguageClientOptions,
	RevealOutputChannelOn,
	ServerOptions,
	TransportKind } from 'vscode-languageclient/node';

// 拡張機能が有効になったときに呼ばれる
export function activate(context: ExtensionContext): void {
	// サーバーのパスを取得
	const serverModule =  Uri.joinPath(context.extensionUri, 'server', 'out', 'server.js').fsPath;
	// デバッグ時の設定
	const debugOptions = { execArgv: ['--nolazy', '--inspect=6011'], cwd: process.cwd() };

	// サーバーの設定
	const serverOptions: ServerOptions = {
		run: {
			module: serverModule,
			transport: TransportKind.ipc,
			options: { cwd: process.cwd() }
		},
		debug: {
			module: serverModule,
			transport: TransportKind.ipc,
			options: debugOptions,
		},
	};
	// LSPとの通信に使うリクエストを定義
	const clientOptions: LanguageClientOptions = {
		// 対象とするファイルの種類や拡張子
		documentSelector: [
			{ scheme: 'file' },
			{ scheme: 'untitled' }
		],
		// 警告パネルでの表示名
		diagnosticCollectionName: 'sample',
		revealOutputChannelOn: RevealOutputChannelOn.Never,
		initializationOptions: {},
		progressOnInitialization: true,
	};

	let client: LanguageClient;
	try {
		// LSPを起動
		client = new LanguageClient('Sample LSP Server', serverOptions, clientOptions);
	} catch (err) {
		void Window.showErrorMessage('拡張機能の起動に失敗しました。詳細はアウトプットパネルを参照ください');
		return;
	}

	// 拡張機能のコマンドを登録
	context.subscriptions.push(
		client.start(),
	);
}
```
</div></details>