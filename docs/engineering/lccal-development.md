# Local Development

## Requirements

- Git
- Node.js
- pnpm
- Docker

## Setup

Clone the repository.

Install dependencies:

```bash
pnpm install
```

Start infrastructure:

```bash
docker compose up -d
```

Copy environment configuration:

```bash
cp .env.example .env
```

Start development servers:

```bash
pnpm dev
```

**Useful commands**

Start infrastructure:

```bash
docker compose up -d
```

Stop infrastructure:

```bash
docker compose down
```

View logs:

```bash
docker compose logs -f
```

Run tests:

```bash
pnpm test
```

Run lint:

```bash
pnpm lint
```

Run type checking:

```bash
pnpm typecheck

```