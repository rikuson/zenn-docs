---
title: "100行で作るP2Pビデオ通話アプリケーション"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["WebRTC"]
published: true
publication_name: micin
---

皆さんはWebRTC(Web Real-Time Communication)をご存知でしょうか？
WebRTCとは、ブラウザ間でリアルタイムの音声、映像、データ通信を可能にするオープンソースの技術です。本稿ではWebRTCを利用してP2Pビデオ通話を実装していきます。

SDKやライブラリなしでビデオ通話を実装するのは難しそう、という印象を持たれるかもしれません。しかし、実は100行程度で動くものを作れるんだよ、ということを本稿で実証したいと思います。とにかく手を動かして概要を理解することに主眼を置き、プロトコルの説明は省いています。

抽象化も型もなく、ただただ素朴に最小限で実装しました。

作成したサンプルアプリケーションはGitHubで公開しています。

https://github.com/rikuson/p2p-video-call-sample

# システム構成

WEBサーバーもシグナリングサーバーもまとめてモノリシックに構成して、Herokuでホスティングします。Herokuの使い方はここでは割愛します。WEBサーバーにはExpress、シグナリングサーバーにはSocket.ioを利用します。STUNサーバーはGoogleが公開しているものを利用します。

![システム構成](/images/rikson_p2p-video-call-app-in-100-lines_system-composition.drawio.png)

必要なパッケージをインストールしておきましょう。

```shell
npm i express socket.io
```

# シグナリングサーバーの実装

シグナリングサーバーとはSDP(Peerの情報)やICE Candidate(通信経路の情報)をクライアント間で交換するためのサーバーです。本稿ではWebSocketで実装していますが、とにかく情報交換できれば何でも良いです。

`index.js` というファイルを下記のように作成します。

```javascript
import { createServer } from "http";
import { Server } from "socket.io";
import express from "express";

const app = express();
const http = createServer(app);
const io = new Server(http);

io.on("connection", (socket) =>
  // 受信したイベント全てを他のクライアントへブロードキャスト
  socket.onAny((event, data) => socket.broadcast.emit(event, data)),
);

// 環境変数からポート番号を読み込み、サーバーを起動
http.listen(Number(process.env.PORT) || 3000);
```

たったのこれだけで出来ちゃいます！

本来は真面目にイベントごとにハンドリングするのが良いと思いますが、ここでは単純化しています。

# ビューの実装

ビデオ通話するには画面が必要ですよね。ビューを追加しましょう。

```diff javascript
  import { createServer } from "http";
  import { Server } from "socket.io";
  import express from "express";

  const app = express();
  const http = createServer(app);
  const io = new Server(http);
+
+ // 静的ファイルを配置するディレクトリを指定
+ app.use(express.static("public"));

  io.on("connection", (socket) =>
    socket.onAny((event, data) => socket.broadcast.emit(event, data)),
  );

  http.listen(Number(process.env.PORT) || 3000);
```

続いて、次のように `public/index.html` を作成します。

```pug
<html>
  <head>
    <!-- 静的ファイルを配置するディレクトリを指定 -->
    <script type="module" src="index.js"></script>
  </head>
  <body style="display: grid; grid-template-columns: 1fr 1fr;">
    <!-- `onClickBtn` はこれから定義 -->
    <button style="position: absolute; z-index: 1;" onClick="onClickBtn()">Join</button>
  </body>
</html>
```

# フロントエンドの実装

`public/index.js` を次のように作成します。SDPの手続きはオファー・アンサーモデルと呼ばれています。

この例ではボタンを押したらオファーを送信します。先に入室したユーザーのオファーは受信されずに捨てられます。後から入室したユーザーがオファー側になり、先に入室したユーザーがアンサー側になります。

```javascript
// Socket.ioのフロントエンド・モジュールはバックエンドからインポート可能
import "/socket.io/socket.io.js";

const pc = new RTCPeerConnection({
  // GoogleのSTUNサーバーを指定
  iceServers: [{ urls: ["stun:stun.l.google.com:19302"] }],
});
const socket = io();

globalThis.onClickBtn = async () => {
  // 端末のカメラとマイクのアクセスをリクエスト
  const stream = await navigator.mediaDevices.getUserMedia({
    audio: true,
    video: true,
  });
  for (const track of stream.getTracks()) {
    pc.addTrack(track);
  }
  const video = document.createElement("video");

  // ブラウザのポリシーによる映像再生エラーの回避
  video.playsInline = true;
  video.muted = true;

  video.style.width = "100%";
  // MediaStreamをvideoタグにアタッチ
  video.srcObject = stream;
  video.play();
  document.body.appendChild(video);

  // LocalDescriptionを生成
  pc.createOffer().then((desc) => {
    // LocalDescriptionをPeerConnectionにセット
    pc.setLocalDescription(desc);
    // LocalDescriptionをリモートユーザーへ送信
    socket.emit("offer", desc);
  });
};

// リモートユーザーがPeerConnectionにMediaStreamTrackを追加したら発火
pc.addEventListener("track", ({ track }) => {
  if (track.kind === "video") {
    const video = document.createElement("video");
    video.playsInline = true;
    video.muted = true;
    video.style.width = "100%";
    video.srcObject = new MediaStream([track]);
    video.play();
    document.body.appendChild(video);
  }
  if (track.kind === "audio") {
    const audio = document.createElement("audio");
    audio.srcObject = new MediaStream([track]);
    audio.play();
  }
});
// RTCPeerConnection.setLocalDescription()の呼び出しに応じて、
// ICE Candidateが見つかった時や収集が終了した際に発火
pc.addEventListener("icecandidate", ({ candidate }) => {
  if (candidate) {
    // ICE Candidateをリモートユーザーへ送信
    socket.emit("ice", candidate);
  }
});

socket
  // リモートユーザーのofferイベントの受信
  .on("offer", (desc) => {
    // RemoteDescriptionをPeerConnectionにセット
    pc.setRemoteDescription(desc);
    // LocalDescriptionを生成
    pc.createAnswer().then((desc) => {
      // LocalDescriptionをPeerConnectionにセット
      pc.setLocalDescription(desc);
      // LocalDescriptionをリモートユーザーへ送信
      socket.emit("answer", desc);
    });
  })
  // リモートユーザーのanswerイベントを受信し、RemoteDescriptionをPeerConnectionにセット
  .on("answer", (desc) => pc.setRemoteDescription(desc))
  // リモートユーザーのiceイベントを受信し、ICE Candidateを追加
  .on("ice", (candidate) => pc.addIceCandidate(candidate));
```

# ローカルで動作確認

`package.json` に `start` コマンドを追加しましょう。

```json
...
  "scripts": {
    "start": "node index.js",
...
```

`npm start` を実行し、 http://localhost:3000 を2つ開いて、Joinボタンを押してみて下さい。同じ端末で複数開くとハウリングするので注意！

![ローカルで動作確認](/images/rikson_p2p-video-call-app-in-100-lines_local-check.gif)

映像と音声が疎通したら成功です。

# おわりに

あとは、作成したアプリケーションをHerokuにデプロイすれば異なるネットワークからP2P接続できることが確認できます。

このサンプルでは3人以上入室できません。3人以上入室可能にするには、RTCPeerConnectionをリモートユーザーごとに作成する必要があり、管理が複雑になってきます。また、P2Pで人数が増えてくると送受信の効率が悪くなるため、SFUやMCUなどのトポロジーを検討する必要が出てきます。

今回はとにかく最小限のコードでサンプルアプリケーションを動かすことが目的のため、多人数での通信は取り扱いませんでした。興味があれば調べてみて下さい。
