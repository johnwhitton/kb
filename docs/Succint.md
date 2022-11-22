# Succint Labs Eth Proof of Consensus Review
- [Succint Labs Eth Proof of Consensus Review](#succint-labs-eth-proof-of-consensus-review)
  - [Overview](#overview)
  - [Trusted Setup](#trusted-setup)
    - [Best Practices for Setup](#best-practices-for-setup)
    - [Trusted Ceremony (Powers of Tau)](#trusted-ceremony-powers-of-tau)
    - [Example Build](#example-build)
  - [circuits](#circuits)
  - [Contracts](#contracts)
    - [Library Contracts](#library-contracts)
    - [Light Client Contracts](#light-client-contracts)
    - [Bridge Contracts](#bridge-contracts)
    - [Additional Contracts](#additional-contracts)
  - [Relayer](#relayer)



## Overview

1. [Demonstration Bridge](https://www.zkbridge.wtf/): Testnet bridge demo connecting Goerli and Gnosis chains.
2. [Succint Blog Oct 29, 2022](https://blog.succinct.xyz/post/2022/10/29/gnosis-bridge/): Proof of Consensus Bridging between Ethereum and Gnosis Chain
> The on-chain light client recreates the light client spec in Solidity (code here). In particular, we implement the process_light_client_finality_update function inside the step function in our smart contract. Then, inside step, where we would typically verify an aggregate BLS signature, we instead replace it with verification of a single Groth16 zkSNARK to reduce gas costs.

> Recall that the validator set of the sync committee rotates every 27 hours. On chain, we keep track of a commitment to the set of validators in the mapping syncCommitteeRootByPeriod. To update this mapping for the next period, we verify the merkle inclusion proof that the current validator set signs for the commitment for the next validator set. This computation happens inside the updateSyncCommittee function.

> Unfortunately, the commitment the validators sign is an SSZ commitment (simple serialization, Eth PoS serialization format) that is quite SNARK unfriendly, as it uses the SHA-256 hash function. It takes ~70 million constraints in a Groth16 circuit to compute the serialization of 512 validator BLS public keys to its corresponding SSZ commitment. Because we don’t want to do this for every single header verification proof (which happens every 6 minutes, i.e. once per epoch), we use an additional SNARK (the commitmentMappingProof argument) to provably map an SSZ commitment to a SNARK-friendly Poseidon commitment, that is stored in the mapping sszToPoseidon. For each BLS signature verification, we pass in the poseidon commitment of the sync committee validators as public input to ensure that the BLS signature we are verifying is from the correct public keys. Overall this approach (using 2 SNARKs) saves us 70M constraints on the BLS signature verification SNARK, which we must run for every update we wish to submit to the light client. The commitment mapping SNARK must only be run every sync committee period (roughly once every 27 hours).
>
> Toolchain
> We use the Circom programming language and the Groth16 proving system to generate our zkSNARKs. While a newer proof system (like PLONK arithmetization + KZG or FRI) would improve proving time, we believe Circom is the most production-ready zkSNARK stack today. In particular, Tornado Cash’s circuits are built on top of Circom and have been used for several years. Additionally, the on-chain verification cost of a Groth16 zkSNARK is the cheapest of all proving systems available today.

3. [eth-proof-of-consensus](https://github.com/succinctlabs/eth-proof-of-consensus): github repository
4. [GIP-57](https://forum.gnosis.io/t/gip-57-should-gnosis-dao-support-research-of-a-zksnark-enabled-light-client-and-bridge/5421): $600,000 Grant from Gnosis to Succint to support
5. [Succint Tweet](https://twitter.com/succinctlabs/status/1572299292177481729) : Succint tweet giving an overview of the bridge
6. [Succint Blog Sep 20, 2022](https://blog.succinct.xyz/post/2022/09/20/proof-of-consensus): Towards the endgame of blockchain interoperability with proof of consensus
7. [GIP-57](https://forum.gnosis.io/t/gip-57-should-gnosis-dao-support-research-of-a-zksnark-enabled-light-client-and-bridge/5421): $600,000 Grant from Gnosis to Succint to support research of a zkSNARK-enabled light client and bridge. 
8. [Succint Video](https://youtu.be/cMSayTJA1B4): ZK8: Succinct Verification of Consensus with zkSNARKs - Uma Roy & John Guibas - Succinct Labs

## Trusted Setup

### Best Practices for Setup
1. [Best Practices for Large Circuits](https://hackmd.io/V-7Aal05Tiy-ozmzTGBYPA): compiling and generating Groth16 proofs for large ZK circuits using the circom / snarkjs toolstack.
> For such large circuits, you need a machine with an Intel processor, lots of RAM and a large hard drive with swap enabled. For example, the zkPairing project used an AWS r5.8xlarge instance with 32-core 3.1GHz, 256G RAM machine with 1T hard drive and 400G swap.
>
> Compilation: for circuits with >20M constraints, one should not compile to WebAssembly because witness generation will exceed the memory cap of WebAssembly. For this reason, one must compile with the C++ flag and remove the wasm flag.

2. [Hermez Zero-Knowledge Proofs](https://blog.hermez.io/hermez-zero-knowledge-proofs/): Overview of the Hermez Trusted Setupi

[Machine](https://aws.amazon.com/ec2/pricing/on-demand/): AWS r5.8xlarge instance with 32-core 3.1GHz, 256G RAM machine with 1T hard drive and 400G swap. $2.016 per hour

### Trusted Ceremony (Powers of Tau)

1. [Perpetual Powers of Tau](https://github.com/weijiekoh/perpetualpowersoftau): The goal is to securely generate zk-SNARK 
2. [snarkjs Prepare phase 2](https://github.com/iden3/snarkjs/blob/master/README.md#7-prepare-phase-2): Provide instructions on prepare phase 2 and links to the Powers of Tau files.
3. [Powers of Tau files on Dropbox](https://www.dropbox.com/sh/mn47gnepqu88mzl/AACaJkBU7mmCq8uU8ml0-0fma?dl=0): 


[Download powersOfTau28_hez_final_27.ptau](https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_27.ptau): 144 GB file containing the encrypted evaluation of the Lagrange polynomials at tau for tau, alpha*tau and beta*tau. It takes the beacon ptau file we generated in the previous step, and outputs a final ptau file which will be used to generate the circuit proving and verification keys.

### Example Build

1. [build_aggregate_bls_verify.sh](https://github.com/succinctlabs/eth-proof-of-consensus/blob/main/circuits/circuits/aggregate_bls_verify.circom)

```
#!/bin/bash
PHASE1=/home/ubuntu/powersOfTau28_hez_final_27.ptau
BUILD_DIR=../build
CIRCUIT_NAME=test_aggregate_bls_verify_512
TEST_DIR=../test
OUTPUT_DIR="$BUILD_DIR"/"$CIRCUIT_NAME"_cpp

run() {
    if [ ! -d "$BUILD_DIR" ]; then
        echo "No build directory found. Creating build directory..."
        mkdir -p "$BUILD_DIR"
    fi

    # echo "****COMPILING CIRCUIT****"
    # start=`date +%s`
    # circom "$TEST_DIR"/circuits/"$CIRCUIT_NAME".circom --O1 --r1cs --sym --c --output "$BUILD_DIR"
    # end=`date +%s`
    # echo "DONE ($((end-start))s)"

    # echo "****Running make to make witness generation binary****"
    # start=`date +%s`
    # make -C "$OUTPUT_DIR"
    # end=`date +%s`
    # echo "DONE ($((end-start))s)"

    echo "****Executing witness generation****"
    start=`date +%s`
    ./"$OUTPUT_DIR"/"$CIRCUIT_NAME" "$TEST_DIR"/input_aggregate_bls_verify_512.json witness.wtns
    end=`date +%s`
    echo "DONE ($((end-start))s)"

    echo "****Converting witness to json****"
    start=`date +%s`
    npx snarkjs wej "$OUTPUT_DIR"/witness.wtns "$OUTPUT_DIR"/witness.json
    end=`date +%s`
    echo "DONE ($((end-start))s)"

    echo "****GENERATING ZKEY 0****"
    start=`date +%s`
    npx --trace-gc --trace-gc-ignore-scavenger --max-old-space-size=2048000 --initial-old-space-size=2048000 --no-global-gc-scheduling --no-incremental-marking --max-semi-space-size=1024 --initial-heap-size=2048000 --expose-gc snarkjs zkey new "$BUILD_DIR"/"$CIRCUIT_NAME".r1cs "$PHASE1" "$OUTPUT_DIR"/"$CIRCUIT_NAME"_p1.zkey
    end=`date +%s`
    echo "DONE ($((end-start))s)"

    echo "****CONTRIBUTE TO PHASE 2 CEREMONY****"
    start=`date +%s`
    npx snarkjs zkey contribute "$OUTPUT_DIR"/"$CIRCUIT_NAME"_p1.zkey "$OUTPUT_DIR"/"$CIRCUIT_NAME"_p2.zkey -n="First phase2 contribution" -e="some random text for entropy"
    end=`date +%s`
    echo "DONE ($((end-start))s)"

    echo "****VERIFYING FINAL ZKEY****"
    start=`date +%s`
    npx --trace-gc --trace-gc-ignore-scavenger --max-old-space-size=2048000 --initial-old-space-size=2048000 --no-global-gc-scheduling --no-incremental-marking --max-semi-space-size=1024 --initial-heap-size=2048000 --expose-gc npx snarkjs zkey verify "$BUILD_DIR"/"$CIRCUIT_NAME".r1cs "$PHASE1" "$OUTPUT_DIR"/"$CIRCUIT_NAME"_p2.zkey
    end=`date +%s`
    echo "DONE ($((end-start))s)"

    echo "****EXPORTING VKEY****"
    start=`date +%s`
    npx snarkjs zkey export verificationkey "$OUTPUT_DIR"/"$CIRCUIT_NAME"_p2.zkey "$OUTPUT_DIR"/"$CIRCUIT_NAME"_vkey.json
    end=`date +%s`
    echo "DONE ($((end-start))s)"

    echo "****GENERATING PROOF FOR SAMPLE INPUT****"
    start=`date +%s`
    ~/rapidsnark/build/prover "$OUTPUT_DIR"/"$CIRCUIT_NAME"_p2.zkey "$OUTPUT_DIR"/witness.wtns "$OUTPUT_DIR"/"$CIRCUIT_NAME"_proof.json "$OUTPUT_DIR"/"$CIRCUIT_NAME"_public.json
    end=`date +%s`
    echo "DONE ($((end-start))s)"

    echo "****VERIFYING PROOF FOR SAMPLE INPUT****"
    start=`date +%s`
    npx snarkjs groth16 verify "$OUTPUT_DIR"/"$CIRCUIT_NAME"_vkey.json "$OUTPUT_DIR"/"$CIRCUIT_NAME"_public.json "$OUTPUT_DIR"/"$CIRCUIT_NAME"_proof.json
    end=`date +%s`
    echo "DONE ($((end-start))s)"

    echo "****EXPORTING SOLIDITY SMART CONTRACT****"
    start=`date +%s`
    npx snarkjs zkey export solidityverifier "$OUTPUT_DIR"/"$CIRCUIT_NAME"_p2.zkey verifier.sol
    end=`date +%s`
    echo "DONE ($((end-start))s)"
}

mkdir -p logs
run 2>&1 | tee logs/"$CIRCUIT_NAME"_$(date '+%Y-%m-%d-%H-%M').log
```


## circuits

1. [Circom Documentation](https://docs.circom.io/getting-started/installation/): Circom is a novel domain-specific language for defining arithmetic circuits that can be used to generate zero-knowledge proofs.
2. [Circom github](https://github.com/iden3/circom)
3. [circomlib github](https://github.com/iden3/circomlib) contains a library of circuit templates.
4. [zkPairing Docs](https://0xparc.org/blog/zk-pairing-1): zkSNARKs for Elliptic Curve Pairings (Part 1)
5.  [circom-paring github](https://github.com/yi-sun/circom-pairing): proof-of-concept implementations of elliptic curve pairings (in particular, the optimal Ate pairing and Tate pairing) for the BLS12-381 curve in circom.
6. [Batch ECDSA Verification (github)](https://github.com/puma314/batch-ecdsa): Implementation of batch ECDSA verification in circom.
7. [circom-ecdsa (github)](https://github.com/0xPARC/circom-ecdsa):  proof-of-concept implementations of ECDSA operations in circom.
8. [snarkjs](https://www.npmjs.com/package/snarkjs): This is a JavaScript and Pure Web Assembly implementation of zkSNARK and PLONK schemes. It uses the Groth16 Protocol (3 point only and 3 pairings) and PLONK.
9. [snarkjs Prepare phase 2](https://github.com/iden3/snarkjs/blob/master/README.md#7-prepare-phase-2): Provide instructions on prepare phase 2 and links to the Powers of Tau files.
10. [Perpetual Powers of Tau](https://github.com/weijiekoh/perpetualpowersoftau): The goal is to securely generate zk-SNARK parameters for circuits of up to 2 ^ 28 (260+ million) constraints.
11. [Powers of Tau files on Dropbox](https://www.dropbox.com/sh/mn47gnepqu88mzl/AACaJkBU7mmCq8uU8ml0-0fma?dl=0): 
12. [eth-proof-of-consensus: circuits aggregate_bls_verify.circom](https://github.com/succinctlabs/eth-proof-of-consensus/blob/main/circuits/circuits/aggregate_bls_verify.circom): example circuit with the following includes

```
include "../circom-pairing/circuits/bls_signature.circom";
include "../circom-pairing/circuits/curve.circom";
include "../circom-pairing/circuits/bls12_381_func.circom";
include "./sha256_bytes.circom";
```

## Contracts

Built using [foundry](https://book.getfoundry.sh/)([github](https://github.com/foundry-rs/foundry)) and [forge](https://book.getfoundry.sh/forge/).  Verifiers ([Light Client Contracts](#light-client-contracts)) can be [generated](https://docs.circom.io/getting-started/proving-circuits/#verifying-from-a-smart-contract) from [circuits](#circuits) using [snarkjs](https://www.npmjs.com/package/snarkjs)

### Library Contracts

1. [eth-proof-of-consensus/contracts/lib/](https://github.com/succinctlabs/eth-proof-of-consensus/tree/main/contracts/lib)
   1. [RLP decoder/reader](https://github.com/hamdiallam/Solidity-RLP): The reader contract provides an interface to first take RLP encoded bytes and convert them into an internal data structure, RLPItem through the function, toRlpItem(bytes).
   2. [curve-merkle-oracle](https://github.com/lidofinance/curve-merkle-oracle): Trustless price oracle for ETH/stETH Curve pool. 

> Mechanics
The oracle works by generating and verifying Merkle Patricia proofs of the following Ethereum state:
> 
> Curve stETH/ETH pool contract account and the following slots from its storage trie:
> 
> admin_balances[0]
> admin_balances[1]
> stETH contract account and the following slots from its storage trie:
> 
> shares[0xDC24316b9AE028F1497c275EB9192a3Ea0f67022]
> keccak256("lido.StETH.totalShares")
> keccak256("lido.Lido.beaconBalance")
> keccak256("lido.Lido.bufferedEther")
> keccak256("lido.Lido.depositedValidators")
> keccak256("lido.Lido.beaconValidators")

### Light Client Contracts

1. [eth-proof-of-consensus: lightclient](https://github.com/succinctlabs/eth-proof-of-consensus/tree/main/contracts/src/lightclient)
   1. [BeaconLightClient.sol](https://github.com/succinctlabs/eth-proof-of-consensus/blob/main/contracts/src/lightclient/BeaconLightClient.sol)
   2. [PoseidonCommitmentVerifier.sol](https://github.com/succinctlabs/eth-proof-of-consensus/blob/main/contracts/src/lightclient/PoseidonCommitmentVerifier.sol)
   3. [BLSAggregatedSignatureVerifier.sol](https://github.com/succinctlabs/eth-proof-of-consensus/blob/main/contracts/src/lightclient/BLSAggregatedSignatureVerifier.sol)

### Bridge Contracts

1. [eth-proof-of-consensus: amb](https://github.com/succinctlabs/eth-proof-of-consensus/tree/main/contracts/src/amb): Arbitrary Message Bridge
   1. [Ethereum Magicians: A standard interface for arbitrary message bridges between chains/layers](https://ethereum-magicians.org/t/a-standard-interface-for-arbitrary-message-bridges-between-chains-layers/6163)
   2. [Token BridgeL ETH-xDai Arbitrary Message Bridge](https://docs.tokenbridge.net/eth-xdai-amb-bridge/about-the-eth-xdai-amb): An Arbitrary Message Bridge (AMB) between the Ethereum Mainnet and the xDai chain
2. [eth-proof-of-consensus: bridge](https://github.com/succinctlabs/eth-proof-of-consensus/tree/main/contracts/src/bridge)



### Additional Contracts

1. [tokenbridge-contracts](https://github.com/succinctlabs/tokenbridge-contracts): core functionality for the POA bridge. They implement the logic to relay assests between two EVM-based blockchain networks. The contracts collect bridge validator's signatures to approve and facilitate relay operations. (forked from [omni](https://github.com/omni/tokenbridge-contracts))

## Relayer



*Note: no public repository for relay functionality was found in [succintlabs github](https://github.com/succinctlabs).*

**TODO**: This section should give an overview of
* Communication Protocol
* Message Formatting
* Relayer CodeBase
* Relayers Roles : Creating Proofs relaying blocks etc.
* Economic incentives.

Additional Information can be found in [DeepDiveOnNearBridge: Relayers](./DeepDiveOnNearBridge.md#relayers)


1. [BeaconLightClient on Gnosis Chain](https://blockscout.com/xdai/mainnet/address/0xa3ae36abaD813241b75b3Bb0e9E7a37aeFD70807): Transactions every 50 blocks on Gnosis i.e. approximately every 3 minutes
2. [Succint Blog Oct 29, 2022](https://blog.succinct.xyz/post/2022/10/29/gnosis-bridge/): Proof of Consensus Bridging between Ethereum and Gnosis Chain
> On Gnosis Chain, after the Ethereum block in which the deposit transaction was included is finalized (generally 2 epochs, ~12 minutes) and the light client has been updated with a block of height greater than or equal to this block, our relayer automatically submits an executeMessage transaction to the Gnosis AMB. 