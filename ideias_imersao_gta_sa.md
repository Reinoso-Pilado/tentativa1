# Ideias de Mods — GTA San Andreas
## Mecânicas Sistêmicas Que Não Existem no Jogo Base Nem em Mods Conhecidos

Cada ideia aqui nasce de uma **lacuna real** da simulação do SA. O critério de inclusão foi direto: se o jogo base ou um CLEO simples já faz isso, não entra. O foco é em **mecânicas emergentes** — sistemas que geram situações que nem o jogo nem o jogador previram, sem precisar de uma missão para disparar.

---

## 1. `witness_system.cs` — Testemunhas Que Delatam

### A lacuna

O jogo base gera wanted level **instantaneamente** quando CJ comete um crime na frente de qualquer pedestre. Não existe uma fase intermediária de "alguém viu e vai contar". Isso significa que crimes em becos sem testemunhas têm o mesmo peso que crimes numa avenida movimentada — o motor ou vê ou não vê, sem nuance.

### O que este sistema cria

Uma janela de risco real entre o crime e a consequência. Quando CJ atira ou mata num ponto onde há civis, **um desses civis específicos** começa a correr em direção ao policial mais próximo para reportar. Se ele chegar, o wanted sobe 1 estrela. Se CJ eliminar ou assustar a testemunha antes disso, o wanted não sobe.

Isso cria decisões táticas que o jogo nunca oferece: dou um tiro furtivo e monitoro as testemunhas? Persigo o ciclista que viu? Corro antes que ele chegue no policial da esquina?

### Implementação

```
Thread: WITNESS  |  Loop: 600 ms

VARS:
  3@    = player actor handle (01F5)
  10@   = handle da testemunha ativa
  11@   = handle do policial mais próximo (destino da testemunha)
  12@   = estado: 0=inativo, 1=testemunha correndo, 2=reportando
  13@   = contador de timeout (max 20 ticks = 12s)
  14@   = blip da testemunha no radar
  6@7@8@ = coords do policial
```

**Trigger — detectar crime e encontrar testemunha:**
```
// Só age se CJ está atirando E sem wanted (crime ainda não foi detectado)
00D6: if
    02E0: is_char_shooting 3@
    AND 010F: is_wanted_level_greater $PLAYER_CHAR 0 == false
    AND 0039: 12@ == 0
jf @WITNESS_END

// Buscar civil no raio de 25m ao redor do CJ
// civilian=1, gang=0, criminal=0
073F: get_random_char_in_sphere (coords de 3@) 25.0 1 0 0 10@
if 056D: actor 10@ defined == false
jf @WITNESS_END  // ninguém viu — sem consequência

// Buscar o policial mais próximo para onde a testemunha vai correr
// Ped model 280 (LAPD) — buscar em raio maior (150m)
073F: get_random_char_in_sphere (coords de 3@) 150.0 0 0 0 11@
if 056D: actor 11@ defined == false
jf @WITNESS_FLEE_ONLY  // sem policial por perto — testemunha apenas foge

// Testemunha corre para o policial
00A0: 11@ 6@ 7@ 8@
0603: task_go_to_coord_any_means 10@ 6@ 7@ 8@ 2 -1   // MoveState 2 = sprint
0187: add_blip_for_char 10@ 14@   // blip amarelo no radar
0006: 12@ = 1
0006: 13@ = 0
jump @WITNESS_END

:WITNESS_FLEE_ONLY
// Sem policial — testemunha foge mas não delata
05DA: task_flee_point 10@ (coords de 3@) 50.0 8000
jump @WITNESS_END
```

**Loop de monitoramento — chegou ao policial?**
```
// Estado 1: testemunha a caminho
if 0039: 12@ == 1
then
    000A: 13@ += 1
    if 0019: 13@ > 20  // timeout 12s
    then jump @WITNESS_ABORT

    // Testemunha está perto do policial (raio 3m)?
    if 00F3: actor 10@ near_actor 11@ radius 3.0 3.0
    then
        // Testemunha reportou — aumentar wanted
        010E: alter_wanted_level_no_drop $PLAYER_CHAR 1
        0ACD: show_text_highpriority "Testemunha delatou para a policia!" 2500
        0164: remove_blip 14@
        jump @WITNESS_ABORT

    // Testemunha morreu? (CJ silenciou ela)
    if 056D: actor 10@ defined == false
    then
        0ACD: show_text_highpriority "Testemunha eliminada." 2000
        jump @WITNESS_ABORT

:WITNESS_ABORT
0006: 12@ = 0
0164: remove_blip 14@
```

