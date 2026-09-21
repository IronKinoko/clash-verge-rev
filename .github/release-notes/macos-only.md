macOS-only build (Apple Silicon).

First-time install: this build is not Apple-signed, so macOS
reports the app as "damaged" after downloading. Drag the app
to /Applications, then run once per download:

    xattr -cr /Applications/Clash\ Verge.app

and open it normally. Auto-updates do not require this step.
