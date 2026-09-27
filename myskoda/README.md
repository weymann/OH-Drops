# MyŠkoda Binding

The MyŠkoda Binding integrates Škoda vehicles into openHAB through the official MyŠkoda Public API.
One `account` bridge represents an API key, and one `vehicle` Thing represents each vehicle that this key covers.
The binding reads the state of the vehicle and can start or stop charging, climatisation, auxiliary heating and active ventilation.

An API key is created in the MyŠkoda portal at <https://go.skoda.eu/api-keys> and is bound to the vehicles that are selected during creation.
The key expires after a while, so the binding reports its expiry as a Channel and warns in the log before it runs out.

## Supported Things

| Thing Type ID | Description |
|---------------|-------------|
| `account` | Bridge that holds the MyŠkoda API key and polls all vehicles the key covers. |
| `vehicle` | A single Škoda vehicle, identified by its VIN. Requires an `account` bridge. |

## Discovery

Auto-discovery is not available for this binding.
The MyŠkoda Public API has no endpoint that lists the vehicles of an API key, and a key only grants access to the vehicles that were selected when it was created.
Every `vehicle` Thing therefore has to be added manually, and the VIN of the vehicle has to be entered.
The VIN is shown in the MyŠkoda app and in the vehicle documents.

Once a vehicle is configured, the binding fills the Thing properties with the data the vehicle reports, for example the user defined vehicle name, the licence plate and the car type.
The `renderUrl` property contains the vehicle picture of the MyŠkoda service and can be used in Main UI widgets.

## Binding Configuration

The binding has no binding level configuration.
Everything is configured on the `account` and `vehicle` Things.

## Thing Configuration

### `account` Thing Configuration

| Name | Type | Description | Default | Required | Advanced |
|------|------|-------------|---------|----------|----------|
| `apiKey` | text | API key created in the MyŠkoda portal, covering the vehicles to use. | N/A | yes | no |
| `refreshInterval` | integer | Interval in seconds at which the vehicles of this account are polled. Minimum 60. | 600 | no | yes |
| `endpoint` | text | Base URL of the MyŠkoda Public API. Change it only for a test setup. | <https://public.api.connect.skoda-auto.cz> | no | yes |

The API key is stored as a password field and is never written to the log or to Thing properties.

### `vehicle` Thing Configuration

| Name | Type | Description | Default | Required | Advanced |
|------|------|-------------|---------|----------|----------|
| `vin` | text | Vehicle Identification Number with 17 characters, for example `TMBJB9NY5RF999999`. | N/A | yes | no |
| `dataParts` | text (multiple) | Parts of the vehicle data to request. No selection requests everything the vehicle supports. | empty | no | yes |
| `securityPin` | text | Security PIN that the vehicle requires to start the auxiliary heating. | N/A | no | yes |
| `refreshInterval` | integer | Vehicle specific polling interval in seconds. `0` uses the interval of the account. | 0 | no | yes |

#### Requesting Data Parts

`dataParts` limits which parts of the vehicle data are requested from the API.
The following values are available:

| Value | Content |
|-------|---------|
| `info` | Vehicle name, licence plate and render image (the VIN is always returned). |
| `status` | Lock, doors, windows, sunroof, trunk, bonnet and lights. |
| `odometer` | Mileage. |
| `fuelStatus` | Fuel level, AdBlue and ranges of vehicles with a combustion engine. |
| `parkingPosition` | Last known parking position and its address. |
| `airConditioning` | Climatisation state, target temperature and window heating. |
| `auxiliaryHeating` | State and settings of the auxiliary heater. |
| `activeVentilation` | State of the active ventilation. |
| `charging` | Charging state, battery, ranges and charging settings. |
| `chargingProfiles` | Saved charging locations with their timers. |
| `operations` | Remote operations the vehicle supports. |

An empty selection is the recommended default: the binding then asks for everything the vehicle supports, so a vehicle without a high voltage battery is never asked for battery data.
If a selected part turns out to be unsupported, the binding stops requesting it and reports the resulting list in the `requestedParts` Thing property.
Deselecting a part keeps its channels at `UNDEF` and reduces the size of the responses.

## Channels

The channel ID in the tables below is the group and channel as it appears in the channel UID, for example `charging#batteryLevel`.
A full channel UID looks like `myskoda:vehicle:home:enyaq:charging#batteryLevel`.

`R` marks read only channels, `R/W` marks channels that accept commands.

### `account` Channels

