# kmatrix-koreader

A native Matrix client for e-readers running [KOReader](https://koreader.site/). 

`kmatrix` runs completely on-device across jailbroken hardware including **Kindle Paperwhite 5**, **reMarkable 2**, and **armv7 Kobo** e-readers. All TLS terminated connections, Matrix synchronization, end-to-end encryption (E2EE), and local message storage occur directly on the device—requiring no companion server or external phone pipeline.

For detailed technical specifications, architecture notes, and development instructions, see [`TECHNICAL.md`](TECHNICAL.md).

---

## Features

- **Zero-Dependency Architecture:** Standalone execution directly on e-ink hardware without requiring local proxy servers or companion apps.
- **On-Device End-to-End Encryption:** Full Olm/Megolm cryptographic verification and room key backup restoration powered by Rust.
- **Optimized for E-Ink:** Lightweight UI thread decoupled from heavy background network synchronization and cryptography operations.
- **Data-at-Rest Protection:** Local SQLite message store encrypted with AES-HKDF keys isolated on restricted internal hardware partitions.

---

## Installation

1. Download the latest release archive (`.tar.gz` or `.zip`) from the **Releases** page.
2. Extract the archive on your computer.
3. Connect your e-reader to your computer via USB and open the KOReader root directory:
   - **Kindle:** `/mnt/us/koreader`
   - **reMarkable / Kobo:** Your respective KOReader data directory.
4. Copy the extracted items into place:
   - Place the `kmatrix.koplugin/` folder into:
     ```text
     /mnt/us/koreader/plugins/
     ```
   - Place the `kmatrixd` binary into the `kmatrix/` folder:
     ```text
     /mnt/us/koreader/
     ```
5. Ensure `kmatrixd` has execution permissions:
   ```bash
   chmod +x /mnt/us/koreader/kmatrix/kmatrixd
