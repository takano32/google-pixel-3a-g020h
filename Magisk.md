# Magisk による root 化と sudo の導入

Google Pixel 3a（G020H / sargo）の LineageOS 22.2 環境に Magisk で root を入れ、
Termux から `sudo` を使えるようにした際の作業記録です。

> 2026-11-24: ファクトリーリセットを実施した。Termux と Magisk アプリは削除されたが、
> boot の Magisk パッチは残っていた。再セットアップの記録は
> 「ファクトリーリセット後の再 root」を参照。

## 結果

| 項目 | 内容 |
| --- | --- |
| 端末 | Google Pixel 3a G020H（日本向け / codename: sargo） |
| OS | LineageOS 22.2（Android 15） |
| ビルド | `lineage-22.2-20261121-NIGHTLY-sargo-signed`（2026-11-21） |
| ビルド種別 | userdebug |
| ブートローダー | unlocked |
| アクティブスロット | b |
| root | Magisk v31.1（31100） |
| boot パッチ | 公式 `boot_patch.sh` で LineageOS の `boot.img` をパッチ |
| sudo | Termux 0.118.3 + `sudo` 1.3.0 |
| 確認 | `sudo id` → `uid=0(root)` |
| 実施日 | 2026-11-23 |

## 使用ファイル

ダウンロード先: `~/Downloads/lineage-sargo/`

| ファイル | サイズ | SHA256 |
| --- | --- | --- |
| `lineage-22.2-20261121-nightly-sargo-signed.zip` | 約 1.1 GB | `147b00d1d97251dad31308fa91d7f9e010c1d5036f0f3972dffcd844e4859546` |
| `boot.img`（Lineage Recovery / ROM の boot） | 64 MB | `888655b714e099556f3098644d81113dd9c990763bfd208c34a9c23f6df1b74f` |
| `Magisk-v31.1.apk` | 約 10 MB | 公式 GitHub Releases から取得 |
| `magisk_patched-31100.img` | 64 MB | `7c8ce927706d26824fd8deeb6ecb4c3f90535088facaac6c7e1fb05864eb3172` |

補足:

- sargo は独立した recovery パーティションを持たず、`boot` が Recovery を兼ねる。
  ROM の `boot.img` は Lineage Recovery のイメージと同一（SHA256 一致）。
- `magisk_patched-31100.img` は現行ビルド専用。LineageOS を OTA 更新したら
  新しい `boot.img` を抽出してパッチし直す。

## Magisk v31.1 の仕様変更

Magisk v31.1 では、従来の「アプリが `su` を要求するとダイアログが出る」方式が廃止された。
root 許可は Magisk アプリの **スーパーユーザー** タブで事前に付与する。

- 対話式の root 許可プロンプトは出ない（HSUM 対応のための変更）。
- アプリ隠し（stub / repackaging）機能は削除された。
- `su` 本体は `/debug_ramdisk/su`。Magisk は PATH 上のディレクトリに `su` への
  リンクを注入し、この端末では `/product/bin/su` に作られた。
  `/system/bin/su` は存在しない。
- `adb shell`（uid 2000）からの `su` は既定で拒否される。
  必要なら Magisk で `com.android.shell` を許可するか、開発者向けオプションの
  「Rooted debugging」を有効にする。
- MagiskSU の `--help` には `-c` と `--mount-master` が表示されない。
  Termux の `sudo` 1.2.0 はこの表示をチェックするため失敗する。**1.3.0 以降が必要**。

## 手順

### 1. ROM から boot.img を抽出

LineageOS の zip は A/B 用の `payload.bin` を含むため、`payload-dumper-go` で取り出す。

```bash
cd /tmp
curl -LO https://github.com/ssut/payload-dumper-go/releases/download/2.1.0/payload-dumper-go_2.1.0_linux_arm64.tar.gz
tar xzf payload-dumper-go_2.1.0_linux_arm64.tar.gz
./payload-dumper-go -p boot -o boot \
  ~/Downloads/lineage-sargo/lineage-22.2-20261121-nightly-sargo-signed.zip

sha256sum boot/boot.img
# 888655b714e099556f3098644d81113dd9c990763bfd208c34a9c23f6df1b74f
```

ホストが x86_64 の場合は `linux_amd64` を取得する。

### 2. Magisk で boot.img をパッチ

#### 方法 A: Magisk アプリを使う場合

1. `Magisk-v31.1.apk` を端末にインストールする。
2. Magisk アプリ →「Magisk」のインストール / 再インストール
   →「ファイルを選択してパッチ」→ `boot.img` を選択。
3. 出力された `magisk_patched-*.img` を端末から取り出す。

#### 方法 B: 公式スクリプトを adb shell で実行（今回の方法）

root は不要。`/data/local/tmp` 上でパッチできる。

