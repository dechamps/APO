# APO

Some random notes about [Windows Audio Process Objects][apo] (APOs) by
[Etienne Dechamps][].

## Manipulating APO configuration

### Location

The configuration for the Windows Audio Engine lives under the following
Windows registry key:

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\MMDevices\Audio
```

Playback devices are found under the `Render` key. Recording devices are found
under the `Capture` key.

Under `Render` and `Capture`, each [endpoint device][endpoint] has its own key,
named after the device GUID (see below).

Under each device key, the `FxProperties` key contains APO configuration.

For example, the following key contains APO configuration for the
`{b39fc22d-4c5d-4e65-8276-db7f999d2d06}` playback device:

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\MMDevices\Audio\Render\{b39fc22d-4c5d-4e65-8276-db7f999d2d06}\FxProperties
```

The initial configuration comes from the driver store, which is itself populated
[from the audio device driver INF file][inf].

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
[enum]: https://matthewvaneerde.wordpress.com/2010/06/03/how-to-enumerate-wasapi-audio-processing-objects-apos-on-your-system/
[Equalizer APO]: https://sourceforge.net/projects/equalizerapo/
[Etienne Dechamps]: mailto:etienne@edechamps.fr
[inf]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/implementing-audio-processing-objects#registering-apos-for-processing-modes-and-effects-in-the-inf-file
[vista]: https://download.microsoft.com/download/9/c/5/9c5b2167-8017-4bae-9fde-d599bac8184a/sysfx.doc
