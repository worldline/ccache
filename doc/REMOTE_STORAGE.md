# ccache Remote Storage TLV Protocol Specification

## 1. Introduction

This document defines the Type-Length-Value (TLV) protocol used for communication between the `ccache` utility and its remote storage backend processes. This protocol enables `ccache` to offload storage operations (like get, put, delete) to a separate server process that manages connections to various remote storage systems.

## 2. Protocol Basics

### 2.1. Message Structure

All messages exchanged between `ccache` (the client) and the remote storage backend (the server) adhere to a nested TLV-based structure. A message consists of a fixed `MessageHeader` followed by a sequence of encoded fields.

```html
┌────────────────────────────────────────────────────────┐
│                    Message Header                      │
├────────────────────────────────────────────────────────┤
│                     TV Field 1                         │
├────────────────────────────────────────────────────────┤
│                     TV Field 2                         │
├────────────────────────────────────────────────────────┤
│                       ...                              │
├────────────────────────────────────────────────────────┤
│                     TV Field N                         │
└────────────────────────────────────────────────────────┘
```

### 2.2. Byte Order and Endianness

All multi-byte numeric values within the protocol (e.g., `version`, `msg_type`, lengths) MUST be encoded in **host byte order**.

## 3. Message Header

The `MessageHeader` provides fundamental information about the message. It contains the tag representing the message type and the message length, which can be up to $2^{64}$ bytes. We do not expect messages to have a payload larger than this. The header size (`TLV_HEADER_SIZE`) is `12` bytes (`4B` msg_type + `8B` length).

**Structure:**

  ```c++
  struct __attribute__((packed)) MessageHeader {
      uint32_t msg_type;     // Type of message
      uint64_t length;       // Number of bytes in this message
  };
  ```

## 4. The Field Format

Each field consists of a Tag, and a Value (for simplicity we call them TV fields).

```html
┌───────────┬───────────────────────────────────────┐
│  Tag      │                 Value                 │
│ (uint8_t) │            (Length bytes)             │
└───────────┴───────────────────────────────────────┘
```

* **Tag (`uint8_t`):** Identifies the type of data in the `Value` field.
  * **SETUP types:** `0x01` to `0x80`
  * **Application types:** `0x81` to `0xFF`
* **Value:** The actual data payload, interpreted based on the `Tag`. The tag dictates the number of bytes to read for the `Value`.

## 5. Field and Message Type Definitions

### 5.1. Tags

Application tags range between `0x01` - `0xFF`.

| Tag Name             | Value      | Data Type | Description                                           | Required/Optional (Context Dependent) |
| :------------------- | :--------- | :-------- | :---------------------------------------------------- | :------------------------------------ |
| `FIELD_TYPE_KEY`     | `0x01`     | `bytes`  | The key for storage operations (e.g., cache object hash). | REQUIRED for GET, PUT, DEL            |
| `FIELD_TYPE_VALUE`   | `0x02`     | `bytes`   | The data payload for storage operations.              | REQUIRED for PUT; OPTIONAL for GET    |
| `FIELD_TYPE_STATUS_CODE`| `0x03`     | `ResponseStatus` | Status code of the operation.                         | REQUIRED for responses                |
| `FIELD_TYPE_ERROR_MSG`| `0x04`     | `string`  | Detailed error message.                               | MUST NOT on success            |
| `FIELD_TYPE_FLAGS`   | `0x05`     | `uint8_t` | Flags for operations (e.g., overwrite).               | OPTIONAL                              |

### 5.2. Message Types

Message types are interpreted as `uint32_t` types. The greeting message is encoded with `0x00`. Request types are in the `0x01`-`0x0F` range, and their corresponding responses are in the `0x81`-`0x8F` range.

| Message Type Name           | Value   | Direction | Description                                       |
| :-------------------------- | :------ | :-------- | :------------------------------------------------ |
| `MSG_TYPE_GREETING`         | `0x00`| Server->C | Server's greeting message for initialisation.     |
| `MSG_TYPE_GET_REQUEST`      | `0x01`  | Client->S | Request to retrieve a value by its key.           |
| `MSG_TYPE_GET_RESPONSE`     | `0x81`| Server->C | Response to a GET request.                        |
| `MSG_TYPE_PUT_REQUEST`      | `0x02`  | Client->S | Request to store a key-value pair.                |
| `MSG_TYPE_PUT_RESPONSE`     | `0x82`| Server->C | Response to a PUT request.                        |
| `MSG_TYPE_DEL_REQUEST`      | `0x03`  | Client->S | Request to remove a key.                          |
| `MSG_TYPE_DEL_RESPONSE`     | `0x83`| Server->C | Response to a DELETE request.                     |

### 5.3. Constants and Enums

* `TLV_HEADER_SIZE`: `12` bytes.
* `OVERWRITE_FLAG` (`0x01`): Used with `FIELD_TYPE_FLAGS` to indicate overwrite permission for PUT operations.

#### 5.3.1. `ResponseStatus` Enum

Used within `FIELD_TYPE_STATUS_CODE`.

