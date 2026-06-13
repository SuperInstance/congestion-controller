# Congestion Controller — Network Congestion Management Framework

**A congestion controller** regulates the rate at which data is sent over a network to avoid overwhelming intermediate routers and causing packet loss. This crate provides the framework and interface for congestion control algorithms — the logic that determines sending rate based on acknowledged packets, timeouts, and loss signals.

## Why It Matters

Every TCP connection, every QUIC stream, and every RDMA flow uses congestion control. Without it, the internet collapses into congestion collapse — routers drop packets faster than senders can retransmit, and throughput drops to zero. The algorithms here (slow-start, congestion avoidance, fast retransmit) are what make the internet work under load. QUIC (HTTP/3) and BBR (Google's congestion control algorithm) are direct descendants of this research. If you're building a custom transport protocol, a real-time streaming system, or a load balancer with rate limiting, you need congestion control logic.

## How It Works

### The Congestion Control Problem

The fundamental challenge: the sender doesn't know the capacity of the path between it and the receiver. It must *probe* the network to discover available bandwidth, then back off when it detects congestion (packet loss or increasing delay).

### Classic Algorithms

**Slow Start**: Begin with `cwnd = 1` (congestion window = 1 segment). For each ACK received, increase `cwnd` by 1 — this causes exponential growth: `cwnd` doubles every RTT. When `cwnd` reaches the slow-start threshold (`ssthresh`), switch to congestion avoidance.

```
Slow Start:     cwnd += 1 per ACK      → exponential (×2 per RTT)
Congestion Avoidance: cwnd += 1/cwnd per ACK → linear (+1 per RTT)
```

**On Loss (Reno)**:
- `ssthresh = cwnd / 2`
- `cwnd = 1` (timeout) or `cwnd = ssthresh` (3 duplicate ACKs)
- Re-enter slow start or fast recovery

**Mathematical Model**: The sending rate is approximately:

```
rate ≈ cwnd / RTT  (segments per second)
```

The throughput of TCP Reno under random loss probability `p` (Mathis model):

```
B ≤ MSS / (RTT × √p)
```

This means even 1% packet loss (`p = 0.01`) limits throughput to ~10 MSS/RTT — a harsh penalty that BBR addresses by measuring bandwidth directly instead of inferring from loss.

### Complexity

Congestion control state updates are `O(1)` per packet — a few comparisons and arithmetic operations. The algorithm maintains constant state: `cwnd`, `ssthresh`, and a few counters.

## Quick Start

```rust
// This crate currently provides a framework stub.
// A full implementation would expose:

pub struct CongestionController {
    cwnd: u32,
    ssthresh: u32,
    mode: CongestionMode,
}

pub enum CongestionMode {
    SlowStart,
    CongestionAvoidance,
    FastRecovery,
}

impl CongestionController {
    pub fn new() -> Self {
        Self { cwnd: 1, ssthresh: 65535, mode: CongestionMode::SlowStart }
    }

    pub fn on_ack(&mut self) {
        match self.mode {
            CongestionMode::SlowStart => {
                self.cwnd += 1;
                if self.cwnd >= self.ssthresh {
                    self.mode = CongestionMode::CongestionAvoidance;
                }
            }
            CongestionMode::CongestionAvoidance => {
                self.cwnd += 1 / self.cwnd; // ~linear growth
            }
            _ => {}
        }
    }

    pub fn on_loss(&mut self) {
        self.ssthresh = self.cwnd / 2;
        self.cwnd = 1;
        self.mode = CongestionMode::SlowStart;
    }
}
```

## API

| Type / Method | Description |
|---|---|
| `CongestionController` | State machine for congestion window management. |
| `CongestionMode` | `SlowStart`, `CongestionAvoidance`, `FastRecovery`. |
| `on_ack()` | Called when an ACK is received — may increase `cwnd`. `O(1)`. |
| `on_loss()` | Called when loss is detected — halves `ssthresh`, resets `cwnd`. `O(1)`. |

## Architecture Notes

Congestion control is a transport-layer concern in the γ (generation/sending) side of γ + η = C in SuperInstance. It regulates the rate at which fleet instances transmit data, preventing self-induced congestion in high-fanout scenarios. See [SuperInstance Architecture](https://github.com/SuperInstance/SuperInstance/blob/main/ARCHITECTURE.md).

## References

1. Jacobson, V. (1988). *Congestion Avoidance and Control*. SIGCOMM. — The foundational TCP Reno paper.
2. Cardwell, N. et al. (2017). *BBR: Congestion-Based Congestion Control*. ACM Queue 14(5). — Google's modern congestion control.
3. RFC 9002 (2021). *QUIC Loss Detection and Congestion Control*. — Modern transport congestion control.

## License

MIT
