---
layout: default
title: /procはなぜファイルなのか
---

# [/procはなぜファイルなのか](#why-proc-is-a-file) {#why-proc-is-a-file}

## [はじめに](#introduction) {#introduction}

[01-process](../../01-process/) の「/proc ファイルシステム」で、プロセスの情報を見るための特別な場所を学びました

```bash
cat /proc/self/status
```

このコマンドを実行すると、自分自身のプロセス情報が表示されます

でも、ちょっと不思議に思いませんか？

`cat` は本来、ファイルの中身を表示するコマンドです

なぜプロセスの情報が「ファイル」として見えるのでしょうか？

このドキュメントでは、/proc が「ファイル」の形式を取る理由と、その設計思想について説明します

---

## [目次](#table-of-contents) {#table-of-contents}

- [仮想ファイルシステムとは](#what-is-virtual-filesystem)
- [Unix哲学「すべてはファイル」](#unix-philosophy-everything-is-a-file)
- [/proc/[pid] の便利なファイル](#useful-files-in-proc-pid)
- [実用例](#practical-examples)
- [まとめ](#summary)
- [参考資料](#references)

---

## [仮想ファイルシステムとは](#what-is-virtual-filesystem) {#what-is-virtual-filesystem}

### [疑似ファイルシステム](#pseudo-filesystem) {#pseudo-filesystem}

<strong>/proc</strong> は<strong>疑似ファイルシステム（pseudo-filesystem）</strong>と呼ばれます

Linux の公式マニュアルには、こう書かれています

> The proc filesystem is a pseudo-filesystem which provides an interface to kernel data structures.

> proc ファイルシステムは、カーネルのデータ構造へのインターフェースを提供する疑似ファイルシステムです

### [通常のファイルシステムとの違い](#difference-from-regular-filesystem) {#difference-from-regular-filesystem}

通常のファイルシステムでは、ファイルはハードディスクや SSD などの<strong>ストレージ</strong>に保存されています

しかし、/proc は違います

{: .labeled}
| 項目 | 通常のファイルシステム | /proc |
| -------------- | -------------------------- | ------------------------------ |
| ファイルの場所 | ディスク上 | メモリ上（カーネル内） |
| データの生成 | ファイル作成時 | 読み取り時にリアルタイムで生成 |
| 内容の変化 | 明示的に書き換えるまで不変 | 読み取るたびに最新の情報を返す |
| 容量の消費 | ディスク容量を使う | ディスク容量を使わない |

### [カーネルへの窓口](#gateway-to-kernel) {#gateway-to-kernel}

/proc は「ファイルのふりをしているが、実際はカーネルへの窓口」です

`cat /proc/[pid]/status` を実行すると、以下のことが起こります

1. カーネルが「status というファイルを読みたい」というリクエストを受け取る
2. カーネルがそのプロセスの情報をリアルタイムで収集する
3. 収集した情報を「ファイルの中身」として返す

ディスク上にファイルがあるわけではなく、カーネルが情報を動的に生成しているのです

---

## [Unix哲学「すべてはファイル」](#unix-philosophy-everything-is-a-file) {#unix-philosophy-everything-is-a-file}

### [設計思想の起源](#origin-of-design-philosophy) {#origin-of-design-philosophy}

1970年代初頭、Ken Thompson と Dennis Ritchie が Unix を開発していたとき、ある問題に直面しました

当時のコンピュータシステムでは、ディスクアクセス、ターミナル制御、プロセス間通信など、それぞれに異なるインターフェースが必要でした

新しい機能が追加されるたびに、新しい API を覚える必要があったのです

彼らの解決策は、シンプルでした

<strong>すべてをファイルとして扱う</strong>

ハードドライブも、ターミナルも、プロセス情報も、すべて同じ操作（open、read、write、close）で扱えるようにしたのです

### [なぜファイルの形式なのか](#why-file-format) {#why-file-format}

<strong>もし専用のシステムコールだったら？</strong>

プロセス情報を取得する専用のシステムコールを作る方法もあります

```c
/* 仮想的な専用システムコール */
struct process_status status;
get_process_status(pid, &status);
printf("メモリ: %ld\n", status.vm_rss);
```

しかし、この設計には問題があります

{: .labeled}
| 問題 | 説明 |
| -------------------------- | ---------------------------------------------------- |
| 新機能ごとに API 追加 | カーネルに情報が増えるたびに、新しい関数が必要になる |
| 言語ごとにラッパーが必要 | C、Python、Go、それぞれに専用ライブラリが必要 |
| シェルからアクセスできない | スクリプトで使うには追加のツールが必要 |
| 既存ツールが使えない | grep、diff、watch などが使えない |

ファイルとして公開すれば、これらの問題がすべて解決します

<strong>ファイル形式の利点</strong>

<strong>1. 既存のツールがそのまま使える</strong>

```bash
# プロセスの状態を見る
cat /proc/self/status

# 特定の情報だけ抽出する
grep VmRSS /proc/self/status

# 複数のプロセスを比較する
diff /proc/1234/status /proc/5678/status
```

cat、grep、diff などの使い慣れたコマンドで、カーネルの情報にアクセスできます

<strong>2. 特別な API を覚えなくて良い</strong>

プロセス情報を取得するための専用関数を覚える必要がありません

ファイルを読む方法さえ知っていれば、カーネルの情報にアクセスできます

<strong>3. プログラミング言語を問わない</strong>

C 言語でも、Python でも、シェルスクリプトでも、ファイルを読み書きできる言語ならどれでもカーネルの情報にアクセスできます

### [他の例](#other-examples) {#other-examples}

/proc 以外にも、「すべてはファイル」の設計思想に基づいたものがあります

{: .labeled}
| パス | 内容 |
| ----------- | ------------------------------ |
| /dev/null | 書き込んだデータをすべて捨てる |
| /dev/zero | 読み取ると無限にゼロを返す |
| /dev/random | ランダムなデータを返す |
| /dev/tty | 現在のターミナル |

これらも「ファイルのふり」をしていますが、実際にはカーネルが提供する特別な機能です

---

## [/proc/\[pid\] の便利なファイル](#useful-files-in-proc-pid) {#useful-files-in-proc-pid}

各プロセスには `/proc/[pid]/` というディレクトリがあります

ここでは、よく使うファイルを紹介します

### [status](#status) {#status}

プロセスの状態を人間が読みやすい形式で表示します

```bash
cat /proc/self/status
```

主なフィールド

{: .labeled}
| フィールド | 内容 |
| ---------- | ---------------------------------- |
| Name | プロセス名 |
| State | 状態（R: 実行中、S: 待機中、など） |
| Pid | プロセス ID |
| PPid | 親プロセスの ID |
| Uid | ユーザー ID |
| VmSize | 仮想メモリサイズ |
| VmRSS | 物理メモリ使用量 |
| Threads | スレッド数 |

### [cmdline](#cmdline) {#cmdline}

プロセス起動時のコマンドラインを表示します

```bash
cat /proc/self/cmdline | tr '\0' ' '
```

引数は NULL 文字（`\0`）で区切られているため、`tr` で空白に置換すると読みやすくなります

### [exe](#exe) {#exe}

実行ファイルへの<strong>シンボリックリンク</strong>です

```bash
readlink /proc/self/exe
```

そのプロセスが実行しているプログラムファイルのパスがわかります

### [cwd](#cwd) {#cwd}

<strong>作業ディレクトリ</strong>へのシンボリックリンクです

```bash
readlink /proc/self/cwd
```

プロセスが「今いる場所」がわかります

### [environ](#environ) {#environ}

<strong>環境変数</strong>を表示します

```bash
cat /proc/self/environ | tr '\0' '\n'
```

環境変数は NULL 文字で区切られているため、`tr` で改行に置換すると読みやすくなります

### [fd/](#fd) {#fd}

開いているファイルディスクリプタの一覧です

```bash
ls -la /proc/self/fd/
```

各番号がシンボリックリンクになっており、どのファイルを開いているかがわかります

標準で開いているファイルディスクリプタ

{: .labeled}
| 番号 | 名前 | 説明 |
| ---- | ------ | -------------- |
| 0 | stdin | 標準入力 |
| 1 | stdout | 標準出力 |
| 2 | stderr | 標準エラー出力 |

### [maps](#proc-pid-maps) {#proc-pid-maps}

メモリマップを表示します

```bash
cat /proc/self/maps
```

プロセスのメモリ配置（テキスト、ヒープ、スタックなど）がわかります

詳しくは [appendices/memory-layout.md](../memory-layout/) を参照してください

### [/proc/self](#proc-self) {#proc-self}

<strong>/proc/self</strong> は特別なシンボリックリンクで、「自分自身のプロセス」を指します

自分の PID を知らなくても、`/proc/self/status` で自分の情報を見られます

---

## [実用例](#practical-examples) {#practical-examples}

### [メモリ使用量を確認する](#checking-memory-usage) {#checking-memory-usage}

```bash
grep -E '^(VmSize|VmRSS)' /proc/self/status
```

出力例

```
VmSize:    12345 kB
VmRSS:      5678 kB
```

{: .labeled}
| フィールド | 意味 |
| ---------- | -------------------------------------- |
| VmSize | 仮想メモリサイズ（確保した総量） |
| VmRSS | 物理メモリ使用量（実際に使っている量） |

### [開いているファイルを確認する](#checking-open-files) {#checking-open-files}

```bash
ls -la /proc/self/fd/
```

出力例

```
lrwx------ 1 user user 64 Jan  1 12:00 0 -> /dev/pts/0
lrwx------ 1 user user 64 Jan  1 12:00 1 -> /dev/pts/0
lrwx------ 1 user user 64 Jan  1 12:00 2 -> /dev/pts/0
```

/dev/pts/0 は疑似ターミナルを表します

### [C 言語での読み取り](#reading-in-c) {#reading-in-c}

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    FILE *file = fopen("/proc/self/status", "r");
    if (file == NULL) {
        return 1;
    }

    char line[256];
    while (fgets(line, sizeof(line), file) != NULL) {
        /* VmRSS で始まる行を探す */
        if (strncmp(line, "VmRSS:", 6) == 0) {
            printf("物理メモリ使用量: %s", line);
        }
    }

    fclose(file);
    return 0;
}
```

このように、通常のファイル読み取りと同じ方法でカーネルの情報を取得できます

---

## [まとめ](#summary) {#summary}

<strong>/proc がファイルである理由</strong>

{: .labeled}
| ポイント | 説明 |
| ---------------------------- | ---------------------------------------------------------- |
| Unix哲学「すべてはファイル」 | 異なるリソースを統一的なインターフェースで扱う設計思想 |
| 既存ツールの活用 | cat、grep、diff などのコマンドでカーネル情報にアクセス可能 |
| 学習コストの削減 | ファイル操作を知っていれば、特別な API は不要 |

<strong>覚えておくこと</strong>

{: .labeled}
| ポイント | 説明 |
| --------------------------------- | ---------------------------------------------------------- |
| /proc はディスク上に存在しない | カーネルがリアルタイムで情報を生成する疑似ファイルシステム |
| /proc/[pid]/ にプロセス情報がある | status、cmdline、fd/ などで詳細情報を確認可能 |
| /proc/self は自分自身を指す | 自分の PID を知らなくても自分の情報にアクセスできる |

---

## [参考資料](#references) {#references}

<strong>Linux マニュアル</strong>

- [proc(5) - Linux manual page](https://man7.org/linux/man-pages/man5/proc.5.html){:target="\_blank"}
  - /proc ファイルシステムの全体説明
- [proc_pid_status(5) - Linux manual page](https://man7.org/linux/man-pages/man5/proc_pid_status.5.html){:target="\_blank"}
  - /proc/[pid]/status の詳細
- [proc_pid_maps(5) - Linux manual page](https://man7.org/linux/man-pages/man5/proc_pid_maps.5.html){:target="\_blank"}
  - /proc/[pid]/maps の詳細

<strong>Linux カーネルドキュメント</strong>

- [The /proc Filesystem - Linux Kernel documentation](https://docs.kernel.org/filesystems/proc.html){:target="\_blank"}
  - カーネル視点からの /proc の説明

<strong>本編との関連</strong>

- [01-process](../../01-process/)
  - /proc ファイルシステムの概要
- [appendices/memory-layout.md](../memory-layout/)
  - /proc/[pid]/maps の読み方
