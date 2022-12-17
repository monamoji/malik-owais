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
	// If the header is known, it does not need to be verified and returns directly.
	number := header.Number.Uint64()
	if chain.GetHeader(header.Hash(), number) != nil {
		return nil
	}
	parent := chain.GetHeader(header.ParentHash, number-1)
	if parent == nil {  // Failed to get parent node
		return consensus.ErrUnknownAncestor
	}
	// Further header verification
	return ethash.verifyHeader(chain, header, parent, false, seal)
}
```

Then look at the implementation of verifyHeader,

```go
func (ethash *Ethash) verifyHeader(chain consensus.ChainReader, header, parent *types.Header, uncle bool, seal bool) error {
	// Ensure that the extra data segment has a reasonable length
	if uint64(len(header.Extra)) > params.MaximumExtraDataSize {
		return fmt.Errorf("extra-data too long: %d > %d", len(header.Extra), params.MaximumExtraDataSize)
	}
	// Check timestamp
	if uncle {
		if header.Time.Cmp(math.MaxBig256) > 0 {
			return errLargeBlockTime
		}
	} else {
		if header.Time.Cmp(big.NewInt(time.Now().Add(allowedFutureBlockTime).Unix())) > 0 {
			return consensus.ErrFutureBlock
		}
	}
	if header.Time.Cmp(parent.Time) <= 0 {
		return errZeroBlockTime
	}
	// The difficulty of the block is checked based on the timestamp and the difficulty of the parent block.
	expected := ethash.CalcDifficulty(chain, header.Time.Uint64(), parent)

	if expected.Cmp(header.Difficulty) != 0 {
		return fmt.Errorf("invalid difficulty: have %v, want %v", header.Difficulty, expected)
	}
	// check gas limit <= 2^63-1
	cap := uint64(0x7fffffffffffffff)
	if header.GasLimit > cap {
		return fmt.Errorf("invalid gasLimit: have %v, max %v", header.GasLimit, cap)
	}
	// check gasUsed <= gasLimit
	if header.GasUsed > header.GasLimit {
		return fmt.Errorf("invalid gasUsed: have %d, gasLimit %d", header.GasUsed, header.GasLimit)
	}

	// gas limit Is it within the allowable range?
	diff := int64(parent.GasLimit) - int64(header.GasLimit)
	if diff < 0 {
		diff *= -1
	}
	limit := parent.GasLimit / params.GasLimitBoundDivisor

	if uint64(diff) >= limit || header.GasLimit < params.MinGasLimit {
		return fmt.Errorf("invalid gas limit: have %d, want %d += %d", header.GasLimit, parent.GasLimit, limit)
	}
	// The check block number should be the parent block block number +1
	if diff := new(big.Int).Sub(header.Number, parent.Number); diff.Cmp(big.NewInt(1)) != 0 {
		return consensus.ErrInvalidNumber
	}
	// Verify that a particular block meets the requirements
	if seal {
		if err := ethash.VerifySeal(chain, header); err != nil {
			return err
		}
	}
	// If all checks pass, verify the special field of the hard fork.
	if err := misc.VerifyDAOHeaderExtraData(chain.Config(), header); err != nil {
		return err
	}
	if err := misc.VerifyForkHashes(chain.Config(), header, uncle); err != nil {
		return err
	}
	return nil
}
```

Ethash calculates the difficulty of the next block through the CalcDifficulty function, and creates different difficulty calculation methods for the difficulty of different stages.

```go
func (ethash *Ethash) CalcDifficulty(chain consensus.ChainReader, time uint64, parent *types.Header) *big.Int {
	return CalcDifficulty(chain.Config(), time, parent)
}

