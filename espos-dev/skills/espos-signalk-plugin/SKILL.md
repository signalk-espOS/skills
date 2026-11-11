---
name: espos-signalk-plugin
description: "Building a Signal K plugin or webapp for signalk-espOS (manager, HMI designer, display stream, signalk-zigbee, or your own) so a boat owner installs it and it works: enabling it by default, defaults that need no configuration, finding devices instead of asking for addresses, and webapps and plugin config forms that work on a touchscreen at the helm."
---

# Signal K plugins a boat owner never has to configure

Verified against signalk-server 2.36.0 (Admin UI on `@rjsf/core` 5) and
signalk-espos-manager 0.8.1, November 2026. Sources: signalk-server
`src/interfaces/plugins.ts`, `docs/develop/plugins/configuration.md`,
`docs/develop/webapps.md`; manager `package.json`, `src/config.ts`,
`web/src/style.css`.

The person installing a signalk-espOS plugin is a boat owner, often on a phone or
the chart plotter's touchscreen. Every field they have to fill in is a step where
they give up. Aim for: install from the App Store, restart, and it works.

## Enable it by default

Add this to `package.json`, next to the `signalk` block:

```json
"signalk-plugin-enabled-by-default": true
```

What the server does with it (`plugins.ts`, `isEnabledByPackageEnableDefault`):

* It only applies while the plugin has **no saved `enabled`**, so a fresh install. A
  user who disables the plugin stays disabled.
* The server writes `{ enabled: true, configuration: {} }` to the plugin's options
  file and starts the plugin with **an empty configuration**. The schema's `default`s
  are not applied (see below), so a plugin that reads `options.port` gets `undefined`.
* It takes effect when the server starts after the install. The App Store asks for
  that restart anyway.
* Adding the flag to a plugin already in use: an options file that has a
  `configuration` but no `enabled` key gets that configuration replaced by `{}`. The
  Admin UI always saves `enabled`, so this hits only hand-written options files; say
  so in the release notes.

Turn it on whenever `start({})` does something useful and harmless. The manager and
the HMI designer do. Leave it off, and say why in the README, only when starting
with nothing configured would cost the boat something: pulling a container image
of hundreds of MB, opening a serial port another program owns, or sending anything
onto a bus. Even then, prefer making `start({})` cheap (do nothing until there is
something to do) and enabling it.

## Defaults that need no configuration

* **Merge defaults in `start()`.** `start()` receives `{}` until the user saves the
  form, and the Admin UI shows schema `default`s that the plugin never received. Keep
  one `applyDefaults(raw)` that turns whatever arrived into complete settings, and
  build the schema's `default`s from the same constants so the two cannot drift. The
  manager (`defaultSettings()` / `applyDefaults()` in `src/config.ts`) and
  signalk-zigbee (`applyDefaults` in `src/config.ts`) do this.
* **Tolerate old and partial configs.** Read each field with a typed fallback
  (`int(raw.port, DEFAULT_PORT)`), drop entries that are missing their key field, and
  never throw from `start()`: Signal K neither awaits nor catches it.
* **Pick the value that works on most boats**, not the most general one. Ports that
  do not clash with signalk-server (3000, 3443) or common add-ons, the self vessel,
  SI units, an update interval a Raspberry Pi can afford.
* **Find things instead of asking for them.** espOS devices announce themselves over
  mDNS (`_espos._tcp`, `_signalk-player._tcp`); browse for them and list what you
  found. A Signal K plugin already runs inside the server, so never ask for the server
  URL or a token. An address field is the fallback for a network where mDNS does not
  pass, not the first thing the user sees.
* **Never check an address by connecting to a device's data port.** The Zigbee
  coordinator's 6638 and the stream capture server take one client, so a probe takes
  the slot from Zigbee2MQTT or the panel. Use the mDNS record, or an HTTP health
  endpoint where the device has one.
* **Things that change a device stay opt-in.** A default may read, discover and show.
  Writing to a device, giving it a key or flashing it needs the user's click, or a
  setting that is off by default and says what it does (the manager's
  `auth.autoProvision`).
