# FieldWorks flatpak packaging

This repository is intended to hold the draft flatpak specification while it's
being prepared. A number of files should be removed from this repository, and
a significant history re-write, before performing a pull request to submit the 
flatpak specification to flathub.

## Multiple branches

(Probably not possible in the initial pull request submission to flathub, but)
flatpak allows multiple branches to be used for a software application. So we
can have a stable branch that provides the latest stable FieldWorks version
(which may go, for example, from 9.0.14 to 9.0.15 to 9.1.10 to 10.0.3), a 9.0
branch that provides the latest 9.0 version (which may go, for example, from
9.0.14 to 9.0.15, but never to 9.1.10), and a 9.1 branch that provides the
latest 9.1 version (eg 9.1.10, 9.1.11, but never 9.2.0 or 10.0.0).

Using multiple branches will allow users to choose whether they want to use
FW 9.0 or FW 9.1, without being required to use the latest stable (as is the
normal process with the apt/deb repository).

This won't be applicable in the original pull request submission to flathub for
the flatpak packaging specification, but is something we can follow up with
later.

## Building

### Dependencies

git clone https://github.com/sillsdev/flathub --branch org.sil.FieldWorks
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install flathub org.gnome.Sdk//3.36
flatpak install flathub org.gnome.Platform//3.36
flatpak update

Install tools:

```bash
sudo apt install flatpak-builder
```

#### Optional dependencies 

Clone projects to help generate dependency download lists:
```bash
git clone https://github.com/silnrsi/encoding-converters-core.git --branch master
git clone https://github.com/sillsdev/flexbridge.git --branch develop
git clone https://github.com/sillsdev/FieldWorks.git --branch support/9.0
```

Install tool flatpak-dotnet-generator.py and dependency dotnet5 sdk:
```bash
flatpak install flathub org.freedesktop.Sdk.Extension.dotnet5 org.freedesktop.Sdk/x86_64/20.08
git clone https://github.com/flatpak/flatpak-builder-tools.git
# Until feature/multiple-csproj PR #206 is merged:
cd flatpak-builder-tools
git remote add marksvc https://github.com/marksvc/flatpak-builder-tools.git
git fetch --all && git checkout marksvc/feature/multiple-csproj
```

Install xonsh to run some dependency-url-generating scripts.
```bash
sudo apt install xonsh
```

### Build

```bash
./build
```

Note that your first build will take time to download and build all dependencies.

Build up to the fieldworks module, apply the sources for fieldworks, but open a
shell before building fieldworks by appending `--build-shell=fieldworks`

Also provide access to the debugging symbols package, and clean up build
results better, by using the build script: `./build`

## Testing

Run FieldWorks in the result (optionally with additional debugging):

```bash
flatpak run --devel --env=FW_DEBUG=true org.sil.FieldWorks
```

Open a shell inside the FieldWorks flatpak instead of running FieldWorks:

```bash
flatpak run --devel --env=FW_DEBUG=true --command=bash org.sil.FieldWorks
```

### Fetch flatpak to test without building it

- View the list of FW flatpak builds at
  https://github.com/sillsdev/flathub/actions?query=branch%3Aorg.sil.FieldWorks .
- Open the most recent successful build.
- Under artifacts, download 'fw-flatpak-bundle'. You need to be logged into
  github to do this until we adjust permissions.
- Unzip and install the downloaded FW: 
  ```bash
  unzip fw-flatpak-bundle.zip
  for f in fieldworks*.flatpak; do flatpak --user install --assumeyes $f; done
  ```
- Update any other flatpaks in case helpful for consistency of testing: `flatpak update`
- Run flatpak FW: `flatpak run org.sil.FieldWorks`

## Debugging flatpak FieldWorks

Switch FieldWorks to the feature/flatpak branch:
`cd ~/fwrepo/fw && git fetch && git checkout feature/flatpak`

Make an expected directory if you haven't built FW on the machine yet:
`mkdir -p  Output_x86_64/Debug`

Open the FieldWorks workspace in VSCode. 

Note: Debugging now appears to be broken for managed debugging. And I can only get
unmanaged debugging to work on one machine and not another. :-/

### Managed debugging

Run FieldWorks from the flatpak by running this command (as copied from FieldWorks launch.json): 

```bash
flatpak run --devel --env=FW_DEBUG=true --env=FW_MONO_OPTS=--debugger-agent=address=127.0.0.1:55555,transport=dt_socket,server=y,suspend=n org.sil.FieldWorks
```

Debug target "Attach to local (such as flatpak)" in VSCode. This was working in
2021-04 but seems to have broken since?

### Unmanaged debugging

Install VSCode gdb debugging extension: `code --install-extension webfreak.debug`

Run FieldWorks from the flatpak by running this command (as copied from FieldWorks launch.json): 

```bash
flatpak run --devel --env=FW_DEBUG=true --env=FW_COMMAND_PREFIX="gdbserver 127.0.0.1:9999" --env=FW_MONO_COMMAND=/app/bin/mono-sgen org.sil.FieldWorks
```

Debug target "Attach to local gdbserver" in VSCode.

You can check if you have org.sil.FieldWorks.Debug installed to get debugging
symbols in /app/lib/debug in the flatpak by running `flatpak list --all | cat`.

### Manual unmanaged debugging

