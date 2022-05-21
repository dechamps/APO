# APO

Some random notes about [Windows Audio Processing Objects][apo] (APOs) by
[Etienne Dechamps][].

## What is an APO?

APO stands for "Audio Processing Object". It is an API and framework designed by
Microsoft for building pluggable audio filters ([DSP][]). It is quite similar to
[VST][] in principle.

More technically, an APO takes the form of a [COM][] class that implements the
[APO interfaces][apodesign]. The resulting class is typically provided as part
of a [DLL][] which is then registered in the system-wide COM registry (e.g.
using [regsvr32][]).

The class can then be instantiated, and each instance can [process][] a
continuous audio stream, making arbitrary changes to the audio data.

### The Windows Audio Engine

What differentiates APOs from other audio filtering frameworks (such as VST) is
that it is the filtering framework used by the [Windows Audio Engine][arch].

The Windows Audio Engine is a core component of the Windows audio stack. Its
role is to bridge the gap between individual application audio streams and
hardware audio devices. As such it handles various tasks such as mixing audio
from multiple applications, automatic format conversions, etc.

Most application audio goes through the Windows Audio Engine. In particular it
will process any audio stream that is opened through the [WASAPI][] (Shared)
API (and by extension [DirectSound][] and [MME][], which use WASAPI Shared
internally). The only exceptions are streams opened in [WASAPI Exclusive][]
mode, [Kernel Streaming][] (WDM-KS), and native [ASIO][], which all bypass the
Windows Audio Engine; but these are seldom used by typical applications.

The Windows Audio Engine [uses a number of APOs internally][enum] as part of its
normal operation, each handling a specific task such as automatic sample rate
conversion. These internal APOs notably include the (in)famous
[CAudioLimiter][asrdebate].

APOs run inside the Windows audio graph process (`audiodg.exe`) which is itself
managed by the Windows Audio service (`audiosrv`).

### System effect APOs (sAPOs)

APOs can be installed as *system effect APOs*, or sAPOs, by implementing the
[`IAudioSystemEffects`][] interface. sAPOs are optional APOs that can be
inserted at various points inside the Windows audio engine pipeline. sAPOs can
be extremely useful because they can be used to arbitrarily filter audio for
all Windows applications running system-wide; and because they run directly
inside the Windows audio engine, they can do so in a very clean and efficient
manner with zero additional latency.

sAPOs can be [positioned][apoarch] to filter audio data at the following stages
of the Windows audio pipeline:

- **Stream Effect (SFX)** APOs are instantiated for every application stream.
  They process the audio signal as it comes in and out of a single application.
  - In particular, in the playback direction, processing occurs before any
    mixing or sample rate conversion takes place.
  - SFX APOs are the only sAPOs that are allowed to change the channel count
    (i.e. downmix/upmix).
- **Mode Effect (MFX)** APOs operate at an intermediate stage, on the mixed
  audio from all streams that share a common [mode][].
- **Endpoint Effect (EFX)** APOs operate on the audio signal that comes in and
  out of the audio device.
  - In particular, in the playback direction, processing occurs after all mixing
    and sample rate conversion takes place (but *before* CAudioLimiter).

The above describes the modern sAPO architecture as it was introduced in Windows
8.1. Previously, different kinds of APOs were used:

- **Local Effect (LFX)** APOs serve the same purpose as SFX APOs.
- **Global Effect (GFX)** APOs serve the same purpose as EFX APOs.

LFX and GFX APOs can still be used in modern versions of Windows, but Microsoft
does not document them anymore; they should be considered deprecated. If both
modern (SFX/MFX/EFX) and legacy (LFX/GFX) sAPOs are configured, then Windows
will use the modern ones.

### System effect APO installation

sAPOs are configured on a per-[audio endpoint device][endpoint] basis. That is,
each audio device uses its own set of APOs.

Which APOs to use is normally decided by the audio device driver; specifically,
they are defined in the [audio driver INF file][inf]. 

An audio device driver can decide to use some of the [sAPOs that are bundled
with Windows][bundled], or it can implement its own [custom sAPO][apodesign] and
include it in the driver package. Keep in mind, however, that APOs always run in
user mode (in the `audiodg.exe` process), not in kernel mode; they are merely
*bundled* with the driver, and are not part of the kernel-mode driver module
itself.

When an audio driver is installed, any custom sAPOs are installed and
registered, and the sAPO configuration from its INF file is copied to the
Windows Audio Engine configuration (see below).

After the audio driver is installed, it is technically possible to go in and
change the Windows Audio Engine configuration after the fact to use different
sAPOs, even third-party sAPOs that are not part of Windows nor the original
driver. This is extremely powerful because that makes it possible to apply any
arbitrary audio filtering to any audio device, independent of driver (think
"system-wide VSTs"). This is precisely how [Equalizer APO][] works. Note,
however, that this approach to setting up sAPOs *is not officially supported by
Microsoft*; the fact that it works should be considered an "happy accident".
Also note that such custom configuration will be overwritten every time the
audio driver is reinstalled, and that some Windows updates have a tendency to
reinstall audio drivers.

