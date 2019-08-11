# Load Switch

Charge controllers are often equipped with a load switch that protects the battery from deep-discharge. If supported by the charge controller, the load switch can also be used for advanced control of connected loads, e.g. switching on lights during the night.

Even though it might seem very simple to switch on and off a load using a MOSFET, a reliable load switch control can get quite complicated. The load switch needs to protect itself and the charge controller from overcurrents and from exceeding its thermal limits. In addition to that, it must protect the wires from short circuits, acting like an electronic fuse. These different protection features are described below in more detail.

## High-side vs. low-side

::: warning TODO
- Circuit diagrams
- Advantages and disadvantages
:::

## Inrush current during turn-on

- Slew-rate limitation using gate capacitor

https://www.onsemi.com/pub/Collateral/AND9093-D.PDF
https://www.rohm.com/electronics-basics/transistors/what-is-a-load-switch

- PWM switching

See also: https://community.victronenergy.com/articles/2992/load-output-of-mppt-charge-controllers.html

## Overcurrent

The maximum continuous current the load switch can handle is mainly determined by the thermal limits of the MOSFET. The relevant temperature is the junction temperature $T_j$ which has to stay below the maximum temperature stated in the datasheet. Typical values are around 150 °C.

### Thermal model

As the junction temperature cannot be measured directly, the following thermal model provides a method to estimate it based on the measured current flow and control the load switch accordingly.

The heat flow $\dot{Q}$ dissipated in the MOSFET is calculated from its on-state resistance $R_{DS(on)}$ and the current $i$ going through it:

$$\dot{Q}(i)=R_{DS(on)} \cdot i^2$$

For simplicity reasons, the on-state resistance is assumed to be independent of the temperature, so the worst case resistance should be considered for a safe estimation.

The dissipated heat is partly stored in the thermal capacity of the MOSFET and its surrounding PCB area, considered as a single thermal mass $mc_p$. Another part of the heat is transferred to the ambient, described by the thermal resistance between junction and ambient, $R_{th,ja}$.

Using an energy balance, the junction temperature $T_{j,k+1}$ at a time step $k + 1$ can be calculated based on the temperature of the previous step $T_{j,k}$, the ambient temperature $T_a$ and the average heat dissipated during the period $\Delta t$ between two steps:

$$T_{j,k+1} = T_{j,k} + \frac{1}{mc_p} \left(\dot{Q}(i) - \frac{T_{j,k} - T_a}{R_{th,ja}} \right) \Delta t$$

With the thermal time constant of the system $\tau = mc_p \cdot R_{th,ja}$ the equation can be rewritten as:

$$T_{j,k+1} = T_{j,k} + \left(\frac{\dot{Q}(i) \cdot R_{th,ja}}{\tau} - \frac{T_{j,k} - T_a}{\tau} \right) \Delta t$$

The time constant depends on the PCB design, the used components and attached heat sinks. It is normally quite small, in the order of a few seconds. It can be determined from a measured temperature response after a load step or estimated based on experience with similar designs.

As the circuit should be designed for a certain maximum constant current $i_{max}$ at a maximum expected ambient temperature $T_{a,max}$, the steady-state thermal resistance $R_{th,ja}$ can be expressed as:

$$R_{th,ja} = \frac{T_{j,max} - T_{a,max}}{\dot{Q}(i_{max})} = \frac{T_{j,max} - T_{a,max}}{R_{ds(on)} \cdot i_{max}^2}$$

The maximum junction temperature $T_{j,max}$ for the layout should be chosen with some safety margin to the value specified in the datasheet.

This leads to the final temperature model equation:

$$T_{j,k+1} = T_{j,k} + \left( \frac{i^2}{i_{max}^2}(T_{j,max} - T_{a,max}) - (T_{j,k} - T_a) \right) \frac{\Delta t}{\tau}$$

For the implementation in the microcontroller, the previous temperature $T_{j,k}$ has to be stored, with a starting condition equal to the ambient temperature $T_a$. $\Delta t$ is the time between each call of the function and $i$ is the measured current through the MOSFET. The other values are constant for a given design and can be determined during testing.

## Short circuit

Normally, there should be a fuse installed at the battery terminal to protect the wires from short circuits, which also protects the load output. However, even if the fuse rating at the battery is not much higher than the rating of the load output, fuses react quite slow. Typical [automotive blade fuses](https://www.littelfuse.com/~/media/automotive/datasheets/fuses/passenger-car-and-commercial-vehicle/blade-fuses/littelfuse_atof_datasheet.pdf) for example need more than 10 ms to melt even for approx. 10x the rated current.

MOSFET datasheets normally state a so-called safe operating area, which shows the maximum allowed current vs. the drain-source-voltage. The following example is from the [Nexperia PSMN5R2-60YL datasheet](https://assets.nexperia.com/documents/data-sheet/PSMN5R2-60YL.pdf):

![Safe operating area of PSMN5R2-60YL](./images/mosfet-safe-operating-area.png)

As long as the maximum junction temperature is not exceeded, this MOSFET can handle 100A continuously. However, under real operating conditions with passive cooling via the PCB, the continuous current will be in the range of 20A. During a short-circuit, the current rises very quickly, only limited by the impedance of the battery, the wires and the PCB.

According to [this study](https://www.sbsbattery.com/PDFs/VRLAshortCurrentsStorageBatterySystems.pdf), a dead short circuit of VRLA lead-acid batteries in the range of 100 Ah can reach currents >2000 A within less than 10 ms. So the MOSFET could get destroyed before any fuse reacted.

<!--
Above study also states that the time constant to reach a steady state current flow und dead short conditions is in the order of 4 ms. This means that the electronic short circuit protection using the MOSFET has to trigger at least an order of magnitude faster.

For the following calculations, we require the firmware to detect a short circuit and switch off the MOSFET within less than 100 µs.
-->

<!--
The battery internal resistance is in the range of 50 m&#8486; for a 60-100 Ah battery (see also [here](https://eu.industrial.panasonic.com/sites/default/pidseu/files/downloads/files/18-292_vrla_whitepaper_interactive.pdf))
-->

### Inrush current when connecting capacitive loads

It would be a simple task to just switch off immediately as soon as a current above a certain limit is reached. However, this sort of protection would create false positives when connecting a load with a high input capacitance. A capacitance acts as a short circuit in the very moment of connection, but the current decreases quickly after the capacitor got charged.

::: warning TODO
Describe boundary conditions and solutions.
:::

## Overvoltage

If the battery voltage rises above allowed limits for some reasons, the load should be switched off. This is a trivial task and will work reliably as long as there is enough margin to the maximum $V_ds$ of the MOSFET.

However, switching off an inductive load like a motor or a relay can result in voltage peaks at the load switch. This is the reason why most charge controllers don't allow switching of inductive loads.

In order to allow the current to decay slowly, a free-wheeling diode $D_{fw}$ can be added to the load output. Such a circuit is shown in the following schematic:

![Load output with freewheeling diode](./images/load-switch-freewheeling-diode.svg)

A freewheeling diode in the charge controller should only be seen as an additional protection measure. Ideally, the diode should be located as close to the load itself as possible.
