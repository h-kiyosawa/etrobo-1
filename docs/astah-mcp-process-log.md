# プロセスログ: MCP で WSL の Claude Code と Windows の Astah をつなぐまで

2026-07-06 実施。WSL2 上の ETロボコン開発環境で動く Claude Code から、
Windows 側の astah* professional を操作できるようにした際の記録。

「そもそも MCP とは何か」の解説を重点に、実際にやったこと・つまずいたこと・
その原因と解決策を時系列でまとめる。手順だけを知りたい場合は
[astah-mcp-setup.md](./astah-mcp-setup.md) を参照。

---

## 1. そもそも MCP ってなに？

### 一言でいうと

**MCP (Model Context Protocol)** は、AI アシスタント(Claude など)と
外部のツール・データソースをつなぐための**共通プロトコル(通信の共通規格)**。
Anthropic が 2024 年 11 月にオープン標準として公開し、現在は Claude 以外の
多くの AI エージェント(Codex CLI、Gemini CLI、Cursor、VSCode Copilot など)も
対応している。

よく使われる例えは **「AI にとっての USB-C」**。
USB-C 以前は機器ごとに専用ケーブルが必要だったように、MCP 以前は
「Astah × Claude 用の連携」「Astah × Copilot 用の連携」…と、
ツールと AI の組み合わせごとに個別の連携実装が必要だった(M×N 問題)。
MCP という共通規格に両側が対応すれば、**どの AI からでもどのツールでも
同じ方法でつながる**(M+N で済む)。

### 登場人物は 3 つ

| 役割 | 説明 | 今回の例 |
|------|------|----------|
| **MCP ホスト** | AI エージェント本体。ユーザーと対話する | Claude Code(WSL 内) |
| **MCP クライアント** | ホストに内蔵され、サーバーとの通信を担当する | Claude Code が内部に持つ |
| **MCP サーバー** | ツールやデータを AI に提供する側 | Astah の MCP プラグイン(Windows 内) |

MCP サーバーは AI に対して主に **ツール(Tools)** を公開する。
ツールとは「AI が呼び出せる関数」のことで、Astah の場合は
「クラス図を作る」「クラスの属性一覧を取得する」「PlantUML を Astah の図に変換する」
といった操作が 100 個以上のツールとして公開されている。
AI はユーザーの依頼(例:「このコードのクラス図を Astah に描いて」)を解釈し、
適切なツールを適切な引数で呼び出す。

### 通信の仕組み

- 中身は **JSON-RPC 2.0** というシンプルなリクエスト/レスポンス形式
- 接続方法(トランスポート)は主に 2 種類:
  - **stdio** — AI がサーバープロセスを子プロセスとして起動し、標準入出力で通信。
    ローカル専用ツールに多い
  - **HTTP** — サーバーが Web サーバーとして待ち受け、AI が HTTP で接続。
    **Astah はこちら**。デフォルトで `http://localhost:3099/mcp` に立つ
- 接続すると AI はまず「どんなツールがあるか」の一覧を取得し、
  以降は必要に応じてツールを呼び出す

### 今回の構成を MCP 用語で描くと

```
[ユーザー] ──依頼──> [Claude Code = MCPホスト]          (WSL2)
                          │ MCPクライアント
                          │ HTTP (JSON-RPC)
                          ▼
                     http://localhost:3099/mcp
                          │
                     [MCPサーバー = Astahプラグイン]     (Windows)
                          │ ツール呼び出しを実行
                          ▼
                     [astah* professional の UMLモデル]
```

ポイント: **ホスト(WSL)とサーバー(Windows)が別の OS 環境にいる**こと。
これが今回のつまずきの原因のすべてだった(後述)。

---

## 2. プロセスログ(時系列)

### Step 1: 調査 — Astah の MCP プラグインはどう待ち受けるのか

公式ドキュメントを調べ、以下が判明:

- Astah v11 以降は AI Chat Copilot プラグインが標準搭載で、その中に MCP サーバー機能がある
- トランスポートは **HTTP**、デフォルトポートは **3099**、エンドポイントは `/mcp`
- Astah を AI クライアントより先に起動しておく必要がある

### Step 2: 疎通テスト → 失敗

WSL 側から接続を試みたが、どちらも失敗:

```bash
curl http://localhost:3099/mcp      # => Connection refused
curl http://172.27.224.1:3099/mcp   # => Connection timed out (WindowsホストIP経由)
```

### Step 3: 原因調査① — WSL のネットワークモード

```bash
wslinfo --networking-mode   # => nat
```

