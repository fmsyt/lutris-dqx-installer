# DQX Installer トラブルシューティング記録

## 問題
Lutrisで32bitアプリ（DQX Benchmark）がインストールできない。`WINEARCH=win32`を渡しても32bit動作されていない。

## 調査・試したこと

### 1. 診断YAMLの作成 (`dqx_32bit_diagnostic.yaml`)
環境変数・Wineバイナリ・ホスト側32bitライブラリ・プレフィックスのアーキテクチャを確認するための診断インストーラーを作成。

**診断結果:**
- `WINEARCH=win32` は正しく渡されていた ✅
- `/usr/lib32` は存在（536ファイル） ✅
- `system.reg: #arch=win64` → **プレフィックスが64bitで作られていた** ❌

### 2. 既存プレフィックス削除ステップの追加
`create_prefix` の前に `$GAMEDIR` を削除する `execute` ステップを追加。

### 3. Wineランナーの問題発覚
`wine-staging-11.2-x86_64`（LutrisデフォルトのRunner）はWoW64モードでビルドされており、win32プレフィックスに非対応。

```
WARNING: Proton is not compatible with 32-bit prefixes, forcing win64
wine: WINEARCH is set to 'win32' but this is not supported in wow64 mode.
```

### 4. Wine 7.2のダウンロード
GitHubから `lutris-wine-7.2-2-x86_64` をダウンロードしてインストール。
- URL: https://github.com/lutris/wine/releases/download/lutris-wine-7.2-2/wine-lutris-7.2-2-x86_64.tar.xz
- 配置先: `~/.local/share/lutris/runners/wine/lutris-7.2-2-x86_64`
- 直接テストでは `#arch=win32` のプレフィックス作成に成功 ✅

YAMLに `wine.version: lutris-7.2-2-x86_64` を追加。

### 5. winetricksタスクが依然として wine-staging-11.2 を使用する問題
`wine.version` は `create_prefix` には効くが、`winetricks` タスクは無視して `wine-staging-11.2` を使ってしまう。
→ `winetricks` タスクをすべて `execute` コマンドに置き換え、`WINE=` を明示指定。

### 6. winetricks コマンドが見つからない
`execute` の環境ではPATHが限られており `winetricks` が見つからなかった。
→ フルパス `/home/motsuni/.local/share/lutris/runtime/winetricks/winetricks` を指定。

### 7. FreeType ライブラリ不足
winetricks が `wine cmd.exe /c echo '%AppData%'` を実行した際に失敗:
```
Wine cannot find the FreeType font library.
```
`lib32-freetype2` が未インストール。
```bash
sudo pacman -S lib32-freetype2
```
→ インストール後に再試行。

### 8. Lutrisのcreate_prefixが不完全なプレフィックスを作成
`lib32-freetype2` インストール後も `wine: could not load kernel32.dll, status c0000135` で失敗。
`create_prefix` タスクが作成したプレフィックスは `drive_c/windows` しかなく `Program Files` 等が欠けていた。

→ `create_prefix` タスクをやめ、`execute` で直接 `wineboot --init` を呼ぶ方式に変更。
→ `wineboot --init` では `Program Files` 等が揃った完全なディレクトリ構造が作成できた ✅

### 9. winetricksが .reg ファイルを見つけられず失敗
`wineboot --init` は成功するが `.reg` ファイル（`system.reg` 等）が作成されない。
winetricks がプレフィックス検証に `*.reg` を使うため、`%AppData%` の取得に失敗して exit code 256 で停止。

```
grep: /home/motsuni/Games/dragon-quest-x-online/*.reg: そのようなファイルやディレクトリはありません
warning: wine cmd.exe /c echo '%AppData%' returned empty string
```

### 10. XAUTHORITY が Lutris execute 環境に引き継がれない問題の特定
ターミナルから直接 `wineboot --init` を実行すると `.reg` ファイルが作成される（`DISPLAY=:0` + `XAUTHORITY` あり）。
Lutris の `execute` コマンドは `XAUTHORITY` を引き継がないため X11 に接続できず、explorer.exe 等が起動失敗 → `.reg` ファイルが作成されない。

