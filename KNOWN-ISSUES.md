# Known issues (ii fork)

## Symmetric WireguardMTU != 1280 breaks the data path entirely

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
