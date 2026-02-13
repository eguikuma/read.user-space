---
layout: default
title: なぜシグナルハンドラには制限があるのか
---

# [なぜシグナルハンドラには制限があるのか](#why-signal-handler-has-restrictions) {#why-signal-handler-has-restrictions}

## [はじめに](#introduction) {#introduction}

[03-signal](../../03-signal/) で、シグナルハンドラを使ってプロセスに通知を受け取る方法を学びました

ハンドラの中で `printf()` を呼んだり、`malloc()` でメモリを確保したくなることがあるかもしれません

しかし、これらの関数をハンドラ内で呼ぶと、プログラムがクラッシュしたり、予測できない動作を引き起こすことがあります

このドキュメントでは、なぜシグナルハンドラには制限があるのか、そしてどのように対処すべきかを説明します

---

## [目次](#table-of-contents) {#table-of-contents}

- [制限とは](#what-are-the-restrictions)
- [なぜ危険なのか](#why-dangerous)
- [安全な関数（async-signal-safe）](#async-signal-safe-functions)
- [対処法](#countermeasures)
- [volatile と sig_atomic_t](#volatile-and-sig-atomic-t)
- [まとめ](#summary)
- [参考資料](#references)

---

## [制限とは](#what-are-the-restrictions) {#what-are-the-restrictions}

シグナルハンドラ内では、<strong>呼び出せる関数に制限</strong>があります

### [呼び出してはいけない関数](#functions-not-to-call) {#functions-not-to-call}

以下の関数はシグナルハンドラ内で呼び出すと危険です

{: .labeled}
| 関数 | 理由 |
| ------------------- | -------------------------------- |
| printf(), fprintf() | 内部バッファを使用している |
| malloc(), free() | ヒープの管理データ構造を操作する |
| fopen(), fclose() | 内部状態を持つ |
| exit() | atexit() ハンドラを呼び出す |

これらの関数を<strong>非同期シグナル安全でない</strong>（async-signal-unsafe）関数と呼びます

### [シグナルハンドラ内でやってよいこと](#what-is-safe-in-signal-handler) {#what-is-safe-in-signal-handler}

- `volatile sig_atomic_t` 型の変数への読み書き
- 非同期シグナル安全な関数の呼び出し

詳細は後述の「安全な関数」セクションで説明します

---

## [なぜ危険なのか](#why-is-it-dangerous) {#why-is-it-dangerous}

### [非同期に割り込む](#asynchronous-interruption) {#asynchronous-interruption}

シグナルは<strong>いつでも</strong>プログラムに割り込む可能性があります

```
メインプログラムの実行
     │
     │ malloc() を実行中
     │ ヒープのデータ構造を更新している途中
     │
     ▼ ← ここでシグナルが発生！
     ┌─────────────────────────┐
     │ シグナルハンドラ        │
     │ malloc() を呼び出す     │ ← 危険！
     │ ヒープが壊れる可能性    │
     └─────────────────────────┘
     │
     ▼ メインプログラムに戻る
     │ ヒープが破壊されている
```

メインプログラムが `malloc()` でヒープを操作している<strong>途中</strong>でシグナルが来ると、ヒープのデータ構造が中途半端な状態になっています

この状態でハンドラ内から `malloc()` を呼ぶと、ヒープが破壊される可能性があります

### [グローバル状態の破壊](#global-state-corruption) {#global-state-corruption}

多くのライブラリ関数は<strong>グローバルな状態</strong>を内部に持っています

<strong>printf() の場合</strong>

`printf()` は効率のために内部バッファを使っています

```
printf("Hello, ") を呼んだ

内部バッファ: [H][e][l][l][o][,][ ][?][?][?]...
                                 ↑
                            書き込み位置

← ここでシグナルが発生、ハンドラで printf("World") を呼ぶ

内部バッファ: [H][e][l][l][o][,][ ][W][o][r][l][d]...
                                 ↑
                            書き込み位置が不整合

結果：出力が壊れる、またはクラッシュ
```

### [デッドロックの可能性](#deadlock-possibility) {#deadlock-possibility}

`printf()` など多くの関数は、マルチスレッド環境で安全に動作するために<strong>ロック</strong>を使っています

```
メインプログラム
     │
     │ printf() を呼び出す
     │ 内部でロックを取得
     │
     ▼ ← ここでシグナルが発生！
     ┌─────────────────────────┐
     │ シグナルハンドラ        │
     │ printf() を呼び出す     │
     │ 同じロックを取得しようと│
     │ する → 永遠に待機      │ ← デッドロック！
     └─────────────────────────┘
```

すでにロックを持っているスレッドが、同じロックをもう一度取得しようとすると、永遠に待機し続けます

これが<strong>デッドロック</strong>です

---

## [安全な関数（async-signal-safe）](#async-signal-safe-functions) {#async-signal-safe-functions}

### [async-signal-safe とは](#what-is-async-signal-safe) {#what-is-async-signal-safe}

<strong>非同期シグナル安全</strong>（async-signal-safe）な関数とは、シグナルハンドラ内から安全に呼び出せる関数のことです

これらの関数は以下の性質を持っています

- 再入可能（reentrant）である ─ 実行中に割り込まれて再度呼ばれても正しく動作する
- グローバルな状態を変更しない
- ロックを使用しない

### [再入可能（reentrant）とは](#what-is-reentrant) {#what-is-reentrant}

<strong>再入可能</strong>とは、「関数の実行中に割り込まれて、同じ関数が再度呼ばれても正しく動作する」性質です

<strong>再入可能な関数の条件</strong>

{: .labeled}
| 条件 | 理由 |
| -------------------------- | -------------------------------- |
| 静的変数を使わない | 前回の呼び出しの状態が残らない |
| グローバル変数を変更しない | 他の実行コンテキストに影響しない |
| ロックを取得しない | デッドロックが起きない |

<strong>なぜ strtok() は再入不可能か</strong>

```c
char *strtok(char *str, const char *delim);
```

strtok() は内部に「前回どこまで処理したか」を静的変数で記憶しています

```
1回目: strtok("hello,world", ",") → "hello" を返す
       内部状態: 次は "world" の位置を覚えている

シグナル発生 → ハンドラで strtok() を呼ぶ
       内部状態: 別の文字列の位置に上書きされる

戻った後: strtok(NULL, ",") → 壊れた結果を返す
```

再入可能なバージョンとして strtok_r() があります

```c
char *strtok_r(char *str, const char *delim, char **saveptr);
/* saveptr に状態を保存するため、再入可能 */
```

### [安全な関数の例](#examples-of-safe-functions) {#examples-of-safe-functions}

POSIX で定義されている主な async-signal-safe 関数：

<strong>システムコール（ファイル操作）</strong>

{: .labeled}
| 関数 | 説明 |
| ------- | ------------------------------ |
| read() | ファイルからデータを読み取る |
| write() | ファイルにデータを書き込む |
| open() | ファイルを開く |
| close() | ファイルディスクリプタを閉じる |
| lseek() | ファイル位置を移動する |

<strong>システムコール（プロセス操作）</strong>

{: .labeled}
| 関数 | 説明 |
| -------- | --------------------------------------- |
| \_exit() | プロセスを終了する（atexit は呼ばない） |
| fork() | プロセスを複製する |
| execve() | プログラムを実行する |
| kill() | シグナルを送信する |
| getpid() | プロセス ID を取得する |

<strong>シグナル関連</strong>

{: .labeled}
| 関数 | 説明 |
| ------------- | ---------------------------------- |
| signal() | シグナルハンドラを設定する |
| sigaction() | シグナルハンドラを設定する（推奨） |
| sigprocmask() | シグナルマスクを操作する |
| raise() | 自分自身にシグナルを送る |

### [なぜ write() は安全なのか](#why-write-is-safe) {#why-write-is-safe}

`write()` はシステムコールであり、カーネルに直接要求を送ります

- 内部バッファを持たない
- グローバルな状態を変更しない
- ユーザー空間のロックを使用しない

そのため、シグナルハンドラ内からでも安全に呼び出せます

```c
void handler(int signum) {
    const char msg[] = "シグナルを受信\n";
    write(STDERR_FILENO, msg, sizeof(msg) - 1);
}
```

<strong>注意</strong>：`printf()` の代わりに `write()` を使いますが、フォーマット機能はありません

### [完全な一覧の確認方法](#how-to-check-complete-list) {#how-to-check-complete-list}

async-signal-safe な関数の完全なリストは `signal-safety(7)` man ページで確認できます

```bash
man 7 signal-safety
```

または、オンラインで確認できます

- [signal-safety(7) - Linux manual page](https://man7.org/linux/man-pages/man7/signal-safety.7.html)

---

## [対処法](#countermeasures) {#countermeasures}

### [パターン1：フラグを立てるだけ](#pattern-1-set-flag-only) {#pattern-1-set-flag-only}

最も安全な方法は、シグナルハンドラでは<strong>フラグを立てるだけ</strong>にすることです

実際の処理はメインループで行います

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/**
 * シグナルを受け取ったかどうかを記録します
 *
 * volatile について
 * ─── コンパイラの最適化を抑制する修飾子です
 * ─── シグナルハンドラから値が変更される可能性があることを示します
 *
 * sig_atomic_t について
 * ─── シグナルハンドラ内で安全に読み書きできる整数型です
 * ─── アトミック（不可分）な操作が保証されています
 */
static volatile sig_atomic_t got_signal = 0;

/**
 * シグナルハンドラ
 * フラグを立てるだけで、それ以外の処理は行いません
 */
void handler(int signum) {
    (void)signum;
    got_signal = 1;
}

int main(void) {
    struct sigaction sa;
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGINT, &sa, NULL);

    printf("Ctrl+C を押してください...\n");

    /**
     * メインループでフラグを確認します
     * シグナルハンドラ内ではなく、ここで安全に処理を行います
     */
    while (!got_signal) {
        /* 通常の処理 */
        sleep(1);
    }

    /**
     * フラグが立ったらここで処理
     * printf() はメインの実行コンテキストなので安全
     */
    printf("シグナルを受信したので、終了します\n");

    return EXIT_SUCCESS;
}
```

### [パターン2：write() でメッセージを出力](#pattern-2-write-message) {#pattern-2-write-message}

デバッグ目的でハンドラ内からメッセージを出力したい場合は、`write()` を使います

```c
#include <signal.h>
#include <unistd.h>

void handler(int signum) {
    (void)signum;

    /**
     * write() は async-signal-safe なので安全に呼び出せます
     * printf() は使えないので、フォーマットはできません
     */
    const char msg[] = "SIGINT received\n";
    write(STDERR_FILENO, msg, sizeof(msg) - 1);
}
```

### [パターン3：self-pipe trick（発展）](#pattern-3-self-pipe-trick) {#pattern-3-self-pipe-trick}

より高度なパターンとして、<strong>self-pipe trick</strong>があります

パイプを作成し、シグナルハンドラでパイプに書き込み、メインループでパイプを監視します

```c
#include <signal.h>
#include <unistd.h>

static int pipe_fd[2];

void handler(int signum) {
    int saved_errno = errno;
    write(pipe_fd[1], &signum, sizeof(signum));
    errno = saved_errno;
}

int main(void) {
    pipe(pipe_fd);

    /* ... sigaction で handler を登録 ... */

    /**
     * メインループで pipe_fd[0] を select/poll/epoll で監視
     * シグナルが来たら pipe_fd[0] から読み取れる
     */
}
```

この方法は [07-ipc](../../07-ipc/) のパイプの知識が前提となります

---

## [volatile と sig_atomic_t](#volatile-and-sig-atomic-t) {#volatile-and-sig-atomic-t}

### [volatile とは](#what-is-volatile) {#what-is-volatile}

`volatile` はコンパイラに「この変数は外部から変更される可能性がある」と伝える修飾子です

コンパイラは通常、最適化のために変数の値をレジスタにキャッシュします

```c
/* volatile がない場合 */
int flag = 0;

while (!flag) {
    /* コンパイラは flag が変わらないと仮定して
     * 無限ループに最適化する可能性がある */
}
```

`volatile` を付けると、毎回メモリから値を読み直すようになります

```c
/* volatile がある場合 */
volatile int flag = 0;

while (!flag) {
    /* 毎回メモリから flag を読み直す
     * シグナルハンドラからの変更を検知できる */
}
```

### [sig_atomic_t とは](#what-is-sig-atomic-t) {#what-is-sig-atomic-t}

`sig_atomic_t` は、シグナルハンドラとメインプログラムの間で安全にやり取りできることが保証された整数型です

この型への読み書きは<strong>アトミック</strong>（不可分）に行われます

```c
#include <signal.h>

/**
 * 通常の int でも動くことが多いですが
 * sig_atomic_t を使うのが正式な方法です
 */
volatile sig_atomic_t flag = 0;
```

### [なぜ両方必要か](#why-both-are-needed) {#why-both-are-needed}

{: .labeled}
| 修飾子/型 | 役割 |
| ------------ | -------------------------------------------------- |
| volatile | コンパイラの最適化を抑制し、毎回メモリから読み直す |
| sig_atomic_t | 読み書きがアトミックであることを保証する |

両方を組み合わせて使用します

```c
static volatile sig_atomic_t signal_received = 0;
```

---

## [まとめ](#summary) {#summary}

### [シグナルハンドラでの制限](#restrictions-in-signal-handler) {#restrictions-in-signal-handler}

{: .labeled}
| できること | できないこと |
| ---------------------------------- | -------------------------------------- |
| volatile sig_atomic_t への読み書き | printf(), malloc() などの呼び出し |
| async-signal-safe 関数の呼び出し | 非同期シグナル安全でない関数の呼び出し |
| フラグを立てる | 複雑な処理 |

### [なぜ危険なのか](#why-dangerous) {#why-dangerous}

- シグナルは<strong>いつでも</strong>割り込む可能性がある
- 関数の<strong>内部状態</strong>が中途半端な状態で呼ばれる可能性がある
- <strong>デッドロック</strong>が発生する可能性がある

### [推奨される対処法](#recommended-countermeasures) {#recommended-countermeasures}

1. シグナルハンドラでは<strong>フラグを立てるだけ</strong>
2. 実際の処理は<strong>メインループ</strong>で行う
3. どうしてもハンドラ内で出力したい場合は <strong>write()</strong> を使う

### [覚えておくこと](#things-to-remember) {#things-to-remember}

- `printf()` はシグナルハンドラ内で呼んではいけない
- `write()` はシグナルハンドラ内から安全に呼べる
- `volatile sig_atomic_t` でフラグを管理する
- 安全な関数のリストは `signal-safety(7)` で確認できる

---

## [参考資料](#references) {#references}

<strong>Linux マニュアル</strong>

- [signal-safety(7) - Linux manual page](https://man7.org/linux/man-pages/man7/signal-safety.7.html){:target="\_blank"}
  - 非同期シグナル安全な関数の一覧と、シグナルハンドラ内で安全に呼べる関数についての詳細な解説
- [signal(7) - Linux manual page](https://man7.org/linux/man-pages/man7/signal.7.html){:target="\_blank"}
  - シグナルの概要、シグナル一覧、デフォルト動作

<strong>POSIX 標準</strong>

- [POSIX.1-2017 Signal Concepts](https://pubs.opengroup.org/onlinepubs/9699919799/functions/V2_chap02.html#tag_15_04){:target="\_blank"}
  - POSIX におけるシグナルの概念と非同期シグナル安全関数の一覧

<strong>本編との関連</strong>

- [03-signal](../../03-signal/)
  - シグナルの基本概念、ハンドラの登録方法
- [signal-list.md](../signal-list/)
  - よく使うシグナルの一覧と使い分け
