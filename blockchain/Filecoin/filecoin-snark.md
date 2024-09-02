# snark in filecoin

## groth16

QAP: $$sU*sV-sW = H*T$$

约定：$$ [a]_1 = g_1^a, [b]_2 = g_2^b, [a]_1[b]_2 = e(g_1, g_2)^{ab}$$

+ setup

任选 $$\alpha, \beta, \delta, \gamma, \tau$$, 计算：

$$\sigma_1 = (\alpha, \beta, \delta, \{\tau^i\}_{i=0}^{n-1}, \{\frac{\beta u_i(\tau)+\alpha v_i(\tau)+w_i(\tau)}{\gamma}\}_{i=0}^{l}, \{\frac{\beta u_i(\tau)+\alpha v_i(\tau)+w_i(\tau)}{\delta}\}_{i=l+1}^{m}, \{\frac{\tau^it(\tau)}{\delta}\}_{i=0}^{n-2})$$

$$\sigma_2 = (\beta, \gamma, \delta, \{\tau^i\}_{i=0}^{n-1})$$

$\sigma = ([\sigma_1]_1, [\sigma_2]_2)$ 

+ prove

（$a_1, a_2, ..., a_m$）为输入，选择r，s， 计算：$\pi = ([A]_1, [C]_1,[B]_2)$

$A = \alpha + \sum_{i=0}^{m} a_iu_i(\tau) + r\delta $

$B = \beta + \sum_{i=0}^{m}a_iv_i(\tau) + s\delta$

$C = \frac{\sum_{i=l+1}^m a_i(\beta u_i(\tau) + \alpha v_i(\tau) + w_i(\tau)) + h(\tau)t(\tau)}{\delta} + As + Br - rs\delta$

+ verify

$[A]_1 [B]_2 = [\alpha]_1 [\beta]_2 + \sum_{i=0}^{l} a_i[\frac{\beta u_i(\tau)+\alpha v_i(\tau)+w_i(\tau)}{\gamma}]_1[\gamma]_2 + [C]_1[\delta]_2$

## bellman

+  parameters

$$vk = (g_1^{\alpha},g_1^{\beta},g_2^{\beta},g_2^{\gamma},g_1^{\delta},g_2^{\delta},\{g_1^{\frac{\beta u_i(s)+\alpha v_i(s)+w_i(s)}{\gamma}}\}_{i=0}^{l})$$

$params = (vk,\{g_1^{\frac{s^it(s)}{\delta}}\}_{i=0}^{n-2},\{g_1^{\frac{\beta u_i(s)+\alpha v_i(s)+w_i(s)}{\delta}}\}_{i=l+1}^{m}, \{g_1^{u_i(s)}\}_{i=0}^{<m}, \{g_1^{v_i(s)}\}_{i=0}^{<m}, \{g_2^{v_i(s)}\}_{i=0}^{<m})$  

```rust
// groth parameters
// size = 96*3 + 192*3 + 96*l + 4
pub struct VerifyingKey<E: Engine> { 
    pub alpha_g1: E::G1Affine,
    pub beta_g1: E::G1Affine,
    pub beta_g2: E::G2Affine,
    pub gamma_g2: E::G2Affine,
    pub delta_g1: E::G1Affine,
    pub delta_g2: E::G2Affine,
    pub ic: Vec<E::G1Affine>,
}

// vk size + 96*(n-1) + 96*(m-l) + 96*~m + 96*(<<m) + 192*(<<m)
pub struct Parameters<E: Engine> {
    pub vk: VerifyingKey<E>,
    pub h: Arc<Vec<E::G1Affine>>,
    pub l: Arc<Vec<E::G1Affine>>,
    pub a: Arc<Vec<E::G1Affine>>, // calc first 
    pub b_g1: Arc<Vec<E::G1Affine>>, // calc first 
    pub b_g2: Arc<Vec<E::G2Affine>>, // calc first 
}
```

+ constraints