```bash
# APK から必要なファイルを展開
unzip -o Magisk-v31.1.apk \
  'lib/arm64-v8a/*' 'assets/boot_patch.sh' 'assets/util_functions.sh' -d magisk
cd magisk
for f in magisk magiskinit magiskboot init-ld; do
  cp "lib/arm64-v8a/lib${f}.so" "$f"
done
cp assets/boot_patch.sh assets/util_functions.sh .

# 端末へ転送してパッチ
mkdir -p /data/local/tmp/magisk
adb push ./. /data/local/tmp/magisk/
adb push ../boot/boot.img /data/local/tmp/magisk/boot.img
adb shell 'cd /data/local/tmp/magisk && chmod -R 755 . && \
  KEEPVERITY=true KEEPFORCEENCRYPT=true sh boot_patch.sh boot.img'

# 生成物を取り出す
adb pull /data/local/tmp/magisk/new-boot.img magisk_patched-31100.img
```

補足:

- `KEEPVERITY=true KEEPFORCEENCRYPT=true` は AVB / 強制暗号化を維持する設定。
  必要に応じて変更する。
- 生成物は `new-boot.img` という名前になる。
- パッチ後の SHA256 は
  `7c8ce927706d26824fd8deeb6ecb4c3f90535088facaac6c7e1fb05864eb3172`。

### 3. パッチ済み boot を書き込み

```bash
adb reboot bootloader
fastboot devices
fastboot --slot all flash boot magisk_patched-31100.img
fastboot reboot
```

- `--slot all` で `boot_a` / `boot_b` の両方に書き込む。
- Magisk アプリが再起動を要求した場合は従う。
  この環境では再起動後に `/product/bin/su` が見えるようになった。
- 起動確認:

```bash
adb shell '/debug_ramdisk/su -v'    # 31.1:MAGISKSU
adb shell 'ps -A | grep magiskd'    # magiskd が動作
```

### 4. Magisk アプリで root を許可

Magisk v31.1 では `su` を実行してもプロンプトは出ない。先に許可する。
未インストールの場合は `adb install -r Magisk-v31.1.apk` で入れておく。

1. Magisk アプリを開く。
2. 下部の **スーパーユーザー** タブを開く。
3. 右下の **+**（Grant Superuser）をタップ。
4. 一覧から許可したいアプリ（Termux）を探してタップし、チェックを付ける。
5. 必要なら検索欄を使う。システムアプリを許可する場合は「システムアプリを表示」を有効にする。

`com.android.shell` を許可すると `adb shell su` も使える。
または 設定 → システム → 開発者向けオプション → **Rooted debugging**
（root 化されたデバッグ）を有効にして `adb root` を使う方法もある（userdebug ビルドのため）。

### 5. Termux と sudo の導入

#### Termux のインストール

F-Droid の安定版 0.118.3（versionCode 1002）を使用した。ホスト側で APK を取得する。

```bash
curl -LO https://f-droid.org/repo/com.termux_1002.apk

# Android 15 の Play Protect により INSTALL_FAILED_VERIFICATION_FAILURE になる場合
adb shell settings put global verifier_verify_adb_installs 0
adb install -r com.termux_1002.apk
adb shell settings put global verifier_verify_adb_installs 1
```

インストール後も Play Protect が「古い Android 向け」として警告を出すが、
動作に問題はない。気になる場合は F-Droid の beta や GitHub Releases の
新しい Termux を検討する。

#### sudo のインストール

Termux の既定ミラー（mephi.ru）では `sudo` 1.2.0 が配布されており、
Magisk v31.1 では動作しない。公式リポジトリに切り替えて 1.3.0 を入れる。

Termux 内で:

```sh
echo 'deb https://packages.termux.dev/apt/termux-main stable main' \
  > $PREFIX/etc/apt/sources.list
pkg update -y
pkg install -y sudo
```

`No mirror or mirror group selected` という警告が出るが、`sources.list` を直接書いた
場合は無害。`termux-change-repo` を使ってもよい。

#### 動作確認

Magisk で Termux を許可した後、Termux で:

```sh
sudo id
# uid=0(root) gid=0(root) groups=0(root) context=u:r:magisk:s0
```

`sudo` 1.2.0 では次のエラーになる:

```
No 'su' binary found that supports the '-c', '--shell',
'--preserve-environment' and '--mount-master' options
```

これは MagiskSU のヘルプ表示に `-c` / `--mount-master` が含まれないため。
`sudo` 1.3.0 では実際に `su --mount-master -c ...` を試す判定に変更されている。

### 6. 使い方

```sh
sudo id
sudo su          # root シェル
sudo <command>   # 単発コマンド
```

## ファクトリーリセット後の再 root（2026-11-24）

ファクトリーリセット後も、**boot の Magisk パッチと `magiskd` は残っていた**。
`/data` が消えるため Termux・Magisk アプリ・Magisk の許可 DB は消えるが、
root の仕組み自体は boot にあるため再 flash / 再パッチは不要だった。

### 状態確認

```bash
adb devices
adb shell 'getprop ro.lineage.version'   # 22.2-20261121-NIGHTLY-sargo
adb shell 'ps -A | grep magiskd'          # magiskd が root で動作
adb shell 'ls -l /debug_ramdisk/su'       # su が存在
```

