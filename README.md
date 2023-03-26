| Risk Level | Description |
| --- | --- |
| Critical | 자산 손실, 데이터 조작으로 이어질 수 있음 |
| High | 공격을 하기에는 어렵지만, 계약에 큰 영향을 미침 ex) 중대한 함수에 접근 가능 |
| Medium | 계약에 큰 영향을 미치지는 않지만 감사 주의가 필요함 |
| Low | 오래되거나 사용되지 않는 코드가 실행에 영향을 미침 |
| Informational | 주의가 필요한 영역 |

# DEX

## **Insufficient inspection**

### 설명

dlanaraa, addLiquidity() line 90

hyeon777, addLiquidity() line 47-64

jun4n, addLiquidity() line 66

비율에 맞지 않게 X토큰과 Y토큰을 추가하여 removeLiquidity 함수를 통해 토큰 탈취가 가능하다.

<br>

### 파급력

dlanaraa, addLiquidity() line 90

hyeon777, addLiquidity() line 47-64

jun4n, addLiquidity() line 66

High : 토큰 탈취가 가능하다.

<br>

### 해결방안

dlanaraa, addLiquidity() line 90

hyeon777, addLiquidity() line 47-64

jun4n, addLiquidity() line 66

풀 비율을 체크하여 토큰을 추가하는 계산이 필요하다.

<br>

<br>

## 1000 wei 미만 underflow

### 설명

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

swap 함수의 input 수량을 토큰으로 변경할 때, 1000 미만의 값은 버려지게 된다.

<br>

교환하고자 하는 토큰이 1000 wei 아래의 값일 경우, 소수점 단위로 떨어져 버려진다.

스왑을 하려고 할 때, 교환이 원활하게 이루어질 수 없어 서비스에 문제가 생긴다.

 <br>

1000 wei 미만인 999 wei로 테스트 해보았을 때, input 수량이 0으로 계산되어 최소 교환량에 미치지 못하고, 교환이 불가하다. 

```solidity
// Dex.t.sol
for (uint i=0; i<100; i++) {
    sumX += dex.swap(0, 1000 ether, 0);
    sumY += dex.swap(999 wei, 0, 0);
}
```

```solidity
// Dex.sol
if(tokenXAmount > 0){
    uint256 x_value = tokenXAmount / 1000 * 999;
    console.log("x_value",x_value);
    amountY = k / (amountX + x_value);
    outputAmount = reserveY - amountY;
    console.log("output",outputAmount);
    require(outputAmount < amountY, "amountY is less than outputAmount");
    require(tokenMinimumOutputAmount < outputAmount, "you claim too much token");
    X.transferFrom(msg.sender, address(this), tokenXAmount);
    Y.transfer(msg.sender, outputAmount);
}
```

```solidity
// console
[FAIL. Reason: you claim too much token] testRemoveLiquidity4() (gas: 218795)
Logs:
  firstLPReturn: 1000000000000000000000000
  secondLPReturn: 2000000000000000000000000
  x_value 0
  output 0
```

<br>

### 파급력

Low : 적은 토큰의 개수인 상황에서 swap은 잘 이루어지지 않는다.

<br>

### 해결방안

수수료 계산을 할 때, y=((x*999/1000)*Y)/(X+(x*999/1000)) 와 같이 계산 식을 재배치 하면 된다.

<br>

<br>

## Unimplemented function

### 설명1

transfer 함수를 구현하지 않아 LP 토큰을 전송하지 못한다.

siwon-huh, transfer() line 106

dlanaraa, transfer() line 138

jw-dream, transfer() line 86

seonghwi-leem transfer() line 106

<br>

### 파급력1

Informational : LP 토큰을 받아도 활용할 수 없다.

<br>

### 해결방안1

transfer 함수를 구현해야 한다.

<br>

### 설명2

jt-dream, remove()

burn 함수가 구현되어 있지 않아 유동성 풀에서 LP 토큰을 제거할 함수 구현이 필요하다.

<br>

### 파급력2

Informational : LP 토큰을 제거할 수 없다.

<br>

### 해결방안2

_burn 함수를 구현해야 한다.

<br>

<br>

## 유동성 검사X

### 설명

공통, addLiquidity() 

유동성 공급 시 유동성 풀 내의 토큰 비율을 검사하지 않아 imbalance 유동성 공급이 가능하다.

```solidity
uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0);
emit log_named_uint("firstLPReturn", firstLPReturn);

uint secondLPReturn = dex.addLiquidity(1000 ether, 5 ether, 0);
emit log_named_uint("secondLPReturn", secondLPReturn);

(uint tx, uint ty)=dex.removeLiquidity(secondLPReturn,0,0);
emit log_named_uint("secondLPremove", tx);
emit log_named_uint("secondLPremove", ty);

assertEq(tx, 1000 ether);
assertEq(ty, 5 ether);
```

<br>

### 파급력

Critical
유동성 공급 이후 removeLiquidity 함수를 실행하면 공급하지 않은 토큰까지 탈취가 가능하다.

TokenY는 5 ether만 공급하지만, 회수하는 양은 그렇지 않다. 

Dex에 공급된 자산 전부를 탈취할 수 있다. 