```rust
struct KeypairAssembly<E: Engine> {
    num_inputs: usize, // input
    num_aux: usize, // aux
    num_constraints: usize, // input + aux
    at_inputs: Vec<Vec<(E::Fr, usize)>>, // U
    bt_inputs: Vec<Vec<(E::Fr, usize)>>, // V
    ct_inputs: Vec<Vec<(E::Fr, usize)>>, // W
    at_aux: Vec<Vec<(E::Fr, usize)>>,
    bt_aux: Vec<Vec<(E::Fr, usize)>>,
    ct_aux: Vec<Vec<(E::Fr, usize)>>,
}

// ConstraintSystem 是一个接口，定义了下面几个函数用于产生不同类型的变量：
impl<E: Engine> ConstraintSystem<E> for KeypairAssembly<E> {
    one(): 产生 Input 类型的变量，索引为 0
    // alloc(): 产生 Aux 类型的变量，索引递增
    fn alloc<F, A, AR>(&mut self, _: A, _: F) -> Result<Variable, SynthesisError>
    {
        let index = self.num_aux;
        self.num_aux += 1;
        self.at_aux.push(vec![]);
        self.bt_aux.push(vec![]);
        self.ct_aux.push(vec![]);
        Ok(Variable(Index::Aux(index)))
    }
    // alloc_input(): 产生 Input 类型的变量，索引递增
    fn alloc_input<F, A, AR>(&mut self, _: A, _: F) -> Result<Variable, SynthesisError>
    {
        let index = self.num_inputs;
        self.num_inputs += 1;
        self.at_inputs.push(vec![]);
        self.bt_inputs.push(vec![]);
        self.ct_inputs.push(vec![]);
        Ok(Variable(Index::Input(index)))
    }
    // enforce(): 构建R1CS
    fn enforce<A, AR, LA, LB, LC>(&mut self, _: A, a: LA, b: LB, c: LC)
    {
        fn eval<E: Engine>(
            l: LinearCombination<E>,
            inputs: &mut [Vec<(E::Fr, usize)>],
            aux: &mut [Vec<(E::Fr, usize)>],
            this_constraint: usize,
        ) {
            for (index, coeff) in l.0 {
                match index {
                    Variable(Index::Input(id)) => inputs[id].push((coeff, this_constraint)),
                    Variable(Index::Aux(id)) => aux[id].push((coeff, this_constraint)),
                }
            }
        }

        eval(
            a(LinearCombination::zero()),
            &mut self.at_inputs,
            &mut self.at_aux,
            self.num_constraints,
        );
        eval(
            b(LinearCombination::zero()),
            &mut self.bt_inputs,
            &mut self.bt_aux,
            self.num_constraints,
        );
        eval(
            c(LinearCombination::zero()),
            &mut self.ct_inputs,
            &mut self.ct_aux,
            self.num_constraints,
        );

        self.num_constraints += 1;
    }
}

// for example
cs.enforce(  
    || "a*b=c",  
    |lc| lc + a,  
    |lc| lc + b,  
    |lc| lc + c  
);

cs.enforce(  
    || "a+b=c",  
    |lc| lc + a + b,  
    |lc| lc + CS::one(),  
    |lc| lc + c  
);
```



+ generate paramters

 1 得到R1CS

```rust
pub fn generate_parameters<E, C>(
    circuit: C,
    g1: E::G1,
    g2: E::G2,
    alpha: E::Fr,
    beta: E::Fr,
    gamma: E::Fr,
    delta: E::Fr, 
    tau: E::Fr, // 不可泄露
) -> Result<Parameters<E>, SynthesisError>
where
    E: Engine,
    C: Circuit<E>,
{
    // 电路转化
    let mut assembly = KeypairAssembly::new();
    // Allocate the "one" input variable, 作为第一个输入a_0 = 1;
    assembly.alloc_input(|| "", || Ok(E::Fr::one()))?;
    // Synthesize the circuit.
    circuit.synthesize(&mut assembly)?;
    // x * 0 = 0, 增加这些约束的原因是为了保证转化后的 QAP 的各个多项式不线性依赖。
    for i in 0..assembly.num_inputs {
        assembly.enforce(|| "", |lc| lc + Variable(Index::Input(i)), |lc| lc, |lc| lc);
    }
    ...
```

2 计算params.h

choose  $\omega, \omega_i = \omega^i \ s.t. \ \omega_n = 1;  t(x) \prod_{i=0}^{n-1} (x-\omega_i), t(\tau) = \tau^n -1$

计算  $\{g_1^{\frac{s^it(s)}{\delta}}\}_{i=0}^{n-2}$

