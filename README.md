wrbt
====

**wrbt**, pronounced *wrabbit*, is a protocol that facilitates peering of cjdns nodes. The two parties peering through **wrbt** are able to exchange credentials securely and privately over an insecure channel. Peering of two cjdns nodes, owned by Alice and Bob, goes something like this.

1. Alice just set up a new node and is in need of a peer, so she generates a set of private and public keys that are completely independent of her cjdns keys. Let's call these `wrbtSk` and `wrbtPk`, respectively.

2. Alice sends out a request to peer via an insecure channel. Since the channel is insecure, we may as well assume it's a public broadcast to the entire world.

    ```
    scheme://host/?type=peer&interface=udp&link=overlay&pk=wrbtPk&metadata=metadataOfAlice&cjdnsVersion=X&wrbtVersion=Y
    ```

3. Bob sees Alice's request and offers to peer. From his node, he generates a new password for Alice and adds it to the list of `authorizedPasswords` in his node's **cjdroute.conf**.

    ```
    {
      "name": "Alice",
      "password": "generatedPasswordForAlice"
    }
    ```

4. Bob then composes the following peering credentials and signs it with his cjdns private key to get `signatureOfBob`.

    ```
    {
      "externalIpOfBob:port": {
        "name": "Bob",
        "publicKey": "cjdnsPublicKeyOfBob",
        "password": "generatedPasswordForAlice"
      }
    }
    ```

5. Bob then composes the peering credentials with the signature into the following message, then encrypts the whole thing with `wrbtPk` and a `generatedNonce`, resulting in `encryptedCredentialsForAlice`.

    ```
    {
      "credentials": {
        "externalIpOfBob:port": {
          "name": "Bob",
          "publicKey": "cjdnsPublicKeyOfBob",
          "password": "generatedPasswordForAlice"
        },
        "metadata": {}
      },
      "signature": "signatureOfBob"
    }
    ```

6. Bob responds to Alice either synchronously,

    ```
    {
      "encryptedCredentials": "encryptedCredentialsForAlice",
      "nonce": "generatedNonce"
    }
    ```

    or asynchronously, where `encodedMessage` is the above URL-encoded.

    ```
    scheme://host/?type=credentials&interface=udp&link=overlay&pk=wrbtPk&message=encodedMessage&cjdnsVersion=X&wrbtVersion=Y
    ```

7. In the asynchronous case, Alice is able to know that this is a response to her earlier request based on the `scheme://host` and `wrbtPk`. The latter also ensures that only she can decrypt the `encryptedCredentialsForAlice`. Alice proceeds to decrypt the message with her `wrbtSk` and `generatedNonce`, revealing `credentials` and `signature`. Alice then attempts to verify that this set of credentials actually came from its owner, i.e. the holder of its cjdns private key. So Alice takes the `publicKey` from `credentials` and hashes it with the `signature`, and verifies that it matches with the hash of the `credentials`. The application then proceeds to ask Alice whether she wants to peer with the following node.

    ```
    Name:     Bob
    Protocol: UDP via Clearnet Overlay
    Address:  fc00:0000:0000:0000:0000:0000:0000:BBBB
    Verified: true
    ```

8. Alice accepts the peering offer and the application adds `credentials` to **cjdroute.conf** of Alice's node. Her node starts connecting to Bob's node.

9. Bob's node sees connections coming in from Alice's node `fc00:0000:0000:0000:0000:0000:0000:AAAA` and optionally pins it to the entry for Alice in `authorizedPasswords`, so that password becomes dedicated to her node only.

>**IP-pinning** is optional.

## Cross-platform Peering & Platform-specific Handling

