<!-- Author: GENERATED | Date: 2025-11-12 -->
# PoWPreferenceManager – Chức năng & Bảo vệ Sự kiện Nostr

## 1. Mục tiêu
`PoWPreferenceManager` quản lý trạng thái bật/tắt và độ khó (difficulty) Proof of Work (PoW) cho các sự kiện Nostr nhằm:
- Giảm spam (đặc biệt các ghi chú theo geohash dễ bị flood).
- Tăng chi phí tính toán trước khi gửi sự kiện → hạn chế gửi hàng loạt vô nghĩa.
- Cho phép UI phản ứng theo thời gian thực (hiển thị đang "mining").

## 2. Thành phần chính
| Thuộc tính | Vai trò |
|------------|---------|
| `_powEnabled: StateFlow<Boolean>` | Phát current trạng thái bật PoW để UI hoặc logic lấy. |
| `_powDifficulty: StateFlow<Int>` | Phát độ khó PoW (ý nghĩa: số bit/leading zeros mong muốn). |
| `_isMining: StateFlow<Boolean>` | Cho UI hiển thị tiến trình (spinner/animation) khi đang tìm nonce. |
| `SharedPreferences` (`pow_preferences`) | Lưu cấu hình bền vững qua lần mở app. |
| `PoWSettings` (data class) | Gói gọn cấu hình hiện tại (enabled + difficulty). |

## 3. API hiện có
| Hàm | Mô tả |
|-----|-------|
| `init(context)` | Khởi tạo, load giá trị đã lưu; gọi 1 lần lúc startup. |
| `isPowEnabled()` | Trả về trạng thái bật/tắt PoW. |
| `setPowEnabled(enabled)` | Bật/tắt và lưu xuống prefs. |
| `getPowDifficulty()` | Lấy độ khó hiện tại. |
| `setPowDifficulty(difficulty)` | Đặt độ khó (clamp 0..32) và lưu. |
| `getCurrentSettings()` | Trả về `PoWSettings`. |
| `resetToDefaults()` | Quay về mặc định (enabled=false, difficulty=12). |
| `getDifficultyLevels()` | Danh sách gợi ý độ khó + mô tả thời gian ước lượng. |
| `isMining()` | Trả về trạng thái mining hiện tại. |
| `startMining()` / `stopMining()` | Cập nhật state mining cho UI. |

## 4. Ý nghĩa độ khó đề xuất (Heuristic)
Bảng mô tả trong code:
```
0  -> Tắt PoW.
8  -> Rất thấp.
12 -> Thấp (~0.1s).
16 -> Trung bình (~2s).
20 -> Cao (~30s).
24 -> Rất cao (~8 phút).
28 -> Cực cao (~2 giờ).
32 -> Tối đa (~8 giờ).
```
(Lưu ý: Thời gian thực phụ thuộc thiết bị & chiến lược thuật toán mining.)

## 5. Cơ chế Bảo vệ Sự kiện (Khái niệm)
Để "bảo vệ" sự kiện Nostr bằng PoW, thường dùng chuẩn NIP-13:
- Tính `sha256(serialized_event)` phải có `difficulty` bit đầu bằng 0 (leading zero bits).
Hoặc:
- Dùng tag `nonce` dạng: `['nonce', value, targetDifficulty]` rồi lặp thay đổi `value` đến khi hash đạt mục tiêu.

### Lý do hiệu quả
- Mỗi sự kiện cần thời gian/tài nguyên CPU để sinh ra hợp lệ.
- Kẻ tấn công muốn spam hàng nghìn sự kiện sẽ bị chậm lại đáng kể.
- Độ khó cao → tốn năng lượng, nên UX cho phép chỉnh mức hợp lý (mặc định 12).

## 6. Luồng Mining Gợi ý (Chưa có trong class – có thể bổ sung)
Pseudo-code:
```kotlin
suspend fun mineEvent(baseEvent: NostrEvent, difficulty: Int): NostrEvent {
    if (difficulty <= 0) return baseEvent
    PoWPreferenceManager.startMining()
    try {
        var nonce = 0L
        val targetPrefixBits = difficulty
        while (true) {
            val mutated = baseEvent.withNonceTag(nonce, difficulty) // thêm/ghi đè tag 'nonce'
            val hash = sha256(mutated.serialize())
            if (leadingZeroBits(hash) >= targetPrefixBits) {
                return mutated
            }
            nonce++
            if (nonce % 10_000 == 0L) yield() // nhường coroutine tránh block UI
        }
    } finally {
        PoWPreferenceManager.stopMining()
    }
}
```

