# BOM #6: Rendezvous layer

The Rendezvous layer performs an application's part in establishing a communications channel with another application.

## Simple and Complex roles
An instance of the Rendezvous layer can take on either `Simple` or `Complex` roles.

The `Simple` role is intended for devices that rely on other infrastructure to evaluate the rendezvous. e.g. a `Complex` or `Relay` role on the other side of a connection. It is simple to implement and is a choice for a simple peripheral that operates in an intuitve way without much user-inputted dynamic behavior.

The `Complex` role is intended for a device that exhibits complex interactive behavior such as connecting to multiple other devices and managing the connections. It is capable of evaluating a rendezvous if a `Simple` device is on the other side of the connection. The application utilizing the Moneysocket protocol stack may be given a chance to direct dynamic behavior. This role may be appropriate for a Lightning Network "wallet" or similar style of app.

## Relay Role

A third role named `Relay` is defined which evaluates the rendezvous between two other device that are connected. The two devices can be any role. Once rendeavous is achieved, it facilitates the pasisng of messages between the two devices encrypted by the encryption key derived from the Shared Seed, of which the Relay cannot decrypt.

## Values

### Shared Seed

The 128-bit `Shared Seed` value is exchanged in the beacon between two applications seeking rendezvous.

### Rendezvous ID

The `Rendezvous ID` value is derived from the 128-bit `Shared Seed` value encoded in the beacon.

The `Rendezvous ID` value is the 256-bit value that is the result of the 2nd round of SHA256 applied to the 128-bit `Shared Seed` value. The 1st round of SHA256 applied to the 128-bit shared seed value is the `AES Encryption Key`. Hence, the `Rendezvous ID` is also the result of applying SHA256 result to the `AES Encryption Key`.

### Encryption Key

All post-rendezvous content passed through the layer and via connection will be encrypted with a layer of ChaChaPoly20 encryption. This ensures that anyone who has not observed the beacon cannot decrypt the content. This 256-bit value is the result of a SHA256 operation on the 128-bit `Shared Seed` value.


### Tiebreaker

The `Tiebreaker` is a 64-bit value to be formatted as a hexadecimal string. Its function is to provide a way to automatically resolve which application among the two at the ends of a connection takes on the `Complex` role in the case of a conflict.

In the case of a conflict during the handshake, the application with the highest numerical value will evaluate the rendezvous.

For automatic resolution, an application should select a randomly-generated value. To insist on evaluating a rendezvou, an application should select this value to be `"ffffffffffffffff"`. To defer to the other side (but still allow oneself to evaluate the rendezvou when confronted with a `Simple` role on the other end), an application should select this value to be `"0000000000000000"`. If there is a value collision, the rendezvous cannot proceed.


## Messages

