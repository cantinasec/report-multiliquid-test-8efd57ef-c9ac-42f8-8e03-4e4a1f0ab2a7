<!--
If the scope is a subset of the files present in the project at the reviewed commit hash, add it here.
This content will appear as a subsection titled "Scope" under the Executive Summary section.
The subsection will be prefixed with:

"The security review had the following components in scope:"

Format the content accordingly.
-->


```bash
programs/swap-program
├── Cargo.toml
├── src
│   ├── constants.rs
│   ├── error.rs
│   ├── instructions
│   │   ├── add_liquidity.rs
│   │   ├── claim_fees.rs
│   │   ├── close_pair.rs
│   │   ├── confirm_new_admin.rs
│   │   ├── init_asset_config_account.rs
│   │   ├── init_global_config.rs
│   │   ├── init_pair.rs
│   │   ├── mod.rs
│   │   ├── remove_liquidity.rs
│   │   ├── set_new_admin.rs
│   │   ├── set_paused_for_asset.rs
│   │   ├── set_paused_for_lp_stable_config.rs
│   │   ├── swap.rs
│   │   ├── update_asset_config_account.rs
│   │   ├── update_global_config.rs
│   │   └── update_pair.rs
│   ├── lib.rs
│   ├── macros.rs
│   ├── math.rs
│   ├── spl_helpers.rs
│   └── state
│       ├── asset_config.rs
│       ├── global_config.rs
│       ├── lp_stable_config.rs
│       ├── mod.rs
│       ├── pair.rs
│       └── user_vault_info.rs
└── Xargo.toml
```