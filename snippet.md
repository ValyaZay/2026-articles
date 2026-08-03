```rust
#[test]
fn swap_base_input_randomized_statefull_fuzz() {
    let debug = false;
    let assert = false;
    

    let mut ctx = AnchorLiteSVM::build_with_program(self::raydium_cp_swap::ID, PROGRAM_BYTES);
    let seed: u64 = 344;
    let mut rng = StdRng::seed_from_u64(seed);
 
    let pools = [
        "HtNfUbDaBamCBPWCFiESkXpewvwVLkwrSWRjjV8FNT7i",
        "BwDCiMxHpWQB31qmLUiNyJvjdZmNChvEMv99ZcWVrciT",
        "9TWza3BboPuuEopdm3kKSh7Cs8h9qnUoF7nn988p4dYf",
        "EfcAPyAQz5QyjA4a2chEKbueRprVf5zL238cWtcCmHej",
        "Cvhwfa9yd35U26jGctrMwgYWq43cC5f5GBi26NiNE2QF",
        "EojCwRSiC3gy26DrF37Z7S2KYdXPRhL8nmHDWmXYZzw3",
        "9UV4xi4qNXahp1zZRjiDEzAfNJcmUFmszweCKcMeM35C",
        "8xUd2YYKbobsAHfMDbuTxXmo6KwNvSouj9TKWJ8fZoWR",
        "B9oG5vDnfLf9zK3sN5JChdDwk6mkt3MNBZsra8SkcgJA",
        "Gx5mtdtSqRorL5a43L8UJW6kMxZU3HHMxfkMzTbt9aBk",
        "2YH3VQHLsnvagLX6AabRqfU9Cdyt5QxhqtSvfyxSbtuX",
        "F3MWZW6npxZRvmsTzXhCdXiGxqEiPPMwY4xgcciAX6Db",
        "J9ED7D3pR7Uw5W6Y52p1Mq3Gfkmumg8fHRvLEiHLL2S7",
        "4XpLnpciABBDRKegqG4X3XuNYsoacTKwtbyvrUynw3WN",
        "J5xGFo7z6JbuZMkpQEMxeS9t1znrccvmHR8EFgAZKdCq",
        "B6aPWyX7pb5N1or4f9q3LFvZeJvqBxgRWLH8p3k8fwvu",
        "6q4hnoLn2LVkioe8HaBSYLZQR7aGSwknUAnt2v9ecjSh",
        "BKkBMYFDtdFdH4uQJf2jepkMAAEraRod5iZ1pVFAGWRV",
        "ExegPzjmGQ9cBXYqrdDVJwvJci1UbmMxwagMuKevwuxC",
        "BC9XPnRvsN9DgQQHoLYiFNedHSgXfGGmVNoJYjL3Zujt",
    ];

    // set pools one time
    let mut pool_to_data: HashMap<Pubkey, SetAccountsResult> = HashMap::new(); 
    for pool_key in pools {
            let pool_key = Pubkey::from_str(pool_key).unwrap();
            let set_accounts_result = set_accounts_from_blockchain_and_get_their_data(pool_key, &mut ctx);
            pool_to_data.insert(pool_key, set_accounts_result);
    }
     
    
    for step in 0..10000 {
        // debug info
        let mut debug_pool_state_before: Account = Account::default();
        let mut debug_input_vault_before: Account = Account::default();

        // random pool key
        let pool_key_index = rng.random_range(0..pools.len());
        let pool_key = Pubkey::from_str(pools[pool_key_index]).unwrap();
        
        let set_accounts_result = &pool_to_data[&pool_key];
        
        // random amount_in
        let amount_to_swap: u64 = rng.random_range(10..1000);
        println!("------- {}", step);
        println!("{pool_key} - amount_in {amount_to_swap}");
        
        let pool_state: PoolState = ctx.get_account(&pool_key).unwrap();//is zero-copy
        if debug {
            debug_pool_state_before = create_debug_pool_state_account(pool_state);
        }
        println!("pool {:#?}", pool_state);

        let swapper = &set_accounts_result.swapper;
        println!("swapper_key {}", swapper.pubkey());

        let zero_for_one: bool = rng.random_bool(1.0 / 2.0);
        println!("zero_for_one {zero_for_one}");
        let input_output = if zero_for_one {
            InputOutput {
                input_token_account: set_accounts_result.swapper_token_acc_0_key,
                output_token_account: set_accounts_result.swapper_token_acc_1_key,
                input_vault: pool_state.token_0_vault,
                output_vault: pool_state.token_1_vault,
                input_token_program: pool_state.token_0_program,
                output_token_program: pool_state.token_1_program,
                input_token_mint: pool_state.token_0_mint,
                output_token_mint: pool_state.token_1_mint,
                input_mint_decimals: pool_state.mint_0_decimals,
            }
        } else {
            InputOutput {
                input_token_account: set_accounts_result.swapper_token_acc_1_key,
                output_token_account: set_accounts_result.swapper_token_acc_0_key,
                input_vault: pool_state.token_1_vault,
                output_vault: pool_state.token_0_vault,
                input_token_program: pool_state.token_1_program,
                output_token_program: pool_state.token_0_program,
                input_token_mint: pool_state.token_1_mint,
                output_token_mint: pool_state.token_0_mint,
                input_mint_decimals: pool_state.mint_1_decimals,
            }
        };

        let amount_to_swap_with_decimals = amount_to_swap * 10u64.checked_pow(input_output.input_mint_decimals.into()).ok_or(MathOverflow).unwrap();

        // get accounts to assert on
        let input_vault_before: TokenAccount = ctx.get_account(&input_output.input_vault).unwrap();
        if debug {
            debug_input_vault_before = create_debug_token_account(input_vault_before);
        }        
        println!("input_vault {:#?}", input_vault_before);
        let output_vault_before: TokenAccount = ctx.get_account(&input_output.output_vault).unwrap();
        let swapper_input_token_account_before: TokenAccount = ctx.get_account(&input_output.input_token_account).unwrap();
        println!("swapper_input_token_account_before {:#?}", swapper_input_token_account_before);
        let swapper_output_token_account_before: TokenAccount = ctx.get_account(&input_output.output_token_account).unwrap();

        let mut clock: Clock = ctx.svm.get_sysvar::<Clock>();
        clock.unix_timestamp = pool_state.open_time as i64 + 1;
        //clock.epoch = 115; // todo - is it crucial to set some close to real epoch? this is stubbed now in get_transfer_fee
        ctx.svm.set_sysvar(&clock);
        
        let swap_accounts = accounts::SwapBaseInput {
            payer: swapper.pubkey(),
            authority: set_accounts_result.authority,
            amm_config: pool_state.amm_config,
            pool_state: pool_key,
            input_token_account: input_output.input_token_account,
            output_token_account: input_output.output_token_account,
            input_vault: input_output.input_vault,
            output_vault: input_output.output_vault,
            input_token_program: input_output.input_token_program,
            output_token_program: input_output.output_token_program,
            input_token_mint: input_output.input_token_mint,
            output_token_mint: input_output.output_token_mint,
            observation_state: pool_state.observation_key,
        };
        
        let swap_inx = ctx
            .program()
            .accounts(swap_accounts)
            .args(args::SwapBaseInput {
                amount_in: amount_to_swap_with_decimals,
                minimum_amount_out: 0
            })
            .instruction()
            .unwrap();
        let swap_tx_result = ctx.execute_instruction(swap_inx, &[&swapper]).unwrap();
        //println!("{:#?}", swap_tx_result);

        // assert accounts state after - it is an amount_out
        let swapper_input_token_account_after: TokenAccount = ctx.get_account(&input_output.input_token_account).unwrap();
        assert_eq!(swapper_input_token_account_after.amount, swapper_input_token_account_before.amount - amount_to_swap_with_decimals);

        let swapper_output_token_account_after: TokenAccount = ctx.get_account(&input_output.output_token_account).unwrap();
        let swap_amount_out = swapper_output_token_account_after.amount - swapper_output_token_account_before.amount;
        println!("swap::amount_out {}", swap_amount_out);// will be without transfer fees!!!

        let pool_state_after_swap: PoolState = ctx.get_account(&pool_key).unwrap();

        // check vaults reserves!!!
        let input_vault_after: TokenAccount = ctx.get_account(&input_output.input_vault).unwrap();
        assert_eq!(input_vault_after.amount, input_vault_before.amount + amount_to_swap_with_decimals);
        let output_vault_after: TokenAccount = ctx.get_account(&input_output.output_vault).unwrap();
        assert_eq!(output_vault_after.amount, output_vault_before.amount - swap_amount_out);

        // Calculation part
        let calculation_data = AmountOutCalculationData { 
            pool_state: pool_state,
            input_vault_amount: input_vault_before.amount, 
            output_vault_amount: output_vault_before.amount, 
            input_vault: input_output.input_vault, 
            output_vault: input_output.output_vault, 
            mint_0_account: &set_accounts_result.mint_0_acc, 
            mint_1_account: &set_accounts_result.mint_1_acc, 
            creator_fee_rate: set_accounts_result.amm_config.creator_fee_rate,
            trade_fee_rate: set_accounts_result.amm_config.trade_fee_rate, 
        };

        let calculated_amount_out = calculate_amount_out(calculation_data, amount_to_swap_with_decimals);
        println!("calculated_amount_out {}", calculated_amount_out);
        assert_eq!(calculated_amount_out, swap_amount_out);

        if debug {
            // record debug model to a file
            let debug_model = DebugModel {
                pool: pool_key, 
                pool_state_before: (pool_key, debug_pool_state_before),
                input_vault_before: (input_output.input_vault, debug_input_vault_before),
            };
            write_to_file(debug_model);
        }

        // roll block
        ctx.svm.advance_slot(500);
        ctx.svm.expire_blockhash();

    }
   
}
```