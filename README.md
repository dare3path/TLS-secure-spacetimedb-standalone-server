# TLS Patch for SpacetimeDB Standalone Server

This repo provides a patch for [SpacetimeDB](https://github.com/clockworklabs/SpacetimeDB) to enable TLS support for SpacetimeDB’s standalone server, supporting either a self-signed cert for that server or a local CA (certificate authority) trusted cert that signed that cert for that server, therefore any connecting client must have either the server’s public cert or the local CA’s public cert in its root store to trust it (or use `--cert` arg and point to it, instead).

- Target: [SpacetimeDB](https://github.com/clockworklabs/SpacetimeDB)
- License: Business Source License 1.1 (see `LICENSE.txt`), switches to AGPLv3 on March 27, 2030

## Requirements:
- the `Cargo.toml` from the repo's root dir has a `[patch.crates-io]` section for a couple of repos which means these need to be cloned and patched locally and modify the `Cargo.toml` paths in there to point to where they are locally.  

## Usage:
- apply patch:  
assuming you're in bash shell:  
```
cd /tmp
git clone https://github.com/dare3path/spacetimedb-tls-patch
git clone https://github.com/clockworklabs/SpacetimeDB
cd SpacetimeDB
```
  look inside the patch file to see which commit to apply it on.  
```
git checkout 
patch -p1 -i ../spacetimedb-tls-patch/spacetimedb-tls.patch
```
Edit `./Cargo.toml` and remove anything from `[patch.crates-io]` section it it exists, unless you want to apply the patches for those and change the paths to match where you've cloned the repos and patched them.  

- install `spacetime` command:  
  if debug:  
```
time cargo build --locked -p spacetimedb-standalone -p spacetimedb-update -p spacetimedb-cli
outdir="./target/debug/"
```
  if release:  
```
time cargo build --release --locked -p spacetimedb-standalone -p spacetimedb-update -p spacetimedb-cli
outdir="./target/release/"
```
  for either:  
```
mkdir -p ~/.local/bin
export STDB_VERSION="$(${outdir}/spacetimedb-cli --version | sed -n 's/.*spacetimedb tool version \([0-9.]*\);.*/\1/p')"
mkdir -p ~/.local/share/spacetime/bin/$STDB_VERSION
cp ${outdir}/spacetimedb-cli ~/.local/share/spacetime/bin/$STDB_VERSION
cp ${outdir}/spacetimedb-standalone ~/.local/share/spacetime/bin/$STDB_VERSION
cp ${outdir}/spacetimedb-update ~/.local/bin/spacetime
```
  make sure you update your path:  
```
export PATH="$PATH:$HOME/.local/bin"
```
  and save it in `~/.bashrc` too for the future.  
```
spacetime version list
```
  might show something like this:  
```
1.0.1
1.1.0
spacetimedb-standalone
1.1.1
```
  select latest:  
```
spacetime version use "$STDB_VERSION"
ls -la ~/.local/share/spacetime/bin/current
```
  that should show something like:  
  `lrwxrwxrwx 1 user users 43 Apr 24 19:15 /home/user/.local/share/spacetime/bin/current -> /home/user/.local/share/spacetime/bin/1.1.1`  
  which confirms it's been selected properly.  
- generate server private and public keys (maybe sign by your own local CA):  
  you'll find instructions here: https://github.com/dare3path/spacetimedb-cert-gen
- start the spacetimedb standalone server in TLS mode:  
  `spacetime start --edition standalone --listen-addr 127.0.0.1:3000 --ssl --cert ../spacetimedb-cert-gen/server.crt --key ../spacetimedb-cert-gen/server.key`
  - can use `--ssl`, `--tls`, `--https` or `--secure`, they're aliases of the same thing.
  - note: mTLS is supported via args `--client-cert` which points to one or more(bundled ie. appended via `cat`) certs to add to the trusted root store for authenticating any certs that clients send to the server to be authenticated; there's `--client-no-trust-system-root-store`(which is the default) to not use the system's root store for authenticating client certs(in addition to what you passed), or `--client-trust-system-root-store` to trust it.  
- start a rust client from a different terminal and connect to the server in TLS mode:  
  `cd ./crates/sdk/` (this is in SpacetimeDB repo)  
  `cargo run --example quickstart-chat -- --cert ../../../spacetimedb-cert-gen/ca.crt`  
  Note that spacetimedb (at the time of this writing) had the client hardcoded to connect to 127.0.0.1:3000, the patch kept this as is and only changed the scheme from http to https (if `--cert` arg is supplied).  
  This will likely `404` if you haven't published the module first (see below).  
  - mTLS for clients/cli supports args like `--client-cert`, `--client-key`  
  - `--trust-system-root-store`(default) or `--no-trust-system-root-store` can be used when clients authenticate the server's cert in addition to `--cert`(aka `--trust-server-cert`).  
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
- other cli commands you can run, in another terminal:  
```
$ spacetime logs quickstart-chat --cert ../../../my/spacetimedb-cert-gen/ca.crt --follow
2025-04-13T05:10:45.901825Z  INFO: spacetimedb: Creating table `message`
2025-04-13T05:10:45.902506Z  INFO: spacetimedb: Creating table `user`
2025-04-13T05:10:45.904265Z  INFO: spacetimedb: Invoking `init` reducer
2025-04-13T05:10:45.906885Z  INFO: spacetimedb: Database initialized
```
```
$ spacetime server ping slocal --cert ../../../my/spacetimedb-cert-gen/ca.crt
WARNING: This command is UNSTABLE and subject to breaking changes.

Server is online: https://127.0.0.1:3000
```
```
$ spacetime server fingerprint slocal --cert ../../../my/spacetimedb-cert-gen/ca.crt
WARNING: This command is UNSTABLE and subject to breaking changes.

Fingerprint is unchanged for server slocal:
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEfwAqvSbxMUZKdCsvzr9tdQJOhuG/
24jZ+oMN18FS6Py9CDNzGeAzu2kqjlybtpYRqkchYqv8o44khX9TZlhyWA==
-----END PUBLIC KEY-----
```

<details> <summary> `spacetime describe` (click me to expand) </summary>

```
$ spacetime describe quickstart-chat --json --cert ../../../my/spacetimedb-cert-gen/ca.crtWARNING: This command is UNSTABLE and subject to breaking changes.

Added trusted certificate from ../../../my/spacetimedb-cert-gen/ca.crt for a new TLS connection.
Added trusted certificate from ../../../my/spacetimedb-cert-gen/ca.crt for a new TLS connection.
{
  "typespace": {
    "types": [
      {
        "Product": {
          "elements": [
            {
              "name": {
                "some": "sender"
              },
              "algebraic_type": {
                "Product": {
                  "elements": [
                    {
                      "name": {
                        "some": "__identity__"
                      },
                      "algebraic_type": {
                        "U256": []
                      }
                    }
                  ]
                }
              }
            },
            {
              "name": {
                "some": "sent"
              },
              "algebraic_type": {
                "Product": {
                  "elements": [
                    {
                      "name": {
                        "some": "__timestamp_micros_since_unix_epoch__"
                      },
                      "algebraic_type": {
                        "I64": []
                      }
                    }
                  ]
                }
              }
            },
            {
              "name": {
                "some": "text"
              },
              "algebraic_type": {
                "String": []
              }
            }
          ]
        }
      },
      {
        "Product": {
          "elements": [
            {
              "name": {
                "some": "identity"
              },
              "algebraic_type": {
                "Product": {
                  "elements": [
                    {
                      "name": {
                        "some": "__identity__"
                      },
                      "algebraic_type": {
                        "U256": []
                      }
                    }
                  ]
                }
              }
            },
            {
              "name": {
                "some": "name"
              },
              "algebraic_type": {
                "Sum": {
                  "variants": [
                    {
                      "name": {
                        "some": "some"
                      },
                      "algebraic_type": {
                        "String": []
                      }
                    },
                    {
                      "name": {
                        "some": "none"
                      },
                      "algebraic_type": {
                        "Product": {
                          "elements": []
                        }
                      }
                    }
                  ]
                }
              }
            },
            {
              "name": {
                "some": "online"
              },
              "algebraic_type": {
                "Bool": []
              }
            }
          ]
        }
      }
    ]
  },
  "tables": [
    {
      "name": "message",
      "product_type_ref": 0,
      "primary_key": [],
      "indexes": [],
      "constraints": [],
      "sequences": [],
      "schedule": {
        "none": []
      },
      "table_type": {
        "User": []
      },
      "table_access": {
        "Public": []
      }
    },
    {
      "name": "user",
      "product_type_ref": 1,
      "primary_key": [
        0
      ],
      "indexes": [
        {
          "name": {
            "some": "user_identity_idx_btree"
          },
          "accessor_name": {
            "some": "identity"
          },
          "algorithm": {
            "BTree": [
              0
            ]
          }
        }
      ],
      "constraints": [
        {
          "name": {
            "some": "user_identity_key"
          },
          "data": {
            "Unique": {
              "columns": [
                0
              ]
            }
          }
        }
      ],
      "sequences": [],
      "schedule": {
        "none": []
      },
      "table_type": {
        "User": []
      },
      "table_access": {
        "Public": []
      }
    }
  ],
  "reducers": [
    {
      "name": "identity_connected",
      "params": {
        "elements": []
      },
      "lifecycle": {
        "some": {
          "OnConnect": []
        }
      }
    },
    {
      "name": "identity_disconnected",
      "params": {
        "elements": []
      },
      "lifecycle": {
        "some": {
          "OnDisconnect": []
        }
      }
    },
    {
      "name": "init",
      "params": {
        "elements": []
      },
      "lifecycle": {
        "some": {
          "Init": []
        }
      }
    },
    {
      "name": "send_message",
      "params": {
        "elements": [
          {
            "name": {
              "some": "text"
            },
            "algebraic_type": {
              "String": []
            }
          }
        ]
      },
      "lifecycle": {
        "none": []
      }
    },
    {
      "name": "set_name",
      "params": {
        "elements": [
          {
            "name": {
              "some": "name"
            },
            "algebraic_type": {
              "String": []
            }
          }
        ]
      },
      "lifecycle": {
        "none": []
      }
    }
  ],
  "types": [
    {
      "name": {
        "scope": [],
        "name": "Message"
      },
      "ty": 0,
      "custom_ordering": true
    },
    {
      "name": {
        "scope": [],
        "name": "User"
      },
      "ty": 1,
      "custom_ordering": true
    }
  ],
  "misc_exports": [],
  "row_level_security": []
}
```

</details>

```
$ spacetime call quickstart-chat send_message "\"Hello from CLI\"" --cert ../../../my/spacetimedb-cert-gen/ca.crt
WARNING: This command is UNSTABLE and subject to breaking changes.

Added trusted certificate from ../../../my/spacetimedb-cert-gen/ca.crt for a new TLS connection.
Added trusted certificate from ../../../my/spacetimedb-cert-gen/ca.crt for a new TLS connection.
```
```
$ spacetime sql quickstart-chat "SELECT * FROM message" --cert ../../../my/spacetimedb-cert-gen/ca.crt
WARNING: This command is UNSTABLE and subject to breaking changes.

Added trusted certificate from ../../../my/spacetimedb-cert-gen/ca.crt for a new TLS connection.
Added trusted certificate from ../../../my/spacetimedb-cert-gen/ca.crt for a new TLS connection.
 sender                                                             | sent                             | text             
--------------------------------------------------------------------+----------------------------------+------------------
 0xc2001e6a388eaebd8a6b1a9b587d2ea6e288e8e2332f9848c8f23c42bdf1ee23 | 2025-04-13T05:11:00.078287+00:00 | "message here"   
 0xc200ade7f794c9fcba8d5ed1bf5e81d187c1e386c27886e64b396e0dceb6c816 | 2025-04-13T17:26:15.511753+00:00 | "Hello from CLI" 
```
this adds an alias for the database name(just like renaming projects on github):  
```
$ spacetime rename --to foo quickstart-chat --cert ../../../my/spacetimedb-cert-gen/ca.crt
Added trusted certificate from ../../../my/spacetimedb-cert-gen/ca.crt for a new TLS connection.
Domain set to foo for identity c200ba48494b35ab616cea275571f895d0ac6bf6e16b391238bf59ec65d1410d.

$ spacetime rename --to foo2 foo --cert ../../../my/spacetimedb-cert-gen/ca.crt
Added trusted certificate from ../../../my/spacetimedb-cert-gen/ca.crt for a new TLS connection.
Domain set to foo2 for identity c200ba48494b35ab616cea275571f895d0ac6bf6e16b391238bf59ec65d1410d.

$ spacetime list --cert ../../../my/spacetimedb-cert-gen/ca.crt
WARNING: This command is UNSTABLE and subject to breaking changes.

Added trusted certificate from ../../../my/spacetimedb-cert-gen/ca.crt for a new TLS connection.
Associated database identities for c200ade7f794c9fcba8d5ed1bf5e81d187c1e386c27886e64b396e0dceb6c816:

 db_identity                                                      
------------------------------------------------------------------
 c200d126731c13032f25f55f69caefb7457542c39c8f5b4279ddd10f299ee677 
 c200d854a8dff255f9a034f26ae374ffcee30dc6428dbffe6c628a655b9670ae 
 c200ba48494b35ab616cea275571f895d0ac6bf6e16b391238bf59ec65d1410d
```
```
$ spacetime energy balance --cert ../my/spacetimedb-cert-gen/ca.crt
WARNING: This command is UNSTABLE and subject to breaking changes.

Added trusted certificate from ../my/spacetimedb-cert-gen/ca.crt for a new TLS connection.
{"balance":"0"}
```
  
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
    (assuming you're in the root of the repo, else adjust path to certs appropriately)  
    `spacetime start --edition standalone --listen-addr 127.0.0.1:3000 --ssl --cert ../spacetimedb-cert-gen/server.crt --key ../spacetimedb-cert-gen/server.key`
  - start a rust client from a different terminal and connect to the server in plaintext mode:  
    `cd ./crates/sdk/` (this is in SpacetimeDB repo)  
    `cargo run --example quickstart-chat`
  - The error seen on client is:  
`thread 'main' panicked at crates/sdk/examples/quickstart-chat/main.rs:79:10:`  
`Failed to connect: FailedToConnect { source: InternalError { message: "Failed to initiate WebSocket connection", cause: Some(Tungstenite { uri: ws://localhost:3000/v1/database/quickstart-chat/subscribe?connection_id=e54407d34c5347e362ea91ef6f03fbc3&compression=Brotli, source: Protocol(HttparseError(Version)) }) } }`  
  - the meaning of the error is: your client is using plaintext connection to a TLS server, either both should be plaintext or both should be TLS. The server doesn't, currently, know how to handle both types on the same port.  
  
Verify that the server is accepting only TLS 1.3 and not TLS 1.2:  
  - this should work:  
  `openssl s_client -connect 127.0.0.1:3000 -tls1_3`  
  - this should fail:  
  `openssl s_client -connect 127.0.0.1:3000 -tls1_2`  
  - and to make sure the certificate verification isn't ignored (ie. fail if verify fails):  
  `openssl s_client -connect 127.0.0.1:3000 -tls1_3 -verify_return_error -CAfile ../spacetimedb-cert-gen/ca.crt`  
  
In Arch Linux, at least, since this uses openssl under the hood, you can temporarily use a different cert trust store by setting environment variable `SSL_CERT_DIR`, or even a specific file bundle by setting env. var. `SSL_CERT_FILE`, before running the program (server or client(s)), as per openssl [docs](https://docs.openssl.org/3.5/man3/SSL_CTX_load_verify_locations/#description).  
For example:
  - make a temporary trust store:  
```
mkdir /tmp/custom_trust_store
cp /path/to/serverSS.crt /tmp/custom_trust_store/
cp /path/to/ca.crt /tmp/custom_trust_store/
c_rehash /tmp/custom_trust_store
```
This creates files like `/tmp/custom_trust_store/123abcde.0` linking to your certificates, which OpenSSL uses for trust verification.  
`c_rehash` is from `openssl` package (on Arch Linux)  
```
export SSL_CERT_DIR=/tmp/custom_trust_store
```
or, alternatively: `export SSL_CERT_FILE=/path/to/ca.crt` or to `serverSS.crt`(but not `server.crt` which is signed by the CA and can only be verified via `ca.crt`, but you can concat `server.crt` and `ca.crt` into `bundle.crt` via `cat` and point to it here and it will work then)  
Now you can run any client:  
```
cargo run --example quickstart-chat
```
this shouldn't complain anymore about the issuer.  
Also see `man 1 trust` if you want to affect the system trust store via the `trust` command (it's from `p11-kit` Arch Linux package) but note that the env. variables mentioned above have no effect on it (try `trust list`)  

## Credits
Made with xAI's Grok3 and Grok2 because I don't know much of how to Rust or SpacetimeDB. So if things seem wrong, or suboptimal, they might be.

## License
- Business Source License 1.1 (see `LICENSE.txt`), switches to AGPLv3 on March 27, 2030

