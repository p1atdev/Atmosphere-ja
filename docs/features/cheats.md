# チート

Atmosphère は Action-Replay スタイルのチートコードに対応しており、SD カードからチートを読み込むことができます。

## チートの読み込みプロセス

Atmosphère は、デフォルトでは、新しいアプリケーションプロセスにアタッチするかどうかを決定する際に、次のことを行います。

-   新しいアプリケーションプロセスに関する情報を `pm` と `loader` から取得する。
-   ユーザが定義したキーの組み合わせが保持されているかどうかをチェックし、保持されていない場合は停止します。
    -   デフォルトは "L is not held" ですが、オーバーライドキーを使って設定することができます。
    -   これを設定するための ini キーは `cheat_enable_key` です。
-   プロセスが実際のアプリケーションであるかどうかをチェックし、そうでない場合は停止する。
    -   これは Homebrew Loader にチートコードを適用しないようにするためのものです。
-   ここで `build_id` はアプリケーションのメイン実行ファイルのビルド ID の最初の 8 バイトを 16 進数で表現したものです。
    -   チートが見つからなければ、チートマネージャーは停止します。
-   新しいアプリケーション・プロセスのカーネル・デバッグ・セッションを開きます。
-   新しいチート・プロセスがアタッチされたことをシステム・イベントにシグナルを送ります。

この動作により、チートコードはユーザーが望むときだけロードされるようになります。

`dmnt` がチートマネージャーを起動していないが、ユーザーが起動させたい場合、チートマネージャーのサービス API は`ForceOpenCheatProcess` コマンドを提供し、homebrew はこれを使用することができます。このコマンドはチートマネージャを強制的にプロセスにアタッチさせようとします。

`dmnt` がチートマネージャを有効にしていて、ユーザが別のデバッガを使いたい場合、チートマネージャのサービス API は`ForceCloseCheatProcess` コマンドを提供し、homebrew はこれを使用することができます。このコマンドはチートマネージャをプロセスから切り離すことになります。

デフォルトでは、ロードされた .txt ファイルにリストされたすべてのチートコードがトグルされます。これはユーザーが `atmosphere!dmnt_cheats_enabled_by_default` [system setting](configurations.md) を編集することで設定可能です。

ユーザーは自作プログラムを使って、チートマネージャーのサービス API を経由して、実行時にチートのオン・オフを切り替えることができます。

## チートコードの互換性

Atmosphère は、小さなカスタム仮想マシンの実行を通じて、チートコードを管理します。Atmosphère のチートコードフォーマットは、既存のチートコードフォーマットと完全に後方互換性があることを保証するために、新しい機能が追加され、既存のチートコードアプライヤーのバグが修正されましたが、注意が払われています。以下、既存のフォーマットからの変更点を簡単にまとめます。

-   条件分岐の処理に関わるバグを修正しました。
    -   従来の実装では、条件ブロックの終了を検出する際に、誤った値をチェックするなどの根本的な不具合がありました。
    -   また、従来の実装は命令を適切にデコードせず、ターミネーターの値を線形にスキャンしていました。そのため、ある命令の直値の中にターミネーターが含まれていた場合、問題が発生することがありました。
    -   従来の実装では境界チェックを行わなかったため、特定の条件付きチートコードによって境界外のメモリを読み込んでしまい、データのアボートによってクラッシュする可能性がありました。
-   条件ブロックのネストをサポートしました。
-   2 つのレジスタに対してより複雑な任意の演算を行う命令を追加しました。
-   レジスタの内容を他のレジスタで指定されたメモリーアドレスに書き込むことができる命令を追加しました。
-   従来の実装では、アプリケーションプロセスと正しく同期していなかったため、特定の状況下（特にローディング画面周辺）で大きな遅延が発生していました。Atmosphère の実装ではこれが修正されています。

## チートコードの形式

以下は、チートコードを管理するために使用される仮想マシンの命令フォーマットに関する文書です。

通常、命令タイプは最初の命令 u32 の上位ニブルでエンコードされます。

### コードタイプ 0x0: メモリへの静的な値の保存

コードタイプ 0x0 ではメモリに静的な値を書き込むことができます。

#### 記法

`0TMR00AA AAAAAAAA VVVVVVVV (VVVVVVVV)`

