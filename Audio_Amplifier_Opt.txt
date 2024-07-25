# Very simple optimization tool for the localized optimization of a design solution
# for a three-stage amplifer using bipolar transistors
# Hugo Serra
# Department of Electrical and Computer Engineering - NOVA FCT
# Rev. 2024/05/16

import random
import sympy as sp
import timeit
from dateutil.relativedelta import relativedelta as rd

gen    = 100#50     # Number of generations that are going to be executed
pop    = 50#200    # Size of the population, both initial random population and the new generation that is created based on the best genetic material
best   = 15#50      # Number of chromosomes that will be used to create the new generation
mut    = 10         # Mutation factor (+%, -%)
var    = 10         # Variation allowed in the components values (+%, -%)
fitTyp = 1          # (0) - Uses exponential fitness equations; (1) - Uses figure of merit type fitness equation
debug  = 2          # Prints fitness/performance of (0) - No chromosome  (1) - Every chromosome, including those that are bad; (2) - Only valid chromosomes
pDesignSpace = 1    # Prints menu containing initial solution and maximum/minimum allow values for each component to be optimized
pSystemSols  = 1    # Prints the solution of the PFR and MPS systems of equation (only of the elite chromosome)
pDCperline   = 17   # When printing DC operating point, how many parameter it prints per line

intervals = ['days','hours','minutes','seconds']

# Lists used to print the results after each generation
peraux = ["IB [0/1]", "VCE [0/1]", "Power dissipation [W]", "Signal Excursion [V]", "Output Common-Mode [V]", "Av_1 [V/V]", "Av_2 [V/V]", "Av_3 [V/V]"]
varaux = ["VCC", "VEE", "RB", "RF", "RM1", "RM2", "RC1", "RE1", "RC2", "RE2"]

# Circuit goals
Gain_des = 120;               Gain_wgt = 1.5   # Gain optimization
PowD_des = 0.1;               PowD_wgt = 0.2   # Power dissipation optimization
SigE_des = 0.01;              SigE_wgt = 1.0   # Signal excursion optimization

# Circuit parameter validation to ensure active region and signal excursion
VCE_MIN = 1.25   # Value to ensure active region
VC_CLP  = 4.00   # Value to check signal excursion
VO_LIM  = 0.25   # Maximum value (absolute) for the common-mode voltage at the output to consider it a valid solution
VCM = 0          # Reference voltage to find signal excursion

# Correction coefficient to test signal excursion
CCOEF = 0.975

# Circuit parameters (that are fixed) - For better results the code can be integrated with LTspice so that better estimates can be obtained for the
# values of VBE and Beta, since the performance is very sensitive (especially the power dissipation) to variations in the values of VBE
R3_val      = 1.00
VB1_val     = 0.00
VLED_val    = 1.71
VT_val      = 25.8e-03
VA_val      = 55.0
VBE0_val    = 0.623;     VBE1_val    = 0.604;     VBE2_val    = 0.697;     VBE2m_val    = 0.694;     VBE3_val    = 0.650;     VBE4_val    = 0.645
BetaDC0_val = 471;       BetaDC1_val = 483;       BetaDC2_val = 436;       BetaDC2m_val = 423;       BetaDC3_val = 198;       BetaDC4_val = 183
BetaAC0_val = 468;       BetaAC1_val = 482;       BetaAC2_val = 379;       BetaAC2m_val = 390;       BetaAC3_val = 199;       BetaAC4_val = 181

# Circuit parameters (that are to be optimized) - First column of values (RB_val, RF_val, etc, represents the initial solution that is used by the optimizer)
VCC_val =  4.50;               VCC_max =  4.5;                                  VCC_min =  4.5;                                  VCC_step = 0.1
VEE_val = -4.50;               VEE_max = -4.5;                                  VEE_min = -4.5;                                  VEE_step = 0.1
RB_val  =  1.470e+03;          RB_max  = round(RB_val*(1+var/100),0);           RB_min  = round(RB_val*(1-var/100),0);           RB_step  = 0.001e+03
RF_val  =  0.997e+03;          RF_max  = round(RF_val*(1+var/100),0);           RF_min  = round(RF_val*(1-var/100),0);           RF_step  = 0.001e+03
RM1_val =  0.491e+03;          RM1_max = round(RM1_val*(1+var/100),0);          RM1_min = round(RM1_val*(1-var/100),0);          RM1_step = 0.001e+03
RM2_val =  0.549e+03;          RM2_max = round(RM2_val*(1+var/100),0);          RM2_min = round(RM2_val*(1-var/100),0);          RM2_step = 0.001e+03
RC1_val =  2.005e+03;          RC1_max = round(RC1_val*(1+var/100),0);          RC1_min = round(RC1_val*(1-var/100),0);          RC1_step = 0.001e+03
RE1_val =  5.00e+00;          RE1_max = 5.00e+00;                             RE1_min = 5.00e+00;                             RE1_step = 0.001e+03
RC2_val =  0.257e+03;          RC2_max = round(RC2_val*(1+var/100),0);          RC2_min = round(RC2_val*(1-var/100),0);          RC2_step = 0.001e+03
RE2_val =  0.023e+03;          RE2_max = round(RE2_val*(1+var/100),0);          RE2_min = round(RE2_val*(1-var/100),0);          RE2_step = 0.001e+03