func CalcDifficulty(config *params.ChainConfig, time uint64, parent *types.Header) *big.Int {
	next := new(big.Int).Add(parent.Number, big1)
	switch {
	case config.IsByzantium(next):
		return calcDifficultyByzantium(time, parent)
	case config.IsHomestead(next):
		return calcDifficultyHomestead(time, parent)
	default:
		return calcDifficultyFrontier(time, parent)
	}
}
```

VerifyHeaders is similar to VerifyHeader except that VerifyHeaders performs bulk check operations. Create multiple goroutines to perform validation operations, and then create a goroutine for assignment control task assignment and result acquisition. Finally return a result channel

```go
func (ethash *Ethash) VerifyHeaders(chain consensus.ChainReader, headers []*types.Header, seals []bool) (chan<- struct{}, <-chan error) {
	// If we're running a full engine faking, accept any input as valid
	if ethash.config.PowMode == ModeFullFake || len(headers) == 0 {
		abort, results := make(chan struct{}), make(chan error, len(headers))
		for i := 0; i < len(headers); i++ {
			results <- nil
		}
		return abort, results
	}

	// Spawn as many workers as allowed threads
	workers := runtime.GOMAXPROCS(0)
	if len(headers) < workers {
		workers = len(headers)
	}

	// Create a task channel and spawn the verifiers
	var (
		inputs = make(chan int)
		done   = make(chan int, workers)
		errors = make([]error, len(headers))
		abort  = make(chan struct{})
	)
	// Generate workers goroutine for check head
	for i := 0; i < workers; i++ {
		go func() {
			for index := range inputs {
				errors[index] = ethash.verifyHeaderWorker(chain, headers, seals, index)
				done <- index
			}
		}()
	}

	errorsOut := make(chan error, len(headers))
	// goroutine Used to send tasks to workers goroutine
	go func() {
		defer close(inputs)
		var (
			in, out = 0, 0
			checked = make([]bool, len(headers))
			inputs  = inputs
		)
		for {
			select {
			case inputs <- in:
				if in++; in == len(headers) {
					// Reached end of headers. Stop sending to workers.
					inputs = nil
				}
			// Statistics and send an error message to errorsOut
			case index := <-done:
				for checked[index] = true; checked[out]; out++ {
					errorsOut <- errors[out]
					if out == len(headers)-1 {
						return
					}
				}
			case <-abort:
				return
			}
		}
	}()
	return abort, errorsOut
}
```

VerifyHeaders uses verifyHeaderWorker when checking a single block header. After the function gets the parent block, it calls verifyHeader to check it.

```go
func (ethash *Ethash) verifyHeaderWorker(chain consensus.ChainReader, headers []*types.Header, seals []bool, index int) error {
	var parent *types.Header
	if index == 0 {
		parent = chain.GetHeader(headers[0].ParentHash, headers[0].Number.Uint64()-1)
	} else if headers[index-1].Hash() == headers[index].ParentHash {
		parent = headers[index-1]
	}
	if parent == nil {
		return consensus.ErrUnknownAncestor
	}
	if chain.GetHeader(headers[index].Hash(), headers[index].Number.Uint64()) != nil {
		return nil // known block
	}
	return ethash.verifyHeader(chain, headers[index], parent, false, seals[index])
}
```

VerifyUncles is used for verification of the unblock. Similar to the check block header, the unchecked block check returns directly in the ModeFullFake mode. Get all the uncle blocks, then traverse the checksum, the checksum will terminate, or the checksum will return.

```go
func (ethash *Ethash) VerifyUncles(chain consensus.ChainReader, block *types.Block) error {
	// If we're running a full engine faking, accept any input as valid
	if ethash.config.PowMode == ModeFullFake {
		return nil
	}
	// Verify that there are at most 2 uncles included in this block
	if len(block.Uncles()) > maxUncles {
		return errTooManyUncles
	}
	// Collecting uncles and their ancestors
	uncles, ancestors := set.New(), make(map[common.Hash]*types.Header)

	number, parent := block.NumberU64()-1, block.ParentHash()
	for i := 0; i < 7; i++ {
		ancestor := chain.GetBlock(parent, number)
		if ancestor == nil {
			break
		}
		ancestors[ancestor.Hash()] = ancestor.Header()
		for _, uncle := range ancestor.Uncles() {
			uncles.Add(uncle.Hash())
		}
		parent, number = ancestor.ParentHash(), number-1
	}
	ancestors[block.Hash()] = block.Header()
	uncles.Add(block.Hash())

	// Verify each of the unblocks
	for _, uncle := range block.Uncles() {
		// Make sure every uncle is rewarded only once
		hash := uncle.Hash()
		if uncles.Has(hash) {
			return errDuplicateUncle
		}
		uncles.Add(hash)

		// Make sure the uncle has a valid ancestry
		if ancestors[hash] != nil {
			return errUncleIsAncestor
		}
		if ancestors[uncle.ParentHash] == nil || uncle.ParentHash == block.ParentHash() {
			return errDanglingUncle
		}
		if err := ethash.verifyHeader(chain, uncle, ancestors[uncle.ParentHash], true, true); err != nil {
			return err
		}
	}
	return nil
}
```

Prepare implements the consensus engine's Prepare interface, which is used to fill the difficulty field of the block header to conform to the ethash protocol. This change is online.

```go
func (ethash *Ethash) Prepare(chain consensus.ChainReader, header *types.Header) error {
	parent := chain.GetHeader(header.ParentHash, header.Number.Uint64()-1)
	if parent == nil {
		return consensus.ErrUnknownAncestor
	}
	header.Difficulty = ethash.CalcDifficulty(chain, header.Time.Uint64(), parent)
	return nil
}
```

Finalize implements the Finalize interface of the consensus engine, rewards digging into block accounts and unblocked accounts, and populates the value of the root of the state tree. And return to the new block.

```go
func (ethash *Ethash) Finalize(chain consensus.ChainReader, header *types.Header, state *state.StateDB, txs []*types.Transaction, uncles []*types.Header, receipts []*types.Receipt) (*types.Block, error) {
	// Accumulate any block and uncle rewards and commit the final state root
	accumulateRewards(chain.Config(), state, header, uncles)
	header.Root = state.IntermediateRoot(chain.Config().IsEIP158(header.Number))

	// Header seems complete, assemble into a block and return
	return types.NewBlock(header, txs, uncles, receipts), nil
}

