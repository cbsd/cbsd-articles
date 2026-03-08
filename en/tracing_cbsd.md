# Profiling and Tracing CBSD

The core of the CBSD codebase is Posix shell code, with the exception of a number of custom build-in functions written specifically for CBSD.

This is where the beauty and poverty of shell scripts come into play: on the one hand, they don't require constantly running services
consuming RAM and CPU time, resulting in near-zero overhead, as the scripts run in an on-demand solution.

Unlike other interpreted languages, such as Perl and Python, the Posix shell interpreter consumes less than 100 kilobytes of disk
space and has minimal RAM overhead. If virtual environment creation operations aren't heavily loaded, this is a definite plus. The downside of the shell is its extremely limited string handling mechanisms,
which still require processing. This forces the use of a large number of external utilities,
such as `awk, tr, cut, sed, grep`, etc., the extensive use of which significantly reduces the speed of scripts.

:information_source: It's worth noting that CBSD calls all external utilities through *XXX_CMD* macros, which are overridden, so for Embedded systems and platforms where responsiveness is important, these utilities are "moved" to RAMFS, which provides some performance gain.

This section covers some of the built-in tracing and profiling options available for CBSD scripts.

## Instruction tracing and function timing

The first thing available to users is sh's built-in instruction tracing mechanism: `set -o xtrace`. Global activation of this mechanism is controlled by the *CBSD_DEBUG* environment variable. For example, run any command with this global variable at the command line:
```
env CBSD_DEBUG=1 cbsd jls
```

The second powerful feature built into CBSD is function execution timing statistics in Prometheus format. Global activation of this mechanism is controlled by the *CIX_FUNCTION_TIME* environment variable. For example, run any command from the command line with this global variable:
```
env CIX_FUNCTION_TIME=1 cbsd jls
```

Statistics will be printed to stderr, which can be conveniently sorted using "sort -n -k2" to obtain the stack and the top slowest operations:
```
% env CIX_FUNCTION_TIME=1 cbsd jls 2>&1 | grep ^cix | sort -n -k2 |tail -n16
cix_function_time{function="get_jid"} 0.001985
cix_function_time{function="get_jid"} 0.002088
cix_function_time{function="get_jid"} 0.002088
cix_function_time{function="populate_output_data"} 0.004224
cix_function_time{function="populate_output_data"} 0.004327
cix_function_time{function="populate_output_data"} 0.004677
cix_function_time{function="init_jail_path"} 0.009386
cix_function_time{function="init"} 0.009388
cix_function_time{function="init_jail_path"} 0.009485
cix_function_time{function="init_jail_path"} 0.009511
cix_function_time{function="init_rcconf"} 0.016368
cix_function_time{function="init_rcconf"} 0.016650
cix_function_time{function="init_rcconf"} 0.017025
cix_function_time{function="show_jaildata_from_sql"} 0.069603
cix_function_time{function="show_local"} 0.073026
cix_function_time{function="show_jails"} 0.073096
```

Before showing how this might look in a more presentable form, let's take a short detour and, for greater detail, let's act as timekeepers and dive into the macro world, stretching time no worse than black holes.

## Slowing down all living things

For greater detail, we'll try to artificially slow down CPU time to get a more detailed and visual picture of the most "heavy" functions.
It should be noted right away that this isn't always the best or correct method. It should be considered a method that can generate ["Heisenbugs"](https://en.wikipedia.org/wiki/Heisenbug) -a slang term
for a software error that disappears or changes its behavior when you try to detect or investigate it. In other words, the trace results and
the application's overall performance won't always correlate with how the software behaves without CPU stress, which can lead to distorted results and
incorrect conclusions about bottlenecks. Nevertheless, CBSD shell scripts are quite simple and straightforward, and the method is suitable.

### Method 1: Slowing Down the FreeBSD Kernel

Let's start with a case where the guest system being profiled is running FreeBSD (of course, the same tricks can be used with the Linux kernel).
We'll introduce an artificial slowdown right at the heart of the scheduler, in the /usr/src/sys/kern/sched_ule.c file, in the *sched_switch* function, and insert a simple DELAY:
```
diff -ruN /usr/src/sys/kern/sched_ule.c.orig /usr/src/sys/kern/sched_ule.c
--- /usr/src/sys/kern/sched_ule.c.orig  2025-10-31 10:03:28.054607000 +0300
+++ /usr/src/sys/kern/sched_ule.c       2026-03-05 13:45:18.075661000 +0300
@@ -2339,6 +2339,8 @@
 #ifdef SMP
        int pickcpu;
 #endif
+
+       DELAY(5000);
 
        THREAD_LOCK_ASSERT(td, MA_OWNED);
```