-   T: 書き込む値の幅 (1, 2, 4, 8 バイト)
-   M: 書き込み先のメモリ領域の種類 (0 = Main NSO, 1 = Heap, 2 = Alias, 3 = Aslr)
-   R: メモリ領域ベースからのオフセットとして使用するレジスタ
-   A: メモリ領域ベースからの即値オフセット
-   V: 書き込む値

#### 例

```
08000000 00123456 00000003 FF000000
```

Main のメモリ領域ベースから `0x00123456` の即値オフセットとなるメモリに、8 バイトの値 `0x00000003FF000000` を書き込みます

---

### コードタイプ 0x1: 条件ブロックの開始

コードタイプ 0x1 はメモリの内容と静的な値との比較を実行します。

もし条件に合わない場合、適切な End または Else ブロックまでの全ての命令はスキップされます。

#### 記法

`1TMC00AA AAAAAAAA VVVVVVVV (VVVVVVVV)`

-   T: 書き込むメモリの幅 (1, 2, 4, 8 バイト).
-   M: 書き込み先のメモリ領域の種類 (0 = Main NSO, 1 = Heap, 2 = Alias, 3 = Aslr)
-   C: 使用する条件。下記参照
-   A: メモリ領域ベースからの即値オフセット
-   V: 書き込む値

#### 条件

-   1: >
-   2: >=
-   3: <
-   4: <=
-   5: ==
-   6: !=

#### 例

```
14150000 12345678 40000000
...
[処理]
...
20000000
```

