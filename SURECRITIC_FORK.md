# SureCritic fork of win32-service

Forked from upstream **win32-service 2.3.2** and patched for the SureCritic M1/Shopkey connector, 
which runs under **Ruby 3.3.2 (x64-mingw-ucrt)**. 
Versioned **2.3.2.1** so a stray `gem install win32-service` can't shadow the fork with stock 2.3.2.

## Why this fork exists

On Ruby 3.3.2 the stock 2.3.2 daemon failed to start the Windows service:

```
Service initializing...
Service initialized
*** failure: Service_Main thread exited abnormally.
```

...and Windows service **recovery never restarted it**. 
Two upstream PRs (never released to a 2.3.x gem) address the underlying issues; we took the parts that fit our design:

1. **chef/win32-service#85 — Ruby 3 slow start (taken in full).**
   The dispatcher ran via an FFI `CreateThread`. 
   Under Ruby 3's threading model `StartServiceCtrlDispatcher` took ~37s, blowing the SCM's 30s start timeout, 
   so the service thread was torn down before signaling start → "Service_Main thread exited abnormally". 
   Fix: run the dispatcher in a plain Ruby `Thread.new` and wait on `hStartEvent` alone (re-checking `hThread.alive?`), 
   with a `sleep(0.1)` GVL yield in the wait loop. ~37s → <1s. **This is the fix that resolves the production bug**
   — a fast start never trips the timeout.

2. **chef/win32-service#83 — exit code on abnormal stop (taken, reworked).**
   On stop the daemon always reported `SERVICE_STOPPED` with exit code `NO_ERROR` (0),
   which the SCM treats as a clean stop (no recovery). #83 changed it to an unconditional `1`. 
   We took the *idea* but made it **conditional**: report a non-zero code only when the stop was NOT requested by the SCM. See below.

## What we did NOT take

- **#83's `failure_flag` gem plumbing** (the `SERVICE_FAILURE_ACTIONS_FLAG` struct,
  the `failure_flag` param on `Service.configure`/`create`, and the `ChangeServiceConfig2` wiring). 
  The connector sets that flag itself at service-configure time via `sc.exe failureflag <svc> 1`, so the gem doesn't need to. 
  Keeping it out keeps the fork minimal and avoids two places setting the same flag.
- **#83's unconditional exit-code `1`** — replaced with the conditional logic below.

## What changed (vs. stock 2.3.2)

- **`lib/win32/daemon.rb`**
  - `Service_Main` `ensure`: instead of always reporting `SERVICE_STOPPED, NO_ERROR`, 
    it now reports a non-zero code **only on a non-graceful stop**:
    ```ruby
    graceful  = [SERVICE_CONTROL_STOP, SERVICE_CONTROL_SHUTDOWN].include?(@@waiting_control_code)
    exit_code = graceful ? NO_ERROR : SERVICE_STOPPED_ABNORMALLY
    SetTheServiceStatus.call(SERVICE_STOPPED, exit_code, 0, 0)
    ```
    So an admin/shutdown-requested stop (and the connector's own restart/patch/uninstall cycles) report a clean 0
    — the service won't relaunch itself on an intentional stop — while a genuine abnormal exit reports non-zero 
    so the SCM can run recovery. (#83)
  - Removed the standalone `ThreadProc` FFI function; the dispatcher now runs in `Thread.new(Service_Main)` inside `#mainloop`, 
    and the start wait is `WaitForSingleObject(@@hStartEvent)` + `hThread.alive?` guard + `sleep(0.1)`. (#85)
- **`lib/win32/windows/constants.rb`** — added `SERVICE_STOPPED_ABNORMALLY = 1` (the named non-zero exit code, instead of a magic `1`).
- **`lib/win32/windows/functions.rb`** — removed the now-unused `CreateThread` 
  and `WaitForMultipleObjects` `attach_pfunc` declarations. (#85)
- **`lib/win32/windows/version.rb`** — `2.3.2` → `2.3.2.1`.

## How recovery actually works (gem vs. connector)

The gem's job is narrow: **fix the Ruby-3 start (so the service stays up) 
and report an honest exit code (0 = clean stop, non-zero = abnormal).** It does NOT own recovery policy.

The connector owns recovery policy in `service_configure` / `service_create`:
- `failure_actions: [RESTART, RESTART, NONE]`, `failure_delay` 1 min, `failure_reset_period` 5 min.
- `sc.exe failureflag <svc> 1` — **required** for the gem's non-zero exit code to mean anything. 
  Without this flag (Windows default 0), the SCM ignores a non-zero-exit stop and recovery never fires. 
  The exit code and the flag are a matched pair.

## Build & install on the build box

```
# in the fork checkout (Ruby 3.3.2 x64-mingw-ucrt)
gem build win32-service.gemspec          # -> win32-service-2.3.2.1.gem

# on the Ocran build box
gem uninstall win32-service -v 2.3.2     # remove stock so only the fork remains
gem install win32-service-2.3.2.1.gem
```

Ocran traces `require 'win32/daemon'` and bundles whatever's installed, so the fork ships in the EXE with no change to the connector source.
**Do not** run a plain `gem install win32-service` on the build box afterward
  — it pulls stock 2.3.2 from RubyGems and the bug returns silently.

## Verifying the shipped EXE got the fix

- The connector logs `win32-service Daemon version: 2.3.2.1` at startup — confirms the EXE bundled the fork (stock would log `2.3.2`).
- Service starts in 1-2 seconds, no 30 second hang; `ServiceMain starting...` follows `Service initialized` promptly.
- A genuine abnormal exit triggers an SCM restart per the connector's recovery actions
  (verify `sc.exe qfailureflag <svc>` shows TRUE and `sc.exe qfailure <svc>` shows the RESTART/RESTART/NONE actions); 
  an admin `sc.exe stop` does NOT trigger a restart.
