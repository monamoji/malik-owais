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
	//There is a dead value written here. This value is written by the SetFallbackNodes method. This method will be analyzed later.
	//This will be a two-way pingpong exchange. Then store the results in the database.
	seeds = tab.bondall(append(seeds, tab.nursery...))

	if len(seeds) == 0 { // No seed nodes are found and may need to wait for the next refresh.
		log.Debug("No discv4 seed nodes found")
	}
	for _, n := range seeds {
		age := log.Lazy{Fn: func() time.Duration { return time.Since(tab.db.lastPong(n.ID)) }}
		log.Trace("Found seed node in database", "id", n.ID, "addr", n.addr(), "age", age)
	}
	tab.mutex.Lock()
	// This method adds all bonded seed to the bucket (provided the bucket is not full)
	tab.stuff(seeds)
	tab.mutex.Unlock()

	// Finally, do a self lookup to fill up the buckets.
	tab.lookup(tab.self.ID, false) // With a seed node. Then find yourself to fill the buckets.
}
```

The bondall method, which is a multithreaded call to the bond method.

```go
// bondall bonds with all given nodes concurrently and returns
// those nodes for which bonding has probably succeeded.
func (tab *Table) bondall(nodes []*Node) (result []*Node) {
	rc := make(chan *Node, len(nodes))
	for i := range nodes {
		go func(n *Node) {
			nn, _ := tab.bond(false, n.ID, n.addr(), uint16(n.TCP))
			rc <- nn
		}(nodes[i])
	}
	for range nodes {
		if n := <-rc; n != nil {
			result = append(result, n)
		}
	}
	return result
}
```

Bond method. Remember in udp.go. This method may also be called when we receive a ping method.

```go
// bond ensures the local node has a bond with the given remote node.
// It also attempts to insert the node into the table if bonding succeeds.
// The caller must not hold tab.mutex.
// A bond is must be established before sending findnode requests.
// Both sides must have completed a ping/pong exchange for a bond to
// exist. The total number of active bonding processes is limited in
// order to restrain network use.
// bond is meant to operate idempotently in that bonding with a remote
// node which still remembers a previously established bond will work.
// The remote node will simply not send a ping back, causing waitping
// to time out.
// If pinged is true, the remote node has just pinged us and one half
// of the process can be skipped.
func (tab *Table) bond(pinged bool, id NodeID, addr *net.UDPAddr, tcpPort uint16) (*Node, error) {
	if id == tab.self.ID {
		return nil, errors.New("is self")
	}
	// Retrieve a previously known node and any recent findnode failures
	node, fails := tab.db.node(id), 0
	if node != nil {
		fails = tab.db.findFails(id)
	}
	// If the node is unknown (non-bonded) or failed (remotely unknown), bond from scratch
	var result error
	age := time.Since(tab.db.lastPong(id))
	if node == nil || fails > 0 || age > nodeDBNodeExpiration {
		// If the database does not have this node. Or the number of errors is greater than 0 or the node times out.
		log.Trace("Starting bonding ping/pong", "id", id, "known", node != nil, "failcount", fails, "age", age)

		tab.bondmu.Lock()
		w := tab.bonding[id]
		if w != nil {
			// Wait for an existing bonding process to complete.
			tab.bondmu.Unlock()
			<-w.done
		} else {
			// Register a new bonding process.
			w = &bondproc{done: make(chan struct{})}
			tab.bonding[id] = w
			tab.bondmu.Unlock()
			// Do the ping/pong. The result goes into w.
			tab.pingpong(w, pinged, id, addr, tcpPort)
			// Unregister the process after it's done.
			tab.bondmu.Lock()
			delete(tab.bonding, id)
			tab.bondmu.Unlock()
		}
		// Retrieve the bonding results
		result = w.err
		if result == nil {
			node = w.n
		}
	}
	if node != nil {
		// Add the node to the table even if the bonding ping/pong
		// fails. It will be relaced quickly if it continues to be
		// unresponsive.
		// This method is more important. If the corresponding bucket has space, the buckets will be inserted directly. If the buckets are full. The ping operation will be used to test the nodes in the bucket to try to make room.
		tab.add(node)
		tab.db.updateFindFails(id, 0)
	}
	return node, result
}
```

pingpong method

```go
func (tab *Table) pingpong(w *bondproc, pinged bool, id NodeID, addr *net.UDPAddr, tcpPort uint16) {
	// Request a bonding slot to limit network usage
	<-tab.bondslots
	defer func() { tab.bondslots <- struct{}{} }()

	// Ping the remote side and wait for a pong.
	if w.err = tab.ping(id, addr); w.err != nil {
		close(w.done)
		return
	}
	//This is set to true when udp receives a ping message. At this time we have received the ping message from the other party.
	// Then we will wait for the ping message differently. Otherwise, you need to wait for the ping message sent by the other party (we initiate the ping message actively).
	if !pinged {
		// Give the remote node a chance to ping us before we start
		// sending findnode requests. If they still remember us,
		// waitping will simply time out.
		tab.net.waitping(id)
	}
	// Bonding succeeded, update the node database.
	// Complete the bond process. Insert the node into the database. The database operation is done here. The operation of the bucket is done in tab.add. Buckets are memory operations. The database is a persistent seeds node. Used to speed up the startup process.
	w.n = NewNode(id, addr.IP, uint16(addr.Port), tcpPort)
	tab.db.updateNode(w.n)
	close(w.done)
}
```

tab.add method

```go
// add attempts to add the given node its corresponding bucket. If the
// bucket has space available, adding the node succeeds immediately.
// Otherwise, the node is added if the least recently active node in
// the bucket does not respond to a ping packet.
// The caller must not hold tab.mutex.
func (tab *Table) add(new *Node) {
	b := tab.buckets[logdist(tab.self.sha, new.sha)]
	tab.mutex.Lock()
	defer tab.mutex.Unlock()
	if b.bump(new) { // If the node exists. Then update its value. Then quit.
		return
	}
	var oldest *Node
	if len(b.entries) == bucketSize {
		oldest = b.entries[bucketSize-1]
		if oldest.contested {
			// The node is already being replaced, don't attempt
			// to replace it.
			// If another goroutine is testing this node. Then cancel the replacement and exit directly.
			// Because the ping time is longer. So this time is not locked. This state is used to identify this situation.
			return
		}
		oldest.contested = true
		// Let go of the mutex so other goroutines can access
		// the table while we ping the least recently active node.
		tab.