func accumulateRewards(config *params.ChainConfig, state *state.StateDB, header *types.Header, uncles []*types.Header) {
	// Select the correct block reward based on chain progression
	blockReward := FrontierBlockReward
	if config.IsByzantium(header.Number) {
		blockReward = ByzantiumBlockReward
	}
	// Accumulate the rewards for the miner and any included uncles
	reward := new(big.Int).Set(blockReward)
	r := new(big.Int)
	// Reward a bad block account块账户
	for _, uncle := range uncles {
		r.Add(uncle.Number, big8)
		r.Sub(r, header.Number)
		r.Mul(r, blockReward)
		r.Div(r, big8)
		state.AddBalance(uncle.Coinbase, r)

		r.Div(blockReward, big32)
		reward.Add(reward, r)
	}
	// Reward the coinbase account
	state.AddBalance(header.Coinbase, reward)
}
```

#### Seal function implementation

In the CPU mining part, the core function of CpuAgent calls the Seal function when performing the mining operation. The Seal function attempts to find a nonce value that satisfies the block difficulty. In ModeFake and ModeFullFake mode, it returns quickly and takes the nonce value directly to 0. In shared PoW mode, use the shared Seal function. Turn on threads goroutine for mining (find the qualified nonce value).

```go
// Seal implements consensus.Engine, attempting to find a nonce that satisfies
// the block's difficulty requirements.
func (ethash *Ethash) Seal(chain consensus.ChainReader, block *types.Block, stop <-chan struct{}) (*types.Block, error) {
	// In ModeFake and ModeFullFake mode, it returns quickly and takes the nonce value directly to 0.
	if ethash.config.PowMode == ModeFake || ethash.config.PowMode == ModeFullFake {
		header := block.Header()
		header.Nonce, header.MixDigest = types.BlockNonce{}, common.Hash{}
		return block.WithSeal(header), nil
	}
	// In shared PoW mode, use the shared Seal function.
	if ethash.shared != nil {
		return ethash.shared.Seal(chain, block, stop)
	}
	// Create a runner and the multiple search threads it directs
	abort := make(chan struct{})
	found := make(chan *types.Block)

	ethash.lock.Lock()
	threads := ethash.threads
	if ethash.rand == nil {
		seed, err := crand.Int(crand.Reader, big.NewInt(math.MaxInt64))
		if err != nil {
			ethash.lock.Unlock()
			return nil, err
		}
		ethash.rand = rand.New(rand.NewSource(seed.Int64()))
	}
	ethash.lock.Unlock()
	if threads == 0 {
		threads = runtime.NumCPU()
	}
	if threads < 0 {
		threads = 0 // Allows disabling local mining without extra logic around local/remote
	}
	var pend sync.WaitGroup
	for i := 0; i < threads; i++ {
		pend.Add(1)
		go func(id int, nonce uint64) {
			defer pend.Done()
			ethash.mine(block, id, nonce, abort, found)
		}(i, uint64(ethash.rand.Int63()))
	}
	// Wait until sealing is terminated or a nonce is found
	var result *types.Block
	select {
	case <-stop:
		// Outside abort, stop all miner threads
		close(abort)
	case result = <-found:
		// One of the threads found a block, abort all others
		close(abort)
	case <-ethash.update:
		// Thread count was changed on user request, restart
		close(abort)
		pend.Wait()
		return ethash.Seal(chain, block, stop)
	}
	// Wait for all mining goroutine returns
	pend.Wait()
	return result, nil
}
```

Mine is a true function for finding the nonce value, it traverses to find the nonce value and compares the PoW value to the target value. The principle can be briefly described as follows:

```go
								RAND(h, n)  <=  M / d
```

Here M represents a very large number, here is 2^256-1; d represents the Header member Difficulty. RAND() is a concept function that represents a series of complex operations and ultimately produces a random number. This function consists of two basic parameters: h is the hash of the Header (Header.HashNoNonce()), 