Let's compile the kernel and reboot. Once the kernel is initialized, all subsequent operations will resemble the OS running on a conventional 386 processor,
where even booting and displaying system messages can feel like slow line-by-line scrolling.
As a "test" script, let's take the bhyve virtual machine creation function in CBSD 15.0.4 and examine it in detail.

Script execution time: *220* seconds.

Last lines by *FUNCTION_TIME*:
```
cix_function_time{function="init"} 0.827038
cix_function_time{function="init_jail_path"} 0.836316
cix_function_time{function="init"} 0.836606
cix_function_time{function="init"} 0.856776
cix_function_time{function="update_netinfo"} 0.995015
cix_function_time{function="get_vm_uplink_interface"} 0.996033
cix_function_time{function="init"} 1.007656
cix_function_time{function="init_rcconf"} 1.232178
cix_function_time{function="update_table"} 1.301270
cix_function_time{function="export_bhyve_data_for_external_hook"} 1.422503
cix_function_time{function="init"} 1.676095
cix_function_time{function="init"} 2.003482
cix_function_time{function="update_table"} 2.593398
cix_function_time{function="register_insert_bhyve"} 3.174361
cix_function_time{function="really_create_vm"} 3.923576
cix_function_time{function="update_table"} 6.812324
cix_function_time{function="merge_profiles"} 13.023428
cix_function_time{function="merge_profiles"} 15.135411
cix_function_time{function="merge_profiles"} 15.217432
```


This means that the update_tables and merge_profiles operations are quite heavy for the `bcreate` operation, accounting for >80% of the time, and should be optimized if possible.
Looking at merge_profiles, we see that the functions call a large number of grep statements in a loop across all profile files to merge parameters. This is a fairly simple operation for
small C code, so as part of the optimization, we replaced the shell function with a large number of grep statements with a single, general-purpose C code. The problem with update_table here
is that during the virtual machine creation process, the SQL client is called sequentially many times—there are as many calls as there are tables. We can also optimize here and execute all operations atomically in a single transaction.

The output in prometheus format allows us to visually obtain the profiling as a graph in Grafana:

<center><img src="https://convectix.com/img/bcreate-cixtime1.png" width="1024" title="bcreate-cixtime" alt="bcreate-cixtime"/></center>

