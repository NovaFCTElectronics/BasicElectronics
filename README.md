# BAALO

BAALO (Bipolar Audio Amplifier Localized Optimization) performs the localized optimization of a design solution for a three-stage amplifier implemented using bipolar transistors.

## Usage

An initial solution and an estimate to the values of $V_{BE}$ and $\beta$ for the initial solution should be given. The reason behind this is because the performance is very sensitive to the value of $V_{BE}$, especially. To get more accurate results, the code can be integrated with LTspice to automatically obtain good estimates to these values. The goal is to fine-tune the initial solution and improve its performance.

If using Visual Studio Code to run the code, simply right click the code and "Run Code" or press <kbd> Ctrl </kbd> +  <kbd> Alt </kbd>  + <kbd> N </kbd>.

## Inputs

There are three types of inputs: **Optimization Related Inputs**, **Performance Related Inputs**, and **Circuit Related Inputs**.

### Optimization Related Inputs:

- *gen*    $\Rightarrow$ Number of generations that are going to be executed;
- *pop*    $\Rightarrow$ Size of the population;
- *best*   $\Rightarrow$ Number of chromosomes which genetic material will be used to create the new generation;
- *mut*    $\Rightarrow$ Mutation factor (+%, -%) limit that is applied to each gene of the chromosome;
- *var*    $\Rightarrow$ Variation allowed in the components values (+%, -%) in relation to the initial solution;
- *fitTyp* $\Rightarrow$ (0) - Uses exponential fitness equations (weights need to be calibrated); (1) - Uses figure of merit type fitness equation (default option);
- *debug*  $\Rightarrow$ Prints fitness/performance of (0) - No chromosome; (1) - Every chromosome, including those that are bad; (2) - Only valid chromosomes (default option).

### Performance Related Inputs:

#### **<ins>Exponential fitness equations (When *fitTyp* = 0):</ins>**

- *Gain_des* $\Rightarrow$ Desired value (goal) for the gain of the amplifier and that is going to be maximized;
- *Gain_wgt* $\Rightarrow$ Weight attributed to the gain indicator;
- *PowD_des* $\Rightarrow$ Desired value (goal) for the power dissipation of the amplifier and that is going to be minimized;
- *PowD_wgt* $\Rightarrow$ Weight attributed to the power dissipation indicator;
- *SigE_des* $\Rightarrow$ Desired value (goal) for the dynamic range of the amplifier and that is going to be maximized;
- *SigE_wgt* $\Rightarrow$ Weight attributed to the dynamic range indicator.

#### **<ins>Figure of Merit fitness equations (When *fitTyp* = 1):</ins>**

When using the Figure of Merit (FoM) fitness equation, no goals are needed, FoM fitness will be maximized:
- $FoM = Gain \times DynamicRange \times PowerDissipation^{-1}$

### Circuit Related Inputs:

#### **<ins>Variables used to discard solutions:</ins>**

- *VCE_MIN* $\Rightarrow$ Minimum voltage value allowed for the $V_{CE}$ of the devices, variable used to ensure that all devices are in the active region;
- *VC_CLP*  $\Rightarrow$ Maximum voltage value (absolute) allowed in the nodes closer to $V_{CC}$ and $V_{EE}$, variable used to check the dynamic range of the amplifier;
- *VO_LIM*  $\Rightarrow$ Maximum voltage value (absolute) for the output common-mode voltage to consider it a valid solution.

#### **<ins>Fixed value variables which do not change throughout the optimization:</ins>**

- *R3_val*   $\Rightarrow$ Resistance value used in the emitter denegration of the push-pull output stage;
- *VB1_val*  $\Rightarrow$ Value used to define the amplifier's input common-mode voltage;
- *VLED_val* $\Rightarrow$ $V_{Don}$ value for the LED used in the current source;
- *VT_val*   $\Rightarrow$ Thermal voltage, used for the calculation of the $r_{\pi}$ resistances;
- *VA_val*   $\Rightarrow$ Early voltage, used for the calculation of the $r_{out}$ resistances.

#### **<ins>Component variables that store the initial solution:</ins>**

- *VCC_val*, *VEE_val* $\Rightarrow$ Variables that define the supply voltages of the amplifier, when using single supply treat the supplies as dual supply (ex. instead of using $V_{CC}$ = 12 V and $V_{EE}$ = 0 V used instead $V_{CC}$ = 6 V and $V_{EE}$ = -6 V, the circuit's performance will be the same);
- *RB_val*,  *RF_val*  $\Rightarrow$ Resistance values for the bias and emitter resistors in the current source stage;
- *RC1_val*, *RE1_val* $\Rightarrow$ Resistance values for the colector and emitter resistors in the differential pair stage;
- *RC2_val*, *RE2_val* $\Rightarrow$ Resistance values for the colector and emitter resistors in the common-emitter stage;
- *RM1_val*, *RM2_val* $\Rightarrow$ Resistance values for the resistors in the $V_{BE}$ multipler stage;

#### **<ins>Component variables value limits and step:</ins>**

To stop the optimizer from straying to far from the initial solution, each component as a maximum and minimum allowed value associated to it, which is represented with the component name and the suffix "_max" and "_min" (they are calculated using the nominal value and previously mentioned variable "var"). This is done so that the currents in each device do not change significantly and the estimates used for $V_{BE}$ and $\beta$ are still moderately good.

Each component also has a variable with suffix "_step" which rounds the component values after mutation depending on the step given.

## Main Contributors

- **Hugo Serra**

## License

This project is licensed under the GPLv3 License - see the project [LICENSE](https://github.com/NovaFCTElectronics/BasicElectronics/blob/main/LICENSE) file for details.
