# TLS Patch for SpacetimeDB Standalone Server

This repo provides a patch for [SpacetimeDB](https://github.com/clockworklabs/SpacetimeDB) to enable TLS support for SpacetimeDB’s standalone server, supporting either a self-signed cert for that server or a local CA (certificate authority) trusted cert that signed that cert for that server, therefore any connecting client must have either the server’s public cert or the local CA’s public cert in its root store to trust it (or use `--cert` arg and point to it, instead).

- Target: [SpacetimeDB](https://github.com/clockworklabs/SpacetimeDB)
- License: Business Source License 1.1 (see `LICENSE.txt`), switches to AGPLv3 on March 27, 2030

## Usage:
- apply patch TODO:  
apply on commit 651f79d22cc4bb1c3c996ef2436186501a5d83bd (origin/master, origin/HEAD, master)
- install `spacetime` command TODO:
- generate server private and public keys (maybe sign by your own local CA):  
  you'll find instructions here: https://github.com/dare3path/spacetimedb-cert-gen
- start the spacetimedb standalone server in TLS mode:  
  `spacetime start --edition standalone --listen-addr 127.0.0.1:3000 --ssl --cert ../spacetimedb-cert-gen/server.crt --key ../spacetimedb-cert-gen/server.key`
  - can use `--ssl`, `--tls`, `--https` or `--secure`, they're aliases of the same thing.
- start a rust client from a different terminal and connect to the server in TLS mode:  
  `cd ./crates/sdk/` (this is in SpacetimeDB repo)  
  `cargo run --example quickstart-chat -- --cert ../../../spacetimedb-cert-gen/ca.crt`  
  Note that spacetimedb commit 651f79d22cc4bb1c3c996ef2436186501a5d83bd had the client hardcoded to connect to 127.0.0.1:3000, the patch kept this and only changed the scheme from http to https (if `--cert` args is supplied).  
  This will likely `404` if you haven't published the module first (see below).
- use cli commands:  
  `spacetime version list`  
  `spacetime version use 1.1.0`  
  `spacetime server list`  
  `spacetime server add --url https://127.0.0.1:3000 slocal --no-fingerprint`  
  `spacetime server set-default slocal` (can use `https://127.0.0.1:3000` instead of `slocal` here)  
  `spacetime server list`, should show this:  
```
WARNING: This command is UNSTABLE and subject to breaking changes.

 DEFAULT  HOSTNAME                   PROTOCOL  NICKNAME  
          maincloud.spacetimedb.com  https     maincloud 
          127.0.0.1:3000             http      local     
     ***  127.0.0.1:3000             https     slocal    
```
  assuming you're still inside `./crates/sdk/`:
```
$ spacetime login --server-issued-login slocal --cert ../../../my/spacetimedb-cert-gen/ca.crt
WARNING: the server that you specified here as 'slocal' isn't the one that will be used by commands like 'spacetime publish' but instead it's the one listed on 'spacetime server list' as the default (3 stars) that will be used, eg. 127.0.0.1:3000 if you haven't manually added any via 'spacetime server add'.

We will log in directly to your target server.
We have logged in directly to your target server.
WARNING: This login will NOT work for any other servers.
Saving config to /home/user/.config/spacetime/cli.toml.
```
  
```
$ spacetime publish -p ../../modules/quickstart-chat/ --cert ../../../my/spacetimedb-cert-gen/ca.crt quickstart-chat
    Finished `release` profile [optimized] target(s) in 0.22s
Optimising module with wasm-opt...
Build finished successfully.
Uploading to slocal => https://127.0.0.1:3000
Publishing module...
Created new database with name: quickstart-chat, identity: c200ba48494b35ab616cea275571f895d0ac6bf6e16b391238bf59ec65d1410d
```
```
$ cargo run --example quickstart-chat -- --cert ../../../my/spacetimedb-cert-gen/ca.crt    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.48s
     Running `/home/user/SOURCE/SpacetimeDB/target/debug/examples/quickstart-chat --cert ../../../my/spacetimedb-cert-gen/ca.crt`
Fully connected and all subscriptions applied.
Use /name to set your name, or type a message!
User c2001e6a388eaebd connected.
message here
c2001e6a388eaebd: message here
message2 here
c2001e6a388eaebd: message2 here
```
(Ctrl+D(+Z on Windows) to exit, or Ctrl+C)  
  
Can use some(TODO: make all relevant) cli commands that require server access by passing args `--cert ca.crt` or `--cert server.crt`, otherwise you'd have to have the public certificate of the CA(certificate authority that signed the server's public key) or of the target server in your cert root store (eg. /etc/ssl/ on linux).  

## Extra
You can still use HTTP/plaintext mode (both the server and the client(s) must be in the same mode - plaintext here):  
- start the spacetimedb standalone server in plaintext mode:  
  `spacetime start --edition standalone --listen-addr 127.0.0.1:3000`
- start a rust client from a different terminal and connect to the server in plaintext mode:  
  `cd ./crates/sdk/` (this is in SpacetimeDB repo)  
  `cargo run --example quickstart-chat`

If you're accidentally going to connect via TLS to the plaintext server, or connect via plaintext to the TLS server, you might encounter cryptic errors:
- a TLS client connecting to a plaintext server:
  - start the spacetimedb standalone server in plaintext mode:  
    `spacetime start --edition standalone --listen-addr 127.0.0.1:3000`
  - start a rust client from a different terminal and connect to the server in TLS mode:  
    `cd ./crates/sdk/` (this is in SpacetimeDB repo)  
    `cargo run --example quickstart-chat -- --cert ../../../spacetimedb-cert-gen/ca.crt`
  - The error seen on client is:  
`thread 'main' panicked at crates/sdk/examples/quickstart-chat/main.rs:79:10:`  
`Failed to connect: FailedToConnect { source: InternalError { message: "Failed to initiate WebSocket connection", cause: Some(Tungstenite { uri: wss://localhost:3000/v1/database/quickstart-chat/subscribe?connection_id=3a28480e485d68ec95bc00b0dc699cb8&compression=Brotli, source: Tls(Native(Ssl(Error { code: ErrorCode(1), cause: Some(Ssl(ErrorStack([Error { code: 167772358, library: "SSL routines", function: "tls_get_more_records", reason: "packet length too long", file: "ssl/record/methods/tls_common.c", line: 662 }, Error { code: 167772473, library: "SSL routines", function: "", reason: "record layer failure", file: "ssl/record/rec_layer_s3.c", line: 688 }]))) }, X509VerifyResult { code: 0, error: "ok" }))) }) } }`  
  - the meaning of the error is: your client is using a TLS connection to connect to a plaintext server, either both should be plaintext or both should be TLS. The server doesn't, currently, know how to handle both types on the same port.
  - if you want better error messages in this case(TLS client connecting to non-TLS server) then you could use a different openssl system library(or `LD_PRELOAD` it, etc.) like the one patched here: https://github.com/dare3path/TLSplain in which case you would then see this more descriptive error:  
`thread 'main' panicked at crates/sdk/examples/quickstart-chat/main.rs:79:10:`  
`Failed to connect: FailedToConnect { source: InternalError { message: "Failed to initiate WebSocket connection", cause: Some(Tungstenite { uri: wss://localhost:3000/v1/database/quickstart-chat/subscribe?connection_id=8b6f1537eaaf45a1dacf89b28f7a9183&compression=Brotli, source: Tls(Native(Ssl(Error { code: ErrorCode(1), cause: Some(Ssl(ErrorStack([Error { code: 167774112, library: "SSL routines", function: "tls_get_more_records", reason: "you attempted a TLS connection to a plaintext HTTP server ie. to a non-TLS server", file: "ssl/record/methods/tls_common.c", line: 747 }, Error { code: 167772473, library: "SSL routines", function: "", reason: "record layer failure", file: "ssl/record/rec_layer_s3.c", line: 689 }]))) }, X509VerifyResult { code: 0, error: "ok" }))) }) } }`  
  ... and now you won't have to wonder if there's something else you did wrong in your client.
- plaintext client connecting to a TLS server:
  - start the spacetimedb standalone server in TLS mode:  
    `spacetime start --edition standalone --listen-addr 127.0.0.1:3000 --ssl --cert ../spacetimedb-cert-gen/server.crt --key ../spacetimedb-cert-gen/server.key`
  - start a rust client from a different terminal and connect to the server in plaintext mode:  
    `cd ./crates/sdk/` (this is in SpacetimeDB repo)  
    `cargo run --example quickstart-chat`
  - The error seen on client is:  
`thread 'main' panicked at crates/sdk/examples/quickstart-chat/main.rs:79:10:`  
`Failed to connect: FailedToConnect { source: InternalError { message: "Failed to initiate WebSocket connection", cause: Some(Tungstenite { uri: ws://localhost:3000/v1/database/quickstart-chat/subscribe?connection_id=e54407d34c5347e362ea91ef6f03fbc3&compression=Brotli, source: Protocol(HttparseError(Version)) }) } }`  
  - the meaning of the error is: your client is using plaintext connection to a TLS server, either both should be plaintext or both should be TLS. The server doesn't, currently, know how to handle both types on the same port.
  
## Credits
Made with xAI's Grok3 and Grok2 because I don't know much of how to Rust or SpacetimeDB. So if things seem wrong, or suboptimal, they might be.

## License
- Business Source License 1.1 (see `LICENSE.txt`), switches to AGPLv3 on March 27, 2030

