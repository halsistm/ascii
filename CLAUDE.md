# ASCII DOOM

ASCII テキストベースの一人称シューティングゲーム。Doom のような雰囲気を Canvas + ASCII 文字でミニマルに再現したシングルファイル HTML5 ゲーム。

## 概要

- **形式**: シングルファイル HTML5（`.html`）
- **言語**: JavaScript（ES6）
- **フォント**: モノスペース（Courier New など）
- **解像度**: 40 列 × 12 行（ASCII グリッド）
- **フレームレート**: 30 FPS（処理とビジュアル安定性のバランス）
- **対応**: デスクトップ、タブレット、iPhone（フルスクリーン対応）

---

## ゲームシステム

### マップ

```
####################
#..........E.......#
#....E.............#
#..................#
#......E........E..#
#..................#
####################
```

- **サイズ**: 20 × 7 タイル
- **`#`**: 壁（衝突判定あり）
- **`.`**: 床（移動可能）
- **`E`**: 敵スポーン地点

マップは固定。衝突判定は円形（半径 0.22）で、**壁すり** を許可（X/Y を分けて判定）。

### プレイヤー

```javascript
const player = {
  x: 3.5, y: 3.5,          // マップ座標（小数対応）
  angle: 0,                // 向き（ラジアン、0 = +x）
  hp: 100, maxHp: 100,
  attackPower: 12,
  attackCooldown: 0,       // 攻撃硬直（秒）
  hurtFlash: 0,            // ダメージ演出タイマー
  attackFlash: 0,          // 攻撃演出タイマー
};
```

**移動**: WASD / 矢印キー
- **前進**: `w` / ↑（移動速度 2.6）
- **後退**: `s` / ↓（移動速度 1.56 = 0.6 × 2.6）
- **左回転**: `a` / ← （旋回速度 π × 1.05 rad/s）
- **右回転**: `d` / →

**攻撃**: Space / Enter / `j` / `f`
- 正面コーン内（±18°）、距離 1.7 以内で最近敵を対象
- ダメージ: `12 ± 2` (10〜14)
- クールタイム: 0.38 秒

**HP**: 敵の攻撃でダメージ → **HP ≤ 0** で GAME OVER

### 敵システム

#### 敵タイプ

| 名前 | 文字 | HP | 速度 | ダメージ | 攻撃範囲 | 攻撃間隔 |
|------|------|----|----|---------|--------|--------|
| スライム | ス | 15 | 0.55 | 1〜2 | 0.95 | 1.4s |
| ゴーレム | ゴ | 38 | 0.30 | 2〜4 | 1.05 | 1.8s |
| キメラ | キ | 22 | 0.75 | 1〜3 | 1.00 | 1.3s |
| メタル | メ | 8 | 1.10 | 1〜2 | 0.90 | 1.0s |
| ゴースト | 幽 | 18 | 0.95 | 1〜3 | 0.95 | 1.1s |

#### AI 行動

1. **接近**: プレイヤーとの距離が攻撃範囲外なら移動
2. **攻撃**: 攻撃範囲内 & クールタイムリセット済み → 攻撃実行
3. **ダメージ**: ダメージ状態で視覚フラッシュ（0.18s）

#### スポーン管理

```javascript
const desired = 3 + Math.min(4, Math.floor(kills / 4));
```

- 初期: 3 体
- 敵を 4 体倒すたびに +1 体（最大 7 体）
- スポーン間隔: 1.5 ~ 3.0 秒（ランダム）
- スポーン位置: マップの `.` ランダムタイル

---

## 3D ビューシステム

### レイキャスト

プレイヤー視点から **40 列** に対して水平レイキャストを実行。距離と交差タイルから**壁の高さ**を決定。

```javascript
// 右側（列 20〜39）
const rayAngle = player.angle + (col - centerCol) * fovSlice;
const dx = Math.cos(rayAngle), dy = Math.sin(rayAngle);
// ... 壁衝突を検出、距離を計算
```

### 壁レンダリング

```javascript
const dist = ... // レイキャストで取得
const brightness = Math.max(0.1, 1.2 - dist * 0.25);  // 距離減衰
const wallHeight = Math.round(6 / (dist + 0.5));      // 射影高さ
```

#### テクスチャ（距離別）

