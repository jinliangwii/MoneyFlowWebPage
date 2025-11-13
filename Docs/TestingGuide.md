# Testing Guide for MoneyFlow

This project has two test targets configured:

1. **MoneyFlowTests** - Unit tests using Swift Testing framework (modern approach)
2. **MoneyFlowUITests** - UI tests using XCTest framework

## Running Tests in Xcode

### Quick Actions
- **Run all tests**: Press `⌘ + U` (Command + U)
- **Run current test**: Click the diamond icon (◇) next to the test function, or press `⌘ + Option + U`
- **Run specific test**: Right-click on any test function → "Run [TestName]"

### Test Navigator
1. Open Test Navigator: `⌘ + 6` or View → Navigators → Show Test Navigator
2. Select tests you want to run
3. Click the "Play" button or right-click → Run

## Running Tests from Command Line

### Basic Command Structure
```bash
xcodebuild test -scheme MoneyFlow -destination '<destination-spec>'
```

### Available Destinations

#### macOS (Recommended for quick unit tests)
```bash
xcodebuild test -scheme MoneyFlow -destination 'platform=macOS'
```

#### iOS Simulator
First, list available simulators:
```bash
xcrun simctl list devices available
```

Then run tests on a specific simulator:
```bash
xcodebuild test -scheme MoneyFlow -destination 'platform=iOS Simulator,name=iPad (A16)'
```

Or use the generic simulator:
```bash
xcodebuild test -scheme MoneyFlow -destination 'platform=iOS Simulator,name=Any iOS Simulator Device'
```

### Run Only Unit Tests
```bash
xcodebuild test -scheme MoneyFlow -destination 'platform=macOS' -only-testing:MoneyFlowTests
```

### Run Only UI Tests
```bash
xcodebuild test -scheme MoneyFlow -destination 'platform=iOS Simulator,name=iPad (A16)' -only-testing:MoneyFlowUITests
```

### Run Specific Test Function
```bash
xcodebuild test -scheme MoneyFlow -destination 'platform=macOS' -only-testing:MoneyFlowTests/MoneyFlowTests/example
```

### Show Test Results
Add verbose output:
```bash
xcodebuild test -scheme MoneyFlow -destination 'platform=macOS' | xcpretty
```

Note: `xcpretty` is optional but provides better formatted output. Install with: `gem install xcpretty`

## Test Frameworks Used

### Swift Testing (MoneyFlowTests)
- Modern testing framework introduced in Swift 5.9+
- Uses `@Test` attribute
- Assertions use `#expect(...)` instead of `XCTAssert`
- Example:
```swift
@Test func example() async throws {
    #expect(1 + 1 == 2)
}
```

### XCTest (MoneyFlowUITests)
- Traditional iOS testing framework
- Uses `XCTestCase` classes
- Assertions use `XCTAssert*` functions
- Example:
```swift
func testExample() throws {
    XCTAssertTrue(true)
}
```

## Tips

1. **Quick iteration**: Use macOS destination for faster test runs (no simulator startup)
2. **CI/CD**: Use command-line `xcodebuild test` for automated testing
3. **Debugging**: Set breakpoints in tests and use `⌘ + Y` to debug tests
4. **Coverage**: Enable code coverage in scheme settings: Product → Scheme → Edit Scheme → Test → Code Coverage


