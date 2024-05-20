# Lightning Service Provider Spec
API specifications for RGB Lightning Service Providers (LSP)

The goal of this repository is to provide a unified API specification for RGB Lightning Service Providers (LSP) to create interoperability between different RGB Lightning Network wallets/clients and RGB LSPs.

## Status Specification

All LSPS specifications include a "Status" field.
"Status" can be one of the following:

* "Draft" - The specification is still under active development and
  subject to change. Implementation is not recommended at this
  time.
* "For Implementation" - The specification has been widely reviewed by
  LSPS participants, is believed to have addressed all raised
  issues, and LSPS participants recommend this specification to be
  implemented.
* "Stable" - The specification has been implemented by at least one
  client and one LSP, which are developed by at least two different
  organizations or open-source project teams, and both development
  teams have reported interoperability without further modifications
  or clarifications of the specification.

## Specs

### **LSPS0** [Transport Layer](LSPS0/README.md)
Describes the basics of how clients and RGB LSPs communicate to each other.

### **LSPS1** [Channel Request](LSPS1/README.md)
A channel purchase API to buy RGB denominated channels channels from an LSP.

## Services
List of RGB Lightning Service Providers in alphabetic order that currently or will support LSP specs in future.

(If you would like to get added to this list, please open a PR and update the table below)

| Name         | Specs       | Status |
| ------------ | ----------- | ------ |
| Kaleidoscope    | -           | -      |
| Thunderstack        | -           | -      |

## Responsible Disclosure
TBD
