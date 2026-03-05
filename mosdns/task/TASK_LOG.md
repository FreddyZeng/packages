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
