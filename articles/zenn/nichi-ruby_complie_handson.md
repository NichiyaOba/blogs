## はじめに

LRM株式会社でエンジニアをしています。大場です。

先日、RubyKaigi 2025に参加してきました。
day1~day3まで全てのセッションに参加しましたが、正直day1はほとんど理解が追いつきませんでした。頻出するパーサーやレクサー、YJITといった言葉のイメージが湧かず、day1終わりに改めてRubyのコンパイル過程を調べました。調べた後臨んだday2以降のセッションは、内容に対する解像度が上がり、論点をきちんと汲み取れるようになりました。この経験から、Rubyのコンパイル過程を復習してみました。Rubyがどのように解釈されているのか確認してみます。

使用するコードは下記に格納しましたので適宜ご利用ください。

https://github.com/NichiyaOba/ruby-practice

## Rubyのコンパイル過程について

Rubyのコンパイル過程は以下の手順で行われています。

1. Rubyコードをレクサーが字句解析してトークン(単語のかたまり)を作る
2. トークンをパーサーが構文解析して抽象構文木を作る
3. 抽象構文木からYARV命令列が生成される
4. YARV命令列をRubyVMが逐次実行する
   YJITが有効化されていた場合、実行時にネイティブコード(機械語)が生成され、最適化が行われる

レクサー、パーサー、RubyVMについて簡単に紹介します。

### レクサー(Lexer)

Rubyのソースコードを読み取り、「キーワード」「変数名」「記号」などの意味を持つ単位(トークン)に分解します。これは自然言語でいうと「文を単語に分ける」工程に似ています。

### パーサー(Parser)

トークンの並びをもとに、文法規則に従ってプログラムの構造(抽象構文木：AST)を構築します。
たとえば if 文や def メソッドのような構造がどのように入れ子になっているかを理解する役割を担います。

### RubyVM(Ruby Virtual Machine)

抽象構文木を受け取り、中間コード(バイトコード)に変換し、Rubyの処理系が実行できる形式にします。
この仮想マシン(VM)は、物理CPUではなくソフトウェアとして動作することで、Rubyコードを柔軟に解釈・実行します。
Ruby 2.0以降は YARV(Yet Another Ruby VM)という名前のVMが使われています。

今回は`Hello, RubyKaigi!`を100回出力するコードでRuby実行時のコンパイル過程を追いかけます。

https://github.com/NichiyaOba/ruby-practice/blob/main/sample.rb


## ハンズオン

### トークン生成時の字句解析
最初のステップはruby文法の字句解析です。ここでrubyの文法に反するものを検知しエラーを返します。これにより後続のトークン変換が正確に実行できる状態を担保します。

ruby文法として許容する文字列を定義するために、オートマトン理論が使われていますのでざっくりと説明を。今回使用する決定性有限オートマトン(以下DFA)については以下の通りです。

1. 最終的に受容する状態を予め定義しておきます。
2. 「初期状態からaを入力するとa'の状態に、bを入力するとb'の状態に遷移する。任意の状態と任意の入力に対して、次の状態が一意に定まる。」というルールに則り、入力された言語の通りに状態を遷移させます。
3. 全ての入力の後に受容する状態にあれば受け入れます。

いきなりモヤっとした方もいらっしゃるかもしれませんが、まずは実際に`def`を受容するDFAで評価してみましょう。`def_dfa.rb`を作成して実行してみます。

https://github.com/NichiyaOba/ruby-practice/blob/main/dfa/def_dfa.rb

https://github.com/NichiyaOba/ruby-practice/blob/main/check_def_dfa.rb

`ruby check_def_dfa.rb`を実行すると

```ruby
✅ 'def' は正しく 'def' として受理されました
❌ スペルミス検出！do は位置 2（文字 'o'）で不正
🔍 行7: 'do' は 'def' の誤記かもしれません
```

と出てきます。

defは正しく受容されていることがわかります。`def`の誤記を検知させたい場合は`sample.rb`の`def`を`dfe`などに変えてみると検知できることがわかると思います。

この挙動について`def_dfa.rb`を基に考えてみます。初期値`d`を見つけたらそこから状態遷移をスタート`q0`の初期状態から`d`の入力を検知すると`q1`に遷移します。それ以外の場合は状態遷移を行いません。引き続き`e`が入力されると`q2`の状態に遷移します。同様に`f`の入力を検知すると`q3`の状態に遷移します。

この`q3`の状態のみ受容するようにしているため`d`から始まる単語は`def`のみが許容されるという挙動になります。

ただし、先述の通り`def_dfa.rb`では`def`のみを受容するため、`d`が出現したことを検知すると`do`は`def`であるべきではないかと指摘されます。

