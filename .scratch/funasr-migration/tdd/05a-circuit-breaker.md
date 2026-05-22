# TDD 05a — 断路器状态机 (差异化策略)

**父 Issue**: [#05 容灾恢复](../issues/05-crash-recovery.md)

## 测试范围

断路器的状态转换逻辑。**6 种错误类型差异化处理** (Q15)：立即打开 (ECONNREFUSED/ECONNRESET/DeserializationFailed) vs 累积打开 (Http5xx/Unhealthy 连续 3 次) vs 重试后打开 (Timeout 重试 1 次)。

## ErrorKind 枚举

```rust
enum ErrorKind {
    ConnectionRefused,    // 立即打开
    ConnectionReset,      // 立即打开
    Timeout,              // 重试 1 次
    Http5xx,              // 连续 3 次
    Unhealthy,            // 连续 3 次
    DeserializationFailed,// 立即打开
}
```

## RED — 先写失败测试

```rust
#[test]
fn circuit_breaker_starts_closed() {
    let cb = CircuitBreaker::new();
    assert_eq!(cb.state(), CircuitState::Closed);
    assert!(cb.allow_request());
}

// === 立即打开类 (ConnectionRefused / ConnectionReset / DeserializationFailed) ===

#[test]
fn connection_refused_opens_immediately() {
    let mut cb = CircuitBreaker::new();
    cb.record_error(ErrorKind::ConnectionRefused);
    assert_eq!(cb.state(), CircuitState::Open);
}

#[test]
fn connection_reset_opens_immediately() {
    let mut cb = CircuitBreaker::new();
    cb.record_error(ErrorKind::ConnectionReset);
    assert_eq!(cb.state(), CircuitState::Open);
}

#[test]
fn deserialization_failed_opens_immediately() {
    let mut cb = CircuitBreaker::new();
    cb.record_error(ErrorKind::DeserializationFailed);
    assert_eq!(cb.state(), CircuitState::Open);
}

// === 累积打开类 (Http5xx / Unhealthy: 连续 3 次) ===

#[test]
fn three_consecutive_5xx_opens() {
    let mut cb = CircuitBreaker::new();
    cb.record_error(ErrorKind::Http5xx);
    cb.record_error(ErrorKind::Http5xx);
    assert_eq!(cb.state(), CircuitState::Closed);
    cb.record_error(ErrorKind::Http5xx);
    assert_eq!(cb.state(), CircuitState::Open);
}

#[test]
fn three_consecutive_unhealthy_opens() {
    let mut cb = CircuitBreaker::new();
    cb.record_error(ErrorKind::Unhealthy);
    cb.record_error(ErrorKind::Unhealthy);
    cb.record_error(ErrorKind::Unhealthy);
    assert_eq!(cb.state(), CircuitState::Open);
}

// === 重试类 (Timeout: 重试 1 次) ===

#[test]
fn timeout_retry_once_then_open() {
    let mut cb = CircuitBreaker::new();
    cb.record_error(ErrorKind::Timeout);
    assert!(!cb.is_open());  // 还没开，等待重试
    cb.record_retry_failed();
    assert_eq!(cb.state(), CircuitState::Open);
}

// === 成功重置 ===

#[test]
fn success_resets_accumulate_counter() {
    let mut cb = CircuitBreaker::new();
    cb.record_error(ErrorKind::Http5xx);
    cb.record_error(ErrorKind::Http5xx);
    cb.record_success();
    cb.record_error(ErrorKind::Http5xx);
    cb.record_error(ErrorKind::Http5xx);
    assert_eq!(cb.state(), CircuitState::Closed);
}

// === 半开 → 恢复 ===

#[test]
fn open_to_halfopen_after_timeout() { /* timeout T 后进入半开 */ }

#[test]
fn halfopen_two_consecutive_health_success_closes() {
    let mut cb = CircuitBreaker::new();
    cb.record_error(ErrorKind::ConnectionRefused); // → Open
    cb.transition_to_half_open();  // timeout 后
    cb.record_health_success();
    assert_eq!(cb.state(), CircuitState::HalfOpen);
    cb.record_health_success();
    assert_eq!(cb.state(), CircuitState::Closed);
}

#[test]
fn halfopen_health_failure_reopens() {
    let mut cb = CircuitBreaker::new();
    cb.record_error(ErrorKind::ConnectionRefused);
    cb.transition_to_half_open();
    cb.record_health_success();
    cb.record_health_failure();
    assert_eq!(cb.state(), CircuitState::Open);
}

// === 阻断行为 ===

#[test]
fn open_circuit_blocks_requests_but_allows_health_probe() {
    let mut cb = CircuitBreaker::new();
    cb.record_error(ErrorKind::ConnectionRefused);
    assert!(!cb.allow_request());
    assert!(cb.allow_health_probe());
}

#[test]
fn halfopen_blocks_requests_but_allows_health_probe() {
    // 半开时正常 chunk 继续丢弃，只允许探测
}
```

## GREEN — 最小实现

```rust
#[derive(Debug, PartialEq)]
enum CircuitState { Closed, Open, HalfOpen }

struct CircuitBreaker {
    state: CircuitState,
    accumulate_count: u32,  // Http5xx/Unhealthy 累计计数
    health_success_count: u32,  // 半开中连续成功计数
    timeout_retry_pending: bool,
}

impl CircuitBreaker {
    fn record_error(&mut self, kind: ErrorKind) {
        match kind {
            ErrorKind::ConnectionRefused
            | ErrorKind::ConnectionReset
            | ErrorKind::DeserializationFailed => self.state = CircuitState::Open,
            ErrorKind::Timeout => self.timeout_retry_pending = true,
            ErrorKind::Http5xx | ErrorKind::Unhealthy => {
                self.accumulate_count += 1;
                if self.accumulate_count >= 3 { self.state = CircuitState::Open; }
            }
        }
    }
    fn record_success(&mut self) { self.accumulate_count = 0; }
    fn record_health_success(&mut self) { /* ... */ }
    fn record_health_failure(&mut self) { /* ... */ }
    fn record_retry_failed(&mut self) { self.state = CircuitState::Open; }
}
```

## REFACTOR

- 用 `enum` 统一 `ErrorKind`

## 预计耗时

< 1.5 小时 (~12 条测试，状态机 + 差异化处理)
