# Metrolist Copilot Instructions

## Project Overview
Metrolist is a YouTube Music client for Android built with **Kotlin, Jetpack Compose, and Media3**. It's a multi-module Android app with seven specialized library modules plus the main app module.

### Core Architecture
- **Single-activity** Compose UI with navigation-compose for screen routing
- **Room database** (v30 migrations) for local persistence of songs, playlists, artists, albums, and listening history
- **Hilt DI** for dependency injection across modules
- **Media3 ExoPlayer** for audio playback with background service capability
- **Innertube module** wraps YouTube API calls; other modules provide features (lyrics, Discord presence, equalizer)

### Module Organization
```
app/              # Main UI, playback service, database layer
innertube/        # YouTube API abstraction (Browse, Search, Player endpoints)
betterlyrics/     # Time-synced lyrics with word-level highlighting
kizzy/            # Discord Rich Presence integration
kugou/            # Kugou lyrics provider fallback
lrclib/           # LRC lyrics provider
lastfm/           # Last.fm API integration for scrobbling
simpmusic/        # SimpMusic lyrics API fallback
```

## Key Architectural Patterns

### Database & Data Layer
- **Room with AutoMigrations**: Uses schema exports in `app/schemas/` with both auto-migrations and manual `Migration*To*` specs
- **Current version**: 30 (see `MusicDatabase.kt` for migration history up to v30, handles complex data transformations)
- **Key entities** in `db/entities/`: `SongEntity`, `PlaylistEntity`, `ArtistEntity`, `AlbumEntity`, `FormatEntity`, `LyricsEntity`, `Event`, `PlayCountEntity`
- **Foreign keys & cascade**: Many-to-many maps (`PlaylistSongMap`, `SongArtistMap`, etc.) with `onDelete = CASCADE`
- **Database views**: `PlaylistSongMapPreview`, `SortedSongArtistMap` for efficient queries
- **Converters**: `LocalDateTime` ↔ timestamp conversion via `Converters` class

### UI Patterns (Jetpack Compose)
- **Screen structure**: Each screen is a top-level `@Composable` function taking `NavController` + optional `TopAppBarScrollBehavior`
- **State management**: Heavy use of `collectAsState()` from Flow/StateFlow; prefer `rememberSaveable()` for state persistence across config changes
- **ViewModel injection**: `hiltViewModel()` for injected ViewModels with reactive data streams
- **Layout patterns**:
  - `LazyVerticalGrid` with `GridCells.Adaptive()` for adaptive grid items (see `AccountScreen.kt`)
  - `LazyColumn`/`LazyRow` for scrollable lists with `items()` or `itemsIndexed()`
  - `TopAppBar` with optional `TopAppBarScrollBehavior` for scroll-connected headers
  - Context via `LocalPlayerConnection.current`, `LocalMenuState.current`, `LocalDatabase.current`

### Playback Service
- **MusicService**: Foreground service extending `MediaLibrarySession` (Media3) with queue management
- **PlayerConnection**: Singleton wrapper providing reactive playback state (via Flow/StateFlow)
  - Key streams: `isEffectivelyPlaying`, `mediaMetadata`, `currentMediaItem`, `queue`
  - Queue implementations in `playback/queues/` handle YouTube, Library, History queue types
- **Download & Cache**: Separate caches via `SimpleCache` (ExoPlayer, Download) with configurable eviction policies
- **Service lifecycle**: Foreground notification in `NotificationUtil.kt` handles notification updates on playback changes

### Navigation & Routing
- **Graph structure**: Central `NavigationBuilder.kt` defines composable routes
- **Arguments**: `navArgument()` with custom `NavType`s for type-safe routing (e.g., `browseId` strings)
- **Deep linking**: Supports YouTube sharing (e.g., `browse/{browseId}` route)
- **Menu state**: `LocalMenuState` CompositionLocal for context menus (long-press actions)

### API Integration
- **Innertube YouTube client**: Singleton in DI (`YouTube` object with suspend functions)
  - Key endpoints: `Browse` (playlists, albums), `Search`, `Player` (get format options), `Lyrics`
- **External providers**: `LastFM`, `KuGou`, `LrcLib` for lyrics fallback chains
- **Proxy support**: Configured via `App.kt`'s `Authenticator` for regions without YouTube Music

## Development Workflows

