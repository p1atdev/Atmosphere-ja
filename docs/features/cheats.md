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

### コードタイプ 0x0: メモリに静的な値を書き込む

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

### コードタイプ 0x4: レジスタに静的な値を書き込む

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

### コードタイプ 0x6: レジスタのメモリアドレスに静的な値を保存する

コードタイプ 0x6 はレジスタで指定されたメモリアドレスに固定値を書き込むことができます

#### 記法

`6T0RIor0 VVVVVVVV VVVVVVVV`

-   T: 書き込むメモリの幅 (1, 2, 4, 8 バイト).
-   R: ベースメモリアドレスとして使用するレジスタ
-   I: レジスタのインクリメントフラグ (0 = R をインクリメントしない, 1 = R を T だけインクリメントする)
-   o: オフセットレジスタの有効フラグ (0 = r をアドレスに加算しない, 1 = r をアドレスに加算する)
-   r: o が 1 の場合にオフセットとして使用されるレジスタ
-   V: メモリに書き込む値

#### 例

```
640F0000 00000012 3456789A
```

レジスタ F で指定されたメモリアドレスに、`0x123456789A` を書き込みます

---

### コードタイプ 0x7: レガシーな算術

コードタイプ 0x7 はレジスタに対して算術を行うことができます

しかし、コードタイプ 0x9 によって非推奨となっており、後方互換性のために残されています

#### 記法

`7T0RC000 VVVVVVVV`

-   T: 算術演算する幅 (1, 2, 4, 8 バイト).
-   R: 算術を適用するレジスタ
-   C: 適用する算術演算の種類。下記参照
-   V: 算術演算に使用される値

#### 算術タイプ

-   0: 加算
-   1: 減算
-   2: 乗算
-   3: 左シフト
-   4: 右シフト

#### 例

```
740F0000 00001234
```

レジスタ F の値に `0x1234` を加算します

---

### コードタイプ 0x8: キー入力条件ブロックの開始

コードタイプ 0x8 はキーコンビネーションが入力されたかどうかによって実行またはスキップされる条件ブロックを開始します

#### 記法

`8kkkkkkk`

-   k: 照合するキーパッドマスク。下記参照

なお、複数のボタンの組み合わせの場合、ビットマスクは ORd でまとめる必要があります

#### キーパッドの値

注: これは `hidKeysDown()` の直接の出力です

-   0000001: A
-   0000002: B
-   0000004: X
-   0000008: Y
-   0000010: 左スティック押し込み
-   0000020: 右スティック押し込み
-   0000040: L
-   0000080: R
-   0000100: ZL
-   0000200: ZR
-   0000400: プラス
-   0000800: マイナス
-   0001000: 左
-   0002000: 上
-   0004000: 右
-   0008000: 下
-   0010000: 左スティック左
-   0020000: 左スティック上
-   0040000: 左スティック右
-   0080000: 左スティック下
-   0100000: 右スティック左
-   0200000: 右スティック上
-   0400000: 右スティック右
-   0800000: 右スティック下
-   1000000: SL
-   2000000: SR

#### 例

```
80000300
...
[処理]
...
20000000
```

ZR と ZL を同時に押した時のみ `処理` が実行されます

---

### コードタイプ 0x9: 算術演算

コードタイプ 0x9 はレジスタに対して算術演算をすることができます

#### レジスタ算術記法

`9TCRS0s0`

-   T: 算術演算の幅 (1, 2, 4, 8 バイト)
-   C: 適用する算術演算の種類。下記参照
-   R: 結果を保存するレジスタ
-   S: 左辺オペランドに使われるレジスタ
-   s: 右辺オペランドに使われるレジスタ

#### 即値算術記法

`9TCRS100 VVVVVVVV (VVVVVVVV)`

-   T: 算術演算の幅 (1, 2, 4, 8 バイト)
-   C: 適用する算術演算の種類。下記参照
-   R: 結果を保存するレジスタ
-   S: 左辺オペランドに使われるレジスタ
-   V: 右辺オペランドに使われる値

#### 算術タイプ

-   0: 加算
-   1: 減算
-   2: 乗算
-   3: 左シフト
-   4: 右シフト
-   5: 論理積 And
-   6: 論理和 Or
-   7: 論理否定 Not (右辺のオペランドを破棄する)
-   8: 排他的論理和 Xor
-   9: なし/移動 (右辺のオペランドを破棄する)

---

### コードタイプ 0xA: メモリアドレスにレジスタを書き込む

コードタイプ 0xA メモリにレジスタを書き込むことができます

#### 記法

`ATSRIOxa (aaaaaaaa)`

-   T: 書き込むメモリの幅 (1, 2, 4, 8 バイト).
-   S: メモリに書き込むレジスタ
-   R: ベースアドレスとして使用するレジスタ
-   I: レジスタのインクリメントフラグ (0 = R をインクリメントしない, 1 = R を T だけインクリメントする)
-   O: オフセットタイプ。下記参照
-   x: O が 1 のとき、オフセットとして使用される値。O が 3, 4, 5 のとき、メモリタイプ
-   a: O が 2, 4, 5 の時、オフセットとして使用される値

