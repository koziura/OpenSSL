# Building OpenSSL 1.1.0 with Microsoft VS 2015

MCCI needs OpenSSL for a Windows project (that will be cross-platform). A casual search didn't turn up either a good source for cross-platform libraries, which meant we have to build them ourselves. A deeper search found a detailed guide [here](http://developer.covenanteyes.com/building-openssl-for-visual-studio/), and yet the details don't match what I found when I checked out the code; and the post doesn't talk about doing it directly from GitHub (which I wanted to do).

Here's the procedure for building OpenSSL on 64-bit Windows 10, with Visual Studio 2015. Others (July 2019) report that this procedure works with Visual Studio 2017 as well. I've not had a chance to try with Visual Studio 2019.

As this procedure dates from late 2016, you may find that there's a CMake or other, newer, procedure that's more suitable.

1. If you don't have it, please install git bash from [git-scm.com](https://git-scm.com/downloads).

2. Start a git bash window.

   > In the following, I'll use lines beginning with `{dir} $` to designate input to a bash Window. We'll also have to use a `cmd.exe` Window for much of the build, and those lines will be marked `{dir}>`, just as on Windows..

3. Create a directory for your work.

    ```bash
    ~ $ mkdir c:/work
    ~ $ cd c:/work
    /c/work $
    ```

4. Clone the SSL repository from github.

    ```bash
    /c/work $ git clone https://github.com/openssl/openssl.git
    ```

5. At this point, we have the bleeding edge checked out in your sandbox. We don't want that for this project. Update the repository checkout to the 1.1 stable branch.

    ```bash
    /c/work $ cd openssl
    /c/work/openssl $ git checkout --track origin/OpenSSL_1_1_0-stable
    ```

   This is when we looked into `NOTES.WIN` and `NOTES.PERL`.