```rust
pub struct EvaluationDomain<E: ScalarEngine, G: Group<E>> {
    coeffs: Vec<G>, // vec![Scalar::<E>(E::Fr::zero()); num_constraints] reseize to 2^exp
    exp: u32,  // 2^exp >= num_constraints
    omega: E::Fr, // 2^exp primitive root_of_unity
    omegainv: E::Fr,  // omega inverse
    geninv: E::Fr, // 
    minv: E::Fr, // m inverse, is 2^exp
}
{
    ...
	let powers_of_tau = vec![Scalar::<E>(E::Fr::zero()); assembly.num_constraints];
    // 计算后长度为 n = 2^{log2(assembly.num_constraints)}
	let mut powers_of_tau = EvaluationDomain::from_coeffs(powers_of_tau)?;
	// 并行计算得到powers_of_tau[i] = tau.pow(i)
    // coeff = t(tau)/delta = (tau^n - 1)/delta
	let mut coeff = powers_of_tau.z(&tau); 
	coeff.mul_assign(&delta_inverse);
    // 计算得到params.h: g_1^{tau^i * t(tau)/delta}
}

```

3 计算u，v 在tau的值， 然后计算param.a， param.b_g1，param.b_g2, params.l, vk.ic

假设$u_l$的值为$(u_{l0},u_{l1},...,u_{lk})$ ，要计算$u_i$对应的$\tau$的值，则需要先用ifft转化为Lagrange多项式，然后再计算在$\tau$的值，这样需要很多次ifft和幂计算

$(\omega_i, \tau^i)_{i=0}^{n-1}$ , ifft得到 $$  \ t(x) = \sum_{i=0}^{n-1} t_ix^i $$

假设$u_l(x) = \sum c_i x^i$  满足 $u_l(\omega_j) = u_{lj}$ ，则

$u_i(\tau) = \sum_{i=0}^{k} c_i \tau^i = \sum_{i=0}^{k} (c_i  \sum_{j=0}^{n-1} t_j \omega_i^j) = \sum_{i=0}^{k} (c_i  \sum_{j=0}^{n-1} t_j \omega_j^i) = \sum_{j=0}^{n-1} (t_j  \sum_{i=0}^{k-1} c_i \omega_j^i) \\= \sum_{j=0}^{n-1} (t_j u_l(\omega_j)) = \sum_{j=0}^{n-1} t_ju_{lj} $ 

从而可以很快计算u，v在$\tau$的取值。

```rust
{
    ...
    // Use inverse FFT to convert to Lagrange coefficients
    // (w_i, tau^i)点值到系数表达
    // t(x) = (t-w_0)(t-w_1)...(t-w_{n-1}) = \sum_{i=0}^{i=n-1} t_i * x^i
    powers_of_tau.ifft(&worker, &mut None)?;
    // 计算
    ...
    fn eval_at_tau<E: Engine>(
        powers_of_tau: &[Scalar<E>],
        p: &[(E::Fr, usize)],
     ) -> E::Fr {
         let mut acc = E::Fr::zero();
         for &(ref coeff, index) in p {
      		powers_of_tau[index].0.mul_assign(coeff);
             acc.add_assign(&n);
         }
         acc
    }
    // Evaluate QAP polynomials at tau
   	let mut at = eval_at_tau(powers_of_tau, at); // at 在tau的值
    let mut bt = eval_at_tau(powers_of_tau, bt); // bt 在tau的值
    let ct = eval_at_tau(powers_of_tau, ct); // ct 在tau的值
    a = g1_wnaf.scalar(at.into_repr()); // param.a
    *b_g1 = g1_wnaf.scalar(bt.into_repr()); // params.b_g1
    *b_g2 = g2_wnaf.scalar(bt.into_repr()); // params.b_g2
    
    at.mul_assign(&beta);
    bt.mul_assign(&alpha);
    let mut e = at;
    e.add_assign(&bt);
    e.add_assign(&ct);
    {
        e.mul_assign(gamma_inverse); 
    	ic = g1_wnaf.scalar(e.into_repr()); // vk.ic
    }
    {
        e.mul_assign(delta_inverse); 
    	l = g1_wnaf.scalar(e.into_repr()); // params.l
    }
}
```

+ prove

