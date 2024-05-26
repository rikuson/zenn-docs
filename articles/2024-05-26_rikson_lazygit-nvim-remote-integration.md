---
title: "NeovimからLazygitを開いて、LazygitからNeovimを開く"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["neovim"]
published: true
---

[Lazygit](https://github.com/jesseduffield/lazygit)というGitクライアントを知っていますか？

![Lazygit](/images/rikson_lazygit-nvim-remote-integration_lazygit.gif)

「出典：[Lazygit](https://github.com/jesseduffield/lazygit)」

このような感じで**スーパーハカー感**の溢れるTUIツールです。
ドヤ顔でTUIツールを操作することで、技術力を雰囲気でカヴァーすることが出来るようになります！

# toggleterm.nvim

LazygitをNeovimから開けるようにすることで、ソースコードの編集中にシームレスにGitの操作を行うことができます。
これを行うための[lazygit.nvim](https://github.com/kdheepak/lazygit.nvim)というプラグインも存在します。lazygit.nvimを利用したい場合は[こちら](https://github.com/kdheepak/lazygit.nvim/tree/ad3e1ea592f9d13e86e0d4e850224d9d78069508?tab=readme-ov-file#usage)を参考にしてください。（「Using neovim-remote」の辺り）

![toggleterm.nvim](/images/rikson_lazygit-nvim-remote-integration_toggleterm.nvim.png)

「出典: [toggleterm.nvim](https://github.com/akinsho/toggleterm.nvim)」

自分はLazygitに限らず、他のツールもNeovimから操作したいと考えたので、[toggleterm.nvim](https://github.com/akinsho/toggleterm.nvim)というプラグインを利用してインテグレーションを作成しました。toggleterm.nvimの設定は次の記事が参考になりました。

https://zenn.dev/stafes_blog/articles/524e4c8c80db24

ただ、一つ気に入らない点がありました。
NeovimからLazygitを開いて、ファイル一覧から `e` で編集しようとすると、次のようにNeovimがネストしてしまうのです。

![nested-nvim](/images/rikson_lazygit-nvim-remote-integration_nested-nvim.gif)

Lazygitに設定を加えることで、この問題を解決できます。

# neovim remote

実はNeovimにはリモート操作機能が存在します。
実行中Neovimに対して、外部からコマンドを送信したり、特定のファイルを開くように指示を出すことができます。

https://neovim.io/doc/user/remote.html

次の例では左側で起動しているNeovimを、右側のシェルからインサートモードにしてHelloWorldを入力させています。

[![nvim-remote](/images/rikson_lazygit-nvim-remote-integration_nvim-remote.gif)](/images/rikson_lazygit-nvim-remote-integration_nvim-remote.gif)

この機能を用いてLazygitとNeovimを連携させることができます。

# 設定ファイルの編集

Lazygitの設定ファイルを編集しましょう。

Macの場合、Lazygitの設定ファイルはデフォルトで `~/Library/Application\ Support/lazygit/config.yml` に存在します。自分はパスを変更して `~/.config/lazygit/config.yml` にしています。
詳細は次のページを確認してください。

https://github.com/jesseduffield/lazygit/blob/master/docs/Config.md

nvim-remote用にプリセットが用意されているので、これを用いて次のように設定ファイルを記載します。

```yaml
os:
  editPreset: "nvim-remote"
```

上記のプリセットを設定することで、Lazygitからファイルを編集する際に裏では次のようなコマンドが実行されています。

```shell
[ -z "$NVIM" ] && (nvim -- {{filename}}) || (nvim --server "$NVIM" --remote-send "q" && nvim --server "$NVIM" --remote-tab {{filename}})
```

Neovim内のTerminalには `$NVIM` という環境変数にNeovimのサーバーを一意に表すアドレスがセットされています。
これを利用して、Neovimの中からLazygitを開いていたら `q` でLazygitを閉じて、親のNeovimに選択されたファイルを新規タブで開かせています。

# 動作確認

![lazygit-with-nvim-remote](/images/rikson_lazygit-nvim-remote-integration_lazygit-with-nvim-remote.gif)

良好な親子関係ですね。これで、NeovimとLazygitの間を行き来しやすくなりました。

このようにRemote機能を利用することで、他にもNeovimの設定の幅が広がりそうです。
