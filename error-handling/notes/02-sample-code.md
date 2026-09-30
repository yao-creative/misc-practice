## Intent

This is **provenance mapping** (axis → exemplar artifact) over the earlier design space $D = \prod_i X_i$. Each company gets one primary artifact and a minimal sample that instantiates its dominant axis.

I'm citing repos and documents by name from memory and did not fetch them. Deep file paths are the most likely thing to have moved, so I stay at repo and document level. Verify links before you rely on them.

## Citation map

| Company | Dominant axis | Primary artifact |
|---|---|---|
| Google | Representation | `abseil/abseil-cpp` (`absl/status/status.h`, `statusor.h`); `googleapis/googleapis` (`google/rpc/status.proto`, `code.proto`) |
| Amazon | Blast radius, retry discipline | Amazon Builders' Library: "Timeouts, retries, and backoff with jitter"; AWS Architecture Blog: "Exponential Backoff And Jitter"; `awslabs/smithy-rs` for SDK retry config |
| Netflix | Recovery mechanism | `Netflix/Hystrix` (maintenance mode); `resilience4j/resilience4j` (the active successor); `Netflix/chaosmonkey` |
| Ericsson / WhatsApp | Containment via supervision | `erlang/otp` (`supervisor` in stdlib). WhatsApp's own code is closed, so OTP is the citable substrate. |
| Stripe | Idempotent retry | Stripe API docs on idempotent requests; Stripe engineering blog: "Designing robust and predictable APIs with idempotency" |
| NASA/JPL | Prevention | Holzmann, "The Power of 10: Rules for Developing Safety-Critical Code"; JPL Institutional Coding Standard for C; `nasa/fprime`, `nasa/cFS` |
| Cloudflare | Representation vs. discipline | `cloudflare/pingora`; Cloudflare's post-mortem for the November 18, 2025 outage |

## Samples

**Google: error as a coproduct value** ($B \sqcup E$ via `StatusOr`):

```cpp
absl::StatusOr<Config> Load(absl::string_view path) {
  if (path.empty()) return absl::InvalidArgumentError("empty path");
  return Parse(path);
}

auto cfg = Load(p);
if (!cfg.ok()) return cfg.status();   // forwards inr, Kleisli-style
```

**Amazon: full-jitter backoff.** This is the delay function from the AWS blog as a pure function with no owned state. The random draw $u \in [0,1)$ is passed in so the function stays referentially transparent:

$$\mathrm{delay}(n, u) = u \cdot \min\!\big(\text{cap},\ \text{base}\cdot 2^{n}\big)$$

```rust
fn delay_ms(attempt: u32, base_ms: u64, cap_ms: u64, u: f64) -> u64 {
    let ceiling = cap_ms.min(base_ms.saturating_mul(1u64 << attempt.min(32)));
    (ceiling as f64 * u) as u64
}
```

**Netflix: circuit breaker in resilience4j:**

```java
CircuitBreaker cb = CircuitBreaker.ofDefaults("inventory");
Supplier<String> guarded =
    CircuitBreaker.decorateSupplier(cb, () -> client.fetch());
Try.ofSupplier(guarded).recover(t -> "cached-fallback");
```

**Erlang/OTP: recovery moved into the tree:**

```erlang
init([]) ->
    SupFlags = #{strategy => one_for_one, intensity => 5, period => 10},
    Child = #{id => worker, start => {worker, start_link, []}},
    {ok, {SupFlags, [Child]}}.
```

`intensity` and `period` bound restarts, which is the containment rule. Exceeding them escalates failure to the parent.

**Stripe: retry made safe.** The idempotency key makes the operation's effect a function of the key, so repeated calls equal one call:

$$\mathrm{apply}(k, r) \circ \mathrm{apply}(k, r) = \mathrm{apply}(k, r)$$

```bash
curl https://api.stripe.com/v1/charges \
  -u "$SK:" -H "Idempotency-Key: 6f1c9e-order-4821" \
  -d amount=2000 -d currency=usd -d source=tok_visa
```

**NASA/JPL: prevention.** Power of 10, Rule 7 requires checking every non-void return value and every parameter's validity:

```c
if (sensor_read(&reading) != STATUS_OK) { enter_safe_mode(); return; }
```

**Cloudflare: the `unwrap` gap.** As I understand the post-mortem, an oversized configuration input reached an `unwrap()` and panicked. The contrast, borrowing and returning a value with no owned state:

```rust
// Panic: the type admits the error, the call site discards it
fn load_bad(raw: &str) -> Features { parse(raw).unwrap() }

// Total: failure stays in B ⊔ E and the caller decides
fn load(raw: &str, limit: usize) -> Result<Features, LoadError> {
    let f = parse(raw)?;
    if f.len() > limit { return Err(LoadError::TooLarge(f.len())); }
    Ok(f)
}
```

Here `unwrap` is $[\mathrm{id}, \bot]$, an eliminator that is partial on $E$ and reintroduces the partiality the type had removed.

## Next decision

If you want deeper provenance, tell me which axis you're prioritizing. I can then run searches and give verified permalinks for that column only, rather than the whole table.