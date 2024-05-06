Notes on reStructuredText - delete this section before submitting
==================================================================

The proposals are submitted in reStructuredText format.  To get inline code, enclose text in double backticks, ``like this``.  To get block code, use a double colon and indent by at least one space

::

 like this
 and

 this too

To get hyperlinks, use backticks, angle brackets, and an underscore `like this <http://www.haskell.org/>`_.


Profiling support for safe FFI imports.
=======================================

.. author:: Andreas Klebinger
.. date-accepted::
.. ticket-url:: implementation of the feature.
.. implemented:: Leave blank. This will be filled in with the first GHC version which
                 implements the described feature.
.. highlight:: haskell
.. header:: This proposal is `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/0>`_.
            **After creating the pull request, edit this file again, update the
            number in the link, and delete this bold sentence.**
.. sectnum::
.. contents::

GHC should empower users by allowing them to mark some or all safe ffi calls as relevant
for profiling instead of never accounting for time spent within them.

Motivation
----------
Every once in a while I hear of a GHC user running into an issue where a profiled programs
runtime far exceeds the time indicated in the time profile.

One possible and the most common reason for such issues is that the program spends much of
it's time executing safe ffi calls. Sadly these do not show up in time profiles generated
by GHC at all. Even if a savy users suspects this to be the issue currently there is no way
to easily check if this is indeed the case.

Therefore I propose the addition of a flag, keyword and a runtime option which allows to include
time spent inside safe ffi calls in a time profile if a user so wishes. This would make
it immediately obvious if the missing time is indeed spent inside safe ffi calls or if
it's due to a more complex problem.


Proposed Change Specification
-----------------------------

FFI imports will get a new property `profiling`.

A function can be **profiled** or **unprofiled**. Their difference is that:

* A profiled functions runtime is included in time profiles.
* A unprofiled functions runtime is ignored in time profiles.

By default safe/interruptible ffi calls will be unprofiled and unsafe ffi calls profiled.
This matches current behaviour of GHC.

Setting profiledness when declaring an FFI import:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When importing functions ghc will permit two new keywords: `profiled` and `unprofiled`.
These can appear after or in place of the safety attribute.

Precisely we extend ffi declarations from:

::
  ...
  fdecl	→	import callconv [safety] impent var :: ftype

to this:

  ...
  fdecl	→	import callconv [safety] [profiling] impent var :: ftype
  profiling → profiled
            | unprofiled

Setting profiledness on a per-module basis:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* `-funprofiled-safe-ffi` will mark safe ffi/interruptible calls as unprofiled.
* `-fprofiled-safe-ffi` will mark safe ffi/interruptible calls as profiled.

Calls will be marked as profiled/unprofiled independent of the their import declaration if
these flags are used. They don't affect unsafe ffi calls at all.

That is if we imports:

::
  -- This import will never be affected as it's an unsafe import
  foreign import stdcall unsafe "c_unsafe"
  c_unsafe :: CInt -> CInt -> CInt -> IO CInt

  -- This import will be treated as profiled under `fprofiled-safe-ffi`
  foreign import ccall safe unprofiled "memcpy"
    memcpy_freeze :: MutableByteArray# s -> MutableByteArray# s -> CSize
           -> IO (Ptr a)

  -- This import will be treated as unprofiled under `funprofiled-safe-ffi`
  foreign import ccall safe profiled "sleep"
    c_sleep :: CUInt -> IO CUInt

  addTriplet :: MutableByteArray# RealWorld -> IO Word8

This avoids the need to annotate all ffi imports manually when trying to find out where time is
spent as it can be enabled on a per package/module basis or even for a full build.

Setting profiledness of safe calls globally at runtime:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A new runtime flag `-ps` will treat all safe ffi calls as profiled independent
of how they have been compiled where possible. This is intended to allow users
to quickly verify if safe ffi calls might be the culprint of a performance issue
before forcing them to recompile.



Proposed Library Change Specification
-------------------------------------

There are no library changes planned in this proposal beyond those required for TH to
support these new declarations.

Examples
--------

::
  {-# LANGUAGE ForeignFunctionInterface #-}

  import Foreign.C

  foreign import ccall safe "sleep" c_simulated_work :: Int -> IO Int

  {-# OPAQUE ffi_call #-}
  ffi_call x = {-# SCC c_ffi #-} c_simulated_work x -- Takes x seconds to run

  {-# OPAQUE some_work #-}
  -- takes about 0.5s on my arm box
  some_work :: Integer -> Integer
  some_work x = {-# SCC haskell_work #-} sum [1..x :: Integer]

  main = {-# SCC main #-} do
      print =<< ffi_call 4
      print $ some_work 15000000

In the above program we will spend 4 seconds doing "work" via an ffi call and about .5 seconds doing work
in haskell code. Currently when trying to profile code like this we get a profile that 100% of the time was
spent under `haskell_work` and a runtime of merely ~0.5 seconds. Despite the real runtime being over 4 seconds.

::
  ...
  total time  =        0.54 secs   (535 ticks @ 1000 us, 1 processor)
  ...

  COST CENTRE  MODULE SRC               %time %alloc

  c_ffi        Main   Main.hs:8:32-49    88.3    0.0
  haskell_work Main   Main.hs:13:40-60   11.6  100.0

But if I use my WIP branch of GHC for the same program I get something far closer to reality:

::
  COST CENTRE  MODULE SRC               %time %alloc

  c_ffi        Main   Main.hs:8:32-49    93.4    0.0
  haskell_work Main   Main.hs:13:40-60    6.5  100.0

Effect and Interactions
-----------------------
Supporting profiling of safe ffi in this manner would easily allow us to attribute
time spent executing such calls to the code paths from which such calls were made.

I'm not aware of any major interactions with other language features.

Costs and Drawbacks
-------------------
I spent some time developing a proof of concept GHC branch I can already use locally before
writing this proposal. Therefore I think the developement cost is limited as is the
maintenance cost.

I don't think these features would add much complexity for people learning about profiling
haskell. Safe ffi calls are already special and something one needs to know about. With these
features this would simply be more explicity.

Backward Compatibility
----------------------
This change would be compatible with all existing code.


Alternatives
------------
There are alternatives to diagnose runtime spent in safe ffi calls like usage of
tools like `perf`. Writing plugins measuring the time before and after ffi calls
or staring at the code base for extended periods of time.

However I think none of those are better than this proposal.

Unresolved Questions
--------------------
Explicitly list any remaining issues that remain in the conceptual design and
specification. Be upfront and trust that the community will help. Please do
not list *implementation* issues.

Hopefully this section will be empty by the time the proposal is brought to
the steering committee.


Implementation Plan
-------------------
I (Andreas Klebinger) am interested in implementing this proposal.

Endorsements
-------------
-
