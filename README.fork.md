# Stela — fork AMD/Vulkan (SoulDotPrime)

Este repositório é um fork de [tylertms/Stela](https://github.com/tylertms/Stela).
O objectivo **não** é substituir o Stela: é acrescentar-lhe um caminho gráfico
para **GPUs AMD** (a começar pela Radeon RX 480, Polaris10/GFX8) e fazer o
arranque funcionar em hardware x86-64 real.

O backend **NVIDIA original fica intacto**. Nenhuma alteração deste fork muda o
comportamento do caminho NVK/NVIDIA.

## Diferenças em relação ao upstream

### 1. Build e distribuição com RADV/ACO (`cargo xtask dist --radv`)

`xtask dist --radv` gera uma imagem UEFI **separada**, `stela-radv.efi`, e um
`stela` (o instalador Linux/macOS) compilado com a feature `amdgpu-port`:

- `xtask/src/dist.rs` — `run(radv: bool)`; quando `radv`, compila o `stela-cli`
  com `--features amdgpu-port` e aponta `STELA_EFI` para a imagem RADV.
- `xtask/src/main.rs` — a flag `--radv` no subcomando `dist`.
- `xtask/src/boot.rs` — `boot::build(release, radv)` exige o driver `"amd"`
  (não `"radeon"`) quando `radv`.

Sem `--radv`, tudo se comporta como no upstream.

### 2. RADV e ACO compilados na imagem

A imagem RADV leva o driver Vulkan RADV do Mesa e o compilador ACO, e o
firmware da AMD, para que o Stela consiga levar uma GPU AMD até um pipeline
Metal. Confirma-se na imagem por `nm` (`radv_icdGetInstanceProcAddr`,
`aco_compile_shader`).

### 3. Testes de admissão GFX8 (`crates/stela-driver-gpu/src/metal/amd/tests.rs`)

Testes que provam que uma RX 480 (`1002:67df`) é admitida pela interface de
famílias do amdgpu: família `130` (`AMDGPU_FAMILY_VI`), arquitectura `VI`,
alinhamentos 256/256, página esparsa de 64 KB, `dma_bits == 40`.

> **O que estes testes não provam:** que um pipeline Metal corra de facto em
> GFX8. Pitch pré-GFX9 no addrlib, invocações fixas de 1024, `subgroupSize` 64
> e o caminho ACO mais antigo continuam por testar em hardware.

### 4. Entrada USB no arranque (`--usb-input`)

O Stela só ligava o hub de input do guest quando `--input` (virtio) era pedido,
e `mount.rs` cablava-o apenas na *última* controladora USB e *nunca na
primeira* — com uma só controladora, o hub nunca era cablado.

- `stela-boot-core/src/options.rs` (+`parse.rs`) — opção nova `--usb-input`,
  que abre os controladores xHCI do host e entrega ao guest os teclados e
  ratos USB. Distinta de `--input`, que continua a ser virtio.
- `stela-boot/src/engine/start.rs` — `--usb-input` entra no desempate dos
  dispositivos a abrir e é passado como `input` ao `mount::devices`.
- `stela-runtime/src/machine/mount.rs` — o hub de input passa a ser cablado
  também quando o DeviceTree coloca **uma só** controladora USB.

### 5. BSP não marcado pelo firmware (`stela-boot-core/src/topology.rs`)

O Stela exigia que o firmware marcasse o *bootstrap processor* (bit BSP do UEFI
MP Services) e abortava com
`native processor topology is ambiguous: firmware marks no bootstrap processor`.
O BIOS AMI da placa AMD 990FX lista os processadores e **nunca** marca esse
bit, o que parava o arranque antes do macOS.

Agora, quando nenhum processador está marcado, o Stela lê o APIC ID do
processador em que está a correr (`CPUID`) e identifica-o na lista do firmware —
o mesmo que o Linux faz com o `boot_cpu_physical_apicid`. Os APIC IDs continuam
a ser observados um a um, e quem responder com identidade diferente fica de
fora. `topology::running_apic_id`, `topology::select_naming` e
`topology::names_no_bootstrap_processor` são a implementação; há um teste
(`firmware_that_names_no_bootstrap_processor_still_has_one`).

## Hardware de referência (o alvo real deste fork)

| Parte | Modelo |
|---|---|
| CPU | AMD FX-8320E (Bulldozer/Piledriver, x86-64-v2, AVX sem AVX2) |
| Placa-mãe | AMD 990FX (UEFI AMI, CSM desligado) |
| RAM | 8 GB DDR3-1600 |
| GPU | PowerColor Red Dragon Radeon RX 480 8 GB (Polaris10, `1002:67df`, GFX8) |
| Driver Vulkan | Mesa RADV 25.2.8, Vulkan 1.4.318 |
| Disco | SATA em AHCI |

**Nunca se assume AVX2.** A CPU alvo tem AVX e não AVX2.

## Estado

Experimental. O upstream já avisa que o Stela é lento e pode não arrancar em
hardware suportado; este fork acrescenta-lhe um caminho gráfico que **nunca foi
testado numa GPU AMD em macOS**. Ver `RELATORIO-*.md` no workspace de
desenvolvimento para o que está medido e o que é hipótese.

## Instalar (como no upstream)

1. Compilar: `cargo xtask dist --radv` (produz `target/release/stela` e
   `target/boot/x86_64-unknown-uefi/release/stela-radv.efi`).
2. `sudo ./target/release/stela create /dev/sdX --installer online`
3. Arrancar o PC da pen, com Secure Boot desligado.
