//! # StreamDEX - Privacy-Preserving DEX on LEZ
//!
//! This crate implements a privacy-preserving Automated Market Maker (AMM)
//! using the deshield → swap → reshield pattern with ephemeral vault PDAs.
//!
//! ## Architecture
//!
//! - **Shielded Account Program**: Private balance management with encrypted amounts
//! - **AMM Pool Program**: Constant product AMM with x*y=k invariant
//! - **Vault PDA Module**: Ephemeral single-use accounts for private swaps
//!
//! ## Security
//!
//! - Formal verification with Kani for all math operations
//! - Property-based testing with proptest for edge cases
//! - No integer overflow/underflow guaranteed
//! - Replay protection via monotonically increasing nonces

#![deny(unsafe_code)]
#![deny(missing_docs)]
#![deny(clippy::all)]
#![deny(clippy::cargo)]
#![forbid(arithmetic_overflow)]
#![cfg_attr(not(test), deny(unused_crate_dependencies))]

use std::convert::TryInto;
use std::fmt;
use std::num::NonZeroU64;

use log::{debug, error, info, warn};
use serde::{Deserialize, Serialize};
use thiserror::Error;

// ---------------------------------------------------------------------------
// Constants
// ---------------------------------------------------------------------------

/// Base fee percentage (0.3%) represented as basis points
const BASE_FEE_BPS: u64 = 30;

/// Protocol fee percentage (0.05%) represented as basis points
const PROTOCOL_FEE_BPS: u64 = 5;

/// Maximum basis points (100%)
const MAX_BPS: u64 = 10_000;

/// Minimum liquidity threshold to prevent dust attacks
const MINIMUM_LIQUIDITY: u64 = 1_000;

/// Maximum swap amount as fraction of pool (90% to prevent manipulation)
const MAX_SWAP_FRACTION_BPS: u64 = 9_000;

/// Gas buffer for ephemeral vault operations
const GAS_BUFFER: u64 = 10_000;

// ---------------------------------------------------------------------------
// Error Types
// ---------------------------------------------------------------------------

