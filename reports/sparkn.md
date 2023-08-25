<table>
  <tr>
    <td><img src="https://res.cloudinary.com/droqoz7lg/image/upload/q_90/dpr_0.800000011920929/c_fill,g_auto,h_320,w_320/f_auto/v1/company/mdsu3k5i2qjdx1sk1pav?_a=BATCr5AA0" height="250" width="250" /></td>
    <td>
      <h1>Sparkn Protocol</h1>
      <h2>CodeHawks</h2>
      <p>Prepared by: Rex Joseph</p>
      <p>Date: August 20 to 28, 2023</p>
    </td>
  </tr>
</table>

# About **Sparkn**

The SPARKN protocol is a Web3 project that aims to build a marketplace for anyone who wants to solve their problems or anyone who wants to help solve problems. As a first step, we have created the protocol. The details of how to use the protocol is up to the users.

# Summary & Scope

The [Cyfrin/2023-08-sparkn](https://github.com/Cyfrin/2023-08-sparkn) repository was audited.

The following contracts were in scope:
- ProxyFactory.sol
- Distributor.sol
- Proxy

# Summary of Findings

| ID     | Title                        | Severity      | Fixed |
| ------ | ---------------------------- | ------------- | ----- |
| [H-01] | Duplicate winner addresses will be funded | High |   |
| [H-02] | Funds can be lost to a non-deployed proxy & non-valid proxy for a contest | High |   |
| [M-01] | Inability for rewards to be distributed to a winner if their address is blacklisted by one of the ERC20 StableCoin or tokens | Medium |   |

# Detailed Findings

## [H-01] Duplicate winner addresses will be funded

The `_distribute` function in Distributor.sol will fund all winners including duplicate addresses.

## Vulnerability Detail

Duplicate winner addresses will be funded

## Impact

With the issue of financial loses lingering for winners supporting the solution or pioneering one in a challenge/contest, some addresses may get rewarded wrongly, and others will get nothing. The phrase is true: It's easy for the bank to take money out of your account but hard for them to put it back.

## Code Snippet

```solidity
    function _distribute(address token, address[] memory winners, uint256[] memory percentages, bytes memory data)
        internal
    {
        // token address input check
        if (token == address(0)) revert Distributor__NoZeroAddress();
        if (!_isWhiteListed(token)) {
            revert Distributor__InvalidTokenAddress();
        }
        // winners and percentages input check
        if (winners.length == 0 || winners.length != percentages.length) revert Distributor__MismatchedArrays();
        uint256 percentagesLength = percentages.length;
        uint256 totalPercentage;
        for (uint256 i; i < percentagesLength;) {
            totalPercentage += percentages[i];
            unchecked {
                ++i;
            }
        }
        // check if totalPercentage is correct
        if (totalPercentage != (10000 - COMMISSION_FEE)) {
            revert Distributor__MismatchedPercentages();
        }
        IERC20 erc20 = IERC20(token);
        uint256 totalAmount = erc20.balanceOf(address(this));

        // if there is no token to distribute, then revert
        if (totalAmount == 0) revert Distributor__NoTokenToDistribute();
        // @audit handle winner address uniqueness
        // @audit handle case for user's address blacklisted by reward token
        uint256 winnersLength = winners.length; // cache length
        for (uint256 i; i < winnersLength;) {
            uint256 amount = totalAmount * percentages[i] / BASIS_POINTS;
            erc20.safeTransfer(winners[i], amount);
            unchecked {
                ++i;
            }
        }

        // send commission fee as well as all the remaining tokens to STADIUM_ADDRESS to avoid dust remaining
        _commissionTransfer(erc20);
        emit Distributed(token, winners, percentages, data);
    }
```

## Tool Used

Manual Review / VSCode

## Recommendation

Have a logic to only fund unique addresses and filter out duplicates. You can achieve this in a couple of ways but here's one:
1 - have a: 

```solidity
  mapping(address => bool) private seenAddresses;
```
2 - make another for loop to check for duplicate addresses and populate seenAddresses mapping

3 - check if address has been seen before e.g.:

4 - 
```solidity
  if (seenAddresses[winner]) {
    // revert or skip or do something else;
  }
  seenAddresses[winner] = true;
```
5. then proceed to fund the unique winners addresses

## [H-02] Funds can be lost to a non-deployed proxy & non-valid proxy for a contest

Not validating the existence of a valid proxy in the `distributeByOwner` function can result in attempting to distribute and in fact, distributing to a non-deployed proxy or a proxy contract linked to another contest; thereby resulting in loss of funds for the contest

## Vulnerability Detail

This vulnerability can be seen in the distributeByOwner (L205-219) function at the `ProxyFactory.sol` contract which lacks checks for the proxy address being passed in the params to be validated as deployed and/or not belonging to another open contest on the platform

## Impact

The caller of this function, including a bad actor can exploit this vulnerability to distribute rewards to a completely different contest or organizer in the case the owner has been compromised. A typical scenario would be setting up a challenge/competetition not intended to be fulfilled > getting sponsored > exploiting supporter's work > ultimately compromising owner and providing a proxy address to another contest (used specifically for dispersing funds to a bunch of random supporters non-existent) This will not only present the protocol in bad faith but ultimately lose sponsor's funds and supporters support.

## Code Snippet

```solidity
    function distributeByOwner(
        address proxy,
        address organizer,
        bytes32 contestId,
        address implementation,
        bytes calldata data
    ) public onlyOwner {
        if (proxy == address(0)) revert ProxyFactory__ProxyAddressCannotBeZero();
        bytes32 salt = _calculateSalt(organizer, contestId, implementation);
        if (saltToCloseTime[salt] == 0) revert ProxyFactory__ContestIsNotRegistered();
        // distribute only when it exists and expired
        if (saltToCloseTime[salt] + EXPIRATION_TIME > block.timestamp) revert ProxyFactory__ContestIsNotExpired();
        // @audit more check for proxy existence before calling _distribute
        _distribute(proxy, data);
    }
```

## Tool Used

Manual Review / VSCode

## Recommendation

1. getProxyAddress before attempting to distribute
2. Employ a hardened check utilizing salt, owner, contest ID and implementation comparison
3. Verify proxy is infact existent/deployed before attempting to distribute

## [M-01] Inability for rewards to be distributed to a winner if their address is blacklisted by one of the ERC20 StableCoin or tokens

In the case a winner's wallet is blacklisted by a Stablecoin or ERC20 token prior to winning in a contest, the transaction that should distribute their reward will fail; hence leaving them out of the reward pool and their funds not being dispersed.

## Vulnerability Detail

This issue can be seen in the _distribute function of the `Distributor.sol` contract from lines 116-156 not considering an instance where one of the winners' address is blacklisted by the reward token which essentially prevents them from being rewarded and rendering a fix for such a scenario.

## Impact

This will appear to be a dishonest and insincere scenario(s) because the winners will prolly be alerted they were part of the winners but won't receive any reward whatsoever thereby being portraying the protocol a bad image.

## Code Snippet

```solidity
  uint256 winnersLength = winners.length; // cache length
  for (uint256 i; i < winnersLength;) {
    // @audit handle case for user's address blacklisted by reward token
    uint256 amount = totalAmount * percentages[i] / BASIS_POINTS;
    erc20.safeTransfer(winners[i], amount);
    unchecked {
        ++i;
    }
  }
```

## Tool Used

Manual Review / VSCode

## Recommendation

Plug in checks for cases where winner(s) from the list of addresses is blacklisted > handle cases for blacklisted addresses (e.g save them, skip them, keep their rewards for a later date, multiple ways to handle their distribution) > proceed to distribute rewards to non affected/non-blacklisted winner addresses.