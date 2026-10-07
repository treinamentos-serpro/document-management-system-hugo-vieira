# Especificação - Document Management System

## 1. Objetivo

Permitir que usuários enviem, consultem e baixem seus documentos, mantendo os arquivos no filesystem local e os metadados em memória.

## 2. Escopo

### Dentro do escopo

- Upload de um documento por requisição.
- Listagem dos documentos pertencentes ao usuário da requisição.
- Download de um documento pelo identificador, limitado ao proprietário.
- Identificação simples do proprietário por requisição.
- Armazenamento dos arquivos localmente com `multer` e `diskStorage`.
- Armazenamento dos metadados em memória durante a execução do processo.

### Fora do escopo

- Autenticação, autorização robusta e cadastro de usuários.
- Armazenamento externo, em nuvem ou em banco de dados.
- Versionamento, edição, exclusão ou compartilhamento de documentos.
- Persistência dos metadados após reinicialização do backend.
- Varredura antivírus ou validação avançada do conteúdo dos arquivos.

## 3. Requisitos funcionais

| ID | Requisito |
| --- | --- |
| RF-01 | O usuário envia um único arquivo no campo `file` de uma requisição multipart. |
| RF-02 | O sistema atribui um identificador único e um nome de armazenamento não derivado do nome original. |
| RF-03 | O sistema grava o conteúdo do arquivo no diretório local configurado. |
| RF-04 | O sistema registra em memória o identificador, nome original, tamanho, data de upload, proprietário e dados necessários ao download. |
| RF-05 | O usuário lista somente os documentos associados ao identificador de usuário enviado na requisição. |
| RF-06 | O usuário baixa um documento pelo identificador, desde que pertença ao usuário da requisição. |
| RF-07 | O sistema retorna erro apropriado para arquivo ausente, requisição inválida, documento inexistente ou arquivo acima do limite. |
| RF-08 | O nome original é preservado para apresentação e download, mas não é usado como caminho de armazenamento. |

A identidade é enviada no cabeçalho `X-User-Id`. Ela serve apenas para separar os documentos no protótipo; não comprova a identidade de quem fez a requisição.

## 4. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | Os arquivos são gravados no filesystem local com `multer` configurado com `diskStorage`. |
| RNF-02 | Os metadados ficam em memória e são perdidos quando o processo backend reinicia. |
| RNF-03 | O diretório de armazenamento e o limite de tamanho são configuráveis por variáveis de ambiente. |
| RNF-04 | O backend usa CommonJS, Express e o runner de testes nativo do Node.js. |
| RNF-05 | A comunicação do frontend com o backend usa `fetch` e o prefixo `/api`; o proxy do Vite remove esse prefixo no desenvolvimento. |
| RNF-06 | Os endpoints não expõem o nome interno do arquivo armazenado nem permitem fornecer caminhos de filesystem. |
| RNF-07 | Erros são tratados nas fronteiras HTTP e de filesystem, sem expor stack traces ao cliente. |

Configuração inicial proposta:

| Variável | Padrão | Uso |
| --- | --- | --- |
| `PORT` | `3000` | Porta HTTP do backend. |
| `STORAGE_DIR` | `backend/storage` | Diretório local dos arquivos enviados. |
| `MAX_FILE_SIZE_BYTES` | `10485760` | Limite de upload, inicialmente 10 MiB. |

## 5. Modelo de dados

### Metadados do documento

| Campo | Tipo | Exposição | Descrição |
| --- | --- | --- | --- |
| `id` | string | API | Identificador único, gerado pelo servidor. |
| `originalName` | string | API | Nome original enviado pelo cliente. |
| `size` | number | API | Tamanho do arquivo em bytes. |
| `uploadedAt` | string | API | Data e hora do upload em ISO 8601. |
| `owner` | string | API | Valor de `X-User-Id` usado no upload. |
| `storageName` | string | Interno | Nome gerado pelo servidor para o arquivo no disco. |
| `contentType` | string | Interno | Tipo de conteúdo informado pelo upload, com fallback seguro se ausente. |

O repositório mantém os registros indexados por `id` em memória. `storageName` é um nome opaco gerado pelo servidor; nunca deve ser formado a partir de entrada do usuário.

### Identidade do usuário

`X-User-Id` é obrigatório, não vazio e tratado como identificador opaco. Como não há autenticação neste escopo, clientes podem falsificar esse valor. O mecanismo é adequado apenas ao protótipo local e não deve ser considerado controle de acesso para produção.

## 6. Contratos de API

Todas as respostas JSON de erro seguem o formato:

```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Descrição legível do erro."
  }
}
```

### `POST /upload`

No frontend, chamar `/api/upload`; o proxy do Vite encaminha para `/upload` no backend.

- Cabeçalho obrigatório: `X-User-Id`.
- Corpo: `multipart/form-data`, com exatamente um arquivo no campo `file`.
- Limite: `MAX_FILE_SIZE_BYTES`.
- Sucesso: `201 Created`, corpo JSON com os metadados públicos do documento.

```json
{
  "id": "identificador",
  "originalName": "relatorio.pdf",
  "size": 2048,
  "uploadedAt": "2026-10-07T12:00:00.000Z",
  "owner": "usuario-1"
}
```

