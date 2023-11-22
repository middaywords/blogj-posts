---
title: cilium maps
date: 2023-10-18 15:42:19
tags:
- cilium
- bpf
---

# cilium maps

## cilium map list

我们知道，cilium 有很多 map，可以通过 cilium map list 看到现在跑的有哪些

```shell
root@host:/home/cilium# cilium map list
Name                     Num entries   Num errors   Cache enabled
cilium_lxc               7             0            true
cilium_ipcache           14            0            true
cilium_lb4_services_v2   0             0            true
cilium_lb4_backends_v3   0             0            true
cilium_lb4_reverse_nat   0             0            true
cilium_policy_00977      0             0            false
cilium_node_map          0             0            false
cilium_metrics           0             0            false
cilium_world_cidrs4      0             0            false
```

我们也可以从 host 上看到

```shell
root@1000-worker4:/# ls /var/run/cilium/
access_log.sock  cgroupv2  cilium-cni.log  cilium.pid  cilium.sock
health.sock  monitor1_2.sock  state  xds.sock
```

当我们在 cilium-agent pod 里执行 `cilium` 命令的时候，发生了什么，这篇博客旨在说明这背后发生的事情。

对于 `cilium` CLI 的每个子命令，实现了一个 `cobra.Command`的对象，对于 cilium map，有相关定义

```golang 
// mapListCmd represents the map_list command
var mapListCmd = &cobra.Command{
	Use:     "list",
	Short:   "List all open BPF maps",
	Example: "cilium map list",
	Run: func(cmd *cobra.Command, args []string) {
		// ...
	},
}

// mapGetCmd represents the map_get command
var mapGetCmd = &cobra.Command{
	Use:     "get <name>",
	Short:   "Display cached content of given BPF map",
	Example: "cilium map get cilium_ipcache",
	Run: func(cmd *cobra.Command, args []string) {
        // ...
	},
}
```

`Run` 函数里面在干嘛呢？就是通过 client 给 daemon 发了个 request。

```golang
// cilium/cmd/map_list.go
resp, err := client.Daemon.GetMap(nil)
	// api/v1/client/daemon/daemon_client.go
	-> func (a *Client) GetMap
		-> Client.transport.Submit
			// vendor/github.com/go-openapi/runtime/client/runtime.go
			-> func (r *Runtime) Submit(operation *runtime.ClientOperation)

```

request 在哪里处理呢？在 cilium-agent 里面处理

```go
// daemon/cmd/daemon_main.go
	// /map
	restAPI.DaemonGetMapHandler = NewGetMapHandler(d)
	restAPI.DaemonGetMapNameHandler = NewGetMapNameHandler(d)
	restAPI.DaemonGetMapNameEventsHandler = NewGetMapNameEventsHandler(d, mapGetterImpl{})
```

在 `NewGetMapHandler` 中，使用了一个 `mapRegister` 来存 cilium-agent 存储 mapname -> Map 对象的映射，遍历 `mapRegister` 即可拿到所有的 cilium bpf map。

可以看到 ping 住的 map 比 cilium map list 里面要多

机器1：cilium1.14

```shell
root@1000-worker2:/home/cilium# cilium map list
Name                       Num entries   Num errors   Cache enabled
cilium_lb4_services_v2     12            0            true
cilium_lb4_source_range    0             0            true
cilium_policy_03640        0             0            false
cilium_runtime_config      0             0            false
cilium_lxc                 0             0            true
cilium_lb4_reverse_nat     12            0            true
cilium_lb_affinity_match   0             0            true
cilium_lb4_affinity        0             0            false
cilium_ipcache             9             0            true
cilium_lb4_backends_v3     0             0            true
cilium_l2_responder_v4     0             0            false
cilium_auth_map            0             0            false
cilium_node_map            0             0            false
cilium_metrics             0             0            false
cilium_world_cidrs4        0             0            false
# 只有两个目录里面有内容
root@1000-worker2:/home/cilium# ls /sys/fs/bpf/tc/globals/
cilium_auth_map            cilium_ct4_global           cilium_lb4_affinity       cilium_lxc              cilium_snat_v4_external
cilium_call_policy         cilium_ct_any4_global       cilium_lb4_backends_v3    cilium_metrics          cilium_tunnel_map
cilium_calls_hostns_03640  cilium_encrypt_state        cilium_lb4_reverse_nat    cilium_node_map         cilium_world_cidrs4
cilium_calls_netdev_00002  cilium_events               cilium_lb4_reverse_sk     cilium_nodeport_neigh4
cilium_calls_netdev_00030  cilium_ipcache              cilium_lb4_services_v2    cilium_policy_03640
cilium_calls_netdev_00346  cilium_ipv4_frag_datagrams  cilium_lb4_source_range   cilium_runtime_config
cilium_calls_overlay_2     cilium_l2_responder_v4      cilium_lb_affinity_match  cilium_signals
root@1000-worker2:/sys/fs/bpf# ls cilium/socketlb/links/cgroup/cil_sock
cil_sock4_connect      cil_sock4_post_bind    cil_sock4_sendmsg      cil_sock6_getpeername  cil_sock6_recvmsg
cil_sock4_getpeername  cil_sock4_recvmsg      cil_sock6_connect      cil_sock6_post_bind    cil_sock6_sendmsg
```

机器2: cilium 1.13

```shell
root@1000-worker4:/home/cilium# cilium map list
Name                     Num entries   Num errors   Cache enabled
cilium_ipcache           13            0            true
cilium_lb4_services_v2   0             0            true
cilium_lb4_backends_v3   0             0            true
cilium_lb4_reverse_nat   0             0            true
cilium_policy_03131      0             0            false
cilium_lxc               6             0            true
cilium_node_map          0             0            false
cilium_metrics           0             0            false
cilium_world_cidrs4      0             0            false
root@1000-worker4:/sys/fs/bpf# tree
.
|-- ip -> tc
|-- sa -> tc
|-- sk -> tc
|-- tc
|   `-- globals
|       |-- cilium_call_policy
|       |-- cilium_calls_hostns_03131
|       |-- cilium_calls_netdev_00011
|       |-- cilium_calls_netdev_00027
|       |-- cilium_calls_overlay_2
|       |-- cilium_ct4_global
|       |-- cilium_ct_any4_global
|       |-- cilium_encrypt_state
|       |-- cilium_events
|       |-- cilium_ipcache
|       |-- cilium_ipv4_frag_datagrams
|       |-- cilium_lb4_backends_v3
|       |-- cilium_lb4_reverse_nat
|       |-- cilium_lb4_services_v2
|       |-- cilium_lxc
|       |-- cilium_metrics
|       |-- cilium_node_map
|       |-- cilium_policy_03131
|       |-- cilium_signals
|       |-- cilium_tunnel_map
|       `-- cilium_world_cidrs4
`-- xdp -> tc
```

## cilium bpf ...

为什么上面可以看到，`/sys/fs/bpf` 下的内容比 cilium map list 的多呢？因为 cilium map list 是操作所有
