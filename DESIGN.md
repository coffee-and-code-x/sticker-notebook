# デジタルシール手帳 設計仕様書

## 概要

単一 HTML ファイル（`index.html`）で完結するモバイルファーストのシール制作アプリ。  
対象ユーザーは小学生（6〜12歳）。フレームワーク・ビルドツール不使用、vanilla JS + Canvas API のみ。

---

## アーキテクチャ

```
index.html
├── <style>          CSS（変数・レイアウト・コンポーネント）
├── HTML             4画面のシェル + シールメーカー UI
└── <script>         モジュール構成（オブジェクトリテラル）
    ├── 定数          COLOR_PALETTE / PEN_COLORS / STAMP_LIST
    ├── state         単一グローバルオブジェクト
    ├── StickerStorage
    ├── ShapeRenderer
    ├── DrawingPad
    ├── PreviewRenderer
    ├── MakerUI
    ├── AppShell
    └── ヘルパー関数群
```

---

## 画面システム

`body[data-screen="maker"]` の CSS 属性セレクタで表示画面を切り替え。  
`showScreen(name)` が `body.dataset.screen` を書き換えるだけで、各 `.screen` が `display: flex / none` で切り替わる。

| 画面 | ID | 状態 |
|---|---|---|
| 手帳 | `#screen-home` | Phase 2（プレースホルダー）|
| シールメーカー | `#screen-maker` | ✅ 実装済み |
| コレクション | `#screen-collection` | Phase 3（プレースホルダー）|
| QR 共有 | `#screen-share` | Phase 4（プレースホルダー）|

---

## state オブジェクト

```javascript
const state = {
  // シール設定
  shape:          'circle',   // 'circle' | 'star' | 'heart' | 'square'
  bgColor:        '#ff6b9d',  // 背景色（16色パレットから選択）

  // スタンプ
  stamp:          null,       // 絵文字文字列 or null
  stampX:         150,        // canvas 座標 0-300（デフォルト中央）
  stampY:         150,
  stampPlaceMode: false,      // スタンプ配置モード中か

  // テキスト
  stickerText:    '',
  textX:          150,        // canvas 座標 0-300（デフォルト中央）
  textY:          230,        // デフォルト下部
  textPlaceMode:  false,      // テキスト配置モード中か

  // 描画ツール
  penColor:       '#222222',
  penSize:        10,         // 4 | 10 | 20
  eraserMode:     false,
  eraserSize:     16,

  // アンドゥ
  undoStack:      [],         // ImageData[] 最大20件
  redoStack:      [],
};
```

---

## キャンバス構成（2層オーバーレイ）

```
.canvas-stack（display: grid）
├── #preview-canvas  下層：合成済み最終出力（pointer-events: none）
└── #draw-canvas     上層：ユーザー描画・透明背景（イベント受取）
```

`grid-area: 1 / 1` で同セルに重ね合わせ。  
`#preview-canvas` に `pointer-events: none` を指定し、タッチを `#draw-canvas` に素通しする。  
`#draw-canvas` の背景は透明（canvas デフォルト）なので `#preview-canvas` の内容が自然に透けて見える。

### canvas サイズ

- 内部解像度: 300 × 300 px（固定）
- CSS 表示サイズ: `min(300px, calc(100vw - 56px))`（モバイル対応）

---

## PreviewRenderer

state が変わるたびに `PreviewRenderer.render()` を呼び、毎回フル再描画する。

```
Layer 0  白背景（全面 fillRect）
Layer B  形状アウトライン ボーダー（buildPath + lineWidth=4、内側 2px は fill で隠れる）
Layer 1  shape clip + bgColor fill
Layer 2  shape clip + drawImage(draw-canvas)
Layer 3  shape clip + スタンプ絵文字（state.stampX / stampY）
Layer 4  shape clip + テキスト（state.textX / textY、白文字 + shadowBlur=8）
```

#### 形状パラメータ

- `cx = cy = 150`（300px キャンバスの中央）
- `r = 136`（端まで 14px の余白）

---

## ShapeRenderer

`buildPath(ctx, cx, cy, r, shape)` でパスを構築するのみ（clip しない）。  
ボーダー描画と clip 描画の両方で共用できるよう分離している。

```javascript
ShapeRenderer.buildPath(ctx, cx, cy, r, shape);  // パスのみ
ShapeRenderer.clipToShape(ctx, cx, cy, r, shape); // buildPath + clip
```

