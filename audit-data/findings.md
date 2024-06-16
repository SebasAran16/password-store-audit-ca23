### [S-#] Storage State Variable on-chain is publicly accessible despite visibility. Password is not private

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

### [S-#] High impact method 'PasswordStore::setPassword' not restricted. Anyone can set a new password

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

### [S-#] TITLE (Root Cause + Impact) `PasswordStore::getPassword` natspec indicates a parameter that does not exist, leaving incorrect documentation of the code.

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
