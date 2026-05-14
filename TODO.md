# TODO

## 検証待ち

- [ ] Linux Mint で使用した Wine バージョンごとの検証結果を記録する
  - runner 名ではなく実際に参照された Wine 実体のバージョンも併記する
  - 通信可否（SSL / TLS1.2 エラー有無、ダウンロード開始可否）を書く
  - 結果に応じて README.md / REQUIREMENTS.md / tried.md の記載を更新する

- [ ] `lib32-freetype2` なしで再インストールして挙動を確認する
  - 文字化けだけで済むか、クラッシュ・起動失敗になるか
  - 結果に応じて REQUIREMENTS.md の記載を更新する

## 未確認の要件（REQUIREMENTS.md に反映予定）

- [ ] GPU要件を調べて記録する
- [ ] X11環境での動作を確認する
