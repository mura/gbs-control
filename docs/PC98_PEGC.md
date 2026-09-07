# gbs-control PC-98 PEGC (640x480 @ 60Hz) 技術仕様・実機検証ログ集 (PC98_PEGC.md)

本ドキュメントは、gbs-control における **PC-9821 PEGC（640x480 @ 60Hz / 31.47kHz / VTotal 524〜525本）** 対応の実装内容、H-PLL 位相特性、各出力解像度別の実機検証ログおよび確定パラメータをまとめた記録です。

※ PC-98 24kHz（メイン）実機タイミング・実験ログは **[`PC98_24kHz.md`](PC98_24kHz.md)**、31kHz（400本）実機ログは **[`PC98_31kHz.md`](PC98_31kHz.md)**、GBS-Control / TV5725 レジスタ仕様・コマンド挙動は **[`GBS.md`](GBS.md)** を参照してください。

---

## 1. PEGC 映像信号の基本スペックと DOS (400本) との比較

| 項目 | PEGC (480p モード) | 31kHz DOS モード | 24kHz DOS モード | 備考 |
| :--- | :---: | :---: | :---: | :--- |
| **表示解像度** | **640 x 480** (4:3) | 640 x 400 (16:10) | 640 x 400 (16:10) | IBM VGA 480p 互換 |
| **水平周波数 (fH)** | **31.47 kHz** | 31.47 kHz | 24.83 kHz | |
| **垂直周波数 (fV)** | **59.94 Hz (実測 ~60Hz)** | 70.08 Hz | 56.42 Hz | 一般モニタでネイティブ対応可 |
| **垂直総ライン (VTotal)** | **524〜525 lines** | 448〜449 lines | 439〜440 lines | 400本系とは約 85本の差 |
| **水平総ドット (HTotal)** | **800 dots** | 800 dots | ~848 dots | 水平アクティブ: 640 dots |
| **同期信号** | セパレート (RGBHV) | セパレート (RGBHV) | セパレート (RGBHV) | D-Sub 15pin 2列 |
| **同期極性** | H: 負極性, V: 負極性 | H: 負極性, V: 負極性 | H: 負極性, V: 負極性 | `H-V-` |

---

## 2. PEGC 固有の TV5725 ハードウェア挙動と物理メカニズム

### 2.1 H-PLL ゲイン (`PLLAD_FS`) と垂直同期ロック
- **課題**: PEGC は走査線数が 525本（DOS 400本より 85本多い）かつ 60Hz 駆動のため、400本 DOS と同じ `PLLAD_FS = 0`（Low Gain）のままだと H-PLL のループ帯域幅が不足し、垂直同期が外れて高速縦スクロール（垂直ビート）が発生する。
- **解決策**: PEGC 入力検知時のみ **`PLLAD_FS = 1`（High Gain）** に引き上げることで、垂直同期が瞬時にロックし、安定した静止画が得られる。

### 2.2 `PLLAD_FS` による定常クロック位相遅延（見切れ）
- `PLLAD_FS` を `0`（Low Gain）から `1`（High Gain）に切り替えると、HSync エッジから ADC サンプリングクロック 1 発目が出るまでの**定常位相遅延（物理クロック位相）が約 44 ドット分シフト**する。
- したがって、400本 DOS 用の水平取り込み窓（`IF_HB_ST2 = 0x490`）のままだと、PEGC では左側の映像が約 44 ドット（1〜2文字分）はみ出して見切れてしまう。
- **解決策**: PEGC 時は解像度に合わせて `IF_HB_ST2` を左へシフト（小さい値へ設定）することで、左右見切れゼロと完全センタリングを達成する。

---

## 3. 実機検証ログ (Work Log)

