# DQX Benchmark Installer for Linux

[ドラゴンクエストX オンライン](https://www.dqx.jp/) ベンチマークソフトを Linux 上の Lutris でインストール・実行するための Lutris インストーラーです。

[Lutris 公式のインストーラー](https://lutris.net/games/dragon-quest-x-online/) が動作しなかったことをきっかけに作成しました。

インストール後はそのままゲーム本体としても使用できます。

---

## 動作確認環境

- **OS**: Arch Linux (CachyOS)
- **ディスプレイサーバー**: Wayland (XWayland経由)
- **Lutris**: 0.5.22
- **Wine**: wine-staging 11.6

### 動作しなかった Wine バージョン

| バージョン | 失敗の原因 |
|---|---|
| wine-ge-8-26 | winetricks の cmd.exe 起動に X11 ディスプレイが必要で失敗 |
| wine-staging-11.2 | Lutris API から削除済みで取得不可 |
| wine 7.x系 | Lutris 環境で子プロセスが kernel32.dll を読めない |

---

## 前提条件

### パッケージのインストール

```bash
# Lutris
sudo pacman -S lutris

# Wine staging
sudo pacman -S wine-staging

# 日本語ロケール（未生成の場合）
# /etc/locale.gen に `ja_JP.UTF-8 UTF-8` を追記してから:
sudo locale-gen

# フォントライブラリ（Wine 32bitサポート用）
sudo pacman -S lib32-freetype2
```

### Wine Runner の登録

Lutris の管理するランナーとしてシステムの wine-staging を登録します。

```bash
mkdir -p ~/.local/share/lutris/runners/wine/wine-staging-11.6-x86_64/bin
for bin in wine wine64 wineboot wineserver winecfg; do
  ln -sf /usr/bin/$bin ~/.local/share/lutris/runners/wine/wine-staging-11.6-x86_64/bin/$bin
done
```

> **注意**: `pacman -Syu` で wine-staging が更新されるとバージョンが変わり、インストーラーが動かなくなります。更新後は上記コマンドを再実行してください（ディレクトリ名のバージョン部分も合わせて変更が必要です）。

---

## インストール手順

```bash
lutris -i /path/to/dqx_installer.yaml
```

または Lutris の GUI からインストーラーファイルを指定して実行します。

---

## 既知の問題・注意事項

### Lutris 初回設定時のファイルエラー

初回インストール後、Lutris の設定画面でゲーム名などを編集すると game config ファイルのリネームに失敗することがあります。
その場合は以下のように手動でリネームしてください:

```bash
# ファイル名を確認
ls ~/.config/lutris/games/dragon-quest-x-online-*.yml

# -benchmark が抜けている場合はリネーム
mv ~/.config/lutris/games/dragon-quest-x-online-<hash>.yml \
   ~/.config/lutris/games/dragon-quest-x-online-benchmark-<hash>.yml
```

### Wine のバージョンを上げたい場合

Wine のバージョンを変更すると動画（ムービー）が再生されなくなる場合があります。
変更する場合はインストーラーを作り直してください。

### SteamDeck

日本語入力ができません（既知の問題）。

---

## コントローラー

### HORI PAD

初期状態ではボタンが認識されません。Steam をインストールすることで必要なドライバーが適用されます（起動は不要）。

```bash
sudo pacman -S steam
# または steam-devices だけでも動作する可能性あり（未確認）
sudo pacman -S steam-devices
```

---

## 日本語入力（IME）

fcitx5-mozc を使用している場合、ゲーム内チャットで日本語入力を使うには XIM サポートの有効化が必要です。

1. `fcitx5-configtool` を開く
2. アドオン → **XIM** を有効化

また、環境変数が設定されていない場合は `~/.xprofile` や `~/.profile` に追記してください:

```bash
export XMODIFIERS=@im=fcitx
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
```

---

## ハードウェア要件

GPU・VRAM・RAM等の要件は公式サイトを参照してください:
https://www.dqx.jp/online/start_game/product/windows.php

---

## 詳細

- [`REQUIREMENTS.md`](REQUIREMENTS.md) - インストール要件の詳細
- [`tried.md`](tried.md) - トラブルシューティング記録
