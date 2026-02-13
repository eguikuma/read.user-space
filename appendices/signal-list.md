---
layout: default
title: SIGKILLとSIGTERMは何が違うのか
---

# [SIGKILLとSIGTERMは何が違うのか](#sigkill-vs-sigterm) {#sigkill-vs-sigterm}

## [はじめに](#introduction) {#introduction}

[03-signal](../../03-signal/) で、シグナルを使ってプロセスに通知する方法を学びました

ターミナルで `kill` コマンドを使ったことがあるかもしれません

```bash
kill 1234       # PID 1234 に終了を要求
kill -9 1234    # PID 1234 を強制終了
```

なぜ `-9` を付けると「強制」になるのでしょうか

`-9` は何を意味しているのでしょうか

このドキュメントでは、シグナルの違いと使い分けを説明します

---

## [目次](#table-of-contents) {#table-of-contents}

- [SIGTERMとSIGKILLの違い](#difference-between-sigterm-and-sigkill)
- [よく使うシグナル一覧](#commonly-used-signals-list)
- [デフォルト動作の分類](#default-action-classification)
- [シグナル番号について](#about-signal-numbers)
- [まとめ](#summary)
- [参考資料](#references)

---

## [SIGTERMとSIGKILLの違い](#difference-between-sigterm-and-sigkill) {#difference-between-sigterm-and-sigkill}

### [基本的な違い](#basic-differences) {#basic-differences}

{: .labeled}
| 項目 | SIGTERM | SIGKILL |
| -------- | ---------- | ------------ |
| 番号 | 15 | 9 |
| 意味 | 終了要求 | 強制終了 |
| 捕捉可能 | できる | できない |
| 無視可能 | できる | できない |
| 終了処理 | 実行できる | 実行できない |

<strong>捕捉可能</strong>とは、シグナルハンドラで処理できることを意味します

<strong>SIGTERM は「お願い」、SIGKILL は「命令」</strong>

SIGTERM を受け取ったプロセスは、終了処理（ファイルの保存、接続の切断など）を行ってから終了できます

SIGKILL を受け取ったプロセスは、何もせずに即座に終了させられます

### [なぜ2種類あるのか](#why-two-types) {#why-two-types}

プロセスには「終了前にやるべきこと」がある場合があります

- データベースへの書き込みを完了させる
- 一時ファイルを削除する
- ネットワーク接続を正しく閉じる
- ログに終了を記録する

SIGTERM を使えば、プロセスはこれらの処理を行ってから終了できます

しかし、プロセスが SIGTERM を無視したり、暴走して応答しなくなることがあります

そのような場合に、<strong>最後の手段</strong>として SIGKILL を使います

### [使い分けの指針](#guidelines-for-choosing) {#guidelines-for-choosing}

{: .labeled}
| 状況 | 使うシグナル |
| -------------------- | -------------------------------- |
| 通常の終了 | SIGTERM（または指定なしの kill） |
| SIGTERM に応答しない | SIGKILL（kill -9） |
| 完全に暴走している | SIGKILL |

<strong>推奨される手順</strong>

1. まず `kill <pid>` を試す（SIGTERM が送られる）
2. 数秒待つ
3. 終了しなければ `kill -9 <pid>` を使う（SIGKILL）

```bash
# 推奨手順
kill 1234
sleep 5
kill -9 1234  # まだ終了していなければ
```

### [コード例](#code-example) {#code-example}

SIGTERM を捕捉して終了処理を行う例：

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

volatile sig_atomic_t running = 1;

void handler(int signum) {
    (void)signum;
    running = 0;
}

int main(void) {
    struct sigaction sa;
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGTERM, &sa, NULL);

    while (running) {
        /* メインの処理 */
        sleep(1);
    }

    /* 終了処理 */
    printf("終了処理を実行中...\n");
    /* ファイルの保存、接続の切断など */

    return EXIT_SUCCESS;
}
```

SIGKILL はハンドラを登録できないため、上記のコードでも `kill -9` を送ると即座に終了します

---

## [よく使うシグナル一覧](#commonly-used-signals-list) {#commonly-used-signals-list}

### [プロセス終了系](#process-termination-signals) {#process-termination-signals}

{: .labeled}
| シグナル | 番号 | デフォルト動作 | 説明 |
| -------- | ---- | -------------- | --------------------------------------- |
| SIGTERM | 15 | 終了 | 終了要求（kill コマンドのデフォルト） |
| SIGKILL | 9 | 終了 | 強制終了（捕捉不可） |
| SIGINT | 2 | 終了 | Ctrl+C で送られる割り込み |
| SIGHUP | 1 | 終了 | 端末の切断、設定の再読み込み要求 |
| SIGQUIT | 3 | コアダンプ | Ctrl+\ で送られる終了（コアダンプ付き） |

<strong>SIGHUP の一見奇妙な二重の役割</strong>

SIGHUP は「HangUP（電話を切る）」の略で、もともとはモデム接続が切れたときに送られるシグナルでした

1970年代、UNIX は物理的な端末（テレタイプ）からシリアル回線で接続して使用していました

回線が切れると、そのセッションのプロセスは終了すべきです

しかし現代では、物理的な回線切断はほとんどありません

そこで SIGHUP は「設定ファイルの再読み込み」という新しい意味でも使われるようになりました

{: .labeled}
| 時代 | SIGHUP の意味 |
| -------- | -------------------------- |
| 1970年代 | 端末との接続が切れた |
| 現代 | 設定を再読み込みしてほしい |

nginx、Apache などのデーモンは、SIGHUP で設定を再読み込みします

<strong>SIGINT と SIGTERM の違い</strong>

どちらもプロセスを終了させますが、用途が異なります

- SIGINT：ユーザーが<strong>対話的に</strong>中断するとき（Ctrl+C）
- SIGTERM：プログラムや管理者が<strong>自動的に</strong>終了させるとき

### [プロセス制御系](#process-control-signals) {#process-control-signals}

{: .labeled}
| シグナル | 番号 | デフォルト動作 | 説明 |
| -------- | ---- | -------------- | ------------------------------- |
| SIGSTOP | 19 | 停止 | プロセスを一時停止（捕捉不可） |
| SIGCONT | 18 | 再開 | 停止したプロセスを再開 |
| SIGTSTP | 20 | 停止 | Ctrl+Z で送られる停止（捕捉可） |

<strong>SIGSTOP と SIGTSTP の違い</strong>

{: .labeled}
| 項目 | SIGSTOP | SIGTSTP |
| -------- | ------------------ | ------------ |
| 捕捉可能 | できない | できる |
| 送信元 | プログラム | Ctrl+Z |
| 用途 | 確実に停止させたい | 対話的な停止 |

### [子プロセス関連](#child-process-signals) {#child-process-signals}

{: .labeled}
| シグナル | 番号 | デフォルト動作 | 説明 |
| -------- | ---- | -------------- | ---------------------------------------- |
| SIGCHLD | 17 | 無視 | 子プロセスの状態変化（終了、停止、再開） |

SIGCHLD は<strong>デフォルトで無視</strong>されます

子プロセスの終了を検知したい場合は、明示的にハンドラを登録する必要があります

詳細は [03-signal](../../03-signal/) の「SIGCHLD と子プロセス管理」を参照してください

### [ユーザー定義](#user-defined-signals) {#user-defined-signals}

{: .labeled}
| シグナル | 番号 | デフォルト動作 | 説明 |
| -------- | ---- | -------------- | ---------------------- |
| SIGUSR1 | 10 | 終了 | ユーザー定義シグナル 1 |
| SIGUSR2 | 12 | 終了 | ユーザー定義シグナル 2 |

SIGUSR1 と SIGUSR2 は、アプリケーションが自由に意味を定義できるシグナルです

よくある使い方：

- 設定ファイルの再読み込み
- ログローテーション
- デバッグ情報の出力

```bash
# nginx に設定の再読み込みを要求（SIGHUP を使う例）
kill -HUP $(cat /var/run/nginx.pid)
```

### [パイプ・ソケット関連](#pipe-socket-signals) {#pipe-socket-signals}

{: .labeled}
| シグナル | 番号 | デフォルト動作 | 説明 |
| -------- | ---- | -------------- | -------------------------------- |
| SIGPIPE | 13 | 終了 | 読み手のいないパイプへの書き込み |

<strong>なぜ SIGPIPE でプロセスが終了するのか</strong>

パイプの読み手（受信側）が先に終了した場合、書き手（送信側）が write() しても誰も読みません

<strong>もし SIGPIPE がなかったら？</strong>

- write() がエラー（EPIPE）を返すだけ
- 多くのプログラムは write() の戻り値をチェックしない
- 無限に書き込み続ける可能性がある

SIGPIPE のデフォルト動作（終了）により、「誰も読まないデータを書き続ける」無駄を防いでいます

<strong>SIGPIPE を無視したい場合</strong>

ネットワークサーバーなど、接続切断を正常に処理したいプログラムでは、SIGPIPE を無視して EPIPE エラーで処理することがあります

```c
signal(SIGPIPE, SIG_IGN);  /* SIGPIPE を無視 */
/* write() が EPIPE を返すようになる */
```

### [エラー系](#error-signals) {#error-signals}

{: .labeled}
| シグナル | 番号 | デフォルト動作 | 説明 |
| -------- | ---- | -------------- | ---------------------------------- |
| SIGSEGV | 11 | コアダンプ | 不正なメモリアクセス |
| SIGBUS | 7 | コアダンプ | バスエラー（アライメント違反など） |
| SIGFPE | 8 | コアダンプ | 算術エラー（ゼロ除算など） |
| SIGABRT | 6 | コアダンプ | abort() 関数の呼び出し |
| SIGILL | 4 | コアダンプ | 不正な命令 |

これらのシグナルは通常、プログラムのバグによって発生します

<strong>SIGSEGV（セグメンテーション違反）</strong>は最もよく見るエラーシグナルです

```
Segmentation fault (core dumped)
```

---

## [デフォルト動作の分類](#default-action-classification) {#default-action-classification}

シグナルを受け取ったとき、ハンドラを登録していなければ<strong>デフォルト動作</strong>が実行されます

### [動作の種類](#types-of-actions) {#types-of-actions}

{: .labeled}
| 動作 | 英語 | 説明 |
| ---------- | ---- | -------------------------------- |
| 終了 | Term | プロセスを終了する |
| コアダンプ | Core | コアダンプを生成してから終了する |
| 停止 | Stop | プロセスを一時停止する |
| 再開 | Cont | 停止中のプロセスを再開する |
| 無視 | Ign | 何もしない |

### [動作ごとのシグナル分類](#signals-by-action) {#signals-by-action}

<strong>終了（Term）</strong>

SIGHUP, SIGINT, SIGKILL, SIGPIPE, SIGALRM, SIGTERM, SIGUSR1, SIGUSR2

<strong>コアダンプ（Core）</strong>

SIGQUIT, SIGILL, SIGABRT, SIGFPE, SIGSEGV, SIGBUS, SIGSYS, SIGTRAP, SIGXCPU, SIGXFSZ

<strong>停止（Stop）</strong>

SIGSTOP, SIGTSTP, SIGTTIN, SIGTTOU

<strong>再開（Cont）</strong>

SIGCONT

<strong>無視（Ign）</strong>

SIGCHLD, SIGURG, SIGWINCH

### [コアダンプとは](#what-is-core-dump) {#what-is-core-dump}

<strong>コアダンプ</strong>とは、プロセスが異常終了したときにメモリの内容をファイルに保存したものです

デバッグに使用できます

```bash
# コアダンプを有効にする
ulimit -c unlimited

# プログラムを実行（異常終了するとコアダンプが生成される）
./my_program

# コアダンプをデバッグ
gdb ./my_program core
```

---

## [シグナル番号について](#about-signal-numbers) {#about-signal-numbers}

### [なぜ番号があるのか](#why-signal-numbers-exist) {#why-signal-numbers-exist}

シグナルは内部的には<strong>整数値</strong>で管理されています

名前（SIGTERM など）は、人間が読みやすくするためのマクロです

```c
/* signal.h での定義（Linux x86/ARM の例）*/
#define SIGHUP     1
#define SIGINT     2
#define SIGQUIT    3
#define SIGKILL    9
#define SIGTERM   15
```

### [番号の調べ方](#how-to-find-signal-numbers) {#how-to-find-signal-numbers}

`kill -l` コマンドでシグナルの一覧を表示できます

```bash
$ kill -l
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL
 5) SIGTRAP      6) SIGABRT      7) SIGBUS       8) SIGFPE
 9) SIGKILL     10) SIGUSR1     11) SIGSEGV     12) SIGUSR2
