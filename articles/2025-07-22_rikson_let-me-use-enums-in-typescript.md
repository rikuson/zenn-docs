---
title: "【TypeScript】enumを使ったっていいじゃないか"
emoji: "🙇‍♂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: false
publication_name: micin
---

TypeScriptで定数をどのように表現するか、というのは意外に意見が分かれるテーマです。本稿では、enumの特徴とその代替手段を整理し、何を採用すべきかを考えてみます。

enumを推奨しないという意見が多いのは知っていても、**何故そう言われているのか**、を理解していますか？
「使ってはダメ」と思考停止するのではなく、この記事を読むことでメリット・デメリットを理解して自分で選択できるようになる、かもしれません！

# 列挙型（enum）とは

enumはJavaScriptにはないTypeScript独自の構文で、定数を集合としてまとめて扱いたい時に値をネストすることができます。

```typescript
enum Mode {
  Auto = 'Auto',
  UDP = 'UDP',
  TCP = 'TCP',  
}
const mode: Mode = Mode.Auto
```

文字列だけでなく、数値を列挙することも可能です。次のように書くと値の定義を省略して自動的に0から連番を与えることができます。

```typescript
enum Mode {
  Auto,
  UDP,
  TCP,
}
console.log(Mode.TCP) // 2
```

また、名前を逆引きすることもできます。

```typescript
enum Mode {
  Auto,
  UDP,
  TCP,
}
console.log(Mode[0]) // "Auto"
```

#  利用シーン

例えば、数値フラグによって関数の挙動が変わる場合、数値を直接与えるとその挙動がソースコードから予測しづらいです。

```typescript
connect(1);
```

こういった場合にenumを定義しておくことで、数値に意味づけを与えることができます。

```typescript
enum Mode {
  Auto,
  UDP,
  TCP,
}
connect(Mode.UDP)
```

文字列の場合は、意味は明確なのですが、次のようにタイポなどでエラーになる可能性があります。

```typescript
connect('Aut')
```

enumを定義しておくことで、タイポによるエラーを防止できます。

```typescript
enum Mode {
  Auto = 'Auto',
  UDP = 'UDP',
  TCP = 'TCP',  
}
connect(Mode.Auto)
```

enumを使わなくても単にconst使った定数定義でも良いのですが、値を構造化してまとめて扱いたい際にenumは便利です。

# enumへの批判

一見便利なenumですが、なぜこんなに嫌われているのでしょうか。それには次のような批判が見られます。

## enumはJavaScriptの構文を逸脱している

TypeScriptはJavaScriptを拡張して型システムを導入した言語ですが、型以外の部分でJavaScriptを逸脱した拡張はあまりありません。
しかし、enumに関しては元々JavaScriptに存在しない構文を追加している点で受け入れられないという意見があります。

ただし、ECMAScriptのProposalでenumを追加しようというものもあり、もしかしたらJavaScriptに取り込まれる未来が来るかも・・・！？

https://github.com/tc39/proposal-enum

## Tree-Shakingでサイズが最適化されない

enumはコンパイル後に特殊な挙動を示し、Tree-Shakingが適用されない問題があります。

次のようなコードを考えます。

```typescript
enum Mode {
  Auto,
  UDP,
  TCP,
}
console.log(Mode.Auto)
```

これをコンパイルすると次のようなJavaScriptに展開されます。

```javascript
"use strict";
var Mode;
(function (Mode) {
    Mode[Mode["Auto"] = 0] = "Auto";
    Mode[Mode["UDP"] = 1] = "UDP";
    Mode[Mode["TCP"] = 2] = "TCP";
})(Mode || (Mode = {}));
console.log(Mode.Auto);
```

バンドラーは即時関数をデッドコードと判断することができず、実際に利用していないとしてもバンドルに含まれてしまいます。

この問題を解決するため、TypeScript@1.4からは `const enum` という構文が導入されました。

```typescript
const enum Mode {
  Auto,
  UDP,
  TCP,
}
console.log(Mode.Auto)
```

コンパイルすると、次のように値がインライン展開されます。

```typescript
"use strict";
console.log(0 /* Mode.Auto */);
```

## const enumの罠

`const enum` の利用によってTree-Shakingの問題は解決されますが、その代わりに新たな落とし穴がいくつか存在します。

### babelでコンパイルできない

`const enum` を含むコードを `@babel/preset-typescript` を使ってコンパイルしようとすると以下のようなエラーになります。

```
SyntaxError: 'const' enums are not supported.
```

Babelで利用する場合はプラグインを利用するなどの対応が必要になります。

### フルビルドしないと参照先がenumの定義変更に追随できない

`const enum` の値は利用元のファイルに直接埋め込まれるため、参照先と定義元の同期がとれないことがあります。

例えば以下のような構造を考えます。

```typescript
// constants.ts
export const enum Status {
  Active = 1,
  Inactive = 2,
}

// main.ts
import { Status } from './constants';
console.log(Status.Active); // コンパイル後 → console.log(1);
```