1. 由于$t(\omega_i) = 0$,  利用$\omega = \sigma^{(r-1)/n}, \omega^n = 1$, 计算u，v，w(代码中为a，b，c)在$\{\delta \omega^i\}_{i=0}^{n-1}$上的值
2. $t(\sigma \omega_i) = \sigma^n -1$, 然后计算$h(\sigma \omega_i) = \frac{a_i *b_i -c_i}{\sigma^n -1}$ ; iFFT计算得到多项式系数；
3. 根据尺度变换原理 $f(\delta x) = \frac{1}{\delta}F(x)$， 计算得到的系数除以$\sigma$即得到h(x)的系数
4. h(x)的系数乘以params.h对应的值，加和起来就得到$\frac{h(\tau)t(\tau)}{\delta}$
5. exp计算

```rust
fn create_proof_batch_priority_inner<E, C, P: ParameterSource<E>>(
    circuits: Vec<C>,
    params: P,
    r_s: Vec<E::Fr>, // 随机选择
    s_s: Vec<E::Fr>, // 随机选择
    priority: bool,
) -> Result<Vec<Proof<E>>, SynthesisError>
where
    E: Engine,
    C: Circuit<E> + Send,
{
    // R1CS， 和generate parameters中的circuit一样
    let mut provers = circuits{
        let mut prover = ProvingAssignment::new();
        prover.alloc_input(|| "", || Ok(E::Fr::one()))?; // a_0 = 1
        circuit.synthesize(&mut prover)?;
        for i in 0..prover.input_assignment.len() {
            prover.enforce(|| "", |lc| lc + Variable(Index::Input(i)), |lc| lc, |lc| lc);
         }
         Ok(prover)
    })
    
	// 计算h(x) = (su(x)*sv(x)-sw(x))/t(x)
	// 由于t(w_i)=0， 取w=
    let a_s = {
        // sa, sb, sc 3次ifft得到多项式，再3次fft计算在coset上的值，
        a.ifft(&worker, &mut fft_kern)?; // 点值转化为a的系数
        a.coset_fft(&worker, &mut fft_kern)?; // fft计算在coset的值
    	...
        // h = (a_new * b_new - c_new)/ t()
        a.mul_assign(&worker, &b);
        a.sub_assign(&worker, &c);
        a.divide_by_z_on_coset(&worker); 
        // ifft h点值转化为系数 h(x)
        a.icoset_fft(&worker, &mut fft_kern)?;
    })
	// gpu multiexp calculate
    // multiexp n-2 h_i{h(x)}, h(x)的系数作为指数
    // multiexp m l_i^{s_i}
    // multiexp l+m a_i^{s_i}
    // multiexp l+m b_g1_i^{s_i}
    // multiexp l+m b_g1_i^{s_i}
    
    // calc Proof(A,B,C)
}
```

+ verify

在一个proof的时候，验证$(A, B, C)$, 验证等式为：

$-[A]_1 [B]_2 + \sum_{i=0}^{l} a_i[\frac{\beta u_i(\tau)+\alpha v_i(\tau)+w_i(\tau)}{\gamma}]_1[\gamma]_2 + [C]_1([\delta]_2) = -[\alpha]_1 [\beta]_2 $

在有k个proof, 验证$(A_i, B_i, C_i)$, 对应的input为$\{a_{ij}\}_{j=0}^{l}, i=0,1,2..,k-1$

随机选择$\{r_i\}_{i=0}^{k-1}$，$r_{sum} = \sum_{i=0}^{k-1} r_i$ , 验证等式变换为：

$\sum_{i=0}^{k-1}[A_i^{r_i}]_1[B_i^{-1}]_2 + [\sum_{j=0}^{l} ic_j^{\sum_{i=0}^{k-1}r_ia_{ij}}]_1[\gamma]_2 + [\sum_{i=0}^{k-1} C_i^{r_i}]_1[\delta]_2 = ([\alpha]_1[\beta]_2)^{-\sum_{i=0}^{k-1} r_i}$



代码按照如下计算：

combine input： $p_j = \sum_{i=0}^{k-1} r_i*a_{ij}, j=0,1,2..l$，

$acc_{pi}= ic_{0}^{r_{sum}} + \sum_{i=1}^{l}ic_i^{p_i} $ 

$acc_y = ([\alpha]_1 [\beta]_2)^{-r_{sum}}$

$acc_c = \sum_{i=0}^{k-1} C_i^{r_i}$

