---
title: "Checking if a Used Core i7-6700 is Genuine Without Benchmarks"
description: "When purchasing a used Intel CPU, you might wonder if the heat spreader has been swapped with another..."
pubDatetime: 2026-09-04T04:07:06.484Z
updatedDate: 2026-09-04T04:13:02.754Z
---

When purchasing a used Intel CPU, you might wonder if the heat spreader has been swapped with another CPU or if it's a fake with a modified model number.

This time, using an actual **Intel Core i7-6700** as an example, we'll try to determine how much we can verify its authenticity without using performance measurements, by checking:

*   Heat spreader markings
*   S-Spec
*   FPO (Batch Number)
*   ATPO (Serial Number)
*   Data Matrix
*   Partial ATPO

## The Core i7-6700 We're Checking

The CPU surface has the following markings:

```text
i7-6700
SR2L2 3.40GHZ
X012C345 (e4)
```

First, let's clarify what each of these means.

| Display    | Meaning                 |
| ---------- | ----------------------- |
| i7-6700    | CPU model name          |
| SR2L2      | S-Spec                 |
| 3.40GHZ    | Base clock speed        |
| X012C345   | FPO (Batch Number)      |
| (e4)       | Not ATPO                |

According to Intel's official information, the Core i7-6700 has an S-Spec of `SR2L2` and an R0 stepping. Also, the i7-6700 has a base clock speed of 3.40GHz.

Therefore, there are no inconsistencies in the markings on the surface so far.

## What is FPO?

FPO stands for "Finished Process Order," and Intel treats it as the Batch Number.

In this CPU,

```text
X012C345
```

is the FPO.

The FPO is located on the heat spreader.

Importantly, it is different from `SR2L2`.

```text
SR2L2    → S-Spec
X012C345 → FPO
```

Even in Intel's Warranty Information, the FPO is entered as the "Batch number."

## What is ATPO?

ATPO stands for "Assembly Test Process Order," and it is treated as the CPU's Serial Number.

While the FPO is a batch number, the ATPO is a number to identify each individual processor.

Intel CPUs have a very small **2D Matrix** on the outer edge of the substrate.

Decoding this 2D Matrix allows you to obtain the Full ATPO (complete serial number).

## Reading a 2D Matrix with a Smartphone

This time, I was able to read the code using the "Code Scan - QR & Barcode Scanner" app from the App Store on my iPhone.

https://apps.apple.com/jp/app/code-scan-qr-barcode-scanner/id1554812545

Because the Intel CPU code is very small, it may not be recognized properly by a regular QR code reader. Intel also states that you can read the 2D Matrix using a compatible app that utilizes your smartphone's camera.

When reading, it helps to:

*   Clean the surface of the CPU to remove any dirt.
*   Take the picture in a well-lit area.
*   If there is reflection, shine a light on it from an angle.
*   Focus on the code.
*   If necessary, use macro photography.
*   **Cover the parts other than the 2D Matrix with dark paper or objects to increase the contrast between the code and the board.**

This will increase the chances of success.

In particular, in the tests I conducted, **covering the parts other than the code with something dark and making the 2D Matrix stand out made it easier to recognize.**

The CPU board has characters, wiring patterns, and metal parts, so when viewed from the camera, there are many fine patterns in addition to the 2D Matrix. Therefore, the method of covering the surroundings to hide them and making only the code you want to read stand out is worth trying when the scan is unstable.

With this i7-6700, I scanned it with my iPhone using this method as well, and the app gave me the following result:

```text
U6SG012345678
```

What you need is the `Content` side.

Therefore, the Full ATPO of this CPU is:

```text
U6SG012345678
```

Note that `Hex` is not a different serial number.

For example:

```text
55 → U
36 → 6
53 → S
47 → G
```

It is a hexadecimal representation of the characters in the Content string.

## Finding the Partial ATPO

Next, carefully examine the outer edge of the CPU substrate.

On this CPU, a very small string:

```text
45678
```

was printed.

This is the Partial ATPO.

According to Intel, the Partial ATPO is the last 3-5 characters of the Full ATPO.

Therefore, compare it with the Full ATPO read from the Data Matrix earlier.

```text
Full ATPO
U6SG012345678
        ↓↓↓↓↓
        45678

Partial ATPO on the substrate
45678
```

**The last 5 characters matched perfectly.**

This is an important checkpoint.

Intel also advises, as a procedure for checking counterfeit CPUs, to read the Full ATPO from the 2D Matrix and compare the last 5 characters with the Partial ATPO printed on the outer edge of the CPU. If it is genuine, they should match.

Therefore, in this CPU:

```text
Data Matrix
     ↓
U6SG012345678
        ↓
      45678
        ↑
"45678" on the substrate
```

This correspondence was confirmed.

At least, the **Data Matrix on the substrate and the Partial ATPO that humans can read are correctly matched.**

## Can the Heat Spreader and Substrate be Directly Matched?

This is the part I was particularly concerned about while researching.

On the heat spreader side, there is:

