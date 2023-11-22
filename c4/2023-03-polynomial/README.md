# Findings for 2023-03-polynomial 

- [[HIGH]]([HIGH]-1630553436/README.md) - Fees are not increased to the ```totalFunds``` variable in the ```openShort``` operation consequently those fees are lost
- [[MEDIUM]]([MEDIUM]-1630623024/README.md) - The protocol may run out of funds due to a malfunction in ```processDeposits()``` function, causing the protocol may not cover their margins
- [[MEDIUM]]([MEDIUM]-1632116863/README.md) - Attacker can flood the ```withdrawalQueue``` variable with zero withdraw amounts causing that the ```queueWithdraw()``` function cost more gas to process