```solidity
[FAIL. Reason: Assertion failed.] testAddLiquidity1() (gas: 230487)
Logs:
  firstLPReturn: 1000000000000000000000000
  secondLPReturn: 1000000000000000000000000
  secondLPremove: 1000000000000000000000
  secondLPremove: 502500000000000000000
  Error: a == b not satisfied [uint]
        Left: 502500000000000000000
       Right: 5000000000000000000
```

<br>

### 해결방안

유동성 풀에 있는 토큰의 비율을 검사하는 코드를 작성한다.

```solidity
require(reserveX * tokenYAmount == reserveY * tokenXAmount);
```

<br>

<br>

## RemoveLiquidity


### 설명

hyeon777, removeLiquidity() line 75-

유동성 제거 과정에서 공급량보다 더 많은 토큰의 양이 제거될 수 있다.

user2 계정은 1000 ether를 추가했지만, 1500 ether를 제거할 수 있다.

```solidity
// Dex.t.sol
	function testRemoveLiquidity1() external {
	
	vm.startPrank(user1);
	{
	    uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0);
	    emit log_named_uint("firstLPReturn", firstLPReturn);
	}
	vm.stopPrank();
	vm.startPrank(user2);
	{
	    uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0);
	    emit log_named_uint("firstLPReturn", firstLPReturn);
	
	    (uint tx, uint ty)=dex.removeLiquidity(1500 ether,0,0);
	}
}
```

토큰이 정상적으로 전송되는 것을 확인할 수 있다.

```solidity
// Dex.sol
bool successX=X.transfer(msg.sender, _tx);
bool successY=Y.transfer(msg.sender, _ty);
console.log(successX,successY);

_burn(msg.sender, LPTokenAmount);
```

<br>

### 파급력

Critical

토큰을 업데이트 하기 전에 transfer 함수를 실행하여 악의적인 사용자는 재진입이 가능하다.

<br>

### 해결방안

잔액을 확인하는 require 함수가 필요하다.

LP 토큰의 잔액을 업데이트 한 후에 토큰 전송을 해야 한다.

```solidity
require(X.balanceOf(msg.sender)>=_tx);
require(Y.balanceOf(msg.sender)>=_ty);

_burn(msg.sender, LPTokenAmount)
X.transfer(msg.sender, _tx);
Y.transfer(msg.sender, _ty);
```
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

```solidity
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

```solidity
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

```solidity
user_interest = user_borrowal.mul(five_hundred_day_interest.pow(block_interval)).div((10**9).pow(block_interval)).sub(user_borrowal);
usdc_total_interest = usdc_total_borrowal.mul(five_hundred_day_interest.pow(block_interval)).div((10**9).pow(block_interval)).sub(usdc_total_borrowal);
```

<br>

2-Sunghoon-Moon, _updateInterest() line 267, calculateInterest() line 292

곱셈을 먼저 하거나 수학 라이브러리를 사용한다.

<br>
<br>

## 2. **Incorrect implementation**

### 설명1

dlanaraa, liquidate(), line 91

100 ether 이하일 때의 조건이므로, 100이 아닌 100 ether로 설정해주어야 한다.

```jsx
require(_borrow[user] < 100 || amount == (_borrow[user] * 25 / 100), "");
```

<br>

### 파급력1

Low : 조건에 맞게 설정되지 않아 조건을 만족했을 때도 실행이 되지 않는다.

<br>

### 해결방안1

코드를 올바른 조건으로 바꾸어주면 된다.

```jsx
require(_borrow[user] < 100 ether || amount == (_borrow[user] * 25 / 100), "");
```

<br>

### 설명2

usdc 가격을 잘못 설정했다.

usdc의 가격을 잘못 설정하면 담보금에 맞는 알맞은 가치의 usdc를 빌려줄 수 없다.

예를 들어 usdc의 가격이 오르게 되었을 때, 담보물보다 더 큰 가격을 빌려주게 된다.

<br>


hangi-dreamer, borrow() line 64

```jsx
require(oracle.getPrice(address(0)) / 10 ** 18 * etherDepositAmounts[msg.sender] * 50 / 100 - borrowAmounts[msg.sender][tokenAddress] >= amount, "Insufficient Collaterals");
```

<br>

Namryeong-Kim, update() line 138

```solidity
vaults[msg.sender].availableBorrowETH2USDC = vaults[msg.sender].collateralETH * oracle.getPrice(address(0x0)) * LTV / (100*1e18) ;
```

<br>

koor00t, getBorrowLTV() line 119, getUserLTV() line 124

```solidity
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

```jsx
require(oracle.getPrice(address(0)) / oracle.getPrice(tokenAddress) * etherDepositAmounts[msg.sender] * 50 / 100 - borrowAmounts[msg.sender][tokenAddress] >= amount, "Insufficient Collaterals");
```

<br>

Namryeong-Kim, update() line 138

1e18이 아닌 oracle.getPrice(_tokenAddress)를 사용해 계산한다.

```solidity
_update(_tokenAddress);
vaults[msg.sender].availableBorrowETH2USDC = vaults[msg.sender].collateralETH * oracle.getPrice(address(0x0)) * LTV / (100*oracle.getPrice(_tokenAddress));
```

<br>

koor00t, getBorrowLTV() line 119, getUserLTV() line 124
DECIMAL이 아닌 dreamoracle.getPrice(address(usdc))로 바꾸어준다.

```solidity
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
