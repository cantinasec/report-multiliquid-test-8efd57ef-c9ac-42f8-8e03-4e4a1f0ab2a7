## Medium Risk
### `U64DynamicAddress` Pricing cannot distinguish between assets
<!--
Number: 20
Cantina code repository status: fixed
Hyperlink: [Issue 20](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/20)
Labels: [None]
Fixed on: [commit [1d541265](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/1d541265ca6a013eaa0093c0acb8d3e774bfb22e)]
-->


**Severity:** Medium Risk

**Context:** _(No context files were provided by the reviewer)_


**Description:** When a `U64DynamicAddress` type is processed in the `get_nav` function, it reads the nav from the first account in the unchecked `remaining_accounts` array. There are a few issues with how this is done.

1. Only the first of the user-provided `remaining_accounts` is ever read from and used for all dynamic prices of the asset.
2. The only requirement for this account is that it be owned by a specified `nav_program_id` account. Should this `nav_program_id`  own multiple accounts then any one of them could be passed and read as a price for any other assets in this process.


**Recommendation:**

1. Either iterate through the remaining_accounts or don't allow multiple dynamic pricing accounts for a single asset.
2. Determine if supporting dynamic accounts is required and whether one could not ask the data account owner for the price of the asset rather than directly querying the data account. Another possibility could include having the pricing data account owned by the swap program itself and tracking which account would go to which asset.



<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [1d541265](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/1d541265ca6a013eaa0093c0acb8d3e774bfb22e).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Cantina Managed:** @red-swan i did find the fix accurate, but would also love you to double check. 

**Cantina Managed:** https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/1d541265ca6a013eaa0093c0acb8d3e774bfb22e

**Cantina Managed:** I'm of the same opinion. Looks good to me.
-->



## Low Risk
### Global config initialization can be frontrun
<!--
Number: 1
Cantina code repository status: acknowledged
Hyperlink: [Issue 1](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/1)
Labels: [None]
Fixed on: [None]
-->


**Severity:** Low Risk