Messages are encoded as defined in [04-messages.md](BOM #4).

### REQUEST\_RENDEZVOUS

- This message is exempt from transport layer encryption

The TLV data
- MUST set `message_type` value to `0x00`
- MUST set `message_subtype` value to `0x01`

The Language Object:

- MUST set `message_type` String-typed value to `REQUEST`
- MUST set `message_subtype` String-typed value to `RENDEZVOUS`
- MUST have a name/value pair `role` with a value of String type
    - MUST be set to either `SIMPLE`, `COMPLEX`, or `RELAY` to indicate the role assigned to the layer implementation

- if `role` is set to `SIMPLE`
    - MUST have a name/value pair `rendezvous_id` with a value of String type.
        - MUST be set to a hex-formatted string of 256-bit `rendezvous_id` value

- if `role` is set to `COMPLEX`
    - MUST have a name/value pair `tiebreaker` with a value of String type
        - MUST be randomly-generated or specifically-chosen according to desired Tiebreaking behavior
            - if automatic tiebreaking desired:
                - MUST set to hex-formatted random 64-bit value
            - if self-prioritize tiebreaking desired:
                - MUST set to `"ffffffffffffffff"`
            - if self-deprioritize tiebreaking desired:
                - MUST set to `"0000000000000000"`
    - MUST have a name/value pair `rendezvous_id` with a value of String type.
        - TODO
        - if the application is capable of falling back to `` role in the event of a tiebreaker:
            - MUST be set to a hex-formatted string of 256-bit `rendezvous_id` value
        - if the application is not capable of falling back to `SIMPLE` role in the event of a tiebreaker:
            - MUST be set to `null`

- if `role` is set to `RELAY`
    - TODO


### NOTIFY\_RENDEZVOUS

- This message is exempt from transport layer encryption (TODO - should it be encrypted?)

The TLV data
- MUST set `message_type` value to `0x01`
- MUST set `message_subtype` value to `0x02`

The Language Object:

- MUST set `message_type` String-typed value to `NOTIFY`
- MUST set `message_subtype` String-typed value to `RENDEZVOUS`
- MUST set 'request_reference_uuid` String-typed value to the `request_uuid` of the corresponding `REQUEST_RENDEZVOUS` request.
- MUST have a name/value pair `rendezvous_id` with a value of String type.
    - MUST be set to a hex-formatted string of 256-bit `rendezvous_id` value
    - MUST be the value of the `rendezvous_id` in the corresponding `REQUEST_RENDEZVOUS` request.


### NOTIFY\_RENDEZVOUS\_NOT\_READY

- This message is exempt from transport layer encryption

The TLV data
- MUST set `message_type` value to `0x01`
- MUST set `message_subtype` value to `0x03`

The Language Object:

- MUST set `message_type` String-typed value to `NOTIFY`
- MUST set `message_subtype` String-typed value to `RENDEZVOUS_NOT_READY`
- MUST set 'request_reference_uuid` String-typed value to the `request_uuid` of the corresponding `REQUEST_RENDEZVOUS` request.
- MUST have a name/value pair `rendezvous_id` with a value of String type.
    - MUST be set to a hex-formatted string of 256-bit `rendezvous_id` value
    - MUST be the value of the `rendezvous_id` in the corresponding `REQUEST_RENDEZVOUS` request.

## Behavior

### Handshake

- Upon receiving a nexus announcement from layer below:
    - MUST send a `REQUEST_RENDEZVOUS` message.
    - MUST set `role` to be `COMPLEX` or `SIMPLE`
    - if `SIMPLE` role:
        - MUST set a `rendezvous_id`
    - if `COMPLEX` role:
        - MUST set a `tiebreaker` value
        - MAY set a `rendezvous_id` value

- Upon receiving a `REQUEST_RENDEZVOUS` message.
    - if local role is `SIMPLE` and the message indicates the remote role is `COMPLEX`
        - MUST ignore any `REQUEST_RENDEZVOUS` messages from other end
        - MUST wait for remote role to send `NOTIFY_RENDEZVOUS` `NOTIFY_RENDEZVOUS_NOT_READY`

    - if local role is `COMPLEX` and the message indicates the remote role is `SIMPLE`
        - if `rendezvous_id` matches an previous `SIMPLE` connection waiting for rendezvous
            - MUST send a `NOTIFY_RENDEZVOUS` notification
            - MUST announce a nexus for the two single connection to the above layer
        - if `rendezvous_id` does not match a previous `SIMPLE` connection waiting for rendezvous
            - MUST send a `NOTIFY_RENDEZVOUS_NOT_READY` notification
            - MUST NOT announce a nexus to the above layer

    - if local role is `COMPLEX` and the message indicates the remote role is `COMPLEX`
        - TODO figure out implementation and test and clarify
        - If the `tiebreaker` evaluates local application as the winner

        - if the message has a `rendezvous_id` set

        - if the message does not have a `rendezvous_id` set
            - if the local application does not have a `rendezvous_id` for the connection
                - MUST send an `ERROR_NOTIFICATION`
                - MUST disconnect within 1 second
        - if the message does not have a `rendezvous_id` set

- Upon receiving a `NOTIFY_RENDEZVOUS` message.
    - MUST announce a nexus to above layer

- Upon receiving a `NOTIFY_RENDEZVOUS_NOT_READY` message.
    - MAY keep the connection open to wait for `NOTIFY_RENDEZVOUS`

- Upon receiving a `NOTIFY_RENDEZVOUS_END` message over a connection
    - If role is `SIMPLE`
        - MUST disconnect connection

    - If role is `COMPLEX`
        - if two connections have been connected in rendezvous
            - MUST echo `NOTIFY_RENDEZVOUS_END` to the other connection
        - MUST disconnect all connections associated with `rendezvous_id`

    - MAY attempt a reconnect with same `rendezvous_id` after disconnect is completed

## Appendix A - Shared Seed Derivation Test Vectors

