---
layout: default
title: なぜビット演算を使うのか
---

# [なぜビット演算を使うのか](#why-use-bitwise-operations) {#why-use-bitwise-operations}

## [はじめに](#introduction) {#introduction}

<strong>ビット演算</strong>とは、数値を「ビット（0 と 1 の並び）」として操作する演算です

C 言語やシステムプログラミングでは、複数のオプション（フラグ）を1つの数値にまとめて渡す場面で使われます

```c
int fd = open("file.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
```

### [なぜシステムプログラミングでビット演算が必須なのか](#why-bitwise-is-essential-in-system-programming) {#why-bitwise-is-essential-in-system-programming}

<strong>もし bool 配列でフラグを渡すとしたら？</strong>

```c
/* 仮想的な設計（非効率） */
struct OpenFlags {
    bool wronly;
    bool creat;
    bool trunc;
    /* ... 他のフラグも全て個別に定義 */
};
open("file.txt", &flags, 0644);
```

{: .labeled}
| 問題 | 説明 |
| ------------ | ----------------------------------------------------------------- |
| メモリ効率 | bool 1 つに 1 バイト使う vs ビットなら 32 フラグを 4 バイトに格納 |
| API の複雑さ | フラグが増えるたびに構造体が肥大化 |
| 互換性 | 古い API との後方互換性が困難 |

ビット演算を使えば、1 つの整数で多くのオプションを効率よく表現できます

システムコールの API は 1970 年代から変わっていませんが、フラグの追加は容易です

---

## [目次](#table-of-contents) {#table-of-contents}

