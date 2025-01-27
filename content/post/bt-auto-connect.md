---
title: "Creating Bluetooth sockets on Android without pairing"
date: 2018-03-26T02:41:36-07:00
blogpost: true
---

{{< rawhtml >}}
  <video autoplay playsinline loop muted class="playpause-with-visibility">
    <!--
      `ffmpeg -t 0:00:12 -i noise-demo-2018-03-24_15.55.55.mkv -r 10 noise-demo_%05d.png`
      Manually remove bad frames
      `ffmpeg -framerate 10 -pattern_type glob -i "noise-demo_*.png" -pix_fmt yuv420p -c:v libx264 -crf 32 -b:v 0 -an noise-demo-web.mp4`
    -->
    <source src="/post/bt-auto-connect/noise-demo-web.mp4">
    <!--No webm because it's always noticeably worse at this filesize?-->
  </video>
{{< /rawhtml >}}

I implemented a way to automatically create Bluetooth Classic sockets between two nearby Android devices running the same app. This method works continuously in the background without wakelocks, and it does not require pairing, root, or any user interaction.

<!--more-->

The video[^scrcpy] is of two physical phones, in this case my Moto G4 Play (left) and original Google Pixel (right), automatically connecting and syncing. The G4 is in airplane mode with Bluetooth on. These phones didn't start paired and never became paired. Neither phone is rooted.

[^scrcpy]: Recorded using [scrcpy][scrcpy-github], which records the screens of physical Android devices over adb.

[scrcpy-github]: https://github.com/Genymobile/scrcpy

## Motivation

A while ago, I was thinking about how fragile the infrastructure supporting long-distance wireless communication is:

* Cell phone towers and the servers implementing chat services are single points of failure.
* Natural disasters can easily wipe out infrastructure. In Puerto Rico, [it took months to restore cell phone service after Hurricane Maria][cellservice-maria].
* Cell phone service tends to be unreliable in sparsely populated areas.
* At concerts, conventions, protests, or similar situations, cell phone towers can be overloaded making it difficult to get packets through.
* In [China][censorship-china], [North Korea][censorship-nk], [Turkey][censorship-turkey], and anywhere else that freedom of speech is not respected, existing communications infrastructure is monitored and censored.

[cellservice-maria]: https://en.wikipedia.org/wiki/Hurricane_Maria#Puerto_Rico_3
[censorship-china]: https://en.wikipedia.org/wiki/Great_Firewall
[censorship-nk]: https://en.wikipedia.org/wiki/Human_rights_in_North_Korea#Civil_liberties
[censorship-turkey]: https://www.afp.com/en/news/826/turkey-gives-watchdog-power-block-internet-broadcasts-doc-12z0r61

With these concerns in mind, I began working on [Noise][noise-github], a peer-to-peer messaging app that works over long distances using [epidemic routing][epidemic-routing]. Epidemic routing works best when nearby devices are able to connect automatically, which I'm able to do by using a Bluetooth LE beacon to bootstrap a Bluetooth Classic socket. In Noise, Bluetooth connection management is handled entirely in the [`BluetoothSyncService` class][noise-bt-impl].

[noise-github]: https://github.com/aarmea/noise
[noise-bt-impl]: https://github.com/aarmea/noise/blob/8deb23b18b344e1392b08ae7c2db94b875e398e7/app/src/main/java/com/alternativeinfrastructures/noise/sync/bluetooth/BluetoothSyncService.java
[epidemic-routing]: http://issg.cs.duke.edu/epidemic/epidemic.pdf

## Discovery using Bluetooth LE

Like the name implies, Bluetooth Low Energy is a standard designed to reduce the power needed to transfer certain types of data. It does this using a broadcast model, where central devices listen for beacons from peripherals[^ble-gatt]. Android Lollipop and higher support BLE in both the [central][ble-central-android] and [peripheral][ble-peripheral-android] roles[^ble-tested]. By constantly advertising and scanning for a predefined beacon, an app can locate nearby running instances of itself.

[^ble-gatt]: BLE also provides a pair-free key-value store via GATT, but this does not scale to this application.

[^ble-tested]: I've tested that this works on my original Google Pixel, Moto G4 Play, and Sony Z5 Compact. Here's a [list][ble-beacon-devices] of other devices that should work.

[ble-central-android]: https://developer.android.com/guide/topics/connectivity/bluetooth-le.html
[ble-peripheral-android]: https://source.android.com/devices/bluetooth/ble_advertising
[ble-beacon-devices]: https://altbeacon.github.io/android-beacon-library/beacon-transmitter-devices.html

