# tacaml: OCaml Bindings for TA-Lib

`tacaml` provides OCaml bindings to the [TA-Lib (Technical Analysis Library)](https://ta-lib.org/). This project offers both raw C bindings and higher-level, type-safe wrappers for over 160 technical analysis functions commonly used in financial markets.

## Features

- **Comprehensive Bindings**: Access to over 160 TA-Lib indicators.
- **Type Safety**: Utilizes GADTs to ensure compile-time type matching for indicator inputs and outputs, preventing common errors.
- **Efficient Data Handling**: Integrates with OCaml's `Bigarray` for efficient array operations with C code.
- **Modular Design**: Separated concerns for wrappers, calculation logic, and type-safe packing.
- **Error Handling**: Consistent `Result` types for robust error management.

## Installation

### Prerequisites

Before installing `tacaml`, you must have the TA-Lib C library installed on your system. Refer to the [TA-Lib website](https://ta-lib.org/hdr_install.html) for installation instructions specific to your operating system.

### OCaml Dependencies

This project requires the following OCaml packages:

- `ocaml`
- `dune` (>= 3.17)
- `ctypes`
- `containers`

You can install these using `opam`:

```bash
opam install dune ctypes containers
```

### Building and Installing `tacaml`

Clone the repository:

```bash
git clone https://github.com/hesterjeng/longleaf.git # Replace with actual repo URL if different
cd longleaf
```

Build and install the library:

```bash
dune build
dune install
```

Alternatively, you can use `just` commands if you have `just` installed:

```bash
just install
```

Or, you can install it in `opam`:

```bash
opam install .
```

## Usage

The `Tacaml` module is the primary entry point for the library. It exposes submodules for raw C bindings (`Tacaml.C`), type-safe wrappers (`Tacaml.Safe`), input/output handling, and more.

### Initialization

Before using any TA-Lib functions, you must initialize the library:

```ocaml
let () = 
  match Tacaml.initialize () with
  | Ok () -> print_endline "TA-Lib initialized successfully."
  | Error e -> Printf.eprintf "Failed to initialize TA-Lib: %d\n" e
```

### Calculating Indicators

`tacaml` provides a type-safe way to calculate indicators. You first define an indicator using `Tacaml.Safe`, then prepare your input data and output destinations.

```ocaml
(* Example: Calculate Simple Moving Average (SMA) *)
open Tacaml
open Bigarray

let calculate_sma () = 
  let open Result.Syntax in
  let* () = initialize () in

  (* Define input data (e.g., a Bigarray of floats) *)
  let input_data = Array1.of_array float32 c_layout
    [| 1.0; 2.0; 3.0; 4.0; 5.0; 6.0; 7.0; 8.0; 9.0; 10.0 |]
  in
  let input_source = Input.Source.float input_data in

  (* Define the SMA indicator with a period of 3 *)
  let sma_indicator = Safe.Sma { period = 3 } in

  (* Define output destination *)
  let output_array = Array1.create float32 c_layout (Array1.dim input_data) in
  let output_destination = Output.Destination.float output_array in

  (* Calculate the indicator *)
  let* (start_idx, num_elements) = calculate sma_indicator input_source output_destination in

  Printf.printf "SMA calculated. Start index: %d, Number of elements: %d\n" start_idx num_elements;
  (* Print the output array (for demonstration) *)
  for i = 0 to Array1.dim output_array - 1 do
    Printf.printf "Output[%d]: %f\n" i (Array1.get output_array i)
  done;
  Ok ()

let () = 
  match calculate_sma () with
  | Ok () -> print_endline "SMA calculation complete."
  | Error (`TALibCode e) -> Printf.eprintf "SMA calculation failed with TA-Lib error: %d\n" e
  | Error (`FatalError msg) -> Printf.eprintf "SMA calculation failed with fatal error: %s\n" msg
```

For more examples and detailed usage, refer to the `src/` directory and the generated `odoc` documentation.

### String-based Indicator Parsing

`tacaml` provides a convenient `of_string` function for parsing indicators from string representations. This is particularly useful for configuration files, command-line arguments, or dynamic indicator creation.

```ocaml
(* Parse indicators from strings *)
let parse_indicators () =
  let indicators = [
    "Sma { timeperiod = 20 }";
    "Rsi { timeperiod = 14 }";
    "Macd { fast_period = 12; slow_period = 26; signal_period = 9 }";
    "Bbands { timeperiod = 20; nb_dev_up = 2.0; nb_dev_dn = 2.0; ma_type = SMA }";
    "Cdldoji ()";
    "Stoch { fast_k_period = 5; slow_k_period = 3; slow_k_ma_type = SMA; slow_d_period = 3; slow_d_ma_type = SMA }";
  ] in
  
  List.iter (fun indicator_str ->
    match Tacaml.of_string indicator_str with
    | Ok indicator -> 
      Printf.printf "Parsed: %s -> %s\n" indicator_str (Tacaml.to_string indicator)
    | Error msg -> 
      Printf.printf "Failed to parse '%s': %s\n" indicator_str msg
  ) indicators
```

#### Supported String Formats

The parser supports all 160+ TA-Lib indicators with their respective parameter formats:

**Unit Constructors** (no parameters):
```ocaml
"Avgprice ()"
"Obv ()"
"Cdldoji ()"        (* Candlestick patterns *)
"Ht_dcperiod ()"    (* Hilbert Transform indicators *)
```

**Simple Time Period Constructors**:
```ocaml
"Sma { timeperiod = 20 }"
"Rsi { timeperiod = 14 }"
"Cci { timeperiod = 20 }"
"Dema { timeperiod = 30 }"
"Willr { timeperiod = 14 }"
```

**Complex Multi-Parameter Constructors**:
```ocaml
"Bbands { timeperiod = 20; nb_dev_up = 2.0; nb_dev_dn = 2.0; ma_type = SMA }"
"Macd { fast_period = 12; slow_period = 26; signal_period = 9 }"
"Apo { fast_period = 12; slow_period = 26; ma_type = EMA }"
"Sar { acceleration = 0.02; maximum = 0.2 }"
"T3 { timeperiod = 5; v_factor = 0.7 }"
"Mama { fast_limit = 0.5; slow_limit = 0.05 }"
"Ultosc { timeperiod1 = 7; timeperiod2 = 14; timeperiod3 = 28 }"
```

**Stochastic Oscillators**:
```ocaml
"Stoch { fast_k_period = 5; slow_k_period = 3; slow_k_ma_type = SMA; slow_d_period = 3; slow_d_ma_type = SMA }"
"Stochf { fast_k_period = 5; fast_d_period = 3; fast_d_ma_type = SMA }"
"Stochrsi { timeperiod = 14; fast_k_period = 5; fast_d_period = 3; fast_d_ma_type = SMA }"
```

**Candlestick Patterns with Penetration**:
```ocaml
"Cdlabandonedbaby { penetration = 0.3 }"
"Cdldarkcloudcover { penetration = 0.5 }"
"Cdlmorningstar { penetration = 0.3 }"
```

**Moving Average Types**:
The parser supports all TA-Lib moving average types: `SMA`, `EMA`, `WMA`, `DEMA`, `TEMA`, `TRIMA`, `KAMA`, `MAMA`, `T3`.

#### Error Handling

The parser provides detailed error messages for debugging:
```ocaml
match Tacaml.of_string "Sma { invalid_field = 20 }" with
| Ok _ -> ()
| Error msg -> Printf.printf "Parse error: %s\n" msg
(* Output: Parse error: Expected: { timeperiod = int } *)
```

## Development

### Code Organization

- `src/tacaml.ml`: Main module exposing the public API.
- `src/wrappers.ml`: GADT-based type definitions for all indicators.
- `src/calculate.ml`: Core calculation engine for TA-Lib function calls.
- `src/pack.ml`: Type-safe packing system for input/output matching.
- `src/functions.ml`: Enumeration of all supported TA-Lib functions.
- `src/type.ml`: Type definitions for different output types (Float, Int, Bool).

### Building Documentation

To generate `odoc` documentation:

```bash
dune build @doc
```

The generated documentation will be available in `_build/default/_doc/index.html`.

## Testing

Run tests using Dune:

```bash
dune runtest
```

## Contributing

Contributions are welcome! Please refer to the project's GitHub repository for guidelines on how to contribute.

## License

This project is licensed under the GPL-3.0-or-later License. See the `LICENSE` file for details.
