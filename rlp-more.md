## Ethereum RLP coding

> RLP (Recursive Length Prefix), which is the encoding method used in the serialization of Ethereum. RLP is mainly used for network transmission and persistent storage of data in Ethereum.

### Why do you have to rebuild wheels?

There are many methods for object serialization, such as JSON encoding, but JSON has an obvious disadvantage: the encoding result is relatively large. For example, the following structure:

```go
type Student struct{
    Name string `json:"name"`
    Sex string `json:"sex"`
}
s := Student{Name:"icattlecoder", Sex:"male"}
bs,_ := json.Marsal(&s)
print(string(bs))
// {"name":"icattlecoder","sex":"male"}
```

Variable s has serialization result `{"name":"icattlecoder","sex":"male"}`, the length of the string is 35, the actual data is valid `icattlecoder`, and `male` a total of 16 bytes, we can see that too much redundant information is introduced when serializing JSON. Assuming Ethereum uses JSON to serialize, then the original 50GB blockchain may now be 100GB, which is double in size.

Therefore, Ethereum needs to design a coding method with smaller results.

### RLP encoding definition

RLP actually only encodes the following two types of data:

1. Byte array

2. An array of byte arrays, called a list

**Rule 1** : For a single byte whose value is between [0, 127], its encoding is itself.

Example 1: `a` The encoding is `97`.

**Rule 2** : If the byte array is long `l <= 55`, the result of the encoding is the array itself, plus the `128+l` prefix.

Example 2: The empty string encoding is `128`, ie `128 = 128 + 0`.

Example 3: The `abc` result of the encoding is `131 97 98 99`, in which `131=128+len("abc")`, `97 98 99` in order `a b c`.

**Rule 3** : If the array length is greater than 55, the first result of the encoding is the length of the encoding of 183 plus the length of the array, then the encoding of the length of the array itself, and finally the encoding of the byte array.

Example 4: Encode the following string:

```text
The length of this sentence is more than 55 bytes, I know it because I pre-designed it
```

This string has a total of 86 bytes, and the encoding of 86 requires only one byte, which is its own, so the result of the encoding is as follows:

```byte
184 86 84 104 101 32 108 101 110 103 116 104 32 111 102 32 116 104 105 115 32 115 101 110 116 101 110 99 101 32 105 115 32 109 111 114 101 32 116 104 97 110 32 53 53 32 98 121 116 101 115 44 32 73 32 107 110 111 119 32 105 116 32 98 101 99 97 117 115 101 32 73 32 112 114 101 45 100 101 115 105 103 110 101 100 32 105 116
```

The first three bytes are calculated as follows:

1. `184 = 183 + 1` Because the array length is `86` encoded and only takes up one byte.
2. `86` Array length is `86`
3. `84` the `T` character

**Rule 4** : If the list length is less than 55, the first bit of the encoding result is the length of the encoding of the 192 plus list length, and then the encoding of each sublist is sequentially connected.

Note that rule 4 itself is recursively defined.
Example 6: `["abc", "def"]` The result of the encoding is `200 131 97 98 99 131 100 101 102`.
Where in `abc` the encoded `131 97 98 99`, `def` encoding is `131 100 101 102`. The total length of the two encoded sub-strings is 8, so the encoding results of the calculated one: `192 + 8 = 200`.

**Rule 5** : If the list length exceeds 55, the first digit of the encoding result is the encoding length of 247 plus the length of the list, then the encoding of the length 