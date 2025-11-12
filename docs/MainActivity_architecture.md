<!-- Author: GENERATED | Date: 2025-11-12 -->
# MainActivity Architecture & Khởi tạo (onCreate)

## 1. Mục tiêu của `MainActivity`
`MainActivity` là entry point màn hình chính của ứng dụng, chịu trách nhiệm:
- Điều phối quy trình Onboarding (Bluetooth, Location, Battery Optimization, Permissions).
- Khởi tạo dịch vụ mesh Bluetooth (`BluetoothMeshService`).
- Thiết lập UI Compose và ViewModels (`MainViewModel`, `ChatViewModel`).
- Quản lý trạng thái nền/foreground của ứng dụng & điều hướng từ notification.

## 2. Thứ tự khởi động (Startup Sequence chi tiết)
```
A. Người dùng mở app -> Activity được tạo
B. onCreate()
   1. super.onCreate()
   2. enableEdgeToEdge()
   3. Tạo PermissionManager
   4. Tạo BluetoothMeshService (meshService)
   5. Tạo BluetoothStatusManager (đăng ký callback handleBluetoothEnabled/Disabled)
   6. Tạo LocationStatusManager (callback handleLocationEnabled/Disabled)
   7. Tạo BatteryOptimizationManager (callback handleBatteryOptimizationDisabled/Failed)
   8. Tạo OnboardingCoordinator (callback handleOnboardingComplete/Failed)
   9. setContent { Compose Scaffold + OnboardingFlowScreen }
  10. Khởi chạy coroutine repeatOnLifecycle(STARTED) thu thập onboardingState -> handleOnboardingStateChange
  11. Nếu state ban đầu == CHECKING -> gọi checkOnboardingStatus()

C. checkOnboardingStatus()
   12. delay(500ms) hiển thị trạng thái CHECKING ngắn
   13. Gọi checkBluetoothAndProceed()

D. checkBluetoothAndProceed()
   14. Nếu first-time launch -> bỏ qua Bluetooth -> proceedWithPermissionCheck()
   15. Ngược lại: cập nhật bluetoothStatus -> phân nhánh:
       - ENABLED -> checkLocationAndProceed()
       - DISABLED / NOT_SUPPORTED -> chuyển state BLUETOOTH_CHECK

E. checkLocationAndProceed()
   16. Nếu first-time -> proceedWithPermissionCheck()
   17. Ngược lại: cập nhật locationStatus -> phân nhánh:
       - ENABLED -> checkBatteryOptimizationAndProceed()
       - DISABLED / NOT_AVAILABLE -> LOCATION_CHECK

F. checkBatteryOptimizationAndProceed()
   18. Nếu first-time -> proceedWithPermissionCheck()
   19. Nếu user đã skip trước đó -> proceedWithPermissionCheck()
   20. Cập nhật batteryOptimizationStatus -> phân nhánh:
       - DISABLED / NOT_SUPPORTED -> proceedWithPermissionCheck()
       - ENABLED -> BATTERY_OPTIMIZATION_CHECK

G. proceedWithPermissionCheck()
   21. delay(200ms) chuyển mượt
   22. Nếu first-time -> PERMISSION_EXPLANATION
   23. Nếu đã đủ quyền -> INITIALIZING -> initializeApp()
   24. Nếu thiếu quyền -> PERMISSION_EXPLANATION

H. Người dùng cấp quyền (OnboardingCoordinator) -> handleOnboardingComplete()
   25. Re-check Bluetooth/Location/BatteryOptimization
   26. Nếu bất kỳ chưa đạt -> chuyển sang màn hình tương ứng (BLUETOOTH_CHECK / LOCATION_CHECK / BATTERY_OPTIMIZATION_CHECK)
   27. Nếu tất cả đạt -> INITIALIZING -> initializeApp()

I. initializeApp()
   28. delay(1000ms) chờ hệ thống ổn định
   29. PoWPreferenceManager.init()
   30. LocationNotesInitializer.initialize()
   31. Kiểm tra lại permissions (revocation guard)
   32. meshService.delegate = chatViewModel
   33. meshService.startServices()
   34. handleNotificationIntent(intent)
   35. delay(500ms)
   36. update state -> COMPLETE

J. Sau COMPLETE:
   - onResume(): xác nhận Bluetooth/Location vẫn bật, nếu tắt -> quay lại màn hình kiểm tra.
   - onPause(): đặt background state cho mesh/chat.
   - onNewIntent(): xử lý notification (private/geohash channel).
   - onDestroy(): cleanup managers, stop meshService nếu COMPLETE.
```

