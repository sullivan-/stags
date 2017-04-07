# stags: Scala tags generator

## Installation

```bash
git clone git@github.com:pjrt/stags.git
cd stags
sbt install
```

This will install the cli in `~/.local/bin` (while the library is installed
in `~/.local/lib`). Make sure `~/.local/bin` is in your `$PATH`.

## Usage

```bash
stags ./
```

This will fetch all Scala files under the current directory. The tags file
will be generated in `./tags`. To place the tags file somewhere else, do:

```bash
stags ./ -o path/to/tags
```

## Features

The two main differences between stags and a general ctags generator like
[Universal CTags](https://github.com/universal-ctags/ctags) is its ability to
understand Scala code, and generating appropriate static tags for each filed,
and the ability to produce qualified tags.

### Understanding Scala intricacies and static tagging them

What are static tags? Static tags mean tags for "static functions". In the C
world this meant functions that could only be used in the file where they
were defined; you could think of them as "private". Vim understand static tags
and will match them first before anything else.

Static tags lend themselves nicely to private field and functions, so `stags`
does so except while taking care of some odd Scala intricacies.

If a def/val/class/ect is `private`, then it is static. This means that if it
is `private[X]` then we check if `X` is simply the enclosing parent. However,
if X != enclosing, then we mark it as non-static. For example

```scala
object X {
  private[X] def f = …
}

object K {
  private[somepackage] def g = …
}
```

In this example, `f` would be static, but `g` isn't because `g` might be
accessed from outside the file.

Other cases that are marked as static are:

* field constructors in classes (ie: `class X(a: Int, b: String, c: Boolean)`)
  * But non-static tags for the *first* parameter group of `case` classes (since those are accessible by default)
    * `case class X(a: Int)(b: Int)` <- `a` will be non-static, while `b` is
  * If any of them are marked as "private", then it is not static
* single field in an implicit class
  * `implicit class X(val x: Int)` <- `x` is static
  * this is done because chances are that `x` will never be accessed anywhere but this file

### Qualified tags

A common pattern found when programming is the use of qualified functions in
order to avoid conflicts. This means doing the following:

```scala
import org.example.SomeObject
import org.example.OtherObject

SomeObject.foo(...)
OtherObject.foo(...)
```

In order to differentite betweent the two, `stags` generates tags for all
fields along with an extra tag that combines their parent with the tag itself.

So the follow code would produce three tags: `Example`, `foo` and `Example.foo`:

```scala
object Example {
  def foo(...)
}
```

Now vim won't understand such a tag right off the bat. The following
modification is required:

TODO:pjrt This does not work correctly when trying to access the parent of the
tag. IE: when trying to fetch `Example` in `Example.foo`. It will jump to `foo`
```viml
function! TagJumpDot()
  let l:orig_keyword = &iskeyword
  set iskeyword+=\.
  let l:word = expand("<cword>")
  let &iskeyword = l:orig_keyword
  execute "ta " . l:word
endfunction

nnoremap <silent> <C-]> :<C-u>call TagJumpDot()<CR>
```
