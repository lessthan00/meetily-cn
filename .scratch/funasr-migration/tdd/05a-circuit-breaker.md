# TDD 05a — 断路器状态机

**父 Issue**: [#05 容灾恢复](../issues/05-crash-recovery.md)

## 测试范围

断路器的状态转换逻辑：从正常 → 计数 → 打开 → 半开 → 关闭。

## RED — 先写失败测试

```rust
// funasr_client.rs tests

#[test]
fn circuit_breaker_starts_closed() {
    let cb = CircuitBreaker::new(3);
    assert_eq!(cb.state(), CircuitState::Closed);
    assert!(cb.allow_request());
}


#[test]
fn single_error_does_not_open() {
    let mut cb = CircuitBreaker::new(3);
    cb.record_error(ErrorKind::Http5xx);
    assert_eq!(cb.state(), CircuitState::Closed);
    assert!(cb.allow_request());
}


#[test]
fn connection_refused_opens_immediately() {
    let mut cb = CircuitBreaker::new(3);
    cb.record_error(ErrorKind::ConnectionRefused);
    assert_eq!(cb.state(), CircuitState::Open);
    assert!(!cb.allow_request());
}


#[test]
fn connection_reset_opens_immediately() {
    let mut cb = CircuitBreaker::new(3);
    cb.record_error(ErrorKind::ConnectionReset);
    assert_eq!(cb.state(), CircuitState::Open);
}


#[test]
fn three_consecutive_5xx_opens() {
    let mut cb = CircuitBreaker::new(3);
    cb.record_error(ErrorKind::Http5xx);
    cb.record_error(ErrorKind::Http5xx);
    assert_eq!(cb.state(), CircuitState::Closed);  // 还没开
    cb.record_error(ErrorKind::Http5xx);
    assert_eq!(cb.state(), CircuitState::Open);
}


#[test]
fn timeout_retry_once_then_open() {
    let mut cb = CircuitBreaker::new(3);  // timeout 重试阈值 = 1
    cb.record_error(ErrorKind::Timeout);
    assert!(cb.should_retry());  // 允许重试 1 次
    cb.record_retry_failed();
    assert_eq!(cb.state(), CircuitState::Open);
}


#[test]
fn successful_request_resets_5xx_counter() {
    let mut cb = CircuitBreaker::new(3);
    cb.record_error(ErrorKind::Http5xx);
    cb.record_error(ErrorKind::Http5xx);
    cb.record_success();  // 中间成功 → 重置计数
    cb.record_error(ErrorKind::Http5xx);
    cb.record_error(ErrorKind::Http5xx);
    assert_eq!(cb.state(), CircuitState::Closed);  // 只累计到 2
}


#[test]
fn open_circuit_blocks_all_requests() {
    let mut cb = CircuitBreaker::new(3);
    cb.record_error(ErrorKind::ConnectionRefused);
    assert!(!cb.allow_request());
    assert!(!cb.allow_request());  // 持续阻断
}


#[test]
fn health_probe_allowed_when_open() {
    let mut cb = CircuitBreaker::new(3);
    cb.record_error(ErrorKind::ConnectionRefused);
    assert!(cb.allow_health_probe());  // 健康探测不被阻断
    assert!(!cb.allow_request());      // 但正常请求仍被阻断
}


#[test]
fn two_consecutive_health_success_closes_circuit() {
    let mut cb = CircuitBreaker::new(3);
    cb.record_error(ErrorKind::ConnectionRefused);
    assert_eq!(cb.state(), CircuitState::Open);

    // 15s 后第一次健康探测成功
    cb.record_health_success();
    assert_eq!(cb.state(), CircuitState::HalfOpen);

    // 又一次成功
    cb.record_health_success();
    assert_eq!(cb.state(), CircuitState::Closed);
    assert!(cb.allow_request());
}


#[test]
fn health_failure_keeps_circuit_open() {
    let mut cb = CircuitBreaker::new(3);
    cb.record_error(ErrorKind::ConnectionRefused);

    cb.record_health_success();   // → HalfOpen
    cb.record_health_failure();   // 探测失败
    assert_eq!(cb.state(), CircuitState::Open);  // 回退
}
```

## GREEN — 最小实现

```rust
#[derive(Debug, PartialEq)]
enum CircuitState { Closed, Open, HalfOpen }

struct CircuitBreaker {
    error_threshold: u32,
    error_count: u32,
    health_count: u32,
    state: CircuitState,
    timeout_retry_remaining: u32,
}

impl CircuitBreaker {
    fn record_error(&mut self, kind: ErrorKind) { /* ... */ }
    fn record_success(&mut self) { /* ... */ }
    fn record_health_success(&mut self) { /* ... */ }
    fn record_health_failure(&mut self) { /* ... */ }
    fn allow_request(&self) -> bool { /* ... */ }
    fn allow_health_probe(&self) -> bool { /* ... */ }
    fn should_retry(&self) -> bool { /* ... */ }
}
```

## REFACTOR

- 考虑用 `enum` 统一 `ErrorKind`（ConnectionRefused / ConnectionReset / Timeout / Http5xx）

## 预计耗时

< 1.5 小时（状态机 + 边界测试最多）
