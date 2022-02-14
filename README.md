# Code
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "./data/Tax.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";
import "./interfaces/IHODLRewardDistributor.sol";
import "./interfaces/IFactory.sol";
import "./interfaces/IRouter.sol";
import "./SwapHandler.sol";

contract BabyLilFloki is ERC20, Ownable {
    using SafeMath for uint256;

    struct Whitlisted {
        bool maxTx;
        bool maxBalance;
        bool tax;
    }

    function decimals() public view virtual override returns (uint8) {
        return 18;
    }
    uint256 constant BASE = 10 ** 18; // 18 decimals
    uint256 constant TOTAL_SUPPLY = 1_000_000_000_000 * BASE;



    address public wbnb;
    address public swapRouter;
    address public wbnbPair;


    address public autoLPWallet;
    address public marketingWallet;

    uint256 public minimumShareForRewards;
    bool public autoBatchProcess;

    Tax public buyerTax = Tax(
        2, // AUTOLP
        0, // HOLDER
        3 // Marketing
    );

    Tax public sellerTax = Tax(
        2, // AUTOLP
        10, // HOLDER
        3 // Marketing
    );

    Tax public transferTax = Tax(
        0, // AUTOLP
        0, // HOLDER 
        0 // Marketing
    );

    mapping(address => bool) public isLpPair;

    mapping(address => Whitlisted) public whitlisted;

    IHODLRewardDistributor public hodlRewardDistributor;

    bool public isDistributorSet;

    bool public reflectionEnabled = false;

    SwapHandler public swapHundler;

    uint256 public autoLPReserved;
    uint256 public hodlReserved;
    uint256 public marketingReserved;

    uint256 public processingGasLimit = 500000;

    constructor(
        string memory name_,
        string memory symbol_,
        SwapHandler swapHandler_,
        address wbnb_,
        address swapRouter_,
        address payable autoLP_,
        address payable marketing_
    ) ERC20(name_, symbol_) {
        // init wallets addresses
        wbnb = wbnb_;
        swapRouter = swapRouter_;
        autoLPWallet = autoLP_;
        marketingWallet = marketing_;

        // create pair for OPSY/
        wbnbPair = IFactory(
            IRouter(swapRouter_).factory()
        ).createPair(wbnb_, address(this));

        isLpPair[wbnbPair] = true;

        swapHundler = swapHandler_;

        // whiteliste wallets
        whitlisted[autoLP_] = Whitlisted(
            true, // max transfer
            true, // max balance
            true  // Tax
        );

        whitlisted[marketing_] = Whitlisted(
            true, // max transfer
            true, // max balance
            true  // Tax
        );

        whitlisted[address(this)] = Whitlisted(
            true, // max transfer
            true, // max balance
            true  // Tax
        );

        whitlisted[address(swapHundler)] = Whitlisted(
            true, // max transfer
            true, // max balance
            true  // Tax
        );

        whitlisted[swapRouter_] = Whitlisted(
            true, // max transfer
            true, // max balance
            false  // Tax
        );
        // mint supply to wallet
        _mint(autoLP_, TOTAL_SUPPLY);
    }

    function initDistributor(
        address distributor_
    ) external onlyOwner {
        hodlRewardDistributor = IHODLRewardDistributor(distributor_);

        require(hodlRewardDistributor.owner() == address(this), "initDistributor: Erc20 not owner");

        hodlRewardDistributor.excludeFromRewards(wbnbPair);
        hodlRewardDistributor.excludeFromRewards(swapRouter);
        hodlRewardDistributor.excludeFromRewards(autoLPWallet);
        hodlRewardDistributor.excludeFromRewards(marketingWallet);
        hodlRewardDistributor.excludeFromRewards(address(this));
        hodlRewardDistributor.excludeFromRewards(address(swapHundler));

        whitlisted[distributor_] = Whitlisted(
            true,
            true,
            true
        );

        isDistributorSet = true;
    }

    function transfer(
        address to_,
        uint256 amount_
    ) public virtual override returns (bool) {
        return _customTransfer(_msgSender(), to_, amount_);
    }

    function transferFrom(
        address from_,
        address to_,
        uint256 amount_
    ) public virtual override returns (bool) {
        // check allowance
        require(allowance(from_, _msgSender()) >= amount_, "> allowance");
        bool success = _customTransfer(from_, to_, amount_);
        approve(from_, allowance(from_, _msgSender()).sub(amount_));
        return success;
    }


    /**
        When taxes are generated from swaps 
        we cannot make the swap to avax due to reentrency gard
        on LPpool , so unstead we add it to a reserve , on next transfer
        this function is called and can also be called by any user
        if they are willing to pay gas.
    */
    function processReserves() public {
        swapHundler.swapToNativeWrappedToken(
            autoLPReserved,
            hodlReserved,
            marketingReserved
        );

        autoLPReserved = 0;
        hodlReserved = 0;
        marketingReserved = 0;
    }

    function setAutoLPWallet(
        address newautoLPWallet_
    ) external onlyOwner {
        require(
            newautoLPWallet_ != autoLPWallet,
            "ReflectionERC20: same as current wallet"
        );
        require(
            newautoLPWallet_ != address(0),
            "ReflectionERC20: cannot be address(0)"
        );
        autoLPWallet = newautoLPWallet_;
    }

    function setMarketingWallet(
        address newMarketingWallet_
    ) external onlyOwner {
        require(
            newMarketingWallet_ != marketingWallet,
            "ReflectionERC20: same as current wallet"
        );
        require(
            newMarketingWallet_ != address(0),
            "ReflectionERC20: cannot be address(0)"
        );
        marketingWallet = newMarketingWallet_;
    }

    /**
        Sets the whitlisting of a wallet 
        you can set it's whitlisting from maxTransfer #fromMaxTx
        or from payign tax #fromTax separatly
    */
    function whitelist(
        address wallet_,
        bool fromMaxTx_,
        bool fromMaxBalance_,
        bool fromTax_
    ) external onlyOwner {
        whitlisted[wallet_] = Whitlisted(
            fromMaxTx_,
            fromMaxBalance_,
            fromTax_
        );
    }

    /**
        this wallet will be excluded from rewards 
        it is had any amount of rewards they will be
        distributed to all share holders
    */
    function excludeFromHodlRewards(
        address wallet_
    ) external onlyOwner {
        if(autoLPReserved + hodlReserved + marketingReserved > 0)
            processReserves();
        hodlRewardDistributor.excludeFromRewards(wallet_);
    }

    /**
        This wallet will be included in rewards
    */
    function includeFromHodlRewards(
        address wallet_
    ) external onlyOwner {
        if(autoLPReserved + hodlReserved + marketingReserved > 0)
            processReserves();
        hodlRewardDistributor.includeInRewards(wallet_);
    }

    function setBuyerTax(
        uint256 autoLP_,
        uint256 holder_,
        uint256 marketing_
    ) external onlyOwner {
        transferTax = Tax(
            autoLP_, holder_, marketing_
        );
    }

    function setSellerTax(
        uint256 autoLP_,
        uint256 holder_,
        uint256 marketing_
    ) external onlyOwner {
        transferTax = Tax(
            autoLP_, holder_, marketing_
        );
    }

    function setTransferTax(
        uint256 autoLP_,
        uint256 holder_,
        uint256 marketing_
    ) external onlyOwner {
        transferTax = Tax(
            autoLP_, holder_, marketing_
        );
    }

    function setReflection(
        bool isEnabled_
    ) external onlyOwner {
        require(isDistributorSet, "Distributor_not_set");
        if(autoLPReserved + hodlReserved + marketingReserved > 0)
            processReserves();
        reflectionEnabled = isEnabled_;
    }

    function setIsLPPair(
        address pairAddess_,
        bool isPair_
    ) external onlyOwner {
        isLpPair[pairAddess_] = isPair_;
    }

    function setPeocessingGasLimit(
        uint256 maxAmount_
    ) external onlyOwner {
        processingGasLimit = maxAmount_;
    }
    /**
        prevents accidental renouncement of owner ship 
        can sill renounce if set explicitly to dead address
     */
    function renounceOwnership() public virtual override onlyOwner {}

    /**
        sets the minimum balance required to make holder eligible to reseave reflection rewards
     */
    function setMinimumShareForRewards(uint256 minimumAmount_) external onlyOwner {
        minimumShareForRewards = minimumAmount_;
    }

    /**
        Token uses some of the transaction gas to distribute rewards 
        you can enable disable/enable here 
        users can still claim
     */
    function setAutoBatchProcess(bool autoBatchProcess_) external onlyOwner {
        autoBatchProcess = autoBatchProcess_;
    }

    function claimRewardsFor(address wallet_) external {
        // No danger here claim sends to the share holder
        hodlRewardDistributor.claimPending(wallet_);
    }

    /**
        this is the implementation the custom transfer for this token
     */
    function _customTransfer(
        address from_,
         address to_,
          uint256 amount_
    ) internal returns (bool) {
        // if whitlisted or we are internally swapping no tax
        if (whitlisted[from_].tax || whitlisted[to_].tax) {
            _transfer(from_, to_, amount_);
        } else {
            uint256 netTransfer = amount_;

            if (reflectionEnabled) {
                Tax memory currentAppliedTax = isLpPair[from_] ? buyerTax : isLpPair[to_] ? sellerTax : transferTax;
                uint256 prevTotal = autoLPReserved + hodlReserved + marketingReserved;
                autoLPReserved += amount_.mul(currentAppliedTax.autoLP).div(100);
                hodlReserved += amount_.mul(currentAppliedTax.holder).div(100);
                marketingReserved += amount_.mul(currentAppliedTax.marketing).div(100);
                uint256 totalTax = autoLPReserved + hodlReserved + marketingReserved;
                uint256 currentTax = totalTax.sub(prevTotal);
                netTransfer = amount_.sub(currentTax);

                if(currentTax > 0)
                    _transfer(from_, address(swapHundler), currentTax);

                // if we have tokens and we are not in swap => swap and distribute to wallets
                if (totalTax > 0 && from_ != wbnbPair && to_ != wbnbPair)
                    processReserves();                
            }
            // transfer 
            _transfer(from_, to_, netTransfer);
            // This will trigger after_transfer and will update shares for from_ and to_ is needed
        }
        return true;
    }

    function _massProcess() internal {
        if(autoBatchProcess)
            hodlRewardDistributor.batchProcessClaims(
                gasleft() > processingGasLimit ? processingGasLimit : gasleft().mul(80).div(100)
            );
    }

    function _afterTokenTransfer(
        address from_,
        address to_,
        uint256 amount_
    ) internal override {
        super._afterTokenTransfer(from_,to_,amount_);
        if (isDistributorSet) {
            _updateShare(from_);
            _updateShare(to_);
            _massProcess();
        }
    }

    function _updateShare(
        address wallet
    ) internal {
        if (!hodlRewardDistributor.excludedFromRewards(wallet))
            hodlRewardDistributor.setShare(wallet, balanceOf(wallet) > minimumShareForRewards ? balanceOf(wallet) : 0);
    }
}
