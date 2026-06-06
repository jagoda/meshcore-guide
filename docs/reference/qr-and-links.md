# QR Codes and Links

MeshCore uses a `meshcore://` URL scheme to share node contacts and channel keys without requiring the recipient to be in RF range. These URLs can be shared as QR codes, clipboard text, or clickable links in messaging apps.

For the canonical parameter definitions, see the [QR Codes spec](https://docs.meshcore.io/qr_codes/) on `docs.meshcore.io`.

---

## Contact sharing

Add a node to someone's contact list — name, public key, and node type — packaged as a URL.

### URL format

```
meshcore://contact/add?name=<name>&public_key=<hex>&type=<type>
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `name` | Yes | Human-readable contact name. URL-encode spaces and special characters (e.g., `+` for space). |
| `public_key` | Yes | The 32-byte Ed25519 public key as 64 lowercase hex characters. |
| `type` | Yes | Node role as a numeric code (see table below). |

### Node type codes

| Code | Role |
|------|------|
| `1` | Companion Radio (Chat) |
| `2` | Repeater |
| `3` | Room Server |
| `4` | Sensor |

### Example

```
meshcore://contact/add?name=Alice&public_key=9cd8fcf22a47333b591d96a2b848b73f457b1bb1a3ea2453a885f9e5787765b1&type=1
```

Scanning or clicking this URL adds *Alice* (a Companion Radio, public key `9cd8…5b1`) to the contact list. The companion app can then send her a direct message immediately, even before receiving one of her adverts.

### Where contact QR codes come from

- The companion app generates a QR code for your own identity in the share/profile screen.
- Repeater admin tools can export a QR code for the repeater's public key.
- Community maps (e.g., the MeshCore public map) may display node QR codes.

---

## Channel sharing

Add a shared group channel — name and secret — to a device's channel list.

### URL format

```
meshcore://channel/add?name=<name>&secret=<hex>
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `name` | Yes | Channel display name. URL-encode if needed. |
| `secret` | Yes | The 16-byte AES channel secret as 32 lowercase hex characters. |

### Example — default public channel

```
meshcore://channel/add?name=Public&secret=8b3387e9c5cdea6ac9e5edbaa115cd72
```

The hex value `8b3387e9c5cdea6ac9e5edbaa115cd72` is the well-known default public channel key. Any device that adds this channel joins the global community channel. Treat it as a public square — assume no privacy.

!!! note "T-Deck Base64 representation"
    Some T-Deck firmware displays the channel secret in Base64:
    `izOH6cXN6mrJ5e26oRXNcg==`
    This is the same 16-byte key. QR codes always use the hex form.

### Creating a private channel QR

Generate a random 16-byte key (your OS `urandom`, a password manager, or a cryptographic key generator), encode it as 32 hex characters, and substitute it into the URL. Share the URL or QR code only with intended members.

---

## How apps handle these URLs

- **Android / iOS companion app:** scan QR from within the app (Contacts → scan or Channels → scan), or share the URL directly via the system share sheet.
- **T-Deck:** write the `meshcore://channel/add?…` URL to `clipboard.txt` on the SD card, then paste from clipboard in the channels screen.
- **Custom clients:** parse the URL scheme, extract parameters, and call the appropriate companion API command to add the contact or channel.

---

## See also

- [Adverts and Contacts](../concepts/adverts-and-contacts.md) — how contacts are normally added via advert reception.
- [Channels vs. Direct](../concepts/channels-vs-direct.md) — the channel model and shared-secret design.
- [QR Codes spec](https://docs.meshcore.io/qr_codes/) — canonical parameter definitions on `docs.meshcore.io`.
