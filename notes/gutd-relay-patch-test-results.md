# gutd relay kernel-mode patch test notes

Patch source: `/workspace/gutd`, branch `cursor/tc-encap-adjust-room-3a8e`, commit `3a445d7`.

## Change

`src/bpf/tc_gut_egress.bpf.c` now passes UDP tunnel encapsulation flags to both `bpf_skb_adjust_room()` call sites:

- main TC egress path
- SIP signaling TC tail-call path

The flags are selected by `ipver`:

- `BPF_F_ADJ_ROOM_ENCAP_L3_IPV4` or `BPF_F_ADJ_ROOM_ENCAP_L3_IPV6`
- `BPF_F_ADJ_ROOM_ENCAP_L4_UDP`
- `BPF_F_ADJ_ROOM_FIXED_GSO`

## Build/test results in Cursor Cloud VM

Passed:

```text
cargo build --release
cargo test --release
```

`cargo test --release` result: 24 library tests + 13 integration tests passed. BPF skeleton generation completed successfully.

Attempted upstream relay stand:

```text
sudo env GUTD_BINARY=/workspace/gutd/target/release/gutd bash tests/test_relay_dnat.sh
```

Environment blockers:

1. Default `iptables-nft` backend failed in network namespace on UDP match; `iptables-legacy` works.
2. After switching test PATH to `iptables-legacy`, gutd failed at TC attach because this VM kernel lacks `sch_clsact` qdisc support:

```text
libbpf: Kernel error message: Specified qdisc kind is unknown
modprobe: FATAL: Module sch_clsact not found in directory /lib/modules/6.1.147
sudo tc qdisc add dev lo clsact -> Error: Specified qdisc kind is unknown.
```

Therefore full kernel-mode relay DNAT execution could not be completed on this VM, but the patched BPF code compiles and the non-privileged test suite passes.
