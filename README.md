# APSE 1.1 — Model Description Contract (MDC)

> **Versão:** 1.1.0 · **Data:** 2026-06-14 · **Status:** Pre-Alpha

| Componente | Repositório | Linguagem | Status |
|---|---|---|---|
| AXON | [axon-lang](https://github.com/phoenixc13/axon-lang) | Rust / LLVM IR | Pre-Alpha |
| ENABLE | [enable-lib](https://github.com/phoenixc13/enable-lib) | Rust | Pre-Alpha |
| APEX | [apex-middleware](https://github.com/phoenixc13/apex-middleware) | Rust | Planejado |
| MDC | [apse-mdc](https://github.com/phoenixc13/apse-mdc) | Markdown / JSON | Pre-Alpha |

---

## O que é o MDC?

O **Model Description Contract** (MDC) é o documento normativo central do ecossistema APSE 1.1. Ele define:

- As **interfaces públicas** de cada componente (tipos, funções, mensagens)
- Os **formatos de wire** (serialização entre processos/máquinas)
- As **invariantes de sistema** que todos os componentes devem respeitar
- O **fluxo de dados** canonical: `AXON → ENABLE → APEX`

Nenhum componente pode mudar sua API pública sem atualizar este documento primeiro.

---

## Arquitetura APSE 1.1

```
┌─────────────────────────────────────────────────────────────┐
│                     APSE 1.1 ECOSYSTEM                      │
│                                                             │
│  ┌──────────┐   AxonFrame    ┌──────────┐  EnablePacket    │
│  │          │ ─────────────► │          │ ──────────────►  │
│  │  AXON    │                │  ENABLE  │                  │
│  │          │ ◄───────────── │          │ ◄────────────── ┐│
│  │ axon-lang│   (feedback)   │enable-lib│                  ││
│  └──────────┘                └──────────┘      ┌──────────┐││
│   Spike Engine               DCT + Graph        │  APEX    │││
│   Hypothesis gen.            Local backprop      │middleware│◄┘│
│   Zero-copy events           Cognitive refine.   └──────────┘  │
│                                                   Orchestrator  │
└─────────────────────────────────────────────────────────────────┘
```

### Responsabilidades

| Componente | Responsabilidade Principal |
|---|---|
| **AXON** | Gerar hipóteses cognitivas como eventos esparsos (spike trains). Compilar programas AXON para bytecode. |
| **ENABLE** | Receber eventos AXON, materializá-los em Dynamic Connection Tensors (DCT), executar forward/backward pass local, emitir `EnablePacket`. |
| **APEX** | Receber `EnablePacket` do ENABLE, orquestrar decisões, rotear saídas para atuadores ou feedback para AXON. |

---

## Contratos de Interface

### 1. AXON → ENABLE: `AxonFrame`

```rust
/// Um lote de eventos de um único janela de hipótese.
struct AxonFrame {
  hypothesis: u32,        // índice da hipótese
  events:     Vec<AxonEvent>,
  n_features: u32,        // largura do espaço de features
}

struct AxonEvent {
  neuron_id:  u32,        // neurônio/feature que disparou
  magnitude:  f32,        // intensidade de ativação
  tick_us:    u64,        // timestamp em microssegundos
}
```

**Invariantes:**
- `events` deve estar ordenado por `tick_us` crescente
- `neuron_id < n_features` para todo evento
- `magnitude` deve estar em `[-1.0, 1.0]`
- `n_features` deve ser consistente entre frames de uma mesma sessão

---

### 2. ENABLE → APEX: `EnablePacket`

```rust
/// Pacote serializável que ENABLE envia ao APEX após cada forward pass.
struct EnablePacket {
  output:       Vec<f32>,   // saída do grafo (achatada)
  shape:        Vec<usize>, // forma do tensor de saída
  iteration:    u64,        // número de iteração do EnableEngine
  timestamp_ms: u64,        // timestamp wall-clock (ms desde epoch Unix)
}
```

**Invariantes:**
- `output.len() == shape.iter().product()`
- `iteration` deve ser monotonicamente crescente por sessão
- `timestamp_ms` deve ser UTC

---

### 3. APEX → AXON: Feedback (planejado)

```rust
/// Sinal de feedback do APEX para AXON ajustar geração de hipóteses.
struct ApexFeedback {
  session_id:    u64,
  reward_signal: f32,       // [-1.0, 1.0] — reforço/punição
  prune_mask:    Vec<bool>, // hipóteses a desativar
  timestamp_ms:  u64,
}
```

---

## Tipos Compartilhados

```rust
/// Dispositivo de computação.
enum Device { Cpu, Cuda(u8), Metal }

/// Tipo de dado do tensor.
enum DType  { F32, F16, BF16, I8, Bool }

/// Eixo semântico do DCT.
enum Axis {
  Hypotheses,      // dim 0 -- número de hipóteses ativas
  Features,        // dim 1 -- espaço de features AXON
  Time,            // dim 2 -- janela temporal
  Custom(usize),   // dimensão livre
}
```

---

## Fluxo de Dados Canonical

```
[AXON]
  │  Gera AxonEvent(neuron_id, magnitude, tick_us)
  │  Agrupa em AxonFrame(hypothesis, events[], n_features)
  ▼
[AxonBridge @ ENABLE]
  │  Acumula frames até batch_size
  │  Projeta sparse → dense: DCT[h, f, t]
  ▼
[ComputationGraph @ ENABLE]
  │  forward() → ReLU, MatMul, Softmax...
  │  backward() → zero_grad, gradiente local
  ▼
[EnablePacket]
  │  output: Vec<f32>, shape, iteration, timestamp_ms
  ▼
[APEX]
  │  Roteia decisões
  │  Gera ApexFeedback → AXON (loop fechado)
  ▼
[Atuadores / Saída do Sistema]
```

---

## Documentos Detalhados

| Documento | Descrição |
|---|---|
| [docs/axon.md](docs/axon.md) | Contrato completo do AXON |
| [docs/enable.md](docs/enable.md) | Contrato completo do ENABLE |
| [docs/apex.md](docs/apex.md) | Contrato completo do APEX |
| [docs/wire-format.md](docs/wire-format.md) | Serialização entre componentes |
| [APSE.schema.json](APSE.schema.json) | Schema JSON machine-readable |

---

## Versionamento do MDC

Este documento segue **Semantic Versioning**:
- **MAJOR**: mudança incompatível de wire format
- **MINOR**: novo campo opcional ou novo componente
- **PATCH**: correção de documentação, sem mudança de ABI

Qualquer PR que altere uma interface pública de qualquer componente **deve** incluir uma atualização correspondente neste repositório.

---

## Licença

MIT © 2026 phoenixc13