13) SIGPIPE     14) SIGALRM     15) SIGTERM     ...
```

特定のシグナル番号から名前を調べる：

```bash
$ kill -l 9
KILL
```

特定のシグナル名から番号を調べる：

```bash
$ kill -l SIGTERM
15
```

### [アーキテクチャによる違い](#architecture-differences) {#architecture-differences}

<strong>注意</strong>：シグナル番号は CPU アーキテクチャによって異なる場合があります

{: .labeled}
| シグナル | x86/ARM | Alpha/SPARC | MIPS |
| -------- | ------- | ----------- | ---- |
| SIGCHLD | 17 | 20 | 18 |
| SIGSTOP | 19 | 17 | 23 |
| SIGCONT | 18 | 19 | 25 |

※ 主要なアーキテクチャのみ記載（PARISC など他のアーキテクチャでも番号が異なります）

そのため、コードでは番号ではなく<strong>名前</strong>を使います

```c
/* 良い例：名前を使う */
kill(pid, SIGTERM);

/* 悪い例：番号を直接使う */
kill(pid, 15);
```

---

## [まとめ](#summary) {#summary}

### [SIGTERM vs SIGKILL](#sigterm-vs-sigkill-summary) {#sigterm-vs-sigkill-summary}

{: .labeled}
| 項目 | SIGTERM | SIGKILL |
| -------- | ---------- | ---------- |
| 捕捉 | 可能 | 不可 |
| 終了処理 | 可能 | 不可 |
| 用途 | 通常の終了 | 最後の手段 |

### [覚えておくこと](#things-to-remember) {#things-to-remember}

- `kill` コマンドはデフォルトで SIGTERM を送る
- `kill -9` は SIGKILL を送る（強制終了）
- SIGKILL と SIGSTOP は捕捉できない
- シグナル番号は名前で参照する（移植性のため）
- デフォルト動作は Term, Core, Stop, Cont, Ign の5種類

### [よく使うシグナル](#commonly-used-signals-summary) {#commonly-used-signals-summary}

{: .labeled}
| シグナル | 番号 | 用途 |
| -------- | ---- | -------------------- |
| SIGTERM | 15 | 終了要求 |
| SIGKILL | 9 | 強制終了 |
| SIGINT | 2 | Ctrl+C |
| SIGSTOP | 19 | 一時停止 |
| SIGCONT | 18 | 再開 |
| SIGCHLD | 17 | 子プロセスの状態変化 |

※ 番号は x86/ARM アーキテクチャの場合（他のアーキテクチャでは異なる場合があります）

---

## [参考資料](#references) {#references}

<strong>Linux マニュアル</strong>

- [signal(7) - Linux manual page](https://man7.org/linux/man-pages/man7/signal.7.html){:target="\_blank"}
  - シグナルの概要、シグナル一覧、デフォルト動作
- [kill(1) - Linux manual page](https://man7.org/linux/man-pages/man1/kill.1.html){:target="\_blank"}
  - kill コマンドの使い方、シグナル一覧の表示（-l オプション）
- [kill(2) - Linux manual page](https://man7.org/linux/man-pages/man2/kill.2.html){:target="\_blank"}
  - kill() システムコール

<strong>本編との関連</strong>

- [03-signal](../../03-signal/)
  - シグナルの基本概念、ハンドラの登録方法、SIGCHLD の使い方