| 距離 | 文字列 | 説明 |
|------|--------|------|
| ≤ 0.5 | `█` | 至近（ほぼ壁が画面を占める） |
| ≤ 1.0 | `█` または `▓` | 視野内 |
| ≤ 2.0 | `▓` または `▒` | 中距離 |
| > 2.0 | `░` | 遠景 |

#### ダメージフラッシュ

プレイヤーがダメージを受けると、画面全体が赤くしたくなるが、ASCII 環境では代わりに：

```javascript
if (player.hurtFlash > 0) {
  ctx.fillStyle = 'rgba(255, 50, 50, 0.3)';
  ctx.fillRect(...);
}
```

色付きセルでダメージ表現。

### 敵の距離表示

敵と視点を結ぶレイとプレイヤー視向を比較し、画面内にいるかを判定。

```javascript
const dx = e.x - player.x, dy = e.y - player.y;
const d = Math.sqrt(dx*dx + dy*dy);
const ang = Math.atan2(dy, dx) - player.angle;
// ... 正規化して視野内チェック
```

視野内なら、距離に応じた敵文字を表示（例: `ス` / `★`）。

---

## HUD（ヘッズアップディスプレイ）

### レイアウト

```
[    3D View (9 行)    ]
┌─────────────────────┐
│                     │  行 9: 区切り線
├─────────────────────┤
│ HP: ████░░░░░░░░░░  │  行 10
├─────────────────────┤
│ 敵  スライム×2      │  行 11
└─────────────────────┘
```

### 要素

- **HP バー**: 現在値 / 最大値（ブロック表示）
- **敵リスト**: 現在いる敵の種類と数
- **ステータスメッセージ**: 攻撃結果、ダメージ、敵のセリフ

```javascript
statusMsg = 'スライムに 12 ダメージ';
statusMsgTimer = 0.9;  // 0.9 秒表示
```

---

## マップビュー（M キー）

`m` / `M` キーで鳥瞰図に切り替え。

```javascript
for (let iy = 0; iy < MAP_H; iy++){
  for (let ix = 0; ix < MAP_W; ix++){
    const char = MAP_RAW[iy][ix];
    if (char === '#') ctx.fillStyle = '#666';      // 壁
    else if (...) ctx.fillStyle = '#999';          // 床
    // プレイヤーを◎、敵を敵文字で表示
  }
}
```

---

## 入力システム

### キーボード

```javascript
case 'w': case 'W':      setKey('up', true);
case 'a': case 'A':      setKey('left', true);
case 's': case 'S':      setKey('down', true);
case 'd': case 'D':      setKey('right', true);
case ' ', 'Enter', 'j':  setKey('attack', true);
case 'm': case 'M':      toggleMap();
```

### タッチ / ポインタ

D-Pad（方向）と ATTACK（大円形ボタン）をタッチで操作。

```javascript
const press   = (ev) => { setKey(key, true);  el.classList.add('pressed'); };
const release = (ev) => { setKey(key, false); el.classList.remove('pressed'); };
el.addEventListener('pointerdown', press);
el.addEventListener('pointerup', release);
el.addEventListener('pointerleave', release);
```

**CSS クラス**: `pressed` で押下状態のビジュアル反応。

### MAP ボタン特殊処理

D-Pad の中央：「押している間」ではなく「押す」で MAP ビューをトグル。

```javascript
const mapPress = (ev) => {
  toggleMap();
  mapBtn.classList.add('pressed');
};
mapBtn.addEventListener('pointerdown', mapPress);
```

---

## レンダリング

### グリッド初期化

```javascript
function clearGrid() {
  for (let r = 0; r < ROWS; r++){
    for (let c = 0; c < COLS; c++){
      grid[r][c] = { char: ' ', color: '#000', bgcolor: '#000' };
    }
  }
}
```

### 描画パイプライン

1. **clearGrid()**: グリッドをリセット
2. **buildView()** または **buildMapView()**: グリッドに文字・色を書き込み
3. **buildHud()**: HUD 行を追加
4. **drawFrame()**: キャンバスに レンダリング

### drawFrame() の詳細

```javascript
function drawFrame(){
  for (let r = 0; r < ROWS; r++){
    for (let c = 0; c < COLS; c++){
      const cell = grid[r][c];
      ctx.fillStyle = cell.bgcolor;
      ctx.fillRect(c * CHAR_W, r * CHAR_H, CHAR_W, CHAR_H);
      ctx.fillStyle = cell.color;
      ctx.fillText(cell.char, (c + 0.5) * CHAR_W, r * CHAR_H);
    }
  }
}
```

