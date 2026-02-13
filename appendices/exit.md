<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# なぜfork後は\_exitなのか

## はじめに

[02-fork-exec](../02-fork-exec.md) で、fork() と exec() を使ったプロセス生成を学びました

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

## 目次

- [exit()の動作](#exitの動作)
- [\_exit()の動作](#_exitの動作)
- [fork後の問題](#fork後の問題)
- [使い分け](#使い分け)
- [まとめ](#まとめ)
- [参考資料](#参考資料)

---

## exit()の動作

### exit()とは

`exit()` は、C 標準ライブラリが提供するプロセス終了関数です

```c
#include <stdlib.h>

void exit(int status);
```

### 終了処理の流れ

`exit()` を呼び出すと、以下の処理が順番に実行されます

```
exit() の処理

1. atexit() / on_exit() で登録された関数を逆順に呼び出す
2. すべての stdio ストリームをフラッシュして閉じる
3. tmpfile() で作成された一時ファイルを削除する
4. _exit() を呼び出してカーネルに制御を渡す
```

### atexit()とは

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

### stdioバッファのフラッシュ

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

## \_exit()の動作

### \_exit()とは

`_exit()` は、システムコールとしてカーネルに直接終了を要求する関数です

```c
#include <unistd.h>

void _exit(int status);
```

C99 以降では `<stdlib.h>` の `_Exit()` も同じ動作をします

### 即座にカーネルへ

`_exit()` は、`exit()` のような終了処理を行いません

```
_exit() の処理

1. atexit() / on_exit() で登録された関数を呼び出さない
2. stdio ストリームをフラッシュしない
3. 即座にカーネルに制御を渡す
4. カーネルがファイルディスクリプタを閉じる
```

### exit()との比較

| 処理                             | exit() | \_exit()               |
| -------------------------------- | ------ | ---------------------- |
| atexit() 登録関数の実行          | する   | しない                 |
| stdio バッファのフラッシュ       | する   | しない                 |
| tmpfile() 一時ファイルの削除     | する   | しない                 |
| ファイルディスクリプタのクローズ | する   | する（カーネルが処理） |

### なぜ 2 つの関数が存在するのか

exit() は C 標準ライブラリの関数で、「便利な後処理」を提供します

\_exit() はシステムコールで、「カーネルへ即座に終了を要求」します

<strong>なぜ exit() だけでは不十分なのか？</strong>

| 場面                  | exit() の問題                              |
| --------------------- | ------------------------------------------ |
| fork() 後の子プロセス | 親のクリーンアップ処理が二重実行される     |
| シグナルハンドラ内    | atexit() が async-signal-safe でない可能性 |
| 異常終了時            | クリーンアップを飛ばして即座に終了したい   |

\_exit() は「C ライブラリを経由せずに終了したい」場面のために存在します

---

## fork後の問題

### なぜexit()が危険なのか

fork() で子プロセスを作ると、親プロセスのメモリ空間がコピーされます

これには stdio バッファの内容や atexit() の登録情報も含まれます

### 問題1：二重フラッシュ

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

### 問題2：atexit()ハンドラの二重実行

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

### 解決策：\_exit()を使う

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

## 使い分け

### \_exit()を使う場面

```
_exit() を使うべき場面

─── fork() 後の子プロセスで exec() が失敗したとき
─── fork() 後に exec() を呼ばずに終了するとき
─── 親プロセスのクリーンアップ処理を実行したくないとき
```

### exit()を使う場面

```
exit() を使うべき場面

─── 通常のプログラム終了
─── main() から return する代わりに終了するとき
─── クリーンアップ処理を確実に実行したいとき
```

### fork() + exec() パターン

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

### 判断フローチャート

```
fork() 後に子プロセスを終了する

exec() を呼ぶか？
├─ はい → exec() 成功後は exit() でも問題ない
│         exec() 失敗時は _exit() を使う
│
└─ いいえ → _exit() を使う
```

---

## まとめ

| 関数     | 用途                      | 特徴                              |
| -------- | ------------------------- | --------------------------------- |
| exit()   | 通常のプログラム終了      | atexit() 実行、バッファフラッシュ |
| \_exit() | fork() 後の子プロセス終了 | クリーンアップなし、即座に終了    |

<strong>覚えておくこと</strong>

- fork() 後の子プロセスでは `_exit()` を使う
- `exit()` を使うと、バッファの二重フラッシュや atexit() ハンドラの二重実行が起きる可能性がある
- exec() が成功した後は、新しいプログラムなので `exit()` を使っても問題ない

---

## 参考資料

<strong>Linux マニュアル</strong>

- [exit(3) - Linux manual page](https://man7.org/linux/man-pages/man3/exit.3.html)
  - プロセスの正常終了、終了処理の詳細
- [\_exit(2) - Linux manual page](https://man7.org/linux/man-pages/man2/_exit.2.html)
  - 即座にプロセスを終了するシステムコール
- [atexit(3) - Linux manual page](https://man7.org/linux/man-pages/man3/atexit.3.html)
  - 終了時に呼び出される関数の登録
- [fork(2) - Linux manual page](https://man7.org/linux/man-pages/man2/fork.2.html)
  - プロセスの複製

<strong>本編との関連</strong>

- [02-fork-exec](../02-fork-exec.md)
  - fork() と exec() の基本動作
