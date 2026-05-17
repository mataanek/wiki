# Netatmo Weather Station Dashboard

This dashboard shows collected Netatmo weather station data using Dataview queries.

## Current Conditions (Last 5-minute collection)

```dataview
TABLE 
    station_name as "Station",
    main_temperature_c as "Temperature (°C)",
    main_humidity_percent as "Humidity (%)",
    main_pressure_hpa as "Pressure (hPa)",
    main_co2_ppm as "CO₂ (ppm)",
    main_noise_db as "Noise (dB)"
FROM "smart_home/netatmo"
WHERE file.name = sensor_log.jsonl
FLATTEN file.log as log
SORT log.timestamp DESC
LIMIT 1
```

## Daily Trends (Last 24 hours)

### Temperature Trend
```dataview
TABLE 
    date as "Date",
    avg(temperature_c) as "Avg Temp (°C)",
    min(temperature_c) as "Min Temp (°C)",
    max(temperature_c) as "Max Temp (°C)"
FROM "smart_home/netatmo/historical"
WHERE file.name = "70:ee:50:c2:2c:ee_main_Temperature_historical_1d.json"
GROUP BY date
ORDER BY date DESC
LIMIT 7
```

### Humidity Trend
```dataview
TABLE 
    date as "Date",
    avg(humidity_percent) as "Avg Humidity (%)",
    min(humidity_percent) as "Min Humidity (%)",
    max(humidity_percent) as "Max Humidity (%)"
FROM "smart_home/netatmo/historical"
WHERE file.name = "70:ee:50:c2:2c:ee_main_Humidity_historical_1d.json"
GROUP BY date
ORDER BY date DESC
LIMIT 7
```

### CO₂ Levels
```dataview
TABLE 
    date as "Date",
    avg(co2_ppm) as "Avg CO₂ (ppm)",
    min(co2_ppm) as "Min CO₂ (ppm)",
    max(co2_ppm) as "Max CO₂ (ppm)"
FROM "smart_home/netatmo/historical"
WHERE file.name = "70:ee:50:c2:2c:ee_main_CO2_historical_1d.json"
GROUP BY date
ORDER BY date DESC
LIMIT 7
```

## Weekly Summary (Last 7 days)

```dataview
TABLE 
    week_start as "Week Start",
    week_end as "Week End",
    round(avg(temperature_c), 1) as "Avg Temp (°C)",
    round(avg(humidity_percent), 1) as "Avg Humidity (%)",
    round(avg(pressure_hpa), 1) as "Avg Pressure (hPa)",
    round(avg(co2_ppm), 0) as "Avg CO₂ (ppm)",
    round(avg(noise_db), 1) as "Avg Noise (dB)"
FROM (
    SELECT 
        dateformat(date, "yyyy-MM-dd") as date,
        temperature_c,
        humidity_percent,
        pressure_hpa,
        co2_ppm,
        noise_db
    FROM "smart_home/netatmo/historical"
    WHERE file.name = "70:ee:50:c2:2c:ee_main_Temperature_historical_7d.json"
    FLATTEN data as point
    FLATTEN point.value as temp_val
    WHERE temp_temp_val != null
    LET temperature_c = temp_val[0]
) 
GROUP BY week_start = dateformat(date - (date - date("2026-05-09") % 7), "yyyy-MM-dd")
    week_end = dateformat(week_start + 6, "yyyy-MM-dd")
```

## Module Status (Current)

```dataview
TABLE 
    module_name as "Module",
    temperature_c as "Temp (°C)",
    humidity_percent as "Humidity (%)",
    pressure_hpa as "Pressure (hPa)",
    co2_ppm as "CO₂ (ppm)",
    noise_db as "Noise (dB)"
FROM "smart_home/netatmo"
WHERE contains(file.path, "sensor_log.jsonl")
FLATTEN file.log as log
FLATTEN log.modules as module
WHERE module.module_name != null
SORT module.module_name
```

## Data Collection Status

```dataview
TABLE 
    file_name as "File",
    modified as "Last Modified",
    size as "Size (bytes)"
FROM "smart_home/netatmo"
WHERE file.extension = "json"
SORT modified DESC
```

> **Last updated**: {{date}}
> **Data collection**: Active (5-minute cron + daily historical extraction)
> **Storage location**: `~/wiki/smart_home/netatmo/`