| Channel | Type | Read/Write | Description |
|---------|------|------------|-------------|
| `apiKeyExpires` | DateTime | R | Date and time at which the API key stops working, used to be reminded before it expires. |
| `rateLimitRemaining` | Number | R | Number of requests the API key may still send in the current period. |

### `vehicle` Channels

#### Group `status`

| Channel | Type | Read/Write | Description |
|---------|------|------------|-------------|
| `status#lock` | Switch | R | `ON` when the vehicle is locked, `UNDEF` when the vehicle cannot tell. |
| `status#lockDetail` | String | R | Lock state of doors and trunk as reported by the vehicle: `YES`, `NO`, `OPENED`, `TRUNK_OPENED` or `UNKNOWN`. |
| `status#doors` | Contact | R | `OPEN` when any door is open. |
| `status#windows` | Contact | R | `OPEN` when any side window is open. |
| `status#sunroof` | Contact | R | `OPEN` when the sunroof is open. |
| `status#trunk` | Contact | R | `OPEN` when the trunk is open. |
| `status#bonnet` | Contact | R | `OPEN` when the bonnet is open. |
| `status#lights` | Switch | R | `ON` when any light is switched on. |
| `status#lastUpdated` | DateTime | R | Date and time at which the vehicle captured this data. |

The binding cannot lock or unlock a vehicle, because the API has no such operation.

#### Group `odometer`

| Channel | Type | Read/Write | Description |
|---------|------|------------|-------------|
| `odometer#mileage` | Number:Length | R | Total distance the vehicle has been driven. |
| `odometer#lastUpdated` | DateTime | R | Date and time at which the vehicle captured this data. |

#### Group `fuel`

| Channel | Type | Read/Write | Description |
|---------|------|------------|-------------|
| `fuel#totalRange` | Number:Length | R | Combined remaining range of all engines. |
| `fuel#adBlueRange` | Number:Length | R | Remaining range with the AdBlue of a diesel engine. |
| `fuel#primaryRange` | Number:Length | R | Remaining range of the primary engine. |
| `fuel#primaryLevel` | Number:Dimensionless | R | Fuel level of the primary engine in percent. |
| `fuel#primaryStateOfCharge` | Number:Dimensionless | R | State of charge of the primary engine in percent. |
| `fuel#primaryEngineType` | String | R | Type of the primary engine: `ELECTRIC`, `GASOLINE`, `DIESEL`, `CNG`, `LPG` or `UNKNOWN`. |
| `fuel#secondaryRange` | Number:Length | R | Remaining range of the secondary engine. |
| `fuel#secondaryLevel` | Number:Dimensionless | R | Fuel level of the secondary engine in percent. |
| `fuel#secondaryStateOfCharge` | Number:Dimensionless | R | State of charge of the secondary engine in percent. |
| `fuel#secondaryEngineType` | String | R | Type of the secondary engine, same values as the primary engine type. |
| `fuel#lastUpdated` | DateTime | R | Date and time at which the vehicle captured this data. |

A hybrid reports two engines, and the engine type tells which of them carries a fuel level and which one carries a state of charge.
Only the field the vehicle reports is filled, the other channel stays `UNDEF`.

#### Group `charging`

