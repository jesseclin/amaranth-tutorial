# シミュレーション

モジュールをシミュレートする最良の方法は、Amaranthの `Simulator` を使用することです。

## ポートを定義する

モジュール内に `ports` 関数を定義し、モジュールのポートの配列を返します:

```python
from amaranth import Elaboratable


class YourModule(Elaboratable):
    ...
    def ports(self):
        return [self.yourmodule.p1, self.yourmodule.p2, ...]
```

## トップレベルモジュールを作成する

シミュレーションのためのトップレベルモジュールを作成します:

```python
from amaranth import Module
from amaranth.sim import Simulator, Delay, Settle
from your_module import YourModule

if __name__ == "__main__":
    m = Module()
    m.submodules.yourmodule = yourmodule = YourModule()

    sim = Simulator(m)

    def process():
        # 未定

    sim.add_process(process) # または、sim.add_sync_process(process) を参照してください。
    with sim.write_vcd("test.vcd", "test.gtkw", traces=yourmodule.ports()):
        sim.run()
```

>（これはもう当てはまらないかもしれません）現在、Amaranthにはモジュールへの入力がトレースファイルに出力されないバグがあります。これを解決するために、このような入力ごとに、`Simulator` の構築**前**の `main` に次のコードを配置してください:
>
> ```python
>     input1 = Signal(...)
>     m.d.comb += yourmodule.input1.eq(input1)
>     ...
>     sim = Simulator(m)
> ```
> あなたの `process` の中で、この入力を `yourmodule.input1` ではなく `input1` として参照してください。これにより、Amaranth は `input1` をトレースファイルに含めるようになります。

## クロックを定義する（あれば）

クロックがある場合は、`Simulator` の構築後に各クロックを追加し、クロック周期を秒単位で指定してください。たとえば、1MHz クロックのクロックドメイン `fast_clock` と、（ほぼ）1.1MHz クロックの `faster_clock` は次のようになります:

```python
   sim = Simulator(m)
   sim.add_clock(1e-6, domain="fast_clock")
   sim.add_clock(0.91e-6, domain="faster_clock")
```

`domain` を省略すると、クロック周期がデフォルトのクロックドメインである `sync` に割り当てられます。

## プロセス関数

`process` 関数は、Amaranth がシミュレーションで次に何をするかを知るために呼び出す Python ジェネレーターです。ジェネレーターであるため、`process` は実行するステートメントを `yield` しなければなりません。例えば:

```python
    def process():
        yield x.eq(0)
        yield y.eq(0xFF)
```

上記の例では、`x` を 0 に設定し、`y` を 0xFF に設定します。これらの間に実質的な遅延はありません。

Amaranthの `Value` を `yield` できます。これを使用して、さまざまな比較を行うことができます:

```python
    def process():
        yield x.eq(0)
        yield Delay(1e-6)  # causes a delay of 1 microsecond
        yield y.eq(0xFF)
        yield Settle()     # forces all combinatorial computation to happen
        got = yield yourmodule.sum
        want = yield (x+y)[:8]
        if got != want:
            print(f"Oh noes! Error! Got {got:02x}, wanted {want:02x}")
```

上記の例では、`x` が 0 に設定され、その後1マイクロ秒の遅延が発生し、`y` が 0xFF に設定されます。すべての組み合わせ論理に解決するチャンスが与えられ、最後に `yourmodule.sum` と `(x+y)[:8]` が評価され、それらが等しくない場合、診断メッセージが端末出力に送信されます。

さらに、同時に複数のプロセスを実行することもできます!

```python
    def x_process():
        yield Delay(1e-6)
        yield x.eq(0)
        yield Settle()

    def y_process():
        yield Delay(1.2e-6)
        yield y.eq(0xFF)
        yield Settle()

    sim.add_process(x_process)
    sim.add_process(y_process)
```

上記の例では、`x` は1マイクロ秒で0に設定され、`y` は1.2マイクロ秒で0xFFに設定されます。

