# Desenvolvimento do notebook `unet.ipynb`

Registro das verificações que deram certo durante o desenvolvimento do notebook. As decisões de projeto e as informações do dataset estão no [README.md](README.md).

## 27/11/2026

### 1. GPU

- **Teste:** `torch.cuda.is_available()` e `torch.cuda.get_device_name(0)`.
- **Resultado:** `True`, com a placa `NVIDIA GeForce RTX 4060 Ti`.
- O `device` usa a GPU quando ela existe e cai para a CPU quando não existe, para o notebook rodar em qualquer computador da equipe.

### 2. Listas de divisão

- **Teste:** `read_list` nos três arquivos de divisão.
- **Resultado:** 231 lâminas de treino, 28 de validação e 31 de teste, com os nomes limpos (por exemplo, `['SCC_33', 'BCC_101', 'BCC_122', 'IEC_15']`).
- Os três arquivos não têm linhas vazias nem espaços sobrando.

### 3. Leitura de uma máscara

- **Teste:** `io.imread(Path(MASKS_PATH) / f"{train_list[0]}.png")`, com a lâmina `SCC_33`.
- **Resultado:** formato `(763, 599, 3)`, tipo `uint8`.

### 4. Conversão de cor para classe

- **Teste:** `masks_for_classes(test_mask)` na máscara da `SCC_33`.
- **Resultado:** formato `(763, 599)`, com as classes `[0, 1, 3, 8, 10]`: fundo, queratina, derme papilar, inflamação e carcinoma espinocelular.
- Faz sentido: a `SCC_33` é uma lâmina de carcinoma espinocelular e não aparece nenhum dos outros dois cânceres (classes 9 e 11).

### 5. Conferência da conversão

A conferência faz o caminho de volta: `PALETA[classes]` reconstrói a máscara colorida a partir das classes, e o resultado é comparado com a máscara original.

- **Visual:** máscara original e reconstruída desenhadas lado a lado. As duas ficaram idênticas.
- **Numérica:** `(reconstruida == test_mask).all(axis=-1).mean()` deu **0,99997**. Os 0,003% restantes são os pixels de borda cuja cor não estava na paleta e foi trocada pela mais próxima. Este número foi calculado fora do notebook, com o mesmo código.

### 6. Carga de uma amostra de lâminas

- **Teste:** `load_images_list(train_list[:3])`, que lê as imagens e as máscaras, converte as máscaras e devolve uma lista de pares `(imagem, classes)`, com barra de progresso do `tqdm`.
- **Resultado:**
  - 3 pares;
  - no primeiro par, imagem `(763, 599, 3)` e classes `(763, 599)`;
  - classes `[0, 1, 3, 8, 10]`, as mesmas do teste da `SCC_33`. Isso confirma que a ordem dos pares segue a ordem dos nomes.

### 7. Conferência visual do par imagem–máscara

- **Teste:** a lâmina, a máscara reconstruída com `PALETA[classes]` e as duas sobrepostas (máscara com `alpha=0.4`), lado a lado, no primeiro par da amostra (`SCC_33`).
- **Resultado:** o contorno do tecido acompanha as regiões coloridas, e os buracos da lâmina caem no fundo da máscara. A máscara corresponde à imagem.

### 8. Carga dos três conjuntos

- **Teste:** `load_images_list` em `train_list`, `test_list` e `validation_list`.
- **Resultado:**
  - 231 pares de treino, 31 de teste e 28 de validação;
  - nos três conjuntos, todas as imagens têm a mesma altura e largura do seu mapa de classes (`all(img.shape[:2] == cls.shape for img, cls in ...)` deu `True` para treino, teste e validação).

### 9. Preenchimento das lâminas menores que 512

- **Teste:** `random_cut(train[200][0], train[200][1])`, com a lâmina `IEC_3` (386 × 281), que tem os dois lados menores que 512. A imagem é completada com 255 (branco do vidro) e as classes com 0 (fundo), embaixo e à direita.
- **Resultado:** imagem `(512, 512, 3)` e classes `(512, 512)`.

### Medição preliminar: carregar o conjunto de treino

Feita fora do notebook, para planejar o próximo passo.

- **Teste:** ler as 231 lâminas de treino e converter as máscaras com a mesma conversão do notebook.
- **Resultado:**
  - cerca de **87 segundos**, a maior parte na conversão das máscaras;
  - cerca de **1,1 GB** de RAM (843 MB de imagens e 281 MB de mapas de classe);
  - todas as imagens têm a mesma altura e largura das suas máscaras.
