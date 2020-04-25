
![NexMon logo](https://github.com/seemoo-lab/nexmon/raw/master/gfx/nexmon.png)

# Nexmon Channel State Information Extractor
This project allows you to extract channel state information (CSI) of OFDM-modulated
Wi-Fi frames (802.11a/(g)/n/ac) on a per frame basis with up to 80 MHz bandwidth 
on the Broadcom Wi-Fi Chips listed below.

WiFi Chip   | Firmware Version  | Used in
----------- | ----------------- | --------------------
bcm4339     | 6_37_34_43        | Nexus 5
bcm43455c0  | 7_45_189          | Raspberry Pi B3+/B4
bcm4358     | 7_112_300_14_sta  | Nexus 6P
bcm4366c0   | 10_10_122_20      | Asus RT-AC86U

## Usage

After following the [getting started](#getting-started) guide for your device below, you can begin extracting CSI by doing the following. The first step can be run locally or on the extraction device, all the subsequent steps shall be executed on the latter.
1. Use utils/makecsiparams/makecsiparams to generate a base64 encoded parameter string that can be used to configure the extractor.
   The following example call generates a parameter string that enables collection on channel 157 with 80 MHz bandwidth on the first core for the first spatial stream for frames starting with 0x88 originating from 00:11:22:33:44:55 or aa:bb:aa:bb:aa:bb:
    ```
    makecsiparams -c 157/80 -C 1 -N 1 -m 00:11:22:33:44:55,aa:bb:aa:bb:aa:bb -b 0x88
    m+IBEQGIAgAAESIzRFWqu6q7qrsAAAAAAAAAAAAAAAAAAA==
    ```
   For a full list of possible parameters run `makecsiparams -h`.
2. *bcm43455c0 only*: make sure wpa_supplicant is not running: `pkill wpa_supplicant`
3. Make sure your interface is up: `ifconfig wlan0 up` (replace wlan0 with your interface name)
4. Configure the extractor using nexutil and the generated parameters (adapt the argument of -v with your parameters):
    ```
    nexutil -Iwlan0 -s500 -b -l34 -vm+IBEQGIAgAAESIzRFWqu6q7qrsAAAAAAAAAAAAAAAAAAA==
    ```
5. Enable monitor mode:

    **bcm4339,bcm4358**: `nexutil -Iwlan0 -m1`

    **bcm43455c0**:
    ```
    iw phy `iw dev wlan0 info | gawk '/wiphy/ {printf "phy" $2}'` interface add mon0 type monitor
    ifconfig mon0 up
    ```

    **bcm4366c0**: `/usr/sbin/wl -i eth6 monitor 1`
6. Collect CSI by listening on UDP socket 5500, e.g. by using tcpdump: `tcpdump -i wlan0 dst port 5500`. There will be one UDP packet per configured core and spatial stream for each incoming frame matching the configured filter.

## Analyzing the CSI

Each UDP packet containing collected CSI has 10.10.10.10 as source address and is destined to 255.255.255.255 on port 5500. The payload starts with four magic bytes 0x11111111, followed by the six byte source mac address as well as the two byte sequence number of the Wi-Fi frame that triggered the collection of the CSI contained in this packet. The next two bytes contain core and spatial stream number where the lowest three bits indicate the core and the next three bits the spatial stream number, e.g. 0x0019 (0b00011001) means core 0 and spatial stream 3. The chanspec used during extraction can be found in the subsequent two bytes. After two bytes identifying the chip version, the actual CSI data follows. Relative to using 20, 40, or 80 MHz wide channels those are 64, 128, or 256 times four bytes long. For the bcm4339 and bcm43455c0 the data contains interleaved int16 real and int16 imaginary parts for each complex CSI value. The bcm4358 and bcm4366c0 return values in a floating point format with one bit sign of the following nine or twelve bits of a real part and the same for an imaginary part, followed by an exponent of five or six bits. We provide matlab scripts under utils/matlab/ for reading and plotting both formats. Make sure to compile a mex file from utils/matlab/unpack_float.c before reading values of the bcm4358 or bcm4366c0 for the first time. Then fill in the configuration section in utils/matlab/csireader.m and run the script. There is an example capture file utils/matlab/example.pcap holding four UDPs of a capture on a bcm4358 for two cores and two spatial streams.

## Example

The figure below shows the amplitude of 80MHz CSI changing over time extracted for one core and one spatial stream on a bcm4366c0 (Asus RT-AC86U). Corresponding Wi-Fi frames were transmitted using frame injection on a Nexus 5 smartphone, enabled using [nexmon](https://nexmon.org). The quick and heavy alternations occuring in regular intervals result from shaking the transmitter. For the rest of the capture the transmitter was pretty much kept steady in one hand.

![Example of 80MHz CSI over time](gfx/csi-example-over-time.svg)

# Getting Started

To compile the source code, you are required to first clone the original nexmon repository that contains our C-based patching framework for Wi-Fi firmwares. Then you clone this repository as one of the sub-projects in the corresponding patches sub-directory. This allows you to build and compile all the firmware patches required to extract CSI. The following guides you through the required procedure for the different platforms.

## bcm4339, bcm4358

The following steps will get you started on Xubuntu 16.04 LTS:
1. Install some dependencies: `sudo apt-get install git gawk qpdf adb flex bison`
2. **Only necessary for x86_64 systems**, install i386 libs: 
  ```
  sudo dpkg --add-architecture i386
  sudo apt-get update
  sudo apt-get install libc6:i386 libncurses5:i386 libstdc++6:i386
  ```
3. Clone the nexmon base repository: `git clone https://github.com/seemoo-lab/nexmon.git`.
4. Download and extract Android NDK r11c (use exactly this version!).
5. Export the NDK_ROOT environment variable pointing to the location where you extracted the 
   ndk so that it can be found by our build environment.
6. Navigate to the previously cloned nexmon directory and execute `source setup_env.sh` to set 
   a couple of environment variables.
7. Run `make` to extract ucode, templateram and flashpatches from the original firmwares.
8. Navigate to utilities and run `make` to build all utilities such as nexmon.
9. Attach your rooted Nexus 5 or Nexus 6P smartphone.
10. Run `make install` to install all the built utilities on your phone.
11. Navigate to patches/bcm43{39,58}/{6_37_34_43,7_112_300_14_sta}/ and clone this repository: 
    `git clone https://github.com/seemoo-lab/nexmon_csi.git`
12. Enter the created subdirectory nexmon_csi and run 
    `make install-firmware` to compile our firmware patch and install it on the attached smartphone.

## bcm43455c0

On your Raspberry Pi 3B+/4 running Raspbian with kernel 4.19 run the following:
1. Make sure the following commands are executed as root: `sudo su`
2. Upgrade your Raspbian installation: `apt-get update && apt-get upgrade`
3. Install the kernel headers to build the driver and some dependencies: `apt install raspberrypi-kernel-headers git libgmp3-dev gawk qpdf bison flex make`
4. Clone the nexmon base repository: `git clone https://github.com/seemoo-lab/nexmon.git`.
5. Go into the root directory of the repository: `cd nexmon`
5. Check if `/usr/lib/arm-linux-gnueabihf/libisl.so.10` exists, if not, compile it from source:
   `cd buildtools/isl-0.10`, `./configure`, `make`, `make install`, `ln -s /usr/local/lib/libisl.so /usr/lib/arm-linux-gnueabihf/libisl.so.10`
6. Check if `/usr/lib/arm-linux-gnueabihf/libmpfr.so.4` exists, if not, compile it from source:
   `cd buildtools/mpfr-3.1.4`, `./configure`, `make`, `make install`, `ln -s /usr/local/lib/libmpfr.so /usr/lib/arm-linux-gnueabihf/libmpfr.so.4`
8. Then you can setup the build environment for compiling firmware patches
   * Setup the build environment: `source setup_env.sh`

   * Run `make` to extract ucode, templateram and flashpatches from the original firmwares.
9. Navigate to patches/bcm43455c0/7_45_189/ and clone this repository:
    `git clone https://github.com/seemoo-lab/nexmon_csi.git`
10. Enter the created subdirectory nexmon_csi and run
    `make install-firmware` to compile our firmware patch and install it on the Raspberry Pi.
11. Install nexutil: from the nexmon root directory switch to the nexutil folder: `cd utilities/nexutil/`. Compile and install nexutil: `make && make install`.
12. *Optional*: remove wpa_supplicant for better control over the WiFi interface: `apt-get remove wpasupplicant`

## bcm4366c0

The following steps will get you started on Xubuntu 18.04.3 LTS:
1. Install some dependencies: `sudo apt-get install git gawk qpdf flex bison`
2. **Only necessary for x86_64 systems**, install i386 libs: 
  ```
  sudo dpkg --add-architecture i386
  sudo apt-get update
  sudo apt-get install libc6:i386 libncurses5:i386 libstdc++6:i386
  ```
3. Clone the nexmon base repository: `git clone https://github.com/seemoo-lab/nexmon.git`.
4. Navigate to the previously cloned nexmon directory and execute `source setup_env.sh` to set 
   a couple of environment variables.
5. Run `make` to extract ucode, templateram and flashpatches from the original firmwares.
6. Navigate to patches/bcm4366c0/10_10_122_20/ and clone this repository: 
    `git clone https://github.com/seemoo-lab/nexmon_csi.git`
7. Enter the created subdirectory nexmon_csi and run 
    `make install-firmware REMOTEADDR=<address of your rt-ac86u>` to compile our firmware patch and install it on your RT-AC86U router.
8. Clone the aarch64 toolchain repository: `git clone https://github.com/RMerl/am-toolchains.git`.
9. Set the compile environment:
  ```
  export AMCC=$(pwd)/am-toolchains/brcm-arm-hnd/crosstools-aarch64-gcc-5.3-linux-4.1-glibc-2.22-binutils-2.25/usr/bin/aarch64-buildroot-linux-gnu-
  export LD_LIBRARY_PATH=$(pwd)/am-toolchains/brcm-arm-hnd/crosstools-aarch64-gcc-5.3-linux-4.1-glibc-2.22-binutils-2.25/usr/lib
  ```
10. Go back to the nexmon repository root, compile and install nexutil:
  ```
  cd utilities/libnexio
  ${AMCC}gcc -c libnexio.c -o libnexio.o -DBUILD_ON_RPI
  ${AMCC}ar rcs libnexio.a libnexio.o
  cd ../nexutil
  echo "typedef uint32_t uint;" > types.h
  sed -i 's/argp-extern/argp/' nexutil.c
  ${AMCC}gcc -static -o nexutil nexutil.c bcmwifi_channels.c b64-encode.c b64-decode.c -DBUILD_ON_RPI -DVERSION=0 -I. -I../libnexio -I../../patches/include -L../libnexio/ -lnexio
  scp nexutil admin@<address of your rt-ac86u>:/jffs/nexutil
  ssh admin@<address of your rt-ac86u> "/bin/chmod +x /jffs/nexutil"
  ```

# Extract from our License

Any use of the Software which results in an academic publication or
other publication which includes a bibliography must include
citations to the nexmon project a) and the paper cited under b):

a) "Matthias Schulz, Daniel Wegemer and Matthias Hollick. Nexmon:
       The C-based Firmware Patching Framework. https://nexmon.org"

