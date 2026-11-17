# Dragonfly – private butterfly network

This guide starts a private Filecoin network from a stock `butterflynet` Lotus build with 512MiB sectors. It runs on a single host with its own genesis and one genesis miner (`t01000`), which produces the blocks.

**Why it stays private:**

- **Own genesis.** We generate it from our own preseal. The genesis CID and the network name, which sets the gossip topics, differ from public butterfly's, so foreign peers are rejected.
- **No public bootstrap.** The daemon runs with `--bootstrap=false`.

Network params, actor bundle and sector sizes all stay stock butterflynet: 30s blocks, a 150-epoch precommit delay and 2GiB consensus minimum power.

## Prerequisites

- **Lotus butterflynet binaries.** Build `lotus`, `lotus-miner` and `lotus-seed` with `make butterflynet` (not `make all`, which builds mainnet), following the [Lotus build docs](https://lotus.filecoin.io/lotus/install/linux/). Every machine that joins later must run the exact same tag.

  ```sh
  lotus --version         # lotus version 1.38.x+butterflynet+git.…
  ```

- `jq` and `curl`.
- **Proof params** for 512MiB sectors, which the genesis miner needs for PoSt:

  ```sh
  lotus fetch-params 512MiB
  ```

The commands use Lotus's default repos (`~/.lotus`, `~/.lotusminer`) and these values:

| Value | Used as |
|---|---|
| Network name | `dragonfly` |
| Preseal sector count | `4` (512MiB each, minimum 4) |

## 1 · Preseal the genesis miner's sectors

Presealing creates the genesis miner's sectors before the chain exists. For each sector, `lotus-seed` runs AddPiece on random data, then PreCommit1, PreCommit2 and FinalizeSector, and drops the unsealed copy. There is no Commit1/Commit2 SNARK step: the genesis simply trusts the resulting CommR. It also writes one market deal per sector.

```sh
lotus-seed --sector-dir=/opt/filecoin/build/fly-genesis pre-seal \
  --miner-addr t01000 \
  --sector-size 512MiB \
  --num-sectors 4
```

> [!NOTE]
> `--sector-dir` is a global flag, so it goes before `pre-seal`. Never use `--fake-sectors`: butterfly verifies real Winning and Window PoSt, so the genesis miner must hold real replicas.

| Output in `/opt/filecoin/build/fly-genesis/` | What it is |
|---|---|
| `sealed/s-t01000-N` | The sealed replicas (512MiB each) |
| `cache/s-t01000-N/` | `p_aux`, `t_aux` and `tree-r-last`, needed for PoSt |
| `sectorstore.json` | Marks the directory as a Lotus storage path |
| `pre-seal-t01000.json` | Miner manifest: owner/worker, sector size, CommR/CommD and the deal per sector |
| `pre-seal-t01000.key` | Hex-encoded BLS key used as owner and worker. **This is your faucet.** |

> [!WARNING]
> **This directory is permanent storage.** The genesis miner proves these sectors for the life of the network, so never move or delete it while the network runs.

> [!WARNING]
> **You need at least 4 sectors** (4×512MiB = 2GiB). Butterfly's actor bundle sets the consensus minimum to 2GiB, and the power actor only adds a miner to `TotalRawBytePower` once it reaches that minimum. With fewer sectors, `lotus-seed genesis car` fails with `TotalRawBytePower (0) doesn't match previously calculated rawPow (…)`.

Cost: 512MiB SDR runs on CPU in minutes and needs a few GiB of temporary cache. Sectors are sealed one after another.

Check:

```sh
ls /opt/filecoin/build/fly-genesis
jq '."t01000".Sectors | length' /opt/filecoin/build/fly-genesis/pre-seal-t01000.json   # 4
```

## 2 · Create the genesis CAR

Public butterfly's genesis is compiled into the binary (`build/genesis/butterflynet.car.zst`). We can't reuse it, because its pre-sealed sectors and keys belong to the Lotus team and nobody on our network could produce blocks. We build our own from the preseal instead.

> [!WARNING]
> **Do steps 2–4 in one go.** The genesis timestamp is the moment you run `genesis car`. If the miner starts hours later, the chain begins with a long run of null rounds.

```sh
mkdir -p /opt/filecoin/build/fly-genesis && cd /opt/filecoin/build/fly-genesis
lotus-seed genesis new --network-name=dragonfly genesis.json
lotus-seed genesis add-miner genesis.json /opt/filecoin/build/fly-genesis/pre-seal-t01000.json
lotus-seed genesis car -o dragonfly.car genesis.json
```

- `add-miner` gives the preseal owner key 50,000,000 FIL (change it with `--balance`). The miner id must be `t01000` + its index in the template.
- Leaving out `--timestamp` on `genesis new` makes `genesis car` use the current time.

Check:

```sh
jq '{NetworkName, NetworkVersion, miners: (.Miners|length)}' genesis.json
ls -la dragonfly.car
```

## 3 · Host daemon: first boot

**Terminal A:**

```sh
lotus daemon --genesis=/opt/filecoin/build/fly-genesis/dragonfly.car --bootstrap=false
```

**Terminal B:**

```sh
lotus wait-api
lotus wallet import --as-default /opt/filecoin/build/fly-genesis/pre-seal-t01000.key
```

Check:

```sh
lotus wallet list                 # the preseal address, ~50M FIL, marked default
curl -s -X POST -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","method":"Filecoin.StateNetworkName","params":[],"id":1}' \
  http://127.0.0.1:1234/rpc/v1 | jq -r .result      # dragonfly
```

## 4 · Genesis miner t01000

In Terminal B:

```sh
lotus-miner init --genesis-miner --actor=t01000 \
  --sector-size=512MiB \
  --pre-sealed-sectors=/opt/filecoin/build/fly-genesis \
  --pre-sealed-metadata=/opt/filecoin/build/fly-genesis/pre-seal-t01000.json \
  --nosync
lotus-miner run --nosync
```

Wait a minute, then in Terminal C check that the height increases about every 30s:

```sh
lotus chain head; sleep 65; lotus chain head
lotus chain list --count 3
lotus-miner info
lotus-miner proving info
lotus state power t01000
```

> [!NOTE]
> **Minimum power:** the genesis miner needs at least 2GiB raw power (4×512MiB), or genesis creation fails (see step 1). Keep WindowPoSt healthy (`lotus-miner proving info`), because faulted sectors reduce power.

---

Flags and behaviour were checked against the Lotus v1.38.x source: `cmd/lotus-seed/main.go`, `cmd/lotus-seed/genesis.go`, `cli/lotus/daemon.go`, `cli/miner/init.go` and `build/buildconstants/params_butterfly.go`.

References:
[ribasushi/ltsh](https://github.com/ribasushi/ltsh) ·
[builtin-actors runtime/build.rs](https://github.com/filecoin-project/builtin-actors/blob/master/runtime/build.rs) ·
[Lotus butterfly docs](https://lotus.filecoin.io/kb/butterfly-network/)
