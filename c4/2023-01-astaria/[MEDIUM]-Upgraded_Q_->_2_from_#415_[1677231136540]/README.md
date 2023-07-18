# Original link
https://github.com/code-423n4/2023-01-astaria-findings/issues/635
Judge has assessed an item in Issue #415 as 2 risk. The relevant finding follows:

 2 - The PrivateVault deposit can be executed even when the vault was paused.
There is not restriction for deposit in the Private vault.

Recommendation
Add a whenNotPaused() modifier for the deposit() function.