# Профилирование и трассировка CBSD

Основа кодовой базы CBSD составляет код на posix shell, за исключением ряда кастомных build-in функций, написанных специально для нужд CBSD. 
Здесь раскрываются блеск и нищета скриптов на shell - с одной стороны, они не требуют постоянно работающих сервисов, 
потребляющий оперативную память и процессорное время, что дает околонулевой оверхед, поскольку скрипты отрабатывают в решиме "on-demand". 

В отличии от других интерпретируемых языков, таких как perl и python, интерпретатор posix shell потребляет менее 100 килобайт дискового 
пространства и минимальный оверхед по работе с оперативной памятью и если операции по созданию виртуальных окружений не идут большим 
потоком - это выглядит как несомненный плюс. Обратная сторона shell - крайне скудные механизмы по работе со строчками, 
обработка которых все таки требуется. Это вынуждает использовать большое количество внешних утилит, 
таких как `awk, tr, cut, sed, grep` и тд, обильное использование которых значительно снижают скорость работы скриптов.

:information_source: стоит отметить, что CBSD использует вызов всех внешних утилит через *XXX_CMD* макросы, которые переопределяются, поэтому для Embedded систем и платформ, где отзывчивость важна, данные утилиты "выносяться" на RAMFS, что дает некоторый прирост).

Здесь рассматриваются некоторые встроенные варианты трассировки и профилирования CBSD скриптов.

## Трассировка инструкций и тайминг функций

Первое, что доступно пользователям - это встроенный в sh механизм трассировки инструкций: `set -o xtrace`. Глобальную активацию этого механизма регулирует переменная окружения *CBSD_DEBUG*. Например, выполните в командой строке любую команду с этой глобальной переменной:
```
env CBSD_DEBUG=1 cbsd jls
```

Вторая мощная и встроенная в CBSD функция - статистика по таймингу выполнения функций в формате prometheus, глобальная активация этого механизма регулируется переменной окружения *CIX_FUNCTION_TIME*. Например, выполните в командной строке любую команду с этой глобальной переменной:
```
env CIX_FUNCTION_TIME=1 cbsd jls
```

В stderr будет напечатана статистика, которую удобно отсортировать по "sort -n -k2" для получения стека и топа медленных операций:
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

Прежде чем показать, как это может выглядеть в более презентативной форме, сделаем небольшое отступление и для бОльшей детализации побудем в качестве хранителей времени и окунемся в макромир, растянув время не хуже черных дыр.

## Замедляем все живое