```text
i7-6700
SR2L2
X012C345
```

On the other hand, on the substrate side, there is:

```text
Full ATPO:
U6SG012345678

Partial ATPO:
45678
```

In summary:

```text
[Heat Spreader]
i7-6700
SR2L2
FPO: X012C345
       │
       │ ← This is what I wanted to confirm
       │
[CPU Substrate]
Full ATPO: U6SG012345678
Partial ATPO: 45678
```

Regarding Full ATPO and Partial ATPO:

```text
U6SG012345678
        ↓↓↓↓↓
        45678
```

This can be physically confirmed.

However, there is no publicly available rule that users can use to calculate and confirm the relationship between:

```text
FPO X012C345
      ↕
ATPO U6SG012345678
```

If you can successfully search Intel's Warranty Information, you can confirm the combination of FPO and ATPO, but in this i7-6700, no search results were found.

Therefore, **I could not completely confirm that the heat spreader was originally attached to this substrate at the factory.**

## How to Check for "A Fake with an i7-6700 Heat Spreader Placed on a Different CPU"?

For example, consider the following:

```text
Different CPU substrate
   ＋
Heat spreader labeled i7-6700
```

This is a form of forgery.

In this case, it will look like an i7-6700 just by looking at the surface.

In this case, what is effective is the identification information inside the CPU.

Intel also advises that if you can install the CPU in a PC, use the **Intel Processor Diagnostic Tool** to check:

```text
Genuine Intel: Pass
```

and that the product name matches.

Furthermore, with CPU-Z and HWiNFO, check:

*   Core i7-6700
*   Skylake
*   4 cores, 8 threads
*   8MB L3
*   R0 stepping
*   CPUID is consistent with Skylake

If you check these, it will be much easier to detect a simple forgery where "the heat spreader says i7-6700, but the inside is a different CPU."

This is different from comparing benchmark scores; it's a method of checking the identification information returned by the CPU itself.

## Results This Time

In this Core i7-6700:

| Check                      | Result    |
| -------------------------- | --------- |
| i7-6700 is printed on the surface | ○         |
| S-Spec `SR2L2`            | Matches Intel's official information |
| 3.40GHZ                   | Matches Intel's specifications |
| FPO `X012C345`            | Confirmed |
| 2D Matrix                 | Successfully read |
| Full ATPO                 | Successfully obtained |
| Partial ATPO              | `45678`   |
| Full ATPO ending          | `45678`   |
| Full/Partial ATPO         | **5 characters match perfectly** |
| Warranty Information       | Could not find the product |
| FPO and ATPO Intel DB check | **Unconfirmed** |

Therefore, within the scope that we could confirm this time, **no information contradicting the fact that it is a genuine Core i7-6700 was found.**

In particular,

**The last 5 characters of the Full ATPO obtained from the Data Matrix and the Partial ATPO printed directly on the substrate matched.**

This is a strong piece of evidence.

However, since the combination of FPO and ATPO could not be confirmed in Intel's Warranty Information,

**We could not completely confirm that the heat spreader and substrate were originally a set from the factory.**

## Summary of How to Check a Used Intel CPU

If you want to check without using performance measurements, the following order is practical:

```text
① Check the model number on the CPU surface
        ↓
② Compare the S-Spec with Intel's official information
        ↓
③ Record the FPO
        ↓
④ Scan the 2D Matrix on the outer edge of the substrate
        ↓
⑤ Obtain the Full ATPO
        ↓
⑥ Find the Partial ATPO on the substrate
        ↓
⑦ Compare with the last 3-5 characters of the Full ATPO
        ↓
⑧ Enter the FPO + ATPO in Intel Warranty Information
        ↓
⑨ If you can install it in a PC, use the
   Intel Processor Diagnostic Tool to check
   for Genuine Intel and the product name
```

In particular, ④-⑦ can be checked without running the CPU.

## Caution: It's Best Not to Publish the ATPO

The ATPO is the CPU's Serial Number.

When publishing photos on blogs or flea market sites, it is best to hide part of it, like this:

```text
U6SG0123*****
```

Also, since it may be possible to read the Full ATPO from the code even if you hide the 2D Matrix itself, it is safe to **put a mosaic on the 2D Matrix as well.**

## Summary

Intel CPUs have multiple identification pieces of information, not just "Core i7" written on them:

**S-Spec → FPO → Data Matrix → Full ATPO → Partial ATPO**

In this i7-6700, we were able to confirm up to:

```text
SR2L2
FPO: X012C345
Full ATPO: U6SG0123*****
Partial ATPO: 45678
```

and the correspondence between Full ATPO and Partial ATPO was also confirmed.

When checking the authenticity of a used CPU, it is useful to use not only benchmark results but also **this method of cross-checking physical traceability information.**

However, this does not cryptographically prove the authenticity of the semiconductor. In particular, with older CPUs, search results may not be found in Intel's current Warranty search, so it is practical to combine:

**Physical marking consistency + ATPO consistency + identification information inside the CPU**

to make a judgment.