| Channel | Type | Read/Write | Description |
|---------|------|------------|-------------|
| `charging#state` | String | R | Charging state: `CONNECT_CABLE`, `CHARGING`, `CONSERVING`, `READY_FOR_CHARGING`, `DISCHARGING` or `CHARGING_INTERRUPTED`. |
| `charging#active` | Switch | R/W | `ON` starts the charging process, `OFF` stops it, `UNDEF` when the vehicle reports no usable state. |
| `charging#batteryLevel` | Number:Dimensionless | R | State of charge of the high voltage battery in percent. |
| `charging#remainingRange` | Number:Length | R | Remaining range with the high voltage battery. |
| `charging#chargePower` | Number:Power | R | Power the vehicle currently charges with. |
| `charging#chargeRate` | Number:Speed | R | Range the vehicle gains per hour while charging. |
| `charging#remainingTime` | Number:Time | R | Time until the charging process is complete. |
| `charging#fullyChargedAt` | DateTime | R | Date and time at which the vehicle expects to be fully charged. |
| `charging#plugConnected` | Switch | R | `ON` when the charging cable is plugged in. |
| `charging#plugLocked` | Switch | R | `ON` when the charging cable is locked to the vehicle. |
| `charging#chargeType` | String | R | Type of the current charging process: `AC`, `DC` or `OFF`. |
| `charging#inSavedLocation` | Switch | R | `ON` when the vehicle is at a saved charging location. |
| `charging#targetStateOfCharge` | Number:Dimensionless | R/W | Target state of charge in percent. |
| `charging#chargeMode` | String | R/W | Charging mode: `MANUAL`, `TIMER`, `TIMER_CHARGING_WITH_CLIMATISATION`, `PREFERRED_CHARGING_TIMES`, `ONLY_OWN_CURRENT`, `IMMEDIATE_DISCHARGING` or `HOME_STORAGE_CHARGING`. |
| `charging#batteryCareMode` | Switch | R | `ON` when the gentle charging mode for the battery is active. |
| `charging#careModeTargetValue` | Number:Dimensionless | R | Target state of charge that battery care mode uses. |
| `charging#autoUnlockPlug` | Switch | R | `ON` when the vehicle unlocks the charging cable after charging. |
| `charging#maxChargeCurrent` | String | R | Current limit setting used for alternating current charging: `REDUCED` or `MAXIMUM`. |
| `charging#maxChargeCurrentAmpere` | Number:ElectricCurrent | R | Maximum charging current in ampere. |
| `charging#lastUpdated` | DateTime | R | Date and time at which the vehicle captured this data. |

Vehicles typically accept a target state of charge between 50 and 100 in steps of 10, which is what the user interface offers.
The binding itself accepts the full range from 1 to 100 that the API documents, so a rule can send an unusual value if the vehicle supports it.
If the vehicle is at a saved charging location, the target state of charge of that profile applies and is shown in `chargingProfiles#currentTarget`.
Charging care mode, the automatic plug unlock and the alternating current limit are read only, because the API offers no operation to change them.

#### Group `chargingProfiles`

| Channel | Type | Read/Write | Description |
|---------|------|------------|-------------|
| `chargingProfiles#currentName` | String | R | Name of the charging profile the vehicle is currently in. |
| `chargingProfiles#currentTarget` | Number:Dimensionless | R | Target state of charge of that profile in percent. |
| `chargingProfiles#nextChargingTime` | String | R | Local time of day at which the profile starts charging, in `HH:mm` format. |
| `chargingProfiles#profilesJson` | String | R | All charging profiles including their timers, as JSON. |
| `chargingProfiles#lastUpdated` | DateTime | R | Date and time at which the vehicle captured this data. |

Writing charging profiles is not supported yet.
The API requires the complete profile to be submitted, which does not fit a channel, so this is planned as a Thing action.

#### Group `climate`

| Channel | Type | Read/Write | Description |
|---------|------|------------|-------------|
| `climate#airConditioning` | Switch | R/W | `ON` starts the climatisation, `OFF` stops it. |
| `climate#airConditioningState` | String | R | Climatisation state: `OFF`, `COOLING`, `HEATING`, `HEATING_AUXILIARY`, `VENTILATION`, `COMPLETED`, `UNKNOWN` or `UNSUPPORTED`. |
| `climate#targetTemperature` | Number:Temperature | R/W | Cabin temperature the next climatisation run aims for. |
| `climate#temperatureReachedAt` | DateTime | R | Date and time at which the cabin is expected to reach the target temperature. |
| `climate#withoutExternalPower` | Switch | R | `ON` when climatisation may run without the vehicle being connected to power. |
| `climate#atUnlock` | Switch | R | `ON` when climatisation starts automatically at unlock. |
| `climate#windowHeating` | Switch | R | `ON` when the electric window heating is enabled. |
| `climate#windowHeatingFront` | Switch | R | State of the front window heating. |
| `climate#windowHeatingRear` | Switch | R | State of the rear window heating. |
| `climate#auxiliaryHeating` | Switch | R/W | `ON` starts the auxiliary heater, `OFF` stops it. Requires the `securityPin`. |
| `climate#auxiliaryHeatingState` | String | R | Heater state: `OFF`, `PREHEATING`, `HEATING_AUXILIARY`, `VENTILATION`, `UNKNOWN` or `UNSUPPORTED`. |
| `climate#ventilation` | Switch | R/W | `ON` starts the active ventilation, `OFF` stops it. |
| `climate#ventilationState` | String | R | Ventilation state: `OFF`, `PREHEATING`, `VENTILATION`, `UNKNOWN` or `UNSUPPORTED`. |
| `climate#airConditioningUpdated` | DateTime | R | Date and time at which the vehicle captured its climatisation data. |
| `climate#auxiliaryHeatingUpdated` | DateTime | R | Date and time at which the vehicle captured its auxiliary heater data. |
| `climate#ventilationUpdated` | DateTime | R | Date and time at which the vehicle captured its ventilation data. |

