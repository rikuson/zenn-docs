---
title: "GStreamerの概要とRustでの呼び出し方"
emoji: "👼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "GStreamer"]
published: true
publication_name: micin
---

GStreamerは、オープンソースのマルチメディアフレームワークであり、音声や動画のストリーミング、編集などを柔軟に行うことができます。噛み砕いて言うと「プログラマブルなFFmpeg」みたいな感じです。
8月に弊社で開催された開発合宿というイベントにて、AI対話ボットを作成するために利用しました。近いうちにAI対話ボットについてもご紹介できればと思っています。

本稿では、GStreamerの概要、Rustでの呼び出し方を紹介します。
環境はmacOS@12.7.4、GStreamer@1.24.6、Rust@1.79.0です。

# インストール

:::message
Homebrew経由だとwebrtcbinがうまくインストールできず、Dockerだとrswebrtcがうまくインストールできないという事象に遭遇しました。
これらの問題を回避するため、GStreamerの公式サイトから直接バイナリをダウンロードしてインストールしました。
:::

以下の手順に従って、必要なバイナリをダウンロードしてインストールします。

1. [GStreamer ダウンロードページ](https://gstreamer.freedesktop.org/download/#macos)にアクセス
2. `runtime installer`と`development installer`の両方をダウンロード
3. ダウンロードしたインストーラーを実行し、指示に従ってインストール

インストールが完了すると、GStreamerの実行に必要なライブラリやツールが`/Library/Frameworks/GStreamer.framework`配下に配置されます。
次のようにコマンドラインツールにパスを通しておきましょう。

```shell
export PATH="/Library/Frameworks/GStreamer.framework/Commands:$PATH"
export PKG_CONFIG_PATH="/Library/Frameworks/GStreamer.framework/Versions/Current/lib/pkgconfig"
```

# 動作確認

下記のコマンドを実行して動作を確認しましょう。「ポー」という音が聞こえればOKです。

```shell
gst-launch-1.0 audiotestsrc ! autoaudiosink
```

`gst-launch-1.0`は、GStreamerのコマンドラインツールの1つで、GStreamerパイプラインを簡単に構築して実行するためのユーティリティです。プログラミングせずに直接コマンドラインから試すことができます。プロダクションで使うためのものではありません。

# GStreamerの基本要素

GStreamerは複数の要素（element）を組み合わせたパイプラインを構成することで、様々な処理を実現できます。プラグインベースのアーキテクチャを採用しており、プラグインを追加することで容易に機能を拡張可能です。

![pipeline](/images/rikson_gstreamer-for-rustacean_pipeline.drawio.png)

ここでは、最も基本的な`src`要素と`sink`要素に加えて、`filter`、`mux`、および`demux`要素について説明します。

:::details プラグインの分類
GStreamerではプラグインをインストールする際に、複数のプラグインをまとめてインストールできるプラグインセットが提供されています。

- gstreamer1.0-plugins-base: 基本セット (重要で典型的なコンポーネント一式)
- gstreamer1.0-plugins-good: LGPLライセンスで配布されている良質なプラグインセット
- gstreamer1.0-plugins-bad: 品質（コードレビュー、ドキュメンテー ション、テストコード、アクティブなメンテナ、需要）が基準に達していないプラグインセット
- gstreamer1.0-plugins-ugly: 良質ながらライセンスで配布時に問題の起 こる可能性があるプラグインセット
  :::

## source

`src`要素は、メディアデータを生成または取得して、パイプラインにデータを流し込む役割を持ちます。例えば、`audiotestsrc`は、サイン波などのテスト音声を生成するソース要素です。その他にも、ファイルからメディアデータを読み込む`filesrc`や、ネットワークからデータを受け取る`udpsrc`など、さまざまなソース要素が用意されています。

## sink

`sink`要素は、パイプライン内のデータを消費し、再生・保存などを行います。例えば、`autoaudiosink`は、システムのデフォルト音声出力デバイスで音声を再生するシンク要素です。保存する場合は`filesink`、RTPで配信する場合は`rtpsink`などが利用できます。

## filter

`filter`要素は、パイプライン内でデータを変換したり、加工したりする役割を持ちます。例えば、音声データにエフェクトをかける場合や、ビデオの解像度を変更する場合などに使用されます。

以下は、`filter`要素を使用して音声にエコーをかける例です。

```shell
gst-launch-1.0 audiotestsrc wave=ticks ! audioconvert ! audioecho delay=50000000 intensity=0.6 ! autoaudiosink
```

このコマンドでは、`audiotestsrc`が生成した音声データが`audioconvert`で処理され、その後`audioecho`フィルターでエコーが追加されます。最終的に、`autoaudiosink`で音声が再生されます。

## mux（マルチプレクサ）

`mux`要素は、複数のストリーム（例えば音声と映像）を1つのストリームにまとめる役割を持ちます。たとえば、音声と映像を組み合わせて1つの動画ファイルとして保存する際に使用されます。

```shell
gst-launch-1.0 videomixer name=mix sink_1::xpos=0 sink_1::ypos=0
 ! autovideosink videotestsrc ! videoscale ! video/x-raw,width=100,height=100 ! mix.sink_1
```

## demux（デマルチプレクサ）

`demux`要素は、`mux`要素とは逆に、1つのストリームを複数のストリームに分離する役割を持ちます。たとえば、動画ファイルを読み込んで、音声と映像を個別に処理する際に使用されます。

以下は、GStreamerを使用して音声付きの動画ファイルを再生するサンプルコマンドです。このコマンドでは、`demux`要素を使用して音声と映像を分離し、それぞれを再生します。

```shell
gst-launch-1.0 filesrc location=sample.mp4 ! qtdemux name=demux \
  demux.video_0 ! queue ! decodebin ! autovideosink \
  demux.audio_0 ! queue ! decodebin ! audioconvert ! autoaudiosink
```

- `filesrc`：動画ファイル（`sample.mp4`）を読み込む
- `qtdemux`：動画ファイルを解析し、音声と映像のストリームを分離
- `queue`：映像と音声の処理を並列に行うために使用
- `decodebin`：音声や映像をデコードして再生可能な形式に変換
- `autovideosink`：映像を画面に表示
- `audioconvert`：音声データを適切な形式に変換
- `autoaudiosink`：音声をスピーカーから再生

このパイプラインを使用することで、動画ファイルの音声と映像を同期して再生できます。

# Pad（パッド）

各要素のデータの入口/出口をシンクパッド/ソースパッドと言います。データの入力元の要素に対して自身の要素はデータを消費するシンクなので入り口はシンクパッド、出力先の要素に対してはデータを生成するソースなのでソースパッドという命名なのでしょう。

![pad](/images/rikson_gstreamer-for-rustacean_filter.drawio.png)

ソース要素とシンク要素は末端の要素なのでパッドが1つしかありません。

![edge](/images/rikson_gstreamer-for-rustacean_edge.drawio.png)

一方、デマルチプレクサ（demux）やマルチプレクサ（mux）などの要素は、複数のストリームを扱うため、複数のパッドを持つことができます。
`mux`は複数のストリームを1つのストリームにまとめるため、複数のシンクパッドを持ちます。

![mux](/images/rikson_gstreamer-for-rustacean_mux.drawio.png)

これに対し、`demux`は1つのストリームから複数の映像や音声ストリームを分離するため、複数のソースパッドを持ちます。

![demux](/images/rikson_gstreamer-for-rustacean_demux.drawio.png)

# Capabilities

`Capabilities`は、GStreamerの各要素が処理できるメディアデータのフォーマットや属性を定義したものです。パイプライン内で異なる要素を接続する際、互いの`Pad`がどのようなデータを受け渡しできるかを判断するために重要です。互換性がある場合にのみ、要素同士が正常に接続され、データが正しく処理されます。

`gst-inspect-1.0`はGStreamerの要素の仕様を調べることができるコマンドです。`audioecho`の仕様を調べてみましょう。

```shell
gst-inspect-1.0 audioecho
```

`Pad Templates`セクションには、要素が持つパッドの情報が記載されています。

```
Pad Templates:
  SINK template: 'sink'
    Availability: Always
    Capabilities:
      audio/x-raw
                 format: { (string)F32LE, (string)F64LE }
                   rate: [ 1, 2147483647 ]
               channels: [ 1, 2147483647 ]
                 layout: interleaved

  SRC template: 'src'
    Availability: Always
    Capabilities:
      audio/x-raw
                 format: { (string)F32LE, (string)F64LE }
                   rate: [ 1, 2147483647 ]
               channels: [ 1, 2147483647 ]
                 layout: interleaved

Element has no clocking capabilities.
Element has no URI handling capabilities.
```

- Availability: PadにはAlways Pads、Sometimes Pads、Request Padsという種類がある
- Capabilities:
  - `audio/x-raw`: パッドが生データのオーディオを扱うことを示す
  - `format`: オーディオデータのフォーマットで、`F32LE`（32-bit Little Endian Float）と`F64LE`（64-bit Little Endian Float）を許容
  - `rate`: サンプルレートで、範囲 `[ 1, 2147483647 ]` まで対応可能
  - `channels`: チャンネル数で、範囲 `[ 1, 2147483647 ]` まで対応可能
  - `layout`: データの配置方法を示す

もしもCapabilitiesに一致しないデータが入力されると次のようにエラーになります。

```shell
$ gst-launch-1.0 fdsrc ! audioecho
Setting pipeline to PAUSED ...
Pipeline is PREROLLED ...
Setting pipeline to PLAYING ...
New clock: GstSystemClock
foo
ERROR: from element /GstPipeline:pipeline0/GstFdSrc:fdsrc0: Internal data stream error.
Additional debug info:
../libs/gst/base/gstbasesrc.c(3127): gst_base_src_loop (): /GstPipeline:pipeline0/GstFdSrc:fdsrc0:
streaming stopped, reason not-negotiated (-4)
Execution ended after 0:00:04.009247573
Setting pipeline to NULL ...
Freeing pipeline ...
```

# RustでGStreamerを使用する

GStreamerのRustバインディングであるgstreamer-rsというcrateを利用してRustから呼び出すことができます。

## プロジェクトの作成

まず、Rustの環境を設定する必要があります。RustのパッケージマネージャーであるCargoを使用して、GStreamer関連のクレートを追加します。以下のコマンドでプロジェクトを作成し、`gstreamer`クレートを追加します。

```shell
cargo new myapp
cd myapp
cargo add gstreamer
```

## 音声再生の実装

Rustで音声を再生するGStreamerパイプラインを実装してみましょう。`src/main.rs` を次の通りに編集してください。

```rust
use gstreamer::prelude::*;

pub fn main() {
    // Initialize GStreamer
    gstreamer::init().unwrap();

    // Create the elements
    let source = gstreamer::ElementFactory::make("audiotestsrc")
        .name("source")
        .build()
        .expect("Could not create parser element");
    let sink = gstreamer::ElementFactory::make("autoaudiosink")
        .name("sink")
        .build()
        .expect("Could not create sink element");

    // Create the empty pipeline
    let pipeline = gstreamer::Pipeline::with_name("test-pipeline");

    // Build the pipeline
    pipeline.add_many([&source, &sink]).unwrap();
    gstreamer::Element::link_many(&[&source, &sink]).unwrap();

    // Start playing
    pipeline
        .set_state(gstreamer::State::Playing)
        .expect("Unable to set the pipeline to the `Playing` state");

    // Wait until error or EOS
    let bus = pipeline.bus().unwrap();
    for msg in bus.iter_timed(gstreamer::ClockTime::NONE) {
        use gstreamer::MessageView;

        match msg.view() {
            MessageView::Eos(..) => break,
            MessageView::Error(err) => {
                println!(
                    "Error from {:?}: {} ({:?})",
                    err.src().map(|s| s.path_string()),
                    err.error(),
                    err.debug()
                );
                break;
            }
            _ => (),
        }
    }

    // Shutdown pipeline
    pipeline
        .set_state(gstreamer::State::Null)
        .expect("Unable to set the pipeline to the `Null` state");
}
```

## 実行

GStreamerのバイナリ版の場合、次のように環境変数を設定しないとライブラリが認識されません。

```shell
export RUSTFLAGS="-C link-args=-Wl,-rpath,/Library/Frameworks/GStreamer.framework/Versions/1.0/lib"
```

`cargo run` を実行して、ポーっと音が出たらオッケーです。

# 参考

公式チュートリアル。

https://gstreamer.freedesktop.org/documentation/tutorials/index.html

公式チュートリアルをRustで書き直したものが公開されています。

https://gitlab.freedesktop.org/gstreamer/gstreamer-rs/-/tree/main/tutorials/src/bin

Rust製プラグイン作成のチュートリアル。

https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs/-/tree/main/tutorial

GStreamerの基本概念について。

https://qiita.com/alivelime/items/50d796c09baabb765625

プラグイン作成に関する日本語記事。サンプルコードが参考になりました。

https://qiita.com/uzuna/items/6c183253736e26598c45

多重化の説明が分かりやすかったです。

https://www.pixela.co.jp/products/pickup/dev/gstreamer/gst_1_overview_of_gstreamer.html