### DPR (Device Pixel Ratio)

Retina 対応：

```javascript
DPR = Math.min(window.devicePixelRatio || 1, 3);
canvas.width = Math.round(cssW * DPR);
ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
```

---

## ゲームループ（30 FPS）

```javascript
const FRAME_DT = 1 / 30;  // ≒ 33.3ms
let acc = 0;

function loop(now) {
  requestAnimationFrame(loop);
  const dt = (now - lastTime) / 1000;
  lastTime = now;
  acc += dt;
  if (acc < FRAME_DT) return;  // フレーム間引き
  
  update(acc);
  clearGrid();
  buildView();
  buildHud();
  drawFrame();
  acc = 0;
}
```

**なぜ 30 FPS？**
- iOS Safari でモバイル端末のバッテリー消費を抑制
- ASCIIレンダリングで 60 FPS の性能利得が少ない
- 入力遅延とビジュアル安定性の良バランス

---

## スタイリング

### CSS の特徴

**リセット**:
```css
* { box-sizing: border-box; margin: 0; padding: 0; }
```

**セーフエリア対応**（ノッチ付きスマートフォン）:
```css
padding-top: env(safe-area-inset-top);
padding-bottom: env(safe-area-inset-bottom);
```

**CRT スキャンライン効果**:
```css
.screen-wrap::after {
  background:
    radial-gradient(ellipse at center, transparent 55%, rgba(0,0,0,0.55) 100%),
    repeating-linear-gradient(to bottom, transparent 0 2px, rgba(0,0,0,0.18) 2px 3px);
  mix-blend-mode: multiply;
}
```

**ボタン**:
- **通常**: `background: #0c0c0a; color: #d8d8cc;`
- **押下**: `background: #e6e6dd; color: #000; transform: translateY(1px);`
- トランジション: `transform 60ms ease`

---

## 開発メモ

### 主要な設計判断

1. **シングルファイル**: CDN なし、外部アセットなし → 即座に実行可能
2. **30 FPS**: CPU 負荷と視覚品質のバランス
3. **レイキャスト**: 完全な 3D レンダリングより軽量
4. **敵スポーン管理**: 動的難度調整（キル数に応じた敵数増加）
5. **壁すり**: X/Y 衝突判定を分離 → マップ幾何学の自然な移動感

### デバッグポイント

- **敵が壁を超える**: `isWall()` の判定が甘い可能性 → 衝突半径を増やす
- **敵が動かない**: `tryMove()` のロジックを確認
- **攻撃が当たらない**: コーン角度（`cone = 0.32`）と距離（`reach = 1.7`）を調整
- **フレームレート不安定**: ゲームループの `dt` キャップを確認（`Math.min(0.1, dt)`）

### パフォーマンス最適化

✅ **済み**:
- グリッドキャッシュ（毎フレーム再作成は避ける）
- DPR スケーリング（キャンバス解像度を適切に）
- `canvas` の `image-rendering: pixelated`

🔧 **可能な改善**:
- `clearGrid()` → `grid = new Array(ROWS * COLS).fill(...)`（オブジェクト池の活用）
- 敵リストの線形探索 → 空間分割（グリッドやクワッドツリー）
- レイキャスト最適化：DDA アルゴリズムの導入

---

## ゲームプレイのコツ

1. **最初は敵が少ない**: 3 体から開始。敵を倒してペースを上げる
2. **攻撃は前方精密**: 正面コーン内でないと外れる → 向きを調整
3. **後退は低速**: 前進より遅いため、危ないときは前に出て背中を向く
4. **ゴーレムは要注意**: HP 38・高ダメージ（2〜4）→ 複数に囲まれるな
5. **MAP で敵位置を把握**: M キーで鳥瞰図確認 → 逃げ道を見つける

---

## ライセンス

シングルファイル・シンプルな設計のため、自由に改造・派生可。

---

## 関連プロジェクト

- **monsss** / 放課後化学反応: Monster Strike ライク・マーブルシューティング
- **∞Gallery**: Three.js 3D ルーム + 生物図鑑システム
- **UFO Abductor**: Three.js 異世界探索ゲーム

