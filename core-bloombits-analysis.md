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

1. scheduleRequests gor