# AXON — Contrato de Interface (MDC)

> Parte do [APSE MDC](../README.md) · Versão 1.1.0

## Visão Geral

**AXON** é o motor de geração de hipóteses cognitivas do APSE 1.1. Ele processa estímulos de entrada, cria redes de neurônios esparsos e emite eventos (spike trains) para o ENABLE processar.

- Repositório: [axon-lang](https://github.com/phoenixc13/axon-lang)
- Linguagem: Rust + LLVM IR (compilador AXON)
- Dependências externas: nenhuma em runtime

---

## Tipos Públicos

### `AxonEvent`

Evento atômico emitido por um neurônio.

```rust
#[repr(C)]
pub struct AxonEvent {
  pub neuron_id:  u32,   // [0, n_features)
  pub magnitude:  f32,   // [-1.0, 1.0]
  pub tick_us:    u64,   // microssegundos UTC desde epoch
}
```

| Campo | Tipo | Restrições |
|---|---|---|
| `neuron_id` | `u32` | `< n_features` do frame pai |
| `magnitude` | `f32` | `[-1.0, 1.0]`, NaN proibido |
| `tick_us` | `u64` | Monotônico dentro do frame |

### `AxonFrame`

Agrupamento de eventos de uma janela de hipótese.

```rust
#[repr(C)]
pub struct AxonFrame {
  pub hypothesis: u32,           // índice [0, MAX_HYPOTHESES)
  pub events:     Vec<AxonEvent>, // ordenados por tick_us ASC
  pub n_features: u32,           // largura do espaço de features
}
```

**Invariantes obrigatórias:**
1. `events` ordenados por `tick_us` crescente
2. `neuron_id < n_features` para todo evento
3. `n_features` consistente em toda a sessão
4. `magnitude` ∈ `[-1.0, 1.0]`
5. `hypothesis < MAX_HYPOTHESES` (padrão: 65536)

---

## Interface de Saída

AXON emite `AxonFrame` via:

### Canal em memória (mesmo processo)
```rust
// AXON preenche um ring buffer; ENABLE consome via AxonBridge
axon_engine.emit_frame(frame: AxonFrame);
```

### Canal IPC (processos separados)
- Protocolo: `length-prefixed bincode`
- Socket: Unix domain socket ou TCP
- Endereço padrão: `unix:///tmp/apse-axon.sock`
- Tamanho máximo de frame: 4 MB

---

## Interface de Entrada (Feedback)

AXON aceita `ApexFeedback` do APEX para ajustar geração de hipóteses:

```rust
pub struct ApexFeedback {
  pub session_id:    u64,
  pub reward_signal: f32,        // [-1.0, 1.0]
  pub prune_mask:    Vec<bool>,  // len == n_hypotheses_ativas
  pub timestamp_ms:  u64,
}
```

---

## Ciclo de Vida de uma Sessão

```
1. axon_engine.start_session(config: AxonConfig)
2. loop {
     axon_engine.step()           // processa um tick
     if axon_engine.has_frame() {
       let frame = axon_engine.pop_frame();
       bridge.push(frame);         // envia ao ENABLE
     }
     if let Some(fb) = apex.feedback() {
       axon_engine.apply_feedback(fb);
     }
   }
3. axon_engine.end_session()
```

---

## Constantes

| Constante | Valor padrão | Descrição |
|---|---|---|
| `MAX_HYPOTHESES` | 65536 | Máximo de hipóteses paralelas |
| `MAX_FEATURES` | 16384 | Máximo de features por frame |
| `MAX_EVENTS_PER_FRAME` | 4096 | Limite de eventos por frame |
| `TICK_RESOLUTION_US` | 1 | Resolução mínima em microssegundos |

---

## Compatibilidade de Versão

| MDC Version | axon-lang | Breaking Change |
|---|---|---|
| 1.1.0 | 0.1.x | N/A (inicial) |

---

[← Voltar ao MDC](../README.md) | [ENABLE →](enable.md)
