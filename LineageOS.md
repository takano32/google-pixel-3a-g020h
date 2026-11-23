# LineageOS 22.2 インストール記録

Google Pixel 3a（G020H / sargo）に LineageOS 22.2 と MindTheGapps を導入した際の作業記録です。

## 結果

| 項目 | 内容 |
| --- | --- |
| 端末 | Google Pixel 3a G020H（日本向け / codename: sargo） |
| 導入前 OS | Android 12 `SP2A.220505.008`（2022-05-05、最終 Android 12） |
| 導入後 OS | LineageOS 22.2（Android 15） |
| ビルド | `lineage-22.2-20261121-nightly-sargo-signed`（2026-11-21 / セキュリティパッチ 2026-11-01） |
| Google Apps | MindTheGapps 15.0.0 arm64（2026-09-15） |
| ブートローダー | unlocked |
| アクティブスロット | a |
| 実施日 | 2026-11-23 |

## 使用ファイル

ダウンロード先: `~/Downloads/lineage-sargo/`

| ファイル | サイズ | SHA256 |
| --- | --- | --- |
| `lineage-22.2-20261121-nightly-sargo-signed.zip` | 約 1.1 GB | `147b00d1d97251dad31308fa91d7f9e010c1d5036f0f3972dffcd844e4859546` |
| `boot.img`（Lineage Recovery） | 64 MB | `888655b714e099556f3098644d81113dd9c990763bfd208c34a9c23f6df1b74f` |
| `MindTheGapps-15.0.0-arm64-20260915_032013.zip` | 約 484 MB | `e979601ef70b03d9214ea24ca7a103cb408ab102718fff38ec170a6ca03e263d` |

取得元:

- LineageOS 本体 / recovery: https://download.lineageos.org/devices/sargo
- MindTheGapps: https://github.com/MindTheGapps/15.0.0-arm64/releases/latest

## 作業環境

- Arch Linux ARM の Distrobox コンテナ（`arch`）上で adb / fastboot を実行
- ホスト: Debian GNU/Linux 13 (trixie)
- adb / fastboot: `37.0.0p1-android-tools`
- ホスト側 udev ルール `/etc/udev/rules.d/99-google-adb.rules`:

```udev
SUBSYSTEM=="usb", ATTR{idVendor}=="18d1", OWNER="takano32", MODE="0600"
```

Distrobox の user namespace では USB デバイスノードの所有者が `nobody` になり、
`adb devices` が `no permissions` になるため、ホスト側で所有者を明示した。
`udevadm control --reload-rules` はコンテナ内では varlink エラーになるので、必ずホストで実行する。

## 手順

### 1. 事前確認

```bash
adb devices
adb shell getprop ro.product.device          # sargo
adb shell getprop ro.build.version.release    # 12
adb shell getprop ro.build.id                 # SP2A.220505.008
adb shell getprop ro.boot.flash.locked        # 1（ロック状態）
```

- LineageOS 22.2 が必要とする Android 12 ファームウェア（最終ビルド）であることを確認
- データをバックアップし、Google アカウントを端末から削除（FRP 対策）
- 設定 → 開発者向けオプション → OEM ロック解除を有効化
- USB デバッグを有効化し、USB 接続で adb が見えることを確認

### 2. ブートローダー解除

```bash
adb -d reboot bootloader
fastboot devices
fastboot flashing unlock
```

端末画面で Volume キーを使って `Unlock` を選択し、Power で確定する。
この時点で全データが消去される。

```bash
fastboot getvar unlocked    # unlocked: yes
```

### 3. Lineage Recovery の書き込み

```bash
fastboot flash boot boot.img
```

補足: 初回は `boot_a` のみに書き込んだ状態で通常再起動したところ、ストック Android が起動した
（通常起動では Recovery に入らないため）。その後、両スロットに書き込み直した。

```bash
fastboot --slot all flash boot boot.img
```

### 4. Recovery の起動

端末を一度電源オフにし、**Volume Down + Power** でブートローダーメニューを開く。
メニューから **Recovery Mode** を選択する（`Start` は選ばない）。

LineageOS Recovery が起動したことを確認する。

### 5. データ初期化

Recovery で:

1. **Factory reset**
2. **Format data / factory reset**

以下のエラーが出るが、Pixel の Titan M 関連の既知の無害なエラー。
最後に `Data wipe complete.` が出れば成功。

```
ERROR: libnos_datagram: can't send GSA mbox command: Invalid argument
```

### 6. LineageOS 本体の sideload

Recovery で **Apply update** → **Apply from ADB** を選択し、ホストから実行:

```bash
adb -d sideload ~/Downloads/lineage-sargo/lineage-22.2-20261121-nightly-sargo-signed.zip
```

47% 付近で以下のように終了するが、これは正常（インストールは成功している）:

```
adb: failed to read command: Success
```

### 7. MindTheGapps の sideload

Recovery が「reboot to recovery to install add-ons」と聞いてきたら **Yes** を選択。
再起動した Recovery で再び **Apply update** → **Apply from ADB** を選び、実行:

```bash
adb -d sideload ~/Downloads/lineage-sargo/MindTheGapps-15.0.0-arm64-20260915_032013.zip
```

署名検証で失敗の表示が出たら **Yes** を選択する（GApps は LineageOS の鍵で署名されていないため）。
最後に `Total xfer: 1.00x` を確認。

### 8. 初回起動

Recovery で **Reboot system now** を選択。初回起動は最大 15 分程度かかる。

## トラブルと対処

| 症状 | 原因 / 対処 |
| --- | --- |
| `adb devices` が `no permissions` | Distrobox の user namespace で USB ノードが `nobody` になる。ホストに udev ルールを追加 |
| `udevadm` が varlink エラー | コンテナ内からはホストの udev を操作不可。ホスト側で実行する |
| `fastboot devices` が表示されない | USB の再接続、同じポートの使用、USB 2.0 ポート / ハブを試す |
| `fastboot reboot recovery` が `Unsupported` | ブートローダーメニューから Recovery Mode を選ぶ |
| 通常再起動でストック Android が起動 | 通常起動では Recovery に入らない。Recovery Mode を選択する。boot を両スロットへ書き込む |
| sideload が 47% で止まる | `adb: failed to read command: Success` は正常終了 |
| `libnos_datagram` エラー | Titan M 関連の既知の無害なエラー。`Data wipe complete.` を確認 |

## インストール後の注意

- ブートローダーは unlocked のままにする。LineageOS では relock 非推奨。
- 起動時のブートローダー警告は仕様。無視してよい。
- 更新は 設定 → システム → アップデーター から行う。
- データ消去後は USB デバッグがオフに戻る。必要なら再度有効化する。
- ワイヤレスデバッグ（Android 11 以降）を使う場合は、開発者向けオプションから有効化する。
- ホストの udev ルールは USB ADB / fastboot を使い続けるなら残しておく。
- ダウンロードしたファイル（約 1.6 GB）は、動作確認後に削除してよい。
- root 化と `sudo` の導入は [Magisk.md](Magisk.md) を参照。

## 参考リンク

- [LineageOS Wiki - Google Pixel 3a (sargo)](https://wiki.lineageos.org/devices/sargo/)
- [Install LineageOS on sargo](https://wiki.lineageos.org/devices/sargo/install/)
- [LineageOS Downloads - sargo](https://download.lineageos.org/devices/sargo)
- [LineageOS Google Apps](https://wiki.lineageos.org/gapps/)
- [Factory Images for Nexus and Pixel Devices](https://developers.google.com/android/images)
- [Full OTA Images for Nexus and Pixel Devices](https://developers.google.com/android/ota)
