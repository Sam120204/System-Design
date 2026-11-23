# QUIC

QUIC is an encrypted transport protocol carried in UDP datagrams. QUIC manages connections, reliable streams, loss recovery, and congestion control. It integrates TLS 1.3 into the handshake, and HTTP/3 uses QUIC to carry HTTP.

![Transport protocols listed in the original reference: TCP, UDP, and QUIC](images/quic-in-transport-outline.png)

## How QUIC changes multiplexing

HTTP/2 can interleave several streams on a single TCP byte stream. TCP delivers data in byte order, so a missing segment can delay all subsequent bytes on that connection. QUIC assigns ordering and offsets per stream. If data for stream A is missing, data already received for stream B can generally be delivered if stream B has no gap. A single lost packet can still contain data for several streams, and all streams on the connection share network capacity and congestion control.

| Capability | System design implication |
| --- | --- |
| Integrated handshake | A new connection can typically send application data after one round trip. Certain resumed sessions can send early data in zero round trips, subject to replay risk and server policy |
| Independent streams | Loss on one stream need not block delivery of unrelated stream data that already arrived |
| Connection IDs | An established connection can often survive a client's address or port change, subject to peer support and path validation |
| Encryption | QUIC protects transport and application data, which affects what network intermediaries can observe and how they route traffic |

Early data can be replayed, so application handling must account for that risk. Connection migration also requires infrastructure that can route packets for the existing connection to its owner when the client's network tuple changes. [QUIC transport](https://www.rfc-editor.org/rfc/rfc9000.html) · [QUIC and TLS](https://www.rfc-editor.org/rfc/rfc9001.html)

## Design example

A CDN serving mobile browsers can offer HTTP/3 at the edge and keep HTTP/2 available for clients or networks that cannot establish QUIC. HTTP/3 can help parallel fetches over a lossy path and preserve a connection when a phone moves between Wi-Fi and cellular. The CDN can still select a separately supported protocol for the origin hop. Measure connection success, tail latency, and fallback rates across the client networks you serve.
