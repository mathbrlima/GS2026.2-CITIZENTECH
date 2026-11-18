# Sentinela das Águas — Alerta Inteligente de Enchentes

**GS2026.2 – CitizenTech** · Computer Organization and Architecture · Prof. Dr. Marcus Grilo
**Plataforma:** Raspberry Pi Pico · MicroPython · Wokwi

## Integrantes

| Nome completo | RM |
|---|---|
| Matheus Bruno | RM572944 |
| Enzo Stahal    | RM569001 |

**Simulação (Wokwi):**  https://wokwi.com/projects/480807567396977665 

**Vídeo demonstrativo:**  https://youtu.be/T3CiwTr223o

---

## 1. Identificação do problema

Em cidades grandes como São Paulo, temporais de verão fazem córregos canalizados e piscinões subirem em poucos minutos. O asfalto e as calçadas impermeabilizam o solo, então a água da chuva vai quase toda de uma vez para os canais. O transbordo acontece rápido, muitas vezes antes de qualquer aviso chegar a quem está perto.

**Quem é afetado:**
- **Moradores de áreas ribeirinhas e de risco:** precisam de tempo para proteger a família e seus bens ou para sair de casa.
- **Motoristas, motociclistas e pedestres:** entram em vias que estão começando a alagar sem perceber o perigo, e veículos e pessoas acabam arrastados.
- **Defesa Civil e Prefeitura:** dependem de informação em tempo real, ponto a ponto, para decidir onde agir primeiro.

**Por que a tecnologia ajuda:** hoje boa parte dos alertas é regional ("risco de alagamento na Zona Leste") e chega por canais que nem todos acompanham. Um dispositivo IoT instalado **no próprio córrego** mede a situação local a cada meio segundo. Ele avisa quem está fisicamente ali, com luz, som e painel, bloqueia a via sozinho em caso extremo e envia os dados para uma central. Isso transforma minutos de vantagem em vidas e patrimônio preservados.

## 2. A solução

A **Sentinela das Águas** é uma estação de monitoramento instalada sobre o córrego. Ela:

1. **Mede o nível da água** com um sensor ultrassônico posicionado acima do canal.
2. **Mede a intensidade da chuva** com um pluviômetro, simulado por um potenciômetro.
3. **Calcula a velocidade de subida da água** (cm/min) a partir do histórico dos últimos 10 s.
4. **Classifica o risco em 4 níveis** e aciona as saídas correspondentes.
5. **Bloqueia a via automaticamente** com uma cancela (servo motor) na emergência.
6. **Permite que a Defesa Civil confirme o alerta** com um botão, silenciando a sirene sem liberar a via.
7. **Envia telemetria em JSON** pela serial, simulando a comunicação com uma central IoT da prefeitura.

### Diferencial: alerta preditivo
Um sistema simples só reage ao nível atual. A Sentinela também considera **as causas** da subida. Se a chuva estiver forte (≥ 50 mm/h) ou a água estiver subindo rápido (≥ 30 cm/min), o risco é **antecipado em um nível**, ou seja, o alerta sai **antes** de a água chegar ao limite. Com chuva extrema (≥ 80 mm/h), o sistema entra em ATENÇÃO mesmo com o córrego ainda baixo.

### Níveis de risco e respostas

| Risco | Condição | LEDs | Sirene | Cancela | Significado |
|---|---|---|---|---|---|
| 🟢 NORMAL | nível < 150 cm | verde | desligada | aberta | Situação segura |
| 🟡 ATENÇÃO | nível ≥ 150 cm ou chuva ≥ 80 mm/h | amarelo | desligada | aberta | Monitorar, preparar equipes |
| 🟠 ALERTA | nível ≥ 250 cm | amarelo + vermelho piscando | bipes intermitentes | aberta | Evacuar áreas baixas, evitar a via |
| 🔴 EMERGÊNCIA | nível ≥ 320 cm | vermelho fixo | sirene bitonal | **fechada** | Transbordo iminente, via bloqueada |

> Agravantes (a partir de ATENÇÃO): chuva ≥ 50 mm/h **ou** subida ≥ 30 cm/min → risco sobe um nível.

## 3. Componentes e pinagem

| Componente | Representa | Pinos na Pico |
|---|---|---|
| Raspberry Pi Pico | Unidade de processamento da estação | — |
| Sensor ultrassônico HC-SR04 | Medidor de nível do córrego (instalado a 400 cm do fundo) | TRIG → GP10 · ECHO → GP11 · VCC → VBUS |
| Potenciômetro | Pluviômetro (0 a 100 mm/h) | SIG → GP26 (ADC0) · VCC → 3V3 |
| Display LCD 16x2 I2C (end. 0x27) | Painel público de informação | SDA → GP4 · SCL → GP5 · VCC → VBUS |
| LED verde + resistor 220 Ω | Sinal "normal" | GP12 |
| LED amarelo + resistor 220 Ω | Sinal "atenção/alerta" | GP13 |
| LED vermelho + resistor 220 Ω | Sinal "alerta/emergência" | GP14 |
| Servo motor | Cancela de bloqueio da via | PWM → GP15 · V+ → VBUS |
| Buzzer | Sirene sonora | GP16 |
| Botão (azul) | Confirmação da Defesa Civil | GP2 (pull-up interno) e GND |

