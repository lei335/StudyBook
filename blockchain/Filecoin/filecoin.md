# filecoin

## basic

+ merkle tree

```rust
pub struct StoreConfig {
    /// A directory in which data (a merkle tree) can be persisted.
    pub path: PathBuf,

    /// A unique identifier used to help specify the on-disk store
    /// location for this particular data.
    pub id: String, // such as tree-d;

    /// The number of elements in the DiskStore.  This field is
    /// optional, and unused internally.
    pub size: Option<usize>,

    /// The number of merkle tree rows_to_discard then cache on disk.
    pub rows_to_discard: usize, // depend on arity such as BINARY_ARITY(2), OCT_ARITY(8)
}

data_tree = create_base_merkle_tree(StoreConfig, usize, &[u8]) {
    from_par_iter_with_config{
    	build{
        	// process each layer; read 4096 in parallel,  hash per branch(8*32)
            // write to store
        	process_layer
    	}
	}
}
```



## sealing

+ add piece

copy from piece file to staged file （32GB） 

由于Fr($r \approx 2^{255}$, r = 0x73eda753299d7d483339d80809a1d80553bda402fffe5bfeffffffff00000001)

原始数据按照254bits切割，然后padding为 256bits，使得数据小于r。

```rust
pub struct PieceInfo {
    pub commitment: Commitment, 
    pub size: UnpaddedBytesAmount,
}

pub fn add_piece<R, W>(
    source: R,
    target: W,
    piece_size: UnpaddedBytesAmount,
    piece_lengths: &[UnpaddedBytesAmount],
) -> Result<(PieceInfo, UnpaddedBytesAmount)>
where
    R: Read,
    W: Write,
{
    // 根据reader计算commitment（按照二叉树计算的hash根）
    commitment_reader 
}
```





### pre commit1

degree: 6

expansion: 8

1. data  -> 2x data_tree, comm_d;  generate repicaID；(32GB raw data + 32GB tree)
2. generate graph parents; (32GB/32B*14 = 56GB memory)
3. compute layers linearly (32GB*11)  ~30minute per layer

