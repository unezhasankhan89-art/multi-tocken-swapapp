// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IERC20 {
    function transfer(address recipient, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
}

contract Project {
    address public owner;
    uint256 public feePercent = 3; // 0.3% fee (3/1000)
    
    struct LiquidityPool {
        uint256 tokenAReserve;
        uint256 tokenBReserve;
        bool exists;
    }
    
    // Mapping: tokenA => tokenB => LiquidityPool
    mapping(address => mapping(address => LiquidityPool)) public liquidityPools;
    
    // Track liquidity providers
    mapping(address => mapping(address => mapping(address => uint256))) public liquidityShares;
    
    event LiquidityAdded(address indexed tokenA, address indexed tokenB, uint256 amountA, uint256 amountB, address indexed provider);
    event TokensSwapped(address indexed tokenIn, address indexed tokenOut, uint256 amountIn, uint256 amountOut, address indexed trader);
    event LiquidityRemoved(address indexed tokenA, address indexed tokenB, uint256 amountA, uint256 amountB, address indexed provider);
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this");
        _;
    }
    
    /**
     * @dev Add liquidity to a token pair pool
     * @param tokenA Address of first token
     * @param tokenB Address of second token
     * @param amountA Amount of tokenA to add
     * @param amountB Amount of tokenB to add
     */
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint256 amountA,
        uint256 amountB
    ) external returns (bool) {
        require(tokenA != tokenB, "Tokens must be different");
        require(amountA > 0 && amountB > 0, "Amounts must be greater than 0");
        
        // Transfer tokens from user to contract
        require(
            IERC20(tokenA).transferFrom(msg.sender, address(this), amountA),
            "TokenA transfer failed"
        );
        require(
            IERC20(tokenB).transferFrom(msg.sender, address(this), amountB),
            "TokenB transfer failed"
        );
        
        // Update liquidity pool
        LiquidityPool storage pool = liquidityPools[tokenA][tokenB];
        pool.tokenAReserve += amountA;
        pool.tokenBReserve += amountB;
        pool.exists = true;
        
        // Track liquidity shares for provider
        liquidityShares[tokenA][tokenB][msg.sender] += amountA;
        
        emit LiquidityAdded(tokenA, tokenB, amountA, amountB, msg.sender);
        return true;
    }
    
    /**
     * @dev Swap tokens using constant product formula (x * y = k)
     * @param tokenIn Address of input token
     * @param tokenOut Address of output token
     * @param amountIn Amount of input tokens
     * @param minAmountOut Minimum acceptable output amount (slippage protection)
     */
    function swap(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 minAmountOut
    ) external returns (uint256 amountOut) {
        require(tokenIn != tokenOut, "Tokens must be different");
        require(amountIn > 0, "Amount must be greater than 0");
        
        LiquidityPool storage pool = liquidityPools[tokenIn][tokenOut];
        require(pool.exists, "Liquidity pool does not exist");
        
        // Calculate output amount using constant product formula
        // amountOut = (amountIn * reserveOut) / (reserveIn + amountIn)
        // Apply fee: amountIn = amountIn * (1000 - fee) / 1000
        uint256 amountInWithFee = (amountIn * (1000 - feePercent)) / 1000;
        amountOut = (amountInWithFee * pool.tokenBReserve) / (pool.tokenAReserve + amountInWithFee);
        
        require(amountOut >= minAmountOut, "Slippage tolerance exceeded");
        require(amountOut <= pool.tokenBReserve, "Insufficient liquidity");
        
        // Transfer input tokens from user
        require(
            IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn),
            "Input token transfer failed"
        );
        
        // Transfer output tokens to user
        require(
            IERC20(tokenOut).transfer(msg.sender, amountOut),
            "Output token transfer failed"
        );
        
        // Update reserves
        pool.tokenAReserve += amountIn;
        pool.tokenBReserve -= amountOut;
        
        emit TokensSwapped(tokenIn, tokenOut, amountIn, amountOut, msg.sender);
        return amountOut;
    }
    
    /**
     * @dev Get the exchange rate and expected output for a swap
     * @param tokenIn Address of input token
     * @param tokenOut Address of output token
     * @param amountIn Amount of input tokens
     * @return amountOut Expected output amount
     */
    function getExchangeRate(
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) external view returns (uint256 amountOut) {
        require(tokenIn != tokenOut, "Tokens must be different");
        require(amountIn > 0, "Amount must be greater than 0");
        
        LiquidityPool storage pool = liquidityPools[tokenIn][tokenOut];
        require(pool.exists, "Liquidity pool does not exist");
        
        uint256 amountInWithFee = (amountIn * (1000 - feePercent)) / 1000;
        amountOut = (amountInWithFee * pool.tokenBReserve) / (pool.tokenAReserve + amountInWithFee);
        
        return amountOut;
    }
    
    /**
     * @dev Get pool reserves for a token pair
     */
    function getPoolReserves(address tokenA, address tokenB) 
        external 
        view 
        returns (uint256 reserveA, uint256 reserveB, bool exists) 
    {
        LiquidityPool storage pool = liquidityPools[tokenA][tokenB];
        return (pool.tokenAReserve, pool.tokenBReserve, pool.exists);
    }
    
    /**
     * @dev Update fee percentage (only owner)
     */
    function updateFee(uint256 newFeePercent) external onlyOwner {
        require(newFeePercent <= 10, "Fee too high"); // Max 1%
        feePercent = newFeePercent;
    }
}