The comfort functions run for a limited time that the vehicle decides.
`climate#targetTemperature` is not written on its own: it is the value that the next climatisation or auxiliary heating start uses.
The three freshness channels are separate because the vehicle captures the state of the three functions independently.

#### Group `position`

| Channel | Type | Read/Write | Description |
|---------|------|------------|-------------|
| `position#state` | String | R | `PARKED` or `IN_MOTION`. |
| `position#location` | Location | R | Last known parking position. |
| `position#address` | String | R | Address of the last known parking position. |

The API reports the last known parking position, so a vehicle in motion keeps showing the previous location and only its state changes to `IN_MOTION`.
This group is only filled when the `parkingPosition` part is requested.

### Vehicle Thing Properties

| Property | Description |
|----------|-------------|
| `vin` | Vehicle Identification Number. |
| `name` | User defined vehicle name, or the model name when the vehicle has no name. |
| `licensePlate` | Licence plate of the vehicle. |
| `renderUrl` | URL of the vehicle picture of the MyŠkoda service. |
| `carType` | `HYBRID`, `GASOLINE`, `DIESEL`, `CNG`, `LPG` or `UNKNOWN`. |
| `supportedOperations` | Remote operations the vehicle reported as supported. |
| `availableChargeModes` | Charge modes the vehicle offers. |
| `requestedParts` | Parts of the vehicle data the binding currently requests, or `all`. |

## Full Example

### Thing Configuration

```java
Bridge myskoda:account:home "MyŠkoda Account" [ apiKey="YOUR_API_KEY", refreshInterval=600 ] {
    Thing vehicle enyaq "My Enyaq" [ vin="TMBJB9NY5RF999999", securityPin="1234" ]
}
```

To limit the requested data, add the parts as a list.
Each part is a separate value:

```java
Bridge myskoda:account:home "MyŠkoda Account" [ apiKey="YOUR_API_KEY" ] {
    Thing vehicle enyaq "My Enyaq" [ vin="TMBJB9NY5RF999999", dataParts="status", "odometer", "charging", "airConditioning" ]
}
```

### Item Configuration

```java
Number:Length    Skoda_Mileage              "Mileage [%.0f %unit%]"                 { channel="myskoda:vehicle:home:enyaq:odometer#mileage" }
Number:Length    Skoda_Range                "Range [%.0f %unit%]"                   { channel="myskoda:vehicle:home:enyaq:fuel#totalRange" }
Number:Dimensionless Skoda_BatteryLevel     "Battery [%.0f %%]"                     { channel="myskoda:vehicle:home:enyaq:charging#batteryLevel" }
Number:Power     Skoda_ChargePower          "Charging Power [%.1f %unit%]"          { channel="myskoda:vehicle:home:enyaq:charging#chargePower" }
Switch           Skoda_Charging             "Charging"                              { channel="myskoda:vehicle:home:enyaq:charging#active" }
String           Skoda_ChargingState        "Charging State [%s]"                   { channel="myskoda:vehicle:home:enyaq:charging#state" }
Number:Dimensionless Skoda_TargetCharge     "Target Charge [%.0f %%]"               { channel="myskoda:vehicle:home:enyaq:charging#targetStateOfCharge" }
Switch           Skoda_Lock                 "Locked"                                { channel="myskoda:vehicle:home:enyaq:status#lock" }
Contact          Skoda_Doors                "Doors [%s]"                            { channel="myskoda:vehicle:home:enyaq:status#doors" }
Contact          Skoda_Windows              "Windows [%s]"                          { channel="myskoda:vehicle:home:enyaq:status#windows" }
Switch           Skoda_AirConditioning      "Air Conditioning"                      { channel="myskoda:vehicle:home:enyaq:climate#airConditioning" }
Number:Temperature Skoda_TargetTemperature  "Target Temperature [%.1f %unit%]"      { channel="myskoda:vehicle:home:enyaq:climate#targetTemperature" }
Switch           Skoda_AuxiliaryHeating     "Auxiliary Heating"                     { channel="myskoda:vehicle:home:enyaq:climate#auxiliaryHeating" }
Location         Skoda_Location             "Location"                              { channel="myskoda:vehicle:home:enyaq:position#location" }
DateTime         Skoda_ChargingUpdated      "Charging Data From [%1$tY-%1$tm-%1$td %1$tH:%1$tM]" { channel="myskoda:vehicle:home:enyaq:charging#lastUpdated" }
```

