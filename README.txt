This repository does the required climate data edits to be used in the Canadian forest fire spread model WISE. Hourly values are transformed from netcdf files to a singular .txt file for each of the 3 test areas in Finland. The required job.fgmj files for running WISE are then edited based on the temporal extent of the weather files and ignition locations.

How to run:
start and end times will be defined by the workflow, e.g.:
python run_wildfires_wise.py -year_start 2021 -year_end 2021 -month_start 06 -month_end 06 -day_start 24 -day_end 30 -expid a0c1

The run_wildfires_wise.py will take the start and end dates as input and select the correct netcdf files based on this. Next the required input and output directories will be binded to the singularity container, after which it will be run from the run_wildfires_wise.py script. In the container the run_wise.py will first select the correct netcdf files and combine them into a singular file. The ncdf_edits_multiarea.py will take this combined ncdf file and select the closest cell to each ignition location, make the necessary transformation and calculations and create the weather files (weather.txt) for each of the three scenarios. Next the modify_fgmj.py will make the required alterations to the job.fgmj files which WISE uses for model run setting. Finally, the run_wildfires_wise.py will run the WISE model for the three different scenarios. Application output consists of raster, vector and text files. The daily run results are divided into separate folders based on the test area, run start time and the date used for the model run, e.g. scen_lieksa_2024-03-26_22_31_2000-08-26 contains results for the Lieksa area for the date 26.8.2000 and the application was ran on 26.3.2024 at 22:31. The text (.txt) file summary contains information about the used input files, model run setting used and ignition locations. The vector (.kml) file perim contains fire propagation polygons at an hourly temporal resolution. The raster files (.tif) consist of max_crown_fraction_burned (maximum crown fraction burned in each cell, %), max_flame_length (maximum flame lenght in each cell, meters), max_intensity (maximum fire intensity in each cell, kilowats), surface_fuel_consumed (surface fuel consumption in each cell, kg), crown_fuel_consumed (crown fuel consumption in each cell, kg) and total_fuel_consumption (total fuel consumption from both canopy and surface, kg) files. The spatial resolution of the raster files is 16 x 16 m. These files are created from the last time step of the fire spread to include all values from the model run. 

Data request file:
Data request file is the request.yml file

Combined daily FWI and WISE runner:
Run the installed package in the WISE container, for example:

python -m wise.run_wildfires --year 2022 --month 6 --day 6 \
  --input-dir /input_data --output-dir /fwi_output

The input directory must contain the hourly weather required by both apps,
including adjacent-day hours needed by FWI. The FWI output directory must
persist across days; the WISE output directory must be mounted at
/wise_output. Use --extra-input-dir for additional FWI weather directories.
FWI writes prepared _LNOON_inputs.nc files in the input directory, and WISE
writes combined_ncdf.nc there, so the input directory must be writable.
Historical FWI output is stored in --output-dir; projection output is stored
in its projections/<experiment> subdirectory, with historical handover data
under historical_boundary/<experiment>.
The runner calculates daily regular FWI, then checks the nearest grid cells
for all nine WISE ignition areas. If any FWI is at least 21.3, it runs all
WISE scenarios for 08:00-20:00 local time, using the previous calendar day's
component file for starting codes. Missing previous components retain WISE's
warning and template fallback; invalid components may skip individual areas.
WISE records individual scenario failures in its status JSON; launching all
scenarios does not guarantee that all of them complete successfully.
The runner fails when the requested daily FWI file or a valid threshold value
is missing. In a fresh FWI integration the first date is normally skipped,
so initialize using sufficient prior weather or provide an appropriate restart.
FWI initialization uses zero moisture codes and requires spin-up. A daily
invocation is required: --monthly-chunk, --prepare-only and
--no-write-components are not supported by the combined runner.

NetCDF georeferencing:
The TIFF outputs are reprojected to a regular WGS84 (EPSG:4326) grid.
Latitude and longitude describe pixel centres and include CF coordinate
attributes. The spatial_ref grid mapping includes CF ellipsoid parameters,
WGS84 WKT and the GDAL GeoTransform. Scenario, source CRS and resampling
method are recorded in the global attributes.
Declared source NoData values are preserved as the NetCDF variable's
_FillValue. Valid zero values are not reclassified as missing. If a source
TIFF has no NoData value, uncovered destination pixels are not currently
distinguished from valid zeros.
Burn grid is dimensionless (units "1"), with flags 0 = non_burned and
1 = burned. Maximum flame length uses "m"; maximum fire intensity uses
"kW m-1" (kW/m). Crown fuel consumed, surface fuel consumed and total
fuel consumption use "kg m-2" (kg/m2). Maximum crown fraction burned is
dimensionless (units "1") on a 0-1 scale, not a percentage.
All seven raster variables have dimensions (time, lat, lon), with one time
value per file. The time coordinate labels the configured simulation start
date at 00:00, not the file creation time, raster export time or an hourly
snapshot. For multi-day runs it labels the start date of the whole run.
It has standard_name "time" and axis "T", and is stored as double-precision
days since 1990-01-01 00:00:00 with calendar "standard" and no _FillValue.
The date is taken from the run arguments, not inferred from the filename.
Consumers expecting a two-dimensional raster array should select time=0.
Changes to src/wise/run_wise.py must be installed in the runtime container,
or explicitly bind-mounted over its installed module, before they affect runs.
Existing NetCDF files are not modified by updating the script.

