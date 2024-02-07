# 値

> 注意：この章は単なる短い紹介にすぎません。Amaranth言語ガイドの[Values](https://amaranth-lang.org/docs/amaranth/latest/lang.html#values)、[Constants](https://amaranth-lang.org/docs/amaranth/latest/lang.html#constants)、[Shapes](https://amaranth-lang.org/docs/amaranth/latest/lang.html#shapes)、および[Signals](https://amaranth-lang.org/docs/amaranth/latest/lang.html#signals)の章では、さらに詳細に説明されています。

`Value`は基本的な型ですが、一般的には直接`Value`を使用しません。代わりに、`Const`や`Signal`などのサブタイプを使用します。主に`Signal`を使用しますが、まずは`Const`を見て、定数を指定する方法の概要を把握しましょう。その後で`Signal`の方が理解しやすくなります。

## Const

`Const`は、指定されたビット数を持つ変更されないリテラル値です。

## Shapes: 幅と符号

Amaranthは値がいくつのビットを持っているかをどのように知るのでしょうか？単純に整数から `Const` を構築する場合、例えば `a = Const(10)` とすると、整数に合わせて必要なだけのビットが得られます。この例では、4ビットです。しかし、幅と符号（*shape*）を追加することで、必要なビット数を指定できます： `a = Const(10, unsigned(16))` とすると、16ビットの符号なしの値が得られます。

値が符号付きであることを指定でき、これは大小比較などに影響します。`a = Const(-10)` は自動的に符号付きの値を作成します。ただし、`a = Const(10, signed(16))` は明示的に16ビットの符号付きの値を作成します。

符号付きの値は2の補数値です。`Const(-10)` は、2の補数で `-10` を表すための*5ビットの符号付きの値*を作成します。

`Value`の形状と符号を取得し、また文字列表現を表示することもできます：

```python
>>> from amaranth import *
>>> a = Const(10)
>>> a.shape()
unsigned(4)
>>> a.width
4
>>> a.signed
False
>>> a.shape().width
4
>>> a.shape().signed
False
>>> a
(const 4'd10)
```

```python
>>> from amaranth import *
>>> a = Const(-10)
>>> a.shape()
signed(5)
>>> a.width
5
>>> a.signed
True
>>> a
(const 5'sd-10)
```

便利なことに、`len()`は値のサイズを取得します：

```python
>>> from amaranth import *
>>> a = Const(-10)
>>> len(a)
5
```

一般的には、`Const`は `Const(<value>, <shape>)` を使用して作成できます。すでに `signed(n)` と `unsigned(n)` の形状を見ています。

## Shapes: 範囲

`Const(10, unsigned(16))`と同様に、サイズが16ビットの符号なし整数を保持できる定数が作成されます。`Const(10, range(51))`は、0から50までの範囲の整数を保持できる十分な大きさの定数を作成します。Pythonでは `range(51)` です。

もし範囲に負の数が含まれている場合、その定数は符号付きになります： `Const(3, range(-5, 11))` は、-5から+10までの値を保持できるサイズの定数であり、これにより、2の補数の5ビット信号となります:

```python
>>> from amaranth import *
>>> x = Const(3, range(-5, 11))
>>> x.shape()
signed(5)
```
### Shapes: 列挙型

すべての整数値を持つPythonの列挙型(enums)が与えられた場合、それを定数に変換することができます:

```python
from enum import Enum, unique

@unique
class Func(Enum):
    NONE = 0
    ADD = 1
    SUB = 2
    MUL = 3
    DIV = 4

...

>>> x = Const(2, Func)
>>> x.shape()
unsigned(3)
>>> x
(const 3'd2)
```

これは、列挙型の最小値と最大値を見つけ、それを定数の範囲として使用するのと同等です。上記の例では、これは `Const(2, range(0, 5))` と同じです。

また、列挙型の整数値の代わりに列挙型の値を使用できるため、より使いやすいかもしれません:

```python
>>> x = Value.cast(Func.SUB)
>>> x.shape()
unsigned(3)
>>> x
(const 3'd2)
```
列挙型の列挙値は、`Const`が使用される場所ならどこでも使用できるため、`Func.SUB`は`Const(2, range(0, 5))`や`Const(2, Func)`、`Value.cast(Func.SUB)`と同等です。

## 信号

`Signal（シグナル）`とは値が変化するものです。信号はフリップフロップまたはワイヤーとして実装されることがあります。バックエンドは、それらがどのように使用されているかに基づいて判断します。これは、Verilogでの`wire`、`reg`、または`logic`と同等です。

`Const`と同様に、`Signal（シグナル）`にも形状があります：幅と符号付きです。たとえば、16ビットの符号なし`Signal`を作成するには、`a = Signal(16)`または`a = Signal(unsigned(16))`とします。符号付き16ビットの`Signal`を指定するには、`a = Signal(signed(16))`とします。これにより、-0x8000から+0x7FFFまでの値が表されます。

形状をまったく指定しない場合、つまり `a = Signal()` とすると、そのような `Signal（シグナル）` の形状は1ビットであり、符号なしです。

再び、`Const`と同様に、`Signal（シグナル）` の形状を取得できます:

```python
>>> from amaranth import *
>>> a = Signal(signed(4))
>>> a.shape()
signed(4)
```
また、`signed(4)`は、-8から+7までの任意の値を表す4ビットの2の補数信号を提供します。-16から+15を表すために *サインを含む* 4ビットの`Signal（シグナル）`を作成したかった場合は、当然、`signed(5)`を使用する必要があります。

### 範囲からのシグナル

`Signal(range(11))`は、0から10までの任意の整数を含む十分に大きな符号なし信号を作成します。Pythonではこれは`range(11)`です。`Const`と同様に、範囲に負の数が含まれている場合、結果として得られるシグナルは符号付きになります:

```python
>>> from amaranth import *
>>> x = Signal(range(-5, 11))
>>> x.shape()
signed(5)
```

### 列挙からのシグナル

上記のPython列挙が与えられた場合、そのような値を保持するためのシグナルを作成することができます:

```python
from enum import Enum, unique

@unique
class Func(Enum):
    NONE = 0
    ADD = 1
    SUB = 2
    MUL = 3
    DIV = 4

...

>>> x = Signal(Func)
>>> x.shape()
unsigned(3)
```

再び、`Const`と同様に、これは列挙の最小値と最大値を見つけ、それをシグナルの範囲として使用するのと同等です。上記の例では、これは`Signal(range(0, 5))`と同じです。

### シグナル名

シグナルの名前は、デフォルトでそれに割り当てる変数の名前と同じです:

```python
>>> from amaranth import *
>>> x = Signal(unsigned(16))
>>> x.name
'x'
```

名前は、そのコンストラクタの`name`という名前の引数を介して明示的に指定することができます。

```python
>>> from amaranth import *
>>> x = Signal(unsigned(16), name="addr")
>>> x.name
'addr'
```

エラボレート時間に、名前が既存の名前と衝突した場合、接頭辞や接尾辞を使用して新しい名前が選択されます。