| 日付 | 検証解像度 | 試行内容と観察結果 | 判定・確定値 |
| :--- | :---: | :--- | :--- |
| 2026-09-06 | 1080p | PEGC 表示時に左側半角1文字の見切れが発生。シリアルコマンド `'6'` で水平取り込み位置を追い込み。16インチ 16:9 ディスプレイ実測で左 57mm / 右 61mm からさらに 2mm 右へ移動させ、**左右の余白が各 59mm の完全均等センタリング** に到達（ユーザー目視: 「OK。センターになったと思う。上下左右ともに見切れてるところは見つからない」） | **`IF_HB_ST2 = 0x464`**<br>**`IF_HB_SP2 = 0x068`**<br>**`VSCALE = 480`** |
| 2026-09-06 | 全解像度 | 同期過渡期のノイズ（例: `CsVT: 97`）により、PEGC 画面なのに一瞬で DOS 400p 扱いになり `0x490`（左見切れ設定）で上書きされる誤爆バグを特定。有効ライン範囲の厳密化（`420..470` vs `490..560`）と **5フレーム連続一致デバウンス（安定化フィルタ）** を導入して誤爆を完全根絶 | 動的判定アーキテクチャ確立 |
| 2026-09-06 | 960p | 960p（1280x960 162MHz）にて左側 2文字分の見切れが発生。シリアルコマンド `'6'` を段階的に送信し、`IF_HB_ST2 = 0x470`（1136）にて見切れていた文字がすべて現れることを目視確認（ユーザー目視: 「全部表示された」） | **`IF_HB_ST2 = 0x470`**<br>**`IF_HB_SP2 = 0x074`**<br>**`VSCALE = 562`** |
| 2026-09-06 | 480p | 480p にて左側 1.5文字、下側 1行分の見切れが発生。水平方向はシリアル `'6'` により `IF_HB_ST2 = 0x47C` で左端が全部出たことを確認。垂直方向はシリアル `'*'` で上へ引き上げ、上端境界（`IF_VB_ST: 14, SP: 16`）を特定。下端も液晶ディスプレイの物理端（480ドット全域）まで出し切る完全 1:1 等倍ドットバイドット表示を確立 | **`IF_HB_ST2 = 0x47C`**<br>**`IF_HB_SP2 = 0x080`**<br>**`IF_VB_ST = 14, SP = 16`**<br>**`VSCALE = 1023`** |

---

## 4. PEGC 実機における出力解像度別 確定パラメータ一覧

### 4.1 1080p (1920x1080 @ 60.00Hz)
- **垂直スケーラー**: `VDS_VSCALE = 480`（上下フル 1440x1080 表示）
- **水平スケーラー**: `VDS_HSCALE_BYPS = 1`（バイパス）
- **H-PLL ゲイン**: `PLLAD_FS = 1`（High Gain）
- **垂直ブランキング**: `IF_VB_ST = 6`, `IF_VB_SP = 8`
- **Display Blanking**: `VDS_DIS_HB_ST = 1356`, `VDS_DIS_HB_SP = 348`（1080p ピラーボックス窓）
- **水平キャプチャ位置**: **`IF_HB_ST2 = 0x464`（1124）, `IF_HB_SP2 = 0x068`（104）**
  - 実測左右余白各 59mm の完全均等センタリング・上下左右見切れゼロ。

### 4.2 960p (1280x960 @ 59.91Hz / 162MHz 駆動)
- **垂直スケーラー**: `VDS_VSCALE = 562`（480ライン入力用）
- **水平スケーラー**: `VDS_HSCALE = 550`
- **H-PLL ゲイン**: `PLLAD_FS = 1`（High Gain）
- **垂直ブランキング**: `IF_VB_ST = 6`, `IF_VB_SP = 8`
- **Display Blanking**: `VDS_DIS_HB_ST = 2610`, `VDS_DIS_HB_SP = 626`
- **水平キャプチャ位置**: **`IF_HB_ST2 = 0x470`（1136）, `IF_HB_SP2 = 0x074`（116）**
  - 左端・右端ともに 1 ドットも欠けずに全部表示。

### 4.3 480p (720x480 / 640x480 @ 60.00Hz)
- **垂直スケーラー**: `VDS_VSCALE = 1023`（完全 1:1 バイパス・ドットバイドット）
- **H-PLL ゲイン**: `PLLAD_FS = 1`（High Gain）
- **垂直ブランキング**: **`IF_VB_ST = 14`, `IF_VB_SP = 16`**
- **表示垂直ブランキング**: `VDS_VB_SP = 24`, `VDS_DIS_VB_SP = 31`, `VDS_DIS_VB_ST = 527`（全画面）
- **水平キャプチャ位置**: **`IF_HB_ST2 = 0x47C`（1148）, `IF_HB_SP2 = 0x080`（128）**
  - 左右見切れゼロ、上端 100% 完全表示、下端物理液晶端到達の完全 1:1 等倍表示。

---

## 5. DOS 400本 ⇔ PEGC 480本 動的切り替えアーキテクチャ (`gbs-control.ino`)

ファームウェアは、VTotal（走査線数）の測定値を監視し、以下の条件で自動的にレジスタを切り替える：

