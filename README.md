# CumulusMX — Fork with WindGuru Fix

**Forked from [cumulusmx/CumulusMX](https://github.com/cumulusmx/CumulusMX)**

This fork fixes a bug in WindGuru uploads that causes `NaN` wind values for cloud-connected weather stations (e.g., Davis WeatherLink Cloud). The upstream project is the Cumulus MX weather program originally written by Steve Loft and now maintained by the CumulusMX community.

---

## Bug: WindGuru NaN Wind Values

### Symptoms

- WindGuru station page shows **no data** (or a single data point after reboot, then nothing)
- CumulusMX debug logs show uploads with `wind_avg=NaN` and `wind_min=868.1`
- Affects stations using cloud-based APIs (Davis WeatherLink Cloud, and potentially other cloud-connected station types)
- Bug has been present since approximately **early 2025**

### Root Cause

WindGuru's `GetURL()` method in `WebUploadWindGuru.cs` computes wind statistics by scanning the `WindRecent[]` array for entries within the configured upload interval window. Cloud-connected stations (e.g., Davis WeatherLink Cloud with `data_structure_type=23`) only populate `WindRecent[]` via `DoWind()` every ~4 minutes when the cloud API returns new data.

With a WindGuru upload interval of 1–5 minutes, the interval window frequently contains **zero entries**, causing:

- `avgwind = totalwind / numvalues` → `0 / 0` → **`NaN`**
- `minwind` stays at the sentinel value `999` → converts to **`868.1 knots`**
- WindGuru rejects the upload or records zero values

The `WindRecent[]` array has 720 slots and is populated by `DoWind()` in `WeatherStation.cs`. For cloud stations like Davis WeatherLink Cloud (`DavisCloudStation.cs`), `DoWind()` is only called when the API returns fresh data — roughly every 4 minutes. Traditional directly-connected stations (serial, USB) call `DoWind()` every few seconds, so they rarely hit this bug.

### Investigation Timeline

1. Noticed WindGuru showing no data for over a year despite CumulusMX running normally
2. Manual WindGuru API test (curl) confirmed authentication and API were working
3. Enabled CumulusMX debug logging — discovered uploads were happening but with `wind_avg=NaN`
4. Traced through source: `MinuteChanged()` → `WebUploadWindGuru.DoUpdate()` → `GetURL()`
5. Identified the division-by-zero in `GetURL()` when `numvalues == 0`
6. Confirmed Davis Cloud station's `DoWind()` frequency (~4 min) vs WindGuru interval (1–5 min)
7. Verified fix: setting interval to 10 minutes produced valid data (`wind_avg=5.2`)
8. Implemented proper fix with fallback to current station values

### Fix Applied

**File**: `CumulusMX/ThirdParty/WebUploadWindGuru.cs` — `GetURL()` method

Changes:
1. **Sentinel value**: Changed `double minwind = 999` to `double minwind = double.MaxValue` to avoid bogus minimum wind values if the sentinel leaks through
2. **Division-by-zero guard**: Added `if (numvalues > 0)` check before computing the average
3. **Fallback for empty windows**: When no `WindRecent[]` entries are found in the interval window, fall back to `station.WindAverage` and `station.RecentMaxGust` — these are always kept current by the station driver regardless of `DoWind()` frequency

```csharp
double avgwind;
if (numvalues > 0)
{
    avgwind = totalwind / numvalues;
}
else
{
    cumulus.LogDebugMessage("WindGuru: No WindRecent entries in interval window, using current wind values");
    avgwind = station.WindAverage;
    maxwind = station.RecentMaxGust;
    minwind = avgwind;
}
```

### Workaround (without this fix)

Set the WindGuru upload interval to **10 minutes or higher** in CumulusMX settings. This gives cloud stations enough time to populate `WindRecent[]` entries within the window. However, this reduces upload frequency — the fix in this fork allows intervals as low as 1 minute.

---

## About CumulusMX

Cumulus MX is a weather station program that supports a wide range of weather stations and uploads data to various third-party weather services. It was originally written by Steve Loft and is now community-maintained.

A note from Steve when he released the code:
> "*[the code is]* Offered completely without support in the hope that it might be useful. The code is very badly structured due to the 'Frankenstein' way it was cobbled together from various places. Some of it is a machine translation of parts of Cumulus 1."

### Requirements

In order to function correctly the program needs the supporting distribution files and folders. These can be found in [the CumulusMX-DistributionFiles repo](https://github.com/cumulusmx/CumulusMX-DistributionFiles).

### Upstream Resources

- **Upstream repo**: [cumulusmx/CumulusMX](https://github.com/cumulusmx/CumulusMX)
- **Support forum**: [cumulus.hosiene.co.uk](https://cumulus.hosiene.co.uk/)
- **Change log**: [CHANGELOG.md](CHANGELOG.md)
- **Distribution files**: [CumulusMX-DistributionFiles](https://github.com/cumulusmx/CumulusMX-DistributionFiles)

### Building

CumulusMX is a .NET application. To build from source:

```bash
dotnet build CumulusMX.sln
```

Or to publish a self-contained release:

```bash
dotnet publish CumulusMX.sln -c Release
```

---

## License

CumulusMX is released under the [GNU GPL v3](https://www.gnu.org/licenses/gpl-3.0.en.html), consistent with the upstream project.
