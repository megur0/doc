---
title: "Git worktreeとClaude Codeを使った並列開発の基礎 - Claude"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(Claude)](./README.md) > Git worktreeとClaude Codeを使った並列開発の基礎


## Git worktreeとClaude Codeを使った並列開発の基礎

### 1. Git worktreeとは

`git worktree` は、**1つのGitリポジトリから複数の作業ディレクトリ（worktree）を作成し、それぞれで異なるブランチを同時にチェックアウトできる仕組み**です。

通常のGitでは、1つの作業ディレクトリに対して同時に1つのブランチしかチェックアウトできません。

`git worktree`を使うと、例えば次のような構成にできます。

```
project/                 → main
project-feature-a/       → feature-a
project-feature-b/       → feature-b
```

これらは別々のディレクトリですが、同じGitリポジトリを共有しています。

### 2. Git worktreeで共有されるものと共有されないもの

worktreeでは、**作業ファイルは分離され、Gitリポジトリは共有されます。**

#### 共有されるもの

* Gitの履歴
* Gitオブジェクト
* ローカルブランチなどのGitリポジトリ情報
* リポジトリとしてのGit管理基盤

#### 共有されないもの

* 各worktreeの作業中のファイル
* 各worktreeでの未コミット変更
* 各worktreeのHEAD
* 各worktreeでチェックアウトされているブランチ

例えば、feature用worktreeで `src/game.ts` を変更しても、main用worktreeの `src/game.ts` は変わりません。

feature側でコミットを作成した場合、そのコミット自体は同じGitリポジトリの履歴としてmain側からも認識できます。

しかし、main側のファイルが自動的にそのコミットの状態になるわけではありません。

mainへ変更を取り込むには、通常どおり `git merge` や `git rebase` などを行います。

### 3. Git worktreeではgit pullは必要なのか

同じローカルGitリポジトリに属するworktree間で、ブランチやコミットを共有するために `git pull` を行う必要はありません。

例えばfeature worktreeで、

```
git commit -m "Add feature"
```

を実行すると、そのコミットは同じローカルGitリポジトリの履歴に存在します。

別のworktreeからも、そのコミットやブランチを認識できます。

ただし、mainのブランチにその変更を反映するには、

```
git merge feature
```

などの操作が必要です。

また、GitHubなどのリモートリポジトリ上の変更を取得するための `git pull` は別の話です。

つまり、

* worktree間のローカルGit情報の共有 → pull不要
* リモートリポジトリから最新情報を取得 → pullが必要

と考えると分かりやすいです。

### 4. git cloneとの違い

`git clone` は、Gitリポジトリそのものを別の場所に複製する操作です。

例えば、

```
project-a/
project-b/
```

という2つのディレクトリを、それぞれ `git clone` で作った場合、Gitから見ると別々のローカルリポジトリです。

一方、worktreeでは、

```
repository
   ├── main worktree
   ├── feature-a worktree
   └── feature-b worktree
```

というように、複数の作業ディレクトリが1つのGitリポジトリを共有します。

#### cloneの場合

それぞれのcloneは独立したGitリポジトリなので、通常は、

```
clone A
   ↓ push
remote repository
   ↓ pull
clone B
```

という形で変更を共有します。

#### worktreeの場合

同じローカルGitリポジトリを共有しているため、ローカルのworktree同士で情報を共有するためにリモートリポジトリを経由する必要はありません。

### 5. worktreeと通常のブランチの違い

通常のブランチでは、

```
main
  ↓
git checkout -b feature-x
  ↓
feature-x
```

のようにブランチを切り替えて作業します。

この場合、作業ディレクトリは1つです。

そのため、mainで作業しながら同時にfeature-xを別のディレクトリで作業することはできません。

worktreeを使うと、

```
project/              → main
project-feature-x/    → feature-x
```

のように、同時に複数ブランチを別々のディレクトリで扱えます。

したがって、

* 通常のブランチ → ブランチを分ける
* worktree → ブランチに加えて作業ディレクトリも分ける

と考えると分かりやすいです。

### 6. worktreeとcloneの違い

大まかには次のように整理できます。

| 方法        | ブランチ | 作業ディレクトリ | Gitリポジトリ |
| --------- | ---- | -------- | -------- |
| 通常のbranch | 分離   | 共有       | 1つ       |
| worktree  | 分離   | 分離       | 共有       |
| clone     | 分離可能 | 分離       | 別々       |

worktreeは、**「Gitリポジトリは共有したまま、作業場所だけを分けたい」**場合に適しています。

cloneは、**「Gitリポジトリ自体も完全に分離したい」**場合に適しています。

