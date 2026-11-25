# Common Clk Framework (CCF) Details

This guide covers the Common Clk Framework (CCF) and the clock-provider
drivers under `drivers/clk/`. It captures recurring review feedback from the clk
maintainers. The core framework lives in `drivers/clk/clk.c` and the provider
API in `include/linux/clk-provider.h`.

A clock *provider* (controller/driver) implements `struct clk_ops` and
registers `struct clk_hw` objects. A clock *consumer* (the device that uses a
clock) calls the `clk_get()` / `clk_prepare_enable()` / `clk_set_rate()`
consumer API. Most review problems come from blurring these two roles.

## Suspicious Clock Flags Need a Justifying Comment — or a Different Design

Flags like `CLK_IS_CRITICAL`, `CLK_IGNORE_UNUSED`, and `CLK_GET_RATE_NOCACHE`
are almost always a symptom of a missing consumer driver or of something poking
the hardware behind CCF's back. When one is genuinely needed, the *reason* is
non-obvious to the next reader and must be captured in a comment that survives
without git archaeology; restating what the flag does (already documented in
`include/linux/clk-provider.h`) is not a justification — say *why this hardware
needs it*.

- **A do-nothing critical clock should not be a clock at all.** The preferred
  remedy for a `CLK_IS_CRITICAL` clock that has no children and is only ever
  "turn a bit on and leave it on" is to set the bit once in `probe()` (e.g.
  `regmap_update_bits()`) and **not register it with the framework** — this
  saves the memory and registration time of a clock nobody will ever control.
  Registering it critical is a band-aid that pins it on forever.

- **A flag with no matching ops does nothing.** `CLK_IS_CRITICAL` and
  `CLK_IGNORE_UNUSED` only take effect through the `.enable`/`.disable` (or
  disable-unused) path. On a clock whose ops only implement `.recalc_rate`, or
  on a generic mux/divider with no `.enable`/`.disable`, these flags are inert —
  drop them.

- **`CLK_IS_CRITICAL` means "cannot be gated", not "rate cannot change".**
  These are constantly conflated. A critical clock can still have its rate
  changed and be reparented; the framework still turns it on once. Only mark the
  leaf clock that must stay enabled — do **not** mark its parents critical;
  CCF keeps the parents enabled through the dependency chain.

- **`CLK_IS_CRITICAL` + `CLK_IGNORE_UNUSED` together is nonsense.** If a clock
  is critical it is never part of the disable-unused pass, so the second flag is
  meaningless. Pick the one that matches intent — and frequently what the author
  really wanted was `CLK_IS_CRITICAL`, or vice versa.

- **`CLK_IGNORE_UNUSED` guarantees almost nothing.** It only skips the one-time
  `clk_disable_unused()` pass at late init. It does not guarantee the clock is
  on to begin with, nor that it stays on — a probe deferral or a consumer coming
  and going can still gate it. It is a workaround for the missing boot-state
  "hand-off" to consumers; the real fix is a consumer driver that calls
  `clk_prepare_enable()`. Unused clocks are a by-product of the bootloader, not
  a stable contract.

- **`CLK_GET_RATE_NOCACHE` usually marks a driver bug.** It means something
  changes the rate outside CCF's knowledge (a display/firmware driver poking
  registers behind CCF's back). The better fix is usually proper suspend/restore
  of the clock registers. Question blanket use across every clock in a driver.

- **Prefer read-only ops over flags when Linux must not touch the clock.**
  Register with read-only ops (`*_ro_ops`) rather than marking a clock
  `CLK_IS_CRITICAL`. RO ops must be genuinely read-only — a `_ro_ops` with a
  writable `determine_rate`/`set_rate` is a bug; a clock that cannot change rate
  should implement neither. For hardware owned by firmware,
  model it as RO ops **plus** `CLK_GET_RATE_NOCACHE`, or better, do not register
  those clocks at all and list them in the `protected-clocks` DT property (see
  `Documentation/devicetree/bindings/clock/qcom,gcc.yaml`) so the OS leaves them
  alone.

- **Do not enshrine the critical flag in device tree.** `of_clk_detect_critical()`
  / a `clock-critical` property may describe a board-level always-on need, but
  must never substitute for a consumer driver calling `clk_prepare_enable()`.