if pDesignSpace == 1:   # Prints initial solution and maximum/minimum allowed values in the design space for each component to be optimized
    print("------------------------------------------------------------------------------------------------------------------")
    print("| Design space     |  VCC |  VEE |   RB    |   RF    |   RM1   |   RM2   |   RC1   |   RE1   |   RC2   |   RE2   |")
    print("| Initial solution | "'{:4.3}'.format(VCC_val) +" | "'{:4.3}'.format(VEE_val) +" | "'{:7.6}'.format(RB_val) +" | "'{:7.6}'.format(RF_val) +" | "'{:7.6}'.format(RM1_val) +" | "'{:7.6}'.format(RM2_val) +" | "'{:7.6}'.format(RC1_val) +" | "'{:7.6}'.format(RE1_val) +" | "'{:7.6}'.format(RC2_val) +" | "'{:7.6}'.format(RE2_val) + " |")
    print("| Maximum value    | "'{:4.3}'.format(VCC_max) +" | "'{:4.3}'.format(VEE_max) +" | "'{:7.6}'.format(RB_max) +" | "'{:7.6}'.format(RF_max) +" | "'{:7.6}'.format(RM1_max) +" | "'{:7.6}'.format(RM2_max) +" | "'{:7.6}'.format(RC1_max) +" | "'{:7.6}'.format(RE1_max) +" | "'{:7.6}'.format(RC2_max) +" | "'{:7.6}'.format(RE2_max) + " |")
    print("| Minimum value    | "'{:4.3}'.format(VCC_min) +" | "'{:4.3}'.format(VEE_min) +" | "'{:7.6}'.format(RB_min) +" | "'{:7.6}'.format(RF_min) +" | "'{:7.6}'.format(RM1_min) +" | "'{:7.6}'.format(RM2_min) +" | "'{:7.6}'.format(RC1_min) +" | "'{:7.6}'.format(RE1_min) +" | "'{:7.6}'.format(RC2_min) +" | "'{:7.6}'.format(RE2_min) + " |")
    print("------------------------------------------------------------------------------------------------------------------")



