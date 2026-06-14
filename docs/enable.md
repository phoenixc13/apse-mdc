# ENABLE — Contrato de Interface (MDC)

> Parte do [APSE MDC](../README.md) · Versão 1.1.0

## Visão Geral

**ENABLE** é a camada de refinamento cognitivo do APSE 1.1. Recebe eventos esparsos do AXON, materializa-os em Dynamic Connection Tensors (DCT), executa o grafo de computação e emite pacotes processados para o APEX.

- Repositório: [enable-lib](https://github.com/phoenixc13/enable-lib)
- Linguagem: Rust
- Crate: `enable` v0.1.0

---

## Tipos Públicos

### `DynamicConnectionTensor` (DCT)

Estrutura de dados central do ENABLE. Tensor esparso com máscara de conexões ativas.

```rust
pub struct DynamicConnectionTensor {
  pub shape:     Vec<usize>,    // [Hypotheses, Features, Time, ...]
  pub dtype:     DType,
  pub device:    Device,
  pub data_f32:  Vec<f32>,      // dados principais (flat)
  pub gradient:  Vec<f32>,      // gradientes (flat)
  pub mask:      ConnectionMask, // conexões ativas
  pub branches:  Vec<Branch>,   // ramificações do grafo
}
```

### `EnableConfig`

Configuração do engine.

```rust
pub struct EnableConfig {
  pub max_iterations:  u64,      // iterações antes de parar (0 = infinito)
  pub learning_rate:   f32,      // [0.0, 1.0]
  pub batch_size:      usize,    // frames por lote
  pub device:          Device,
  pub dtype:           DType,
  pub prune_threshold: f32,      // conexões abaixo disso são podadas
}
```

**Invariantes:**
- `learning_rate` ∈ `(0.0, 1.0]`
- `batch_size >= 1`
- `prune_threshold >= 0.0`

### `IterationResult`

Resultado de uma iteração do engine.

```rust
pub struct IterationResult {
  pub iteration:     u64,
  pub loss:          f32,
  pub active_nodes:  usize,
  pub pruned_nodes:  usize,
  pub output_packet: Option<EnablePacket>,
}
```

### `EnablePacket`

Pacote de saída enviado ao APEX.

```rust
pub struct EnablePacket {
  pub output:       Vec<f32>,   // saída do grafo (achatada)
  pub shape:        Vec<usize>, // forma do tensor de saída
  pub iteration:    u64,
  pub timestamp_ms: u64,        // UTC milliseconds
}
```

**Invariantes:**
- `output.len() == shape.iter().product()`
- `iteration` monotônico crescente por sessão
- `timestamp_ms` em UTC

---

## Interface Pública do Engine

```rust
impl EnableEngine {
  // Criação
  pub fn new(config: EnableConfig) -> Self;

  // Execução
  pub fn run(&mut self) -> IterationResult;
  pub fn step(&mut self) -> IterationResult;

  // Estado
  pub fn iteration(&self) -> u64;
  pub fn is_running(&self) -> bool;
  pub fn graph(&self) -> &ComputationGraph;
}
```

---

## Interface do AxonBridge

```rust
impl AxonBridge {
  pub fn new(device: Device, dtype: DType, batch_size: usize) -> Self;

  // Retorna true quando batch está cheio
  pub fn push(&mut self, frame: AxonFrame) -> bool;

  // Drena buffer → nó de entrada no grafo
  pub fn flush(&mut self, graph: &mut ComputationGraph) -> Option<NodeId>;

  pub fn buffered(&self) -> usize;
  pub fn batch_size(&self) -> usize;
}
```

---

## Interface do ComputationGraph

```rust
impl ComputationGraph {
  pub fn new() -> Self;

  // Construção do grafo
  pub fn add_input(shape, dtype, device) -> NodeId;
  pub fn add_add(a, b, shape, dtype, device) -> NodeId;
  pub fn add_matmul(a, b, shape, dtype, device) -> NodeId;
  pub fn add_relu(src, shape, dtype, device) -> NodeId;

  // Execução
  pub fn forward(&mut self) -> Option<NodeId>;
  pub fn zero_grad(&mut self);

  // Introspect
  pub fn len(&self) -> usize;
  pub fn node(&self, id: NodeId) -> Option<&Node>;
  pub fn node_mut(&mut self, id: NodeId) -> Option<&mut Node>;
  pub fn topo_order(&self) -> &[NodeId];
}
```

---

## Ciclo de Vida

```
1. let config  = EnableConfig { batch_size: 32, ... };
2. let engine  = EnableEngine::new(config);
3. let mut bridge = AxonBridge::new(Device::Cpu, DType::F32, 32);

4. loop {
     // recebe frame do AXON
     let frame = axon.pop_frame();
     if bridge.push(frame) {
       bridge.flush(&mut engine.graph_mut());
     }

     let result = engine.step();

     if let Some(packet) = result.output_packet {
       apex.send(packet);  // envia ao APEX
     }
   }
```

---

## Eixos Semânticos do DCT

| Eixo | Índice padrão | Descrição |
|---|---|---|
| `Axis::Hypotheses` | 0 | Número de hipóteses ativas (do AXON) |
| `Axis::Features` | 1 | Espaço de features (neuron_id) |
| `Axis::Time` | 2 | Janela temporal (tick_us agrupado) |
| `Axis::Custom(n)` | n | Livre para expansão |

---

## Compatibilidade de Versão

| MDC Version | enable-lib | Breaking Change |
|---|---|---|
| 1.1.0 | 0.1.x | N/A (inicial) |

---

[← AXON](axon.md) | [Voltar ao MDC](../README.md) | [APEX →](apex.md)
