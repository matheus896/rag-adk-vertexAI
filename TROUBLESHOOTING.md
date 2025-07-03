# Guia de Solução de Problemas para o Agente ADK RAG

Este documento detalha os problemas encontrados durante a configuração e execução do agente ADK RAG e as etapas tomadas para resolvê-los.

---

### Problema 1: `403 PERMISSION_DENIED`

**Sintoma:**
Ao tentar executar o agente, a seguinte mensagem de erro era exibida:
```
google.genai.errors.ClientError: 403 PERMISSION_DENIED. {'error': {'code': 403, 'message': "Permission 'aiplatform.endpoints.predict' denied on resource '...'"}}
```

**Causa:**
A identidade que executava a chamada para a API da Vertex AI (inicialmente, a conta de usuário `...@gmail.com`) não possuía a permissão IAM `aiplatform.endpoints.predict`, que é necessária para fazer previsões em modelos da AI Platform.

**Solução Inicial (Incompleta):**
A primeira tentativa de solução foi conceder o papel `Vertex AI User` (`roles/aiplatform.user`) diretamente à conta de usuário através do comando:
```bash
gcloud projects add-iam-policy-binding adk-rag-464720 --member="user:...@gmail.com" --role="roles/aiplatform.user"
```
Embora o comando tenha sido executado com sucesso, o erro persistiu, indicando que o ambiente de execução não estava utilizando as credenciais do usuário.

---

### Problema 2: Persistência do `403 PERMISSION_DENIED` e Autenticação

**Causa:**
A investigação revelou que o aplicativo estava configurado para usar uma **Conta de Serviço (Service Account)** para autenticação, e não as credenciais do usuário logado no gcloud. A conta de serviço em uso também não tinha as permissões necessárias.

**Solução:**
A solução foi configurar o ambiente para usar uma nova conta de serviço com as permissões adequadas.

1.  **Criação da Chave da Conta de Serviço:** Uma nova chave JSON foi gerada para a conta de serviço `teste-adk-rag-4@adk-rag-464720.iam.gserviceaccount.com`.
    ```bash
    gcloud iam service-accounts keys create "adk-rag-agent/rag_agent/service-account-key.json" --iam-account="teste-adk-rag-4@adk-rag-464720.iam.gserviceaccount.com"
    ```

2.  **Configuração do Ambiente:** O arquivo `.env` foi atualizado para que o aplicativo usasse essa chave, adicionando a seguinte variável de ambiente:
    ```
    GOOGLE_APPLICATION_CREDENTIALS="service-account-key.json"
    ```

---

### Problema 3: `DefaultCredentialsError: File ... not found`

**Sintoma:**
Após configurar a conta de serviço, um novo erro surgiu:
```
google.auth.exceptions.DefaultCredentialsError: File service-account-key.json was not found.
```

**Causa:**
O caminho para o arquivo de credenciais (`service-account-key.json`) no arquivo `.env` estava incorreto. O script era executado a partir do diretório `adk-rag-agent/`, mas o arquivo de chave estava localizado em `adk-rag-agent/rag_agent/`. O caminho relativo estava errado.

**Solução:**
O caminho no arquivo `adk-rag-agent/rag_agent/.env` foi corrigido para apontar para a localização correta do arquivo de chave:
```diff
- GOOGLE_APPLICATION_CREDENTIALS="service-account-key.json"
+ GOOGLE_APPLICATION_CREDENTIALS="rag_agent/service-account-key.json"
```

---

### Problema 4: `404 NOT_FOUND` para o Modelo

**Sintoma:**
Finalmente, o erro mudou para:
```
google.genai.errors.ClientError: 404 NOT_FOUND. {'error': {'code': 404, 'message': 'Publisher Model `.../models/gemini-2.5-flash-lite` was not found...'}}
```

**Causa:**
O nome do modelo especificado no código (`gemini-2.5-flash-lite`) estava incorreto ou não estava disponível na região `us-central1` para o projeto.

**Solução:**
O nome do modelo no código do agente foi corrigido para um modelo válido e disponível, como `gemini-2.5-flash`. Após essa alteração, o aplicativo foi executado com sucesso.