### Cuidados de engine
- `073F` retorna peds já existentes no mundo — não usar `01C2` nesses handles, o motor ainda gerencia esses peds
- O blip (`0187`) deve ser removido com `0164` em **todo** caminho de saída ou o blip vaza no radar indefinidamente
- `056D` antes de qualquer opcode que use `10@` ou `11@` — peds podem morrer por outras causas entre ticks

---

## 2. `drug_run.cs` — Rota de Tráfico Interceptável

### A lacuna

Traficantes no SA ficam estáticos em esquinas. Nenhum veículo no mundo aberto carrega "drogas" de um ponto a outro de forma visível e interceptável pelo jogador. O tráfico existe no jogo como textura de fundo, nunca como evento observável com consequências.

### O que este sistema cria

A cada 6 horas de jogo, um carro específico (baseado no modelo de gangue da zona) aparece num ponto fixo e dirige até outro ponto com uma "carga". Um blip discursivo aparece no radar a curta distância. CJ pode: ignorar, interceptar destruindo o carro, ou interceptar e roubar o carro. Se CJ ataca o entregador, o motorista foge atirando e pode ligar de volta para backup.

Nenhuma missão é ativada. É um evento de mundo que existe independente do jogador.

### Implementação

```
Thread: DRUGRUN  |  Loop: 5000 ms

VARS:
  10@  = handle do carro da entrega
  11@  = handle do motorista
  12@  = handle do blip do carro
  13@  = estado: 0=aguardando, 1=em rota, 2=chegou/destruído
  14@  = timer de espera entre corridas (em ticks de 5s)
  15@  = horas do jogo (para trigger)
  16@  = minutos (descartado)
  0@1@2@ = coords de spawn
  6@7@8@ = coords de destino
```

**Pontos fixos de spawn e destino** (hardcoded como zonas conhecidas):
```
// Spawn: Bairro de Idlewood (Ballas), rua específica
// Destino: Glen Park, ponto de entrega
// Modelo de carro: Greenwood (520) — carro típico da zona sul
// Modelo do motorista: bmycr (ID 10) — criminoso genérico não marcado

// Coordenadas hardcoded (ajustar por zona):
// spawn:   0@ = 2034.5  1@ = -1416.3  2@ = 17.2
// destino: 6@ = 2118.7  7@ = -1684.2  8@ = 13.6
```

**Spawn e rota:**
```
// Timer: só spawnar às horas certas (00BF)
00BF: get_time_of_day 15@ 16@
// Ativa às 2h, 8h, 14h, 20h
// Usar: 0039 (== 2), 0039 (== 8), etc. com OR conditions

if 0039: 13@ == 0
AND (hora correta)
then
    // Spawnar carro
    0247: request_model 520
    // ... wait/load
    00A5: create_car 520 at 0@ 1@ 2@ 10@
    0249: release_model 520

    // Motorista como driver
    0247: request_model 10
    // ... wait/load
    009A: create_actor pedtype 4 model 10 at 0@ 1@ 2@ 11@
    0249: release_model 10
    05C2: actor 11@ enter_car 10@ as_driver -1 ms

    // Blip curto alcance — só aparece quando CJ está perto
    04CE: add_short_range_sprite_blip_for_coord 0@ 1@ 2@ 41 12@
    // sprite 41 = caixote/carga (ícone discreto no mapa)

    0001: wait 2000 ms  // aguarda motorista entrar

    // Configurar rota
    00A7: car 10@ drive_to 6@ 7@ 8@
    00AD: set_car 10@ max_speed_to 35.0
    00AE: set_car 10@ driving_style_to 4   // FOLLOWTRAFFIC — anda normalmente
    0006: 13@ = 1
```

**Monitoramento em rota — motorista atacado:**
```
if 0039: 13@ == 1
then
    // Carro destruído?
    if 056E: car 10@ defined == false
    then jump @DRUGRUN_CLEANUP

    // Motorista foi atacado? (verificar se está morto)
    if 056D: actor 11@ defined == false
    then jump @DRUGRUN_CLEANUP

    // Motorista chegou ao destino?
    if 00EC: actor 11@ near_point 6@ 7@ radius 8.0 8.0
    then jump @DRUGRUN_CLEANUP
```

