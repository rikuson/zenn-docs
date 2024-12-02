---
title: "エラー制御のベストプラクティスを考える"
emoji: "⚠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
publication_name: micin
---

この記事は [MICIN Advent Calendar 2024](https://adventar.org/calendars/10022) の 2日目の記事です。

https://adventar.org/calendars/10022

前回はabekohさんの、「[MICIN Advent Calendar 2024、始まります！](https://note.micin.jp/n/n93a9cb28af4e)」 でした。

---

弊社では、定期的に社内読書会を開催しています。最近、その読書会で「[Good Code, Bad Code](https://amzn.to/3CRI5HY)」について扱い、議論を重ねてきました。この本を通じて、エラー制御についてさらに深く考えるきっかけを得ました。TypeScriptにおけるエラー制御についてもう一歩踏み込んだ内容を記事としてまとめてみたいと思い、執筆テーマとして選びました。

# エラーの表現方法

通常、エラーが発生した際には下位のコードから上位のコードへエラーを通知する必要があります。いくつかのやり方を紹介しますが、明示的な方法と暗黙的な方法に分類できます。明示的な方法を選択すると、コードの呼び出し元は静的解析によってエラーを無視できず、ハンドリングを強制されます。暗黙的な方法を選択すると、エラーは通知されますが、呼び出し元がそれに気づかないことがあります。基本的には明示的な方が望ましいと考えています。

## マジックナンバーを返す

プログラム中に何らかの識別子として直接書かれた数値は、**マジックナンバー**と呼ばれます。一部のレガシーコードでは、エラーの表現としてマジックナンバーが利用されることがあります。

例えば、JavaScriptの配列の標準メソッド `findIndex` は要素が見つからない場合、 `-1` を返します。

```typescript
const ary = ["foo", "bar"];
const i = ary.findIndex((a) => a === "baz"); // -1
console.log(ary[i]); // undefined
```

マジックナンバーからは実装者の意図を読み取ることが困難な上、型検査が活かせないことがあり、可能な限り避けるべきです。返すべき値がない場合はundefinedやnullを返すか、Errorを投げるなどの代替案を検討しましょう。

## nullまたはundefinedを返す

例えば、次のようにErrorオブジェクトからユーザーフレンドリーなメッセージを表示するコードがあります。些末なエラーでユーザーがすべきことがない場合、 `getHint` 関数は空文字を返します。

```typescript
const getHint = (error: Error) => {
  if (error instanceof NetworkError) {
    return "通信要件をご確認ください。";
  }
  if (error instanceof TimeoutError) {
    return "タイムアウトしました。ページをリロードしてください。";
  }
  return "";
};

const showHint = (hint: string) => alert(hint);

try {
  doSomething();
} catch (e) {
  showHint(getHint(e));
}
```

上記の実装では `getHint` から空文字が返ってきた場合、無言のメッセージが `alert` で表示されてしまいます。次のようにnullを返すことで、型エラーとなり条件分岐の実装し忘れを未然に防ぐことができます。もしくは、戻り値を指定しない（undefinedを返す）returnでも良いです。

```diff typescript
  const getHint = (error: Error) => {
    if (error instanceof NetworkError) {
      return "通信要件をご確認ください。";
    }
    if (error instanceof TimeoutError) {
      return "タイムアウトしました。ページをリロードしてください。";
    }
-   return "";
+   return null;
  };

  const showHint = (hint: string) => alert(hint);

  try {
    doSomething();
  } catch (e) {
    showHint(getHint(e)); // Argument of type 'string | null' is not assignable to parameter of type 'string'.
  }
```

## Errorオブジェクトを投げる

ここまで見てきたように、エラーを表現するために、nullやundefinedをreturnすることが有効な場合があります。しかし、nullやundefinedではエラーの理由などの詳細情報を表現できません。

より詳細な情報が必要な場合はErrorクラスと `try-catch` 文を使いましょう。次の割り勘プログラムは人数に0が入力されるとゼロ除算を検知してエラーメッセージを出力します。

```typescript
const divide = (num1: number, num2: number) => {
  if (num2 === 0) {
    throw new Error("人数は1以上の値を指定してください。");
  }
  return num1 / num2;
};

const splitBillPerPerson = (price: number, count: number) => {
  const perPerson = divide(price, count);
  return perPerson;
};

try {
  alert("一人当たり" + splitBillPerPerson(1000, 0) + "円です。");
} catch (e) {
  alert(e.message);
}
```

このようにErrorクラスを用いることでエラーの情報を付け足すことができました。

しかし、上記の例はTypeScriptにおいて標準的なエラーハンドリングの手法ですが、問題を抱えています。TypeScriptではcatchされた変数の型はデフォルトで any 型になってしまうため、型安全にエラーを扱うことができません。

`divide` は第二引数が0のときエラーを投げますが、静的解析でその情報を知ることができません。`splitBillPerPerson` を呼び出すと `divide` から生じたエラーは大域脱出して現れます。`try-catch` を使わずに `splitBillPerPerson` を呼び出してもエラーにはならないため、ハンドリングの漏れが発生しやすくなります。

:::message

#### 検査例外

JavaやSwiftなどの言語には検査例外という機能が備わっており、エラーハンドリングの漏れを防止します。
TypeScriptにも提案はありますが、「Closed as not planned」となっています。

https://github.com/microsoft/TypeScript/issues/13219

:::

## カスタムエラーを定義する

前述のエラーハンドリングには型の安全性だけでなく、再利用性の観点でも問題があります。 `divide` は第一引数から第二引数を割るという汎用的な関数であるにも関わらず、エラーメッセージの内容が上位のコードのロジックと強く結びついてしまっています。`divide` に汎用性を持たせるにはエラーを抽象化する必要があります。

次のようにゼロ除算のカスタムエラーを定義し、エラーを抽象化してみました。

```typescript
class ZeroDivisionError extends Error {
  name = "ZeroDivisionError";
  constructor(num: number) {
    super(`${num} divided by 0.`);
  }
}

const divide = (num1: number, num2: number) => {
  if (num2 === 0) {
    throw new ZeroDivisionError(num1);
  }
  return num1 / num2;
};

const splitBillPerPerson = (price: number, count: number) => {
  try {
    const perPerson = divide(price, count);
    return perPerson;
  } catch (cause) {
    if (cause instanceof ZeroDivisionError) {
      throw new Error("人数は1以上の値を指定してください。", { cause });
    }
    throw new Error(
      "エラーが発生しました。カスタマーサポートへご連絡ください。",
      { cause },
    );
  }
};

try {
  alert("一人当たり" + splitBillPerPerson(1000, 0) + "円です。");
} catch (e) {
  alert(e.message);
}
```

ビジネスロジックに基づいたユーザーフレンドリーなエラーメッセージは上位のコードで定義し、下位のコードでは汎用的なメッセージを出力しました。これにより下位のコードの再利用性が高まりました。エラーの種別を判定するため、 `instanceof` 演算子が利用できます。

上位のエラーの `cause` プロパティに下位のエラーを格納することで、エラーの原因をネストしながら上位のコードへ伝搬できます。`cause` プロパティを活用することで、トレーサビリティが向上し、エラーが発生した際に原因の特定の役に立ちます。

:::message

#### エラーコードって必要？

エラーが発生した際にユーザーに表示するため、あるいは仕様書に記載するためにエラーコードが用いられることがあります。 `10001` のような数値で表現されることもあれば、 `E002345` のような文字列で表現されることもあります。

組み込み開発などでメモリが小さかったり、エラーメッセージを表示するほどのディスプレイサイズがないなど、制約の強い環境で利用するのは理にかなっていると思います。しかし、ほとんどの場合は要らないんじゃないかというのが個人的見解です。

マジックナンバーと同様に識別子から意味を推測することが出来ず、仕様書と照らし合わせないと何のエラーなんだかさっぱり分かりません。また、連番形式でエラーを定義すると、後からエラーを追加・削除した際に数値が入り乱れてしまいますし、重複しないように気を使う必要があります。

そんなことをするくらいなら、列挙体を使って管理したほうが見やすいし、整理もしやすいし、コード内で一意性も保証されます。先ほどの例に適用すれば次のようになります。

```typescript
enum UtilityError {
  ZeroDivision = "ZeroDivisionError",
}

class ZeroDivisionError extends Error {
  name = UtilityError.ZeroDivision;
  constructor(num: number) {
    super(`${num} divided by 0.`);
  }
}
```

`10001` よりも `UtilityError.ZeroDivision` の方が圧倒的に分かりやすいです。列挙体で表現することで、グループ化できるし、重複することもありません。

:::

## Result型を返す

先述の `try-catch` の問題に対する1つの解決策として、エラーをResult型にラップし、戻り値として返す方法があります。[neverthrow](https://github.com/supermacro/neverthrow)のようなライブラリを使うことで、このアプローチを簡単に実現できます。

```typescript
import { ok, err } from "neverthrow";

class ZeroDivisionError extends Error {
  name = "ZeroDivisionError";
  constructor(num: number) {
    super(`${num} divided by 0.`);
  }
}

const divide = (num1: number, num2: number) => {
  if (num2 === 0) {
    return err(new ZeroDivisionError(num1));
  }
  return ok(num1 / num2);
};

const splitBillPerPerson = (price: number, count: number) => {
  const perPerson = divide(price, count).mapErr((e) => {
    if (e instanceof ZeroDivisionError) {
      return err(new Error("人数は1以上の値を指定してください。", { cause }));
    }
    return err(
      new Error("エラーが発生しました。カスタマーサポートへご連絡ください。", {
        cause,
      }),
    );
  });
  return perPerson;
};

splitBillPerPerson(1000, 0)
  .map((price) => "一人当たり" + price + "円です。")
  .mapErr((e) => e.message)
  .andThen((message) => alert(message));
```

Result型を用いることで、関数が何のエラーを発生させるのか戻り値の型定義を見るだけで一目瞭然となり、エラーハンドリングの漏れをコンパイル時に検知できます。また、成功時も失敗時も関数が値を返すため、一貫性のあるインターフェースになります。

ただし、完全にthrowから逃れられるわけではありません。Sentryなどのモニタリングサービスにエラーを渡すためにはthrowする必要があったり、依存ライブラリからエラーがthrowされる場合はResult型に変換する必要があります。このように、外部のエコシステムとの繋ぎ目でResult型とエラーを相互変換するという面倒が生じます。

Result型は非標準的なコーディングスタイルを強いるため、TypeScriptに持ち込むべきではなく、そもそもResult型がサポートされた言語を選択すべきだという見方もあります。そうは言っても、フロントエンド開発においては、別の言語を使えというのも乱暴なので、規模やチームに合わせて選択すべきかなと思います。

:::message

#### Railway指向プログラミング

「Railway指向プログラミング」は、ハッピーパス（成功する処理の流れ）のみを考えるのではなく、常に成功のレールと失敗のレールの2つを前提に設計するアプローチです。Result型を導入することで、そのようなコーディングスタイルが可能となります。

[DMMF（Domain Modeling Made Functional）](https://amzn.to/3ZijROy)の著者であるScott Wlaschinが説明している[Railway oriented programming](https://youtu.be/fYo3LN9Vf_M?si=DY41eGZ9y-rluzZm)の動画では、正常系と異常系の行き来をレールというメタファーを用いてうまく図解していて、とても分かりやすかったです。

:::

# 回復可能性

慎重に作り込んだとしても、ソフトウェアは必ずエラーを発生させます。ユーザーは入力をミスをするし、開発者がバグを埋め込んでしまうこともあれば、外部サービスが障害を起こすこともあります。絶対にエラーが起こらないようにするというアプローチは、いくら資金を注ぎ込んでも失敗します。

それよりも、いかにリスクを減らして適切にエラーを処理するかということが肝要です。一口にエラーといっても、リカバリできるものとそうでないものがあり、それぞれ対処が異なります。

## プログラムをリカバリさせる

リカバリ可能なエラーには次のようなものがあります。

- ユーザーの入力エラー
- 一時的なネットワークエラー
- 重要でないタスクのエラー
  - ユーザーの利用状況を記録するタスク
  - アプリケーションのログを記録するタスク

これらのエラーが生じた際に、いちいちアプリケーションがクラッシュしていてはユーザーエクスペリエンスを低下させてしまいます。例えば、次のようにアプリケーションログの書き込みエラーによって、アプリケーションをクラッシュさせて使えなくすることは避けるべきです。

```typescript
try {
  doSomething();
} catch (e) {
  log(JSON.stringify(e)); // ログの書き込みでエラーが発生してクラッシュする
}
```

ログの記録に失敗したとしても、ユーザーにとってはどうでもよく、彼等の目的を達成したいと思うはずです。

```typescript
const safeLog = (err) => {
  try {
    log(JSON.stringify(err));
  } catch (internalErr) {
    console.error(internalErr);
  }
};

try {
  doSomething();
} catch (e) {
  safeLog(e);
}
```

このように、障害発生時に機能を縮小させてでも稼働を継続させる手法を**フェイルソフト**と呼びます。

## プログラムに安らかな死を

前述のようにエラーが発生しても重要な機能を止めないというのは素晴らしいのですが、リカバリできない種類のエラーも存在します。そのような場合、賢明なプログラマはアプリケーションがクラッシュする可能性があると分かっていて、敢えて見過ごします。
そんなの怠慢ではないかと思われるかもしれませんが、決してそうではありません。ここでは簡易的な決済システムを考えてみましょう。

```typescript
class Payment {
  private handler: DatabaseHandler;

  constructor(private db: Database) {
    try {
      this.handler = db.connect(process.env.DB_HOST);
    } catch (e) {
      logger.error(e);
    }
  }

  save(keyValue: PaymentData) {
    try {
      this.handler.insert("payments", keyValue);
    } catch (e) {
      logger.error(e);
    }
  }
}

const onPaymentReceived = (req: Request, res: Response) => {
  const { userId } = req.params;
  const { price } = req.body;

  if (price < 1) {
    res.status(400).send({
      status: "error",
      message: "支払い金額は1円以上を指定してください。",
    });
  }

  const payment = new Payment(db);
  payment.save({ userId, price });
  res.send({
    status: "success",
    message: "支払いが完了しました。",
  });
};

app.post("/payment/:userId", onPaymentReceived);
```

ユーザーから支払いを受け取ると、 `onPaymentReceived` 関数が実行され、支払い金額が1円以上の場合は `payments` テーブルにユーザーIDと支払い金額を保存しています。エラーが発生する処理は忘れず `try-catch` で囲っていて、一見堅牢な実装に思えるかもしれません。

このコードは、エラーが発生するたびにログを出力するものの、致命的な問題が起きても処理を続行します。例えば、データベースに障害が発生して、一時的に接続できなくなったとしましょう。障害が発生している間、ユーザーは支払いデータが保存されていないにも関わらず、「支払いが完了しました。」というメッセージを受け取り続けます。このことの何がまずいのかは言うまでもないですね。

エラーをcatchしてログに出力することでお茶を濁すのではなく、停止させてしまいましょう。

```diff typescript
  class Payment {
    private handler: DatabaseHandler;

    constructor(private db: Database) {
      try {
        this.handler = db.connect(process.env.DB_HOST);
      } catch (e) {
-       logger.error(e);
+       throw e;
      }
    }

    save(keyValue: PaymentData) {
      try {
        this.handler.insert("payments", keyValue);
      } catch (e) {
-       logger.error(e);
+       throw e;
      }
    }
  }
```

このように、障害発生時にシステムを停止させてでも安全に処理する手法を**フェイルセーフ**と呼びます。

ここでもう1つ重要なのが、「**技術的例外とビジネス例外を明確に区別する**」ということです。これは[プログラマが知るべき97のこと](https://xn--97-273ae6a4irb6e2hsoiozc2g4b8082p.com/)の中でDan Bergh Johnssonによって書かれていることです。

先ほどのサンプルコードと比較して、 `logger.error` が呼ばれなくなったことに注目してください。これではエラーがログに記録されないじゃないかと思われるかもしれません。確かにこのコードだけを切り取るとそうなのですが、通常はフレームワークがエラーを受け取って上位のコードで実行されるべきものです。

このことを踏まえると次のように書き換えられます。

```diff typescript
  class Payment {
    private handler: DatabaseHandler;

    constructor(private db: Database) {
-     try {
-       this.handler = db.connect(process.env.DB_HOST);
-     } catch (e) {
-       throw e;
-     }
+     this.handler = db.connect(process.env.DB_HOST);
    }

    save(keyValue: PaymentData) {
-     try {
-       this.handler.insert("payments", keyValue);
-     } catch (e) {
-       throw e;
-     }
+     this.handler.insert("payments", keyValue);
    }
  }

  const onPaymentReceived = (req: Request, res: Response) => {
    const { userId } = req.params;
    const { price } = req.body;

    if (price < 1) {
      res.status(400).send({
        status: "error",
        message: "支払い金額は1円以上を指定してください。",
      });
    }

    const payment = new Payment(db);
    payment.save({ userId, price });
    res.send({
      status: "success",
      message: "支払いが完了しました。",
    });
  };

  app.post("/payment/:userId", onPaymentReceived);
+ app.use((err, req, res) => {
+   res
+     .status(500)
+     .send({
+       status: "error",
+       message: "エラーが発生しました。カスタマーサポートへご連絡ください。",
+     });
+   logger.error(err);
+ });
```

技術的例外は個別にハンドリングする必要はありません。見た目もすっきりとした上に安全なコードになりました。

---

MICINではメンバーを大募集しています。
「とりあえず話を聞いてみたい」でも大歓迎ですので、お気軽にご応募ください！

https://recruit.micin.jp/
