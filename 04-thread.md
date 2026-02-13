---
layout: default
title: プロセス内で並行処理
---

# [04-thread：プロセス内で並行処理](#concurrent-processing-in-process) {#concurrent-processing-in-process}

## [はじめに](#introduction) {#introduction}

前のトピック（03-signal）で、シグナルによるプロセス間の通知を学びました

シグナルは「外部から」プロセスに通知を送る仕組みでした

しかし、時には「1 つのプロセスの中で複数の処理を同時に行いたい」場合があります

たとえば、ファイルをダウンロードしながら進捗バーを表示する場合などです

02-fork-exec で学んだ fork() でこれを実現すると、親子間でデータを共有するのが大変です（別々のメモリ空間なので）

ここで登場するのが<strong>スレッド</strong>です

スレッドは、同じプロセス内で<strong>メモリを共有</strong>しながら複数の処理を並行して実行できます

### [日常の例え](#everyday-analogy) {#everyday-analogy}

プロセスを「1 つの家」、スレッドを「家に住む家族」と考えてみましょう

fork() は、隣に新しい家を建てて引っ越すようなものです

スレッドは、同じ家の中で家族がそれぞれ別の作業をするようなものです

家（メモリ）は共有しているので、リビングのテレビや冷蔵庫の中身は全員が使えます

でも、同時に冷蔵庫を開けようとすると困ることがあります（これが後で学ぶ「競合状態」です）

### [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

<strong>pthread</strong>（POSIX Threads）は、スレッドを扱うための標準 API です

<strong>API（Application Programming Interface）</strong>とは、プログラムが特定の機能を使うための「窓口」のことです

<strong>POSIX（Portable Operating System Interface）</strong>とは、UNIX 系 OS で共通して使える機能を定めた標準規格です

pthread を使えば、Linux でも macOS でも同じコードでスレッドを扱えます

このページでは、以下の関数を学びます

- <strong>pthread_create()</strong>
  - スレッドを作成する
- <strong>pthread_join()</strong>
  - スレッドの終了を待つ
- <strong>pthread_exit()</strong>
  - スレッドを終了する
- <strong>pthread_mutex_lock() / pthread_mutex_unlock()</strong>
  - 排他制御を行う
- <strong>pthread_cond_wait() / pthread_cond_signal()</strong>
  - 条件変数で待機・通知する

---

## [目次](#table-of-contents) {#table-of-contents}

