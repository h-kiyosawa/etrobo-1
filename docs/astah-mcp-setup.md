# WSL ロボコン環境と Windows Astah の MCP 接続手順書

WSL2 上の ETロボコン開発環境(Claude Code)から、Windows 側の astah* professional を
MCP (Model Context Protocol) 経由で操作できるようにするための手順書。

接続が完了すると、Claude Code から Astah の UML モデル(クラス図・シーケンス図・
ステートマシン図・アクティビティ図など)の参照・作成・編集ができるようになる。

## 構成イメージ

```
┌─ Windows ─────────────────────────────┐
│  astah* professional                  │
│   └─ MCP プラグイン                    │
│       └─ MCP サーバー                  │
│          http://127.0.0.1:3099/mcp    │
│                 ▲                     │
│                 │ localhost 共有       │
│                 │ (WSL ミラーモード)    │
│  ┌─ WSL2 ───────┼──────────────────┐  │
│  │  Claude Code (etrobo 環境)       │  │
│  └──────────────────────────────────┘  │
└───────────────────────────────────────┘
```

## 前提条件

| 項目 | 要件 |
|------|------|
| OS | Windows 11 22H2 以降(WSL ミラーモードに必要) |
| WSL | WSL2 |
| astah | astah* professional v10.1 以降(MCP プラグイン使用時)。v11 以降は AI Chat Copilot プラグインが標準搭載 |
| AI クライアント | Claude Code(WSL 内にインストール済み) |

## 手順 1: Astah 側で MCP サーバーを有効化(Windows)

1. MCP プラグインをインストールする
   - v10.1〜v10.x: 配布されている `.jar` ファイルを Astah の図上にドラッグ&ドロップし、Astah を再起動
   - v11 以降: AI Chat Copilot プラグインが標準搭載のためインストール不要
2. Astah を起動し、拡張ビューから **Chat Copilot ペイン**を開く
3. **Settings → MCP** セクションで MCP サーバーを**有効化**する
   - ポートはデフォルトの **3099** のままでよい(変更した場合は以降の手順の
     ポート番号を読み替えること)
4. 動作確認: Windows の PowerShell で以下を実行し、応答があることを確認

   ```powershell
   curl.exe -s -o NUL -w "%{http_code}" http://localhost:3099/mcp
   ```

   `400` が返れば正常(MCP エンドポイントは素の GET に 400 を返す仕様)。
   接続エラーになる場合は MCP サーバーが起動していない。

> **注意:** Astah の MCP サーバーは AI クライアントより**先に起動**しておくこと。

## 手順 2: WSL をミラーモードに変更(Windows)

Astah の MCP サーバーは `127.0.0.1` のみで待ち受けるため、デフォルトの
NAT モードの WSL2 からは到達できない(`localhost` は WSL 内を指し、
Windows ホスト IP 経由もループバック限定バインドのため接続不可)。
WSL の**ミラーモード**を使うと、WSL 内の `localhost` が Windows の
loopback と共有され、そのまま接続できるようになる。

1. `C:\Users\<ユーザー名>\.wslconfig` を作成(既存の場合は `[wsl2]`
   セクションに追記)する:

   ```ini
   [wsl2]
   networkingMode=mirrored
   ```

2. Windows の PowerShell で WSL を再起動する
   (**WSL 内で実行中の作業はすべて終了するので注意**):

   ```powershell
   wsl --shutdown
   ```

3. WSL のターミナルを開き直し、ミラーモードになったことを確認する:

   ```bash
   wslinfo --networking-mode   # => mirrored と表示されればOK
   ```

## 手順 3: Claude Code に MCP サーバーを登録(WSL)

1. etrobo 環境のディレクトリで以下を実行する:

   ```bash
   claude mcp add --transport http astah http://localhost:3099/mcp
   ```

   ※ デフォルトではプロジェクトスコープ(`~/.claude.json` の該当プロジェクト配下)に
   登録される。他のディレクトリからも使いたい場合は `--scope user` を付ける。

2. 疎通確認:

   ```bash
   curl -s -o /dev/null -w "%{http_code}" http://localhost:3099/mcp
   # => 400 が返ればOK(接続自体は成功している)
   ```

## 手順 4: 接続確認

1. **Astah を起動した状態で** Claude Code を起動(または再起動)する
2. Claude Code 内で `/mcp` を実行し、`astah` サーバーが `connected` に
   なっていることを確認する
3. **初回接続時は Astah 側に接続リクエストのダイアログが表示される**ので
   **[Connect]** をクリックする
4. `getOrCreateClassDiagram` や `getDiagramIdsInProject` などの
   `mcp__astah__*` ツール群が使えるようになれば完了

## トラブルシューティング

### `/mcp` で「Connection refused」になる

原因の切り分け手順:

1. **Astah 側のサーバーが起動しているか** — WSL から Windows 側を直接確認できる:

   ```bash
   powershell.exe -NoProfile -Command "netstat -ano | Select-String ':3099'"
   ```

   `127.0.0.1:3099 ... LISTENING` が出なければ、Astah が未起動か
   MCP サーバーが無効。手順 1 を確認する。

2. **WSL がミラーモードになっているか**:

   ```bash
   wslinfo --networking-mode
   ```

   `nat` と表示される場合は手順 2 が未反映。`.wslconfig` の配置場所
   (Windows 側のユーザーフォルダ直下)と、`wsl --shutdown` を実行したかを確認する。

3. **起動順序** — Astah → Claude Code の順で起動する。Claude Code 起動後に
   Astah を立ち上げた場合は `/mcp` から再接続する。

### ミラーモードにできない場合(Windows 10 など)

代替として、Windows 側でポートプロキシを設定する(管理者 PowerShell):

```powershell
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=3099 connectaddress=127.0.0.1 connectport=3099
netsh advfirewall firewall add rule name="WSL Astah MCP" dir=in action=allow protocol=TCP localport=3099
```

この場合、Claude Code への登録 URL は `localhost` ではなく WSL から見た
Windows ホスト IP(`ip route show default` で表示されるゲートウェイ、
例: `http://172.27.224.1:3099/mcp`)を使う。ただしこの IP は WSL の
再起動で変わることがある点に注意。

### ポートを 3099 から変更した場合

Astah 側の設定(手順 1)と Claude Code の登録 URL(手順 3)の両方を
同じ番号に揃えること。登録し直す場合:

```bash
claude mcp remove astah
claude mcp add --transport http astah http://localhost:<新ポート>/mcp
```

## 参考リンク

- [Astah MCP Server Setup Guide](https://astah.net/support/astah-mcp-servers/)
- [Astah Pro MCP プラグイン](https://astah.change-vision.com/ja/plugin/astah-pro-mcp.html)
- [Claude Code MCP ドキュメント](https://code.claude.com/docs/ja/mcp)
