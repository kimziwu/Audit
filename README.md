| Risk Level | Description |
| --- | --- |
| Critical | 자산 손실, 데이터 조작으로 이어질 수 있음 |
| High | 공격을 하기에는 어렵지만, 계약에 큰 영향을 미침 ex) 중대한 함수에 접근 가능 |
| Medium | 계약에 큰 영향을 미치지는 않지만 감사 주의가 필요함 |
| Low | 오래되거나 사용되지 않는 코드가 실행에 영향을 미침 |
| Informational | 주의가 필요한 영역 |

# DEX

## 1. 유동성 검사하지 않음

### 설명

유동성 공급 시 유동성 풀 내의 토큰 비율을 검사하지 않는다.

비율에 맞지 않게 X토큰과 Y토큰을 추가할 수 있고, 이후에 removeLiquidity 함수를 통해 토큰 탈취가 가능하다.

<br>

hyeon777, addLiquidity() 

dlanaraa, addLiquidity()

jt-dream, addLiquidity()

jun4n, addLiquidity()

<br>

```solidity
function PoC() external {
	uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0);
	emit log_named_uint("firstLPReturn", firstLPReturn);
	
	uint secondLPReturn = dex.addLiquidity(1000 ether, 3 ether, 0);
	emit log_named_uint("secondLPReturn", secondLPReturn);
	
	(uint tx, uint ty) = dex.removeLiquidity(secondLPReturn, 0, 0);
	
	emit log_named_uint("secondLPremove", tx);
	emit log_named_uint("secondLPremove", ty);
	
	assertEq(tx, 1000 ether);
	assertEq(ty, 3 ether);
}
```

```solidity
[PASS] PoC() (gas: 213215)
Logs:
  receive 0
  receive 1000000000000000000000000
  true
  transfer 1000000000000000000000000
  tx,ty 500000000000000000000 500000000000000000000

Test result: ok. 1 passed; 0 failed; finished in 5.66ms
```

3 ether를 공급했지만 501.5 ether를 돌려 받는다.

공급하지 않은 토큰까지 탈취가 가능하다. 

<br>

### 파급력

Critical : removeLiquidity 함수를 통해 공급하지 않은 토큰까지 탈취가 가능하다. Dex에 공급된 자금 전부를 탈취할 수 있다.

<br>

### 해결방안

유동성 풀에 있는 토큰의 비율을 검사하는 코드를 작성한다.

```solidity
require(reserveX * tokenYAmount == reserveY * tokenXAmount);
```

<br>

<br>

## 2. 잘못된 설정 - transfer

### 설명

transfer 함수가 external로 설정되어 외부에서 접근이 가능하다.

hangi-dreamer, transfer() line 159

```solidity
function transfer(address to, uint256 lpAmount) external returns (bool) {
```

<br>

```solidity
function PoC() external {

    uint lp = dex.addLiquidity(1000 ether, 1000 ether, 0);
    vm.startPrank(address(0x01));
    {
        bool success=dex.transfer(address(0x01), lp);
        console.log(success);
        console.log("transfer",lp);

        (uint tx, uint ty) = dex.removeLiquidity(lp,0,0);
    }
    vm.stopPrank();
}
```

```solidity
[PASS] PoC() (gas: 212460)
Logs:
  receive 0
  receive 1000000000000000000000000
  true
  transfer 1000000000000000000000000
```

<br>

### 파급력

Critical : 원하는 만큼 토큰을 생성하고 removeLiquidity 함수를 통해 현금화할 수 있어서 위험하다.

<br>

### 해결방안

internal로 설정하거나 트랜잭션의 발신자를 검사한다.

```solidity
function transfer(address to, uint256 lpAmount) internal returns (bool) {
```

<br>

<br>

## 3. 나눗셈 오류 - round down

### 설명

교환하고자 하는 토큰이 1000 wei 미만의 값일 경우, 소수점 단위로 떨어져 버려진다.

hyeon777, swap() line 27,37

```solidity
if(tokenXAmount > 0){
	uint256 x_value = tokenXAmount / 1000 * 999;
	...
}
else{
	uint256 y_value = tokenYAmount / 1000 * 999;
	 ...
}

```

