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

## Example

```js
import {
  RPScreenRecorder,
  ReplayKit
} from 'ReplayKit';

import {
  AVAssetWriter,
  AVFoundation
} from 'AVFoundation';

import { NSURL } from 'Foundation';

import { UIScreen } from 'UIKit';

import { CoreMedia } from 'CoreMedia'; 

const AVVideoCodecKey = AVFoundation.AVVideoCodecKey;
const AVVideoWidthKey = AVFoundation.AVVideoWidthKey;
const AVVideoHeightKey = AVFoundation.AVVideoHeightKey;

// Prepare asset writer
const fileURL = NSURL.fileURLWithPath('<your-file-path>');
const assetWriter = AVAssetWriter.alloc().initWithOutputURLFileType(fileURL, AVFoundation.AVFileTypeMPEG4);

const videoOutputSettings = {
  AVVideoCodecKey: AVFoundation.AVVideoCodecTypeH264,
  AVVideoWidthKey: UIScreen.mainScreen.bounds.size.width,
  AVVideoHeightKey: UIScreen.mainScreen.bounds.size.height
};
            
const videoInput  = AVAssetWriterInput.assetWriterInputWithMediaTypeOutputSettings(AVFoundation.AVMediaTypeVideo, videoOutputSettings);
videoInput.expectsMediaDataInRealTime = true;
assetWriter.add(videoInput);

// Start screen capture
RPScreenRecorder.shared().startCaptureWithHandlerCompletionHandler((sample, bufferType, error) => {
  if (!CoreMedia.CMSampleBufferDataIsReady(sample)) {  
    return;
  }

  if (assetWriter.status === AVFoundation.AVAssetWriterStatusUnknown) {
    assetWriter.startWriting();
    assetWriter.startSessionAtSourceTime(CoreMedia.CMSampleBufferGetPresentationTimeStamp(sample));
  }

  if (assetWriter.status === AVAssetWriterStatusFailed) {
    Ti.API.error(`Error recording screen: ${assetWriter.error}`);
    return;
  }

  if (bufferType === ReplayKit.RPSampleBufferTypeVideo) {
    if (videoInput.isReadyForMoreMediaData) {
      videoInput.append(sample)
    }
  }
}, error => {
  Ti.API.error(`Error recording screen: ${error.localizedDescription}`);
});

// Stop screen capture after 10s
setTimeout(() => {
  RPScreenRecorder.shared().stopCaptureWithHandler(error => {
    if (!error) {
      Ti.API.info('Success!');
    } else {
      Ti.API.error(`Error stopping capture: ${error.localizedDescription}`);
    }
  });
}, 10 * 1000);
```

## License

MIT

## Author

Hans Knöchel ([@hansemannnn](https://twitter.com/hansemannnn) / [Web](http://hans-knoechel.de))