Для бОльшей детализации, мы постараемся искусственно затормозить CPU time, чтобы более детально и наглядно получить представления о наиболее "тяжелых" функциях. 
Сразу следует сказать, что это не всегда хороший и правильный метод. Его следует относить к методам, которые могут порождать ["Гейзенбаги"](https://ru.wikipedia.org/wiki/%D0%93%D0%B5%D0%B9%D0%B7%D0%B5%D0%BD%D0%B1%D0%B0%D0%B3) - жаргонный термин 
для программной ошибки, которая исчезает или меняет свое поведение, когда вы пытаетесь ее обнаружить или изучить. Другими словами, результаты трассировок и 
работа приложения в целом не всегда будет соотноситься с тем, как ведет себя ПО без дефицита ресурса по CPU, что может давать искаженные результаты и 
некорректные выводы об узких местах. Тем не менее, CBSD шелл скрипты достаточно просты и прямолинейны и методика подходит.

### Метод 1: замедляем ядро FreeBSD

Начнем с кейса, когда гостевая система, в которой проводится профилирование, работает под управлением FreeBSD (разумеется, с ядром Linux можно провернуть те же самые махинации). 
Мы введем искусственное замедление прямо в "сердце" планировщика, в файл /usr/src/sys/kern/sched_ule.c, в функцию *sched_switch* и вставим простейший DELAY:
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

Скомпилируем ядро и перезагрузимся. Как только инициируется ядро, все последующие операции будут напоминать работу ОС на условном 386 процессоре, 
где даже загрузка и вывод системных сообщений могут проходить с эффектом медленного построчного скроллинга. 
В качестве "подопытного" скрипта, возьмем функцию создания виртуальной машины bhyve в CBSD 15.0.4 и посмотрим ее "в разрезе".

Выполнение скрипта: *220* секунд.

Последние строчки по *FUNCTION_TIME*:
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


Это означает, что операции update_tables и merge_profiles достаточно тяжеловесны для операции `bcreate` и составляют >80% времени и по возможности, их следует оптимизировать.
При рассмотрении merge_profiles видно, что функции вызывают большое количество grep в цикле из всех файлов профилей для объединения параметров. Это достаточно простая операция для
небольшого кода на Си, поэтому в рамках оптимизации, мы заменили shell-функцию с большим количеством grep на один универсальный си-код. Проблемы update_table здесь
заключаются в том, что в процессе создания виртуальной машины, SQL клиент вызывается последовательнно больше количество раз - сколько таблиц, столько и вызовов. Мы также
можем здесь провести оптимизацию и выполнить транзакцией атомарно все операции за один раз.

Вывод в формате prometheus позволяет нам достаточно наглядно получить профилирование в виде графика в Grafana:

<center><img src="https://convectix.com/img/bcreate-cixtime1.png" width="1024" title="bcreate-cixtime" alt="bcreate-cixtime"/></center>


А если мы напишем небольшой конвертер в формат, понятный flamegraph:
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

            # Фильтрация "Include" (белый список)
            if args.include and not any(re.search(inc, name) for inc in args.include):
                continue

            # Фильтрация "Exclude" (черный список)
            if args.exclude and any(re.search(exc, name) for exc in args.exclude):
                continue

            all_data.append((name, val))

    if not all_data:
        print("No data left after filtering.", file=sys.stderr)
        return

    # Авто-определение иерархии (верхние уровни стека)
    sorted_all = sorted(all_data, key=lambda x: x[1], reverse=True)
    hierarchy = []
    seen = set()

    # Считаем родительскими функциями те, что занимают >50% времени от самой тяжелой
    max_time = sorted_all[0][1]
    for name, val in sorted_all:
        if name not in seen and val > (max_time * 0.5):
            hierarchy.append(name)
            seen.add(name)
        if len(hierarchy) >= 3: break

    # Сборка "folded stacks"
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

То сможем это визуализировать в виде стека:

<center><img src="https://convectix.com/img/bcreate-cixtime1.svg" width="1024" title="bcreate-cixtime" alt="bcreate-cixtime"/></center>

После модификаций по оптимизации вышеописанных операций, версия скрипта `bcreate` для CBSD 15.0.5 уже дает следующие показатели по top-медленных функций:
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
И общее время выполнения снизилось до *146* секунд, это значит мы ускорили скрипт в 1.5 раза!

Если же мы вернемся на реальную, не замедленную систему, общее время выполнения скрипта становится *5* секунд.

### Метод 1: замедляем гипервизор

Предыдущий вариант демонстрировал замедление системы на примере ядра FreeBSD и это работало как на bare-metal, так и в виртуальной машине.
Однако, если вы производите подобные упражнения только с виртуальной машиной, более универсальным решением будет замедлить гипервизор.
В этом случае, нам не нужно модифицировать исходный код ядра запускаемой ОС.

Просто добавить искусственный delay по VM-exit недостаточно, поскольку это эффективно затормозит I/O-нагруженные ВМ, но если хост 
почти не выходит из VM-enter (например CPU-бенчмарк), эффект будет слабый и нужно добавить динамическую часть, в зависимости от того, 
как часто (сколько TSC-тактов) гость выполняет VM_exit. Чтобы не патчить и пересобирать каждый раз модуль vmm.ko, добавим sysctl-параметры 
для легкого регулирования задержкой со стороны хоста:
```
sysctl hw.vmm.vcpu_delay_us=100
sysctl hw.vmm.vcpu_slowdown_pct=100
```

Патч для FreeBSD 15.0-RELEASE:
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

Эффект будет аналогичен замедлению через код ядра, но благодаря применению на уровне гипервизора, возможность сократить
бурную жизнедеятельность систем теперь появилась для любых ОС.