FWI initialization (optional):
Pass --fwi-dir /fwi_files to select YYYY_MM_DD_fwi_comp.nc from that
directory, where YYYY_MM_DD is the day before the simulation start date.
For example, a 2016-08-21 trigger/run selects 2016_08_20_fwi_comp.nc.
Selection is exact (no recursive search, nearest date or *_fwi.nc fallback).
The directory must belong to the same model/experiment as the weather;
it must expose the previous day's component file, not only today's FWI.
The selected file is still validated for its internal date and FFMC/DMC/DC
contents. Missing files retain the warning/template-fallback policy below.
Alternatively, retain the single-file interface:
Pass --fwi-file /fwi_files/YYYY_MM_DD_fwi_comp.nc to run_wise.py to replace
the daily FFMC, DMC and DC starting codes in all nine generated job.fgmj
files. The file must contain exactly one time entry matching the calendar
day before the simulation start date (local-date label; noon timestamps are
accepted). For example, a July 6 run requires July 5 FWI codes. For multi-day
runs only the initial state is supplied; it is not reset every day.
FFMC, DMC and DC must have time, lat and lon dimensions on a regular
latitude/longitude grid. Each ignition uses the nearest latitude and
longitude cell, with circular longitude distance supporting both 0-360
and -180-180 conventions, including Portuguese ignition coordinates.
The selected file, date, grid cell and values are printed to the run log.
The status report records the FWI file path, simulation_date,
required_fwi_date and date_rule "previous_day"; generated jobs contain the
actual starting values. FWI model/experiment attributes are not copied.
--fwi-dir and --fwi-file are mutually exclusive. The Python interface accepts
fwi_dir=... or fwi_file=..., respectively. The workflow wrapper must parse
and forward the selected keyword to the installed wise.run_wise.run_wise.
Status JSON records fwi_directory and the resolved fwi_file; initialization
diagnostics include selected_file. No supplied FWI input retains templates.
FFMC must be finite and within 0-101; DMC and DC must be finite and
nonnegative. A missing supplied file prints a WARNING and all areas run
with template starting values. Missing/invalid selected index values print
a WARNING identifying the file, date, ignition, grid cell and affected codes;
only the affected areas are skipped (no job creation, model run or output
conversion). Other areas continue. No values are clipped or substituted.
The status JSON includes fwi_initialization with warnings, fallback status,
selected valid codes and per-area diagnostics. Skipped areas have status
fwi_skipped and contribute to the overall partial/failed output status.
An output status of success can therefore still include a template-fallback
warning: inspect fwi_initialization for scientific provenance.
Missing/ambiguous dates, missing variables, malformed files and invalid grid
coordinates still fail input preparation rather than using fallback values.
Without --fwi-file the existing template values are retained and this is
reported in the log.
Only condition.startingCodes.ffmc/dmc/dc.value are updated. Starting
precipitation and hourly FFMC settings (hffmc, hffmcTime, hffmcMethod)
are unchanged. WISE treats startingCodes as yesterday's daily codes,
then calculates the current day's codes from the weather stream. Same-day
FWI codes must not be used as yesterday's starting state.
With the current LAWSON method, hourly FFMC is derived from previous/current
daily FFMC and weather; the separate hffmc seed is not used by that
calculation path. This does not apply generally to VAN_WAGNER, which can
use an explicit hourly seed and time. The template's printed hourly FFMC
94 is therefore retained, not replaced with the daily FFMC.
FWI files use solar-local-noon state while WISE computes at noon local
standard time; correcting the date does not align these time conventions.
Starting precipitation is also unchanged (zero); supplying previous-day
afternoon rain remains separate work.
Availability and synchronization belong to the caller;
the file must be complete before WISE input preparation begins.

