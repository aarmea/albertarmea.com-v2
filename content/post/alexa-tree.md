---
title: "Making animatronics useful with Alexa"
date: 2018-01-17T00:02:13Z
---

{{< youtube id="Yi8ShLDlquk" >}}

I was looking for a white elephant gift, and the silliness of this animatronic
Christmas tree was too hard to pass up. The humor wore off after the third time
I heard it sing, so I decided to make it useful by adding Alexa.

<!--more-->

For context, the tree originally played a song, and motors in its mouth and body
made it sing and dance to the music. This is all the tree was designed to do:

<!-- ffmpeg -ss 00:00:01 -i VID_20171214_131516_2.mp4 -s 1280x720 -vframes 1 -q:v 5 VID_20171214_131516_2-web.jpg -->
<video controls poster="/post/alexa-tree/original.jpg">
  <!-- ffmpeg -ss 00:00:01 -i VID_20171214_131516_2.mp4 -s 1280x720 -c:v libvpx-vp9 -crf 42 -b:v 0 VID_20171214_131516_2-web.webm -->
  <source src="/post/alexa-tree/original.webm" type="video/webm">
  <!-- ffmpeg -ss 00:00:01 -i VID_20171214_131516_2.mp4 -s 1280x720 -c:v libx264 -crf 26 -b:v 0 VID_20171214_131516_2-web.mp4 -->
  <source src="/post/alexa-tree/original.mp4" type="video/mp4">
</video>

The first step was to take the tree apart to see what components can be reused.
Fortunately, this was relatively easy -- aside from the short section of fabric
I had to cut and two small blobs of hot glue used for strain relief, all of the
components were held in place with screws.

[See here for a full teardown.](/aside/alexa-tree-teardown)

# Building new hardware

Once I had the tree's electronics enclosure out, I could start picking parts and
building hardware to replace its original circuits.