```rust
// 多个partition的proof可以一起验证
/// Randomized batch verification - see Appendix B.2 in Zcash spec
pub fn verify_proofs_batch<'a, E: Engine, R: rand::RngCore>(
    pvk: &'a BatchPreparedVerifyingKey<E>,
    rng: &mut R,
    proofs: &[&Proof<E>],
    public_inputs: &[Vec<E::Fr>],
) -> Result<bool, SynthesisError>
where
    <<E as ff::ScalarEngine>::Fr as ff::PrimeField>::Repr: From<<E as ff::ScalarEngine>::Fr>,
{
    let pi_num = pvk.ic.len() - 1;
    let proof_num = proofs.len();

    // 随机选择一个数，将不同proof的public_input结合起来
    let mut r: Vec<E::Fr> = Vec::with_capacity(proof_num);
    for _ in 0..proof_num { 
        r.push(E::Fr::random);
    }
    let mut sum_r = E::Fr::zero();
    for i in r.iter() {
        sum_r.add_assign(i);
    }

    // 不同proof的input同一位置结合起来
    let pi_scalars: Vec<_> = (0..pi_num)
        .into_par_iter()
        .map(|i| {
            for j in 0..proof_num {
                // r_j * a_j,i
                r[j].mul_assign(&public_inputs[j][i]);
                pi.add_assign(&tmp);
            }
            pi.into_repr()
        })
        .collect();

    // create group element corresponding to public input combination
    // This roughly corresponds to Accum_Gamma in spec
    let mut acc_pi = pvk.ic[0].mul(sum_r.into_repr());
    acc_pi.add_assign(
        &multiexp(
            &worker,
            (Arc::new(pvk.ic[1..].to_vec()), 0),
            FullDensity,
            Arc::new(pi_scalars),
            &mut multiexp_kern,
        )
        .wait()
        .unwrap(),
    );

    // This corresponds to Accum_Y
    // -Accum_Y
    sum_r.negate();
    // This corresponds to Y^-Accum_Y
    let acc_y = pvk.alpha_g1_beta_g2.pow(&sum_r.into_repr());

    // This corresponds to Accum_Delta
    let mut acc_c = E::G1::zero();
    for (rand_coeff, proof) in r.iter().zip(proofs.iter()) {
        let mut tmp: E::G1 = proof.c.into();
        tmp.mul_assign(*rand_coeff);
        acc_c.add_assign(&tmp);
    }

    // This corresponds to Accum_AB
    let ml = r
        .par_iter()
        .zip(proofs.par_iter())
        .map(|(rand_coeff, proof)| {
            // [z_j] pi_j,A
            let mut tmp: E::G1 = proof.a.into();
            tmp.mul_assign(*rand_coeff);
            let g1 = tmp.into_affine().prepare();

            // -pi_j,B
            let mut tmp: E::G2 = proof.b.into();
            tmp.negate();
            let g2 = tmp.into_affine().prepare();

            (g1, g2)
        })
        .collect::<Vec<_>>();
    let mut parts = ml.iter().map(|(a, b)| (a, b)).collect::<Vec<_>>();

    // MillerLoop(Accum_Delta)
    let acc_c_prepared = acc_c.into_affine().prepare();
    parts.push((&acc_c_prepared, &pvk.delta_g2));

    // MillerLoop(\sum Accum_Gamma)
    let acc_pi_prepared = acc_pi.into_affine().prepare();
    parts.push((&acc_pi_prepared, &pvk.gamma_g2));

    let res = E::miller_loop(&parts);
    Ok(E::final_exponentiation(&res).unwrap() == acc_y)
}
```

## 应用

```rust
// 定义一个结构，包含一些验证用的数据，如provder_id等
pub struct XXXCircuit {
}
// 提供一个circuit方法，构建XXXCircuit数据
fn circuit()
// synthesize方法，写明验证步骤（和verify proof函数很像），验证函数转化为R1CS
fn synthesize()
// 对应synthesize的要求的input，生成用于verify的输入
fn generate_public_inputs() Vec<Fr>

```

## stacked

+ paramters

for 2k sector size， c challenges per partition ; 

l = num_inputs = 1+ 3 +18c; 

m = num_inputs+ num_aux (取决于layer个数，challenge个数) ~= 1+ 3 +18*c+ 6081228*c; 

n = 大于(num_constraints)的2的指数 = (4423837) 8388608

$vk\ size = 96*43+ 192*3 + 4= 4708\ bytes$

$param\ size = 4708 + 96*(8388608-1+ 4420837+ 4385080 + 1174174) + 192*1174174 + 5*4 = 1988841144 $  

