
# 並列プログラミングに関するメモ

## バリア

プロセッサは、プログラムに記述された命令の実行順（命令オーダ）や、メモリアクセス順序（メモリオーダ）を保証するとは限らない。性能を向上させるために命令やメモリアクセストランザクションの入れ替えを行う。命令オーダおよびメモリオーダの入れ替えを行うか否かはプロセッサの仕様に依存する。メモリやデバイス、CPU制御レジスタを共有するマルチプロセッサシステム（場合によっては単一プロセッサシステム）では、各コアで前記命令・メモリのリオーダが発生することで、共有資源を正しく観測できなかったり、デバイスを正しく制御できない場合がある。
バリアとは、そのようなプロセッサシステムにおける、命令およびメモリアクセスの順序を制御する命令の集合を指す。

>NOTE: ARMv8 AArch64では、メモリオーダは相互接続するDDRなどの[デバイス依存](https://community.arm.com/support-forums/f/architectures-and-processors-forum/6354/barriers-in-in-order-cores-like-cortex-a53-a7)（ほとんどの場合リオーダすると考えた方が安全）であり、命令オーダはCA53/CA57などはインオーダ、CA72等はアウトオブオーダである（[参考](https://en.wikipedia.org/wiki/Comparison_of_ARMv8-A_processors)）

### バリアの種類（ARMv8）

#### データメモリバリア(DMB)

##### DMB SY

バリア以前のメモリアクセス命令は、バリア命令以降のすべてのメモリアクセス命令が実行される前に完了する。
メモリアクセス命令以外の命令には効果をもたない。

##### DMB ST

バリア以前のすべてのストア命令は、バリア命令以降のすべてのストア命令が実行される前に完了する。
メモリアクセス命令以外の命令には効果をもたない。

##### DMB LD

バリア以前のすべてのロード命令は、バリア命令以降のすべてのメモリアクセス命令が実行される前に完了する。
メモリアクセス命令以外の命令には効果をもたない。

#### データ同期バリア(DSB)
##### DSB SY

バリア以前のメモリアクセス命令は、バリアの完了時に完了する。
バリア命令以降に配置されたすべての命令は、バリアが完了するまで実行されない。

##### DSB ST

バリア以前のすべてのストア命令は、バリアの完了時に完了する。
バリア命令以降に配置されたすべての命令は、バリアが完了するまで実行されない。

##### DSB LD

バリア以前のすべてのロード命令は、バリアの完了時に完了する。
バリア命令以降に配置されたすべての命令は、バリアが完了するまで実行されない。

#### DMB/DSBのその他のオプション

なお、以下のようにオプションを変更すると、上記の3つオプションに加えて、バリアの作用を及ぼすメモリアクセス先（共有ドメイン）を限定することができる。

- `SY`(全システム) -> `OSH`(Outer Sharableドメイン) -> `ISH`(Inner Sharableドメイン) -> `NSH`(Non-sharableドメイン)
- `ST`(全システム) -> `OSHST`(Outer Sharableドメイン) -> `ISHST`(Inner Sharableドメイン) -> `NSHST`(Non-sharableドメイン)
- `LD`(全システム) -> `OSHLD`(Outer Sharableドメイン) -> `ISHLD`(Inner Sharableドメイン) -> `NSHLD`(Non-sharableドメイン)

各共有ドメインの定義イメージを以下に引用する（[参考元](https://developer.arm.com/documentation/100941/0100/Memory-attributes)）：

![img](https://documentation-service.arm.com/static/5efa1dbedbdee951c1ccded0?token=)


#### 命令バリア(ISB)

バリアの挿入位置以前の命令による、システム制御レジスタへのアクセスおよびアクセスによる副作用が全て完了し、挿入位置以降の命令から観測できることを保証する。
さらに、バリアの挿入位置以降の命令によるシステム制御レジスタへのアクセスおよびアクセスによる副作用は、バリアの挿入位置以前には発生しないことを保証する。

例）

以下の例では、システム制御レジスタCPACR_EL1にリードモディファイライトすることで、FPU（浮動小数点演算ユニットと）とSIMD（ベクトル処理）を有効化したあとでバリアを実行している。したがって、以降の命令C、命令Dは浮動小数点演算ユニット・ベクトル処理が有効化された状態で実行されることが保証される。また、ISB実行時にパイプラインは粛清されるので、命令C、命令Dはバリアの実行後、再度メモリ（またはキャッシュ）からフェッチされる。

```
MRS X1, CPACR_EL1
ORR X1, X1, #(0x3 << 20) // FPU と SIMD を有効化
MSR CPACR_EL1, X1
ISB // 命令バリア
命令C
命令D
```

#### LinuxでのArmv8バリアの対応付け

#### 参考

1. https://developer.arm.com/documentation/dui0489/c
1. https://developer.arm.com/documentation/ddi0596/2021-12/Base-Instructions/DSB--Data-Synchronization-Barrier-?lang=en
1. https://github.com/Broadcom/arm64-linux/blob/master/Documentation/memory-barriers.txt

### バリアのユースケース

#### データメモリバリアのユースケース

##### コア間共有データの更新をメモリ（フラグ）を使って通知する

あるコアAが更新し、もう一方のコアBが参照する共有データがあり、データがCPU命令によって１度に更新できないサイズである場合、すべてのデータの更新完了をメモリ上のフラグやカウンタを使って通知する方法が考えられる。この場合、コアAにおいて、データの更新（ライト操作）とフラグの更新（ライト操作）は順序どおり実行されなければならないので、データの更新直後にライトメモリバリアが必要である。また、コアBにおいてもフラグの参照（リード操作）とデータの参照（リード操作）が順序どおり実行されなければならないので、データの参照直後にリードメモリバリアが必要である。以下にこの場合の疑似コードを示す：

```
コアA                          コアB
共有データ[0] = 更新データ0      if(更新フラグ == true) then
共有データ[1] = 更新データ1        リードメモリバリア()
...                             バッファ[0] = 共有データ[0] 
共有データ[n] = 更新データn        バッファ[1] = 共有データ[1]
ライトメモリバリア()               ...
更新フラグ = true                 バッファ[n] = 共有データ[n]
                                 受信データを使った処理（バッファ）
                               endif
```

コード例）

参照元：https://elixir.bootlin.com/linux/latest/source/net/ipv4/tcp.c#L4342

- 前記疑似コードのコアAにあたる処理

```c
static void __tcp_alloc_md5sig_pool(void)
{
	struct crypto_ahash *hash;
	int cpu;

	hash = crypto_alloc_ahash("md5", 0, CRYPTO_ALG_ASYNC);
	if (IS_ERR(hash))
		return;

    // 共有データの更新
    for_each_possible_cpu(cpu) {
		...省略（知りたいときは参照元をみて！）
	}
	/* before setting tcp_md5sig_pool_populated, we must commit all writes
	 * to memory. See smp_rmb() in tcp_get_md5sig_pool()
	 */
	smp_wmb(); // ライトメモリバリア
	tcp_md5sig_pool_populated = true;  // フラグの更新
}
```

- 前記疑似コードのコアBにあたる処理

```c
struct tcp_md5sig_pool *tcp_get_md5sig_pool(void)
{
	local_bh_disable();

	if (tcp_md5sig_pool_populated) { // フラグの参照
		/* coupled with smp_wmb() in __tcp_alloc_md5sig_pool() */
		smp_rmb(); // リードメモリバリア
		return this_cpu_ptr(&tcp_md5sig_pool); // 共有データを使用した処理
	}
	local_bh_enable();
	return NULL;
}
```

##### その他の例

例）

以下の例では、リストデータへの`node`の挿入（`prev->next = node`操作）前に、ライトデータメモリバリアによって、`node->prev`の更新が完了することを保証し、リスト上の要素の`->prev`が常にリスト上の前の要素をさすようにしている。もし、ライトメモリバリアを挿入しない場合、`node`の挿入時点で`node->prev`が不定なメモリアドレスをさす場合があり、この時点でリストを走査する処理は不正なアドレスを参照する危険性がある。

参照元：https://elixir.bootlin.com/linux/latest/source/kernel/locking/osq_lock.c#L124

```c
bool osq_lock(struct optimistic_spin_queue *lock)
{
    ...
	node->prev = prev; // ①バリア前に完了することが保証されている

	/*
	 * osq_lock()			unqueue
	 *
	 * node->prev = prev		osq_wait_next()
	 * WMB				MB
	 * prev->next = node		next->prev = prev // unqueue-C
	 *
	 * Here 'node->prev' and 'next->prev' are the same variable and we need
	 * to ensure these stores happen in-order to avoid corrupting the list.
	 */
	smp_wmb();

	WRITE_ONCE(prev->next, node); // ②バリア後（つまり①の書込み完了後）にライトされることが保証されている
    ...
}
```


### C/C++でのバリアの使い方

C/C++プログラムでは、バリアはインラインアセンブラで記述するか、ほとんどの場合プラットフォームの提供するプロセッサアーキテクチャ非依存のマクロで、バリアを利用する。

<!-- ### glibc
### linuxカーネル
### Toppers -->

## ライブラリ依存の機能

未

## OS依存の機能

未
