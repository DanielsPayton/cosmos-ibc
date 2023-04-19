---
ics: 33
title: Multi-hop Channel
stage: draft
required-by: 4
category: IBC/TAO
kind: instantiation
author: Derek Shiell <derek@polymerlabs.org>, Wenwei Xiong <wenwei@polymerlabs.org>, Bo Du <bo@polymerlabs.org>
created: 2022-11-11
modified: 2022-12-16
---

## Synopsis

This document describes a standard for multi-hop IBC channels. Multi-hop channels specifies a way to route messages across a path of IBC enabled blockchains utilizing multiple pre-existing IBC connections.

### Motivation

The current IBC protocol defines messaging in a point-to-point paradigm which allows message passing between two directly connected IBC chains, but as more IBC enabled chains come into existence there becomes a need to relay IBC packets across chains because IBC connections may not exist between the two chains wishing to exchange messages. IBC connections may not exist for a variety of reasons which could include economic inviability since connections require client state to be continuously exchanged between connection ends which carries a cost.

### Definitions

Associated definitions are as defined in referenced prior standards (where the functions are defined), where appropriate.

`Connection` is as defined in [ICS 3](https://github.com/cosmos/ibc/tree/main/spec/core/ics-003-connection-semantics).

`Channel` is as defined in [ICS 4](https://github.com/cosmos/ibc/tree/main/spec/core/ics-004-channel-and-packet-semantics).

`Channel Path` is defined as the path of connection IDs along which a channel is defined.

`Connection Hop` is defined as the connection ID of the connection between two chains along a channel path.

### Desired Properties

- IBC channel handshake and message packets should be able to be utilize pre-existing connections to form a logical proof chain to relay messages between unconnected chains.
- Relaying for a multi connection IBC channel should NOT require additional writes to intermediate hops.
- Minimal additional required state and changes to core and app IBC specs.
- Retain desired properties of connection, channel and packet definitions.
- Retain backwards compatibility for messaging over a single connection hop.

## Technical Specification

The bulk of the spec will be around proof generation and verification. IBC connections remain unchanged. Additionally, channel handshake and packet message types as well as general round trip messaging semantics and flow will remain the same. There is additional work on the verifier side on the receiving chain as well as the relayers who need to query for proofs.

Messages passed over multiple hops require proof of the connection path from source chain to receiving chain as well as the packet commitment on the source chain. The connection path is proven by verifying the connection state and consensus state of each connection in the path to the receiving chain. On a high level, this can be thought of as a channel path proof chain where the receiving chain can prove a key/value on the source chain by iteratively proving each connection and consensus state in the channel path starting with the consensus state associated with the final client on the receiving chain. Each subsequent consensus state and connection is proven until the source chain's consensus state is proven which can then be used to prove the desired key/value on the source chain.

### Channel Handshake and Packet Messages

For both channel handshake and packet messages, additional connection hops are defined in the pre-existing `connectionHops` field. The connection IDs along the channel path must be pre-existing and in the `OPEN` state to guarantee delivery to the correct recipient. See `Path Forgery Protection` for more info.

The spec for channel handshakes and packets remains the same. See [ICS 4](https://github.com/cosmos/ibc/tree/main/spec/core/ics-004-channel-and-packet-semantics).

In terms of connection topology, a user would be able to determine a viable channel path from sender -> receiver using information from the [chain registry](https://github.com/cosmos/chain-registry). They can also independently verify this information via network queries.

### Multihop Relaying

Relayers would deliver channel handshake and IBC packets as they currently do except that they are required to provide proof of the channel path. Relayers would scan packet events for the connectionHops field and determine if the packet is multi-hop by checking the number of hops in the field. If the number of hops is greater than one then the packet is a multi-hop packet and will need extra proof data.

For each multi-hop channel (detailed proof logic below):

1. Scan source chain for IBC messages to relay.
2. Read the connectionHops field in from the scanned message to determine the channel path.
3. Lookup connection endpoints via chain registry configuration and update the clients associated with the connections in the channel path to reflect the latest consensus state on the sending chain including the key/value to be proven.
4. Query for proof of connection, and consensus state for each intermediate connection in the channel path.
5. Query proof of packet commitment or handshake message commitment on source chain.
6. Submit proofs and data to RPC endpoint on receiving chain.

Relayers are connection topology aware with configurations sourced from the [chain registry](https://github.com/cosmos/chain-registry).

### Proof Generation & Verification

Graphical depiction of proof generation.

![proof_generation.png](proof_generation.png)

Relayer multi-hop proof queries.

![proof_query.png](proof_query.png)

Multi-hop proof verfication logic.

![proof_verification.png](proof_verification.png)

Pseudocode proof generation for a channel between `N` chains `C[0] --> C[i] --> C[N]`

```go

// Proof generation helper functions
//
// GetClientStateHeight returns the client state height
func (Chain*) GetClientStateHeight() exported.Height
// GetConnection returns the connection end for the preceding chain in the channel path
func (Chain*) GetConnection() ConnectionEnd
// GetConsensusStateAtHeight returns the consensus state for the given client id at the specified height.
func (Chain*) GetConsensusStateAtHeight(clientID string, height exported.Height) exported.ConsensusState
// QueryProofAtHeight returns a proof of the key at the given height and returns the height at which the proof will succeed.
func (Chain*) QueryProofAtHeight(key string, height int64) (proof []byte, height exported.Height)


// ProofData is a generic proof struct.
type ProofData struct {
    Key   *MerklePath
    Value []]byte
    Proof []byte
}

// MultihopProof defines set of proofs to verify a multihop message.
// Consensus and Connection proofs are ordered from receiving to sending chain but do not including
// the chain[1] consensus/connection state on chain[0] since it is already known on the receiving chain.
type MultihopProof struct {
    KeyProofIndex uint32           // the index of the consensus state to prove the key/value on indexed from sending chain
                                   // KeyProofIndex = 0 means the key/value is proven on the sending chain (chain[N])
    KeyProof *ProofData            // the key/value proof on the on chain[KeyProofIndex] in the channel path
    ConsensusProofs []*ProofData   // array of consensus proofs starting with proof of consensusState of chain[1] on chain[2]
    ConnectionProofs []*ProofData  // array of connection proofs starting with proof of conn[1,2] on chain[2]
}

// GenerateMultihopProof generates proof of a key/value at the proofHeight on indexed chain (chain0).
// Chains are provided in order from the sending (source) chain to the receiving (verifying) chain.
// It is assumed that all clients, from `chain[N]` to `chain[1]` have been updated, in this order, after the `key/value` has
// been stored on `chain[N]`
func GenerateMultihopProof(chains []*Chain, key string, value []byte, proofHeight exported.Height) *MultihopProof {

    abortTransactionUnless(len(chains) > 2)

    // generate and assign consensus, clientState, and connection proofs
    var proof MultihopProof
    proof.ConsensusProofs = GenerateConsensusProofs(chains)
    proof.ConnectionProofs = GenerateConnectionProofs(chains)

    N := len(chains) - 1
    chainA := chains[N]   // source chain
    chainB := chains[N-1] // next hop chain

    // check a consensus state at proof height is present on chain B
    consensusStateAtProofHeight := chainB.GetConsensusStateAtHeight(chainA.ClientID, proofHeight)
    abortTransactionUnless(consensusStateAtProofHeight != nil)

    // query the key/value proof on the indexed chain at the proof height
    keyProof, _ := chainA.QueryProofAtHeight(key, proofHeight)

    // assign the key/value proof
    proof.KeyProof = &ProofData{
        Key:   nil,  // key to prove constructed during verification
        Value: nil,  // proven values are constructed during verification (except for frozen client proofs)
        Proof: keyProof,
    }

    return &proof
}

// GenerateFrozenClientProof generate a multihop proof of a frozen channel given a set of chains starting from the misbehaving chain.
// Chains are provided in order from the verifying chain to the misbehaving chain.
func GenerateFrozenClientProof(chains []*Chain, frozenChainIndex uint64, proofHeight exported.Height) *MultihopProof {

    abortTransactionUnless(len(chains) > 2)
    abortTransactionUnless(frozenChainIndex < len(chains))

    // generate and assign consensus, clientState, and connection proofs
    var proof MultihopProof
    proof.ConsensusProofs = GenerateConsensusProofs(chains)
    proof.ConnectionProofs = GenerateConnectionProofs(chains)

    chainA := chains[frozenChainIndex]   // misbehaving chain
    chainB := chains[frozenChainIndex-1] // next hop chain

    // check a consensus state at proof height is present on chain B
    consensusStateAtProofHeight := chainB.GetConsensusStateAtHeight(chainA, proofHeight)
    abortTransactionUnless(consensusStateAtProofHeight != nil)

    // connectionEnd on misbehaving chain representing the counterparty connection on the next chain
    counterpartyConnectionEnd := abortTransactionUnless(Unmarshal(proof.ConnectionProofs[N].Value))

    // key path to frozen client state
    key := host.FullClientStatePath(counterpartyConnectionEnd.ClientId)
    value := abortTransactionUnless(Marshal(chainB.GetClientState(counterpartyConnectionEnd.ClientId)))

    // query the key/value proof on the next hop chain at the proof height
    keyProof, _ := chainB.QueryProofAtHeight(key, proofHeight.GetRevisionHeight())

    // assign the key/value proof
    proof.KeyProof = &ProofData{
        Key:   nil,    // key to prove constructed during verification using the provided connection state proofs
        Value: value,  // provide the client state value
        Proof: keyProof,
    }

    return &proof
}


// GenerateConsensusProofs generates consensus state proofs starting from the sending chain (N) to the receiving chain (0).
// Compute proof for each chain starting from the sending chain, C[N], to the receiving chain, C[0].
// The proof is computed by querying the consensus state at the height of the client state on the next chain. In
// order for the proof logic to work, the client on the next chain must be updated to reflect the state of the prior chain.
// For example, for a channel between 4 chains, Chain[0] --> Chain[1] --> Chain[2] ... --> Chain[N], the proof is computed as follows:
// Step 1: Ensure Client[0] height on Chain[1] is greater than or equal to the proof height on C0. If not, update Client[0] on Chain[1].
// Step 2: Query proof of Chain[0] consensus state at proof height on Chain[1]
//         |Chain[N] ---> Chain[N-1] ---> Chain[N-2]| ---> Chain[N-3] ... ---> Chain[0]
//          (i)             (i+1)            (i+2)
// Step 3: Ensure Client[N] height on Chain[N-1] is greater than or equal to the proof height on C[N]. If not, update Client[N] on Chain[N-2].
// Step 4: Query proof of Chain[N] consensus state at proof height on Chain[N-1]
//         Chain[N] ---> |Chain[N-1] ---> Chain[N-2] ---> Chain[N-3]| ... ---> Chain[0]
//                           (i)            (i+1)            (i+2)
// ...
// Step M-2: Ensure Client[N-2] height on Chain[N-1] is greater than or equal to the proof height on Chain[N-2]. If not, update Client[N-2] on C[N-1].
// Step M-1:  Chain[N] ---> Chain[N-1] ...  ---> |Chain[2] --> Chain[1] ---> Chain[0]|
//                                                  (i)         (i+1)         (i+2)
// Step M: Ensure Client[1] height on Chain[0] is greater than or equal to the proof height on Chain[1]. If not, update Client[1] on Chain[0].
// Relayers may use slightly different algorithms, for example performing the client updates while collecting the proofs.
func GenerateConsensusProofs(chains []*Chain) []*ProofData {
    assert(len(chains) > 2)

    var proofs []*ProofData

    // iterate all but the last two chains
    for i := N; i > 1; i-- {

        previousChain := chains[i]   // previous chains state root is on currentChain and is the sending chain for i==0.
        currentChain  := chains[i-1] // currentChain is where the proof is queried and generated
        nextChain     := chains[i-2] // nextChain holds the state root of the currentChain

        // height of previous chain client state on current chain
        consensusHeight := currentChain.GetClientStateHeight(previousChain.ClientID)

        // this check assumes the relayer has updated the client on the next chain to the latest height and we verify that here.
        proofHeight := nextChain.GetClientStateHeight(currentChain.ClientID) // height of current chain state on next chain
        abortTransactionUnless(consensusHeight <= proofHeight) // ensure that nextChain's client state is update to date

        // consensus state of previous chain on current chain at consensusHeight which is the height of previous chain client state on current chain
        consensusState := currentChain.GetConsensusStateAtHeight(previousChain.ClientID, consensusHeight)

        // key for consensus state of previous chain
        consensusKey := host.FullConsensusStateKey(previousChain.ClientID, consensusHeight)

        // proof of previous chain's consensus state at consensusHeight on currentChain at proofHeight
        consensusProof, _ := currentChain.QueryProofAtHeight(consensusKey, proofHeight.GetRevisionHeight())

        // create a prefixed key
        prefixedKey := currentChain.GetMerklePath(consensusKey)

        // prepend the proofs so they are ordered from receiving end towards sending end
        proofs = append(
            &ProofData{
                Key:   &prefixedKey,
                Value: consensusState,
                Proof: consensusProof,
            },
            consStateProofs
        )
    }
     return proofs
  }

// GenerateConnectionProofs generates connection state proofs starting with C1 -->  starting from the source chain to the N-1'th chain.
// Compute proof for each chain starting from the sending chain, C[0], to the receiving chain, C[N].
// The proof is computed by querying the connection state at the height of the client state on the next chain. In
// order for the proof logic to work, the client on the next chain must be updated to the latest height.
// For example, for a channel between 4 chains, C[0] --> C[1] --> C[2] ... --> C[N], the proof is computed as follows:
// Step 1: Ensure Client[0] height on C[1] is greater than or equal to the proof height on C0. If not, update Client[0] on C[1].
// Step 2: Query proof of C0 connection state at proof height on C1
//         |C[0] ---> C[1] ---> C[2]| ---> C[3] ... ---> C[N]
//         (i)       (i+1)      (i+2)
// Step 3: Ensure Client[1] height on C[2] is greater than or equal to the proof height on C[1]. If not, update Client[1] on C[2].
// Step 4: Query proof of C[1] consensus state at proof height on C[2]
//         C[0] ---> |C[1] ---> C[2] ---> C[3]| ... ---> C[N]
//                    (i)        (i+1)     (i+2)
// ...
// Step M-2: Ensure Client[N-2] height on C[N-1] is greater than or equal to the proof height on C[N-2]. If not, update Client[N-2] on C[N-1].
// Step M-1:  C[0] ---> C[1] ...  ---> |C[N-2] --> C[N-1] ---> CN|
//                                    (i)         (i+1)    (i+2)
// Step M: Ensure Client[N-1] height on C[N] is greater than or equal to the proof height on C[N-1]. If not, update Client[N-1] on C[N].
// Relayers may use slightly different algorithms, for example performing the client updates while collecting the proofs.
// Note: This implementation assumes the test chains know the connection ids of the previous chain. They are constructed
// such that chain1 knows the ConnectionID to connect to chain0 and so on. The connection end counterparty connection ids
// correspond to the connectionHops field in the IBC channel.
func GenerateConnectionProofs(chains []*Chain) []*ProofData {
    assert(len(chains) > 2)

    var proofs []*ProofData

    // iterate all but the last two chains
    for i := N; i > 1; i-- {

        previousChain := chains[i]  // previous chains state root is on currentChain and is the source chain for i==0.
        currentChain := chains[i-1] // currentChain is where the proof is queried and generated
        nextChain := chains[i-2]    // nextChain holds the state root of the currentChain

        // height of previous chain state on current chain
        connectionHeight := currentChain.GetClientStateHeight(previousChain.ClientID)

        // this check assumes the relayer has updated the client on the next chain to the latest height and we verify that here.
        proofHeight := nextChain.GetClientStateHeight(currentChain.ClientID) // height of current chain state on next chain
        abortTransactionUnless(connectionHeight <= proofHeight) // ensure that nextChain's client state is update to date

        // the connection end on the current chain representing the previous chain.
        connectionEnd := currentChain.GetConnection()

        // the connection key
        connectionKey := host.ConnectionKey(currentChain.ConnectionID())

        // query proof of the currentChain's connection with the previousChain at nextHeight
        consensusProof, _ := currentChain.QueryProofAtHeight(connectionKey, proofHeight.GetRevisionHeight())

        // create the merkle path with the chain's prefix key
        prefixedKey := currentChain.GetMerklePath(connectionKey)

        // prepend the proofs so they are ordered from receiving end towards sending end
        proofs = append(
            &ProofData{
                Key:   &prefixedKey,
                Value: connectionEnd,
                Proof: connectionProof,
            },
            proofs
        )
    }
    return proofs
}
```

### Multi-hop Proof Verification Steps

The following outlines the general proof verification steps specific to a multi-hop IBC message.

1. Unpack the multihop proof bytes into consensus states, connection states and channel/commitment proof data.
2. Check the counterparty client on the receiving end is active and the client height is greater than or equal to the proof height.
3. Iterate through the connections states to determine the maximum `delayPeriod` for the channel path and verify that the counterparty consensus state on the receiving chain satisfies the delay requirement.
4. Iterate through connection state proofs and verify each connectionEnd is in the OPEN state and check that the connection ids match the channel connectionHops.
5. Verify the intermediate state proofs. Starting with known `ConsensusState[0]` at the given `proofHeight` on `Chain[1]` prove the prior chain's consensus and connection state.
6. Verify that the client id in each consensus state proof key matches the client id in the ConnectionEnd in the previous connection state proof.
7. Repeat step 5, proving `ConsensusState[i]`, and `Conn[i,i-1]` where `i` is the proof index starting with the consensus state on `Chain[2]`. `ConsensusState[1]` is already known on `Chain[0]`. Note that chains are indexed from executing (verifying) chain to  and proofs are indexed in the opposite direction to match the connectionHops ordering.
   - Verify ParseClientID(ConsensusProofs[i].Key) == ConnectionEnd.ClientID
   - ConsensusProofs[i].Proof.VerifyMembership(ConsensusState.GetRoot(), ConsensusProofs[i].Key, ConsensusProofs[i].Value)
   - ConnectionProofs[i].Proof.VerifyMembership(ConsensusState.GetRoot(), ConnectionProofs[i].Key, ConnectionProofs[i].Value)
   - ConsensusState = ConsensusProofs[i].Value
   - i++
8. Finally, prove the expected channel or packet commitment in `ConsensusState[N-2]` (sending chain consensus state) on `Chain[1]`

For more details see [ICS4](https://github.com/cosmos/ibc/tree/main/spec/core/ics-004-channel-and-packet-semantics).

### Multi-hop Proof Verification Pseudo Code

Pseudocode proof generation for a channel between `N` chains `C[N] --> C[i] --> C[0]`

```go
// Parse the connectionID from the connection proof key and return it.
func parseConnectionID(prefixedKey *PrefixedKey) string {
    abortTransactionUnless(prefixedKey.KeyPath[0] == "connections")
    abortTransactionUnless(len(prefixedKey.KeyPath) >= 2)
    parts := strings.Split(prefixedKey.GetKeyPath()[len(prefixedKey.KeyPath)-1], "/")
    abortTransactionUnless(len(parts) >= 2)
    return parts[len(parts)-1]
}

// VerifyMultihopMembership verifies a multihop membership proof.
// Inputs: consensusState - The consensusState for chain[N-1], which is known on the receiving chain (chain[N]).
//         connectionHops - The expected connectionHops for the channel from the receiving chain to the sending chain.
//         proof          - The serialized multihop proof data.
//         prefix         - Merkleprefix to be combined with key to generate Merklepath for the key/value proof verification.
//         key            - The key to prove in the indexed consensus state.
//         value          - The value to prove in the indexed consensus state.
func VerifyMultihopMembership(
    consensusState exported.ConsensusState,
    connectionHops []string,
    proof []byte,
    prefix exported.Prefix,
    key string,
    value []byte,
) {
    // deserialize proof bytes into multihop proofs
    proofs := abortTransactionUnless(Unmarshal(proof))
    abortTransactionUnless(len(proofs.ConsensusProofs) >= 1)
    abortTransactionUnless(len(proofs.ConnectionProofs) == len(proofs.ConsensusProofs))

    // verify connection hop ordering and connections are in OPEN state
    abortTransactionUnless(VerifyConnectionHops(proofs.ConnectionProofs, connectionHops))

    // verify intermediate consensus and connection states from receiver --> sender
    abortTransactionUnless(VerifyConsensusAndConnectionStates(consensusState, proofs.ConsensusProofs, proofs.ConnectionProofs))

    // verify a key/value proof on source chain's consensus state.
    abortTransactionUnless(VerifyKeyMembership(proofs, prefix, key, value))
}

// VerifyMultihopNonMembership verifies a multihop non-membership proof.
// Inputs: consensusState - The consensusState for chain[1], which is known on the receiving chain (chain[0]).
//         connectionHops - The expected connectionHops for the channel from the receiving chain to the sending chain.
//         proof          - The serialized multihop proof data.
//         prefix         - Merkleprefix to be combined with key to generate Merklepath for the key/value proof verification.
//         key            - The key to prove absent in the indexed consensus state
func VerifyMultihopNonMembership(
    consensusState exported.ConsensusState,
    connectionHops []string,
    proof []byte,
    prefix exported.Prefix,
    key string,
) {
    // deserialize proof bytes into multihop proofs
    proofs := abortTransactionUnless(Unmarshal(proof))
    abortTransactionUnless(len(proofs.ConsensusProofs) >= 1)
    abortTransactionUnless(len(proofs.ConnectionProofs) == len(proofs.ConsensusProofs))

    // verify connection hop ordering and connections are in OPEN state
    abortTransactionUnless(VerifyConnectionHops(proofs.ConnectionProofs, connectionHops))

    // verify intermediate consensus and connection states from receiver --> sender
    abortTransactionUnless(VerifyConsensusAndConnectionStates(consensusState, proofs.ConsensusProofs, proofs.ConnectionProofs))

    // verify a key/value proof on source chain's consensus state.
    abortTransactionUnless(VerifyKeyNonMembership(proofs, prefix, key))
}

// VerifyConnectionHops checks that each connection in the multihop proof is OPEN and matches the connections in connectionHops.
func VerifyConnectionHops(
    connectionProofs []*ProofData,
    connectionHops []string,
) {
    // check all connections are in OPEN state and that the connection IDs match and are in the right order
    for i, connData := range connectionProofs {
        connectionEnd := abortTransactionUnless(Unmarshal(connData.Value))

        // Verify the rest of the connectionHops (first hop already verified)
        // 1. check the connectionHop values match the proofs and are in the same order.
        connectionID := parseConnectionID(connData.Key)
        abortTransactionUnless(connectionID == connectionHops[i+1])

        // 2. check that the connectionEnd's are in the OPEN state.
        abortTransactionUnless(connectionEnd.GetState() == int32(connectiontypes.OPEN))
    }
}

// VerifyConsensusAndConnectionStates verifies the state of each intermediate consensus and connection state
// starting from the receiving chain and finally proving the sending chain consensus and connection state.
func VerifyConsensusAndConnectionStates(
    consensusState exported.ConsensusState,
    consensusProofs []*ProofData,
    connectionProofs []*ProofData,
) {
    // iterate through proofs to prove from executing chain (receiver) to counterparty chain (sender)
    var clientID string
    for i := 0; i < len(consensusProofs); i++ {

        // check the clientID from the previous connection is part of the next
        // consensus state's key path.
        abortTransactionUnless(VerifyClientID(clientID, consensusProofs[i].Key))

        // prove the consensus state of chain[i] on chain[i-1]
        consensusProof := abortTransactionUnless(Unmarshal(consensusProofs[i].Proof))
        abortTransactionUnless(consensusProof.VerifyMembership(
            commitmenttypes.GetSDKSpecs(),
            consensusState.GetRoot(),
            *consensusProof.Key,
            consensusProof.Value,
        ))

        // prove the connection state of chain[i] on chain[i-1]
        connectionProof := abortTransactionUnless(Unmarshal(connectionProofs[i].Proof))
        abortTransactionUnless(connectionProof.VerifyMembership(
            commitmenttypes.GetSDKSpecs(),
            consensusState.GetRoot(),
            *connectionProof.Key,
            connectionProof.Value,
        ))

        var connection ConnectionEnd
        abortTransactionUnless(connectionProof.Value, &connection)
        clientID = connection.ClientID

        // update the consensusState to chain[i] to prove the next consensus/connection states
        consensusState = abortTransactionUnless(UnmarshalInterface(consensusProof.Value))
    }
}

// VerifyKeyMembership verifies a key in the indexed chain consensus state.
func VerifyKeyMembership(
    proofs *MsgMultihopProof,
    prefix exported.Prefix,
    key string,
    value []byte,
) {
    // create prefixed key for proof verification
    prefixedKey := abortTransactionUnless(commitmenttypes.ApplyPrefix(prefix, commitmenttypes.NewMerklePath(key)))

    // extract indexed consensus state from consensus proofs
    consensusState := abortTransactionUnless(UnmarshalInterface(proofs.ConsensusProofs[proofs.KeyProofIndex].Value))

    // assign the key proof to verify on the source chain
    keyProof := abortTransactionUnless(Unmarshal(proofs.KeyProof.Proof))

    abortTransactionUnless(keyProof.VerifyMembership(
        commitmenttypes.GetSDKSpecs(),
        consensusState.GetRoot(),
        prefixedKey,
        value,
    ))

}

// VerifyKeyNonMembership verifies a key in the indexed chain consensus state.
func VerifyKeyNonMembership(
    proofs *MsgMultihopProof,
    prefix exported.Prefix,
    key string,
) {
    // create prefixed key for proof verification
    prefixedKey := abortTransactionUnless(commitmenttypes.ApplyPrefix(prefix, commitmenttypes.NewMerklePath(key)))

    // extract indexed consensus state from consensus proofs
    consensusState := abortTransactionUnless(UnmarshalInterface(proofs.ConsensusProofs[proofs.KeyProofIndex].Value))

    // assign the key proof to verify on the source chain
    keyProof := abortTransactionUnless(Unmarshal(proofs.KeyProof.Proof))

    abortTransactionUnless(keyProof.VerifyNonMembership(
        commitmenttypes.GetSDKSpecs(),
        consensusState.GetRoot(),
        prefixedKey,
    ))
}

// VerifyClientID verifies that the client id in the provided key matches an expected client id.
func VerifyClientID(expectedClientID string, key string) {
    clientID := strings.Split(key, "/")[1] // /clients/<clientid>/...
    abortTransactionUnless(expectedClientID == clientID)
}
```

### Path Forgery Protection

From the view of a single network, a list of connection IDs describes an unforgeable path of pre-existing connections to a receiving chain. This is ensured by atomically incrementing connection IDs.

We must verify a proof that the connection ID of each hop matches the proven connection state provided to the verifier. Additionally we must link the connection state to the consensus state for that hop as well. We are essentially proving out the connection path of the channel.

## Backwards Compatibility

The existing IBC message sending should not be affected. Minor changes to the verification logic would be required to identify a multi-hop packet and apply the proper proof verification logic. Multi-hop packets should be easily identified by the existence of more than one connection ID in the connectionHops field.

## Forwards Compatibility

If a decentralized chain name service is developed for the data currently in the chain registry on github, it may be possible for a calling app to specify only a unique chain name rather than an explicit set of connectionHops. The chain name service would maintain a mapping of chain names to channel paths. The name service could push routing table updates to subscribed chains. This should require fewer writes since routing table updates should be fewer than packets.

## Example Implementation

Coming soon.

## Other Implementations

Coming soon.

## History

Dec 16, 2022 - Revisions to channel spec and proof verification

Nov 11, 2022 - Initial draft

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).