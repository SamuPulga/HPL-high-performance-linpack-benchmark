# Benchmark HPL — HP EliteBook 630 G9

Implementación y optimización de **HPL (High-Performance Linpack)** — el mismo software usado para construir el ranking oficial [Top500](https://www.top500.org/) de supercomputadoras — corrido sobre un solo nodo: mi propia laptop de uso diario.

Proyecto académico (curso de Computación de Alto Rendimiento, EAFIT), con entrega comparable frente a los resultados de mis compañeros de curso en sus propios equipos.

**Resultado final: `138.94 GFLOPS` (Rmax) — `72.0%` de eficiencia respecto al pico teórico.**

---

##  Hardware

| | |
|---|---|
| Equipo | HP EliteBook 630 13" G9 |
| CPU | Intel Core i7-1255U (Alder Lake, híbrida): 2 P-cores (4 hilos, hasta 4.7 GHz) + 8 E-cores (8 hilos, hasta 3.5 GHz) — **10 núcleos físicos / 12 hilos** |
| Vectorización | AVX2 (AVX-512 deshabilitado de fábrica en toda la línea) |
| TDP | 15W |
| RAM | 32 GB |
| Red | Un solo nodo — sin clúster. WiFi solo para SSH remoto, no para cómputo distribuido |

Elegido deliberadamente como nodo único (en vez de clúster) para tener control total del hardware y más RAM disponible por proceso.

##  Stack técnico

| Herramienta | Rol |
|---|---|
| Ubuntu Server 24.04 LTS | SO base, instalado nativo sin GUI |
| Ansible + [`top500-benchmark`](https://github.com/geerlingguy/top500-benchmark) | Automatización de la compilación completa (MPI + BLAS + HPL) |
| MPICH | Paralelización de HPL entre los 12 hilos |
| BLIS 3.0 | Librería BLAS ganadora |
| Intel MKL / OpenBLAS | Alternativas evaluadas y descartadas |
| GCC 15.2.0 / gfortran | Compilación (`-O3 -march=native -mtune=native -flto=auto`) |
| `turbostat` | Medición de frecuencia/consumo real, clave para calcular Rpeak con rigor |
| tmux | Persistencia de sesiones SSH ante cortes de conexión |

##  Proceso

1. Instalación nativa de Ubuntu Server (USB con Rufus, particionado completo, sin GUI).
2. Compilación automatizada vía Ansible (`top500-benchmark`), con banderas de optimización a medida.
3. Validación del pipeline con corridas iniciales (residuales `PASSED`).
4. Exploración sistemática de parámetros, variando una variable a la vez:
   - Tamaño de matriz `N` (40,000–55,000, límite ~95% de RAM para evitar swap)
   - Grilla de procesos `P×Q` (1×12, 2×6, 3×4)
   - Tamaño de bloque `NB` (192, 256, 384)
   - Factorización (`PFACT`, `RFACT`) y broadcast (`BCAST`)
5. Comparación de librerías BLAS (BLIS vs Intel MKL vs OpenBLAS), recompilando el stack completo para cada una.
6. Intentos de tuning a nivel de hardware: CPU governor, límites RAPL, undervolting, Transparent Huge Pages.
7. Cálculo iterativo de Rpeak, refinado en 4 etapas hasta una metodología rigurosa (frecuencias reales medidas con `turbostat` durante la corrida exacta que produjo el Rmax final).

##  Problemas encontrados

| Problema | Causa | Solución |
|---|---|---|
| `Need at least 12 processes` recurrente | Ansible detecta mal el conteo de núcleos en la CPU híbrida P/E-core | Corrección manual con `sed` tras cada recompilación |
| Batería agotada a mitad de prueba | Cargador desconectado en corrida larga | Verificación previa de carga + `tmux` para no perder progreso |
| WiFi bloqueado sin razón aparente | `rfkill` soft-block activo | Desbloqueo manual vía sysfs |
| Reconexión al cambiar de red | Red universitaria usa WPA2-Enterprise (PEAP/MSCHAPv2), no WPA2 simple | Configuración EAP adicional en netplan |
| MKL/OpenBLAS muy por debajo de BLIS | Sospecha de mal mapeo de procesos en la arquitectura híbrida (no confirmado al 100%) | Se descartaron ambas; BLIS quedó como definitiva |
| Undervolt no se aplicaba | Mitigación de firmware contra Plundervolt (CVE-2019-11157), persistente incluso sin Secure Boot | Sin solución legítima disponible — línea cerrada por decisión consciente |
| Estimaciones de Rpeak inconsistentes (125→175→~300 GFLOPS) | Ventanas de muestreo cortas o no representativas | Metodología final: 44 lecturas de `turbostat` durante la corrida exacta del Rmax |

##  Resultados

**Configuración ganadora:** `N=45000` `NB=256` `P=3` `Q=4` · `PFACT=Right` `RFACT=Crout` `BCAST=1ringM` · BLIS 3.0 · GCC 15.2.0

| Métrica | Valor |
|---|---|
| **Rmax** | 138.94 GFLOPS |
| **Rpeak** | 192.99 GFLOPS |
| **Eficiencia** | 72.0% |

**Comparaciones documentadas:**

- **Grilla P×Q:** `P=3,Q=4` (138.94) > `P=1,Q=12` (~115–119) > `P=2,Q=6` (103.80) — resultado contraintuitivo, se esperaba que la grilla "plana" ganara en un solo nodo.
- **Librerías BLAS:** BLIS (~138) > Intel MKL (~72) > OpenBLAS (~52) — MKL, siendo la librería propia de Intel, tuvo el peor desempeño de las tres.
- **NB:** 256 superó consistentemente a 192 y 384.
- Ajustes de hardware (governor, límites de potencia) sin mejora medible; Transparent Huge Pages dio una mejora marginal (136.85 → 138.94 GFLOPS).

##  Trabajo futuro

- [ ] Probar librería ATLAS (tercera opción soportada, nunca evaluada)
- [ ] Barrido más fino de `N` alrededor de 45,000
- [ ] Explorar `NBMIN`, `SWAP threshold`, `DEPTH`
- [ ] Promediar la configuración ganadora sobre múltiples corridas (se observó variación de hasta ~30 GFLOPS entre corridas idénticas)
- [ ] Confirmar la causa raíz del bajo rendimiento de Intel MKL en arquitectura híbrida

---

*Proyecto académico — Ingeniería de Sistemas, Universidad EAFIT.*
