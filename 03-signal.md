---
layout: default
title: プロセスに通知する
---

# [03-signal：プロセスに通知する](#notifying-processes) {#notifying-processes}

## [はじめに](#introduction) {#introduction}

前のトピック（[02-fork-exec](../02-fork-exec/)）で、<strong>ゾンビプロセス</strong>について学びました

<strong>ゾンビプロセス</strong>とは、終了したが親プロセスがまだ回収していないプロセスのことです

子プロセスが終了しても、親が wait() を呼ばないとゾンビになります

<strong>wait()</strong> は、子プロセスの終了を待ち、終了したプロセスを回収する<strong>システムコール</strong>です

<strong>システムコール</strong>とは、プログラムが OS に「これをやって」とお願いする仕組みです

でも、親はいつ wait() を呼べばいいのでしょうか？

子がいつ終了するかわからない場合、ずっと wait() で待ち続けるのは効率が悪いです

ここで登場するのが<strong>シグナル</strong>です

子プロセスの状態が変化すると、親に<strong>SIGCHLD</strong>（シグ・チャイルド）というシグナルが送られます

SIGCHLD は「子プロセスの状態が変化した」ことを通知するシグナルです

状態変化とは、子プロセスの「終了」「停止」「再開」のいずれかを指します

親はこのシグナルを「待ち受け」て、シグナルが届いたら wait() を呼ぶことで、効率的にゾンビを防げます

### [日常の例え](#everyday-analogy) {#everyday-analogy}

シグナルは、「誰かに肩を叩かれる」ようなものです

仕事に集中しているとき、肩を叩かれると振り返りますよね

それと同じで、プロセスも実行中にシグナルを受け取ると、今やっていることを中断して対応します

叩き方にも種類があります

軽く叩けば「確認してほしい」、強く叩けば「すぐ止まれ」、激しく揺さぶれば「強制終了」といった具合です

### [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

<strong>システムコール</strong>とは、プログラムが OS に「これをやって」とお願いする仕組みです

このページでは、以下のシステムコールを学びます

- <strong>signal()</strong>
  - シグナルハンドラを登録する（簡易版）
- <strong>sigaction()</strong>
  - シグナルハンドラを登録する（堅牢版）
- <strong>kill()</strong>
  - プロセスにシグナルを送る
- <strong>raise()</strong>
  - 自分自身にシグナルを送る
- <strong>pause()</strong>
  - シグナルが届くまで待つ

---

## [目次](#table-of-contents) {#table-of-contents}

