<!-- Author: GENERATED | Date: 2025-11-12 -->
# ChatScreen & ChatViewModel Kiến trúc + Luồng gửi tin nhắn

## 1. Mục tiêu tổng quan
`ChatScreen` là composable điều phối UI chat theo kiến trúc component, còn `ChatViewModel` là coordinator xử lý logic: gửi/nhận tin nhắn, private chat, kênh công khai, kênh geohash (Nostr), gợi ý lệnh, gợi ý mention, media (ảnh, file, voice). Mục tiêu:
- Gom toàn bộ trạng thái UI động qua LiveData (iOS compatibility pattern).
- Trừu tượng hóa xử lý message qua các manager chuyên trách (MessageManager, ChannelManager, PrivateChatManager, MediaSendingManager, GeohashViewModel...).
- Đa tuyến đường gửi: Mesh Bluetooth vs Nostr (geohash / fallback private khi chưa handshake).

## 2. Phân tách vai trò
| Thành phần | Vai trò chính |
|------------|---------------|
| `ChatScreen` | Kết xuất UI: header nổi, danh sách tin nhắn, input, sidebar, sheets, dialogs. |
| `ChatViewModel` | Điều phối logic nghiệp vụ gửi/nhận, quản lý state tổng. |
| `MessageManager` | Quản lý danh sách tin nhắn timeline Mesh & system messages. |
| `ChannelManager` | Quản lý kênh public (join/leave, tin nhắn channel, mã hóa nếu có key). |
| `PrivateChatManager` | Quản lý private chat, handshake Noise, gửi tin riêng. |
| `MediaSendingManager` | Gửi voice/image/file, cập nhật trạng thái transfer. |
| `GeohashViewModel` | Gửi & nhận tin Nostr theo geohash, quản lý participant & DMs geohash. |
| `MeshDelegateHandler` | Adapter callbacks từ `BluetoothMeshService` sang cập nhật state. |
| `NotificationManager` | Thông báo nền, đánh dấu đọc, xóa khi user xem. |
| `DataManager` | Persist nickname, favorites, blocked. |

## 3. Trạng thái UI chính (LiveData)
| LiveData | Ý nghĩa |
|----------|--------|
| `messages` | Tin nhắn Mesh timeline (mặc định). |
| `channelMessages` | Map channel -> list tin nhắn. |
| `privateChats` | Map peerID -> list tin nhắn riêng. |
| `selectedPrivateChatPeer` | Peer hiện đang chat riêng. |
| `currentChannel` | Channel public hiện đang xem. |
| `selectedLocationChannel` | Channel geohash (Location vs Mesh). |
| `showSidebar`, `showAppInfo`, `showPasswordPrompt` | Các màn/overlay phụ. |
| `commandSuggestions`, `mentionSuggestions` | Gợi ý lệnh & mention realtime. |
| `nickname` | Nickname hiển thị (sender). |

`ChatScreen` chọn dữ liệu để hiển thị theo ưu tiên: private -> channel -> geohash -> mesh.

## 4. Kiến trúc UI trong `ChatScreen`
- Header nổi: `ChatFloatingHeader` (AppBar compact + status bar overlay).
- Nội dung chính: `MessagesList` (scroll, long-press, mention injection, media preview).
- Input: `ChatInputSection` (textbox + media buttons + gợi ý lệnh/mention).
- Overlay: Sidebar trượt vào, các sheet (Location Channels, Location Notes), dialog password, debug sheet, user action sheet.
- Image Viewer full-screen tách biệt để không ảnh hưởng state sidebar.
- Scroll-to-bottom FAB hiển thị khi người dùng cuộn lên.

## 5. Injection / Bridge ngoài
`FileShareDispatcher.setHandler { peer, channel, path -> viewModel.sendFileNote(...) }` được cài trong composition (LaunchedEffect) để nhận file share từ tầng thấp (input component) → ViewModel.

## 6. Luồng gửi tin nhắn (Message Sending Flow)
### 6.1 Phân nhánh ban đầu trong `sendMessage(content)`
```
if content.isEmpty -> return
if startsWith('/') -> xử lý command
else -> parse mentions -> phân loại:
    a) Private chat đang mở? -> gửi riêng
    b) Đang ở geohash Location channel? -> gửi qua Nostr (geohashViewModel)
    c) Channel public? -> gửi tin channel (có thể mã hóa nếu có key)
    d) Mặc định Mesh timeline -> gửi broadcast mesh
```

