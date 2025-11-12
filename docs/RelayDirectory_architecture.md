<!-- Author: GENERATED | Date: 2025-11-12 -->
# RelayDirectory Architecture (Kiến trúc)

## 1. Mục tiêu
`RelayDirectory` cung cấp danh sách các Nostr relay kèm tọa độ địa lý và hàm truy vấn nhanh các relay gần nhất với một geohash người dùng. Nó:
- Nạp dữ liệu ban đầu từ asset tĩnh (`assets/nostr_relays.csv`).
- Tự động tải về bản cập nhật mới nhất từ GitHub (URL nguồn). 
- Xác định độ cũ (stale) của dữ liệu (>24h) và làm mới định kỳ mỗi phút nếu cần.
- Bảo đảm thread-safety khi đọc/ghi danh sách relay.

## 2. Tổng quan thành phần
| Thành phần | Vai trò |
|------------|--------|
| `object RelayDirectory` | Singleton cung cấp API công khai. |
| `RelayInfo` | Data class chứa `url`, `latitude`, `longitude`. |
| `relays` + `relaysLock` | Bộ nhớ tạm danh sách relay và khóa đồng bộ hóa. |
| `SharedPreferences` | Lưu thời điểm cập nhật cuối (`last_update_ms`). |
| `OkHttpClient` | Tải CSV mới qua HTTP. |
| Coroutine (`ioScope`) | Chạy các tác vụ nền (download, refresh). |

## 3. Dữ liệu & Trạng thái
- `initialized`: cờ đảm bảo chỉ khởi tạo một lần.
- `relays`: danh sách hiện tại đã parse.
- `KEY_LAST_UPDATE_MS`: timestamp lần cập nhật thành công gần nhất.
- Thời gian hết hạn: `ONE_DAY_MS = 24h`.

## 4. Chu trình khởi tạo (`initialize`)
Pseudo-flow:
```
if already initialized -> return
try tải file đã download (nếu tồn tại & hợp lệ)
  nếu tải không được: fallback asset
đánh dấu initialized = true
coroutine: nếu dữ liệu đã quá 24h -> fetch mới
bắt đầu vòng lặp refresh mỗi phút -> kiểm tra stale -> fetch
```
Đảm bảo ứng dụng luôn có ít nhất một danh sách relay (từ asset nếu mạng lỗi).

## 5. Luồng làm mới (Refresh Flow)
```
startPeriodicRefresh() -> while (true):
    if isStale(): fetchAndMaybeSwap()
    delay(60 giây)
```
`isStale()` so sánh `now - last_update_ms >= 24h`.

## 6. Tải & Hoán đổi danh sách (`fetchAndMaybeSwap`)
Các bước:
1. Tạo file tạm trong `cacheDir`.
2. `downloadToFile(URL, tmp)` dùng OkHttp.
3. Parse CSV -> `parsed` list.
4. Nếu rỗng: bỏ qua.
5. Sao chép từ file tạm sang file đích (`filesDir/nostr_relays_latest.csv`).
6. Tính SHA-256 để log.
7. Ghi đè `relays` bên trong `synchronized(relaysLock) { ... }`.
8. Cập nhật `last_update_ms` trong SharedPreferences.

Đặc điểm an toàn:
- Chỉ thay thế danh sách nếu parse thành công và có >=1 entry.
- Sử dụng file tạm để tránh trạng thái nửa chừng.

## 7. Phân tích CSV (`parseCsv`)
Logic:
- Bỏ qua dòng trống hoặc header bắt đầu bằng "relay url".
- Tách bởi dấu phẩy, cần >=3 phần tử.
- Chuẩn hóa URL (nếu không chứa `://` -> thêm tiền tố `wss://`).
- Parse lat/lon `toDoubleOrNull()` (loại bỏ dòng lỗi). 
- Tạo `RelayInfo` và đưa vào `result`.

Edge cases xử lý:
- Dòng hỏng định dạng -> bỏ qua.
- URL trống -> bỏ qua.
- Lat/Lon không hợp lệ -> bỏ qua.

## 8. Tìm relay gần nhất (`closestRelaysForGeohash`)
Các bước:
1. Snapshot thread-safe: `synchronized(relaysLock) { relays.toList() }`.
2. Decode trung tâm geohash: `Geohash.decodeToCenter(geohash)` -> (lat, lon). Nếu lỗi: trả `emptyList()`.
3. Sắp xếp snapshot theo khoảng cách Haversine tới điểm trung tâm.
4. `take(nRelays.coerceAtLeast(0))` lấy tối đa số yêu cầu.
5. Trả về danh sách URL.

Đặc điểm hiệu năng:
- Dùng `asSequence()` để tránh tạo nhiều collection tạm thời.
- Phù hợp khi N không lớn (danh sách relay có kích thước vừa phải). Nếu lớn hơn có thể tối ưu bằng partial selection (e.g. QuickSelect) nhưng chưa cần.

## 9. Công thức khoảng cách (Haversine)
```
R = 6371000 (m)
a = sin(dLat/2)^2 + cos(lat1) * cos(lat2) * sin(dLon/2)^2
c = 2 * atan2( sqrt(a), sqrt(1-a) )
distance = R * c
```
Sử dụng mét cho phép so sánh trực tiếp khi sort.

