## scheduler.go

The scheduler is a schedule for a single bit value retrieval based on a section's Bloom filter. In addition to scheduling retrieval operations, this structure can deduplicate requests and cache results, minimizing network/database overhead even in complex filtering situations.

### data structure

Request represents a bloom retrieval task to preferentially retrieve from the local database or from the network. Section indicates the block segment number, 4096 blocks per segment, and bit indicates which bit of the Bloom filter is retrieved (a total of 2048 bits). This was introduced in the previous [yellow book](./references/yellowpaper/paper.pdf) [eth-bloombits-and-filter-analysis](eth-bloombits-and-filter-analysis.md)

```go
// request represents a bloom retrieval task to prioritize and pull from the local
// database or remotely from the network.
type request struct {
	section uint64 // Section index to retrieve the a bit-vector from
	bit uint // Bit index within the section to retrieve the vector of
}
```

Response: The status of the currently scheduled request. Failure to send a request will generate a response object to finalize the state of the request. The cached is used to cache the result of this section.

```go
// response represents the state of a requested bit-vector through a scheduler.
type response struct {
	cached []byte        // Cached bits to dedup multiple requests
	done   chan struct{} // Channel to allow waiting for completion
}
```

scheduler

```go
// scheduler handles the scheduling of bloom-filter retrieval operations for
// entire section-batches belonging to a single bloom bit. Beside scheduling the
// retrieval operations, this struct also deduplicates the requests and caches
// the results to minimize network/database overhead even in complex filtering
// scenarios.
type scheduler struct {
	bit       uint                 // Index of the bit in the bloom filter this scheduler is responsible for bit(0-2047)
	responses map[uint64]*response // Currently pending retrieval requests or already cached responses
	lock      sync.Mutex           // Lock protecting the responses from concurrent access
}
```

### Constructor

newScheduler and reset method

```go
// newScheduler creates a new bloom-filter retrieval scheduler for a specific
// bit index.
func newScheduler(idx uint) *scheduler {
	return &scheduler{
		bit:       idx,
		responses: make(map[uint64]*response),
	}
}
// reset cleans up any leftovers from previous runs. This is required before a
// restart to ensure the no previously requested but never delivered state will
// cause a lockup.
func (s *scheduler) reset() {
	s.lock.Lock()
	defer s.lock.Unlock()

	for section, res := range s.responses {
		if res.cached == nil {
			delete(s.responses, section)
		}
	}
}
```

### the run method

The run method creates a pipeline that receives the sections that require the request from the sections channel and returns the results in the order requested by the done channel. It is possible to run the same scheduler concurrently, which will result in duplication of tasks.

```go
// run creates a retrieval pipeline, receiving section indexes from sections and
// returning the results in the same order through the done channel. Concurrent
// runs of the same scheduler are allowed, leading to retrieval task deduplication.
func (s *scheduler) run(sections chan uint64, dist chan *request, done chan []byte, quit chan struct{}, wg *sync.WaitGroup) {
	// sections channel type
	// dist send then receive
	// done return channel

	// Create a forwarder channel between requests and responses of the same size as
	// the distribution channel (since that will block the pipeline anyway).
	pend := make(chan uint64, cap(dist))

	// Start the pipeline schedulers to forward between user -> distributor -> user
	wg.Add(2)
	go s.scheduleRequests(sections, dist, pend, quit, wg)
	go s.scheduleDeliveries(pend, done, quit, wg)
}
```

### scheduler flowchart

```mermaid
graph LR

subgraph ""
  id1(sections)
  id2((scheduleRequests))
end
id1 --> id2
id2 --> dist

subgraph ""
  id3(scheduleDelivers)
  id4>deliver]
end
id2 --<b>pend</b>--> id3
id4 --<b>response done</b>--> id3
id3 --> done

style id2 stroke: #333, stroke-width:2px;
style id3 stroke: #333, stroke-width:2px;
```

The ellipse in the figure represents the goroutine. The rectangle represents the channel. The triangle represents the external method call.

1. scheduleRequests goroutine receives a section message from sections
2. scheduleRequests assembles the received section into requtest and sends it to the dist channel, and builds the object response[section]
3. scheduleRequests sends the previous section to the pend queue. scheduleDelivers receives a pend message and blocks it on response[section].done
4. Externally call the deliver method, write the result request of seciton to response[section].cached. and close response[section].done channel
5. scheduleDelivers receives the response[section].done message. Send response[section].cached to the done channel

### scheduleRequests

```go
// scheduleRequests reads section retrieval requests from the input channel,
// deduplicates the stream and pushes unique retrieval tasks into the distribution
// channel for a database or network layer to honour.
func (s *scheduler) scheduleRequests(reqs chan uint64, dist chan *request, pend chan uint64, quit chan struct{}, wg *sync.WaitGroup) {
	// Clean up the goroutine and pipeline when done
	defer wg.Done()
	defer close(pend)

	// Keep reading and scheduling section requests
	for {
		select {
		case <-quit:
			return

		case section, ok := <-reqs:
			// New section retrieval requested
			if !ok {
				return
			}
			// Deduplicate retrieval requests
			unique := false

			s.lock.Lock()
			if s.responses[section] == nil {
				s.responses[section] = &response{
					done: make(chan struct{}),
				}
				unique = true
			}
			s.lock.Unlock()

			// Schedule the section for retrieval and notify the deliverer to expect this section
			if unique {
				select {
				case <-quit:
					return
				case dist <- &request{bit: s.bit, section: section}:
				}
			}
			select {
			case <-quit:
				return
			case pend <- section:
			}
		}
	}
}
```

