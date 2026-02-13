---
layout: default
title: read.user-space
---

# [read.user-space](#read-user-space) {#read-user-space}

<strong>「OS がプログラムを動かす仕組み」</strong>を、使う側の視点で学びます

---

## [このリポジトリは何のためにあるのか](#what-is-this-repository-for) {#what-is-this-repository-for}

パソコンやスマートフォンで、私たちは毎日たくさんのアプリを使っています

ブラウザでウェブサイトを見たり、音楽を聴いたり、ゲームをしたり

でも、ちょっと考えてみてください

「アプリを起動する」とき、パソコンの中では<strong>何が起きている</strong>のでしょうか？

このリポジトリでは、その「裏側」を学びます

---

## [「ユーザー空間」とは何か](#what-is-user-space) {#what-is-user-space}

コンピュータの世界には、大きく分けて2つの「空間」があります

### [<strong>1. カーネル空間（Kernel Space）</strong>](#kernel-space) {#kernel-space}

OS（オペレーティングシステム）の心臓部が動いている場所です

ハードウェア（CPU、メモリ、ディスクなど）を直接操作できます

※ <strong>ハードウェア</strong>とは、コンピュータを構成する物理的な部品のことです

- <strong>CPU</strong>
  - プログラムの命令を実際に処理する装置です
- <strong>メモリ</strong>（RAM）
  - パソコンが作業中のデータを一時的に置いておく場所です
- <strong>ディスク</strong>（SSD や HDD）
  - データを永続的に保存する場所です

一般のプログラムはここに直接触れることができません

### [<strong>2. ユーザー空間（User Space）</strong>](#user-space) {#user-space}

私たちが普段使うプログラム（アプリ）が動いている場所です

ブラウザ、エディタ、ゲームなど、すべてここで動いています

カーネル空間に直接アクセスすることはできず、「システムコール」という仕組みを使って OS に頼みます

※ <strong>システムコール</strong>とは、プログラムが OS に「これをやって」とお願いする仕組みです

例えば「ファイルを開いて」「メモリをください」といった依頼を、システムコールを通じて OS に伝えます

このリポジトリでは、<strong>ユーザー空間</strong>に焦点を当てます

つまり、「プログラムが OS とどうやりとりして動いているか」を学びます

---

## [なぜこれを学ぶのか](#why-learn-this) {#why-learn-this}

プログラミングを学ぶとき、多くの人は「書き方」から始めます

「for ループはこう書く」「関数はこう定義する」など

でも、「なぜそう動くのか」を知らないと、いつか壁にぶつかります

例えば

- プログラムが「フリーズ」（固まって動かなくなること）したとき、何が起きているのか？
- 「メモリ不足」というエラーは、具体的に何が足りないのか？
- ターミナルで `Ctrl+C`を押すと、なぜプログラムが止まるのか？

これらの疑問に答えるには、OS の仕組みを知る必要があります

---

## [このリポジトリで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

{: .labeled}
| 順番 | トピック | 学ぶこと |
| ---- | ------------------------------------------ | ------------------------------------------------- |
| 01 | [process](./01-process/) | プログラムが「動いている状態」とは何か |
| 02 | [fork-exec](./02-fork-exec/) | 新しいプログラムを起動する仕組み |
| 03 | [signal](./03-signal/) | プログラムに「合図」を送る仕組み |
| 04 | [thread](./04-thread/) | 1つのプログラムの中で複数の処理を同時に行う仕組み |
| 05 | [file-descriptor](./05-file-descriptor/) | OS がファイルやデバイスを扱う方法 |
| 06 | [stdio](./06-stdio/) | プログラムの「入力」と「出力」 |
| 07 | [ipc](./07-ipc/) | プロセス間でデータをやり取りする仕組み |

---

## [前提知識](#prerequisites) {#prerequisites}

このリポジトリを始めるために、特別な知識は必要ありません

ただし、以下ができると学習がスムーズです

- ターミナル（コマンドを入力する黒い画面）で簡単なコマンドを打てる
  - `ls`、`cd`、`cat` など
- 何かしらのプログラミング言語を少し触ったことがある
  - どの言語でも OK です

C 言語の知識がなくても始められます

