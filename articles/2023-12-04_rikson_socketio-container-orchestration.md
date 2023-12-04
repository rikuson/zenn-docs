---
title: "Socket.ioコンテナオーケストレーションハンズオン"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "socketio"]
published: true
publication_name: micin
---

この記事は [MICIN Advent Calendar 2023](https://adventar.org/calendars/9595) の4日目の記事です。

https://adventar.org/calendars/9595

前回はarapowerさんの、[「手軽に稼働環境を増減できるデプロイの仕組み」](https://zenn.dev/micin/articles/49c8a685828e97) でした。

---

プラットフォームチーム所属の竹内です。主にビデオ通話基盤の開発を行なっています。
本稿ではSocket.ioの最新機能を紹介しつつ、Kubernetesを利用したスケーリングやデプロイメントのポイントを紹介します。実際に手を動かしながら課題に直面することを体験してもらえるようにハンズオン形式にしました。

Socket.ioのアプリケーションの実装は簡単なのですが、サーバーをスケールさせたりシームレスなデプロイを実現しようとすると、難易度が上がります。自前運用をするにはそれなりの覚悟が必要です。今ではSocket.ioからSaaSに乗り換えてしまいたいと考えているのですが、戦いの記録をここに書き残しておこうと思います。

例として、ローカル環境にKubernetesクラスターを構築し、次のようなリアルタイムに同期するホワイトボードアプリケーションを作成します。

![動作例](/images/rikson_socketio-container-orchestration_scaling.gif)

今回作成した成果物は [GitHub](https://github.com/rikuson/socket.io-k8s-example) で公開しています。

# 技術選定

主なシステムコンポーネントと利用したバージョンについて記載します。

## Socket.io@4.7.2

モダンブラウザでWebSocketサポートが十分に行き届いている現在、引き続きSocket.ioを利用するメリットとしては次のようなものが挙げられます。

- 再接続制御
- レガシー環境向けのフォールバック
- WebTransportの先取り

### 再接続制御

Socket.ioを利用すれば、再接続時のメッセージの再送やステートの復旧処理などを実装する手間が省けます。今年の2月にリリースされた[Connection State Recovery](https://socket.io/docs/v4/connection-state-recovery)を利用したステートの復旧について、本稿で実際に動作を確認していきます。

### レガシー環境向けのフォールバック

引き続きWebSocketをブロックしている環境等でHTTPロングポーリングを利用する場合は、Socket.ioを利用することで実装コストを下げることができるでしょう。ただし、HTTPロングポーリング方式でスケーリングさせる場合、スティッキーセッションを使う必要があるので注意が必要です。

### WebTransportの先取り

WebTransportはUDPベースの次世代のWebSocketと言われているリアルタイムメッセージングプロトコルです。Socket.ioでは今年の6月にWebTransportの対応バージョンがリリースされました。
WebTransport対応環境ではWebSocketよりも更に高速なリアルタイム通信が可能となります。

https://socket.io/get-started/webtransport

本稿では詳しく触れませんが、興味のある方は調べてみてください。

## Docker Desktop for Mac@4.12.0

弊社のビデオ通話基盤ではオーケストレーションシステムとしてKubernetesではなく、ECSを利用しています。ローカル環境での利用しやすさから、本稿ではKubernetesを採用しました。

ローカルでKubenetes環境を構築するにはminikubeを利用する方法もありますが、個人的にはDocker DesktopビルトインのKubernetes機能を利用する方が好みです。
有効にしていない方は、まずはKubernetes機能を有効にしてください。

## Helm@3.13.2

HelmはKubernetesのパッケージマネージャーです。一から直接マニフェストファイルを書くより、少ない手順で動作サンプルを作成することが出来るため、採用しました。
初学者にはおまじないを唱えている気分になると思いますが、素早くサンプルを動かせることを優先させました。

また、Helmを利用することでKubernetesオブジェクトをひとまとまりで管理できます。
初学者にとってローカル環境によく分からないゴミが残るのはストレスになります。
Helmを利用することで各オブジェクトについてクラスターの状態を気にすることなく、作って壊す試行錯誤がコマンド1つで行うことが出来ます。

## Redis@18

Socket.ioをスケーリングするにはAdapterと呼ばれるミドルウェアを利用する方法が手軽です。
公式に提供されているAdapterは次のようなものがあります。

- [Redis adapter](https://socket.io/docs/v4/redis-adapter/)
- [Redis Streams adapter](https://socket.io/docs/v4/redis-streams-adapter/)
- [MongoDB adapter](https://socket.io/docs/v4/mongo-adapter/)
- [Postgres adapter](https://socket.io/docs/v4/postgres-adapter/)
- [Cluster adapter](https://socket.io/docs/v4/cluster-adapter/)

今回は一時的なデータのみを扱いSQLなどは不要であること、インメモリーで高速に動作することからRedisを選定しました。RedisのAdapterは2種類ありますが、「Redis adapter」は従来のもので、「Redis Streams adapter」は今年の4月にリリースされた新しいものです。2つのAdapterの違いとしては次のように説明されています。

> The main difference with the existing Redis adapter (which use the [Redis Pub/Sub mechanism](https://redis.io/docs/manual/pubsub/)) is that this adapter will properly handle any temporary disconnection to the Redis server and resume the stream without losing any packets.

後述するConnection State Recoveryについても「Redis Streams adapter」のみでサポートされています。本稿では後者の「Redis Steams adapter」採用しました。

# Socket.ioサンプルアプリケーションの動作確認

まずは、Socket.ioが公開しているホワイトボードアプリケーションを構築し、動作を確認します。ソースコードを取得しましょう。

```shell
git clone --depth 1 https://github.com/socketio/socket.io.git
```

尚、本稿執筆時点のリビジョン番号は [8c9ebc30e5452ff9381af5d79f547394fa55633c](https://github.com/socketio/socket.io/tree/8c9ebc30e5452ff9381af5d79f547394fa55633c) です。リポジトリの状態が変わって上手くいかない場合は戻してみてください。

パッケージをインストールし、サーバーを起動します。

```shell
cd socket.io/examples/whiteboard
npm i && npm start
...
listening on port 3000
```

[http://localhost:3000](http://localhost:3000) を開き、動作を確認してみましょう。2画面で同期することが確認できます。

![動作サンプル](/images/rikson_socketio-container-orchestration_integration.gif)

動作が確認できたらCtrl-cでプロセスを終了させます。

# フロントエンドとバックエンドの分離

WebサーバーとSocket.ioサーバーにかかる負荷の大きさはそれぞれ異なります。効率的にスケーリングするためにはサーバーを分離する必要があります。今回はWebサーバー側のスケーリングは考えません。

```shell
mkdir frontend backend
mv public frontend
mv package.json package-lock.json index.js node_modules backend
```

 `backend/Dockerfile` を追加します。

```docker
FROM node:20.9-slim

COPY . /app
WORKDIR /app
RUN npm ci
ENTRYPOINT ["npm", "start"]
```

`node_modules` はコンテナにコピーしたくないので `backend/.dockerignore` を追加します。

```
node_modules/
```

`docker-compose.yaml` を追加します。frontendはnginxでホスティングし、静的ファイルをドキュメントルートにマウントします。

```yaml
version: '3.5'

services:
  frontend:
    image: nginx:1.25.3
    ports:
      - 80:80
    expose:
      - 80
    volumes:
      - ./frontend/public:/usr/share/nginx/html
  backend:
    build: ./backend
    init: true
    entrypoint:
      - npm
      - start
    ports:
      - 3000:3000
    expose:
      - 3000
```

`frontend/public/main.js` を編集します。プロトコルはWebsocketのみ対応させます。

```diff:javascript
-var socket = io();
+var socket = io('http://localhost:3000', { transports: ['websocket'] });
```

`frontend/public/index.html` を編集します。

```diff:html
-<script src="/socket.io/socket.io.js"></script>
+<script src="http://localhost:3000/socket.io/socket.io.js"></script>
```

`backend/index.js` を編集します。

```diff:javascript
-app.use(express.static(__dirname + '/public'))
```

コンテナを起動し、[http://localhost](http://localhost) を開いて動作を確認しましょう。

```shell
docker compose up
```

動作が確認できたらCtrl-cでコンテナを停止させます。

# Socket.ioサーバーのクラスター化

まずは、ディレクトリとサンプルチャートを作成します。

```shell
helm create infra
```

`infra/values.yaml` を編集します。backendのイメージに置き換え、 `pullPolicy` を `Never` にすることで、ローカルイメージの使用を強制します。ServiceタイプをClusterIPからLoadBalancerへ変更し、ポート番号を変更します。

```diff:yaml
 image:
-  repository: nginx
-  pullPolicy: IfNotPresent
+  repository: whiteboard-backend
+  pullPolicy: Never
   # Overrides the image tag whose default is the chart appVersion.
-  tag: ""
+  tag: "latest"
 ...
 service:
-  type: ClusterIP
-  port: 80
+  type: LoadBalancer
+  port: 3000
```

`backend/index.js` を編集します。コンテナのヘルスチェック用のエンドポイントを追加し、デバッグ用のログメッセージを追加。

```diff:javascript
+app.get('/', (req, res) => res.send('ok'));

 function onConnection(socket){
+  console.log('Socket is connected', {
+    id: socket.id,
+  });
   socket.on('drawing', (data) => socket.broadcast.emit('drawing', data));
 }
```

Dockerイメージをビルドします。

```shell
docker compose build --no-cache
```

チャートをインストールします。

```shell
helm upgrade --install example infra
```

Webサーバーを起動し、再度 [http://localhost](http://localhost) を開いて動作を確認しましょう。

```shell
docker compose up frontend
```

これで、Kubernetes上でSocket.ioを動作させることができました。意外に簡単ですよね！
動作が確認できたらCtrl-cでコンテナを停止します。

# スケーリング

`infra/values.yaml` を編集しオートスケールを有効化します。最小レプリカ数は2にしておきましょう。

```diff:yaml
 autoscaling:
-  enabled: false
-  minReplicas: 1
+  enabled: true
+  minReplicas: 2
```

チャートを更新します。

```shell
helm upgrade --install example infra
```

Podが2つ作成されていることを確認しましょう。（反映されるまでしばらくかかります。）

```shell
kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
example-infra-54cf745cc4-9xrm7   1/1     Running   0          6m36s
example-infra-54cf745cc4-zqrbw   1/1     Running   0          6m38s
```

次のように `kubectl logs` コマンドにPod名を渡して `f` オプションを付けると、リアルタイムなログ監視ができます。
クライアントがPodに接続されると、「Socket is connected」というメッセージが表示されます。

```shell
kubectl logs -f example-infra-54cf745cc4-9xrm7
```

再度、Webサーバーを起動し、http://localhost を開いてください。

```
docker compose up frontend
```

2画面開いて、2つのPodのデバッグメッセージを確認しながら何度かタブを更新して、下図のようにそれぞれ異なるPodに接続されることを確認してください。

![1対1接続](/images/rikson_socketio-container-orchestration_parallel-connection.drawio.png)

動作を確認しましょう。

![同期されない例](/images/rikson_socketio-container-orchestration_scale-out.gif)

描画が同期されていませんね。異なるSocket.ioサーバー間のメッセージをブロードキャストするには下図のように [Adapter](https://socket.io/docs/v4/adapter/) を利用する必要があります。

![ブロードキャスト](/images/rikson_socketio-container-orchestration_broadcast.drawio.png)

Ctrl-cでコンテナを停止したら、一旦チャートをアンインストールし、Kubernetes環境を削除しておきます。

```shell
helm uninstall example
```

[Redis Streams Adapter](https://socket.io/docs/v4/redis-streams-adapter/) をインストールしましょう。

```shell
(cd backend && npm i @socket.io/redis-streams-adapter redis)
```

`backend/index.js` を編集し、インストールしたパッケージを呼び出します。環境変数からRedisの接続情報を受け取り、接続確立後にSocket.ioのServerクラスインスタンスを作成し、Adapterを渡します。

```diff:javascript
 const express = require('express');
 const app = express();
 const http = require('http').Server(app);
-const io = require('socket.io')(http);
+const { Server } = require('socket.io');
+const { createAdapter } = require('@socket.io/redis-streams-adapter');
+const { createClient } = require('redis');
 const port = process.env.PORT || 3000;
+const redis = createClient({
+  url: process.env.REDIS_URL,
+  password: process.env.REDIS_PASSWORD,
+});

 app.get('/', (req, res) => res.send('ok'));

 function onConnection(socket){
   console.log('Socket is connected', {
     id: socket.id,
   });
   socket.on('drawing', (data) => socket.broadcast.emit('drawing', data));
 }

-io.on('connection', onConnection);
+redis.connect().then(() => {
+  const io = new Server(http, {
+    adapter: createAdapter(redis),
+  });
+  io.on('connection', onConnection);
+});
```

`backend/Dockerfile` にRedisの接続情報に関する環境変数を定義しましょう。

```diff:Dockerfile
 FROM node:20.9-slim

+ENV REDIS_URL "redis://adapter:6379"
+ENV REDIS_PASSWORD ""

 COPY . /app
 WORKDIR /app
 RUN npm ci
 ENTRYPOINT ["npm", "start"]
```

`docker-compose.yaml` にRedisを追加します。

```diff:yaml
 version: '3.5'

 services:
   frontend:
     image: nginx:1.25.3
     ports:
       - 3000:80
     expose:
       - 3000
     volumes:
       - ./frontend/public:/usr/share/nginx/html
   backend:
     build: ./backend
     ports:
       - 80
     expose:
       - 80
+    depends_on:
+      - adapter
+  adapter:
+    image: redis:7.0.4
+    ports:
+      - 6379:6379
+    expose:
+      - 6379
```

ビルドして、コンテナを起動し、 http://localhost を開いて動作を確認しましょう。

```shell
docker compose build --no-cache
docker compose up
```

特にエラーなどが出なければ、Ctrl-cでコンテナを停止してください。

次にHelmチャートの設定を追加していきます。
`infra/Chart.yaml` を開いて、SubchartとしてRedisを追加します。

```diff:yaml
+dependencies:
+  - name: redis
+    repository: https://charts.bitnami.com/bitnami
+    version: 18.x.x
```

`infra/values.yaml` に環境変数等を追加します。

```diff:yaml
+global:
+  redis:
+    password: password
+redis:
+  replica:
+    replicaCount: 1
```

`infra/template/deployment.yaml` を編集して、環境変数をコンテナに渡します。

```diff:yaml
       containers:
         - name: {{ .Chart.Name }}
           securityContext:
             {{- toYaml .Values.securityContext | nindent 12 }}
           image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
           imagePullPolicy: {{ .Values.image.pullPolicy }}
+          env:
+            - name: REDIS_URL
+              value: redis://{{ .Release.Name }}-redis-master:6379
+            - name: REDIS_PASSWORD
+              value: {{ .Values.global.redis.password }}
           ports:
             - name: http
               containerPort: {{ .Values.service.port }}
               protocol: TCP
```

Subchartをインストールし、Chartを再インストールし、コンテナを起動して [http://localhost](http://localhost) を開いて動作確認してみましょう。

```shell
helm dependency build infra
helm upgrade --install example infra
docker compose up frontend
```

![スケーリング](/images/rikson_socketio-container-orchestration_scaling.gif)

異なるPodに接続されていても描画が同期されていますね。

# デプロイメント

WebSocketなどのリアルタイム通信するアプリケーションをステートを維持したままダウンタイムなくデプロイするにはいくつか工夫が必要になります。Socket.ioの再送制御やConnection State Recovery機能を利用することで、これらの問題を解決できます。

まずは、Socketごとにバックエンドでステートを保持するようにアプリケーションを書き換えます。
`frontend/public/main.js` を編集します。描画時には色情報を送信せず、色を選択した時のみ色情報を送る仕様に変更します。

```diff:javascript
 ...
     socket.emit('drawing', {
       x0: x0 / w,
       y0: y0 / h,
       x1: x1 / w,
       y1: y1 / h,
-      color: color
     });
 ...
   function onColorUpdate(e){
     current.color = e.target.className.split(' ')[1];
+    socket.emit('colorUpdate', { color: current.color });
   }
```

`backend/index.js` を編集します。デバッグメッセージを追加します。 `init` というメッセージを送っているのはSocket.ioのバグっぽい振る舞い[^1]に対するワークアラウンドです。初期ステートとして黒色をセットします。また、 `colorUpdate` メッセージを受信したら、ステートを更新します。Connection State Recoveryを有効化し、切断されても2分間はステートを保持するように設定を追加します。 `skipMiddlewares` をtrueにして、再接続時には初期化処理が実行されないようにします。 `drawing` メッセージでステートから取得した色情報を送信します。

```diff:javascript
 function onConnection(socket){
   console.log('Socket is connected', {
     id: socket.id,
+    data: socket.data,
+    recovered: socket.recovered,
   });
+  socket.emit('init'); // FIXME: Workaround for single user connection state recovery
-  socket.on('drawing', (data) => socket.broadcast.emit('drawing', data));
+  socket.on('drawing', (data) => socket.broadcast.emit('drawing', { ...data, color: socket.data.color }));
+  socket.on('colorUpdate', (data) => socket.data.color = data.color);
 }

 redis.connect().then(() => {
   const io = new Server(http, {
     adapter: createAdapter(redis),
+    connectionStateRecovery: {
+      maxDisconnectionDuration: 2 * 60 * 1000,
+      skipMiddlewares: true,
+    },
   });
+  io.use((socket, next) => {
+    socket.data.color = 'black';
+    next();
+  });
   io.on('connection', onConnection);
 });
```

ビルドして、デプロイし、コンテナを起動して [http://localhost](http://localhost) を開いて動作を確認しましょう。 `kubectl rollout restart deploy <Deployment名>` で現在のPodがロールアウトされて新しく入れ替わります。フロントエンドのキャッシュが残っている場合は削除してから確認してください。

```shell
docker compose build --no-cache
kubectl rollout restart deploy example-infra
docker compose up frontend
```

動画のように描画した後にローリングアップデートして、再描画をしてみましょう。

```shell
kubectl rollout restart deploy example-infra
```

![ステートロス](/images/rikson_socketio-container-orchestration_stateloss.gif)

Podが入れ替わった後に色が青色から黒色に変化しています。これはステートが保持されずに初期ステートへ戻ってしまっているためです。

これにはいくつかの原因がありますが、まずConnection State Recoveryを正常に動作させるにはGraceful Shutdownを行う必要があります。 `backend/index.js` を編集し、SIGTERMを受信したらSocket.ioの終了処理を行いましょう。

```diff:javascript
   io.on('connection', onConnection);
+  process.on('SIGTERM', () => {
+    console.log('Shutting down')
+    io.close();
+  });
```

また、 `backend/Dockerfile` のENTRYPOINTを見ると、npmコマンド経由でアプリケーションを実行しています。npmコマンド経由でアプリケーションを実行すると、コンテナへSIGTERMを送ってもnpmが握りつぶしてしまい、SIGTERMがアプリケーションへ届きません。

`backend/Dockerfile` を編集し、nodeコマンドを直接呼び出すように書き換えます。

```diff:Dockerfile
-ENTRYPOINT ["npm", "start"]
+ENTRYPOINT ["node", "index"]
```

これでアプリケーションのGraceful Shutdownが可能になるのですが、「PID1問題」と呼ばれるよく知られた問題が残っています。PID1のプロセスは全プロセスの起点となる特殊なプロセスで、SIGTERMを受け付けないことがあります。Node.jsはPID1で利用されることを想定されていません。しかし、ENTRYPOINTに指定されたコマンドはPID1で実行されてしまいます。

この問題の解決方法としては、 [tini](https://github.com/krallin/tini) 等のinitを利用してシグナルをプロキシする方法や [Share Process Namespace](https://kubernetes.io/ja/docs/tasks/configure-pod-container/share-process-namespace/) を利用してプロセスIDの名前空間を共有する方法があります。ここでは前者の tini を利用する方法を紹介します。

`backend/Dockerfile` を編集します。

```diff:Dockerfile
 FROM node:20.9-slim

+ENV TINI_VERSION v0.19.0
+ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
+RUN chmod +x /tini
+ENTRYPOINT ["/tini", "--"]

 ENV REDIS_URL "redis://adapter:6379"
 ENV REDIS_PASSWORD ""

 COPY . /app
 WORKDIR /app
 RUN npm ci
-ENTRYPOINT ["node", "index"]
+CMD ["node", "index"]
```

ビルド、デプロイします。

```shell
docker compose build --no-cache
kubectl rollout restart deploy example-infra
```

:::message

M1 Macでtiniを使うと次のエラーが出力されました。

```
qemu-x86_64: Could not open '/lib64/ld-linux-x86-64.so.2': No such file or directory
```

次のように `docker-compose.yaml` にplatformを指定することで、解消できました。

```diff:yaml
   backend:
     build: ./backend
+    platform: linux/amd64
```

再度、上記コマンドでビルド、デプロイしてください。

:::

もう一度、確認してみます。

色が維持されたまま、無事ローリングアップデートが完了するかと思います。
これでステートを維持したままダウンタイムのないデプロイが可能となります。

[^1]: Redisからステートを検索する際のオフセットをクライアント側で保持しています。メッセージの受信時に最新のエントリのIDをオフセットとして受け取る仕組みとなっているため、受信メッセージが何もないとオフセットの値が空になります。すると、ステート復旧処理がトリガーされません。[issue](https://github.com/socketio/socket.io/issues/4888) を作成しておきました。

# 後片付け

作成したHelmチャートとDockerイメージを削除して、後片付けを行います。

```shell
helm uninstall example
docker compose down --rmi all --volumes --remove-orphans
```

# おわりに

ステートフルなアプリケーションのツラみや面白みが少しでも伝わったでしょうか？
ここでは取り上げませんでしたが、他にも次のように考慮すべき問題が残っています。

- デプロイ時のクライアントの同時再接続によるサーバー高負荷問題
- ファイルディスクリプタの上限
- オートスケールに利用するメトリクスの検討
- WebSocket以外のプロトコルの対応

本格的に自前運用を始める前に一度立ち止まってSaaSの利用などの代替手段がないか検討すべきです。それでも必要なら、自分のスキルを磨くチャンスだと思って挑戦しましょう。

---

MICINではメンバーを大募集しています。
「とりあえず話を聞いてみたい」でも大歓迎ですので、お気軽にご応募ください！

https://recruit.micin.jp/
