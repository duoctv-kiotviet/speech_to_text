# Sử dụng TransactionId với Audio Recording

## Tổng quan

Bạn có thể truyền `transactionId` vào method `listen()` để tạo tên file ghi âm có cấu trúc:
```
speech_recording_<transactionId>_<timestamp>.m4a
```

Điều này giúp dễ dàng quản lý và tra cứu các file ghi âm theo transaction ID.

## Cách sử dụng

### Sử dụng với SpeechListenOptions

```dart
import 'package:speech_to_text/speech_to_text.dart';

class SpeechRecorderWithTransaction {
  final SpeechToText _speech = SpeechToText();
  
  Future<void> initialize() async {
    await _speech.initialize();
  }
  
  Future<void> startListeningWithTransaction(String transactionId) async {
    await _speech.listen(
      onResult: (result) {
        print('Recognized: ${result.recognizedWords}');
      },
      onRecordingComplete: (filePath) {
        print('Recording saved: $filePath');
        // File sẽ có tên: speech_recording_TRANS123_1234567890.m4a
        // Trong đó:
        // - TRANS123 là transactionId
        // - 1234567890 là timestamp
      },
      listenOptions: SpeechListenOptions(
        transactionId: transactionId,
        partialResults: true,
        onDevice: false,
      ),
    );
  }
  
  Future<void> stop() async {
    await _speech.stop();
  }
}
```

### Ví dụ thực tế

```dart
import 'package:flutter/material.dart';
import 'package:speech_to_text/speech_to_text.dart';
import 'dart:io';

class OrderSpeechRecorder extends StatefulWidget {
  final String orderId;
  
  const OrderSpeechRecorder({Key? key, required this.orderId}) : super(key: key);
  
  @override
  State<OrderSpeechRecorder> createState() => _OrderSpeechRecorderState();
}

class _OrderSpeechRecorderState extends State<OrderSpeechRecorder> {
  final SpeechToText _speech = SpeechToText();
  bool _isListening = false;
  String _text = '';
  String? _recordingPath;
  
  @override
  void initState() {
    super.initState();
    _initSpeech();
  }
  
  Future<void> _initSpeech() async {
    await _speech.initialize();
  }
  
  Future<void> _startListening() async {
    setState(() {
      _isListening = true;
      _text = '';
      _recordingPath = null;
    });
    
    // Sử dụng orderId làm transactionId
    await _speech.listen(
      onResult: (result) {
        setState(() {
          _text = result.recognizedWords;
        });
      },
      onRecordingComplete: (filePath) {
        setState(() {
          _recordingPath = filePath;
        });
        
        // File name sẽ là: speech_recording_ORDER123_1234567890.m4a
        print('Order ${widget.orderId} recording saved: $filePath');
        
        // Có thể upload file với orderId
        _uploadRecording(filePath, widget.orderId);
      },
      listenOptions: SpeechListenOptions(
        transactionId: widget.orderId,
        partialResults: true,
      ),
    );
  }
  
  Future<void> _stopListening() async {
    await _speech.stop();
    setState(() {
      _isListening = false;
    });
  }
  
  Future<void> _uploadRecording(String filePath, String orderId) async {
    // Upload file lên server với orderId
    try {
      final file = File(filePath);
      // ... upload logic ...
      print('Uploading recording for order: $orderId');
      
      // Sau khi upload thành công, xóa file local
      await file.delete();
    } catch (e) {
      print('Error uploading: $e');
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Order ${widget.orderId} - Voice Note'),
      ),
      body: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          children: [
            Text('Order ID: ${widget.orderId}'),
            SizedBox(height: 20),
            Text(_text.isEmpty ? 'Tap mic to start...' : _text),
            SizedBox(height: 20),
            if (_recordingPath != null) ...[
              Text('Recording saved!'),
              Text(
                'File: ${_recordingPath!.split('/').last}',
                style: TextStyle(fontSize: 12, color: Colors.grey),
              ),
            ],
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _isListening ? _stopListening : _startListening,
        child: Icon(_isListening ? Icons.stop : Icons.mic),
      ),
    );
  }
}
```

## Use Cases

### 1. E-commerce / Order Management
```dart
// Ghi âm ghi chú cho đơn hàng
await speech.listen(
  listenOptions: SpeechListenOptions(
    transactionId: 'ORDER_12345',
  ),
  onRecordingComplete: (filePath) {
    // File: speech_recording_ORDER_12345_1234567890.m4a
    uploadOrderNote(orderId: 'ORDER_12345', audioFile: filePath);
  },
);
```

### 2. Customer Support / Tickets
```dart
// Ghi âm hỗ trợ khách hàng
await speech.listen(
  listenOptions: SpeechListenOptions(
    transactionId: 'TICKET_${ticketNumber}',
  ),
  onRecordingComplete: (filePath) {
    // File: speech_recording_TICKET_789_1234567890.m4a
    attachAudioToTicket(ticketNumber, filePath);
  },
);
```

### 3. Healthcare / Patient Records
```dart
// Ghi âm ghi chú bệnh nhân
await speech.listen(
  listenOptions: SpeechListenOptions(
    transactionId: 'PATIENT_${patientId}',
  ),
  onRecordingComplete: (filePath) {
    // File: speech_recording_PATIENT_456_1234567890.m4a
    savePatientNote(patientId, filePath);
  },
);
```

### 4. Meeting / Conference Notes
```dart
// Ghi âm cuộc họp
await speech.listen(
  listenOptions: SpeechListenOptions(
    transactionId: 'MEETING_${meetingId}_${DateTime.now().toIso8601String()}',
  ),
  onRecordingComplete: (filePath) {
    saveMeetingRecording(meetingId, filePath);
  },
);
```

## Lợi ích

1. **Dễ dàng tra cứu**: Tên file chứa transaction ID giúp dễ dàng tìm kiếm
2. **Tổ chức tốt hơn**: Files được nhóm theo transaction/order/ticket
3. **Audit trail**: Có thể theo dõi lịch sử ghi âm theo ID
4. **Upload hiệu quả**: Server có thể dễ dàng liên kết file với record tương ứng

## Lưu ý

- `transactionId` là optional - nếu không truyền, file name sẽ là `speech_recording_<timestamp>.m4a`
- `transactionId` nên là string hợp lệ cho file name (tránh ký tự đặc biệt như `/`, `\`, `:`, v.v.)
- Timestamp luôn được thêm vào để đảm bảo tên file unique
- Format file: MPEG-4 AAC (.m4a)

## Ví dụ tên file

Với `transactionId = "ORDER_12345"` và timestamp `1709876543`:
```
iOS/macOS: speech_recording_ORDER_12345_1709876543.m4a
Android:   speech_recording_ORDER_12345_1709876543.m4a
```

Không có `transactionId`:
```
iOS/macOS: speech_recording_1709876543.m4a
Android:   speech_recording_1709876543.m4a
```