C 言語自体を深く学ぶことが目的ではなく、コード例を読み解くのに必要な最低限の知識は、その都度説明します

---

## [なぜ C 言語を使うのか](#why-c-language) {#why-c-language}

このリポジトリのコード例は C 言語で書かれています

「なぜ Python や JavaScript ではないのか？」と思うかもしれません

理由は3つあります

### [<strong>1. Linux 自体が C で書かれている</strong>](#linux-written-in-c) {#linux-written-in-c}

OS の「言葉」を直接使えるということです

OS が提供する機能を、最も素直な形で学べます

C 言語では、`getpid()` や `fork()` といったシステムコールを直接呼び出せます

「プログラムが OS とどうやりとりしているか」を、仲介なしに見ることができます

### [<strong>2. 抽象化されていない</strong>](#not-abstracted) {#not-abstracted}

Python や JavaScript は、OS の機能を「便利にラップ」しています

つまり、裏で何が起きているか見えにくくなっています

C 言語では、OS の機能がそのまま見えます

### [<strong>3. 「なぜ他の言語は楽なのか」がわかる</strong>](#why-other-languages-are-easier) {#why-other-languages-are-easier}

C 言語でメモリ管理の大変さを体験すると、Python の「自動でやってくれる便利さ」が理解できます

---

## [C 言語の読み方](#how-to-read-c) {#how-to-read-c}

このリポジトリのコード例を読むために、最低限知っておくと便利な C 言語の知識をまとめます

C 言語を深く学ぶ必要はありません

「こういう意味なんだな」と分かれば十分です

### [<strong>1. プログラムの始まり：main 関数</strong>](#main-function) {#main-function}

```c
int main(int argc, char *argv[]) {
    /* ここにプログラムの処理を書く */
    return 0;
}
```

C 言語のプログラムは、必ず `main` という名前の関数から始まります

- <strong>int</strong>
  - この関数が整数（数値）を返すことを示します
  - `return 0;` で「正常終了」を OS に伝えます
- <strong>argc</strong>
  - コマンドライン引数の数です
  - 例：`./program hello world` なら argc は 3（program、hello、world）
- <strong>argv</strong>
  - コマンドライン引数の中身です
  - 例：`argv[0]` は "program"、`argv[1]` は "hello"

<strong>具体例で理解する argc と argv</strong>

ターミナルで以下のようにコマンドを実行したとします

```bash
./greeting Alice Bob
```

このとき、argc と argv は以下のようになります

{: .labeled}
| 変数 | 値 | 説明 |
| ------- | ------------ | ------------------------------ |
| argc | 3 | 引数の数（プログラム名を含む） |
| argv[0] | "./greeting" | プログラム名 |
| argv[1] | "Alice" | 1番目の引数 |
| argv[2] | "Bob" | 2番目の引数 |
| argv[3] | NULL | 終端を示す |

<strong>身近な例</strong>

普段使うコマンドも同じ仕組みです

```bash
ls -la /home
```

{: .labeled}
| 変数 | 値 |
| ------- | ------- |
| argc | 3 |
| argv[0] | "ls" |
| argv[1] | "-la" |
| argv[2] | "/home" |

ls コマンドは、argv[1] を見て「オプションだ」と判断し、argv[2] を見て「このディレクトリを表示する」と判断しています

このリポジトリのコード例では argc と argv を使わないこともあります

その場合、`(void)argc;` のように書いて「使わない」ことを明示しています

### [<strong>2. 型とは何か</strong>](#types) {#types}

C 言語では、すべての変数に「型」があります

型とは、その変数に入るデータの種類を示すものです

{: .labeled}
| 型 | 説明 | 例 |
| ----- | ---------------------- | -------------------- |
| int | 整数 | 42、-10、0 |
| char | 1 文字 | 'A'、'z'、'0' |
| float | 小数 | 3.14、-0.5 |
| void | 「何もない」ことを示す | 戻り値がない関数など |

OS が使う特別な型もあります

{: .labeled}
| 型 | 説明 |
| ------- | -------------------------------- |
| pid_t | プロセス ID を格納する型 |
| size_t | サイズを格納する型 |
| ssize_t | サイズ（負の値も可）を格納する型 |

