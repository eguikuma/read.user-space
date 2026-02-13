<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# なぜman 2 openなのか

## はじめに

本編の「参考資料」では、こんなリンクを見たことがあるでしょう

```
- [open(2) - Linux manual page](https://man7.org/linux/man-pages/man2/open.2.html)
- [printf(3) - Linux manual page](https://man7.org/linux/man-pages/man3/printf.3.html)
```

`open(2)` と `printf(3)` の括弧内の数字は何を意味しているのでしょうか

なぜ `2` と `3` で違うのでしょうか

このドキュメントでは、man ページの仕組みと読み方を説明します

---

## 目次

- [セクション番号とは](#セクション番号とは)
- [読み方](#読み方)
- [実践](#実践)
- [まとめ](#まとめ)
- [参考資料](#参考資料)

---

## セクション番号とは

### man ページはセクションに分かれている

UNIX のマニュアル（man ページ）は、内容の種類によって<strong>セクション</strong>に分類されています

括弧内の数字がセクション番号です

| セクション | 内容                     | 例                             |
| ---------- | ------------------------ | ------------------------------ |
| 1          | ユーザーコマンド         | ls, cat, grep                  |
| 2          | システムコール           | open, fork, read               |
| 3          | ライブラリ関数           | printf, malloc, pthread_create |
| 4          | 特殊ファイル（デバイス） | /dev/null, /dev/tty            |
| 5          | ファイルフォーマット     | /etc/passwd, /proc             |
| 6          | ゲーム                   | -                              |
| 7          | その他・概要             | signal(7), pthreads(7)         |
| 8          | システム管理コマンド     | mount, ifconfig                |

### なぜ 8 つのセクションに分かれているか

<strong>UNIX の設計思想</strong>

1970年代の UNIX では、マニュアルは紙の印刷物でした

大量のページを効率よく探すために、内容の種類でセクションを分けました

| セクション    | 対象ユーザー   |
| ------------- | -------------- |
| 1（コマンド） | 全てのユーザー |
| 2, 3（関数）  | プログラマ     |
| 8（管理）     | システム管理者 |

<strong>なぜ同じ名前が複数のセクションに存在するか</strong>

同じ名前でも、コマンドと関数では意味が違います

<strong>例：printf</strong>

```
printf(1) → シェルコマンドの printf
printf(3) → C 言語の printf() 関数
```

セクション番号があることで、どちらを指しているか明確になります

```bash
man 1 printf   # コマンドの man ページ
man 3 printf   # C 言語関数の man ページ
```

### 本編でよく見るセクション

| セクション | 内容                 | 本編での例                              |
| ---------- | -------------------- | --------------------------------------- |
| 2          | システムコール       | open(2), fork(2), read(2), write(2)     |
| 3          | ライブラリ関数       | printf(3), malloc(3), pthread_create(3) |
| 5          | ファイルフォーマット | proc(5)                                 |
| 7          | 概要                 | signal(7), pthreads(7), unix(7)         |

<strong>セクション 2 と 3 の違い</strong>

- セクション 2（システムコール）：OS（カーネル）に直接依頼する機能
- セクション 3（ライブラリ関数）：C 言語ライブラリが提供する関数

`open()` は OS に「ファイルを開いて」と依頼するので、セクション 2 です

`printf()` は C 言語ライブラリの関数なので、セクション 3 です

<strong>なぜこの区別が重要なのか</strong>

| 観点           | システムコール (2)                    | ライブラリ関数 (3)                 |
| -------------- | ------------------------------------- | ---------------------------------- |
| パフォーマンス | ユーザー/カーネル空間の切り替えコスト | カーネル呼び出しを減らす最適化あり |
| エラー処理     | errno が直接設定される                | 関数によってエラー通知方法が異なる |
| シグナル安全性 | 多くが async-signal-safe              | 多くが async-signal-safe でない    |

例えば、シグナルハンドラ内で使えるかどうかを判断するとき、この区別が重要になります

`write()` (2) はシグナルハンドラ内で使えますが、`printf()` (3) は使えません

---

## 読み方

### man ページの構成

man ページは標準的な構成を持っています

| セクション名 | 内容                                       |
| ------------ | ------------------------------------------ |
| NAME         | 名前と一行説明                             |
| SYNOPSIS     | 使い方（関数のプロトタイプ、必要なヘッダ） |
| DESCRIPTION  | 詳細な説明                                 |
| RETURN VALUE | 戻り値（成功時・失敗時）                   |
| ERRORS       | エラー時に設定される errno の値            |
| EXAMPLES     | 使用例（ある場合）                         |
| SEE ALSO     | 関連する man ページ                        |

### SYNOPSIS の読み方

SYNOPSIS は関数の「使い方」を示します

<strong>例：open(2) の SYNOPSIS</strong>

```c
#include <fcntl.h>

int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```

これを読むと、以下がわかります

- `#include <fcntl.h>` が必要
- 引数は 2 つまたは 3 つ
- 戻り値は `int`（ファイルディスクリプタ）

### RETURN VALUE と ERRORS

<strong>RETURN VALUE</strong>

成功時と失敗時の戻り値を説明しています

```
RETURN VALUE
    open() returns the new file descriptor (a nonnegative integer)
    on success.  On error, -1 is returned and errno is set to
    indicate the error.
```

- 成功：新しいファイルディスクリプタ（0 以上の整数）
- 失敗：-1 を返し、errno にエラーの種類を設定

<strong>ERRORS</strong>

失敗時に errno に設定される値の一覧です

```
ERRORS
    EACCES  The requested access to the file is not allowed...
    ENOENT  O_CREAT is not set and the named file does not exist...
```

エラーの原因を特定するときに参照します

### SEE ALSO

関連する man ページへのリンクです

```
SEE ALSO
    close(2), openat(2), read(2), write(2), ...
```

`open(2)` を読んだ後、`read(2)` や `write(2)` も読むと理解が深まります

---

## 実践

### よく使うコマンド

| コマンド         | 説明                                                   |
| ---------------- | ------------------------------------------------------ |
| `man open`       | open の man ページを表示（最初に見つかったセクション） |
| `man 2 open`     | セクション 2 の open を表示                            |
| `man 3 printf`   | セクション 3 の printf を表示                          |
| `man -k keyword` | キーワードで man ページを検索                          |
| `man -f name`    | 指定した名前の man ページ一覧を表示                    |

<strong>例：fork に関連する man ページを探す</strong>

```bash
$ man -k fork
fork (2)             - create a child process
vfork (2)            - create a child process and block parent
```

<strong>例：printf という名前の man ページを確認する</strong>

```bash
$ man -f printf
printf (1)           - format and print data
printf (3)           - formatted output conversion
```

### オンラインで読む

man コマンドがなくても、オンラインで読めます

<strong>man7.org</strong>

本編の参考資料で使用しているサイトです

```
https://man7.org/linux/man-pages/man2/open.2.html
                                    ~~~~ ~~~~
                                    セクション番号
```

URL の構造がセクション番号に対応しています

---

## まとめ

| 項目         | 説明                                     |
| ------------ | ---------------------------------------- |
| セクション 2 | システムコール（OS に依頼する機能）      |
| セクション 3 | ライブラリ関数（C 言語の関数）           |
| セクション 7 | 概要・その他（signal の概要など）        |
| `man 2 open` | open() システムコールの man ページを表示 |
| SYNOPSIS     | 関数の使い方（ヘッダ、引数、戻り値の型） |
| RETURN VALUE | 成功/失敗時の戻り値                      |
| ERRORS       | エラー時の errno 値                      |

<strong>覚えておくこと</strong>

- `open(2)` の「2」はシステムコールを意味する
- `printf(3)` の「3」はライブラリ関数を意味する
- 同じ名前でも、セクション番号で区別できる
- SYNOPSIS を読めば、関数の使い方がわかる

---

## 参考資料

<strong>man コマンドとマニュアル構成</strong>

- [man(1) - Linux manual page](https://man7.org/linux/man-pages/man1/man.1.html)
  - man コマンド自体のマニュアル
- [man-pages(7) - Linux manual page](https://man7.org/linux/man-pages/man7/man-pages.7.html)
  - man ページの構成と慣習、セクション番号の定義

<strong>本編との関連</strong>

本編の各 README.md の「参考資料」セクションでは、以下の形式で man ページを参照しています

- [01-process](../01-process.md) → getpid(2), fork(2), proc(5) など
- [02-fork-exec](../02-fork-exec.md) → fork(2), execve(2), wait(2) など
- [03-signal](../03-signal.md) → signal(2), signal(7), sigaction(2) など
- [04-thread](../04-thread.md) → pthreads(7), pthread_create(3) など
- [05-file-descriptor](../05-file-descriptor.md) → open(2), read(2), write(2) など
- [06-stdio](../06-stdio.md) → stdio(3), fopen(3), printf(3) など
- [07-ipc](../07-ipc.md) → pipe(2), socket(2), unix(7) など

これらのリンクを辿ることで、各関数の詳細な仕様を確認できます
