use anchor_lang::prelude::*;
use anchor_lang::solana_program::sysvar::clock::Clock;
use std::collections::HashMap;
use thiserror::Error;

declare_id!("CedarLending111111111111111111111111111111111");

// ──────────────────────────────────────────────────────────────────────────────
// Constants
// ──────────────────────────────────────────────────────────────────────────────

/// Basis points denominator (100% = 10000 bps)
const BASIS_POINTS_DENOMINATOR: u64 = 10_000;

/// Initial index value (1e18 precision)
const INITIAL_INDEX: u128 = 1_000_000_000_000_000_000;

/// Maximum basis points value
const MAX_BASIS_POINTS: u64 = 10_000;

/// Minimum liquidation bonus in basis points
const MIN_LIQUIDATION_BONUS: u64 = 100; // 1%

/// Maximum liquidation bonus in basis points
const MAX_LIQUIDATION_BONUS: u64 = 2000; // 20%

/// Maximum utilization rate in basis points
const MAX_UTILIZATION_RATE: u64 = 10_000;

/// Account discriminator size for Anchor accounts
const DISCRIMINATOR_SIZE: usize = 8;

// ──────────────────────────────────────────────────────────────────────────────
// Errors
// ──────────────────────────────────────────────────────────────────────────────

#[error_code]
pub enum CedarError {
    #[msg("Arithmetic overflow or underflow occurred")]
    ArithmeticOverflow,
    
    #[msg("Insufficient liquidity in the pool to fulfill the request")]
    InsufficientLiquidity,
    
    #[msg("User position is unhealthy and cannot perform this action")]
    UnhealthyPosition,
    
    #[msg("Oracle price feed returned an invalid price")]
    InvalidOraclePrice,
    
    #[msg("Oracle price data is stale and cannot be used")]
    StaleOraclePrice,
    
    #[msg("Unauthorized access: caller does not have required permissions")]
    Unauthorized,
    
    #[msg("Invalid pool configuration parameters")]
    InvalidPoolConfig,
    
    #[msg("Borrow amount exceeds the maximum borrow limit")]
    BorrowLimitExceeded,
    
    #[msg("Repay amount exceeds the outstanding debt")]
    RepayExceedsDebt,
    
    #[msg("Withdraw amount exceeds the available supply")]
    WithdrawExceedsSupply,
    
    #[msg("Liquidation is not profitable for the liquidator")]
    LiquidationNotProfitable,
    
    #[msg("Pool is currently paused and cannot perform operations")]
    PoolPaused,
    
    #[msg("Invalid input parameters provided")]
    InvalidInput,
    
    #[msg("Pool has reached its supply cap")]
    SupplyCapReached,
    
    #[msg("Pool has reached its borrow cap")]
    BorrowCapReached,
    
    #[msg("Oracle feed is not configured")]
    OracleNotConfigured,
    
    #[msg("Token mint mismatch")]
    TokenMintMismatch,
}

// ──────────────────────────────────────────────────────────────────────────────
// State Definitions
// ──────────────────────────────────────────────────────────────────────────────

/// Risk parameters for a lending pool
#[derive(AnchorSerialize, AnchorDeserialize, Clone, Debug)]
pub struct RiskParameters {
    /// Maximum loan-to-value ratio in basis points (e.g., 7500 = 75%)
    pub max_ltv: u64,
    
    /// Liquidation threshold in basis points (e.g., 8000 = 80%)
    pub liquidation_threshold: u64,
    
    /// Bonus for liquidators in basis points (e.g., 500 = 5%)
    pub liquidation_bonus: u64,
    
    /// Protocol reserve factor in basis points (e.g., 1000 = 10%)
    pub reserve_factor: u64,
    
    /// Maximum supply amount for the pool
    pub supply_cap: u64,
    
    /// Maximum borrow amount for the pool
    pub borrow_cap: u64,
}

impl RiskParameters {
    /// Validates risk parameters are within acceptable ranges
    pub fn validate(&self) -> Result<()> {
        require!(
            self.max_ltv <= BASIS_POINTS_DENOMINATOR,
            CedarError::InvalidPoolConfig
        );
        require!(
            self.liquidation_threshold <= BASIS_POINTS_DENOMINATOR,
            CedarError::InvalidPoolConfig
        );
        require!(
            self.liquidation_threshold >= self.max_ltv,
            CedarError::InvalidPoolConfig
        );
        require!(
            self.liquidation_bonus >= MIN_LIQUIDATION_BONUS 
            && self.liquidation_bonus <= MAX_LIQUIDATION_BONUS,
            CedarError::InvalidPoolConfig
        );
        require!(
            self.reserve_factor <= BASIS_POINTS_DENOMINATOR,
            CedarError::InvalidPoolConfig
        );
        Ok(())
    }
}