これらは OS によってサイズが異なる可能性があるため、専用の型が用意されています

<strong>なぜ int ではなく size_t / ssize_t なのか</strong>

「サイズを表すなら int でいいのでは？」と思うかもしれません

専用の型を使う理由が3つあります

<strong>1. 32ビットと64ビットの違い</strong>

{: .labeled}
| 環境 | int の範囲 | size_t の範囲 |
| -------- | ---------- | ------------------- |
| 32ビット | 約 ±21億 | 0 ～ 約42億 |
| 64ビット | 約 ±21億 | 0 ～ 非常に大きな数 |

64ビット環境では、4GB を超えるファイルを扱えます

int のままでは、大きなサイズを表現できません

<strong>2. 符号の有無</strong>

{: .labeled}
| 型 | 符号 | 用途 |
| ------- | -------------------- | --------------------------------------- |
| size_t | 符号なし（0 以上） | バッファサイズ、配列の長さ |
| ssize_t | 符号あり（負の値可） | 読み書きのバイト数（-1 でエラーを示す） |

read() や write() は、失敗時に -1 を返します

そのため、戻り値は ssize_t（符号付き）です

```c
ssize_t bytes_read = read(fd, buffer, sizeof(buffer));
if (bytes_read == -1) {
    /* エラー処理 */
}
```

<strong>3. ポータビリティ</strong>

専用の型を使うことで、異なる OS やアーキテクチャでも正しく動作します

「この変数はサイズを表す」という意図がコードから明確に伝わります

### [<strong>3. #include と #define</strong>](#include-and-define) {#include-and-define}

<strong>#include</strong> は、他のファイルの内容を取り込む命令です

```c
#include <stdio.h>  /* 標準入出力（printf など）を使うため */
#include <unistd.h> /* OS の機能（getpid など）を使うため */
```

Python の `import` や JavaScript の `require` に似ています

<strong>#define</strong> は、名前に値を割り当てる命令です

```c
#define MAX_SIZE 100
```

この場合、コード中の `MAX_SIZE` がすべて `100` に置き換わります

変数との違いは、コンパイル時（プログラムを実行形式に変換するとき）に置換されることです

### [<strong>4. ポインタの基礎</strong>](#pointer-basics) {#pointer-basics}

ポインタは C 言語で最も難しい概念の1つですが、基本だけ押さえれば読めます

<strong>ポインタとは</strong>

メモリ上の「住所」を指し示す値です

変数がマンションの「部屋」だとすると、ポインタはその「部屋番号」です

<strong>記号の意味</strong>

{: .labeled}
| 記号 | 意味 | 例 |
| ---- | ----------------------------------------- | ------------------------------------------------- |
| \* | ポインタ型を示す / ポインタが指す値を取得 | `int *p;`（int へのポインタ）、`*p`（p が指す値） |
| & | 変数のアドレス（住所）を取得 | `&x`（x のアドレス） |

<strong>例</strong>

```c
int x = 10;      /* x という部屋に 10 が入っている */
int *p = &x;     /* p には x の部屋番号が入る */
printf("%d", *p); /* p が指す部屋の中身（10）を表示 */
```

このリポジトリでは、ポインタの複雑な操作はほとんど出てきません

「`*` が付いていたらアドレスを扱っている」程度の理解で十分です

<strong>void \* 型（汎用ポインタ）</strong>

`void *` は、任意の型のアドレスを格納できる特別なポインタ型です

```c
void *p;      /* どんな型のアドレスでも入れられる */
int *num_ptr;
char *str_ptr;

p = num_ptr;  /* int * を代入できる */
p = str_ptr;  /* char * も代入できる */
```

<strong>なぜ void \* が必要か</strong>

「どんな型のデータでも受け取れる関数」を作るときに使います

例えば、スレッドを作る関数 `pthread_create()` は、スレッドに渡すデータの型を事前に決められません

そのため、引数の型を `void *` にして、「何でも渡せる」ようにしています

<strong>使用時の注意</strong>

`void *` から実際の型として使うには、型変換（キャスト）が必要です

```c
void *p = &x;
int *num_ptr = (int *)p;  /* void * を int * に変換 */
```