b) "Francesco Gringoli, Matthias Schulz, Jakob Link, and Matthias
       Hollick. [Free Your CSI: A Channel State Information Extraction
       Platform For Modern Wi-Fi Chipsets](https://doi.org/10.1145/3349623.3355477). In Proceedings of the 13th
       Workshop on Wireless Network Testbeds, Experimental evaluation
       & CHaracterization (WiNTECH 2019), October 2019."
  ```
  @electronic{nexmon:project,
      author = {Schulz, Matthias and Wegemer, Daniel and Hollick, Matthias},
      title = {Nexmon: The C-based Firmware Patching Framework},
      url = {https://nexmon.org},
      year = {2017}
  }

  @inproceedings{10.1145/3349623.3355477,
      author = {Gringoli, Francesco and Schulz, Matthias and Link, Jakob and Hollick, Matthias},
      title = {Free Your CSI: A Channel State Information Extraction Platform For Modern Wi-Fi Chipsets},
      year = {2019},
      url = {https://doi.org/10.1145/3349623.3355477},
      booktitle = {Proceedings of the 13th International Workshop on Wireless Network Testbeds, Experimental Evaluation & Characterization},
      pages = {21–28},
      series = {WiNTECH ’19}
  }
  ```
# References

* Matthias Schulz, Daniel Wegemer and Matthias Hollick. **Nexmon: The C-based Firmware Patching 
  Framework**. https://nexmon.org
* Francesco Gringoli, Matthias Schulz, Jakob Link, and Matthias Hollick. **[Free Your CSI: 
  A Channel State Information Extraction Platform For Modern Wi-Fi Chipsets](https://doi.org/10.1145/3349623.3355477)**.
  In Proceedings of the 13th Workshop on Wireless Network Testbeds, Experimental evaluation & CHaracterization (WiNTECH 2019), October 2019. https://doi.org/10.1145/3349623.3355477

[//]: # "[Get references as bibtex file](https://nexmon.org/bib)"

# Contact

* [Francesco Gringoli](http://netweb.ing.unibs.it/~gringoli/) <francesco.gringoli@unibs.it>
* [Matthias Schulz](https://seemoo.tu-darmstadt.de/mschulz) <mschulz@seemoo.tu-darmstadt.de>
* Jakob Link <jlink@seemoo.tu-darmstadt.de>

# Powered By

## Secure Mobile Networking Lab (SEEMOO)
<a href="https://www.seemoo.tu-darmstadt.de">![SEEMOO logo](https://github.com/seemoo-lab/nexmon/raw/master/gfx/seemoo.png)</a>
## Multi-Mechanisms Adaptation for the Future Internet (MAKI)
<a href="http://www.maki.tu-darmstadt.de/">![MAKI logo](https://github.com/seemoo-lab/nexmon/raw/master/gfx/maki.png)</a>
## LOEWE centre emergenCITY
<a href="https://www.emergencity.de/">![emergenCITY logo](https://www.emergencity.de/assets/img/logo_emergencity.png)</a>
## Technische Universität Darmstadt
<a href="https://www.tu-darmstadt.de/index.en.jsp">![TU Darmstadt logo](https://github.com/seemoo-lab/nexmon/raw/master/gfx/tudarmstadt.png)</a>
## University of Brescia
<a href="http://netweb.ing.unibs.it/">![University of Brescia logo](https://github.com/seemoo-lab/nexmon/raw/master/gfx/brescia.png)</a>
