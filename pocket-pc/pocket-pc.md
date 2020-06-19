# Compiling Programs for Pocket PC in 2020

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

``` bash
$ arm-mingw32ce-g++ -o test.exe main.cpp
```

### "Archive has no index"

The first time you run this code, you'll likely get an error that looks like:

```
/usr/local/stow/mingw32ce/bin/../lib/gcc/arm-mingw32ce/9.3.0/../../../../arm-mingw32ce/bin/ld: /usr/local/stow/mingw32ce/bin/../lib/gcc/arm-mingw32ce/9.3.0/../../../../arm-mingw32ce/lib/libstdc++.dll.a: error adding symbols: archive has no index; run ranlib to add one
collect2: error: ld returned 1 exit status
```

I don't know much about the specifics of this, but I think it's to do with the fact that the static libraries provided with the compiler have not had their metadata built. For more useful information about what a static library `.a` file actually is and why this process is required, check out [this StackOverflow answer](https://stackoverflow.com/a/47924864).

To fix this, run:

```bash
$ sudo arm-mingw32ce-ranlib /usr/local/stow/mingw32ce/arm-mingw32ce/lib/libstdc++.dll.a
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

## Programming API

A useful document concerning the Windows API with respect to Windows CE can be found [here](https://users.cs.northwestern.edu/~pdinda/rtclass/windows.pdf). Microsoft API documentation for Windows CE can be found [here](https://docs.microsoft.com/en-us/previous-versions/windows/embedded/aa452822%28v%3dmsdn.10%29).

### Creating a Window

It seems to be very easy to create a floating window for Windows CE, but quite difficult to work out the correct set of parameters to create a normal fullscreen window with an "OK" button in the top right. After some experimentation, I found that the following call seems to work:

``` c++
CreateWindow(L"WindowClass",        // Class
             L"WindowTitle",        // Title
             WS_VISIBLE,            // Style
             CW_USEDEFAULT,         // X
             CW_USEDEFAULT,         // Y
             CW_USEDEFAULT,         // Width
             CW_USEDEFAULT,         // Height
             NULL,                  // Parent window
             NULL,                  // Child/menu
             hApplicationInstance,  // Instance passed into WinMain
             NULL);                 // User data
```

The crucual parameter appears to be the window style `WS_VISIBLE`. If this is ommitted, the window is displayed as floating and without a close button, requiring it to be killed through the task manager.

### Using Windows Libraries

Beyond the core functionality of the Windows API, certain controls will require linking to the Windows API libraries. I got a bit confused at first regarding how to do this, but it turned out it was just because of my lack of experience in using the GCC command line.

If you want to link with what would normally be CommCtrl.dll on Windows, the compiler invocation would look something like this:

``` bash
# Compile the source file into an object file (main.o)
$ arm-mingw32ce-g++ -c main.cpp

# Link main.o into an executable called test.exe, linking also against libcommctrl.a
# The linker -l option automatically adds the "lib" prefix and the ".a" suffix, so
# all you need to supply is -lcommctrl
$ arm-mingw32ce-g++ -o test.exe main.o -lcommctrl
```

Note that the CEGCC toolchain provides all the relevant Windows modules as static libraries, as far as I can see. We may need to see what can be done to link against these dynamically in future.

### Showing the Bottom Command Bar

By default, a standard window created with the `CreateWindow()` function call will not display the command bar at the bottom of the screen. However, while searching through the exposed functions in the CEGCC static libraries, I stumbled across `libaygshell.a` and looked up what it was used for. [This page](https://social.msdn.microsoft.com/Forums/en-US/c1271d68-2ab5-4e72-8bd3-8c01ca75b5fd/what-is-aygshell-and-why-cant-the-aygshellh-be-found?forum=vssmartdevicesnative) notes that:

> `aygshell` was originally some shell extensions designed for the Windows Mobile environment.
> ...
> As an example the `SHCreateMenuBar()` API which creates menubars at the bottom of your window is part of `aygshell`. Standard Windows CE applications would typically use something like `CommandBar_Create()` instead (which places the menubar at the top of the window).

As my devices are running Windows Mobile, it looks like this library should be used in order to create a native-looking command bar within an application. The actual API documentation for the library seems to be hard to come by, however, so it will probably be worth looking at the headers that are provided with CEGCC.