<!-- convert -strip -resize 1600 -interlace Plane -quality 70 IMG_20171214_135942.jpg IMG_20171214_135942-web.jpg -->
![Tree's electronics assembly after removal](/post/alexa-tree/assembly.jpg)

## Choosing parts

I had two main constraints for parts: they had to fit inside the enclosure for
neatness, and I needed to be able to buy the parts in person to finish it on
time for Christmas. While this meant that all of the Raspberry Pi HATs were out
of the question, [Vetco Electronics](https://vetco.net/) has an excellent
selection of more generic parts near Seattle.

Here's what I ended up using:

* [Raspberry Pi Zero W][pi-zero][^pi-zero-buy] for wakeword detection and to
  communicate with the [Alexa Voice Service][avs]. The enclosure is too small to
  fit a [gutted Echo Dot][echo-dot] or a full-sized Pi.
* [Sabrent USB audio adapter][audio-adapter] for audio I/O because no Raspberry
  Pi has a built-in input. I had a few of these lying around. They're fairly
  compact and work well, so it seemed like a natural choice.
* [Micro-USB OTG adapter][usb-otg] to plug in the audio adapter since the Pi
  Zero only has micro-USB ports for compactness.
* [LM386-based amplifier circuit][amp-circuit] to drive the existing speaker.
  While it doesn't provide the best audio quality, it's good enough for audible
  speech.
* [Electret microphone][microphone][^vetco-microphone] to listen for commands.
  The datasheet suggests a driver circuit to provide a bias voltage for power,
  but I ended up not needing it because the audio adapter includes one.
* 2x [BA6286 motor controller][motor-controller] to drive the mouth and dancing
  motors, since the Pi's GPIO pins aren't rated for this kind of current draw.
* USB power supply rated for at least 2A, like [this wall adapter][power-supply]
  or [this battery pack][power-supply-portable].  I would have liked to use the
  AA batteries for power, but common DC-DC boost circuits like the
  [MintyBoost][mintyboost] only support up to 1A.
* [Kapton tape][kapton] for insulation and mounting.

[^pi-zero-buy]: I picked up my Pi Zero W at Micro Center during a previous trip to the east coast. As of this post, [they have them for $5][pi-zero-mc].
[^vetco-microphone]: At Vetco, this item corresponds to three sizes of microphone. I went with the largest one they had because it has an integrated amplifier circuit.

[pi-zero]: https://www.raspberrypi.org/products/raspberry-pi-zero-w/
[pi-zero-mc]: http://www.microcenter.com/product/486575/Zero_W
[avs]: https://developer.amazon.com/alexa-voice-service
[echo-dot]: https://www.ifixit.com/Teardown/Amazon+Echo+Dot+Teardown/61304
[audio-adapter]: https://www.amazon.com/Sabrent-External-Adapter-Windows-AU-MMSA/dp/B00IRVQ0F8/ref=as_li_ss_tl?ie=UTF8&qid=1515618548&sr=8-3&keywords=usb+headphone+adapter&linkCode=ll1&tag=albertarmeabl-20&linkId=a1db2fa73cf4a9027069fca2e7c2906f
[usb-otg]: https://www.amazon.com/Rankie-3-Pack-Female-Converter-Adapter/dp/B00YOX4JU6/ref=as_li_ss_tl?ie=UTF8&qid=1515726330&sr=8-1&keywords=usb+otg+adapter&linkCode=ll1&tag=albertarmeabl-20&linkId=b38d9478786e964e346fd7679818ab4b
[amp-circuit]: http://www.circuitbasics.com/build-a-great-sounding-audio-amplifier-with-bass-boost-from-the-lm386/
[microphone]: https://vetco.net/products/button-microphone
[motor-controller]: https://vetco.net/products/ba6286-single-h-bridge-motor-driver-15v-1a
[power-supply]: https://www.amazon.com/CanaKit-Raspberry-Supply-Adapter-Charger/dp/B00MARDJZ4/ref=as_li_ss_tl?ie=UTF8&qid=1515728162&sr=8-1-spons&keywords=raspberry+pi+power+supply&psc=1&linkCode=ll1&tag=albertarmeabl-20&linkId=1bed8e9f956168b8e8d8903a7730d2f4
[power-supply-portable]: https://www.amazon.com/Anker-20100mAh-Portable-Charger-PowerCore/dp/B00X5RV14Y/ref=as_li_ss_tl?ie=UTF8&qid=1516158179&sr=8-3&keywords=anker+power+bank&linkCode=ll1&tag=albertarmeabl-20&linkId=8c68574efe7edc07cadf8f816025d8eb
[kapton]: https://vetco.net/products/kapton-tape-18mm-wide-x-30-meters-long
[mintyboost]: https://www.adafruit.com/product/14

## Prototyping on a breadboard

With the parts picked out, I needed to test the components on a breadboard.
First, I removed the plastic case from the audio adapter using a spudger and
soldered wires to the headphone and microphone contacts to make it more
breadboard-friendly. Then I wired up the "great sounding" version of the
amplifier circuit by following their schematic, and next to it I wired the two
motor controllers using [the datasheet][motor-controller-datasheet] and some
experimentation. I omitted the volume potentiometer in the final circuit because
there is one built in to the audio adapter.

[motor-controller-datasheet]: /post/alexa-tree/ba6286.pdf

![Breadboard with amplifier and motor circuits](/post/alexa-tree/breadboard.jpg)

The wires leaving the audio circuit go to the audio adapter's microphone in and
headphone out.

The motor controller isolates the power supply to itself from the supply for the
motor. While this is useful for high-power applications, the 5V USB supply is
sufficient for the tree's motors so I tied these pins together. The controller
also has a pin to set the output current by connecting a resistor to 5V. On the
breadboard, I used a potentiometer to determine the correct values. For the
dancing motor, a 57 ohm resistor made the motor spin at the same speed as it did
originally. The mouth needed the maximum current to stay open, so I just used a
wire here. Before I soldered a connection from the logic input pins to the Pi, I
measured the current through it by connecting an ammeter in series between each
pin (pictured above as loose yellow wires connected to of the motor controllers)
and a 3.3V supply. With the motor running, the current was on the order of
microamps, which the Pi is able to drive without issues. Refer to the Raspberry
Pi GPIO [pinout][pi-pinout] when choosing pins.

[pi-pinout]: https://pinout.xyz/

## Final assembly

Since I planned to make only one of these trees, I soldered the two circuits
directly to a piece of perfboard. While this meant I didn't have to design and
etch a printed circuit board, that might have been preferable: manually routing
connections was time consuming, making intentional solder blobs for connections
was annoying, and the extra wire length added some noise to the amplifier
circuit. In the end, though, I was able to get the circuits to fit on a single
board about the size of the Pi itself.

<!-- convert IMG_20171225_060602.jpg IMG_20171225_060614.jpg -crop 1588x1588+0+0 +append -strip -resize 1600 -interlace Plane -quality 70 perfboard.jpg -->
![Protoboard with motor control and amplifier circuits, front and
back](/post/alexa-tree/protoboard.jpg)

Because the bottom of the Pi and my circuits would be back-to-back, I insulated
them with Kapton tape. Electrical tape probably would have been fine, but
because the motor controllers got hot (but not burning) to the touch, I didn't
want to take any chances.

On the Pi, the red and black wires go to the 5V and ground "rails" on the
perfboard. The white wires go to the switch, which works by tying GPIO pin 18 to
ground when pushed. The blue wires go to the logic inputs of the motor
controllers. The remaining wires are for the tree's LEDs. I connected a resistor
in series to limit the current through the LEDs to 20 mA. (The tree has another
set of LEDs too, but I accidentally burned them out while probing the components
to figure out how they were wired.)

![Raspberry Pi with connections to tree and custom
circuits](/post/alexa-tree/pi-connections.jpg)

Conveniently, the enclosure had a hole in front that was the perfect size for
the microphone.

Not so conveniently, my parts were still too big to fit in the enclosure. My
solution was to cut off the battery bay and two screw terminals for some extra
space, which also lets me access the SD card without disassembling the tree for
installing Raspbian and Wi-Fi configuration. (When assembling the enclosure,
make sure that the SD card slot is facing the bottom.)

<!-- convert IMG_20171225_081757.jpg -interlace Plane -quality 70 -strip -resize 1600 circuits-enclosure.jpg -->
![Circuits next to the enclosure, demonstrating their relative
sizes](/post/alexa-tree/circuits-enclosure.jpg)

Before screwing the enclosure shut, I connected a micro USB cable to the Pi's
power input. Then I wrote down the GPIO pins the motors, button, and LEDs were
connected to, and re-assembly was just the disassembly in reverse. I drilled a
small hole in the battery cover for the USB cable before screwing it back on.

# Setting up Alexa

## Raspbian and access via local network

The Raspberry Pi is stuck inside the tree with no way to display anything, so a
graphical interface would just be bloat. As a result, I used the [Raspbian
Stretch Lite][raspbian] image to start with. Download and install the image
according to [their instructions][raspbian-setup].

[raspbian]: https://www.raspberrypi.org/downloads/raspbian/
[raspbian-setup]: https://www.raspberrypi.org/documentation/installation/installing-images/README.md

There is no keyboard, serial connection, or Ethernet port, so the next step is
to set up Wi-Fi and SSH before booting up the Pi for the first time. In the SD
card's boot partition, create a basic `wpa_supplicant.conf`. You may have to
remove and reinsert the SD card into your computer if the boot partition isn't
mounted after installing Raspbian.

```conf
# If you're not in the US, replace the country code for regulatory compliance
country=US
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

# WPA
network={
	ssid="your network name"
	psk="your network password"
	key_mgmt=WPA-PSK
}

# WEP
network={
	ssid="your WEP network name"
	key_mgmt=NONE
	wep_key0=0123456789
	wep_tx_keyidx=0
}

# Public network
network={
	ssid="public network name"
	key_mgmt=NONE
}

# Add more network entries and the tree will connect to the first available one
```

Also in the boot partition, create an empty file named `ssh` to enable SSH.
Unmount the SD card from your computer, put it in the tree's Pi, and plug in the
tree to your USB power supply. With this configuration, Raspbian will boot,
connect to one of the specified Wi-Fi networks, and enable SSH with the default
credentials.

Then, discover the Pi's IP address using `arp`, `nmap`, or another tool
according to [this Stack Exchange post][pi-discover]. With the IP address in
hand, connect to the Pi using the default username "pi" and password
"raspberry". This is very insecure -- once connected, change the password via
`passwd` to prevent others on your network from accessing your Pi.

[pi-discover]: https://raspberrypi.stackexchange.com/questions/13936/find-raspberry-pi-address-on-local-network

I also recommend these other steps for security hygiene:

* Set up [key-based login][ssh-key] and optionally disable password login over
  SSH.
* Install [fail2ban][fail2ban] to prevent brute-force login attacks.
* Set up [automatic updates][auto-update] via the `unattended-upgrades` and
  `apt-listchanges` packages.

[ssh-key]: https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2
[fail2ban]: https://packages.debian.org/jessie/fail2ban
[auto-update]: https://wiki.debian.org/UnattendedUpgrades

## AlexaPi for Alexa

I decided to use [AlexaPi][alexapi] to actually call the Alexa Voice Service
instead of [Amazon's reference app][alexa-sample] because AlexaPi lets you
choose a wakeword engine, supports GPIO pins for triggers and status lights, and
none of its components expire after 120 days.

[alexapi]: https://github.com/alexa-pi/AlexaPi
[alexa-sample]: https://github.com/alexa/alexa-avs-sample-app

First, follow their [instructions to set up your Amazon
account][alexapi-amazon]. Then, install AlexaPi on your Pi:

```bash
# Install here for ease of use with their scripts
cd /opt

# Clone the AlexaPi repository
sudo apt-get install git
sudo git clone https://github.com/alexa-pi/AlexaPi.git
cd AlexaPi

# Latest from dev branch as of this writing, includes Snowboy wakeword detector
sudo git checkout 8cc9f02

# Install depenencies and setup account with their script
sudo src/scripts/setup.sh
```

[alexapi-amazon]: https://github.com/alexa-pi/AlexaPi/wiki/Installation/53b3b23be05d8ef038ebe130fdba34c739e3e1bd#1-register-at-amazon

Then, I needed to set up PulseAudio to run in system-wide mode. The AlexaPi
project has [some instructions][alexapi-pulseaudio] that mostly worked for me. I
opted to use the VLC handler, but audio input did not work out of the box. What
worked for me was to manually specify the input device rather than use "pulse"
in AlexaPi's configuration. By default, this configuration is in
`/etc/opt/AlexaPi/config.yaml`:

```yaml
sound:
  # Name of your microphone device
  # If you provide an invalid one, the log (`journalctl -u AlexaPi.service`)
  # gives you a list of valid devices during startup
  input_device: "USB Audio Device: - (plughw:1,0)"

  playback_handler: "vlc"

  output: "pulseaudio"
  output_device: ""

  # Set this to the highest level that doesn't distort
  default_volume: 80
```

[alexapi-pulseaudio]: https://github.com/alexa-pi/AlexaPi/wiki/Audio-setup-&-debugging/ef2c312c7fa1642b943318d0e635fc3f46164ac3#running-pa-in-system-wide-mode-recommended

I also needed to configure ALSA to work with PulseAudio. Paste this into
`/etc/asound.conf`:

```conf
pcm.!default {
	type pulse
}
ctl.!default {
	type pulse
}
```

At this point, you should be able to reboot your Pi and once Alexa says "Hello",
the tree will work as an Echo (albeit poorly). If not, check the log via
`journalctl -u AlexaPi.service` to see what specifically went wrong. If you
still need to set up the output device, I found the [Arch Wiki's documentation
on PulseAudio][pulseaudio-output] useful.

[pulseaudio-output]: https://wiki.archlinux.org/index.php/PulseAudio/Examples#Set_the_default_output_source

While testing, I determined that [Snowboy][snowboy], a purpose-built hotword
detector, was much more reliable than using the general-purpose speech
recognizer [CMUSphinx][cmusphinx] for this purpose. To enable this, in AlexaPi's
`config.yaml`, first disable the Sphinx recognizer and then enable Snowboy:

[snowboy]: https://snowboy.kitt.ai/
[cmusphinx]: https://cmusphinx.github.io/

```yaml
triggers:
  pocketsphinx:
    enabled: false

  snowboy:
    enabled: true
    voice_confirm: true

    model: "{distribution}/alexa/alexa_02092017.umdl"
    sensitivity: 0.5
```

To test these changes quickly, you can restart just the AlexaPi service without
restarting the entire Pi with `sudo systemctl restart AlexaPi.service`. The
wakeword will still be "Alexa", but it should be much more responsive.

Given that I just stuffed Alexa into a Christmas tree, I figured an appropriate
wake phrase would be "Oh Christmas tree". On [Snowboy's
dashboard][snowboy-dashboard], you can train and download a new model for any
phrase after creating an account. Alternately, you can just use [my
model][christmastree-model], though it might not work very well for you because
it's trained only on my voice. You can also record some samples to improve my
model on [its page][christmastree-page].

[snowboy-dashboard]: https://snowboy.kitt.ai/dashboard
[christmastree-model]: /post/alexa-tree/Oh_Christmas_Tree.pmdl
[christmastree-page]: https://snowboy.kitt.ai/hotword/14881

Once you have a model file, copy it to the Pi somewhere the alexapi account can
access, like inside `/opt/AlexaPi`, using `scp` or a graphical tool like
[FileZilla][filezilla]. Then, edit AlexaPi's configuration to use the model:

```yaml
triggers:
  snowboy:
    model: "/opt/AlexaPi/Oh_Christmas_Tree.pmdl"
```

[filezilla]: https://filezilla-project.org/

Finally, set up the GPIO pins to the pins so that they match the the pins you
chose earlier for the motors, lights, and button. To test the mouth motor, I
assigned its pin to the playback light. When testing, if the mouth moves but
does not open, its motor is spinning in the wrong direction. Change the
configuration to the mouth's alternate pin to spin the motor in the other
direction and the mouth should open for all of Alexa's response.

My complete AlexaPi `config.yaml` is available [here][alexapi-config].

[alexapi-config]: https://gist.github.com/aarmea/4f29cb7c4a46e6a1f936fe1950d7a87f

<div class="video-container">
  <iframe src="https://www.youtube.com/embed/k3fyvoydrGU?showinfo=0" frameborder="0" allowfullscreen></iframe>
</div>

## Moving the mouth naturally

AlexaPi does not support animatronic mouth movements out of the box, so I
researched how similar projects achieved mouth movements:

* [Furlexa][furlexa-guide]'s gearing makes the mouth open and close continuously
  the entire time the motor is running. Unlike Furlexa, the tree's mouth opens
  when power is applied and closes when it is removed.
* The Echo Dot-based projects, like [Project Yorick][yorick] and [another
  Christmas tree][echo-tree], apply a threshold to the Echo's sound output to
  determine the mouth's position. Unfortunately, these projects use dedicated
  hardware to do this, which I did not want to add.
* This [Billy Bass Alexa clone][billy-bass-clone] uses a Raspberry Pi for Alexa
  but has a separate microphone next to the speaker to capture the sound data
  needed to animate the mouth. This is another example of extra hardware that
  should not be necessary.

[furlexa-guide]: https://howchoo.com/g/otewzwmwnzb/amazon-echo-furby-using-raspberry-pi-furlexa
[yorick]: https://www.youtube.com/watch?v=3Nss_2_rwdE
[echo-tree]: https://www.youtube.com/watch?v=QIIQkEWcyrA
[billy-bass-clone]: https://anotherpiblog.blogspot.com/2017/01/billy-bass-alexa-and-raspberry-pi.html

My first thought was to patch AlexaPi to analyze the audio as it is being
played. However, Amazon returns an MP3, and AlexaPi hands off playback
responsibility to VLC. Decoding this again outside VLC just to control the
motors seemed redundant.

PulseAudio provides [`pacat`][pacat-man] to redirect audio streams to files and
vice versa, much like `cat` does for text. I used this to capture the system
audio output in realtime with the [RPi.GPIO Python library][gpio-library] to
control the motors in [my script][mouth-script]. Change the mouth pin and
PulseAudio source to match your hardware, then run the script (ideally inside a
`tmux` or `screen` session so that you can disconnect from the Pi) to enable
natural mouth movements.

[pacat-man]: https://linux.die.net/man/1/pacat
[gpio-library]: https://pypi.python.org/pypi/RPi.GPIO
[mouth-script]: https://gist.github.com/aarmea/f3010fd58629acda9c458e883ee010b5

<!-- ffmpeg -i VID_20180110_182049~2.mp4 -s 1280x720 -vframes 1 -q:v 5 VID_20180110_182049~2-web.jpg -->
<video controls poster="/post/alexa-tree/moving-mouth.jpg">
  <!-- ffmpeg -i VID_20180110_182049~2.mp4 -s 1280x720 -c:v libvpx-vp9 -crf 42 -b:v 0 VID_20180110_182049~2-web.webm -->
  <source src="/post/alexa-tree/moving-mouth.webm" type="video/webm">
  <!-- ffmpeg -i VID_20180110_182049~2.mp4 -s 1280x720 -c:v libx264 -crf 26 -b:v 0 VID_20180110_182049~2-web.mp4 -->
  <source src="/post/alexa-tree/moving-mouth.mp4" type="video/mp4">
</video>
