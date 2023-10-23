---
title: cilium maps
date: 2023-10-18 15:42:19
tags:
- cilium
- bpf
---

# cilium maps

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

