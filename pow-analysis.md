##eth POW analysis

### Consensus engine description

In the CPU mining part, the CpuAgent mine function calls the self.engine.Seal function when performing the mining operation. The engine here is the consensus engine. Seal is one of the most important interfaces. It implements the search for nonce values ​​and the calculation of hashes. And this function is an important function that guarantees consensus and cannot be forged. In the PoW consensus algorithm, the Seal function implements proof of work. This part of the source code is under consensus/ethhash.

### Consensus engine interface

```go
type Engine interface {
	// Get the block digger, ie coinbase
	Author(header *types.Header) (common.Address, error)


	// VerifyHeader is used to check the block header and check it by consensus rules. The verification block can be used here. Ketong passes the VerifySeal method.
	VerifyHeader(chain ChainReader, header *types.Header, seal bool) error


	// VerifyHeaders is similar to VerifyHeader, and this is used to batch checkpoints. This method returns an exit signal
	// Used to terminate the operation for asynchronous verification.
	VerifyHeaders(chain ChainReader, headers []*types.Header, seals []bool) (chan<- struct{}, <-chan error)

	// VerifyUncles Rules for verifying a bad block to conform to the consensus engine
	VerifyUncles(chain ChainReader, block *types.Block) error

	// VerifySeal Check the block header according to the rules of the consensus algorithm
	VerifySeal(chain ChainReader, header *types.Header) error

	// Prepare The consensus field used to initialize the block header is based on the consensus engine. These changes are all implemented inline.
	Prepare(chain ChainReader, header *types.Header) error

	// Finalize Complete all state modifications and eventually assemble them into blocks.
	// The block header and state database can be updated to conform to the consensus rules at the time of final validation.
	Finalize(chain ChainReader, header *types.Header, state *state.StateDB, txs []*types.Transaction,
		uncles []*types.Header, receipts []*types.Receipt) (*types.Block, error)

	// Seal Production of a new block based on the input block
	Seal(chain ChainReader, block *types.Block, stop <-chan struct{}) (*types.Block, error)

	// CalcDifficulty It is the difficulty adjustment algorithm that returns the difficulty value of the new block.
	CalcDifficulty(chain ChainReader, time uint64, parent *types.Header) *big.Int

	// APIs Return the RPC provided by the consensus engine APIs
	APIs(chain ChainReader) []rpc.API
}
```

### ethhash implementation

#### ethhash structure

```go
type Ethash struct {
	config Config

	caches   *lru // In memory caches to avoid regenerating too often
	datasets *lru // In memory datasets to avoid regenerating too often

	// Mining related fields
	rand     *rand.Rand    // Properly seeded random source for nonces
	threads  int           // Number of threads to mine on if mining
	// channel Used to update mining notices
	update   chan struct{} // Notification channel to update mining parameters
	hashrate metrics.Meter // Meter tracking the average hashrate

	// Test network related parameters
	// The fields below are hooks for testing
	shared    *Ethash       // Shared PoW verifier to avoid cache regeneration
	fakeFail  uint64        // Block number which fails PoW check even in fake mode
	fakeDelay time.Duration // Time delay to sleep for before returning from verify

	lock sync.Mutex // Ensures thread safety for the in-memory caches and mining fields
}
```

Ethhash is a concrete implementation of PoW. Since there are a large number of datasets to be used, there are two pointers to lru. And control the number of mining threads through threads. And in test mode or fake mode, simple and fast processing, so that it can get results quickly.

The Athor method obtained the miner's address from which the block was dug.

```go
func (ethash *Ethash) Author(header *types.Header) (common.Address, error) {
	return header.Coinbase, nil
}
```

VerifyHeader is used to verify that the block header information conforms to the ethash consensus engine rules.

```go
// VerifyHeader checks whether a header conforms to the consensus rules of the
// stock Ethereum ethash engine.
func (ethash *Ethash) VerifyHeader(chain consensus.ChainReader, header *types.Header, seal bool) error {
	// When in ModeFullFake mode, any header information is accepted
	if ethash.config.PowMode == ModeFullFake {
		return nil
	}
	// If the header is known, it does not need to be verified and returns direc