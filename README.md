# example-proj-cfg

Designer-authored game config for `example-proj`. Excel sheets live under
[`designer_cfg/`](./designer_cfg/); running `npm run build` compiles them
into an LMDB database under `dist/` via
[`@dogsvr/cfg-luban-cli`](../cfg-luban/cfg-luban-cli/README.md).

## Layout

```
example-proj-cfg/
├── designer_cfg/        # Excel source (git-tracked)
│   ├── Datas/*.xlsx
│   ├── Defines/builtin.xml
│   └── luban.conf
├── tools/               # Toolchain binaries
│   ├── Luban/Luban.dll  #   Luban release unpacked here
│   └── flatc            #   FlatBuffers compiler binary
├── package.json         # scripts.build wires toolchain paths; devDep: @dogsvr/cfg-luban-cli
└── dist/                # build artifacts
```

Both `tools/` and `dist/` are gitignored; `tools/` is a per-developer one-time setup, `dist/` is rebuilt by `npm run build`.

## Prerequisites

> **Platform note**: the `tools/` layout described here assumes **Linux x86-64**, and this is the only environment the project has been tested on. `Luban.dll` itself is cross-platform (managed by `dotnet`), but `flatc` is a native binary — macOS / Windows developers must substitute an appropriate `flatc` build from the [FlatBuffers releases](https://github.com/google/flatbuffers/releases) (`flatc.exe` on Windows, Mach-O binary on macOS). Other OS/arch combos may or may not work — file an issue if something breaks.

System-level: `dotnet` runtime (for Luban.dll), `python3 + openpyxl` (for reading `__tables__.xlsx`). See also the [cfg-luban-cli prerequisites](../cfg-luban/cfg-luban-cli/README.md#prerequisites).

Repo-local:

- `tools/Luban/Luban.dll` — unpack a [Luban release](https://github.com/focus-creative-games/luban/releases) into `tools/Luban/`
- `tools/flatc` — a [flatc binary](https://github.com/google/flatbuffers/releases) (≥ 23.x) matching your OS/arch (see platform note above), `chmod +x`
- `npm install` — pulls `@dogsvr/cfg-luban-cli`

## Generate

```sh
# default: uses bundled tools/Luban/Luban.dll + tools/flatc, writes to dist/
npm run build

# override toolchain paths if needed
npm run build -- --luban-dll /opt/luban/Luban.dll --flatc /opt/flatc

# pass any additional cfg-luban-cli flag through (-- is required)
npm run build -- --target client        # target defaults to "all"
```

> **The `--` separator**: everything after `--` is passed verbatim to the underlying `cfg-luban-cli build` invocation. Because the package.json script already specifies `--luban-dll`, `--flatc`, `--designer`, `--output`, and your override comes later, the CLI's last-wins flag parsing picks your value up.

> **Note on directory name**: this project writes to `dist/` (consistent with the npm packages in this org). The `cfg-luban-cli` tool itself still defaults to `./generated` when `--output` is omitted; we override it explicitly in `scripts.build`.

This produces:

```
dist/
├── fbs/             # .fbs schema
├── json/            # sorted JSON data
├── bin/             # per-table FlatBuffers binaries
├── ts/              # TypeScript accessors
├── table_keys.json  # primary-key metadata
└── db/              # LMDB (data.mdb + lock.mdb)
```

For a breakdown of the 5 pipeline stages (Luban → extract-keys → sort-json → flatc → lmdb) and how to invoke them individually (useful when debugging a single step), see [cfg-luban-cli usage](../cfg-luban/cfg-luban-cli/README.md#usage).

## Consume from `example-proj`

The runtime (`@dogsvr/cfg-luban`) needs these pieces from `dist/`:

- `dist/db/` — the LMDB directory (passed to `openCfgDb({ dbPath })`)
- `dist/table_keys.json` — primary-key metadata (passed to `openCfgDb({ tableKeysPath })`)
- `dist/ts/cfg` — the flatc barrel re-exporting every `TbXxx` class; either pass the whole module via `cfgModule` (Style A, shown below) or import individual classes and register them manually (Style B, see [`@dogsvr/cfg-luban`](../cfg-luban/cfg-luban/README.md#usage))

Wire paths through `worker_thread_config.json` so hosts can point server processes at different config builds without recompiling:

```jsonc
// example-proj/config/worker_thread_config.json
{
    "cfgDbPath":     "../example-proj-cfg/dist/db",
    "tableKeysPath": "../example-proj-cfg/dist/table_keys.json"
}
```

```ts
// example-proj/src/.../worker_logic.ts
import * as dogsvr from '@dogsvr/dogsvr/worker_thread';
import { openCfgDb } from '@dogsvr/cfg-luban';
// Barrel import — adjust the relative path to match your server's file depth.
import * as cfgModule from '<relative-path>/example-proj-cfg/dist/ts/cfg';

dogsvr.workerReady(async () => {
    dogsvr.loadWorkerThreadConfig();
    const cfg = dogsvr.getThreadConfig<{ cfgDbPath: string; tableKeysPath: string }>();

    openCfgDb({
        dbPath: cfg.cfgDbPath,
        tableKeysPath: cfg.tableKeysPath,
        cfgModule,
    });
});
```

See the runtime API surface (`getCfgRow`, `getCfgRowList`, `forEachCfgRow`) in [@dogsvr/cfg-luban](../cfg-luban/cfg-luban/README.md).
