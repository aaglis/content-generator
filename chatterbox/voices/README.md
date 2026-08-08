# Vozes do Chatterbox (presets / referências)

O Chatterbox **clona** a voz a partir de um áudio de referência. **O sotaque vem da referência.**
=> Use referências em **pt-BR** ou a saída sai com sotaque estrangeiro.

Cada arquivo `.wav`/`.mp3` aqui vira uma voz selecionável (o nome do arquivo = id da voz).

## Vozes que já vêm prontas (pt-BR, geradas com Piper)

- `faber-br.wav`  — masculina grave (BR)
- `jeff-br.wav`   — masculina 2 (BR)
- `maria-br.wav`  — feminina (BR, feminizada do faber)

Regenerar: `cd viral-content-factory-backend && bash scripts/setup-chatterbox-voices.sh`

## Adicionar a sua própria voz (recomendado p/ qualidade máxima)

Dropa aqui um `.wav`/`.mp3` com ~10s de fala **limpa** (sem ruído/música), em **português**:

```
adam.wav      -> voz "adam"
narrador.wav  -> voz "narrador"
```

Aparece no seletor do frontend na hora (bind-mount, sem rebuild).
Para clonar uma pessoa específica, o mesmo arquivo de referência serve.

> As vozes em inglês de exemplo (Gabriel/Henry/Elena/Cora do repo do servidor) dão sotaque gringo
> no pt-BR — por isso aqui usamos referências brasileiras.