```rust
pub struct PoRepConfig {
    pub sector_size: SectorSize, // 32GB
    pub partitions: PoRepProofPartitions, // 32GB 设置为 10
    pub porep_id: [u8; 32], // 32字节id
}

fn generate_replica_id::<Tree::Hasher, _>(
    &prover_id, // 生成者id
    sector_id.into(), // 对于每个prover_id， 从0开始
    &ticket,  // 随机数，大周期变更或者阶段更新？
    comm_d, // 数据的commitment
    &porep_config.porep_id, // StackedDrg2KiBV1, StackedDrg32GiBV1
);

pub fn seal_pre_commit_phase1<R, S, T, Tree: 'static + MerkleTreeTrait>(
    porep_config: PoRepConfig,
    cache_path: R,
    in_path: S,
    out_path: T,
    prover_id: ProverId,
    sector_id: SectorId,
    ticket: Ticket,
    piece_infos: &[PieceInfo],
) -> Result<SealPreCommitPhase1Output<Tree>>
where
    R: AsRef<Path>,
    S: AsRef<Path>,
    T: AsRef<Path>,
{
     let compound_setup_params = compound_proof::SetupParams {
        vanilla_params: setup_params(
            PaddedBytesAmount::from(porep_config),
            usize::from(PoRepProofPartitions::from(porep_config)),
            porep_config.porep_id,
      )?,
      partitions: Some(usize::from(PoRepProofPartitions::from(porep_config))),
      priority: false,
    };
    
    //生成merkle tree并存放在disk，32GB+32GB
    data_tree = create_base_merkle_tree(Option<StoreConfig>, size: usize, data: &[u8],)
    let comm_d_root: Fr = data_tree.root().into();
    let comm_d = commitment_from_fr(comm_d_root); // fr -> bytes
	drop(data_tree) // 释放
	// 验证comm_d和之前计算的commiment是否一样
	verify_pieces(&comm_d, piece_infos, sector_size)
}

// generate replica id
let replica_id = generate_replica_id()
// 根据replica id计算layer数据
let labels = StackedDrg::<Tree, DefaultPieceHasher>::replicate_phase1(
        pp: &'a PublicParams<Tree>,
        replica_id: &<Tree::Hasher as Hasher>::Domain,
        config: StoreConfig,
) {
     generate_labels(&pp.graph, &pp.layer_challenges, replica_id, config) {
         layer_labels[..layer_size]
         exp_labels[..layer_size] // store previous layer
         for layer in ...layers {
         if layer == 1 {
            for node in 0..graph.size() {
                buffer[..4] = layer_index
             	buffer[4..12] = node as u64
                if node == 0 {
                    h = hash(replica_id, buffer)
                }else {
                    cp = parents(6) of node at this layer
                    parent = cp 
                    h = hash(replica_id, buffer, parents, parents, parents, parents, parents, parents, parents[0])
                }
                layer_labels[node] = h->Fr // strip last two bits,
             }
         } else {
             for node in 0..graph.size() {
                buffer[..4] = layer_index
             	buffer[4..12] = node as u64
                if node == 0 {
                    h = hash(replica_id, buffer)
                }else {
                    cp = parents(6) of node at this layer
                    pp = parents(8) of node at previous
                    parent = cp + pp
                    h = hash(replica_id, buffer, parent, parent, parent[..9])
                }
                layer_labels[node] = h->Fr // strip last two bits,
             }
         }
         new_from_slice_with_config()  // store this layer on disk; accord to path and layer id   
         swap(layer_labels, exp_labels) // 交换两层 
         }
    }
}
let out = SealPreCommitPhase1Output {
    labels, // disk informaion of all layers
    config,
    comm_d,
};
Ok(out)
```



### pre commit2

GPU accelerate

1. data + last layer -> tree_r_last, comm_r_last (32GB)
2. layers -> layer tree, comm_c; comm_r = hash(comm_c|| comm_r_last)

layers 的相同位置的值聚合起来作为layer tree的leaf node

```shell
FIL_PROOFS_USE_GPU_COLUMN_BUILDER – 使用GPU，进行column hash的计算

FIL_PROOFS_MAX_GPU_COLUMN_BATCH_SIZE – 每次计算Column的batch大小，默认400000

FIL_PROOFS_COLUMN_WRITE_BATCH_SIZE – 每次刷Column数据的batch大小，默认262144

FIL_PROOFS_USE_GPU_TREE_BUILDER – 使用GPU，构造Merkle树

FIL_PROOFS_MAX_GPU_TREE_BATCH_SIZE – 每次Encoding计算的batch大小，默认700000
```