**Reação a ataque:** se CJ atirar no carro (`02E0: is_char_shooting` + `056E: car 10@`), o motorista entra em modo de fuga:
```
0751: task_flee_char_any_means 11@ 3@ 80.0 15000 1 3000 1000 25.0
// flee from CJ, shoot=1, shootTime=3s, cooldown=1s, stealCar dist=25m
```

**Cleanup:**
```
:DRUGRUN_CLEANUP
if 056D: actor 11@ defined
then 01C2: mark_actor_as_no_longer_needed 11@
if 056E: car 10@ defined
then 01C3: mark_car_as_no_longer_needed 10@
0164: remove_blip 12@
0006: 13@ = 0
0006: 14@ = 0  // reset timer para próxima corrida
```

### Por que é interessante
O jogador observa o mundo fazendo algo independente. Ele escolhe o custo-benefício: atacar o entregador gera inimigos e possibly wanted; ignorar deixa a gangue operar. Não há marcador de missão. É um evento que acontece com ou sem o jogador.

---

## 3. `zone_heat.cs` — Calor de Zona (Reputação por Território)

### A lacuna

No SA, não importa quantas vezes CJ invade Idlewood e mata Ballas — a resposta deles é sempre a mesma no dia seguinte. O jogo não tem memória de comportamento violento por zona. O wanted level é global e temporário; não há reputação local persistente.

### O que este sistema cria

Cada zona do mapa tem um **"calor" interno** (float 0.0–10.0) que sobe quando CJ causa violência nela e decai lentamente com o tempo. Zonas com calor alto respondem com spawns mais agressivos da gangue local. Zonas com calor máximo ficam em estado de "guerra ativa" por algumas horas de jogo.

A persistência é feita via arquivo `.dat` lido na inicialização — o calor sobrevive a reloads de save.

### Implementação

```
Thread: ZONEHEAT  |  Loop: 2000 ms

VARS:
  0@1@2@ = coords do jogador
  10@    = nome da zona atual (GXT key)
  11@    = calor atual da zona (float)
  12@    = hora atual
  13@    = última hora de disparo registrada
  14@    = handle do arquivo de dados
  15@    = flag "jogador atirou neste tick"
```

**Estrutura do arquivo `CLEO\zone_heat.dat`:**
```
// Formato simples: uma linha por entrada
// zona_key:calor
// Ex:
// IDLEWOOD:4.2
// GANTON:1.0
// BALLAS1:7.8
```

**Leitura na inicialização:**
```
:HEAT_INIT
// Verificar se arquivo existe
if 0AAB: does_file_exist "CLEO\zone_heat.dat"
then
    0A9A: open_file "CLEO\zone_heat.dat" mode 0 14@  // mode 0 = leitura
    // Ler cada par zona:calor
    // 0ADA: scan_file para parsear "GANTON:4.2"
    // Armazenar em shared vars (0AB3) indexados por hash do nome da zona
    0A9B: close_file 14@
```

**Loop principal:**
```
00A0: 3@ 0@ 1@ 2@
0843: get_name_of_zone 0@ 1@ 2@ 10@  // 10@ = GXT key da zona atual

// Ler calor atual via shared var indexada pelo hash da zona
0AB4: get_cleo_shared_var 0 11@  // var 0 = calor da zona atual

// CJ atirou neste tick?
if 02E0: is_char_shooting 3@
then
    // Aumentar calor em 0.5 por tick com disparo
    000B: 11@ += 0.5
    if 0020: 11@ > 10.0  // cap em 10.0
    then 0007: 11@ = 10.0
    0AB3: set_cleo_shared_var 0 11@   // salvar de volta

// Decaimento: a cada 2h de jogo, reduzir 0.3
00BF: get_time_of_day 12@ 13@
// Lógica de decaimento por hora...
000D: 11@ -= 0.3
if 0023: 0.0 > 11@
then 0007: 11@ = 0.0

// Resposta ao calor — mais peds agressivos em zonas quentes
// Calor > 7: spawnar extra gang member agressivo na zona
if 0020: 11@ > 7.0
AND 0039: 14@ == 0  // 14@ = guard flag para não spawnar em duplicata
then
    jump @HEAT_SPAWN_AGGRESSOR
```

