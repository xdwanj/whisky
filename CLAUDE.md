# Repository Guidelines

## Project Structure & Module Organization
`Whisky/` contains the macOS SwiftUI app. Keep UI in `Views/`, state and orchestration in `View Models/`, reusable helpers in `Utils/`, and type/category extensions in `Extensions/`. Assets and localization live in `Whisky/Assets.xcassets` and `Whisky/Localizable.xcstrings`.

`WhiskyKit/` is a standalone Swift package for shared domain and Wine-related logic. Core models live under `WhiskyKit/Sources/WhiskyKit/Whisky/`, low-level PE parsing under `PE/`, and platform helpers under `Extensions/` and `Utils/`. `WhiskyCmd/` provides the CLI target, and `WhiskyThumbnail/` contains the Quick Look thumbnail extension.

## Build, Test, and Development Commands
Use full Xcode, not Command Line Tools only, for app builds.

- `open Whisky.xcodeproj` opens the app, CLI, and thumbnail targets in Xcode.
- `xcodebuild -project Whisky.xcodeproj -scheme Whisky -configuration Debug build` builds the main app from the command line.
- If local signing is unavailable, use `xcodebuild -project Whisky.xcodeproj -scheme Whisky -configuration Debug -derivedDataPath /tmp/WhiskyDerivedData CODE_SIGNING_ALLOWED=NO CODE_SIGNING_REQUIRED=NO CODE_SIGN_IDENTITY='' DEVELOPMENT_TEAM='' build` for a local unsigned build.
- The local app bundle from that build is `/tmp/WhiskyDerivedData/Build/Products/Debug/Whisky.app`, and can be launched with `open /tmp/WhiskyDerivedData/Build/Products/Debug/Whisky.app`.
- `xcodebuild -project Whisky.xcodeproj -scheme WhiskyCmd build` builds the bundled CLI helper.
- `swift build --package-path WhiskyKit` builds the Swift package in isolation.
- `swiftlint --strict` runs the same lint gate used in CI.

Keep generated artifacts out of the repository root before app builds. The Xcode SwiftLint phase runs `swiftlint --strict` from the repo root and relies on `.swiftlint.yml` exclusions rather than `.gitignore`, so stale `build/` or `WhiskyKit/.build/` directories can cause false failures by linting generated files and vendored package sources.

## Coding Style & Naming Conventions
Follow existing Swift conventions: 4-space indentation, `UpperCamelCase` for types, `lowerCamelCase` for properties and methods, and singular filenames that match the main type, such as `Bottle.swift` or `BottleVM.swift`. Prefer small extensions in `Extensions/` and keep SwiftUI view logic thin by moving work into view models or `WhiskyKit`.

SwiftLint enforces `force_unwrapping` and a required GPL file header. New Swift files should start from the Xcode template so the header matches `Whisky.xcodeproj/xcshareddata/IDETemplateMacros.plist`.

## Testing Guidelines
There is currently no committed `Tests/` target. For package-level changes, at minimum run `swift build --package-path WhiskyKit`. When adding tests, place them under `WhiskyKit/Tests`, mirror the source module name, and use `XCTest` method names like `testBottleLoadsPinnedPrograms()`.

## Commit & Pull Request Guidelines
Recent history favors short, imperative subjects such as `Bump version`, `Maintenance Notice`, or scoped messages like `feat: avx feature switch (#1034)`. Keep the first line concise, describe behavior changes in the body when needed, and reference the issue or PR number when applicable.

Pull requests should summarize user-visible impact, list validation steps, and include screenshots for SwiftUI changes. Call out macOS or Apple Silicon assumptions explicitly when they affect review.