impl Default for RiskParameters {
    fn default() -> Self {
        Self {
            max_ltv: 7500,              // 75%
            liquidation_threshold: 8000, // 80%
            liquidation_bonus: 500,      // 5%
            reserve_factor: 1000,        // 10%
            supply_cap: u64::MAX,
            borrow_cap: u64::MAX,
        }
    }
}

/// Current state of a lending pool
#[derive(AnchorSerialize, AnchorDeserialize, Clone, Debug)]
pub struct PoolState {
    /// Total amount supplied to the pool
    pub total_supplied: u64,
    
    /// Total amount borrowed from the pool
    pub total_borrowed: u64,
    
    /// Total reserves accumulated by the protocol
    pub total_reserves: u64,
    
    /// Current supply index for interest accrual
    pub supply_index: u128,
    
    /// Current borrow index for interest accrual
    pub borrow_index: u128,
    
    /// Timestamp of the last state update
    pub last_update: i64,
    
    /// Current utilization rate in basis points
    pub utilization_rate: u64,
}

impl PoolState {
    /// Calculates the current utilization rate
    pub fn calculate_utilization_rate(&self) -> Result<u64> {
        if self.total_supplied == 0 {
            return Ok(0);
        }
        
        let utilization = self.total_borrowed
            .checked_mul(BASIS_POINTS_DENOMINATOR)
            .ok_or(CedarError::ArithmeticOverflow)?
            .checked_div(self.total_supplied)
            .ok_or(CedarError::ArithmeticOverflow)?;
            
        Ok(utilization.min(MAX_UTILIZATION_RATE))
    }
}

impl Default for PoolState {
    fn default() -> Self {
        Self {
            total_supplied: 0,
            total_borrowed: 0,
            total_reserves: 0,
            supply_index: INITIAL_INDEX,
            borrow_index: INITIAL_INDEX,
            last_update: 0,
            utilization_rate: 0,
        }
    }
}

/// State of a user's position in a pool
#[derive(AnchorSerialize, AnchorDeserialize, Clone, Debug)]
pub struct UserState {
    /// Amount supplied by the user
    pub supplied_amount: u64,
    
    /// Amount borrowed by the user
    pub borrowed_amount: u64,
    
    /// Supply index snapshot at last user activity
    pub supply_index_snapshot: u128,
    
    /// Borrow index snapshot at last user activity
    pub borrow_index_snapshot: u128,
    
    /// Timestamp of the last user activity
    pub last_activity: i64,
}

impl UserState {
    /// Checks if the user has any active position
    pub fn is_active(&self) -> bool {
        self.supplied_amount > 0 || self.borrowed_amount > 0
    }
}

impl Default for UserState {
    fn default() -> Self {
        Self {
            supplied_amount: 0,
            borrowed_amount: 0,
            supply_index_snapshot: 0,
            borrow_index_snapshot: 0,
            last_activity: 0,
        }
    }
}

// ──────────────────────────────────────────────────────────────────────────────
// Account Definitions
// ──────────────────────────────────────────────────────────────────────────────

/// Main lending pool account
#[account]
#[derive(Default)]
pub struct Pool {
    /// Authority that can manage pool parameters
    pub authority: Pubkey,
    
    /// Token mint for the pool
    pub token_mint: Pubkey,
    
    /// Current pool state
    pub pool_state: PoolState,
    
    /// Risk parameters for the pool
    pub risk_params: RiskParameters,
    
    /// Oracle feed for price data
    pub oracle_feed: Pubkey,
    
    /// Whether the pool is paused
    pub is_paused: bool,
    
    /// PDA bump seed
    pub bump: u8,
}

impl Pool {
    /// Validates that the pool is not paused
    pub fn check_not_paused(&self) -> Result<()> {
        require!(!self.is_paused, CedarError::PoolPaused);
        Ok(())
    }
    
