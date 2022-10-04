---
eip: <to be assigned>
title: Approval Expirations for ERC20 Tokens
description: This EIP introduces a standardized way for users to set expiration limits on ERC-20 token approvals.
author: Go+ Security <@GoPlusSecurity>, Jeff Hall <@JeffHal25193696>, Xavi <@XaaaaavitheFool>, Turan Vural <@turanzv>
discussions-to: <URL>
status: Draft
type: Standards Track
category (*only required for Standards Track): Interface
created: 2022-10-02) format
requires (*optional):
---

This EIP introduces a standard way for users to set expiration limits on ERC-20 token approvals to prevent the loss of assets due to contractrual vulnerabilities or authorization attacks.

## Abstract
The standard adds a customizable expiration date as a limit to the _approve() function of token contracts, such as that of the ERC-20 interface, so that all allownces that exceed the expiration date will expire. In this way, the user's approvals for assets will be automatically recalled within a certain period. When an attacker exploits a spender contract vulnerability or spender identity to steal assets, it will fail due to the expiration of approvals.

## Motivation
The approval attack is the most common attack method in the industry as of the writing of this EIP, with nearly all phishing attacks and asset theft related to it. Although developing good usage habits and having basic knowledge of on-chain security basics can help users avoid phishing, scams, DNS hijacking, JS injection, and other common approval attack methods, this still can't stop the attacks caused by the spender's own security problems.

In the Transit Swap attack of October 1st, 2022, hackers used a vulnerability in the Transit Swap contract to steal $200 million in assets. Due in part to professional security audits and the strong brand recognition of Transit Swap, many users were given a false sense of security and trusted the platform's service. Transit Swaps's contract had been running smoothly for over a year and had accumulated a large amount of user allownces, making the contract a high-reward target for malicious actors.

There have been many similar incidents to the Transit Swap attack. Security audits cannot gurantee that a contract is 100% devoid of vulnerability. Some vulnerabilities are intentionally left by malicious developers themselves. In line with good trust practices and general security hygeine, unconditional trust is risky, and modifying approvals into limited trusts in spender with automatic recovery will be a key tool to natively prevent attacks.

## Specification

Validity limits are introduced by expanding `_allowance()` into a struct `Allowance{}` and adding the `_expiration` timestamp.

The `_expiration` parameter is set via a new input `period` into the `approve()` method.

A `DEFAULT_PERIOD` constant is added for legacy implementations that do not specify the `period` parameter.

A separate approval method with the header `approve(address spender, uint256 amount, uint256 period)` modifies the `_expiration` value for existing approvals.

```solidity
contract ERC20 is Context, IERC20, IERC20Metadata {

    struct Allowance {
        uint256 _amount;
        uint256 _expiration;
    }

    mapping(address => uint256) private _balances;

    mapping(address => mapping(address => Allowance)) private _allowances;

    uint256 private constant DEFAULT_PERIOD = 86400 * 30;

    uint256 private _totalSupply;

    string private _name;
    string private _symbol;


    function approve(address spender, uint256 amount) public virtual override returns (bool) {
        address owner = _msgSender();
        _approve(owner, spender, amount, DEFAULT_PERIOD);
        return true;
    }


    function approve(address spender, uint256 amount, uint256 period) public returns (bool) {
        address owner = _msgSender();
        _approve(owner, spender, amount, period);
        return true;
    }
    
    
    function _approve(
        address owner,
        address spender,
        uint256 amount,
        uint256 period
    ) internal virtual {
        require(owner != address(0), "ERC20: approve from the zero address");
        require(spender != address(0), "ERC20: approve to the zero address");

        _allowances[owner][spender]._amount = amount;
        _allowances[owner][spender]._expiration = block.timestamp + period;
        emit Approval(owner, spender, amount);
    }
```

Transactions using the `transferFrom()` method will be checked against `_expiration` through the `_spendAllowance()`. Transactions that take place after the alloted time period will fail.