#### オフセットタイプ

-   0: オフセットなし
-   1: レジスタオフセットを使用する
-   2: 固定オフセットを使用する
-   3: メモリ領域 + ベースレジスタ
-   4: メモリ領域 + 相対アドレス (アドレスレジスタを無視する)
-   5: メモリ領域 + 相対アドレス + オフセットレジスタ

---

### コードタイプ 0xB: 予約

コードタイプ 0xB は現在、将来の拡張のために予約されています。

---

### コードタイプ 0xC-0xF: 拡張幅命令

Code Types 0xC-0xF signal to the VM to treat the upper two nybbles of the first dword as instruction type, instead of just the upper nybble.

This reserves an additional 64 opcodes for future use.

---

### コードタイプ 0xC0: レジスタ条件ブロックの開始

Code type 0xC0 performs a comparison of the contents of a register and another value. This code support multiple operand types, see below.

If the condition is not met, all instructions until the appropriate conditional block terminator are skipped.

#### 記法

```
C0TcSX##
C0TcS0Ma aaaaaaaa
C0TcS1Mr
C0TcS2Ra aaaaaaaa
C0TcS3Rr
C0TcS400 VVVVVVVV (VVVVVVVV)
C0TcS5X0
```

-   T: Width of memory write (1, 2, 4, or 8 bytes).
-   c: Condition to use, see below.
-   S: Source Register.
-   X: Operand Type, see below.
-   M: Memory Type (operand types 0 and 1).
-   R: Address Register (operand types 2 and 3).
-   a: Relative Address (operand types 0 and 2).
-   r: Offset Register (operand types 1 and 3).
-   X: Other Register (operand type 5).
-   V: Value to compare to (operand type 4).

#### オペランドタイプ

-   0: Memory Base + Relative Offset
-   1: Memory Base + Offset Register
-   2: Register + Relative Offset
-   3: Register + Offset Register
-   4: Static Value
-   5: Other Register

#### 条件

-   1: >
-   2: >=
-   3: <
-   4: <=
-   5: ==
-   6: !=

---

### コードタイプ 0xC1: レジスタの保存・復元

Code type 0xC1 performs saving or restoring of registers.

#### 記法

`C10D0Sx0`

-   D: Destination index.
-   S: Source index.
-   x: Operand Type, see below.

#### オペランドタイプ

-   0: Restore register
-   1: Save register
-   2: Clear saved value
-   3: Clear register

---

### コードタイプ 0xC2:マスクによるレジスタの保存・復元

Code type 0xC2 performs saving or restoring of multiple registers using a bitmask.

#### 記法

`C2x0XXXX`

-   x: Operand Type, see below.
-   X: 16-bit bitmask, bit i == save or restore register i.

#### オペランドタイプ

-   0: Restore register
-   1: Save register
-   2: Clear saved value
-   3: Clear register

---

### コードタイプ 0xC3: 固定レジスタに静的な値を読み書きする

Code type 0xC3 reads or writes a static register with a given register.

#### 記法

`C3000XXx`

-   XX: Static register index, 0x00 to 0x7F for reading or 0x80 to 0xFF for writing.
-   x: Register index.

---

### コードタイプ 0xF0: ダブル拡張幅命令

Code Type 0xF0 signals to the VM to treat the upper three nybbles of the first dword as instruction type, instead of just the upper nybble.

This reserves an additional 16 opcodes for future use.

---

### コード来ぷ 0xFF0: プロセスを一時停止

Code type 0xFF0 pauses the current process.

#### 記法

`FF0?????`

---

### コードタイプ 0xFF1: プロセスを再開

Code type 0xFF1 resumes the current process.

#### 記法

`FF1?????`

---

### コードタイプ 0xFFF: デバッグログ

Code type 0xFFF writes a debug log to the SD card under the folder `/atmosphere/cheat_vm_logs/`.

#### 記法

```
FFFTIX##
FFFTI0Ma aaaaaaaa
FFFTI1Mr
FFFTI2Ra aaaaaaaa
FFFTI3Rr
FFFTI4X0
```

-   T: Width of memory write (1, 2, 4, or 8 bytes).
-   I: Log id.
-   X: Operand Type, see below.
-   M: Memory Type (operand types 0 and 1).
-   R: Address Register (operand types 2 and 3).
-   a: Relative Address (operand types 0 and 2).
-   r: Offset Register (operand types 1 and 3).
-   X: Value Register (operand type 4).

#### オペランドタイプ

-   0: Memory Base + Relative Offset
-   1: Memory Base + Offset Register
-   2: Register + Relative Offset
-   3: Register + Offset Register
-   4: Register Value