### Sitemap Configuration

```java
sitemap myskoda label="MyŠkoda" {
    Frame label="Vehicle" {
        Text item=Skoda_Mileage
        Text item=Skoda_Range
        Text item=Skoda_Lock
        Text item=Skoda_Doors
        Text item=Skoda_Windows
    }
    Frame label="Charging" {
        Text item=Skoda_BatteryLevel
        Switch item=Skoda_Charging
        Text item=Skoda_ChargingState
        Setpoint item=Skoda_TargetCharge minValue=50 maxValue=100 step=10
        Text item=Skoda_ChargePower
        Text item=Skoda_ChargingUpdated
    }
    Frame label="Climate" {
        Switch item=Skoda_AirConditioning
        Setpoint item=Skoda_TargetTemperature minValue=16 maxValue=30 step=0.5
        Switch item=Skoda_AuxiliaryHeating
    }
    Frame label="Position" {
        Mapview item=Skoda_Location height=10
    }
}
```

## API Key, Expiry and Rate Limits

The API key is created in the MyŠkoda portal and is bound to the vehicles that are selected there.
A vehicle whose VIN is not covered by the key is reported as `OFFLINE` with the message that the API key does not cover this vehicle.

Keys expire.
The `apiKeyExpires` channel of the account bridge carries the expiry date, and the binding logs a warning within the last seven days before the key runs out.
Create a new key in the portal before that date and update the `apiKey` parameter of the account bridge.

The request quota belongs to the API key and is therefore shared by all vehicles of one account.
The bridge polls the vehicles one after the other, never in parallel, and pauses all requests when the quota is used up.
The `rateLimitRemaining` channel shows how many requests are left in the current period.
Polling faster than 600 seconds is rarely useful, because a sleeping vehicle only sends new data when it wakes up.

## Data Freshness

A vehicle that is parked and asleep does not send new data, so the channels keep the last values the vehicle reported.
Every group has a `lastUpdated` channel that shows when the vehicle captured its data, which is the reliable answer to the question of how fresh a value is.
Climate data is reported separately for climatisation, auxiliary heating and ventilation, so the climate group carries three freshness channels.

`REFRESH` on any channel triggers an immediate poll of that vehicle.
The refresh respects the quota of the API key, so it may be deferred.

## Limitations

- There is no discovery, because the API cannot list the vehicles of an API key.
- The vehicle cannot be locked, unlocked, honked or flashed, and windows and sunroof cannot be controlled, because the API has no such operations.
- Charging care mode, the automatic plug unlock and the alternating current limit are read only.
- Charging profiles can be read but not written yet.
- Comfort functions run for a duration that the vehicle chooses; the binding does not set a duration.
- Auxiliary heating requires the security PIN of the vehicle to be configured.

## Troubleshooting

- A vehicle shows `OFFLINE` with "The API key does not cover this vehicle": the VIN is not part of the vehicles selected for this API key.
- All vehicles show `OFFLINE` with a communication error: the quota of the API key is used up, or the MyŠkoda service is temporarily unavailable.
- A channel stays `UNDEF`: the vehicle does not support the corresponding part of the data, or the part is currently disabled.
- Values do not change even though the vehicle moves: the vehicle has not sent new data yet, check the `lastUpdated` channels.
- The auxiliary heating does not start: check that the `securityPin` parameter is configured.

## Reporting Problems

The binding logs its decisions with the standard loggers of openHAB.
Enable them in the openHAB console before you reproduce a problem:

```shell
log:set DEBUG org.openhab.binding.myskoda
log:set TRACE org.openhab.binding.myskoda
```

`DEBUG` writes one line per poll with the requested data parts, the parts the vehicle delivered and the errors the response carried for the rest.
It also reports every rate limit deferral, every capability the binding stops requesting and every command it refuses.

`TRACE` additionally writes the complete HTTP exchange, which is what identifies a changed or unexpected payload.

Set the logger back to its default after the problem has been reproduced:

```shell
log:set INFO org.openhab.binding.myskoda
```

Trace output contains vehicle data, including the VIN, the licence plate and the parking address, so review and redact it before sharing it.
The API key is never logged, and the security PIN is masked in traced request bodies.
