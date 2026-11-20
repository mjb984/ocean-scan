# Ocean Scan
Read-only Fisker Ocean scanner for a vLinker FD+ over Bluetooth, as an installable web app.
Open https://mjb984.github.io/ocean-scan/ in Chrome on Android.

- Our own code from the public ELM327, ISO-TP and UDS standards.
- Only read requests are allowed (UDS 0x22, 0x19 read sub-functions, OBD 01/09); clear, reset, write and security access are refused in code.
- Results upload to the owner's private repo using a key stored only on the phone. No data is stored in this repo.
