---
layout: post
title:  "Google Summer of Code 2020 with Radare"
date:   2020-08-31 23:00:00 +0800
categories: re
---
During the summer, I have participated in Google Summer of Code 2020 to develop open-source software (OSS) with [Radare](https://rada.re/). My work is focused on improving the `r2` debugger, which includes:
  - Reverse Debugging
  - WinDbg KDNet

Thanks to every member in the radare dev team for providing me all the guidance and new insights in reverse-engineering.

## Reverse Debugging

Reverse debugging is the ability to "reverse" program execution or stepping backward in a debugger. This is usually done using software to record the program states and restore them.

In the previous implementation, `r2` captures registers and memory snapshots as checkpoints which the debugger can be restored to. However, this is problematic when trying to locate the previous instruction or continuing backward. For example, the program executed a `JMP` instruction or there are multiple breakpoints from the nearest checkpoint to the current instruction.

This can be supplemented by tracing all registers and memory changes after executing each instruction. Then, we apply the latest changes to get back the previous state of each register and memory.

### dbg.trace_continue

However, tracing every change in an instruction causes significant overhead in performance. Therefore, we can skip tracing some parts of the program by setting `dbg.trace_continue` to `false`, so the debugger will not trace every instruction when continuing and capture a snapshot at the next stop (e.g. breakpoints).

{% include image name="left.png" caption="dbg.trace_continue=on" %}
{% include image name="right.png" caption="dbg.trace_continue=off" %}

## KDNet

Microsoft Windows provides KD interface to support kernel-mode debugging on Windows platform. `r2` has supported KD interface over serial connections over the past. On the other hand, KDNet supports kernel-mode debugging over network connections using the same KD interface. KDNet is basically KD protocol wrapped in UDP connections. This has multiple advantages over serial connections:
- Easy to set up on remote Windows machine compared to serial ports.
- Encrypted connections.

Thanks to [Lekensteyn/kdnet](https://github.com/Lekensteyn/kdnet/) for getting me started. I have also reverse-engineered the DbgEng.dll to further fill out missing details on KDNet protocol.

{% include image name="kdnet.png" caption="KDNet Packet Structure" %}

The KDNet packet consists of:
- KDNet Header
  - Magic - 0x4D444247 ("MDBG")
  - Version - KDNet protocol version
  - Type
    - Data (0x0)
    - Control (0x1)
- KDNet Data (16-byte aligned)
  - Sequence Number
  - Direction (first bit)
    - Debuggee -> Debugger (0x0)
    - Debugger -> Debuggee (0x1)
  - Pad Size (last 7 bits)
  - KD Packet
- HMAC

### Encryption

When we set up KDNet debugging, we have to provide an 256-bit encryption key or allow `bcdedit` generate one for us. This encryption key is first used to establish the connection and negotiate another control key that will be used to encrypt all packets afterwards.

The secret key used for SHA256HMAC is obtained by negating each byte of the encryption key (`~(key[i])`).

The KDNet data is encrypted with AES256 using the encryption key or control key after the connection is established and its first 16-bytes SHA256HMAC as IV.

---
## References

- [GSoC Project Link](https://summerofcode.withgoogle.com/organizations/4946212249141248/#5939223015718912)
- [Reverse Debugging Github PRs](https://github.com/radareorg/radare2/pulls?q=is%3Apr+project%3Aradareorg%2Fradare2%2F25)
- [KDNet Github PR](https://github.com/radareorg/radare2/pull/17340)
- Talks and slides at r2con2020 (TBA)

