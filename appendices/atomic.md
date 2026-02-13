---
layout: default
title: なぜ i++ は危険なのか
---

# [なぜ i++ は危険なのか](#why-increment-is-dangerous) {#why-increment-is-dangerous}

## [はじめに](#introduction) {#introduction}

[04-thread](../../04-thread/) で、複数のスレッドが同じ変数にアクセスすると<strong>競合状態</strong>が発生することを学びました

```c
int counter = 0;

void *increment(void *arg) {
    for (int i = 0; i < 100000; i++) {
        counter++;  /* 競合状態が発生 */
    }
    return NULL;
}
```

2 つのスレッドがこの関数を実行すると、counter は 200000 になるはずです

しかし、実際は 200000 より少ない値になることがあります

このページでは、なぜ `counter++` が期待通りに動かないのかを深掘りします

---

## [目次](#table-of-contents) {#table-of-contents}

- [i++の正体](#true-nature-of-increment)
- [何が起きるか](#what-happens)
- [アトミックとは](#what-is-atomic)
- [対策](#countermeasures)
- [まとめ](#summary)
- [参考資料](#references)

---

## [i++の正体](#true-nature-of-increment) {#true-nature-of-increment}

`i++` は 1 行のコードですが、CPU レベルでは<strong>3 つの操作</strong>に分かれます

<strong>1. ロード（Load）</strong>

メモリから i の値を CPU のレジスタに読み込みます

<strong>2. 加算（Add）</strong>

レジスタ内の値に 1 を加えます

<strong>3. ストア（Store）</strong>

レジスタの値をメモリに書き戻します

```
i++ の内部動作

1. レジスタ ← メモリ[i]   （ロード）
2. レジスタ ← レジスタ + 1 （加算）
3. メモリ[i] ← レジスタ   （ストア）
```

この 3 つの操作の間に、他のスレッドが割り込む可能性があります

---

## [何が起きるか](#what-happens) {#what-happens}

2 つのスレッドが同時に `counter++` を実行する場合を考えます

counter の初期値は 0 です

{: .labeled}
| 順番 | スレッド A | スレッド B | counter の値 |
| ---- | ------------------ | ------------------ | ------------ |
| 1 | ロード（0 を読む） | | 0 |
| 2 | | ロード（0 を読む） | 0 |
| 3 | 加算（0 + 1 = 1） | | 0 |
| 4 | | 加算（0 + 1 = 1） | 0 |
| 5 | ストア（1 を書く） | | 1 |
| 6 | | ストア（1 を書く） | 1 |

両方のスレッドが `counter++` を実行したのに、counter は 1 しか増えていません

これが<strong>競合状態</strong>です

### [なぜこうなるのか](#why-this-happens) {#why-this-happens}

スレッド B がロードした時点で、スレッド A はまだストアを完了していません

そのため、スレッド B は古い値（0）を読んでしまいます

結果として、スレッド A の更新がスレッド B によって上書きされ、1 回分の増加が失われます

---

## [アトミックとは](#what-is-atomic) {#what-is-atomic}

<strong>アトミック（atomic）</strong>とは、「分割不可能」という意味です

原子（atom）が「これ以上分割できない最小単位」とされていたことに由来します

<strong>アトミック操作</strong>とは、途中で割り込まれることなく、最初から最後まで一気に実行される操作です

### [日常の例え](#everyday-analogy) {#everyday-analogy}

ATM での銀行振込を考えてみましょう

```
振込の手順
1. 送金元の残高を確認する
2. 送金元から金額を引く
3. 送金先に金額を加える
```

もし手順 2 と手順 3 の間でシステムが停止したら、お金が消えてしまいます

これを防ぐため、振込処理は「全部成功するか、全部失敗するか」のどちらかになります

途中の状態は存在しません

これが<strong>アトミック</strong>の考え方です

### [i++ をアトミックにする](#making-increment-atomic) {#making-increment-atomic}

`i++` の 3 つの操作を、他のスレッドから見て「1 つの操作」に見えるようにすれば、競合状態を防げます

### [なぜ CPU のサポートが必要なのか](#why-cpu-support-is-needed) {#why-cpu-support-is-needed}

アトミック操作は、ソフトウェアだけでは実現できません

CPU が特別な命令を提供しているからこそ可能です

<strong>x86 の例：LOCK プレフィックス</strong>

```
通常の加算命令：ADD [memory], 1
アトミック加算：LOCK ADD [memory], 1
```

LOCK プレフィックスを付けると、その命令の実行中は他のコアがそのメモリにアクセスできなくなります

<strong>なぜソフトウェアだけでは不十分か？</strong>

{: .labeled}
| 方法 | 問題点 |
| ------------------------ | -------------------------------------- |
| 複数の命令で実現 | 命令の間で割り込まれる可能性がある |
| OS に依頼 | システムコールのオーバーヘッドが大きい |
| CPU 命令（ハードウェア） | 高速かつ確実にアトミック性を保証 |

atomic 型の関数は、内部でこれらの CPU 命令を使用しています

---

## [対策](#countermeasures) {#countermeasures}

### [mutex（ミューテックス）](#mutex) {#mutex}

[04-thread](../../04-thread/#ミューテックスmutexによる排他制御) で学んだ mutex を使う方法です

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void *increment(void *arg) {
    for (int i = 0; i < 100000; i++) {
        pthread_mutex_lock(&mutex);
        counter++;
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}
```

mutex は「一度に 1 つのスレッドだけ」がクリティカルセクションを実行できるようにします

詳細は [04-thread](../../04-thread/) を参照してください

### [atomic 型（C11）](#atomic-type) {#atomic-type}

C11 から、`<stdatomic.h>` ヘッダで<strong>アトミック型</strong>が導入されました

<strong>atomic_int</strong> などの型を使うと、ロック不要でアトミック操作ができます

```c
#include <stdatomic.h>

atomic_int counter = 0;

void *increment(void *arg) {
    for (int i = 0; i < 100000; i++) {
        atomic_fetch_add(&counter, 1);
    }
    return NULL;
}
```

<strong>atomic_fetch_add()</strong> は、値の読み取り・加算・書き込みをアトミックに行います

mutex と比べて軽量で、単純なカウンタの更新に適しています

<strong>よく使うアトミック型</strong>

{: .labeled}
| 型 | 説明 |
| ----------- | -------------------- |
| atomic_int | アトミックな int |
| atomic_long | アトミックな long |
| atomic_bool | アトミックな bool |
| atomic_flag | ロックフリーなフラグ |

<strong>よく使うアトミック関数</strong>

{: .labeled}
| 関数 | 説明 |
| ------------------------- | ------------------ |
| atomic_load() | 値を読み取る |
| atomic_store() | 値を書き込む |
| atomic_fetch_add() | 加算して旧値を返す |
| atomic_fetch_sub() | 減算して旧値を返す |
| atomic_compare_exchange() | 条件付き交換 |

---

### [sig_atomic_t](#sig-atomic-t) {#sig-atomic-t}

<strong>sig_atomic_t</strong> は、シグナルハンドラ内で安全に読み書きできる整数型です

[03-signal](../../03-signal/) で学んだように、シグナルハンドラには制限があります

```c
#include <signal.h>

volatile sig_atomic_t flag = 0;

void handler(int signum) {
    flag = 1;  /* シグナルハンドラ内で安全 */
}

int main(void) {
    signal(SIGINT, handler);

    while (!flag) {
        /* フラグが立つまで待機 */
    }

    return 0;
}
```

<strong>volatile</strong> は、コンパイラの最適化を抑制する修飾子です

シグナルハンドラから変更される変数には、必ず `volatile` を付けます

<strong>注意</strong>

sig_atomic_t は、シグナルハンドラとメインプログラム間の通信用です

スレッド間の共有には、mutex または atomic 型を使ってください

{: .labeled}
| 用途 | 適切な選択 |
| ---------------------- | --------------------- |
| シグナルハンドラ | volatile sig_atomic_t |
| スレッド間（一般） | mutex |
| スレッド間（カウンタ） | atomic 型 |

### [mutex と atomic の選び方](#choosing-mutex-vs-atomic) {#choosing-mutex-vs-atomic}

<strong>atomic を選ぶべき場合</strong>

- 1 つの変数だけを更新する
- 単純な操作（インクリメント、フラグの設定など）
- パフォーマンスが重要

<strong>mutex を選ぶべき場合</strong>

- 複数の変数を一貫性を保って更新する
- 複雑な条件判断を含む操作
- データ構造全体を保護する

```c
/* atomic が適切：単純なカウンタ */
atomic_fetch_add(&counter, 1);

/* mutex が適切：複数の変数の一貫性が必要 */
pthread_mutex_lock(&mutex);
balance_a -= amount;
balance_b += amount;  /* 両方が同時に更新される必要がある */
pthread_mutex_unlock(&mutex);
```

---

## [まとめ](#summary) {#summary}

<strong>i++ が危険な理由</strong>

`i++` は 1 行のコードに見えますが、実際は 3 つの操作（ロード・加算・ストア）に分かれます

複数のスレッドが同時に実行すると、操作が交互に実行され、更新が失われる可能性があります

<strong>アトミックとは</strong>

途中で割り込まれることなく、最初から最後まで一気に実行される操作です

<strong>対策の使い分け</strong>

{: .labeled}
| 状況 | 対策 |
| ---------------------- | --------------------- |
| 複雑な処理を保護したい | mutex |
| 単純なカウンタを更新 | atomic 型（C11） |
| シグナルハンドラで使う | volatile sig_atomic_t |

---

## [参考資料](#references) {#references}

<strong>C 言語規格</strong>

- [Atomic operations library - cppreference](https://en.cppreference.com/w/c/atomic){:target="\_blank"}
  - C11 のアトミック操作ライブラリ
- [sig_atomic_t - cppreference](https://en.cppreference.com/w/c/program/sig_atomic_t){:target="\_blank"}
  - シグナルハンドラで安全に使える整数型

<strong>Linux マニュアル</strong>

- [pthread_mutex_lock(3) - Linux manual page](https://man7.org/linux/man-pages/man3/pthread_mutex_lock.3.html){:target="\_blank"}
  - ミューテックスのロック操作

<strong>本編との関連</strong>

- [04-thread](../../04-thread/)
  - 競合状態とミューテックスの詳細
- [03-signal](../../03-signal/)
  - シグナルハンドラの制限と安全な関数