<br>

1000 wei 미만인 999 wei로 테스트 해보았을 때, sumY가 0으로 계산되어 swap이 작동되지 않는다.

```solidity
for (uint i=0; i<100; i++) {
    sumX += dex.swap(0, 1000 ether, 0);
    sumY += dex.swap(999 wei, 0, 0);
}
console.log("output",sumY);
```

```solidity
[FAIL. Reason: you claim too much token] testRemoveLiquidity4() (gas: 218795)
Logs:
  firstLPReturn: 1000000000000000000000000
  secondLPReturn: 2000000000000000000000000
  output 0

```

<br>

Sophie00Seo, addLiquidity() line 67

tokenXAmount와 tokenYAmount에 매우 적은 양이 추가될 때 underflow가 발생하여 0으로 설정된다.

```solidity
lpAmount = tokenXAmount * tokenYAmount / _decimal; // is amount best? no overflow?
```

<br>

```solidity
function testAddLiquidity1() external {
  uint firstLPReturn = dex.addLiquidity(0.0000000001 ether, 0.000000001 ether, 0);
  emit log_named_uint("firstLPReturn", firstLPReturn);
}
```

```solidity
[FAIL. Reason: Assertion failed.] testAddLiquidity1() (gas: 226205)
Logs:
  lpAmount 0
```

<br>

hangi-dreamer, addLiquidity() line 82,83

추가하려는 토큰의 양이 1000 wei 미만의 값일 경우, 소수점 단위로 떨어져 버려진다.

```solidity
if (lptTotalSupply < 1) {
    priceOfX = oracle.setPrice(_tokenX, tokenXAmount / decimals);
    priceOfY = oracle.setPrice(_tokenY, tokenYAmount / decimals);
}
```

<br>

```solidity
function testAddLiquidity1() external {
	uint firstLPReturn = dex.addLiquidity(999 wei, 1000 ether, 0);
	emit log_named_uint("firstLPReturn", firstLPReturn);
}
```

```solidity
Running 1 test for test/Dex.t.sol:DexTest
[FAIL. Reason: Assertion failed.] testAddLiquidity1() (gas: 219009)
Logs:
  firstLPReturn: 0
```

<br>

### 파급력

Low : 함수가 적절하게 동작하지 않아 서비스에 문제가 생기지만, 그 정도의 적은 토큰양이 공급될 일은 거의 없다.

<br>

### 해결방안

sqrt와 같은 방법을 사용해서 적은 수량의 토큰도 반영될 수 있게 해야 한다.

<br>

<br>

## 4. 구현하지 않은 함수 -1

### 설명

removeLiquidity 함수가 제대로 구현되어 있지 않다.

transferFrom 함수를 통해 토큰을 전송하는 로직이 구현되지 않아 토큰이 실제 가치로 교환되지 않는다.

hangi-dreamer, removeLiquidity()

```solidity
function PoC() external {
    dex.addLiquidity(100 ether, 100 ether, 0);
    tokenX.transfer(address(0x01), 1000 ether);
    tokenY.transfer(address(0x01), 1000 ether);
    
    vm.startPrank(address(0x01));
    {
        tokenX.approve(address(dex), 1000 ether);
        tokenY.approve(address(dex), 1000 ether);
        uint lp = dex.addLiquidity(1000 ether, 100 ether, 0);

        console.log("beofre", lp, tokenX.balanceOf(address(0x01)), tokenY.balanceOf(address(0x02)));
        dex.removeLiquidity(lp, 0, 0);
        console.log("after", lp, tokenX.balanceOf(address(0x01)), tokenY.balanceOf(address(0x02)));
    }
    vm.stopPrank();
}
```

```solidity
Logs:
  receive 0
  receive 55000000000000000000000
  beofre 55000000000000000000000 0 0
  after 55000000000000000000000 0 0
```

<br>

### 파급력

Critical : 실제 토큰으로 변환이 되지 않아 지급받지 못해 유동성 공급자들이 피해를 입게 된다. 

<br>

### 해결방안

