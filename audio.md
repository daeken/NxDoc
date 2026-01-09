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

`nn::audio::detail::IAudioRendererManager` (`audren:u`) is the entrypoint for audio rendering.

```
interface nn::audio::detail::IAudioRendererManager is audren:u {
	[0] OpenAudioRenderer(nn::audio::detail::AudioRendererParameterInternal parameter, u64 workBufferSize, nn::applet::AppletResourceUserId, pid, handle<copy> workBufferTransferMemory, handle<copy> process) -> object<nn::audio::detail::IAudioRenderer>;
	[1] GetWorkBufferSize(nn::audio::detail::AudioRendererParameterInternal parameter) -> u64;
	[2] GetAudioDeviceService(nn::applet::AppletResourceUserId) -> object<nn::audio::detail::IAudioDevice>;
	@version(3.0.0+)
	[3] OpenAudioRendererAuto(nn::audio::detail::AudioRendererParameterInternal parameter, u64 workBufferAddr, u64 workBufferSize, nn::applet::AppletResourceUserId, pid, handle<copy> process) -> object<nn::audio::detail::IAudioRenderer>;
	@version(4.0.0+)
	[4] GetAudioDeviceServiceWithRevisionInfo(nn::applet::AppletResourceUserId, u32) -> object<nn::audio::detail::IAudioDevice>;
}

type nn::audio::detail::AudioRendererRenderingDevice = enum<u8> {
	Dsp = 0;
	Cpu = 1;
};

type nn::audio::detail::AudioRendererExecutionMode = enum<u8> {
	Auto = 0;
	Manual = 1;
};

type nn::audio::detail::AudioRendererParameterInternal = struct<0x34> {
  s32 SampleRate;
  s32 SampleCount;
  s32 MixBufferCount;
  s32 SubMixBufferCount;
  s32 VoiceCount;
  s32 SinkCount;
  s32 EffectCount;
  s32 PerformanceManagerCount;
  u8 VoiceDropEnable;
  u8 Reserved;
  nn::audio::detail::AudioRendererRenderingDevice RenderingDevice;
  nn::audio::detail::AudioRendererExecutionMode ExecutionMode;
  s32 SplitterCount;
  s32 SplitterDestinationDataCount;
  s32 ExternalContextSize;
  s32 Revision;
};
```

`OpenAudioRenderer` and `OpenAudioRendererAuto` are endpoints for creation. The naming is a bit misleading, as the primary difference between the two is that the latter takes an address for a work buffer, rather than transfer memory. The most important parameter is the `AudioRendererParameterInternal`:

Sample rate (Hz) is either 32000 or 48000. Sample count is how many samples are in each update. Mix buffer count is the maximum number of mixing buffers used during rendering. Sub mix buffer count is (XXX). Voice count is the maximum number of rendered voices. Sink count is the number of outputs (which can either be a device or a ring buffer). Effect count is the number of active (XXX) effects. Performance manager count is the maximum number of performance metric frames (per update? (XXX)). VoiceDropEnable is a boolean flag for whether voices can be dropped if needed for performance. Rendering device indicates whether the rendering is performed on DSP or CPU. Execution mode indicates whether operating in automatic or manual mode (however, Ryujinx comments appear to say that this must be set to Auto no matter what). Splitter count is (XXX). Splitter destination data count is (XXX). External context size is (XXX).

Revision is a fourcc that indicates the audio renderer revision, starting with `REV1` (`0x31564552`) with the most significant byte (the number in the fourcc) being increased by one for each subsequent revision. The current revision (XXX) is revision 15, or -- as you might guess -- `REV?` (`0x3f564552`). This is the decision of a madman and we should be concerned.

Using the Audio Renderer
========================


