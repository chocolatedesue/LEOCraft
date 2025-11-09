# Shannon-Based Capacity Estimates in LEOCraft

LEOCraft’s FSPL loss model (`LEOCraft/attenuation/fspl.py`) uses the Shannon capacity theorem to translate geometric link distances into GSL bandwidth. The relevant equations are summarized below.

## 1. Free-Space Path Loss & Received Power
Given antenna separation `d` and wavelength `λ = c / f`, the received power in dBm is

$$
P_{\text{rx}}^{\text{dBm}} = P_{\text{tx}}^{\text{dBm}} + G_{\text{tx}}^{\text{dB}} + G_{\text{rx}}^{\text{dB}} - 20\log_{10} \left( \frac{4\pi d}{\lambda} \right)
$$

`G_tx` and `G_rx` can be supplied directly or derived from antenna diameter `D` via

$$
G^{\text{dB}} = 10\log_{10} \left( \eta \left( \frac{\pi D}{\lambda} \right)^2 \right)
$$

where `η` is antenna efficiency (60 % by default).

## 2. Converting Power to Linear Units
To feed Shannon’s formula we translate dBm to watts:

$$
P_{\text{signal}} = 10^{\frac{P_{\text{rx}}^{\text{dBm}} - 30}{10}}
$$

When a satellite simultaneously serves `n` ground stations (the “partition”), LEOCraft divides both bandwidth and signal power by `n` to mimic time/frequency sharing:

$$
B = \frac{B_0}{n}, \qquad P_{\text{signal, eff}} = \frac{P_{\text{signal}}}{n}
$$

## 3. SNR via G/T Ratio
Instead of modeling noise temperature per antenna, the implementation multiplies by the system’s figure of merit `G/T` (linear). Using Boltzmann’s constant `k_B` and absolute temperature implicit in `G/T`, the signal-to-noise ratio becomes

$$
\text{SNR} = \frac{P_{\text{signal, eff}} \cdot (G/T)}{k_B \cdot B}
$$

This collapses gain and noise temperature into a single term while retaining dependence on distance (through `P_signal`) and sharing (through `B`).

## 4. Shannon Capacity
Finally, Shannon’s theorem gives the per-link data rate in bits per second:

$$
C = B \cdot \log_2(1 + \text{SNR})
$$

Capacities are cast to integers and later divided by `1e9` to express Gbps when populating NetworkX edge attributes during `Constellation.connect_ground_station()`.

## Practical Notes
- Shorter distances increase `P_rx`, raising SNR and capacity dramatically; thus shells with lower altitudes or higher gain antennas yield wider GSL pipes.
- The “partition” factor is driven by `sat_coverage[sat_name]`—reducing concurrent ground stations per satellite directly boosts both bandwidth and signal share per link.
- ISLs currently bypass Shannon estimation: they keep a fixed capacity (50 Gbps) while still recording geometric length for routing and stretch metrics.