- **Driver-specific flags do not belong in the core flags.** If a driver needs
  to distinguish its own clock types, keep a flag in its own private struct, not
  in `clk_hw::core::flags`.

## Parents: Use parent_data / parent_hws, Not Global Name Strings

Referencing a parent by its global CCF clock name is a bug waiting to happen:
internal CCF clock names are not a stable interface, only DT names are. A clock
registered with no parent at all is an orphan with rate 0, and enabling it never
enables its input — the clock appears to come "out of thin air."

- **Describe parents by pointer or firmware index, in this order of
  preference.** Use `.hw`/`.parent_hws` (via `clk_init_data.hws` /
  `CLK_HW_INIT_HWS()`) for clocks internal to the controller; use
  `struct clk_parent_data` with `.index` (and/or `.fw_name`) for inputs that
  come from device tree. `.index` is cheapest (no string compare). `.name` is a
  legacy fallback *only* for drivers migrating off a binding that never listed
  parents — new drivers must leave it unset.

- **`.index` is an index into the DT node's `clocks` property**, matching a
  `clock-names` entry for `.fw_name` — it is **not** an index into this
  provider's own clock array.

- **The name-only `clk_parent_data` trap.** A statically-initialized
  `&(const struct clk_parent_data){ .name = "x" }` is really
  `{ .index = 0, .name = "x" }`, which silently makes the core look up index 0
  of the node's `clocks` property first. When you must use `.hw` or `.name`,
  also set `.index = -1` explicitly.

- **Do not expose a controller-internal clock in DT just to find it by name.**
  Reference it with a direct `clk_hw` pointer instead. `clock-output-names` is
  essentially a debug feature; do not require it.

- **`parent_names`/`parent_name` string arrays are deprecated.** New drivers
  must not use them, and must not use `CLK_HW_INIT` (string-matching).

- **`num_parents` must match the real topology.** Use `ARRAY_SIZE()`. A gate or
  branch has exactly one parent — it is not a mux.

- **Fully initialize `struct clk_init_data` with `= {}`** so a partially
  initialized `init` does not leave `parent_data`/`parent_hws`/`num_parents` as
  stack garbage, and so field order does not matter. Be consistent across every
  clock in the driver.

```c
// WRONG: parent referenced by global CCF name string
static const char * const foo_parents[] = { "xtal" };
init.parent_names = foo_parents;

// CORRECT: DT input by fw_name (binding-stable), internal by hw pointer
init.parent_data = &(const struct clk_parent_data){ .fw_name = "xtal" };
init.num_parents = 1;
```

## enable/disable Run in Atomic Context; Sleep Only in prepare/unprepare

`.enable` and `.disable` run under the CCF **enable_lock spinlock** with
interrupts disabled. Anything that sleeps there — a long busy-wait, a
sleeping-bus access (I2C/SPI GPIO expander, slow regmap), `usleep_range()` — is
a "sleeping in atomic context" bug that can deadlock or corrupt state.
`.set_rate`, `.prepare`, and `.unprepare` run under the **prepare_lock mutex**
and are the place for operations that can sleep.

- The enable lock is for *fast* operations only, such as a single MMIO register
  write. Use `udelay()`/`mdelay()` for tiny unavoidable waits, never
  `usleep_range()`. Move PLL-lock polling, sleeping GPIO toggles, and regulator
  calls into `.prepare`/`.unprepare`. Do not add a `.prepare` callback if there
  is no sleeping work; a pure MMIO gate belongs in `.enable`.

- **Never call into another framework from inside a clk op.** Calling a reset
  driver or a *consumer* clk API while holding the clk spinlock/mutex creates
  recursive / ABBA deadlocks and blocks making the core locks finer-grained.
  Coordinate "reset must be deasserted before clk enable" style sequencing in a
  PM/genpd domain, not in `clk_ops`.

- **Keep a private driver lock when clocks share a register.** The core's
  prepare/enable locks protect the framework, not your registers — do not rely
  on them for mutual exclusion between register writes in your own driver. A PLL
  whose `.set_rate` restores an enable bit needs a lock to serialize against
  `.enable`/`.disable`, and should read back the enable state rather than assume
  it.

- **Error pointers are handled asymmetrically.** `clk_disable()`/
  `clk_unprepare()` deliberately ignore `NULL` and error pointers to simplify
  cleanup paths; `clk_enable()`/`clk_prepare()` deliberately do **not**. Passing
  an error pointer into enable/prepare is a driver bug to fix, not a core
  problem.

