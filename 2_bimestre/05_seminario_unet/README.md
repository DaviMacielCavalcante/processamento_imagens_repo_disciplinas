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

## Dependências

| Pacote | Para quê |
| --- | --- |
| `torch` e `torchvision` | Criar e treinar a U-Net. |
| `segmentation-models-pytorch` | U-Net pronta, com o encoder já pré-treinado no ImageNet. Com só 290 lâminas, isso faz bastante diferença. |
| `scikit-image` | Ler as imagens TIF e as máscaras PNG. Traz `numpy` e `pillow` junto. |
| `matplotlib` | Mostrar lado a lado a lâmina, a máscara real e a máscara prevista. |
| `ipykernel` | Rodar os notebooks no VS Code usando o ambiente do uv. |

Todas têm versão para o Python 3.14 do projeto. Para instalar, na raiz do repositório:

```bash
uv add torch torchvision segmentation-models-pytorch scikit-image matplotlib ipykernel
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
