# TODOs

## 1. Modularize the `freeswitch` Nixpkgs package

The NixOS Discourse post [Guidelines / best practices to package software with plugins / modules?](https://discourse.nixos.org/t/guidelines-best-practices-to-package-software-with-plugins-modules/51462) goes into it deeper, but the gist is that

+ FreeSWITCH can be compiled without any modules

  > ASIDE
  > Technically, the only dependencies needed are the C library files of `sofia-sip`, `spandsp`,and `curl`, but it is easier to just their respective wrapper-modules.
  >
  > The other 2 modules that are not necessary for FreeSWITCH to compile, but it will be hard (impossible?) to handle the FreeSWITCH instance:
  >
  > + `mod_event_socket`
  >   `fs_cli` needs this to connect to the running FreeSWITCH instance
  >
  > + `mod_commands`
  >   Without  this,  there  are  only  some  very   basic
  >   commands available on the `freeswitch` console.  Not
  >   even sure if such a deployment could even be able to
  >   do anything functional?

+ The **in-tree** modules can be compiled with FreeSWITCH itself and then by themselves afterwards, but with Nix this is won't work because the Nix store is immutable. To add modules later, one will have to recompile the entire package again with new inputs.

  > WAIT, is that the case if a `withModule` system is established? The modules will be compiled individually, but a new wrapper FreeSWITCH executable will have to be created. (Or not; wasn't able to figure out how to compile the in-tree modules by themselves in a Nix derivation or how they would be linked to the built FreeSWITCH package...)

+ The **off-tree** modules reside in different GitHub repos, but they could still be compiled as in-tree modules using a special `modules.conf` syntax. (See https://github.com/freeswitch/mod_kazoo)

### 1.1 How to are Nix package plugin / module systems implemented?

Using the `passthru` attribute in `mkDerivation`; usually `withPlugins` or `withModules`:

+ https://discourse.nixos.org/t/packaging-custom-assets-for-another-program-package/25440/2
+ https://stackoverflow.com/questions/38282478/how-to-package-plugins-for-an-application-for-nixos

Look for `passthru`'s and `withPackage =` or `with.* =` strings in Nixpkgs. Most I saw used `makeWrapper` (or its sibling, `wrapProgram`) to bundle / link the plugins with the main executable, but not sure how it would work with FreeSWITCH.

### 1.2 How to compile in-tree modules by themselves?

1. **Figure out how to do it without Nix**

   ```
    $ cd src/mod/event_handlers/mod_erlang_event/
    $ make mod_erlang_event

    gcc     mod_erlang_event.c   -o mod_erlang_event
    mod_erlang_event.c:36:10: fatal error: switch.h: No such file or directory
    36 | #include <switch.h>
        |          ^~~~~~~~~~
    compilation terminated.
    make: *** [<builtin>: mod_erlang_event] Error 1
   ```

2. **Do it with Nix**

   I tried it with Nix first, but only got until this part as I know nothing about Autotools:

   ```
   nix-repl> :trace-enable true

   nix-repl> mee = { stdenv, fetchFromGitHub, erlang, pkg-config, autoreconfHook }: stdenv.mkDerivation rec { freeswitch_module_name = "mod_erlang_event"; pname = "freeswitch-${freeswitch_module_name}"; version = "1.10.12"; src = fetchFromGitHub { owner = "signalwire"; repo = "freeswitch"; rev = "v1.10.12"; sha256 = "sha256-uOO+TpKjJkdjEp4nHzxcHtZOXqXzpkIF3dno1AX17d8="; }; sourceRoot = "${src.name}/src/mod/event_handlers/${freeswitch_module_name}"; nativeBuildInputs = [ erlang pkg-config autoreconfHook]; buildInputs = [ erlang ]; }

   nix-repl> :b (cloned_pkgs.callPackage mee {})
   error: builder for '/nix/store/cz6285jg6s6a75z03x3zablwl7i2ywyl-freeswitch-mod_erlang_event-1.10.12.drv' failed with exit code 1;
       last 7 log lines:
       > Running phase: unpackPhase
       > unpacking source archive /nix/store/l5apfs8adkaigpyvmglzck009g3dag0p-source
       > source root is source/src/mod/event_handlers/mod_erlang_event
       > Running phase: patchPhase
       > Running phase: autoreconfPhase
       > autoreconf: export WARNINGS=
       > autoreconf: error: 'configure.ac' is required
       For full logs, run 'nix log /nix/store/cz6285jg6s6a75z03x3zablwl7i2ywyl-freeswitch-mod_erlang_event-1.10.12.drv'.
   [4 built (16 failed), 400 copied (1604.2 MiB), 366.3 MiB DL]
   ```

## 2. Vanilla configs pulled in by default

Not sure if TODO 1. above would help solve this, but no matter what `modules.conf` file is generated, the `autoload_configs` folder will hold XML configs for a lot of other modules and `modules.conf.xml` will show a lot of modules enabled that were not specified in the input `modules.conf` (I presume there is a one-to-one correspondence between the XML configs present and the modules enable in `modules.conf.in`).

Not sure about **off-tree** modules, but **in-tree** ones have their on sample config XML in their own folders, so if TODO 1. works out, they could be pulled in individually. (But then, if a module is needed, it will probably be rare not to modify the config. One exception that comes to mind is `mod_cdr_csv, if I remember the TR2 settings correctly.)

## 3. Declarative configuration

Instead of editing XML files, the modules and FreeSWITCH itself could expose attributes to generate the XML files themselves. Not sure how this is done in other packages.