## 10. Hash SHA-256 (`fileSha256Hex` / `streamSha256Hex`)
- Đọc stream với buffer 8192 byte.
- Cập nhật `MessageDigest`.
- Chuyển bytes sang hex (zero-pad từng byte). 
- Dùng cho mục đích kiểm chứng integrity khi log.

## 11. Đồng bộ & Thread-safety
| Cơ chế | Vai trò |
|--------|---------|
| `@Volatile initialized` | Tránh đọc trạng thái stale giữa các thread. |
| `synchronized(this)` trong `initialize` | Ngăn double init song song. |
| `relaysLock` | Bảo vệ `relays` khi đọc/ghi. |
| Snapshot `relays.toList()` | Đảm bảo không iterate trên danh sách bị mutate. |

Không sử dụng coroutines write concurrently trực tiếp vào `relays` ngoài vùng synchronized.

## 12. Logging & Observability
- Mỗi nguồn: asset (`📦`), downloaded (`✅`), file (`📄`).
- Log hash + số entries giúp kiểm tra nhanh thay đổi.
- Warning khi parse rỗng / HTTP không thành công.

## 13. Edge Cases Đã Xử Lý
| Tình huống | Ứng xử |
|------------|--------|
| Không có file tải về | Fallback asset. |
| CSV tải về rỗng | Bỏ qua, giữ danh sách cũ. |
| Exception khi download | Log warning, giữ dữ liệu hiện tại. |
| Geohash không hợp lệ | Trả `emptyList()`. |
| nRelays < 0 | Dùng `coerceAtLeast(0)` -> lấy 0. |
| Lặp tải về thất bại nhiều lần | Giữ nguyên dữ liệu cũ (không corrupt). |

## 14. Giới hạn & Cải tiến tiềm năng
| Chủ đề | Hiện tại | Đề xuất |
|--------|----------|---------|
| Hiệu năng sort | Full sort mỗi truy vấn | Dùng cấu trúc spatial index (KD-tree) nếu relay > vài nghìn. |
| Tính mới dữ liệu | Chu kỳ 24h | Cho phép cấu hình qua remote hoặc adaptive theo thay đổi. |
| Kiểm tra integrity | Chỉ SHA-256 & != empty | So sánh hash cũ → bỏ qua nếu không đổi (tiết kiệm ghi). |
| Đồng bộ thời gian | Dựa vào clock thiết bị | Thêm sanity check nếu clock lệch lớn. |
| Cache geohash phổ biến | Không | Thêm LRU cache cho truy vấn lặp lại geohash nhỏ. |

## 15. Bảo mật & Tin cậy
- Không chạy code thực thi từ file fetch (chỉ parse CSV thuần). 
- Bỏ qua dòng lỗi tránh crash.
- Hash giúp điều tra nếu có giả mạo nội dung.

## 16. Tóm tắt hàm chính
| Hàm | Mô tả ngắn |
|-----|------------|
| `initialize(app)` | Khởi tạo, load dữ liệu ưu tiên file tải về, fallback asset, lên lịch refresh. |
| `closestRelaysForGeohash(gh, n)` | Trả danh sách URL relay gần tâm geohash. |
| `haversineMeters(...)` | Tính khoảng cách địa lý giữa hai điểm. |
| `normalizeRelayUrl(raw)` | Chuẩn hóa thành URL đầy đủ (thêm `wss://` nếu thiếu). |
| `isStale(app)` | Kiểm tra dữ liệu quá 24h chưa. |
| `startPeriodicRefresh(app)` | Vòng lặp mỗi phút kiểm tra stale. |
| `fetchAndMaybeSwap(app)` | Tải, parse, kiểm tra rỗng, swap an toàn vào bộ nhớ & prefs. |
| `downloadToFile(url, file)` | Tải HTTP lưu thẳng ra file đích. |
| `loadFromFile(file, label)` | Parse và nạp danh sách từ file cục bộ (downloaded). |
| `loadFromAssets(app)` | Parse asset CSV ban đầu. |
| `parseCsv(input)` | Chuyển nội dung CSV -> `List<RelayInfo>`. |
| `fileSha256Hex(file)` | Hash nhanh file. |
| `streamSha256Hex(input)` | Hash stream chung.

## 17. Trình tự ví dụ hoạt động
Scenario: App khởi động lần đầu.
```
initialize() -> asset CSV parse -> relays có dữ liệu -> check stale (lần đầu: last=0 -> stale) -> fetchAndMaybeSwap() cố tải GitHub
  nếu tải thành công -> thay thế danh sách + ghi timestamp
  nếu thất bại -> vẫn dùng asset
Periodic loop: mỗi 60s -> isStale()? (sẽ false trong ~24h) -> không tải
```

## 18. Kết luận
`RelayDirectory` là module nhỏ gọn, ổn định, hướng tới độ tin cậy dữ liệu relay với chiến lược:
- Fallback an toàn.
- Refresh nền khi cần thiết.
- API truy vấn gần nhất đơn giản, dễ mở rộng.

Có thể mở rộng bằng tối ưu spatial, cache truy vấn, và kiểm tra hash delta để giảm IO.

---
Nếu cần thêm: biểu đồ sequence (Init → Fallback → Refresh), hoặc unit test cho `parseCsv` và `closestRelaysForGeohash` edge cases.