```rust
pub fn seal_pre_commit_phase2<R, S, Tree: 'static + MerkleTreeTrait>(
    porep_config: PoRepConfig,
    phase1_output: SealPreCommitPhase1Output<Tree>,
    cache_path: S,
    replica_path: R,
) -> Result<SealPreCommitOutput>
where
    R: AsRef<Path>,
    S: AsRef<Path>,
{
    load data_tree from disk
    steup compound_setup_params
    setup compound_public_params
    replicate_phase2() {
    transform_and_replicate_layers_inner () {
        store configs in StoreConfig and verify(tree_d_config(2), tree_r_last_config(8), tree_c_config(8))
        // support 2,8,11 layers; support cpu/gpu calc
        generate_tree_c_cpu () {
            // 根据cpu切分layer的数据； 每个cpu做一块；
            let n = num_cpus::get(); //such as 16 cores
            let num_chunks = if n > nodes_count * 2 { 1 } else { n }; // equal n
            let chunk_size = (nodes_count as f64 / num_chunks as f64).ceil() // chunk into n chunks
            // 每一层相同位置的32B做一次Poseidon hash；形成一个新的layer，然后生成8叉树的tree_c
            // 此处代码性能太低
            // layer保存在本地；
            from_par_iter_with_config
            // layer的低三层不保存
            create_disk_tree()   
        }
        // gpu: FIL_PROOFS_USE_GPU_COLUMN_BUILDER = 1; FIL_PROOFS_USE_GPU_TREE_BUILDER =1;
        generate_tree_c_gpu () {
            // 创建同步通道，rx，tx send， rx blocks until receive data
            let (builder_tx, builder_rx) = mpsc::sync_channel(0);
            let max_gpu_column_batch_size = 400,000
            let max_gpu_tree_batch_size = 700,000
            let column_write_batch_size = 162,144
            // 从disk读取苏数据
            send {
                 // layer作为1维，位置作为2维
            	let mut layer_data = vec![Vec::with_capacity(400,000); layers];
            	layer_data = read data from disk //each layer 12.8 MB = 400,000*32B, 140.8MB = 12.8MB*11 layers
            	//  转化二位数组
            	columns以位置作为1维，layer作为2维
            	// 发送
            	build_tx.send(columns)
            }
            // 处理数据
            receive {
                // GPU的build
                ColumnTreeBuilder{BatcherType::GPU, nodes_count, max_gpu_column_batch_size, max_gpu_tree_batch_size}
                while {
                    // 等待数据
                    columns = builder_rx.recv()
                    // 处理数据，计算column hash use gpu
                    column_tree_builder.add_columns(columns)
                    // 最后一块；cal column hash then build tree（利用gpu算hash）
                    let (base_data, tree_data) = column_tree_builder.add_final_columns(columns)
                    // 按照8MB = 266144*32B将数据写入磁盘
                    flatten_and_write_store(&base_data, 0)
                    // 接着写入tree data
                    flatten_and_write_store(&tree_data, base_data.len())
                }
            }
        }
        // encoded(last layer, data), generate merkle tree(8) and store;   
        generate_tree_r_last()  // use gpu or cpu
        comm_r = hash(tree_c.root, tree_r_last.root)
        Tau {comm_d, comm_r}
        PersistentAux(comm_c, comm_r_last)
        TemporaryAux(lebel_configs, tree_d_config, tree_r_last_config, tree_c_config) // 配置路径信息
        }    
    }
    persistent p_aux, t_aux;
}
```



### interactive

1. send (comm_d, comm_r) to chain
2. get challenge seed (from chain or somewhere)

### commit1

生成proof需要验证：node i属于comm_d; encode后属于comm_r

1. 根据challenge seed，挑战sector的某些node，为挑战的node以及parent node生成data merkle proof，和layer merkle proof。

此时需要本地存储的（raw data + 11 layer data+ encoded data + data tree + encode data tree + layer tree）

