# Fetch Client

` // To Do : StateFetcher, Client(HeadersClient, BodiesClient), Downloader (Headers, Bodies), poll Action `

_Using FetchClient to Get Data in the Pipeline Stages_

<img width="608" alt="fetch client" src="https://github.com/user-attachments/assets/570099cd-7f6a-4d23-864d-d3d19d15653c">


[File: crates/net/network/src/fetch/client.rs](https://github.com/paradigmxyz/reth/blob/main/docs/crates/network.md#using-fetchclient-to-get-data-in-the-pipeline-stages)

```Rust
pub struct FetchClient {
    pub(crate) request_tx: UnboundedSender<DownloadRequest>,
    pub(crate) peers_handle: PeersHandle,
}
```