One way the [Android application](https://github.com/berlinmeshnet/cjdns-android) can add peers is through the `http://findpeer.hyperboria.net` implementation of **wrbt**. When Caleb installed the Android application, he does not know anyone on Hyperboria, so he tweeted the following message generated by the Android app, in hopes that someone will offer to peer with his Android node.

```
#followTheWhiteRabbit findpeer.hyperboria.net/?type=peer...
```

The URL, shortened by Twitter, expands into something like this when clicked.

```
http://findpeer.hyperboria.net/?type=peer&interface=udp&link=overlay&pk=wrbtPk&metadata=metadataOfCaleb&cjdnsVersion=X&wrbtVersion=Y
```

Danielle, who actively monitors the **#followTheWhiteRabbit** hashtag, is on her computer when she saw the tweet. That computer is already a cjdns node, so she decides to use a **wrbt** desktop application to generate a response, offering to peer with Caleb. The desktop application takes Caleb's URL as an input, and does all the **wrbt** magic as described above, resulting in the following which Danielle replies to Caleb with via Twitter.

```
http://findpeer.hyperboria.net/?type=credentials&interface=udp&link=overlay&pk=wrbtPk&message=encodedMessage&cjdnsVersion=X&wrbtVersion=Y
```

Ed, who saw Caleb's tweet from his phone, also happens to have cjdns running on his phone with the Android application. He clicks on the link, and is asked if he wants to open the link with the cjdns application, because application advertises itself as a handler for the scheme `http` and host `findpeer.hyperboria.net`. This allows the cjdns Android app to do what the desktop app did for Danielle.

Faye saw the tweet on her laptop, has no idea what this is, but clicks on the link anyways. She is directed to a welcome page, introducing her to the wonderful world of Hyperborea.

Georgina, who also knows nothing about the existance of a parallel internet, clicked on the tweet from her Android phone. `findpeer.hyperboria.net` is set up in a way that redirects all Android clients to `market://details?id=cjdns.android.package.name`, which opens up the cjdns Android application download page in Google Play.

## Auto-peering & Implementation-specific Behaviour

The implementation-specified `scheme://host` allows for implementation-specific behaviour. The previous example, `http://findpeer.hyperboria.net`, is only one example for an implementation. It is one where peering is asynchronous, requiring the other party to extend a delayed response after manually deciding to peer with Caleb, perhaps based on some criteria embedded in the [metadata](#optional-metadata).

You can easily imagine a cjdns node that synchronously hands out unique credentials to whoever asks for it at `http://[fc00:0000:0000:0000:0000:0000:0000:XXXX]`. This, of course, requires Caleb to have an existing peer to reach it in the first place. This same server can also hand out credentials in the clearnet by having a clearnet domain. Unrestricted handing out of peering credentials is probably self-destructive, but **wrbt** does not limit one from implementing whatever self-destructive behaviour. On the other hand, it ensures that the exchange of credentials and other private information are conducted in a secure manner, and by enforcing a unique password per peer, it allows the node to break away from any particularly abusive relationship. The node can also engage in hook-ups by issuing passwords that expire, i.e. removal from `authorizedPasswords` after some fixed time.

## Privacy & Anonymity

The private exchange of peering credentials is protected by `wrbtSk` and `wrbtPk`. It is important that each responding node generates a unique nonce when using `wrbtPk` to generate the `encryptedCredentials`.

Aside from the peering credentials, **wrbt** aims to keep the relation between the identity of a node, i.e. its `fc00::/8` address, and its operator from becoming public information. When Alice first advertises that she needs to find a peer, she has exposed that she operates a node associated with whatever metadata embedded in that peering request. However, that request does not contain her cjdns public key or `fc00::/8` address, so her node cannot be uniquely identified based on that information alone. It wasn't until she accepted Bob's peering offer that her `fc00::/8` address is communicated to Bob, and Bob alone, never publicly. This ensures the knowledge of the association between node identitiy and personal identity is limited to trusted peers.

Bob trusts Alice the moment he sends along his credentials in response to Alice's request. Alice has all the information necessary, i.e. `cjdnsPublicKeyOfBob` and `externalIpOfBob`, to associate Bob's node with his personal identity. **wrbt** ensures that the response can be decrypted by Alice alone, but Bob is solely responsible for judging to trust Alice when he sends along credentials.

## Verification of Identity & Man-in-the-middle Attacks

The signing of the credentials performed by Bob, with his cjdns private key, allows for Alice to verify that the credentials are in fact coming from the operator of the node. It means Bob is not just sending to her credentials of a random node that does not belong to him. If Alice can also verify that this response did in fact come from Bob, say, [a trusted third-party tweeted a fingerprint of Bob's response](https://firstlook.org/theintercept/2014/10/28/smuggling-snowden-secrets/), to rule out a man-in-the-middle attack, then Alice can verify that she is indeed peering with a node operated by Bob.

>**Signature** is optional.

This verification, however, may not always be necessary. Consider the case where Halley wants to get on Hyperboria, and her friend Icarus wants to give her peering credentials to his VPS from his Android phone, which is peered with the VPS. Icarus doesn't mind sharing the same password, or he has passwords set aside for situations like this. The point is, his Android node does not have the private key of the VPS node, so there is no way for Icarus to sign the credentials he is about to send Halley. Icarus will then send a peering response without a signature, and Halley's application would ask if she wants to peer with the following node.

```
Name:     Icarus
Protocol: UDP via Clearnet Overlay
Address:  fc00:0000:0000:0000:0000:0000:0000:IIII
Verified: false
```

## Insecure Broadcasting of Credentials

>**Encryption** is optional.

Some medium of exchange are inherently secure and allow for Bob to broadcast his peering credentials in clear text. One such use case is when Alice and Bob want to peer their Android nodes over NFC, by physically tapping their phones. In this case, Bob does not need any information from Alice, he simply broadcasts the following over NFC.

```
scheme://host/?type=credentials&interface=udp&link=overlay&message=encodedMessage&cjdnsVersion=X&wrbtVersion=Y
```

Note the absence of the `pk` query parameter, which in a secure exchange would have been acquired from Alice's peering request. Since this is a clear text broadcast, secured by proximity in the NFC case, the`encodedMessage` is just the URL-encoded text of the following unencrypted credentials. Upon receiving this, Alice's node should recognize this as an unencrypted message by the lack of its `pk` query parameter.

```
{
  "credentials": {
    "externalIpOfBob:port": {
      "name": "Bob",
      "publicKey": "cjdnsPublicKeyOfBob",
      "password": "generatedPasswordForAlice"
    }
  },
  "signature": "signatureOfBob"
}
```

## Standard Protocols for Connectivity

We need a standard way for Alice to, in her peering request, indicate to Bob how her node would like to peer with other nodes. By specifying the `interface` and `link`, Bob will be able to figure out whether his node can peer with Alice's, and unambiguously create a set of credentials for her.

|Interface |Query parameter |Support          |
|:---------|:---------------|:----------------|
|UDP       |`interface=udp` |IPv4 or IPv6     |
|ETH       |`interface=eth` |All beacon modes |

|Physical link    |Query parameter   |Via                       |UDP                |ETH                |
|:----------------|:-----------------|:-------------------------|:-----------------:|:-----------------:|
|WiFi Direct      |`link=wifiDirect` |Connected devices         |:heavy_check_mark: |:heavy_check_mark: |
|Bluetooth        |`link=bluetooth`  |Paired devices            |:heavy_check_mark: |:heavy_check_mark: |
|Clearnet Overlay |`link=overlay`    |WiFi AP or mobile network |:heavy_check_mark: |                   |

## Auto-peering Mobile Nodes & Ad-hoc Networks

The standard way for a cjdns node to handle a peering request should be to *ask* the operator whether to offer peering by responding with a set of credentials. However, in an ad-hoc network made up of mobile nodes, constantly moving in- and out-of-range, asking all the time may be impractical. Instead, the operator may elect to enable *auto-peering*. This does not mean a shared password or accepting passwordless unauthenticated connections. It only means the node will automatically and silently respond to peering requests with newly created credentials. This way, mobile nodes can acquire new peers as they move around, and the unique passwords allow each node to shut down a particular misbehaving peer.

## Unauthentication Connections

>**Password** is optional.

While most peering credentials should contain a password, some protocols can support unauthenticated connections. This would be common when connecting over the Ethernet interface. In this case, Alice specifies `interface=eth` in her peering request, then Bob sends credentials without `password`.

## Query Parameters

### Peering Request

```
scheme://host/?type=peer&interface=udp&link=overlay&pk=wrbtPk&metadata=metadataOfAlice&cjdnsVersion=X&wrbtVersion=Y
```

|Key            |Value                                                                           |Required           |
|:--------------|:-------------------------------------------------------------------------------|:-----------------:|
|`type`         |`peer`                                                                          |:heavy_check_mark: |
|`interface`    |See [Standard Protocols for Connectivity](#standard-protocols-for-connectivity) |:heavy_check_mark: |
|`link`         |See [Standard Protocols for Connectivity](#standard-protocols-for-connectivity) |:heavy_check_mark: |
|`pk`           |Curve25519 public key, encoded with `nacl.encoding.URLSafeBase64Encoder`        |:heavy_check_mark: |
|`metadata`     |JSON blob of metadata, encoded with `nacl.encoding.URLSafeBase64Encoder`        |                   |
|`cjdnsVersion` |integer                                                                         |:heavy_check_mark: |
|`wrbtVersion`  |integer                                                                         |:heavy_check_mark: |

### Credentials (Asynchronous)

```
scheme://host/?type=credentials&interface=udp&link=overlay&pk=wrbtPk&message=encodedMessage&cjdnsVersion=X&wrbtVersion=Y
```

|Key              |Value                                                                           |Required           |
|:----------------|:-------------------------------------------------------------------------------|:-----------------:|
|`type`           |`credentials`                                                                   |:heavy_check_mark: |
|`interface`      |See [Standard Protocols for Connectivity](#standard-protocols-for-connectivity) |:heavy_check_mark: |
|`link`           |See [Standard Protocols for Connectivity](#standard-protocols-for-connectivity) |:heavy_check_mark: |
|`pk`             |public key from peering request; omit when broadcasting                         |                   |
|`encodedMessage` |credentials encoded with `nacl.encoding.URLSafeBase64Encoder`                   |:heavy_check_mark: |
|`cjdnsVersion`   |integer                                                                         |:heavy_check_mark: |
|`wrbtVersion`    |integer                                                                         |:heavy_check_mark: |

## Optional Metadata

Metadata, a JSON blob of key-value-pairs, can optionally be provided in either a peering request or along with a credentials bundle. When Alice provides metadata in a peering request, the content is available as clear text, accessible to whoever sees her peering request. When metadata is included by Bob as part of the credentials bundle, it is encrypted with Alice's `wrbtPk`, and is intended to be read by Alice only.

The following is a standard set of metadata entries.

|Key                 |Value                                           |
|:-------------------|:-----------------------------------------------|
|`name`              |string                                          |
|`platform`          |`desktop` or `android`                          |
|`location`          |string formatted "latitudeFloat,longitudeFloat" |
|`isOverlayIpStatic` |boolean                                         |
