The core/state package provides a layer of cache for the state trie of Ethereum.
The structure of state is mainly as shown below

![image](picture/state_1.png)

The blue rectangle represents the module and the gray rectangle represents the external module.

- Database mainly provides an abstraction of the trie tree, providing a cache of the trie tree and a cache of contract code length
- The journal mainly provides the operation log and the function of the operation rollback.
- State_object is an abstraction of the account object that provides some functionality for the account.
- The statedb mainly provides some of the functions of the state trie.

## database.go

Database.go provides an abstraction of the database.
data structure

```go
// Database wraps access to tries and contract code.
type Database interface {
	// Accessing tries:
	// OpenTrie opens the main account trie.
	// OpenStorageTrie opens the storage trie of an account.
	OpenTrie(root common.Hash) (Trie, error)
	OpenStorageTrie(addrHash, root common.Hash) (Trie, error)
	// Accessing contract code:
	ContractCode(addrHash, codeHash common.Hash) ([]byte, error)
	// The size of the access contract. This method may be called frequently. Because there is a cache.
	ContractCodeSize(addrHash, codeHash common.Hash) (int, error)
	// CopyTrie returns an independent copy of the given trie.
	CopyTrie(Trie) Trie
}

// NewDatabase creates a backing store for state. The returned database is safe for
// concurrent use and retains cached trie nodes in memory.
func NewDatabase(db ethdb.Database) Database {
	csc, _ := lru.New(codeSizeCacheSize)
	return &cachingDB{db: db, codeSizeCache: csc}
}

type cachingDB struct {
	db            ethdb.Database
	mu            sync.Mutex
	pastTries     []*trie.SecureTrie  // trie cache
	codeSizeCache *lru.Cache		  // contract code size cache
}
```

OpenTrie, look up from the cache. If you find a copy of the trie then return the cache, otherwise rebuild a tree to return.

```go
func (db *cachingDB) OpenTrie(root common.Hash) (Trie, error) {
	db.mu.Lock()
	defer db.mu.Unlock()

	for i := len(db.pastTries) - 1; i >= 0; i-- {
		if db.pastTries[i].Hash() == root {
			return cachedTrie{db.pastTries[i].Copy(), db}, nil
		}
	}
	tr, err := trie.NewSecure(root, db.db, MaxTrieCacheGen)
	if err != nil {
		return nil, err
	}
	return cachedTrie{tr, db}, nil
}

func (db *cachingDB) OpenStorageTrie(addrHash, root common.Hash) (Trie, error) {
	return trie.NewSecure(root, db.db, 0)
}
```

ContractCode and ContractCodeSize, ContractCodeSize has a cache.

```go
func (db *cachingDB) ContractCode(addrHash, codeHash common.Hash) ([]byte, error) {
	code, err := db.db.Get(codeHash[:])
	if err == nil {
		db.codeSizeCache.Add(codeHash, len(code))
	}
	return code, err
}

func (db *cachingDB) ContractCodeSize(addrHash, codeHash common.Hash) (int, error) {
	if cached, ok := db.codeSizeCache.Get(codeHash); ok {
		return cached.(int), nil
	}
	code, err := db.ContractCode(addrHash, codeHash)
	if err == nil {
		db.codeSizeCache.Add(codeHash, len(code))
	}
	return len(code), err
}
```

The structure and commit method of cachedTrie, when the commit is called, the pushTrie method is called to cache the previous Trie tree.

```go
// cachedTrie inserts its trie into a cachingDB on commit.
type cachedTrie struct {
	*trie.SecureTrie
	db *cachingDB
}

func (m cachedTrie) CommitTo(dbw trie.DatabaseWriter) (common.Hash, error) {
	root, err := m.SecureTrie.CommitTo(dbw)
	if err == nil {
		m.db.pushTrie(m.SecureTrie)
	}
	return root, err
}

func (db *cachingDB) pushTrie(t *trie.SecureTrie) {
	db.mu.Lock()
	defer db.mu.Unlock()
	// behave like a ring buffer, will remove the longest item of of the ring
	if len(db.pastTries) >= maxPastTries {
		copy(db.pastTries, db.pastTries[1:])
		db.pastTries[len(db.pastTries)-1] = t
	} else {
		db.pastTries = append(db.pastTries, t)
	}
}
```

## journal.go

Journal represents the operation log and provides a corresponding rollback function for the logs of various operations. Some transaction type operations can be done based on this log.

The type definition defines the interface of journalEntry and provides the function of undo. Journal is a list of journalEntry. It acts as redux in React ecosystem.

```go
type journalEntry interface {
	undo(*StateDB)
}

type journal []journalEntry
```

A variety of different log types and undo methods.

