# Spotify

*A Spotify Web API client with the complete authorisation code flow, browsing, search, library and playback.*

![Swift](https://img.shields.io/badge/Swift-5.0-F05138?style=flat-square&logo=swift&logoColor=white) ![UIKit](https://img.shields.io/badge/UIKit-2396F3?style=flat-square&logo=uikit&logoColor=white) ![iOS](https://img.shields.io/badge/iOS-17.4%2B-000000?style=flat-square&logo=apple&logoColor=white) ![Architecture](https://img.shields.io/badge/architecture-MVVM-6366F1?style=flat-square) ![Dependencies](https://img.shields.io/badge/SDWebImage-8B5CF6?style=flat-square) ![Dependencies](https://img.shields.io/badge/AVFoundation-0071E3?style=flat-square)

![App](https://github.com/user-attachments/assets/8188db37-b2ec-428a-9f22-be7b822136a3)

## Overview

The application signs a user in against their real Spotify account, keeps the token pair fresh, and
then drives browse, search, playlist, album, category and library screens from the Web API. Playback
runs through `AVPlayer`, with a single presenter deciding whether a tap plays one track or a whole
queue.

The interface is written entirely in code. There is no storyboard.

## Authorisation flow

```mermaid
sequenceDiagram
    participant U as User
    participant W as WelcomeViewController
    participant A as AuthViewController
    participant M as AuthManager
    participant S as Spotify Accounts

    U->>W: Sign in
    W->>A: present WKWebView with signInURL
    A->>S: GET /authorize?response_type=code
    S-->>A: redirect carrying ?code=
    A->>M: exchangeCodeForToken(code)
    M->>S: POST /api/token with Basic auth
    S-->>M: access token, refresh token, expiry
    M->>M: cacheToken in UserDefaults
    M-->>A: success
    A-->>W: dismiss and show TabBarViewController
```

Every later call goes through `withValidToken`, which refreshes the token when it is within the
expiry window and queues callers that arrive while a refresh is already running, so a burst of
requests triggers only one refresh.

## Architecture

```mermaid
flowchart TD
    subgraph Screens
        H["HomeViewController"]
        SR["SearchViewController"]
        L["LibraryViewController"]
        P["PlaylistViewController"]
        AL["AlbumViewController"]
    end
    H --> API["APICaller"]
    SR --> API
    L --> API
    P --> API
    AL --> API
    API --> AUTH["AuthManager<br/>withValidToken"]
    AUTH --> TOK["UserDefaults token store"]
    API --> WEB["api.spotify.com/v1"]
    H --> CVM["Cell view models"]
    CVM --> IMG["SDWebImage"]
    P --> PP["PlaybackPresenter"]
    AL --> PP
    PP --> PVC["PlayerViewController"]
    PVC --> PCV["PlayerControlsView"]
    PP --> AV["AVPlayer / AVQueuePlayer"]
```

`APICaller` exposes one method per endpoint, each building a request through a shared
`createRequest` helper and decoding into a dedicated response model. Screens never see tokens or URLs.

## Implementation notes

- **Compositional layout for browse.** The home screen composes new releases, featured playlists and
  recommended tracks as three sections with different item sizing inside one collection view.
- **Playback behind a presenter.** `PlaybackPresenter` holds the track or track list, chooses between
  `AVPlayer` and `AVQueuePlayer`, and acts as the data source for the player screen, so no view
  controller owns the audio session.
- **Delegated controls.** `PlayerControlsView` reports intent (play, pause, next, back, volume) to its
  delegate instead of touching the player, which keeps the view reusable.
- **One response model per endpoint.** Search, categories, albums, playlists and recommendations each
  decode into their own `Codable` type rather than a shared loose dictionary.
- **Haptics as a service.** `HapticsManager` centralises feedback so call sites stay one line.

## Project structure

```
Spotify/
├── Managers/       AuthManager, APICaller, HapticsManager
├── Models/         Codable response types for every endpoint
├── ViewModels/     cell view models for browse, search and playlists
├── Presenter/      PlaybackPresenter
├── Controllers/    Core tab controllers and Other detail controllers
└── Views/          browse cells, player controls, headers, search result cells
```

## Requirements

Xcode 15 or later, iOS 17.4 or later, CocoaPods for SDWebImage. A Spotify developer client ID and
secret are required, together with a redirect URI registered in the Spotify dashboard.
