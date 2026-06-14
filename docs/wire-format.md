# Wire Format — Serialização entre Componentes APSE 1.1

> Parte do [APSE MDC](../README.md) · Versão 1.1.0

## Visão Geral

Este documento define como os dados são serializados e transportados entre os componentes AXON, ENABLE e APEX.

---

## Canais de Transporte

| Canal | Uso | Protocolo |
|---|---|---|
| **In-process** | AXON → ENABLE (mesmo binário) | Ring buffer / ptr direto |
| **Unix Socket** | Entre processos na mesma máquina | `unix:///tmp/apse-*.sock` |
| **TCP** | Componentes em máquinas diferentes | `tcp://host:port` |

---

## Protocolo de Mensagem (length-prefixed)

Todo frame enviado por socket usa o formato:

```
┌────────────────┐┌──────────────────────────────────────────────┐
│ length: u32 BE │            payload (N bytes)                │
└────────────────┘└──────────────────────────────────────────────┘
   4 bytes                    length bytes
```

- `length` = tamanho do payload em bytes, big-endian uint32
- `payload` = struct serializada (ver abaixo)
- Máximo: 4 MB por mensagem

---

## Formato de Serialização

### Padrão: `bincode` (Rust)

Para comunicação entre componentes Rust, o formato padrão é `bincode` com as configurações:

```rust
let config = bincode::config::standard()
  .with_big_endian()
  .with_fixed_int_encoding();
```

### Alternativa: JSON (debug/interop)

Quando a feature `serde` está ativada em `enable-lib`, `EnablePacket` pode ser serializado como JSON:

```json
{
  "output": [0.12, 0.88, 0.45, ...],
  "shape": [4, 8],
  "iteration": 42,
  "timestamp_ms": 1750000000000
}
```

---

## Layouts de Struct em Wire

### `AxonEvent` (14 bytes)

```
Offset  Size  Field
0       4     neuron_id   (u32 BE)
4       4     magnitude   (f32 IEEE 754 BE)
8       6     tick_us     (u64 BE, apenas 6 bytes usados, 2 bytes de padding)
```

> Nota: Em bincode padrão, `u64` ocupa 8 bytes. O layout acima é ilustrativo.

### `AxonFrame` (variável)

```
Offset  Size      Field
0       4         hypothesis     (u32 BE)
4       4         n_features     (u32 BE)
8       8         events_count   (u64 BE)
16      14 * N    events[]       (N × AxonEvent)
```

### `EnablePacket` (variável)

```
Offset  Size        Field
0       8           iteration      (u64 BE)
8       8           timestamp_ms   (u64 BE)
16      8           shape_dims     (u64 BE -- número de dimensões)
24      8 * D       shape[]        (D × u64 BE)
24+8D   8           output_len     (u64 BE)
32+8D   4 * L       output[]       (L × f32 BE)
```

### `ApexFeedback` (variável)

```
Offset  Size        Field
0       8           session_id     (u64 BE)
8       4           reward_signal  (f32 BE)
12      8           timestamp_ms   (u64 BE)
20      8           mask_len       (u64 BE)
28      1 * M       prune_mask[]   (M × bool/u8)
```

---

## Endian

Todos os campos multi-byte são **big-endian** no wire. Componentes em plataformas little-endian devem converter.

---

## Endereços Padrão

| Canal | Endereço padrão | Observações |
|---|---|---|
| AXON → ENABLE | `unix:///tmp/apse-axon.sock` | Apenas leitura pelo ENABLE |
| ENABLE → APEX | `unix:///tmp/apse-enable.sock` | Apenas leitura pelo APEX |
| APEX → AXON | `unix:///tmp/apse-apex.sock` | Apenas leitura pelo AXON |
| APEX → Atuador | `tcp://0.0.0.0:7100` | Configurável via `ApexConfig` |

---

## Controle de Fluxo

- Se o consumidor não ler rápido o suficiente, o produtor **descarta** frames antigos (ring buffer overwrite).
- `EnablePacket` nunca é descartado -- backpressure é aplicado ao ENABLE se APEX estiver lento.
- Máximo de `MAX_PACKET_QUEUE = 256` EnablePackets em fila no APEX.

---

[← APEX](apex.md) | [Voltar ao MDC](../README.md)
