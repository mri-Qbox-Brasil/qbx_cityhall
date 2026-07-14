# qbx_cityhall — Manual

Prefeitura com NPC atendente: emite carteiras físicas (identidade, habilitação, porte de arma) e, opcionalmente, permite ao jogador escolher um emprego civil.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Emissão de licenças](#emissão-de-licenças)
5. [Menu de empregos](#menu-de-empregos)
6. [Integrações](#integrações)
7. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
8. [Localização](#localização)
9. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qbx_core` | Sim | `GetPlayer`, `SetJob`, `RemoveMoney`, notificações, `playerdata` |
| `ox_lib` | Sim | Callbacks, context menu, zonas, `showTextUI`, locale |
| `qbx_idcard` | Sim | `CreateMetaLicense` — cria o item físico da licença |
| `ox_target` | Não | Só quando `useTarget = true` no `config/client.lua` |
| `ox_inventory` | Não | Indiretamente, via `qbx_idcard`, para receber o item |

---

## Instalação

1. Copie a pasta `qbx_cityhall` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure qbx_cityhall
   ```
3. Os itens `id_card`, `driver_license` e `weaponlicense` precisam existir no `ox_inventory`. Eles já vêm no conjunto padrão do `qbx_idcard`.

Não há SQL. Não há conflitos conhecidos.

---

## Configuração

### `config/shared.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `cityhalls` | array | Sim | Lista de prefeituras. O jogador sempre interage com a mais próxima |
| `cityhalls[i].coords` | vec3 | Sim | Centro da prefeitura. O servidor valida distância de até 20 metros ao aplicar para um emprego |
| `cityhalls[i].showBlip` | bool | Sim | Habilita o blip no mapa. Só tem efeito se `blip` também estiver definido |
| `cityhalls[i].blip` | table | Não | `label`, `shortRange`, `sprite`, `display`, `scale`, `colour`. Vem comentado por padrão — sem ele nenhum blip é criado, mesmo com `showBlip = true` |
| `cityhalls[i].licenses` | table | Sim | Mapa `chave -> { item, label, cost }` das licenças emitidas nessa prefeitura |
| `cityhalls[i].licenses[k].item` | string | Sim | Nome do item. O servidor só aceita `id_card`, `driver_license` ou `weaponlicense` |
| `cityhalls[i].licenses[k].label` | string | Sim | Nome exibido no menu |
| `cityhalls[i].licenses[k].cost` | number | Sim | Preço em dinheiro vivo (`cash`) |
| `employment.enabled` | bool | Sim | Liga/desliga o menu de empregos. Padrão: `false` |
| `employment.jobs` | table | Sim | Mapa `nomeDoJob -> Label` dos empregos disponíveis. O nome precisa existir no `qbx_core` |

Licenças padrão: `id` (`id_card`, $50), `driver` (`driver_license`, $50), `weapon` (`weaponlicense`, $50).

### `config/client.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `useTarget` | bool | Sim | `true` usa `ox_target` no NPC; `false` usa zona box com tecla `E`. Padrão: `false` |
| `debugPoly` | bool | Sim | Debug de zonas |
| `peds` | array | Sim | NPCs atendentes |
| `peds[i].model` | string \| hash | Sim | Modelo do ped. Padrão: `a_m_m_hasjew_01` |
| `peds[i].coords` | vec4 | Sim | Posição e heading do ped |
| `peds[i].scenario` | string | Sim | Scenario tocado em loop. Padrão: `WORLD_HUMAN_STAND_MOBILE` |
| `peds[i].cityhall` | bool | Sim | Marca o ped como atendente de prefeitura |
| `peds[i].zoneOptions` | table | Não | `length`, `width`, `debugPoly`. Usado quando `useTarget = false`. Sem ele, a zona não é criada |

Os peds são spawnados no `OnPlayerLoaded` e no start do recurso, e removidos no unload/stop.

---

## Emissão de licenças

O menu "Identidade" **só lista as licenças que o jogador já possui na metadata** (`QBX.PlayerData.metadata.licences`). Ou seja, a prefeitura emite a carteira física de um direito que o personagem já tem — ela não concede o direito.

Ao selecionar uma licença:

1. O servidor valida que o `item` está na lista permitida (`id_card`, `driver_license`, `weaponlicense`).
2. Cobra `cost` em `cash`. Sem saldo, o pedido é recusado.
3. Chama `exports.qbx_idcard:CreateMetaLicense(source, item)` para gerar o item com os metadados do personagem.

---

## Menu de empregos

Aparece apenas quando `employment.enabled = true`. Lista os jobs de `employment.jobs` e aplica o escolhido no grade `0`.

O servidor valida, antes de aplicar:
- O job está em `employment.jobs`.
- O jogador está a menos de 20 metros da prefeitura mais próxima.

---

## Integrações

### qbx_idcard

Dependência efetiva da emissão de licenças. O `qbx_cityhall` cobra o valor e delega a criação do item para `exports.qbx_idcard:CreateMetaLicense`, que preenche os metadados (nome, data de nascimento, etc.) do personagem no item.

### ox_target

Quando `useTarget = true`, o NPC recebe uma opção de target (`open_cityhall<i>`, ícone `fa-solid fa-city`, distância 1.5). Quando `false`, o recurso cria uma zona box de 2x2x3 na posição do ped e exibe um TextUI; a interação é pela tecla `E` (controle 38).

---

## Entrypoints para outros recursos

O recurso não expõe exports. Os dois callbacks abaixo são registrados no servidor e chamados pelo próprio cliente — ambos validam origem e distância, mas estão disponíveis:

### `qbx_cityhall:server:requestId`

Cobra a licença e entrega o item. `item` é a chave em `licenses` (`id`, `driver`, `weapon`); `hall` é o índice da prefeitura em `cityhalls`.

```lua
lib.callback('qbx_cityhall:server:requestId', false, nil, item, hall)
```

### `qbx_cityhall:server:applyJob`

Aplica o emprego escolhido, se ele existir em `employment.jobs` e o jogador estiver a até 20 metros da prefeitura.

```lua
lib.callback('qbx_cityhall:server:applyJob', false, nil, job)
```

---

## Localização

Strings via `ox_lib` locale, em `locales/`:

`ar`, `cs`, `da`, `de`, `en`, `es`, `et`, `fr`, `nl`, `pt-br`, `tr`

Idioma ativo definido no `server.cfg`:

```
setr ox:locale "pt-br"
```

---

## Estrutura de arquivos

```
qbx_cityhall/
├── client/
│   └── main.lua          — spawn dos peds, blips, zona/target, menus de identidade e emprego
├── server/
│   └── main.lua          — callbacks requestId e applyJob, cobrança e validação de distância
├── config/
│   ├── client.lua        — useTarget, peds e zonas
│   └── shared.lua        — prefeituras, licenças e empregos
├── locales/
│   └── *.json            — traduções (11 idiomas)
└── fxmanifest.lua
```
