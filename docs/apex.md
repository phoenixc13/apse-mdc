# APEX — Contrato de Interface (MDC)

> Parte do [APSE MDC](../README.md) · Versão 1.1.0

## Visão Geral

**APEX** é o orquestrador do ecossistema APSE 1.1. Recebe `EnablePacket` do ENABLE, toma decisões de roteamento, envia feedback ao AXON e despacha saídas para atuadores externos.

- Repositório: [apex-middleware](https://github.com/phoenixc13/apex-middleware) *(planejado)*
- Linguagem: Rust
- Status: Especificação preliminar

---

## Tipos Públicos

### `ApexConfig`

```rust
pub struct ApexConfig {
  pub session_id:         u64,
  pub feedback_enabled:   bool,    // enviar ApexFeedback ao AXON
  pub output_socket:      String,  // endereço do atuador
  pub max_packet_queue:   usize,   // fila máxima de EnablePackets
  pub reward_decay:       f32,     // decaimento do sinal de reforço [0,1]
}
```

### `ApexDecision`

Decisão tomada pelo APEX após processar um `EnablePacket`.

```rust
pub struct ApexDecision {
  pub session_id:    u64,
  pub iteration:     u64,          // espelha EnablePacket.iteration
  pub action:        ApexAction,
  pub confidence:    f32,          // [0.0, 1.0]
  pub timestamp_ms:  u64,
}

pub enum ApexAction {
  /// Encaminhar saída para atuador externo.
  Forward { payload: Vec<f32> },
  /// Solicitar mais dados (aguardar próxima iteração).
  Wait,
  /// Encerrar sessão.
  Terminate { reason: String },
  /// Enviar feedback de reforço ao AXON.
  Feedback(ApexFeedback),
}
```

### `ApexFeedback`

Feedback enviado de volta ao AXON.

```rust
pub struct ApexFeedback {
  pub session_id:    u64,
  pub reward_signal: f32,          // [-1.0, 1.0] -- reforço/punição
  pub prune_mask:    Vec<bool>,    // hipóteses a desativar
  pub timestamp_ms:  u64,
}
```

**Invariantes:**
- `reward_signal` ∈ `[-1.0, 1.0]`
- `prune_mask.len()` deve igualar o número de hipóteses ativas em AXON
- `confidence` ∈ `[0.0, 1.0]`

---

## Interface Pública do ApexMiddleware

```rust
impl ApexMiddleware {
  pub fn new(config: ApexConfig) -> Self;

  // Processa um EnablePacket e retorna uma decisão
  pub fn process(&mut self, packet: EnablePacket) -> ApexDecision;

  // Envia feedback diretamente ao AXON
  pub fn send_feedback(&self, fb: ApexFeedback);

  // Despacha para atuador externo
  pub fn dispatch(&self, action: &ApexAction);

  // Estado
  pub fn session_id(&self) -> u64;
  pub fn packets_processed(&self) -> u64;
}
```

---

## Ciclo de Vida

```
1. let apex = ApexMiddleware::new(config);

2. loop {
     // recebe pacote do ENABLE
     let packet = enable_rx.recv();

     let decision = apex.process(packet);

     match &decision.action {
       ApexAction::Forward { payload } => {
         apex.dispatch(&decision.action);
       }
       ApexAction::Feedback(fb) => {
         apex.send_feedback(fb.clone());
         // AXON recebe e ajusta geração de hipóteses
       }
       ApexAction::Terminate { .. } => break,
       ApexAction::Wait => {}
     }
   }
```

---

## Regras de Roteamento (Padrão)

| Condição | Ação |
|---|---|
| `confidence >= 0.85` | `Forward` → atuador |
| `0.50 <= confidence < 0.85` | `Feedback` com reward positivo |
| `confidence < 0.50` | `Feedback` com reward negativo |
| `loss > MAX_LOSS` | `Terminate` |
| `iteration % FEEDBACK_INTERVAL == 0` | `Feedback` periódico |

---

## Constantes

| Constante | Valor padrão | Descrição |
|---|---|---|
| `CONFIDENCE_THRESHOLD` | 0.85 | Limiar para Forward |
| `MAX_LOSS` | 10.0 | Loss máxima antes de Terminate |
| `FEEDBACK_INTERVAL` | 10 | Iterações entre feedbacks periódicos |
| `MAX_PACKET_QUEUE` | 256 | Fila máxima de pacotes |

---

## Compatibilidade de Versão

| MDC Version | apex-middleware | Breaking Change |
|---|---|---|
| 1.1.0 | 0.1.x (planejado) | N/A (inicial) |

---

[← ENABLE](enable.md) | [Voltar ao MDC](../README.md) | [Wire Format →](wire-format.md)
