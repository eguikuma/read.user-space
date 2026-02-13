---
layout: default
title: プロセスを作る
---

# [02-fork-exec：プロセスを作る](#creating-processes) {#creating-processes}

## [はじめに](#introduction) {#introduction}

前のトピック（01-process）で、「プロセスとは実行中のプログラムである」ことを学びました

では、プロセスはどうやって生まれるのでしょうか？

ターミナルで `ls` と打つと、`ls` プロセスが生まれます

でも、誰がそのプロセスを作っているのでしょうか？

答えは<strong>シェル</strong>です

<strong>シェル</strong>とは、ユーザーが入力したコマンドを受け取り、実行するプログラムです

bash や zsh がシェルの例です

詳しくは [01-process](../01-process/) の用語集を参照してください

シェルは、コマンドを実行するために新しいプロセスを作ります

このページでは、プロセスを作る仕組み（fork と exec）を学びます

### [日常の例え](#everyday-analogy) {#everyday-analogy}

fork() と exec() を「分身と変身」と考えてみましょう

fork() は、自分の分身（コピー）を作ることです

exec() は、その分身が別人に変身することです

シェルがコマンドを実行するとき、まず自分の分身を作り（fork）、分身が別のプログラムに変身します（exec）

### [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

<strong>システムコール</strong>とは、プログラムが OS に「これをやって」とお願いする仕組みです

このページでは、以下の 3 つのシステムコールを学びます

- <strong>fork()</strong>
  - 自分のコピー（子プロセス）を作る
- <strong>exec()</strong>
  - 自分自身を別のプログラムに置き換える
- <strong>wait()</strong>
  - 子プロセスの終了を待つ

これらは OS の機能を直接呼び出す<strong>低レベル</strong>な操作です

<strong>低レベル</strong>とは、ハードウェアや OS に近い操作のことです

反対に、<strong>高レベル</strong>とは、人間にとって分かりやすく抽象化された操作のことです

低レベルな操作は難しそうに見えますが、シェルやコンテナなど多くのプログラムの基盤となっています

---

## [目次](#table-of-contents) {#table-of-contents}

