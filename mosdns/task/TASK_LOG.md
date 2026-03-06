# Task Log

## 2026-03-04
- **C-B001-01** (Bug Fix): 
  - Analyzed the mosdns restart loop issue described as "mosdns 更新GeoIP & GeoSite 数据库 失败后, 一直自动重启".
  - Discovered that PROCD's respawn infinitely restarts mosdns when its required parsed config files (e.g. `geosite_cn.txt`) are missing because `mosdns` exits with status 1 on failure to find them. By unconditionally destroying the generated dat files prior to checking the newly unzipped status, the system was prone to this state.
  - Implemented missing file existence guards (`if [ ! -s ... ]`) within `/etc/init.d/mosdns` pre-startup logic. Returning 1 properly aborts registration to procd's watcher, systematically defeating the infinite crash loop pattern.
  - Implemented diagnostic log tags such as `[INIT-B001-①]`.
  - Replaced unresilient short-circuits (`&& rm -rf && exit`) in `/usr/share/mosdns/mosdns.sh` with solid deterministic `if...then` traps preventing unhandled downloads.
  - Addressed R9 strict adherence. Assigned BID. Created B-001 bug schema for future maintainers.

- **C-B002-01** (Bug Fix):
  - Fixed an unhandled vulnerability during dat geodata installation pointing to `/usr/share/v2ray/`.
  - Added deterministic path creation check `mkdir -p /usr/share/v2ray`.
  - Wrapped `cp -a` behind rigid `if [ $? -ne 0 ]; then` validation returning 1 immediately to halt `mosdns` restart triggers on bad file dumps.
  - Enforced Bug isolation tag `[UPD-B002-①]`.

## 2026-03-05
- **C-B003-01** (Refactor):
  - Streamlined `adlist_update` and `geodat_update` in `/usr/share/mosdns/mosdns.sh` favoring a strict validation-before-overwrite (KISS) principle.
  - Substituted the pre-emptive `rm -rf` destructive behavior inside the `adlist_update` function with a transient staging area (`adlist.new`) performing instantaneous atomic namespace substitution upon validated extraction.
  - Eliminated complex `.bak` rolling restore implementations for Geodata files; validated `.dat` hashes securely govern direct replacement from sandbox `.sha256sum`.
  - Instantiated traceability labels `[UPD-B003-②]`.

- **C-B004-01** (Bug Fix):
  - Addressed the fatal flaw of updating databases over multiple filesystems which caused power-loss corruption.
  - Implemented shadow-folder atomic switch mechanism (`mv -f /usr/share/v2ray_tmp/* /usr/share/v2ray/`) to utilize atomic `rename(2)` within the same physical partition (overlayfs).
  - Ensured that cross-filesystem `cp` from tmpfs (`/tmp`) only targets the shadow temporary folder, completely shielding the live directory.
  - Generated diagnostics tracing tag `[UPD-B004-①]`.
  - Created B-004.md schema to govern root cause modeling.

- **C-F006-01** (Feature):
  - Addressed F-006: User requirement to specify a custom DNS server exclusively for MosDNS geodata/adlist `curl` updates.
  - Mitigated proxy lookup collision by avoiding reliance on the default OpenWrt resolver.
  - Encountered OpenWrt `libcurl` limitations (absence of `--dns-servers` and `--doh-url` support on embedded builds).
  - Re-implemented highly portable, `grep`-free `awk` filtering on `nslookup` output within `get_curl_resolve_args()` inside `/usr/share/mosdns/mosdns.sh` to extract pure IPv4 addresses.
  - Configured `mosdns.sh` to hardcode the resolving DNS server to `119.29.29.29`, bypassing any LUCI UI elements per user instruction.
  - Dynamically injected `--resolve HOST:PORT:IP` instructions into down-stream `curl` processes fetching GitHub repositories.

- **C-F007-01** (Feature):
  - Addressed F-007: Specify Custom DNS Server for SSR+ `update.lua`.
  - Injected `get_curl_command()` lua function into `/usr/share/shadowsocksr/update.lua`.
  - Replaced native `wget` execution with a dynamically formatted `curl --resolve` string.
  - Mirrored the F-006 `nslookup` + `awk` technique using Lua's `io.popen()` to fetch the target IPv4 of the URL against `119.29.29.29`.
  - Ensured reliable routing list downloads directly overriding SSR+ active port hijackings.

