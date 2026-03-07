# Data Indexing

[The Graph](https://thegraph.com/) serves as the indexing layer that transforms raw blockchain events into clean, queryable data for the SapphireDAO ecosystem. Instead of manually reading logs or tracking state transitions, The Graph continuously monitors Payment Processor contracts and stores structured invoice information off-chain. This enables swift access to invoice details, payment history, dispute status, release timestamps, and general activities.

After development, the subgraph is published to The Graph’s decentralized network. [Curators](https://thegraph.com/docs/en/resources/roles/curating/) can signal Graph Token(GRT) on the subgraph if they believe it provides valuable data, which helps allocate indexing resources to it and enhances network performance.

In SapphireDAO, The Graph functions as the primary indexing and query layer for historical and structured data, supporting features like filtering, sorting, and pagination. This architecture ensures fast, reliable data access without placing excessive load on the blockchain or necessitating custom indexing infrastructure.
