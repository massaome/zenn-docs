---
title: "Expo プロジェクトの開発環境セットアップガイド 2025"
emoji: "🎉"
type: "tech"
topics: ["expo", "reactnative", "typescript", "biome", "github"]
published: true
published_at: 2025-11-04 08:00
---

## 🎯 はじめに

最近、Expoの環境構築を行うことがあり、次回以降の備忘録のためにメモとして残しておこうと思います。
Linter・Formatterの設定とCIの設定までになります。

## 📖 本文

### 1. Expoプロジェクトの作成

`create-expo-app`を使ってプロジェクトを作成します。

```bash
pnpm create expo-app@latest ./ -t
```

作成時にテンプレートとして**Navigation (TypeScript)** を選択してください。

### 2. フォルダを整理する

プロジェクトのルートディレクトリに`src`フォルダを作成し、以下のフォルダを移動します。
好みですが、設定ファイルとソースコードを判別しやすくするために行なっています。

#### 移動対象のフォルダ

1. `app` フォルダ → `src/app`
2. `components` フォルダ → `src/components`
3. `constants` フォルダ → `src/constants`

フォルダを移動すると、ファイルパスが変わるため、相対パスを修正する必要があります。

- `src/app/_layout.tsx`

### 3. TypeScriptの設定

`tsconfig.json` にエイリアスを設定することで、相対パスの代わりにシンプルなカスタムエイリアスでモジュールをインポートできます。

```json:tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",　// プロジェクトのルートを基準ディレクトリに設定
    "paths": {
      "@/*": ["src/*"] // src フォルダ以下を「@/」で指定可能にする
    }
  }
}
```

https://docs.expo.dev/guides/typescript/#path-aliases-optional

`package.json` を更新

```json:package.json
"typecheck": "tsc --noEmit",
```

### 4. Biome

LinterとFormatterとしてBiomeを使っていきます。
Biomeの選定理由として、個人的にできるだけシンプルでデフォルトの設定で使いたい、ということで選んでいます。

https://biomejs.dev/ja/guides/getting-started/

https://zenn.dev/pepabo/articles/biome-essential-rules-for-better-code


```bash
pnpm add --save-dev --save-exact @biomejs/biome
```

#### 設定

```bash
pnpm exec biome init --jsonc
```

```json:biome.jsonc
{
	// Biomeの設定スキーマを指定
	"$schema": "https://biomejs.dev/schemas/2.1.1/schema.json",
	// バージョン管理システムとの連携設定
	"vcs": {
		"enabled": true,
		"clientKind": "git",
		"useIgnoreFile": true
	},
	// ファイル処理に関する設定
	"files": {
		"ignoreUnknown": false
	},
	// コードフォーマッターの設定
	"formatter": {
		"enabled": true,
		"indentStyle": "tab"
	},
	// リンター（コード品質チェック）の設定
	"linter": {
		"enabled": true,
		"rules": {
			"recommended": true,
			"nursery": {
				//CSSクラス名をアルファベット順に並び替えるようにする
				"useSortedClasses": "warn"
			},
			// 正確性に関するルール設定
			"correctness": {
				// 依存配列漏れを防ぐ
				"useExhaustiveDependencies": "warn"
			}
		}
	},
	// JavaScript固有の設定
	"javascript": {
		"formatter": {
			"quoteStyle": "double"
		}
	},
	// エディタ支援機能の設定
	"assist": {
		"enabled": true,
		"actions": {
			"source": {
				// インポートの自動整理を有効化
				"organizeImports": "on"
			}
		}
	}
}

```

#### npmスクリプトの設定

`package.json`の`scripts`セクションに以下のBiomeコマンドを追加します：

```json:package.json
	"scripts": {
		"check": "biome check .",
		"check:fix": "biome check --write .",
		"ci": "biome ci . --reporter=github"
	},
```

各スクリプトの説明:
- `check`: コードの品質チェック（エラー表示のみ）
- `check:fix`: チェック + 自動修正
- `ci`: CI環境用（GitHub向けレポート形式）

### 5. (お好みで)　Git Hooks（lefthook）の設定　

生成AIによって作成、編集したファイルはformatterがかかってないことが多く、大抵コミットしてからCIで落ちて気づくことが多々あります。
そこでGit Hooksを使用してコミット前に自動的にBiomeを実行する設定を行います。
デメリットとしては、コミットする時に少し時間がかかることです。
実験的に導入してみたことがあるのですが、コミット時の待ち時間が思ったより気になって、最新では一旦取りやめにしてます。

#### lefthookのインストール

```bash
pnpm add -D lefthook
```