1. [シグナルとは何か](#what-is-a-signal)
2. [よく使うシグナル](#commonly-used-signals)
3. [シグナルのデフォルト動作](#default-signal-behavior)
4. [シグナルハンドラとは](#what-is-signal-handler)
5. [signal() によるハンドラ登録](#registering-handler-with-signal)
6. [sigaction() によるハンドラ登録](#registering-handler-with-sigaction)
7. [シグナルを送る方法](#how-to-send-signals)
8. [シグナルを待つ方法](#how-to-wait-for-signals)
9. [SIGCHLD と子プロセス管理](#sigchld-and-child-process-management)
10. [シグナルの制限事項](#signal-limitations)
11. [次のステップ](#next-steps)
12. [用語集](#glossary)
13. [参考資料](#references)

---

## [シグナルとは何か](#what-is-a-signal) {#what-is-a-signal}

### [なぜシグナルが必要なのか](#why-signals-are-needed) {#why-signals-are-needed}

<strong>もしシグナルがなかったら？</strong>

プロセスは「何か起きたか」を自分で繰り返し確認する必要があります

これを<strong>ポーリング</strong>と呼びます

```c
/* ポーリングの例（非効率） */
while (1) {
    if (check_if_child_terminated()) {
        handle_termination();
    }
    sleep(1);  /* 1秒待って再チェック */
}
```

<strong>ポーリングの問題点</strong>

{: .labeled}
| 問題 | 説明 |
| -------- | ---------------------------------------------- |
| CPU 消費 | 何も起きていなくても繰り返しチェックする |
| 応答遅延 | sleep() の間隔だけ反応が遅れる |
| 複雑化 | すべてのイベントを自分でチェックする必要がある |

<strong>シグナルによる解決</strong>

シグナルは「何か起きたときにプロセスに通知する」仕組みです

プロセスは自分から確認しに行く必要がなく、OS が通知してくれます

これにより、CPU を無駄に消費せず、即座に反応できます

### [基本的な説明](#basic-explanation) {#basic-explanation}

<strong>シグナル</strong>は、プロセスに送られる<strong>非同期の通知</strong>です

Linux の公式マニュアルには、こう書かれています

> Signals are a limited form of inter-process communication (IPC).

> シグナルは、プロセス間通信の限定的な形式です

<strong>プロセス間通信（IPC：Inter-Process Communication）</strong>とは、複数のプロセスがデータをやり取りする仕組みの総称です

詳しくは [07-ipc](../07-ipc/) で学びます

### [シグナルで何ができるか](#what-can-signals-do) {#what-can-signals-do}

シグナルは以下のような用途で使われます

- プロセスを終了させる（SIGTERM、SIGKILL）
- プロセスを一時停止・再開させる（SIGSTOP、SIGCONT）
- 子プロセスの状態変化を親に通知する（SIGCHLD）
- ユーザーの割り込み操作を伝える（SIGINT = Ctrl+C）

### [非同期とは](#what-is-asynchronous) {#what-is-asynchronous}

<strong>非同期</strong>とは、「いつ発生するか予測できない」という意味です

プロセスは、いつシグナルが届くかわかりません

どのタイミングでも届く可能性があります

そのため、シグナルを受け取ったときの処理は慎重に設計する必要があります

---

## [よく使うシグナル](#commonly-used-signals) {#commonly-used-signals}

シグナルには番号と名前があります

よく使うシグナルを表にまとめます

{: .labeled}
| シグナル | 番号 | デフォルト動作 | 説明 |
| -------- | ---- | -------------- | ------------------------------------- |
| SIGINT | 2 | 終了 | Ctrl+C で送られる割り込みシグナル |
| SIGTERM | 15 | 終了 | 終了要求（kill コマンドのデフォルト） |
| SIGKILL | 9 | 終了 | 強制終了（ハンドル不可） |
| SIGCHLD | 17 | 無視 | 子プロセスの状態変化 |
| SIGSTOP | 19 | 停止 | プロセスを一時停止（ハンドル不可） |
| SIGCONT | 18 | 再開 | 停止したプロセスを再開 |
| SIGUSR1 | 10 | 終了 | ユーザー定義シグナル 1 |
| SIGUSR2 | 12 | 終了 | ユーザー定義シグナル 2 |
| SIGTSTP | 20 | 停止 | Ctrl+Z で送られる停止シグナル |
| SIGSEGV | 11 | コアダンプ | 不正なメモリアクセス |

※ 上記の番号は x86/ARM 系アーキテクチャの値であるため、他のアーキテクチャでは異なる場合があります

<strong>フォアグラウンド</strong>とは、ターミナルと対話中のプロセスのことです

Ctrl+C や Ctrl+Z は、フォアグラウンドで動いているプロセスに対して送られます

反対に、<strong>バックグラウンド</strong>はターミナルから切り離されて動作するプロセスです

### [シグナル番号について](#about-signal-numbers) {#about-signal-numbers}

シグナル番号は OS や<strong>アーキテクチャ</strong>によって異なる場合があります

<strong>アーキテクチャ</strong>とは、CPU の種類や設計のことです

例えば、x86（Intel や AMD の PC 向け CPU）と ARM（スマートフォンや Mac の CPU）では、シグナル番号が共通ですが、Alpha、SPARC、MIPS、PARISC などのアーキテクチャでは異なります

以下は signal(7) man page からの抜粋です

{: .labeled}
| シグナル | x86/ARM | Alpha/SPARC | MIPS | PARISC |
| -------- | ------- | ----------- | ---- | ------ |
| SIGCHLD | 17 | 20 | 18 | 18 |
| SIGCONT | 18 | 19 | 25 | 26 |
| SIGSTOP | 19 | 17 | 23 | 24 |
| SIGTSTP | 20 | 18 | 24 | 25 |

そのため、コードでは番号ではなく名前（SIGINT、SIGTERM など）を使います

---

## [シグナルのデフォルト動作](#default-signal-behavior) {#default-signal-behavior}

シグナルを受け取ったとき、ハンドラを登録していなければ<strong>デフォルト動作</strong>が実行されます

### [デフォルト動作の種類](#types-of-default-actions) {#types-of-default-actions}

{: .labeled}
| 動作 | 説明 |
| ------------------ | -------------------------------- |
| Term（終了） | プロセスを終了する |
| Core（コアダンプ） | コアダンプを生成してから終了する |
| Stop（停止） | プロセスを一時停止する |
| Cont（再開） | 停止中のプロセスを再開する |
| Ign（無視） | シグナルを無視する（何もしない） |

### [コアダンプとは](#what-is-core-dump) {#what-is-core-dump}

<strong>コアダンプ</strong>とは、プロセスが異常終了したときにメモリの内容をファイルに保存したものです

デバッグに使用できます

SIGSEGV（不正なメモリアクセス）などで発生します

---

## [シグナルハンドラとは](#what-is-signal-handler) {#what-is-signal-handler}

### [基本的な説明](#basic-explanation) {#basic-explanation}

<strong>シグナルハンドラ</strong>は、シグナルを受け取ったときに実行される関数です

デフォルト動作を上書きできます

### [ハンドラ関数の形式](#handler-function-format) {#handler-function-format}

```c
void handler(int signum) {
    /* シグナルを受け取ったときの処理 */
}
```

- 引数：受け取ったシグナル番号
- 戻り値：なし（void）

### [複数のシグナルで同じハンドラを使う](#using-same-handler-for-multiple-signals) {#using-same-handler-for-multiple-signals}

引数のシグナル番号を見て、処理を分岐できます

```c
void handler(int signum) {
    if (signum == SIGINT) {
        /* Ctrl+C の処理 */
    } else if (signum == SIGTERM) {
        /* 終了要求の処理 */
    }
}
```

---

## [signal() によるハンドラ登録](#registering-handler-with-signal) {#registering-handler-with-signal}

### [基本的な使い方](#basic-usage) {#basic-usage}

signal() は、シグナルハンドラを登録する最も簡単な方法です

```c
#include <signal.h>

signal(SIGINT, handler);  /* SIGINT のハンドラを登録 */
```

### [特別な値](#special-values) {#special-values}

{: .labeled}
| 値 | 意味 |
| ------- | -------------------- |
| SIG_DFL | デフォルト動作に戻す |
| SIG_IGN | シグナルを無視する |

```c
signal(SIGINT, SIG_IGN);  /* SIGINT を無視 */
signal(SIGINT, SIG_DFL);  /* SIGINT をデフォルト動作に戻す */
```

### [signal() の問題点](#problems-with-signal) {#problems-with-signal}

signal() にはいくつかの問題があります

- 一部の OS で、ハンドラ実行後に自動的にデフォルトに戻る
- シグナル処理中に同じシグナルが来たときの動作が不明確
- <strong>移植性</strong>に問題がある

<strong>移植性</strong>とは、異なる OS やコンピュータでも同じように動作することです

signal() は OS によって動作が異なるため、移植性が低いとされています

そのため、新しいコードでは sigaction() を使うことが推奨されています

---

## [sigaction() によるハンドラ登録](#registering-handler-with-sigaction) {#registering-handler-with-sigaction}

### [基本的な使い方](#basic-usage-sigaction) {#basic-usage-sigaction}

sigaction() は、signal() より堅牢なハンドラ登録関数です

```c
#include <signal.h>
#include <string.h>

struct sigaction sa;
memset(&sa, 0, sizeof(sa));
sa.sa_handler = handler;
sigemptyset(&sa.sa_mask);
sa.sa_flags = SA_RESTART;

sigaction(SIGINT, &sa, NULL);
```

<strong>memset()</strong> は、メモリ領域を指定した値で埋める関数です

ここでは構造体 sa のすべてのメンバーを 0 で初期化しています

<strong>sigemptyset()</strong> は、シグナル集合を空にする関数です

<strong>sigset_t</strong> 型は、シグナルの集合（どのシグナルを含むか）を表す型です

### [struct sigaction の主なメンバー](#main-members-of-sigaction) {#main-members-of-sigaction}

{: .labeled}
| メンバー | 説明 |
| ---------- | ------------------------------------------ |
| sa_handler | ハンドラ関数へのポインタ |
| sa_mask | ハンドラ実行中にブロックするシグナルの集合 |
| sa_flags | 動作を変更するフラグ |

シグナルを<strong>ブロック</strong>するとは、そのシグナルを一時的に受け取らないようにすることです

ブロックされたシグナルは<strong>保留</strong>され、ブロックが解除されたときに配送されます

sa_mask を設定すると、ハンドラ実行中に他のシグナルが割り込むのを防げます

### [よく使うフラグ](#commonly-used-flags) {#commonly-used-flags}

{: .labeled}
| フラグ | 説明 |
| ------------ | ---------------------------------------------------- |
| SA_RESTART | シグナルで中断されたシステムコールを自動的に再開する |
| SA_NOCLDSTOP | 子プロセスが停止したときには SIGCHLD を送らない |

### [signal() との違い](#differences-from-signal) {#differences-from-signal}

- ハンドラ実行後に自動リセットされない
- 動作が POSIX で明確に定義されている
- 追加オプションで細かい制御ができる

<strong>POSIX</strong>（Portable Operating System Interface）は、Unix 系 OS の標準仕様です

Linux、macOS、FreeBSD など、異なる OS 間で互換性を保つためのルールを定めています

### [なぜ signal() と sigaction() の2つが存在するのか](#why-two-functions-exist) {#why-two-functions-exist}

<strong>歴史的経緯</strong>があります

signal() は UNIX の初期から存在する古い<strong>API</strong>です

<strong>API</strong>（Application Programming Interface）とは、プログラムから機能を呼び出すための取り決めのことです

ここでは「関数の使い方」と考えて問題ありません

しかし、signal() の動作は OS ごとに異なっていました

- ハンドラ実行後にデフォルトに戻す OS
- ハンドラをそのまま維持する OS
- シグナル処理中に同じシグナルを受け取ったときの動作の違い

これらの問題を解決するために、POSIX 標準で<strong>sigaction()</strong> が導入されました

sigaction() は動作が明確に定義されており、すべての POSIX 準拠システムで同じ動作が保証されています

### [どちらを使うべきか](#which-to-use) {#which-to-use}

{: .labeled}
| 場面 | 推奨 |
| ------------------------ | -------------------- |
| 新規開発 | sigaction() |
| 既存コードのメンテナンス | 既存の方式に合わせる |
| 学習目的の簡単なデモ | signal() でも可 |
| 移植性が必要なコード | sigaction() |

signal() は以下の場合にのみ使用を検討してください

- 非常に単純な学習用コード
- 古いコードとの互換性が必要な場合

---

## [シグナルを送る方法](#how-to-send-signals) {#how-to-send-signals}

### [kill() 関数](#kill-function) {#kill-function}

kill() は、任意のプロセスにシグナルを送る関数です

```c
#include <signal.h>

kill(pid, SIGTERM);  /* pid のプロセスに SIGTERM を送る */
```

名前は「kill」ですが、終了させるだけではありません

任意のシグナルを送れます

### [raise() 関数](#raise-function) {#raise-function}

raise() は、自分自身にシグナルを送る関数です

```c
#include <signal.h>

raise(SIGTERM);  /* 自分自身に SIGTERM を送る */
```

raise() の動作は、プログラムがシングルスレッドかマルチスレッドかで異なります

- <strong>シングルスレッドプログラム</strong>：`kill(getpid(), sig)` と等価
- <strong>マルチスレッドプログラム</strong>：`pthread_kill(pthread_self(), sig)` と等価

マルチスレッドプログラムでは、raise() は呼び出したスレッド自身にシグナルを送ります

一方、`kill(getpid(), sig)` はプロセス全体にシグナルを送り、ブロックしていない任意のスレッドがシグナルを受け取る可能性があります

### [コマンドラインからシグナルを送る](#sending-signals-from-command-line) {#sending-signals-from-command-line}

シェルの kill コマンドでシグナルを送れます

```bash
kill -SIGTERM 1234    # PID 1234 に SIGTERM を送る
kill -15 1234         # 同じ（15 は SIGTERM の番号）
kill 1234             # デフォルトで SIGTERM が送られる
kill -9 1234          # SIGKILL（強制終了）を送る
```

---

## [シグナルを待つ方法](#how-to-wait-for-signals) {#how-to-wait-for-signals}

### [pause() 関数](#pause-function) {#pause-function}

pause() は、シグナルが届くまでプロセスを眠らせます

```c
#include <unistd.h>

pause();  /* シグナルが届くまで待機 */
```

CPU を使わずに待機できるので、効率的です

### [無限ループ + sleep() との違い](#difference-from-infinite-loop-sleep) {#difference-from-infinite-loop-sleep}

```c
/* 非効率な方法 */
while (!signal_received) {
    sleep(1);
}

/* 効率的な方法 */
while (!signal_received) {
    pause();
}
```

sleep() の場合、指定した秒数ごとにしかシグナルを検知できません

pause() は、シグナルが届いた瞬間に反応できます

### [pause() の注意点](#notes-on-pause) {#notes-on-pause}

pause() には<strong>競合状態（レースコンディション）</strong>の問題があります

```c
while (!signal_received) {  /* ← ここでチェック */
    pause();                 /* ← ここで待機 */
}
```

上のコードで、チェックと待機の間にシグナルが届くとどうなるでしょうか

1. `while (!signal_received)` でチェック → シグナルはまだ来ていない
2. <strong>この瞬間にシグナルが届く</strong>（ハンドラが実行され `signal_received = 1` になる）
3. `pause()` を呼ぶ → しかし、もうシグナルは来ない

結果として、pause() は永遠に戻らなくなる可能性があります

この問題を解決するには <strong>sigsuspend()</strong> を使います

<strong>シグナルマスク</strong>とは、現在ブロック（受け取らないように）しているシグナルの一覧です

sigsuspend() は、シグナルマスクの変更と待機を<strong>アトミック</strong>に行うため、レースコンディションが発生しません

<strong>アトミック</strong>とは、「途中で中断されずに一度に実行される」という意味です

チェックと待機が一体となって実行されるため、その間にシグナルが割り込む隙間がなくなります

```c
sigset_t mask, oldmask;

/**
 * シグナルをブロックします
 *
 * sigaddset() は、シグナル集合にシグナルを追加する関数です
 * sigprocmask() は、プロセスのシグナルマスクを変更する関数です
 */
sigemptyset(&mask);
sigaddset(&mask, SIGUSR1);
sigprocmask(SIG_BLOCK, &mask, &oldmask);

while (!signal_received) {
    /* ブロックを解除しつつ待機（アトミック） */
    sigsuspend(&oldmask);
}

/* 元のマスクに戻す */
sigprocmask(SIG_SETMASK, &oldmask, NULL);
```

sigsuspend() の詳細は [sigsuspend(2) - Linux manual page](https://man7.org/linux/man-pages/man2/sigsuspend.2.html) を参照してください

pause() は学習目的のシンプルな例です

実際のプログラムでは sigsuspend() を使用してください

---

## [SIGCHLD と子プロセス管理](#sigchld-and-child-process-management) {#sigchld-and-child-process-management}

### [ゾンビ問題の解決策](#solution-to-zombie-problem) {#solution-to-zombie-problem}

02-fork-exec で学んだゾンビ問題を、SIGCHLD で解決できます

子プロセスが終了すると、親に SIGCHLD が送られます

このシグナルをハンドルして waitpid() を呼べば、ゾンビを防げます

### [実装パターン](#implementation-pattern) {#implementation-pattern}

```c
void sigchld_handler(int signum) {
    (void)signum;

    pid_t pid;
    int status;

    /* 終了した子をすべて回収 */
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        /* 終了した子の処理 */
    }
}
```

<strong>waitpid() の引数</strong>

{: .labeled}
| 引数 | 値 | 説明 |
| --------- | ------- | ---------------------------------------- |
| 第 1 引数 | -1 | すべての子プロセスを対象にする |
| 第 2 引数 | &status | 終了ステータスを格納する変数のアドレス |
| 第 3 引数 | WNOHANG | 終了した子がいなければすぐに戻る（後述） |

第 1 引数に特定の PID を指定すると、その子プロセスだけを対象にできます

このコード例では waitpid() のみを呼んでいます

上記のコードでは waitpid() のみを呼んでいますが、学習目的で printf() を使う例もあります

ただし、本来はシグナルハンドラ内で printf() を呼ぶことは推奨されません（詳細は「シグナルの制限事項」を参照）

### [WNOHANG フラグ](#wnohang-flag) {#wnohang-flag}

WNOHANG を指定すると、waitpid() がブロックしません

終了した子がいなければ、すぐに 0 を返します

### [なぜループが必要か](#why-loop-is-needed) {#why-loop-is-needed}

複数の子が同時に終了した場合、SIGCHLD は 1 回しか届かないことがあります

ループで waitpid() を呼ぶことで、すべての子を確実に回収できます

---

## [シグナルの制限事項](#signal-limitations) {#signal-limitations}

### [ハンドルできないシグナル](#unhandleable-signals) {#unhandleable-signals}

以下のシグナルは、ハンドラを登録できません

{: .labeled}
| シグナル | 理由 |
| -------- | ---------------------------------- |
| SIGKILL | 暴走プロセスを確実に終了させるため |
| SIGSTOP | プロセスを確実に停止させるため |

これらは OS によって強制的に処理されます

### [なぜ SIGKILL は捕捉できないのか](#why-sigkill-cannot-be-caught) {#why-sigkill-cannot-be-caught}

<strong>もし SIGKILL をハンドルできたら？</strong>

```c
/* 仮想的なコード（実際には不可能） */
void sigkill_handler(int signum) {
    /* 何もしない = 終了させない */
}
signal(SIGKILL, sigkill_handler);  /* これはできない */
```

このようなコードが書けたら、以下の問題が発生します

- 無限ループに陥ったプロセスを終了できない
- 悪意のあるプログラムが終了を拒否できる
- システム管理者がプロセスを制御できない

<strong>「最後の手段」としての SIGKILL</strong>

通常、プロセスを終了させるには SIGTERM を使います

SIGTERM は捕捉できるため、プロセスはクリーンアップ処理（ファイルを閉じる、一時ファイルを削除するなど）を行えます

しかし、プロセスが SIGTERM を無視したり、ハンドラがハングしたりする場合があります

そのため、<strong>絶対に終了させる手段</strong>として SIGKILL が存在します

同様に、SIGSTOP は<strong>絶対に停止させる手段</strong>として存在します

### [非同期シグナル安全](#async-signal-safety) {#async-signal-safety}

シグナルハンドラ内で呼べる関数には制限があります

これを<strong>非同期シグナル安全（async-signal-safe）</strong>と呼びます

- write() は安全
- printf() は安全ではない
- malloc() は安全ではない

### [なぜ特定の関数が危険なのか](#why-certain-functions-are-dangerous) {#why-certain-functions-are-dangerous}

シグナルは「いつでも」届く可能性があります

もしメインプログラムが関数を実行中にシグナルが届き、ハンドラ内で同じ関数を呼ぶと、関数が途中から「再入」されることになります

<strong>malloc() が危険な具体例</strong>

```
メインプログラム                  シグナルハンドラ
─────────────────────────────────────────────────────
malloc() を呼ぶ
 ├─ 空きリストを検索中...
 │   ← ここでシグナルが届く
 │                                 malloc() を呼ぶ
 │                                  ├─ 空きリストを更新
 │                                  └─ 戻る
 ├─ 空きリストを更新（壊れた状態）
 └─ メモリ破壊！
```

malloc() は内部で「空きメモリのリスト」を管理しています

リストを更新している途中でシグナルが届き、ハンドラ内でも malloc() を呼ぶと、リストが不整合な状態になります

結果として、メモリ破壊やクラッシュが発生します

printf() が安全でない理由は複数あります

<strong>1. 内部バッファの不整合</strong>

printf() は内部でバッファとそれに関連するカウンタ/インデックスを管理しています

メインプログラムが printf() を実行中（バッファ更新の途中）にシグナルハンドラが割り込み、そこでも printf() を呼ぶと、2 回目の呼び出しが不整合なデータで動作し、予期しない結果になります

<strong>2. デッドロックの可能性</strong>

printf() は内部でロック（排他制御）を使用している場合があります

メインプログラムがロックを保持している状態でシグナルハンドラが printf() を呼ぶと、同じロックを取得しようとしてデッドロック（永遠に停止）になる可能性があります

安全な関数の一覧は signal-safety(7) で確認できます

学習目的では printf() を使った例もありますが、実際のプログラムでは write() システムコールを使うか、フラグを設定してハンドラ外で処理してください

### [シグナルはキューされない](#signals-are-not-queued) {#signals-are-not-queued}

同じシグナルが連続して送られた場合、まとめて 1 回として扱われることがあります

これを「シグナルはキューされない」と言います

<strong>リアルタイムシグナル</strong>（SIGRTMIN 以上）は例外で、キューされます

リアルタイムシグナルは、通常のシグナル（SIGINT など）とは異なり、複数回送ると複数回配送されることが保証されています

ただし、このリポジトリでは通常のシグナルのみを扱います

---

## [次のステップ](#next-steps) {#next-steps}

このトピックでは、「シグナルでプロセスに通知する方法」を学びました

- シグナルとは何か
- シグナルハンドラの登録方法
- SIGCHLD でゾンビを防ぐ方法
- プロセス間でシグナルを送り合う方法

次の [04-thread](../04-thread/) では、プロセスの中で複数の処理を同時に行う方法を学びます

- 1 つのプロセスの中で複数の流れを持つとは？
- スレッドとプロセスの違い
- スレッドでシグナルを扱うときの注意点

これらの疑問に答えます

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 英語 | 説明 |
| -------------------- | ----------------- | -------------------------------------------------- |
| シグナル | Signal | プロセスに送られる非同期の通知 |
| シグナルハンドラ | Signal Handler | シグナルを受け取ったときに実行される関数 |
| シグナルマスク | Signal Mask | 現在ブロックしているシグナルの一覧 |
| デフォルト動作 | Default Action | ハンドラを登録していないときのシグナルへの応答 |
| 非同期 | Asynchronous | いつ発生するか予測できないこと |
| 割り込み | Interrupt | 実行中の処理を中断させるイベント |
| ブロック | Block | シグナルを一時的に受け取らないようにすること |
| マスク | Mask | シグナルを一時的にブロックする仕組み |
| 保留 | Pending | マスクされたシグナルが配送を待っている状態 |
| 非同期シグナル安全 | Async-Signal-Safe | シグナルハンドラ内で安全に呼べる関数 |
| コアダンプ | Core Dump | プロセス終了時のメモリ内容を保存したファイル |
| ジョブ制御 | Job Control | シェルがプロセスを一時停止・再開する機能 |
| 競合状態 | Race Condition | タイミングによって結果が変わってしまう状況 |
| アトミック | Atomic | 途中で中断されずに一度に実行される操作 |
| ゾンビプロセス | Zombie Process | 終了したが親プロセスが回収していないプロセス |
| プロセス間通信 | IPC | 複数のプロセスがデータをやり取りする仕組みの総称 |
| システムコール | System Call | プログラムが OS に処理を依頼する仕組み |
| POSIX | POSIX | Unix 系 OS の標準仕様 |
| API | API | プログラムから機能を呼び出すための取り決め |
| アーキテクチャ | Architecture | CPU の種類や設計 |
| 移植性 | Portability | 異なる環境でも同じように動作すること |
| フォアグラウンド | Foreground | ターミナルと対話中のプロセス |
| バックグラウンド | Background | ターミナルから切り離されて動作するプロセス |
| デッドロック | Deadlock | 複数の処理が互いに待ち合って永遠に進まなくなる状態 |
| リアルタイムシグナル | Realtime Signal | キューされる特別なシグナル（SIGRTMIN 以上） |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

- [signal(7) - Linux manual page](https://man7.org/linux/man-pages/man7/signal.7.html){:target="\_blank"}
  - シグナルの概要、シグナル番号のアーキテクチャ別一覧
- [signal(2) - Linux manual page](https://man7.org/linux/man-pages/man2/signal.2.html){:target="\_blank"}
  - signal() システムコール
- [sigaction(2) - Linux manual page](https://man7.org/linux/man-pages/man2/sigaction.2.html){:target="\_blank"}
  - sigaction() システムコール
- [kill(2) - Linux manual page](https://man7.org/linux/man-pages/man2/kill.2.html){:target="\_blank"}
  - シグナルの送信
- [raise(3) - Linux manual page](https://man7.org/linux/man-pages/man3/raise.3.html){:target="\_blank"}
  - 自分自身へのシグナル送信、シングルスレッド/マルチスレッドでの動作の違い
- [pause(2) - Linux manual page](https://man7.org/linux/man-pages/man2/pause.2.html){:target="\_blank"}
  - シグナルを待つ
- [signal-safety(7) - Linux manual page](https://man7.org/linux/man-pages/man7/signal-safety.7.html){:target="\_blank"}
  - 非同期シグナル安全な関数の一覧と、シグナルハンドラ内で安全に呼べる関数についての詳細な解説