1. [スレッドとは何か](#what-is-a-thread)
2. [スレッドとプロセスの違い](#difference-between-thread-and-process)
3. [pthread_create() によるスレッド作成](#creating-threads-with-pthread-create)
4. [pthread_join() による終了待ち](#waiting-for-threads-with-pthread-join)
5. [pthread_exit() によるスレッド終了](#thread-termination-with-pthread-exit)
6. [メモリの共有](#memory-sharing)
7. [競合状態（データ競合）](#race-condition)
8. [ミューテックス（mutex）による排他制御](#mutex-for-mutual-exclusion)
9. [スレッドセーフな関数](#thread-safe-functions)
10. [条件変数（condition variable）](#condition-variables)
11. [スレッドとシグナル](#threads-and-signals)
12. [pthread 関数のエラー処理](#pthread-error-handling)
13. [次のステップ](#next-steps)
14. [用語集](#glossary)
15. [参考資料](#references)

---

## [スレッドとは何か](#what-is-a-thread) {#what-is-a-thread}

### [基本的な説明](#basic-explanation) {#basic-explanation}

<strong>スレッド</strong>は、プロセス内で実行される「処理の流れ」です

1 つのプロセスには、少なくとも 1 つのスレッドがあります（メインスレッド）

pthread_create() を使うと、追加のスレッドを作成できます

Linux の公式マニュアルには、こう書かれています

> A single process can contain multiple threads, all of which are executing the same program.

> 1 つのプロセスは複数のスレッドを含むことができ、それらはすべて同じプログラムを実行しています

### [スレッドが持つもの](#what-a-thread-has) {#what-a-thread-has}

各スレッドは、以下の要素を<strong>独自に</strong>持ちます

- <strong>スタック</strong>
  - 局所変数を格納する領域です
  - [01-process](../01-process/) の「メモリの構造」で学習しました
- <strong>プログラムカウンタ</strong>
  - CPU が「今どの命令を実行しているか」を示す場所です
  - 各スレッドは独自の実行位置を持ちます
- <strong>レジスタ</strong>
  - CPU 内部の小さなメモリです
  - 計算中の値を一時的に保持します
- <strong>スレッド ID</strong>
  - そのスレッドを識別するための番号です

### [スレッドが共有するもの](#what-threads-share) {#what-threads-share}

同じプロセス内のスレッドは、以下の要素を<strong>共有</strong>します

- <strong>ヒープ</strong>
  - 動的に確保されるメモリ領域です
  - [01-process](../01-process/) の「メモリの構造」で学習しました
- <strong>グローバル変数</strong>
  - プログラム全体で使える変数です
- <strong>静的変数</strong>
  - `static` キーワードで宣言された変数です
  - 関数内で宣言しても、プログラム終了まで値が保持されます
- <strong>ファイルディスクリプタ</strong>
  - [05-file-descriptor](../05-file-descriptor/) で詳しく学習します
- <strong>プロセス ID</strong>

### [「リソースを共有する」とは](#what-sharing-resources-means) {#what-sharing-resources-means}

スレッドについて「リソースを共有する」という説明をよく目にします

<strong>リソース</strong>（資源）とは、プログラムが使う「モノ」全般を指します

ここで、スレッドが<strong>共有するリソース</strong>と<strong>共有しないリソース</strong>を整理しておきましょう

<strong>スレッド間で共有されるリソース</strong>

{: .labeled}
| リソース | 説明 |
| ------------------------- | --------------------------------------------------------------------------- |
| 仮想アドレス空間 | プロセスに割り当てられたメモリの「見え方」（[01-process](../01-process/)） |
| ヒープ領域 | malloc() で確保したメモリ |
| グローバル変数 | プログラム全体で使う変数 |
| 静的変数（static） | 関数内でも値が保持される変数 |
| ファイルディスクリプタ | 開いているファイルの番号（[05-file-descriptor](../05-file-descriptor/)） |
| シグナルハンドラ | シグナル受信時の動作設定（[03-signal](../03-signal/)） |
| 作業ディレクトリ | プロセスの「今いる場所」 |
| ユーザー ID / グループ ID | 権限情報 |

<strong>スレッドごとに独立しているリソース</strong>

{: .labeled}
| リソース | 説明 |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| スタック | 関数の局所変数を格納する領域 |
| プログラムカウンタ | 今どの命令を実行しているか |
| レジスタ | CPU の一時的な作業領域 |
| スレッド ID | そのスレッドを識別する番号 |
| シグナルマスク | どのシグナルを一時的にブロックするか（ブロック中のシグナルは無視されるのではなく保留され、ブロック解除時に配信されます）（[03-signal](../03-signal/)） |
| errno | エラー番号を格納する変数（[01-process](../01-process/) の「C 言語の読み方」で説明） |

<strong>共有と独立の意味</strong>

- <strong>共有</strong>：あるスレッドが変更すると、他のスレッドからも見える
- <strong>独立</strong>：あるスレッドが変更しても、他のスレッドには影響しない

この違いを理解することが、スレッドプログラミングの第一歩です

---

## [スレッドとプロセスの違い](#difference-between-thread-and-process) {#difference-between-thread-and-process}

### [なぜスレッドが必要なのか](#why-threads-are-needed) {#why-threads-are-needed}

<strong>もしすべてを fork() で実現したら？</strong>

fork() でも並行処理は可能ですが、いくつかの問題があります

<strong>問題 1：データ共有のオーバーヘッド</strong>

fork() では子プロセスは親のメモリをコピー（Copy-on-Write）します

データを共有するには、パイプや共有メモリなどの<strong>IPC（プロセス間通信）</strong>が必要です

```
fork() でデータを共有する場合：
親プロセス → [パイプ/共有メモリ] → 子プロセス
              ↑ 設定が複雑、オーバーヘッドあり
```

<strong>問題 2：fork() のコスト</strong>

fork() はページテーブルのコピーや、プロセステーブルへの登録など、OS レベルの処理が必要です

スレッド作成は、同じプロセス内での作業なので、はるかに軽量です

<strong>スレッドによる解決</strong>

スレッドはメモリを共有しているため

- グローバル変数やヒープを直接読み書きできる
- IPC の設定が不要
- 作成コストが小さい

### [比較表](#comparison-table) {#comparison-table}

{: .labeled}
| 項目 | プロセス（fork） | スレッド（pthread） |
| ------------------ | -------------------------------------------------- | ------------------------ |
| メモリ空間 | 独立（コピー） | 共有 |
| 作成コスト | 大きい | 小さい |
| 通信方法 | IPC（プロセス間通信、[07-ipc](../07-ipc/)）が必要 | 変数を直接共有 |
| 終了時の影響 | 他のプロセスに影響なし | 同じプロセス内に影響あり |
| デバッグのしやすさ | 比較的しやすい | 難しい（競合状態等） |

### [fork() との対比](#comparison-with-fork) {#comparison-with-fork}

fork() では、以下のような動作になります

- 子プロセスでグローバル変数を変更しても、親には影響しない
- これは、fork() がメモリをコピーするから

スレッドでは逆になります

- あるスレッドがグローバル変数を変更すると、他のスレッドからも見える
- これは、スレッドがメモリを共有しているから

### [いつスレッドを使うか](#when-to-use-threads) {#when-to-use-threads}

スレッドが適しているケース

- 同じデータを複数の処理で共有したい
- 軽量な並行処理が必要
- 応答性を維持しながら<strong>バックグラウンド処理</strong>（ユーザーの操作を妨げない裏での処理）をしたい

fork() が適しているケース

- 処理を完全に分離したい
- 別のプログラムを実行したい（exec と組み合わせ）
- 一方の障害が他方に影響しないようにしたい

---

## [pthread_create() によるスレッド作成](#creating-threads-with-pthread-create) {#creating-threads-with-pthread-create}

### [基本的な使い方](#basic-usage) {#basic-usage}

pthread_create() は、新しいスレッドを作成します

```c
#include <pthread.h>

pthread_t thread;
pthread_create(&thread, NULL, thread_function, argument);
```

<strong>pthread_t</strong> は、スレッドを識別するための型です

プロセス ID を格納する pid_t と同様に、スレッド ID を格納します

### [引数の説明](#argument-description) {#argument-description}

{: .labeled}
| 引数 | 型 | 説明 |
| --------- | ------------------ | --------------------------------------------- |
| 第 1 引数 | pthread*t \* | スレッド ID を格納する変数 |
| 第 2 引数 | pthread_attr_t \* | スレッドの属性（NULL でデフォルト設定を使用） |
| 第 3 引数 | void *(\_)(void \*) | スレッドで実行する関数 |
| 第 4 引数 | void \* | 関数に渡す引数 |

<strong>void \*</strong> については [01-process](../01-process/) の「C 言語の読み方」で説明しています

### [スレッド関数の形式](#thread-function-format) {#thread-function-format}

```c
void *thread_function(void *argument) {
    /* スレッドの処理 */
    return NULL;  /* または pthread_exit() */
}
```

- 引数：void \* 型（任意のポインタを受け取れる）
- 戻り値：void \* 型（任意のポインタを返せる）

---

## [pthread_join() による終了待ち](#waiting-for-threads-with-pthread-join) {#waiting-for-threads-with-pthread-join}

### [基本的な使い方](#basic-usage-join) {#basic-usage-join}

pthread_join() は、指定したスレッドが終了するまで待機します

```c
pthread_t thread;
void *result;

pthread_create(&thread, NULL, thread_function, NULL);
pthread_join(thread, &result);  /* スレッドの終了を待つ */
```

### [wait() との類似点](#similarity-with-wait) {#similarity-with-wait}

02-fork-exec で学んだ wait() は、子プロセスの終了を待ちました

pthread_join() は、スレッド版の wait() のようなものです

{: .labeled}
| 関数 | 対象 | 取得できるもの |
| -------------- | ---------- | ----------------- |
| wait() | 子プロセス | 終了ステータス |
| pthread_join() | スレッド | 戻り値（void \*） |

### [join しないとどうなるか](#what-happens-without-join) {#what-happens-without-join}

main() が return や exit() で終了すると、プロセス全体が終了し、すべてのスレッドも終了します

また、スレッドのリソースが解放されずに残る可能性があります

必ず pthread_join() でスレッドの終了を待つか、<strong>デタッチ状態</strong>にしましょう

<strong>デタッチ状態</strong>とは、スレッドの終了を待たなくても良い設定です

デタッチされたスレッドは、終了時に自動的にリソースが解放されます

pthread_detach() で設定できますが、詳細は省略します

---

## [pthread_exit() によるスレッド終了](#thread-termination-with-pthread-exit) {#thread-termination-with-pthread-exit}

### [基本的な使い方](#basic-usage-exit) {#basic-usage-exit}

pthread_exit() は、現在のスレッドを終了します

```c
void *thread_function(void *argument) {
    /* 処理 */
    pthread_exit(NULL);  /* スレッドを終了 */
}
```

### [return との違い](#difference-from-return) {#difference-from-return}

スレッド関数では、return と pthread_exit() はほぼ同じです

```c
return NULL;        /* スレッドを終了 */
pthread_exit(NULL); /* スレッドを終了（同じ効果） */
```

ただし、pthread_exit() はスレッド関数以外からも呼べます

### [メインスレッドで pthread_exit() を呼ぶと](#calling-pthread-exit-in-main) {#calling-pthread-exit-in-main}

メインスレッドで pthread_exit() を呼ぶと、メインスレッドだけが終了します

プロセス自体は終了せず、他のスレッドは実行を継続します

```c
int main() {
    pthread_t thread;
    pthread_create(&thread, NULL, thread_function, NULL);
    pthread_exit(NULL);  /* メインは終了するが、thread は続く */
}
```

### [main() で return すると](#returning-from-main) {#returning-from-main}

main() 関数で return すると、暗黙的に exit() が呼ばれ、<strong>プロセス全体が終了</strong>します

つまり、<strong>他のスレッドも強制終了</strong>されます

```c
int main() {
    pthread_t thread;
    pthread_create(&thread, NULL, thread_function, NULL);
    return 0;  /* exit() が呼ばれ、thread も終了する */
}
```

{: .labeled}
| 終了方法 | 動作 |
| ------------------------ | -------------------------------------------- |
| return（main内） | プロセス全体が終了（全スレッド終了） |
| pthread_exit()（main内） | メインスレッドだけ終了（他のスレッドは継続） |
| exit() | プロセス全体が終了（全スレッド終了） |

<strong>他のスレッドの完了を待ちたい場合</strong>

- pthread_join() で待機するか
- main() で pthread_exit() を呼ぶ

どちらかを選択してください

---

## [メモリの共有](#memory-sharing) {#memory-sharing}

### [共有されるもの](#what-is-shared) {#what-is-shared}

同じプロセス内のスレッドは、以下のメモリを共有します

- グローバル変数
- 静的変数（static）
- ヒープ（malloc で確保した領域）

### [共有されないもの](#what-is-not-shared) {#what-is-not-shared}

各スレッドは独自のスタックを持ちます

- <strong>局所変数</strong>（<strong>自動変数</strong>とも呼ばれる）はスタックに格納される
  - 「自動」は、関数に入ると自動的に作られ、出ると自動的に消えることを意味します
- 他のスレッドからは直接アクセスできない

ただし、ポインタを渡せば間接的にアクセスできます

### [fork() との対比](#comparison-with-fork-memory) {#comparison-with-fork-memory}

fork() では、子プロセスが変数を変更しても親には見えません

```c
/* fork() の場合 */
global_counter = 100;  /* 子で変更 */
/* 親の global_counter は 0 のまま */
```

スレッドでは、変更が他のスレッドから見えます

```c
/* スレッドの場合 */
global_counter = 100;  /* スレッドで変更 */
/* メインスレッドの global_counter も 100 になる */
```

---

## [競合状態（データ競合）](#race-condition) {#race-condition}

### [競合状態とは](#what-is-race-condition) {#what-is-race-condition}

<strong>競合状態（Race Condition）</strong>とは、複数のスレッドが同じデータに同時にアクセスし、実行順序によって結果が変わってしまう状態です

### [具体例](#concrete-examples) {#concrete-examples}

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

### [なぜ起こるか](#why-it-occurs) {#why-it-occurs}

`counter++` は、実際には 3 つの操作に分かれます

1. counter の値をメモリから読み取る
2. 読み取った値に 1 を加える
3. 結果をメモリに書き込む

2 つのスレッドがこれらの操作を交互に実行すると、以下のようなことが起こります

{: .labeled}
| スレッド A | スレッド B | counter の値 |
| ----------------- | ----------------- | ------------ |
| 読み取り（0） | | 0 |
| | 読み取り（0） | 0 |
| 加算（0 + 1 = 1） | | 0 |
| | 加算（0 + 1 = 1） | 0 |
| 書き込み（1） | | 1 |
| | 書き込み（1） | 1 |

両方のスレッドが counter++ を実行したのに、counter は 1 しか増えていません

---

## [ミューテックス（mutex）による排他制御](#mutex-for-mutual-exclusion) {#mutex-for-mutual-exclusion}

### [ミューテックスとは](#what-is-mutex) {#what-is-mutex}

<strong>ミューテックス（Mutex）</strong>は、Mutual Exclusion（相互排他）の略です

一度に 1 つのスレッドだけがコードの特定部分を実行できるようにします

### [基本的な使い方](#basic-usage-mutex) {#basic-usage-mutex}

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void *thread_function(void *arg) {
    pthread_mutex_lock(&mutex);   /* ロックを取得 */
    /* クリティカルセクション */
    /* ここは一度に 1 スレッドだけ実行できる */
    pthread_mutex_unlock(&mutex); /* ロックを解放 */
    return NULL;
}
```

### [クリティカルセクション](#critical-section) {#critical-section}

<strong>クリティカルセクション</strong>とは、同時に複数のスレッドが実行してはいけない区間です

lock と unlock の間がクリティカルセクションになります

### [競合状態の解決](#resolving-race-condition) {#resolving-race-condition}

```c
int counter = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void *increment(void *arg) {
    for (int i = 0; i < 100000; i++) {
        pthread_mutex_lock(&mutex);
        counter++;  /* 安全にインクリメント */
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}
```

これで counter は確実に 200000 になります

### [デッドロックに注意](#beware-of-deadlock) {#beware-of-deadlock}

<strong>デッドロック</strong>とは、複数のスレッドが互いにロックを待ち続けて、どちらも進めなくなる状態です

```c
/* スレッド A */
pthread_mutex_lock(&mutex1);
pthread_mutex_lock(&mutex2);  /* mutex2 を待つ */

/* スレッド B */
pthread_mutex_lock(&mutex2);
pthread_mutex_lock(&mutex1);  /* mutex1 を待つ */
```

上記の場合、スレッド A は mutex2 を、スレッド B は mutex1 を待ち続け、永遠に進みません

### [なぜデッドロック防止は難しいのか](#why-deadlock-prevention-is-difficult) {#why-deadlock-prevention-is-difficult}

デッドロックが発生する 4 つの条件（<strong>Coffman 条件</strong>）は知られています

{: .labeled}
| 条件 | 説明 |
| ---------- | ------------------------------------------- |
| 相互排他 | リソースは一度に 1 つのスレッドしか使えない |
| 保持と待機 | リソースを保持したまま、他のリソースを待つ |
| 横取り不可 | 保持中のリソースを強制的に奪えない |
| 循環待ち | 待ちの連鎖が循環している |

<strong>理論的には</strong>、これらの条件を 1 つでも崩せばデッドロックは発生しません

<strong>実務上の困難</strong>

- <strong>相互排他を崩す</strong>：データの整合性を保つために必要なことが多い
- <strong>保持と待機を崩す</strong>：すべてのロックを一度に取得する必要があり、パフォーマンスが低下
- <strong>横取り不可を崩す</strong>：データの破損を招く可能性がある
- <strong>循環待ちを崩す</strong>：ロックの順序を全プログラマが守る必要があり、大規模プロジェクトでは困難

実際には「ロックを取得する順序を統一する」（循環待ちの防止）が最も現実的な対策です

---

## [スレッドセーフな関数](#thread-safe-functions) {#thread-safe-functions}

### [スレッドセーフとは](#what-is-thread-safe) {#what-is-thread-safe}

<strong>スレッドセーフ</strong>とは、複数のスレッドから同時に呼び出しても安全に動作することです

### [なぜ同じ関数が安全でないことがあるのか](#why-same-function-can-be-unsafe) {#why-same-function-can-be-unsafe}

一部の関数は<strong>内部状態</strong>を保持しています

この内部状態が複数のスレッドで共有されると、予期しない動作が発生します

<strong>strtok() の例</strong>

strtok() は文字列を分割する関数ですが、内部に「次の位置」を記憶しています

```c
char str1[] = "a,b,c";
char str2[] = "x,y,z";

/* スレッド A */
char *token1 = strtok(str1, ",");  /* 内部状態を "str1 の次の位置" に設定 */

/* スレッド B が割り込む */
char *token2 = strtok(str2, ",");  /* 内部状態を "str2 の次の位置" に上書き！ */

/* スレッド A に戻る */
char *token3 = strtok(NULL, ",");  /* str2 の次の位置を参照してしまう！ */
```

<strong>strtok_r() による解決</strong>

strtok_r() は、内部状態を呼び出し側で管理します

```c
char *saveptr1, *saveptr2;

/* スレッド A */
char *token1 = strtok_r(str1, ",", &saveptr1);  /* 状態を saveptr1 に保存 */

/* スレッド B */
char *token2 = strtok_r(str2, ",", &saveptr2);  /* 状態を saveptr2 に保存 */

/* スレッド A */
char *token3 = strtok_r(NULL, ",", &saveptr1);  /* saveptr1 を使うので安全 */
```

`_r` は "reentrant"（再入可能）の略で、スレッドセーフな関数を示す命名規則です

---

## [条件変数（condition variable）](#condition-variables) {#condition-variables}

### [なぜミューテックスだけでは不十分なのか](#why-mutex-alone-is-insufficient) {#why-mutex-alone-is-insufficient}

「データが準備できたら処理を始める」というシナリオを考えてみましょう

<strong>ミューテックスだけで実装すると</strong>

```c
/* 待機する側（ポーリング） */
while (1) {
    pthread_mutex_lock(&mutex);
    if (data_ready) {
        /* データを処理 */
        pthread_mutex_unlock(&mutex);
        break;
    }
    pthread_mutex_unlock(&mutex);
    sleep(1);  /* 1秒待って再チェック */
}
```

<strong>この方法の問題点</strong>

{: .labeled}
| 問題 | 説明 |
| -------- | ---------------------------------------------- |
| CPU 消費 | 何度もロック・アンロックを繰り返す |
| 応答遅延 | sleep() の間は条件の変化に気づけない |
| 非効率 | データが準備されていなくても何度もチェックする |

<strong>条件変数による解決</strong>

条件変数を使うと、「条件が満たされるまで効率的に待つ」ことができます

- 待機中は CPU を消費しない
- 条件が満たされた瞬間に起きる
- ポーリングの無駄がない

### [条件変数とは](#what-is-condition-variable) {#what-is-condition-variable}

<strong>条件変数</strong>は、ある条件が満たされるまでスレッドを待機させる仕組みです

ミューテックスと組み合わせて使います

### [基本的な使い方](#basic-usage-cond) {#basic-usage-cond}

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int data_ready = 0;

/* 待機する側 */
void *consumer(void *arg) {
    pthread_mutex_lock(&mutex);
    while (!data_ready) {
        pthread_cond_wait(&cond, &mutex);  /* 条件を待つ */
    }
    /* データを処理 */
    pthread_mutex_unlock(&mutex);
    return NULL;
}

/* 通知する側 */
void *producer(void *arg) {
    pthread_mutex_lock(&mutex);
    data_ready = 1;
    pthread_cond_signal(&cond);  /* 条件が満たされたことを通知 */
    pthread_mutex_unlock(&mutex);
    return NULL;
}
```

### [pthread_cond_wait() の動作](#pthread-cond-wait-behavior) {#pthread-cond-wait-behavior}

1. mutex のロックを解放する
2. 条件変数で待機する
3. シグナルを受け取ったら mutex を再取得する

### [なぜ while でチェックするか](#why-check-with-while) {#why-check-with-while}

```c
while (!data_ready) {  /* if ではなく while */
    pthread_cond_wait(&cond, &mutex);
}
```

pthread_cond_wait() は、条件が満たされていなくても戻ることがあります

これを<strong>スプリアスウェイクアップ</strong>（偽の起床）と呼びます

POSIX 仕様では「スプリアスウェイクアップが発生する可能性がある（may occur）」と明記されています

主な原因は以下のとおりです

- <strong>マルチプロセッサ環境での競合</strong>：複数のスレッドが同じ条件変数で待機している場合、pthread_cond_signal() が 1 つのスレッドだけを起こすはずでも、実装上の都合で複数のスレッドが起きてしまうことがあります
- <strong>起床後の競合</strong>：あるスレッドが起きてから実際に実行されるまでの間に、別のスレッドが先に条件を変更してしまうことがあります

while でループすることで、本当に条件が満たされているか再確認できます

---

## [スレッドとシグナル](#threads-and-signals) {#threads-and-signals}

### [境界トピックとして](#as-boundary-topic) {#as-boundary-topic}

スレッドとシグナルの組み合わせは複雑です

ここでは基本的な注意点のみ説明します

### [どのスレッドがシグナルを受け取るか](#which-thread-receives-signals) {#which-thread-receives-signals}

プロセス宛てのシグナル（kill コマンドで送られるものなど）は、どのスレッドが受け取るか不定です

kill コマンドについては [03-signal](../03-signal/) で学びました

特定のスレッドにシグナルを送るには pthread_kill() を使います

<strong>pthread_kill()</strong> は、指定したスレッドにシグナルを送る関数です

### [推奨されるパターン](#recommended-pattern) {#recommended-pattern}

マルチスレッドプログラムでは、以下のパターンが推奨されます

1. メインスレッドでシグナルを<strong>ブロック</strong>（一時的に受け取らない設定に）する
2. 専用のスレッドでシグナルを処理する

```c
/**
 * シグナル処理専用スレッドのパターン
 *
 * sigset_t はシグナルの集合を表す型です（03-signal で学習）
 */
sigset_t set;
sigemptyset(&set);           /* 集合を空にする */
sigaddset(&set, SIGINT);     /* SIGINT を集合に追加 */
pthread_sigmask(SIG_BLOCK, &set, NULL);  /* これらのシグナルをブロック */

/* シグナル処理スレッドで */
int sig;
sigwait(&set, &sig);  /* シグナルを同期的に待つ */
```

### [03-signal で学んだ注意点](#notes-learned-in-signal) {#notes-learned-in-signal}

シグナルハンドラ内では、<strong>非同期シグナル安全</strong>な関数しか呼べません

<strong>非同期シグナル安全</strong>とは、シグナルハンドラ内で安全に呼び出せることを意味します

詳しくは [03-signal](../03-signal/) を参照してください

スレッドを使う場合も同様です

---

## [pthread 関数のエラー処理](#pthread-error-handling) {#pthread-error-handling}

### [通常のシステムコールとの違い](#difference-from-regular-syscalls) {#difference-from-regular-syscalls}

pthread 関数のエラー処理は、通常のシステムコールとは異なる重要な特徴があります

<strong>通常のシステムコール</strong>（open()、read() など）

- 失敗時に -1 を返す
- エラー番号を errno に設定する

<strong>pthread 関数</strong>

- 成功時に 0 を返す
- 失敗時にエラー番号を<strong>戻り値として直接返す</strong>
- <strong>errno は設定しない</strong>

Linux の公式マニュアル（pthreads(7)）には、こう書かれています

> Most pthreads functions return 0 on success, and an error number on failure. Note that the pthreads functions do not set errno.

> ほとんどの pthread 関数は、成功時に 0 を返し、失敗時にエラー番号を返します
>
> pthread 関数は errno を設定しないことに注意してください

### [正しいエラー処理の例](#correct-error-handling-example) {#correct-error-handling-example}

```c
int result = pthread_create(&thread, NULL, thread_function, NULL);

if (result != 0) {
    /* errno ではなく、戻り値を直接使う */
    fprintf(stderr, "pthread_create failed: %s\n", strerror(result));
    return EXIT_FAILURE;
}
```

### [なぜ errno を使わないのか](#why-not-using-errno) {#why-not-using-errno}

pthread 関数が errno を使わない理由は、スレッドセーフであるためです

errno はスレッドごとに独立していますが、pthread 関数が errno を変更すると、他のライブラリ関数のエラー情報を上書きしてしまう可能性があります

戻り値で直接エラー番号を返すことで、この問題を回避しています

### [よくある間違い](#common-mistakes) {#common-mistakes}

```c
/* 間違った例：errno を使っている */
if (pthread_create(&thread, NULL, thread_function, NULL) != 0) {
    perror("pthread_create");  /* perror() は errno を参照するので不正確 */
}
```

```c
/* 正しい例：戻り値を使う */
int result = pthread_create(&thread, NULL, thread_function, NULL);
if (result != 0) {
    fprintf(stderr, "pthread_create: %s\n", strerror(result));
}
```

---

## [次のステップ](#next-steps) {#next-steps}

このトピックでは、「プロセス内で並行処理を行う方法」を学びました

- スレッドとは何か
- fork() との違い（メモリ共有）
- 競合状態とその解決方法（ミューテックス）
- 条件変数による同期

次の [05-file-descriptor](../05-file-descriptor/) では、OS のファイル管理機構を学びます

- ファイルディスクリプタとは何か
- 01-process で観察した fd の仕組みを詳しく理解
- リダイレクトの実装原理

これらの疑問に答えます

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 英語 | 説明 |
| ------------------------ | ----------------------------------- | -------------------------------------------------------------- |
| スレッド | Thread | プロセス内の実行の流れ |
| メインスレッド | Main Thread | プロセス開始時に作られる最初のスレッド |
| POSIX スレッド | POSIX Threads | pthread ライブラリの正式名称 |
| API | Application Programming Interface | プログラムが特定の機能を使うための窓口 |
| POSIX | Portable Operating System Interface | UNIX 系 OS の標準規格 |
| 並行処理 | Concurrent Processing | 複数の処理を同時期に進めること |
| 排他制御 | Mutual Exclusion | 同時アクセスを防ぐ仕組み |
| ミューテックス | Mutex | 排他制御を実現するためのロック機構 |
| クリティカルセクション | Critical Section | 同時実行を許さないコード区間 |
| 競合状態 | Race Condition | 実行順序によって結果が変わる状態 |
| デッドロック | Deadlock | 互いにロックを待ち続けて進めなくなる状態 |
| 条件変数 | Condition Variable | 条件を待つための同期機構 |
| スプリアスウェイクアップ | Spurious Wakeup | 条件を満たさずに待機から復帰すること |
| スレッドセーフ | Thread-Safe | 複数スレッドから安全に呼べること |
| プログラムカウンタ | Program Counter | CPU が実行中の命令の位置を示すもの |
| レジスタ | Register | CPU 内部の高速な一時記憶領域 |
| 仮想アドレス空間 | Virtual Address Space | プロセスに割り当てられたメモリの見え方 |
| 静的変数 | Static Variable | static キーワードで宣言された変数 |
| 自動変数 | Automatic Variable | 局所変数の別名、スタックに格納される |
| IPC | Inter-Process Communication | プロセス間通信 |
| バックグラウンド処理 | Background Processing | ユーザー操作を妨げない裏での処理 |
| デタッチ | Detach | スレッドの終了を待たない設定にすること |
| シグナルハンドラ | Signal Handler | シグナル受信時に実行される関数 |
| シグナルマスク | Signal Mask | ブロックするシグナルの設定（ブロック中は保留され後で配信） |
| 非同期シグナル安全 | Async-Signal-Safe | シグナルハンドラ内で安全に呼び出せること |
| pthread_t | - | スレッド ID を格納する型 |
| errno | - | エラー番号を格納するスレッドごとの変数（pthread は使用しない） |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

- [pthreads(7) - Linux manual page](https://man7.org/linux/man-pages/man7/pthreads.7.html){:target="\_blank"}
  - POSIX スレッドの概要
- [pthread_create(3) - Linux manual page](https://man7.org/linux/man-pages/man3/pthread_create.3.html){:target="\_blank"}
  - スレッドの作成
- [pthread_join(3) - Linux manual page](https://man7.org/linux/man-pages/man3/pthread_join.3.html){:target="\_blank"}
  - スレッドの終了待ち
- [pthread_exit(3) - Linux manual page](https://man7.org/linux/man-pages/man3/pthread_exit.3.html){:target="\_blank"}
  - スレッドの終了
- [pthread_mutex_lock(3) - Linux manual page](https://man7.org/linux/man-pages/man3/pthread_mutex_lock.3.html){:target="\_blank"}
  - ミューテックスのロック
- [pthread_cond_wait(3) - Linux manual page](https://man7.org/linux/man-pages/man3/pthread_cond_wait.3.html){:target="\_blank"}
  - 条件変数での待機
- [pthread_cancel(3) - Linux manual page](https://man7.org/linux/man-pages/man3/pthread_cancel.3.html){:target="\_blank"}
  - スレッドのキャンセル