**Spawn de agressor baseado em calor:**
```
:HEAT_SPAWN_AGGRESSOR
// Buscar gang member existente da zona (não criar novo — usar o que já está lá)
073F: get_random_char_in_sphere 0@ 1@ 2@ 40.0 0 1 0 20@  // gang=1
if 056D: actor 20@ defined
then
    // Dar ao gang member existente a tarefa de perseguir CJ
    05E2: task_kill_char_on_foot 20@ 3@
    // Não criar novo ped — usar o que o motor já spawnaria
```

**Salvamento periódico:**
```
// A cada 5 minutos de jogo, gravar estado em arquivo
if (timer condition)
then
    0A9A: open_file "CLEO\zone_heat.dat" mode 1 14@  // mode 1 = escrita
    0AD9: write_formatted_string_to_file 14@ "%s:%.1f\n" 10@ 11@
    0A9B: close_file 14@
```

### Por que é interessante
Cria uma **memória territorial** que o jogo nunca teve. Uma zona que CJ invadiu três vezes consecutivas fica legitimamente mais perigosa. O jogador percebe que suas ações têm peso acumulado, não apenas o wanted level temporário. E esse peso persiste entre sessões.

---

## 4. `ghost_informant.cs` — Informante Fantasma

### A lacuna

No SA, a polícia reage ao que vê diretamente. Nunca há um NPC que pareça civil mas que esteja reportando CJ à distância. Não existe mecânica de "estar sendo observado sem saber".

### O que este sistema cria

A cada 4 horas de jogo, um ped civil específico dentro do raio do jogador é silenciosamente marcado como "informante". Ele não ataca. Ele segue CJ discretamente (usando `0850` a uma distância maior que o normal). Se CJ cometer qualquer crime enquanto o informante estiver no raio de 40m, um wanted level com 1 minuto de delay é ativado — como se a polícia tivesse recebido um relato tardio.

O único sinal visual que o informante existe é que ele **nunca foge** de tiroteios quando outros peds fogem. O jogador que prestar atenção vai perceber.

### Implementação

```
Thread: GHOST  |  Loop: 500 ms

VARS:
  3@  = player actor handle
  10@ = handle do informante ativo
  11@ = estado: 0=sem informante, 1=ativo, 2=reportando
  12@ = contador de delay de wanted (ticks após crime)
  13@ = flag "viu crime"
  14@ = horas do jogo (para trigger)
  15@ = minutos (descartado)
```

**Trigger (a cada 4h de jogo):**
```
00BF: get_time_of_day 14@ 15@
// Usar múltiplos de 4h: 0, 4, 8, 12, 16, 20
if (0039: 14@ == 0 OR 0039: 14@ == 4 OR ...)
AND 0039: 11@ == 0
then
    // Buscar civil próximo (não gangue, não criminoso)
    073F: get_random_char_in_sphere (coords CJ) 30.0 1 0 0 10@
    if 056D: actor 10@ defined
    then
        // Fazer o informante seguir CJ silenciosamente
        // Distância maior que o normal — não óbvio demais
        0850: task_follow_footsteps 10@ 3@
        // Calar o informante (sem falas de medo em tiroteios)
        0489: shut_char_up 10@ 1
        0006: 11@ = 1
        0006: 13@ = 0
```

**Monitoramento:**
```
if 0039: 11@ == 1
then
    if 056D: actor 10@ defined == false
    then jump @GHOST_CLEANUP  // informante morreu

    // Crime cometido dentro do raio?
    if 02E0: is_char_shooting 3@
    AND 00F3: actor 10@ near_actor 3@ 40.0 40.0
    then
        0006: 13@ = 1  // marcou o crime

    // Delay de 24 ticks (12s) após marcar o crime
    if 0039: 13@ == 1
    then
        000A: 12@ += 1
        if 0019: 12@ > 24
        then
            // Wanted com delay
            010E: alter_wanted_level_no_drop $PLAYER_CHAR 1
            0ACD: show_text_highpriority "Chamada de emergencia recebida." 3000
            jump @GHOST_CLEANUP

:GHOST_CLEANUP
if 056D: actor 10@ defined
then 0489: shut_char_up 10@ 0   // restaurar falas
0006: 11@ = 0
0006: 12@ = 0
0006: 13@ = 0
```