## Manipulating system effect APO configuration

All commands shown in this section are Powershell commands. Commands that make
changes will only work when run as Administrator.

### Location

The configuration for the Windows Audio Engine lives under the following
Windows registry key:

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\MMDevices\Audio
```

Playback devices are found under the `Render` key. Recording devices are found
under the `Capture` key.

Under `Render` and `Capture`, each [audio endpoint device][endpoint] has its own
key, named after the endpoint GUID (see below).

Under each endpoint device key, the `FxProperties` key contains system effect
APO configuration.

For example, the following key contains system effect APO configuration for the
playback endpoint device with GUID `{b39fc22d-4c5d-4e65-8276-db7f999d2d06}`:

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\MMDevices\Audio\Render\{b39fc22d-4c5d-4e65-8276-db7f999d2d06}\FxProperties
```

The initial configuration comes from the driver store, which is itself populated
[from the audio device driver INF file][inf].

### Determining the GUID of an audio endpoint device

One way is to determine the GUID of an endpoint device is to open the [Equalizer
APO][] Configurator, select a device, and click on "Copy Device command to
clipboard". The copied command includes the endpoint GUID.

Otherwise, you can use the following command:

```powershell
Get-PnpDevice -Class AudioEndpoint | Format-Table -Property InstanceId,Present,FriendlyName
```

Example output:

```
FriendlyName                                                  Present InstanceId
------------                                                  ------- ----------
Headset (CORSAIR VIRTUOSO Wireless Gaming Headset)              False SWD\MMDEVAPI\{0.0.0.00000000}.{A8079308-1956-4FB7-BBB0-8D8842611102}
Headset Microphone (CORSAIR VIRTUOSO Wireless Gaming Headset)   False SWD\MMDEVAPI\{0.0.1.00000000}.{4ED8903D-2C88-4EAC-93E4-39B0BFE7475C}
Monitors (Xonar U7)                                              True SWD\MMDEVAPI\{0.0.0.00000000}.{5D16E8DA-37A3-4657-8D52-57FFE97F4EF9}
S/PDIF 1 (Virtual Audio Cable)                                  False SWD\MMDEVAPI\{0.0.1.00000000}.{42BA34C2-E6E7-452D-BCE3-ADCBEC30E1A4}
Line 1 (Virtual Audio Cable)                                    False SWD\MMDEVAPI\{0.0.0.00000000}.{185D6665-3115-45BD-9E85-39BA4E326DDD}
```

