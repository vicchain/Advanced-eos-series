# Optimizing the RAM usage

Each smart contract is tied to a corresponding EOS account that the smart contract is deployed to. This EOS account needs a specific amount of RAM to hold the smart contract’s (webassembly) code and to store the database entries. There’s only a limited amount of RAM available (64GB) and it’s a free market. Meaning, the price is driven by supply and demand and has lately become interesting for speculators that drove the price up a lot.

Back then, buying 1 KB of RAM costs 0.32 EOS. The latest price can be [checked on eos.feexplorer.io](https://eos.feexplorer.io/). This means **the size of the contract and its database** will directly determine how much developers need to pay for smart contracts.

**How much RAM does an EOS smart contract need?**

Estimating the amount of **RAM needed for the database** is pretty straightforward. It’s proportional to the amount of bytes of your serialized data entries which can be easily computed. In my cases, this is the C++ struct I store for each new throne claim:
```
struct claim_record
{
    // upper 56 bits contain kingdom order, lower 8 bits contain kingOrder
    uint64_t kingdomKingIndex; // this also acts as key of the table
    time claimTime; // is 64 bit
    account_name name; // typedef for uint64_t
    std::string displayName; // restricted in my code to max 100 chars
    std::string image; // restricted in my code to 32 chars

    uint64_t primary_key() const { return kingdomKingIndex; }
    EOSLIB_SERIALIZE(claim_record, ...)
};
```
We can just sum up the bytes needed for each field:
```
8bytes + 8bytes + 8bytes + 100bytes + 32bytes (+114 bytes unknown overhead) = 270bytes
```
Optimizing *structs* is hard, but one thing I did was to encode both the current round of the game and the king index in the upper and lower bits of a single *uint64_t*.

Estimating the amount of **RAM an EOS smart contract requires** is a lot harder and I could only ever check it after I deployed it to the blockchain.

Here are some things I noticed and that will help you reduce the number of RAM your EOS contract needs:

* The web assembly compiler is smart enough to exclude header files and function definitions that you do not use in your contract. So it comes with [Dead Code Elimination](https://en.wikipedia.org/wiki/Dead_code_elimination). There’s no need to comment out unused header files.
* **Not using *std::to_string* saves you 188.3 KiB** (back then 71 EOS / 500 $). Initially, I used it to print a number in an *assert* message. Removing it saved me a lot of RAM.
* Marking functions as *inline* will usually also shave off some of your contract’s size.
* I checked som other third-party functions that I could optimize and ended up replacing  *boost::split’s* string splitting function with my own implementation. It reduced the RAM requirements by another 20 KiB.

Resulting in a **smart contract that needs 200 KiB** to be deployed (excluding the dynamic amount of RAM required for storing the entries). Using the [EOS resource planner](https://www.eosrp.io/#calc) I calculated I need to pay 70 EOS for 220 KiB of RAM. This was about 500$ for a really simple, optimized smart contract. If I didn’t do the optimizations I would have paid double that.

> Update: The RAM price halfed in the last month. So it would be around 250$ now - but the high volatility of RAM price makes it hard to plan.

Comparing this to Ethereum, it’s a lot what developers have to pay and this might hold new developers back from developing on EOS. For indie developers making small fun projects to get started, it’s just too much.