### Por que é interessante
É uma mecânica de **paranoia passiva**. O jogador que descobriu o sistema vai começar a escanear os peds ao redor procurando o "que não foge". Cria tensão sem nenhum inimigo visível. E em 90% das sessões o jogador nem vai perceber que o sistema existe — até que um wanted apareça sem causa aparente e ele comece a questionar.

---

## 5. `territory_decay.cs` — Dominância por Abandono

### A lacuna

No SA, conquistas territoriais são permanentes após a missão do story mode. No mundo aberto, a força das gangues em cada zona (`076D`) não muda dinamicamente com base no comportamento do jogador. Uma zona conquistada permanece conquistada para sempre independente de CJ nunca mais aparecer lá.

### O que este sistema cria

Cada zona tem um timer invisível. Se CJ não aparecer em um território conquistado por X horas de jogo, a força da gangue rival começa a subir gradualmente (`076C: set_zone_gang_strength`). Se chegar ao máximo, a zona é marcada como "recontestada" com um blip discreto no radar. Isso força CJ a ter uma rotina de patrulha territorial ou aceitar que o mapa encolhe.

### Implementação

```
Thread: TDECAY  |  Loop: 10000 ms (10s — mudança lenta)

VARS:
  0@1@2@ = coords do jogador
  10@    = zona atual (GXT key)
  11@    = força atual da gangue inimiga na zona (int)
  12@    = handle do blip de alerta
  13@    = flag de blip ativo
  14@15@ = hora atual / minutos (descartado)
  16@    = slot de shared var para o timer da zona
```

**Lógica de decaimento:**
```
00A0: 3@ 0@ 1@ 2@
0843: get_name_of_zone 0@ 1@ 2@ 10@

// CJ está nesta zona — resetar timer dela
0AB3: set_cleo_shared_var 16@ 0   // 16@ = índice do timer para essa zona

// Para zonas onde CJ NÃO está:
// Incrementar timer de ausência (via shared vars por zona)
// A cada 10s sem CJ, incrementar em 1
// Após 360 ticks (1h de jogo real), incrementar força inimiga em 1

// Verificar força atual dos Ballas na zona IDLEWOOD:
076D: get_zone_gang_strength "IDLEWOOD" 1 11@   // gang 1 = Ballas

// Força chegou ao máximo (10)?
if 0019: 11@ > 8
AND 0039: 13@ == 0
then
    // Adicionar blip de alerta no mapa
    018A: add_blip_for_coord (coords fixas de IDLEWOOD) 12@
    018B: change_blip_display 12@ 2   // display 2 = blip piscando
    0ACD: show_text_highpriority "Idlewood sendo recontestada pelos Ballas!" 4000
    0006: 13@ = 1

// CJ foi lá e limpou — reduzir força
if 0154: actor 3@ in_zone "IDLEWOOD"
AND 0019: 11@ > 0
AND 02E0: is_char_shooting 3@
then
    000A: 11@ -= 1  // Na prática usar 076C para persistir a mudança
    076C: set_zone_gang_strength "IDLEWOOD" 1 11@
    // Remover blip se força caiu
    if 001B: 6 > 11@
    then
        0164: remove_blip 12@
        0006: 13@ = 0
```

**Persistência entre sessões:**
O estado atual de força das zonas pode ser lido via `076D` e gravado num arquivo `CLEO\territory.dat` usando `0A9A`+`0AD9`. Na próxima sessão, o script restaura esses valores com `076C`.

### Por que é interessante
O mapa vira um recurso que **se deteriora por negligência**. O jogador que passa horas explorando Las Venturas vai voltar pra Los Santos e encontrar o mapa mudado. É a primeira vez que o mundo aberto do SA tem consequência passiva por ausência — sem missão, sem cutscene.

---

## 6. `ambient_gang_war.cs` — Guerra de Gangues Autônoma

### A lacuna

O sistema de guerras territoriais do SA (`0879`, `0A03`) só existe dentro do contexto de missões do story mode. No mundo aberto livre, GSF e Ballas nunca organizam um confronto planejado entre si sem a participação direta do jogador. Eles se atacam por proximidade aleatória, mas nunca há um "ataque coordenado a um ponto" entre facções sem o player estar envolvido.

### O que este sistema cria

A cada 90 minutos de jogo, o script verifica uma zona de fronteira entre territórios (ex: ponto de contato entre Ganton e Idlewood). Spawna 2–3 Ballas e 2–3 GSF no mesmo local e os direciona para o ponto de conflito. Eles lutam entre si até um lado vencer. O resultado (qual lado tem sobreviventes) determina uma pequena mudança na força territorial via `076C`. O jogador recebe uma notificação discreta e pode ir participar, ajudar um lado ou simplesmente ignorar.

