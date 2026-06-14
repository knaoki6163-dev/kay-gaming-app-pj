# 駅の出口探索ゲーム 3D — 開発メモ (DEVNOTES)

対象ファイル: `station-escape-3d.html`（1ファイル完結 / 約1,360行）
描画: Three.js r128（CDN: cdnjs）。ビルド不要、ブラウザで開けば動く。

---

## 1. 全体像 / ゲーム内容
- 一人称3D。**地下2階(B2) → 地下1階(B1) → 地上**の3フロア構成の駅から脱出するゲーム。
- 目標: 指定された番号（毎回ランダム）の**地上の出口**に到達でクリア。**赤い敵**に捕まるとゲームオーバー。
- B2 は**5面の独立した島式プラットホーム**（電車・柱・売店・看板・ゴミ箱、両側に線路）。各ホームの**両端に階段(計10本)**があり地下1階へ。
- B1・地上は**フロアごとに別レイアウトの迷路**。地下1階→地上の階段は4本。
- プレイヤーは5ホームのどれかにランダム開始。各ホームに敵1体（後述hunt）。

---

## 2. 座標系・グリッド
- グリッド `grids[f]`（f=1:B2, 2:B1, 3:地上）。各 `C.ROWS × C.COLS`（=49×27）。`1`=壁, `0`=床。
- ワールド変換: タイル(x,y) → world (x:列→X, y:行→Z)。`tileToWorld(tx,ty) = ((tx+0.5)*T, (ty+0.5)*T)`、T=`C.TILE`(=5)。
- フロア高さ `FH=C.WALL_H`(=4.5)。フロアfの床=`(f-1)*FH`。地上だけ天井を `C.GROUND_CEIL_EXTRA` 分高くする。

## 3. 主要CONFIG（ファイル冒頭）
- `SEED`(=1): 迷路固定シード。`COLS/ROWS`、`NUM_FLOORS`(=3)、`TILE`、`WALL_H`。
- 敵: `CHASER_FLOORS`(配置階の配列), `CHASER_SPEED/WANDER/SIGHT`, `ALERT_SIGHT_MULT`, `CHASER_MEMORY`, `B2_CHASER_MEMORY`, `CATCH_DIST`。
- 通行人: `PEDESTRIAN_COUNT`(基準/3), 各階倍率 `B2_PED_MULT / B1_PED_MULT / GROUND_PED_MULT`, 速度・回避・急ぎ足系。
- 階段: `STAIR_COUNT`(=4, **B1→地上のみ**), `STAIR_LEN`, `START_CLEAR`(現状未使用寄り)。
- B2レイアウト定数: `LANES`(ホームのx範囲), `TRACKS`(線路のx範囲), `PLAT_TOP/PLAT_BOT`, `TOP_Y0/BOT_Y0`(階段y), `OBJ_ROWS/objType`(中央オブジェクト)。

## 4. 生成パイプライン（`startGame`内）
順序: `RNG=mulberry32(SEED)` → `generateAllFloors()` → `findStairs()` → `cleanupDeadEnds()` → 開始ホーム決定 → `buildWorld()`。

- **RNGは生成専用**（`mulberry32`）。通行人/敵の湧き・揺らぎは `Math.random`（マップは固定、湧きは毎回変化）。
- `generateAllFloors`: B2は `generatePlatformFloor()`(決定的・RNG不使用)、B1/地上は `generateMazeGrid()`（再帰バックトラッカ＋ループ穴開け＋ランダム広場＋`braidFill`で行き止まり除去）。
- `findStairs`:
  - **gap1(B2↔B1)**: 各ホーム両端に1マス幅・3段の `makePlatStair(x0,y0,up)` を10本。`up=+1`(下端)/`-1`(上端)で昇る向きが逆。`carvePlatStair`で両グリッドへ掘り込み（run=床、entry/exit/beyond・rails・接続トンネル）。
  - **gap2(B1↔地上)**: `makeStair`(1マス幅) をRNGで `STAIR_COUNT` 本配置（`carveStairInto`）。
- `cleanupDeadEnds`(B1・地上のみ): `braidConnect`(行き止まりをループ化)→`fillDead`(残りを埋める)→`connectComps`(分断成分をBFSで掘って接続。手すりは保護せず通す＝連結優先)→再`fillDead`。
- **重要**: 生成の正しさ（全フロア連結・各ホームから地上の出口へ到達可）は Node の使い捨てシミュレーションで総当たり検証して `SEED` を選定している。生成ロジックを変えたら **同じロジックでシード再検証**が必要（過去の `/tmp/simXX.js` 相当）。

## 5. 階段の高さ（`stairInfo`）
- `stairInfo(x,z,level)`: 乗っている階段を判定し `{t, y, lvl}` を返す。`level∈{lvl, lvl+1}` のみ対象。
- `t=(along-a0)/(a1-a0)`、**`up===-1` のとき `t=1-t`**（上端階段の向き反転）。高さ `y=(lvl-1)*FH + t*FH`。
- プレイヤー/敵の `level` は `t>=0.5` で `lvl+1`、未満で `lvl`。これでフロア遷移。