/// Comprehensive error enumeration for all StreamDEX operations
#[derive(Error, Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum StreamDexError {
    // --- Arithmetic Errors ---
    #[error("Integer overflow during {operation}")]
    IntegerOverflow { operation: &'static str },

    #[error("Integer underflow during {operation}")]
    IntegerUnderflow { operation: &'static str },

    #[error("Division by zero in {operation}")]
    DivisionByZero { operation: &'static str },

    // --- AMM Errors ---
    #[error("Insufficient liquidity in pool {pool_id}")]
    InsufficientLiquidity { pool_id: String },

    #[error("Swap amount {amount} exceeds maximum allowed {max_amount}")]
    SwapAmountExceeded { amount: u64, max_amount: u64 },

    #[error("Invalid pool state: {reason}")]
    InvalidPoolState { reason: String },

    #[error("Slippage tolerance exceeded: expected {expected}, actual {actual}")]
    SlippageExceeded { expected: u64, actual: u64 },

    // --- Vault Errors ---
    #[error("Vault {vault_id} not found or already closed")]
    VaultNotFound { vault_id: String },

    #[error("Vault {vault_id} has insufficient balance: {balance} < {required}")]
    VaultInsufficientBalance {
        vault_id: String,
        balance: u64,
        required: u64,
    },

    #[error("Nonce {nonce} already used for vault creation")]
    NonceAlreadyUsed { nonce: u64 },

    // --- Shielded Account Errors ---
    #[error("Shielded account {account_id} not found")]
    ShieldedAccountNotFound { account_id: String },

    #[error("Insufficient shielded balance: {balance} < {required}")]
    InsufficientShieldedBalance { balance: u64, required: u64 },

    #[error("Invalid encryption key for shielded account")]
    InvalidEncryptionKey,

    // --- Authorization Errors ---
    #[error("Unauthorized: caller {caller} is not the owner of {resource}")]
    Unauthorized {
        caller: String,
        resource: String,
    },

    #[error("Signature verification failed")]
    SignatureVerificationFailed,

    // --- General Errors ---
    #[error("Invalid input: {details}")]
    InvalidInput { details: String },

    #[error("Pool {pool_id} is paused for maintenance")]
    PoolPaused { pool_id: String },

    #[error("Internal error: {details}")]
    InternalError { details: String },
}

impl StreamDexError {
    /// Returns `true` if the error is recoverable (e.g., can retry)
    pub fn is_recoverable(&self) -> bool {
        matches!(
            self,
            StreamDexError::InsufficientLiquidity { .. }
                | StreamDexError::SwapAmountExceeded { .. }
                | StreamDexError::SlippageExceeded { .. }
        )
    }
}

// ---------------------------------------------------------------------------
// Type Aliases
// ---------------------------------------------------------------------------

/// Result type alias for StreamDEX operations
pub type StreamDexResult<T> = Result<T, StreamDexError>;

/// Unique identifier for a pool (e.g., "USDC-LEZ")
pub type PoolId = String;

/// Unique identifier for a vault (PDA address)
pub type VaultId = String;

/// Unique identifier for a shielded account
pub type AccountId = String;

/// Basis points representation (1 BPS = 0.01%)
pub type BasisPoints = u64;

// ---------------------------------------------------------------------------
// Core Data Structures
// ---------------------------------------------------------------------------

/// Represents a liquidity pool with constant product AMM
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct Pool {
    /// Unique pool identifier
    pub id: PoolId,

    /// Reserve amount of token A (e.g., USDC)
    pub reserve_a: u64,

    /// Reserve amount of token B (e.g., LEZ)
    pub reserve_b: u64,

    /// Total liquidity tokens minted
    pub total_liquidity: u64,

    /// Base fee in basis points
    pub fee_bps: BasisPoints,

    /// Protocol fee in basis points
    pub protocol_fee_bps: BasisPoints,

    /// Whether the pool is active
    pub is_active: bool,

    /// Pool creator's address
    pub creator: String,

    /// Timestamp of pool creation
    pub created_at: u64,

    /// Last swap timestamp
    pub last_swap_at: u64,

    /// Cumulative volume in token A
    pub cumulative_volume_a: u128,

    /// Cumulative volume in token B
    pub cumulative_volume_b: u128,

    /// Number of swaps executed
    pub swap_count: u64,
}

impl Pool {
    /// Creates a new pool with validated parameters
    pub fn new(
        id: PoolId,
        reserve_a: u64,
        reserve_b: u64,
        creator: String,
        created_at: u64,
    ) -> StreamDexResult<Self> {
        // Validate reserves
        if reserve_a == 0 || reserve_b == 0 {
            return Err(StreamDexError::InvalidPoolState {
                reason: "Initial reserves must be non-zero".to_string(),
            });
        }

        if reserve_a < MINIMUM_LIQUIDITY || reserve_b < MINIMUM_LIQUIDITY {
            return Err(StreamDexError::InvalidPoolState {
                reason: format!(
                    "Initial reserves must be at least {}",
                    MINIMUM_LIQUIDITY
                ),
            });
        }

        Ok(Self {
            id,
            reserve_a,
            reserve_b,
            total_liquidity: 0,
            fee_bps: BASE_FEE_BPS,
            protocol_fee_bps: PROTOCOL_FEE_BPS,
            is_active: true,
            creator,
            created_at,
            last_swap_at: created_at,
            cumulative_volume_a: 0,
            cumulative_volume_b: 0,
            swap_count: 0,
        })
    }

    /// Returns the constant product k = reserve_a * reserve_b
    #[inline]
    pub fn constant_product(&self) -> StreamDexResult<u128> {
        let a = u128::from(self.reserve_a);
        let b = u128::from(self.reserve_b);
        a.checked_mul(b).ok_or(StreamDexError::IntegerOverflow {
            operation: "constant_product",
        })
    }

    /// Calculates output amount for a given input amount using constant product formula
    ///
    /// # Arguments
    ///
    /// * `input_amount` - Amount of input tokens
    /// * `input_reserve` - Current reserve of input token
    /// * `output_reserve` - Current reserve of output token
    ///
    /// # Returns
    ///
    /// The calculated output amount after fees
    ///
    /// # Errors
    ///
    /// Returns `StreamDexError::IntegerOverflow` if arithmetic overflows
    /// Returns `StreamDexError::DivisionByZero` if division by zero occurs
    /// Returns `StreamDexError::InvalidPoolState` if pool state is invalid
    pub fn calculate_output_amount(
        input_amount: u64,
        input_reserve: u64,
        output_reserve: u64,
        fee_bps: BasisPoints,
    ) -> StreamDexResult<u64> {
        // Validate inputs
        if input_amount == 0 {
            return Err(StreamDexError::InvalidInput {
                details: "Input amount must be non-zero".to_string(),
            });
        }

        if input_reserve == 0 || output_reserve == 0 {
            return Err(StreamDexError::InvalidPoolState {
                reason: "Reserves must be non-zero".to_string(),
            });
        }

        if fee_bps > MAX_BPS {
            return Err(StreamDexError::InvalidInput {
                details: format!("Fee {} exceeds maximum {}", fee_bps, MAX_BPS),
            });
        }

        // Calculate fee amount
        let fee_amount = input_amount
            .checked_mul(fee_bps)
            .ok_or(StreamDexError::IntegerOverflow {
                operation: "fee_amount_mul",
            })?
            .checked_div(MAX_BPS)
            .ok_or(StreamDexError::DivisionByZero {
                operation: "fee_amount_div",
            })?;

        // Calculate input amount after fee
        let input_after_fee = input_amount
            .checked_sub(fee_amount)
            .ok_or(StreamDexError::IntegerUnderflow {
                operation: "input_after_fee",
            })?;

        // Calculate numerator: input_after_fee * output_reserve
        let numerator = u128::from(input_after_fee)
            .checked_mul(u128::from(output_reserve))
            .ok_or(StreamDexError::IntegerOverflow {
                operation: "numerator",
            })?;

        // Calculate denominator: input_reserve + input_after_fee
        let denominator = u128::from(input_reserve)
            .checked_add(u128::from(input_after_fee))
            .ok_or(StreamDexError::IntegerOverflow {
                operation: "denominator",
            })?;

        // Calculate output amount
        let output_amount = numerator
            .checked_div(denominator)
            .ok_or(StreamDexError::DivisionByZero {
                operation: "output_amount",
            })?;

        // Convert back to u64 with overflow check
        let result = u64::try_from(output_amount).map_err(|_| StreamDexError::IntegerOverflow {
            operation: "output_amount_conversion",
        })?;

        // Validate output is less than input (basic economic check)
        if result >= input_amount {
            return Err(StreamDexError::InvalidPoolState {
                reason: "Output amount must be less than input amount".to_string(),
            });
        }

        Ok(result)
    }

    /// Executes a swap on the pool
    ///
    /// # Arguments
    ///
    /// * `input_amount` - Amount of input tokens to swap
    /// * `is_token_a_input` - Whether token A is the input token
    /// * `min_output_amount` - Minimum output amount (slippage protection)
    ///
    /// # Returns
    ///
    /// The actual output amount received
    ///
    /// # Errors
    ///
    /// Returns `StreamDexError::PoolPaused` if pool is paused
    /// Returns `StreamDexError::InsufficientLiquidity` if pool has insufficient liquidity
    /// Returns `StreamDexError::SwapAmountExceeded` if swap amount exceeds maximum
    /// Returns `StreamDexError::SlippageExceeded` if slippage tolerance is exceeded
    pub fn execute_swap(
        &mut self,
        input_amount: u64,
        is_token_a_input: bool,
        min_output_amount: u64,
    ) -> StreamDexResult<u64> {
        // Check if pool is active
        if !self.is_active {
            return Err(StreamDexError::PoolPaused {
                pool_id: self.id.clone(),
            });
        }

        // Validate input amount
        if input_amount == 0 {
            return Err(StreamDexError::InvalidInput {
                details: "Swap input amount must be non-zero".to_string(),
            });
        }

        // Calculate maximum swap amount based on pool reserves
        let (input_reserve, output_reserve) = if is_token_a_input {
            (self.reserve_a, self.reserve_b)
        } else {
            (self.reserve_b, self.reserve_a)
        };

        // Check if pool has sufficient liquidity
        if input_reserve == 0 || output_reserve == 0 {
            return Err(StreamDexError::InsufficientLiquidity {
                pool_id: self.id.clone(),
            });
        }

        // Calculate maximum allowed swap amount
        let max_swap_amount = input_reserve
            .checked_mul(MAX_SWAP_FRACTION_BPS)
            .ok_or(StreamDexError::IntegerOverflow {
                operation: "max_swap_amount_mul",
            })?
            .checked_div(MAX_BPS)
            .ok_or(StreamDexError::DivisionByZero {
                operation: "max_swap_amount_div",
            })?;

        // Check if swap amount exceeds maximum
        if input_amount > max_swap_amount {
            return Err(StreamDexError::SwapAmountExceeded {
                amount: input_amount,
                max_amount: max_swap_amount,
            });
        }

        // Calculate output amount
        let output_amount = Self::calculate_output_amount(
            input_amount,
            input_reserve,
            output_reserve,
            self.fee_bps,
        )?;

        // Check slippage tolerance
        if output_amount < min_output_amount {
            return Err(StreamDexError::SlippageExceeded {
                expected: min_output_amount,
                actual: output_amount,
            });
        }

        // Update reserves
        if is_token_a_input {
            self.reserve_a = self
                .reserve_a
                .checked_add(input_amount)
                .ok_or(StreamDexError::IntegerOverflow {
                    operation: "reserve_a_add",
                })?;
            self.reserve_b = self
                .reserve_b
                .checked_sub(output_amount)
                .ok_or(StreamDexError::IntegerUnderflow {
                    operation: "reserve_b_sub",
                })?;
        } else {
            self.reserve_b = self
                .reserve_b
                .checked_add(input_amount)
                .ok_or(StreamDexError::IntegerOverflow {
                    operation: "reserve_b_add",
                })?;
            self.reserve_a = self
                .reserve_a
                .checked_sub(output_amount)
                .ok_or(StreamDexError::IntegerUnderflow {
                    operation: "reserve_a_sub",
                })?;
        }

        // Update pool statistics
        self.swap_count = self
            .swap_count
            .checked_add(1)
            .ok_or(StreamDexError::IntegerOverflow {
                operation: "swap_count",
            })?;

        if is_token_a_input {
            self.cumulative_volume_a = self
                .cumulative_volume_a
                .checked_add(u128::from(input_amount))
                .ok_or(StreamDexError::IntegerOverflow {
                    operation: "cumulative_volume_a",
                })?;
            self.cumulative_volume_b = self
                .cumulative_volume_b
                .checked_add(u128::from(output_amount))
                .ok_or(StreamDexError::IntegerOverflow {
                    operation: "cumulative_volume_b",
                })?;
        } else {
            self.cumulative_volume_b = self
                .cumulative_volume_b
                .checked_add(u128::from(input_amount))
                .ok_or(StreamDexError::IntegerOverflow {
                    operation: "cumulative_volume_b",
                })?;
            self.cumulative_volume_a = self
                .cumulative_volume_a
                .checked_add(u128::from(output_amount))
                .ok_or(StreamDexError::IntegerOverflow {
                    operation: "cumulative_volume_a",
                })?;
        }

        self.last_swap_at = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap_or_default()
            .as_secs();

        // Log successful swap
        info!(
            "Swap executed: pool={}, input={}, output={}, fee_bps={}",
            self.id, input_amount, output_amount, self.fee_bps
        );

        Ok(output_amount)
    }

    /// Adds liquidity to the pool
    ///
    /// # Arguments
    ///
    /// * `amount_a` - Amount of token A to add
    /// * `amount_b` - Amount of token B to add
    ///
    /// # Returns
    ///
    /// The amount of liquidity tokens minted
    ///
    /// # Errors
    ///
    /// Returns `StreamDexError::PoolPaused` if pool is paused
    /// Returns `StreamDexError::InvalidInput` if amounts are invalid
    /// Returns `