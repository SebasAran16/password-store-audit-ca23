---
title: PasswordStore Audit Report
author: Sebastian Zambrano Arango
date: June 16, 2024
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
  - \usepackage[utf8]{inputenc}
  - \usepackage{fontspec}
  - \usepackage{listingsutf8}
  - \setmonofont{DejaVuSansMono}
  - |
    \lstset{
      basicstyle=\ttfamily,
      columns=fullflexible,
      keepspaces=true,
      literate={└}{{\texttt{\symbol{"2514}}}}1
               {──}{{\texttt{\symbol{"2500}\symbol{"2500}}}}1
               {├}{{\texttt{\symbol{"251C}}}}1
               {│}{{\texttt{\symbol{"2502}}}}1
    }
---

\begin{titlepage}
\centering
\begin{figure}[h]
\centering
\includegraphics[width=0.5\textwidth]{logo.pdf}
\end{figure}
\vspace*{2cm}
{\Huge\bfseries PasswordStore Audit Report\par}
\vspace{1cm}
{\Large Version 1.0\par}
\vspace{2cm}
{\Large\itshape Sebastian Zambrano Arango\par}
\vfill
{\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared and Audited by: [Sebastian Zambrano Arango](https://sebastianarango.com)

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
    - [Scope](#scope)
    - [Roles](#roles)
- [Executive Summary](#executive-summary)
    - [Issues found](#issues-found)
- [Findings](#findings)
- [High](#high)
- [Medium](#medium)
- [Low](#low)
- [Informational](#informational)
- [Gas](#gas)

# Protocol Summary

PasswordStore is a protocol designed to securely store and retrieve a user's password on the blockchain. The smart contract is intended for use by a single address, which has exclusive permissions to set and access the stored password.

# Disclaimer

Sebastian Zambrano Arango team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details

Commit Hash:
```
ff694529248699c59d991022108530247a92d56d
```

## Scope

```
./src/
#- PasswordStore.sol
```

## Roles

- **Owner:** The user that can both set and retrieve the save password.
- **Outsiders:** Any other user not having the owner role interacting with the contract.

# Executive Summary

General notes about the audit, like: Hours worked, complications, eases, findings, etc..

## Issues found

| Severity      | Number of Issues Found |
|---------------|------------------------|
| High          | 2                      |
| Medium        | 0                      |
| Low           | 0                      |
| Informational | 1                      |
| Gas           | NA                     |
| Total         | 3                      |

# Findings

## High

### [H-1] Storage State Variable on-chain is publicly accessible despite visibility. Password is not private

**Description:** Any date stored on-chain is visible to anyone, being readable directly from Blockchain. The `PasswordStore::s_password` variable is intended to be private and only be accessed through `PasswordStore::getPassword`, restricted to the owner of the contract, however the getter method becomes pointless.

We show one such method of reading any data off chain below.

**Impact:** Anyone can read the private password, severely breaking the functionality of the protocol.

**Proof of Concept:** The below test case shows how anyone can read the password directly from the Blockchain:


1. Create local running Blockchain:
```bash
make anvil
```

2. Deploy the contract to the chain:
```bash
make deploy
```

3. Get the storage slot for the password:
```bash
forge in <CONTRACT_ADDRESS> storage
```

4. Get the storage value for the slot on the local Blockchain and parse it to ascii:
```bash
cast storage <CONTRACT_ADDRESS> 1 --rpc-url http://127.0.0.1:8545 | xargs -I PASSWORD cast --to-ascii PASSWORD
```

**Recommended Mitigation:** Due to this finding the architecture of the contract should be rethought.

One possible wat to solve this is to encrypt the password off-chain, and save the encrypted result on-chain. However, this would require te user to remember a decryption password off-chain to get the original one. Additionally, there is caution needed when maintaining a getter for the original password, as this would require sending the decryption password creating a possible vulnerability of exposing it, meaning exposing the original one.

### [H-2] High impact method 'PasswordStore::setPassword' not restricted. Anyone can set a new password

**Description:** Although the 'PasswordStore::getPassword' is restricted to owner only, the 'PasswordStore::setPassword' is not restricted. Allowing anyone to set a new password, although it is supposed to only be accessible to the contract's owner.

```javascript
function setPassword(string memory newPassword) external {
->      // @audit - No AccessControl        
        s_password = newPassword;
        emit SetNetPassword();
    }
```

**Impact:** Anyone can set/change the password of the contract, severely breaking the Contract intended functionality.

**Proof of Concept:** The test `testFuzz_anyone_can_set_password` was added to `PasswordStore.t.sol` and is a proof that any random address (not being the owner) can set a new password and next time the owner get the Smart Contract address the new set password by a random address is returned:

<details>
<summary>Test</summary>

```javascript
function testFuzz_anyone_can_set_password(address randomAddress) public {
        vm.assume(randomAddress != owner);

        string memory expectedPassword = "newPassword";
        vm.prank(randomAddress);
        passwordStore.setPassword(expectedPassword);

        vm.prank(owner);
        string memory newPassword = passwordStore.getPassword();

        assertEq(newPassword, expectedPassword);
    }
```

</details>

**Recommended Mitigation:** To either extend the `AccessControl` contract from `OpenZeppelin` to the Smart Contract and the usage of the `onlyRole` modifier or limit the usage with the `owner` state variable:

<details>
<summary>AccessControl</summary>

```javascript
    function setPassword(string memory newPassword) external onlyRole(OWNER_ROLE) {
    s_password = newPassword;
    emit SetNetPassword();
    }
```

</details>
<details>
<summary>State Variable</summary>

```javascript
    if (msg.sender != s_owner) {
    revert PasswordStore_NotOwner();
}
_;
```

</details>

## Informational

### [I-1] `PasswordStore::getPassword` natspec indicates a parameter that does not exist, leaving incorrect documentation of the code.

**Description:**

```javascript
    /*
     * @notice This allows only the owner to retrieve the password.
->   * @param newPassword The new password to set.
     */
    function getPassword() external view returns (string memory) {
```

The `PasswordStore::getPassword` function signature is `getPassword()` while the natspec say it should be `getPassword(string)`.

**Impact:** Contract documentation is incorrect.

**Recommended Mitigation:** Remove commented line in the Contract natspec.

```diff
-   * @param newPassword The new password to set.
```