```go
createObjectChange struct {  // Create a log of the object. The undo method deletes the created object from the StateDB.
	account *common.Address
}
func (ch createObjectChange) undo(s *StateDB) {
	delete(s.stateObjects, *ch.account)
	delete(s.stateObjectsDirty, *ch.account)
}
// For the modification of stateObject, the undo method is to change the value to the original object.
resetObjectChange struct {
	prev *stateObject
}
func (ch resetObjectChange) undo(s *StateDB) {
	s.setStateObject(ch.prev)
}
// Suicide should be to delete the account, but if there is no commit, the object has not been deleted from the stateDB, otherwise, it will wait.
suicideChange struct {
	account     *common.Address
	prev        bool // whether account had already suicided
	prevbalance *big.Int
}
func (ch suicideChange) undo(s *StateDB) {
	obj := s.getStateObject(*ch.account)
	if obj != nil {
		obj.suicided = ch.prev
		obj.setBalance(ch.prevbalance)
	}
}

// Changes to individual accounts.
balanceChange struct {
	account *common.Address
	prev    *big.Int
}
nonceChange struct {
	account *common.Address
	prev    uint64
}
storageChange struct {
	account       *common.Address
	key, prevalue common.Hash
}
codeChange struct {
	account            *common.Address
	prevcode, prevhash []byte
}

func (ch balanceChange) undo(s *StateDB) {
	s.getStateObject(*ch.account).setBalance(ch.prev)
}
func (ch nonceChange) undo(s *StateDB) {
	s.getStateObject(*ch.account).setNonce(ch.prev)
}
func (ch codeChange) undo(s *StateDB) {
	s.getStateObject(*ch.account).setCode(common.BytesToHash(ch.prevhash), ch.prevcode)
}
func (ch storageChange) undo(s *StateDB) {
	s.getStateObject(*ch.account).setState(ch.key, ch.prevalue)
}

// refund process for DAO events - decentralized automous organization
refundChange struct {
	prev *big.Int
}
func (ch refundChange) undo(s *StateDB) {
	s.refund = ch.prev
}

// add log modification
addLogChange struct {
	txhash common.Hash
}

func (ch addLogChange) undo(s *StateDB) {
	logs := s.logs[ch.txhash]
	if len(logs) == 1 {
		delete(s.logs, ch.txhash)
	} else {
		s.logs[ch.txhash] = logs[:len(logs)-1]
	}
	s.logSize--
}

// This is to increase the original byte[] of SHA3 seen by the VM, and increase the correspondence between SHA3 hash -> byte[]
addPreimageChange struct {
	hash common.Hash
}
func (ch addPreimageChange) undo(s *StateDB) {
	delete(s.preimages, ch.hash)
}

touchChange struct {
	account   *common.Address
	prev      bool
	prevDirty bool
}
// mark to delete
var ripemd = common.HexToAddress("0000000000000000000000000000000000000003")
func (ch touchChange) undo(s *StateDB) {
	if !ch.prev && *ch.account != ripemd {
		s.getStateObject(*ch.account).touched = ch.prev
		if !ch.prevDirty {
			delete(s.stateObjectsDirty, *ch.account)
		}
	}
}
```

## state_object.go

stateObject represents the Ethereum account being modified.

data structure

```go
type Storage map[common.Hash]common.Hash

// stateObject represents an Ethereum account which is being modified.
// The usage pattern is as follows:
// First you need to obtain a state object.
// Account values can be accessed and modified through the object.
// Finally, call CommitTrie to write the modified storage trie into a database.

// The usage patterns are as follows:
// First you need to get a state_object.
// Account values can be accessed and modified by objects.
// Finally, call CommitTrie to write the modified storage trie to the database.

type stateObject struct {
	address  common.Address
	addrHash common.Hash // hash of ethereum address of the account
	data     Account  // This is the actual Ethereum account information.
	db       *StateDB // state database

	// DB error.
	// State objects are used by the consensus core and VM which are
	// unable to deal with database-level errors. Any error that occurs
	// during a database read is memoized here and will eventually be returned
	// by StateDB.Commit.
	//
	// Write caches.
	trie Trie // storage trie, which becomes non-nil on first access
	code Code // contract bytecode, which gets set when code is loaded

	cachedStorage Storage // Storage entry cache to avoid duplicate reads
	dirtyStorage  Storage // Storage entries that need to be flushed to disk

	// Cache flags.
	// When an object is marked suicided it will be delete from the trie
	// during the "update" phase of the state transition.
	dirtyCode bool // true if the code was updated
	suicided  bool
	touched   bool
	deleted   bool
	onDirty   func(addr common.Address) // Callback method to mark a state object newly dirty
}

// Account is the Ethereum consensus representation of accounts.
// These objects are stored in the main account trie.
type Account struct {
	Nonce    uint64
	Balance  *big.Int
	Root     common.Hash // merkle root of the storage trie
	CodeHash []byte
}
```

