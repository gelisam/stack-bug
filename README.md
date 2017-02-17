# Summary

`stack ghci` fails, but if we edit `hs-source-dirs` to `.` instead of `./.`, it succeeds. This shouldn't make a difference, should it?

## Details

In the folder which contains the `mypackage` folder, type `stack ghci`. We get errors about `MyModule`:

    $ stack ghci
    [...]
    GHCi, version 8.0.1: http://www.haskell.org/ghc/  :? for help

    <no location info>: error: module ‘MyModule’ is a package module
    Failed, modules loaded: none.

    <no location info>: error: module ‘MyModule’ is a package module
    Failed, modules loaded: none.

    <no location info>: error:
	Could not find module ‘MyModule’
	Perhaps you meant Module (needs flag -package-key ghc-8.0.1)
    Loaded GHCi configuration from /private/var/folders/78/00jw64hs67g4t8rmc3pkv15m0000gn/T/ghci12969/ghci-script

If we edit `mypackage/mypackage.cabal` so that `hs-source-dirs` is `.` instead of `./.`, the problem disappears:

    $ stack ghci
    [...]
    GHCi, version 8.0.1: http://www.haskell.org/ghc/  :? for help
    [1 of 1] Compiling MyModule         ( /Users/gelisam/working/haskell/stack-bug/mypackage/MyModule.hs, interpreted )
    Ok, modules loaded: MyModule.
    [2 of 2] Compiling Main             ( /Users/gelisam/working/haskell/stack-bug/mypackage/myexecutable/Main.hs, interpreted )
    Ok, modules loaded: MyModule, Main.
    Loaded GHCi configuration from /private/var/folders/78/00jw64hs67g4t8rmc3pkv15m0000gn/T/ghci13077/ghci-script

## Analysis

So [apparently](https://github.com/sol/hpack/issues/119) Cabal ignores `.` entries, but in the case of `hs-source-dirs`, the [default value](https://www.haskell.org/cabal/users-guide/developing-packages.html#build-information) is `.` anyway, so this doesn't change anything. Indeed, removing `hs-source-dirs` from the cabal file entirely has the same effect as replacing the `./.` with `.`.

But the problem isn't Cabal, it's the flags which stack passes to ghci. With `./.`:

    $ stack -v ghci
    [...]
    [debug] Run process: ghc --interactive
      [...]
      -i
      -i/[...]/mypackage/myexecutable
      [...]

With `.` or without `hs-source-dirs`:

    $ stack -v ghci
    [...]
    [debug] Run process: ghc --interactive
      [...]
      -i
      -i/[...]/mypackage
      -i/[...]/mypackage/myexecutable
      [...]

With `./.`, the folder containing `MyModule` is not listed as an include path, which is why `:add MyModule` complains that it cannot find the file (the "package module" in the error message is a red herring, it just means that the source file was not found).

    $ stack exec -- ghci -i
    Prelude> :add MyModule
    <no location info>: error: module ‘MyModule’ is a package module

    $ stack exec -- ghci -i -imypackage
    Prelude> :add MyModule
    [1 of 1] Compiling MyModule         ( mypackage/MyModule.hs, interpreted )