## Determining Rates with determine_rate()

Rate rounding is done entirely through the `.determine_rate` op (the old
`.round_rate` op has been removed from `struct clk_ops`). `.determine_rate`
receives a `struct clk_rate_request` the core can extend, enabling rate ranges,
coordinated rate changes, and per-clk locking.

- **The rounding contract:** the consumer-facing `clk_round_rate()` must return
  exactly the rate a following `clk_set_rate()` would produce — `.determine_rate`
  and `.set_rate` must agree on policy. `.determine_rate` reports the achievable
  hardware rate, not blindly the requested rate; compute what the hardware can
  actually do.

- **Clamp, do not fail.** If the request exceeds the maximum, return the clamped
  maximum, not `-EINVAL`. Returning 0 from `.determine_rate` should basically
  never happen — offer a higher achievable rate instead.

- **`.determine_rate` must not touch hardware.** Rate programming belongs in
  `.set_rate` (configure boot rates from DT `assigned-clock-rates`). The math
  should not depend on the current hardware state — compute from the request,
  since values can change in the interim.

- **The request is pre-populated.** `best_parent_hw` and `best_parent_rate` are
  already filled in the `clk_rate_request` by `clk_core_init_rate_req()`
  (`drivers/clk/clk.c`) — use them rather than calling `clk_hw_get_parent()` and
  re-fetching the parent rate. A `.determine_rate` on a mux must consider all
  possible parents.

- **`.set_rate` must re-round.** Do not assume the incoming rate is already
  rounded — the parent rate may have changed since `clk_round_rate()`.

- **Reuse the divider helpers:** `divider_determine_rate()`,
  `divider_ro_determine_rate()`, `divider_recalc_rate()`, and
  `clk_hw_determine_rate_no_reparent()` (which already prevents reparenting, so
  `CLK_SET_RATE_NO_REPARENT` is then redundant).

- **Round consistently across a chain**, and use **floor ops for SD/MMC (SDCC)
  clocks** so the rate never exceeds what was requested. If a divider rounds
  closest (`CLK_DIVIDER_ROUND_CLOSEST`), the mux feeding it should too
  (`CLK_MUX_ROUND_CLOSEST`). A mux feeding a device clock (NAND/eMMC) must never
  round *up*.

## CLK_SET_RATE_PARENT vs CLK_SET_RATE_NO_REPARENT

These two flags are routinely confused because both involve parents, but they
control different things. Getting them wrong means rate requests silently stop
propagating, or a carefully selected mux parent gets overwritten by the next
`clk_set_rate()`.

- `CLK_SET_RATE_PARENT` propagates a rate request *up* to the parent so an
  adjustable parent/pre-divider can be changed. It has **nothing to do with
  inheriting the parent's rate** — a common misconception. A pass-through clock
  that only forwards a rate often needs no custom ops at all: set
  `CLK_SET_RATE_PARENT` and let the core do the work. Default to setting it
  whenever nothing prevents propagation; it stops naturally at a fixed parent.

- `CLK_SET_RATE_NO_REPARENT` tells CCF to *keep* the current parent of a mux
  rather than switching parents to hit a rate. It is meaningless on a non-mux.
  Set it when the mux parent is chosen manually (its ID is exposed and a
  consumer selects it) so `clk_set_rate()` does not overwrite the selection.
  Prefer selecting the parent in DT via `assigned-clock-parents`.

- If a PLL/clock is fixed by design, do **not** set `CLK_SET_RATE_PARENT`; use
  `assigned-clock-rates` in DT. Sibling clocks on the same source should not
  disagree on this flag without an explanation. If a particular parent must
  never be selected, drop it from the parent list.

## Prefer Distinct clk_ops Over Branching Inside One clk_op

A callback that inspects a framework flag (`CLK_SET_RATE_GATE`, etc.), a
`pll->id`/type field, or the clock's name to decide what to do is harder to
read, slower, and couples behavior to data that can drift. The result is a
"land mine": a `clk_ops` structure should be self-contained and greppable, not
dependent on a matching flag.

- Give different clock types different `clk_ops` structures instead of one set
  of ops that branches at runtime. Dispatch on the ops pointer, not on an
  internal id or a string comparison.

