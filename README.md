# What is ENS v2 and How it is different from ENSV1?

## What is ENS v1?

ENS (Ethereum Name Service) launched in 2017 as Ethereum's naming system, turning complex addresses like `0x1234...` into human-readable names like `alice.eth`.

### Core Components

ENS v1 is built on three core components:

1. **Registry**: A single contract on Ethereum mainnet mapping every name to its resolver
2. **Resolvers**: Contracts that store the actual data for a name (ETH address, content hash, text records, etc.)
3. **Registrars**: Contracts that allocate names to users (like the `.eth` registrar that manages all `.eth` name registrations)

### Resolution Process
These components work together in a two-step resolution process:

1. Query the registry to find the resolver assigned to a name
2. Query that resolver to fetch the actual record (e.g., ETH address, content hash)

This architecture has been reliable and battle-tested for years. However, it was designed when Ethereum existed only on L1, and today's ecosystem looks very different.

<img width="1528" height="1161" alt="HOW ENSV1 WORKS?" src="https://github.com/user-attachments/assets/f9857717-6f27-487a-b526-2d94be9365f7" />


## The Evolution: Why v2?

Since ENS launched in 2017, the Ethereum ecosystem has evolved significantly. One major shift has been the rise of Layer 2 blockchains, offering lower transaction fees and better scalability.

ENS adapted with CCIP-Read (a protocol that allows resolvers to fetch verified data from off-chain sources/L2), enabling projects like `cb.id`, `uni.eth`, and `dao.eth` to delegate resolution to L2s. However, the `.eth` registry (the contract managing all `.eth` name registrations) remained on L1. Every registration, renewal, and ownership update still requires an L1 transaction with L1 gas fees.

### ENS v1 Limitations

| Problem | What it Means | Example |
|---------|---------------|---------|
| **Expensive mainnet operations** | Registration and renewal happen on Ethereum L1 where gas fees can be high | Registering alice.eth requires paying L1 transaction fees |
| **L2 setup still needs L1** | Even if you want to manage your name on L2, you must make one L1 transaction to delegate | Want to use Optimism for your name? Still need one expensive L1 tx to set resolver |
| **Wrapper contract overhead** | Locking names or adding permissions requires additional Name Wrapper contract | Want to lock alice.eth permanently? Need to wrap it in separate contract |
| **Expired names don't disappear** | Registry has no expiration concept; expired names stay visible until re-registered | alice.eth expires Dec 2024 but still shows Alice as owner until someone registers it |
| **No bulk subdomain operations** | Flat structure means updating many subdomains requires individual transactions | Delete 50 subdomains = 50 separate transactions |
| **Shared resolver leaves stale data** | Single Public Resolver means old owner's records may persist after transfer | alice.eth transfers to Bob, but Alice's email still appears |

## What is ENS v2?

ENS v2 is a fundamental architectural redesign that moves the `.eth` registry to Layer 2 and restructures how names are organized and managed.

### The Two Biggest Changes

1. **Location**: The authoritative `.eth` registry moves from Ethereum mainnet to Layer 2, significantly reducing transaction costs
2. **Architecture**: Instead of one flat registry for all names, ENS v2 uses hierarchical registries where each name has its own registry contract

## Understanding Core Component Changes

### 1. ENS Registry Architecture: v1 vs v2

#### v1 = Single Flat Registry

ENS v1 uses one global contract to store every name in a flat mapping. Think of it as a single giant database table where `alice.eth` and `bob.alice.eth` are just separate entries with no real parent-child relationship.

This simplicity comes with limits: no per-name logic, no native expiration, and no way to manage subtrees atomically.

<img width="1082" height="824" alt="SingeRegistry" src="https://github.com/user-attachments/assets/78a5fd15-a9c2-45f2-8e36-d73f048d13da" />

#### v2 = Hierarchical Tree of Registries

ENS v2 restructures the system: each name becomes its own registry contract, responsible only for its direct children.

- The root registry manages TLDs
- `.eth` manages all 2LDs
- `alice.eth` manages all `*.alice.eth` names

Each level is responsible only for the names directly beneath it, forming a clean, modular tree instead of one giant table.


<img width="568" height="734" alt="Screenshot 2026-01-07 at 3 43 57 PM" src="https://github.com/user-attachments/assets/ba79f85e-c159-47d0-96f0-773ef1faea19" />

### Registries as ERC-1155 NFT Collections

Every v2 registry implements the ERC-1155 token standard, making each registry a native NFT collection. This means that subdomains are represented as NFTs automatically. When you own `alice.eth`, you get your own registry contract for it. This registry is an ERC-1155 NFT collection, and each subdomain you create becomes a token in that collection.

**Example:**

The `alice.eth` registry is an NFT collection containing:

```
alice.eth Registry (ERC-1155 Collection)
├─ wallet.alice.eth = Token ID #1
├─ nft.alice.eth = Token ID #2
└─ web.alice.eth = Token ID #3
```

