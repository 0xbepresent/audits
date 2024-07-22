# Findings for 2023-10-wildcat 

- [[HIGH]]([HIGH]-1964030350/README.md) - When borrower closes the market, he won't pay all incurring debt while he was delinquency
- [[HIGH]]([HIGH]-1959945543/README.md) - Lender, who is going to be marked as `blocked` (sanctioned), can transfer his assets to another collude lender avoiding the funds to be sent to the escrow contract
- [[HIGH]]([HIGH]-1957919865/README.md) - Once the sanctioned lender is being unsanctioned, the lender assets are transferred to the borrower instead of the lender, causing the lost of lender's assets
- [[QA]]([QA]-1964275855/README.md) - The `reserveRatioBips` can be reset to a non zero value once the market was closed.
- [[MEDIUM]]([MEDIUM]-1960033844/README.md) - The `MinimumAnnualInterestBips` is not validated when the `WildcatMarketController::setAnnualInterestBips()` is called
