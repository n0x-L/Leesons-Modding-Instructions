Tuesday, June 16, 2020
# Leesons Guide to Android Modding

## Sections
* Setting Up a Build Environment
* Creating the Build
* Flashing the Build to the Phone
* Adding GMS (Google Mobile Services) to the Phone
* Adding Log Lines to the Phone
* Adding Log Lines to Apps
* Useful Logcat Commands
* Smali Logging
* Useful adb Commands

## Assumptions:
You have basic knowledge of navigating a Linux computer VIA a terminal, aka you recognize most of the commands on the list like:
`$ cd`
`$ cat`
`$ grep`
`$ mkdir`
`$ sudo`
`$ ls`
etc..
or else practice your linux terminal skills:
https://overthewire.org/wargames/bandit/

You have basic knowledge of how Git works, if not I recommend the book Git Pro which you can download as a free Ebook from:
https://git-scm.com/book/en/v2

## Hardware/Software Prerequisites
* Has to be Ubuntu or some Linux distro, no Windows (Mac can work but this guide is written for Ubuntu)
* You can probably do it with 500G of hard drive space but I would go for 1TB so you aren't cutting it too close
* You can do it all with 8Gib of RAM but if you can get 16Gib or more that would be ideal
* Save yourself from a huge headache and make sure you have a proper USB cable for your phone, not some weiner cable from the nearest gas station.

## Setting Up a Build Environment

Main documentation: https://source.android.com/setup/start

The documentation on the site says the main version of Ubuntu supported right now is Ubuntu 14, however I started with Ubuntu 14 and because it’s so outdated a lot of issues came up (especially with $ repo sync) and I eventually moved to Ubuntu 18 (I wouldn’t recommend using 20 which also did not work for me because of certain default packages installed.)

You can do the following to get Ubuntu 18 installed on your system:

Download Ubuntu 18 ISO and create bootable usb
Load Ubuntu 18 onto computer VIA usb and install the OS

With Ubuntu 18 installed and running, start by applying necessary updates to the OS if they don’t prompt you to run them automatically:

`$ sudo apt-get update && sudo apt-get upgrade`

NOTE: I had a lot of issues when I was moving between versions of Ubuntu (I legit went from 20 to 14 to 16 to 18) and an issue I had going from Ubuntu 20 to 14 which stumped me for days was my display cable had to be changed to a VGA cable, otherwise it just looked like Ubuntu was frozen on the loading screen forever.

Download and install fastboot and adb (android debugging bridge)

`$ sudo apt install android-tools-adb android-tools-fastboot`

.. along with other necessary libraries

`$ sudo apt-get install openjdk-8-jdk android-tools-adb bc bison build-essential curl flex g++-multilib gcc-multilib gnupg gperf imagemagick lib32ncurses5-dev lib32readline-dev lib32z1-dev libesd0-dev liblz4-tool libncurses5-dev libsdl1.2-dev libssl-dev libwxgtk3.0-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc yasm zip zlib1g-dev`

**Install Git**

`$ sudo apt install git`

Configure git with your info

`$ git config --global user.name "Your Name”`  
`$ git config --global user.email "you@example.com"`

Create bin directory (for linking, etc)
ref: https://source.android.com/setup/build/downloading

`$ mkdir ~/bin`

and add it to your PATH:

`$ PATH=~/bin:$PATH`

More info for Mac users: https://mycyberuniverse.com/create-personal-bin-directory-run-scripts-without-specifying-full-path.html


**Install the Repo tool and ensure its executable**

`$ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo`  
`$ chmod a+x ~/bin/repo`


Initialize an empty directory to be your working directory for your Android build and navigate into it

`$ mkdir WORKING_DIRECTORY`  
`$ cd WORKING_DIRECTORY`

