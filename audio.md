Introduction
============

Audio is one of the most complicated aspects of the Switch, with different interfaces each having multiple revisions over time. Supporting both audio in and audio out, as well as both a direct audio output (`audout`) and a rendered audio output (`audren`). Direct audio output is fairly straightforward and self-explanatory, though I may document it here in the future; audio rendering is a beast, which is what this document will be focusing on.

Open Questions
==============

- What determines the number of output channels?
	- How does surround-sound work? Can we go beyond 5.1?

(Unverified/unknown things are tagged with `(XXX)`)

Creation
========

`nn::audio::detail::IAudioRendererManager` (`audren:u`) is the entrypoint for audio rendering. IPC command 0 opens an audio renderer and accepts a transfer memory, the process handle for that memory, the size of the buffer, and an initialization struct:

```
type nn::audio::detail::AudioRendererParameterInternal = struct<0x34> {
  s32 SampleRate;
  s32 SampleCount;
  s32 MixBufferCount;
  s32 SubMixBufferCount;
  s32 VoiceCount;
  s32 SinkCount;
  s32 EffectCount;
  s32 PerformanceManagerCount;
  s32 VoiceDropEnable;
  s32 SplitterCount;
  s32 SplitterDestinationDataCount;
  s32 ExternalContextSize;
  s32 Revision;
};
```

Sample rate (Hz) is either 32000 or 48000. Sample count is how many samples are in each update. Mix buffer count is the maximum number of mixing buffers used during rendering. Sub mix buffer count is (XXX). Voice count is the maximum number of rendered voices. Sink count is the number of outputs (which can either be a device or a ring buffer). Effect count is the number of active (XXX) effects. Performance manager count is the maximum number of performance metric frames (per update? (XXX)). VoiceDropEnable is a boolean flag for whether voices can be dropped if needed for performance. Splitter count is (XXX). Splitter destination data count is (XXX). External context size is (XXX).

Revision is a fourcc that indicates the audio renderer revision, starting with `REV1` (`0x31564552`) with the most significant byte (the number in the fourcc) being increased by one for each subsequent revision. The current revision (XXX) is revision 15, or -- as you might guess -- `REV?` (`0x3f564552`). This is the decision of a madman and we should be concerned.

With Switch OS 3.0.0, a second command (3) was added, called `OpenAudioRendererForManualExecution`. This is virtually identical, but takes in a raw buffer address in the process rather than utilizing transfer memory.

Using the Audio Renderer
========================


