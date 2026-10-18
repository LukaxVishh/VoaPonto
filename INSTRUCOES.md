# Manual e Diretrizes de Engenharia e Desenvolvimento: VoaPonto

Este documento reúne todas as especificações técnicas, padrões de código, governança de dados, requisitos de segurança, fluxos transacionais e diretrizes operacionais para a equipe de desenvolvimento.

---

## 1. Princípios de Engenharia e Padrões de Projeto

### 1.1 Simplicidade, Clareza e Robustez
- **Evitar Abstrações Prematuras:** Código claro e direto é sempre superior a hierarquias profundas de classes e padrões de fábrica desnecessários.
- **Tipagem Estrita Obrigatória:** Proibido o uso de `any` no TypeScript. Tipos e interfaces devem ser compartilhados via biblioteca interna (`packages/shared-types`).
- **Validação na Fronteira (Input Validation):** Nenhuma requisição externa (REST, WebSocket ou Webhook) atinge as camadas de serviço sem ser validada via `Zod` ou `class-validator`.

### 1.2 Transacionalidade e Garantia ACID
Qualquer operação que altere o estado financeiro ou de lances deve ser executada obrigatoriamente dentro de uma transação de banco de dados (`BEGIN ... COMMIT`).

#### Exemplo de Boas Práticas (Feedback de Código)

❌ **Código Inadequado (Sem transação e suscetível a race conditions):**
```typescript
// PROBLEMA: Se a operação de pagamento falhar ou houver requisição concorrente,
// o status do leilão fica inconsistente com o saldo retido.
async function processMatch(auctionId: string, winningBidId: string) {
  const auction = await db.auction.findUnique({ where: { id: auctionId } });
  if (auction.status !== 'IN_AUCTION') throw new Error('Leilão não disponível');

  await db.auction.update({
    where: { id: auctionId },
    data: { status: 'AWAITING_PAYMENT', winningBidId },
  });

  await db.escrowAccount.create({
    data: { auctionId, amount: 1000, status: 'HELD' }
  });
}
```

✅ **Código Correto (Com transação ACID e bloqueio pessimista/otimista):**
```typescript
// CORREÇÃO: Transação atômica que garante consistência total ou rollback completo.
async function processMatch(auctionId: string, winningBidId: string) {
  return await db.$transaction(async (tx) => {
    const auction = await tx.auction.findUnique({
      where: { id: auctionId },
    });

    if (auction?.status !== 'IN_AUCTION') {
      throw new ConflictException('O leilão não está mais apto para match.');
    }

    const updatedAuction = await tx.auction.update({
      where: { id: auctionId },
      data: {
        status: 'AWAITING_PAYMENT',
        winningBidId,
        updatedAt: new Date(),
      },
    });

    await tx.escrowHold.create({
      data: {
        auctionId,
        winningBidId,
        amount: auction.currentLowestBid,
        status: 'PENDING_PAYMENT',
      },
    });

    return updatedAuction;
  });
}
```

---

## 2. Estrutura do Repositório (Monorepo Turborepo)

```
leilao/
├── README.md                 # Especificação completa da visão de negócio, custos e regras
├── INSTRUCOES.md             # Manual de engenharia e diretrizes de desenvolvimento
├── package.json              # Configuração raiz do monorepo (pnpm workspaces)
├── turbo.json                # Pipeline de builds e caches distribuídos
├── apps/
│   ├── api/                  # Backend principal (NestJS, REST, WebSockets, BullMQ)
│   │   ├── src/
│   │   │   ├── modules/
│   │   │   │   ├── auth/          # Autenticação, JWT, KYC e RBAC
│   │   │   │   ├── auctions/      # Engine de leilão reverso e cronômetros
│   │   │   │   ├── escrow/        # Custódia, split e webhooks de pagamento
│   │   │   │   ├── tickets/       # Submissão de PNR, OCR e workers de checagem
│   │   │   │   ├── sellers/       # Gestão de contas de milhas e limite de CPFs
│   │   │   │   └── notifications/ # Push notifications (Expo) e mensageria
│   │   │   └── main.ts
│   ├── web/                  # Painel Web do Passageiro e Vendedor (Next.js 14+)
│   │   ├── src/
│   │   │   ├── app/          # App Router com Server Components e SEO
│   │   │   ├── components/   # Componentes modulares com Tailwind e Shadcn UI
│   │   │   └── hooks/        # Hooks de queries (TanStack Query)
│   └── mobile/               # App Mobile Nativo (React Native com Expo SDK 51+)
│       ├── app/              # Expo Router (navegação nativa por rotas)
│       ├── components/       # Componentes de interface móvel nativa
│       └── services/         # Integração com câmera, biometria e push notifications
├── packages/
│   ├── database/             # Schema Prisma/PostgreSQL, migrações e seeds
│   │   └── prisma/
│   │       └── schema.prisma
│   ├── shared-types/         # Tipagens TypeScript compartilhadas entre API, Web e Mobile
│   ├── eslint-config/        # Regras unificadas de linting
│   └── tsconfig/             # Configurações padrão de TypeScript
└── docs/                     # Diagramas, especificações de endpoints e fluxos
```

---

## 3. Segurança, LGPD e Autenticação

### 3.1 Tratamento de Dados Pessoais de Passageiros
- Os dados do comprador e dos passageiros (Nome completo, CPF, Data de Nascimento, E-mail e Telefone) são **estritamente mascarados** durante a fase de leilão. O vendedor tem acesso apenas ao trecho, data do voo e quantidade de passageiros.
- Os dados reais só são revelados ao vendedor vencedor **após** a confirmação do pagamento integral retido no Escrow.
- Todas as visualizações de dados sensíveis geram registros de auditoria imutáveis (`audit_logs`) com IP, User-Agent e carimbo de tempo.

### 3.2 Segurança nos Webhooks de Pagamento
- Qualquer evento de webhook disparado pelo gateway (ex: `PIX_RECEIVED`, `CHARGE_REFUNDED`) deve ser validado através de assinatura digital HMAC no header da requisição antes de desencadear mudanças de status.
- Todas as rotinas de webhook devem ser **estritamente idempotentes**: caso o gateway envie a mesma notificação múltiplas vezes, o sistema deve ignorar eventos já processados sem causar efeitos colaterais duplicados.

---

## 4. Política de Tratamento de Erros e Logs

1. **Códigos de Resposta HTTP Padronizados:**
   - `400 Bad Request`: Erro de validação de dados de entrada.
   - `401 Unauthorized`: Token JWT ausente ou expirado.
   - `403 Forbidden`: Usuário autenticado tentando acessar recurso sem permissão de perfil.
   - `404 Not Found`: Recurso não localizado.
   - `409 Conflict`: Conflito de estado de negócio (ex: lance submetido após o leilão ser encerrado).
   - `500 Internal Server Error`: Falhas inesperadas (devem ser capturadas por interceptor global e registradas no serviço de monitoramento).
2. **Logs Estruturados:** Utilizar `Pino` ou `Winston` com formato JSON estruturado contendo `correlationId` em cada requisição para facilitar a rastreabilidade entre serviços.
