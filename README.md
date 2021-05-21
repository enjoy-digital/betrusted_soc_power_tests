```
                   ___      __               __         __    ____     _____
                  / _ )___ / /_______ _____ / /____ ___/ /___/ __/__  / ___/
                 / _  / -_) __/ __/ // (_-</ __/ -_) _  /___/\ \/ _ \/ /__
                /____/\__/\__/_/  \_,_/___/\__/\__/\_,_/   /___/\___/\___/
                     ___  ___ _    _____ ____  / /____ ___ / /____
                    / _ \/ _ \ |/|/ / -_) __/ / __/ -_|_-</ __(_-<
                   / .__/\___/__,__/\__/_/    \__/\__/___/\__/___/
                  /_/
        Power estimations/tests of a LiteX design on Arty board for Betrusted project.
                           Copyright (c) 2021, EnjoyDigital
                             Powered by Migen & LiteX
```
![License](https://img.shields.io/badge/License-BSD%202--Clause-orange.svg)

<p align="center"><img src="https://user-images.githubusercontent.com/1450143/119099192-ef31d780-ba16-11eb-8f01-cb9d9688c6ec.JPG"></p>

[> Intro
---------
This is a small project to do power saving tests/estimations of a LiteX design on an Arty Board and check different power optimizations techniques.

The aim is to provide useful reference data/implementations for the Betrusted project.

[> Designs
-------------

Different designs can be built:
- An empty SoC to measure board's static power:
`./digilent_arty --static --build --load`
- A Simple SoC with 100MHz input --> PLL --> VexRiscv CPU SoC:
`./digilent_arty --build --load`
- The same Simple SoC without the PLL:
`./digilent_arty --no-pll --build --load`

The buttons of the boards are used to easily test the different power optimization:

- PLL reset.
- PLL power_down.
- Clock Gating
- etc...

The measures are done with a simple low-cost USB power meter, so are not very precise but still useful to raise the interesting patterns.

[> Measures
---------------

**SoC w/ and w/o PLL power comparisons:**
|                             | Current | Power  |
|-----------------------------|---------|--------|
| Empty design                | 0.13A   | 0.631W |
| SoC w/ PLL (Running)        | 0.16A   | 0.770W |
| SoC w/ PLL (PLL Down)       | 0.13A   | 0.631W |
| SoC w/ PLL (PLL Reset)      | 0.13A   | 0.631W |
| SoC w/ PLL (Clock Gating)   | 0.15A   | 0.724W |
| SoC w/o PLL (Running)       | 0.14A   | 0.680W |
| SoC w/o PLL (Clock Gating)  | 0.13A   | 0.631W |

**SoC w/ PLL and  varying clock frequency:**
|                             | Current | Power  |
|-----------------------------|---------|--------|
| SoC w/ PLL 100MHz (Running) | 0.16A   | 0.770W |
| SoC w/ PLL 10MHz (Running)  | 0.15A   | 0.724W |
| SoC w/ PLL 200MHz (Running) | 0.17A   | 0.814W |

[> Observations
-------------------
Due to the low cost USB power meter used, measurements are not very precise but raise interesting patterns:

With our Simple SoC at 100MHz:
- PLL consumes ~90mW.
- Logic consumes ~50mW (and seems ~linear with frequency: < 5mW at 10MHz; ~80-90mW at 200MHz).


We are testing a very simple SoC here (11% LUTS/4% Regs) on a XC7A35T, Logic consumption would be probably around ~500mW with the XC7A35T fully used and
still clocked at 100MHz, so a total power or ~590mW with the PLL.

Clock gating is very simple to implement (just set buf="BUFGCE" on S7PLL and provide a custom CE signal), it keeps the system in idle ready to restart when CE is asserted again. On a design extrapolated to full logic utilization, power consumption would go down
from 590mW to nearly 90mW.

Reducing clock speed through custom control of the BUFGCE's CE could also be interesting
to keep the system running and ready re-start at full speed immediately on specific events.
Since still running, the system could even handle the speed throttling directly itself and could be first power saving mode. If possible, this would be easier to implement than DRP
reconfiguration of the output clocks of the PLL.

When the system has to be put in Idle mode, CE could be this time controlled externally and kept low. The system would keep it's internal state and would re-start immediately on CE release.

For a deep power down mode, we saw that PLL consumption is not negligible and we should then also power it down. With the following power down sequence, we could even probably in some cases avoid a system reset:

- **System Up**
- BUFGCE's CE de-assertion.
-  PLL Power Down.
- **Deep Power Down**
- PLL Power Up and wait for lock.
- BUFGCE's CE assertion.
- **System Up**

This still has to be tested/investigated.