### 6.2 Gửi Command
- `commandProcessor.processCommand(...)` nhận lambda thực thi gửi.
- Nếu đang ở geohash location: lambda dùng `geohashViewModel.sendGeohashMessage(...)`.
- Ngược lại: `meshService.sendMessage(messageContent, mentions, channel)`.

### 6.3 Gửi Private Message
1. Resolve alias → canonical peer qua `ConversationAliasResolver` (giải quyết noise key / nostr alias).
2. `privateChatManager.startPrivateChat(canonical, meshService)` nếu thay đổi.
3. `privateChatManager.sendPrivateMessage(content, peer, recipientNickname, myNickname, myPeerID, callback)`.
4. Callback thực thi router:
```
router = MessageRouter.getInstance(app, meshService)
router.sendPrivate(...)
  - Ưu tiên mesh nếu Noise session established
  - Fallback Nostr nếu chưa handshake
```

### 6.4 Gửi Geohash Channel Message
- Kiểm tra `selectedLocationChannel is ChannelID.Location`.
- Gọi `geohashViewModel.sendGeohashMessage(content, channelInfo, myPeerID, nickname)`.
- Tạo event Nostr ephemeral → route qua Tor (nếu bật) hoặc direct Nostr relays.

### 6.5 Gửi Channel Public Message (Mesh)
1. Tạo `BitchatMessage` local (optimistic) → thêm vào `channelManager.addChannelMessage(channel, message, myPeerID)`.
2. Kiểm tra mã hóa: `channelManager.hasChannelKey(channel)`.
3. Nếu có key: `channelManager.sendEncryptedChannelMessage(...)` với callbacks:
   - `onEncryptedPayload` hiện tại vẫn gửi payload plaintext → TODO cải tiến (chèn encrypted binary vào packet).
   - `onFallback` gửi plaintext nếu mã hóa thất bại.
4. Nếu không mã hóa: `meshService.sendMessage(content, mentions, channel)`.

### 6.6 Gửi Mesh Broadcast (không channel)
1. Tạo `BitchatMessage` → `messageManager.addMessage(message)`.
2. `meshService.sendMessage(content, mentions, null)`.

### 6.7 Mentions
- `messageManager.parseMentions(content, knownNicknames, myNickname)` trích `@nickname`.
- Đưa `mentions` vào message để client khác xử lý highlight / notification.

### 6.8 Media (Voice/Image/File)
Các hàm:
```
sendVoiceNote(peer?, channel?, path)
sendImageNote(peer?, channel?, path)
sendFileNote(peer?, channel?, path)
```
Delegated hoàn toàn tới `mediaSendingManager`, quản lý:
- Tạo placeholder message với trạng thái transfer (progress).
- Bắt sự kiện từ `TransferProgressManager.events.collect` để cập nhật deliveryStatus.
- Hủy transfer: `cancelMediaSend(messageId)`.

### 6.9 Favoriting & Notification route
Khi toggle favorite:
- Cập nhật persistence (FavoritesPersistenceService, DataManager).
- Gửi thông báo [FAVORITED]/[UNFAVORITED] qua private mesh nếu session established, nếu không thì Nostr fallback.

## 7. Bảo đảm thứ tự / An toàn dữ liệu
| Biện pháp | Mô tả |
|----------|-------|
| Optimistic insert trước gửi (channel & mesh) | UI phản hồi tức thì, ngay cả khi mạng chậm. |
| Alias resolve trước private send | Tránh gửi vào alias sai / session chưa thiết lập. |
| Fallback Nostr cho private chưa handshake | Đảm bảo message không bị kẹt. |
| Retry nội bộ (ngầm) qua router khi session vừa established | `MessageRouter.onSessionEstablished(peer)` flush outbox. |
| Encrypted channel fallback plaintext | Không mất message dù thất bại mã hóa. |

## 8. Các điểm mở rộng / cải tiến
| Vấn đề | Đề xuất |
|--------|---------|
| Encrypted channel gửi plaintext trong onEncryptedPayload | Gửi thực sự cipher text qua mesh packet custom type. |
| Không xác thực PoW với geohash message | Kết hợp PoWPreferenceManager trên gửi Nostr công khai. |
| Không batching mention parse | Cache nickname set, tránh tạo set mỗi lần. |
| Reflection trong `ensureGeohashDMSubscriptionIfNeeded` | Thay bằng public API trong GeohashViewModel. |
| Lặp updateReactiveStates mỗi giây | Thay bằng event-driven callbacks từ PeerManager. |

