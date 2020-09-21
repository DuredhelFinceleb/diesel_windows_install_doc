# Diesel install on Windows with support for SQLite, MySQL and Postgresql

## Introduction

I had a lot of troubles installing Diesel on Windows with support for all supported backends, SQLite, MySQL & Postgresql.
I have read many many web sites, blogs, forums, FAQs, projects issues, etc. to get this done and when I thought it was over I have finally made it.
I hope this helps.

I assume you know about chocolatey and know how to install it.
Likewise, I assume you know how to define/modify env vars in Windows, I know that when you come from Unix it can be confusing.

## OS version

My dev machine is a Windows 10 Pro 64 bits with the 2020 May update.
It's a Hyper-V VM with 8 vCores & 16GB RAM.

## Prerequisites

### VCRedist

Not sure this one is actually needed...
Using chocolatey
```
choco install vcredist-all
```

### Visual studio community

I've installed VSC2019 using chocolatey
```
choco install visualstudio2019community
```
Then add this to your PATH:
```
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.26.28801\bin\Hostx64\x64
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\MSBuild\Current\Bin\amd64
```
These are the paths to 'nmake' and 'msbuild'.
In my case, it was needed by VCPKG when compiling libpq

Do not forget to close your terminal & start a new one for the new paths to take effect.

### Python 3

Again, not sure it's required for this, but in my case, Rust would not compile some stuff without this so...
Still using chocolatey, I got Python 3.8
```
choco install python
```

### Rust 

A working Rust installation of course :-)
Mine was 1.46.0

### VCPKG

This is required to install all the libs.
We assume it will be installed into C:\src

```
c:
cd \src
git clone https://github.com/microsoft/vcpkg
.\vcpkg\bootstrap-vcpkg.bat
.\vcpkg\vcpkg integrate install
```

The bootstrap step can be quite long especially when compiling the ICU thing.
Don't loose patience it will end eventually.
Long story short, the more cores (and RAM) the quicker it will be, so if you're using a VM, try to increase its specs if possible (8 vCores & 16GB RAM is OK, 2 vCores & 4GB RAM is really painful)

The integrate step requires Administrator privileges

Add this env var:
```
VCPKG_ROOT=C:\src\vcpkg
```
Do not forget to close your terminal & start a new one for the env var to take effect.

## Install libs for Mysql, Sqlite3 & PostgreSQL

Run this
```
.\vcpkg\vcpkg --triplet x64-windows-static-md install libmysql sqlite3 libpq
```

## Compilation, first try

In theory, you should be able to build diesel_cli
```
cargo install diesel_cli
```

If it works for you, congratulations you're finished! If not, keep on reading...

In my case, it failed on building libpq complaining a lot about SSL stuff.
After some research, it turns out that the best way to get around this is to get libpq directly the latest PostgreSQL package.
However, you can't just add the path to that PostgreSQL package lib folder and be done with it: cargo will compile PostgreSQL support just fine and will crash on compiling MySQL support because... of SSL stuff! 
Apparently it seems that both libraries require different version of SSL libs...
To be honest, I'm not sure what is really going on here.
Still there's a way to solve this! Read on.

## Install PostgreSQL 

I've downloaded from PostgreSQL website & installed v12.4.1 64 bits
Just follow the steps. I've used the default paths proposed by the installer.

When it's done, have a look at all the '*.lib' files into the folder 'C:\Program Files\PostgreSQL\12\lib'

We are going to going to install via VCPKG some of these libraries and to copy others into VCPKG installed libs folder.
This might be overkill, but it worked for me...

## Install via VCPKG PostgreSQL libs

Run this to install iConv, GetText, XML2 & XSLT

```
.\vcpkg\vcpkg --triplet x64-windows-static-md install libiconv gettext libxml2 libxslt  
```

## Copy PostgreSQL libs to VCPKG

We are going to rename the libpq compiled by VCPKG.
Then we will copy from PostgreSQL libs folder libpq and all the libs that cannot be found in VCPKG repository.

```
cd \src\vcpkg\installed\x64-windows-static-md\lib
ren libpq.lib libpq.lib.vcpkg
copy "\Program Files\PostgreSQL\12\lib\libpq.lib" .
copy "\Program Files\PostgreSQL\12\lib\postgres.lib" .
copy "\Program Files\PostgreSQL\12\lib\wx*.lib" .
```

## Install & use diesel_cli!

Now you can finally run "cargo install diesel_cli"
Indeed, at this point, it finally compiled for me! :-)

Once it's compiled, you still have to add PostgreSQL bin & lib paths to your PATH env var.
This is supposed to be done before compiling diesel_cli, but this is a problem since cargo will try to get its libs first from the PostgreSQL libs folder including those which make MySQL compilation hang.

Anyways, add those two paths to your PATH:
```
C:\Program Files\PostgreSQL\12\bin
C:\Program Files\PostgreSQL\12\lib
```

Maybe the same thing should be done for MySQL bin & lib folders, but since I don't have it installed, I cannot say.

You can now enjoy Diesel!
For example, test it by running the "Getting started" guide.

## One last final tip

When testing via the "Getting started" guide, I did this with Powershell to create empty files (equivalent to Unix "touch") 
```
echo $null >> .env
```
Well, don't do this.
The file produced is empty all right, but it is not UTF-8 and diesel_cli won't read it.
You will loose lots of time trying to understand why your perfectly good file is not read.
I guess using Notepad++ or VSCode is still the better option.