전송 함수를 구현한다.

<br>

<br>

## 5. 검증 미흡

### 설명

dlanaraa, removeLiquidity() line 117,118

함수에서 필요한 검증들이 누락되었고, 주석처리 되어 있다.

```solidity
// require(LPTokenAmount > 0, "less LPToken");
// require(balanceOf(msg.sender) >= LPTokenAmount, "less LPToken");
```

<br>

### 파급력

low : 악의적인 사용자가 유동성 제거 과정에서 손해를 입힐 수 있는 잠재적인 취약점이 존재한다.

<br>

### 해결방안

검증과정을 구현한다.

<br>

<br>

# Lending

## 1. Arithmetic Issues (Integer overflow/underflow)

### 설명

이자 계산과 같이 큰 숫자를 포함하는 계산은 너무 크거나 작아질 때, 정수 오버플로우 또는 언더플로우가 발생할 수 있다.

<br>

siwon-huh, updateInterest(), line 80,82

일수가 커지면 숫자가 커져서 오버플로우가 발생한다.

```solidity
user_interest = user_borrowal * five_hundred_day_interest ** block_interval / (10**9) ** block_interval - user_borrowal;
usdc_total_interest = usdc_total_borrowal * five_hundred_day_interest ** block_interval / (10**9) ** block_interval - usdc_total_borrowal;

```

```
// test
vm.roll(block.number + (86400 * 10000 / 12));

// console
Failing tests:
Encountered 1 failing test in test/LendingTest.t.sol:Testx
[FAIL. Reason: Arithmetic over/underflow] testExchangeRateChangeAfterUserBorrows() (gas: 750910)

```

<br>

2-Sunghoon-Moon, _updateInterest() line 267, calculateInterest() line 292

1e18로 나눌 때, 1 ether 이하는 무시되어 계산이 다르게 나온다.

```
uint256 result = (book.usdc_deposit / 1e18 * RAY) + (prime_interest - (prime * RAY / 1e18)) * book.usdc_deposit / (USDCBalance + USDCtotalBorrow) ;   // 지분율
uint256 _borrowAmount = borrowAmount / 1e18;  // 보정
uint256 interest = mul(_borrowAmount, rpow(1001 * RAY / 1000, blockPeriodDays));

```

<br>

### 파급력

Low : 잘못된 계산 결과로 인해 다른 잔액이 저장되지만 적은 금액에서 발생하는 문제이므로 큰 파급력이 없다.

<br>

### 해결방안

siwon-huh, updateInterest(), line 80,82

SafeMath와 같은 수학 라이브러리를 사용해서 예방한다.

```
user_interest = user_borrowal.mul(five_hundred_day_interest.pow(block_interval)).div((10**9).pow(block_interval)).sub(user_borrowal);
usdc_total_interest = usdc_total_borrowal.mul(five_hundred_day_interest.pow(block_interval)).div((10**9).pow(block_interval)).sub(usdc_total_borrowal);

```

<br>

2-Sunghoon-Moon, _updateInterest() line 267, calculateInterest() line 292

곱셈을 먼저 하거나 수학 라이브러리를 사용한다.

```solidity
uint256 result = (book.usdc_deposit * RAY / 1e18) + (prime_interest - (prime * RAY / 1e18)) * book.usdc_deposit / (USDCBalance + USDCtotalBorrow) ;   // 지분율
```

<br>
<br>

## 2. **Incorrect implementation**

### 설명1

dlanaraa, liquidate(), line 91

100 ether 이하일 때의 조건이므로, 100이 아닌 100 ether로 설정해주어야 한다.

```
require(_borrow[user] < 100 || amount == (_borrow[user] * 25 / 100), "");

```

<br>

### 파급력1

Low : 조건에 맞게 설정되지 않아 조건을 만족했을 때도 실행이 되지 않는다.

<br>

### 해결방안1

코드를 올바른 조건으로 바꾸어주면 된다.

```
require(_borrow[user] < 100 ether || amount == (_borrow[user] * 25 / 100), "");

```

<br>

### 설명2

usdc 가격을 잘못 설정했다.