6. Get ActiveState Perl from [https://www.activestate.com/activeperl](https://www.activestate.com/activeperl). My normal style is to download the file and then run the install as a second step. I got the 64-bit version.

7. Install ActiveState Perl.  Select a typical install, but don't let it change `PATH` or add automatic execution of "`.pl`" extensions (uncheck the boxes on the install menu). (Well, you can let it change them if _you_ want; but on my system I don't normally use Perl, and I don't want any more random things on my path,)

8. Use Visual Studio to open a 64-bit compile window. In my case, I clicked the Windows button, then typed `VS2015 x64`, and then clicked on `VS2015 x64 Native Tools Command Prompt`.

9. In that window, change directory to the repo created above.

    ```cmd
    C> cd c:\work\openssl
    ```

10. Add `c:\perl64\bin` to your path.

    ```cmd
     C:\work\openssl> path=%path%;c:\perl64\bin
    ```

11. We need ``Text::Template``, and with ActivePerl, we need to use the Package Manager.

    * Start Perl Package manager (press the Windows key and type `perl`, and Windows will offer you "Perl Package Manger").
    * On the menu bar, select "View>All packages".
    * Search for "Text-template" (**not** "Text::Template").  Select it; I used version 1.46. Be careful, as there are several packages starting with "Text-template"; the one we want is named exactly "Text-template".
    * On the menu bar, select "Action->Install". This is sort of like `git add` -- it stages the package for installation, but doesn't install it yet
    * On the menu bar, select "File>Run Marked Actions"

12. Download and install `nasm` from [nasm.us][nasm.us].  I installed it at `c:\tools\nasm`; and I manually added it to my path using:

    ```cmd
    C:\work\openssl> path=%path%;c:\tools\nasm
    ```

13. Choose where you're going to install OpenSSL. I did not want to add this in `C:\Program Files`, which is the default. I've run into lots of problems in the past trying to mix debug and release builds with Visual Studio, and it seems even more exciting with VS 2015; so I want to do side-by-side installs. I decided to put the release build in `{destdir}\vc-win64a`, and the debug build in `{destdir}\vc-win64a-debug`. I want to put the configuration files in `{destdir}\SSL`.

14. I didn't want assembly language (even though I got `nasm` above -- the build seems to require it). So I used the following configuration.

    ```cmd
    C:\Work\openssl> perl Configure VC-WIN64A --prefix=c:\tools\OpenSSL\vc-win64a --openssldir=c:\tools\OpenSSL\SSL no-asm
    ```

    > **Note:** while `Configure` is running, it pops up a window suggesting that `nmake` is not in the path, and offering to download `dmake`. Ignore this message! If you are running a VS2015 command window, as suggested above, `nmake` is certainly in your path.

    Several variations on this Configure command are possible.

    * You can select different targets by changing `VC-WIN64A` whatever target you want to build for. I have heard that `VC-WIN32` works for 32-bit targets. But in that case, don't forget that you need to launch a different kind of command window from Visual Studio, targeting 32-bit x86).

    * I am told that the `no-shared` keyword will suppress builds of DLLs.

15. After configuring, do this to build:

    ```cmd
    c:\work\openssl> nmake
    ```

    On my system, that took about 3 minutes.  There were some warnings, but nothing that stopped the compile.

16. The instructions suggest doing a test using `nmake test`, so let's do that.

    ```cmd
    c:\work\openssl> nmake test
    ```

    That also ran for about 3 minutes. During one of the tests, Windows firewall pops up a box about network access. I simply denied permission. The test passed anyway.

17. To install:

    ```cmd
    c:\work\openssl> nmake install
    ```

18. We need to repeat this for a debug build. For sanity, we start by once again cloning the repo. Go back to the github shell (do you remember the github shell from above?) and clone the local repo to another.

    ```bash
    /c/work/openssl $ cd ..
    /c/work $ git clone openssl openssl-debug
    /c/work $ cd openssl-debug
    /c/work/openssl-debug $
    ```

    It might also be possible to say `nmake clean`  and just rebuild; but the instructions in `INSTALL` seem to indicate that if you change the configuration significantly, you need a new sandbox.

19. Again, we need to configure, so back to the `cmd.exe` window. This time, we need to change the prefix so that the debug components get installed at a different location.

    ```cmd
    c:\work\openssl> cd ..\openssl-debug
    c:\work\openssl-debug> perl Configure VC-WIN64A --debug --prefix=c:\tools\OpenSSL\vc-win64a-dbg --openssldir=c:\tools\OpenSSL\SSL no-asm
    c:\work\openssl-debug> nmake
    c:\work\openssl-debug> nmake test
    c:\work\openssl-debug> nmake install
    ```

    Apart from adding the `--debug` switch and changing the target directory, you should use the same options to `Configure` as were used for the non-debug configuration.

20. As a sanity check, we want to try running `openssl`. In the `cmd.exe` window, we'll enter `c:\tools\OpenSSL\vc-win64a\bin\openssl.exe help`. We should get the usual output:

    ```cmd
    c:\work\openssl-debug> c:\tools\OpenSSL\vc-win64a\openssl.exe help
    Standard commands
    asn1parse         ca                ciphers           cms
    crl               crl2pkcs7         dgst              dhparam
    dsa               dsaparam          ec                ecparam
    enc               engine            errstr            exit
    gendsa            genpkey           genrsa            help
    list              nseq              ocsp              passwd
    pkcs12            pkcs7             pkcs8             pkey
    ...
    ```

That's all there is to it.

With this procedure you will, unfortunately, get two copies of *everything* (including the HTML stuff), not just two copies of the executables. I leave it as an exercise for the interested reader to determine how to get only the binaries installed side-by-side, with a common "shared" directory for the html and include fies.

[nasm.us]: https://nasm.us

## Miscellany

Written with [StackEdit](https://stackedit.io/) and then a lot of help from @clintel's post [Fenced code blocks inside ordered and unordered lists](https://gist.github.com/clintel/1155906). Polished with David Anson's `markdownlint` plugin ([davidanson.vscode-markdownlint](https://github.com/DavidAnson/vscode-markdownlint)) for VS Code.

Updated 2019-07-20 to fix formatting and typos.

Copoyright (C) 2016, 2019 [MCCI Corporation](https://mcci.com). Written by Terry Moore, MCCI. License: [CC-BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

## #ifndef ETIMEDOUT
## #define ETIMEDOUT 9938
## #endif