| 形状 | 実装 |
|---|---|
| `circle` | `arc()` |
| `square` | `quadraticCurveTo` 8点の角丸矩形（corner radius = `min(r*0.08, 8)`）|
| `heart` | 6セグメント対称ベジェ曲線（C1 連続、バウンディングボックス四辺に接触）|
| `star` | 外径 r・内径 r×0.46 の10頂点ラウンドスター（`quadraticCurveTo` で角を丸める）|

---

## DrawingPad

PointerEvents API でマウス・タッチを統合処理。

| 機能 | 実装 |
|---|---|
| 描画 | `pointerdown → pointermove` で `lineTo` ストローク |
| ドット | `pointerdown` 時に `arc` で 1 点描画 |
| 消しゴム | `source-over` + `strokeStyle='#ffffff'`（透明穴ではなく白上書き）|
| アンドゥ | `pointerdown` 前に `getImageData` でスナップショット保存（上限 20 件）|
| リドゥ | アンドゥ時に現在状態を redoStack へ退避 |
| パームリジェクション | `e.width > 80px` のタッチを無視 |
| 2点タッチ防止 | `activePointers`（Set）で管理、2本目の指が触れると `isDrawing = false` で中断 |
| スタンプ配置 | `stampPlaceMode` 中はタップ座標を `state.stampX/Y` に反映するだけで描画しない |
| テキスト配置 | `textPlaceMode` 中はタップ座標を `state.textX/Y` に反映するだけで描画しない |

#### モード排他制御

- スタンプ/テキスト配置モードはペン・消しゴム選択時に自動解除
- 指を離した瞬間（`pointerup`）に配置確定、自動的に描画モードへ復帰

---

## MakerUI パネル構成

画面を縦スクロールするパネル群で構成。

| パネル | 内容 |
|---|---|
| `panel-shapes` | かたち選択（まる・ほし・はーと・しかく）|
| `panel-colors` | いろ選択（16色グリッド、背景色）|
| `panel-stamps` | スタンプ選択（24絵文字横スクロール）+ 「🎯 いちをきめる」|
| `panel-text` | もじ入力（最大12文字）+ 「🎯 いちをきめる」|
| `panel-draw` | おえかき（ツールバー + 色スウォッチ + canvas-stack）|
| `panel-save` | 「💾 シールにする！」保存ボタン |

#### おえかきツールバー

```
行1: [S•][M•][L•]  ............  [🩹][↩][🗑]
行2: 色スウォッチ 16色
     ─────────────────────
     canvas-stack（preview + draw 重ね）
```

---

## 定数

```javascript
COLOR_PALETTE  // 16色（背景色パレット）
PEN_COLORS     // 16色（描画色パレット、黒を先頭に配置）
STAMP_LIST     // 24絵文字 ['⭐','🌟','💖', ... ,'❄️','🔥']
```

---

## データ永続化（localStorage）

キー: `stickerNotebook_stickers`

```json
[{
  "id":        "sticker_lbz4k2a",
  "createdAt": "2026-05-04T10:00:00.000Z",
  "source":    "self",
  "imageData": "data:image/png;base64,..."
}]
```

- 新しいシールは先頭に追加（`unshift`）
- `QuotaExceededError` 発生時はフレンドリーなオーバーレイを表示（`alert` 不使用）
- `StickerStorage.delete(id)` で個別削除

---

## UI 設計原則（小学生向け）

| 項目 | 実装値・方針 |
|---|---|
| 最小タップターゲット | 48px（Apple HIG 推奨 44px 以上）|
| フォント | Hiragino Maru Gothic Pro / BIZ UDPGothic（丸ゴシック）|
| テキスト | ひらがな優先（「けしごむ」「ぜんぶけす」「いちをきめる」）|
| 破壊操作 | 全消しに「ぜんぶけすよ？いいの？」確認ダイアログ |
| カラーピッカー | `<input type="color">` 不使用、16色スウォッチに統一 |
| ペンサイズ | range スライダー廃止、3段階ボタン（黒丸ドットで太さを視覚化）|
| カラーテーマ | `--c-primary: #ff6b9d`（ピンク）、`--c-bg: #fff9f0`（クリーム）|
| ビューポート | `max-width: 480px`、`min-height: 100dvh`、`user-scalable=no` |

---

## 今後の実装（Phase 2〜4）

| Phase | 機能 | 概要 |
|---|---|---|
| 2 | 手帳ページ | 作ったシールをページに貼り付けて飾れる手帳 |
| 3 | コレクション | 保存済みシールの一覧表示・削除 |
| 4 | QR 共有 | QR コード生成・カメラ読取でシールをやり取り |

QR 関連ライブラリ（qrcodejs / jsQR）は Phase 1 から CDN で読み込み済み。