+ prove constraint

```rust
// 将vanilla proof 的每个challenge对应的proof 转化为 proof of a single challenge；用于sythnesize输入
pub struct Proof<Tree: MerkleTreeTrait, G: Hasher> {
    /// Inclusion path for the challenged data node in tree D.
    pub comm_d_path: AuthPath<G, U2, U0, U0>,
    /// The value of the challenged data node.
    pub data_leaf: Option<Fr>, // 1. as private input, verify with comm_d 
    /// The index of the challenged node.
    pub challenge: Option<u64>, // 4. public input, 
    /// Inclusion path of the challenged replica node in tree R.
    pub comm_r_last_path: TreeAuthPath<Tree>,
    /// Inclusion path of the column hash of the challenged node  in tree C.
    pub comm_c_path: TreeAuthPath<Tree>,
    /// Column proofs for the drg parents.
    pub drg_parents_proofs: Vec<TreeColumnProof<Tree>>, // 2. as private input, verify with comm_c
    /// Column proofs for the expander parents.
    pub exp_parents_proofs: Vec<TreeColumnProof<Tree>>, // 3. as private input, verify with comm_c
    _t: PhantomData<Tree>,
}

// construct circuits and then synthesize
// inputs: 1 + 3(public inputs) + 18*challenge_per_partition (statements?)
fn synthesize<CS: ConstraintSystem<Bls12>>(self, cs: &mut CS) -> Result<(), SynthesisError> {
    // Allocate replica_id, comm_d, comm_r as Fr, make them public inputs （3）
    let replica_id_num = num::AllocatedNum::alloc(cs, || {replica_id})?; // Allocate replica_id
    replica_id_num.inputize(cs.namespace(|| "replica_id_input"))?; // make replica_id a public input
	// Allocate comm_r_last as Fr and comm_c as Fr; verify comm_r = H(comm_c || comm_r_last)
    for (i, proof) in proofs.into_iter().enumerate() { // 对于每一个挑战
        proof.synthesize(
        	&comm_d_num, // as input for verify original data
        	&comm_c_num, // as input to verify collum hash
        	&comm_r_last_num, // as input to verify encoded data
        	&replica_id_bits, // as input to create_labels
        ){
        // 确保proof不为空
        assert!(!drg_parents_proofs.is_empty());
        assert!(!exp_parents_proofs.is_empty());

        // data_leaf as PrivateInput; verify the data leaf in the tree D
        // output position as a statement 
        enforce_inclusion(cs,comm_d_path,comm_d,&data_leaf_num,)?;

        // -- verify replica column openings; 
        // output 6 statements
        let mut drg_parents = Vec::with_capacity(layers);
       	for (i, parent) in drg_parents_proofs.into_iter().enumerate() {
            let (parent_col, inclusion_path) =
            parent.alloc(cs.namespace(|| format!("drg_parent_{}_num", i)))?;
            assert_eq!(layers, parent_col.len());
            // calculate column hash
            let val = parent_col.hash(cs.namespace(|| format!("drg_parent_{}_constraint", i)))?;
            // enforce inclusion of the column hash in the tree C, output position as witness 
            enforce_inclusion(cs, inclusion_path, comm_c, &val,)?;
            drg_parents.push(parent_col);
        }

        // output 8 statements
        let mut exp_parents = Vec::new();
        for (i, parent) in exp_parents_proofs.into_iter().enumerate() {
            // do same as drg parents
            exp_parents.push(parent_col);
        }

        // challenge index as a statement 
        let challenge_num = uint64::UInt64::alloc(cs.namespace(|| "challenge"), challenge)?;
        challenge_num.pack_into_input(cs.namespace(|| "challenge input"))?;

        // Collect the parents; and expanded to 38, reconstruct the label at layer 
        for layer in 1..=layers {
            let label = create_label(cs, replica_id, expanded_parents, layer_num, challenge_num.clone())?;
            column_labels.push(label);
        }

        // encode the node, verify inclusion of the encoded node, output a statement
        let encoded_node = encode(cs, &column_labels[column_labels.len() - 1] , &data_leaf_num)?;
        enforce_inclusion(cs, comm_r_last_path, comm_r_last, &encoded_node)?;
 
        // calculate column_hash of labels
        let column_hash = hash_single_column(cs, &column_labels)?;
	   // enforce inclusion of the column hash in the tree C, output a statement
        enforce_inclusion(cs, comm_c_path, comm_c, &column_hash, )?;
        }    
        Ok(())
    }
}
```