起動直後は `/product/bin/su` がまだ見えないことがある。Magisk アプリが再起動を
要求した場合は従う。

### 再セットアップ

1. ファクトリーリセットで USB デバッグがオフに戻るので、開発者向けオプションで再度有効化する。
2. Magisk アプリを再インストールする。

   ```bash
   adb install -r ~/Downloads/lineage-sargo/Magisk-v31.1.apk
   ```

3. Magisk アプリ → **スーパーユーザー** → **+** → フィルタ（**システムアプリを表示**）
   → `com.android.shell`（表示名「シェル」）を検索して許可する。
4. 確認:

   ```bash
   adb shell su -c id
   # uid=0(root) gid=0(root) groups=0(root) context=u:r:magisk:s0
   ```

今回の再構成では Termux / `sudo` は入れず、ADB シェル向けの root のみにした。

- `com.android.shell` を許可すると、USB デバッグを許可した adb 接続元から root 操作ができる。
  不要なら Magisk のスーパーユーザー画面で無効化する。
- ファクトリーリセットで `/data` が消えるため、以前の Magisk 許可設定は消える。
  root 自体は boot パッチなので残る。
- boot を純正に戻した場合は root も消える。再 root は
  `magisk_patched-31100.img` を再 flash するか、新しい `boot.img` をパッチする。
- Termux と `sudo` が必要な場合は「5. Termux と sudo の導入」を再実行する
  （`sudo` は 1.3.0 以降）。

## 注意 / 運用

- LineageOS の OTA 更新で `boot` が上書きされ、Magisk は消える。
  OTA 後に新しい `boot.img` を抽出し、再度パッチ・書き込みを行う。
- `magisk_patched-31100.img` は現行ビルド専用。OTA 後に再利用しない。
- sargo の `boot` は Recovery 兼用のため、Magisk パッチ版を書き込むと
  Lineage Recovery は使えなくなる。
  Recovery を戻す場合は純正の `boot.img` を書き込む（同時に Magisk も消える）。

  ```bash
  fastboot --slot all flash boot ~/Downloads/lineage-sargo/boot.img
  ```

- ブートローダーは unlocked のまま。起動時の警告は仕様。
- Magisk v31.1 はアプリを隠さず、Magisk アプリを自動で再インストールしない。
  必要なら自分で管理する。
- root 許可は事前付与のみ。付与したアプリは無効化するまで永続的に root を使える。
- `com.android.shell` を許可すると、adb 接続元から root 操作が可能になる。取り扱いに注意する。
- USB 接続が頻繁に切れる場合は、ケーブル・ポート・ハブを確認する。

## トラブルと対処

| 症状 | 原因 / 対処 |
| --- | --- |
| `adb install` が `INSTALL_FAILED_VERIFICATION_FAILURE` | Play Protect の検証。一時的に `verifier_verify_adb_installs=0` にしてインストール |
| Termux 起動時に Play Protect の警告 | Termux 0.118.3 の targetSdk が古いため。動作には影響なし |
| `sudo` が `No 'su' binary found ...` | `sudo` 1.2.0 が Magisk v31.1 非対応。1.3.0 へ更新 |
| `adb shell su` が `Permission denied` | v31.1 は対話プロンプトなし。`com.android.shell` を許可するか Termux を使う |
| Magisk アプリが再起動を要求 | 初回セットアップの完了。再起動する |
| `/system/bin/su` が見つからない | v31.1 では `/debug_ramdisk/su` / `/product/bin/su` を使う。異常ではない |
| `fastboot devices` が表示されない | USB の再接続、同じポート、USB 2.0 ポート / ハブを試す |
| OTA 後に root が消えた | OTA が boot を上書きした。新 boot.img を再パッチして書き込む |
| ファクトリーリセット後に root が消えたように見える | boot パッチは残る。Magisk アプリを再インストールし、`magiskd` と `/debug_ramdisk/su` を確認する |
| `adb root` が `ADB Root access is disabled ...` | 開発者向けオプションの **Rooted debugging** を有効にする。shell から `service call adbroot_service` で変更することはできない（`Call must be system`） |
| ファクトリーリセット後に Termux / sudo が消えた | `/data` が消えたため。必要なら Termux と `sudo` 1.3.0 を再導入する |

## 参考リンク

- [Magisk Releases](https://github.com/topjohnwu/Magisk/releases)
- [Magisk Documentation](https://topjohnwu.github.io/Magisk/)
- [Magisk v31.1 Release Notes](https://github.com/topjohnwu/Magisk/releases/tag/v31.1)
- [Termux (F-Droid)](https://f-droid.org/packages/com.termux/)
- [Termux sudo (agnostic-apollo/sudo)](https://github.com/agnostic-apollo/sudo)
- [payload-dumper-go](https://github.com/ssut/payload-dumper-go)
- [LineageOS Wiki - sargo](https://wiki.lineageos.org/devices/sargo/)
- [LineageOS Downloads - sargo](https://download.lineageos.org/devices/sargo)
- [LineageOS 22.2 インストール記録](LineageOS.md)
