<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# なぜ改行コードは複数あるのか

## はじめに

テキストファイルを扱っていると、改行コードの違いに遭遇することがあります

```
Unix/Linux:   LF（\n）
Windows:      CRLF（\r\n）
旧 Mac:       CR（\r）
```

なぜ「改行」という単純な概念に複数の表現方法があるのでしょうか？

この違いはタイプライターの時代にまで遡ります

---

## 目次

- [CR/LFの由来](#crlfの由来)
- [各OSの選択](#各osの選択)
- [C言語での扱い](#c言語での扱い)
- [実務での注意](#実務での注意)
- [まとめ](#まとめ)
- [参考資料](#参考資料)

---

## CR/LFの由来

### タイプライターの機構

CR（Carriage Return）と LF（Line Feed）は、タイプライターの物理的な動作に由来します

```
タイプライターの構造

用紙：キャリッジ（台車）に固定されている
印字：活字が固定位置で紙に打ち付ける
移動：1文字打つごとにキャリッジが左に移動
```

1行を打ち終わったとき、次の行に移るには2つの動作が必要でした

```
1. Carriage Return（復帰）
   キャリッジを右端に戻す（X軸の移動）

2. Line Feed（改行）
   紙を1行分上に送る（Y軸の移動）
```

### テレタイプへの継承

1900年代初頭、タイプライターの概念は電気通信機器（テレタイプ）に引き継がれました

1954年のテレタイプのマニュアルには、次のように記載されています

```
新しい行を始めるには、CAR RET、LINE FEED、LTRS の順にキーを押す
```

### なぜ CR の後に LF なのか

テレタイプの印字ヘッドは、右端から左端に戻るのに物理的な時間がかかりました

```
CR 送信後の問題

印字ヘッドが戻る途中で次の文字を印字すると
→ 行の途中に文字が印刷されてしまう（にじみ）
```

LF を CR の後に送ることで、印字ヘッドが戻る時間を稼いでいました

```
時間的な流れ

CR 送信 → 印字ヘッドが戻り始める（約100ms）
LF 送信 → 紙が送られる（この間もヘッドは移動中）
次の文字 → ヘッドが左端に到達、正しく印字される
```

---

## 各OSの選択

### Unix/Linux：LF のみ

Multics（Unix が影響を受けたシステム）の開発者たちは、CR+LF の冗長さに着目しました

```
Multics の設計思想

「改行」という論理的な概念は1文字で表現できる
物理デバイスへの変換はデバイスドライバが担当すべき
```

Unix はこの設計を継承し、改行を LF（`\n`）のみで表現します

```c
/*
 * LF のみ
 */
printf("Hello\n");
```

macOS も OS X（2001年）以降、Unix 系として LF を採用しています

### Windows：CRLF

Windows は、CP/M → MS-DOS → Windows という系譜で CRLF を継承しています

```
歴史的な継承

1970年代：CP/M（Intel 8080用OS）が CRLF を採用
1980年代：MS-DOS が CP/M の慣習を継承
1990年代：Windows が MS-DOS との互換性を維持
```

当時のコンピュータはプリンタやテレタイプと日常的に接続されており、タイプライター互換を維持する実用的な理由がありました

### 旧 Mac（OS 9 以前）：CR のみ

Apple は独自の選択として CR のみを採用していました

```
Mac の歴史

1984年〜2001年：CR のみ
2001年〜（OS X）：LF に移行（Unix ベースになったため）
```

### 比較表

| OS                  | 改行コード | 16進数    | エスケープ |
| ------------------- | ---------- | --------- | ---------- |
| Unix/Linux/macOS    | LF         | 0x0A      | `\n`       |
| Windows             | CRLF       | 0x0D 0x0A | `\r\n`     |
| 旧 Mac（OS 9 以前） | CR         | 0x0D      | `\r`       |

### なぜ統一されなかったのか

<strong>後方互換性の壁</strong>

| OS      | 統一しなかった理由                             |
| ------- | ---------------------------------------------- |
| Windows | 膨大な既存ソフトウェア・データとの互換性       |
| Unix    | すでに LF で統一されており、変更する理由がない |

Windows が LF のみに移行しようとすると、30 年以上にわたる CRLF のテキストファイルやソフトウェアとの互換性が壊れます

<strong>現実的な解決策</strong>

統一する代わりに、各層で変換する仕組みが発達しました

- C 言語のテキストモード
- エディタの自動検出
- Git の改行コード正規化

---

## C言語での扱い

### テキストモードとバイナリモード

C 言語の `fopen()` には、テキストモードとバイナリモードがあります

```c
/*
 * テキストモード
 */
FILE *fp_text = fopen("file.txt", "r");
/*
 * バイナリモード
 */
FILE *fp_bin  = fopen("file.bin", "rb");
```

### テキストモードの動作

テキストモードでは、OS に応じて改行コードが<strong>自動変換</strong>されます

```
Windows でのテキストモード

書き込み時：\n → \r\n に変換
読み込み時：\r\n → \n に変換
```

```c
/*
 * Windows でテキストモードで書き込む場合
 */
/*
 * 実際には "Hello\r\n" がファイルに書かれる
 */
fprintf(fp, "Hello\n");

/*
 * Windows でテキストモードで読み込む場合
 */
/*
 * \r\n は \n として読み込まれる
 */
fgets(buf, size, fp);
```

この変換のおかげで、プログラム内では `\n` だけを使えば、OS の違いを意識する必要がありません

### バイナリモードの動作

バイナリモードでは、変換は一切行われません

```c
/*
 * バイト列をそのまま読む
 */
FILE *fp = fopen("image.png", "rb");
```

画像やアーカイブなどのバイナリファイルは、必ずバイナリモードで開く必要があります

```
テキストモードでバイナリファイルを開くと

Windows：ファイル内の 0x0D 0x0A が 0x0A に変換される
→ ファイルが壊れる
```

### Unix 系での 'b' フラグ

POSIX 準拠システム（Linux、macOS など）では、'b' フラグは<strong>効果がありません</strong>

```c
/*
 * Linux では以下は同じ動作
 */
fopen("file.txt", "r");
fopen("file.txt", "rb");
```

ただし、移植性のために 'b' を明示することが推奨されています

---

## 実務での注意

### Git での改行コード問題

異なる OS の開発者が同じリポジトリで作業すると、改行コードの違いが問題になります

```
よくある問題

Windows 開発者：CRLF でコミット
Linux 開発者：LF でコミット
→ 差分に大量の改行変更が表示される
```

### core.autocrlf 設定

Git の `core.autocrlf` 設定で、チェックアウト時の自動変換を制御できます

| 値      | 動作                                                 |
| ------- | ---------------------------------------------------- |
| `true`  | チェックアウト時に LF → CRLF、コミット時に CRLF → LF |
| `input` | コミット時のみ CRLF → LF、チェックアウト時は変換なし |
| `false` | 変換しない                                           |

```bash
# Windows での推奨設定
git config --global core.autocrlf true

# macOS/Linux での推奨設定
git config --global core.autocrlf input
```

### .gitattributes（推奨）

リポジトリ単位で改行コードを制御するには、`.gitattributes` ファイルを使用します

```
# .gitattributes の例

# デフォルトで自動判定
* text=auto

# テキストファイルは LF に統一
*.c text eol=lf
*.h text eol=lf
*.md text eol=lf

# Windows 専用ファイルは CRLF
*.bat text eol=crlf
*.cmd text eol=crlf

# バイナリファイルは変換しない
*.png binary
*.jpg binary
```

`.gitattributes` は `core.autocrlf` より優先されるため、リポジトリで統一したルールを適用できます

### エディタの設定

多くのエディタは、改行コードの自動検出と変換機能を持っています

```
VS Code の設定例

/*
 * 新規ファイルは LF で作成
 */
"files.eol": "\n"
/*
 * エンコーディング自動検出
 */
"files.autoGuessEncoding": true
```

---

## まとめ

| 項目             | 内容                                               |
| ---------------- | -------------------------------------------------- |
| CR/LF の由来     | タイプライターの物理的な動作（復帰と改行）         |
| Unix/Linux/macOS | LF（`\n`）─ 論理的な改行は1文字で十分              |
| Windows          | CRLF（`\r\n`）─ タイプライター互換を継承           |
| C 言語           | テキストモードで自動変換、バイナリモードは変換なし |
| Git              | `.gitattributes` でリポジトリ単位の統一を推奨      |

<strong>覚えておくこと</strong>

- 改行コードの違いはタイプライター時代の名残
- C 言語のテキストモードが OS 間の違いを吸収してくれる
- バイナリファイルは必ずバイナリモードで開く
- Git では `.gitattributes` で改行コードを統一する

---

## 参考資料

<strong>Linux マニュアル</strong>

- [fopen(3) - Linux manual page](https://man7.org/linux/man-pages/man3/fopen.3.html)
  - ファイルオープン、テキストモードとバイナリモード

<strong>Git ドキュメント</strong>

- [Configuring Git to handle line endings - GitHub Docs](https://docs.github.com/articles/dealing-with-line-endings)
  - Git での改行コード設定ガイド
- [Git - gitattributes Documentation](https://git-scm.com/docs/gitattributes)
  - .gitattributes の text 属性と eol 属性

<strong>歴史的背景</strong>

- [CRLF vs. LF: Normalizing Line Endings in Git](https://www.aleksandrhovhannisyan.com/blog/crlf-vs-lf-normalizing-line-endings-in-git/)
  - 改行コードの歴史と Git での対処法

<strong>本編との関連</strong>

- [06-stdio](../06-stdio.md)
  - fopen() のテキストモード/バイナリモードの違い
