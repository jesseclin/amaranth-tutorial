# 近日公開予定

## サポートされているデバイス

サポートされているデバイスとツールチェーンの詳細については、[ベンダーディレクトリ](https://github.com/amaranth-lang/amaranth/tree/main/amaranth/vendor) を参照してください。

2019年12月20日現在、サポートされているデバイス:

| Device            | Platform classname        | Toolchain required |
| ----------------- | ------------------------- | ------------------ |
| Intel             | IntelPlatform             | Quartus            |
| Lattice ECP5      | LatticeECP5Platform       | Trellis            |
| Lattice ICE40     | LatticeICE40Platform      | IceStorm/iCECube2  |
| Lattice MachXO2   | LatticeMachXO2Platform    | Diamond            |
| Xilinx 7 series   | Xilinx7SeriesPlatform     | Vivado             |
| Xilinx Spartan 3A | XilinxSpartan3Or6Platform | ISE                |
| Xilinx Spartan 6  | XilinxSpartan3Or6Platform | ISE                |
| Xilinx UltraScale | XilinxUltraScalePlatform  | Vivado             |

## ボードの定義

多くのボードは [amaranth_boards](https://github.com/amaranth-lang/amaranth-boards/tree/main/amaranth_boards) で定義されています。

そこから1つをコピーして必要に応じて変更するか、上記でサポートされているデバイスプラットフォームクラスの1つからサブクラス化された新しいクラスを作成できます。

### クラスのプロパティ

* `device`: 文字列。どれを選択するかはベースプラットフォームクラスを参照してください。これは、正しいチップ用にコンパイルするためのツールチェーンに渡されるオプションに影響を与えます。
* `package`: 文字列。どれを選択するかはベースプラットフォームクラスを参照してください。これは、チップの正しいパッケージ用にコンパイルするためのツールチェーンに渡されるオプションに影響を与えます。
* `resources`: `Resource` のリスト。使用するピンの名前と、各ピンに対する構成オプションを指定します。
* `default_clk`: デフォルトのクロックドメインのクロックのリソース名。
* `default_rst`: デフォルトのクロックドメインのリセットのリソース名。
* `connectors`: オプションで、`Connector` のリスト。これがどのような目的に役立つかは明確ではありません。特定のツールチェーンと関連があるかもしれません。
  
### リソース

`Resource` は、名前、番号、およびリソースの1つ以上の構成項目を含む構造体です。ボードに `Resource` を追加することで、次の2つのことができます。

* デバイス上のピンを構成します。
* `elaborate` 関数内でプラットフォームから名前でリソースの `Pin` を要求できるようにします。そのような `Pin` には、入力用の `i` や出力用の `o` など、いくつかの `Signal` が関連付けられており、モジュールで使用できます。

例えば、このような `Resource` をプラットフォームの `resources` リストに含めることで:

```python
    Resource("abc", 0, Pins("J3", dir="i"))
```

その後、デバイス上のピン `J3` が入力として構成され、次のようにして `abc` リソースの入力 `Signal` を要求できます:

```python
    platform.request("abc").i
```

#### リソース構成項目

* `Pins`: リソースに関連付けられたスペースで区切られたピン名、方向タイプ、およびピンを横切るときに信号を自動的に反転するかどうか（例：アクティブローレベルの信号）を指定します。方向タイプは次のとおりです：
  * `i`: 入力のみ。ピンの信号は `.i` です。
  * `o`: 出力のみ。ピンの信号は `.o` です。
  * `io`: 双方向。ピンの信号は入力の場合は `.i`、出力の場合は `.o`、ピンの方向のための `.oe` があります：入力の場合は0、出力の場合は1
  * `oe`: トライステート。ピンの信号は出力の場合は `.o`、出力を有効にするための `.oe` があります：無効にする場合は0、有効にする場合は1

* `PinsN`: すべてのピンがアクティブローの場合の `Pins` の省略形。

* `DiffPairs`: 1つ以上の差動ペア（正および負のピン）のスペースで区切られたピン名を指定します。

* `Clock`: 指定された周波数（Hz）のクロックであることを示します。

* `Attrs`: プラットフォーム固有の属性、例えば選択する電圧規格など。

Lattice ICE40ボードで使用されるクロックピンの完全なリソース仕様は次のようになります:

```python
    Resource("clk", 0, Pins("J3", dir="i"), Clock(12e6), Attrs(GLOBAL=True, IO_STANDARD="SB_LVCMOS"))
```

この記述は、FPGA 上のピン `J3` に `clk` というクロック信号が存在することを示しています。この信号は周波数 12MHz で、FPGA 全体で利用できる "グローバル" 信号であり、LVCMOS 電圧規格を使用しています。

ただし、プラットフォームで使用されているツールチェーン（開発環境）を知らなければ、必要な属性情報が何であるかを判断できませんので注意してください。

### 例

Lattice ICE40-HX8Kブレイクアウトボードの例。

```python
from amaranth.build import Platform, Resource, Pins, Clock, Attrs, Connector
from amaranth.build.run import LocalBuildProducts
from amaranth.cli import main_parser, main_runner
from amaranth.vendor.lattice_ice40 import LatticeICE40Platform

class ICE40HX8KBEVNPlatform(LatticeICE40Platform):
    device = "iCE40HX8K"
    package = "CT256"

    resources = [
        Resource("clk1", 0, Pins("J3", dir="i"), Clock(12e6),
                 Attrs(GLOBAL=True, IO_STANDARD="SB_LVCMOS")),  # GBIN6
        Resource("rst", 0, Pins("R9", dir="i"),
                 Attrs(GLOBAL=True, IO_STANDARD="SB_LVCMOS")),  # GBIN5
        Resource("led", 0, Pins("C3", dir="o"),
                 Attrs(IO_STANDARD="SB_LVCMOS")),  # LED2
    ]

    default_clk = "clk1"  # デフォルトのクロックリソースが必要です。
    default_rst = "rst"   # デフォルトのリセットリソースが必要です。

    connectors = [
        Connector(
            "j",
            1,  # J1
            "A16 -   A15 B15 B13 B14 -   -   B12 B11 "
            "A11 B10 A10 C9  -   -   A9  B9  B8  A7  "
            "B7  C7  -   -   A6  C6  B6  C5  A5  C4  "
            "-   -   B5  C3  B4  B3  A2  A1  -   -   "),
        Connector(
            "j",
            2,  # J2
            "-   -   -   R15 P16 P15 -   -   N16 M15 "
            "M16 L16 K15 K16 -   -   K14 J14 G14 F14 "
            "J15 H14 -   -   H16 G15 G16 F15 F16 E14 "
            "-   -   E16 D15 D16 D14 C16 B16 -   -   "),
        Connector(
            "j",
            3,  # J3
            "R16 -   T15 T16 T13 T14 -   -   N12 P13 "
            "N10 M11 T11 P10 -   -   T10 R10 P8  P9  "
            "T9  R9  -   -   T7  T8  T6  R6  T5  R5  "
            "-   -   R3  R4  R2  T3  T1  T2  -   -   "),
        Connector(
            "j",
            4,  # J4
            "-   -   -   R1  P1  P2  -   -   N3  N2  "
            "M2  M1  L3  L1  -   -   K3  K1  J2  J1  "
            "H2  J3  -   -   G2  H1  F2  G1  E2  F1  "
            "-   -   D1  D2  C1  C2  B1  B2  -   -   "),
    ]

    def toolchain_program(self, products: LocalBuildProducts, name: str):
        iceprog = os.environ.get("ICEPROG", "iceprog")
        with products.extract("{}.bin".format(name)) as bitstream_filename:
            subprocess.check_call([iceprog, "-S", bitstream_filename])

# 重要！WSLの場合、以下をインストールしてください
# https://github.com/FPGAwars/toolchain-icestorm/releases/download/v1.11.1/toolchain-icestorm-windows_x86-1.11.1.tar.gz
#
# また、やや古いですが、https://github.com/FPGAwars/toolchain-icestorm/wiki#testing-iceprog も参照してください。

if __name__ == "__main__":
    ICE40HX8KBEVNPlatform().build(Blinker(), do_program=True)  # WSLではFalseに設定してください！
```

## ## ビルド

```sh
python3 file.py
```

これにより、`build` というディレクトリが作成され、出力ファイルが含まれます：

* `top.il`: yosysのilang出力。
* `top.bin`: デバイスに送信するビットストリーム（例：iceprog経由）。
* `top.rpt`: nextpnrからの統計情報。最も役立つのは最後のセルとLUTの数です。
* `top.tim`: タイミング解析。どれくらい高速に実行できるかを示します。おそらく。
* 

## 付録: 既知のLattice ICE40属性

* `GLOBAL`: bool型。Trueの場合、ピンはグローバルです。グローバルピンはデータシートのピンリストで `GBIN` と指定されます。
* `IO_STANDARD`: 文字列:
  * `SB_LVCMOS`: すべてのシングルエンドピンおよび差動出力ピン用
  * `SB_LVDS_INPUT`: 差動入力ピン用

その他の属性はサポートされる場合がありますが、文書化されていません。
