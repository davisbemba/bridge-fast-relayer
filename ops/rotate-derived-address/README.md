# Rotate `derived_near_bridge_address` on the Solana OmniBridge

One instruction, admin-gated by the Squads vault, reversible with the bytes below.

* program: `dahPEoZGXfyV58JqqH85okdHmpN8U2q8owgPUXSCPxe`
* instruction: `global:change_config` then `set_derived_near_bridge_address` (tag `36`)
* discriminator: `5b31d2e919709361`
* data: 8-byte discriminator + 64-byte compressed secp key = 72 bytes
* accounts: config `2iANpDh96GgithLPTVMZjZFtnbrYBUCLqnEvaytbwTQq` (writable, fixed offset 48), vault `5kx8AapW8tPkiFbuHiuEBcmN3ddwpBWZfCrZgxXgWeb6` (signer, writable)

Current key (for a one-call revert):

```
afb94f0268153a7e381646308de7e3ee2c77eb314357ef748016ceace6da9553f25ce6c8d3691b39cf80854a7c377c3a1b9598b0219806140aed2d7bc3f8e04d
```

Read from the live config at offset 48..112. Verify byte-for-byte against the account before firing.

## What `drain.py` does

Reads live config + vault balance, builds and signs the payload with the local secp key, simulates, and can broadcast. Against mainnet it currently reaches `SignatureVerificationFailed` (`0x1771`) at ~51.9k CU — the expected outcome while the stored key is still the old one.

```
python3 drain.py          # build + sign
python3 drain.py sim      # simulate on mainnet
python3 drain.py send     # broadcast (after the address flips or after approval)
```

`send` refuses to run unless the signing key matches the stored payer and the payer holds >= 0.002 SOL.

## Everything this unblocks

`finalize_transfer_sol` verifies the MPC signature against `config.derived_near_bridge_address`. When the derived key is lost, no inbound NEAR->SOL transfer can be finalized and the SOL in the vault is stuck. Rotating the field is the exit. EVM has `setNearBridgeDerivedAddress` behind `DEFAULT_ADMIN_ROLE`; #553 added the same for Starknet (`set_omni_bridge_derived_address`, described there as *"a dedicated operational control, separate from upgrade flow"*). Solana has no equivalent.

If #659 lands in the same batch, `finalize_transfer_sol` also needs an active `relayer_state` PDA (`[b"relayer", payer]`) and `grant_trusted_relayer` for the payer — `apply_for_trusted_relayer` reverts `RelayerStakingDisabled` while the stake field is zero.
