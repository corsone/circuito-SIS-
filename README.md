This is a project for the Computer Architecture course that I completed during the first year of my Bachelor's degree in Computer Science.

# Project specifications

The project involves the development of a sequential circuit that controls a chemical apparatus designed to adjust an initial solution with a known pH to a neutral pH. The pH value is expressed on a scale from 0 to 14. The circuit controls two dispensing valves: one for acidic solution and one for basic solution.

If the initial solution is acidic, the circuit will dispense the basic solution until the final solution reaches the neutral threshold (pH between 7 and 8). Similarly, if the initial solution is basic, the circuit will dispense the acidic solution until the pH reaches neutrality. An acidic pH is defined as a value strictly less than 7, while a basic solution is one with a pH strictly greater than 8.

The pH value is encoded in fixed-point representation, with 4 bits allocated for the integer part and the remaining bits for the decimal part.

The two valves have different dispensing rates. The valve for the basic solution dispenses an amount that raises the initial pH by 0.25 per clock cycle. The valve for the acidic solution dispenses an amount that lowers the initial pH by 0.50 per clock cycle.

# The Overall Architecture of the Circuit

The circuit consists of an FSM (controller) and a datapath (processor).

The circuit has 3 inputs:
- **RST** (1 bit)
- **START** (1 bit)
- **pH** (8 bits, 4 for the integer part and 4 for the decimal part)

The outputs are as follows:
- **FINE_OPERAZIONE** (1 bit)
- **ERRORE_SENSORE** (1 bit)
- **VALVOLA_ACIDO** (1 bit)
- **VALVOLA_BASICO** (1 bit)
- **PH_FINALE** (8 bits)
- **NCLK** (8 bits)

The mechanism operates as follows:
- When the **RST** signal is asserted, the system resets to the Reset state, setting all output ports to zero.
- To proceed, the system receives the **START** signal with a value of 1 and the initial pH signal for a single clock cycle. The system can then begin the processing phase.
- If the initial solution is acidic, the system opens the basic solution valve by setting the corresponding output to 1. Similarly, if the initial solution is basic, the system opens the acidic solution valve by setting the **VALVOLA_ACIDO** output to 1.
- The system keeps the valves open for the duration needed to reach the neutrality threshold (calculated by the system).
- Once the operation is completed, the system closes all open valves, outputs the final pH value on the **PH_FINALE** port, and asserts the **FINE_OPERAZIONE** output port.
- The **NCLK** port reports the number of clock cycles needed to bring the solution to neutrality.
If the pH value is invalid (greater than 14), the system must report an error by asserting the **ERRORE_SENSORE** output.

# The State Diagram of the Controller(FSM)

## Input
The FSM has the following inputs:

- **RESET**: A signal provided by the user.
- **START**: A signal provided by the user.
- **PH**: Represents the pH value, also provided by the user.
- **FINE**: A signal coming from the datapath that indicates when the FSM output should be set to 1.

## Output
The outputs from the FSM are:

- **FINE_OPERAZIONE**: A signal that directly goes to the output, indicating that the pH has reached the neutral threshold.
- **ERRORE**: A signal that directly goes to the output, indicating that the pH value entered is greater than 14.
- **VA** and **VB**: Signals indicating which valve should be opened based on the current state; these signals also go directly to the output.
- **INIZIO**: This signal goes to the datapath and is used to determine whether to use the value from the register (if 0) or the pH value (if 1).
- **c_s1** and **c_s0**: These two signals go to the datapath and are used to indicate the current state of the FSM.

## States
The FSM we designed consists of four states: Initial, Acidic, Basic, and Neutral.

### Initial State
The Initial state remains in itself in two cases:
- If the project is not started, meaning if the RESET signal is set to 1 or the START signal is set to 0, all outputs will be set to 0.
- If RESET is 0 and START is 1 but an error pH value is provided, the system will assert the ERRORE signal to 1.

