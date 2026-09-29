# SGMAtmosphere 1.12.0

Base:
- Atmosphere 1.12.0 oficial
- Upstream commit: 28d6a2e11f2640077bbb8a9fbd463d3fa12b6984
- Horizon OS: 23.0.0

## Alterações SGM

### 1. Branding SGM

Arquivo:

stratosphere/ams_mitm/source/set_mitm/setsys_mitm_service.cpp

Adiciona `SGM` à versão exibida pelo sistema.

Foi mantido o buffer `display_version` com o mesmo tamanho do destino para
evitar escrita além do campo de firmware version.

Formato:

HOS | SGM | AMS 1.12.0|E/S


### 2. Compatibilidade DebugFlags

Arquivo:

stratosphere/loader/source/ldr_capabilities.cpp

Em `PreProcessCapability()`:

- detecta CapabilityId::DebugFlags;
- conta AllowDebug / ForceDebugProd / ForceDebug;
- quando mais de uma flag está ativa, normaliza para:

CapabilityDebugFlags::Encode(false, false, true)

Objetivo:

Compatibilidade com homebrews/forwarders antigos, incluindo cenários usados por:
- Tinfoil
- RetroArch
- forwarders antigos

O `FixDebugCapabilityForHbl()` oficial do Atmosphere 1.12.0 foi preservado.


### 3. Compatibilidade TLS antiga

Arquivo:

libraries/libmesosphere/source/kern_k_scheduler.cpp

O Atmosphere oficial atualiza `thread_cpu_time` dentro da Thread Local Region
durante a troca de threads.

Para preservar compatibilidade com homebrew compilado contra o ABI TLS antigo
da libnx, o SGM não faz essa escrita.

Código SGM:

cpu::SwitchThreadLocalRegion(
    GetInteger(next_thread->GetThreadLocalRegionAddress())
);

Esta alteração foi necessária para o Tinfoil voltar a funcionar no HOS 23.0.0.

Teste realizado:

- Atmosphere/SGM inicia normalmente
- RetroArch funciona
- Tinfoil funciona após aplicar a compatibilidade TLS
- HOS 23.0.0


## Histórico

SGMAtmosphere 1.11.2 já possuía as alterações de compatibilidade usadas como
referência para esta migração.

Na migração inicial para 1.12.0 foram aplicados Branding + DebugFlags.

O Tinfoil ainda fechava imediatamente.

Comparando o SGMAtmosphere 1.11.2 com o Atmosphere oficial foi identificada a
alteração adicional em `kern_k_scheduler.cpp`.

Após restaurar essa alteração TLS no 1.12.0, o Tinfoil voltou a funcionar.
