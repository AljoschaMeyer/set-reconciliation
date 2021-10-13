**A much more elaborate write-up is at [https://github.com/AljoschaMeyer/master_thesis](https://github.com/AljoschaMeyer/master_thesis)**

# Simple And Efficient Set Reconciliation

Imagine two computers connected over a network, and each computer holds a set of values. Set reconciliation is the problem of efficiently exchanging messages between them such that in the end both hold the union of the two sets. If there is a total order on the items, and if items can be hashed to smaller fingerprints with negligible collision probability, this can be done in logarithmic time in a fairly simple manner.

## Communication Protocol

Assume first that the items an endpoint holds are sorted according to the total order. Assume further that we can efficiently compute fingerprints for any subrange of the sorted values (we'll see how to do so later). We define the fingerprint of an empty range to be 0. These building blocks allow us to create an efficient message-passing protocol based on exchanging fingerprints for smaller and smaller subranges.

When an endpoint receives a fingerprint for a range (i.e. the value of the fingerprint as well as information on where the range begins and ends), it quickly computes the fingerprint over its local items in the same range. This leads to three different cases. If the local fingerprint equals the received fingerprint, this subrange is already reconciled, the endpoint merely replies that no further work is necessary. Else, if the received fingerprint is 0, the endpoint replies with all the items it holds in that range. Otherwise, the endpoint splits the range into two subranges and sends fingerprints for both of them. An endpoint can initiate the exchange by sending the fingerprint over the full range, or more elegantly by acting as if it had just received such a fingerprint that didn't match the local fingerprint over the full range.

The messages can be bundled in rounds. In the first round, metadata about two ranges would be sent. The second round would consist of a reply concerning up to four ranges, the next reply would concern up to eight ranges, and so on. The protocol thus terminates after O(log(n)) rounds of communication, where n is the size of the smaller set. In the worst case, message size doubles in each round, for a total of O(n) pieces of transmitted data. But this worst-case behavior only occurs if the difference between the sets has size O(n) as well, in which case transmitting O(n) pieces of data cannot be avoided anyways.

Instead of splitting ranges into two parts, they can just as well be split into more than two ones. This reduces the number of rounds by increasing the base of the logarithm.

Each message pertains to a number of ranges. These ranges can be encoded efficiently by merely specifying the smallest item of each range. For example, if the items are natural numbers, a message might look like `some-fingerprint 15 another-fingerprint 19 0-hash 24 some-numbers 30 done 38 some-more-numbers`. This would indicate that the fingerprint over all locally available numbers strictly less than `15` was `some-fingerprint`, the fingerprint over all locally available numbers greater than or equal to `15` and strictly less than `19` was `another-fingerprint`, no numbers between 19 and 24 were available, some numbers between 24 and 30 would be sent (because in the previous round the other endpoint sent a 0 fingerprint), no more communication would be necessary for dealing with any numbers between 30 and 38, and some more numbers greater than or equal to 38 would be sent.

## Computing Fingerprints

While the above protocol is sufficiently efficient in terms of communication complexity, we also need to be able to compute the messages efficiently - in particular we need to compute fingerprints for arbitrary ranges efficiently. The following fingerprint definition presents a fairly simple solution: Fix a hash function. The fingerprint of a range is the XOR (exclusive or) of the hashes of all the items in the range. If the range only contains one item, the fingerprint is simply the hash of that item. If the range is empty, the fingerprint is 0.

To be able to quickly compute these fingerprints, we store the set of all locally available values in a self-balancing binary search tree. In each node, we store the fingerprint of the right subtree. Note that this information can be maintained during insertion and deletion operations without impacting the time complexity of the balanced search tree. A child node stores 0 instead.

The fingerprint for a range starting at item `s` (inclusive) and ending at item `t` (exclusive) is computed as follows: First, trace a path from the root node to `t`. XOR the right-subtree-fingerprints stored in all nodes whose item is greater than or equal to `t`, we call the resulting value `T`. `T` is the fingerprint over all values >= `t`. Then, do the same for `s`, obtaining `S`, the fingerprint over all values >= `s`. The fingerprint for the range from `s` to `t` is simply the XOR of `S` and `T`. All values >= `t` are included twice in that XOR and thus cancel each other out, leaving only the values we care about.

Instead of using XOR, one can use any binary operation `+` that forms a *transitive group* to define the protocol, i.e. an operation that satisfies the following properties:

- associativity: `(a + b) + c == a + (b + c)` for all `a`, `b`, and `c`, otherwise different trees containing the same items might compute different fingerprints.
- neutral element: there is an element `0` such that `0 + a == a == a + 0` for all `a`, the neutral element is used as the fingerprint for empty ranges.
- inverses: for all `a` there is an element `-a` such that `a + -a == 0 == -a + a`, this is used to obtain the subrange fingerprint from `S` and `T`. `S XOR T` merely happens to be the same as `T + -S`, which is the more general form.
- transitivity: for all `a` and `c`, there exists an element `b` such that `a + b == c`, this property makes sure that the probability of fingerprint collisions stays small.

The implementation via binary search trees can be extended to use the much more efficient B-trees instead without requiring any protocol changes. The kind of tree (or even whether a tree is used at all) remains an implementation detail.

## Fun With Ranges

Beyond full set reconciliation, the range-based approach allows for some neat tricks. For example, one can reconcile only certain subranges of the full set by simply indicating that the other parts don't need more communication (the `done` part in the message example above) in the initial message.

One can also emulate the efficient replication of append-only-logs. When replicating append-only-logs with a dedicated protocol, two endpoints merely need to tell each other about the index of the newest log entry they have stored locally, the endpoint with the newer entries can send the difference to the other endpoint. The set reconciliation protocol can emulate this if log entries are sorted by their sequence number:

The initiator sends a message with the fingerprint over all locally available entries and then a 0 hash for all entries of larger sequence number. If the other endpoint has more entries available, it responds with all of them. Otherwise, the other endpoint can reply with the same sort of message, after which the initiating endpoint knows which entries to transmit. While this requires implementations to handle the first message they send in a special way for append-only-logs, the message protocol itself can remain unchanged. If the responding endpoint has fewer entries and does not implement special handling for append-only-logs, the protocol degrades gracefully to replicate in logarithmic time.

The protocol can also be extended to handle higher dimensional range queries, simply by using kd-trees in the endpoints.

## Related Work

- [Minsky, Yaron, and Ari Trachtenberg. "Practical set reconciliation." 40th Annual Allerton Conference on Communication, Control, and Computing. Vol. 248. 2002.](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.456.7200&rep=rep1&type=pdf): Proposes a protocol with similar properties. While they are slightly more efficient in bandwidth usage, they require more complicated techniques, and more importantly, their tree construction is not self-balancing.
- [Eppstein, David, et al. "What's the difference? Efficient set reconciliation without prior context." ACM SIGCOMM Computer Communication Review 41.4 (2011): 218-229.](citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.220.6282&rep=rep1&type=pdf): One of several papers that use more complicated randomized algorithms for performing set reconciliation in a single communication round.
- [CCNx Sync Protocol](https://github.com/ProjectCCNx/ccnx/blob/master/doc/technical/SynchronizationProtocol.txt): Uses the same kind of fingerprints (with wrapping addition rather than XOR) as well as a tree construction. These concepts have probably been reinvented independently in a bunch of other contexts as well.

## Acknowledgments

Zenna Fiscella brought up the idea of multidimensional range queries, and Jan Winkelmann immediately suggested kd-trees as a solution.
