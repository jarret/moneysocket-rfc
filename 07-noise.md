
# BOM #7: Noise protocol handshake layer

The noise protocol layer establishes a secure ephemeral communications layer between the two connected devices using [Noise Protocol](http://noiseprotocol.org) XX (TODO maybe XK auth also).

## Noise Protocol

### Noise XX

Each instance of the Moneysocket protocol stack has a static private key that is persisted and long-lived. This is used to identify the device to the counterpart and create an encrypted session.

In NoiseXX, the static key (`s`) needs to be transmitted from each device to the other device, along with an short-lived ephemeral key (`e`) that is generated for a particular connection (and discarded immediately after the connection closes).


### Tiebreaker

For Noise Protocol, there needs to be a common understanding of which of the two devices entering the handshake is the "initiator". For many protocols, the initiator of the connection is an obvious choice, but in this layer of abstraction for Moneysocket, the choice is not so clear. Hence, we need to select one of the two participants as the initiator automatically. We do so by including a `tiebreaker` value on the handshake initiation. When a connection is established, each side will attempt to initiate the handshake with a tiebreaker value set. The side with the highest value will proceed as the initiator for noise protocol handshake purposes. The side with the lowest value will abandon the attempt to be the initiator and will proceed as the responder.


### Message Encryption

Once the handshake is completed and the layer announces a nexus above, subsequent messages submitted to the layer will be put through the appropriate encryption using the established ephemeral keys and derived shared secret.

## Messages

### REQUEST\_NOISE

The TLV Data

- MUST set `message_type` value to `0x00`
- MUST set `message_subtype` value to `0x04`

The Language Object:

- MUST set `message_type` String-typed value to `REQUEST`
- MUST set `message_subtype` String-typed value to `NOISE`
- MUST set `noise_type` to String-typed value to `Noise_XX_secp256k1_ChaChaPoly_SHA256`
- MUST have a name/value pair `tiebreaker` with a value of String type
    - MUST be randomly-generated or specifically-chosen according to desired Tiebreaking behavior
        - if automatic tiebreaking desired:
            - MUST set to hex-formatted random 64-bit value
        - if self-prioritize tiebreaking desired:
            - MUST set to `"ffffffffffffffff"`
        - if self-deprioritize tiebreaking desired:
            - MUST set to `"0000000000000000"`
- MUST set `noise_message` to String-typed value that is a hexidecimal representation of the data packet expected for the initiation of Noise\_XX\_secp256k1\_ChaChaPoly\_SHA256

### NOTIFY\_NOISE

The TLV Data

- MUST set `message_type` value to `0x01`
- MUST set `message_subtype` value to `0x04`

The Language Object:

- MUST set `message_type` String-typed value to `NOTIFY`
- MUST set `message_subtype` String-typed value to `NOISE`
- MUST set `noise_type` to String-typed value to `Noise_XX_secp256k1_ChaChaPoly_SHA256`
- MUST set `noise_message` to String-typed value that is a hexidecimal representation of the data packet expected for the reply to the initiation of Noise\_XX\_secp256k1\_ChaChaPoly\_SHA256


### NOTIFY\_NOISE\_COMPLETE

The TLV Data

- MUST set `message_type` value to `0x01`
- MUST set `message_subtype` value to `0x05`


- MUST set `message_type` String-typed value to `NOTIFY`
- MUST set `message_subtype` String-typed value to `NOISE`
- MUST set `noise_type` to String-typed value to `Noise_XX_secp256k1_ChaChaPoly_SHA256`
- MUST set `noise_message` to String-typed value that is a hexidecimal representation of the data packet expected for the completion of the hanshake Noise\_XX\_secp256k1\_ChaChaPoly\_SHA256