* **Show state, not just settings.** Put "found 2 devices", "waiting for the
  coordinator" or the last error in `app.setPluginStatus()` / `setPluginError()` and
  in the webapp, so the user sees it works without opening a log.

## Touchscreen first

Signal K webapps open on phones, tablets and the plotter. Fingers are wet, the boat
moves, the sun is on the screen.

* **Targets at least 44 × 44 CSS px**, with space between them. The manager sets
  `min-height: 44px` on every button and text input and gives icon-only links their
  own 44 px box (`web/src/style.css`). Never make a row the target and an icon inside
  it a second target.
* **No hover-only UI.** Anything shown on `:hover` (tooltips, row actions, menus) is
  unreachable by touch. Show it, or open it on tap.
* **No drag, long-press or double-tap as the only way** to do something. Offer a
  button too. Pinch zoom stays on: keep `<meta name="viewport"
  content="width=device-width, initial-scale=1">` and never add `user-scalable=no`
  or `maximum-scale=1`.
* **Inputs that fit the keyboard**: `type="number"` for values, plus
  `inputmode="decimal"` only where a value cannot be negative (the iOS decimal pad has
  no minus key). An address field takes a bare host, IP or `host:port`, which
  `type="url"` refuses, so use `type="text"` with `inputmode="url"`. Add
  `autocapitalize="off"` and `spellcheck="false"` for identifiers. A choice of a few options is a set of buttons or a select, not a
  text field.
* **Fonts at 16 px or more** in inputs (smaller makes iOS zoom in on focus), and high
  contrast. Follow `prefers-color-scheme`, but that is the device's setting, not
  the time of day. Always offer a dark theme the user can pick. Following Signal
  K's `environment.mode` (night), which signalk-espos-manager publishes, comes on
  top: the manager publishes no mode until the server has a position, so the
  user's choice is the fallback.
* **Works at 360 px wide** with no horizontal scrolling; tables become lists on a
  narrow screen.
* **Confirm destructive actions in the page**, not with `alert()` or `confirm()`,
  which some kiosk browsers block and which are small on a plotter. The manager
  (fleet-key removal) and the HMI designer still call `window.confirm`; don't copy
  that.

### The plugin config form

The Admin UI renders the plugin's `schema` with react-jsonschema-form 5 and the
plugin's `uiSchema` (`PluginConfigurationForm.tsx`), with rjsf's stock fields and
widgets only. The form itself is the server's, so a plugin can only steer it:

* **Fewer fields.** Everything with a working default stays out of the way. Put
  rarely changed settings in an object titled "Advanced" and move it to the end with
  `'ui:order'`. The `'ui:field': 'collapsible'` example in the server's
  `configuration.md` comes from react-jsonschema-form-extras, which the current Admin
  UI does not load, so it renders nothing collapsible.
* **Choices, not text.** A `oneOf` of `{ const, title }` entries (a select), or
  `'ui:widget': 'radio'` for two or three options, instead of a string the user types.
  Booleans are checkboxes; give them a `title` that reads as a sentence.
* **Explain in the form**: `description` on the property, `'ui:help'` for one line
  below it, `'ui:placeholder'` for an example value. `'ui:order'` puts the field the
  user most likely changes first.

When the form is not enough (lists of devices, a key to generate, a status to
show), ship a `./PluginConfigurationPanel` instead: a React component the Admin UI
loads through Module Federation in place of the generated form, declared with the
`signalk-plugin-configurator` keyword (`docs/develop/webapps.md`; signalk-espos-stream
builds one with webpack from `src/configpanel/`). The touch rules above apply to it
in full. It receives `configuration` and `save` as props; saving through `save`
keeps the stored configuration where `start()` reads it.

## Before you ship

Install the packed plugin (`npm pack`, then `npm install --ignore-scripts <tgz>` in
the server's config directory), restart, and check without touching any setting:

1. The plugin is enabled and running, and its status line says what it found.
2. The webapp and the config page work on a phone: every control reachable by
   thumb, no horizontal scrolling, nothing that needs hover.
3. Saving the config form once without changes leaves the plugin working the same way.

## Related

`espos-ecosystem` (the plugin repos and their house rules), `espos-keys-ota-fleet`
(what the manager does for a boat).
