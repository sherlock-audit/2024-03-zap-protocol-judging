Proper Cerulean Gorilla

high

# TokenSale implementation contract can be destroyed, making all proxy instances unusable

## Summary
The "destroy" function on the TokenSale implementation contract can be called by anyone. This executes a "selfdestruct" on the contract. This makes the implementation contract unusable by the proxies pointing to it, risking user assets and the protocol functionality.

## Vulnerability Detail
The TokenSale contract can selfdestruct by calling the "destroy" function. It is supposed to be only callable from the "admin" wallet. When the implementation contract is deployed, proxies are supposed to point to it, and they will hold the funds and storage. 
However, a malicious actor can call "initialize" on the **implementation** contract and initialize it with his wallet as admin. This allows the attacker to call "destroy" on the implementation, causing it to selfdestruct. This will make all deployed proxy instances of TokenSale unusable since the implementation code doesnt exists anymore.

## Impact
Loss of user funds if there are TokenSale instances with funds. 

## Code Snippet
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L186-L194
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/TokenSale.sol#L71-L94

deployment of proxy:
https://github.com/sherlock-audit/2024-03-zap-protocol/blob/main/zap-contracts-labs/contracts/Admin.sol#L303-L313

## Tool used

Manual Review

## Recommendation

Instead of using selfdestruct to halt contract activity. Is recommended to just freeze the contract adding flags. 
Example: add a boolean on the state that will be turned into true when we want to freeze the contract. Every function should check for this flag before execution.
