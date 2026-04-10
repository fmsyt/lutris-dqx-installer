# DQX Benchmark インストール要件

## システム要件

### OS
- Linux（Wayland + XWayland環境で動作確認済み）
- X11単独環境は未確認

### ホスト側パッケージ
| パッケージ | 用途 | 備考 |
|---|---|---|
| `lib32-freetype2` | Wine 32bitサポート用フォントライブラリ | Arch系: `sudo pacman -S lib32-freetype2` |

### Lutris
- バージョン: 0.5.22 で動作確認済み
- Lutris のインストールは各ディストリのパッケージマネージャーから

### Wine Runner（Lutris管理）
- `wine-staging-11.6-x86_64`（システムの wine-staging をシンボリックリンクで登録）
- インストール方法:
  ```bash
  sudo pacman -S wine-staging
  mkdir -p ~/.local/share/lutris/runners/wine/wine-staging-11.6-x86_64/bin
  for bin in wine wine64 wineboot wineserver winecfg; do
    ln -sf /usr/bin/$bin ~/.local/share/lutris/runners/wine/wine-staging-11.6-x86_64/bin/$bin
  done
  ```
- **注意**: WoW64モード専用。`WINEARCH=win32` は使用不可
- **注意**: `pacman -Syu` で wine-staging が更新されると再登録が必要

**動作しないバージョン:**
- `wine-ge-8-26`: winetricks の cmd.exe 起動にX11ディスプレイが必要で失敗
- `wine 7.x系`: Lutris環境で子プロセスが kernel32.dll を読めない

### winetricks（Lutris管理）
- Lutris 同梱の winetricks が使用される（自動）

---

## インストールされるコンポーネント（winetricks）

| コンポーネント | 用途 |
|---|---|
| `quartz` | DirectShow（動画・音声再生） |
| `wininet` | インターネット接続（ランチャー通信） |
| `winhttp` | HTTP通信 |
| `ie8` | Internet Explorer 8 コンポーネント |
| `dxtrans` | DirectX Transforms |
| `fakejapanese_ipamona` | 日本語フォント（IPA Mona） |
| `fakejapanese_vlgothic` | 日本語フォント（VL ゴシック） |

**使用しないもの:**
- `wmp10`: WoW64プレフィックスでインストール失敗（DQXには不要）

---

## ロケール
- `LC_ALL=ja_JP.UTF-8`（日本語表示のため）
- ホスト側に `ja_JP.UTF-8` ロケールが生成されていること
  - Arch系: `/etc/locale.gen` に `ja_JP.UTF-8 UTF-8` を追記して `locale-gen`

---

## ダウンロードされるファイル
| ファイル | URL |
|---|---|
| DQXBenchmarkInstaller.exe | https://download.dqx.jp/dqbenchmark/DQXBenchmarkInstaller.exe |

---

## コントローラー

### HORI PAD（動作確認済み）
- 初期状態ではAボタン等が認識されない
- **Steamをインストールすることで必要なドライバーが適用され動作するようになる**
  - Arch系: `sudo pacman -S steam`
  - Steam自体を起動する必要はなく、インストールするだけでよい
  - 備考: `steam-devices` パッケージ（HIDルール）が効いている可能性あり。`sudo pacman -S steam-devices` だけでも動作するかもしれないが未確認

### その他のコントローラー
- 動作確認なし（不明）

---

## 日本語入力（IME）

### fcitx5-mozc（動作確認済み）
- ゲーム内チャットで日本語入力を使うには **XIMサポートの有効化** が必要
- fcitx5 の設定でXIMアドオンを有効にする
  - `fcitx5-configtool` → アドオン → XIM を有効化
- 環境変数の設定も必要な場合あり（`~/.xprofile` や `~/.profile` に追記）:
  ```bash
  export XMODIFIERS=@im=fcitx
  export GTK_IM_MODULE=fcitx
  export QT_IM_MODULE=fcitx
  ```

### SteamDeck
- 日本語入力不可（既知の問題）

---

## 不明・未確認の要件
- **ハードウェア要件（GPU/VRAM/RAM等）**: 公式サイトを参照 https://hiroba.dqx.jp/sc/public/playguide/pcspec/
- **Wayland対応**: Wayland (XWayland経由) で動作確認済み。X11単独環境は未確認
- **SteamDeck**: 動作するが日本語入力不可（既知の問題）
