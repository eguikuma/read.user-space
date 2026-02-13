---
layout: default
title: OS のファイル管理
---

# [05-file-descriptor：OS のファイル管理](#os-file-management) {#os-file-management}

## [はじめに](#introduction) {#introduction}

前のトピック（04-thread）で、スレッドによる並行処理を学びました

そこで「スレッドはファイルディスクリプタを共有する」ことに触れました

また、[01-process](../01-process/) では、プロセスが開いているファイルを「観察」しました

`/proc/[pid]/fd/` ディレクトリを見ると、プロセスが開いているファイルの一覧が表示されました

でも、「ファイルディスクリプタとは何なのか」「どうやって使うのか」は詳しく説明しませんでした

このトピックでは、<strong>ファイルディスクリプタの仕組み</strong>を詳しく学びます

### [日常の例え](#everyday-analogy) {#everyday-analogy}

ファイルディスクリプタを「図書館の貸出カード番号」に例えてみましょう

図書館（OS）には、たくさんの本（ファイル）があります

本を借りるとき、図書館は「貸出番号 3 番」のように番号を割り当てます

あなたはこの番号を使って、本を読んだり、返却したりします

本棚のどこにあるかを覚える必要はありません

番号さえ分かれば、図書館が本を探してきてくれます

ファイルディスクリプタも同じです

プログラムがファイルを開くと、OS が「fd 3」のように番号を割り当てます

この番号を使って、読み書きや閉じる操作を行います

本を返却すると、その番号は次の貸出に再利用されることがあります

fd も同様に、close() 後は別のファイルに再利用されます

### [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

このページでは、以下のシステムコールを学びます

- <strong>open()</strong>
  - ファイルを開き、ファイルディスクリプタを取得する
- <strong>close()</strong>
  - ファイルを閉じ、ファイルディスクリプタを解放する
- <strong>read()</strong>
  - ファイルからデータを読み取る
- <strong>write()</strong>
  - ファイルにデータを書き込む
- <strong>lseek()</strong>
  - 読み書き位置を移動する
- <strong>dup() / dup2()</strong>
  - ファイルディスクリプタを複製する（リダイレクトの基盤）

---

## [目次](#table-of-contents) {#table-of-contents}

