# SGMAtmosphere USB — Porting & Upgrade Notes

Este documento registra a arquitetura USB validada no SGMAtmosphere e serve como
referência permanente para futuras atualizações do Atmosphère/Nexus.

> Objetivo: manter o SGMAtmosphere com suporte a armazenamento/conteúdo USB,
> preservando a identidade SGM, a splash original do Atmosphère e as validações
> normais do Atmosphère/Horizon.

## 1. Estado validado

Base usada no primeiro port funcional:

- Atmosphère: 1.12.2
- Horizon OS suportado pela base: 23.5.0
- Nexus Atmosphere:
  - repositório: `TR4UNEXUS/Nexus-Atmosphere`
  - commit de referência: `c246c2ee3866c4897cb5741f1cbf5026a1000582`
- Nexus release de referência:
  - tag: `v1.12.2-n1`
  - asset: `Nexus-1.12.2-N1-No-HOC.zip`
  - SHA-256: `3bbedb949c4dbaf632b3f5abf31d9eb836d9e08e12eebea223bbe8e566499bdf`

Testes funcionais já realizados:

1. Firmware inicia normalmente com backend USB ativo.
2. Nexus Manager reconhece o dispositivo USB.
3. Jogo previamente instalado no SD foi movido para USB e executou normalmente.
4. Ao remover o USB, o mesmo jogo deixou de abrir, confirmando leitura real do USB.
5. Stardew Valley foi instalado diretamente no USB e executou normalmente.
6. Vampire Survivors instalou, mas falhou no launch em um teste isolado. Como outros
   títulos funcionaram no mesmo fluxo, esse caso não caracteriza falha geral do backend.

Testes com software comercial devem usar conteúdo legalmente possuído/obtido pelo usuário.

## 2. Política de identidade SGM

### Splash

A splash deve permanecer **100% original do Atmosphère**.

Não usar:

- splash Nexus;
- splash SGM personalizada;
- splash proveniente de packs HATS/CNX.

Ao portar uma nova versão, preservar/restaurar da tag oficial correspondente:

- `img/splash.bin`
- `stratosphere/boot/source/boot_splash_screen.cpp`
- `stratosphere/boot/source/boot_splash_screen.hpp`
- arquivos `boot_splash_*.inc` oficiais correspondentes.

O Nexus também possui splash compilada em `boot.kip`. Portanto, um `boot.kip`
pré-compilado do Nexus pode reintroduzir a identidade visual Nexus mesmo com
`img/splash.bin` stock.

### Branding em Configurações -> Sistema

Arquivo:

`stratosphere/ams_mitm/source/set_mitm/setsys_mitm_service.cpp`

Valor esperado:

```cpp
constexpr const char package_version[] = "SGM";
```

Não manter `NEXUS` no texto apresentado ao usuário.

## 3. Arquitetura USB a preservar

A solução Nexus não é apenas um driver USB. O launch depende de alterações
coordenadas em vários subsistemas.

### 3.1 USB / libstratosphere

Preservar as adições relevantes em:

- `libraries/libstratosphere/include/stratosphere/usb/`
- `libraries/libstratosphere/source/usb/`

Essa camada fornece transporte, filesystem/volume USB e o proxy de conteúdo.

### 3.2 ams_mitm

Preservar o backend USB em:

- `stratosphere/ams_mitm/source/usb_mitm/`

Também existem integrações relevantes em:

- `stratosphere/ams_mitm/source/fs_mitm/`
- `stratosphere/ams_mitm/source/ns_mitm/`

O `ams_mitm` fornece o serviço utilizado pelo route de conteúdo. Não remover
essas integrações só porque o drive já é detectado.

### 3.3 NCM

Preservar armazenamento Host/USB e mapeamento persistente.

Arquivos/subsistemas importantes incluem:

- `libraries/libstratosphere/source/ncm/ncm_host_content_storage_impl.cpp`
- `libraries/libstratosphere/source/ncm/ncm_usb_title_map.cpp`
- alterações relacionadas no content manager/install task.

