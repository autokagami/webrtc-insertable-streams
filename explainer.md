# Explainer - WebRTC Insertable Streams

## Problem to be solved

We need an API for processing media that:
* Allows the processing to be specified by the user, not the browser
* Allows the processed data to be handled by the browser as if it came through
  the normal pipeline
* Allows the use of techniques like WASM to achieve effective processing
* Allows the use of techniques like Workers to avoid blocking on the main thread
* Does not negatively impact security or privacy of current communications


## Approach

This document builds on concepts previously proposed by
[WebCodecs](https://github.com/pthatcherg/web-codecs/), and applies them to the existing
RTCPeerConnection API in order to build an API that is:

* Familiar to existing PeerConnection users
* Able to support insertion of user-defined components
* Able to support high performance user-specified transformations
* Able to support user defined component wrapping and replacement

The central idea is to expose components in an RTCPeerConnection as a collection of
streams (as defined by the [WHATWG Streams API](https://streams.spec.whatwg.org/)),
which can be manipulated to introduce new components, or to wrap or replace existing
components.


## Use cases

The first use case to be supported by the API is the processing of encoded media, with
end-to-end encryption intended as the motivating application. As such, the first version
of the API will focus on this use case. However, the same approach can be used in future
iterations to support additional use cases such as:

* Funny Hats (processing inserted before encoding or after decoding)
* Background removal
* Voice processing
* Dynamic control of codec parameters
* App-defined bandwidth distribution between tracks
* Custom codecs for special purposes (in combination with WebCodecs)

## Code Examples
0. Feature detection can be done as follows:

<pre>
const supportsInsertableStreams = window.RTCRtpSender &&
      !!RTCRtpSender.prototype.createEncodedStreams;
</pre>

1. Let an PeerConnection know that it should allow exposing the data flowing through it
as streams.

To ensure backwards compatibility, if the Insertable Streams API is not used, an
RTCPeerConnection should work exactly as it did before the introduction of this API.
Therefore, we explicitly let the RTCPeerConnection know that we want to use insertable
streams. For example:

<pre>
let pc = new RTCPeerConnection({
    encodedInsertableStreams: true,
});
</pre>

2. Set up transform streams that perform some processing on data.

The following example negates every bit in the original data payload
of an encoded frame and adds 4 bytes of padding.

<pre>
    let senderTransform = new TransformStream({
      start() {
        // Called on startup.
      },

      async transform(encodedFrame, controller) {
        let view = new DataView(encodedFrame.data);
        // Create a new buffer with 4 additional bytes.
        let newData = new ArrayBuffer(encodedFrame.data.byteLength + 4);
        let newView = new DataView(newData);

        // Fill the new buffer with a negated version of all
        // the bits in the original frame.
        for (let i = 0; i < encodedFrame.data.byteLength; ++i)
          newView.setInt8(i, ~view.getInt8(i));
        // Set the padding bytes to zero.
        for (let i = 0; i < 4; ++i)
          newView.setInt8(encodedFrame.data.byteLength + i, 0);

        // Replace the frame's data with the new buffer.
        encodedFrame.data = newData;

        // Send it to the output stream.
        controller.enqueue(encodedFrame);
      },

      flush() {
        // Called when the stream is about to be closed.
      }
    });
</pre>

3. Create a MediaStreamTrack, add it to the RTCPeerConnection and connect the
Transform stream to the track's sender.

<pre>
let stream = await navigator.mediaDevices.getUserMedia({video:true});
let [track] = stream.getTracks();
let videoSender = pc.addTrack(track, stream)
let senderStreams = videoSender.createEncodedStreams();

// Do ICE and offer/answer exchange.

senderStreams.readable
  .pipeThrough(senderTransform)
  .pipeTo(senderStreams.writable);
</pre>

4. Do the corresponding operations on the receiver side.

<pre>
let pc = new RTCPeerConnection({encodedInsertableStreams: true});
pc.ontrack = e => {
  let receiverTransform = new TransformStream({
    start() {},
    flush() {},
    async transform(encodedFrame, controller) {
      // Reconstruct the original frame.
      let view = new DataView(encodedFrame.data);

      // Ignore the last 4 bytes
      let newData = new ArrayBuffer(encodedFrame.data.byteLength - 4);
      let newView = new DataView(newData);

      // Negate all bits in the incoming frame, ignoring the
      // last 4 bytes
      for (let i = 0; i < encodedFrame.data.byteLength - 4; ++i)
        newView.setInt8(i, ~view.getInt8(i));

      encodedFrame.data = newData;
      controller.enqueue(encodedFrame);
      },
    });

  let receiverStreams = e.receiver.createEncodedStreams();
  receiverStreams.readable
    .pipeThrough(receiverTransform)
    .pipeTo(receiverStreams.writable);
}
</pre>

## API

The following are the IDL modifications proposed by this API.
Future iterations may add additional operations following a similar pattern.

<pre>
// New dictionary.
dictionary RTCInsertableStreams {
    ReadableStream readable;
    WritableStream writable;
};

// New enum for video frame types. Will eventually re-use the equivalent defined
// by WebCodecs.
enum RTCEncodedVideoFrameType {
    "empty",
    "key",
    "delta",
};

// New dictionaries for video and audio metadata.
dictionary RTCEncodedVideoFrameMetadata {
    long long frameId;
    sequence&lt;long long&gt; dependencies;
    unsigned short width;
    unsigned short height;
    long spatialIndex;
    long temporalIndex;
    long synchronizationSource;
    sequence&lt;long&gt; contributingSources;
};

dictionary RTCEncodedAudioFrameMetadata {
    long synchronizationSource;
    sequence&lt;long&gt; contributingSources;
};

// New interfaces to define encoded video and audio frames. Will eventually
// re-use or extend the equivalent defined in WebCodecs.
// The additionalData fields contain metadata about the frame and will
// eventually be exposed differently.
interface RTCEncodedVideoFrame {
    readonly attribute RTCEncodedVideoFrameType type;
    readonly attribute unsigned long long timestamp;
    attribute ArrayBuffer data;
    RTCVideoFrameMetadata getMetadata();
};

interface RTCEncodedAudioFrame {
    readonly attribute unsigned long long timestamp;
    attribute ArrayBuffer data;
    RTCAudioFrameMetadata getMetadata();
};

// New field in RTCConfiguration
partial dictionary RTCConfiguration {
    boolean encodedInsertableStreams = false;
};

// New methods for RTCRtpSender and RTCRtpReceiver
partial interface RTCRtpSender {
    RTCInsertableStreams createEncodedStreams();
};

partial interface RTCRtpReceiver {
    RTCInsertableStreams createEncodedStreams();
};

</pre>

## Design considerations ##

This design is built upon the Streams API. This is a natural interface
for stuff that can be considered a "sequence of objects", and has an ecosystem
around it that allows some concerns to be handed off easily.

In particular:

* Sequencing comes naturally; streams are in-order entities.
* With the Transferable Streams paradigm, changing what thread is doing
  the processing can be done in a manner that has been tested by others.
* Since other users of Streams interfaces are going to deal with issues
  like efficient handover and WASM interaction, we can expect to leverage
  common solutions for these problems.

There are some challenges with the Streams interface:

* Queueing in response to backpressure isn't an appropriate reaction in a
  real-time environment. This can be mitigated at the sender by not queueing,
  preferring to discard frames or not generating them.
* How to interface to congestion control signals, which travel in the
  opposite direction from the streams flow.
* How to integrate error signalling and recovery, given that most of the
  time, breaking the pipeline is not an appropriate action.
  
These things may be solved by use of non-data "frames" (in the forward direction),
by reverse streams of non-data "frames" (in the reverse direction), or by defining
new interfaces based on events, promises or callbacks.

Experimentation with the prototype API seems to show that performance is
adequate for real-time processing; the streaming part is not contributing
very much to slowing down the pipelines.

## Alternatives to Streams ##
One set of alternatives involve callback-based or event-based interfaces; those
would require developing new interfaces that allow the relevant WebRTC
objects to be visible in the worker context in order to do processing off
the main thread. This would seem to be a significantly bigger specification
and implementation effort.

Another path would involve specifying a worklet API, similar to the AudioWorklet,
and specifying new APIs for connecting encoders and decoders to such worklets.
This also seemed to involve a significantly larger set of new interfaces, with a
correspondingly larger implementation effort, and would offer less flexibility
in how the processing elements could be implemented.





