# Compiling for Pocket PC

During the 2020 coronavirus lockdown, I bought some old Pocket PC-based mobile devices off eBay for a bit of fun, and wanted to see whether I could get them to run my own programs. This page details my explorations into this.

## Devices

The two devices I bought are:

### HTC Advantage X7500

![Image of HTC Advantage](https://upload.wikimedia.org/wikipedia/commons/f/f0/HTC_Athena.jpg)

[Wikipedia](https://en.wikipedia.org/wiki/HTC_Advantage_X7500)

£45. Who needs an iPad Pro when you have one of these?

### O2 XDA Stellar

![Image of O2 XDA Stellar](https://cdn-0.phonesdata.com/files/models/O2-XDA-Stellar-199.jpg)

[Wikipedia](https://en.wikipedia.org/wiki/O2_Xda#Xda_Stellar)

£7.99. Listed as spares/repairs because the seller didn't have the cables or sync software, but it turns out it works just fine. Bargain.

## File Transfer

First of all, we need to establish how to put software onto the devices. It would be nice to be able to plug them into the computer and just have them show up as USB mass storage devices, but apparently that's too much of an ask for 2007-era tech.

Either ActiveSync or the newer [Windows Mobile Device Centre](https://support.microsoft.com/en-gb/help/931937/description-of-windows-mobile-device-center) (with the option of [additional driver upgrades](https://www.microsoft.com/en-us/Download/confirmation.aspx?id=3182)) must be installed in order to communicate with the devices. This seems to be a bit of an issue in Windows 10, as the Creators Update broke compatibility with the utility. [This StackOverflow question](https://stackoverflow.com/questions/32052429/how-to-connect-a-windows-mobile-pda-to-windows-10) has some good instructions, but I found I also had to follow some more instructions from a page I've forgotten the link to:

* Open the Services utility (search `services.msc` in the start menu)
* Find "Windows 2003-based-device connectivity" and open the properties window for this service.
* Under the "Log On" tab, choose "Local System account".
* Restart the service.

Even then, connectivity is still spotty - it'll work sometimes, and other times not, for seemingly no reason. To make things easier, I used one of the times I did have connectivity to install the [Mocha FTP server](https://www.mochasoft.dk/freeware/ftpd.htm) on the devices. Since they're both able to connect to WiFi OK, it was much easier and more reliable to transfer files using an FTP client.

Note that the FTP server needs the license name `freeware` and the license key `111425` to bypass the trial state. The website does provide this information, but I missed it the first time I tried to use the application.

One other method, of course, is to copy any applications onto a compatible memory card and then insert this into the device. The XDA Stellar takes MicroSD cards, but the HTC Advantage (which was the one I wanted to use because of the bigger screen) takes MiniSD, which I wasn't even aware of before buying it. I have no MiniSD cards or adapters at home, and couldn't be bothered to have to wait for one to be delivered, so I went for the FTP server option instead.

## Compilers

The [CEGCC project](http://cegcc.sourceforge.net/) hosts compile tools based on GCC which allow compailation for the Windows CE (Pocket PC) platform. These tools are over ten years old, and after some experiments I wasn't able to make much progress with them. Luckily you can bypass that grief, along with the early-2000s frame-based website design, by using the much more up-to-date [GCC 9.3.0-based project](https://max.kellermann.name/projects/cegcc/) by Max Kellermann. The binaries are pre-built for [Debian x64](http://max.kellermann.name/download/xcsoar/devel/cegcc/mingw32ce-mk-2020-03-23-amd64.tar.xz), or can be built from scratch using the [GitHub repo](https://github.com/MaxKellermann/cegcc-build).

I created a simple test program on my Ubuntu system using the Debian-based CEGCC compilers, but oddly was unable to FTP to (or even ping) the devices from that system, even though they were definitely on the local WiFi network and had internet access, so couldn't test it. I don't know whether this is because Windows has some compatibility-mode networking layer that Pocket PC devices use but Ubuntu doesn't?

For better ease of testing, I installed the compilers on a virtual machine within Windows instead, so that I could get FTP access.

*EDIT: Turns out FTP isn't always reliable on Windows either... I'm starting to suspect it's the WiFi connection to the device that's the problem, given it's using 2007-era WiFi protocols and drivers.*

The following sections document the steps required to get the compilers to work.

## Stow

Since the binaries are provided as simple files in a tarball, they must be installed into a recognised directory on the system. I was reluctant to just lob them into the general `bin` and `lib` directories, but after a bit of searching discovered the `stow` tool. This manages creating symlinks to self-contained sets of binaries such as these compile tools: if you "install" binaries using `stow`, it creates symlinks in the binary directories recognised by the system that point to your target programs, and "uninstalling" them simply removes the relevant symlinks.

To use `stow` to install the compiler binaries, follow these steps:

* Extract the contents of the tarball to a specific directory, eg. `mingw32ce`.
* Copy this directory to the `stow` directory (this will probably require `sudo`). By default, this is `/usr/local/stow`.
* Open a terminal in the `/usr/local/stow` directory.
* Run `sudo stow -S mingw32`. Symlinks will be created in the relevant directories (`/usr/local/bin`, etc.).

## Sample Program

A simple program that utilises the WindowsCE API to show a message box is as follows:

``` c++
#include <windows.h>

int main(int argc, char** argv)
{
    MessageBox(nullptr, L"This is a Pocket PC application.", L"Message", MB_OK);
    return 0;
}
```

Save this in a working folder as `main.cpp`.

The CEGCC compiler for C++ code is `arm-mingw32ce-g++`. To compile the code, run:

```
arm-mingw32ce-g++ -o test.exe main.cpp
```

### "Archive has no index"

The first time you run this code, you'll likely get an error that looks like:

```
/usr/local/stow/mingw32ce/bin/../lib/gcc/arm-mingw32ce/9.3.0/../../../../arm-mingw32ce/bin/ld: /usr/local/stow/mingw32ce/bin/../lib/gcc/arm-mingw32ce/9.3.0/../../../../arm-mingw32ce/lib/libstdc++.dll.a: error adding symbols: archive has no index; run ranlib to add one
collect2: error: ld returned 1 exit status
```

I don't know much about the specifics of this, but I think it's to do with the fact that the static libraries provided with the compiler have not had their metadata built. To fix this, run:

```
sudo arm-mingw32ce-ranlib /usr/local/stow/mingw32ce/arm-mingw32ce/lib/libstdc++.dll.a
```

For me, this then popped up an additional error:

```
arm-mingw32ce-ranlib: error while loading shared libraries: libfl.so.2: cannot open shared object file: No such file or directory
```

This is because the `flex` library (for lexer support) was missing from my system. I installed it using `sudo apt install flex`, ran the previous command again, and it worked.

Now, after re-running the `g++` call, we get a `test.exe` executable. If this is copied onto the device and run, a message box will be displayed!

### "Error while loading shared libraries: libmpc.so.3"

I also tried the above on Windows Subsystem for Linux 2, out of interest. Upon first attempting to compile the test application, I got the error:

```
/usr/local/stow/mingw32ce/bin/../libexec/gcc/arm-mingw32ce/9.3.0/cc1plus: error while loading shared libraries: libmpc.so.3: cannot open shared object file: No such file or directory
```

This was fixed by installing the `libmpc-dev` package: `sudo apt install libmpc-dev'.
