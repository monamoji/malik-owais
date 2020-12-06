## Ethereum's Bloom filter

The block header of Ethereum contains an area called logsBloom. This area stores the Bloom filter for all the receipts in the current block, for a total of 2048 bits. That is 256 bytes.

And the receipt of one of our transactions contains a lot of log records. Each log record contains the address of the contract, multiple Topics. There is also a Bloom filter in our receipt, which records all the log information.

![image](picture/bloom_1.png)

If we look at the formal definition of log records in the Yellow Book.

O stands for our log record, Oa stands for the address of the logger, Oto, Ot1 stands for the Topics of the log, and Od stands for time.

![image](picture/bloom_2.png)

Oa is 20 bytes, Ot is 32 bytes, Od is a lot of bytes.

![image](picture/bloom_3.png)

We define a Bloom filter function M to convert a log object into a 256-byte hash.

![image](picture/bloom_4.png)

M3:2045 is a special function that sets three of the 2048 bit bits to one. For the specific method, please refer to the formula below.

![image](picture/bloom_5.png)

For any input value, first ask his KEC output, and then take the value of [0,1][2,3], [4,5] of the KEC output to modulo 2048, and get three values. These three values ​​are the subscripts that need to be set in the output 2048. That is to say, for any input, if the value of its corresponding three subscripts is not 1, then it is definitely not in this block. If, if the corresponding three bits are all 1, it cannot be stated that it is in this block. This is the characteristic of the Bloom filter.

The Bloom filter in the receipt is the union of the Bloom filter outputs for all logs.

At the same time, the logBloom in the block header is the union of the Bloom filters of all the receipts.

## ChainIndexer with BloomIndexer

I first saw ChainIndexer, I didn't really understand what it is. In fact, as you can see from the name, it is the index of Chain. In ethereum we have seen the BloomIndexer, which is the index of the Bloom filter.

The ability to find the specified Log is provided in our protocol.

The user can find the specified Log, the starting block number, and the ending block number by passing the following parameters, filtering according to the address specified by the contract Addresses, and filtering according to the specified Topics.

```go
// FilterCriteria represents a request to create a new filter.
type FilterCriteria struct {
	FromBlock *big.Int
	ToBlock   *big.Int
	Addresses []common.Address
	Topics    [][]common.Hash
}
```

If the interval between the start and end is large, it is inefficient to directly retrieve the logBloom area of ​​each block header in turn. Because each block header is stored separately, it may require a lot of random access to the disk.

So the Ethereum protocol maintains a set of indexes locally to speed up the process.

The general principle is. Each 4096 block is called a Section, and the logBloom in a Section is stored together. For each Section, use a two-dimensional data, A[2048][4096]. The first dimension 2048 represents the length of the bloom filter of 2048 bytes. The second dimension 4096 represents all the blocks in a Section, and each location represents one of the blocks in order.

- A[0][0]=blockchain[section*4096+0].logBloom[0],
- A[0][1]=blockchain[section*4096+1].logBloom[0],
- A[0][4096]=blockchain[(section+1)*4096].logBloom[0],
- A[1][0]=blockchain[section*4096+0].logBloom[1],
- A[1][1024]=blockchain[section*4096+1024].logBloom[1],
- A[2047][1]=blockchain[section*4096+1].logBloom[2047],

If Section is filled, it will be written as 2048 KV.
![image](picture/bloom_6.png)

## bloombit.go code analysis

This code is relatively non-independent. If you look at this code alone, it's a bit confusing, because it only implements some interfaces. The specific processing logic is not here, but in the core. But here I will analyze the information I have mentioned before. Subsequent more detailed logic is analyzed in detail when analyzing the core code.

The service thread startBloomHandlers, this method is to respond to the specific query request, given the specified Section and bit to query from the levelDB and then return. Looking at it alone is a bit confusing. The call to this method is more complicated. It involves a lot of logic in the core. I will not elaborate here first. Until there is this method.

```go
type Retrieval struct {
	Bit      uint			//the value of bit 0-2047 represents the value you want to get
	Sections []uint64		// those Section
	Bitsets  [][]byte		// the result of the query
}
// startBloomHandlers starts a batch of goroutines to accept bloom bit database
// retrievals from possibly a range of filters and serving the data to satisfy.
func (eth *Ethereum) startBloomHandlers() {
	for i := 0; i < bloomServiceThreads; i++ {
		go func() {
			for {
				select {
				case <-eth.shutdownChan:
					return

					// request channel
				case request := <-eth.bloomRequests:
					// get task from the channel
						task := <-request

					task.Bitsets = make([][]byte, len(task.Sections))
					for i, section := range task.Sections {
						head := core.GetCanonicalHash(eth.chainDb, (section+1)*params.BloomBitsBlocks-1)
						blob, err := bitutil.DecompressBytes(core.GetBloomBits(eth.chainDb, task.Bit, section, head), int(params.BloomBitsBlocks)/8)
						if err != nil {
							panic(err)
						}
						task.Bitsets[i] = blob
						}
						// return result via request channel
					request <- task
				}
			}
		}()
	}
}
```

### Data structure

The process of building the index for the main user of the BloomIndexer object is an interface implementation of core.ChainIndexer, so only some necessary interfaces are implemented. The logic for creating an index is also in core.ChainIndexer.

```go
// BloomIndexer implements a core.ChainIndexer, building up a rotated bloom bits index
// for the Ethereum header bloom filters, permitting blazing fast filtering.
type BloomIndexer struct {
	size uint64 // section size to generate bloombits for

	db  ethdb.Database       // database instance to write index data and metadata into
	gen *bloombits.Generator // generator to rotate the bloom bits crating the bloom index

	section uint64      // Section is the section number being processed currently  section
	head    common.Hash // Head is the hash of the last header processed
}

// NewBloomIndexer returns a chain indexer that generates bloom bits data for the
// canonical chain for fast logs filtering.
func NewBloomIndexer(db ethdb.Database, size uint64) *core.ChainIndexer {
	backend := &BloomIndexer{
		db:   db,
		size: size,
	}
	table := ethdb.NewTable(db, string(core.BloomBitsIndexPrefix))

	return core.NewChainIndexer(db, table, backend, size, bloomConfirms, bloomThrottling, "bloombits")
}
```

Reset implements the ChainIndexerBackend method and starts a new section.

```go
// Reset implements core.ChainIndexerBackend, starting a new bloombits index
// section.
func (b *BloomIndexer) Reset(section uint64) {
	gen, err := bl