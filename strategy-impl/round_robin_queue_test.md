Round-Robin Queue Tests
=======================

```{.go .lib}
package decision

import (
    "testing"

    peer "gx/ipfs/QmXYjuNuxVzXKJCfWasQk1RqkhVLDM9jtUKhqc2WPQmFSB/go-libp2p-peer"
    testutil "gx/ipfs/QmQgLZP9haZheimMHqqAjJh2LhRmNfEoZDfbtkpeMhi9xK/go-testutil"
)
```

Peer Management
---------------

### TestWeightAllocations*

`TestWeightAllocationsConstant` runs tests that all peers are allocated the same
amount of resources when they all have the same weight.

```{.go .lib}
func TestWeightAllocationsConstant(t *testing.T) {
```

Create a new RRQ, and an array to stored the expected allocations for each peer.

```{.go .lib}
    roundBurst := 50
    rrq := newRRQueueCustom(roundBurst, ProbSend)
    numPeers := 5
```

Each peer will have their ledger's `Value` set to `1`, which means all peers
will receive an equal portion of the allocation.

```{.go .lib}
    expected := int(float64(roundBurst) / float64(numPeers))
```

Create peers and add their weight to the RRQ.

```{.go .lib}
    for i := 0; i < numPeers; i++ {
        // get random peer ID
        peerID := testutil.RandPeerIDFatal(t)
        // get a cid
        //cid := cid.NewCidV0(u.Hash([]byte(fmt.Sprint(i))))
        // generate new ledger, and set its value to 1
        receipt := newLedger(peerID).Receipt()
        receipt.Value = float64(1)

        rrq.updateWeight(peerID, receipt)
    }
```

Initialize the round, which calculates the allocation for each peer. Then check
that the resulting allocations are as expected.

```{.go .lib}
    rrq.initRound()

    if len(rrq.allocations) != numPeers {
        t.Fatalf("Expected %d allocations, got %d", numPeers,
                 len(rrq.allocations))
    }
    for i, rrp := range rrq.allocations {
        if rrp.allocation != expected {
            t.Fatalf("Bad allocation: peer %d -- expected %d, got %d",
                     i, expected, rrp.allocation)
        }
    }
}
```

`TestWeightAllocationsVarying` tests that peers receive the correct allocations
with varying weights (specifically, a linear increasing set of weights
containing all integers in the range `[0, numPeers - 1]`).

```{.go .lib}
func TestWeightAllocationsVarying(t *testing.T) {
    roundBurst := 50
    rrq := newRRQueueCustom(roundBurst, ProbSend)
    numPeers := 5
    expected := make(map[peer.ID]int)

    for i := 0; i < numPeers; i++ {
        // get random peer ID
        peerID := testutil.RandPeerIDFatal(t)
        // generate new ledger, and set its value to 1
        receipt := newLedger(peerID).Receipt()
        receipt.Value = float64(i)

        rrq.updateWeight(peerID, receipt)

        allocation := int(float64(i * roundBurst) / float64(sum1ToN(numPeers-1)))
        expected[peerID] = allocation
    }
    rrq.initRound()

    // check that the correct number of peers were allocated (numPeers - 1
    // because one peer has a weight of 0)
    if len(rrq.allocations) != numPeers - 1 {
        t.Fatalf("Expected %d allocations, got %d", numPeers - 1,
                 len(rrq.allocations))
    }

    // check that all peers received the correct allocation
    for _, val := range rrq.allocations {
        if expected[val.id] != val.allocation {
            t.Fatalf("Bad allocation: peer %s -- expected %d, got %d",
                     val.id, expected[val.id], val.allocation)
        }
    }
}
```

Helper Functions
----------------

```{.go .lib}
func sum1ToN(n int) int {
    sum := 0
    for i := 1; i <= n; i++ {
        sum += i
    }
    return sum
}
```