## generator.go

Generator An object used to generate section-based Bloom filter index data. The main data structure inside the generator is the data structure of bloom[2048][4096]bit. The input is 4096 header.logBloom data. For example, the logBloom of the 20th header is stored in bloom[0:2048][20]

data structure:

```go
// Generator takes a number of bloom filters and generates the rotated bloom bits
// to be used for batched filtering.
type Generator struct {
	blooms   [types.BloomBitLength][]byte // Rotated blooms for per-bit matching
	sections uint                         // Number of sections to batch together
	nextBit  uint                         // Next bit to set when adding a bloom
}
```

structure:

```go
// NewGenerator creates a rotated bloom generator that can iteratively fill a
// batched bloom filter's bits.
//
func NewGenerator(sections uint) (*Generator, error) {
	if sections%8 != 0 {
		return nil, errors.New("section count not multiple of 8")
	}
	b := &Generator{sections: sections}
	for i := 0; i < types.BloomBitLength; i++ { //BloomBitLength=2048
		b.blooms[i] = make([]byte, sections/8)  // 1 byte = 8 bits
	}
	return b, nil
}
```

AddBloom adds a block header to the searchBloom

```go
// AddBloom takes a single bloom filter and sets the corresponding bit column
// in memory accordingly.
func (b *Generator) AddBloom(index uint, bloom types.Bloom) error {
	// Make sure we're not adding more bloom filters than our capacity
	if b.nextBit >= b.sections { // exceeded the maximum number of sections
		return errSectionOutOfBounds
	}
	if b.nextBit != index {  // bloom not in section
		return errors.New("bloom filter with unexpected index")
	}
	// Rotate the bloom and insert into our collection
	byteIndex := b.nextBit / 8
	bitMask := byte(1) << byte(7-b.nextBit%8) // Find the bit of the byte that needs to be set in the byte

	for i := 0; i < types.BloomBitLength; i++ {
		bloomByteIndex := types.BloomByteLength - 1 - i/8
		bloomBitMask := byte(1) << byte(i%8)

		if (bloom[bloomByteIndex] & bloomBitMask) != 0 {
			b.blooms[i][byteIndex] |= bitMask
		}
	}
	b.nextBit++

	return nil
}
```

Bitset returns

```go
// Bitset returns the bit vector belonging to the given bit index after all
// blooms have been added.
func (b *Generator) Bitset(idx uint) ([]byte, error) {
	if b.nextBit != b.sections {
		return nil, errors.New("bloom not fully generated yet")
	}
	if idx >= b.sections {
		return nil, errSectionOutOfBounds
	}
	return b.blooms[idx], nil
}
```

## matcher.go

Matcher is a pipeline system scheduler and logic matcher that performs binary and/or operations on the bitstream, creating a stream of potential blocks to examine the data content.

Data structure

```go
// partialMatches with a non-nil vector represents a section in which some sub-
// matchers have already found potential matches. Subsequent sub-matchers will
// binary AND their matches with this vector. If vector is nil, it represents a
// section to be processed by the first sub-matcher.
// partialMatches Represents the result of a partial match. There are three conditions that need to be filtered, addr1, addr2, addr3 , and you need to find data that matches these three conditions at the same time. Then we start the pipeline that contains these three conditions.
// The result of the first match is sent to the second. The second performs a bitwise AND operation on the first result and its own result, and then sends it to the third process as a result of the match.
type partialMatches struct {
	section uint64
	bitset []byte
}
// Retrieval represents a request for retrieval task assignments for a given
// bit with the given number of fetch elements, or a response for such a request.
// It can also have the actual results set to be used as a delivery data struct.
// Retrieval: Represents the retrieval of a block Bloom filter index. This object is sent to startBloomHandlers in eth/bloombits.go. This method loads the Bloom filter index from the database and returns it in Bitsets.
type Retrieval struct {
	Bit uint
	Sections []uint64
	Bitsets [][]byte
}
// Matcher is a pipelined system of schedulers and logic matchers which perform
// binary AND/OR operations on the bit-streams, creating a stream of potential
// blocks to inspect for data content.
type Matcher struct {
	sectionSize uint64 // Size of the data batches to filter on
	filters [][]bloomIndexes // Filter the system is matching for
	schedulers map[uint]*scheduler // Retrieval schedulers for loading bloom bits
	retrievers chan chan uint // Retriever processes waiting for bit allocations
	counters chan chan uint // Retriever processes waiting for task count reports
	retrievals chan chan *Retrieval // Retriever processes waiting for task allocations
	deliveries chan *Retrieval // Retriever processes waiting for task response deliveries
	running uint32