For the interactive container setup, add a read-only input bind:
--bind fwi_files:/fwi_files:ro
Then append this argument to the existing Python run command:
--fwi-dir /fwi_files
Or pass the exact file instead:
--fwi-file /fwi_files/2032_08_07_fwi_comp.nc
This example initializes a 2032-08-08 run. January 1 requires December 31
of the previous year. If no preceding-day FWI state exists, the missing-file
warning/template fallback policy still applies; this is not FWI-driven
initialization and must be distinguished in downstream use.

Triggered daily runs use the trigger date as both start and end date.
The input weather directory must contain the existing OPA filenames for
2t, 2d, 10u, 10v and avg_tprate, with UTC hourly coverage for that date.
Directory arguments need no trailing slash. combined_ncdf.nc is still
written into the weather directory: writable access and isolated work
directories are required until workflow workspace handling is agreed.
All nine scenarios still run; threshold detection is external to WISE.

Initialization source references (WISE Weather Module, pre-March-2023 revision):
https://github.com/WISE-Developers/WISE_Weather_Module/blob/191ad08b02393d1ecaaaa04a84f5a4ca4e4de091/cpp/WeatherStream.cpp#L679-L698
https://github.com/WISE-Developers/WISE_Weather_Module/blob/191ad08b02393d1ecaaaa04a84f5a4ca4e4de091/cpp/DayCondition.cpp#L129-L235

Weather and simulation timezones:
Climate input hours are interpreted as UTC. Each Finnish/Portuguese job
selects its timezone using Europe/Helsinki or Europe/Lisbon rules at 10:00
local on the simulation start date. A simple summer/winter selection maps
that result to a WISE timezone ID/code. Historical Portuguese standard-time
changes are included; future rules follow the installed timezone database.
Python >=3.9 and timezone data (system zoneinfo or the tzdata dependency)
are required. The existing wildfires_wise_0.2.0.sif lacks timezone data:
add this read-only bind to the existing singularity shell/exec command:
--bind /usr/share/zoneinfo:/usr/share/zoneinfo:ro
No remote timezone service is used.

Finland uses EET/65544 (UTC+2) or EEST/131081 (UTC+3).
Portugal normally uses UTC/65553 (UTC+0) or WEST/131089 (UTC+1).
UTC is intentional: Builder advertises WET/65682, but the installed WISE
1.0.6-beta native library does not recognize it. Historical Portuguese
UTC+1 standard time uses CET/65541; UTC+2 summer time uses CEST/131079.
Every existing timezone ID/code in the generated job is updated, including
the project timezoneIndex. Templates themselves are not modified.

The complete UTC weather timestamp is converted to the job's fixed offset
before writing HOURLY (date) and HOUR. Values and ordering are unchanged;
late UTC hours can appear on the next local date. An ordinary Finnish
summer UTC day therefore spans 03:00 to 02:00 the following date, not 00-23.
The weather stream/project start is the first weather observation.
Scenario start and ignition are 10:00 local on the simulation start date;
scenario end is 21:00 local on the final input UTC date. All seven raster
exports and the vector perimeter end use that same scenario end.
Previous-day FWI selection stays anchored to the simulation start date,
not the converted weather dates. Hourly FFMC and initial rain are unchanged.

The selected WISE IDs apply a fixed offset, not automatic clock changes.
On a transition day, all weather rows use the offset selected at ignition:
UTC instants are preserved with no missing/repeated rows, while clock
labels before/after the transition outside the scenario can differ from
civil time. Such weather coverage prints an explicit WARNING.
A multi-day scenario crossing an offset change is rejected with a request
to split the run into separate dates; it is not silently mistimed.
Weather timestamps must be consecutive whole UTC hours and cover the
requested scenario. Invalid coverage is an input-preparation error.

Failure handling:
WISE area failures and timeouts do not stop other areas. Even if all nine
areas fail, the script records the failed date and returns normally so the
caller can continue other dates/applications. Exit code zero therefore does
not guarantee complete scientific output.
Each invocation that reaches model execution writes a uniquely named
DATE_RUNID_status.json and a wise_logs_RUNID directory in the output directory.
The JSON report is atomically updated after each area and conversion. It
records per-area execution status, return code, stdout/stderr log paths,
missing outputs and per-variable conversion results. Overall output status
is success (nine complete areas), partial (one to eight), or failed (none).
The report remains running if execution is interrupted before finalization.
Missing required rasters skip the entire affected scenario; corrupt rasters
fail individually while remaining conversions continue.
The per-area timeout is 12000 seconds (3 hours 20 minutes); messages and
reports use the configured value. Keeping a finite limit prevents a hung
model from blocking later areas; select a limit compatible with job wall time.
Input preparation, report/log persistence and restart-file failures are not
intentionally suppressed. Container/scheduler termination must still be
handled by the external workflow.
