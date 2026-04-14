# **[DRAFT]** SVSM Observability and Configuration Protocol

The SVSM Observability and Configuration Protocol (OCP) is a new sub-protocol
of the SVSM specification and uses protocol ID value 5. The initial version of
the OCP is 1. Versioning is strictly additive, i.e., all calls present in a
protocol version must also be present in any later version.

The following enumerates the set of calls supported by version 1 of the OCP:

| Call ID | First version supported | Name | Function |
|--------:|------------------------:|------|----------|
| 0 | 1 | `SVSM_OCP_LIST` | Query a list of supported observability and configuration sources. |
| 1 | 1 | `SVSM_OCP_READ` | Read data from an observability source. |
| 2 | 1 | `SVSM_OCP_WRITE` | Write data to a configuration source. |

## SVSM_OCP_LIST Call

The call is used to query a list or sub-list of observability and configuration
sources of the SVSM. The SVSM maintains a list of these sources in an array and
specific sources are addressed via their index into that array. Any array index
is stable during the runtime of the SVSM, only new sources can be added, but no
sources can be removed.

### Registers

| Register | Size (Bytes) | Alignment | In/Out | Description                            |
|----------|-------------:|----------:|:------:|----------------------------------------|
| `RAX`    | 4            |           | OUT    | Result value                           |
| `RCX`    | 4            |           | IN     | First index to return from the array   |
| `RCX`    | 4            |           | OUT    | Number of array entries returned       |
| `RDX`    | 4            |         8 | IN     | GPA of a buffer to store array entries |
| `R8`     | 4            |           | IN     | Number of array entries to return      |

The `SVSM_OCP_LIST` queries a subset of the sources array. The first index to
be returned is passed via the `RAX` register. The `RCX` register points to a
GPA where the SVSM will store the array of entries. The `RDX` register
specifies a maximum number of array entries to return.

The number of entries returned by this call is stored in the `RAX` register. If
the returned value is less than the value of `RDX` passed in, then the end of
the array has been reached.

### Source entry layout

One entry in the array is 128 bytes in size and uses the following layout:

| Offset | Size (Bytes) | Description                         |
|-------:|-------------:|-------------------------------------|
| `0x00` | 4            | Flags                               |
| `0x04` | 124          | Name of the source encoded as UTF-8 |

The Flags stored in each entry are a bit field stored in little-endian byte
order. The defined flags are:

| Bit(s) | Name           | Description                                    |
|-------:|----------------|------------------------------------------------|
| 0      | `WRITEABLE`    | The source supports the `SVSM_OCP_WRITE` call. |
| 31:1   | Reserved – MBZ | All other bits are reserved and must be zero.  |

## SVSM_OCP_READ Call

This call reads data from a single observability or configuration source.

### Registers

| Register | Size (Bytes) | Alignment | In/Out | Description                                                                       |
|----------|-------------:|----------:|:------:|-----------------------------------------------------------------------------------|
| `RAX`    | 4            |           | OUT    | Result value                                                                      | 
| `RCX`    | 4            |           | IN     | Array index of the source to read from                                            |
| `RDX`    | 8            | 8         | IN     | GPA of buffer to copy data into                                                   |
| `R8`     | 4            |           | IN     | Number of bytes to read                                                           |
| `R8`     | 4            |           | OUT    | Number of bytes copied. If less than requested then end of data has been reached. |
| `R9`     | 4            |           | IN     | Byte offset into data to start read from                                          |

The `SVSM_OCP_READ` call will copy the requested amount of data from a source
into a buffer provided by the caller in `RCX`. Reading starts at the offset
specified in `R8`.

If the specified source index in `RAX` does not exist, the call will return
`SVSM_ERR_INVALID_PARAMETER`.

## SVSM_OCP_WRITE Call

This call will attempt to write data into a specified observability or
configuration source.

### Registers

| Register | Size (Bytes) | Alignment | In/Out | Description                                        |
|----------|-------------:|----------:|:------:|----------------------------------------------------|
| `RAX`    | 4            |           | OUT    | Result value                                       |
| `RCX`    | 4            |           | IN     | Array index of the source to write to              |
| `RDX`    | 8            | 8         | IN     | GPA of buffer with data to write                   |
| `R8 `    | 4            |           | IN     | Number of bytes to write                           |
| `R8`     | 4            |           | OUT    | Number of bytes written                            |
| `R9`     | 4            |           | IN     | Byte offset into data to start the write operation |

The `SVSM_OCP_WRITE` call attempts to write data from buffer specified in `RCX`
to the observability or configuration source specified in `RAX`. The size of
the data to write is specified in `RDX` and the offset to write the data to in
`R8`.

Sources can only be written to if the Flags field in the `SVSM_OCP_LIST` call
has the `WRITEABLE` bit set. If the source is not writable the call will return
`SVSM_ERR_INVALID_PARAMETER`.

The format of the data allowed to write is source dependent. If a given data
format is not understood by the source the call will also return
`SVSM_ERR_INVALID_PARAMETER`.
