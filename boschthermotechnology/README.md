# Bosch Thermotechnology Binding

This binding integrates Bosch and Buderus heating systems that are controlled through the MyBuderus or Bosch DashApp mobile app.
Those apps talk to the Bosch Thermotechnology **PointT** cloud API, protected by a **SingleKey ID** login.
This binding uses the same cloud API, so it only works with gateways that are already set up in one of those apps — it does not talk to the gateway directly on the local network.
If your gateway is an older KM50/KM100/KM200 model reachable on your local network, use the `km200` binding instead.

Each gateway can bundle more than one physical subsystem — a heat pump, a photovoltaic/solar-thermal installation, a pool heater, an air conditioner, and so on.
The binding represents each of these as its own Thing underneath the gateway, instead of exposing every possible channel on one large Thing.
See [Thing Hierarchy](#thing-hierarchy) below for how these Things relate to each other.

## Thing Hierarchy

Every setup has exactly one `account` Bridge and one `gateway` Bridge per physical gateway.
Everything functional — heat pump, PV, pool, and so on — is a separate Thing underneath the `gateway` Bridge, as siblings of each other, never nested inside one another:

```text
account (Bridge)
└── gateway (Bridge)
    ├── heatpump
    ├── pv
    ├── pool
    ├── ventilation-zone
    ├── zone-thermostat (one per RF room thermostat)
    ├── energy-monitoring
    ├── ac-unit
    └── water-softener
```

Not every gateway has all eight child things — which ones apply depends on the hardware actually installed, and discovery only proposes the ones it can detect (see [Discovery](#discovery)).

## Supported Things

| Thing Type | Thing ID | Description |
|------------|----------|-------------|
| Bridge | `account` | One SingleKey ID (Bosch/Buderus) account login. Required as the parent of one or more gateways. |
| Bridge | `gateway` | A single heating gateway registered to the account, e.g. a Buderus Logamatic control unit. Holds only channels that belong to the physical box itself; everything functional is a child thing underneath it. |
| Thing | `heatpump` | Heating circuits, domestic hot water circuits, and heat source(s) of one gateway. |
| Thing | `pv` | Photovoltaic inverter and solar-thermal collector readings of one gateway. |
| Thing | `pool` | Pool heating status of one gateway. |
| Thing | `ventilation-zone` | One HRV ventilation zone of a gateway. |
| Thing | `zone-thermostat` | One multi-zone RF room thermostat. A gateway can have more than one. |
| Thing | `energy-monitoring` | Historical energy-monitoring recordings of one gateway's heat source(s). |
| Thing | `ac-unit` | One room air-conditioning (RAC) unit. |
| Thing | `water-softener` | One water softener. Not yet functional — the PointT resource path for this device class has not been confirmed against a live gateway, see the note under [`water-softener` Thing Configuration](#water-softener-thing-configuration). |

## Discovery

Once an `account` Bridge is authorized (see [Account Authorization](#account-authorization) below), it automatically discovers every gateway registered to that account through the PointT API and lists it in the Inbox.
Once a `gateway` Bridge in turn comes online, it probes the PointT API for the child things that gateway actually has (heat pump circuits, PV/solar-thermal, pool, ventilation, RF zone thermostats, AC units) and lists whichever ones it finds in the Inbox.
`water-softener` is never proposed by discovery, since its resource path is unconfirmed — see [Supported Things](#supported-things).

Adding any Thing manually is also possible, but its `gatewayId` (and, for `zone-thermostat`, `zoneId`) configuration parameter must then be filled in by hand.

## Account Authorization

SingleKey ID uses an OAuth2 login flow that ends with a redirect back into the (non-existent) MyBuderus mobile app (`com.buderus.tt.dashtt://app/login?code=...`), so openHAB cannot capture the login result automatically.
Browsers cannot open that address and show it only in the developer console, which makes copying it by hand awkward.
The binding therefore ships small helper scripts (in its `docs` folder) that run the browser login on your own computer, catch the redirect, and print the one value you need.
This is required once per account:

1. Add an `account` Bridge. A few seconds later, its `authUrl` configuration parameter contains the SingleKey ID login URL, and the Bridge shows `OFFLINE` ("Not yet authorized").
2. Copy the value of the `authUrl` parameter.
3. On a desktop computer that has a browser, run the helper script for your operating system from the binding's `docs` folder (the full source of every script is also listed in [Helper Script Sources](#helper-script-sources) at the end of this document). The script asks for the URL; paste the copied `authUrl` when prompted.

   | System | Command |
   |--------|---------|
   | Windows (PowerShell or `cmd.exe`) | [`get-auth-code-windows.ps1`](#windows-powershell): `powershell -ExecutionPolicy Bypass -File .\get-auth-code-windows.ps1` |
   | Linux | [`get-auth-code-linux.sh`](#linux): `./get-auth-code-linux.sh` (needs `xdg-open` and `xdg-mime`) |
   | macOS | [`get-auth-code-macos.sh`](#macos): `./get-auth-code-macos.sh` (needs `osacompile`, which ships with macOS) |

   The `-ExecutionPolicy Bypass` option is only needed because PowerShell blocks unsigned script files by default; it applies to this one call only. The same command works in the regular Windows console (`cmd.exe`).

4. Log in with your Bosch/Buderus account in the browser that opens. If the browser asks whether it may open an application, confirm.
5. The script prints the authorization code and copies it to the clipboard. Paste it into the Bridge's `pasteAuthorizationRedirectUrl` configuration parameter and save.

The Bridge goes `ONLINE` once the login succeeds, and stays authorized afterward — the login step is not required again unless the Bridge is removed and re-added.

The scripts need a graphical session with a browser, so they do not work on a headless server (for example over SSH). They do not have to run on the openHAB server, though: any desktop computer with a browser will do, because it only needs the `authUrl` and the resulting code is simply pasted into the Bridge configuration. There is no network connection between that computer and openHAB.

While it runs, the script registers a handler for the `com.buderus.tt.dashtt://` address for the current user only (no administrator rights needed), waits up to five minutes for the redirect, and removes the handler again afterward — also if you abort it.
The code is valid only briefly and only once, so paste it right away. The `authUrl` is tied to the running Bridge: if openHAB restarts before you have finished, copy the newly generated `authUrl` and run the script again.

If you prefer not to run a script, open the `authUrl` in a browser and log in, then copy the full `com.buderus.tt.dashtt://app/login?code=...` address from the browser's developer tools (console message "Failed to launch ...", or the `Location` header in the network tab) into `pasteAuthorizationRedirectUrl`.

## Thing Configuration

### `account` Bridge Configuration

| Name | Type | Description | Default | Required | Advanced |
|------|------|--------------|---------|----------|----------|
| `authUrl` | text | Automatically filled in with the SingleKey ID login URL. Read-only, empty once authorized. | N/A | no | no |
| `pasteAuthorizationRedirectUrl` | text | One-time: paste the code printed by the helper script here to complete login (see [Account Authorization](#account-authorization)). Never stored. | N/A | no | no |

### `gateway` Thing Configuration

| Name | Type | Description | Default | Required | Advanced |
|------|------|--------------|---------|----------|----------|
| `gatewayId` | text | The gateway id assigned by the PointT API. Filled in automatically by discovery. | N/A | yes | yes |
| `refreshInterval` | integer | Interval the gateway's own system resources are polled, in seconds. | 60 | no | yes |

### `heatpump`, `pv`, `pool`, `ventilation-zone`, `energy-monitoring`, `ac-unit`, `water-softener` Thing Configuration

These seven child thing types share the same two configuration parameters:

| Name | Type | Description | Default | Required | Advanced |
|------|------|--------------|---------|----------|----------|
| `gatewayId` | text | The gateway id this thing belongs to. Filled in automatically by discovery. | N/A | yes | yes |
| `refreshInterval` | integer | Interval this thing's resources are polled, in seconds. | 60 | no | yes |

#### `water-softener` Thing Configuration

The PointT resource path prefix for water softeners has not been confirmed against a live gateway — only the device-type string `watersoftener` is known from reverse-engineering the mobile app.
A `water-softener` Thing therefore always goes `OFFLINE`/`CONFIGURATION_PENDING` once its `gatewayId` is set, and exposes no channels yet.

### `zone-thermostat` Thing Configuration

| Name | Type | Description | Default | Required | Advanced |
|------|------|--------------|---------|----------|----------|
| `gatewayId` | text | The gateway id this zone thermostat belongs to. Filled in automatically by discovery. | N/A | yes | yes |
| `zoneId` | text | The zone number (e.g. `1` for `zone1`) as returned by the gateway. Filled in automatically by discovery. | N/A | yes | no |
| `refreshInterval` | integer | Interval this thing's resources are polled, in seconds. | 60 | no | yes |

## Channels

### `gateway` Channels

The `gateway` Thing groups its channels by function.
Use the group id together with the channel id (`<group>#<channel>`) when linking Items.

| Channel Group | Channel ID | Type | Read/Write | Description |
|---------------|-----------|------|------------|--------------|
| `system` | `outdoor-temperature` | Number:Temperature | R | Outdoor temperature reported by the gateway. |
| `system` | `away-mode-enabled` | Switch | RW | Enables or disables away mode for the whole gateway. |
| `system` | `silent-mode-enabled` | Switch | RW | Enables or disables silent (noise-reduced) mode for the whole gateway. |
| `system` | `season-optimizer-mode` | Number | RW | Season optimizer mode: 0 = Off, 1 = Automatic, 2 = Forced heating (wire values `off`/`automatic`/`forcedHeat`, confirmed live). |
| `system` | `holiday-mode-active` | Switch | R | ON while at least one holiday mode is active (list shape assumed, not yet confirmed live). |
| `notifications` | `active` | Number | R | Number of active notifications/faults (list shape assumed, not yet confirmed live). |

In addition to the channels above, the `gateway` Thing exposes `serialId`, `firmwareVersion`, and `hardwareVersion` as read-only Thing properties (visible in the Properties panel).
These are refreshed once a day on their own schedule, independent of `refreshInterval` and the regular channel poll, since this metadata rarely changes.

### `heatpump` Channels

Unlike every other child thing, `heatpump` has no fixed channel list — its channel groups are built dynamically, one `heating-circuit-<circuitId>` group per heating circuit and one `dhw-circuit-<circuitId>` group per domestic hot water circuit the gateway actually reports (e.g. `heating-circuit-hc1`, `dhw-circuit-dhw1`), plus one fixed `heat-source-1` group.
The circuit id in the group name comes from the gateway itself, not from list position, so it stays stable even if the gateway ever reports its circuits in a different order.

| Channel Group | Channel ID | Type | Read/Write | Description |
|---------------|-----------|------|------------|--------------|
| `heating-circuit-<circuitId>` | `manual-room-setpoint` | Number:Temperature | RW | Manual room setpoint of this heating circuit (5–30 °C). |
| `heating-circuit-<circuitId>` | `boost-mode` | Switch | RW | Heating boost of this heating circuit (the vendor app's "Boost" function); wire values `on`/`off`. |
| `heating-circuit-<circuitId>` | `boost-duration` | Number | RW | Boost duration, raw value in the unit reported by the gateway (not yet confirmed live). |
| `heating-circuit-<circuitId>` | `boost-temperature` | Number:Temperature | RW | Room temperature to be reached during the boost (0.5 °C steps). |
| `heating-circuit-<circuitId>` | `boost-remaining-time` | Number | R | Remaining time of the running boost, raw value in the unit reported by the gateway. |
| `dhw-circuit-<circuitId>` | `charge-duration` | Number:Time | RW | Domestic hot water single-charge duration of this DHW circuit (15–2880 min). |
| `dhw-circuit-<circuitId>` | `single-charge-setpoint` | Number:Temperature | RW | Domestic hot water single-charge temperature setpoint of this DHW circuit (50–70 °C). |
| `dhw-circuit-<circuitId>` | `operation-mode` | Number | RW | Domestic hot water operation mode of this DHW circuit: `0`=Off, `1`=Low, `2`=High, `3`=Own program, `4`=Eco (wire values `Off`/`low`/`high`/`ownprogram`/`eco`, confirmed live). |
| `dhw-circuit-<circuitId>` | `charge` | Switch | RW | Starts or stops a domestic hot water instant charge of this DHW circuit. |
| `dhw-circuit-<circuitId>` | `reduce-temp-on-alarm` | Switch | RW | Reduces the domestic hot water temperature while an alarm is active. |
| `heat-source-1` | `ch-status` | Switch | R | ON while central heating is active, OFF if the heat source reports `off`. |
| `heat-source-1` | `actual-supply-temperature` | Number:Temperature | R | Actual supply (flow) temperature of the heat source. |
| `heat-source-1` | `return-temperature` | Number:Temperature | R | Return temperature of the heat source. |
| `heat-source-1` | `number-of-starts` | Number | R | Number of times the heat source has started. |
| `heat-source-1` | `working-time-total-system` | Number:Time | R | Total working time of the heat source. Unit assumed to be hours, not yet confirmed. |

Current domestic hot water temperature and heating circuit room temperature are still not available in this binding — their resource paths were not confirmed against a real gateway response yet.
Cascade installations with more than one heat source are not yet supported — `heat-source-1` is always the only heat-source group, regardless of how many heat sources the gateway actually has.

### `pv` Channels

| Channel Group | Channel ID | Type | Read/Write | Description |
|---------------|-----------|------|------------|--------------|
| `photovoltaic` | `enabled` | Switch | RW | Enables or disables photovoltaic integration. |
| `photovoltaic` | `inverter-info` | String | R | Raw photovoltaic inverter information, as a JSON string. Exact response shape not yet confirmed. |
| `solar-thermal` | `collector-temperature` | Number:Temperature | R | Solar-thermal collector temperature. |
| `solar-thermal` | `yield` | Number:Energy | R | Solar-thermal yield. |

### `pool` Channels

`pool` channels are not grouped — link them directly by channel id.

| Channel ID | Type | Read/Write | Description |
|-----------|------|------------|--------------|
| `enabled` | Switch | RW | Enables or disables pool heating. |
| `setpoint-temperature` | Number:Temperature | RW | Pool water temperature setpoint. |
| `current-temperature` | Number:Temperature | R | Current pool water temperature. |

### `ventilation-zone` Channels

| Channel ID | Type | Read/Write | Description |
|-----------|------|------------|--------------|
| `operation-mode` | Number | RW | Ventilation mode: 0 = Off, 1 = Auto, 2 = Demand, 3 = Minimum, 4 = Reduced, 5 = Nominal, 6 = Maximum, 7 = Party, 8 = Sleep, 9 = Intensive, 10 = Bypass, 11 = Fireplace, 12 = Free Function. |
| `filter-remaining-time` | Number:Time | R | Remaining filter run time before replacement is due. |

### `zone-thermostat` Channels

| Channel ID | Type | Read/Write | Description |
|-----------|------|------------|--------------|
| `manual-room-setpoint` | Number:Temperature | RW | Manual room setpoint of this zone thermostat (5–30 °C). Assumes heating mode; cooling mode is not yet supported. |
| `average-current-temperature` | Number:Temperature | R | Average current room temperature of this zone thermostat. |
| `child-lock` | Switch | RW | Enables or disables the child lock of this zone thermostat. |

### `energy-monitoring` Channels

| Channel ID | Type | Read/Write | Description |
|-----------|------|------------|--------------|
| `actual-ch-power` | Number:Power | R | Actual central heating power. May need rework once it is confirmed whether this resource returns a single value or a time series. |
| `actual-dhw-power` | Number:Power | R | Actual domestic hot water power. Same caveat as `actual-ch-power`. |
| `total-consumed-energy` | Number:Energy | R | Total consumed energy of the heat source. Same caveat as `actual-ch-power`. |

### `ac-unit` Channels

| Channel ID | Type | Read/Write | Description |
|-----------|------|------------|--------------|
| `operation-mode` | Number | RW | 0 = Auto, 1 = Heat, 2 = Cool, 3 = Dry, 4 = Fan, 5 = Off (values not yet confirmed for PointT). |
| `fan-speed` | Number | RW | 0 = Auto, 1 = Quiet, 2 = Low, 3 = Medium, 4 = High, 5 = Turbo (values not yet confirmed for PointT). |
| `temperature-setpoint` | Number:Temperature | RW | RAC temperature setpoint. |

### `water-softener` Channels

None yet — see [`water-softener` Thing Configuration](#water-softener-thing-configuration).

## Full Example

### Thing Configuration

```java
Bridge boschthermotechnology:account:myaccount "Bosch Thermotechnology Account" [ ] {
    Bridge gateway mygateway "Buderus Gateway" [ gatewayId="123456789", refreshInterval=60 ] {
        Thing heatpump myheatpump "Heat Pump" [ gatewayId="123456789", refreshInterval=60 ]
        Thing pv mypv "Photovoltaic" [ gatewayId="123456789", refreshInterval=60 ]
        Thing pool mypool "Pool" [ gatewayId="123456789", refreshInterval=60 ]
        Thing zone-thermostat myzone1 "Living Room Thermostat" [ gatewayId="123456789", zoneId="1", refreshInterval=60 ]
        Thing energy-monitoring myenergy "Energy Monitoring" [ gatewayId="123456789", refreshInterval=60 ]
    }
}
```

### Item Configuration

```java
Number:Temperature   OutdoorTemperature      "Outdoor Temperature [%.1f %unit%]"       { channel="boschthermotechnology:gateway:myaccount:mygateway:system#outdoor-temperature" }
Switch                AwayModeEnabled         "Away Mode"                                { channel="boschthermotechnology:gateway:myaccount:mygateway:system#away-mode-enabled" }

Number:Temperature   ManualRoomSetpoint      "Manual Room Setpoint [%.1f %unit%]"      { channel="boschthermotechnology:heatpump:myaccount:mygateway:myheatpump:heating-circuit-hc1#manual-room-setpoint" }
Number:Time          DhwChargeDuration       "DHW Charge Duration [%d %unit%]"         { channel="boschthermotechnology:heatpump:myaccount:mygateway:myheatpump:dhw-circuit-dhw1#charge-duration" }
Number:Temperature   DhwSingleChargeSetpoint "DHW Single Charge Setpoint [%.1f %unit%]" { channel="boschthermotechnology:heatpump:myaccount:mygateway:myheatpump:dhw-circuit-dhw1#single-charge-setpoint" }
Number                DhwOperationMode        "DHW Operation Mode [%d]"                 { channel="boschthermotechnology:heatpump:myaccount:mygateway:myheatpump:dhw-circuit-dhw1#operation-mode" }
Switch                DhwCharge               "DHW Instant Charge"                      { channel="boschthermotechnology:heatpump:myaccount:mygateway:myheatpump:dhw-circuit-dhw1#charge" }
Switch                 HeatSourceChStatus      "Heat Source Active"                      { channel="boschthermotechnology:heatpump:myaccount:mygateway:myheatpump:heat-source-1#ch-status" }
Number:Temperature    HeatSourceSupplyTemp    "Heat Source Supply Temp [%.1f %unit%]"    { channel="boschthermotechnology:heatpump:myaccount:mygateway:myheatpump:heat-source-1#actual-supply-temperature" }

Switch                PvEnabled               "PV Enabled"                              { channel="boschthermotechnology:pv:myaccount:mygateway:mypv:photovoltaic#enabled" }
Number:Temperature    SolarCollectorTemp      "Solar Collector Temp [%.1f %unit%]"      { channel="boschthermotechnology:pv:myaccount:mygateway:mypv:solar-thermal#collector-temperature" }

Switch                PoolEnabled             "Pool Enabled"                            { channel="boschthermotechnology:pool:myaccount:mygateway:mypool:enabled" }
Number:Temperature    PoolSetpointTemp        "Pool Setpoint Temp [%.1f %unit%]"        { channel="boschthermotechnology:pool:myaccount:mygateway:mypool:setpoint-temperature" }
Number:Temperature    PoolCurrentTemp         "Pool Current Temp [%.1f %unit%]"         { channel="boschthermotechnology:pool:myaccount:mygateway:mypool:current-temperature" }

Number:Temperature    LivingRoomSetpoint      "Living Room Setpoint [%.1f %unit%]"      { channel="boschthermotechnology:zone-thermostat:myaccount:mygateway:myzone1:manual-room-setpoint" }
Number:Temperature    LivingRoomCurrentTemp   "Living Room Current Temp [%.1f %unit%]"  { channel="boschthermotechnology:zone-thermostat:myaccount:mygateway:myzone1:average-current-temperature" }

Number:Power           ActualChPower           "Actual CH Power [%.1f %unit%]"           { channel="boschthermotechnology:energy-monitoring:myaccount:mygateway:myenergy:actual-ch-power" }
Number:Energy          TotalConsumedEnergy     "Total Consumed Energy [%.1f %unit%]"     { channel="boschthermotechnology:energy-monitoring:myaccount:mygateway:myenergy:total-consumed-energy" }
```

### Sitemap Configuration

```perl
sitemap boschthermotechnology label="Bosch Thermotechnology" {
    Frame label="Gateway" {
        Text item=OutdoorTemperature
        Switch item=AwayModeEnabled
    }
    Frame label="Heat Pump" {
        Setpoint item=ManualRoomSetpoint minValue=5 maxValue=30 step=0.5
        Setpoint item=DhwSingleChargeSetpoint minValue=50 maxValue=70 step=0.5
        Setpoint item=DhwChargeDuration minValue=15 maxValue=2880 step=15
        Selection item=DhwOperationMode mappings=[0="Off", 1="Low", 2="High", 3="Own program", 4="Eco"]
        Switch item=DhwCharge
        Text item=HeatSourceChStatus
        Text item=HeatSourceSupplyTemp
    }
    Frame label="Photovoltaic" {
        Switch item=PvEnabled
        Text item=SolarCollectorTemp
    }
    Frame label="Pool" {
        Switch item=PoolEnabled
        Setpoint item=PoolSetpointTemp minValue=20 maxValue=35 step=0.5
        Text item=PoolCurrentTemp
    }
    Frame label="Living Room" {
        Setpoint item=LivingRoomSetpoint minValue=5 maxValue=30 step=0.5
        Text item=LivingRoomCurrentTemp
    }
    Frame label="Energy Monitoring" {
        Text item=ActualChPower
        Text item=TotalConsumedEnergy
    }
}
```

## Helper Script Sources

The scripts used in [Account Authorization](#account-authorization), reproduced here as plain text. They are identical to the files in the binding's `docs` folder.

### Windows PowerShell

File: `docs/get-auth-code-windows.ps1`

```powershell
<#
.SYNOPSIS
  Gets the SingleKey ID authorization code for the openHAB "Bosch Thermotechnology" binding.
.DESCRIPTION
  Asks for the authUrl of the account bridge, opens it in the browser, and catches the redirect
  with a temporary handler for the com.buderus.tt.dashtt:// scheme (current user only, removed
  afterward). The printed code, which is also copied to the clipboard, goes into the bridge
  field "pasteAuthorizationRedirectUrl".
#>
$AuthUrl = Read-Host 'Paste the authUrl from the openHAB bridge'
if (-not $AuthUrl) { Write-Host 'No URL given.'; exit 1 }

$key = 'HKCU:\Software\Classes\com.buderus.tt.dashtt'
$tmp = Join-Path $env:TEMP 'buderus-redirect.txt'
Remove-Item $tmp -ErrorAction SilentlyContinue

try {
    # Register a handler for the scheme (current user only)
    New-Item "$key\shell\open\command" -Force | Out-Null
    Set-ItemProperty $key -Name '(Default)' -Value 'URL:Buderus Login'
    Set-ItemProperty $key -Name 'URL Protocol' -Value ''
    $cmd = 'powershell.exe -NoProfile -WindowStyle Hidden -Command "Set-Content -LiteralPath ''{0}'' -Value ''%1''"' -f $tmp
    Set-ItemProperty "$key\shell\open\command" -Name '(Default)' -Value $cmd

    Start-Process $AuthUrl
    Write-Host 'Log in in the browser. If it asks whether to open an application, confirm.'
    Write-Host 'Waiting for the redirect (5 minutes max) ...'

    $deadline = (Get-Date).AddMinutes(5)
    while (-not (Test-Path $tmp) -and (Get-Date) -lt $deadline) { Start-Sleep -Seconds 1 }

    if (Test-Path $tmp) {
        Start-Sleep -Milliseconds 300
        $url = (Get-Content -LiteralPath $tmp -Raw).Trim()
        if ($url -match '[?&]code=([^&]+)') {
            $code = [uri]::UnescapeDataString($Matches[1])
            Set-Clipboard -Value $code
            Write-Host "`nCode (also copied to the clipboard):`n$code"
        } elseif ($url -match '[?&]error=([^&]+)') {
            Write-Host "`nLogin failed: $([uri]::UnescapeDataString($Matches[1]))"
        } else {
            Write-Host "`nNo code found in the redirect:`n$url"
        }
    } else {
        Write-Host 'Timed out: no redirect received.'
    }
}
finally {
    # Always remove the handler again
    Remove-Item -Recurse -Force $key -ErrorAction SilentlyContinue
    Remove-Item $tmp -ErrorAction SilentlyContinue
}
```

### Linux

File: `docs/get-auth-code-linux.sh`

```bash
#!/usr/bin/env bash
# Gets the SingleKey ID authorization code for the openHAB "Bosch Thermotechnology" binding (Linux).
# It asks for the authUrl of the account bridge, opens it in the browser, and catches the redirect
# with a temporary handler (current user only) for the com.buderus.tt.dashtt:// scheme (removed afterward).
# The printed code, which is also copied to the clipboard, goes into the bridge field
# "pasteAuthorizationRedirectUrl".

set -u
SCHEME="com.buderus.tt.dashtt"
APPS_DIR="${XDG_DATA_HOME:-$HOME/.local/share}/applications"
DESKTOP="$APPS_DIR/buderus-login-helper.desktop"
HANDLER="$(mktemp -t buderus-handler.XXXXXX.sh)"
OUT="$(mktemp -t buderus-redirect.XXXXXX)"
rm -f "$OUT"

read -r -p "Paste the authUrl from the openHAB bridge: " AUTH_URL
[ -n "$AUTH_URL" ] || { echo "No URL given."; exit 1; }

cleanup() {
  rm -f "$DESKTOP" "$HANDLER" "$OUT"
  # Remove the association from mimeapps.list in case xdg-mime added it there
  for f in "${XDG_CONFIG_HOME:-$HOME/.config}/mimeapps.list"; do
    [ -f "$f" ] && sed -i "\#x-scheme-handler/$SCHEME=#d" "$f"
  done
  command -v update-desktop-database >/dev/null 2>&1 && update-desktop-database "$APPS_DIR" 2>/dev/null
}
trap cleanup EXIT INT TERM

# Helper script that writes the redirect to a file
cat > "$HANDLER" <<EOF
#!/bin/sh
printf '%s' "\$1" > "$OUT"
EOF
chmod +x "$HANDLER"

mkdir -p "$APPS_DIR"
cat > "$DESKTOP" <<EOF
[Desktop Entry]
Type=Application
Name=Buderus Login Helper
NoDisplay=true
Exec=$HANDLER %u
MimeType=x-scheme-handler/$SCHEME;
EOF
command -v update-desktop-database >/dev/null 2>&1 && update-desktop-database "$APPS_DIR" 2>/dev/null
xdg-mime default "$(basename "$DESKTOP")" "x-scheme-handler/$SCHEME"

echo "Opening the browser. Log in and, if asked whether to open an application, confirm."
xdg-open "$AUTH_URL" >/dev/null 2>&1 || echo "Could not start the browser, please open the URL manually."
echo "Waiting for the redirect (5 minutes max) ..."

for _ in $(seq 1 300); do
  [ -s "$OUT" ] && break
  sleep 1
done

if [ ! -s "$OUT" ]; then
  echo "Timed out: no redirect received."
  exit 1
fi
sleep 0.3
URL="$(cat "$OUT")"

CODE="$(printf '%s' "$URL" | sed -n 's/.*[?&]code=\([^&]*\).*/\1/p')"
if [ -z "$CODE" ]; then
  ERR="$(printf '%s' "$URL" | sed -n 's/.*[?&]error=\([^&]*\).*/\1/p')"
  if [ -n "$ERR" ]; then echo "Login failed: $ERR"; else echo "No code found in the redirect: $URL"; fi
  exit 1
fi
CODE="$(printf '%b' "${CODE//%/\\x}")"   # URL-decode

if   command -v wl-copy >/dev/null 2>&1; then printf '%s' "$CODE" | wl-copy;                       COPIED=1
elif command -v xclip   >/dev/null 2>&1; then printf '%s' "$CODE" | xclip -selection clipboard;   COPIED=1
elif command -v xsel    >/dev/null 2>&1; then printf '%s' "$CODE" | xsel --clipboard --input;     COPIED=1
else COPIED=0; fi

echo
echo "Code:"
echo "$CODE"
[ "$COPIED" = 1 ] && echo "(copied to the clipboard)"
```

### macOS

File: `docs/get-auth-code-macos.sh`

```bash
#!/usr/bin/env bash
# Gets the SingleKey ID authorization code for the openHAB "Bosch Thermotechnology" binding (macOS).
# It asks for the authUrl of the account bridge, opens it in the browser, and catches the redirect
# with a temporary AppleScript application (current user only) for the com.buderus.tt.dashtt:// scheme (removed afterward).
# The printed code, which is also copied to the clipboard, goes into the bridge field
# "pasteAuthorizationRedirectUrl".

set -u
SCHEME="com.buderus.tt.dashtt"
WORK="$(mktemp -d -t buderus-login)"
APP="$WORK/BuderusLoginHelper.app"
OUT="$WORK/redirect.txt"
LSREG="/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister"

read -r -p "Paste the authUrl from the openHAB bridge: " AUTH_URL
[ -n "$AUTH_URL" ] || { echo "No URL given."; exit 1; }

cleanup() {
  "$LSREG" -u "$APP" >/dev/null 2>&1
  rm -rf "$WORK"
}
trap cleanup EXIT INT TERM

# AppleScript application that writes the redirect to a file
osacompile -o "$APP" \
  -e 'on open location theURL' \
  -e "do shell script \"printf %s \" & quoted form of theURL & \" > \" & quoted form of \"$OUT\"" \
  -e 'end open location' || { echo "osacompile failed."; exit 1; }

# Register the URL scheme in Info.plist
PL="$APP/Contents/Info.plist"
PB=/usr/libexec/PlistBuddy
"$PB" -c "Add :CFBundleURLTypes array" "$PL"
"$PB" -c "Add :CFBundleURLTypes:0 dict" "$PL"
"$PB" -c "Add :CFBundleURLTypes:0:CFBundleURLName string BuderusLogin" "$PL"
"$PB" -c "Add :CFBundleURLTypes:0:CFBundleURLSchemes array" "$PL"
"$PB" -c "Add :CFBundleURLTypes:0:CFBundleURLSchemes:0 string $SCHEME" "$PL"
"$PB" -c "Add :LSUIElement bool true" "$PL"

codesign --force --sign - "$APP" >/dev/null 2>&1
"$LSREG" -f "$APP"

echo "Opening the browser. Log in and, if asked whether to open an application, confirm."
open "$AUTH_URL"
echo "Waiting for the redirect (5 minutes max) ..."

for _ in $(seq 1 300); do
  [ -s "$OUT" ] && break
  sleep 1
done

if [ ! -s "$OUT" ]; then
  echo "Timed out: no redirect received."
  exit 1
fi
sleep 0.3
URL="$(cat "$OUT")"

CODE="$(printf '%s' "$URL" | sed -n 's/.*[?&]code=\([^&]*\).*/\1/p')"
if [ -z "$CODE" ]; then
  ERR="$(printf '%s' "$URL" | sed -n 's/.*[?&]error=\([^&]*\).*/\1/p')"
  if [ -n "$ERR" ]; then echo "Login failed: $ERR"; else echo "No code found in the redirect: $URL"; fi
  exit 1
fi
CODE="$(printf '%b' "${CODE//%/\\x}")"   # URL-decode

printf '%s' "$CODE" | pbcopy
echo
echo "Code (copied to the clipboard):"
echo "$CODE"
```
