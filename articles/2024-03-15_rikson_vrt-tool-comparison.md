---
title: "Visual Regression Testingツールの比較検討"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["VRT"]
published: true
publication_name: micin
---

本稿ではVisual Regression Testingツール導入の際に弊社で比較検討した内容を紹介します。導入したいけど、どのツールが良いのか分からないという方のお役に立てれば幸いです。

# Visual Regression Testing (VRT) とは

Visual Regression Test (VRT)とはアプリケーションの外観の変化を自動検出するリグレッションテストの手法です。意図しない変更が加わっていないことを外観ベースで確認できます。

UIオートメーションによってアプリケーションを操作し、比較元のキャプチャ（ベースライン）との画素の差分を検出します。差分が閾値を超えるとテストが落ちるので、開発者はソースコードの変更を安心して行うことができます。

似たような手法にDOMの構造を比較するスナップショットテストがありますが、DOMの構造だけでは判定できない範囲もVRTによってカバーできます。例えば、グローバルなCSSやVideoタグで描画される映像などはDOMの構造には反映されません。

VRTの一番のメリットは一言でいうと「**安心できる**」ということだと考えています。ロジックベースのテストをいくら積み上げても、開発者の想定の範囲外はテストできません。全てのテストが成功したら正常に動作しているであろうという推論に過ぎず、やはり目視で確認したくなってしまいます。
それは、テスト結果とアプリケーションの正常性に乖離を感じているからだと思います。VRTでは人間が目視で確認するのとほぼ同じ方法でアプリケーションをテストするので、ロジックベースのテストを補完できます。

# VRT導入の経緯

VRT自体は社内の一部のリポジトリでは既に導入されていました。所属部署内でもVRT導入の話が持ち上がった際に、他部署ではChromaticを無料プランで利用している旨を上長から伺い、横断的に導入の検討してはどうかと提案を受けました。
そのような経緯で、自分が音頭を取って社内でVRT手法の比較検討をする運びとなりました。

部署横断的に話し合うことの狙いとしては次のようなものでした。

- ツール投資のコスト効率を高めたい
- 水面下の意思決定を避けたい
- ナレッジの蓄積
- 品質の底上げ

部署ごとに違うやり方をしていると、せっかく有料ツールを導入しても効果を最大化できません。
また、別のツールを使いたかったのに、他部署で勝手に話が進んでしまっていると不公平感が出てしまいます。

# ツールの選定

要求としては「StorybookにおけるStory単位での統合テスト」と「UIオートメーションを用いたE2Eテスト」の実現でした。
利用用途のほとんどは前者でしたが、統合テストが整備できたら、徐々にE2Eにも広げてマニュアルテストの工数を削減したいという目論見もあります。

ここで紹介したツール以外にもっと知りたい方は、[awesome-regression-testing](https://github.com/mojoaxel/awesome-regression-testing)というリポジトリに関連ツールが多数紹介されています。

https://github.com/mojoaxel/awesome-regression-testing

## Playwright

VRTツールのサブパッケージとして幅広く利用されているPlaywrightですが、単体でもVRTはできます。Storybook以外のVRTを簡易的に構築したい場合は、Playwrightを利用するのも1つの手段です。

最小構成で始めるなら、ベースラインはリポジトリに含めるのが簡単です。ベースラインを作成するOSはCIのOSに一致させないと不要な差分が検出されてしまうため、開発OSとCIのOSを合わせる必要があります。
開発OSはmacOSなのですが、プライベートリポジトリのGitHub ActionsでmacOSを利用するのは料金が高くなるため、運用費を節約するためにはCI側はLinuxにする必要があります。DockerでLinuxを動かして、StorybookをDockerに寄せる構成が良さそうです。

GitHub Actionsの具体的な設定方法は次の公式ドキュメントに記載されています。

https://playwright.dev/docs/ci-intro

差分表示は次のようにDiff、Actual、Expectedで切り替えて確認できます。赤色に表示されているのは画素の差分です。

![playwright](/images/rikson_vrt-tool-comparison_playwright.png)

## Storybook Test Runner

READMEに記載されているように、Storybook Test RunnerでもサブパッケージのPlaywrightを使ってVRTを実施できます。

https://github.com/storybookjs/test-runner/tree/next?tab=readme-ov-file#image-snapshot

基本は前述のPlaywrightでの利用方法と同じで、Dockerを利用する形になります。
参考までに[サンプルリポジトリ](https://github.com/rikuson/storybook-test-runner-sample)を作成してみました。差分が生じるとArtifactに差分表示ファイルがアップロードされるようにしました。

ダウンロードして展開すると画像ファイルが生成されており、左がExpected、右がActual、中央がDiffという形式になっています。表示が見づらいのが難点です。

![storybook-test-runner](/images/rikson_vrt-tool-comparison_storybook-test-runner.png)

## Chromatic

VRTはもちろん、Storybookをホスティングしたり、GitHubやFigmaとの連携機能を提供するフリーミアムのサービスです。Storybookのメンテナーによって開発、運用されています。リポジトリを連携させればすぐに始められる手軽さが魅力です。ベータ版の機能ですが、Playwrightも動かせるようになっています。

![chromatic](/images/rikson_vrt-tool-comparison_chromatic.gif)
「出典：[Chromatic](https://www.chromatic.com/)」

ベースラインはリポジトリで管理する必要はなく、クラウド上に保存されます。
UI実装に対して、デザイナーやプロダクトマネージャーからフィードバックをもらうようなワークフローが構築可能です。
うまくハマれば手放せないツールとなりそうです。

## reg-viz

[reg-viz](https://github.com/reg-viz)はVRTのためのOSSを複数提供しています。[reg-suit](https://github.com/reg-viz/reg-suit)を使って、GitHub ActionsとS3を組み合わせることで、Chromaticと似た仕組みを自前で構築できます。

![reg-suit](/images/rikson_vrt-tool-comparison_reg-suit.png)
「出典：[reg-viz](https://reg-viz.github.io/reg-suit/)」

[reg-actions](https://github.com/reg-viz/reg-actions)で、GitHub ActionsとGitHub Artifactを利用して、より簡易的な利用もできます。GitHub PRコメントで差分を確認できます。

![reg-actions](/images/rikson_vrt-tool-comparison_reg-actions.png)
「出典：[Visual Regression Testをサポートするreg-actionsをリリースした](https://zenn.dev/fraim/articles/e020e82985ac6d)」

# ツールの選定結果

PlaywrightやStorybook Test Runnerではベースラインを作成するためにローカルにDockerを構築する必要がある点で反対意見が出ました。

費用を度外視すればChromaticが最も使いやすいことは我々の中で共通認識がありました。しかし、社内のVRT利用量を鑑みると無料枠には収まらず、有償プランの中で検討する必要がありました。

reg-suitを利用すれば、VRTにおいてChromaticと類似した機能を実現できますが、インフラを運用する手間がやや増えてしまいます。

ChromaticのStarterプランなら総合的なメリットが上回るだろうという判断で、Chromaticを導入することが決まりました。ChromaticのPlaywrightサポートに伴い、E2EテストをChromaticで管理するという活用も検討しています。今後、利用上限に達することがあれば、使用感を鑑みて代替手段も再考したいと思います。

---

MICINではメンバーを大募集しています。
「とりあえず話を聞いてみたい」でも大歓迎ですので、お気軽にご応募ください！

https://recruit.micin.jp/
