
- [High](High-https:--github.com-sherlock-audit-2023-02-openq-judging-issues-220/README.md) - 0xbepresent - The ```DepositManagerV1.sol::fundBountyToken()``` must accept only whitelisted tokens.

- [High](High-https:--github.com-sherlock-audit-2023-02-openq-judging-issues-296/README.md) - 0xbepresent - User claim is compromised if the deposited NFT is refunded by the funder.

- [High](High-https:--github.com-sherlock-audit-2023-02-openq-judging-issues-383/README.md) - 0xbepresent - ```fundingTotals``` is not updated when funder withdraw his funding in the ```TieredPercentage``` bounty.

- [Medium](Medium-https:--github.com-sherlock-audit-2023-02-openq-judging-issues-153/README.md) - 0xbepresent - ```tokenAddresses``` count is not decreased on refunds causing a limitation in deposits.

- [High](High-https:--github.com-sherlock-audit-2023-02-openq-judging-issues-309/README.md) - 0xbepresent - The first assigned winner can close the competition via ```ClaimManagerV1.sol::permissionedClaimTieredBounty()``` even when the other winners are not assigned yet.
