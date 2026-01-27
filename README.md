# Pong with Centrifugo

![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat&logo=amazonwebservices&logoColor=white)
![Amazon EKS](https://img.shields.io/badge/EKS-FF9900?style=flat&logo=amazoneks&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat&logo=terraform&logoColor=white)
![ArgoCD](https://img.shields.io/badge/ArgoCD-EF7B4D?style=flat&logo=argo&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Go](https://img.shields.io/badge/Go-00ADD8?style=flat&logo=go&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=flat&logo=javascript&logoColor=black)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat&logo=redis&logoColor=white)

A real-time multiplayer Pong game built with Centrifugo WebSocket server, Go backend, and vanilla JavaScript frontend. Players connect via WebSocket for low-latency gameplay, with automatic disconnect handling through Redis presence monitoring.

**Live demo:** [pong.stabalmo.pro](https://pong.stabalmo.pro)

<p align="center">
  <a href="https://pong.stabalmo.pro">
    <img src="./preview-game.gif" alt="Pong gameplay preview" width="720" />
  </a>
</p>

## Get started

See the [Getting Started](./GETTING-STARTED.md) guide for installation instructions.

## Repositories

| Repository | Description |
|---|---|
| [frontend](https://github.com/PongCentrifugo/frontend) | Vanilla JavaScript game client with canvas rendering and Centrifugo WebSocket integration |
| [backend](https://github.com/PongCentrifugo/backend) | Go REST API handling game logic, JWT authentication, and Redis presence monitoring |
| [iac](https://github.com/PongCentrifugo/iac) | Infrastructure as Code for Redis and Centrifugo services |
| [terraform](https://github.com/PongCentrifugo/terraform) | AWS infrastructure provisioning with EKS, ElastiCache Redis, S3, and CloudFront |

## Next steps

- Read the [Technical Architecture Guide](https://github.com/PongCentrifugo/.github/blob/main/TECHNICAL-ARCHITECTURE.md) for in-depth documentation on how Centrifugo, Redis, and Backend work together.
- Learn about [Centrifugo](https://centrifugal.dev/) real-time messaging server.
- Browse the backend API in the [backend repository](https://github.com/PongCentrifugo/backend).

## How it works

Players authenticate via JWT tokens and connect to Centrifugo WebSocket channels. Game state updates flow through dedicated channels for each player position. When a player disconnects (browser close, network failure, or page refresh), the backend detects this through dual strategies: Redis pub/sub for instant leave events, and presence monitoring for stale connections. The game automatically ends and the lobby resets.

## Development approach

The backend and frontend were largely written using vibe-coding (AI-assisted development). My focus was on system architecture, infrastructure design, and understanding how Centrifugo, Redis, and real-time communication work together. I'm not proficient in Go or Vue/JavaScript, but that didn't stop me from building a working multiplayer game.

This approach lets you focus on what matters most — architecture decisions, SRE practices, and understanding the system as a whole — while AI handles the implementation details. Even [Linus Torvalds uses AI assistance](https://github.com/torvalds/AudioNoise/commit/71b256a7fcb0aa1250625f79838ab71b2b77b9ff) for his personal projects.

## Contributing

Contributions are welcomed and encouraged. Pull requests welcome. For major changes, open an issue first.

## About

A real-time multiplayer Pong implementation demonstrating WebSocket-based game architecture with Centrifugo, Go, and vanilla JavaScript.

### Resources

- [README](./README.md)
- [Technical Architecture](https://github.com/PongCentrifugo/.github/blob/main/TECHNICAL-ARCHITECTURE.md)
- [MIT License](./LICENSE)
- [Security Policy](./SECURITY.md)

### Author

Created by [Danila Alferov](https://stabalmo.pro) ([LinkedIn](https://www.linkedin.com/in/stabalmo), [@Stabalmo](https://github.com/Stabalmo))
