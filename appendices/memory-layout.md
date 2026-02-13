---
layout: default
title: ヒープとスタックは何が違うのか
---

# [ヒープとスタックは何が違うのか](#heap-vs-stack) {#heap-vs-stack}

## [はじめに](#introduction) {#introduction}

[01-process](../../01-process/) の「メモリの構造」で、プロセスのメモリが 5 つの領域に分かれていることを学びました

```c
int global = 100;           /* データセグメント */
int uninitialized;          /* BSS セグメント */

int main(void) {
    int local = 200;        /* スタック */
    int *ptr = malloc(sizeof(int));  /* ヒープ */
    *ptr = 300;
    free(ptr);
    return 0;
}
```

このコードで、`local` と `ptr` が指す領域は、どちらも「データを格納する場所」です

しかし、その性質はまったく異なります

スタックは「自動で用意され、自動で片付く」カフェテリアのトレーのようなもの

ヒープは「自分で借りて、自分で返す」駐車場のようなもの

このドキュメントでは、その違いと、間違った使い方をしたときに何が起こるかを詳しく説明します

---

## [目次](#table-of-contents) {#table-of-contents}

- [全体像](#overall-picture)
- [スタック](#stack)
- [ヒープ](#heap)
- [スタックとヒープの比較](#stack-vs-heap-comparison)
- [よくある問題](#common-problems)
- [確認方法](#confirmation-methods)
- [まとめ](#summary)
- [参考資料](#references)

---

## [全体像](#overall-picture) {#overall-picture}

プロセスのメモリは、低いアドレスから高いアドレスに向かって以下のように配置されます

{: .labeled}
| アドレス | 領域 | 成長方向 | 内容 |
| -------- | ------------------ | ---------- | ---------------------------------- |
| 低 | テキストセグメント | 固定 | プログラムのコード（読み取り専用） |
| ↓ | データセグメント | 固定 | 初期値ありのグローバル変数 |
| ↓ | BSS セグメント | 固定 | 初期値なしのグローバル変数 |
| ↓ | ヒープ | ↓ 下へ成長 | malloc() で動的に確保 |
| ↓ | （空き領域） | | |
| ↓ | スタック | ↑ 上へ成長 | ローカル変数、関数呼び出し情報 |
| 高 | | | |

### [なぜ 5 つの領域に分けるのか](#why-five-memory-regions) {#why-five-memory-regions}

<strong>もし全てのデータを 1 つの領域に混ぜたら？</strong>

{: .labeled}
| 問題 | 説明 |
| ------------ | ---------------------------------------------------- |
| セキュリティ | コードを書き換えて任意のプログラムを実行される可能性 |
| 効率 | 変わらないデータも毎回コピーする無駄 |
| 安定性 | 意図しない上書きでプログラムが壊れる |

各領域には分離する明確な理由があります

<strong>テキストセグメントが読み取り専用な理由</strong>

プログラムのコード（機械語）は実行中に変わりません

書き込みを禁止することで、バグや攻撃によるコード改ざんを防ぎます

また、同じプログラムを複数実行しても、コード部分は共有できるためメモリを節約できます

<strong>データセグメントと BSS セグメントを分ける理由</strong>

```c
int initialized = 100;  /* データセグメント：初期値あり */
int uninitialized;      /* BSS セグメント：初期値なし */
```

初期値のない変数は、実行ファイルに「100」のような値を書く必要がありません

「ここに 4 バイトの領域がある」という情報だけ記録すれば、OS がゼロで初期化します

これにより実行ファイルのサイズが小さくなります

<strong>例</strong>：`int array[1000000];` をグローバルに宣言しても、実行ファイルは 4MB 増えません

<strong>ヒープとスタックが別れている理由</strong>

スタックは関数呼び出しの構造（LIFO）に最適化されています

一方ヒープは、任意の順序で確保・解放できる柔軟さが必要です

両者の管理方法は根本的に異なるため、同じ仕組みでは効率が悪くなります

<strong>なぜヒープとスタックは逆方向に成長するのか</strong>

プログラムの実行中、どちらの領域がどれだけ必要になるかは事前にわかりません

互いに向かって成長させることで、空き領域を効率よく共有できます

ヒープ専用・スタック専用に領域を固定しなくて済むため、メモリを柔軟に使えます

詳しい概要は [01-process](../../01-process/#メモリの構造) を参照してください

---

## [スタック](#stack) {#stack}

### [スタックとは](#what-is-stack) {#what-is-stack}

<strong>スタック（Stack）</strong>は、関数呼び出しで自動的に確保され、関数終了時に自動的に解放されるメモリ領域です

LIFO（Last In, First Out：後入れ先出し、「ライフォ」と読みます）構造で管理されます

### [日常の例え](#everyday-analogy) {#everyday-analogy}

カフェテリアのトレーを想像してください

```
トレーの積み方

1. トレーを上に積む（関数を呼び出す）
2. 一番上のトレーから取る（関数から戻る）
3. 途中のトレーは取れない
```

スタックも同じです

最後に呼び出した関数が、最初に終了します

### [スタックに置かれるもの](#what-is-on-the-stack) {#what-is-on-the-stack}

{: .labeled}
| 種類 | 説明 |
| ------------------ | ------------------------ |
| ローカル変数 | 関数内で宣言した変数 |
| 関数の引数 | 関数に渡されたパラメータ |
| 戻りアドレス | 関数終了後に戻る場所 |
| 保存されたレジスタ | 呼び出し元の状態 |

### [スタックフレーム](#stack-frame) {#stack-frame}

関数が呼び出されるたびに、<strong>スタックフレーム</strong>という単位でメモリが確保されます

```
関数呼び出しとスタックフレーム

main() が func_a() を呼び、func_a() が func_b() を呼んだ場合

        高いアドレス
        ┌─────────────────┐
        │  main() の      │
        │  スタックフレーム │
        ├─────────────────┤
        │  func_a() の    │
        │  スタックフレーム │
        ├─────────────────┤
        │  func_b() の    │ ← 現在実行中
        │  スタックフレーム │
        └─────────────────┘
        低いアドレス（成長方向）
```

func_b() が終了すると、そのスタックフレームは自動的に解放されます

### [自動的な管理](#automatic-management) {#automatic-management}

```c
void function(void) {
    int local = 100;  /* スタックに確保される */
    /* ... */
}  /* 関数終了時に自動で解放される */
```

プログラマが明示的に解放する必要はありません

スコープを抜けると自動的に片付きます

---

## [ヒープ](#heap) {#heap}

### [ヒープとは](#what-is-heap) {#what-is-heap}

<strong>ヒープ（Heap）</strong>は、プログラマが明示的に確保・解放するメモリ領域です

`malloc()` で確保し、`free()` で解放します

### [日常の例え](#everyday-analogy) {#everyday-analogy}

月極駐車場を想像してください

```
駐車場の契約

1. 契約する（malloc）→ 駐車スペースを借りる
2. 使う → 車を停める
3. 解約する（free）→ 駐車スペースを返す
```

解約しないと、ずっと料金がかかります（メモリを占有し続けます）

### [malloc() と free()](#malloc-and-free) {#malloc-and-free}

```c
#include <stdlib.h>

int main(void) {
    /* ヒープに int サイズのメモリを確保 */
    int *ptr = malloc(sizeof(int));

    if (ptr == NULL) {
        /* 確保に失敗した場合 */
        return 1;
    }

    *ptr = 100;  /* 値を格納 */

    free(ptr);   /* 明示的に解放が必要 */
    ptr = NULL;  /* 解放後は NULL を代入（推奨） */

    return 0;
}
```

<strong>malloc() の戻り値</strong>

{: .labeled}
| 戻り値 | 意味 |
| --------- | ---------------------------- |
| NULL 以外 | 確保したメモリへのポインタ |
| NULL | 確保に失敗（メモリ不足など） |

必ず戻り値を確認してから使用します

### [内部の仕組み（概要）](#internal-mechanism) {#internal-mechanism}

malloc() は、内部で 2 つのシステムコールを使い分けています

{: .labeled}
| システムコール | 使われる場面 | 特徴 |
| -------------- | -------------------------- | ------------------------ |
| brk / sbrk | 小さなメモリ（128KB 未満） | ヒープ領域を拡張 |
| mmap | 大きなメモリ（128KB 以上） | 新しいメモリ領域をマップ |

閾値（MMAP_THRESHOLD）は環境によって異なりますが、glibc のデフォルトは 128KB です

※ glibc は動的閾値を使用しており、使用パターンに応じて 128KB ～ 512KB（32ビット環境）または 128KB ～ 32MB（64ビット環境）の間で自動調整されます

<strong>なぜ使い分けるのか</strong>

brk でヒープを拡張する方式は効率的ですが、大きなメモリを確保して解放すると、その領域がヒープの途中に残ってしまいます

mmap で別の領域に確保すれば、解放時に OS に返却できます

詳細は [brk(2)](https://man7.org/linux/man-pages/man2/brk.2.html) と [mmap(2)](https://man7.org/linux/man-pages/man2/mmap.2.html) を参照してください

### [確保と解放の責任](#allocation-deallocation-responsibility) {#allocation-deallocation-responsibility}

malloc() したメモリは、必ず free() で解放する責任があります

解放を忘れると、プログラムが終了するまでそのメモリは使われ続けます

これが<strong>メモリリーク</strong>です

---

## [スタックとヒープの比較](#stack-vs-heap-comparison) {#stack-vs-heap-comparison}

{: .labeled}
| 項目 | スタック | ヒープ |
| -------------------- | ----------------------------- | ------------------------------ |
| 確保のタイミング | 関数呼び出し時（自動） | malloc() 呼び出し時（明示的） |
| 解放のタイミング | 関数終了時（自動） | free() 呼び出し時（明示的） |
| 成長方向 | 高アドレス → 低アドレス | 低アドレス → 高アドレス |
| サイズ制限 | 比較的小さい（通常 8MB 程度） | 比較的大きい（物理メモリまで） |
| 速度 | 高速（ポインタ移動のみ） | 比較的遅い（空き領域を検索） |
| フラグメンテーション | 発生しない | 発生する可能性がある |

※ フラグメンテーション（断片化）：メモリの確保と解放を繰り返すことで、使えない小さな空き領域が増える現象

<strong>使い分けの指針</strong>

{: .labeled}
| 状況 | 適切な選択 |
| -------------------------------------- | ---------- |
| サイズが小さく、関数内で完結するデータ | スタック |
| サイズが大きいデータ | ヒープ |
| 寿命が関数を超えるデータ | ヒープ |
| サイズが実行時まで決まらないデータ | ヒープ |

---

## [よくある問題](#common-problems) {#common-problems}

### [スタックオーバーフロー](#stack-overflow) {#stack-overflow}

スタック領域を使い果たすと、<strong>スタックオーバーフロー</strong>が発生します

<strong>発生原因</strong>

- 深すぎる再帰呼び出し
- 大きすぎるローカル変数

```c
/* 危険な例：無限再帰 */
void infinite_recursion(void) {
    int large_array[10000];  /* スタックを消費 */
    infinite_recursion();     /* 自分を呼び出し続ける */
}
```

<strong>症状</strong>

- Segmentation fault（セグメンテーション違反）
- プログラムの異常終了

<strong>対策</strong>

- 深い再帰をループに書き換える
- 大きな配列はヒープに確保する

### [メモリリーク](#memory-leak) {#memory-leak}

malloc() で確保したメモリを free() し忘れると、<strong>メモリリーク</strong>が発生します

```c
/* 危険な例：解放し忘れ */
void leak_memory(void) {
    int *ptr = malloc(sizeof(int) * 1000);
    /* ptr を使った処理 */

    /* free(ptr) を忘れている */
    /* 関数終了時、ptr のアドレスが失われ、解放できなくなる */
}
```

<strong>症状</strong>

- メモリ使用量が徐々に増加
- 長時間動作するプログラムでメモリ不足

<strong>対策</strong>

- malloc() と free() を対にする
- 確保した場所の近くで解放する設計
- Valgrind などのツールで定期的にチェック

### [ダブルフリー](#double-free) {#double-free}

同じポインタを 2 回 free() すると、<strong>ダブルフリー</strong>が発生します

```c
/* 危険な例：二重解放 */
int *ptr = malloc(sizeof(int));
free(ptr);
free(ptr);  /* 未定義動作：クラッシュする可能性 */
```

<strong>対策</strong>

free() 後に NULL を代入します

```c
int *ptr = malloc(sizeof(int));
free(ptr);
ptr = NULL;  /* 次の free(ptr) は何もしない */
```

free(NULL) は安全で、何も起こりません

### [解放済みメモリへのアクセス](#use-after-free) {#use-after-free}

free() したメモリにアクセスすると、<strong>Use After Free</strong>（解放後使用）が発生します

```c
/* 危険な例：解放後にアクセス */
int *ptr = malloc(sizeof(int));
*ptr = 100;
free(ptr);
printf("%d\n", *ptr);  /* 未定義動作 */
```

<strong>対策</strong>

- free() 後に NULL を代入
- ポインタの有効性を確認してからアクセス

---

## [確認方法](#confirmation-methods) {#confirmation-methods}

### [/proc/\[pid\]/maps](#proc-pid-maps) {#proc-pid-maps}

実行中のプロセスのメモリマップを確認できます

```bash
cat /proc/self/maps
```

出力例：

```
55a1b2c3d000-55a1b2c3e000 r--p 00000000 08:01 12345  /path/to/program
55a1b2c3e000-55a1b2c3f000 r-xp 00001000 08:01 12345  /path/to/program
55a1b2c50000-55a1b2c71000 rw-p 00000000 00:00 0      [heap]
7fff12345000-7fff12366000 rw-p 00000000 00:00 0      [stack]
```

{: .labeled}
| フィールド | 説明 |
| -------------- | ---------------------------------------------------------- |
| アドレス範囲 | メモリ領域の開始-終了アドレス |
| パーミッション | r（読み取り）、w（書き込み）、x（実行）、p（プライベート） |
| [heap] | ヒープ領域 |
| [stack] | スタック領域 |

詳細は [proc_pid_maps(5)](https://man7.org/linux/man-pages/man5/proc_pid_maps.5.html) を参照してください

### [Valgrind によるメモリリーク検出](#valgrind-memory-leak-detection) {#valgrind-memory-leak-detection}

Valgrind はメモリエラーを検出するツールです

```bash
valgrind --leak-check=full ./program
```

出力例（リークがある場合）：

```
==12345== LEAK SUMMARY:
==12345==    definitely lost: 1,000 bytes in 1 blocks
==12345==    indirectly lost: 0 bytes in 0 blocks
==12345==      possibly lost: 0 bytes in 0 blocks
==12345==    still reachable: 0 bytes in 0 blocks
==12345==         suppressed: 0 bytes in 0 blocks
```

{: .labeled}
| カテゴリ | 意味 |
| --------------- | -------------------------------------- |
| definitely lost | 確実にリークしている |
| indirectly lost | リークしたメモリから参照されている |
| possibly lost | リークの可能性がある |
| still reachable | 終了時にまだ参照可能（通常は問題なし） |

詳細は [Valgrind Memcheck Manual](https://valgrind.org/docs/manual/mc-manual.html) を参照してください

### [スタックサイズの確認と変更](#checking-and-changing-stack-size) {#checking-and-changing-stack-size}

現在のスタックサイズ制限を確認します

```bash
ulimit -s
```

出力例：

```
8192
```

これは 8192KB（8MB）を意味します

一時的に変更する場合：

```bash
ulimit -s 16384  # 16MB に変更
```

---

## [まとめ](#summary) {#summary}

<strong>スタックとヒープの違い</strong>

{: .labeled}
| 項目 | スタック | ヒープ |
| ------------ | ---------------------- | --------------------- |
| 管理方法 | 自動 | 手動（malloc / free） |
| 確保関数 | なし（自動） | malloc() |
| 解放関数 | なし（自動） | free() |
| よくある問題 | スタックオーバーフロー | メモリリーク |

<strong>覚えておくこと</strong>

{: .labeled}
| ポイント | 理由 |
| --------------------------- | ------------------------------------ |
| malloc() したら必ず free() | メモリリークを防ぐ |
| 大きなデータはヒープに | スタックオーバーフローを防ぐ |
| free() 後は NULL を代入 | ダブルフリーと Use After Free を防ぐ |
| Valgrind で定期的にチェック | メモリリークを早期発見 |

---

## [参考資料](#references) {#references}

<strong>Linux マニュアル</strong>

- [malloc(3) - Linux manual page](https://www.man7.org/linux/man-pages/man3/malloc.3.html){:target="\_blank"}
  - malloc()、free()、calloc()、realloc() の使い方
- [brk(2) - Linux manual page](https://man7.org/linux/man-pages/man2/brk.2.html){:target="\_blank"}
  - ヒープ領域を拡張するシステムコール
- [mmap(2) - Linux manual page](https://man7.org/linux/man-pages/man2/mmap.2.html){:target="\_blank"}
  - 大きなメモリ確保で使われるシステムコール
- [proc_pid_maps(5) - Linux manual page](https://man7.org/linux/man-pages/man5/proc_pid_maps.5.html){:target="\_blank"}
  - /proc/[pid]/maps の読み方

<strong>ツール</strong>

- [Valgrind Memcheck Manual](https://valgrind.org/docs/manual/mc-manual.html){:target="\_blank"}
  - メモリリーク検出ツールの使い方

<strong>本編との関連</strong>

- [01-process](../../01-process/)
  - メモリの構造の概要
- [04-thread](../../04-thread/)
  - スレッドごとのスタック