というわけでdoも受容するDFAを定義して試しましょう。`def_do_dfa.rb`と`check_def_do_dfa.rb`を作成します。

https://github.com/NichiyaOba/ruby-practice/blob/main/dfa/def_do_dfa.rb

https://github.com/NichiyaOba/ruby-practice/blob/main/check_def_do_dfa.rb

作成を終えたら`ruby check_def_do_dfa.rb`を実行します。

```ruby
✅ 'def' は正しくキーワードとして受理されました
✅ 'do' は正しくキーワードとして受理されました
```

正しく`def`と`do`が受容されていることがわかります。

実際のRuby本体では`parse.y`に文法が定義されています。この文法定義は Lrama、Bisonといったパーサージェネレーターに渡され、C言語で実装された構文解析器である`parse.c`が生成されます。今回のハンズオンのように直接的な状態遷移は`parse.y`に記述されていませんが、パーサージェネレーターによって生成された構文解析器の振る舞いはハンズオンで実装した状態遷移と同様です。

https://github.com/ruby/ruby/blob/master/parse.y

ただし、do に対する end の対応や、ネスト構造の整合性といった文法エラーは、構文木の構築時(構文解析フェーズ)で初めて検出されるため、字句解析レベルのDFA処理では判定できません。

### レクサー(Lexer)で作られたトークンを確認する

次に、字句解析された`.rb`形式のファイルからレクサーがどのようなトークンを生成しているか調べます。`sample.rb`と同じディレクトリに`lex_inspect.rb`を作成して実行してみます。

https://github.com/NichiyaOba/ruby-practice/blob/main/lex_inspect.rb

`ruby lex_inspect.rb`を実行すると下記出力が得られました。これがレクサーによって作成されたトークンです。

```ruby
[[[1, 0], :on_comment, "# sample.rb\n", BEG],
 [[2, 0], :on_kw, "def", FNAME],
 [[2, 3], :on_sp, " ", FNAME],
 [[2, 4], :on_ident, "greet", ENDFN],
 [[2, 9], :on_lparen, "(", BEG|LABEL],
 [[2, 10], :on_ident, "name", ARG],
 [[2, 14], :on_rparen, ")", ENDFN],
 [[2, 15], :on_ignored_nl, "\n", BEG],
 [[3, 0], :on_sp, "  ", BEG],
 [[3, 2], :on_ident, "message", CMDARG],
 [[3, 9], :on_sp, " ", CMDARG],
 [[3, 10], :on_op, "=", BEG],
 [[3, 11], :on_sp, " ", BEG],
 [[3, 12], :on_tstring_beg, "\"", BEG],
 [[3, 13], :on_tstring_content, "Hello, ", BEG],
 [[3, 20], :on_embexpr_beg, "\#{", BEG],
 [[3, 22], :on_ident, "name", END|LABEL],
 [[3, 26], :on_embexpr_end, "}", END|LABEL],
 [[3, 27], :on_tstring_content, "!", BEG],
 [[3, 28], :on_tstring_end, "\"", END],
 [[3, 29], :on_nl, "\n", BEG],
 [[4, 0], :on_sp, "  ", BEG],
 [[4, 2], :on_ident, "puts", CMDARG],
 [[4, 6], :on_sp, " ", CMDARG],
 [[4, 7], :on_ident, "message", END|LABEL],
 [[4, 14], :on_nl, "\n", BEG],
 [[5, 0], :on_kw, "end", END],
 [[5, 3], :on_nl, "\n", BEG],
 [[6, 0], :on_ignored_nl, "\n", BEG],
 [[7, 0], :on_int, "100", END],
 [[7, 3], :on_period, ".", DOT],
 [[7, 4], :on_ident, "times", ARG],
 [[7, 9], :on_sp, " ", ARG],
 [[7, 10], :on_kw, "do", BEG],
 [[7, 12], :on_ignored_nl, "\n", BEG],
 [[8, 0], :on_sp, "  ", BEG],
 [[8, 2], :on_ident, "greet", CMDARG],
 [[8, 7], :on_lparen, "(", BEG|LABEL],
 [[8, 8], :on_tstring_beg, "\"", BEG|LABEL],
 [[8, 9], :on_tstring_content, "RubyKaigi", BEG|LABEL],
 [[8, 18], :on_tstring_end, "\"", END],
 [[8, 19], :on_rparen, ")", ENDFN],
 [[8, 20], :on_nl, "\n", BEG],
 [[9, 0], :on_kw, "end", END],
 [[9, 3], :on_nl, "\n", BEG]]
```

トークン種別は下記になります。