`DISPLAY=:0` を `execute` コマンドに明示しても `XAUTHORITY` がないため効果なし。

→ `setup_prefix.sh` スクリプトを作成し、ターミナルからプレフィックスを事前作成する方式を試みた。
→ 事前作成は成功（`.reg` ファイルあり、`#arch=win32` 確認）したが、winetricks の `%AppData%` 取得は依然失敗。
　（wine 7.2 で子プロセスが kernel32.dll を読み込めないため）

### 11. wine 7.2 の子プロセス問題
wine 7.2 では `create_prefix` の `wineboot` は成功するが、その後の `wine cmd.exe` 等の子プロセスが kernel32.dll を読み込めない。
winetricks はプレフィックス検証に `wine cmd.exe /c echo '%AppData%'` を実行するため、ここで必ず失敗する。

```
wine: Unhandled page fault on execute access to 00000000 at address 00000000 (thread 002c)
wine: could not load kernel32.dll, status c0000135
```

### 12. 方針転換: wine-staging-11.2 の win64 (WoW64) プレフィックスに切り替え
wine-staging-11.2 で win64 プレフィックス（WoW64モード）を作成してテストした結果：
- `.reg` ファイル ✅
- `Program Files (x86)` を含む完全な drive_c ✅
- `%AppData%` = `C:\users\motsuni\AppData\Roaming` ✅

DQX ベンチマーク（32bit EXE）は WoW64 モードで `Program Files (x86)` にインストールされて動作する。

### 13. create_prefix が WINEARCH=win32 を強制する問題
`game.arch: win32` が YAML にあると、Lutris の `create_prefix` タスクが `WINEARCH=win32` を強制してしまう。
wine-staging-11.2 (WoW64) は `WINEARCH=win32` を拒否するため、prefix 作成が失敗する。

→ `game.arch: win32` を削除して解決。

また、`winetricks:` という記法は Lutris 0.5.22 に存在せず `The command "winetricks" does not exist.` エラーになった。
→ 正しい記法 `task: name: winetricks` に変更。

### 14. wmp10 が WoW64 プレフィックスでインストール失敗
winetricks の `wmp10` が WoW64 プレフィックスで `l3codecp.acm` が見つからずに exit 256 で停止。
（`l3codeca.acm` はインストールされているが winetricks は `l3codecp.acm` を期待する）
DQX ベンチマークに wmp10 は不要なため、削除して解決。

### 15. wine-staging-11.2 が Lutris API から削除されていた
クリーンインストール後、wine-staging-11.2 が Lutris API から取得できなくなっていた（404）。

試した代替案：
- **wine-ge-8-26**（Lutris APIで唯一提供されるバージョン）: winetricks の `wine cmd.exe /c echo '%AppData%'` がX11ディスプレイなしで失敗 → NG
- **`wine.version: system`**: Lutris が API から情報取得しようとして失敗 → NG
- **システムの wine-staging をシンボリックリンクで登録**: ✅

```bash
mkdir -p ~/.local/share/lutris/runners/wine/wine-staging-11.6-x86_64/bin
for bin in wine wine64 wineboot wineserver winecfg; do
  ln -sf /usr/bin/$bin ~/.local/share/lutris/runners/wine/wine-staging-11.6-x86_64/bin/$bin
done
```

YAMLの `wine.version: wine-staging-11.6-x86_64` に合わせて指定。

**注意**: `pacman -Syu` で wine-staging が更新されるとバージョンが変わるため、再登録が必要になる。

## 最終的な解決策
- `wine.version: wine-staging-11.6-x86_64`（システムの wine-staging をシンボリックリンクで登録）
- `game.arch` は指定しない（win64 WoW64 プレフィックスを作成）
- `game.exe: drive_c/Program Files (x86)/...`（32bit アプリは WoW64 で動作）
- winetricks は `task: name: winetricks` 記法を使用
- `wmp10` は使用しない

**インストール成功！** 🎉

## インストール後の注意

