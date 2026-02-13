<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# リンクはなぜ2種類あるのか

## はじめに

[05-file-descriptor](../05-file-descriptor.md) で、ファイルディスクリプタの仕組みを学びました

その中で<strong>inode</strong>（ファイルの実体情報を持つデータ構造）が登場しました

ターミナルで `rm` コマンドを使ったことがあるかもしれません

```bash
rm file.txt       # ファイルを「削除」
```

でも、この「削除」は本当にファイルを消しているのでしょうか

実は、Linux のファイル削除は「リンクを解除する」という操作です

`rm` の内部では `unlink()` というシステムコールが呼ばれています

「リンク」とは何でしょうか

このドキュメントでは、Linux のリンクの仕組みと、なぜ2種類あるのかを説明します

---

## 目次

- [2種類のリンク](#2種類のリンク)
- [ハードリンク](#ハードリンク)
- [シンボリックリンク](#シンボリックリンク)
- [ハードリンクの制限](#ハードリンクの制限)
- [使い分けの指針](#使い分けの指針)
- [まとめ](#まとめ)
- [参考資料](#参考資料)

---

## 2種類のリンク

### なぜ2種類のリンクが必要なのか

<strong>もしハードリンクだけだったら？</strong>

- ディレクトリへのリンクが作れない（循環の危険）
- 別のファイルシステムへのリンクが作れない
- リンク先が見えない（どれが「オリジナル」か分からない）

<strong>もしシンボリックリンクだけだったら？</strong>

- 原本を移動・削除するとリンクが壊れる
- データの永続的な参照ができない

2種類あることで、用途に応じて使い分けができます

| 要件                                         | 適切なリンク       |
| -------------------------------------------- | ------------------ |
| データの永続的な参照                         | ハードリンク       |
| 柔軟な参照（ディレクトリ、別パーティション） | シンボリックリンク |

### 基本的な違い

Linux には2種類のリンクがあります

| 項目         | ハードリンク   | シンボリックリンク                |
| ------------ | -------------- | --------------------------------- |
| 作成コマンド | `ln file link` | `ln -s file link`                 |
| 指すもの     | inode          | パス文字列                        |
| 別名         | hard link      | symbolic link, symlink, soft link |
| 原本削除時   | リンクは有効   | リンクは無効になる                |

<strong>ハードリンク</strong>は、ファイルの inode を直接指します

<strong>シンボリックリンク</strong>は、ファイルのパス（場所の文字列）を指します

### 日常の例え

ハードリンクは「同じ家に複数の住所がある」ようなものです

どの住所に行っても、同じ家にたどり着きます

1つの住所を削除しても、家は残ります

シンボリックリンクは「道案内の看板」のようなものです

看板は「〇〇へはこちら」と方向を示しています

看板が指す先の建物がなくなると、看板は意味をなさなくなります

---

## ハードリンク

### inodeとリンクカウント

[05-file-descriptor](../05-file-descriptor.md) で学んだように、<strong>inode</strong> はファイルの実体情報を持つデータ構造です

inode には<strong>リンクカウント（st_nlink）</strong>というフィールドがあります

リンクカウントは、その inode を指しているディレクトリエントリの数です

### ファイル作成時の動作

ファイルを作成すると、以下のことが起きます

1. 新しい inode が作成される
2. inode にファイルの実体（データ）が紐付けられる
3. ディレクトリエントリがファイル名と inode を結びつける
4. リンクカウントが 1 になる

```
ファイル作成後の状態

ディレクトリエントリ         inode
┌──────────────────┐      ┌─────────────────────┐
│ file.txt → inode │ ───→ │ st_nlink = 1        │
└──────────────────┘      │ サイズ、権限、...   │
                          │ データブロックへの  │
                          │ ポインタ            │
                          └─────────────────────┘
```

### ハードリンク作成時の動作

ハードリンクを作成すると、同じ inode を指す新しいディレクトリエントリが作成されます

```bash
ln file.txt hardlink.txt
```

```
ハードリンク作成後の状態

ディレクトリエントリ         inode
┌──────────────────┐      ┌─────────────────────┐
│ file.txt → inode │ ───→ │ st_nlink = 2        │
└──────────────────┘  ┌─→ │ サイズ、権限、...   │
┌──────────────────┐  │   │ データブロックへの  │
│ hardlink.txt     │ ─┘   │ ポインタ            │
│          → inode │      └─────────────────────┘
└──────────────────┘
```

リンクカウントが 2 になりました

### ファイル「削除」の仕組み

`rm` コマンドは、実際には `unlink()` システムコールを呼び出します

unlink() は以下の動作をします

1. ディレクトリエントリを削除する
2. リンクカウントを 1 減らす
3. リンクカウントが 0 になり、かつファイルを開いているプロセスがなければ、inode とデータを解放する

```bash
rm file.txt
```

```
rm file.txt 実行後の状態

ディレクトリエントリ         inode
（file.txt は削除済み）   ┌─────────────────────┐
                      ┌─→ │ st_nlink = 1        │
┌──────────────────┐  │   │ サイズ、権限、...   │
│ hardlink.txt     │ ─┘   │ データブロックへの  │
│          → inode │      │ ポインタ            │
└──────────────────┘      └─────────────────────┘
```

リンクカウントが 1 になりましたが、まだ 0 ではないのでファイルは削除されません

hardlink.txt から、元のファイルの内容にアクセスできます

### コード例

ハードリンクを作成し、リンクカウントを確認する例：

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <unistd.h>

int main(void) {
    struct stat st;

    /**
     * link() でハードリンクを作成します
     *
     * link() とは
     * ─── 既存のファイルに対するハードリンクを作成します
     *
     * 引数
     * ─── 第1引数：既存のファイルのパス
     * ─── 第2引数：新しいリンクのパス
     *
     * 戻り値
     * ─── 成功時は 0
     * ─── 失敗時は -1（errno が設定される）
     *
     * 参考
     * ─── https://man7.org/linux/man-pages/man2/link.2.html
     */
    if (link("original.txt", "hardlink.txt") == -1) {
        perror("link");
        return EXIT_FAILURE;
    }

    /**
     * stat() でファイル情報を取得します
     *
     * st_nlink にリンクカウントが格納されています
     */
    if (stat("original.txt", &st) == -1) {
        perror("stat");
        return EXIT_FAILURE;
    }

    printf("─── リンクカウント\n");
    printf("────── st_nlink（%lu）\n", (unsigned long)st.st_nlink);

    return EXIT_SUCCESS;
}
```

実行結果：

```
─── リンクカウント
────── st_nlink（2）
```

---

## シンボリックリンク

### パスを指すリンク

シンボリックリンクは、<strong>ファイルのパス（文字列）</strong>を格納した特殊なファイルです

inode ではなく、「このパスを見てね」という情報を持っています

```bash
ln -s /path/to/original.txt symlink.txt
```

```
シンボリックリンクの構造

シンボリックリンク           原本ファイル
┌─────────────────────┐      ┌──────────────────┐
│ symlink.txt         │      │ original.txt     │
│ 内容：              │      │ （inode A）      │
│ "/path/to/          │ ───→ │ 実際のデータ     │
│  original.txt"      │      └──────────────────┘
│ （inode B）         │
└─────────────────────┘
```

シンボリックリンク自体も inode を持ちますが、その内容は「パス文字列」です

### シンボリックリンクの解決

プログラムがシンボリックリンクにアクセスすると、カーネルが<strong>自動的にリンクを辿ります</strong>

```c
/* symlink.txt は /path/to/original.txt を指している */
int fd = open("symlink.txt", O_RDONLY);
/* 実際には original.txt が開かれる */
```

この「リンクを辿る」処理を<strong>シンボリックリンクの解決</strong>といいます

### ダングリングリンク

原本ファイルが削除されると、シンボリックリンクは<strong>存在しないパス</strong>を指すことになります

```bash
rm original.txt
cat symlink.txt   # エラー：No such file or directory
```

このような状態のシンボリックリンクを<strong>ダングリングリンク（dangling link）</strong>といいます

「dangling」は「ぶら下がっている」「宙ぶらりんの」という意味です

ハードリンクでは、原本を削除してもリンクが有効なままであることと対照的です

### ls -l での表示

シンボリックリンクは `ls -l` で確認できます

```bash
$ ls -l
lrwxrwxrwx 1 user user   20 Jan 15 10:00 symlink.txt -> /path/to/original.txt
-rw-r--r-- 2 user user 1024 Jan 15 09:00 original.txt
```

- ファイルタイプが `l`（シンボリックリンク）
- `->` の後にリンク先のパスが表示される
- パーミッションは `rwxrwxrwx`（シンボリックリンク自体のパーミッションは通常無視される）

### コード例

シンボリックリンクを作成し、リンク先を確認する例：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void) {
    char buffer[1024];
    ssize_t len;

    /**
     * symlink() でシンボリックリンクを作成します
     *
     * symlink() とは
     * ─── ファイルへのシンボリックリンクを作成します
     *
     * 引数
     * ─── 第1引数：リンク先のパス（文字列として保存される）
     * ─── 第2引数：シンボリックリンクのパス
     *
     * 戻り値
     * ─── 成功時は 0
     * ─── 失敗時は -1（errno が設定される）
     *
     * 参考
     * ─── https://man7.org/linux/man-pages/man2/symlink.2.html
     */
    if (symlink("/path/to/original.txt", "symlink.txt") == -1) {
        perror("symlink");
        return EXIT_FAILURE;
    }

    /**
     * readlink() でシンボリックリンクの内容を読み取ります
     *
     * readlink() とは
     * ─── シンボリックリンクが指すパス文字列を取得します
     * ─── リンクを辿らず、リンク自体の内容を読みます
     *
     * 注意
     * ─── 終端のヌル文字（'\0'）は付加されません
     * ─── 戻り値（読み取ったバイト数）を使って終端を付ける必要があります
     *
     * 参考
     * ─── https://man7.org/linux/man-pages/man2/readlink.2.html
     */
    len = readlink("symlink.txt", buffer, sizeof(buffer) - 1);
    if (len == -1) {
        perror("readlink");
        return EXIT_FAILURE;
    }

    buffer[len] = '\0';  /* 終端を付加 */

    printf("─── シンボリックリンクの内容\n");
    printf("────── %s\n", buffer);

    return EXIT_SUCCESS;
}
```

実行結果：

```
─── シンボリックリンクの内容
────── /path/to/original.txt
```

---

## ハードリンクの制限

ハードリンクには2つの重要な制限があります

### 制限1：ディレクトリには作成できない

ハードリンクは<strong>通常ファイル</strong>にのみ作成できます

<strong>ディレクトリ</strong>にハードリンクを作成しようとすると、エラーになります

```bash
$ ln /home/user/mydir /home/user/mydir-link
ln: /home/user/mydir: hard link not allowed for directory
```

<strong>なぜディレクトリへのハードリンクが禁止されているのか</strong>

ファイルシステムはディレクトリを<strong>木構造（ツリー構造）</strong>で管理しています

```
/
├── home/
│   └── user/
│       ├── file.txt
│       └── documents/
└── tmp/
```

もしディレクトリへのハードリンクが許可されると、木構造に<strong>ループ（循環）</strong>ができる可能性があります

```
documents/ が /home/user/ へのハードリンクだったら...

/home/user/documents/ → /home/user/
/home/user/documents/documents/ → /home/user/
/home/user/documents/documents/documents/ → ...（無限ループ）
```

`find` コマンドや `du` コマンドなど、ディレクトリを再帰的に走査するプログラムが無限ループに陥ります

この問題を防ぐため、ディレクトリへのハードリンクは禁止されています

（ただし `.` と `..` は例外で、カーネルが管理する特殊なハードリンクです）

### 制限2：異なるファイルシステムを跨げない

ハードリンクは<strong>同じファイルシステム内</strong>でのみ作成できます

```bash
$ ln /home/user/file.txt /mnt/usb/file-link.txt
ln: failed to create hard link '/mnt/usb/file-link.txt': Invalid cross-device link
```

<strong>なぜファイルシステムを跨げないのか</strong>

ハードリンクは inode を直接指します

inode 番号は<strong>各ファイルシステム内でのみ一意</strong>です

```
ファイルシステム A の inode 12345
ファイルシステム B の inode 12345
→ 同じ番号でも、まったく別のファイル
```

異なるファイルシステムの inode を指すことはできないため、ファイルシステムを跨ぐハードリンクは作成できません

この制限に遭遇したときのエラーコードは <strong>EXDEV</strong>（cross-device link）です

### シンボリックリンクには制限がない

シンボリックリンクは「パス文字列」を格納するだけなので、これらの制限がありません

| 項目                             | ハードリンク | シンボリックリンク |
| -------------------------------- | ------------ | ------------------ |
| ディレクトリへのリンク           | 不可         | 可能               |
| 異なるファイルシステムへのリンク | 不可         | 可能               |

---

## 使い分けの指針

### ハードリンクを使う場面

| 場面                                   | 理由                                                 |
| -------------------------------------- | ---------------------------------------------------- |
| バックアップ                           | 同じデータを複数の場所から参照でき、容量を節約できる |
| 原本が移動・削除されてもアクセスしたい | inode を直接指すので、パスの変更に影響されない       |
| ダングリングリンクを避けたい           | 最後のリンクが消えるまでデータは残る                 |

<strong>実用例：rsync --link-dest</strong>

`rsync` の `--link-dest` オプションは、ハードリンクを使って増分バックアップを実現します

変更のないファイルは前回のバックアップへのハードリンクとなり、ディスク容量を節約できます

### シンボリックリンクを使う場面

| 場面                           | 理由                                                           |
| ------------------------------ | -------------------------------------------------------------- |
| ディレクトリへのリンク         | ハードリンクでは不可                                           |
| 別のファイルシステムへのリンク | ハードリンクでは不可                                           |
| 実行ファイルのバージョン管理   | `python -> python3.11` のように切り替えが簡単                  |
| 設定ファイルの共有             | dotfiles を Git 管理し、ホームディレクトリにシンボリックリンク |

<strong>実用例：/usr/bin のコマンド</strong>

多くの Linux ディストリビューションでは、コマンドのバージョン管理にシンボリックリンクを使います

```bash
$ ls -l /usr/bin/python3
lrwxrwxrwx 1 root root 9 Jan  1 00:00 /usr/bin/python3 -> python3.11

$ ls -l /usr/bin/vi
lrwxrwxrwx 1 root root 20 Jan  1 00:00 /usr/bin/vi -> /etc/alternatives/vi
```

### 判断フローチャート

```
リンクを作りたい
    │
    ├─ ディレクトリ？ ─────────────→ シンボリックリンク
    │
    ├─ 別のファイルシステム？ ────→ シンボリックリンク
    │
    ├─ 原本削除後もアクセス必要？ ─→ ハードリンク
    │
    ├─ パスの変更に追従したい？ ──→ シンボリックリンク
    │
    └─ よくわからない ────────────→ シンボリックリンク（一般的な選択）
```

迷ったときは<strong>シンボリックリンク</strong>を選ぶのが安全です

制限が少なく、`ls -l` でリンク先が見えるので管理しやすいためです

---

## まとめ

### 2種類のリンクの比較

| 項目                   | ハードリンク       | シンボリックリンク |
| ---------------------- | ------------------ | ------------------ |
| 指すもの               | inode              | パス文字列         |
| 作成コマンド           | `ln`               | `ln -s`            |
| システムコール         | link()             | symlink()          |
| ディレクトリへのリンク | 不可               | 可能               |
| ファイルシステム跨ぎ   | 不可               | 可能               |
| 原本削除時             | リンクは有効       | ダングリングリンク |
| リンクの見分け方       | ls -l で見分け困難 | `->` で表示        |

### 覚えておくこと

- `rm` は実際には `unlink()` を呼んでいる
- ファイルはリンクカウントが 0 になり、開いているプロセスがなくなったとき削除される
- ディレクトリへのハードリンクは循環を防ぐため禁止されている
- ファイルシステムを跨ぐハードリンクは inode の一意性のため不可
- 迷ったらシンボリックリンクを使う

---

## 参考資料

<strong>Linux マニュアル</strong>

- [ln(1) - Linux manual page](https://man7.org/linux/man-pages/man1/ln.1.html)
  - ln コマンドの使い方、`-s` オプションでシンボリックリンク作成
- [link(2) - Linux manual page](https://man7.org/linux/man-pages/man2/link.2.html)
  - ハードリンクを作成するシステムコール
- [symlink(2) - Linux manual page](https://man7.org/linux/man-pages/man2/symlink.2.html)
  - シンボリックリンクを作成するシステムコール
- [symlink(7) - Linux manual page](https://man7.org/linux/man-pages/man7/symlink.7.html)
  - シンボリックリンクの扱いに関する詳細、リンクの解決ルール
- [unlink(2) - Linux manual page](https://man7.org/linux/man-pages/man2/unlink.2.html)
  - リンクを削除するシステムコール、rm の内部動作
- [inode(7) - Linux manual page](https://man7.org/linux/man-pages/man7/inode.7.html)
  - inode の構造、st_nlink（リンクカウント）の説明

<strong>本編との関連</strong>

- [05-file-descriptor](../05-file-descriptor.md)
  - inode の概念、カーネルの 3 層構造
