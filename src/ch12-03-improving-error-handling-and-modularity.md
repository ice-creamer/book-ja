<!--
## Refactoring to Improve Modularity and Error Handling
-->

## リファクタリングしてモジュール性とエラー処理を向上させる

<!--
To improve our program, we’ll fix four problems that have to do with the
program’s structure and how it’s handling potential errors.
-->

プログラムを改善するために、プログラムの構造と起こりうるエラーに対処する方法に関連する4つの問題を修正していきましょう。

<!--
First, our `main` function now performs two tasks: it parses arguments and
opens files. For such a small function, this isn’t a major problem. However, if
we continue to grow our program inside `main`, the number of separate tasks the
`main` function handles will increase. As a function gains responsibilities, it
becomes more difficult to reason about, harder to test, and harder to change
without breaking one of its parts. It’s best to separate functionality so each
function is responsible for one task.
-->

1番目は、`main`関数が2つの仕事を受け持っていることです: 引数を解析し、ファイルを開いています。
このような小さな関数なら、これは、大した問題ではありませんが、`main`内でプログラムを巨大化させ続けたら、
`main`関数が扱う個別の仕事の数も増えていきます。関数が責任を受け持つごとに、
正しいことを確認しにくくなり、テストも行いづらくなり、機能を壊さずに変更するのも困難になっていきます。
機能を小分けして、各関数が1つの仕事のみに責任を持つようにするのが最善です。

<!--
This issue also ties into the second problem: although `query` and `filename`
are configuration variables to our program, variables like `f` and `contents`
are used to perform the program’s logic. The longer `main` becomes, the more
variables we’ll need to bring into scope; the more variables we have in scope,
the harder it will be to keep track of the purpose of each. It’s best to group
the configuration variables into one structure to make their purpose clear.
-->

この問題は、2番目の問題にも結びついています: `query`と`filename`はプログラムの設定用変数ですが、
`f`や`contents`といった変数は、プログラムのロジックを担っています。`main`が長くなるほど、
スコープに入れるべき変数も増えます。そして、スコープにある変数が増えれば、各々の目的を追うのも大変になるわけです。
設定用変数を一つの構造に押し込め、目的を明瞭化するのが最善です。

<!--
The third problem is that we’ve used `expect` to print an error message when
opening the file fails, but the error message just prints `file not found`.
Opening a file can fail in a number of ways besides the file being missing: for
example, the file might exist, but we might not have permission to open it.
Right now, if we’re in that situation, we’d print the `file not found` error
message, which would give the user the wrong information!
-->

3番目の問題は、ファイルを開き損ねた時に`expect`を使ってエラーメッセージを出力しているのに、
エラーメッセージが`ファイルが見つかりませんでした`としか表示しないことです。
ファイルを開く行為は、ファイルが存在しない以外にもいろんな方法で失敗することがあります:
例えば、ファイルは存在するかもしれないけれど、開く権限がないかもしれないなどです。
現時点では、そのような状況になった時、「ファイルが見つかりませんでした」というエラーメッセージを出力し、
これはユーザに間違った情報を与えるのです。

<!--
1行目最後の方のandを順接の理由で訳している
-->

<!--
Fourth, we use `expect` repeatedly to handle different errors, and if the user
runs our program without specifying enough arguments, they’ll get an `index out
of bounds` error from Rust that doesn’t clearly explain the problem. It would
be best if all the error-handling code was in one place so future maintainers
have only one place to consult in the code if the error-handling logic needs to
change. Having all the error-handling code in one place will also ensure that
we’re printing messages that will be meaningful to our end users.
-->

4番目は、異なるエラーを処理するのに`expect`を繰り返し使用しているので、ユーザが十分な数の引数を渡さずにプログラムを起動した時に、
問題を明確に説明しない「範囲外アクセス(index out of bounds)」というエラーがRustから得られることです。
エラー処理のコードが全て1箇所に存在し、将来エラー処理ロジックが変更になった時に、
メンテナンス者が1箇所のコードのみを考慮すればいいようにするのが最善でしょう。
エラー処理コードが1箇所にあれば、エンドユーザにとって意味のあるメッセージを出力していることを確認することにもつながります。

<!--
Let’s address these four problems by refactoring our project.
-->

プロジェクトをリファクタリングして、これら4つの問題を扱いましょう。

<!--
### Separation of Concerns for Binary Projects
-->

### バイナリプロジェクトの責任の分離

<!--
The organizational problem of allocating responsibility for multiple tasks to
the `main` function is common to many binary projects. As a result, the Rust
community has developed a process to use as a guideline for splitting the
separate concerns of a binary program when `main` starts getting large. The
process has the following steps:
-->

`main`関数に複数の仕事の責任を割り当てるという構造上の問題は、多くのバイナリプロジェクトでありふれています。
結果として、`main`が肥大化し始めた際にバイナリプログラムの個別の責任を分割するためにガイドラインとして活用できる工程をRustコミュニティは、
開発しました。この工程は、以下のような手順になっています:

<!--
* Split your program into a *main.rs* and a *lib.rs* and move your program’s
logic to *lib.rs*.
* As long as your command line parsing logic is small, it can remain in
*main.rs*.
* When the command line parsing logic starts getting complicated, extract it
from *main.rs* and move it to *lib.rs*.
-->

* プログラムを*main.rs*と*lib.rs*に分け、ロジックを*lib.rs*に移動する。
* コマンドライン引数の解析ロジックが小規模な限り、*main.rs*に置いても良い。
* コマンドライン引数の解析ロジックが複雑化の様相を呈し始めたら、*main.rs*から抽出して*lib.rs*に移動する。

<!--
The responsibilities that remain in the `main` function after this process
should be limited to the following:
-->

この工程の後に`main`関数に残る責任は以下に限定される:

<!--
* Calling the command line parsing logic with the argument values
* Setting up any other configuration
* Calling a `run` function in *lib.rs*
* Handling the error if `run` returns an error
-->

