## Ethereum's Bloom filter

The block header of Ethereum contains an area called logsBloom. This area stores the Bloom filter for all the receipts in the current block, for a total of 2048 bits. That is 256 bytes.

And the receipt of one of our transactions contains a lot of log records. Each log record contains the address of the contract, multiple Topics. There is also a Bloom filter in our receipt, which records all the lo