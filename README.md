# build-FreeFileSync-on-raspberry-pi
FreeFileSync is a great open source file synchronization tool. However, despite its open source nature, there is no build instruction for it. 

This repo records my own way of building FreeFileSync on Raspberry Pi with Raspbian Feb 2020 installed. 

## 0. Download and extract the source code

As of writing, the latest version of FreeFileSync is 10.22 and it can be downloaded from: 

https://freefilesync.org/download/FreeFileSync_10.22_Source.zip. 

Note wget **DOES NOT** work with this URL on the first try. You can either manually download it, or try wget a second time to get the source code downloaded.

## 1. Install dependencies
The following dependencies need to be installed to make code compile.
- libgtk2.0-dev
- libxtst-dev

```
sudo apt-get update
sudo apt-get install ibgtk2.0-dev
sudo apt-get install libxtst-dev
```

## 2. Compile dependencies

The following dependencies could not be installed from `apt-get` command and has to be compiled from source code.

### 2.1 gcc

FreeFileSync requires a c++ compiler that supports c++2a.
The default version of gcc with Raspbian February 2020 is 8.3.0 and does not work.

I followed the instruction at: https://www.raspberrypi.org/forums/viewtopic.php?t=239609 to build and install the gcc 9.3.0 with minor modifications. See [build_gcc.sh](build_gcc.sh) for the script with only c/c++ languages enabled.

If you follow the steps correctly, you should see the new verison of g++ using "g++ -v": 
```
pi@raspberrypi:~/Downloads/FreeFileSync_10.22/FreeFileSync/Source $ g++ --version
g++ (GCC) 9.3.0

```

### 2.2 openssl

FreeFileSYnc 10.22 requires openssl version to be `0x1010105fL` above, otherwise you got error:
```
../../zen/open_ssl.cpp:21:38: error: static assertion failed: OpenSSL version too old
   21 | static_assert(OPENSSL_VERSION_NUMBER >= 0x1010105fL, "OpenSSL version too old");
```

Tried openssl-1.1.1f but got another error:
```
../../zen/open_ssl.cpp:576:68: error: 'SSL_R_UNEXPECTED_EOF_WHILE_READING' was not declared in this scope
  576 |             if (sslError == SSL_ERROR_SSL && ERR_GET_REASON(ec) == SSL_R_UNEXPECTED_EOF_WHILE_READING) //EOF: only expected for HTTP/1.0
```

So has to go with the latest openssl 3 from github:
```
git clone git://git.openssl.org/openssl.git --depth 1
cd openssl
./config
make
sudo make install
```
### 2.3 libssh2
System provided version 1.8.0-2.1 does not work with macro not found:
```LIBSSH2_ERROR_CHANNEL_WINDOW_FULL```

Has to build from source:
```
wget https://www.libssh2.org/download/libssh2-1.9.0.tar.gz
tar xvf libssh2-1.9.0.tar.gz
cd libssh2-1.9.0/
mkdir build
cd build/
../configure
make
sudo make install
```

### 2.4 libcurl
Could not get any package from `apt-get` working so had to build from source.
```
wget https://curl.haxx.se/download/curl-7.69.1.zip
unzip curl-7.69.1.zip
cd curl-7.69.1/
mkdir build
cd build/
../configure
make
sudo make install
```

### 2.5 wxWidgets
The latest version compiles without problem:
```
wget https://github.com/wxWidgets/wxWidgets/releases/download/v3.1.3/wxWidgets-3.1.3.tar.bz2
tar xvf wxWidgets-3.1.3.tar.bz2
cd wxWidgets-3.1.3/
mkdir gtk-build
cd gtk-build/
../configure --disable-shared --enable-unicode
make
sudo make install
```

## 4. Tweak the code:

Even with the latest dependencies, there is still some comiplation errors which needs to tweat the code to fix. 

### 4.1 FreeFileSync/Source/afs/sftp.cpp

add at line 58
```
#define MAX_SFTP_OUTGOING_SIZE 30000
#define MAX_SFTP_READ_SIZE 30000
```
### 4.2 zen/open_ssl.cpp
Change some function definitions to avoid compliation error with function not found:
```
180c180
< using EvpToBioFunc = int (*)(BIO* bio, EVP_PKEY* evp);
---
> using EvpToBioFunc = int (*)(BIO* bio, const EVP_PKEY* evp);
237c237
< int PEM_write_bio_PrivateKey2(BIO* bio, EVP_PKEY* key)
---
> int PEM_write_bio_PrivateKey2(BIO* bio, const EVP_PKEY* key)
```

### 4.3 [Optional] FreeFileSyc/Source/Makefile
To make the exectuable easier to run, add after line 28:
```
LINKFLAGS += -Wl,-rpath -Wl,\$$ORIGIN
```

## 6. Compile

Run "make" in folder FreeFileSync_10.22_Source/FreeFileSync/Source. 

The binary should be waiting for you in FreeFileSync_10.22_Source/FreeFileSync/Build/Bin. 

## 7. zip all dependencies
After the executable is binary, copy all dependencies libraries to the same folder as the binary, the copy `Build/Resources` folder, zip them in a file.

Then end zip file should look like this:
```
Archive:  FreeFileSync_10.22_armv7l.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2020-04-27 22:38   Bin/
  9074216  2020-04-27 22:35   Bin/FreeFileSync_armv7l
   560412  2020-04-17 13:40   Bin/libssl.so.3
 17574572  2020-04-10 15:12   Bin/libstdc++.so.6
   492832  2020-04-17 14:14   Bin/libcurl.so.4
  7543332  2020-04-10 15:10   Bin/libgcc_s.so.1
  3476740  2020-04-17 13:40   Bin/libcrypto.so.3
   961484  2020-04-17 14:07   Bin/libssh2.so.1
      349  2020-03-18 21:57   FreeFileSync.desktop
        0  2020-03-18 21:57   Resources/
      234  2020-03-18 21:57   Resources/Gtk3Styles.css
    12402  2020-03-18 21:57   Resources/RealTimeSync.png
   182060  2020-03-18 21:57   Resources/harp.wav
    87678  2020-03-18 21:57   Resources/bell2.wav
    13232  2020-03-18 21:57   Resources/FreeFileSync.png
   223687  2020-03-18 21:57   Resources/cacert.pem
      440  2020-03-18 21:57   Resources/Gtk2Styles.rc
    67340  2020-03-18 21:57   Resources/ding.wav
   143370  2020-03-18 21:57   Resources/bell.wav
   230274  2020-03-18 21:57   Resources/gong.wav
   388959  2020-03-18 21:57   Resources/Icons.zip
   522150  2020-03-18 21:57   Resources/Languages.zip
    55006  2020-03-18 21:57   Resources/notify.wav

```

Now the zip file should contain all the dependencies and the binary `Bin/FreeFileSync_armv7l` is able to run on a new raspberry pi with Raspbian OS directly.