* 引数の値でコマンドライン引数の解析ロジックを呼び出す
* 他のあらゆる設定を行う
* *lib.rs*の`run`関数を呼び出す
* `run`がエラーを返した時に処理する

<!--
This pattern is about separating concerns: *main.rs* handles running the
program, and *lib.rs* handles all the logic of the task at hand. Because we
can’t test the `main` function directly, this structure lets us test all of
your program’s logic by moving it into functions in *lib.rs*. The only code
that remains in *main.rs* will be small enough to verify its correctness by
reading it. Let’s rework our program by following this process.
-->

このパターンは、責任の分離についてです: *main.rs*はプログラムの実行を行い、
そして、*lib.rs*が手にある仕事のロジック全てを扱います。`main`関数を直接テストすることはできないので、
この構造により、プログラムのロジック全てを*lib.rs*の関数に移すことでテストできるようになります。
*main.rs*に残る唯一のコードは、読めばその正当性が評価できるだけ小規模になるでしょう。
この工程に従って、プログラムのやり直しをしましょう。

<!--
#### Extracting the Argument Parser
-->

#### 引数解析器を抽出する

<!--
We’ll extract the functionality for parsing arguments into a function that
`main` will call to prepare for moving the command line parsing logic to
*src/lib.rs*. Listing 12-5 shows the new start of `main` that calls a new
function `parse_config`, which we’ll define in *src/main.rs* for the moment.
-->

引数解析の機能を`main`が呼び出す関数に抽出して、コマンドライン引数解析ロジックを*src/lib.rs*に移動する準備をします。
リスト12-5に新しい関数`parse_config`を呼び出す`main`の冒頭部を示し、
この新しい関数は今だけ*src/main.rs*に定義します。

<!--
<span class="filename">Filename: src/main.rs</span>
-->

<span class="filename">ファイル名: src/main.rs</span>

```rust,ignore
fn main() {
    let args: Vec<String> = env::args().collect();

    let (query, filename) = parse_config(&args);

    // --snip--
}

fn parse_config(args: &[String]) -> (&str, &str) {
    let query = &args[1];
    let filename = &args[2];

    (query, filename)
}
```

<!--
<span class="caption">Listing 12-5: Extracting a `parse_config` function from
`main`</span>
-->

<span class="caption">リスト12-5: `main`から`parse_config`関数を抽出する</span>

<!--
We’re still collecting the command line arguments into a vector, but instead of
assigning the argument value at index 1 to the variable `query` and the
argument value at index 2 to the variable `filename` within the `main`
function, we pass the whole vector to the `parse_config` function. The
`parse_config` function then holds the logic that determines which argument
goes in which variable and passes the values back to `main`. We still create
the `query` and `filename` variables in `main`, but `main` no longer has the
responsibility of determining how the command line arguments and variables
correspond.
-->

それでもまだ、コマンドライン引数をベクタに集結させていますが、`main`関数内で引数の値の添え字1を変数`query`に、
添え字2を変数`filename`に代入する代わりに、ベクタ全体を`parse_config`関数に渡しています。
そして、`parse_config`関数にはどの引数がどの変数に入り、それらの値を`main`に返すというロジックが存在します。
まだ`main`内に`query`と`filename`という変数を生成していますが、もう`main`は、
コマンドライン引数と変数がどう対応するかを決定する責任は持ちません。

<!--
This rework may seem like overkill for our small program, but we’re refactoring
in small, incremental steps. After making this change, run the program again to
verify that the argument parsing still works. It’s good to check your progress
often, to help you identify the cause of problems when they occur.
-->

このやり直しは、私たちの小規模なプログラムにはやりすぎに思えるかもしれませんが、
少しずつ段階的にリファクタリングしているのです。この変更後、プログラムを再度実行して、
引数解析がまだ動作していることを実証してください。問題が発生した時に原因を特定する助けにするために頻繁に進捗を確認するのはいいことです。

<!--
#### Grouping Configuration Values
-->

#### 設定値をまとめる

<!--
We can take another small step to improve the `parse_config` function further.
At the moment, we’re returning a tuple, but then we immediately break that
tuple into individual parts again. This is a sign that perhaps we don’t have
the right abstraction yet.
-->

もう少し`parse_config`関数を改善することができます。現時点では、タプルを返していますが、
即座にタプルを分解して再度個別の値にしています。これは、正しい抽象化をまだできていないかもしれない兆候です。

<!--
Another indicator that shows there’s room for improvement is the `config` part
of `parse_config`, which implies that the two values we return are related and
are both part of one configuration value. We’re not currently conveying this
meaning in the structure of the data other than grouping the two values into
a tuple; we could put the two values into one struct and give each of the
struct fields a meaningful name. Doing so will make it easier for future
maintainers of this code to understand how the different values relate to each
other and what their purpose is.
-->

まだ改善の余地があると示してくれる他の徴候は、`parse_config`の`config`の部分であり、
返却している二つの値は関係があり、一つの設定値の一部にどちらもなることを暗示しています。
現状では、一つのタプルにまとめていること以外、この意味をデータの構造に載せていません;
この二つの値を1構造体に置き換え、構造体のフィールドそれぞれに意味のある名前をつけることもできるでしょう。
そうすることで将来このコードのメンテナンス者が、異なる値が相互に関係する仕方や、目的を理解しやすくできるでしょう。

<!--
> Note: Some people call this anti-pattern of using primitive values when a
> complex type would be more appropriate *primitive obsession*.
-->

> 注釈: この複雑型(complex type)がより適切な時に組み込みの値を使うアンチパターンを、
> *primitive obsession*(`訳注`: 初めて聞いた表現。*組み込み型強迫観念*といったところだろうか)と呼ぶ人もいます。

<!--
Listing 12-6 shows the improvements to the `parse_config` function.
-->

リスト12-6は、`parse_config`関数の改善を示しています。

<!--
<span class="filename">Filename: src/main.rs</span>
-->