Erros esperados: `400` para identidade ou arquivo ausente/inválido; `413` para arquivo acima do limite; `500` para falha de gravação.

### `GET /documents`

No frontend, chamar `/api/documents`.

- Cabeçalho obrigatório: `X-User-Id`.
- Sucesso: `200 OK`, corpo JSON com uma lista, que pode estar vazia.
- A lista contém apenas documentos cujo `owner` corresponde ao cabeçalho.
- Cada item usa os campos públicos do contrato de upload; não inclui `storageName`.

Erros esperados: `400` para identidade ausente ou inválida; `500` para falha inesperada.

### `GET /documents/:id/download`

No frontend, chamar `/api/documents/:id/download`.

- Cabeçalho obrigatório: `X-User-Id`.
- Sucesso: `200 OK`, conteúdo binário do arquivo.
- Usar `Content-Disposition: attachment` com o nome original sanitizado e `Content-Type` compatível com os metadados disponíveis.
- Responder `404` tanto para documento inexistente quanto para documento de outro proprietário.
- Responder `404` se o registro existir, mas o arquivo local não puder ser encontrado.

Erros esperados: `400` para identidade ou identificador inválido; `404` para recurso indisponível; `500` para falha inesperada de leitura.

## 7. Decisões arquiteturais e riscos

- Fluxo de dependências do backend: `routes -> controllers -> services -> repositories`.
- `routes/` declara endpoints e conecta middleware e controllers.
- `middleware/` configura o upload com `multer`; o destino usa `diskStorage` e o diretório local configurado.
- `controllers/` valida dados HTTP, delega ao serviço e define status e resposta.
- `services/` aplica regras de propriedade e coordena operações de documentos.
- `repositories/` encapsula gravação/leitura no filesystem e acesso aos metadados em memória.
- O frontend mantém componentes em `components/`, páginas em `pages/` e chamadas HTTP em `services/`.
- O prefixo usado pelo frontend é `/api`; o proxy existente do Vite o remove antes de encaminhar ao backend.
- O armazenamento é exclusivamente local. Não introduzir provedores externos.
- Reiniciar o backend perde os metadados, embora arquivos possam permanecer no disco; isso pode deixar arquivos órfãos.
- Falhas entre gravação do arquivo e registro do metadado podem deixar arquivos sem referência. O serviço deve tratar a falha e tentar remover o arquivo recém-gravado.
- `X-User-Id` não autentica o usuário. O isolamento por proprietário reduz exposição acidental, mas não protege contra falsificação do cabeçalho.
- O limite de upload deve ser aplicado pelo middleware, antes de carregar o arquivo integralmente em memória.

## 8. Plano de execução

As etapas abaixo descrevem trabalho futuro. Esta especificação não executa nem altera arquivos de backend ou frontend.

### Etapa 1 - Configuração e contratos do backend

Arquivos previstos: `backend/src/app.js` e novos arquivos em `backend/src/routes/`, `backend/src/controllers/` e `backend/src/middleware/`.

Critérios de aceite:
- O backend expõe os endpoints definidos e mantém `/health`.
- Configuração de porta, diretório e limite de upload tem valores padrão documentados.
- Requisições sem `X-User-Id` recebem erro consistente.

### Etapa 2 - Upload e persistência local

Arquivos previstos: novos arquivos em `backend/src/services/` e `backend/src/repositories/`; middleware de upload em `backend/src/middleware/`; testes em `backend/test/app.test.js`.

Critérios de aceite:
- Upload válido grava o arquivo em `STORAGE_DIR` e retorna metadados com `201`.
- O nome no disco é gerado pelo servidor.
- Arquivo ausente, limite excedido e falha de gravação são tratados sem expor detalhes internos.
- Metadados do upload ficam disponíveis em memória.

### Etapa 3 - Listagem e download com isolamento por proprietário

Arquivos previstos: controllers, services e repositories do backend; testes em `backend/test/app.test.js`.

Critérios de aceite:
- A listagem retorna somente os documentos do usuário indicado.
- O proprietário pode baixar seu documento pelo identificador.
- Documento inexistente e documento pertencente a outro usuário retornam `404`.
- Arquivo desaparecido no disco é tratado como indisponível.

### Etapa 4 - Interface e integração do frontend

Arquivos previstos: novos componentes em `frontend/src/components/`, serviço em `frontend/src/services/` e ajustes em `frontend/src/App.jsx`.

Critérios de aceite:
- A interface permite selecionar e enviar um arquivo, consultar a lista e iniciar o download.
- As chamadas usam `fetch` pelo prefixo `/api`.
- Estados de carregamento, sucesso, lista vazia e erro são apresentados ao usuário.
- O cabeçalho `X-User-Id` é enviado conforme a decisão de identidade do protótipo.

### Etapa 5 - Verificação integrada e documentação

Arquivos previstos: testes existentes em `backend/test/app.test.js` e documentação do projeto em `README.md`.

Critérios de aceite:
- `npm test` no backend cobre upload, listagem, propriedade, download e erros principais.
- `npm run build` no frontend conclui sem erros.
- A documentação explica inicialização, variáveis de ambiente, contratos e limitações de persistência e autenticação.
