---
layout: page
title: メタプログラミング
category: advanced
order: 6
lang: jp
---

メタプログラミングはコードを書くために、コードを利用する手法のことです。Elixirでは、メタプログラミングを用いることで、言語を拡張して、自分たちのニーズに合わせたり動的にコードを変更することができるようになります。はじめにElixirが内部ではどのように表現されているのかを見ていき、その後Elixirを修正する方法を見ていきます。最終的には、こうした知識を用いて、言語を拡張することができるようになります。

注記: メタプログラミングは用心が必要で、絶対に必要な場合にのみ用いるようにしましょう。過度の使用はほぼ確実にコードの複雑化を招き、理解とデバッグを難しいものにしてしまいます。

## 目次

- [Quote](#quote)
- [Unquote](#unquote)
- [マクロ](#macros)
	- [プライベートなマクロ](#private-macros)
	- [マクロの健康](#macro-hygiene)
	- [バインディング](#binding)

## Quote

メタプログラミングの最初のステップは、式がどのように表現されるかを理解することです。Elixirでは、抽象構文木(AST)、つまり私たちのコードの内部的な表現は、タプルで構成されています。これらのタプルは次の3つの部分から成ります。関数の名前、メタデータ、そして関数の引数です。

これらの内部構造を見るため、Elixirは`quote/2`関数を提供しています。`quote/2`を使うことで、Elixirのコードを下層の表現に変換することができます:

```elixir
iex> quote do: 42
42
iex> quote do: "Hello"
"Hello"
iex> quote do: :world
:world
iex> quote do: 1 + 2
{:+, [context: Elixir, import: Kernel], [1, 2]}
iex> quote do: if value, do: "True", else: "False"
{:if, [context: Elixir, import: Kernel],
 [{:value, [], Elixir}, [do: "True", else: "False"]]}
iex(6)>
```

最初の3つがタプルを返さないことに気づきましたか？quoteされた時に自身を返すリテラルが5つあります:

```elixir
iex> :atom
:atom
iex> "string"
"string"
iex> 1 # 数値
1
iex> [1, 2] # リスト
[1, 2]
iex> {"hello", :world} # 2要素のタプル
{"hello", :world}
```

## Unquote

コードの内部表現を取り出すことができましたが、これをどうやって修正するのでしょうか。新しいコードや値を注入するのは`unquote/1`に任せます。式をunquoteすると、評価されてAST内部に注入されます。`unquote/1`をデモするため、いくつか例を見てみましょう:

```elixir
iex> denominator = 2
2
iex> quote do: divide(42, denominator)
{:divide, [], [42, {:denominator, [], Elixir}]}
iex> quote do: divide(42, unquote(denominator))
{:divide, [], [42, 2]}
```

最初の例では、変数`denominator`はquoteされ、その出力結果のASTは変数にアクセスするためのタプルを含みます。次の`unquote/1`の例では出力されたコードは代わりに`denominator`の値を含んでいます。

## マクロ

`quote/2`と`unquote/q`について理解したので、マクロを深く見ていく用意ができました。大事なので忘れないでください。マクロは他のメタプログラミングと同様、慎重に用いるべきです。

簡潔に表現するなら、マクロはquoteされた式を返すために設計された特別な関数で、その返り値はアプリケーションコードに挿入されます。マクロは関数のように呼び出されるというより、quoteされた式に置き換わるのだと思ってください。マクロを用いることで、Elixirを拡張しアプリケーションに動的にコードを追加するために必要なものが揃います。

`defmacro/2`を使ってマクロを定義することから始めましょう。`defmacro/2`はElixirの多くの要素(じっくり考えてみましょう)と同様、それ自身がマクロです。例として`unless`をマクロとして実装します。マクロがquoteされた式を返す必要があることを忘れないでください:

```elixir
defmodule OurMacro do
  defmacro unless(expr, do: block) do
    quote do
      if !unquote(expr), do: unquote(block)
    end
  end
end
```
モジュールをrequireして、このマクロを試してみましょう:

```elixir
iex> require OurMacro
nil
iex> OurMacro.unless true, do: "Hi"
nil
iex> OurMacro.unless false, do: "Hi"
"Hi"
```

マクロはアプリケーション内のコードを置き換えるので、私たちはいつ、何をコンパイルするか制御することができます。この例として`Logger`モジュールをあげることができます。ログ機能が無効な時には、一切のコードが注入されず、生成されるアプリケーションはLoggerへの参照や関数呼び出しを含みません。これは実装がNOP(何もしない)場合でも関数呼び出しのオーバーヘッドが残るような他の言語とは異なります。

これをデモするため、有効無効を切り替えられる単純なロガーを作ってみます:

```elixir
defmodule Logger do
  defmacro log(msg) do
    if Application.get_env(:logger, :enabled) do
      quote do
        IO.puts("Logged message: #{unquote(msg)}")
      end
    end
  end
end

defmodule Example do
  require Logger

  def test do
    Logger.log("This is a log message")
  end
end
```

ログ機能が有効なら`test`関数は以下のような感じでコードに出力されるでしょう:

```elixir
def test do
  IO.puts("Logged message: #{"This is a log message"}")
end
```

しかし、無効な場合には次のようになるはずです:

```elixir
def test do
end
```

### プライベートなマクロ

ありふれたものではありませんが、Elixirはプライベートなマクロに対応しています。プライベートなマクロは`defmacrop`を用いて定義され、それが定義されているモジュールからのみ呼び出すことができます。プライベートマクロはそれが呼び出されるより前に定義されなくてはなりません。

### マクロの健康

マクロが展開される際、呼び出し元の文脈(訳注:コンテキスト。変数などの環境情報)とどのように相互作用するかというのはマクロの衛生学(macro hygiene)として知られています。デフォルトでは、Elixirのマクロは健全な(hygienic)マクロで、文脈とは衝突しません:

```elixir
defmodule Example do
  defmacro hygienic do
    quote do: val = -1
  end
end

iex> require Example
nil
iex> val = 42
42
iex> Example.hygienic
-1
iex> val
42
```

しかし、`val`の値を操作したい場合はどうでしょうか。変数を不健全なものとしてマークするには`var!/2`を用いることができます。先ほどの例を更新して、`var!/2`を用いた別のマクロを加えます:

```elixir
defmodule Example do
  defmacro hygienic do
    quote do: val = -1
  end

  defmacro unhygienic do
    quote do: var!(val) = -1
  end
end
```

文脈とどのように相互作用するか比較してみましょう:

```elixir
iex> require Example
nil
iex> val = 42
42
iex> Example.hygienic
-1
iex> val
42
iex> Example.unhygienic
-1
iex> val
-1
```

`var!/2`をマクロに含めることで、`val`の値をマクロに渡すことなく操作しました。非健康的なマクロの使用は最小限に抑えるべきです。`var!/2`を含むことで、変数を解決する際に衝突するリスクが増大します。

### バインディング

既に`unquote/1`の活用については扱いましたが、コードに値を注入する別の方法があります。バインディングです。変数バインディングを用いると、マクロ内に複数の変数を加え、1度しかunquoteされないようにし、誤った再評価を防ぐことができます。変数バインディングを用いるには、`quote/2`内でキーワードリストの`bind quoted`オプションを渡す必要があります。

`bind quote`の恩恵を確認し、再評価の問題をデモするため、例を用いましょう。まず、単純に式を2回出力するマクロを作ってみます:

```elixir
defmodule Example do
  defmacro double_puts(expr) do
    quote do
      IO.puts unquote(expr)
      IO.puts unquote(expr)
    end
  end
end
```

この新しいマクロを試すのに、現在のシステム時刻を渡してみましょう。当然2度出力されるべきです:

```elixir
iex> Example.double_puts(:os.system_time)
1450475941851668000
1450475941851733000
```

時刻が異なっています！何が起こったのでしょうか。`unquote/1`を同じ式に複数回用いることで、再評価される結果となり、意図していない挙動になってしまいました。この例で`bind_quoted`を使うように変更し、どうなるか見てみましょう:

```elixir
defmodule Example do
  defmacro double_puts(expr) do
    quote bind_quoted: [expr: expr] do
      IO.puts expr
      IO.puts expr
    end
  end
end

iex> require Example
nil
iex> Example.double_puts(:os.system_time)
1450476083466500000
1450476083466500000
```

`bind_quoted`を用いて、期待した結果を得ました。つまり同じ時刻が2度表示されるようになりました。

`quote/2`、`unquote/1`、そして`defmacro/2`を扱い、Elixirをニーズに合わせて拡張するために必要な全てのツールを手に入れました。