+ verify

```rust
// total public inputs num = 3 + 18*challenges; 
// 对应a_1-a_{constrains.input_num}; a_0 = 1
public inputs vec<Fr>{
    replica_id;
    comm_d;
    comm_r;
    vec<proof input, challenges>
}

// each proof has 18 witness inputs 
public proof input vec<Fr>{
    data_leaf postion; //1 Fr
    drg parents position; //6 Fr
    exp parents position; // 8 Fr
    challenge postion; // 1 Fr
    encoded data position; // 1 Fr
    labels position; // Fr
}

// 作为verify的a_i参数，用于验证proof的正确性
fn generate_public_inputs(
    pub_in: &<StackedDrg<Tree, G> as ProofScheme>::PublicInputs,
    pub_params: &<StackedDrg<Tree, G> as ProofScheme>::PublicParams,
    k: Option<usize>,
) -> Result<Vec<Fr>> {
    let graph = &pub_params.graph;
    let mut inputs = Vec::new();
    let replica_id = pub_in.replica_id;
    inputs.push(replica_id.into());
    let comm_d = pub_in.tau.as_ref().expect("missing tau").comm_d;
    inputs.push(comm_d.into());\
    let comm_r = pub_in.tau.as_ref().expect("missing tau").comm_r;
    inputs.push(comm_r.into());
    
    let por_setup_params = por::SetupParams {
        leaves: graph.size(),
        private: true,
    };
    let por_params = por::PoR::<Tree>::setup(&por_setup_params)?;
    let por_params_d = por::PoR::<BinaryMerkleTree<G>>::setup(&por_setup_params)?;
    let all_challenges = pub_in.challenges(&pub_params.layer_challenges, graph.size(), k);
    for challenge in all_challenges.into_iter() {
        // comm_d inclusion proof for the data leaf
        inputs.extend(generate_inclusion_inputs::<BinaryMerkleTree<G>>(&por_params_d,challenge,k,)?);
        // drg parents
        let mut drg_parents = vec![0; graph.base_graph().degree()];
        graph.base_graph().parents(challenge, &mut drg_parents)?;
        // Inclusion Proofs: drg parent node in comm_c
        for parent in drg_parents.into_iter() {
            inputs.extend(generate_inclusion_inputs::<Tree>(&por_params, parent as usize, k, )?); 
        }
        // exp parents
        let mut exp_parents = vec![0; graph.expansion_degree()];
        graph.expanded_parents(challenge, &mut exp_parents)?;
        // Inclusion Proofs: expander parent node in comm_c
        for parent in exp_parents.into_iter() {
            inputs.extend(generate_inclusion_inputs::<Tree>(&por_params, parent as usize, k, )?); 
        }
        inputs.push(u64_into_fr(challenge as u64));
        // Inclusion Proof: encoded node in comm_r_last
        inputs.extend(generate_inclusion_inputs::<Tree>(&por_params, challenge, k, )?);
        // Inclusion Proof: column hash of the challenged node in comm_c
        inputs.extend(generate_inclusion_inputs::<Tree>(&por_params, challenge, k, )?);
    }
    Ok(inputs)
}

// generate_inclusion_inputs for each por inclusion
fn generate_public_inputs(
    pub_inputs: &<PoR<Tree> as ProofScheme<'a>>::PublicInputs,
    pub_params: &<PoR<Tree> as ProofScheme<'a>>::PublicParams,
    _k: Option<usize>,
) -> Result<Vec<Fr>> {
    let mut inputs = Vec::new();
    let path_bits = challenge_into_auth_path_bits(pub_inputs.challenge, pub_params.leaves);
    inputs.extend(multipack::compute_multipacking::<Bls12>(&path_bits));
    if let Some(commitment) = pub_inputs.commitment {
        ensure!(!pub_params.private, "Params must be public");
        inputs.push(commitment.into());
    } else {
        ensure!(pub_params.private, "Params must be private");
    }
    Ok(inputs)
}
```



## window proof

+ prove constraint

