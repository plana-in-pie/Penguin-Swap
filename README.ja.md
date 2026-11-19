# penguin-swap — ChromeOS (Crostini / Flex) の Linux コンテナでスワップを有効化する

[English](README.md)

質問・フィードバックは [Bluesky @plana-in-pie.bsky.social](https://bsky.app/profile/plana-in-pie.bsky.social) へどうぞ。

## 前提：ペンギンにはスワップが用意されていない

ChromeOS の Linux コンテナ（penguin）には、初期状態ではスワップ領域が存在しない（`/proc/swaps` は空で、`free` の Swap は 0）。ChromeOS の設定にも、コンテナのスワップを増やす項目はない。メモリが不足すれば、そのままプロセスが落ちる。

## 問題

自前でスワップファイルを作ろうとしても、通常の手順では有効化できない。コンテナのルートは btrfs で、それが壁になる。

```console
$ sudo fallocate -l 2G /swapfile
$ sudo chmod 600 /swapfile
$ sudo mkswap /swapfile
$ sudo swapon /swapfile
swapon: /swapfile: swapon failed: Invalid argument
```

エラーメッセージは理由を教えてくれない。プラットフォームが禁止しているわけではない。**ファイルの属性**が原因。

## 原因

btrfs 上のスワップファイルには、カーネル側の要件がある。

- コピーオンライト（CoW）が無効であること（NOCOW）
- 穴（ホール）がなく、全量が確保されていること
- root 所有で、パーミッションが 600 であること

通常の作り方では CoW が有効なままなので、`swapon` が `Invalid argument` で弾く。

## 手順

空ファイルに NOCOW を付けてから、全量を書き込んで確保するのが肝。

```bash
# 2GB のスワップファイルを作る
sudo touch /swapfile
sudo chattr +C /swapfile          # NOCOW（空ファイルのうちに付ける）
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 確認
cat /proc/swaps
free -h
```

容量を増やしたい場合は、2つ目のファイルを追加できる（合計容量になる）。

```bash
sudo touch /swapfile2
sudo chattr +C /swapfile2
sudo dd if=/dev/zero of=/swapfile2 bs=1M count=2048 status=progress
sudo chmod 600 /swapfile2
sudo mkswap /swapfile2
sudo swapon /swapfile2
```

## 起動のたびに：有効化

**スワップはコンテナの再起動で外れる。** ファイル（と署名）は残っているので、起動後に有効化し直すだけでよい。作り直しも `mkswap` も不要。

```bash
sudo swapon /swapfile /swapfile2   # 作ったスワップファイルを全部並べる
cat /proc/swaps                    # 確認
```

## 自動で有効化する

手動の手順を省くには、`~/.bashrc` に以下のブロックを足す。シェルの起動時（ターミナルアプリを開いた時など）に、各ファイルを確認して、まだ有効でないものだけを有効化する。

```bash
cat >> ~/.bashrc <<'EOF'

# penguin-swap: activate swap files at shell start
for _f in /swapfile /swapfile2; do
  if [ -f "$_f" ] && ! grep -q "^$_f " /proc/swaps; then sudo -n /usr/sbin/swapon "$_f" 2>/dev/null; fi
done
unset _f
EOF
```

ファイルの一覧は、自分が作ったスワップファイルに合わせること。前提は、penguin の標準であるパスワードなし sudo。2026-11-19 に実測（ChromeOS Flex、Debian 13）：コンテナ再起動後にターミナルを開くだけで、4GB 全量がコマンドなしで有効になった。

## 自動で有効化をやめる

`~/.bashrc` に足したブロックを削除する。次のシェルから、自動では有効化されなくなる。すぐにスワップ自体も止めるなら：

```bash
sudo swapoff /swapfile /swapfile2
```

同日に実測：ブロックを外して再起動した後は、`/proc/swaps` は空のままだった。

## 解除する

スワップを止めて、ディスク容量を取り戻すには：

```bash
sudo swapoff /swapfile /swapfile2
sudo rm /swapfile /swapfile2
free -h   # Swap が 0 に戻る
```

## 運用上の注意

- スワップはコンテナ全体で共有される。特定のアプリ専用ではない。
- スワップは「容量」を増やすもので、「速度」は増やさない。実際に使うデータは RAM にないと実用にならない。ただし SSD 上では、読み戻しの待ち時間が小さいため、一時的なピークの吸収にはよく効く（HDD 時代のスラッシングの記憶とは別物）。

## 実測（2026-11-19）

- 環境：MacBook Air 13-inch Early 2014 に ChromeOS Flex、Linux コンテナ（Debian 13、btrfs）。コンテナのメモリ総量は 2,792MB で固定（引き上げ手段なし）。
- 対象：Wine 11.0 上で動作する PC 版ゲーム（ホーム画面の占有 約2.3GB＋常駐 約0.6GB で、総量を超えて必ず落ちていた）。
- スワップなし：起動試験8回でホーム到達4回、持続0回。
- スワップ2GB：ホーム到達し、設定画面や BGM 再生など数分間の操作が可能になった。ただしピーク通過後の余裕がほぼ無く、継続操作で終了。
- スワップ4GB：ホーム持続・カフェ（3D）・生徒一覧まで到達。お知らせの WebView を「今日は表示しない」で抑止すると最大のスパイク（約950MB）が消え、さらに安定した。

## ライセンス

MIT License
