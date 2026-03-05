# Change Index

| CID | Type | Module | Description | Related BID/FID | Date |
|-----|------|--------|-------------|-----------------|------|
| C-B001-01 | Fix | init.d | Block procd watch registration if required geodata text dumps are absent to prevent loop crashes | B-001 | 2026-03-04 |
| C-B002-01 | Fix | update | Secure geodata updating process preventing cp failure and adding auto mkdir path creation | B-002 | 2026-03-04 |
| C-B003-01 | Refactor | update | Implement atomic switch mechanism for AdList and Geodata ensuring replacement only on download/validation success | B-003 | 2026-03-05 |
| C-B004-01 | Fix | update | Implement Shadow-folder atomic switches via rename syscall for geodata to prevent power-loss corruption | B-004 | 2026-03-05 |
| C-F006-01 | Feat | update | Add custom DNS resolution via hardcoded 119.29.29.29 for robust geodata/adlist downloads, bypassing proxy DNS collisions | F-006 | 2026-03-05 |