```rust
pub fn seal_commit_phase1<T: AsRef<Path>, Tree: 'static + MerkleTreeTrait>(
    porep_config: PoRepConfig,
    cache_path: T,
    replica_path: T,
    prover_id: ProverId,
    sector_id: SectorId,
    ticket: Ticket, // 链上随机数？
    seed: Ticket,
    pre_commit: SealPreCommitOutput,
    piece_infos: &[PieceInfo],
) -> Result<SealCommitPhase1Output<Tree>> {
    let replica_id = generate_replica_id()
    let public_inputs = stacked::PublicInputs {
        replica_id,
        tau: {comm_d,comm_r}),
        k: None,
        seed, 
    };
    let private_inputs = stacked::PrivateInputs::<Tree, DefaultPieceHasher> {
        p_aux,
        t_aux,
    };
    let compound_setup_params = compound_proof::SetupParams {};
    let compound_public_params = fn (compound_setup_params);
    // merkle proof: 32+32+(height-1)*(neighours*32+4); 
    // 2 32GB: 32 * 2 + 30 * (32 + 4) = 1144 bytes
    // 8 32GB: 32 * 2 + 10 * (32*7 + 4) = 2344 bytes
	pub struct PathElement<H: Hasher, Arity: PoseidonArity> {
    	hashes: Vec<H::Domain>, // 32Bytes*neighours, for 8x树，为32 bytes*7
    	index: usize, // 4 bytes
    	_arity: PhantomData<Arity>,
	}
	pub struct InclusionPath<H: Hasher, Arity: PoseidonArity> {
    	path: Vec<PathElement<H, Arity>>, // tree_height-1
	}
    struct SingleProof<H: Hasher, Arity: PoseidonArity> {
    	/// Root of the merkle tree.
    	root: H::Domain, // 32 bytes
    	/// The original leaf data for this prof.
    	leaf: H::Domain, // 32 bytes
    	/// The path from leaf to root.
    	path: InclusionPath<H, Arity>,
	}
    
	pub struct Column<H: Hasher> { // 356 bytes
        pub(crate) index: u32, // 4 bytes
        pub(crate) rows: Vec<H::Domain>, // 每一层的数据 32 bytes*layers; 352 bytes
    	_h: PhantomData<H>,
	}
	pub struct ColumnProof<Proof: MerkleProofTrait> { // 2700 bytes
    	pub(crate) column: Column<Proof::Hasher>, // 356 bytes
    	pub(crate) inclusion_proof: Proof, // 对应的tree_c merkle proof
	}
	ReplicaColumnProof { // 15*2700 = 40500
        pub c_x: ColumnProof<Proof>, // tree_c at challenge node
        pub drg_parents: Vec<ColumnProof<Proof>>, // 
        pub exp_parents: Vec<ColumnProof<Proof>>, // 
    }

	// labelingProof at each layer
	pub struct LabelingProof<H: Hasher> { // 1196
    	pub(crate) parents: Vec<H::Domain>, // 37 *32
    	pub(crate) layer_index: u32, // 4 bytes
    	pub(crate) node: u64, // 8 bytes
    	_h: PhantomData<H>,
	}
	
	// last layer 
	pub struct EncodingProof<H: Hasher> { //1196
    	pub(crate) parents: Vec<H::Domain>, // 37 *32
    	pub(crate) layer_index: u32, // 4 bytes
    	pub(crate) node: u64, // 8 bytes
    	_h: PhantomData<H>,
	}

	// 32GB, 58340 bytes, 176 challenges / 10 partitions; ~1MB*10
	pub struct Proof<Tree: MerkleTreeTrait, G: Hasher> {
        // 原始数据的proof， 32 * 2 + 30 * (32 + 4) = 1144 bytes
        pub comm_d_proofs: MerkleProof<G, typenum::U2>,
        // 编码数据的proof; 32 * 2 + 10 * (32*7 + 4）= 2344 bytes
        pub comm_r_last_proof:
        MerkleProof<Tree::Hasher, Tree::Arity, Tree::SubTreeArity, Tree::TopTreeArity>,
        // tree_c列proof, 40500 bytes
        pub replica_column_proofs: ReplicaColumnProof<
        MerkleProof<Tree::Hasher, Tree::Arity, Tree::SubTreeArity, Tree::TopTreeArity>,
    >,
    	/// Indexed by layer in 1..layers, 13156 bytes
    	pub labeling_proofs: Vec<LabelingProof<Tree::Hasher>>, // 13156 = 11*1196
        //  最后一层，验证encode， 1196 bytes
    	pub encoding_proof: EncodingProof<Tree::Hasher>, 
	}

	// Stacked.vanilla.proof_scheme.prove_all_partitions() 
    let vanilla_proofs = StackedDrg::prove_all_partitions(){
    	// 根据challenge个数。生成proof; ~8MB
    	prove_layers() {
            comm_d_proof(merkel proof) 
            ReplicaColumnProof
           	comm_r_last_proof 
            LalelingProof(layers) 
            EncodingProof()
        }       
    }
	// 
	// Stacked.vanilla.proof_scheme.verify_all_partitions() 
	verify_all_partitions(){
        verify comm_c, comm_r_last, and comm_r（从链上获取comm_r）
        verify comm_d_proof
        verify replica_column_proofs
        verify final_replica_layer
        verify labels
        verify encoding
	}

	let out = SealCommitPhase1Output {
        vanilla_proofs,
        comm_r,
        comm_d,
        replica_id,
        seed,
        ticket,
    };
}

    
```

