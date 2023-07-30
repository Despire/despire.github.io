---
layout: post
title: "Merkle Tree"
date: 2023-07-29
categories: [Tech]
---

# Authenticated Data structures

Given a sequence of items 

$$(x_1, x_2 ... x_n) := X$$

an autheticated data structure is used to compute the hash $$ H(X) $$. Given the hash we can then prove properties using $$X$$. A Merkle Tree
is a form of an authenticated data structure on which we can prove membership and non-membership properties.[^2]

# Merkle Trees

> The description will focus on binary merkle trees where the number of children is equal to 2. It is possible to create k-nary merkle trees.

Given $$X$$ we construct a merkle tree as follows. The individual elements $$x_i \in X$$ are treated as leaf nodes from which a hash $$H(x_i)$$ is constructed.
Given the base of the tree we then proceed by constructing parents for pairs of node $$n_1, n_2 \in N$$. Let $$h_i$$ denote the hash of node $$n_i$$, we then create the parent for the pair by concataneting their
hashes and rehashing the result $$H(h_i, h_{i+1})$$. This process is repeated until only a single node is left for which the hash is the combination of all the hashes of its children. The resulting tree is depicted
by the following figure.[^1]

![alt text](/assets/img/merkle-tree/merkle-tree.png)

# Membership Proof

Let us now work with the merkle tree to provide a proof that a specific $$x_6 \in X$$ is part of the structure. Let $$(y_{5},y_{12},y_{13}) := T $$ be the set of hashes of the siblings on
the path until the root node of the tree. One can then verify that $$x_6$$ is part of the structure by computing

$$y^{'}_{6} := H(x_{6})$$

$$y^{'}_{11} := H(y_{5}, y^{'}_{6})$$

$$y^{'}_{14} := H(y^{'}_{11}, y_{12})$$

$$y^{'}_{15} := H(y_{13}, y^{'}_{14})$$

If $$y^{'}_{15}$$ equal to $$y_{15}$$ then the verifier can accept that $$x_6$$ is in fact part of the structure.

# A Simple Implementation In Go

The full implementation can be accessed on [this github repository](https://github.com/Despire/merkle-tree/blob/main/merkle/tree.go)

Let us start by first constructing the tree structure from a set of values.

```go
// NewTree constructs a new MerkleTree from the given values.
func NewTree(values [][]byte) *Tree {
    root, leaves := construct(values)
    return &Tree{
        Root:   root,
        Leaves: leaves,
    }
}

func construct(values [][]byte) (*Node, []*Node) {
    leaves := make([]*Node, len(values))

    for i := range leaves {
        h := sha512.Sum512(values[i])

        leaves[i] = &Node{
            Hash: h[:],
        }
    }

    // if the number of leaves is odd we add a duplicate
    // of the last value.
    if len(leaves)%2 == 1 {
        h := sha512.Sum512(values[len(values)-1])

        leaves = append(leaves, &Node{
            Hash: h[:],
        })
    }

    return root(leaves), leaves
}

func root(queue []*Node) *Node {
    // recursively construct the parents of the pairs until only
    // a single node remains which is the root of the merkle tree.
    for len(queue) != 1 {
        left, right := queue[0], queue[1]
        queue = queue[2:]

        h := sha512.Sum512(append(left.Hash, right.Hash...))

        node := &Node{
            Parent: nil,
            Left:   left,
            Right:  right,
            Hash:   h[:],
        }

        left.Parent = node
        right.Parent = node

        queue = append(queue, node)
    }

    return queue[0]
}
```

Next the implementation for the proof of the presence of a given value

```go
type PathPoint struct {
    Hash     []byte
    Appended bool
}

// Proof builds a proof for a MerkleTree
func (tree *Tree) Proof(h []byte) ([]PathPoint, error) {
    if tree == nil {
        return nil, errors.New("empty tree")
    }

    current := findNodeWithHash(tree.Leaves, h)
    if current == nil {
        return nil, errors.New("no node with such hash")
    }

    var path []PathPoint

    // collect all the siblings until the root is reached.
    parent := current.Parent
    for parent != nil {
        if current == parent.Left {
            path = append(path, PathPoint{
                Hash:     parent.Right.Hash,
                Appended: true,
            })
        } else {
            path = append(path, PathPoint{
                Hash:     parent.Left.Hash,
                Appended: false,
            })
        }

        current = parent
        parent = current.Parent
    }

    return path, nil
}
}
```

We can then use the merkle tree to get proofs and verify them.

```go
func main() {
    tree := merkle.NewTree([][]byte{
        []byte("x1"),
        []byte("x2"),
        []byte("x3"),
        []byte("x4"),
    })

    // obtain proof for x2
    block := sha512.Sum512([]byte("x2"))

    proof, err := tree.Proof(block[:])
    if err != nil {
        panic(err)
    }

    fmt.Println(tree.Verify())
    fmt.Println(tree.VerifyProof(block[:], proof))

    // obtain proof for x5
    block = sha512.Sum512([]byte("x5"))
    _, err = tree.Proof(block[:])
    fmt.Println(err) // no node with such hash
}

```


**Footnotes**

[^1]: [Chapter 8.9 Merkle Trees: proving properties of a hashed list](https://toc.cryptobook.us/book.pdf)
[^2]: [Chapter 8.9.1 Authenticated data structures](https://toc.cryptobook.us/book.pdf)