    /// Validates that the pool has sufficient liquidity
    pub fn check_sufficient_liquidity(&self, amount: u64) -> Result<()> {
        let available = self.pool_state.total_supplied
            .checked_sub(self.pool_state.total_borrowed)
            .ok_or(CedarError::ArithmeticOverflow)?
            .checked_sub(self.pool_state.total_reserves)
            .ok_or(CedarError::ArithmeticOverflow)?;
            
        require!(available >= amount, CedarError::InsufficientLiquidity);
        Ok(())
    }
}

/// User position in a specific pool
#[account]
#[derive(Default)]
pub struct UserPosition {
    /// Owner of the position
    pub owner: Pubkey,
    
    /// Pool this position belongs to
    pub pool: Pubkey,
    
    /// User state data
    pub user_state: UserState,
    
    /// PDA bump seed
    pub bump: u8,
}

impl UserPosition {
    /// Validates that the caller is the position owner
    pub fn check_owner(&self, caller: &Pubkey) -> Result<()> {
        require!(self.owner == *caller, CedarError::Unauthorized);
        Ok(())
    }
}

/// Oracle feed configuration
#[account]
#[derive(Default)]
pub struct OracleFeed {
    /// Oracle program ID
    pub oracle_program: Pubkey,
    
    /// Price feed account
    pub price_feed: Pubkey,
    
    /// Maximum age of price data in seconds
    pub max_age: i64,
    
    /// Number of decimals for the price feed
    pub decimals: u8,
}

impl OracleFeed {
    /// Validates that the oracle feed is properly configured
    pub fn validate(&self) -> Result<()> {
        require!(
            self.oracle_program != Pubkey::default(),
            CedarError::OracleNotConfigured
        );
        require!(
            self.price_feed != Pubkey::default(),
            CedarError::OracleNotConfigured
        );
        require!(self.max_age > 0, CedarError::InvalidOraclePrice);
        Ok(())
    }
}

/// Global protocol configuration
#[account]
#[derive(Default)]
pub struct ProtocolConfig {
    /// Admin authority
    pub admin: Pubkey,
    
    /// Fee recipient address
    pub fee_recipient: Pubkey,
    
    /// Protocol fee rate in basis points
    pub protocol_fee_rate: u64,
    
    /// Minimum liquidation bonus in basis points
    pub min_liquidation_bonus: u64,
    
    /// Maximum liquidation bonus in basis points
    pub max_liquidation_bonus: u64,
    
    /// PDA bump seed
    pub bump: u8,
}

impl ProtocolConfig {
    /// Validates protocol configuration parameters
    pub fn validate(&self) -> Result<()> {
        require!(
            self.protocol_fee_rate <= BASIS_POINTS_DENOMINATOR,
            CedarError::InvalidInput
        );
        require!(
            self.min_liquidation_bonus <= self.max_liquidation_bonus,
            CedarError::InvalidInput
        );
        Ok(())
    }
}

// ──────────────────────────────────────────────────────────────────────────────
// Instruction Contexts
// ──────────────────────────────────────────────────────────────────────────────

/// Initialize a new lending pool
#[derive(Accounts)]
pub struct InitPool<'info> {
    /// Pool account to be initialized
    #[account(
        init,
        payer = authority,
        space = DISCRIMINATOR_SIZE + std::mem::size_of::<Pool>(),
        seeds = [b"pool", token_mint.key().as_ref()],
        bump
    )]
    pub pool: Account<'info, Pool>,
    
    /// Authority creating the pool
    #[account(mut)]
    pub authority: Signer<'info>,
    
    /// Token mint for the pool
    pub token_mint: Account<'info, token::Mint>,
    
    /// Oracle feed for price data
    pub oracle_feed: Account<'info, OracleFeed>,
    
    /// System program
    pub system_program: Program<'info, System>,
}

/// Supply tokens to a pool
#[derive(Accounts)]
pub struct Supply<'info> {
    /// Pool to supply to
    #[account(
        mut,
        has_one = token_mint @ CedarError::TokenMintMismatch
    )]
    pub pool: Account<'info, Pool>,
    
    /// User position account (created if needed)
    #[account(
        init_if_needed,
        payer = user,
        space = DISCRIMINATOR_SIZE + std::mem::size_of::<UserPosition>(),
        seeds = [b"position", pool.key().as_ref(), user.key().as_ref()],
        bump
    )]
    pub user_position: Account<'info, UserPosition>,
    
    /// User supplying tokens
    #[account(mut)]
    pub user: Signer<'info>,
    
    /// Token mint
    pub token_mint: Account<'info, token::Mint>,
    
    /// User's token account
    #[account(mut)]
    pub user_token_account: Account<'info, token::TokenAccount>,
    
    /// Pool's token account
    #[account(mut)]
    pub pool_token_account: Account<'info, token::TokenAccount>,
    
    /// Token program
    pub token_program: Program<'info, token::Token>,
    
    /// System program
    pub system_program: Program<'info, System>,
}