### 7. worktreeの主なメリット

worktreeの大きなメリットは、複数のブランチを同時に別々のディレクトリで扱えることです。

例えば、

```
main
   ├── feature/ui
   ├── feature/network
   └── feature/inventory
```

という複数の作業を、それぞれ別worktreeで同時に進められます。

特に次のようなケースで有効です。

* 複数の作業を並列して進めたい
* 複数のAIエージェントを同時に動かしたい
* 長時間かかる作業を別環境で実行したい
* 現在の作業ディレクトリを変更せずに別ブランチを作業したい
* 実験的な変更を独立した環境で行いたい

### 8. Claude Codeとworktree

Claude Codeでは、Git worktreeを利用してClaude用の独立した作業環境を作ることができます。

例えば、

```
main worktree
    ↓
通常の開発環境

Claude用worktree
    ↓
feature-x
    ↓
Claude Codeが作業
```

という構成にできます。

これにより、Claude Codeに大規模な変更をさせても、現在のmain側の作業環境を直接変更せずに済みます。

### 9. Claude Codeにおける「コンテキストの分離」

Claude Codeとworktreeを組み合わせる場合、**AIエージェントの作業コンテキストを分離しやすい**ことも大きなメリットです。

例えば、

```
Claude A
├── worktree A
├── feature-A
└── feature-Aに関する作業

Claude B
├── worktree B
├── feature-B
└── feature-Bに関する作業
```

のように、複数のClaude Codeセッションを独立して動かせます。

これにより、

* Claude Aは機能Aの実装に集中
* Claude Bは機能Bの実装に集中

という並列開発がしやすくなります。

ただし、**「コンテキストの分離」はGit worktreeそのものが提供する機能ではありません。**

Git worktreeが提供するのは、あくまでGit上の作業ディレクトリとブランチの分離です。

Claude Codeがそれぞれのworktreeで独立したセッションを動かすことで、結果としてAIの作業コンテキストも分離できます。

### 10. Claude Codeでworktreeを使う意味

Claude Codeでは、AIエージェントが複数ファイルを変更したり、大規模なリファクタリングを行ったりすることがあります。

そのため、

```
現在の作業環境
      ↓
Claude用の独立したworktree
      ↓
AIが自由に変更
      ↓
人間が確認
      ↓
必要ならmerge
```

という流れにすると安全です。

特に複数のClaude Codeセッションを並列実行する場合、worktreeのメリットが大きくなります。

### 11. worktreeを使わなくてもよいケース

Claude Codeを1セッションずつ順番に使うだけなら、必ずしもworktreeを使う必要はありません。

例えば、

```
git checkout -b feature-x
```

として普通のブランチを作り、

```
Claude Codeで実装
    ↓
動作確認
    ↓
mainへmerge
    ↓
feature-xを削除
```

という運用でも十分です。

worktreeは、特に「複数の作業を同時に進めたい」という状況で価値が高くなります。

### 12. worktreeの後片付け

worktreeは、ブランチをmergeしただけでは通常自動的には削除されません。

例えば、

```
git merge feature-x
```

でmainに変更を取り込んだ後、

```
git worktree remove <worktree-path>
```

でworktreeを削除できます。

その後、

```
git branch -d feature-x
```

で不要になったブランチを削除します。

基本的には、

```
作業
  ↓
commit
  ↓
mainへmerge
  ↓
worktreeをremove
  ↓
不要なbranchをdelete
```

という順番で整理できます。

### 13. 3つの方法を一言で整理

#### git checkout -b

「ブランチだけ分ける」

作業ディレクトリは基本的に1つなので、ブランチを切り替えながら作業します。

#### git worktree

「ブランチと作業ディレクトリを分ける」

Gitリポジトリは共有したまま、複数ブランチを同時に別ディレクトリで作業できます。

#### git clone

「Gitリポジトリそのものを分ける」

それぞれが独立したGitリポジトリになります。通常はリモートリポジトリを介して変更を共有します。

### 14. 最終的な使い分け

基本的には次のように考えるとよいです。

```
1つの作業を順番に進める
    ↓
git checkout -b

複数の作業を同時に進める
    ↓
git worktree

Gitリポジトリ自体を完全に分離する
    ↓
git clone

複数のAIエージェントを並列で動かす
    ↓
git worktree + Claude Code
```

特にClaude Codeでは、

**「コードの作業場所を分離する」＋「AIエージェントの作業コンテキストを分離する」**

という組み合わせによって、複数のAIエージェントを安全に並列稼働させやすくなることが、worktreeを利用する大きなメリットです。
