# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A fork of **ACL4SSR** — a collection of rule/config data for Clash proxy subscription conversion. There is **no application to build, and no test suite**. The repo is data files (rule lists, GeoIP DB, IP lists) plus subconverter config files plus one Python generator. "Working on" this repo means editing config/rule text and letting GitHub Actions keep data fresh. (`README.md` is just a joke — "删库跑路".)

## The core artifact: subconverter `.ini` configs

`Clash/config/*.ini` are **subconverter "external config" files** (the `[custom]` INI dialect), consumed by a subconverter service to transform a raw node subscription into a full Clash config. They are not Clash configs themselves. Grammar you'll edit:

- `ruleset=<组名>,<url-or-inline>` — attach a rule list (or inline `[]GEOIP,CN` / `[]FINAL`) to a proxy group.
- `custom_proxy_group=<组名>`select`<node-or-[]group>`...`` — define a selector group; `` ` `` separates fields, `[]name` references another group.
- Trailing flags: `enable_rule_generator`, `overwrite_original_rules`, `clash_rule_base=<url>` (the base Clash template that rules get injected into).

**Two things that will bite you:**

1. **Rule order = match priority (first match wins).** The Full config deliberately pins AI rulesets (`🤖 OpenAi`) to the very top so `claude.ai` isn't captured by a later `🌍 国外媒体` / keyword rule. Adding a broad ruleset above a specific one silently reroutes traffic. Keep new rulesets in the right position; `[]FINAL` (漏网之鱼) must stay last.
2. **Node names are hardcoded and cross-referenced.** In `ACL4SSR_Online_Full.ini`, group definitions reference literal node names (`LAX-GIA-Ultra-直连`, `LAX-GIA-Ultra-家宽`, `HKG-Tencent-直连`). The file header warns "不要随意改变关键字". A group's `select` node names must match the actual subscription's node names, and a group name used in one `ruleset=` or `[]ref` must match its `custom_proxy_group=` definition exactly (emoji included).

### Which `.ini` is actually maintained

**Only `ACL4SSR_Online_Full.ini` is customized for this fork** (points at `riofutab/clashfile`, uses the custom `LAX-GIA-Ultra` node names). The other `Online_*.ini` variants are **stock upstream ACL4SSR** left as-is — don't assume edits belong there. Variant name decoding, for reference: `Full`/`Mini` = ruleset breadth, `MultiMode` = adds rule/global/direct switch, `MultiCountry` = country-based auto groups, `_CF` = Cloudflare CFnat 优选 group, `_WARP` = WARP group, `NoAuto` = no url-test.

## Rule lists — `Clash/*.list`

Clash rule-provider lists (`DOMAIN-SUFFIX,...`, `IP-CIDR,...,no-resolve`, `DOMAIN-KEYWORD,...`). **Some are machine-generated — do not hand-edit:**

- `adobe.list`, `CloudflareCIDR.list`, plus root `CloudflareCIDR.txt` / `baipiao.txt` / `GeoLite2-Country.mmdb` are overwritten by scheduled Actions. Edit the generator/workflow, not the output.
- `yaml/bdg.yaml` is a different format — a standalone Clash **Meta** config using `rule-providers:` pulling from MetaCubeX `meta-rules-dat`. Unrelated to the subconverter `.ini` pipeline.

## The one runnable thing

`CloudflareCIDR-main.py` regenerates `Clash/CloudflareCIDR.list` + `CloudflareCIDR.txt` from `ipverse/asn-ip` for a hardcoded ASN allowlist (`included_asns`). To run locally:

```bash
pip install requests
python CloudflareCIDR-main.py
```

It downloads/unzips a repo, filters those ASNs' aggregated IPv4 ranges, and cleans up after itself. To change which networks are included, edit the `included_asns` list.

## Automation (`.github/workflows/`)

Four scheduled jobs, each commits back to the repo via `GITHUB_TOKEN`:

| Workflow | Updates | Schedule | Notes |
|---|---|---|---|
| `adobe.yml` | `Clash/adobe.list` | every 3 days | curls a hosts list, rewrites `0.0.0.0 ` → `DOMAIN,` |
| `CloudflareCIDR.yml` | `Clash/CloudflareCIDR.list` | every 5 days | runs the Python script |
| `Update GeoLite2-Country.yml` | `GeoLite2-Country.mmdb` | weekly (Mon) | wget w/ retry + size sanity-check |
| `baipiao_ip_list.yml` | `baipiao.txt` | daily | **requires repo variable `PROXYIP_URL`**; skips on non-200 |

When editing a workflow, preserve its "commit only if changed" guard so the repo isn't spammed with empty commits.