### Build & Variants
```bash
# Debug build (FOSS variant, default)
./gradlew assembleDebug

# Release build with all optimizations
./gradlew assembleRelease

# Flavor breakdown: 2 dimensions (variant + abi)
# Variant: foss (F-Droid) | gms (Google Cast)
# ABI: universal | arm64 | armeabi | x86 | x86_64
```

### Database Migrations
- **Schema location**: `app/schemas/com.metrolist.music.db.InternalDatabase/{versionNumber}.json`
- **Auto vs. Manual**: Use `@AutoMigration` annotations + optional `AutoMigrationSpec` class for post-migration logic
- **Adding a column**: Edit entity, bump version, create migration spec if data transformation needed
- **Testing**: Fallback to destructive migration on debug with `fallbackToDestructiveMigration()` (see `AppModule.kt`)

### Adding a New Feature Screen
1. Create `ui/screens/NewFeatureScreen.kt` with `@Composable fun NewFeatureScreen(navController: NavController)`
2. Add ViewModel in `viewmodels/NewFeatureViewModel.kt` with `@HiltViewModel`
3. Register route in `NavigationBuilder.kt`: `composable(route = "newfeature", content = { NewFeatureScreen(navController) })`
4. Inject UI components from `ui/component/` (e.g., `YouTubeGridItem`, `ShimmerHost` placeholders)
5. Use `LocalPlayerConnection.current` for playback, `LocalDatabase.current` for DB queries

### Adding External Module Integration
- Create new module folder (e.g., `newlibrary/`)
- Add to `settings.gradle.kts`: `include(":newlibrary")`
- Create DI module in `newlibrary/di/` if needed
- Expose via `newlibrary/build.gradle.kts` with library functions
- Import into `app` and use in services/screens

## Project-Specific Conventions

### Naming & Organization
- **Package structure**: `com.metrolist.music.{feature}` (e.g., `com.metrolist.music.playback`, `com.metrolist.music.ui.screens`)
- **Database entities**: Suffix with `Entity` (e.g., `SongEntity`)
- **ViewModels**: Class name + `ViewModel` (e.g., `AccountViewModel`)
- **Routes**: Lowercase kebab-case for navigation (e.g., `account`, `mood_and_genres`)

### State Consistency
- **Database is source of truth**: UI fetches via DAOs, caches in ViewModels' StateFlow
- **Optimistic updates**: When modifying playlists/library, update DB immediately, show snackbar on failure
- **Listen to updates**: Use `database.song(videoId)` collector to watch entity changes

### Error Handling
- **Network errors**: Show snackbar via `LocalSnackbarHostState`
- **Logging**: Use `Timber.d()` / `Timber.e()` (configured in `App.kt`)
- **Crash handling**: `CrashHandler` in `utils/` captures uncaught exceptions

### Testing & CI
- **Build CI**: GitHub Actions in `.github/workflows/`
- **Environment secrets**: `LASTFM_API_KEY`, `LASTFM_SECRET` injected at build time (see `build.gradle.kts`)
- **Build variants in CI**: Both FOSS and GMS flavors built for releases

## Common Tasks & Code Locations

| Task | Location |
|------|----------|
| Add song property | `db/entities/SongEntity.kt` (+ migration spec) |
| New menu action | `ui/menu/{Type}Menu.kt` (e.g., `SongMenu.kt`) |
| Modify playback behavior | `playback/MusicService.kt` or `PlayerConnection.kt` |
| Settings page | `ui/screens/settings/` (e.g., `SettingsScreen.kt`) |
| Lyrics integration | `lyrics/` folder + `LyricsProvider` interface |
| UI theme/colors | `ui/theme/Color.kt` or `Typography.kt` |

## Key Dependencies
- **Compose**: 1.10.1, Material3: 1.5.0-alpha09
- **Media3**: 1.7.1 (ExoPlayer, Session)
- **Room**: 2.8.4
- **Hilt**: 2.57.2
- **Ktor Client**: 3.3.3 (HTTP requests)
- **Timber**: 5.0.1 (Logging)
- **Coil 3**: Image loading with disk cache
- **NewPipe Extractor**: YouTube video source extraction

## Notes for Contributors
- **Min SDK**: 26 (Android 8.0)
- **Target SDK**: 36 (Android 15)
- **Kotlin**: 2.3.0 with KSP for annotation processing
- **Java**: Target 21 (desugaring enabled for older APIs)
- **Locale support**: Weblate integration for crowdsourced translations
