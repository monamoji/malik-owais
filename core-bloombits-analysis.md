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
	running uint32 // Atomic flag whether a session is live or not
}
```

The general flow picture of the matcher, the ellipse on the way represents the goroutine. The rectangle represents the channel. Triangles represent method calls.

![image](picture/matcher_1.png)

1. First, Matcher creates a corresponding number of subMatch based on the number of incoming filters. Each subMatch corresponds to a filter object. Each subMatch will get new results by bitwise matching the results of its own search and the previous search result. If all the bits of the new result are set, the result of the search will be passed to the next one. This is a short-circuit algorithm that implements the summation of the results of all filters. If the previous calculations can't match anything, then there is no need to match the following conditions.
2. Matcher will start the corresponding number of schedules based on the number of subscripts of the blender's Bloom filter.
3. subMatch will send the request to the corresponding schedule.
4. Schedule dispatches the request to the distributor via dist and manages it in the distributor.
5. Multiple (16) multiplex threads are started, requests are fetched from the distributor, and the request is sent to the bloomRequests queue, which starts accessing the database, fetching the data, and returning it to the multiplex.
6. Multiplex tells the distributor the answer via the deliveries channel.
7. Distributor calls the dispatch method of schedule and sends the result to schedule
8. Schedule returns the result to subMatch.
9. SubMatch calculates the result and sends it to the next subMatch for processing. If it is the last subMatch, the result is processed and sent to the results channel.

matcher

```go
filter := New(backend, 0, -1, []common.Address{addr}, [][]common.Hash{{hash1, hash2, hash3, hash4}})
// The relationship between groups is the relationship between the group and the group.
// (addr && hash1) ||(addr && hash2)||(addr && hash3)||(addr && hash4)
```

The constructor, which needs special attention is the input filters parameter. This parameter is a three-dimensional array [][]bloomIndexes === [first dimension][second dimension][3].

```go
// filter.go is the caller of the matcher

// You can see that no matter how many addresses, there is only one location in the filters.
// Filters[0] = addresses
// filters[1] = topics[0] = multi-topic
// filters[2] = topics[1] = multi-topic
// filters[n] = topics[n] = multi-topic

// Filter's parameter addresses and topics filter algorithm is (with any address in addresses) and (with any topic in topics[0]) and (with any topic in topics[1]) and (with any topic in topics[n])

// It can be seen that for the filter is the execution and operation of the data in the first dimension, the execution or operation of the data in the second dimension.

// In the NewMatcher method, the specific data of the third dimension is converted into three specified positions of the Bloom filter. So in the filter.go var filters [][][]byte in the Matcher filter becomes [][][3]

