It is recommended to first understand the Appendix B. Recursive Length Prefix in the Yellow Book.

More detail can refer to: [RLP in detail](rlp-more.md)

From an engineering point of view, rlp is divided into two types of data definitions, and one of them can recursively contain another type:

// T consists of L or B
$$T ≡ L∪B$$
// Any member of L belongs to T (T is also composed of L or B: note recursive definition)
$$L ≡ {t:t=(t[0],t[1],...) ∧ \forall n<‖t‖ t[n]∈T}$$  
// Any member of T belongs to O
$$B ≡ {b:b=(b[0],b[1],...) ∧ \forall n<‖b‖ b[n]∈O}$$

- [\forall](https://en.wikibooks.org/wiki/LaTeX/Mathematics#Symbols) reference LaTeX syntax standard, can use Katex in visualcode to view
- O is defined as a bytes collection
- If you think of T as a tree-like data structure, then B is the leaf, which contains only the byte sequence structure; and L is the trunk, containing multiple Bs or itself.

That is, we rely on the basis of B to form a recursive definition of T and L: the whole RLP consists of T, T contains L and B, and the members of L are both T. Such recursive definitions can describe very flexible data structures

In the specific coding, only the coding space of the first byte can be used to distinguish these structural differences:

```
B coding rules: leaves
RLP_B0 [0000 0001, 0111 1111] If it is a byte other than 0 and less than 128[1000 0000], no header is needed, and the content is encoded.
RLP_B1 [1000 0000, 1011 0111] If the length of the byte content is less than 56, that is, 55[0011 0111], the length is compressed into a byte in big endian, and 128 is added to form the header, and then the actual content is connected.
RLP_B2 (1011 0111, 1100 0000) For longer content, describe the length of the content length in a space where the second bit is not 1. Its space is (192-1)[1011 1111]-(183+1)[1011 1000]=7[0111] ，That is, the length needs to be less than 2^(7*8) is a huge number that cannot be used up.
L coding rules: branches
RLP_L1 [1100 0000, 1111 0111) If it is a combination of multiple above encoded content, it is expressed by 1 in the second bit. Subsequent content length is less than 56, that is,55[0011 0111] 则先将长度压缩后放到第一个 byte Medium (plus 192 [1100 0000]), then connect to the actual content
RLP_L2 [1111 0111, 1111 1111] For longer content, the length of the content length is described in the remaining space. Its space is 255[1111 1111]-247[1111 0111]=8[1000]，That is, the length needs to be less than 2^(8*8). It is also a huge number that cannot be used up.
```

Please copy the following code into the editor with text folding for code review (Recommended Notepad++ to properly collapse all functions under the github reference format for easy global understanding)

```go
// [/rlp/encode_test.go#TestEncode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode_test.go#L272)
func TestEncode(t *testing.T) {
	runEncTests(t, func(val interface{}) ([]byte, error) {
		b := new(bytes.Buffer)
		err := Encode(b, val)
		// [/rlp/encode.go#Encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L80)
		func Encode(w io.Writer, val interface{}) error {
			if outer, ok := w.(*encbuf); ok {
				// Encode was called by some type's EncodeRLP.
				// Avoid copying by writing to the outer encbuf directly.
				return outer.encode(val)
			}
			eb := encbufPool.Get().(*encbuf)
			// [/rlp/encode.go#encbuf](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L121)
			type encbuf struct { // Stateful encoder
				str     []byte      // string data, contains everything except list headers
				lheads  []*listhead // all list headers

				// [/rlp/encode.go#listhead](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L128)
				type listhead struct {
					offset int // index of this header in string data
					size   int // total size of encoded data (including list headers)
				}

				lhsize  int         // sum of sizes of all encoded list headers
				sizebuf []byte      // 9-byte auxiliary buffer for uint encoding
			}

			defer encbufPool.Put(eb)
			eb.reset()
			if err := eb.encode(val); err != nil { // encbuf.encode As a B-encoded (internal) function, eb is a stateful encbuf
				// [/rlp/encode.go#encbuf.encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L181)
				func (w *encbuf) encode(val interface{}) error {
				rval := reflect.ValueOf(val)
				ti, err := cachedTypeInfo(rval.Type(), tags{}) // Get the current type of content encoding function from the cache
				if err != nil {
					return err
				}
				return ti.writer(rval, w) // Execution function
				// [/rlp/encode.go#writeUint](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L392)
				func writeUint(val reflect.Value, w *encbuf) error {
					i := val.Uint()
					if i == 0 {
						w.str = append(w.str, 0x80) // Prevent all 0 exception patterns in the sense of encoding, encoding 0 as 0x80
					} else if i < 128 { // Implement RLP_B0 encoding logic
						// fits single byte
						w.str = append(w.str, byte(i))
					} else { // Implement RLP_B1 encoding logic: because uint has a byte length of only 8 and will not exceed 56
						// TODO: encode int to w.str directly
						s := putint(w.sizebuf[1:], i) // The bit with the uint high is zero, removed by the byte granularity, and returns the