Mapas persistentes:

`SD:/atmosphere/usb_content/`

Arquivos esperados incluem:

- `title_map.bin`
- `title_content_map.bin`
- `content_locations.bin`
- temporários/backups/repair relacionados.

Esses mapas são hints de roteamento. Eles não substituem metadata/tickets/rights
normais do Horizon.

### 3.4 LR

Preservar a resolução/conversão de localização USB, incluindo:

- `libraries/libstratosphere/source/lr/lr_content_location_resolver_impl.cpp`

LR precisa manter identidade canônica compatível com o launch normal enquanto o
conteúdo físico é redirecionado para USB.

### 3.5 Loader

Preservar a integração USB do Loader, principalmente:

- `stratosphere/loader/source/ldr_content_management.cpp`
- auxiliares de seleção/trace USB da mesma árvore.

Loader resolve o título por NCM, traduz a localização USB e abre o filesystem de
código mantendo o fluxo normal de verificação.

### 3.6 FS / launch bridge

Ao atualizar, comparar cuidadosamente alterações Nexus em:

- FS
- NCM
- LR
- Loader
- ams_mitm
- libstratosphere USB

Não tratar a solução como somente “NCM + driver USB”.

## 4. Configuração SGM USB

Arquivo:

`SD:/atmosphere/config/system_settings.ini`

```ini
[atmosphere]
enable_usb_storage = u8!0x1
enable_usb_content = u8!0x1
enable_usb_content_route = u8!0x1

enable_usb_content_native_path = u8!0x0
enable_usb_content_force_path = u8!0x0
enable_usb_content_force_metadata = u8!0x0
enable_usb_external_kip = u8!0x0
```

`enable_usb_external_kip` deve permanecer desligado enquanto o provider estiver
integrado ao `ams_mitm`.

## 5. O que NÃO importar do Nexus/CNX/HATS

Não fazer cherry-pick cego do Nexus ou CNX.

Não trazer automaticamente:

- HOC/overclock;
- temas;
- extras de packs HATS;
- sys-patch;
- módulos opcionais não necessários ao backend USB;
- splash Nexus;
- branding Nexus;
- patches que desabilitem verificação de assinatura/rights/NPDM/NCA.

A árvore Nexus usada como referência possuía alterações em
`fusee/program/source/fusee_stratosphere.cpp` relacionadas a bypass de
verificação. Essas alterações **não fazem parte do SGM USB**.

Sempre restaurar/auditar `fusee` contra a tag oficial do Atmosphère da mesma versão.

Auditoria usada no port 1.12.2:

```sh
grep -R "NoNcaHeaderSignatureCheckPatch" -n fusee/program/source
```

Resultado esperado no SGM: vazio.

## 6. Limite fixo do package3

Região de KIPs:

```text
0x100000 -> 0x400000
0x300000 = 3 MiB = 3145728 bytes
```

`fusee/build_package3.py` rejeita a geração se a soma ultrapassar `0x300000`.

**Não aumentar esse limite.**

### Medição inicial no Windows/MSYS

```text
emummc       160188
Loader       213020
NCM          230624
PM            49852
sm            31952
boot        1660584
spl           53568
ams_mitm      775280

TOTAL       3175088
OVER          29360
```

KIPs pré-compilados Nexus 1.12.2 N1 de referência:

```text
Loader          205180
NCM             224748
ProcessManager   46132
sm               31056
boot           1660636
spl              49500
ams_mitm        744160
```

Após otimização direcionada, o `ams_mitm.kip` local SGM chegou a:

```text
686848 bytes
```

## 7. Estratégia de build

A documentação Nexus 1.12.2 N1 usa GCC 15.2.0 e recomenda Linux/WSL + devkitPro.

No Windows/MSYS:

- evitar `make clean` global;
- limpar somente o módulo necessário;
- usar `-Os` nos módulos/translation units grandes quando necessário;
- medir KIPs antes de regenerar tudo.

