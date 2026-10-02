# NetworkCore Documentation

NetworkCore connects charge point operators (CPOs) with distribution partners (DPs) such as
mobility apps, fleets and car makers. CPOs connect once over OCPI 2.2.1. Partners integrate one
REST API to find chargers, start charging and follow every session live.

## Guides

| Guide | For |
| --- | --- |
| [CPO onboarding](cpo/onboarding.md) | Charge point operators connecting their network over OCPI 2.2.1 |
| [DP API reference](dp-api/reference.md) | Distribution partners building on the NetworkCore API |

## Endpoints

| Service | Base URL |
| --- | --- |
| OCPI (for CPOs) | `https://ocpi.networkcore.org` |
| DP API (for partners) | `https://dp.networkcore.org` |

## Contact

Questions about onboarding or an integration: contact your NetworkCore representative. When you
report a problem, include the `X-Correlation-ID` from the response.