1. [fork() とは何か](#what-is-fork)
2. [fork() の戻り値](#fork-return-value)
3. [fork() 後のメモリ](#memory-after-fork)
4. [exec() とは何か](#what-is-exec)
5. [exec() ファミリー](#exec-family)
6. [fork() + exec() パターン](#fork-exec-pattern)
7. [親プロセスの責任：wait()](#parent-responsibility-wait)
8. [ゾンビプロセス](#zombie-process)
9. [孤児プロセス](#orphan-process)
10. [次のステップ](#next-steps)
11. [用語集](#glossary)
12. [参考資料](#references)

---

## [fork() とは何か](#what-is-fork) {#what-is-fork}

### [基本的な説明](#basic-explanation) {#basic-explanation}

<strong>fork()</strong> は、プロセスが自分のコピーを作るシステムコールです

Linux の公式マニュアルには、こう書かれています

> fork() creates a new process by duplicating the calling process.

> fork() は、呼び出したプロセスを複製することで新しいプロセスを作成します

### [fork() で何が起きるか](#what-happens-with-fork) {#what-happens-with-fork}

fork() を呼び出すと、以下のことが起きます

1. 現在のプロセス（親プロセス）のコピーが作られます
2. コピー（子プロセス）は、fork() の直後から実行を開始します
3. 親プロセスと子プロセスは、同時に動き続けます

### [「同時に」とはどういう意味か](#what-does-simultaneously-mean) {#what-does-simultaneously-mean}

「同時に動き続ける」とは、<strong>両方のプロセスが実行可能な状態になる</strong>ということです

実際の実行順序は、OS の<strong>スケジューラ</strong>が決定します

スケジューラとは、「どのプロセスを、どの順番で、どれくらいの時間実行するか」を決める OS の機能です

#### [シングルコアの場合](#single-core) {#single-core}

CPU が 1 つしかないため、本当の意味で「同時に」は実行されません

スケジューラが親と子を高速に切り替えることで、あたかも同時に動いているように見せています

これを<strong>時分割（タイムシェアリング）</strong>と呼びます

時分割では、各プロセスに短い時間（数ミリ秒〜数十ミリ秒）だけ CPU を使う権利が与えられます

この時間が終わると、次のプロセスに切り替わります

切り替えが非常に高速なので、人間には「同時に動いている」ように見えます

#### [マルチコアの場合](#multi-core) {#multi-core}

複数の CPU コアがあれば、親と子が本当に同時に（<strong>並列</strong>に）実行される可能性があります

<strong>並列（パラレル）</strong>とは、複数の処理が物理的に同時に実行されることです

これに対し、<strong>並行（コンカレント）</strong>とは、複数の処理が「同時に進んでいるように見える」ことです

シングルコアでの時分割は「並行」、マルチコアでの同時実行は「並列」です

どのコアでどのプロセスを実行するかは、やはりスケジューラが決めます

#### [親と子、どちらが先に実行されるか](#which-runs-first) {#which-runs-first}

<strong>答えは「わからない」</strong>です

これはスケジューラの判断に依存し、実行ごとに異なる可能性があります

そのため、親と子の実行順序に依存するコードを書いてはいけません

```c
/* 悪い例：子が先に実行されることを期待している */
pid_t pid = fork();
if (pid == 0) {
    /* 子プロセス：ファイルに書き込む */
} else {
    /* 親プロセス：ファイルを読み取る（子が先に書いたと仮定） */
}

/* 良い例：wait() で子の終了を待ってから読み取る */
pid_t pid = fork();
if (pid == 0) {
    /* 子プロセス：ファイルに書き込む */
} else {
    wait(NULL);  /* 子の終了を待つ */
    /* 親プロセス：ファイルを読み取る */
}
```

### [なぜ「コピー」なのか](#why-copy) {#why-copy}

<strong>もしゼロから新しいプロセスを作る設計だったら？</strong>

新しいプロセスを作るとき、すべてを指定する必要があります

- どのプログラムを実行するか
- どの環境変数を渡すか
- どのファイルディスクリプタを開くか
- どのディレクトリを作業ディレクトリにするか
- どのユーザー権限で実行するか

これらをすべて指定する API を作ると、非常に複雑になります

<strong>コピーする設計の利点</strong>

fork() は「自分のコピーを作る」という単純な操作です

子プロセスは自動的に親から以下を継承します

- 開いているファイル
- 環境変数
- 作業ディレクトリ
- ユーザー権限

「ゼロから指定」ではなく「継承してから必要な部分だけ変更」という方式です

このシンプルな設計は初期の UNIX で採用され、今日まで使われ続けています

---

## [fork() の戻り値](#fork-return-value) {#fork-return-value}

fork() は、親プロセスと子プロセスに異なる値を返します

これにより、自分が親なのか子なのかを区別できます

### [戻り値の意味](#meaning-of-return-value) {#meaning-of-return-value}

{: .labeled}
| 戻り値 | 意味 |
| ------------------ | -------------------- |
| 正の数（子の PID） | 親プロセスに返される |
| 0 | 子プロセスに返される |
| -1 | エラー（fork 失敗） |

### [コード例](#code-example) {#code-example}

<strong>pid_t</strong> は、プロセス ID を格納するための型です

詳しくは [01-process](../01-process/) の「C 言語の読み方」を参照してください

```c
pid_t pid = fork();

if (pid < 0) {
    /* エラー処理 */
} else if (pid == 0) {
    /* 子プロセスの処理 */
} else {
    /* 親プロセスの処理 */
}
```

---

## [fork() 後のメモリ](#memory-after-fork) {#memory-after-fork}

fork() でコピーされたプロセスは、<strong>別々のメモリ空間</strong>を持ちます

<strong>メモリ空間</strong>とは、プロセスが使用できるメモリの範囲のことです

OS は各プロセスに独立したメモリ空間を与えます

これにより、あるプロセスが他のプロセスのメモリを壊すことを防ぎます

詳しくは [01-process](../01-process/) の「メモリの構造」を参照してください

### [メモリが分離される](#memory-separation) {#memory-separation}

親プロセスと子プロセスは、同じ変数名を持っていても、別々の変数です

子で変数を変更しても、親の変数には影響しません

### [Copy-on-Write](#copy-on-write) {#copy-on-write}

実際には、fork() 直後は親子で物理メモリページを共有しています

ただし、ページテーブル（仮想アドレスから物理アドレスへのマッピング情報）はコピーされます

そして、どちらかのプロセスが書き込みを行った時点で、その部分だけが実際にコピーされます

これを<strong>Copy-on-Write（CoW）</strong>と呼びます

#### [なぜ効率的なのか](#why-efficient) {#why-efficient}

fork() の後、すぐに exec() を呼ぶ場合を考えてみましょう

exec() はプロセスのメモリを新しいプログラムで完全に置き換えます

もし fork() 時に全メモリをコピーしていたら、exec() で捨てられる分は無駄になります

Copy-on-Write なら、実際に使われるメモリだけをコピーするので、無駄がありません

#### [具体例](#concrete-examples) {#concrete-examples}

```
親プロセス：100MB のメモリを使用

【Copy-on-Write なし】
fork() → 子プロセス用に 100MB をコピー（時間がかかる）
exec() → 子のメモリを新プログラムで上書き（コピーした 100MB は無駄に）

【Copy-on-Write あり】
fork() → ページテーブルをコピーし、物理メモリは共有（一瞬で終わる）
exec() → 子のメモリを新プログラムで上書き（無駄なし）
```

プログラマから見た動作は「完全にコピー」と同じです

---

## [exec() とは何か](#what-is-exec) {#what-is-exec}

### [基本的な説明](#basic-explanation) {#basic-explanation}

<strong>exec()</strong> は、プロセスが自分自身を別のプログラムに置き換えるシステムコールです

Linux の公式マニュアルには、こう書かれています

> The exec() family of functions replaces the current process image with a new process image.

> exec() 関数ファミリーは、現在のプロセスイメージを新しいプロセスイメージに置き換えます

<strong>プロセスイメージ</strong>とは、メモリ上に展開されたプログラムの内容のことです

コード（命令）、データ、スタックなど、プロセスを構成するすべてのメモリ内容を指します

### [exec() で何が起きるか](#what-happens-with-exec) {#what-happens-with-exec}

exec() を呼び出すと、以下のことが起きます

1. 現在のプロセスのプログラム（コード、データ）が新しいプログラムに置き換わります
2. <strong>PID は変わりません</strong>（同じプロセスのまま）
3. 成功すると、exec() 以降のコードは実行されません（別のプログラムになるから）

### [「置き換え」であって「新規作成」ではない](#replacement-not-creation) {#replacement-not-creation}

exec() は新しいプロセスを作りません

自分自身が別のプログラムに「なる」のです

### [なぜ「新規作成」ではなく「置き換え」なのか](#why-replacement-not-creation) {#why-replacement-not-creation}

<strong>もし exec() が新しいプロセスを作る設計だったら？</strong>

- PID が変わってしまう
- 親子関係が変わってしまう（元のプロセスの「子」になる）
- シェルから見ると、実行したコマンドの PID を追跡できなくなる
- ジョブ管理（fg、bg、Ctrl+Z など）が複雑になる

<strong>置き換える設計の利点</strong>

- PID が保持される
- 親子関係が保持される（シェルの子のまま）
- 開いているファイルディスクリプタを引き継げる（リダイレクトの仕組み）
- シェルが子プロセスを簡単に管理できる

fork() と exec() の組み合わせにより、「親から継承」→「必要なものを変更」→「別のプログラムに変身」という柔軟なプロセス生成が可能になります

---

## [exec() ファミリー](#exec-family) {#exec-family}

exec() には複数のバリエーションがあります

### [名前の規則](#naming-convention) {#naming-convention}

{: .labeled}
| 文字 | 意味 |
| ---------------- | -------------------------------------------------------------------- |
| l（list） | 引数をリスト形式で渡す |
| v（vector） | 引数を配列形式で渡す |
| p（PATH） | ファイル名にスラッシュがない場合、PATH を検索する |
| e（environment） | 環境変数を明示的に指定する（指定しない関数は親の環境変数を継承する） |

<strong>リスト形式</strong>とは、引数を関数に直接カンマ区切りで渡す方法です

```c
execlp("ls", "ls", "-l", NULL);  /* 引数を直接列挙 */
```

<strong>配列形式</strong>とは、引数を配列にまとめてから渡す方法です

```c
char *args[] = {"ls", "-l", NULL};
execvp("ls", args);  /* 配列で渡す */
```

<strong>PATH</strong> とは、コマンドを探す場所の一覧を保持する環境変数です

詳しくは [01-process](../01-process/) の「環境変数」を参照してください

<strong>環境変数</strong>とは、プロセスに渡される「名前=値」形式の設定情報です

詳しくは [01-process](../01-process/) の「環境変数」を参照してください

### [よく使う関数](#commonly-used-functions) {#commonly-used-functions}

{: .labeled}
| 関数 | 引数形式 | PATH 検索 | 用途 |
| ------ | -------- | --------- | ------------------------------------------------ |
| execl | リスト | なし | 引数が固定で、完全な場所がわかっている場合 |
| execv | 配列 | なし | 引数が可変で、完全な場所がわかっている場合 |
| execlp | リスト | あり | 引数が固定で、コマンド名だけわかっている場合 |
| execvp | 配列 | あり | 引数が可変で、コマンド名だけわかっている場合 |
| execle | リスト | なし | 環境変数を指定したい場合 |
| execve | 配列 | なし | 唯一のシステムコール（他の関数はこれのラッパー） |

### [使い分けのポイント](#usage-guide) {#usage-guide}

- <strong>よく使うのは execlp() と execvp()</strong>です
- PATH を検索してくれるので便利です
- 引数の数が固定なら execlp()、可変なら execvp() を使います

---

## [fork() + exec() パターン](#fork-exec-pattern) {#fork-exec-pattern}

### [なぜ「コピーしてから置き換える」のか](#why-copy-then-replace) {#why-copy-then-replace}

シェルがコマンドを実行するとき、以下のステップを踏みます

1. fork() で子プロセスを作る
2. 子プロセスで exec() を呼び出す
3. 親プロセスは wait() で子の終了を待つ

もし fork() せずに exec() したら、シェル自体が別のプログラムに置き換わってしまいます

fork() することで、親（シェル）は生き続け、子だけが別のプログラムになります

### [なぜ 2 段階に分けるのか](#why-two-steps) {#why-two-steps}

<strong>もし「fork と exec が一体化した spawn()」のような関数だったら？</strong>

```c
/* 仮想的な spawn() 関数 */
spawn("/bin/ls", args, env);  /* プロセス生成と同時にプログラム実行 */
```

この設計では、fork() と exec() の間で何もできません

<strong>2 段階に分けることで可能になること</strong>

```c
pid_t pid = fork();
if (pid == 0) {
    /* --- この間でいろいろな設定ができる --- */

    /* 1. リダイレクトの設定 */
    close(1);                        /* 標準出力を閉じる */
    open("output.txt", O_WRONLY);    /* ファイルを fd=1 で開く */

    /* 2. パイプの接続 */
    dup2(pipe_fd[1], 1);             /* パイプに標準出力を接続 */

    /* 3. 環境変数の変更 */
    setenv("DEBUG", "1", 1);

    /* 4. 作業ディレクトリの変更 */
    chdir("/tmp");

    /* --- 設定完了後に exec --- */
    execvp("ls", args);
}
```

シェルの `ls > output.txt` というコマンドは、まさにこの仕組みで実現されています

fork() と exec() を分離したことで、UNIX はシンプルな部品の組み合わせで複雑な動作を実現できます

### [シェルの動作原理](#shell-operation-principle) {#shell-operation-principle}

シェルは、以下のループを繰り返しています

1. プロンプトを表示してコマンドを読み取る
2. fork() で子プロセスを作る
3. 子プロセスで exec() を呼び出す
4. 親プロセスは wait() で子の終了を待つ
5. 1 に戻る

---

## [親プロセスの責任：wait()](#parent-responsibility-wait) {#parent-responsibility-wait}

### [wait() とは](#what-is-wait) {#what-is-wait}

<strong>wait()</strong> は、子プロセスの終了を待つシステムコールです

Linux の公式マニュアルには、こう書かれています

> wait() and waitpid() are used to wait for state changes in a child of the calling process.

> wait() と waitpid() は、呼び出したプロセスの子の状態変化を待つために使用されます

### [なぜ wait() が必要なのか](#why-wait-is-needed) {#why-wait-is-needed}

1. <strong>子の終了ステータスを取得するため</strong>
   - 子がどのように終了したか（正常終了、シグナルで終了など）を知る
   - 終了コードを取得する

2. <strong>ゾンビプロセスを防ぐため</strong>
   - wait() しないと、子はゾンビになる
   - ゾンビについては次のセクションで説明

### [終了ステータスの解析](#analyzing-exit-status) {#analyzing-exit-status}

wait() で取得したステータスは、以下の<strong>マクロ</strong>で解析します

<strong>マクロ</strong>とは、C 言語で名前に値や処理を割り当てる仕組みです

ここで使うマクロは、ステータス値を解析するための便利な関数のように使えます

{: .labeled}
| マクロ | 説明 |
| ------------------- | -------------------------- |
| WIFEXITED(status) | 正常終了したかどうか |
| WEXITSTATUS(status) | 終了コード（0〜255） |
| WIFSIGNALED(status) | シグナルで終了したかどうか |
| WTERMSIG(status) | 終了させたシグナル番号 |

---

## [ゾンビプロセス](#zombie-process) {#zombie-process}

### [ゾンビプロセスとは](#what-is-zombie) {#what-is-zombie}

<strong>ゾンビプロセス</strong>は、子プロセスが終了したが、親が wait() していない状態です

[01-process](../01-process/) で「プロセスの状態」として触れましたが、ここでは「なぜ生まれるか」を学びます

### [なぜゾンビが生まれるのか](#why-zombie-occurs) {#why-zombie-occurs}

<strong>もし子プロセスが終了と同時に完全に消えたら？</strong>

親プロセスは「子がどうなったか」を知ることができません

- 子は正常に終了したのか、エラーで終了したのか？
- 終了コードは何だったのか？
- シグナルで強制終了されたのか？

この情報は、親が適切に処理を続けるために必要です

例えば、シェルは `$?` でコマンドの終了コードを参照できます

```bash
ls /nonexistent
echo $?   # 2（エラーを示す終了コード）
```

<strong>カーネルが終了情報を保持する必要がある</strong>

1. 子プロセスが終了する
2. カーネルは「親に終了を伝える」ために、子のエントリを残す
3. 親が wait() するまで、子はプロセステーブルに残り続ける
4. この状態が「ゾンビ」

ゾンビは「親が終了情報を取得するのを待っている」状態です

### [ゾンビを消す方法](#how-to-eliminate-zombie) {#how-to-eliminate-zombie}

- 親が wait() を呼ぶ
- 親が終了すると、子は init（PID 1）に引き取られ、init が wait() する

<strong>init</strong> とは、Linux が起動したときに最初に実行されるプロセスです

PID 1 を持ち、すべてのプロセスの「先祖」となります

現在のほとんどの Linux ディストリビューションでは、<strong>systemd</strong> が init の役割を担っています

### [ゾンビが占有するリソース](#zombie-resources) {#zombie-resources}

ゾンビプロセスは、すでに終了しているため CPU やメモリはほとんど使いません

しかし、以下のリソースを占有し続けます

#### [プロセステーブルのエントリ](#process-table-entry) {#process-table-entry}

OS は、実行中のすべてのプロセスを<strong>プロセステーブル</strong>という表で管理しています

<strong>エントリ</strong>とは、表の中の「1 行」のことです

各プロセスに対応する 1 行のデータがプロセステーブルに格納されています

ゾンビはプロセステーブルに「終了したが、まだ回収されていない」という情報を残し続けます

このエントリには以下の情報が含まれています

- PID（プロセス ID）
- 終了ステータス
- 使用した CPU 時間などの統計情報

#### [PID の枯渇](#pid-exhaustion) {#pid-exhaustion}

Linux では、PID は有限の数値です（通常、最大 32768 または 4194304）

ゾンビは PID を解放しないため、大量のゾンビが発生すると新しいプロセスを作成できなくなります

```
$ cat /proc/sys/kernel/pid_max
32768
```

#### [具体的なシナリオ](#concrete-scenario) {#concrete-scenario}

以下のようなサーバプログラムを考えてみましょう

```c
/* 問題のあるサーバコード */
while (1) {
    pid_t pid = fork();
    if (pid == 0) {
        /* 子プロセス：リクエストを処理して終了 */
        handle_request();
        exit(0);
    }
    /* 親プロセス：wait() を呼ばずに次のリクエストを待つ */
}
```

このサーバがリクエストを処理するたびに、ゾンビが 1 つ増えます

1 日に 10,000 リクエストを処理すると、1 日で 10,000 個のゾンビが生まれます

数日で PID が枯渇し、サーバは新しいプロセスを作成できなくなります

#### [正しい実装](#correct-implementation) {#correct-implementation}

```c
/* 正しいサーバコード */
while (1) {
    pid_t pid = fork();
    if (pid == 0) {
        handle_request();
        exit(0);
    }
    /* 親プロセス：終了した子を回収する */
    waitpid(-1, NULL, WNOHANG);  /* ブロックせずに回収 */
}
```

または、SIGCHLD シグナルを使って子の終了を検知する方法もあります（03-signal で詳しく学びます）

---

## [孤児プロセス](#orphan-process) {#orphan-process}

### [孤児プロセスとは](#what-is-orphan) {#what-is-orphan}

<strong>孤児プロセス</strong>は、親プロセスが先に終了した子プロセスです

### [孤児は init に引き取られる](#orphan-adopted-by-init) {#orphan-adopted-by-init}

親が終了すると、子の PPID は 1 になります

これは init（または systemd）に引き取られたことを意味します

### [孤児はゾンビにならない](#orphan-not-zombie) {#orphan-not-zombie}

init は孤児が終了した際に wait() を呼び出します

そのため、孤児が終了しても init が適切に回収するので、ゾンビにはなりません

### [ゾンビと孤児の違い](#difference-zombie-orphan) {#difference-zombie-orphan}

{: .labeled}
| 状態 | ゾンビ | 孤児 |
| ------------------ | --------------------- | --------------------------- |
| 何が先に終了したか | 子が先に終了 | 親が先に終了 |
| 子の状態 | 終了済み（回収待ち） | まだ動いている |
| 問題 | wait() されるまで残る | 問題なし（init が引き取る） |

---

## [次のステップ](#next-steps) {#next-steps}

このトピックでは、「プロセスを作る方法」を実践的に学びました

- fork() で自分のコピーを作る
- exec() で自分を別のプログラムに置き換える
- wait() で子の終了を待つ
- ゾンビと孤児の違い

次の [03-signal](../03-signal/) では、プロセスに「通知」を送る方法を学びます

- `Ctrl+C` を押すとプログラムが止まるのはなぜか
- 子プロセスが終了したことを、親はどうやって知るのか
- プロセスに「終了しろ」と伝える方法は何か

これらの疑問に答えます

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 英語 | 説明 |
| ---------------- | -------------------- | -------------------------------------------------------------------------------------------------- |
| フォーク | fork | プロセスが自分のコピーを作ること |
| エグゼック | exec | プロセスが自分を別のプログラムに置き換えること（PID は変わらず、成功すると呼び出し元には戻らない） |
| 親プロセス | Parent Process | fork() を呼び出したプロセス |
| 子プロセス | Child Process | fork() によって生まれた新しいプロセス |
| ゾンビプロセス | Zombie Process | 終了したが親が wait() していないプロセス |
| 孤児プロセス | Orphan Process | 親が先に終了したプロセス |
| 終了ステータス | Exit Status | プロセスが終了時に返す数値（0 は成功を意味する） |
| 終了コード | Exit Code | 終了ステータスの別名 |
| ブロッキング | Blocking | 処理が完了するまで次に進まない動作 |
| システムコール | System Call | プログラムが OS の機能を呼び出す仕組み |
| Copy-on-Write | Copy-on-Write（CoW） | 変更が発生するまで物理メモリを共有する最適化 |
| スケジューラ | Scheduler | どのプロセスを実行するか決定する OS の機能 |
| 時分割 | Time Sharing | CPU 時間を細かく区切って複数プロセスを交互に実行する方式 |
| 並行 | Concurrent | 複数の処理が同時に進んでいるように見えること |
| 並列 | Parallel | 複数の処理が物理的に同時に実行されること |
| メモリ空間 | Memory Space | プロセスが使用できるメモリの範囲 |
| プロセスイメージ | Process Image | メモリ上に展開されたプログラムの内容 |
| マクロ | Macro | C 言語で名前に値や処理を割り当てる仕組み |
| init | init | Linux 起動時に最初に実行されるプロセス（PID 1） |
| systemd | systemd | 現在のほとんどの Linux で init の役割を担うプロセス管理システム |
| プロセステーブル | Process Table | OS がすべてのプロセスを管理するための表 |
| エントリ | Entry | テーブル（表）の中の 1 行分のデータ |
| 低レベル | Low-level | ハードウェアや OS に近い操作 |
| 高レベル | High-level | 人間にとって分かりやすく抽象化された操作 |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

- [fork(2) - Linux manual page](https://man7.org/linux/man-pages/man2/fork.2.html){:target="\_blank"}
  - プロセスの複製
- [execve(2) - Linux manual page](https://man7.org/linux/man-pages/man2/execve.2.html){:target="\_blank"}
  - プログラムの実行（exec ファミリーの基盤）
- [exec(3) - Linux manual page](https://man7.org/linux/man-pages/man3/exec.3.html){:target="\_blank"}
  - exec ファミリーのフロントエンド
- [wait(2) - Linux manual page](https://man7.org/linux/man-pages/man2/wait.2.html){:target="\_blank"}
  - 子プロセスの状態変化を待つ
- [exit(3) - Linux manual page](https://man7.org/linux/man-pages/man3/exit.3.html){:target="\_blank"}
  - プロセスの正常終了
