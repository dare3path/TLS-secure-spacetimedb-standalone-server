# TLS Patch for SpacetimeDB Standalone Server

This repo provides a patch for [SpacetimeDB](https://github.com/clockworklabs/SpacetimeDB) to enable TLS support for SpacetimeDB’s standalone server, supporting self-signed or local CA (certificate authority) trusted certs for that server. Any connecting client must have either the server’s public cert or the local CA’s public cert in its root store to trust it. There’s a command-line option to point to such a cert for both the SpacetimeDB CLI and standalone server.

- Target: [SpacetimeDB](https://github.com/clockworklabs/SpacetimeDB)
- License: Business Source License 1.1 (see `LICENSE.txt`), switches to AGPLv3 on March 27, 2030

## License
- Business Source License 1.1 (see `LICENSE.txt`), switches to AGPLv3 on March 27, 2030