### Lutris設定画面で初回編集するとgame configファイルの参照が壊れる
インストール直後にLutrisの設定画面でゲーム設定を編集・保存すると、game configファイルのリネームに失敗してエラーが出る場合がある。

```
FileNotFoundError: '.../dragon-quest-x-online-benchmark-XXXXXXXXXX.yml'
  -> '.../dragon-quest-x-online-XXXXXXXXXX.yml'
```

- 原因: インストーラーYAMLの `slug`（`dragon-quest-x-online-benchmark`）と `game_slug`（`dragon-quest-x-online`）が異なるため、Lutrisがリネーム先のファイルを見つけられない
- 結果: 参照先ファイルがなくなり、個別設定（Wine prefixパス等）が失われる
- 対処: エラーログに表示されるファイル名を確認し、手動でリネームする

```bash
mv ~/.local/share/lutris/games/dragon-quest-x-online-XXXXXXXXXX.yml \
   ~/.local/share/lutris/games/dragon-quest-x-online-benchmark-XXXXXXXXXX.yml
```

リネーム後は設定の保存が正常に動作する。

## Linux Mint での SSL/TLS エラー調査

Arch Linux では動作した構成を Linux Mint に持っていったところ、ランチャー側で **「SSL 通信に失敗」「TLS1.2 がない」** ようなエラーが発生した。

### 16. ブートローダー配下の別プロセスを `strace -ff` で追跡

ブートローダーが別プロセスを起動するため、通常の Lutris ログでは通信失敗箇所が見えなかった。

そのため、以下のように `strace -ff` でプロセス分岐後の通信・ファイルアクセスを追跡した。

```bash
strace -ff -s 256 \
  -e trace=process,network,openat,connect,recvfrom,sendto \
  -o /tmp/dqx/dqx.trace \
  lutris -d
```

### 17. 通信自体は成立していることを確認

トレースから以下を確認:

- `DQXBoot.exe` が `winhttp.dll` / `wininet.dll` / `crypt32.dll` / `schannel.dll` / `secur32.dll` を読み込んでいる
- `DQXBoot.exe` が `202.67.58.124:443` に接続している
- `DQXLauncher.exe` が `game.dqx.jp` を名前解決している
- `game.dqx.jp` への HTTP リクエスト `POST /smgame/patch/check/version?...` が送信されている

→ DNS 不通や単純な TCP 接続失敗ではない。

### 18. `ca-certificates` は既に導入済み

証明書不足も疑ったが、Mint 側では `ca-certificates` は既にインストール済みだった。

また、トレース上でも以下のように証明書ストア参照は行われていた:

```text
/etc/ssl/certs/ca-certificates.crt
/usr/lib/ssl/cert.pem
```

### 19. `ntlm_auth` 不足を特定

`schannel` / `secur32` 初期化の直後に、Wine が外部コマンド `ntlm_auth --version` を探して起動しようとしていることを確認。

しかし Linux Mint 環境では `ntlm_auth` が存在せず、以下のように全パスで `ENOENT` → `exit 127` になっていた:

```text
execve("/usr/bin/ntlm_auth", ["ntlm_auth", "--version"], ...) = -1 ENOENT
...
exit_group(127)
```

→ 表面的には「TLS1.2 がない」エラーだが、実際には **Wine の `schannel` / `secur32` 系が依存する `ntlm_auth` 欠如** が原因候補として最も有力。

### 20. `winbind` をインストール

Mint / Ubuntu 系では `ntlm_auth` は通常 `winbind` パッケージに含まれるため、以下を導入した。

```bash
sudo apt install winbind
```

その後、以下で `ntlm_auth` が見えるか確認する想定:

```bash
which ntlm_auth
ntlm_auth --version
```

### 21. `ntlm_auth` は解消したが、エラーは継続

`winbind` 導入後に再度 `strace -ff` を取得したところ、`ntlm_auth` は以下のように正常に起動して `exit 0` していた:

```text
execve("/usr/bin/ntlm_auth", ["ntlm_auth", "--version"], ...) = 0
...
exit_group(0)
```