1. [ファイルディスクリプタとは何か](#what-is-a-file-descriptor)
2. [標準入出力と標準エラー出力](#standard-io-and-stderr)
3. [カーネルの 3 層構造](#kernel-three-layer-structure)
4. [open() によるファイルオープン](#opening-files-with-open)
5. [read() と write()](#read-and-write)
6. [close() の重要性](#importance-of-close)
7. [lseek() による位置制御](#position-control-with-lseek)
8. [dup() と dup2() によるリダイレクト](#redirect-with-dup-and-dup2)
9. [fork() と fd の継承](#fork-and-fd-inheritance)
10. [次のステップ](#next-steps)
11. [用語集](#glossary)
12. [参考資料](#references)

---

## [ファイルディスクリプタとは何か](#what-is-a-file-descriptor) {#what-is-a-file-descriptor}

### [基本的な説明](#basic-explanation) {#basic-explanation}

<strong>ファイルディスクリプタ（fd）</strong>は、プロセスが開いたファイルに対して OS が割り当てる整数値です

Linux の公式マニュアルには、こう書かれています

> A file descriptor is a small, non-negative integer that is used as a handle to refer to an open file.

> ファイルディスクリプタは、開いているファイルを参照するためのハンドルとして使われる、小さな非負の整数です

<strong>ハンドル</strong>とは、何かを操作するための「取っ手」のようなものです

ファイルディスクリプタは、ファイルを操作するためのハンドルです

### [なぜ整数値なのか](#why-an-integer) {#why-an-integer}

<strong>もし文字列（ファイル名）で識別していたら？</strong>

- ファイル名を毎回比較するのは遅い
- 同じファイルを複数回開いたとき、区別できない
- ファイルが移動・リネームされたとき、参照が切れる

<strong>整数値のメリット</strong>

fd は配列のインデックスとして使えます

```c
/* OS 内部のイメージ */
struct file *fd_table[1024];  /* ファイルディスクリプタテーブル */

/* fd=3 のファイルにアクセス */
struct file *f = fd_table[3];  /* O(1) でアクセス可能 */
```

- <strong>高速アクセス</strong>：配列のインデックスとして直接参照できる（O(1) = 定数時間）
- <strong>単純な管理</strong>：整数の比較や代入は高速
- <strong>参照の安定性</strong>：ファイル名が変わっても、開いている fd は有効

プログラムから見ると、ファイルは「どこかにあるデータの塊」です

OS がこの複雑さを隠してくれます

プログラムは「fd 3 番から読みたい」と言うだけで、OS がファイルの場所を探し、データを取ってきてくれます

### [「ファイル」の意味](#meaning-of-file) {#meaning-of-file}

Unix では「すべてはファイル」という設計思想があります

ファイルディスクリプタが指すものは、通常のファイルだけではありません

- 通常のファイル（テキストファイル、画像など）
- <strong>ディレクトリ</strong>（フォルダ、ファイルを整理するための入れ物）
- <strong>パイプ</strong>（プロセス間でデータをやり取りする仕組み、[07-ipc](../07-ipc/) で学習）
- <strong>ソケット</strong>（ネットワーク通信のための接続口）
- <strong>デバイス</strong>（ハードウェアを抽象化したもの、`/dev/null` など）

これらすべてを、同じ read()、write()、close() で操作できます

---

## [標準入出力と標準エラー出力](#standard-io-and-stderr) {#standard-io-and-stderr}

### [3 つの特別なファイルディスクリプタ](#three-special-file-descriptors) {#three-special-file-descriptors}

プロセスが起動すると、3 つのファイルディスクリプタが自動的に開かれています

{: .labeled}
| fd | 名前 | 説明 | C 言語のシンボル |
| --- | ------------------------ | ------------------------ | ---------------- |
| 0 | 標準入力（stdin） | キーボードからの入力 | STDIN_FILENO |
| 1 | 標準出力（stdout） | 画面への出力 | STDOUT_FILENO |
| 2 | 標準エラー出力（stderr） | エラーメッセージ用の出力 | STDERR_FILENO |

### [なぜ最初から開かれているのか](#why-open-from-the-start) {#why-open-from-the-start}

<strong>もしプログラムが自分で開く必要があったら？</strong>

- すべてのプログラムが「キーボード」「画面」を開くコードを書く必要がある
- シェルのリダイレクト（`>` や `<`）が機能しない
- パイプ（`|`）でプログラムを繋ぐことができない

<strong>規約として決めることのメリット</strong>

- プログラムは「fd 0 から読み、fd 1 に書く」だけで良い
- 実際の接続先が何か（キーボード、ファイル、パイプ）を知らなくて良い
- シェルが裏で接続先を変えられる（リダイレクト）

この規約のおかげで、`cat < input.txt | grep foo > output.txt` のような組み合わせが可能になります

STDIN_FILENO、STDOUT_FILENO、STDERR_FILENO は、unistd.h（Unix 標準ヘッダーファイル）で定義されている定数です

fd の値（0、1、2）に名前を付けたもので、コードの可読性（読みやすさ）を高めます

### [なぜ 3 つあるのか](#why-three) {#why-three}

標準出力と標準エラー出力が分かれているのは、出力を分離できるようにするためです

```bash
./program > output.txt 2> error.txt
```

このコマンドは、標準出力を `output.txt` に、標準エラー出力を `error.txt` にリダイレクトします

プログラムの結果とエラーメッセージを別々に保存できます

### [printf() と write() の関係](#relationship-between-printf-and-write) {#relationship-between-printf-and-write}

printf() は内部で fd 1（標準出力）に write() しています

```c
printf("Hello\n");

/* 上と同じ効果 */
write(1, "Hello\n", 6);
```

printf() は便利な機能（フォーマット、バッファリング）を提供しますが、最終的には write() を呼び出しています

<strong>バッファリング</strong>とは、データを一時的に溜めておいて、まとめて処理する仕組みです

詳しくは 06-stdio で学びます

---

## [カーネルの 3 層構造](#kernel-three-layer-structure) {#kernel-three-layer-structure}

### [境界トピックとして](#as-boundary-topic) {#as-boundary-topic}

ファイルディスクリプタの「裏側」には、<strong>カーネル</strong>（OS の中核）内部の複雑な仕組みがあります

カーネルについては [01-process](../01-process/) で説明しています

ここでは、プログラマが知っておくべき概念を簡単に説明します

詳細はカーネル空間の学習で扱います

### [3 つのテーブル](#three-tables) {#three-tables}

カーネルは、ファイルを管理するために 3 つのテーブルを持っています

<strong>1. ファイルディスクリプタテーブル（プロセスごと）</strong>

各プロセスが持つテーブルです

fd 番号から「ファイルテーブルのどのエントリか」を指しています

<strong>エントリ</strong>とは、テーブル（表）の中の「1 行」のことです

<strong>2. ファイルテーブル（システム全体で共有）</strong>

開いているファイルの情報を持つテーブルです

- 現在の読み書き位置（<strong>オフセット</strong>）
- ファイルの状態フラグ（読み取り専用かなど）
- inode テーブルへの参照

<strong>オフセット</strong>とは、ファイルの先頭から何バイト目にいるかを示す位置情報です

<strong>3. inode テーブル（システム全体で共有）</strong>

<strong>inode</strong>（アイノード）は、ファイルの実体に関する情報を持つデータ構造です

「index node」の略で、ファイルシステムがファイルを管理するための「カード」のようなものです

inode テーブルには、以下の情報が含まれます

- ファイルサイズ
- 所有者
- パーミッション
- ディスク上の位置

### [なぜ 3 層に分けるのか](#why-three-layers) {#why-three-layers}

<strong>もし 2 層（fd → inode 直結）だったら？</strong>

```
[2 層構造の問題]
プロセス A の fd 3 → inode（オフセット情報をどこに持つ？）
プロセス B の fd 3 → 同じ inode
```

この構造では、以下のことが実現できません

- <strong>同じファイルを違う位置で読む</strong>：2 つの fd が同じファイルを開いても、別々のオフセットを持てない
- <strong>dup() の動作</strong>：dup() で複製した fd は同じオフセットを共有すべきだが、実現できない
- <strong>fork() 後のオフセット共有</strong>：親子プロセスが同じ位置で読み書きできない

<strong>3 層構造による解決</strong>

中間の「ファイルテーブル」がオフセットを保持することで

- 別々の fd で同じファイルを開けば、別々のオフセットを持てる
- dup() すれば、同じファイルテーブルエントリを指し、オフセットを共有できる
- fork() すれば、親子が同じファイルテーブルエントリを共有できる

### [なぜこの構造が重要か](#why-this-structure-is-important) {#why-this-structure-is-important}

この 3 層構造のおかげで、複数の fd が同じファイルを指すことができます

また、fork() 後に親子プロセスがオフセットを共有する理由も、この構造で説明できます

---

## [open() によるファイルオープン](#opening-files-with-open) {#opening-files-with-open}

### [基本的な使い方](#basic-usage) {#basic-usage}

open() は、ファイルを開いてファイルディスクリプタを返します

```c
#include <fcntl.h>

int fd = open("file.txt", O_RDONLY);
```

### [引数の説明](#argument-description) {#argument-description}

{: .labeled}
| 引数 | 型 | 説明 |
| --------- | ------------ | ---------------------- |
| 第 1 引数 | const char\* | ファイルのパス |
| 第 2 引数 | int | フラグ（開き方を指定） |
| 第 3 引数 | mode_t | パーミッション（任意） |

<strong>mode_t</strong> は、ファイルのパーミッション（アクセス権限）を表す型です

### [主要なフラグ](#main-flags) {#main-flags}

{: .labeled}
| フラグ | 説明 |
| --------- | ---------------------------------- |
| O_RDONLY | 読み取り専用で開く |
| O_WRONLY | 書き込み専用で開く |
| O_RDWR | 読み書き両用で開く |
| O_CREAT | ファイルが存在しなければ作成する |
| O_TRUNC | ファイルが存在すれば中身を空にする |
| O_APPEND | 書き込みを常に末尾に追加する |
| O_CLOEXEC | exec() 時に自動的に閉じる |

### [O_CLOEXEC：なぜデフォルトで閉じないのか](#o-cloexec-why-not-default-close) {#o-cloexec-why-not-default-close}

<strong>歴史的経緯</strong>

初期の UNIX では、exec() 後も fd が継承されるのが標準動作でした

シェルのリダイレクトは、この継承を前提に設計されています

```c
/* シェルの動作原理（02-fork-exec で学習） */
fork();
/* 子プロセスでリダイレクトを設定 */
dup2(file_fd, STDOUT_FILENO);
exec(program);  /* fd は継承される */
```

<strong>問題：意図しない fd の継承</strong>

しかし、すべての fd が継承されると問題が生じることがあります

- データベース接続などの fd が子プロセスに漏れる
- セキュリティ上のリスク
- リソースリークの原因

<strong>O_CLOEXEC による解決</strong>

```c
/* exec() 時に自動的に閉じたい fd */
int fd = open("secret.txt", O_RDONLY | O_CLOEXEC);
```

O_CLOEXEC を指定すると、exec() 時にその fd は自動的に閉じられます

既存のプログラムとの互換性を維持するため、「デフォルトで継承、明示的に閉じる」という設計になっています

### [フラグの組み合わせ](#combining-flags) {#combining-flags}

フラグはビット OR（`|`）で組み合わせます

```c
/* 読み書き用で開く */
/* なければ作成し、あれば中身を空にする */
int fd = open("file.txt", O_RDWR | O_CREAT | O_TRUNC, 0644);
```

### [パーミッション](#permissions) {#permissions}

<strong>パーミッション</strong>とは、ファイルに対する「誰が何をできるか」の設定です

O_CREAT を使うとき、第 3 引数でパーミッションを指定します

```c
/* 0644 = 所有者は読み書き可、他は読み取りのみ */
int fd = open("file.txt", O_CREAT | O_WRONLY, 0644);
```

{: .labeled}
| 値 | 意味 |
| ---- | ----------------------------- |
| 0644 | rw-r--r--（一般的なファイル） |
| 0755 | rwxr-xr-x（実行可能ファイル） |
| 0600 | rw-------（所有者のみ） |

### [戻り値](#return-value) {#return-value}

{: .labeled}
| 値 | 意味 |
| ---- | ---------------------------------------- |
| >= 0 | ファイルディスクリプタ（成功） |
| -1 | エラー（errno にエラー番号が設定される） |

errno については、[01-process](../01-process/) の「C 言語の読み方」で説明しています

---

## [read() と write()](#read-and-write) {#read-and-write}

### [read() の基本](#read-basics) {#read-basics}

read() は、ファイルからデータを読み取ります

<strong>バッファ</strong>とは、データを一時的に格納しておく領域（入れ物）です

ファイルから読み取ったデータは、まずバッファに格納されます

```c
#include <unistd.h>

char buffer[100];
ssize_t bytes_read = read(fd, buffer, sizeof(buffer));
```

### [read() の引数](#read-arguments) {#read-arguments}

{: .labeled}
| 引数 | 型 | 説明 |
| --------- | ------ | ------------------------ |
| 第 1 引数 | int | ファイルディスクリプタ |
| 第 2 引数 | void\* | データを格納するバッファ |
| 第 3 引数 | size_t | 読み取る最大バイト数 |

<strong>void\*</strong> は、任意の型のデータを指すことができるポインタです

read() や write() はどんな型のデータでも扱えるため、void\* を使います

size_t と ssize_t については、[01-process](../01-process/) の「C 言語の読み方」で説明しています

### [read() の戻り値](#read-return-value) {#read-return-value}

{: .labeled}
| 値 | 意味 |
| --- | --------------------------- |
| > 0 | 実際に読み取ったバイト数 |
| 0 | ファイル終端（EOF）に達した |
| -1 | エラー |

### [write() の基本](#write-basics) {#write-basics}

write() は、データをファイルに書き込みます

```c
const char *message = "Hello, World!\n";
ssize_t bytes_written = write(fd, message, strlen(message));
```

### [write() の引数](#write-arguments) {#write-arguments}

{: .labeled}
| 引数 | 型 | 説明 |
| --------- | ------------ | ------------------------ |
| 第 1 引数 | int | ファイルディスクリプタ |
| 第 2 引数 | const void\* | 書き込むデータのポインタ |
| 第 3 引数 | size_t | 書き込むバイト数 |

第 2 引数が <strong>const void\*</strong> なのは、write() が渡されたデータを変更しないことを保証するためです

read() の第 2 引数は void\*（const なし）ですが、これは読み取ったデータをバッファに書き込む必要があるためです

### [write() の戻り値](#write-return-value) {#write-return-value}

{: .labeled}
| 値 | 意味 |
| --- | ------------------------ |
| > 0 | 実際に書き込んだバイト数 |
| -1 | エラー |

### [注意点：部分的な読み書き](#partial-read-write) {#partial-read-write}

read() や write() は、要求したバイト数より少ない量を読み書きすることがあります

```c
/* 100 バイト読もうとしても、50 バイトしか読めないこともある */
ssize_t n = read(fd, buffer, 100);  /* n は 50 かもしれない */
```

これは正常な動作です

全データを読み書きするには、ループで繰り返す必要があります

---

## [close() の重要性](#importance-of-close) {#importance-of-close}

### [基本的な使い方](#basic-usage-close) {#basic-usage-close}

close() は、ファイルディスクリプタを解放します

```c
close(fd);
```

### [なぜ close() が必要か](#why-close-is-needed) {#why-close-is-needed}

ファイルディスクリプタは有限の<strong>リソース</strong>です

<strong>リソース</strong>とは、プログラムが使用できる OS の資源（メモリ、ファイル、ネットワーク接続など）のことです

プロセスが開けるファイル数には上限があります（デフォルトでは 1024 程度、RLIMIT_NOFILE で制御）

close() しないと、以下の問題が起きます

- ファイルディスクリプタが枯渇し、新しいファイルを開けなくなる
- 書き込んだデータがディスクに反映されないことがある
- 他のプロセスがファイルにアクセスできないことがある

### [リソースリークを防ぐ](#preventing-resource-leaks) {#preventing-resource-leaks}

<strong>リソースリーク</strong>とは、使い終わったリソースを解放し忘れることです

水道の蛇口を閉め忘れるようなもので、リソースが無駄に消費され続けます

```c
int fd = open("file.txt", O_RDONLY);
if (fd == -1) {
    perror("open");
    return 1;
}

/* ファイルを使う処理 */

close(fd);  /* 必ず閉じる */
```

エラーが発生した場合も、close() を忘れないようにしましょう

---

## [lseek() による位置制御](#position-control-with-lseek) {#position-control-with-lseek}

### [基本的な使い方](#basic-usage-lseek) {#basic-usage-lseek}

lseek() は、読み書き位置（オフセット）を移動します

```c
#include <unistd.h>

off_t new_position = lseek(fd, offset, whence);
```

### [引数の説明](#lseek-argument-description) {#lseek-argument-description}

{: .labeled}
| 引数 | 型 | 説明 |
| --------- | ----- | ---------------------- |
| 第 1 引数 | int | ファイルディスクリプタ |
| 第 2 引数 | off_t | 移動量（バイト数） |
| 第 3 引数 | int | 基準位置 |

<strong>off_t</strong> は、ファイル内の位置（オフセット）を表す型です

大きなファイルを扱えるように、通常の int より大きな値を格納できます

### [基準位置（whence）](#reference-position-whence) {#reference-position-whence}

{: .labeled}
| 値 | 説明 |
| -------- | -------------------------------------------- |
| SEEK_SET | ファイルの先頭を基準に offset バイト目へ移動 |
| SEEK_CUR | 現在の位置を基準に offset バイト移動 |
| SEEK_END | ファイルの末尾を基準に offset バイト移動 |

SEEK_END で offset に正の値を指定すると、ファイル末尾を超えた位置に移動できます（その後データを書き込むと、間に「穴」ができます）

### [使用例](#usage-examples) {#usage-examples}

```c
/* ファイルの先頭に移動 */
lseek(fd, 0, SEEK_SET);

/* 現在位置から 10 バイト進む */
lseek(fd, 10, SEEK_CUR);

/* ファイルの末尾に移動 */
lseek(fd, 0, SEEK_END);

/* ファイルサイズを取得 */
off_t size = lseek(fd, 0, SEEK_END);
```

### [戻り値](#lseek-return-value) {#lseek-return-value}

{: .labeled}
| 値 | 意味 |
| ---- | ------------------------------------------------ |
| >= 0 | 新しい読み書き位置（ファイル先頭からのバイト数） |
| -1 | エラー |

---

## [dup() と dup2() によるリダイレクト](#redirect-with-dup-and-dup2) {#redirect-with-dup-and-dup2}

### [dup() の基本](#dup-basics) {#dup-basics}

dup() は、ファイルディスクリプタを複製します

```c
int new_fd = dup(old_fd);
```

新しい fd と古い fd は、同じファイルを指します

どちらで読み書きしても、同じオフセットが使われます

### [dup2() の基本](#dup2-basics) {#dup2-basics}

dup2() は、指定した番号に複製します

```c
dup2(old_fd, new_fd);
```

`new_fd` が既に開いていれば、まず閉じてから複製します

### [なぜ dup2() がアトミックでなければならないか](#why-dup2-must-be-atomic) {#why-dup2-must-be-atomic}

<strong>もし close() と dup() を別々に呼んでいたら？</strong>

```c
/* 危険なコード（マルチスレッド環境で問題が発生） */
close(STDOUT_FILENO);  /* fd 1 を閉じる */
/* ← この瞬間、別のスレッドが open() を呼ぶと fd 1 が割り当てられる！ */
dup(file_fd);          /* file_fd を複製（fd 1 ではない番号になる可能性） */
```

<strong>dup2() のアトミック性</strong>

dup2() は「閉じる」と「複製する」を<strong>アトミック</strong>（途中で中断されない一連の操作）に行います

```c
/* 安全なコード */
dup2(file_fd, STDOUT_FILENO);  /* アトミックに fd 1 を置き換え */
```

他のスレッドが割り込む隙間がないため、確実に目的の fd 番号に複製できます

### [dup2() の特殊なケース](#dup2-special-case) {#dup2-special-case}

`old_fd` と `new_fd` が同じ値の場合、dup2() は何もせずに `new_fd` をそのまま返します（エラーにはなりません）

```c
/* fd が 3 の場合、何もせず 3 を返す */
int result = dup2(fd, fd);  /* result は 3 */
```

これは意図的な仕様で、条件分岐なしで安全に dup2() を呼び出せるようになっています

### [リダイレクトの仕組み](#how-redirect-works) {#how-redirect-works}

シェルの `>` リダイレクトは、dup2() を使って実現されています

```bash
./program > output.txt
```

シェルは内部で以下のような処理をしています

```c
int file_fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
dup2(file_fd, STDOUT_FILENO);  /* fd 1 を file_fd に置き換える */
close(file_fd);
execvp(program, args);  /* 別のプログラムを実行 */
```

execvp については [02-fork-exec](../02-fork-exec/) で学んでいます

このコードのポイントは「プログラム自身は出力先を知らない」ということです

プログラムは常に fd 1（標準出力）に write() しているだけで、その先が画面なのかファイルなのかを意識していません

シェルが裏で fd 1 の接続先をすり替えることで、出力先が変わります

各ステップで fd の状態がどう変化するかを見てみましょう

<strong>ステップ 1：ファイルを開く</strong>

```c
int file_fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
/* file_fd には例えば 3 が返る */
```

```
fd 0 → キーボード（標準入力）
fd 1 → 画面（標準出力）        ← プログラムはここに書く
fd 2 → 画面（標準エラー）
fd 3 → output.txt               ← 今開いたファイル
```

<strong>ステップ 2：fd 1 の接続先をすり替える</strong>

```c
dup2(file_fd, STDOUT_FILENO);  /* dup2(3, 1) */
```

dup2 は「fd 1 を閉じてから、fd 3 のコピーを fd 1 に作る」という操作です

fd 1 の接続先が画面から output.txt に変わります

```
fd 0 → キーボード
fd 1 → output.txt               ← すり替わった！
fd 2 → 画面
fd 3 → output.txt               ← まだ残っている
```

<strong>ステップ 3：不要な fd を閉じる</strong>

```c
close(file_fd);  /* close(3) */
```

fd 1 と fd 3 が両方 output.txt を指していますが、fd 3 はもう用済みです

fd を無駄に開いたままにしないために閉じます

```
fd 0 → キーボード
fd 1 → output.txt               ← これだけ残れば十分
fd 2 → 画面
```

<strong>ステップ 4：プログラムを実行する</strong>

```c
execvp(program, args);  /* 例えば ls を実行 */
```

exec でプロセスの中身が別のプログラムに置き換わります

しかし fd テーブルはそのまま引き継がれるので、プログラムが fd 1 に write() すると、それは output.txt に書き込まれます

プログラム自身は「自分の出力がファイルに向いている」ことを一切知りません

### [実践例](#practical-example) {#practical-example}

```c
/* 標準出力をファイルにリダイレクト */
int fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
dup2(fd, STDOUT_FILENO);
close(fd);

printf("This goes to file!\n");  /* ファイルに書き込まれる */
```

---

## [fork() と fd の継承](#fork-and-fd-inheritance) {#fork-and-fd-inheritance}

### [fd は子プロセスに継承される](#fd-inherited-by-child) {#fd-inherited-by-child}

02-fork-exec で学んだように、fork() は子プロセスを作成します

このとき、親が開いているファイルディスクリプタは子にコピーされます

<strong>pid_t</strong> は、プロセス ID を格納するための型です

詳しくは [01-process](../01-process/) を参照してください

```c
int fd = open("file.txt", O_RDWR);
pid_t pid = fork();

if (pid == 0) {
    /* 子プロセス：fd は親と同じファイルを指す */
    write(fd, "Child\n", 6);
} else {
    /* 親プロセス */
    write(fd, "Parent\n", 7);
}
```

### [オフセットの共有](#offset-sharing) {#offset-sharing}

親子プロセスは、同じファイルテーブルエントリを共有します

そのため、<strong>オフセット（読み書き位置）も共有</strong>されます

```c
/* 親が 10 バイト書いた後、子が書くと 10 バイト目から始まる */
```

これは、パイプ（07-ipc で学習）で親子間通信を行うときに重要です

### [02-fork-exec との関連](#relation-to-fork-exec) {#relation-to-fork-exec}

[02-fork-exec](../02-fork-exec/) で学んだように、exec() 後も fd は維持されます

この性質のおかげで、シェルはリダイレクトを設定してから別のプログラムを実行できます

### [スレッドとの違い（04-thread との関連）](#difference-from-threads) {#difference-from-threads}

04-thread で学んだスレッドでも fd は共有されますが、仕組みが異なります

<strong>スレッドの場合</strong>

- 同じプロセス内なので、fd テーブル自体を共有します
- 1 つのスレッドが fd を閉じると、他のスレッドも使えなくなります

<strong>fork() の場合</strong>

- 子プロセスは親の fd テーブルを<strong>コピー</strong>します
- 親子は別々の fd テーブルを持ちます
- しかし、同じファイルテーブルエントリを指しています
- そのため、オフセットは共有されますが、close() は独立しています

{: .labeled}
| 項目 | スレッド | fork() |
| ----------- | ------------ | ------ |
| fd テーブル | 共有 | コピー |
| オフセット | 共有 | 共有 |
| close() | 他に影響する | 独立 |

---

## [次のステップ](#next-steps) {#next-steps}

このトピックでは、「OS のファイル管理機構」を学びました

- ファイルディスクリプタとは何か
- open()、read()、write()、close() の使い方
- dup2() によるリダイレクトの仕組み
- fork() 後の fd 継承

次の [06-stdio](../06-stdio/) では、標準入出力ライブラリを学びます

- printf() や scanf() は内部で何をしているか
- fopen() と open() の違い
- バッファリングの仕組み

また、fd の知識は [07-ipc](../07-ipc/)（プロセス間通信）でも重要です

- パイプは fd のペアで実現されています
- pipe() システムコールで 2 つの fd（読み取り用と書き込み用）が作成されます
- fd 継承を使って、親子プロセス間でパイプを共有します

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 英語 | 説明 |
| ---------------------- | --------------- | ------------------------------------------------ |
| ファイルディスクリプタ | File Descriptor | 開いているファイルを識別する整数値 |
| 標準入力 | Standard Input | fd 0、通常はキーボード入力 |
| 標準出力 | Standard Output | fd 1、通常は画面出力 |
| 標準エラー出力 | Standard Error | fd 2、エラーメッセージ用 |
| オフセット | Offset | ファイル内の読み書き位置 |
| フラグ | Flag | ファイルの開き方を指定するビット値 |
| パーミッション | Permission | ファイルのアクセス権限 |
| リダイレクト | Redirect | 入出力先を変更すること |
| inode | Index Node | ファイルの実体情報を持つデータ構造 |
| EOF | End Of File | ファイルの終端 |
| リソース | Resource | プログラムが使用できる OS の資源 |
| リソースリーク | Resource Leak | 使い終わったリソースを解放し忘れること |
| ハンドル | Handle | リソースを操作するための識別子 |
| バッファ | Buffer | データを一時的に格納する領域 |
| バッファリング | Buffering | データを溜めてまとめて処理する仕組み |
| ソケット | Socket | ネットワーク通信のための接続口 |
| パイプ | Pipe | プロセス間でデータをやり取りする仕組み |
| ディレクトリ | Directory | ファイルを整理するためのフォルダ |
| デバイス | Device | ハードウェアを抽象化してファイルとして扱えるもの |
| カーネル | Kernel | OS の中核部分 |
| エントリ | Entry | テーブル（表）の中の 1 行分のデータ |
| 可読性 | Readability | コードの読みやすさ |
| mode_t | - | ファイルのパーミッションを表す型 |
| pid_t | - | プロセス ID を格納する型 |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

- [open(2) - Linux manual page](https://man7.org/linux/man-pages/man2/open.2.html){:target="\_blank"}
  - ファイルを開く
- [close(2) - Linux manual page](https://man7.org/linux/man-pages/man2/close.2.html){:target="\_blank"}
  - ファイルを閉じる
- [read(2) - Linux manual page](https://man7.org/linux/man-pages/man2/read.2.html){:target="\_blank"}
  - ファイルから読み取る
- [write(2) - Linux manual page](https://man7.org/linux/man-pages/man2/write.2.html){:target="\_blank"}
  - ファイルに書き込む
- [lseek(2) - Linux manual page](https://man7.org/linux/man-pages/man2/lseek.2.html){:target="\_blank"}
  - 読み書き位置を移動する
- [dup(2) - Linux manual page](https://man7.org/linux/man-pages/man2/dup.2.html){:target="\_blank"}
  - ファイルディスクリプタを複製する
- [dup2(2) - Linux manual page](https://man7.org/linux/man-pages/man2/dup2.2.html){:target="\_blank"}
  - 指定した番号にファイルディスクリプタを複製する
