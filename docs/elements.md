## Elements
***

### **rTokens**

rTokens are wrapped versions of underlying ERC20 assets that earn interest that can be redirected to flow to recipients that the holder of the underlying asset chooses.

The rToken contract keeps track of the total amount of minted rTokens, user balances, and User Buckets. 

##### *Deployment*

At deployment, an rToken contract must specify the address of the underlying ERC20 asset. To be useful as an rToken, the underlying ERC20 asset should be available to supply on one or more  platforms for which an Allocation Strategy has been written to manage and redirect interest.

##### *rToken Accounts*

Each address has an rToken Account. The account data includes:
`userBuckets` – an array specifying the amounts of rTokens committed to each active `hatID`
internal accounting information,
account statistics.

##### *Minting*

rTokens are minted and redeemed at a ratio of 1:1 to their underlying asset. So 25 DAI can mint 25 rDAI, which are then redeemable for 25 DAI.

Like other ERC20 wrapper contracts, rToken contracts require holders of the underlying asset to `approve` the amount of the asset to be used to mint rTokens. This amount can be queried with `allowance`.

There are three minting functions:

1. `mint(uint256 amount)` - 

1. `mintWithSelectedHat(uint256 amount uint256 hatID)` - 

##### *Control Over Interest Streams*

The holder of the underlying asset remains in control of where the interest streams flow, but cannot claw back interest that has already been redirected. The holder can stop sending interest to a recipient (or recipients) by: 1) redeeming all the principal for the underlying asset or 2) changing the set of recipients with the `changeHat` function. 

##### *Accounting Construct*

rTokens work by recording the amount of the underlying asset that an address mints (the principal) as a fixed debt that must be repaid, and then “lending” out the now interest earning principal to the interest recipients. The recipients can redeem the amount of rTokens that have accrued from interest, but they cannot redeem the debt portion which represents the principal.

*Example*
 
> Alice mints 1,000 rDAI and designates Bob as the only interest recipient. 
> 
> At that time Bob has received a loan of 1,000 rDAI and a debt to repay 1,000 rDAI to Alice, meaning a net position of 0 rDAI.  
>    
> If Bob calls redeemAll() to try to get all the underlying DAI, he will not receive anything because he can only redeem the amount in excess of the debt. 
>     
> Now fast forward to a time when 75 DAI in interest has accumulated. Bob's “loan” is now worth 1,075 DAI but his debt remains 1,000 DAI, so he can redeem 75 DAI.   



##### *Common write functions*

###### Initialize

- `initialize(address underlyingAsset, string memory name_, string memory symbol_, uint256 decimals_) public` – set the parameters of an rToken at deployment

###### Approve
`approve(address owner address spender) external public returns (bool) ` - allow rToken minting from underlying ERC20 asset

###### Minting
- `mint(uint256 mintAmount) external nonReentrant returns (bool)` - swap underlying asset for rTokens at 1:1 ratio, e.g. `mint` 100 rDAI from 100 DAI - by default, will send interest to minter's address 


- `mintWithSelectedHat(uint256 mintAmount, uint256 hatID) external nonReentrant returns (bool)` - mint rTokens into specified `hatID` 

- `mintWithNewHat(uint256 mintAmount, address[] calldata recipients, uint32[] calldata proportions) external nonReentrant returns (bool)` - create a new hat and mint rTokens into that `hatID`

- NEW FOR V2 `mintWithUserBuckets(uint256 mintAmount) external nonReentrant returns (bool)` - same as `mint` except the newly minted rTokens will increase all `userBucket` balances proportionally - can be gas-inefficient  

###### Redemption
- `redeem(uint256 redeemAmount) external nonReentrant returns (bool)` - redeem specified rTokens for underlying ERC20 asset at 1:1 ratio

- `redeemAll() external nonReentrant returns (bool)` - redeem all rTokens for underlying ERC20 asset at 1:1 ratio

###### Transfers
- `transfer(address dst, uint256 amount) external nonReentrant returns (bool)` - standard ERC20 transfer function  

- `transferAll(address dst) external nonReentrant returns (bool)` - standard ERC20 transferAll function - can be gas inefficient if userBuckets include multiple allocation strategies

###### Claiming Accrued Interest
- `payInterest(address )`

###### Changing Hats
- `changeHatForBucket(uint256 newHatID, uint32 bucket) external nonReentrant returns (bool)`

- `changeHatForAllBuckets(uint256 newHatID) external nonReentrant returns (bool)`

##### *Common read functions*

- `getHatsByAddress(address owner) external view returns (hats[])`

- `getBucketsByAddress(address owner) external view returns (userBuckets[])`

- `receivedSavingsOf(address owner) external view returns (uint256 amount)`

- `receivedLoanOf(address owner) external view returns (uint256 amount)`

- `interestPayableOf(address owner) external view returns (uint256 amount)`

- `getUnderlyingAsset() public view returns (address)`

- `balanceOf(address owner) external view returns (uint256 amount)`

- `allowance(address owner, address spender) external view returns (uint256)` 

---
### Hats


Hats contain the instructions for where to generate and redirect the interest on rTokens. Each hat specifies to which addresses the interest should flow and in what proportions, where the underlying asset should be invested to earn interest, and if the hat can be changed, by who. 
Hats are ERC721 NFTs.

##### Hat Parameters

A hat contains the following four parameters:

`recipients` - an array of up to 50 addresses where the interest should flow

`proportions` - an array specifying the proportion of the interest each recipient should receive – this array must have the same length as `recipients`

`allocationStrategy` - an address (enum?) specifying the liquidity pool to deposit the underlying asset to earn interest from (e.g. Compound, Aave, etc)

`manager` - an address specifying who can modify the hat's parameters – can be set to the 0x000... address to make a hat immutable

The parameters `recipients` and `proportions` are the central programmable components of rToken interest. The number of recipients in a hat can be just one or up to 50 (this maximum number is there to avoid a possible out-of-gas reversion error).  Proportions can be set arbitrarily and the contract will allocate the interest as a proportion of the maximum uint32 number (4,294,967,295 in decimal), so if the proportions entered were  [1, 2, 3, 4] meaning the fourth recipient received 4x the interest as the first, the `proportions` stored on chain would be [429496730, 858993459, 1288490188, 1717986918].

The `allocationStrategy` parameter points to the address of the deployed contract that invests the underlying asset in a particular lending protocol, such as Compound's cTokens or Aave's aTokens. 

The `manager` parameter will typically be the creator of a hat that others might want to use, such as a charity dapp.  The manager has the power to modify the other three parameters of the hat - `recipients` `proportions` and `allocationStrategy` - pursuant to the rules surrounding modifying hats (link). 

None of the hat parameters are exclusive, either separately or together – there are no squatter's rights to being the first manager for a potentially popular combination of `recipients`, `proportions` and `allocationStrategy.` 

##### Changeable Hats
Changeable hats have several substantive use cases, including allowing trusted dapp managers to be able to redirect interest to better serve their users' ends, to swap out of an underperforming lending pool to obtain higher yields or to switch to a more stable/liquid lending pool.rToken-powered dapps would otherwise require each user to manually change hats whenever it recommended a different set of parameters.  Changeable hats do introduce a potential for abuse by the managers, but because changes are emitted as events, malicious managers cannot act in stealth, nor do they have any control over the holder's principal assets, only forward looking interest streams.

##### Immutable Hats
There may be use cases where hat immutability is desirable to prevent the possibility of mismanagement. Immutable hats can be created by assigning the burn address (0x0000000000000000000000000000000000000000) in the `manager` parameter. 

---
### User Buckets

User Buckets, added for rToken V2, are tranches of rTokens assigned to a particular Hat. A bucket can contain some or all of the user's rToken balance.  Buckets are a design solution to the “hat yanking” problem described here[https://medium.com/rtoken-project/rdai-v2-the-redirectening-90b6a9115799].  

Each rToken holder has a `userBuckets` array with each slot specifying the `hatID` and the `amount` of the rToken used in that Hat. The first array position (slot 0) is reserved as a self-hat that redirects the interest to the user, similar to directly lending assets in a liquidity pool. 

User Buckets are created and maintained automatically within the rToken contract.

*Example*

> Assume Alice mints 100 rDAI, and donates the interest on 40 rDAI to Charity X. 
> 
> Her `userBuckets` array now contains two slots, with the self-hat slot 0 holding `amount: 60` and slot 1 pointing to `hatID: [Hat with Charity X address]` holding `amount: 40`. 
> 
> Later Alice decides to send the interest on the remaining 60 rDAI to Charity Y.  
> 
> Now `userBuckets` has three slots with `amount` 0, 40 and 60 respectively.
>
> [TBD: when a user redeems the amount in a bucket, is that `userBuckets` array slot deleted  or preserved with a 0 `amount`? Preserving old buckets might lead to state bloat, but might be useful for historical accounting purposes. Deleting buckets could reduce state size, but may lead to complications if dapps or other entities seek to query particular hat/bucket data based on slot position]

##### *Common read functions*
`getUserBuckets(address owner) public view returns []` - returns an array containing the address' `userBuckets`  

### Allocation Strategies

Allocation Strategies specify how to earn interest from a decentralized finance liquidity pools and how to track the correct rToken balance, including both the original principal amount and the amount of interest that is redirected.

Each Allocation Strategy is customized to the mechanics of a particular decentralized liquidity protocol that allows holders of underlying ERC20 assets to earn interest. Currently rToken allocation strategies have been written for Compound and Aave. 


##### Creating an Allocation Strategy


##### Whitelisting an Allocation Strategy


---
### Switchboard Registry

Hats are created, stored and modified in a registry contract called Switchboard.sol.  

Any rToken may be used with any hat, provided its underlying asset (e.g. DAI, SNX, etc) is available on the hat's allocation strategy platform.


##### Creating a New Hat 
Anyone can create a new hat by calling `createHat` and passing `address[] recipients` `uint256[] proportions` `address(enum?) allocationStrategy` and `address manager`.

Created hats are assigned a sequential immutable `hatID`. Hats cannot be deleted.

##### Modifying a Hat 
The `manager` address has the power to modify the hat's parameters, including by changing the `manager` which transfers control of the hat. 

Modifications to a hat emit the HatModified event.


modifyHat(uint256 hatID, address[] memory recipients, uint256[] memory proportions, address (enum?) allocationStrategy, address manager) external onlyManager returns (bool)
[TBD: Is there a gas savings reason to split this function into three or four separate functions like modifyHatRecipients, modifyHatProportions, modifyHatStrategy, modifyHatManager?)

