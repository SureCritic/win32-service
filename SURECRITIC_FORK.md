# SureCritic fork of win32-service

Forked from upstream **win32-service 2.3.2** to fix two Ruby-3 service bugs that
hit the SureCritic M1/Shopkey connector on a fresh boot. All changes are in
`lib/win32/daemon.rb`; nothing else in the gem is touched.

## Why this fork exists

On Ruby 3.3.2 the stock 2.3.2 daemon failed to start the Windows service:

```
Service initializing...
Service initialized
*** failure: Service_Main thread exited abnormally.
```

...and Windows service **recovery never restarted it**, despite restart-on-failure
being configured. Two upstream PRs (never released to a 2.3.x gem) fix it:

1. **chef/win32-service#85 — Ruby 3 slow start.**
   The dispatcher ran via an FFI `CreateThread`. Under Ruby 3's threading model
   `StartServiceCtrlDispatcher` took ~37s, blowing the SCM's 30s start timeout, so
   the service thread was torn down before signaling start → "Service_Main thread
   exited abnormally". Fix: run the dispatcher in a plain Ruby `Thread.new`, and
   wait on `hStartEvent` alone (re-checking `hThread.alive?`). ~37s → <1s.

2. **chef/win32-service#83 — failures didn't trigger recovery (+ boot deadlock).**
   On stop the daemon reported `SERVICE_STOPPED` with win32 exit code `NO_ERROR`
   (0). The SCM treats exit-code-0 as a *clean* stop and does **not** run recovery
   actions, so the abnormal exit above never restarted. Fix: report exit code `1`
   on stop. Also adds a `sleep(0.1)` in the start-wait loop to yield the GVL so the
   dispatcher thread can run (prevents a boot-time starvation/deadlock).

We took only the two `daemon.rb` hunks from each PR. PR #83's separate
`failure_flag` config option was **not** taken — the connector doesn't use it.

## What changed (vs. 2.3.2)

`lib/win32/daemon.rb` only — see `daemon.rb.patch` (apply from the gem root with
`git apply daemon.rb.patch` or `patch -p1 < daemon.rb.patch`):

- `Service_Main` ensure block: `SetTheServiceStatus(SERVICE_STOPPED, NO_ERROR, …)`
  → `SetTheServiceStatus(SERVICE_STOPPED, 1, …)`  (#83)
- Removed the standalone `ThreadProc` FFI function (now inlined as a Ruby thread). (#85)
- `mainloop`: `CreateThread` + `WaitForMultipleObjects` → `Thread.new(Service_Main)`
  running `StartServiceCtrlDispatcher`, then a `WaitForSingleObject(hStartEvent)`
  loop with `hThread.alive?` guard + `sleep(0.1)`.  (#85 + #83)

## Versioning

Bump the gemspec version to **2.3.2.1** (higher than stock 2.3.2 so a stray
`gem install win32-service` can't shadow the fork). Note both fixes in the gem's
own CHANGELOG.

## Build & install on the build box

```
# in the fork checkout
gem build win32-service.gemspec          # -> win32-service-2.3.2.1.gem

# on the Ocran build box (Ruby 3.3.2 x64-mingw-ucrt)
gem uninstall win32-service -v 2.3.2     # remove stock so only the fork remains
gem install win32-service-2.3.2.1.gem
```

Ocran traces `require 'win32/daemon'` and bundles whatever's installed, so the
fork ships in the EXE with no change to the connector source. **Do not** run a
plain `gem install win32-service` on the build box afterward — it pulls stock
2.3.2 from RubyGems and the bug returns silently.

## Verifying the shipped EXE got the fix

After a build, confirm on a test box: the service starts in seconds (no 30s hang),
the log shows `ServiceMain starting...` after `Service initialized`, and killing
the EM thread results in an SCM-driven restart per the configured recovery actions.
