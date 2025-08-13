// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@aave/core-v3/contracts/flashloan/base/FlashLoanSimpleReceiverBase.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract SimpleFlashLoan is FlashLoanSimpleReceiverBase {
    address payable public owner;

    event FlashLoanRequested(address asset, uint256 amount);
    event FlashLoanReceived(address asset, uint256 amount, uint256 premium);
    event ContractBalance(uint256 balance);
    event FlashLoanRepaid(address asset, uint256 amount, uint256 premium);

    error OnlyOwner();
    error InsufficientBalance();

    modifier onlyOwner() {
        if (msg.sender != owner) revert OnlyOwner();
        _;
    }

    constructor(IPoolAddressesProvider provider) FlashLoanSimpleReceiverBase(provider) {
        owner = payable(msg.sender);
    }

    /// @notice Request a flash loan using the token contract address and amount
    function requestFlashloan(address tokenContract, uint256 amount) external onlyOwner {
        require(tokenContract != address(0), "Invalid token address");
        require(amount > 0, "Amount must be greater than zero");

        emit FlashLoanRequested(tokenContract, amount);

        // Initiate the flash loan from Aave
        POOL.flashLoanSimple(
            address(this),
            tokenContract,
            amount,
            "",
            0
        );
    }

    /// @notice Called by Aave after transferring the flash loaned amount
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata
    ) external override returns (bool) {
        require(msg.sender == address(POOL), "Caller is not pool");
        require(initiator == address(this), "Initiator must be this contract");

        emit FlashLoanReceived(asset, amount, premium);

        uint256 bal = IERC20(asset).balanceOf(address(this));
        emit ContractBalance(bal);

        uint256 repayAmount = amount + premium;
        if (bal < repayAmount) revert InsufficientBalance();

        // Approve the Aave pool to pull the owed amount
        IERC20(asset).approve(address(POOL), repayAmount);

        emit FlashLoanRepaid(asset, amount, premium);
        return true;
    }

    /// @notice Withdraw specific ERC20 tokens from contract to owner
    function withdrawTokenAmount(address token, uint256 amount) external onlyOwner {
        uint256 bal = IERC20(token).balanceOf(address(this));
        if (bal < amount) revert InsufficientBalance();
        bool success = IERC20(token).transfer(owner, amount);
        require(success, "Token transfer failed");
    }

    /// @notice Withdraw all tokens of a type from contract to owner
    function withdrawAllTokens(address token) external onlyOwner {
        uint256 bal = IERC20(token).balanceOf(address(this));
        require(bal > 0, "No tokens to withdraw");
        bool success = IERC20(token).transfer(owner, bal);
        require(success, "Token transfer failed");
    }

    /// @notice Withdraw all ETH/Native tokens from contract to owner
    function withdrawAllETH() external onlyOwner {
        uint256 bal = address(this).balance;
        require(bal > 0, "No ETH to withdraw");
        (bool sent, ) = owner.call{value: bal}("");
        require(sent, "ETH transfer failed");
    }

    /// @notice Get contract ETH balance
    function getBalance() external view returns (uint256) {
        return address(this).balance;
    }

    /// @notice Get contract ERC20 token balance
    function getTokenBalance(address token) external view returns (uint256) {
        return IERC20(token).balanceOf(address(this));
    }

    // Receive ETH fallback
    receive() external payable {
        emit ContractBalance(address(this).balance);
    }
}
