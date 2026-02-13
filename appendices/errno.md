<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# ENOENTとは何か

## はじめに

本編でファイル操作のコードを見ていると、こんなエラーメッセージに出会うことがあります

```
No such file or directory
```

このエラーメッセージは、内部的には <strong>ENOENT</strong> という番号で表されています

では、ENOENT とは何でしょうか

そして、エラー番号はどこに格納されているのでしょうか

このドキュメントでは、C 言語のエラー処理の基本である <strong>errno</strong> の仕組みを説明します

---

## 目次

- [ENOENTの意味](#enoentの意味)
- [errnoの仕組み](#errnoの仕組み)
- [よく見るエラー](#よく見るエラー)
- [調べ方](#調べ方)
- [まとめ](#まとめ)
- [参考資料](#参考資料)

---

## ENOENTの意味

### 名前の由来

<strong>ENOENT</strong> は <strong>E</strong>rror <strong>NO ENT</strong>ry の略です

「エントリがない」= ディレクトリ内にそのファイル名が存在しない、という意味です

Linux x86/ARM では、ENOENT の値は <strong>2</strong> です

### 発生する場面

ENOENT は、主に以下の場面で発生します

| 場面                                 | 例                                        |
| ------------------------------------ | ----------------------------------------- |
| 存在しないファイルを開こうとした     | `open("/nonexistent.txt", O_RDONLY)`      |
| パスの途中のディレクトリが存在しない | `open("/no/such/dir/file.txt", O_RDONLY)` |
| 存在しないファイルを削除しようとした | `unlink("/nonexistent.txt")`              |

### コード例

存在しないファイルを開くと、ENOENT が発生します

```c
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>

int main(void) {
    int fd = open("/nonexistent/file.txt", O_RDONLY);

    if (fd == -1) {
        printf("errno = %d\n", errno);           /* 2 */
        printf("ENOENT = %d\n", ENOENT);         /* 2 */
        printf("エラー: %s\n", strerror(errno)); /* No such file or directory */
    }

    return 0;
}
```

実行結果は以下のようになります

```
errno = 2
ENOENT = 2
エラー: No such file or directory
```

---

## errnoの仕組み

### errnoとは

<strong>errno</strong> は、システムコールやライブラリ関数がエラーを起こしたとき、その理由を示す番号が格納される変数です

`<errno.h>` ヘッダーをインクルードすることで使用できます

```c
#include <errno.h>
```

### なぜ errno という仕組みが必要なのか

C 言語では、関数は 1 つの値しか返せません

```c
/* 成功時は 0 以上の fd、失敗時は -1 を返す */
int fd = open("file.txt", O_RDONLY);
```

<strong>もし戻り値だけでエラーを伝えようとしたら？</strong>

- 「ファイルが存在しない」「権限がない」「ディスクが一杯」を区別できない
- すべて -1 としか分からない

<strong>代替案と問題点</strong>

| 代替案                   | 問題点                     |
| ------------------------ | -------------------------- |
| エラー番号を戻り値にする | 正常な戻り値と区別が難しい |
| 引数でエラーを返す       | 全関数の引数が増える       |
| 構造体で返す             | C 言語の制約で使いにくい   |

errno という「副作用で設定される変数」が、C 言語の制約の中での現実的な解決策でした

この設計は UNIX の初期（1970年代）から使われています

### スレッドローカル

歴史的には errno はグローバル変数でしたが、現代の実装では<strong>スレッドローカル</strong>になっています

Linux の公式マニュアルには、こう書かれています

> errno is thread-local; setting it in one thread does not affect its value in any other thread.

> errno はスレッドローカルです
>
> あるスレッドで設定しても、他のスレッドの値には影響しません

これにより、マルチスレッド環境でも各スレッドが独自のエラー情報を持つことができます

本編 [04-thread](../04-thread.md) でスレッドを学ぶ際に、この特性が重要になります

### いつ設定されるか

errno は、関数が<strong>失敗したとき</strong>に設定されます

重要な注意点があります

| 状況                 | errno の値                                                        |
| -------------------- | ----------------------------------------------------------------- |
| 関数が失敗した       | エラー番号が設定される                                            |
| 関数が成功した       | 値は<strong>未定義</strong>（変更されないことが多いが保証はない） |
| 別の関数を呼び出した | <strong>上書きされる可能性</strong>がある                         |

そのため、エラー処理は<strong>失敗した直後</strong>に行う必要があります

### 正しい使い方

<strong>なぜ失敗直後に errno を保存すべきか</strong>

多くの関数が内部で errno を変更します

例えば、printf() も内部で write() を呼ぶため、errno が上書きされる可能性があります

```c
/* 危険な例 */
int fd = open("file.txt", O_RDONLY);
if (fd == -1) {
    printf("ファイルを開けませんでした\n");  /* この中で errno が変わる可能性 */
    if (errno == ENOENT) {  /* ここの errno は open() の結果とは限らない */
        /* ... */
    }
}
```

<strong>安全な書き方</strong>

```c
int fd = open("file.txt", O_RDONLY);
if (fd == -1) {
    /* 失敗した直後に errno を確認する */
    int saved_errno = errno;  /* 保存しておくと安全 */

    /* ログ出力などで別の関数を呼ぶと errno が上書きされる可能性がある */
    printf("ログ出力中...\n");

    /* 保存した値を使う */
    printf("エラー: %s\n", strerror(saved_errno));
}
```

### やってはいけない使い方

```c
int fd = open("file.txt", O_RDONLY);
/* ここで errno を確認していない */

printf("何か処理...\n");  /* この関数が errno を変更するかもしれない */

/* この時点の errno は、open() のエラーとは限らない */
if (errno == ENOENT) {  /* 危険！ */
    printf("ファイルがありません\n");
}
```

---

## よく見るエラー

システムプログラミングでよく遭遇するエラー番号を紹介します

### ファイル・ディレクトリ関連

| エラー名 | 番号 | 意味                      | 発生する場面                   |
| -------- | ---- | ------------------------- | ------------------------------ |
| ENOENT   | 2    | No such file or directory | ファイルが存在しない           |
| EACCES   | 13   | Permission denied         | 権限がない                     |
| EEXIST   | 17   | File exists               | O_EXCL でファイルが既に存在    |
| ENOTDIR  | 20   | Not a directory           | パスの途中がディレクトリでない |
| EISDIR   | 21   | Is a directory            | ディレクトリに対して不正な操作 |

### ファイルディスクリプタ関連

| エラー名 | 番号 | 意味                          | 発生する場面                       |
| -------- | ---- | ----------------------------- | ---------------------------------- |
| EBADF    | 9    | Bad file descriptor           | 無効なファイルディスクリプタ       |
| EMFILE   | 24   | Too many open files           | プロセスのファイル上限に達した     |
| ENFILE   | 23   | Too many open files in system | システム全体のファイル上限に達した |

### 引数・操作関連

| エラー名 | 番号 | 意味             | 発生する場面               |
| -------- | ---- | ---------------- | -------------------------- |
| EINVAL   | 22   | Invalid argument | 引数が不正                 |
| EFAULT   | 14   | Bad address      | 無効なメモリアドレス       |
| ERANGE   | 34   | Result too large | 結果がバッファに収まらない |

### 非同期・再試行関連

| エラー名    | 番号 | 意味                             | 発生する場面                   |
| ----------- | ---- | -------------------------------- | ------------------------------ |
| EAGAIN      | 11   | Resource temporarily unavailable | ノンブロッキングで再試行が必要 |
| EWOULDBLOCK | 11   | 同上（EAGAIN と同じ値）          | ソケット操作でよく使われる     |
| EINTR       | 4    | Interrupted system call          | シグナルで中断された           |

EAGAIN は、本編 [05-file-descriptor](../05-file-descriptor.md) や [07-ipc](../07-ipc.md) でノンブロッキング I/O を扱う際に重要になります

EINTR は、本編 [03-signal](../03-signal.md) でシグナルを学ぶ際に登場します

### リソース関連

| エラー名 | 番号 | 意味                    | 発生する場面     |
| -------- | ---- | ----------------------- | ---------------- |
| ENOMEM   | 12   | Cannot allocate memory  | メモリ不足       |
| EBUSY    | 16   | Device or resource busy | リソースが使用中 |
| ENOSPC   | 28   | No space left on device | ディスク容量不足 |

### 注意：番号は環境によって異なる場合がある

エラー番号の値は、アーキテクチャによって異なる場合があります

そのため、コードでは番号ではなく<strong>名前</strong>で比較することが重要です

```c
/* 良い例：名前で比較 */
if (errno == ENOENT) {
    /* ... */
}

/* 悪い例：番号で比較（移植性がない） */
if (errno == 2) {
    /* ... */
}
```

---

## 調べ方

### perror()

<strong>perror()</strong> は、errno に対応するエラーメッセージを標準エラー出力に表示します

```c
#include <stdio.h>
#include <fcntl.h>

int fd = open("nonexistent.txt", O_RDONLY);
if (fd == -1) {
    perror("open");  /* 出力: open: No such file or directory */
}
```

引数に渡した文字列の後にコロンとエラーメッセージが続きます

### strerror()

<strong>strerror()</strong> は、errno の値を文字列に変換して返します

```c
#include <errno.h>
#include <string.h>
#include <stdio.h>

printf("エラー: %s\n", strerror(errno));
/* 出力: エラー: No such file or directory */
```

perror() と違い、文字列として取得できるため、ログに書き込んだり加工したりできます

### 使い分け

| 関数       | 用途                                       |
| ---------- | ------------------------------------------ |
| perror()   | 簡単にエラーを表示したいとき               |
| strerror() | エラーメッセージを文字列として使いたいとき |

### man ページでの確認

ターミナルで以下のコマンドを実行すると、エラー番号の一覧を確認できます

```bash
man errno
```

または

```bash
man 3 errno
```

各システムコールのマニュアルには、ERRORS セクションがあり、発生しうるエラーが記載されています

```bash
man 2 open    # open() で発生するエラーを確認
```

man ページの読み方については、[appendices/man.md](./man.md) を参照してください

---

## まとめ

| 項目       | 説明                                                                     |
| ---------- | ------------------------------------------------------------------------ |
| ENOENT     | ファイルが存在しない（<strong>E</strong>rror <strong>NO ENT</strong>ry） |
| errno      | エラー番号を格納するスレッドローカル変数                                 |
| perror()   | エラーメッセージを標準エラー出力に表示                                   |
| strerror() | エラー番号を文字列に変換                                                 |

覚えておくこと

| ポイント                    | 理由                           |
| --------------------------- | ------------------------------ |
| errno は失敗時のみ確認する  | 成功時の値は未定義             |
| 失敗直後に errno を確認する | 別の関数呼び出しで上書きされる |
| 番号ではなく名前で比較する  | 移植性を保つため               |

---

## 参考資料

<strong>Linux マニュアル</strong>

- [errno(3) - Linux manual page](https://man7.org/linux/man-pages/man3/errno.3.html)
  - errno の仕組みとエラー番号の一覧
- [perror(3) - Linux manual page](https://man7.org/linux/man-pages/man3/perror.3.html)
  - perror() 関数の使い方
- [strerror(3) - Linux manual page](https://man7.org/linux/man-pages/man3/strerror.3.html)
  - strerror() 関数の使い方

<strong>POSIX 標準</strong>

- [errno - The Open Group Base Specifications](https://pubs.opengroup.org/onlinepubs/9699919799/functions/errno.html)
  - POSIX における errno の定義

<strong>本編との関連</strong>

- [01-process](../01-process.md)
  - errno を使ったエラー処理
- [03-signal](../03-signal.md)
  - EINTR（シグナルによる中断）の扱い
- [04-thread](../04-thread.md)
  - errno がスレッドローカルであることの説明
- [05-file-descriptor](../05-file-descriptor.md)
  - open() のエラー処理、EAGAIN の扱い
- [06-stdio](../06-stdio.md)
  - fopen() のエラー処理
- [07-ipc](../07-ipc.md)
  - mkfifo() で EEXIST をチェック