- [スイッチで理解するビット演算](#understanding-bitwise-with-switches)
- [よくある疑問](#frequently-asked-questions)
- [実践パターン](#practical-patterns)
- [まとめ](#summary)
- [参考資料](#references)

---

## [スイッチで理解するビット演算](#understanding-bitwise-with-switches) {#understanding-bitwise-with-switches}

4つのスイッチがある部屋を想像してください

```
スイッチ:  [4] [3] [2] [1]
状態:       ON OFF  ON  ON  → 1011
```

---

### [AND（`&`）](#and-operator) {#and-operator}

<strong>スイッチの例：特定のスイッチが ON か確認する</strong>

```
現在:       1101  （スイッチ 1, 3, 4 が ON）
確認したい:  0100  （スイッチ 3 は ON？）
          ──────
結果:       0100  （0 以外 → ON）
```

<strong>計算ルール：両方 1 なら 1（掛け算と同じ）</strong>

```
1 & 1 = 1
1 & 0 = 0
0 & 0 = 0
```

<strong>用途：フラグの確認</strong>

```c
if (flags & O_CREAT) {
    /*
     * O_CREAT が有効
     */
}
```

---

### [OR（`|`）](#or-operator) {#or-operator}

<strong>スイッチの例：スイッチを ON にする</strong>

```
現在:       1001  （スイッチ 1, 4 が ON）
ON にしたい: 0100  （スイッチ 3 を ON にしたい）
          ──────
結果:       1101  （スイッチ 1, 3, 4 が ON）
```

<strong>計算ルール：どちらか 1 なら 1（足し算、ただし上限 1）</strong>

```
1 | 1 = 1
1 | 0 = 1
0 | 0 = 0
```

<strong>用途：フラグを ON / 複数フラグの組み合わせ</strong>

```c
flags |= O_CREAT;
open("file.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
```

---

### [NOT（`~`）](#not-operator) {#not-operator}

<strong>スイッチの例：全部のスイッチを反転する</strong>

```
現在:  1011
       ↓ 反転
結果:  0100
```

<strong>計算ルール：0 と 1 を入れ替える</strong>

```
~1 = 0
~0 = 1
```

<strong>用途：AND と組み合わせてフラグを OFF</strong>

```c
/*
 * O_CREAT だけ OFF
 */
flags &= ~O_CREAT;
```

---

### [XOR（`^`）](#xor-operator) {#xor-operator}

<strong>スイッチの例：押すたびに ON/OFF が切り替わる</strong>

```
現在:       1101  （スイッチ 3 は ON）
切り替える:  0100  （スイッチ 3）
          ──────
結果:       1001  （スイッチ 3 が OFF になった）

もう一度:
現在:       1001  （スイッチ 3 は OFF）
切り替える:  0100  （スイッチ 3）
          ──────
結果:       1101  （スイッチ 3 が ON に戻った）
```

<strong>計算ルール：違ったら 1、同じなら 0</strong>

```
1 ^ 1 = 0
1 ^ 0 = 1
0 ^ 0 = 0
```

<strong>用途：フラグの切り替え（トグル）</strong>

```c
/*
 * ON なら OFF、OFF なら ON
 */
flags ^= O_CREAT;
```

---

### [左シフト（`<<`）](#left-shift) {#left-shift}

<strong>スイッチの例：スイッチを左にずらす</strong>

```
現在:  0001
       ← 1つ左へ
結果:  0010

現在:  0001
       ← 2つ左へ
結果:  0100
```

<strong>計算ルール：1回左にずらすと 2 倍</strong>

```
0001 = 1
0010 = 2  （1 << 1）
0100 = 4  （1 << 2）
1000 = 8  （1 << 3）
```

<strong>用途：フラグの定義</strong>

```c
/*
 * 0001
 */
#define FLAG_A (1 << 0)
/*
 * 0010
 */
#define FLAG_B (1 << 1)
/*
 * 0100
 */
#define FLAG_C (1 << 2)
```

---

### [右シフト（`>>`）](#right-shift) {#right-shift}

<strong>スイッチの例：スイッチを右にずらす</strong>

```
現在:  1000
       → 1つ右へ
結果:  0100

現在:  1000
       → 2つ右へ
結果:  0010
```

<strong>計算ルール：1回右にずらすと半分</strong>

```
1000 = 8
0100 = 4  （8 >> 1）
0010 = 2  （8 >> 2）
```

<strong>用途：2 で割る</strong>

```c
/*
 * = 4
 */
8 >> 1
/*
 * = 2
 */
8 >> 2
```

---

## [よくある疑問](#frequently-asked-questions) {#frequently-asked-questions}

### [なぜフラグは OR で組み合わせるのか](#why-combine-flags-with-or) {#why-combine-flags-with-or}

`open()` の第2引数では、複数のフラグを `|` で組み合わせます

```c
open("file.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
```

<strong>フラグは「重複しないビット位置」で定義されている</strong>

```c
/*
 * 実際の定義（Linux の例）
 */
#define O_RDONLY    00000000
#define O_WRONLY    00000001
#define O_RDWR      00000002
#define O_CREAT     00000100
#define O_TRUNC     00001000
```

それぞれのフラグは「別々のビット」を使っています

OR で組み合わせると、各フラグのビットが「合成」されます

```
O_WRONLY:  00000001
O_CREAT:   00000100
O_TRUNC:   00001000
        ──────────── OR
結果:      00001101  （3つのフラグが ON）
```

<strong>なぜ足し算ではなく OR なのか</strong>

- 足し算だと、同じフラグを2回指定すると値が変わってしまいます
- OR なら、同じフラグを2回指定しても結果は同じです

```c
/*
 * 足し算の場合（危険）
 */
/*
 * 別の値になる！
 */
O_CREAT + O_CREAT = 0200

/*
 * OR の場合（安全）
 */
/*
 * 同じ値のまま
 */
O_CREAT | O_CREAT = O_CREAT
```

---

### [0x1F とは何をしているのか](#what-does-0x1f-do) {#what-does-0x1f-do}

`0x1F` のような値を見たら、<strong>マスク</strong>だと思ってください

マスクとは「特定のビットだけを取り出す」ための値です

<strong>0x1F を2進数で見る</strong>

```
0x1F = 0001 1111  （下位5ビットが全部 1）
```

<strong>AND でマスクを適用する</strong>

```c
/*
 * 214
 */
int value = 0b11010110;
int lower5 = value & 0x1F;
```

```
value:   11010110
mask:    00011111  (0x1F)
      ────────────  AND
result:  00010110  （下位5ビットだけ取り出した）
```

<strong>よく使うマスク</strong>

{: .labeled}
| マスク | 2進数 | 取り出すビット |
| ------ | -------- | -------------- |
| `0x0F` | 00001111 | 下位4ビット |
| `0x1F` | 00011111 | 下位5ビット |
| `0x3F` | 00111111 | 下位6ビット |
| `0x7F` | 01111111 | 下位7ビット |
| `0xFF` | 11111111 | 下位8ビット |

<strong>マスクの作り方</strong>

「下位 n ビット」を取り出すマスクは `(1 << n) - 1` で作れます

```c
/*
 * = 0b100000 - 1 = 0b011111 = 0x1F
 */
(1 << 5) - 1
```

---

## [実践パターン](#practical-patterns) {#practical-patterns}

{: .labeled}
| やりたいこと | 演算 | コード |
| ---------------- | --------- | ---------------------- |
| フラグを ON | OR | `flags \|= O_CREAT;` |
| フラグを OFF | NOT + AND | `flags &= ~O_CREAT;` |
| フラグを切り替え | XOR | `flags ^= O_CREAT;` |
| フラグを確認 | AND | `if (flags & O_CREAT)` |

---

## [まとめ](#summary) {#summary}

{: .labeled}
| 演算 | 記号 | 計算イメージ | スイッチのイメージ | グループ |
| -------- | ---- | ---------------- | ------------------ | -------- |
| AND | `&` | 掛け算 | ON か確認する | 適用系 |
| OR | `\|` | 足し算（上限 1） | ON にする | 適用系 |
| NOT | `~` | 反転 | 全部反転する | 変換系 |
| XOR | `^` | 違ったら 1 | 切り替える | 適用系 |
| 左シフト | `<<` | 2 倍 | 左にずらす | 変換系 |
| 右シフト | `>>` | 半分 | 右にずらす | 変換系 |

<strong>グループの違い</strong>

- 変換系（NOT, シフト）：値を 1 つ用意して変換します
- 適用系（AND, OR, XOR）：値を 2 つ用意して組み合わせます

---

## [参考資料](#references) {#references}

<strong>C 言語規格</strong>

- [Bitwise Operators - cppreference](https://en.cppreference.com/w/c/language/operator_arithmetic#Bitwise_logic_operators){:target="\_blank"}
  - C 言語のビット演算子（&, |, ^, ~）の定義

<strong>Linux マニュアル</strong>

- [open(2) - Linux manual page](https://man7.org/linux/man-pages/man2/open.2.html){:target="\_blank"}
  - O_CREAT, O_TRUNC などのフラグ定義

<strong>本編との関連</strong>

- [05-file-descriptor](../../05-file-descriptor/)
  - open() のフラグを使った実践例
