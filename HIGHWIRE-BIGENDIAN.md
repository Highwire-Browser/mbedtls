# HighWire big-endian fixes for Atari 68k

Branch: `highwire-m68k-bigendian`, based on the upstream `mbedtls-3.6.7` tag.

These are the changes [HighWire](https://github.com/Highwire-Browser/highwire) needs
to speak TLS on a big-endian m68k. They are NOT submitted upstream and are not
offered as a contribution.

## The three real bugs

Each breaks a different stage of a TLS 1.3 handshake on any big-endian target
(m68k, PowerPC, SPARC, s390x, MIPS BE), and all three are present in upstream 3.6.7.

| file | function | symptom |
|---|---|---|
| `library/bignum_core.c` | `mbedtls_mpi_core_read_le()` | `MBEDTLS_ERR_ECP_INVALID_KEY` — X25519 key exchange fails |
| `library/gcm.c` | `gcm_mult_smalltable()` | `MBEDTLS_ERR_SSL_INVALID_MAC` — every AES-GCM suite fails |
| `library/bignum_core.h` | `GET_BYTE()` | RSA certificate signatures verify as garbage |

Two of those break the *default* modern TLS 1.3 configuration.

## Two platform changes, held to a lower standard

Recorded honestly rather than presented as fixes:

- **`library/alignment.h`** forces `MBEDTLS_IS_BIG_ENDIAN` on m68k under a comment
  reading "FORCE BIG-ENDIAN FOR M68K DEBUGGING". It works, but it reads like a
  debugging workaround that was never revisited — the right fix is probably to
  find why the normal detection does not fire on this target.
- **`library/platform_util.c`** adds a `gettimeofday()` fallback for
  `mbedtls_ms_time()` where `CLOCK_MONOTONIC` is unavailable, as on FreeMiNT.

## Not a patch: the startup hang

mbedTLS 3.6.6 changed the default of `MBEDTLS_PLATFORM_DEV_RANDOM` from
`/dev/urandom` to `/dev/random`. On FreeMiNT `/dev/random` blocks until the
entropy pool fills, and a machine idling at a splash screen never fills it — so
the application hangs inside `psa_crypto_init()` with no error and no log line.

Fix is one assignment before entropy is gathered, no rebuild of mbedTLS needed:

```c
mbedtls_platform_dev_random = "/dev/urandom";
```
