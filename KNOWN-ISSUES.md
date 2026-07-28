# Known issues (ii fork)

## RESOLVED (root cause found): WireguardMTU below 1280 cannot work — IPv6 minimum MTU

Found live on nextral 2026-07-28 (tunneld 0.1.21-sharing, client
v0.1.19-sharing), clean A/B/A:

- Server default 1280 + stock client: everything works (iimatey-probe green
  end-to-end through the public URL).
- `TUNNELD_WIREGUARD_MTU=1200` + fully restarted client (re-registers,
  receives 1200 in the registration response, netstack created at 1200 on
  BOTH ends): wg handshake still completes, but ALL TCP through the tunnel
  fails — tunneld's proxy 502s after ~10s on every request, even tiny HEADs.
- Revert to 1280 + restart client: instantly green again.

Also reproduced asymmetrically: client-side 1279 or 1200 against a 1280
server kills the path the same way (expected — the ends must match; but
symmetric non-1280 failing is not expected).

Both netstacks receive the same value (`tunneld/tunneld.go` CreateNetTUN;
`tunnelsdk/tunnel.go` CreateNetTUN via `res.WireguardMTU`), so something
else in the stack appears to assume 1280. Until this is fixed, the
`TUNNELD_WIREGUARD_MTU` knob and the client `TUNNEL_MTU` lab flag (34a1a56)
are unusable. Worth diffing against upstream coder/wgtunnel for later fixes.

Why we care: external clients reach wg.sharing.io via oci-ingress, which
forwards over tailscale (MTU 1280). Full-size wg datagrams (~1340 outer at
inner-1280) only survive that hop via IP fragmentation, and fragment-hostile
NATs (Windows/WSL) black-hole the fragments — small frames pass, screen-sized
output vanishes (`iimatey-probe` exit 4 signature). Lockstep MTU 1200 would
fit the tailscale hop without fragmentation and fix those clients — once
symmetric non-1280 works.

### Root cause (2026-07-28, local lab A/B over loopback)

Symmetric 1280: works. Symmetric **1400: works**. Symmetric 1200: dead
(even small requests). The floor is exactly 1280 because the tunnel's
inner network is IPv6-only (`wireguard-server-ip` "Must be an IPv6
address") and RFC 8200 mandates a minimum link MTU of 1280 for IPv6 —
gvisor/netstack correctly refuses to operate below it. Not a hidden
constant, not a bug.

Consequences:
- Lockstep MTU lowering below 1280 is impossible while the inner net is
  v6; the `--wireguard-mtu` knobs are valid only for values >= 1280 (and
  the ends must still match).
- Fixing fragmentation on a sub-1340 underlay hop (e.g. tailscale's 1280)
  therefore means raising the UNDERLAY MTU (e.g. TS_DEBUG_MTU on both
  tailscale ends of the hop), relocating tunneld to a 1500-MTU vantage,
  or making the fragment-dropping NAT tolerate fragments — not lowering
  the wireguard MTU.

### DEPLOYED FIX (2026-07-28): underlay MTU raised — fragmentation eliminated

TS_DEBUG_MTU=1360 set in /etc/default/tailscaled on BOTH ends of the
oci-ingress <-> nextral leg (tailscale0 mtu 1360 verified both ends,
direct path ~50ms). Outer wireguard datagrams (~1340 at inner-1280) now
fit the hop with no IP fragmentation, so fragment-hostile NATs
(Windows/WSL etc.) never see fragments. Verified: DF-ping 1300B passes /
1368B correctly rejected; iimatey-probe reports large ~1300B frames
surviving both ways (exit 0) through the public path.

Rollback if anything regresses: remove the TS_DEBUG_MTU line + restart
tailscaled, both ends.
