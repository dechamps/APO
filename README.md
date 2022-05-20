# APO

Some random notes about [Windows Audio Process Objects][apo] (APOs) by
[Etienne Dechamps][].

## Manipulating APO configuration

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

Under each endpoint device key, the `FxProperties` key contains APO
configuration.

For example, the following key contains APO configuration for the
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

## Useful links

- [Audio Processing Object Architecture][apo]
- [Equalizer APO][], notably the [developer documentation][eapodev]
- [Audio Endpoint Devices][endpoint]
- [Custom Audio Effects in Windows Vista][vista] (notably explains the meaning
  of the deprecated LFX and GFX APO placements)
- [Enumerate APOs][enum]

[apo]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/audio-processing-object-architecture
[eapodev]: https://sourceforge.net/p/equalizerapo/wiki/Developer%20documentation/
[endpoint]: https://docs.microsoft.com/en-us/windows/win32/coreaudio/audio-endpoint-devices
[endpoint ID string]: https://docs.microsoft.com/en-us/windows/win32/coreaudio/endpoint-id-strings
[enum]: https://matthewvaneerde.wordpress.com/2010/06/03/how-to-enumerate-wasapi-audio-processing-objects-apos-on-your-system/
[EnumAudioEndpoints]: https://docs.microsoft.com/en-us/windows/win32/api/mmdeviceapi/nf-mmdeviceapi-immdeviceenumerator-enumaudioendpoints
[Equalizer APO]: https://sourceforge.net/projects/equalizerapo/
[Etienne Dechamps]: mailto:etienne@edechamps.fr
[inf]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/implementing-audio-processing-objects#registering-apos-for-processing-modes-and-effects-in-the-inf-file
[vista]: https://download.microsoft.com/download/9/c/5/9c5b2167-8017-4bae-9fde-d599bac8184a/sysfx.doc
