# EIP-4393 Micropayments Standard for NFTs and Multi Tokens

| Author         | Jules Lai                                                                                           |
| -------------- | --------------------------------------------------------------------------------------------------- |
| Description    | A standard interface for tip tokens that allows tipping to holders of NFTs and multi tokens         |
| Discussions-to | https://ethereum-magicians.org/t/eip-proposal-micropayments-standard-for-nfts-and-multi-tokens/7366 |
| Status         | Draft                                                                                               |
| Type           | Standards Track                                                                                     |
| Category       | ERC                                                                                                 |
| Requires       | 165, 721, 1155                                                                                      |

## Table of Contents

- [EIP-4393 Micropayments Standard for NFTs and Multi Tokens](#eip-4393-micropayments-standard-for-nfts-and-multi-tokens)
  - [Table of Contents](#table-of-contents)
  - [Abstract](#abstract)
  - [Motivation](#motivation)
  - [Specification](#specification)
    - [Tip Token transfer and value calculations](#tip-token-transfer-and-value-calculations)
    - [Royalty distribution to shared holders](#royalty-distribution-to-shared-holders)
    - [Caveats](#caveats)
    - [Minimising Gas Costs](#minimising-gas-costs)
  - [Rationale](#rationale)
    - [Simplicity](#simplicity)
    - [Use of NFTs](#use-of-nfts)
    - [New Business Models](#new-business-models)
    - [Guaranteed audit trail](#guaranteed-audit-trail)
  - [Backwards Compatibility](#backwards-compatibility)
  - [Reference Implementation](#reference-implementation)
  - [Security Considerations](#security-considerations)
  - [Copyright](#copyright)

## Abstract

For the purpose of this proposal, a micropayment is termed as a financial transaction that involves usually a small sum of money, called "tips", that are sent for specific [ERC-721](./eip-721.md) NFTs and [ERC-1155](./eip-1155.md) multi tokens as rewards to their holders. A holder (also referred to as controller) is used here as a more generic term for owner in a semantic sense, as NFTs may represent non-digital assets, such as services etc.

This standard proposal outlines a generalised way to allow tipping via implementation of an `ITipToken` interface. The interface is intentionally kept to a minimum in order to allow for maximum use cases.

A user first deposits a compatible [ERC-20](./eip-20.md) to the tip token contract that is then held in escrow, in exchange for tip tokens. These tip tokens can then be sent by the user to NFTs and multi-tokens (that have been approved by the tip token contract for tipping) to be redeemed for the original ERC20 deposits on withdrawal by the holders as rewards.

Some use cases include:

- In game purchases and other virtual goods

- Tipping messages, posts, music and video content

- Donations/crowdfunding

- Distribution of royalties

- Pay per click advertising

- Incentivising use of services

- Reward cards and coupons

These can all leverage the security, immediacy and transparency of blockchain.

## Motivation

A cheap way to send tips to any type of NFT or multi token. This can be achieved by gas optimising the tip token contract and sending the tips in batches using the `tipBatch` function from the interface.

To make it easy to implement into dapps a tipping service to reward the NFT and multi token holders. Allows for fairer distribution of revenue back to NFT holders from the user community.

To make the interface as minimal as possible in order to allow adoption into many different use cases.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

Smart contracts implementing the ERC-xxxx standard MUST implement all of the functions in the ERC-xxxx interface. MUST also emit the events specified in the interface so that a complete state of the tip token contract can be derived from the events emitted alone.

Smart contracts implementing the ERC-xxxx standard MUST implement the ERC-165 supportsInterface function and MUST return the constant value true if 0x985A3267 is passed through the interfaceID argument. Note that revert in this document MAY mean a require, throw (not recommended as depreciated) or revert solidity statement with or without error messages.

Note that, nft (or NFT in caps) in the code and as mentioned in this document, MAY also refer to an ERC-1155 fungible token.

```
interface ITipToken {
    /// @dev This emits when the tip token implementation approves the address
    /// of an NFT for tipping.
    /// The holders of the 'nft' are approved to receive rewards.
    /// When an NFT Transfer event emits, this also indicates that the approved
    /// addresses for that NFT (if any) is reset to none.
    event ApprovalForNFT(
        address[] holders,
        address indexed nft,
        uint256 indexed id,
        bool approved
    );

    /// @dev This emits when a user has deposited an ERC20 compatible token to
    /// the tip token's contract address or to an external address.
    /// This also indicates that the deposit has been exchanged for an
    /// amount of tip tokens
    event Deposit(
        address indexed user,
        address indexed rewardToken,
        uint256 amount,
        uint256 tipTokenAmount
    );

    /// @dev This emits when a holder withdraws an amount of ERC20 compatible
    /// reward. This reward comes from the tip token's contract address or from
    /// an external address, depending on the tip token implementation
    event Withdraw(
        address indexed holder,
        address indexed rewardToken,
        uint256 amount
    );

    /// @dev This emits when the tip token constructor or initialize method is
    /// executed. The 'rewardToken_' address is passed to the ERC20 constructor or
    /// initialize method.
    /// Importantly the ERC20 compatible token 'rewardToken_' to use as reward
    /// to NFT holders is set at this time and remains the same throughout the
    /// lifetime of the tip token contract.
    event InitializeTipToken(
        address indexed tipToken_,
        address indexed rewardToken_,
        address owner_
    );

    /// @dev This emits everytime a user tips an NFT holder.
    /// Also includes the reward token address and the reward token amount that
    /// will be held pending until the holder withdraws the reward tokens.
    event Tip(
        address indexed user,
        address[] holder,
        address indexed nft,
        uint256 id,
        uint256 amount,
        address rewardToken,
        uint256[] rewardTokenAmount
    );

    /// @notice Enable or disable approval for tipping for a single NFT held
    /// by a holder or a multi token shared by holders
    /// @dev MUST revert if calling nft's supportsInterface does not return
    /// true for either IERC721 or IERC1155.
    /// MUST revert if any of the 'holders' is the zero address.
    /// MUST revert if 'nft' has not approved the tip token contract address.
    /// MUST emit the 'ApprovalForNFT' event to reflect approval or not approval
    /// @param holders The holders of the NFT (NFT controllers)
    /// @param nft The NFT contract address
    /// @param id The NFT token id
    /// @param approved True if the 'holder' is approved, false to revoke approval
    function setApprovalForNFT(
        address[] memory holders,
        address nft,
        uint256 id,
        bool approved
    ) external;

    /// @notice Checks if 'holder' and 'nft' with token 'id' have been approved
    /// by setApprovalForNFT
    /// @dev This does not check that the holder of the NFT has changed. That is
    /// left to the implementer to detect events for change of ownership and to
    /// take appropriate action
    /// @param holder The holder of the NFT (NFT controller)
    /// @param nft The NFT contract address
    /// @param id The NFT token id
    /// @return True if 'holder' and 'nft' with token 'id' have previously been
    /// approved by the tip token contract
    function isApprovalForNFT(
        address holder,
        address nft,
        uint256 id
    ) external returns (bool);

    /// @notice Sends tip from msg.sender to holder of a single NFT or
    /// to shared holders of a multi token
    /// @dev If 'nft' has not been approved for tipping, MUST revert
    /// MUST revert if 'nft' is zero address.
    /// MUST burn the tip 'amount' to the 'holder' and send the reward to
    /// an account pending for the holder(s).
    /// If 'nft' is a multi token that has multiple holders then each holder
    /// MUST receive tip amount in proportion of their balance of multi tokens
    /// MUST emit the 'Tip' event to reflect the amounts that msg.sender tipped
    /// to holder(s) of 'nft'.
    /// @param nft The NFT contract address
    /// @param id The NFT token id
    /// @param amount Amount of tip tokens to send to the holder of the NFT
    function tip(
        address nft,
        uint256 id,
        uint256 amount
    ) external;

    /// @notice Sends a batch of tips to holders of 'nfts' for gas efficiency
    /// @dev If NFT has not been approved for tipping, revert
    /// MUST revert if the input arguments lengths are not all the same
    /// MUST revert if any of the user addresses are zero
    /// MUST revert the whole batch if there are any errors
    /// MUST emit the 'Tip' events so that the state of the amounts sent to
    /// each holder and for which nft and from whom, can be reconstructed.
    /// @param users User accounts to tip from
    /// @param nfts The NFT contract addresses whose holders to tip to
    /// @param ids The NFT token ids that uniquely identifies the 'nfts'
    /// @param amounts Amount of tip tokens to send to the holders of the NFTs
    function tipBatch(
        address[] memory users,
        address[] memory nfts,
        uint256[] memory ids,
        uint256[] memory amounts
    ) external;

    /// @notice Deposit an ERC20 compatible token in exchange for tip tokens
    /// @dev The price of tip tokens can be different for each deposit as
    /// the amount of reward token sent ultimately is a ratio of the
    /// amount of tip tokens to tip over the user's tip tokens balance available
    /// multiplied by the user's deposit balance.
    /// The deposited tokens can be held in the tip tokens contract account or
    /// in an external escrow. This will depend on the tip token implementation.
    /// Each tip token contract MUST handle only one type of ERC20 compatible
    /// reward for deposits.
    /// This token address SHOULD be passed in to the tip token constructor or
    /// initialize method. SHOULD revert if ERC20 reward for deposits is
    /// zero address.
    /// MUST emit the 'Deposit' event that shows the user, deposited token details
    /// and amount of tip tokens minted in exchange
    /// @param user The user account
    /// @param amount Amount of ERC20 token to deposit in exchange for tip tokens.
    /// This deposit is to be used later as the reward token
    function deposit(address user, uint256 amount) external payable;

    /// @notice An NFT holder can withdraw their tips as an ERC20 compatible
    /// reward at a time of their choosing
    /// @dev MUST revert if not enough balance pending available to withdraw.
    /// MUST send 'amount' to msg.sender account (the holder)
    /// MUST reduce the balance of reward tokens pending by the 'amount' withdrawn.
    /// MUST emit the 'Withdraw' event to show the holder who withdrew, the reward
    /// token address and 'amount'
    /// @param amount Amount of ERC20 token to withdraw as a reward
    function withdraw(uint256 amount) external payable;

    /// @notice MUST have idential behaviour to ERC20 balanceOf and is the amount
    /// of tip tokens held by 'user'
    /// @param user The user account
    /// @return The balance of tip tokens held by user
    function balanceOf(address user) external view returns (uint256);

    /// @notice The balance of deposit available to become rewards when
    /// user sends the tips
    /// @param user The user account
    /// @return The remaining balance of the ERC20 compatible token deposited
    function balanceDepositOf(address user) external view returns (uint256);

    /// @notice The amount of reward token owed to 'holder'
    /// @dev The pending tokens can come from the tip token contract account
    /// or from an external escrow, depending on tip token implementation
    /// @param holder The holder of NFT(s) (NFT controller)
    /// @return The amount of reward tokens owed to the holder from tipping
    function rewardPendingOf(address holder) external view returns (uint256);
}
```

### Tip Token transfer and value calculations

Tip token values are exchanged with ERC-20 deposits and vice-versa. It is left to the tip token implementer to decide on the price of a tip token and hence how much tip to mint in exchange for the ERC-20 deposited. One possibility is to have fixed conversion rates per geographical region so that users from poorer countries are able to send the same number of tips as those from richer nations for the same level of appreciation for content/assets etc. Hence, not skewed by average wealth when it comes to analytics to discover what NFTs are actually popular, allowing creators to have a level playing field.

Whenever a user sends a tip, an equivalent value of deposited ERC20 MUST be transferred to a pending account for the NFT or multi-token holder, and the tip tokens sent MUST be burnt. This equivalent value is calculated using a simple formula:

_total user balance of ERC20 deposit _ tip amount / total user balance of tip tokens\*

Thus adding \*free\* tips to a user's balance of tips for example, simply dilutes the overall value of each tip for that user, as collectively they still refer to the same amount of ERC20 deposited.

Note if the tip token contract inherits from an ERC20, tips can be transferred from one user to another directly. The deposit amount would be already in the tip token contract (or an external escrow account) so only tip token contract's internal mapping of user account to deposit balances needs to be updated. It is RECOMMENDED that the tip amount be burnt from user A and then minted back to user B in the amount that keeps user B's average ERC20 deposited value per tip the same, so that the value of the tip does not fluctuate in the process of tipping.

If not inheriting from ERC20, then minting the tip tokens MUST emit event Transfer(address indexed from, address indexed to, uint256 value) where sender is the zero address for a mint and to is the zero address for a burn. The Transfer event MUST be the same signature as the Transfer function in the IERC20 interface.

### Royalty distribution to shared holders

ERC-1155 allows for shared holders of a token id. Imagine a scenario where an article represented by an NFT was written by multiple contributors. Here, each contributor is a holder and the fractional sharing percentage between them can be represented by the balance that each holds in the ERC-1155 token id. So for two holders A and B of ERC-1155 token 1, if holder A's balance is 25 and holder B's is 75 then any tip sent to token 1 would distribute 25% of the reward pending to holder A and the remaining 75% pending to holder B.

### Caveats

To keep the ITipToken interface simple and general purpose, each tip token contract MUST use one ERC20 compatible deposit type at a time. If tipping is required to support many ERC20 deposits then each tip token contract MUST be deployed separately per ERC20 compatible type required. Thus, if tipping is required from both ETH and BTC wrapper ERC20 deposits then the tip token contract is deployed twice. The tip token contract's constructor is REQUIRED to pass in the address of the ERC20 token supported for the deposits for the particular tip token contract. Or in the case for upgradeable tip token contracts, an initialize method is REQUIRED to pass in the ERC20 token address.

This EIP does not provide details for where the ERC20 reward deposits are held. It MUST be available at the time a holder withdraws the rewards that they are owed. A RECOMMENDED implementation would be to keep the deposits locked in the tip token contract address. By keeping a mapping structure that records the balances pending to holders then the
deposits can remain where they are when a user tips, and only transferred out to a holder's address when a holder withdraws it as their reward.

This standard does not specify the type of ERC20 compatible deposits allowed. Indeed, could be tip tokens themselves. But it is RECOMMENDED that balances of the deposits be checked after transfer to find out the exact amount deposited to keep internal accounting consistent. In case, for example, the ERC20 contract takes fees and hence reduces the actual amount deposited.

### Minimising Gas Costs

By caching tips off-chain and then batching them up to call the tipBatch method of the ITipToken interface then essentially the cost of initialising transactions is paid once rather than once per tip. Plus, further gas savings can be made off-chain if multiple tips sent by the same user to the same NFT token are accumulated together and sent as one entry in the batch.

Further savings can be made by grouping users together sending to the same NFT, so that checking the validity of the NFT and whether it is an ERC721 or ERC1155, is performed once for each group.

Clever ways to minimise on-chain state updating of the deposit balances for each user and the reward balances of each holder, can help further to minimise the gas costs when sending in a batch if the batch is ordered beforehand. For example, can avoid the checks if the next NFT in the batch is the same. This left to the tip token contract implementer. Whatever optimisation is applied, it MUST still allow information of which account tipped which account and for what NFT to be reconstructed from the Tip and the TipBatch events emitted.

## Rationale

### Simplicity

ITipToken interface uses a minimal number of functions, in order to keep its use as general purpose as possible, whilst there being enough to guide implementation that fulfils its purpose for micropayments to NFT holders.

### Use of NFTs

Each NFT is a unique non-fungible token digital asset stored on the blockchain that are uniquely identified by its address and token id. It's truth burnt using cryptographic hashing on a secure blockchain means that it serves as an anchor for linking with a unique digital asset, service or other contractual agreement. Such use cases may include (but only really limited by imagination and acceptance):

- Digital art, collectibles, music, video, licenses and certificates, event tickets, ENS names, gaming items, objects in metaverses, proof of authenticity of physical items, service agreements etc.

This mechanism allows consumers of the NFT a secure way to easily tip and reward the NFT holder.

### New Business Models

To take the music use case for example. Traditionally since the industry transitioned from audio distributed on physical medium such as CDs, to an online digital distribution model via streaming, the music industry has been controlled by oligopolies that served to help in the transition. They operate a fixed subscription model and from that they set the amount of royalty distribution to content creators; such as the singers, musicians etc. Using tip tokens represent an additional way for fans of music to reward the content creators. Each song or track is represented by an NFT and fans are able to tip the song (hence the NFT) that they like, and in turn the content creators of the NFT are able to receive the ERC20 rewards that the tips were bought for. A fan led music industry with decentralisation and tokenisation is expected to bring new revenue, and bring fans and content creators closer together.

Across the board in other industries a similar ethos can be applied where third party controllers move to a more facilitating role rather than a monetary controlling role that exists today.

### Guaranteed audit trail

As the Ethereum ecosystem continues to grow, many dapps are relying on traditional databases and explorer API services to retrieve and categorize data. The ERC-xxxx standard guarantees that event logs emitted by the smart contract MUST provide enough data to create an accurate record of all current tip token and ERC20 reward balances. A database or explorer can provide indexed and categorized searches of every tip token and reward sent to NFT holders from the events emitted by any tip token contract that implements this standard. Thus, the state of the tip token contract can be reconstructed from the events emitted alone.

## Backwards Compatibility

A tip token contract can be fully compatible with ERC20 specification and inherit some functions such as transfer if the tokens are allowed to be sent directly to other users. Note that balanceOf has been adopted and MUST be the number of tips held by a user's address. If inheriting from, for example, OpenZeppelin's implementation of ERC20 token then their contract is responsible for maintaining the balance of tip token. Therefore, tip token balanceOf function SHOULD simply directly call the parent (super) contract's balanceOf function.

What hasn't been carried over to tip token standard, is the ability for a spender of other users' tips. For the moment, this standard does not foresee a need for this.

This EIP does not stress a need for tip token secondary markets or other use cases where identifying the tip token type with names rather than addresses might be useful, so these functions were left out of the ITipToken interface and is the remit for implementers.

## Reference Implementation

A [reference/example implementation and tests](https://github.com/julesl23/eip_msnmt) in the GitHub repository that can use be used to assist in implementing this specification.

## Security Considerations

Though it is RECOMMENDED that users' deposits are kept locked in the tip token contract or external escrow account, and SHOULD NOT be used for anything but the rewards for holders, this cannot be enforced. This standard stipulates that the rewards MUST be available for when holders withdraw their rewards from the pool of deposits.

Before any users can tip an NFT, the holder of the NFT has to give their approval for tipping from the tip token contract. This standard stipulates that holders of the NFTs receive the rewards. It SHOULD be clear in the tip token contract code that it does so, without obfuscation to where the rewards go. Any fee charges SHOULD be made obvious to users before acceptance of their deposit. There is a risk that rogue implementers may attempt to \*hijack\* potential tip income streams for their own purposes. But additionally the number and frequency of transactions of the tipping process should make this type of fraud quicker to be found out.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