### Implementação

```
Thread: AMBWAR  |  Loop: 5000 ms

VARS:
  10@11@12@ = handles dos 3 GSF spawnados
  13@14@15@ = handles dos 3 Ballas spawnados
  16@       = estado: 0=em paz, 1=guerra em andamento
  17@       = contador de timeout da guerra (máx 60 ticks = 5min)
  18@       = vencedor: 0=empate, 1=GSF, 2=Ballas
  19@       = blip do ponto de conflito
  0@1@2@    = coords do ponto de conflito (hardcoded por zona)
```

**Ponto de conflito hardcoded:**
```
// Fronteira Ganton/Idlewood — rua entre os dois bairros
// 0@ = 2097.3   1@ = -1532.1   2@ = 13.5
```

**Spawn dos dois lados:**
```
// Lado 1: GSF (fam1 = 105)
0247: request_model 105
// ... wait/load
009A: create_actor pedtype 8 model 105 at (offset +5m de 0@1@2@) 10@
0249: release_model 105

// Lado 2: Balla (ballas1 = 102)
0247: request_model 102
// ... wait/load
009A: create_actor pedtype 7 model 102 at (offset -5m de 0@1@2@) 13@
0249: release_model 102

// (Repetir para os 2 restantes de cada lado)

// Atribuir targets: cada GSF ataca um Balla
05E2: task_kill_char_on_foot 10@ 13@   // GSF 1 → Balla 1
05E2: task_kill_char_on_foot 11@ 14@   // GSF 2 → Balla 2
05E2: task_kill_char_on_foot 12@ 15@   // GSF 3 → Balla 3
// E vice-versa
05E2: task_kill_char_on_foot 13@ 10@
05E2: task_kill_char_on_foot 14@ 11@
05E2: task_kill_char_on_foot 15@ 12@

// Blip no ponto para o jogador
018A: add_blip_for_coord 0@ 1@ 2@ 19@
0ACD: show_text_highpriority "Conflito territorial em andamento perto de Ganton..." 4000
0006: 16@ = 1
0006: 17@ = 0
```

**Contagem de sobreviventes e resultado:**
```
:AMBWAR_CHECK
if 0039: 16@ == 1
then
    000A: 17@ += 1
    if 0019: 17@ > 60  // timeout 5 min
    then jump @AMBWAR_CONCLUDE

    // Contar sobreviventes de cada lado
    0006: 20@ = 0   // gsf_alive
    0006: 21@ = 0   // ballas_alive
    if 056D: actor 10@ defined then 000A: 20@ += 1
    if 056D: actor 11@ defined then 000A: 20@ += 1
    if 056D: actor 12@ defined then 000A: 20@ += 1
    if 056D: actor 13@ defined then 000A: 21@ += 1
    if 056D: actor 14@ defined then 000A: 21@ += 1
    if 056D: actor 15@ defined then 000A: 21@ += 1

    // Ambos os lados zerados = empate, encerrar
    if 0039: 20@ == 0 AND 0039: 21@ == 0
    then jump @AMBWAR_CONCLUDE

:AMBWAR_CONCLUDE
// Determinar vencedor e ajustar zona
if 0019: 20@ > 21@     // GSF venceu
then
    076D: get_zone_gang_strength "IDLEWOOD" 1 22@
    000E: 22@ -= 1       // Ballas perdem 1 ponto de força em Idlewood
    076C: set_zone_gang_strength "IDLEWOOD" 1 22@
    0ACD: show_text_highpriority "Grove Street venceu o confronto!" 3000

if 0019: 21@ > 20@     // Ballas venceram
then
    076D: get_zone_gang_strength "GANTON" 1 22@
    000A: 22@ += 1       // Ballas ganham força em Ganton
    076C: set_zone_gang_strength "GANTON" 1 22@
    0ACD: show_text_highpriority "Ballas avancam para Ganton!" 3000

// Cleanup
if 056D: actor 10@ defined then 01C2: mark_actor_as_no_longer_needed 10@
// ... (repetir para 11@, 12@, 13@, 14@, 15@)
0164: remove_blip 19@
0006: 16@ = 0
```