## 6. 当たり判定・移動
- `canStand(x,z,r,f)`: 円の四隅が `grids[f]` 床か。`gFloor/gWall` がフロア別。
- B2の中央の柱/売店等は**壁ではなく** `platObstacles`(円)。`obstacleHit(x,z,extra)` で判定。プレイヤー=`playerCanStand`、敵=`entCanStand`、歩行者=移動時にチェック。売店だけ大きい半径。
- `hasLOS(...,f)`: そのフロアの壁でのレイ判定（視線）。

## 7. 敵AI（`updateChasers`）
- 状態: 通常(`mem`方式: 視認で `mem=CHASER_MEMORY`、0まで追跡→徘徊)。
- **alerted**: 一度でも視認したら恒久ON。`sight=CHASER_SIGHT*ALERT_SIGHT_MULT`(探知UP)。徘徊時はランダムでなく**最終目撃地点付近へ寄る**。
- **hunt(地下2階限定)**: B2で視認したら永続追跡。プレイヤーがB2なら本人を、地上へ逃げたら最寄り階段へ向かい**上の階へ追ってくる**。`level>=2`でhunt解除＝通常化。
- **showAlert**: 画面赤枠＝「現在視認中」のみ（hunt中の見失い追跡はサイレント）。
- `moveToward(ent,tx,tz,...)`: 軸分離移動＋詰まり時の回り込み。`nearestStairForMove`で昇降方向の最寄り階段へ。
- 捕獲: `checkResult` で同フロア＆`CATCH_DIST`以内。

## 8. 通行人（大量・InstancedMesh）
- `pedestrians[]`に位置/階/服色等。描画は **InstancedMesh**: 脚`pedLegsInst`・頭`pedHeadInst`(全員共有), 胴は**服色ごと**`pedTorsoInsts[ci]`（r128は`setColorAt`非対応のため色別メッシュ）。`frustumCulled=false`。
- `updatePedestrians`: ランダム徘徊＋プレイヤー回避(`PED_AVOID_R`)＋接触時の即退避(`PED_DODGE_SPEED`)＋一時的な急ぎ足(`PED_RUSH_*`)。毎フレーム各パートの行列を `setMatrixAt`。
- 列車の**乗降客** `boarders` は別管理（少数なのでGroupのまま、ドア前を往復）。

## 9. 描画(`buildWorld`)
- B2: `buildPlatform()`（中央オブジェクトを小物として描画＋当たり円、線路に**窓/ドア付き電車**(`makeTrainSideTex`)＋開いたドアの乗降客、境界壁）。
- 床スラブは**`placeExits`の後**に生成（幅広出口で掘った床にも必ずスラブを付けるため。※過去バグ修正点）。
- 天井: 各下階は `buildFloorCeilings(f)`（B2は全面・黒系で地下1階を透けさせない）、地上は高い天井プレーン。
- 階段: 段々ボックス＋斜め手すり（`up`で向き反転）。
- 出口(`placeExits`): 外周の床候補→間隔をあけて選択→**外周時計回り順に番号1..6**。`3/5番`は**幅5倍の改札風**(壁を5マス開口＋手前を広場化＋幅広＋ゲート柱)。外は**上り階段ではなくフラットな地上広場**(`buildOutsidePlaza`)＋明るい空(霧無効)。

## 10. UI / HUD
- 目標表示(大きい黄色)、TIME、現在地、画面周囲の赤い警告(`#alert`、showAlert連動で点滅)。
- ミニマップ: 現在フロアのみ描画。壁/床/階段(上り赤/下り青の矢印)/出口(正解=金)/敵/自機。**M キーまたはクリックで開閉**(`setMapCollapsed`)。
- 操作: PC=クリックでポインタロック+WASD+マウス、スマホ=ドラッグ視点+左下パッド。

## 11. 既知の注意点 / TODO候補
- 通行人が多数（合計数千）。**描画はインスタンスで軽い**が、移動・回避・`obstacleHit`・プレイヤー×歩行者判定は**人数ぶん毎フレーム**走るので重くなりうる。→ 空間グリッド/格子ハッシュで近傍だけ判定すれば大幅軽量化可能。
- 生成変更時は**必ずシード再検証**（連結・到達性）。
- `whiteMat` 等、未使用化した素材が一部残存（無害）。
- 段差階段は視覚のみ段、プレイヤー高さは滑らかなランプ（足元の段差は一人称で見えない前提）。

## 12. バージョン
- 画面/コードの `APP_VERSION` で確認（現行 Ver.47 系）。raw.githack のブランチURLはキャッシュされるため、確実に最新を見たい時は**コミットハッシュ固定URL**を使う。
