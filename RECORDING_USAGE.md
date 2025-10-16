# Audio Recording Feature

## Overview

The speech_to_text plugin now supports automatic audio recording during speech recognition sessions. When you call the `listen()` method, the plugin will simultaneously record the audio and provide the file path via a callback when recording is complete.

## Usage

### Basic Example

```dart
import 'package:speech_to_text/speech_to_text.dart';
import 'dart:io';

class SpeechRecorder {
  final SpeechToText _speech = SpeechToText();
  String? _lastRecordingPath;
  
  Future<void> initialize() async {
    bool available = await _speech.initialize(
      onError: (error) => print('Error: ${error.errorMsg}'),
      onStatus: (status) => print('Status: $status'),
    );
    
    if (!available) {
      print('Speech recognition not available');
    }
  }
  
  Future<void> startListening() async {
    await _speech.listen(
      onResult: (result) {
        print('Recognized: ${result.recognizedWords}');
      },
      onRecordingComplete: (filePath) {
        print('Recording saved to: $filePath');
        _lastRecordingPath = filePath;
        // You can now use this file path to:
        // - Upload to server
        // - Play back the audio
        // - Process the audio file
        // - etc.
      },
    );
  }
  
  Future<void> stopListening() async {
    await _speech.stop();
  }
  
  // Clean up the recording file when done
  Future<void> deleteRecording() async {
    if (_lastRecordingPath != null) {
      try {
        final file = File(_lastRecordingPath!);
        if (await file.exists()) {
          await file.delete();
          print('Recording file deleted');
        }
      } catch (e) {
        print('Error deleting recording: $e');
      }
    }
  }
}
```

### Complete Example with UI

```dart
import 'package:flutter/material.dart';
import 'package:speech_to_text/speech_to_text.dart';
import 'dart:io';

class SpeechRecorderPage extends StatefulWidget {
  @override
  _SpeechRecorderPageState createState() => _SpeechRecorderPageState();
}

class _SpeechRecorderPageState extends State<SpeechRecorderPage> {
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
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Recording saved!')),
        );
      },
    );
  }
  
  Future<void> _stopListening() async {
    await _speech.stop();
    setState(() {
      _isListening = false;
    });
  }
  
  Future<void> _deleteRecording() async {
    if (_recordingPath != null) {
      try {
        await File(_recordingPath!).delete();
        setState(() {
          _recordingPath = null;
        });
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Recording deleted')),
        );
      } catch (e) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Error deleting: $e')),
        );
      }
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Speech Recorder')),
      body: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          children: [
            Text(_text.isEmpty ? 'Start speaking...' : _text),
            SizedBox(height: 20),
            if (_recordingPath != null) ...[
              Text('Recording saved to:'),
              Text(_recordingPath!, style: TextStyle(fontSize: 12)),
              ElevatedButton(
                onPressed: _deleteRecording,
                child: Text('Delete Recording'),
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

## File Storage

### iOS/macOS
Audio files are saved to the app's temporary directory with the format:
- `speech_recording_<timestamp>.m4a`
- Location: `NSTemporaryDirectory()`
- Format: MPEG-4 AAC
- Quality: High

### Android
Audio files are saved to the app's cache directory with the format:
- `speech_recording_<timestamp>.m4a`
- Location: `Context.getCacheDir()`
- Format: MPEG-4 AAC
- Sample Rate: 44100 Hz
- Bit Rate: 128 kbps

## File Cleanup

### Automatic Cleanup
The operating system will automatically clean up files in temporary/cache directories when:
- The device runs low on storage
- The app is uninstalled
- After a certain period (iOS typically ~3 days for temp files)

### Manual Cleanup
You should delete recording files when you're done with them to save storage space:

```dart
Future<void> cleanupRecording(String filePath) async {
  try {
    final file = File(filePath);
    if (await file.exists()) {
      await file.delete();
      print('File deleted successfully');
    }
  } catch (e) {
    print('Error deleting file: $e');
  }
}
```

### Cleanup on App Termination
Consider cleaning up old recordings when your app starts:

```dart
import 'dart:io';
import 'package:path_provider/path_provider.dart';

Future<void> cleanupOldRecordings() async {
  try {
    final Directory tempDir = await getTemporaryDirectory();
    final List<FileSystemEntity> files = tempDir.listSync();
    
    for (var file in files) {
      if (file is File && file.path.contains('speech_recording_')) {
        // Delete files older than 24 hours
        final FileStat stat = await file.stat();
        final DateTime modifiedTime = stat.modified;
        final Duration age = DateTime.now().difference(modifiedTime);
        
        if (age.inHours > 24) {
          await file.delete();
          print('Deleted old recording: ${file.path}');
        }
      }
    }
  } catch (e) {
    print('Error cleaning up old recordings: $e');
  }
}
```

## Best Practices

1. **Always handle the recording callback**: Even if you don't need the recording immediately, store the path so you can clean it up later.

2. **Delete files when done**: Don't rely solely on OS cleanup. Delete files as soon as you're done with them.

3. **Handle errors**: Wrap file operations in try-catch blocks to handle potential I/O errors.

4. **Check file existence**: Always check if a file exists before trying to delete it.

5. **Consider privacy**: Audio recordings may contain sensitive information. Ensure proper security measures when:
   - Uploading to servers
   - Storing for extended periods
   - Sharing with other apps

## Permissions

No additional permissions are required beyond the existing microphone permissions:

- **iOS**: `NSMicrophoneUsageDescription` in Info.plist
- **Android**: `RECORD_AUDIO` permission in AndroidManifest.xml

## Limitations

1. **Simultaneous recording**: Only one recording can be active at a time (one per listen session).

2. **File format**: The format is fixed (MPEG-4 AAC) and cannot be changed.

3. **Storage space**: Ensure your app has sufficient storage space. A typical 1-minute recording is approximately 1-2 MB.

4. **Battery usage**: Audio recording increases battery consumption, especially for longer sessions.

## Troubleshooting

### Recording file not created
- Check that microphone permissions are granted
- Ensure there's sufficient storage space
- Check logs for error messages

### File path is empty or null
- The callback is only called when recording completes successfully
- If listening is canceled (vs stopped), the callback may not be called
- Check for errors in the speech recognition session

### Cannot access the file
- The file path is absolute and can be accessed directly
- Ensure the file wasn't already deleted
- Check that the app has permission to access the file

## Support

For issues or questions, please refer to the main plugin documentation or file an issue on GitHub.

