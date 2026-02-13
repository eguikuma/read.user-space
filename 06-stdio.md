---
layout: default
title: 標準入出力ライブラリ
---

# [06-stdio：標準入出力ライブラリ](#what-is-stdio) {#what-is-stdio}

ファイルディスクリプタを<strong>「便利に使う」</strong>ための仕組みを学びます

---

## [はじめに](#introduction) {#introduction}

前のトピック（[05-file-descriptor](../05-file-descriptor/)）では、OS がファイルを管理する仕組みを学びました

- open() でファイルを開き、fd（ファイルディスクリプタ）を取得する
- read() と write() でデータを読み書きする
- close() でファイルを閉じる

これらは OS が提供する<strong>低レベル</strong>な機能です

<strong>低レベル</strong>とは、OS に近い、より基本的な機能のことです

反対に、printf() のような便利な機能を提供するものを<strong>高レベル</strong>と呼びます

しかし、普段のプログラミングでは printf() や scanf() をよく使いますよね

実は、printf() は内部で write() を呼び出しています

では、なぜ printf() という「別の関数」が存在するのでしょうか？

### [なぜ read()/write() だけでは不十分なのか](#why-read-write-is-insufficient) {#why-read-write-is-insufficient}

<strong>もし write() だけでプログラムを書いたら？</strong>

```c
/* 数値を出力したい場合 */
int n = 42;

/* write() だけでは... */
char buf[20];
int len = 0;
/* 数値を文字列に変換するコードを自分で書く必要がある */
/* buf に "42" を格納して len = 2 にする処理... */
write(1, buf, len);

/* printf() なら */
printf("%d", n);  /* これだけ */
```

<strong>read()/write() だけの問題</strong>

{: .labeled}
| 問題 | 説明 |
| ---------------------- | ---------------------------------------------------- |
| フォーマット変換がない | 数値→文字列、文字列→数値の変換を自分で書く必要がある |
| バッファ管理が手動 | 1文字ずつ write() すると大量のシステムコールが発生 |
| 行単位の読み取りが面倒 | 改行を探しながら read() を繰り返す必要がある |

答えは「便利だから」です

このトピックでは、<strong>標準入出力ライブラリ（stdio）</strong>を学びます

stdio は、fd を「より便利に使う」ための仕組みです

---

## [日常の例え](#everyday-analogy) {#everyday-analogy}

stdio の仕組みを、「水道」に例えてみましょう

<strong>低レベル関数（write、read）</strong>

蛇口から直接バケツに水を汲むようなものです

バケツに水を入れるたびに、蛇口をひねる必要があります

1 文字書くたびに OS を呼び出すのは、毎回蛇口をひねるようなものです

<strong>stdio（printf、fwrite）</strong>

蛇口とバケツの間に「タンク」を置くようなものです

水はまずタンクに溜まり、タンクがいっぱいになったらバケツに流れます

これが<strong>バッファリング</strong>です

少量の水を何度も流すより、まとめて流す方が効率的です

同様に、少量のデータを何度も OS に渡すより、まとめて渡す方が効率的です

---

## [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

- <strong>FILE 構造体</strong>
  - fd を包み込んで、より使いやすくしたもの
- <strong>stdin、stdout、stderr</strong>
  - 標準入出力を表すストリーム（データの流れ）
- <strong>fopen()、fclose()</strong>
  - ファイルを開く・閉じる（高レベル版）
- <strong>printf()、fprintf()、sprintf()</strong>
  - フォーマット付き出力
- <strong>scanf()、fscanf()、sscanf()</strong>
  - フォーマット付き入力
- <strong>fgets()、fputs()</strong>
  - 行単位のテキスト読み書き
- <strong>fread()、fwrite()</strong>
  - バイナリデータの読み書き
- <strong>バッファリング</strong>
  - データをまとめて処理する仕組み
- <strong>fflush()、setvbuf()</strong>
  - バッファの制御

---

## [目次](#table-of-contents) {#table-of-contents}