<strong>ポインタのポインタ</strong>

関数に「ポインタ自体を変更してもらいたい」ときは、ポインタのポインタ（`**`）を使います

```c
void *result;
pthread_join(thread, &result);  /* result のアドレスを渡す */
```

`&result` は「result 変数のアドレス」です

関数内で `result` の値を変更できるようにするため、アドレスを渡しています

### [<strong>5. 構造体</strong>](#structs) {#structs}

構造体は、複数の関連するデータをまとめて1つの型として定義する仕組みです

<strong>基本的な形</strong>

```c
typedef struct {
    int number;
    const char *name;
} Person;
```

この例では、`number`（整数）と `name`（文字列）をまとめた `Person` という型を作っています

<strong>typedef について</strong>

`typedef` を付けることで、`struct` を省略して型名だけで使えるようになります

```c
Person p;  /* typedef があるとこう書ける */
```

`typedef` がない場合は、毎回 `struct Person p;` と書く必要があります

<strong>使い方</strong>

```c
Person p;
p.number = 1;
p.name = "Alice";
```

ドット（`.`）を使って、構造体の中のデータにアクセスします

<strong>ポインタ経由のアクセス：-> 演算子</strong>

構造体へのポインタがある場合、`->` 演算子を使ってメンバーにアクセスします

```c
Person p;
p.number = 1;

Person *ptr = &p;       /* p へのポインタ */
ptr->number = 2;        /* ポインタ経由でメンバーにアクセス */
```

`ptr->number` は `(*ptr).number` と同じ意味です

つまり、「ポインタが指す先の構造体の中の number」ということです

<strong>なぜ -> が必要か</strong>

`*ptr.number` と書くと、C 言語の優先順位のルールにより `*(ptr.number)` と解釈されてしまいます

`->` 演算子は、この問題を避けてコードを読みやすくするためのものです

<strong>. と -> の使い分け</strong>

{: .labeled}
| 状況 | 書き方 | 例 |
| ------------------------------ | ------ | ------------- |
| 構造体変数から直接アクセス | `.` | `person.name` |
| 構造体へのポインタからアクセス | `->` | `ptr->name` |

このリポジトリでは、システムコールの結果を格納する構造体へのポインタを扱うときに `->` が登場します

### [<strong>6. エラー処理の基礎：errno と perror()</strong>](#error-handling-basics) {#error-handling-basics}

システムコールやライブラリ関数が失敗したとき、C 言語では <strong>errno</strong> という仕組みでエラーの種類を知ることができます

<strong>errno とは</strong>

エラーの種類を示す番号が設定されるグローバル変数です

`<errno.h>` ヘッダーで定義されています

例えば、ファイルを開こうとして失敗したとき、errno には「ファイルが見つからない」「権限がない」などを示す番号が設定されます

<strong>perror() とは</strong>

errno の値に対応するエラーメッセージを表示する関数です

```c
#include <stdio.h>
#include <errno.h>

perror("ファイルを開けませんでした");
```

引数に渡した文字列の後に、errno に対応するエラーメッセージが表示されます

<strong>出力例</strong>

```
ファイルを開けませんでした：No such file or directory
```

<strong>使い方のパターン</strong>

```c
if (open("存在しないファイル", O_RDONLY) == -1) {
    perror("open() に失敗しました");
    /* 適切なエラー処理 */
}
```

システムコールが失敗したときは、戻り値が -1 になることが多いです

そのタイミングで perror() を呼ぶと、何が原因で失敗したのかがわかります

### [<strong>7. プログラムの終了：exit() と EXIT_SUCCESS / EXIT_FAILURE</strong>](#program-exit) {#program-exit}

プログラムを終了させる方法には、`return` と `exit()` の2つがあります

<strong>return と exit() の違い</strong>

{: .labeled}
| 方法 | 動作 |
| ------ | --------------------------------------------------- |
| return | 現在の関数から戻る（main() の場合はプログラム終了） |
| exit() | どこから呼んでもプログラム全体を即座に終了する |

<strong>exit() の使い方</strong>

```c
#include <stdlib.h>

exit(EXIT_SUCCESS);  /* 正常終了 */
exit(EXIT_FAILURE);  /* 異常終了 */
```