/// Withdraw tokens from a pool
#[derive(Accounts)]
pub struct Withdraw<'info> {
    /// Pool to withdraw from
    #[account(
        mut,
        has_one = token_mint @ CedarError::TokenMintMismatch
    )]
    pub pool: Account<'info, Pool>,
    
    /// User position account
    #[account(
        mut,
        seeds = [b"position", pool.key().as_ref(), user.key().as_ref()],
        bump = user_position.bump,
        has_one = owner @ CedarError::Unauthorized
    )]
    pub user_position: Account<'info, UserPosition>,
    
    /// User withdrawing tokens
    pub user: Signer<'info>,
    
    /// Token mint
    pub token_mint: Account<'info, token::Mint>,
    
    /// User's token account
    #[account(mut)]
    pub user_token_account: Account<'info, token::TokenAccount>,
    
    /// Pool's token account
    #[account(mut)]
    pub pool_token_account: Account<'info, token::TokenAccount>,
    
    /// Token program
    pub token_program: Program<'info, token::Token>,
}

/// Borrow tokens from a pool
#[derive(Accounts)]
pub struct Borrow<'info> {
    /// Pool to borrow from
    #[account(
        mut,
        has_one = oracle_feed @ CedarError::InvalidPoolConfig
    )]
    pub pool: Account<'info, Pool>,
    
    /// User position account
    #[account(
        mut,
        seeds = [b"position", pool.key().as_ref(), user.key().as_ref()],
        bump = user_position.bump,
        has_one = owner @ CedarError::Unauthorized
    )]
    pub user_position: Account<'info, UserPosition>,
    
    /// User borrowing tokens
    pub user: Signer<'info>,
    
    /// Oracle feed for price data
    pub oracle_feed: Account<'info, OracleFeed>,
    
    /// Token mint
    pub token_mint: Account<'info, token::Mint>,
    
    /// User's token account
    #[account(mut)]
    pub user_token_account: Account<'info, token::TokenAccount>,
    
    /// Pool's token account
    #[account(mut)]
    pub pool_token_account: Account<'info, token::TokenAccount>,
    
    /// Token program
    pub token_program: Program<'info, token::Token>,
}

/// Repay borrowed tokens
#[derive(Accounts)]
pub struct Repay<'info> {
    /// Pool to repay to
    #[account(
        mut,
        has_one = token_mint @ CedarError::TokenMintMismatch
    )]
    pub pool: Account<'info, Pool>,
    
    /// User position account
    #[account(
        mut,
        seeds = [b"position", pool.key().as_ref(), user.key().as_ref()],
        bump = user_position.bump,
        has_one = owner @ CedarError::Unauthorized
    )]
    pub user_position: Account<'info, UserPosition>,
    
    /// User repaying tokens
    pub user: Signer<'info>,
    
    /// Token mint
    pub token_mint: Account<'info, token::Mint>,
    
    /// User's token account
    #[account(mut)]
    pub user_token_account: Account<'info, token::TokenAccount>,
    
    /// Pool's token account
    #[account(mut)]
    pub pool_token_account: Account<'info, token::TokenAccount>,
    
    /// Token program
    pub token_program: Program<'info, token::Token>,
}

/// Liquidate an unhealthy position
#[derive(Accounts)]
pub struct Liquidate<'info> {
    /// Pool being liquidated
    #[account(
        mut,
        has_one = oracle_feed @ CedarError::InvalidPoolConfig,
        has_one = token_mint @ CedarError::TokenMintMismatch
    )]
    pub pool: Account<'info, Pool>,
    
    /// User position being liquidated
    #[account(
        mut,
        seeds = [b"position", pool.key().as_ref(), borrower.key().as_ref()],
        bump = user_position.bump
    )]
    pub user_position: Account<'info, UserPosition>,
    
    /// Borrower being liquidated
    /// CHECK: Validated through position seeds