## 3. ViewModels được sử dụng
| ViewModel | Khởi tạo | Vai trò |
|-----------|----------|---------|
| `MainViewModel` | `by viewModels()` | Giữ các StateFlow cho onboarding: bluetoothStatus, locationStatus, batteryOptimizationStatus, errorMessage, loading flags. |
| `ChatViewModel` | Factory tùy chỉnh (truyền `application`, `meshService`) | Quản lý logic chat, private chat, location channels, xử lý back press & notification actions. |

Factory tạo `ChatViewModel` bảo đảm `meshService` đã tồn tại trước khi ViewModel cần đến nó.

## 4. Các Manager được khởi tạo
| Manager | Mục đích |
|---------|----------|
| `PermissionManager` | Kiểm tra & yêu cầu runtime permissions, phát hiện lần đầu chạy. |
| `BluetoothStatusManager` | Kiểm tra, yêu cầu bật Bluetooth, theo dõi trạng thái qua BroadcastReceiver. |
| `LocationStatusManager` | Kiểm tra & yêu cầu bật dịch vụ vị trí (GPS). |
| `BatteryOptimizationManager` | Kiểm tra việc tắt tối ưu pin để dịch vụ nền hoạt động ổn định. |
| `OnboardingCoordinator` | Đóng gói logic yêu cầu quyền, callback khi hoàn tất/thất bại. |
| `BluetoothMeshService` | Dịch vụ kết nối mesh cục bộ (Bluetooth) hỗ trợ chat offline / gần. |

## 5. UI Compose khởi tạo
`setContent { BitchatTheme { Scaffold { OnboardingFlowScreen(...) } } }`
- `OnboardingFlowScreen` đọc state từ `mainViewModel` bằng `collectAsState()`.
- Vẽ màn hình tương ứng theo `onboardingState`.
- Khi ở các state chứa Chat (`CHECKING`, `INITIALIZING`, `COMPLETE`) gắn back callback tuỳ chỉnh.

## 6. Bảng chức năng hàm (Function Catalog)
| Hàm | Trigger chính | Vai trò | Side-effects | Điều kiện dừng / lỗi |
|-----|---------------|---------|--------------|----------------------|
| `onCreate` | Activity tạo | Khởi tạo toàn bộ manager & UI | Tạo service, đăng ký coroutine thu thập state | Nếu state ≠ CHECKING: không gọi `checkOnboardingStatus` |
| `OnboardingFlowScreen` | Compose recomposition | Chọn màn hình phù hợp theo state | Có thể đăng ký BroadcastReceiver BT, thêm back callback | Receiver cleanup nếu dispose |
| `handleOnboardingStateChange` | Mỗi lần state thay đổi | Ghi log khi COMPLETE/ERROR | Không thay đổi state | Không có lỗi logic bên trong |
| `checkOnboardingStatus` | Lần đầu vào state CHECKING | Bắt đầu pipeline kiểm tra | Gọi `checkBluetoothAndProceed` sau delay | - |
| `checkBluetoothAndProceed` | Sau bước CHECKING hoặc callback bật BT | Phân nhánh flow Bluetooth | Cập nhật state + loading flags | Nếu BT NOT_SUPPORTED: có thể dẫn tới ERROR sau |
| `handleBluetoothEnabled` | Callback từ manager | Chuyển nhanh sang Location | Tắt loading, cập nhật status | - |
| `handleBluetoothDisabled` | Callback khi fail | Xử lý permission vs unsupported | Cập nhật errorMessage nếu NOT_SUPPORTED | Có thể chuyển sang ERROR |
| `checkLocationAndProceed` | Sau Bluetooth ENABLED | Phân nhánh flow Location | Cập nhật state + loading flags | Nếu NOT_AVAILABLE -> LOCATION_CHECK (sau đó ERROR) |
| `handleLocationEnabled` | Callback từ manager | Chuyển sang BatteryOptimization | Tắt loading | - |
| `handleLocationDisabled` | Callback fail | Nếu NOT_AVAILABLE -> ERROR | Cập nhật state/error | - |
| `checkBatteryOptimizationAndProceed` | Sau Location ENABLED | Quyết định tối ưu pin | Cập nhật state | Skip nếu first-time hoặc user skipped |
| `handleBatteryOptimizationDisabled` | User tắt tối ưu pin | Tiếp tục tới permission check | Tắt loading | - |
| `handleBatteryOptimizationFailed` | User không tắt được | Giữ ở màn hình tối ưu pin | Tắt loading | - |
| `proceedWithPermissionCheck` | Sau các kiểm tra môi trường | Nhảy vào xin quyền hoặc init | Cập nhật state | - |
| `handleOnboardingComplete` | Sau khi cấp quyền | Re-validate môi trường | Có thể quay lại màn hình kiểm tra | Lỗi nếu vẫn thiếu quyền sau cấp |
| `handleOnboardingFailed` | Khi pipeline thất bại | Đặt ERROR state | Ghi errorMessage | Người dùng có thể retry |
| `initializeApp` | Khi state INITIALIZING | Thiết lập lõi ứng dụng | Khởi tạo PoW, LocationNotes, start mesh, xử lý intent | Lỗi -> ERROR state |
| `handleNotificationIntent` | Mở từ notification | Điều hướng tới private/geohash chat | Clear notification tương ứng | - |
| `onResume` | Activity foreground | Đồng bộ trạng thái dịch vụ | Cập nhật state nếu BT/Location bị tắt | - |
| `onPause` | Activity background | Báo background cho mesh/chat | Điều chỉnh chiến lược kết nối | - |
| `onNewIntent` | Intent mới khi đang chạy | Xử lý notification nếu COMPLETE | Gọi handleNotificationIntent | - |
| `onDestroy` | Activity hủy | Cleanup managers & stop services | Dừng mesh nếu COMPLETE | Catch & log exceptions |

