# Subnets and CIDR Notation Explained

> Understanding what a subnet actually is and how to read and explain CIDR notation confidently.

---

## What Is a Subnet?

A subnet is a smaller, defined range of IP addresses carved out of a larger
network. It allows segmentation of a network into isolated sections — for
example separating public-facing resources from private backend resources —
and controlling traffic between them.

CIDR is the *notation* used to define a subnet, not what a subnet *is*.
Saying "a subnet is a CIDR range" is technically not wrong but it is
incomplete. The concept matters more than the syntax.

---

## What Is CIDR Notation?

An IP address is 32 bits long. Written in binary, `10.0.0.0` looks like:

```
00001010 . 00000000 . 00000000 . 00000000
```

CIDR notation adds a slash and a number after the IP address — for example
`10.0.0.0/24`. That number tells you how many bits are **fixed** (defining
the network) and how many bits are **free** (available for individual addresses).

---

## How to Read 10.0.0.0/24

```
10  .  0  .  0  .   0   /24
|----fixed 24 bits----|free|
                       2⁸ = 256 addresses
```

- **/24** means the first 24 bits are fixed — the network address, cannot change
- The remaining **8 bits are free** — these define individual hosts in the subnet
- 2⁸ = **256 total addresses**
- Range: `10.0.0.0` to `10.0.0.255` — only the last octet changes
- AWS reserves 5 addresses in every subnet = **251 usable IPs**

---

## How to Read 10.0.0.0/16

```
10  .  0  .  0  .  0   /16
|--fixed 16 bits--|  free  |
                   2¹⁶ = 65,536 addresses
```

- **/16** means the first 16 bits are fixed
- The remaining **16 bits are free**
- 2¹⁶ = **65,536 total addresses**
- Range: `10.0.0.0` to `10.0.255.255` — the last two octets change
- Typical size for a VPC — large enough to carve many subnets from it

---

## The Rule: Bigger Number = Smaller Network

The larger the number after the slash, the fewer free bits, the smaller
the address range.

| CIDR | Fixed bits | Free bits | Total addresses | Typical use |
|---|---|---|---|---|
| 10.0.0.0/8 | 8 | 24 | 16,777,216 | Very large network |
| 10.0.0.0/16 | 16 | 16 | 65,536 | VPC |
| 10.0.0.0/24 | 24 | 8 | 256 | Typical subnet |
| 10.0.0.0/28 | 28 | 4 | 16 | Small subnet |

---

## VPC vs Subnet — The Hierarchy

The VPC defines the total IP pool. Subnets are slices carved from that pool.

```
VPC: 10.0.0.0/16  (65,536 addresses — the whole building)
├── Subnet: 10.0.1.0/24  (256 addresses — floor 1, public)
├── Subnet: 10.0.2.0/24  (256 addresses — floor 2, private)
└── Subnet: 10.0.3.0/24  (256 addresses — floor 3, public AZ-b)
```

A subnet prefix must always be a higher number than the VPC prefix — a subnet
can never be larger than the VPC it lives in. A /15 subnet cannot fit inside
a /16 VPC because /15 is a larger range than /16.

---

## The 5 Reserved Addresses

AWS reserves 5 IP addresses in **every subnet** — not in the VPC itself.
Using `10.0.1.0/24` as the example:

| Address | Reserved for |
|---|---|
| 10.0.1.0 | Network address — identifies the subnet |
| 10.0.1.1 | VPC router — default gateway |
| 10.0.1.2 | AWS DNS server |
| 10.0.1.3 | Reserved by AWS for future use |
| 10.0.1.255 | Broadcast address |

The VPC does not lose 5 addresses — each subnet loses 5. Creating 10 subnets
means 50 addresses are reserved across the VPC.

---

## How to Explain It Confidently

**If asked "what is a subnet?"**

*"A subnet is a smaller segment of IP addresses carved from a larger network.
It lets you isolate resources — for example keeping public-facing servers
separate from private backend servers — and control what traffic can flow
between them."*

**If given 10.0.0.0/24 and asked what /24 means:**

*"The /24 means 24 bits are fixed and 8 bits are free. Two to the power of 8
gives 256 total addresses — from 10.0.0.0 to 10.0.0.255. AWS reserves 5 in
every subnet, so there are 251 usable IP addresses."*

The progression that shows understanding: concept → calculation → range →
practical usable count. Not just reciting a number but showing where it
comes from.

---

*Part of my AWS Cloud Bootcamp — see the root [README](../README.md) for all labs.*