- When behavior can be expressed by the presence or absence of an op, do that
  rather than adding a new `CLK_*` flag. For example, export and reuse a shared
  helper (a no-op `determine_rate`) instead of inventing a flag that must travel
  alongside the ops.

## Operate on clk_hw and the Provider API, Not struct clk / the Consumer API

Provider code must use `struct clk_hw` and the `clk_hw_*` provider API, never
`struct clk` and the consumer API. There is an active effort to remove the
`struct clk` instance auto-created with every `clk_hw`, so new code must not
depend on it. Seeing both `<linux/clk.h>` (consumer) and
`<linux/clk-provider.h>` (provider) included in one file is a smell —
provider/consumer drivers are rare.

- Register with `devm_clk_hw_register*()` (not the deprecated `clk_register_*`),
  and expose clocks via `of_clk_hw_onecell_get()`, or `of_clk_hw_simple_get()`
  for a single leaf clock.

- Use `clk_hw_get_rate()` / `clk_hw_round_rate()` / `clk_hw_is_enabled()` /
  `clk_hw_get_parent_by_index()`; reach the owning device with
  `clk_hw_get_dev()` (plus drvdata) or `dev_get_regmap(dev->parent, NULL)`
  rather than carrying a `struct clk`. Do **not** use `__clk_lookup()`,
  `__clk_get_hw()`, `__clk_is_enabled()`, or `hw.clk` — use `clk_hw_get_clk()`
  if you genuinely need a `struct clk`.

- A provider must not call the *consumer* API (`clk_get()`,
  `devm_clk_get_enabled()`, `clk_prepare_enable()`) on its own inputs — describe
  the input with `parent_data` and let the framework wire it up. Do any default
  hardware setup with direct register writes (or by calling the ops directly)
  before registering the clocks.

