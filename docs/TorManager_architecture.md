<!-- Author: YOUR_GITHUB_EMAIL@example.com | Date: 2025-11-11 -->
# TorManager Architecture & Anonymity Flow

## 1. Mục tiêu
`TorManager` quản lý vòng đời Tor nhúng (Arti) và đảm bảo tất cả lưu lượng mạng của ứng dụng (HTTP, relay Nostr) được ép qua SOCKS proxy ẩn danh, chống rò rỉ kết nối trực tiếp.

## 2. Tổng quan thành phần
- Singleton: `object TorManager`
- Thư viện nền: `info.guardianproject.arti.ArtiProxy` (Tor implementation bằng Rust).
- Tầng cấu hình: `TorPreferenceManager` lưu và phát chế độ `TorMode` (`ON`/`OFF`).
- Tầng mạng: `OkHttpProvider` & `NostrRelayManager` được reset để rebuild kết nối qua proxy.
- Cơ chế trạng thái: `MutableStateFlow<TorStatus>` phát hiện tiến độ bootstrap và trạng thái hoạt động.

## 3. Sơ đồ trạng thái
```
TorMode (ON/OFF) --> applyMode() --> LifecycleState (STARTING/RUNNING/STOPPING/STOPPED)
                              |--> TorState (OFF, STARTING, BOOTSTRAPPING, RUNNING, STOPPING, ERROR)
```
- Trạng thái được cập nhật bằng parsing log từ Arti (`handleArtiLogLine`).

## 4. Cấu trúc dữ liệu chính
```kotlin
// Fraction: TorState & TorStatus
enum class TorState { OFF, STARTING, BOOTSTRAPPING, RUNNING, STOPPING, ERROR }

data class TorStatus(
    val mode: TorMode = TorMode.OFF,
    val running: Boolean = false,
    val bootstrapPercent: Int = 0, // 0 / 75 / 100 mapping
    val lastLogLine: String = "",
    val state: TorState = TorState.OFF
)
```
Ý nghĩa:
- `bootstrapPercent`: ánh xạ mốc log ("Sufficiently bootstrapped" -> 75, guard usable -> 100).
- `running`: proxy tiến trình đã khởi lên hay chưa.

## 5. Khởi tạo (`init`)
```kotlin
fun init(application: Application) {
    if (initialized) return
    // ...existing code...
    val savedMode = TorPreferenceManager.get(application)
    if (savedMode == TorMode.ON) {
        socksAddr = InetSocketAddress("127.0.0.1", currentSocksPort)
        OkHttpProvider.reset()
    }
    appScope.launch { applyMode(application, savedMode) }
    appScope.launch { TorPreferenceManager.modeFlow.collect { mode -> applyMode(application, mode) } }
}
```
Điểm chính:
- Double-check tránh khởi tạo lại.
- Nếu mode lưu là ON: thiết lập SOCKS ngay để khóa lưu lượng.
- Bắt đầu lắng nghe thay đổi mode động.

## 6. Áp dụng chế độ (`applyMode`)
```kotlin
suspend fun applyMode(application: Application, mode: TorMode) {
    applyMutex.withLock {
        desiredMode = mode
        when (mode) {
            TorMode.OFF -> { /* stopArti, remove socks, reset network */ }
            TorMode.ON  -> {
                socksAddr = InetSocketAddress("127.0.0.1", currentSocksPort)
                startArti(application, useDelay = false)
                appScope.launch { waitUntilBootstrapped(); /* finalize proxy */ }
            }
        }
    }
}
```
Cơ chế an toàn:
- Khóa `applyMutex` tránh race ON/OFF.
- SOCKS được set TRƯỚC khi Tor sẵn sàng (prevent leak).

## 7. Khởi động Arti (`startArti`)
```kotlin
private suspend fun startArti(application: Application, useDelay: Boolean = false) {
    stopArtiAndWait() // đảm bảo sạch phiên cũ
    val logListener = ArtiLogListener { s -> handleArtiLogLine(s.toString()) }
    val proxy = ArtiProxy.Builder(application)
        .setSocksPort(currentSocksPort)
        .setDnsPort(currentSocksPort + 1)
        .setLogListener(logListener)
        .build()
    artiProxyRef.set(proxy)
    proxy.start()
    _status.value = _status.value.copy(running = true, state = TorState.STARTING)
    startInactivityMonitoring()
}
```
Bảo vệ ẩn danh:
- DNS port riêng: tránh rò rỉ truy vấn DNS.
- Log listener: nguồn duy nhất cập nhật bootstrap.
- Trước khi start mới, force stop cũ -> tránh file lock.

## 8. Xử lý lỗi & retry
Bind error:
```kotlin
if (isBindError(e) && bindRetryAttempts < MAX_RETRY_ATTEMPTS) {
    currentSocksPort++
    socksAddr = InetSocketAddress("127.0.0.1", currentSocksPort)
    startArti(application, useDelay = false)
}
```
Non-bind error:
```kotlin
scheduleRetry(application) // exponential backoff <= 30s
```
Ý nghĩa:
- Lỗi chiếm cổng: tăng port ngay, cập nhật SOCKS địa chỉ mới để tất cả future connection trỏ đúng.
- Lỗi khác: tránh spam bằng backoff.

