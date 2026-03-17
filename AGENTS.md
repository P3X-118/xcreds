# AGENTS.md - XCreds Development Guide

This file provides guidance for AI agents working on the XCreds codebase.

## Project Overview

XCreds is a macOS SSO and authentication tool consisting of:
- **XCreds.app** - Main user-space application
- **XCreds Login Plugin** - Security agent that replaces login window
- **XCreds AutoFill Extension** - Password autofill extension
- **tap** - Test agent plugin

## Build Commands

### Prerequisites

```bash
git clone https://github.com/twocanoes/xcreds.git
cd xcreds
git submodule init
git submodule update
carthage update
```

### Building

**Build the main app:**
```bash
xcodebuild -project XCreds.xcodeproj -scheme XCreds -configuration Debug build
```

**Build Release:**
```bash
xcodebuild -project XCreds.xcodeproj -scheme XCreds -configuration Release build
```

**Build specific scheme:**
```bash
xcodebuild -project XCreds.xcodeproj -scheme "XCreds Login Plugin" -configuration Debug build
```

**Build with custom derived data path:**
```bash
xcodebuild -project XCreds.xcodeproj -scheme XCreds -configuration Debug -derivedDataPath ./build build
```

### Running the App

The app requires running from Xcode or with proper code signing. For development:
```bash
open XCreds.xcodeproj
```
Then press Cmd+R in Xcode.

### Resolving Dependencies

```bash
carthage update
xcodebuild -resolvePackageDependencies
```

### Testing

**Note:** This project currently has **no formal test suite**. Tests are run manually via the `app_to_test.sh` script which deploys to a remote Mac for testing.

If adding tests, use Xcode's XCTest framework:
```bash
# Run all tests (if configured)
xcodebuild -project XCreds.xcodeproj -scheme XCreds test

# Run a specific test class
xcodebuild -project XCreds.xcodeproj -scheme XCreds test -only-testing:TestClassName

# Run a specific test
xcodebuild -project XCreds.xcodeproj -scheme XCreds test -only-testing:TestClassName/testMethod
```

### Linting

No SwiftLint or SwiftFormat configuration exists. If adding, consider:
- 120 character line limit
- Enabled rules should match existing code style

### Code Signing

Builds require appropriate code signing identity. For development, use:
```bash
xcodebuild -project XCreds.xcodeproj -scheme XCreds -configuration Debug CODE_SIGN_IDENTITY="-" CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO build
```

## Code Style Guidelines

### General Principles

- Use Swift 5+ features (async/await, property wrappers, etc.)
- Maintain compatibility with macOS 10.15+ (check `@available` attributes)
- Use guard statements for early returns
- Keep functions focused and small

### Naming Conventions

- **Types/Classes/Structs/Enums**: PascalCase (e.g., `TokenManager`, `KeychainError`)
- **Properties/Variables**: camelCase (e.g., `userName`, `isEnabled`)
- **Functions/Methods**: camelCase (e.g., `getPassword()`, `fetchTokens()`)
- **Constants**: camelCase with meaningful names (e.g., `maxRetryCount`)
- **Enums**: PascalCase for type, camelCase for members

### Imports

Organize imports in the following order:
1. Foundation
2. Cocoa/AppKit
3. Third-party frameworks (OIDCLite, ArgumentParser, CryptoKit, etc.)
4. Local project imports

```swift
import Foundation
import Cocoa
import OIDCLite
import ArgumentParser
import CryptoKit
```

### Formatting

- 4 spaces for indentation (not tabs)
- No trailing whitespace
- One blank line between top-level definitions
- Opening brace on same line as declaration

### Types and Protocols

- Use structs for simple data containers
- Use classes for objects with identity or complex behavior
- Use protocols for abstraction
- Mark classes as `final` when not subclassed

```swift
struct Creds {
    var password: String?
    public var accessToken: String?
}

final class TokenManager {
    // ...
}
```

### Error Handling

Use custom error enums for domain-specific errors:

```swift
enum KeychainError: Error {
    case notConnected
    case notLoggedIn
    case noPassword
}

enum ProcessTokenResult: Error {
    case error(String)
    case invalidCredentials
}
```

Use try/catch with specific error handling:

```swift
do {
    let data = try encoder.encode(auditRecord)
    try data.write(to: configFileURL)
} catch {
    TCSLogWithMark(error.localizedDescription)
}
```

### Optionals

- Use optionals for values that may be nil
- Use guard let/if let for safe unwrapping
- Use ?? for default values when appropriate

```swift
guard let password = password, password.isEmpty == false else {
    return nil
}

let homeDir = userDetails["homeDirectory"] ?? "/Users/default"
```

### Logging

The project uses multiple logging mechanisms:

1. **TCSLogWithMark()** - Primary logging (from TCSUnifiedLogger):
```swift
TCSLogWithMark("Finding password in keychain")
TCSLogWithMark("Error: \(error.localizedDescription)")
```

2. **OSLog** - For structured logging:
```swift
os_log("using provided keychain", log: log!, type: .debug)
```

### macOS Version Compatibility

Always mark new APIs with `@available`:

```swift
@available(macOS, deprecated: 11)
struct xcreds: ParsableCommand {
    // ...
}
```

### Bridging Headers

When adding Objective-C code:
1. Create or update bridging header in the target
2. Import Objective-C headers:

```objc
// XCreds-Bridging-Header.h
#import "KerbUtil.h"
#import "TCSTKSmartCard.h"
```

### Async/Await

Use modern async/await for asynchronous operations:

```swift
func oidc() async throws -> OIDCLite {
    // async implementation
}

Task {
    do {
        let oidc = try await tokenManager.oidc()
    } catch {
        TCSLogWithMark(error.localizedDescription)
    }
}
```

### Properties and Access Control

- Use `public` for API that needs to be accessed outside the type
- Use `private` for internal implementation details
- Use `internal` (default) for module-internal API

```swift
public var accessToken: String?
private var internalCache: [String: Any] = nil
```

### UserDefaults

Use the project's defaults wrapper:

```swift
let defaults = DefaultsOverride.standard
let value = defaults.string(forKey: PrefKeys.clientID.rawValue)
```

### Architecture Notes

- **DSQueryable protocol**: Implement for directory service queries
- **TokenManagerFeedbackDelegate**: Protocol for token update callbacks
- **OIDCLite**: External dependency for OIDC authentication
- **KeychainUtil**: Singleton for keychain operations
- **Shared/**: Code shared between app and login plugin

## Common Issues

### Build Failures
- Ensure Carthage dependencies are built: `carthage build`
- Clean derived data if needed: `xcodebuild clean`
- Check code signing requirements

### Keychain Issues
- App requires keychain access entitlements
- Login plugin uses login.keychain

### Login Plugin Issues
- Requires proper code signing for distribution
- Must be installed in correct location for testing

## Release Process

The build script handles versioning and release:
```bash
./build.sh
```

This increments build numbers, updates manifests, and creates archives.