```solidity
    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public virtual override returns (bool) {
        address spender = _msgSender();
        _spendAllowance(from, spender, amount);
        _transfer(from, to, amount);
        return true;
    }

    function _spendAllowance(
        address owner,
        address spender,
        uint256 amount
    ) internal virtual {
        Allowance memory currentAllowance = _allowances[owner][spender];
        require(block.timestamp <= currentAllowance._expiration, "Approval expired!");
        if (currentAllowance._amount != type(uint256).max) {
            require(currentAllowance._amount >= amount, "ERC20: insufficient allowance");
            unchecked {
                _approve(owner, spender, currentAllowance._amount - amount, currentAllowance._expiration - block.timestamp);
            }
        }
    }
```

The `allowance()`, `increaseAllowance()`, and `decreaseAllowance()` method has been modified to accommodate the new features. `allowanceExpiration()` has been added to query the expiry date.

```solidity
    function allowance(address owner, address spender) public view virtual override returns (uint256) {
        return _allowances[owner][spender]._amount;
    }

    function allowanceexpiration(address owner, address spender) public view returns (uint256) {
        return _allowances[owner][spender]._expiration;
    }
    
    
    function increaseAllowance(address spender, uint256 addedValue) public virtual returns (bool) {
        address owner = _msgSender();
        _approve(owner, spender, allowance(owner, spender) + addedValue, DEFAULT_PERIOD);
        return true;
    }

    function increaseAllowance(address spender, uint256 addedValue, uint256 period) public virtual returns (bool) {
        address owner = _msgSender();
        _approve(owner, spender, allowance(owner, spender) + addedValue, period);
        return true;
    }
    
    
    function decreaseAllowance(address spender, uint256 subtractedValue) public virtual returns (bool) {
        address owner = _msgSender();
        uint256 currentAllowance = allowance(owner, spender);
        require(currentAllowance >= subtractedValue, "ERC20: decreased allowance below zero");
        unchecked {
            _approve(owner, spender, currentAllowance - subtractedValue, DEFAULT_PERIOD);
        }

        return true;
    }

    function decreaseAllowance(address spender, uint256 subtractedValue, uint256 period) public virtual returns (bool) {
        address owner = _msgSender();
        uint256 currentAllowance = allowance(owner, spender);
        require(currentAllowance >= subtractedValue, "ERC20: decreased allowance below zero");
        unchecked {
            _approve(owner, spender, currentAllowance - subtractedValue, period);
        }

        return true;
    }
```

The above modifications adds an expiration date to ERC20 token approvals. Users can freely update their allowance validity at any time by calling `approve(address spender, uint256 amount, uint256 period)`.

## Rationale
The `Allowance{}` struct is created in order to add expiration functionality to `original_allowance()`.

At the same time, this standard is absolutely compatible with the traditional ERC20 standard.  `DEFAULT_PERIOD` is added so that the original `approve(address spender, uint256 amount)` method header can remain for backwards compatibility and ease of programming. `DEFAULT_PERIOD` can be set to a constant or variable according to the developer's needs.

A separate approval method with the header `approve(address spender, uint256 amount, uint256 period)` has been introduced to allow users to customize the `_expiration` value of the approval.

The internal method `_spendAllowance()` is introduced for code cleanliness with the function of checking the allowance amount and expiration.

The `allowance()` method has been modified to accommodate the functionality described  in this EIP. `allowanceExpiration()` has been added query the allowance expiration date.

## Backwards Compatibility
This standard is compatible with the ERC-20, ERC-721 and ERC-1155 standards. Tokens issued in the form of proxy contracts can be updated to comply with this EIP.

## Reference Implementation
Implementation can be referenced in `../assets/eip-####/`.

## Security Considerations
When upgrading a standard ERC20-based proxy contract to this standard, attention should be paid to the location of assets.

## Copyright
Copyright and related rights waived via [CC0](../LICENSE.md).