## 9. Dừng & Restart
```kotlin
private suspend fun stopArtiAndWait(timeoutMs: Long = STOP_TIMEOUT_MS) {
    stopArtiInternal()
    waitForStateTransition(target = TorState.OFF, timeoutMs)
    delay(200) // grace period
}
private suspend fun restartArti(application: Application) {
    stopArtiAndWait(); delay(RESTART_DELAY_MS); startArti(application)
}
```
Giữ SOCKS: restart không xóa `socksAddr` (ngoại trừ OFF).

## 10. Giám sát inactivity bootstrap
```kotlin
private fun startInactivityMonitoring() {
    inactivityJob = appScope.launch {
        while (true) {
            delay(INACTIVITY_TIMEOUT_MS)
            if (timeSinceLastActivity > INACTIVITY_TIMEOUT_MS && _status.value.bootstrapPercent < 100) {
                restartArti(app); break
            }
        }
    }
}
```
Nếu bootstrap treo -> restart để tiếp tục tiến trình.

## 11. Phân tích log (`handleArtiLogLine`)
```kotlin
private fun handleArtiLogLine(s: String) {
    when {
        s.contains("Sufficiently bootstrapped", true) -> _status.value = _status.value.copy(bootstrapPercent = 75, state = TorState.BOOTSTRAPPING)
        s.contains("guard [scrubbed] is usable", true) -> _status.value = _status.value.copy(state = TorState.RUNNING, bootstrapPercent = 100, running = true)
        s.contains("Stopping", true) -> _status.value = _status.value.copy(state = TorState.STOPPING)
        s.contains("Stopped", true) -> _status.value = _status.value.copy(state = TorState.OFF, running = false, bootstrapPercent = 0)
    }
}
```
Mapping tối giản từ log → trạng thái.

## 12. Cơ chế chờ đồng bộ
```kotlin
private suspend fun waitUntilBootstrapped() {
    while (true) {
        val s = statusFlow.first { (it.bootstrapPercent >= 100 && it.state == TorState.RUNNING) || !it.running || it.state == TorState.ERROR }
        if (!s.running || s.state == TorState.ERROR) return
        if (s.bootstrapPercent >= 100 && s.state == TorState.RUNNING) return
    }
}
```
Đảm bảo chỉ tiếp tục logic nâng cấp proxy/clients khi Tor thực sự usable.

## 13. Kiểm tra sẵn sàng proxy
```kotlin
fun isProxyEnabled(): Boolean {
    val s = _status.value
    return s.mode != TorMode.OFF && s.running && s.bootstrapPercent >= 100 && socksAddr != null && s.state == TorState.RUNNING
}
```
Điều kiện chặt: mode ON + RUNNING + bootstrap đủ + địa chỉ tồn tại.

## 14. Biện pháp chống rò rỉ
| Biện pháp | Mục đích |
|-----------|----------|
| Đặt SOCKS trước bootstrap | Ngăn kết nối trực tiếp khởi tạo trong giai đoạn chuyển chế độ |
| Reset OkHttp và Relay | Xóa pool / socket cũ không proxy |
| DNS qua Tor (`dnsPort`) | Loại bỏ DNS leak |
| Restart khi inactivity | Tránh treo chưa hoàn tất bootstrap |
| Tăng port khi bind fail | Tránh kẹt ở cổng bị chiếm |

## 15. Đa luồng & Đồng bộ hóa
- `@Volatile` đảm bảo visibility các biến đơn giản.
- `Mutex applyMutex` serial hóa thay đổi mode.
- `AtomicReference` cho `ArtiProxy` tránh race start/stop.
- `StateFlow` phát cho UI/subscriber an toàn.
- Deferred (`CompletableDeferred`) trong `waitForStateTransition` để chờ sự kiện log.

## 16. Rủi ro & Cải tiến đề xuất
Rủi ro:
- Không giám sát health sau RUNNING (mất log dài hạn).
- Tăng port vô hạn nếu liên tục bind fail.
- `bootstrapPercent` giả lập gây hiểu sai tiến độ.

Cải tiến:
- Thêm upper bound port (ví dụ 65500) + rollback.
- Health check định kỳ (ping circuit) sau RUNNING.
- Parsing chi tiết hơn để cung cấp % thực tế.
- Circuit isolation theo domain (nâng cao quyền riêng tư).

## 17. Tóm tắt nhanh chức năng chính
| Hàm | Vai trò |
|-----|---------|
| `init` | Khởi tạo manager, áp dụng mode lưu |
| `applyMode` | Chuyển ON/OFF, đảm bảo an toàn & không rò rỉ |
| `startArti` | Khởi động ArtiProxy + listener + monitoring |
| `stopArtiAndWait` | Dừng sạch và chờ xác nhận OFF |
| `restartArti` | Chu trình restart giữ proxy |
| `scheduleRetry` | Exponential backoff cho lỗi không bind |
| `startInactivityMonitoring` | Phát hiện treo bootstrap |
| `handleArtiLogLine` | Mapping log → trạng thái nội bộ |
| `waitUntilBootstrapped` | Chặn đến khi Tor usable |
| `isProxyEnabled` | Kiểm tra điều kiện sử dụng an toàn |

## 18. Kết luận
`TorManager` xây dựng một lớp bao bọc vòng đời Tor nhất quán, tập trung vào:
- Ngăn rò rỉ lưu lượng không ẩn danh.
- Tự hồi phục trước lỗi.
- Cung cấp tín hiệu sẵn sàng rõ ràng cho các lớp mạng khác.

---
Nếu cần mở rộng: thêm biểu đồ sequence, unit test scenario (ON->FAIL->RETRY->RUNNING), hoặc tài liệu tích hợp với `OkHttpProvider`.

