---
layout: post
title: Claude Banged my Module
---

My name is Davis. I have a homelab, and made a fun discovery with Claude Code while messing around with SFP modules.

A few years ago, I bought a Brocade VDX 6720 with the intent of it being a 10G backbone for my network. I paid maybe $100 USD for it, and it seemed like a great deal. This was in 2022, just before 10G copper switches and hardware got really cheap. NBASE-T switches were still pretty expensive too.

I hadn't ever messed with enterprise gear, and I quickly learned I needed specific Brocade coded modules for this switch. As far as I could tell, there was no way to disable vendor checking like Intel X520 or Cisco switches. Around this time, 10G > multi gig RJ45 transceivers were getting cheap, so I bought a handful of them from Flypro specifically coded for Brocade.

After receiving these modules, it turned out they weren't Brocade coded at all and would not work in my switch. I ordered a ReveLPROG programmer with the SFP attachment to read the EEPROM and discovered they were just generic coded modules. After a bit of back and forth with Flypro support, I had the SFP programming password, and I was able to flash the EEPROM of a known working Brocade module to the Flypro RJ45 module, and it worked no problem in my VDX 6720.

The main point of this story was how I discovered that SFP vendor flags were a thing, and that they can be reprogrammed and potentially have a password.

There are a handful of commercial products that can program and flash modules. Popular options seem to be from FS.com, ReveLPROG, SFP Doctor (open source), and many others. Mostly they all use weird software, some are windows only, etc etc.

## Back to 2026:

I'm reconfiguring my homelab, and trying to connect a dual port Intel X520 NIC to my MikroTik CCR2004 with old Finisar 10G Fibre Channel transceivers. I had successfully used these in media converters and a few other pieces of gear. When I couldn't get the link up, I assumed the X520 and ixgbe driver needed unsupported module support enabled. One of the perks of these cards is this feature, and they will work with pretty much any module.

Being incredibly lazy, I fired up Claude Code and told it to enable the flag so I could get the link up. Claude comes back and reports the flag IS enabled, and there should be no issue with the transceivers. I remembered that ethtool can be used to read back EEPROMs, and had Claude investigate what the problem could be. A couple prompts later, I find out that these modules report as Fibre Channel modules, and that's why the card and ixgbe won't bring the link up.

Out of pure ignorance, I ask Claude if it can re write the EEPROM using the network card, so I don't have to dig out a windows machine and fire up the terrible ReveLPROG software. Claude piddles around for a bit, trying a few things that didn't work. I'm just sitting there with my eyes glazed over, vaguely getting a sense of what's happening. Eventually Claude tells me it can bitbang the I2C protocol and write whatever we want to the module. Pretty wild stuff.

A few prompts and scripts later, I've got my Finisar modules flashed to report themselves as Ethernet modules, and my pile of RJ45 SFPs are flashed back to factory so they negotiate properly with my CCR2004.

See below for Claude's explanation of how this was done. This is something I NEVER would have been able to do myself, and the span of an hour or so I had a solution that was free and honestly worked way better than any of the commercial module programmers.

## How SFP EEPROM Writing Works Through the 82599's BAR0

### The hardware path

Every SFP+ cage has an I2C bus (two wires: clock and data) connecting the host to the SFP module's EEPROM. On the Intel 82599ES, this bus is controlled by a single 32-bit register at BAR0 offset 0x028 (I2CCTL), which exposes four bits:

| Bit | Name     | Function                  |
|-----|----------|---------------------------|
| 0   | CLK_IN   | Read the clock line state |
| 1   | CLK_OUT  | Drive the clock line      |
| 2   | DATA_IN  | Read the data line state  |
| 3   | DATA_OUT | Drive the data line       |

That's it — the 82599 has no I2C controller logic. It just gives you raw GPIO access to the two wires. Everything else is software.

### What we're doing

We memory-map the NIC's BAR0 (the first PCI resource, exposed by the kernel at `/sys/bus/pci/devices/<addr>/resource0`) into userspace with `mmap()`. This gives us direct read/write access to all 82599 registers, no kernel driver involvement.

From there, we bit-bang the I2C protocol by toggling CLK_OUT and DATA_OUT and reading DATA_IN, implementing:

1. **START/STOP conditions** — SDA transitions while SCL is high
2. **Byte transmission** — shift out 8 bits MSB-first, read ACK
3. **Byte reception** — release SDA, clock in 8 bits, send ACK/NACK

### The I2C addressing

SFP modules expose two I2C slave addresses (per SFF-8472):

- **0xA0** (write) / **0xA1** (read) — the main EEPROM (A0h page): module identity, compliance codes, vendor info, checksums
- **0xA2** (write) / **0xA3** (read) — the diagnostic page (A2h): thresholds, calibration, real-time monitoring, and the password register at bytes 123–126

### Write protection and passwords

Most SFP modules have their EEPROM write-protected. The SFF-8472 spec defines a password mechanism: write 4 bytes to A2h registers 123–126, and if the password matches, the module unlocks its A0h page for writes. The unlock is temporary — it typically expires after a timeout or power cycle.

So a write sequence looks like:

1. Send I2C write to 0xA2, register 123, with 4 password bytes
2. Send I2C write to 0xA0, register N, with the new value
3. Wait ~50ms for EEPROM write cycle
4. Send I2C read from 0xA0, register N, to verify

### Why this works without the kernel

The ixgbe kernel driver initializes the 82599 during PCI probe (powers on the PHY, sets up the MAC, etc.), which also powers the SFP cage's I2C bus. Once that's done, the I2C lines are live. We're not going through the driver's I2C abstraction or the kernel's i2c-dev subsystem — we're writing directly to hardware registers through the memory-mapped PCI BAR. The kernel doesn't know or care.

The tradeoff is that there's no bus arbitration. If the kernel driver tried to read the SFP at the same time we're mid-transaction, the bus would get corrupted. In practice this isn't an issue because the driver only polls SFP state infrequently and our bit-bang transactions are fast.
