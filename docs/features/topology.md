# Cluster Topology

**Settings → Topology** in the admin UI is the live picture of a kiok cluster: the ZooKeeper
ensemble the cluster coordinates through, every Master with the current leader marked, every
Worker, and whether each one is ready. It refreshes every 10 seconds, and any node's log can be
streamed from it.

## ZooKeeper ensemble

kiok keeps its cluster state that must survive any single node in ZooKeeper: Master leader
election, the node registry (which Masters and Workers exist and whether they are ready), the
cluster-ready flag, and SAML assertion replay protection. When the ensemble loses quorum, leader
election and node registration stop. The ZooKeeper card at the top of the page shows its state
before that happens.

For every server in `kiok.zk.serverList`:

| Column | Meaning |
|---|---|
| Server | `host:port` from the connect string (a chroot suffix such as `/kiok` is ignored) |
| Role | `leader`, `follower`, `observer`, or `standalone` for a single-server ZooKeeper |
| Status | `up` when the server answers and is serving; `down` otherwise, with the reason underneath |
| Version | the ZooKeeper version the server runs |
| Connections | client connections open on that server |
| Latency min/avg/max | request latency the server has measured, in milliseconds |
| Znodes | node count in the server's data tree |
| Zxid | the last transaction id the server has applied; on a healthy ensemble all servers converge to the same value |

The header shows how many servers are up (`2/3 up`), the connect string, and the **session** of
the Master that answered the page: `connected` or `reconnected` while it holds a ZooKeeper
session, `suspended` when the connection dropped but the session may still be alive, `lost` when
the session expired.

### How the state is read

The Master that serves the request asks each server with ZooKeeper's `srvr` four-letter-word
command over a plain TCP connection to its client port. All servers are asked in parallel, each
with a timeout of `kiok.zk.topology.probe.timeout.ms` (default 2000 ms), so the card costs at most
one timeout even when servers are unreachable.

`srvr` is on ZooKeeper's default four-letter-word whitelist (`4lw.commands.whitelist`) since
3.5, so no ZooKeeper configuration is needed. A server is shown `down` when:

- the connection or the read times out, or the connection is refused — the reason is the network
  error, for example `SocketTimeoutException: Connect timed out`;
- it answers `This ZooKeeper instance is not currently serving requests` — it is running but has
  no quorum;
- it answers `srvr is not executed because it is not in the whitelist` — the whitelist was
  narrowed and no longer contains `srvr`. Add `srvr` back to `4lw.commands.whitelist` to see the
  server's state.

The probe is read-only and any Master answers it; it does not go through the leader.

### Reading it during an incident

- **One server `down`, the rest `up`, session `connected`.** A three-server ensemble keeps quorum
  with one server out, so kiok keeps working. Restart the server; it rejoins as a follower and its
  zxid catches up.
- **A new `leader` after a server stops.** If the stopped server was the leader, the remaining
  servers elect a new one within seconds; the session briefly shows `reconnected`.
- **Most servers `down`, or the remaining ones answer "not currently serving".** Quorum is lost:
  no leader election, no node registration, and Masters' sessions turn `suspended`, then `lost`.
  Restore enough servers for a majority.
- **Zxids that stay apart.** A follower whose zxid does not converge with the leader's is not
  keeping up; check its disk and network.

## Masters and Workers

| Column | Meaning |
|---|---|
| Node | node id (`<host>-<role>-<internal port>`) |
| Host, Admin, Internal | the node's advertised host, admin HTTP port (Masters) and internal NIO port |
| Role | `Leader` for the Master that holds leadership, `Follower` for the others |
| Status | `ready` once the node has finished bootstrap and synchronised with the leader |

**Log** on any row streams that node's log live (offset-based, with Live / Paused). It is always that
node's own log, whichever Master served the page: a Worker's log comes from the Worker over the
internal channel, and another Master's log is fetched from that Master. An id that names no node says
so (`unknownNode: true`) rather than showing some other log. Each poll returns only complete lines
written since the previous one — byte-accurate, so non-ASCII text does not drift — and the viewer starts
again from the end of the file if the log was truncated or rotated.
**Enter Maintenance** in the page header holds the cluster still — see
[Cluster Maintenance Mode](cluster-maintenance.md).

## API

All endpoints need an authenticated admin.

```
GET /api/v1/admin/nodes        # masters, workers and the ZooKeeper ensemble
GET /api/v1/admin/zookeeper    # the ZooKeeper ensemble only
```

```json
{
  "connectString": "zk1:2181,zk2:2181,zk3:2181",
  "session": { "nodeId": "master-master-19999", "state": "CONNECTED", "connected": true },
  "servers": [
    { "host": "zk1", "port": 2181, "status": "UP", "mode": "follower", "version": "3.9.6",
      "zxid": "0x200000004", "connections": 2, "latencyMin": 0, "latencyAvg": 5.94, "latencyMax": 57,
      "received": 1520, "sent": 1533, "outstanding": 0, "nodeCount": 16, "error": null },
    { "host": "zk3", "port": 2181, "status": "DOWN", "mode": null, "version": null, "zxid": null,
      "connections": null, "latencyMin": null, "latencyAvg": null, "latencyMax": null,
      "received": null, "sent": null, "outstanding": null, "nodeCount": null,
      "error": "SocketTimeoutException: Connect timed out" }
  ]
}
```

`/api/v1/admin/nodes` returns `{ "masters": [...], "workers": [...], "zookeeper": { ... } }` with the
same `zookeeper` object.

## Configuration

| Property | Default | Description |
|---|---|---|
| `kiok.zk.serverList` | `localhost:2181` | The ensemble the cluster uses — and the servers the card lists. |
| `kiok.zk.topology.probe.timeout.ms` | `2000` | Connect and read timeout (ms) of the `srvr` probe sent to each server. A server that does not answer within it is shown `down`. |

## Verified

`tests/e2e-zk-topology.sh` in the kiok repository runs a Master and a Worker on a three-server
ensemble (`tests/docker-compose-zk-ensemble.yml`) and checks: all three servers `up` with one
leader and two followers and every field reported; a stopped server `down` with the reason while
the other two stay `up` and the Master's session stays connected; the server `up` again after a
restart; and the Topology page rendering the section in a headless browser.