```rust
/// This is the `FallbackPoSt` circuit.
pub struct FallbackPoStCircuit<Tree: MerkleTreeTrait> {
    pub prover_id: Option<Fr>,
    pub sectors: Vec<Sector<Tree>>,
}

pub struct Sector<Tree: MerkleTreeTrait> {
    pub comm_r: Option<Fr>,
    pub comm_c: Option<Fr>,
    pub comm_r_last: Option<Fr>,
    pub leafs: Vec<Option<Fr>>,
    pub paths: Vec<AuthPath<Tree::Hasher, Tree::Arity, Tree::SubTreeArity, Tree::TopTreeArity>>,
    pub id: Option<Fr>,
}

// 构建sector circuit, synthesize
fn synthesize<CS: ConstraintSystem<Bls12>>(self, cs: &mut CS) -> Result<(), SynthesisError> {
    let Sector {comm_r, comm_c, comm_r_last, leafs, paths, .. } = self;
    assert_eq!(paths.len(), leafs.len());
    // 1. Verify comm_r
    let comm_r_last_num = num::AllocatedNum::alloc(cs, || {comm_r_last})?;
    let comm_c_num = num::AllocatedNum::alloc(cs, || {comm_c})?;
    let comm_r_num = num::AllocatedNum::alloc(cs, || { comm_r})?;
    comm_r_num.inputize(cs.namespace(|| "comm_r_input"))?;
    // 1. Verify H(Comm_C || comm_r_last) == comm_r
    {
        let hash_num = <Tree::Hasher as Hasher>::Function::hash2_circuit(
            cs.namespace(|| "H_comm_c_comm_r_last"), &comm_c_num, &comm_r_last_num,
        )?;
        // Check actual equality
        constraint::equal(cs, || "enforce_comm_c_comm_r_last_hash_comm_r", &comm_r_num, &hash_num,);
    }

    // 2. Verify Inclusion Paths
    for (i, (leaf, path)) in leafs.iter().zip(paths.iter()).enumerate() {
        PoRCircuit::<Tree>::synthesize(
            cs.namespace(|| format!("challenge_inclusion_{}", i)),
            Root::Val(*leaf),
            path.clone(),
            Root::from_allocated::<CS>(comm_r_last_num.clone()),
            true,
        )?;
    }
    Ok(())
}
```



+ verify

```rust
fn generate_public_inputs(
    pub_inputs: &<FallbackPoSt<'a, Tree> as ProofScheme<'a>>::PublicInputs,
    pub_params: &<FallbackPoSt<'a, Tree> as ProofScheme<'a>>::PublicParams,
    partition_k: Option<usize>,
) -> Result<Vec<Fr>> {
    let mut inputs = Vec::new();
    let por_pub_params = por::PublicParams {
        leaves: (pub_params.sector_size as usize / NODE_SIZE),
        private: true,
    };
    
    let num_sectors_per_chunk = pub_params.sector_count;
    let partition_index = partition_k.unwrap_or(0);
    let sectors = pub_inputs
    	.sectors
    	.chunks(num_sectors_per_chunk)
    	.nth(partition_index)
    	.ok_or_else(|| anyhow!("invalid number of sectors/partition index"))?;

    for (i, sector) in sectors.iter().enumerate() {
        // 1. Inputs for verifying comm_r = H(comm_c || comm_r_last)
        inputs.push(sector.comm_r.into());

        // 2. Inputs for verifying inclusion paths
        for n in 0..pub_params.challenge_count {
            let challenge_index = ((partition_index * pub_params.sector_count + i) 
                * pub_params.challenge_count + n) as u64;
            let challenged_leaf_start = fallback::generate_leaf_challenge(
                &pub_params,
                pub_inputs.randomness,
                sector.id.into(),
                challenge_index,
            )?;

            let por_pub_inputs = por::PublicInputs {
                commitment: None,
                challenge: challenged_leaf_start as usize,
            };
            let por_inputs = PoRCompound::<Tree>::generate_public_inputs(
                &por_pub_inputs,
                &por_pub_params,
                partition_k,
            )?;
            inputs.extend(por_inputs);
        }
    }
    
    let num_inputs_per_sector = inputs.len() / sectors.len();
    // duplicate last one if too little sectors available
    while inputs.len() / num_inputs_per_sector < num_sectors_per_chunk {
        let s = inputs[inputs.len() - num_inputs_per_sector..].to_vec();
        inputs.extend_from_slice(&s);
    }
    assert_eq!(inputs.len(), num_inputs_per_sector * num_sectors_per_chunk);
	Ok(inputs)
}
```