| 種別               | 意味                 | 例                           |
|--------------------|----------------------|------------------------------|
| `:on_kw`           | キーワード           | `def`, `end`, `do`, `if`     |
| `:on_ident`        | 識別子（変数名など） | `name`, `message`            |
| `:on_op`           | 演算子               | `=`, `+`, `-`                |
| `:on_int`          | 整数                 | `100`                        |
| `:on_tstring_content` | 文字列の中身     | `"Hello"` の中の `Hello`     |
| `:on_embexpr_beg`  | `#{` の始まり        | 式展開開始                   |
| `:on_period`       | `.` ドット           | `object.method` の `.`       |
| `:on_sp`           | スペース（空白）     | `" "`                         |
| `:on_nl`           | 改行（通常の改行）   | `\n`                          |
| `:on_ignored_nl`   | 無視される改行       | 文法上意味のない改行         |
| `:on_lparen`       | `(` 左括弧           | `greet(name)` の `(`         |
| `:on_rparen`       | `)` 右括弧           | 同上                         |

この`ripper`の挙動はRubyのソースコードを見れば詳細が確認できます。

https://github.com/ruby/ruby/blob/master/lib/prism/translation/ripper.rb

このトークン化された情報をもとに次ステップでパーサーが構文木を確認します。

### パーサー(Parser)で作られた構文木を確認する

レクサーによって作られたトークンが構文木に変換されている様子を確認します。`sample.rb`と同ディレクトリに`ast_inspect.rb`を作成します。

https://github.com/NichiyaOba/ruby-practice/blob/main/ast_inspect.rb

作成が完了したら`ruby ast_inspect.rb`を実行します。

```ruby
[:program,
 [[:def,
   [:@ident, "greet", [2, 4]],
   [:paren, [:params, [[:@ident, "name", [2, 10]]], nil, nil, nil, nil, nil, nil]],
   [:bodystmt,
    [[:assign,
      [:var_field, [:@ident, "message", [3, 2]]],
      [:string_literal,
       [:string_content,
        [:@tstring_content, "Hello, ", [3, 13]],
        [:string_embexpr, [[:var_ref, [:@ident, "name", [3, 22]]]]],
        [:@tstring_content, "!", [3, 27]]]]],
     [:command,
      [:@ident, "puts", [4, 2]],
      [:args_add_block, [[:var_ref, [:@ident, "message", [4, 7]]]], false]]],
    nil,
    nil,
    nil]],
  [:method_add_block,
   [:call, [:@int, "100", [7, 0]], [:@period, ".", [7, 3]], [:@ident, "times", [7, 4]]],
   [:do_block,
    nil,
    [:bodystmt,
     [[:method_add_arg,
       [:fcall, [:@ident, "greet", [8, 2]]],
       [:arg_paren,
        [:args_add_block,
         [[:string_literal, [:string_content, [:@tstring_content, "RubyKaigi", [8, 9]]]]],
         false]]]],
     nil,
     nil,
     nil]]]]]
```

実行結果を見てみると、

```ruby
[[:def,
   [:@ident, "greet", [2, 4]],
```

2行目に`def`があり、3行目にはレクサーによってトークン化された` [[2, 4], :on_ident, "greet", ENDFN],`の記述があり、`def greet`の記述がメソッド定義として表現されているのがわかります。

```ruby
 [:paren, [:params, [[:@ident, "name", [2, 10]]], nil, nil, nil, nil, nil, nil]],
```

4行目には引数として変数名nameを受け付けることが` [[2, 10], :on_ident, "name", ARG],`のトークンを使用して表現されています。

```ruby
[:var_field, [:@ident, "message", [3, 2]]],
      [:string_literal,
       [:string_content,
        [:@tstring_content, "Hello, ", [3, 13]],
        [:string_embexpr, [[:var_ref, [:@ident, "name", [3, 22]]]]],
        [:@tstring_content, "!", [3, 27]]]]],
```

また7~12行目を見てみると、`"message"`の変数に`[:@tstring_content, "Hello, ", [3, 13]],`の「Hello,」と` [:string_embexpr, [[:var_ref, [:@ident, "name", [3, 22]]]]],`で表現された引数の「name」,`[:@tstring_content, "!", [3, 27]]`の「!」が格納されています。

```ruby
     [:command,
      [:@ident, "puts", [4, 2]],
      [:args_add_block, [[:var_ref, [:@ident, "message", [4, 7]]]], false]]],
```

13~15行目ではputsが呼ばれ、messageの出力が定義されています。

