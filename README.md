# Sentinela das Águas — Alerta inteligente de enchentes

GS2026.2 CitizenTech · Raspberry Pi Pico + MicroPython · Wokwi

## Problema
Em São Paulo, córregos e piscinões transbordam em minutos durante temporais. Motoristas entram em vias alagadas, pedestres são arrastados e moradores de áreas de risco recebem o alerta tarde demais. Os mais afetados são as comunidades ribeirinhas, quem passa a pé ou de carro pelas vias próximas e as equipes da Defesa Civil, que dependem de informação em tempo real.

## Solução
Uma estação instalada no córrego mede o nível da água e a chuva, calcula a velocidade de subida e classifica o risco em 4 níveis. A camada preditiva (chuva forte ou água subindo rápido) antecipa o alerta **antes** do transbordo. Em emergência, uma cancela bloqueia automaticamente a via. A telemetria sai em JSON pela serial, simulando o envio para uma central IoT.

| Risco | Condição (nível) | LEDs | Sirene | Cancela |
|---|---|---|---|---|
| NORMAL | < 150 cm | verde | desligada | aberta |
| ATENÇÃO | ≥ 150 cm (ou chuva ≥ 80 mm/h) | amarelo | desligada | aberta |
| ALERTA | ≥ 250 cm | amarelo + vermelho piscando | bipes | aberta |
| EMERGÊNCIA | ≥ 320 cm | vermelho fixo | sirene bitonal | **fechada** |

Chuva ≥ 50 mm/h ou subida ≥ 30 cm/min sobe o risco em um nível (a partir de ATENÇÃO).
O botão azul (Defesa Civil) silencia a sirene por 30 s sem liberar a via.

## Componentes e pinagem
| Componente | Função | Pino Pico |
|---|---|---|
| HC-SR04 | nível da água (sensor a 400 cm do fundo) | TRIG GP10 · ECHO GP11 · VBUS |
| Potenciômetro | pluviômetro (0–100 mm/h) | GP26 (ADC0) · 3V3 |
| LCD 16x2 I2C (0x27) | painel público | SDA GP4 · SCL GP5 · VBUS |
| LEDs verde/amarelo/vermelho + 220 Ω | semáforo de risco | GP12 · GP13 · GP14 |
| Buzzer | sirene | GP16 |
| Servo | cancela da via | GP15 · VBUS |
| Botão | Defesa Civil silencia a sirene | GP2 (pull-up interno) |

## Como montar no Wokwi
1. wokwi.com → New Project → Raspberry Pi Pico → MicroPython.
2. Substitua o `main.py` e o `diagram.json` (aba diagram.json) pelos deste projeto.
3. Clique em Play. Clique no HC-SR04 para mudar a distância e gire o potenciômetro.

## Roteiro sugerido para o vídeo (3–4 min)
1. **Problema (30 s):** enchentes em SP, quem é afetado.
2. **Solução e circuito (45 s):** mostrar cada componente e o que ele representa.
3. **Demonstração (2 min):**
   - Distância 400 cm → NORMAL (verde, via liberada).
   - Distância ~240 cm (nível 160) → ATENÇÃO; mudar rápido mostra o agravamento por "subida rápida".
   - Girar o potenciômetro acima de 50 mm/h → risco sobe por "chuva forte".
   - Distância ~70 cm (nível 330) → EMERGÊNCIA: sirene, vermelho fixo, cancela baixa.
   - Apertar o botão → sirene muda, cancela continua fechada.
   - Voltar a distância para 400 cm → NORMAL, cancela sobe.
   - Mostrar o JSON no monitor serial (telemetria IoT).
4. **Código (30 s):** função `classificar()` e o loop não bloqueante (entradas → processamento → saídas → telemetria).
5. **Benefício (15 s):** aviso antecipado, vias bloqueadas antes do alagamento, dados para a Defesa Civil.