## 4. Funcionamento do código (`main.py`)

O programa segue o ciclo clássico da computação física: **entrada → processamento → saída → comunicação**. Ele roda em um **loop não bloqueante**: cada tarefa tem seu próprio intervalo controlado por `time.ticks_ms()`, então os LEDs piscam, a sirene alterna e o LCD atualiza ao mesmo tempo, sem um `sleep` longo travando o sistema.

| Tarefa | Intervalo | O que faz |
|---|---|---|
| Leitura dos sensores | 500 ms | Dispara o HC-SR04, converte o eco em distância (`duração / 58`), calcula `nível = 400 − distância`, lê a chuva no ADC |
| Cálculo da subida | 500 ms | Guarda (tempo, nível) dos últimos 10 s e calcula cm/min |
| Classificação | 500 ms | Função `classificar()` aplica os limiares e os agravantes preditivos |
| LEDs e sirene | contínuo | Pisca a 400 ms; sirene alterna 700/1200 Hz a cada 300 ms (PWM) |
| Painel LCD | 1 s | Linha 1: nível e risco. Linha 2: alterna chuva, subida, estado da via e motivo do alerta |
| Telemetria IoT | 2 s | Envia um JSON com todos os dados pela serial |

**Destaques técnicos:**
- **Driver próprio do LCD I2C:** o controlador HD44780 é comandado em modo 4 bits através do expansor PCF8574, sem biblioteca externa.
- **Interrupção (IRQ) no botão** com *debounce* por software (250 ms). O botão responde na hora, mesmo no meio do loop.
- **Servo por PWM a 50 Hz:** o ângulo é convertido em largura de pulso (500–2500 µs) e depois em `duty_u16`.
- **Tratamento de falha do sensor:** se o eco não volta (`time_pulse_us` < 0), a leitura é descartada e o último valor válido é mantido.
- **Log de eventos:** toda mudança de estado é registrada na serial com o motivo (nível, chuva forte, subida rápida ou chuva extrema).

Exemplo de telemetria:
```json
{"estacao": "corrego-01", "nivel_cm": 330, "chuva_mmh": 62, "subida_cm_min": 45.0,
 "risco": "EMERGENC", "motivo": "nivel", "via_bloqueada": true,
 "sirene_silenciada": false, "alertas_emitidos": 3}
```

## 5. Como testar na simulação

Clique em ▶ Play. Com a simulação rodando, clique no sensor HC-SR04 para ajustar a distância e gire o potenciômetro.

| Teste | Ação | Resultado esperado |
|---|---|---|
| 1 | Distância 400 cm, potenciômetro no mínimo | NORMAL, LED verde, via liberada |
| 2 | Distância ≈ 240 cm (nível 160) | ATENÇÃO, LED amarelo. Se a mudança for rápida, vai a ALERTA por "subida rápida" |
| 3 | Mantendo 240 cm, potenciômetro acima da metade | ALERTA por "chuva forte", bipes |
| 4 | Distância ≈ 140 cm (nível 260), potenciômetro no mínimo | ALERTA pelo nível |
| 5 | Distância ≈ 70 cm (nível 330) | EMERGÊNCIA: vermelho fixo, sirene, cancela fecha, LCD "Via: BLOQUEADA" |
| 6 | Apertar o botão "Defesa Civil" | Sirene silencia por 30 s (contagem no LCD), cancela continua fechada |
| 7 | Distância 400 cm, potenciômetro quase no máximo (≥ 80 mm/h) | ATENÇÃO por "chuva extrema" mesmo com córrego vazio |
| 8 | Distância 400 cm, potenciômetro no mínimo | Volta a NORMAL, cancela abre |

O monitor serial mostra as mudanças de estado e a telemetria JSON em tempo real.

## 6. Benefícios para a sociedade

- **Aviso antecipado e local:** quem está perto do córrego é alertado por luz, som e painel, sem depender de celular ou internet.
- **Prevenção de mortes no trânsito:** a cancela impede automaticamente a entrada em uma via prestes a alagar.
- **Decisão baseada em dados:** a Defesa Civil recebe nível, chuva e tendência de cada ponto e pode priorizar equipes.
- **Baixo custo e replicável:** usa componentes baratos e pode ser instalada em vários pontos críticos da cidade, formando uma rede de monitoramento.
- **Histórico para planejamento urbano:** os dados acumulados ajudam a identificar onde investir em drenagem e piscinões.

## 7. Limitações e próximos passos

- Na versão real, usar a **Pico W** para enviar a telemetria via Wi-Fi/MQTT a um painel da prefeitura e disparar alertas por SMS ou aplicativo.
- Trocar o potenciômetro por um **pluviômetro de báscula** real e proteger o sensor ultrassônico contra chuva (o ultrassom pode sofrer com espuma e detritos; em campo, um sensor de radar seria mais robusto).
- Alimentação por **painel solar com bateria**, para funcionar durante quedas de energia, comuns em temporais.
- Calibrar os limiares de acordo com a profundidade de cada córrego.
