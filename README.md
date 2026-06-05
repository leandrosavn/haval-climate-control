# Haval Climate Control

Aplicativo Android de controle climático inteligente para veículos Haval, desenvolvido para rodar no display multimídia integrado do veículo. O app intercepta e automatiza comandos do sistema HVAC com base em regras de temperatura, mantendo o habitáculo confortável sem intervenção manual.

---

## Como funciona

O app roda como um **serviço em foreground** que se comunica diretamente com o `IntelligentVehicleControlService` do veículo via SDK proprietário (Beantechs), usando [Shizuku](https://shizuku.rikka.app/) para obter as permissões de sistema necessárias. A cada mudança de propriedade do veículo, o serviço avalia as regras e envia comandos ao HVAC quando necessário.

A interface é um **HMI automotivo** em tela cheia (1792 × 660 dp, proporção ~2.5:1) construído em Jetpack Compose, com tema monocromático escuro e acento verde apenas para estados ativos.

---

## Regras implementadas

### Detecção de partida do carro

Quando o sensor de temperatura interna sai do estado offline (`87 °C` = carro desligado) para uma leitura real:

- O controle é **desarmado** na partida — o app **não liga o AC automaticamente**.
- **Proteção de 30 segundos**: nos primeiros 30 s após a partida o AC não é desligado pelas regras abaixo, mesmo que a temperatura interna esteja abaixo do setpoint.

---

### Trava geral — controle "armado"

Enquanto o AC estiver **desligado e o controle não estiver armado**, o app **não altera nada** (AC, curva de conforto ou aquecimento) — apenas espelha o estado na tela.

O controle **arma** assim que o AC é ligado, seja porque já veio ligado na partida ou porque o usuário ligou depois. A partir daí, os blocos abaixo passam a atuar.

---

### Bloco A — Controle do AC *(requer controle armado + `car.hvac.auto_enable = 1`)*

Gerencia o liga/desliga do compressor (`car.hvac.ac_enable`) com base na diferença entre temperatura interna e setpoint do motorista (`car.hvac.driver_temperature`).

#### Desligar o AC

| Condição | Ação |
|---|---|
| Interna ≤ Setpoint − Histerese **e** AC ligado **e** fora da proteção de 30 s | Desliga AC |

#### Ligar o AC

| Condição | Ação |
|---|---|
| Interna ≥ Setpoint + 0,5 °C **e** AC desligado | Liga AC |
| Interna ≥ Setpoint **e** AC desligado por **mais de 1 minuto** | Liga AC |

#### Histerese dinâmica

| Temperatura externa | Histerese aplicada |
|---|---|
| ≤ 28 °C | **0,5 °C** |
| > 28 °C | **1,0 °C** (dias muito quentes — evita ciclos curtos) |

---

### Bloco A — Curva de conforto *(requer controle armado + `car.hvac.auto_enable = 1`)*

Ajusta a curva de conforto do HVAC (`car.hvac.setting.comfort_curve`) — que define a intensidade do fluxo de ar / agressividade com que o HVAC atinge a temperatura alvo — de acordo com o **modo de conforto** selecionado na tela:

| Modo | Curva enviada | Comportamento |
|---|---|---|
| **SUAVE** | `0` | fluxo suave, fixo |
| **NORMAL** | `1` | fluxo intermediário, fixo |
| **FORTE** | `2` | fluxo agressivo, fixo |
| **AUTO** | varia | escolhida pela **temperatura externa** (ver abaixo) |

No modo **AUTO**:

| Temperatura externa | Curva |
|---|---|
| ≥ 24 °C | `2` — agressiva |
| 19 °C – 24 °C | `1` — intermediária |
| < 19 °C | `0` — suave |

> O comportamento físico de cada curva (rotação do ventilador, distribuição de ar) é definido pelo firmware da GWM/Haval — o app apenas envia o valor `0/1/2`.

**Detecção de alteração externa** — se a curva for mudada pelo menu nativo do HVAC, o app detecta e passa para **modo manual**, parando de sobrescrever o valor.

---

### Ventilação dos bancos — sempre manual

O app **nunca** altera a ventilação dos bancos (`car.comfort_setting.driver_seat_ventilation_level` / `car.comfort_setting.passenger_seat_ventilation_level`). Os cards na tela são **somente leitura**, exibindo o nível atual definido manualmente pelo usuário.

> Em versões anteriores (≤ 1.10.4) havia um controle automático de ventilação dos bancos; ele foi **removido na v1.10.5**.

---

### Bloco C — Aquecimento por temperatura externa *(requer controle armado; independente do `auto_enable`)*

Controla o aquecimento (`car.hvac.heating_enable`) com base na temperatura externa:

| Temperatura externa | Ação |
|---|---|
| < 20 °C | `heating_enable = 1` — aquecimento ligado |
| ≥ 20 °C | `heating_enable = 0` — aquecimento desligado |

O comando só é enviado quando o valor muda (comparação com cache local).

---

## Interface

- **Tela principal** — layout HMI em 3 colunas: coluna esquerda (temperatura interna + status AC/aquecimento + fluxo de ar animado), coluna central (visualização top-down do carro + temperatura de setpoint + external), coluna direita (ventilação dos bancos — somente leitura — + logs de ações).
- **Toggle de controle automático** — desativa/reativa todas as regras de uma vez.
- **Seletor de modo de conforto** — alterna entre `AUTO`, `SUAVE`, `NORMAL` e `FORTE` (curva de conforto).
- **Log de ações** — histórico em tempo real com timestamp dos últimos 50 eventos (HVAC e bancos em abas separadas).
- **Auto-update** — verifica novas versões uma vez por dia via GitHub Releases API e oferece download direto na tela.

---

## Arquitetura

```
MainActivity (Compose HMI)
    │
    ├── ClimateStateHolder (Kotlin object — estado reativo compartilhado)
    │       ├── mutableStateOf → UI recompõe automaticamente
    │       └── commandCallback → envia comandos ao serviço
    │
    └── ClimateControlService (Foreground Service)
            ├── Shizuku → permissões de sistema
            ├── IIntelligentVehicleControlService → SDK Beantechs
            ├── vehicleDataListener → dispara evaluateClimateControl() a cada mudança
            └── evaluateClimateControl()
                    ├── Detecção de partida + trava geral (armar controle)
                    ├── Bloco A — AC + curva de conforto
                    └── Bloco C — Aquecimento
                    (ventilação dos bancos é sempre manual — somente leitura)
```

---

## Requisitos

- Display multimídia Haval com Android (testado no H6 GT 2023)
- [Shizuku](https://shizuku.rikka.app/) instalado e rodando
- Permissão `MANAGE_EXTERNAL_STORAGE` (para instalação de atualizações)

---

## Build & Release

O pipeline CI/CD roda no GitHub Actions ao **publicar uma tag `v*`** (ou via `workflow_dispatch` manual):

1. Lê `versionName` / `versionCode` do `app/build.gradle.kts`
2. Assina o APK com keystore via secrets
3. Publica GitHub Release com a tag `v{versionName}`

> Fluxo típico: faça commit em `master`, incremente a versão no `app/build.gradle.kts`, então crie e envie a tag (`git tag v1.x.y && git push origin v1.x.y`) para disparar o build.

O app verifica automaticamente a última release disponível e compara com a versão instalada para oferecer atualização.

---

## Versionamento

| Tipo de mudança | Incremento |
|---|---|
| Bug fix | patch (`1.0.x`) |
| Nova funcionalidade | minor (`1.x.0`) |
| Mudança incompatível | major (`x.0.0`) |

`versionCode` sempre +1 a cada release.
