# Seminário – Tema 5: Segmentação semântica com U-Net

Enunciado: [pdi_cc8_2bim_seminario_cnn.pdf](pdi_cc8_2bim_seminario_cnn.pdf)

- Entrega dos arquivos no Classroom: até 23h59 de 03/12/2026
- Apresentação: 04/12/2026, de 10 a 15 minutos

## Dataset

**Histopathology Non-Melanoma Skin Cancer Segmentation Dataset** (Universidade de Queensland)

- Página do dataset (UQ eSpace): https://espace.library.uq.edu.au/view/UQ:8be4bd0
- DOI: https://doi.org/10.14264/8be4bd0
- Artigo que descreve o dataset: Thomas, S. M.; Lefevre, J. G.; Baxter, G.; Hamilton, N. A. (2021). *Non-melanoma skin cancer segmentation for histopathology dataset*. Data in Brief. https://doi.org/10.1016/j.dib.2021.107587

### Como citar (slide de fontes)

> Thomas, S., & Hamilton, N. (2021). *Histopathology Non-Melanoma Skin Cancer Segmentation Dataset* [Data set]. The University of Queensland. https://doi.org/10.14264/8be4bd0

Montada a partir dos metadados oficiais do DOI. O registro lista Simon Thomas duas vezes como autor, o que parece erro de cadastro, por isso ele aparece só uma vez aqui.

### Conteúdo

- 290 lâminas de pele coradas com H&E: carcinoma basocelular (140), espinocelular (60) e intraepidérmico (90).
- Imagens em TIF e máscaras em PNG, marcadas à mão em 12 classes: glândulas, inflamação, folículos pilosos, hipoderme, derme reticular, derme papilar, epiderme, queratina, fundo, BCC, SCC e IEC. A legenda de cores está no repositório.
- As imagens vêm na resolução original e também reduzidas 2x, 5x e 10x. As versões reduzidas devem bastar para a demo.
- A divisão em treino, validação e teste já vem pronta: é a mesma usada no artigo original.

## Licença

**Permitted reuse with commercial use restriction**: https://guides.library.uq.edu.au/research-and-teaching-staff/data-deposit-checklist/license-reuse-noncommercial#s-lg-box-wrapper-25741028

- Uso permitido apenas para fins não comerciais.
- É obrigatório citar o dataset com a citação completa que aparece na página do UQ eSpace. Ela vai no slide de fontes.
- A licença não trata de redistribuição. Não subir o dataset no Classroom nem no GitHub: enviar só o código, a indicação de onde baixar e as imagens de resultado.
- Extrair os dados **fora do repositório** (por exemplo, em `~/dados/histologia_pele`). O download tem mais de 10 GB, e assim não há risco de ele ir parar no git.

## Dependências

| Pacote | Para quê |
| --- | --- |
| `torch` e `torchvision` | Criar e treinar a U-Net. |
| `segmentation-models-pytorch` | U-Net pronta, com o encoder já pré-treinado no ImageNet. Com só 290 lâminas, isso faz bastante diferença. |
| `scikit-image` | Ler as imagens TIF e as máscaras PNG. Traz `numpy` e `pillow` junto. |
| `matplotlib` | Mostrar lado a lado a lâmina, a máscara real e a máscara prevista. |
| `ipykernel` | Rodar os notebooks no VS Code usando o ambiente do uv. |
| `optuna` | Buscar automaticamente os melhores hiperparâmetros de treino. |
| `plotly` | Gerar os gráficos do Optuna (histórico das tentativas e importância de cada hiperparâmetro). |

Todas já estão no `pyproject.toml` da raiz do repositório. Para instalar, na raiz:

```bash
uv sync
```

O download tem alguns GB, porque o `torch` vem com as bibliotecas da GPU.

## Arquitetura

U-Net do `segmentation-models-pytorch` com encoder ResNet34 pré-treinado no ImageNet.

Caminho de um recorte pela rede. Cada caixa mostra o tamanho da imagem (altura × largura) e o número de canais (filtros):

```text
 DESCIDA (encoder: ResNet34)                     SUBIDA (decoder)

 entrada 512×512 · 3 (RGB)                       saída 512×512 · 12 classes
       │                                                 ▲
       ▼                                           512×512 · 16
 256×256 · 64   ─────────── ponte ───────────►   256×256 · 32
       ▼                                                 ▲
 128×128 · 64   ─────────── ponte ───────────►   128×128 · 64
       ▼                                                 ▲
  64×64 · 128   ─────────── ponte ───────────►    64×64 · 128
       ▼                                                 ▲
  32×32 · 256   ─────────── ponte ───────────►    32×32 · 256
       ▼                                                 ▲
  16×16 · 512   ───────────── fundo ─────────────────────┘
```

Na descida, a imagem encolhe pela metade a cada nível e o número de filtros cresce. Na subida acontece o contrário. As pontes (skip connections) levam os detalhes de cada nível direto para o lado oposto.

