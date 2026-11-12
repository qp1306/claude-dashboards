# Claude Dashboards

Home Assistant dashboards for the house: **Home Claude**, **Home Claude Insights**, **Sigenergy Energy Centre** and **Climate** (AirTouch 5).

## Guides

- [Adding new Shelly devices](docs/adding-shelly-devices.md) ([PDF](docs/adding-shelly-devices.pdf)): the full chain from Shelly → Home Assistant → dashboards.

## Folders

| Folder | Contents |
|---|---|
| `docs/` | Guides |
| `dashboards/` | Raw configuration exports of each dashboard |
| `packages/` | Home Assistant packages (`/config/packages/`) |
| `shelly-scripts/` | Scripts that run on the Shelly devices |

> Never commit `secrets.yaml`, tokens or device passwords.
