description:
  Policy congestion tool (Alkmaar, GM0361). For one 6-digit postcode area (PC6) it computes an
  hourly energy profile for a full year (8760 h): gross and net electricity demand, rooftop PV
  generation, heat demand, and how that heat is supplied by gas boilers vs heat pumps.
  Policy lever: an electrification fraction (0.0-1.0) that moves part of the gas-supplied heat
  to heat pumps, so you can see the extra electricity load (grid congestion) per PC6.
  Method: annual consumption per address (Common Data Factory) is aggregated to PC6, enriched
  with energy-label heat intensities and climate normalization, split into residential /
  non-residential, then shaped into hours with NEDU standard load profiles (E1/E2 electricity,
  G1/G2 gas) and a PVGIS hourly PV profile (lat 52.63, lon 4.75, year 2019).
  Served as a FastAPI service: GET /simulate/{pc6}?electrification=<0..1>
  (behind Traefik at https://reformers01.ewi.tudelft.nl/policy-api/simulate/{pc6}).

inputs:
  PostcodeArea (pc6 code e.g. "1842EM", municipality code GM0361)
  ElectrificationScenario (electrification fraction 0.0-1.0 of gas heat switched to heat pumps)
  AddressEnergyData (annual electricity import kWh 2023, PV production/export kWh 2023, gas m3 2023,
    gas connections, floor area m2, building use (gebruiksdoel), construction year, energy label)
    -- static file from Common Data Factory
  StandardLoadProfiles (hourly fractions E1, E2 electricity and G1, G2 gas, NEDU 2026) -- static files
  SolarProfile (hourly PV yield kWh/kWp and air temperature(stream), from JRC PVGIS API)
  Assumptions (heat pump COP 3.0, boiler efficiency 0.92, PV self-consumption share 0.30,
    gas 9.77 kWh/m3, climate normalization factor 1.15, heat intensity per energy label)

outputs:
  PostcodeArea(
    electrical_demand_gross_kwh(stream, hourly),
    pv_generation_kwh(stream, hourly),
    electrical_demand_net_kwh(stream, hourly; negative = export to grid),
    heat_demand_kwh_th(stream, hourly),
    hp_electricity_input_kwh(stream, hourly),
    gas_input_kwh(stream, hourly))
  -- returned as a CSV (columns: timestamp + the six fields above); the API response is
     {"status": "success", "pc6": "<PC6>", "url": "/dashboard/processed/pc6_profile_<PC6>.csv"}
  PostcodeArea annual summary (pc6_annual_data.csv: annual heat demand, gas input, non-gas heat,
    current electrification share, residential share, profile weights E1/E2/G1/G2) -- all PC6s

links:
  Municipality-PostcodeArea (GM0361 Alkmaar contains PC6 areas; PC6 also carries buurtcode, wijkcode)
  PostcodeArea-Address (addresses / buildings aggregated into a PC6)
  PostcodeArea-ElectrificationScenario
  PostcodeArea-SolarProfile
  PostcodeArea-StandardLoadProfiles (weighted by residential / non-residential share)

key-internal-resources:
  policy-tool-backend/api.py                      (FastAPI entry point, /simulate/{pc6})
  policy-tool-backend/src/main.py                 (pipeline; CLI --pc6, --electrification, --full)
  policy-tool-backend/config/settings.yaml        (all physical assumptions and label intensities)
  policy-tool-backend/src/processing/engine.py    (hourly profile calculation)
  policy-tool-backend/src/processing/enrichment.py (heat demand + electrification scenario)
  policy-tool-backend/src/models/schemas.py       (output schema, PC6ProfileSchema)
  policy-tool-backend/data/raw/energiedata-match-gemeentecode=[GM0361]-csv.csv (CDF input data)
  policy-tool-backend/data/raw/slp/slp_electricity.csv, slp_gas.csv (load profiles)
  policy-tool-backend/data/processed/pc6_profile_1842EM.csv (example output)
  docker-compose.yml                              (service policy-tool-backend, port 8000, Traefik /policy-api)