func New(backend Backend, begin, end int64, addresses []common.Address, topics [][]common.Hash) *Filter {
	// Flatten the address and topic filter clauses into a single bloombits filter
	// system. Since the bloombits are not positional, nil topics are permitted,
	// which get flattened into a nil byte slice.
	var filters [][][]byte
	if len(addresses) > 0 {
		filter := make([][]byte, len(addresses))
		for i, address := range addresses {
			filter[i] = address.Bytes()
		}
		filters = append(filters, filter)
	}
	for _, topicList := range topics {
		filter := make([][]byte, len(topicList))
		for i, topic := range topicList {
			filter[i] = topic.Bytes()
		}
		filters = append(filters, filter)
	}

// NewMatcher creates a new pipeline for retrieving bloom bit streams and doing
// address and topic filtering on them. Setting a filter component to `nil` is
// allowed and will result in that filter rule being skipped (OR 0x11...1).
func NewMatcher(sectionSize uint64, filters [][][]byte) *Matcher {
	// Create the matcher instance
	m := &Matcher{
		sectionSize: sectionSize,
		schedulers:  make(map[uint]*scheduler),
		retrievers:  make(chan chan uint),
		counters:    make(chan chan uint),
		retrievals:  make(chan chan *Retrieval),
		deliveries:  make(chan *Retrieval),
	}
	// Calculate the bloom bit indexes for the groups we're interested in
	m.filters = nil

	for _, filter := range filters {
		// Gather the bit indexes of the filter rule, special casing the nil filter
		if len(filter) == 0 {
			continue
		}
		bloomBits := make([]bloomIndexes, len(filter))
		for i, clause := range filter {
			if clause == nil {
				bloomBits = nil
				break
			}
			// The clause corresponds to the data of the third dimension of the input, which may be an address or a topic
			// calcBloomIndexes calculates the three subscripts in the Bloom filter corresponding to this data (0-2048), that is, if the corresponding three bits in the Bloom filter are both 1, then the data may be clause it's here.

			bloomBits[i] = calcBloomIndexes(clause)
		}
		// Accumulate the filter rules if no nil rule was within
		// In the calculation, if only one of the bloomBits can be found. Then think that the whole is established.
		if bloomBits != nil {
			// different bloombits
			m.filters = append(m.filters, bloomBits)
		}
	}
	// For every bit, create a scheduler to load/download the bit vectors
	for _, bloomIndexLists := range m.filters {
		for _, bloomIndexList := range bloomIndexLists {
			for _, bloomIndex := range bloomIndexList {
				// For all possible subscripts. We all generate a scheduler to perform the corresponding position.
				m.addScheduler(bloomIndex)
			}
		}
	}
	return m
}
```

Start

```go
// Start starts the matching process and returns a stream of bloom matches in
// a given range of blocks. If there are no more matches in the range, the result
// channel is closed.
func (m *Matcher) Start(begin, end uint64, results chan uint64) (*MatcherSession, error) {
	// Make sure we're not creating concurrent sessions
	if atomic.SwapUint32(&m.running, 1) == 1 {
		return nil, errors.New("matcher already running")
	}
	defer atomic.StoreUint32(&m.running, 0)

	// Initiate a new matching round
	// A session is started, and as a return value, the life cycle of the lookup is managed.
	session := &MatcherSession{
		matcher: m,
		quit:    make(chan struct{}),
		kill:    make(chan struct{}),
	}
	for _, scheduler := range m.schedulers {
		scheduler.reset()
	}
	// This run establishes the process and returns a partialMatches type of pipeline representing partial results of the query.
	sink := m.run(begin, end, cap(results), session)

	// Read the output from the result sink and deliver to the user
	session.pend.Add(1)
	go func() {
		defer session.pend.Done()
		defer close(results)

		for {
			select {
			case <-session.quit:
				return

			case res, ok := <-sink:
				// New match result found
				// So you need to iterate through the bitmap, find the blocks that are set, and return the block number back.
				if !ok {
					return
				}
				// Calculate the first and last blocks of the section
				sectionStart := res.section * m.sectionSize

				first := sectionStart
				if begin > first {
					first = begin
				}
				last := sectionStart + m.sectionSize - 1
				if end < last {
					last = end
				}
				// Iterate over all the blocks in the section and return the matching ones
				for i := first; i <= last; i++ {
					// Skip the entire byte if no matches are found inside
					next := res.bitset[(i-sectionStart)/8]
					if next == 0 {
						i += 7
						continue
					}
					// Some bit it set, do the actual submatching
					if bit := 7 - i%8; next&(1<<bit) != 0 {
						select {
						case <-session.quit:
							return
						case results <- i:
						}
					}
				}
			}
		}
	}()
	return session, nil
}
```

run method

```go
// run creates a daisy-chain of sub-matchers, one for the address set and one
// for each topic set, each sub-matcher receiving a section only if the previous
// ones have all found a potential match in one of the blocks of the section,
// then binary AND-ing its own matches and forwaring the result to the next one.
// The method starts feeding the section indexes into the first sub-matcher on a
// new goroutine and returns a sink channel receiving the results.
func (m *Matcher) run(begin, end uint64, buffer int, session *MatcherSession) chan *partialMatches {
	// Create the source channel and feed section indexes into
	source := make(chan *partialMatches, buffer)

	session.pend.Add(1)
	go func() {
		defer session.pend.Done()
		defer close(source)

		for i := begin / m.sectionSize; i <= end/m.sectionSize; i++ {
			// This for loop constructs the first input source of subMatch, and the remaining subMatch takes the previous result as its own source.
			// The bitset field of this source is 0xff, which represents a complete match. It will be compared with the match of our step to get the result of this step match.
			select {
			case <-session.quit:
				return
			case source <- &partialMatches{i, bytes.Repeat([]byte{0xff}, int(m.sectionSize/8))}:
			}
		}
	}()
	// Assemble the daisy-chained filtering pipeline
	next := source
	dist := make(chan *request, buffer)

	// Build the pipeline, the previous output as the input to the next subMatch.
	for _, bloom := range m.filters {
		next = m.subMatch(next, dist, bloom, session)
	}
	// Start the request distribution
	session.pend.Add(1)
	// distributor go routine
	go m.distributor(dist, session)

	return next
}
```

subMatch method

```go
// subMatch creates a sub-matcher that filters for a set of addresses or topics, binary OR-s those matches, then
// binary AND-s the result to the daisy-chain input (source) and forwards it to the daisy-chain output.
// The matches of each address/topic are calculated by fetching the given sections of the three bloom bit indexes belonging to
// that address/topic, and binary AND-ing those vectors together.

// SubMatch is the most important function that combines the first dimension of the filters [][][3] with the second dimension or the third dimension.
func (m *Matcher) subMatch(source chan *partialMatches, dist chan *request, bloom []bloomIndexes, session *MatcherSession) chan *partialMatches {
	// Start the concurrent schedulers for each bit required by the bloom filter
	// The incoming bloom []bloomIndexes parameter is the second, third dimension of filters [][3]

	sectionSources := make([][3]chan uint64, len(bloom))
	sectionSinks := make([][3]chan []byte, len(bloom))
	for i, bits := range bloom { // i represents the number of second dimensions
		for j, bit := range bits {  //j Represents the subscript of the Bloom filter. There are definitely only three values (0-2048).
			sectionSources[i][j] = make(chan uint64, cap(source))
			sectionSinks[i][j] = make(chan []byte, cap(source))
			// Initiate a scheduling request for this bit, passing the section that needs to be queried via sectionSources[i][j]
			// Receive results via sectionSinks[i][j]
			// dist is the channel through which the scheduler passes the request. This is in the introduction of the scheduler.
			m.schedulers[bit].run(sectionSources[i][j], dist, sectionSinks[i][j], session.quit, &session.pend)
		}
	}

	process := make(chan *partialMatches, cap(source)) // entries from source are forwarded here after fetches have been initiated
	results := make(chan *partialMatches, cap(source))

	session.pend.Add(2)
	go func() {
		// Tear down the goroutine and terminate all source channels
		defer session.pend.Done()
		defer close(process)

		defer func() {
			for _, bloomSources := range sectionSources {
				for _, bitSource := range bloomSources {
					close(bitSource)
				}
			}
		}()
		// Read sections from the source channel and multiplex into all bit-schedulers
		for {
			select {
			case <-session.quit:
				return

			case subres, ok := <-source:
				// New subresult from previous link
				if !ok {
					return
				}
				// Multiplex the section index to all bit-schedulers
				for _, bloomSources := range sectionSources {
					for _, bitSource := range bloomSources {
						// Pass to the input channel of all the schedulers above. Apply for these
						// The specified bit of the section is searched.
						// The result will be sent to sectionSinks[i][j]
						select {
						case <-session.quit:
							return
						case bitSource <- subres.section:
						}
					}
				}
				// Notify the processor that this section will become available
				select {
				case <-session.quit:
					return
				case process <- subres: // Wait until all requests are submitted to the scheduler to send a message to the process.
				}
			}
		}
	}()

	go func() {
		// Tear down the goroutine and terminate the final sink channel
		defer session.pend.Done()
		defer close(results)

		// Read the source notifications and collect the delivered results
		for {
			select {
			case <-session.quit:
				return

			case subres, ok := <-process:
				// There is a problem here. Is it possible to order out. Because the channels are all c