Heap メモリ領域ベースから `0x12345678` の即値オフセットとなるメモリの内容が、`0x40000000` と一致する場合のみ `[処理]` が実行されます。終了ブロックについては、[コードタイプ 0x2](#コードタイプ-0x2:-条件ブロックの終了) を参照してください。

---

### コードタイプ 0x2: 条件ブロックの終了

コードタイプ 0x2 は条件ブロックの終了を表します。 (コードタイプ 0x1 または コードタイプ 0x8 によって開始されます)

Else が実行された時、 適切な End または Else ブロックまでの全ての命令はスキップされます。

#### 記法

`2X000000`

-   X: 終了タイプ (0 = End, 1 = Else).

#### 例

```
14150000 12345678 40000000
...
[処理 1]
...
21000000
...
[処理 2]
...
20000000
```

一行目の条件ブロックで条件に合わなかった場合にのみ、`[処理 2]` が実行されます。

```
80000001
...
[処理]
...
20000000
```

A ボタンが押された時のみ `[処理]` が実行されます。キー入力条件については [コードタイプ 0x8]() を参照してください。

---

### コードタイプ 0x3: ループの開始/終了

コードタイプ 0x3 はループを一定回数だけ繰り返すことができます。

#### ループの開始記法

`300R0000 VVVVVVVV`

-   R: ループのカウンタとして使用するレジスタ
-   V: ループする回数

#### ループの終了記法

`310R0000`

-   R: ループのカウンタとして使用するレジスタ

#### 例

```
300F0000 00000100
...
[処理]
...
310F0000
```

`[処理]` が `0x100` 回実行されます

---

### コードタイプ 0x4: レジスタに静的な値を保存する

コードタイプ 0x4 はレジスタに静的な値をセットすることができます

#### 記法

`400R0000 VVVVVVVV VVVVVVVV`

-   R: 使用するレジスタ
-   V: 保存する値

#### 例

```
400F0000 00000012 3456789A
```

レジスタ `F` に `0x123456789A` を保存します

---

### コードタイプ 0x5: レジスタにメモリの値を読み込む

コードタイプ 0x5 はメモリの値をレジスタに読み込むことができ、 固定アドレス、または間接参照ポインタを持つレジスタの到達先を使用します

#### 固定アドレスから読み込む記法

`5TMR00AA AAAAAAAA`

-   T: 読み込むメモリの幅 (1, 2, 4, 8 バイト)
-   M: 読み込むメモリの領域の種類 (0 = Main NSO, 1 = Heap, 2 = Alias, 3 = Aslr).
-   R: 値を保存するレジスタ
-   A: メモリ領域ベースからの即値オフセット

#### 例

```
580F0000 00123456
```

Main メモリから `0x123456` のオフセットとなるアドレスの、8 バイトのメモリの値をレジスタ F に保存します

#### レジスタから読み込む記法

`5T0R10AA AAAAAAAA`

-   T: 読み込むメモリの幅 (1, 2, 4, 8 バイト)
-   R: 値を保存するレジスタ (このレジスタはベースメモリアドレスにも使用されます)
-   A: レジスタ R との即値オフセット

#### 例

```
580F1000 00000078
```

レジスタ F に保存されている値のアドレスから、オフセット `0x78` となるアドレスのメモリの値を レジスタ F に保存します

---

### Code Type 0x6: Store Static Value to Register Memory Address
Code type 0x6 allows writing a fixed value to a memory address specified by a register.

#### Encoding
`6T0RIor0 VVVVVVVV VVVVVVVV`

+ T: Width of memory write (1, 2, 4, or 8 bytes).
+ R: Register used as base memory address.
+ I: Increment register flag (0 = do not increment R, 1 = increment R by T).
+ o: Offset register enable flag (0 = do not add r to address, 1 = add r to address).
+ r: Register used as offset when o is 1.
+ V: Value to write to memory.

---

### Code Type 0x7: Legacy Arithmetic
Code type 0x7 allows performing arithmetic on registers.

However, it has been deprecated by Code type 0x9, and is only kept for backwards compatibility.

#### Encoding
`7T0RC000 VVVVVVVV`

+ T: Width of arithmetic operation (1, 2, 4, or 8 bytes).
+ R: Register to apply arithmetic to.
+ C: Arithmetic operation to apply, see below.
+ V: Value to use for arithmetic operation.

#### Arithmetic Types
+ 0: Addition
+ 1: Subtraction
+ 2: Multiplication
+ 3: Left Shift
+ 4: Right Shift

---

### Code Type 0x8: Begin Keypress Conditional Block
Code type 0x8 enters or skips a conditional block based on whether a key combination is pressed.

#### Encoding
`8kkkkkkk`

+ k: Keypad mask to check against, see below.

Note that for multiple button combinations, the bitmasks should be ORd together.

#### Keypad Values
Note: This is the direct output of `hidKeysDown()`.

+ 0000001: A
+ 0000002: B
+ 0000004: X
+ 0000008: Y
+ 0000010: Left Stick Pressed
+ 0000020: Right Stick Pressed
+ 0000040: L
+ 0000080: R
+ 0000100: ZL
+ 0000200: ZR
+ 0000400: Plus
+ 0000800: Minus
+ 0001000: Left
+ 0002000: Up
+ 0004000: Right
+ 0008000: Down
+ 0010000: Left Stick Left
+ 0020000: Left Stick Up
+ 0040000: Left Stick Right
+ 0080000: Left Stick Down
+ 0100000: Right Stick Left
+ 0200000: Right Stick Up
+ 0400000: Right Stick Right
+ 0800000: Right Stick Down
+ 1000000: SL
+ 2000000: SR

---

### Code Type 0x9: Perform Arithmetic
Code type 0x9 allows performing arithmetic on registers.

#### Register Arithmetic Encoding
`9TCRS0s0`

+ T: Width of arithmetic operation (1, 2, 4, or 8 bytes).
+ C: Arithmetic operation to apply, see below.
+ R: Register to store result in.
+ S: Register to use as left-hand operand.
+ s: Register to use as right-hand operand.

#### Immediate Value Arithmetic Encoding
`9TCRS100 VVVVVVVV (VVVVVVVV)`

+ T: Width of arithmetic operation (1, 2, 4, or 8 bytes).
+ C: Arithmetic operation to apply, see below.
+ R: Register to store result in.
+ S: Register to use as left-hand operand.
+ V: Value to use as right-hand operand.

#### Arithmetic Types
+ 0: Addition
+ 1: Subtraction
+ 2: Multiplication
+ 3: Left Shift
+ 4: Right Shift
+ 5: Logical And
+ 6: Logical Or
+ 7: Logical Not (discards right-hand operand)
+ 8: Logical Xor
+ 9: None/Move (discards right-hand operand)

---

### Code Type 0xA: Store Register to Memory Address
Code type 0xA allows writing a register to memory.

#### Encoding
`ATSRIOxa (aaaaaaaa)`

+ T: Width of memory write (1, 2, 4, or 8 bytes).
+ S: Register to write to memory.
+ R: Register to use as base address.
+ I: Increment register flag (0 = do not increment R, 1 = increment R by T).
+ O: Offset type, see below.
+ x: Register used as offset when O is 1, Memory type when O is 3, 4 or 5.
+ a: Value used as offset when O is 2, 4 or 5.

#### Offset Types
+ 0: No Offset
+ 1: Use Offset Register
+ 2: Use Fixed Offset
+ 3: Memory Region + Base Register
+ 4: Memory Region + Relative Address (ignore address register)
+ 5: Memory Region + Relative Address + Offset Register

---

### Code Type 0xB: Reserved
Code Type 0xB is currently reserved for future use.

---

### Code Type 0xC-0xF: Extended-Width Instruction
Code Types 0xC-0xF signal to the VM to treat the upper two nybbles of the first dword as instruction type, instead of just the upper nybble.

This reserves an additional 64 opcodes for future use.

---

### Code Type 0xC0: Begin Register Conditional Block
Code type 0xC0 performs a comparison of the contents of a register and another value. This code support multiple operand types, see below.

If the condition is not met, all instructions until the appropriate conditional block terminator are skipped.

#### Encoding
```
C0TcSX##
C0TcS0Ma aaaaaaaa
C0TcS1Mr
C0TcS2Ra aaaaaaaa
C0TcS3Rr
C0TcS400 VVVVVVVV (VVVVVVVV)
C0TcS5X0
```

+ T: Width of memory write (1, 2, 4, or 8 bytes).
+ c: Condition to use, see below.
+ S: Source Register.
+ X: Operand Type, see below.
+ M: Memory Type (operand types 0 and 1).
+ R: Address Register (operand types 2 and 3).
+ a: Relative Address (operand types 0 and 2).
+ r: Offset Register (operand types 1 and 3).
+ X: Other Register (operand type 5).
+ V: Value to compare to (operand type 4).

#### Operand Type
+ 0: Memory Base + Relative Offset
+ 1: Memory Base + Offset Register
+ 2: Register + Relative Offset
+ 3: Register + Offset Register
+ 4: Static Value
+ 5: Other Register

#### Conditions
+ 1: >
+ 2: >=
+ 3: <
+ 4: <=
+ 5: ==
+ 6: !=

---

### Code Type 0xC1: Save or Restore Register
Code type 0xC1 performs saving or restoring of registers.

#### Encoding
`C10D0Sx0`

+ D: Destination index.
+ S: Source index.
+ x: Operand Type, see below.

#### Operand Type
+ 0: Restore register
+ 1: Save register
+ 2: Clear saved value
+ 3: Clear register

---

### Code Type 0xC2: Save or Restore Register with Mask
Code type 0xC2 performs saving or restoring of multiple registers using a bitmask.

#### Encoding
`C2x0XXXX`

+ x: Operand Type, see below.
+ X: 16-bit bitmask, bit i == save or restore register i.

#### Operand Type
+ 0: Restore register
+ 1: Save register
+ 2: Clear saved value
+ 3: Clear register

---

### Code Type 0xC3: Read or Write Static Register
Code type 0xC3 reads or writes a static register with a given register.

#### Encoding
`C3000XXx`

+ XX: Static register index, 0x00 to 0x7F for reading or 0x80 to 0xFF for writing.
+ x: Register index.

---

### Code Type 0xF0: Double Extended-Width Instruction
Code Type 0xF0 signals to the VM to treat the upper three nybbles of the first dword as instruction type, instead of just the upper nybble.

This reserves an additional 16 opcodes for future use.

---

### Code Type 0xFF0: Pause Process
Code type 0xFF0 pauses the current process.

#### Encoding
`FF0?????`

---

### Code Type 0xFF1: Resume Process
Code type 0xFF1 resumes the current process.

#### Encoding
`FF1?????`

---

### Code Type 0xFFF: Debug Log
Code type 0xFFF writes a debug log to the SD card under the folder `/atmosphere/cheat_vm_logs/`.

#### Encoding
```
FFFTIX##
FFFTI0Ma aaaaaaaa
FFFTI1Mr
FFFTI2Ra aaaaaaaa
FFFTI3Rr
FFFTI4X0
```

+ T: Width of memory write (1, 2, 4, or 8 bytes).
+ I: Log id.
+ X: Operand Type, see below.
+ M: Memory Type (operand types 0 and 1).
+ R: Address Register (operand types 2 and 3).
+ a: Relative Address (operand types 0 and 2).
+ r: Offset Register (operand types 1 and 3).
+ X: Value Register (operand type 4).

#### Operand Type
+ 0: Memory Base + Relative Offset
+ 1: Memory Base + Offset Register
+ 2: Register + Relative Offset
+ 3: Register + Offset Register
+ 4: Register Value
