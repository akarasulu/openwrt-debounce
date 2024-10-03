# Debounce Service

An OpenWrt service to quiesce and decouple commands. The pattern was originally used to safely restart services from event handler scripts in `/etc/hotplug.d/`.

>**WARNING**: Bash is required on target OpenWrt systems.

## Install and Uninstall

Deploy to OpenWrt systems by cloning onto any OS with `bash` and `direnv` installed:

```shell
cd debounce-service
direnv allow
install root@router
```

Uninstall by running the `uninstall` script the same way. The `direnv allow` command adds the project's git root to the path.

## Configuration

Each line in the `debounce.conf` file corresponds to the configuration parameters of a debounce service instance. Each instance debounces a single command associated with a single countdown timer with its timeout value:

```conf
# name|command and arguments|timeout|ticks|log priority
dnsmasq_restart|service dnsmasq restart|10|2|info
```

**NOTE**: ticks are vestigial (will remove later)

## Usage

Client handler hooks write the command string to a debounce service instance's fifo file:

```shell
echo 'service dnsmasq restart' | tee "$(cat /tmp/debounce_dnsmasq_restart)"/fifo
```

## Implementation

A debounce service instance at the other end of the pipe reads the line. If the command strings match, the debounce service instance resets its count down timer and begins counting down. When the timer reaches zero, it executes the command. This count down delay also gives handler client scripts time to exit releasing held locks.

The command is executed once instead of many times over in a train of activity. The timeout essentially enforces T seconds of quiescence after the last attempt to issue the command before actually running the command. This absorbs unnecessary invocations like restarting services. It [debounces](https://medium.com/ghostcoder/debounce-vs-throttle-vs-queue-execution-bcde259768) and decouples them preventing potential runaway cyclic chain invocations that might result.

## Raison d'être

OpenWrt event handler scripts under `/etc/hotplug.d/` are invoked by various services / facilities. These hooks make it so users can customize behavior. All good right?

However, 9 out of 10 handlers update OpenWrt configurations. For those changes to take effect the service needs to reload their configurations and this usually amounts to a restart. When the restarted service is either the direct parent or an ancestor process of the handler script, total mayhem results. Things like race conditions or worse, storms of invocation cycles result in run away chain reactions: i.e. dnsmasq DHCP event handlers. The mayhem is unpredictable depending on how the service works.

Using D-BUS was supposed to prevent these problems, and it does when done right. The purpose of a bus is to remove interdependence between event emitting sources and event consumers. Ideally, services emit events on the D-BUS so their handlers (implemented as listeners) respond externally to the event source: decoupling is the whole point. Unfortunately some services don't use the D-BUS, or even when they do, there's poor documentation on which events are or are not emitted.

These problems occur often. As a consequence, I found myself implementing this debouncing service pattern for many handlers to decouple and dampen the impact of invocation cascades. I extracted it as a standalone service to reuse it.

## Why use bash?

```shell
ssh router 'ls -lh /bin/bash'
-rwxr-xr-x    1 root     root      949.5K Jan 23  2023 /bin/bash
```

Compelling tradeoffs exchanged for just under a megabyte of extra space:

* standardize shell scripts and libraries across multiple distributions
* reuse preexisting script libraries consistently across all systems
* limited primitives and quirks make ash less productive and more error prone

Embedded systems including routers are rapidly evolving to increase ROM / flash storage capacities. Space limitations are no longer a factor in restricting the use of more reliable and robust software.

## TODO

* [ ] Go with UCI configurations in `/etc/config/debounce` to replace debounce.conf
* [ ] Make an opkg and push to a repository remove these [un]install scripts
