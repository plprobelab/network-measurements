
# RFM 17.1 | Sharing Provider Records with Multiaddresses

_DRI/Team_: [`Mikel Cortes-Goicoechea`](https://github.com/cortze) & [`Leonardo Bautista-Gomez`](https://github.com/leobago) ([`Barcelona Supercomputing Center`](https://bsc.es/))

_Date_: 11/11/2022

## Table of contents

1. [Introduction](#1-Introduction)
2. [Findings](#2-Findings)
3. [Methodology](#3-Methodology)
4. [Discussion](#4-Discussion)
5. [Contributions](#5-Contributions)
6. [Conclusion](#6-Conclusion)
7. [References](#7-references)

## 1-Introduction
**_Context_**: [Slack Discussion](https://filecoinproject.slack.com/archives/C03FFEVK30F/p1662995123912119) & [GitHub Issue](https://github.com/ipfs/kubo/issues/9264) 

Content routing through DHT lookups in IPFS involves two different steps:

1. Routing the desired CID to the nodes that host the content - find the Provider's Records (PR) in the public Provider Records DHT
2. Mapping the peer ID of the nodes hosting the content to their public multiaddress  - find the multiaddress of a peer in the Public Address DHT

In this context, we understand the PR as the link between a CID and a Peer ID of the content provider, which is how the PRs are stored in the public DHT. However, in the networking layer, when a host requests or looks for the Providers of a given CID, the method [`dht.FindProviders()`](https://github.com/libp2p/go-libp2p-kad-dht/blob/dae5a9a5bd9c7cc8cfb5073c711bc308efad0ada/routing.go#L445) returns two arrays of `peer.AddrInfo`:

- The first array relates to the `[]AddrInfo` of content providers that the remote host is aware of.
- The second array relates to peers closer to the CID from the remote peer's perspective (from its routing table).

Given that the structure of the `peer.AddrInfo` is:

```go
type AddrInfo struct {
	ID    ID
	Addrs []Multiaddr
}
```

One could think: Why do we split the retrieval process into two different steps or lookups when we could simply return the `ID` and `[]Multiaddr` of the peer in the same response?

- The simple answer is: it already happens (sometimes).

Each node in the network keeps locally a peerstore with the information of each currently and previously connected peer. In this local peerstore, nodes generally keep items like the `UserAgent`, `Latency`, `Multiaddresses`, and so on from the remote peer. But most importantly, hosts keep locally an expiration time or Time To Live (TTL) for those `Multiaddresses` associated with that peer. This TTL gets updated or extended each time the local hosts interact with that remote peer, which generally happens when the local host refreshes its routing table or when a Provider publishes or republishes the PR.   

In the current IPFS network, the TTL of those `Multiaddresses` is between 10 and 30 mins (depending on the `go-libp2p` [version](https://github.com/libp2p/go-libp2p/commit/c2821791bac7d638890accc98798bf4cbfe122e7)), while the TTL for the PR is ~24 hours (still pending to be re-adjusted to 48 hours after the submission of [RFM17](https://github.com/protocol/network-measurements/blob/master/results/rfm17-provider-record-liveness.md)).

So, when some node requests the PR of a given CID, the node that stores the PR will only reply with the `[]Multiaddr` together with the `PeerID` of the content provider if the TTL for the `Multiaddr` hasn't expired yet. This prevents sharing non-updated records for any peer in the network. Effectively, it avoids sending the client to the wrong multiaddress.

However, since the PRs have a TTL of 24 hours, and we intend to extend it up to 48h, is it worth shortening the current process that performs two lookups to a single one? In other words: 
- Should we extend the TTL of the `ID` - `Multiaddr` mapping? (30-10 mins VS 24 hours matching the PR TTL)
- Should we always share the `Multiaddr` with the `ID` if any peer asks for a PR that we know?

This extension of RFM17 aims to prove that the mapping between `ProviderID` and `[]Multiaddr` is actually shared for those 10 to 30 mins after connecting with a PR Holder to store the records. Ultimately, and perhaps as an item of future work, proving that extending that TTL to match the PR expiration time would remove the need to make a second DHT lookup to map the provider's `ID` and its `Multiaddress`. This will bring a significant performance increase to the DHT Lookup process, as it will reduce the latency by half, i.e., clients will need to "walk" the DHT once instead of twice, as is the case today.

## 2-Findings
- As we were expecting, peers no longer share the provider’s `ID` + `Multiaddress` after 30 mins (some of them still share them after the expected 10 mins depending on the IPFS client that they are using. The TTL increase was included in `go-libp2p@v0.22.0` August 18th, 2022)
- Even though the lookup process can find the `Multaddress` for the provider over those 10-30 mins, the lookup process sometimes returns an empty `Multaddress` field in the `peer.AddrInfo`. The problem? The `dht.FindProviders()` method only reports once each content provider, so if the first peer that reports the PR only includes the content provider's `PeerID`, later coming `PeerID` + `Multiaddress` mapping will no longer be notified. 


## 3-Methodology

The study uses as a basis the CID-Hoarder from RFM17. It includes some complementary modifications that trace down the retrieval process of the DHT lookup.

The tool uses two hosts, one for publishing the content (gets closed after the publication process), and the second for pinging the PR Holders individually, asking for the PRs, and performing a public DHT lookup for the content.

The result of each ping for the PR holders and the result of the lookup for the PRs are written in the `stdout` as logs. The logs contain both the peer reporting the PR and the content of the received `AddrInfo` response. These logs are then parsed in Python to produce the plots analyzed in section [4](#4-Discussion).

**_Notes_**: *Holder: Peers elected to store the PR in the publication process for the CIDs.*

## 4-Discussion
As previously introduced, this RFM aims to measure how PRs are shared when someone tries to find the content provider for a given CID. To do so, we will divide this section into four different chapters, i) the outcome of a request for a PR to the PR holders (successful, failed), ii) the reply from those peers that share the PR during the lookup process, iii) the final result of the `dht.FindProviders()` method form the `go-libp2p-kad-dht` implementation (same as the `kubo` one), and iv) an overview of the measured IP churn of the DHT servers in IPFS.

The experiment was done for a total of 100 CIDs for over 50 minutes on January 17th, 2023.

### 4.1-PR holder's direct reply

Looking closely at the ratio of PR holders that reply with the PR from Figure [1], we see that 6 to 10 PR holders are continuously sharing them over the entire study. Please keep in mind that there is a delay of 3 mins between ping rounds and that the network is currently experiencing some difficulties because half of the network is apparently unreachable. 

![img](./../implementations/rmf17.1-sharing-prs-with-multiaddresses/plots/pr_from_prholders_pings.png)

_Figure 1: Number of PR holders replying with the PR._

However, suppose we check the actual content of the `AddrInfo` that we receive back from the remote peers, as displayed in Figure [2]. In that case, we can observe that those 6 to 10 stable PR holders only share both `PeerID` + `Multiaddress` for around three ping rounds or 9 to 12 minutes. Afterward, the median drops to 4-5 stable peers sharing the combo until ping round 10, i.e., 30 mins, which is then followed by a period of only `PeerID` replies. 

![img](./../implementations/rmf17.1-sharing-prs-with-multiaddresses/plots/id_plus_multiaddres_per_prholders.png)

_Figure 2: Number of PR Holders replying with the `PeerID` + `Multiaddress` combo._

### 4.2-Reply of peers reporting the PR during the DHT lookup

Results are similar when we analyze the replies of the peers that report back the PR from the DHT lookup process. We set to 20 (same as K) the number of content providers we were looking for to track the multiple remote peers, adding a timeout of two minutes for the [`LookupForProviders()`](https://github.com/cortze/go-libp2p-kad-dht/blob/6b490320a6c1b70eba2031260a2515c26e7519fe/routing.go#L473) operation (same as [`FindProviders()`](https://github.com/cortze/go-libp2p-kad-dht/blob/6b490320a6c1b70eba2031260a2515c26e7519fe/routing.go#L456) but without checking the PRs locally at the `ProviderStore`). Figure [3] represents the number of remote peers reporting the PR for the CIDs we were looking for, where we can see seven stable peers by median over the entire study. 

![img](./../implementations/rmf17.1-sharing-prs-with-multiaddresses/plots/pr_from_lookup_process.png)

_Figure 3: Number of remote peers replying with the PR during the DHT lookup._

We spotted no difference with the individual when comparing the number of content provider's `Multiaddress` shared among PRs. Figure [4] also shows the same drop of peers sharing the combo after ~9-12 mins, with a sudden decline after round 10 (~30 mins). It is clear from these results that the network is quite segmented in terms of client versions, where the TTL of the `Multiaddress` records varies from 10 to 30 mins. 

![img](./../implementations/rmf17.1-sharing-prs-with-multiaddresses/plots/id_plus_multiaddres_from_lookup.png)

_Figure 4: Number or remote peers replying with the `PeerID` + `Multiaddress` combo during the DHT lookup process._


### 4.3-Result from the DHT lookup

It is essential to notice that despite being able to show empirically what we already knew, the combo of `PeerID` + `Multiaddress` gets shared only over the TTL of the `Multiaddress` records, the result that we got from the DHT lookup process don't fully match the results previously showed. 

Figure [5] displays the final result of the `LookupForProviders()` method, distinguishing the content of the received PR. In the figure, we can appreciate that for rounds one, two, and three (first 10 minutes) the Provider Records always come along the `Multiaddress` of the provider. 

![img](./../implementations/rmf17.1-sharing-prs-with-multiaddresses/plots/lookup_result.png)

_Figure 5: Result of the `dht.FindProviders` method, together with the filtered content of the received provider's `AddrInfo`._

We expect to retrieve the combo over the current TTL of the multiaddress records. However, the fragmentation between IPFS nodes in the network, with different TTLs for the multiaddress, makes some inconsistent replies when asking for the PRs. In the current [`libp2p/go-libp2p-kad-dht`](https://github.com/libp2p/go-libp2p-kad-dht) implementation, the return value of the lookup gets defined by the first `AddrInfo` response we get for each provider in the network (code [here](https://github.com/libp2p/go-libp2p-kad-dht/blob/e33a4be6e9a3a8fb603d21126e2d8a42c5e37d1b/routing.go#L490)). This means that if after 15 minutes of publishing a content some client wants to retrieve it through a DHT lookup if the first peer that we connect from the closest ones replies with only the `PeerID`, that requester client will have to perform a second DHT lookup to map the `PeerID` with the providers `Multiaddress`. This phenomenon will still happen even though a second reply might arrive with the entire combo 20ms later. 

### 4.4 IP Churn of PR Holders

The possible improvement we want to measure with the RFM by increasing the TTL of the provider's Multiaddress would only be beneficial for the network if peers keep their IP at least for the same time that the PRs are alive in the network. To measure the IP churn among nodes in the IPFS network, Figure [6] refers to an experiment done in [RFM-17](https://github.com/protocol/network-measurements/blob/master/results/rfm17-provider-record-liveness.md) where the hoarder was pinging PR Holders of 10k CIDs for over 80 hours.

![img](./../implementations/rmf17.1-sharing-prs-with-multiaddresses/plots/active_pr_holders_80h.png)

_Figure 6: Distribution of online PR Holders over 80 hours._

In the figure, we can appreciate how the online-ness of the PR Holders barely variates, keeping a median of 13 to 15 online PR Holders online. Considering that the hoarder only attempts to connect the PR Holders to the initial `Multiaddress` that we tracked when storing the PRs, we can conclude that the IP churn is not that perceptible among DHT servers.   


## 5-Contributions

To improve the impact that [increasing the provider multiaddres's TTL (@dennis-tra)](https://github.com/libp2p/go-libp2p-kad-dht/pull/795) would have in the retrieval of content, we have suggested to [adjust the PeerSet logic in the DHT lookup process (@cortze)](https://github.com/libp2p/go-libp2p-kad-dht/pull/802) to notify those `Multiaddress` of providers that previously were only reported with the `PeerID` of the provider. Figure [7] shows the result of the DHT lookup method when applying the proposed changes, where we observe that the lookup process always reports the combo `PeerID` + `Multiaddress` as soon as someone has a valid TTL for the `Multiaddress`. 

![img](./../implementations/rmf17.1-sharing-prs-with-multiaddresses/plots/retrievability_applying_code_mods.png)

_Figure 7: Result of the `dht.FindProviders` method applying code suggestions._

To minimize the impact of increasing the TTL of the provider's address, `kubo` maintainers suggested adding a backup DHT lookup for a possible updated provider's multiaddress [libp2p/go-libp2p#1835 (@dennis-tra)](https://github.com/libp2p/go-libp2p/pull/1835) in the `libp2p.RoutedHost` in case the multiaddress shared with the PRs ended into a connection failure.


## 6-Conclusion

With this study, we have demonstrated empirically that the combo of `PeerID` + `Multiaddress` are, in fact, shared for as long as the TTL of the `Multiaddress` records don't expire. This means that if we increase this TTL to match the PR expiration time, we could reduce to a single DHT lookup the process of retrieving a CID's content using the public DHT. This would significantly improve the overall DHT lookup time, as it would decrease latency by half.

On the other hand, we have identified some code limitations that could affect the impact of such improvements as the network keeps fragmented based on client versions and mismatches between configurations. Thus, the final intention of removing that extra second lookup might only achieve the expected result if the majority of the network accepts and upgrades the TTL of provider's multiaddresses. For that reason, we suggest merging both `increasing the provider Multiaddress TTL` and the `new PeerSet logic` together to maximize the results.  


## 7-References

* [RFM-17](https://github.com/protocol/network-measurements/blob/master/results/rfm17-provider-record-liveness.md)
* [CID-Hoarder](https://github.com/cortze/ipfs-cid-hoarder)