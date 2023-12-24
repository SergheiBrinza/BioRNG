# BioRNG — True Random Number Generation from Live Fish Behavior

> A computer-vision entropy extraction system that turns the unpredictable movement of aquarium fish into a cryptographically verifiable stream of random bits.

![BioRNG Lab Setup](images/biorng-lab.jpg)

---

In 2012, a team from UC San Diego and the University of Michigan scanned 6.2 million public RSA keys harvested from the open internet. What they found caused quiet panic: 0.2% of all HTTPS keys shared a prime factor with at least one other key. The algorithm wasn't broken. RSA worked exactly as designed. The problem was weak entropy — devices were generating keys from predictable seeds because they didn't have enough randomness at boot time. Routers, embedded systems, headless servers — machines with no mouse, no keyboard, no human typing at unpredictable intervals. Machines that booted into a deterministic state and needed cryptographic keys *right now*.

That paper (Heninger et al., "Mining Your Ps and Qs", USENIX Security 2012) sent shockwaves through the security community. And the fundamental problem it exposed hasn't gone away. If anything, it's gotten worse — the number of IoT devices, virtual machines, and embedded controllers that need cryptographic randomness has grown by orders of magnitude since then. The entropy pool is running dry.

This project attacks the problem from an unexpected angle. An aquarium full of fish.

---

## What entropy actually means, and why you should care

The word gets thrown around loosely, so let's be precise. In cryptography, entropy is a measure of how unpredictable a data source is. Claude Shannon formalized this in 1948 with the information entropy formula:

$$H(X) = -\sum_{i=1}^{n} p(x_i) \log_2 p(x_i)$$

where $p(x_i)$ is the probability of outcome $x_i$. Flip a fair coin — each outcome has probability 0.5, and Shannon entropy is exactly 1 bit per flip. Maximum uncertainty. Maximum entropy.

But Shannon entropy assumes you know the probability distribution. Security applications need a more conservative measure — min-entropy:

$$H_{\min}(X) = -\log_2 \left( \max_i \, p(x_i) \right)$$

Min-entropy captures the worst case — the probability of the single most likely outcome. If an attacker gets one guess, min-entropy tells you how likely that guess is to succeed. The NIST SP 800-90B standard, the definitive document for evaluating entropy sources in cryptographic applications, is built entirely around min-entropy estimation.

Here's what matters for this project: every modern cryptographic system — TLS, SSH, VPN, disk encryption, digital signatures, blockchain — depends on a pseudorandom number generator (PRNG) seeded with true entropy. A PRNG is deterministic. It stretches a short seed into a long stream of bits that *look* random. But if you know the seed, you know every bit the PRNG will ever produce. The entire security chain hangs on one question: was the seed truly unpredictable?

Enter the fish.

---

## Why a living organism makes a good entropy source

The standard approach to hardware entropy is physical noise: thermal fluctuations in resistors, shot noise in diodes, phase jitter in ring oscillators, quantum effects in photodetectors. All of these work. Intel's RDRAND instruction, built into every x86 processor since Ivy Bridge, samples an on-die hardware noise source. But there's a catch — you can't inspect the source. You're trusting a black box inside Intel's silicon. And in 2019, a group from Worcester Polytechnic Institute showed that electromagnetic interference can bias an oscillator-based TRNG from outside the chip, leaving no trace in standard statistical tests.

A living organism offers something different. Fish locomotion in an aquarium is the output of a massively parallel biological computation: sensory processing (vision, lateral line mechanoreception, olfaction), motor planning, proprioceptive feedback, social signaling between individuals, circadian modulation, stochastic kinetics of ion channels in neurons. The state space is enormous. The system is nonlinear. And — this is the critical part — the noise floor has a biological, not electronic, origin. You can't inject frequencies into a fish's nervous system the way you can into a ring oscillator.

The question is whether this biological noise actually produces enough entropy to be useful. And whether it can be extracted cleanly.

Short answer: yes and yes. But the details matter.

---

## Prior work — and how this project differs

The idea of using an aquarium for randomness generation isn't entirely new. Katyal, Mishra and Baluni published a short paper in 2013 ("True Random Number Generator using Fish Tank Image", IJCA Vol. 78, No. 16) where they photographed an aquarium and hashed raw pixel data to seed a linear congruential generator. In 2019, Jason Fenech at the University of London built BubblesRNG — an OpenCV-based system that tracked air bubbles in a water container and XORed their coordinates into a random stream.

