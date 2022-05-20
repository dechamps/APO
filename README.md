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

## Useful links

- [Audio Processing Object Architecture][apo]
- [Equalizer APO][], notably the [developer documentation][eapodev]
- [Audio Endpoint Devices][endpoint]

[apo]: https://docs.microsoft.com/en-us/windows-hardware/drivers/audio/audio-processing-object-architecture
[eapodev]: https://sourceforge.net/p/equalizerapo/wiki/Developer%20documentation/
[endpoint]: https://docs.microsoft.com/en-us/windows/win32/coreaudio/audio-endpoint-devices
[Equalizer APO]: https://sourceforge.net/projects/equalizerapo/
[Etienne Dechamps]: mailto:etienne@edechamps.fr