1. [FILE 構造体とは何か](#what-is-file-struct)
2. [標準ストリーム](#standard-streams)
3. [fopen と open の違い](#difference-between-fopen-and-open)
4. [フォーマット付き入出力](#formatted-io)
5. [行単位の入出力](#line-io)
6. [バイナリ入出力](#binary-io)
7. [バッファリングの仕組み](#how-buffering-works)
8. [バッファリングの制御](#buffering-control)
9. [ファイル位置の制御](#file-position-control)
10. [エラー処理と EOF](#error-handling-and-eof)
11. [fd との相互変換](#mutual-conversion-with-fd)
12. [次のステップ](#next-steps)
13. [用語集](#glossary)
14. [参考資料](#references)

---

## [FILE 構造体とは何か](#what-is-file-struct) {#what-is-file-struct}

<strong>FILE</strong> は、stdio.h（標準入出力ライブラリのヘッダーファイル）で定義される<strong>構造体</strong>です

<strong>構造体</strong>とは、複数の関連するデータをまとめて1つの型として扱うものです

[05-file-descriptor](../05-file-descriptor/) で学んだ fd は、単なる整数（0、1、2、3...）でした

FILE は、fd に加えて以下の情報を持っています

- <strong>バッファ</strong>
  - データを一時的に溜めておく領域
- <strong>バッファ位置</strong>
  - バッファ内のどこまで読み書きしたか
- <strong>ファイル位置</strong>
  - ファイル内のどこまで読み書きしたか
- <strong>エラーフラグ</strong>
  - エラーが発生したかどうか
- <strong>EOF フラグ</strong>
  - ファイルの終端に達したかどうか

FILE は<strong>不透明な構造体（opaque type）</strong>です

「不透明」とは、内部の詳細が隠されているという意味です

FILE の中身がどうなっているかは公開されておらず、直接アクセスすることはできません

必ず stdio の関数（fopen、fclose、fprintf など）を通じて操作します

FILE を使うときは、FILE へのポインタ（FILE \*）を使います

<strong>ポインタ</strong>とは、データが置かれているメモリの場所（アドレス）を指し示す値です

```c
FILE *fp = fopen("example.txt", "r");
```

---

## [標準ストリーム](#standard-streams) {#standard-streams}

<strong>ストリーム</strong>とは、データの流れを抽象化したものです

「水が流れるパイプ」のようなイメージで、データを順番に読み書きします

プログラムが起動すると、3 つの FILE（ストリーム）が自動的に開かれます

{: .labeled}
| 名前 | 対応する fd | 説明 |
| ------ | ------------------ | ---------------------------- |
| stdin | 0（STDIN_FILENO） | 標準入力（通常はキーボード） |
| stdout | 1（STDOUT_FILENO） | 標準出力（通常は画面） |
| stderr | 2（STDERR_FILENO） | 標準エラー出力（通常は画面） |

STDIN_FILENO、STDOUT_FILENO、STDERR_FILENO は、unistd.h で定義されている定数です

これらは fd の値（0、1、2）を名前で表したものです

[05-file-descriptor](../05-file-descriptor/) で学んだ fd 0、1、2 が、FILE として使えるようになっています

```c
printf("Hello");           /* stdout に出力 */
fprintf(stdout, "Hello");  /* 同じ意味 */
fprintf(stderr, "Error");  /* stderr に出力 */
```

---

## [fopen と open の違い](#difference-between-fopen-and-open) {#difference-between-fopen-and-open}

ファイルを開く方法は 2 つあります

<strong>open()（低レベル）</strong>

05-file-descriptor で学んだ関数です

fd（整数）を返します

```c
int fd = open("file.txt", O_RDONLY);
```

<strong>fopen()（高レベル）</strong>

stdio の関数です

FILE\*（ポインタ）を返します

```c
FILE *fp = fopen("file.txt", "r");
```

fopen() では、ファイルの開き方を<strong>モード文字列</strong>（"r"、"w" などの短い文字列）で指定します

<strong>モード文字列と open フラグの対応</strong>

{: .labeled}
| fopen モード | open フラグ | 説明 |
| ------------ | ------------------------------- | ---------------------- |
| "r" | O_RDONLY | 読み取り専用 |
| "w" | O_WRONLY \| O_CREAT \| O_TRUNC | 書き込み専用（上書き） |
| "a" | O_WRONLY \| O_CREAT \| O_APPEND | 追記モード |
| "r+" | O_RDWR | 読み書き両用 |
| "w+" | O_RDWR \| O_CREAT \| O_TRUNC | 読み書き両用（上書き） |
| "a+" | O_RDWR \| O_CREAT \| O_APPEND | 読み書き両用（追記） |

fopen の方が簡単に使えます

ただし、fopen では O_EXCL（ファイルが存在したらエラー）などの細かい制御ができません

細かい制御が必要な場合は open を使います

### [open() と fopen() の使い分け](#when-to-use-open-vs-fopen) {#when-to-use-open-vs-fopen}

<strong>どちらを使うべきか迷ったときの指針</strong>を示します

#### [fopen() を使うべき場面](#when-to-use-fopen) {#when-to-use-fopen}

{: .labeled}
| 場面 | 理由 |
| -------------------------- | -------------------------------------------------- |
| テキストファイルの読み書き | printf、fprintf、fscanf が便利 |
| 設定ファイルの読み込み | fgets で行単位の処理が簡単 |
| ログファイルへの出力 | fprintf のフォーマット機能が便利 |
| CSV ファイルの処理 | fscanf でパース、fprintf で出力 |
| 移植性が重要なコード | stdio は C 標準なので<strong>移植性</strong>が高い |

<strong>移植性</strong>とは、異なる OS やコンピュータでも同じように動作することです

#### [open() を使うべき場面](#when-to-use-open) {#when-to-use-open}

{: .labeled}
| 場面 | 理由 |
| -------------------------------- | -------------------------------- |
| パイプやソケットの操作 | FILE は通常ファイル向けの設計 |
| O_EXCL でファイルの排他作成 | fopen では指定できない |
| O_SYNC で同期書き込み | OS がディスクに書き込むまで待つ |
| dup2() でリダイレクト | fd が必要 |
| mmap() でメモリマップ | ファイルをメモリに展開する |
| select() / poll() で多重化 | 複数の fd を同時に監視する |
| バッファリングを完全に制御したい | stdio のバッファが邪魔になる場合 |

これらの高度な技法は、このトピックでは扱いません

#### [一般的な推奨](#general-recommendation) {#general-recommendation}

迷ったら fopen() を使ってください

理由は以下の通りです

- バッファリングにより効率的
- フォーマット関数（printf、scanf）が使える
- エラー処理（ferror、feof）がわかりやすい
- C 標準なので移植性が高い

open() は、fopen() では実現できない場面でのみ使用してください

#### [混在させる場合の注意](#caution-when-mixing) {#caution-when-mixing}

同じファイルを open() と fopen() の両方で操作することは避けてください

バッファリングの不整合が発生する可能性があります

どうしても混在させる必要がある場合は、fileno() と fdopen() を使って相互変換してください（「fd との相互変換」セクションを参照）

---

## [フォーマット付き入出力](#formatted-io) {#formatted-io}

<strong>出力関数</strong>

{: .labeled}
| 関数 | 出力先 | 説明 |
| -------------------------------- | ------ | ---------------------------- |
| printf(format, ...) | stdout | 標準出力に出力 |
| fprintf(fp, format, ...) | FILE\* | 指定したファイルに出力 |
| sprintf(str, format, ...) | 文字列 | 文字列に出力 |
| snprintf(str, size, format, ...) | 文字列 | サイズを指定して文字列に出力 |

<strong>入力関数</strong>

{: .labeled}
| 関数 | 入力元 | 説明 |
| ------------------------ | ------ | ---------------------------- |
| scanf(format, ...) | stdin | 標準入力から読み取り |
| fscanf(fp, format, ...) | FILE\* | 指定したファイルから読み取り |
| sscanf(str, format, ...) | 文字列 | 文字列から読み取り |

<strong>主なフォーマット指定子</strong>

<strong>フォーマット指定子</strong>とは、printf() や scanf() で「どんな形式のデータか」を指定する記号です

% から始まり、続く文字でデータの種類を表します

{: .labeled}
| 指定子 | 型 | 説明 |
| ------ | ------------ | ------------------ |
| %d | int | 10 進整数 |
| %u | unsigned int | 符号なし 10 進整数 |
| %x | unsigned int | 16 進整数 |
| %f | double | 浮動小数点数 |
| %s | char\* | 文字列 |
| %c | char | 1 文字 |
| %p | void\* | ポインタ |
| %zu | size_t | サイズ |
| %% | - | % 文字そのもの |

---

## [行単位の入出力](#line-io) {#line-io}

<strong>入力関数</strong>

{: .labeled}
| 関数 | 説明 |
| -------------------- | ----------------------------------------------------------------------------- |
| fgets(buf, size, fp) | 改行または size-1 文字まで読み取り |
| gets(buf) | <strong>使用禁止</strong>（バッファオーバーフローの危険、C11 で標準から削除） |

<strong>バッファオーバーフロー</strong>とは、用意した領域を超えてデータが書き込まれることです

プログラムの異常動作やセキュリティ上の問題を引き起こします

gets() は入力サイズを制限できないため、C99 で非推奨となり、C11 で標準ライブラリから完全に削除されました

必ず fgets() を使用してください

fgets は<strong>改行文字</strong>（\n）も読み取ります

<strong>改行文字</strong>とは、Enter キーを押したときに入力される「行の終わり」を示す文字です

例えば "Hello\n" と入力すると、buf には "Hello\n\0" が格納されます

<strong>\0（NULL 文字）</strong>は、文字列の終端を示す特別な文字です

C 言語の文字列は、必ず \0 で終わります

<strong>出力関数</strong>

{: .labeled}
| 関数 | 説明 |
| -------------- | -------------------------------------- |
| fputs(str, fp) | 文字列を出力（改行は付けない） |
| puts(str) | 文字列を stdout に出力し、改行を付ける |

---

## [バイナリ入出力](#binary-io) {#binary-io}

<strong>テキストデータ</strong>は、人間が読める文字で構成されたデータです（例：ソースコード、設定ファイル）

<strong>バイナリデータ</strong>は、人間が直接読めない形式のデータです（例：画像、実行ファイル、構造体の内容）

ここでは、バイナリデータを読み書きする関数を紹介します

{: .labeled}
| 関数 | 説明 |
| ---------------------------- | ------------------------ |
| fread(buf, size, count, fp) | バイナリデータを読み取り |
| fwrite(buf, size, count, fp) | バイナリデータを書き込み |

<strong>引数の意味</strong>

- buf：データを格納するバッファ
- size：1 要素のサイズ（バイト）
- count：読み書きする要素数
- fp：ファイル

<strong>例：構造体の保存</strong>

```c
struct Person {
    char name[32];
    int age;
};

struct Person p = {"Alice", 25};

/* 書き込み */
fwrite(&p, sizeof(struct Person), 1, fp);

/* 読み取り */
fread(&p, sizeof(struct Person), 1, fp);
```

---

## [バッファリングの仕組み](#how-buffering-works) {#how-buffering-works}

<strong>なぜバッファリングが必要か</strong>

システムコール（write、read）は「高コスト」な操作です

<strong>ユーザー空間</strong>から<strong>カーネル空間</strong>への切り替えが発生するためです

これらの空間については [01-process](../01-process/) で詳しく説明しています

<strong>もしバッファリングなしで "Hello" を出力したら？</strong>

```c
/* バッファリングなし（直接 write） */
write(1, "H", 1);  /* システムコール 1回目 */
write(1, "e", 1);  /* システムコール 2回目 */
write(1, "l", 1);  /* システムコール 3回目 */
write(1, "l", 1);  /* システムコール 4回目 */
write(1, "o", 1);  /* システムコール 5回目 */
/* 5回のコンテキストスイッチ（ユーザー空間⇔カーネル空間の切り替え） */

/* バッファリングあり（stdio） */
printf("Hello");   /* バッファに溜まるだけ */
/* ... 後でまとめて 1回の write() で出力 */
```

5 文字出力するのに 5 回のシステムコールは無駄です

stdio は、データをバッファに溜めてから、まとめてシステムコールを呼び出します

<strong>バッファリングの種類</strong>

{: .labeled}
| 種類 | 定数 | 動作 | 典型的な使用場所 |
| ------------ | ------- | -------------------------------------------- | ---------------- |
| フルバッファ | \_IOFBF | バッファが一杯になったら出力 | ファイル |
| 行バッファ | \_IOLBF | 改行文字で出力、または端末からの入力時に出力 | 端末（stdout） |
| バッファなし | \_IONBF | 即座に出力 | stderr |

<strong>なぜ 3 種類必要なのか</strong>

{: .labeled}
| モード | なぜそのモードか |
| ------------ | ---------------------------------------------------------------------------- |
| フルバッファ | ファイルへの書き込みは効率が最優先<br>まとめて書くほど速い |
| 行バッファ | 対話的な端末では、改行ごとに出力を見たい<br>でも 1 文字ごとは遅すぎる |
| バッファなし | エラーメッセージは即座に見たい<br>クラッシュ直前のエラーも表示される必要がある |

<strong>stderr がバッファなしな理由</strong>

もし stderr がバッファリングされていたら、プログラムがクラッシュしたとき、バッファに溜まったエラーメッセージが表示されずに失われる可能性があります

デバッグ情報が消えてしまうのは困るので、stderr は常に即座に出力されます

これらの定数は stdio.h で定義されています

名前の由来は、\_IOFBF = I/O Full Buffering、\_IOLBF = I/O Line Buffering、\_IONBF = I/O No Buffering です

<strong>行バッファの重要な特性</strong>

行バッファは、以下の 2 つの条件でフラッシュ（出力）されます

1. 改行文字（\n）が出力されたとき
2. 端末に接続されたストリーム（通常は stdin）から入力を読み取ったとき

2 番目の条件により、printf() で改行なしのプロンプトを出力した後、scanf() で入力を待つと、プロンプトが自動的に表示されます

```c
printf("名前を入力: ");  /* 改行なし */
scanf("%s", name);       /* stdin からの読み取りで stdout がフラッシュされる */
```

<strong>バッファリングのデフォルト</strong>

- stdout（端末に接続）：行バッファ
- stdout（ファイルにリダイレクト）：フルバッファ
- stderr：バッファなし
- その他のファイル：フルバッファ

---

## [バッファリングの制御](#buffering-control) {#buffering-control}

<strong>fflush()</strong>

バッファの内容を強制的に出力します

```c
printf("処理中...");
fflush(stdout);  /* 即座に表示される */
sleep(2);
printf("完了\n");
```

fflush(NULL) を呼び出すと、すべての出力ストリームをフラッシュします

### [いつ fflush() が必要か](#when-fflush-is-needed) {#when-fflush-is-needed}

<strong>1. 改行なしの出力を即座に表示したいとき</strong>

```c
printf("進捗: ");
for (int i = 0; i <= 100; i += 10) {
    printf("%d%% ", i);
    fflush(stdout);  /* これがないと、ループ終了まで何も表示されない */
    sleep(1);
}
```

<strong>2. fork() の前</strong>

```c
printf("親プロセス");  /* バッファに溜まっている */
/* fflush(stdout); ← これを忘れると... */
pid_t pid = fork();    /* 子プロセスもバッファをコピーして持つ */
/* 親子両方で "親プロセス" が出力される可能性がある */
```

fork() はバッファごとコピーするため、fflush() しないと同じ内容が 2 回出力されることがあります

<strong>3. クラッシュ前のデバッグ出力</strong>

```c
printf("ここまで来た\n");
/* プログラムがクラッシュ */
```

フルバッファの場合、クラッシュ時にバッファの内容が失われる可能性があります

デバッグ時は fflush() を入れるか、stderr を使うと安全です

<strong>setvbuf()</strong>

バッファリングモードを変更します

```c
/* バッファなしモードに変更 */
setvbuf(stdout, NULL, _IONBF, 0);

/* 行バッファモードに変更 */
setvbuf(stdout, NULL, _IOLBF, 0);

/* フルバッファモードに変更（バッファサイズ指定） */
char buf[4096];
setvbuf(fp, buf, _IOFBF, sizeof(buf));
```

setvbuf はストリームを開いた直後、最初の入出力操作の前に呼び出す必要があります

---

## [ファイル位置の制御](#file-position-control) {#file-position-control}

{: .labeled}
| 関数 | 説明 |
| ------------------------- | ------------------------ |
| fseek(fp, offset, whence) | ファイル位置を移動 |
| ftell(fp) | 現在位置を取得 |
| rewind(fp) | 先頭に戻る |
| fgetpos(fp, pos) | 現在位置を取得（別形式） |
| fsetpos(fp, pos) | 位置を設定（別形式） |

<strong>fseek の whence</strong>

<strong>whence</strong> は「どこから」という意味の英語で、移動の基準位置を指定します

{: .labeled}
| 定数 | 説明 |
| -------- | ---------------- |
| SEEK_SET | ファイル先頭から |
| SEEK_CUR | 現在位置から |
| SEEK_END | ファイル末尾から |

[05-file-descriptor](../05-file-descriptor/) で学んだ lseek と同じ考え方です

---

## [エラー処理と EOF](#error-handling-and-eof) {#error-handling-and-eof}

<strong>エラーフラグと EOF フラグ</strong>

FILE には、エラーフラグと EOF フラグがあります

{: .labeled}
| 関数 | 説明 |
| ------------ | -------------------- |
| ferror(fp) | エラーフラグを確認 |
| feof(fp) | EOF フラグを確認 |
| clearerr(fp) | 両方のフラグをクリア |

<strong>EOF の検出</strong>

fgets は EOF またはエラー時に NULL を返します

fread は正常に読み取った要素数を返し、EOF またはエラー時は要求した数より少ない値（完全な EOF では 0）を返します

fread は EOF とエラーを区別しないため、ferror() と feof() で判別する必要があります

```c
while (fgets(buf, sizeof(buf), fp) != NULL) {
    /* 行を処理 */
}

if (feof(fp)) {
    /* 正常にファイル末尾に達した */
} else if (ferror(fp)) {
    /* エラーが発生した */
}
```

---

## [fd との相互変換](#mutual-conversion-with-fd) {#mutual-conversion-with-fd}

stdio と低レベル関数を組み合わせて使いたい場合があります

<strong>fileno()：FILE\* から fd を取得</strong>

```c
FILE *fp = fopen("file.txt", "r");
int fd = fileno(fp);  /* fd を取得 */
```

<strong>fdopen()：fd から FILE\* を作成</strong>

```c
int fd = open("file.txt", O_RDONLY);
FILE *fp = fdopen(fd, "r");  /* FILE* を作成 */
```

<strong>注意点</strong>

fdopen で作成した FILE を fclose すると、内部の fd も閉じられます

fd を別途 close する必要はありません（二重 close はエラーになります）

---

## [次のステップ](#next-steps) {#next-steps}

このトピックでは、「標準入出力ライブラリ」を学びました

- FILE 構造体と fd の関係
- fopen、printf、fread などの関数
- バッファリングの仕組みと制御

次の [07-ipc](../07-ipc/) では、プロセス間通信を学びます

- パイプは fd のペアで実現されている
- 05-file-descriptor で学んだ fd の知識が活きる
- 06-stdio で学んだバッファリングの注意点も重要

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 英語 | 説明 |
| ---------------------- | ------------------- | ------------------------------------------------ |
| ストリーム | Stream | データの流れを抽象化したもの |
| 構造体 | Struct | 複数のデータをまとめて1つの型として扱うもの |
| 不透明な構造体 | Opaque Type | 内部構造が隠されており、直接アクセスできないもの |
| ポインタ | Pointer | メモリの場所（アドレス）を指し示す値 |
| 低レベル関数 | Low-level Function | OS に近い基本的な関数（write、read など） |
| 高レベル関数 | High-level Function | 低レベル関数を便利にした関数（printf など） |
| バッファ | Buffer | データを一時的に溜めておく領域 |
| バッファリング | Buffering | データをバッファに溜めてまとめて処理すること |
| フルバッファ | Full Buffering | バッファが一杯になったら出力するモード |
| 行バッファ | Line Buffering | 改行または端末入力時に出力するモード |
| フラッシュ | Flush | バッファの内容を強制的に出力すること |
| 移植性 | Portability | 異なる環境でも同じように動作すること |
| モード文字列 | Mode String | fopen() でファイルの開き方を指定する文字列 |
| フォーマット指定子 | Format Specifier | printf 等で使う %d、%s など |
| 改行文字 | Newline | 行の終わりを示す文字（\n） |
| NULL 文字 | Null Character | 文字列の終端を示す文字（\0） |
| バッファオーバーフロー | Buffer Overflow | バッファの容量を超えてデータが書き込まれること |
| テキストデータ | Text Data | 人間が読める文字で構成されたデータ |
| バイナリデータ | Binary Data | 人間が直接読めない形式のデータ |
| EOF | End Of File | ファイルの終端 |
| whence | Whence | 「どこから」の意味、位置の基準を指定する引数名 |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

- [stdio(3) - Linux manual page](https://man7.org/linux/man-pages/man3/stdio.3.html){:target="\_blank"}
  - 標準入出力ライブラリの概要
- [fopen(3) - Linux manual page](https://man7.org/linux/man-pages/man3/fopen.3.html){:target="\_blank"}
  - ファイルを開く
- [fclose(3) - Linux manual page](https://man7.org/linux/man-pages/man3/fclose.3.html){:target="\_blank"}
  - ファイルを閉じる
- [printf(3) - Linux manual page](https://man7.org/linux/man-pages/man3/printf.3.html){:target="\_blank"}
  - フォーマット付き出力
- [scanf(3) - Linux manual page](https://man7.org/linux/man-pages/man3/scanf.3.html){:target="\_blank"}
  - フォーマット付き入力
- [fread(3) - Linux manual page](https://man7.org/linux/man-pages/man3/fread.3.html){:target="\_blank"}
  - バイナリ読み取り
- [fwrite(3) - Linux manual page](https://man7.org/linux/man-pages/man3/fwrite.3.html){:target="\_blank"}
  - バイナリ書き込み
- [fgets(3) - Linux manual page](https://man7.org/linux/man-pages/man3/fgets.3.html){:target="\_blank"}
  - 行単位読み取り
- [fputs(3) - Linux manual page](https://man7.org/linux/man-pages/man3/fputs.3.html){:target="\_blank"}
  - 行単位書き込み
- [fflush(3) - Linux manual page](https://man7.org/linux/man-pages/man3/fflush.3.html){:target="\_blank"}
  - バッファをフラッシュ
- [setvbuf(3) - Linux manual page](https://man7.org/linux/man-pages/man3/setvbuf.3.html){:target="\_blank"}
  - バッファリングの制御
- [setbuf(3) - Linux manual page](https://man7.org/linux/man-pages/man3/setbuf.3.html){:target="\_blank"}
  - バッファリングの制御（行バッファの詳細動作）
- [fseek(3) - Linux manual page](https://man7.org/linux/man-pages/man3/fseek.3.html){:target="\_blank"}
  - ファイル位置の移動
- [ftell(3) - Linux manual page](https://man7.org/linux/man-pages/man3/ftell.3.html){:target="\_blank"}
  - 現在位置の取得
- [ferror(3) - Linux manual page](https://man7.org/linux/man-pages/man3/ferror.3.html){:target="\_blank"}
  - エラーフラグの確認
- [feof(3) - Linux manual page](https://man7.org/linux/man-pages/man3/feof.3.html){:target="\_blank"}
  - EOF フラグの確認
- [fileno(3) - Linux manual page](https://man7.org/linux/man-pages/man3/fileno.3.html){:target="\_blank"}
  - FILE\* から fd を取得
- [fdopen(3) - Linux manual page](https://man7.org/linux/man-pages/man3/fdopen.3.html){:target="\_blank"}
  - fd から FILE\* を作成
