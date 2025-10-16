# Audio Recording Feature - Implementation Summary

## Tổng quan (Overview)

Đã thêm tính năng ghi âm tự động khi sử dụng speech-to-text. Khi gọi `listen()`, plugin sẽ đồng thời:
1. Nhận dạng giọng nói thành văn bản
2. Ghi âm audio vào file
3. Callback về đường dẫn file khi hoàn tất

## Các thay đổi đã thực hiện

### 1. Dart Layer (`speech_to_text/lib/speech_to_text.dart`)
- ✅ Thêm `SpeechRecordingComplete` typedef cho callback
- ✅ Thêm parameter `onRecordingComplete` vào method `listen()`
- ✅ Thêm constant `recordingCompleteMethod` cho platform channel
- ✅ Thêm handler `_onRecordingComplete()` để xử lý callback từ native

### 2. Platform Interface (`speech_to_text_platform_interface/`)
- ✅ Thêm `onRecordingComplete` callback vào `SpeechToTextPlatform`
- ✅ Thêm `recordingCompleteMethod` constant
- ✅ Thêm xử lý callback trong `_handleCallbacks()`

### 3. iOS Implementation (Swift)
**File:** `speech_to_text/darwin/speech_to_text/Sources/speech_to_text/SpeechToTextPlugin.swift`

Thay đổi:
- ✅ Thêm enum case `recordingComplete` vào `SwiftSpeechToTextCallbackMethods`
- ✅ Thêm biến `audioFile` và `recordingFilePath`
- ✅ Khởi tạo `AVAudioFile` trong `listenForSpeech()`
- ✅ Ghi audio buffer vào file trong `installTap` callback
- ✅ Finalize và callback file path trong `stopCurrentListen()`
- ✅ File format: MPEG-4 AAC, high quality
- ✅ File location: `NSTemporaryDirectory()`

### 4. Android Implementation (Kotlin)
**File:** `speech_to_text/android/src/main/kotlin/com/csdcorp/speech_to_text/SpeechToTextPlugin.kt`

Thay đổi:
- ✅ Thêm enum value `recordingComplete` vào `SpeechToTextCallbackMethods`
- ✅ Thêm imports: `MediaRecorder`, `AudioFormat`, `AudioRecord`, `File`, `FileOutputStream`
- ✅ Thêm biến `mediaRecorder` và `recordingFilePath`
- ✅ Thêm method `startAudioRecording()` - khởi tạo và bắt đầu ghi âm
- ✅ Thêm method `stopAudioRecording()` - dừng và callback file path
- ✅ Tích hợp vào `startListening()` và `notifyListening()`
- ✅ File format: MPEG-4 AAC, 44100Hz, 128kbps
- ✅ File location: `Context.getCacheDir()`

### 5. Documentation
- ✅ `RECORDING_USAGE.md` - Hướng dẫn sử dụng chi tiết
- ✅ `AUDIO_RECORDING_FEATURE.md` - Tài liệu tổng quan

## Cách sử dụng

```dart
import 'package:speech_to_text/speech_to_text.dart';

SpeechToText speech = SpeechToText();

await speech.initialize();

await speech.listen(
  onResult: (result) {
    print('Text: ${result.recognizedWords}');
  },
  onRecordingComplete: (filePath) {
    print('Audio saved to: $filePath');
    // Sử dụng file path để:
    // - Upload lên server
    // - Phát lại audio
    // - Xử lý file
    // - etc.
  },
);

await speech.stop(); // Callback sẽ được gọi sau khi stop
```

## Đặc điểm kỹ thuật

### iOS/macOS
- **Format**: MPEG-4 AAC
- **Quality**: High
- **Location**: Temporary Directory
- **Filename**: `speech_recording_<timestamp>.m4a`

### Android
- **Format**: MPEG-4 AAC
- **Sample Rate**: 44100 Hz
- **Bit Rate**: 128 kbps
- **Location**: Cache Directory
- **Filename**: `speech_recording_<timestamp>.m4a`

## File Cleanup

### Tự động
- iOS: Hệ thống tự xóa sau ~3 ngày hoặc khi thiếu dung lượng
- Android: Hệ thống tự xóa khi thiếu dung lượng

### Thủ công (Khuyến nghị)
```dart
import 'dart:io';

Future<void> deleteRecording(String filePath) async {
  try {
    final file = File(filePath);
    if (await file.exists()) {
      await file.delete();
    }
  } catch (e) {
    print('Error: $e');
  }
}
```

## Lưu ý quan trọng

1. **Không cần permission mới**: Sử dụng permission RECORD_AUDIO đã có
2. **File size**: ~1-2 MB cho 1 phút ghi âm
3. **Battery**: Ghi âm tốn pin, nên giới hạn thời gian
4. **Privacy**: File chứa audio nhạy cảm, cần bảo mật
5. **Cleanup**: Nên xóa file ngay sau khi sử dụng xong

## Testing

Để test tính năng này:

1. Chạy example app:
```bash
cd speech_to_text/example
flutter run
```

2. Test các trường hợp:
   - ✅ Start listen → stop → kiểm tra callback nhận được file path
   - ✅ Kiểm tra file tồn tại tại đường dẫn được trả về
   - ✅ Phát file audio để verify chất lượng
   - ✅ Test cleanup: xóa file sau khi sử dụng
   - ✅ Test với các thời lượng khác nhau
   - ✅ Test cancel (callback có thể không được gọi)

## Troubleshooting

### Callback không được gọi
- Kiểm tra có gọi `stop()` không (cancel có thể không trigger callback)
- Kiểm tra logs để xem có lỗi ghi âm không

### File không tồn tại
- Kiểm tra permission microphone
- Kiểm tra storage space
- Xem logs native để tìm lỗi

### Chất lượng audio kém
- iOS: Format và quality đã được set high
- Android: Sample rate 44100Hz và bit rate 128kbps là standard

## Future Enhancements (Tương lai)

Có thể mở rộng thêm:
- [ ] Tùy chọn file format (WAV, MP3, etc.)
- [ ] Tùy chỉnh quality/bitrate
- [ ] Tự động cleanup files cũ
- [ ] Compression options
- [ ] Streaming upload trong khi ghi

## Kết luận

Tính năng đã được implement đầy đủ trên cả iOS và Android với:
- ✅ API đơn giản, dễ sử dụng
- ✅ File format chuẩn (MPEG-4 AAC)
- ✅ Tự động lưu vào temp/cache directory
- ✅ Callback rõ ràng với file path
- ✅ Documentation đầy đủ
- ✅ Error handling cẩn thận

Tất cả code đã được thêm vào và sẵn sàng sử dụng!