### Escolhas

- **Entrada em recortes de 512×512.** A lâmina inteira não cabe na memória da GPU. O lado do recorte precisa ser múltiplo de 32, porque a imagem é dividida por 2 cinco vezes.
- **Encoder pré-treinado.** A ResNet34 já aprendeu bordas e texturas em milhões de fotos. Com só 290 lâminas, começar disso é muito melhor do que treinar do zero. Ela é leve e cabe em uma GPU de 8 GB.
- **Saída com 12 canais**, um por classe de tecido. Cada pixel recebe a classe com a nota mais alta.
- **Função de perda: entropia cruzada + Dice.** Fundo e derme ocupam muito espaço, enquanto câncer e glândulas ocupam pouco. O Dice impede que a rede ignore as classes pequenas.
- **Métrica: IoU por classe**, para mostrar o acerto em cada tecido em vez de um número geral que esconde os erros nas classes pequenas.

### Diferenças em relação à U-Net original (Ronneberger et al., 2015)

- O original aprendia tudo do zero. Aqui o encoder já vem pré-treinado.
- No original, a saída era menor que a entrada (572 → 388 pixels). Aqui ela sai do mesmo tamanho.
- O original subia com convolução transposta. Aqui sobe com interpolação seguida de convolução.
- Aqui há 5 reduções de tamanho, contra 4 no original.

## Salvar e carregar a rede treinada

Para a demo não precisar treinar a rede de novo:

```python
# no fim do treino
model.save_pretrained("modelo_unet_pele")

# no notebook da demo
import segmentation_models_pytorch as smp
model = smp.from_pretrained("modelo_unet_pele")
model.eval()
```

A pasta salva guarda a arquitetura (`config.json`) e os pesos aprendidos (`model.safetensors`). O `from_pretrained` recria a rede já treinada.

- **Salvar o melhor modelo, não só o último:** durante o treino, salvar sempre que o IoU na validação melhorar.
- **Não versionar os pesos no git:** o arquivo fica perto de 100 MB, e o GitHub recusa arquivos acima disso. Enviar pelo Classroom ou pelo Drive.
- **A demo roda sem GPU:** o modelo carrega na CPU por padrão; só fica mais lento.
- **Em outro computador, rodar a demo uma vez antes, com internet:** ao recriar a rede, a biblioteca baixa os pesos do ImageNet do encoder na primeira vez.

## Otimização dos hiperparâmetros (Optuna)

Mesma abordagem do projeto [`autotuning_cnn`](https://github.com/DaviMacielCavalcante/autotuning_cnn): 20 tentativas com `TPESampler` + `MedianPruner` e objetivo `min(val_loss)`.

Diferença em relação ao projeto anterior, que usava Keras: em PyTorch, a poda das tentativas ruins vai direto no laço de treino, sem callback.

```python
for epoch in range(epochs):
    treinar_uma_epoca(...)
    val_loss = avaliar(...)
    trial.report(val_loss, epoch)
    if trial.should_prune():
        raise optuna.TrialPruned()
```

Entre uma tentativa e outra, liberar a GPU com `del model` e `torch.cuda.empty_cache()`, que faz o papel do `keras.backend.clear_session()`.

### Espaço de busca

| Hiperparâmetro | Valores |
| --- | --- |
| `learning_rate` | 1e-4 a 3e-3 (escala log) |
| `weight_decay` | 1e-6 a 1e-3 (escala log) |
| `batch_size` | 4, 8 ou 16, o que couber na GPU |
| `decoder_use_norm` | `batchnorm` ou `layernorm` |
| `decoder_attention_type` | nenhuma ou `scse` |

### Cuidados (lições do `autotuning_cnn`)

- **Embaralhar os recortes.** Os recortes saem de cada lâmina em sequência e ficam agrupados, o mesmo cenário do esquecimento catastrófico do projeto anterior (§30i do `APRENDIZADO.md`). Usar `DataLoader(shuffle=True)` e conferir.
- **BatchNorm com batch pequeno.** Com recortes de 512×512 numa GPU de 8 GB, o batch vai ser pequeno, e o BatchNorm fica instável (§25). Por isso o batch e o tipo de normalização estão no espaço de busca.
- **Não ajustar o peso entre entropia cruzada e Dice.** Se a composição da loss mudar entre tentativas, o `val_loss` deixa de ser comparável.
- **Comparação justa.** U-Net padrão contra a vencedora do Optuna, 3 execuções de cada, com o mesmo critério de parada (§31 e §30b).
- **Separar por lâmina.** Recortes da mesma lâmina nunca podem cair no treino e na validação ao mesmo tempo, senão o resultado sai inflado. Usar a divisão que vem com o dataset, que já é por lâmina.
- **Ordem.** Primeiro a U-Net padrão treinando e salva; depois o Optuna, com as imagens reduzidas e poucas épocas por tentativa.