### commit 2

merkle proof->snark proof

```rust
pub fn seal_commit_phase2<Tree: 'static + MerkleTreeTrait>(
    porep_config: PoRepConfig,
    phase1_output: SealCommitPhase1Output<Tree>,
    prover_id: ProverId,
    sector_id: SectorId,
) -> Result<SealCommitOutput> {
    // 公共参数
    let public_inputs = stacked::PublicInputs {
        replica_id,
        tau: Some(stacked::Tau {comm_d,comm_r,
        }),
        k: None,
        seed,
    };
    
    // groth 参数
    let groth_params = get_stacked_params::<Tree>(porep_config)?;
    // 参数准备
    let compound_setup_params = compound_proof::SetupParams {
        vanilla_params: setup_params(
            PaddedBytesAmount::from(porep_config),
            usize::from(PoRepProofPartitions::from(porep_config)),
            porep_config.porep_id,
        )?,
        partitions: Some(usize::from(PoRepProofPartitions::from(porep_config))),
        priority: false,
    };

    let compound_public_params = <StackedCompound<Tree, DefaultPieceHasher> as CompoundProof<
        StackedDrg<Tree, DefaultPieceHasher>,
        _,
    >>::setup(&compound_setup_params)?;
    
    pub_params{
        layered_drgporep::PublicParams{ graph: stacked_graph::StackedGraph{expansion_degree: 8 base_graph: drgraph::BucketGraph{size: 16777216; degree: 6; hasher: poseidon_hasher} }, challenges: LayerChallenges { layers: 2, max_count: 2 }, tree: merkletree-poseidon_hasher-8-0-0 }
    } => hash()
    pub const VERSION: usize = 27;
    pub const GROTH_PARAMETER_EXT: &str = "params";  // generate_random_parameters
    pub const PARAMETER_METADATA_EXT: &str = "meta"; // sector大小
    pub const VERIFYING_KEY_EXT: &str = "vk"; // parameters.vk
    // for example
    v27-stacked-proof-of-replication-merkletree-poseidon_hasher-8-0-0-sha256_hasher-6babf46ce344ae495d558e7770a585b2382d54f225af8ed0397b8be7c3fcd472.meta
    v27-stacked-proof-of-replication-merkletree-poseidon_hasher-8-0-0-sha256_hasher-6babf46ce344ae495d558e7770a585b2382d54f225af8ed0397b8be7c3fcd472.params
    v27-stacked-proof-of-replication-merkletree-poseidon_hasher-8-0-0-sha256_hasher-6babf46ce344ae495d558e7770a585b2382d54f225af8ed0397b8be7c3fcd472.vk
    
    let groth_proofs = StackedCompound::<Tree, DefaultPieceHasher>::circuit_proofs(
        &public_inputs,
        vanilla_proofs,
        &compound_public_params.vanilla_params,
        &groth_params,
        compound_public_params.priority,
    ) {
    }; 
}
```

## analysis

|           | pre commit1 | pre commit2 | commit1 | commit2 |
| --------- | ----------- | ----------- | ------- | ------- |
| gpu       |             |             |         |         |
| cpu       |             |             |         |         |
| memory    |             |             |         |         |
| to disk   |             |             |         |         |
| from disk |             |             |         |         |