What if we write a small converter into a format understandable by [flamegraph](https://github.com/brendangregg/FlameGraph) ( `pkg install -y flamegraph` ):
<details>
  <summary>convert.py</summary>

```
#!/usr/bin/env python311
import sys
import re
import argparse

def main():
    parser = argparse.ArgumentParser(description='Convert metrics to Flamegraph folded format.')
    parser.add_argument('file', nargs='?', help='Path to metrics file (reads from stdin if omitted)')
    parser.add_argument('--exclude', '-e', action='append', help='Regex to exclude functions')
    parser.add_argument('--include', '-i', action='append', help='Regex to include ONLY these functions')
    args = parser.parse_args()

    # Чтение данных
    if args.file:
        try:
            with open(args.file, 'r') as f:
                lines = f.readlines()
        except FileNotFoundError:
            print(f"Error: File {args.file} not found.", file=sys.stderr)
            return
    else:
        lines = sys.stdin.readlines()

    pattern = re.compile(r'function="([^"]+)"\}\s+([\d.]+)')
    all_data = []

    for line in lines:
        match = pattern.search(line)
        if match:
            name, val = match.group(1), float(match.group(2))

            if args.include and not any(re.search(inc, name) for inc in args.include):
                continue

            if args.exclude and any(re.search(exc, name) for exc in args.exclude):
                continue

            all_data.append((name, val))

    if not all_data:
        print("No data left after filtering.", file=sys.stderr)
        return

    sorted_all = sorted(all_data, key=lambda x: x[1], reverse=True)
    hierarchy = []
    seen = set()

    max_time = sorted_all[0][1]
    for name, val in sorted_all:
        if name not in seen and val > (max_time * 0.5):
            hierarchy.append(name)
            seen.add(name)
        if len(hierarchy) >= 3: break

    stacks = {}
    h_set = set(hierarchy)
    for name, val in all_data:
        if name in h_set: continue
        full_path = ";".join(hierarchy + [name])
        stacks[full_path] = stacks.get(full_path, 0) + int(val * 1000000)

    for stack, weight in stacks.items():
        print(f"{stack} {weight}")

if __name__ == "__main__":
    main()

```
</details>

Then we can visualize this as a stack:

<center><img src="https://convectix.com/img/bcreate-cixtime1.svg" width="1024" title="bcreate-cixtime" alt="bcreate-cixtime"/></center>

After modifications to optimize the above operations, the version of the `bcreate` script for CBSD 15.0.5 already gives the following indicators for top-slow functions:
```
cix_function_time{function="init_rcconf"} 1.009101
cix_function_time{function="update_netinfo"} 1.045359
cix_function_time{function="get_vm_uplink_interface"} 1.046434
cix_function_time{function="export_bhyve_data_for_external_hook"} 1.558380
cix_function_time{function="init"} 1.761473
cix_function_time{function="really_create_vm"} 2.515143
cix_function_time{function="init"} 2.591617
cix_function_time{function="register_insert_bhyve"} 2.817843
```
And the total execution time dropped to *146* seconds, meaning we've sped up the script by 1.5 times!

If we return to a real, non-slowed down system, the total script execution time drops to *5* seconds.

### Method 1: Slow down the hypervisor

The previous example demonstrated system slowdown using the FreeBSD kernel as an example, and it worked both on bare-metal and in a virtual machine.
However, if you're only performing these exercises with a virtual machine, a more universal solution is to slow down the hypervisor.
In this case, we don't need to modify the kernel source code of the OS being launched.

Simply adding an artificial delay to VM exit isn't enough, as this will effectively slow down I/O-heavy VMs. However, if the host
almost never exits VM exit (for example, during a CPU benchmark), the effect will be weak, and a dynamic delay must be added, depending on
how often (in TSC cycles) the guest executes VM_exit. To avoid patching and rebuilding the vmm.ko module each time, let's add sysctl parameters
for easy host-side delay adjustment:
```
sysctl hw.vmm.vcpu_delay_us=100
sysctl hw.vmm.vcpu_slowdown_pct=100
```

Patch for FreeBSD 15.0-RELEASE:
```
diff --git a/sys/amd64/vmm/vmm.c b/sys/amd64/vmm/vmm.c
index 050cc93d26..f1b222e92b 100644
--- a/sys/amd64/vmm/vmm.c
+++ b/sys/amd64/vmm/vmm.c
@@ -64,6 +64,8 @@
 #include <x86/apicreg.h>
 #include <x86/ifunc.h>
 
+extern uint64_t	tsc_freq;
+
 #include <machine/vmm.h>
 #include <machine/vmm_instruction_emul.h>
 #include <machine/vmm_snapshot.h>
@@ -181,6 +183,16 @@ static int trap_wbinvd;
 SYSCTL_INT(_hw_vmm, OID_AUTO, trap_wbinvd, CTLFLAG_RDTUN, &trap_wbinvd, 0,
     "WBINVD triggers a VM-exit");
 
+static int vcpu_delay_us;
+SYSCTL_INT(_hw_vmm, OID_AUTO, vcpu_delay_us, CTLFLAG_RWTUN,
+    &vcpu_delay_us, 0,
+    "Base microseconds to busy-wait in host after each VM-exit");
+
+static int vcpu_slowdown_pct;
+SYSCTL_INT(_hw_vmm, OID_AUTO, vcpu_slowdown_pct, CTLFLAG_RWTUN,
+    &vcpu_slowdown_pct, 0,
+    "Extra busy-wait time as a percentage of vcpu runtime per VM-exit");
+
 /* global statistics */
 VMM_STAT(VCPU_MIGRATIONS, "vcpu migration across host cpus");
 VMM_STAT(VMEXIT_COUNT, "total number of vm exits");
@@ -1126,7 +1138,7 @@ vm_run(struct vcpu *vcpu)
 	struct vm_eventinfo evinfo;
 	int error, vcpuid;
 	struct pcb *pcb;
-	uint64_t tscval;
+	uint64_t tscval, runtime;
 	struct vm_exit *vme;
 	bool retu, intr_disabled;
 	pmap_t pmap;
@@ -1163,10 +1175,25 @@ vm_run(struct vcpu *vcpu)
 
 	save_guest_fpustate(vcpu);
 
-	vmm_stat_incr(vcpu, VCPU_TOTAL_RUNTIME, rdtsc() - tscval);
+	runtime = rdtsc() - tscval;
+	vmm_stat_incr(vcpu, VCPU_TOTAL_RUNTIME, runtime);
 
 	critical_exit();
 
+	if (__predict_false(vcpu_delay_us > 0 || vcpu_slowdown_pct > 0)) {
+		uint64_t usec;
+
+		usec = vcpu_delay_us;
+		if (vcpu_slowdown_pct > 0 && tsc_freq != 0) {
+			uint64_t runtime_us;
+
+			runtime_us = runtime * 1000000ULL / tsc_freq;
+			usec += runtime_us * (uint64_t)vcpu_slowdown_pct / 100ULL;
+		}
+		if (usec > 0)
+			DELAY((int)usec);
+	}
+
 	if (error == 0) {
 		retu = false;
 		vcpu->nextrip = vme->rip + vme->inst_length;
```

The effect will be similar to slowing down through kernel code, but thanks to its implementation at the hypervisor level, 
the ability to reduce system performance is now available for any OS.


