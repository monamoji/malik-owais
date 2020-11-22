Genesis is the content of the genesis block. A blockchain is formed by rules from the same genesis block. Different networks have different genesis blocks, creation networks of the main network and test network. It is different.  
This module sets the state of genesis based on the initial value of the incoming genesis and the database. If there is no genesis block, create it in the database.
data structure

```go
// Genesis specifies the header fields, state of a genesis block. It also defines hard
// fork switch-over blocks through the chain configuration.
type Genesis struct {
	Config     *params.ChainConfig `json:"config"`
	Nonce      uint64              `json:"nonce"`
	Timestamp  uint64              `json:"timestamp"`
	ExtraData  []byte              `json:"extraData"`
	GasLimit   uint64              `json:"gasLimit"   gencodec:"required"`
	Difficulty *big.Int            `json:"difficulty" gencodec:"required"`
	Mixhash    common.Hash         `json:"mixHash"`
	Coinbase   common.Address      `json:"coinbase"`
	Alloc      GenesisAlloc        `json:"alloc"      gencodec:"required"`

	// These fields are used for consensus tests. Please don't use them
	// in actual genesis blocks.
	Number     uint64      `json:"number"`
	GasUsed    uint64      `json:"gasUsed"`
	ParentHash common.Hash `json:"parentHash"`
}

// GenesisAlloc specifies the initial state that is part of the genesis block.
type GenesisAlloc map[common.Address]GenesisAccount
```

SetupGenesisBlock,

```go
// SetupGenesisBlock writes or updates the genesis block in db.
//
// The block that will be used is:
//
//                          genesis == nil       genesis != nil
//                       +------------------------------------------
//     db has no genesis |  main-net default  |  genesis
//     db has genesis    |  from DB           |  genesis (if compatible)
//
// The stored chain configuration will be updated if it is compatible (i.e. does not
// specify a fork block below the local head block). In case of a conflict, the
// error is a *params.ConfigCompatError and the new, unwritten config is returned.
// The returned chain configuration is never nil.

// genesis is not nil if it is testnet dev or rinkeby mode.
func SetupGenesisBlock(db ethdb.Database, genesis *Genesis) (*params.ChainConfig, common.Hash, error) {
	if genesis != nil && genesis.Config == nil {
		return params.AllProtocolChanges, common.Hash{}, errGenesisNoConfig
	}

	// Just commit the new block if there is no stored genesis block.
	stored := GetCanonicalHash(db, 0)
	if (stored == common.Hash{}) { // If there is no block, the start of the geth will start here.
		if genesis == nil {
			log.Info("Writing default main-net genesis block")
			genesis = DefaultGenesisBlock()
		} else { // custome genesis block
			log.Info("Writing custom genesis block")
		}
		// update blockchain db
		block, err := genesis.Commit(db)
		return genesis.Config, block.Hash(), err
	}

	// Check wh