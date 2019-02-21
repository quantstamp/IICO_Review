# Interactive Initial Coin Offering Protocol

## Overview

This smart contract audit was prepared by [Quantstamp](https://www.quantstamp.com/), the protocol for securing smart contracts.

This security audit report follows a generic template. Future Quantstamp reports will follow a similar template and they will be fully generated by automated tools.

## Specification

Our understanding of the specification was based on the following documentation:

* [IICO Paper](https://people.cs.uchicago.edu/~teutsch/papers/ico.pdf)
* [IICO Protocol Medium Post](https://medium.com/truebit/interactive-coin-offering-a-protocol-explained-267065ef3819) 
* [IICO Crowdsale Medium Post](https://medium.com/modular-network/building-the-interactive-token-sale-1545514bf108)

We also reviewed the instructions provided in the [README.md](https://github.com/TrueBitFoundation/interactive-coin-offerings) file in the github repository at the time of the audit.

## Methodology

The review was conducted during 2018-Mar-13 through 2018-Apr-13 by the Quantstamp team, which included senior engineers Kacper Bak, Jonathan Haas, and Ed Zulkoski.

Their procedure can be summarized as follows:

1. Code review
    1. Review of the specification
    2. Manual review of code
    3. Comparison to specification
2. Testing and automated analysis
    1. Test coverage analysis
    2. Symbolic execution (automated code path evaluation)
3. Best-practices review
4. Itemize recommendations

## Source Code

The following source code was reviewed during the audit.

|Repository	|Commit	|
|---	|---	|
|[interactive-coin-offerings](https://github.com/TrueBitFoundation/interactive-coin-offerings)	|[766a511](https://github.com/TrueBitFoundation/interactive-coin-offerings/commit/766a511c676e5f962e070d4c3c3feda118c2f1af)	|

# Security Audit

Quantstamp's objective was to evaluate the Interactive Initial Coin Offering Protocol contract repository for security-related issues, code quality, and adherence to best-practices.

Possible issues include (but are not limited to):

* Transaction-ordering dependence
* Timestamp dependence
* Mishandled exceptions and call stack limits
* Unsafe external calls
* Integer overflow / underflow
* Number rounding errors
* Reentrancy and cross-function vulnerabilities
* Denial of service / logical oversights

# Test coverage

We evaluated the test coverage using truffle and solidity-coverage. The below notes outline the setup and steps that were performed.

## Setup

Testing setup: 

* Truffle v4.0.1
* TestRPC v6.0.7 
* solidity-coverage v0.4.9
* Oyente v0.2.7 
* Mythril v0.10.7
* truffle-flattener v1.2.1

## Steps

Steps taken to run the full test suite:

* Installed dependencies as described in the file [README.md](https://github.com/TrueBitFoundation/interactive-coin-offerings/blob/master/README.md).
* Ran the truffle tests: `bash run_test.sh`.
* Installed the `solidity-coverage` tool: `npm install --save-dev solidity-coverage`.
* Ran the coverage tool: `./node_modules/.bin/solidity-coverage`.
* Installed the `mythril` tool from Pypi: `pip3 install mythril`.
* Ran the `mythril` tool: `myth -x /interactive-coin-offerings/truffle/contracts/`.
* To workaround limitations of the `Oyente` tool, we flattened the source code using `truffle-flattener`.
* Installed the `Oyente` tool from Docker: `docker pull luongnguyen/oyente && docker run -i -t luongnguyen/oyente`.
* Ran the `Oyente` tool: `cd /oyente/oyente && python oyente.py -s Contract.sol`.

## Evaluation

The coverage result of the `InteractiveCrowdsaleLib.sol` file:

```
99.42% Statements 172/173
78.43% Branches 80/102
100% Functions 16/16
99.5% Lines 198/199
```

The only uncovered line was in a function to log error messages; since this was simply used for testing this is not an issue (`InteractiveCrowdsaleTestContract#46`).

Symbolic execution (the Oyente tool) did not detect any vulnerabilities of types Parity Multisig Bug 2, Transaction-Ordering Dependence (TOD), Callstack Depth Attack, Timestamp Dependency, and Re-Entrancy Vulnerability.

The Mythril tool reported a number of potential vulnerabilities within the `CrowdsaleLib.sol` file. 
Mythril specified a number of calls with gas to dynamic addresses (`CrowdsaleLib.sol#246`, `CrowdsaleLib.sol#178`, `CrowdsaleLib.sol#170`, and `CrowdsaleLib.sol#204`). At the moment, Mythril reports external calls to dynamic addresses (from storage, calldata or `msg.sender`) with at least 2,300 gas. We confirmed, however, that the logic of the calling contract is unaffected if the called contract misbehaves (e.g. attempts to exploit reentrancy). **After further manual analysis, we have classified these calls as false positives, given our explanation below.**
 The calls are as follows: 

* `self.token.balanceOf(this)`, `self.token.burnToken(_burnAmount)` 
    * As documented within [PT](https://www.pivotaltracker.com/n/projects/1189488/stories/85006746), contract types are implicitly convertible to `address` and explicitly convertible to and from all integer types. Although Mythril reports `this` as a dynamic address, the usage of such is not of concern for reentrancy. This is demonstrated below. 

```
contract Helper {
  function getBalance() returns(uint) {
    return this.balance; // balance is "inherited" from the address type
  }
}
contract IsAnAddress {
  Helper helper;
  function setHelper(address a) {
    helper = Helper(a); // Explicit conversion
  }
  function sendAll() {
    // send all funds to helper (both balance and send are address members)
    helper.send(this.balance);
  }
}
```

* `self.token.transfer(msg.sender, total)` 
    * `someAddress.send()` and `someAddress.transfer()` are considered safe against reentrancy. While these methods may still trigger custom code execution (as contracts can execute code when they receive ether), the called contract is only given a stipend of 2,300 gas which is currently only enough to log an event.

* Mythril reports a potential integer underflow within `CrowdsaleLib.sol#282`, in the statement `return self.startingTokenBalance - self.withdrawTokensMap[self.owner]`. As values within `withdrawTokensMap` are always less than or equal to `startingTokenBalance` and  `startingTokenBalance` is always a positive value or zero (as it is of type `uint256`), this operation is not of concern.
    * For usage of `CrowdsaleLib` outside of the context of IICO, we recommend that either `assert` or `BasicMathLib`, or [SafeMath library](https://github.com/OpenZeppelin/zeppelin-solidity/blob/master/contracts/math/SafeMath.sol) is used to protect against possible integer underflow. As per the [Solidity Documentation](https://solidity.readthedocs.io/en/develop/control-structures.html#error-handling-assert-require-revert-and-exceptions), usage of `assert` causes the EVM to revert all changes made to the state as there is no safe way to continue execution, because an expected effect did not occur. Furthermore, following this paradigm allows formal analysis tools to verify that the invalid opcode can never be reached, meaning code invariants are always preserved.
* As a transaction's miner has leeway in reporting the time at which the mining occurred (to a certain extent), `block.timestamp` should be used with hesitation in circumstances requiring second-level precision in measuring time. As IICO is designed for usage over a far larger increment of time (e.g. a month) usage of `block.timestamp` within the contract does not pose a concern.

# Recommendations

* The constant `1000000000000000000` is used throughout `InteractiveCrowdsaleLib`. We recommend storing it as a named constant.
* The unnamed constant `100` is utilized in `CrowdsaleLib.sol#115` (in the function `init()` called by `InteractiveCrowdsaleLib.sol`): `require((_saleData[i + 2] == 0) || (_saleData[i + 2] >= 100));`. This number imposes a limit to the number of tokens that each individual address can buy. The rationale for choosing `100` is not present within the naming of the constant nor relevant documentation. We recommend naming the constant and documenting its purpose and rationale behind picking this specific number.
* We recommend the implementation of timestamp processing for sale data be adjusted to account for the scenario in which the initial index (`CrowdsaleLib.sol#264`) is considered a valid milestone time (`CrowdsaleLib.sol#266`) -- this can be accomplished using `<=` in place of the present `<` in `CrowdsaleLib.sol#264` as shown below:

```
    while((index <= self.milestoneTimes.length) && (self.milestoneTimes[index] <= timestamp)) {
      index++;
    }
```



## Usage of Contracts Interdependently 

Quantstamp evaluated the usage of all relevant contracts in the context of the IICO contract. Our presumptions throughout (and relevant recommendations) are dependent upon this use case. If contracts are re-used in other contexts, e.g., when `CrowdSaleLib.sol` is used in other token sales, our findings and recommendations may not be applicable. For example, depending on the initialization values, integer under-/overflows may occur in `CrowdSaleLib.sol`.

## Usage of BasicMathLib

We noted that most arithmetic expressions within the contract do not utilize `BasicMathLib.sol`, primarily due to basic integer overflow checking inline. In instances where a secondary level of assurance is required that no overflow or arithmetic-focused vulnerabilities are to occur, we suggest using `BasicMathLib.sol`. While this does add an additional gas overhead, `BasicMathLib.sol` provides an additional layer of protection against arithmetic vulnerabilities.  Furthermore, minimization of inline overflow checking promotes code clarity.

## Code Documentation

We noted that the majority of the functions were self-explanatory, and standard documentation tags (such as `@dev`, `@param`, and `@returns`) were included throughout.

The following are some minor (mostly typographical) issues throughout the code:

* InteractiveCrowdsaleLib#65:  "fluctations" -> "fluctuations"
* InteractiveCrowdsaleLib#78:  "minimim" -> "minimum"
* InteractiveCrowdsaleLib#106: "manual" -> "Manual"
* InteractiveCrowdsaleLib#136: "minimim" -> "minimum"
* InteractiveCrowdsaleLib#136: "successfull" -> "successful"
* InteractiveCrowdsaleLib#182: "caluclating" -> "calculating"
* InteractiveCrowdsaleLib#195: "amound" -> "amount"
* InteractiveCrowdsaleLib#226: "becuase" -> "because"
* InteractiveCrowdsaleLib#340: "commitements" -> "commitments"
* InteractiveCrowdsaleLib#359: "succesful" -> "successful"
* InteractiveCrowdsaleLib#554: "retreiveFinalResult(...)" -> "retrieveFinalResult(...)"
* InteractiveCrowdsaleLib#592: "subtract that amount the address..." -> "subtract that amount *from* the address..."
* InteractiveCrowdsaleLib#603: "can't" -> "Can't"
* InteractiveCrowdsaleLib#197-198: the type ( `uint256`) is specified on 197 but not 198.

## Minor Inconsistency Between the Code and Medium Post

In a [Medium post](https://medium.com/modular-network/building-the-interactive-token-sale-1545514bf108), the following equation states that a person bid + cap guarantees them a certain percentage of all tokens:

```
minimum percentage = (ETH Payment)/(personal valuation cap)
```

This is not technically true, due to bonuses. As an example, suppose tokensPerEth = 1, the linearly decreasing bonus percentage is set to 300% (e.g. at the beginning of the bonus phase, 1 ether gets 4 tokens), and at the end of the sale 2M ether gets raised. Suppose that at the very beginning of the crowdsale, Alice contributes 1M ether obtaining 4M tokens. No other contributions occur during the bonus phase. During the second [non-bonus] phase, the remaining 1M ether gets raised, among which Bob contributes 100k ether at a valuation cap of 2M. Bob's expected minimum percentage of tokens according to the formula is 100k/2M = 5%. However, 5M total tokens are minted, so Bob's actual percentage is 100k/5M = 2%.

# Appendix

## File Signatures

Below are SHA256 file signatures of the relevant files reviewed in the audit.

```
$ shasum -a 256 ./truffle/contracts/*
d22abd24f7bb362f5299d79a6046bd09dec8200cf8e5872eadf8a95d60fa4001  ./truffle/contracts/Array256Lib.sol
9795777efcc1f276fd90c227db10d3de44874c1302e732b64a2509d0745699a3  ./truffle/contracts/BasicMathLib.sol
00c80abd22d3da993b66d122214225e608bf6470012fa68dab5acfc7656fbdb3  ./truffle/contracts/CrowdsaleLib.sol
0f47105f42d2fb9207617736f0b491d81e82c662b48ba153f23b422c6ae6004b  ./truffle/contracts/CrowdsaleToken.sol
c728a7c5f2a675098a33280f13b0603a26e56c6c49fc71a73e2576a1bf0f73f8  ./truffle/contracts/InteractiveCrowdsaleLib.sol
add147268b9ee84f6aba53c66a281efc6ad2b93dc6013e0e7d10aa0d225bea2c  ./truffle/contracts/InteractiveCrowdsaleTestContract.sol
bd54de4375748d5d50052a78d591e458fef40a51a6f55a5c75af8f55ada7b763  ./truffle/contracts/LinkedListLib.sol
0f282e11243b040cdfa7a9056d28cc31d8762d0a9766bde9c39d04a4f2c9592e  ./truffle/contracts/Migrations.sol
7df8209deec3e3ee0403d70862edcd2fcd54500bfc38e32a0233a145e8215350  ./truffle/contracts/TokenLib.sol

$ shasum -a 256 ./truffle/test/* ./truffle/test/*/* ./truffle/test/*/*/*
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  ./truffle/test/helpers
4a8f2b05078893e3418e9d58f0c2b420281064082cd5b010d1340f93871b4493  ./truffle/test/interactivecrowdsalelib.js
e5d05872e4a802b243d5df3fa34141293b952c113f3dc1d8e176320f40d33ef1  ./truffle/test/helpers/advanceToBlock.js
6c8798eb0f7d8e7efbc3e2e3f74d9192e15f377d012dfd8257f68caa2798cfe1  ./truffle/test/helpers/ether.js
3cc2c670f02e3f85d8bacd32cfd1f45777dc8e38c3e205ebf6bd51866da23060  ./truffle/test/helpers/increaseTime.js
afb1f613e33da03a862d7915e4de1979b9e262b728d47a768d347d7119dfb36a  ./truffle/test/helpers/latestTime.js
1f44ddc6461826e472e656d2ecdb7f735816b1c28496359d2115c14e82e08896  ./truffle/test/helpers/pointerUtils.js
daeba9b4d0d13f98f5ab6c13e7f8531eae0a7199cea676fc83a782fd008481a7  ./truffle/test/helpers/timer.js
b9dd367cc368e60feeb4b174f28502114b5e5c5b8cfd9d164db0ecd09f6b33a3  ./truffle/test/helpers/utils.js
```

# Disclosure

## Purpose of report

The scope of our review is limited to a review of Solidity code and only the source code we note as being within the scope of our review within this report. Cryptographic tokens are emergent technologies and carry with them high levels of technical risk and uncertainty. The Solidity language itself remains under development and is subject to unknown risks and flaws. The review does not extend to the compiler layer, or any other areas beyond Solidity that could present security risks.

The report is not an endorsement or indictment of any particular project or team, and the report does not guarantee the security of any particular project. This report does not consider, and should not be interpreted as considering or having any bearing on, the potential economics of a token, token sale or any other product, service or other asset.

No third party should rely on the reports in any way, including for the purpose of making any decisions to buy or sell any token, product, service or other asset. Specifically, for the avoidance of doubt, this report does not constitute investment advice, is not intended to be relied upon as investment advice, is not an endorsement of this project or team, and it is not a guarantee as to the absolute security of the project.

## Links to other websites

You may, through hypertext or other computer links, gain access to web sites operated by persons other than Quantstamp, Inc. (Quantstamp). Such hyperlinks are provided for your reference and convenience only, and are the exclusive responsibility of such web sites' owners. You agree that Quantstamp are not responsible for the content or operation of such web sites, and that Quantstamp shall have no liability to you or any other person or entity for the use of third-party web sites. Except as described below, a hyperlink from this web site to another web site does not imply or mean that Quantstamp endorses the content on that web site or the operator or operations of that site. You are solely responsible for determining the extent to which you may use any content at any other web sites to which you link from the report. Quantstamp assumes no responsibility for the use of third-party software on the website and shall have no liability whatsoever to any person or entity for the accuracy or completeness of any outcome generated by such software.

## Timeliness of content

The content contained in the report is current as of the date appearing on the report and is subject to change without notice, unless indicated otherwise by Quantstamp; however, Quantstamp does not guarantee or warrant the accuracy, timeliness, or completeness of any report you access using the internet or other means, and assumes no obligation to update any information following publication.

©2018 Quantstamp, Inc.  All rights reserved. This content shall not be used, copied, modified, redistributed or otherwise disseminated except to the extent expressly permitted by Quantstamp.

DISCLAIMER:  
This report is based on the scope of materials and documentation provided for a limited review at the time provided. Results may not be complete nor inclusive of all vulnerabilities.  The review and this report are provided on an as-is, where-is, and as-available basis. You agree that your access and/or use, including but not limited to any associated services, products, protocols, platforms, content, and materials, will be at your sole risk. Cryptographic tokens are emergent technologies and carry with them high levels of technical risk and uncertainty. The Solidity language itself and other smart contract languages remain under development and are subject to unknown risks and flaws. The review does not extend to the compiler layer, or any other areas beyond Solidity or the smart contract programming language, or other programming aspects that could present security risks.  You may risk loss of tokens, Ether, and/or other loss.  A report is not an endorsement (or other opinion) of any particular project or team, and the report does not guarantee the security of any particular project. A report does not consider, and should not be interpreted as considering or having any bearing on, the potential economics of a token, token sale or any other product, service or other asset.  No third party should rely on the reports in any way, including for the purpose of making any decisions to buy or sell any token, product, service or other asset.  To the fullest extent permitted by law, we disclaim all warranties, express or implied, in connection with this report, its content, and the related services and products and your use thereof, including, without limitation, the implied warranties of merchantability, fitness for a particular purpose, and non-infringement. We do not warrant, endorse, guarantee, or assume responsibility for any product or service advertised or offered by a third party through the report, its content, and the related services and products, any open source or third party software, code, libraries, materials, or information linked to, called by, referenced by or accessible through the product, any hyperlinked website, or any website or mobile application featured in any banner or other advertising, and we will not be a party to or in any way be responsible for monitoring any transaction between you and any third-party providers of products or services. As with the purchase or use of a product or service through any medium or in any environment, you should use your best judgment and exercise caution where appropriate.   FOR AVOIDANCE OF DOUBT, THE REPORT, ITS CONTENT, ACCESS, AND/OR USAGE THEREOF, INCLUDING ANY ASSOCIATED SERVICES OR MATERIALS, SHALL NOT BE CONSIDERED OR RELIED UPON AS ANY FORM OF FINANCIAL, INVESTMENT, TAX, LEGAL, REGULATORY, OR OTHER ADVICE.