####################################################################################################
######## Function to obtain DC operating point, solving it numerically, differential system ########
####################################################################################################
def pfr_num(idx, VCC, VEE, VB1, VLED, VBE0, VBE1, VBE2, VBE2m, VBE3, VBE4, Beta0, Beta1, Beta2, Beta2m, Beta3, Beta4, RB, RF, RM1, RM2, RC1, RE1, RC2, RE2, R3):   # requires as inputs all numeric variables, unknowns are defined at the top of the code

    # Symbols necessary to solve systems of equations
    VB0, VC0, VE0, VCE0, IB0, IC0, IE0, IRB, ILED = sp.symbols('VB0, VC0, VE0, VCE0, IB0, IC0, IE0, IRB, ILED')   # Current mirror unknowns used to solve the DC operating point system
    VB1a, VC1a, VE1a, VCE1a, IB1a, IC1a, IE1a = sp.symbols('VB1a, VC1a, VE1a, VCE1a, IB1a, IC1a, IE1a')           # Differential pair unknowns used to solve the DC operating point system
    VB1b, VC1b, VE1b, VCE1b, IB1b, IC1b, IE1b = sp.symbols('VB1b, VC1b, VE1b, VCE1b, IB1b, IC1b, IE1b')           # Differential pair unknowns used to solve the DC operating point system
    VB2, VC2, VE2, VEC2, IB2, IC2, IE2 = sp.symbols('VB2, VC2, VE2, VEC2, IB2, IC2, IE2')                         # Common-emitter unknowns used to solve the DC operating point system
    VB2m, VC2m, VE2m, VCE2m, IB2m, IC2m, IE2m = sp.symbols('VB2m, VC2m, VE2m, VCE2m, IB2m, IC2m, IE2m')           # VBE multiplier unknowns used to solve the DC operating point system
    VB3, VC3, VE3, IB3, IC3, IE3 = sp.symbols('VB3, VC3, VE3, IB3, IC3, IE3')                                     # Buffer stage unknowns used to solve the DC operating point system
    VB4, VC4, VE4, IB4, IC4, IE4 = sp.symbols('VB4, VC4, VE4, IB4, IC4, IE4')                                     # Buffer stage unknowns used to solve the DC operating point system
    VO = sp.symbols('VO')                                                                                         # Extra unknowns used to solve the DC operating point system

    eqns = [
        # Current source equations
        (VB0 - VCC)/RB + ILED + IB0,                                      # Nodal equation from node VB0
        (VE0 - VEE)/RF - IB0 - Beta0*IB0,                                 # Nodal equation from node VE0

        VBE0 - VB0 + VE0,                                                 # VBE relation to nodes VB0 and VE0
        VCE0 - VC0 + VE0,                                                 # VCE relation to nodes VC0 and VE0

        IC0 - Beta0*IB0,                                                  # Current relation for transistor in active region
        IE0 - (Beta0 + 1)*IB0,                                            # Current relation for transistor in active region

        VLED - VBE0 - (Beta0 + 1)*IB0*RF,                                 # Mesh equation through the LED and the BE junction of transistor Q0
        IRB - (VCC - VB0)/RB,                                             # Ohm's law

        # Differential pair equations
        (VC0 - VE1a)/RE1 + (VC0 - VE1b)/RE1 + IC0,                        # Nodal equation from node VC0

        (VE1a - VC0)/RE1 - IB1a - Beta1*IB1a,                             # Nodal equation from node VE1a
        (VE1b - VC0)/RE1 - IB1b - Beta1*IB1b,                             # Nodal equation from node VE1b

        (VC1a - VCC)/RC1 + Beta1*IB1a,                                    # Nodal equation from node VC1a
        (VC1b - VCC)/RC1 + Beta1*IB1b - IB2,                              # Nodal equation from node VC1b

        VB1a - VB1,                                                       # Variable equivalence
        VB1b - VB1,                                                       # Variable equivalence

        VBE1 - CCOEF*VB1 + VB1 + VE1a,                                    # VBE relation to nodes VB1a and VE1a
        VBE1 + CCOEF*VB1 - VB1 + VE1b,                                    # VBE relation to nodes VB1b and VE1b

        VCE1a - VC1a + VE1a,                                              # VCE relation to nodes VC0 and VE0
        VCE1b - VC1b + VE1b,                                              # VCE relation to nodes VC0 and VE0

        IC1a - Beta1*IB1a,                                                # Current relation for transistor in active region
        IC1b - Beta1*IB1b,                                                # Current relation for transistor in active region

        IE1a - (Beta1 + 1)*IB1a,                                          # Current relation for transistor in active region
        IE1b - (Beta1 + 1)*IB1b,                                          # Current relation for transistor in active region
        
        # Common-emitter equations
        (VE2 - VCC)/RE2 + IB2 + (Beta2 + 1)*IB2,                          # Nodal equation from node VE2
        -Beta2*IB2 + IB3 + Beta2m*IB2m + (VC2 - VB2m)/RM1,                # Nodal equation from node VC2

        VB2 - VC1b,                                                       # Variable equivalence

        VBE2 - VE2 + VC1b,                                                # VEB relation to nodes VE2 and VB2
        VEC2 - VE2 + VC2,                                                 # VEC relation to nodes VE2 and VC2

        IC2 - Beta2*IB2,                                                  # Current relation for transistor in active region
        IE2 - (Beta2 + 1)*IB2,                                            # Current relation for transistor in active region

        # VBE multiplier equations
        (VB2m - VC2)/RM1 + (VB2m - VE2m)/RM2 + IB2m,                      # Nodal equation from node VB2m
        (VE2m - VB2m)/RM2 - (Beta2m + 1)*IB2m + (VE2m - VEE)/RC2 - IB4,   # Nodal equation from node VE2m

        VC2 - VC2m,                                                       # Variable equivalence

        VBE2m - VB2m + VE2m,                                              # VBE relation to nodes VB2m and VE2m
        VCE2m - VC2 + VE2m,                                               # VCE relation to nodes VC2m and VE2m

        IC2m - Beta2m*IB2m,                                               # Current relation for transistor in active region
        IE2m - (Beta2m + 1)*IB2m,                                         # Current relation for transistor in active region

        # Buffer stage equation
        -(Beta3 + 1)*IB3 + (VE3 - VO)/R3,                                 # Nodal equation from node VE3
        (VO - VE3)/R3 + (VO - VE4)/R3,                                    # Nodal equation from node VO
        (Beta4 + 1)*IB4 + (VE4 - VO)/R3,                                  # Nodal equation from node VE4

        VB3 - VC2,                                                        # Variable equivalence
        VB4 - VE2m,                                                       # Variable equivalence

        VC3 - VCC,                                                        # Variable equivalence
        VC4 - VEE,                                                        # Variable equivalence

        VBE3 - VC2 + VE3,                                                 # VBE relation to nodes VB3 and VE3
        VBE4 - VE4 + VE2m,                                                # VBE relation to nodes VB4 and VE4

        IC3 - Beta3*IB3,                                                  # Current relation for transistor in active region
        IC4 - Beta4*IB4,                                                  # Current relation for transistor in active region

        IE3 - (Beta3 + 1)*IB3,                                            # Current relation for transistor in active region
        IE4 - (Beta4 + 1)*IB4,                                            # Current relation for transistor in active region
    ]

    varList = ["VB0", "VC0", "VE0", "VCE0", "IB0", "IC0", "IE0", "VB1a", "VC1a", "VE1a", "VCE1a", "IB1a", "IC1a", "IE1a", "VB1b", "VC1b", "VE1b", "VCE1b", "IB1b", "IC1b", "IE1b", "VB2", "VC2", "VE2", "VEC2", "IB2", "IC2", "IE2", "VB2m", "VC2m", "VE2m", "VCE2m", "IB2m", "IC2m", "IE2m", "VB3", "VC3", "VE3", "IB3", "IC3", "IE3", "VB4", "VC4", "VE4", "IB4", "IC4", "IE4", "VO", "ILED", "IRB"]
    sysSol = list(sp.linsolve(eqns, [VB0, VC0, VE0, VCE0, IB0, IC0, IE0, VB1a, VC1a, VE1a, VCE1a, IB1a, IC1a, IE1a, VB1b, VC1b, VE1b, VCE1b, IB1b, IC1b, IE1b, VB2, VC2, VE2, VEC2, IB2, IC2, IE2, VB2m, VC2m, VE2m, VCE2m, IB2m, IC2m, IE2m, VB3, VC3, VE3, IB3, IC3, IE3, VB4, VC4, VE4, IB4, IC4, IE4, VO, ILED, IRB]))
    solList = dict(map(lambda i,j : (i,j) , varList, sysSol[0]))

    if (pSystemSols == 1 and idx == -1):
        printDC(varList, sysSol)

    return solList