## 7. Quy trình Onboarding theo điều kiện
| User loại | Đường đi chính |
|----------|----------------|
| First-time | CHECKING -> (skip BT/Location/BatteryOpt) -> PERMISSION_EXPLANATION -> PERMISSION_REQUESTING -> handleOnboardingComplete -> INITIALIZING -> COMPLETE |
| Existing, đủ quyền & mọi thứ bật | CHECKING -> Bluetooth OK -> Location OK -> BatteryOpt DISABLED -> INITIALIZING -> COMPLETE |
| Existing, thiếu Bluetooth | CHECKING -> BLUETOOTH_CHECK -> (user bật) -> Location... |
| Existing, thiếu Location | CHECKING -> Bluetooth OK -> LOCATION_CHECK -> (user bật) -> BatteryOpt... |
| Existing, BatteryOpt ENABLED | CHECKING -> BT OK -> Location OK -> BATTERY_OPTIMIZATION_CHECK -> (user tắt/skip) -> PERMISSION / INITIALIZING |

## 8. Biện pháp độ tin cậy / UX
| Biện pháp | Lợi ích |
|----------|---------|
| Delay nhỏ (500ms/200ms/1000ms) | Tránh giật & race điều kiện hệ thống, cho cảm giác mượt. |
| Re-check permissions trong `initializeApp` | Ngăn trường hợp revoke giữa chừng. |
| Loading flags riêng | Phản hồi chính xác trạng thái chờ người dùng. |
| Back callback tùy chỉnh trong Chat | Điều hướng nội bộ mượt, tránh thoát app ngoài ý muốn. |

## 9. Edge Cases được xử lý
| Tình huống | Ứng xử |
|-----------|--------|
| Bluetooth NOT_SUPPORTED | Cho vào màn hình Bluetooth, sau đó có thể ERROR. |
| Location NOT_AVAILABLE | Chuyển LOCATION_CHECK rồi ERROR giữ thông báo. |
| BatteryOpt không hỗ trợ | Bỏ qua bước tắt. |
| Người dùng skip BatteryOpt | Lưu preference skip -> bỏ qua lần sau. |
| Quyền bị revoke trong initializeApp | Dừng init -> ERROR với thông báo cụ thể. |
| Notification intent đến trước COMPLETE | Xử lý chỉ sau khi đã COMPLETE (guard trong onNewIntent). |

## 10. Điểm có thể cải tiến
| Đề xuất | Lý do |
|---------|-------|
| Trích xuất state machine riêng (sealed classes cho transition event) | Test & bảo trì dễ hơn. |
| Gom các check môi trường vào một orchestrator đơn | Giảm lặp mã điều kiện tương tự. |
| Thêm timeout cho INITIALIZING | Báo lỗi nếu meshService không start sau X giây. |
| Analytics mỗi bước | Đo funnel onboarding. |
| Multi-language error messages | Nâng cao UX quốc tế. |

## 11. Tóm tắt nhanh
`MainActivity` triển khai một pipeline onboarding nhiều bước, có khả năng bỏ qua thông minh cho lần đầu, re-check quyền trước khi khởi chạy dịch vụ lõi, và cung cấp phòng vệ trước thay đổi môi trường (BT/Location) khi quay lại foreground. Mọi hàm được kích hoạt tuần tự hoặc do callback, đảm bảo không khởi tạo phần tốn tài nguyên (mesh/chat) trước khi các điều kiện nền tảng đạt.

---
Muốn thêm sơ đồ sequence hoặc refactor đề xuất? Hãy yêu cầu tiếp.
