---
title: "Agent DeviceでAIエージェントにiOSアプリを操作させてみた"
emoji: "📱"
type: "tech"
topics: ["reactnative", "expo", "agentdevice", "ai", "mcp"]
published: true
published_at: 2026-06-20 08:00
---

## はじめに

Callstack Incubator が公開した [Agent Device](https://agent-device.dev/) は、AI エージェント向けの CLI ツールです。iOS / Android / TV / デスクトップアプリを操作できます。アクセシビリティツリーをもとに、UI 要素へ `@e1` のような参照で触れます。これにより Claude Code や Codex のようなコーディングエージェントから「実機を見て・触る」ことが可能になります。

実際の使い心地として面白いのは、**CLI コマンドを自分で叩く必要がほぼない**点です。Skill 対応クライアント向けの Skill や、MCP 対応クライアント向けの MCP サーバが公式に提供されているため、登録さえ済ませれば自然言語で依頼するだけで動きます。エージェントが内部で `open` → `snapshot` → `screenshot` のシーケンスを組み立てて実行してくれます。

本記事では macOS + iOS シミュレーターで Skill 経由のワークフローを動かし、内部でどのような操作をしているかを CLI の基本ループも交えて整理します。

:::message
動作確認した環境は次のとおりです。Agent Device はコマンド名・引数・対応プラットフォームがバージョンで変わるため、手元では `agent-device --version` と `agent-device help workflow` の出力を一次情報として確認してください。

- agent-device 0.14.9
- macOS + iOS シミュレーター (iPhone 17 Pro)
- 対象アプリは Expo テンプレートを Expo Go で起動 (Metro: `127.0.0.1:8081`)
:::

## Agent Device とは何か

公式の [README](https://github.com/callstackincubator/agent-device) は「Mobile app verification for AI agents」と説明しています。要点を整理するとこうなります。

- **アクセシビリティツリーで UI を表現する**。スクリーンショットではなく、要素の role / label / text を構造化したスナップショットを返すので、LLM のトークンを節約できる
- **`@e1` のような短い ref で操作する**。スナップショット内の各要素に短いハンドルが割り振られ、`press @e2` のような形でそのまま操作できる。ref はスナップショットごとに振り直されるので、画面が変わったら取り直す
- **クロスプラットフォームに対応する**。iOS / Android / tvOS / Android TV / macOS / Linux を同じコマンド体系で扱える
- **セッション単位で状態を保持する**。`open` で開いたアプリは `close` するまで状態を保ち、複数コマンドを跨いで操作できる
- **記録と再生ができる**。操作スクリプトとして保存し、決定論的に再実行できる
- **エージェント連携を前提とする**。Skill / Rules / MCP サーバ / 直接 CLI と複数の連携方法を公式が用意しており、クライアントが対応する形態を選べる

Vercel Labs の [agent-browser](https://github.com/vercel-labs/agent-browser) から着想を得ており、公式 README も「the same idea for mobile, TV, and desktop apps」と述べています。コンセプトは「コーディングエージェントがモバイル UI を直接触れるようにする」ことです。

## Appium / Maestro とどう違うか

Agent Device はモバイルアプリを操作するツールなので、Appium や Maestro のような E2E テストツールと近い領域に見えます。ただ、主眼は少し違います。

| 観点 | Appium | Maestro | Agent Device |
| --- | --- | --- | --- |
| 主用途 | E2E テスト自動化 | E2E テスト自動化 | AI エージェントによる UI 操作と replay |
| 操作の書き方 | WebDriver API | YAML ベースの DSL | CLI / `.ad` ファイル / Skill / MCP |
| UI の見方 | 要素ツリー | 要素マッチング | アクセシビリティツリー + ref |
| エージェント連携 | 後付けで組み込む | 後付けで組み込む | 最初から前提にしている |
| CI での運用 | 自前構築または外部サービス | Maestro Cloud や CI 連携が用意されている | `.ad` の suite 実行はできる。テンプレート構想はあるが、専用基盤はまだ薄い |

テストケースを人間が明示的に書き、CI で安定実行したいなら Appium や Maestro が候補になります。一方で、Claude Code などのエージェントに「今のアプリを見て、この設定を変えて、結果を確認して」と依頼する用途では Agent Device のほうが自然です。

とはいえ、`.ad` ファイルがあるので Agent Device でも E2E のような再実行はできます。違いは「最初から人間が DSL を書く」よりも、「エージェントに実機 UI を探索させ、その成果を replay 可能な E2E シナリオに落とす」流れを作りやすい点です。

これは、アドホックな手動確認を回帰テストへ昇格させるまでの距離が短くなる、という意味で嬉しさがあります。バグ修正後の再現確認、仕様変更中のスモークテスト、アクセシビリティラベルや遷移の調査などを、まず自然言語で試し、安定した操作順と確認ポイントが見えたら `.ad` に残せます。

一方で、CI に載せる観点では Maestro のほうが専用基盤は整っています。[Maestro Cloud](https://docs.maestro.dev/maestro-cloud) には並列実行、デバイス分離、GitHub Actions / Bitrise / CircleCI などの CI 連携、PR 連携が用意されています。Agent Device も `.ad` の suite 実行、retry、artifacts 出力はできますし、[GitHub Actions / EAS 向けテンプレート](https://agent-device.dev/templates)も公開されています。ただしテンプレートは早期アクセス (Request Early Access) の位置づけなので、Maestro Cloud のようなマネージド実行基盤としてすぐ使えるものとは見え方が違います。どの runner で simulator / emulator を起動するか、成果物をどこへ集約するか、PR の必須チェックにどう組み込むかは、現時点ではプロジェクト側で設計する部分が大きそうです。

## セットアップ: CLI と Skill / MCP を入れる

セットアップは大きく 2 ステップです。

### 1. CLI をインストール

```bash
npm install -g agent-device@latest
agent-device --version
```

pnpm / yarn でも同様にグローバルインストールできます。グローバルを汚したくない場合は `npx agent-device@latest --version` のように都度実行する形でも動きます。

### 2. Claude Code に Skill を入れる

ここでは Claude Code を例にします。Skill 対応クライアントなら、公式 Skill を入れるのが一番手軽です。

```bash
npx skills add callstackincubator/agent-device
```

これで Claude Code のセッション内から `agent-device` Skill が利用可能になります。Skill は内部的に `agent-device help workflow` を読み込み、バージョンに合わせた正しいコマンド体系をエージェントに渡してくれます。

MCP に対応したクライアント (Cursor など) からは、MCP サーバとして登録します。

```json
{
  "mcpServers": {
    "agent-device": {
      "command": "agent-device",
      "args": ["mcp"]
    }
  }
}
```

グローバルインストールしていない場合は `command` を `npx`、`args` を `["-y", "agent-device@<バージョン>", "mcp"]` に差し替えます。

:::message
Skill と MCP は両方を入れなくてかまいません。クライアントが対応している方を 1 つ選べば十分です (Claude Code のように Skill 対応かつ CLI を叩ける環境なら Skill だけで完結します)。エージェント運用上はこの Skill / MCP のインストールが本命で、CLI を直接叩くのは挙動を理解したいときや CI/CD に組み込みたいときの選択肢、と整理しています。
:::

## Claude Code から自然言語で依頼する

Skill のインストールが終わったら、Claude Code のチャットで普通に依頼を書くだけです。実際に自分の Expo テンプレートで「設定画面から言語を English へ切り替えたとき UI が想定どおり英語化されるか」を確認させたとき、投げた一文がこれです。

> agent-device を使って、iOS シミュレーターで立ち上がっているアプリの設定の言語を English にしたときの動作確認、スクショを撮ってみて

エージェントが実行したコマンド列を順に並べると、こんな流れになりました。

1. `agent-device devices --platform ios` で稼働中のシミュレーターを列挙
2. `agent-device apps --platform ios` で対象アプリを探索 → 該当アプリは未インストールで Expo Go 経由と判断
3. `lsof -nP -iTCP:8081 -sTCP:LISTEN` で Metro が立っていることを確認
4. `agent-device open "Expo Go" exp://127.0.0.1:8081 --platform ios` でアプリを開く
5. `agent-device snapshot -i` で UI ツリーを取り、`@e26 [button] "設定, tab, 2 of 2"` のような ref を解析
6. 設定タブ → 言語セル → ピッカーから `English` を順にタップ
7. 主要画面ごとに `agent-device screenshot ./xxx.png` で証跡を保存

人間側はコマンド体系もアクセシビリティラベルの形式も知らなくて済みます。Skill が「`snapshot -i` を取って ref を ID で指す」「アクションのたびに取り直す」という規律をエージェントに守らせてくれます。

## (参考) CLI を直接触って挙動を理解する

普段は Skill 経由で済ませることが多くなりますが、デバッグ時や CI/CD に組み込むときには CLI を直接叩くことになります。基本パターンを押さえておくと、エージェントが何をしているかも理解しやすくなります。

```bash
# 1. アプリ一覧
agent-device apps --platform ios

# 2. アプリを起動 (bundle id を指定)
agent-device open com.apple.mobilesafari --platform ios

# 3. 操作可能な要素のスナップショット
agent-device snapshot -i
```

`snapshot -i` の出力イメージはこのような形です (整形済み)。

```text
@e1 button "Tabs"
@e2 textfield "Address" placeholder="Search or enter website name"
@e3 button "Bookmarks"
...
```

ここから先は ref を指定して操作します。

```bash
agent-device fill @e2 "https://zenn.dev"
agent-device screenshot ./artifacts/zenn-top.png
agent-device close
```

基本ループは `open -> snapshot -i -> press/fill/scroll -> verify -> close` です。画面遷移のたびに `snapshot` を取り直すのが鉄則です。

:::message
コマンド名・引数・サポートされているプラットフォームはバージョンによって変わります。実際の運用では `agent-device help workflow` の出力を一次情報として扱うのが安全です。
:::

## `.ad` ファイルで E2E シナリオを実行してみる

自然言語での依頼は探索には便利ですが、同じ確認を何度も実行したい場合は `.ad` ファイルとして保存しておくと扱いやすくなります。`.ad` は Agent Device のシナリオファイルで、`open`、`press`、`wait`、`is`、`close` のような操作を順に書けます。

今回のテンプレートでは、設定画面から言語を日本語に切り替え、UI が日本語化されたことを確認するシナリオを `e2e/settings-language.ad` として置きました。

```ad
# 設定タブから言語を日本語に切り替えて、UI が日本語化されたことを確認する再利用シナリオ。
# 実行: agent-device test ./e2e/settings-language.ad
# 前提: Metro 起動済み (pnpm start) かつ iOS シミュレーターで Expo Go が利用可能。
# 開始言語(en/ja)に依存しないよう、言語行とタブのラベルは || で両言語ぶんを列挙している。
context platform=ios device="iPhone 17 Pro" kind=simulator theme=unknown
open "Expo Go" "exp://127.0.0.1:8081"
wait 12000
press "label=\"Settings, tab, 2 of 2\" || label=\"設定, tab, 2 of 2\""
wait 1500
press "label=\"Language, English\" || label=\"Language, 日本語\" || label=\"言語, English\" || label=\"言語, 日本語\""
wait 1500
press "label=\"日本語\""
wait 1500
is "visible" "label=\"言語, 日本語\""
is "visible" "label=\"利用規約\""
close
```

実行は `agent-device test` です。事前に Metro を起動し、iOS シミュレーターで Expo Go が使える状態にしておきます。

```bash
agent-device test ./e2e/settings-language.ad
```

実行後は `.agent-device/test-artifacts/` 配下に結果が残ります。今回の実行では、次のような `result.txt` が生成されました。

```text
file: <workspace>/e2e/settings-language.ad
session: default:test:<run-id>:1-settings-language:attempt-1
attempt: 1/1
status: passed
replayed: 11
healed: 0
```

実際のファイルは、次のようなパスに出力されます。

```text
.agent-device/test-artifacts/<run-id>/e2e__settings-language.ad/attempt-1/result.txt
```

同じディレクトリには `replay.ad` も保存されており、どのシナリオを実行した結果なのかをあとから確認できます。

```text
.agent-device/test-artifacts/<run-id>/e2e__settings-language.ad/attempt-1/
├── replay.ad
└── result.txt
```

ポイントは、画面の文言やアクセシビリティラベルを条件として書けることです。上の例では開始言語が英語と日本語のどちらでも動くように、`||` で候補ラベルを列挙しています。`is "visible"` で `言語, 日本語` と `利用規約` が表示されていることまで確認しているので、単なる操作再生ではなく、最低限の期待値も含めた E2E シナリオになっています。

この形にしておくと、自然言語で試した確認を「その場限りの検証」で終わらせず、あとから再実行できるスモークテストにできます。最初から完成度の高い E2E を書くというより、エージェントに探索させた操作順、アクセシビリティラベル、確認ポイントを人間が確認し、`.ad` に落として育てていくイメージです。

CI で実行する場合も、コマンド自体は `agent-device test` で済みます。

```bash
agent-device test ./e2e --timeout 60000 --retries 1 --artifacts-dir ./tmp/agent-device-artifacts
```

ただし、モバイル CI として見ると、ここから先の環境づくりは別途必要です。たとえば、iOS シミュレーターや Android エミュレーターを起動する runner、Expo / Metro の起動、テスト用 app binary の準備が必要になります。スクリーンショットや replay log の保存、PR コメントや必須チェックへの反映なども、GitHub Actions などの CI 側で組むことになります。Agent Device 側にも GitHub Actions / EAS 向けテンプレートの構想はありますが、現時点では早期アクセス扱いです。このあたりは Maestro Cloud のようなマネージド基盤を持つツールとの差分として見ておくとよさそうです。

## まとめ

- Agent Device はモバイル UI を「AI エージェントが理解しやすい形」で公開する CLI で、スクリーンショットに頼らない設計が新鮮だった
- セットアップさえ終われば、CLI コマンドを覚えなくても自然言語で依頼するだけで動く。スクリーンショットではなく構造化スナップショットを返し、要素を短い ref `@eN` で指せるため、LLM のコンテキスト消費も抑えられる
- Skill / MCP が公式に用意されているため、コーディングエージェントに自然言語で依頼するワークフローと相性が良い
- `.ad` ファイルに残せば E2E のように再実行できるので、アドホックな手動確認をスモークテストへ昇格させやすい
- CI に載せること自体はできる。ただし Maestro Cloud のような一般利用しやすいマネージド実行基盤とはまだ見え方が違い、runner や artifacts 管理はプロジェクト側で設計する必要がある

## 参考リンク

- [Agent Device 公式サイト](https://agent-device.dev/)
- [Agent Device GitHub リポジトリ](https://github.com/callstackincubator/agent-device)
- [Agent Device: Replay & E2E Testing (公式ドキュメント)](https://oss.callstack.com/agent-device/docs/replay-e2e)
- [Agent Device Templates](https://agent-device.dev/templates)
- [Vercel agent-browser (着想元)](https://github.com/vercel-labs/agent-browser)
- [Callstack ブログ: Agent Device](https://www.callstack.com/blog/agent-device-ai-native-mobile-automation-for-ios-android)
- [Maestro Cloud ドキュメント](https://docs.maestro.dev/maestro-cloud)
