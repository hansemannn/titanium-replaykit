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

```js
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

## Example

```js
import {
  RPScreenRecorder,
  ReplayKit
} from 'ReplayKit';

import {
  AVAssetWriter,
  AVAssetWriterInput,
  AVFoundation
} from 'AVFoundation';

import { NSURL, NSError } from 'Foundation';

import { UIScreen } from 'UIKit';

import { CoreMedia } from 'CoreMedia';

const AVVideoCodecKey = AVFoundation.AVVideoCodecKey;
const AVVideoWidthKey = AVFoundation.AVVideoWidthKey;
const AVVideoHeightKey = AVFoundation.AVVideoHeightKey;

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
  }
  startCapture() {
    if (!RPScreenRecorder.sharedRecorder().isAvailable()) {
      callback(null, 'Screen Recorder is not available!');
      return;
    }

    // Prepare asset writer
    const fileURL = NSURL.fileURLWithPath(this.filePath);
    this.assetWriter = AVAssetWriter.assetWriterWithURLFileTypeError(fileURL, AVFoundation.AVFileTypeMPEG4, null); // TODO: Handle error?

    const videoOutputSettings = {
      AVVideoCodecKey: AVFoundation.AVVideoCodecTypeH264,
      AVVideoWidthKey: UIScreen.mainScreen.bounds.size.width,
      AVVideoHeightKey: UIScreen.mainScreen.bounds.size.height
    };

    this.videoInput  = AVAssetWriterInput.assetWriterInputWithMediaTypeOutputSettings(AVFoundation.AVMediaTypeVideo, videoOutputSettings);
    this.videoInput.expectsMediaDataInRealTime = true;
    this.videoInput.mediaTimeScale = 60

    this.assetWriter.addInput(this.videoInput);
    this.assetWriter.movieTimeScale = 60

    // Start screen capture
    RPScreenRecorder.sharedRecorder().startCaptureWithHandlerCompletionHandler((sample, bufferType, error) => {
      if (!CoreMedia.CMSampleBufferDataIsReady(sample)) {
        return;
      }

      if (this.assetWriter.status === AVFoundation.AVAssetWriterStatusUnknown) {
        this.assetWriter.startWriting();
        this.assetWriter.startSessionAtSourceTime(CoreMedia.CMSampleBufferGetPresentationTimeStamp(sample));
      }

      if (this.assetWriter.status === AVAssetWriterStatusFailed) {
        Ti.API.error(`Error handling writer: ${this.assetWriter.error.localizedDescription}`);
        this.callback(null, assetWriter.error.localizedDescription);
        return;
      }

      if (bufferType === ReplayKit.RPSampleBufferTypeVideo) {
        if (this.videoInput.isReadyForMoreMediaData()) {
          this.videoInput.append(sample)
        }
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
}
```

## Limitations

Right now, the screen recorder only records the video input. For audio inputs, add the designated lines
to the `AVAssetWriter` instance.

## License

MIT

## Author

Hans Knöchel ([@hansemannnn](https://twitter.com/hansemannnn) / [Web](http://hans-knoechel.de))