<span class="filename">ファイル名: src/main.rs</span>

```rust,should_panic
# use std::env;
# use std::fs::File;
#
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = parse_config(&args);

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    let mut f = File::open(config.filename).expect("file not found");

    // --snip--
}

struct Config {
    query: String,
    filename: String,
}

fn parse_config(args: &[String]) -> Config {
    let query = args[1].clone();
    let filename = args[2].clone();

    Config { query, filename }
}
```

<!--
<span class="caption">Listing 12-6: Refactoring `parse_config` to return an
instance of a `Config` struct</span>
-->

<span class="caption">リスト12-6: `parse_config`をリファクタリングして`Config`構造体のインスタンスを返す</span>

<!--
We’ve added a struct named `Config` defined to have fields named `query` and
`filename`. The signature of `parse_config` now indicates that it returns a
`Config` value. In the body of `parse_config`, where we used to return string
slices that reference `String` values in `args`, we now define `Config` to
contain owned `String` values. The `args` variable in `main` is the owner of
the argument values and is only letting the `parse_config` function borrow
them, which means we’d violate Rust’s borrowing rules if `Config` tried to take
ownership of the values in `args`.
-->

`query`と`filename`というフィールドを持つよう定義された`Config`という構造体を追加しました。
`parse_config`のシグニチャは、これで`Config`値を返すと示すようになりました。`parse_config`の本体では、
以前は`args`の`String`値を参照する文字列スライスを返していましたが、
今では所有する`String`値を含むように`Config`を定義しています。`main`の`args`変数は引数値の所有者であり、
`parse_config`関数だけに借用させていますが、これは`Config`が`args`の値の所有権を奪おうとしたら、
Rustの借用規則に違反してしまうことを意味します。

<!--
We could manage the `String` data in a number of different ways, but the
easiest, though somewhat inefficient, route is to call the `clone` method on
the values. This will make a full copy of the data for the `Config` instance to
own, which takes more time and memory than storing a reference to the string
data. However, cloning the data also makes our code very straightforward
because we don’t have to manage the lifetimes of the references; in this
circumstance, giving up a little performance to gain simplicity is a worthwhile
trade-off.
-->

`String`のデータは、多くの異なる手法で管理できますが、最も単純だけれどもどこか非効率的な手段は、
値に対して`clone`メソッドを呼び出すことです。これにより、`Config`インスタンスが所有するデータの総コピーが生成されるので、
文字列データへの参照を保持するよりも時間とメモリを消費します。ですが、データをクローンすることで、
コードがとても素直にもなります。というのも、参照のライフタイムを管理する必要がないからです。
つまり、この場面において、少々のパフォーマンスを犠牲にして単純性を得るのは、価値のある代償です。

<!--
> ### The Trade-Offs of Using `clone`
>
> There’s a tendency among many Rustaceans to avoid using `clone` to fix
> ownership problems because of its runtime cost. In Chapter 13, you’ll learn
> how to use more efficient methods in this type of situation. But for now,
> it’s okay to copy a few strings to continue making progress because we’ll
> make these copies only once and your filename and query string are very
> small. It’s better to have a working program that’s a bit inefficient than to
> try to hyperoptimize code on your first pass. As you become more experienced
> with Rust, it’ll be easier to start with the most efficient solution, but for
> now, it’s perfectly acceptable to call `clone`.
-->

> ### `clone`を使用する代償
>
> 実行時コストのために`clone`を使用して所有権問題を解消するのを避ける傾向が多くのRustaceanにあります。
> 第13章で、この種の状況においてより効率的なメソッドの使用法を学ぶでしょう。ですがとりあえずは、
> これらのコピーをするのは1回だけですし、ファイル名とクエリ文字列は非常に小さなものなので、
> いくつかの文字列をコピーして進捗するのは良しとしましょう。最初の通り道でコードを究極的に効率化しようとするよりも、
> ちょっと非効率的でも動くプログラムを用意する方がいいでしょう。もっとRustの経験を積めば、
> 最も効率的な解決法から開始することも簡単になるでしょうが、今は、`clone`を呼び出すことは完璧に受け入れられることです。

<!--
We’ve updated `main` so it places the instance of `Config` returned by
`parse_config` into a variable named `config`, and we updated the code that
previously used the separate `query` and `filename` variables so it now uses
the fields on the `Config` struct instead.
-->

`main`を更新したので、`parse_config`から返された`Config`のインスタンスを`config`という変数に置くようになり、
以前は個別の`query`と`filename`変数を使用していたコードを更新したので、代わりに`Config`構造体のフィールドを使用するようになりました。

<!--
Now our code more clearly conveys that `query` and `filename` are related and
that their purpose is to configure how the program will work. Any code that
uses these values knows to find them in the `config` instance in the fields
named for their purpose.
-->

これでコードは`query`と`filename`が関連していることと、その目的がプログラムの振る舞い方を設定するということをより明確に伝えます。
これらの値を使用するあらゆるコードは、`config`インスタンスの目的の名前を冠したフィールドにそれらを発見することを把握しています。

<!--
#### Creating a Constructor for `Config`
-->

#### `Config`のコンストラクタを作成する

<!--
So far, we’ve extracted the logic responsible for parsing the command line
arguments from `main` and placed it in the `parse_config` function. Doing so
helped us to see that the `query` and `filename` values were related and that
relationship should be conveyed in our code. We then added a `Config` struct to
name the related purpose of `query` and `filename` and to be able to return the
values’ names as struct field names from the `parse_config` function.
-->

ここまでで、コマンドライン引数を解析する責任を負ったロジックを`main`から抽出し、`parse_config`関数に配置しました。
そうすることで`query`と`filename`の値が関連し、その関係性がコードに載っていることを確認する助けになりました。
それから`Config`構造体を追加して`query`と`filename`の関係する目的を名前付けし、
構造体のフィールド名として`parse_config`関数からその値の名前を返すことができています。

