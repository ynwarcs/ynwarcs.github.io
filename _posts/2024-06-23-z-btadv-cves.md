---
layout: post
title: Two bluetooth vulnerabilities in Windows
master_id: bthadv-2022
---
<script>
addCustomTypes([]);
</script>

Back in 2020, [Andy Nguyen](https://twitter.com/theflow0?lang=en) released [Bleeding Tooth](https://google.github.io/security-research/pocs/linux/bleedingtooth/writeup.html), a set of vulnerabilities in the Linux Bluetooth subsystem. Along with the write-up, a functional exploit was also released, which could achieve RCE on a physically proximate machine with no user interaction required. I'd only found out about these in autumn of 2022, but thought it may be worth looking into the Windows implementation of components which were found vulnerable. I didn't expect to find much, especially since the information had been out there for two years and other people would've surely already had a look, but I figured I could afford to waste a few days on inspecting some of the components in the worst case.

I ended up finding two vulnerabilities and reporting them to Microsoft, who released the fixes in the March 2023 update. The CVEs assigned are [CVE-2023-24871](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2023-24871) and [CVE-2023-23388](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2023-23388), with the former being disclosed as an RCE and the latter as an LPE. The reality is a little more complicated, as the first vulnerability existed in multiple places throughout the Bluetooth stack on Windows, and could be used both as an RCE vector and as an LPE vector, which the advisory failed to mention. The second vulnerability was indeed only an LPE, and is quite less interesting than the first one. The first of the two vulnerabilities is one of my favourites, because it allowed an RCE and an LPE at the same time, the RCE can be achieved with no user interaction, and the manifestation of the vulnerability provides a lot of control over the memory corruption. That's quite rare nowadays. This is why the major focus in this series will be on the first vulnerability. The second vulnerability is a run-of-the-mill Windows LPE and wouldn't really warrant a post normally, but I found it accidentally while writing up an exploit for the first vulnerability, so I felt it deserves to be a part of the story as well.

# tl;dr
----

<details markdown="1"><summary>Spoilers here! If you care about them, skip ahead.</summary>
- CVE-2023-24871 is an integer overflow vulnerability in the Bluetooth Low Energy library in Windows. The vulnerability is in the code which counts the amount of advertising sections in a Bluetooth advertisement data packet - since an unsigned 8-bit integer is used for this purpose, having more than 255 advertisement sections in the packet will trigger an integer overflow. This will lead to a miscalculation of the amount of memory required for processing the sections and will trigger a heap-based out-of-bounds write, allowing the attacker to corrupt memory with a high degree of control.
- The affected library is linked statically into multiple modules in Windows Bluetooth stack, including a kernel driver and usermode DLLs used by privileged services. The vulnerable function is used to parse advertisement data coming from both remote devices and local applications, meaning that the vulnerability can be used as a vector for both RCE & LPE.
- Microsoft fixed the vulnerability in March 2023 Patch Tuesday, by explicitly detecting the overflow and exiting the function in that case. They failed to acknowledge the LPE component of the vulnerability, providing only an RCE advisory.
- CVE-2023-23388 is an input validation failure in the Bluetooth Service, in code which parses user-mode requests made by applications via RPC. A parameter is treated as an index into an array of functions, and the corresponding function is then called. The parameter was treated as a signed integer during validation, which only ensured that the value is smaller than the size of the array of functions. An attacker could supply a negative value of the parameter and successfully pass the validation, having the code interpret an unexpected value as a function pointer and call it immediately. This could be used as an LPE attack vector in the same manner as the other vulnerability.
- This vulnerability was also fixed in March 2023 Patch Tuesday, by ensuring that the value of the parameter is non-negative.
</details>

# content
----
I could probably cram everything I've written into a single post, but felt it'd be better structured through individual posts. These are:
- [CVE-2023-24871 - intro & vulnerability description](/y-cve-2023-24871-intro-descr), which gives a brief introduction into Bluetooth Low Energy and advertisements, and then describes CVE-2023-24871.
- [CVE-2023-24871 - RCE](/x-cve-2023-24871-rce), which explains how the vulnerability can be used as an RCE vector. I provide a PoC but no functional exploit.
- [CVE-2023-24871 - LPE](/w-cve-2023-24871-lpe), which explains how the vulnerability can be used as an LPE vector. A functional exploit is provided.
- [CVE-2023-23388](/v-cve-2023-23388), which talks about the other vulnerability. Once again a PoC is provided but no functional exploit.

*[← Home](/)* | *[CVE-2023-24871 - intro & vulnerability description →](/y-cve-2023-24871-intro-descr)*