The `InstanceId` is normally `SWD\MMDEVAPI\` followed by the
*[endpoint ID string][]*. The endpoint ID string is itself composed of a
`{0.0.0.00000000}.` or  `{0.0.1.00000000}.` prefix (denoting Render and Capture
endpoints, respectively), followed by the endpoint GUID. So in the above
example, `Monitors` is a Render endpoint device and its GUID is
`{5D16E8DA-37A3-4657-8D52-57FFE97F4EF9}`.

Endpoint GUIDs will change if the audio driver is reinstalled. Note that some
Windows updates reinstall all audio drivers as a side effect.

In code, the proper way to enumerate audio endpoints is to use
[`IMMDeviceEnumerator::EnumAudioEndpoints()`][EnumAudioEndpoints].

### `FxProperties` values

The `FxProperties` registry key holds the system effect APO configuration for a
specific audio endpoint device.

The configuration is organized as a set of registry values, which correspond
to individual properties. A property is identified by a GUID, followed by a
comma `,`, followed by a number. For example:
`{b725f130-47ef-101a-a5f1-02608c9eebac},10`.

The official list of supported system effect APO properties can be found in the
[audio driver INF settings][infsettings] documentation (look for properties
that are installed under `HKR,FX\0` in the INF File Sample).  Alternatively,
the `audioenginebaseapo.idl` file in the [Windows SDK][] contains a more
exhaustive list (e.g. it includes legacy LFX/GFX properties).

The most important properties are those that control which sAPOs are used. These
are:

| Type | Property ID                                | Property name                     |
|------|--------------------------------------------|-----------------------------------|
| SFX  | `{d04e05a6-594b-4fb6-a80d-01af5eed7d1d},5` | [`PKEY_FX_StreamEffectClsid`][]   |
| MFX  | `{d04e05a6-594b-4fb6-a80d-01af5eed7d1d},6` | [`PKEY_FX_ModeEffectClsid`][]     |
| EFX  | `{d04e05a6-594b-4fb6-a80d-01af5eed7d1d},7` | [`PKEY_FX_EndpointEffectClsid`][] |
| LFX  | `{d04e05a6-594b-4fb6-a80d-01af5eed7d1d},1` | `PKEY_FX_PreMixEffectClsid`       |
| GFX  | `{d04e05a6-594b-4fb6-a80d-01af5eed7d1d},2` | `PKEY_FX_PostMixEffectClsid`      |

These are string properties whose value is the [CLSID][] of the COM class that
implements the APO.

If any of SFX, MFX or EFX are present, then LFX and GFX are ignored.

All properties are optional. If no properties are present, or if the
`FxProperties` key is absent entirely, then no system effect APOs are used.

Another notable property is [`PKEY_AudioEndpoint_Disable_SysFx`][]
(`{1da5d803-d492-4edd-8c23-e0c0ffee7f0e},5`, DWORD), which, if set to 1,
disables all sAPOs. It is mapped to the "Enable audio enhancements" checkbox in
the Windows audio device settings.

## Useful links

- [Windows Audio Architecture][arch]
- [Audio Processing Object Architecture][apo]
- [Audio INF File Settings][infsettings]
- [Equalizer APO][], notably the [developer documentation][eapodev]
- [Audio Endpoint Devices][endpoint]
- [Custom Audio Effects in Windows Vista][vista] (notably explains the meaning
  of the deprecated LFX and GFX APO placements)
- [Enumerate APOs][enum]

[apo]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/audio-processing-object-architecture
[apoarch]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/audio-processing-object-architecture#audio-processing-objects-architecture
[apodesign]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/implementing-audio-processing-objects#design-considerations-for-custom-apo-development
[ASIO]: https://en.wikipedia.org/wiki/Audio_Stream_Input/Output
[asrdebate]: https://www.audiosciencereview.com/forum/index.php?threads/ending-the-windows-audio-quality-debate.19438/
[bundled]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/audio-signal-processing-modes#audio-effects
[CLSID]: https://docs.microsoft.com/en-us/windows/win32/com/com-class-objects-and-clsids
[COM]: https://en.wikipedia.org/wiki/Component_Object_Model
[DLL]: https://en.wikipedia.org/wiki/Dynamic-link_library
[eapodev]: https://sourceforge.net/p/equalizerapo/wiki/Developer%20documentation/
[endpoint]: https://docs.microsoft.com/en-us/windows/win32/coreaudio/audio-endpoint-devices
[endpoint ID string]: https://docs.microsoft.com/en-us/windows/win32/coreaudio/endpoint-id-strings
[enum]: https://matthewvaneerde.wordpress.com/2010/06/03/how-to-enumerate-wasapi-audio-processing-objects-apos-on-your-system/
[EnumAudioEndpoints]: https://docs.microsoft.com/en-us/windows/win32/api/mmdeviceapi/nf-mmdeviceapi-immdeviceenumerator-enumaudioendpoints
[Equalizer APO]: https://sourceforge.net/projects/equalizerapo/
[Etienne Dechamps]: mailto:etienne@edechamps.fr
[DirectSound]: https://en.wikipedia.org/wiki/DirectSound
[DSP]: https://en.wikipedia.org/wiki/Digital_signal_processing
[`IAudioSystemEffects`]: https://docs.microsoft.com/en-us/windows/win32/api/audioenginebaseapo/nn-audioenginebaseapo-iaudiosystemeffects
[inf]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/implementing-audio-processing-objects#registering-apos-for-processing-modes-and-effects-in-the-inf-file
[infsettings]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/media-class-inf-extensions
[Kernel Streaming]: https://docs.microsoft.com/en-us/windows-hardware/drivers/stream/kernel-streaming
[MME]: https://en.wikipedia.org/wiki/Windows_legacy_audio_components#Multimedia_Extensions_(MME)
[mode]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/audio-signal-processing-modes
[`PKEY_AudioEndpoint_Disable_SysFx`]: https://docs.microsoft.com/en-us/windows/win32/coreaudio/pkey-audioendpoint-disable-sysfx
[`PKEY_FX_EndpointEffectClsid`]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/pkey-fx-endpointeffectclsid
[`PKEY_FX_ModeEffectClsid`]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/pkey-fx-modeeffectclsid
[`PKEY_FX_StreamEffectClsid`]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/pkey-fx-streameffectclsid
[process]: https://docs.microsoft.com/en-us/windows/win32/api/audioenginebaseapo/nf-audioenginebaseapo-iaudioprocessingobjectrt-apoprocess
[regsvr32]: https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/regsvr32
[vista]: https://download.microsoft.com/download/9/c/5/9c5b2167-8017-4bae-9fde-d599bac8184a/sysfx.doc
[VST]: https://en.wikipedia.org/wiki/Virtual_Studio_Technology
[arch]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/windows-audio-architecture
[WASAPI]: https://docs.microsoft.com/en-us/windows/desktop/coreaudio/wasapi
[WASAPI Exclusive]: https://docs.microsoft.com/en-us/windows/win32/coreaudio/exclusive-mode-streams
[Windows Property System]: https://docs.microsoft.com/en-us/windows/win32/properties/windows-properties-system
[Windows SDK]: https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/
