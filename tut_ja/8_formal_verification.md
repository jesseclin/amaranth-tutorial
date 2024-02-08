# 近日公開予定

## バウンデッドモデルチェック

## カバレッジ

## 楽しみと利益のためのアサート、アシューム、およびカバー


```python
from amaranth.asserts import Assert, Assume, Cover
from amaranth.cli import main_parser, main_runner
from somewhere import Adder

if __name__ == "__main__":
    parser = main_parser()
    args = parser.parse_args()

    m = Module()
    m.submodules.adder = adder = Adder()

    m.d.comb += Assert(adder.out == (adder.x + adder.y)[:8])

    with m.If(adder.x == (adder.y << 1)):
        m.d.comb += Cover((adder.out > 0x00) & (adder.out < 0x40))

    main_runner(parser, args, m, ports=[] + adder.ports())
```

## Past, Rose, Fell, Stable

```python
from amaranth.asserts import Assert, Assume, Cover
from amaranth.asserts import Past, Rose, Fell, Stable
```

## sbyファイル

```ini
[tasks]
cover
bmc

[options]
bmc: mode bmc
cover: mode cover
depth 40
multiclock off

[engines]
smtbmc boolector

[script]
read_ilang toplevel.il
prep -top top

[files]
toplevel.il
```

## Running formal verification

出力を生成し、次にSymbiyosysを実行してください:

```sh
python3 adder_test.py generate -t il > toplevel.il
sby -f <file>.sby
```
