# Simple Vault — неделя 9: депозит и share-токены

Учебная программа Solana на Anchor 0.31.1. Эта версия продолжает проверенную реализацию недели 8 и добавляет только депозит.

## Реализовано

### Неделя 8 — сохранено

- `initialize_vault` создаёт `VaultState` PDA, SPL token account хранилища и `share_mint`.
- `create_user_card` создаёт персональную `UserCard` PDA для пользователя.
- Один пользователь может иметь одну карточку в рамках одного vault.

### Неделя 9 — добавлено

Инструкция:

```rust
deposit(ctx: Context<Deposit>, amount: u64)
```

Пользователь передаёт количество базовых SPL-токенов в минимальных единицах mint (для mint с 6 decimals: `1 token = 1_000_000`).

Алгоритм:

1. Проверяется, что `amount > 0`.
2. Рассчитывается количество share-токенов:
   - первый депозит: `shares = amount`;
   - последующие депозиты: `shares = amount * total_shares / total_tokens`.
3. Выполняется CPI `transfer` в SPL Token Program: токены переходят с пользовательского token account в `vault_token_account`. Подписывает пользователь.
4. Выполняется CPI `mint_to` в SPL Token Program: share-токены выпускаются на `user_share_account`. Подписывает VaultState PDA через seeds.
5. Обновляются `VaultState.total_tokens`, `VaultState.total_shares`, `UserCard.deposited` и `UserCard.shares`.

Расчёт выполняется через `u128` и checked arithmetic, чтобы избежать переполнения при умножении. Нулевой депозит и депозит, при котором получилось бы 0 shares, отклоняются. Проверяются PDA, mint и token authority перед выполнением инструкции.

### Пример

Если первый пользователь внёс 100 токенов, он получает 100 shares. Если второй пользователь затем внёс 50 токенов, он получает 50 shares. В результате:

- `vault_token_account`: 150 токенов;
- `share_mint.supply`: 150 shares;
- `VaultState.total_tokens`: 150;
- `VaultState.total_shares`: 150.

## Что намеренно НЕ реализовано

- `withdraw`;
- сжигание share-токенов (`burn`);
- возврат токенов из vault через CPI transfer с подписью PDA.

Это содержание недели 10. В этой версии есть только депозит и выпуск shares.

## Структура проекта

```text
vault-dev-students-vault-card/
├── programs/
│   └── simple_vault/
│       ├── Cargo.toml
│       └── src/
│           └── lib.rs
├── tests/
│   ├── week8.ts
│   └── week9.ts
├── Anchor.toml
├── Cargo.toml
├── package.json
└── README.md
```

## Требования

- Rust stable
- Solana CLI 1.18+
- Anchor CLI 0.31.1
- Node.js 18+
- Yarn

## Установка и сборка

```bash
yarn install
anchor build
```

Если используешь новый program keypair, синхронизируй program ID и пересобери:

```bash
anchor keys sync
anchor build
```

## Развёртывание и тестирование на devnet

В `Anchor.toml` настроен devnet и кошелёк `~/.config/solana/id.json`.

```bash
solana config set --url devnet
solana address
solana balance
anchor deploy --provider.cluster devnet
yarn test
```

Сначала обязательно задеплой обновлённую программу недели 9. Скрипт `yarn test` запускает `anchor test --provider.cluster devnet --skip-local-validator --skip-deploy`, то есть тесты не выполняют deploy автоматически.

Для тестов кошельку администратора нужны SOL: он оплачивает создание mint/token accounts и переводит небольшое количество SOL тестовым пользователям.

## Проверки

`tests/week8.ts` — регрессионные проверки недели 8:

- создание vault и share mint;
- создание пользовательских PDA-карточек;
- запрет повторного создания карточки.

`tests/week9.ts` — проверки недели 9:

- пользователь A вносит 100 токенов и получает 100 shares;
- пользователь B вносит 50 токенов и получает 50 shares;
- проверяются балансы token accounts, supply share mint, счётчики vault и поля UserCard;
- нулевой депозит отклоняется;
- депозит без UserCard отклоняется;
- депозит сверх баланса отклоняется Token Program;
- после неуспешных транзакций состояние vault не изменяется.

> В среде подготовки архива нет установленного Cargo и Anchor CLI, поэтому фактическая сборка и выполнение devnet-тестов здесь не подтверждены. Выполни команды выше в своей среде и пришли ошибки, если они появятся.
