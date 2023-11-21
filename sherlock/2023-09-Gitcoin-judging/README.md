
- [High](High-1907963223/README.md) - 0xbepresent - Malicious registrant can front-run `RFPSimpleStrategy._allocate()` in order to change the `proposalBid` and get a bigger payout in the distribution

- [Medium](Medium-1907696038/README.md) - 0xbepresent - The `RFPSimpleStrategy._registerRecipient()` does not work when the strategy was created using the `useRegistryAnchor=true` causing that nobody can register to the pool

- [Medium](Medium-1908007607/README.md) - 0xbepresent - Pool's strategies does not support `fee on transfer` tokens causing an error in the counting system

- [Medium](Medium-1908006447/README.md) - 0xbepresent - Error in counting the `allocator.voiceCreditsCastToRecipient` causing the `recipient` to have more votes and get the majority of the pool

- [High](High-1907859923/README.md) - 0xbepresent - The `QVSimpleStrategy.maxVoiceCreditsPerAllocator` can be evaded by the allocator causing that he can allocate infinite credits to the same recipient