`exit()` は `<stdlib.h>` ヘッダーで定義されています

<strong>EXIT_SUCCESS と EXIT_FAILURE</strong>

{: .labeled}
| 定数 | 意味 | 値（通常） |
| ------------ | -------- | ---------- |
| EXIT_SUCCESS | 正常終了 | 0 |
| EXIT_FAILURE | 異常終了 | 1 |

シェルから終了コードを確認できます

```bash
./program
echo $?  # 直前のプログラムの終了コードを表示
```

<strong>使い分け</strong>

- main() 関数内で終了するなら `return 0;` や `return EXIT_SUCCESS;`
- main() 以外の関数からプログラムを終了したいなら `exit()`

### [<strong>8. ビット演算の基礎</strong>](#bitwise-operations-basics) {#bitwise-operations-basics}

C 言語では、数値をビット（0 と 1 の並び）として操作できます

このリポジトリで特に重要なのは、<strong>ビット OR 演算</strong>（`|`）です

<strong>ビット OR 演算とは</strong>

2つの数値の各ビットを比較し、どちらかが 1 なら 1 にする演算です

```
  0101  (5)
| 0011  (3)
------
  0111  (7)
```

<strong>フラグを組み合わせる</strong>

システムコールでは、複数のオプションを指定するときにビット OR を使います

```c
int fd = open("file.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
```

この例では、以下の3つのオプションを同時に指定しています

{: .labeled}
| フラグ | 意味 |
| -------- | ------------------------------ |
| O_WRONLY | 書き込み専用で開く |
| O_CREAT | ファイルがなければ作成する |
| O_TRUNC | ファイルがあれば中身を空にする |

<strong>なぜ足し算ではなく OR なのか</strong>

フラグは、各ビットが独立した意味を持つように設計されています

```
O_WRONLY = 0001
O_CREAT  = 0100
O_TRUNC  = 1000
```

ビット OR で組み合わせると

```
0001 | 0100 | 1000 = 1101
```

足し算でも同じ結果になりますが、OR を使う理由は

- 同じフラグを2回指定しても問題にならない（OR なら結果が変わらない）
- 「フラグを組み合わせる」という意図がコードから明確に伝わる

### [<strong>9. 標準エラー出力：stderr と fprintf()</strong>](#stderr-and-fprintf) {#stderr-and-fprintf}

プログラムの出力には、<strong>標準出力（stdout）</strong>と<strong>標準エラー出力（stderr）</strong>の2つがあります

<strong>stdout と stderr の違い</strong>

{: .labeled}
| 出力先 | 用途 | リダイレクト |
| ------ | ---------------- | -------------------- |
| stdout | 通常の出力 | `>` で転送される |
| stderr | エラーメッセージ | `>` では転送されない |

<strong>例</strong>

```bash
./program > output.txt
```

この場合、stdout はファイルに書き込まれますが、stderr は画面に表示されたままです

エラーメッセージを見逃さないための仕組みです

<strong>fprintf() とは</strong>

出力先を指定して出力する関数です

```c
#include <stdio.h>

printf("通常のメッセージ\n");           /* stdout に出力 */
fprintf(stderr, "エラーメッセージ\n");  /* stderr に出力 */
```

`printf()` は常に stdout に出力しますが、`fprintf()` は第1引数で出力先を指定できます

<strong>使い分け</strong>

- 通常のメッセージ → `printf()` または `fprintf(stdout, ...)`
- エラーメッセージ → `fprintf(stderr, ...)`

---

## [参考資料](#references) {#references}

このリポジトリの内容は、以下のソースに基づいています

各トピックで使用する個別の man ページは、各トピックドキュメントに記載しています

- [Linux man-pages](https://man7.org/linux/man-pages/){:target="\_blank"}
  - Linux システムコールやライブラリ関数の公式マニュアル
- [POSIX.1-2017](https://pubs.opengroup.org/onlinepubs/9699919799/){:target="\_blank"}
  - UNIX 系 OS の標準仕様
- [GNU C Library Manual](https://www.gnu.org/software/libc/manual/){:target="\_blank"}
  - C 言語の標準ライブラリの公式マニュアル
