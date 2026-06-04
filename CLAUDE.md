# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this repository.

## Build

First-time setup — pull all submodules (curlpp, googletest, here-olp-sdk, rapidjson, smasher):
```bash
git submodule update --init --recursive
```

Install system dependencies (Ubuntu):
```bash
sudo apt-get install cmake libgmock-dev libgtest-dev rapidjson-dev doxygen libcurl4-openssl-dev protobuf-compiler libprotobuf-dev libboost-all-dev
```

On macOS, equivalents via Homebrew: `boost`, `protobuf`, `curl`, `rapidjson`, `googletest`, `doxygen`.

Build (outputs to `./build/`):
```bash
./scripts/build.sh -DCMAKE_BUILD_TYPE=Debug
```

The build script runs `cmake .. <args>` then `cmake --build . -j 10` from `./build/`. Pass any extra CMake flags as arguments (e.g. `-DCMAKE_BUILD_TYPE=Release`).

## Tests

Run all tests (requires map data and mock server — see below):
```bash
./build/dbs-map-api/tests/GoogleTestRunner
# or equivalently
./scripts/run_tests.sh
```

Run a single test by name:
```bash
./build/dbs-map-api/tests/GoogleTestRunner --gtest_filter=MapService.GetRcaTopology
```

### Mock server

Tests that exercise the download/OTA path (`download/` test files) require a local mock server on port 8080. Run it with Docker:
```bash
docker run -p 8080:8080 ghcr.io/vladimir-berlin/dbs-onboard-map-api-Public/dbs-map-mock-server:latest
```

Or directly with Python (from `dbs-map-mock-server/`):
```bash
cd dbs-map-mock-server && python dbs_map_mock_server.py
```

Tests configure themselves via `GetDBClientForCISettings()` which points to `localhost:8080`.

## Code coverage

```bash
./scripts/code_coverage.sh   # outputs build/coverage/coverage.xml
```

Coverage is enabled by building with `-DENABLE_TESTS=ON` (the default), which adds `-O0 -g --coverage` flags to the `dbs-map-api` target.

## Code style

Pre-commit hooks enforce formatting and license headers. After cloning:
```bash
pre-commit install
pre-commit run --all-files   # run manually
```

**C++ formatting** — Allman brace style, 4-space indent, no column limit (`.clang-format`):
```bash
clang-format -i <file>
```

**License headers** — every file must have an SPDX license header. C++ files use Apache-2.0; build/config files use CC0-1.0.

## Architecture

### Public API surface
`MapService` (`dbs-map-api/include/dbs-map-api/MapService.h`) is the single entry point. It accepts a `MapServiceConfig` struct and exposes:
- `GetLayersForRectangle` / `GetLayersForTiles` / `GetLayersForCorridor` → `ConsolidatedLayers::Ptr`
- `GetRcaTopology` / `GetLandmarks` / `GetZones` — individual layer reads by tile IDs
- `UpdateMap` / `UpdateLocalMap` / `CleanLocalCache` — OTA update path

Geographic types (`GeoCoordinates`, `GeoRectangle`, `Polyline`, etc.) are Boost.Geometry aliases defined in `CommonTypes.h`. Tile identifiers are `PartitionId` (a `std::string`).

### Pimpl + decoder pipeline
`MapService` delegates everything to `MapServiceImpl` (pimpl). `MapServiceImpl` holds three decoder instances and a `MapUpdater`:

```
MapService → MapServiceImpl
               ├── BaseDecoder<RcaTopology::Ptr>         (RcaTopologyDecoder)
               ├── BaseDecoder<vector<Landmark::Ptr>>    (LandmarkDecoder)
               ├── BaseDecoder<vector<Zone::Ptr>>        (ZoneDecoder)
               ├── MapFileSystem                          (local tile storage)
               └── MapUpdater                             (OTA download)
```

`BaseDecoder<T>` (`src/decoder/BaseDecoder.h`) is a pure-virtual template. Each decoder reads raw protobuf bytes from `MapFileSystem` and deserialises them into domain model objects.

Area-based queries (rectangle, corridor) convert to tile IDs first via `src/utils/Geo.cpp`, then delegate to the tile-ID variants.

### Download / OTA stack
`MapUpdater` calls `LayerClient` → `CatalogClient` → `HttpClient` (a curlpp wrapper). The HERE OLP SDK (`externals/here-olp-sdk`) provides `olp-cpp-sdk-core` which is linked into the main library. Custom HTTP settings (host, headers, TLS) are configured through `ClientSettings`.

### Map data layout
Map data lives on disk under `map_local_path_` (default: set by `GetDefaultConfig()`). During tests the build system copies `dbs-map-api/hdmap/` to `build/hdmap/`. The catalog is `validate.s4r2.oss.4` and contains four layers: `rca-topology`, `rca-centerline`, `landmarks`, `risk-assessment-zones`. Tiles are keyed by HERE tile IDs at zoom level 12.

### Protobuf schema
Proto sources are under `dbs-map-api/proto/db-sensors4rail-phase2/`. The CMake helper `CompileProtobufSchema.cmake` compiles each schema subdirectory into a separate library and adds the generated headers to the include path automatically.

### External dependencies as submodules
| Submodule | Purpose |
|---|---|
| `externals/curlpp` | C++ RAII wrapper around libcurl |
| `externals/googletest` | Google Test + Mock (used when `ENABLE_TESTS=ON`) |
| `externals/here-olp-sdk` | HERE OLP SDK core (HTTP, auth, catalog access) |
| `externals/rapidjson` | JSON parsing for catalog/layer metadata |
| `externals/smasher` | MurmurHash3 (`MurmurHash3.cpp` compiled directly into the main target) |

## CI

GitHub Actions (`.github/workflows/ci.yml`) runs on push to `main` and all PRs: build → test → coverage upload to Codecov. The mock server image (`dbs-map-mock-server/Dockerfile`, Python/Flask) is published to GHCR and pulled during CI for OTA tests.