Both projects share a fundamental limitation: they treat the aquarium as a black box of pixels. No animal tracking. No behavioral modeling. No formal entropy assessment. No NIST SP 800-90B evaluation. The pixel-hashing approach mixes real biological entropy with camera sensor noise, JPEG compression artifacts, and lighting fluctuations — in the end, you're measuring your camera, not the fish.

BioRNG works differently. The system tracks individual fish in 3D using deep-learning detection and multi-object tracking. The entropy source isn't pixels. It's the *trajectory* of each animal — coordinates, velocities, accelerations, inter-individual distances, turning angles. These are behavioral measurements with a clear biological origin. And they can be formally assessed for min-entropy using NIST-approved methods.

This makes BioRNG, to the best of our knowledge, the first system that combines real-time neural-network multi-object animal tracking with formal cryptographic entropy verification.

---

## Hardware

The physical setup is deliberately simple. The complexity in BioRNG lives in the biology, not the hardware.

The system uses 2 to 3 synchronized machine vision cameras: one positioned directly above the aquarium (top view), one or two on the sides (lateral view). The top camera captures each fish's $(x, y)$ position in the horizontal plane. The side camera gives the $z$ coordinate (depth). Together they reconstruct the full 3D trajectory.

Why not one camera? A top-down view loses vertical information. Fish frequently change depth — rising to the surface, diving to the bottom — and these depth changes are driven by different neural circuits than horizontal swimming. A single 2D projection discards roughly a third of available kinematic entropy.

Both cameras are synchronized via a hardware trigger (a simple microcontroller generating a common TTL pulse) or software PTP (Precision Time Protocol) over Ethernet.

Operating frame rate: 30 to 60 fps. Above 60 fps adds minimal entropy, because the locomotion bandwidth of fish is limited — dominant frequencies in tail-beat kinematics are below 30 Hz (Muller & van Leeuwen, J. Exp. Biol., 2004). Sampling at 60 fps satisfies the Nyquist theorem with margin.

Infrared ring LED illumination at 850 nm, mounted coaxially with the top camera. Fish don't perceive light at 850 nm — their visual sensitivity cuts off around 750 nm — so IR illumination has zero effect on behavior. The IR-cut filter is removed from the camera. This yields clean, high-contrast images without interference from visible room lighting or circadian light cycles.

---

## How many fish — and why it matters

There's a mathematical answer to this question. The rate of entropy generation depends on the number of independently moving animals, the dimensionality of the tracked state, and the frame rate.

If we track $N$ fish, each contributing 3 spatial coordinates $(x, y, z)$ at frame rate $f$ (fps), the raw data rate before entropy extraction is:

$$R_{\text{raw}} = N \times 3 \times f \quad \text{values per second}$$

For $N = 6$ fish at 60 fps, that's 1080 coordinate values per second. If each value contributes at least a few bits of min-entropy (say, 2–4 bits from the least significant digits of each coordinate after whitening), the system produces on the order of 2000–4000 bits/sec of true entropy. Modest compared to a dedicated hardware TRNG (which can output megabits per second), but more than sufficient as a seed source for a CSPRNG.

The optimal number of fish is a compromise. Too few (1–3): low entropy rate, long periods of inactivity, no inter-individual interactions. Too many (>10–12): tracking becomes unreliable, occlusion frequency scales as roughly $O(N^2)$. The sweet spot is 4 to 8 fish — keeps occlusion manageable (a well-tuned ByteTrack tracker maintains >95% identity consistency at $N \leq 8$ in a 40x25 cm tank), provides enough entropy from inter-individual interactions, and guarantees that at least 2–3 fish are active at any given moment.

The relationship between tracked objects and usable entropy can be estimated empirically, but a useful theoretical anchor comes from the joint entropy of $N$ weakly correlated sources:

$$H(X_1, X_2, \ldots, X_N) = \sum_{i=1}^{N} H(X_i) - \sum_{i < j} I(X_i; X_j) + \text{higher-order terms}$$

