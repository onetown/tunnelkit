![iOS 11+](https://img.shields.io/badge/ios-11+-green.svg)
![macOS 10.15+](https://img.shields.io/badge/macos-10.15+-green.svg)
[![License GPLv3](https://img.shields.io/badge/license-GPLv3-lightgray.svg)](LICENSE)
[![GitHub Actions](https://github.com/passepartoutvpn/tunnelkit/actions/workflows/test.yml/badge.svg)](https://github.com/passepartoutvpn/tunnelkit/actions/workflows/test.yml)

# TunnelKit

This library provides a generic framework for VPN development and a simplified Swift/Obj-C implementation of the OpenVPN® protocol for the Apple platforms. The crypto layer is built on top of [OpenSSL 1.1.1][dep-openssl], which in turn enables support for a certain range of encryption and digest algorithms.

## Getting started

The client is known to work with [OpenVPN®][openvpn] 2.3+ servers.

- [x] Handshake and tunneling over UDP or TCP
- [x] Ciphers
    - AES-CBC (128/192/256 bit)
    - AES-GCM (128/192/256 bit, 2.4)
- [x] HMAC digests
    - SHA-1
    - SHA-2 (224/256/384/512 bit)
- [x] NCP (Negotiable Crypto Parameters, 2.4)
    - Server-side
- [x] TLS handshake
    - Server validation (CA, EKU)
    - Client certificate
- [x] TLS wrapping
    - Authentication (`--tls-auth`)
    - Encryption (`--tls-crypt`)
- [x] Compression framing
    - Via `--comp-lzo` (deprecated in 2.4)
    - Via `--compress`
- [x] Compression algorithms
    - LZO (via `--comp-lzo` or `--compress lzo`)
- [x] Key renegotiation
- [x] Replay protection (hardcoded window)

The library therefore supports compression framing, just not newer compression. Remember to match server-side compression and framing, otherwise the client will shut down with an error. E.g. if server has `comp-lzo no`, client must use `compressionFraming = .compLZO`.

### Support for .ovpn configuration

TunnelKit can parse .ovpn configuration files. Below are a few details worth mentioning.

#### Non-standard

- Single-byte XOR masking
    - Via `--scramble xormask <character>`
    - XOR all incoming and outgoing bytes by the ASCII value of the character argument
    - See [Tunnelblick website][about-tunnelblick-xor] for more details

#### Unsupported

- UDP fragmentation, i.e. `--fragment`
- Compression via `--compress` other than empty or `lzo`
- Connecting via proxy
- External file references (inline `<block>` only)
- Static key encryption (non-TLS)
- `<connection>` blocks
- `vpn_gateway` and `net_gateway` literals in routes

#### Ignored

- Some MTU overrides
    - `--link-mtu` and variants
    - `--mssfix`
- Multiple `--remote` with different `host` values (first wins)
- Static client-side routes

Many other flags are ignored too but it's normally not an issue.

## Installation

### Requirements

- iOS 11.0+ / macOS 10.15+
- SwiftPM 5.3
- Git (preinstalled with Xcode Command Line Tools)

It's highly recommended to use the Git package provided by [Homebrew][dep-brew].

### Caveats

Make sure to set "Enable Bitcode" (iOS) to NO, otherwise the library [would not be able to link OpenSSL][about-pr-bitcode].

Recent versions of Xcode (latest is 13.1) have an issue where the "Frameworks" directory is replicated inside application extensions. This is not a blocker during development, but will prevent your archive from being validated against App Store Connect due to the following error:

    ERROR ITMS-90206: "Invalid Bundle. The bundle at '*.appex' contains disallowed file 'Frameworks'."

You will need to add a "Run Script" phase to your main app target where you manually remove the offending folder, i.e.:

    rm -rf "${BUILT_PRODUCTS_DIR}/${PLUGINS_FOLDER_PATH}/YourTunnelTarget.appex/Frameworks"

for iOS and:

    rm -rf "${BUILT_PRODUCTS_DIR}/${PLUGINS_FOLDER_PATH}/YourTunnelTarget.appex/Contents/Frameworks"

for macOS.

### Demo

Download the library codebase locally:

    $ git clone https://github.com/passepartoutvpn/tunnelkit.git

There are demo targets containing a simple app for testing the tunnel, called `BasicTunnel`. Open `Demo/TunnelKit.xcodeproject` in Xcode and run it on both iOS and macOS.

For the VPN to work properly, the `BasicTunnel` demo requires:

- _App Groups_ and _Keychain Sharing_ capabilities
- App IDs with _Packet Tunnel_ entitlements

both in the main app and the tunnel extension target.

In order to test connectivity in your own environment, modify the file `Demo/Demo/Configuration.swift` to match your VPN server parameters.

Example:

    private let ca = CryptoContainer(pem: """
	-----BEGIN CERTIFICATE-----
	MIIFJDCC...
	-----END CERTIFICATE-----
    """)

Make sure to also update the following constants in the same files, according to your developer account and your target bundle identifiers:

    public static let appGroup
    public static let tunnelIdentifier

Remember that the App Group on macOS requires a team ID prefix.

## Documentation

The library is split into several modules, in order to decouple the low-level protocol implementation from the platform-specific bridging, namely the [NetworkExtension][ne-home] VPN framework.

Full documentation of the public interface is available and can be generated by opening the package in Xcode and running "Build Documentation" (Xcode 13).

### TunnelKitCore

Contains the building blocks of a VPN protocol. Eventually, a consumer would implement the `Session` interface, expected to start and control the VPN session. A session is expected to work with generic network interfaces:

- `LinkInterface` (e.g. a socket)
- `TunnelInterface` (e.g. an `utun` interface)

There are no physical network implementations (e.g. UDP or TCP) in this module.

### TunnelKitManager

This component includes convenient classes to control the VPN tunnel from your app without the NetworkExtension headaches. Have a look at `VPNProvider` implementations:

- `MockVPNProvider` (default, useful to test on simulator)
- `NetworkExtensionVPNProvider` (anything based on NetworkExtension)

### TunnelKitAppExtension

Provides a layer on top of the NetworkExtension framework. Most importantly, bridges native [NWUDPSession][ne-udp] and [NWTCPConnection][ne-tcp] to an abstract `GenericSocket` interface, thus making a multi-protocol VPN dramatically easier to manage.

### TunnelKitIKE

Here you find `NativeProvider`, a generic way to manage a VPN profile based on the native IPSec/IKEv2 protocols. Just wrap a `NEVPNProtocolIPSec` or `NEVPNProtocolIKEv2` object in a `NetworkExtensionVPNConfiguration` and use it to install or connect to the VPN.

### TunnelKitOpenVPN*

#### Protocol

Here are the low-level entities on top of which an OpenVPN connection is established. Code is mixed Swift and Obj-C, most of it is not exposed to consumers. The protocol implementation in particular depends on OpenSSL.

The entry point is the `OpenVPNSession` class. The networking layer is fully abstract and delegated externally with the use of opaque `IOInterface` (`LinkInterface` and `TunnelInterface`) and `OpenVPNSessionDelegate` protocols.

#### AppExtension

Another goal of this area is packaging up a black box implementation of a [NEPacketTunnelProvider][ne-ptp], which is the essential part of a Packet Tunnel Provider app extension. You will find the main implementation in the `OpenVPNTunnelProvider` class.

A debug log snapshot is optionally maintained and shared by the tunnel provider to host apps via the App Group container.

#### Manager

On the client side, you manage the VPN profile with the `OpenVPNProvider` class, which is a specific implementation of `NetworkExtensionVPNProvider`.

### TunnelKitLZO

Due to the restrictive license (GPLv2), LZO support is provided as an optional component.

## License

Copyright (c) 2021 Davide De Rosa. All rights reserved.

### Part I

This project is licensed under the [GPLv3][license-content].

### Part II

As seen in [libsignal-protocol-c][license-signal]:

> Additional Permissions For Submission to Apple App Store: Provided that you are otherwise in compliance with the GPLv3 for each covered work you convey (including without limitation making the Corresponding Source available in compliance with Section 6 of the GPLv3), the Author also grants you the additional permission to convey through the Apple App Store non-source executable versions of the Program as incorporated into each applicable covered work as Executable Versions only under the Mozilla Public License version 2.0 (https://www.mozilla.org/en-US/MPL/2.0/).

### Part III

Part I and II do not apply to the LZO library, which remains licensed under the terms of the GPLv2+.

### Contributing

By contributing to this project you are agreeing to the terms stated in the [Contributor License Agreement (CLA)][contrib-cla].

For more details please see [CONTRIBUTING][contrib-readme].

## Credits

- [lzo][dep-lzo-website] - Copyright (c) 1996-2017 Markus F.X.J. Oberhumer
- [PIATunnel][dep-piatunnel-repo] - Copyright (c) 2018-Present Private Internet Access
- [SURFnet][ppl-surfnet]
- [SwiftyBeaver][dep-swiftybeaver-repo] - Copyright (c) 2015 Sebastian Kreutzberger
- [XMB5][ppl-xmb5] for the [XOR patch][ppl-xmb5-xor] - Copyright (c) 2020 Sam Foxman

This product includes software developed by the OpenSSL Project for use in the OpenSSL Toolkit. ([https://www.openssl.org/][dep-openssl])

Copyright (c) 2002-2018 OpenVPN Inc. - OpenVPN is a registered trademark of OpenVPN Inc.

## Contacts

Twitter: [@keeshux][about-twitter]

Website: [passepartoutvpn.app][about-website]

[openvpn]: https://openvpn.net/index.php/open-source/overview.html

[dep-brew]: https://brew.sh/
[dep-openssl]: https://www.openssl.org/

[ne-home]: https://developer.apple.com/documentation/networkextension
[ne-ptp]: https://developer.apple.com/documentation/networkextension/nepackettunnelprovider
[ne-udp]: https://developer.apple.com/documentation/networkextension/nwudpsession
[ne-tcp]: https://developer.apple.com/documentation/networkextension/nwtcpconnection

[license-content]: LICENSE
[license-signal]: https://github.com/signalapp/libsignal-protocol-c#license
[license-mit]: https://choosealicense.com/licenses/mit/
[contrib-cla]: CLA.rst
[contrib-readme]: CONTRIBUTING.md

[dep-piatunnel-repo]: https://github.com/pia-foss/tunnel-apple
[dep-swiftybeaver-repo]: https://github.com/SwiftyBeaver/SwiftyBeaver
[dep-lzo-website]: http://www.oberhumer.com/opensource/lzo/
[ppl-surfnet]: https://www.surf.nl/en/about-surf/subsidiaries/surfnet
[ppl-xmb5]: https://github.com/XMB5
[ppl-xmb5-xor]: https://github.com/passepartoutvpn/tunnelkit/pull/170
[about-tunnelblick-xor]: https://tunnelblick.net/cOpenvpn_xorpatch.html
[about-pr-bitcode]: https://github.com/passepartoutvpn/tunnelkit/issues/51

[about-twitter]: https://twitter.com/keeshux
[about-website]: https://passepartoutvpn.app
