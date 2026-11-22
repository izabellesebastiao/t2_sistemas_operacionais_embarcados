# Chamada de sistema `taskinfo()` no Apache NuttX

- Trabalho 2 – Sistemas Operacionais Embarcados (DEC7562).
- Plataformas: ESP32-C6-DevKitC (`esp32c6-devkitc`, RISC-V).

## 1. Funcionalidade

```c
#include <sys/taskinfo.h>

int taskinfo(pid_t pid, FAR struct taskinfo_s *info);
```

Devolve uma "foto" consistente de uma tarefa (`pid == 0` refere-se à tarefa atual) em `info`:

- PID
- nome
- estado (`TSTATE_*`)
- prioridade
- política (`SCHED_FIFO`/`SCHED_RR`/...)
- tamanho da pilha
- pilha usada (marca d'água)
- número de vezes que a tarefa foi escalonada (`nswitches`)

Retorna 0, ou -1 com `errno`:

| Situação                             | errno    |
| ------------------------------------ | -------- |
| `info == NULL` ou `pid < 0`          | `EINVAL` |
| PID inexistente / tarefa já terminou | `ESRCH`  |

## 2. Arquivos do NuttX modificados ou adicionados

Repositório `nuttxspace/nuttx`:

| Arquivo                                                                                                                            | Mudança                                                                    |
| ---------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `include/sys/taskinfo.h`                                                                                                           | **novo** – `struct taskinfo_s` e protótipo                                 |
| `sched/task/task_info.c`                                                                                                           | **novo** – implementação da chamada                                        |
| `include/nuttx/sched.h`                                                                                                            | campo `uint32_t nswitches` em `struct tcb_s` (sob `CONFIG_SCHED_TASKINFO`) |
| `sched/sched/sched_switchcontext.c`                                                                                                | `to->nswitches++` em `nxsched_switch_context()`                            |
| `sched/Kconfig`                                                                                                                    | nova opção `CONFIG_SCHED_TASKINFO`                                         |
| `sched/task/Make.defs`, `sched/task/CMakeLists.txt`                                                                                | compilam `task_info.c` quando a opção está ligada                          |
| `syscall/syscall.csv`                                                                                                              | registro `"taskinfo"` (gera os stubs nos modelos PROTECTED/KERNEL)         |
| `boards/risc-v/esp32c6/esp32c6-devkitc/configs/taskinfo/defconfig`, `boards/xtensa/esp32/esp32-devkitc/configs/taskinfo/defconfig` | **novo** – nsh + `SCHED_TASKINFO` + `STACK_COLORATION` + app de teste      |

Repositório `nuttxspace/apps`:

| Arquivo                                                                                                  | Mudança                       |
| -------------------------------------------------------------------------------------------------------- | ----------------------------- |
| `examples/taskinfo_test/` (`Kconfig`, `Make.defs`, `Makefile`, `CMakeLists.txt`, `taskinfo_test_main.c`) | **novo** – aplicação de teste |

## 3. Estruturas e funções internas utilizadas

- `struct tcb_s` (TCB): origem de `task_state`, `sched_priority`, `flags` (bits de política), `adj_stack_size`, `name` e do novo `nswitches`.
- `nxsched_get_tcb(pid)`: busca o TCB na tabela hash de PIDs. `this_task()`: TCB da tarefa atual (`pid == 0`).
- `enter_critical_section()` / `leave_critical_section()`: protegem a leitura (ver abaixo).
- `up_check_tcbstack()`: varre a pilha atrás da marca de coloração (`CONFIG_STACK_COLORATION`) e informa o uso máximo. Sem essa opção, `stack_used` vale 0.
- `nxsched_switch_context()`: chamada a cada troca de contexto; é onde o contador `nswitches` da tarefa que entra é incrementado.

**Sincronização.** A busca do TCB e a cópia dos campos para uma estrutura local acontecem dentro de uma seção crítica; só depois de sair dela o resultado é copiado para `info`. Em relação ao rascunho da proposta, usei `enter_critical_section()` em vez de `sched_lock()`: no ESP32 (2 núcleos, SMP) `sched_lock()` não impede que _outro_ núcleo apague a tarefa durante a leitura, e a seção crítica impede. A região é curta (cópia de alguns campos e a varredura da pilha).

## 4. Compilação e execução

Pré-requisitos: toolchain no `PATH` (`riscv-none-elf-gcc` para o ESP32-C6, `xtensa-esp32-elf-gcc` para o ESP32), `esptool`, `kconfig-frontends`.

```sh
cd nuttxspace/nuttx
export PATH=$HOME/toolchains/riscv-none-elf-gcc/bin:$PATH      # ESP32-C6
./tools/configure.sh -l esp32c6-devkitc:taskinfo               # ESP32: esp32-devkitc:taskinfo
make -j8
esptool --chip esp32c6 --port /dev/ttyUSB0 write-flash 0x0 nuttx.bin   # ver nota abaixo
```

Se preferir partir de `esp32c6-devkitc:nsh`, ligue por `make menuconfig`: _Tasks and Scheduling → Enable taskinfo() system call_,
_Build Setup → Debug Options → Stack coloration_ e _Application Configuration → Examples → taskinfo() system call test_.

(Ou use `make flash ESPTOOL_PORT=/dev/ttyUSB0`.) Abra o console serial (115200 8N1) e, no NSH:

```
nsh> taskinfo_test
```

### Testando sem a placa (simulador `sim`)

O NuttX tem um simulador que roda como processo Linux; o código novo do kernel é independente de arquitetura.

```sh
cd nuttxspace/nuttx
make distclean
./tools/configure.sh -l sim:nsh
for o in SCHED_TASKINFO STACK_COLORATION EXAMPLES_TASKINFO_TEST SCHED_WAITPID; do kconfig-tweak --enable $o; done
kconfig-tweak --set-val RR_INTERVAL 200
make olddefconfig && make -j8
./nuttx        # no prompt nsh>, digite: taskinfo_test
```

## 5. Testes (`apps/examples/taskinfo_test`)

| #   | Teste                                                  | Esperado                                                                                       |
| --- | ------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| 1   | `taskinfo(0)` e `taskinfo(getpid())`                   | 0; pid correto; estado RUNNING; `0 < stack_used <= stack_size`; ambos concordam                |
| 2   | tarefa `sleeper` (prio 90)                             | nome, prioridade, estado SLEEPING, política herdada (FIFO ou RR), pilha ≥ 2048, `switches ≥ 1` |
| 3   | `sched_setscheduler()` para FIFO/90 e depois RR/91     | política e prioridade refletidas                                                               |
| 4   | `usleep()` e nova chamada                              | `switches` aumenta                                                                             |
| 5   | tarefa nova mede `stack_used` antes/depois de recursão | `stack_used` aumenta (a marca d'água só cresce, por isso a tarefa é nova)                      |
| 6   | `info == NULL`, `pid < 0`, PID inexistente             | -1 com `EINVAL`/`EINVAL`/`ESRCH`; `info` intacto                                               |
| 7   | `task_delete()` e nova chamada                         | -1/`ESRCH`                                                                                     |

**Resultados.** O projeto compila sem erros para `esp32c6-devkitc:taskinfo` e `esp32-devkitc:taskinfo` (gera `nuttx.bin`),
e os arquivos passam no `tools/checkpatch.sh`.

Execução no simulador (`sim:nsh`, ver acima) — **31 aprovados, 0 falhas**:

```
taskinfo() test

[1] Calling task
  [PASS] taskinfo(0) returns 0
    pid=8 name="taskinfo_test" state=RUNNING prio=100 policy=SCHED_RR
    stack=69552 used=1816 switches=1
  [PASS] pid == getpid()
  [PASS] caller is RUNNING
  [PASS] stack_size > 0
  [PASS] 0 < stack_used <= stack_size
  [PASS] taskinfo(getpid()) returns 0
  [PASS] pid==0 and pid==getpid() agree

[2] Another task
  [PASS] task_create(sleeper)
  [PASS] taskinfo(sleeper) returns 0
    pid=9 name="sleeper" state=SLEEPING prio=90 policy=SCHED_RR
    stack=67504 used=888 switches=1
  [PASS] pid matches
  [PASS] name == "sleeper"
  [PASS] priority == 90
  [PASS] state is SLEEPING
  [PASS] policy is SCHED_FIFO or SCHED_RR (inherited)
  [PASS] stack_size >= 2048
  [PASS] switches >= 1

[3] Policy and priority change
  [PASS] sched_setscheduler(SCHED_FIFO, 90)
  [PASS] policy == SCHED_FIFO
  [PASS] sched_setscheduler(SCHED_RR, 91)
  [PASS] policy == SCHED_RR
  [PASS] priority == 91

[4] Context switch counter
  [PASS] caller's switches grows after usleep()
    2 -> 3

[5] Stack usage
  [PASS] task_create(worker)
  [PASS] worker finished
  [PASS] stack_used grows after deeper calls
    600 -> 2144

[6] Errors
  [PASS] info == NULL -> -1/EINVAL
  [PASS] pid < 0 -> -1/EINVAL
  [PASS] nonexistent pid -> -1/ESRCH
  [PASS] info untouched on error

[7] Terminated task
  [PASS] task_delete(sleeper)
  [PASS] deleted task -> -1/ESRCH

Result: 31 passed, 0 failed
```