Because BLE advertisements are limited to 31 bytes, [Noise's beacon][noise-beacon-impl] consists of exactly one service UUID, which is 16 bytes long:

```nohighlight
5ac825f4-6084-42a6-0000-XXXXXXXXXXXX
[Noise UUID half ] [--] [MAC addr. ]
```
[noise-beacon-impl]: https://github.com/aarmea/noise/blob/8deb23b18b344e1392b08ae7c2db94b875e398e7/app/src/main/java/com/alternativeinfrastructures/noise/sync/bluetooth/BluetoothSyncService.java#L97

For the service UUID, I randomly generated a UUID and ignored its second half. The first half identifies that this advertisement is for Noise and the second half contains this device's Bluetooth MAC address. The 15 leftover bytes in the beacon plus the 2 unused bytes in the UUID leaves 17 bytes that can be used later, such as for a subset of the message vector used in epidemic routing.

Noise [retrieves `BluetoothLeAdvertiser` and `BluetoothLeScanner`][noise-btdevice-impl] from the `BluetoothAdapter` to simultaneously [start the beacon and listen for it][noise-discover-impl]. A crucial benefit of using BLE for discovery is that both advertising and scanning occur directly on the radio. To save power, [the phone can sleep while this happens][oreo-ble-sleep][^oreo-ble-sleep-impl] -- Android will wake the app if it finds a nearby device.

[^oreo-ble-sleep-impl]: I have not implemented this yet since I only own one phone that supports Oreo.

[noise-btdevice-impl]: https://github.com/aarmea/noise/blob/8deb23b18b344e1392b08ae7c2db94b875e398e7/app/src/main/java/com/alternativeinfrastructures/noise/sync/bluetooth/BluetoothSyncService.java#L366
[noise-discover-impl]: https://github.com/aarmea/noise/blob/8deb23b18b344e1392b08ae7c2db94b875e398e7/app/src/main/java/com/alternativeinfrastructures/noise/sync/bluetooth/BluetoothSyncService.java#L191
[oreo-ble-sleep]: http://www.davidgyoungtech.com/2017/08/07/beacon-detection-with-android-8

## Making the connection

Android provides [`listenUsingInsecureRfcommWithServiceRecord`][socket-listen-android] and [`createInsecureRfcommSocketToServiceRecord`][socket-connect-android] to create unauthenticated connections without pairing as long as both the server and client agree on a UUID. For simplicity, I also used the beacon's service UUID to facilitate the connection.

[socket-listen-android]: https://developer.android.com/reference/android/bluetooth/BluetoothAdapter.html#listenUsingInsecureRfcommWithServiceRecord(java.lang.String,%20java.util.UUID)
[socket-connect-android]: https://developer.android.com/reference/android/bluetooth/BluetoothDevice.html#createInsecureRfcommSocketToServiceRecord(java.util.UUID)

After setting up BLE, Noise [listens for Bluetooth Classic connections in a separate thread][noise-listen-impl] so it can continue in the background. The other ("client") end makes these connections [in the BLE scan callback][noise-connect-trigger] using the MAC address from service UUID rather than the default one from the beacon. This is needed because the address that Android automatically includes on the beacon is a temporary one that cannot be used to create a Bluetooth Classic socket.

[noise-listen-impl]: https://github.com/aarmea/noise/blob/8deb23b18b344e1392b08ae7c2db94b875e398e7/app/src/main/java/com/alternativeinfrastructures/noise/sync/bluetooth/BluetoothSyncService.java#L268
[noise-connect-trigger]: https://github.com/aarmea/noise/blob/8deb23b18b344e1392b08ae7c2db94b875e398e7/app/src/main/java/com/alternativeinfrastructures/noise/sync/bluetooth/BluetoothSyncService.java#L211

After these steps, each device has a socket connection to the other, and sync can proceed as defined by epidemic routing.

## MAC address whack-a-mole

Android provides a [`getAddress()`][btadapter-getaddress] on the `BluetoothAdapter` that originally retrieved the Bluetooth adapter's physical MAC address. However, Google has been making this more and more difficult:

[btadapter-getaddress]: https://developer.android.com/reference/android/bluetooth/BluetoothAdapter.html#getAddress()

* Pre-Marshmallow: `getAddress()` works fine
* Marshmallow up to Oreo: [`getAddress()` always returns `02:00:00:00:00:00`][marshmallow-block-mac]. On these versions, Noise is able to [use reflection to retrieve the address][noise-mac-reflection].
* Oreo and up: [using `getAddress()` requires the app to have the `LOCAL_MAC_ADDRESS` permission][oreo-block-mac], which is only granted to system apps or [via root][root-pm-grant]. Attempting to call it anyway will result in an uncatchable `SecurityException`. As a workaround, the MAC address is visible in Settings, so it can be manually pasted into the app's storage[^noise-oreo-mac].

[^noise-oreo-mac]: This wasn't implemented when I originally posted this, but the setting exists as of [this commit][noise-mac-setting].

[marshmallow-block-mac]: https://developer.android.com/about/versions/marshmallow/android-6.0-changes.html#behavior-hardware-id
[oreo-block-mac]: https://www.xda-developers.com/android-o-introduces-changes-and-improvements-to-device-identifiers/
[root-pm-grant]: https://github.com/aarmea/HandsfreeActions/blob/35b1e140098e2d5945042416a25bb0b590b2e468/HandsfreeActions/src/main/java/com/albertarmea/handsfreeactions/RemapperService.java#L73
[noise-mac-reflection]: https://github.com/aarmea/noise/blob/8deb23b18b344e1392b08ae7c2db94b875e398e7/app/src/main/java/com/alternativeinfrastructures/noise/sync/bluetooth/BluetoothSyncService.java#L130
[noise-mac-setting]: https://github.com/aarmea/noise/commit/17d37c380ec4f093821db421ab77d96fc5683667

I recognize that Google is trying to improve privacy with this move -- in advertising, the MAC address is used to track users between device resets. Additionally, because the MAC address is included in the beacon, if an attacker manages to correlate an address to a person, that attacker can then determine if that person is nearby.

Unfortunately, as far as I know this is the most accessible way to create sockets between two Android phones, so this hack will have to stay. Suggestions are welcome.

## Permissions

As expected, this requires the `BLUETOOTH` and `BLUETOOTH_ADMIN` permissions. On Marshmallow and up, starting a BLE scan requires the `ACCESS_COARSE_LOCATION` permission (presumably because an app can determine where it is if a unique beacon is nearby). [Noise's AndroidManifest.xml][noise-manifest] also has `LOCAL_MAC_ADDRESS` in the unlikely event that it is installed as a system app.

[noise-manifest]: https://github.com/aarmea/noise/blob/c15aa06b4a19cdc41b805c4b85b6c5a733bf9c2b/app/src/main/AndroidManifest.xml