where $I(X_i; X_j)$ is the mutual information between fish $i$ and $j$. In schooling species, mutual information is nonzero — fish coordinate their movements — but it doesn't dominate. Empirical measurements of transfer entropy in pairs of zebrafish (Butail et al., Entropy, 2014) show that only 5–15% of each fish's behavioral entropy is shared with its nearest neighbor. The rest is independent. Adding fish gives nearly linear entropy gain up to the tracking reliability limit.

---

## Software pipeline

Stage 1 uses YOLOv8 (Ultralytics, 2023) for per-frame object detection, fine-tuned on 800–1200 annotated frames of the specific setup. Stage 2 links detections across frames via ByteTrack (Zhang et al., ECCV 2022), producing a time series for each fish:

$$\mathbf{s}_i(t) = \left[ x_i(t), \, y_i(t), \, z_i(t), \, \dot{x}_i(t), \, \dot{y}_i(t), \, \dot{z}_i(t) \right]$$

Stage 3 extracts entropy-significant features at each time step: instantaneous speed $v_i(t) = \sqrt{\dot{x}_i^2 + \dot{y}_i^2 + \dot{z}_i^2}$, turning angle $\theta_i(t)$, acceleration magnitude, inter-individual distances $d_{ij}(t) = \|\mathbf{r}_i(t) - \mathbf{r}_j(t)\|$, and spatial features (distance to tank centroid, normalized vertical position).

Stage 4 — whitening. Raw features aren't uniformly distributed. Fish spend more time at certain speeds, certain depths, certain distances from the wall. Direct use of these features as random numbers would introduce bias. The standard solution: cryptographic hashing as a conditioner.

$$\text{output}(t) = \text{SHA-256}\left(\mathbf{b}(t) \,\|\, \text{counter}(t)\right)$$

SHA-256 is deterministic — it doesn't create entropy. But it redistributes it uniformly across a 256-bit output. If the input contains at least 256 bits of min-entropy (accumulated over several time steps if needed), the output is computationally indistinguishable from uniform random.

Stage 5 — continuous health testing per NIST SP 800-90B Section 4.4: Repetition Count Test (threshold $C = 1 + \lceil -\log_2(\alpha) / H \rceil$ where $\alpha$ is the false-positive probability, typically $2^{-20}$) and Adaptive Proportion Test. Full offline entropy assessment uses the NIST SP 800-90B test suite (`ea_non_iid` estimator).

---

## Status and open questions

The detection and tracking pipeline is functional. Entropy extraction and NIST assessment are in progress. The core open question: how does min-entropy vary with circadian phase, water temperature, feeding schedule, social stress? Does a sick fish produce less entropy than a healthy one? These are publishable questions at the intersection of information theory and neuroethology.

A second line of inquiry: adversarial robustness. Can an attacker influence fish behavior to reduce entropy output — through vibration, light patterns, chemical stimuli? Characterizing these attacks is a direct analog of the electromagnetic injection attacks studied on silicon TRNGs, but in the biological domain.

---

## References

1. Heninger, N. et al. "Mining Your Ps and Qs: Detection of Widespread Weak Keys in Network Devices." USENIX Security, 2012.
2. Shannon, C. E. "A Mathematical Theory of Communication." Bell System Technical Journal, 27(3), 379–423, 1948.
3. Turan, M. S. et al. "Recommendation for the Entropy Sources Used for Random Bit Generation." NIST SP 800-90B, 2018.
4. Zhang, Y. et al. "ByteTrack: Multi-Object Tracking by Associating Every Detection Box." ECCV, 2022.
5. Jocher, G. et al. "Ultralytics YOLOv8." github.com/ultralytics/ultralytics, 2023.
6. Butail, S. et al. "Information Flow in Animal-Robot Interactions." Entropy, 16(3), 1315–1330, 2014.
7. Muller, U. K. & van Leeuwen, J. L. "Swimming of larval zebrafish." J. Exp. Biol., 207, 853–868, 2004.
8. Katyal, R., Mishra, A. & Baluni, A. "True Random Number Generator using Fish Tank Image." IJCA, 78(16), 2013.
9. Wojke, N., Bewley, A. & Paulus, D. "Simple Online and Realtime Tracking with a Deep Association Metric." ICIP, 2017.

---

*Detection and tracking pipeline functional. Entropy extraction and NIST assessment in progress.*

MIT License.