**Context:** [init_global_config.rs#L27](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/init_global_config.rs#L27)


**Description:** There is no access control on the global config initialization. A malicious actor can frontrun the initialization with their own parameters. If this goes undetected, user funds could be at risk.


**Recommendation:** Consider one of the following:

1. Restrict initialization to privileged accounts.
2. Always ensure initialization scripts validate configurations after the transaction submission.


<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Acknowledged.

**Cantina Managed:** Acknowledged.

**_COMMENTS_**:

**Cantina Managed:** Update: team acknowledged this issue via telegram. 

> aknowledge. We will inform MultiLiquid about this risk.
-->



### Asset type modification post-initialization causes denial of service on swaps
<!--
Number: 2
Cantina code repository status: fixed
Hyperlink: [Issue 2](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/2)
Labels: [None]
Fixed on: [commit [765675dd](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/765675dd7f570056e4985405031180212bf16980)]
-->


**Severity:** Low Risk

**Context:** [update_asset_config_account.rs#L51](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/update_asset_config_account.rs#L51)


**Description:** The `update_asset_config_account()` function allows the protocol admin to modify the asset_type field of an AssetConfig account after initialization. This will break the swap implementation, causing swaps to fail.

The `swap()` function enforces strict asset type validation at `swap.rs`:

```rust
// check that the asset types are correct
require!(ctx.accounts.rwa_config.asset_type == AssetType::Rwa, ErrorCode::InvalidAssetType);
require!(ctx.accounts.stable_config.asset_type == AssetType::Stable, ErrorCode::InvalidAssetType);
```

When an admin changes the asset_type of an asset from its original designation (e.g., changing a Stable to an Rwa, or vice versa), all swaps involving that asset will permanently revert with an InvalidAssetType error. This effectively bricks all liquidity pools using that asset. However, LPs are not affected, as such validations do not happen in the withdrawal liquidity-related functions.


**Recommendation:** Make asset_type immutable after initialization by removing it from the `update_asset_config_account()` function:

```rust
pub fn update_asset_config_account(
     ctx: Context<UpdateAssetConfigAccount>,
     nav_data: Vec<NavData>,
     price_difference_bps: u16,
     // Remove asset_type parameter
 ) -> Result<()> {
     // ....
     // Remove: asset_config.asset_type = asset_type;

     Ok(())
 }
```

If asset type updates must be supported for legitimate reasons, add comprehensive validation:

1. Check that no active pairs exist using this asset before allowing type changes.
2. Add a time-lock mechanism requiring a delay before the change takes effect.
3. Emit events to warn LPs of the upcoming change.
4. Consider adding a separate "migration" function with stricter controls.

<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [765675dd](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/765675dd7f570056e4985405031180212bf16980).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Multiliquid:** fixed by commit 765675dd7f570056e4985405031180212bf16980

**Multiliquid:** I have added a field in asset config state which store a counter of how many times this asset is used in a pair. If it is used in at least 1 pair, it is impossible to update the asset type (thus we can prevent breaking the swap instruction)
-->



### No validation for combined fees causes transaction panic when total exceeds 100%
<!--
Number: 8
Cantina code repository status: fixed
Hyperlink: [Issue 8](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/8)
Labels: [None]
Fixed on: [commit [acb2ca06](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/acb2ca06fba8317cc51ec42ad353134d897a9d67)]
-->


**Severity:** Low Risk

**Context:** [math.rs#L90-L97](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/math.rs#L90-L97)


**Description:** The protocol validates individual fee parameters but fails to validate that combined fees (pair fees + protocol fees) don't exceed 100%, causing transaction panics during swaps.

1. Init_pair / update_pair.

```rust
require!(redemption_fee_bps <= 9900, ErrorCode::OutOfRange);  // Up to 99%
require!(discount_rate_bps <= 9900, ErrorCode::OutOfRange);    // Up to 99%
```

2. Init_global_config.

```rust
require!(protocol_fees_bps <= 9900, ErrorCode::OutOfRange);  // Up to 99%
```


As a result, the total fees calculated exceed the amount_in parameter during stable-to-asset swaps, resulting in an underflow.


**Proof of Concept:** Consider the following scenario:

1. Admin sets protocol_fees_bps = 200 (2%).
2. LP sets redemption_fee_bps = 9900 (99%).
3. User attempts swap: 9900 + 200 = 10100 > 10000.
4. Transaction panics with a MathOverflow error instead of meaningful message.


**Recommendation:** Add validations in all fee configuration functions to ensure the total_fee_bps remains less than 10,000. Or consider adding a validation in the `calculate_swap_results()` function.


<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [acb2ca06](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/acb2ca06fba8317cc51ec42ad353134d897a9d67).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Cantina Managed:** DO NOT want to pollute the report with a large js file. So the full PoC isn't added to the finding:

```js
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { SwapProgram } from "../target/types/swap_program";
import { expect } from "chai";
import { Keypair, PublicKey, SystemProgram } from "@solana/web3.js";
import {
  TOKEN_PROGRAM_ID,
  createMint,
  createAccount,
  mintTo,
  getAccount,
} from "@solana/spl-token";

describe("swap-program", () => {
  // Configure the client to use the local cluster
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);

  const program = anchor.workspace.SwapProgram as Program<SwapProgram>;

  const malicious = Keypair.generate();

  // Test state
  let globalConfigPda: PublicKey;
  let stableMint: PublicKey;
  let assetMint: PublicKey;
  const liquidityProvider = Keypair.generate();
  let lpStableTokenAccount: PublicKey;
  let lpAssetTokenAccount: PublicKey;

  before(async () => {
    // Derive the global config PDA
    [globalConfigPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("global_config")],
      program.programId
    );

    // Check if global config exists, if not initialize it
    try {
      await program.account.globalConfig.fetch(globalConfigPda);
      console.log("Global config already initialized");
    } catch (error) {
      // Global config doesn't exist, initialize it
      console.log("Initializing global config...");
      const feeWallet = provider.wallet.publicKey;
      const protocolFeesBps = 9900; // 1% fee

      await program.methods
        .initGlobalConfig(feeWallet, protocolFeesBps)
        .accounts({
          globalConfig: globalConfigPda,
          admin: provider.wallet.publicKey,
          systemProgram: SystemProgram.programId,
        })
        .rpc();

      console.log("Global config initialized successfully");
    }

    // Create mints for the pair
    console.log("Creating stable coin mint...");
    stableMint = await createMint(
      provider.connection,
      provider.wallet.payer,
      provider.wallet.publicKey,
      provider.wallet.publicKey,
      6 // USDC-like decimals
    );
    console.log("Stable mint created:", stableMint.toString());

    console.log("Creating asset token mint...");
    assetMint = await createMint(
      provider.connection,
      provider.wallet.payer,
      provider.wallet.publicKey,
      provider.wallet.publicKey,
      6 // decimals
    );
    console.log("Asset mint created:", assetMint.toString());

    // Initialize asset config for stable coin
    await initAssetConfig(stableMint, { stable: {} });
    console.log("Stable coin asset config initialized");

    // Initialize asset config for asset token
    await initAssetConfig(assetMint, { rwa: {} });
    console.log("Asset token asset config initialized");
  });

  async function initAssetConfig(mint: PublicKey, assetType: any) {
    const [assetConfigPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("asset"), mint.toBuffer()],
      program.programId
    );

    const [feeVaultPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("fee_vault"), mint.toBuffer()],
      program.programId
    );

    const [programAuthorityPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("program_authority")],
      program.programId
    );

    const navData = [
      {
        hardcoded: {
          hardcodedPrice: new anchor.BN(1_000_000_000), // 1.0 with 9 decimals
          priceDecimals: 9,
        },
      },
    ];

    const priceDifferenceBps = 500; // 5% max difference

    await program.methods
      .initAssetConfigAccount(navData, priceDifferenceBps, assetType)
      .accounts({
        assetConfig: assetConfigPda,
        mintAddress: mint,
        admin: provider.wallet.publicKey,
        feeVaultTokenAccount: feeVaultPda,
        programAuthority: programAuthorityPda,
        globalConfig: globalConfigPda,
        tokenProgram: TOKEN_PROGRAM_ID,
        systemProgram: SystemProgram.programId,
      })
      .rpc();
  }

  it("Initialize A Pair", async () => {
    // Derive all required PDAs
    const [pairPda] = PublicKey.findProgramAddressSync(
      [
        Buffer.from("pair"),
        liquidityProvider.publicKey.toBuffer(),
        stableMint.toBuffer(),
        assetMint.toBuffer(),
      ],
      program.programId
    );

    const [stableVaultPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("vault"), stableMint.toBuffer(), liquidityProvider.publicKey.toBuffer()],
      program.programId
    );

    const [assetVaultPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("vault"), assetMint.toBuffer(), liquidityProvider.publicKey.toBuffer()],
      program.programId
    );

    const [programAuthorityPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("program_authority")],
      program.programId
    );

    const [lpVaultStablePda] = PublicKey.findProgramAddressSync(
      [Buffer.from("user_vault_info"), stableMint.toBuffer(), liquidityProvider.publicKey.toBuffer()],
      program.programId
    );

    const [lpVaultAssetPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("user_vault_info"), assetMint.toBuffer(), liquidityProvider.publicKey.toBuffer()],
      program.programId
    );

    const [lpStableConfigPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("lp_stable_config"), stableMint.toBuffer(), liquidityProvider.publicKey.toBuffer()],
      program.programId
    );

    // Set pair parameters
    const redemptionFeeBps = 9900; // 2% fee when swapping stable to asset
    const discountRateBps = 300; // 3% discount when swapping asset to stable

    console.log("Initializing pair...");
    console.log("Liquidity Provider:", liquidityProvider.publicKey.toString());
    console.log("Stable Mint:", stableMint.toString());
    console.log("Asset Mint:", assetMint.toString());

    // Call initPair instruction
    const tx = await program.methods
      .initPair(liquidityProvider.publicKey, redemptionFeeBps, discountRateBps)
      .accounts({
        admin: provider.wallet.publicKey,
        pair: pairPda,
        stableCoinMintAddress: stableMint,
        assetTokenMintAddress: assetMint,
        stableCoinVaultTokenAccount: stableVaultPda,
        assetTokenVaultTokenAccount: assetVaultPda,
        programAuthority: programAuthorityPda,
        lpVaultStablePda: lpVaultStablePda,
        lpVaultAssetPda: lpVaultAssetPda,
        globalConfig: globalConfigPda,
        lpStableConfig: lpStableConfigPda,
        tokenProgramStable: TOKEN_PROGRAM_ID,
        tokenProgramAsset: TOKEN_PROGRAM_ID,
        systemProgram: SystemProgram.programId,
      })
      .rpc();

    console.log("Transaction signature:", tx);

    // Fetch and verify the pair account
    const pair = await program.account.pair.fetch(pairPda);

    expect(pair.redemptionFeeBps).to.equal(redemptionFeeBps);
    expect(pair.discountRateBps).to.equal(discountRateBps);
    expect(pair.liquidityProvider.toString()).to.equal(liquidityProvider.publicKey.toString());
    expect(pair.stableCoinMintAddress.toString()).to.equal(stableMint.toString());
    expect(pair.assetTokenMintAddress.toString()).to.equal(assetMint.toString());
    expect(pair.paused).to.equal(false);

    console.log("Pair initialized successfully!");
    console.log("Redemption Fee BPS:", pair.redemptionFeeBps);
    console.log("Discount Rate BPS:", pair.discountRateBps);
    console.log("Liquidity Provider:", pair.liquidityProvider.toString());
    console.log("Stable Coin Mint:", pair.stableCoinMintAddress.toString());
    console.log("Asset Token Mint:", pair.assetTokenMintAddress.toString());
    console.log("Paused:", pair.paused);

    // Also verify the LP stable config was created
    const lpStableConfig = await program.account.lpStableConfig.fetch(lpStableConfigPda);
    expect(lpStableConfig.stableCoinMintAddress.toString()).to.equal(stableMint.toString());
    expect(lpStableConfig.liquidityProvider.toString()).to.equal(liquidityProvider.publicKey.toString());
    expect(lpStableConfig.paused).to.equal(false);

    console.log("LP Stable Config verified!");

    // Unpause the global config to allow liquidity operations
    console.log("\n=== Unpausing Global Config ===");
    await program.methods
      .updateGlobalConfig(null, false, null) // fee_wallet, paused, protocol_fees_bps
      .accounts({
        globalConfig: globalConfigPda,
        admin: provider.wallet.publicKey,
      })
      .rpc();
    console.log("Global config unpaused");

    // Airdrop SOL to the liquidity provider
    const airdropSig = await provider.connection.requestAirdrop(
      liquidityProvider.publicKey,
      10 * anchor.web3.LAMPORTS_PER_SOL
    );
    await provider.connection.confirmTransaction(airdropSig);
    console.log("Liquidity provider funded with SOL");

    // Create token accounts for the liquidity provider
    lpStableTokenAccount = await createAccount(
      provider.connection,
      provider.wallet.payer,
      stableMint,
      liquidityProvider.publicKey
    );
    console.log("LP Stable Token Account created:", lpStableTokenAccount.toString());

    lpAssetTokenAccount = await createAccount(
      provider.connection,
      provider.wallet.payer,
      assetMint,
      liquidityProvider.publicKey
    );
    console.log("LP Asset Token Account created:", lpAssetTokenAccount.toString());

    // Mint tokens to the liquidity provider
    const stableAmount = 1_000_000 * 1_000_000; // 1,000,000 USDC (6 decimals)
    const assetAmount = 10_000 * 1_000_000; // 10,000 asset tokens (6 decimals)

    await mintTo(
      provider.connection,
      provider.wallet.payer,
      stableMint,
      lpStableTokenAccount,
      provider.wallet.publicKey,
      stableAmount
    );
    console.log("Minted", stableAmount, "stable tokens to LP");

    await mintTo(
      provider.connection,
      provider.wallet.payer,
      assetMint,
      lpAssetTokenAccount,
      provider.wallet.publicKey,
      assetAmount
    );
    console.log("Minted", assetAmount, "asset tokens to LP");

    // Add liquidity - stable coins
    console.log("\n=== Adding Stable Liquidity ===");
    const stableLiquidityAmount = 500_000 * 1_000_000; // 500,000 USDC

    await program.methods
      .addLiquidity(new anchor.BN(stableLiquidityAmount))
      .accounts({
        liquidityProvider: liquidityProvider.publicKey,
        mintAddress: stableMint,
        lpTokenAccount: lpStableTokenAccount,
        vaultTokenAccount: stableVaultPda,
        programAuthority: programAuthorityPda,
        globalConfig: globalConfigPda,
        tokenProgram: TOKEN_PROGRAM_ID,
      })
      .signers([liquidityProvider])
      .rpc();

    console.log("Added", stableLiquidityAmount, "stable tokens to vault");

    // Verify stable vault balance
    const stableVaultInfo = await getAccount(provider.connection, stableVaultPda);
    console.log("Stable vault balance:", stableVaultInfo.amount.toString());
    expect(stableVaultInfo.amount.toString()).to.equal(stableLiquidityAmount.toString());

    // Add liquidity - asset tokens
    console.log("\n=== Adding Asset Liquidity ===");
    const assetLiquidityAmount = 5_000 * 1_000_000; // 5,000 asset tokens

    await program.methods
      .addLiquidity(new anchor.BN(assetLiquidityAmount))
      .accounts({
        liquidityProvider: liquidityProvider.publicKey,
        mintAddress: assetMint,
        lpTokenAccount: lpAssetTokenAccount,
        vaultTokenAccount: assetVaultPda,
        programAuthority: programAuthorityPda,
        globalConfig: globalConfigPda,
        tokenProgram: TOKEN_PROGRAM_ID,
      })
      .signers([liquidityProvider])
      .rpc();

    console.log("Added", assetLiquidityAmount, "asset tokens to vault");

    // Verify asset vault balance
    const assetVaultInfo = await getAccount(provider.connection, assetVaultPda);
    console.log("Asset vault balance:", assetVaultInfo.amount.toString());
    expect(assetVaultInfo.amount.toString()).to.equal(assetLiquidityAmount.toString());

    console.log("\n=== Liquidity Added Successfully! ===\n");

    // Create random user for swaps
    console.log("=== Setting up Random User for Swaps ===");
    const randomUser = Keypair.generate();

    // Airdrop SOL to the random user
    const userAirdropSig = await provider.connection.requestAirdrop(
      randomUser.publicKey,
      5 * anchor.web3.LAMPORTS_PER_SOL
    );
    await provider.connection.confirmTransaction(userAirdropSig);
    console.log("Random user funded with SOL");

    // Create token accounts for the random user
    const userStableTokenAccount = await createAccount(
      provider.connection,
      provider.wallet.payer,
      stableMint,
      randomUser.publicKey
    );
    console.log("User Stable Token Account created:", userStableTokenAccount.toString());

    const userAssetTokenAccount = await createAccount(
      provider.connection,
      provider.wallet.payer,
      assetMint,
      randomUser.publicKey
    );
    console.log("User Asset Token Account created:", userAssetTokenAccount.toString());

    // Mint some stable tokens to the user for swapping
    const userStableAmount = 10_000 * 1_000_000; // 10,000 USDC
    await mintTo(
      provider.connection,
      provider.wallet.payer,
      stableMint,
      userStableTokenAccount,
      provider.wallet.publicKey,
      userStableAmount
    );
    console.log("Minted", userStableAmount, "stable tokens to random user");

    // Also mint some asset tokens to the user for swapping in the other direction
    const userAssetAmount = 100 * 1_000_000; // 100 asset tokens
    await mintTo(
      provider.connection,
      provider.wallet.payer,
      assetMint,
      userAssetTokenAccount,
      provider.wallet.publicKey,
      userAssetAmount
    );
    console.log("Minted", userAssetAmount, "asset tokens to random user");

    // SWAP 1: Stable to Asset (ExactIn)
    console.log("\n=== Swap 1: Stable to Asset (ExactIn) ===");
    const swapAmount1 = 1_000 * 1_000_000; // 1,000 USDC

    // Derive additional PDAs needed for swap
    const [assetConfigPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("asset"), assetMint.toBuffer()],
      program.programId
    );

    const [stableConfigPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("asset"), stableMint.toBuffer()],
      program.programId
    );

    const [feeVaultPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("fee_vault"), stableMint.toBuffer()],
      program.programId
    );

    // Get balances before swap
    const userStableBeforeSwap1 = await getAccount(provider.connection, userStableTokenAccount);
    const userAssetBeforeSwap1 = await getAccount(provider.connection, userAssetTokenAccount);
    console.log("User stable balance before swap:", userStableBeforeSwap1.amount.toString());
    console.log("User asset balance before swap:", userAssetBeforeSwap1.amount.toString());

    await program.methods
      .swap(
        new anchor.BN(swapAmount1),
        { stableToAsset: {} }, // swapDirection
        { exactIn: {} } // swapType
      )
      .accounts({
        user: randomUser.publicKey,
        pair: pairPda,
        programAuthority: programAuthorityPda,
        liquidityProvider: liquidityProvider.publicKey,
        stableCoinMintAddress: stableMint,
        assetTokenMintAddress: assetMint,
        assetTokenUserTokenAccount: userAssetTokenAccount,
        stableCoinUserTokenAccount: userStableTokenAccount,
        stableCoinVaultTokenAccount: stableVaultPda,
        assetTokenVaultTokenAccount: assetVaultPda,
        globalConfig: globalConfigPda,
        feeTokenAccount: feeVaultPda,
        rwaConfig: assetConfigPda,
        stableConfig: stableConfigPda,
        lpStableConfig: lpStableConfigPda,
        tokenProgramStable: TOKEN_PROGRAM_ID,
        tokenProgramAsset: TOKEN_PROGRAM_ID,
      })
      .signers([randomUser])
      .rpc();

    // Get balances after swap
    const userStableAfterSwap1 = await getAccount(provider.connection, userStableTokenAccount);
    const userAssetAfterSwap1 = await getAccount(provider.connection, userAssetTokenAccount);
    console.log("User stable balance after swap:", userStableAfterSwap1.amount.toString());
    console.log("User asset balance after swap:", userAssetAfterSwap1.amount.toString());
    console.log("Stable tokens spent:", (Number(userStableBeforeSwap1.amount) - Number(userStableAfterSwap1.amount)).toString());
    console.log("Asset tokens received:", (Number(userAssetAfterSwap1.amount) - Number(userAssetBeforeSwap1.amount)).toString());
    });
});
```

**Multiliquid:** handled by acb2ca06fba8317cc51ec42ad353134d897a9d67

i've added the check only in stable > asset direction, because for asset to stable, fees are applied successfully thus it is impossible to be >100%.

I disagree with the level of this finding:
1) the probability to have protocol fees + LP fees > 100% is really unlikely. It doesn't really make sense for a LP or multiliquid to set fees at this kind of level, because the protocol just wouldn't be used with such high fees. (so likelihood is low)
2) the impact is a panic not properly handled, but funds are safe, there is not really a security issue here IMO. So i would classify impact as low as well.


**Cantina Managed:** I would still think it'll be better to validate total fee < 10,000 in the asset to stable direction as well before applying them. 

**Multiliquid:** why?

**Cantina Managed:** Just being defensive here. Cuz anyway if the total fee is > 10,000 it shouldn't proceed and revert in both branches. @fredericbry 
-->



### Missing `protocol_fee_bps` upper bounds validation in update global config
<!--
Number: 9
Cantina code repository status: fixed
Hyperlink: [Issue 9](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/9)
Labels: [None]
Fixed on: [commit [758e9db7](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/758e9db752783981fbaa539e79c8944fe4664ba7)]
-->


**Severity:** Low Risk

**Context:** [update_global_config.rs#L43-L45](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/update_global_config.rs#L43-L45)


**Description:** The `update_global_config()` function allows updating protocol_fees_bps without any validation, unlike `init_global_config()`, which enforces a maximum of 9900 BPS.

As a result, the function allows setting protocol_fees_bps to any u16 value (up to 65535), breaking protocol_fee_bps assumptions throughout the protocol.


**Recommendation:** Consider adding the same validation used in `init_global_config()`:

```diff
   if let Some(protocol_fees_bps) = protocol_fees_bps {
+     require!(protocol_fees_bps <= 9900, ErrorCode::OutOfRange);
      new_protocol_fees_bps = protocol_fees_bps;
  }
```



<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [758e9db7](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/758e9db752783981fbaa539e79c8944fe4664ba7).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Multiliquid:** fixed by 758e9db752783981fbaa539e79c8944fe4664ba7
-->



### Vault token accounts not closed in `close_pair()` instruction
<!--
Number: 19
Cantina code repository status: fixed
Hyperlink: [Issue 19](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/19)
Labels: [None]
Fixed on: [commit [c350258e](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/c350258e83132139a892ae100eb1e6e66d680550)]
-->


**Severity:** Low Risk

**Context:** [close_pair.rs#L39-L59](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/close_pair.rs#L39-L59)


**Description:** The close_pair instruction fails to close the SPL token vault accounts (stable_coin_vault_token_account and asset_token_vault_token_account) when the reference counter (used) reaches zero.

While the instruction properly:
1. Transfers remaining tokens back to the LP.
2. Closes the UserVaultInfo PDA accounts.

It does not close the actual SPL token vault accounts themselves, resulting in orphaned token accounts that permanently lock rent on-chain.


**Recommendation:** Add SPL token account closure to the `close_pair()` instruction `when used == 0`:

```rust
close_account(
  CpiContext::new_with_signer(
     ctx.accounts.token_program_stable.to_account_info(),
        CloseAccount {
          account: ctx.accounts.stable_coin_vault_token_account.to_account_info(),
          destination: ctx.accounts.admin.to_account_info(),
          authority: ctx.accounts.program_authority.to_account_info(),
        },
       &[signer_seeds],
   )
)?;
```


<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [c350258e](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/c350258e83132139a892ae100eb1e6e66d680550).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Multiliquid:** fixed by commit c350258e83132139a892ae100eb1e6e66d680550
-->



## Gas Optimization
### Unnecessary CPI call when protocol fees are zero
<!--
Number: 6
Cantina code repository status: acknowledged
Hyperlink: [Issue 6](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/6)
Labels: [None]
Fixed on: [None]
-->


**Severity:** Gas Optimization

**Context:** [swap.rs#L264-L273](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/swap.rs#L264-L273), [swap.rs#L285-L294](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/swap.rs#L285-L294)


**Description:** The `swap()` function in swap.rs makes CPI calls to transfer protocol fees even when the fee amount is zero, unnecessarily wasting compute units.


**Recommendation:** Consider adding a conditional check to skip the transfer CPI when protocol fees are zero (only if protocol_fees can be zero). If it will never be set to zero, then adding an additional check is redundant and this finding could be safely "acknowledged".


<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Acknowledged.

**Cantina Managed:** Acknowledged.

**_COMMENTS_**:

**Multiliquid:** around line 242, you can find:
if ctx.accounts.global_config.protocol_fees_bps > 0 {

  require!(protocol_fees > 0, ErrorCode::ProtocolFeesMustBePositive);
}

so protocol fees are always > 0

**Cantina Managed:** I mean, **protocol_fees_bps** could be zero. So if its zero the **protocol_fee** value is zero and we do a CPI on zero value.

**Multiliquid:** no protocol fees can't be 0, there is a check that fees > 0 before the CPI

**Cantina Managed:** ```rust
if ctx.accounts.global_config.protocol_fees_bps > 0 {
   require!(protocol_fees > 0, ErrorCode::ProtocolFeesMustBePositive);
}
```    

if protocol_fees_bps == 0 then that check is not done. 
-->



## Informational
### Unused macros and error codes
<!--
Number: 3
Cantina code repository status: fixed
Hyperlink: [Issue 3](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/3)
Labels: [None]
Fixed on: [commit [a418ee44](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/a418ee44c37f94f78ec62c10790cbb30d493aa7d)]
-->


**Severity:** Informational

**Context:** [error.rs#L4](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/error.rs#L4), [macros.rs#L1](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/macros.rs#L1)


**Description:** The codebase contains several instances of dead code that is defined but never used throughout the program:

1. Unused Macro.
The assert_range macro is defined and exported with #[macro_export] but has zero usages across the entire codebase.

2. Unused Error Code Variants.
Seven error code enum variants are defined, but never referenced in any instruction or validation logic:
- InvalidNewAdmin.
- NotAdmin.
- InvalidMintAddress.
- MustProvidePriceDecimals.
- MustProvideNavProgramIdOrNavAccountDiscriminator.
- MustProvideNavPriceOffset.
- LpFeeMustBePositive.


**Recommendation:**
1. Remove the unused assert_range macro from `macro.rs`.
2. Remove all seven unused error code variants from `error.rs` to maintain a clean error enumeration that only includes actively used error cases.


<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [a418ee44](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/a418ee44c37f94f78ec62c10790cbb30d493aa7d).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Multiliquid:** fixed by commit a418ee44c37f94f78ec62c10790cbb30d493aa7d
(macro had already been removed after a comment)
-->



### Missing token account type constraints in `spl_helpers::transfer()`
<!--
Number: 4
Cantina code repository status: fixed
Hyperlink: [Issue 4](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/4)
Labels: [None]
Fixed on: [commit [b34c0a7d](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/b34c0a7d735ad7fd502459d2c809d17c50af445c)]
-->


**Severity:** Informational

**Context:** [spl_helpers.rs#L6-L7](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/spl_helpers.rs#L6-L7)


**Description:** The `spl_helpers::transfer()` function accepts generic AccountInfo parameters for the from and to accounts instead of strongly-typed `InterfaceAccount<'info, TokenAccount>` parameters.

Current state:

- All nine usage instances (swap.rs, claim_fees.rs, close_pair.rs, add_liquidity.rs, remove_liquidity.rs) properly validate token accounts at the instruction level with `InterfaceAccount<'info, TokenAccount>` and appropriate constraints.
- Accounts are downgraded to AccountInfo when passed to the helper via `.to_account_info()`.
- The helper relies entirely on the SPL Token Program validation during CPI.

While the current implementation is secure, it weakens Rust's type-safety guarantees. Future code modifications or new call sites could accidentally pass incorrect account types, with errors only caught at runtime during CPI execution rather than at compile-time or during Anchor validation.


**Recommendation:**
1. Change the helper function signature to enforce strong typing and remove downgrading to type AccountInfo while passing it to the helper in all usage instances.

```rust
from: &InterfaceAccount<'info, TokenAccount>,
to: &InterfaceAccount<'info, TokenAccount>
```

2. Alternatively, add relevant documentation in the `spl_helpers::transfer` function to ensure this behavior is properly documented for future developers.

<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [b34c0a7d](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/b34c0a7d735ad7fd502459d2c809d17c50af445c).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Multiliquid:** fixed by b34c0a7d735ad7fd502459d2c809d17c50af445c
-->



### Unused version field in `AssetConfig` struct
<!--
Number: 5
Cantina code repository status: fixed
Hyperlink: [Issue 5](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/5)
Labels: [None]
Fixed on: [commit [70417c6a](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/70417c6abf107e9081c7f7df63287fc090a17ab1)]
-->


**Severity:** Informational

**Context:** [asset_config.rs#L70-L71](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/state/asset_config.rs#L70-L71)


**Description:** The `AssetConfig` struct contains a version field that is defined but never used anywhere in the codebase. The version field is documented as "version of the NavData struct".


**Recommendation:** Consider removing the unused field (or) implement version tracking if required and start from version 1 instead of 0.


<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [70417c6a](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/70417c6abf107e9081c7f7df63287fc090a17ab1).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Multiliquid:** fixed by 70417c6abf107e9081c7f7df63287fc090a17ab1
-->



### Error code `MathOverflow` used instead of `MathUnderflow` for `checked_sub()` operations
<!--
Number: 7
Cantina code repository status: fixed
Hyperlink: [Issue 7](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/7)
Labels: [None]
Fixed on: [commit [5a99d740](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/5a99d740eed89b28b6a81d6942f8495232a04a32)]
-->


**Severity:** Informational

**Context:** [math.rs#L106-L112](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/math.rs#L106-L112), [math.rs#L180-L182](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/math.rs#L180-L182), [math.rs#L193-L196](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/math.rs#L193-L196)


**Description:** The math.rs module inconsistently uses error codes for arithmetic operations. Multiple instances of `checked_sub()` (subtraction) incorrectly return MathOverflow instead of MathUnderflow, making debugging significantly harder.


**Recommendation:** Update all `checked_sub()` operations to use MathUnderflow error code.


<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [5a99d740](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/5a99d740eed89b28b6a81d6942f8495232a04a32).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Multiliquid:** fixed by 5a99d740eed89b28b6a81d6942f8495232a04a32
-->



### Typographical errors
<!--
Number: 10
Cantina code repository status: fixed
Hyperlink: [Issue 10](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/10)
Labels: [None]
Fixed on: [commit [3c79e75d](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/3c79e75d912dfb331e44b97bf758028db922b35c)]
-->


**Severity:** Informational

**Context:** [asset_config.rs#L4](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/state/asset_config.rs#L4), [asset_config.rs#L81](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/state/asset_config.rs#L81)


**Description:**

```diff
- /// NavData is used to store info about the methpds to retrieve the NAV data for a specific asset
+ /// NavData is used to store info about the methods to retrieve the NAV data for a specific asset
  
  
- // DRY function to read the price from an account (expecting the price to be stored as au64)
+ // DRY function to read the price from an account (expecting the price to be stored as u64)
```


<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [3c79e75d](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/3c79e75d912dfb331e44b97bf758028db922b35c).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Multiliquid:** fixed by 3c79e75d912dfb331e44b97bf758028db922b35c
-->



### NavData enum documentation is incomplete and misleading
<!--
Number: 11
Cantina code repository status: fixed
Hyperlink: [Issue 11](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/11)
Labels: [None]
Fixed on: [commit [9c51a57c](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/9c51a57ca0e5cdda592a9fc4b362538aff0a63d2)]
-->


**Severity:** Informational

**Context:** [asset_config.rs#L5-L11](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/state/asset_config.rs#L5-L11)


**Description:** The inline documentation for the `NavData` enum in `asset_config.rs` states that "Currently we support three methods" for retrieving NAV data. However, the actual implementation supports four distinct methods. The fourth variant, `PythPush`, is completely omitted from the documentation comment.


**Recommendation:** Update the inline documentation to reflect all four supported NAV data retrieval methods accurately.


<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [9c51a57c](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/9c51a57ca0e5cdda592a9fc4b362538aff0a63d2).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Multiliquid:** fixed by 9c51a57ca0e5cdda592a9fc4b362538aff0a63d2
-->



### Integer overflow in `min_acceptable_nav` calculation causes transaction failure for large NAV values
<!--
Number: 12
Cantina code repository status: fixed
Hyperlink: [Issue 12](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/12)
Labels: [None]
Fixed on: [commit [4cf5446b](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/4cf5446b3adfa8d12d8dc7f86a7007d6f3ffbec5)]
-->


**Severity:** Informational

**Context:** [asset_config.rs#L260-L264](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/state/asset_config.rs#L260-L264)


**Description:** The `get_nav()` function in `asset_config.rs` performs a price difference validation check that is vulnerable to arithmetic overflow when NAV values are significant (has less price decimals).

When `max_nav` is sufficiently large (e.g., 1e18 = 1e9 with zero decimals), the multiplication operation `max_nav.checked_mul(10000 -  price_difference_bps as u64)` overflows the `u64` type and returns `None`. The subsequent `.unwrap()` call then panics with the error `Option::unwrap()` on a None value.

This issue results in:                                                                                                                                
1. Denial of Service: Any asset with a NAV above approximately 1.84 × 10^15 (with 9 decimals) will cause all transactions to fail.
2. Unpredictable Failures: The issue only manifests when NAV reaches certain thresholds, making it difficult to diagnose.
3. No Graceful Error Handling: The transaction panics rather than returning a proper error code.


**Proof of Concept:**
- `max_nav` = 1_000_000_000_000_000_000 (1 quintillion, representing a price of 1e9 asset with 0 decimals)                                      - `price_difference_bps` = 500 (5% tolerance).
- Calculation: 1_000_000_000_000_000_000 × (10000 - 500) = 1_000_000_000_000_000_000 × 9500                                                  - Result: 9.5 × 10^21, which exceeds `u64::MAX` (18,446,744,073,709,551,615 ≈ 1.84 × 10^19)                                              - The `checked_mul()` returns `None`, causing a panic.


**Recommendation:**
1. Replace all `.unwrap()` calls with proper error handling using `.ok_or(ErrorCode::MathOverflow)?` which will return meaningful error.

2. Add comprehensive tests with edge cases: Maximum safe NAV values, Minimum price_difference_bps values, and Various combinations that approach u64 limits.

3. Document maximum supported NAV values in the code comments.


**Multiliquid:** Fixed by 4cf5446b3adfa8d12d8dc7f86a7007d6f3ffbec5 (for now, we've added error handling with MathOverflow).

The likelihood is really close to 0 (most of the stables will have a value around $1, and for RWAs, we can't imagine having such a NAV. Since it is not something a random user (the NAV) could set, we don't see a security issue here either.


**Cantina:** While the newly added error checks will detect and revert on overflows with more explicit error messages, the protocol's capacity to safely handle large NAV values remains constrained. It may still result in unintended or unpredictable behavior.

<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [4cf5446b](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/4cf5446b3adfa8d12d8dc7f86a7007d6f3ffbec5).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Cantina Managed:** Full PoC:

```js
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { SwapProgram } from "../target/types/swap_program";
import { expect } from "chai";
import { Keypair, PublicKey, SystemProgram } from "@solana/web3.js";
import {
  TOKEN_PROGRAM_ID,
  createMint,
  createAccount,
  mintTo,
  getAccount,
} from "@solana/spl-token";

describe("swap-program", () => {
  // Configure the client to use the local cluster
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);

  const program = anchor.workspace.SwapProgram as Program<SwapProgram>;

  const malicious = Keypair.generate();

  // Test state
  let globalConfigPda: PublicKey;
  let stableMint: PublicKey;
  let assetMint: PublicKey;
  const liquidityProvider = Keypair.generate();
  let lpStableTokenAccount: PublicKey;
  let lpAssetTokenAccount: PublicKey;

  before(async () => {
    // Derive the global config PDA
    [globalConfigPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("global_config")],
      program.programId
    );

    // Check if global config exists, if not initialize it
    try {
      await program.account.globalConfig.fetch(globalConfigPda);
      console.log("Global config already initialized");
    } catch (error) {
      // Global config doesn't exist, initialize it
      console.log("Initializing global config...");
      const feeWallet = provider.wallet.publicKey;
      const protocolFeesBps = 9900; // 1% fee

      await program.methods
        .initGlobalConfig(feeWallet, protocolFeesBps)
        .accounts({
          globalConfig: globalConfigPda,
          admin: provider.wallet.publicKey,
          systemProgram: SystemProgram.programId,
        })
        .rpc();

      console.log("Global config initialized successfully");
    }

    // Create mints for the pair
    console.log("Creating stable coin mint...");
    stableMint = await createMint(
      provider.connection,
      provider.wallet.payer,
      provider.wallet.publicKey,
      provider.wallet.publicKey,
      6 // USDC-like decimals
    );
    console.log("Stable mint created:", stableMint.toString());

    console.log("Creating asset token mint...");
    assetMint = await createMint(
      provider.connection,
      provider.wallet.payer,
      provider.wallet.publicKey,
      provider.wallet.publicKey,
      6 // decimals
    );
    console.log("Asset mint created:", assetMint.toString());

    // Initialize asset config for stable coin
    await initAssetConfig(stableMint, { stable: {} });
    console.log("Stable coin asset config initialized");
  });

  async function initAssetConfig(mint: PublicKey, assetType: any) {
    const [assetConfigPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("asset"), mint.toBuffer()],
      program.programId
    );

    const [feeVaultPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("fee_vault"), mint.toBuffer()],
      program.programId
    );

    const [programAuthorityPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("program_authority")],
      program.programId
    );

    const navData = [
      {
        hardcoded: {
          hardcodedPrice: new anchor.BN(1_000_000_000), // 1.0 with 9 decimals
          priceDecimals: 0,
        },
      },
    ];

    const priceDifferenceBps = 500; // 5% max difference

    await program.methods
      .initAssetConfigAccount(navData, priceDifferenceBps, assetType)
      .accounts({
        assetConfig: assetConfigPda,
        mintAddress: mint,
        admin: provider.wallet.publicKey,
        feeVaultTokenAccount: feeVaultPda,
        programAuthority: programAuthorityPda,
        globalConfig: globalConfigPda,
        tokenProgram: TOKEN_PROGRAM_ID,
        systemProgram: SystemProgram.programId,
      })
      .rpc();
  }

  it("Test get_nav for stable mint", async () => {
    console.log("\n=== Testing get_nav for stable mint ===");
    console.log("Stable Mint address:", stableMint.toString());

    const [stableAssetConfigPda] = PublicKey.findProgramAddressSync(
      [Buffer.from("asset"), stableMint.toBuffer()],
      program.programId
    );

    // Call test_get_nav instruction
    const nav = await program.methods
      .testGetNav()
      .accounts({
        mintAddress: stableMint,
        assetConfig: stableAssetConfigPda,
      })
      .view();

    console.log("NAV returned for stable mint:", nav.toString());

  });
});
```

**Cantina Managed:** I introduced a function to get the nav directly. If you need, I can share the lib.rs file too.

**Cantina Managed:** This issue runs deep... so the fix recommendation is vague and not finalized.

**Multiliquid:** fixed by 4cf5446b3adfa8d12d8dc7f86a7007d6f3ffbec5
(for now, i've added a error handling with MathOverflow)

I kind of disagree with the severity of this one too, to have such failure would imply to have NAV = $1 000 000 000 (with 9 decimals added). 

The likelihood is really close to 0 (most of the stables will have a value around $1, and for RWAs, i can't imagine having such NAV (may be we need to inform Multiliquid about this, because they are integrating the asset configs by themselves). Since it is not something that could be set by a random user (the NAV), I don't really see a security issue here neither.

**Cantina Managed:** downgraded to `informational`
-->



### Function `set_new_admin()` does not check for duplicate assignment
<!--
Number: 13
Cantina code repository status: fixed
Hyperlink: [Issue 13](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/13)
Labels: [None]
Fixed on: [commit [6510e1e9](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/6510e1e9fab6c1b639d58a5140d0230469fef08c)]
-->


**Severity:** Informational

**Context:** [set_new_admin.rs#L23](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/set_new_admin.rs#L23)


**Description:** The `set_new_admin()` function lacks validation to ensure that the new admin address differs from the current admin address. This allows the current admin to unnecessarily set themselves as the pending admin again, wasting transaction fees and emitting misleading events without actual state changes.


**Recommendation:** Add validation to prevent setting the new admin to the current admin address:

```rust
require!(new_admin != global_config.admin, ErrorCode::AlreadyAdmin);
```

<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [6510e1e9](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/6510e1e9fab6c1b639d58a5140d0230469fef08c).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Multiliquid:** fixed by 6510e1e9fab6c1b639d58a5140d0230469fef08c
-->



### Swap input amount limited to ~18.45 tokens for 18-decimal tokens due to `u64` type
<!--
Number: 14
Cantina code repository status: acknowledged
Hyperlink: [Issue 14](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/14)
Labels: [None]
Fixed on: [None]
-->


**Severity:** Informational

**Context:** [add_liquidity.rs#L56](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/add_liquidity.rs#L56), [remove_liquidity.rs#L58](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/remove_liquidity.rs#L58), [swap.rs#L165](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/swap.rs#L165)


**Description:** The swap instruction uses u64 to represent token amounts, which significantly restricts the maximum swappable amount for tokens with high decimal precision.

For tokens with 18 decimals, the maximum value of u64 (18,446,744,073,709,551,615) translates to only approximately 18.45 tokens in human-readable terms:

u64::MAX / 10^18 = 18,446,744,073,709,551,615 / 1,000,000,000,000,000,000 ≈ 18.45.

This limitation severely restricts the protocol's usability for:
- High-value swaps involving tokens with 18 decimals.
- Liquidity provision operations that require larger amounts.
- Any operation where users need to transact more than ~18 tokens.

The issue affects not just swap operations but potentially all instruction parameters that handle token amounts using u64, including `add_liquidity()`, `remove_liquidity()`, and related functions.


**Recommendation:** Replace u64 with u128 for all token amount parameters throughout the program. The u128 type provides sufficient range to handle tokens with up to 38 decimals at scale:

u128::MAX / 10^18 ≈ 340 trillion tokens.

<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Acknowledged.

**Cantina Managed:** Acknowledged.

**_COMMENTS_**:

**Multiliquid:** The max decimals on solana are 9, so this can't happen. Also, token balance are stored as a u64 in token accounts, so it doesn't make sense to be able to handle more than the max that they can have in balance

**Cantina Managed:** I would then suggest restricting tokens with greater than nine decimals while creating the asset config. Agree most tokens have nine decimals, but we can have 18 decimals too.  

Also the amount value in swap is instantly casted to u128 anyway.

**Cantina Managed:** downgraded to "informational" @fredericbry 

**Multiliquid:** you're right, i thought "9" was a hard limit, but it actually isn't. We can create a token with 18 decimals, i just did.
I don't see the point of restricting anything about decimals, because since token account balance are stored as u64, we are safe anyway, isn't it?

**Cantina Managed:** Should be relatively safe for now. no fund loss risk.

**Multiliquid:** We'll acknowledge this one, since it is impossible to swap more than what you can hold, thus in my opinion fix this is useless
-->



### Redundant `mint_address` assignment in `update_asset_config_account()`
<!--
Number: 15
Cantina code repository status: fixed
Hyperlink: [Issue 15](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/15)
Labels: [None]
Fixed on: [commit [c19af01f](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/c19af01ffd1cad84ffd89b1c4ffbaa00f40cc611)]
-->


**Severity:** Informational

**Context:** [update_asset_config_account.rs#L48](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/update_asset_config_account.rs#L48)


**Description:** The `update_asset_config_account()` function contains unnecessary computation where mint_address is assigned or validated.

Since the asset_config account is a PDA derived with mint_address as a seed, the Anchor framework already enforces that the account can only be accessed with the correct mint_address used in derivation.

Any assignment or validation of this field is redundant and wastes compute units.


**Recommendation:** Remove the redundant mint_address assignment or validation from the update_asset_config_account function.


<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [c19af01f](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/c19af01ffd1cad84ffd89b1c4ffbaa00f40cc611).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Multiliquid:** this has already been handled with commit c19af01ffd1cad84ffd89b1c4ffbaa00f40cc611
-->



### Redundant pause state initialization to default value
<!--
Number: 16
Cantina code repository status: fixed
Hyperlink: [Issue 16](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/16)
Labels: [None]
Fixed on: [commit [e8625298](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/e862529801c69e725a7b7eff9e1dfcc8bfa6f7e2)]
-->


**Severity:** Informational

**Context:** [init_asset_config_account.rs#L79](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/init_asset_config_account.rs#L79), [init_pair.rs#L144](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/init_pair.rs#L144)


**Description:** In the `init_pair()` and `init_asset_config_account()` functions, the pause state is explicitly set to false, which is already the default value for boolean fields in newly initialized accounts. This redundant assignment wastes compute units without providing any functional benefit.


**Recommendation:** Remove the explicit pause state assignment to false in both `init_pair()` and `init_asset_config_account()` functions.


**Multiliquid:** Fixed in [e8625298](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/e862529801c69e725a7b7eff9e1dfcc8bfa6f7e2) and [7f3591ab](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/7f3591ab48671c4007cba0dc51f3583763a312a3)

<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [e8625298](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/e862529801c69e725a7b7eff9e1dfcc8bfa6f7e2).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Multiliquid:** handled by commit 7f3591ab48671c4007cba0dc51f3583763a312a3

**Cantina Managed:** fix incomplete @fredericbry, should also remove in `init_asset_config_account()` function. 

**Multiliquid:** it has already been done with this commit: e862529801c69e725a7b7eff9e1dfcc8bfa6f7e2
-->



### Use `UncheckedAccount` instead of `AccountInfo`
<!--
Number: 17
Cantina code repository status: acknowledged
Hyperlink: [Issue 17](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/17)
Labels: [None]
Fixed on: [commit [c19af01f](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/c19af01ffd1cad84ffd89b1c4ffbaa00f40cc611)]
-->


**Severity:** Informational

**Context:** _(No context files were provided by the reviewer)_


**Description:** Multiple instructions across the entire program use `AccountInfo` instead of `UncheckedAccount`, against the anchor's suggestion.


**Recommendation:** Consider replacing all instances of `AccountInfo` in-favor of `UncheckedAccount`.


<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Acknowledged.

**Cantina Managed:** Acknowledged.

**_COMMENTS_**:

**Cantina Managed:** There are still uses of `AccountInfo` in 
- `close_pair.rs`
- `asset_config.rs`
- `spl_helpers.rs`

**Multiliquid:** handled by commit c19af01ffd1cad84ffd89b1c4ffbaa00f40cc611

**Cantina Managed:** Note to self: clone code and check if all instances are changed.
-->



### Unchecked `.unwrap()` in `SwapExecuted` event emission
<!--
Number: 18
Cantina code repository status: fixed
Hyperlink: [Issue 18](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/findings/18)
Labels: [None]
Fixed on: [commit [8abbd7f5](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/8abbd7f5bcf29d8bba3da7ded09259499e4547a9)]
-->


**Severity:** Informational

**Context:** [swap.rs#L331](https://cantina.xyz/code/8efd57ef-c9ac-42f8-8e03-4e4a1f0ab2a7/programs/swap-program/src/instructions/swap.rs#L331)


**Description:** The `amount_in_for_vault.checked_add(protocol_fees).unwrap()` could panic if the addition overflows.


**Recommendation:** Consider using `.ok_or` at the end of the checked_add to ensure the overflow is caught and reverted accordingly.



<!--
POTENTIAL RESOLUTION STATEMENT:

**Multiliquid:** Fixed in commit [8abbd7f5](https://github.com/exo-tech-xyz/multiliquid-swap-program/commit/8abbd7f5bcf29d8bba3da7ded09259499e4547a9).

**Cantina Managed:** Fix verified.

**_COMMENTS_**:

**Multiliquid:** handled by 8abbd7f5bcf29d8bba3da7ded09259499e4547a9
-->

