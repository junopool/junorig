# Junorig

Juno Cash miner based on XMRig with support for the `rx/juno` algorithm.

## Quick Start

### 1. Build

```bash
git clone https://github.com/junopool/junorig
cd junorig
mkdir build && cd build
cmake ..
make -j$(nproc)
```

### 2. Mine

```bash
./xmrig -o mine.junopool.org:3333 -u YOUR_JUNO_ADDRESS -a rx/juno
```

Replace `YOUR_JUNO_ADDRESS` with your Juno Cash **unified address** (starts with `j1...`).

## Options

| Option | Description |
|--------|-------------|
| `-o` | Pool address (mine.junopool.org:3333) |
| `-u` | Your Juno Cash wallet address |
| `-a` | Algorithm (rx/juno) |
| `-t` | Number of CPU threads (default: all) |

## Example

```bash
# Use 4 threads
./xmrig -o mine.junopool.org:3333 -u j1YourAddress... -a rx/juno -t 4

# Use all threads
./xmrig -o mine.junopool.org:3333 -u j1YourAddress... -a rx/juno
```

## Pool

- Website: https://junopool.org
- Stratum: mine.junopool.org:3333

## Credits

Based on [XMRig](https://github.com/xmrig/xmrig) by xmrig team.