**Owning the token = owning the subdomain** - The NFT ownership IS the subdomain ownership. This is a major improvement over v1, which required the Name Wrapper contract to achieve NFT functionality. In v2, it's built into the core architecture.

### Technical Architecture of Registries

#### The L2 Problem

On L2, you need cryptographic proofs. More contracts = more proofs = expensive. ENS v2 puts all resolution data in one shared contract—1 proof instead of many.

#### The Two-Contract System

1. **Registry contracts** (one per name): Handle ownership, transfers, custom rules
2. **RegistryDatastore** (one shared): Stores resolution data for everyone

When `alice.eth` registry is asked "what's wallet's resolver?", it looks up the answer in the Datastore.

#### 1. IRegistry Interface

Every registry must implement this standard interface:

```solidity
interface IRegistry is IERC1155 {
    event NewSubname(string label);
    
    function getSubregistry(string calldata label) external view returns (address);
    function getResolver(string calldata label) external view returns (address);
}
```

#### 2. RegistryDatastore

A single shared contract that stores ALL resolver and subregistry addresses for ALL registries.

```solidity
interface IRegistryDatastore {
    function getSubregistry(address registry, uint256 tokenId) external view returns (address);
    function getResolver(address registry, uint256 tokenId) external view returns (address);
    function setSubregistry(uint256 tokenId, address subregistry) external;
    function setResolver(uint256 tokenId, address resolver, uint64 flags) external;
}
```

**Storage split:**
- **Datastore stores**: Subregistry addresses, resolver addresses, flags (for custom data like expiration timestamps)

- **Registry contract stores**: NFT ownership, custom logic (transfer rules, access controls)

### What This Design Unlocks

**Native Expiration**: Expired names actually disappear from the registry immediately. In v1, `alice.eth` would stay visible after expiring until someone re-registered it. In v2, it vanishes instantly, no stale data.

**Bulk Operations**: Update entire subtrees at once. Want to delete 100 subdomains under `alice.eth`? Update the `alice.eth` registry once instead of 100 individual transactions. When names expire or transfer, everything underneath resets automatically.

**Custom Logic Per Name**: Each registry is its own contract with custom rules. Deploy a registry with time-locked transfers, DAO governance for subdomains, or custom expiration logic, while staying compatible with ENS. Standard implementations exist for most users, but advanced customization is possible.

**Built-in NFT Support**: Every registry is ERC-1155, so subdomains are NFTs automatically. No wrapper contract needed. OpenSea, wallets, and NFT infrastructure recognize ENS subdomains natively, transfers and marketplaces just work.


## Resolution: Finding and Querying the Right Resolver

### The Components

**Resolvers**: Unchanged from v1. A resolver stores your name's data, ETH address, avatar, Twitter handle. When someone looks up your name, they query your resolver.

**Universal Resolver**: In v1, every app had to implement resolution logic themselves, find the registry, find the resolver, handle edge cases. v2 introduces the Universal Resolver: one contract that handles all resolution logic. Developers call one function, it works for v1 names, v2 names on L1, and v2 names on L2 automatically. Future upgrades only need to update the Universal Resolver, apps benefit without code changes.

<img width="2523" height="1607" alt="UniversalResolver" src="https://github.com/user-attachments/assets/5ebcfc9b-b37d-46fd-afc3-18d6d64c54c0" />


### How Resolution Actually Works

Resolving `foo.2ld.eth`:

1. **Root Registry** → "Where's .eth?" → Returns .eth Registry address
2. **.eth Registry** → "Where's 2ld?" → No registry, but returns resolver: `0xResolver1`
3. **Use closest resolver** → `0xResolver1` from step 2
4. **Query resolver** → "What's the ETH address for foo.2ld.eth?" → Returns address

Since `foo` doesn't have its own registry, the parent's resolver (`2ld.eth`) handles it, this is wildcard resolution.

### Why This Works: Wildcard Resolution

Notice that `foo.2ld.eth` doesn't actually exist as a separate entry, there's no registry for it. But the resolver for `2ld.eth` can still answer questions about it. This is called wildcard resolution: a parent's resolver can handle all its subdomains, even ones that don't explicitly exist.

**L2 Resolution**: When names live on L2, this same process happens using CCIP-Read, which uses cryptographic proofs to fetch data from L2 securely.

## Moving Forward with ENS v2

ENS v2 moves `.eth` to L2 for cheaper operations while letting owners "eject" names to L1 for maximum security, names can move between chains anytime. Unmigrated v1 names keep working automatically via the Fallback Resolver, and migration is optional (choose L2 for cost or L1 for guarantees). 

For developers, use the Universal Resolver as your single entry point, one function resolves any name across v1, v2, L1, and L2. The hierarchical architecture enables custom registries with specialized logic while resolver interfaces remain unchanged, so your v1 knowledge transfers directly.

---

## Resources

- [ENS Documentation](https://docs.ens.domains)
- [ENS GitHub](https://github.com/ensdomains)
