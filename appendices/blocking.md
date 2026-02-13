<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# なぜread()は止まるのか

## はじめに

`read()` を呼び出すと、プログラムの実行が「止まる」ことがあります

```c
char buf[256];
/*
 * ここで止まることがある
 */
read(fd, buf, sizeof(buf));
/*
 * いつまでも表示されない...
 */
printf("読み取り完了\n");
```

これは<strong>ブロッキング I/O</strong> と呼ばれる動作です

データがまだ届いていないとき、`read()` は「データが届くまで待つ」のがデフォルトの動作です

---

## 目次

- [なぜread()は止まるのか](#なぜreadは止まるのか-1)
- [ブロッキングする関数](#ブロッキングする関数)
- [ノンブロッキング I/O とは](#ノンブロッキング-io-とは)
- [実践パターン](#実践パターン)
- [発展：I/O 多重化への橋渡し](#発展io-多重化への橋渡し)
- [まとめ](#まとめ)
- [参考資料](#参考資料)

---

## なぜread()は止まるのか

### デフォルト動作は「待つ」

`read()` がブロックするのは、<strong>データがないときに待つのがデフォルト動作</strong>だからです

```
read() の動作

データがある場合
─── 即座にデータを読み取って戻る

データがない場合（デフォルト）
─── データが届くまで待機する（ブロック）
```

### なぜブロッキングがデフォルトなのか

<strong>もしデフォルトがノンブロッキングだったら？</strong>

```c
/* 全てのプログラムがこのようなコードを書く必要がある */
while (1) {
    int n = read(fd, buf, sizeof(buf));
    if (n == -1 && errno == EAGAIN) {
        /* データがない、どうする？ */
        sleep(1);  /* 待機して再試行 */
        continue;
    }
    break;
}
```

ほとんどのプログラムは「データが来たら処理する」という単純なパターンです

全員にループと再試行を書かせるのは無駄です

<strong>ブロッキングがデフォルトの理由</strong>

| 理由             | 説明                                           |
| ---------------- | ---------------------------------------------- |
| シンプルなコード | 「読む」→「処理する」の直線的な流れで書ける    |
| CPU 効率         | 待機中はプロセスがスリープし、CPU を消費しない |
| 80/20 の法則     | 大多数のプログラムはこの動作で十分             |

```c
/* ブロッキング（デフォルト）：シンプルに書ける */
char buf[256];
int n = read(fd, buf, sizeof(buf));
/* ここに来たときには、必ずデータが読めている（またはエラー） */
```

ノンブロッキングは「本当に必要な場面」でだけ使う設計になっています

### どんなときにブロックするか

ブロッキングが発生するのは、主に以下のケースです

| 対象                   | ブロック条件                   |
| ---------------------- | ------------------------------ |
| パイプ                 | 書き込み側からデータが来るまで |
| ソケット               | 相手からデータが届くまで       |
| 端末（標準入力）       | ユーザーが入力するまで         |
| FIFO（名前付きパイプ） | 相手側が open() するまで       |

通常のファイル（ディスク上のファイル）は、データがすぐに読めるため、通常はブロックしません

---

## ブロッキングする関数

`read()` 以外にも、ブロッキングする関数があります

| 関数        | ブロック条件                       |
| ----------- | ---------------------------------- |
| `read()`    | 読み取るデータがない               |
| `write()`   | 書き込み先のバッファが満杯         |
| `accept()`  | 接続要求がない                     |
| `recv()`    | 受信データがない                   |
| `connect()` | TCP 接続確立中（ハンドシェイク中） |

### ブロッキングの共通点

すべて<strong>「相手を待つ」操作</strong>です

```
read()    ─── データを待つ
write()   ─── 空きを待つ
accept()  ─── 接続を待つ
recv()    ─── データを待つ
connect() ─── 応答を待つ
```

---

## ノンブロッキング I/O とは

### 「待たない」モード

<strong>ノンブロッキング I/O</strong> は、データがなくても即座に戻る動作モードです

```
ブロッキング（デフォルト）
─── データがないとき：待機する

ノンブロッキング（O_NONBLOCK）
─── データがないとき：エラー（EAGAIN）を返して即座に戻る
```

### O_NONBLOCK フラグ

ノンブロッキングモードを有効にするには、`O_NONBLOCK` フラグを使います

```c
/*
 * 方法1：open() で指定
 */
int fd = open("/dev/tty", O_RDONLY | O_NONBLOCK);

/*
 * 方法2：fcntl() で後から設定
 */
int flags = fcntl(fd, F_GETFL);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);
```

### EAGAIN と EWOULDBLOCK

ノンブロッキングモードでデータがないとき、`read()` は `-1` を返し、`errno` に `EAGAIN`（または `EWOULDBLOCK`）が設定されます

```c
ssize_t n = read(fd, buf, sizeof(buf));
if (n == -1) {
    if (errno == EAGAIN || errno == EWOULDBLOCK) {
        /*
         * データがまだない（エラーではない）
         * 後で再試行する
         */
    } else {
        /*
         * 本当のエラー
         */
        perror("read");
    }
}
```

<strong>EAGAIN の意味</strong>

```
EAGAIN = "Error, try AGAIN"
─── 今はデータがないが、後で再試行すれば読めるかもしれない
```

POSIX では `EAGAIN` と `EWOULDBLOCK` のどちらが返るか規定していないため、両方をチェックするのが安全です

---

## 実践パターン

### パターン 1：open() で O_NONBLOCK を指定

```c
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <stdio.h>

int main(void)
{
    /*
     * ノンブロッキングモードで開く
     */
    int fd = open("/dev/tty", O_RDONLY | O_NONBLOCK);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    char buf[256];
    ssize_t n = read(fd, buf, sizeof(buf));

    if (n == -1) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            printf("データがまだありません\n");
        } else {
            perror("read");
        }
    } else {
        printf("%zd バイト読み取りました\n", n);
    }

    close(fd);
    return 0;
}
```

### パターン 2：fcntl() で後から設定

```c
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <stdio.h>

int main(void)
{
    /*
     * 標準入力
     */
    int fd = 0;

    /*
     * 現在のフラグを取得
     */
    int flags = fcntl(fd, F_GETFL);
    if (flags == -1) {
        perror("fcntl F_GETFL");
        return 1;
    }

    /*
     * O_NONBLOCK を追加
     */
    if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1) {
        perror("fcntl F_SETFL");
        return 1;
    }

    char buf[256];
    ssize_t n = read(fd, buf, sizeof(buf));

    if (n == -1) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            printf("入力がまだありません\n");
        } else {
            perror("read");
        }
    } else if (n == 0) {
        printf("EOF\n");
    } else {
        printf("%zd バイト読み取りました\n", n);
    }

    return 0;
}
```

### パターン 3：ループで再試行

```c
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <stdio.h>

int main(void)
{
    /*
     * 標準入力
     */
    int fd = 0;

    /*
     * ノンブロッキングモードに設定
     */
    int flags = fcntl(fd, F_GETFL);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);

    char buf[256];

    /*
     * データが来るまでループ
     */
    while (1) {
        ssize_t n = read(fd, buf, sizeof(buf) - 1);

        if (n > 0) {
            buf[n] = '\0';
            printf("読み取り：%s", buf);
            break;
        } else if (n == -1) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                /*
                 * まだデータがない
                 */
                printf("待機中...\n");
                /*
                 * 1秒待って再試行
                 */
                sleep(1);
            } else {
                perror("read");
                break;
            }
        } else {
            /*
             * n == 0: EOF
             */
            printf("EOF\n");
            break;
        }
    }

    return 0;
}
```

---

## 発展：I/O 多重化への橋渡し

### ノンブロッキングの限界

ノンブロッキング I/O には限界があります

```c
/*
 * 複数のファイルディスクリプタを監視したい場合
 */
while (1) {
    /*
     * fd1 をチェック
     */
    read(fd1, ...);
    /*
     * fd2 をチェック
     */
    read(fd2, ...);
    /*
     * fd3 をチェック
     */
    read(fd3, ...);
    /*
     * 待機
     */
    sleep(1);
}
```

この方法は CPU を無駄に使い、応答も遅くなります

### I/O 多重化

<strong>I/O 多重化</strong>（I/O Multiplexing）は、複数のファイルディスクリプタを効率的に監視する仕組みです

```
select() / poll() / epoll()
─── 「どの fd が読み書き可能か」を一度に確認できる
─── 準備ができた fd だけを処理できる
```

| 関数       | 特徴                           |
| ---------- | ------------------------------ |
| `select()` | 移植性が高い、fd 数に上限あり  |
| `poll()`   | fd 数の制限がない              |
| `epoll()`  | Linux 専用、大量の fd に効率的 |

### 使い分けの目安

```
小規模・移植性重視 ─── select()
中規模 ─────────── poll()
大規模・Linux 専用 ─ epoll()
```

I/O 多重化の詳細は、別途資料で学習することをお勧めします

---

## まとめ

| 項目         | ブロッキング（デフォルト） | ノンブロッキング     |
| ------------ | -------------------------- | -------------------- |
| 動作         | データが来るまで待つ       | 即座に戻る           |
| フラグ       | なし                       | O_NONBLOCK           |
| データなし時 | 待機する                   | EAGAIN を返す        |
| 利点         | コードが簡単               | 他の処理を並行できる |
| 欠点         | 処理が止まる               | エラー処理が必要     |

<strong>ノンブロッキングを選ぶ判断基準</strong>

| 質問                                                   | Yes の場合                    |
| ------------------------------------------------------ | ----------------------------- |
| 複数の入力元（ソケット、パイプ等）を同時に監視したい？ | ノンブロッキング + I/O 多重化 |
| 待機中にタイマーやアニメーションなど別処理が必要？     | ノンブロッキング              |
| 単一の入力を待つだけで良い？                           | ブロッキング                  |
| 応答性より実装の簡単さが重要？                         | ブロッキング                  |

<strong>典型的な使い分け</strong>

```
ブロッキング
─── シンプルなコマンドラインツール
─── 1つのファイルを読み書きするだけのプログラム

ノンブロッキング
─── ネットワークサーバー（多数のクライアント）
─── GUI アプリケーション（応答性が必要）
─── I/O 多重化と組み合わせる場合
```

---

## 参考資料

<strong>Linux マニュアル</strong>

- [read(2) - Linux manual page](https://man7.org/linux/man-pages/man2/read.2.html)
  - read() システムコールの詳細、ブロッキング動作の説明
- [fcntl(2) - Linux manual page](https://man7.org/linux/man-pages/man2/fcntl.2.html)
  - F_GETFL/F_SETFL によるフラグ操作、O_NONBLOCK の設定方法
- [accept(2) - Linux manual page](https://man7.org/linux/man-pages/man2/accept.2.html)
  - ソケット接続受け入れのブロッキング動作
- [recv(2) - Linux manual page](https://man7.org/linux/man-pages/man2/recv.2.html)
  - ソケットからのデータ受信、EAGAIN/EWOULDBLOCK の説明

<strong>I/O 多重化</strong>

- [select(2) - Linux manual page](https://man7.org/linux/man-pages/man2/select.2.html)
  - 複数の fd を同期的に監視する
- [poll(2) - Linux manual page](https://man7.org/linux/man-pages/man2/poll.2.html)
  - select() の改良版
- [epoll(7) - Linux manual page](https://man7.org/linux/man-pages/man7/epoll.7.html)
  - Linux 専用の高効率 I/O イベント通知

<strong>本編との関連</strong>

- [05-file-descriptor](../05-file-descriptor.md)
  - read()/write() の基本動作
- [07-ipc](../07-ipc.md)
  - パイプ、FIFO、ソケットのブロッキング動作