Open the FieldWorks flatpak environment to bash, and write a .gdbinit file:

```bash
host: flatpak run --devel --env=FW_DEBUG=true --command=bash  org.sil.FieldWorks
```

Create file in flatpak of ~/.gdbinit containing: (Ok this may not at all be necessary)
```
handle SIGXCPU SIG33 SIG35 SIG36 SIG37 SIG38 SIGPWR SIG38 nostop noprint

define mono_backtrace
 select-frame 0
 set $i = 0
 while ($i < $arg0)
   set $foo = (char*) mono_pmip ($pc)
   if ($foo)
     printf "#%d %p in %s\n", $i, $pc, $foo
   else
     frame
   end
   up-silently
   set $i = $i + 1
 end
end

define mono_stack
 set $mono_thread = mono_thread_current ()
 if ($mono_thread == 0x00)
   printf "No mono thread associated with this thread\n"
 else
   set $ucp = malloc (sizeof (ucontext_t))
   call (void) getcontext ($ucp)
   call (void) mono_print_thread_dump ($ucp)
   call (void) free ($ucp)
 end
end

add-auto-load-safe-path /opt/mono5-sil/bin/mono-sgen-gdb.py
add-auto-load-safe-path /app/bin/mono-sgen-gdb.py
```

Then either start debugging by launching mono from gdb, or using gdbserver:

#### In flatpak, directly with GDB

```bash
flatpak: FW_COMMAND_PREFIX=gdb FW_MONO_COMMAND=/app/bin/mono-sgen FW_MONO_OPTS='--' fieldworks-flex
flatpak: (gdb) b UniscribeEngine::FindBreakPoint
flatpak: (gdb) run --debug /app/lib/fieldworks/FieldWorks.exe -app Flex
```

#### With gdbserver

Start gdbserver in flatpak, similarly to how it will listen to vscode, but then run the gdb client in the flatpak:

```bash
flatpak: FW_COMMAND_PREFIX="gdbserver 127.0.0.1:9999" FW_MONO_COMMAND=/app/bin/mono-sgen nohup fieldworks-flex &
flatpak: gdb /app/bin/mono-sgen
flatpak: (gdb) target remote 127.0.0.1:9999
flatpak: (gdb) b UniscribeEngine::FindBreakPoint
flatpak: (gdb) c
```

#### With gdb, across flatak boundary

Note: Currently this has an error.

```bash
host: flatpak run --devel --env=FW_DEBUG=true org.sil.FieldWorks
host: pgrep --full FieldWorks.exe # find PID
host: gdb /opt/mono5-sil/bin/mono-sgen
host: (gdb) attach PID
```

##### Example with gedit

```bash
host: flatpak run --devel org.gnome.gedit
host: ps faxww|grep gedit # Identify the PID for the right gedit.
```
Either
```bash
host: gdb
host: (gdb) attach PID
```
This gives errors like

> warning: "target:/app/bin/gedit": could not open as an executable file: Operation not permitted.                                                            
> warning: `target:/app/bin/gedit': can't open to read symbols: Operation not permitted.
> warning: Could not load vsyscall page because no executable was specified                                                                                    
> warning: Target and debugger are in different PID namespaces; thread lists and other data are likely unreliable.  Connect to gdbserver inside the container.

Or
```bash
host: gdb .local/share/flatpak/app/org.gnome.gedit/x86_64/stable/cd0da06d4896fabaa97f2fdecae39b5a5efe8461b0784f2ff2b17610994120af/files/bin/gedit
host: (gdb) attach PID
```
This gives errors like

> Error while mapping shared library sections:
> Could not open `target:/app/lib/gedit/libgedit-40.0.so' as an executable file: Operation not permitted            
> ...
> warning: Unable to find dynamic linker breakpoint function.
> GDB will be unable to debug shared library initializers
> and track explicitly loaded dynamic code.
> warning: Target and debugger are in different PID namespaces; thread lists and other data are likely unreliable.  Connect to gdbserver inside the container.

So this approach may not work for FW either.

#### With gdbserver, across flatpak boundary

This does work, in Ubuntu 20.04.

```bash
host: flatpak run --devel --env=FW_DEBUG=true --command=bash org.sil.FieldWorks
flatpak: FW_COMMAND_PREFIX="gdbserver 127.0.0.1:9999" FW_MONO_COMMAND=/app/bin/mono-sgen fieldworks-flex
host: gdb 
host: (gdb) target remote 127.0.0.1:9999
host: (gdb) b UniscribeEngine::FindBreakPoint
host: (gdb) c
```

##### Example with gedit

This does work, in Ubuntu 20.04.

```bash
host: flatpak run --devel --command=bash  --share=network org.gnome.gedit 
flatpak: gdbserver 127.0.0.1:9999 gedit
host: gdb
host: (gdb) target remote 127.0.0.1:9999
```

## Validate appdata

```bash
flatpak install flathub org.freedesktop.appstream-glib
flatpak run org.freedesktop.appstream-glib validate .../fw/DistFiles/Linux/fieldworks-applications.desktop.appdata.xml
```

## Pushing

Pushing to FieldWorks.git branch feature/flatpak, bypassing gerrit, is done with:
```bash
git pull --rebase
git push --force origin feature/flatpak:refs/heads/feature/flatpak
```