```ruby
 [:call, [:@int, "100", [7, 0]], [:@period, ".", [7, 3]], [:@ident, "times", [7, 4]]],
   [:do_block,
    nil,
    [:bodystmt,
     [[:method_add_arg,
       [:fcall, [:@ident, "greet", [8, 2]]],
       [:arg_paren,
        [:args_add_block,
         [[:string_literal, [:string_content, [:@tstring_content, "RubyKaigi", [8, 9]]]]],
```

20行目を見ると`100`と`times`と`call`で100回呼び出し、25行目でgreetを実行する、引数には28行目の`RubyKaigi`を使うと定義されており、コードで表された一連の流れが抽象構文木でも表現されていることがわかります。

### RubyVM::InstructionSequenceで作られたYARV命令列を確認する

次に、抽象構文木から生成されたYARV命令列を確認してみます。`iseq_inspect.rb`を作成します。

https://github.com/NichiyaOba/ruby-practice/blob/main/iseq_inspect.rb

作成が完了したら`ruby iseq_inspect.rb`を実行します。

```ruby
== disasm: #<ISeq:<main>@sample.rb:1 (1,0)-(9,3)>
0000 definemethod                           :greet, greet             (   2)[Li]
0003 putobject                              100                       (   7)[Li]
0005 send                                   <calldata!mid:times, argc:0>, block in <main>
0008 leave

== disasm: #<ISeq:greet@sample.rb:2 (2,0)-(5,3)>
local table (size: 2, argc: 1 [opt　　s: 0, rest: -1, post: 0, block: -1, kw: -1@-1, kwrest: -1])
[ 2] name@0<Arg>[ 1] message@1
0000 putobject                              "Hello, "                 (   3)[LiCa]
0002 getlocal_WC_0                          name@0
0004 dup
0005 objtostring                            <calldata!mid:to_s, argc:0, FCALL|ARGS_SIMPLE>
0007 anytostring
0008 putobject                              "!"
0010 concatstrings                          3
0012 setlocal_WC_0                          message@1
0014 putself                                                          (   4)[Li]
0015 getlocal_WC_0                          message@1
0017 opt_send_without_block                 <calldata!mid:puts, argc:1, FCALL|ARGS_SIMPLE>
0019 leave                                                            (   5)[Re]

== disasm: #<ISeq:block in <main>@sample.rb:7 (7,10)-(9,3)>
0000 putself                                                          (   8)[LiBc]
0001 putstring                              "RubyKaigi"
0003 opt_send_without_block                 <calldata!mid:greet, argc:1, FCALL|ARGS_SIMPLE>
0005 leave                                                            (   9)[Br]
```

上記実行結果を確認します。
たとえばgreetメソッドを定義する2ブロック目では、0000~0010で「Hello,」、「引数：name@0」、「!」の3つをスタックに積んで`concatstrings`で結合することで`Hello, #{name@0}!`を作成しています。0012ではmessage変数に代入、0014では今のオブジェクトで定義されているmessageメソッドを参照できるようにレシーバを指定しています。0015はレシーバを元にmessageをスタックに積み、0017ではmessageを使用して`puts`の処理を実行、0019でメソッドの実行終了およびputsの戻り値を返して処理を終了しています。

3ブロック目でRubyKaigiを引数として渡したgreetメソッドの実行が、1ブロック目ではgreetメソッドの100回実行が定義されています。

Ruby仮想マシンは内部的にC言語を使用しており、YARV命令列を基に対応したC言語を実行していきます。上記のYARV命令列がRuby仮想マシンに認識されC言語として実行されるという流れです。

### YJITで機械語実行された割合を確認する

Ruby仮想マシンがYARV命令列→C言語に変換して実行するとき、一部をC言語を介さず直接ネイティブコードで実行する仕組みがYJITです。`Ruby→C言語→バイナリコード`の間のC言語を取り除き`Ruby→バイナリコード`の流れで実行することでC言語を間に挟むことなく直接バイナリコードを実行、高速化できます。

これまでの`sample.rb`を`ruby --yjit --yjit-stats sample.rb`で実行すると以下の結果が返ってきます。

