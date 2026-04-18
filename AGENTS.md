# Repository Guidelines

## Project Structure & Module Organization
`Whisky/` contains the macOS SwiftUI app. Keep UI in `Views/`, state and orchestration in `View Models/`, reusable helpers in `Utils/`, and type/category extensions in `Extensions/`. Assets and localization live in `Whisky/Assets.xcassets` and `Whisky/Localizable.xcstrings`.

`WhiskyKit/` is a standalone Swift package for shared domain and Wine-related logic. Core models live under `WhiskyKit/Sources/WhiskyKit/Whisky/`, low-level PE parsing under `PE/`, and platform helpers under `Extensions/` and `Utils/`. `WhiskyCmd/` provides the CLI target, and `WhiskyThumbnail/` contains the Quick Look thumbnail extension.

## Build, Test, and Development Commands
Use full Xcode, not Command Line Tools only, for app builds.

- `open Whisky.xcodeproj` opens the app, CLI, and thumbnail targets in Xcode.
- `xcodebuild -project Whisky.xcodeproj -scheme Whisky -configuration Debug build` builds the main app from the command line.
- `xcodebuild -project Whisky.xcodeproj -scheme WhiskyCmd build` builds the bundled CLI helper.
- `swift build --package-path WhiskyKit` builds the Swift package in isolation.
- `swiftlint --strict` runs the same lint gate used in CI.

## Coding Style & Naming Conventions
Follow existing Swift conventions: 4-space indentation, `UpperCamelCase` for types, `lowerCamelCase` for properties and methods, and singular filenames that match the main type, such as `Bottle.swift` or `BottleVM.swift`. Prefer small extensions in `Extensions/` and keep SwiftUI view logic thin by moving work into view models or `WhiskyKit`.

SwiftLint enforces `force_unwrapping` and a required GPL file header. New Swift files should start from the Xcode template so the header matches `Whisky.xcodeproj/xcshareddata/IDETemplateMacros.plist`.

## Testing Guidelines
There is currently no committed `Tests/` target. For package-level changes, at minimum run `swift build --package-path WhiskyKit`. When adding tests, place them under `WhiskyKit/Tests`, mirror the source module name, and use `XCTest` method names like `testBottleLoadsPinnedPrograms()`.

## Commit & Pull Request Guidelines
Recent history favors short, imperative subjects such as `Bump version`, `Maintenance Notice`, or scoped messages like `feat: avx feature switch (#1034)`. Keep the first line concise, describe behavior changes in the body when needed, and reference the issue or PR number when applicable.

Pull requests should summarize user-visible impact, list validation steps, and include screenshots for SwiftUI changes. Call out macOS or Apple Silicon assumptions explicitly when they affect review.