- **C-F006-02** (Fix):
  - Addressed F-006: Enforce strict hardcode DNS fallback in `/usr/share/mosdns/mosdns.sh`.
  - Upgraded `get_curl_resolve_args()` to output `--resolve HOST:PORT:119.29.29.29` explicitly if the primary `awk` IP extraction yields empty.
  - Aligned with SSR+ `update.lua` logic, ensuring no proxy DNS leak can occur under failure conditions.

## 2026-03-06
- **C-B005-01** (Bug Fix):
  - Analyzed compilation failure observed in `10_编译固件.txt`.
  - Identified the root cause as a data file clash. Package `luci-app-mosdns` attempted to install bundled `/usr/share/v2ray/geoip.dat` and `geosite.dat` directly from its `root` directory while its `Makefile` paradoxically registered explicit dependencies on `v2ray-geoip` and `v2ray-geosite`. This resulted in opkg triggering `check_data_file_clashes` during rootfs assembly.
  - Mitigated the conflict by recursively eliminating the redundant bundled static dat assets (`rm -rf root/usr/share/v2ray`) from the `luci-app-mosdns` packaging manifest folder.
  - Documented as B-005. Restoration allows the build chain to resume unaffected.

- **C-B005-02** (Reverted):
  - User requested retaining the architectural integrity of `+v2ray-geoip` and `+v2ray-geosite` dependencies in `luci-app-mosdns/Makefile`. Therefore, embedding files directly inside `luci-app-mosdns` and deleting the LUCI_DEPENDS edge is rejected. Restored `luci-app-mosdns/Makefile` to its previous state.

- **C-B005-03** (Refactor):
  - Addressed user requirement: "Retain `v2ray-geodata` system integration, but eliminate all Github downloads during compilation, injecting static dat files directly."
  - Target: `packages/net/v2ray-geodata/Makefile`.
  - Stripped out all `define Download/geoip` and `define Download/geosite` blocks.
  - Removed `$(call Download,geoip)` execution from `Build/Prepare`.
  - Replaced it with local filesystem copy operations: `$(CP) ./files/geoip.dat $(PKG_BUILD_DIR)/`, physically moving the pre-provided `.dat` binaries into the working directory.
  - Placed the payload `geoip.dat` and `geosite.dat` securely inside `packages/net/v2ray-geodata/files/`.
  - Result: `make package/net/v2ray-geodata/compile` no longer touches the internet. It instantly packages the provided static assets into `v2ray-geoip` and `v2ray-geosite`. Downstream dependents like `luci-app-mosdns` can keep their `LUCI_DEPENDS` completely intact and will link exactly the local assets as intended without opkg collisions.

- **C-B006-01** (Bug Fix):
  - Addressed OpenWrt startup race condition where SSR+ (`S99shadowsocksr`) initializes its iptables firewall routing before MosDNS (`S90mosdns`) has finished loading `geosite.dat` and binding UDP port 5335.
  - Modified `mosdns` STOP variable to 16 to maintain symmetric teardown execution ordering (LIFO shutdown constraint).
  - Drafted and ultimately **rejected** a dual-polling `boot()` override in `S90mosdns` that asynchronously issued `/etc/init.d/shadowsocksr restart` due to fatal Time-Gap race condition vulnerabilities against the imminent `S99` execution.
  - Enforced a rigorous `wait_for_mosdns` readiness probe directly inside the `start()` function of `/etc/init.d/shadowsocksr`.
  - Probe logic polls OpenWrt `netstat -unlp` / `netstat -tlnp` every 1 second (up to 60 seconds) strictly asserting the `/mosdns` binary has acquired its network socket before relinquishing execution flow back to the SSR initializer.
  - Implemented POSIX resilient `$((i + 1))` arithmetic syntax, averting `let i++` crashes on minimalist Ash shell environments.
  - Linked Tracking BUG B-006. Diagnostic tagging applied: `[INIT-B006-①~④]`.
