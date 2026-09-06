# gbs-control (PC98パッチ)

本リポジトリは、オープンソースのアップスケーラー [gbs-control](https://github.com/ramapcsx2/gbs-control) に、**NEC PC-9801 / PC-9821 シリーズの 24kHz / 31kHz 映像信号への対応パッチ** を適用したフォーク版ファームウェアです。

PC-98 特有の周波数（24.83kHz / 56.4Hz、31.5kHz / 70Hz）やセパレート同期（RGBHV）を処理し、現代の液晶モニタ・テレビ（HDMI / DVI / VGA @ 60Hz）に低遅延で安定表示させることを目的としています。

---

## 🌟 PC-98 対応の機能・特徴

- **24kHz / 31kHz 映像信号のシームレス対応**:
  - PC-9801 の標準解像度である **24kHz (56.4Hz)**、および PC-9821 の高周波数モードである **31kHz (70.1Hz)** の両方の映像信号に対応。
  - SDRAM フレームバッファにより、どちらの信号も現代のディスプレイに適した標準的な **60.00Hz** へ低遅延でフレームレート変換して出力します。
- **専用高画質アップスケールプリセット**:
  - **1280x960 (4:3)**: 水平 2.0倍（640→1280）、垂直 2.4倍（400→960、4:3 比率拡大）で出力します。DOS やゲーム画面に最適です。
  - **1920x1080 (16:9 FHD ピラーボックス)**: 垂直 2.7倍（400→1080、垂直全画面フィット）、左右に黒帯を付加して出力します。
  - **480p (720x480 @ 60Hz)**: 垂直 400ライン等倍（1.0倍）で 480p 枠内へ変換して出力します。
- **操作インターフェース**:
  - **WebUI（ブラウザ）**: スマホやPCのブラウザから「PC-98 Mode」を ON/OFF 可能です。
  - **OLED メニュー**: 本体に取り付けた OLED ディスプレイとロータリーエンコーダから「PC-98: Off / On」を手元で切り替え可能です。
  - **シリアルコマンド**: シリアルコンソールから `'9'` を送信することで PC-98 モードの ON/OFF が可能です。

---

## 🖥️ 動作確認環境

- **検証実機**: NEC PC-9821 Ap2
- **接続構成**: PC-9821 Ap2 本体のディスプレイ端子（D-Sub 15pin 2列）を 3列変換コネクタ経由で GBS の VGA 入力端子へ接続し、HDMI 出力にて動作確認を実施。


---

## 📚 詳細技術ドキュメント

開発・検証で得られた詳細なタイミング仕様、レジスタ設定値、実験ログは `docs/` に記録されています：
- **[`docs/PC98_24kHz.md`](docs/PC98_24kHz.md)**: PC-98 24kHz の詳細タイミング、1080p/960p 専用プリセット設計値、実機検証ログ
- **[`docs/PC98_31kHz.md`](docs/PC98_31kHz.md)**: PC-98 31kHz (70Hz) の検証記録
- **[`docs/GBS.md`](docs/GBS.md)**: GBS-Control / TV5725 のレジスタ仕様、パイプライン挙動、ハードウェア制約知見集

---

## (Original) gbs-control

Documentation: https://ramapcsx2.github.io/gbs-control/

Gbscontrol is an alternative firmware for Tvia Trueview5725 based upscalers / video converter boards.  
Its growing list of features includes:   
- very low lag
- sharp and defined upscaling, comparing well to other -expensive- units
- no synchronization loss switching 240p/480i (output runs independent from input, sync to display never drops)
- on demand motion adaptive deinterlacer that engages automatically and only when needed
- works with almost anything: 8 bit consoles, 16/32 bit consoles, 2000s consoles, home computers, etc
- little compromise, eventhough the hardware is very affordable (less than $30 typically)
- lots of useful features and image enhancements
- optional control interface via web browser, utilizing the ESP8266 WiFi capabilities
- good color reproduction with auto gain and auto offset for the tripple 8 bit @ 160MHz ADC
- optional bypass capability to, for example, transcode Component to RGB/HV in high quality
 
Supported standards are NTSC / PAL, the EDTV and HD formats, as well as VGA from 192p to 1600x1200 (earliest DOS, home computers, PC).
Sources can be connected via RGB/HV (VGA), RGBS (game consoles, SCART) or Component Video (YUV).
Various variations are supported, such as the PlayStation 2's VGA modes that run over Component cables.

Gbscontrol is a continuation of previous work by dooklink, mybook4, Ian Stedman and others.  

Bob from RetroRGB did an overview video on the project. This is a highly recommended watch!   
https://www.youtube.com/watch?v=fmfR0XI5czI

Development threads:  
https://shmups.system11.org/viewtopic.php?f=6&t=52172   
https://circuit-board.de/forum/index.php/Thread/15601-GBS-8220-Custom-Firmware-in-Arbeit/   
   