Constructor

```go
// newObject creates a state object.
func newObject(db *StateDB, address common.Address, data Account, onDirty func(addr common.Address)) *stateObject {
	if data.Balance == nil {
		data.Balance = new(big.Int)
	}
	if data.CodeHash == nil {
		data.CodeHash = emptyCodeHash
	}
	return &stateObject{
		db:            db,
		address:       address,
		addrHash:      crypto.Keccak256Hash(address[:]),
		data:          data,
		cachedStorage: make(Storage),
		dirtyStorage:  make(Storage),
		onDirty:       onDirty,
	}
}
```

The encoding method RLP will only encode the Account object.

```go
// EncodeRLP implements rlp.Encoder.
func (c *stateObject) EncodeRLP(w io.Writer) error {
	return rlp.Encode(w, c.data)
}
```

Some functions that change state

```go
// flush to state
func (self *stateObject) markSuicided() {
	self.suicided = true
	if self.onDirty != nil {
		self.onDirty(self.Address())
		self.onDirty = nil
	}
}

func (c *stateObject) touch() {
	c.db.journal = append(c.db.journal, touchChange{
		account:   &c.address,
		prev:      c.touched,
		prevDirty: c.onDirty == nil,
	})
	if c.onDirty != nil {
		c.onDirty(c.Address())
		c.onDirty = nil
	}
	c.touched = true
}
```

Storage processing

```go
// getTrie from database
func (c *stateObject) getTrie(db Database) Trie {
	if c.trie == nil {
		var err error
		c.trie, err = db.OpenStorageTrie(c.addrHash, c.data.Root)
		if err != nil {
			c.trie, _ = db.OpenStorageTrie(c.addrHash, common.Hash{})
			c.setError(fmt.Errorf("can't create storage trie: %v", err))
		}
	}
	return c.trie
}

// GetState returns a value in account storage from hash input.
func (self *stateObject) GetState(db Database, key common.Hash) common.Hash {
	value, exists := self.cachedStorage[key]
	if exists {
		return value
	}
	// Load from DB in case it is missing.
	enc, err := self.getTrie(db).TryGet(key[:])
	if err != nil {
		self.setError(err)
		return common.Hash{}
	}
	if len(enc) > 0 {
		_, content, _, err := rlp.Split(enc)
		if err != nil {
			self.setError(err)
		}
		value.SetBytes(content)
	}
	if (value != common.Hash{}) {
		self.cachedStorage[key] = value
	}
	return value
}

// SetState updates a value in account storage.
func (self *stateObject) SetState(db Database, key, value common.Hash) {
	self.db.journal = append(self.db.journal, storageChange{
		account:  &self.address,
		key:      key,
		prevalue: self.GetState(db, key),
	})
	self.setState(key, value)
}

func (self *stateObject) setState(key, value common.Hash) {
	self.cachedStorage[key] = value
	self.dirtyStorage[key] = value

	if self.onDirty != nil {
		self.onDirty(self.Address())
		self.onDirty = nil
	}
}
```

Commit trie

```go
// CommitTrie the storage trie of the object to dwb.
// This updates the trie root.
func (self *stateObject) CommitTrie(db Database, dbw trie.DatabaseWriter) error {
	self.updateTrie(db)
	if self.dbErr != nil {
		return self.dbErr
	}
	root, err := self.trie.CommitTo(dbw)
	if err == nil {
		self.data.Root = root
	}
	return err
}

// updateTrie writes cached storage modifications into the object's storage trie.
func (self *stateObject) updateTrie(db Database) Trie {
	tr := self.getTrie(db)
	for key, value := range self.dirtyStorage {
		delete(self.dirtyStorage, key)
		if (value == common.Hash{}) {
			self.setError(tr.TryDelete(key[:]))
			continue
		}
		// Encoding []byte cannot fail, ok to ignore the error.
		// default will return empty
		v, _ := rlp.EncodeToBytes(bytes.TrimLeft(value[:], "\x00"))
		self.setError(tr.TryUpdate(key[:], v))
	}
	return tr
}

// UpdateRoot sets the trie root to the current root hash of
func (self *stateObject) updateRoot(db Database) {
	self.updateTrie(db)
	self.data.Root = self.trie.Hash()
}
```

With some additional features, deepCopy provides a deep copy of state_object.

```go
func (self *stateObject) deepCopy(db *StateDB, onDirty func(addr common.Address)) *stateObject {
	stateObject := newObject(db, sel