<!--
So now that the purpose of the `parse_config` function is to create a `Config`
instance, we can change `parse_config` from a plain function to a function
named `new` that is associated with the `Config` struct. Making this change
will make the code more idiomatic. We can create instances of types in the
standard library, such as `String`, by calling `String::new`. Similarly, by
changing `parse_config` into a `new` function associated with `Config`, we’ll
be able to create instances of `Config` by calling `Config::new`. Listing 12-7
shows the changes we need to make.
-->

したがって、今や`parse_config`関数の目的は`Config`インスタンスを生成することになったので、
`parse_config`をただの関数から`Config`構造体に紐づく`new`という関数に変えることができます。
この変更を行うことで、コードがより慣用的になります。`String`などの標準ライブラリの型のインスタンスを、
`String::new`を呼び出すことで生成できます。同様に、`parse_config`を`Config`に紐づく`new`関数に変えれば、
`Config::new`を呼び出すことで`Config`のインスタンスを生成できるようになります。リスト12-7が、
行う必要のある変更を示しています。

<!--
<span class="filename">Filename: src/main.rs</span>
-->

<span class="filename">ファイル名: src/main.rs</span>

```rust,should_panic
# use std::env;
#
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args);

    // --snip--
}

# struct Config {
#     query: String,
#     filename: String,
# }
#
// --snip--

impl Config {
    fn new(args: &[String]) -> Config {
        let query = args[1].clone();
        let filename = args[2].clone();

        Config { query, filename }
    }
}
```

<!--
<span class="caption">Listing 12-7: Changing `parse_config` into
`Config::new`</span>
-->

<span class="caption">リスト12-7: `parse_config`を`Config::new`に変える</span>

<!--
We’ve updated `main` where we were calling `parse_config` to instead call
`Config::new`. We’ve changed the name of `parse_config` to `new` and moved it
within an `impl` block, which associates the `new` function with `Config`. Try
compiling this code again to make sure it works.
-->

`parse_config`を呼び出していた`main`を代わりに`Config::new`を呼び出すように更新しました。
`parse_config`の名前を`new`に変え、`impl`ブロックに入れ込んだので、`new`関数と`Config`が紐づくようになりました。
再度このコードをコンパイルしてみて、動作することを確かめてください。

<!--
### Fixing the Error Handling
-->

### エラー処理を修正する

<!--
Now we’ll work on fixing our error handling. Recall that attempting to access
the values in the `args` vector at index 1 or index 2 will cause the program to
panic if the vector contains fewer than three items. Try running the program
without any arguments; it will look like this:
-->

さて、エラー処理の修正に取り掛かりましょう。ベクタが2個以下の要素しか含んでいないときに`args`ベクタの添え字1か2にアクセスしようとすると、
プログラムがパニックすることを思い出してください。試しに引数なしでプログラムを実行してください。すると、こんな感じになります:

<<<<<<< HEAD
```text
$ cargo run
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/minigrep`
thread 'main' panicked at 'index out of bounds: the len is 1
but the index is 1', src/main.rs:29:21
(スレッド'main'は、「境界外アクセス: 長さは1なのに添え字も1です」でパニックしました)
note: Run with `RUST_BACKTRACE=1` for a backtrace.
=======
```console
{{#include ../listings/ch12-an-io-project/listing-12-07/output.txt}}
>>>>>>> upstream/master
```

<!--
The line `index out of bounds: the len is 1 but the index is 1` is an error
message intended for programmers. It won’t help our end users understand what
happened and what they should do instead. Let’s fix that now.
-->

`境界外アクセス: 長さは1なのに添え字も1です`という行は、プログラマ向けのエラーメッセージです。
エンドユーザが起きたことと代わりにすべきことを理解する手助けにはならないでしょう。これを今修正しましょう。

<!--
#### Improving the Error Message
-->

#### エラーメッセージを改善する

<!--
In Listing 12-8, we add a check in the `new` function that will verify that the
slice is long enough before accessing index 1 and 2. If the slice isn’t long
enough, the program panics and displays a better error message than the `index
out of bounds` message.
-->

リスト12-8で、`new`関数に、添え字1と2にアクセスする前にスライスが十分長いことを実証するチェックを追加しています。
スライスの長さが十分でなければ、プログラムはパニックし、`境界外インデックス`よりもいいエラーメッセージを表示します。

<!--
<span class="filename">Filename: src/main.rs</span>
-->

<span class="filename">ファイル名: src/main.rs</span>

```rust,ignore
// --snip--
fn new(args: &[String]) -> Config {
    if args.len() < 3 {
        // 引数の数が足りません
        panic!("not enough arguments");
    }
    // --snip--
```

<!--
<span class="caption">Listing 12-8: Adding a check for the number of
arguments</span>
-->

<span class="caption">リスト12-8: 引数の数のチェックを追加する</span>

<!--
This code is similar to the `Guess::new` function we wrote in Listing 9-9,
where we called `panic!` when the `value` argument was out of the range of
valid values. Instead of checking for a range of values here, we’re checking
that the length of `args` is at least 3 and the rest of the function can
operate under the assumption that this condition has been met. If `args` has
fewer than three items, this condition will be true, and we call the `panic!`
macro to end the program immediately.
-->

このコードは、リスト9-9で記述した`value`引数が正常な値の範囲外だった時に`panic!`を呼び出した`Guess::new`関数と似ています。
ここでは、値の範囲を確かめる代わりに、`args`の長さが少なくとも3であることを確かめていて、
関数の残りの部分は、この条件が満たされているという前提のもとで処理を行うことができます。
`args`に2要素以下しかなければ、この条件は真になり、`panic!`マクロを呼び出して、即座にプログラムを終了させます。

<!--
With these extra few lines of code in `new`, let’s run the program without any
arguments again to see what the error looks like now:
-->

では、`new`のこの追加の数行がある状態で、再度引数なしでプログラムを走らせ、エラーがどんな見た目か確かめましょう:

<<<<<<< HEAD
```text
$ cargo run
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/minigrep`
thread 'main' panicked at 'not enough arguments', src/main.rs:30:12
(スレッド'main'は「引数が足りません」でパニックしました)
note: Run with `RUST_BACKTRACE=1` for a backtrace.
=======
```console
{{#include ../listings/ch12-an-io-project/listing-12-08/output.txt}}
>>>>>>> upstream/master
```

<!--
This output is better: we now have a reasonable error message. However, we also
have extraneous information we don’t want to give to our users. Perhaps using
the technique we used in Listing 9-9 isn’t the best to use here: a call to
`panic!` is more appropriate for a programming problem than a usage problem, as
discussed in Chapter 9. Instead, we can use the other technique you learned
about in Chapter 9—returning a `Result` that indicates either success or an
error.
-->

この出力の方がマシです: これでエラーメッセージが合理的になりました。ですが、
ユーザに与えたくない追加の情報も含まれてしまっています。おそらく、
ここではリスト9-9で使用したテクニックを使用するのは最善ではありません: 
`panic!`の呼び出しは、第9章で議論したように、使用の問題よりもプログラミング上の問題により適しています。
代わりに、第9章で学んだもう一つのテクニックを使用することができます。成功か失敗かを示唆する`Result`を返すことです。

<!--
#### Returning a `Result` from `new` Instead of Calling `panic!`
-->

#### `panic!`を呼び出す代わりに`new`から`Result`を返す

<!--
We can instead return a `Result` value that will contain a `Config` instance in
the successful case and will describe the problem in the error case. When
`Config::new` is communicating to `main`, we can use the `Result` type to
signal there was a problem. Then we can change `main` to convert an `Err`
variant into a more practical error for our users without the surrounding text
about `thread 'main'` and `RUST_BACKTRACE` that a call to `panic!` causes.
-->

代わりに、成功時には`Config`インスタンスを含み、エラー時には問題に言及する`Result`値を返すことができます。
`Config::new`が`main`と対話する時、`Result`型を使用して問題があったと信号を送ることができます。
それから`main`を変更して、`panic!`呼び出しが引き起こしていた`thread 'main'`と`RUST_BACKTRACE`に関する周囲のテキストがない、
ユーザ向けのより実用的なエラーに`Err`列挙子を変換することができます。

<!--
Listing 12-9 shows the changes we need to make to the return value of
`Config::new` and the body of the function needed to return a `Result`. Note
that this won’t compile until we update `main` as well, which we’ll do in the
next listing.
-->

リスト12-9は、`Config::new`の戻り値に必要な変更と`Result`を返すのに必要な関数の本体を示しています。
`main`も更新するまで、これはコンパイルできないことに注意してください。その更新は次のリストで行います。

<!--
<span class="filename">Filename: src/main.rs</span>
-->

<span class="filename">ファイル名: src/main.rs</span>

```rust,ignore
impl Config {
    fn new(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let filename = args[2].clone();

        Ok(Config { query, filename })
    }
}
```

<!--
<span class="caption">Listing 12-9: Returning a `Result` from
`Config::new`</span>
-->

<span class="caption">リスト12-9: `Config::new`から`Result`を返却する</span>

<!--
Our `new` function now returns a `Result` with a `Config` instance in the
success case and a `&'static str` in the error case. Recall from “The Static
Lifetime” section in Chapter 10 that `&'static str` is the type of string
literals, which is our error message type for now.
-->

`new`関数は、これで、成功時には`Config`インスタンスを、エラー時には`&'static str`を伴う`Result`を返すようになりました。
第10章の「静的ライフタイム」節から`&'static str`は文字列リテラルの型であることを思い出してください。
これは、今はエラーメッセージの型になっています。

<!--
We’ve made two changes in the body of the `new` function: instead of calling
`panic!` when the user doesn’t pass enough arguments, we now return an `Err`
value, and we’ve wrapped the `Config` return value in an `Ok`. These changes
make the function conform to its new type signature.
-->

`new`関数の本体で2つ変更を行いました: 十分な数の引数をユーザが渡さなかった場合に`panic!`を呼び出す代わりに、
今は`Err`値を返し、`Config`戻り値を`Ok`に包んでいます。これらの変更により、関数が新しい型シグニチャに適合するわけです。

<!--
Returning an `Err` value from `Config::new` allows the `main` function to
handle the `Result` value returned from the `new` function and exit the process
more cleanly in the error case.
-->

`Config::new`から`Err`値を返すことにより、`main`関数は、`new`関数から返ってくる`Result`値を処理し、
エラー時により綺麗にプロセスから抜け出すことができます。

<!--
#### Calling `Config::new` and Handling Errors
-->

#### `Config::new`を呼び出し、エラーを処理する

<!--
To handle the error case and print a user-friendly message, we need to update
`main` to handle the `Result` being returned by `Config::new`, as shown in
Listing 12-10. We’ll also take the responsibility of exiting the command line
tool with a nonzero error code from `panic!` and implement it by hand. A
nonzero exit status is a convention to signal to the process that called our
program that the program exited with an error state.
-->

エラーケースを処理し、ユーザフレンドリーなメッセージを出力するために、`main`を更新して、
リスト12-10に示したように`Config::new`から返されている`Result`を処理する必要があります。
また、`panic!`からコマンドラインツールを0以外のエラーコードで抜け出す責任も奪い取り、
手作業でそれも実装します。0以外の終了コードは、
我々のプログラムを呼び出したプロセスにプログラムがエラー状態で終了したことを通知する慣習です。

<!--
<span class="filename">Filename: src/main.rs</span>
-->

<span class="filename">ファイル名: src/main.rs</span>

```rust,ignore
use std::process;

fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args).unwrap_or_else(|err| {
        // 引数解析時に問題
        println!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    // --snip--
```

<!--
<span class="caption">Listing 12-10: Exiting with an error code if creating a
new `Config` fails</span>
-->