```cpp
// doPostPresetLoadSteps() & runSyncWatcher()
if (sourceLines >= 490 && sourceLines <= 560) {
    // PEGC 480-line mode (typical vt: 525)
    if (rto->presetID == 0x05 || rto->presetID == 0x15) { // 1080p
        GBS::VDS_VSCALE::write(480);
        GBS::IF_HB_ST2::write(0x464);
        GBS::IF_HB_SP2::write(0x068);
        GBS::IF_VB_ST::write(6);
        GBS::IF_VB_SP::write(8);
    } else if (rto->presetID == 0x01 || rto->presetID == 0x11) { // 960p
        GBS::VDS_VSCALE::write(562);
        GBS::IF_HB_ST2::write(0x470);
        GBS::IF_HB_SP2::write(0x074);
        GBS::IF_VB_ST::write(6);
        GBS::IF_VB_SP::write(8);
    } else if (rto->presetID == 0x04 || rto->presetID == 0x14) { // 480p
        GBS::VDS_VB_SP::write(24);
        GBS::VDS_DIS_VB_SP::write(31);
        GBS::VDS_DIS_VB_ST::write(527);
        GBS::IF_HB_ST2::write(0x47C);
        GBS::IF_HB_SP2::write(0x080);
        GBS::IF_VB_ST::write(14);
        GBS::IF_VB_SP::write(16);
    }
    GBS::PLLAD_FS::write(1); // PEGC High Gain
    latchPLLAD();
} else if (sourceLines >= 420 && sourceLines <= 470) {
    // DOS 400-line mode (24k vt: 439 / 31k vt: 449)
    if (rto->presetID == 0x05 || rto->presetID == 0x15) { // 1080p
        GBS::VDS_VSCALE::write(400);
    } else if (rto->presetID == 0x01 || rto->presetID == 0x11) { // 960p
        GBS::VDS_VSCALE::write(468);
    } else if (rto->presetID == 0x04 || rto->presetID == 0x14) { // 480p
        GBS::VDS_VB_SP::write(64);
        GBS::VDS_DIS_VB_SP::write(72);
        GBS::VDS_DIS_VB_ST::write(488);
    }
    GBS::IF_HB_ST2::write(0x490);
    GBS::IF_HB_SP2::write(0x094);
    GBS::IF_VB_ST::write(6);
    GBS::IF_VB_SP::write(8);
    GBS::PLLAD_FS::write(0); // DOS Low Gain
    latchPLLAD();
}
```

- **5フレーム連続一致デバウンス**:
  `abs(sourceLines - candidatePc98Lines) <= 5` を 5回連続で満たした場合のみ切り替えを実行することで、解像度切り替え過渡期やノイズによる誤動作を 100% 防止する。
- **解像度（プリセット）切り替え時の自動再適用 (`rto->presetID != lastPresetID`)**:
  PEGC 表示中に出力解像度（1080p ⇔ 960p ⇔ 480p）を切り替えた際にも、同期確立後に新プリセット用の PEGC 最適値が即座に自動再適用され、見切れやズレが一切発生しない設計を確立。

---

## 6. 同一解像度再選択時の見切れ防止と知見 (2026-09 実機検証)

### 6.1 現象と真因
- **現象**:
  PEGC（640x480）表示中に WebUI 等から同一解像度（1080p など）を再ロードすると、左側が 1 文字分見切れる現象が発生していた。
- **真因**:
  プリセットロード時、`doPostPresetLoadSteps()` の最後で呼び出される `applyPc98Timings()` 内に、走査線数を考慮せず無条件で `IF_HB_ST2::write(0x490)`（DOS 400本用）を書き込む処理が残っていた。
  同一解像度の再選択では垂直走査線数（`CsVT`）に変化がないため、同期監視ループ（`syncWatcher`）が再トリガーされず、上書き破壊された `0x490` のまま残っていた。
  （差分: `0x490 - 0x464 = 0x2C = 44ドット` 右寄りのため、左側が約 1 文字見切れていた）

### 6.2 対策と実装
1. **`applyPc98Timings()` の走査線数連動化**:
   `applyPc98Timings()` 内でも現在走査線数を読み取り、PEGC（490〜560）であれば `0x464`（1080p）/ `0x470`（960p）/ `0x47C`（480p）および `PLLAD_FS = 1` を確実に適用するように条件分岐。
2. **TV5725 VTOTAL 2倍レートカウントの正規化**:
   TV5725 の同期プロセッサは過渡状態や動作モードによって VTOTAL を 2倍（800〜1150）で返す場合があるため、走査線判定を行うすべての箇所で以下の正規化を適用：
   ```cpp
   if (currentVt >= 800 && currentVt <= 1150) {
       currentVt /= 2;
   }
   ```
これにより、コールドスタート、同一解像度再選択、および解像度相互切り替えのいずれの遷移パスにおいても、常に左右見切れゼロのジャストフィット表示が 100% 保証される。

