table.go mainly implements the Kademlia protocol of p2p.

### Introduction to the Kademlia protocol (recommended reading the [pdf document](./references/Kademlia.pdf) in the references)

KThe Kademlia Agreement (referred to as Kad) is a result of a study published in 2002 by PetarP. Maymounkov and David Mazieres of New York University. Kademlia: A peerto - peer information system based on the XOR metric. Simply put, Kad is a distributed hash table (DHT) technology, but compared to other DHT implementation technologies, such as Chord, CAN, Pastry, etc., Kad uses a unique XOR algorithm for distance measurement. A new DHT topology has been established, which greatly improves the routing query speed compared to other algorithms.

### table structure and fields

```go
const (
	alpha      = 3  // Kademlia concurrency factor
	bucketSize = 16 // Kademlia bucket size
	hashBits   = len(common.Hash{}) * 8
	nBuckets   = hashBits + 1 // Number of buckets

	maxBondingPingPongs = 16
	maxFindnodeFailures = 5

	autoRefreshInterval = 1 * time.Hour
	seedCount           = 30
	seedMaxAge          = 5 * 24 * time.Hour
)

type Table struct {
	mutex   sync.Mutex        // protects buckets, their content, and nursery
	buckets [nBuckets]*bucket // index of known nodes by distance
	nursery []*Node           // bootstrap nodes
	db      *nodeDB           // database of known nodes

	refreshReq chan chan struct{}
	closeReq   chan struct{}
	closed     chan struct{}

	bondmu    sync.Mutex
	bonding   map[NodeID]*bondproc
	bondslots chan struct{} // limits total number of active bonding processes

	nodeAddedHook func(*Node) // for testing

	net  transport
	self *Node // metadata of the local node
}
```

### initialization

```go
func newTable(t transport, ourID NodeID, ourAddr *net.UDPAddr, nodeDBPath string) (*Table, error) {
	// If no node database was given, use an in-memory one
	// This is introduced in the previous database.go. Open leveldb. If path is empty. Then open a memory based db
	db, err := newNodeDB(nodeDBPath, Version, ourID)
	if err != nil {
		return nil, err
	}
	tab := &Table{
		net:        t,
		db:         db,
		self:       NewNode(ourID, ourAddr.IP, uint16(ourAddr.Port), uint16(ourAddr.Port)),
		bonding:    make(map[NodeID]*bondproc),
		bondslots:  make(chan struct{}, maxBondingPingPongs),
		refreshReq: make(chan chan struct{}),
		closeReq:   make(chan struct{}),
		closed:     make(chan struct{}),
	}
	for i := 0; i < cap(tab.bondslots); i++ {
		tab.bondslots <- struct{}{}
	}
	for i := range tab.buckets {
		tab.buckets[i] = new(bucket)
	}
	go tab.refreshLoop()
	return tab, nil
}
```

The above initialization starts a goroutine refreshLoop(). This function mainly performs the following work.

1. Refresh every hour (autoRefreshInterval)
2. If a refreshReq request is received. Then do the refresh work.
3. If a close message is received. Then close it.

So the main job of the function is to start the refresh work. doRefresh

```go
// refreshLoop schedules doRefresh runs and coordinates shutdown.
func (tab *Table) refreshLoop() {
	var (
		timer   = time.NewTicker(autoRefreshInterval)
		// accumulates waiting callers while doRefresh runs
		waiting []chan struct{}
		// where doRefresh reports completion
		done    chan struct{}
	)
loop:
	for {
		select {
		case <-timer.C:
			if done == nil {
				done = make(chan struct{})
				go tab.doRefresh(done)
			}
		case req := <-tab.refreshReq:
			waiting = append(waiting, req)
			if done == nil {
				done = make(chan struct{})
				go tab.doRefresh(done)
			}
		case <-done:
			for _, ch := range waiting {
				close(ch)
			}
			waiting = nil
			done = nil
		case <-tab.closeReq:
			break loop
		}
	}

	if tab.net != nil {
		tab.net.close()
	}
	if done != nil {
		<-done
	}
	for _, ch := range waiting {
		close(ch)
	}
	tab.db.close()
	close(tab.closed)
}
```

doRefresh method

```go
// doRefresh performs a lookup for a random target to keep buckets
// full. seed nodes are inserted if the table is empty (initial
// bootstrap or discarded faulty peers).
func (tab *Table) doRefresh(done chan struct{}) {
	defer close(done)

	// The Kademlia paper specifies that the bucket refresh should
	// perform a lookup in the least recently used bucket. We cannot
	// adhere to this because the findnode target is a 512bit value
	// (not hash-sized) and it is not easily possible to generate a
	// sha3 preimage that falls into a chosen bucket.
	// We perform a lookup with a random target instead.
	var target NodeID
	rand.Read(target[:])
	result := tab.lookup(target, false) // Lookup is to find the k nodes closest to the target
	if len(result) > 0 {
		return
	}

	// The table is empty. Load nodes from the database and insert
	// them. This should yield a few previously seen nodes that are
	// (hopefully) still alive.
	// querySeeds function is described in the database.go section, which randomly finds available seed nodes from the database.
	//The database is blank at the very beginning. That is, the seeds returned at the beginning are empty.
	seeds := tab.db.querySeeds(seedCount, seedMaxAge)
	// Call the bondall function. Will try to contact these nodes and insert them into the table.
	//tab.nursery is the seed node specified on the command line.
	//At the beginning of the startup. The value of tab.nursery is built into the code. This is worthwhile.
	//$GOPATH/src/github.com/ethereum/go-ethereum/mobile/params.go
	//There is a dead val