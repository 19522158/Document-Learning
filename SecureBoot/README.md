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
      
3. Workflow
  ![Workflow](Diagram.drawio.svg)

4. User-Space Verification (Before & During the Write)
   This process happens while your camera is actively running its normal Linux environment. A daemon (like RAUC, SWUpdate, or Mender) is running in the background.
   The Steps:
     - The Manifest Check (Pre-flight): The OTA server doesn't send the massive 500MB firmware image right away. It first sends a tiny file called a Manifest or Metadata. This file contains the cryptographic hash (fingerprint) of the upcoming firmware, the version number, and the hardware compatibility list. The Manifest itself is cryptographically signed.
     - User-Space Signature Verification: The OTA daemon uses a Public Key stored in its current Root Filesystem (often an x.509 certificate) to verify the Manifest's signature. If it fails, the daemon drops the connection immediately.
     - Stream and Hash (During Download): If the Manifest is valid, the daemon begins streaming the actual firmware payload. As the data flows in over the network, the daemon calculates the hash of the data on the fly.
     - The Write: The daemon writes the incoming data directly to the inactive partition (Slot B).
     - The Read-Back (Post-Write): Once the write is complete, a robust OTA daemon will read the data back out of Slot B and hash it one final time. It compares this final hash to the hash in the Manifest.

5. Bootloader Verification (The Ultimate Gatekeeper)
    Once the OTA daemon is satisfied, it sets a flag (usually an environment variable like boot_target=Slot B) and issues the reboot command. Linux shuts down. The User-Space daemon is dead. The CPU resets.
    Now, U-Boot (the Second Stage Bootloader) wakes up. U-Boot operates in a highly constrained, bare-metal environment. It does not trust the OTA daemon, it does not trust the network, and it does not trust the flash memory.
    The Steps:
      - Read the Environment: U-Boot reads the environment variables and sees it needs to boot from Slot B.
      - Load into RAM: U-Boot reads the Linux Kernel and the Device Tree from Slot B into the main DDR RAM. (In modern embedded systems, these are bundled together into a single file called a FIT Image - Flattened Image Tree).
      - The Hardware-Anchored Check: U-Boot takes the Public Key that is compiled directly into its own code (which was verified by the hardware eFuses seconds earlier) and mathematically verifies the cryptographic signature attached to the FIT Image in RAM.
      - Execution or Rejection:
          + If Valid: U-Boot passes the execution pointer to the new kernel. The new firmware boots.
          + If Invalid: U-Boot immediately halts the boot process. It decrements a bootcount variable, changes the boot_target back to Slot A, and issues a hardware reset. The device safely boots back into the old, working firmware.

6. Overview SWUpdate
   a. Overview
   When you build an OTA update, SWUpdate packages everything into a single file ending in .swu. But a .swu file is actually just a standard cpio archive (similar to a .tar or .zip file).
   Inside that single .swu file, the items are stored in a very specific, mandated order:
     - sw-description: This is the "Manifest." It is a tiny text file that lists all the hardware rules, where to install things, and most importantly, the SHA-256 hashes (fingerprints) of the actual firmware files.
     - sw-description.sig: This is the cryptographic signature of the sw-description file, created by your company's Private Key.
     - rootfs.ext4 (The actual payload): Your heavy, 500MB+ Linux filesystem.
     - zImage / fitImage (The actual payload): Your Linux kernel.
   b. How SWUpdate Reads the Stream
    Step 1: The Bouncer (Verifying the Manifest)
      - Because sw-description and its signature are at the very front of the .swu file, SWUpdate reads them into RAM first.
      - It grabs the Public Key stored on your device (usually in /etc/swupdate/public.pem).
      - It uses that Public Key to mathematically verify sw-description.sig against the sw-description text file.
      - Result: If the math fails, SWUpdate stops immediately and deletes everything. If it passes, SWUpdate now absolutely trusts the sw-description text file. It knows a hacker didn't alter it.
    Step 2: The Check-off List (Streaming & Hashing the Payload)
      - Inside the trusted sw-description file, there is a line that looks like this: filename = "rootfs.ext4"; sha256 = "a1b2c3d4e5f6...";
      - As SWUpdate continues to read the .swu stream, it encounters the massive rootfs.ext4 file.
      - Once it reaches the end of rootfs.ext4, the CPU finalizes the SHA-256 calculation. It now has a single, final SHA-256 hash for the file it just wrote.
    Step 3: The Final Match
      - SWUpdate takes the hash it just calculated on-the-fly, and compares it to the sha256 = "a1b2c3d4..." string written inside the trusted sw-description file.
      - If they match: The payload is perfect. It wasn't corrupted during the download, and no one tampered with it. SWUpdate tells the bootloader to switch partitions.
      - If they do NOT match: The payload is corrupted. Maybe a network packet dropped, or maybe your flash memory has a bad block. SWUpdate throws an error and does not tell the bootloader to switch partitions. Your device is saved from booting a broken filesystem.
  
7. My Question
      But if the hacker hold the true signature manifest, and only change the rootfs.ext4 in .swu file. So that, it can pass the manifest trust and the device continue to use the sw-description text file

      The Setup: The hacker takes your official, perfectly valid v1.0.swu file.
      The Swap: They unzip it, delete your real rootfs.ext4, put in their own malicious rootfs.ext4 (let's say they added a hidden backdoor), and zip it back up into a new .swu file. They leave your original sw-description and sw-description.sig completely untouched.
      The Trap: They send this modified .swu file to your camera.
      Now, let's watch what SWUpdate does:
      - Step 1: The Bouncer (Manifest Check): SWUpdate reads the original sw-description and the original sw-description.sig. Because the hacker didn't touch these, the math passes. SWUpdate says, "Okay, I trust this manifest."
      - Step 2: The Stream: SWUpdate starts reading the hacker's malicious rootfs.ext4. As it reads it, it calculates the SHA-256 hash of this malicious file. Let's say the hacker's file generates a hash of 999xxx...
      - Step 3: The Final Match (Where the attack dies): SWUpdate finishes writing the file. Now it looks at the trusted sw-description manifest to see what the hash should be.
      - The trusted manifest says: filename = "rootfs.ext4"; sha256 = "111abc...";
      - SWUpdate compares them: 999xxx... does NOT equal 111abc...
      -> Boom. The update fails. SWUpdate immediately realizes that the payload does not match the description in the signed manifest. It deletes the hacker's partition, throws an error, and refuses to tell the bootloader to switch slots.

      Why the Hacker is Trapped:
      For the hacker to make the update succeed, they have two impossible choices:
      - Choice A: They leave the manifest alone. (Result: The payload hash won't match the manifest hash. SWUpdate rejects it at Step 3).
      - Choice B: They open sw-description and change the text from sha256 = "111abc..." to sha256 = "999xxx..." so it matches their malicious file.
      - (Result: Because they changed the text inside sw-description, it no longer mathematically matches the sw-description.sig file. SWUpdate will reject it at Step 1, before it even looks at the payload).