- **`NULL` is a valid clock.** `clk_get()` may legitimately return `NULL` ("don't
  care"); do not convert it to an error pointer, and treat `IS_ERR_OR_NULL()` on
  a clk as usually wrong. For a onecell provider with holes, fill unused slots
  with `ERR_PTR(-ENOENT)` (or a custom `.get` that returns `-EPROBE_DEFER` for
  not-yet-registered indices) rather than handing back `NULL` for an invalid
  index. Just register the clocks and let CCF resolve orphans — register
  roots-to-leaves to minimize transient orphans.

## A Provider Implements What Consumers Request — It Does Not Set Policy

The clock controller's job is to implement the hardware; deciding *what rate a
clock runs at* or *when it is enabled* is the consumer's job. Baking board- or
use-case-specific decisions into a shared SoC clock controller forces
constraints on every board that uses that SoC.

- **Never force an initial rate from the clock driver.** Do not hard-code
  divider values or hide rate "pokes" in init register tables. If a device needs
  a specific rate, its own driver calls `clk_set_rate()` / `clk_prepare_enable()`,
  or the rate is set in DT via `assigned-clock-rates`. The `.init` callback is
  discouraged (ill-defined semantics, slated for removal) — do setup in probe.

- **Do not enable a clock from `.set_rate()`.** Enabling belongs in
  `.enable`/`.prepare`; doing it from `.set_rate` does not walk the tree and is
  fragile.

- **No board/DVFS/cpufreq/interconnect policy in the clock controller.** A clk
  driver must not implement cpufreq via notifiers (cpufreq has
  `.target_intermediate()`; see `drivers/cpufreq/imx-cpufreq-dt.c`), must not
  reference interconnect resources, and must not expose OPP tables or hide a
  firmware/OPP API behind the clk API. Do not model a complex block (e.g. a
  DisplayPort link-rate PLL) as a clk tree when the controller should just call
  `phy_configure()`. "Clocks" that merely proxy a firmware kHz interface and
  have no parent are usually **interconnects** — write an interconnect provider.

- **`.recalc_rate()` reads hardware; it does not cache or write.** It reports the
  rate implied by the *current* register parameters and `parent_rate`; do not
  stash a shadow rate in software and sync it, and never write registers from
  `recalc_rate`. A disabled PLL still has a configured frequency — do not return
  0 for that. 0 Hz is itself a valid rate ("don't know"), and `clk_get_rate()`
  must always return something; do not block registration on a 0 rate. Whether a
  zeroed divider field means "pass parent through" (return `parent_rate`) or
  "off" (return 0) depends on the hardware — guard the divide and comment that
  the field may read 0 on init.

- For a shared clock several consumers must not disturb, use the rate protection
  API: `clk_set_rate_exclusive()` sets and protects atomically, whereas
  `clk_rate_exclusive_get()` then `clk_set_rate()` is racy.

## Model the Real Hardware Topology with Standard CCF Elements

Describe clocks using the standard building blocks — mux, divider, gate,
fixed-factor, and the composite helpers — with one `clk_hw` per hardware
element. Bespoke monolithic ops or giant precomputed "magic rate" tables are
hard to review, hide rate-propagation bugs, and duplicate code CCF already has.

- Split a composite PLL into PLL + post-dividers + muxes. Add a `fixed_factor`
  element on any path where the output is a fixed multiple/fraction of the input
  so `clk_get_rate()` reflects reality. For coupled gates, add an intermediate
  parent clock that owns the shared enable bit so enable-counting lives in the
  core. Do not twist a genuinely single complex clock into basic types just to
  avoid custom ops, though.

- Model PLLs as (fractional) multipliers computed at runtime, not as large
  precomputed rate tables. Do not invent software-only clocks that are not in the
  hardware so consumers can request them everywhere — a platform that lacks a
  clock uses a different compatible. Conversely, a clock that physically exists
  must be described even if currently unused; do not add dead/"to add later"
  code either.

- Prefer the existing divider flags over open-coding their behavior:
  `CLK_DIVIDER_ONE_BASED`, `CLK_DIVIDER_ALLOW_ZERO`, `CLK_DIVIDER_EVEN_INTEGERS`,
  `CLK_DIVIDER_MAX_AT_ZERO`. Without `CLK_DIVIDER_ALLOW_ZERO`, a 0 divider value
  (e.g. from firmware) hits the `WARN()` in `divider_recalc_rate()`.

## Device Tree Describes Hardware, Not Driver Structure

DT describes clock controllers — their register region, inputs, and outputs —
not the Linux clk/reset framework split or the driver's internal layout. Writing
bindings to mirror software causes overlapping `reg` properties, node-per-clock
sprawl, and bindings that cannot be shared across OSes.

- **One register region = one controller = one node.** `reg` properties must not
  overlap; two drivers must never map the same region (they cannot coordinate
  writes). Clocks are referenced through `#clock-cells` on the controller node.

- **Avoid `syscon`.** It lets unrelated drivers reach in and poke registers they
  should not own. Put `#clock-cells` (and `#reset-cells`, etc.) directly on the
  device node and split sub-functions into their own drivers (see below). Do not
  use `simple-mfd`/syscon child nodes to represent one device's sub-blocks.

- **No node-per-clock bindings, no `bit-offset`-style bindings, no
  `clock-output-names`.** Fixed oscillators belong at the DT root (they are on
  the board), expressed as `fixed-clock` / `assigned-clock-rates`, not hard-coded
  in the provider.

- **Clock-ID `#define`s and the order of the `clocks` property are permanent
  ABI.** Only append new IDs at the end; never renumber, rename, or delete one
  (mark an unused one `/* unused */`). A parent that was index 0 must stay index
  0 forever. Deprecate-and-add a new compatible rather than mutating an existing
  binding; a driver may ignore an extra `reg` index for back-compat.

## Co-located Reset/Other Controllers: Split Into Their Own Drivers

A clock controller register block often genuinely contains a reset controller
(this is normal for e.g. Qualcomm GCC) — its reset IDs are expected to be
defined in the binding. The problem is not that resets exist, but that reset,
pinctrl, or IRQ *logic* implemented inside the clk driver escapes review by the
right maintainers and tangles lifetimes.

- When clk and reset share one register range, they are one device: one DT node,
  one driver maps the region once (and holds the regmap), then spawns child
  devices for the sub-functions. Two mechanisms are available — pick whichever
  suits the driver:
  - **Auxiliary bus**: create the child with `devm_auxiliary_device_create()`
    (`drivers/base/auxiliary.c`); the child acquires the shared regmap via
    `dev_get_regmap(adev->parent, NULL)`. The auxiliary device's release function
    frees its own memory and must **not** be tied to the clk driver's bind
    lifetime.
  - **MFD**: register the sub-devices with `devm_mfd_add_devices()` from an MFD
    parent (`drivers/mfd/`), sharing a regmap created on the parent node.

  Either way, the reset half lives in `drivers/reset/` and IRQ chips in
  `drivers/irqchip/`, so the right maintainers review them.

- Still add the reset `#define`s to the binding header even if the clk driver
  itself does not implement them. Keep genuinely unrelated SoC glue out of
  subsystem drivers; coordinate cross-subsystem sequencing in a PM/genpd domain.

## Managed Resources and Registration Lifetime

A clock provider that unregisters or disables clocks while consumers are still
attached causes use-after-free and corrupts enable/prepare counts. Relying on
`devm_` cleanup, in the correct order, avoids an entire class of unbind bugs.

- Pass a `struct device` (not a bare `device_node`) so the driver is not
  DT-specific and can use `devm_clk_hw_register()`,
  `devm_platform_ioremap_resource()`, `devm_of_clk_add_hw_provider()`, etc. If
  everything is devm-managed, a `.remove()` is usually unnecessary — and a
  `.remove()` that disables/unregisters live clocks can hit active consumers.
  Use `devm_add_action_or_reset()` when cleanup must be ordered relative to
  clock shutdown. (devres tracks allocations to undo, not arbitrary deferred
  work, and only applies once a device is bound.)

- **Map registers and do hardware setup before registering clocks**, and call
  `of_clk_add_hw_provider()` only *after* all clocks are registered — so there
  is nothing to unwind on failure and consumers never see a half-built provider.

- **Fill `clk_hw` completely before registering it.** Ops can run immediately
  after `devm_clk_hw_register()`, so setting fields afterward is a race.

- **Do not reference `clk_hw::init` after registration** — the core sets it to
  `NULL`, so reading `init->name` afterward oopses. Save the name first, or rely
  on the core, which now prints the failing clock's name on registration error.

- **On failure, fail cleanly — never register uninitialized or partial data**
  into the provider; a corrupt clock makes consumers oops.

- Mark `clk_ops`, parent/mux tables, match data, and `clk_init_data` `const` so
  they land in read-only memory; use `struct_size()` / `devm_kcalloc()` /
  `__counted_by()` (see `struct clk_hw_onecell_data`) for trailing/flex arrays.

## Firmware-Owned Clocks: Be a Thin CCF Adapter

When firmware owns the clocks, a driver that keeps a partial duplicate of the
firmware's state (hard-coded rate tables, kernel-side divider math) cannot stay
consistent — CCF assumes it knows the topology while the firmware makes the real
decisions. Model such clocks as a thin adapter whose gate/rate requests go to
the firmware. See `drivers/clk/clk-scmi.c` and `drivers/firmware/arm_scmi/clock.c`.

- Keep a **static, `const`, RO-memory `clk_ops`** (or a small finite table of
  them); do not allocate a `clk_ops` per clock or move the ops struct to RW
  memory to build it at runtime.

- **Firmware has its own flag numberspace.** It must not encode or mirror CCF's
  flag bit numbers (the core may renumber them freely); translate explicitly
  (`if (fw_flag & X) ccf |= CLK_...`). SCMI consumers may just list the clock
  number rather than inventing `#define`s.

## Error Handling, Logging, and Register Access

Noise and missing checks make clk drivers hard to debug and can leave
uninitialized values in use. CCF is called very frequently, so hot-path logging
is especially harmful.

- Check `regmap_read()` / `regmap_update_bits()` return values and propagate the
  error — on a failed read the output variable is uninitialized. (Do not add
  checks to thin MMIO regmap calls that cannot fail, but be consistent.) Forward
  the real error with `PTR_ERR()` and `%pe`; beware `PTR_ERR(NULL) == 0` on error
  paths. Do not flatten errors (a negative poll value reported as `-EBUSY`, a
  masked `-ETIMEDOUT`). `.init`/probe return negative-on-error only.

- Use `dev_err_probe()` when a `struct device` is available (quiet on
  `-EPROBE_DEFER`). **Remove "I'm alive"/probe-success prints**, allocation-
  failure messages (the allocator already dumps a stack), `WARN_ON` after a
  failed allocation, and any `dev_err`/`dev_dbg` in hot paths like
  `recalc_rate`/`set_rate`. Do not duplicate what the core or `regmap` tracing
  already logs. Never use `__GFP_NOFAIL` or `BUG_ON`/panic on allocation.

- Prefer **regmap** over hand-rolled `readl`/`writel` + a custom spinlock: it
  provides locking (`regmap_config.use_raw_spinlock`) and
  `regmap_update_bits()`. Always set `regmap_config.max_register` (an unset max
  lets the debugfs regmap file oops the kernel), select the specific
  `REGMAP_MMIO` (not top-level `REGMAP`), and do not wrap regmap in your own IO
  helpers. `writel()`/regmap already contain the needed memory barriers — drop
  manual `mb()`/`wmb()`, and comment any barrier that genuinely must stay. Avoid
  `__raw_*` accessors.

- Use `readl_poll_timeout()` / `readl_poll_timeout_atomic()` /
  `regmap_read_poll_timeout()` (with a real time bound) for status polling, not
  a bare CPU spin loop. Use `FIELD_GET()`/`FIELD_PREP()`/`GENMASK()` (fold the
  shift into `GENMASK` to avoid overflow) instead of open-coded shift/mask.

- Rate math must be 64-bit-safe: use `div_u64()` / `div64_ul()` /
  `DIV64_U64_ROUND_UP()`, and `abs_diff()` rather than `abs(a - b)` on
  `unsigned long` rate differences (the subtraction can exceed a signed type,
  made worse by requests seeded to `ULONG_MAX`).

## Quick Checks

- **COMPILE_TEST**: new clk drivers should build on other architectures via
  `depends on <SOC_SYMBOL> || COMPILE_TEST` (meaningful only when the symbol is
  user-selectable). Drivers using `ioremap` should `depend on HAS_IOMEM`.

- **Avoid `CLK_OF_DECLARE`**: prefer a normal platform driver. Reserve
  `CLK_OF_DECLARE_DRIVER` for the few clocks genuinely needed early
  (clocksource/clockevent) and register the rest from a platform driver bound to
  the same node; add a comment naming the other driver that probes the
  compatible.

- **Separate clk_ops per type, not runtime branches**: see the dedicated section
  — reviewers flag callbacks that switch on a flag, id, or clock name.

- **Do not re-validate device tree**: the kernel is not a DT validator; drop
  code that checks DT contents a schema should cover. (Still sanity-check counts
  *you* control — e.g. a parent count read from firmware that is wildly out of
  range.)

- **Single-use headers**: fold register-offset/definition headers used by one
  driver into that C file; only SoC-family-wide definitions belong in a shared
  header (name it `common.h`, never `clk.h`). Include what you use; do not rely
  on implicit/transitive includes; a provider includes `clk-provider.h`, not
  `clk.h`.

- **Kconfig / modules**: `select` a parent-clock provider rather than
  `depends on` it (orphan handling and `fw_devlink` make ordering work);
  no `default y` (set it in a defconfig); make drivers `tristate`/modular where
  possible and actually test module load/unload; add `MODULE_DESCRIPTION()`,
  `MODULE_DEVICE_TABLE(of, …)`, `MODULE_LICENSE("GPL")` (not `"GPL v2"`); drop
  pointless `MODULE_ALIAS`; export shared helpers with `EXPORT_SYMBOL_GPL`.

- **Core changes need KUnit tests**: changes to `drivers/clk/clk.c`, the divider
  code, or a new helper should land with tests (a dedicated `clk-<thing>_test.c`
  following `clk-gate_test.c`, or `drivers/clk/clk_test.c`; add config to
  `drivers/clk/.kunitconfig`). Introduce the failing test first for bisect; use
  `drivers/clk/clk_kunit_helpers.c` and `clk_hw_register_kunit()`; respect
  lockdep (provider ops like `__clk_determine_rate()` need the prepare lock — use
  `clk_hw_get_clk_kunit()` + `clk_round_rate()`, not `*_nolock` helpers). Tests
  are hermetic and DAMP (copy, do not over-parameterize), use at least one
  `KUNIT_EXPECT_*`, check for a negative error rather than a specific errno, use
  `kunit_kzalloc()`, and are not modified once written. Expose test-only symbols
  with `EXPORT_SYMBOL_IF_KUNIT`.

- **Namespacing**: non-static symbols are global kernel-wide — prefix driver
  functions/tables with the SoC/driver name; do not put a `clk_hw` prefix on a
  helper that does not take a `clk_hw`.