### Hàm kiểm tra:
```kotlin
fun verifyEventPoW(event: NostrEvent): Boolean {
    val nonceTag = event.tags.find { it.size >= 3 && it[0] == "nonce" } ?: return false
    val declaredDifficulty = nonceTag[2].toIntOrNull() ?: return false
    if (declaredDifficulty <= 0) return false
    val hash = sha256(event.serialize())
    return leadingZeroBits(hash) >= declaredDifficulty
}
```

## 7. Đề xuất mở rộng vào class (nhẹ, an toàn)
| Mở rộng | Lợi ích |
|--------|---------|
| `fun currentDifficultyOrZero()` | Trả về 0 nếu disabled để code send dễ đọc. |
| `fun shouldApplyPoW()` | Tách logic enable + difficulty > 0. |
| Thêm `StateFlow<PoWSettings>` | Cho UI theo dõi đồng bộ cả hai giá trị. |
| Thêm `lastMineDurationMs` | Cho phép UI hiển thị thời gian thực tế mining → gợi ý người dùng điều chỉnh difficulty. |

## 8. Kiến trúc & Thread-safety
| Kỹ thuật | Vai trò |
|----------|---------|
| `StateFlow` | Reactive, tránh race điều khiển UI. |
| `coerceIn(0, 32)` | Giữ difficulty trong ngưỡng an toàn để không treo thiết bị. |
| `startMining/stopMining` | Cung cấp tín hiệu UI thống nhất, tránh nhiều nguồn cập nhật rời rạc. |
| `apply()` trong SharedPreferences | Ghi async không block UI. |

## 9. Edge Cases & Bảo vệ
| Trường hợp | Handling hiện tại |
|-----------|------------------|
| Gọi setters trước `init()` | Không crash: `_powEnabled` & `_powDifficulty` cập nhật StateFlow; `sharedPrefs` chưa init → không lưu. (Có thể thêm cảnh báo log). |
| Difficulty âm / >32 | Bị kẹp trong `setPowDifficulty`. |
| Mining bật nhưng thiết bị yếu | User có thể giảm độ khó với UI mapping có giải thích. |
| Spam khi `enabled=false` | Lý do chọn default=12 chỉ hiệu lực nếu user bật; nếu cần cưỡng chế, có thể đổi DEFAULT_POW_ENABLED=true. |

## 10. Chiến lược Bảo vệ Sâu hơn (Defense in Depth)
1. Kết hợp PoW với rate limit per pubkey/geohash: ví dụ tối đa X sự kiện / phút.
2. Áp dụng adaptive difficulty: nếu vùng geohash nhiều spam → tự động tăng `difficulty` tạm thời.
3. Kiểm tra PoW inbound: bỏ qua sự kiện không đạt ngưỡng khi chế độ bảo vệ địa lý đang bật.
4. Song song hóa mining (nếu dùng thiết bị mạnh) bằng chia nonce range cho nhiều coroutine → giảm latency gửi.
5. Cho phép hủy mining (cancel coroutine) khi user thay đổi nội dung trước khi xong.

## 11. Đo lường & Quan sát
Đề xuất thêm metrics:
- Số vòng lặp nonce trung bình.
- Tỷ lệ sự kiện bị bỏ qua vì không đạt PoW.
- Thời gian mining trung bình theo độ khó.

## 12. UX Gợi ý
| Tình huống | Gợi ý UI |
|-----------|----------|
| Mining >5s | Hiển thị tip: "Giảm độ khó để gửi nhanh hơn." |
| Difficulty >=24 | Hiển thị cảnh báo nóng máy / pin. |
| Disabled (0) | Hiển thị badge "FAST" chống hiểu lầm. |

## 13. Tóm tắt nhanh
`PoWPreferenceManager` hiện chỉ giữ và phát cấu hình PoW + trạng thái mining. Để thực sự "bảo vệ" sự kiện bạn cần tích hợp thêm:
- Hàm mining tạo nonce dựa trên độ khó.
- Hàm xác thực inbound theo NIP-13.
- Cơ chế hủy & đo thời gian.

## 14. Next Steps (Roadmap nhẹ)
1. Thêm module `PoWMiner` riêng: chịu trách nhiệm sinh nonce + verify → giữ class này tinh gọn.
2. Tạo unit test: clamp difficulty, reset defaults, mining state transitions.
3. Thêm logging khi gọi setter trước init (dev cảnh báo). 
4. Bật PoW mặc định cho vùng geohash đông người (config động).

---
Nếu bạn muốn tôi thêm code mẫu `PoWMiner` hoặc hàm verify trực tiếp vào project, hãy yêu cầu tiếp.

