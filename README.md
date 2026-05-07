# シール手帳 🌈

小学生（6〜12歳）向けのデジタルシール制作・手帳アプリ。

**🔗 [sticker-notebook.hjncg9y7d4.workers.dev](https://sticker-notebook.hjncg9y7d4.workers.dev)**

---

## 特徴

- **ビルド不要** — `index.html` 1ファイルで完結。サーバー不要でそのまま動く
- **モバイルファースト** — iPhone Safari に最適化。ホーム画面に追加してアプリとして使える（PWA）
- **オフライン動作** — シールデータはブラウザの `localStorage` に保存
- **フレームワークなし** — vanilla JS + Canvas API のみ

## 機能

| 画面 | 内容 |
|------|------|
| シールメーカー | 形・色・スタンプ・ペンで自分だけのシールを作成 |
| 手帳 | 作ったシールを自由に貼り付け・移動・削除 |
| コレクション | 作成したシールの一覧 |
| QR共有 | ドット絵シールをQRコードで送受信 |

## 使い方

1. ブラウザで URL を開く
2. iPhoneの場合：Safari → 共有ボタン → **ホーム画面に追加**
3. シールメーカーでシールを作って手帳に貼る

## 開発

```bash
# ローカルで開く（ビルド不要）
open index.html

# または任意のHTTPサーバーで
npx serve .
```

mainブランチへのプッシュで Cloudflare Pages に自動デプロイされます。

## 技術構成

- **言語**: HTML / CSS / JavaScript（ES2020）
- **描画**: Canvas API（2層オーバーレイ）
- **永続化**: localStorage
- **ホスティング**: Cloudflare Pages
- **アイコン**: SVG Data URI（外部ファイルなし）