usdc의 가격을 잘못 설정하면 담보금에 맞는 알맞은 가치의 usdc를 빌려줄 수 없다.

예를 들어 usdc의 가격이 오르게 되었을 때, 담보물보다 더 큰 가격을 빌려주게 된다.

<br>

hangi-dreamer, borrow() line 64

```
require(oracle.getPrice(address(0)) / 10 ** 18 * etherDepositAmounts[msg.sender] * 50 / 100 - borrowAmounts[msg.sender][tokenAddress] >= amount, "Insufficient Collaterals");

```

<br>

Namryeong-Kim, update() line 138

```
vaults[msg.sender].availableBorrowETH2USDC = vaults[msg.sender].collateralETH * oracle.getPrice(address(0x0)) * LTV / (100*1e18) ;

```

<br>

koor00t, getBorrowLTV() line 119, getUserLTV() line 124

```
uint256 ethValue = (amount * dreamoracle.getPrice(address(0))) / DECIMAL;
uint256 CollateralusdcValue = (ethCollateral * dreamoracle.getPrice(address(0))) / DECIMAL;

```

<br>

### 파급력2

Medium : usdc 가격이 변할 때 계산이 맞지 않는다.

<br>

### 해결방안2

hangi-dreamer, borrow() line 64

10 ** 18 이 아닌 oracle.getPrice(tokenAddress)를 사용해 계산한다.

```
require(oracle.getPrice(address(0)) / oracle.getPrice(tokenAddress) * etherDepositAmounts[msg.sender] * 50 / 100 - borrowAmounts[msg.sender][tokenAddress] >= amount, "Insufficient Collaterals");

```

<br>

Namryeong-Kim, update() line 138

1e18이 아닌 oracle.getPrice(_tokenAddress)를 사용해 계산한다.

```
_update(_tokenAddress);
vaults[msg.sender].availableBorrowETH2USDC = vaults[msg.sender].collateralETH * oracle.getPrice(address(0x0)) * LTV / (100*oracle.getPrice(_tokenAddress));

```

<br>

koor00t, getBorrowLTV() line 119, getUserLTV() line 124
DECIMAL이 아닌 dreamoracle.getPrice(address(usdc))로 바꾸어준다.

```
uint256 ethValue = (amount * dreamoracle.getPrice(address(0))) / dreamoracle.getPrice(address(usdc));
uint256 CollateralusdcValue = (ethCollateral * dreamoracle.getPrice(address(0))) / dreamoracle.getPrice(address(usdc));

```

<br>
<br>

## 3. Reentrancy

### 설명

예치한 담보금액을 보낼 때, 담보금액을 업데이트 하기전에 함수를 실행하여 금액을 받고 다시 호출하는 재진입 공격이 일어날 수 있다.

<br>

2-Sunghoon-Moon, withdraw()

ETHBalance 변수와 participateBooks 변수를 이용해 담보금액을 저장하고 관리한다.

이 변수들이 전송 함수 이후에 위치하고 있어 외부함수 호출로 인한 재진입 공격이 일어날 수 있다.

```solidity
require(participateBooks[msg.sender].eth_balance - borrowedETH >= amount);

(bool success, ) = payable(msg.sender).call{value: amount}("");

ETHBalance -= amount;

participateBooks[msg.sender].eth_balance -= amount;
participateBooks[msg.sender].eth_deposit -= amount;
participateBooks[msg.sender].eth_collateral -= amount;

```

<br>

Namryeong-Kim, withdraw()

vaults 변수와 tempValue 변수를 이용해 담보금액을 저장하고 관리한다.

이 변수들이 전송 함수 이후에 위치하고 있어 외부함수 호출로 인한 재진입 공격이 일어날 수 있다.

```solidity
(bool success, ) = payable(msg.sender).call{value: _amount}("");
require(success, "ERROR");

vaults[msg.sender] = tempVault;
```

<br>

### 파급력

Critical : 악의적인 사용자가 담보금을 연속적으로 탈취할 수 있다.

<br>

### 해결방안

call 함수를 담보금액 업데이트 이후 실행하도록 설정한다.