**警告**: 同じ信号を複数のプロセスから駆動すると、両方のプロセスが信号に同時に割り当てると未定義の動作が発生する可能性があります。

### 非同期プロセス

時間に基づいて信号がいつ変化するかを正確に指定したい場合は、非同期の `process` を作成できます。このようなプロセスをシミュレータに追加するには、`add_process` を使用します:

```python
    sim.add_process(process)
```

### Synchronous processes

クロックエッジに基づいて信号の変化を指定したい場合は、同期 `process` を作成できます。このようなプロセスをシミュレータに追加するには、`add_sync_process` を使用し、それがクロックされるクロックドメインを指定します:

```python
    sim.add_sync_process(process1, domain="name1")
    sim.add_sync_process(process2, domain="name2")
```

同期プロセスから値なしで `yield` すると、そのプロセスは次のクロックエッジを待ちます。同期プロセスでは、プロセスが開始される前に常に1つのクロックエッジが発生することに注意してください。トレースを確認する際には、これを考慮してください。

また、ステートメントがクロックエッジに対していつ実行されるかを理解することも重要です。ステートメントは、常に**直前のクロックエッジの直後**に実行されます。したがって、次の例では:

```python
    def process():
        yield x.eq(0)  # step 1
        yield          # step 2
        yield x.eq(1)  # step 3
        yield          # step 4
```

プロセスが実行される前に常に1つのクロックエッジが発生します。その後、`x` が0に設定されます（ステップ1）。その後、別のクロックエッジが発生します（ステップ2）。`x` は、そのクロックエッジの直後に無限小の時間で1に設定されます（ステップ3）。その後、さらに別のクロックエッジが発生します（ステップ4）。トレースでは、信号の変化がクロックエッジの直後に表示されるわけではありません。それらはクロックエッジと一致して現れます。信号の変化がクロックエッジの直後に行われるというルールを覚えておいてください。そのように変化する出力は、実際にはそのクロックエッジの直後に変化したと考えることができます。

### パッシブとアクティブなプロセス

プロセスはパッシブまたはアクティブになります。*アクティブ* プロセスは、シミュレーターに実行する指示がなくなると、シミュレーターに終了を要求します。実際には、それがシミュレーターの終了点を制御します。すべてのアクティブなプロセスが終了すると、シミュレーションが終了します。一方、*パッシブ* プロセスは、シミュレーターに終了を要求しません。

デフォルトでは、`add_process` および `add_sync_process` で追加されたプロセスはアクティブです。プロセスは `yield Active()` または `yield Passive()` を使用してモードを変更できます。

## シミュレーションの終了

上記のように、すべてのアクティブなプロセスが終了すると、シミュレーションが終了します。これが `sim.run()` の動作です。

ただし、代わりに `sim.run_until()` を使用して、特定の時間でシミュレーションを終了できます。`run_passive` キーはデフォルトで `False` であり、すべてのアクティブなプロセスが終了した場合にもシミュレーションが終了します。この動作は、`run_passive` を `True` に設定することで変更できます。この場合、シミュレーションは指定された時間に到達するまで終了しません。たとえば、次のようにして、100マイクロ秒の間シミュレーションを実行し、アクティブなプロセスが完了しているかどうかに関係なく、停止します：

```python
    with sim.write_vcd("test.vcd", "test.gtkw", traces=yourmodule.ports()):
        sim.run_until(100e-6, run_passive=True)
```

## シミュレーションの実行と出力の表示

シミュレーションは、単純にメインモジュールを実行することで実行されます：

```sh
python3 main_module.py
```

出力は `test.vcd` ファイルと `test.gtkw` ファイルです。`gtkwave` を実行すると、出力を表示できます。`gtkwave` を `test.vcd` に対して実行すると、`gtkwave` が開いたときに表示する信号を選択する必要があります。一方、`test.gtkw` に対して実行すると、`sim.write_vcd()` の呼び出しで与えた `traces` キーに含まれる信号が表示されます。

```sh
gtkwave test.vcd
gtkwave test.gtkw
```