#### lefthook.ymlファイルの作成

プロジェクトのルートディレクトリに`lefthook.yml`ファイルを作成します：

```yaml:lefthook.yml
pre-commit:
  commands:
    check:
      glob: "*.{js,ts,cjs,mjs,d.cts,d.mts,jsx,tsx,json,jsonc}"
      run: npx @biomejs/biome check --write --no-errors-on-unmatched --files-ignore-unknown=true --colors=off {staged_files}
      stage_fixed: true
```

https://biomejs.dev/ja/recipes/git-hooks/#lefthook

#### lefthookのセットアップ

```bash
npx lefthook install
```

#### 設定の説明

- コミット前に自動実行: `pre-commit`フックでコミット前にBiomeを実行
- 自動修正: `--write`フラグにより修正可能な問題を自動で修正
- 自動ステージング: `stage_fixed: true`で修正されたファイルを自動でステージング
- 対象ファイル: `{staged_files}`でステージングされたファイルのみを対象

これにより、コミット時に自動的にコードの品質と一貫性が保たれます。

#### ※追記　rcファイルの設定（GUI環境での問題解決）

GUI環境（VSCode、GitKraken、SourceTreeなど）でGit操作を行う際、環境変数やPATHの設定が正しく読み込まれないことがありました。
これにより、lefthookが`npm`や`pnpm`コマンドを見つけられずにエラーが発生する場合があるそうです。

##### rcファイルの設定方法

`lefthook-local.yml`ファイルを作成してrcファイルのパスを指定します：

```yaml:lefthook-local.yml
# 絶対パスで指定することが重要
rc: ~/.lefthookrc
```

`~/.lefthookrc`ファイルを作成して、必要な環境変数を設定します：

```bash:~/.lefthookrc
# nvm使用時の設定例
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# fnm使用時の設定例
export FNM_DIR="$HOME/.fnm"
[ -s "$FNM_DIR/fnm.sh" ] && \. "$FNM_DIR/fnm.sh"

# 直接PATHに追加する場合
export PATH="$PATH:$HOME/.nvm/versions/node/v20.11.0/bin"

# pnpmのパスを追加
export PATH="$PATH:$HOME/.local/share/pnpm"
```

##### 設定の反映

rcファイルを設定した後、Git Hooksを再インストールします：

```bash
npx lefthook install -f
```

これにより、GUI環境でもlefthookが正しくNode.jsツールチェーンにアクセスできるようになります。

参考：[Lefthook rc configuration](https://lefthook.dev/configuration/rc.html)

### 6. CI/CD設定（GitHub Actions）

プルリクエスト作成時に自動的にコード品質チェックを実行するCIワークフローを設定します。

#### .github/workflows/ci.ymlファイルの作成

```yaml:.github/workflows/ci.yml
name: Continuous Integration

on:
  pull_request:
    types: [opened, reopened, synchronize]
  workflow_dispatch:

jobs:
  ci-checks:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: ["20.x"]
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: "pnpm"

      - name: Install dependencies
        shell: bash
        run: pnpm install

      - name: Run Biome CI checks
        run: pnpm run ci

      - name: Typecheck
        run: pnpm run typecheck
```

### 7. VS Codeの設定

settings.jsonに以下の設定を行う

```json:.vscode/settings.json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "biomejs.biome",
  "editor.codeActionsOnSave": {
    "quickfix.biome": "explicit",
    "source.organizeImports.biome": "explicit"
  },
  "[javascript]": {
    "editor.defaultFormatter": "biomejs.biome"
  }
}
```

extensions.jsonに以下の設定を行う

```json:.vscode/extensions.json
{
	"recommendations": [
		"biomejs.biome",
		"yoavbls.pretty-ts-errors",
		"expo.vscode-expo-tools",
		"usernamehw.errorlens",
		"wix.vscode-import-cost"
	]
}
```

## 📋 まとめ

ここまで設定すれば、あとは要件次第でUIライブラリとかORMとか、好きなものをいれればアプリ開発ができそうです。
来年以降、見直しが入ればアップデートしていきます。

## 🔗 参考文献

- [Expo TypeScript Guide - Path Aliases](https://docs.expo.dev/guides/typescript/#path-aliases-optional)
- [Biome 公式ドキュメント](https://biomejs.dev/ja/guides/getting-started/)
- [Biome を使うときに最低限入れておきたい設定集](https://zenn.dev/pepabo/articles/biome-essential-rules-for-better-code)
- [Biome Git Hooks 設定](https://biomejs.dev/ja/recipes/git-hooks/#lefthook)




