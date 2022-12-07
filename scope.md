---
htmltitle: |
  <code>\UseName</code> でもエラーが出したい！
title: \UseName でもエラーが出したい！
author: thth
date: 2022-12-08
source: https://github.com/tairahikaru/texadvent2022/blob/main/scope.md
layout: mylayout
documentclass: bxjsarticle
classoption: pandoc
toc: true
colorlinks: true
header-includes: |
  \usepackage{newunicodechar,xparse}
  \newunicodechar{⟨}{\meta}
  \NewDocumentCommand\meta{u{⟩}}{$\langle\mbox{#1}\rangle$}
  \title{\texttt{\symbol{`\\}UseName} でもエラーが出したい！}
  \renewcommand{\title}[1]{}
---

{% raw %}

これは、「[TeX & LaTeX Advent Calendar 2022](https://adventar.org/calendars/7868)」の 6 日目の記事『[`\NewDocumentCommand` さえあればいい。][本編]』を投稿したあとに思いついた話です。
この記事単体でも読めるようになっています。

これ自体は 8 日目の記事です。
昨日は [CareleSmith9](http://circle9tym.blog.fc2.com/) さんで、明日は [zr_tex8r](https://twitter.com/zr_tex8r) さんです。

`\newcommand`、`\newenvironment` の使い方はわかっている前提です。

この記事は [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)、コードの部分はパブリックドメイン・無保証です。

# `\UseName` とは

`\UseName{<命令名>}` とすると、`\<命令名>` に置き換えられます。
たとえば `\UseName{newcommand}` が `\newcommand` になるといった具合です。
このように命令化した `<文字列>` を `<別の命令>` に渡したいときは、`\ExpandArgs{c}<別の命令>{<文字列>}` とすると `<別の命令>\<文字列>` に置き換えられるのでした。

[先日の記事][本編]では、「このように文字列から作られた命令が未定義だった場合、エラーが出ない」という副作用があるということを説明しました。

```latex
% \Undefined は未定義
\UseName{Undefined}% エラーでない
\Undefined%          エラーでない
```

この記事ではこの挙動を改善し、「命令が未定義であった場合にはエラーを出すバージョンの `\UseName`」として `\MyUseName` を作っていきます。

# `{}` と局所化

`{}` を見ると「命令に引数を渡すときのカッコ」だと思う人が多いでしょうが、実は、それ以外の用途があるのをご存知ですか？
「引数の範囲を示す」以外の場面で使われると、どのような意味を持つのでしょう。

```latex
{} というカッコは波カッコとか中カッコとか呼ばれる。
```

![`{}` が消えた](brace.jpeg)

上のコードを実行してみると、`{}` という文字は出力されずに消えてしまいました。
実は、`{}` というカッコは、「引数の範囲を示す」以外の場面では「影響範囲を示す」という効果を持ちます。

```latex
{\Large 大きい文字}と普通の文字
```

![`\Large` の影響が `{}` で限定されてる](large.jpeg)

文字を大きくして出力する命令である `\Large` の効果が、`{}` の中だけに影響していますね。

この「影響範囲」というのは、`\Large` などの組版結果に直接影響を及ぼす命令に限りません。
`\newcommand` や `\NewDocumentCommand` などの「命令の定義を変える」ような命令の影響も、`{}` の中だけに制限されます。

```latex
\newcommand{\mycommand}{foo}
\mycommand %   -> foo
{
  \mycommand % -> foo
  \renewcommand{\mycommand}{bar}
  \mycommand % -> bar
}
\mycommand %   -> foo
```

賢明なる読者の方はお気づきでしょうが、`\UseName` や `\ExpandArgs` の副作用が影響する範囲も `{}` を使うと限定することができます。


```latex
% \Undefined は未定義
{
  \UseName{Undefined}% エラーでない
  \Undefined %         エラーでない
}
\Undefined %           エラーでる！
```

このように効果範囲を限定することを、この記事では「局所化する」ということにします。

以上を踏まえると、`\MyUseName` を実現するは以下のような手順を踏めばよいでしょう。

1. 局所化を開始する
2. `\ExpandArgs` を使って文字列を命令化する
3. 局所化を終了する
4. 命令化した文字列を使う

<div markdown="block" class="warn">

LaTeX が提供する命令には、局所化の影響を受けないものもあります。
たとえば `\setcounter` や `\stepcounter` がこれにあたります。
もちろん、`\setlength` など局所化の影響を受ける命令もあります。

```latex
\newlength{\mylength}
\skipeval{\mylength} %   -> 0.0pt
\newcounter{mycounter}%
\themycounter%           -> 0

{
  \setlength{\mylength}{1pt}
  \skipeval{\mylength} % -> 1.0pt
  \setcounter{mycounter}{1}%
  \themycounter%         -> 1
}

\skipeval{\mylength} %   -> 0.0pt
\themycounter%           -> 1
```

</div>

# 局所化をバラバラに

`\MyUseName` の実装方針が決まったので、何も考えずに `{}` を使ってとりあえずこんなコードを書いてみました：

```latex
\newcommand{\MyUseName}[1]{%
  {% 局所化の開始
    \ExpandArgs{c}\MyUseNameI{#1}%
}% \MyUseName の定義の終わり
\newcommand{\MyUseNameI}[1]{%
  }% 局所化の終了
  #1%
}% \MyUseNameI の定義の終わり
```

もちろんこれは期待通りに動作しません。
引数の中に含まれる `{}` の数は釣り合ってなければいけないので、上のコードでは以下の部分が `\MyUseName` の定義だということになります。

```latex
  {% 局所化の開始
    \ExpandArgs{c}\MyUseNameI{#1}%
}% \MyUseName の定義の終わり
\newcommand{\MyUseNameI}[1]{%
  }% 局所化の終了
  #1%
```

`{}` で局所化している限りでは、この問題は解決できなさそうです。
`{}` とは違い、引数に入れるときにバラバラにしても大丈夫で、局所化できるようなもの……、ありますよね。
それは、「環境」です。

なにもしない `Empty` 環境を作ってみて、その効果を確かめてみましょう。

```latex
\newenvironment{Empty}{}{}
\newcommand{\mycommand}{foo}
\mycommand %   -> foo
\begin{empty}
  \mycommand % -> foo
  \renewcommand{\mycommand}{bar}
  \mycommand % -> bar
\end{empty}
\mycommand %   -> foo
```

`\renewcommand` が環境の中だけに影響しているのが分かります。

`{}` の代わりに `\begin{Empty}\end{Empty}` を使うことにして、先ほどの `\MyUseName` を書き直します。

```latex
\newenvironment{Empty}{}{}
\newcommand{\MyUseName}[1]{%
  \begin{Empty}% 局所化の開始
    \ExpandArgs{c}\MyUseNameI{#1}%
}
\newcommand{\MyUseNameI}[1]{%
  \end{Empty}% 局所化の終了
  #1%
}
```

それでは試してみましょう。

```latex
\MyUseName{Undefined}
% \MyUseName を定義に置き換える
% #1 <- Undefined

\begin{Empty}% 局所化開始
\ExpandArgs{c}\MyUseNameI{Undefined}
% \ExpandArgs を置き換える
% c <- Undefined
% \Undefined が副作用によりエラー出なくなる

\MyUseNameI\Undefined
% \MyUseNameI を置き換える
% #1 <- \Undefined

\end{Empty}% 局所化終了
% \Undefined の副作用の効果なくなる
\Undefined
```

無事に、`! Undefined control sequence.` のエラーが出ましたね。
めでたしめでたし。

# 引数に渡す場合

今度は `\ExpandArgs` のように、別の命令の引数に渡す場合を考えてみます。
といっても `\MyUseName` の定義を少し変えるだけです。

```latex
\newenvironment{Empty}{}{}
\newcommand{\MyUseName}[2][]{%
  \begin{Empty}%
    \ExpandArgs{c}\MyUseNameI{#2}{#1}%
}
\newcommand{\MyUseNameI}[2]{%
  \end{Empty}%
  #2#1%
}
```

`\MyUseName` にオプション引数をつけました。
これで、`\MyUseName[<別の命令>]{文字列}` とすると `<別の命令>\<文字列>` に置き換えられるようになります。

# おわり

普通の LaTeX ユーザであれば、うっかりスペルミスをしてもエラーが出てくれる `\MyUseName` の方が使い勝手がよいのではないでしょうか。
なぜもともとの `\UseName` や `\ExpandArgs` の実装がこんなことになっているのかというと、それには TeX 言語やら expl3 やらの話が関係してきます。
TeX 言語や expl3 に手を出すつもりのない人は、諦めるか `\MyUseName` と [`\NewDocumentCommand`][本編] で頑張りましょう。

[本編]: .

{% endraw %}