```ruby
***YJIT: Printing YJIT statistics on exit***
method call fallback reasons: 
    (all relevant counters are zero)
invokeblock fallback reasons: 
    (all relevant counters are zero)
invokesuper fallback reasons: 
    (all relevant counters are zero)
method call exit reasons: 
    (all relevant counters are zero)
invokeblock exit reasons: 
    (all relevant counters are zero)
invokesuper exit reasons: 
    (all relevant counters are zero)
getblockparamproxy exit reasons: 
    (all relevant counters are zero)
getinstancevariable exit reasons:
    (all relevant counters are zero)
setinstancevariable exit reasons:
    (all relevant counters are zero)
leave exit reasons:
    interp_return:         71 (100.0%)
left shift (ltlt) exit reasons: 
    (all relevant counters are zero)
invalidation reasons: 
    (all relevant counters are zero)
　　　・・・　　　省略　　　・・・
freed_code_size:                   0
yjit_alloc_size:               5,211
live_context_size:               165
live_context_count:               11
live_page_count:                   1
freed_page_count:                  0
code_gc_count:                     0
num_gc_obj_refs:                   9
object_shape_count:              227
side_exit_count:                   0
total_exit_count:                 71
total_insns_count:           237,045
vm_insns_count:              235,696
yjit_insns_count:              1,349
ratio_in_yjit:                  0.6%
avg_len_in_yjit:                19.0
total_exits:                    0
Top-1 most frequent C calls (100.0% of C calls):
    Object#puts:         71 (100.0%)
```

`total_insns_count:237,045`が全命令行数であり、そのうちYJITによってネイティブコードで実行された行数が`yjit_insns_count:1,349`になります。割合は`ratio_in_yjit:0.6%`で確認することできます。
0.6%とかなり低い数値になっていますが、`puts`は内部的にC言語で実装されており、直接ネイティブコードに変換できません。必ずC言語を経由して機械語になっているため「Ruby→機械語」の割合を示す`ratio_in_yjit`が低い値を示しています。

実験的に「Ruby→機械語」でC言語を経由せずに処理してくれそうな簡単な演算処理でも検証してみました。math.rbを作成して実行してみます。

https://github.com/NichiyaOba/ruby-practice/blob/main/math.rb

この処理ならC言語で実装された`puts`の処理と比較して`ratio_in_yjit`が高い値を示しそうです。

`ruby --yjit --yjit-stats math.rb`を実行すると以下の結果が得られます。

```ruby
***YJIT: Printing YJIT statistics on exit***
method call fallback reasons: 
    (all relevant counters are zero)
invokeblock fallback reasons: 
    (all relevant counters are zero)
invokesuper fallback reasons: 
    (all relevant counters are zero)
method call exit reasons: 
    (all relevant counters are zero)
invokeblock exit reasons: 
    (all relevant counters are zero)
invokesuper exit reasons: 
    (all relevant counters are zero)
getblockparamproxy exit reasons: 
    (all relevant counters are zero)
getinstancevariable exit reasons:
    (all relevant counters are zero)
setinstancevariable exit reasons:
    (all relevant counters are zero)
leave exit reasons:
    interp_return:     99,971 (100.0%)
left shift (ltlt) exit reasons: 
    (all relevant counters are zero)
invalidation reasons: 
    (all relevant counters are zero)
　　　・・・　　　省略　　　・・・
freed_code_size:                   0
yjit_alloc_size:               3,840
live_context_size:                90
live_context_count:                6
live_page_count:                   1
freed_page_count:                  0
code_gc_count:                     0
num_gc_obj_refs:                   3
object_shape_count:              227
side_exit_count:                   0
total_exit_count:             99,971
total_insns_count:         2,134,203
vm_insns_count:            1,234,464
yjit_insns_count:            899,739
ratio_in_yjit:                 42.2%
avg_len_in_yjit:                 9.0
total_exits:                    0
```

`total_insns_count:2,134,203`の全命令行に対して`yjit_insns_count:899,739`がネイティブコードで実行されており、割合は`ratio_in_yjit:42.2%`になっています。`puts`に代表されるC言語を経由しない処理の場合は、Rubyから直接機械語を実行出来る割合が高いことがわかります。

## 復習を終えて

RubyKaigiではレクサー、パーサー、Jit、オートマトンという単語がかなりの頻度で出現していたように思います。実際にハンズオン形式で触れてみてようやく、それぞれの役割や関係性が理解できるようになりました。特にRuby実行のパフォーマンスに関連したセッション内容は、Ruby、C言語、機械語のどのレイヤーに働きかけたのかが分からないとぼんやり速くなったんだなで終わってしまいます。これらはしっかり把握してからRubyKaigiに参加すべきだったと少し後悔しています。
ハンズオンを実施しながらRubyコンパイル過程を改めて復習すると、セッション中に登壇者が伝えたかったことをさらに深く理解できるようになりました。きちんと復習してからRubyKaigi中に取ったメモを振り返ると、次々と新たな気づきが見つかりそうです。

## 参考文献

https://gihyo.jp/article/2024/01/ruby3.3-lrama#ghbzLfIiOP

https://ydah.net/blog/posts/20231223/?utm_source=chatgpt.com

https://yui-knk.hatenablog.com/entry/2023/11/01/082815