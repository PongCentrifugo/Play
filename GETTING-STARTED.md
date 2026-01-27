# Getting Started

## Requirements

- Docker and Docker Compose
- Go 1.22+ (for local development)
- Node.js 18+ (for frontend development)

## Install

Clone the required repositories:

```bash
git clone https://github.com/PongCentrifugo/iac
git clone https://github.com/PongCentrifugo/backend
git clone https://github.com/PongCentrifugo/frontend
```

Start Redis:
```bash
cd iac/redis && docker-compose up -d
```

Start Centrifugo:
```bash
cd iac/centrifugo
cp config.example.json config.json
docker-compose up -d
```

Start Backend:
```bash
cd backend
cp .env.example .env
docker-compose up -d
```

Start Frontend:
```bash
cd frontend
npm install && npm run dev
```

Open `http://localhost:5173` to play.
