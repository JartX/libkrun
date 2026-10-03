# macOS balloon reclaim (#794 + fix)

This branch is libkrun **main at `e5d277cd`** (the same point as the v1.19.4
release, 2026-07-03) plus:

- the two commits of [libkrun/libkrun#794](https://github.com/libkrun/libkrun/pull/794)
  by **@BinSquare** (BinBin He): stage-2 unmap of free-page-reported ranges and
  refault-remap on the next guest access;
- one follow-up commit that makes the reclaimed memory actually go back to macOS
  and fixes a refault race. See the commit message for the details.

All the design credit for the unmap/refault approach goes to @BinSquare; this
branch only changes how the host pages are released and how a racing refault
is handled.

## Why the follow-up

With #794 alone, `phys_footprint` drops after the guest frees memory, but the
host does not get it back: the pages are marked reusable, `Pages free` does not
move, and under host memory pressure macOS compresses other memory instead of
discarding them. Replacing the range with a fresh anonymous mapping
(`mmap(MAP_FIXED)`) after `hv_vm_unmap` frees the pages outright.

## Results

Measured with the same change **backported to v1.19.4**: krunkit 1.3.2 links
the 1.x library and cannot load this 2.0.0-dev build (`krun_add_virtiofs4` and
`krun_set_console_output` no longer exist). On this base the branch builds
cleanly but has not been run.

macOS 26.5.2, krunkit 1.3.2, Colima `--vm-type krunkit`, 3 GiB guest,
1.5 GiB tmpfs alloc/free:

| | upstream 1.19.4 | this branch (`KRUN_BALLOON_RECLAIM=1`) |
|---|---|---|
| host free memory after release | unchanged | +1.3 – 1.5 GiB immediately |
| krunkit footprint after release | ~2.8 GiB (stays) | ~1.2 GiB |
| data integrity under host pressure (tmpfs + disk, sha256) | ok | ok |
| 4 vCPU, 4 × 12 write/verify cycles of 250 MiB | 0 failures, 60 s | 0 failures, 60 s |

Without `KRUN_BALLOON_RECLAIM=1` the behaviour is the same as upstream.

Only memory the guest reports as free is returned; the guest page cache stays
until the guest drops it.

## Build and use (macOS, arm64)

```sh
brew install rust lld libkrunfw dtc
export LIBCLANG_PATH=/Library/Developer/CommandLineTools/usr/lib
export RUSTFLAGS="-C link-arg=-Wl,-rpath,$LIBCLANG_PATH"
make BLK=1 NET=1 TIMESYNC=1            # GPU=1 not tested
# produces libkrun.2.dylib: needs a client built against the 2.x API.
# For krunkit 1.3.x, apply the three commits on top of v1.19.4 instead and
# copy libkrun.1.19.4.dylib as libkrun.1.dylib:
mkdir -p ~/libkrun-reclaim && cp target/release/libkrun.1.19.4.dylib ~/libkrun-reclaim/
ln -sf libkrun.1.19.4.dylib ~/libkrun-reclaim/libkrun.1.dylib

DYLD_LIBRARY_PATH=~/libkrun-reclaim KRUN_BALLOON_RECLAIM=1 \
  colima start -p krun --vm-type krunkit
```

The guest kernel needs `CONFIG_VIRTIO_BALLOON` and `CONFIG_PAGE_REPORTING`
(the Colima/Lima images have both).
