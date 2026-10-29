# RK3576 GPU userspace selection

Install `radxa-system-config-rk3576-gpu` and its dependencies, then reboot.
Enable the Mali overlay for the vendor driver, or disable it for Panfrost.
Keep `graphical.target` and log in to **Plasma (Wayland)** normally.
Keep the image's existing OpenGL ES configuration for Plasma.

Before the display manager starts, `radxa-gpu-select.service` checks the bound
driver and gives either Mesa or the packaged Mali EGL/GLES/GBM libraries
priority through `ldconfig`. The required `libmali-bifrost-g52-g29p1` package
provides standard-SONAME forwarding libraries; standalone libmali symlinks
cannot replace them. DMA heap access is granted to
the `render` group and the active local session, including the GDM greeter.

In Mali mode, the service also enables QtWayland's existing Mali surface
lifetime handling (QTBUG-127980) for `plasmashell` through
`/run/radxa-gpu/plasmashell.env`. Debian's Qt can query the GLVND renderer
instead of the non-GLVND Mali EGL context and miss its automatic Mali check.
`QT_WAYLAND_ENABLE_DONECURRENT_WORKAROUND=1` releases the current EGL binding
at frame end, avoiding accesses to an old Wayland surface after destruction.
Plasma keeps threaded rendering and Mali hardware acceleration. The environment
file is regenerated empty in Mesa mode; other applications are unaffected.

Upstream: https://github.com/qt/qtwayland/commit/125af88b2c498a57bce2135a806cf940904afe86

```sh
cat /run/radxa-gpu/mode
cat /run/radxa-gpu/plasmashell.env
systemctl status radxa-gpu-select.service display-manager.service
ldconfig -p | grep -E 'lib(EGL|GLESv2|gbm)\.so'
# Run from the logged-in Plasma session:
qdbus6 org.kde.KWin /KWin supportInformation
```

Expect ARM / Mali-G52 in Mali mode and Mesa / Mali-G52 (Panfrost) otherwise.
Reboot after changing the overlay; desktop OpenGL/GLX acceleration is outside
this EGL/GLES setup. To remove the setup completely, remove both this package
and the vendor libmali package, then run `ldconfig` and restart the session.
