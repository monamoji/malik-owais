![image](picture/sign_state_1.png)

![image](picture/sign_state_3.png) It is the state of t+1 (account trie).

![image](picture/sign_state_4.png) It is a state transition function, which can also be understood as an execution engine.

![image](picture/sign_state_5.png) is a transaction

![image](picture/sign_state_6.png)

![image](picture/sign_state_7.png) Is a state transition function at the block level.

![image](picture/sign_state_8.png) It is a block and consists of many transactions.

![image](picture/sign_state_9.png) Transaction at position 0.

![image](picture/sign_state_10.png) Is the block termination state transition function (a function that rewards the miner).

![image](picture/sign_ether.png) Ether logo

![image](picture/sign_ether_value.png) The conversion relationship between the various units used in Ethereum and Wei (for example: a Finney corresponds to 10^15 Wei).

![image](picture/sign_machine_state.png) machine-state

## Some basic rules

- For most functions, they are identified by uppercase letters.
- Tuples are generally identified by capital letters
- A scalar or fixed-size array of bytes is identified by a lowercase letter. For example, n represents the nonce of the transaction, and there may be some 