## 9. Edge Cases gửi tin
| Tình huống | Xử lý hiện tại |
|-----------|----------------|
| Nội dung rỗng | return sớm. |
| Command trong geohash | Route qua Nostr. |
| Private peer alias noiseHex | Resolve canonical trước gửi. |
| Channel có key nhưng mã hóa lỗi | Fallback plaintext gửi. |
| Mentions không tồn tại | Chỉ parse nickname đã biết, bỏ qua phần còn lại. |
| Geohash DM chưa subscription | `ensureGeohashDMSubscriptionIfNeeded` cố gắng đăng ký. |

## 10. Trình tự gửi Private Message (Sequence)
```
User nhập -> ChatInputSection onSend -> viewModel.sendMessage()
  -> không phải command, có selectedPrivatePeer?
    -> Resolve alias canonical
    -> privateChatManager.sendPrivateMessage(..., callback)
       -> callback -> MessageRouter.sendPrivate(...)
          -> if mesh session established: encrypt & gửi BLE
          -> else: wrap Nostr event & gửi relays
    -> UI hiển thị tin nhắn private (optimistic) + chờ ack/readReceipt
```

## 11. Trình tự gửi Geohash Channel Message
```
sendMessage(): selectedLocationChannel is Location?
  -> geohashViewModel.sendGeohashMessage(content,...)
     -> derive identity (geohash-based)
     -> build Nostr event (tags: geohash, optional nickname)
     -> sign & gửi relays (TorManager SOCKS nếu bật)
     -> update participant counts / local echo
```

## 12. Trình tự gửi Channel Public Message
```
sendMessage(): currentChannel != null
  -> create BitchatMessage local -> addChannelMessage()
  -> hasChannelKey?
     YES -> encrypt attempt -> onEncryptedPayload/fallback -> meshService.sendMessage()
     NO  -> meshService.sendMessage()
  -> Delivery ack -> cập nhật trạng thái UI (qua meshDelegateHandler)
```

## 13. Chức năng chính tóm tắt
| Hàm | Chức năng cốt lõi |
|-----|------------------|
| `sendMessage` | Pipeline phân loại & gửi mỗi loại tin nhắn. |
| `sendVoice/Image/FileNote` | Gửi media qua MediaSendingManager + progress. |
| `startPrivateChat` | Khởi private chat + DM subscription (geohash). |
| `joinChannel` | Tham gia kênh & khởi tạo danh sách tin nhắn. |
| `switchToChannel` | Chọn kênh đang xem. |
| `geohashViewModel.sendGeohashMessage` | Gửi tin nhắn Nostr định vị (geohash). |
| `toggleFavorite` | Cập nhật trạng thái favorite + thông báo. |
| `panicClearAllData` | Xóa mọi dữ liệu nhạy cảm & reset nickname. |

## 14. Bảo mật & Quyền riêng tư
| Biện pháp | Mô tả |
|----------|-------|
| Noise handshake trước mesh private | Bảo đảm mã hóa end-to-end khi có thể. |
| Fallback Nostr khi chưa session | Tránh mất tin; vẫn cần xem xét mã hóa (gift wrap). |
| Channel key encryption | Mã hóa nội dung kênh nếu key tồn tại. |
| Panic mode | Xóa khóa, identity, favorites, geohash bookmarks, Clears cryptographic state. |
| Geohash identity derivation | Tạo danh tính riêng theo vùng, giảm liên kết giữa vùng khác nhau. |

## 15. Gợi ý test đơn vị
| Test | Mục tiêu |
|------|----------|
| sendMessage private fallback Nostr | Khi chưa handshake session. |
| sendMessage channel encrypted fallback | Giả lập lỗi mã hóa. |
| sendMessage geohash route | Khi `selectedLocationChannel` là Location. |
| command routing in geohash | Lệnh gửi đúng qua Nostr. |
| mention parsing uniqueness | Không duplicate mentions. |
| panicClearAllData side-effects | Dữ liệu & danh sách tin về rỗng, nickname đổi. |

## 16. Kết luận
Kiến trúc phân lớp: UI (Compose) nhẹ, logic ViewModel chia thành managers chuyên trách, giúp mở rộng dễ và giữ tương thích iOS. Luồng gửi tin nhắn cover đa đường truyền (Mesh, Nostr), fallback an toàn, và cơ chế optimistic UI. Cải tiến tiếp theo: mã hóa channel thực thụ, tối ưu alias resolution, chuyển phản ứng session sang event-driven, và tích hợp PoW vào geohash.

---
Nếu cần mình bổ sung sơ đồ sequence (Mermaid/PlantUML) hoặc triển khai PoW cho geohash gửi tin, hãy yêu cầu tiếp.

