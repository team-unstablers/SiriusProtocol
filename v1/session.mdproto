---
syntax: "proto3"
package: "sirius.msgdef.v1"
import:
  - "common-types.proto"
---

# SESSION MANAGEMENT

Message definitions for session authentication and management in the Sirius protocol.
After the handshake (`ClientHello` / `ServerHello`), the user authentication process takes place before any features can be used.

---

# CONSTANTS

## AuthMethod

```constset
constset AuthMethod: string {
  /// No authentication required (NOT recommended for security reasons)
  const none = "none";

  /// Simple password authentication (uses a password pre-configured by the server)
  const simplePassword = "simple-password";

  /// Account/password-based authentication (e.g., OS account integration)
  const password = "password";

  /// SSH key-based authentication
  const sshKey = "ssh-key";
}
```

### DESCRIPTION

- Authentication method identifiers used in `AuthChallenge` and `AuthRequest` messages.
- This is an open enum of string type, allowing new authentication methods to be added in the future.

---

# MESSAGES

## AuthChallenge

```protobuf
/// Message sent by the server to request authentication from the client.
// @opcode: 0x0011
message AuthChallenge {
  /// List of authentication methods accepted by the server
  // @constset: AuthMethod
  repeated string acceptedMethods = 1;

  /// Random data (nonce) to prevent replay attacks
  /// Recommended length: 32 bytes or more
  bytes nonce = 2;

  /// Authentication prompt message to display to the user
  optional string message = 3;
}
```

### DESCRIPTION

- Sent by the server to inform the client that authentication is required, presenting the available authentication methods.
- Upon receiving this message, the client MUST select one of the `acceptedMethods` and send an `AuthRequest`.
- The `nonce` is used to ensure the integrity of the authentication process and to prevent replay attacks.

## AuthRequest

```protobuf
/// Message sent by the client to submit authentication credentials to the server.
// @opcode: 0x0012
message AuthRequest {
  /// The authentication method selected by the client
  // @constset: AuthMethod
  string method = 1;

  /// MUST include the nonce value received from the server as-is
  bytes nonce = 2;

  /// Authentication data (password, signature, etc.). Format varies by authentication method.
  optional bytes payload = 3;
}
```

### DESCRIPTION

- Sent by the client to submit the selected authentication method and corresponding credentials to the server.
- The `payload` contains authentication information encoded in the format appropriate for the selected `method`.

### IMPLEMENTATION NOTES

- When `method` is `none`, the `payload` MAY be empty.
- When `method` is `password` or `simple-password`, the `payload` MAY contain encrypted or hashed password information. The specific format is determined by the security requirements of the server implementation.
- The server MUST verify that the `nonce` matches the value it originally issued.

## AuthResponse

```protobuf
/// Response message sent by the server upon successful authentication.
// @opcode: 0x0013
message AuthResponse {
  /// Unique ID of the created session
  SRUUID sessionId = 1;
}
```

### DESCRIPTION

- Indicates that the authentication process has been completed successfully.
- The `sessionId` is a unique UUID identifying the session.
- On authentication failure, a `ServerNotice` (ERROR/FATAL) or a new `AuthChallenge` (retry request) is sent instead of this message.
