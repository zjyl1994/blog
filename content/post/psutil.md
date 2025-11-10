---
title: "Golang采集系统信息"
date: 2018-01-31T15:58:00+08:00
lastmod: 2018-01-31T15:58:00+08:00
draft: false
---

一直以来就想做一个可以显示服务器信息的集中式监控页面。经过两三天艰苦努力终于放弃(前端实现地图上展示信息太难了。

虽然对应的前端做不出来，但是对应的信息采集后端已经写好并且可以使用，为了不浪费我的成果，在这贴出来。你们谁要是写个对应的前端的话，请务必通知我。

<!--more-->

```go
package main

import (
	"encoding/json"
	"flag"
	"fmt"
	"github.com/shirou/gopsutil/cpu"
	"github.com/shirou/gopsutil/disk"
	"github.com/shirou/gopsutil/host"
	"github.com/shirou/gopsutil/load"
	"github.com/shirou/gopsutil/mem"
	"github.com/shirou/gopsutil/net"
	"log"
	"net/http"
	"time"
)

type StatusServer struct {
	Percent  StatusPercent
	CPU      []CPUInfo
	Mem      MemInfo
	Swap     SwapInfo
	Load     *load.AvgStat
	Network  map[string]InterfaceInfo
	BootTime uint64
	Uptime   uint64
}
type StatusPercent struct {
	CPU  float64
	Disk float64
	Mem  float64
	Swap float64
}
type CPUInfo struct {
	ModelName string
	Cores     int32
}
type MemInfo struct {
	Total     uint64
	Used      uint64
	Available uint64
}
type SwapInfo struct {
	Total     uint64
	Used      uint64
	Available uint64
}
type InterfaceInfo struct {
	Addrs    []string
	ByteSent uint64
	ByteRecv uint64
}

func main() {
	port := flag.String("port", ":58000", "HTTP listen port")
	flag.Parse()
	http.HandleFunc("/", getInfo)
	err := http.ListenAndServe(*port, nil)
	if err != nil {
		log.Fatalln("ListenAndServe: ", err)
	}
}

func getInfo(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Access-Control-Allow-Origin", "*")
	w.Header().Add("Access-Control-Allow-Headers", "Content-Type")
	w.Header().Set("Content-Type", "application/json")
	fmt.Fprintf(w, infoMiniJSON())
}

func infoMiniJSON() string {
	v, _ := mem.VirtualMemory()
	s, _ := mem.SwapMemory()
	c, _ := cpu.Info()
	cc, _ := cpu.Percent(time.Second, false)
	d, _ := disk.Usage("/")
	n, _ := host.Info()
	nv, _ := net.IOCounters(true)
	l, _ := load.Avg()
	i, _ := net.Interfaces()

	ss := new(StatusServer)
	ss.Load = l
	ss.Uptime = n.Uptime
	ss.BootTime = n.BootTime
	ss.Percent.Mem = v.UsedPercent
	ss.Percent.CPU = cc[0]
	ss.Percent.Swap = s.UsedPercent
	ss.Percent.Disk = d.UsedPercent
	ss.CPU = make([]CPUInfo, len(c))
	for i, ci := range c {
		ss.CPU[i].ModelName = ci.ModelName
		ss.CPU[i].Cores = ci.Cores
	}
	ss.Mem.Total = v.Total
	ss.Mem.Available = v.Available
	ss.Mem.Used = v.Used
	ss.Swap.Total = s.Total
	ss.Swap.Available = s.Free
	ss.Swap.Used = s.Used
	ss.Network = make(map[string]InterfaceInfo)
	for _, v := range nv {
		var ii InterfaceInfo
		ii.ByteSent = v.BytesSent
		ii.ByteRecv = v.BytesRecv
		ss.Network[v.Name] = ii
	}
	for _, v := range i {
		if ii, ok := ss.Network[v.Name]; ok {
			ii.Addrs = make([]string, len(v.Addrs))
			for i, vv := range v.Addrs {
				ii.Addrs[i] = vv.Addr
			}
			ss.Network[v.Name] = ii
		}
	}
	b, err := json.Marshal(ss)
	if err != nil {
		return ""
	} else {
		return string(b)
	}
}
```

运行起来后，用浏览器访问被采集机器的58000端口就可以看到机器的信息了。

```json
{
	"Percent": {
		"CPU": 0,
		"Disk": 11.68627166041493,
		"Mem": 21.34952360680132,
		"Swap": 1.498353300558446
	},
	"CPU": [{
		"ModelName": "Intel(R) Xeon(R) CPU E5-2630 v4 @ 2.20GHz",
		"Cores": 1
	}],
	"Mem": {
		"Total": 2097467392,
		"Used": 447799296,
		"Available": 1649668096
	},
	"Swap": {
		"Total": 2145382400,
		"Used": 32145408,
		"Available": 2113236992
	},
	"Load": {
		"load1": 0,
		"load5": 0,
		"load15": 0
	},
	"Network": {
		"ens32": {
			"Addrs": ["192.168.1.68/24", "fe80::20c:29ff:fed6:1bea/64"],
			"ByteSent": 10643267063,
			"ByteRecv": 3625202751
		},
		"lo": {
			"Addrs": ["127.0.0.1/8", "::1/128"],
			"ByteSent": 8310060,
			"ByteRecv": 8310060
		}
	},
	"BootTime": 1512095971,
	"Uptime": 5290007
}
```

gopsutil可以获取更多信息，具体可以去看[gopsutil文档](https://godoc.org/github.com/shirou/gopsutil)。