####################################################################################################


####################################################################################################
######### Function to obtain small signal model, solving both numerically and symbolically #########
####################################################################################################
def mps_num(idx, pfr, vin, Beta1, Beta2, Beta2m, Beta3, Beta4, RC1, RE1, RC2, RE2, R3, RM1, RM2):   # requires as inputs all numeric variables

    vc1, ve1, ib1 = sp.symbols('vc1, ve1, ib1')                                       # Differential pair unknowns used to solve the small signal model system
    vc2, ve2, ib2, vb2m, ve2m, ib2m = sp.symbols('vc2, ve2, ib2, vb2m, ve2m, ib2m')   # Common-emitter and VBE multiplier unknowns used to solve the small signal model system
    ve3, ib3, ve4, ib4, vo = sp.symbols('ve3, ib3, ve4, ib4, vo')                     # Buffer stage unknowns used to solve the small signal model system

    Rpi0  = VT_val/pfr['IB0'];    Ro0  = VA_val/pfr['IC0']
    Rpi1  = VT_val/pfr['IB1b'];   Ro1  = VA_val/pfr['IC1b']
    Rpi2  = VT_val/pfr['IB2'];    Ro2  = VA_val/pfr['IC2']
    Rpi2m = VT_val/pfr['IB2m'];   Ro2m = VA_val/pfr['IC2m']
    Rpi3  = VT_val/pfr['IB3'];    Ro3  = VA_val/pfr['IC3']
    Rpi4  = VT_val/pfr['IB4'];    Ro4  = VA_val/pfr['IC4']

    resList = ["Rpi0", "Ro0", "Rpi1", "Ro1", "Rpi2", "Ro2", "Rpi2m", "Ro2m", "Rpi3", "Ro3", "Rpi4", "Ro4"]
    resSol  = [(Rpi0, Ro0, Rpi1, Ro1, Rpi2, Ro2, Rpi2m, Ro2m, Rpi3, Ro3, Rpi4, Ro4)]

    eqns = [
        # Differential pair equations
        vc1/RC1 + Beta1*ib1 + (vc1 - ve2)/Rpi2 + (vc1 - ve1)/Ro1,     # Nodal equation at the colector of transistor Q1
        -Beta1*ib1 + (ve1 - vin)/Rpi1 + ve1/RE1 + (ve1 - vc1)/Ro1,    # Nodal equation at the emitter of transistor Q1
        ib1 - (vin - ve1)/Rpi1,                                       # Ohm's law in resistor Rpi1

        # Common-emitter equations
        Beta2*ib2 + (ve2 - vc1)/Rpi2 + ve2/RE2 + (ve2 - vc2)/Ro2,     # Nodal equation at the emitter of transistor Q2
        -Beta2*ib2 + (vc2 - ve2)/Ro2 + (vc2 - ve2m)/Ro2m + Beta2m*ib2m + (vc2 - vb2m)/RM1 + (vc2 - ve3)/Rpi3,        # Nodal equation at the colector of transistor Q2/Q2MVBE
        ib2 - (ve2 - vc1)/Rpi2,                                       # Ohm's law in resistor Rpi2

        # VBE multiplier equations
        -Beta2m*ib2m + (ve2m - vc2)/Ro2m + ve2m/RC2 + (ve2m - vb2m)/Rpi2m + (ve2m - ve4)/Rpi4 + (ve2m - vb2m)/RM2,   # Nodal equation at the emitter of transistor Q2MVBE
        (vb2m - vc2)/RM1 + (vb2m - ve2m)/Rpi2m + (vb2m - ve2m)/RM2,   # Nodal equation at the base of transistor Q2MVBE
        ib2m - (vb2m - ve2m)/Rpi2m,                                   # Ohm's law in resistor Rpi2m

        # Buffer stage equation
        (ve3 - vo)/R3 + (ve3 - vc2)/Rpi3 - Beta3*ib3 + ve3/Ro3,       # Nodal equation at the emitter of transistor Q3
        ib3 - (vc2 - ve3)/Rpi3,                                       # Ohm's law in resistor Rpi3

        (ve4 - vo)/R3 + (ve4 - ve2m)/Rpi4 + Beta4*ib4 + ve4/Ro4,      # Nodal equation at the emitter of transistor Q4
        ib4 - (ve4 - ve2m)/Rpi4,                                      # Ohm's law in resistor Rpi4

        (vo - ve3)/R3 + (vo - ve4)/R3,                                # Nodal equation at the circuits output node
    ]

    varList = ["vc1", "ve1", "ib1", "vc2", "ve2", "ib2", "vb2m", "ve2m", "ib2m", "ve3", "ib3", "ve4", "ib4", "vo"]
    sysSol  = list(sp.linsolve(eqns, [vc1, ve1, ib1, vc2, ve2, ib2, vb2m, ve2m, ib2m, ve3, ib3, ve4, ib4, vo]))
    solList = dict(map(lambda i,j : (i,j) , varList, sysSol[0]))

    if (pSystemSols == 1 and idx == -1):
        printAC(varList, sysSol, resList, resSol)

    return solList
