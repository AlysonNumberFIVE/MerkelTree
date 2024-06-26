# MerkelTree

## Table of contents
- API reference
- How to run it
- Documentation
- Author


## API reference
This is a repository for a Merket Tree implementation project.  The following operations are implemented:
- Get/Lookup
- Put/Insert
- Update
- GenerateProof
- VerifyProof

### Main Merkel Tree data structures
`main.go`:
```
type MerkelTree struct {
	root           *Node
	lookupNodeList map[string]*Mapping
}
```
Holds the root node as well as a hashmap chain lookup list for easy leaf node searching.
```
type Mapping struct {
	node              *Node
	hashUpdateHistroy [][]byte
}
```
Mapping holds a pointer to the leaf node referenced by the hash key passed into `lookupNodeList`.

`node.go`:
```
type Node struct {
	left  *Node
	right *Node
	prev  *Node
	data  []byte
	hash  []byte
}
```
Holds node information for leaf and branch nodes. A bidiretional node with 2 children and 1 parent/prev node.

<br>
<br>
### Lookup
`main.go`:
```
func (merkelTree *MerkelTree) Lookup(hash []byte) (*Node, error)
```
Returns a pointer to the leaf node with the requested `hash` and `error` if the hash is invalid/data doesn't exist

### Insert
`main.go`:
```
func (merkelTree *MerkelTree) Insert(data []byte) ([]byte, error)
```
Creates a new record/node in the tree. It returns a hash if successful and an `error` on failure.<br>
`Insert` must not be used to update existing data. Submitting the same piece of data will throw a `rather use Update()` error
as the hash is created from `data`.<br><br>
In a production Merkel Tree, this duplicate condition wouldn't be possible as data would be tied to a unique entry timestamp, making a duplicate insert impossible (unless it's under malicious intent).

### Update
`main.go`:
```
func (merkelTree *MerkelTree) Update(newData, hash []byte) ([]byte, error)
```
`Update` takes in `newData` that will be overwriting the data that exists at `hash` and return a new `hash` for the new data.<br><br>
`Update` has support for stale hashes. If the data at `hash` has been updated more than once, all historical `hash`s that node has always had will be valid for lookup as the lookup structure uses a <a href="https://en.wikibooks.org/wiki/Data_Structures/Hash_Tables">chained hashmap</a> to preserve historical hashes.

### MerkelProof data structures:
`proof.go`:
```
type MerkelProof struct {
	LeafHash   []byte
	ProofList  [][]byte
	Directions []bool
}
```
Struct for the merkel proof itself. The proof list holds the hashes of all the nodes required to create the proof, the directions taken on the way up the tree (important for proof verification) as well as the original leaf node for this proof.


### GenerateProof
`proof.go`:
```
func (merkelTree *MerkelTree) GenerateProof(leafHash []byte) (*MerkelProof, error)
```
`GenerateProof` creates a new proof for the node referenced by `leafHash`. It returns a `MerkelProof` if successful or `error` otherwise; in cases of an invalid hash that fails during inital lookup.

### VerifyProof
`proof.go`:
```
func VerifyProof(proof *MerkelProof, rootHash []byte) bool
```
Verifies the validity of a `MerkelProof` by using the Merkel Tree verification algorithm.

## How to run it.
from `main.go`
```
func main() {

	tree := InitMerkelTree()
	fmt.Println("First -- insert A")
	hashA, err := tree.Insert([]byte("A"))
	if err != nil {
		log.Fatal(err)
	}
	tree.Visualizer(tree.root, "", false)

	fmt.Println("Second -- insert B")
	hashB, err := tree.Insert([]byte("B"))
	if err != nil {
		log.Fatal(err)
	}
	tree.Visualizer(tree.root, "", false)

	fmt.Println("Third -- insert duplicate A")
	_, err = tree.Insert([]byte("B"))
	fmt.Printf("Error : %+v\n", err)
	tree.Visualizer(tree.root, "", false)

	fmt.Println("Fourth -- insert C, D, E")
	tree.Insert([]byte("C"))
	tree.Insert([]byte("D"))
	tree.Insert([]byte("E"))
	tree.Visualizer(tree.root, "", false)

	fmt.Println("Fifth -- update A to F")
	hashF, err := tree.Update([]byte("F"), hashA)
	if err != nil {
		log.Fatal(err)
	}
	tree.Visualizer(tree.root, "", false)

	fmt.Println("Sixth -- update F to A again (use A's old hash)")
	tree.Update([]byte("A"), hashA)
	tree.Visualizer(tree.root, "", false)

	fmt.Println("Seventh -- update A to M (use F's hash this time)")
	_, err = tree.Update([]byte("M"), hashF)
	if err != nil {
		log.Fatal(err)
	}
	tree.Visualizer(tree.root, "", false)

	fmt.Println("Eighth -- lookup M using A's hash")
	nodeM, err := tree.Lookup(hashA)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("data is ", string(nodeM.data))

	fmt.Println("Ninth -- lookup M using F's hash")
	nodeM, err = tree.Lookup(hashF)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("data is ", string(nodeM.data))

	nodeD, err := tree.Lookup(Hash128([]byte("D")))
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("data is ", string(nodeD.data))

	proofA, err := tree.GenerateProof(hashF)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("proofA is ", VerifyProof(proofA, tree.root.hash))

	proofA, err = tree.GenerateProof(hashA)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("proofA is ", VerifyProof(proofA, tree.root.hash))

	proofNull, _ := tree.GenerateProof(Hash128([]byte("4444")))
	fmt.Println("proofNull is ", VerifyProof(proofNull, tree.root.hash))

	proofB, err := tree.GenerateProof(hashB)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("proofB is ", VerifyProof(proofB, tree.root.hash))
	tree.GenerateProof(nil)
	tree.Insert(nil)
	tree.Update(nil, nil)
	tree.Lookup(nil)
	VerifyProof(nil, nil)
}
```

Running this code will give you the following terminal output.

```
First -- insert A
└── A
Second -- insert B
    └── B
└── Y
    ├── A
Third -- insert duplicate A
Error : Hash already exists. Use Update() to update an existing hash
    └── B
└── Y
    ├── A
Fourth -- insert C, D, E
        └── B
    └── X
        ├── D
└── Y
    │   └── A
    ├── X
    │   │   └── C
    │   ├── X
    │   │   ├── E
Fifth -- update A to F
        └── B
    └── X
        ├── D
└── Y
    │   └── F
    ├── X
    │   │   └── C
    │   ├── X
    │   │   ├── E
Sixth -- update F to A again (use A's old hash)
        └── B
    └── X
        ├── D
└── Y
    │   └── A
    ├── X
    │   │   └── C
    │   ├── X
    │   │   ├── E
Seventh -- update A to M (use F's hash this time)
        └── B
    └── X
        ├── D
└── Y
    │   └── M
    ├── X
    │   │   └── C
    │   ├── X
    │   │   ├── E
Eighth -- lookup M using A's hash
data is  M
Ninth -- lookup M using F's hash
data is  M
data is  D
proofA is  true
proofA is  true
proofNull is  false
proofB is  true
```

Accompanying code inside of `merkel_tree_test.go` has extensive usecases.



## Documentation

An accompanying PDF design document accompanies this code and must be treated as a helper for reading this code <b> and not a substitute for going through the code</b>.

## Authored by Alyson Ng
