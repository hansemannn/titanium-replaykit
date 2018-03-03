# iOS 11+ ReplayKit API

Record or stream video from the screen, and audio from the app and microphone 
in Titanium and Hyperloop.

Using the [ReplayKit](https://developer.apple.com/documentation/replaykit) framework, users can record video from the screen, and audio 
from the app and microphone. They can then share their recordings with other users 
through email, messages, and social media. You can build app extensions for live 
broadcasting your content to sharing services.

## Requirements

- [x] iOS 11+
- [x] Titanium SDK 7.1.0 (`appc sdk install -b 7_1_X`)
- [x] Hyperloop 3.1.0
- [x] An iOS device (I was not able to run the native API's on the Simulator)

(Alternatively, you can use `require()` statements with SDK 7.0.2 and Hyperloop 3.0.2)

## Usage

1. Create a new file `ti.screenrecorder.js` in your project
2. Paste the contents of the "TiScreenRecorder" class in it
3. Include the following snippet into your code
```js
import TiScreenRecorder from 'ti.screenrecorder';

const screenRecorder = new TiScreenRecorder({
  filePath: `${Ti.Filesystem.applicationDataDirectory}/recording-${(new Date()).getTime()}.mp4`,
  callback: (recording, err) => {
    if (err) {
      alert(`Error recording screen: ${err}`)
      return;
    }
    const message = `Success! See recording in ${recording.nativePath}`
    alert(message);
    Ti.API.info(message);
  }
});

// Start
// screenRecorder.startCapture();

// Stop
// screenRecorder.stopCapture();
```
4. Thats it! You can now capture the screen and mic-audio!

(App-audio is not supported so far, but can be done the same way than the mic-audio ~ via `AVAssetWriterInput`)

## Example

```js
import {
  RPScreenRecorder,
  ReplayKit
} from 'ReplayKit';

import {
  AVAssetWriter,
  AVAssetWriterInput,
  AVCaptureDevice,
  AVFoundation
} from 'AVFoundation';

import { NSURL, NSError } from 'Foundation';

import { UIScreen } from 'UIKit';

import { CoreMedia } from 'CoreMedia';

const AVVideoCodecKey = AVFoundation.AVVideoCodecKey;
const AVVideoWidthKey = AVFoundation.AVVideoWidthKey;
const AVVideoHeightKey = AVFoundation.AVVideoHeightKey;

const AVFormatIDKey = AVFoundation.AVFormatIDKey;
const AVNumberOfChannelsKey = AVFoundation.AVNumberOfChannelsKey;
const AVSampleRateKey = AVFoundation.AVSampleRateKey;
const AVEncoderBitRateKey = AVFoundation.AVEncoderBitRateKey;

export default class TiScreenRecorder {
  constructor(args = {}) {
    const filePath = args.filePath;
    const callback = args.callback;

    if (!filePath) {
      Ti.API.error('Missing `filePath` property!');
      return;
    }

    if (!callback) {
      Ti.API.error('Missing `callback` property!');
      return;
    }

    this.filePath = filePath;
    this.callback = callback;

    this.isMicEnabled = false;
    this.videoSessionStarted = false;
    this.micSessionStarted = false;
  }
  startCapture() {
    if (!RPScreenRecorder.sharedRecorder().isAvailable()) {
      callback(null, 'Screen Recorder is not available!');
      return;
    }

    // Prepare asset writer
    const fileURL = NSURL.fileURLWithPath(this.filePath);
    this.assetWriter = AVAssetWriter.assetWriterWithURLFileTypeError(fileURL, AVFoundation.AVFileTypeMPEG4, null); // TODO: Handle error?
    this.assetWriter.movieTimeScale = 60

    // Video input
    const videoOutputSettings = {
      AVVideoCodecKey: AVFoundation.AVVideoCodecTypeH264,
      AVVideoWidthKey: UIScreen.mainScreen.bounds.size.width,
      AVVideoHeightKey: UIScreen.mainScreen.bounds.size.height
    };

    this.videoInput  = AVAssetWriterInput.assetWriterInputWithMediaTypeOutputSettings(AVFoundation.AVMediaTypeVideo, videoOutputSettings);
    this.videoInput.expectsMediaDataInRealTime = true;
    this.videoInput.mediaTimeScale = 60
    this.assetWriter.addInput(this.videoInput);

    const audioOutputSettings = {
      AVFormatIDKey: 1633772320,
      AVNumberOfChannelsKey: 1,
      AVSampleRateKey: 44100.0,
      AVEncoderBitRateKey: 64000
    };

    this.audioInput = AVAssetWriterInput.assetWriterInputWithMediaTypeOutputSettings(AVFoundation.AVMediaTypeAudio, audioOutputSettings);
    this.audioInput.expectsMediaDataInRealTime = true;
    this.assetWriter.addInput(this.audioInput);

    RPScreenRecorder.sharedRecorder().microphoneEnabled = true;

    if (!this.assetWriter.startWriting()) {
      this.callback(null, this.assetWriter.error.localizedDescription);
      return;
    }

    // Start screen capture
    RPScreenRecorder.sharedRecorder().startCaptureWithHandlerCompletionHandler((sample, bufferType, error) => {
      if (!CoreMedia.CMSampleBufferDataIsReady(sample)) {
        Ti.API.warn('Not ready for writing ...');
        return;
      }

      if (this.assetWriter.status === AVFoundation.AVAssetWriterStatusFailed) {
        Ti.API.error(`Error handling writer: ${this.assetWriter.error.localizedDescription}`);
        this.callback(null, assetWriter.error.localizedDescription);
        return;
      }

      if (this.assetWriter.status !== AVFoundation.AVAssetWriterStatusWriting) {
        Ti.API.warn(`Not writing ... Status = ${this.assetWriter.status}`);
        return;
      }

      switch (bufferType) {
        case ReplayKit.RPSampleBufferTypeVideo:
          if (!this.videoSessionStarted) {
            Ti.API.info('Starting video session ...');
            this.videoSessionStarted = true;
            this.assetWriter.startSessionAtSourceTime(CMSampleBufferGetPresentationTimeStamp(sample));
          }

          if (this.videoInput.isReadyForMoreMediaData) {
            this.videoInput.appendSampleBuffer(sampleBuffer);
          }
          break;
        case ReplayKit.RPSampleBufferTypeAudioMic:
          if (!this.isMicEnabled) {
            Ti.API.warn('Received audio buffer, but mic not enabled. Skipping ...');
            break;
          }

          if (!this.micSessionStarted) {
            Ti.API.info('Starting audio session ...');
            this.micSessionStarted = true;
            this.assetWriter.startSessionAtSourceTime(CMSampleBufferGetPresentationTimeStamp(sample));
          }

          if (this.audioInput.isReadyForMoreMediaData) {
            this.audioInput.appendSampleBuffer(sampleBuffer);
          }
          break;

        default:
          break;
      }
    }, error => {
      if (!error) { return; }
      Ti.API.error(`Error recording screen: ${error.localizedDescription}`);
      this.callback(null, error.localizedDescription);
    });
  }

  stopCapture() {
    RPScreenRecorder.sharedRecorder().stopCaptureWithHandler(error => {
      if (!error && this.assetWriter !== AVFoundation.AVAssetWriterStatusFailed) {
        this.audioInput.markAsFinished();
        this.videoInput.markAsFinished();

        this.assetWriter.finishWritingWithCompletionHandler(() => {
          this.callback(Ti.Filesystem.getFile(this.filePath), null);
        });
      } else {
        Ti.API.error(`Error stopping capture: ${error.localizedDescription}`);
        this.callback(null, error.localizedDescription);
      }
    });
  }

  requestAudioPermissions(block) {
    AVCaptureDevice.requestAccessForMediaTypeCompletionHandler(AVFoundation.AVMediaTypeAudio, granted => {
      this.isMicEnabled = granted;
      block(granted);
    });
  }

  /* unused */
  requestVideoPermissions(block) {
    AVCaptureDevice.requestAccessForMediaTypeCompletionHandler(AVFoundation.AVMediaTypeVideo, granted => {
      block(granted);
    });
  }
}
```

## Limitations

Right now, the screen recorder only records the video input. For audio inputs, add the designated lines
to the `AVAssetWriter` instance.

## License

MIT

## Author

Hans Knöchel ([@hansemannnn](https://twitter.com/hansemannnn) / [Web](http://hans-knoechel.de))