O tamanho `686848` do `ams_mitm.kip` é referência apenas para 1.12.2.

### boot

Para release SGM, usar `boot.kip` compilado da árvore SGM com splash restaurada
da tag oficial Atmosphère.

Evitar `boot.kip` Nexus na release final.

### Fallback de desenvolvimento validado

Durante o port 1.12.2, para evitar recompilações completas no Windows:

- KIPs Nexus No-HOC para Loader/NCM/PM/sm/spl;
- `boot.kip` local/stock para manter splash Atmosphère;
- `ams_mitm.kip` local para branding SGM;
- `package3` gerado pelo builder SGM.

Em Linux/WSL com toolchain correspondente, preferir build completa da árvore SGM.

## 8. Compatibilidades SGM existentes

Ao atualizar o USB para nova base, preservar compatibilidades SGM ainda necessárias.

Na base Nexus 1.12.2 foi observado comportamento equivalente às compatibilidades
históricas SGM relacionadas a:

- DebugFlags antigos;
- comportamento TLS necessário para homebrews legados.

Não reaplicar commits antigos cegamente sem comparar a base nova.

## 9. Procedimento para futuras atualizações

Quando sair Atmosphère/Nexus novo:

1. Identificar a tag oficial Atmosphère alvo.
2. Identificar o commit/tag Nexus correspondente.
3. Criar branch de port SGM.
4. Comparar Nexus contra Atmosphère oficial da mesma versão.
5. Separar alterações por subsistema:
   - USB/libstratosphere
   - ams_mitm
   - FS
   - NCM
   - LR
   - Loader
6. Importar somente o necessário ao backend USB.
7. Restaurar splash oficial Atmosphère.
8. Aplicar branding `SGM`.
9. Auditar `fusee` e remover qualquer bypass de verificação.
10. Preservar compatibilidades SGM necessárias.
11. Compilar.
12. Medir KIPs e confirmar `TOTAL <= 0x300000`.
13. Gerar `package3`.
14. Configurar flags USB.
15. Executar checklist funcional.

Não reutilizar cegamente números de versão passados para
`fusee/build_package3.py`; obter valores da nova árvore/build system.

## 10. Checklist de release

### Boot

- [ ] Boot via Hekate/emuMMC.
- [ ] Splash original Atmosphère.
- [ ] Configurações -> Sistema mostra `SGM`, não `NEXUS`.
- [ ] HOME inicia sem USB.

### USB

- [ ] Dispositivo USB reconhecido.
- [ ] `ams:usb` disponível.
- [ ] Manager lista armazenamento.
- [ ] `atmosphere/usb_content/` é atualizado.

### Launch

- [ ] Jogo conhecido abre no SD.
- [ ] Mover SD -> USB.
- [ ] Mesmo jogo abre pelo USB.
- [ ] Remover USB impede launch daquele conteúdo.
- [ ] Instalação direta USB de título conhecido/legítimo funciona.
- [ ] Título instalado diretamente abre pelo USB.

### Robustez

Validar a cada release:

- [ ] sleep/wake com USB;
- [ ] reconexão após desconectar/reconectar;
- [ ] múltiplos títulos;
- [ ] reboot frio;
- [ ] boot sem drive;
- [ ] atualização de HOS/firmware.

## 11. Logs úteis

Diretório:

`SD:/atmosphere/logs/`

Arquivos úteis podem incluir:

- `usb_storage_status.txt`
- `usb_loader_trace.txt`
- `usb_ncm_launch_trace.txt`
- outros `usb_*.txt`

Em regressões, coletar logs antes de alterar o backend.

## 12. Regra principal

```text
Atmosphère oficial da nova versão
        +
somente mudanças USB necessárias do Nexus correspondente
        +
compatibilidades SGM ainda necessárias
        +
branding SGM
        +
splash original Atmosphère
        =
SGMAtmosphere USB
```

Manter o menor delta possível em relação ao Atmosphère oficial para facilitar
auditoria, manutenção e upgrades futuros.
