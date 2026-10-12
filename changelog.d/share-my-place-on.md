### Changed

- **Place sharing is now a choice for each device.** Every device starts with **Share my place** off, including your existing devices and the web reader. Each keeps its own place and read status. To share again, open **Account → Devices** and tick **Share my place** on each device you want to sync.
- **Highlights stay on the device that made them.** Highlights from the web reader or another device no longer appear on your e-reader. Every device's highlights and notes still show together in the web reader, labelled by device. Older highlights without a recorded device remain available to readers of their original kind.
- **KOReader needs the paired CWNG plugin for per-device control and device-only highlights.** A Kobo synced to CWNG also supports per-device control. E-readers that CWNG cannot identify keep their existing sync behaviour.
- **Administrators can temporarily restore the previous place and read-status sync behaviour with `CWNG_SHARE_PLACE_GATES=0`.** This emergency switch leaves device-only highlights in effect.

### Fixed

- If CWNG temporarily cannot register the web reader, its reading place stays private while Hardcover progress updates continue as they do for an unchecked web reader.
