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