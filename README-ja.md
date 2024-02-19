# Djot

Djot は軽量のマークアップ構文です。この機能のほとんどは[commonmark](https://spec.commonmark.org)から派生していますが、commonmarkにおいて構文が複雑で効率的な解析が難しい点をいくつか修正しています。また定義リスト、脚注、表、いくつかの新しい種類のインライン・フォーマット（挿入、削除、ハイライト、上付き、下付き）、数学、スマート句読点、どの要素にも適用できる属性、ブロックレベル、インラインレベル、未加工コンテンツ用の汎用コンテナをサポートしており、commonmarkよりもはるかに機能が充実しています。

このプロジェクトは、私(jgm)がエッセイ「[Beyond Markdown](https://johnmacfarlane.net/beyond-markdown.html)」で提案したアイデアのいくつかを実装する試みとして始まりました。(以下の「[理論的根拠](#理論的根拠)」を参照してください。)

このリポジトリには、[構文の説明](https://htmlpreview.github.io/?https://github.com/jgm/djot/blob/master/doc/syntax.html)、[チートシート](https://github.com/dai/djot/blob/main/doc/cheatsheet.md)、およびdjotとMarkdownの主な違いの概要を説明するMarkdownユーザー向けの[クイック スタート](https://github.com/dai/djot/blob/main/doc/quickstart-for-markdown-users.md)が含まれています。

ローカルにインストールしなくても、[djotプレイグラウンド](https://djot.net/playground/)でdjotを試すことができます。


## 理論的根拠

設計目標は次のとおりです:

1. djotマークアップはバックトラッキングなしで線形時間にて解析できる必要があります。

2. インライン要素の解析は"ローカル"である必要があり、後で定義される参照に依存しません。これはcommonmarkには当てはまらないことです: `[foo][bar]`で`[foo]`の後はリンクを含むテキスト"bar"または"[foo][bar]"、あるいはテキスト"foo"を含むリンク、"foo"の後に"[bar]"が続く場合があります。これは、参照`[foo]`と`[bar]`がドキュメント内の別の場所 (おそらく後で) で定義されているかどうかに応じて異なります。この非局所性により、正確な構文の強調表示はほぼ不可能になります。

3. 強調のルールはもっとシンプルであるべきです。commonmarkでは二重文字が強い強調に使用されているため多くの潜在的な曖昧さが生じますが、これらは17個のルールの気の遠くなるようなリストによって解決されます。これらのルールの優れたメンタルモデルを形成するのは困難です。ほとんどの場合、彼らは人間が最も自然に解釈する方法で物事を解釈しますが---常にそうとは限りません。

4. 表現上の盲点は避けるべきです。`a<em>?</em>b`commonmarkでは、フランキング ルールによって最初のアスタリスクが`a*?*b`右フランキングとして分類されるため、HTMLを生成する場合はアンラッキーです。これを回避する方法はありますが、醜いです(`a`の代わりに数値エンティティを使用します)。djotでは、この種の表現上の死角があってはなりません。

5. どのコンテンツがリスト項目に属するかのルールは単純である必要があります。commonmarkでは、リスト項目の下のコンテンツは、リスト マーカーの後の最初のスペース以外のコンテンツまでインデントする必要があります(リスト項目がインデントされたコードで始まる場合は、マーカーの後ろに5つのスペース)。インデントされたコンテンツが十分にインデントされておらず、リスト項目に含まれない場合、多くの人が混乱します。

6. パーサーは、Unicode文字クラス、HTML グ、またはエンティティを認識したり、Unicodeの大文字と小文字の折りたたみを実行したりすることを強制されるべきではありません。これにより、さらに複雑さが増します。

7. 構文はハード改行に適している必要があります。段落をハード改行することにより、異なる解釈が生じてはなりません(たとえば、ピリオドが続く数字が行頭になる場合など)。(多くの人が、なぜハード改行するのか疑問に思うと思います。
答え: HTMLへの変換や、長い行をソフト改行する特別なエディターを使用せずに、文書をそのまま読めるようにするためです。ソースの可読性は1つであったことを思い出してください。MarkdownとCommonmarkの主な目的の一部です)。

8. 構文は次の意味で均一に構成されている必要があります。つまり、一連の行がリスト項目またはブロック引用符の外側で特定の意味を持つ場合、その内側でも同じ意味を持つ必要があります。この原則は[commonmark spec](https://spec.commonmark.org/0.30/#principle-of-uniformity)で明確に示されていますが、この仕様は完全に準拠しているわけではありません(commonmark/commonmark-spec#634を参照)。

9. 任意の属性を任意の要素に付加できる必要があります。

10. テキスト、インライン コンテンツ、およびブロック レベルのコンテンツには、任意の属性を適用できる汎用コンテナーが必要です。これにより、AST 変換を使用した拡張性が可能になります。

11. これらの目標に合わせて、構文は可能な限り単純にする必要があります。したがってたとえば、2つの異なるスタイルの見出しやコード ブロックは必要ありません。

これらの目標が次の決定の動機となりました。

- 目標7のため、ブロック レベルの要素は段落(または見出し)を中断することはできません。したがって、djotにおいて以下は1つの段落であり、
(commonmarkが考えるように)段落の後に順序付きリストが続き、その後にブロック引用符が続き、その後にセクションの見出しが続くということではありません。

```
My favorite number is probably the number
1. It's the smallest natural number that is
> 0. With pencils, though, I prefer a
# 2.
```

Commonmarkでは、`1. `以外のマーカーで始まるリストが段落を中断することを禁止するなど、目標7に対していくらか譲歩しています。しかし、これは構文の規則性と予測可能性を犠牲にした妥協です。一般的なルールを決めたほうが良いでしょう。

最後の決定の意味は、「タイトな」リストは引き続き可能ですが (項目間に空白行を入れない)、サブリストの前には常に空白行を置く必要があるということです。したがって、代わりに
  Commonmark does make some concessions to goal 7, by forbidding
  lists beginning with markers other than `1.` to interrupt paragraphs.
  But this is a compromise and a sacrifice of regularity and
  predictability in the syntax. Better just to have a general rule.

- An implication of the last decision is that, although "tight"
  lists are still possible (without blank lines between items),
  a *sublist* must always be preceded by a blank line. Thus,
  instead of

  ```
  - Fruits
    - apple
    - orange
  ```

  you must write

  ```
  - Fruits

    - apple
    - orange
  ```

  (This blank line doesn't count against "tightness.")
  reStructuredText makes the same design decision.

- Also to promote goal 7, we allow headings to "lazily"
  span multiple lines:

  ```
  ## My excessively long section heading is too
  long to fit on one line.
  ``` 

  While we're at it, we'll simplify by removing setext-style
  (underlined) headings. We don't really need two heading
  syntaxes (goal 11).

- To meet goal 5, we have a very simple rule: anything that is
  indented beyond the start of the list marker belongs in
  the list item.

  ```
  1. list item

    > block quote inside item 1

  2. second item
  ```

  In commonmark, this would be parsed as two separate lists with
  a block quote between them, because the block quote is not
  indented far enough. What kept us from using this simple rule
  in commonmark was indented code blocks. If list items are
  going to contain an indented code block, we need to know at
  what column to start counting the indentation, so we fixed on
  the column that makes the list look best (the first column of
  non-space content after the marker):

  ```
  1.  A commonmark list item with an indented code block in it.

          code!
  ```

  In djot, we just get rid of indented code blocks. Most people
  prefer fenced code blocks anyway, and we don't need two
  different ways of writing code blocks (goal 11).

- To meet goal 6 and to avoid the complex rules commonmark
  adopted for handling raw HTML, we simply do not allow raw HTML,
  except in explicitly marked contexts, e.g.
  `` `<a id="foo">`{=html} `` or

  ````
  ``` =html
  <table>
  <tr><td>foo</td></tr>
  </table>
  ```
  ````

  Unlike Markdown, djot is not HTML-centric. Djot documents
  might be rendered to a variety of different formats, so although
  we want to provide the flexibility to include raw content in
  any output format, there is no reason to privilege HTML. For
  similar reasons we do not interpret HTML entities, as
  commonmark does.

- To meet goal 2, we make reference link parsing local.
  Anything that looks like `[foo][bar]` or `[foo][]` gets
  treated as a reference link, regardless of whether `[foo]`
  is defined later in the document. A corollary is that we
  must get rid of shortcut link syntax, with just a single
  bracket pair, `[like this]`. It must always be clear what is a
  link without needing to know the surrounding context.

- In support of goal 6, reference links are no longer
  case-insensitive. Supporting this beyond an ASCII context
  would require building in unicode case folding to every
  implementation, and it doesn't seem necessary.

- A space or newline is required after `>` in block quotes,
  to avoid the violations of the principle of uniformity 
  noted in goal 8:

  ```
  >This is not a
  >block quote in djot.
  ```

- To meet goal 3, we avoid using doubled characters for
  strong emphasis. Instead, we use `_` for emphasis and `*` for
  strong emphasis. Emphasis can begin with one of these
  characters, as long as it is not followed by a space,
  and will end when a similar character is encountered,
  as long as it is not preceded by a space and some
  different characters have occurred in between. In the case
  of overlap, the first one to be closed takes precedence.
  (This simple rule also avoids the need we had in commonmark to
  determine unicode character classes---goal 6.)

- Taken just by itself, this last change would introduce a
  number of expressive blind spots. For example, given the
  simple rule,
  ```
  _(_foo_)_
  ```
  parses as
  ``` html
  <em>(</em>foo<em>)</em>
  ```
  rather than
  ``` html
  <em>(<em>foo</em>)</em>
  ```
  If you want the latter
  interpretation, djot allows you to use the syntax
  ```
  _({_foo_})_
  ```
  The `{_` is a `_` that can only open emphasis, and the `_}` is
  a `_` that can only close emphasis. The same can be done with
  `*` or any other inline formatting marker that is ambiguous
  between an opener and closer. These curly braces are
  *required* for certain inline markup, e.g. `{=highlighting=}`,
  `{+insert+}`, and `{-delete-}`, since the characters `=`, `+`,
  and `-` are found often in ordinary text.

- In support of goal 1, code span parsing does not backtrack.
  So if you open a code span and don't close it, it extends to
  the end of the paragraph. That is similar to the way fenced
  code blocks work in commonmark.

  ```
  This is `inline code.
  ```

- In support of goal 9, a generic attribute syntax is
  introduced. Attributes can be attached to any block-level
  element by putting them on the line before it, and to any
  inline-level element by putting them directly after it.

  ```
  {#introduction}
  This is the introductory paragraph, with
  an identifier `introduction`.

             {.important color="blue" #heading}
  ## heading

  The word *atelier*{weight="600"} is French.
  ```

- Since we are going to have generic attributes, we no longer
  support quoted titles in links. One can add a title
  attribute if needed, but this isn't very common, so we don't
  need a special syntax for it:

  ```
  [Link text](url){title="Click me!"}
  ```

- Fenced divs and bracketed spans are introduced in order to
  allow attributes to be attached to arbitrary sequences of
  block-level or inline-level elements. For example,

  ```
  {#warning .sidebar}
  ::: Warning
  This is a warning.
  Here is a word in [français]{lang=fr}.
  :::
  ```

## Syntax

For a full syntax reference, see the
[syntax description](https://htmlpreview.github.io/?https://github.com/jgm/djot/blob/master/doc/syntax.html).

A vim syntax highlighting definition for djot is provided in
`editors/vim/`.

## Implementations

There are currently six djot implementations:

- [djot.js (JavaScript/TypeScript)](https://github.com/jgm/djot.js)
- [djot.lua (Lua)](https://github.com/jgm/djot.lua)
- [djota (Prolog)](https://github.com/aarroyoc/djota)
- [jotdown (Rust)](https://github.com/hellux/jotdown)
- [godjot (Go)](https://github.com/sivukhin/godjot)
- [djoths (Haskell)](https://github.com/jgm/djoths)

djot.lua was the original reference implementation, but
current development is focused on djot.js, and it is possible
that djot.lua will not be kept up to date with the latest syntax
changes.

## File extension

The extension `.dj` may be used to indicate that the contents
of a file are djot-formatted text.

## License

The code and documentation are released under the MIT license.

