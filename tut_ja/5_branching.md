# 分岐

> 注: この章は単なる短い紹介です。Amaranth言語ガイドの[制御構造](https://amaranth-lang.org/docs/amaranth/latest/lang.html#control-structures)章では詳細に説明されています。

標準のHDLと同様に、条件付きの論理を使用することができます。

## If-Elif-Else

通常のPythonの `if-elif-else` 文をステートメントとして使用することはできません。代わりに、次のような `with` 構文を使用します:

```python
with m.If(condition1):
    m.d.comb += statements1
with m.Elif(condition2):
    m.d.comb += statements2
with m.Else():
    m.d.comb += statements3
```

通常のPythonの`if-elif-else`を使用すると、その条件は論理の*生成*中に評価されますが、論理自体ではありません。これは、異なる論理を生成するためにフラグを使用したい場合に便利であり、これは`elaborate()`に渡される`platform`文字列を使用する良い方法です。例えば:

```python
if (platform == "this"):
    m.d.comb += statement1
else:
    m.d.comb += statement2
```

もし`platform`が`"this"`であれば、生成されたハードウェアには`statement1`だけが現れます。そうでなければ、`statement2`だけが現れます。

### 条件

`If-Elif-Else`の条件は比較です。たとえば、`a == 1`や`(a >= b) & (a <= c)`のようなものです。後者の例では、各項の周りに括弧を使用しました。各比較は、本質的には1ビットのシグナルになり、`&`は論理的な`and`ではなく、*ビット単位の演算子*です。疑わしい場合は、単に括弧を使用してください。

`a`が1ビット以上のシグナルであり、それを条件として使用する場合、例えば`with m.If(a):`のように、条件は`a`のどのビットでも1であれば真になります。

## Switch-Case-Default

以下の `with` 構文を使用して、標準のHDLと同様に `Switch-Case-Default` を使用できます:

```python
with m.Switch(expression):
    with m.Case(value1):
        statements1
    with m.Case(value2):
        statements2
    with m.Default():
        statements3
```

1つの `Case` に複数の値を持たせることもできます:

```python
with m.Switch(expression):
    with m.Case(value1, value2):
        statements1
    with m.Case(value3, value4, value5, value6):
        statements2
```

デフォルトの場合に信号を変更したくない場合は、`Default` ケースを省略することができます:

```python
m.d.comb += x.eq(1)
with m.Switch(y):
    with m.Case(0, 1, 2):
        m.d.comb += x.eq(2)
```

上記の例では、`y` が0、1、または2の場合、`x` には2が割り当てられます。それ以外の場合、`x` は値1のままです。ステートメントのオーバーライドに関するセクションを思い出してください。

## ビットパターン

`Case` でパターンを一致させる方法は、Pythonのバイナリ数字の文字列を使用することです。例えば、`"0011101011"` です。ドントケアビットはダッシュを使用して指定され、例えば `"001---101"` です。文字列内のビット数は、比較される式のビット数とまったく一致する必要があります。

`Value` には `matches` というメソッドがあり、`Switch-Case` のように機能します。例:

```python
with m.If(a.matches("11---", 3, b)):
    statement1
```

以下と同じです:

```python
with m.Switch(a):
    with m.Case("11---", 3, b):
        statement1
```