Transitioning from the Initial state to one of the other states depends on the type of pH value provided:
- **Acidic pH**: Transitions to the Acidic state and activates the Basic valve (VA) by setting it to 1.
- **Basic pH**: Transitions to the Basic state and activates the Acid valve (VB) by setting it to 1.
- **Neutral pH**: Transitions to the Neutral state and asserts the FINE_OPERAZIONE signal to indicate the completion of the operation.

### Acidic State

- **Remaining in Acidic State**: Once in the Acidic state, it will stay in this state if the `FINE` input from the datapath is 0, regardless of the pH value. In this case, all outputs are "don't care" (irrelevant).
  
- **Transition to Neutral State**: If the `FINE` input is 1, it indicates that the pH has transitioned from acidic to neutral. Therefore, the FSM transitions to the Neutral state and asserts the `FINE_OPERAZIONE` signal to 1.

- **Transition to Initial State**: If the `RESET` signal is set to 1, the FSM transitions to the Initial state, setting all outputs to 0.

- **Handling New pH Input**: To input a new pH value before completing the operation, you must first set `RESET` to 1 and then insert the new pH value. If the new pH value is input without resetting the system first, it will not be considered until the next clock cycle after `RESET` has been set to 1.

### Basic State

- **Remaining in Basic State**: In this state, similar to the Acidic state, the FSM will remain in the Basic state if the `FINE` input from the datapath is 0, regardless of the pH value. In this case, all outputs are "don't care" (irrelevant).

- **Transition to Neutral State**: If the `FINE` input is 1, it indicates that the pH has transitioned from basic to neutral. The FSM then transitions to the Neutral state and asserts the `FINE_OPERAZIONE` signal to 1.

- **Transition to Initial State**: If the `RESET` signal is set to 1, the FSM transitions to the Initial state, setting all outputs to 0.

- **Handling New pH Input**: To input a new pH value before completing the operation, you must first set `RESET` to 1 and then insert the new pH value. If a new pH value is input without resetting the system first, it will not be considered until the next clock cycle after `RESET` has been set to 1.

# Datapath

The datapath, in addition to the 8-bit pH signal, also receives the `START`, `c_s1i`, and `c_s0i` signals from the FSM. The latter two signals are passed through a register to avoid creating loops within the circuit.

At the beginning of the datapath, there is a 2-input multiplexer (mux) that manages the entry of the pH value into the datapath based on the `START` signal. The output of the mux is then fed into three different components:

1. **Adder with Constant 0.25**: This component adds 0.25 to the pH value.
2. **Subtractor with Constant 0.50**: This is implemented using an adder to subtract 0.50 from the pH value.
3. **4-Input Multiplexer**: This mux decides which result of the operations to consider. It receives as input:
   - The initial pH value in case of an error or if the pH is already neutral.
   - The pH value increased by 0.25 if the pH is acidic.
   - The pH value decreased by 0.50 if the pH is basic.

The selection of the mux is controlled by the `c_s1i` and `c_s0i` signals from the FSM.

The output of this multiplexer is then fed into components that check if the pH is within the range of 7 to 8. This mux, controlled by `c_s1i` and `c_s0i`, outputs the `FINE` signal, which will be:
- `1` if the pH is already neutral or has become neutral after the operations.
- `0` if there are errors or if the pH has not yet reached neutrality.

Additionally, the output of this mux is passed through a register, and the output of this component is then fed into the initial mux, ensuring that the initial pH is not considered during circuit execution. The `FINE` signal is then fed into two additional 2-input multiplexers as selectors:

1. **Final pH Output Mux**: This mux decides when to output the final pH value. Until `FINE` is set to `1`, it will output `0`.

2. **Clock Cycle Counter Mux**: Located on the right side of the datapath in the figure, this mux controls the clock cycle counter. Similarly, until `FINE` is set to `1`, it will output `0`.

The clock cycle counter works as follows:
- The first mux is controlled by the `START` signal. As long as `START` is set to `1`, the counter does not begin counting.
- The counter also has a second mux to manage when to increment by 1 or not add anything (in cases of errors or when the pH is already neutral).

The outputs of the two multiplexers are then fed into an adder. The output of this adder is input into both the initial mux of the counter and the mux responsible for outputting the final number of clock cycles.