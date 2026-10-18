# Plataforma de Leilão e Intermediação Segura de Passagens com Milhas

## 1. Visão Geral do Projeto
A plataforma visa conectar detentores de milhas e pontos de programas de fidelidade (como Smiles, LATAM Pass e Azul Fidelidade) que buscam monetizar seu saldo a pessoas interessadas em adquirir passagens aéreas por preços mais vantajosos.

O modelo opera como um **leilão reverso com custódia garantida (Escrow)**:
- O passageiro solicita o voo desejado e estipula o valor máximo que aceita pagar.
- Os vendedores competem oferecendo o menor valor viável para emitir a passagem com suas milhas.
- O pagamento do passageiro fica retido na plataforma até que a emissão seja validada e confirmada.

---

## 2. Princípios de Negócio e Regras Fundamentais

1. **Custódia Garantida (Escrow):**
   - O vendedor não recebe dinheiro antecipadamente do comprador.
   - O vendedor só é acionado para emitir após a confirmação do pagamento integral pelo comprador no gateway da plataforma.
   - O comprador recebe reembolso integral (100%) caso o vendedor não cumpra o prazo de emissão ou forneça dados inválidos.

2. **Rastreabilidade e Validação da Emissão:**
   - Para solicitar a liberação do repasse, o vendedor deve submeter o PNR (código localizador), o sobrenome do passageiro principal e o PDF oficial da emissão.
   - A plataforma checa a correspondência dos dados (nomes, trechos, datas) e valida o bilhete antes da conclusão financeira.

3. **Gestão de Limite de Beneficiários (CPFs):**
   - As companhias aéreas possuem limites anuais estritos de CPFs distintos para emissão por conta de milhas (ex: Smiles ~25 CPFs/ano, LATAM ~24/25 CPFs/ano, Azul ~5 CPFs pré-cadastrados).
   - O sistema controla a quantidade de CPFs restantes declarada por cada vendedor para evitar aceites de pedidos incompatíveis.

---

## 3. Arquitetura Técnica Sugerida

- **Backend:** Node.js com TypeScript (NestJS ou Fastify)
- **Banco de Dados:** PostgreSQL com suporte estrito a transações ACID
- **Mensageria e Filas:** Redis com BullMQ (gestão de timeouts de leilão e filas de validação)
- **Frontend:** Next.js (React) + Tailwind CSS
- **Gateway Financeiro & Split:** Asaas ou Pagar.me (suporte a PIX com QR code dinâmico, subcontas e split automático de pagamento)
