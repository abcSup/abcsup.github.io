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

Reverse debugging is the ability to "reverse" program execution or stepping backward in a debugger. This is usually done using software to track the program previous states and restore them. There are many implementations of reverse-debugging with different use cases, such as, [rr](https://rr-project.org/), [gdb](https://www.gnu.org/software/gdb/) and [qira](htttp://qira.me/).

rr uses the record-replay technique that captures non-deterministic events (syscall and signal interrupt) and replays them. When reversing execution, rr will restore the program to a nearest checkpoint and re-run it until the desired point. This helps rr maintain high performance when running the target program during recording and generate a far smaller trace file. However, it has a few constraints on the platform it is running on. For example, it requires hardware performance counters that are available on Intel platform.

Meanwhile, `qira` and `gdb` both record the changes in registers and memory of ever executed instruction and replay them by applying the latest recorded changes before a specified execution time. However, this imposes a huge overhead in performance and generates bigger trace files, but it is easier to support multiple platforms and useful for other analysis in future. Therefore, `r2` implements similar technique to achieve reverse debugging.

In the previous implementation, `r2` captures registers and memory snapshots as checkpoints which the debugger can be restored to. However, this is difficult to locate the previous instruction if the current instruction address has multiple entry points. For example, after the program has executed the `JMP` instruction, there is no information that helps us determine the instruction address before the jump.

This can be supplemented by tracing registers and memory changes after executing an instruction. We tracks all the changes in each register and memory in their own table with an index that marks the execution time.

When stepping or continuing back, we locate the previous instruction and apply the latest changes in registers and memory before the index. 

### dbg.trace_continue

However, tracing every change in an instruction causes significant overhead in performance. Therefore, we can skip tracing at some parts of the program by setting `dbg.trace_continue` to `false`, so the debugger will no longer trace every instruction when continuing and capture a snapshot at the next stop (e.g. breakpoints).

{% include image name="left.png" caption="dbg.trace_continue=on" %}
{% include image name="right.png" caption="dbg.trace_continue=off" %}

## KDNet

Microsoft Windows provides KD interface to support kernel-mode debugging on Windows platform. `r2` has supported KD interface over serial connections over the past. On the other hand, KDNet supports kernel-mode debugging over network connections using the same KD interface. KDNet is basically KD protocol wrapped in UDP connections. This has multiple advantages over serial connections:
- Easy to set up on remote Windows machine compared to serial ports.
- Encrypted connections.

The following command is used to set up KDNet on a Windows machine:

`C:\ bcdedit /dbgsettings net hostip:w.x.y.z port:n key:Key`

Thanks to [Lekensteyn/kdnet](https://github.com/Lekensteyn/kdnet/) for getting me started. I have also reverse-engineered the DbgEng.dll to further fill out missing details on KDNet protocol. The `KdNetConnection` class in DbgEng.dll contains most information about the protocol.

The KDNet packet consists of:

{% include image name="kdnet.png" caption="KDNet Packet Structure" %}

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

Here are the methods that handle the encryption and decryption of KDNet packets in DbgEng.dll:

- KdNetConnection::EncryptKdPacket
- KdNetConnection::DecryptKdPacket
- KdNetConnection::EncryptKdPacketV1
- KdNetConnection::DecryptKdPacketV1

The `EncryptKdPacket` method is responsible for creating the KDNet packets and encrypt them later by calling `EncryptKdPacketV1`.

When we set up KDNet debugging, we have to provide an 256-bit encryption key (`Key`) or allow `bcdedit` generate one for us. This encryption key is first used to establish the connection and to negotiate another control key that will be used to encrypt all packets afterwards.

When KDNet debugging is enabled on the target machine, it will continuously send the `Poke` packet (`KdNetConnection::SendPokePacket`) to the specified host until it receives the `Response` packet (`KdNetConnection::SendResponsePacket`). The `Response` packet holds the control key defined by the host machine, which is 256-bit and randomly generated.

The secret key used for SHA256HMAC is obtained by negating each byte of the encryption key (`~(Key[i])`).

The KDNet data is encrypted with AES256 using the encryption key or control key after the connection is established and its first 16-bytes SHA256HMAC as IV.

More information about the KDNet protocol can be found in my KDNet PR in the references section.

---
## References

- [GSoC Project Link](https://summerofcode.withgoogle.com/organizations/4946212249141248/#5939223015718912)
- [Reverse Debugging Github PRs](https://github.com/radareorg/radare2/pulls?q=is%3Apr+project%3Aradareorg%2Fradare2%2F25)
- [KDNet Github PR](https://github.com/radareorg/radare2/pull/17340)
- Talks and slides at r2con2020 (TBA)

