# シグナルの分割と結合

> 注: この章は単なる短い紹介です。Amaranth言語ガイドの[ビットシーケンス演算子](https://amaranth-lang.org/docs/amaranth/latest/lang.html#bit-sequence-operators)章では詳細に説明されています。

## シグナルのスライシング

配列のインデックス演算子を使用して、シグナルからビットを抽出できます。例えば、16ビットのシグナル`s`がある場合、最下位ビットは`s[0]`で取得し、最上位ビットは`s[15]`で取得できます。

```python
>>> x = Signal(16)
>>> x.shape()
Shape(width=16, signed=False)
>>> x[15].shape()
Shape(width=1, signed=False)
```

```python
>>> x = Signal(signed(16))
>>> x.shape()
Shape(width=16, signed=True)
>>> x[15].shape()
Shape(width=1, signed=False)
```

Pythonの配列と同様に、シグナルのビットは常に一定の順序で並んでいます。これは、HDLでのインデックス付けに慣れている人々にとって混乱の原因となることがあります。例えば、`s[7:0]`はシグナルの下位8ビットを抽出しているように見えますが、Pythonではこれが機能しないため、*Pythonでプログラミングしているので*正しい方法ではありません。シグナルの下位8ビットを抽出する正しい方法は`s[0:8]`または`s[:8]`です。これは、Python配列の最初の8つの要素を抽出する方法と同じです。要するに、シグナルの*最初の* Nビットは、そのシグナルのN *下位*ビットとして扱われます。

Pythonであるため、負のスライスインデックスは末尾からのオフセットです。したがって、シグナルから最上位ビット（"最後のビット"）を取得する方法は単に`x[-1]`です。

ストライドを使用することもできます。`x[0:8:2]`は単に`x`のビット0、2、4、および6です。

シグナルからビットを取り出すと常に符号なしのシグナルが結果として得られることに注意してください。

```python
>>> x = Signal(signed(16))
>>> x.shape()
Shape(width=16, signed=True)
>>> x[:8].shape()
Shape(width=8, signed=False)
```

シグナルの一部に割り当てることもできます:

```python
m.d.comb += x[:8].eq(y)
```

### ヒント: 比較時にスライスを使用する

このような状況では:

```python
a = Signal(unsigned(16))
b = Signal(unsigned(16))
c = Signal(unsigned(16))

m.d.comb += c.eq(a+b)
```

`a+b`がオーバーフローした場合、`c`は結果の下位16ビットになると期待しています。しかし、以下を考えてみてください:

```python
a = Signal(unsigned(16))
b = Signal(unsigned(16))
z = Signal()

m.d.comb += z.eq((a+b) == 0)
```

ここでは、`a+b`は17ビットの信号でなければなりません。Pythonでは、整数の幅は必要に応じて拡張されるためです。そのため、16ビットのオーバーフローは17ビットのオーバーフローではありません。そのため、`a=0xFFFF`、`b=1`などの値では、この比較は失敗します。加算の結果は`0x10000`になり、明らかに0ではありません。

したがって、結果をスライスする際には注意してください:

```python
m.d.comb += z.eq((a+b)[:16] == 0)
```

または、中間信号を使用するだけです:

```python
tmp = Signal(unsigned(16))

m.d.comb += tmp.eq(a+b)
m.d.comb += z.eq(tmp == 0)
```

これは、特に符号なしと符号付きの信号を組み合わせる場合に特に厄介になります。考えてみてください:

```python
ptr = Signal(unsigned(16))
addr = Signal(unsigned(16))
offset = Signal(signed(5)) # -16 to +15

m.d.comb += ptr.eq(addr + offset)
```

ここでは、`ptr`が16ビットの値であることを期待しています。しかし、ここで何が起こるでしょうか?

```python
y = Signal()

m.d.comb += y.eq((addr + offset) == 0xFFFF)
```

仮に`addr`が0で`offset`が-1であるとします。この比較は機能するでしょうか？そう思われるかもしれませんが、実際は機能しません！`addr`は符号なしの16ビット値で、0から0xFFFFまでの範囲を持ち、`offset`は2の補数で、-16から+15の範囲を持つ5ビット値です。これらを加算すると、-0x10から0x1000Eまでの範囲の値を受け入れるために、符号付きでかつ十分に広い信号が必要です。これには、2の補数で18ビットの値が必要です。そのうち、大きさにはおおよそ17ビット、符号には1ビットが必要です。

したがって、この場合の`addr+offset`の結果は-1であり、これは2の補数で18ビットの場合、`0x3FFFF`となります。これをスライスすると`0xFFFF`になりますが、スライスを行わない場合、比較は機能しません。

## 信号の連結

`Cat`を使用して他の信号から新しい信号を作成できます:

```python
m.d.comb += x.eq(Cat(a, b, ...))
```

これにより、指定された信号が*最初の要素が最後に*連結されます。これは重要です：上記の例では、`a`が`x`の最も下位ビットになることがいく分かるかもしれません。つまり、`a`と`b`の連結は`ab`ではなく`ba`です。

これで、16ビット信号のバイトを簡単に交換できます:

```python
m.d.sync += x.eq(Cat(x[8:], x[:8]))
```

`Cat`にも代入できるため、バイトの交換は次のように行うこともできます:

```python
m.d.sync += Cat(x[8:], x[:8]).eq(x)
```

## 信号の複製

`Cat(x, x)` を使って信号を複製することができます。しかし、`Repl(x, 2)` を使っても同様に信号を複製できます。

`Cat` と `Repl` を組み合わせると、例えば値を符号拡張することができます:

```python
uint16 = Signal(unsigned(16)) # はい、*un*signedに注意してください。
uint32 = Signal(unsigned(32))

m.d.comb += uint32.eq(Cat(uint16, Repl(uint16[15], 16)))
```

もちろん、同じことは適切なシグナルタイプを使用するだけで実行できます:

```python
int16 = Signal(signed(16))
int32 = Signal(signed(32))

m.d.comb += int32.eq(int16)
```

生成されたコードは適切な動作をします。

## 配列

次のようにして、シグナルの配列を作成できます:

```python
# これらすべては、16ビット要素を3つ持つ配列を作成します:

# a、b、cから配列を作成します:
a = Signal(unsigned(16))
b = Signal(unsigned(16))
c = Signal(unsigned(16))

abc = Array([a, b, c])

# 16ビットの信号の配列を作成します:
x = Array([Signal(unsigned(16)), Signal(unsigned(16)), Signal(unsigned(16))])

# Pythonのリスト内包表記を利用して、16ビットの信号の配列も作成します:
y = Array([Signal(unsigned(16)) for _ in range(3)])
```

多次元配列も作成できます:

```python
# 16ビット信号の3行5列の配列を作成します:
yy = Array([Array([Signal(unsigned(16)) for _ in range(5)]) for _ in range(3)])
```

定数を使用して配列にインデックスを付けることができます:

```python
z = y[2]
```

もしインデックスが範囲外の場合、これは「エラボレート時間」エラーを引き起こします。

しかし、別のシグナルでインデックスを行うこともできます:

```python
i = Signal(unsigned(16))

z = y[i]
```

もちろん、エラボレーション中にはエラーが発生しません。実際の結果は、シミュレータやHDLコンパイラに依存します。アクセスが無効にならないようにできる限り確認することが最善です。有効な範囲を持つインデックスを宣言する方法の1つです:

```python
y = Array([Signal(unsigned(16)) for _ in range(5)])
i = Signal.range(5)

z = y[i]
```

もちろん、`i`が3ビットの信号であるため、5、6、または7になることを防ぐものは何もありません。これにより予期しない結果が生じる可能性があります。

別の方法は、単純に無効な値を処理することです:

```python
y = Array([Signal(unsigned(16)) for _ in range(5)])
i = Signal.range(5)

z = y[i % 4]
```

これはあまり良い方法ではありません。なぜなら、依然として予期しない結果を引き起こす可能性があるからです。

最終的には、`i`が有効な値のみを含むことを形式的に検証する必要があります。後のセクションで形式的検証について詳しく説明します。

## Record

`Record`は信号の束です。`Record`を定義するには、まず`Layout`を定義する必要があります。

### Layouts

```python
from amaranth.hdl.rec import *

class MyLayout(Layout):
    def __init__(self):
        super().__init__([
            (<signal-name>, <shape|layout> [, <direction>]),
            (<signal-name>, <shape|layout> [, <direction>]),
            ...
        ])
```

Here is an example of a bus with 8 data bits, 16 address bits, and some control signals:

```python
class BusLayout(Layout):
    def __init__(self):
        super().__init__([
            ("data", unsigned(8)),
            ("addr", unsigned(16)),
            ("wr", 1),
            ("en", 1),
        ])
```

A signal in a layout can have its shape be a layout:

```python
class DataBusLayout(Layout):
    def __init__(self):
        super().__init__([
            ("data", unsigned(8)),
        ])

class AddrBusLayout(Layout):
    def __init__(self):
        super().__init__([
            ("addr", unsigned(16)),
        ])

class AllBusLayout(Layout):
    def __init__(self):
        super().__init__([
            ("addr_bus", AddrBusLayout()),
            ("data_bus", DataBusLayout()),
        ])
```

### Laying out a Record with a Layout

Once a `Layout` is defined, you can define a `Record` using that `Layout`, and use it as a signal:

```python
class Bus(Record):
    def __init__(self):
        super().__init__(BusLayout())

...

# Later, in a Module:
    self.bus = Bus()
    m.d.comb += self.bus.data.eq(0xFF)
    m.d.sync += self.bus.wr.eq(0)

# You can even operate on the entire record:
    self.bus2 = Bus()
    m.d.comb += self.bus2.eq(self.bus)
```

### Directions and connecting records

It is often advantageous to define signals so that the zero value means either invalid or inactive. That way, you can have many of those signals and logical-or them together. So for example, you might have three modules, each of which output a one-bit `write` signal, but only one module will write at a time. Then if your `write` signal is active high (so zero means no write), you can simply logical-or the `write` signals from each module together to get a master `write` signal.

As another example, each module could output 8 bits of data, but only one module at a time would send data to the data bus. In this case, if a module is inactive, it should output 0 on its `data` port. The value of the data bus is then just the values of all modules' `data` ports, logical-ored together.

This method of "connecting" signals together is called *fan-in*. If the direction of each signal in a record's layout is `DIR_FANIN`, then you can connect several records to a "master" record like this:

```python
    self.master_record = Bus()
    m.d.comb += self.master_record.connect(bus1, bus2, bus3, ...)
```

The `connect` method on a record returns an array of statements which logical-ors each signal together. The exact same thing could be accomplished "manually" by operating on the entire record:

```python
    self.master_record = Bus()
    m.d.comb += self.master_record.eq(bus1 | bus2 | bus3 | ...)
```

The disadvantage is that `connect` can connect *parts* of records, if the field names match. In this sense, the "subordinate" records must have every signal that the "master" record has. That is, the "subordinate" records can have extra signals, but the "master" record must not.

*Fan-out* is where each subordinate record gets a copy of the master record. If the direction of each signal in a record's layout is `DIR_FANOUT`, then you can connect several records to a "master" record like this:

```python
    self.master_record = Bus()
    m.d.comb += self.master_record.connect(bus1, bus2, bus3, ...)
```

The syntax is exactly the same, but the direction is different, from master record to each subordinate record. Again, you could do this "manually":

```python
    self.master_record = Bus()
    m.d.comb += [
        bus1.eq(self.master_record),
        bus2.eq(self.master_record),
        bus3.eq(self.master_record),
        ...
    ]
```

But this is longer, and also doesn't handle when the master record has extra signals not in the subordinate records.