ここで `Status.Active = 10` に変更した場合でも、開発時におけるインクリメンタルビルドでは `constants.ts` の変更だけでは `main.ts` が再コンパイルされないことがあります。結果、`main.ts` には古い値（1）が残り続け、バグを引き起こします。

そのため、モノレポ・アプリケーション間での共有や、ライブラリの公開APIに含めることなどは非推奨となっています。

## Node.jsと互換性がない

最新のNode.jsではTypeScriptのコードを直接実行できますが、enumやnamespace、クラスのパラメータプロパティなどの構文が含まれていると、次のようなエラーになります。

```
SyntaxError [ERR_INVALID_TYPESCRIPT_SYNTAX]:   x TypeScript enum is not supported in strip-only mode
```

TypeScriptの `erasableSyntaxOnly` というオプションを利用すると、これらの構文が禁止され、Node.js互換なコードを書くことができます。

Node.jsの実行時に `--experimental-transform-types` フラグを付ければenumが含まれていもエラーになりませんが、実験的な機能です。

## 型の安全性に問題がある

次のようなコードでコンパイルエラーにならないという数値列挙型の型安全性の問題を指摘する声もあります。ただし、この問題はTypeScript5.0で解決済みです。

```typescript
enum Mode {
  Auto,
  UDP,
  TCP,
}
const mode: Mode = 5
```

また、逆マッピングを使ってキーを取得する方法がありますが、存在しない値を指定した際にエラーにならず、undefinedになります。

```typescript
enum Mode {
  Auto,
  UDP,
  TCP,
}
console.log(Mode[0]) // Auto
console.log(Mode[5]) // undefined
```

## 挙動に一貫性がない

数値列挙型は非公称型となり、文字列列挙型は公称型となります。

```typescript
enum StringEnum {
  Foo = "foo",
}
enum NumberEnum {
  Bar = 1,
}
const foo1: StringEnum = StringEnum.Foo // OK
const foo2: StringEnum = "foo" // Error
const bar1: NumberEnum = NumberEnum.Bar // OK
const bar2: NumberEnum = 1 // OK
```

# 代替手段と、そのトレードオフ

例えば、enumを使って次のように定数 `Mode` を定義し、それを利用して型 `Proxy` を定義します。

```typescript
enum Mode {
  Auto = 'Auto',
  UDP = 'UDP',
  TCP = 'TCP',
}
export type Proxy = {
  // ...
  mode: Mode; // 型定義: (property) mode: Mode
}
```

このコードをそれぞれ別の方法で書き直してみましょう。

## ユニオン型を使う

enumを使わず、ユニオン型でシンプルに定義することも可能です。

```typescript
const MODE_AUTO = 'Auto'
const MODE_UDP = 'UDP'
const MODE_TCP = 'TCP'
type Mode = typeof MODE_AUTO | typeof MODE_UDP | typeof MODE_TCP // 型定義: type Mode = 'Auto' | 'UDP' | 'TCP'
export type Proxy = {
  // ...
  mode: Mode; // 型定義: (property) mode: Mode
}
```

公称型ではないので、文字列を直接指定してもエラーにはなってくれません。

enumのようにひとまとまりで扱えないので、変数の命名を揃えるなどして工夫する必要があります。

## オブジェクトリテラルを使う

オブジェクトリテラルに `as const` を付けることで、プロパティをreadonlyにできます。この方法を使うと、次のように書けます。

```typescript
const Mode = {
  Auto: 'Auto',
  UDP: 'UDP',
  TCP: 'TCP',
} as const
type Mode = (typeof Mode)[keyof typeof Mode] // 型定義: type Mode = 'Auto' | 'UDP' | 'TCP'
export type Proxy = {
  // ...
  mode: Mode; // 型定義: (property) mode: Mode
}
```

オブジェクトリテラルを利用することで、enumと同様に値をひとまとまりに扱うことができますが、型定義がやや冗長になります。

こちらも同様に公称型ではないので、文字列を直接指定してもエラーにはなってくれません。

# 結局どうすればいいのか

個人的には逆マッピングは使わないので型の安全性の問題はそこまで気にならないし、Tree-Shakingについてもそこまで大きなサイズ差が生じるシナリオが思いつかず、もうBabelも使ってないしNode.jsで直接TypeScriptを動かすこともないので、そこまで**ひどい**と言うほどのものでもないような気がしています。
また、代替手段を利用した場合、公称型ではなくなり直接値を代入できてしまうので、コーディングスタイルが統一されないリスクも生じます。

正直、どの方法も一長一短があり、enumを使うことは誤りであると断言することはできず、結局は好みの問題と思います。
共同開発では嫌われたくないので、反対派がいればそれに合わせます。別に固執するほどのことでもないし。

よかったら、皆さんのご意見も聞かせてください！

