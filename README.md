# ステロイド誘発性高血糖 インスリン調整支援ツール

非ICU・朝1回PSL想定のパターンに対する、インスリン調整の支援ツール（医師個人使用）。
**このリポジトリが正本**で、GitHub Pages で配信される版が唯一の最新版です。

公開URL（iPhoneホーム画面追加可）：https://yuukenn384-code.github.io/insulin-steroid-tool/

## 収録
| ファイル | 内容 |
|---|---|
| `index.html` | 本体（単一HTML・日本語UI・オフライン動作） |
| `HANDOVER.md` | 設計・臨床ロジック・既知の負債・拡張方針の引き継ぎ（冒頭「2026-06-12 改訂サマリ」を最優先で読む） |

データは端末内 localStorage のみ（外部送信なし・端末間は export/import）。臨床判断支援であり最終決定は指示医。

## 改訂のしかた（単一正本ワークフロー）
編集は**このローカルクローン（`~/insulin-steroid-tool`）でのみ**行う。手元コピーは作らない（版管理は git 履歴に一本化）。

```bash
cd ~/insulin-steroid-tool
git pull                       # 最新化
# index.html を編集（ブラウザでローカルプレビュー・iPhone実機相当を確認）
git add -A && git commit -m "改訂内容"
git push                       # GitHub Pages に自動反映
```

- 過去版に戻すときは git 履歴から（`git log` → 任意コミット）。手動バックアップは不要。
- iCloud `インスリンツール/` は旧素材を退避済みで**廃止**（編集対象ではない）。
- 派生アプリの作り方・テスト：`AI-Company/Archive/2026-06-12_インスリンツール改訂/`。