それでもランチャー画面上では同じ SSL / TLS1.2 系エラーが継続したため、`ntlm_auth` 欠如は副次的な問題で、まだ別の原因があると判断。

### 22. Lutris のゲーム設定ファイルを確認

Lutris のゲーム設定は `~/.config/lutris/games/` にあり、以下の 2 つが存在した:

- `dragon-quest-x-online-benchmark-1754049642.yml`
- `dragon-quest-x-online-benchmark-1778774054.yml`

このうち古い方は `wine-ge-8-26-x86_64` + `win32` の古い設定で、現在の検証対象は新しい方だった。

新しい設定ファイルでは、以下のように **見かけ上は** `wine-staging-11.6-x86_64` が指定されていた:

```yaml
wine:
  version: wine-staging-11.6-x86_64
```

また、各 `task` の `wine_path` も `~/.local/share/lutris/runners/wine/wine-staging-11.6-x86_64/bin/wine` を指していた。

### 23. runner 名と実体の Wine バージョンがずれていた

runner 実体を確認すると、`wine-staging-11.6-x86_64` ディレクトリ内のバイナリは実体ファイルではなく `/usr/bin/wine` などへのシンボリックリンクだった:

```text
~/.local/share/lutris/runners/wine/wine-staging-11.6-x86_64/bin/wine -> /usr/bin/wine
~/.local/share/lutris/runners/wine/wine-staging-11.6-x86_64/bin/wine64 -> /usr/bin/wine64
```

さらに Mint 側のシステム Wine は以下だった:

```bash
wine --version
# wine-9.0 (Ubuntu 9.0~repack-4build3)
```

トレースでも実際に起動していたのは `execve("/usr/lib/wine/wine", ...)` であり、**実行時に使われていたのは system Wine 9.0** だった。

→ つまり、**設定上は `wine-staging-11.6-x86_64` を選んでいても、runner 実体が `/usr/bin/wine` 参照のため、中身は 11.6 ではなかった**。

### 24. 原因整理: 固定バージョン runner のつもりが system Wine 参照になっていた

Arch で動作確認した時点では system の `wine-staging` が 11.6 だったため、このシンボリックリンク方式でも結果的に 11.6 相当で動いていた可能性が高い。

しかし Mint 側では `/usr/bin/wine` の実体が別バージョン（少なくとも確認時点では 9.0）だったため、**同じ runner 名でも実際の Wine バージョンが変わってしまっていた**。

そのため、今回の Linux Mint 側の不具合は **「11.6 を指定したのに無視された」というより、「11.6 という名前の runner 自体が system Wine 参照で、中身が別バージョンにすり替わっていた」** と整理するのが正確。

### 25. 状況変化: 通信とダウンロードが開始

その後、裏でダウンロード処理が動き始め、実際に通信できていることを確認。

したがって現時点では、Linux Mint 側の SSL / TLS1.2 エラーは **Wine runner 名と実体バージョンのズレに起因する一時的な問題だった可能性が高い**。

最終的な動作可否やどの Wine バージョンで安定するかは、別途検証結果を整理する。

### 26. `winbind` を削除しても動作継続

追加確認として、Mint 側で `apt` から `winbind` を削除した状態でも、通信・ダウンロード処理は継続できそうなことを確認。

→ この結果から、`winbind` / `ntlm_auth` は今回の根本原因ではなく、**必須要件ではない可能性が高い**。

現時点での有力な整理は以下:

- 表面的な SSL / TLS1.2 エラーの主因は `winbind` 不足ではない
- 主因は **runner 名と実体 Wine バージョンのズレ**
- `winbind` は切り分けの途中で見つかった副次的な要素だった可能性が高い

## ファイル構成
- `dqx_installer.yaml` - メインインストーラー（最終版）
- `dqx_32bit_diagnostic.yaml` - 診断用インストーラー
- `run_diagnostic.sh` - 診断実行スクリプト
- `setup_prefix.sh` - ターミナルからプレフィックスを手動作成するスクリプト（不要になったが参考として残す）
- `tried.md` - この記録ファイル
