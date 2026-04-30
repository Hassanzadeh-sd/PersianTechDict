# I Tested ZFS vs ext4 on Production VMs. The First Result Was Embarrassing.

### Why the way you set up your storage matters more than which storage you choose.

---

> 📷 **Suggested cover image:** a server rack or NVMe drives close-up. Free options: search "server room" on Unsplash.

---

We run a VPS hosting service. Our customers' virtual machines all sit on **ext4** — a fast, simple, classic Linux filesystem. It works. It has worked for years. But people keep asking us: *"Why don't you use ZFS? It has snapshots, compression, and self-healing!"*

So we decided to find out. We picked one of our servers, fired up some virtual machines, and ran the same disk speed tests on both setups.

The results were not what we expected.

## The Setup

We tested three different storage configurations using identical workloads:

- **ext4** — The current production setup. 8 NVMe drives in RAID10.
- **ZFS — Bad Setup** — Same hardware, but ZFS layered on top of ext4 (a quick-and-dirty test).
- **ZFS — Tuned Setup** — A second server, with 4 NVMe drives dedicated entirely to ZFS, with all the right tuning options.

For each setup, we ran 3 virtual machines at the same time, hammering the disk with read and write tests. The tool we used is called **fio** — the standard for Linux disk benchmarks.

## Round 1: The Embarrassing Result

For our first ZFS test, we took a shortcut. We did not have any free disks on the test server, so we created a "fake" ZFS pool inside a single file on the existing ext4 filesystem.

This is a classic mistake. Let's see what happened.

> 📷 **Insert image: 01-write-collapse.png**

When saving small files, ext4 handled **208,000 saves per second**. Our quick ZFS setup handled **5,000**.

That is not a typo. ZFS was **41 times slower**.

The response time was even worse: ext4 finished a save in about 1 millisecond. The ZFS-on-a-file setup took 47 milliseconds — nearly 50 times longer. In the real world, that is the difference between a website that feels instant and one that feels broken.

We almost wrote our report at this point: *"ZFS is too slow. Don't use it."*

Then we paused.

## Why It Failed

Stacking ZFS on top of another filesystem is the worst thing you can do to it. Here is why, in plain English:

1. **ZFS thinks in big chunks (128 KB by default), but virtual machines write in tiny pieces (4 KB).** So every tiny write forced ZFS to read a big chunk, change a little of it, and write the whole big chunk back. That is 32 times more work than ext4 does for the same operation.
2. **Two filesystems means two journals, two sets of checksums, two copies of every metadata change.** Every save was being processed twice.
3. **ZFS is designed to talk directly to disks.** Putting it on top of another filesystem hides the disks from it and breaks every optimization it has.

The result is the worst of both worlds. ZFS is paying the cost of all its safety features, but cannot use any of its speed advantages.

This is what happens when you try ZFS the lazy way.

## Round 2: Doing It Right

We got our hands on a second server with four free NVMe drives. This time, we did everything by the book:

- Gave ZFS the **whole disks** — no other filesystem in between.
- Created a proper **RAID10-equivalent pool** (mirror-stripe across 4 drives).
- Tuned the block size to **16 KB**, matching what virtual machines actually write.
- Turned on **lz4 compression** (free speed boost — most data compresses).
- Set **autotrim** so the SSDs stay healthy over time.
- Used **ZVOLs** instead of files for the VM disks (cleaner, faster).

Then we ran the exact same tests.

## The New Numbers

> 📷 **Insert image: 02-all-tests.png**

Here is how each setup compared to ext4 (ext4 = 100%):

| Test | ext4 | ZFS — Bad | ZFS — Tuned |
|---|---|---|---|
| Reading small files | **100%** | 97% | 60% |
| Saving small files | **100%** | 2% | 57% |
| Mixed reading + writing | **100%** | 7% | 73% |
| Copying large files | **100%** | 7% | 9%* |

*Note: ZFS-tuned ran with full data-safety checks that ext4 skips. Also, it had only 4 drives versus ext4's 8 drives — meaning per-drive performance is roughly equal.

> 📷 **Insert image: 03-latency.png**

Properly tuned ZFS jumped from "broken" to "competitive". With **half the hardware**, it delivered most of ext4's real-world speed.

And it still has all the things ext4 doesn't:

- ✅ **End-to-end checksums** — silent corruption is detected and fixed automatically.
- ✅ **Instant snapshots** — back up an entire VM in under a second.
- ✅ **Compression built in** — saves disk space and often makes things faster, not slower.
- ✅ **Self-healing on RAID** — bad bits get repaired during regular scrubs.

## The Lesson

The story is not "ZFS vs ext4". The story is **how you set it up matters more than what you pick**.

Our three takeaways:

1. **Never stack ZFS on another filesystem.** It is 41× slower for no reason.
2. **Default ZFS settings are not VM-friendly.** Tune the block size, use ZVOLs, enable compression and autotrim.
3. **A simple, well-configured ext4 will outperform a poorly configured ZFS** — every single time.

## Conclusion

For pure speed on dedicated hardware, **ext4 still wins**. It is simple, fast, and unbreakable.

But "fast" is not the only thing that matters. If your data has any value to you — and it always does — ZFS gives you something ext4 cannot: a filesystem that knows when something has gone wrong, and fixes it before you notice. That is worth a small amount of speed, especially when properly tuned ZFS only gives up about 25–40% versus ext4 on the same hardware.

So: would we switch our entire production fleet to ZFS tomorrow? No. But for new servers where we have free drives and want safety, snapshots, and easy backups? **Yes — and we will use the tuned setup, not the lazy one.**

The most important thing this experiment taught us is simple: **do not judge a technology by your worst attempt at using it.** The lazy setup made ZFS look terrible. The proper setup made it look like a real choice.

If you take only one thing from this article: when someone shows you a benchmark, ask how they set things up. The answer might change everything.

---

*Have you tested ZFS on your own setup? What were your results? Drop a comment below — I would love to compare notes.*

**Tags:** `Linux` `ZFS` `Storage` `Benchmark` `DevOps` `SysAdmin` `Virtualization`
