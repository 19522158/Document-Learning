# Secure Boot: The Chain of Trust
In embedded systems, physical access is the ultimate vulnerability. Secure Boot ensures that even if an attacker physically modifies the software on the flash memory, the hardware will detect the tampering and flat-out refuse to run it.

1. The Core Concept: Asymmetric Cryptography
  Secure Boot relies on asymmetric cryptography (usually RSA or ECDSA).
  - The Private Key (The Pen): This is used to sign the firmware. It acts as your company's unforgeable signature. It must be kept in a heavily guarded, offline Hardware Security Module (HSM) at your company's headquarters or in a secure CI/CD cloud pipeline.
  - The Public Key (The Magnifying Glass): This is used mathematically to verify the signature made by the Private Key. It cannot be reverse-engineered to figure out the Private Key, and it cannot be used to sign new firmware

2. The Chain of Trust (Execution Flow)
  Once the Hardware Root of Trust validates the first Public Key, trust is handed off sequentially. Every stage mathematically verifies the cryptographic signature of the next stage before passing execution control.
  - Boot ROM (Silicon) uses eFuses to verify → FSBL.
  - FSBL uses its verified Public Key to verify → U-Boot (Second Stage Bootloader).
  - U-Boot uses its verified Public Key to verify → FIT Image (Linux Kernel + Device Tree bundled together).
  - Linux Kernel uses a cryptographic hash tree to verify → Root Filesystem (via dm-verity).
