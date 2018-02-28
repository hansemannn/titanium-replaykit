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

import { NSURL } from 'Foundation';

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
    // Prepare asset writer
    const fileURL = NSURL.fileURLWithPath(this.filePath);
    const assetWriter = AVAssetWriter.assetWriterWithURLFileTypeError(fileURL, AVFoundation.AVFileTypeMPEG4, null); // TODO: Handle error?

    const videoOutputSettings = {
      AVVideoCodecKey: AVFoundation.AVVideoCodecTypeH264,
      AVVideoWidthKey: UIScreen.mainScreen.bounds.size.width,
      AVVideoHeightKey: UIScreen.mainScreen.bounds.size.height
    };

    const videoInput  = AVAssetWriterInput.assetWriterInputWithMediaTypeOutputSettings(AVFoundation.AVMediaTypeVideo, videoOutputSettings);
    videoInput.expectsMediaDataInRealTime = true;
    assetWriter.addInput(videoInput);

    // Start screen capture
    RPScreenRecorder.sharedRecorder().startCaptureWithHandlerCompletionHandler((sample, bufferType, error) => {
      if (!CoreMedia.CMSampleBufferDataIsReady(sample)) {
        return;
      }

      if (assetWriter.status === AVFoundation.AVAssetWriterStatusUnknown) {
        assetWriter.startWriting();
        assetWriter.startSessionAtSourceTime(CoreMedia.CMSampleBufferGetPresentationTimeStamp(sample));
      }

      if (assetWriter.status === AVAssetWriterStatusFailed) {
        Ti.API.error(`Error handling writer: ${assetWriter.error.localizedDescription}`);
        this.callback(null, assetWriter.error.localizedDescription);
        return;
      }

      if (bufferType === ReplayKit.RPSampleBufferTypeVideo) {
        if (videoInput.isReadyForMoreMediaData) {
          videoInput.append(sample)
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
      if (!error) {
        this.callback(Ti.Filesystem.getFile(this.filePath), null);
        Ti.API.info('Success!');
      } else {
        Ti.API.error(`Error stopping capture: ${error}`);
        this.callback(null, error.localizedDescription);
      }
    });
  }
}
```

## License

MIT

## Author

Hans Knöchel ([@hansemannnn](https://twitter.com/hansemannnn) / [Web](http://hans-knoechel.de))
