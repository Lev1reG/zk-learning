# Multiplier ZK Circuit

This project demonstrates a basic zero-knowledge proof using zk-SNARKs with [Circom](https://docs.circom.io/) & [SnarkJS](https://github.com/iden3/snarkjs).

## What is it?

This project implements a simple multiplier circuit that allows a prover to prove that they know two numbers whose product is a given value without revealing the numbers themselves. The verifier can verify that the prover knows these numbers without knowing what the numbers are.

- Deren knows two numbers `a` and `b`.
- The product of these two numbers is `c`, i.e., `c = a * b`.
- Deren does not reveal `a` and `b` to the verifier.
- The verifier can check the proof to confirm that Deren knows `a` and `b` such that `c = a * b` without knowing the values of `a` and `b`.

## Prerequisites

### Install Circom

You need to have rust installed before you can install Circom. If you don't have it, you can install it by following the instructions [here](https://www.rust-lang.org/tools/install)

To install Circom, run the following command:

```bash
cargo install --git https://github.com/iden3/circom
```

### Install SnarkJS

You need to have Node.js & npm installed before you can install SnarkJS. If you don't have them, you can download and install them from [Node.js official website](https://nodejs.org/en/download).

To install SnarkJS, run the following command:

```bash
npm install -g snarkjs
```

### Circuit Logic

```circom
pragma circom 2.0.0;

template Multiplier() {
    signal input a;
    signal input b;
    signal output c;

    c <== a * b;
}

component main = Multiplier();
```

The above code defines a simple multiplier circuit where `a` and `b` are inputs, and `c` is the output which is the product of `a` and `b`.

## How to Reproduce the Steps

### 1. Compile the Circuit

We need to compile the circuit into:

- R1CS file (circuit representation as math equations)
- WASM file (used to compute witness)
- sym (symbolic file for debugging)

To compile the circuit, run the following command:

```bash
mkdir build
circom multiplier.circom --r1cs --wasm --sym -o build/
```

### 2. Generate Powers of Tau

Powers of Tau is a cryptographic setup phase that is required for zk-SNARKs.

First, create a directory for the Powers of Tau files:

```bash
mkdir pot
```

To generate the Powers of Tau, start the ceremony with the following command:

```bash
snarkjs powersoftau new bn128 12 pot/pot12_0000.ptau -v
```

We need to contribute to the Powers of Tau ceremony. This ceremony is a multi-party computation to contribute randomness to the setup phase. To do this, run:

```bash
snarkjs powersoftau contribute pot/pot12_0000.ptau pot/pot12_final.ptau --name="Your Name" -v
```

You can replace `"Your Name"` with your actual name or any identifier you prefer (used as a contribution identifier).

Next, we need to prepare the phase 2 of the Powers of Tau. This step is necessary to change generic `ptau` file to a specific one for our circuit:

```bash
snarkjs powersoftau prepare phase2 pot/pot12_final.ptau pot/final.ptau
```

Finally, we need to run the Groth16 setup to create the proving and verification keys:

```bash
snarkjs groth16 setup build/multiplier.r1cs pot/final.ptau build/multiplier.zkey
```

### 3. Export Verification Key

We need to export the verification key to a JSON file so that it can be used later for verification:

```bash
snarkjs zkey export verificationkey build/multiplier.zkey build/verification_key.json
```

### 4. Prepare the Input File

Create an input file `input.json` with the values for `a` and `b`. For example:

```json
{
  "a": 3,
  "b": 4
}
```

### 5. Compute the Witness

To generate the witness, we need to run the WASM file with the input file:

```bash
node build/multiplier_js/generate_witness.js build/multiplier_js/multiplier.wasm input.json build/witness.wtns
```

This generates a witness file `witness.wtns` that contains the computation's trace.

### 6. Generate the Proof

Now we can generate the proof using the witness and the proving key:

```bash
snarkjs groth16 prove build/multiplier.zkey build/witness.wtns build/proof.json build/public.json
```

### 7. Verify the Proof

Finally, we can verify the proof using the verification key:

```bash
snarkjs groth16 verify build/verification_key.json build/public.json build/proof.json
```

If the proof is valid, you should see a message:

```bash
[INFO]  snarkJS: OK!
```

If the proof is invalid, you will see an error message:

```bash
[ERROR] snarkJS: Invalid proof
```

## How the Prover and Verifier Interact

### 1. Prover Side

The prover uses:

- Circuit (`multiplier.circom`, compiled to `.r1cs`, `.wasm`)
- Proving key (`multiplier.zkey`)
- Their own private inputs (`input.json`)

The prover generate:

- `witness.wtns`: Encoded "secret solution" to the circuit
- `proof.json`: The actual zk-SNARK proof (public, safe to share)
- `public.json`: The public output(s) — what the verifier should know (e.g., c = a \* b)

### 2. Verifier Side

The verifier needs:

- Verification key (`verification_key.json`)
- The proof (`proof.json`)
- The public outputs (`public.json`)

The verifier runs:

```bash
snarkjs groth16 verify build/verification_key.json build/public.json build/proof.json
```

## Acknowledgements

This project is inspired by [https://medium.com/@excid/experimenting-with-zk-snarks-527d13705d3c](https://medium.com/@excid/experimenting-with-zk-snarks-527d13705d3c) with the help of ChatGPT.
