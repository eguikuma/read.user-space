---
layout: default
title: なぜfork後は\_exitなのか
---

# [なぜfork後は\_exitなのか](#why-exit-after-fork) {#why-exit-after-fork}

## [はじめに](#introduction) {#introduction}

[02-fork-exec](../../02-fork-exec/) で、fork() と exec() を使ったプロセス生成を学びました

子プロセスを終了させるとき、`exit()` と `_exit()` のどちらを使うべきでしょうか？

```c
pid_t pid = fork();
if (pid == 0) {
    /*
     * 子プロセス
     * ここで終了するとき、exit() と _exit() どちらを使うべき？
     */
}
```

結論から言うと、<strong>fork() 後の子プロセスでは \_exit() を使うべき</strong>です

`exit()` を使うと、親子で stdio バッファが二重にフラッシュされたり、atexit() で登録したクリーンアップ処理が二重に実行されたりする問題が発生します

---

## [目次](#table-of-contents) {#table-of-contents}

- [exit()の動作](#how-underscore-exit-works)
- [\_exit()の動作](#how-underscore-exit-works)
- [fork後の問題](#problems-after-fork)
- [使い分け](#when-to-use-which)
- [まとめ](#summary)
- [参考資料](#references)

---

## [exit()の動作](#how-exit-works) {#how-exit-works}

### [exit()とは](#what-is-exit) {#what-is-exit}

`exit()` は、C 標準ライブラリが提供するプロセス終了関数です

```c
#include <stdlib.h>

void exit(int status);
```

### [終了処理の流れ](#exit-process-flow) {#exit-process-flow}

`exit()` を呼び出すと、以下の処理が順番に実行されます

```
exit() の処理

1. atexit() / on_exit() で登録された関数を逆順に呼び出す
2. すべての stdio ストリームをフラッシュして閉じる
3. tmpfile() で作成された一時ファイルを削除する
4. _exit() を呼び出してカーネルに制御を渡す
```

### [atexit()とは](#what-is-atexit) {#what-is-atexit}

`atexit()` は、プログラム終了時に呼び出される関数を登録する仕組みです

```c
#include <stdlib.h>

int atexit(void (*function)(void));
```

登録された関数は、`exit()` が呼ばれたとき、または `main()` から return したときに実行されます

```c
#include <stdio.h>
#include <stdlib.h>

void cleanup(void)
{
    printf("クリーンアップ処理を実行\n");
}

int main(void)
{
    atexit(cleanup);
    printf("プログラム終了\n");
    exit(0);
}
```

実行結果

```
プログラム終了
クリーンアップ処理を実行
```

### [stdioバッファのフラッシュ](#stdio-buffer-flush) {#stdio-buffer-flush}

`printf()` などの stdio 関数は、出力を<strong>バッファリング</strong>します

<strong>バッファリング</strong>とは、データを一時的にメモリに溜めておき、まとめて出力する仕組みです

```
バッファリングの仕組み

/*
 * バッファに格納される
 */
printf("Hello");
/*
 * バッファに追加される
 */
printf("World");
/*
 * ...
 */
/*
 * ここでバッファの内容が実際に出力される
 */
exit(0);
```

`exit()` は、このバッファの内容を確実に出力（フラッシュ）してからプロセスを終了します

---

## [\_exit()の動作](#how-underscore-exit-works) {#how-underscore-exit-works}

### [\_exit()とは](#what-is-underscore-exit) {#what-is-underscore-exit}

`_exit()` は、システムコールとしてカーネルに直接終了を要求する関数です

```c
#include <unistd.h>

void _exit(int status);
```

C99 以降では `<stdlib.h>` の `_Exit()` も同じ動作をします

### [即座にカーネルへ](#immediate-kernel-exit) {#immediate-kernel-exit}

`_exit()` は、`exit()` のような終了処理を行いません

```
_exit() の処理

1. atexit() / on_exit() で登録された関数を呼び出さない
2. stdio ストリームをフラッシュしない
3. 即座にカーネルに制御を渡す
4. カーネルがファイルディスクリプタを閉じる
```

### [exit()との比較](#comparison-with-exit) {#comparison-with-exit}

{: .labeled}
| 処理 | exit() | \_exit() |
| -------------------------------- | ------ | ---------------------- |
| atexit() 登録関数の実行 | する | しない |
| stdio バッファのフラッシュ | する | しない |
| tmpfile() 一時ファイルの削除 | する | しない |
| ファイルディスクリプタのクローズ | する | する（カーネルが処理） |

### [なぜ 2 つの関数が存在するのか](#why-two-functions-exist) {#why-two-functions-exist}

exit() は C 標準ライブラリの関数で、「便利な後処理」を提供します

\_exit() はシステムコールで、「カーネルへ即座に終了を要求」します

<strong>なぜ exit() だけでは不十分なのか？</strong>

{: .labeled}
| 場面 | exit() の問題 |
| --------------------- | ------------------------------------------ |
| fork() 後の子プロセス | 親のクリーンアップ処理が二重実行される |
| シグナルハンドラ内 | atexit() が async-signal-safe でない可能性 |
| 異常終了時 | クリーンアップを飛ばして即座に終了したい |

\_exit() は「C ライブラリを経由せずに終了したい」場面のために存在します

---

## [fork後の問題](#problems-after-fork) {#problems-after-fork}

### [なぜexit()が危険なのか](#why-exit-is-dangerous) {#why-exit-is-dangerous}

fork() で子プロセスを作ると、親プロセスのメモリ空間がコピーされます

これには stdio バッファの内容や atexit() の登録情報も含まれます

### [問題1：二重フラッシュ](#problem-1-double-flush) {#problem-1-double-flush}

親プロセスの stdio バッファに「Hello」が溜まっている状態で fork() すると、子プロセスも同じ「Hello」をバッファに持ちます

```
fork() 時点でのバッファ

親プロセス：バッファに「Hello」
        ↓ fork()
子プロセス：バッファに「Hello」（コピー）
```

親子両方が `exit()` を呼ぶと、同じ内容が二度出力されます

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void)
{
    /*
     * 改行なし → バッファに溜まる
     */
    printf("Hello");

    pid_t pid = fork();
    if (pid == 0) {
        /*
         * 子プロセス
         */
        /*
         * ここでバッファの「Hello」がフラッシュされる
         */
        exit(0);
    }

    wait(NULL);
    /*
     * ここでも「Hello」がフラッシュされる
     */
    exit(0);
}
```

実行結果（問題あり）

```
HelloHello
```

「Hello」が二度出力されてしまいます

### [問題2：atexit()ハンドラの二重実行](#problem-2-double-atexit) {#problem-2-double-atexit}

atexit() で登録した関数も、親子両方で実行されます

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

void cleanup(void)
{
    printf("クリーンアップ実行（PID=%d）\n", getpid());
}

int main(void)
{
    atexit(cleanup);

    pid_t pid = fork();
    if (pid == 0) {
        /*
         * 子プロセス
         */
        /*
         * cleanup() が実行される
         */
        exit(0);
    }

    wait(NULL);
    /*
     * cleanup() がまた実行される
     */
    exit(0);
}
```

実行結果

```
クリーンアップ実行（PID=12346）
クリーンアップ実行（PID=12345）
```

データベース接続のクローズやロックファイルの削除などの処理が二重に実行されると、予期しない問題が発生する可能性があります

### [解決策：\_exit()を使う](#solution-use-underscore-exit) {#solution-use-underscore-exit}

子プロセスで `_exit()` を使えば、これらの問題を回避できます

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void)
{
    /*
     * 改行なし → バッファに溜まる
     */
    printf("Hello");

    pid_t pid = fork();
    if (pid == 0) {
        /*
         * 子プロセス
         */
        /*
         * バッファをフラッシュしない
         */
        _exit(0);
    }

    wait(NULL);
    /*
     * 改行を追加
     */
    printf("\n");
    /*
     * 親だけがフラッシュする
     */
    exit(0);
}
```

実行結果（正しい）

```
Hello
```

---

## [使い分け](#when-to-use-which) {#when-to-use-which}

### [\_exit()を使う場面](#when-to-use-underscore-exit) {#when-to-use-underscore-exit}

```
_exit() を使うべき場面

─── fork() 後の子プロセスで exec() が失敗したとき
─── fork() 後に exec() を呼ばずに終了するとき
─── 親プロセスのクリーンアップ処理を実行したくないとき
```

### [exit()を使う場面](#when-to-use-exit) {#when-to-use-exit}

```
exit() を使うべき場面

─── 通常のプログラム終了
─── main() から return する代わりに終了するとき
─── クリーンアップ処理を確実に実行したいとき
```

### [fork() + exec() パターン](#fork-exec-pattern) {#fork-exec-pattern}

典型的な fork() + exec() パターンでは、exec() が失敗した場合に `_exit()` を使います

```c
pid_t pid = fork();
if (pid == 0) {
    /*
     * 子プロセス
     */
    execvp(command, args);

    /*
     * exec() が失敗した場合のみここに到達
     */
    perror("exec");
    /*
     * exit() ではなく _exit() を使う
     */
    _exit(1);
}
```

exec() が成功した場合、子プロセスは新しいプログラムに置き換わります

新しいプログラムは独自の atexit() 登録とバッファを持つため、`exit()` を使っても問題ありません

### [判断フローチャート](#decision-flowchart) {#decision-flowchart}

```
fork() 後に子プロセスを終了する

exec() を呼ぶか？
├─ はい → exec() 成功後は exit() でも問題ない
│         exec() 失敗時は _exit() を使う
│
└─ いいえ → _exit() を使う
```

---

## [まとめ](#summary) {#summary}

{: .labeled}
| 関数 | 用途 | 特徴 |
| -------- | ------------------------- | --------------------------------- |
| exit() | 通常のプログラム終了 | atexit() 実行、バッファフラッシュ |
| \_exit() | fork() 後の子プロセス終了 | クリーンアップなし、即座に終了 |

<strong>覚えておくこと</strong>

- fork() 後の子プロセスでは `_exit()` を使う
- `exit()` を使うと、バッファの二重フラッシュや atexit() ハンドラの二重実行が起きる可能性がある
- exec() が成功した後は、新しいプログラムなので `exit()` を使っても問題ない

---

## [参考資料](#references) {#references}

<strong>Linux マニュアル</strong>

- [exit(3) - Linux manual page](https://man7.org/linux/man-pages/man3/exit.3.html){:target="\_blank"}
  - プロセスの正常終了、終了処理の詳細
- [\_exit(2) - Linux manual page](https://man7.org/linux/man-pages/man2/_exit.2.html){:target="\_blank"}
  - 即座にプロセスを終了するシステムコール
- [atexit(3) - Linux manual page](https://man7.org/linux/man-pages/man3/atexit.3.html){:target="\_blank"}
  - 終了時に呼び出される関数の登録
- [fork(2) - Linux manual page](https://man7.org/linux/man-pages/man2/fork.2.html){:target="\_blank"}
  - プロセスの複製

<strong>本編との関連</strong>

- [02-fork-exec](../../02-fork-exec/)
  - fork() と exec() の基本動作