<span class="caption">リスト12-10: 新しい`Config`作成に失敗したら、エラーコードで終了する</span>

<!--
In this listing, we’ve used a method we haven’t covered before:
`unwrap_or_else`, which is defined on `Result<T, E>` by the standard library.
Using `unwrap_or_else` allows us to define some custom, non-`panic!` error
handling. If the `Result` is an `Ok` value, this method’s behavior is similar
to `unwrap`: it returns the inner value `Ok` is wrapping. However, if the value
is an `Err` value, this method calls the code in the *closure*, which is an
anonymous function we define and pass as an argument to `unwrap_or_else`. We’ll
cover closures in more detail in Chapter 13. For now, you just need to know
that `unwrap_or_else` will pass the inner value of the `Err`, which in this
case is the static string `not enough arguments` that we added in Listing 12-9,
to our closure in the argument `err` that appears between the vertical pipes.
The code in the closure can then use the `err` value when it runs.
-->

このリストにおいて、以前には講義していないメソッドを使用しました: `unwrap_or_else`です。
これは標準ライブラリで`Result<T, E>`に定義されています。`unwrap_or_else`を使うことで、
`panic!`ではない何らか独自のエラー処理を定義できるのです。この`Result`が`Ok`値だったら、
このメソッドの振る舞いは`unwrap`に似ています: `Ok`が包んでいる中身の値を返すのです。
しかし、値が`Err`値なら、このメソッドは、*クロージャ*内でコードを呼び出し、
クロージャは私たちが定義し、引数として`unwrap_or_else`に渡す匿名関数です。クロージャについては第13章で詳しく講義します。
とりあえず、`unwrap_or_else`は、今回リスト12-9で追加した`not enough arguments`という静的文字列の`Err`の中身を、
縦棒の間に出現する`err`引数のクロージャに渡していることだけ知っておく必要があります。
クロージャのコードはそれから、実行された時に`err`値を使用できます。

<!--
We’ve added a new `use` line to import `process` from the standard library. The
code in the closure that will be run in the error case is only two lines: we
print the `err` value and then call `process::exit`. The `process::exit`
function will stop the program immediately and return the number that was
passed as the exit status code. This is similar to the `panic!`-based handling
we used in Listing 12-8, but we no longer get all the extra output. Let’s try
it:
-->

新規`use`行を追加して標準ライブラリから`process`をインポートしました。クロージャ内のエラー時に走るコードは、
たった2行です: `err`の値を出力し、それから`process::exit`を呼び出します。`process::exit`関数は、
即座にプログラムを停止させ、渡された数字を終了コードとして返します。これは、リスト12-8で使用した`panic!`ベースの処理と似ていますが、
もう余計な出力はされません。試しましょう:

<<<<<<< HEAD
```text
$ cargo run
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.48 secs
     Running `target/debug/minigrep`
Problem parsing arguments: not enough arguments
=======
```console
{{#include ../listings/ch12-an-io-project/listing-12-10/output.txt}}
>>>>>>> upstream/master
```

<!--
Great! This output is much friendlier for our users.
-->

素晴らしい！この出力の方が遥かにユーザに優しいです。

<!--
### Extracting Logic from `main`
-->

### `main`からロジックを抽出する

<!--
Now that we’ve finished refactoring the configuration parsing, let’s turn to
the program’s logic. As we stated in “Separation of Concerns for Binary
Projects”, we’ll extract a function named `run` that will hold all the logic
currently in the `main` function that isn’t involved with setting up
configuration or handling errors. When we’re done, `main` will be concise and
easy to verify by inspection, and we’ll be able to write tests for all the
other logic.
-->

これで設定解析のリファクタリングが終了したので、プログラムのロジックに目を向けましょう。
「バイナリプロジェクトの責任の分離」で述べたように、
現在`main`関数に存在する設定のセットアップやエラー処理に関わらない全てのロジックを保持することになる`run`という関数を抽出します。
やり終わったら、`main`は簡潔かつ視察で確かめやすくなり、他のロジック全部に対してテストを書くことができるでしょう。

<!--
Listing 12-11 shows the extracted `run` function. For now, we’re just making
the small, incremental improvement of extracting the function. We’re still
defining the function in *src/main.rs*.
-->

リスト12-11は、抜き出した`run`関数を示しています。今は少しずつ段階的に関数を抽出する改善を行っています。
それでも、*src/main.rs*に関数を定義していきます。

<!--
<span class="filename">Filename: src/main.rs</span>
-->

<span class="filename">ファイル名: src/main.rs</span>

```rust,ignore
fn main() {
    // --snip--

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    run(config);
}

fn run(config: Config) {
    let mut f = File::open(config.filename).expect("file not found");

    let mut contents = String::new();
    f.read_to_string(&mut contents)
        .expect("something went wrong reading the file");

    println!("With text:\n{}", contents);
}

// --snip--
```

<!--
<span class="caption">Listing 12-11: Extracting a `run` function containing the
rest of the program logic</span>
-->

<span class="caption">リスト12-11: 残りのプログラムロジックを含む`run`関数を抽出する</span>

<!--
The `run` function now contains all the remaining logic from `main`, starting
from reading the file. The `run` function takes the `Config` instance as an
argument.
-->

これで`run`関数は、ファイル読み込みから始まる`main`関数の残りのロジック全てを含むようになりました。
この`run`関数は、引数に`Config`インスタンスを取ります。

<!--
#### Returning Errors from the `run` Function
-->

#### `run`関数からエラーを返す

<!--
1行目。ここではwith ...を順接の理由で訳している。with ...は普通、状態を表す表現
ちょっと意味が強すぎるかもしれない。
With you next to me, I'll drive to wherever you like. (君が隣にいる状態で、何処へでも君の好きな場所にドライブするよ)
-->

<!--
With the remaining program logic separated into the `run` function, we can
improve the error handling, as we did with `Config::new` in Listing 12-9.
Instead of allowing the program to panic by calling `expect`, the `run`
function will return a `Result<T, E>` when something goes wrong. This will let
us further consolidate into `main` the logic around handling errors in a
user-friendly way. Listing 12-12 shows the changes we need to make to the
signature and body of `run`.
-->

残りのプログラムロジックが`run`関数に隔離されたので、リスト12-9の`Config::new`のように、
エラー処理を改善することができます。`expect`を呼び出してプログラムにパニックさせる代わりに、
`run`関数は、何か問題が起きた時に`Result<T, E>`を返します。これにより、
さらにエラー処理周りのロジックをユーザに優しい形で`main`に統合することができます。
リスト12-12にシグニチャと`run`本体に必要な変更を示しています。

<!--
<span class="filename">Filename: src/main.rs</span>
-->

<span class="filename">ファイル名: src/main.rs</span>

```rust,ignore
use std::error::Error;

// --snip--

fn run(config: Config) -> Result<(), Box<Error>> {
    let mut f = File::open(config.filename)?;

    let mut contents = String::new();
    f.read_to_string(&mut contents)?;

    println!("With text:\n{}", contents);

    Ok(())
}
```

<!--
<span class="caption">Listing 12-12: Changing the `run` function to return
`Result`</span>
-->

<span class="caption">リスト12-12: `run`関数を変更して`Result`を返す</span>

<!--
We’ve made three significant changes here. First, we changed the return type of
the `run` function to `Result<(), Box<Error>>`. This function previously
returned the unit type, `()`, and we keep that as the value returned in the
`Ok` case.
-->

ここでは、3つの大きな変更を行いました。まず、`run`関数の戻り値を`Result<(), Box<Error>>`に変えました。
この関数は、以前はユニット型、`()`を返していて、それを`Ok`の場合に返される値として残しました。

<!--
For the error type, we used the *trait object* `Box<Error>` (and we’ve brought
`std::error::Error` into scope with a `use` statement at the top). We’ll cover
trait objects in Chapter 17. For now, just know that `Box<Error>` means the
function will return a type that implements the `Error` trait, but we don’t
have to specify what particular type the return value will be. This gives us
flexibility to return error values that may be of different types in different
error cases.
-->

エラー型については、*トレイトオブジェクト*の`Box<Error>`を使用しました(同時に冒頭で`use`文により、
`std::error::Error`をスコープに導入しています)。トレイトオブジェクトについては、第17章で講義します。
とりあえず、`Box<Error>`は、関数が`Error`トレイトを実装する型を返すことを意味しますが、
戻り値の型を具体的に指定しなくても良いことを知っておいてください。これにより、
エラーケースによって異なる型のエラー値を返す柔軟性を得ます。

<!--
Second, we’ve removed the calls to `expect` in favor of the `?` operator, as we
talked about in Chapter 9. Rather than `panic!` on an error, the `?` operator
will return the error value from the current function for the caller to handle.
-->

2番目に、`expect`の呼び出しよりも`?`演算子を選択して取り除きました。第9章で語りましたね。
エラーでパニックするのではなく、`?`演算子は呼び出し元が処理できるように、現在の関数からエラー値を返します。

<!--
Third, the `run` function now returns an `Ok` value in the success case. We’ve
declared the `run` function’s success type as `()` in the signature, which
means we need to wrap the unit type value in the `Ok` value. This `Ok(())`
syntax might look a bit strange at first, but using `()` like this is the
idiomatic way to indicate that we’re calling `run` for its side effects only;
it doesn’t return a value we need.
-->

3番目に、`run`関数は今、成功時に`Ok`値を返すようになりました。`run`関数の成功型は、
シグニチャで`()`と定義したので、ユニット型の値を`Ok`値に包む必要があります。
最初は、この`Ok(())`という記法は奇妙に見えるかもしれませんが、このように`()`を使うことは、
`run`を副作用のためだけに呼び出していると示唆する慣習的な方法です; 必要な値は返しません。

<!--
When you run this code, it will compile but will display a warning:
-->

このコードを実行すると、コンパイルは通るものの、警告が表示されるでしょう:

<<<<<<< HEAD
```text
warning: unused `std::result::Result` which must be used
(警告: 使用されなければならない`std::result::Result`が未使用です)
  --> src/main.rs:18:5
   |
18 |     run(config);
   |     ^^^^^^^^^^^^
= note: #[warn(unused_must_use)] on by default
=======
```console
{{#include ../listings/ch12-an-io-project/listing-12-12/output.txt}}
>>>>>>> upstream/master
```

<!--
3行目中盤、andだが、逆接のように訳している。andはフローが流れていることを表すだけなので、こうなっている模様
-->

<!--
Rust tells us that our code ignored the `Result` value and the `Result` value
might indicate that an error occurred. But we’re not checking to see whether or
not there was an error, and the compiler reminds us that we probably meant to
have some error-handling code here! Let’s rectify that problem now.
-->

コンパイラは、コードが`Result`値を無視していると教えてくれて、この`Result`値は、
エラーが発生したと示唆しているかもしれません。しかし、エラーがあったか確認するつもりはありませんが、
コンパイラは、ここにエラー処理コードを書くつもりだったんじゃないかと思い出させてくれています！
今、その問題を改修しましょう。

<!--
#### Handling Errors Returned from `run` in `main`
-->

#### `main`で`run`から返ってきたエラーを処理する

<!--
We’ll check for errors and handle them using a technique similar to one we used
with `Config::new` in Listing 12-10, but with a slight difference:
-->

リスト12-10の`Config::new`に対して行った方法に似たテクニックを使用してエラーを確認し、扱いますが、
少し違いがあります:

<!--
<span class="filename">Filename: src/main.rs</span>
-->

<span class="filename">ファイル名: src/main.rs</span>

```rust,ignore
fn main() {
    // --snip--

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    if let Err(e) = run(config) {
        println!("Application error: {}", e);

        process::exit(1);
    }
}
```

<!--
We use `if let` rather than `unwrap_or_else` to check whether `run` returns an
`Err` value and call `process::exit(1)` if it does. The `run` function doesn’t
return a value that we want to `unwrap` in the same way that `Config::new`
returns the `Config` instance. Because `run` returns `()` in the success case
we only care about detecting an error, so we don’t need `unwrap_or_else` to
return the unwrapped value because it would only be `()`.
-->

`unwrap_or_else`ではなく、`if let`で`run`が`Err`値を返したかどうかを確認し、そうなら`process::exit(1)`を呼び出しています。
`run`関数は、`Config::new`が`Config`インスタンスを返すのと同じように`unwrap`したい値を返すことはありません。
`run`は成功時に`()`を返すので、エラーを検知することにのみ興味があり、`()`でしかないので、
`unwrap_or_else`に包まれた値を返してもらう必要はないのです。

<!--
The bodies of the `if let` and the `unwrap_or_else` functions are the same in
both cases: we print the error and exit.
-->

`if let`と`unwrap_or_else`関数の中身はどちらも同じです: エラーを出力して終了します。

<!--
### Splitting Code into a Library Crate
-->

### コードをライブラリクレートに分割する

<!--
Our `minigrep` project is looking good so far! Now we’ll split the
*src/main.rs* file and put some code into the *src/lib.rs* file so we can test
it and have a *src/main.rs* file with fewer responsibilities.
-->

ここまで`minigrep`は良さそうですね！では、テストを行え、*src/main.rs*ファイルの責任が減らせるように、
*src/main.rs*ファイルを分割し、一部のコードを*src/lib.rs*ファイルに置きましょう。

<!--
Let’s move all the code that isn’t the `main` function from *src/main.rs* to
*src/lib.rs*:
-->

`main`関数以外のコード全部を*src/main.rs*から*src/lib.rs*に移動しましょう:

<!--
* The `run` function definition
* The relevant `use` statements
* The definition of `Config`
* The `Config::new` function definition
-->

* `run`関数定義
* 関係する`use`文
* `Config`の定義
* `Config::new`関数定義

<!--
The contents of *src/lib.rs* should have the signatures shown in Listing 12-13
(we’ve omitted the bodies of the functions for brevity). Note that this won't
compile until we modify *src/main.rs* in Listing 12-14.
-->

*src/lib.rs*の中身にはリスト12-13に示したようなシグニチャがあるはずです(関数の本体は簡潔性のために省略しました)。
リスト12-14で*src/main.rs*に変更を加えるまで、このコードはコンパイルできないことに注意してください。

<!--
<span class="filename">Filename: src/lib.rs</span>
-->

<span class="filename">ファイル名: src/lib.rs</span>

```rust,ignore
use std::error::Error;
use std::fs::File;
use std::io::prelude::*;

pub struct Config {
    pub query: String,
    pub filename: String,
}

impl Config {
    pub fn new(args: &[String]) -> Result<Config, &'static str> {
        // --snip--
    }
}

pub fn run(config: Config) -> Result<(), Box<Error>> {
    // --snip--
}
```

<!--
<span class="caption">Listing 12-13: Moving `Config` and `run` into
*src/lib.rs*</span>
-->

<span class="caption">リスト12-13: `Config`と`run`を*src/lib.rs*に移動する</span>

<!--
We’ve made liberal use of `pub` here: on `Config`, on its fields and its
`new` method, and on the `run` function. We now have a library crate that has a
public API that we can test!
-->

ここでは、寛大に`pub`を使用しています: `Config`のフィールドと`new`メソッドと`run`関数です。
これでテスト可能な公開APIのあるライブラリクレートができました！

<!--
Now we need to bring the code we moved to *src/lib.rs* into the scope of the
binary crate in *src/main.rs*, as shown in Listing 12-14.
-->

さて、*src/lib.rs*に移動したコードを*src/main.rs*のバイナリクレートのスコープに持っていく必要があります。
リスト12-14に示したようにですね。

<!--
<span class="filename">Filename: src/main.rs</span>
-->

<span class="filename">ファイル名: src/main.rs</span>

```rust,ignore
extern crate minigrep;

use std::env;
use std::process;

use minigrep::Config;

fn main() {
    // --snip--
    if let Err(e) = minigrep::run(config) {
        // --snip--
    }
}
```

<!--
<span class="caption">Listing 12-14: Bringing the `minigrep` crate into the
scope of *src/main.rs*</span>
-->

<span class="caption">リスト12-14: `minigrep`クレートを*src/main.rs*のスコープに持っていく</span>

<!--
To bring the library crate into the binary crate, we use `extern crate
minigrep`. Then we add a `use minigrep::Config` line to bring the `Config` type
into scope, and we prefix the `run` function with our crate name. Now all the
functionality should be connected and should work. Run the program with `cargo
run` and make sure everything works correctly.
-->

ライブラリクレートをバイナリクレートに持っていくのに、`extern crate minigrep`を使用しています。
それから`use minigrep::Config`行を追加して`Config`型をスコープに持ってきて、
`run`関数にクレート名を接頭辞として付けます。これで全機能が連結され、動くはずです。
`cargo run`でプログラムを走らせて、すべてがうまくいっていることを確かめてください。

<!--
Whew! That was a lot of work, but we’ve set ourselves up for success in the
future. Now it’s much easier to handle errors, and we’ve made the code more
modular. Almost all of our work will be done in *src/lib.rs* from here on out.
-->

ふう！作業量が多かったですね。ですが、将来成功する準備はできています。
もう、エラー処理は遥かに楽になり、コードのモジュール化もできました。
ここから先の作業は、ほぼ*src/lib.rs*で完結するでしょう。

<!--
Let’s take advantage of this newfound modularity by doing something that would
have been difficult with the old code but is easy with the new code: we’ll
write some tests!
-->

古いコードでは大変だけれども、新しいコードでは楽なことをして新発見のモジュール性を活用しましょう:
テストを書くのです！