####################################################################################################



####################################################################################################
######### Function to print the results obtained from the PRF and MPS system of equations ##########
####################################################################################################
def printDC(varList, sysSol):

    if (len(sysSol[0]) < pDCperline):
        aux = [0, len(sysSol[0])]
    if (len(sysSol[0]) > pDCperline):
        aux = [-1]
        for i in range(-(-len(sysSol[0])//pDCperline)):
            if ((i+1)*pDCperline < len(sysSol[0])):
                aux.append((i+1)*pDCperline-1)
            else:
                aux.append(len(sysSol[0])-1)

    print("DC operating point results: ", end='')

    for y in range(len(aux)-1):
        print("\n-", end='')
        for x in range((aux[y+1]-aux[y])*11): print("-", end='')
        print("\n"+"| ", end='')
        for x in range(aux[y]+1, aux[y+1]+1):
            print(f"  {varList[x]}", end='')
            if (varList[x][0].upper() == 'V' and len(varList[x]) == 2): print("     | ", end='')
            if (varList[x][0].upper() == 'V' and len(varList[x]) == 3): print("    | ", end='')
            if (varList[x][0].upper() == 'V' and len(varList[x]) == 4): print("   | ", end='')
            if (varList[x][0].upper() == 'V' and len(varList[x]) == 5): print("  | ", end='')
            if (varList[x][0].upper() == 'I' and len(varList[x]) == 2): print("     | ", end='')
            if (varList[x][0].upper() == 'I' and len(varList[x]) == 3): print("    | ", end='')
            if (varList[x][0].upper() == 'I' and len(varList[x]) == 4): print("   | ", end='')
        print("\n"+"| ", end='')
        for x in range(aux[y]+1, aux[y+1]+1):
            if (varList[x][0].upper() == 'V'): print(" "+'{:+.3f}'.format(round(sysSol[0][x], 3))+"  | ", end='')
            else: print('{:+.2e}'.format(sysSol[0][x])+" | ", end='')
        print("\n-", end='')
        for x in range((aux[y+1]-aux[y])*11): print("-", end='')
    print('')    

    return 1

def printAC(varList, sysSol, resList, resSol):

    # Print AC voltage/current information
    aux = len(sysSol[0])

    print("\nAC operating point results: ", end='')
    print("\n-", end='')

    for x in range(aux*11): print("-", end='')

    print("\n"+"| ", end='')

    for x in range(aux):
        print(f"  {varList[x]}", end='')
        if (varList[x][0].upper() == 'V' and len(varList[x]) == 2): print("     | ", end='')
        if (varList[x][0].upper() == 'V' and len(varList[x]) == 3): print("    | ", end='')
        if (varList[x][0].upper() == 'V' and len(varList[x]) == 4): print("   | ", end='')
        if (varList[x][0].upper() == 'V' and len(varList[x]) == 5): print("  | ", end='')
        if (varList[x][0].upper() == 'I' and len(varList[x]) == 2): print("     | ", end='')
        if (varList[x][0].upper() == 'I' and len(varList[x]) == 3): print("    | ", end='')
        if (varList[x][0].upper() == 'I' and len(varList[x]) == 4): print("   | ", end='')

    print("\n"+"| ", end='')

    for x in range(aux):
        print('{:+.2e}'.format(sysSol[0][x])+" | ", end='')

    print("\n-", end='')

    for x in range(aux*11): print("-", end='') 

    print('')

    # Print AC resistance information
    aux = len(resSol[0])

    for x in range(aux*11): print("-", end='')

    print("-\n"+"| ", end='')

    for x in range(aux):
        print(f"  {resList[x]}", end='')
        if (resList[x][1].lower() == 'p' and len(resList[x]) == 4): print("   | ", end='')
        if (resList[x][1].lower() == 'p' and len(resList[x]) == 5): print("  | ", end='')
        if (resList[x][1].lower() == 'o' and len(resList[x]) == 3): print("    | ", end='')
        if (resList[x][1].lower() == 'o' and len(resList[x]) == 4): print("   | ", end='')
        if (resList[x][1].lower() == 'I' and len(resList[x]) == 2): print("     | ", end='')
        if (resList[x][1].lower() == 'I' and len(resList[x]) == 3): print("    | ", end='')
        if (resList[x][1].lower() == 'I' and len(resList[x]) == 4): print("   | ", end='')

    print("\n"+"| ", end='')

    for x in range(aux):
        print('{:+.2e}'.format(resSol[0][x])+" | ", end='')

    print("\n-", end='')

    for x in range(aux*11): print("-", end='')

    print('')

    return 1
####################################################################################################



####################################################################################################
####################################################################################################
####################################################################################################
def sig_exc(pfrSol, mpsSol):
    VAt1a_p, VAt1a_n, VAt1b_p, VAt1b_n, VAt2_p, VAt2_n = sp.symbols('VAt1a_p, VAt1a_n, VAt1b_p, VAt1b_n, VAt2_p, VAt2_n')

    eqns = [pfrSol['VC1a'] + 2*VAt1a_p * abs(mpsSol['vc1']) - VC_CLP, pfrSol['VC1a'] - 2*VAt1a_n * abs(mpsSol['vc1']) + VC_CLP, 
            pfrSol['VC1b'] + 2*VAt1b_p * abs(mpsSol['vc1']) - VC_CLP, pfrSol['VC1b'] - 2*VAt1b_n * abs(mpsSol['vc1']) + VC_CLP, 
            pfrSol['VC2']  + 2*VAt2_p  * abs(mpsSol['vc2']) - VC_CLP, pfrSol['VC2']  - 2*VAt2_n  * abs(mpsSol['vc2']) + VC_CLP]
    
    excResult = list(sp.linsolve(eqns, [VAt1a_p, VAt1a_n, VAt1b_p, VAt1b_n, VAt2_p, VAt2_n]))

    varList = ["VAt1a_p", "VAt1a_n", "VAt1b_p", "VAt1b_n", "VAt2_p", "VAt2_n"]
    excResult = dict(map(lambda i,j : (i,j) , varList, excResult[0]))

    return min(excResult.items(), key=lambda x: abs(VCM - x[1]))
####################################################################################################



####################################################################################################
########################### Function where the chromosomes are evaluated ###########################
####################################################################################################
def crmeval_num(idx, VCC, VEE, RB, RF, RM1, RM2, RC1, RE1, RC2, RE2):

    # Get DC operating point
    pfrSolution = pfr_num(idx, VCC, VEE, VB1_val, VLED_val, VBE0_val, VBE1_val, VBE2_val, VBE2m_val, VBE3_val, VBE4_val, BetaDC0_val, BetaDC1_val, BetaDC2_val, BetaDC2m_val, BetaDC3_val, BetaDC4_val, RB, RF, RM1, RM2, RC1, RE1, RC2, RE2, R3_val)   # Can get values to the system of solutions using keys

    # Get AC parameters
    mpsSolution = mps_num(idx, pfrSolution, 0.5, BetaAC1_val, BetaAC2_val, BetaAC2m_val, BetaAC3_val, BetaAC4_val, RC1, RE1, RC2, RE2, R3_val, RM1, RM2)   

    # Signal excursion is calculated based on the DC voltage in nodes VC1/VC2 and based on the gain of the first and second stage
    excSolution = sig_exc(pfrSolution, mpsSolution)

    crmResult = []

    # Verification to see if all base currents are positive to ensure transistors are ON
    crmResult.append(1) if all( [pfrSolution['IB0'] > 0, pfrSolution['IB1a'] > 0, pfrSolution['IB1b'] > 0, pfrSolution['IB2'] > 0, pfrSolution['IB2m'] > 0, pfrSolution['IB3'] > 0, pfrSolution['IB4'] > 0] ) else crmResult.append(0)

    # Verification to see if all VCE voltages are higher than a given threshold to ensure active region
    crmResult.append(1) if all( [pfrSolution['VCE0'] > VCE_MIN, pfrSolution['VCE1a'] > VCE_MIN, pfrSolution['VCE1b'] > VCE_MIN, pfrSolution['VEC2'] > VCE_MIN, pfrSolution['VCE2m'] > VCE_MIN] ) else crmResult.append(0)

    crmResult.append(round((VCC-VEE)*(pfrSolution['IRB'] + pfrSolution['IC1a'] + pfrSolution['IC1b'] + pfrSolution['IE2'] + pfrSolution['IC3']), 6))   # Power dissipation calculation
    crmResult.append(round(excSolution[1], 6))      # Signal excursion
    crmResult.append(round(pfrSolution['VO'], 3))   # Output common-mode voltage

    crmResult.append(round(mpsSolution['vc1'], 2))   # AC gain magnitude at the output of the first stage
    crmResult.append(round(mpsSolution['vc2'], 2))   # AC gain magnitude at the output of the second stage
    crmResult.append(round(mpsSolution['vo'], 2))    # AC gain magnitude at the output of the third stage

    varList = ["IB", "VCE", "PD", "SE", "VCM_OUT", "Gain_1", "Gain_2", "Gain_3"]
    crmResult = dict(map(lambda i,j : (i,j) , varList, crmResult))

    return crmResult
####################################################################################################



####################################################################################################
######################################## Auxilary function ########################################
####################################################################################################
def round_crm(x, max, min, step):   # Used to round chromosome values to a given step and to not allow them to go over/under their maximum/minimum value respectively

    ans = round(x/step)*step

    return max if ans > max else min if ans < min else ans
####################################################################################################



####################################################################################################
################## Exponential fitness function - V: Value, D: Desired, W: Weight ##################
####################################################################################################
def maxfitness(V, D, W):
    return 1.0 - sp.exp(-W * V / D)

def minfitness(V, D, W):
    return 1.0 - sp.exp(- D / V / W)

def equfitness(V, D, W):
    return 2.0 / ( sp.exp(-W * (V - D) / D) + sp.exp(W * (V - D) / D) )
####################################################################################################



####################################################################################################
################## Fitness function where performance of the solution is obtained ##################
####################################################################################################
def fitness(idx, VCC, VEE, RB, RF, RM1, RM2, RC1, RE1, RC2, RE2):

    ans = crmeval_num(idx, VCC, VEE, RB, RF, RM1, RM2, RC1, RE1, RC2, RE2)

    if (ans['IB'] == 1 and ans['VCE'] == 1 and abs(ans['VCM_OUT']) <= VO_LIM):
        if fitTyp == 0:
            fitRes = maxfitness(ans['Gain_3'], Gain_des, Gain_wgt)*minfitness(ans['PD'], PowD_des, PowD_wgt)*maxfitness(abs(ans['SE']), SigE_des, SigE_wgt)
            if (debug == 1 or debug == 2): print(f"Chromosome result: {ans} and Fitness (EXP): {round(fitRes, 3)}")
        elif fitTyp == 1:
            fitRes = ans['Gain_3'] * abs(ans['SE']) / ans['PD']
            if (debug == 1 or debug == 2): print(f"Chromosome result: {ans} and Fitness (FoM): {round(fitRes, 3)}")
        else:
            print(f"Variable \"fitTyp\" with value {fitTyp} can only take a 0/1 value")
            exit()
        return fitRes
    else:
        if (debug == 1): print("Chromosome result: Not within the desired IB/VCE/VCM_OUT range, ignored as possible solution")
        return 0
####################################################################################################



####################################################################################################
########################################### MAIN PROGRAM ###########################################
####################################################################################################
start = timeit.default_timer()

# Generate solutions
solutions = []

# Use initial solution as a individual
solutions.append( (round_crm(VCC_val, VCC_max, VCC_min, VCC_step),
                        round_crm(VEE_val, VEE_max, VEE_min, VEE_step),
                        round_crm(RB_val, RB_max, RB_min, RB_step),
                        round_crm(RF_val, RF_max, RF_min, RF_step),
                        round_crm(RM1_val, RM1_max, RM1_min, RM1_step),
                        round_crm(RM2_val, RM2_max, RM2_min, RM2_step),
                        round_crm(RC1_val, RC1_max, RC1_min, RC1_step),
                        round_crm(RE1_val, RE1_max, RE1_min, RE1_step),
                        round_crm(RC2_val, RC2_max, RC2_min, RC2_step),
                        round_crm(RE2_val, RE2_max, RE2_min, RE2_step)))

for i in range(pop-1):
    # creating solutions to be evaluated comprised of three values (x,y,z)
    solutions.append( (round_crm(random.uniform(VCC_min, VCC_max), VCC_max, VCC_min, VCC_step),
                        round_crm(random.uniform(VEE_min, VEE_max), VEE_max, VEE_min, VEE_step),
                        round_crm(random.uniform(RB_min, RB_max), RB_max, RB_min, RB_step),
                        round_crm(random.uniform(RF_min, RF_max), RF_max, RF_min, RF_step),
                        round_crm(random.uniform(RM1_min, RM1_max), RM1_max, RM1_min, RM1_step),
                        round_crm(random.uniform(RM2_min, RM2_max), RM2_max, RM2_min, RM2_step),
                        round_crm(random.uniform(RC1_min, RC1_max), RC1_max, RC1_min, RC1_step),
                        round_crm(random.uniform(RE1_min, RE1_max), RE1_max, RE1_min, RE1_step),
                        round_crm(random.uniform(RC2_min, RC2_max), RC2_max, RC2_min, RC2_step),
                        round_crm(random.uniform(RE2_min, RE2_max), RE2_max, RE2_min, RE2_step)))

for i in range(gen):
    k = 0
    rankedsolutions = []
    for j in solutions:
        rankedsolutions.append( (fitness(k, j[0], j[1], j[2], j[3], j[4], j[5], j[6], j[7], j[8], j[9]), j) )
        k += 1

    if(i == 0):
        print(f"\n========== Initial solution  ==========")
        varList = dict(map(lambda i,j : (i,j) , varaux, rankedsolutions[0][1]))
        perList = dict(map(lambda i,j : (i,j) , peraux, list(crmeval_num(-1, varList['VCC'], varList['VEE'], varList['RB'], varList['RF'], varList['RM1'], varList['RM2'], varList['RC1'], varList['RE1'], varList['RC2'], varList['RE2']).values())))
        print(f"Fitness: {round(rankedsolutions[0][0], 3)}, Performance: {perList}\nVariables: {varList}")

    rankedsolutions.sort()
    rankedsolutions.reverse()

    print(f"\n====================================================================================================")
    print("============================== Best solution found in generation #"'{:3}'.format(i+1)+" ==============================")
    print(f"====================================================================================================")
    varList = dict(map(lambda i,j : (i,j) , varaux, rankedsolutions[0][1]))
    perList = dict(map(lambda i,j : (i,j) , peraux, list(crmeval_num(-1, varList['VCC'], varList['VEE'], varList['RB'], varList['RF'], varList['RM1'], varList['RM2'], varList['RC1'], varList['RE1'], varList['RC2'], varList['RE2']).values())))
    print(f"Fitness: {round(rankedsolutions[0][0], 3)}, Performance: {perList}\nVariables : {varList}")

    bestsolutions = rankedsolutions[:best]

    elem_0, elem_1, elem_2, elem_3, elem_4, elem_5, elem_6, elem_7, elem_8, elem_9 = ([] for _ in range(10))

    for j in bestsolutions:
        elem_0.append(j[1][0])
        elem_1.append(j[1][1])
        elem_2.append(j[1][2])
        elem_3.append(j[1][3])
        elem_4.append(j[1][4])
        elem_5.append(j[1][5])
        elem_6.append(j[1][6])
        elem_7.append(j[1][7])
        elem_8.append(j[1][8])
        elem_9.append(j[1][9])

    newGen = []
    newGen.append((elem_0[0], elem_1[0], elem_2[0], elem_3[0], elem_4[0], elem_5[0], elem_6[0], elem_7[0], elem_8[0], elem_9[0]))

    for _ in range(pop-1):
        e0 = random.choice(elem_0) * random.uniform(1-mut/100,1+mut/100);     e0 = round_crm(e0, VCC_max, VCC_min, VCC_step)   # VCC
        e1 = random.choice(elem_1) * random.uniform(1-mut/100,1+mut/100);     e1 = round_crm(e1, VEE_max, VEE_min, VEE_step)   # VEE
        e2 = random.choice(elem_2) * random.uniform(1-mut/100,1+mut/100);     e2 = round_crm(e2, RB_max, RB_min, RB_step)      # RB
        e3 = random.choice(elem_3) * random.uniform(1-mut/100,1+mut/100);     e3 = round_crm(e3, RF_max, RF_min, RF_step)      # RF
        e4 = random.choice(elem_4) * random.uniform(1-mut/100,1+mut/100);     e4 = round_crm(e4, RM1_max, RM1_min, RM1_step)   # RM1
        e5 = random.choice(elem_5) * random.uniform(1-mut/100,1+mut/100);     e5 = round_crm(e5, RM2_max, RM2_min, RM2_step)   # RM2
        e6 = random.choice(elem_6) * random.uniform(1-mut/100,1+mut/100);     e6 = round_crm(e6, RC1_max, RC1_min, RC1_step)   # RC1
        e7 = random.choice(elem_7) * random.uniform(1-mut/100,1+mut/100);     e7 = round_crm(e7, RE1_max, RE1_min, RE1_step)   # RE1
        e8 = random.choice(elem_8) * random.uniform(1-mut/100,1+mut/100);     e8 = round_crm(e8, RC2_max, RC2_min, RC2_step)   # RC2
        e9 = random.choice(elem_9) * random.uniform(1-mut/100,1+mut/100);     e9 = round_crm(e9, RE2_max, RE2_min, RE2_step)   # RE2

        newGen.append((e0,e1,e2,e3,e4,e5,e6,e7,e8,e9))

    solutions = newGen

x = rd(seconds=round(timeit.default_timer()-start))
print("Optimization time:",' '.join('{} {}'.format(getattr(x,k),k) for k in intervals if getattr(x,k)))