Run the repo init command in your new working directory to get the latest version of the Android repo you want. You can (and probably should) specify a branch for a particular build. In my case I chose the newest version of Android Nougat for the Nexus 5x:
(other branches, second table down the page: https://source.android.com/setup/start/build-numbers#source-code-tags-and-builds) 

`$ repo init -u https://android.googlesource.com/platform/manifest -b android-7.1.2_r28`

NOTE: When choosing which Tag/branch you want, take note of the corresponding Build code (displayed in the first column) which you will use to determine which device drivers to download in the “Add device drivers” step coming up soon

Now for the big download of the Android source tree (> 1 hr)

`$ repo sync`

NOTE: This step also adds the .repo file in your WORKING_DIRECTORY (use $ls -a from your WORKING_DIRECTORY to see it) which contains the ‘Manifest’ file (not the same as the Manifest.xml file for apps) which you’ll modify later to include Google Mobile Services.


NOTE for Ubuntu 14 users: Often there are problems with this step since it is such a big download. I kept getting a "error TCP unexpected length" and "remote end hung up unexpectedly" and mostly issues receiving packets. Apparently this can be the cause of a problem with window scaling in the networking protocol. Window scaling is turned ON by default in linux (net.ipv4.tcp_window_scaling=1) and what can sometimes happen is that a router can rewrite the window scale TCP option on SYN packets as they pass through, with some being set to a scale factor of zero while leaving the option in place. The receiving side sees the option, and responds with a window scale factor of its own, and at this point the initiating system believes that its scale factor has been accepted and scales its window accordingly. The other end, however, believes that the scale factor is zero. The result is a misunderstanding over the real size of the receive window, with the system behind the router believing it to be much smaller than it really is. The expected scale factor is large, and the result is at best very slow communication. In many cases the small window can cause no packets to be transmitted at all, breaking TCP between the two affected systems entirely. 

Thus the proposed solution given by Android is to run the following command to turn off window scaling (a reboot of the system will reset it to being on)

`$ sudo sysctl -w net.ipv4.tcp_window_scaling=0`

then again run repo sync but with an option -j1:

`$ repo sync -j1`

NOTE: using -j1 in the repo sync command will use only 1 thread, and will sync the repositories by the correct order so to avoid dependency issues during the re-build.
(source: http://blog.udinic.com/2014/05/24/aosp-part-1-get-the-code-using-the-manifest-and-repo/)


**Add the device drivers**

https://developers.google.com/android/drivers

Download the appropriate drivers (link above) where you can find the matching downloads based on the phone type, branch/tag, Build code, and the ‘code’ name if applicable (which in our case it is since we use ‘bullhead’). 

For our purposes it’ll be the bullhead, Nexus 5x, Android 7.1.2, N2G48C, since we chose android-7.1.2_r28 when we did our branch sync. (remember the Build code to choose comes from the branch you chose in step 10.)
ie:
Where 
	Nexus 5x is the *Supported Device*
	Android 7.1.2 is the *Tag*
	N2G48C is the *Build* 

(ref second table down on page: https://source.android.com/setup/start/build-numbers#source-code-tags-and-builds)

Download both links (ZIPs) and extract the .sh scripts from the zips and place them in your WORKING_DIRECTORY. Then navigate into the resulting folders and run the scripts with ./ aka:

`$ ./extract-lge-bullhead.sh`  
`$ ./extract-qcom-bullhead.sh`

and accept the license agreements. (NOTE: if you find it’s exiting before you get a chance to type “I ACCEPT” press the space bar instead of the enter button through “reading” the license). 

The scripts should have created a “vendor” folder, which means you can safely delete the .sh scripts in the WORKING_DIRECTORY which set up folders in the new vendor folder.

## Creating the Build

From your WORKING_DIRECTORY initialize the tools for building by running:

`$ . build/envsetup.sh`

 Show the lunch menu by entering

`$ lunch`

and choose your build by typing in the corresponding number that appears next to the name and hitting enter.


To enable the use of ccache, all you need to do is make sure that the USE_CCACHE environment variable is set to 1 before you start your build:

`$ export USE_CCACHE=1`

You won’t gain any acceleration the first time you run, since the cache will be empty at that time. Every other time you build from scratch, though, the cache will help accelerate the build process.

Next check how many cpu cores you have:

`$ nproc`

And use that number after the ‘j’ in your make command that you will run next (ie I have 4 CPUs so I am using 4):

`$ make -j4`

NOTE: After about 90 mins you may run into an error such as:

"Building with Jack: intermediates/with-local/classes.dex 
FAILED: /bin/bash out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/with-local/classses.dex.rsp 
Out of memory error
...
make: *** [ninja_wrapper] Error 1"

which may be caused if you are using an 8GiB ram system instead of 16 GiB ram and trying to build an AOSP 7.x. You can fix the default heap size by adding the following line to ~/.jack-settings.

`$ nano ~/.jack-settings`

Then paste the following line at the bottom and save and exit:

`JACK_SERVER_VM_ARGUMENTS="-Dfile.encoding=UTF-8 -XX:+TieredCompilation -Xmx4096m"`

NOTE: It is best to COPY and PASTE the line above because when I tried just typing it in, the dashes differed (minus sign versus dash sign friends) and it didn't work.

Then restart the jack server:

`$ prebuilts/sdk/tools/jack-admin kill-server`  
`$ prebuilds/sdk/tools/jack-admin start-server`

Then try the make again

`$ make -j4`

Takes longer but should allow for compilation.

A successful build will be indicated saying in green text "build successful”

NOTE: may get an error about flex ie “FAILED: /bin/bash -c “prebuilts/misc/linux-x86/flex/flex-2.5.39 -out/host/linux …” in which case run the following command:

`$ export LC_ALL=C`

followed by the make command again

`$ make -j4`

Once you get a successful make message, checkout a local branch of frameworks/base from your WORKING_DIRECTORY:

`$ cd frameworks/base`  
`$ git checkout -b fworks_dev`

(where fworks_dev is just a name I made up)

## Flashing the Build to the Phone

Connect the Nexus 5X via USB cable (the charging cable) to the computer and run the following command to see if it’s connected:

`$ sudo adb devices`

Place the device in bootloader mode by running:

`$ sudo adb reboot bootloader`

(an image of the Android mascot laying down with it’s front panel open should appear in a few seconds on the screen of the phone)

**Flash the necessary partitions**

`$ sudo fastboot flash boot out/target/product/bullhead/boot.img`  
`$ sudo fastboot flash system out/target/product/bullhead/system.img`  
`$ sudo fastboot flash vendor out/target/product/bullhead/vendor.img`

Reboot the phone

`$ sudo fastboot reboot`

NOTE: You many get a message like "Android System There's an internal problem with your device. Contact your manufacturer for details." and this message can be ignored for now.

NOTES ABOUT PARTITIONS:

ramdisk.img
- Contains the root file system of Android, including 
  - init.* configuration files
  - default.prop containing the read only properties of this AOSP build
  - /system mounting point

system.img
- Contains the components generated by the AOSP build, including framework, applications, daemons

userdata.img
- Partition to hold the user data. Usually empty after the build

recovery.img, ramdisk-recovery.img
- basic image partition used to recover user data or even the actual system if anything goes wrong.


## Adding GMS (Google Mobile Services) to the Phone

Main reference: https://github.com/opengapps/opengapps

NOTE: You can either install GApps with a download from the above link or you can build your own GApps through the git repo.

**Download or Build GApps through the git repo**

To check which type of architecture the phone is on use the following adb command:

`$ adb shell cat /proc/cpuinfo`

Based on the following guide: https://developer.android.com/ndk/guides/abis the first line tells me its aarch64 or arm64

For the APK level I found this list, posted here for your convenience:

Platform Version   API Level

Android 9.0        28

Android 8.1        27

Android 8.0        26

**Android 7.1        25**

Android 7.0        24

Android 6.0        23

Android 5.1        22

Android 5.0        21

Android 4.4W       20

Android 4.4        19

Android 4.3        18

Android 4.2        17

Android 4.1        16

Android 4.0.3      15

Android 4.0        14

Android 3.2        13

Android 3.1        12

Android 3.0        11

Android 2.3.3      10

Android 2.3        9

Android 2.2        8

Android 2.1        7

Android 2.0.1      6

Android 2.0        5

Android 1.6        4

Android 1.5        3

Android 1.1        2

Android 1.0        1



`$ mkdir gapps`  
`$ cd gapps`  
`$ git clone https://github.com/opengapps/opengapps.git`  
`$ cd opengapps`  
`$ ./download_sources.sh --shallow arm64`  
`$ sudo apt-get install lzip`  
`$ make arm64-25-micro`  

**Download and install custom recovery tool “TWRP” so that you can install GApps**
(ref: https://www.xda-developers.com/how-to-install-twrp/)

To download, go to https://twrp.me/ and click on the "Devices" link at the top right, then you can pick your phone/build.

I’m going to use the “Fastboot Install Method”, meaning I click on the “Primary (Americas)” under “Download Links” and select the image at the top of the list.

After the download completes, navigate to the folder containing the twrp.img download via your terminal:

`$ cd ~/Downloads`

**Install TWRP via adb and fastboot**

`$ sudo adb reboot bootloader`  
`$ sudo fastboot flash recovery twrp.img`  
`$ sudo fastboot reboot`  

**Flash everything**

If you made any changes to the phone since the original flash, re-flash the images:

`$ sudo fastboot flash boot out/target/product/bullhead/boot.img`  
`$ sudo fastboot flash system out/target/product/bullhead/system.img`  
`$ sudo fastboot flash vendor out/target/product/bullhead/vendor.img`  
`$ sudo fastboot flash userdata out/target/product/bullhead/userdata.img`  
`$ sudo fastboot flash cache out/target/product/bullhead/cache.img`  

Then reboot the phone

`$ sudo fastboot reboot`

**Remount the /system Partition**
FYI I first tried mounting system via the original recovery AND via TWRP, neither succeeded. So without this step, my setup would confirm GApps was installed, but nothing would appear on the phone, aka this is the only way I discovered to properly mount the system partition.

`$ adb root`  
`$ adb disable-verity`  
`$ adb reboot`  
`$ adb root`  
`$ adb remount`  

**Install the GApps ZIP**

Boot into TWRP by booting into recovery, meaning:
`$ sudo adb reboot bootloader`

Use the volume buttons to scroll options to “recovery mode”, and when you get to it press the power button to enter. TWRP will eventually load.

Chose "Advanced", then "adb sideload", check off both wipe dalvik and cache boxes, then on your computer in the directory where your opengapps zip is sitting (by default it will be in the "out" folder of your opengapps git repo) run the following command to load and install the ZIP:

`$ adb sideload open_gapps-arm64-7.1-micro-20200706-UNOFFICIAL.ZIP`

Now choose “Reboot”; once complete you should see the Google apps on the phone.

Now set up phone with bogus Google account

Results:
-Started downloading Google Play Store
-Email from Google Community Team saying my account comes with access to Google products, apps, and services
-Started downloading Google Calendar
-When installing my app (adb install -g myapp.apk) for the first time it asked if I wanted to send the app for scanning, obviously I said NO)

## Adding Log Lines to the Phone

Make sure the phone is in developer mode and that USB debugging is enabled:
https://www.howtogeek.com/129728/how-to-access-the-developer-options-menu-and-enable-usb-debugging-on-android-4.2/

The way logs are set up in Android is simple, for each log there is a TAG which is just a string representing a name, the log message (another string), and a letter indicating the log “level” (ie information, error, warning, etc). This structure is as follows:

`Log.i(“TAG”, “log message”);`

Or

`Slog.i(“TAG”, “log message”);    // for System Log`


It can be tricky to find a file to place a log in because at least for me I wasn’t sure yet exactly which code would be run (and I didn’t want to write a log line that would never get executed). To deal with this I ran the logcat command and watched which logs were coming up so I could see what file were being executed, and to make it even easier I decided to run logcat while turning the bluetooth on and off, this gave me a lot of info on where I could find the code and some corresponding log lines that ran when bluetooth was turned on and off. I ran logcat like so 
`$ sudo adb logcat`

While regular logs don’t by default tell you where the file is, the tag is almost always the name of the class file. With a simple $ grep -lr ‘TAG = “BluetoothManagerService”’ for example, it showed me where I could find the corresponding class file. I was also purposefully looking for logs that would show up in the class files inside frameworks/base/services/core/java/com/android/server

Add a log line to one of the java code files in frameworks/base/services/core/java/com/android/server such as BluetoothManagerService.java

`Slog.i(“MyTag”, “Turned Bluetooth on!!!!!!”)`

(the excessive excitement of my log message is just so I can see it easily when I run logcat)

After saving the files you need to re-make the changed files and then re-flash the image. This all comes down to Android.mk files. From the root WORKING_DIRECTORY

`$ sudo adb devices`  
`$ . build/envsetup.sh`  
`$ lunch`  
`$ mmm frameworks/base/services`  
`$ make snod`  
`$ sudo adb reboot bootloader`  
`$ sudo fastboot flash system out/target/product/bullhead/system.img`  
`$ sudo fastboot reboot`


NOTE: It wasn't working at first because I was running the mmm command from frameworks/base/services/core instead of frameworks/base/services 
I changed it because when I looked at the Android.mk file in this directory it mentioned building the frameworks/base/services/core directory, so I figured this is the file I needed to run mmm with.

Now you can test it by running logcat with a filter on your tag (and turning bluetooth on and off if that’s where you ended up putting your log line):

`$ sudo adb logcat MyTag:I *:S`

Where MyTag is the name first parameter of the log (the tag), and the ‘I’ represents the log level (informational in this case) and the *:S says silence everything else (so we can filter out the noise)

NOTE: If nothing is printing other than:

It means there are no logs corresponding to your filter. It may mean the code where your log is hasn’t been run and you may have to place your logs in a different part of the class file.

If you’ve run all of this more than once you also may want to clear your logcat with the following in order to remove potential ‘false-positives’:

`$ sudo adb logcat -c`


## Adding Smali Log Lines to Apps 

**Install signapk for ubuntu**

`$ sudo apt install signapk`


**Clone the release-tools repository**

`$ git clone https://github.com/glitterballs/release-tools.git`


Decompile the apk. Navigate to the folder with the downloaded apk and run

`$ apktool d myapk.apk`
 
Navigate into the resulting folder

`$ cd myapk`

Add log lines to desired smali files, ie:


`move-result-object v0`  
`const-string v1, "MYTAG read input stream "`  
`invoke-static {v1, v0}, Landroid/util/Log;->i(Ljava/lang/String;Ljava/lang/String;)I`  

Save your changes and navigate back to the root folder of myapk where the android manifest, apktool.yml, and etc live. Now re-package the apk:

`$ apktool b $1 -o myapk.apk`

where "myapk" is the name of the folder you are in that represents the apk 

Check for the new .apk file, if it’s not in the current folder move up one up from the directory you're in where you will see the "unsigned.apk" file and run the next command to sign the apk:

`$ signapk /home/user_name/release-tools/SignApk/certificate.pem /home/user_name/release-tools/SignApk/key.pk8 unsigned.apk signed.apk`

NOTE: You cannot not use the same name for ‘unsigned.apk’ and ‘signed.apk’ or it will throw an IllegalArgumentException

and finally install the new apk
`$ adb install -r signed.apk`


NOTE: if you get an error like" adb: failed to install signed.apk: Failure [INSTALL_FAILED_UPDATE_INCOMPATIBLE: Package $name_of_app signatures do not match the previously installed version; ignoring!]
then just delete the app (simply uninstall it) from the phone and try again.


Note: Package manager install method:

`$ adb shell pm install APK_FILE`  
`$ adb shell pm uninstall com.example.MyApp`  


// installed earlier version of app
`$ apktool b apk_folder -o unsignedapp.apk`  
`$ signapk /certifcate.pem /key.pk8 unsignedapp.apk signedapp.apk`  
`$ sudo adb install -r signedapp.apk`

## Other Useful Logcat Commands


Filter on a specific app: (adb -d logcat <package_name>:I *:S)

`$ sudo adb -d logcat com.mobincube.android.sc_T58MY:I *:S`

Clear logcat

`$ sudo adb logcat -c`

Write logs to text file

`$ sudo adb logcat -d > myLogs.txt`

Apply log filters on log text:
`$ adb logcat -v tag | grep -e “failed”`

https://medium.com/@hissain.khan/filtering-android-adb-logcat-efficiently-in-bash-command-line-4992fb1acd61

Filtering logs by tag:

`$ adb logcat ActivityManager:I *:S`

where ActivityManager represents the TAG name, the I is the log level (info) and the *:S silences everything else
`$ adb logcat | grep -e “LEL-JJ\|mobincube\|mobimento\|instantcoffee\|socket”`


## Smali Logging

For recording an integer value

`const-string v8, "log-tag"`  
`invoke-static {v1}, Ljava/lang/String;->valueOf(I)Ljava/lang/String;`  
`move-result-object v9`  
`invoke-static {v8, v9}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)`

For recording an exception (they can take time to display the full message in logcat)

`const-string v3, “logTag classFile lineNo “`  
`const-string v4, “Exception “`   
`invoke-static {v8, v4, v0}, Landroid/util/Log;->i(Ljava/lang/String;Ljava/lang/String;Ljava/lang/Throwable;)I`

For reading a returned string

// command which returns a string
`move-result-object v0`  
`const-string v1, "MyTag read input stream "`  
`invoke-static {v1, v0}, Landroid/util/Log;->i(Ljava/lang/String;Ljava/lang/String;)I`

Start a script session in the terminal:

`$ script -a my_terminal_activities`


`$ sudo adb logcat -v tag | grep -e “LEL-JJ\|mobincube\|mobimento\|AndroidRuntime\|instantcoffee”`


Useful ADB Commands

To see architecture info (whether its ARM ARM64 or x86):

`$ sudo adb shell cat /proc/cpuinfo`