WSL2 のデフォルトは **NAT モード**で、WSL は Windows とは別の仮想ネットワークにいる。
つまり **WSL 内の `localhost` は WSL 自身を指し、Windows の `localhost` とは別物**。
`localhost:3099` への接続が拒否されたのはこのため。

### Step 4: 原因調査② — そもそも Astah 側は動いているのか

WSL からは届かないが、サーバー自体が動いているかは切り分けたい。
WSL の interop 機能(WSL から Windows のコマンドを直接実行できる)を使い、
**Windows 側の視点で** localhost:3099 を叩いた:

```bash
powershell.exe -NoProfile -Command "Invoke-WebRequest -Uri 'http://localhost:3099/mcp' ..."
# => HTTP 400
```

**HTTP 400 はサーバー稼働の証拠**。MCP の HTTP エンドポイントは、MCP の
プロトコルヘッダーを持たない素の GET に対して 400 を返す仕様のため、
「400 が返る = サーバーは生きていて、届いてさえいれば話せる」ことを意味する。

### Step 5: 原因調査③ — なぜホスト IP 経由でも届かないのか

```bash
powershell.exe -NoProfile -Command "netstat -ano | Select-String ':3099'"
# => TCP  127.0.0.1:3099  LISTENING
```

Astah の MCP サーバーは **127.0.0.1(ループバック)のみで待ち受け**ていた。
ループバック限定で待ち受けるサーバーには、外部ネットワーク扱いになる
NAT モードの WSL からはホスト IP 経由でも到達できない。これで原因が確定:

> **NAT モードの WSL2 から、Windows の 127.0.0.1 限定サーバーには
> どの経路でも直接届かない。**

なお、外部から接続できないのはセキュリティ的には正しい設計
(LAN 上の他マシンから勝手にモデルを操作されない)であり、
Astah 側の不具合ではない。

### Step 6: 解決策の選定 — ミラーモード vs ポートプロキシ

| 案 | 仕組み | 採否 |
|----|--------|------|
| **WSL ミラーモード** | WSL のネットワークを Windows と共有し、`localhost` を共通化する | ✅ 採用 |
| netsh ポートプロキシ | Windows 側で 0.0.0.0:3099 → 127.0.0.1:3099 の転送を挟む | 不採用 |

ミラーモードは Windows 11 22H2 以降が必要だが、今回の環境は build 26200 で
問題なし。ポートプロキシ案は管理者権限が必要なうえ、接続先に使う WSL の
ゲートウェイ IP が再起動で変わりうるため、ミラーモードを採用した。

### Step 7: ミラーモード化

`C:\Users\<ユーザー名>\.wslconfig` を新規作成:

```ini
[wsl2]
networkingMode=mirrored
```

Windows の PowerShell で `wsl --shutdown` を実行して WSL を再起動
(実行中の Claude Code セッションも終了するため、`claude --continue` で再開)。

### Step 8: Claude Code に登録 → 接続成功

```bash
claude mcp add --transport http astah http://localhost:3099/mcp
```

`/mcp` で接続を確認。初回接続時に Astah 側に表示された接続リクエスト
ダイアログで [Connect] をクリックし、`mcp__astah__*` のツール群
(クラス図・シーケンス図・ステートマシン図の作成、PlantUML 変換など)が
Claude Code から利用可能になった。**接続成功。**

---

## 3. 学び・つまずきポイントまとめ

1. **WSL2 (NAT) の `localhost` は Windows の `localhost` ではない。**
   WSL ⇔ Windows 間で localhost 待ち受けのサーバーにつなぐなら、
   ミラーモードが最もシンプルな解。

2. **MCP エンドポイントの HTTP 400 は「正常稼働」のサイン。**
   ブラウザや curl で素の GET を投げて 400 が返れば、サーバーは生きている。

3. **WSL interop (`powershell.exe`) は切り分けの強力な武器。**
   「WSL から見えるか」と「Windows 自身から見えるか」を同じターミナルから
   比較でき、ネットワークの問題かサーバーの問題かを一発で切り分けられる。

4. **起動順序に注意。** MCP サーバー(Astah)→ MCP ホスト(Claude Code)の順。
   Claude Code 起動後に Astah を立ち上げた場合は `/mcp` から再接続すればよい。

## 関連ドキュメント

- 接続手順書(再現用): [astah-mcp-setup.md](./astah-mcp-setup.md)
- [MCP 公式サイト](https://modelcontextprotocol.io/)
- [Claude Code MCP ドキュメント](https://code.claude.com/docs/ja/mcp)
- [Astah MCP Server Setup Guide](https://astah.net/support/astah-mcp-servers/)