| Status Code       | Value     | Description                                          |
| :---------------- | :-------- | :--------------------------------------------------- |
| `SUCCESS`         | `0x00`    | Operation completed successfully.                    |
| `NO_FILE`         | `0x01`    | Key not found.                                       |
| `TIMEOUT`         | `0x02`    | Operation timed out.                                 |
| `LOCAL_ERROR`     | `0x03`    | Error on the local (client) side.                    |
| `ERROR`           | `0x04`    | A general error occurred.                            |

## 6. Message Details and Semantics

### Definitions

**The message:**

```markdown
<msg>          ::= <msg_tag> <msg_len> <msg_data>
                ; also = (<greet_msg> | <get_req> | <put_req> | <remove_req> |
                             <get_resp> | <put_resp> | <remove_resp>)
<msg_len>      ::= uint64_t
<msg_data>     ::= uint8_t*        ; <msg_len> bytes, UTF-8
                                   ; includes TV fields
```

**Some TV-fields:**

```markdown
<flags>        ::= <FIELD_TYPE_FLAGS> uint8_t   ; bit 0 (LSB): overwrite, other bits: not defined
<key>          ::= <FIELD_TYPE_KEY> <key_data>
<key_data>     ::= bytes   ; 20 bytes
<value>        ::= <FIELD_TYPE_VALUE> <value_data> ; MUST come as last fields in message
                                                   ; else length is not deducible
<value_data>   ::= uint8_t*
```

### Server greeting to client

```markdown
<greet_msg>    ::= <MSG_TYPE_GREETING> <msg_len> <greeting>
<greeting>     ::= (<version> <cap_len> <capabilities>)* ; version for capabilities
                                                         ; supports 255 versions and 255
                                                         ; capabilities 
<version> ::= 0x00               ; capabilities version 1
<cap_len> ::= uint8_t            ; 255 entries supported
<capabilities> ::= <cap_tag>*    ; <cap_len> uint8_t entries
                                 ; capability: get/put/remove
```

### Client requests

```markdown
<request>      ::= <get_req> | <put_req> | <remove_req>
<get_req>      ::= <MSG_TYPE_GET_REQUEST> <length> <key>
<put_req>      ::= <MSG_TYPE_PUT_REQUEST> <length> <flags> <key> <value>
<remove_req>   ::= <MSG_TYPE_DEL_REQUEST> <length> <key>
```

### Server responses

```markdown
<response>     ::= <get_resp> | <put_resp> | <remove_resp>
<get_resp>     ::= <MSG_TYPE_GET_RESPONSE> <length> (<ok> <value> | <noop> | <err> | <timeout>)
<put_resp>     ::= <MSG_TYPE_PUT_RESPONSE> <length> (<ok>         | <noop> | <err> | <timeout>)
<remove_resp>  ::= <MSG_TYPE_DEL_RESPONSE> <length> (<ok>         | <noop> | <err> | <timeout>)
<ok>           ::= <FIELD_TYPE_STATUS_CODE> 0x00           ; operation done
<noop>         ::= <FIELD_TYPE_STATUS_CODE> 0x01           ; operation not done (key not found/stored/removed)
<err>          ::= <FIELD_TYPE_STATUS_CODE> 0x02 <err_msg> ; e.g. bad parameters or failed connection
<timeout>      ::= <FIELD_TYPE_STATUS_CODE> 0x03           ; e.g. slow host lookup or connection setup
<err_msg>      ::= <FIELD_TYPE_ERROR_MESSAGE> uint8_t*     ; bytes represent a string (use for logging)
```

## 7. Configuration and Negotiation

### 7.1. Passing Configuration

`ccache` parses configuration items of a provided URL and passes it to the client.

* **Remote URL:** Passed via **environment variables** to the backend process as `_CCACHE_REMOTE_URL`.
* **Socket Path:** Passed as `_CCACHE_SOCKET_PATH`.
* **Buffer size:** Propagated as `_CCACHE_BUFFER_SIZE`, such that it becomes less necessary to communicate it within the protocol.
* **Attributes:**: Client should read `_CCACHE_NUM_ATTR` first before accessing the attributes; each is defined as a pair (`_CCACHE_ATTR_KEY_i`, `_CCACHE_ATTR_VALUE_i`) for $0\leq i <$ `_CCACHE_NUM_ATTR`.

The backend process is responsible for parsing and acting upon these attributes and credentials.

### 7.2. Capabilities Negotiation

The SETUP phase is intended for negotiation. As it stands, the server initially sends a greeting to initialise the connection. The client check the available configurations and directly sends fitting messages to the proxy. If the server does not support a fitting setup, the client may immediately disconnect and no remote storage will take place. This model reduces the setup to a single message.

### TODO Timeouts -- Needs reviewing

The `SETUP_TYPE_OPERATION_TIMEOUT` tag can be used to convey timeout information. The interpretation depends on the context:

* **Client to Server:** The client MAY send `SETUP_TYPE_OPERATION_TIMEOUT` to suggest a timeout for **operation timeout**. This specifies how long the backend process should wait before it times out on an operation.
* **Server to Client:** The server MAY send `SETUP_TYPE_OPERATION_TIMEOUT` in `SETUP_RESPONSE` to specify its configured **operation timeout**. It MAY also suggest a different **operation timeout**.

The client is responsible for managing its own "Connection timeout" (waiting for initial server reply) and "Operation timeout" (aborting an operation if it takes too long).
