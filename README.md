# Solid Timeline

> A decentralized Facebook-style social feed where **you own your data**

[![Live Demo](https://img.shields.io/badge/demo-live-brightgreen)](https://solid-social.github.io/timeline/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Solid](https://img.shields.io/badge/solid-compatible-7C4DFF.svg)](https://solidproject.org)

**[Live Demo](https://solid-social.github.io/timeline/)** | **[Solid Project](https://solidproject.org)**

---

## What is this?

Solid Timeline is a **Facebook-style social feed** built on [Solid](https://solidproject.org) - Tim Berners-Lee's project to re-decentralize the web. Your posts, likes, and comments are stored on **your** Solid pod, not a corporate server.

```
Your Data → Your Pod → Your Control
```

## Features

- **Decentralized** - Posts stored on your Solid pod, not our servers
- **Facebook-style UI** - Familiar feed with posts, likes, and comments
- **No build step** - Pure HTML/JS, just open and use
- **Solid OIDC auth** - Login with any Solid identity provider
- **Real RDF data** - Uses SIOC vocabulary for semantic interoperability
- **Works offline** - Static files, host anywhere

## Quick Start

### Use it now

1. Go to **[solid-social.github.io/timeline](https://solid-social.github.io/timeline/)**
2. Click **Login** and enter your Solid IDP (e.g., `solidweb.org`)
3. Start posting!

### Self-host

```bash
# Clone and serve
git clone https://github.com/solid-social/timeline.git
cd timeline
npx serve .
# Open http://localhost:3000
```

No npm install. No build. Just files.

## How It Works

### Architecture

```
┌─────────────────┐     ┌─────────────────┐
│   Your Browser  │────▶│   Your Pod      │
│                 │     │                 │
│  Solid Timeline │     │  /public/       │
│  (static HTML)  │     │    timeline/    │
│                 │     │      2024-01-03.ttl
└─────────────────┘     └─────────────────┘
        │
        ▼
┌─────────────────┐
│  Solid IDP      │
│  (auth only)    │
└─────────────────┘
```

### Data Storage

Posts are stored as RDF in daily files on your pod:

```
https://you.solidweb.org/public/timeline/
├── 2024-01-01.ttl
├── 2024-01-02.ttl
└── 2024-01-03.ttl   ← Today's posts
```

### Data Format

Posts use the [SIOC](http://rdfs.org/sioc/spec/) vocabulary:

```turtle
@prefix sioc: <http://rdfs.org/sioc/ns#>.
@prefix dct: <http://purl.org/dc/terms/>.

<#post-1704307200000> a sioc:Post ;
    sioc:content "Hello, decentralized world!" ;
    dct:created "2024-01-03T12:00:00Z"^^xsd:dateTime ;
    dct:creator <https://you.solidweb.org/profile/card#me> .
```

This means your posts are:
- **Portable** - Move to any Solid pod
- **Interoperable** - Other apps can read them
- **Queryable** - Use SPARQL to search
- **Yours** - Delete anytime, no trace left behind

## Tech Stack

| Layer | Technology |
|-------|------------|
| UI | [Preact](https://preactjs.com/) + [htm](https://github.com/developit/htm) |
| Auth | [solid-oidc](https://www.npmjs.com/package/solid-oidc) |
| RDF | [rdflib.js](https://github.com/linkeddata/rdflib.js) |
| Build | None (ESM imports from esm.sh) |

**Zero build step.** Everything loads from CDN via ES modules.

## Configuration

The app auto-discovers your timeline from your WebID profile:

1. Looks for `solid:timeline` predicate
2. Falls back to `{pim:storage}/public/timeline/`

To set a custom location, add to your WebID:

```turtle
<#me> <http://www.w3.org/ns/solid/terms#timeline>
      <https://you.pod/custom/timeline/> .
```

## Get a Solid Pod

Don't have a pod? Get one free:

- [solidweb.org](https://solidweb.org) - Community server
- [solidcommunity.net](https://solidcommunity.net) - Inrupt's community server
- [inrupt.net](https://inrupt.net) - Inrupt's pod service

## Contributing

PRs welcome! Some ideas:

- [ ] Image/media uploads
- [ ] Real-time updates via WebSocket
- [ ] Following other timelines
- [ ] End-to-end encryption
- [ ] Mobile PWA

## Related Projects

- [Solid Chat](https://github.com/solid-chat/app) - Decentralized WhatsApp-style chat
- [Solid Project](https://solidproject.org) - The Solid specification
- [rdflib.js](https://github.com/linkeddata/rdflib.js) - RDF library for JavaScript

## License

MIT - Do whatever you want.

---

<p align="center">
  <i>Your data. Your pod. Your timeline.</i>
</p>
