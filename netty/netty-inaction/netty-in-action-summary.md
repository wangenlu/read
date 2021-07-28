# Summary

## Introducing Netty

- hide the complexity of underlying implementations behind simpler abstractions

## Netty feature summary

| Category         | Netty features                                                                                                                                                                                          |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Design           | Unified API for multiple transport types, both blocking and non-blocking.Simple but powerful threading model.True connectionless datagram socket support.Chaining of logic components to support reuse. |
| Ease of use      | Extensive Javadoc and large example set.No required dependencies beyond JDK 1.6+. (Some optional features may requireJava 1.7+ and/or additional dependencies.)                                         |
| Performance      | Better throughput and lower latency than core Java APIs.Reduced resource consumption thanks to pooling and reuse.Minimal memory copying.                                                                |
| Robustness       | No OutOfMemoryError due to slow, fast, or overloaded connection.Eliminates unfair read/write ratio typical of NIO applications in high-speed networks.                                                  |
| Security         | Complete SSL/TLS and StartTLS support.Usable in restricted environments such as Applet or OSGI.                                                                                                         |
| Community-driven | Release early and often.                                                                                                                                                                                |
