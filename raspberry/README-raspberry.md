# Peugeot infotainment Project Raspberry info

The OS for the Pi is a custom built linux distribution using Yocto. The GUI is made in Qt 5.

## Yocto

Consult the Yocto documentation for additional information.

All customization was done in a custom `meta-psa` layer. This layer contains the image definition, boot splash, qt and gstreamer customizations, and the GUI app itself.

The custom layer, and its requirements will be uploaded at a later date once it's finalized.

## GUI App

The GUI App, also named `PSA App` is written using Qt 5. The raspberry doesn't have a window manager, so it's rendering a fullscreen window using `eglfs`. The app is structured around tabs, where each tab shows a customized widget. Each widget is self contained, and only aware of itself. Additional information is communicated to the widget from the Main window using signals and slots.

For additional information, consult the [code](https://github.com/Mive82/psa-app).
