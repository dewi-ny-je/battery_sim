# Battery Simulator integration/custom component for home assistant

[![downloads](https://img.shields.io/badge/dynamic/json?color=41BDF5&logo=home-assistant&label=integration%20usage&suffix=%20installs&cacheSeconds=15600&url=https://analytics.home-assistant.io/custom_integrations.json&query=$.battery_sim.total)](https://github.com/dewi-ny-je/battery_sim/)

Allows you to model how much energy you would save with a home battery if you currently export energy to the grid e.g. from solar panels. Requires an energy monitor that can measure import and export energy. Whenever you are exporting energy your simulated battery will charge up and whenevery you are importing it will discharge. Battery charge percentage and total energy saved are in the attributes. 

Please note this is a simulation and a real battery may behave differently and not all batteries will support all the features available in this simulation. In particular battery_sim allows you to simulate batteries that charge and discharge across multiple phases and various modes including charge_only, discharge_only etc that may not be available in all real world batteries.

## Setup

The easiest way to get battery_sim is to use HACS to add it as an integration. If you do not want to use HACS, copy this repository into the `custom_components` folder in your Home Assistant configuration directory.

After installation, create one or more batteries. The recommended approach is to go to **Settings > Devices & Services**, click **Add Integration**, search for **Battery Simulation**, and work through the dialog for each battery you want to simulate.

You can also define batteries in `configuration.yaml`. Each battery is created under `battery_sim:` using a unique slug key. All YAML parameters currently supported by the integration are listed below.

### YAML parameters

| Parameter | Required | Description |
| --- | --- | --- |
| `import_sensor` | Yes | Entity ID of the cumulative energy-import sensor in kWh, for example the output of a `utility_meter`. |
| `export_sensor` | Yes | Entity ID of the cumulative energy-export sensor in kWh. |
| `size_kwh` | Yes | Usable battery capacity in kWh. Use a floating-point value such as `13.5`. |
| `max_discharge_rate_kw` | Yes | Maximum discharge power in kW. Use a floating-point value such as `5.0`. |
| `max_charge_rate_kw` | No | Maximum charge power in kW. Defaults to `1.0` if omitted. |
| `discharge_efficiency` | No | Battery discharge efficiency from `0` to `1`. If omitted, the integration falls back to `efficiency` when that legacy key is present, otherwise `1.0`. You can enter either a single value between 0 and 1, or a power curve such as `0:0.90, 2.5:0.94, 5:0.95`. |
| `charge_efficiency` | No | Battery charge efficiency from `0` to `1`. Defaults to `1.0` if omitted. You can enter either a single value between 0 and 1, or a power curve such as `0:0.90, 2.5:0.94, 5:0.95`. |
| `efficiency` | No | Legacy single-value efficiency key kept for backward compatibility. It is used as the default for `discharge_efficiency` when the newer split efficiency keys are not set. |
| `energy_tariff` | No | Entity ID of a tariff sensor. For backward-compatible YAML setups this populates the import tariff input. |
| `energy_import_tariff` | No | Entity ID of an import tariff sensor. |
| `energy_export_tariff` | No | Entity ID of an export tariff sensor. |
| `name` | No | Friendly name shown in Home Assistant. If omitted, the YAML object key is used. |
| `rated_battery_cycles` | No | Number of full cycles at which end-of-life degradation is reached. Defaults to `6000`. |
| `end_of_life_degradation` | No | Remaining usable capacity at `rated_battery_cycles`, expressed from `0` to `1`. Defaults to `0.8`. |
| `update_frequency` | No | Maximum interval between updates in seconds. Defaults to `60`, which is also the recommended value. Faster updates do not improve accuracy. |

### Example YAML

```yaml
battery_sim:
  tesla_powerwall:
    name: Tesla Powerwall
    import_sensor: sensor.circuitsetup_cumulative_import_energy_kwh
    export_sensor: sensor.circuitsetup_cumulative_export_energy_kwh
    size_kwh: 13.5
    max_discharge_rate_kw: 5.0
    max_charge_rate_kw: 3.68
    discharge_efficiency: 0:0.92, 2.5:0.95, 5:0.95
    charge_efficiency: 0:0.90, 2:0.94, 3.68:0.95
    rated_battery_cycles: 6000
    end_of_life_degradation: 0.8
    update_frequency: 60
    energy_tariff: sensor.energy_tariff
  lg_chem_resu10h:
    name: LG Chem
    import_sensor: sensor.circuitsetup_cumulative_import_energy_kwh
    export_sensor: sensor.circuitsetup_cumulative_export_energy_kwh
    size_kwh: 9.3
    max_discharge_rate_kw: 5.0
    max_charge_rate_kw: 3.3
    discharge_efficiency: 0.975
    charge_efficiency: 0.975
    energy_import_tariff: sensor.grid_import_tariff
    energy_export_tariff: sensor.grid_export_tariff
```

## Battery Efficiencies

This integration supports separate `charge_efficiency` and `discharge_efficiency` values because battery efficiency is not flat across the full operating range. In practice, manufacturers often publish an efficiency curve: efficiency changes with charge or discharge power, and lower power levels typically produce worse results than the headline datasheet number.

You can configure each efficiency either as:

- a single value, for example `0.95`
- or a power curve, for example `0:0.88, 0.5:0.90, 2.5:0.94, 5:0.95`

The power values are in kW. During each battery update, the integration computes the average charging or discharging power as:

`energy transferred during the interval / interval duration`

It then linearly interpolates the efficiency from the configured points and uses that value for the update. Two extra sensors report the charge and discharge efficiency used for the most recent update.

A simplified efficiency curve usually looks something like this:

| Charge/discharge power | Typical behavior |
| --- | --- |
| Very low power | Efficiency drops because fixed inverter and standby losses dominate. |
| Medium power | Efficiency is usually at or near the best point on the curve. |
| Very high power | Efficiency can taper off again because of conversion and thermal losses. |

If you use fixed efficiencies rather than an efficiency curve, the best approach is to choose values that represent your most common operating range. If most of your simulated battery activity happens at low power, set lower `charge_efficiency` and `discharge_efficiency` values to approximate that part of the curve and choose more conservative efficiency values than the best-case number in the datasheet.

When reading a datasheet, make sure the quoted efficiency covers the whole path you care about. Some manufacturers quote inverter efficiency only, which may describe battery-to-AC conversion while excluding charging losses into the battery. In those cases, use conservative values for both charge and discharge.

## Battery Degradation

This integration models the degradation of the battery linearly, from 100% usable capacity (no degradation) at 0 cycles and (by default)
80% usable capacity at 6000 cycles. The values can be provided in the settings.

The state of charge (SOC) is not limited progressively, the capacity associated with 100% SOC simply decreases over time.

A new action is provided to manually set the current number of battery cycles, to simulate immediately old batteries.

## Energy Dashboard

You can configure battery_sim to display your simulated battery on your Energy Dashboard:

![Screenshot 2022-03-15 19 36 47](https://user-images.githubusercontent.com/79175134/158349586-cfc64761-0614-4067-a18a-5603d2288d7c.png)


![image](https://user-images.githubusercontent.com/79175134/157999078-0174ab36-9f71-47c8-8585-73d6eb3acec8.png)

## Debug

If you are having problems it is helpful to get the debug log for the battery by adding:

```
logger:
  default: critical
  logs:
    custom_components.battery_sim: debug
```

to your configuration.yaml and then restarting. If you leave it to run for a few minutes go to logs then and click "load full log" you should see entries from the battery saying it's been set up and then each time it receives an update. If you need to raise an issue then including this code is helpful.

## Acknowledgements

Original idea and integration developed by hif2k1. Thanks for your work! 