### Por que é interessante
O mundo passa a ter **causas e efeitos que não dependem do jogador**. CJ vai sair de casa e o mapa vai estar diferente porque uma guerra aconteceu enquanto ele não estava por perto. O jogador que quiser manter o território vai ter que monitorar os conflitos. O que quiser apenas explorar vai perceber que o mapa muda sem ele.

---

## 7. Arquitetura Compartilhada — Regras Para Todos os Scripts

Estas regras valem para qualquer script novo, baseadas no padrão estabelecido em `grove_recruit_follow.cs`:

### Threads e Timing
| Script | Thread Name | Loop |
|--------|-------------|------|
| `grove_recruit_follow.cs` | `GRECRUIT` | 300 ms |
| `witness_system.cs` | `WITNESS` | 600 ms |
| `drug_run.cs` | `DRUGRUN` | 5000 ms |
| `zone_heat.cs` | `ZONEHEAT` | 2000 ms |
| `ghost_informant.cs` | `GHOST` | 500 ms |
| `territory_decay.cs` | `TDECAY` | 10000 ms |
| `ambient_gang_war.cs` | `AMBWAR` | 5000 ms |

### Regras de Memória (Invioláveis)
- `01C2`/`01C3` em **todo** cleanup de peds/carros criados por `009A`/`00A5`
- `056D`/`056E` antes de **qualquer** opcode que use um handle
- `0164: remove_blip` em **todo** caminho de saída de código que criou um blip
- Peds encontrados por `073F` ou `0AE1` **não recebem** `01C2` — o motor já os gerencia
- `0249: release_model` imediatamente após `009A`/`00A5`

### Compartilhamento de Dados Entre Scripts
Para que `zone_heat.cs` e `territory_decay.cs` não conflitem ao chamar `076C` simultaneamente, usar **CLEO shared vars** (`0AB3`/`0AB4`) como mutex:
```
// Antes de chamar 076C:
0AB4: get_cleo_shared_var 99 25@   // var 99 = lock de zona
if 0039: 25@ == 0
then
    0AB3: set_cleo_shared_var 99 1  // adquirir lock
    076C: set_zone_gang_strength ...
    0AB3: set_cleo_shared_var 99 0  // liberar lock
```

### Opcodes Chave Por Categoria (verificados em `opcodes.txt`)
| Categoria | Opcode | Linha em opcodes.txt |
|-----------|--------|---------------------|
| Hora do jogo | `00BF: get_time_of_day` | 161 |
| Ped em zona | `0154: is_char_in_zone` | 217 |
| Nome da zona | `0843: get_name_of_zone` | 1201 |
| Força da gangue | `076D: get_zone_gang_strength` | 1054 |
| Alterar força | `076C: set_zone_gang_strength` | 1053 |
| Ped aleatório em esfera | `073F: get_random_char_in_sphere` | 1022 |
| Ped aleatório recursivo | `0AE1: get_random_char_in_sphere_no_save_recursive` | 1702 |
| Ped atirando | `02E0: is_char_shooting` | 381 |
| Wanted level atual | `01C0: store_wanted_level` | 282 |
| Wanted sem drop | `010E: alter_wanted_level_no_drop` | 200 |
| Fuga inteligente | `0751: task_flee_char_any_means` | 1037 |
| Matar (agachar) | `0634: task_kill_char_on_foot_while_ducking` | 837 |
| Perseguir atirando | `0637: task_go_to_coord_while_shooting` | 839 |
| Calar ped | `0489: shut_char_up` | 590 |
| Offset de veículo | `0407: get_offset_from_car_in_world_coords` | 528 |
| Arquivo: abrir | `0A9A: open_file` | 1644 |
| Arquivo: escrever | `0AD9: write_formatted_string_to_file` | 1694 |
| Arquivo: ler | `0ADA: scan_file` | 1695 |
| Arquivo: fechar | `0A9B: close_file` | 1645 |
| Shared var: set | `0AB3: set_cleo_shared_var` | 1665 |
| Shared var: get | `0AB4: get_cleo_shared_var` | 1666 |
| Blip por coord | `018A: add_blip_for_coord` | 251 |
| Blip por ped | `0187: add_blip_for_char` | 249 |
| Blip curto alcance | `04CE: add_short_range_sprite_blip_for_coord` | 628 |
| Remover blip | `0164: remove_blip` | 226 |
